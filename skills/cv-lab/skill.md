---
name: cv-lab
description: Academic paper review pipeline for computer vision LaTeX papers with professor, peer, and official reviewers
aliases:
  - cv_lab
  - paper-review
argument-hint: <path-to-project-directory> [--threshold N] [--max-rounds M]
---

# CV Lab - Computer Vision Paper Review Pipeline

[CV_LAB REVIEW PIPELINE - ROUND {{ROUND}}/{{MAX_ROUNDS}}]

## CRITICAL EXECUTION RULES - YOU MUST FOLLOW THESE

**YOU ARE AN AUTONOMOUS REVIEW PIPELINE. DO NOT STOP UNTIL THE PIPELINE COMPLETES.**

Each round has 5 phases that MUST execute in sequence. After completing one phase, you MUST immediately proceed to the next. DO NOT pause, ask the user, or stop between phases.

```
Phase 0: Initialize (first round only) — add \usepackage{xcolor} to preamble
Phase 1: Professor Review        → then IMMEDIATELY proceed to Phase 2
Phase 2: 4 Peer Reviews (parallel) → then IMMEDIATELY proceed to Phase 3
Phase 3: Plan & Revise           → revisions in \textcolor{blue}{...} → IMMEDIATELY proceed to Phase 4
Phase 4: Official Review (1-10)  → then IMMEDIATELY proceed to Phase 5
Phase 5: Decision Gate           → if score >= threshold: OUTPUT [PROMISE:CV_LAB_COMPLETE]
                                  → if max rounds reached: OUTPUT [PROMISE:CV_LAB_MAX_ROUNDS]
                                  → if score < threshold: PAUSE for user confirmation
                                    → Show revision summary + score
                                    → Wait for user to say "continue" or "stop"
                                    → On "continue": START NEXT ROUND at Phase 1
```

**REVISION VISIBILITY RULE:**
All text changes made during Phase 3 (Plan & Revise) MUST be wrapped in `\textcolor{blue}{...}` so the user can visually identify what was revised in the compiled PDF. This includes:
- New sentences or paragraphs: `\textcolor{blue}{New text here}`
- Replaced text: Delete old text, insert `\textcolor{blue}{replacement text}`
- New equations: Wrap in `\textcolor{blue}{$...$}` or `{\color{blue} \begin{equation}...\end{equation}}`
- New table rows/cells: Wrap cell content in `\textcolor{blue}{...}`
- New citations: `\textcolor{blue}{\cite{new_ref}}`

At the start of each new round, REMOVE the `\textcolor{blue}{...}` wrappers from the PREVIOUS round's revisions (they are now accepted baseline text). Only the CURRENT round's revisions should be blue.

**STOP CONDITIONS (only these):**
- `[PROMISE:CV_LAB_COMPLETE]` - Official review score >= threshold. Paper accepted.
- `[PROMISE:CV_LAB_MAX_ROUNDS]` - Max rounds exhausted. Paper not accepted.
- User says "stop", "cancel", or "abort"
- Phase 5 Decision Gate when score < threshold: PAUSE and wait for user confirmation

**NEVER stop after just Phase 1 (Professor Review). NEVER stop after just Phase 2 (Peer Reviews). NEVER stop without an Official Review score. If you find yourself about to stop without outputting a PROMISE tag AND without waiting for user confirmation at the Decision Gate, YOU ARE NOT DONE - continue to the next phase.**

---

Orchestrate a rigorous academic paper review loop for LaTeX **project directories** in computer vision. Simulates the full conference review process with a professor advisor, 4 specialized peer reviewers, and an official conference reviewer.

## Usage

```
/claude-labs:cv-lab /path/to/paper_project/
/claude-labs:cv-lab /path/to/paper_project/ --threshold 8
/claude-labs:cv-lab ./my-eccv-paper --threshold 7 --max-rounds 5
/claude-labs:cv-lab . --threshold 6
```

### Parameters

- **path** - Path to the LaTeX project directory (required). The directory should contain a `main.tex` (or similar root `.tex` file) and may include subdirectories for sections, images, bibliography, and style files.
- **--threshold N** - Minimum official review score (1-10) to accept the paper. Default: 7.
- **--max-rounds M** - Maximum review-revise cycles before stopping. Default: 5.

