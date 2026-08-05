# agenix — agent plugins

Marketplace-installable plugins that teach AI coding agents to use
[agenix](https://agenix.fedtuna.ai/docs/): an engine that separates what an agent
knows (files you can read and diff) from what runs it (any executor CLI) and
from how it reaches a model.

This repo is **both** a Codex marketplace (`.agents/plugins/marketplace.json`)
and a Claude Code marketplace (`.claude-plugin/marketplace.json`); the two plugin
manifests share one skill (`plugins/agenix/skills/agenix/SKILL.md`).

## Codex

```bash
codex plugin marketplace add fedtuna-ai/fedtuna-agent-skills
codex plugin add agenix@fedtuna-agent-skills
```

## Claude Code

```
/plugin marketplace add fedtuna-ai/fedtuna-agent-skills
/plugin install agenix@fedtuna-agent-skills
```

## Layout

```
.agents/plugins/marketplace.json      # Codex marketplace
.claude-plugin/marketplace.json       # Claude marketplace
plugins/agenix/
  .codex-plugin/plugin.json           # Codex plugin manifest (skills: ./skills/)
  .claude-plugin/plugin.json          # Claude plugin manifest
  skills/agenix/SKILL.md              # the skill itself — how to use the CLI
```

## What is in the skill

The profile (one file: executor, model, credential, knowledge), the knowledge
directory (persona, rules, facts, skills, episodes, goals, competence, world),
the commands (`run`, `chat`, `compile`, `auth`, `models`, `eval`, `audit`,
`compare`, `teach`, `grow`, `serve`), and — the part that saves the most time —
the mistakes: model notation, `builtin`, subscription tokens that are stored but
not named, `-x` swapping the executor and nothing else, and the shell calls a
headless executor refuses until it is told otherwise.
