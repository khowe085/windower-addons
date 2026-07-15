# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of **Windower 4 addons** for Final Fantasy XI, written in **Lua 5.1**. Windower is a Windows-only launcher/injection platform that loads addons at runtime and exposes them a global API. The repo is early — tooling, config, and a template addon (`example/`) are in place; real addons are added as sibling folders.

Because addons run inside Windower on Windows, they cannot be executed or fully tested in this Linux devcontainer. Development here is authoring, linting, and formatting; runtime verification happens in a live Windower/FFXI client on Windows.

## Toolchain & commands

The devcontainer (Alpine) provides Lua 5.1, luacheck, StyLua, busted, and the GitHub CLI.

- **Test:** `busted` — runs every `tests/*_spec.lua` (config in `.busted`). Filter with `busted tests/status_spec.lua` or `busted --filter "status"`.
- **Lint:** `luacheck .` — config in `.luacheckrc` (std `lua51`). luacheck auto-applies its `busted` std to `*_spec.lua`, so `describe`/`it`/`assert` need no whitelist.
- **Format:** `stylua .` (or `stylua --check .`) — config in `stylua.toml` (2-space indent). Also runs on save in VS Code via the StyLua extension.
- **Run a script standalone:** `lua <file>` — only for pure Lua with no Windower globals; anything touching the addon API must run inside Windower.

There is no build step. "Passing" locally means green `busted` + clean `luacheck` + `stylua --check`.

## Windower addon conventions

These are platform facts that are not discoverable from the (currently empty) source tree:

- **Globals in `.luacheckrc`.** The whitelisted globals are `windower`, `texts`, `res`, `packets`, `files`, `_addon`. Two kinds are mixed here:
  - *Truly injected by Windower:* `windower` (core API — events, chat, packet send, ffxi/player/target data) and `_addon` (addon metadata: name, version, author, commands).
  - *Libraries assigned to globals by convention:* `texts`/`res`/`packets` are loaded with `require('texts')` / `require('resources')` / `require('packets')` and conventionally stored in a same-named global; `files` similarly wraps addon-relative file I/O. See the Libraries wiki below. When you introduce a new global, register it in `.luacheckrc` so luacheck stays clean.
