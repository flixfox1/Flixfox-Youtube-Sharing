# AI Agent Skills & Workflow Sharing

Resources and skill files shared from my YouTube channel — covering AI agent orchestration, prompt engineering, and practical development workflows.

This is a living repo. New content gets added as I publish new videos.

## What's Inside

### Orchestrator Agent Skills

Reusable skill definitions and task orchestration patterns for AI coding agents (Kiro, Cursor, Windsurf, etc.).

- `task-orchestrator/` — A general-purpose task orchestrator skill that reads `TASK-*.md` files, resolves dependencies, dispatches sub-agents, and tracks progress. Drop it into your agent's skill folder and let it manage multi-step plans for you.

### Quick Reference — Problem Files

Real-world problem-solving examples used in videos. These are reference-only — they show how I break down complex problems into structured task plans.

- `坐标问题/` — Coordinate system unification case study: planning docs, multi-model comparisons (Codex, Gemini), and a full set of `TASK-*.md` files demonstrating the orchestrator in action.

## How to Use

1. The Orchestrator Agent Skills include a general-purpose task subdivision skill and a task orchestrator skill. Together they automatically break down complicated problems from a detailed Blueprint file into structured, executable task plans.
2. The problem files under `Quick Reference only` are there purely for quick browsing — don't spend time analyzing them deeply. They're just real examples to give you a feel for what the output looks like.
3. Watch the corresponding YouTube videos for full context and walkthroughs.
## Repo Structure

```
.
├── Orchestrator Agent Skills/
│   ├── task-orchestrator/          # Reusable orchestrator skill
│   │   └── SKILL.md
│   └── Quick Reference only - Problem files/
│       └── 坐标问题/              # Coordinate system case study
│           ├── tasks-coord/        # Task files (TASK-0 through TASK-9)
│           ├── PROGRESS.md
│           └── ... planning docs
└── README.md
```

## Contributing

Found something useful? Have suggestions? Feel free to open an issue or PR.

## License

MIT
