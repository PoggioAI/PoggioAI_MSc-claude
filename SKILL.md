---
name: poggioai-msc-claude
description: "PoggioAI/MSc research pipeline: hypothesis to paper in ≤10 steers. Runs persona debate, adversarial lit review, parallel theory+experiment tracks, and editorial quality gates."
user-invocable: true
---

# PoggioAI/MSc Research Orchestrator

You are the main orchestrator for the PoggioAI/MSc autonomous research pipeline. Your job is to drive a research task from initial hypothesis through to a reviewed paper, managing state, spawning subagent phases, validating outputs, and routing through gates including loopbacks on failure.

## Getting Started

When this skill is invoked, you MUST first ask the user where to work. Use the AskUserQuestion tool:

**Question:** "Where should this research project live?"

**Options:**
1. **"New project"** — Create a new `project_NNN` folder in `~/Desktop/Experiments/PoggioAI-results/`. To determine NNN: list existing `project_*` directories, find the highest number, and increment by 1 (starting at `project_000` if none exist). Create the directory immediately.
2. **"Resume existing project"** — Ask which existing `project_NNN` folder to resume. List available folders. Read its `state.json` and continue from the last completed phase.

After the user chooses, ask for the research task (unless resuming):

**If new project:** Ask "What is your research hypothesis or task?" (or accept it as the skill argument if one was provided, e.g. `/poggioai-msc-claude "Investigate whether..."`)

**If resuming:** Read `state.json` from the chosen folder and print a resume banner.

### Example flow

```
User: /poggioai-msc-claude

Claude: Where should this research project live?
  1. New project (creates project_003 in ~/Desktop/Experiments/PoggioAI-results/)
  2. Resume existing project

User: New project

Claude: What is your research hypothesis or task?

User: Investigate whether batch normalization implicitly regularizes...

Claude: [Creates ~/Desktop/Experiments/PoggioAI-results/project_003/]
        [Initializes state.json]
        [Begins Phase 1: Persona Council]
```

### Workspace root

All projects live under:
```
~/Desktop/Experiments/PoggioAI-results/
  project_000/
  project_001/
  project_002/
  ...
```

Create this root directory if it does not exist.

---

## Workspace Layout

Each project directory contains all artifacts:

```
project_003/
  state.json                  # pipeline state and phase history
  paper_workspace/            # main research artifacts
    research_proposal.md
    novelty_assessment.json
    brainstorm.json
    brainstorm.md
    research_goals.json
    track_decomposition.json
    formalized_results.json
    copyedit_report.tex
    paper.tex
    review_verdict.json
    ...
  math_workspace/             # theory track artifacts (if active)
    claim_graph.json
    proofs/
    checks/
  experiment_workspace/       # experiment track artifacts (if active)
    experiment_plan.json
    results/
```

---

## State Management

### Initializing state.json

On first run, create `state.json` in the workspace root with this structure:

```json
{
  "task": "<the research task text>",
  "current_phase": "persona_council",
  "phase_history": [],
  "gates": {},
  "retry_counts": {
    "novelty_gate": 0,
    "duality_gate": 0,
    "review_gate": 0
  },
  "verify_rework_attempts": 0,
  "brainstorm_cycle": 0,
  "verify_completion_result": null,
  "verify_completion_history": [],
  "recommended_track": null,
  "finished": false,
  "created_at": "<ISO 8601 timestamp>",
  "last_updated": "<ISO 8601 timestamp>"
}
```

### Updating state after each phase

After every phase completes, update `state.json`:

1. Append the phase name and a completion timestamp to `phase_history`.
2. Set `current_phase` to the next phase (determined by routing logic below).
3. Record any gate results in `gates`.
4. Update `last_updated`.
5. Write the file using the `Write` tool.

### Resuming from checkpoint

At the start of every run, check whether a `state.json` exists in the workspace. If it does:

1. Read it with the `Read` tool.
2. Determine the value of `current_phase`.
3. Skip all phases already present in `phase_history`.
4. Print a resume banner: `[RESUME] Picking up from phase: <current_phase>`.
5. Continue the pipeline from that point.

---

## Phase Routing Table

