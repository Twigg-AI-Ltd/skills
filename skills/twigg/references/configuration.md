# Configuration

Four surfaces are published per scope (the organisation, or a namespace) and
versioned: **instructions** (system prompt), **tools** (saved tool definitions),
**budgets** (how much history a turn carries) and **retention** (how old parts
are replayed). None travels on a request: each turn resolves them for its
chat's namespace by walking up the path. Live docs:
https://twigg.ai/docs/context.md

Manage them through the API, the console (https://twigg.ai/dashboard?utm_source=agent-skill&utm_medium=skill&utm_content=configuration) or as
code with `@twigg/cli`. Prefer the CLI in a codebase: the config sits in the
repo and is reviewed like code.

## Inheritance

- The organisation scope (`global` in the file, `namespace: null` in the API) is
  the default for everything.
- A child (`acme/support`) resolves on top of its parent (`acme`).
- Instructions and tools combine with `mode`: `append` (default) adds to the
  parent, `replace` discards it.
- Budgets and retention do not merge: the nearest one wins.
- Check the result: `npx twigg config resolve --scope acme/support` or
  `GET /api/v1/config/resolved?namespace=acme/support`.

## `@twigg/cli`

```bash
npm install --save-dev @twigg/cli
npx twigg config pull      # write what is configured today
npx twigg config plan      # what would change
npx twigg config apply     # publish it (asks first; --yes in CI)
```

`twigg.config.yaml`:

```yaml
# yaml-language-server: $schema=https://twigg.ai/schemas/config.v1.json
version: 1
global:
  instructions: { $file: prompts/global.md }
namespaces:
  acme:
    instructions:
      mode: append
      body: { $file: prompts/acme.md }
    tools:
      definitions:
        - $file: tools/get_order.json
    budgets:
      default: { version: 3, headroom: 0.2, history_mode: summarize }
  acme/support:
    instructions:
      mode: replace
      body: { $file: prompts/support.md }
    tools: null          # withdraw the override; inherit from acme
```

Rules:

- A scope or surface left out of the file is unmanaged and never touched.
  `null` withdraws that scope's override.
- `$file` paths are relative to the config file.
- `${TWIGG_VAR_NAME}` is substituted; only `TWIGG_VAR_*` variables are read.
- `twigg.config.ts` with `defineConfig` can reuse the SDK's `tool()` objects.
- Other commands: `rollback --surface tools`, `history`, `validate`,
  `convert --format ts`. `--scope acme` narrows plan, apply and pull.
- `plan --detailed-exitcode` exits 2 when there are changes, for CI drift checks.
- In CI: plan on pull requests, `apply --yes` on merge, one run at a time.

Run `apply` only when the user asks; it changes live behaviour for every chat
in those scopes.

## Budgets

A budget caps the estimated input of a turn:
`(context_window - reserved_output) x (1 - headroom)`, optionally lowered by
`max_history_tokens` (at least 32,000). Keyed by `default` or a catalogue model
name.

| option | meaning |
|---|---|
| `version` | Required: `3`. |
| `headroom` | Required fraction left unused; the default policy uses 0.10. |
| `max_history_tokens` | Optional ceiling, at least 32,000. |
| `history_mode` | `summarize` (default): a summary prefix plus recent originals. `drop_oldest`: omit oldest history, no summary calls. |
| `eager_compaction` | `{"mode":"at_fill","at":0.8}` (default) or `{"mode":"off"}` (required with `drop_oldest`). |

Leave budgets at their defaults unless the user has a reason (cost ceiling,
no summarisation). Lowering `max_history_tokens` is the usual cost lever.

## Retention

Retention decides how a part looks when replayed at a given depth. It never
changes what is stored; `/history` still returns everything.

- Thresholds are a share of the window measured back from the newest part:
  `1` replays however old, `0` never replays, `0.25` within the newest quarter.
- Rules cover attachments (all, per MIME family, or exact type), reasoning,
  tool payloads (all, a name prefix like `gh_*`, or one tool; results and
  arguments separately), failed calls, and prose.
- Truncation is `keep`, `always` or `over` a threshold, keeping `head`, `tail`
  or `head_and_tail`. Keep `mark_truncations` on.
- `anchor_parts` keeps the first N parts verbatim forever.
- Rules match first-hit in list order; a broad rule above a narrow one is
  refused.
- Media defaults to `0`: sent once, never replayed.

Typical use: a tool that returns large documents gets
`over` truncation past `0.25` so old results stop costing tokens every turn.

## Compaction

When history no longer fits, Twigg summarises an older prefix and keeps recent
history verbatim. Summaries are lossy, reusable, and billed as their own
`compaction` line. Compaction currently runs on an OpenAI model chosen by the
platform, whichever model the turn uses; mention this if the user has data
residency or provider constraints.
