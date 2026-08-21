# ISO Builder GUI

![CI](https://github.com/YOUR_USERNAME/iso-builder-gui/actions/workflows/ci.yml/badge.svg)
![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)

A GTK3 desktop app for building custom Debian/Ubuntu-family live ISOs,
wrapped around Debian's `live-build` toolchain — set up so **you never
redownload packages or re-bootstrap the base system on every build**,
plus project management, a package catalog, disk-space checks, an ISO
writer, and AI-assisted log diagnosis.

## Honesty notes (read this before filing a "why doesn't X work" issue)

- **Build engine scope**: `live-build` (what this app drives) only
  works for Debian-family distros — anything apt-based/debootstrap-
  compatible. Debian, Ubuntu, Kali, Devuan, and Ubuntu-derivatives
  (Mint/elementary/Zorin/Deepin-style builds, built from an Ubuntu or
  Debian base since their own repos/tooling aren't apt-mirror-shaped)
  are genuinely buildable. Arch, Manjaro, Fedora, CentOS Stream,
  openSUSE, and Alpine each need a completely different native
  toolchain (archiso, lorax/livemedia-creator, kiwi, mkimage
  respectively) that this app does not implement — they appear in the
  Distros/Packages tabs for reference and package browsing only, with
  Build ISO disabled for them. See `isobuilder/distros.py`.
- **Package sizes**: the package catalog's sizes are curated estimates
  for labeling/rough total-size math, not live-fetched exact figures —
  real sizes vary by release, mirror, and architecture. Labeled as
  "approx" throughout the UI.

## Features

- **Projects tab** — create/save/load/delete/export projects (each a
  full build config: distro, packages, live user, output dir, etc.).
  Export bundles the project JSON (and its last-built ISO, if present)
  into one `.zip`.
- **Distros tab** — browse the distro catalog; buildable ones (see
  above) can be applied straight to the current project.
- **Packages tab** — curated per-family package catalog grouped by
  category (Desktop Environments, System, Development, Security,
  Multimedia, Utilities/Browsers), checkboxes with a live running
  total size, plus a free-text field for anything not in the catalog.
- **Storage tab** — real-time free-space check on both the build and
  output directories, cache size, a pre-build buildability check (the
  app refuses to start a doomed build instead of failing 20 minutes
  in), one-click package/full cache clearing, and required-tool
  detection + one-click "Install Missing Tools".
- **Etcher tab** — write a finished ISO to a USB drive (or burn to
  CD/DVD). Devices are auto-detected (`lsblk`) and only ever chosen
  from that list — never freehand-typed — with an explicit "this will
  erase everything on X" confirmation before anything happens. Uses
  `ddrescue` when installed (retries on read errors), falling back to
  `dd`; `wodim`/`growisofs` for optical media.
- **AI Assist tab** — three ways to diagnose a build log:
  1. **Local Analyzer** — free, fully offline, no key or network.
     Pattern-matches known `live-build` failure signatures. Runs
     automatically in the Diagnostics block on any failed build, too.
  2. **Anthropic (Claude)** — optional, your own API key.
  3. **OpenAI (GPT)** — optional, your own API key.
  All keys are stored locally only (0600-permission files under
  `~/.local/share/isobuilder/`), sent only to the corresponding
  provider's own API, never bundled or sent anywhere else.
- **Automatic issue fixing** — on a failed build (when Auto-fix is on
  in the project's options), the app inspects what actually went wrong
  and takes a targeted action before retrying: disk-full clears the
  package cache, a GPG/keyring error triggers a keyring refresh, a
  permission error gets a `chmod` pass — then always does a full `lb
  clean` (not just `--binary`, which isn't enough to fix stale-stage
  problems like a missing kernel) and retries once.
- **Guaranteed-ISO export**: after a build, the app fixes permissions
  on the build directory (root-owned artifacts being unreadable by the
  GUI process was the actual cause of most "no .iso produced" reports
  on builds that had genuinely succeeded), copies the ISO out, and
  verifies the copy actually landed and isn't zero bytes before
  calling it a success.
- **Remember privileged session** checkbox — check it once, enter the
  root password once (a proper dialog), and every privileged command
  for the rest of that app session reuses that single authenticated
  shell instead of re-prompting per command. (Only needed for the `su`
  fallback path — see Privilege escalation below.)
- **Dark theme toggle** in the header.
- **Terminal tab** — a real embedded shell (multi-tab), with Copy/Paste
  buttons, right-click menu, and Ctrl+Shift+C / Ctrl+Shift+V.

## Privilege escalation

Build/install/cache/etcher commands need root. The app auto-detects,
per machine:
1. Already root → runs directly.
2. `sudo` works → uses it.
3. Neither → falls back to `pkexec` (graphical prompt) per command, OR
   — if you check **Remember privileged session** — a single `su -`
   login (pty-driven, password entered once in a dialog) reused for
   the whole session.

`install.sh` uses the same three-way detection for its own setup
steps, so it works whether or not your account has sudo rights.

## Why this doesn't redownload things every time

Most "make me an ISO" scripts call `debootstrap` + `apt-get` fresh
each run, so every build re-fetches hundreds of MB. This app instead:

1. Uses `lb config --cache true --cache-packages true --cache-stages
   bootstrap`, so `live-build` keeps every downloaded `.deb` and the
   base bootstrap tarball on disk.
2. Points **every project** at one shared cache directory
   (`~/.local/share/isobuilder/work/cache`) via a symlink, so building
   two different ISOs still reuses whatever packages overlap.
3. Only ever runs the full `lb clean` (which preserves `cache/` by
   design) between builds/retries — never wipes the cache unless you
   explicitly click "Clear cache" in the Storage tab.

## Install (one time)

```bash
git clone <this-repo>   # or unzip the release
cd iso-builder-gui
chmod +x install.sh
./install.sh
```

Installs `live-build`, `debootstrap`, `squashfs-tools`, `xorriso`,
`isolinux`, `syslinux-utils`, `dosfstools`, `ddrescue`, `wodim`, the
GTK3 Python bindings, and the VTE terminal widget, then adds an "ISO
Builder" entry to your application menu. Tested against Debian 12/13
and Ubuntu 22.04/24.04 hosts.

**No sudo access?** `install.sh` auto-detects root/sudo/su the same
way the app does — you don't need to be in `/etc/sudoers` for setup.

## Run

From your app menu ("ISO Builder"), or:

```bash
~/.local/share/isobuilder-app/launch.sh
```

Output (including crashes) is also teed to
`~/.local/share/isobuilder/last-launch.log`.

## Layout on disk

```
~/.local/share/isobuilder/
  projects/<name>.json         # saved project configs
  anthropic_api_key.txt        # optional, 0600
  openai_api_key.txt           # optional, 0600
  crash.log                    # only if the app fails to start
  last-launch.log              # full output of the most recent launch
  work/cache/                  # SHARED across all projects
    packages.binary/           # every downloaded .deb
    bootstrap/                 # cached debootstrap base stage
  work/<project>/              # live-build working directory per project
```

## Troubleshooting

**Missing tools despite installing them** — GUI apps launched from a
desktop menu often get a `$PATH` missing `/usr/sbin`/`/sbin`, where
`debootstrap` and `mkfs.vfat` live. The app checks those directories
explicitly (not just `$PATH`), both for detection and for the actual
privileged commands.

**"Build finished but no .iso was produced"** — almost always
root-owned permissions the GUI couldn't read; fixed by the permission
pass described above. If it still happens, the Diagnostics block (and
the Local Analyzer, which also runs automatically there) will show
free disk space, directory contents, and any error lines from the run.

**`cp: cannot stat 'chroot/boot/vmlinuz-*'`** — means no kernel was
actually installed in the chroot, usually from stale build-stage
bookkeeping after an interrupted earlier run. Auto-fix's full `lb
clean` + retry (see above) resolves this; it's automatic when Auto-fix
is on for the project.

**Not in sudoers** — `install.sh` and the app both auto-detect and
fall back to `su`. Check **Remember privileged session** in the
Projects tab to only be prompted once per app session instead of once
per command.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).




<img width="1366" height="768" 
alt="313685" src="https://github.com/user-attachments/assets/c0c45b3f-6e8b-4927-b603-6e5dd5904a54" />

<img width="1366" height="768" alt="313682" src="https://github.com/user-attachments/assets/259143ec-0c6e-4383-bebe-a5dc5b5aca1f" />
<img width="1366" height="768" alt="313681" src="https://github.com/user-attachments/assets/764dde53-e38c-44e4-a92a-70c6c81497bd" />
<img width="1366" height="768" alt="313680" src="https://github.com/user-attachments/assets/a292f270-42d2-4c76-8db5-2c3ea7f94df4" />
<img width="1366" height="768" alt="313679" src="https://github.com/user-attachments/assets/dc4f45b0-f88f-4e84-a869-276252fb3ee0" />
<img width="1366" height="768" alt="313678" src="https://github.com/user-attachments/assets/18e7e6f2-e10f-4c1e-afab-ff72c62172aa" />
<img width="1366" height="768" alt="313676" src="https://github.com/user-attachments/assets/6558064e-22c2-4bd7-b5bb-1744a4dc5db1" />
<img width="1366" height="768" alt="313677" src="https://github.com/user-attachments/assets/02a23f45-22c4-449c-8af1-d401b9963f24" />
# iso-builder-by-immux
Iso maker tool
