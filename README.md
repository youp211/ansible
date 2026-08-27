# lmde7-fde — LUKS2 full-disk encryption prep + finalize for LMDE 7

Two-phase Ansible project run **from the LMDE 7 "Gigi" Cinnamon live ISO**.
You run the Mint installer yourself in between; these playbooks own everything
the installer can't do: the encrypted layout before it, and the boot plumbing
after it.

```
Phase 1  : prep.yml               -> GPT / LUKS2(argon2id) / LVM / mkfs / fscrypt
Phase 1.5: mount-for-installer.yml -> assembles the full layout at /target
Phase 2  : (you)                  -> installer, Manual Partitioning -> EXPERT MODE
Phase 3  : finalize.yml           -> crypttab, CIS fstab, bootloader, TPM2+PIN,
                                     resume=, hardened cmdline, initramfs,
                                     MOK, SELinux, AIDE
Later    : kernel.yml             -> signed upstream kernel .debs (+ apt repo)
```

## Resulting layout

```
<disk>
├─ p1  1024 MiB  ESP (FAT32)                    /boot/efi   start sector 2048
├─ p2  1024 MiB  ext4                           /boot       start sector 2099200
└─ p3  rest      LUKS2 aes-xts-plain64/512, argon2id (1 GiB, 4 s), 4096 B sectors
   └─ VG mint-debian13 (PE 4 MiB)
      ├─ root           64 GiB  ext4   /
      ├─ var            32 GiB  ext4   /var            nodev,nosuid
      ├─ var_log        16 GiB  ext4   /var/log        nodev,nosuid,noexec
      ├─ var_log_audit   8 GiB  ext4   /var/log/audit  nodev,nosuid,noexec
      ├─ var_tmp         8 GiB  ext4   /var/tmp        nodev,nosuid,noexec
      ├─ tmp             8 GiB  ext4   /tmp            nodev,nosuid,noexec
      ├─ swap    multiplier×RAM  swap   (2×32 GiB = 64 GiB on the test box)
      └─ home        95%FREE    f2fs   /home           nodev,nosuid  (fscrypt-ready)
```

Start sectors assume 512 B logical sectors; on 4Kn drives the playbook derives
and verifies the equivalent MiB-clean values automatically.

Fixed LVs total **136 GiB** plus swap. Swap sizing is mode-driven:
`swap_size_mode: ram` (default) makes it `swap_ram_multiplier` × exact
physical RAM bytes (`lsmem` total online, MiB-ceiled) — 2× on the 32 GiB test
box lands on exactly 64 GiB; `fixed` uses the size on the `lvs` swap entry.
Preflight refuses disks under `min_disk_size_gib` (**512**), refuses
hibernation with a multiplier below 1, and fit-checks the whole map so /home
keeps ≥ 64 GiB. Unlock chain: **TPM2 + PIN** (PCR 7) with the argon2id
**recovery passphrase in keyslot 0** as permanent fallback. `/home` gets f2fs
with the `encrypt` feature and fscrypt metadata initialised — **zero
protectors, zero policies**; nothing is encrypted at the directory layer until
you create a policy yourself later.

## Before you start

1. Boot the LMDE 7 Cinnamon ISO (UEFI mode), connect to the network.
2. Get Ansible + collections into the live session (RAM only):

   ```
   sudo apt update && sudo apt install -y ansible
   ```

   Debian's `ansible` package bundles `community.general` and
   `community.crypto`. (Alternative: `ansible-core` +
   `ansible-galaxy collection install -r requirements.yml`.)
3. Edit `group_vars/all.yml` — the destructive-run interlock requires all
   three, and prep refuses to run until they match reality:

   ```yaml
   target_disk: /dev/nvme0n1
   expected_disk_size_gib: 2048      # pre-set for the 2 TiB test box; for any
                                     # other disk: lsblk -bdno SIZE <disk> -> bytes // 2^30
   confirm_disk_wipe: true
   ```

   Prep prints a sizing summary (detected RAM → computed swap → /home
   headroom) before touching anything.

