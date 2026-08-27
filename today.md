# today.md — 2026-08-26 work log

Two features landed in `lmde7-fde`, each with a design doc written before any
code: **SELinux** (mandatory access control, replacing AppArmor) and
**MOK + signed kernels** (one signing key, a self-built upstream kernel, and an
apt repo to ship it over).

Nothing was executed against a real target. `prep.yml` was never run, no kernel
was built, no MOK was enrolled in firmware.

---

## 1. SELinux — `SELINUX.md`, `roles/selinux`, `selinux.yml`

Debian's reference policy installed and configured on the target, AppArmor
handed the LSM slot over, filesystem relabel scheduled, auditd wired up.

**Ships `permissive` on purpose.** A refpolicy Cinnamon desktop that has never
run permissive is a desktop you cannot log into, and on this box a broken boot
costs a passphrase dance. Permissive loads the policy, labels everything and logs
every would-be denial, so promotion to `enforcing` is backed by an audit log
rather than by hope.

### Three kernel facts that shaped the design

Checked against this workstation's own `/boot/config-*` — it *is* LMDE 7, so
these are the target's answers, not guesses:

| Fact | Consequence |
|---|---|
| `CONFIG_SECURITY_SELINUX_BOOTPARAM` **not set** | `selinux=0` is not an escape hatch here. `selinux=1` is an unknown parameter. |
| `CONFIG_SECURITY_SELINUX_DISABLE` **absent** (removed upstream in 6.4) | `SELINUX=disabled` is a no-op trap — so `disabled` is an *illegal* value for `selinux_mode`. Off is `selinux_enabled: false`, which removes the cmdline argument. |
| `CONFIG_SECURITY_SELINUX_DEVELOP=y` | `enforcing=0` **does** work, and is the documented escape hatch. |

AppArmor wins `CONFIG_LSM` by default, so `security=selinux` is the only switch
that changes it. AppArmor is disabled + masked, and **symmetrically unmasked**
when `selinux_enabled: false` — flipping the flag never strands the box with no
MAC at all.

### Load-bearing details

- **Its own GRUB drop-in** (`91-fde-selinux.cfg`), separate from
  `90-fde-hardening.cfg`, so `selinux.yml` can flip SELinux without re-running
  `target_config` — i.e. without going near crypttab, fstab or the GRUB password.
- **`_DEFAULT`, not `GRUB_CMDLINE_LINUX`**, so the *recovery* menu entry boots
  SELinux-free. A deliberate door back in.
- **`update-uki.j2` carries the same string.** A chainloaded UKI ignores
  `grub.cfg` entirely, so enabling UKIs would otherwise silently drop SELinux.
- **The cmdline drop-in is never written unless the policy is actually on disk.**
  An LSM with no rules to load is worse than no LSM.
- **AIDE ordering works out.** Debian's AIDE links `libselinux1` and its
  `InodeData` group includes `X` (the xattr group), so a relabel changes every
  `/boot` hash. The relabel boot never reaches `multi-user.target`, so the
  baseline lands on boot 2 — correct under the default `aide_init_mode:
  firstboot`. The role warns on `aide_init_mode: now`, where it would genuinely
  false-alarm.
- **Expect two reboots** after finalize: boot 1 relabels and reboots itself.

### Bug caught during testing

The `selinux_relabel_mode: never` branch reported *"already carries SELinux
labels"* on an unlabelled target — false, and misleading in exactly the case
where it matters. Split into two distinct messages, the unlabelled one a warning.

---

## 2. MOK + signed kernels — `KERNEL.md`, `roles/mok`, `roles/kernel`, `kernel.yml`

Tied in `~/git/scripts/kernel/make-kernel-deb.sh` and `build-latest-kernel.sh`:
same kernel.org fetch, same `sha256sums.asc` verification, same seed-config +
`olddefconfig` approach, same `make bindeb-pkg`.

