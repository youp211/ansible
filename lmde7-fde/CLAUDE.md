# CLAUDE.md — working notes for lmde7-fde

A three-phase Ansible project that builds a LUKS2 full-disk-encrypted LMDE 7
("Gigi", Debian 13/trixie base) system. `README.md` is the user-facing story,
`PLAYBOOKS.md` the one-line index of what to run, `VERIFY.md` the on-device
checklist, and `AIDE.md` / `SELINUX.md` / `KERNEL.md` / `SECUREBOOT.md` /
`BOOTPOLICY.md` the five subsystems with their own operating manuals. This file
is the operating manual for *changing* the tree.

Everything below is relative to this directory (`lmde7-fde/`), which is where
`ansible.cfg` lives and therefore the only directory the playbooks may be run
from. The sibling `gaming-host/` project is unrelated and shares nothing.

## Phases

| Playbook | When | Owns |
|---|---|---|
| `prep.yml` | live ISO, before the installer | GPT, LUKS2 (argon2id), LVM, mkfs, fscrypt metadata — **destroys the disk** |
| `mount-for-installer.yml` | live ISO | assembles the whole layout at `/target` for expert-mode install |
| `finalize.yml` | after the installer, before reboot | crypttab, CIS fstab, LV migration, TPM2+PIN, resume, GRUB cmdline, initramfs, UKI, SELinux, AIDE |
| `selinux.yml` | any time, live or booted | SELinux only — mode, booleans, on/off. Regenerates grub.cfg and rebuilds UKIs itself, because both carry the cmdline |
| `kernel.yml` | any time, booted (mostly) | MOK generation, then enrollment (`enroll.yml`, invoked as a post-task — `finalize.yml` generates but never enrolls), apt repo publish/subscribe. Builds only with `kernel_build_enabled` |
| `kernel-build.yml` | any time, booted | Compile + sign + install a kernel, nothing else. Running it IS the request, so it ignores `kernel_build_enabled`. Forty minutes, tens of GiB |
| `tpm2.yml` | booted, from the UKI you will keep booting | Re-seal the LUKS2 keyslot to the current PCRs. Required after `secureboot.yml` (PCR 7 moves) and after any kernel change when `tpm2_pcrs` includes 11 |
| `secureboot.yml` | booted | Two routes via `secureboot_mode`. `own-keys` replaces factory PK/KEK/db with your own — needs firmware keys cleared first, and `secureboot_enroll` commits. `append-mok` keeps the factory hierarchy and stages the MOK on the ESP for a firmware-side db append; it writes no firmware state at all. Generating/staging is always safe |
| `aide.yml` | any time, live or booted | AIDE only (`-e target_mount=/` on a running system) |

`site.yml` imports all of them behind `never`-tagged imports; nothing runs without a tag.

## Hard rules

1. **`group_vars/all.yml` is the only place values live.** Roles reference variables
   bare — no `| default(...)` inside a task, ever. A value must be changeable in one
   place and be guaranteed to take effect everywhere.
2. **Every new key goes into `roles/config_contract`.** That role runs first in
   `finalize.yml`/`aide.yml` and asserts each required key exists, naming the missing
   ones. It is what turns "old config + new roles" into a two-second failure instead
   of a half-finished run. Enumerated values (`tpm2_boot_unlock`, `aide_init_mode`)
   get a second assert listing the legal values.
3. **Everything touching the installed system goes through `{{ target_mount }}`.**
   Files: `{{ target_mount }}/etc/...`. Commands: `chroot {{ target_mount }} ...`.
   Both forms stay correct when `target_mount: /` (post-boot reruns), so never
   special-case it — except for teardown/unmount, which must refuse `/`.
4. **Idempotent reruns are a feature.** Finalize is expected to be re-run on the
   booted system (e.g. to flip `uki_enabled`). Enrollment steps skip when already
   enrolled; prompts go inert; roles no-op.
