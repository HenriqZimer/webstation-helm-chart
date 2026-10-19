# Changelog

All notable changes to this Helm chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-10-19

### Added
- Initial release: Deployment, Service, optional Ingress, PVCs and Secret for
  `ghcr.io/romm-streaming/romm-webstation` (Selkies desktop + every emulator + `romm-broker`),
  RomM's `protocol: webstation` emulator-streaming backend.
- `broker.secret` / `broker.existingSecret` -> `BROKER_SECRET` (required by the broker).
- `persistence.config` (`/config`) and `persistence.library` (`/romm/library`, shared with RomM, read-only by default).
- Intel/AMD (`/dev/dri`) and NVIDIA GPU passthrough, `shmSize`, `seccompUnconfined`, amd64 `nodeSelector`.
- `values.schema.json`.
