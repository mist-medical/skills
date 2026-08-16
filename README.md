# LLM Skill Files for the MIST Ecosystem

This repository contains **Agent Skills** for Claude Code. Each gives Claude deep,
structured knowledge about one tool in the MIST ecosystem so it can answer questions, debug
issues and guide workflows without re-reading the source or the docs every time.

## Available Skills

| Skill | Tool | Description |
|-------|------|-------------|
| [`mist_expert`](mist_expert/SKILL.md) | [MIST](https://github.com/mist-medical/MIST) | Expert assistant for the MIST 3D medical image segmentation framework |
| [`misfit_expert`](misfit_expert/SKILL.md) | [MISFIT](https://github.com/mist-medical/MISFIT) | Expert assistant for the MISFIT medical imaging foundation-model pretraining toolkit |
| [`mist_autoresearch_expert`](mist_autoresearch_expert/SKILL.md) | [mist-autoresearch](https://github.com/mist-medical/mist-autoresearch) | Expert assistant for the LLM-driven autoresearch loops built on MIST |

## Usage

Copy or symlink a skill directory into your skills folder — `~/.claude/skills/` for every
project on the machine, or `<project>/.claude/skills/` to share it with that repo's
collaborators:

```bash
git clone https://github.com/mist-medical/skills.git
cp -r skills/mist_expert ~/.claude/skills/
```

Skills are discovered when a session starts, so restart Claude Code after adding one.
`/skills` lists what is loaded.

From there a skill applies in **two** ways:

- **Automatically** — Claude reads the `description` and loads the skill when your request
  matches it. Paste a MIST traceback or ask about `config.json` and the expert context is
  there without you naming it.
- **Explicitly** — invoke it by name with `/mist_expert`, as before.

The automatic path is the reason these are skills rather than slash commands: it does not
require the user to already know the skill exists.

### Migrating from the slash-command layout

These files were previously flat `.md` files intended for `.claude/commands/`. If you
installed one that way, replace it:

```bash
rm ~/.claude/commands/mist_expert.md
cp -r mist_expert ~/.claude/skills/
```

`/mist_expert` keeps working, and you additionally get automatic loading.

### Other assistants

The skill body is plain markdown and portable. For Cline, copy `SKILL.md` into the
project's `.clinerules/` (renaming it after the skill). Note that Cline rules are always in
context rather than loaded on demand, so install only the ones a given project needs.

## About MIST

[MIST](https://github.com/mist-medical/MIST) is a simple, scalable, end-to-end framework
for 3D medical image segmentation. It handles everything from raw NIfTI files to trained
models and evaluated predictions. Full documentation is at
[mist-medical.readthedocs.io](https://mist-medical.readthedocs.io/).
