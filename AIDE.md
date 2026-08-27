# AIDE.md — file-integrity monitoring for the FDE box

`roles/aide` installs and configures AIDE (Advanced Intrusion Detection
Environment) on the target system. It is the detection half of the threat model
`README.md` describes under *"What the TPM binding is and is not worth here"*.

## Why this exists

The FDE design leaves exactly one deliberately unencrypted, unsigned surface:
**`/boot` and the ESP**. TPM2 sealing binds to PCR 7 only — Secure Boot *policy*,
not the kernel or initrd — so a tampered `initrd` still satisfies the seal and
could capture the PIN as it is typed. Offline (stolen-disk) attack is covered by
LUKS2; the gap is the evil maid who edits `/boot` while the machine is off.

AIDE closes the *detection* side of that gap, and the layout works in its favour:

- **The database lives inside the container.** `/var/lib/aide/aide.db` is on the
  `var` LV, i.e. inside LUKS2. An attacker who can rewrite `/boot` cannot rewrite
  the hashes it will be measured against — that is the whole game with file
  integrity monitoring, and most installs get it wrong.
- **The check runs before you type anything.** `fde-aide-boot-check.service` runs
  a `/boot`-limited check on every boot, so tampering surfaces on the boot after
  it happened rather than at 01:50 the next morning.

AIDE is detection, not prevention. It tells you the initrd changed; it cannot
stop it. The prevention story is still the UKI path (`uki_enabled: true`).

## What the role does

| Step | Result |
|---|---|
| packages | `aide`, `aide-common`, `liblockfile-bin` into the target. Debconf default is *not* to init the database, so nothing is hashed during `apt` |
| rules | `/etc/aide/aide.conf.d/99_fde_local` — prunes the noisy trees listed in `aide_exclude_paths` |
| settings | `/etc/default/aide` — `COMMAND`, `COPYNEWDB`, `MAILTO`, `CRON_DAILY_RUN` driven from `group_vars` |
| validation | `aide --config-check`; hard-fails the run when `aide_require_config_check` is true |
| baseline | `fde-aide-init.service` — one-shot at first boot, `aideinit -y -f`, self-skipping once `aide.db` exists |
| boot check | `fde-aide-boot-check.service` — `aide --check --limit '^/boot'` on every boot |
| re-baseline | `/usr/local/sbin/fde-aide-boot-accept`, run from the kernel and initramfs hooks so legitimate updates do not cry wolf |
| daily | Debian's own `dailyaidecheck.timer` (whole system, 01:50 + up to 2 h jitter) |

Debian's shipped configuration already checks everything under `/` with the
`Full` attribute set (`99_aide_root`: `/ 0 Full`), including `/boot` and — via
`98_aide_vfat` — the ESP with inode checks dropped, which is what FAT requires.
So the role adds no `/boot` rules of its own: they would duplicate rules that are
already maximal. What it adds is *pruning*, a baseline that happens at the right
moment, and the boot-time check.

## Files it installs

```
/etc/aide/aide.conf.d/99_fde_local           pruning rules (managed)
/etc/default/aide                            wrapper settings (keys managed in place)
/usr/local/sbin/fde-aide-boot-check          check /boot + ESP against the db
/usr/local/sbin/fde-aide-boot-accept         re-baseline just the /boot subtree
/etc/systemd/system/fde-aide-init.service    first-boot `aideinit`
/etc/systemd/system/fde-aide-boot-check.service
/etc/kernel/postinst.d/zzz-fde-aide-accept        accept after a kernel install
/etc/kernel/postrm.d/zzz-fde-aide-accept          ... after a kernel removal
/etc/initramfs/post-update.d/zzz-fde-aide-accept  ... after any update-initramfs
/var/lib/aide/aide.db                        the baseline (inside LUKS)
/var/log/aide/aide.log                       daily-check reports
```

## Operating it

**First boot.** `fde-aide-init.service` hashes the system once (single-digit
minutes; it is `Nice=19`/idle-IO). Until it finishes there is no baseline and
`fde-aide-boot-check` skips itself.

```
systemctl status fde-aide-init.service
journalctl -u fde-aide-init.service
tail /var/log/aide/aideinit.log /var/log/aide/aideinit.errors
```

**Every boot.**

```
systemctl status fde-aide-boot-check.service    # green = /boot matches the baseline
journalctl -u fde-aide-boot-check.service -b
```

A failed unit means AIDE reported added, removed or changed entries under
`/boot`. Read the report before doing anything else: if you did not update a
kernel, initramfs, or GRUB since the last accept, treat the machine as
compromised — the initrd is the one thing on this box an attacker can edit
offline and still have executed with your PIN in flight.