### One key signs three things

`roles/mok` generates a single RSA-4096 keypair at `/etc/uki/keys/MOK.*` — on the
root LV, so inside the LUKS2 container.

| Artifact | Signed with | Verified by |
|---|---|---|
| UKI | `sbsign` | shim, against MokList |
| `vmlinuz` | `sbsign` | shim's verify protocol, via GRUB |
| every `.ko` | `CONFIG_MODULE_SIG_KEY` | the kernel's own keyring |

`uki_sign_key` / `uki_sign_cert` became **Jinja aliases** of `mok_key` /
`mok_cert`, so the coupling lives in the config instead of in your head — and
rule 1 ("one place") still holds. `uki_sign: true` no longer needs a hand-made
key.

Generation is `creates:`-guarded and **never re-keys on a rerun**: that would
invalidate every signature already issued and orphan the MOK enrolled in
firmware.

### The `tmp` dir trap — the real find

This tree's own CIS fstab is what bites:

```
/tmp        8 GiB   noexec     <- disqualified
/var/tmp    8 GiB   noexec     <- disqualified
/var       32 GiB              <- too small for a DEBUG_INFO tree
/home    95%FREE               <- the only one with room
```

A kernel build **cannot** run on `noexec`: it compiles its own tools
(`scripts/basic/fixdep`, `objtool`) and then executes them out of the source tree
thousands of times — `Permission denied` about ninety seconds in. Hence
`kernel_build_dir: /home/kernel-build`, guarded by two asserts (not-noexec, and
free space) that both name the fix.

### Signing goes *inside* the `.deb`

Signing `/boot` after install would leave the published package unsigned, so every
machine pulling from the repo would have to re-sign locally. The build script
repacks with `dpkg-deb -R`/`-b` and **regenerates `md5sums`** — a signed vmlinuz
changes its own checksum, and stale `md5sums` makes `dpkg -V` report the kernel as
tampered for the life of the installation.

Three `scripts/config` edits redirect signing away from the ephemeral key the
build otherwise invents and throws away:

```
CONFIG_MODULE_SIG_KEY="/etc/uki/keys/MOK.pem"      # was "certs/signing_key.pem"
CONFIG_MODULE_SIG_ALL=y
CONFIG_SYSTEM_TRUSTED_KEYS="/etc/uki/keys/MOK.crt" # was ""
```

### Enrollment

`mokutil --import` cannot be fully automated — shim wants a human at the console,
and that is the feature. The playbook stages it non-interactively
(`--generate-hash` driven with expect, then `--import --hash-file`) and prompts
for the password you type again at the blue MokManager screen.

Enrollment is firmware-level and machine-level, not per-install: doing it from the
live ISO writes the same `MokNew` variable, and doing it twice is harmless.

### Server mode

Two independent halves, both off by default, with a fully populated commented
example in `group_vars`:

- **publish** — rsync the signed debs, then `reprepro includedeb` over ssh.
  Carries **no GPG secret key**; reprepro signs `Release` on the host.
- **client** — deb822 source with `Signed-By:` and a per-repo keyring. An empty
  public key is **refused** rather than falling back to `trusted=yes`, which
  would let the repo ship an unsigned kernel and undo everything above.

---

## 3. Lint pass — `.ansible-lint`

`ansible-lint` was not installed and sudo needs a password, so it was built in a
throwaway venv. The first attempt in the `/tmp` scratchpad died with
`failed to map segment from shared object` — `noexec` blocking native `.so`
loads, the same constraint that dictated `kernel_build_dir`. Rebuilt in
`/run/user/1000` (exec-capable tmpfs, self-clearing) with
`--system-site-packages` so it lints against the real `ansible-core 2.19.4`.

**92 → 75 → 0.**

Fixed (17):

- 3 in today's roles — two `name[casing]`, and a 261-char line in
  `selinux/verify.yml` reflowed to a folded scalar (both branches confirmed to
  render identically afterwards).
