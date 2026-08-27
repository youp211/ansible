# SECUREBOOT.md — owning the platform keys

`roles/secureboot` replaces the factory Secure Boot key hierarchy with one you
control: your own PK, your own KEK, and a `db` containing only the certificate
that signs your UKIs. The end state is a machine whose firmware will load
**nothing you did not sign**.

This is the last piece of the chain the rest of the tree builds. `KERNEL.md`
produces a MOK-signed kernel and a MOK-signed UKI; `SECUREBOOT.md` is what makes
the firmware actually care.

## Why, on this machine specifically

The stock chain is `firmware → shim → GRUB → kernel`, and firmware trusts shim
because shim is signed by **Microsoft Corporation UEFI CA 2011**. That CA has
signed a great many third-party bootloaders, and any one of them that turns out
to be vulnerable is an evil-maid tool: drop a *validly signed* binary on the ESP,
chain to your own code, capture the LUKS passphrase as it is typed.

Removing that CA from `db` closes it — and immediately breaks shim, because shim
is the thing it vouched for. Which is fine, because the UKI path does not need
shim: a Unified Kernel Image is kernel + initrd + cmdline in one signed PE
binary, loaded straight by firmware. `firmware → UKI` has no third-party CA
anywhere in it.

Two properties fall out of that, and they are the actual point:

- **The initrd is signed.** In the GRUB chain it is not — Debian's GRUB verifies
  the kernel through shim's protocol and loads the initrd unverified. A UKI puts
  the initrd inside the signature.
- **The cmdline is signed.** No editing `init=/bin/sh` at a boot prompt.

## The four variables

| Variable | Holds | Signed by | Enrolled |
|---|---|---|---|
| `PK` | Platform Key — one cert, owns the platform | itself | **last** |
| `KEK` | Key Exchange Keys — may update db/dbx | PK | second |
| `db` | what firmware will load | KEK | **first** |
| `dbx` | explicit revocations | KEK | (unused here) |

**Order matters and is counter-intuitive: db, then KEK, then PK.** While `PK` is
empty the firmware is in *setup mode* and accepts unsigned writes to all of them.
Writing `PK` is what closes setup mode, so it has to be the last thing you do —
enrol it first and every subsequent write needs a signature chain that does not
exist yet.

```
SetupMode=1 (PK empty)          -> anything may be written
   write db   (your signing cert)
   write KEK  (your KEK cert)
   write PK   (your PK cert)    -> SetupMode=0, hierarchy live
```

## Three keys, not one

`db` gets the **MOK** — the key `roles/mok` already generated and that already
signs your UKIs and kernel modules. That is the whole tie-in: the UKIs on the ESP
become bootable under Secure Boot without rebuilding anything.

`PK` and `KEK` are **generated separately** and are not the signing key. This is
not ceremony:

- The signing key lives on the running root filesystem because `update-uki` has
  to use it unattended on every kernel install. It is exposed to anything that
  gets root.
- PK and KEK are needed only to *change the policy*. If the signing key is ever
  compromised you rotate `db` using KEK and the platform stays yours. If PK were
  the signing key, compromising it would mean losing the platform itself.

Keys live in `secureboot_dir` (`/etc/secureboot/keys`), mode `0600`, on the root
LV inside the LUKS2 container — same reasoning as the MOK.

## Setup mode is the window

`efi-updatevar` can write these variables directly, but only while the firmware
is in setup mode. Getting there is a firmware action this playbook cannot
perform: **clear the Secure Boot keys in firmware setup** (HP: BIOS Admin
Password must be set first, then Security → Secure Boot Key Management → clear).

The role asserts setup mode before touching anything, and tells you this if it is
not open:

```
SetupMode = 1     -> enrolment possible
SetupMode = 0     -> factory (or already-owned) hierarchy in place; clear it first
```

While in setup mode the machine has **no Secure Boot protection at all**. It is a
transient state to pass through, not to sit in.

## The check that stops you bricking the boot

Before writing anything, the role verifies that **the certificate going into db
actually verifies the UKI you are currently booting**. Enrolling a db that cannot
load your own boot image is the one mistake here that costs a firmware trip, and
it is entirely preventable:

```
sbverify --cert <db cert> /boot/efi/EFI/Linux/<newest>.efi
```

If that fails the role refuses to enrol. `secureboot_require_bootable: false`
overrides it, and there is no good reason to.

## Measured boot: the signed PCR policy

Sealing the LUKS keyslot to PCR 7 alone has a hole. PCR 7 measures Secure Boot
*policy* — PK, KEK, db, dbx, the SB state, and the certificate used to verify
each image. It does **not** measure which image booted. Every UKI signed by the
same key therefore produces an identical PCR 7, so an attacker can select an
older signed kernel with known CVEs from the firmware boot menu and the TPM will
unseal exactly as it would for the current one.

