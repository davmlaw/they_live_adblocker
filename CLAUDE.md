## CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is a fork of **uBOL-home** — the *distribution* repo for uBlock Origin Lite (uBOL), an MV3 content blocker. The actual extension source lives in the `uBlock/` git submodule (gorhill/uBlock). This outer repo's job is to (a) build the MV3 packages from that submodule and (b) hold the built artifacts (`chromium/`, `firefox/`, plus `dist/safari/`) that are committed and shipped to the stores.

Most automated commits (`[workflow] Update uBOLite MV3 package files for ...`) are produced by `.github/workflows/create_release.yml` and represent a fresh rebuild — they are not hand-edited source.

## Submodules (required for any build)

```bash
git submodule update --init --recursive
```

Two submodules:
- `uBlock/` — gorhill/uBlock source tree; the real codebase
- `publish-extension/` — store-publishing tooling used by the top-level `Makefile`

A checkout without submodules initialized cannot build anything.

## Building MV3 packages

Builds run from **inside** the `uBlock/` submodule, not from the repo root. The release workflow uses:

```bash
cd uBlock
tools/make-mv3.sh <platform> <version> before=<path-to-this-repo>
# platform: chromium | edge | firefox | safari
```

Output: `uBlock/dist/build/uBOLite.<platform>/`.

The `before=<path-to-this-repo>` flag is **not** a customization overlay — it only feeds `salvage-ruleids.mjs`, which compares the previously-released DNR ruleset against the new build and reuses rule IDs to minimize diff size in the committed `rulesets/` JSON. Fork-specific *code* changes cannot live in the outer repo; they must live in the `uBlock/` submodule (or be applied to it during build).

Top-level `Makefile` only wraps Safari builds (`make safari-extension`, `make safari-app`) and store publishing (`make publish-chromium version=…`, etc.). For chromium/edge/firefox the workflow goes through `uBlock/tools/make-mv3.sh` directly.

## Lint

```bash
npm install        # or: make init
npm run lint       # only lints docs/tests/*.js — there is no project-wide JS lint
```

No test suite is wired up (`npm test` is a stub).

## Layout of the built extension (chromium/, firefox/)

These directories are *generated output* committed to the repo. The interesting subdirs:
- `manifest.json` — MV3 manifest; `declarative_net_request.rule_resources` enumerates ruleset IDs
- `rulesets/main/*.json` — DNR rule files (EasyList, EasyPrivacy, ublock-filters, etc.); `rulesets/scripting/`, `rulesets/redirect/`, `rulesets/css/` hold cosmetic/scriptlet payloads
- `js/` — extension JS (popup, dashboard, picker, zapper, background service worker, DNR editors). `js/resources/` and `js/offscreen/` contain scriptlets and the offscreen document
- `_locales/` — i18n
- `web_accessible_resources/` — assets injectable into pages

For fork-specific changes, edit source in the `uBlock/` submodule (`uBlock/platform/mv3/extension/...` for MV3-only files, `uBlock/src/...` for shared assets) and rebuild. **Hand-edits to `chromium/`/`firefox/` in this outer repo will be clobbered by the next release workflow run** — those directories are pure build output. To track customizations cleanly, either fork the `gorhill/uBlock` submodule and point `.gitmodules` at the fork, or maintain a patch series applied to the submodule at build time.

## Releasing

`.github/workflows/create_release.yml` (manual `workflow_dispatch`) generates a calendar version `YYYY.M*100+D.H*100+M`, rebuilds all platforms, replaces the `chromium/` and `firefox/` directories in the repo, zips each platform package, and creates a GitHub release. Don't try to reproduce the version-bumping commit manually — trigger the workflow.

Store-upload Makefile targets (`publish-chromium`, `publish-edge`, `publish-firefox`, `publish-safari-*`) download the GitHub release asset by tag and push it to the respective store. They require credentials (see `secret-tool lookup` calls in `Makefile`).
