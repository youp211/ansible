# BOOTPOLICY.md — what may boot, and what kernels may arrive

Two independent blocks, both reversible in one variable:

| Knob | Blocks | Owned by |
|---|---|---|
| `uki_boot_policy` | which entries the GRUB menu offers | `roles/uki` (`finalize.yml`, `uki.yml`) |
| `kernel_hold_distro` | which kernel packages apt may install | `roles/kernel` (`kernel.yml`) |

The boot policy stops you *selecting* an unsigned image that already exists; the
apt hold stops new unsigned images *arriving*.

## The problem

`finalize.yml` builds signed UKIs and parks them on the ESP, and `25_uki`
chainloads them. But Debian's `10_linux` keeps emitting a classic
`linux`/`initrd` entry for **every** kernel in `/boot`, plus a recovery entry for
each — and because it sorts before `25_uki`, the unsigned path is also the
default. What a classic entry costs you, relative to the UKI beside it:

- **Not signed.** `sbsign` signs the UKI as one PE binary; a classic entry hands
  firmware nothing to verify.
- **Not measured the same way.** PCR 11 covers the UKI's PE sections. A classic
  boot never extends it, so a signed PCR policy bound to PCR 11 is bypassed
  rather than failed.
- **Cmdline is editable.** GRUB's `e` key edits it in place: drop
  `security=selinux`, add `init=/bin/sh`. A UKI's cmdline lives inside the
  signature.

## `uki_boot_policy`

| Value | GRUB offers | Use when |
|---|---|---|
| `all` | every kernel in `/boot`, classic + recovery, plus the UKIs | installing from media; `uki_enabled: false`; debugging |
| `uki-backed` | UKIs first, then classic entries **only** for kernels that have a UKI | you want the signed path as default but keep single-user recovery |
| `uki-only` | UKIs and nothing else | hardened steady state — the shipped default |

### Mechanism

`grub-mkconfig` runs each drop-in only if `test -x` passes, so the lever is the
executable bit on `/etc/grub.d/10_linux`. There is no configuration knob:
`10_linux` hardcodes its kernel globs with no indirection to point elsewhere,
and editing it is not an option because it is a **conffile** — dpkg would prompt
on every `grub-common` upgrade.

`chmod -x` alone is not durable for the same reason: a new `10_linux` from
`grub-common` resets the mode, silently re-enabling every classic entry during a
security update. The counter is the mechanism Debian provides for exactly this:

```
dpkg-statoverride --update --add root root 0644 /etc/grub.d/10_linux
```

Reverting to `all` removes the override and restores `0755`. Both directions are
idempotent.

### `uki-backed` filters, it does not reimplement

`26_fde_linux` runs the disabled script through a filter — `sh /etc/grub.d/10_linux
| awk …`. `sh <file>` needs only the read bit, and the drop-in inherits
`grub-mkconfig`'s exported environment, so the entries are the genuine article
and this tree carries no duplicate of `linux_entry()` to keep in sync with
upstream. The filter buffers each `menuentry … {` block to its closing brace at
the same indentation (correct with or without `GRUB_DISABLE_SUBMENU`), reads the
version from the block's `linux` line, and drops the block if no matching `.efi`
exists on the ESP. A block with no `vmlinuz-` passes through untouched, so
`30_os-prober` entries are never collateral. It is numbered **26**, after
`25_uki`, so the signed entries come first and `GRUB_DEFAULT=0` selects a UKI.

### The never-empty invariant

A menu with zero Linux entries is an unbootable machine that still boots far
enough to look fine. Both scripts fall back rather than emit nothing: `25_uki`
runs `10_linux` unfiltered if the UKI directory is empty and `10_linux` is
non-executable; `26_fde_linux` passes through unfiltered if the ESP holds no UKI.
And the playbook refuses to disable `10_linux` at all unless it can first count
at least one `.efi` on the ESP — which is why `roles/uki` applies the policy as
its **last** task, after `update-uki` has run.

### What this does *not* block

- **Other operating systems.** `30_os_prober` is a different drop-in; its guard
  is `grub_disable_os_prober`.
- **The kernels themselves.** Nothing is deleted; they are simply not offered.
- **GRUB's command line.** `c` at the menu still loads any kernel by hand. That
  is what `grub_password_pbkdf2` is for.
- **The firmware boot menu.** Every signed UKI on the ESP stays individually
  selectable there.
- **Anything, once Secure Boot is off.** This is menu hygiene and default
  selection, not an enforcement boundary. The boundary is Secure Boot plus the
  signature.

### Retention, and why the ESP is a second limit

Under `uki-only`, `uki_retain` *is* the depth of the rollback path: a kernel that
ages out of the UKI set has no menu entry at all. The shipped value is `3` — the
running kernel and two rollbacks. `0` keeps one image per installed kernel.

The ESP is a second, independent limit and smaller than people expect: roughly
181 MiB per UKI (the initramfs is ~89% of it, because `initramfs.conf` ships
`MODULES=most`) against a 1 GiB ESP and `uki_esp_min_free_mib` reserved, so about
five images fit. `MODULES=dep` would roughly triple that — and is also the
classic way to produce an initramfs that cannot find root after a hardware
change, so it is offered as a lever rather than taken. Kernel `CONFIG_DEBUG_INFO`
is *not* a lever: the shipped modules are stripped and `vmlinuz` is built from a
stripped `vmlinux`.

`update-uki` therefore builds **newest-first**, skips an image it cannot fit,
keeps an image that already exists rather than pruning it, and prints
`skipped N image(s) for space`, which `roles/uki` turns into a warning naming
what was dropped. A kernel silently losing its only menu entry is the failure
this exists to prevent.

