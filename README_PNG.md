# optimize-png: Batch PNG Optimiser

**optimize-png** is a bash script that batch-optimises PNG files with [oxipng](https://github.com/oxipng/oxipng) (and ImageMagick for trimming).
It losslessly recompresses every file in a folder, strips unnecessary embedded data, and optionally trims excess uniform margins, fully automated from the command line.

## Why this script?

Manually cleaning dozens of PNG files is tedious and error-prone.
This script automates the entire process:

- processes a whole folder in one command
- uses all available CPU cores
- automatically backs up original files before making any change
- never makes a file larger: a result is written only if it is smaller
- installs all missing dependencies on first run

## What it does

| Step | Tool | What happens |
|------|------|-------------|
| 1 (only with `--trim`) | ImageMagick | Trims uniform background pixels at edges with zero colour tolerance |
| 2 | oxipng | Lossless recompression; strips embedded ICC profiles, EXIF data, comments and unnecessary PNG chunks |

### What `--trim` does

Without `--trim` the script never touches the canvas dimensions (oxipng only recompresses and strips metadata).

With `--trim`, ImageMagick also runs `-fuzz 0% -trim`:

- `-fuzz 0%` means **only perfectly identical pixels** at the image edges are considered background and removed. This conservative approach protects logos and designs with intricate borders, gradients or semi-transparent edges.
- `-trim` removes the border region of matching pixels.
- `+repage` resets the canvas size to the trimmed image, removing any virtual offset ImageMagick would otherwise keep from the original geometry.

### Why strip metadata?

PNG files often contain embedded metadata that is invisible to the eye but adds weight to the file: ICC colour profiles, EXIF chunks, creation timestamps, comments and software tags.
oxipng (`--strip all`) removes all of this, reducing file size without affecting the image content. With `--keep-icc` the ICC profile is kept (`--strip safe`).

## Requirements

- Debian or WSL2 Debian (Windows users: see below)
- oxipng: **downloaded automatically** on first run (latest release, into `/usr/local/bin`) if not in the PATH
- ImageMagick (`magick` or `convert`): needed **only** for `--trim`, or as a fallback when oxipng cannot be installed. Installed automatically via `apt` when needed.

## Installation

Save [optimize_png.bash](https://raw.githubusercontent.com/mapi68/optimize-svg-png/refs/heads/master/optimize_png.bash) then run:

```bash
mkdir -p ~/.local/bin
cp optimize_png.bash ~/.local/bin/optimize_png
chmod +x ~/.local/bin/optimize_png
```

The script is now available system-wide as `optimize_png`.

## Usage

```bash
optimize_png [--trim] [--keep-icc] <folder>
```

The original files are backed up to `<folder>_backup_png` before processing.
If the backup folder already exists the script exits immediately without modifying anything.

### Options

`--trim` — trim excess uniform background pixels at edges (zero colour tolerance) and reset the canvas size. Images that would collapse to a single pixel are left untouched.

`--keep-icc` — keep the embedded ICC colour profile and strip everything else. Without it the profile is removed too, which can shift colours on images that are not plain sRGB.

**How the work is split.** Without `--trim` the whole job is done by [oxipng](https://github.com/oxipng/oxipng) (`-o 4 --strip all -p`): lossless, pixel-identical, it never rewrites a file that would not get smaller, and it removes the same metadata as ImageMagick `-strip`. ImageMagick is **not used and not required**. With `--trim` ImageMagick crops the margins first (oxipng cannot crop) and oxipng runs right after. With `--keep-icc` oxipng uses `--strip safe`, which keeps the ICC profile.

If `oxipng` is not in the PATH the script downloads the latest release from <https://github.com/oxipng/oxipng/releases/latest/download> for the current architecture (`x86_64` or `aarch64`, static musl build) and installs it in `/usr/local/bin` (using `sudo` when needed). If that fails, the script warns and falls back to ImageMagick alone (re-encoding with `-strip`, and only replacing a file when the result is smaller). Files ImageMagick failed on are not passed to oxipng.

In the ImageMagick stage a file is replaced only if the result is strictly smaller; otherwise the original is kept. File permissions are preserved, `.PNG` is matched too, and the exit status is 1 if any file failed.

Without `--trim` only metadata is stripped and the original canvas dimensions are preserved exactly.

### Example

```bash
optimize_png /home/user/picons/png
optimize_png --trim /home/user/picons/png
```

```
Backup saved to: /home/user/picons/png_backup_png
--------------------------------------------------------
Starting PNG optimization in: /home/user/picons/png
Files found: 12  |  CPU cores: 8
--------------------------------------------------------
Mode: strip metadata only (use --trim to also trim margins)
--------------------------------------------------------
  ✓ icon_01.png
  ✓ icon_02.png
  ✓ icon_03.png
  ...
--------------------------------------------------------
Processing completed in 3s using 8 cores.

========================================================
                        REPORT
========================================================
------------------------------------------------------------------
  icon_01.png          45.32 KB  →   38.17 KB  (15.8%)
  icon_02.png          12.80 KB  →   10.44 KB  (18.4%)
  icon_03.png          78.55 KB  →   61.20 KB  (22.1%)
  ...
------------------------------------------------------------------
  TOTAL (12 files)    412.80 KB  →  334.55 KB
  Space saved: 78.25 KB  (18.9%)
========================================================
```

### Help

```bash
optimize_png --help
```

## Report

At the end of each run the script prints a per-file report showing:

- original size
- optimised size
- percentage saved

Sizes are displayed in B, KB or MB depending on the largest file in the batch.
The final line shows total space saved across all processed files.

## Safety

- The backup folder is created **after** confirming that PNG files exist in the target folder, so no empty backup is ever created.
- Each file is first written to a temporary file (`mktemp`). In the ImageMagick stage (`--trim`) the original is overwritten only if ImageMagick succeeds and the result is smaller; oxipng itself only rewrites a file when the result is smaller.
- If the backup folder already exists the script exits immediately without modifying anything.

## Limitations

- Processes only `.png` files at the top level of the specified folder (not recursive).
- The `-fuzz 0%` setting (when using `--trim`) is very conservative and trims only perfectly uniform background. Files with artwork extending to the edges may not be trimmed at all.
- The script is designed for Debian and Debian-based systems. It may work on other Linux distributions but `apt`-based auto-installation will not function outside Debian/Ubuntu.

## Windows users (WSL2)

The script runs on Windows via **WSL2** (Windows Subsystem for Linux) with a Debian distribution.
All dependencies are installed automatically inside the Linux environment — nothing needs to be installed on the Windows side.

### Install WSL2 with Debian

Open PowerShell as Administrator and run:

```powershell
wsl --install -d Debian
```

Reboot if prompted, then open the **Debian** app from the Start menu and complete the first-time user setup.

### Install the script inside WSL2

Open the Debian terminal and follow the [Installation](#installation) steps above.

### Access Windows files from WSL2

Your Windows drives are accessible under `/mnt/`:

```bash
optimize_png /mnt/c/Users/YourName/Desktop/png
```

### Performance tip

WSL2 I/O is slower when reading and writing files on the Windows filesystem (`/mnt/c/...`).
For large batches, copying files to the Linux filesystem first is significantly faster:

```bash
cp -r /mnt/c/Users/YourName/Desktop/png ~/png
optimize_png ~/png
cp ~/png/*.png /mnt/c/Users/YourName/Desktop/png/
```
