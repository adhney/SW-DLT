# Multi-Step Download Flow & Auto-Save by Username

## Summary

Two connected features for SW-DLT:
1. When sharing a story or post URL, the Shortcut asks "Just this one or all from this user" before downloading
2. Downloads auto-save to a folder structure organized by username and content type, eliminating the share sheet prompt

## Architecture: Hybrid Approach

- **Shortcut** handles user-facing menus (scope selection: single vs all)
- **Python** handles URL parsing, username extraction, folder management, and downloading

## Section 1: URL Parsing & Username Extraction (Python)

New utility in `SW_DLT.py` that parses URLs and extracts:
- **username** from URL path
- **content type** (story, post, reel, generic)
- **item ID** if present

Instagram URL patterns:
- `/stories/<username>/<id>/` -> story, specific item
- `/stories/<username>/` -> story, all
- `/p/<shortcode>/` -> post (possibly carousel)
- `/reel/<shortcode>/` -> reel

Fallback for non-Instagram or unrecognized URLs: username = domain name, content type = `other`.

## Section 2: Shortcut Menu Flow

Updated Shortcut flow:

1. User shares/copies URL -> runs SW-DLT
2. Existing menu: Video / Audio / Playlist / Gallery
3. **New:** Shortcut checks if URL contains `/stories/` or `/p/` patterns (simple string match)
4. If detected -> second menu: "Just this one" / "All from this user"
5. If not detected -> skips to download

Scope choice passed to Python as a new argument:
- `-s` for single
- `-all` for all

## Section 3: Auto-Save by Username & Content Type

Folder structure:

```
Files App -> a-Shell -> Documents -> SW-DLT -> Downloads/
  <username>/
    stories/
      2026-03-14_story1.mp4
    posts/
      photo1.jpg
      photo2.jpg
    reels/
      reel1.mp4
    other/
      file1.mp4
```

Rules:
- Python creates `~/Documents/SW-DLT/Downloads/<username>/<type>/` before downloading
- `<type>` is `stories`, `posts`, `reels`, or `other`
- Files named with date prefix: `YYYY-MM-DD_<original_name_or_timestamp_id>`
- Multi-file "all" downloads save individually (no zipping)
- After saving, Shortcut shows a notification with the folder path (no share sheet)
- Fallback for unknown URLs: `Downloads/other/<domain>/`

## Section 4: Changes to Existing Download Methods

### single_video / single_audio

- Scope `-all` for stories: remove `playlist_items` restriction, let yt-dlp download all. Output to `<username>/stories/`
- Scope `-s` (single): keep `noplaylist: True` + `playlist_items: "1-1"`
- Scope `-all` for posts (carousels): remove `playlist_items`, download all items
- Change `outtmpl` to point to new folder structure

### gallery_download

- Scope `-s`: keep `--range "1"`
- Scope `-all`: remove range restriction
- Change `--directory` to new folder structure
- No zipping for `-all` — files stay in folder individually

### single_download (shared method)

- Accepts save path built from username + content type
- Returns success response with folder path
- For multi-file results, Shortcut opens folder in Files app

### Constructor changes

- `args[4]` (optional): scope flag (`-s` or `-all`)
- Username and content type parsed internally from URL

## Section 5: Error Handling & Edge Cases

| Scenario | Behavior |
|----------|----------|
| URL parsing fails (no username) | Falls back to `Downloads/other/<domain>/`, skips scope menu |
| "All" requested but only one item exists | Downloads single file normally |
| "All" stories but none active | Existing error handling shows exception |
| File name collision | Date prefix reduces collisions; append counter (`_1`, `_2`) if needed |
| Non-Instagram URL with "all" scope | Ignore scope, download normally |
| Auth / offline issues | No change, existing cookie auth and error handling |