- 6 `name[play]` in `site.yml` — `import_playbook` does accept `name:`, so the
  phases are labelled now.
- 8 cheap repo-wide — `name[casing]` consistency, trailing whitespace, `#-` →
  `# -` comment spacing.

The remaining 75 were all pre-existing and all intentional, so they are skipped
in `.ansible-lint` with a stated reason each:

| Rule | n | Reason |
|---|---|---|
| `var-naming[no-role-prefix]` | 39 | The rule assumes role-local vars. These are cross-role by design — `part_prefix` is consumed by 8 files, `swap_mapper` by 6. |
| `name[template]` | 23 | Task names are imperative sentences that say why; the interesting value is mid-sentence. |
| `yaml[commas]` + `yaml[line-length]` | 10 | The `lvs` block is a column-aligned table on purpose. |
| `command-instead-of-module` | 3 | `rsync -aHAXS` and `mount --bind` (not `--rbind`) are chosen for exact semantics the modules do not expose. |

The skip list was checked for over-breadth by dropping a deliberately broken
role into the tree: `no-changed-when`, `risky-shell-pipe`,
`risky-file-permissions`, `name[missing]`, `fqcn[action-core]`, `no-free-form`
and `role-name` all still fire. Nothing that catches a real defect was silenced.

The tree now passes at ansible-lint's **production** profile (it was meeting only
`min` before), 0 findings in 82 files.

---

## 4. Secrets — unified handling, distinct values

Prompted by "what are we using for the TPM PIN, and we should use the same
across all". Answer: `tpm2_pin`, a `vars_prompt` in `finalize.yml`, handed to
`systemd-cryptenroll` via the **`NEWPIN`** environment variable and folded into
the LUKS2 `systemd-tpm2` token's TPM policy. Its brute-force protection is the
TPM's dictionary-attack lockout, not argon2.

The *handling* is now uniform; the *values* are deliberately not:

| Secret | Source | Protected by | Reused? |
|---|---|---|---|
| `luks_passphrase` | prompted | argon2id 1 GiB / 4 s, no rate limit | no |
| `tpm2_pin` | prompted | TPM dictionary-attack lockout | no |
| MokManager password | **generated**, printed once | one-shot; hash sits in root-readable `MokAuth` | n/a |

Reuse is refused rather than discouraged: `config_contract` fails the run if the
PIN equals the passphrase, because the PIN is typed at every boot and the
passphrase is the offline break-glass — one observed boot would otherwise hand
over full offline disk access, and the TPM's anti-hammering does not extend to
the LUKS keyslot.

### `no_log` alone was not enough — the real finding

`roles/tpm2_enroll` had `# no_log: true` **commented out** on the one task that
holds both secrets at once. Uncommenting it turned out to be insufficient:

```
$ ansible-playbook ... -vvv          # WITH no_log: true
<localhost> EXEC /bin/sh -c 'NEWPIN=<the actual PIN> python3 .../AnsiballZ_command.py ...'
```

`no_log` censors task args and results but **not** the connection plugin's EXEC
line, and `environment:` becomes a command-line prefix. Verified: identical leak
with and without `no_log`, on success and on failure. The fix passes both
secrets through a `0600` file on `/run` (tmpfs, never on disk) that the shell
sources and an `always:` block removes — measured at **0 occurrences** at `-vvv`.

Two related calls, each verified rather than assumed:

- The enrollment failure path used to vanish under `no_log` — presumably why it
  was commented out. Now the command is `failed_when: false`, and a following
  task prints `stderr` from the registered result (which survives `no_log`
  intact) before an assert re-raises with guidance.
- The secrets asserts carry **no** `no_log`: a failing assert prints the
  assertion *expression*, never the value — confirmed at `-vvv` with a canary.
  Adding `no_log` there would censor the fail_msg, which is the one thing the
  operator needs.