PCR 11 is the one that closes it: `systemd-stub` measures the UKI's own sections
(`.linux`, `.initrd`, `.cmdline`, …) into it, so a different or tampered image
produces a different value.

Binding *directly* to PCR 11 works and is what `tpm2_pcrs: "7+11"` does — but it
is miserable to live with, because every kernel or initrd change moves PCR 11
and invalidates the keyslot. Each `kernel-build.yml` would cost a passphrase
boot and a `tpm2.yml` re-enrolment.

### How the signed policy avoids that

Instead of binding to a *value*, bind to a **public key**, and let the TPM accept
any PCR 11 state that has been signed by the matching private key:

```
build time   systemd-measure predicts PCR 11 for this exact UKI and signs it;
             ukify embeds the signature in .pcrsig and the public key in .pcrpkey
boot time    systemd-stub hands the signature to systemd-cryptsetup
             the TPM verifies: "is the current PCR 11 covered by a signature
             from the key this keyslot trusts?"  -> unseal
```

So a kernel update re-signs PCR 11 as part of building the UKI, and the keyslot
keeps working with **no re-enrolment**. An attacker cannot forge a signature for
a tampered image, and an old image is only bootable for as long as its signature
is still one you issued — which `uki_retain` controls by deleting it.

Three keys are now in play, and they are deliberately all different:

| Key | Signs | Consequence if stolen |
|---|---|---|
| `mok_key` | UKIs and kernel modules | attacker can produce a bootable image |
| `uki_pcr_key` | predicted PCR 11 values | attacker can make a tampered image unseal |
| `secureboot_dir/PK.key`, `KEK.key` | firmware policy updates | attacker owns the platform |

The PCR public key goes to `/etc/systemd/tpm2-pcr-public-key.pem`, which is one
of the paths `systemd-cryptenroll` searches by default — so the enrolment finds
it without being told.

### It also puts enrolment back where it belongs

Direct PCR 11 binding forced enrolment to happen after first boot, from the exact
image being sealed against (see `KERNEL.md`). The signed policy does not: the
binding is to a key, not to a live measurement, so `finalize.yml` can enrol
again. `tpm2_pcrs` returns to `"7"` for the direct part, and
`tpm2_public_key_pcrs: "11"` carries the image binding.

### It needs a systemd initrd, and this machine does not have one

Measured the hard way. The signed policy requires `tpm2-pcr-signature.json` to be
readable when the volume is unlocked — `systemd-cryptenroll(1)` searches
`/etc/systemd/`, `/run/systemd/`, `/usr/lib/systemd/`. `systemd-stub` does hand
the signature over, but across a **systemd-initrd** handoff. Debian's
initramfs-tools produces a script-based initrd (`init` plus
`scripts/local-top/`), which never receives it.

The result is nasty because nothing complains:

```
symptom      boot asks for the PIN and rejects it
evidence     TPM2_PT_LOCKOUT_COUNTER: 0x0   <- a wrong PIN would increment this
meaning      the TPM was never asked; the policy session could not be built
             at all, because the signature was missing
```

And the enrolment gives no warning either, by documented design:

> If a signature file is specified or found it is used to verify if the volume
> can be unlocked … before the new slot is written to disk. **If no signature
> file is specified or found no such safety verification is done.**

No signature present, so no safety check, so an unusable keyslot is written
cleanly. `roles/tpm2_enroll` now inspects the initrd and refuses rather than
letting that happen again.

To actually use a signed policy here you would need to move to a systemd initrd
(dracut). Until then `tpm2_public_key_pcrs: ""` and bind PCRs directly with
`tpm2_pcrs` — accepting a re-enrolment whenever those PCRs move.

### The auto-discovery trap

`systemd-cryptenroll` does not require `--tpm2-public-key` to use a signed
policy. From its man page:

> The `--tpm2-public-key=` option accepts a path to a PEM encoded RSA public key,
> to bind the encryption to. **If this is not specified explicitly, but a file
> `tpm2-pcr-public-key.pem` exists in one of the directories `/etc/systemd/`,
> `/run/systemd/`, `/usr/lib/systemd/`** (searched in this order) …

So parking the PCR public key at `/etc/systemd/tpm2-pcr-public-key.pem` — which
looks like the tidy, conventional place, and means enrolment "finds it without
being told" — has a consequence that is easy to miss: **omitting the flag no
longer disables the signed policy, it silently enables it.**

That is not hypothetical. After the signed policy was found unusable on this
initrd and `tpm2_public_key_pcrs` was set to `""`, the re-enrolment ran with

```
systemd-cryptenroll --tpm2-device=auto --tpm2-with-pin=yes --tpm2-pcrs=7 /dev/…
```

— no public-key flag anywhere, exactly as intended — and produced a keyslot with
`tpm2-pubkey` populated and `tpm2-pubkey-pcrs: 11` all the same. The PIN went on
failing, and the command line gave no hint why.