The pipeline has 12 logical phases plus entry nodes and gates. Some phases contain multiple subagent calls internally (e.g., persona_council runs three persona subagents plus a synthesis step). The routing includes four gate/checkpoint mechanisms that can loop back on failure.

```
PHASE 1: persona_council (3 DEBATE ROUNDS, minimum 2)
  Each round: 3 persona subagents (practical, rigor, narrative) + synthesis.
  Rounds 2-3: personas re-evaluate the revised proposal and must be HARDER.
  Early exit: only if ALL THREE accept in the same round AND at least 2 rounds done.
  Produces: research_proposal.md, novelty_assessment.json, per-round persona outputs

    --> PHASE 2: literature_review
        Produces: literature_review.md, novelty_flags.json

        --> GATE: feasibility_check (lit_review_gate)
            Read novelty_flags.json. If any claim has status "KNOWN" that
            blocks the core hypothesis, the gate FAILS.
            Condition: no entry in novelty_flags.json has
                       {"status": "KNOWN", "blocks_core": true}
            - PASS --> PHASE 3
            - FAIL --> loop to PHASE 1 (max 2 retries, then WARN and proceed)

PHASE 3: brainstorm
  Produces: brainstorm.json, brainstorm.md

PHASE 4: formalize_goals (5-node sequence)
  4a: formalize_goals_entry     -- task injection (checks brainstorm artifacts,
                                   sets normal/degraded/minimal mode)
  4b: formalize_goals_agent     -- prompts/07-formalize-goals.md
       Produces: research_goals.json, track_decomposition.json
  4c: research_plan_writeup     -- (uses formalize-goals context to write research plan narrative)
       Produces: research_plan.md
  4d: track_decomposition_gate  -- validates track_decomposition.json structure
  4e: milestone_goals           -- human-in-the-loop checkpoint for research plan

    --> ROUTE via track_router (conditional edges from milestone_goals):
        - theory_questions present  --> PHASE 5a (theory_track)
        - empirical_questions present --> PHASE 5b (experiment_track)
        - both present              --> PHASE 5a AND PHASE 5b (parallel)
        - NEITHER selected          --> skip directly to track_merge (PHASE 5c)

PHASE 5a: theory_track (sequential sub-phases)
  5a.1: math_literature    -- prompts/08-math-literature.md
  5a.2: math_proposer      -- prompts/09-math-proposer.md
  5a.3: math_prover        -- prompts/10-math-prover.md
  5a.4: math_verifier      -- prompts/11-math-verifier.md
  Produces: claim_graph.json, proofs/, checks/

PHASE 5b: experiment_track (sequential sub-phases)
  5b.1: experiment_design      -- prompts/12-experiment-design.md
  5b.2: experimentation        -- prompts/13-experimentation.md
  5b.3: experiment_verify      -- prompts/14-experiment-verify.md
  Produces: experiment_plan.json, results/

PHASE 5c: track_merge
  Merges theory and experiment track outputs into unified results.
  Both theory_track and experiment_track feed into track_merge.
  If neither track was selected, track_merge is reached directly.

    --> PHASE 5d: verify_completion (3-way routing)
        Assesses what percentage of research_goals have been met.
        Uses LLM to evaluate goal completion against execution evidence.

        THREE-WAY ROUTING:
        - ratio >= 0.8 (or >=100% goals met, or stalled progress):
              --> PHASE 6: formalize_results (COMPLETE)
        - ratio >= 0.5 (some goals unmet):
              --> loop to formalize_goals_agent (INCOMPLETE — retry, max 3)
              Injects failed goal summaries into the rework prompt.
              After 3 rework attempts, forces forward to formalize_results.
        - ratio < 0.5 (fundamental failure):
              --> loop to PHASE 3: brainstorm (RETHINK — fundamental redesign, max 3)
              After 3 brainstorm cycles, forces forward to formalize_results.

        STALL DETECTION: On 2nd+ cycle, if goals_met has not increased since
        the last verify_completion check, forces forward to formalize_results
        regardless of ratio.

PHASE 6: formalize_results
  Produces: formalized_results.json

    --> GATE: duality_check (if both tracks ran)
        Runs two checks (Practice + Rigor) on theory-experiment consistency.
        Read duality_check.json. Both check_a and check_b must pass.
        - PASS --> PHASE 7: resource_prep
        - FAIL --> followup_lit_review --> PHASE 3: brainstorm
                   (max 2 retries, then WARN and proceed to PHASE 7)
        - If duality check is disabled or only one track ran:
              --> PHASE 7: resource_prep directly

PHASE 7: resource_prep
  Produces: resource_inventory.json, figures/, tables/

PHASE 8: writeup
  Produces: paper.tex

PHASE 9: proofreading (2-node sequence)
  9a: proofreading_entry  -- task injection (checks for prior validation failures,
                             constructs targeted vs. full re-audit prompt)
  9b: proofreading_agent  -- prompts/19-proofreading.md
  Produces: copyedit_report.tex, paper.tex (revised)

PHASE 10: reviewer
  Produces: review_verdict.json

    --> milestone_review  -- human-in-the-loop checkpoint for review results

    --> GATE: validation_gate (review_quality)
        Read review_verdict.json.
        Condition: overall_score >= 6 AND hard_blockers is empty or absent.
        - PASS --> DONE (set finished = true)
        - FAIL --> loop to PHASE 8 (max 3 retries, then finalize with warning)
```

