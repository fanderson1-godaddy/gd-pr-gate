# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] - 2026-10-05

### Added
- Initial release of PR security gate workflow
- Dependency scanning via `actions/dependency-review-action`
- Semgrep scanning via Dodge SARIF (with CLI fallback)
- Sticky PR comment with findings tables
- Configurable thresholds (`dep_fail_severity`, `semgrep_fail_severity`)
- Dry-run mode (`block: false`)
- Documentation: usage, how-it-works, behavior diagram
- Support for 9 Donut repos (dify-*, marketing-*)

[Unreleased]: https://github.com/gdcorp-actions/gd-pr-gate/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/gdcorp-actions/gd-pr-gate/releases/tag/v1.0.0