**After a legitimate boot-chain change.** Kernel installs and removals, and any
`update-initramfs` run, re-baseline themselves through the hooks. Anything that
writes to `/boot` or the ESP without going through those — a bare `grub-install`
or `update-grub`, dropping a file on the ESP, re-running `finalize.yml` — needs:

```
sudo /usr/local/sbin/fde-aide-boot-accept
```

That runs `aide --update --limit '^/boot'` into a private database file and moves
it over `aide.db`, so entries outside `/boot` are carried across untouched and
the shared `aide.db.new` the daily job uses is never disturbed.

**Whole-system re-baseline** (after a big upgrade, once you have read the report):

```
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db      # daily job leaves it here
# or, to rescan from scratch:
sudo aideinit -y -f
```

**Daily report.** `COPYNEWDB=no` means changes keep being reported until you
accept them. Without an MTA installed the mail step is a no-op and the report
lives in `/var/log/aide/aide.log` plus the journal — that is the expected state
on this box; do not read a missing mail as "no changes".

## Configuration surface

All in `group_vars/all.yml`, all asserted by `roles/config_contract`:

| Key | Default | Meaning |
|---|---|---|
| `aide_enabled` | `true` | master switch; false makes the role a no-op with a message |
| `aide_packages` | `aide`, `aide-common`, `liblockfile-bin` | `liblockfile-bin` provides `dotlockfile`, which the daily check needs for its lock |
| `aide_exclude_paths` | `/home`, `/tmp`, `/var/tmp`, `/var/cache`, `/var/lib/flatpak`, `/var/lib/systemd/coredump` | pruned with non-recursive negative rules (`-<path>$ 0`); AIDE never descends into them |
| `aide_boot_limit` | `^/boot` | the regex the boot check and the accept script are limited to (covers the ESP at `/boot/efi`) |
| `aide_boot_check` | `true` | install + enable the per-boot check |
| `aide_boot_check_wall` | `true` | also `wall` the alarm to logged-in terminals |
| `aide_boot_accept_on_kernel_update` | `true` | install the accept hooks |
| `aide_accept_hook_dirs` | kernel `postinst.d`/`postrm.d`, `initramfs/post-update.d` | where those hooks are dropped, as `zzz-fde-aide-accept` — the `zzz` matters, it has to sort after `zz-update-grub` so the new `grub.cfg` is on disk before the baseline is taken |
| `aide_init_mode` | `firstboot` | `firstboot` \| `now` \| `never` — when the baseline is taken |
| `aide_daily_enabled` | `true` | enable `dailyaidecheck.timer` |
| `aide_daily_command` | `update` | `check` or `update`; `update` refreshes `aide.db.new` at no extra cost |
| `aide_daily_copynewdb` | `no` | `no` \| `yes` \| `ifnochange`. `yes` means a change is reported once and then silently accepted — do not |
| `aide_mailto` | `root` | recipient if an MTA ever gets installed |
| `aide_require_config_check` | `true` | hard-fail the run if `aide --config-check` rejects the config |

## Decisions & caveats

**Why `/home` is excluded.** It is f2fs, user-churny, and often the largest
filesystem on the box; hashing it daily buys noise, not signal, and noise is how
file integrity monitoring dies. It is also the tree an attacker who is already
root does not need to touch. Delete the entry from `aide_exclude_paths` if you
want it covered — the role does not care.

**Why the baseline is taken at first boot, not in the chroot.** `finalize.yml`
rebuilds the initramfs, installs GRUB and regenerates `grub.cfg` *after* the roles
run, and the installed system's own first boot writes machine-id, seeds and
journals. A database made in the chroot would be stale before it was ever read.
`aide_init_mode: now` exists for the post-boot rerun case
(`aide.yml -e target_mount=/`), where "now" and "first boot" are the same moment.

**AIDE cannot cost you a bootable system.** The role is included as the *last*
post-task of `finalize.yml`, after the initramfs rebuild, `grub-install` and
`update-grub`, and its package install is non-fatal — a chroot without working
DNS produces a warning and a skipped role, not a half-finalized machine. The one
thing that will stop the play is `aide --config-check` rejecting the
configuration, and by then everything boot-critical is already on disk. Set
`aide_require_config_check: false` if you would rather it warn.

**The accept hook is a trust decision.** Anything running as root can invoke
`fde-aide-boot-accept` and launder a `/boot` change into the baseline. That is
not a weakening: root can already rewrite `aide.db` directly. It does mean AIDE
here is aimed at the *offline* attacker — which is exactly the gap this project
has.

**PCR 7 and Secure Boot.** If you later go the UKI + custom-keys route, the
initrd stops being an unsigned blob and this check becomes a second line rather
than the only line. Keep it anyway: it also catches the disk quietly rotting.
