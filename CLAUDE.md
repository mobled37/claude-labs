# Claude Labs

Claude Code plugin for academic paper review orchestration.

## Skills

- `cv-lab` (alias: `cv_lab`) - Computer vision paper review pipeline

## How to Use

```
/claude-labs:cv-lab paper.tex
/claude-labs:cv-lab paper.tex --threshold 8
/claude-labs:cv-lab paper.tex --threshold 7 --max-rounds 5
```

## Review Pipeline

1. **Professor Review** - Defensive review of tone, consistency, arguments, experiments (inline LaTeX comments)
2. **4 Peer Reviews** (parallel) - Architecture, Mathematics, Figures, Related Works specialists (each searches different related papers)
3. **Plan & Revise** - Aggregate feedback, prioritize, revise the paper
4. **Official Review** - Conference-style review with 1-10 rating
5. **Loop** - If score < threshold, repeat from step 1

## Agents

| Agent | Model | Role |
|-------|-------|------|
| professor | opus | Defensive advisor review (tone, arguments, experiments) |
| peer-reviewer | sonnet | 4 specialized reviewers (architecture, math, figures, related works) |
| official-reviewer | opus | Conference reviewer with 1-10 rating |

## Comment Tags

- `% [PROFESSOR]` - Professor comments
- `% [PEER-1]` through `% [PEER-4]` - Peer reviewer comments
- `% [OFFICIAL]` - Official reviewer comments
- `% [RESOLVED]` - Addressed comments after revision
