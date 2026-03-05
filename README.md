# Claude Labs

A Claude Code plugin for academic paper review orchestration. Features a multi-agent review pipeline and interactive Q&A with expert reviewers for computer vision LaTeX papers.

| Skill | Description |
|-------|-------------|
| **CV Lab** (`cv-lab`) | Full review pipeline - professor, 4 peers, revisions, official review loop |
| **Ask Professor** (`ask-professor`) | Interactive Q&A with a senior CV professor about your paper |
| **Ask Peers** (`ask-peers`) | Interactive Q&A with 4 specialized peer reviewers who search related papers |

## Installation

```bash
# Add as a local marketplace
claude plugin marketplace add /path/to/claude_labs

# Install the plugin
claude plugin install claude-labs
```

## CV Lab - Paper Review Pipeline

CV Lab orchestrates a rigorous academic review loop with three types of reviewers:

| Reviewer | Model | Role |
|----------|-------|------|
| **Professor** | Opus | Defensive advisor review - tone, argument consistency, experimental rigor |
| **4 Peer Reviewers** | Sonnet | Specialized parallel reviewers - architecture, math, figures, related works |
| **Official Reviewer** | Opus | Conference-style review with weighted 1-10 rating |

### Usage

```bash
/claude-labs:cv-lab /path/to/paper_project/                  # Default: threshold=7, max-rounds=5
/claude-labs:cv-lab .                                        # Review project in current directory
/claude-labs:cv-lab /path/to/paper_project/ --threshold 8    # Require score 8+ (clear accept)
/claude-labs:cv-lab ./my-eccv-paper --threshold 6 --max-rounds 3   # Lenient, fewer rounds
/claude-labs:cv-lab /path/to/paper_project/ --threshold 5 --max-rounds 2   # Quick check for early drafts
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `path` | (required) | Path to the LaTeX **project directory** (containing `main.tex`, sections, figures, bib, etc.) |
| `--threshold N` | `7` | Minimum official review score (1-10) to accept |
| `--max-rounds M` | `5` | Maximum review-revise cycles |

### Expected Project Structure

The skill auto-discovers the project layout. A typical CV paper project:

```
paper_project/
  main.tex                    # Root .tex file (entry point)
  main.bib                    # Bibliography
  src/
    sec/
      1. introduction.tex     # Section files
      2. related works.tex
      3. preliminaries.tex
      4. proposed method.tex
      5. experiments.tex
      6. conclusion.tex
      x_supplementary.tex
    imgs/
      figure1.pdf             # Figures
      ...
  eccv.sty                    # Style files (auto-detects venue)
  llncs.cls                   # Document class
```

Any layout is supported: flat, nested, or multi-file with `\input{}`/`\include{}`.

### Review Pipeline

```
                    +-------------------+
                    | Professor Review  |
                    | (tone, arguments, |
                    |  experiments)     |
                    +---------+---------+
                              |
                              v
              +---+-----------+-----------+---+
              |   |           |           |   |
              v   v           v           v   |
          Peer-1  Peer-2   Peer-3    Peer-4   |
          Arch.   Math     Figures   Related   |
                                     Works    |
              +---+-----------+-----------+---+
                              |
                              v
                    +---------+---------+
                    |  Plan & Revise    |
                    |  (aggregate and   |
                    |   fix issues)     |
                    +---------+---------+
                              |
                              v
                    +---------+---------+
                    | Official Review   |
                    | (1-10 rating)     |
                    +---------+---------+
                              |
                     score >= threshold?
                        /          \
                      Yes           No
                       |             |
                       v             v
                    ACCEPT     Loop back to
                               Professor Review
