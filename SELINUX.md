# SELINUX.md — mandatory access control for the FDE box

`roles/selinux` replaces AppArmor with SELinux on the installed system: Debian's
reference policy, labelled filesystems, the audit subsystem wired up, and a
kernel command line that survives both the GRUB and the UKI boot paths.

It ships **permissive by default**. That is not a half-measure — on Debian a
refpolicy desktop that has never been run permissive is a desktop you cannot log
into, and this tree's whole recovery story costs a passphrase dance. Permissive
loads the policy, labels every file, and logs every decision that *would* have
been denied; promoting to enforcing is then one variable away and is backed by
the denial log you collected instead of by hope. See *The promotion path*.

## Why this exists

`README.md` covers the encryption story and `AIDE.md` the detection story. Both
are about the machine when it is **off** — a stolen disk, an evil maid editing
`/boot`. Neither does anything about the machine while it is **running**: LUKS2
is transparent once unlocked, and AIDE reports yesterday's tampering.

MAC is the layer for the running machine. LMDE ships AppArmor and enables it,
which is a real layer — but it is path-based, profile-by-profile, and Debian
ships profiles for a couple of dozen daemons. SELinux is label-based and covers
every object in the kernel's world by default, including the ones this layout
cares about: the `/var/log/audit` LV, the `/boot` mount, the fscrypt-ready
`/home`, and the device-mapper nodes underneath all of it.

The two cannot coexist. Both set `LSM_FLAG_EXCLUSIVE`, so the kernel initialises
exactly one of them; picking SELinux is picking *against* AppArmor, and this
role does that deliberately and reversibly.

## Three kernel facts that shape the design

Checked against this workstation's own `/boot/config-*` (it *is* LMDE 7, so the
answers are the target's answers, not a guess):

| Fact | Consequence |
|---|---|
| `CONFIG_SECURITY_SELINUX_BOOTPARAM` is **not set** | `selinux=1` / `selinux=0` are unknown parameters. The famous "boot with `selinux=0`" escape hatch **does not work here.** |
| `CONFIG_SECURITY_SELINUX_DISABLE` is **absent** (removed upstream in 6.4) | `SELINUX=disabled` in `/etc/selinux/config` is a no-op trap. Runtime disable no longer exists. |
| `CONFIG_SECURITY_SELINUX_DEVELOP=y` | `enforcing=0` on the kernel command line **does** work, and is the escape hatch this tree documents. |

So there is exactly one switch that turns SELinux on: `security=selinux` on the
kernel command line, which is also what Debian's own `selinux-activate(8)` uses.
Absent it, the first exclusive LSM in `CONFIG_LSM` wins — and in Debian's build
order that is `apparmor`. Present it, the kernel's `chosen_major_lsm` is SELinux
and AppArmor is never initialised.

`selinux_enabled: false` therefore disables by *removing the command line
argument*, never by writing `SELINUX=disabled`.

## What the role does

| Step | Result |
|---|---|
| packages | `selinux-basics`, `selinux-policy-default`, `policycoreutils`, `policycoreutils-python-utils`, `selinux-utils`, `setools`, `checkpolicy`, `semodule-utils` into the target. **Non-fatal** — a chroot without DNS must not cost you the TPM2 enrollment |
| audit | `auditd` + `audispd-plugins`, enabled. This is what finally puts the `var_log_audit` LV to work, and what makes `ausearch`/`audit2allow` usable |
| config | `/etc/selinux/config` — `SELINUX=` and `SELINUXTYPE=` rewritten in place, every other key left to the package |
| booleans | `selinux_booleans` applied with `semanage boolean -m -N` (`-N` = no policy reload, the only form that works offline in a chroot) |
| AppArmor | `apparmor.service` disabled **and masked** — see *Why mask* below |
| cmdline | `/etc/default/grub.d/91-fde-selinux.cfg`, a drop-in of its own so `selinux.yml` can flip SELinux without touching `target_config`'s `90-fde-hardening.cfg` |
| UKI | the same string is compiled into `update-uki`, so enabling UKIs cannot silently drop SELinux |
| relabel | `/.autorelabel` containing `-F`, scheduling a full relabel at the next boot |
| verify | `sestatus` and `check-selinux-installation`, informational in a chroot, real post-boot |

Everything after *packages* is gated on the install having actually succeeded.
In particular the command line drop-in is **never** written unless the policy is
on disk: putting `security=selinux` in front of a kernel with no policy to load
is how you get a boot that has an LSM and no rules.

### Why disabled, not masked

