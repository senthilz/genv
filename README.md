# genv

> Load env vars from a single gopass entry into bash, zsh or nushell, with
> one decrypt instead of one per secret. A tiny Rust CLI with no
> dependencies, for macOS and Linux.

```sh
genv-load work/env      # ~0.3 s: every variable exported into this shell
genv-unload             # gone again
```

## Why

Keeping secrets in gopass and loading them with one `gopass show` per
variable works, but each lookup is a separate GPG decryption (about 180 ms).
21 lookups took **about 3.5 s** every time. genv keeps all the variables in
**one** entry, so loading is one decrypt: **about 0.3 s** for 41 variables. The
same entry works in bash, zsh and nushell, on macOS and Linux.

Details, measurements and the full security model:
[docs/why-and-security.md](docs/why-and-security.md).

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
commands (sh) or JSON (nu), and the wrapper applies them.

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

## Security

- **Nothing on disk:** secrets go from the encrypted gopass store straight into your shell.
- **Fails closed:** on any error genv prints nothing, so the shell is unchanged.
- **Safe to `eval`:** values are literal and single-quoted; nothing inside a value runs.
- **Values are never printed** in errors, `genv names` or debug output.
- **Not protected after loading:** every program you start inherits the
  variables. Use `genv-unload`, or a subshell for one command.
- **Not a compliance solution:** there's no access control, audit log or rotation enforcement.

More: [security model](docs/why-and-security.md#security-model).
Report vulnerabilities privately: [SECURITY.md](SECURITY.md).

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
