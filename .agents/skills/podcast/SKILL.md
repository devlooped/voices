---
name: podcast
description: Receive an X post URL, retrieve full content and post date, then for each supported podcast language: create lang/yyyy/MM/ folder, translate if needed, generate short title + summary + keywords (if no post cover image has been determined yet, also request an English "imagine" prompt from the LLM), determine episode artwork (prefer X article header photo from data.article.cover_media if present — download + center-crop to square via ffmpeg; else use the imagine prompt to launch `/imagine` + 1:1), copy the artwork as #-slug.jpg alongside the mp3 for *every* language, write #-slug.md (YAML frontmatter with title/summary/date/link/author + spoken content in `title\nby <author>, posted on <date>\n\n<content>` structure), synthesize #-slug.mp3 using male/female voice chosen according to post author (via voices.toml + Azure Dragon HD/Omni), convert MP3 to WAV then generate #-slug.srt via speech recognition (--continuous) + post-processing of RECOGNIZED output, update the language feed.xml (newest-first item with enclosure + itunes:image + Podcasting 2.0 <podcast:transcript>), and maintain lang/index.html (episode table + local audio players). Local files only — upload (including the jpgs), git commit ("Add episode [#] - [title]"), and push are handled by a separate upload skill. Uses xurl, dnx Microsoft.CognitiveServices.Speech.CLI, az CLI, ffmpeg, ffprobe.
---

# Podcast — X Post to Multi-Language Local Episode Generator

Turn an X post into ready-to-publish podcast episodes for every supported language. The skill produces local files and updates local feed/index artifacts only. A separate `upload` skill performs Azure blob upload (azcopy), git commit with message "Add episode [#] - [title]", and push.

## What success looks like

When finished, all of the following must be true:

1. The X post was fetched via `xurl read` and its original (attributed) content archived to `posts/<yyyy-MM-dd>-<post-id>.md` with YAML frontmatter (original metadata + link + author + date) + raw text.
2. For every supported podcast language (en, es from voices.toml):
   - Directory structure `<lang>/<yyyy>/<MM>/` exists for the post date.
   - `<N>-<slug>.md` exists with YAML frontmatter (`title`, `summary`, `date`, `link`, `author`) followed by the spoken post content in this exact structure (plain text, no markdown):
     ```
     <title>
     by <author>, posted on <date>

     <content>
     ```
   - `<N>-<slug>.mp3` was synthesized using the gender-appropriate voice (inferred from post author name, default male) resolved from the effective `voices.<lang>.<gender>` section, encoded as **48 kHz mono MP3 at 192 kbps** (`audio-48khz-192kbitrate-mono-mp3` — the highest quality MP3 preset the Azure Speech CLI offers).
   - `<N>-<slug>.srt` transcript was generated reliably: MP3 → 16 kHz mono WAV (ffmpeg) → `dnx ... recognize --continuous` (log capture) → parse `RECOGNIZED:` lines → timed .srt using probed duration (proportional char length or fixed). The .srt sits next to the .mp3.
   - Artwork is determined once (if a post cover image hasn't been determined *yet*, the LLM is asked for an English "imagine" prompt which is used to generate an initial one; the same artwork is then copied for all languages): if the fetched post JSON has `data.article.cover_media`, the corresponding photo URL from `includes.media` is downloaded and center-cropped to square (1024x1024) via ffmpeg and saved as `<N>-<slug>.jpg`; otherwise an "imagine" prompt from the LLM JSON is used to launch `/imagine <prompt>, 1:1` and the result is saved the same way. The jpg (using each language's own slug) sits next to the mp3 in *every* language directory. Every language's feed item includes the corresponding `<itunes:image>`.
   - `<lang>/feed.xml` contains the new episode as the **first** `<item>` under `<channel>`, with:
     - Correct `<enclosure>` using `https://{{storage}}.blob.core.windows.net/{{container}}/<lang>/<yyyy>/<MM>/<N>-<slug>.mp3`
     - `itunes:season` = year, `itunes:episode` = N (per-language TV-series numbering)
     - Accurate `itunes:duration`, file length, MIME `audio/mpeg`
     - `<podcast:transcript>` (Podcasting 2.0) pointing at the future blob `.srt` URL
     - `<itunes:image>` (for the per-language copy of the episode artwork jpg)
   - `<lang>/index.html` exists or is updated with a simple table of episodes (newest/highest N first) including HTML5 `<audio controls>` players for local testing.
