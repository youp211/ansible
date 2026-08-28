# SECUREBOOT.md — owning the platform keys

`roles/secureboot` makes firmware trust this machine's own boot images.
`secureboot_mode` picks between two routes:

- **`own-keys`** replaces the factory hierarchy with one you control: your PK,
  your KEK, and a `db` containing only the certificate that signs your UKIs. The
  end state is a machine whose firmware loads **nothing you did not sign**.
- **`append-mok`** leaves the factory hierarchy exactly as it shipped and adds
  one certificate to `db` — the MOK. Firmware then loads your UKIs *and*
  everything it already trusted.

This is the last piece of the chain: `KERNEL.md` produces a MOK-signed kernel and
UKI, and this is what makes firmware care.

## Why it is worth doing

The stock chain is `firmware → shim → GRUB → kernel`, and firmware trusts shim
because shim is signed by **Microsoft Corporation UEFI CA 2011**. That CA has
signed a great many third-party bootloaders, and any one of them that turns out
to be vulnerable is an evil-maid tool: drop a *validly signed* binary on the ESP,
chain to your own code, capture the passphrase as it is typed.

Removing that CA from `db` closes it — and breaks shim, which is fine, because
the UKI path does not need shim. `firmware → UKI` has no third-party CA anywhere
in it, and two properties fall out:

- **The initrd is signed.** In the GRUB chain it is not: Debian's GRUB verifies
  the kernel through shim's protocol and loads the initrd unverified.
- **The cmdline is signed.** No editing `init=/bin/sh` at a boot prompt.

## Two routes, and how to tell which one you want

They are not ranked — the right answer is a property of the machine, and it is
measurable before you commit.

| | `own-keys` | `append-mok` |
|---|---|---|
| factory PK/KEK/db | replaced | untouched |
| db ends up holding | your MOK (+ hedges) | everything it shipped with, **plus** your MOK |
| shim / third-party loaders | refused | still load |
| needs firmware setup mode | yes — clear the keys first | no |
| what the playbook writes | PK, KEK, db | nothing; it stages a file on the ESP |
| the actual enrolment | `efi-updatevar`, from Linux | a keystroke in firmware setup |

`append-mok` cannot write the variable and does not try. Appending to `db`
outside setup mode requires a KEK-signed update, and on a factory hierarchy the
KEK belongs to the board vendor. So the role does what software can do — verify
the certificate, stage it where the firmware's file browser can reach it, write
down the steps — and leaves the append to you. A db change gated on physical
presence is the design, not a workaround.

**Choose `own-keys` when db can be reduced to your key alone.** That is the whole
benefit: the 2011 CA leaves db, and with it every third-party bootloader it ever
signed.

**Choose `append-mok` when something on the machine forces a CA to stay in db
anyway.** Once it stays, `own-keys` trusts the same set the factory hierarchy did
and you have paid a key-clear for nothing.

### The option ROM that usually decides it

A discrete GPU carries its UEFI GOP driver in its option ROM, and under Secure
Boot that driver is verified against db like anything else. If it fails you lose
pre-OS display on that card. Almost all of them are signed under **Microsoft
Corporation UEFI CA 2011** — the same CA `own-keys` exists to remove. Read it off
the card rather than guessing:

```bash
echo 1 | sudo tee /sys/bus/pci/devices/0000:03:00.0/rom >/dev/null
sudo cat /sys/bus/pci/devices/0000:03:00.0/rom > gpu.rom
echo 0 | sudo tee /sys/bus/pci/devices/0000:03:00.0/rom >/dev/null
```

The ROM is a chain of PCI images; the one you want has code type **3** (EFI) in
its `PCIR` header, and its EFI header gives `EFIImageOffset` plus a compression
flag at offset `0x0c`. **If that flag is 1 the PE is EDK2-compressed**, `sbverify`
will say `Invalid DOS header magic`, and no packaged tool in trixie will
decompress it — the algorithm is EDK2 `BaseUefiDecompressLib` (LZ77 + Huffman,
`PBit=4` for EFI 1.1, 5 for Tiano). Once decompressed you get an `MZ` header and
`sbverify --list` works normally.

