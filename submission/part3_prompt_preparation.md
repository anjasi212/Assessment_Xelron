# Part 3: Prompt Preparation

**Selected PR:** beetbox/beets #3877 — Web Plugin Read-Only Mode

---

## 3.1.1 Repository Context

Beets is a command-line music library management tool written entirely in Python. It is aimed at technically inclined music enthusiasts — people who care deeply about having well-organized, correctly tagged audio files and want programmatic control over their collection. The project has been active for over a decade and has a large plugin ecosystem.

The core of beets is a SQLite-backed library database that stores metadata for every track and album the user imports. During import, beets queries external services like MusicBrainz to fetch accurate metadata (artist, album, year, genre, track numbers, etc.) and writes it back to the audio files' embedded tags. Beyond import, beets exposes a rich query language for searching the library, a flexible file path templating system, and a plugin hook architecture that lets both built-in and third-party plugins add new commands, new fields, and new behaviors.

One of the built-in plugins is the `web` plugin, which spins up a local Flask HTTP server that exposes the beets library over a REST-like API. It allows users to browse their music library from a browser, search for tracks, stream audio files, and — before this PR — also modify or delete library entries via HTTP DELETE and PATCH endpoints. This plugin is particularly useful for users who want to build simple front-ends, home media server integrations, or mobile access to their local collection.

The intended users of beets are individual users running it on their own machine or a home server. They are typically comfortable with YAML configuration files and a command-line interface but may not be security specialists. The web plugin is often exposed on a local network, making default-secure behavior important.

---

## 3.1.2 Pull Request Description

Before this PR, the beets web plugin's Flask application exposed HTTP DELETE and PATCH endpoints with no access control whatsoever. A DELETE request to `/item/<id>` would remove that item from the beets library, and a PATCH request could overwrite metadata fields. There was no authentication, no confirmation, and no way to disable this behavior short of not loading the plugin at all. Issue #3870 documented user concern that this was dangerous in any networked context — even a local network — because any device or script that could reach the server could silently corrupt or destroy the library.

This PR introduces a `readonly` configuration key inside the `web:` plugin section of beets' `config.yaml`. When `readonly` is set to `true` (which is the new default value), the Flask server rejects any incoming DELETE or PATCH request, returning an appropriate HTTP error. GET requests for browsing and streaming continue to work normally. When a user explicitly sets `readonly: false` in their config, the old behavior is restored and write operations are permitted again.

The previous behavior was: all endpoints (GET, DELETE, PATCH) were always enabled, and there was no configuration option to change this. The new behavior is: the server starts in read-only mode by default; write endpoints return errors unless the user has consciously opted in to allowing mutations. This is a deliberate breaking change: anyone using DELETE or PATCH via the web plugin will find their requests rejected after upgrading until they add `readonly: false` to their configuration.

---

## 3.1.3 Acceptance Criteria

✓ When `readonly` is not set in `config.yaml`, the web plugin must default to read-only mode, and HTTP DELETE requests to `/item/<id>` or `/album/<id>` must return a non-2xx error response (e.g., 405 Method Not Allowed or 403 Forbidden).

✓ When `readonly: false` is explicitly set in the `web:` section of `config.yaml`, the web plugin must allow HTTP DELETE and PATCH requests to mutate the library as they did before the change.

✓ When `readonly: true` is explicitly set in the `web:` section of `config.yaml`, the web plugin must reject DELETE and PATCH requests even if a previous version of the code accepted them.

✓ All existing GET endpoints (item listing, album listing, item detail, album detail, audio streaming) must continue to work correctly regardless of the value of `readonly`.

✓ The beets documentation for the `web` plugin must include the `readonly` configuration option, describe its default value, explain the behavior it controls, and provide the one-line config snippet a user needs to re-enable write access.

✓ The implementation must correctly bridge the beets configuration system (`self.config['readonly']`) and the Flask application configuration (`app.config['READONLY']`) so that the setting read from `config.yaml` is the one actually enforced at request time.

✓ The CHANGES.rst changelog must document this as a potentially breaking change with clear migration guidance.

---

## 3.1.4 Edge Cases

**Edge Case 1: Config value not present at all (missing key)**
A user who has been running beets with the web plugin before this PR will not have `readonly` in their `config.yaml`. The implementation must handle a missing key gracefully by falling back to `True` (read-only). Using `.get(bool, True)` or an equivalent default in the beets confuse configuration accessor is required; raising a `KeyError` or `ConfigError` at startup would be a regression.

