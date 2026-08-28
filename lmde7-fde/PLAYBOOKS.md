# PLAYBOOKS.md — what each playbook does, in one line

Every playbook is `hosts: live`. Nothing runs without being named: `site.yml`
imports all of them behind `never`-tagged imports.

## Install phases — run in this order, once

| Playbook | Where | What it does |
|---|---|---|
| `prep.yml` | live ISO, **before** the installer | Repartitions `target_disk`: GPT → LUKS2 (argon2id) → LVM → filesystems → fscrypt metadata. **Destroys the disk.** |
| `mount-for-installer.yml` | live ISO, after `prep.yml` | Assembles the complete final layout at `/target` so the installer's *expert mode* writes straight onto the encrypted LVs. |
| `finalize.yml` | after the installer, **before** rebooting | crypttab, CIS fstab, LV data migration, TPM2+PIN enrolment, `resume=`, hardened cmdline, initramfs, UKIs, SELinux, AIDE. |

## Subsystems — run any time, in any order, as often as you like

| Playbook | What it does |
|---|---|
| `uki.yml` | Builds and signs Unified Kernel Images and points a firmware boot entry at the newest. Enrols nothing — this is the *prepare for* Secure Boot step. |
| `selinux.yml` | SELinux only: install, mode, booleans, off. Regenerates `grub.cfg` and rebuilds UKIs itself, because both carry the cmdline. |
| `aide.yml` | AIDE only: the integrity baseline and boot-time check for the one surface FDE leaves open. |
| `kernel.yml` | The MOK signing key, apt policy for kernels, and the apt repo publish/subscribe modes. Builds a kernel only with `kernel_build_enabled`. |
| `kernel-build.yml` | Compiles, signs and installs one kernel and nothing else. Running it *is* the request, so it ignores `kernel_build_enabled`. **~40 minutes, tens of GiB.** |
| `tpm2.yml` | Re-seals the LUKS2 keyslot to the current PCRs. Required after `secureboot.yml` (PCR 7 moves) and after any kernel change when `tpm2_pcrs` includes 11. |
| `secureboot.yml` | Makes firmware trust this machine's images. `own-keys` replaces the factory PK/KEK/db (`secureboot_enroll=true` commits); `append-mok` keeps the factory hierarchy and stages the MOK on the ESP for a firmware-side db append, writing no firmware state at all. |
| `site.yml` | Imports every playbook above behind tags: `--tags prep`, `--tags finalize`, … |

## Five rules for invoking them

```bash
# 1. On a booted system, always say so. Everything is written through
#    {{ target_mount }}, which defaults to the live-ISO /target.
sudo ansible-playbook finalize.yml -e target_mount=/ -e finalize_teardown=false

# 2. For a remote target, do NOT prefix with sudo. become happens on the target;
#    a local sudo makes ansible use root's SSH identity and fails as unreachable.
ansible-playbook -i inventory/remote.ini uki.yml -e target_mount=/

# 3. Never pass a secret with -e. Extra-vars land on the connection plugin's
#    EXEC line, which no_log does NOT suppress. Let the vars_prompt ask.

# 4. Check before you commit anything to firmware.
ansible-playbook --syntax-check <playbook>

# 5. Never pass "-l localhost". The [live] group holds exactly ONE host and it
#    is named `workstation`, so -l is redundant everywhere and wrong with that
#    argument.
sudo ansible-playbook secureboot.yml -e target_mount=/
```

The host is **`workstation`**, not `localhost`, in both inventories, and that is
load-bearing: Ansible matches `host_vars/` on the inventory hostname, so calling
it `localhost` makes every override in `host_vars/workstation.yml` silently
inert on local runs and falls back to `group_vars/all.yml`.

Two ways to end up with no hosts, and they look identical:

```bash
ansible-playbook secureboot.yml --list-hosts -l localhost   # ERROR: no hosts to target
cd /tmp && ansible-playbook ~/lmde7-fde/secureboot.yml --list-hosts   # hosts (0)
```

The second is the sneakier one: `inventory = inventory/hosts.ini` comes from
`ansible.cfg`, which Ansible reads only from the **current working directory**.
Run a playbook by absolute path from somewhere else and it finds no inventory,
matches nothing, and exits 0 having done nothing. Always run from the project
root — or use `./enroll-sb`, which resolves the playbook relative to itself.

## What must never run unprompted

- **`prep.yml`** — it repartitions `target_disk`. There is no undo.
- **`selinux.yml -e target_mount=/`** — swaps the active LSM and schedules a full
  relabel, i.e. it changes how the next boot goes.
- **`kernel.yml -e kernel_build_enabled=true`** and **`kernel-build.yml`** —
  forty minutes and tens of GiB. Never point `kernel_build_dir` at `/tmp` or
  `/var/tmp`; both are `noexec` under this tree's own fstab and a kernel build
  fails outright.
- **`secureboot.yml -e secureboot_enroll=true`** — writes firmware key state. On
  a dual-boot machine it can leave the other OS unbootable.

`aide.yml -e target_mount=/` is the one playbook that is routinely reasonable to
run on a live machine.

## Where the detail lives

`README.md` is the story, `VERIFY.md` the on-device checklist. Per subsystem:
**SELINUX.md**, **KERNEL.md**, **SECUREBOOT.md**, **AIDE.md**, **BOOTPOLICY.md**.
`CLAUDE.md` is the operating manual for changing the tree.
