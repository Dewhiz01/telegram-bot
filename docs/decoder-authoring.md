# Decoder Authoring Guide for New Contract Versions

This guide explains how to author, extend, and test event decoders for Soroban smart contracts in `@mimir/telegram-bot`. It documents the runtime dispatch architecture, the incremental versioning pattern, XDR safety boundaries, and the checklist required for pull requests.

---

## 1. Overview & Decoder Architecture

The Mimir notifier listens to events emitted by the `mimir-market` and `mimir-squad` Soroban smart contracts on the Stellar network.

### 1.1 Soroban Wire Format (`#[contractevent]`)

Soroban contracts use the `#[contractevent]` macro, which standardizes event wire layout into topics and values:
```text
topic[0]   snake_case event name symbol (e.g. "claim_created", "market_settled")
topic[1..] #[topic]-annotated fields, in declaration order (u64 IDs, Address strkers, etc.)
value      ScVal map containing all remaining non-topic fields keyed by symbol name
```

For example, a `ClaimChallenged` contract event with `#[topic] id: u64`, `#[topic] challenger: Address`, and `stake: i128` arrives over RPC as:
* `topic[0]`: Symbol `"claim_challenged"`
* `topic[1]`: U64 `1`
* `topic[2]`: Address `"GBAC..."`
* `value`: Map `{ "stake": 50000000n }`

### 1.2 The Never-Throw Invariant

The decoding entry point [`ecodeEvent`](../src/stellar/decode.ts) in `src/stellar/decode.ts` **never throws**. Any unexpected condition&mdash;missing topics, oversized XDR, malformed values, unknown event names, or scalar parsing mismatches&mdash;safely returns a typed `UnknownPayload`:

```ts
export interface UnknownPayload {
  name: "unknown";
  eventName: string;
  reason?: string;
}
```

A live notifier daemon must never crash because an unhandled or future contract event appeared on-chain.

### 1.3 Unknown Event Names vs. Malformed Known Events

Decoders distinguish two distinct failure modes, and each is counted separately for observability:

| Case | When it occurs | Status code | Counter |
| ---- | ---- | ---- | ---- |
| **Unknown event name** | Topic[0] is not a known event for the contract source and version (e.g. a future event not yet supported) | `unknown_event` | `unknownEventCount` |
| **Malformed known event** | Topic[0] is a known event name, but its topics or value fail schema/scalar validation | `malformed_event` | `malformedEventCount` |

Both cases return an `UnknownPayload` with a distinct `reason` string and are nonfatal to event polling: the scanner continues to the next event and the cursor advances normally. The corresponding counter is incremented so operators can alert on either condition independently.

---

## 2. Configuration & Version-Dispatch Flow

The bot dispatches decoders according to per-contract version settings configured in the environment.

### 2.1 Configuration Path

```text
Environment (.env)
  MARKET_CONTRACT_VERSION="v2"
  QUAD_CONTRACT_VERSION="v1"
        │
        ▼
Configuration Loader (src/config.ts)
  c.contractVersion("MARKET_CONTRACT_VERSION", "v1")
  c.contractVersion("QUAD_CONTRACT_VERSION", "v1")
  - Validates against /^[a-z0-9_.-]+$/; lowercases and trims.
  - Stored on StellarConfig as marketContractVersion and squadContractVersion.
        │
        ▼
Poller Target Initialization (src/poller.ts)
  targets = [
    { source: "market", contractId: config.marketContractId, version: config.marketContractVersion },
    { source: "squad",  contractId: config.squadContractId,  version: config.squadContractVersion  },
  ]
        │
        ▼
Event Scanner (src/stellar/events.ts)
  decoded.push(decodeEvent(target.source, event, target.version));
        │
        ▼
Decoder Registry Dispatch (src/stellar/decode.ts)
  normVersion = normalizeContractVersion(version);
  decoderMap  = source === "market" ? MARKET_DECODERS : SQUAD_DECODERS;
  decoder     = decoderMap[normVersion] ?? decoderMap[DEFAULT_CONTRACT_VERSION] ?? decodeMarketV1;
  payload     = decoder(eventName, topics, fields);
```

