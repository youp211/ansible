# KERNEL.md — self-built signed kernels, the MOK that signs them, and the repo that ships them

`roles/mok` owns one RSA keypair. `roles/kernel` builds an upstream kernel into
`.deb` packages, signs the image and the modules with that key, installs them,
and optionally pushes them to an apt repo other machines pull from.

Both are driven by `kernel.yml`. `finalize.yml` runs `roles/mok` only, and only
its *generation* half — a kernel build has no business happening inside a
live-ISO finalize, and the key has to exist before `roles/uki` can sign
anything.

Generating and enrolling are deliberately separate. Generating is harmless
anywhere. Enrolling writes firmware state and puts a MokManager screen in front
of the next boot, and at finalize time the key signs nothing yet — `uki_sign`
defaults off and no custom kernel exists. So `kernel.yml` invokes `enroll.yml`
explicitly, as a post-task after the build, which is the point at which there is
something signed to enroll for. Your first boot after `finalize.yml` therefore
has the SELinux relabel reboot and no MokManager screen.

## Why this exists

The FDE chain this tree builds has a gap it names openly in `AIDE.md`: `/boot`
and the ESP are unencrypted and unsigned, TPM2 seals to PCR 7 (Secure Boot
*policy*, not the kernel), and AIDE reports tampering after the fact. The
prevention story was always "the UKI path" — and a UKI is only worth anything if
it is *signed by a key the firmware trusts*.

That key is the MOK. Once it exists, it signs three different things, and the
same key being behind all of them is the point:

| Artifact | Signed with | Verified by |
|---|---|---|
| UKI (`uki_enabled`) | `sbsign` | shim, against MokList |
| `vmlinuz` (classic GRUB path) | `sbsign` | shim's verify protocol, via GRUB |
| every `.ko` | `CONFIG_MODULE_SIG_KEY` at `modules_install` | the kernel's own keyring |

Upstream kernels are the other half. Debian's kernel is signed by Debian, so it
Just Works under Secure Boot — and it is also months behind, which is why
`make-kernel-deb.sh` existed in the first place. Building your own means signing
your own, which means owning a key, which means enrolling it. That is the whole
chain this role closes.

## The MOK

One keypair, generated once, never regenerated:

```
/etc/uki/keys/MOK.key    RSA private key, mode 0600      (on the root LV, inside LUKS)
/etc/uki/keys/MOK.crt    X.509 cert, PEM  — sbsign, CONFIG_SYSTEM_TRUSTED_KEYS
/etc/uki/keys/MOK.der    the same cert, DER — what mokutil enrolls
/etc/uki/keys/MOK.pem    key + cert concatenated — CONFIG_MODULE_SIG_KEY
```

`mok_key` / `mok_cert` are the canonical variables; `uki_sign_key` and
`uki_sign_cert` are Jinja aliases of them in `group_vars/all.yml`, so the same
key reaches the UKI role without the path being written down twice.

**The generation task is guarded by `creates:` and will never overwrite an
existing key.** Regenerating is not a rerun, it is a break: every UKI, kernel and
module already signed becomes unverifiable, and the enrolled MOK in firmware
stops matching. To roll the key deliberately, move the directory aside by hand,
re-run, and re-enroll — the role will not do it for you.

The private key lives on the root LV, i.e. inside the LUKS2 container. An
attacker with the powered-off disk cannot sign a kernel with it. An attacker with
root on the running machine can, which is the ordinary limit of Secure Boot and
not something this tree pretends to fix.

### When MOK enrolment stops being the point

`mok_enroll` is `false` on a machine that has taken over its platform keys
(`SECUREBOOT.md`). MokList is *shim's* database, consulted only by shim — and a
custom `db` plus direct UKI boot removes shim from the chain entirely. A staged
import then lands in `MokNew` and sits there forever, because MokManager never
runs to consume it, and the role re-stages it on every run.

Note also that clearing the firmware Secure Boot keys wipes MokList along with
`db`/`KEK`/`PK`, so after that step the enrolment probe correctly reports the key
as *absent*. It is the enrolment that is pointless, not the detection.

Set it back to `true` if you return to a shim/GRUB boot path.

### Providing your own key instead

The generation task is guarded by `creates:` on `mok_key`, so **an existing key
is always reused, never regenerated**. That is the whole "provided" path: drop
your own key and certificate at `mok_key` / `mok_cert` (or point those variables
somewhere else) before the run, and the role derives the two forms the toolchain
needs from what you supplied —

