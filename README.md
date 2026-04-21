[![KOReader](https://raw.githubusercontent.com/koreader/koreader.github.io/master/koreader-logo.png)](https://koreader.rocks)

#### KOReader is a document viewer primarily aimed at e-ink readers.

> **Thai fork** — this build adds dictionary-based Thai word-segmentation
> (via [libthai][link-libthai]) so Thai text wraps on word boundaries
> instead of breaking mid-word. Android ARM64 only; everything else is
> unchanged from [upstream KOReader][link-koreader-gh].

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

This fork adds a **dictionary-based post-pass** on top of libunibreak:
after the per-character break loop for a Thai run, the UTF-32 buffer is
handed to [`libthai`][link-libthai]'s `th_brk`, which returns word-
boundary offsets from the bundled Thai dictionary (`thbrk.tri`, ~580 KB).
Those offsets are mapped back to `ALLOW_WRAP` flags in crengine's
layout engine.

The integration is **additive** — non-Thai text (English, numbers, CJK,
punctuation) goes through the original libunibreak path unchanged, so
rendering of English-only or mixed-language books is byte-identical to
upstream.

**Example** — the same sentence, same font, same column width:

> ภาษาไทยเป็นภาษาที่ไม่มีช่องว่างระหว่างคำ

| Upstream KOReader | This fork |
|---|---|
| breaks anywhere (mid-word) or one unbroken line | breaks at `ภาษา \| ไทย \| เป็น \| ภาษา \| ที่ \| ไม่ \| มี \| ช่องว่าง \| ระหว่าง \| คำ` |

**Scope:**
- Works for any reflowable format routed through crengine: **EPUB, FB2,
  HTML, MOBI, RTF, TXT, CHM, DOC**.
- Does **not** touch PDF / DjVu (those are fixed-layout; k2pdfopt reflow
  is out of scope for this fork).

## Main features

* **Thai word-segmentation** — dictionary-based, via libthai *(new in
  this fork)*.
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

## Credits

This fork is a personal build of [KOReader][link-koreader-gh] by the
KOReader team and many contributors. **All the heavy lifting — the
reader itself, rendering engines (crengine, MuPDF, DjVuLibre), UI,
plugins — is their work.** The only contribution of this fork is the
additive libthai integration for Thai word-segmentation.

Upstream bugs → report at the [upstream issue tracker][link-koreader-issues].
Fork-specific issues → [this repo's tracker][link-fork-issues].

Thai dictionary data (`thbrk.tri`) comes from the Ubuntu `libthai-data`
package; [libthai][link-libthai] and [libdatrie][link-libdatrie] are
LGPL-licensed.

[badge-license]:https://img.shields.io/github/license/koreader/koreader
[badge-release]:https://img.shields.io/github/release/captainboto/koreader-thai.svg

[link-gh-releases]:https://github.com/captainboto/koreader-thai/releases
[link-fork-issues]:https://github.com/captainboto/koreader-thai/issues
[link-koreader-gh]:https://github.com/koreader/koreader
[link-koreader-issues]:https://github.com/koreader/koreader/issues
[link-libthai]:https://linux.thai.net/projects/libthai
[link-libdatrie]:https://linux.thai.net/projects/datrie
[link-wiki]:https://github.com/koreader/koreader/wiki
[link-wiki-zip]:https://github.com/koreader/koreader/wiki/ZIP
