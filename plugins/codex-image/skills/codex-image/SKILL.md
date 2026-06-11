---
name: codex-image
description: Generate images through OpenAI's image model using your ChatGPT/Codex subscription — no OpenAI API key, no per-image dollar cost (it draws on your ChatGPT/Codex plan limits). Use whenever the user wants to create, generate, or make an image / picture / illustration / icon / logo / asset and mentions "Codex", "ChatGPT image", "on my ChatGPT plan", "DALL·E / gpt-image", or just asks to generate an image and would rather use a ChatGPT/Codex subscription than another generator. Shells out to the bundled Codex CLI's built-in image tool and saves a PNG, then shows it inline.
---

# Codex Image Generation

Generate images via OpenAI's image model, billed against your **ChatGPT/Codex subscription**
(`~/.codex/auth.json` in `auth_mode: chatgpt`, `OPENAI_API_KEY: null` — so no API key and no
per-image dollar cost; it consumes your ChatGPT/Codex usage limits). The bundled Codex CLI's image
tool renders the PNG; save it where the user wants and Read it back so it shows inline.

## Prerequisites
- **Codex** installed:
  - macOS: the desktop app (`/Applications/Codex.app`, bundles the CLI at `Contents/Resources/codex`)
    or the standalone `codex` CLI on PATH.
  - Windows: the official installer `powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"`
    (puts `codex.exe` in `%LOCALAPPDATA%\Programs\OpenAI\Codex\bin` and adds it to the user PATH),
    or `npm install -g @openai/codex`, or WSL2.
  - Linux: `curl -fsSL https://chatgpt.com/codex/install.sh | sh` or npm.
- Codex signed in with a **ChatGPT plan**: run `codex login` (NOT an API key). It works when
  `~/.codex/auth.json` shows `auth_mode: chatgpt` (`%USERPROFILE%\.codex\auth.json` on Windows —
  same `~/.codex` path inside Git Bash).

## How to run

1. Resolve the Codex binary — PATH first, then known per-OS install locations:
   ```bash
   CODEX="$(command -v codex || command -v codex.exe || true)"
   if [ -z "$CODEX" ]; then for c in \
     "/Applications/Codex.app/Contents/Resources/codex" \
     "$LOCALAPPDATA/Programs/OpenAI/Codex/bin/codex.exe" \
     "$USERPROFILE/.codex/packages/standalone/current/bin/codex.exe" \
     "$APPDATA/npm/codex.cmd"; do
     [ -x "$c" ] && CODEX="$c" && break
   done; fi
   ```
   Still empty? Diagnose instead of giving up — the user may have installed Codex another way:
   try `powershell -c "Get-Command codex"` and `where codex` (Windows), `npm prefix -g` (then look
   for `codex.cmd` there), or `wsl -e which codex` (if it's in WSL2, run the whole command below
   prefixed with `wsl -e`). As a last resort, ask the user how/where they installed Codex and use
   that path.

2. Run it non-interactively. Choose an output path (default: `./<short-slug>.png` in the current
   working directory, or wherever the user asked):
   ```bash
   "$CODEX" exec --skip-git-repo-check -s workspace-write -c model_reasoning_effort="low" \
     "Generate ONE image at <SIZE> and save it as <OUTPATH> in the current working directory (copy it there if the image tool saves it elsewhere first). Subject: <USER'S PROMPT, verbatim + any style notes>. After saving, print the absolute path on its own line prefixed with 'SAVED: '." </dev/null
   ```

3. Read the path from the `SAVED:` line (or `find ~/.codex/generated_images -iname '*.png' -mmin -5`),
   then **Read the PNG** so it renders inline.

## Gotchas (already solved — don't re-derive these)
- `--skip-git-repo-check` is REQUIRED, or Codex refuses to run outside a trusted/git directory.
- `-s workspace-write` is REQUIRED so it can write the file to disk.
- `</dev/null` stops `codex exec` from hanging while it waits on stdin.
- `-c model_reasoning_effort="low"` keeps it fast/cheap — Codex may default to a high reasoning
  effort, which is overkill for image gen and burns far more plan tokens.
- The raw image ALWAYS lands in `~/.codex/generated_images/<session>/ig_*.png` first. Copy (don't
  move) it to the requested path.
- Sizes: pass what the user asks — `1024x1024` (square, default), `1536x1024` (landscape),
  `1024x1536` (portrait).

## Notes
- Windows: Claude Code's Bash tool runs in Git Bash, so the snippets above work as-is
  (`~` = `%USERPROFILE%`, Windows env vars like `$LOCALAPPDATA` are inherited). Generated
  images land in `%USERPROFILE%\.codex\generated_images\` just like on macOS/Linux.
- Usage counts against your ChatGPT/Codex plan limits, not API billing.
- For several images, ask Codex for variations in one call, or loop the command.
- Swap in `-m <model>` to force a cheaper/faster agent model.
- Same engine as the `codex-image-bridge` plugin (RolandOne/codex-image-generation), but
  PATH-independent and self-owned — no third-party plugin auto-running shell.