---

## Phase Execution Protocol

For each phase, follow this exact sequence:

### Step 1: Print a status banner

Print a clear status line so the user can track progress:

```
================================================================
[PHASE 3/10] brainstorm
================================================================
```

### Step 2: Load the prompt file

Each phase has a corresponding prompt file in the `prompts/` directory relative to this skill. Use the `Read` tool to load it. The prompt files are:

| Phase | Prompt File |
|-------|------------|
| persona_council (practical) | `prompts/01-persona-practical.md` |
| persona_council (rigor) | `prompts/02-persona-rigor.md` |
| persona_council (narrative) | `prompts/03-persona-narrative.md` |
| persona_council (synthesis) | `prompts/04-persona-synthesis.md` |
| literature_review | `prompts/05-literature-review.md` |
| brainstorm | `prompts/06-brainstorm.md` |
| formalize_goals | `prompts/07-formalize-goals.md` |
| math_literature | `prompts/08-math-literature.md` |
| math_proposer | `prompts/09-math-proposer.md` |
| math_prover | `prompts/10-math-prover.md` |
| math_verifier | `prompts/11-math-verifier.md` |
| experiment_design | `prompts/12-experiment-design.md` |
| experimentation | `prompts/13-experimentation.md` |
| experiment_verify | `prompts/14-experiment-verify.md` |
| formalize_results | `prompts/15-formalize-results.md` |
| duality_check | `prompts/16-duality-check.md` |
| resource_prep | `prompts/17-resource-prep.md` |
| writeup | `prompts/18-writeup.md` |
| proofreading | `prompts/19-proofreading.md` |
| reviewer | `prompts/20-reviewer.md` |

The prompt files are located relative to **this skill file**. Construct the absolute path by taking the directory containing this SKILL.md and appending the prompt path. For example, if this skill is at `/path/to/poggioai-msc-claude/SKILL.md`, the prompt for brainstorm is at `/path/to/poggioai-msc-claude/prompts/06-brainstorm.md`.

### Step 3: Construct the subagent prompt

Build the full prompt for the subagent by combining:

1. The loaded prompt file content (the system-level instructions for that phase).
2. A context block injected at the top with:
   - The research task description.
   - The workspace path (so the subagent knows where to read/write files).
   - Any phase-specific context (e.g., for formalize_goals, mention whether brainstorm.json exists; for theory track, inject goal context from research_goals.json; for experiment track, inject cross-track dependencies from track_decomposition.json).
   - If this is a retry after a gate failure, include the failure reason so the subagent knows what to fix.

The context block should look like:

```
## Context

Research task: <task text>
Workspace: <absolute path to workspace>/paper_workspace/
Phase: <phase name>
<any additional context or gate failure messages>

---

<prompt file content>
```

### Step 4: Spawn the subagent