### Expected Project Structure

The skill auto-discovers the project layout. A typical CV paper project looks like:

```
paper_project/
  main.tex                      # Root .tex file (entry point)
  main.bib                      # Bibliography
  src/
    sec/
      1. introduction.tex       # Section files (any naming)
      2. related works.tex
      3. preliminaries.tex
      4. proposed method.tex
      5. experiments.tex
      6. conclusion.tex
      x_supplementary.tex
    imgs/
      figure1.pdf               # Figures (PDF, PNG, EPS)
      ...
  eccv.sty                      # Style files
  llncs.cls                     # Document class
  splncs04.bst                  # Bibliography style
```

The skill handles any layout: flat (all .tex in root), nested (sections in subdirectories), or multi-file with `\input{}`/`\include{}` references.

## Architecture

```
User: "/cv-lab /path/to/paper_project/ --threshold 7"
              |
              v
      [CV LAB ORCHESTRATOR]
              |
    +---------+---------+
    |                   |
    v                   v
  Parse Args        Discover project
    |               (find main.tex,
    |                scan \input{},
    |                list sections,
    |                find .bib, imgs)
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

1. **Parse arguments**: Extract project directory path, threshold (default: 7), max-rounds (default: 5)
2. **Discover project structure**:
   - Use `Glob` to find all `.tex` files in the project directory recursively (`**/*.tex`)
   - Identify the **root .tex file** (typically `main.tex` — look for `\documentclass` or `\begin{document}`)
   - Parse the root .tex for `\input{}` and `\include{}` to map the section dependency tree
   - Find all `.bib` files (`**/*.bib`) for bibliography context
   - Find all image directories (`**/imgs/`, `**/figures/`, `**/fig/`) and list figure files
   - Find style files (`.sty`, `.cls`, `.bst`) to identify the target venue (e.g., `eccv.sty` → ECCV)
3. **Create state directory**: `.cv-lab/` inside the project directory
4. **Initialize state**:
   ```json
   {
     "project_dir": "/path/to/paper_project",
     "root_tex": "main.tex",
     "all_tex_files": [
       "main.tex",
       "src/sec/1. introduction.tex",
       "src/sec/2. related works.tex",
       "src/sec/3. preliminaries.tex",
       "src/sec/4. proposed method.tex",
       "src/sec/5. experiments.tex",
       "src/sec/6. conclusion.tex"
     ],
     "bib_files": ["main.bib"],
     "image_dirs": ["src/imgs"],
     "detected_venue": "ECCV",
     "threshold": 7,
     "max_rounds": 5,
     "current_round": 0,
     "scores": [],
     "status": "in_progress"
   }
   ```
5. **Backup original**: Copy the entire project to `.cv-lab/backup/round-0/` (excluding `.cv-lab/` itself)
6. **Add xcolor package**: Check if `\usepackage{xcolor}` or `\usepackage{color}` already exists in the root .tex preamble (before `\begin{document}`). If NOT present, add `\usepackage{xcolor}` to the preamble using the Edit tool. This enables `\textcolor{blue}{...}` for revision visibility.

**IMPORTANT**: When providing paper content to reviewers, concatenate ALL section .tex files in order (following `\input{}` order from root), and include the bibliography content from `.bib` files. Each section should be clearly labeled with its filename for precise inline comments:
```
=== FILE: main.tex ===
[content]

=== FILE: src/sec/1. introduction.tex ===
[content]

=== FILE: src/sec/2. related works.tex ===
[content]
...

=== BIBLIOGRAPHY: main.bib ===
[content]
```

### Phase 1: Professor Review [DO NOT STOP AFTER THIS PHASE]

The professor reviews the paper with defensive scrutiny. This is the advisor's pre-submission check.

**Agent**: Use `Agent` tool with a prompt that includes the full professor agent instructions from `agents/professor.md`.

**Prompt template**:
```
You are the Professor Reviewer for a computer vision paper.

[Include full professor agent prompt]

PROJECT STRUCTURE:
- Root: {{PROJECT_DIR}}
- Venue: {{DETECTED_VENUE}} (detected from style files)
- Sections: {{ALL_TEX_FILES}}
- Bibliography: {{BIB_FILES}}
- Figure directories: {{IMAGE_DIRS}}

