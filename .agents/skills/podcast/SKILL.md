---
name: podcast
description: Receive an X post URL, retrieve full content and post date, then for each supported podcast language: create lang/yyyy/MM/ folder (yyyy/MM = current date), translate if needed, generate short title + summary + keywords (if no post cover image has been determined yet, also request an English "imagine" prompt from the LLM), determine episode artwork (prefer X article header photo from data.article.cover_media if present — download + center-crop to square via ffmpeg; else use the imagine prompt to launch `/imagine` + 1:1), copy the artwork as #-slug.jpg alongside the mp3 for *every* language, write #-slug.md (YAML frontmatter with title/summary/date=current/link/author-name-only + spoken content with localized byline using the X post date, e.g. en `April 3rd, 2026` / es `3 de Abril de 2026`), synthesize #-slug.mp3 in ~2200-char sentence-boundary chunks (Azure 10-minute-per-call limit) then ffmpeg-concat the parts, embed ID3v2.3 tags (title/artist/album/comment/track/date/genre) via ffmpeg -c copy, using male/female voice chosen according to post author (via voices.toml + Azure Dragon HD/Omni), update the language feed.xml (newest-first item with enclosure + itunes:image; item pubDate = current date), and maintain lang/index.html (episode table + local audio players). Local files only — upload (including the jpgs), git commit ("Add episode [#] - [title]"), and push are handled by a separate upload skill. Uses xurl, dnx Microsoft.CognitiveServices.Speech.CLI, az CLI, ffmpeg, ffprobe.
---

# Podcast — X Post to Multi-Language Local Episode Generator

Turn an X post into ready-to-publish podcast episodes for every supported language. The skill produces local files and updates local feed/index artifacts only. A separate `upload` skill performs Azure blob upload (azcopy), git commit with message "Add episode [#] - [title]", and push.

## What success looks like

When finished, all of the following must be true:

1. The X post was fetched via `xurl read` and its original (attributed) content archived to `posts/<yyyy-MM-dd>-<post-id>.md` with YAML frontmatter (`author` = display name only; body retains `By Name (@user)` attribution line) + raw text.
2. For every supported podcast language (en, es from voices.toml):
   - Directory structure `<lang>/<yyyy>/<MM>/` exists for the **episode date** (today's date when generating).
   - `<N>-<slug>.md` exists with YAML frontmatter (`title`, `summary`, `date`, `link`, `author` = display name only, no `@handle`) followed by the spoken post content in this exact structure (plain text, no markdown):
     ```
     <title>
     by <author>, posted on <spoken-date>

     <content>
     ```
     (`<spoken-date>` is localized long form — **not** `yyyy-MM-dd`; see §4.4.)
   - `<N>-<slug>.mp3` was synthesized using the gender-appropriate voice (inferred from post author name, default male) resolved from the effective `voices.<lang>.<gender>` section, encoded as **48 kHz mono MP3 at 192 kbps** (`audio-48khz-192kbitrate-mono-mp3`). Long episodes are split into sentence-boundary chunks of **≤ ~2200 characters** (Azure hard-caps each synthesis call at ~10 minutes of audio), synthesized per chunk, then merged with `ffmpeg -f concat -c copy`. After merge (or direct copy for single-chunk episodes), **ID3v2.3 tags are embedded** with `ffmpeg -c copy -id3v2_version 3` (Azure output has no ID3 header — feed validators require it). A single-call output of exactly `10:00` / `600.000000` seconds on long text is a truncation failure — re-chunk and re-synthesize.
   - Artwork is determined once (if a post cover image hasn't been determined *yet*, the LLM is asked for an English "imagine" prompt which is used to generate an initial one; the same artwork is then copied for all languages): if the fetched post JSON has `data.article.cover_media`, the corresponding photo URL from `includes.media` is downloaded and center-cropped to square (1024x1024) via ffmpeg and saved as `<N>-<slug>.jpg`; otherwise an "imagine" prompt from the LLM JSON is used to launch `/imagine <prompt>, 1:1` and the result is saved the same way. The jpg (using each language's own slug) sits next to the mp3 in *every* language directory. Every language's feed item includes the corresponding `<itunes:image>`.
   - `<lang>/feed.xml` contains the new episode as the **first** `<item>` under `<channel>`, with:
     - Correct `<enclosure>` using `https://{{storage}}.blob.core.windows.net/{{container}}/<lang>/<yyyy>/<MM>/<N>-<slug>.mp3`
     - `itunes:season` = episode year (today), `itunes:episode` = N (per-language TV-series numbering)
     - Accurate `itunes:duration`, file length, MIME `audio/mpeg`
     - `<itunes:image>` (for the per-language copy of the episode artwork jpg)
   - `<lang>/index.html` exists or is updated with a simple table of episodes (newest/highest N first) including HTML5 `<audio controls>` players for local testing.
3. GUIDs are always `yyyy-MM-dd-N-slug` (date prefix = **`$episodeDate`**, not `$postDate`).
4. Channel `lastBuildDate` is set to current GMT; per-item `pubDate` uses the **episode date** (today — not the X post's `created_at`), so feed ordering matches episode number (higher N = newer pubDate).
5. No Azure upload, no git operations, and no remote changes occurred. All artifacts are local and ready for the upload skill.
6. The user is given episode numbers, titles, file paths, and the future blob URLs for each language.

## Architecture & Layout

**Episode file naming:** `<N>-<slug>.md`, `<N>-<slug>.mp3`, `<N>-<slug>.jpg` (N = sequential episode within year for that language feed). The .jpg (episode artwork) uses the *same* visual for every language but is named using that language's title-derived slug so it sits next to its mp3.

**Directory layout per language:**
```
<lang>/
  feed.xml
  index.html
  logo.jpg
  cover.png
  <yyyy>/
    <MM>/
      <N>-<slug>.md
      <N>-<slug>.mp3
      <N>-<slug>.jpg
```

**Original archive (never translated):**
```
posts/
  <yyyy-MM-dd>-<post-id>.md     # YAML + raw attributed source text
```

**Enclosure + artwork URLs (written into feed):**  
`https://<storage>.blob.core.windows.net/<container>/<lang>/<yyyy>/<MM>/<N>-<slug>.mp3`  
(and matching `.jpg` for `<itunes:image>`).

**Episode numbering (TV-series model):**  
`itunes:season` = calendar year of the **episode date** (current date when generating)  
`itunes:episode` = next sequential number for that year **per language feed**.

**Two dates — do not conflate:**
- **Episode date** = today (when the episode is generated). Drives folder paths (`<lang>/<yyyy>/<MM>/`), YAML `date`, GUID, feed `pubDate`, `itunes:season` year, and episode numbering.
- **Post date** = X `created_at`. Drives the `posts/` archive filename, archive YAML `date`, and the spoken byline (`posted on …` / `publicado el …`) only.

**Reference feed:** Use patterns from `C:\Code\podcast\feed.xml` and the existing `en/feed.xml` / `es/feed.xml` for item shape, namespaces, and tone.

## Prerequisites

| Tool | Purpose | Verify | Install (Windows) |
|------|---------|--------|-------------------|
| xurl (required fork) | Fetch full X post content (note_tweet / article / text) | `xurl version` ends with "by @kzu" | `go install github.com/kzu/xurl@v0.1.0` |
| dnx / .NET 10 SDK | Run Azure Speech CLI (synthesize) | `dotnet --version` (≥ 10) and presence of `dnx` (e.g. `Get-Command dnx` or `C:\Program Files\dotnet\dnx.cmd`) | Install .NET 10.0 SDK (dnx is included as `dotnet dnx`) |
| az CLI | Fetch Azure Speech key + region from voices.toml `[azure]` | `az account show` | `winget install Microsoft.AzureCLI` |
| ffmpeg + ffprobe | Artwork crop, MP3 concat, probe duration/size | `ffmpeg -version` and `ffprobe -version` | `winget install ffmpeg` (includes ffprobe) |
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
4. `ffmpeg -version` (required for artwork crop and MP3 concat).
5. Run `xurl read <url>` (never with inline secrets).

If no usable auth: tell the user to run `xurl auth oauth2` (or appropriate) **outside** the agent session.

## Robust execution patterns (this environment)

- **Capture noisy / long output first**: For `dnx ...`, `az ...`, and help commands, use `... 2>&1 | Out-File $tmp` (or `> $tmp`), then inspect with `Get-Content $tmp`. Direct `| cat`, `| head`, `| Select` on dnx often triggers harness "Get-Content cannot bind" spam.
- **Prefer PowerShell cmdlets**: `Get-ChildItem` / `Get-Content -Raw` / `Out-File -Encoding utf8` / `ConvertFrom-Json` / `Select-String` over `ls`, `cat`, Unix pipes for reliability on Windows pwsh.
- **Run synthesis in the current pwsh session**: Define helper functions inline and call them directly. Avoid spawning a nested `powershell -File` for TTS — it adds failure modes and makes chunk-array bugs harder to spot.
- **UTF-8**: When writing SSML or .md files use `-Encoding utf8` (or `[System.IO.File]::WriteAllText` with UTF-8 no BOM for SSML). When reading xurl JSON use `-Raw | ConvertFrom-Json`.
- **Duration**: Use `[TimeSpan]::FromSeconds([double]$raw).ToString("m\:ss")`.
- **Temporary files**: Use `$env:TEMP\...` for ssml, logs, and json. Clean up in §4.8 unless debugging.
- **Long synthesis**: Expect many "SYNTHESIZING: audio.length=..." messages — redirect or tail as needed. Always chunk long text (see §4.5); never feed the full `.md` body in one SSML call when it exceeds ~2200 chars.
- **Chunked TTS temp files**: Keep per-chunk `.ssml` / `part-*.mp3` / `concat.txt` under `$env:TEMP` until merge succeeds; delete in §4.8.
- **Verify before synthesizing**: Log each chunk's character count. If a chunk reports `1` char (or audio is ~0:00) on a multi-paragraph article, the PowerShell single-element array unwrap bug fired — see §4.5 and Troubleshooting.

## Re-rendering an existing episode MP3

When the user asks to re-render/regen MP3s for existing `.md` files (no new X fetch):

1. Read each `<N>-<slug>.md`; extract spoken body (everything after the closing `---` frontmatter delimiter).
2. Resolve voice from `voices.toml` for that language + author gender (infer from YAML `author` or prior episode).
3. Run the §4.5 chunk → synthesize → concat pipeline into the existing `<N>-<slug>.mp3` path.
4. Re-apply §4.6 ID3 tags from the episode `.md` frontmatter + feed metadata.
5. Re-probe duration/size; update `itunes:duration` and enclosure `length` in feed if they changed.
6. Skip translation, artwork, archive, and feed prepend — only replace the audio (and feed duration/length if needed).

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
    - [ ] Compute yyyy/MM from **episode date** (today), slug (kebab), next N for (lang, episode-year) from feed
    - [ ] Write <lang>/<yyyy>/<MM>/<N>-<slug>.md (yaml `date` = episode date yyyy-MM-dd; spoken body: title + localized byline with long-form **post date**, e.g. en `by <name>, posted on April 3rd, 2026` / es `por <name>, publicado el 3 de Abril de 2026`, then content)
    - [ ] If no artwork determined yet: if article cover photo URL was resolved earlier, download it and center-crop to square 1024x1024 via ffmpeg as <N>-<slug>.jpg (skip /imagine); else (if imagine in this JSON) start subagent `/imagine <imagine>, 1:1 aspect ratio`; copy result (or cover-derived jpg) as <N>-<slug>.jpg into *this* language's dir (and later for other langs too)
    - [ ] Resolve voices.<lang>.<gender> + chunk spoken body (≤ ~2200 chars, sentence boundaries) + synthesize each chunk via dnx (`--format audio-48khz-192kbitrate-mono-mp3`) + ffmpeg-concat into final .mp3; verify per-chunk char counts and total duration is not exactly 10:00 on long posts
    - [ ] Embed ID3v2.3 tags (§4.6) into final .mp3; verify file starts with `ID3` magic bytes
    - [ ] Probe duration/size with ffprobe (after tagging — file size includes ID3 header)
    - [ ] Update <lang>/feed.xml (prepend item + itunes:image + current lastBuildDate)
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

**Author:** From `includes.users` matching `data.author_id`:
- `$authorName` — display `name` if present, else `username` (this is the YAML `author` value and the spoken byline name; **no `@handle`** — the handle is already in `link`).
- `$authorLine` — for archive/translation attribution only: `By {name} (@{username})` or `By @{username}` when name is missing.

Also capture the **post date** (`$postDate` = `data.created_at` as `yyyy-MM-dd`) and keep the main post body.

Set the **episode date** (`$episodeDate`) to today's date in `yyyy-MM-dd` (authoritative: the `Today's date:` field in the user/session context). Episode and post dates are independent — an old X post published as a podcast episode today uses today's date for folders/GUID/pubDate and the original `created_at` for the spoken byline.

**Attributed source text (for archive + translation + title generation input):** prepend `$authorLine` + blank line + body.

**Spoken/read-aloud content (for episode .md body and TTS):** constructed *after* title generation (see 4.4).

**Dates & link:**
- `$postDate` (`data.created_at`) → `posts/` archive only + spoken byline `<spoken-date>`.
- `$episodeDate` (today) → `<lang>/<yyyy>/<MM>/` folders, episode YAML `date`, GUID, feed `pubDate`, `itunes:season` year, episode numbering.
- Canonical link: the input URL or construct `https://x.com/{username}/status/{id}`.

**Outcome:** Attributed full source text (for archive/translate) + `$postDate` + `$episodeDate` + link + `$authorName`. The spoken content structure is applied later using the generated title + localized byline from `$postDate` (`by $authorName, posted on <spoken-date>` in en; `por $authorName, publicado el <spoken-date>` in es) + main content.

---

## Step 2 — Archive the original post

**Goal:** Persist the exact source (never translated) for future reference.

- Create `posts/` directory if missing.
- Write `posts/<yyyy-MM-dd>-<post-id>.md` using **`$postDate`** (X `created_at`, not the episode date):
  ```yaml
  ---
  id: "<post-id>"
  link: "https://x.com/..."
  author: "Name"
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
- `yyyy`, `MM` (zero-padded) from **`$episodeDate`** (today — not `$postDate`).
- Slug: kebab-case, lowercase, ascii, safe for filenames/URLs (derived from title). Sanitize: remove diacritics, replace non-alphanum with `-`, collapse multiple dashes, trim. Example: "SpaceX: Not an IPO — It's a Referendum" → `spacex-not-an-ipo-its-a-referendum`.
- Next `N`:
  - Load `<lang>/feed.xml`
  - Parse `<itunes:episode>` + `<itunes:season>` (or default to 0). Only consider items whose season matches the **episode** year (`$episodeDate`).
  - `N = (max for this year) + 1` (or 1 if none)
  - Practical pattern (simple feeds): load as text or `[xml]`, look for `<itunes:episode>` / `<itunes:season>` (or `episode`/`season` after casting). When no items exist for the year, use 1. Fall back to directory scan of `<lang>/<yyyy>/` subfolders + filenames if feed parsing is painful.
- Target directory: `<lang>/<yyyy>/<MM>/`
- Files: `<N>-<slug>.md`, `<N>-<slug>.mp3`, `<N>-<slug>.jpg`

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
date: "2026-06-22"
link: "https://x.com/username/status/..."
author: "Name"
---
<title>
by Name, posted on <spoken-date>

<full translated post content — plain text only, no **bold**, no markdown lists, no formatting>
```

(`date` in frontmatter is **`$episodeDate`** — today. `<spoken-date>` comes from **`$postDate`** — when the X post was written.)

**Two date formats — do not confuse them:**
- **YAML `date`** (frontmatter, folders, GUID, feed `pubDate`): ISO `yyyy-MM-dd` from **`$episodeDate`** (e.g. `2026-06-22`).
- **Spoken byline date** (`<spoken-date>`): localized long form for TTS, derived from **`$postDate`** (X `created_at`):

| Lang | Byline template | `<spoken-date>` example |
|------|-----------------|-------------------------|
| `en` | `by $authorName, posted on <spoken-date>` | `April 3rd, 2026` |
| `es` | `por $authorName, publicado el <spoken-date>` | `3 de Abril de 2026` |

**English ordinals:** day + correct suffix (`1st`, `2nd`, `3rd`, `4th`, `21st`, `22nd`, `23rd`, …). Month name in full (`January` … `December`).

**Spanish:** day + `de` + month (capitalized: `Abril`, `Junio`, …) + `de` + year. No ordinal suffix on the day.

Example bodies:
```
Hope: The Future Is Bright
by Brivael Le Pogam, posted on April 3rd, 2026

The kid in front of the screen
...
```

```
Esperanza: El futuro es radiante
por Brivael Le Pogam, publicado el 3 de Abril de 2026

El chico frente a la pantalla
...
```

The YAML `author` field and the spoken byline use **`$authorName` only** (display name, no `@handle`). The X handle is already implied by `link`.

The body after the frontmatter is the complete text that will be spoken (starting with the generated title, the localized byline with `<spoken-date>`, then the content). Use the generated `title` for the first line and `$authorName` for the byline. The main content is the translated substantive post text (without the archive-style `By … (@…)` prefix).

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

Escape only `& < >`. Write each chunk to its own temp `.ssml` file.

**Audio output format (podcast default):** Use the highest-quality MP3 preset, not bare `--format mp3` (which defaults to 16 kHz / 128 kbps):

```
audio-48khz-192kbitrate-mono-mp3
```

(48 kHz sample rate, 192 kbps mono — best MP3 option in `spx help synthesize mp3`.)

#### Azure 10-minute synthesis limit (mandatory chunking)

Each `dnx ... synthesize` call has a **hard ~10-minute audio cap** (~600.000000 s, ~14.4 MB at 192 kbps). Feeding the full episode body in one SSML call silently truncates the tail — `ffprobe` reports exactly `10:00` even when thousands of characters remain unspoken.

**Always chunk** the spoken `.md` body (everything after the frontmatter) before synthesis:

1. **Split at sentence boundaries**, not paragraph/double-newline boundaries. X articles often use single newlines between section headers and body text; paragraph-based splitting leaves one giant chunk that still hits the 10-minute wall.
2. **Target ≤ ~2200 characters per chunk** (including spaces). Accumulate sentences until the next sentence would exceed the limit, then start a new chunk.
3. **Synthesize each chunk** to a temp `part-<i>.mp3` (same voice, params, format).
4. **Merge** with ffmpeg stream copy (no re-encode):
   ```pwsh
   # concat.txt — one line per part, forward slashes in paths
   file 'C:/Users/.../TEMP/synth-.../part-0.mp3'
   file 'C:/Users/.../TEMP/synth-.../part-1.mp3'
   ffmpeg -y -f concat -safe 0 -i concat.txt -c copy "<lang>/<yyyy>/<MM>/<N>-<slug>.mp3"
   ```
5. **Verify**: log per-chunk **character count** and duration before moving on. For bodies > ~4000 chars, expect multiple chunks and total duration **> 10:00**. Exactly `10:00` on long text means truncation — reduce chunk size and re-run.

#### PowerShell single-chunk array unwrap (critical)

When a function `return $chunks` and `$chunks` is a one-element list, **PowerShell unwraps it to a plain string**. The synthesis loop then treats that string as a scalar: `$chunks.Count` is `1`, but `$chunks[0]` is the **first character only** — producing ~0:00 audio (or a nonsense clip) despite the body being thousands of characters. Multi-chunk episodes (5–6 parts) are unaffected; short single-chunk episodes hit this every time.

**Always prevent unwrapping** in both the function return and the call site:

```pwsh
function Split-TextChunks([string]$text, [int]$maxChars = 2200) {
    $sentences = [regex]::Split($text, '(?<=[.!?])\s+')
    $chunks = [System.Collections.Generic.List[string]]::new()
    $current = [System.Text.StringBuilder]::new()
    foreach ($s in $sentences) {
        if ([string]::IsNullOrWhiteSpace($s)) { continue }
        $candidate = if ($current.Length -eq 0) { $s } else { $current.ToString() + ' ' + $s }
        if ($candidate.Length -gt $maxChars -and $current.Length -gt 0) {
            $chunks.Add($current.ToString().Trim())
            $current.Clear() | Out-Null; $current.Append($s) | Out-Null
        } else {
            $current.Clear() | Out-Null; $current.Append($candidate) | Out-Null
        }
    }
    if ($current.Length -gt 0) { $chunks.Add($current.ToString().Trim()) }
    return ,@($chunks.ToArray())   # unary comma — never return $chunks bare
}

# Call site — keep as array even for one chunk:
$chunks = @(Split-TextChunks $body)
for ($i = 0; $i -lt $chunks.Count; $i++) {
    # $chunks[$i] is the full chunk string
}
```

Short posts (single chunk under the limit) copy `part-0.mp3` directly to the final `.mp3` — no concat step needed.

**Per-chunk synthesize** (exact form required by AGENTS.md):
```pwsh
$format = "audio-48khz-192kbitrate-mono-mp3"
dnx Microsoft.CognitiveServices.Speech.CLI -- synthesize --key $key --region $region --file $ssml --audio output $partMp3 --format $format 2>&1 | Out-File "$env:TEMP\synth-chunk-$i.log"
```

Synthesis produces many "SYNTHESIZING: ..." progress lines. Redirect to a log file (or use `| Select -Last 5`) for cleaner sessions.

**Never** use plain `--text --voice` for HD voices — parameters must be on the `<voice>` element.

### 4.6 Embed ID3v2 tags and probe metadata

Azure Speech outputs bare MPEG frames with **no ID3v2 header**. Podcast feed validators (and many players) require ID3 tags; without them you get errors like `No ID3v2 headers found` and `Could not detect bitrate mode (no ID3 tags)`.

**Immediately after** concat or single-chunk copy — and **before** probing size for the feed enclosure — write ID3v2.3 tags with ffmpeg stream copy (no re-encode):

```pwsh
function Add-EpisodeId3Tags {
    param(
        [string]$Mp3,
        [string]$Title,
        [string]$Author,      # YAML author; strip leading "By " for TPE1 if present
        [string]$Album,       # [podcast.<lang>].title
        [string]$AlbumArtist, # [podcast].author
        [string]$Summary,
        [int]$Episode,        # N from filename / itunes:episode
        [string]$Year         # yyyy from episode date
    )
    $artist = $Author -replace '^By\s+', ''
    $tmp = Join-Path $env:TEMP ("id3-" + [guid]::NewGuid().ToString() + ".mp3")
    ffmpeg -y -i $Mp3 -c copy -id3v2_version 3 `
        -metadata "title=$Title" `
        -metadata "artist=$artist" `
        -metadata "album=$Album" `
        -metadata "album_artist=$AlbumArtist" `
        -metadata "date=$Year" `
        -metadata "comment=$Summary" `
        -metadata "genre=Podcast" `
        -metadata "track=$Episode" `
        $tmp 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) { throw "ffmpeg ID3 tagging failed for $Mp3" }
    Move-Item -Force $tmp $Mp3
}
```

| ID3 frame | ffmpeg `-metadata` | Source |
|-----------|-------------------|--------|
| TIT2 | `title` | generated episode title |
| TPE1 | `artist` | YAML `author` (name only; strip `By ` prefix) |
| TALB | `album` | `[podcast.<lang>].title` |
| TPE2 | `album_artist` | `[podcast].author` |
| TYER/TDRC | `date` | episode year (`yyyy` from YAML `date`) |
| COMM | `comment` | generated `summary` |
| TRCK | `track` | episode number `N` |
| TCON | `genre` | `Podcast` |

**Verify** tagging succeeded before probing enclosure length:

```pwsh
$magic = [System.Text.Encoding]::ASCII.GetString([System.IO.File]::ReadAllBytes($mp3)[0..2])
if ($magic -ne 'ID3') { throw "Missing ID3v2 header on $mp3" }
ffprobe -v quiet -show_entries format_tags $mp3   # optional sanity check
```

Tagging adds a small ID3 header (~0.5–1.5 KB); always probe file size **after** tagging for the enclosure `length` attribute.

Probe the MP3 (robust PowerShell):
```pwsh
$dur = [double](ffprobe -v quiet -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 $mp3)
$ts = [TimeSpan]::FromSeconds($dur)
$duration = $ts.ToString("m\:ss")     # e.g. "2:35"  (avoid :D2 format specifier pitfalls)
$size = (Get-Item $mp3).Length
```

Use `$duration` for `itunes:duration` and `$size` for the enclosure `length` attribute.

### 4.7 Update <lang>/feed.xml

Load the existing `<lang>/feed.xml`.

- Ensure root has `xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd"`

