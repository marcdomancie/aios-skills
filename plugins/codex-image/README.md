# codex-image

![codex-image — generate images on your ChatGPT/Codex plan](preview.png)

A Claude Code skill that generates images through OpenAI's image model using your **ChatGPT/Codex subscription** — no OpenAI API key, no per-image dollar cost (it draws on your ChatGPT/Codex plan's usage limits).

## What it does

1. Resolves the Codex CLI (prefers PATH, falls back to the bundled binary inside `Codex.app`).
2. Shells out to `codex exec` with the load-bearing flags, so Codex's built-in image tool renders a PNG.
3. Copies the PNG to the path you asked for and shows it inline.

No `OPENAI_API_KEY` needed — auth is Codex's own (`~/.codex/auth.json` in `auth_mode: chatgpt`).

## Requirements

- **Claude Code** (this is a Claude Code skill).
- **Codex** — the desktop app (`/Applications/Codex.app`, bundles the CLI) or the standalone `codex` CLI on PATH.
- Codex signed in with a **ChatGPT plan**: `codex login` (Plus / Pro / Team — *not* an API key).

## Install (from the zip)

1. Unzip, then move the whole `codex-image` folder into your Claude Code skills directory so the
   layout is:
   ```
   ~/.claude/skills/codex-image/SKILL.md
   ~/.claude/skills/codex-image/README.md
   ```
   On macOS/Linux:
   ```bash
   mkdir -p ~/.claude/skills
   mv ~/Downloads/codex-image ~/.claude/skills/codex-image
   ```
2. Restart Claude Code (or open a new session) so the skill registers.
3. Make sure Codex is installed and logged in: `codex login`.

## Usage in Claude Code

Just ask naturally (the description triggers it), or call it directly:

```
/codex-image a neon synthwave skyline, wide landscape banner
"make an image of a friendly orange robot with Codex"
"generate a square app icon on my ChatGPT plan, 1024x1024"
"recreate this in the style of ~/assets/reference.jpg"
"make 5 logo options for 'Field & Flour' on my ChatGPT plan"
"revise ./hero.png — make the lighting warmer, keep everything else the same"
```

The generated PNG saves to wherever you ask (default: current folder) and shows inline.

### Sizes — how they actually work

Codex's built-in image tool is **prompt-only**: it picks its own dimensions (~1–2.2 MP) and
silently ignores explicit pixel requests. What works:

- **Aspect ratio** steers reliably via wording — *square*, *wide landscape banner* (~2.4:1),
  *portrait*. Max ~3:1.
- **Exact dimensions** (1080x1080, 1280x720, …) — the skill has Codex resize the render locally
  with PIL LANCZOS after generation. Upscales add pixels, not detail; native detail tops out
  around 2 MP. (The API's native-4K `size`/`quality` params need an `OPENAI_API_KEY` — not this
  ChatGPT-auth path.)

### Quality & multiple outputs — what the plan path can do

gpt-image-2's `quality` (low/medium/high), `n` (multiple outputs), and `size` are **API parameters,
not built-in-tool controls** — so on the ChatGPT-plan path they aren't settable. They live only in
Codex's fallback CLI, which calls the real Image API and **requires an `OPENAI_API_KEY`** (= real
per-image billing). On the plan:

- **Higher quality** = sharper prompting + regeneration. The built-in tool is already high-fidelity
  by default; say *"photorealistic"*, demand *"crisp legible text"*, and state the intended use.
- **Multiple outputs** (e.g. 5 logo options) = the skill **loops one generation per variant** with
  diversified prompts, then checks they're distinct. (`n` proper is the API-key path only.)
- True `quality=high`, native 4K, real `n`, masks, or model-native transparency → only via the
  CLI fallback + your own API key, with your explicit OK.

### Reference images, edits & revisions

Point it at any image file on disk and it can use it as a style reference, edit target, or
compositing input — style transfer, object replacement, text replacement, sketch-to-render.

**To revise an image you already made, the previous PNG gets re-sent** (each Codex run is a fresh
session with no memory): the skill re-attaches it and tells Codex *"change only X, keep everything
else the same."* For mechanical tweaks (recolor, crop, overlay text) Codex may edit it with a quick
script; for creative changes (*"warmer lighting"*, *"turn this sketch photoreal"*) it re-renders with
the image model.

Transparent backgrounds work via Codex's built-in chroma-key + local removal pipeline (just ask
for "a transparent PNG").

## How it runs (the load-bearing bits)

```bash
CODEX="$(command -v codex || echo /Applications/Codex.app/Contents/Resources/codex)"
"$CODEX" exec --skip-git-repo-check -s workspace-write -c model_reasoning_effort="low" \
  "Generate ONE image in square composition. Subject: <prompt>. Then resize to EXACTLY 1080x1080 with Python PIL LANCZOS and save as ./out.png ..." </dev/null
```

- `--skip-git-repo-check` — Codex otherwise refuses to run outside a trusted/git dir.
- `-s workspace-write` — lets it save the file.
- `</dev/null` — stops `codex exec` hanging on stdin.
- `-c model_reasoning_effort="low"` — fast/cheap; a high default is overkill for image gen.

The raw image always lands in `~/.codex/generated_images/<session>/ig_*.png` first; the skill copies it to your requested path.

## Notes

- Usage counts against **your** ChatGPT/Codex plan limits, not API billing. Everyone who uses this uses their own Codex login — nothing is shared between machines.
- macOS-oriented (the app-bundle fallback path), but works anywhere `codex` is on PATH.
- Same engine as the `codex-image-bridge` plugin (RolandOne/codex-image-generation), but PATH-independent and self-owned.

## License

Free to use — share it onward if it's useful.