3. GUIDs are always `yyyy-MM-dd-N-slug`.
4. Channel `lastBuildDate` is set to current GMT; per-item `pubDate` uses the X post's created_at.
5. No Azure upload, no git operations, and no remote changes occurred. All artifacts are local and ready for the upload skill.
6. The user is given episode numbers, titles, file paths, and the future blob URLs for each language.

## Architecture & Layout

**Episode file naming:** `<N>-<slug>.md`, `<N>-<slug>.mp3`, `<N>-<slug>.srt`, `<N>-<slug>.jpg` (N = sequential episode within year for that language feed). The .jpg (episode artwork) uses the *same* visual for every language but is named using that language's title-derived slug so it sits next to its mp3.

**Directory layout per language:**
```
<lang>/
  feed.xml
  index.html
  cover.png
  <yyyy>/
    <MM>/
      <N>-<slug>.md
      <N>-<slug>.mp3
      <N>-<slug>.srt
      <N>-<slug>.jpg
```

**Original archive (never translated):**
```
posts/
  <yyyy-MM-dd>-<post-id>.md     # YAML + raw attributed source text
```

**Enclosure + artwork URLs (written into feed):**  
`https://<storage>.blob.core.windows.net/<container>/<lang>/<yyyy>/<MM>/<N>-<slug>.mp3`  
(and matching `.srt` for the transcript tag, and `.jpg` for `<itunes:image>`).

**Episode numbering (TV-series model):**  
`itunes:season` = calendar year of the post date  
`itunes:episode` = next sequential number for that year **per language feed**.

**Reference feed:** Use patterns from `C:\Code\podcast\feed.xml` and the existing `en/feed.xml` / `es/feed.xml` for item shape, namespaces, and tone.

## Prerequisites

| Tool | Purpose | Verify | Install (Windows) |
|------|---------|--------|-------------------|
| xurl (required fork) | Fetch full X post content (note_tweet / article / text) | `xurl version` ends with "by @kzu" | `go install github.com/kzu/xurl@v0.1.0` |
| dnx / .NET 10 SDK | Run Azure Speech CLI (synthesize + recognize) | `dotnet --version` (≥ 10) and presence of `dnx` (e.g. `Get-Command dnx` or `C:\Program Files\dotnet\dnx.cmd`) | Install .NET 10.0 SDK (dnx is included as `dotnet dnx`) |
| az CLI | Fetch Azure Speech key + region from voices.toml `[azure]` | `az account show` | `winget install Microsoft.AzureCLI` |
| ffmpeg + ffprobe | Convert MP3→WAV for reliable recognition + probe duration/size | `ffmpeg -version` and `ffprobe -version` | `winget install ffmpeg` (includes ffprobe) |
| jq (recommended) | Parse JSON (xurl responses, ffprobe) | `jq --version` | `winget install jqlang.jq` |

**xurl setup & secret safety (MANDATORY — inline from ecosystem patterns):**

- Never read, print, parse, or send `~/.xurl` (or copies) to the LLM.
- Never use `--verbose` or pass secret flags (`--bearer-token`, etc.) in agent sessions.
- Always verify with `xurl auth status` only.
- The @kzu fork is required for full long-form content.

Before running:
1. `command -v xurl` (install via go if missing).
2. `xurl version` — must end with `by @kzu`. If not, ask user permission to replace and re-install the fork.
3. `xurl auth status` — detect available auth. **Prefer `app`** (bearer: ✓). Fall back only if needed. Use `--auth app` for reads when available.
4. `ffmpeg -version` (required for MP3→WAV conversion for recognition).
5. Run `xurl read <url>` (never with inline secrets).

If no usable auth: tell the user to run `xurl auth oauth2` (or appropriate) **outside** the agent session.

## Robust execution patterns (this environment)

