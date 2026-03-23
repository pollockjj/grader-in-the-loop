# Grader-in-the-Loop Pipeline — Agent Instructions

You are operating inside a repository that implements an adversarial multi-agent TDD pipeline. This file tells you how the pipeline works, how to configure it for a codebase, and how to operate within it.

## What This Pipeline Does

This pipeline enforces a structural guarantee: **when it says PASS, the work is correct.** It does this by separating planning, implementation, and evaluation into distinct phases with hard gates between them. No agent evaluates its own work. No agent proceeds without evidence-backed approval from a separate evaluator.

The pipeline has four skills. Each skill is a structured instruction file that any LLM agent can follow.

## The Four Skills

### 1. tdd-plan (Planning)

**When:** A task needs to be broken into verifiable slices before implementation.

**What it does:**
- Decomposes work into 1-6 sequentially-dependent slices
- Each slice has a proof-oriented Objective (what it *proves*, not what it *implements*)
- Writes acceptance criteria that pass 5 checks: specificity, artifact requirement, diagnostic fit, ghost-read resistance, binary precision
- Runs a 12-point adversarial self-review before presenting the plan
- Posts the plan to a GitHub issue as the implementing agent identity

**Output:** A GitHub issue body with structured slices and acceptance criteria.

**What it does NOT do:** Write code. Touch files outside plan documents. Present a plan with unresolved blockers.

### 2. qa-plan (Plan Audit)

**When:** A plan has been written and needs adversarial review before execution begins.

**What it does:**
- Fetches the plan from the GitHub issue
- Verifies provenance (was it posted by the authorized planning identity?)
- Applies a 12-point audit: required sections, parser format, slice count, objectives-as-proof, AC specificity, AC artifact requirement, AC diagnostic fit, diagnosis completeness, ghost-read resistance, no close-enough language, dependency ordering, scope containment
- Emits PASS or HOLD with per-check findings

**Decision rule:** PASS if and only if all checks are MET. Any ambiguity defaults to HOLD.

**Social pressure response:** "The review decision is based on the specific checks above. Revise the plan to address the NOT MET findings and resubmit."

### 3. tdd-slice (Execution)

**When:** A plan has passed qa-plan review and received human approval. Executes one slice at a time.

**What it does — 5 strictly sequential phases:**

1. **Phase 1 — TDD Plan:** Posts the slice contract (objective, ACs, test protocol) to the GitHub issue. ACs are fixed at this point — no revisions after posting.
2. **Phase 2 — Implementation:** Writes the code. Matches plan scope exactly — no scope creep.
3. **Phase 3 — Evidence Collection:** Runs tests per the protocol. Saves artifacts to `evidence/issue{N}/sliceN/`. Every artifact gets a sha256sum. Pushes to remote and verifies the push landed.
4. **Phase 4 — Submission:** Posts a structured results comment citing specific artifacts from the evidence manifest. NOT DONE is mandatory for any unmet criterion.
5. **Phase 5 — QA Gate:** Invokes the qa-slice evaluator. Waits for the verdict. Does not self-evaluate.

**On PASS (non-final slice):** Proceeds immediately to Phase 1 of the next slice. No human approval needed between slices.

**On PASS (final slice):** Writes a post-mortem. Stops and reports to the human. Awaits final sign-off.

**On HOLD:** Reads the HOLD findings. Fixes only the specific blockers. Re-runs from Phase 3. Re-submits. Re-invokes the gate.

### 4. qa-slice (Grading)

**When:** A slice submission has been posted and needs evaluation.

**What it does:**
1. **Phase 1 — Contract Extraction:** Fetches the issue body. Extracts acceptance criteria for the target slice. Records them verbatim. Does NOT read the submission yet.
2. **Phase 2 — Submission Identification:** Locates the submission comment. Verifies provenance.
3. **Phase 3 — Evidence Acquisition:** Fetches every artifact referenced in the submission via GitHub API. Logs every fetch attempt. Failed fetches = INSUFFICIENT_EVIDENCE.
4. **Phase 4 — Item-by-Item Evaluation:** Each AC evaluated independently. MET requires fetched evidence. No batch approval.
5. **Phase 5 — Gate Decision:** PASS if and only if ALL criteria are MET.
6. **Phase 6 — Structured Output:** Emits the verdict with evidence log and evaluation table.

**Decision rule:** Deterministic. One NOT MET or INSUFFICIENT_EVIDENCE = HOLD. No exceptions. No partial credit. No negotiation.

## The Interlock