### 2.2 Version Normalization and Fallback Behavior

1. **Normalization:** [`normalizeContractVersion`](../src/stellar/decode.ts) trims whitespace, converts input to lowercase, and falls back to `DEFAULT_CONTRACT_VERSION` (`"v1"`) when given an empty or undefined string.
2. **Unsupported Versions:** When an unmapped or future version string (e.g., `"v99"`) is passed, `decoderMap[normVersion]` returns `undefined`. The lookup immediately falls back to `decoderMap[DEFAULT_CONTRACT_VERSION]`, which points to `decodeMarketV1` or `decodeSquadV1`.
3. **No Rejection:** Unsupported version strings are **not** rejected with a thrown error. The event is safely processed through the baseline v1 decoder, and `event.version` records the normalized version string for observability.

---

## 3. The Incremental Versioning Pattern (v1 vs. v2)

The codebase uses an **incremental delegation pattern** rather than duplicating entire decoder implementations across contract versions.

A new version decoder only handles:
1. Existing events that received new or changed fields.
2. Brand new events specific to that version.

All other events delegate directly to the previous version decoder.

### 3.1 Market Decoder (`decodeMarketV2`)

In `src/stellar/decode.ts`:

```ts
export function decodeMarketV2(
  name: string,
  topics: unknown[],
  fields: Record<string, unknown>,
): MarketPayload | AdminPayload | null {
  // 1. claim_created extended with optional 'title' in v2
  if (name === "claim_created" && fields.title !== undefined) {
    return {
      name,
      claimId: num(topicAt(topics, 1, "claim_created.id"), "claim_created.id"),
      creator: addr(topicAt(topics, 2, "claim_created.creator"), "claim_created.creator"),
      category: str(fields.category, "claim_created.category"),
      title: str(fields.title, "claim_created.title"),
    };
  }

  // 2. claim_resolved extended with optional 'resolver' in v2
  if (name === "claim_resolved" && fields.resolver !== undefined) {
    return {
      name,
      claimId: num(topicAt(topics, 1, "claim_resolved.id"), "claim_resolved.id"),
      winnerSide: num(fields.winner_side, "claim_resolved.winner_side"),
      summary: str(fields.summary, "claim_resolved.summary"),
      confidence: num(fields.confidence, "claim_resolved.confidence"),
      evidenceHash: hex(fields.evidence_hash, "claim_resolved.evidence_hash"),
      resolver: optAddr(fields.resolver, "claim_resolved.resolver"),
    };
  }

  // 3. All other events (or v2 events omitting the extension fields) fall through to v1
  return decodeMarketV1(name, topics, fields);
}
```

### 3.2 Squad Decoder (`decodeSquadV2`)

Similarly, `decodeSquadV2` only intercepts `market_created` when `fields.category !== undefined`. All other squad events (`deposited`, `withdrawn`, `resolved`, `claimed`, `fees_claimed`) or events where `category` is omitted delegate directly to `decodeSquadV1`.

### 3.3 Nature of the Extension Pattern & Fallback Boundaries

Inspecting `fields.<field> !== undefined` an application-level pattern used in `decodeMarketV2` and `decodeSquadV2` to accommodate backward-compatible contract changes:
* **Fallback is conditional:** Falling through to the prior decoder (`decodeMarketV1` or `decodeSquadV1`) succeeds **only if** the event without the new field conforms to what that prior decoder expects.
* **Required fields still apply:** If any baseline topic or field required by the prior decoder (such as `category` on `claim_created`, or valid strkey addresses) is missing or malformed, the prior decoder will throw a `DecodeError`, and `decodeEvent` will return a `malformed_event` payload (see &sect;1.3).
* **Breaking changes:** If a new contract version fundamentally changes existing field types or topic order, delegation alone is insufficient; that event must be explicitly decoded in the newer decoder without delegating to the old one.

---

## 4. Authoring Decoders: Step-by-Step

### 4.1 Adding a Field to an Existing Event

When a contract upgrade adds a field to an existing event (e.g., adding `tags` to `claim_created`):

