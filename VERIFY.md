# VERIFY.md — on-device verification checklist

Run §1–6 from the live session after `finalize.yml` (before reboot if
`finalize_teardown: false`, otherwise reopen/mount or just verify §1–2 and do
the rest post-boot). §7+ are post-reboot.

## 1. Partition map is exact-MiB

```
fdisk -l <disk>
```

Expected (512 B logical sectors):

| Part | Start | Size |
|---|---|---|
| p1 | 2048 | 1 GiB |
| p2 | 2099200 | 1 GiB |
| p3 | 4196352 | rest |

On 4Kn drives divide by 8 (256 / 262400 / 524544).

## 2. LUKS header

```
cryptsetup luksDump <disk-p3>
```

- Version: 2; Sector: 4096 bytes; Cipher: aes-xts-plain64, 512 bit
- Keyslot 0: pbkdf **argon2id**, Memory ~1048576
- Keyslot 1 + `Tokens: 0: systemd-tpm2` (PIN: true, PCRs: 7)

## 3. Stack + sizes

```
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS <disk>
```

`cryptroot` → `mint--debian13-{root 64G, var 32G, var_log 16G,
var_log_audit 8G, var_tmp 8G, tmp 8G, swap = multiplier×RAM (64G on the
test box), home rest}`; home FSTYPE
`f2fs`, everything else ext4.

## 4. Target config (mount root LV anywhere and inspect)

- `/etc/crypttab`: `cryptroot UUID=<p3-uuid> none luks,tries=3`
- `/etc/fstab`: UUID-based, all eight mounts + `tmpfs /dev/shm` with
  `nodev,nosuid,noexec`; `/var` has `nodev,nosuid` (no noexec — dpkg needs it)
- `/etc/default/grub.d/90-fde-hardening.cfg` present
- `/etc/default/grub.d/91-fde-selinux.cfg` present; `/etc/selinux/config` has
  `SELINUX=permissive` and `SELINUXTYPE=default`; `/.autorelabel` exists and contains `-F`
- `/etc/initramfs-tools/conf.d/resume`: `RESUME=/dev/mapper/mint--debian13-swap`

## 5. TPM2 PIN unlock — the two things that must BOTH be true

Stock initramfs-tools cannot use a PIN-protected LUKS2 token (see README).
The playbook works around it with `00_fde_tpm2_unlock`, which must be present
AND must run before `cryptroot`.

```
# a) token plugin + TCTI are inside the initrd
lsinitramfs /boot/initrd.img-* | grep -E 'token-systemd-tpm2|libtss2-tcti-device'

# b) our unlock script runs FIRST (finalize asserts this too)
tmp=$(mktemp -d); unmkinitramfs /boot/initrd.img-$(uname -r) "$tmp"
grep -n -E '00_fde_tpm2_unlock|local-top/cryptroot' \
  "$(find "$tmp" -path '*/scripts/local-top/ORDER')"
```

`00_fde_tpm2_unlock` must appear on an EARLIER line than `cryptroot`. If it is
missing entirely, the most likely cause is a shell syntax error — initramfs-
tools silently drops any boot script that fails `sh -n`.

## 6. fscrypt is setup but inert

```
fscrypt status /home        # (chroot or post-boot)
```

Expected: `... has 0 protectors and 0 policies`.

## 7. First boot

- Initramfs prompt should accept the **TPM2 PIN**. If it only ever asks for a
  passphrase, the token plugin isn't engaging (§5) — the recovery passphrase
  boots you regardless; debug from the running system.
- Wrong PIN a few times → falls back to passphrase. Expected.

## 8. Running system

```
findmnt -t ext4,f2fs,vfat,tmpfs -o TARGET,SOURCE,FSTYPE,OPTIONS
swapon --show          # multiplier×RAM (64G @ 2× on 32 GiB) on /dev/dm-*
cat /sys/power/resume  # major:minor of the swap DM device, not 0:0
```

## 9. Hibernation (only with Secure Boot OFF)

```
mokutil --sb-state
systemctl hibernate
```

SB enabled ⇒ kernel lockdown ⇒ `hibernate` refused — that's the documented
trade, not a playbook bug. With SB off: full cycle back to your session, and
the resume path still demands PIN/passphrase (swap is inside the container).

## 10. Kernel cmdline

```
cat /proc/cmdline
```

Contains: `init_on_alloc=1 init_on_free=1 slab_nomerge page_alloc.shuffle=1
randomize_kstack_offset=on pti=on vsyscall=none resume=/dev/mapper/mint--debian13-swap
security=selinux audit=1 audit_backlog_limit=8192`

`security=selinux` is the only thing that enables SELinux on this kernel —
`CONFIG_SECURITY_SELINUX_BOOTPARAM` is not set, so `selinux=1` would be an
unknown parameter. On a `uki_enabled` system read the UKI's own copy instead,
since grub.cfg's line is not what booted:

```
objdump -s -j .cmdline /boot/efi/EFI/Linux/lmde-*.efi
```

## 11. AIDE (post-boot, after `fde-aide-init.service` has finished)

```
systemctl status fde-aide-init.service        # inactive/dead once the baseline exists
ls -l /var/lib/aide/aide.db                   # the baseline, on the var LV (inside LUKS)
systemctl status fde-aide-boot-check.service  # green = /boot + ESP match the baseline
systemctl list-timers dailyaidecheck.timer
```

The baseline hashes the whole system and takes minutes on first boot; until it
lands, `fde-aide-boot-check` skips itself (`ConditionPathExists`) and reports as
`inactive` rather than failed.