Prepend a new `<item>` as the **first** child of `<channel>`.

Item fields (match reference style):
- `<title>`, `<description>` (use generated summary), `<guid>` (exactly `yyyy-MM-dd-N-slug` using **`$episodeDate`**), `<link>` (original X URL), `<pubDate>` (RFC 822 from **`$episodeDate`** — today, not `$postDate`)
- `<enclosure url="https://<storage>.blob.../<lang>/.../<N>-<slug>.mp3" length="..." type="audio/mpeg" />`
- `<itunes:season>` (episode year from `$episodeDate`), `<itunes:episode>`, `<itunes:duration>`, `<itunes:summary>`, `<itunes:keywords>`, `<itunes:explicit>No</itunes:explicit>`
- `<itunes:image href="https://<storage>.blob.../<lang>/.../<N>-<slug>.jpg" />` (present for every language because artwork was copied using the language's slug)

Update channel `<lastBuildDate>` to the current GMT time.

Do **not** change the per-item pubDates.

Do **not** add `<podcast:transcript>` tags — Spotify does not consume them for this feed.

### 4.8 Update <lang>/index.html

Create or overwrite `<lang>/index.html` with a minimal, self-contained page for local testing:

- Header row with `<lang>/logo.jpg` at the top-left, to the left of the podcast title and description from `[podcast.<lang>]` in `voices.toml`.
- Table (newest first): Episode | Title | `<audio controls src="relative/path/to/<N>-<slug>.mp3">`
- Scan the year/month subdirs or parse the just-updated feed.xml to keep the list accurate.
- Use this header markup (title/description from config; logo path is always `logo.jpg` relative to `<lang>/index.html`):

```html
<header class="site-header">
  <img src="logo.jpg" alt="" class="logo">
  <div>
    <h1><!-- podcast title --></h1>
    <p><!-- podcast description --></p>
  </div>
</header>
```

- Use this CSS so the native `<audio controls>` player stays full-size (reserve the Audio column width; `width: 100%` alone collapses the control in a narrow table cell):

```css
.site-header { display: flex; align-items: center; gap: 1rem; margin-bottom: 1.5rem; }
.site-header .logo { width: 64px; height: 64px; flex-shrink: 0; }
.site-header h1 { margin: 0; }
.site-header p { margin: 0.25rem 0 0; }
table { border-collapse: collapse; width: 100%; max-width: 900px; table-layout: fixed; }
th, td { padding: 0.5rem; border-bottom: 1px solid #333; text-align: left; vertical-align: middle; }
th:first-child, td:first-child { width: 3rem; }
th:last-child, td:last-child { width: 360px; }
audio { display: block; width: 100%; min-height: 40px; }
```

This file is never uploaded to the podcast feed; it is for human convenience when testing locally.

### 4.9 Cleanup
Delete temporary SSML, per-chunk `part-*.mp3`, `concat.txt`, synth chunk logs, JSON, downloaded cover photos (`$env:TEMP\cover-*.jpg`), and any intermediate image files from the subagent unless the user wants them kept for debugging. Keep the final merged .mp3 + .md + .jpg (the .jpg must remain next to the mp3 for every language).

---

## Step 5 — Report to the user

For each language report:
- Episode number N and GUID
- Title + summary + keywords
- Paths to .md, .mp3 (and .jpg for the artwork if generated)
- Path to updated feed.xml and index.html
- Future blob enclosure URL for mp3 (and jpg image URL)
- Author gender / voice chosen
- The imagine prompt that was used (if any, always English) and the artwork source: "X article cover photo (center-cropped to square)" or "generated via /imagine" (or none)

Confirm that only local files were created/modified and nothing was uploaded or committed.

---

## Examples

**Basic usage:**
"podcast https://x.com/someone/status/1234567890123456789"

Agent:
1. Fetches post with xurl, archives original to `posts/2026-06-18-1234567890123456789.md` (archive date = X post date; episode folder/pubDate would use today, e.g. `2026/06/` if generating on June 22nd)
2. Infers author gender (e.g. male)
3. For the first language processed: generates title/summary/keywords; if no cover photo has been determined yet, also requests an English "imagine" prompt from the LLM. If the xurl JSON had `data.article.cover_media`, downloads + ffmpeg center-crops it to square as the jpg (no subagent); otherwise uses the imagine to start `/imagine <imagine>, 1:1` and saves the result. Writes the jpg using that language's slug, writes the .md (`author` = name only; spoken byline uses long-form localized date per §4.4), chunks + synthesizes + ffmpeg-merges .mp3, updates its feed (incl. itunes:image) + index.html.
4. For subsequent languages: translate as needed, generate title/summary/keywords (3-field, no imagine since artwork already determined), copy the same artwork jpg using the language-specific slug next to the mp3, write .md + chunked .mp3 + .jpg, update feed + index.html.
5. Reports everything (including per-lang jpg paths + image blob URLs); all local.

**Re-render existing episode:**
"re-render en/2026/06/1-spacex-not-an-ipo-its-a-referendum.md"

Agent: extracts spoken body from the .md, runs §4.5 with `return ,@($chunks.ToArray())`, verifies chunk char counts (expect ~2000 for a short article, not `1`), writes the .mp3 in place, re-applies §4.6 ID3 tags, re-probes duration/size.

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

**Long-form posts:** always chunk at ~2200 chars (sentence boundaries) before building SSML, synthesize per chunk, ffmpeg-concat (see §4.5). Never assume one SSML file covers the full article.

**Short single-chunk posts:** always use the array-unwrap-safe return/call pattern (see §4.5). This is the most common re-render failure mode.

Full escaping, language mapping, chunking, merge, and command shape are described in the steps above.

---

## Feed Item shape

Follow the item structure from the reference `C:\Code\podcast\feed.xml`.

Always insert newest item first.

---

## Troubleshooting

| Symptom | Action |
|---------|--------|
| `xurl version` missing "by @kzu" | Install/replace with `go install github.com/kzu/xurl@v0.1.0` |
| No xurl auth | User runs `xurl auth ...` outside session; prefer app |
| dnx not found / `dnx --version` fails | Use `dotnet --version`; dnx is invoked as `dnx <Package>` (it is `dotnet dnx` under the hood). Confirm `C:\Program Files\dotnet\dnx.cmd` exists. |
| dnx help or output piped with `| cat` / `| head` produces repeated "Get-Content cannot be bound" spam | Always redirect first: `... 2>&1 | Out-File $tmp`; then `Get-Content $tmp`. The harness is sensitive to certain pipe patterns with verbose CLIs. |
| Duration formatting error in PowerShell (`Format specifier was invalid`) | Use `[TimeSpan]::FromSeconds($dur).ToString("m\:ss")` instead of custom `{0}:{1:D2}` -f patterns. |
| xurl JSON looks garbled (├⌐ etc.) in terminal | Save to file (`> $json`), then `Get-Content $json -Raw | ConvertFrom-Json`. Never trust raw console for accented text. |
| Voice not found / flat output | Verify full voice name from voices.toml includes `:DragonHDLatestNeural` / `:DragonHDOmniLatestNeural`; parameters on `<voice>` |
| MP3 sounds muffled / low fidelity | Confirm `--format audio-48khz-192kbitrate-mono-mp3`, not bare `--format mp3` (16 kHz / 128 kbps default). Verify with `ffprobe` (expect `sample_rate=48000`, `bit_rate=192000`). |
| MP3 duration exactly `10:00` / `600.000000` s on a long article | Azure truncated the synthesis. Re-chunk at sentence boundaries (≤ ~2200 chars); do **not** rely on paragraph splits (X articles use single newlines). Merge parts with `ffmpeg -f concat -c copy`. |
| One chunk has >9000 chars, others tiny | Paragraph-based splitting failed — switch to sentence-boundary splitting (see §4.5). |
| Log shows `chunk 0 : 1 chars` or ~0:00 audio on a full article | PowerShell single-element array unwrap (see §4.5). Fix `return ,@($chunks.ToArray())` and `$chunks = @(Split-TextChunks $body)`; re-run. |
| ffmpeg concat fails | Use `concat.txt` with `file 'path/with/forward/slashes'` and `-safe 0`. All parts must share the same codec/format (identical `--format` on every synthesize call). |
| Feed validation: `No ID3v2 headers found` / `Could not detect bitrate mode` | Azure output lacks ID3. Run §4.6 `Add-EpisodeId3Tags` with `ffmpeg -c copy -id3v2_version 3`, then re-probe size and update enclosure `length`. |
| Wrong episode number | Check that feed parsing looks only at items with matching `itunes:season` year. When the feed has no `<item>` yet, N=1. |
| index.html stale | Re-run the language pass or manually re-scan dirs after adding episodes |
| Secret leakage risk | Never pass --bearer* etc.; never cat ~/.xurl |
| `ls -la`, `cat`, Unix pipes failing | This is PowerShell on Windows (pwsh). Prefer `Get-ChildItem`, `Get-Content`, `Out-File`, `Select-String`. |

## Agent freedom

This skill specifies **outcomes**. Implement with pwsh, Python, direct string/ DOM XML edits, etc., as long as:
- Local files match the exact naming, locations, and content rules (including <N>-<slug>.jpg next to the mp3 for every language)
- feed.xml remains valid RSS with correct namespaces and newest-first ordering, and includes <itunes:image> in each new item
- .mp3 is produced via sentence-boundary chunking (≤ ~2200 chars) + per-chunk synthesis + ffmpeg concat when needed; ID3v2.3 tags embedded via §4.6; duration verified not truncated at exactly 10:00; single-chunk returns use the array-unwrap-safe pattern
- No upload or git side-effects
- The same visual artwork is used for all languages (copied by the lang-specific slug name)
- No .srt files or `<podcast:transcript>` tags are generated

The existing `en/feed.xml` / `es/feed.xml` and `C:\Code\podcast\feed.xml` are the shape references.

---

## Related

- Separate `upload` skill will handle azcopy (now including the per-language .jpg artwork) + "Add episode [#] - [title]" commits + push.
- Old `tts` behavior is fully subsumed here (synthesis details preserved and extended).