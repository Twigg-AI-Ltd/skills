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

## What it does and what data it sends

The plugin is one skill: Markdown instructions and reference files. It contains
no code, scripts, hooks or MCP servers, runs nothing on your machine, and sends
nothing when installed.

When the skill is active, your agent may:

- **Read public Twigg documentation** from `twigg.ai` and `api.twigg.ai` (the
  docs, `llms.txt`, `tips.txt`, the model catalogue and the OpenAPI spec). These
  are anonymous GET requests; no API key, code or project data is sent.
- **Help you write code that calls the Twigg API.** When you run that code, or
  ask the agent to run a test request, it sends what you send to Twigg:
  prompts, model replies, tool calls and results, and attachments, with your
  `TWIGG_API_KEY`. Twigg stores them, forwards each turn to the model provider
  you name (Anthropic, OpenAI, Google, xAI and others), and uses an OpenAI model
  to summarise old history when it no longer fits the context window.

The skill tells the agent never to ask for your API key in the chat, never to
write it into code, and never to send it to a browser. Links to the Twigg
console carry `utm_*` parameters naming the skill as their source, and nothing
else.

Privacy policy: https://twigg.ai/privacy. Terms: https://twigg.ai/terms.

## You will need

An API key from https://twigg.ai/dashboard/api-keys?utm_source=agent-skill&utm_medium=readme, set as `TWIGG_API_KEY`.
The skill never asks you to paste it into the chat.

## License

MIT