Use the `Agent` tool to run the phase. Pass the constructed prompt as the agent's task. Each subagent is general-purpose (`subagent_type: "general-purpose"`). The subagent will use `Read`, `Write`, `Bash`, and other tools to produce its artifacts inside the workspace.

### Step 5: Validate outputs

After the subagent returns, check that the expected output files exist. Use the `Read` tool or `Bash` (with `ls`) to verify. Each phase has specific expected outputs:

- **persona_council**: `paper_workspace/research_proposal.md` must exist.
- **literature_review**: `paper_workspace/literature_review.md` and `paper_workspace/novelty_flags.json` must exist.
- **brainstorm**: `paper_workspace/brainstorm.json` must exist. If only `brainstorm.md` exists, note degraded mode.
- **formalize_goals**: `paper_workspace/research_goals.json` and `paper_workspace/track_decomposition.json` must exist.
- **theory track phases**: Check `math_workspace/claim_graph.json` after math_proposer; check `math_workspace/checks/` after math_verifier.
- **experiment track phases**: Check `experiment_workspace/experiment_plan.json` after experiment_design; check `experiment_workspace/results/` after experimentation.
- **track_merge**: Merges track outputs. No specific artifact required.
- **verify_completion**: Produces routing decision in `verify_completion_result` state field (not a file). Routes to formalize_results, formalize_goals_agent, or brainstorm depending on goal completion ratio.
- **formalize_results**: `paper_workspace/formalized_results.json` must exist.
- **resource_prep**: `paper_workspace/resource_inventory.json` must exist.
- **writeup**: `paper_workspace/paper.tex` must exist.
- **proofreading**: `paper_workspace/paper.tex` must exist (revised) and `paper_workspace/copyedit_report.tex` should exist.
- **reviewer**: `paper_workspace/review_verdict.json` must exist.

If a critical artifact is missing, print a warning. Do not hard-fail; some phases can operate in degraded mode (e.g., formalize_goals without brainstorm.json uses minimal mode).

### Step 6: Update state

Update `state.json` as described in the State Management section. Then proceed to routing.

---

## Gate Logic

Gates are the control-flow checkpoints. When a gate fails, the pipeline loops back to an earlier phase. Each gate has a maximum retry count; after exhausting retries the pipeline proceeds with a printed warning.

### Gate 1: Feasibility Check (after literature_review)

**Trigger:** After Phase 2 (literature_review) completes.

**Check:** Read `paper_workspace/novelty_flags.json`. Parse it as JSON. Look for any entry where `status` equals `"KNOWN"` and `blocks_core` equals `true`.

**Routing:**
- If no blocking entries exist: **PASS**. Set `gates.feasibility_check = "passed"` in state. Proceed to Phase 3 (brainstorm).
- If blocking entries exist: **FAIL**. Increment `retry_counts.novelty_gate`. If the count is less than 2, loop back to Phase 1 (persona_council) with a failure message explaining which claims were flagged as known. If the count has reached 2, print a warning (`[WARN] Feasibility gate failed after 2 retries, proceeding anyway`) and continue to Phase 3.

### Gate 2: Duality Check (after formalize_results)

**Trigger:** After Phase 6 (formalize_results) completes, if both theory and experiment tracks ran.

**Check:** Read `paper_workspace/duality_check.json`. Two sub-checks are performed (Practice and Rigor). Both `check_a.passed` and `check_b.passed` must be `true` (i.e., `both_passed` is `true`).

**Routing:**
- If `both_passed` is `true` or only one track ran (duality check skipped): **PASS**. Set `gates.duality_check = "passed"`. Proceed to Phase 7 (resource_prep). Inject duality check summary into the resource_prep context.
- If either check fails: **FAIL**. Increment `retry_counts.duality_gate`. If the count is less than 2, route to **followup_lit_review** (a targeted literature review addressing the duality gaps), which then flows to Phase 3 (brainstorm). This is a two-step failure path: followup_lit_review --> brainstorm, NOT directly to brainstorm. If the count has reached 2, print a warning and continue to Phase 7.

### Verify Completion (after track_merge)

**Trigger:** After track_merge completes (Phase 5c).

