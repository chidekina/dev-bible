# Contributing to Dev Bible

## Adding a New Topic

1. Pick the right track (`en/tracks/00-cs-fundamentals/` … `06-cloud/`)
2. Create a new `.md` file using the template below
3. Add an entry to the track's `README.md`
4. Mirror the file in `pt-br/` (translate or stub with `> 🚧 Translation in progress`)
5. Open a PR

## File Template

```markdown
# Topic Name

## What & Why
<!-- Explain what this is and why a developer needs to know it -->

## Core Concepts
<!-- Key vocabulary and mental models -->

## How It Works
<!-- Internals, diagrams (ASCII or Mermaid), step-by-step breakdown -->

## Code Examples
<!-- Realistic, runnable examples with comments -->

## Common Mistakes & Pitfalls
<!-- What trips people up -->

## When to Use / Not Use
<!-- Decision guidance -->

## Real-World Scenario
<!-- A concrete story or case study -->

## Interview Questions
<!-- 5-10 questions with answers -->

## Exercises
<!-- 3-5 hands-on challenges with hints -->

## Further Reading
<!-- Links to official docs, papers, videos -->
```

## Quality Bar

- Every claim must be accurate — cite sources when unsure
- Code examples must be runnable
- No topic stubs — if you open a file, fill it completely
- English first, then PT-BR translation

## Style

- Active voice, short sentences
- Use code blocks for all code
- Use tables for comparisons
- Use `> 💡` for tips, `> ⚠️` for warnings
