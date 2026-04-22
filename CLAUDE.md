# CLAUDE.md — diva

## Project overview

Diva makes Claude Code respond in compressed diva-style prose — cuts ~65-75% output tokens, full technical accuracy. Works as a Claude Code plugin (hooks + skills).

---

## File structure and what owns what

### Single source of truth files — edit only these

| File | What it controls |
|------|-----------------|
| `skills/diva/SKILL.md` | Diva behavior: intensity levels, rules, wenyan mode, auto-clarity, persistence. Only file to edit for behavior changes. |
| `skills/diva-help/SKILL.md` | Quick-reference card. One-shot display, not a persistent mode. |
| `diva-compress/SKILL.md` | Compress sub-skill behavior. |

### Auto-synced — do not edit directly

| File | Synced from |
|------|-------------|
| `diva/SKILL.md` | `skills/diva/SKILL.md` |
| `plugins/diva/skills/diva/SKILL.md` | `skills/diva/SKILL.md` |
| `plugins/diva/skills/compress/` | `diva-compress/` |

---

## Hook system (Claude Code)

Three hooks in `hooks/` plus a `diva-config.js` shared module and a `package.json` CommonJS marker. Communicate via flag file at `$CLAUDE_CONFIG_DIR/.diva-active` (falls back to `~/.claude/.diva-active`).

```
SessionStart hook ──writes "full"──▶ $CLAUDE_CONFIG_DIR/.diva-active ◀──writes mode── UserPromptSubmit hook
                                                       │
                                                    reads
                                                       ▼
                                              diva-statusline.sh
                                            [DIVA] / [DIVA:ULTRA] / ...
```

`hooks/package.json` pins the directory to `{"type": "commonjs"}` so the `.js` hooks resolve as CJS even when an ancestor `package.json` (e.g. `~/.claude/package.json` from another plugin) declares `"type": "module"`. Without this, `require()` blows up with `ReferenceError: require is not defined in ES module scope`.

All hooks honor `CLAUDE_CONFIG_DIR` for non-default Claude Code config locations.

### `hooks/diva-config.js` — shared module

Exports:
- `getDefaultMode()` — resolves default mode from `DIVA_DEFAULT_MODE` env var, then `$XDG_CONFIG_HOME/diva/config.json` / `~/.config/diva/config.json` / `%APPDATA%\diva\config.json`, then `'full'`
- `safeWriteFlag(flagPath, content)` — symlink-safe flag write. Refuses if flag target or its immediate parent is a symlink. Opens with `O_NOFOLLOW` where supported. Atomic temp + rename. Creates with `0600`. Protects against local attackers replacing the predictable flag path with a symlink to clobber files writable by the user. Used by both write hooks. Silent-fails on all filesystem errors.

### `hooks/diva-activate.js` — SessionStart hook

Runs once per Claude Code session start. Three things:
1. Writes the active mode to `$CLAUDE_CONFIG_DIR/.diva-active` via `safeWriteFlag` (creates if missing)
2. Emits diva ruleset as hidden stdout — Claude Code injects SessionStart hook stdout as system context, invisible to user
3. Checks `settings.json` for statusline config; if missing, appends nudge to offer setup on first interaction

Silent-fails on all filesystem errors — never blocks session start.

### `hooks/diva-mode-tracker.js` — UserPromptSubmit hook

Reads JSON from stdin. Three responsibilities:

**1. Slash-command activation.** If prompt starts with `/diva`, writes mode to flag file via `safeWriteFlag`:
- `/diva` → configured default (see `diva-config.js`, defaults to `full`)
- `/diva lite` → `lite`
- `/diva ultra` → `ultra`
- `/diva wenyan` or `/diva wenyan-full` → `wenyan`
- `/diva wenyan-lite` → `wenyan-lite`
- `/diva wenyan-ultra` → `wenyan-ultra`
- `/diva-compress` → `compress`

**2. Natural-language activation/deactivation.** Matches phrases like "activate diva", "turn on diva mode", "talk like diva" and writes the configured default mode. Matches "stop diva", "disable diva", "normal mode", "deactivate diva" etc. and deletes the flag file.

**3. Per-turn reinforcement.** When flag is set, emits a small `hookSpecificOutput` JSON reminder so the model keeps diva style across turns. The full ruleset still comes from SessionStart — this is just an attention anchor.

### `hooks/diva-statusline.sh` — Statusline badge

Reads flag file at `$CLAUDE_CONFIG_DIR/.diva-active`. Outputs colored badge string for Claude Code statusline:
- `full` or empty → `[DIVA]` (orange)
- anything else → `[DIVA:<MODE_UPPERCASED>]` (orange)

Configured in `settings.json` under `statusLine.command`. PowerShell counterpart at `hooks/diva-statusline.ps1` for Windows.

### Hook installation

**Plugin install** — hooks wired automatically by plugin system.

**Standalone install** — `hooks/install.sh` (macOS/Linux) or `hooks/install.ps1` (Windows) copies hook files into `~/.claude/hooks/` and patches `~/.claude/settings.json` to register SessionStart and UserPromptSubmit hooks plus statusline.

**Uninstall** — `hooks/uninstall.sh` / `hooks/uninstall.ps1` removes hook files and patches settings.json.

---

## Skill system

Skills = Markdown files with YAML frontmatter consumed by Claude Code's plugin system.

### Intensity levels

Defined in `skills/diva/SKILL.md`. Six levels: `lite`, `full` (default), `ultra`, `wenyan-lite`, `wenyan-full`, `wenyan-ultra`. Persists until changed or session ends.

### Auto-clarity rule

Diva drops to normal prose for: security warnings, irreversible action confirmations, multi-step sequences where fragment ambiguity risks misread, user confused or repeating question. Resumes after. Defined in skill — preserve in any SKILL.md edit.

### diva-compress

Sub-skill in `diva-compress/SKILL.md`. Takes file path, compresses prose to diva style, writes to original path, saves backup at `<filename>.original.md`. Validates headings, code blocks, URLs, file paths, commands preserved. Retries up to 2 times on failure with targeted patches only. Requires Python 3.10+.

---

## Evals

`evals/` has three-arm harness:
- `__baseline__` — no system prompt
- `__terse__` — `Answer concisely.`
- `<skill>` — `Answer concisely.\n\n{SKILL.md}`

Honest delta = **skill vs terse**, not skill vs baseline. Baseline comparison conflates skill with generic terseness — that cheating. Harness designed to prevent this.

`llm_run.py` calls `claude -p --system-prompt ...` per (prompt, arm), saves to `evals/snapshots/results.json`. `measure.py` reads snapshot offline with tiktoken (OpenAI BPE — approximates Claude tokenizer, ratios meaningful, absolute numbers approximate).

---

## Benchmarks

`benchmarks/` runs real prompts through Claude API, records raw token counts. Results committed as JSON in `benchmarks/results/`.

To reproduce: `uv run python benchmarks/run.py` (needs `ANTHROPIC_API_KEY` in `.env.local`).

---

## Key rules for agents working here

- Edit `skills/diva/SKILL.md` for behavior changes. Never edit synced copies.
- Hook files must silent-fail on all filesystem errors. Never let hook crash block session start.
- Any new flag file write must go through `safeWriteFlag()` in `diva-config.js`. Direct `fs.writeFileSync` on predictable user-owned paths reopens the symlink-clobber attack surface.
- Hooks must respect `CLAUDE_CONFIG_DIR` env var, not hardcode `~/.claude`. Same for `install.sh` / `install.ps1` / statusline scripts.
