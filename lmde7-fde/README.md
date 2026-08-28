# lmde7-fde — LUKS2 full-disk encryption for LMDE 7 / Debian 13

Ansible that builds a full-disk-encrypted LMDE 7 ("Gigi", Debian 13/trixie)
system and then hardens its boot chain. You run the Mint installer yourself in
between; these playbooks own everything the installer cannot do — the encrypted
layout before it, and the boot plumbing after it.

```
Phase 1   prep.yml                -> GPT / LUKS2(argon2id) / LVM / mkfs / fscrypt
Phase 1.5 mount-for-installer.yml -> assembles the full layout at /target
Phase 2   (you)                   -> installer, Manual Partitioning -> EXPERT MODE
Phase 3   finalize.yml            -> crypttab, CIS fstab, TPM2+PIN, resume=,
                                     hardened cmdline, initramfs, UKI, SELinux, AIDE
Later     kernel.yml, uki.yml, secureboot.yml, selinux.yml, aide.yml, tpm2.yml
```

Eleven playbooks in all. **[PLAYBOOKS.md](PLAYBOOKS.md)** is the one-line index
of what each does and which must never run unprompted — start there if you just
want to know what to run.

## Resulting layout

```
<disk>
├─ p1  1024 MiB  ESP (FAT32)                    /boot/efi
├─ p2  1024 MiB  ext4                           /boot
└─ p3  rest      LUKS2 aes-xts-plain64/512, argon2id (1 GiB, 4 s), 4096 B sectors
   └─ VG mint-debian13
      ├─ root           64 GiB  ext4   /
      ├─ var            32 GiB  ext4   /var            nodev,nosuid
      ├─ var_log        16 GiB  ext4   /var/log        nodev,nosuid,noexec
      ├─ var_log_audit   8 GiB  ext4   /var/log/audit  nodev,nosuid,noexec
      ├─ var_tmp         8 GiB  ext4   /var/tmp        nodev,nosuid,noexec
      ├─ tmp             8 GiB  ext4   /tmp            nodev,nosuid,noexec
      ├─ swap    multiplier×RAM  swap
      └─ home        95%FREE    f2fs   /home           nodev,nosuid  (fscrypt-ready)
```

Partitions are MiB-aligned; on 4Kn drives the playbook derives and verifies the
equivalent values itself. Fixed LVs total 136 GiB plus swap, which is
`swap_ram_multiplier` × physical RAM by default. Preflight refuses disks under
`min_disk_size_gib`, refuses hibernation with a multiplier below 1, and
fit-checks the map so `/home` keeps ≥ 64 GiB.

Unlock chain: **TPM2 + PIN** (PCR 7), with the argon2id recovery passphrase in
keyslot 0 as permanent fallback. `/home` gets f2fs with the `encrypt` feature and
fscrypt metadata initialised — **zero protectors, zero policies**; nothing is
encrypted at the directory layer until you create a policy yourself.

## Before you start

1. Boot the LMDE 7 Cinnamon ISO in UEFI mode and connect to the network.
2. Get Ansible into the live session: `sudo apt update && sudo apt install -y
   ansible`. Debian's `ansible` package bundles `community.general` and
   `community.crypto` (alternative: `ansible-core` + `ansible-galaxy collection
   install -r requirements.yml`).
3. Edit `group_vars/all.yml`. Prep refuses to run until all three match reality:

   ```yaml
   target_disk: /dev/disk/by-id/nvme-<model>_<serial>   # by-id: /dev/nvmeXn1 names swap
   expected_disk_size_gib: 1863     # lsblk -bdno SIZE <disk> -> bytes // 2^30
   confirm_disk_wipe: true
   ```

   Prep prints a sizing summary (detected RAM → computed swap → `/home`
   headroom) before touching anything.

## The three phases

```bash
sudo ansible-playbook prep.yml                  # DESTROYS target_disk
sudo ansible-playbook mount-for-installer.yml   # assembles /target
sudo live-installer-expert-mode                 # from a terminal; the GUI hides it
sudo ansible-playbook finalize.yml              # then read VERIFY.md; do not reboot yet
```

Prep prompts once for the LUKS recovery passphrase and leaves the container
open. In expert mode, assign nothing and format nothing — it installs into
whatever is mounted at `/target`, so the CIS LVs are populated directly. At
**"Installation paused"** just continue: fstab, crypttab and the bootloader are
finalize's job.

Finalize prompts for the recovery passphrase (which authorises TPM2 enrollment)
and the new TPM2 PIN. Then work through **[VERIFY.md](VERIFY.md)** before
rebooting.

**Expect two reboots, not one.** SELinux ships enabled, and a never-labelled
filesystem must be relabelled before the policy means anything. Boot 1 relabels
and reboots itself; boot 2 is the real one.

Afterwards, the subsystems are re-runnable on the booted system without touching
crypttab, the bootloader or the TPM2 enrollment:

```bash
sudo ansible-playbook selinux.yml -e target_mount=/    # mode, booleans, on/off
sudo ansible-playbook aide.yml    -e target_mount=/    # exclusions, re-baseline
sudo ansible-playbook kernel.yml  -e target_mount=/    # MOK, apt policy, repo
```

