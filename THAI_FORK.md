# KOReader — Thai fork

This fork adds **dictionary-based Thai word segmentation** to KOReader's
EPUB/HTML line breaker. Upstream KOReader inherits libunibreak's UAX #14
behaviour for Thai (class SA), which means Thai paragraphs either render as
one unwrappable run or break at every character — neither is acceptable for
Thai readers.

The fix is an **additive patch**: libthai + libdatrie are bundled as new
static thirdparty libraries, crengine's `lvtextfm.cpp` gets a single post-
pass hook after the libunibreak loop, and the hook hands Thai-containing
spans to `th_brk_find_breaks()` so real word boundaries get
`LCHAR_ALLOW_WRAP_AFTER`. Non-Thai pages are untouched (fast-path Unicode
range check before any allocation).

## What changed

| Repo / path | Change |
|---|---|
| `base/cmake/CMakeLists.txt` | Register `libdatrie` and `libthai` thirdparty projects; declare them as crengine build dependencies. |
| `base/thirdparty/libdatrie/CMakeLists.txt` | **New.** Build libdatrie 0.2.13 as a static library via `external_project`. |
| `base/thirdparty/libthai/CMakeLists.txt` | **New.** Build libthai 0.1.29 as a static library (depends on libdatrie via pkg-config). |
| `base/thirdparty/cmake_modules/koreader_thirdparty_libs.cmake` | Expose `libdatrie::datrie` and `libthai::thai` dependency targets; add them to the crengine link line. |
| `base/thirdparty/kpvcrlib/CMakeLists.txt` | Add `thaibreak.cpp` to the crengine source list. |
| `base/thirdparty/kpvcrlib/crengine/crengine/src/thaibreak.cpp` | **New.** UTF-32 ↔ TIS-620 conversion + `th_brk_find_breaks()` wrapper; caches a singleton `ThBrk *` handle. |
| `base/thirdparty/kpvcrlib/crengine/crengine/include/textlang.h` | Add `dict_word_break_func_t` typedef, `_dict_word_break_func` member, `_is_th` flag, `isThai()` and `getDictWordBreakFunc()` accessors. |
| `base/thirdparty/kpvcrlib/crengine/crengine/src/textlang.cpp` | Register `"th"` in `_hyph_dict_table` (no hyph dict); detect Thai lang tag; wire `_dict_word_break_func = &thai_dict_break`. |
| `base/thirdparty/kpvcrlib/crengine/crengine/src/lvtextfm.cpp` | Post-libunibreak pass: per span, if `src->lang_cfg->getDictWordBreakFunc()` is set and the span contains Thai codepoints (U+0E00–U+0E7F), invoke it on `(m_text, len, m_flags)`. |
| `frontend/document/credocument.lua` | `setenv("LIBTHAI_DICTDIR", "./data/thai", 1)` before crengine loads, so libthai finds the bundled `thbrk.tri`. |
| `data/thai/README.md` | Provisioning notes for the `thbrk.tri` dictionary asset. |

## Build

> ⚠ **Windows**: KOReader's build tooling (`kodev`, `make`, autotools,
> Android NDK) is a POSIX pipeline. On Windows you **must** build inside
> WSL2 or a Linux VM. The clone on Windows is only for code editing.

From a Linux/macOS/WSL shell:

```sh
cd koreader
./kodev fetch-thirdparty            # clones/downloads libthai + libdatrie
./kodev build android --fast        # Android NDK build; first run is slow
```

The resulting APK sits under `build/android-arm/`. Install it on a device
or emulator. On first run, KOReader copies its `data/` tree to app private
storage; `data/thai/thbrk.tri` must be present in the source tree at
package time.

## Verify

1. Open an EPUB containing Thai text. The canonical canary is:

   > ภาษาไทยเป็นภาษาที่ไม่มีช่องว่างระหว่างคำ

   Expected wrap points: `ภาษา|ไทย|เป็น|ภาษา|ที่|ไม่|มี|ช่อง|ว่าง|ระหว่าง|คำ`.

2. Open a mixed-language EPUB (Thai + English + digits in one paragraph).
   English must still break on spaces, Thai on words — per-span `lang_cfg`
   means the hook only fires for Thai spans.

3. Regression-test with an English EPUB and a Chinese EPUB. Rendering
   must be identical to upstream (the Thai hook is a no-op: the fast-path
   Unicode-range check bails out before any allocation).

## Scope / non-goals

- **Android only**, per the approved plan. Kindle/Kobo/Linux/macOS may work
  — the patch itself is portable — but they are not part of the verification
  matrix for this fork.
- PDF reflow is out of scope. Thai PDFs are fixed-layout; reflow via
  k2pdfopt is a separate problem.
- UI text (menus, dialogs) uses KOReader's TextBoxWidget, not crengine.
  Short Thai strings typically fit; a wrapping issue would be fixable with
  a small Lua-side call into libthai via FFI, without touching C.
- No ML segmenters (deepcut / attacut). libthai's dictionary approach is
  proven in Pango, Firefox, and KDE, and costs ~1 MB total vs. 10–50 MB
  for a TF-Lite model.

## Upstream sync

The Thai fork lives on a `thai` branch in each of
`koreader/koreader`, `koreader/koreader-base`, and `koreader/crengine`.
The only merge-hot file is `lvtextfm.cpp`; the hook is behind a
`getDictWordBreakFunc()` null-check, so conflicts are rare and small.
