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
     "Generate ONE image in <ASPECT: square / wide landscape banner / portrait> composition. Subject: <USER'S PROMPT, verbatim + any style notes>. Then resize the result to EXACTLY <W>x<H> with python3 + PIL (use `ImageOps.fit` for exact dims without distortion) and save it as <OUTPATH> (if `<OUTPATH>` is outside the dir you launched Codex from, write to `/tmp/<slug>.png` and move it afterward — see Gotchas; locate the source in this run's own session-id dir, not a global glob). After saving, print the absolute path on its own line prefixed with 'SAVED: ' and the pixel dimensions prefixed with 'SIZE: '." </dev/null
   ```
   Skip the resize clause when the user doesn't need exact dimensions — the native render
   (~1–2.2 MP, see Sizes section) is fine on its own.

3. Read the path from the `SAVED:` line, then **Read the PNG** so it renders inline. If you must
   find the file yourself, scope it to THIS run's own session dir —
   `~/.codex/generated_images/<SESSION_ID>/ig_*.png`, where `<SESSION_ID>` is the value Codex prints
   in its `session id:` startup line. Do NOT use a dir-wide newest-file glob like
   `find ~/.codex/generated_images -mmin -5` — it cross-picks other runs' images under parallelism
   (see "Running several at once").

## Gotchas (already solved — don't re-derive these)
- `--skip-git-repo-check` is REQUIRED, or Codex refuses to run outside a trusted/git directory.
- `-s workspace-write` is REQUIRED so it can write the file to disk.
- `</dev/null` stops `codex exec` from hanging while it waits on stdin.
- `-c model_reasoning_effort="low"` keeps it fast/cheap — Codex may default to a high reasoning
  effort, which is overkill for image gen and burns far more plan tokens.
- The raw image ALWAYS lands in `~/.codex/generated_images/<SESSION_ID>/ig_*.png` first — each run
  gets its OWN session-id subdir (the id is in Codex's `session id:` startup line). Copy (don't
  move) it to the requested path, and resolve it from THAT subdir — never a dir-wide newest-file
  glob, which breaks under parallelism (see "Running several at once").
- `-s workspace-write` only allows writes under the Codex **workdir + `/tmp` + `$TMPDIR`**, NOT
  arbitrary paths. If `<OUTPATH>` is outside the dir you launched Codex from, the save fails with
  `PermissionError` and Codex quietly falls back to `/tmp`. Don't assume cwd == output location —
  they differ whenever you invoke this skill from one repo but target another. Robust fix: have
  Codex save to `/tmp/<slug>.png` and `mv` it to `<OUTPATH>` yourself; or pass `--cd <output-dir>`
  so the destination is inside the workspace.
- Use **`python3`** (not `python`) for the resize — bare `python` is usually not on PATH and Codex
  hits `command not found: python`.

## Running several at once (parallel / batch)

**Run the batch — don't over-orchestrate it.** The deliverable is images on disk, fast. A real session
burned **~14 min on 50 images that should take ~3** by going off-task: building elaborate concept /
matrix documents, smoke-testing one image then rewriting, debugging its own launcher twice, narrating
tangents. For a batch request: pick sane defaults (ask at most ONE quick question, and only if genuinely
blocked), **reuse the proven single-wave script below — do not re-derive it**, launch it in the
background, and report. No planning docs, no smoke-test-then-rewrite loop, no tangents. Stay on the images.

**Launcher choice — default to a shell loop, NOT one-agent-per-image.** The image work is the same
`codex exec` call either way; the launcher only changes orchestration cost. A shell loop backgrounds
N runs at once for ~free; spawning one subagent per image costs a full model context each
(benchmarked head-to-head: **~316k tokens for 5 agents vs ~0 for the equivalent shell script**, same
wall-clock and image quality) — and subagents can *misjudge* (in that test one bailed on a benign
warning whose image had actually rendered fine). Reserve subagents for work that needs per-item
*reasoning* (bespoke prompt design, self-critique-and-retry), never for mechanical fan-out.

**Two things that bite under concurrency (both seen in practice):**
1. **Never use a dir-wide "newest PNG" lookup across concurrent runs.** Each `codex exec` writes to
   its OWN `~/.codex/generated_images/<SESSION_ID>/` subdir (the id is in its `session id:` stdout
   line), so a global `find ~/.codex/generated_images -mmin -N` returns a *sibling run's* image and
   two outputs come back byte-identical. Resolve each output from its own session-id subdir. Most
   robust pattern: run **generate-only**, then resize/place each from its session dir yourself (this
   also sidesteps the sandbox write-path trap — Codex only writes under `~/.codex`).
2. **Distinct filenames up front, then verify.** Name each `out-1.png`, `out-2.png`… and after the
   batch confirm they differ (`md5`/`md5sum`). Matching hashes = the cross-pick above → regenerate
   dupes **solo**. md5 only catches *byte-identical* dupes, though — for a *variety* set also
   **diversify setting + wardrobe in the prompts up front**, or you get five different-but-samey
   shots that read as duplicates.

**Reference batch — single-wave + hardened (portable; see cross-platform notes):**
```bash
CODEX="$(command -v codex || echo /Applications/Codex.app/Contents/Resources/codex)"
PY="$(command -v python3 || command -v python)"
WORK=/tmp/imgbatch; mkdir -p "$WORK"; rm -f "$WORK/USAGE_LIMIT"
ENTRIES=("slug1|||scene one" "slug2|||scene two")          # one element per image
job() {                                                    # generate-only → resolve OWN session → place → tick
  local slug="$1" body="$2" log="$WORK/log-$3.txt"
  "$CODEX" exec --skip-git-repo-check -s workspace-write -c model_reasoning_effort="low" \
    "<PREAMBLE> SCENE: $body  Just generate the image; do not resize." </dev/null >"$log" 2>&1
  grep -q "hit your usage limit" "$log" && { echo "  ⛔ usage-limit $slug"; touch "$WORK/USAGE_LIMIT"; return; }
  local sid img=""; sid=$(grep -m1 'session id:' "$log" | awk '{print $NF}')
  [ -n "$sid" ] && img=$(find "$HOME/.codex/generated_images/$sid" -name 'ig_*.png' 2>/dev/null | head -1)  # find: no zsh "no matches found"
  if [ -n "$img" ]; then
    "$PY" -c "from PIL import Image,ImageOps; ImageOps.fit(Image.open('$img'),(1080,1920),Image.LANCZOS).save('OUTDIR/$slug.png')" \
      && echo "  ✓ $slug" || echo "  ✗ resize $slug"     # live progress tick per completion
  else echo "  ✗ gen $slug"; fi
}
i=0
for e in "${ENTRIES[@]}"; do job "${e%%|||*}" "${e##*|||}" "$i" & i=$((i+1)); done   # SINGLE WAVE: launch all, no K-barrier
wait
# verify: ls OUTDIR | wc -l   ·   for f in OUTDIR/*.png; do md5 -q "$f"; done | sort | uniq -d   (dupes → regen solo)
```

**Two foot-guns that silently waste a launch (both hit in practice):**
- **Don't assemble the prompt preamble with `$(cat <<'EOF' … EOF)`.** Heredoc inside command-substitution
  misparses on macOS bash — especially with `(` / `)` in the text — and the script dies *before generating
  anything*. Assign it as a plain single-quoted variable instead: `PRE='…'`.
- **Don't double-background.** When launching via the `run_in_background` tool, run the script **directly** —
  never wrap it in `nohup … &` or a trailing `&`. Double-detaching means (a) the `✓` ticks never reach the
  task-output file (the chip looks dead the whole time) and (b) the *completed* notification fires instantly
  while work is still running, so you can't tell when it actually finished. The per-job `&` *inside* the
  script is correct; the script itself stays foreground in the background shell.

**Concurrency — go WIDE, single wave (measured + replicated: 2 brands, 6× K=50 runs, same-content A/B).**
Counter-intuitively, **launch the whole batch at once with one `wait` (as the snippet above does) — do
NOT split into K-sized waves with a barrier.** A `wait`-every-K loop drains and refills the API pipe at
each wave edge and is much slower: 50 square posts ran **243 s at K=30 (2 waves) vs 144 s at K=50 (1 wave)**
— ~40 % faster with no barrier. What the testing showed:
- The API **pipelines ~25–30 concurrently**; a single wave keeps that pipe continuously full → fastest. Extra launches just queue (harmless), they don't stall at a barrier (costly).
- **RAM:** a 50-wide wave peaks ~6–7 GB of `codex` RSS and saturates a 16 GB Mac (memory compressor pegged) — yet it's **stable: 0 failures across every K=50 run**. The machine's *baseline* fullness, not K, dominates memory; closing other apps helps more than lowering K. Only if RAM is genuinely tight, use a **rolling pool** (replace each finished job instantly, keep ~25–30 in flight) — caps RAM *without* the barrier stall. Never use the slow `wait`-every-K barrier.
- The **only hard cap is your plan's usage quota** — not concurrency, not RAM. Hitting it returns in ~5 s per call with `ERROR: You've hit your usage limit … try again at <time>` and writes no image → detect that string and **abort** (the snippet's `USAGE_LIMIT` flag) instead of grinding out zeros.
- **Foreground for small, background for big — and always show real progress.** **≤ ~6 images: run FOREGROUND** (well under the 600 s cap), then Read + display them in the *same reply* — no background-chip black-box, results land inline (a 2-image, ~90 s job backgrounded is pure downside). **> ~6 images (or > ~5 min): background.** Either way, emit the `✓ <slug>` tick **from inside each per-image job the moment it lands — NEVER from a post-`wait` loop** (a post-wait loop dumps every tick at once at the very end, which reads as "no progress" even though it "ticked"). **Retry** failures once at lower concurrency (reword on moderation); finish with a count + `md5` distinctness check. (And don't double-background — see foot-guns above.)

**Cross-platform.** This shell approach runs on macOS, Linux, and **Windows via Git Bash** (the shell
Claude Code's Bash tool uses on Windows) — **not** PowerShell/cmd. Portability rules, all applied above:
- **Arrays:** macOS's tool shell is often **zsh (1-indexed)**, Linux/Git-Bash is **bash (0-indexed)** —
  a hardcoded `for i in 0 1 2 …` over an array silently shifts/drops on one of them. Iterate
  **elements** (`for e in "${arr[@]}"`), identical in both.
- **No `sips`** (macOS-only) — use **Python/PIL** for resize *and* dimensions.
- **`md5` vs `md5sum`** — macOS has `md5 -q`, Linux/Git-Bash have `md5sum`; detect, or use Python `hashlib`.
- **Resolve `codex` and `python`/`python3` dynamically** (`command -v`) — paths + interpreter names differ per OS.
- If there is no bash-like shell at all, the subagent launcher is the shell-agnostic fallback (at the token cost above).

Orchestrating with subagents (when warranted): tell each to report its `session id:` and resolve from
that dir; then you verify distinctness and re-run any duplicates yourself.

## Sizes & resolution (verified empirically — the API docs do NOT apply here)

The OpenAI Image API exposes `size` (any WxH up to 3840px edge, 4K) and `quality` (low/medium/high)
parameters — but **Codex's built-in image tool on ChatGPT auth exposes NONE of them**. It is
prompt-only: explicit pixel requests like "generate at 3840x2160" are silently ignored (tested:
came back 1672x941). Native output is always ~1–2.2 MP at model-chosen dimensions.

What you CAN control, and how:
- **Aspect ratio** — steer it with prompt wording, it works reliably:
  - "square composition" → ~1024–1254px square
  - "wide landscape banner" → ~2.4:1 (e.g. 1949x807, 2172x724)
  - "portrait" → ~2:3 (e.g. 1024x1536)
  - Ratios cap at ~3:1; don't ask for wider.
- **Exact dimensions** — generate at the right aspect ratio, then resize locally. Use **`python3`**
  + PIL `ImageOps.fit(im, (W,H), Image.LANCZOS)` (cover + center-crop → exact size, no distortion);
  a plain `resize((W,H))` stretches faces when the aspect differs. `sips` works on macOS too. Tell
  Codex to do this in the same `exec` call: "...then resize the result to EXACTLY <W>x<H> with
  python3 PIL ImageOps.fit (LANCZOS) and save as <OUTPATH>." (Bare `python` is usually not on
  PATH — use `python3`.)
- Common targets: square post 1080x1080 (gen square), FB cover ~2.05:1 (gen wide banner),
  IG story 1080x1920 (gen portrait), YT thumbnail 1280x720 (gen landscape).
- **Upscales add pixels, not detail** — a 2x LANCZOS upscale of a ~2MP native render is fine for
  display sharpness, but tell the user that real detail tops out at the native render. True native
  4K requires the Image API with an `OPENAI_API_KEY` (Codex's CLI fallback) — not ChatGPT auth.

## Reference images & edits (verified working)

The built-in tool handles generate-from-reference and edits — but the reference **must be a real
file on disk that Codex can open by absolute path** (its `view_image` tool reads from the
filesystem, nothing else). Settle this BEFORE you generate:
- In the prompt, tell Codex: "First, load <ABSPATH> with your view_image tool. It is a STYLE
  REFERENCE ONLY (not an edit target): <describe what to take from it>." Then describe the new
  image. Labeling the role matters — Codex distinguishes style reference / edit target /
  compositing input.
- Multiple references work (e.g. product shot + scene → composite; several items → one image).
- Edits support object removal/replacement, text replacement, lighting/weather, style transfer,
  identity-preserve, sketch-to-render. Mask-guided edits are prompt-based guidance, not
  pixel-precise.
- **A pasted/inline chat image is NOT something Codex can use — and you can't hand it through
  either.** When the user pastes an image into chat, you (Claude) can *see* it, but Codex's
  `view_image` has no file to open, so the reference is effectively lost unless you put it on disk
  first. In order of reliability: (1) **ask the user for the file path** — by far the most reliable,
  and the recommended default; (2) only if they can't, extract the base64 block from the newest
  session transcript (`~/.claude/projects/<project-slug>/*.jsonl` — records have
  `message.content[].type == "image"` with `source.data` + `source.media_type`), decode it to a temp
  file, and pass that path — then confirm the written file opens and is the right image before
  relying on it.

## Model facts that DO carry over from the gpt-image-2 API docs

- **No transparent backgrounds** — gpt-image-2 doesn't support them. Codex's own imagegen skill
  works around this with a chroma-key background + local removal script; just ask Codex for "a
  transparent PNG" and it runs that pipeline itself.
- **Text rendering** is good but imperfect — quote exact wording verbatim in the prompt, keep it
  short, and check the output; regenerate on typos. Precise layout/composition control is weak.
- **Latency**: complex prompts can take up to ~2 minutes — keep the Bash timeout generous (600s).
- **Moderation**: blocked prompts fail at input or output stage. Don't retry verbatim — reword
  (remove targeting/brand/person specifics) and explain to the user.

## Notes
- Windows: Claude Code's Bash tool runs in Git Bash, so the snippets above work as-is
  (`~` = `%USERPROFILE%`, Windows env vars like `$LOCALAPPDATA` are inherited). Generated
  images land in `%USERPROFILE%\.codex\generated_images\` just like on macOS/Linux.
- Usage counts against your ChatGPT/Codex plan limits, not API billing.
- For several images, ask Codex for variations in one call, or loop the command.
- Swap in `-m <model>` to force a cheaper/faster agent model.
- Same engine as the `codex-image-bridge` plugin (RolandOne/codex-image-generation), but
  PATH-independent and self-owned — no third-party plugin auto-running shell.
