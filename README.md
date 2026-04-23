# Fastfony — Claude Code skill

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that helps a developer build a Symfony starter-kit using [Fastfony](https://fastfony.com) bundles.

## What this skill does

When a user mentions Fastfony, asks to bootstrap a Symfony starter-kit, or wants to add authentication / user management to a Symfony app, Claude loads this skill and follows its instructions to:

- Bootstrap a fresh Symfony project with sensible defaults.
- Install and configure Fastfony bundles (currently: [`fastfony/identity-bundle`](https://github.com/fastfony/identity-bundle)).
- Customize bundle behavior (template overrides, entity extension, config).
- Troubleshoot common integration issues.

## Layout

```
skill-fastfony/
├── SKILL.md                         ← main entry (loaded when the skill triggers)
└── references/
    ├── identity-bundle.md           ← full integration guide for identity-bundle
    ├── bootstrap-project.md         ← create a new Symfony starter-kit from scratch
    ├── conventions.md               ← code & bundle conventions
    └── troubleshooting.md           ← common issues and fixes
```

`SKILL.md` is intentionally short — it routes to the right reference file depending on the user's request.

## Try it

### On Claude.ai

1. Zip this directory (or upload the folder directly if the UI allows).
2. In Claude.ai, open the project settings → **Skills** → upload.
3. Start a conversation with a Symfony starter-kit request, e.g. *"Help me bootstrap a Symfony starter-kit with authentication using Fastfony."*

### Locally with Claude Code

Place this directory under `~/.claude/skills/` (or a project-scoped `.claude/skills/`). Claude Code will pick it up automatically.

```bash
ln -s "$PWD" ~/.claude/skills/fastfony
```

## Roadmap

As more Fastfony bundles are released, each gets:

1. An entry in the bundle table inside `SKILL.md`.
2. A dedicated `references/<bundle>.md` integration guide.

Planned:

- `fastfony/blog-bundle` (not published yet)
- `fastfony/billing-bundle` (not published yet)

See the [Fastfony GitHub organization](https://github.com/fastfony) for the current bundle list.

## Contributing

This skill evolves alongside the Fastfony ecosystem. PRs welcome:

- New bundle reference files following the template of [`references/identity-bundle.md`](references/identity-bundle.md).
- Improvements to conventions or troubleshooting based on real-world usage.
- Bug reports on skill behavior (what should have triggered but didn't, or vice versa).

## License

MIT — same as most Fastfony bundles, so the skill can ship alongside them without license friction.
