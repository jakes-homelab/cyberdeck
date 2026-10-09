# Python environments on the deck

Three layers, from "just works" to "fully isolated per project".

## 1. The default env — always there

`~/.venvs/base` is a uv-managed venv that's **always on PATH** (a `profile.d` hook
adds it at login). Out of the box:

- `python` is that env's interpreter,
- `rich` and `textual` are importable, plus anything you put in `python.pip` (or a section's `pip:`),
- nothing to activate.

It's *on PATH*, not *activated* (no `VIRTUAL_ENV` set) — so a project env cleanly
takes over when you enter one, and you fall back to this when you leave.

## 2. Per-project, no activation — `uv run` (lean on this)

From a project directory:

```sh
uv run python app.py
uv run pytest
```

`uv run` executes in the project's env (creating/syncing it as needed) **without
activating anything** in your shell — no lingering state, nothing to `deactivate`.

## 3. Per-project, activated — venv or direnv

Create a venv — two ways, same result (a `.venv` you activate identically):

```sh
# classic (Python stdlib):
python3 -m venv .venv              # or:  python3.10 -m venv .venv   (a specific version)

# uv (faster; picks any uv-installed version with one flag):
uv venv                            # default interpreter
uv venv --python 3.10 .venv        # a specific version

# then, either way:
source .venv/bin/activate
# ... work ...
deactivate                         # stays active until you deactivate or close the
                                   # shell — `cd` elsewhere does NOT turn it off.
```

Both produce a standard `.venv`; `uv venv` is just faster and makes choosing a
specific interpreter (from the ones uv installed) a one-flag job. Inside an
activated venv, install with `pip install X` (classic) or `uv pip install X`
(faster, same result).

### Auto-activate with `direnv`

`direnv` activates on `cd` **in** and deactivates on `cd` **out**. The binary is
installed by the `python` role; add the shell hook once (this comes via your
dotfiles, or add it by hand for now):

```sh
# ~/.zshrc   (or ~/.bashrc)
eval "$(direnv hook zsh)"
```

Then drop a `.envrc` in the project directory:

```sh
# .envrc — use this project's uv venv automatically
[ -d .venv ] || uv venv
export VIRTUAL_ENV="$PWD/.venv"
PATH_add "$PWD/.venv/bin"
```

and trust it once (direnv refuses to run an untrusted `.envrc`):

```sh
direnv allow
```

### How this interacts with the default env

The default env (#1) is always on PATH; you never "activate" it. When you `cd`
into a direnv project, direnv prepends `.venv/bin` **on top** → `python` is now the
project's env. When you `cd` out, direnv unloads it → PATH reverts and you're back
on the default env.

So: **default is the fallback, direnv overrides inside the project, leaving the
directory restores the default.** Nothing is double-activated.