5. **Boot-critical vs. convenience.** Anything that can abort the run before
   crypttab/fstab are written must be boot-critical. Convenience packages and extras
   run with `failed_when: false` plus a warning task (see
   `roles/target_config/tasks/packages.yml`).
6. **Asserts carry the fix.** `fail_msg` names the variable to flip or the VERIFY.md
   section to read. See `tpm2_require_initramfs_token` for the pattern.

## Style

- Task names are imperative and say *why* when the why is non-obvious
  ("Verify the TPM2 unlock script is ordered BEFORE cryptroot in the initramfs").
- Comments carry the reasoning and the citations — Debian bug numbers,
  `cryptsetup-open(8)` semantics, GRUB argon2id support. That research is the
  actual value in this tree; keep it inline where the code depends on it.
- Shell blocks: `args: executable: /bin/bash` + `set -o pipefail` when piping.
  `changed_when` is always explicit.
- Templates are `.j2`, named after the file they become, and start with a
  "managed by lmde7-fde" comment.
- Roles have `tasks/` (+ `templates/`). No `defaults/`, no `vars/`, no `meta/` —
  that would violate rule 1.

## Testing

There is no CI and no test box in the loop. What is safe to run here:

```
ansible-playbook --syntax-check finalize.yml aide.yml selinux.yml kernel.yml kernel-build.yml secureboot.yml prep.yml mount-for-installer.yml site.yml
ansible-lint            # clean; see .ansible-lint for the deliberate opt-outs
```

`ansible-lint` passes at the **production** profile with zero findings. That is
the point of `.ansible-lint`: five style rules this project disagrees with are
skipped *with a stated reason each*, so a run is a regression detector rather
than 75 lines of intentional noise. Do not add to that skip list to make a new
finding go away — fix the finding, or argue the case in the file.

It is not packaged here by default (`apt-cache policy ansible-lint` shows it is
available, but installing needs sudo). Without root, build it in a **tmpfs that
allows exec** — the scratchpad under `/tmp` is `noexec` and native `.so` files
fail to load with `failed to map segment from shared object`:

```
python3 -m venv --system-site-packages /run/user/$(id -u)/lintenv
/run/user/$(id -u)/lintenv/bin/python3 -m pip install ansible-lint
cd lmde7-fde && /run/user/$(id -u)/lintenv/bin/python3 -m ansiblelint --offline
```

`--system-site-packages` makes it lint against the system `ansible-core` rather
than pulling its own. Invoke via `python3 -m ansiblelint`, not the `bin/`
console script — those are real files and would need exec on the venv itself.

## Idempotency

A second run must report `failed=0` and change only tasks that genuinely have no
"did anything change" signal to gate on. Measured on the live box:

| Playbook | 2nd-run `changed` | What, and why it is irreducible |
|---|---|---|
| `kernel.yml` | **0** | — |
| `secureboot.yml` | **0** | — |
| `aide.yml` | 1 | `systemctl daemon-reload` |
| `selinux.yml` | 2 | `daemon-reload` + `update-grub` |

`daemon-reload` and `update-grub` report nothing useful about whether they
altered anything, so `changed_when: true` is honest for them. **Anything else
showing up as changed on a rerun is a bug**, and three of them were found this
way — a probe discarding the stderr its verdict was written to, a UKI rebuild
that rewrote 470 MiB of ESP per run, and per-task `when:` guards that broke
short-circuit ordering. Check with:

```
sudo ansible-playbook <pb> -e target_mount=/ 2>&1 \
  | awk '/^TASK \[/{t=$0} /^changed:/{print t}'
```

Two idioms worth copying when adding tasks:

- **Guard a group with one `when:` on a `block:`**, not a condition per task. A
  `when:` list short-circuits in order, so a condition referencing a register
  from an earlier *skipped* task is evaluated before the guard meant to prevent
  it, and raises `object of type 'dict' has no attribute 'stdout'`.
