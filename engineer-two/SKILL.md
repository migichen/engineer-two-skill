---
name: engineer-two
description: Use when asked to autonomously improve on a SOTA(state-of-the-art) method (given problem + baseline code) toward a user-stated goal, producing a better method, verified codebase, and paper, with reviews scored against the user's goal.
---

# engineer2: ScientistTwo-style goal-driven research

Input: a scientific problem G (a SOTA paper and its code). Output: improved paper P+ and reproducible codebase C+.
Keep state across modules: `h_best`, `E_best` (results), `C_best` (code), `E_base`/`C_base` (baseline), the trace log `R` of every idea tried, and `E_abl`.
Use subagents for the roles named below where useful.

## Pipeline
0. Setup
- Versioning (applies throughout)
1. Limitations
2. Seed Ideas
3. Idea Evaluation
4. Idea Refinement
5. Ablation
6. Drafting & Review-Rebuttal
7. Meta-Review

## Module: Setup
Before starting, ask the user these questions in a single message. Show the suggested value with each one, and use it if the user accepts:
- Goal: what are you trying to achieve with this work (e.g. target metric/threshold, deployment constraint, speed/memory budget, intended use)? (no suggestion; the user must supply this). Save the answer verbatim to `goal.md` at project root, and re-read it whenever a Goal rating is to be made.  Maintain this file in concise instruction bullets whenever user provides more details of the goal.
- Input G: which SOTA paper file and baseline repo path? (no suggestion; the user must supply these)
- N_seed (seed pool size)? Suggest 8.
- N0 (seed ideas evaluated in round 0)? Suggest 2.
- Subset: which part of the benchmark? Suggest the 1-2 smallest datasets on the primary metric, using under ~10% of full-run compute.
- Good/Bad rule: Good = beats `E_base` on most subset datasets on the primary metric; Bad = worse on most of them; otherwise Engineer. OK?
- "Better" for the verification gates: prefer `E_new` on the primary metric averaged across datasets, with no dataset regressing beyond noise. OK?
- N_p (ablation plans)? Suggest one per component (3-6). N_t (rebuttal tasks)? Suggest ≤3.
- Output location? Suggest `research_out/` (R.jsonl, E_*.json, code/, paper/paper.md + paper/figures/; write FAILURE.md if the run terminates).

Finally, store all the above project setup except already in `goal.md` into `project_setup.md` at project root and re-read it if you miss the context.  Be concise and only bulleted instructions. Update this file when more policies are decided by either user or yourself, like data file paths.

## Module: Versioning
Applies throughout. Give each idea an `<idea_id-codename>`, e.g. `07-kernel-earlystop` (a sequential id plus a short slug).
Before starting a new version, **copy** (snapshot) the current version into `<output folder>/<idea_id-codename>/v<n>/`. A snapshot holds the idea text, results, code, the whole `paper/` directory (including `figures/`) and the review. Number `v<n>` per idea, and start a new one for each refinement or paper draft. Engineer reruns stay in the same version. Paper drafts go under `h_best`'s id.
Never overwrite a snapshot. Snapshot rejected versions too, and log their paths in `R`. If a refinement fails a gate, restore the working tree from the best snapshot.
When conceiving a new idea, scan these folders to avoid repeating yourself.

## Module: Limitations
1. As the agent **Limitation Extractor**, list the limitations of the SOTA method on G.
2. As the agent **Limitation Verifier**, judge whether the list is enough to guide novel improvements.
3. If it is not enough, extract the missing limitations and verify again. Stop when the verifier confirms all actionable limitations are covered, or after 16 extract→verify cycles (the first one counts).

## Module: Seed Ideas
Save untested ideas into seeds.md.
1. Write an initial idea h0 that targets the limitations.
2. As the agent **Promising Checker**, score its potential success rate against the goal. Retrieve 2 related papers via web search and compare against them.
3. As the agent **Idea Generator**, add distinct ideas to deal with the limitations until you have N_seed candidates.
4. Sort the seed pool H0 by the potential success rate against the goal, highest first.

## Module: Idea Evaluation
Treat this as one callable unit, `Coder(G, h) -> (h, E, C, decision, feedback)`, run by another agent. 
1. **Baseline** (run once, before Idea Refinement; every `Coder` call reuses it): reproduce G's main experiments on a representative benchmark subset to get `E_base` and `C_base`.
2. **Subset run**: implement h by modifying `C_base`, then run it on the subset.
3. **Subset Critic**: compare against `E_base` and decide, applying Setup's Good/Bad rule:
   - `Bad`: worse than the baseline. Discard the idea.
   - `Good`: better than the baseline. Approve it for scale-up.
   - `Engineer`: neither; it needs tuning or code fixes. As the agent **Subset Engineer**, apply the critic's feedback, rerun, and send it back to the critic.
