# Multi-Step Download & Auto-Save Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a "just this one or all" prompt for Instagram stories/posts and auto-save downloads to username/type folders.

**Architecture:** Hybrid — Shortcut handles the scope menu UI, Python handles URL parsing, username extraction, folder creation, and downloading. The Shortcut plist is a massive XML file that must be edited carefully. Python changes are in a single file (`src/SW_DLT.py`).

**Tech Stack:** Python 3 (a-Shell), yt-dlp, gallery-dl, Apple Shortcuts (plist XML)

**Note:** This project runs on iOS in a-Shell. There is no test framework. Verification is done by reading the code and testing manually on device. Each task is verified by reviewing the diff before committing.

---

## Chunk 1: Python Changes

### Task 1: Add `parse_url()` function

**Files:**
- Modify: `src/SW_DLT.py` (add function before `SW_DLT` class)

- [ ] **Step 1: Add the `re` import**

Add `import re` to the imports at the top of the file (after `import os`).

- [ ] **Step 2: Add `parse_url()` function**

Add this function after the `Consts` class and before the `SW_DLT` class:

```python
def parse_url(url):
    """Parse URL to extract username, content type, and whether it's Instagram."""
    parsed = urllib.parse.urlparse(url)
    hostname = parsed.hostname or ""
    path = parsed.path.rstrip("/")

    # Instagram story: /stories/<username>/<id> or /stories/<username>
    story_match = re.match(r"/stories/([^/]+)(?:/(\d+))?", path)
    if "instagram.com" in hostname and story_match:
        return {
            "platform": "instagram",
            "username": story_match.group(1),
            "content_type": "stories",
            "has_item_id": story_match.group(2) is not None
        }

    # Instagram post: /p/<shortcode>
    post_match = re.match(r"/p/([^/]+)", path)
    if "instagram.com" in hostname and post_match:
        return {
            "platform": "instagram",
            "username": None,  # resolved later via extract_info
            "content_type": "posts",
            "has_item_id": True
        }

    # Instagram reel: /reel/<shortcode>
    reel_match = re.match(r"/reel/([^/]+)", path)
    if "instagram.com" in hostname and reel_match:
        return {
            "platform": "instagram",
            "username": None,  # resolved later via extract_info
            "content_type": "reels",
            "has_item_id": True
        }

    # Fallback: use domain as username, generic type
    domain = hostname.replace("www.", "")
    return {
        "platform": "other",
        "username": domain or "unknown",
        "content_type": "other",
        "has_item_id": False
    }
```

- [ ] **Step 3: Review the diff and commit**

```bash
git add src/SW_DLT.py
git commit -m "Add parse_url() for Instagram URL pattern detection"
```

---

### Task 2: Add scope flag parsing and `save_path` builder to constructor

**Files:**
- Modify: `src/SW_DLT.py:21-59` (SW_DLT.__init__)

- [ ] **Step 1: Update constructor to scan for scope flag**

Add scope parsing after the existing arg handling (after line 59). The scope flag (`--single` or `--all`) is always the last arg if present. Strip it from args before existing positional parsing.

Replace the constructor with:

```python
def __init__(self, file_id, *args):
    # Scan for scope flag (always last arg if present)
    args = list(args)
    self.scope = "--single"
    if args and args[-1] in ("--single", "--all"):
        self.scope = args.pop()

    # args[0]: media URL to download
    # args[1]: main process to run
    # args[2] (dependent): resolution for video, type for playlist, or range for gallery
    # args[3] (dependent): framerate for video
    self.media_url = args[0]
    self.file_id = file_id
    self.date_id = datetime.datetime.today().strftime("%Y-%m-%d")
    self.url_info = parse_url(self.media_url)
    self.ytdlp_globals = {
        "color": "never",
        "quiet": True,
        "no_warnings": True,
        "noprogress": True,
        "progress_hooks": [show_progress],
        "postprocessor_hooks": [format_processing],
        "cookiesfrombrowser": ("safari",)
    }

    processes = {
        "-v": self.single_video,
        "-a": self.single_audio,
        "-p": self.playlist_download,
        "-g": self.gallery_download
    }
    self.run = processes[args[1]]
    self.video_res = ""
    self.video_fps = ""
    self.playlist_type = ""
    self.gallery_range = ""

    if len(args) > 2:
        self.video_res = args[2] if args[1] == "-v" else ""
        self.playlist_type = args[2] if args[1] == "-p" else ""
        self.gallery_range = args[2].replace('"','').replace("'","") if args[1] == "-g" else ""

    if len(args) > 3:
        self.video_fps = args[3]
```

