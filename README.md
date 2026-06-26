<p align="center">
  <img src="docs/stayawake.png" alt="stayawake" width="200">
</p>

# kstayawake

Temporarily stop KDE/PowerDevil from sleeping on idle or lid-close for *N*
minutes, then restore normal behavior and — if the lid is still shut — suspend.

Lets a long job finish with the lid closed without leaving the machine awake
forever. Entirely user-space: **no sudo**.

## Usage

```
kstayawake N        arm / re-arm: no idle+lid sleep for N minutes. Re-running
                    resets the countdown to N (20 left + `kstayawake 15` -> 15)
kstayawake off      restore normal settings now; suspend if the lid is closed
kstayawake status   show whether armed and the time remaining
kstayawake --help   show this help
```

`stop` is an alias for `off`.

## How it works

PowerDevil owns lid/idle handling on this host (logind delegates via a block
inhibitor), so the tool flips `LidAction` and `AutoSuspendAction` to `0`
(NoAction) under the `[AC]` and `[Battery]` profiles in
`~/.config/powerdevilrc`, then reloads PowerDevil over D-Bus
(`qdbus-qt6 … refreshStatus`). The **LowBattery** profile is left untouched, so
a dying unattended laptop still suspends.

- The original settings are snapshotted once on first arm (re-arming never
  clobbers them) so `off`/timeout restores the exact prior values.
- The countdown runs in a transient `systemd --user` unit
  (`kstayawake.service`) watching a deadline file under
  `$XDG_RUNTIME_DIR/kstayawake/`.
- Suspend uses `systemctl suspend` (polkit `allow_active=yes` → no password).
- On logout the unit gets SIGTERM; its trap restores settings but does **not**
  suspend. Only a real timeout, or explicit `off`, suspends.

After writing to `powerdevilrc`, the tool reloads and confirms the live lid
action settled to the wanted value, re-issuing the write on mismatch — a guard
against KConfig's mtime-grain change detection missing a fast edit.

### Test hooks

- `KSTAYAWAKE_DRYRUN=1` — log "would suspend" instead of suspending.
- `KSTAYAWAKE_FORCE_LID_CLOSED=1` — make the lid-closed check always true.

## Install

Symlink the script onto your `$PATH` and the completion into a directory on
your zsh `$fpath`:

```sh
ln -s ~/kstayawake/bin/kstayawake          ~/bin/kstayawake
ln -s ~/kstayawake/completions/_kstayawake ~/.zsh/completion/_kstayawake
```

Then restart zsh (or `rm -f ~/.zcompdump*; compinit`) to pick up completions.

## Requirements

- KDE Plasma with PowerDevil
- `kreadconfig6` / `kwriteconfig6` (KDE Frameworks 6)
- `qdbus-qt6`
- `systemd` user instance
