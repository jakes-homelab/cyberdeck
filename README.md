# cyberdeck

Turn a small-screen Linux handheld into a homelab terminal.

Flash a stock Raspberry Pi OS card, open SSH, run one playbook — and get a usable
terminal (zsh, tmux, git) sized for a tiny screen. It is **profile-driven**: screen
geometry, board and driver stack are data, so a new handheld is a new file, not a
rewrite.

> **Status: increment 1 (bare provisioning), in progress.** Layer 1 roles `base`
> and `tooling` are complete; the `dotfiles` role is written but **unproven** — it
> depends on the public dotfiles repo, which does not exist yet. Hardware
> enablement, WireGuard, and companion services are later increments (see below).

## The three layers

| Layer | What | Runs where |
|---|---|---|
| **1. Device** | Ansible: provision the handheld — base system, hardware enablement, tooling | on the device |
| **2. Config** | shell, tmux, pager, editor — themed and sized for a tiny screen | on the device, from the public dotfiles repo |
| **3. Companions** | backend services that transform text so a 320×320 panel can render it | anywhere |

A stranger with a handheld and no homelab gets a working device from layers 1 and 2
alone. Layer 3 is additive.

## The discipline (read before you commit)

This repo is **public from the first commit**. The rule travels with the repo:

1. **Nothing environment-specific in tracked files or history** — no domains,
   hostnames, IP addresses, usernames, or key names. Not in a file, not in a
   commit.
2. **Environment identity is an input.** `config.example.yml` and
   `inventory/hosts.example.ini` are committed; the real `config.yml` and
   `inventory/hosts.ini` are gitignored and yours alone.
3. **No secrets ever pass through the repo.** WiFi and SSH material are supplied at
   provision time via `boot/firstrun.sh` (gitignored); WireGuard keys (a later
   increment) are generated on-device.

If a change can't pass all three, it doesn't belong here.

**Secrets** follow one rule: none in the *repo*; secrets on the *device* are fine,
injected at provision time from your gitignored `config.yml`. Personal creds (WiFi,
a fine-grained GitHub PAT) are allowed on the deck; lab creds are not — a device
that leaves the house must not carry lab access. Full reasoning and the two-tier
model: [docs/secrets-posture.md](docs/secrets-posture.md).

## Device profiles

A profile (`profiles/<name>.yml`) is data: screen geometry, board, input device,
driver stack, quirks. Roles read the profile; nothing hardcodes a device. The first
profile is `picocalc-pizero2w` — a PicoCalc shell with a Raspberry Pi Zero 2 W
inside. Add a handheld by adding a profile file.

**Screen geometry is load-bearing.** tmux layout, pager width, editor defaults and
every layer-3 renderer derive from it. Hardcode `320` anywhere and this becomes a
PicoCalc project again.

## Quickstart

```bash
# 0. Install dependencies
ansible-galaxy collection install -r requirements.yml

# 1. Your environment inputs
cp config.example.yml config.yml            && $EDITOR config.yml
cp inventory/hosts.example.ini inventory/hosts.ini && $EDITOR inventory/hosts.ini

# 2. Flash a stock Raspberry Pi OS card, then prepare first boot
cp boot/firstrun.sh.example boot/firstrun.sh && $EDITOR boot/firstrun.sh
#    drop firstrun.sh on the card's boot partition (see the script's header)

# 3. Boot the device, then provision it
ansible-playbook site.yml
```

Re-running the playbook makes no changes — it is idempotent.

> **Re-flashing a card** changes its SSH host key. Clear the stale entry before
> re-provisioning: `ssh-keygen -R <device-host>`.

## Layout

```
site.yml                      play: base -> tooling -> dotfiles
config.example.yml            environment inputs (copy to config.yml)
inventory/hosts.example.ini   inventory (copy to inventory/hosts.ini)
boot/firstrun.sh.example      first-boot script template
profiles/picocalc-pizero2w.yml   profile #1
roles/base/                   locale, timezone, packages, SSH hardening
roles/tooling/                zsh, tmux, git, stow — light by constraint
roles/dotfiles/               clone public dotfiles, stow (gated)
```

## Roadmap

**Done:** bare provisioning — inventory, `base`, `tooling`, `dotfiles`.

**Planned roles** (scaffolded under `roles/`, each wired into `site.yml` as it
lands; exact sequencing is set by the SPEC amendment):

- `wifi` — prioritised network profiles from the `wifi` list, so a reflashed deck reconnects unattended.
- `workstation` — uv Python toolchain (several versions), plus the `apt` and `pip` lists.
- `identity` — `authorized_keys`, and on-device key generation with a public-key report.
- `repos` — clone the `repos` list.
- `creds` — personal-tier env file (`0600`); lab creds excluded by design.

**Also from the spec, not yet scheduled:** hardware enablement (capture the
display + keyboard work as a profile-driven role), small-screen config (tmux,
pager, editor sized from profile geometry), and companion services (text-cleaning
backends for 320×320).

## License

Apache License 2.0 — see [LICENSE](LICENSE).
