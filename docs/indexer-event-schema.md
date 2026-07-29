# Indexer Event Schema

Contract-emitted events that off-chain indexers and wallets can subscribe to
for the StellarTip contract.  This document is the **indexer-facing counterpart**
of [`events_catalog.md`](./events_catalog.md) — it focuses on topic-filtering
queries, trust semantics, and ordering guarantees rather than Rust-level type
shapes.

---

## Trust model

- **Emitter address is ground truth.**  Every event listed below is emitted from
  the StellarTip contract's deployed address.  An indexer MUST filter by
  `contract == <deployed_address>` before interpreting payloads.
- **Events are not independently signed.**  The Stellar host provides intrinsic
  replay protection and ledger-finality guarantees, but topic/payload bytes
  have no extra cryptographic signature beyond what the host attaches.
- **Rollback on panic.**  If `tip()` panics (e.g. `TransferFailed`), all events
  emitted *during* that invocation (including any partial `TIP` event) are
  atomically rolled back alongside all storage writes.  An indexer will never
  observe a `TIP` event whose corresponding token transfer did not settle.

---

## Ordering

| Scope | Guarantee |
|-------|-----------|
| Within one transaction | Emission order (`env.events().publish(...)` call order) |
| Across transactions in one ledger | Ledger-insertion order |
| Across ledgers | Ledger sequence-number order |

---

## Event index

There are **14 events**.  Each section includes the topic-filtering query
string an indexer can use to subscribe.

In the query templates below, `C...` denotes the deployed contract
address (32-byte Stellar account ID, hex) and `alice...` denotes a
Stellar account ID belonging to a creator or supporter.  The filter
strings use a conceptual `key=value` notation; the actual Soroban RPC
wire format uses JSON-RPC `topicFilters` as a 2-D array of SCVal
values: `[["Symbol", "Address"], ...]`.

---

### 1. `INIT` — Contract Initialised

**Topic:** `(Symbol("INIT"), Address(admin))`

**Payload:** `(Address(fee_recipient), u32(fee_bps))`

```
contract=C...&topic=INIT
```

---

### 2. `CREG` — Creator Registered

**Topic:** `(Symbol("CREG"), Address(creator))`

**Payload:** `(Symbol(username), u64(ledger_timestamp))`

```
contract=C...&topic=CREG
contract=C...&topic=CREG&topic.creator=alice...
```

---

### 3. `TIP` — Tip Sent

**Topic:** `(Symbol("TIP"), Address(from))`

**Payload:** `(Address(creator), Address(token), i128(amount), i128(fee), u64(index))`

```
contract=C...&topic=TIP
contract=C...&topic=TIP&topic.from=alice...
```

---

### 4. `WDRW` — Withdrawal

**Topic:** `(Symbol("WDRW"), Address(creator))`

**Payload:** `(Address(token), i128(amount))`

```
contract=C...&topic=WDRW
contract=C...&topic=WDRW&topic.creator=alice...
```

---

### 5. `PUPD` — Profile Updated

**Topic:** `(Symbol("PUPD"), Address(creator))`

**Payload:** `(Symbol(username), String(display_name))`

```
contract=C...&topic=PUPD
```

---

### 6. `UREG` — Creator Unregistered

**Topic:** `(Symbol("UREG"), Address(creator))`

**Payload:** `()` (empty)

```
contract=C...&topic=UREG
```

---

### 7. `PAUS` — Contract Paused

**Topic:** `(Symbol("PAUS"), Address(admin))`

**Payload:** `()` (empty)

```
contract=C...&topic=PAUS
```

---

### 8. `UNPA` — Contract Unpaused

**Topic:** `(Symbol("UNPA"), Address(admin))`

**Payload:** `()` (empty)

```
contract=C...&topic=UNPA
```

---

### 9. `FEEC` — Fee Percentage Changed

**Topic:** `(Symbol("FEEC"), Address(admin))`

**Payload:** `u32(new_fee_bps)`

```
contract=C...&topic=FEEC
```

---

### 10. `FERC` — Fee Recipient Changed

**Topic:** `(Symbol("FERC"), Address(admin))`

**Payload:** `Address(new_fee_recipient)`

```
contract=C...&topic=FERC
```

---

### 11. `ADMC` — Admin Changed

**Topic:** `(Symbol("ADMC"), Address(old_admin))`

**Payload:** `Address(new_admin)`

```
contract=C...&topic=ADMC
```

---

### 12. `CAPMC` — Max Creators Cap Changed

**Topic:** `(Symbol("CAPMC"), Address(admin))`

**Payload:** `u32(new_cap)` — `0` means unlimited.

```
contract=C...&topic=CAPMC
```

---

### 13. `CAPMT` — Max Tips Per Creator Cap Changed

**Topic:** `(Symbol("CAPMT"), Address(admin))`

**Payload:** `u32(new_cap)` — `0` means unlimited.

```
contract=C...&topic=CAPMT
```

---

### 14. `MINTC` — Minimum Tip Amount Changed

**Topic:** `(Symbol("MINTC"), Address(admin))`

**Payload:** `i128(new_minimum)` — `0` means no minimum.

```
contract=C...&topic=MINTC
```

---

## Indexer integration checklist

1. **Filter by contract address.**  Discard any event whose emitter is not the
   known StellarTip deployment.
2. **Decode topics as `(Symbol, Address)`.**  The first topic element is always
   a 1–9 character Soroban `symbol_short!`; the second is a 32-byte account ID.
3. **Decode payloads per the table above.**  Use Soroban XDR / SCVal decoding
   with the types listed for each event.
4. **Handle rollbacks.**  If an RPC returns a failed transaction, discard all
   events from it — the Stellar host guarantees they were never persisted.
5. **Paginate with ledger sequence.**  Use `startLedger` / `endLedger` cursor
   semantics; event order within a ledger is deterministic.

---

## Cross-references

- [`events_catalog.md`](./events_catalog.md) — Rust-level topic/payload shapes
  sourced directly from `env.events().publish()` calls.
- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — storage layout, lifecycle state
  machine, and cross-cutting invariants.
- [`API_REFERENCE.md`](./API_REFERENCE.md) — public method signatures & error
  codes.
