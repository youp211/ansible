# ansible

Two self-contained Ansible projects for LMDE 7 / Debian 13 workstations. They
share nothing but a distribution — pick the one you want and run it from its own
directory.

| Project | What it does |
|---|---|
| **[lmde7-fde/](lmde7-fde/)** | Builds a LUKS2 full-disk-encrypted LMDE 7 system and hardens its boot chain: encrypted layout before the installer, then crypttab, CIS mount options, TPM2+PIN unlock, signed Unified Kernel Images, Secure Boot key ownership, SELinux and AIDE. |
| **[gaming-host/](gaming-host/)** | Tunes a Linux gaming desktop that also streams to Moonlight clients: GameMode, CPU power policy, Sunshine capture/encode, per-stream display switching. |

Each directory holds its own `ansible.cfg`, inventory and docs, and **must be run
from inside itself** — Ansible reads `ansible.cfg` only from the current working
directory, so a playbook invoked by absolute path from elsewhere silently finds
no inventory and exits having done nothing.

```bash
cd lmde7-fde   && sudo ansible-playbook finalize.yml -e target_mount=/
cd gaming-host/ansible && ansible-playbook site.yml --ask-become-pass --check
```

## Before you run anything

Both projects ship with placeholder identity. Set it first:

- `lmde7-fde/group_vars/all.yml` — `target_disk`, `expected_disk_size_gib`,
  `confirm_disk_wipe`. `prep.yml` **destroys the disk it is pointed at** and
  refuses to run until all three match reality.
- `gaming-host/ansible/inventory.ini` — `target_user`.

`lmde7-fde/PLAYBOOKS.md` lists every playbook, how to invoke it, and which ones
must never run unprompted. Read it before the first run.

## Status

Both trees are in use on real hardware and are written to be re-run: a second run
reports `failed=0` and changes only the handful of tasks that have no "did
anything change" signal to gate on. Neither has CI; `ansible-lint` at the
production profile and `--syntax-check` are the regression detectors, and
`lmde7-fde/CLAUDE.md` documents how to run both.