```
cp mykey.key /etc/uki/keys/MOK.key   && chmod 600 /etc/uki/keys/MOK.key
cp mycert.crt /etc/uki/keys/MOK.crt
ansible-playbook kernel.yml -e target_mount=/ -e kernel_build_enabled=true
```

`MOK.der` (for `mokutil`) and `MOK.pem` (key+cert, for `CONFIG_MODULE_SIG_KEY`)
are derived from those, also under `creates:`, so supplying all four by hand
works too. The certificate needs `CA:FALSE` and a `codeSigning` EKU — what the
role generates when you do not supply one.

### Enrolling it

`mokutil --import` is deliberately not fully automatable: shim requires a human
at the console to confirm the enrollment in MokManager. What the playbook can do
is stage it non-interactively, and it does:

```
mokutil --generate-hash        # driven with expect; prompts "input password:" twice
mokutil --import MOK.der --hash-file <0600 temp>   # documented non-interactive form
```

The password is **generated**, not prompted for — alphanumeric minus the glyphs
that are ambiguous when you read them off one screen and retype them into a UEFI
console (`I l 1 O o 0`). It is one-shot: MokManager consumes it at the next boot
and it is never needed again, and its hash lands in the `MokAuth` EFI variable,
which any root process can read. Nothing worth choosing belongs there. See
README "Secrets".

It is printed once, loudly. **Write it down before rebooting** — MokManager runs
before any filesystem is mounted, so the copy kept at
`/etc/uki/keys/mok-enroll-password` is unreachable exactly when you need it. That
copy exists so a later run can remind you, and it is deleted automatically on the
first run after `mokutil --test-key` confirms the key is enrolled.

The generated password is what MokManager asks for at the next boot:

```
reboot -> blue MokManager screen -> Enroll MOK -> Continue -> Yes
       -> "password" = the one the playbook asked for -> Reboot
```

Skip it and nothing breaks *today* — an un-enrolled MOK just means the signatures
are ignored. It breaks the moment Secure Boot is on and you try to boot a kernel
only your key vouches for.

**Enrollment is a firmware-level, machine-level act, not a per-install one.**
Running it from the live ISO writes the same `MokNew` EFI variable that running
it from the booted system would. That is why `mok_enroll` works with
`target_mount=/target` as well as `/`, and why enrolling twice is harmless.

**Secure Boot is currently OFF on this workstation** (`mokutil --sb-state`). With
SB off, enrollment and image signing are both inert; module signature *checking*
still works, so `kernel_sign_modules` is doing real work either way.

## The build

Lifted from `~/git/scripts/kernel/build-latest-kernel.sh` and
`make-kernel-deb.sh` — same shape, same kernel.org fetch, same
`sha256sums.asc` check, same "seed from the running kernel's config then
`olddefconfig`" trick, same `make bindeb-pkg`. Rendered into
`/usr/local/sbin/fde-kernel-build`, which is the template that carries the
differences:

| Changed | Why |
|---|---|
| build dir is `kernel_build_dir`, not `$HOME/kernel-build` | see *The build directory problem* |
| three `scripts/config` edits before building | point module signing at the MOK instead of a throwaway key |
| `sbsign` on the image, inside the `.deb` | so the *published package* is signed, not just this machine's `/boot` |
| `dpkg-deb -R` / `-b` repack with regenerated `md5sums` | a signed vmlinuz changes its own checksum; leaving `md5sums` stale makes `dpkg -V` cry forever |
| `-dbg` package dropped unless asked for | it was 1.0 GB on the last real build |

The three config edits, applied with `scripts/config` after `olddefconfig`:

```
CONFIG_MODULE_SIG_KEY="/etc/uki/keys/MOK.pem"    # was "certs/signing_key.pem",
                                                 # an ephemeral key thrown away each build
CONFIG_MODULE_SIG_ALL=y                          # sign every module at modules_install
CONFIG_SYSTEM_TRUSTED_KEYS="/etc/uki/keys/MOK.crt"   # was "" — embed the cert in the
                                                 # kernel's builtin keyring so its own
                                                 # modules validate without the MOK
                                                 # having reached the platform keyring
```

`CONFIG_EFI_STUB=y` is what makes `vmlinuz` a real PE/COFF binary and therefore
`sbsign`-able. It is on in the seed config; the role asserts it rather than
assuming it, because a kernel built without it fails signing at the very end of a
forty-minute build.

### The build directory problem

This is the part that bites, and it is this repo's own fault:

```
/tmp        8 GiB   nodev,nosuid,noexec     <- CIS policy from group_vars
/var/tmp    8 GiB   nodev,nosuid,noexec     <- same
/var       32 GiB   nodev,nosuid
/home    95%FREE    nodev,nosuid            <- the only one with room
```