`apparmor.service` already carries `ConditionSecurity=apparmor`, so with SELinux
holding the LSM slot systemd skips it and it loads no profiles. Masking adds no
safety on top of that — and it breaks package installs.

Measured on the live box after the switch. With the unit **masked**, a maintainer
script gets:

```
Failed to reload apparmor.service: Unit apparmor.service is masked.
```

which is what Discord's postinst hit. Merely **disabled**, the three calls Debian
maintainer scripts actually make all return `rc=0` in silence:

| call | masked | disabled |
|---|---|---|
| `systemctl start apparmor` | error | rc=0, silent (condition skips it) |
| `systemctl restart apparmor` | error | rc=0, silent |
| `systemctl try-reload-or-restart apparmor` | error | rc=0, silent |
| `systemctl reload apparmor` | error | "not active, cannot reload" |

and the unit stays `inactive` with `aa-enabled` reporting `No - disabled at boot`
either way. So disabling is strictly better: same inertness, no false alarms.

Recovery mode is the case masking was supposed to cover, and it does not: the
recovery entry omits `GRUB_CMDLINE_LINUX_DEFAULT`, so it boots with AppArmor as
the active LSM and *no profiles loaded* whether the unit is masked or merely
disabled. Masking bought nothing there.

It is symmetric: `selinux_enabled: false` re-enables AppArmor, and unmasks it too,
so a box that an earlier version of this role masked is repaired rather than left
stuck. Flipping the flag back and forth never strands the box with no MAC at all.

## Files it installs

```
/etc/default/grub.d/91-fde-selinux.cfg   security=selinux + audit= (managed)
/etc/selinux/config                      SELINUX= / SELINUXTYPE= (keys managed in place)
/.autorelabel                            contains "-F" — consumed and deleted at next boot
/etc/selinux/default/                     policy store, ~350 modules (from the package)
/var/log/audit/audit.log                  on its own LV, inside LUKS
```

## The boot sequence after finalize

The first boot is *two* boots, and that is normal:

```
finalize.yml           writes 91-fde-selinux.cfg, update-grub, touches /.autorelabel
   |
boot 1   kernel: security=selinux -> SELinux enabled, nothing on disk is labelled
         systemd loads policy in PERMISSIVE (nothing would boot otherwise)
         selinux-autorelabel-generator.sh sees /.autorelabel
             -> hijacks default.target to selinux-autorelabel.target
             -> fixfiles -F restore  (minutes; every file gets security.selinux)
             -> rm /.autorelabel ; systemctl --force reboot
   |
boot 2   labels correct, policy loaded, mode = selinux_mode. This is the real boot.
```

`multi-user.target` is never reached on boot 1, which is why the AIDE units — both
`WantedBy=multi-user.target` — do not fire until boot 2. That ordering is load
bearing; see below.

## Interactions with the rest of the tree

**AIDE.** Debian's AIDE is linked against `libselinux1`, and its `InodeData`
group includes `X`, the compound extended-attribute group that covers
`security.selinux`. A relabel therefore changes what AIDE sees for every file in
`/boot`. With the default `aide_init_mode: firstboot` this is harmless — the
relabel finishes on boot 1, the baseline is taken on boot 2, and the baseline
records the post-relabel labels. With `aide_init_mode: now` the baseline is taken
in the chroot, *before* the relabel, and the boot check on boot 2 will alarm on
a change you made yourself. The role warns when it sees that combination; the fix
is `sudo /usr/local/sbin/fde-aide-boot-accept` once, after the relabel reboot.

**UKI.** With `uki_enabled: true` GRUB chainloads a Unified Kernel Image whose
*embedded* command line is the only one that applies — `/etc/default/grub` is not
consulted at all. `update-uki` therefore compiles `selinux_cmdline` in itself.
`selinux.yml` re-runs the `uki` role for the same reason: flipping SELinux on
while UKIs are live has to rebuild them or the change simply does not take.

**The recovery menu entry.** The drop-in appends to `GRUB_CMDLINE_LINUX_DEFAULT`,
not `GRUB_CMDLINE_LINUX`, matching what `90-fde-hardening.cfg` does with the rest
of the hardening. The consequence is deliberate: GRUB's *recovery* entries omit
`_DEFAULT`, so recovery mode boots with AppArmor-less, SELinux-less DAC only —
a guaranteed way back into a box whose policy you have broken. Move the line to
`GRUB_CMDLINE_LINUX` if you would rather have no such door; you then depend on
`enforcing=0` alone.

