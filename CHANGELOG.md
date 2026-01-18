<!-- markdownlint-disable MD024 -->
# Changelog

All notable changes to this project will be documented in this file.

---

## [0.1.1] - 2026-01-18

### Fixed

- Fix deprecated `security_opt` syntax
- Fix `pids_limit` too restrictive

### Documentation

- Add alternatives section
- Explain `pids_limit` upgrade considerations

---

## [0.1.0] - 2026-01-07

Initial release.

### Added

- Docker Compose configuration with security hardening
- Resource limits (CPU, memory, process)
- Logging configuration with rotation
- Optional file provider for static image list
- Socket proxy support (recommended) with configurable network
- Environment variables for cron schedule and Docker endpoint
- Example configuration with file provider and Ntfy notifier
- Documentation with setup instructions

---

## About

This project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).  
The changelog format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
