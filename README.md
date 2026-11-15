# genv

> Load env vars from a single gopass entry into bash, zsh or nushell, with
> one decrypt instead of one per secret. A tiny Rust CLI with no
> dependencies, for macOS and Linux.

```sh
genv-load work/env      # ~0.3 s: every variable exported into this shell
genv-unload             # gone again
```

## Why this exists

### The starting point: secrets in shell files

Most people's work environment starts out as a shell function in a dotfiles
repo that exports the tokens and passwords a day's work needs:

```sh
load-work-keys() {
  export GITHUB_TOKEN="ghp_…"          # literal secret in a file
  export JIRA_API_TOKEN="ATATT…"
  export DATADOG_APIKEY="9c41…"
  …
}
```

It's convenient, and then the dotfiles repo gets pushed. Every secret is now
in git history on a remote, and rotating them all is the only real fix.

### The usual fix: one gopass lookup per secret

The standard remedy is to keep secrets in [gopass](https://www.gopass.pw/) and
look them up at load time:

```sh
load-work-keys() {
  export GITHUB_TOKEN=$(gopass show -o work/github-token)
  export JIRA_API_TOKEN=$(gopass show -o work/jira-token)
  export DATADOG_APIKEY=$(gopass show -o work/datadog-api-key)
  … # 20 more
}
```

That fixes the leak, but creates three new problems.

**1. It's slow.** Each `gopass show` is a separate GPG decryption. Measured on
an Apple Silicon Mac with gpg-agent already unlocked:

| | Time |
|---|---|
| one `gopass show -o` | ~180 ms |
| the same file with plain `gpg --decrypt` | ~176 ms |
| gopass's own startup (`gopass --version`) | ~66 ms |
| **21 lookups in a row** | **~3.5 s** |

So almost all the time is GPG doing a public-key decryption per entry. That's
why "rewrite the loader in a faster language" doesn't help: a Rust program
making 21 decrypts is just as slow. The only real fix is **fewer decrypts**.

**2. It fails silently.** If one lookup fails (wrong path, locked agent,
cancelled pinentry), `$(…)` returns an empty string and the variable is set
to `""`, or keeps a stale value from earlier. The function still prints "loaded".

**3. It only works in POSIX shells.** The same function can't be sourced in
nushell, and keeping a second copy per shell means they drift.

### What genv does instead

genv keeps **all the variables in one gopass entry**:

```text
GITHUB_TOKEN,GH_TOKEN=ghp_…
JIRA_API_TOKEN=ATATT…
DATADOG_APIKEY,TF_VAR_DATADOG_APIKEY=9c41…
JIRA_PROJECT_KEY=OPS
```

and turns it into your shell's syntax:

- **One decrypt.** `genv load` runs `gopass show` once, so loading takes
  about 0.2–0.3 s however many variables there are. In real use, 41
  variables went from ~3.5 s to ~0.3 s.
- **All or nothing.** If gopass fails or any line is malformed, genv prints
  nothing and exits 1, so the wrapper changes nothing. No half-loaded shells,
  no empty secrets that look loaded.
- **One file, every shell.** The same entry loads into bash and zsh (`eval`
  of single-quoted exports) and nushell (`from json | load-env`), on macOS
  and Linux, from one small binary.
- **Safe to `eval`.** Values are literal: no `$VAR` expansion and no
  command substitution. Every value is single-quoted with `'` escaped as `'\''`,
  so a value like `$(rm -rf ~)` stays text. Tests run hostile values
  through real bash, zsh and nushell and check nothing executes.
- **No secrets in output you didn't ask for.** Errors name the line and
  variable, never the value. `genv names` lists names only. `Debug` output
  hides values.
- **Unloadable.** genv records which names it set (in `GENV_LOADED`), so
  `genv-unload` removes exactly those, and the secrets don't linger in a long-lived shell.
- **One place to rotate.** A secret lives on exactly one line, so rotating
  it is one `gopass edit`. Aliases (`GITHUB_TOKEN,GH_TOKEN=…`) mean no
  duplicate copies to forget.

Nothing is written to disk. Secrets go from the encrypted store straight
into your shell's environment.

### What it doesn't do

- It doesn't replace gopass or GPG. It's a thin layer that calls `gopass show` once.
- It can't protect secrets once they're in your shell: every program you
  start inherits them. Use `genv-unload`, or load in a subshell for one command.
- It doesn't sync anything. gopass's own git sync already does that.

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

## Acknowledgements

genv is a thin layer on top of other people's work:

- **[gopass](https://github.com/gopasspw/gopass)**, the team password manager
  that stores and syncs the secrets. genv only calls `gopass show`; all the
  encryption, storage, git sync and key management is gopass.
- **[pass](https://www.passwordstore.org/)** by Jason A. Donenfeld, the
  original "standard unix password manager" that gopass builds on.
- **[GnuPG](https://gnupg.org/)**, which does the actual decryption.

genv is an independent project and isn't affiliated with or endorsed by the
gopass project.

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or
[MIT license](LICENSE-MIT), at your option.