Key changes:
- Scope flag scanned and stripped from args before positional parsing
- `self.date_id` format changed to `YYYY-MM-DD` (ISO, sortable)
- `self.url_info` stores parsed URL data
- `self.scope` stores `--single` or `--all`

- [ ] **Step 2: Add `build_save_path()` method**

Add this method to the `SW_DLT` class, after the constructor:

```python
def build_save_path(self, username=None):
    """Build the auto-save folder path for Instagram downloads."""
    info = self.url_info
    if info["platform"] != "instagram":
        return None  # non-Instagram uses existing share sheet flow

    name = username or info["username"] or "unknown"
    content_type = info["content_type"]
    save_dir = os.path.join(
        os.environ["HOME"], "Documents", "SW-DLT", "Downloads",
        name, content_type
    )
    os.makedirs(save_dir, exist_ok=True)
    return save_dir
```

- [ ] **Step 3: Review the diff and commit**

```bash
git add src/SW_DLT.py
git commit -m "Add scope flag parsing and save path builder to constructor"
```

---

### Task 3: Update `single_download()` to support auto-save and scope

**Files:**
- Modify: `src/SW_DLT.py` (single_download method)

- [ ] **Step 1: Replace `single_download()` method**

```python
def single_download(self, dl_options):
    # Uses yt-dlp to download single video or audio items
    with yt_dlp.YoutubeDL(dl_options) as vid_obj:
        meta_data = vid_obj.extract_info(self.media_url, download=False)
        if meta_data is None:
            raise Exception(Consts.DERROR_EXC)

        vid_title = meta_data.get("title", self.date_id)

        # Resolve username from metadata if not available from URL
        username = self.url_info.get("username")
        if not username and self.url_info["platform"] == "instagram":
            username = meta_data.get("uploader") or meta_data.get("channel") or "unknown"

        save_dir = self.build_save_path(username)

        # If Instagram auto-save: adjust outtmpl and scope
        if save_dir:
            dl_options["outtmpl"] = os.path.join(
                save_dir, f"{self.date_id}_%(title)s.%(ext)s"
            )
            if self.scope == "--all":
                dl_options.pop("playlist_items", None)
                dl_options.pop("noplaylist", None)

        vid_obj.params.update(dl_options)
        vid_obj.download([self.media_url])

    # Instagram auto-save: return saved response
    if save_dir:
        files = [f for f in os.listdir(save_dir) if f.startswith(self.date_id)]
        output = {
            "output_code": "saved",
            "folder_path": save_dir,
            "file_count": len(files),
            "file_title": vid_title
        }
        return f'shortcuts://run-shortcut?name=SW-DLT&input=text&text={urllib.parse.quote(json.dumps(output))}'

    # Non-Instagram: existing behavior (find file by file_id)
    for file in os.listdir():
        if file.startswith(self.file_id):
            output = {
                "output_code": "success",
                "file_name": os.path.abspath(file),
                "file_title": vid_title
            }
            return f'shortcuts://run-shortcut?name=SW-DLT&input=text&text={urllib.parse.quote(json.dumps(output))}'
    raise Exception(Consts.DERROR_EXC)
```

- [ ] **Step 2: Review the diff and commit**

```bash
git add src/SW_DLT.py
git commit -m "Update single_download to support Instagram auto-save and scope"
```

---

### Task 4: Update `single_video()` and `single_audio()` for scope-aware options

