# Twigg agent skills

A skill that teaches coding agents (Claude Code, Codex and others that read
`SKILL.md`) how to build on [Twigg](https://twigg.ai/?utm_source=agent-skill&utm_medium=readme): when it fits, how to get
started, and the practices that keep an integration small.

Twigg is a hosted context store and context engine. It keeps every message,
tool call and result, and builds each model call from them, so an agent,
assistant or chatbot sends only the new input of each turn and never the
history.

## Install

**Claude Code**

```
/plugin marketplace add Twigg-AI-Ltd/skills
/plugin install twigg@twigg
```

**Any agent, with the `skills` CLI**

```bash
npx skills add Twigg-AI-Ltd/skills
```

**By hand**

Copy `skills/twigg` into your agent's skills directory, for example
`~/.claude/skills/twigg` for Claude Code, or `~/.agents/skills/twigg` for Codex
and other agents that use the shared `.agents/skills` location.
In a project, `.claude/skills/` or `.agents/skills/` shares it with everyone on
that repo.

**No install**

Tell your agent: "Read https://twigg.ai/llms.txt and https://twigg.ai/tips.txt
before building on Twigg."

## What is in it

```
skills/twigg/
  SKILL.md                  when to use Twigg, getting started, the loop, core rules
  references/
    tips.md                 structure, configuration, turns, cleanup
    patterns.md             agents, web chat, messaging bots, multi-tenant, batch, migration
    typescript.md           @twigg/sdk recipes
    python.md               twigg (PyPI) recipes
    http.md                 plain HTTP and the stream
    configuration.md        instructions, tools, budgets, retention, @twigg/cli
    errors.md               error codes, retries, recovery, debugging
    n8n.md                  guiding a non-technical user through n8n
```

The agent loads `SKILL.md` when a task matches and reads a reference only when
it needs one. The skill points at the live docs for anything that may have
changed since it was written.

## You will need

An API key from https://twigg.ai/dashboard/api-keys?utm_source=agent-skill&utm_medium=readme, set as `TWIGG_API_KEY`.
The skill never asks you to paste it into the chat.

## License

MIT
