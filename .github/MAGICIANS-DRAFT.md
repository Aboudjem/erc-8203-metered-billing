# Draft reply for ERC-8203 Magicians thread #28041

**Where to post:** new reply in https://ethereum-magicians.org/t/erc-8203-agent-off-chain-conditional-settlement-extension-interface/28041

**Char count target:** under 2000 (Magicians etiquette)

**Humanizer pass:** done — no Tier 1/Tier 2 banned vocab, no em-dashes, no banned openers, prose paragraphs, casual technical tone

---

## Draft (paste this):

@xrqin promised this last week, here's the metered billing worked example as a non-normative profile.

Repo: https://github.com/Aboudjem/erc-8203-metered-billing
Spec: https://github.com/Aboudjem/erc-8203-metered-billing/blob/main/docs/profile.md

Short version: open a session by locking a max budget, stream signed meter receipts off-chain, settle in one tx using `settleConditional` with `proofType: ORACLE_ATTESTATION`. The metering oracle goes in the `verifier` field (the one we added in post #15). Sequence numbers prevent receipt replay. If the provider goes silent the client gets the whole lock back via `refundConditional` after expiry.

Three things I think are worth discussing:

1. The profile uses COMPOSITE for sessions that need both an oracle attestation and a timelock. Single-condition sessions could just use ORACLE_ATTESTATION directly. Both work, but I'm not sure which one belongs in the worked example. Currently I show COMPOSITE because it makes the dispute path explicit.

2. For batching across many concurrent sessions a provider serves, the profile uses `proofType: RECEIPT_ROOT` with merkle leaves of `(lockId, cumulativeAmount, oracleSignature)`. ~20x cheaper than per-session txs at 50 sessions. Worth showing this in the spec or keep it in the profile?

3. Open question I left in the doc: should the oracle attest to units only and let the contract compute amount, or attest to the final amount directly? Units-only is cleaner but locks the price model into the contract. Final-amount is more flexible but trusts the oracle with pricing.

The reference impl repo (https://github.com/Aboudjem/erc-8203-ref) doesn't change for any of this, the profile is purely additive on top of the existing interface.

@JackyWang the partial-claims question on batched settlements you raised back in #14: in this profile each leaf is its own settlement so the partial-claims question doesn't come up at this layer. If a session itself needs partial claims I think that's an extension worth discussing separately.

---

## Char count check
Use `wc -c` after stripping the metadata. The body is around 1700 chars.

## After posting
1. Update `data/contributions.json` via `erc_contribution_log` with `type: comment`, `erc: 8203`, the post URL, and a one-line summary
2. Mark opportunity 8203 as `done` (or keep `in-progress` if you want to track follow-ups)
3. Add bookmark on the post once it's up
