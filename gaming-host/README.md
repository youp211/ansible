# gaming-host — Linux gaming & game-streaming tuning

Ansible configuration for a Linux desktop that plays games locally and streams
them to [Moonlight](https://moonlight-stream.org/) clients over
[Sunshine](https://github.com/LizardByte/Sunshine).

**Target:** LMDE 7 / Debian 13, Cinnamon on Xorg, Intel CPU (`intel_pstate`
HWP), AMD GPU (`amdgpu`), Steam + Proton. Most of it applies to any Debian
derivative; the AMD/VAAPI and `intel_pstate` bits are the parts to check first
on other hardware.

## Layout

| Path | What it is |
|---|---|
| `ansible/` | the reproducible configuration — the source of truth |
| `PERFORMANCE-NOTES.md` | what was measured, what won, and what turned out to be wrong |
| `TUNING-COMMANDS.md` | diagnostic and tuning commands by subsystem, tagged read-only / changes-state / disruptive |

## Running it

Set `target_user` in `ansible/inventory.ini` first, then:

```bash
cd ansible
ansible-playbook site.yml --ask-become-pass --check   # dry run, changes nothing
ansible-playbook site.yml --ask-become-pass           # apply
```

The default run is safe during a live stream. A `--check` run afterwards must
report `changed=0`.

| Role | Does |
|---|---|
| `packages` | gamemode, mangohud, linux-cpupower, lm-sensors, vainfo, stress-ng |
| `kernel_tuning` | `/etc/sysctl.d/99-gaming.conf`, negative-nice rlimit |
| `cpu_power` | `hwp_dynamic_boost` unit, `powersave` baseline governor |
| `gamemode` | `/etc/gamemode.ini`, **polkit authorization**, enables `gamemoded` |
| `sunshine` | encoder/capture config, per-stream display switching, autostart |
| `steam_launch_options` | wraps installed games in `gamemoderun` |

Opt-in, excluded from the default run because each replaces something working:

```bash
ansible-playbook site.yml --tags mesa             # Mesa -> backports
ansible-playbook site.yml --tags sunshine_native  # rebuild + reinstall Sunshine
```

## How the tuning is meant to work

The CPU idles on **`powersave`** with `hwp_dynamic_boost`. **GameMode** raises
it to `performance` for the lifetime of a game and reverts on exit. Pinning
`performance` globally costs ~26 W at idle for nothing; under load the
difference is only ~5 W. Per-game gets the clocks where they matter and the
efficiency everywhere else.

When a client connects, Sunshine's `global_prep_cmd` **drops the host display
to whatever resolution the client asked for** and restores the desktop mode
afterwards. On a 4K144 host serving a 1080p60 client this was by far the
largest single win — see `PERFORMANCE-NOTES.md`.

## Gotchas worth knowing before touching this

- **`capture = x11` is deliberate.** KMS capture is lower latency, but under
  Xorg the cursor is on a separate DRM plane and never reaches the stream.
  Revisit on Wayland.
- **Cinnamon never activates `graphical-session.target`,** so a systemd user
  unit that is `WantedBy` it stays dead forever while reporting `enabled`.
  Sunshine autostarts from an XDG desktop entry instead.
- **Steam rewrites `localconfig.vdf` on exit** — close it before editing launch
  options or the change vanishes.
- **`gamemoderun` is applied to every installed game, anti-cheat titles
  included.** It works via `LD_PRELOAD`; clear a game's Launch Options in Steam
  to exclude it.
- **Fan control is often BIOS-only.** Many boards expose no CPU/case fan PWM to
  Linux; check `hwmon` before planning around fan curves.

## Verifying, rather than assuming

Several things here fail *silently* — reporting healthy while doing nothing.
After any change:

```bash
gamemoded -t                         # GameMode self-test; -s while a game runs
vainfo | grep "Driver version"       # the driver actually loaded in this session
grep "Screencasting with" ~/.config/sunshine/sunshine.log
systemctl --user is-active <unit>    # is-enabled is NOT is-active
ulimit -e                            # 39 == nice -19 permitted
cd ansible && ansible-playbook site.yml --check   # must be changed=0
```

`TUNING-COMMANDS.md` §"The habits that actually caught things" lists the
specific failure each of these caught.
