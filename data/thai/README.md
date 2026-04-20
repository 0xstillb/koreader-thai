# Thai word-break dictionary (`thbrk.tri`)

This directory holds the **libthai dictionary** that crengine consults via
`thai_dict_break()` (see
`base/thirdparty/kpvcrlib/crengine/crengine/src/thaibreak.cpp`) to segment Thai
text at word boundaries during line breaking.

Thai has no inter-word spaces, and UAX #14 defers Thai to an external resolver.
Without this dictionary, Thai paragraphs either render as one unbroken run (ugly
off-screen overflow) or break at every character (ugly stairs).

## Provisioning the dictionary

The dictionary is **not committed to this repo** — it ships as part of the
upstream libthai tarball (LGPL). During `kodev fetch-thirdparty`,
`base/thirdparty/libthai/` downloads and builds libthai; the extracted source
tree contains `data/thbrk.tri`. The project build copies it here.

If you are bootstrapping by hand, download libthai 0.1.29 from
https://linux.thai.net/pub/ThaiLinux/software/libthai/ , extract it, and drop
`data/thbrk.tri` into this directory so the final path is:

```
koreader/data/thai/thbrk.tri
```

At runtime, `frontend/document/credocument.lua` sets the environment variable
`LIBTHAI_DICTDIR=./data/thai` before the first crengine document loads, so
libthai finds the dictionary by the name `thbrk.tri` inside this dir.

## Verifying it works

Open any EPUB containing Thai text. Lines must wrap at word boundaries, not
mid-word. A good canary string:

> ภาษาไทยเป็นภาษาที่ไม่มีช่องว่างระหว่างคำ

Expected wrap points: `ภาษา|ไทย|เป็น|ภาษา|ที่|ไม่|มี|ช่อง|ว่าง|ระหว่าง|คำ`.
