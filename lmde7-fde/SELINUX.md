# SELINUX.md — mandatory access control for the FDE box

`roles/selinux` replaces AppArmor with SELinux on the installed system: Debian's
reference policy, labelled filesystems, the audit subsystem wired up, and a
kernel command line that survives both the GRUB and the UKI boot paths.

It ships **permissive by default**. That is not a half-measure — a refpolicy
desktop that has never been run permissive is a desktop you cannot log into, and
this tree's recovery story costs a passphrase dance. Permissive loads the policy,
labels every file, and logs every decision that *would* have been denied, so
promoting to enforcing is one variable away and is backed by a denial log instead
of by hope.

## Why this exists

LUKS2 and AIDE are both about the machine when it is **off** — a stolen disk, an
evil maid editing `/boot`. Neither does anything while it is **running**: LUKS2
is transparent once unlocked, and AIDE reports yesterday's tampering. MAC is the
layer for the running machine.

LMDE ships AppArmor, which is a real layer, but it is path-based and Debian ships
profiles for a couple of dozen daemons. SELinux is label-based and covers every
object in the kernel's world by default, including the ones this layout cares
about: the `/var/log/audit` LV, the `/boot` mount, the fscrypt-ready `/home`, and
the device-mapper nodes underneath. The two cannot coexist — both set
`LSM_FLAG_EXCLUSIVE`, so the kernel initialises exactly one — so this role picks
SELinux deliberately and reversibly.

## Three kernel facts that shape the design

| Fact | Consequence |
|---|---|
| `CONFIG_SECURITY_SELINUX_BOOTPARAM` is **not set** | `selinux=1` / `selinux=0` are unknown parameters. The famous "boot with `selinux=0`" escape hatch **does not work here.** |
| `CONFIG_SECURITY_SELINUX_DISABLE` is **absent** (removed upstream in 6.4) | `SELINUX=disabled` in `/etc/selinux/config` is a no-op trap. Runtime disable no longer exists. |
| `CONFIG_SECURITY_SELINUX_DEVELOP=y` | `enforcing=0` on the kernel command line **does** work, and is the escape hatch this tree documents. |

So exactly one switch turns SELinux on: `security=selinux` on the kernel command
line, which is also what Debian's own `selinux-activate(8)` uses. Absent it, the
first exclusive LSM in `CONFIG_LSM` wins, and in Debian's build order that is
AppArmor. `selinux_enabled: false` therefore disables by *removing the command
line argument*, never by writing `SELINUX=disabled`.

## What the role does

| Step | Result |
|---|---|
| packages | policy + tooling into the target. **Non-fatal** — a chroot without DNS must not cost you the TPM2 enrollment |
| audit | `auditd` + `audispd-plugins`, enabled. This is what puts the `var_log_audit` LV to work and makes `ausearch`/`audit2allow` usable |
| config | `/etc/selinux/config` — `SELINUX=` and `SELINUXTYPE=` rewritten in place, every other key left to the package |
| booleans | `selinux_booleans` applied with `semanage boolean -m -N` (`-N` = no policy reload, the only form that works offline in a chroot) |
| AppArmor | `apparmor.service` disabled — see below |
| cmdline | `/etc/default/grub.d/91-fde-selinux.cfg`, its own drop-in so `selinux.yml` can flip SELinux without touching `target_config`'s hardening drop-in |
| UKI | the same string is compiled into `update-uki`, so enabling UKIs cannot silently drop SELinux |
| relabel | `/.autorelabel` containing `-F`, scheduling a full relabel at next boot |
| verify | `sestatus` and `check-selinux-installation` |

Everything after *packages* is gated on the install having succeeded. In
particular the command line drop-in is **never** written unless the policy is on
disk: putting `security=selinux` in front of a kernel with no policy to load is
how you get an LSM with no rules.

### Disabled, not masked

`apparmor.service` already carries `ConditionSecurity=apparmor`, so with SELinux
holding the LSM slot systemd skips it and it loads no profiles. Masking adds no
safety on top of that — and it breaks package installs. Measured: with the unit
**masked**, a maintainer script gets `Failed to reload apparmor.service: Unit
apparmor.service is masked.` Merely **disabled**, the three calls Debian
maintainer scripts actually make (`start`, `restart`, `try-reload-or-restart`)
all return rc=0 in silence because the condition skips them, and the unit stays
inactive either way. Same inertness, no false alarms.

Recovery mode is the case masking was supposed to cover, and it does not: the
recovery entry omits `GRUB_CMDLINE_LINUX_DEFAULT`, so it boots with AppArmor as
the active LSM and no profiles loaded whether the unit is masked or disabled.

