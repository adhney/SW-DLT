# Multi-Step Download Flow & Auto-Save by Username

## Summary

Two connected features for SW-DLT:
1. When sharing a story or post URL, the Shortcut asks "Just this one or all" before downloading
2. Downloads auto-save to a folder structure organized by username and content type, eliminating the share sheet prompt

## Scope

Initial implementation targets **Instagram URLs only**. Non-Instagram URLs use the fallback behavior (domain-based folder, no scope prompt). Other platforms (TikTok, Twitter/X, Reddit) can be added later by extending the URL parser.

## Architecture: Hybrid Approach

- **Shortcut** handles user-facing menus (scope selection: single vs all)
- **Python** handles URL parsing, username extraction, folder management, and downloading

## Section 1: URL Parsing & Username Extraction (Python)

New `parse_url()` utility in `SW_DLT.py` that parses URLs and extracts:
- **username** from URL path or metadata
- **content type** (story, post, reel, generic)
- **item ID** if present

Instagram URL patterns:
- `/stories/<username>/<id>/` -> story, specific item. Username from URL path.
- `/stories/<username>/` -> story, all. Username from URL path.
- `/p/<shortcode>/` -> post (possibly carousel). **Username NOT in URL** — must be resolved via `extract_info(url, download=False)` using the `uploader` or `channel` metadata field.
- `/reel/<shortcode>/` -> reel. Same as post — username from metadata.

Fallback for non-Instagram or unrecognized URLs: username = domain name (e.g. `twitter.com`), content type = `other`.

## Section 2: Shortcut Menu Flow

Updated Shortcut flow:

1. User shares/copies URL -> runs SW-DLT
2. Existing menu: Video / Audio / Playlist / Gallery
3. **New:** Shortcut checks if URL matches `instagram.com` domain AND contains `/stories/` or `/p/` or `/reel/` patterns
4. If detected -> second menu: "Just this one" / "All"
5. If not detected -> skips to download (no scope prompt)

"All" means:
- For stories: all active stories from that user
- For posts: all items in this specific carousel/post (NOT all posts by the user)
- For reels: ignored (reels are single items)

### Argument passing

Scope is always the **last argument** passed to the Python script. The script scans `sys.argv` for the scope flag rather than relying on positional indexing, since preceding optional arguments vary by download type.

- `--single` for single item
- `--all` for all items

If no scope flag is found in args, default behavior is `--single`.

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
- Files named with date prefix: `YYYY-MM-DD_<original_name_or_timestamp_id>` (note: this replaces the existing `DD-MM-YY-HH-MM-SS` convention with a sortable ISO format)
- Multi-file "all" downloads save individually (no zipping)
- Fallback for unknown URLs: `Downloads/other/<domain>/`

### Return flow

The Python script returns a JSON response via `shortcuts://` URL:
- New `output_code`: `"saved"` (distinct from existing `"success"`)
- New field `folder_path`: absolute path to the save folder
- New field `file_count`: number of files saved

The Shortcut handles `"saved"` by showing a notification: "Saved X files to SW-DLT/<username>/<type>". No share sheet. No Files app navigation (iOS Shortcuts cannot open a specific subfolder in Files reliably).

### Backward compatibility

- **Instagram URLs** with scope prompt -> auto-save to folder, notification
- **Non-Instagram URLs** -> existing behavior unchanged (share sheet)
- This is not a toggle — the behavior is determined by whether the URL is Instagram or not

## Section 4: Changes to Existing Download Methods

### single_video / single_audio

- Scope `--all` for stories: remove `playlist_items` restriction, let yt-dlp download all. Output to `<username>/stories/`
- Scope `--single`: keep `noplaylist: True` + `playlist_items: "1-1"`
- Scope `--all` for posts (carousels): remove `playlist_items`, download all items in this post
- Change `outtmpl` to point to new folder structure

### gallery_download

- Scope `--single`: keep existing `--range` behavior
- Scope `--all`: remove range restriction
- Use `--dest` (not `--directory`) to set the absolute output path to the new folder structure
- No zipping for `--all` — files stay in folder individually

### single_download (shared method)

- Builds save path from `parse_url()` result (username + content type)
- Creates folder structure before downloading
- Returns `"saved"` response with folder path and file count
- For username extraction from post/reel URLs: calls `extract_info(url, download=False)` and reads `uploader` field before the main download

### Constructor changes

- Scope flag (`--single` or `--all`) scanned from `sys.argv` (last arg), not positional
- `self.scope` attribute added, defaults to `--single`
- Username and content type parsed internally from URL via `parse_url()`

## Section 5: Error Handling & Edge Cases

| Scenario | Behavior |
|----------|----------|
| URL parsing fails (no username) | Falls back to `Downloads/other/<domain>/`, skips scope menu |
| Username metadata extraction fails | Falls back to domain name |
| "All" requested but only one item exists | Downloads single file normally |
| "All" stories but none active | Existing error handling shows exception |
| File name collision | Date prefix reduces collisions; append counter (`_1`, `_2`) if needed |
| Non-Instagram URL with scope flag | Ignore scope, download normally with share sheet |
| Auth / offline issues | No change, existing cookie auth and error handling |
| Playlist menu option selected for story URL | Scope prompt does not appear (only for Video/Audio/Gallery) |