A single signature under the 2011 CA with no *Option ROM UEFI CA 2023* fallback
means db has to keep that CA, which means shim stays trusted whether you want it
or not, which means `own-keys` buys nothing.

The dual-boot half of the question is quicker and worth doing in the same sitting:

```bash
# by-id: disk enumeration order is not stable across reboots
sudo mount -o ro /dev/disk/by-id/<other-os-esp> /mnt/winesp
sudo sbverify --list /mnt/winesp/EFI/Microsoft/Boot/bootmgfw.efi
```

A `bootmgfw.efi` signed only by *Windows UEFI CA 2023* — not dual-signed with the
2011 PCA — makes that CA load-bearing too. Use `--list`, not `--cert`:
`sbverify --cert` will report "Signature verification OK" against several
different Microsoft CAs for the same binary, which is not the question you asked.

## What `append-mok` does

1. Asserts the MOK exists (`roles/mok` makes it).
2. Verifies **every** UKI on the ESP against `mok_cert`, not just the newest —
   the older images are the fallback, and a fallback firmware will not load is
   not a fallback.
3. Probes `mokutil --db` for `mok_cn`, so a rerun after the append reports
   "already in db". **`--db`, not `--list-enrolled`**: MokList is *shim's*
   database, and a MOK there does nothing for a machine whose firmware loads the
   UKI directly. Only db moves firmware.
4. Stages the certificate as **both** `MOK.cer` and `MOK.der` — identical DER
   bytes under two names, because vendor firmwares filter the key browser by
   extension and do not agree on which.
5. Renders `ENROLL-MOK.txt` beside them with the steps and the certificate's
   SHA-256 fingerprint. The firmware trip happens with no shell and no notes; the
   fingerprint is the only thing distinguishing the right file from a wrong one.

Then, in firmware setup: **Key Management → db Management → Append**. Never
*Replace* — that wipes the vendor and Microsoft certificates. Nothing here is
one-way: the appended certificate can be deleted in the same menu, and *Restore
factory keys* puts db back as it shipped.

## `own-keys`: the four variables

| Variable | Holds | Signed by | Enrolled |
|---|---|---|---|
| `PK` | Platform Key — one cert, owns the platform | itself | **last** |
| `KEK` | Key Exchange Keys — may update db/dbx | PK | second |
| `db` | what firmware will load | KEK | **first** |
| `dbx` | explicit revocations | KEK | (unused here) |

**Order is counter-intuitive: db, then KEK, then PK.** While `PK` is empty the
firmware is in *setup mode* and accepts unsigned writes to all of them. Writing
`PK` is what closes setup mode, so it must be last — enrol it first and every
subsequent write needs a signature chain that does not exist yet.

### Three keys, not one

`db` gets the **MOK** — the key that already signs your UKIs and modules, so the
images on the ESP become bootable without rebuilding anything. `PK` and `KEK` are
generated **separately**, and that is not ceremony: the signing key lives on the
running root because `update-uki` needs it unattended on every kernel install, so
it is exposed to anything that gets root. PK and KEK are needed only to *change
policy*. Compromise the signing key and you rotate `db` using KEK; the platform
stays yours. If PK were the signing key, compromising it would mean losing the
platform.

Keys live in `secureboot_dir` (`/etc/secureboot/keys`), mode `0600`, on the root
LV inside the LUKS2 container.

### Setup mode is the window

`efi-updatevar` can write these variables directly, but only in setup mode, and
getting there is a firmware action this playbook cannot perform: **clear the
Secure Boot keys in firmware setup** (often behind a BIOS admin password). The
role asserts setup mode before touching anything:

```
SetupMode = 1     -> enrolment possible
SetupMode = 0     -> factory (or already-owned) hierarchy in place; clear it first
```

While in setup mode the machine has **no Secure Boot protection at all**. It is a
state to pass through, not to sit in.

### The check that stops you bricking the boot

Before writing anything the role verifies that the certificate going into db
actually verifies the UKI you are currently booting — `sbverify --cert <db cert>
<newest UKI>`. Enrolling a db that cannot load your own boot image is the one
mistake here that costs a firmware trip, and it is entirely preventable.
`secureboot_require_bootable: false` overrides it, and there is no good reason to.