1. **Update the Payload Type:** Mark the field as optional in `src/stellar/decode.ts` so older events without the field remain type-compatible:
   ```ts
   export type MarketPayload =
     | { name: "claim_created"; claimId: number; creator: string; category: string; title?: string; tags?: string }
     | ...;
   ```
2. **Update the Versioned Decoder:**
   Inspect the field in the versioned decoder. If present, parse it with the appropriate scalar helper; otherwise, fall back to the prior version:
   ```ts
   if (name === "claim_created" && fields.tags !== undefined) {
     return {
       name,
       claimId: num(topicAt(topics, 1, "claim_created.id"), "claim_created.id"),
       creator: addr(topicAt(topics, 2, "claim_created.creator"), "claim_created.creator"),
       category: str(fields.category, "claim_created.category"),
       tags: str(fields.tags, "claim_created.tags"),
     };
   }
   ```
3. **Handle Missing Values:** If the event arrives without the new field, the guard evaluates to `false` and delegates to the prior decoder (`decodeMarketV1`). The event will decode successfully without the extension field, **provided all baseline fields required by the prior decoder are present and valid**.

### 4.2 Adding a New Event Name

When a contract introduces a completely new event (e.g., `claim_escalated`):

1. **Define the Payload Type:**
   ```ts
   export type MarketClaimEscalatedPayload = {
     name: "claim_escalated";
     claimId: number;
     escalator: string;
     bond: bigint;
   };
   ```
2. **Add to the Payload Union:**
   Include `MarketClaimEscalatedPayload` in `MarketPayload`.
3. **Register Topic Schema:**
   Add the topic layout to `MARKET_TOPIC_SCHEMAS` (or `SQUAD_TOPIC_SCHEMAS`) for topic validation:
   ```ts
   export const MARKET_TOPIC_SCHEMAS: Readonly<Record<string, TopicSchema>> = {
     ...
     claim_escalated: { expectedCount: 3, args: ["int", "address"] },
   };
   ```
4. **Implement Decoder Case:**
   Add `case "claim_escalated":  to the decoder function using scalar helpers:
   ```ts
   case "claim_escalated":
     return {
       name,
       claimId: num(topicAt(topics, 1, "claim_escalated.id"), "claim_escalated.id"),
       escalator: addr(topicAt(topics, 2, "claim_escalated.escalator"), "claim_escalated.escalator"),
       bond: amount(fields.bond, "claim_escalated.bond"),
     };
   ```
5. **Admin Event Routing:**
   Events that represent administrative operations (such as oracle changes or fee adjustments) route through `decodeAdmin`. Unrecognized event names fall through `switch (name)` to `default: return decodeAdmin(name, topics, fields)`. If `decodeAdmin` also returns `null`, `decodeEvent` yields `{ name: "unknown", eventName, reason: "unknown_event" }`.

---

## 5. Registering a Future Contract Version (e.g. Hypothetical `v3`)

> **Note:** Decoder names like `decodeMarketV3` and version `"v3"` in this section are hypothetical examples illustrating how future contract versions should be wired. They do not currently exist in the repository.

To introduce a new contract version:

1. **Understand `ContractVersion` vs. Registries:**
   In `src/stellar/decode.ts`:
   ```ts
   export type ContractVersion = "v1" | "v2" | string;
   export const DEFAULT_CONTRACT_VERSION: ContractVersion = "v1";
   export const SUPPORTED_CONTRACT_VERSIONS: ContractVersion[] = ["v1", "v2"];
   ```
   Notice that `ContractVersion` ends with ` | string`, so TypeScript already permits any arbitrary string value. However:
   - `SUPPORTED_CONTRACT_VERSIONS` is the runtime array of officially recognized versions; update it when adding support (e.g. `["v1", "v2", "v3"]`).
   - More crucially, runtime version dispatch relies on the `MARKET_DECODERS` and `SQUAD_DECODERS` registries. A version string will only execute its specific decoder if registered in these maps.
2. **Author the Hypothetical Decoder:**
   Implement `decodeMarketV3` (or `decodeSquadV3`) following the delegation pattern described in &sect;3.