- **Capture stderr when probing a tool for state.** `mokutil --test-key` prints
  its verdict on stderr and exits 255; `2>/dev/null` turns that into a silent
  wrong answer rather than a visible failure.

- **Probe before adding to a database that rejects duplicates.** A bare
  `dpkg-statoverride --add` fails on the second run, so gate it on
  `--list <path>` (rc 1 = absent) registered by an *unconditional* task. Parsing
  the failure message instead makes the task's success depend on wording that
  belongs to dpkg, not to us. See `roles/uki/tasks/boot_policy.yml`.

**Never run `prep.yml`** — it repartitions `target_disk`. Do not run `finalize.yml`
against `/target` unless the user says the live session is set up. `aide.yml -e
target_mount=/` is the one playbook that is reasonable to actually execute on a
booted machine, and only when asked. `selinux.yml -e target_mount=/` is **not** in
that category: it swaps the active LSM and schedules a full relabel, so it changes
how the next boot goes. Never run it unprompted. `kernel.yml` with
`kernel_build_enabled` is a forty-minute compile that eats tens of GiB — never run
it unprompted either, and never point `kernel_build_dir` at `/tmp` or `/var/tmp`
(both `noexec` under this tree's own fstab, which fails a kernel build outright).

Role logic that touches no chroot can be exercised against a throwaway tree —
build a fake `/etc/selinux/config` + `/etc/fstab` in the scratchpad, import the
individual task files from a temporary playbook, and point `-e target_mount=` at
it. That is how the SELinux relabel decision, the `/etc/selinux/config`
`lineinfile` anchoring and the GRUB drop-in sourcing chain were verified without
a test box.

## Tools with implicit defaults

Three commands in this tree change behaviour based on files they find rather
than flags they are given. The first two cost a debugging session each:

- **`systemd-cryptenroll`** auto-loads `tpm2-pcr-public-key.pem` from
  `/etc/systemd/`, `/run/systemd/` or `/usr/lib/systemd/` when
  `--tpm2-public-key` is absent — so a signed PCR policy cannot be turned off by
  omitting the flag. `uki_pcr_pub` keeps the key off that path for exactly this
  reason. See SECUREBOOT.md "The auto-discovery trap".
- **`mokutil --test-key`** writes its verdict to **stderr** and exits 255.
  `2>/dev/null` turns a correct answer into a silent wrong one.
- **`sbverify --cert`** answers a different question than you probably meant.
  It returned `Signature verification OK` against four *different* Microsoft CAs
  for one `bootmgfw.efi`. Use **`sbverify --list`** to see what actually signed a
  binary; `--cert` is for asserting a specific key covers it, and is what
  `roles/secureboot` uses once the cert is already known.
- **`grub-mkconfig`** decides which `/etc/grub.d/` drop-ins to run from the
  **executable bit**, not from any variable — `test -x "$i"` in the loop. That
  is the only lever for suppressing `10_linux`, whose kernel globs are
  hardcoded. And because `10_linux` is a **conffile**, dpkg resets its mode when
  `grub-common` is upgraded, silently re-enabling every classic boot entry
  during a security update. `dpkg-statoverride` is the counter. See
  BOOTPOLICY.md.

When a probe disagrees with reality, check what the tool reads and which stream
it writes to before changing the logic around it.

## Environment gotchas

- When the development machine runs the same distribution as the target, check
  package facts locally with `apt-cache policy` / `apt-get download` +
  `dpkg-deb -x` instead of guessing at Debian packaging behaviour. Do that before
  writing rules that depend on a package's layout.
- `/tmp` is mounted `noexec` under this project's own fstab policy, so a binary
  extracted into a scratch directory there cannot be executed. Read the shipped
  man pages (`man -l <extracted>/usr/share/man/...`) instead.
- The tree is under git. There is history to consult, but do not commit unless
  asked.
