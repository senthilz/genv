# gopass-env

> Load env vars from a single gopass entry into bash, zsh or nushell, with
> one decrypt instead of one per secret. A tiny Rust CLI with no
> dependencies, for macOS and Linux.

```sh
gopass-env-load work/env      # ~0.3 s: every variable exported into this shell
gopass-env-unload             # gone again
```

## Why

Keeping secrets in gopass and loading them with one `gopass show` per
variable works, but each lookup is a separate GPG decryption (about 180 ms).
21 lookups took **about 3.5 s** every time. gopass-env keeps all the variables in
**one** entry, so loading is one decrypt: **about 0.3 s** for 41 variables. The
same entry works in bash, zsh and nushell, on macOS and Linux.

Details, measurements and the full security model:
[docs/why-and-security.md](docs/why-and-security.md).

## Entry format

Store the variables in a single gopass entry, e.g. `work/env`:

```text
# Work environment: loaded with  gopass-env-load work/env
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
- `GOPASS_ENV_LOADED` is reserved. gopass-env uses it to remember which names it set.

Edit it with `gopass edit work/env`. List the names without values with
`gopass-env names work/env`.

## Shell setup

**bash / zsh**: in `~/.bashrc` or `~/.zshrc`:

```sh
gopass-env-load()   { local out; out=$(command gopass-env load "$1" --shell sh) && eval "$out"; }
gopass-env-unload() { local out; out=$(command gopass-env unload --shell sh) && eval "$out"; }
```

**nushell**: in `config.nu`:

```nu
def --env gopass-env-load [entry: string] { gopass-env load $entry --shell nu | from json | load-env }
def --env gopass-env-unload [] { hide-env --ignore-errors ...(gopass-env unload --shell nu | from json) }
```

Then, in any of the three:

```sh
gopass-env-load work/env          # export everything
gopass-env-unload                 # remove it again
```

A program can't change its parent shell's environment, so gopass-env prints
commands (sh) or JSON (nu), and the wrapper applies them.

## Commands

| Command | Output |
|---|---|
| `gopass-env load ENTRY --shell sh` | `export NAME='value'` lines, plus `GOPASS_ENV_LOADED` |
| `gopass-env load ENTRY --shell nu` | a JSON record for `from json \| load-env` |
| `gopass-env unload --shell sh\|nu` | `unset …` / a JSON list for `hide-env` |
| `gopass-env names ENTRY` | variable names only, one per line |

On any error, gopass-env writes **nothing** to stdout and exits 1. Errors name
the line and variable, never the value.

`GOPASS_ENV_GOPASS=/path/to/gopass` overrides which gopass binary is used.

## Install

Needs a Rust toolchain (≥ 1.85) and `gopass` on `PATH`. There are no other dependencies.

```sh
cargo install --locked --path .          # from a checkout
```

The binary is self-contained. Build it once per OS/architecture (macOS
arm64, Linux x86_64/arm64) and copy it to `~/.local/bin` or `/usr/local/bin`.

> **Formerly `genv`.** It was renamed because Homebrew's `coreutils` installs
> GNU `env` as `genv`, which shadowed it on Intel Macs. If you used the old
> name, delete `~/.cargo/bin/genv`, rename the wrappers to
> `gopass-env-load` / `gopass-env-unload`, and reload any shell that still
> has `GENV_LOADED` set.

## Security

- **Nothing on disk:** secrets go from the encrypted gopass store straight into your shell.
- **Fails closed:** on any error gopass-env prints nothing, so the shell is unchanged.
- **Safe to `eval`:** values are literal and single-quoted; nothing inside a value runs.
- **Values are never printed** in errors, `gopass-env names` or debug output.
- **Not protected after loading:** every program you start inherits the
  variables. Use `gopass-env-unload`, or a subshell for one command.
- **Not a compliance solution:** there's no access control, audit log or rotation enforcement.

More: [security model](docs/why-and-security.md#security-model).
Report vulnerabilities privately: [SECURITY.md](SECURITY.md).

## Development

```sh
cargo test        # unit tests + end-to-end tests in real bash, zsh and nu (skipped if not installed)
cargo clippy --all-targets -- -D warnings
```

## Acknowledgements

gopass-env is a thin layer on top of other people's work:

- **[gopass](https://github.com/gopasspw/gopass)**, the team password manager
  that stores and syncs the secrets. gopass-env only calls `gopass show`; all the
  encryption, storage, git sync and key management is gopass.
- **[pass](https://www.passwordstore.org/)** by Jason A. Donenfeld, the
  original "standard unix password manager" that gopass builds on.
- **[GnuPG](https://gnupg.org/)**, which does the actual decryption.

gopass-env is an independent project and isn't affiliated with or endorsed by the
gopass project.

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or
[MIT license](LICENSE-MIT), at your option.
