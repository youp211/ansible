# TUNING-COMMANDS.md — diagnostic and tuning command reference

Every command used to diagnose and tune this gaming/streaming host, grouped by
purpose. Findings and decisions live in `PERFORMANCE-NOTES.md`; the reproducible
config lives in `ansible/`.

Commands marked **[read-only]** are safe to run any time. Commands marked
**[changes state]** modify the system. Commands marked **[disruptive]** will
drop a live stream or interrupt the desktop.

---

## 1. System identification

```bash
uname -a                                  # kernel version
cat /etc/os-release                       # distro
sudo dmidecode -t baseboard               # motherboard model
nproc --all                               # logical CPU count
lspci -nnk | grep -A2 -Ei "vga|3d"        # GPU + which kernel driver is bound
```
**[read-only]** — `dmidecode` identifies the board, which is what tells you
whether fan control is exposed to Linux at all or is BIOS-only.

## 2. SELinux

```bash
getenforce                                # quick mode check
sestatus                                  # full status incl. policy name
sudo ausearch -m avc -ts recent           # recent AVC denials
dmesg | grep -i avc                       # denials if auditd is absent
```
**[read-only]** — `ausearch` is the one that matters; `dmesg`/`journalctl` come
back empty when auditd owns the log.

## 3. Thermals, power and clocks

```bash
sensors                                   # all temps, fans, GPU power (PPT)
sensors | grep -E "Package id 0|^Core "   # CPU package + per-core
sensors | grep -E "edge:|junction:|PPT"   # GPU temps + power draw

# Average clock across all threads
awk '/MHz/{s+=$4;n++} END{printf "%.0f MHz\n", s/n}' /proc/cpuinfo

# CPU package power draw via RAPL (10s average, in watts)
RAPL=/sys/class/powercap/intel-rapl:0:0/energy_uj
a=$(sudo cat $RAPL); sleep 10; b=$(sudo cat $RAPL)
echo "scale=1; ($b - $a) / 10 / 1000000" | bc
```
**[read-only]** — RAPL is how the governor question was settled. But a raw
before/after is worthless unless the **workload is pinned**: the first attempt
here compared `performance` with a game running against `powersave` after the
game was quit, and produced a meaningless 4.4x "difference". Use a synthetic
load so both samples see identical work:

```bash
sudo apt-get install -y --no-install-recommends stress-ng
stress-ng --cpu 4 --timeout 25s &         # fixed, reproducible load
sleep 8                                    # let clocks settle before sampling
# ...then take the RAPL measurement above
```
Controlled result: idle 4.0 W (powersave) vs 30.0 W (performance);
under load 106.5 W vs 111.3 W. The cost of `performance` is idle waste.

Requires `lm-sensors`. GPU fan/temps come from the `amdgpu` hwmon node.

## 4. Fan control (what is and isn't possible here)

```bash
# Which hwmon nodes expose fan/pwm controls at all
for h in /sys/class/hwmon/hwmon*; do
  echo "-- $h ($(cat $h/name)) --"; ls $h | grep -E "^(fan|pwm)"
done
lsmod | grep -Ei "it87|nct6|asus_wmi"     # Super I/O drivers (none bound here)
```
**[read-only]** — result: only the GPU fan is controllable from Linux. The
CPU/case fans need BIOS Q-Fan.

## 5. CPU frequency scaling

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver            # intel_pstate
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor | sort | uniq -c
ls /sys/devices/system/cpu/intel_pstate/                           # HWP tunables
```
**[read-only]** — only `performance` and `powersave` exist under intel_pstate
HWP. There is no `schedutil` on this machine.

```bash
# Set governor on all cores
for f in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo powersave | sudo tee "$f" >/dev/null
done

