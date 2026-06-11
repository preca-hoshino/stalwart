# Change Log

All notable changes to this project will be documented in this file. This project adheres to [Semantic Versioning](http://semver.org/).

## [0.1.1] - 2026-06-11

### Changed

- Schema source file reverted to English original (`schema.json`), removed pre-gzipped binary; gzip compression now applied at runtime via `flate2` during HTTP response.
- All internal crate versions bumped to `0.1.1`.

### Added

- Chinese (zh) locale translations for all calendar-related i18n strings in `resources/locales/i18n.yml` (41 keys).
- `flate2` dependency added to `crates/http` for runtime gzip encoding.

### Removed

- `UPGRADING/` directory (upgrade notes no longer maintained in-tree).
- Pre-compressed `schema.json.gz` and `schema-old.json` files.

## [0.1.0] - 2026-06-11

### Added

- All-in-one mail server forked from [stalwartlabs/mail-server](https://github.com/stalwartlabs/mail-server).
- Enterprise features unlocked for personal use.
- GHCR container image publishing via GitHub Actions CI/CD.
- Multi-platform builds: Linux (amd64, arm64, armv7, armv6), Windows, macOS (x86_64, aarch64).
- Musl-based Alpine Linux variants.
- Chinese (zh-CN) localized configuration schema.