# KERNEL.md — self-built signed kernels, the MOK that signs them, and the repo that ships them

`roles/mok` owns one RSA keypair. `roles/kernel` builds an upstream kernel into
`.deb` packages, signs the image and the modules with that key, installs them,
and optionally pushes them to an apt repo other machines pull from. Both are
driven by `kernel.yml`.

`finalize.yml` runs `roles/mok` only, and only its *generation* half. Generating
is harmless anywhere; enrolling writes firmware state and puts a MokManager
screen in front of the next boot, and at finalize time the key signs nothing yet.
So `kernel.yml` invokes `enroll.yml` explicitly, as a post-task after the build —
the point at which there is something signed to enrol for.

## Why this exists

The FDE chain has a gap it names openly in `AIDE.md`: `/boot` and the ESP are
unencrypted and unsigned, TPM2 seals to PCR 7 (Secure Boot *policy*, not the
kernel), and AIDE reports tampering after the fact. The prevention story is the
UKI path — and a UKI is only worth anything if it is *signed by a key the
firmware trusts*.

That key is the MOK, and the same key being behind all three artifacts is the
point:

| Artifact | Signed with | Verified by |
|---|---|---|
| UKI (`uki_enabled`) | `sbsign` | shim, against MokList — or firmware, against db |
| `vmlinuz` (classic GRUB path) | `sbsign` | shim's verify protocol, via GRUB |
| every `.ko` | `CONFIG_MODULE_SIG_KEY` at `modules_install` | the kernel's own keyring |

Upstream kernels are the other half. Debian's kernel is signed by Debian, so it
Just Works under Secure Boot — and is also months behind. Building your own means
signing your own, which means owning a key, which means enrolling it.

## The MOK

One keypair, generated once, never regenerated:

```
/etc/uki/keys/MOK.key    RSA private key, mode 0600  (on the root LV, inside LUKS)
/etc/uki/keys/MOK.crt    X.509 cert, PEM  — sbsign, CONFIG_SYSTEM_TRUSTED_KEYS
/etc/uki/keys/MOK.der    the same cert, DER — what mokutil enrols
/etc/uki/keys/MOK.pem    key + cert concatenated — CONFIG_MODULE_SIG_KEY
```

`mok_key` / `mok_cert` are canonical; `uki_sign_key` and `uki_sign_cert` are
Jinja aliases of them in `group_vars/all.yml`, so the same key reaches the UKI
role without the path being written twice.

**The generation task is guarded by `creates:` and will never overwrite an
existing key.** Regenerating is not a rerun, it is a break: every UKI, kernel and
module already signed becomes unverifiable, and the enrolled MOK stops matching.
To roll the key deliberately, move the directory aside by hand, re-run, re-enrol.

The private key lives inside the LUKS2 container, so an attacker with the
powered-off disk cannot sign a kernel with it. An attacker with root on the
running machine can, which is the ordinary limit of Secure Boot.

### Providing your own key instead

Because generation is guarded by `creates:`, an existing key is always reused.
That is the whole "provided" path — drop your key and certificate at `mok_key` /
`mok_cert` before the run and the role derives `MOK.der` and `MOK.pem` from them
(also under `creates:`, so supplying all four by hand works too). The certificate
needs `CA:FALSE` and a `codeSigning` EKU, which is what the role generates when
you do not supply one.

### Enrolling it

`mokutil --import` is deliberately not fully automatable: shim requires a human
at the console. What the playbook can do is stage it non-interactively:

```
mokutil --generate-hash                            # driven with expect
mokutil --import MOK.der --hash-file <0600 temp>   # documented non-interactive form
```

The password is **generated**, not prompted for — alphanumeric minus the glyphs
that are ambiguous when read off one screen and retyped into a UEFI console
(`I l 1 O o 0`). It is one-shot: MokManager consumes it at the next boot and its
hash lands in the `MokAuth` EFI variable, which any root process can read.
Nothing worth choosing belongs there.

It is printed once. **Write it down before rebooting** — MokManager runs before
any filesystem is mounted, so the copy kept at
`/etc/uki/keys/mok-enroll-password` is unreachable exactly when you need it. That
copy is deleted automatically on the first run after `mokutil --test-key`
confirms enrolment.

```
reboot -> blue MokManager screen -> Enroll MOK -> Continue -> Yes
       -> the generated password -> Reboot
```

Enrolment is a **firmware-level, machine-level** act, not a per-install one:
running it from the live ISO writes the same `MokNew` variable that running it
from the booted system would. Enrolling twice is harmless.

### When MOK enrolment stops being the point

`mok_enroll` should be `false` on a machine that has taken over its platform keys
(`SECUREBOOT.md`). MokList is *shim's* database, consulted only by shim — and a
custom `db` plus direct UKI boot removes shim from the chain entirely. A staged
import then sits in `MokNew` forever because MokManager never runs to consume it.
Note also that clearing the firmware Secure Boot keys wipes MokList along with
db/KEK/PK, so afterwards the enrolment probe correctly reports the key as absent;
it is the enrolment that is pointless, not the detection.

