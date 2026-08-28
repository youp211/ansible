# AIDE.md — file-integrity monitoring for the FDE box

`roles/aide` installs and configures AIDE on the target system. It is the
detection half of the threat model `README.md` describes under *"What the TPM
binding is and is not worth"*.

## Why this exists

The FDE design leaves exactly one deliberately unencrypted, unsigned surface:
**`/boot` and the ESP**. TPM2 sealing binds to PCR 7 — Secure Boot *policy*, not
the kernel or initrd — so a tampered initrd still satisfies the seal and could
capture the PIN as it is typed. Offline attack is covered by LUKS2; the gap is
the evil maid who edits `/boot` while the machine is off.

The layout works in AIDE's favour on the two points most installs get wrong:

- **The database lives inside the container.** `/var/lib/aide/aide.db` is on the
  `var` LV, i.e. inside LUKS2. Whoever can rewrite `/boot` cannot rewrite the
  hashes it will be measured against.
- **The check runs before you type anything.** `fde-aide-boot-check.service`
  runs a `/boot`-limited check on every boot, so tampering surfaces on the boot
  after it happened rather than at 01:50 the next morning.

AIDE is detection, not prevention. It tells you the initrd changed; it cannot
stop it. Prevention is the UKI path.

## What the role does

| Step | Result |
|---|---|
| packages | `aide`, `aide-common`, `liblockfile-bin`. Debconf default is *not* to init the database, so nothing is hashed during `apt` |
| rules | `/etc/aide/aide.conf.d/99_fde_local` — prunes the noisy trees in `aide_exclude_paths` |
| settings | `/etc/default/aide` — `COMMAND`, `COPYNEWDB`, `MAILTO`, `CRON_DAILY_RUN` from `group_vars` |
| validation | `aide --config-check`; hard-fails the run when `aide_require_config_check` |
| baseline | `fde-aide-init.service` — one-shot at first boot, self-skipping once `aide.db` exists |
| boot check | `fde-aide-boot-check.service` — `aide --check --limit '^/boot'` every boot |
| re-baseline | `/usr/local/sbin/fde-aide-boot-accept`, run from the kernel and initramfs hooks so legitimate updates do not cry wolf |
| daily | Debian's own `dailyaidecheck.timer` (whole system, 01:50 + jitter) |

Debian's shipped configuration already checks everything under `/` with the
`Full` attribute set, including `/boot` and the ESP with inode checks dropped
(which is what FAT requires). So the role adds no `/boot` rules of its own — they
would duplicate rules that are already maximal. What it adds is *pruning*, a
baseline at the right moment, and the boot-time check.

## Files it installs

```
/etc/aide/aide.conf.d/99_fde_local           pruning rules (managed)
/etc/default/aide                            wrapper settings (keys managed in place)
/usr/local/sbin/fde-aide-boot-check          check /boot + ESP against the db
/usr/local/sbin/fde-aide-boot-accept         re-baseline just the /boot subtree
/etc/systemd/system/fde-aide-init.service    first-boot baseline
/etc/systemd/system/fde-aide-boot-check.service
/etc/kernel/postinst.d/zzz-fde-aide-accept        accept after a kernel install
/etc/kernel/postrm.d/zzz-fde-aide-accept          ... after a kernel removal
/etc/initramfs/post-update.d/zzz-fde-aide-accept  ... after any update-initramfs
/var/lib/aide/aide.db                        the baseline (inside LUKS)
```

## Operating it

**First boot.** `fde-aide-init.service` hashes the system once (single-digit
minutes; `Nice=19`/idle-IO). Until it finishes there is no baseline and the boot
check skips itself.

**Every boot.**

```
systemctl status fde-aide-boot-check.service    # green = /boot matches the baseline
journalctl -u fde-aide-boot-check.service -b
```

A failed unit means AIDE reported added, removed or changed entries under
`/boot`. Read the report before doing anything else: if you did not update a
kernel, initramfs or GRUB since the last accept, treat the machine as
compromised — the initrd is the one thing on this box an attacker can edit
offline and still have executed with your PIN in flight.

