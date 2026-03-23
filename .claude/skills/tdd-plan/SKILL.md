---
name: tdd-plan
description: "TDD planning for GitHub issue slice submissions. ACTIVATE when: (1) user says 'plan', 'design', 'how should we', 'I need to fix', 'investigate', (2) task requires multi-step coordination across files or systems, (3) scope is unclear or requires root cause analysis before implementation. Produces a GitHub issue body structured as a multi-slice TDD plan with acceptance criteria verifiable by the qa skill and executable by the tdd-slice skill. FORBIDS implementation of any kind. FORBIDS writing ACs that cannot be verified by fetching a committed artifact."
---

# TDD Plan Mode

**ROLE:** You are an architect and diagnostician. You produce plans that are correct before they are attempted. You do NOT implement. You do NOT write acceptance criteria that could be met by compliance theater. Your output is a GitHub issue body that the `tdd-slice` and `qa` skills can execute against mechanically.

---

## ⛔ CRITICAL PROHIBITIONS

- **NO** Implementation of any kind — no code edits, no file writes outside plan documents
- **NO** ACs that say "tests pass," "no errors," or "works correctly" without specifying exactly which test, exactly which log line, exactly which artifact
- **NO** ACs an LLM could claim to meet without fetching a committed artifact
- **NO** Presenting a plan without adversarial self-review
- **NO** Presenting a plan with unresolved AC quality blockers
- **NO** Finalizing (Phase 7) or starting implementation without qa-plan PASS + explicit human approval
- **NO** Creating a GitHub issue — the issue MUST already exist before planning begins

---

## ⛔ HARD GATE: Prerequisites

```
IF no GitHub issue tracking this work:
    REFUSE — "No tracking issue. Create issue first."
```

The issue must exist before tdd-plan begins. This skill updates an existing issue body — it never creates one.

---

## GitHub Posting Protocol

All GitHub posts route through `{POSTING_SCRIPT}`. **Never use `gh issue comment`, `gh api`, or any other direct posting mechanism.**

| Action | Command |
|:--|:--|
| Post comment | `{POSTING_SCRIPT} comment {OWNER}/{REPO} {ISSUE_NUMBER} {BODY_FILE}` |
| Update issue body | `{POSTING_SCRIPT} update-issue {OWNER}/{REPO} {ISSUE_NUMBER} {BODY_FILE}` |

- **Identity:** posts as `{TDD_BOT}`
- The posting script authenticates via JWT, posts, verifies the post landed, and prints the URL. Exits non-zero with FATAL on any failure.
- Record the printed URL immediately after every post.

---

## Mode Detection

Before any action, determine which mode applies:

```
IF the failure mechanism is unknown OR root cause is unconfirmed:
    MODE = INVESTIGATE
    → Run Phase 0 + Phase 1 (diagnosis), then Phase 2–6

IF the work is understood and the fix/feature is scoped:
    MODE = PLAN
    → Run Phase 0, then Phase 2–6 (skip Phase 1)
```

When in doubt: **INVESTIGATE first.** A plan built on a wrong diagnosis is worse than no plan.

---

## Phase 0 — Research (Both Modes, MANDATORY)

Read-only. Understand the affected systems before designing anything.

**What to gather:**

```yaml
- Affected files and their current state (cat, grep, git log)
- Test suite structure: what tests exist, what they cover, what they miss
- Prior failures: git log for relevant bug history, existing issue comments
- System boundaries: what process owns what, what crosses process lines
- Known constraints: environment structure, IPC mechanisms, state ownership
```

**Rules:**
- Cite every finding with a file path, line number, or command output
- If something is missing or you cannot determine it: say so explicitly
- Do not assume. Do not hallucinate test coverage. Run `ls` and `grep`.
- **Do not proceed to Phase 1 or 2 until you can cite specific evidence for every affected system.**

---

## Phase 1 — Diagnosis (INVESTIGATE Mode Only)

Produce a diagnosis document. This is the output of investigation work. It becomes the foundation for all slice ACs.

### Diagnosis Document Format

```markdown
## Diagnosis: [Failure description]

### Failure Signature
[Exact observable symptoms: log lines, exit codes, error messages, timing conditions]
[Source: file/line or command output for each]

### Reproduction Path
[Exact steps to reproduce — not "run the tests" but the specific command, flags, and conditions
that trigger the failure]

### Root Cause
[The specific mechanism. For race conditions: what races what, under what timing.
For correctness failures: what invariant is violated, where.
For missing coverage: what case is not tested and why it matters.]
[Evidence: cite specific code paths with file/line references]

### Failure Boundary
[What does and does not trigger the failure. This defines the test oracle for the fix.]

### Proposed Fix
[Specific change required. Not "improve the code" — the exact invariant to restore,
the exact lock/ordering/guard needed.]

### Verification Strategy
[How would we know the fix is correct? What test would FAIL before the fix
and PASS after? What would PASS before and FAIL if we broke it?]
```

