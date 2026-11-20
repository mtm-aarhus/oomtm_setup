# oomtm_setup

Our own copy of the programs [oomtm](https://github.com/mtm-aarhus/oomtm) installs on the robot machines **without admin rights**, so a robot can always get the exact version it expects, even when the original download address changes or disappears.

It happened on 2026-11-20: oomtm pointed at LibreOffice 26.2.4 under `download.documentfoundation.org/.../stable/`, and `/stable/` only keeps the current releases. When 26.2.5 came out, the 26.2.4 address returned 404, and every new robot machine failed to install LibreOffice.

## What is here

The files are **GitHub Release assets**, not files in the repository. GitHub refuses files over 100 MB in a repository; the LibreOffice MSI alone is about 356 MB. A release asset can be up to 2 GB.

One release per program and version:

| Program | Release tag | File (asset) | Size | SHA-256 | Used by |
|---|---|---|---|---|---|
| LibreOffice | `libreoffice-26.2.7.3` | `LibreOffice_26.2.7.3_Win_x86-64.msi` | 373,121,024 bytes | `480fecf759fc6393bd614fbf43753eb95a8198fc1a065d41b727e3b664cd1e59` | Office files → PDF |
| Chromium (Chrome for Testing headless shell) | `chromium-158.0.8127.36` | `chrome-headless-shell-win64.zip` | 127,876,264 bytes | `9655950dfeddf565561cd64cfe8ee716ccd6482fa3798503b632219efc65a71e` | Mails, HTML, text and CSV → PDF |

Download each file from the original address in "Updating" below and check its hash before uploading it. oomtm refuses a file whose hash doesn't match.

oomtm downloads from:

```
https://github.com/mtm-aarhus/oomtm_setup/releases/download/<tag>/<file>
```

and falls back to the original address if the file isn't here. Either way the download must match the SHA-256 pinned in oomtm, or it is thrown away and nothing is installed.

**The repository must stay public.** The robots download without a GitHub login.

## How oomtm installs them (no admin)

- **LibreOffice:** the MSI is unpacked with `msiexec /a` (an "administrative install", which only copies the files, so there's no UAC prompt) into `%LOCALAPPDATA%\oomtm\libreoffice\<version>\`. Each version gets its own folder.
- **Chromium:** the zip is unpacked into `%LOCALAPPDATA%\oomtm\chromium\<version>\`.

Both installs are guarded by a lock file, so when several robots start at once only one downloads and the others wait.

## Updating to a new version

1. **Download the new file from the original source**
   - LibreOffice: from the archive, which keeps every build:
     `https://downloadarchive.documentfoundation.org/libreoffice/old/<version>/win/x86_64/LibreOffice_<version>_Win_x86-64.msi`
     (`<version>` has four parts, e.g. `26.2.7.3`; list them at `https://downloadarchive.documentfoundation.org/libreoffice/old/`)
   - Chromium: the current stable version is in
     `https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json`
     (`channels.Stable`, the `chrome-headless-shell` download for `win64`).

2. **Compute its SHA-256** in PowerShell:

   ```powershell
   (Get-FileHash .\LibreOffice_26.2.7.3_Win_x86-64.msi -Algorithm SHA256).Hash.ToLower()
   ```

3. **Create the release here** with the tag from the table and the file as an asset. Use either the web page (Releases → Draft a new release) or:

   ```powershell
   gh release create libreoffice-26.2.7.3 .\LibreOffice_26.2.7.3_Win_x86-64.msi --repo mtm-aarhus/oomtm_setup --title "LibreOffice 26.2.7.3" --notes "SHA-256: <hash>"
   ```

   Keep the file name exactly as downloaded. Put the hash in the release notes, so anyone can check the file.

4. **Update oomtm**, in one commit:
   - LibreOffice: `LIBREOFFICE_VERSION` and `LIBREOFFICE_SHA256` in `src/oomtm/pdf.py`
   - Chromium: `CHROMIUM_VERSION` and `CHROMIUM_SHA256` in `src/oomtm/engines.py`
   - Update the table above as well.

5. **Run oomtm's conversion tests** (see oomtm's README, "Tests"). They convert the whole test set with the new version and show the result.

6. **Point the robots at the new oomtm commit** (`oomtm[pdf] @ git+https://github.com/mtm-aarhus/oomtm.git@<commit>` in each robot's `pyproject.toml`). The robots that convert to PDF are KontAKTF2ToPDF, KontAKTGOToPDF, KontAKTNovaToPDF, KontAKTMappeToPDF and KontAKTJournalizeGO.

   On its first run with the new commit, each robot machine downloads and unpacks the new version once. The old version stays in its own folder, in case an older robot on the same machine still uses it.

**Delete old releases only** when no robot's oomtm commit points at that version any more.

## Licences

- **LibreOffice** is under the Mozilla Public License 2.0, which allows redistributing the unmodified installer.
- **Chrome for Testing** is Google's build of the open-source Chromium (BSD licence, with third-party components). Check Google's terms for Chrome for Testing before relying on this copy as the main source.
