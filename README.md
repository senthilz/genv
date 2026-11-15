# genv

Load environment variables from **one gopass entry** into bash, zsh or
nushell, on macOS or Linux.

One entry means **one decrypt**: about 0.2 s, instead of one `gopass show`
per variable (21 of those took about 3.5 s).

## Entry format

Store the variables in a single gopass entry, e.g. `work/env`:

```text
# Work environment: loaded with  genv-load work/env
JIRA_PROJECT_KEY=OPS
JIRA_API_TOKEN=ATATT...
GITHUB_TOKEN,GH_TOKEN=ghp_...            # one value, several names
TF_HTTP_PASSWORD,GITLAB_ACCESS_TOKEN=glpat-...
JIRA_TRANSITION_RESOLVE=In Progress,Review,Ready,Done
export EDITOR=vim                        # "export " is optional
```

Rules:
- One `NAME=value` per line. Blank lines and `#` comments are ignored.
- Names are `[A-Za-z_][A-Za-z0-9_]*`. Use `A,B=value` to set several names to one value.
- **Values are literal.** There's no `$VAR` expansion, no `$(…)`, and no
  escapes; everything after the first `=` is the value. One pair of matching
  surrounding quotes is removed, so `X="a b"` and `X=a b` are the same.
- Each value is on one line.
- `GENV_LOADED` is reserved. genv uses it to remember which names it set.

Edit it with `gopass edit work/env`. List the names without values with
`genv names work/env`.

## Shell setup

**bash / zsh**: in `~/.bashrc` or `~/.zshrc`:

```sh
genv-load()   { local out; out=$(command genv load "$1" --shell sh) && eval "$out"; }
genv-unload() { local out; out=$(command genv unload --shell sh) && eval "$out"; }
```

**nushell**: in `config.nu`:

```nu
def --env genv-load [entry: string] { genv load $entry --shell nu | from json | load-env }
def --env genv-unload [] { hide-env --ignore-errors ...(genv unload --shell nu | from json) }
```

Then, in any of the three:

```sh
genv-load work/env          # export everything
genv-unload                 # remove it again
```

A program can't change its parent shell's environment, so genv prints
commands (sh) or JSON (nu) and the wrapper applies them. The sh output is
safe to `eval`: every value is single-quoted, with `'` escaped as `'\''`,
so nothing inside a value runs. The tests check this with hostile values in
real bash, zsh and nu.

## Commands

| Command | Output |
|---|---|
| `genv load ENTRY --shell sh` | `export NAME='value'` lines, plus `GENV_LOADED` |
| `genv load ENTRY --shell nu` | a JSON record for `from json \| load-env` |
| `genv unload --shell sh\|nu` | `unset …` / a JSON list for `hide-env` |
| `genv names ENTRY` | variable names only, one per line |

On any error, genv writes **nothing** to stdout and exits 1. Errors name
the line and variable, never the value.

`GENV_GOPASS=/path/to/gopass` overrides which gopass binary is used.

## Install

Needs a Rust toolchain (≥ 1.85) and `gopass` on `PATH`. There are no other dependencies.

```sh
cargo install --locked --path .          # from a checkout
```

The binary is self-contained. Build it once per OS/architecture (macOS
arm64, Linux x86_64/arm64) and copy it to `~/.local/bin` or `/usr/local/bin`.

## Notes

- **Secrets end up in your shell's environment** and are inherited by every
  program you start from it. Use `genv-unload` when you're done, or load in a
  subshell for one command:
  `( genv-load work/env && terraform plan )` (bash/zsh).
- genv overwrites the decrypted text in its own memory before exiting. That's
  a best effort: copies in gopass, gpg-agent and the shell are out of its reach.
- Nothing is written to disk.

## Development

```sh
cargo test        # unit tests + end-to-end tests in real bash, zsh and nu (skipped if not installed)
cargo clippy --all-targets -- -D warnings
```

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or
[MIT license](LICENSE-MIT), at your option.
