# PoggioAI/MSc-claude

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![PoggioAI](https://img.shields.io/badge/PoggioAI-MSc-orange)](https://PoggioAI.github.io)

A Claude Code skill that runs autonomous multi-agent research: takes a hypothesis and produces a literature-grounded, evidence-backed manuscript draft in 10 human steers or fewer.

[Website](https://PoggioAI.github.io) | [Paper](https://poggioai.github.io/papers/poggioai-msc-v0.pdf) | [Full Python System](https://github.com/PoggioAI/PoggioAI_MSc) | [Discord](https://discord.gg/Pz7spPPY)

---

## How it works

This is a **Claude Code skill** -- a single orchestrator file (`SKILL.md`) plus 20 prompt files that together implement a research state machine. When invoked, the orchestrator:

1. Creates a timestamped workspace and initializes `state.json`
2. Executes phases sequentially, spawning each as a subagent with its prompt file
3. Validates outputs after each phase and enforces gate conditions
4. Routes through loopbacks on failure (novelty gate, verify completion, duality gate, review gate)
5. Checkpoints after every phase so runs can be resumed

No Python, no dependencies, no API keys beyond Claude Code itself.

---

## Installation

```bash
git clone https://github.com/PoggioAI/PoggioAI_MSc-claude.git
cp -r PoggioAI_MSc-claude ~/.claude/skills/poggioai-msc-claude
```

Then invoke with `/poggioai-msc-claude` in any Claude Code session.

---

## Usage

### Inline task

```
/poggioai-msc-claude "Investigate whether neural scaling laws exhibit phase transitions analogous to thermodynamic critical phenomena"
```

### From a task file

Create `research_task.md` in your working directory, then:

```
/poggioai-msc-claude
```

### Resume an interrupted run

```
/poggioai-msc-claude --resume /path/to/research-20260325-143022
```

The orchestrator reads `state.json` and picks up from the last completed phase.

---

## Pipeline phases

| # | Phase | Description |
|---|-------|-------------|
| 1 | **Persona Council** | Three competing personas (Practical, Rigor, Narrative) debate the research direction; a synthesis coordinator integrates their feedback |
| 2 | **Literature Review** | Adversarial novelty falsification -- assumes every claim is already known and searches for evidence |
| -- | *Feasibility Gate* | Blocks if core hypothesis is flagged as known; loops back to persona council (max 2 retries) |
| 3 | **Brainstorm** | Generates candidate approaches with structured pros/cons |
| 4 | **Formalize Goals** | Entry node, goal formalization, research plan writeup, track decomposition gate, milestone checkpoint |
| 5a | **Theory Track** | Math literature review, claim proposal, proof construction, rigorous + empirical verification |
| 5b | **Experiment Track** | Experiment design, execution, result verification |
| 5c | **Track Merge** | Merges theory and experiment outputs |
| 5d | **Verify Completion** | 3-way routing: complete (>=80%), incomplete (>=50%, retry goals), rethink (<50%, restart brainstorm) |
| 6 | **Formalize Results** | Synthesizes all track outputs into structured results |
| -- | *Duality Gate* | Checks theory-experiment consistency; failure routes through followup lit review then brainstorm |
| 7 | **Resource Prep** | Prepares figures, tables, and resource inventory |
| 8 | **Writeup** | Produces LaTeX manuscript |
| 9 | **Proofreading** | Entry node (targeted vs. full audit), then copyediting pass |
| 10 | **Reviewer** | Quality assessment with hard blockers (B1-B5), milestone checkpoint, validation gate |

---

## Quality mechanisms

Eight features distinguish this from "just asking Claude to write a paper":

1. **Multi-round persona debate** -- Three agents with competing objectives (feasibility vs. rigor vs. narrative) debate for 3 rounds, getting HARDER each round. Minimum 2 rounds before any idea can proceed. No single-pass acceptance.
2. **Adversarial novelty falsification** -- Literature review assigns every claim a status (OPEN / PARTIAL / KNOWN / EQUIVALENT_KNOWN) and blocks if core hypothesis is already known
3. **Resumable literature review** -- Literature review works in batches and saves partial progress. If it times out, the orchestrator detects incomplete results and re-spawns the subagent to continue from where it left off — no work is lost.
4. **Theory-experiment independence** -- Tracks share a decomposition but NOT continuous working context, preventing blind spot inheritance
5. **Hard blockers** -- Reviewer enforces 5 binary checks (B1-B5). If any is true, score is capped at 4/10. No rubber-stamping.
6. **Verify completion with stall detection** -- LLM-based goal assessment with 3-way routing and progress tracking across cycles
7. **Modular writeup with loopback** -- Review failures route back through writeup and proofreading with specific feedback injection
8. **Escalating review failure** -- The system does NOT just keep rewriting the same bad paper. Review gate has three escalation levels:
   - Score 4-5 (editorial problems): loops back to writeup (1st failure), then all the way back to persona council ideation (2nd failure)
   - Score ≤ 3 (fundamentally weak research): loops back to persona council immediately for full restart
   - If a full ideation restart has already been attempted and fails again, the system finalizes and hands off to the human

---

## File structure

```
poggioai-msc-claude/
  SKILL.md                    # Orchestrator -- state machine, routing, gate logic
  README.md                   # This file
  prompts/
    01-persona-practical.md   # Practical Compass persona
    02-persona-rigor.md       # Rigor & Novelty persona
    03-persona-narrative.md   # Narrative Architect persona
    04-persona-synthesis.md   # Synthesis coordinator
    05-literature-review.md   # Adversarial novelty falsification
    06-brainstorm.md          # Approach generation
    07-formalize-goals.md     # Goal formalization + track decomposition
    08-math-literature.md     # Theory-specific literature review
    09-math-proposer.md       # Claim/theorem proposal
    10-math-prover.md         # Proof construction (named technique library)
    11-math-verifier.md       # Rigorous + empirical verification
    12-experiment-design.md   # Experiment design
    13-experimentation.md     # Experiment execution
    14-experiment-verify.md   # Result validation
    15-formalize-results.md   # Results synthesis
    16-duality-check.md       # Theory-experiment consistency
    17-resource-prep.md       # Figures, tables, resource inventory
    18-writeup.md             # LaTeX manuscript generation
    19-proofreading.md        # Copyediting and validation
    20-reviewer.md            # Quality gate with hard blockers
  templates/
    state.json                # State machine template
  examples/
    quickstart-task.txt       # Example research hypothesis
  scripts/                    # Utility scripts
```

---

## The ~10-minute timeout problem and how we solve it

Claude Code subagents can time out after approximately 10 minutes. This is a hard constraint of the platform — there is no way to extend it. For research phases that involve web search, proof construction, or paper writing, 10 minutes is often not enough.

**Our solution: mandatory multi-pass execution with automatic resume.**

Every phase runs in a resume loop. The orchestrator spawns the subagent, and when it returns (whether it finished or timed out), it checks the artifacts. If they're incomplete, it re-spawns the subagent with a `RESUME:` prefix so it continues from where it stopped. This happens automatically — no human intervention needed.

Each phase has a minimum and maximum number of passes:

| Phase | Min | Max | Why |
|-------|-----|-----|-----|
| Literature review | 2 | 5 | Web search is slow; multiple passes catch timeouts and deepen coverage |
| Brainstorm | 2 | 5 | Deeper exploration produces better approaches |
| Formalize goals | 2 | 5 | Validation and cross-checking need iteration |
| Theory track (all 4 phases) | 2 | unbounded | Proofs often need multiple attempts; quality scales with iteration |
| Experiment track (all 3 phases) | 2 | unbounded | Experiments may need reruns; verification improves with depth |
| Writeup | 2 | 5 | Paper writing always benefits from revision |
| Proofreading | 2 | 5 | More issues caught with each pass |

This multi-pass approach also **improves quality**: even when the first pass completes successfully, the second pass reviews and refines the output. For theory and experiment phases, the system keeps iterating until it explicitly detects no further progress (stall detection).

## 30+ hour campaigns

For long-running research campaigns, use Claude Code Desktop Scheduled Tasks:

```
/schedule poggioai-msc-claude to advance the campaign every 30 minutes
```

Each invocation reads `state.json`, runs one or more phases (with multi-pass within each), checkpoints, and exits. The scheduler re-invokes to advance the next phase.

Note: Scheduled tasks are session-scoped and auto-expire after 3 days. For campaigns longer than 3 days, re-create the schedule and use `--resume`.

---

## Comparison with full system

### What you keep at full quality

| Feature | Status |
|---------|--------|
| Prompt logic (personas, novelty falsification, hard blockers, proof rigor) | Full |
| Artifact-centric workflow (all outputs are files on disk) | Full |
| Quality gates + routing (feasibility, verify-completion, duality, review) | Full |
| Verify completion with 3-way routing (complete/incomplete/rethink) | Full |
| Duality gate with followup literature review | Full |
| Entry nodes for context injection (formalize_goals_entry, proofreading_entry) | Full |
| Resume from checkpoint | Full (JSON instead of SQLite) |

### What is worse than the full system

This is an honest list. If these matter to you, use the [full Python system](https://github.com/PoggioAI/PoggioAI_MSc) instead.

| What's worse | Why it matters | How bad is it? |
|-------------|----------------|----------------|
| **No multi-model counsel** | The full system runs Claude + GPT + Gemini in parallel, then debates. This skill uses Claude only. You lose cross-model disagreement, which is the mechanism that catches single-model blind spots. | Significant for literature review and writeup quality. The full system's counsel mode is its strongest quality differentiator. |
| **No true parallelism** | Theory and experiment tracks run sequentially, not in parallel. Subagents run one at a time. | Doubles wall-clock time for "both" track runs. No quality impact, just slower. |
| **No tree search breadth** | The full system explores N proof strategies in parallel via DAG-layered best-first search. This skill tries strategies sequentially with backtracking. | For hard theorems, the full system finds proofs faster by exploring multiple approaches simultaneously. This skill will eventually try the same approaches, just slower. |
| **No autonomous repair** | The full system's campaign heartbeat can deploy a Claude Code repair agent to diagnose and fix failed stages automatically. This skill just stops and asks you. | Minor — you're steering anyway. The repair agent saves time on infrastructure failures, not research failures. |
| **No experiment execution sandbox** | The full system integrates AI-Scientist-v2 for actual code execution with subprocess isolation. This skill designs experiments but cannot run code. | You'll need to run experiments manually. The skill produces experiment_design.json; you execute it yourself. |
| **No budget tracking** | The full system tracks token usage and cost per stage via JSON ledgers with hard budget caps. This skill has no spend visibility. | You won't know how much each phase cost. Monitor your API usage dashboard separately. |
| **No SLURM/HPC integration** | The full system can submit GPU jobs via SLURM. This skill runs locally only. | Only matters if you need GPU experiments on a cluster. |
| **Weaker error recovery** | The full system has circuit breakers, exponential backoff, quorum logic for counsel failures, and campaign-level death-loop detection. This skill relies on Claude's built-in retry logic. | Occasionally a phase may fail silently where the full system would retry or escalate. Check artifacts manually. |
| **Context window limits** | A single Claude Code session has finite context. Very long runs (20+ phases) may degrade as context fills up. The full system uses fresh LLM calls per agent with no shared context accumulation. | For long campaigns, use scheduled tasks (each invocation gets fresh context) rather than a single continuous session. |

### Bottom line

Use this skill for: quick research explorations, literature reviews, paper outlining, and ML theory work where you're actively steering.

Use the full system for: production-quality papers, multi-model quality assurance, automated experiment execution, and unattended multi-day campaigns.

---

## Configuration

### Modify prompt files

Each prompt file in `prompts/` is self-contained. Edit any file to change agent behavior for that phase. The orchestrator loads prompts by path -- no registration needed.

### Change phase routing

Edit the Phase Routing Table and Gate Logic sections in `SKILL.md`. Key parameters:
- Feasibility gate max retries: 2 (search for `novelty_gate`)
- Verify completion rework max: 3 (search for `verify_rework_attempts`)
- Verify completion brainstorm max: 3 (search for `brainstorm_cycle`)
- Duality gate max retries: 2 (search for `duality_gate`)
- Review gate max retries: 3 (search for `review_gate`)

### Adjust quality thresholds

- Verify completion ratios: 0.8 (complete), 0.5 (incomplete), <0.5 (rethink) -- in SKILL.md verify completion section
- Review pass score: 6/10 -- in Gate 3 section
- Hard blocker definitions: B1-B5 in `prompts/20-reviewer.md`

---

## Known issues and limitations

These are known architectural issues with the multi-pass orchestration. They are documented here for transparency. Contributions addressing any of them are welcome.

**1. Resume passes may overwrite good artifacts.** Pass 1 produces an artifact. Pass 2 gets a `RESUME:` prompt and is told to "refine." But the subagent may interpret "refine" as "rewrite from scratch," producing a worse version that overwrites the good one. The prompts say "do not restart" but Claude doesn't always follow that with long, complex outputs. A future fix would back up artifacts before each resume pass (`cp file.md file_pass1.md`).

**2. Unbounded theory/experiment loops could burn excessive tokens.** Math prover and experiment phases have `max_passes: unbounded`. If a proof is genuinely hard, the system could loop many times with minimal progress. Stall detection checks whether artifacts changed between passes, but subtly different failed proofs may evade this check. Consider setting a hard cap (e.g., 10 passes) if you're cost-conscious.

**3. Resume prompt may conflict with phase prompt.** The `RESUME (pass N):` prefix is prepended to the full phase prompt. But the phase prompt has its own "Process" section starting from step 1. The subagent might ignore the RESUME prefix and restart from step 1. We mitigate this by explicitly instructing "SKIP steps you already completed" in the resume prefix, but it's not foolproof.

---

## Contributing

Contributions are welcome. To improve the skill:

1. Fork the repository
2. Edit prompt files or SKILL.md routing logic
3. Test on a research task end-to-end (use `--resume` to iterate on specific phases)
4. Submit a PR with a description of what changed and why

Areas where contributions are especially valuable:
- Prompt improvements for specific research domains
- Better experiment execution strategies
- Additional quality gates or verification steps

---

## License

MIT License.

Based on [PoggioAI/MSc](https://github.com/PoggioAI/PoggioAI_MSc) by Mahmoud Abdelmoneum, Pierfrancesco Beneventano, and Tomaso Poggio (MIT + Perseus Labs).