Each retained image is also a retained set of kernel CVEs, bootable by whoever is
at the keyboard, and PCR 7 cannot tell two of them apart. Three is the balance
struck here.

### Losing recovery mode

`uki-only` removes the `(recovery mode)` entries too — they come from `10_linux`.
Recovery is then an older UKI, the firmware boot menu, a live ISO, or flipping
to `all` and re-running. `uki-backed` is the tier that keeps single-user mode.

### The TPM2 guard

Rebuilding a UKI changes what systemd-stub measures into **PCR 11**. If the LUKS2
keyslot is sealed against PCR 11, the seal is now against a value that will never
occur again — and nothing fails during the run. The playbook reports success, and
at the next boot the machine asks for the PIN and rejects the correct one. The
tell is `TPM2_PT_LOCKOUT_COUNTER` staying `0x0`: the TPM never counted a wrong
guess, because the policy did not match.

So `roles/uki` probes the LUKS2 header **before** anything is rebuilt — the
header, not `tpm2_pcrs`, because what is enrolled may predate the current config:

```
cryptsetup luksDump <dev> | sed -n 's/^  \([0-9]\+\): systemd-tpm2$/\1/p'
cryptsetup token export --token-id N <dev>   # "tpm2-pcrs":[7]
```

If the bound PCRs intersect `uki_image_measured_pcrs` (4, 8, 9, 11 — the ones
that depend on *which image* booted), `uki_tpm2_guard` decides: `prompt`
(default) stops and asks, `fail` refuses, `ignore` warns only. **PCR 7 is the
safe case and the common one** — it covers Secure Boot policy and the certificate
that authorised the image, not the image itself. `secureboot.yml` is what moves
PCR 7, and `tpm2.yml` is the answer there.

Whichever way it goes, the recovery passphrase is in a separate keyslot sealed to
nothing and always works. After an intentional break, boot the new image and
re-seal with `sudo ansible-playbook tpm2.yml -e target_mount=/`.

One behaviour worth knowing: `ansible.builtin.pause` returns an **empty** answer
when stdin is not a terminal, so a non-interactive run always aborts at the
prompt. That is the safe direction, but it means unattended runs want
`uki_tpm2_guard=ignore`.

## `kernel_hold_distro`

Blocks the distribution's kernel packages so the self-built, MOK-signed kernel is
the only one that can be installed or upgraded.

**Why a pin and not `apt-mark hold`.** `apt-mark hold` holds a *package name*,
and Debian ships each ABI as a new name pulled in by a metapackage — so holding
the metapackage blocks new kernels but holds nothing against
`apt install linux-image-<anything>`, and shows up only in `apt-mark showhold`,
far from the kernel config. The pin is scoped by **origin** instead, which is the
distinction that matters: block kernels from the distribution's archives, leave
your own repo able to ship signed kernels normally.

```
Package: linux-image-* linux-headers-* linux-modules-* linux-kbuild-*
Pin: release o=Debian
Pin-Priority: -1
```

`-1` means "never select a version from this release". It does not touch
installed packages — the installed version lives in the `a=now` pseudo-release at
priority 100 — so nothing is removed or downgraded. Origins are a variable
(`kernel_hold_origins`) because they are site facts, not constants; check yours
with `apt-cache policy`. `linux-libc-dev` is deliberately **not** in the package
list — it is a glibc build dependency, and blocking it breaks unrelated upgrades.

**What it costs you.** The distro kernels already installed stop receiving
security updates. That is the point and also the risk: your self-built kernel
becomes the only one getting fixes, and only when you run `kernel-build.yml`.
This is a commitment to a maintenance habit, not a one-time hardening step.

It composes with `kernel_keep_all` rather than fighting it: that one stops
kernels being *removed*, this one stops new ones being *added*.

**Lifting it temporarily:**

```
apt -o Dir::Etc::Preferences=/dev/null install linux-image-amd64
```

or set `kernel_hold_distro: false` and re-run `kernel.yml`, which removes the
file.

## Verifying

```sh
# which drop-ins grub-mkconfig will actually run
ls -l /etc/grub.d/ | grep -E '10_linux|25_uki|26_fde_linux'
dpkg-statoverride --list /etc/grub.d/10_linux

# the resulting menu — the literal list of what the machine will offer
awk -F"'" '/menuentry |submenu /{print $2}' /boot/grub/grub.cfg

# UKIs backing it, and their signatures
ls -1 /boot/efi/EFI/Linux/
for f in /boot/efi/EFI/Linux/*.efi; do sbverify --list "$f" >/dev/null 2>&1 \
  && echo "SIGNED   $f" || echo "unsigned $f"; done

# the apt block
cat /etc/apt/preferences.d/99-fde-kernel-hold.pref
apt-cache policy linux-image-amd64          # candidate should read (none)
apt-get -s upgrade | grep -i linux-image    # should propose nothing
```

## Rollback

Nothing here is destructive or one-way.

| Symptom | Fix |
|---|---|
| menu offers no Linux entry | boot the UKI from the firmware menu, or GRUB `c` then `chainloader (hd0,gpt1)/EFI/Linux/<image>.efi` |
| want the old menu back | `uki_boot_policy: all`, re-run `finalize.yml` or `uki.yml` |
| want it back *now*, no ansible | `dpkg-statoverride --remove /etc/grub.d/10_linux; chmod +x /etc/grub.d/10_linux; rm -f /etc/grub.d/26_fde_linux; update-grub` |
| apt refuses a kernel you want | `kernel_hold_distro: false`, re-run `kernel.yml` — or remove the pin file |