## Decisions worth knowing

**Three secrets, deliberately different values.**

| Secret | Where | What protects it | Reuse cost |
|---|---|---|---|
| `luks_passphrase` | prompted by prep/mount/finalize | argon2id, 1 GiB / 4 s — and nothing else | it *is* the disk |
| `tpm2_pin` | prompted by finalize | the TPM's dictionary-attack lockout | typed every boot, shoulder-surfable |
| MokManager password | **generated** by `roles/mok` | one-shot; hash lands in a root-readable EFI variable | none — worthless after enrollment |

Making the PIN equal to the passphrase means one observed boot hands over full
offline access, because the TPM's anti-hammering protects the TPM policy, not
the LUKS keyslot. `roles/config_contract` refuses that combination, and refuses
either value being too short. Every prompt is `private`, every task touching a
secret carries `no_log`, and the one call needing two secrets at once passes
them through a `0600` file on `/run` rather than `environment:` — `no_log` does
not suppress the connection plugin's `EXEC` line, and `-vvv` would otherwise
print the PIN in the clear.

**One key signs everything, and the playbook makes it.** `roles/mok` generates a
single RSA keypair at `/etc/uki/keys/MOK.*` — on the root LV, inside the LUKS2
container — and it signs the UKIs, the kernel image and every module. It is
generated once and never regenerated by a rerun: re-keying invalidates every
signature already issued. Enrolling it is the one step no playbook can finish
for you; shim wants a human at the console. See **[KERNEL.md](KERNEL.md)**.

**Why `/boot` is unencrypted.** Trixie ships GRUB 2.12, which cannot unlock
argon2id LUKS2 (that lands in 2.14). Encrypted `/boot` would force a weak PBKDF2
keyslot — downgrading the whole container to its weakest slot — and TPM2 unlock
happens in the initramfs, where GRUB cannot participate anyway. Kernel/initrd
integrity is Secure Boot's job, not obscurity's.

**What the TPM binding is and is not worth.** With PCR 7 only and an unsigned
initramfs on an unencrypted `/boot`, TPM2+PIN is a strong defence against
*offline* attack but not against an evil maid editing the initramfs: PCR 7
measures Secure Boot policy, not your kernel. The real fix is the UKI path — a
signed Unified Kernel Image folds kernel, initrd and cmdline into one Secure
Boot-verified object. See **[SECUREBOOT.md](SECUREBOOT.md)**.

**TPM2 PIN needs a workaround on initramfs-tools, and it is installed.** Debian
unlocks root through initramfs-tools' `cryptroot`, whose `unlock_mapping()`
always passes `--key-file` and never `--token-only`, `--token-type` or
`--token-id`. Per `cryptsetup-open(8)` a LUKS2 token *protected by a PIN* is not
tried unless one of those is given, so a `--tpm2-with-pin=yes` token is never
attempted at boot on a stock system. (`tpm2-device=` in crypttab does not help:
it is unrecognised by the initramfs script — Debian #1031254.) The fix is
`/etc/initramfs-tools/scripts/local-top/00_fde_tpm2_unlock`, which runs before
`cryptroot`, does `cryptsetup open --token-only --token-type systemd-tpm2`, and
exits 0 on any failure. No path in it can stop the machine booting, and finalize
hard-asserts both that the plugin is in the initrd and that the script wins the
ordering.

**Secure Boot vs hibernation — pick one.** Stock kernels enter lockdown
(integrity) when Secure Boot is on, and lockdown blocks hibernation.
`hibernate_enabled: true` wires `resume=` regardless; it simply will not fire
until SB is off.

**SELinux replaces AppArmor, and starts permissive.** The two are mutually
exclusive (both set `LSM_FLAG_EXCLUSIVE`), so `selinux.yml` hands the slot over
explicitly and hands it back just as explicitly. Permissive is deliberate: a
refpolicy desktop that has never run permissive is a desktop you cannot log
into. See **[SELINUX.md](SELINUX.md)**.

**AIDE watches what FDE cannot.** `/boot` and the ESP are unencrypted by design,
which leaves an evil-maid initramfs edit as the live gap.
`roles/aide` puts the database on the `var` LV — inside the container — so the
hashes cannot be rewritten by whoever rewrites `/boot`. See
**[AIDE.md](AIDE.md)**.

**Once the UKIs boot, take the unsigned path out of the menu.** Debian's
`10_linux` keeps emitting classic kernel+initrd entries and sorts *before*
`25_uki`, so the unsigned, cmdline-editable entry is also the default one.
`uki_boot_policy` decides what the menu may offer, and `kernel_hold_distro`
decides which kernels may arrive. Both reversible in one variable:
**[BOOTPOLICY.md](BOOTPOLICY.md)**.

**Config contract.** `group_vars/all.yml` is the only place values live — roles
reference variables bare, with no inline `| default(...)`. `roles/config_contract`
runs first and asserts every required key is present, naming any that are
missing, so an updated tree over an edited config fails in two seconds rather
than halfway through.

**Scope:** storage and boot-chain hardening only. OS-level CIS (auditd rules,
sysctl, sshd) is deliberately out of scope. AIDE is the one exception — it is
there to watch the boot chain this project leaves unencrypted.