## Phase 1 — prep

```
cd lmde7-fde
sudo ansible-playbook prep.yml
```

Prompts once for the LUKS recovery passphrase. Ends with the container **left
open** and a cheat-sheet of what to assign in the installer.

## Phase 2 — Mint installer (Expert mode)

```
sudo ansible-playbook mount-for-installer.yml
```

This mounts the entire final layout — root, /boot, ESP, every CIS LV, and
f2fs /home — at `/target`. Then launch expert mode **from a terminal** (the GUI does not expose the
option): `sudo live-installer-expert-mode`. Assign nothing,
format nothing — expert mode installs into whatever is mounted at `/target`,
so the CIS LVs are populated directly and the mount-point picker never gets a
say. At **"Installation paused"** just continue: fstab, crypttab, and the
bootloader are finalize's job. When it finishes: **do not reboot.**

(The old assign-in-the-installer path still works where the picker allows it;
finalize's migration role backfills any LV the installer didn't populate
either way.)

## Phase 3 — finalize

```
sudo ansible-playbook finalize.yml
```

Prompts for the recovery passphrase (authorises TPM2 enrollment, reopens the
container if needed) and the new TPM2 PIN. Installs GRUB EFI + shim into the
target if the expert-mode install skipped the bootloader. Then work through
`VERIFY.md` before rebooting.

**Expect two reboots, not one.** SELinux is configured by default, and a
never-labelled filesystem has to be relabelled before the policy means
anything. The first boot comes up, relabels every filesystem, and reboots
itself; the second boot is the real one. `SELINUX.md` covers it.

Two roles can be re-run on their own afterwards, without touching crypttab,
the bootloader or the TPM2 enrollment:

```
sudo ansible-playbook selinux.yml -e target_mount=/    # mode, booleans, on/off
sudo ansible-playbook aide.yml    -e target_mount=/    # exclusions, re-baseline
sudo ansible-playbook kernel.yml  -e target_mount=/    # MOK enrollment, kernel build
```

## Decisions & caveats

**Secrets: same handling, deliberately different values.** Three secrets exist,
and the temptation to make them one password is worth resisting:

| Secret | Where | What protects it | Reuse cost |
|---|---|---|---|
| `luks_passphrase` | prompted by `prep`/`mount`/`finalize` | argon2id, 1 GiB / 4 s — and nothing else. No lockout, no rate limit | it *is* the disk |
| `tpm2_pin` | prompted by `finalize` | the TPM's dictionary-attack lockout | typed at every boot, so shoulder-surfable |
| MokManager password | **generated** by `roles/mok` | one-shot; hash lands in the root-readable `MokAuth` EFI variable | none — it is worthless after enrollment |

Making the PIN equal to the passphrase means one observed boot hands over full
offline access to the disk, because the TPM's anti-hammering protects the TPM
policy and not the LUKS keyslot. `roles/config_contract` refuses that
combination outright, along with either value being too short
(`luks_passphrase_min_length`, `tpm2_pin_min_length`).

The MokManager password is generated rather than chosen for the same reason in
reverse: it ends up as a hash in an EFI variable any root process can read, it
is consumed at the blue screen on the next boot, and it is never needed again —
so nothing you care about should ever be typed there. It is printed once; write
it down before rebooting, because MokManager runs before any filesystem is
mounted and the copy kept in `/etc/uki/keys/` is unreachable at that moment.

The *handling* is uniform: every prompt is `private`, confirmed wherever the
value is being set rather than checked, and every task that touches a secret
carries `no_log`. The one place that needs two secrets at once — the
`systemd-cryptenroll` call — passes them through a `0600` file on `/run` rather
than through `environment:`, because `no_log` does **not** suppress the
connection plugin's `EXEC` line and `-vvv` would otherwise print
`NEWPIN=<your PIN>` in the clear.

