# Running part of the playbook: tags

A full run walks every role, even the ones with nothing to change. To run only
what you're working on, use **tags**: every core role carries its own name plus a
slice, and **each of your sections is a tag too**.

| Tag | Runs | What it is |
|---|---|---|
| `core` | init, base, wifi, tooling, console_fonts, lock, just, identity, creds, tailscale | a secure, connected terminal |
| `extras` | python, comms, all your sections, catalogue | what you install and iterate on |
| `sections` | all your sections + catalogue | just your own software |
| `<section name>` | that one section + catalogue | e.g. `games`, `dotfiles` |
| `access` | identity, creds, tailscale | overlay: the revocable credentials (also in `core`) |
| `<role name>` | that one role | e.g. `python`, `lock` |

```bash
ansible-playbook site.yml -K --list-tags                    # what tags exist (your sections included)
ansible-playbook site.yml -K --tags games --list-tasks      # what a tag WOULD run
ansible-playbook site.yml -K --tags games                   # one section (+ refresh deck-catalogue)
ansible-playbook site.yml -K --tags dotfiles,workstation    # several sections
ansible-playbook site.yml -K --tags sections                # all your sections
ansible-playbook site.yml -K --tags extras                  # python + comms + all your sections
ansible-playbook site.yml -K --tags access                  # after rotating a PAT / key / tailnet auth key
ansible-playbook site.yml -K --skip-tags init               # everything except the hardware step
ansible-playbook site.yml -K --tags games --check --diff    # dry run: what would change
```

How tags behave:

- `--tags X` runs what's tagged `X`, plus the setup steps tagged `always` (version
  check, config checks, loading the device profile). Untagged steps — e.g. the
  closing success banner — are skipped.
- A role can carry several tags; `access` overlaps `core` on purpose.
- Tags filter, they never reorder: a filtered run still goes top to bottom.
- A filtered run assumes the rest ran once before. A section with `pip:` needs the
  `python` role to have run at least once.
- Section tags select precisely, but **skipping is coarse**: `--skip-tags games`
  skips *all* sections (Ansible can't build one task per section from your config,
  so one task carries every section's tag). Select what you want instead.
- `deck-catalogue` refreshes whenever any section runs.