It is symmetric: `selinux_enabled: false` re-enables AppArmor and unmasks it too,
so a box an earlier version of this role masked is repaired rather than stranded.

## The boot sequence after finalize

The first boot is *two* boots, and that is normal:

```
finalize.yml   writes 91-fde-selinux.cfg, update-grub, touches /.autorelabel
   |
boot 1   kernel: security=selinux -> SELinux enabled, nothing on disk is labelled
         systemd loads policy in PERMISSIVE (nothing would boot otherwise)
         the autorelabel generator sees /.autorelabel
             -> hijacks default.target to selinux-autorelabel.target
             -> fixfiles -F restore  (minutes; every file gets security.selinux)
             -> rm /.autorelabel ; systemctl --force reboot
   |
boot 2   labels correct, policy loaded, mode = selinux_mode. The real boot.
```

`multi-user.target` is never reached on boot 1, which is why the AIDE units do
not fire until boot 2. That ordering is load bearing.

## Interactions with the rest of the tree

**AIDE.** Debian's AIDE is linked against `libselinux1` and its `InodeData` group
covers `security.selinux`, so a relabel changes what AIDE sees for every file in
`/boot`. With the default `aide_init_mode: firstboot` this is harmless — the
relabel finishes on boot 1 and the baseline is taken on boot 2. With
`aide_init_mode: now` the baseline is taken in the chroot, *before* the relabel,
and the boot check on boot 2 alarms on a change you made yourself. The role warns
on that combination; the fix is one `fde-aide-boot-accept` after the reboot.

**UKI.** With `uki_enabled: true` the UKI's *embedded* command line is the only
one that applies — `/etc/default/grub` is not consulted at all. `update-uki`
compiles `selinux_cmdline` in itself, and `selinux.yml` re-runs the `uki` role for
the same reason: flipping SELinux while UKIs are live has to rebuild them or the
change does not take.

**The recovery menu entry.** The drop-in appends to `GRUB_CMDLINE_LINUX_DEFAULT`,
not `GRUB_CMDLINE_LINUX`, matching the rest of the hardening. GRUB's recovery
entries omit `_DEFAULT`, so recovery mode boots with DAC only — a guaranteed way
back into a box whose policy you have broken. Move the line to
`GRUB_CMDLINE_LINUX` if you would rather have no such door.

**fscrypt / f2fs `/home`.** fscrypt encrypts file contents and filenames, not
xattrs, so `security.selinux` is stored and read normally on f2fs and a relabel
works. A relabel of a *locked* encrypted directory walks base64 ciphertext names
and labels them by the parent's default, which is correct — and moot today,
because `prep.yml` sets fscrypt up without applying any policy.

**`migrate_mounts`.** It rsyncs with `-aHAXS`; `-X` carries xattrs, so labels
survive the LV migration.

## The promotion path

Permissive is the starting state, not the destination.

1. Boot the relabelled system and use it normally for a week — log in and out,
   suspend, update, run the apps you actually run.
2. Read what accumulated:

   ```
   ausearch -m avc,user_avc,selinux_err -ts boot
   ausearch -m avc -ts recent | audit2allow -l          # human-readable summary
   ```
3. Decide, per denial: a **boolean**, a **file-context fix**
   (`semanage fcontext -a -t <type> '<regex>'` then `restorecon -Rv <path>`), or a
   genuinely missing rule. Never reach for `audit2allow -M` first — a generated
   module that allows whatever leaked is how an SELinux install becomes
   decorative.
4. When `ausearch -m avc -ts boot` is quiet across a normal day, set
   `selinux_mode: enforcing`, re-run `selinux.yml`, reboot.

`setenforce 1` flips a running system for a smoke test without a reboot, and
`setenforce 0` flips it straight back. That is the cheap way to find out whether
step 4 is going to hurt.

## Steam, games, and the desktop under enforcing

Measured in permissive mode with Steam running and a game launched. Only **two**
denial classes touch Steam; everything else is already allowed, because Debian's
`default` policy ships the `unconfined` module and the desktop session runs as
`unconfined_t`.

**1. `execmem` — fixed by a boolean.** Writable-and-executable memory: JITs, and
Steam's `inspect-library` probe. Note the majority of these come from the desktop
itself, not Steam. The `allow_execmem` boolean covers it and ships `off`:

```yaml
selinux_booleans:
  allow_execmem: true
```

**2. `nnp_transition` — needs a local module.** Steam's container runtime
(pressure-vessel) sets `PR_SET_NO_NEW_PRIVS`, and the policy capability
`nnp_nosuid_transition` then makes the kernel check `process2:nnp_transition`
before letting `unconfined_t` transition to `ldconfig_t`. There is **no boolean
for it** and the policy has no such rule, so under enforcing the transition fails
and pressure-vessel cannot set up the runtime. Narrow enough to write by hand
rather than letting `audit2allow -M` generalise from whatever leaked:

```
# fde-steam.te
policy_module(fde_steam, 1.0)
gen_require(`
    type unconfined_t;
    type ldconfig_t;
')
allow unconfined_t ldconfig_t:process2 { nnp_transition nosuid_transition };
```

```
checkmodule -M -m -o fde-steam.mod fde-steam.te
semodule_package -o fde-steam.pp -m fde-steam.mod
semodule -i fde-steam.pp
```

Until both are in place, **do not set `selinux_mode: enforcing` on a box you play
games on.** Permissive costs nothing here.

## Escape hatches

In descending order of preference — the first two need only the GRUB menu:

| Situation | Do this |
|---|---|
| Enforcing broke something, box still boots | `setenforce 0`, fix, `setenforce 1` |
| Enforcing broke the boot | press `e` in GRUB, append `enforcing=0` to the `linux` line, F10 |
| Need SELinux gone for one boot | the **recovery** entry already omits `security=selinux` |
| Need SELinux gone for good | `selinux_enabled: false`, re-run `selinux.yml`, reboot |

`selinux=0` is **not** on that list, and neither is `SELINUX=disabled`.

## Configuration

All in `group_vars/all.yml`; `roles/config_contract` asserts every key exists and
that the enumerated ones are legal.

| Variable | Default | Meaning |
|---|---|---|
| `selinux_enabled` | `true` | master switch. `false` actively undoes: drop-in removed, `/.autorelabel` removed, AppArmor unmasked |
| `selinux_mode` | `permissive` | `permissive` \| `enforcing`. `disabled` is deliberately not legal |
| `selinux_policy_type` | `default` | `default` \| `mls`. Must match the installed `selinux-policy-*` package; the role asserts it does |
| `selinux_relabel_mode` | `auto` | `auto` = relabel only when the target looks unlabelled \| `always` \| `never` |
| `selinux_cmdline` | `security=selinux audit=1 audit_backlog_limit=8192` | appended to the GRUB drop-in *and* compiled into `update-uki` |
| `selinux_disable_apparmor` | `true` | disable `apparmor.service` while SELinux is on |
| `selinux_install_auditd` | `true` | turning this off while `audit=1` stays on the cmdline means kernel audit records go to the ring buffer with nothing collecting them |
| `selinux_booleans` | `{}` | `name: true/false` map |
| `selinux_packages` / `selinux_audit_packages` | see config | policy + tooling |

`selinux_relabel_mode: auto` decides by reading the extended attributes of
`{{ target_mount }}/etc/fstab` and scheduling a relabel when `security.selinux` is
absent. That is a fact about the filesystem rather than a stamp file the role
wrote, so it stays correct across reinstalls and restores. It does **not** notice
a policy change that moved file contexts — after changing `selinux_policy_type`,
adding `semanage fcontext` rules, or installing modules, use `always` once.

## Running it

Inside a full run, `finalize.yml` calls the role after `uki`, before the
post-tasks that rebuild the initramfs and regenerate `grub.cfg`. On its own:

```
sudo ansible-playbook selinux.yml -e target_mount=/     # on the booted system
sudo ansible-playbook selinux.yml                       # live ISO, /target mounted
```

`selinux.yml` regenerates `grub.cfg` itself and re-runs the `uki` role when
`uki_enabled` is true, because both carry the command line.

## Operating it

```
sestatus                          # mode, policy, whether the store is loaded
check-selinux-installation        # sanity tests; prints problems only
ls -Z /boot /home /var/log/audit  # labels present?
getenforce ; setenforce 0|1       # runtime mode, until reboot

ausearch -m avc -ts boot          # denials this boot
semanage boolean -l -C            # booleans changed from policy default
semanage fcontext -l -C           # local file-context rules
restorecon -Rvn /path             # -n = dry run
fixfiles -F restore               # full relabel, now, on a running system
```

## What this does not do

- **No custom policy modules.** A policy module is a per-machine artifact built
  against denials this tree cannot predict; shipping a generated one would be
  shipping someone else's guess. `selinux-policy-dev` is installable if you want
  to write them.
- **No confined login users.** Debian's `default` policy ships `unconfined`
  enabled, so interactive logins land in `unconfined_t` and the desktop behaves.
  Confining users is a real hardening step and a real way to break the desktop;
  it belongs in a deliberate follow-up.
- **No MLS.** `selinux_policy_type: mls` is accepted and wired, but untested.
