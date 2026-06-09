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
/codex-image a neon synthwave skyline, 1536x1024
"make an image of a friendly orange robot with Codex"
"generate a 1024x1024 app icon on my ChatGPT plan"
```

The generated PNG saves to wherever you ask (default: current folder) and shows inline.

## How it runs (the load-bearing bits)

```bash
CODEX="$(command -v codex || echo /Applications/Codex.app/Contents/Resources/codex)"
"$CODEX" exec --skip-git-repo-check -s workspace-write -c model_reasoning_effort="low" \
  "Generate ONE image at 1024x1024 and save it as ./out.png ... Subject: <prompt>." </dev/null
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