A kernel build **cannot** run on a `noexec` filesystem. It is not a permissions
nicety: the build compiles and then *executes* its own tools out of the source
tree — `scripts/basic/fixdep`, `objtool`, `scripts/sorttable` — thousands of
times. `noexec` turns that into `Permission denied` about ninety seconds in. So
`/tmp` and `/var/tmp`, the two obvious "temp dirs", are both disqualified by this
tree's own fstab.

And it is big: with `CONFIG_DEBUG_INFO=y` (on in the stock seed config) the
object tree runs 30–50 GB, which does not fit in `/var`'s 32 GiB either.

Hence `kernel_build_dir: /home/kernel-build`. Two asserts guard it, and both name
the fix:

- the filesystem behind it must not be mounted `noexec`
- it must have `kernel_build_min_free_gib` free (default 60)

If you would rather not spend 50 GB, `kernel_debug_info: false` turns off
`CONFIG_DEBUG_INFO` and brings the tree under 10 GB. It is not the default,
because it also takes `CONFIG_DEBUG_INFO_BTF` with it, and that silently breaks
`bpftrace`, `bcc` and anything else that wants BTF. Losing eBPF to save disk is a
trade you should make on purpose, not inherit from a default.

## Interactions with the rest of the tree

**AIDE.** A kernel install rewrites `/boot`. The existing
`/etc/kernel/postinst.d/zzz-fde-aide-accept` hook already re-baselines it after
`zz-update-grub` runs, so a signed-kernel install does not leave the boot check
red. Nothing new was needed — this is the hook doing exactly the job `AIDE.md`
describes.

**SELinux.** `dpkg` does not relabel what it unpacks. New files under `/boot`
normally inherit the right type from the parent directory's transition rules, but
"normally" is not "always", so the role runs `restorecon -RF /boot` after
installing when `selinux_enabled`. Cheap, and it keeps `ls -Z /boot` honest.

**UKI.** `/etc/kernel/postinst.d/zz-update-uki` already rebuilds UKIs for every
installed kernel, and `update-uki` already `sbsign`s them when `uki_sign` is on.
With the MOK now generated automatically, `uki_sign: true` stops requiring a
manual key-making step — that is the whole tie-in.

**Kernel retention.** `keep-all-kernels.sh` from the same source directory is
rendered as `/usr/local/sbin/fde-keep-kernels` and run after install when
`kernel_keep_all` is set, with one change: it does **not** touch
`GRUB_TIMEOUT`/`GRUB_TIMEOUT_STYLE` in `/etc/default/grub`, because this tree
owns the boot menu through `grub.d` drop-ins and a `sed` into the base file would
fight it. It writes the apt `NeverAutoRemove` drop-in and sets
`GRUB_DISABLE_SUBMENU=y` through a drop-in of our own.

**Secure Boot vs hibernation.** Unchanged and worth restating: SB on means kernel
lockdown means no hibernation. Signing your own kernel does not buy you out of
that — lockdown keys off SB being enabled, not off who signed the image.

## Server mode — shipping the packages

Off by default, with every knob present and a fully populated example in
`group_vars/all.yml`. Two independent halves:

**Publisher** (`kernel_repo_publish`) — after a successful build, rsync the
signed `.deb`s to the repo host and fold them into a `reprepro` archive over ssh:

```
rsync -e ssh --chmod=F644 <debs> repoadmin@repo.lan:/srv/apt/incoming/
ssh repoadmin@repo.lan reprepro -b /srv/apt/lmde7 includedeb gigi <debs>
```

`reprepro` signs the `Release` file itself with whatever `SignWith:` names in its
`conf/distributions` — the playbook does not carry a GPG secret key, and should
not. `kernel_repo_gpg_key` exists only so the *client* half knows which public
key to trust.

**Client** (`kernel_repo_client`) — point this machine at that repo:

```
/etc/apt/keyrings/lmde7-fde-kernel.asc          the repo's public key
/etc/apt/sources.list.d/lmde7-fde-kernel.sources  deb822, Signed-By: that keyring
```

deb822 format and a per-repo keyring, not `apt-key` and not a bare
`trusted=yes`: a repo added with `trusted=yes` is a repo that can ship you an
unsigned kernel, which would undo the entire point of the preceding sections.

The two halves are independent on purpose. The machine that builds does not have
to be a machine that installs, and vice versa.

## PCR 11, and why enrolment moved back into finalize

