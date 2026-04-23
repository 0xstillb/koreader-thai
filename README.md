[![KOReader](https://raw.githubusercontent.com/koreader/koreader.github.io/master/koreader-logo.png)](https://koreader.rocks)

#### KOReader is a document viewer primarily aimed at e-ink readers.

> **Thai fork** — this build adds a **neural + dictionary hybrid** Thai
> word-segmenter so Thai text wraps on word boundaries instead of breaking
> mid-word. Primary path is a small CNN ([Deepcut][link-deepcut], ~2 MB,
> ~98% F1); fallback is [libthai][link-libthai]'s dictionary segmenter.
> Android ARM64 only; everything else is unchanged from
> [upstream KOReader][link-koreader-gh].

[![AGPL Licence][badge-license]](COPYING)
[![Latest release][badge-release]][link-gh-releases]

[**Download Android APK**][link-gh-releases] •
[User guide](http://koreader.rocks/user_guide/) •
[Wiki][link-wiki] •
[Upstream KOReader][link-koreader-gh]

## Thai word-segmentation

Thai (like Lao, Khmer, Burmese) does not use spaces between words, so a
reader must find word boundaries itself before deciding where to wrap
lines. Upstream KOReader uses `libunibreak`, which classifies Thai as
class **SA** ("complex, externally resolved") — without an external
resolver it either breaks at *every* character or *never* breaks a Thai
run. Both give poor output.

This fork adds a **word-segmentation post-pass** on top of libunibreak:
after the per-character break loop for a Thai run, the UTF-32 buffer is
handed to a hybrid segmenter that returns word-boundary offsets, and
those offsets are mapped back to `ALLOW_WRAP` flags in crengine's layout
engine.

**Two engines, one hook:**

1. **Primary — [Deepcut][link-deepcut] (CNN).** A small convolutional
   network (~535k params) trained on the BEST2010 corpus, exported to
   ONNX and run on-device via [ONNX Runtime Mobile][link-ort]. Scores
   **~98% F1** on modern Thai text, closing the gap on the four things
   a dictionary-only segmenter misses: **proper names, loanwords, slang
   and compound words**. Model file: `data/thai/deepcut.onnx` (~2 MB).
2. **Fallback — [libthai][link-libthai]'s `th_brk`.** Dictionary-based,
   ~85% F1, tiny (`thbrk.tri`, ~580 KB). Kicks in if the ORT session
   fails to initialise (missing `.so`, missing model file, unsupported
   op) so Thai still word-breaks — just less cleverly.

The integration is **additive** — non-Thai text (English, numbers, CJK,
punctuation) goes through the original libunibreak path unchanged, so
rendering of English-only or mixed-language books is byte-identical to
upstream.

**Post-process rules currently enabled (Deepcut success path only):**
- `แซ่ + (Chinese surname whitelist)` — force split after `แซ่`, suppress
  mid-surname splits, force split after surname.
- `รีบ` — glue known false split (`รี|บ`) back into one word.

**Example 1** — same sentence, same font, same column width:

> ภาษาไทยเป็นภาษาที่ไม่มีช่องว่างระหว่างคำ

| Upstream KOReader | This fork |
|---|---|
| breaks anywhere (mid-word) or one unbroken line | breaks at `ภาษา \| ไทย \| เป็น \| ภาษา \| ที่ \| ไม่ \| มี \| ช่องว่าง \| ระหว่าง \| คำ` |

**Example 2** — a proper name + a loanword, where a dict-only segmenter
typically fails:

> คุณสมชายไปกินข้าวที่ร้านสตาร์บัคส์กับเพื่อนเมื่อวานนี้

| libthai (fallback) | Deepcut (primary) |
|---|---|
| `คุณ \| สม \| ชาย \| ไป \| กิน \| ข้าว \| ที่ \| ร้าน \| สตา \| ร์ \| บัค \| ส์ \| กับ \| เพื่อน \| เมื่อวาน \| นี้` | `คุณ \| สมชาย \| ไป \| กิน \| ข้าว \| ที่ \| ร้าน \| สตาร์บัคส์ \| กับ \| เพื่อน \| เมื่อวานนี้` |
| splits the name "สมชาย" and "Starbucks" into fragments | keeps names and brand loanwords whole |

**Scope:**
- Works for any reflowable format routed through crengine: **EPUB, FB2,
  HTML, MOBI, RTF, TXT, CHM, DOC**.
- Does **not** touch PDF / DjVu (those are fixed-layout; k2pdfopt reflow
  is out of scope for this fork).

**Performance notes (iReader Ocean 5 Pro, Mediatek G99, 4 GB RAM):**
- ORT session init: one-off ~200 ms on first Thai paragraph (lazy).
- Per-page Thai inference: ~10–20 ms for a typical 40-char line.
- First-time open of a 20–30 MB Thai EPUB: **+30–60 s** vs libthai-only
  build (crengine pre-layouts every paragraph at open; ORT runs once per
  Thai run then results are cached in the paginated document). Subsequent
  page turns are unchanged.

## Main features

* **Thai word-segmentation** — hybrid CNN (Deepcut, ~98% F1) primary
  path + libthai dictionary fallback *(new in this fork)*.
* **multi-format documents**: reflowable e-books (EPUB, FB2, Mobi, DOC,
  RTF, HTML, CHM, TXT) and fixed-page formats (PDF, DjVu, CBT, CBZ).
  Scanned PDF/DjVu can be reflowed with the built-in K2pdfopt library.
  [ZIP files][link-wiki-zip] are also supported for some formats.
* **full-featured reading**: highly customizable reader view with
  typesetting options — arbitrary page margins, line spacing, external
  fonts and styles. Multi-lingual hyphenation dictionaries bundled.
* **integrated** with *calibre* (search metadata, receive ebooks
  wirelessly, browse library via OPDS), *Wallabag*, *Wikipedia*,
  *Google Translate*, and other content providers.
* **optimized for e-ink devices**: custom UI without animation, with
  paginated menus, adjustable text contrast, and easy zoom to fit
  content or page in paged media.
* **extensible**: via plugins.
* **fast**: on some older devices, measured to have less than half the
  page-turn delay of the built-in reading software.
* **and much more**: look up words with StarDict dictionaries /
  Wikipedia, add your own online OPDS catalogs and RSS feeds, FTP
  client, SSH server, …

See the [user guide](http://koreader.rocks/user_guide/) and the
[wiki][link-wiki] to discover more features.

## Screenshots

<a href="https://github.com/koreader/koreader-artwork/raw/master/koreader-menu.png"><img src="https://github.com/koreader/koreader-artwork/raw/master/koreader-menu-thumbnail.png" alt="" width="200px"></a>
<a href="https://github.com/koreader/koreader-artwork/raw/master/koreader-footnotes.png"><img src="https://github.com/koreader/koreader-artwork/raw/master/koreader-footnotes-thumbnail.png" alt="" width="200px"></a>
<a href="https://github.com/koreader/koreader-artwork/raw/master/koreader-dictionary.png"><img src="https://github.com/koreader/koreader-artwork/raw/master/koreader-dictionary-thumbnail.png" alt="" width="200px"></a>

## Installation (Android only)

This fork ships only an **Android ARM64** APK. For other platforms
(Kindle, Kobo, PocketBook, Cervantes, reMarkable, Linux), use the
[upstream KOReader][link-koreader-gh].

1. Download the latest APK from the [Releases page][link-gh-releases].
2. On your Android device, enable **Install unknown apps** for your file
   manager (Settings → Apps → *your file manager* → Install unknown apps).
3. Open the downloaded `.apk` file — tap **Install**.
4. No Google Play required. Tested on iReader Ocean 5 Pro (Android 11).

The APK is signed with a personal debug keystore (v1 + v2 + v3 schemes)
— installable by sideload, not publishable to Play Store.

## Recommended settings for Thai reading

The dict-break hook tells crengine *where* it *can* break a Thai line.
You'll want a couple of typesetting knobs tuned so the result looks right
in justified text — otherwise a short line with few word boundaries can
stretch spaces too wide. These are the settings I use:

Open a Thai book, then **Top menu → Typeset (A⁺) → Paragraph**:

| Setting | Value | What it does |
|---|---:|---|
| **Max word expansion** | **20%** | Cap on how much a single space can stretch when justifying. Higher than the default "more" (15%) so short Thai lines don't look cramped, but still below the "obvious gap" threshold. |
| **Word spacing — scaling** | **95%** | Base width of a space, relative to the font's metrics. 95% compensates for libthai's word boundaries sitting tightly next to Thai glyphs. |
| **Word spacing — reduction** | **75%** | Floor (minimum) width a space is allowed to shrink to during justification. 75% prevents adjacent Thai words from visually merging. |

These are *defaults I like*, not hard rules — adjust to taste. KOReader
saves per-book settings under the book's `.sdr/` folder.

### Bundled style tweaks (optional)

The repo ships two example style tweaks under
[`data/thai/styletweaks/`][link-styletweaks] for readers who want a
typographically-tuned body text:

- **`noto_thai_koreader_justify.css`** — Noto Sans Thai body + H1,
  justified with `text-align-last: left` and a small negative
  `word-spacing` to compensate for crengine's tendency to stretch spaces
  on Thai runs.
- **`noto_thai_koreader_sarabun.css`** — same idea, but the body text
  uses **Sarabun** (a popular Thai book font) and only H1 stays on
  Noto Sans Thai Bold.

**On-device behavior (Android ARM64 fork build):**
1. On first app launch, KOReader auto-seeds bundled Thai assets to:
   - `/storage/emulated/0/koreader/styletweaks/`
   - `/storage/emulated/0/koreader/fonts/`
2. Open a book → top menu → **Style tweaks** (pencil icon).
3. Tick the tweak you want. It applies immediately.

No manual font install is required for the bundled Sarabun/Noto Sans Thai/
Maitree set in this fork build.

## Credits

This fork is a personal build of [KOReader][link-koreader-gh] by the
KOReader team and many contributors. **All the heavy lifting — the
reader itself, rendering engines (crengine, MuPDF, DjVuLibre), UI,
plugins — is their work.** The only contribution of this fork is the
additive Thai word-segmentation hook (Deepcut + libthai).

Upstream bugs → report at the [upstream issue tracker][link-koreader-issues].
Fork-specific issues → [this repo's tracker][link-fork-issues].

Third-party components bundled for Thai:

- **Deepcut CNN weights** (`data/thai/deepcut.onnx`, ~2 MB) — converted
  from [Deepcut][link-deepcut]'s `cnn_without_ne_ab.h5` Keras checkpoint
  to ONNX opset 15. MIT-licensed.
- **Thai dictionary data** (`data/thai/thbrk.tri`, ~580 KB) — from the
  Ubuntu `libthai-data` package. [libthai][link-libthai] and
  [libdatrie][link-libdatrie] are LGPL-licensed.
- **[ONNX Runtime Mobile][link-ort] 1.17.3** (`libonnxruntime.so`,
  ~3.7 MB, arm64-v8a) — prebuilt from Microsoft's Maven repo, MIT-
  licensed. Runs the Deepcut CNN on-device.

[badge-license]:https://img.shields.io/github/license/koreader/koreader
[badge-release]:https://img.shields.io/github/release/captainboto/koreader-thai.svg

[link-gh-releases]:https://github.com/captainboto/koreader-thai/releases
[link-fork-issues]:https://github.com/captainboto/koreader-thai/issues
[link-koreader-gh]:https://github.com/koreader/koreader
[link-koreader-issues]:https://github.com/koreader/koreader/issues
[link-libthai]:https://linux.thai.net/projects/libthai
[link-libdatrie]:https://linux.thai.net/projects/datrie
[link-deepcut]:https://github.com/rkcosmos/deepcut
[link-ort]:https://onnxruntime.ai/docs/tutorials/mobile/
[link-styletweaks]:https://github.com/captainboto/koreader-thai/tree/thai/data/thai/styletweaks
[link-wiki]:https://github.com/koreader/koreader/wiki
[link-wiki-zip]:https://github.com/koreader/koreader/wiki/ZIP
