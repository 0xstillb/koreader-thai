# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

KOReader Thai (Kindle edition) — a fork of KOReader with Thai word segmentation for Kindle e-readers.
Uses **libthai + libdatrie only** (no Deepcut, no ONNX Runtime). Based on v2026.04.21-thai.1 tag.

### Repos
| Repo | Branch | Purpose |
|------|--------|---------|
| koreader-thai | `kindle-thai-libthai` | Main app (Lua frontend, plugins, data) |
| koreader-base | `kindle-thai-libthai` | Build system, thirdparty libs (libthai, libdatrie, crengine) |
| crengine | `kindle-thai-libthai` | EPUB/HTML rendering engine with Thai word-break hook |

The `thai` branch in each repo is the **Android** release — do not modify it from this working directory.

## Thai Word Breaking Architecture

1. `frontend/document/credocument.lua` sets `LIBTHAI_DICTDIR=./data/thai` before crengine init
2. `data/thai/thbrk.tri` (588 KB) — prebuilt libthai dictionary
3. crengine's `lvtextfm.cpp` has a post-libunibreak hook that detects Thai spans (U+0E00–U+0E7F)
4. The hook calls `thaibreak.cpp` which uses libthai's `th_brk_find_breaks()` for word boundaries
5. libthai depends on libdatrie (double-array trie)

On Kindle, CWD is `/mnt/us/koreader` (set by `platform/kindle/koreader.sh`), so `./data/thai` resolves correctly.

## Build Commands (WSL)

```bash
# Kindle Paperwhite 2/3 (priority target)
./kodev release kindlepw2

# Kindle FW >= 5.16.3
./kodev release kindlehf

# Kindle >= Kindle4
./kodev release kindle

# Fetch submodules if needed
./kodev fetch-thirdparty
```

Cross-compilation toolchains must be in PATH: `arm-kindlepw2-linux-gnueabi-gcc`, `arm-kindlehf-linux-gnueabihf-gcc`, etc.
Install via [koxtoolchain](https://github.com/koreader/koxtoolchain), output goes to `~/x-tools/`.

## Key Files (Thai-specific)

| File | Purpose |
|------|---------|
| `frontend/document/credocument.lua` | Sets LIBTHAI_DICTDIR env var |
| `data/thai/thbrk.tri` | libthai word-break dictionary |
| `Makefile` (line 54) | `$(wildcard data/thai)` in DATADIR_FILES |
| `base/thirdparty/libdatrie/CMakeLists.txt` | Build libdatrie 0.2.13 static |
| `base/thirdparty/libthai/CMakeLists.txt` | Build libthai 0.1.29 static |
| `base/thirdparty/kpvcrlib/CMakeLists.txt` | Adds thaibreak.cpp to crengine |
| `base/thirdparty/kpvcrlib/crengine/crengine/src/thaibreak.cpp` | UTF-32↔TIS-620 + th_brk wrapper |
| `base/thirdparty/kpvcrlib/crengine/crengine/src/lvtextfm.cpp` | Post-libunibreak Thai hook |
| `base/thirdparty/kpvcrlib/crengine/crengine/src/textlang.cpp` | Thai lang detection |

## Kindle Runtime

- Install path: `/mnt/us/koreader/`
- Launcher: `platform/kindle/koreader.sh` — does `cd /mnt/us/koreader` before exec
- Data storage: relative to CWD (`.`)
- Dict path: `/mnt/us/koreader/data/thai/thbrk.tri`
- Packages: `.zip` (manual install) and `.targz` (OTA update)
- Install via KUAL extension

## Scope Rules

- ONLY Thai word breaking — no Deepcut, no ONNX, no SimpleUI
- Do not modify EPUB content or inject ZWSP
- Do not add Android-specific code
- Do not modify the `thai` branch (Android release)
- Keep patches minimal for upstream sync compatibility

## Verification

Test string: `ภาษาไทยเป็นภาษาที่ไม่มีช่องว่างระหว่างคำ`
Expected breaks: `ภาษา|ไทย|เป็น|ภาษา|ที่|ไม่|มี|ช่อง|ว่าง|ระหว่าง|คำ`