**fscrypt / f2fs `/home`.** fscrypt encrypts file contents and filenames, not
xattrs, so `security.selinux` is stored and read normally on f2fs and a relabel
of `/home` works. A relabel of a *locked* encrypted directory walks base64
ciphertext names and labels them by the parent's default — which is correct, and
moot today because `prep.yml` sets fscrypt up without applying any policy.

**`migrate_mounts`.** It already rsyncs with `-aHAXS`; `-X` carries xattrs, so
labels survive the LV migration. The first-boot relabel makes this belt-and-
braces rather than load bearing.

**`/tmp`, `/var/tmp` `noexec`.** No interaction. `fixfiles` writes temp files but
never executes them.

## The promotion path

Permissive is the *starting* state, not the destination.

1. Boot the relabelled system and use it normally for a week — log in and out,
   suspend, update, run the desktop apps you actually run. Every would-be denial
   is recorded.
2. Read what accumulated:

   ```
   ausearch -m avc,user_avc,selinux_err -ts boot
   ausearch -m avc -ts recent | audit2allow -l          # human-readable summary
   ```
3. Decide, per denial: a **boolean** (`semanage boolean -l | grep <thing>`), a
   **file-context fix** (`semanage fcontext -a -t <type> '<regex>'` then
   `restorecon -Rv <path>`), or a genuinely missing rule. Never reach for
   `audit2allow -M` first — a generated module that allows whatever leaked is how
   an SELinux install becomes decorative.
4. When `ausearch -m avc -ts boot` is quiet across a normal day, set
   `selinux_mode: enforcing` and re-run:

   ```
   sudo ansible-playbook selinux.yml -e target_mount=/
   sudo reboot
   ```

`setenforce 1` will flip a running system for a smoke test without a reboot, and
`setenforce 0` flips it straight back. That is the cheap way to find out whether
step 4 is going to hurt.

## Steam, games, and the desktop under enforcing

Measured on this box in permissive mode with Steam running and a game (STRAFTAT)
launched, so this is what an actual session produces rather than a guess. Only
**two** denial classes touch Steam, and everything else it does is already
allowed — Debian's `default` policy ships the `unconfined` module, so the desktop
session and everything it launches runs as `unconfined_t`.

**1. `execmem` — fixed by a boolean.**

```
avc: denied { execmem } ... comm="cinnamon"          scontext=...:unconfined_t
avc: denied { execmem } ... comm="i386-linux-gnu-"   scontext=...:unconfined_t
```

Writable-and-executable memory: JITs, and Steam's `inspect-library` probe. Note
the majority of these come from **Cinnamon itself**, not Steam — the desktop
needs it regardless. The `allow_execmem` boolean covers it and ships `off`:

```yaml
selinux_booleans:
  allow_execmem: true
```

**2. `nnp_transition` — needs a local module.**

```
avc: denied { nnp_transition nosuid_transition } for comm="pv-adverb"
  scontext=...:unconfined_t  tcontext=...:ldconfig_t  tclass=process2
```

`pv-adverb` and `steam-runtime-inspect-library` are pressure-vessel, Steam's
container runtime. It sets `PR_SET_NO_NEW_PRIVS`, and the policy capability
`nnp_nosuid_transition` (on in this policy) then makes the kernel check
`process2:nnp_transition` before letting `unconfined_t` transition to
`ldconfig_t`. There is **no boolean for it** — checked with `getsebool -a` — and
`sesearch -A -s unconfined_t -t ldconfig_t -c process2` returns nothing, so the
policy has no such rule. Under enforcing the transition fails and pressure-vessel
cannot set up the runtime.

This is the case that justifies a local module, and it is narrow enough to write
by hand rather than letting `audit2allow -M` generalise from whatever leaked:

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
games on.** Permissive costs nothing here: Steam works exactly as it did before
SELinux was enabled, and the log is what tells you this in advance.

## Escape hatches

In descending order of preference — the first two need only the GRUB menu:

| Situation | Do this |
|---|---|
| Enforcing broke something, box still boots | `setenforce 0`, then fix, then `setenforce 1` |
| Enforcing broke the boot | press `e` in GRUB, append `enforcing=0` to the `linux` line, F10. Works because `CONFIG_SECURITY_SELINUX_DEVELOP=y` |
| Need SELinux gone entirely for one boot | the **recovery** menu entry already omits `security=selinux` — or append `lsm=landlock,lockdown,yama,loadpin,safesetid,integrity,apparmor,bpf,ipe` by hand |
| Need SELinux gone for good | `selinux_enabled: false`, re-run `selinux.yml`, reboot. Removes the drop-in and unmasks AppArmor |

