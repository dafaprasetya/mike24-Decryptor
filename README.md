# mike24-Decryptor — `.mike24` Partial-Encryption Recovery Tool

```
    ____  ____  ___    __  _____    ____  ____
   / __ \/ __ \/   |  / / / /   |  / __ \/  _/
  / /_/ / /_/ / /| | / /_/ / /| | / /_/ // /
 / ____/ _, _/ ___ |/ __  / ___ |/ _, _// /
/_/   /_/ |_/_/  |_/_/ /_/_/  |_/_/ |_/___/
  _____     __             ____                 _ __         ___
 / ___/_ __/ /  ___ ____  / __/__ ______ ______(_) /___ __  / _ \_______  ___ ________ ___ _
/ /__/ // / _ \/ -_) __/ _\ \/ -_) __/ // / __/ / __/ // / / ___/ __/ _ \/ _ `/ __/ _ `/  ' \
\___/\_, /_.__/\__/_/   /___/\__/\__/\_,_/_/ /_/\__/\_, / /_/  /_/  \___/\_, /_/  \_,_/_/_/_/
    /___/                                          /___/                /___/
```

**`mike24_recovery`** gets back the original data that the **`.mike24` ransomware** (Montage group; Microsoft detection
`Ransom:Win64/Dementor.NSN!MTB`) leaves untouched inside large files, **without the attacker's key and without paying**.

> ⚠️ **This is NOT a decryptor**, despite the repository name. No key is recovered and nothing is decrypted.
> The tool salvages the parts of each file that the ransomware never encrypted. Fully encrypted files stay encrypted.

Authors: **EDITH & Prahari White Hat Org** · part of the **PRAHARI Cyber Security Program**

---

## Why recovery is possible

`.mike24` files start with a `RAWR` header followed by a version byte:

| Header version | What the ransomware did | What this tool can do |
|---|---|---|
| `01` | **Whole file encrypted** (small files) | Nothing from the file itself. Needs an original copy (backup, another PC, e-mail attachment) or disk carving. `--find-copies` helps collect copies. |
| `02` | **Only the first ~4 MB and the last ~4 MB encrypted** (large files) | Everything in between is still original data and is recovered. |

In practice files **larger than ~8–10 MB** are partially encrypted; smaller files are usually fully encrypted. The tool
reads the header of **every** file, so it never guesses from the file size alone.

## What gets recovered