- **Capture noisy / long output first**: For `dnx ...`, `az ...`, and help commands, use `... 2>&1 | Out-File $tmp` (or `> $tmp`), then inspect with `Get-Content $tmp`. Direct `| cat`, `| head`, `| Select` on dnx often triggers harness "Get-Content cannot bind" spam.
- **Prefer PowerShell cmdlets**: `Get-ChildItem` / `Get-Content -Raw` / `Out-File -Encoding utf8` / `ConvertFrom-Json` / `Select-String` over `ls`, `cat`, Unix pipes for reliability on Windows pwsh.
- **UTF-8**: When writing SSML, transcripts, or .md files use `-Encoding utf8`. When reading xurl JSON use `-Raw | ConvertFrom-Json`.
- **Duration**: Use `[TimeSpan]::FromSeconds([double]$raw).ToString("m\:ss")`.
- **Temporary files**: Use `$env:TEMP\...` for ssml, wav, logs, and json. Clean up in 4.9 unless debugging.
- **Long synthesis**: Expect many "SYNTHESIZING: audio.length=..." messages — redirect or tail as needed.

## Workflow checklist

```
- [ ] Accept X post URL (or ID); extract ID if needed
- [ ] Ensure xurl (@kzu fork) + auth (app preferred); verify version ends "by @kzu"
- [ ] xurl read the post; save JSON; check for errors
- [ ] Inspect JSON for `data.article.cover_media`; if present resolve photo URL from `includes.media` (store for artwork decision)
- [ ] Extract full body (note_tweet priority → article → text) + author line
- [ ] Archive original attributed text to posts/<date>-<id>.md (yaml + raw)
- [ ] Infer author gender from name (default male) for voice selection
- [ ] For each supported language (en, es):
    - [ ] Translate full text if source lang != target (one pass per lang)
    - [ ] Single LLM call → JSON {title, summary, keywords} (if no post cover image has been determined yet, also request an English "imagine" prompt)
    - [ ] Compute yyyy/MM, slug (kebab), next N for (lang, year) from feed
    - [ ] Write <lang>/<yyyy>/<MM>/<N>-<slug>.md (yaml frontmatter + spoken body using the required structure: title\nby <author>, posted on <date>\n\n<content>)
    - [ ] If no artwork determined yet: if article cover photo URL was resolved earlier, download it and center-crop to square 1024x1024 via ffmpeg as <N>-<slug>.jpg (skip /imagine); else (if imagine in this JSON) start subagent `/imagine <imagine>, 1:1 aspect ratio`; copy result (or cover-derived jpg) as <N>-<slug>.jpg into *this* language's dir (and later for other langs too)
    - [ ] Resolve voices.<lang>.<gender> + build SSML + synthesize .mp3 via dnx (`--format audio-48khz-192kbitrate-mono-mp3`)
    - [ ] Probe duration/size with ffprobe
    - [ ] Convert .mp3 to 16 kHz mono .wav (ffmpeg) for reliable recognition
    - [ ] Run recognize --continuous on the .wav (capture to temp log file first), extract RECOGNIZED: lines, build timed .srt using total duration + proportional lengths
    - [ ] Update <lang>/feed.xml (prepend item + itunes:image + podcast:transcript tag + current lastBuildDate)
    - [ ] Update <lang>/index.html (newest-first table + <audio> players)
- [ ] Report episode details + future blob URLs per language
- [ ] Confirm zero upload / git activity
```

---

## Step 1 — Receive X post and fetch content

**Goal:** Obtain the full post text, author, created_at date, and canonical link.

