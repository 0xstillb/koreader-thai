# Handoff (Short Session)

## Pushed Status (2026-04-23)
- `koreader-thai` `thai`: `3e6d4bf7d`
- `koreader-base` `thai`: `424f1425`
- `crengine` `thai`: `1f66d30e`

## What Landed
- Deepcut remains primary, libthai remains fallback.
- Fixed false split case: `รี|บ` -> `รีบ` (postprocess).
- Chinese surname postprocess still active (`แซ่ + whitelist`).
- `DEEPCUT_MODEL_PATH` now resolved from absolute module path (less cwd risk).
- Android first-run seeding for Thai fonts + style tweaks is already in this branch.
- README updated for latest runtime behavior.

## Next Session (Token-light)
1. Device verify only (no code first):
   - Test words: `รีบ`, `แซ่หรวน`, `แซ่เหลียง`, plus 20 real-world lines.
   - Confirm no regression in normal Thai text.
2. If more bad splits found:
   - Add minimal whitelist entries in `thaibreak.cpp` (`THAI_GLUED_WORDS`).
   - Rebuild APK and re-test.
3. Keep 3 backlog topics separate (do not mix in same session):
   - EPUB open performance
   - Bluetooth reconnect after sleep
   - Frontlight control (iReader Ocean 5 Pro)

## Model Recommendation
- Main implementation/debug: **`gpt-5.4`**
- Fast triage/log summarization: **`gpt-5.4-mini`**