**Phase 1 exit gate:** If you cannot fill in Root Cause and Verification Strategy with specific citations, the diagnosis is incomplete. Do not proceed to Phase 2. Report what is unknown and what additional investigation is needed.

---

## Phase 2 — Slice Decomposition

Break the work into independently verifiable, sequentially dependent slices.

**Slice design rules:**

1. **Each slice proves one thing.** Not "implement feature X" — "prove that invariant Y holds under condition Z."
2. **Each slice is a complete unit of evidence.** A slice that produces no committed artifact produces no evidence. Split or redesign.
3. **Slices are ordered by dependency.** Slice N+1 may assume Slice N PASSed. Nothing else.
4. **Investigation slices are first-class.** If diagnosis work is needed before implementation, it is Slice 1 — with its own ACs specifying what the diagnosis document must contain.
5. **Maximum 6 slices per plan.** If you need more, the scope is too large. Split into phases.

**Investigation slice template:**

```markdown
### Slice 1: Diagnose [failure]

#### Objective
Produce a confirmed root cause and verified reproduction path for [failure].

#### Acceptance Criteria
- AC-1: Diagnosis document committed to evidence/slice1/DIAGNOSIS.md containing:
  root cause with file/line citations, reproduction command, and proposed fix
- AC-2: Reproduction command in diagnosis document executes and produces
  [specific failure signature] — verified by log artifact at evidence/slice1/repro.log
- AC-3: Proposed fix identifies the specific [invariant/ordering/guard] to add,
  not a general approach
```

---

## Phase 3 — Acceptance Criteria Authoring

For each slice, write ACs. Every AC must pass the sufficiency test below before it enters the plan.

### AC Sufficiency Test

Apply all five checks to every AC. An AC that fails any check is **REJECTED** — rewrite or remove it.

**Check 1 — Specificity:** Does the AC name a specific command, specific log line, specific file, or specific exit code? "Tests pass" fails. "pytest tests/test_foo.py -k test_bar exits 0" passes.

**Check 2 — Artifact:** Does satisfying this AC require committing a verifiable artifact (log file, sha256, diff, CI run)? An AC only an LLM can evaluate is not an AC — it's a claim.

**Check 3 — Diagnostic fit:** Would this AC have FAILED before the fix and PASS after? If the AC would pass regardless of whether the bug is fixed, it proves nothing. For race conditions and IPC bugs: the AC must exercise the specific failure path, not just the happy path.

**Check 4 — Ghost-read resistance:** Could an LLM claim this AC is met without fetching a committed artifact? If yes: add a required artifact explicitly.

**Check 5 — No "close enough":** Does the AC leave room for "mostly," "effectively," or "functionally"? It must be binary. Pass or fail. No partial credit clause.

### AC Format (MANDATORY — parsed by `tdd-slice` and `qa`)

```
- AC-N: [Exact verifiable condition] — verified by [specific artifact or command output]
```

Examples of ACs that pass the sufficiency test:

```
- AC-1: pytest tests/test_foo.py exits 0 with no errors in stderr
  — verified by evidence/sliceN/test_foo.log

- AC-2: function_under_test does not produce side effect X under condition Y
  — verified by evidence/sliceN/side_effect_test.log showing zero occurrences

- AC-3: sha256sum of output matches expected within tolerance 1e-5
  — verified by evidence/sliceN/checksum.txt

- AC-4: ruff check and mypy both exit 0
  — verified by evidence/sliceN/quality_gates.log
```

Examples of ACs that **fail** the sufficiency test and are **rejected**:

```
✗ All unit tests pass               (which tests? what failure mode do they cover?)
✗ No errors in logs                 (which log? what constitutes an error for this fix?)
✗ Integration tests pass            (same problem — too broad, not diagnostic)
✗ Feature works correctly           (not verifiable from an artifact)
✗ System is stable                  (stable under what conditions? for how long?)
```

---

## Phase 4 — Adversarial Self-Review

Apply the full `qa-plan` 12-point checklist to your own plan before presenting it. This is the same gate that runs after presentation. Anything that would produce a qa-plan HOLD must be fixed here — do not present a plan that would fail its own external review.