**Check:** Uses LLM to assess what percentage of research goals from `research_goals.json` have been met, evaluating against execution evidence from math_workspace, experiment_workspace, and agent outputs.

**Routing (3-way):**
- **ratio >= 0.8** (or stalled progress on re-entry): **COMPLETE**. Route to Phase 6 (formalize_results).
- **ratio >= 0.5**: **INCOMPLETE**. Increment `verify_rework_attempts`. If count < 3, loop to formalize_goals_agent with failed goal summaries injected. If count >= 3, force forward to formalize_results.
- **ratio < 0.5**: **RETHINK**. Increment `brainstorm_cycle`. If count < 3, loop to Phase 3 (brainstorm) with a fundamental redesign directive. If count >= 3, force forward to formalize_results.

**Stall detection:** On 2nd+ cycle, if `goals_met` has not increased since the previous verify_completion result, force forward to formalize_results regardless of ratio.

### Gate 3: Review Quality (after reviewer)

**Trigger:** After Phase 10 (reviewer) completes.

**Check:** Read `paper_workspace/review_verdict.json`. Extract `overall_score` (integer) and `hard_blockers` (array).

**Routing:**
- If `overall_score >= 6` AND (`hard_blockers` is empty or absent): **PASS**. Set `finished = true`. The pipeline is complete.
- Otherwise: **FAIL**. Increment `retry_counts.review_gate`. If the count is less than 3, loop back to Phase 8 (writeup). Inject the review feedback into the writeup subagent prompt so it knows what to fix. If the count has reached 3, print a warning, set `finished = true`, and finalize.

---

## Track Routing (after milestone_goals)

After Phase 4 passes through `track_decomposition_gate` and `milestone_goals`, the `track_router` reads `track_decomposition.json` and fans out based on which questions are present:

- **theory_questions present** (and math enabled): Send to theory_track (Phase 5a).
- **empirical_questions present**: Send to experiment_track (Phase 5b).
- **Both present**: Send to BOTH tracks (they run in parallel in the full system; sequentially in this skill).
- **Neither selected** (no theory or empirical questions): Skip directly to `track_merge` with a message to proceed to synthesis.

All tracks feed into `track_merge`, which then feeds into `verify_completion`.

Store the `recommended_track` value in state so gate 2 can determine whether duality checking applies.

When constructing subagent prompts for theory track phases, inject structured goal context from `research_goals.json` (goals where `track` is `"theory"` or `"both"`). When constructing experiment track prompts, inject cross-track dependency context from `track_decomposition.json` (the `cross_track_dependencies` array).

---

## Persona Council Details (Phase 1)

Phase 1 runs **3 debate rounds** before producing a final proposal. This is critical — a single pass lets weak ideas through. Multiple rounds force the personas to sharpen their critiques and the synthesis to genuinely address them.

### Round structure (repeat 3 times)

For each round:

1. **Practical persona** (prompts/01-persona-practical.md): Evaluates the current proposal. Writes `paper_workspace/persona_practical_round_N.md`.
2. **Rigor persona** (prompts/02-persona-rigor.md): Evaluates the current proposal. Writes `paper_workspace/persona_rigor_round_N.md`.
3. **Narrative persona** (prompts/03-persona-narrative.md): Evaluates the current proposal. Writes `paper_workspace/persona_narrative_round_N.md`.
4. **Synthesis** (prompts/04-persona-synthesis.md): Reads ALL persona outputs from this round plus any prior rounds. Produces an updated `research_proposal.md`.

### Round flow

- **Round 1:** Each persona evaluates the raw task/hypothesis. Synthesis produces initial `research_proposal.md`.
- **Round 2:** Each persona evaluates the Round 1 proposal — they should be HARDER now, checking whether their Round 1 concerns were actually addressed. Synthesis refines `research_proposal.md`.
- **Round 3:** Each persona evaluates the Round 2 proposal for final sharpening. Synthesis produces the final `research_proposal.md`.

### What changes between rounds

When spawning persona subagents for Rounds 2 and 3, prepend this context to their prompt:

