# Grader-in-the-Loop

**Coding agent reliability is an emergent property of workflow architecture, not of the underlying model.**

## The Insight

Two things already exist:

1. **Benchmark-grade evaluation** — deterministic, evidence-fetching, HOLD-default grading. This is how we *measure* coding agents.
2. **Test-driven development** — acceptance criteria written before implementation, verified against artifacts. This is standard software engineering.

The connection: move the benchmark grader from post-hoc measurement into the live execution loop. The implementing agent writes code. A separate evaluating agent grades it — with the same rigor a benchmark would use. No agent evaluates its own work.

The result: **precision converges across all models.** When the pipeline says PASS, the code is correct — regardless of whether the model behind it is frontier or open-source. Capability differences between models manifest as *cost* (cycles, tokens, wall-clock time), not as *reliability failures* (silent bugs, hallucinated evidence, premature completion claims).

## The Architecture

Four instruction files. No framework. No orchestration engine. Just structured constraints that any LLM can follow.

```
tdd-plan ──→ qa-plan ──→ tdd-slice ──→ qa-slice
 (plan)      (audit)     (execute)     (grade)
                │                         │
                └── HOLD? fix and retry ──┘
```

### tdd-plan — The Architect
Decomposes work into sequentially-dependent slices. Each slice has a proof-oriented Objective ("prove that X holds under Y") not an implementation task ("add feature X"). Acceptance criteria pass a 5-point sufficiency test: specificity, artifact requirement, diagnostic fit, ghost-read resistance, binary precision.

### qa-plan — The Plan Auditor
Stateless 12-point adversarial audit of the plan document. HOLD-default. Checks whether every AC is specific enough, artifact-backed, diagnostic, and mechanically parseable by downstream skills. Social pressure immune — canned response script for pushback.

### tdd-slice — The Implementer
Five strictly sequential phases: plan → implement → collect evidence → submit → invoke QA gate. Cannot evaluate its own work. Cannot proceed without a QA PASS. Cannot revise acceptance criteria after committing to them.

### qa-slice — The Grader
Extracts the contract from the issue body *before* reading the submission (preventing anchoring bias). Fetches every artifact via API — unfetched artifacts are INSUFFICIENT_EVIDENCE. Item-by-item evaluation against the fixed contract. The decision is deterministic: PASS if and only if ALL criteria are MET.

### Structural Properties

| Property | Mechanism |
|:--|:--|
| **Identity separation** | Implementing and evaluating agents operate under separate credentials. Correlated self-evaluation errors are structurally impossible. |
| **Contract fixation** | Acceptance criteria are locked before implementation begins and extracted before the submission is read. Specification drift and anchoring bias are mechanically prevented. |
| **HOLD-default** | Ambiguity, missing artifacts, and unverifiable claims default to HOLD. The pipeline never guesses. |
| **Artifact fetching** | The grader fetches evidence via API. Claims without fetched artifacts are treated as non-existent. Ghost reads are impossible. |
| **Execution gating** | The QA verdict is the only unlock for the next phase. The implementer's self-assessment is a null input to the gate function. |

## The Claim

ProjDevBench (Lu et al., 2026) measured 27.38% acceptance across six frontier agent-model pairs. Dominant failures: specification misalignment (41.86%), self-evaluation bias (systemic).

These are not model failures. They are architecture failures. The grader-in-the-loop pipeline addresses them mechanically:

| ProjDevBench Failure Mode | Pipeline Remedy | Mechanism |
|:--|:--|:--|
| Specification misalignment (41.86%) | Contract fixation | ACs written and audited before any code. The grader evaluates against the fixed contract, not the agent's interpretation. |
| Self-evaluation bias (systemic) | Identity separation + artifact fetching | The agent cannot grade its own work. The grader fetches evidence — it does not read claims. |
| Time limit exceeded (13.91%) | Execution gating | HOLD loops surface capability limits as overt timeouts, not silent failures. A model that can't fix its work fails loudly. |

### What Converges (The Floor)

When the pipeline emits PASS, the work meets its acceptance criteria — verified by fetched artifacts, not agent assertions. This holds regardless of model. The false positive rate drops to zero.

### What Diverges (The Cost)

Frontier models reach PASS in fewer cycles. Open-source models take more cycles, consume more tokens, and occasionally time out on hard problems. **Capability differences become compute costs, not reliability failures.**

This means: organizations can trade cheap local compute (looping a smaller model longer) for guaranteed reliability, rather than paying a premium for frontier models and hoping they self-correct.

## Evidence

Evidence is collected as GitHub issues with full execution traces — plan reviews, slice submissions, QA verdicts, HOLD remediation cycles, and post-mortems. Every artifact is committed and fetchable.

| Source | What It Shows |
|:--|:--|
| [Pipeline execution traces](evidence/) | Multi-slice TDD cycles on real-world coding problems: CUDA IPC, process isolation, wheel resolution, DLPack staging |
| [ProjDevBench benchmark runs](benchmarks/) | Cross-model × cross-harness results on standardized problems |

*Controlled 2×2 experiment (with/without pipeline × frontier/open-source model) is in progress.*

## Using This Pipeline

The pipeline is four SKILL.md instruction files and a posting protocol. It runs on any coding agent that can read instructions, execute shell commands, and post to GitHub.

**To adopt it for your own codebase:** read the [CLAUDE.md](CLAUDE.md) in this repo. It explains how to configure the skills, set up the GitHub App identities, and point the pipeline at your problems.

**Requirements:**
- A GitHub repo with issues enabled
- Two GitHub App registrations (one for the implementer identity, one for the evaluator identity)
- A coding agent (Claude Code, Codex, or any agent that follows structured instructions)

## Citation

If you use this methodology or build on it, please cite:

```
Pollock, J. (2026). Grader-in-the-Loop: Agent Reliability as an Emergent Property
of Workflow Architecture. GitHub: pollockjj/grader-in-the-loop
```

## License

[MIT](LICENSE) — free as in free beer. Take it, use it, modify it, ship it.
