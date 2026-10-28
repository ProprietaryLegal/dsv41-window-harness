# DSV41 window harness: fenced, paired measurement windows for a production LLM server

How PLI Labs measures a candidate change on the same two DGX Spark nodes that serve production, without risking
production: open a HOLD, stop serving, alternate control and candidate starts, decide with a pre-registered rule, then
restore production and prove it with six greedy references before closing. This repo is the shared harness ("WIN2") that
every experiment runner on the DSV41 deployment adopts. Snapshot of the private `offload/win2` lane at `06b505a3656d`
(2026-10-27), after its third revision (WIN2-R3) landed; the full tree is `lane/win2` in
[dsv41-offload](https://github.com/ProprietaryLegal/dsv41-offload).

**Full source:** [dsv41-offload](https://github.com/ProprietaryLegal/dsv41-offload) · **Hub:** [pli-labs](https://github.com/ProprietaryLegal/pli-labs)

<!-- pli:blurb:start -->

> Published by [PLI Labs](https://proprietarylegal.ai/labs.html), the open research arm of [Proprietary Legal Intelligence](https://proprietarylegal.ai). PLI builds AI systems around a legal organization's own work product, combining retrieval, staged workflows, validation, and attorney review. Visit [proprietarylegal.ai](https://proprietarylegal.ai) to explore our work and discuss what your organization needs.

<!-- pli:blurb:end -->

## What is in `code/`

| File | Role |
|---|---|
| `offload/launch/windows/decision_v2.py` | the decision rule: per-cell WIN / LOSS / INCONCLUSIVE from repeated control and candidate starts (paired or Welch intervals, t quantiles, direction handling for lower-is-better metrics); guard metrics can only cause a LOSS |
| `offload/launch/windows/chain.py`, `chain_prev_ok.sh`, `chain_abort.sh` | chain mode: run windows back to back with production intentionally down in between, with a marker that fails closed, a gate that refuses to start against a serving production or a foreign HOLD, and an abort path that runs the strict restore |
| `offload/launch/windows/hold_admission.py`, `scoped_remote.py`, `spark_quiet.py`, `post_close_watch.py`, `production_health.py` | HOLD admission, fenced remote commands under owned systemd scopes, proof that both nodes are quiet, and the post-close health watch |
| `offload/launch/windows/greedy_kl.py`, `up2_checks.py`, `checks.py` | correctness gates: greedy references with a registered near-tie tolerance and KL bounds |
| `offload/launch/windows/win2_history.py` | append-only, hash-chained control archive and history re-scoring |
| `offload/gb10_budget/window_cells.py` | decision cells (prompt sizes, batch shapes, budgets) |
| `offload/launch/window.sh`, `start_exp.sh`, `serve_exp.sh` | opening a window and starting an experiment arm |
| `offload/launch/windows/smoke_start.py` | R3 smoke start: prove a candidate arm starts and serves (health, one greedy prompt, one seeded request whose speculative counters must move) before it costs a window |
| `offload/launch/windows/window_cap.py` | R3 two-window cap: a lane gets two counted windows; a third needs a new, dated hypothesis |
| `offload/launch/windows/resolution.py` | R3 resolution filter: refuses to queue a measurement whose registered expected gain is under twice the 95 % half-width its design can resolve (from the control archive, one prompt identity at a time) |
| `offload/launch/windows/dist_gate.py` | R3 distributional gate: top-20 next-token log-probabilities at fixed positions over 64 public-corpus prompts (short, 32K, 150K); token ids and log-probabilities only, no generated text |
| `offload/launch/windows/aa_pilot.py` | R3 seeded A/A pilot at the end of the restore: the same production configuration measured twice, to register the A/A band |
| `offload/gb10_budget/test_window_cells_seeded.py` | R3 seeded paired cells: per-request seeds, per-request counters, cold/warm prefix proof |
| `WIN2.md` | the rule in prose, with its measured no-effect reject and accept rates |

## Lessons the harness encodes

- **Measure where the arms differ.** A depth schedule (K5) was once "rejected" on single-user cells where it is identical
  to the control: an A/A comparison. Under the WIN2 rule the same data is INCONCLUSIVE, as it should be.
- **Guards must not make losing easy.** With secondary metrics counted as losses on a repeat test alone, a candidate with
  no real effect was rejected 16–33 % of the time and accepted 1.4–1.8 % (simulation). A guard now causes a LOSS only when
  its whole 95 % interval is below zero.
- **The loss spread must not grow with repeats.** A range over more control runs widens and weakens the test; the rule
  uses a fixed multiple of the control standard deviation.
- **Provenance per row.** Every archived control row carries the production commit, the clock cap of its own start, the
  date and the token budget, and spreads refuse to pool different clocks or commits.
- **Greedy near-ties.** Exact greedy identity is the restore gate. For drafting changes, BF16 near-ties flip at
  0.5/1.0-nat logit steps; the tolerance check locates the first differing token on the common prefix of the two answers.

## Status and tests

Built and unit-tested on CPU; used by the deployment's experiment runners. Revision 3 (seeded paired cells, smoke start,
two-window cap, resolution filter, distributional gate, A/A pilot) landed on 2026-10-27.

Tests: 391 passed, 45 failed or errored (distinct tests), 20 skipped (CPU only, no network). Of the failures: 39 also fail on the private source commit in the same sandbox (pre-existing: GPU, serving hosts, deployment state, git history a single-commit tree lacks, or an existing failure); 0 pass when their file runs alone; 4 need data on our hosts; 2 need a withheld prompt set or a file outside the data boundary. Per-branch results for the full tree are in `reports/tests/TESTS.md` of
[dsv41-offload](https://github.com/ProprietaryLegal/dsv41-offload) (`lane/win2`).

Host names are role names (`sparkhead` = rank 0, `sparkworker` = rank 1, `x86host` = the x86 offload host,
`controller` = the workstation running the harness). Our directories appear as neutral absolute paths under
`/opt/dsv41` (for example `/opt/dsv41/data`, `/opt/dsv41/campaigns/DSV41-OFFLOAD`); point them at your own locations.

## Licence

Apache-2.0 (`LICENSE`); third-party notices in `NOTICE` and `LICENSES/`.
