# Adding software: `python` + your sections

Everything you install beyond the core system lives in `config.yml` in two kinds
of block:

- **`python:`** — the Python toolchain: which versions, which is the default, and
  the libraries in the default env.
- **Your sections** — groups *you* name (`workstation`, `dotfiles`, `games`, …).
  Each section is also an Ansible tag, so `--tags games` installs just that group.

## Sections

List them in `sections:`; they install in that order, after the core system:

```yaml
sections:
  - workstation
  - dotfiles
  - games
```

Every section uses the **same four keys**, all optional, always run in this order:

| Key | What | Runs as |
|---|---|---|
| `apt` | system packages | root |
| `pip` | libraries for the default python env (`~/.venvs/base`) | you |
| `git` | repos to clone, each with ordered `install` commands | you |
| `downloads` | pinned files (url + sha256), each with ordered `install` commands | you |

```yaml
games:
  apt:
    - chocolate-doom
  pip:
    - pygame              # lands in the default env, importable from `python`
  git:
    - name: doom-kit      # your own repo: WADs + config + install.sh
      repo: "git@git.example.com:you/doom-kit.git"
      version: v1
      install:
        - ./install.sh
      entrypoint: doom
      description: "chocolate-doom + my WADs and setup"
```

**Writing lists:** one item per line, so each can carry a comment. `[a, b]` is the
same list written inline. An *empty* list has no one-per-line form — leave the key
out (a bare `pip:` with nothing under it also counts as empty).

**Section names** are yours, with two rules the play checks for you: every name in
`sections:` must exist as a block, and a name can't clash with something the
playbook already uses (core roles, `python`, `comms`, `extras`, `apt`, `git`, …).

## `git` entries

```yaml
  git:
    - name: py-thing                                  # required
      repo: "https://github.com/you/py-thing.git"     # required — the repo to clone
      version: v1.2.0                                 # optional — see below
      dest: ~/src/py-thing                            # optional (this is the default)
      install:                                        # optional
        - uv venv .venv
        - uv pip install --python .venv/bin/python -r requirements.txt
        - ln -sf "$PWD/run.sh" ~/.local/bin/py-thing
      entrypoint: py-thing                            # optional — shown in deck-catalogue
      description: "An example tool"                  # optional — shown in deck-catalogue
```

### `version`: branch, tag, or commit

One field — git resolves the name itself. What matters is whether it **moves**:

| `version:` | Moves? | What a playbook run does |
|---|---|---|
| *(omitted)* | yes | follows the repo's default branch |
| a branch (`main`) | yes | pulls new upstream commits **and re-runs `install`** |
| a tag (`v1.2.0`) | no | nothing, until you change it |
| a commit SHA | no | nothing, until you change it |

A **GitHub release** is a tag plus files attached to it. To build it from source,
use `git` with the tag as `version`; to use its prebuilt binary, use `downloads`.

### Kits: put the logic in its own repo

When something needs several steps — copy assets, write a config, make a
launcher — keep the config entry small and put the steps in a repo with its own
`install.sh` (a "kit"). The kit is versioned, testable on your laptop, and reusable
on any machine. Rules for `install.sh`: **no `sudo`** (it runs as you; system
packages go in the section's `apt`), and keep private assets (e.g. commercial WADs)
in a **private** repo the deck reaches with a read-only deploy key.

### Dotfiles are just a section

```yaml
dotfiles:
  apt:
    - tealdeer
  git:
    - name: dotfiles-public
      repo: "https://github.com/you/dotfiles.git"
      dest: ~/dotfiles-public
      install:
        - stow --restow zsh tmux bin tealdeer
```

## `downloads` entries

```yaml
  downloads:
    - name: aichat                                    # required
      url: "https://github.com/sigoden/aichat/releases/download/v0.30.0/aichat-v0.30.0-aarch64-unknown-linux-musl.tar.gz"
      sha256: "eb1cd0948569404c5d9d01c10b32b902e11f8231073315456454dec246bdf26e"
      install:                                        # required
        - tar -xzf "$FILE" -C ~/.local/bin aichat
      entrypoint: aichat
      description: "All-in-one LLM CLI — chat, REPL, shell assistant"
```

- **`$FILE`** is the downloaded file's path; `install` decides what to do with it.
- **`sha256`** is required; the download is verified and skipped when it already matches.
- **Anything compiled (Rust, Go, C) → `downloads`.** Building on a 512 MB board
  (≈259 MB free) is not an option. Pick the `aarch64` / `arm64` asset for a 64-bit
  Pi OS (`dpkg --print-architecture` → `arm64`); `-musl` builds are fully static.
- **Getting the sha256:** the release's published checksum, or `sha256sum <file>`
  (macOS: `shasum -a 256 <file>`). **Upgrading** = change `url` and `sha256` together.

## How `install` runs

- **In order, stopping at the first failure** — like joining the lines with `&&`.
- **As you**, in a **bash login shell** — your PATH (`~/.local/bin`, `uv`, the
  default python env).
- **Where:** in the clone for `git` (`$PWD` is the repo); in the download cache for
  `downloads` (use `$FILE`).
- **When:** only when something changed — the commit (`git`), the `sha256`
  (`downloads`), or **the `install` lines themselves**. A failed install retries next run.

## `python:`

```yaml
python:
  versions:
    - "3.8"
    - "3.10"
    - "3.13"
  default: "3.13"     # must be one of versions; `python` runs this one
  pip:                # the default env (~/.venvs/base, always on PATH)
    - rich
    - textual
```

uv installs each version once. The default env is built on `default`; change
`default` and the env is rebuilt (its libraries reinstalled). A section's `pip:`
adds to the same env — one env, not one per section (RAM and disk are tight).
Empty `versions` = no Python toolchain; a section with `pip:` then fails clearly.
More: [pyenv.md](pyenv.md).

## What you see in the Ansible output

Each section shows its steps, one line per entry:

```
TASK [section : [games] git: clone (or update)] ******************
changed: [deck] => (item=doom-kit)

TASK [section : [games] git: install (in order, stop at first failure)] ***
changed: [deck] => (item=doom-kit)
```

`ok` = already in place; `changed` = it just ran. A failing `install` shows each
command as it ran, ending at the one that broke:

```
+ ./install.sh
cp: cannot stat 'wads/doom2.wad': No such file or directory
```

## Seeing what's installed

```
$ deck-catalogue
GIT
  doom-kit           7fd1a60    chocolate-doom + my WADs and setup  [doom]
DOWNLOADS
  aichat             eb1cd094   All-in-one LLM CLI — chat, REPL, shell assistant  [aichat]
APT
  chocolate-doom     Doom engine ...
```

## Migrating

The play refuses the old shapes and points here.

| Old | New |
|---|---|
| `python: {enable, versions}` (last = default) | `python: {versions, default, pip}` |
| `packages.apt` / `packages.pip` | a section's `apt` / `pip` (e.g. `workstation:`) — or `python.pip` for default-env libs |
| top-level `git:` list (`url:`) | a section's `git:` (`repo:`) |
| top-level `downloads:` list | a section's `downloads:` |
| `dotfiles.public_repo` (+ stow list) | a `dotfiles` section: `git` entry with `install: [stow --restow …]` |
| older still: `repos:` with `build: pip\|uv\|make\|custom` | `git` entries with explicit `install:` lines |

Markers carry over: an entry keeps its `name`, so it isn't reinstalled just
because it moved into a section. For dotfiles, set `dest: ~/dotfiles-public` (the
old clone location) and the existing links stay.
