# ChronicLoader

ChronicLoader is a PS5 payload updater and `autoload.txt` manager.

It checks the latest GitHub releases for the repos you configure, downloads the payload files you want, and saves them with stable filenames so your autoloader setup does not need to keep changing.

It can also manage `autoload.txt`, keep a download log, and track SHA-256 hashes so it can tell the difference between a real update, an unchanged file, and a damaged local copy.

ChronicLoader can:

- Download direct release assets such as `.elf` and `.bin`
- Download `.zip` release assets and pull payload files out of them (i.e. shadowmountplus)
- Save files into `/data/ps5_autoloader` or another folder you choose
- Maintain `autoload.txt` with optional delays between payloads
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

## How It Works

When ChronicLoader runs, it:

1. Loads `/data/chronicloader/chronicloaderSettings.json`
2. Checks the latest release for each configured GitHub repo
3. Finds the matching asset or assets
4. Downloads them to the PS5
5. Renames them to stable output names when needed
6. Compares SHA-256 values so unchanged files do not get downloaded again unless something is missing or damaged
7. Writes log and state entries with the source asset name, release tag, destination path, size, and SHA-256

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

The file has three main parts:

- `notifications`
  - Controls how chatty the payload is while it runs
  - `verbose: false` keeps notifications minimal
  - `verbose: true` shows friendlier progress notifications such as starting a download or extracting a zip
- `autoload`
  - Controls whether ChronicLoader also maintains `autoload.txt`
  - `enabled` turns autoload maintenance on or off
  - `output_path` is where the generated `autoload.txt` will be written
  - `custom_entries` lets you preserve extra autoload lines that are not tied to a tracked repo entry
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
    "verbose": false
  },
  "autoload": {
    "enabled": true,
    "output_path": "/data/ps5_autoloader/autoload.txt",
    "custom_entries": []
  },
  "repos": [
    {
      "release_url": "https://github.com/seregonwar/zftpd/releases",
      "formats": "all",
      "dest_dir": "/data/ps5_autoloader",
      "asset_rule": "re:^zftpd-ps5-(v)?[0-9][0-9A-Za-z._-]*\\.elf$",
      "autoload_preview_name": "zftpd-ps5.elf",
      "autoload": true,
      "delay_before_ms": 0
    },
    {
      "release_url": "https://github.com/drakmor/ShadowMountPlus/releases",
      "formats": "elf",
      "dest_dir": "/data/ps5_autoloader",
      "asset_rule": "re:^ShadowMountPlus_(v)?[0-9][0-9A-Za-z._-]*\\.zip$",
      "autoload_preview_name": "shadowmountplus.elf",
      "autoload": true,
      "delay_before_ms": 5000
    }
  ]
}
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

## Troubleshooting

If nothing downloads:

- Make sure the PS5 has network access
- Make sure the repo URL is a real GitHub releases page
- Check the live klog output
- Check `/data/chronicloader/chronic.log`
- Make sure the asset family you selected actually matches the latest release

If a zip repo downloads but nothing is extracted:

- Check whether the payload file inside the zip is really an `.elf` or `.bin`
- Rebuild the config in the HTML builder and inspect the latest release again
