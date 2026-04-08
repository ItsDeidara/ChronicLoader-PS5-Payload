# ChronicLoader

ChronicLoader is a PS5 payload updater and `autoload.txt` manager.

It checks the latest GitHub releases for the repos you configure, downloads the payload files you want, and saves them with stable filenames so your autoloader setup does not need to keep changing.

It can also manage `autoload.txt`, keep a download log, and track SHA-256 hashes so it can tell the difference between a real update, an unchanged file, and a damaged local copy.

ChronicLoader can:

- Download direct release assets such as `.elf` and `.bin`
- Download `.zip` release assets and pull payload files out of them (i.e. shadowmountplus)
- Save files into `/data/ps5_autoloader` or another folder you choose
- Maintain `autoload.txt` with optional delays between payloads
- Automatically append itself to `autoload.txt` so it runs on every boot
- Back up the autoloader folder to a timestamped ZIP before making any changes
- Keep a download log in `/data/chronicloader/chronic.log`
- Keep SHA-256 state in `/data/chronicloader/chronic_state.json`
- Read its runtime config from `/data/chronicloader/chronicloaderSettings.json`

## What You Need

- A PS5 with network access
- A way to send payloads to the console
- `klogsrv-ps5.elf` if you want live logs
- `chronicloader.elf`

Optional:

- `chronicloader_settings_builder.html` to generate the config file more easily

## Main Files

- Payload: `/data/ps5_autoloader` is the default output folder on the PS5
- Runtime config: `/data/chronicloader/chronicloaderSettings.json`
- Download log: `/data/chronicloader/chronic.log`
- SHA-256 state: `/data/chronicloader/chronic_state.json`
- Backups: `/data/chronicloader/chronicBackups/` (timestamped ZIP archives)

## How It Works

When ChronicLoader runs, it:

1. Loads `/data/chronicloader/chronicloaderSettings.json`
2. Backs up the autoloader folder to a timestamped ZIP (if backups are enabled)
3. Checks the latest release for each configured GitHub repo
4. Finds the matching asset or assets
5. Downloads them to the PS5
6. Renames them to stable output names when needed
7. Compares SHA-256 values so unchanged files do not get downloaded again unless something is missing or damaged
8. Writes log and state entries with the source asset name, release tag, destination path, size, and SHA-256
9. Writes `autoload.txt` with all tracked entries and optionally appends itself as the last entry

If the config folder is missing, ChronicLoader creates `/data/chronicloader/` and writes default template files for you.

## How To Use It

1. Load `chronicloader.elf` with whatever method you use for PS5 payloads
2. If this is the first run, let ChronicLoader create its default files in `/data/chronicloader/`
3. Build or edit `/data/chronicloader/chronicloaderSettings.json`
4. Run ChronicLoader again so it can check your configured repos and update the payload files
5. Reboot if the final notification tells you changes were applied


## Settings File

ChronicLoader reads its config from:

`/data/chronicloader/chronicloaderSettings.json`

The easiest way to build this file is with:

`chronicloader_settings_builder.html`

The file has four main parts:

- `notifications`
  - Controls how chatty the payload is while it runs
  - `verbose: false` keeps notifications minimal and only shows per-repo results
  - `verbose: true` shows progress notifications such as downloading, extracting, and verifying
- `autoload`
  - Controls whether ChronicLoader also maintains `autoload.txt`
  - `enabled` turns autoload maintenance on or off
  - `output_path` is where the generated `autoload.txt` will be written
  - `self_autoload` appends ChronicLoader itself as the last entry in `autoload.txt` with an 8-second delay so it runs again on every boot
  - `custom_entries` lets you preserve extra autoload lines that are not tied to a tracked repo entry
- `backup`
  - Controls automatic backup of the autoloader folder before ChronicLoader makes any changes
  - `enabled` turns backups on or off
  - `max_backups` sets how many backup ZIPs to keep (oldest are deleted when this limit is exceeded)
  - Backups are saved to `/data/chronicloader/chronicBackups/` as timestamped ZIP files
- `repos`
  - This is the list of GitHub release pages ChronicLoader will check
  - Each repo entry tells ChronicLoader what release page to inspect, which asset family to follow, where to save it, and whether to include it in autoload

