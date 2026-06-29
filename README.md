# Claude Code skills

Personal [Claude Code](https://claude.com/claude-code) skills, version-controlled.

This repo lives in place at `~/.claude/skills/`, where Claude Code reads skills from. Each skill is a directory containing a `SKILL.md` (with `name` + `description` frontmatter) and any supporting reference files.

> Note: some skills in this directory are intentionally **not** tracked (see `.gitignore`) because they contain internal/company material or personal config.

## Skills

- **free-disk-space** — Diagnose a full or nearly-full disk on macOS and safely reclaim space, using a prioritized, safety-ranked cleanup playbook (git garbage, Docker images, Xcode/iOS caches, node_modules, stale git worktrees, app caches).

## Setup on a new machine

Clone into your Claude config directory:

```bash
git clone <repo-url> ~/.claude/skills
```

Claude Code picks up the skills automatically.
