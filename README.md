# cyberdeck

Turn a small-screen Linux handheld into a homelab terminal.

Flash a stock Raspberry Pi OS card, open SSH, run one playbook — and get a usable
terminal (zsh, tmux, git) sized for a tiny screen. It is **profile-driven**: screen
geometry, board and driver stack are data, so a new handheld is a new file, not a
rewrite.

> **Status.** All planned provisioning roles are implemented — hardware init,
> base/SSH, WiFi, tooling, console fonts, lock, the `just` menu, Python/uv, identity, creds,
> comms, Tailscale, catalogue — plus your own **sections** (apt / pip / git / downloads), which
> is also how dotfiles are installed (proven on hardware). Small-screen config is
> console-only and lives in the public dotfiles (prompt + tmux status adapt to
> terminal width), so the deck needs no extra role for it. Layer-3
> companion services are still ahead — see the Roadmap.

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
3. **No secrets ever pass through the repo.** WiFi, SSH keys, and tokens are
   supplied at provision time from your gitignored `config.yml` (and the Pi Imager
   / `boot/firstrun.sh`), never committed.

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

## Installation

You run cyberdeck from a **control machine** (your laptop or desktop). It connects
to the deck over SSH and configures it — you don't install anything on the deck by
hand.

**Requirements (pinned):** ansible-core **2.21** (via `ansible` **14.0.0**),
Python **3.14**. The play asserts core ≥ 2.21 and fails early otherwise;
`requirements.txt` pins the toolchain.

