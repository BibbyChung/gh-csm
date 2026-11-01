# gh-csm

![CI](https://github.com/BibbyChung/gh-csm/actions/workflows/ci.yml/badge.svg)

GitHub Codespaces quota dashboard and manager, packaged as a gh extension: one command shows
how much compute / storage quota is left this month, and stops or wakes a codespace while
you are at it.

```
$ gh csm quota
── bb9755 · 2026-10 UTC · resets in 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 2-core 34h · 4-core 17h left
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month
```

`--all` prints one panel per account with `(active)` in the header; an account that has not
unlocked the `user` scope shrinks to a single hint line:

```
$ gh csm quota --all
── bb9755 (active) · 2026-10 UTC · resets in 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 2-core 34h · 4-core 17h left
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month

── BibbyChung · 2026-10 UTC · resets in 7d22h ────
  ⚠ missing user scope — unlock: gh auth switch --user BibbyChung && gh auth refresh -h github.com -s user,codespace && gh auth switch --user bb9755
```

## Features

- **quota**: three lines and done — compute / storage bars (green <70%, yellow 70–89%,
  red ≥90%), the remaining core-hours converted to 2-core / 4-core hours (integers, rounded
  down, never overreported), and a reset countdown in the header (`7d22h` / `5h` / `42m`).
  `--month YYYY-MM` renders a past UTC billing month instead of the current one.
  Lists and actions live in `stop`; quota makes a single billing API call.
- **stop**: without arguments it is an interactive picker (pick a number, then confirm);
  `-c <name>` stops directly.
- **start**: wakes the codespace and runs a snippet, or drops into an interactive shell when the
  snippet is omitted (a stopped codespace wakes automatically). Without `-c` it opens the same
  picker as `stop`, then asks for the command — Enter means interactive shell.
- **Multiple accounts**: the gh active account by default, `--account <login>` to pick one,
  `--all` to stack every account.
- **selftest**: offline number crunching on a real billing fixture, checks formulas and colors.

## Requirements

- `gh` (2.40+, already `gh auth login`ed)
- `jq`
- bash 3.2 or newer (macOS ships 3.2.57 as `/bin/bash`, which is fine)
- Supported platforms: macOS and Linux (any environment with bash 3.2+, gh 2.40+, jq).

## Install

```bash
gh extension install BibbyChung/gh-csm
```

After that `gh csm ...` works. Upgrade:

```bash
gh extension upgrade gh-csm
```

For development (inside the repo, creates a symlink, edits take effect immediately):

```bash
gh extension install .
```

### How releases work

This is a script extension, so `main` is the release channel:

- `gh extension install BibbyChung/gh-csm` clones the `main` branch HEAD — the latest commit on
  `main` is the release.
- `gh extension upgrade gh-csm` pulls the latest `main`; that is how you get a new version.
- Git tags and GitHub Releases are informational only (a version marker and a changelog for
  humans) — they play no role in install or upgrade.

## One-time scope unlock

The billing endpoint only accepts the classic OAuth `user` scope; the `codespace` scope is for
listing / stopping / entering codespaces. Unlock once per account:

```bash
gh auth refresh -h github.com -s user,codespace
```

**Multiple accounts**: `gh auth refresh` has no `--user`, so an inactive account has to be
switched to and back.

```bash
gh auth switch --user other-account
gh auth refresh -h github.com -s user,codespace
gh auth switch --user your-main-account
```

`gh csm` prints this sequence in the panel, substituting the account it is showing and your
current active account; an account that is already active only needs the `gh auth refresh` line.

An account without the scopes does not break the whole dashboard — its panel keeps a single
unlock line while the other accounts render normally.

## Usage

```bash
gh csm                        # help
gh csm quota                  # quota panel (quota must be spelled out; there is no default subcommand)
gh csm quota --all            # every signed-in account, panels stacked
gh csm quota --account bb9755 # one account
gh csm quota --month 2026-10  # a past UTC billing month (no reset countdown)

gh csm stop                  # picker, then confirm
gh csm stop -c my-codespace  # stop directly

gh csm start                                 # picker, then prompt for a command (Enter = shell)
gh csm start -c my-codespace 'echo gogogo...'   # wake + run, exit when done
gh csm start -c my-codespace                    # wake + interactive shell

gh csm selftest              # offline self-test
```

### Past months

`--month YYYY-MM` renders an earlier UTC billing month. The header drops `resets in …` (a
countdown only means something for the current month); storage is still divided by that
month's actual hours. The current UTC month is allowed; future months are rejected with the
latest available month in the error.

### Exit codes

`quota` exits 0 when every requested account rendered, and 1 when any requested account could
not be read — missing `user` scope, or the billing fetch failed. `--all` still prints every
panel and exits 1 if at least one account failed.

| code | meaning |
|---|---|
| 0 | every requested account rendered |
| 1 | bad arguments / auth error, or at least one requested account unreadable |

### Quota limits

Defaults are GitHub Free: 120 core-hours + 15 GB-month. Pro (180 / 20) or any other plan is an
environment variable override:

```bash
GH_CSM_COMPUTE_LIMIT=180 GH_CSM_STORAGE_LIMIT=20 gh csm quota
```

To make it permanent, put it in your shell rc:

```bash
export GH_CSM_COMPUTE_LIMIT=180
export GH_CSM_STORAGE_LIMIT=20
```

The API does not expose the account's plan, so this is the only way.

## Limitations (honest disclosure)

- **Numbers lag**: GitHub billing settles roughly once an hour, so a codespace you just stopped
  will not show up immediately.
- **Billing month is UTC**: quota's month and reset countdown are UTC. Near the end/start of a
  month your local date can fall in a different month; that is expected. Only the `stop`
  picker's last used is converted to local time.
- **Shutdown still bills storage**: stopping only stops compute; storage keeps accruing. Delete
  to actually stop the bleeding.
- **Storage SKU**: `Codespaces storage` and `Codespaces prebuild storage` count toward the same
  GigabyteHours pool. The prebuild label is inferred from the API's `codespaces_prebuild_storage`
  id and has not been verified against a real prebuild account; if GitHub returns a different
  label it is skipped with a warning, never miscounted.
- **Deleting does not remove usage already recorded**: billing history stays.
- **classic OAuth only**: the billing endpoint rejects fine-grained PATs; unlock with
  `gh auth refresh` only.
- **Running inside a codespace**: the environment's built-in `GITHUB_TOKEN` shadows your gh
  login; the tool ignores it and says so once. If you still get 403, `unset GITHUB_TOKEN` and
  rerun.
- **github.com only**, no GitHub Enterprise Server.
- `stop` / `start` need the `codespace` scope; `quota` needs `user` for billing.

## Development

```bash
shellcheck gh-csm
/bin/bash gh-csm selftest
```

CI (`.github/workflows/ci.yml`) runs the same two checks on Ubuntu (bash 5.x) and macOS
(bash 3.2, the system `/bin/bash`).

Design decisions, API contract, formulas and the minefield list live in [SPEC.md](SPEC.md).

## License

[MIT](LICENSE)