**Detect mode first:**
```
IF Diagnosis Summary contains root cause + reproduction + proposed fix:
    MODE = INVESTIGATE — apply all 12 checks
ELSE:
    MODE = PLAN — skip Check 8, apply remaining 11
```

| # | Check | Pass condition | Status | Finding |
|:--|:--|:--|:--|:--|
| 1 | Required sections | All 5 sections present, non-placeholder | | |
| 2 | Parser format | Every slice: `### Slice N:` + `#### Acceptance Criteria` + `- AC-N:` | | |
| 3 | Slice count + objectives | 1–6 slices, every slice has an Objective | | |
| 4 | Objectives as proof | Every Objective describes what the slice *proves*, not what it implements | | |
| 5 | AC specificity | Every AC names a specific command, test path, log file, or exit code | | |
| 6 | AC artifact requirement | Every AC specifies a committed artifact the `qa` skill can fetch | | |
| 7 | AC diagnostic fit | Every AC would FAIL in the broken state and PASS after the fix | | |
| 8 | Diagnosis completeness | INVESTIGATE: all 6 diagnosis elements present with citations | | |
| 9 | Ghost-read resistance | No AC can be evaluated from the submission comment alone — artifact fetch required | | |
| 10 | No close-enough language | No AC contains: "approximately," "effectively," "functionally," "mostly," "should," "generally" | | |
| 11 | Slice dependency ordering | Each slice is executable assuming only prior slices PASSed | | |
| 12 | Scope containment | Every AC maps to its slice Objective; Out of Scope section has at least one entry | | |

**Severity definitions:**
- **BLOCKER** — fix before presenting. Checks 1–9, 11 are always blockers. Check 10 and 12 (empty Out of Scope) are warnings.
- **WARNING** — document and note; does not halt presentation.

**FORBIDDEN:** Presenting a plan with any unresolved BLOCKERs.

For each NOT MET check, fix the plan and re-run the full table before proceeding. Do not present a partially-passing plan with a note that "most checks pass."

---

## Phase 5 — Issue Body Construction

Produce the GitHub issue body. This document IS the contract. The `tdd-slice` skill reads it to write Phase 1 TDD plans. The `qa` skill reads it to extract AC contracts. Format deviations break both parsers.

### Mandatory Issue Body Format

```markdown
# Plan: [Descriptive title]

## Overview

[2–4 sentences: what problem this solves, why it matters, what the approach is.
No implementation detail here — that belongs in slice objectives.]

## Diagnosis Summary

[INVESTIGATE mode: one paragraph summarizing root cause from Phase 1.
PLAN mode: one paragraph describing the known problem and its mechanism.
This is the context that makes slice objectives legible.]

## Slices

---

### Slice 1: [Title]

**Objective:** [One sentence: what this slice proves, not what it implements.]

#### Acceptance Criteria

- AC-1: [verbatim, passes sufficiency test]
- AC-2: [verbatim, passes sufficiency test]
- AC-N: [verbatim, passes sufficiency test]

---

### Slice 2: [Title]

**Objective:** [One sentence.]

#### Acceptance Criteria

- AC-1: [verbatim]
- AC-N: [verbatim]

---

[Continue for each slice]

## Constraints

[Project-specific constraints: Python path, test runner, isolation flags,
forbidden commands, environment restrictions.]

## Out of Scope

[Explicit list of related work NOT included in this plan — prevents scope creep
in tdd-slice Phase 2]
```

**Parser compatibility requirements (do not deviate):**
- Slice headings: exactly `### Slice N:` (three hashes, capital S, number, colon)
- AC section heading: exactly `#### Acceptance Criteria` (four hashes)
- AC list items: `- AC-N:` prefix on every criterion
- No other `#### Acceptance Criteria` sections outside slice blocks

---

## Phase 6 — Present, Invoke qa-plan Gate, Await Human Approval

Present the complete plan including:
- Phase 0 research summary with citations
- Phase 1 diagnosis document (INVESTIGATE mode)
- Phase 4 self-review findings table (all checks MET)
- Full issue body written to the plan file

Then update the issue body and invoke the qa-plan gate:

```bash
# Update issue body per posting protocol
{POSTING_SCRIPT} update-issue {OWNER}/{REPO} {ISSUE_NUMBER} {plan_file}

# Invoke the qa-plan gate
{GATE_SCRIPT} plan {OWNER}/{REPO} {ISSUE_NUMBER}
```

Run that command as one blocking terminal call. Wait up to 300 seconds for it to exit before treating it as failed or stuck.

