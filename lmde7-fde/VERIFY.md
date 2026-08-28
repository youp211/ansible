# VERIFY.md — on-device verification checklist

Run §1–6 from the live session after `finalize.yml`. §7 onward are post-reboot.

## 1. Partition map is exact-MiB

```
fdisk -l <disk>
```

| Part | Start (512 B sectors) | Size |
|---|---|---|
| p1 | 2048 | 1 GiB |
| p2 | 2099200 | 1 GiB |
| p3 | 4196352 | rest |

On 4Kn drives divide by 8 (256 / 262400 / 524544).

## 2. LUKS header

```
cryptsetup luksDump <disk-p3>
```

- Version 2; sector 4096 bytes; cipher `aes-xts-plain64`, 512 bit
- Keyslot 0: pbkdf **argon2id**, memory ~1048576
- Keyslot 1 + `Tokens: 0: systemd-tpm2` (PIN: true, PCRs: 7)

## 3. Stack and sizes

```
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS <disk>
```

`cryptroot` → root 64G, var 32G, var_log 16G, var_log_audit 8G, var_tmp 8G,
tmp 8G, swap = multiplier×RAM, home rest. `home` is f2fs, everything else ext4.

## 4. Target config (mount the root LV and inspect)

- `/etc/crypttab`: `cryptroot UUID=<p3-uuid> none luks,tries=3`
- `/etc/fstab`: UUID-based, all eight mounts + `tmpfs /dev/shm` with
  `nodev,nosuid,noexec`; `/var` has `nodev,nosuid` (no `noexec` — dpkg needs it)
- `/etc/default/grub.d/90-fde-hardening.cfg` and `91-fde-selinux.cfg` present
- `/etc/selinux/config` has `SELINUX=permissive`, `SELINUXTYPE=default`;
  `/.autorelabel` exists and contains `-F`
- `/etc/initramfs-tools/conf.d/resume`: `RESUME=/dev/mapper/<vg>-swap`

## 5. TPM2 PIN unlock — two things that must BOTH be true

Stock initramfs-tools cannot use a PIN-protected LUKS2 token (see README). The
workaround is `00_fde_tpm2_unlock`, which must be present **and** run before
`cryptroot`.

```
# a) token plugin + TCTI are inside the initrd
lsinitramfs /boot/initrd.img-* | grep -E 'token-systemd-tpm2|libtss2-tcti-device'

# b) our unlock script runs FIRST (finalize asserts this too)
tmp=$(mktemp -d); unmkinitramfs /boot/initrd.img-$(uname -r) "$tmp"
grep -n -E '00_fde_tpm2_unlock|local-top/cryptroot' \
  "$(find "$tmp" -path '*/scripts/local-top/ORDER')"
```

`00_fde_tpm2_unlock` must appear on an EARLIER line than `cryptroot`. If it is
missing entirely, the likeliest cause is a shell syntax error — initramfs-tools
silently drops any boot script that fails `sh -n`.

## 6. fscrypt is set up but inert

```
fscrypt status /home     # expect: "has 0 protectors and 0 policies"
```

## 7. First boot

The initramfs prompt should accept the **TPM2 PIN**. If it only ever asks for a
passphrase, the token plugin is not engaging (§5) — the recovery passphrase boots
you regardless; debug from the running system. A wrong PIN a few times falls back
to the passphrase, as expected.

## 8. Running system

```
findmnt -t ext4,f2fs,vfat,tmpfs -o TARGET,SOURCE,FSTYPE,OPTIONS
swapon --show          # multiplier×RAM, on /dev/dm-*
cat /sys/power/resume  # major:minor of the swap DM device, not 0:0
```

## 9. Hibernation (only with Secure Boot OFF)

```
mokutil --sb-state
systemctl hibernate
```

SB enabled ⇒ kernel lockdown ⇒ hibernate refused. That is the documented trade,
not a bug. With SB off: a full cycle back to your session, and the resume path
still demands PIN/passphrase because swap is inside the container.

## 10. Kernel cmdline

```
cat /proc/cmdline
```

Contains `init_on_alloc=1 init_on_free=1 slab_nomerge page_alloc.shuffle=1
randomize_kstack_offset=on pti=on vsyscall=none resume=/dev/mapper/<vg>-swap
security=selinux audit=1 audit_backlog_limit=8192`.

`security=selinux` is the only thing that enables SELinux on this kernel. On a
`uki_enabled` system read the UKI's own copy instead, since `grub.cfg`'s line is
not what booted:

```
objdump -s -j .cmdline /boot/efi/EFI/Linux/lmde-*.efi
```

## 11. AIDE (after `fde-aide-init.service` has finished)

```
systemctl status fde-aide-init.service        # inactive/dead once the baseline exists
ls -l /var/lib/aide/aide.db                   # the baseline, inside LUKS
systemctl status fde-aide-boot-check.service  # green = /boot + ESP match
systemctl list-timers dailyaidecheck.timer
```

The baseline takes minutes on first boot; until it lands the boot check skips
itself and reports `inactive` rather than failed. Prove the check actually fires:

```
sudo touch /boot/aide-canary
sudo systemctl start fde-aide-boot-check.service ; systemctl status fde-aide-boot-check
sudo rm /boot/aide-canary
sudo systemctl start fde-aide-boot-check.service   # green again
```

A failed unit after a kernel update is expected only if the accept hook did not
run. A failed unit with no update behind it is the alarm this role exists for.

## 12. SELinux

**Expect two reboots after finalize, not one.** Boot 1 relabels every filesystem
and reboots itself; boot 2 is the real one.

```
sestatus
```

Expected on boot 2: status `enabled`, policy `default`, mode `permissive` (or
`enforcing` once promoted), config file matching. Then the things `sestatus` does
not tell you:

```
aa-enabled                       # must fail / say not available: AppArmor lost the slot
ls -Zd / /boot /home /var/log/audit    # real types, not "?" and not unlabeled_t
check-selinux-installation       # silence == clean
systemctl status auditd          # running, logging to the var_log_audit LV
ls /.autorelabel                 # must be GONE — the relabel consumed it
```

`unlabeled_t` anywhere under `/` means the relabel did not finish: check
`journalctl -b -1 -u selinux-autorelabel` and re-run `sudo fixfiles -F restore`,
or re-run the playbook with `-e selinux_relabel_mode=always`.

Prove denials are being recorded — permissive is only worth anything if the log
is real:

```
ausearch -m avc -ts boot          # may legitimately be empty on a quiet desktop
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -l
```

Read those for a week before setting `selinux_mode: enforcing`. Escape hatch
smoke test, worth doing once while the desktop still works: press `e` in GRUB,
append `enforcing=0`, F10, then `getenforce` must say Permissive.

## 13. MOK, signed kernel, and the repo

```
mokutil --sb-state                          # Secure Boot on or off
mokutil --test-key /etc/uki/keys/MOK.der    # "is already enrolled" once MokManager ran
mokutil --list-new                          # non-empty = staged, MokManager pending
ls -l /etc/uki/keys/                        # MOK.key + MOK.pem must be 0600
```

If `--list-new` shows the key but `--test-key` does not, you have not been through
MokManager yet. Lost the password because the terminal scrolled? It is kept at
`/etc/uki/keys/mok-enroll-password` (0600) until enrollment is confirmed. If it is
gone and you still have not enrolled, `mokutil --revoke-import` cancels the
pending request and a re-run of `kernel.yml` stages a fresh one.

`finalize.yml` never enrols — a freshly finalized machine has `MOK.der` on disk
and an empty `--list-new`. That is correct.

The key is only useful if it actually signed something:

```
sbverify --cert /etc/uki/keys/MOK.crt /boot/vmlinuz-$(uname -r)
modinfo -F signer $(modinfo -n loop)        # the MOK's CN, not a random build key
dpkg -V linux-image-$(uname -r)             # silent: md5sums regenerated after signing
```

`modinfo -F signer` printing `Build time autogenerated kernel key` means
`kernel_sign_modules` did not take effect — the build fell back to the ephemeral
key. The `dpkg -V` check matters because an unsigned `.deb` means every other
machine pulling it from the repo gets an unsigned kernel.

Retention and the repo client:

```
/usr/local/sbin/fde-keep-kernels --status     # every installed kernel, and what booted
apt-get -s autoremove | grep -i linux-image   # must propose removing NOTHING
cat /etc/apt/sources.list.d/lmde7-fde-kernel.sources   # Signed-By:, never trusted=yes
apt-get update                                # no NO_PUBKEY, no "not signed"
```

## 14. Boot policy — what the menu will offer (BEFORE rebooting)

Run this after any playbook that touched `uki_boot_policy`:

```
awk -F"'" '/menuentry |submenu /{print $2}' /boot/grub/grub.cfg
```