PAPER TO REVIEW (all sections concatenated in order):
=== FILE: main.tex ===
[content of main.tex]

=== FILE: src/sec/1. introduction.tex ===
[content]

=== FILE: src/sec/2. related works.tex ===
[content]
... (all section files in \input{} order)

=== BIBLIOGRAPHY: main.bib ===
[content of .bib file]

REVIEW ROUND: {{ROUND}}/{{MAX_ROUNDS}}
{{IF ROUND > 1}}
PREVIOUS ROUND SCORE: {{PREV_SCORE}}/10
PREVIOUS OFFICIAL REVIEW SUMMARY: {{PREV_OFFICIAL_SUMMARY}}
Focus especially on issues flagged in the previous official review that remain unresolved.
{{ENDIF}}

Instructions:
1. Read the complete paper carefully (all section files)
2. Insert inline LaTeX comments using % [PROFESSOR] format at relevant locations
3. When commenting, specify WHICH FILE the comment belongs to:
   % [PROFESSOR] CRITICAL (src/sec/4. proposed method.tex): This claim is not supported...
4. Provide your review summary
5. For EACH file you have comments on, output the modified content with inline comments
```

**After professor review**:
- Apply the professor's inline comments to the actual .tex files in the project using the Edit tool
- Save a combined annotated version to `.cv-lab/round-{{N}}/after-professor/` (mirror the project structure)
- Extract and log all `% [PROFESSOR]` comments to `.cv-lab/round-{{N}}/professor-summary.md`

**>>> PHASE 1 COMPLETE. DO NOT STOP. IMMEDIATELY PROCEED TO PHASE 2: PEER REVIEW <<<**

### Phase 2: Peer Review (4 Parallel Reviewers) [DO NOT STOP AFTER THIS PHASE]

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

PROJECT STRUCTURE:
- Root: {{PROJECT_DIR}}
- Venue: {{DETECTED_VENUE}}
- Sections: {{ALL_TEX_FILES}}
- Bibliography: {{BIB_FILES}}
- Figure directories: {{IMAGE_DIRS}}
- Figures available: {{LIST_OF_FIGURE_FILES}}

PAPER TO REVIEW (with professor comments already inserted):
=== FILE: main.tex ===
[content with professor comments]

=== FILE: src/sec/1. introduction.tex ===
[content with professor comments]
... (all section files)

=== BIBLIOGRAPHY: main.bib ===
[content]

REVIEW ROUND: {{ROUND}}/{{MAX_ROUNDS}}

Instructions:
1. Search for 3-5 related papers relevant to your focus area using WebSearch
2. Review the paper from your specialized perspective
3. Insert inline LaTeX comments specifying the target file:
   % [PEER-{{REVIEWER_ID}}] MAJOR (src/sec/3. preliminaries.tex): Equation 3 is missing...
4. For Reviewer 3 (Figures): Also review figure filenames and how they are referenced
   in the .tex files (\includegraphics paths)
5. For Reviewer 4 (Related Works): Also review the .bib file for completeness
6. Provide your review summary with focus area score (1-10)
7. For EACH file you have comments on, output the modified content
   (preserve existing % [PROFESSOR] comments)
```

**After peer review**:
- Apply each reviewer's inline comments to the actual .tex files using the Edit tool
- Save combined annotated versions to `.cv-lab/round-{{N}}/after-peers/` (mirror project structure)
- Extract and log all `% [PEER-*]` comments to per-reviewer summaries

**Comment merging strategy**:
- Read each reviewer's output
- For each `% [PEER-N]` comment, identify the target file (from the file annotation in the comment)
- Apply comments to the correct .tex file in the project
- If two reviewers comment on the same location, stack comments in reviewer order
- Preserve all existing `% [PROFESSOR]` comments

**>>> PHASE 2 COMPLETE. DO NOT STOP. IMMEDIATELY PROCEED TO PHASE 3: PLAN & REVISE <<<**

### Phase 3: Plan & Revise [DO NOT STOP AFTER THIS PHASE]

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

