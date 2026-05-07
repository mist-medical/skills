# LLM Skill Files for MIST (and other upcoming tools)

This repository contains skill files for use with Claude Code. Each skill file gives Claude deep, structured knowledge about a specific tool or framework so it can answer questions, debug issues, and guide workflows without needing to re-read source code or documentation each time.

## Available Skills

| File | Tool | Description |
|------|------|-------------|
| [`mist_expert.md`](mist_expert.md) | [MIST](https://github.com/mist-medical/MIST) | Expert assistant for the MIST 3D medical image segmentation framework |

## Usage

To use a skill in Claude Code, add the skill file as a slash command by placing it (or symlinking it) in your `.claude/commands/` directory:

```bash
cp mist_expert.md /path/to/your/project/.claude/commands/mist_expert.md
```

Then invoke it in Claude Code with:

```
/mist_expert
```

Claude will load the skill and respond as a MIST expert for the rest of the conversation.

## About MIST

[MIST](https://github.com/mist-medical/MIST) is a simple, scalable, end-to-end framework for 3D medical image segmentation. It handles everything from raw NIfTI files to trained models and evaluated predictions. Full documentation is available at [mist-medical.readthedocs.io](https://mist-medical.readthedocs.io/).