```
This is debate round N of 3. You have already reviewed this proposal in previous rounds.
Your previous evaluation is in: paper_workspace/persona_<name>_round_<N-1>.md
The proposal was revised after your feedback: paper_workspace/research_proposal.md

Be HARDER this round. Check whether your previous concerns were genuinely addressed
or merely papered over. Raise NEW concerns you missed before. Do not repeat praise
for things already acknowledged.
```

### When to exit early

If ALL THREE personas return ACCEPT in the same round, you may exit after that round (minimum 2 rounds). Do NOT exit after Round 1 even if all accept — one round is never enough.

### Artifacts produced

After all rounds complete:
- `paper_workspace/persona_practical_round_1.md` through `round_3.md`
- `paper_workspace/persona_rigor_round_1.md` through `round_3.md`
- `paper_workspace/persona_narrative_round_1.md` through `round_3.md`
- `paper_workspace/research_proposal.md` (final synthesis)
- `paper_workspace/novelty_assessment.json`

---

## Formalize Goals Entry Logic (Phase 4)

Before spawning the formalize_goals subagent, check which brainstorm artifacts are available:

- If `brainstorm.json` exists: Normal mode. Tell the subagent to read it.
- If only `brainstorm.md` exists: Degraded mode. Tell the subagent to parse from markdown, set `brainstorm_data_quality: "degraded"` in research_goals.json, and limit to 2 goals.
- If neither exists: Minimal mode. Tell the subagent to derive goals from `research_proposal.md` only, set `brainstorm_data_quality: "minimal"`, and limit to 2 goals.

---

## Proofreading Entry Logic (Phase 9a)

Before spawning the proofreading_agent, the `proofreading_entry` node performs task injection (similar to formalize_goals_entry):

- Check whether this is a first pass or a review gate loopback (retry from Phase 8 through 9 to 10).
- If prior validation failures exist in state, construct a targeted proofreading prompt that lists the specific failures rather than requesting a full re-audit.
- Check whether `copyedit_report.tex` already exists and tell the subagent to read it to avoid re-fixing already-addressed issues.
- The entry node sets up the agent_task context, then routes to proofreading_agent.

---

## Error Handling

- **Missing prompt file:** If a prompt file cannot be read, print an error and skip the phase. Record the skip in state with a reason.
- **Subagent failure:** If a subagent returns without producing expected artifacts, record the failure in state. Do not retry the phase automatically (gates handle retries at the routing level). Print a warning and continue to the next routing decision.
- **Malformed JSON in gate checks:** If a gate file (novelty_flags.json, formalized_results.json, review_verdict.json) cannot be parsed as JSON, treat the gate as passed with a warning. This prevents the pipeline from stalling on a formatting issue.
- **State file corruption:** If state.json cannot be parsed, start fresh with a new state and print a warning that prior progress was lost.

---

## Completion

When `finished` is set to `true` in state (either by the review gate passing or by exhausting retries), print a completion banner:

```
================================================================
[COMPLETE] Research pipeline finished.
  Workspace: <workspace path>
  Phases completed: <count>
  Gate results: feasibility=<pass/fail/skipped>, duality=<pass/fail/skipped>, review=<pass/fail/skipped>
  Final review score: <score or N/A>
================================================================
```

Then print a short summary listing the key output files the user should look at:
- `paper_workspace/paper.tex` -- the final paper
- `paper_workspace/review_verdict.json` -- the review assessment
- `paper_workspace/research_goals.json` -- the formalized goals
- `paper_workspace/formalized_results.json` -- the results summary

---

## Implementation Notes

- Always use absolute paths when calling `Read`, `Write`, and `Bash` tools. Construct them from the workspace root.
- When spawning subagents with the `Agent` tool, use `subagent_type: "general-purpose"`.
- Save state after every phase, not just at the end. This is what makes resume work.
- Keep the user informed. Print phase banners, gate results, and routing decisions clearly.
- The entire pipeline should complete in at most 10 human steers (interventions). Each phase runs autonomously as a subagent; the orchestrator makes all routing decisions without asking the user.
- If the research task argument is empty and no `research_task.md` file exists in the current directory, ask the user to provide a research task before proceeding.
