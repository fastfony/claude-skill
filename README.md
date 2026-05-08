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

## Support

Found a bug or want a new bundle covered? Open an issue on this repository, or email [support@fastfony.com](mailto:support@fastfony.com).

## Like it?

If the skill saves you time, [buy the maintainer a coffee ☕](https://buymeacoffee.com/neothone) — entirely optional, always appreciated.

## License

[MIT](LICENSE) — like every Fastfony bundle and pack. Use it, fork it, ship it, sell what you build with it.