**One key signs everything, and the playbook makes it.** `roles/mok` generates a
single RSA keypair at `/etc/uki/keys/MOK.*` — on the root LV, so it lives inside
the LUKS2 container — and that one key signs the UKIs, the kernel image and every
module. `uki_sign_key`/`uki_sign_cert` are Jinja aliases of `mok_key`/`mok_cert`,
so the coupling is in the config rather than in your head. It is generated once
and never regenerated by a rerun: re-keying invalidates every signature already
issued and orphans the MOK enrolled in firmware.

Enrolling it is the one step no playbook can finish for you — shim wants a human
at the console. `kernel.yml` stages `mokutil --import` and prompts for the
password; you type the same one into MokManager at the next boot. `KERNEL.md`
has the whole chain, including why a kernel build cannot run in `/tmp` or
`/var/tmp` (this project's own fstab mounts both `noexec`, and a kernel build
executes the tools it compiles out of its own source tree).

**SELinux replaces AppArmor, and starts permissive.** LUKS2 protects the
machine when it is off and AIDE reports on it after the fact; neither does
anything while the machine is running. That is MAC's job, and LMDE's AppArmor
covers the couple of dozen daemons Debian ships profiles for. SELinux is
label-based and covers everything. The two are mutually exclusive — both set
`LSM_FLAG_EXCLUSIVE`, so the kernel initialises exactly one — so `selinux.yml`
hands the slot over explicitly and hands it back just as explicitly when
`selinux_enabled: false`.

It ships `permissive` on purpose. A refpolicy desktop that has never been run
permissive is a desktop you cannot log into, and on this box a broken boot costs
a passphrase dance. Permissive loads the policy, labels everything and logs
every would-be denial, so the promotion to enforcing is backed by an audit log.
`SELINUX.md` has the promotion path and the escape hatches — note that
`selinux=0` is **not** one of them on Debian 13 kernels (`enforcing=0` is).

**Why /boot is unencrypted.** Trixie ships GRUB 2.12, which cannot unlock
argon2id LUKS2 (that lands in 2.14). Encrypted /boot would force a weak
PBKDF2 keyslot — a downgrade of the whole container to its weakest slot — and
your TPM2+PIN unlock happens in the initramfs, where GRUB can't participate
anyway. Kernel/initrd integrity is Secure Boot's job, not obscurity's.

**Secure Boot vs hibernation — pick one.** Stock Debian/LMDE kernels enter
lockdown (integrity) when Secure Boot is on, and lockdown blocks hibernation.
`hibernate_enabled: true` wires `resume=` regardless; it simply won't fire
until SB is off. Your call which side of that trade you want.

**TPM2 PIN needs a workaround on initramfs-tools — and it is installed.**
Debian/LMDE unlock the root device through initramfs-tools' `cryptroot`,
which calls `unlock_mapping()` in `/lib/cryptsetup/functions`. That function
always passes `--key-file` and never passes `--token-only`, `--token-type` or
`--token-id`. Per `cryptsetup-open(8)`, a LUKS2 token *protected by a PIN* is
not tried unless one of those three options is given — so a
`systemd-cryptenroll --tpm2-with-pin=yes` token is **never attempted at boot**
on a stock system, even though the plugin is sitting in the initramfs.
(`tpm2-device=` in crypttab does not help either: it is an unrecognised
option to the initramfs script — Debian #1031254. Non-root devices appear to
work only because those are unlocked post-boot by systemd-cryptsetup, which
does understand it.)

The fix here is `/etc/initramfs-tools/scripts/local-top/00_fde_tpm2_unlock`:
it runs before `cryptroot` (the name sorts first, and initramfs-tools orders
independent scripts by glob order), performs
`cryptsetup open --token-only --token-type systemd-tpm2`, and exits 0 on any
failure. `cryptroot`'s `setup_mapping()` returns early when the mapping
already exists, so a success is simply inherited and a failure falls through
to the normal passphrase prompt. There is no path in that script that can
stop the machine from booting. finalize hard-asserts both that the plugin is
in the initrd and that the script wins the ordering.