```

### How Each Reviewer Works

#### Professor Review (Phase 1)

The professor acts as a protective advisor who wants the paper to succeed at a top venue. They challenge every weakness before external reviewers find them:

- **Tone Analysis** - Overclaiming, hedging, inconsistent voice, vague definitions
- **Argument Consistency** - Introduction vs. conclusion alignment, method vs. experiment match, logical gaps
- **Experimental Critique** - Missing baselines, cherry-picked metrics, no error bars, insufficient ablations
- **Experiment Suggestions** - Concrete experiments that reviewers will expect

Comments are inserted inline as LaTeX comments:
```latex
% [PROFESSOR] CRITICAL: The claim "state-of-the-art performance" in the abstract
% is not supported - Table 1 shows your method ranks 3rd on COCO.
```

#### Peer Review (Phase 2 - 4 Reviewers in Parallel)

Four specialized reviewers run simultaneously, each searching for **different related papers** to bring diverse perspectives:

| Reviewer | Focus Area | Search Strategy |
|----------|-----------|-----------------|
| **Peer-1** | Architecture | ViT variants, CNN designs, hybrid models, efficiency |
| **Peer-2** | Mathematics | Loss functions, optimization, theoretical foundations |
| **Peer-3** | Figures & Visualization | Presentation quality, diagram clarity, qualitative results |
| **Peer-4** | Related Works | Missing citations, concurrent work, novelty validation |

Each reviewer:
1. Searches for 3-5 related papers via web search
2. Reviews from their specialized perspective
3. Inserts inline comments: `% [PEER-1] MAJOR: ...`

#### Plan & Revise (Phase 3)

Aggregates all feedback, creates a prioritized revision plan (CRITICAL > MAJOR > MINOR), and applies fixes to the LaTeX source. Resolved comments are marked:
```latex
% [PROFESSOR] RESOLVED (was CRITICAL): Added statistical significance tests to Table 2.
```

#### Official Review (Phase 4)

Evaluates the revised paper with weighted scoring:

| Dimension | Weight |
|-----------|--------|
| Novelty & Originality | 25% |
| Technical Soundness | 25% |
| Experimental Validation | 25% |
| Clarity & Presentation | 15% |
| Significance & Impact | 10% |

Produces a final 1-10 rating following standard conference calibration:
- **8+**: Accept - strong contribution with good execution
- **7**: Weak Accept - above threshold, good contribution with some weaknesses
- **6**: Borderline - interesting idea but notable weaknesses
- **5-**: Below threshold - needs revision

### Rating Scale

| Score | Decision | Meaning |
|-------|----------|---------|
| 10 | Perfect | Groundbreaking (virtually never given) |
| 9 | Strong Accept | Excellent, significant contribution |
| 8 | Accept | Clear accept, strong contribution |
| 7 | Weak Accept | Above threshold, good with some weaknesses |
| 6 | Borderline Accept | Marginally above, notable weaknesses |
| 5 | Borderline Reject | Some merit but significant issues |
| 4 | Weak Reject | Limited novelty or notable flaws |
| 3 | Reject | Fundamental issues |
| 2 | Strong Reject | Major flaws throughout |
| 1 | Trivial/Wrong | Technically flawed |

### Comment Tags Reference

| Tag | Source | Severity Levels |
|-----|--------|----------------|
| `% [PROFESSOR]` | Professor review | CRITICAL, MAJOR, MINOR, SUGGESTION |
| `% [PEER-1]` | Architecture reviewer | CRITICAL, MAJOR, MINOR, QUESTION, MISSING_REF |
| `% [PEER-2]` | Mathematics reviewer | CRITICAL, MAJOR, MINOR, QUESTION, MISSING_REF |
| `% [PEER-3]` | Figures reviewer | CRITICAL, MAJOR, MINOR, QUESTION, MISSING_REF |
| `% [PEER-4]` | Related works reviewer | CRITICAL, MAJOR, MINOR, QUESTION, MISSING_REF |
| `% [OFFICIAL]` | Official reviewer | STRENGTH, WEAKNESS, QUESTION, RATING_FACTOR |
| `% [RESOLVED]` | Revision marker | Indicates addressed feedback |

### State & Resume

All review state is stored in `.cv-lab/` inside the paper project directory:

```
paper_project/
  .cv-lab/                        # Add to .gitignore
    state.json                    # Pipeline state (round, scores, venue, files)
    backup/round-0/               # Full project snapshot before any changes
      main.tex
      main.bib
      src/sec/*.tex
    round-1/
      after-professor/            # Snapshots mirror project structure
        main.tex
        src/sec/*.tex
      after-peers/
        ...
      revision-plan.md            # Prioritized revision plan
      after-revision/
        ...
      official-review.md          # Official review with rating
      professor-summary.md
      peer-{1-4}-summary.md
    round-2/
      ...
    final-report.md               # Aggregate report across all rounds
```

Revisions are applied **in-place** to the actual project files. Snapshots in `.cv-lab/round-N/` preserve each stage for rollback.

To resume an interrupted review, simply run the command again - it detects existing state and continues from the last completed phase.

### Configuration

Create `.cv-lab/config.json` in your paper directory for persistent defaults:

```json
{
  "threshold": 7,
  "max_rounds": 5,
  "venue": "CVPR",
  "skip_compilation_check": false,
  "keep_resolved_comments": true
}
```

## Ask Professor - Interactive Q&A

Talk directly to a senior CV professor who has read your paper. Ask about tone, arguments, experiments, submission strategy -- anything an experienced advisor would help with.

```bash
/claude-labs:ask-professor /path/to/paper/                         # Start Q&A session
/claude-labs:ask-professor /path/to/paper/ "Is my intro too bold?" # Direct question
/claude-labs:ask-professor . "What baselines am I missing?"        # Current directory
```

The professor:
- References specific sections, equations, tables, and figures
- Predicts what real reviewers would flag
- Gives concrete suggestions for every critique
- Knows CV venue standards (CVPR/ICCV/ECCV/NeurIPS)

## Ask Peers - Interactive Q&A with Specialized Reviewers

Talk to a panel of 4 specialized peer reviewers who **search for related papers** to answer your questions.

```bash
/claude-labs:ask-peers /path/to/paper/                                   # Start panel Q&A
/claude-labs:ask-peers /path/to/paper/ "Is my novelty claim valid?"      # Auto-routes to relevant reviewer(s)
/claude-labs:ask-peers /path/to/paper/ --reviewer 4 "What am I missing?" # Route to Peer-4 (Related Works)
/claude-labs:ask-peers . "How does my architecture compare?"             # Routes to Peer-1 (Architecture)
```

| Reviewer | Specialty | What They Search For |
|----------|-----------|---------------------|
| **Peer-1** | Architecture | ViT variants, CNN designs, hybrid models, efficiency papers |
| **Peer-2** | Mathematics | Loss functions, optimization, theoretical foundations |
| **Peer-3** | Figures & Viz | Presentation standards, benchmark papers in your subfield |
| **Peer-4** | Related Works | Missing citations, concurrent work, overlooked prior art |

Each reviewer actively searches for related papers via web search when answering, bringing external knowledge beyond what's in the paper.

## Project Structure

```
claude-labs/
  .claude-plugin/
    plugin.json             # Plugin metadata
    marketplace.json        # Marketplace listing
  agents/
    professor.md            # Professor reviewer agent definition
    peer-reviewer.md        # Peer reviewer agent definition (4 variants)
    official-reviewer.md    # Official reviewer agent definition
  skills/
    cv-lab/
      skill.md              # CV Lab review pipeline orchestration
    ask-professor/
      skill.md              # Interactive professor Q&A
    ask-peers/
      skill.md              # Interactive peer reviewer Q&A
  CLAUDE.md
  package.json
```

## License

MIT