# Boost frequency on idle-wakeup (only meaningful under powersave;
# a no-op under performance, which is already pinned at max)
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/hwp_dynamic_boost
```
**[changes state]** — runtime only, resets on reboot. Made persistent by
`gaming-cpu-tuning.service` (see `ansible/roles/cpu_power/`).

## 6. Memory, disk, load

```bash
free -h
uptime                                    # load average
ps -eo pid,cmd,%cpu,%mem --sort=-%cpu | head
top -bn1 -p $(pgrep -x sunshine)          # single process, non-interactive
iostat -x 1 2                             # disk utilization (sysstat)
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10              # [changes state] runtime only
```
Persisted via `/etc/sysctl.d/99-gaming.conf`.

## 7. GPU utilization (AMD)

```bash
cat /sys/class/drm/card*/device/gpu_busy_percent     # % busy
cat /sys/class/drm/card*/device/mem_info_vram_used   # VRAM bytes
cat /sys/class/drm/card*/device/power_dpm_force_performance_level
```
**[read-only]** — there is no `nvidia-smi` equivalent; these sysfs nodes plus
`sensors` (for PPT watts) are the AMD equivalents.

## 8. Hardware video encode (VAAPI)

```bash
vainfo                                    # driver version + supported profiles
vainfo | grep "Driver version"
vainfo | grep EncSlice                    # ENCODE support specifically
```
**[read-only]** — `VAEntrypointEncSlice` is the entry that matters; `VLD` is
decode only. This is how the Mesa upgrade was verified not to break encoding.

## 9. Network / Wi-Fi quality

```bash
ip route                                  # find the default interface
ip -s link show wlp10s0                   # errors / dropped counters
iw dev wlp10s0 link                       # SSID, signal, negotiated bitrate
iw dev wlp10s0 station dump | grep -iE "signal|retries|failed|beacon loss"
tc qdisc show dev wlp10s0                 # queueing discipline (bufferbloat)
ss -tuln | grep -E ":(47984|47989|47990|48010)"   # Sunshine listening ports
```
**[read-only]** — `station dump` is the important one: tx retries / tx failed /
beacon loss are what actually reveal a bad link. All were zero here.

## 10. Display modes

```bash
export DISPLAY=:0 XAUTHORITY=$HOME/.Xauthority
xrandr                                    # outputs, modes, current (*)
xrandr --verbose                          # exact modelines and refresh rates
xrandr --listmonitors

# Switch mode  [changes state]
xrandr --output DisplayPort-1 --mode 1920x1080 --rate 60
xrandr --output DisplayPort-1 --mode 3840x2160 --rate 144
```
Automated per-stream by `/usr/local/bin/sunshine-display-mode`, wired into
Sunshine's `global_prep_cmd`. **This was the single biggest win.**

It matches **both** resolution and refresh to what the client requests
(`SUNSHINE_CLIENT_WIDTH` / `_HEIGHT` / `_FPS`). Since `xrandr --rate` only
accepts rates the panel advertises, it picks the *closest advertised rate that
still covers the request*, falling back to the highest the mode offers:

| Client fps | Chosen (panel has 143.98 / 60.00 / 59.94) |
|---|---|
| 24 / 30 / 50 | 59.94 (nothing lower exists) |
| 60 | **60.00** — exact, not 59.94, which would drift a frame every ~16 s |
| 75 / 90 / 120 / 144 | 143.98 |
| 240 | 143.98 (highest available) |

An unknown resolution, or a missing client value, is a no-op rather than an
error, so a bad request can never strand the desktop in an unusable mode.

## 11. GameMode

```bash
gamemoded -t                              # SELF-TEST - run this, do not assume
gamemoded -s                              # is it currently active?
systemctl --user status gamemoded
journalctl --user -u gamemoded -n 30      # pkexec "Not authorized" errors show here
grep -oP '(?<=<action id=")[^"]+' \
  /usr/share/polkit-1/actions/com.feralinteractive.GameMode.policy
```
**[read-only]** — `gamemoded -t` is essential. GameMode reports itself healthy
while being completely inert if polkit denies its helpers.

To actually engage it, a game must request it — Steam launch options:
```
gamemoderun %command%
```

### Applying it to every installed game at once

Steam has no global launch-options setting, so this edits
`~/.local/share/Steam/userdata/<id>/config/localconfig.vdf` directly.

**Steam must be closed** — it rewrites this file from memory on exit, so any
edit made while it is running is silently discarded:
```bash
steam -shutdown            # graceful; wait for the process to actually exit
until ! pgrep -x steam >/dev/null; do sleep 1; done
cp -a localconfig.vdf localconfig.vdf.bak.$(date +%s)
```

**Scope the edit to the right section.** App IDs appear in at least five
places in this file (`Software/Valve/Steam/apps`, `depots`, `friends`,
`WebStorage/apps`, `apps`). A naive "find the appid block" pass hit 39 blocks
for 22 games and had to be rolled back. Only
`UserLocalConfigStore/Software/Valve/Steam/apps` is correct — track the
section path while scanning, do not match on the app ID alone.

Verify before restarting Steam:
```bash
# brace balance must be 0, and line delta must equal the number of games
python3 -c "t=open('localconfig.vdf').read(); print(t.count('{')-t.count('}'))"
grep -c gamemoderun localconfig.vdf
```
Then restart Steam and re-count — if the number drops, Steam rejected the
file or was still running during the edit.

This is automated and idempotent via
`/usr/local/bin/steam-gamemoderun` (`ansible/roles/steam_launch_options/`):

```bash
steam-gamemoderun --check     # report what would change, write nothing
steam-gamemoderun             # apply; refuses to write while Steam is running
```
It skips Proton builds and Steam Linux Runtimes (installed "apps" that are
tools, not games), backs up localconfig.vdf, and verifies brace balance before
writing.

**Anti-cheat caveat:** `gamemoderun` works via `LD_PRELOAD`. That is fine for
the overwhelming majority of titles and there are no confirmed GameMode bans,
but kernel-level anti-cheat (BattlEye, EAC) can treat preloading as
suspicious. Applied here to all 22 installed games as an explicit user
decision, including DayZ, Dead by Daylight, Elden Ring Nightreign, CS2,
Rocket League, PoE 2 and Phasmophobia. To exclude a title later, clear its
Launch Options in Steam's UI (Properties → General).

## 12. Sunshine (streaming host)

```bash
pgrep -af "^/usr/bin/sunshine"
tail -f ~/.config/sunshine/sunshine.log
grep -E "Screencasting with" ~/.config/sunshine/sunshine.log   # KMS vs X11
grep -E "Creating encoder" ~/.config/sunshine/sunshine.log     # which encoder
grep -E "CLIENT CONNECTED|CLIENT DISCONNECTED" ~/.config/sunshine/sunshine.log
getcap /usr/bin/sunshine                  # cap_sys_admin for KMS capture

