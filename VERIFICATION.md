# RogerAI - Model-Lineage Verification (the moat)

The headline product is **model-lineage tracking**: every token a buyer pays for carries a
verifiable record of *which model, at which precision, on which node* produced it - and an
honest token count. Not "provably" in a zero-knowledge sense (GPU non-determinism makes exact
re-execution impossible), but a **layered, statistical lineage guarantee** strong enough to
make cheating unprofitable and detectable. This is what no home-GPU competitor (shareai.now,
Hyperbolic, the AI Horde) has shipped.

## Why "lineage," not "provably"
- GPU inference is **non-deterministic** (FP non-associativity + non-batch-invariant kernels):
  1000 temp-0 requests can yield ~80 unique completions. So bitwise re-execution proofs don't
  exist on commodity hardware; zkML is hours-per-inference (research-only).
- Therefore the guarantee is **provenance + statistical attestation**, framed as **lineage**:
  *"this output's lineage traces to model X @ quant Q on node N, and the token count is honest."*

## The layered scheme (cheapest checks first)
1. **L0 - Signed, hash-chained usage receipts (SHIPPED in P0).**
   Node signs `{model, in/out tokens, price, prev_hash}`; broker verifies + counter-signs.
   The chain means a node can't silently drop/reorder/inflate requests without breaking it.
   Receipts are the billing source of truth + the audit trail.
2. **L1 - Independent token re-count (P1).** Broker recomputes prompt+completion tokens from
   the canonical tokenizer over the signed transcript - never trusts the node's counter.
   Bounds over-charging and under-reporting to the tokenizer's tolerance.
3. **L2 - Model/quant lineage attestation (P1, the differentiator).** Per request or sampled:
   - **TOPLOC** (Prime Intellect, open-source, ~1% prover overhead, validates ≤100× faster than
     inference): commit to a locality-sensitive hash of the model's activations; the verifier
     re-checks a few positions and detects model swap, quantization downgrade, or precision
     change with ~100% sensitivity on commodity GPUs.
   - **LOGIC-style logprob KS-test** (Inference.net): pin temp=0/fixed seed; node returns
     compressed top-k logprobs; verifier re-samples ~10 decode positions sub-second and runs a
     Kolmogorov-Smirnov test against the claimed model's distribution. Empirically catches even
     FP8 quant of the *same* model.
   Run on **all requests for new/low-rep nodes**, then **spot-check (PoSP)** trusted nodes at a
   low rate (<1% overhead) so honesty is a Nash equilibrium.
4. **L3 - Proof-of-GPU + weight binding (P2).** Periodic GPU-authenticity challenge (io.net PoW
   / Chutes GraVal style) + model-weight hash so a node can't quietly proxy to cheaper hardware
   or a different checkpoint. Optionally **encrypt work to the physical GPU**.
5. **L4 - Economic backing (P1/P2).** Bonded stake + escrow for new providers (released after a
   clean history); disputes freeze the affected requests (not the whole session); reputation
   tracks dispute rate, uptime, and latency honesty.

## Design rules (learned from competitors' mistakes)
- **Recompute against ground truth, never reward grader-agreement.** Bittensor's Yuma Consensus
  turned verification into a stake-weighted popularity contest (rewards correlate with stake
  ~0.8-0.95, with performance only ~0.1-0.3). A *central* RogerAI broker keeps checks objective.
- **Statistical, not exact.** Tolerance bands on token counts (±1-2%) and logprob distances;
  persistent drift (not one outlier) flags a node.
- **TEE attestation is the gold standard but kills the home-GPU thesis** (needs H100/H200 + TDX)
  → offer it only as a premium "datacenter-verified" tier, not the default.
- **Verification is the brand.** Surface a per-token "lineage verified" state in the CLI + site
  (the UI brief's shield-that-locks-on-settlement icon); let buyers `roger receipt verify <id>`.

## What P0 ships vs. what's next
- **P0 (now):** L0 - node-signed + broker-co-signed hash-chained receipts; cost from the upstream
  usage block; `lineage_method="p0-upstream-usage"`. Proven end-to-end against this box's LiteLLM.
- **P1:** L1 (independent re-count) + L2 (TOPLOC and/or logprob) wired into the broker's relay
  (`internal/protocol` already carries `LineageMethod`/`LineageProof` slots) + `roger receipt verify`.
- **P2+:** L3 GPU/weight binding, L4 stake/escrow/reputation, premium TEE tier.
