# Warp Factories Skills

Public, installable [Agent Skills](https://agentskills.io) for **Warp Factories**. Skills in this repo teach agents — Warp's own or third-party coding agents like Claude Code, Codex, and Cursor — how to set up and work with a Warp Factory.

## Installing a skill

Skills live under `.agents/skills/<skill-name>/SKILL.md`. Install one with the [Skills CLI](https://skills.sh):

```bash
npx skills add warpdotdev/warp-factories-skills --skill factory-setup
```

Swap `factory-setup` for any other skill name in this repo. Run `npx skills add --help` for flags that target a specific agent or install scope. You can also copy a skill folder directly into your project's or home directory's `.agents/skills/`.

## Contributing

Add a new skill as its own folder under `.agents/skills/`, following the [Agent Skills](https://agentskills.io) format (a `SKILL.md` with YAML frontmatter and markdown instructions).