### Generation split from enrollment

`finalize.yml` generates the key but never enrolls it. Generating is harmless
anywhere; `mokutil --import` writes firmware state and puts a MokManager screen
in front of the next boot, and at finalize time the key signs nothing yet.
`kernel.yml` invokes `enroll.yml` explicitly as a post-task after the build.
First boot after finalize is unchanged: relabel reboot, no blue screen.

### Two bugs found by running it

- **SIGPIPE in the password generator.** `tr -dc ... < /dev/urandom | head -c 24`
  under `set -o pipefail` returns **141** — `head` closes the pipe, `tr` dies.
  The password generated correctly and the task failed anyway. Replaced with
  `python3 -c` using the `secrets` module; tested three times, 24 chars, no
  ambiguous glyphs (`I l 1 O o 0` excluded — you retype this at a UEFI console
  with no clipboard).
- **The same class, latent, in `fde-kernel-build`.** `find ... | head -1` inside
  `$(...)` under `set -euo pipefail`. Demonstrated it aborts with 141 once output
  exceeds the 64K pipe buffer; `/boot` never will, so it was hardening rather
  than a live bug. Now `find -print -quit`, which states the intent better anyway.

---

## 5. Live run on this workstation

Run for real on this box (which *is* the machine this repo built — `/` is on
`mint--debian13-root` inside the LUKS container), with a rollback snapshot of
`/etc/default/grub*` and `/proc/cmdline` taken to `/root/lmde7-fde-presnap`.

Deliberately **not** run: `prep.yml` (would repartition this exact disk),
`mount-for-installer.yml` (live-ISO only), `finalize.yml` (re-runs LV migration
and needs the LUKS passphrase and TPM PIN).

**`selinux.yml -e target_mount=/`** — 30 ok, 7 changed, 0 failed. Verified on
disk afterwards rather than trusted:

- policy store built (`/etc/selinux/default/policy/policy.34`), `auditd`
  enabled and running
- `/etc/selinux/config` has `SELINUX=permissive` / `SELINUXTYPE=default` with
  the comment block intact — the anchored-regexp decision paying off
- `/.autorelabel` contains `-F`
- `grub.cfg` normal entry gained `security=selinux audit=1
  audit_backlog_limit=8192` **exactly once**; the recovery entry has none, so
  the documented escape hatch is real
- `apparmor.service` masked (still active this boot — the kernel already chose
  its LSM; the mask lands at the next boot)
- re-run: 7 changed then 2, and those two are the deliberately unconditional
  `daemon-reload` and `update-grub`. `/.autorelabel` mtime unchanged across
  three runs, so nothing is re-arming.

**`kernel.yml -e target_mount=/`** — 27 ok, 8 changed, 0 failed. MOK generated
(4096-bit, Code Signing EKU, key and combined PEM `0600`), enrollment staged in
firmware, one-shot password generated and printed, `.mok-hash` removed by the
`always:` block.

### A real bug, found only by running it

The re-run was **not** idempotent: 4 tasks changed every time, re-staging the
MOK import on every invocation. Cause, measured on this hardware:

```
mokutil --list-new | grep -q 'lmde7-fde Machine Owner Key'   -> rc=141
mokutil --list-new | grep -c 'lmde7-fde Machine Owner Key'   -> rc=0
```

`grep -q` exits at the first match, `mokutil` is killed by SIGPIPE while writing
the remaining ~90 lines, and `set -o pipefail` propagates 141. The `elif` read as
"not pending" and the import ran again. It is racy — by hand it returned
`pending`, under ansible it consistently returned `absent` — which is exactly why
it survived every check that was not a live re-run.