Under `uki-only` that list must contain **at least one** UKI entry and no
`Debian GNU/Linux, with Linux …` entries. An empty result, or one with no UKI in
it, means do not reboot yet.

```
ls -l /etc/grub.d/10_linux                      # 0644 = suppressed, 0755 = stock
dpkg-statoverride --list /etc/grub.d/10_linux   # must print an entry, else dpkg
                                                # restores 0755 on the next upgrade
ls -1 /boot/efi/EFI/Linux/
for f in /boot/efi/EFI/Linux/*.efi; do
  sbverify --list "$f" >/dev/null 2>&1 && echo "SIGNED   $f" || echo "unsigned $f"
done
sudo /usr/local/sbin/update-uki | grep -E 'rebuilt|skipped'
df -h /boot/efi
```

`skipped 0 image(s) for space` is the answer you want. Anything else means the
ESP cut into `uki_retain` and fewer kernels are bootable than you asked for.

Where `kernel_hold_distro` is on:

```
cat /etc/apt/preferences.d/99-fde-kernel-hold.pref
apt-cache policy linux-image-amd64        # Candidate: (none)
apt-get -s upgrade | tail -1              # "0 to remove" — the pin never removes
```

A `Candidate:` that is not `(none)` means the origins in `kernel_hold_origins` do
not match this machine; `apt-cache policy` prints the ones it really has.

## 15. Secure Boot — before and after the firmware trip

**Before: is `append-mok` the right route here?** Two cheap measurements. If
either says a CA has to stay in `db`, `own-keys` buys nothing — see SECUREBOOT.md.

```
# 1. what actually signs the other OS (--list, NOT --cert)
sudo mount -o ro /dev/disk/by-id/<other-os-esp> /mnt/winesp
sudo sbverify --list /mnt/winesp/EFI/Microsoft/Boot/bootmgfw.efi

# 2. what signs the discrete GPU's option ROM
lspci -nn | grep -Ei 'vga|3d'
cat /sys/bus/pci/devices/0000:03:00.0/boot_vga     # 1 = the card firmware uses
```

The option ROM needs unpacking before `sbverify` will read it — SECUREBOOT.md has
the procedure.

**Before: what the role staged.**

```
sudo ls -l /boot/efi/EFI/keys/          # MOK.cer, MOK.der, ENROLL-MOK.txt
sudo cat /boot/efi/EFI/keys/ENROLL-MOK.txt
sudo openssl x509 -in /etc/uki/keys/MOK.crt -noout -fingerprint -sha256
```

The `openssl` fingerprint must match the one in `ENROLL-MOK.txt` **and** what the
firmware displays when you select the file. The key browser shows a fingerprint,
not a subject line, so that is the only check distinguishing the right
certificate from a wrong one.

Confirm every UKI is signed by the key firmware is about to trust — not just the
newest, since the older images are the fallback:

```
for u in /boot/efi/EFI/Linux/*.efi; do
  printf '%-44s ' "$(basename "$u")"
  sudo sbverify --cert /etc/uki/keys/MOK.crt "$u"
done
```

**After: did the append take?**

```
mokutil --db | grep -c 'Subject:'   # count: factory + 1
mokutil --sb-state                  # SecureBoot enabled
efibootmgr -v | grep -Ei 'windows|uki'
```

`db` must still contain the vendor and Microsoft certificates. If the count
dropped, *Replace* was selected instead of *Append* — fix it with **Restore
factory keys**, then append again. Note `mokutil --db` prints only x509 entries;
db also carries SHA-256 image hashes, which only a raw read shows:

```
sudo od -An -tx1 /sys/firmware/efi/efivars/db-d719b2cb-3d3a-4596-a3bc-dad00e67656f | head
```

An append can also silently drop a CA that firmware chose not to carry forward.
Compare the subject list against what the machine shipped with, and re-check that
anything depending on a dropped CA (a future GPU's option ROM, for instance) is
not something you need.

**After: re-seal the TPM2 keyslot.** Enabling Secure Boot moves PCR 7, so the
first boot asks for the recovery passphrase. That is expected, once. Re-seal from
the UKI you intend to keep booting:

```
sudo ansible-playbook tpm2.yml -e target_mount=/
```

Then reboot and confirm the PIN works. If it does not, check the lockout counter
before assuming a typo — `TPM2_PT_LOCKOUT_COUNTER: 0x0` means the TPM was never
asked at all, which is a policy problem (SECUREBOOT.md, "The auto-discovery
trap").