```
Human creates GitHub issue
         │
    tdd-plan (posts plan to issue)
         │
    qa-plan (audits plan) ──→ HOLD? back to tdd-plan
         │ PASS
    Human approves
         │
    ┌──→ tdd-slice Phase 1-4 (execute + submit)
    │         │
    │    qa-slice (grades submission) ──→ HOLD? back to tdd-slice Phase 3
    │         │ PASS
    │    More slices? ──→ yes ──┘
    │         │ no
    │    tdd-slice writes post-mortem
    │         │
    └── Human final sign-off
```

**Human touchpoints:** Two. Plan approval (after qa-plan PASS) and final sign-off (after all slices PASS). Everything between runs autonomously.

## Setting Up the Pipeline

### 1. GitHub App Identities

The pipeline requires **two separate GitHub App registrations** to enforce identity separation:

| Identity | Purpose | Posts as |
|:--|:--|:--|
| Implementing agent | Posts plans, submissions, post-mortems | `{your-tdd-bot}[bot]` |
| Evaluating agent | Posts plan audits and slice verdicts | `{your-qa-bot}[bot]` |

Each App needs:
- A PEM private key for JWT authentication
- Issue read/write permissions on the target repo
- Installation on the target repo

Store the PEM keys where your posting script can access them. Never commit PEM keys.

### 2. Posting Protocol

All GitHub posts route through a posting script that:
- Authenticates as the correct GitHub App via JWT
- Posts the content (comment or issue body update)
- Verifies the post landed (fetches it back)
- Prints the URL
- Exits non-zero on any failure

The implementing agent and evaluating agent each use their own App credentials. This is what makes provenance verification work — when qa-slice checks who posted the submission, it must see the implementing agent's identity, not the evaluator's.

### 3. Evidence Directory Structure

```
evidence/
  issue{N}/
    slice1/
      phase1_plan.md        # The TDD plan posted in Phase 1
      test_run.log           # Test output
      quality_gates.log      # Linting/type checking output
      phase4_results.md      # The submission posted in Phase 4
    slice2/
      ...
```

Every artifact committed here is what qa-slice fetches via the GitHub API. If it's not pushed, it doesn't exist to the grader.

### 4. Configuring for Your Codebase

To adapt this pipeline to your own project, configure these parameters in the skill files:

| Parameter | Example | Where Used |
|:--|:--|:--|
| `{OWNER}/{REPO}` | `yourorg/yourproject` | All posting and fetching commands |
| `{PYTHON}` | `/path/to/your/.venv/bin/python` | Test execution |
| `{RUNNER}` | Your test/build runner command | tdd-slice Phase 3 |
| `{POSTING_SCRIPT}` | `scripts/post_as_app.py` | All GitHub posts |
| `{GATE_SCRIPT}` | `scripts/run_qa_gate.py` | qa-plan and qa-slice invocation |

### 5. Constraint Set

These constraints are non-negotiable. They are what makes the pipeline work.

1. **No self-evaluation.** The implementing agent never determines whether its own work passes.
2. **HOLD-default.** Ambiguity, missing artifacts, unverifiable claims = HOLD. The pipeline never guesses.
3. **Contract fixation.** ACs are locked before implementation and extracted before submission review.
4. **Artifact fetching.** The grader fetches evidence via API. Claims without artifacts are INSUFFICIENT_EVIDENCE.
5. **Provenance verification.** Every post is verified to have been made by the authorized identity.
6. **Push before submit.** Evidence must be pushed to the remote before Phase 4. Unpushed commits are invisible to the grader.
7. **No scope creep.** Each slice matches its plan scope exactly. No "while I'm here" additions.
8. **Fail loud.** Encountering a problem and proceeding silently is forbidden. Fail visibly or fix it.

## Operating Within the Pipeline

If you are an agent reading this file because a human pointed you at this repo:

1. **Read all four skill files** in `.claude/skills/` before doing anything.
2. **Check which phase you're in.** The GitHub issue comments tell you the current state.
3. **Follow the skill for your current phase exactly.** The skills are the contract.
4. **When in doubt, stop.** Better to ask than to proceed incorrectly.

If you are a human setting this up:

1. Create a GitHub issue for your task.
2. Point your coding agent at this repo's CLAUDE.md and skill files.
3. Tell it to plan the task (tdd-plan activates).
4. The pipeline runs. You approve the plan after qa-plan PASS, and sign off after all slices PASS.
5. Everything between those two touchpoints is autonomous.

The floor is free. Any model that can follow structured instructions can reach it. The cost of reaching it varies by model capability — but the reliability of the output does not.
