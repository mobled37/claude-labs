# Claude Labs

A Claude Code plugin for academic paper review orchestration. Currently features **CV Lab** (`cv_lab`), a multi-agent review pipeline for computer vision LaTeX papers that simulates the full conference review process.

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
/claude-labs:cv-lab paper.tex                               # Default: threshold=7, max-rounds=5
/claude-labs:cv-lab paper.tex --threshold 8                  # Require score 8+ (clear accept)
/claude-labs:cv-lab paper.tex --threshold 6 --max-rounds 3   # Lenient, fewer rounds
/claude-labs:cv-lab paper.tex --threshold 5 --max-rounds 2   # Quick check for early drafts
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `path` | (required) | Path to the main `.tex` file |
| `--threshold N` | `7` | Minimum official review score (1-10) to accept |
| `--max-rounds M` | `5` | Maximum review-revise cycles |

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

All review state is stored in `.cv-lab/` in the paper's directory:

```
.cv-lab/
  state.json              # Pipeline state (round, scores, status)
  backup/round-0/         # Original paper backup
  round-1/
    after-professor.tex   # Paper after professor comments
    after-peers.tex       # Paper after merged peer comments
    revision-plan.md      # Prioritized revision plan
    after-revision.tex    # Paper after revisions
    official-review.md    # Official review with rating
  round-2/
    ...
  final-report.md         # Aggregate report across all rounds
```

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
      skill.md              # CV Lab orchestration skill
  CLAUDE.md
  package.json
```

## License

MIT