Prove the check actually fires:

```
sudo touch /boot/aide-canary
sudo systemctl start fde-aide-boot-check.service ; systemctl status fde-aide-boot-check
sudo rm /boot/aide-canary
sudo systemctl start fde-aide-boot-check.service   # green again
```

A failed unit after a kernel update is expected only if the accept hook did not
run — `sudo /usr/local/sbin/fde-aide-boot-accept`, then re-check. A failed unit
with no update behind it is the alarm this whole role exists for: read
`journalctl -u fde-aide-boot-check -b` before doing anything else, and see
AIDE.md.

## 12. SELinux (post-boot)

**Expect two reboots after finalize, not one.** Boot 1 comes up with SELinux
enabled and nothing labelled, sees `/.autorelabel`, relabels every filesystem
and reboots itself. Boot 2 is the real one. See SELINUX.md.

```
sestatus
```

Expected on boot 2:

| Field | Value |
|---|---|
| SELinux status | `enabled` |
| Loaded policy name | `default` |
| Current mode | `permissive` (or `enforcing` once you have promoted) |
| Policy from config file | matches Current mode |

Then the things `sestatus` does not tell you:

```
aa-enabled                       # must say "No - not available" / error: AppArmor lost the LSM slot
ls -Zd / /boot /home /var/log/audit    # real types, not "?" and not unlabeled_t
check-selinux-installation       # selinux-basics' own tests; silence == clean
systemctl status auditd          # running, logging to the var_log_audit LV
ls -l /var/log/audit/audit.log
ls /.autorelabel                 # must be GONE — the relabel consumed it
```

`ls -Z` showing `unlabeled_t` anywhere under `/` means the relabel did not
happen or did not finish: check `journalctl -b -1 -u selinux-autorelabel` and
re-run `sudo fixfiles -F restore`, or re-run the playbook with
`-e selinux_relabel_mode=always` and reboot.

Prove denials are being recorded — permissive mode is only worth anything if the
log is real:

```
ausearch -m avc -ts boot          # may legitimately be empty on a quiet desktop
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -l
```

Read those for a week before setting `selinux_mode: enforcing`. The promotion
path, the boolean/fcontext decision tree and the escape hatches are all in
SELINUX.md.

Escape hatch smoke test, worth doing once while you still have a working
desktop — press `e` in GRUB, append `enforcing=0` to the `linux` line, F10:

```
getenforce      # Permissive, regardless of /etc/selinux/config
```

`selinux=0` is **not** an escape hatch on this kernel. The recovery menu entry
is: it omits `GRUB_CMDLINE_LINUX_DEFAULT`, and that is where `security=selinux`
lives.

## 13. MOK, signed kernel, and the repo (post-boot)

```
mokutil --sb-state                          # Secure Boot on or off
mokutil --test-key /etc/uki/keys/MOK.der    # "is already enrolled" once MokManager ran
mokutil --list-new                          # non-empty = enrollment staged, MokManager pending
ls -l /etc/uki/keys/                        # MOK.key + MOK.pem must be 0600
```

If `--list-new` shows the key but `--test-key` does not, you have not been
through MokManager yet: reboot, take the blue screen, **Enroll MOK → Continue →
Yes**, enter the one-shot password `kernel.yml` printed, Reboot.

Lost the password because the terminal scrolled? It is kept at
`/etc/uki/keys/mok-enroll-password` (0600) until enrollment is confirmed, and
deleted on the first run after that. If it is gone and you still have not
enrolled, `mokutil --revoke-import` cancels the pending request and a re-run of
`kernel.yml` stages a fresh one.

Note that `finalize.yml` never enrolls — it only generates the key — so a
freshly finalized machine has `MOK.der` on disk and an empty `--list-new`. That
is correct, not a failure.

The key is only useful if it actually signed something:

```
sbverify --cert /etc/uki/keys/MOK.crt /boot/vmlinuz-$(uname -r)
modinfo -F signer $(modinfo -n loop)        # the MOK's CN, not a random build key
cat /proc/keys | grep -i 'lmde7-fde'        # in .builtin_trusted_keys / .platform
```

`modinfo -F signer` printing `Build time autogenerated kernel key` means
`kernel_sign_modules` did not take effect — the build fell back to the ephemeral
`certs/signing_key.pem`. Re-run the build and check that `/etc/uki/keys/MOK.pem`
was readable.

The `.deb` itself must be signed too, not just the installed `/boot` copy —
otherwise every other machine pulling it from the repo gets an unsigned kernel:

```
dpkg -V linux-image-$(uname -r)     # silent: md5sums were regenerated after signing
```

Kernel retention and the boot menu:

```
/usr/local/sbin/fde-keep-kernels --status   # every installed kernel, and what booted
cat /etc/apt/apt.conf.d/99-keep-all-kernels.conf
apt-get -s autoremove | grep -i linux-image   # must propose removing NOTHING
```

Repo client, where `kernel_repo_client` is on:

```
cat /etc/apt/sources.list.d/lmde7-fde-kernel.sources   # Signed-By:, never trusted=yes
apt-get update                                          # no NO_PUBKEY, no "not signed"
apt-cache policy | grep -A2 lmde7-fde
```

An `apt-get update` warning about a missing key means `kernel_repo_gpg_key` is
the wrong half of the pair, or the repo host's `reprepro` is not signing
`Release` at all. Do not paper over it with `trusted=yes` — see KERNEL.md.

