# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

noDRM's fork of Apprentice Harper's DeDRM tools: two Calibre plugins that strip
DRM from ebooks, plus assorted standalone tools. Everything ships as Calibre
plugin zips — there is no installable Python package.

## Build

```bash
python3 make_release.py            # -> DeDRM_tools.zip
python3 make_release.py 10.0.28    # -> DeDRM_tools_10.0.28.zip
```

This is the only build command. The outer zip contains `DeDRM_plugin.zip`,
`Obok_plugin.zip`, the user READMEs, and the Windows/Frida KRF key-extraction
helpers. The script deletes its own `release/` working dir on completion, so the
outer zip is the artifact to inspect.

Plugin version lives in `DeDRM_plugin/__version.py`.

CI (`.github/workflows/main.yml`) runs the same script on every push to `master`
and publishes a prerelease "alpha" to the separate `noDRM/DeDRM_tools_autorelease`
repo. **CI builds only — it runs no tests.**

## Testing

There are none. No test files, no pytest/unittest config, no `requirements.txt`,
`setup.py`, or `pyproject.toml`. `DeDRM_plugin/epubtest.py` is a DRM-detection
utility, not a test. Runtime dependencies come from Calibre's bundled Python
rather than pip.

Verification is manual, through Calibre:

```bash
python3 make_release.py
unzip -o DeDRM_tools.zip -d /tmp/dedrm_release
calibre-customize --add /tmp/dedrm_release/DeDRM_plugin.zip
calibredb add <drm-book>      # or import via the Calibre GUI
```

See `CALIBRE_CLI_INSTRUCTIONS.md` for the fuller CLI walkthrough.

## The CALIBRE_COMPAT_CODE mechanism

**The build rewrites source files.** Modules in `DeDRM_plugin/` carry a bare
`#@@CALIBRE_COMPAT_CODE@@` comment line. At package time,
`make_release.py:patch_file()` replaces that line with the body of
`DeDRM_plugin/__calibre_compat_code.py`, which fixes up `sys.path` and sets
`__package__ = "calibre_plugins.dedrm"`.

Any new module in `DeDRM_plugin/` that imports sibling modules **must** include
the marker, placed after the docstring/imports and before the module's own logic
(see `DeDRM_plugin/__version.py:4`). Without it the module imports fine when run
directly and fails inside Calibre.

28 of 48 files carry the marker; leaf modules with no sibling imports omit it.
`Obok_plugin/` uses it **zero** times — that directory is zipped as-is. Don't add
it there.

Both plugins pin their Calibre import name with a `plugin-import-name-*.txt`
marker file, which is what makes `calibre_plugins.dedrm` and
`calibre_plugins.obok_dedrm` resolve regardless of the zip filename.

## Architecture

The two plugins use different Calibre base classes, for a reason:

- **`DeDRM_plugin/`** — a `FileTypePlugin` (`__init__.py:150`), with
  `on_import = True` and `priority = 600`. Calibre calls `run(path_to_ebook)`
  automatically on import; it dispatches by detected format and DRM scheme. It
  falls back to a stub base class when `calibre` isn't importable, so the file
  can also run as a plain script.
- **`Obok_plugin/`** — an `InterfaceActionBase` delegating to
  `action.py:InterfacePluginAction`. A toolbar button rather than a passive
  hook, because Kobo requires an interactive book-picker dialog (`dialogs.py`).

Each plugin zip is self-contained. The two `utilities.py` files are unrelated
implementations — **do not deduplicate them into a shared module**, it would
break packaging.

### DeDRM_plugin module layout

Split along two axes — decrypting a *format* versus retrieving a *key*:

- **Format/decrypt engines:** `k4mobidedrm.py` (Kindle dispatcher),
  `mobidedrm.py`, `topazextract.py`, `kfxdedrm.py` + `ion.py` (Amazon Ion/KFX),
  `ineptepub.py`, `ineptpdf.py` (Adobe ADEPT), `erdr2pml.py` (eReader pdb).
- **Key retrieval, one module per platform/app:** `kindlekey.py` (Mac/PC),
  `androidkindlekey.py`, `adobekey*.py` (ADE), `ignoblekey*.py` (B&N/Nook),
  `kgenpids.py` and `kindlepid.py` (PID generation).
- **Crypto & containers:** `alfcrypto.py` (PC1 and Topaz ciphers), `aescbc.py`,
  `zipfix.py` / `zipfilerugged.py` / `zeroedzipinfo.py` (tolerating malformed
  DRM'd zip containers).
- **Config & platform glue:** `prefs.py`, `config.py`, `utilities.py`,
  `wineutils.py` (runs Windows-only key scripts under Wine on Linux).

## Constraints

- **Python 2/3 compatibility is required**, not optional. Calibre 5.x/6.x run
  Python 3, but Calibre 4.x and earlier run Python 2 and are still supported.
  Keep the `from __future__` imports; avoid Py3-only syntax.
- **`lcpdedrm.py` is an intentional stub.** Readium LCP support was removed
  following a DMCA takedown. Do not implement it. Apple FairPlay is likewise out
  of scope.
- **`DeDRM_plugin/standalone/` is incomplete.** Its own docstring says the CLI is
  not functional yet; Calibre is the supported path. Don't present it as a
  working entry point.
- `Other_Tools/` holds non-packaged extras — Frida/KRF Kindle key extraction
  (with checked-in ELF binaries), a B&N download userscript, and legacy format
  tools. Not part of the plugin build.
- There is no `LICENSE` file at the repo root.
