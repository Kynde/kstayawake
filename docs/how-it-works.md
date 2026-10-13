# How kstayawake works

`kstayawake` is entirely user-space — **no sudo**. On this host PowerDevil owns
lid and idle handling (logind delegates to it via a block inhibitor), so the
tool works by editing PowerDevil's config and reloading it, then arranging for
the change to be undone after a timeout.

## What it changes

While armed, four keys in `~/.config/powerdevilrc` are forced to `0`
(`NoAction`), under the `[SuspendAndShutdown]` group of two profiles:

| Profile   | Key                 | Armed value |
| --------- | ------------------- | ----------- |
| `AC`      | `LidAction`         | `0`         |
| `AC`      | `AutoSuspendAction` | `0`         |
| `Battery` | `LidAction`         | `0`         |
| `Battery` | `AutoSuspendAction` | `0`         |

The **`LowBattery`** profile is deliberately left untouched, so a dying,
unattended laptop still suspends instead of running its battery flat.

Keys are read/written with `kreadconfig6` / `kwriteconfig6` and PowerDevil is
reloaded over D-Bus with `qdbus-qt6 … refreshStatus`.

## Snapshot and restore

On the **first** arm, the current value of each of the four keys is recorded to
a snapshot file (including the "key was absent" case, via a sentinel). Re-arming
never re-snapshots, so it can't clobber the originals. Restoring writes each key
back to its snapshotted value — or deletes it if it wasn't present originally —
so `off`/timeout returns `powerdevilrc` to exactly its prior state.

## The countdown

The countdown runs in a transient `systemd --user` unit, `kstayawake.service`,
started with `systemd-run --user --collect`. It watches a deadline timestamp
file and wakes periodically (every 15 s when there's more than 20 s left, then
sleeps out the remainder) until the deadline passes. Running the daemon as a
user unit — rather than a backgrounded process — means it survives terminal
closes but is still tied to the user session.

Re-running `kstayawake N` just rewrites the deadline file; the existing unit
picks up the new value, so `20m left + kstayawake 15` becomes `15m`.

## Reload and enforce (the KConfig gotcha)

Writing to `powerdevilrc` and *immediately* calling `refreshStatus` can be
missed by KConfig: its change detection has sub-second mtime granularity, so a
fast edit can look unchanged, and PowerDevil then re-persists its stale
in-memory values — making the edit appear to "bounce back".

To guard against this, after each write the tool reloads, reads the **live**
lid action back over D-Bus, and re-issues the write until the live value settles
to what was intended (up to a dozen attempts, then it warns and gives up rather
than hanging).

## Suspend semantics

Suspend is `systemctl suspend` (polkit `allow_active=yes` → no password prompt).
Whether a suspend happens depends on *how* the wake-lock ends:

| Ending                       | Restores settings | Suspends (if lid shut) |
| ---------------------------- | ----------------- | ---------------------- |
| Timeout reached              | yes               | **yes**                |
| `kstayawake off`             | yes               | **yes**                |
| Logout / unit `stop` (TERM)  | yes               | no                     |

The logout case restores via a `SIGTERM` trap but deliberately does **not**
suspend — only a real timeout or an explicit `off` does. "Lid shut" is read from
`/proc/acpi/button/lid/LID/state`.

## State files

All transient state lives under `$XDG_RUNTIME_DIR/kstayawake/` (tmpfs, cleared
on logout):

- `deadline` — Unix timestamp the countdown is counting down to.
- `total` — the armed duration in seconds, used for the progress bar.
- `snapshot` — the original values of the four keys, captured on first arm.

## Test hooks

Two environment variables make the suspend path testable without actually
suspending or closing the lid:

- `KSTAYAWAKE_DRYRUN=1` — log `would suspend` and touch a
  `dryrun-suspended` marker file instead of calling `systemctl suspend`.
- `KSTAYAWAKE_FORCE_LID_CLOSED=1` — make the lid-closed check always return
  true, so the suspend branch is exercised regardless of the real lid state.