| File type | Result |
|---|---|
| ZIP / XLSX / DOCX / PPTX / ODT … | Every inner file that is still intact is extracted (CRC-verified). Files stored inside the first 4 MB are lost. |
| RAR (RAR4 / RAR5) | The intact entries are re-packed into a new `*_RECOVERED.rar` that extracts without errors in WinRAR / 7-Zip. Entries cut by the encrypted end are left out and listed in the report. **Solid** archives cannot be salvaged this way and are flagged. |
| Access (`.mdb` `.mde` `.accdb` `.accde`) | Every table that is still readable is extracted and, on Windows, an **open-able copy `<name>_RECOVERED.mdb`** is built from it (see [Access databases](#access-databases)). To save space the intermediate build files are then removed (`--keep-build-file` keeps them). |
| Anything else | Rebuilt the same way (encrypted zones = zeros). Useful for databases, disk images, video, large logs, etc. |

## Download

Get **`mike24_recovery.exe`** from the [Releases](../../releases) page. It runs on Windows 10 / Server 2016 or newer and
**needs no Python installation**.

Verify the download before running it:

```powershell
Get-FileHash .\mike24_recovery.exe -Algorithm SHA256
```

| File | Version | SHA-256 |
|---|---|---|
| `mike24_recovery.exe` | 3.2.3 | `4b1b86b7fb8a93596ac2f726f0db62f3f8a26390e4ddfae626b6077ef0fc45cf` |
| `mike24_recovery.py` | 3.2.3 | `879c3d1f3e32f4e766601b4e2f5bfa59d31e41d00b988504ce660990d4f17267` |

The `.exe` is a PyInstaller build of `mike24_recovery.py` and is **not code-signed**. Some antivirus engines flag
unsigned PyInstaller executables. If that happens, run the Python script instead (below) or build the `.exe` yourself.

## Quick start

> **Before you start**
> 1. **Isolate** the affected machine from the network and make sure the ransomware is no longer running.
> 2. **Take a VM snapshot or a full disk image.** Never work on the only copy of the evidence.
> 3. Prefer writing the output to **another disk** (USB/external/second virtual disk).

Open a Command Prompt (or PowerShell) **as Administrator** in the folder that contains the tool:

```bat
:: 1. Classify only (fast, writes just report.csv and summary.txt)
mike24_recovery.exe "D:\data" "E:\recovered" --report-only

:: 2. Full recovery
mike24_recovery.exe "D:\data" "E:\recovered"

:: 3. Re-run only the files that failed or were skipped (e.g. the disk was full)
mike24_recovery.exe "D:\data" "E:\recovered" --retry-failed "E:\recovered\report.csv"

:: 4. Also look for original copies of fully encrypted files on other folders / USB drives
mike24_recovery.exe "D:\data" "E:\recovered" --find-copies "F:\backup" "\\pc-finance\share"

:: 5. No second disk available: write to the same disk (take a VM snapshot first!)
mike24_recovery.exe "D:\data" "D:\recovered" --on-disk
```

Double-clicking the `.exe` without arguments shows the help screen.

## Access databases

A partially encrypted Access file loses its first ~4 MB: the header page, the table catalog and most table
definitions. That is why Access says **"Unrecognized database format"**, and why *Compact & Repair* does not help.
The data pages in the middle are still there. Version 3.2 reads them directly:

1. **Tables to CSV** - one CSV per table in `<name>_tables\` (UTF-8, opens in Excel), plus `_tables.txt`
   (rows per table, how the columns were identified) and `_schema.json`.
2. **Open-able copy** - on Windows the tool then builds **`<name>_RECOVERED.mdb`** from those CSV files with the
   Access / Jet database engine that is already installed (64-bit Office engine if present, otherwise the 32-bit Jet
   4.0 engine built into Windows). This new database opens normally in Access.

Column names and types come from, in this order:

| Source | Column names | When |
|---|---|---|
| Table definition still intact in the file | real | definition was stored after the first 4 MB |
| `--mdb-template` | real | definition lost; a healthy database of the **same application** is given |
| Guessed from the rows | `f4_date`, `f12_int32`, `var3`, ... | definition lost and no template |

**Use a template whenever possible.** Any undamaged copy of a database made by the same program works, even an
empty one or an older version: the installation folder of the application, another branch, an old backup.
With a template, `<name>_RECOVERED.*` is a copy of the template (same tables, queries, forms, relationships) in which
the recovered rows replace the rows of the matching tables. Newer columns that the template does not have are added
(names start with `extra_`).

```bat
:: Access files recovered by an older version ("Unrecognized database format"): no need to start again
mike24_recovery.exe "D:\recovered" "E:\tables" --extract-mdb --mdb-template "C:\App\Data.mdb"

:: Full run with a template (Access files get CSV + open-able copy automatically)
mike24_recovery.exe "D:\data" "E:\recovered" --mdb-template "C:\App\Data.mdb" "C:\App\Front.accde"
```

Things to know:

- `[LOST]` in a CSV cell = a memo/attachment value that was stored in the encrypted part.
- Rows rejected when building the open-able copy (for example duplicate primary keys from old row versions that Access
  had already discarded) are counted in `_make_mdb.log`. The CSV files always contain every row that was read.
- **Disk space.** Once `<name>_RECOVERED.*` is built, the build files are cleaned up: the zero-filled rebuild (which
  Access cannot open anyway) is not written, and the CSV folder is deleted - or zipped to `<name>_tables.zip` when
  Access rejected some rows, so no row is lost. A short summary stays as `<name>_tables.txt`. If the copy cannot be
  built (no Windows / no Access engine), everything is kept so the copy can be built later.
  `--keep-build-file` keeps the rebuild and the CSV folder. The `.mike24` input files are never modified or deleted.
- The open-able copy needs Windows PowerShell and the Access/Jet engine. If it cannot be built (e.g. no engine for an
  `.accdb` template), the CSV files are still complete and `make_mdb.ps1` is left in the folder to run later.
- Access 97 (Jet3, 2 KB pages) files: rebuild only, no table extraction.
- `.accde` / `.mde` front-ends: tables can be recovered, forms/reports/VBA code cannot. Reinstall the front-end from
  the application installer instead.

### From source (any OS, Python 3.8+, standard library only)

```bash
python mike24_recovery.py "D:\data" "E:\recovered" --report-only
python mike24_recovery.py "D:\data" "E:\recovered"
```

## Options

| Option | Description |
|---|---|
| `input` | Folder that contains the `.mike24` files. It is **only read**, never modified. |
| `output` | Folder for the recovered data and the reports. Must not be inside `input`. |
| `--report-only` | Classify every file (fully encrypted / partial / type) without writing recovered data. |
| `--find-copies FOLDER [FOLDER ...]` | Index other folders (other PCs, USB drives, old backups, e-mail downloads) and copy the newest original that is **older than the attack** for every encrypted file. Copies newer than the attack are still used but flagged `CHECK`. |
| `--attack-time "YYYY-MM-DD HH:MM"` | Local time of the encryption, used by `--find-copies`. Default: inferred from the oldest `RANSOM_NOTE.html` (creation time) or the oldest `.mike24` file. |
| `--retry-failed REPORT_CSV` | Re-run only the `FAILED` / `SKIPPED` files of a previous run; writes `report-retry-*.csv`, `summary-retry-*.txt` and a merged report. |
| `--on-disk` (`--same-disk`) | Allow the output on the same disk as the data. See *Same-disk warning* below. |
| `--force` | Continue even if the estimated free space is not enough (files that do not fit are marked `SKIPPED`). |
| `--head-mb N` / `--tail-mb N` | Size of the encrypted leading / trailing zones (default 4 / 4). Only change this if your samples show a different pattern. |
| `--extract-mdb` | Input is a folder (or one file) of Access files **already recovered** by an earlier run: extract their tables and build the open-able copy. No `.mike24` needed. |
| `--mdb-template MDB [MDB ...]` | Healthy database(s) of the same application, for real column names/types. Put this option **last** on the command line. |
| `--no-make-mdb` | Only write CSV files, do not build `<name>_RECOVERED.mdb`. |
| `--keep-build-file` | Access files: keep the zero-filled rebuild and the CSV folder after the open-able copy is built (by default they are removed to save space). |
| `--delete-raw` | With `--extract-mdb`: delete each zero-filled rebuilt input file after its open-able copy was built (cleans up the output of older versions). Healthy databases and `.mike24` files are never deleted. |
| `--no-banner` | Do not print the banner (useful in scripts and logs). |
| `--version` | Print the version. |

## Output

```
E:\recovered\
├── recovered\          recovered data, same folder structure as the input
│   ├── ...\Database_RECOVERED.mdb    open-able copy built from the recovered tables (Windows)
│   ├── ...\Database.mdb_tables.txt   summary per table (rows, how the columns were identified)
│   ├── ...\Database.mdb_tables.zip   CSV, only if Access rejected rows (or a folder with --keep-build-file /
│   │                                  when the copy could not be built, plus make_mdb.ps1 to build it later)
│   ├── ...\Backup_RECOVERED.rar      intact RAR entries, re-packed
│   └── ...\Report.xlsx_contents\     intact files from inside a ZIP/Office file
├── original_copies\    originals found with --find-copies
├── report.csv          one line per .mike24 file
└── summary.txt         totals, key IDs, failure causes, retry command
```

`report.csv` columns: `file, size_mb, status, type, key_id, result, notes, copy`.

| Status | Meaning |
|---|---|
| `FULLY ENCRYPTED` | Header `01`. Needs an original copy or carving. |
| `PARTIAL (recoverable)` | Header `02`. Original data recovered as described above. |
| `FAILED` | Error while processing the file (cause in `notes`, e.g. `DISK FULL while writing`). |
| `SKIPPED (disk space)` | Not processed because the output disk did not have enough space. |

The `key_id` column shows the per-host key identifier written by the ransomware. Files encrypted on different machines
have different IDs.

## Safety

- The input folder is opened **read-only**; nothing in it is changed, moved or deleted.
- Free space is checked before the run and before every file. Half-written output is deleted when a write fails, so no
  0-byte files are left behind.
- Long Windows paths (`\\?\`) are supported.
- **Same-disk warning:** the original files that the ransomware deleted may still be in the disk's free space and can be
  brought back only by carving. Writing tens of GB of output to the same disk can overwrite them, especially the small
  fully encrypted files. That is why `--on-disk` is required to write there, and why you should snapshot the VM first.
- The tool never contacts the network and never runs or touches the ransomware binary.

## Limitations

- Not a decryptor: the encrypted head and tail of every file are **lost** without the attacker's key.
- Fully encrypted (small) files cannot be recovered from the file itself.
- Access: rows stored in the first/last ~4 MB are lost, and so is the table catalog. Without a template, tables whose
  definition was lost get guessed column names; deleted rows that were still physically present can come back as
  duplicates (they are rejected by primary keys when the open-able copy is built).
- RAR entries stored in the encrypted zones, and solid RAR archives, cannot be salvaged.
- The 4 MB zone sizes match the samples analysed so far. Other builds of the ransomware may use different values
  (`--head-mb` / `--tail-mb`).
- Always open recovered files on a clean machine and check them before putting them back into production.

## Build the .exe yourself

On Windows with Python 3.8+:

```bat
pip install pyinstaller
pyinstaller --onefile --console --name mike24_recovery mike24_recovery.py
```

On Linux, with Docker and Wine (no Windows needed; this is how the release was built):

```bash
docker run --rm --network none -v "$PWD":/src batonogov/pyinstaller-windows:latest \
  "pyinstaller --onefile --clean -y --console --name mike24_recovery --distpath /src/dist --workpath /tmp/build --specpath /tmp/spec /src/mike24_recovery.py"
```

The release build was checked by running both the `.exe` (under Wine) and the `.py` on the same synthetic test set
(ZIP, RAR4, RAR5, solid RAR5, Access Jet3/ACE, fully encrypted files): all output files were byte-identical.
Table extraction was checked against synthetic Access 2000 and Access 2010 databases (500,000+ rows, all column
types): on undamaged files every row matches exactly; with the first and last 4 MB destroyed and a template given,
no wrong values were produced (80-94% of the rows recovered, the rest was in the destroyed part).

## If you were hit by `.mike24`

- **Do not pay** and do not contact the attackers. Payment does not guarantee a working key and funds further attacks.
- Isolate affected machines, but do not wipe them before evidence is collected.
- Check [No More Ransom](https://www.nomoreransom.org/) for new decryptors.
- Report the incident to your national CERT / cyber agency (in Indonesia: [BSSN](https://www.bssn.go.id/)) and, where
  personal data is involved, follow your data-protection notification rules.
- Close the way in before restoring data: look for **Remote Desktop exposed to the internet**, shared or weak local
  accounts, and remote-access tools, then change every password used on the affected machines.

## Disclaimer

This software is provided "as is", without warranty of any kind. Use it only on systems and data you are authorised to
work on, and only on a copy of the evidence. The authors are not responsible for any data loss.

---

**EDITH & Prahari White Hat Org** · PRAHARI Cyber Security Program
