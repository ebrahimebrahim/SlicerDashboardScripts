# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains scripts that orchestrate building, testing, and packaging [3D Slicer](https://slicer.org) and its extensions across Linux, macOS, and Windows dashboard machines. The infrastructure is maintained by Kitware, Inc. Build results are submitted to [CDash](https://slicer.cdash.org) under `SlicerStable` and `SlicerPreview` projects.

## Architecture

### Dashboard Machines

Each dashboard machine has a hostname used as a file prefix. Current machines:

- **computron** — macOS (native builds)
- **metroplex** — Linux (Docker-based builds using `slicer-buildenv-*` images)
- **bluestreak** — Windows (batch scripts)

### File Naming Convention

Scripts follow the pattern `<hostname>-<target>.cmake` or `<hostname>-<target>.<sh|bat>`:

- `<hostname>.sh` / `<hostname>.bat` — Top-level entry point that orchestrates all builds for that machine. Invoked by cron (Linux/macOS) or Task Scheduler (Windows). Contains a `CMAKE_VERSION` comment parsed by maintenance scripts.
- `<hostname>-slicer_preview_nightly.cmake` — Slicer Preview (nightly from `main`)
- `<hostname>-slicer_stable_package.cmake` — Slicer Stable (tagged release, `SCRIPT_MODE=Experimental`)
- `<hostname>-slicerextensions_preview_nightly.cmake` — Extensions for Preview
- `<hostname>-slicerextensions_stable_nightly.cmake` — Extensions for Stable

### CMake Dashboard Scripts

Each `.cmake` script configures CTest variables and then includes a driver script downloaded from the Slicer repo. Key variables:

- `Slicer_RELEASE_TYPE` — `P` (Preview), `S` (Stable), or `E` (Experimental)
- `SCRIPT_MODE` — `Nightly`, `Continuous`, or `Experimental`
- `GIT_TAG` — Branch or tag to build (`main` for Preview, e.g. `v5.10.0` for Stable)
- `EXTENSIONS_INDEX_BRANCH` — Branch of the ExtensionsIndex repo

The section after `WARNING: DO NOT EDIT BEYOND THIS POINT` downloads and includes the driver script from the Slicer GitHub repo. Do not modify that section.

### Maintenance System (`maintenance/`)

Remote maintenance of Unix dashboards via SSH. Structured as:

```
maintenance/<hostname>/     — Per-machine: crontab, connection info (REMOTE_HOSTNAME, REMOTE_IP, REMOTE_USERNAME), remote scripts
maintenance/common/         — Shared scripts (crontab push/pull, CMake update, remote execution)
maintenance/guides/         — Step-by-step procedures for common maintenance tasks
```

The `maintenance/Makefile` provides targets for remote operations. Run `make help` in the `maintenance/` directory to see available targets. Key targets:

- `make nightly-script-cmake-update CMAKE_VERSION=X.Y.Z` — Update CMake version in nightly scripts
- `make remote-install-cmake CMAKE_VERSION=X.Y.Z` — Install CMake on dashboard machines
- `make pull-crontab` / `make push-crontab` — Sync crontab settings with dashboards

### Helper Tool (`scripts/cdashly/`)

Python CLI for creating dashboard scripts for new machines:

```
pip install scripts/cdashly
cdashly clone <src_hostname> <dest_hostname>   # Clone scripts for a new dashboard
cdashly replace "<pattern>" <old> <new>        # Find/replace across matching files (preview by default, --apply to commit)
```

## Common Maintenance Tasks

### Updating for a New Slicer Release

Follow `maintenance/guides/rename-and-update-release-scripts.md`. Key steps: update version strings across stable/extension scripts using sed, update `GIT_TAG` to the new release tag, and on metroplex create a new `slicer-buildenv-*` script if the build environment changed.

### Updating CMake Version

From `maintenance/`:
```
make nightly-script-cmake-update CMAKE_VERSION=3.X.Y
make remote-install-cmake CMAKE_VERSION=3.X.Y
```

### Crontab Management

From `maintenance/`:
```
make pull-crontab             # Download current crontab from all dashboards
make push-crontab.computron   # Push crontab to a specific dashboard
```
