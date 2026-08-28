# PERFORMANCE-NOTES.md — what was measured, and what it changed

Findings from tuning one gaming + streaming host. Kept because the ranking at
the end is the useful part: most of the effort went into things that barely
mattered, and the largest win was a configuration mistake nobody had noticed.

Measurements come from a 28-thread Intel CPU with `intel_pstate` HWP, an AMD
RDNA3 GPU, a 4K144 panel, and a Moonlight client streaming 1080p60.

## The result

Before and after, both samples taken during real gameplay with a client
connected:

| | Before (host rendering 4K144) | After (1080p60 + GameMode) |
|---|---|---|
| **GPU power** | **266 W** (against a 265 W cap) | **102–111 W** |
| GPU busy | 100% | 56–57% |
| GPU edge / junction | 69–70 °C / 79–80 °C | 46–47 °C |
| **CPU package** | **89–92 °C** | **56–59 °C** |
| Game process CPU | ~465% | 139% |
| Sunshine CPU | 30–73% | **10.4%** |

~155 W less GPU power, off the power cap entirely, CPU package down ~33 °C —
and that is *with* `performance` active through GameMode. Temperatures fell
because the workload shrank roughly 4x, not because the CPU was held back.

## What actually mattered, ranked honestly

1. **Per-stream display-mode switching.** The host panel ran 3840x2160@144
   while the client streamed 1080p60 — about **4x the pixels and 2.4x the
   frames**, all discarded before reaching the client, with Sunshine
   downscaling every frame on top. `sunshine-display-mode`, wired into
   `global_prep_cmd`, matches the output to what the client actually asked for
   and restores the desktop mode afterwards. This dwarfs everything else, and
   it also dropped X11 capture cost from 30–73% CPU to ~12%.
2. **GameMode actually being enabled and authorized.** It was installed but had
   never once run: its daemon was disabled, Debian's polkit policy authorized
   nobody, and no game requested it. All three had to be fixed.
3. **Governor / `hwp_dynamic_boost` / swappiness.** Real but modest, and
   largely overshadowed by (1).
4. **Mesa backports.** Plausible RDNA3 gains, not separately measured.

## The governor measurement

Taken with `stress-ng` so both governors saw an identical, reproducible load.
CPU package power via RAPL (`/sys/class/powercap/intel-rapl:0:0/energy_uj`),
10 s averages.

| Load | `powersave` | `performance` | delta |
|---|---|---|---|
| Idle | 4.0 W / 44 °C / 864 MHz | 30.0 W / 47 °C / 2086 MHz | **+26 W** |
| 4 workers | 106.5 W / 82 °C / 2081 MHz | 111.3 W / 84 °C / 2768 MHz | **+4.8 W** |

`performance` costs ~26 W **at idle, for nothing**. Under load the gap collapses
to ~5 W and buys +687 MHz, which is a reasonable trade. The penalty is idle
waste, not load waste — precisely the case `powersave` + GameMode is built for.
**Decision: stay on `powersave`.**

## GameMode, once it was working

With `gamemoderun %command%` set in Steam's launch options, `gamemoded -s`
reports active in a live game and every thread switches to `performance` for the
game's lifetime, reverting on exit. Two non-fatal caveats show up in
`journalctl --user -u gamemoded`:

- **renice and ioprio are not applied.** GameMode refuses to renice a process it
  does not find at nice 0, and Wine/Proton pre-sets its processes to nice -2, so
  GameMode backs off rather than fighting another priority manager. Deliberate,
  and minor — the governor carries nearly all the benefit.
- `Addition requested for already known client [/usr/bin/env]` — `gamemoderun`
  execs through `env`, so GameMode sees the wrapper. Cosmetic.

Editing launch options in bulk means editing `localconfig.vdf`, and two things
bite: app IDs appear in **five** different sections (only
`UserLocalConfigStore/Software/Valve/Steam/apps` is the right one), and Steam
rewrites the file from memory on exit, so it must be closed first. Procedure in
`TUNING-COMMANDS.md` §11.

## Corrections — things asserted here that turned out to be wrong

Recorded deliberately, because each was stated confidently and then acted on.

1. **"Streaming 4K"** — wrong. Inferred from Sunshine's `Streaming display …
   res 3840x2160`, which is the *captured display*, not the stream. The client
   was 1080p60. This inflated the apparent value of every CPU-side optimization.
2. **Wi-Fi blamed as a latency risk, repeatedly.** Wrong. Measured: Wi-Fi 7,
   160 MHz, -47 dBm, 1297 Mbit/s TX, zero retries, failures, drops or errors.
3. **KMS capture presented as "the" win.** Genuinely lower latency, but under
   Xorg the cursor lives on a separate DRM plane and never reaches the stream.
   Sunshine has a guard that avoids KMS under X11 for exactly this reason;
   setting `capture = kms` explicitly overrode it. Self-inflicted.
4. **Cursor lag diagnosed as a capture/input problem.** Wrong — the client's FPS
   setting was very low, so the cursor could only update as fast as frames
   arrived. Host CPU was ~0–12% and the network was clean throughout; the
   evidence never supported a host-side cause.

**A discarded measurement, kept as a warning.** A first governor comparison
reported 93.7 W vs 21.3 W — a 4.4x gap. It was invalid: the game was quit
between samples, so the two runs saw completely different loads. Both numbers
were real; the world moved between them. The tell was in the output and got read
past — 1396 MHz average under `powersave` is idle clocking, not a machine
running a game. Pin the workload (ideally a synthetic one) before comparing
anything, and check afterwards that it held.

**Was the native Sunshine build worth it?** Its headline justification (KMS
capture) is unused. It still earns its place — no flatpak sandbox limits, a
working uinput/udev path for gamepads, `CAP_SYS_NICE`, and KMS available if the
host ever moves to Wayland — but it was not the performance win it was framed
as.

## Open items

- [ ] Confirm games handle a mid-stream resolution switch gracefully rather than
      needing a restart.
- [ ] Test a gamepad over Moonlight. The udev rule is installed, but it was
      missing from the first build, so verify rather than assume.
- [ ] If the desktop cursor ever feels laggy at a sane client FPS: Moonlight's
      "Optimize mouse for remote desktop" switches to absolute positioning so
      the client draws the cursor locally. Relative mode is correct for actual
      gameplay.