This is the **third** instance of the same class today (after the password
generator and `fde-kernel-build`), so I audited the whole tree for `| grep -q`
under `pipefail` and fixed both sites found: `roles/mok/tasks/enroll.yml` (the
live bug) and `roles/tpm2_enroll/tasks/main.yml` (latent — `luksDump` output
fits the pipe buffer today). Both now capture into a variable and match with
`case`, no pipeline at all. The third hit, in `fde-tpm2-unlock.j2`, is in a file
with no `pipefail`, so the pipeline returns grep's own status and is safe.

After the fix, `kernel.yml` re-runs at **0 changed** and reports "MOK is already
staged". Exactly one pending request in firmware — `mokutil --import` replaces
rather than stacks.

### Pre-existing drift noticed, not caused by this run

`/etc/default/grub.d/90-fde-hardening.cfg` on this box predates two config
options: it strips `quiet` (so `boot_strip_quiet: false` is not honoured) and
has no plymouth handling at all (so `boot_disable_plymouth: true` is not in
effect — `50_lmde.cfg` sets `quiet splash` and `splash` survives). The current
template would fix both. Confirmed against the pre-run snapshot that this
predates today's work; left alone, since re-rendering it means running
`target_config`.

---

## 6. Secure Boot ownership, and an idempotency audit

`SECUREBOOT.md`, `roles/secureboot`, `secureboot.yml`. The factory PK/KEK/db are
gone; the machine now trusts one key.

```
PK   CN=lmde7-fde Platform Key         SetupMode=0
KEK  CN=lmde7-fde Key Exchange Key     SecureBoot=0 (enable in firmware)
db   CN=lmde7-fde Machine Owner Key
```

Chain is now `firmware -> UKI`. No Microsoft CA of either kind, no HP CA, no
shim, no GRUB. PK and KEK are generated separately from the signing key on
purpose: the signing key sits on the running root because `update-uki` needs it
unattended, while PK/KEK only authorise policy changes — so a compromised
signing key costs `db`, not the platform. Enrolment order is db then KEK then
PK, because writing PK is what *closes* setup mode.

Confirmed working before enrolling: `BootCurrent=0001`, `LoaderInfo` empty (no
GRUB), `StubInfo=systemd-stub` — firmware loading the UKI directly.

### Idempotency audit

Every safe playbook run twice, every changed task accounted for:

| Playbook | 2nd run | |
|---|---|---|
| `kernel.yml` | `changed=0` | |
| `secureboot.yml` | `changed=0` | |
| `aide.yml` | `changed=1` | `daemon-reload` |
| `selinux.yml` | `changed=2` | `daemon-reload`, `update-grub` |

Those two have no "did anything change" signal to gate on, so `changed_when:
true` is honest. Getting there fixed four real bugs.

**1. A probe reading the wrong stream.** `mokutil --test-key` writes its verdict
to **stderr** and exits 255. The role captured `2>/dev/null`, saw an empty
string, concluded "absent", and re-staged a MOK import on every run. Worse, I
had already "fixed" this once by matching on a *different* message
(`built-in trusted keyring`) — which was a wrong fix for a misdiagnosed cause,
and conflated the kernel keyring with shim MokList. Both corrected: capture
stderr, and that bogus arm removed with a comment saying why it must not come
back.

**2. Per-task `when:` guards that broke ordering.** Guarding each enrolment task
individually looked equivalent to guarding the group. It is not: Ansible
short-circuits a `when:` list *in order*, so `sb_uki.stdout | trim | length > 0`
was evaluated before the guard meant to skip it, against a register from a
skipped task — `object of type 'dict' has no attribute 'stdout'`. Replaced with
one `when:` on a `block:`.

**3. A UKI rebuild that redid ~470 MiB every run.** `update-uki` rebuilt and
re-signed all three images unconditionally. Now it skips images newer than their
kernel, their initrd, *and the script itself* — that last term matters because
the cmdline lives inside the UKI, and the script is only re-rendered when a
cmdline variable actually changes. Verified both directions: three "up to date"
skips, then touching the script forces all three to rebuild.