**Wait for the gate runner to complete.** Do not proceed until it exits. Do not poll with alternate commands, do not inspect partial output mid-run, and do not self-evaluate the gate while it is still running.

**⛔ QA Verdict Provenance Verification (MANDATORY):**
After the gate runner exits, before acting on PASS or HOLD, verify the verdict comment was posted by `{QA_BOT}`:
```bash
VERDICT_AUTHOR=$(gh api /repos/{OWNER}/{REPO}/issues/{ISSUE_NUMBER}/comments?per_page=100 \
  --jq '[.[] | select(.body | test("Decision\\("))] | last | .user.login')
```
The author MUST be `{QA_BOT}`. If it is any other value:
- STOP immediately
- Report: `"FATAL: QA verdict provenance failure. Expected: {QA_BOT}, Got: {VERDICT_AUTHOR}. Verdict was not posted via the authorized gate runner."`
- Do NOT act on the verdict
- Do NOT proceed to Phase 7

- Do NOT interpret a human saying "looks good" as a qa-plan PASS — qa-plan is a separate gate
- Do NOT proceed to Phase 7 on a qa-plan HOLD

**On qa-plan HOLD:**
Address every NOT MET finding. Re-run Phase 3 (affected ACs), re-run Phase 4 (full 12-point self-review), update the issue body, re-invoke the gate runner. Do not resubmit until Phase 4 is clean.

**On qa-plan PASS:**
Present the PASS to the human and wait for explicit approval: "approved," "do it," "go," or "start." qa-plan PASS + human approval together unlock Phase 7. Neither alone is sufficient.

---

## Phase 7 — Issue Finalization (Post-Approval Only)

On explicit approval:

Update the issue body per the GitHub Posting Protocol:
```bash
{POSTING_SCRIPT} update-issue {OWNER}/{REPO} {ISSUE_NUMBER} {plan_file}
```

**⛔ Post Provenance Verification (MANDATORY):**
After updating the issue body, verify the update landed correctly:
```bash
UPDATED_BY=$(gh api graphql -f query='query($owner:String!,$repo:String!,$num:Int!){repository(owner:$owner,name:$repo){issue(number:$num){editor{login}}}}' \
  -F owner={OWNER} -F repo={REPO} -F num={ISSUE_NUMBER} \
  --jq '.data.repository.issue.editor.login')
```
The login must be `{TDD_BOT}`. If it is any other value or null, the update was not made via the authorized protocol. STOP and report the provenance failure.

Confirm the issue number to the human. That issue number is what `tdd-slice` and `qa` will operate against. Do not begin Slice 1 — the `tdd-slice` skill handles execution.

**Label transition:** On qa-plan PASS + human approval, swap label to `tdd-slice`:
```bash
gh issue edit {ISSUE_NUMBER} -R {OWNER}/{REPO} --remove-label "qa-plan" --add-label "tdd-slice"
```

---

## Exit Conditions

| User/gate action | Response |
|:--|:--|
| qa-plan PASS + human "approved/do it/go" | Phase 7 — finalize issue body, report number, stop |
| qa-plan HOLD | Fix NOT MET findings, re-run Phase 3–4, re-present, re-invoke qa-plan |
| Human "approved" before qa-plan PASS | Decline — "qa-plan has not cleared this plan yet" |
| Human asks questions about the plan | Answer, revise if needed, repeat Phase 4, re-invoke qa-plan |
| Human requests AC changes | Revise, re-run AC sufficiency test, re-run Phase 4, re-invoke qa-plan |
| "Start Slice 1" | That is `tdd-slice` territory — hand off with the issue number |
| "Investigate first" | INVESTIGATE mode — run Phase 0 + Phase 1 before planning |

---

## Integration Reference

This skill is the first leg of the plan → execute → qa stool.

**Downstream interlocks:**

The `tdd-slice` skill reads `### Slice N:` + `#### Acceptance Criteria` from the issue body. It uses the AC list verbatim as its Phase 1 contract. Any deviation from the format silently breaks the contract — the tdd-slice skill will either write wrong ACs or HOLD at Phase 1.

The `qa` skill fetches the issue body and parses `### Slice N:` + `#### Acceptance Criteria` as the gate contract. It evaluates the tdd-slice submission against this contract item by item. If the plan ACs are weak (generic, non-diagnostic), the gate will PASS work that does not actually prove correctness. The quality ceiling of the entire pipeline is the quality of the ACs written here.

**The planning skill is the only place in the stool where correctness is defined. It cannot be delegated, approximated, or inferred from execution results.**
