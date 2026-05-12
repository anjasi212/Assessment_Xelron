# Part 2: Pull Request Analysis

## Repository: beetbox/beets

The beets repository contains 10 provided PRs. After reviewing all of them, I selected **PR #3877** (Web plugin readonly mode) and **PR #3568** (AlbumInfo/TrackInfo class refactor) as the two most comprehensible based on their clear problem statements and well-scoped changes.

---

## PR 1: beetbox/beets #3877 — Web Plugin Read-Only Mode

**Link:** https://github.com/beetbox/beets/pull/3877

### PR Summary

This PR addresses a security concern raised in issue #3870: the beets web plugin, which exposes the music library over HTTP via a Flask app, allowed DELETE and PATCH operations by default without any access control. Any client that could reach the HTTP endpoint could delete items or modify metadata. The PR introduces a new `readonly` configuration option for the `web:` section of `config.yaml`. When `readonly` is `true` (the new default), only HTTP GET requests are permitted. Mutating operations (DELETE, PATCH) are disabled unless the user explicitly sets `readonly: false` in their configuration. This is a deliberate breaking change accepted by the maintainers; existing users who relied on write access can restore it with a one-line config change.

### Technical Changes

- **`beetsplug/web/__init__.py`** — Added Flask route guards that check `app.config['READONLY']` before processing DELETE and PATCH endpoints; routes return a 405 or 403 response when readonly mode is active. The beets `self.config['readonly']` value is read at startup and propagated into `app.config['READONLY']` so Flask can check it per request.
- **`docs/plugins/web.rst`** — Added documentation for the new `readonly` config key, its default value (`true`), and instructions for re-enabling write access.
- **`test/test_web.py`** — Added test cases covering: DELETE and PATCH with `readonly=true` (should be rejected), DELETE and PATCH with `readonly=false` (should succeed), and the default behavior without any explicit setting.
- **`CHANGES.rst`** — Changelog entry added noting the breaking change and migration path.

### Implementation Approach

The implementation adds a startup hook inside the `WebPlugin` class (beets plugin entry point) that reads the `readonly` value from the beets configuration subsystem (`self.config['readonly'].get(bool)`). This boolean is then written into the Flask application's own config dictionary under the key `READONLY`. Because Flask's `app.config` is a thread-safe dictionary accessible from any view function, each route handler for DELETE and PATCH can simply check `current_app.config['READONLY']` and return an error response immediately if it is `True`. This avoids threading issues and keeps the guard logic close to the route definitions rather than in middleware. The default value is set to `True` in the plugin's config template/defaults, so users who have not previously set the option get the secure default automatically on upgrade. The PR deliberately kept the change minimal — no new abstraction layers, no token-based auth — because the goal was a simple, low-friction safety default rather than a full authentication system.

### Potential Impact

The change affects any user of the web plugin who uses DELETE (removing items from the library) or PATCH (updating metadata fields) over HTTP. Scripts, mobile apps, or integrations that called these endpoints will break silently until `readonly: false` is added to `config.yaml`. The GET endpoints (browsing, searching, streaming) are entirely unaffected. The Flask application lifecycle and plugin initialization are lightly touched but no other plugin or core beets code is affected.

---

## PR 2: beetbox/beets #3568 — AlbumInfo / TrackInfo AttrDict Refactor

**Link:** https://github.com/beetbox/beets/pull/3568

### PR Summary

This PR tackles a long-standing limitation in beets' autotag system (issue #1547): the `AlbumInfo` and `TrackInfo` data classes used to carry metadata from MusicBrainz and other sources were rigid, fixed-attribute `namedtuple`-like structures. Adding a new metadata field (for example, a new MusicBrainz attribute) required modifying the class definition itself, which was a barrier for plugins and external sources that wanted to attach extra fields to a candidate. The PR introduces a new `AttrDict` helper class — a dictionary subclass that also supports attribute-style (dot notation) access — and refactors `AlbumInfo` and `TrackInfo` to inherit from or use `AttrDict`. This means any key-value pair can be stored and retrieved with `info.my_new_field` as well as `info['my_new_field']`, making the structures extensible without changing core code.

### Technical Changes

- **`beets/autotag/hooks.py`** — Introduced the `AttrDict` class (a `dict` subclass overriding `__getattr__` and `__setattr__`). `AlbumInfo` and `TrackInfo` are refactored to subclass `AttrDict` instead of being plain classes with fixed slots, allowing arbitrary extra fields to be stored.
- **`beets/autotag/__init__.py` (`apply_metadata`)** — Updated the metadata application logic to correctly read fields from the new `AttrDict`-backed objects, ensuring that when a match is applied to a track or album, all stored fields (both standard and plugin-provided) are handled properly.
- **`CHANGES.rst`** — Changelog entry describing the new extensibility.

### Implementation Approach

The key design decision is the `AttrDict` class. By overriding `__getattr__` to delegate to `dict.__getitem__` and `__setattr__` to delegate to `dict.__setitem__`, objects behave like dictionaries while still allowing `obj.field` syntax. This is important for backwards compatibility: existing code that accesses `info.artist`, `info.album_id`, etc. continues to work without change. New code (plugins, new sources) can add fields simply by assigning to a new attribute: `info.my_field = value`. The `AlbumInfo` and `TrackInfo` constructors are updated to accept keyword arguments and populate the dictionary, preserving the call signature used throughout the codebase. The `apply_metadata` function in `__init__.py` was an important companion fix: the original code iterated over a hardcoded set of field names; the refactor ensures it correctly reads fields from the dict backing store so that plugin-added fields are also applied to the library item.

### Potential Impact

Plugins that construct `AlbumInfo` or `TrackInfo` objects directly (e.g., custom metadata sources) may need minor updates if they relied on positional arguments or the exact class type. The `apply_metadata` path in the autotag pipeline is touched, which is a core hot path during import; any regression here would affect all users doing `beet import`. However, because the external API surface (attribute names) is preserved, most downstream code should be unaffected.