**4. A task that lied about it.** Even once the script skipped the work, the
Ansible task still had `changed_when: true`. `update-uki` now prints
`rebuilt N image(s)` and the task reports honestly.

### And one config change, not a bug

`mok_enroll` is now `false`. MokList is shim's database, and with a custom `db`
plus direct UKI boot, shim is not in the chain — a staged import would sit in
`MokNew` forever because MokManager never runs to consume it. Clearing the
firmware keys also wiped MokList, so the (now-correct) probe reports the key as
absent; it is the enrolment that is pointless, not the detection.

### Also fixed, spotted in a run log

`config_contract` used `that: "{{ ... }}"`. Ansible deprecated templating
delimiters around conditionals and **removes them in core 2.23** — and that role
gates every playbook in the tree, so it would have broken all nine at once.
Fixed and both paths re-tested; the fail path still names the missing key.

---

## 7. Signed PCR policy — built, enrolled, and reverted

The goal: stop every kernel build from invalidating the TPM keyslot. Bind PCR 11
to a **key** instead of a value, so `ukify` re-signs the predicted PCR set at
build time and the TPM accepts any state carrying that signature.

Built it, and the artifacts were all correct:

```
.pcrsig    8398 bytes   4 phases x 4 banks, all pcrs=[11]
.pcrpkey    451 bytes   embedded pubkey e340276b655c6ef7 == on-disk pubkey
token       tpm2-pubkey populated, tpm2-pubkey-pcrs: 11
```

A third key, deliberately distinct from the MOK and from PK/KEK: it signs
predicted PCR values, so stealing it lets an attacker make a tampered image
*unseal*, not *boot*.

### And it did not work

Boot asked for the PIN and rejected it. One number settled the diagnosis:

```
TPM2_PT_LOCKOUT_COUNTER: 0x0
```

A wrong PIN increments that counter. Zero means **the TPM was never asked** — the
failure was upstream of any authorisation check.

Cause: a signed policy needs `tpm2-pcr-signature.json` readable at unlock time.
`systemd-stub` supplies it, but only across a **systemd-initrd** handoff. Debian's
initramfs-tools builds a script-based initrd (`init` + `scripts/local-top/`),
which never receives it, so the policy session could not be constructed at all.
Confirmed: no `/run/systemd/tpm2-pcr-signature.json`, no `/.extra/`, and the
initrd carries `init`, not `usr/lib/systemd/systemd`.

The enrolment gave no warning, by documented design:

> If a signature file is specified or found it is used to verify if the volume
> can be unlocked … before the new slot is written to disk. **If no signature
> file is specified or found no such safety verification is done.**

No signature present, so no safety check, so a cleanly-written unusable keyslot.
I verified the signature was *produced* and never asked who *consumes* it.

**Reverted** to `tpm2_public_key_pcrs: ""` / `tpm2_pcrs: "7"`, and
`roles/tpm2_enroll` now inspects the initrd and refuses a signed policy it cannot
honour — naming both the symptom and the lockout-counter tell.

### One-step re-enrolment: investigated, rejected

`systemd-cryptenroll` accepts an explicit digest (`--tpm2-pcrs=11:sha256=<hex>`)
and `systemd-measure calculate` predicts PCR 11, so in principle a kernel build
could re-seal itself without a reboot. Tested it: extracted every measured
section from the running UKI (`.linux .osrel .cmdline .initrd .uname .sbat
.pcrpkey`) and predicted `2f09df4a…` against an actual `18580030…`. No match,
cause not established — so enrolling an unverified digest would reproduce the
exact silent failure above. Not used.

### The dependency, made explicit instead

PCR 11 only moves on a playbook run, so the re-seal is a dependency of that run.
`kernel-build.yml` now says so when `tpm2_pcrs` mentions 11 and a kernel was
installed: reboot, then `tpm2.yml`. `tpm2.yml` already refuses unless booted from
the newest UKI, so the order cannot be got wrong. Gating verified across all
three paths.

### Other fixes this round

- **`uuid_map` cross-role coupling.** `roles/tpm2_enroll` templated
  `uuid_map['luks']`, a dict built by `target_config` holding an entry per
  partition and LV. Fine from `finalize.yml`, undefined from `tpm2.yml`. The role
  now reads the one value it needs with its own `blkid`, plus an assert naming
  `target_disk` if it comes back empty. Third instance this session of a role
  breaking when invoked from a playbook other than the one it grew up in.
- **vars_prompt yields the literal string `"None"`** non-interactively, not an
  empty value — `type=str len=4`, measured. So an emptiness check passes straight
  through and the length assert fires instead, blaming the passphrase for being
  short when the real problem is a missing TTY. The secrets contract now detects
  and says so. My first fix also had an operator-precedence bug
  (`[a] if c else [] + ([b] if d else [])` parses as `[a] if c else ([] + …)`),
  silently dropping the `tpm2_pin` check.
- **UKI retention churn.** `uki_retain` pruned after building, so every run built
  `lmde-6.12.48`, then deleted it — 160 MiB written and removed per run.
  Retention now applies at the build stage.

### dracut: assessed, deferred

The signed policy needs a systemd initrd. dracut would also delete the whole
`00_fde_tpm2_unlock` workaround (~100 lines) that exists only because
initramfs-tools' `cryptroot` cannot use a PIN-protected LUKS2 token. Against
that: it replaces the initrd that unlocks root on an FDE box, Debian's dracut is
less travelled with LUKS+LVM+f2fs+fscrypt, and it invalidates the initramfs
assertions in `finalize.yml`. The gain is convenience and code deletion, not
security. Deferred as separate deliberate work.

### The bug that made the revert not stick

Reverting `tpm2_public_key_pcrs` to `""` did not fix the PIN, and the reason was
not the playbook. `systemd-cryptenroll` **auto-discovers**
`tpm2-pcr-public-key.pem` under `/etc/systemd/`, `/run/systemd/` or
`/usr/lib/systemd/` when `--tpm2-public-key` is not passed. Earlier in the
session the key had been placed at `/etc/systemd/tpm2-pcr-public-key.pem`
specifically so enrolment would "find it without being told" — and that
convenience meant omitting the flag silently re-enabled the very policy being
removed.

The journal proved the command was right:

```
systemd-cryptenroll --tpm2-device=auto --tpm2-with-pin=yes --tpm2-pcrs=7 /dev/…
```

and the resulting token still had 29 lines of `tpm2-pubkey` material and
`tpm2-pubkey-pcrs: 11`.

Fixed by moving the key to `/etc/uki/keys/pcr-sign.pub`, off the search path, and
passing it explicitly when wanted. After that the token came out
`tpm2-pubkey: 0 lines`, `tpm2-pubkey-pcrs:` blank — and the PIN worked on the
next boot.

Two diagnostic misses of my own worth recording: I first read the token with
`grep -A1 'tpm2-pubkey:'`, which showed the *next field's* bytes and made an
empty key look populated; and a later filtered dump stripped the hex
continuation lines, making a populated key look empty. Counting the continuation
lines is the only reading that is not ambiguous.

**Final state, verified after reboot:** Secure Boot enabled, booted
`\EFI\Linux\lmde-7.2.0.efi` directly from firmware, SELinux policy loaded,
TPM2+PIN unlock working, no failed units.

---

## Files

**New (13 + 2 playbooks + 2 docs):**

```
SELINUX.md  KERNEL.md  selinux.yml  kernel.yml
roles/selinux/tasks/{main,packages,config,relabel,boot,disable,verify}.yml
roles/selinux/templates/91-fde-selinux.cfg.j2
roles/mok/tasks/{main,generate,enroll}.yml
roles/kernel/tasks/{main,preflight,packages,build,install,publish,client}.yml
roles/kernel/templates/{fde-kernel-build,fde-keep-kernels,kernel-repo.sources}.j2
```

**Changed:** `group_vars/all.yml` (109 keys, 434 lines), `finalize.yml`,
`site.yml`, `roles/config_contract/tasks/main.yml` (93 required keys + 3 new
enums + 2 range asserts), `roles/uki/templates/update-uki.j2`, `README.md`,
`VERIFY.md` (new §12 SELinux, §13 MOK/kernel/repo), `CLAUDE.md`.

---

## What was actually verified

No CI, no test box — so everything below was exercised locally against real
packages and real filesystems.

**SELinux**
- `config_contract` run for real: 93 keys pass; bad enum values produce the
  intended message.
- Full GRUB drop-in sourcing chain simulated: exactly one `security=selinux`
  survives, a stale `security=apparmor` / `selinux=1` from a hand-run of
  `selinux-activate` is stripped, `bash -n` clean.
- `relabel.yml` exercised against a fake target: all three modes, the
  enforcing-on-unlabelled assert firing, and the label-probe exit-code mechanism.
- `/etc/selinux/config` `lineinfile` edits line 6 and **not** the
  `# SELINUX= can take...` comment above it — the aide role's laxer
  `^#?[ \t]*KEY=` regexp *would* have rewritten the documentation. Rerun is a
  no-op.

**MOK / kernel**
- Generated a real MOK: 4096-bit, `CA:FALSE` critical, Code Signing EKU, valid
  DER, combined PEM holds both halves, key/cert confirmed a genuine pair, `0600`
  on private material. Rerun left the key byte-identical.
- The `expect` task really drives `mokutil --generate-hash` (both prompts, one
  response entry). Stateless — no firmware touched.
- Deb repack: built a deb with deliberately stale `md5sums`, modified the
  payload, regenerated, `md5sum -c` clean, repacked deb valid.
- Preflight tested against this box's **real** `noexec /tmp` (fired correctly)
  and real `/home` free space.
- All 64 render combinations of `fde-kernel-build` pass `bash -n`.
- Against the actual `linux-image-7.1.8` deb in `~/git/scripts/kernel`:
  confirmed `CONFIG_EFI_STUB=y` (so `sbsign` works), the `./boot/vmlinuz-*`
  layout, and `DEBUG_INFO_NONE` as the correct switch for the size knob.

**Whole tree**
- 7 playbooks pass `--syntax-check`; all YAML parses; 19 templates render and 11
  shell scripts pass syntax checks.

Not verified: anything needing a chroot (`apt`, `semanage`, `systemctl mask`,
`dpkg -i`), a booted SELinux kernel, an actual kernel compile, a real MOK
enrollment, or a live apt repo. `selinux_booleans` ships empty, so that path is
reasoned-through (`semanage boolean -m -N`, checked against the man page) rather
than executed.

---

## Open / flagged

- **"key registered in TMP"** was read as **MOK enrollment** and built that way,
  since signed kernels are useless under Secure Boot without it. If something
  TPM-side was meant instead (sealing the MOK private key to the TPM, say), that
  is a separate piece and is **not** built.
- **Secure Boot is currently OFF** on this box (`mokutil --sb-state`). Enrollment
  and image signing are inert until it is on; module signature checking works
  regardless. And SB on ⇒ kernel lockdown ⇒ no hibernation — signing your own
  kernel does not buy you out of that.
- **`selinux_booleans` ships empty.** With the `unconfined` module enabled a
  desktop needs none out of the box; add what the denial log actually asks for.
- **No custom SELinux policy modules and no confined login users.** Both are
  deliberate follow-ups, not autoconfigure material.
- **First real run order:** `finalize.yml` → two reboots (the second is the
  SELinux relabel) → a week permissive → read `ausearch -m avc` → promote to
  enforcing. Kernel builds are a separate, later, deliberate act.