**After a legitimate boot-chain change.** Kernel installs and removals and any
`update-initramfs` run re-baseline themselves through the hooks. Anything that
writes to `/boot` or the ESP without going through those — a bare `grub-install`,
dropping a file on the ESP, re-running `finalize.yml` — needs:

```
sudo /usr/local/sbin/fde-aide-boot-accept
```

That runs `aide --update --limit '^/boot'` into a private database and moves it
over `aide.db`, so entries outside `/boot` carry across untouched and the shared
`aide.db.new` the daily job uses is never disturbed.

**Whole-system re-baseline** (after a big upgrade, once you have read the report):

```
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db   # daily job leaves it here
sudo aideinit -y -f                                       # or rescan from scratch
```

**Daily report.** `COPYNEWDB=no` means changes keep being reported until you
accept them. Without an MTA the mail step is a no-op and the report lives in
`/var/log/aide/aide.log` plus the journal — do not read missing mail as "no
changes".

## Configuration surface

All in `group_vars/all.yml`, all asserted by `roles/config_contract`:

| Key | Default | Meaning |
|---|---|---|
| `aide_enabled` | `true` | master switch |
| `aide_packages` | `aide`, `aide-common`, `liblockfile-bin` | `liblockfile-bin` provides `dotlockfile`, which the daily check needs for its lock |
| `aide_exclude_paths` | `/home`, `/tmp`, `/var/tmp`, `/var/cache`, … | pruned with non-recursive negative rules; AIDE never descends into them |
| `aide_boot_limit` | `^/boot` | regex the boot check and accept script are limited to (covers the ESP at `/boot/efi`) |
| `aide_boot_check` | `true` | install + enable the per-boot check |
| `aide_boot_check_wall` | `true` | also `wall` the alarm to logged-in terminals |
| `aide_boot_accept_on_kernel_update` | `true` | install the accept hooks |
| `aide_accept_hook_dirs` | kernel `postinst.d`/`postrm.d`, `initramfs/post-update.d` | dropped as `zzz-fde-aide-accept` — the `zzz` matters, it must sort after `zz-update-grub` so the new `grub.cfg` is on disk before the baseline is taken |
| `aide_init_mode` | `firstboot` | `firstboot` \| `now` \| `never` |
| `aide_daily_enabled` | `true` | enable `dailyaidecheck.timer` |
| `aide_daily_command` | `update` | `update` refreshes `aide.db.new` at no extra cost |
| `aide_daily_copynewdb` | `no` | `yes` means a change is reported once and then silently accepted — do not |
| `aide_mailto` | `root` | recipient if an MTA is ever installed |
| `aide_require_config_check` | `true` | hard-fail the run if `aide --config-check` rejects the config |

## Decisions & caveats

**Why `/home` is excluded.** It is user-churny and often the largest filesystem
on the box; hashing it daily buys noise, not signal, and noise is how file
integrity monitoring dies. It is also the tree an attacker who is already root
does not need to touch. Delete the entry if you want it covered.

**Why the baseline is taken at first boot, not in the chroot.** `finalize.yml`
rebuilds the initramfs and regenerates `grub.cfg` *after* the roles run, and the
installed system's own first boot writes machine-id, seeds and journals. A
database made in the chroot would be stale before it was read. `aide_init_mode:
now` exists for the post-boot rerun case, where "now" and "first boot" coincide.

**AIDE cannot cost you a bootable system.** The role is the *last* post-task of
`finalize.yml`, after the initramfs rebuild and `grub-install`, and its package
install is non-fatal — a chroot without DNS produces a warning and a skipped
role, not a half-finalized machine. The one thing that will stop the play is
`aide --config-check` rejecting the configuration, and by then everything
boot-critical is on disk.

**The accept hook is a trust decision.** Anything running as root can invoke
`fde-aide-boot-accept` and launder a `/boot` change into the baseline. That is
not a weakening — root can already rewrite `aide.db` directly. It does mean AIDE
here is aimed at the *offline* attacker, which is exactly this project's gap.

**PCR 7 and Secure Boot.** Once you go the UKI + custom-keys route the initrd
stops being an unsigned blob and this check becomes a second line rather than the
only line. Keep it anyway: it also catches the disk quietly rotting.