4. Allow at most N_eng=2 engineer+rerun cycles after the first critic call. If the idea is still not `Good` after that, mark it `Bad` and prune it.
5. **Full-set**: for `Good` ideas, adapt the code to the full benchmark (all datasets and metrics). Run `C_base` on the full benchmark once (or use G's reported numbers) as the reference. A Full-Set Critic decides `Good`/`Bad`/`Engineer` by the same rule, and the Full-Set Engineer fixes the idea for at most N_eng cycles. Its final decision is `Good` or `Bad`.

## Module: Idea Refinement
1. Round 0: run `Coder` on the top-N0 seed ideas and log every trace in `R`.
2. Round k≥1: as the agent **Idea Evolver**, read all earlier traces (Good results and Bad failure logs) and propose N_k evolved ideas.
3. For exploration, add the next N_e unevaluated seed ideas from H0, in novelty order. Default: N_k=1 evolved and N_e=1 seed idea per round.
4. Run `Coder` on each candidate and append the results to `R`.
5. Stop once S=4 ideas have a final (full-set) decision of `Good`, or after K=4 rounds in total (rounds 0..3). If no idea is `Good` by round K, **terminate the whole pipeline** and report failure.
6. As the agent **Selector**, compare full-benchmark metrics and logs across all `Good` ideas. Set `h_best`, `E_best`, `C_best` to the best one.

## Module: Ablation
1. As the agent **Ablation Planner**, write N_p executable component-level ablation plans for `h_best`.
2. Run each plan by modifying `C_best`. Collect the results as `E_abl`.
3. As the agent **Ablation Critic**, inspect the component breakdown and decide `Good` (optimal) or `Refine`, with a critique.
4. On `Refine`, act as the Full-Set Engineer agent: use the critique (e.g. drop redundant or harmful components) to produce `h_new`, `E_new`, `C_new`.
5. **Verification gate**: as the agent **Result Comparison** role, replace the best state with the new one only if `E_new` is strictly better than `E_best`. If you replace it, redo the ablation planning and runs on the new `h_best`, for reporting only; this cannot trigger a further refinement once N_abl is spent. If it fails, restore the best snapshot.
6. Repeat until the critic says `Good`, or for at most N_abl=1 refinement.

## Module: Drafting & Review-Rebuttal
1. As the agent **Initial Drafter**, combine `h_best`, `E_best`, and `E_abl` into a full manuscript, `paper/paper.md`, in Markdown. Save figures as PNG files in `paper/figures/` and link them with relative paths, e.g. `![Fig 1](figures/fig1.png)`.
2. As the agent **Goal Reviewer**, read `goal.md` and the paper. Write strengths, weaknesses, and questions with respect to the goal, and give a **Goal rating** from 1 to 10: how well the method, results, and paper achieve the user's goal (10 = fully achieved and convincingly demonstrated; 8 = achieved with minor gaps; 5 = partially achieved; 1 = not addressed). Justify the rating against each part of the goal.
3. If the Goal rating is below 8: as the **Rebuttal Planner**, turn the review's goal gaps into N_t supplementary experiment tasks. As the agent **Rebuttal Coder**, run each task on `C_best` to get `E_reb`.
4. As the agent **Paper Enhancer**, fold the review and `E_reb` into the paper by revising claims, tables, and figures. Then review it again.
5. Repeat until the Goal rating is at least 8, or after at most N_peer=2 Reviewer calls in total (the first one counts).

## Module: Meta-Review
1. As the agent **Meta-Reviewer**, read `goal.md`, the paper, and the latest review (with its Goal rating). Decide `Accept` or `Refine`, with a meta-critique.
2. On `Accept`: export, i.e. leave the final `paper/paper.md` + `paper/figures/` and `code/` (= `C_best`) in the output folder. Done.
3. On `Refine` (a critical algorithmic or empirical weakness, or a major part of the goal not met): act as the Full-Set Engineer and use the meta-critique to produce `h_new`, `E_new`, `C_new`.
4. **Verification gate**: if `E_new` is strictly better than `E_best`, update the best state, then rerun Ablation (reporting only, no new refinement) and Drafting & Review-Rebuttal. Otherwise discard the change, restore the best snapshot, and export; the run ends. On any exit other than Accept, export P+ = the latest paper built from `C_best`, and C+ = `C_best`.
5. Repeat until `Accept`, or for at most N_meta=1 refinement. After that refinement, export without running another meta-review.