**Edge Case 2: Config value set via environment variable or CLI override**
Beets supports overriding configuration values via environment variables and command-line flags. If a user sets `readonly` via these mechanisms rather than `config.yaml`, the Flask app must still pick up the correct value. The implementation must read from the beets config abstraction layer (not directly from a file) so that all config sources are respected.

**Edge Case 3: PATCH request to a valid item while in readonly mode**
A PATCH request to `/item/123` with a valid JSON body should be rejected with a clear HTTP error (not a 500 server error, not a 200 success, and not a silent no-op). The error response should be consistent with REST conventions — a 405 Method Not Allowed is appropriate — and should not partially apply any changes before returning the error.

**Edge Case 4: Concurrent requests during startup**
If the Flask server begins accepting connections before the `app.config['READONLY']` key is set (e.g., due to a race in multi-threaded or async Flask configurations), a request could bypass the guard. The readonly flag must be set synchronously before the server begins listening, not lazily on first request.

---

## 3.1.5 Initial Prompt

```
You are implementing a change to the `beets` open-source music library manager (https://github.com/beetbox/beets), specifically to its built-in `web` plugin located in `beetsplug/web/__init__.py`.

Background:-

Beets is a Python CLI tool for managing music libraries. The `web` plugin runs a local Flask HTTP server exposing the library as a REST-like API. Currently, the server allows HTTP GET, DELETE, and PATCH operations with no access control. DELETE removes library items; PATCH modifies metadata. This is a security risk on any networked environment. Issue #3870 requests a way to make the server read-only by default.

What You Need to Implement:-

Add a new boolean configuration option called `readonly` to the `web` plugin section of beets' config system. This option must:

1. Default to `True` (read-only) if not present in the user's `config.yaml`.
2. When `True`, cause the Flask server to reject HTTP DELETE and PATCH requests with a 405 Method Not Allowed response. GET requests must continue to work normally.
3. When `False`, allow DELETE and PATCH to work as they currently do.
4. Be read from beets' configuration layer (`self.config['readonly']` inside the plugin class) and stored in Flask's app config as `app.config['READONLY']` so that route handlers can access it.

Files to Modify:-

- `beetsplug/web/__init__.py`: The plugin class has a `commands()` method that sets up and starts the Flask app. Read `self.config['readonly'].get(bool)` there and set `app.config['READONLY']` before calling `app.run()`. Add guards at the top of every route handler that performs DELETE or PATCH — check `current_app.config.get('READONLY', True)` and return `flask.make_response('...', 405)` if it is True.
- `docs/plugins/web.rst`: Add a documentation entry for `readonly` under the Configuration section, explaining the default, its effect, and providing an example snippet showing how to set `readonly: no`.
- `CHANGES.rst`: Add a changelog entry clearly marked as a breaking change for users relying on DELETE/PATCH via the web plugin.
- `test/test_web.py`: Add tests that (a) verify DELETE is rejected with the default config, (b) verify DELETE succeeds when `readonly: false` is configured, (c) verify PATCH is rejected in readonly mode, (d) verify PATCH succeeds when readonly is disabled.

Acceptance Criteria:-

- Requests rejected in readonly mode must return HTTP 405, not 200 or 500.
- GET endpoints (item listing, streaming, search) must be unaffected in all modes.
- A missing `readonly` key in config must behave identically to `readonly: true`.
- The beets config value and the Flask app config value must be in sync at server startup, not resolved lazily per request.
- All new tests must pass, and no existing tests must break.

Edge Cases to Handle:-

- User has no `readonly` key in config at all → must default to True (safe default).
- User sends PATCH with a valid body in readonly mode → reject the entire request before applying any changes.
- Ensure the READONLY flag is set on `app.config` before `app.run()` is called to avoid any race with incoming connections.

Testing:-

Write pytest-style tests using the existing test harness in `test/test_web.py`. The test infrastructure already includes a helper to set up a beets config and a test Flask client. Add a fixture or helper that starts the web app with `readonly=True` and another with `readonly=False`, then assert the HTTP status codes for DELETE, PATCH, and GET requests.

Please implement all changes with explanatory inline comments, keep the implementation minimal (no new auth framework — just a flag check), and follow the existing code style in the file.
```