# Paired clients
python3 -c "import json;print([x['name'] for x in
  json.load(open('$HOME/.config/sunshine/sunshine_state.json'))
  ['root']['named_devices']])"
```

Restart **[disruptive]** — drops a live stream:
```bash
pkill -x sunshine
DISPLAY=:0 XAUTHORITY=$HOME/.Xauthority \
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus XDG_RUNTIME_DIR=/run/user/1000 \
  setsid nohup /usr/bin/sunshine >/dev/null 2>&1 </dev/null &
```
The environment variables are mandatory when launching over SSH — without them
every capture backend fails.

## 13. Virtual input devices (mouse/gamepad)

```bash
ls -la /dev/uinput                        # "+" means the uaccess ACL applied
cat /usr/lib/udev/rules.d/60-sunshine.rules
grep -A4 -i libvirtualhid /proc/bus/input/devices

# Which pointer is actually receiving events (move the mouse while this runs)
sudo timeout 6 cat /dev/input/event24 | wc -c     # libvirtualhid Mouse (relative)
sudo timeout 6 cat /dev/input/event26 | wc -c     # Pen Tablet (absolute)
```
**[read-only]** — event numbers shift between runs; get them from
`/proc/bus/input/devices` rather than hardcoding. Traffic on the relative
device means the client is in relative mouse mode.

## 14. Packages and versions

```bash
apt-cache policy <pkg>                    # installed vs candidate
apt-cache madison <pkg>                   # all available versions
apt-get install -s -t trixie-backports <pkg>   # SIMULATE - shows removals
dpkg -l <pkg> | grep ^ii
dpkg -L <pkg> | grep -E "\.desktop$|udev|systemd"   # what a package ships
dpkg-deb -I file.deb                      # inspect a built .deb's metadata
dpkg-deb -c file.deb                      # list its contents
```
**[read-only]** — `apt-get install -s` (simulate) is the important habit: it is
what revealed that the Mesa upgrade would remove `mesa-va-drivers`, prompting
the check that `mesa-libgallium` now `Provides` it.

## 15. Compiling Sunshine to a .deb

```bash
# Dependencies. systemd-dev is easy to miss and NOT the same as libsystemd-dev:
# without it the build silently omits the uinput udev rule and gamepads break.
sudo apt-get install -y --no-install-recommends \
  build-essential cmake ninja-build git pkg-config nodejs npm \
  libboost-{filesystem,locale,log,program-options}-dev \
  libavdevice-dev libcap-dev libcurl4-openssl-dev libdrm-dev libevdev-dev \
  libminiupnpc-dev libnotify-dev libnuma-dev libopus-dev libpipewire-0.3-dev \
  libpulse-dev libssl-dev libva-dev libvdpau-dev libvulkan-dev libwayland-dev \
  libx11-dev libxcb-shm0-dev libxcb-xfixes0-dev libxcb1-dev libxfixes-dev \
  libxrandr-dev libxtst-dev glslang-tools systemd-dev

# Mesa dev headers must match the backports runtime or apt downgrades libgbm1
sudo apt-get install -y -t trixie-backports libgbm-dev

git clone --depth 1 --recursive https://github.com/LizardByte/Sunshine.git
cd Sunshine
cmake -G Ninja -B build -S . \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr \
  -DSUNSHINE_ASSETS_DIR=share/sunshine \
  -DSUNSHINE_ENABLE_{WAYLAND,X11,DRM,VAAPI}=ON \
  -DSUNSHINE_ENABLE_CUDA=OFF -DSUNSHINE_ENABLE_TRAY=OFF \
  -DBUILD_TESTS=OFF -DBUILD_DOCS=OFF
nice -n 19 ninja -C build                 # niced so it cannot starve a game
cd build && cpack -G DEB                  # -> build/cpack_artifacts/Sunshine.deb
sudo apt-get install -y ./cpack_artifacts/Sunshine.deb
sudo setcap cap_sys_admin+p /usr/bin/sunshine      # required for KMS capture
```
**[changes state]** — `TRAY=OFF` avoids a full Qt dependency; `DOCS=OFF` avoids
Doxygen ≥1.10. Always `dpkg-deb -c` the result before installing.

## 16. Ansible

```bash
cd gaming-host/ansible
ansible-playbook site.yml --check          # dry run, changes nothing
ansible-playbook site.yml --check --diff   # dry run showing file diffs
ansible-playbook site.yml                  # apply
ansible-playbook site.yml --tags gamemode  # one role only
ansible-playbook site.yml --tags steam     # Steam launch options only

# Opt-in roles (excluded from the default run)
ansible-playbook site.yml --tags mesa
ansible-playbook site.yml --tags sunshine_native   # [disruptive] rebuilds/replaces

ansible-lint                               # must pass at "production" profile
yamllint -d "{extends: relaxed, rules: {line-length: {max: 120}}}" .
```
Re-running `--check` after an apply must report `changed=0`. A task that keeps
reporting `changed` usually means two tasks are fighting over the same file.

## 17. Systemd / session / desktop autostart

```bash
systemctl --user is-enabled <unit>         # NOT the same as is-active
systemctl --user is-active <unit>
systemctl --user is-active graphical-session.target   # inactive under Cinnamon
loginctl list-sessions
loginctl show-session <id> -p Type -p Active
ulimit -e                                  # RLIMIT_NICE (39 == nice -19 allowed)
sudo grep -i "nice priority" /proc/<pid>/limits
desktop-file-validate ~/.config/autostart/foo.desktop
```
**[read-only]** — `is-enabled` returning `enabled` does not mean a unit ran.
That distinction is exactly how the Sunshine unit stayed dead after a reboot.

To recover the desktop session's environment from an SSH shell:
```bash
CINPID=$(pgrep -o -u "$USER" cinnamon)
sudo tr '\0' '\n' < /proc/$CINPID/environ | grep -E "^(DISPLAY|XAUTHORITY|DBUS_|XDG_)"
```

---

## Programs installed during this work

| Package | Why |
|---|---|
| `gamemode` | per-game governor/nice/ioprio switching (was present, unused) |
| `mangohud` | in-game frame-time overlay |
| `linux-cpupower` | CPU frequency inspection |
| `lm-sensors` | temperatures, fan RPM, GPU power |
| `vainfo` | VAAPI encode/decode verification |
| `ansible-lint`, `yamllint` | playbook linting |
| Mesa 26.1.2 (backports) | RDNA3 driver improvements |
| `sunshine` (compiled) | native build, replaces the flatpak |
| Sunshine build deps | see section 15 |

## The habits that actually caught things

1. **Run the tool's own self-test.** `gamemoded -t` found a silent polkit
   denial that every other signal reported as healthy.
2. **`is-enabled` is not `is-active`,** and neither survives a reboot without
   checking again.
3. **Simulate package changes first** (`apt-get install -s`) — it surfaces
   removals before they happen.
4. **Inspect a built package before installing it** (`dpkg-deb -c`) — this is
   how the missing udev rule was caught.
5. **Measure, do not infer.** "Streaming 4K" was inferred from a capture
   resolution and was wrong; Wi-Fi was blamed for latency and was flawless;
   cursor lag was blamed on capture and was a client FPS setting.
6. **State the controlled variable before measuring, and verify it held
   afterward.** The first governor power comparison was taken with a game
   running on one side and not the other, producing a confident, wrong 4.4x
   result. Both numbers were real — the world moved between them. Note that
   habit 5 ("measure, do not infer") does *not* catch this: nothing was
   inferred, the samples were just not comparable.

   The fix is mechanical, not a resolution to be careful. Before: write down
   what is being held constant. After: check it actually was. Here that was
   one `pgrep -x <game>.exe` on each side. Prefer a synthetic load
   (`stress-ng`) so the constant is something you control rather than
   something you hope stayed still.

   The warning sign was in the output and got read past: 1396 MHz average
   under `powersave` is idle clocking, not a machine running a game. A number
   that points the direction you already expect deserves *more* scrutiny,
   not less.
