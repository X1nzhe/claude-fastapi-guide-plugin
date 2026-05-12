# claude-fastapi-guide-plugin

A Claude Code plugin providing read-only, citation-grounded exploration
of the [FastAPI](https://github.com/fastapi/fastapi) framework source.

## What's included

- **3 skills**
  - `explain-feature` — how a feature works, with `path:line` citations
  - `trace-data-flow` — end-to-end timeline of one concrete request
  - `fastapi-dependencies-internals` — DI-subsystem-specific guidance
- **1 subagent**
  - `code-explorer` — structured deep-dive into a single module
- **2 hooks**
  - `SessionStart` — prints a one-line banner
  - `PostToolUse(Read)` — logs reads to `./notes/exploration-log.txt`
    in your CWD (see *Side effects* below)

## Install

This plugin assumes you launch `claude` from **inside a FastAPI
checkout**. Skill instructions reference paths like
`fastapi/dependencies/utils.py` relative to CWD.

```bash
git clone https://github.com/fastapi/fastapi
cd fastapi
claude
> /plugin marketplace add xinzheyuan/claude-fastapi-guide-plugin
> /plugin install claude-fastapi-guide-plugin@xinzheyuan-marketplace
```

## Use

Skills auto-trigger on relevant questions:

```
> How does dependency injection work?
> Trace a POST /items request end-to-end
> What's the two-phase pattern in fastapi/dependencies/?
```

Or invoke them explicitly via slash commands (namespace = plugin name).

## Side effects

The `PostToolUse(Read)` hook creates `notes/exploration-log.txt` in
your CWD. To disable: edit `hooks/hooks.json` in the installed
plugin and remove the `PostToolUse` entry.

This plugin does **not** ship the safety-lockdown permissions
(`defaultMode: plan`, Edit/Write deny) from its lab counterpart —
those are personal preferences, not plugin behavior. If you want a
strict read-only session, add them to your own
`~/.claude/settings.json`.

## Related

[claude-tour-fastapi](https://github.com/xinzheyuan/claude-tour-fastapi)
is the project-scoped (`.claude/` directly) form of this same
toolkit, used for the original lab.

## License

MIT