Apply revisions to the .tex files **in the project directory** using the Edit tool:
- Fix CRITICAL issues first, then MAJOR, then MINOR
- Edit the actual section files (e.g., `src/sec/4. proposed method.tex`), NOT a concatenated copy

**BLUE TEXT RULE - ALL revisions MUST be visually marked:**
- **New text**: Wrap in `\textcolor{blue}{new text here}`
- **Replaced text**: Delete old text, insert `\textcolor{blue}{replacement text}`
- **New equations**: `{\color{blue} \begin{equation}...\end{equation}}`  or inline `\textcolor{blue}{$...$}`
- **New table content**: `\textcolor{blue}{cell content}`
- **New citations**: `\textcolor{blue}{\cite{new_ref}}`
- **New figures/captions**: `\textcolor{blue}{caption text}`

Example revision:
```latex
% BEFORE:
Our method achieves state-of-the-art performance on all benchmarks.

% AFTER:
\textcolor{blue}{Our method achieves competitive performance on COCO and VOC benchmarks,
ranking first on AP$_{50}$ and second on AP$_{75}$ (see Table~\ref{tab:main}).}
```

**If this is Round 2+**: Before applying new revisions, first REMOVE all `\textcolor{blue}{...}` wrappers from the previous round (keep the text inside, just remove the blue wrapper). Use search-and-replace to find `\textcolor{blue}{` patterns. Only the CURRENT round's changes should be blue.

- Mark addressed review comments as resolved:
  ```latex
  % [PROFESSOR] RESOLVED (was CRITICAL): Added statistical significance tests to Table 2.
  % Previously: "Claims not supported by Table 2 results"
  ```
- Keep unresolved comments intact
- When adding new content (e.g., new experiments, citations), also update `main.bib` if new references are needed

**Step 3c: Verify Compilation**

After revisions, verify LaTeX still compiles from the project directory:
```bash
cd {{PROJECT_DIR}}
# Try compilation check (non-blocking if tools not available)
pdflatex -interaction=nonstopmode -halt-on-error main.tex
# Or if latexmk is available:
latexmk -pdf -interaction=nonstopmode -halt-on-error main.tex
```

If compilation fails, fix the LaTeX errors in the affected files before proceeding.

Save a snapshot of all revised .tex files to `.cv-lab/round-{{N}}/after-revision/` (mirror project structure)

**>>> PHASE 3 COMPLETE. DO NOT STOP. IMMEDIATELY PROCEED TO PHASE 4: OFFICIAL REVIEW <<<**

### Phase 4: Official Review [THIS PHASE PRODUCES THE SCORE]

The official reviewer evaluates the revised paper and produces a 1-10 rating.

**Agent**: Use `Agent` tool with the official-reviewer prompt.

**Prompt template**:
```
You are the Official Reviewer for a top-tier computer vision conference.

[Include full official-reviewer agent prompt]

PROJECT: {{PROJECT_DIR}}
VENUE: {{DETECTED_VENUE}}

PAPER TO REVIEW (after revision, all sections):
=== FILE: main.tex ===
[revised content]

=== FILE: src/sec/1. introduction.tex ===
[revised content]
... (all section files with any remaining review comments)

=== BIBLIOGRAPHY: main.bib ===
[content]

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

### Phase 5: Decision Gate [THE ONLY PHASE WHERE STOPPING IS ALLOWED]

**Extract the official review score and evaluate:**

```
if official_score >= threshold:
    status = "ACCEPTED"
    -> Update state.json with final score and status="accepted"
    -> Generate final report to .cv-lab/final-report.md
    -> Output: [PROMISE:CV_LAB_COMPLETE]
    -> Print: "Paper ACCEPTED with score {score}/{threshold}. Review complete after {N} round(s)."
    -> EXIT

elif current_round >= max_rounds:
    status = "MAX_ROUNDS_REACHED"
    -> Update state.json with status="max_rounds_reached"
    -> Generate final report with remaining issues to .cv-lab/final-report.md
    -> Output: [PROMISE:CV_LAB_MAX_ROUNDS]
    -> Print: "Max rounds ({max_rounds}) reached. Final score: {score}/{threshold}. See .cv-lab/final-report.md"
    -> EXIT

