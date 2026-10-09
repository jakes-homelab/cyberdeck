# Deck reference — handy TUIs & commands

A cheat sheet for operating the deck from its own 320×320 console. Not every tool
is installed on every build; the ones cyberdeck adds are marked *(cyberdeck)*.

## The deck menu: `just` *(cyberdeck)*

`just` on its own lists the deck's commands. Works anywhere under `~`; inside a
project that has its own justfile, use `just -g <command>`.

| Command | What |
|---|---|
| `just wifi` | manage WiFi networks (`sudo nmtui`) |
| `just font` / `just font list` / `just font set <font>` | console font (wraps `deck-font`) |
| `just catalogue` | what's installed (`deck-catalogue`) |
| `just mem` | memory use (`free -m`) — budget is ~259 MB available |
| `just lock` | lock this console now (`vlock`) |

The menu is managed by the playbook (`roles/just/files/justfile`) — add commands
there, not on the deck, or they're overwritten on the next run.

## Network

| Command | What |
|---|---|
| `nmtui` | NetworkManager TUI — connect/edit WiFi, see status (the easy one) |
| `nmcli device wifi list` | scan for networks from the CLI |
| `iwgetid` | show the SSID you're on |
| `raspi-config` | Pi settings TUI — also has networking under *System Options* |

## System

| Command | What |
|---|---|
| `raspi-config` | the big Pi settings TUI — interfaces (SPI/I2C), boot/console, locale, overclock |
| `htop` | process/mem monitor (watch that 512 MB) |
| `alsamixer` | audio levels TUI |
| `bluetoothctl` | interactive Bluetooth control |
| `journalctl -f` | follow the system log live |

## Terminal

| Command | What |
|---|---|
| `tmux` | terminal multiplexer — split/detach; essential on one small screen |
| `man <cmd>` | manual pages — reflow to the screen width by themselves; prefer over `--help` |
| `<cmd> --help N` | *(dotfiles)* `--help` reflowed to fit (`N` = `\| narrow`) |
| `tldr <cmd>` | short, example-first cheat sheets (tealdeer; fetches its pages on first use) |
| `less` | chops long lines instead of wrapping (`LESS=-FRSX`) — pan with ←/→ |
| `nano` / `vim` | editors |
| `deck-font list` / `set <font>` | *(cyberdeck)* switch the console font (size vs. legibility on 320×320) |
| `deck-catalogue` | *(cyberdeck)* what's installed — git, downloads, apt, with descriptions |

## Comms *(cyberdeck)*

| Command | What |
|---|---|
| `bbs-<name>` | dial a configured BBS (telnet) |
| `news-<name>` | read a configured Usenet server |
| `tin` | the Usenet reader itself |

## Python on the deck

- Default env `~/.venvs/base` is always on PATH, so `python` has **`rich`** +
  **`textual`** ready; `curses` is built in (stdlib, no install).
- Per-project work: `uv run <cmd>`, or a venv / `direnv`.
- Full guide — the default env, `uv run` vs activate, direnv `.envrc`, and the
  "venv stays active" gotcha: **[docs/pyenv.md](pyenv.md)**.

## Finding more

There's no "list all TUIs" command (TUI isn't a package category). To explore:
`apt list --installed`, `deck-catalogue`, `compgen -c | sort` (every command —
noisy), and `man <cmd>`.
