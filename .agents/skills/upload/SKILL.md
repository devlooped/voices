---
name: upload
description: After the podcast skill has prepared local episode artifacts (posts/ archive, lang/yyyy/MM/#-slug.{md,mp3,srt,jpg}, updated lang/feed.xml + lang/index.html with itunes:image), upload the generated files to Azure Blob Storage using azcopy (to the paths referenced by the feed enclosures, itunes:image, and transcript tags) and commit + push to git with message "Add episode [#] - [title]" (summary in body). Resolves storage/container from voices.toml [azure]. Use when publishing a newly generated episode from the podcast skill.
---

# Upload — Azure Storage + Git Commit & Push for Podcast Episodes

This skill takes the local output of the `podcast` skill (new episode files including artwork jpgs + updated per-language feeds and indexes) and publishes it:

- Uploads the assets to Azure Blob Storage so the podcast RSS (with correct enclosures, itunes:image artwork, and Podcasting 2.0 transcripts) becomes live.
- Creates a git commit and pushes the source changes (markdowns, feeds, indexes, jpg artwork, original post archive).

The `podcast` skill only writes locally. This skill performs the actual publication to blob + git.

## What success looks like

When finished, all of the following must be true:

1. The new episode files are publicly reachable on Azure:
   - `https://<storage>.blob.core.windows.net/<container>/<lang>/<yyyy>/<MM>/<N>-<slug>.mp3`
   - Matching `.srt` for transcripts
   - Matching `.jpg` for itunes:image artwork (one per language, using that language's slug)
   - `.md` files (for indexes)
2. The language `feed.xml` files on blob have been updated with the new `<item>` (first in channel).
3. `lang/index.html` (if used for local/blob testing) is uploaded.
4. A single git commit exists with message exactly like:
   ```
   Add episode 1 - SpaceX: Not an IPO — It's a Referendum

   Brivael Le Pogam describes the abnormal retail rush...
   ```
   (title from the primary language, summary/description in the body; includes the X link if useful).
5. The commit (and any posts/ archive) has been pushed to the remote.
6. User receives the live enclosure URLs (from the feed), confirmation of azcopy success, and the git commit SHA / message.
7. No files were generated or modified locally by this skill (it only uploads and commits existing changes).

## Architecture

**Blob layout (must match what the feed already declares):**
```
<base> = https://<storage>.blob.core.windows.net/<container>/
<base>/<lang>/feed.xml
<base>/<lang>/index.html
<base>/<lang>/<yyyy>/<MM>/<N>-<slug>.mp3
<base>/<lang>/<yyyy>/<MM>/<N>-<slug>.srt
<base>/<lang>/<yyyy>/<MM>/<N>-<slug>.md
<base>/<lang>/<yyyy>/<MM>/<N>-<slug>.jpg
<base>/posts/<yyyy-MM-dd>-<post-id>.md   (optional but recommended for archive)
```

**Git side:** All the local generated/updated files are added, committed, and pushed. The repo remains the source of truth for the markdown and feed sources.

**Config:** `[azure]` section in `voices.toml` supplies `storage` and `container`.

## Prerequisites

| Tool | Purpose | Verify | Install (Windows) |
|------|---------|--------|-------------------|
| azcopy | Upload files to Azure Blob | `azcopy --version` | `winget install Microsoft.Azure.AZCopy` |
| git | Commit and push | `git --version` | Usually pre-installed |
| az CLI | Required for AZCLI auth mode (no interactive azcopy login). Run `az login` once. | `az account show` | `winget install Microsoft.AzureCLI` |

**Auth (always use AZCLI mode - no device login):**
1. Ensure Azure CLI is authenticated once (outside or before the skill):
   ```powershell
   az login
   ```
2. Before **any** azcopy command in the session, set:

```powershell
$env:AZCOPY_AUTO_LOGIN_TYPE = "AZCLI"
$env:AZCOPY_TENANT_ID = (az account show --query tenantId -o tsv)
```

Then run azcopy commands normally. This forces azcopy to use the logged-in Azure CLI credentials.

This approach avoids hanging on `azcopy login` device code flows entirely. The env vars must be set in the same PowerShell session as the azcopy calls.

**Verify before upload:**
- The `podcast` skill has already run and the desired files + feed updates exist locally.
- `git status` shows the expected new/modified files (no unrelated junk staged).

## Workflow checklist

```
- [ ] Read voices.toml for storage + container; compute blob base URL
- [ ] Discover the episode to publish (parse first <item> from en/feed.xml or scan git status + episode dirs)
- [ ] Extract N, title, summary, slug, yyyy, MM, post link, affected languages, and any itunes:image references (for the jpg paths)
- [ ] Verify all expected local files exist (md, mp3, srt, jpg under each lang using the lang's slug for the jpg, updated feeds, index.html, posts archive)
- [ ] Upload via azcopy (episode assets, feeds, indexes; optionally posts/)
- [ ] git add the generated files
- [ ] git commit -m "Add episode [#] - [title]" -m "[summary]\n\n[link]"
- [ ] git push
- [ ] Report live blob URLs and commit details
```

---

## Step 1 — Resolve Azure destination from config

**Goal:** Know exactly where to upload.

1. Read `voices.toml`.
2. From `[azure]`:
   - `storage` (e.g. `kzu`)
   - `container` (e.g. `voices`)
3. Base URL: `https://$storage.blob.core.windows.net/$container`

**Outcome:** `$base` ready for azcopy targets.

---

## Step 2 — Discover the episode(s) ready for upload

**Goal:** Identify exactly which files belong to this publish without hard-coding.

Preferred methods (in order):

**A. From updated feed (recommended)**
- Load `en/feed.xml` (primary language).
- Take the first `<item>` (newest).
- Extract:
  - `<title>`
  - `itunes:episode` (as N)
  - `itunes:season` (year)
  - `<guid>` (used for consistency; usually `yyyy-MM-dd-N-slug`)
  - `<enclosure url="...">` — parse the path after the container to know exact remote subpath.
  - `<description>` or `<itunes:summary>` for commit body.
  - `<link>` (X post).
- From the enclosure or guid + pubDate, determine `yyyy`, `MM`, `slug`.
- Repeat for `es/feed.xml` if it also contains a matching first item (same X link or guid date).

**B. From filesystem + git status (fallback)**
- Run `git status --porcelain` (or `git ls-files --others --modified`).
- Find files matching `*/20[0-9][0-9]/[0-1][0-9]/*.(md|mp3|srt|jpg)`
- Group by the `#-slug` portion.
- For each unique episode base, locate the matching feed entry by scanning the `lang/feed.xml` first item.

Confirm with the user:
"About to upload episode 1: 'SpaceX: Not an IPO — It's a Referendum' (and Spanish version). Continue?"

**Outcome:** Concrete values for N, title, summary, slug, date parts, list of files to upload, and the languages involved.

---

## Step 3 — Verify local artifacts

For the discovered episode:

- `<lang>/<yyyy>/<MM>/<N>-<slug>.md`
- `<lang>/<yyyy>/<MM>/<N>-<slug>.mp3`
- `<lang>/<yyyy>/<MM>/<N>-<slug>.srt`
- `<lang>/<yyyy>/<MM>/<N>-<slug>.jpg` (artwork; present for each language)
- `<lang>/feed.xml` (modified)
- `<lang>/index.html` (if present and updated)
- `posts/<yyyy-MM-dd>-<post-id>.md`

All must exist locally. Fail early if anything is missing.

Optionally re-probe one mp3 with ffprobe to double-check metadata, but since the `podcast` skill already did this, it is normally unnecessary.

---

## Step 4 — Upload to Azure Blob Storage (azcopy)

**Always force AZCLI auth mode first** (in the current PowerShell session):

```powershell
$env:AZCOPY_AUTO_LOGIN_TYPE = "AZCLI"
$env:AZCOPY_TENANT_ID = (az account show --query tenantId -o tsv)
```

(Requires `az login` to have been performed previously.)

For each language:

1. Upload the episode directory contents (or files individually):
   ```
   azcopy copy "en/2026/06/1-spacex-not-an-ipo-its-a-referendum.mp3" `
     "https://kzu.blob.core.windows.net/voices/en/2026/06/1-spacex-not-an-ipo-its-a-referendum.mp3"
   ```
   Repeat for `.srt`, `.md`, and `.jpg`.

   Alternative (recursive for the month dir):
   ```
   azcopy copy "en/2026/06" "https://kzu.blob.core.windows.net/voices/en/2026/06" --recursive
   ```

2. Upload the feed and index (these must overwrite the previous version on blob):
   ```
   azcopy copy "en/feed.xml" "https://kzu.blob.core.windows.net/voices/en/feed.xml"
   azcopy copy "en/index.html" "https://kzu.blob.core.windows.net/voices/en/index.html"
   ```

3. (Recommended) Upload the original post archive:
   ```
   azcopy copy "posts/2026-06-17-2067308757307306269.md" `
     "https://kzu.blob.core.windows.net/voices/posts/2026-06-17-2067308757307306269.md"
   ```

Do the equivalent for `es/`.

Use `--overwrite` if azcopy prompts, or let it default to overwrite for these paths.

After uploads, the enclosure URLs that were already written into the local feed by the `podcast` skill are now live.

**Outcome:** All generated files reachable at their final blob locations.

---

## Step 5 — Git commit and push

1. Stage the exact generated files (be selective to avoid temp logs):
   ```
   git add posts/2026-06-17-2067308757307306269.md `
           en/2026/06/1-spacex-not-an-ipo-its-a-referendum.* `
           en/feed.xml en/index.html `
           es/2026/06/1-spacex-no-fue-una-opv-fue-un-referendum.* `
           es/feed.xml es/index.html
   ```

2. Commit:
   ```
   git commit -m "Add episode 1 - SpaceX: Not an IPO — It's a Referendum" `
                -m "Brivael Le Pogam describes the abnormal retail rush for SpaceX shares. It's not investing or speculation — ordinary people are voting with their money for the camp of builders and makers over the gray bureaucracy of control and permission.

   https://x.com/brivael/status/2067308757307306269"
   ```

   Use the primary (English) title for the subject line. Put the full summary + original link in the body.

3. Push:
   ```
   git push
   ```

   (If on a branch other than the default, the user may need `git push origin <branch>`.)

**Outcome:** The changes are committed with the required message format and published to GitHub.

---

## Step 6 — Report results

Tell the user:

- Live enclosure URL(s) (mp3) — they can be copied from the local feed or reconstructed.
- Live transcript URL(s) (.srt)
- Live artwork image URL(s) (.jpg per language) — from the <itunes:image> in the local feed
- Blob feed URL(s): e.g. `https://kzu.blob.core.windows.net/voices/en/feed.xml`
- Git commit subject and short SHA (run `git log -1 --oneline` after commit).
- Any files that were uploaded.

Suggest:
- Refresh the podcast in a client using the blob feed URL.
- Verify the audio plays and the transcript is available.
- Check the commit on GitHub.

---

## Examples

**Typical run after `podcast` skill:**

User: "Now upload the episode"

Agent:
1. Reads voices.toml → storage=kzu, container=voices.
2. Parses en/feed.xml → episode 1, title="SpaceX: Not an IPO — It's a Referendum", summary=..., guid=2026-06-17-1-..., link=...
3. Same for es.
4. Confirms files exist.
5. Sets AZCOPY_AUTO_LOGIN_TYPE=AZCLI + AZCOPY_TENANT_ID, then azcopy the 4 files (md+mp3+srt+jpg) × 2 langs + 2× feed.xml + 2× index.html + the posts/ file.
6. git add ... ; git commit -m "Add episode 1 - ..." -m "..."; git push.
7. "Episode 1 is now live at https://kzu.blob.../en/2026/06/1-....mp3 . Pushed commit abc1234."

---

## Troubleshooting

| Symptom | What to check |
|---------|---------------|
| azcopy 403 / auth error | Run `az login` (if needed), then set `$env:AZCOPY_AUTO_LOGIN_TYPE = "AZCLI"` and `$env:AZCOPY_TENANT_ID` before every azcopy invocation in the session. |
| File not found on blob after upload | Check the exact remote path matches the enclosure written by the podcast skill. |
| Wrong title or episode number in commit | The discovery step read the wrong feed item; ensure the first `<item>` is the new one. |
| git "nothing to commit" | The files were already committed, or wrong paths were added. Check `git status`. |
| Push rejected (non-fast-forward) | Pull first or rebase. |
| Feed on blob still shows old episode | The feed.xml upload step was skipped or used wrong path. |
| index.html broken after upload | Relative links inside it assume it lives at `<lang>/index.html` — make sure the dir structure matches. |

## Agent freedom

Implement using any combination of:
- PowerShell for XML/YAML parsing and file discovery.
- Direct `azcopy` CLI calls (preferred, matches existing patterns in the repo).
- `git` commands.
- Python or other for XML if easier.

The critical outcomes are:
- Correct files land at the exact blob paths declared in the feed.
- A properly formatted commit is created and pushed.
- The user sees confirmation that both storage and git are updated.

The local `en/feed.xml` / `es/feed.xml` (after the `podcast` run) are the source of truth for what the final remote state should contain.

---

## Related skills

- **podcast** (`.agents/skills/podcast/SKILL.md`) — The generator that must be run first to produce the local files (including per-lang .jpg artwork) and update the feeds locally (with itunes:image).