**Files:**
- Modify: `src/SW_DLT.py` (single_video, single_audio methods)

- [ ] **Step 1: Update `single_video()`**

The `noplaylist` and `playlist_items` should only be set when scope is `--single`. When `--all`, we let yt-dlp download all items. Also update `outtmpl` to use `file_id` as default (save_dir override happens in `single_download`).

```python
def single_video(self):
    default_format = "best/bestvideo+bestaudio"
    custom_format = ""\
        "bestvideo[height={0}][fps<={1}]+bestaudio/"\
        "best[height={0}][fps<={1}]/"\
        "bestvideo[height<={0}][fps<={1}]+bestaudio/"\
        "best[height<={0}][fps<={1}]"\
        "best[height={0}]".format(self.video_res, self.video_fps)

    dl_options = {
        "format": default_format if self.video_res == "-d" else custom_format,
        "outtmpl": f'{self.file_id}.%(ext)s',
        "format_sort": ["res", "ext:mp4:m4a", "codec:avc:m4a"],
        **self.ytdlp_globals
    }

    if self.scope == "--single":
        dl_options["noplaylist"] = True
        dl_options["playlist_items"] = "1-1"

    try:
        return self.single_download(dl_options)
    except (yt_dlp.utils.DownloadError, OSError) as ex:
        raise Exception(ex.args[0])
```

- [ ] **Step 2: Update `single_audio()`**

```python
def single_audio(self):
    dl_options = {
        "format": "bestaudio[ext*=4]/bestaudio[ext=mp3]/best[ext=mp4]/best",
        "postprocessors": [{"key": "FFmpegExtractAudio", "preferredcodec": "m4a"}],
        "outtmpl": f'{self.file_id}.%(ext)s',
        **self.ytdlp_globals
    }

    if self.scope == "--single":
        dl_options["noplaylist"] = True
        dl_options["playlist_items"] = "1-1"

    try:
        return self.single_download(dl_options)
    except (yt_dlp.utils.DownloadError, OSError) as ex:
        raise Exception(ex.args[0])
```

- [ ] **Step 3: Review the diff and commit**

```bash
git add src/SW_DLT.py
git commit -m "Make single_video and single_audio scope-aware"
```

---

### Task 5: Update `gallery_download()` for auto-save and scope

**Files:**
- Modify: `src/SW_DLT.py` (gallery_download method)

- [ ] **Step 1: Replace `gallery_download()` method**

```python
def gallery_download(self):
    try:
        # Resolve username — for posts/reels, try gallery-dl metadata
        username = self.url_info.get("username") or "unknown"
        save_dir = self.build_save_path(username)

        if save_dir:
            # Instagram auto-save to structured folder
            dest_dir = save_dir
        else:
            # Non-Instagram: use temp folder (existing behavior)
            dest_dir = self.file_id
        os.makedirs(dest_dir, exist_ok=True)

        # Build base command
        range_flag = ""
        if self.scope == "--single" and self.gallery_range:
            range_flag = f'--range "{self.gallery_range}"'
        elif self.scope == "--single":
            range_flag = '--range "1"'
        # --all: no range restriction

        base_cmd = "gallery-dl {0} {1} --cookies-from-browser safari".format(
            self.media_url, range_flag).strip()

        # Pre-scan to count total items
        scan_cmd = base_cmd + " --no-download"
        with subprocess.Popen(scan_cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, bufsize=1, universal_newlines=True) as scan:
            total_items = sum(1 for _ in scan.stdout)
        total_items = max(total_items, 1)

        # Actual download with progress tracking
        # Use --dest for absolute path, --directory "" to suppress auto-subdirs
        dl_cmd = base_cmd + ' --dest {0} --directory ""'.format(dest_dir)
        curr_item = 0
        with subprocess.Popen(dl_cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, bufsize=1, universal_newlines=True) as gdl:
            for entry in gdl.stdout:
                curr_item += 1
                show_progress("manual", curr_item, total_items)

        files = os.listdir(dest_dir)
        if len(files) == 0:
            raise OSError()

        # Instagram auto-save: return saved response
        if save_dir:
            output = {
                "output_code": "saved",
                "folder_path": save_dir,
                "file_count": len(files),
                "file_title": self.date_id
            }

        # Non-Instagram single item: return file directly
        elif len(files) < 2:
            file = "{0}/{1}".format(dest_dir, files[0])
            output = {
                "output_code": "success",
                "file_name": os.path.abspath(file),
                "file_title": self.date_id
            }

        # Non-Instagram multiple items: zip and return
        else:
            shutil.make_archive(self.file_id, "zip", dest_dir)
            output = {
                "output_code": "success",
                "file_name": os.path.abspath(self.file_id + ".zip"),
                "file_title": self.date_id
            }

        return f'shortcuts://run-shortcut?name=SW-DLT&input=text&text={urllib.parse.quote(json.dumps(output))}'
    except subprocess.CalledProcessError as ex:
        clean_stderr = ex.output.decode("utf-8").rstrip()
        raise Exception(clean_stderr)
    except (AttributeError, OSError) as ex2:
        raise Exception(Consts.DERROR_EXC)
```