Each repo entry uses these fields:

- `release_url`
  - The GitHub releases page for the project you want to follow
  - Example: `https://github.com/seregonwar/zftpd/releases`
- `formats`
  - A simple filter for the file type you want
  - Common values are `all`, `elf`, `bin`, `zip`, or `elf,bin`
- `dest_dir`
  - The folder on the PS5 where the downloaded file should be saved
  - If you are feeding a payload autoloader, this is usually `/data/ps5_autoloader`
- `asset_rule`
  - The rule ChronicLoader uses to keep following the same file family across future releases even when version numbers change
  - This is usually generated for you by the HTML builder
- `autoload_preview_name`
  - The stable output name used for previews and autoload matching
  - This is what the builder uses when it compares imported `autoload.txt` lines back to repo entries
- `autoload`
  - If `true`, this repo entry can be written into the managed `autoload.txt`
  - If `false`, the file will still be downloaded and updated, but it will not be added to autoload
- `delay_before_ms`
  - The delay written before this entry in `autoload.txt`
  - Use this when one payload should wait for another payload to finish first

A simple example looks like this:

```json
{
  "notifications": {
    "verbose": true
  },
  "autoload": {
    "enabled": true,
    "output_path": "/data/ps5_autoloader/autoload.txt",
    "self_autoload": true,
    "custom_entries": []
  },
  "backup": {
    "enabled": true,
    "max_backups": 5
  },
  "repos": [
    {
      "release_url": "https://github.com/drakmor/ShadowMountPlus/releases",
      "formats": "all",
      "dest_dir": "/data/ps5_autoloader",
      "asset_rule": "re:^ShadowMountPlus_(v)?[0-9][0-9A-Za-z._-]*\\.zip$",
      "autoload_preview_name": "ShadowMountPlus.zip",
      "autoload": true,
      "delay_before_ms": 0
    },
    {
      "release_url": "https://github.com/seregonwar/zftpd/releases",
      "formats": "all",
      "dest_dir": "/data/ps5_autoloader",
      "asset_rule": "re:^zftpd-ps5-(v)?[0-9][0-9A-Za-z._-]*\\.bin$",
      "autoload_preview_name": "zftpd-ps5.bin",
      "autoload": true,
      "delay_before_ms": 1000
    },
    {
      "release_url": "https://github.com/ps5-payload-dev/klogsrv",
      "formats": "all",
      "dest_dir": "/data/ps5_autoloader",
      "asset_rule": "exact:klogsrv-ps5.elf",
      "autoload_preview_name": "klogsrv-ps5.elf",
      "autoload": true,
      "delay_before_ms": 2000
    }
  ]
}
```

When `self_autoload` is enabled, ChronicLoader appends itself as the final entry in `autoload.txt` with an 8-second delay. It discovers its own filename using `/proc/curproc/file` so it works regardless of what the ELF is named. The generated `autoload.txt` for the example above would look like:

```
ShadowMountPlus.zip
!1000
zftpd-ps5.bin
!2000
klogsrv-ps5.elf
!8000
chronicloader.elf
```



## The Log File

ChronicLoader writes a log file here:

`/data/chronicloader/chronic.log`

This log is useful for checking:

- What repo was checked
- Which release tag was used
- Which original asset name was downloaded
- What local filename it was saved as
- The SHA-256 of the downloaded file

That makes it easier to tell if a file was updated, skipped because it was already current, or replaced because the local copy no longer matched the last known hash.


## Typical Workflow

1. Build or update your settings JSON
2. Save it to `/data/chronicloader/chronicloaderSettings.json`
3. Run ChronicLoader
4. Let it pull the newest release files
5. Leave your autoloader setup pointed at the stable output filenames in `/data/ps5_autoloader`

## Backups

When backups are enabled, ChronicLoader creates a ZIP archive of the entire autoloader folder before it starts downloading or updating anything. This means you always have a snapshot of what was working before each run.

Backup files are saved to `/data/chronicloader/chronicBackups/` with timestamped names like `autoloader_20260407_143022.zip`. The `max_backups` setting controls how many of these are kept. When the limit is exceeded, the oldest backups are automatically deleted.

If an update breaks something, you can extract the most recent backup ZIP to restore your previous payload files.