- **Addon layout.** An addon named `SomeName` lives in a lowercase folder `somename/` with a main file `somename.lua` (name must match the folder) plus a `readme.md`. User settings live in `somename/data/settings.xml`, which is **created at runtime and must not be committed** (add `data/` to gitignore). The main file declares metadata on the injected `_addon` table: `_addon.name`, `_addon.author`, `_addon.version`, `_addon.command` (single `//` shortcut, e.g. `//sn`), and optional `_addon.commands` (aliases). Every source file carries the **BSD 3-clause license header** (Windower convention — fill in `<YEAR>`/`<COPYRIGHT HOLDER>`).
- **Settings.** Use the `config` library with hardcoded defaults: `settings = config.load(T{...})`, then `settings:save()` (per-character) or `settings:save('all')` (global). New keys in defaults are merged into the user's XML automatically.
- **Events.** Behavior is driven by handlers registered with `windower.register_event(...)`. Common events: `'load'`, `'unload'`, `'addon command'` (from the addon's command), `'incoming text'` / `'outgoing text'`, `'incoming chunk'` / `'outgoing chunk'` (packets), `'prerender'` / `'postrender'`, `'login'` / `'logout'`, `'zone change'`, `'status change'`. Text/chunk events may return modified data or a block flag.
- **Common libraries** (`require(...)`): `config` (Windower-XML settings load/save), `logger`, `tables`/`strings`/`sets`/`lists`/`functions` (data-structure & functional helpers), `resources`, `packets`, `texts`, `images`, `json`, `chat`. See the Libraries reference below.
- **Gotchas that cause real bugs** (fuller treatment in the mirrored Lua guide below):
  - `windower.ffxi.get_player()` / `get_mob_by_index()` can return `nil` (not logged in, index not loaded) — guard before indexing. A PC has both a permanent ID and a per-zone **mob index**; use `target_index`/index for in-zone lookups.
  - Windower and Lua **fail silently**: too few/many function args yield `nil`/dropped values, and undefined table keys return `nil`, all without erroring. A `nil` in an array also halts `ipairs`/`#` iteration early. Validate inputs yourself.
  - Tables are **1-indexed**; there's no built-in string split or table slice (use `strings`/`tables`). `T{}` (from `tables`) is needed for method-style calls like `t:map(...)`; convert the mob/player arrays with `T(...)`.

## Modular design & testing

Addons are structured so the logic is unit-testable **without a running Windower client**, using **dependency injection**. The rule: pure logic never touches Windower globals directly — it receives what it needs as a `deps` table.

Layout per addon (see `example/` for a working template):

```
somename/
  somename.lua     -- entry point: the ONLY file that reads Windower globals
  lib/*.lua        -- pure modules; factory style `new(deps) -> instance`
tests/
  *_spec.lua       -- busted specs; inject fake deps, no globals needed
  support/*.lua    -- reusable fakes / builders for deps tables
```

- **Pure modules (`lib/`).** Each returns a constructor `new(deps)`. It calls `deps.get_player()` etc. — never `windower.*` directly. Because deps are plain functions, a test "mock" is just a table of stubs; no mocking framework required.
- **Entry point (`somename.lua`).** The only place that reads globals. It builds the real `deps` from the live API (`get_player = function() return windower.ffxi.get_player() end`), injects it into the `lib/` modules, and wires `windower.register_event(...)`. Keep it thin — anything worth testing belongs in `lib/`.
- **Tests (`tests/`).** busted specs `require` a `lib/` module and pass a fake `deps`. See `tests/status_spec.lua` and `tests/support/fakes.lua`.
- **Require paths.** `.busted` sets `lpath` so `require('lib.foo')` resolves both in tests and at Windower runtime (the addon's own folder is its require root). When adding an addon whose `lib/` module names could collide with another's, run busted scoped to that addon or extend `lpath`.

## Line endings (important)

`.gitattributes` enforces **CRLF in the working tree** (`* text=auto eol=crlf`) because this targets Windows/Windower; git still stores LF-normalized blobs, so diffs stay clean. The sole exception is `.devcontainer/Dockerfile`, which is forced to LF for the Linux container. Don't fight the CRLF checkout — it's intentional.

## Reference: Windower Lua API

Authoritative docs for the addon API (consult these rather than guessing at signatures):

- **Wiki home:** https://github.com/Windower/Lua/wiki/
- **Writing Addons** (structure, `_addon` metadata, boilerplate): https://github.com/Windower/Lua/wiki/Writing-Addons
- **Windower Lua API:** https://github.com/Windower/Lua/wiki/Windower-Lua-API
- **Events** (`register_event` names & payloads): https://github.com/Windower/Lua/wiki/Events
- **Functions** (the `windower.*` function reference): https://github.com/Windower/Lua/wiki/Functions
- **Libraries** (`require`-able helpers): https://github.com/Windower/Lua/wiki/Libraries
- **Lua Guide for programmers** — full text mirrored locally at [docs/lua-guide-for-programmers.md](docs/lua-guide-for-programmers.md) (Lua idioms & Windower gotchas for devs coming from other languages). The canonical forum URL 404s; a static archive survives at `https://forums.windower.net/index.php@sharelink=download%3BaHR0cDovL2ZvcnVtcy53aW5kb3dlci5uZXQvaW5kZXgucGhwPy90b3BpYy82OTctbHVhLWd1aWRlLWZvci1wcm9ncmFtbWVycy8,%3BTHVhIEd1aWRlIGZvciBwcm9ncmFtbWVycw,,.html`.