`selinux=0` is **not** on that list, and neither is `SELINUX=disabled`. See
*Three kernel facts*.

## Configuration

All of it in `group_vars/all.yml`; `roles/config_contract` asserts every key
exists and that the enumerated ones are legal.

| Variable | Default | Meaning |
|---|---|---|
| `selinux_enabled` | `true` | master switch. `false` actively undoes: drop-in removed, `/.autorelabel` removed, AppArmor unmasked |
| `selinux_mode` | `permissive` | `permissive` \| `enforcing`. Written to `/etc/selinux/config`. `disabled` is deliberately not a legal value |
| `selinux_policy_type` | `default` | `default` \| `mls`. Must match the `selinux-policy-*` package in `selinux_packages`; the role asserts it does |
| `selinux_relabel_mode` | `auto` | `auto` = relabel only when the target looks unlabelled \| `always` = every run \| `never` |
| `selinux_cmdline` | `security=selinux audit=1 audit_backlog_limit=8192` | appended to the GRUB drop-in *and* compiled into `update-uki` |
| `selinux_disable_apparmor` | `true` | disable + mask `apparmor.service` while SELinux is on |
| `selinux_install_auditd` | `true` | `auditd` + `audispd-plugins`. Turning this off while `audit=1` stays on the command line means kernel audit records go to the ring buffer with nothing collecting them |
| `selinux_booleans` | `{}` | `name: true/false` map, applied with `semanage boolean -m -N` |
| `selinux_packages` | see config | policy + tooling |
| `selinux_audit_packages` | `auditd`, `audispd-plugins` | |

### How `selinux_relabel_mode: auto` decides

It reads the extended attributes of `{{ target_mount }}/etc/fstab` and schedules
a relabel when `security.selinux` is not among them. That is a fact about the
filesystem rather than a stamp file the role wrote, so it stays correct across
reinstalls, restores from backup, and hand-editing. It does **not** notice a
policy change that moved file contexts — after changing `selinux_policy_type`,
adding `semanage fcontext` rules, or installing policy modules, use
`selinux_relabel_mode: always` once.

## Running it

Inside a full run, `finalize.yml` calls the role after `uki`, before the
post-tasks that rebuild the initramfs and regenerate `grub.cfg`.

On its own, afterwards — the way to flip mode, booleans or the master switch
without going near crypttab, GRUB passwords or the TPM2 enrollment:

```
sudo ansible-playbook selinux.yml -e target_mount=/     # on the booted system
sudo ansible-playbook selinux.yml                       # live ISO, /target mounted
sudo ansible-playbook site.yml --tags selinux -e target_mount=/
```

`selinux.yml` regenerates `grub.cfg` itself, and re-runs the `uki` role when
`uki_enabled` is true, because both carry the command line.

## Operating it

```
sestatus                          # mode, policy, whether the store is loaded
check-selinux-installation        # selinux-basics' own sanity tests; prints problems only
ls -Z /boot /home /var/log/audit  # labels present?
seinfo --stats                    # policy size, from setools
getenforce ; setenforce 0|1       # runtime mode, until reboot

ausearch -m avc -ts boot          # denials this boot
ausearch -m avc -ts recent | audit2allow -l
semanage boolean -l -C            # booleans changed from policy default
semanage fcontext -l -C           # local file-context rules
restorecon -Rvn /path             # -n = dry run: what would be relabelled
fixfiles -F restore               # full relabel, now, on a running system
```

A relabel that has to happen at boot instead:

```
sudo touch /.autorelabel ; echo -F | sudo tee /.autorelabel ; sudo reboot
```

## What this does not do

- **No custom policy modules.** The role installs and configures the reference
  policy; it does not compile local `.te` files. That is deliberate — a policy
  module is a per-machine artifact that has to be built against denials this tree
  cannot predict, and shipping a generated one would be shipping someone else's
  guess. `selinux-policy-dev` is installable if you want to write them.
- **No confined login users.** Debian's `default` policy ships the `unconfined`
  module enabled, so interactive logins land in `unconfined_t` and the desktop
  behaves. Confining users (`semanage login -m -s user_u __default__`) is a real
  hardening step and a real way to break Cinnamon; it belongs in a deliberate
  follow-up, not in an autoconfigure.
- **No MLS.** `selinux_policy_type: mls` is accepted and wired, but nothing here
  is tested against it.