- [ ] **Step 2: Review the diff and commit**

```bash
git add src/SW_DLT.py
git commit -m "Update gallery_download for Instagram auto-save and scope"
```

---

### Task 6: Update `main()` to handle scope in arg hashing and metadata

**Files:**
- Modify: `src/SW_DLT.py:258-313` (main function)

- [ ] **Step 1: Update `main()` to strip scope from hash and metadata args**

The scope flag should not change the file_id hash (so resuming works regardless of scope). Update the relevant lines:

```python
def main():
    info_msgs = {
        "-v": f'{Consts.CBLUE}Video Download{Consts.ENDL}\n{Consts.CYELLOW}Custom qualities require processing{Consts.ENDL}',
        "-a": f'{Consts.CBLUE}Audio Download{Consts.ENDL}\n{Consts.CYELLOW}Sometimes audio processing is needed{Consts.ENDL}',
        "-p": f'{Consts.CBLUE}Playlist Download{Consts.ENDL}\n{Consts.CYELLOW}Process time depends on playlist length{Consts.ENDL}',
        "-g": f'{Consts.CBLUE}Gallery Download{Consts.ENDL}\n{Consts.CYELLOW}Process time depends on collection length{Consts.ENDL}',
        "-e": f'{Consts.CYELLOW}Deleting All Dependencies{Consts.ENDL}',
        "update_check": f'{Consts.CBLUE}Preparing{Consts.ENDL}\n{Consts.CYELLOW}Checking for Updates{Consts.ENDL}'
    }
    try:
        globals()["yt_dlp"] = __import__("yt_dlp")

        # Strip scope flag from args for hashing (so resume works regardless of scope)
        hash_args = [a for a in sys.argv if a not in ("--single", "--all")]
        file_id = "SW_DLT_DL_{}".format(hashlib.md5(
            str(hash_args).encode("utf-8")).hexdigest()[0:20])

        sw_dlt_inst = SW_DLT(file_id, *sys.argv[1:])
        header = f'{Consts.SBOLD}SW-DLT{Consts.ENDL}'

        # Pre-download check and cleanup
        subprocess.run("clear && hideKeyboard", shell=True)

        print(header)
        print(info_msgs["update_check"])
        sw_dlt_inst.update_check()

        # If the same partial file is not found, deletes all leftovers (important)
        for file in os.listdir():
            if file.startswith("SW_DLT_DL_") and not file.startswith(file_id):
                if os.path.isdir(file):
                    shutil.rmtree(file)
                    continue
                os.remove(file)
            elif file.startswith(file_id):
                header = f'{Consts.SBOLD}SW-DLT (Resuming Download){Consts.ENDL}'

        subprocess.run("clear")
        print(header)
        print(info_msgs[sys.argv[2]])

        with open('SW_DLT_DL_metadata.json', 'w') as metadata:
            meta_args = [a for a in sys.argv[2:] if a not in ("--single", "--all")]
            args = ' '.join(map(str, meta_args))
            json.dump({f"{sys.argv[1]}": args}, metadata)

        return sw_dlt_inst.run()

    except Exception as exc_url:
        if str(exc_url.args[0]) not in [Consts.DERROR_EXC]:
            b64_err = base64.b64encode(exc_url.args[0].encode()).decode()
            UNK_EXC = '{{"output_code":"exception","exc_trace":"{0}"}}'.format(b64_err)
            return f'shortcuts://run-shortcut?name=SW-DLT&input=text&text={urllib.parse.quote(UNK_EXC)}'
        return f'shortcuts://run-shortcut?name=SW-DLT&input=text&text={urllib.parse.quote(str(exc_url.args[0]))}'
```