- Accept URL (https://x.com/.../status/...) or numeric ID from the user prompt.
- Extract ID if a URL is given.
- Run (with correct auth):
  ```
  xurl [--auth app] [--app NAME] read <url-or-id>
  ```
- Save full JSON to a temp file (e.g. `$env:TEMP\podcast-<id>.json`). **Do not rely on terminal display** — xurl output can appear garbled for non-ASCII in the console. Always read the saved file with `ConvertFrom-Json` (or `jq`) using UTF-8.
- Validate: no top-level `errors`; `data` present.

**Extract body (strict priority):**
1. `data.note_tweet.text` (long-form, preferred for full posts)
2. `data.article` → join `title` + `plain_text` with blank line
3. `data.text` (fallback)

**Recommended extraction (PowerShell, robust):**
```pwsh
$json = Get-Content $tmp -Raw | ConvertFrom-Json
$body = if ($json.data.note_tweet) { $json.data.note_tweet.text }
        elseif ($json.data.article) { $json.data.article.title + "`n`n" + $json.data.article.plain_text }
        else { $json.data.text }
```

**Cover photo for artwork (if X article):** Also extract once here for later use (before any language processing):
```pwsh
$coverMediaKey = $json.data.article.cover_media
$coverPhotoUrl = if ($coverMediaKey -and $json.includes.media) {
    ($json.includes.media | Where-Object { $_.media_key -eq $coverMediaKey -and $_.type -eq 'photo' } | Select-Object -First 1).url
}
```
Store `$coverPhotoUrl` (may be $null). Plain `xurl read` (with the @kzu fork) already includes `article` + `includes.media` for qualifying posts.

**Author line:** From `includes.users` matching `data.author_id`:
- `By {name} (@{username})` or `By @{username}` (store this for the YAML `author` field)

Also capture the raw post date (`data.created_at` as `yyyy-MM-dd`) and keep the main post body.

**Attributed source text (for archive + translation + title generation input):** prepend author line + blank line + body.

**Spoken/read-aloud content (for episode .md body and TTS):** constructed *after* title generation (see 4.4).

**Date & link:**
- Use `data.created_at` for episode date (folders + item pubDate).
- Canonical link: the input URL or construct `https://x.com/{username}/status/{id}`.

**Outcome:** Attributed full source text (for archive/translate) + post date + link + author info. The spoken content structure is applied later using the generated title + `by <author>, posted on <date>` + main content.

---

## Step 2 — Archive the original post

**Goal:** Persist the exact source (never translated) for future reference.

- Create `posts/` directory if missing.
- Write `posts/<yyyy-MM-dd>-<post-id>.md`:
  ```yaml
  ---
  id: "<post-id>"
  link: "https://x.com/..."
  author: "By Name (@user)"
  date: "2026-06-18"
  lang: "en"
  ---
  By Name (@user)

  <original attributed body exactly as extracted>
  ```
- This file is committed later by the upload skill.

**Outcome:** Permanent original-language archive (uses original author + body attribution; this is *not* the spoken/read-aloud version).

---

## Step 3 — Determine voice gender from author

Infer `male` or `female` from the author's display name / username (LLM judgment on common names or self-presentation).

- Default to `male` if ambiguous.
- Use the **same** chosen gender for synthesis in **every** target language for this episode.

---

## Step 4 — Process each supported language

Supported languages are discovered from `voices.toml` (`[podcast.en]`, `[podcast.es]`, ... or `[voices.*]` keys).

For each `lang`:

### 4.1 Translate (if needed)
If the detected source language differs from the target, translate the full attributed text (author line + main post body) into the target language. Preserve paragraphs and structure. The resulting translated text is used as input for title/summary generation. The main translated content (without the author prefix) will be used later when assembling the spoken content for TTS.

### 4.2 Generate title, summary, keywords (and imagine prompt if needed)
For every language, ask the LLM once for a JSON object.

If a post cover image has not been determined yet, request the 4-field form (the "imagine" value must be written in English, as the resulting artwork is language-agnostic):

```json
{
  "title": "Short, clean podcast episode title",
  "summary": "Concise 1-2 sentence description for <description> and <itunes:summary>",
  "keywords": "comma,separated,keywords,for,itunes",
  "imagine": "A vivid, self-contained prompt suitable for /imagine (grok imagine) describing a compelling square podcast artwork for this episode. Subject first, then mood/composition/lighting. No text in the image."
}
```

Once an artwork source (cover or generated) has been determined, use the 3-field form for any remaining languages (no imagine).

Use the translated attributed text as the source for generating the episode title/summary/keywords. The imagine value (when present) will be used immediately after this step to generate artwork **unless** an X article cover photo URL was resolved in Step 1 (in which case the cover photo is used instead and the imagine value is ignored). The imagine prompt is always requested/produced in English.

### 4.3 Compute paths and episode number
- `yyyy`, `MM` (zero-padded) from post `created_at`.
- Slug: kebab-case, lowercase, ascii, safe for filenames/URLs (derived from title). Sanitize: remove diacritics, replace non-alphanum with `-`, collapse multiple dashes, trim. Example: "SpaceX: Not an IPO — It's a Referendum" → `spacex-not-an-ipo-its-a-referendum`.
- Next `N`:
  - Load `<lang>/feed.xml`
  - Parse `<itunes:episode>` + `<itunes:season>` (or default to 0). Only consider items whose season matches the post year.
  - `N = (max for this year) + 1` (or 1 if none)
  - Practical pattern (simple feeds): load as text or `[xml]`, look for `<itunes:episode>` / `<itunes:season>` (or `episode`/`season` after casting). When no items exist for the year, use 1. Fall back to directory scan of `<lang>/<yyyy>/` subfolders + filenames if feed parsing is painful.
- Target directory: `<lang>/<yyyy>/<MM>/`
- Files: `<N>-<slug>.md`, `<N>-<slug>.mp3`, `<N>-<slug>.srt`, `<N>-<slug>.jpg`

### 4.3.1 Generate episode artwork (once, shared visual)
Performed the first time an episode needs artwork (cover or generated) during language processing. The resulting image is language-agnostic and will be copied for every language.

**Precedence (decide once):**
- If a `$coverPhotoUrl` was resolved in Step 1 (from `data.article.cover_media` + matching photo in `includes.media`):
  - Download to a temp file: `Invoke-WebRequest -Uri $coverPhotoUrl -OutFile "$env:TEMP\cover-$postId.jpg"` (public pbs.twimg.com URLs; no auth needed).
  - Ensure the target episode directory exists.
  - Center-crop to square and scale to 1024x1024 (matching existing episode artwork) using ffmpeg:
    ```pwsh
    ffmpeg -y -i "$env:TEMP\cover-$postId.jpg" `
      -vf "crop='min(iw,ih)':'min(iw,ih)':'(iw-min(iw,ih))/2':'(ih-min(iw,ih))/2',scale=1024:1024:flags=lanczos" `
      "<lang>/<yyyy>/<MM>/<N>-<slug>.jpg"
    ```
  - Remember the produced local path as the artwork source.
  - **Do not** start any `/imagine` subagent.
  - On download or ffmpeg failure, log a warning and fall back to the imagine path below if an imagine value exists.
- Else (no cover photo):
  - If the JSON from 4.2 contains a non-empty `imagine` value (requested because no cover was determined yet):
    - Start a subagent with exactly: `/imagine <the imagine string>, 1:1 aspect ratio`
    - Wait for the subagent to complete. It will follow the imagine skill and use image generation to produce an artwork file; the result will include the path to the generated image.
    - Save the produced image (copy + rename as needed) as `<N>-<slug>.jpg` inside the current episode directory.
  - If no imagine value or the subagent yields no usable image, simply continue without a jpg for this episode (graceful; `<itunes:image>` will be omitted).

After artwork is successfully produced (cover or generated), remember the source image path. When later processing other languages (after computing their N and slug), copy the same image bytes using each language's `<N>-<its-slug>.jpg`. This ensures the identical visual sits alongside every language's mp3.

### 4.4 Write the episode .md (plain text)
Create the directory.

Write YAML frontmatter, then the spoken post content with this exact structure (the body is what gets synthesized):
```yaml
---
title: "..."
summary: "..."
date: "2026-06-18"
link: "https://x.com/..."
author: "By Name (@username)"
---
<title>
by <author>, posted on <date>

<full translated post content — plain text only, no **bold**, no markdown lists, no formatting>
```

Example body:
```
SpaceX: Not an IPO — It's a Referendum
by Brivael Le Pogam (@brivael), posted on 2026-06-17

Retail activity on SpaceX is abnormal. And it's just the beginning.
...
```

The body after the frontmatter is the complete text that will be spoken (starting with the generated title, the exact byline, then the content). Use the generated `title` for the first line, reformat the author (drop leading "By " if present) for the byline, and use the post date in `yyyy-MM-dd`. The main content is the translated substantive post text.

### 4.5 Synthesize the MP3 (Azure Dragon HD voices)

Read `voices.toml` at repo root.

Resolve effective config for `voices.<lang>.<gender>` (hierarchical merge):
- Start with `[voices]`
- Overlay `[voices.<lang>]`
- Overlay `[voices.<lang>.<gender>]`

Extract `voice` (e.g. `en-us-Andrew3:DragonHDLatestNeural`) and parameters (`temperature=0.85;top_p=1.0;...`).

Fetch credentials (inline only):
```pwsh
$key = az cognitiveservices account keys list --name <speech> --resource-group <rg> --query key1 -o tsv
$region = az cognitiveservices account show --name <speech> --resource-group <rg> --query location -o tsv
```

Build SSML (use correct `xml:lang`):
- `voices.en.*` → `en-US`
- `voices.es.male` (es-us-alonso) → `es-US`
- `voices.es.female` (es-mx-coralpasteque) → `es-MX`

Template:
```xml
<speak version='1.0' xmlns='http://www.w3.org/2001/10/synthesis'
       xmlns:mstts='http://www.w3.org/2001/mstts' xml:lang='LANG'>
  <voice name='VOICE' parameters='PARAMS'>ESCAPED_TEXT</voice>
</speak>
```

Escape only `& < >`. Write to a temp `.ssml` file.

**Audio output format (podcast default):** Use the highest-quality MP3 preset, not bare `--format mp3` (which defaults to 16 kHz / 128 kbps). The recommended podcast setting is:

```
audio-48khz-192kbitrate-mono-mp3
```

(48 kHz sample rate, 192 kbps mono — best MP3 option in `spx help synthesize mp3`.) File size is larger than the CLI default, but compression artifacts are minimized. The transcript pipeline still down-converts to 16 kHz mono WAV for recognition, so this does not affect `.srt` generation.

Synthesize (exact form required by AGENTS.md):
```pwsh
$key = az cognitiveservices account keys list --name $speech --resource-group $rg --query key1 -o tsv
$region = az cognitiveservices account show --name $speech --resource-group $rg --query location -o tsv
$ssml = "$env:TEMP\ep.ssml"
$out = "<lang>/<yyyy>/<MM>/<N>-<slug>.mp3"
$format = "audio-48khz-192kbitrate-mono-mp3"
dnx Microsoft.CognitiveServices.Speech.CLI -- synthesize --key $key --region $region --file $ssml --audio output $out --format $format 2>&1 | Out-File "$env:TEMP\synth.log"
```

Synthesis produces many "SYNTHESIZING: ..." progress lines. Redirect to a log file (or use `| Select -Last 5`) for cleaner sessions.

**Never** use plain `--text --voice` for HD voices — parameters must be on the `<voice>` element.

### 4.6 Post-synthesis metadata + .srt transcript

Probe the MP3 (robust PowerShell):
```pwsh
$dur = [double](ffprobe -v quiet -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 $mp3)
$ts = [TimeSpan]::FromSeconds($dur)
$duration = $ts.ToString("m\:ss")     # e.g. "2:35"  (avoid :D2 format specifier pitfalls)
$size = (Get-Item $mp3).Length
```

**Reliable transcript generation (required on Windows without GStreamer):**

1. Convert the synthesized MP3 to a recognition-friendly WAV:
   ```pwsh
   $wav = "$env:TEMP\<N>-<slug>.wav"
   ffmpeg -y -i $mp3 -acodec pcm_s16le -ar 16000 -ac 1 $wav 2>&1 | Out-Null
   ```

2. Run recognition with `--continuous` (capture output to a file first — heavy piping of dnx can trigger harness errors). Use a language code that matches the voice roughly (`en-US`, `es-ES`, or `es-US`):
   ```pwsh
   $log = "$env:TEMP\recog-<N>.log"
   dnx Microsoft.CognitiveServices.Speech.CLI -- recognize `
     --key $key --region $region `
     --file $wav --language en-US --continuous 2>&1 | Out-File $log
   ```

3. Extract segments and build timed .srt (use recognized lines + proportional timing based on total duration):
   ```pwsh
   $lines = Get-Content $log | Select-String '^RECOGNIZED:' | ForEach-Object { ($_ -replace '^RECOGNIZED:\s*','').Trim() } | Where-Object { $_ }
   # Then compute per-segment durations proportional to char length (or use fixed increments) and emit
   # standard .srt blocks: index\nHH:MM:SS,fff --> HH:MM:SS,fff\ntext\n
   ```

Direct `--file *.mp3` almost always requires `--format mp3` (or `any`) and frequently fails with `SPXERR_GSTREAMER_NOT_FOUND_ERROR`. The WAV path above is the reliable cross-run method.

Result must be `<N>-<slug>.srt` sitting next to the .mp3. Use the clean structured spoken text (title + byline + content) for the `.md` body; the .srt may contain ASR imperfections — that's acceptable for captions.

### 4.7 Update <lang>/feed.xml

Load the existing `<lang>/feed.xml`.

- Ensure root has `xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd"`
- Add `xmlns:podcast="https://podcastindex.org/namespace/1.0"` if missing.

Prepend a new `<item>` as the **first** child of `<channel>`.

Item fields (match reference style):
- `<title>`, `<description>` (use generated summary), `<guid>` (exactly `yyyy-MM-dd-N-slug`), `<link>` (original X URL), `<pubDate>` (RFC 822 from post date)
- `<enclosure url="https://<storage>.blob.../<lang>/.../<N>-<slug>.mp3" length="..." type="audio/mpeg" />`
- `<itunes:season>`, `<itunes:episode>`, `<itunes:duration>`, `<itunes:summary>`, `<itunes:keywords>`, `<itunes:explicit>No</itunes:explicit>`
- `<itunes:image href="https://<storage>.blob.../<lang>/.../<N>-<slug>.jpg" />` (present for every language because artwork was copied using the language's slug)
- `<podcast:transcript url="https://<storage>.blob.../<lang>/.../<N>-<slug>.srt" type="text/srt" language="<lang>" />`

Update channel `<lastBuildDate>` to the current GMT time.

Do **not** change the per-item pubDates.

### 4.8 Update <lang>/index.html

Create or overwrite `<lang>/index.html` with a minimal, self-contained page for local testing:

- Title with the podcast name from config.
- Table (newest first): Episode | Title | `<audio controls src="relative/path/to/<N>-<slug>.mp3">`
- Scan the year/month subdirs or parse the just-updated feed.xml to keep the list accurate.
- Use this CSS so the native `<audio controls>` player stays full-size (reserve the Audio column width; `width: 100%` alone collapses the control in a narrow table cell):

```css
table { border-collapse: collapse; width: 100%; max-width: 900px; table-layout: fixed; }
th, td { padding: 0.5rem; border-bottom: 1px solid #333; text-align: left; vertical-align: middle; }
th:first-child, td:first-child { width: 3rem; }
th:last-child, td:last-child { width: 360px; }
audio { display: block; width: 100%; min-height: 40px; }
```

This file is never uploaded to the podcast feed; it is for human convenience when testing locally.

### 4.9 Cleanup
Delete temporary SSML, WAV, JSON, recognition logs, downloaded cover photos (`$env:TEMP\cover-*.jpg`), and any intermediate image files from the subagent unless the user wants them kept for debugging. Keep the final .mp3 + .srt + .md + .jpg (the .jpg must remain next to the mp3 for every language).

---

## Step 5 — Report to the user

For each language report:
- Episode number N and GUID
- Title + summary + keywords
- Paths to .md, .mp3, .srt (and .jpg for the artwork if generated)
- Path to updated feed.xml and index.html
- Future blob enclosure URL for mp3 and .srt (and jpg image URL)
- Author gender / voice chosen
- The imagine prompt that was used (if any, always English) and the artwork source: "X article cover photo (center-cropped to square)" or "generated via /imagine" (or none)

Confirm that only local files were created/modified and nothing was uploaded or committed.

---

## Examples

**Basic usage:**
"podcast https://x.com/someone/status/1234567890123456789"

Agent:
1. Fetches post with xurl, archives original to `posts/2026-06-18-1234567890123456789.md`
2. Infers author gender (e.g. male)
3. For the first language processed: generates title/summary/keywords; if no cover photo has been determined yet, also requests an English "imagine" prompt from the LLM. If the xurl JSON had `data.article.cover_media`, downloads + ffmpeg center-crops it to square as the jpg (no subagent); otherwise uses the imagine to start `/imagine <imagine>, 1:1` and saves the result. Writes the jpg using that language's slug, writes the .md (with spoken body in the required `title\nby <author>, posted on <date>\n\n<content>` structure), synthesizes, produces .srt, updates its feed (incl. itunes:image) + index.html.
4. For subsequent languages: translate as needed, generate title/summary/keywords (3-field, no imagine since artwork already determined), copy the same artwork jpg using the language-specific slug next to the mp3, write .md + .mp3 + .srt + .jpg, update feed + index.html.
5. Reports everything (including per-lang jpg paths + image blob URLs); all local.

---

## Configuration (`voices.toml`)

See current file. Hierarchical:
`[voices]` → `[voices.<lang>]` → `[voices.<lang>.<gender>]`

`[azure]` provides `rg`, `speech`, `storage`, `container`.

`[podcast.<lang>]` provides channel title/description/language for reference.

---

## SSML & Synthesis Details (from TTS heritage)

HD parameters go on the `parameters` attribute of `<voice>`. Always use `dnx Microsoft.CognitiveServices.Speech.CLI -- ...` form.

Default podcast MP3 encoding: `audio-48khz-192kbitrate-mono-mp3` (see §4.5). Do not use bare `--format mp3`.

Full escaping, language mapping, and command shape are described in the steps above.

---

## Feed Item & Podcasting 2.0

Follow the item structure from the reference `C:\Code\podcast\feed.xml`.

Always insert newest item first.

Add the `podcast:` namespace when using `<podcast:transcript>`.

---

## Troubleshooting

| Symptom | Action |
|---------|--------|
| `xurl version` missing "by @kzu" | Install/replace with `go install github.com/kzu/xurl@v0.1.0` |
| No xurl auth | User runs `xurl auth ...` outside session; prefer app |
| dnx not found / `dnx --version` fails | Use `dotnet --version`; dnx is invoked as `dnx <Package>` (it is `dotnet dnx` under the hood). Confirm `C:\Program Files\dotnet\dnx.cmd` exists. |
| GStreamer error (`SPXERR_GSTREAMER_NOT_FOUND_ERROR`) on recognize | Do **not** run recognize directly on .mp3. Always convert to 16 kHz mono WAV with ffmpeg first. |
| Only first few words in transcript | Use `--continuous` instead of `--once` for long-form posts. |
| dnx help or output piped with `| cat` / `| head` produces repeated "Get-Content cannot be bound" spam | Always redirect first: `... 2>&1 | Out-File $tmp`; then `Get-Content $tmp`. The harness is sensitive to certain pipe patterns with verbose CLIs. |
| Duration formatting error in PowerShell (`Format specifier was invalid`) | Use `[TimeSpan]::FromSeconds($dur).ToString("m\:ss")` instead of custom `{0}:{1:D2}` -f patterns. |
| xurl JSON looks garbled (├⌐ etc.) in terminal | Save to file (`> $json`), then `Get-Content $json -Raw | ConvertFrom-Json`. Never trust raw console for accented text. |
| Voice not found / flat output | Verify full voice name from voices.toml includes `:DragonHDLatestNeural` / `:DragonHDOmniLatestNeural`; parameters on `<voice>` |
| MP3 sounds muffled / low fidelity | Confirm `--format audio-48khz-192kbitrate-mono-mp3`, not bare `--format mp3` (16 kHz / 128 kbps default). Verify with `ffprobe` (expect `sample_rate=48000`, `bit_rate=192000`). |
| Wrong episode number | Check that feed parsing looks only at items with matching `itunes:season` year. When the feed has no `<item>` yet, N=1. |
| Transcript tag ignored | Ensure namespace on `<rss>` and correct url/type |
| index.html stale | Re-run the language pass or manually re-scan dirs after adding episodes |
| Secret leakage risk | Never pass --bearer* etc.; never cat ~/.xurl |
| `ls -la`, `cat`, Unix pipes failing | This is PowerShell on Windows (pwsh). Prefer `Get-ChildItem`, `Get-Content`, `Out-File`, `Select-String`. |

## Agent freedom

This skill specifies **outcomes**. Implement with pwsh, Python, direct string/ DOM XML edits, etc., as long as:
- Local files match the exact naming, locations, and content rules (including <N>-<slug>.jpg next to the other three files for every language)
- feed.xml remains valid RSS with correct namespaces and newest-first ordering, and includes <itunes:image> in each new item
- .srt is produced reliably: MP3 → WAV (ffmpeg) → recognize --continuous (log capture) → parsed RECOGNIZED lines + timing from ffprobe duration
- No upload or git side-effects
- The same visual artwork is used for all languages (copied by the lang-specific slug name)

The existing `en/feed.xml` / `es/feed.xml` and `C:\Code\podcast\feed.xml` are the shape references.

---

## Related

- Separate `upload` skill will handle azcopy (now including the per-language .jpg artwork) + "Add episode [#] - [title]" commits + push.
- Old `tts` behavior is fully subsumed here (synthesis details preserved and extended).