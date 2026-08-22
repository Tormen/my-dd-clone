# my-dd-clone

Block-clones the APFS physical store backing one volume onto the one backing
another (`dd` over the raw `/dev/rdiskXsY` devices), or into a raw image
file. The copy runs detached via `nohup` and survives closing the terminal.

## Usage

```text
my-dd-clone [-V] [-D] [-E] [--max] [--resume] [--config FILE] <SOURCE> <TARGET> [go]
my-dd-clone [-V] --diff <SOURCE> <TARGET> [go]
my-dd-clone [-V] [--clear]
my-dd-clone -A|abort [JOB [TARGET]] [go]
my-dd-clone --create-config [FILE]
```

`SOURCE`: an APFS volume, as `/Volumes/<NAME>` or `<NAME>`.
`TARGET`: another APFS volume (same forms), or a raw image file — any path
ending in `.img` or `.dmg`. `.img` is the truthful extension (dd writes raw
bytes, not a UDIF image); attach later with
`hdiutil attach -imagekey diskimage-class=CRawDiskImage FILE`.

## How it works

- **`my-dd-clone <SOURCE> <TARGET>`** — analyze only: resolves each volume via
  `diskutil info` (Volume → APFS Container → APFS Physical Store), prints
  devices and exact sizes, and says what `go` would do. No changes.
- **`… go`** — the only word that starts it. Validates sudo up front
  (`sudo -v`, so the detached run never prompts), unmounts **both whole
  containers** (`diskutil unmountDisk`; source only, for a file target),
  then launches `dd if=/dev/r… of=… bs=$DD_BS status=progress` under
  `nohup`, logging to a file and recording a job file (names, devices,
  size, pid, log, start time) in `STATE_DIR`. A file target is chown'd
  back to the invoking user when done.
- **`-E` / `--emergency`** (dying-disk mode — the most precise and
  fail-safe choice for a failing source; plain dd stops at the first read
  error, and analyze reminds you of this) — GNU `ddrescue` replaces dd:
  good sectors are read exactly **once**, then only the bad areas get
  trim/scrape plus `DDRESCUE_RETRIES` retry passes — minimal stress on a
  failing source. Log and resume mapfile land under `EMERGENCY_LOG_DIR`
  (default `/var/log/mine/$(id -un)/my-dd-clone`, sudo-created and chown'd
  to you if missing). Needs `brew install ddrescue`.
- **Resume** — an unfinished run of the same SOURCE → TARGET pair makes
  `go` ask: type `resume` to continue (dd: skip/seek to the recorded
  offset minus a 16-block safety rewind; `-E`: reuse the mapfile, so only
  unread/bad areas are touched) or `new` to start over. `--resume` skips
  the question (for scripts). Works for disk→disk and disk→image alike.
- **`--max`** — a clone onto a larger volume target leaves the copied APFS
  container at the source's size; `--max` automatically grows it to fill
  the partition after a successful copy, escalating as needed:
  `diskutil apfs resizeContainer <targetStore> 0` with settle+retries,
  then `repairDisk` + retry, then an explicit grow to 128 MB below the
  partition size (diskutil refuses smaller deltas on a cloned store). All
  attempts land in the job log. Without `--max`, the analyze warning
  points you here. Volume targets only.
- **`--diff`** — compares instead of cloning: unmounts both sides, reads
  each exactly once, sha256s the first source-length bytes, and reports
  IDENTICAL/DIFFER in the status view. Volume or existing-image target.
- **`my-dd-clone`** (no arguments) — the status view when job records
  exist, otherwise a short usage. Status shows RUNNING / DONE /
  FAILED(exit N) / ABORTED / DIED with progress for both kinds: dd jobs
  get bytes done, percent, rate, ETA (parsed from dd's progress lines in
  the log); `-E` jobs get rescued bytes and percent (from the ddrescue
  log, mapfile position as fallback), plus elapsed and the log/mapfile
  paths. `--clear` drops finished/failed records. A DONE volume-target
  clone stays listed as a reminder to unplug+replug the target (see
  Caveat), and — when the source is a Time Machine destination — to then
  forget + re-add it in Time Machine settings; the record auto-clears
  once each step is detected (the re-add via the destination ID changing).
- **`-A` / `abort`** — analyze-by-default too: lists the running clones
  that match (all, by pid, or by SOURCE [TARGET]); appending `go` aborts
  them. The copy process is TERMinated first; the wrapper records exit
  code and end timestamp itself (no race), and a separate marker file
  flags the job ABORTED. Rerunning the pair with `go` then offers to
  resume.
- **Accounting** — start/end timestamps are recorded, and the copy runs
  under `/usr/bin/time -p`; finished jobs show elapsed plus
  real/user/sys CPU seconds in the status view.
- **Audit trail** — the job log records every command the job issues
  (` >>> ` lines): the unmounts, the exact dd/ddrescue/diff command,
  every `--max` resize attempt, and the final chown — the log alone
  tells the whole story of what was done to which device.

## Safety checks (before anything runs)

- refuses same physical store, target smaller than source, target on the
  boot disk, and a second clone touching a device a running job already uses
- file target: refuses an existing file, insufficient free space, and an
  image path on a volume of the source container (which `go` unmounts)
- warns when source and target share one physical disk, and when a larger
  target's tail beyond the source size will be left untouched
- safety net: every analyze/go re-verifies both devices against a fresh
  `diskutil list` — they must exist as Apple_APFS stores with the SIZE
  each resolved to, the source container↔store mapping must hold, and
  duplicate volume names (a partial clone's twin) are surfaced; on resume,
  the devices recorded at job start are trusted over today's name lookup

## Config

Standard search order (`$MY_DD_CLONE_CONFIG`, `--config FILE`,
`/LINKS/default/my-dd-clone.conf`, `~/.my-dd-clone.conf`,
`/etc/my-dd-clone.conf`, `/usr/local/etc/my-dd-clone.conf`); without one it
lists the locations and exits. Create with
`my-dd-clone --create-config /LINKS/default/my-dd-clone.conf`.
Tunables: `DD_BS` (block size, default `4m`, plain dd only — it does NOT
apply to `-E`), `STATE_DIR` (job records + logs) and `EMERGENCY_LOG_DIR`
(`-E` log + mapfile), both defaulting to `/var/log/mine/$(id -un)/my-dd-clone`
— per user under a root-owned parent, so other users can neither tamper
with nor plant job records (which are sourced; only files owned by root
or the invoking user are trusted), sudo-created on first `go` —
`DDRESCUE_RETRIES` (default `3`), and `EMERGENCY_BS` (the `-E` copy unit,
default `64k` — deliberately small so a read error on a dying disk loses
at most one small cluster).

## Caveat

A successful clone is byte-identical — same APFS container/volume UUIDs as
the source. macOS can get confused with both attached; keep only one
connected. When the clone is DONE: detach the source, then **unplug and
replug the target once** before first use — the container attached during
the clone carries stale kernel linkage (no Disk Utility nesting/mounting,
spurious resize refusals) until a fresh physical attach. `diskutil eject`
is not enough, and macOS can no longer randomize APFS UUIDs
(`apfs.util -s` is defunct). If the source is a **Time Machine
destination**, its stale association also survives the replug (the old
free space is shown and backups fail with "destination not available"):
forget + re-add the volume in Time Machine settings — existing backups
are kept, and the host may additionally need
`diskutil enableOwnership /Volumes/<NAME>` if the volume shows
`Owners: Disabled`. The status view keeps the job listed until it
detects both the re-attach and the re-add.