`finalize.yml` has always enrolled the TPM2 keyslot, and with `tpm2_pcrs: "7"`
that was correct: PCR 7 measures Secure Boot *policy*, which is machine-level and
identical from the live ISO and after boot.

Binding *directly* to PCR 11 broke that assumption, silently. PCR 11 measures the
UKI that is running, and finalize runs from the live ISO — and rebuilds the
initramfs and the UKIs in `post_tasks`, *after* `roles/tpm2_enroll`. Either way
the keyslot ends up sealed to a value that never recurs, the TPM refuses to
unseal, and there is **no error at enrolment time** to tell you.

The **signed PCR policy** removes the problem rather than working around it. The
keyslot binds to a public key, not to a live measurement, so it can be enrolled
from anywhere and survives kernel updates. Configuration is therefore:

```yaml
tpm2_pcrs: "7"                 # direct binding — machine-level, safe from a live ISO
tpm2_public_key_pcrs: "11"     # signed binding — image-level, survives updates
uki_pcr_sign: true             # ukify embeds .pcrsig / .pcrpkey in every UKI
```

`roles/tpm2_enroll` still refuses to seal if `tpm2_pcrs` itself names PCR 11 from
an environment that is not the sealed image — that guard remains for anyone who
sets it deliberately. `tpm2_public_key_pcrs` is exempt by construction.

`tpm2.yml` is still the right tool after `secureboot.yml`, because replacing
PK/KEK/db moves PCR 7 and *that* binding is a direct one.

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `mok_enabled` | `true` | generate the keypair if absent |
| `mok_dir` / `mok_key` / `mok_cert` / `mok_der` / `mok_pem` | `/etc/uki/keys/MOK.*` | where the key lives; `uki_sign_key`/`uki_sign_cert` alias these |
| `mok_cn`, `mok_days`, `mok_key_bits` | see config | cert subject / lifetime / size |
| `mok_password_length` | `24` | length of the generated one-shot MokManager password |
| `mok_enroll` | `true` | let `kernel.yml` stage `mokutil --import`; the one-shot password is generated and printed. `finalize.yml` never enrolls regardless |
| `kernel_build_enabled` | `false` | master switch. A build is a deliberate, forty-minute act |
| `kernel_version` | `""` | empty = latest stable from kernel.org |
| `kernel_localversion` | `""` | appended to the release string |
| `kernel_build_dir` | `/home/kernel-build` | must not be `noexec`, must be big |
| `kernel_build_min_free_gib` | `60` | asserted before the build starts, not after |
| `kernel_debug_info` | `true` | `false` shrinks the tree ~5x and loses BTF |
| `kernel_sign_image` / `kernel_sign_modules` | `true` | `sbsign` the vmlinuz / point module signing at the MOK |
| `kernel_install` | `true` | `dpkg -i` the built packages |
| `kernel_keep_all` | `true` | apt `NeverAutoRemove` for kernels + un-submenu'd GRUB |
| `kernel_repo_publish` | `false` | push to the repo host |
| `kernel_repo_client` | `false` | consume from the repo host |

## Running it

Two entry points, deliberately split:

```
# kernel.yml — the key and the plumbing. Never compiles unless you ask.
sudo ansible-playbook kernel.yml -e target_mount=/                 # MOK + enrollment + repo

# kernel-build.yml — compile, sign, install. Running it IS the request,
# so it ignores kernel_build_enabled.
sudo ansible-playbook kernel-build.yml -e target_mount=/
sudo ansible-playbook kernel-build.yml -e target_mount=/ -e kernel_version=6.12.30
sudo ansible-playbook kernel-build.yml -e target_mount=/ -e kernel_localversion=-fde
```

`kernel-build.yml` runs `roles/mok` first, so the signing key exists before the
build needs it — yours if you supplied one, generated if not. It does **not**
enroll: a build needs the MOK on disk, not in firmware.

Then, if `mok_enroll` staged an enrollment: reboot, work through MokManager, and
confirm with `mokutil --test-key /etc/uki/keys/MOK.der`.

`finalize.yml` runs `roles/mok`'s generation half and never `roles/kernel`,
and never enrolls.

## What this does not do

- **It does not build during finalize.** The live ISO is the wrong place: no
  toolchain, no room, and a forty-minute failure would sit between you and a
  working crypttab.
- **It does not carry a GPG secret key.** Repo signing happens on the repo host,
  by `reprepro`, with a key this playbook never sees.
- **It does not enroll the MOK for you.** It cannot; shim requires a human at the
  console. That is the feature, not a limitation.
- **It does not roll the key.** See *The MOK* — regenerating invalidates every
  signature already issued, so it is a deliberate manual act.