## The build

Rendered into `/usr/local/sbin/fde-kernel-build`: kernel.org fetch,
`sha256sums.asc` check, seed from the running kernel's config then
`olddefconfig`, `make bindeb-pkg`. What differs from a hand-rolled script:

| Changed | Why |
|---|---|
| build dir is `kernel_build_dir` | see below |
| three `scripts/config` edits before building | point module signing at the MOK instead of a throwaway key |
| `sbsign` on the image, inside the `.deb` | so the *published package* is signed, not just this machine's `/boot` |
| `dpkg-deb -R` / `-b` repack with regenerated `md5sums` | a signed vmlinuz changes its own checksum; stale `md5sums` makes `dpkg -V` cry forever |
| `-dbg` package dropped unless asked for | it was 1.0 GB on the last real build |

The three config edits, applied after `olddefconfig`:

```
CONFIG_MODULE_SIG_KEY="/etc/uki/keys/MOK.pem"        # was an ephemeral per-build key
CONFIG_MODULE_SIG_ALL=y                              # sign every module at modules_install
CONFIG_SYSTEM_TRUSTED_KEYS="/etc/uki/keys/MOK.crt"   # embed the cert in the builtin
                                                     # keyring so the kernel's own modules
                                                     # validate without the MOK having
                                                     # reached the platform keyring
```

`CONFIG_EFI_STUB=y` is what makes `vmlinuz` a real PE/COFF binary and therefore
`sbsign`-able. It is on in the seed config; the role asserts it rather than
assuming, because a kernel built without it fails signing at the very end of a
forty-minute build.

### The build directory problem

This is the part that bites, and it is this tree's own fault:

```
/tmp        8 GiB   nodev,nosuid,noexec     <- CIS policy from group_vars
/var/tmp    8 GiB   nodev,nosuid,noexec     <- same
/var       32 GiB   nodev,nosuid
/home    95%FREE    nodev,nosuid            <- the only one with room
```

A kernel build **cannot** run on a `noexec` filesystem. It is not a permissions
nicety: the build compiles and then *executes* its own tools out of the source
tree — `scripts/basic/fixdep`, `objtool`, `scripts/sorttable` — thousands of
times, and `noexec` turns that into `Permission denied` about ninety seconds in.
And it is big: with `CONFIG_DEBUG_INFO=y` the object tree runs 30–50 GB, which
does not fit in `/var` either.

Hence `kernel_build_dir: /home/kernel-build`, with two asserts that both name the
fix: the filesystem must not be `noexec`, and it must have
`kernel_build_min_free_gib` free (default 60). `kernel_debug_info: false` brings
the tree under 10 GB — it also takes `CONFIG_DEBUG_INFO_BTF` with it and silently
breaks `bpftrace`, `bcc` and anything else that wants BTF, so it is a trade to
make on purpose rather than inherit from a default.

## Interactions with the rest of the tree

**AIDE.** A kernel install rewrites `/boot`, and the existing
`/etc/kernel/postinst.d/zzz-fde-aide-accept` hook re-baselines it after
`zz-update-grub` runs. Nothing new was needed.

**SELinux.** `dpkg` does not relabel what it unpacks. New files under `/boot`
normally inherit the right type from the parent's transition rules, but "normally"
is not "always", so the role runs `restorecon -RF /boot` after installing when
`selinux_enabled`.

**UKI.** `/etc/kernel/postinst.d/zz-update-uki` rebuilds UKIs for every installed
kernel and `sbsign`s them when `uki_sign` is on. With the MOK generated
automatically, `uki_sign: true` stops requiring a manual key-making step.

**Kernel retention.** `kernel_keep_all` writes an apt `NeverAutoRemove` drop-in
so old kernels are never *removed*, and sets `GRUB_DISABLE_SUBMENU=y` through a
drop-in of this tree's own rather than `sed`-ing `/etc/default/grub`, which it
does not own. `kernel_hold_distro` is its mirror image — it stops distro kernels
being *added*. Together the installed kernel set becomes exactly what you put
there, which is what makes `uki_boot_policy: uki-only` a menu of your own signed
images. See **BOOTPOLICY.md**.

**Secure Boot vs hibernation.** SB on means kernel lockdown means no hibernation.
Signing your own kernel does not buy you out of that — lockdown keys off SB being
enabled, not off who signed the image.

## Server mode — shipping the packages

Off by default, with a fully populated example in `group_vars/all.yml`. Two
independent halves, because the machine that builds need not be one that
installs.

