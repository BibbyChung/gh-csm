# Changelog

All notable changes to gh-csm are recorded here.

The format is based on [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] - 2026-11-03

### Added

- `gh csm ls`: `gh codespace list` for a chosen account, otherwise verbatim — same table, colors,
  pager and exit code. `--account <login>` (also `--account=<login>`) picks the identity; every
  other argument is forwarded to `gh` exactly as typed.

## [1.1.0] - 2026-11-01

### Added

- `gh csm quota --month YYYY-MM`: render a past UTC billing month. The reset countdown is omitted
  for months other than the current one, and future months are rejected.
- Prebuild storage counts toward the same storage quota as codespace storage.
- CI runs shellcheck and the selftest on Ubuntu and macOS, covering bash 5.x and macOS bash 3.2.

### Changed

- `quota` exit codes are now meaningful: `0` when every requested account rendered, `1` when any of
  them could not be read (missing `user` scope or a failed billing fetch). `--all` still prints
  every panel before exiting.
- The unlock hint for an inactive account now includes the switch back to your active account, so
  following it no longer leaves your global gh login on the other account.

## [1.0.0] - 2026-11-01

### Added

- `gh csm quota`: one billing API call, a three-line panel per account with compute and storage
  bars plus the reset countdown.
- `gh csm stop`: interactive codespace picker with confirmation, or `stop -c <name>` directly.
- `gh csm start`: wake a codespace and run a snippet, or enter an interactive shell; without `-c`
  it opens the same picker as `stop`.
- Multiple signed-in accounts: the gh active account by default, `--account <login>` to pick one,
  `--all` to stack every account in the quota panel.
- Offline `gh csm selftest` over a real billing fixture: formulas, SKU parsing, color thresholds,
  month lengths, timezone conversion and reset countdowns.
- MIT license.

[Unreleased]: https://github.com/BibbyChung/gh-csm/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/BibbyChung/gh-csm/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/BibbyChung/gh-csm/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/BibbyChung/gh-csm/compare/19f3c95...v1.0.0
