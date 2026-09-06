---
name: xhs-cli
description: "Headless-browser-based CLI skill for Xiaohongshu (小红书, RedNote, XHS) to search notes, read posts, browse profiles, like, favorite, comment, publish from the terminal, and chat with the Diandian (点点) AI assistant"
author: jackwener
version: "1.1.0"
tags:
  - xhs
  - xiaohongshu
  - 小红书
  - rednote
  - social-media
  - cli
  - mcp
  - ai
---

> [!NOTE]
> An alternative package [xiaohongshu-cli](https://github.com/jackwener/xiaohongshu-cli) is available, which uses a reverse-engineered API and runs faster.
> This package (`xhs-cli`) uses a headless browser (camoufox) approach — slower but more resilient against risk-control detection.
> Choose whichever best fits your needs.

# xhs-cli Skill

A CLI tool for interacting with Xiaohongshu (小红书). Use it to search notes, read details, browse user profiles, and perform interactions like liking, favoriting, and commenting.

## Prerequisites

```bash
# Install (requires Python 3.8+)
uv tool install xhs-cli
# Or: pipx install xhs-cli
```

## Authentication

All commands require valid cookies to function.

```bash
xhs status                     # Check saved login session (no browser extraction)
xhs login                      # Auto-extract Chrome cookies
xhs login --cookie "a1=..."    # Or provide cookies manually
```

Authentication first uses saved local cookies. If unavailable, it auto-detects local Chrome cookies via browser-cookie3. If extraction fails, QR code login is available.

## Command Reference

### Search

```bash
xhs search "咖啡"              # Search notes (rich table output)
xhs search "咖啡" --json       # Raw JSON output
```

### Read Note

```bash
# View note (xsec_token auto-resolved from search cache)
xhs read <note_id>
xhs read <note_id> --comments  # Include comments
xhs read <note_id> --xsec-token <token>  # Manual token
xhs read <note_id> --json
```

> [!IMPORTANT]
> A note page returns HTTP 404 (error 300031) without a valid `xsec_token`.
> **Always run `xhs search` first** — it caches the token for each result —
> then `xhs read <note_id>` resolves it automatically.

### AI Chat (Diandian 点点)

```bash
xhs ai "周末上海周边有什么小众一日游？"          # Ask, get the answer in a panel
xhs ai "问题" --json                          # {"answer": cleaned, "raw": full text}
```

- Drives the web Diandian page (`xiaohongshu.com/ai_chat`) inside the same
  headless session and captures the streamed answer.
- One-shot per invocation (each run starts a fresh conversation); takes
  ~15-60s depending on answer length.
- The `answer` field is cleaned (question echo, "ai总结N篇笔记" meta chip and
  disclaimer removed); `raw` keeps everything.
- Like any AI summary, answers may hallucinate — for fact-checking workflows,
  prefer `xhs search` + `xhs read` over `xhs ai`.

### User

```bash
# Look up user profile (by internal user_id, hex format)
xhs user <user_id>
xhs user <user_id> --json

# List user's published notes
xhs user-posts <user_id>
xhs user-posts <user_id> --json

# Followers / Following
xhs followers <user_id>
xhs following <user_id>
```

### Discovery

```bash
xhs feed                       # Explore page recommended feed
xhs feed --json
xhs topics "旅行"              # Search topics/hashtags
xhs topics "旅行" --json
```

### Interactions (require login)

```bash
# Like / Unlike (xsec_token auto-resolved)
xhs like <note_id>
xhs like <note_id> --undo

# Favorite / Unfavorite
xhs favorite <note_id>
xhs favorite <note_id> --undo

# Comment
xhs comment <note_id> "好棒！"

# Delete your own note
xhs delete <note_id>
```

### Favorites

```bash
xhs favorites                  # List your favorites
xhs favorites --max 10         # Limit count
xhs favorites --json
```

### Post

```bash
xhs post "标题" --image photo1.jpg --image photo2.jpg --content "正文"
xhs post "标题" --image photo1.jpg --content "正文" --json
```

### Account

```bash
xhs status                     # Quick saved-session check
xhs whoami                     # Full profile info
xhs whoami --json
xhs login                      # Login
xhs logout                     # Clear cookies
```

## JSON Output

Major query commands support `--json` for machine-readable output:

```bash
xhs search "咖啡" --json | jq '.[0].id'           # First note ID
xhs whoami --json | jq '.userInfo.userId'          # Your user ID
xhs favorites --json | jq '.[0].displayTitle'      # First favorite title
```

## Common Patterns for AI Agents

```bash
# Get your user ID for further queries
xhs whoami --json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('userInfo',{}).get('userId',''))"

# Search and get note IDs (xsec_token auto-cached for later use)
xhs search "topic" --json | python3 -c "import sys,json; [print(n['id']) for n in json.load(sys.stdin)[:3]]"

# Check login before performing actions
xhs status && xhs like <note_id>

# Read a note with comments for summarization
xhs read <note_id> --comments --json
```

## Error Handling

- Commands exit with code 0 on success, non-zero on failure
- Error messages are prefixed with ❌
- Login-required commands show clear instruction to run `xhs login`
- `xsec_token` is auto-resolved from cache; manual `--xsec-token` available as fallback

## Session Troubleshooting (important for agents)

- `xhs status` only checks that a local cookie file exists — it can report
  "Logged in" while the **server-side session has already expired**.
- **Treat these errors as an expired session, not a bug**: `search.feeds not
  ready after 15.0s`, `note.noteDetailMap not ready after 15.0s`, `user info
  not ready after 10.0s`. Re-login with `xhs login --qrcode`, then retry.
- QR login quirks:
  - The QR code prints to stdout as terminal art and the whole flow times out
    after **4 minutes**. When running in the background, poll the log for the
    QR URL and render it as an image/PNG for the user to scan quickly
    (`npx -y qrcode -o qr.png "<qr_url>"`), or paste the QR art directly.
  - A QR code expires after ~1-2 minutes; if it expires the page shows a
    refresh overlay — restart the login for a fresh code.
  - If launch hangs on "Falling back to QR code login...", kill leftovers
    first: `taskkill /f /im camoufox.exe` (Windows) and retry.

## Frontend Compatibility Notes (verified 2026-09)

Xiaohongshu's web frontend changes over time. The CLI extracts from
`window.__INITIAL_STATE__` where available and falls back to parsing the
rendered DOM when the state has been consumed by hydration. Current behavior:

- **Search**: results are parsed from `section.note-item` cards when needed.
  Each card's link carries the note id and the `xsec_token` (base64, may end
  with `=`) required by `read`/`like`/`favorite`.
