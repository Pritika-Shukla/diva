# diva

Cut Claude Code output tokens by ~75%. Full technical accuracy kept. Zero setup beyond install.

**Before:**
> "Sure! I'd be happy to help you with that. The issue you're experiencing is likely caused by the fact that your component is re-rendering because you're creating a new object reference on every render cycle."

**After (diva full):**
> "New object ref each render. Inline prop = re-render. Wrap in `useMemo`."

---

## Install

### Option 1 — Claude Code plugin (recommended)

In any Claude Code session:

```
/plugin marketplace add Pritika-Shukla/diva
/plugin install diva@diva
```

Restart Claude Code. Diva activates automatically every session.

### Option 2 — Standalone hooks (no plugin system)

```bash
git clone https://github.com/Pritika-Shukla/diva.git
cd diva
bash hooks/install.sh
```

Restart Claude Code.

**Uninstall:**
```bash
bash hooks/uninstall.sh
```

---

## Usage

Diva activates automatically on session start. Statusline shows `[DIVA]` when active.

| Command | Effect |
|---------|--------|
| `/diva` | Full mode (default) |
| `/diva lite` | Drop filler/hedging, keep full sentences |
| `/diva ultra` | Max compression — abbreviations, arrows, one word when enough |
| `/diva wenyan-full` | Classical Chinese compression (文言文) |
| `stop diva` / `normal mode` | Deactivate |

You can also say **"activate diva"**, **"turn on diva mode"** in plain English.

---

## Intensity levels

| Level | Style |
|-------|-------|
| `lite` | No filler/hedging. Articles kept. Tight but full sentences. |
| `full` | Drop articles, fragments OK, short synonyms. Default. |
| `ultra` | Abbreviate everything (DB/auth/fn/req), arrows for causality, one word where one word enough. |
| `wenyan-lite` | Drop filler, keep grammar, classical register. |
| `wenyan-full` | Full 文言文 — 80-90% character reduction. |
| `wenyan-ultra` | Extreme classical abbreviation. |

---

## What stays unchanged

- All code blocks
- Error messages (quoted exact)
- Security warnings (written in plain prose — diva drops style for safety-critical output)
- Irreversible action confirmations

---

## Requirements

- Claude Code
- Node.js (for hooks)
- Python 3.10+ (for `/diva-compress` file compression only)
