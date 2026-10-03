# Secrets posture

cyberdeck is a **public repo** that provisions a **device that leaves the house**.
Those two facts set the rules for every secret the project touches.

## The one-line rule

**No secrets in the *repo*; secrets on the *device* are fine — injected at
provision time, never committed.**

Your real values live only in `config.yml` (gitignored). The provisioning machine
reads them and pushes them straight onto the device (WiFi profiles, a `0600` env
file). No secret — plaintext or ciphertext — ever enters tracked files or history.

## No SOPS in this project

There is exactly one consumer of the secrets (your provisioning machine) and one
file (`config.yml`), so the project does not adopt SOPS or mint an encryption key.
If you want `config.yml` encrypted **at rest on your own machine**, use your
personal vault (age / pass / SOPS) — that is your choice and the project depends on
none of it. A cyberdeck encryption key living on a roaming device would only widen
the blast radius of a lost device.

## Two credential tiers

| Tier | Examples | Allowed on the deck? |
|---|---|---|
| **Personal** | WiFi PSKs, a fine-grained GitHub PAT, a spend-capped Claude key | **Yes** — your blast radius, revocable in one click |
| **Lab** | Gitea / homelab tokens, infrastructure credentials | **No, by default** |

A token is access just as much as a tunnel is. Putting a homelab token on a device
that leaves the house is the same risk that rules out always-on VPN autoconnect.
If a deck genuinely needs to reach a self-hosted service, give it a **scoped,
revocable device identity** (see below), not a lab token — and write down how to
revoke it.

## SSH keys: generated on the device, never transported

- `ssh.authorized_keys` — **public** keys allowed to log *into* the deck. Not
  secret.
- `ssh.generate_keys` — key pairs the deck generates **for itself** at provision
  time. The private key never leaves the device, never touches `config.yml`, the
  repo, or the provisioning machine. The role prints the public key; you register
  it with GitHub/Gitea as a device identity you can revoke independently.

This is the doctrine-compatible path to git hosting: the deck acts as *itself* with
a narrowly-scoped key, instead of carrying your credentials.

## Known gaps (accepted, not hidden)

- Secrets sit **plaintext at rest on an unencrypted microSD** in a device that
  leaves the house. The mitigation is tier discipline + revocability, not disk
  encryption (full-disk crypto on a Pi Zero 2 W is not worth the cost). Treat a
  lost deck as "rotate the personal-tier creds it held."
- `config.yml` is a single gitignored file with **no backup story** — lose the
  provisioning machine, lose the restore inputs. Back it up in your own vault.
