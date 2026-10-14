<!-- solve-problem:5e93886f-670e-4aba-9e14-b3ab88202108 -->
# Encrypted ordering with same-block user rebates

A [Tangle Solve problem](https://solve.tangle.tools/problems/5e93886f-670e-4aba-9e14-b3ab88202108).

## What is the problem?

**Setting.** In modern blockchains, validators/proposers extract *maximal extractable value* (MEV) by reordering, inserting, or censoring transactions. *Encrypted mempools* mitigate this: transactions are encrypted at submission, ordered while still ciphertext (e.g., by arrival time or commit-reveal), and decrypted only *after* the ordering is fixed.

**The tension.** Encryption kills not only harmful MEV (sandwich attacks, front-running) but also *benign* MEV: arbitrage, liquidations, and — critically here — **rebates**. In many designs (e.g., priority-fee auctions, order-flow auctions like MEV-Share), users or searchers receive a share of the value their transaction creates, paid *in the same block* (e.g., a refund in the same transaction bundle). Computing that rebate requires seeing the transaction contents; but under "decrypt-after-ordering," contents are unavailable until it's too late to condition payments on them.

**Precise problem.** Design a protocol Π in which, at block-building time:
1. Transactions {tx_i} are committed as ciphertexts with binding commitments;
2. The block order is fixed without knowledge of plaintexts;
3. An authorized party (block builder / smart contract) can compute a function f of the plaintexts — e.g., an MEV auction or rebate split — *inside* the encrypted domain, such that the resulting payments are included in the same block as the transactions that generated them;
4. Any unauthorized party (including the builder, before the auction completes) learns nothing about plaintexts beyond what the output legitimately reveals; and
5. Decryption of a transaction is possible *only* after ordering is finalized (binary decrypt-after-ordering is explicitly rejected as too coarse).

Formal tools implied: threshold/DCR encryption, MPC or FHE over the mempool, verifiable encryption, time-lock puzzles, and "programmable disclosure" primitives where *what* gets revealed, and to whom, is a computable function of the encrypted state.

**Ambiguity note.** The census does not fix the trust model (honest-majority threshold committee vs. MPC-with-builder vs. FHE-with-no-trust) or the exact rebate mechanism (pro-rata, first-price auction, VCG). A solver should pick and state one.

## Why does it matter?

A solution preserves the economic glue of MEV supply chains (rebates, backruns, refunds) while eliminating toxic extraction like sandwiching — currently the main argument *against* deploying encrypted mempools at all. Without it, chains adopting encryption (e.g., privacy-focused L2s, fair-ordering rollups) must either ban same-block value-sharing, degrading user experience and pushing flow to private order-flow auctions (OFAs), or accept leaks that reintroduce MEV. The problem sits at the intersection of the AFT/S&P encrypted-mempool literature and the order-flow-auction literature; resolution would make "fair ordering" economically neutral rather than value-destroying.

## What's already known?

- **Encrypted mempools / fair ordering.** Threshold-encrypted mempool designs (Shutter-style, threshold decryption with commit-reveal ordering) prove that ordering-then-decrypting eliminates reordering-based MEV; the SoK cited below is the canonical survey and explicitly identifies *suppression of legitimate auctions* as an open gap of the binary decrypt-after-ordering paradigm.
- **MEV redistribution.** MEV-Share / order-flow auctions (Flashbots) implement same-block rebates, but only in *plaintext* settings where the matcher sees order flow.
- **Private computation primitives.** Threshold FHE, MPC among a committee, and time-lock encryption each support computing on encrypted transactions, but published encrypted-mempool constructions use them only for decryption, not for computing conditioned payments. The gap — MPC/FHE *composed with* block building, with same-block settlement of outputs — appears open (verify against the SoK's open-problems section).
- **Why prior work falls short:** threshold decryption reveals everything at once to the builder; commit-reveal delays payments to a later block; pure FHE is too slow for block-latency auctions at mempool scale.

## What would count as solving it?

Any of the following, for a stated trust model and rebate rule f:
1. **Construction:** a protocol where f(plaintexts) is computed and settled in the same block as the encrypted transactions, with a simulation-based privacy definition (unauthorized parties learn nothing beyond the output), plus liveness/finality under stated adversary fractions.
2. **Impossibility:** a proof that same-block rebates necessarily leak enough information to permit reordering-based extraction (i.e., a fundamental privacy/expressivity tradeoff), which would redirect effort to next-block or commit-then-settle designs.
3. **Bound-characterization:** a precise frontier between what rebate functions f are realizable privately and which force leaks — e.g., showing sealed-bid rebates are fine but conditioning on full transaction semantics is not.
4. **Experiment:** a prototype (FHE or committee-MPC based) meeting block-time latency on realistic mempool loads, benchmarked against the SoK's framework.

## How to start

1. **Read the SoK first** (Passerat-Palmbach 2026), especially its taxonomy of disclosure timing and its open-problems section — the problem statement above is essentially that gap formalized. Cross-check that no 2025–2026 paper (Threshold decryption-based OFAs, "private order flow" proposals, Shutter/Flashbots research notes) already solves it.
2. **Pick a minimal instance:** sealed-bid MEV auction where searchers submit encrypted bids and the winner's payment is a same-block transfer. This avoids full semantics; formalize ideal functionality and check simulatability against a malicious builder.
3. **Natural approach:** threshold-encrypt user transactions; run a committee MPC (or threshold-FHE evaluation) between ordering-lock and block publication whose output is a signed payment authorization the builder must include — the obstacle is that the builder is adversary and also block author, so you need a mechanism forcing inclusion (e.g., decrypt-key release conditioned on a payment-carrying block commitment).
4. **Main obstacle:** circularity — the payment depends on plaintexts, the block depends on the payment, and decryption must not precede ordering. Time-lock puzzles or two-round MPC against block latency are the likely failure points; measure them early.

## Key references

- Passerat-Palmbach, *SoK: Encrypted Mempools Through the MEV Lens*, DOI:10.4230/LIPIcs.AFT.2026.22 — systematizes encrypted-mempool designs by their MEV properties and flags suppression of legitimate auctions under binary decrypt-after-ordering; the source of this open problem. (https://doi.org/10.4230/LIPIcs.AFT.2026.22 — verify.)
- Background (verify exact citations yourself): Shutter Network / threshold-encrypted mempool designs; Flashbots MEV-Share order-flow auction documentation; Boneh–Naor timed commitments / time-lock puzzles for delayed disclosure.

## Warnings

- The census record is a single secondary note ("fleet" source); the SoK's actual open-problems text may frame the gap differently — verify before claiming novelty.
- "Rebate" is ambiguous: payment in the *same block* vs. same *transaction* vs. next block. Same-transaction refunds may be strictly impossible without plaintext execution; state which you target.
- Do not assume the builder is honest-but-curious; the builder is the primary adversary and controls block contents.
- Watch for trivial non-solutions: decrypting only to an auction enclave (trusted hardware) shifts rather than removes trust; referees will ask for a cryptographic trust model.

---
*`BCGT-000318` · unverified*

## Acceptance guidance

Any of the following, for a stated trust model and rebate rule f:
1. **Construction:** a protocol where f(plaintexts) is computed and settled in the same block as the encrypted transactions, with a simulation-based privacy definition (unauthorized parties learn nothing beyond the output), plus liveness/finality under stated adversary fractions.
2. **Impossibility:** a proof that same-block rebates necessarily leak enough information to permit reordering-based extraction (i.e., a fundamental privacy/expressivity tradeoff), which would redirect effort to next-block or commit-then-settle designs.
3. **Bound-characterization:** a precise frontier between what rebate functions f are realizable privately and which force leaks — e.g., showing sealed-bid rebates are fine but conditioning on full transaction semantics is not.
4. **Experiment:** a prototype (FHE or committee-MPC based) meeting block-time latency on realistic mempool loads, benchmarked against the SoK's framework.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/5e93886f-670e-4aba-9e14-b3ab88202108?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000318.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000318)
