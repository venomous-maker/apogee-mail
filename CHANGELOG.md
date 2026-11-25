# Changelog

Every release of apogee-mail, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

Needs grav 1.13.0 or newer.

- Markdown comes from std's `Markdown` (std.text), which this package's own
  converter became; `Markdown.toHtml(text, true)` is the new safe mode.

## 0.2.2 (2026-11-21)

### Requirements

- Uses apogee-queue v0.3.0.

## 0.2.1 (2026-11-20)

### Requirements

- Uses the 0.2.1 releases of apogee-queue and grav-cache.

## 0.2.0 (2026-11-20)

### Requirements

- Needs grav 1.9.0 or later, and uses the 0.2.0 releases of the packages it
  depends on.

## 0.1.1 (2026-11-19)

- Uses the newer releases of apogee-queue, grav-cache, grav-orbit, grav-db (faster row loading and query building in grav-orbit and grav-db).

## 0.1.0 (2026-11-19)

The first tagged release. Needs grav 1.7.0 or newer.

Mail and notifications for [Apogee](https://github.com/venomous-maker), as Laravel's:
