---
name: cv-lab
description: Academic paper review pipeline for computer vision LaTeX papers with professor, peer, and official reviewers
aliases:
  - cv_lab
  - paper-review
argument-hint: <path-to-tex-file> [--threshold N] [--max-rounds M]
---

# CV Lab - Computer Vision Paper Review Pipeline

Orchestrate a rigorous academic paper review loop for LaTeX papers in computer vision. Simulates the full conference review process with a professor advisor, 4 specialized peer reviewers, and an official conference reviewer.

## Usage

```
/claude-labs:cv-lab paper.tex
/claude-labs:cv-lab paper.tex --threshold 8
/claude-labs:cv-lab paper.tex --threshold 7 --max-rounds 5
/claude-labs:cv-lab main.tex --threshold 6
```

### Parameters

- **path** - Path to the main .tex file (required). Can be relative or absolute.
- **--threshold N** - Minimum official review score (1-10) to accept the paper. Default: 7.
- **--max-rounds M** - Maximum review-revise cycles before stopping. Default: 5.

## Architecture

```
User: "/cv-lab paper.tex --threshold 7"
              |
              v
      [CV LAB ORCHESTRATOR]
              |
    +---------+---------+
    |                   |
    v                   v
  Parse Args        Read .tex files
    |                   |
    +-------------------+
              |
              v
  ====== REVIEW LOOP (until score >= threshold) ======
  |                                                   |
  |  Round 1: PROFESSOR REVIEW                        |
  |    -> Read full paper                             |
  |    -> Defensive review (tone, consistency,        |
  |       arguments, experiments)                     |
  |    -> Insert inline % [PROFESSOR] comments        |
  |    -> Output summary + verdict                    |
  |                                                   |
  |  Round 2: PEER REVIEW (4 reviewers in parallel)   |
  |    -> Reviewer 1: Architecture specialist          |
  |    -> Reviewer 2: Mathematics specialist           |
  |    -> Reviewer 3: Figures & visualization          |
  |    -> Reviewer 4: Related works & positioning      |
  |    Each reviewer:                                  |
  |      - Searches for different related papers       |
  |      - Reviews from their focus area               |
  |      - Inserts inline % [PEER-N] comments          |
  |                                                   |
  |  Round 3: PLAN & REVISE                            |
  |    -> Aggregate all feedback (professor + peers)   |
  |    -> Prioritize issues by severity                |
  |    -> Create revision plan                         |
  |    -> Execute revisions on the .tex files          |
  |    -> Verify no LaTeX compilation breaks           |
  |                                                   |
  |  Round 4: OFFICIAL REVIEW                          |
  |    -> Read revised paper + all prior comments      |
  |    -> Score on 5 dimensions (weighted)             |
  |    -> Produce final rating (1-10)                  |
  |    -> Insert inline % [OFFICIAL] comments          |
  |                                                   |
  |  Decision Gate:                                    |
  |    if rating >= threshold: ACCEPT -> exit loop     |
  |    if rating < threshold AND round < max_rounds:   |
  |      -> Loop back to PROFESSOR REVIEW              |
  |    if round >= max_rounds: STOP with final report  |
  |                                                   |
  ====================================================
              |
              v
      [FINAL REPORT]
        - Review history across all rounds
        - Score progression
        - Remaining issues
        - Acceptance decision
```

## Detailed Workflow

### Phase 0: Initialization

1. **Parse arguments**: Extract .tex path, threshold (default: 7), max-rounds (default: 5)
2. **Discover LaTeX files**: From the main .tex, find all `\input{}` and `\include{}` referenced files
3. **Create state directory**: `.cv-lab/` in the paper's directory
4. **Initialize state**:
   ```json
   {
     "paper_path": "paper.tex",
     "threshold": 7,
     "max_rounds": 5,
     "current_round": 0,
     "scores": [],
     "status": "in_progress"
   }
   ```
5. **Backup original**: Copy all .tex files to `.cv-lab/backup/round-0/`

### Phase 1: Professor Review

The professor reviews the paper with defensive scrutiny. This is the advisor's pre-submission check.

**Agent**: Use `Agent` tool with a prompt that includes the full professor agent instructions from `agents/professor.md`.

**Prompt template**:
```
You are the Professor Reviewer for a computer vision paper.

[Include full professor agent prompt]

PAPER TO REVIEW:
[Full content of main .tex file and all included files]

REVIEW ROUND: {{ROUND}}/{{MAX_ROUNDS}}
{{IF ROUND > 1}}
PREVIOUS ROUND SCORE: {{PREV_SCORE}}/10
PREVIOUS OFFICIAL REVIEW SUMMARY: {{PREV_OFFICIAL_SUMMARY}}
Focus especially on issues flagged in the previous official review that remain unresolved.
{{ENDIF}}

Instructions:
1. Read the complete paper carefully
2. Insert inline LaTeX comments using % [PROFESSOR] format at relevant locations
3. Provide your review summary
4. Output the COMPLETE modified .tex content with your inline comments inserted
```

**After professor review**:
- Save the annotated .tex to `.cv-lab/round-{{N}}/after-professor.tex`
- Extract and log all `% [PROFESSOR]` comments

### Phase 2: Peer Review (4 Parallel Reviewers)

Spawn 4 peer reviewers simultaneously, each with a different focus area.

**Agents**: Launch 4 `Agent` calls in parallel, each with the peer-reviewer prompt specialized for their focus.

**CRITICAL**: All 4 reviewers MUST be launched in parallel (not sequentially).

**Reviewer assignments**:

| Reviewer | Focus | Related Paper Search Strategy |
|----------|-------|-------------------------------|
| Peer-1 | Architecture | Search for papers on the specific architecture type (ViT variants, CNN designs, hybrid models) |
| Peer-2 | Mathematics | Search for papers on the loss functions, optimization techniques, theoretical foundations used |
| Peer-3 | Figures & Viz | Search for benchmark papers in the subfield to compare presentation quality |
| Peer-4 | Related Works | Search broadly for missing citations, concurrent work, overlooked prior art |

**Prompt template for each**:
```
You are Peer Reviewer #{{REVIEWER_ID}} ({{FOCUS_AREA}} Specialist) for a computer vision paper.

[Include peer-reviewer agent prompt with REVIEWER_ID substituted]

PAPER TO REVIEW (with professor comments already inserted):
[Content from after-professor.tex]

REVIEW ROUND: {{ROUND}}/{{MAX_ROUNDS}}

Instructions:
1. Search for 3-5 related papers relevant to your focus area using WebSearch
2. Review the paper from your specialized perspective
3. Insert inline LaTeX comments using % [PEER-{{REVIEWER_ID}}] format
4. Provide your review summary with focus area score (1-10)
5. Output the COMPLETE modified .tex content with your inline comments added
   (preserve existing % [PROFESSOR] comments)
```

**After peer review**:
- Merge all 4 reviewers' comments into a single annotated .tex
- Save to `.cv-lab/round-{{N}}/after-peers.tex`
- Extract and log all `% [PEER-*]` comments

**Comment merging strategy**:
- Read each reviewer's output
- For each `% [PEER-N]` comment block, identify its location by surrounding LaTeX context
- Insert all comments into the consolidated file, preserving order
- If two reviewers comment on the same line, stack comments in reviewer order

### Phase 3: Plan & Revise

Aggregate all feedback and systematically revise the paper.

**Step 3a: Create Revision Plan**

Read the annotated .tex with all professor + peer comments. Create a prioritized revision plan:

```markdown
## Revision Plan - Round {{N}}

### Critical Issues (Must Fix)
1. [Issue from PROFESSOR/PEER with location] -> [Planned fix]
2. ...

### Major Issues (Should Fix)
1. [Issue] -> [Planned fix]
2. ...

### Minor Issues (Nice to Fix)
1. [Issue] -> [Planned fix]
2. ...

### Deferred Issues (Out of scope for this round)
1. [Issue] -> [Why deferred]
```

Save plan to `.cv-lab/round-{{N}}/revision-plan.md`

**Step 3b: Execute Revisions**

Apply revisions to the .tex files using the Edit tool:
- Fix CRITICAL issues first, then MAJOR, then MINOR
- Remove addressed review comments (change `% [PROFESSOR] CRITICAL:` to `% [PROFESSOR] RESOLVED:`)
- Keep unresolved comments intact
- Do NOT remove the original comment - mark it as resolved with the fix description:
  ```latex
  % [PROFESSOR] RESOLVED (was CRITICAL): Added statistical significance tests to Table 2.
  % Previously: "Claims not supported by Table 2 results"
  ```

**Step 3c: Verify Compilation**

After revisions, verify LaTeX still compiles (if pdflatex/latexmk is available):
```bash
# Try compilation check (non-blocking if tools not available)
pdflatex -interaction=nonstopmode -halt-on-error paper.tex
```

If compilation fails, fix the LaTeX errors before proceeding.

Save revised .tex to `.cv-lab/round-{{N}}/after-revision.tex`

### Phase 4: Official Review

The official reviewer evaluates the revised paper and produces a 1-10 rating.

**Agent**: Use `Agent` tool with the official-reviewer prompt.

**Prompt template**:
```
You are the Official Reviewer for a top-tier computer vision conference.

[Include full official-reviewer agent prompt]

PAPER TO REVIEW (after revision):
[Content from after-revision.tex]

REVIEW ROUND: {{ROUND}}/{{MAX_ROUNDS}}
ACCEPTANCE THRESHOLD: {{THRESHOLD}}/10

PROFESSOR REVIEW SUMMARY:
{{PROFESSOR_SUMMARY}}

PEER REVIEW SUMMARIES:
{{PEER_1_SUMMARY}}
{{PEER_2_SUMMARY}}
{{PEER_3_SUMMARY}}
{{PEER_4_SUMMARY}}

REVISION PLAN:
{{REVISION_PLAN}}

{{IF ROUND > 1}}
PREVIOUS SCORES: {{SCORE_HISTORY}}
PREVIOUS ROUND ISSUES REMAINING: {{REMAINING_ISSUES}}
{{ENDIF}}

Instructions:
1. Read the revised paper and all prior review feedback
2. Assess whether prior issues were adequately addressed
3. Score each dimension (Novelty, Technical Soundness, Experiments, Clarity, Significance)
4. Compute weighted overall score
5. Insert inline % [OFFICIAL] comments
6. Provide HONEST rating (1-10) - do not inflate to end the loop
7. Output the final annotated .tex and your structured review
```

**After official review**:
- Extract the numerical rating
- Save annotated .tex to `.cv-lab/round-{{N}}/after-official.tex`
- Save review to `.cv-lab/round-{{N}}/official-review.md`
- Update state:
  ```json
  {
    "scores": [..., {"round": N, "score": X, "decision": "ACCEPT/REVISE"}]
  }
  ```

### Phase 5: Decision Gate

```
if official_score >= threshold:
    status = "ACCEPTED"
    -> Generate final report
    -> Clean up review comments (optional: keep as documentation)
    -> Exit loop

elif current_round >= max_rounds:
    status = "MAX_ROUNDS_REACHED"
    -> Generate final report with remaining issues
    -> Exit loop

else:
    current_round += 1
    -> Copy current .tex as starting point for next round
    -> Loop back to Phase 1 (Professor Review)
    -> Professor focuses on remaining issues from official review
```

## State Management

### Directory Structure

```
.cv-lab/
  state.json                    # Overall pipeline state
  backup/
    round-0/                    # Original paper backup
      paper.tex
      sections/*.tex
  round-1/
    after-professor.tex         # Paper after professor comments
    after-peers.tex             # Paper after peer comments merged
    revision-plan.md            # Prioritized revision plan
    after-revision.tex          # Paper after revisions applied
    after-official.tex          # Paper after official comments
    official-review.md          # Official review with rating
    professor-summary.md        # Professor review summary
    peer-1-summary.md           # Peer reviewer summaries
    peer-2-summary.md
    peer-3-summary.md
    peer-4-summary.md
  round-2/
    ...
  final-report.md               # Aggregate report across all rounds
```

### State File Format

```json
{
  "paper_path": "paper.tex",
  "all_tex_files": ["paper.tex", "sections/intro.tex", "sections/method.tex"],
  "threshold": 7,
  "max_rounds": 5,
  "current_round": 2,
  "status": "in_progress",
  "scores": [
    {"round": 1, "score": 4, "decision": "REVISE", "summary": "Weak experiments, missing baselines"},
    {"round": 2, "score": 6, "decision": "REVISE", "summary": "Improved experiments, still needs ablation"}
  ],
  "comment_counts": {
    "round-1": {"professor": 12, "peer-1": 8, "peer-2": 6, "peer-3": 5, "peer-4": 9, "official": 7},
    "round-2": {"professor": 5, "peer-1": 3, "peer-2": 2, "peer-3": 1, "peer-4": 4, "official": 4}
  },
  "created_at": "2026-03-05T10:00:00Z",
  "updated_at": "2026-03-05T12:30:00Z"
}
```

## Final Report Template

Generated at `.cv-lab/final-report.md`:

```markdown
# CV Lab Review Report

**Paper**: {{PAPER_TITLE}}
**File**: {{PAPER_PATH}}
**Date**: {{DATE}}
**Rounds**: {{TOTAL_ROUNDS}}
**Final Status**: {{ACCEPTED / MAX_ROUNDS_REACHED}}
**Final Score**: {{FINAL_SCORE}}/10 (threshold: {{THRESHOLD}})

## Score Progression

| Round | Score | Decision | Key Issues |
|-------|-------|----------|------------|
| 1     | X/10  | REVISE   | ...        |
| 2     | X/10  | REVISE   | ...        |
| 3     | X/10  | ACCEPT   | ...        |

## Review History

### Round 1
- Professor: [Summary]
- Peer Reviewers: [Summary of each]
- Revisions: [What was changed]
- Official: [Score and key feedback]

### Round 2
...

## Remaining Issues
[Any unresolved comments still in the paper]

## Recommendation
[Final recommendation based on review trajectory]
```

## Configuration

Users can configure defaults in `.cv-lab/config.json`:

```json
{
  "threshold": 7,
  "max_rounds": 5,
  "venue": "CVPR",
  "focus_areas": {
    "peer-1": "architecture",
    "peer-2": "mathematics",
    "peer-3": "figures",
    "peer-4": "related-works"
  },
  "skip_compilation_check": false,
  "keep_resolved_comments": true
}
```

## Resume & Cancel

- **Resume**: Run `/claude-labs:cv-lab paper.tex` again. If `.cv-lab/state.json` exists with `status: "in_progress"`, resume from last completed phase.
- **Cancel**: Run `/claude-labs:cv-lab cancel` to stop the review loop. State is preserved for resume.

## Comment Tag Reference

| Tag | Source | Example |
|-----|--------|---------|
| `% [PROFESSOR]` | Professor review | `% [PROFESSOR] CRITICAL: Unsupported claim` |
| `% [PEER-1]` | Architecture reviewer | `% [PEER-1] MAJOR: Missing comparison to ViT` |
| `% [PEER-2]` | Mathematics reviewer | `% [PEER-2] CRITICAL: Eq. 3 has wrong gradient` |
| `% [PEER-3]` | Figures reviewer | `% [PEER-3] MINOR: Figure 2 text too small` |
| `% [PEER-4]` | Related works reviewer | `% [PEER-4] MISSING_REF: <paper> not cited` |
| `% [OFFICIAL]` | Official reviewer | `% [OFFICIAL] WEAKNESS: Limited ablation study` |
| `% [RESOLVED]` | Revision marker | `% [PROFESSOR] RESOLVED: Added significance tests` |

## Examples

### Basic Usage
```
/cv-lab paper.tex
```
Reviews paper.tex with default threshold of 7 and max 5 rounds.

### High Standards
```
/cv-lab paper.tex --threshold 8
```
Requires score of 8+ (clear accept) before stopping.

### Quick Check
```
/cv-lab paper.tex --threshold 5 --max-rounds 2
```
Lower bar, fewer rounds - good for early drafts.

### Resume Interrupted Review
```
/cv-lab paper.tex
```
If `.cv-lab/state.json` exists, resumes from last completed phase.
