# oomtm_setup

Our own copy of the programs [oomtm](https://github.com/mtm-aarhus/oomtm) installs on the robot machines **without admin rights**, so a robot can always get the exact version it expects, even when the original download address changes or disappears.

It happened on 2026-11-20: oomtm pointed at LibreOffice 26.2.4 under `download.documentfoundation.org/.../stable/`, and `/stable/` only keeps the current releases. When 26.2.5 came out, the 26.2.4 address returned 404, and every new robot machine failed to install LibreOffice.

## What is here

The files are **GitHub Release assets**, not files in the repository. GitHub refuses files over 100 MB in a repository; the LibreOffice MSI alone is about 356 MB. A release asset can be up to 2 GB.

One release per program and version:

| Program | Release tag | File (asset) | Size | SHA-256 | Used for |
|---|---|---|---|---|---|
| LibreOffice | `libreoffice-26.2.7.3` | `LibreOffice_26.2.7.3_Win_x86-64.msi` | 373,121,024 bytes | `480fecf759fc6393bd614fbf43753eb95a8198fc1a065d41b727e3b664cd1e59` | Office files → PDF |
| WeasyPrint (Windows build) | `weasyprint-70.0` | `weasyprint-windows-onedir.zip` | 32,271,073 bytes | `ab1151f210b4e6bb7aa7a79e91a67e8ddb760094c107bfda55241b6aaefe7d53` | Mails, HTML, text and CSV → PDF |

oomtm downloads from:

```
https://github.com/mtm-aarhus/oomtm_setup/releases/download/<tag>/<file>
```

and falls back to the original address if the file isn't here. Either way the download must match the SHA-256 pinned in oomtm, or it is thrown away and nothing is installed.

**The repository must stay public.** The robots download without a GitHub login.

### The `releases` folder

`releases/<tag>/<file>` holds each file locally, already named and placed as it must be uploaded: the folder name is the release tag, the file inside is the asset. The folder is in `.gitignore`, so it is never committed (a commit with the MSI would be refused by GitHub).

To upload: on GitHub, **Releases → Draft a new release**, type the folder name as the tag (e.g. `libreoffice-26.2.7.3`), use the same as the title, drag the file in, put its SHA-256 from the table in the description, and publish. Or with the GitHub CLI, from this folder:

```powershell
gh release create libreoffice-26.2.7.3 .\releases\libreoffice-26.2.7.3\LibreOffice_26.2.7.3_Win_x86-64.msi --repo mtm-aarhus/oomtm_setup --title "LibreOffice 26.2.7.3" --notes "SHA-256: 480fecf759fc6393bd614fbf43753eb95a8198fc1a065d41b727e3b664cd1e59"
```

```powershell
gh release create weasyprint-70.0 .\releases\weasyprint-70.0\weasyprint-windows-onedir.zip --repo mtm-aarhus/oomtm_setup --title "WeasyPrint 70.0 (Windows)" --notes "SHA-256: ab1151f210b4e6bb7aa7a79e91a67e8ddb760094c107bfda55241b6aaefe7d53"
```

Keep the file names exactly as they are.

## How oomtm installs them (no admin)

- **LibreOffice:** the MSI is unpacked with `msiexec /a` (an "administrative install", which only copies the files, so there's no UAC prompt) into `%LOCALAPPDATA%\oomtm\libreoffice\<version>\`. Each version gets its own folder.
- **WeasyPrint:** the Python package comes with `oomtm[pdf]`. On Windows it also needs Pango, HarfBuzz and fontconfig, which aren't pip packages. WeasyPrint's own Windows build has them, and oomtm unpacks **only those libraries and fontconfig's settings** into `%LOCALAPPDATA%\oomtm\weasyprint\<version>\`, not `weasyprint.exe`.

Both installs are guarded by a lock file, so when several robots start at once only one downloads and the others wait.

## Updating to a new version

1. **Download the new file from the original source**
   - LibreOffice: from the archive, which keeps every build:
     `https://downloadarchive.documentfoundation.org/libreoffice/old/<version>/win/x86_64/LibreOffice_<version>_Win_x86-64.msi`
     (`<version>` has four parts, e.g. `26.2.7.3`; list them at `https://downloadarchive.documentfoundation.org/libreoffice/old/`)
   - WeasyPrint: `weasyprint-windows-onedir.zip` from the release at `https://github.com/Kozea/WeasyPrint/releases`. The Python package `weasyprint` must be updated to the **same** version (see step 4).

2. **Compute its SHA-256** in PowerShell:

   ```powershell
   (Get-FileHash .\LibreOffice_26.2.7.3_Win_x86-64.msi -Algorithm SHA256).Hash.ToLower()
   ```

3. **Put it in `releases/<tag>/` and create the release** as described above.

4. **Update oomtm**, in one commit:
   - LibreOffice: `LIBREOFFICE_VERSION` and `LIBREOFFICE_SHA256` in `src/oomtm/pdf.py`
   - WeasyPrint: `WEASYPRINT_VERSION` and `WEASYPRINT_SHA256` in `src/oomtm/engines.py`, **and** `weasyprint == <version>` in the `pdf` extra in `pyproject.toml`
   - Update the table above as well.

5. **Run oomtm's conversion tests** (`tests/test_konvertering.py`, see oomtm's README, "Tests"). They convert a whole set of test files with the new version, check the text and characters, run 8 robots at once, and save a picture of every page in `tests/ud/index.html` to look at.

6. **Point the robots at the new oomtm commit** (`oomtm[pdf] @ git+https://github.com/mtm-aarhus/oomtm.git@<commit>` in each robot's `pyproject.toml`). The robots that convert to PDF are KontAKTF2ToPDF, KontAKTGOToPDF, KontAKTNovaToPDF, KontAKTMappeToPDF and KontAKTJournalizeGO.

   On its first run with the new commit, each robot machine downloads and unpacks the new version once. The old version stays in its own folder, in case an older robot on the same machine still uses it.

**Delete old releases only** when no robot's oomtm commit points at that version any more.

## Licences

- **LibreOffice** is under the Mozilla Public License 2.0, which allows redistributing the unmodified installer.
- **WeasyPrint** is under the BSD 3-Clause licence; the libraries in its Windows build (Pango, HarfBuzz, fontconfig, GLib and others) are under LGPL/MIT-style licences that allow redistributing them unmodified.
- The Noto Emoji font oomtm uses is not here: it ships inside oomtm (`src/oomtm/fonts/`, SIL Open Font License).