- [ ] **Step 2: Review the diff and commit**

```bash
git add src/SW_DLT.py
git commit -m "Update main() to handle scope flag in hashing and metadata"
```

---

## Chunk 2: Shortcut Plist Changes

The Shortcut plist (`src/SW-DLT.plist`) is a large XML file defining the iOS Shortcut actions. These changes add the scope selection menu for Instagram URLs.

### Task 7: Add Instagram URL detection and scope menu to Shortcut plist

**Files:**
- Modify: `src/SW-DLT.plist`

**Context:** The plist defines Shortcut actions as XML dictionaries. The scope menu needs to be inserted after the existing download-type menu (Video/Audio/Playlist/Gallery) and before the shell script execution.

- [ ] **Step 1: Identify the insertion point in the plist**

Read the plist to find where the existing menu result is passed to the shell command. The new scope menu should be inserted between the download-type menu and the script execution action.

- [ ] **Step 2: Add an "If" action that checks if the URL contains `instagram.com`**

This checks the URL variable for the Instagram domain AND one of `/stories/`, `/p/`, or `/reel/` patterns.

- [ ] **Step 3: Add a "Choose from Menu" action inside the If block**

Menu options:
- "Just this one" -> sets a variable `scope` to `--single`
- "All" -> sets a variable `scope` to `--all`

- [ ] **Step 4: Add an "Otherwise" (else) block**

Sets `scope` to empty string (no scope flag for non-Instagram URLs).

- [ ] **Step 5: Append the `scope` variable to the shell command arguments**

The existing shell command that runs `python SW_DLT.py <url> <mode> ...` should append the scope variable at the end.

- [ ] **Step 6: Add handling for `"saved"` output_code in the Shortcut return flow**

After the script returns, check if `output_code` is `"saved"`. If so, show a notification with "Saved {file_count} files to {folder_path}" instead of the share sheet.

- [ ] **Step 7: Review the diff and commit**

```bash
git add src/SW-DLT.plist
git commit -m "Add Instagram scope menu and saved notification to Shortcut"
```

---

## Chunk 3: Final Verification & Docs

### Task 8: Update README roadmap and documentation

**Files:**
- Modify: `README.md`
- Modify: `Docs.md`

- [ ] **Step 1: Update README roadmap**

Mark the completed items and add the new features:
- [x] Implement full progress bar on gallery-dl downloads
- [x] Multi-step download flow (just this one / all)
- [x] Auto-save to username/type folders

- [ ] **Step 2: Add documentation for the new features to Docs.md**

Add a new section "Multi-Step Downloads" explaining:
- How the scope prompt works for Instagram stories and posts
- The auto-save folder structure and where to find downloads
- That non-Instagram URLs still use the share sheet

- [ ] **Step 3: Commit docs**

```bash
git add README.md Docs.md
git commit -m "Update docs with multi-step download and auto-save features"
```

---

### Task 9: Push and update release

- [ ] **Step 1: Push all changes**

```bash
git push origin master
```

- [ ] **Step 2: Update the GitHub release**

Update v4.2.3 or create v4.3.0 with release notes covering all new features.
