# Objective 1 — Characterize pairwise detection's topological limits

Question: a DeFiTail-style detector sees an exploit as **pairs** of contracts (attacker + one victim). How does its
detection performance change as the number of interacting contracts N grows (buckets 2, 3-5, 6-10, 11+)?

## Folder contents

| Path | What |
|---|---|
| `build_dataset.ipynb` | Sections 1-4: download DeFiHackLabs, fetch on-chain bytecode, compute N. Sections 5-8: compile attackers, resolve proxies, build pairwise data paths |
| `objective1.ipynb` | DeFiTail-style BERT + GCN, incident-level cross-validation, per-bucket metrics, degradation graph |
| `objective1_report.md` | Write-up of the results |
| `label_review.csv` | 50 incidents for manual label checking (fill in the last three columns) |
| `data/raw/DeFiHackLabs/` | Exploit PoCs (+ forge-std pinned to the repo's lock) |
| `data/raw/bytecode/<chain>/` | Runtime bytecode at the pre-attack block (`index.csv`, `proxies.csv`, one `.hex` per unique code) |
| `data/raw/attacker_bytecode/` | Compiled attacker contracts, one JSON per incident |
| `data/processed/dhl_incidents.csv` | One row per exploit: labels, chain, block, contracts, N, bucket |
| `data/processed/paths.csv` | The training data: one row per pairwise data path |
| `results/` | `degradation_auc.png` (main figure), `degradation.png` (F1), AUC/CI and difference tables, `metrics_*.csv`, `oof_predictions.csv`, `baseline_logreg_*`, per-fold checkpoints, `train.log` |

## Result (details in `objective1_report.md`)
- **Flash loan: pairwise detection degrades once N >= 3.** Incident-level AUC 0.92 at N=2 vs 0.65 at N>=3 for BERT+GCN
  (difference +0.28, 95% CI +0.15 to +0.39); the logistic-regression baseline shows the same drop (+0.13, CI +0.02 to +0.23).
- **Access control: no measurable effect of N** (differences +0.07 and +0.03, both CIs include 0; the raw F1 drop is a
  base-rate effect).
- The reproduced BERT+GCN is modest overall (AUC 0.59 / 0.69) and below the bag-of-opcodes baseline (0.66 / 0.82).

![Incident-level AUC by bucket](results/degradation_auc.png)

## 1. Data

### Sources (all public, no API key; downloaded 2026-10-05 to 2026-10-12)
1. **DeFiHackLabs**: https://github.com/SunWeb3Sec/DeFiHackLabs (Apache-2.0), commit `d0769ef0`. 869 real exploit PoCs.
2. **On-chain bytecode** via `eth_getCode` / `eth_getStorageAt` at the pre-attack block, through free public archive RPCs
   (eth.drpc.org, bsc-mainnet.public.blastapi.io, arbitrum-one.public.blastapi.io, base/polygon/optimism.drpc.org,
   api.avax.network). 6,197 address lookups, 0 errors; 4,780 are contracts.
3. **forge-std** (https://github.com/foundry-rs/forge-std, commit `33539934`) to compile the PoCs with solc via `py-solc-x`.
4. DeFiTail's own figshare artifact (https://doi.org/10.6084/m9.figshare.24117993) was checked and not used: it holds
   only code and a 91-file sample, not the paper's data.

### How N is computed
`n_contracts` = protocol contracts with code at the fork block + 1 attacker. Shared infrastructure (addresses used by 5+
incidents on the same chain: WETH, stablecoins, DEX routers, Aave pool) is excluded. Example: Euler (2023-03) = 4 protocol
contracts + attacker = 5 (7 with shared DAI and Aave pool).

### From incidents to data paths (DeFiTail's method, re-implemented)
1. **Attacker**: the PoC's own contracts compiled with solc (forge-std test helpers excluded). 753 of 765 compile; 12 fail on
   bugs in the upstream sources.
2. **Victims**: protocol contracts' on-chain bytecode; 330 of 417 proxies resolved to their implementation
   (EIP-1167 clones, EIP-1967/1822, legacy OpenZeppelin, storage slots 0-2).
3. **Paths**: per public function, a bounded DFS with a symbolic stack. The selector is fixed, so the dispatcher resolves; a
   path is kept only if every jump target resolves and the stack never underflows (DeFiTail's stack-height feasibility
   check) and execution ends in STOP/RETURN. Loops are unrolled at most ~4 times.
4. **Pairwise stitching**: at each CALL-family opcode, the latest pushed 4-byte constant the victim implements is taken as
   the called function and the victim's path is spliced in. Only one victim is stitched per path (DeFiTail's pairwise view).
5. Stack-only opcodes (PUSH/DUP/SWAP/POP/JUMPDEST) are dropped; at most 3,000 tokens per path and 32 paths per incident.

### Final dataset: 704 incidents, 10,248 paths

| Bucket (N) | Incidents | Access-control + | Flash-loan + | Paths | Incidents with a stitched path |
|---|---|---|---|---|---|
| 2 | 94 | 22 | 5 | 409 | 81% |
| 3-5 | 431 | 67 | 20 | 5,307 | 92% |
| 6-10 | 135 | 14 | 19 | 3,200 | 95% |
| 11+ | 44 | 4 | 7 | 1,332 | 98% |

61 of 765 incidents dropped: 12 compile failures, 49 with no feasible attacker path within the search limits.

## 2. Experiment
- **Model** (`objective1.ipynb`): BERT over each path + 2-layer GCN over the path/opcode graph (PPMI and TF-IDF edges),
  fused `0.5 * p_gcn + 0.5 * p_bert`; two binary tasks (access control, flash loan).
- **Protocol**: 5-fold cross-validation **by incident** (no incident in both train and test), stratified by bucket and label,
  repeated over 3 seeds. Main metric: incident-level ROC-AUC with bootstrap 95% CIs (threshold-free, so not distorted by
  each bucket's share of positives); F1, precision and recall also reported. Reference baseline: logistic regression on
  opcode TF-IDF, same folds.
- **Deviations from the paper** (CPU-only, 8 cores / 7.7 GB): BERT 2 layers / 128 hidden / 256 tokens (head + tail of the
  path) trained from scratch, instead of 4 layers / 512 tokens (benchmarked at ~60 h vs ~6 h for the full run); one encoder
  with two task heads; sparse graph where opcode nodes have learnable features and path nodes start at zero (a per-path
  embedding, i.e. TextGCN's identity features, left unseen test paths as random noise and scored at chance).

## 3. Caveats
- **Not comparable to 98.39% / 97.43%.** Different data (DeFiHackLabs vs DeFiTail's unreleased REKT paths), different label
  definitions, smaller model.
- **Labels are keyword heuristics** on DeFiHackLabs' vulnerability text. `label_review.csv` lists 50 incidents to check.
- **Few positives in some buckets** (e.g. 5 flash-loan incidents at N=2, 4 access-control at 11+): those F1 values are noisy.
- **Every incident is an exploit.** Negatives for each task are exploits of other types, as in DeFiTail's setup.
- **Static approximation of stitching**: callee identity is inferred from selectors, not executed; 50% of paths contain a
  stitch.

## 4. Re-running
`build_dataset.ipynb` is resumable (caches everything). `objective1.ipynb` reuses `results/folds/*.csv` checkpoints; delete
them to retrain.

---

# Objective 2 — Typed N-ary graph representation (same folder)

| Path | What |
|---|---|
| `objective2.ipynb` | Builds typed N-ary / pairwise graphs, representation analysis, R-GCN ablation, tests, figures |
| `objective2_report.md` | Write-up of Objective 2 |
| `data/processed/o2_graphs/` | Graphs for 704 exploits: `pairwise.npz`, `nary_protocol.npz`, `nary_all.npz`, `graph_stats.csv` |
| `results_o2/` | `representation_coverage.png`, `auc_by_bucket.png`, AUC/CI and test tables, predictions, fold checkpoints |

**Result** (details in `objective2_report.md`):
- **The representation works as designed.** N-ary graphs connect all contracts an exploit touches, with 7 edge types
  (control flow, state-changing call, read-only call, delegatecall, return, revert guard, storage dependency). At N >= 11 a
  graph sees ~5 contracts against 2 for the pairwise view (coverage 34% vs 14%).
- **Under a standard R-GCN it does not improve detection yet.** At equal training data, N-ary ties pairwise for both tasks.
  The pairwise view's flash-loan lead comes from having ~4x more training graphs. Typed edges do not help, and very large
  graphs (N >= 11) lose the flash-loan signal.
- These failure modes (dilution in large graphs, too little training data per graph, fragmented relation types) are what the
  Objective 3 architecture is meant to address.

![Objective 2 AUC by bucket](results_o2/auc_by_bucket.png)
