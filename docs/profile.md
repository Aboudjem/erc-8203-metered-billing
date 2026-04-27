# ERC-8203 Worked Example: Metered Billing for Agent Services

> Non-normative profile demonstrating how ERC-8203 conditional settlement primitives
> support usage-based billing between autonomous agents.

## Problem

An agent (Client) consumes services from another agent (Provider): API calls, compute units, token generations, etc. Billing per-call on-chain is prohibitively expensive. We need a pattern where:

- Usage is tracked off-chain at high frequency
- Settlement happens periodically or on-demand
- Disputes are resolved on-chain only when needed
- Sessions can roll over without re-locking funds each time

## Actors

| Role | Description |
|------|-------------|
| **Client Agent** | Consumes services, pre-locks funds |
| **Provider Agent** | Delivers services, submits usage proofs |
| **Metering Oracle** | Attests to cumulative usage (can be the Provider itself, a TEE, or a third-party oracle) |

## Lifecycle

### 1. Session Open

Client creates a ConditionalLock representing the billing session:

```
ConditionalLock {
  lockId:         keccak256(clientAddr, providerAddr, sessionNonce)
  conditionType:  COMPOSITE
  proofType:      ORACLE_ATTESTATION   // or TEE_ATTESTATION
  payer:          clientAgent
  payee:          providerAgent
  asset:          USDC (or native token)
  amount:         maxBudget (e.g. 50 USDC)
  verifier:       meteringOracleAddress
  hostStateHash:  keccak256(sessionParams)
  expiry:         sessionStart + maxDuration
  maxRelayFee:    0  // direct settlement, no relay
}
```

The `amount` is the maximum budget for this session. The COMPOSITE condition combines:
- Sub-condition A: EXTERNAL_ASSERTION (oracle attests cumulative usage)
- Sub-condition B: TIMELOCK (session not expired)

Client signs and shares the lock off-chain with Provider. No on-chain tx yet.

### 2. Metering (Off-Chain)

Provider delivers services. Both parties maintain a running usage counter:

```
MeterState {
  sessionId:       lockId
  totalUnits:      1847        // cumulative API calls
  unitPrice:       0.002 USDC  // per call
  totalOwed:       3.694 USDC
  lastUpdate:      1743350400  // UNIX timestamp
  sequenceNumber:  42          // monotonic, prevents replay
}
```

At regular intervals (e.g. every 100 calls or every 5 minutes), Provider requests a signed meter reading from the Metering Oracle:

```
OracleAttestation {
  sessionId:       lockId
  cumulativeUnits: 1847
  cumulativeAmount: 3.694 USDC
  timestamp:       1743350400
  sequenceNumber:  42
  signature:       sign(meteringOracleKey, keccak256(above))
}
```

This is the SettlementProofRef the Provider accumulates. Only the latest matters.

### 3. Finalize (Happy Path)

Session ends. Provider calls `settleConditional` with the final meter reading:

```
settleConditional(
  channelId,
  lock,              // original ConditionalLock
  proofRef: {
    proofType:   ORACLE_ATTESTATION
    proofDigest: keccak256(finalOracleAttestation)
    verifier:    meteringOracleAddress
  },
  proof:             finalOracleAttestation.signature
)
```

The contract:
1. Verifies oracle signature via ecrecover against `verifier`
2. Confirms `cumulativeAmount <= lock.amount` (within budget)
3. Transfers `cumulativeAmount` to Provider
4. Refunds `lock.amount - cumulativeAmount` to Client
5. Marks lock as Settled

**Gas cost: one on-chain tx for the entire session**, regardless of how many API calls were made.

### 4. Refund (Timeout Path)

If Provider never settles (disappeared, crashed, delivered nothing):

```
refundConditional(channelId, lockId)
```

After `expiry`, Client reclaims the full locked amount. No oracle needed.

### 5. Dispute Path

If Client disputes the meter reading (Provider inflated usage):

- Client can submit their own counter-attestation
- The COMPOSITE condition allows the contract to evaluate both attestations
- If using TEE_ATTESTATION instead of ORACLE_ATTESTATION, the metering runs inside a trusted enclave and disputes are minimized

### 6. Rollover

For long-running agent relationships (e.g. an agent subscribed to a data feed):

```
Session 1: lock(50 USDC, expiry=T1) -> settle(37.20 USDC) -> refund(12.80)
Session 2: lock(50 USDC, expiry=T2) -> settle(48.91 USDC) -> refund(1.09)
Session 3: lock(50 USDC, expiry=T3) -> ... ongoing
```

Each session is an independent ConditionalLock. The `sessionNonce` increments.

**Optimization**: For trusted pairs, sessions can overlap. Client creates Session N+1 lock before Session N finalizes. Provider settles Session N while already delivering under Session N+1. Zero downtime.

## Batching

When a Provider serves many Clients simultaneously, individual settlement is wasteful. Using merkle roots:

```
BatchSettlement {
  merkleRoot:  keccak256(leaf1, leaf2, ..., leafN)
  leafFormat:  (lockId, cumulativeAmount, oracleSignature)
  totalLocks:  50
}
```

Provider submits one `settleConditional` with `proofType: RECEIPT_ROOT` where the proof contains the merkle root + individual merkle paths. One tx settles 50 sessions.

**Cost comparison**:
- Individual: 50 txs * ~80k gas = 4M gas
- Batched: 1 tx * ~200k gas = 200k gas (20x cheaper)

## Composition with ERC-8183

In an ERC-8183 Agentic Commerce job:

1. Client creates ERC-8183 job with `fundJob()`
2. Provider and Client open an ERC-8203 metered billing session for the work
3. Provider delivers incrementally, metering tracks progress
4. On job completion, ERC-8183 evaluator triggers `complete()`
5. ERC-8203 settlement finalizes the exact payment based on actual usage
6. Grace period (per ERC-8183 PR #13) protects against expiry races

This means the ERC-8183 job escrow sets the ceiling, while ERC-8203 metered billing determines the actual payout within that ceiling.

## Security Considerations

- **Budget cap**: `lock.amount` is hard maximum. Oracle cannot attest more than the budget.
- **Replay protection**: `sequenceNumber` in meter readings is monotonic. Old readings cannot replace newer ones.
- **Oracle trust**: Single oracle is a trust assumption. For high-value sessions, use COMPOSITE with multiple oracles or TEE_ATTESTATION.
- **Front-running**: `hostStateHash` binds the lock to session parameters. Cannot be reused across sessions.
- **Griefing**: Provider can refuse to settle, but Client reclaims after expiry. Provider loses earned fees as penalty.

## When to Use This Pattern

| Use Case | Fit |
|----------|-----|
| Agent-to-API billing (per call) | Strong |
| Compute metering (GPU hours) | Strong |
| Data feed subscriptions | Strong (with rollover) |
| One-shot purchases | Overkill. Use direct settlement. |
| Micro-payments < $0.01 | Strong. Batching amortizes gas. |

## Open Questions

1. Should `unitPrice` be fixed at session open or floating? Fixed is simpler but less flexible.
2. Should the oracle attest to units only (letting the contract compute amount) or to the final amount directly?
3. Is a standard `IMeteringOracle` interface worth defining, or is the oracle attestation format sufficient?

---

*This is a non-normative worked example. It demonstrates that ERC-8203 primitives are sufficient for metered billing without additional interfaces. If multiple independent implementations converge on similar recurring-lock semantics, a companion ERC may be warranted.*