`uki_pcr_pub` is therefore `/etc/uki/keys/pcr-sign.pub`, deliberately **off** the
search path, and passed explicitly when a signed policy is actually wanted.
`ukify` takes the path as an argument, so nothing needs the conventional
location.

The tell, if you ever suspect it: dump the token and count the hex continuation
lines under `tpm2-pubkey:`. Zero means no signed policy; anything else means one
is bound whether you asked for it or not.

```
cryptsetup luksDump /dev/… | sed -n '/tpm2-pubkey:/,/tpm2-pubkey-pcrs/p'
```

### What it does not solve

The signing key lives on the running root, because `update-uki` needs it
unattended on every kernel install. Root compromise therefore means an attacker
can sign a PCR set for their own image. That is the same limit as the UKI signing
key and is inherent to unattended rebuilds — the alternative is signing kernels
on another machine.

## What this breaks, deliberately

**shim and GRUB stop loading under Secure Boot.** They are signed by CAs that
will no longer be in `db`. That is the intent, but it means the fallback path
changes: with SB on, the only thing that boots is a UKI you signed. Recovery is
turning Secure Boot off in firmware, not picking a different menu entry.

**Option ROMs.** Some hardware ships firmware blobs signed by the vendor or by
Microsoft's UEFI CA — discrete GPUs, Thunderbolt controllers, some NICs. With
those CAs gone from `db`, such a device may not initialise under Secure Boot.
This machine has already been running with the Microsoft UEFI CA removed, so its
option ROMs are evidently fine without it; `secureboot_keep_vendor_db: true`
keeps the HP certificate in `db` as a hedge, and is the default.

**PCR 7 changes.** It measures Secure Boot policy — PK, KEK, db, dbx and the SB
state. Replacing the hierarchy changes it, which invalidates any TPM2 keyslot
sealed to PCR 7. Expect to fall back to the recovery passphrase on the next boot
and re-enrol. Do that *after* the hierarchy is final, not between steps.

## Reruns

The role is idempotent, and getting there needed a specific shape. Once PK is
installed the firmware leaves setup mode, so a bare `assert SetupMode == 1`
turns every later run into a *failure* rather than a no-op. It instead probes
whether PK already holds `<secureboot_cn_prefix> Platform Key` and, if so,
reports "platform already owned" and skips the whole enrolment block.

That guard is a single `when:` on a `block:`, deliberately. Per-task guards were
tried and were wrong: Ansible short-circuits a `when:` list in order, so a
condition referencing a register from an earlier skipped task (`sb_uki.stdout`)
is evaluated *before* the guard that would have prevented it, and the play dies
with `object of type 'dict' has no attribute 'stdout'`.

Verified: two consecutive runs, `changed=0 failed=0`.

## Recovery

In order of preference:

| Situation | Do this |
|---|---|
| Something signed wrong, machine will not boot with SB on | turn Secure Boot **off** in firmware; everything boots again |
| Want the factory hierarchy back | firmware setup → **Restore factory keys**. Puts HP + Microsoft back exactly as shipped |
| Want to start the custom hierarchy over | firmware setup → clear keys → setup mode → re-run the role |

Nothing here is one-way. HP's factory keys are held in firmware, not in `db`, so
restoring them does not depend on anything on disk.

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `secureboot_enabled` | `false` | master switch; the role no-ops otherwise |
| `secureboot_dir` | `/etc/secureboot/keys` | where PK/KEK live, `0600` |
| `secureboot_db_cert` | `{{ mok_cert }}` | what goes into db — the UKI signing key |
| `secureboot_keep_vendor_db` | `true` | also keep the existing vendor (HP) db certs |
| `secureboot_cn_prefix` | `lmde7-fde` | subject prefix for the generated PK/KEK |
| `secureboot_days` | `3650` | PK/KEK lifetime |
| `secureboot_key_bits` | `4096` | PK/KEK size |
| `secureboot_require_bootable` | `true` | refuse to enrol a db that cannot verify the current UKI |
| `secureboot_enroll` | `false` | actually write the variables. Off by default: generating keys is safe, writing firmware policy is not |

## Running it

```
sudo ansible-playbook secureboot.yml -e target_mount=/                      # generate + stage only
sudo ansible-playbook secureboot.yml -e target_mount=/ -e secureboot_enroll=true
```

Generation and staging are safe to run at any time and change no firmware state.
Enrolment requires setup mode and is the step that commits.

Afterwards: enable Secure Boot in firmware, boot the UKI, then re-enrol TPM2
against the new PCR 7 (see `KERNEL.md`).

## What this does not do

- **It does not manage dbx.** Revocation lists are a firmware-vendor concern and
  a custom hierarchy with a single trusted cert has little use for one.
- **It does not sign anything.** Signing is `roles/uki` and `roles/kernel`; this
  role only decides what the firmware will accept.
- **It cannot clear the keys for you.** Entering setup mode is a firmware action,
  by design — if software could clear PK, Secure Boot would be worthless.