### What `own-keys` breaks, deliberately

None of this applies to `append-mok`, which removes nothing from `db`.

- **shim and GRUB stop loading under Secure Boot.** That is the intent, but it
  changes the fallback path: with SB on, the only thing that boots is a UKI you
  signed. Recovery is turning Secure Boot off, not picking another menu entry.
- **Option ROMs.** Discrete GPUs, Thunderbolt controllers, some NICs ship
  firmware signed by the vendor or by Microsoft's UEFI CA and may not initialise
  without it. `secureboot_keep_vendor_db: true` keeps the vendor certificate as a
  hedge, and is the default.
- **PCR 7 changes.** Replacing the hierarchy invalidates any TPM2 keyslot sealed
  to PCR 7. Expect the recovery passphrase once, then re-enrol — *after* the
  hierarchy is final, not between steps.

## Measured boot: the signed PCR policy

Sealing to PCR 7 alone has a hole. PCR 7 measures Secure Boot *policy* — PK, KEK,
db, dbx, the SB state, and the certificate used to verify each image. It does not
measure *which* image booted, so every UKI signed by the same key produces an
identical PCR 7 and an attacker can select an older signed kernel with known CVEs
from the firmware boot menu and the TPM will unseal exactly as it would for the
current one.

PCR 11 closes it: `systemd-stub` measures the UKI's own sections into it. Binding
*directly* to PCR 11 works (`tpm2_pcrs: "7+11"`) but is miserable to live with —
every kernel or initrd change invalidates the keyslot.

The signed policy binds to a **public key** instead of a value, and lets the TPM
accept any PCR 11 state signed by the matching private key:

```
build time   systemd-measure predicts PCR 11 for this exact UKI and signs it;
             ukify embeds the signature in .pcrsig and the public key in .pcrpkey
boot time    systemd-stub hands the signature to systemd-cryptsetup; the TPM
             verifies the current PCR 11 is covered by a signature from the key
             this keyslot trusts -> unseal
```

A kernel update re-signs PCR 11 as part of building the UKI, so the keyslot keeps
working with no re-enrolment. Three keys are now in play, deliberately all
different:

| Key | Signs | Consequence if stolen |
|---|---|---|
| `mok_key` | UKIs and kernel modules | attacker can produce a bootable image |
| `uki_pcr_key` | predicted PCR 11 values | attacker can make a tampered image unseal |
| `PK.key` / `KEK.key` | firmware policy updates | attacker owns the platform |

### It needs a systemd initrd

Measured the hard way. The signed policy requires `tpm2-pcr-signature.json` to be
readable when the volume is unlocked. `systemd-stub` does hand the signature
over, but across a **systemd-initrd** handoff — and Debian's initramfs-tools
produces a script-based initrd, which never receives it.

The result is nasty because nothing complains:

```
symptom      boot asks for the PIN and rejects it
evidence     TPM2_PT_LOCKOUT_COUNTER: 0x0   <- a wrong PIN would increment this
meaning      the TPM was never asked; the policy session could not be built at
             all, because the signature was missing
```

The enrolment gives no warning either, by documented design: if no signature file
is found, no safety verification is done — so an unusable keyslot is written
cleanly. `roles/tpm2_enroll` now inspects the initrd and refuses rather than
letting that happen again. To actually use a signed policy you need a systemd
initrd (dracut). Until then set `tpm2_public_key_pcrs: ""` and bind PCRs directly
with `tpm2_pcrs`, accepting a re-enrolment whenever those PCRs move.

### The auto-discovery trap

`systemd-cryptenroll` does not require `--tpm2-public-key` to use a signed policy.
If the flag is absent it looks for `tpm2-pcr-public-key.pem` in `/etc/systemd/`,
`/run/systemd/` and `/usr/lib/systemd/`. So parking the PCR public key at
`/etc/systemd/tpm2-pcr-public-key.pem` — the tidy, conventional place — means
**omitting the flag no longer disables the signed policy, it silently enables
it.** Observed: an enrolment run with no public-key flag anywhere still produced
a keyslot with `tpm2-pubkey` populated and `tpm2-pubkey-pcrs: 11`, and the PIN
went on failing with no hint why.

`uki_pcr_pub` is therefore deliberately **off** that search path and passed
explicitly when a signed policy is actually wanted. The tell, if you suspect it:

```
cryptsetup luksDump /dev/… | sed -n '/tpm2-pubkey:/,/tpm2-pubkey-pcrs/p'
```

Zero continuation lines means no signed policy; anything else means one is bound
whether you asked for it or not.

**What it does not solve.** The signing key lives on the running root because
`update-uki` needs it unattended, so root compromise means an attacker can sign a
PCR set for their own image. That is inherent to unattended rebuilds; the
alternative is signing kernels on another machine.

## Reruns

The role is idempotent, and getting there needed a specific shape. Once PK is
installed the firmware leaves setup mode, so a bare `assert SetupMode == 1` would
turn every later run into a *failure* rather than a no-op. It instead probes
whether PK already holds `<secureboot_cn_prefix> Platform Key` and, if so,
reports "platform already owned" and skips the whole enrolment block.

That guard is a single `when:` on a `block:`, deliberately. Per-task guards were
tried and were wrong: Ansible short-circuits a `when:` list in order, so a
condition referencing a register from an earlier skipped task is evaluated
*before* the guard that would have prevented it, and the play dies with
`object of type 'dict' has no attribute 'stdout'`.

## Recovery

| Situation | Do this |
|---|---|
| Something signed wrong, machine will not boot with SB on | turn Secure Boot **off** in firmware; everything boots again |
| Want the factory hierarchy back | firmware setup → **Restore factory keys** |
| Want to start the custom hierarchy over | firmware setup → clear keys → setup mode → re-run the role |

Nothing is one-way. Factory keys are held in firmware, not in `db`, so restoring
them does not depend on anything on disk.

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `secureboot_enabled` | `false` | master switch; the role no-ops otherwise |
| `secureboot_mode` | `own-keys` | `own-keys` replaces the hierarchy; `append-mok` keeps it and stages the MOK |
| `secureboot_esp_key_dir` | `EFI/keys` | where certificates are staged for the firmware's file browser, relative to `uki_esp_mount`. Must be on the ESP — firmware cannot read `secureboot_dir` |
| `secureboot_dir` | `/etc/secureboot/keys` | where PK/KEK live, `0600` |
| `secureboot_db_cert` | `{{ mok_cert }}` | what goes into db — the UKI signing key |
| `secureboot_keep_vendor_db` | `true` | also keep the existing vendor db certs |
| `secureboot_cn_prefix` | `lmde7-fde` | subject prefix for the generated PK/KEK |
| `secureboot_days` / `secureboot_key_bits` | `3650` / `4096` | PK/KEK lifetime and size |
| `secureboot_require_bootable` | `true` | refuse to hand firmware a certificate that cannot verify this machine's UKIs — the newest under `own-keys`, all of them under `append-mok` |
| `secureboot_enroll` | `false` | **`own-keys` only.** Actually write the variables. Generating keys is safe; writing firmware policy is not |

## Running it

```bash
# append-mok: stage the certificate and print the firmware steps. Writes no
# firmware state, ever — safe any time.
sudo ansible-playbook secureboot.yml -e target_mount=/

# own-keys: generate + stage, then commit.
sudo ansible-playbook secureboot.yml -e target_mount=/
sudo ansible-playbook secureboot.yml -e target_mount=/ -e secureboot_enroll=true
```

Afterwards, on **either** route: enable Secure Boot in firmware, boot the UKI,
then re-seal TPM2 against the new PCR 7 with `tpm2.yml`. Turning Secure Boot on
moves PCR 7 by itself, so expect the recovery passphrase once even if `db` did
not change.

## What this does not do

- **It does not manage dbx.** Revocation lists are a firmware-vendor concern.
- **It does not sign anything.** Signing is `roles/uki` and `roles/kernel`; this
  role only decides what firmware will accept.
- **It cannot clear the keys for you.** Entering setup mode is a firmware action
  by design — if software could clear PK, Secure Boot would be worthless.