**What the TPM binding is and is not worth here.** With PCR 7 only and an
unsigned initramfs on an unencrypted `/boot`, TPM2+PIN is a strong defence
against *offline* attack (stolen disk or laptop) but not against an evil-maid
edit of the initramfs: PCR 7 measures Secure Boot policy, not your kernel or
initrd, so a tampered initrd still satisfies the seal and could simply
capture the PIN as you type it. Binding extra PCRs invalidates on every
kernel update and is not a real fix. The real fix is the UKI path below —
a signed Unified Kernel Image folds kernel, initrd and cmdline into one
Secure Boot-verified object. If you are running custom keys via Redfish
anyway, that is the step that makes this TPM enrollment genuinely meaningful
rather than merely convenient.

**GRUB password (`grub_password_pbkdf2`)** is off by default. With
`superusers` set, GRUB requires the password to *boot* unmodified entries
too unless they're marked `--unrestricted` — don't enable this on a machine
that must boot unattended without reading up on that first.

**fscrypt later:** to actually encrypt something, e.g.
`fscrypt encrypt /home/<dir> --user=<you>` (with `libpam-fscrypt` installed,
login-passphrase protectors unlock at login). Until then /home behaves as
plain f2fs.

**UKI later (recommended: after first boot verifies the token path):** on
the booted system, no live ISO needed —
`sudo ansible-playbook finalize.yml -e uki_enabled=true -e target_mount=/ -e finalize_teardown=false`.
The passphrase/PIN prompts are inert on a rerun (enrollment skips when the
token exists), teardown refuses `target_mount=/`, and every other role
no-ops idempotently. UKIs land
in `ESP/EFI/Linux/`, GRUB chainloads them via `25_uki`, and kernel hooks keep
them current. `uki_sign` + your MOK key/cert enables sbsign for SB.

**AIDE watches what FDE cannot.** `/boot` and the ESP are unencrypted and
unsigned by design (see above), which leaves an evil-maid edit of the initramfs
as the live gap in the threat model. `roles/aide` installs AIDE with its
database on the `var` LV — *inside* the container — so the hashes cannot be
rewritten by whoever rewrites `/boot`. `fde-aide-boot-check.service` checks the
boot chain on every boot; kernel and initramfs hooks re-baseline it after your
own updates; Debian's `dailyaidecheck.timer` covers the rest of the system
nightly. The baseline is taken on the first boot of the installed system, not in
the chroot, because finalize rebuilds the initramfs and `grub.cfg` after the
roles run. Reconfigure it any time without touching the FDE plumbing:
`sudo ansible-playbook aide.yml -e target_mount=/`. Details, the accept
workflow and every tunable: **AIDE.md**.

**Config contract.** `group_vars/all.yml` is the only place values live —
the roles reference variables bare, with no inline `| default(...)`, so a
change there is guaranteed to take effect everywhere and nothing can silently
override it from inside a task. `roles/config_contract` runs first in
finalize and asserts every required key is present, naming any that are
missing. If you extract an updated tree over an edited config, that assert is
what tells you exactly which keys to add, in the first two seconds of the run
rather than halfway through.

**TPM2 unlock at boot** is controlled by `tpm2_boot_unlock`: `always` (PIN
prompt every boot, passphrase still the fallback), `cmdline` (only when
`fde_tpm2` is on the kernel cmdline — press "e" in GRUB to test without
committing), or `never` (script not installed).

**Scope:** storage/boot-chain hardening only. OS-level CIS (auditd rules,
sysctl, sshd, etc.) is deliberately out of scope per spec. AIDE is the one
deliberate exception: it is there to watch the boot chain this project leaves
unencrypted, not as a general CIS control.
# ansible