- **Read**: note detail and comments fall back to the rendered
  `#detail-title` / `#detail-desc` / `.comment-item` elements. The returned
  `time`/`ipLocation` fields are best-effort in fallback mode.
- `xsec_token` values are single-use-ish and expire; re-run `xhs search` to
  refresh them if `read` starts failing.

### When Xiaohongshu updates its frontend again (expected)

This **will happen again** — it broke extraction once in September 2026 and
the fix above is a fallback layer, not a permanent guarantee. Recognize and
repair it as follows:

- **Symptom**: `search.feeds not ready`, `note.noteDetailMap not ready`,
  `note detail DOM not ready`, or "Failed to extract" errors **with a valid
  login** (i.e. the feed/explore pages render fine in a real browser). If the
  page instead shows a login wall, it is an expired session — re-login first
  and only then suspect a frontend change.
- **Diagnose**: open the failing page in a browser and inspect the rendered
  DOM. Check whether the old anchors still exist (`section.note-item`,
  `#detail-title`, `#detail-desc`, `.comment-item`, `.textarea` on ai_chat)
  and whether card hrefs still carry `xsec_token`. Also dump
  `Object.keys(window)` — if `__INITIAL_STATE__` has come back, prefer it.
- **Fix**: update the selector strings inside the fallback JS blocks in
  `xhs_cli/client.py` (`search_notes`, `_extract_note_detail_dom`,
  `_extract_comments_dom`, `ai_chat`) — they are plain CSS selectors and can
  be rewritten without touching the surrounding flow.
- **Discipline**: keep the DOM fallbacks as the source of truth; treat any
  return of `__INITIAL_STATE__`-only parsing as a regression.

## Environment Notes

- The bundled uBlock Origin addon is **intentionally skipped**
  (`exclude_addons`): addons.mozilla.org is unreachable on some networks and
  stalls every launch. Add it back by removing the `exclude_addons` argument
  in `xhs_cli/client.py` / `xhs_cli/auth.py` if you need ad blocking.
- Headless launches force **software rendering** (`firefox_user_prefs`) to
  avoid `GFX1-RenderCompositorSWGL` crashes on machines with flaky GPU
  drivers.
- If launches still time out (`BrowserType.launch: Timeout ... exceeded`) or
  the browser crashes on start, the local Camoufox build is likely corrupted —
  reinstall it:

  ```bash
  taskkill /f /im camoufox.exe        # clear stuck processes (Windows)
  python -m camoufox fetch            # re-download / re-extract
  ```

## Safety Notes

- Do not ask users to share raw cookie values in chat logs.
- Prefer auto-extraction via `xhs login` over manual cookie input.
- If auth fails, ask the user to re-login via `xhs login`.
- This tool drives a real logged-in session — space out requests and keep
  volumes low; aggressive scraping risks account-level risk control
  (error 300011 "账号异常" / 300031 page 404).

