# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`forge` is a [BlueBuild](https://blue-build.org) recipe that defines a custom Fedora Atomic (rpm-ostree) desktop image — a KDE workstation for software development, layered on top of `ghcr.io/ublue-os/aurora-dx-nvidia-open`. There is no application source code here; the "build" is a container image assembled by BlueBuild from declarative YAML plus optional shell scripts, and it is built and published entirely by CI (GitHub Actions), not locally.

## Repository layout

- `recipes/recipe.yml` — the single source of truth for the image. Declares the base image, Fedora package installs/removals (`dnf` module), fonts, systemd services to enable, Flatpaks, and image signing. Modules run in the order listed.
- `files/system/` — files copied verbatim into the image filesystem (mirrors the target root, e.g. `files/system/etc/...`, `files/system/usr/...`). Currently empty scaffolding (`.gitkeep` only).
- `files/scripts/` — shell scripts that can be invoked from `recipe.yml` via a `script` module during the build. `example.sh` shows the required pattern (`set -oue pipefail`).
- `modules/` — placeholder for custom/local BlueBuild modules (currently empty scaffolding).
- `cosign.pub` — public key for verifying signed image releases via `cosign verify --key cosign.pub ghcr.io/johangwbasson/forge`.
- `.github/workflows/build.yml` — the actual build pipeline: runs on push (non-doc changes), PRs, a daily 06:00 UTC schedule, and manual dispatch; uses `blue-build/github-action` to build and publish every recipe listed under `matrix.recipe`.

## Working with recipe.yml

- Packages to install go under the `dnf` module's `install.packages` list; packages to strip from the base image go under `remove.packages`. Keep the existing grouped-by-comment sections (Development/CLI, Build, KDE applications, Docker, Multimedia, etc.) and add new packages to the matching group rather than creating ad-hoc entries.
- Extra DNF repos (e.g. RPM Fusion, Docker CE) are declared under the same `dnf` module's `repos` key.
- Fonts are split across two modules: system packages for Noto/IBM Plex fonts via `dnf`, and the dedicated `fonts` module for nerd-fonts/google-fonts installed by name.
- Flatpaks are declared under the `default-flatpaks` module, split by `scope: system` (available to all users) vs `scope: user`.
- Services to enable at boot go under the `systemd` module's `system.enabled` list.
- The `signing` module must remain the last module in the list — it signs the final image.
- There is a commented-out NVIDIA section noting that the old `nvidia.sh` script should **not** be carried back into the image, and that the correct Aurora NVIDIA base/akmods configuration needs verification before that block is enabled. Respect that note if working in this area — don't reintroduce a custom nvidia script without addressing the stated verification concern.
- Multiple recipes can exist under `recipes/`; each one must be added to the `matrix.recipe` list in `.github/workflows/build.yml` to actually be built.

## Validating changes

There is no local build/lint/test tooling in this repo — validation happens through the BlueBuild GitHub Action in CI. When editing `recipe.yml`:
- Check changes against the schema referenced at the top of the file (`https://schema.blue-build.org/recipe-v1.json`).
- Consult the [BlueBuild docs](https://blue-build.org/how-to/setup/) for module syntax when adding a module type not already used here.
- A PR triggers a real CI build via `build.yml`, which is the practical way changes get verified.

## Scripts

Any script placed in `files/scripts/` and referenced from a `script` module in `recipe.yml` must start with `set -oue pipefail` (see `files/scripts/example.sh`) so the build fails loudly on error instead of silently continuing.