else:
    -> Update state.json with incremented current_round
    -> Snapshot current project state to .cv-lab/round-{{N}}/
    -> PAUSE AND PRESENT REVISION SUMMARY TO USER:

    Print the following:
    ---
    ## Round {{N}} Complete — Score: {{score}}/10 (threshold: {{threshold}})

    ### Official Review Summary
    [Key strengths and weaknesses from official review]

    ### Revisions Made This Round (shown in blue in the .tex files)
    [List of changes made, organized by file]

    ### Remaining Issues for Next Round
    [Unresolved comments that will be addressed]

    ### Score Progression
    | Round | Score | Decision |
    |-------|-------|----------|
    [score history table]

    **The revised paper has blue text marking all changes. Please review the .tex files or compile the PDF to inspect.**

    **Type "continue" to start Round {{N+1}}, or "stop" to end the review.**
    ---

    -> WAIT for user response
    -> If user says "continue", "yes", "go", "next", or similar:
       current_round += 1
       -> LOOP BACK TO PHASE 1 (Professor Review)
       -> At the START of the new round, first remove previous round's \textcolor{blue}{...} wrappers
       -> Professor focuses on remaining issues from the official review
    -> If user says "stop", "cancel", "abort", "no", or similar:
       -> Output: [PROMISE:CV_LAB_USER_STOPPED]
       -> Print: "Review stopped by user at Round {{N}}. Final score: {{score}}/10."
       -> Generate final report to .cv-lab/final-report.md
       -> EXIT
```

**CRITICAL: If you reach this point without a PROMISE tag AND without pausing for user confirmation, YOU ARE NOT DONE - you must either output a PROMISE tag or ask the user to continue.**

## State Management

### Directory Structure

The `.cv-lab/` directory is created **inside the paper project directory**:

```
paper_project/
  main.tex                        # (edited in-place by revisions)
  main.bib
  src/sec/*.tex                   # (edited in-place by revisions)
  src/imgs/*.pdf
  .cv-lab/                        # Review state (add to .gitignore)
    state.json                    # Overall pipeline state
    backup/
      round-0/                    # Original project snapshot
        main.tex
        main.bib
        src/sec/1. introduction.tex
        src/sec/2. related works.tex
        ...
    round-1/
      after-professor/            # Snapshot after professor comments
        main.tex
        src/sec/*.tex
      after-peers/                # Snapshot after peer comments merged
        main.tex
        src/sec/*.tex
      revision-plan.md            # Prioritized revision plan
      after-revision/             # Snapshot after revisions applied
        main.tex
        main.bib
        src/sec/*.tex
      after-official/             # Snapshot after official comments
        main.tex
        src/sec/*.tex
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
  "project_dir": "/path/to/paper_project",
  "root_tex": "main.tex",
  "all_tex_files": [
    "main.tex",
    "src/sec/1. introduction.tex",
    "src/sec/2. related works.tex",
    "src/sec/3. preliminaries.tex",
    "src/sec/4. proposed method.tex",
    "src/sec/5. experiments.tex",
    "src/sec/6. conclusion.tex"
  ],
  "bib_files": ["main.bib"],
  "image_dirs": ["src/imgs"],
  "detected_venue": "ECCV",
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
**Project**: {{PROJECT_DIR}}
**Venue**: {{DETECTED_VENUE}}
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

- **Resume**: Run `/claude-labs:cv-lab /path/to/project/` again. If `.cv-lab/state.json` exists inside the project with `status: "in_progress"`, resume from last completed phase.
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
/cv-lab /Users/kylee/Downloads/_ECCV26__Density_DPP/
```
Reviews the entire project with default threshold of 7 and max 5 rounds.

### Current Directory
```
/cv-lab .
```
Reviews the LaTeX project in the current working directory.

### High Standards
```
/cv-lab /path/to/paper/ --threshold 8
```
Requires score of 8+ (clear accept) before stopping.

### Quick Check
```
/cv-lab /path/to/paper/ --threshold 5 --max-rounds 2
```
Lower bar, fewer rounds - good for early drafts.

### Resume Interrupted Review
```
/cv-lab /path/to/paper/
```
If `.cv-lab/state.json` exists inside the project, resumes from last completed phase.