**First, make the deck reachable over SSH.** Easiest path: in **Raspberry Pi
Imager**, before flashing, click the gear / *Edit Settings* and set a hostname, a
username, your **SSH public key** (or a password), and your **WiFi**. Flash, boot
the card, and confirm `ssh <youruser>@<deck-ip>` works. *(Alternative: copy
`boot/firstrun.sh.example` to `boot/firstrun.sh`, fill it in, and drop it on the
card's boot partition.)*

Then, on your control machine:

```bash
# 1. Get the code
git clone https://github.com/jakes-homelab/cyberdeck.git
cd cyberdeck

# 2. Install the pinned Ansible toolchain (ansible 14 == core 2.21, tested)
pip install -r requirements.txt          # or: pipx install ansible==14.0.0
#    The ansible bundle includes the collections we use. If you installed
#    ansible-core on its own, also run:
ansible-galaxy collection install -r requirements.yml

# 3. Create your config and fill it in (what to fill → Configuration below).
#    Set device.host, device.user, and device.ssh_private_key at minimum.
cp config.example.yml config.yml && $EDITOR config.yml

# 4. Generate the inventory from config.yml (host + user + key, one place)
scripts/gen-inventory

# 5. Provision it
ansible-playbook site.yml
#    if that user's sudo asks for a password, add -K:
ansible-playbook site.yml -K
```

On a PicoCalc, the first run brings the hardware up and **reboots the deck once**
mid-play to load the display/keyboard overlays; Ansible waits for it to come back
and continues. After that your screen and keyboard are live. Re-run any time — it
is **idempotent** (a second run reports no changes and does not reboot).

**Watch it live (optional).** Early in the run the play prints a command; run it in
a second terminal to tail the deck's output (apt progress, service logs):

```bash
ssh <youruser>@<deck-ip> 'journalctl -f'
```

> **Re-flashed the card?** Its SSH host key changed — clear the stale entry first:
> `ssh-keygen -R <deck-ip>`.

## Configuration

Two files hold everything you customize. Both are **gitignored** — your copies,
never committed:

- **`inventory/hosts.ini`** — **generated** by `scripts/gen-inventory` from the
  `device` block in `config.yml` (host + user + SSH key), so you don't hand-edit it
  or scatter the connection details. Re-run the script after changing `config.yml`.
- **`config.yml`** — everything else, copied from `config.example.yml`. Every
  section is commented in that file; edit what you need and leave the rest. The two
  you'll reach for first:

**WiFi — where your network names go** (`wifi:`). A list, so the deck reconnects to
whichever is in range; higher `priority` wins. DHCP unless you add a `static` block.

```yaml
wifi:
  - { ssid: "MyHomeWiFi", psk: "my-wifi-password", priority: 100 }
  - { ssid: "MyPhone",    psk: "hotspot-password", priority: 50 }
```

**SSH keys — two directions** (`ssh:`).

```yaml
ssh:
  # Public keys allowed to log IN to the deck (your laptop's key):
  authorized_keys:
    - "ssh-ed25519 AAAA... you@laptop"
  # Keypairs COPIED onto the deck so it can authenticate OUT (your infra / git).
  # You provide them; the private key is copied over at 0600 and never committed:
  identity_keys:
    - { name: cyberdeck, private: "~/.config/cyberdeck/keys/id_ed25519", public: "~/.config/cyberdeck/keys/id_ed25519.pub" }
```

**The rest, briefly:** `device` (host / user / ssh_private_key / timezone / locale / profile) ·
`python` (versions, default, default-env libs) · `sections` (YOUR groups of software — each one an Ansible tag, using apt / pip / git / downloads; see [docs/software.md](docs/software.md)) · `creds` (your dev-identity tokens) · `comms` (public BBS / Usenet
servers) · `lock` (idle blank / logout / tmux lock timers — defaults are
safe, omit to keep them) · `tailscale` (auth key to reach the homelab — **leave empty to skip**,
e.g. when you're already on your LAN).
Your dotfiles are just a section: a `git` entry whose install runs `stow`.

> **Secrets** — WiFi passwords, tokens, and private keys — live **only** in
> `config.yml` and the files it points at. It is gitignored; never commit it. Full
> model: [docs/secrets-posture.md](docs/secrets-posture.md).

## Layout

```
site.yml                      play: init -> base -> wifi -> tooling -> console_fonts -> lock -> just -> identity -> creds -> tailscale -> python -> comms -> YOUR sections -> catalogue
config.example.yml            environment inputs (copy to config.yml)
scripts/gen-inventory         build inventory/hosts.ini from config.yml's device block
inventory/hosts.example.ini   inventory shape (real one is generated, gitignored)
boot/firstrun.sh.example      first-boot script template
profiles/picocalc-pizero2w.yml   profile #1
roles/init/                   PicoCalc hardware: display + keyboard + audio, then reboot
roles/base/                   locale, timezone, packages, SSH hardening
roles/wifi/                   prioritised NetworkManager WiFi profiles (+ static IP)
roles/tooling/                zsh, tmux, git, stow — light by constraint
roles/console_fonts/          console font library + `deck-font` switcher (320x320)
roles/lock/                   console blank + idle logout + tmux vlock (an unattended deck locks)
roles/just/                   the `just` command menu — `just` lists it; `just wifi`, `just font`, …
roles/identity/               copy the deck's own SSH keys (outbound auth)
roles/section/                one of YOUR sections: apt -> pip -> git -> downloads (each with install commands)
roles/catalogue/              record git + downloads + apt; the `deck-catalogue` listing
roles/creds/                  personal-tier env file (0600), sourced at login
roles/python/                 uv + Python versions + the default env (~/.venvs/base) on PATH
roles/comms/                  BBS (telnet) + Usenet (tin) + per-server launchers
roles/tailscale/              join the tailnet (skips without an auth key)
docs/secrets-posture.md       what secrets go where, and why
docs/raspberry.md             deck reference — handy TUIs & commands
docs/pyenv.md                 python env management (default env, uv run, direnv)
docs/software.md              adding software: git clones + pinned downloads, install semantics
docs/tags.md                  run part of the playbook: core / extras / access + per-role tags
```

## Roadmap

**Working:** `init` (PicoCalc hardware — display, keyboard, audio, poweroff — then
a reboot; see [docs/hardware-init.md](docs/hardware-init.md)); bare provisioning
(`base`, `tooling`, `dotfiles`); `wifi` (prioritised NetworkManager profiles +
static IP); `console_fonts` (console font library + the `deck-font` switcher for
the 320×320 panel); `lock` (an unattended deck blanks, logs out bare console
shells, and locks tmux with vlock); `python` (uv + several Python versions + the default env); `identity` (copy the deck's own SSH keys for outbound auth);
**your sections** (apt / pip / git / downloads, each section an Ansible tag); `catalogue` (a `deck-catalogue` listing of
what's installed — git, downloads, apt — with descriptions); `creds` (personal-tier env
file); `comms` (dial public BBSes + read Usenet); `tailscale` (next hop to the
homelab). The play also prints an
`ssh … 'journalctl -f'` command so you can watch the deck's live output while
provisioning runs.

All planned provisioning roles are implemented. Still ahead, from the spec:
finer **small-screen config** (tmux/pager/editor sized from profile geometry, via
the dotfiles layer) and **companion services** (layer 3 — text-cleaning backends
that make web content render at 320×320).

## License

Apache License 2.0 — see [LICENSE](LICENSE).
