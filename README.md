# gh-csm

GitHub Codespaces quota dashboard and manager, packaged as a gh extension: one command shows
how much compute / storage quota is left this month, and stops or wakes a codespace while
you are at it.

```
$ gh csm dash
── bb9755 · 2026-10 UTC · resets in 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 2-core 34h · 4-core 17h left
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month
```

`--all` prints one panel per account with `(active)` in the header; an account that has not
unlocked the `user` scope shrinks to a single hint line:

```
$ gh csm dash --all
── bb9755 (active) · 2026-10 UTC · resets in 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 2-core 34h · 4-core 17h left
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month

── BibbyChung · 2026-10 UTC · resets in 7d22h ────
  ⚠ missing user scope — unlock: gh auth switch --user BibbyChung && gh auth refresh -h github.com -s user,codespace
```

## Features

- **dash**: three lines and done — compute / storage bars (green <70%, yellow 70–89%,
  red ≥90%), the remaining core-hours converted to 2-core / 4-core hours (integers, rounded
  down, never overreported), and a reset countdown in the header (`7d22h` / `5h` / `42m`).
  Lists and actions live in `stop`; dash makes a single billing API call.
- **stop**: without arguments it is an interactive picker (pick a number, then confirm);
  `-c <name>` stops directly.
- **run**: wakes the codespace and runs a snippet, or drops into an interactive shell when the
  snippet is omitted (a stopped codespace wakes automatically). Without `-c` it opens the same
  picker as `stop`, then asks for the command — Enter means interactive shell.
- **Multiple accounts**: the gh active account by default, `--account <login>` to pick one,
  `--all` to stack every account.
- **selftest**: offline number crunching on a real billing fixture, checks formulas and colors.

## Requirements

- `gh` (2.40+, already `gh auth login`ed)
- `jq`
- bash 3.2 or newer (macOS ships 3.2.57 as `/bin/bash`, which is fine)

## Install

```bash
gh extension install <owner>/gh-csm
```

After that `gh csm ...` works. Upgrade:

```bash
gh extension upgrade gh-csm
```

For development (inside the repo, creates a symlink, edits take effect immediately):

```bash
gh extension install .
```

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

An account without the scopes does not break the whole dashboard — its panel keeps a single
unlock line while the other accounts render normally.

## Usage

```bash
gh csm                       # help
gh csm dash                  # quota panel (dash must be spelled out; there is no default subcommand)
gh csm dash --all            # every signed-in account, panels stacked
gh csm dash --account bb9755 # one account

gh csm stop                  # picker, then confirm
gh csm stop -c my-codespace  # stop directly

gh csm run                                 # picker, then prompt for a command (Enter = shell)
gh csm run -c my-codespace 'echo gogogo...'   # wake + run, exit when done
gh csm run -c my-codespace                    # wake + interactive shell

gh csm selftest              # offline self-test
```

### Quota limits

Defaults are GitHub Free: 120 core-hours + 15 GB-month. Pro (180 / 20) or any other plan is an
environment variable override:

```bash
GH_CSM_COMPUTE_LIMIT=180 GH_CSM_STORAGE_LIMIT=20 gh csm dash
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
- **Billing month is UTC**: dash's month and reset countdown are UTC. Near the end/start of a
  month your local date can fall in a different month; that is expected. Only the `stop`
  picker's last used is converted to local time.
- **Shutdown still bills storage**: stopping only stops compute; storage keeps accruing. Delete
  to actually stop the bleeding.
- **Deleting does not remove usage already recorded**: billing history stays.
- **classic OAuth only**: the billing endpoint rejects fine-grained PATs; unlock with
  `gh auth refresh` only.
- **Running inside a codespace**: the environment's built-in `GITHUB_TOKEN` shadows your gh
  login; the tool ignores it and says so once. If you still get 403, `unset GITHUB_TOKEN` and
  rerun.
- **github.com only**, no GitHub Enterprise Server.
- `stop` / `run` need the `codespace` scope; the quota section of `dash` needs `user`.

## Development

```bash
shellcheck gh-csm
/bin/bash gh-csm selftest
```

Design decisions, API contract, formulas and the minefield list live in [SPEC.md](SPEC.md).