**Publisher** (`kernel_repo_publish`) — after a successful build, rsync the signed
`.deb`s to the repo host and fold them into a `reprepro` archive over ssh.
`reprepro` signs the `Release` file itself with whatever `SignWith:` names; the
playbook carries no GPG secret key, and should not. `kernel_repo_gpg_key` exists
only so the *client* half knows which public key to trust.

**Client** (`kernel_repo_client`) — point a machine at that repo:

```
/etc/apt/keyrings/lmde7-fde-kernel.asc            the repo's public key
/etc/apt/sources.list.d/lmde7-fde-kernel.sources  deb822, Signed-By: that keyring
```

deb822 with a per-repo keyring, not `apt-key` and not a bare `trusted=yes`: a repo
added with `trusted=yes` is a repo that can ship you an unsigned kernel, which
would undo the point of everything above.

## PCR 11, and why enrolment lives in finalize

With `tpm2_pcrs: "7"`, enrolling from the live ISO is correct — PCR 7 measures
Secure Boot policy, which is machine-level and identical either side of a boot.

Binding *directly* to PCR 11 breaks that silently. PCR 11 measures the UKI that
is running, and finalize runs from the live ISO and rebuilds the initramfs and
UKIs in `post_tasks`, *after* `roles/tpm2_enroll`. Either way the keyslot ends up
sealed to a value that never recurs, the TPM refuses to unseal, and there is no
error at enrolment time to tell you.

The **signed PCR policy** removes the problem rather than working around it: the
keyslot binds to a public key, not a live measurement, so it can be enrolled from
anywhere and survives kernel updates.

```yaml
tpm2_pcrs: "7"                 # direct binding — machine-level, safe from a live ISO
tpm2_public_key_pcrs: "11"     # signed binding — image-level, survives updates
uki_pcr_sign: true             # ukify embeds .pcrsig / .pcrpkey in every UKI
```

`roles/tpm2_enroll` still refuses to seal if `tpm2_pcrs` itself names PCR 11 from
an environment that is not the sealed image. Read SECUREBOOT.md before enabling
the signed policy — it needs a systemd initrd, which Debian's initramfs-tools is
not.

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `mok_enabled` | `true` | generate the keypair if absent |
| `mok_dir` / `mok_key` / `mok_cert` / `mok_der` / `mok_pem` | `/etc/uki/keys/MOK.*` | where the key lives; `uki_sign_key`/`uki_sign_cert` alias these |
| `mok_cn`, `mok_days`, `mok_key_bits` | see config | cert subject / lifetime / size |
| `mok_password_length` | `24` | length of the generated one-shot MokManager password |
| `mok_enroll` | `true` | let `kernel.yml` stage `mokutil --import`. `finalize.yml` never enrols regardless |
| `kernel_build_enabled` | `false` | master switch. A build is a deliberate, forty-minute act |
| `kernel_version` | `""` | empty = latest stable from kernel.org |
| `kernel_localversion` | `""` | appended to the release string |
| `kernel_build_dir` | `/home/kernel-build` | must not be `noexec`, must be big |
| `kernel_build_min_free_gib` | `60` | asserted before the build starts, not after |
| `kernel_debug_info` | `true` | `false` shrinks the tree ~5x and loses BTF |
| `kernel_sign_image` / `kernel_sign_modules` | `true` | `sbsign` the vmlinuz / point module signing at the MOK |
| `kernel_install` | `true` | `dpkg -i` the built packages |
| `kernel_keep_all` | `true` | apt `NeverAutoRemove` for kernels + un-submenu'd GRUB |
| `kernel_repo_publish` / `kernel_repo_client` | `false` | push to / consume from the repo host |

## Running it

```bash
# kernel.yml — the key and the plumbing. Never compiles unless you ask.
sudo ansible-playbook kernel.yml -e target_mount=/

# kernel-build.yml — compile, sign, install. Running it IS the request,
# so it ignores kernel_build_enabled.
sudo ansible-playbook kernel-build.yml -e target_mount=/
sudo ansible-playbook kernel-build.yml -e target_mount=/ -e kernel_version=6.12.30
```

`kernel-build.yml` runs `roles/mok` first so the signing key exists before the
build needs it — yours if you supplied one, generated if not. It does **not**
enrol: a build needs the MOK on disk, not in firmware. Afterwards, if
`mok_enroll` staged one: reboot, work through MokManager, and confirm with
`mokutil --test-key /etc/uki/keys/MOK.der`.

## What this does not do

- **It does not build during finalize.** The live ISO is the wrong place: no
  toolchain, no room, and a forty-minute failure would sit between you and a
  working crypttab.
- **It does not carry a GPG secret key.** Repo signing happens on the repo host.
- **It does not enrol the MOK for you.** It cannot; shim requires a human at the
  console. That is the feature, not a limitation.
- **It does not roll the key.** Regenerating invalidates every signature already
  issued, so it is a deliberate manual act.
