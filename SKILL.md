---
name: anime-reel
description: Make an anime-style episode (original characters drawn in code, lip-synced to voice MP3s, like the Zero Se Freelance series) from a script plus one voice MP3 per line. Use whenever the user runs /anime-reel or asks for an anime reel, anime episode, anime-style cartoon, or lip-synced anime characters from a script and voice lines, in 9:16 for reels and 16:9 for YouTube.
---

# anime-reel

Input: a script + one voice MP3 per line (user attaches both). Output: a 9:16 reel and a 16:9 copy, each with the original audio, plus a 720p copy under 20 MB for WhatsApp.

## Hard rules (every run)
- Draw everything in code: HTML canvas, rendered frame by frame in headless Chrome, encoded with ffmpeg. No stock footage, no image generators. Use `engine/` as the base; extend it, don't replace it, so the look stays the same every time.
- Time every word with forced alignment (torchaudio MMS_FA via `scripts/align.py`). Never guess timings. If alignment fails for a line, stop and tell the user why; do not fall back to even spacing.
- Show a contact sheet of stills before the full render and wait for approval.
- Ask before overwriting any existing file. `encode.sh` refuses to overwrite and exits 2; relay that to the user.
- Keep the original audio untouched in the final MP4 (voice lines are only placed on a timeline, never re-processed). No background music unless the user asks.

## Method
1. **Setup.** Run `bash scripts/check_setup.sh` (ffmpeg, Node, puppeteer, torch/torchaudio, uroman). Fix anything missing before continuing. Run `bash scripts/sfx.sh <project>/sfx` to synthesise hit/whoosh/impact/pop sounds with ffmpeg.
2. **Split the script into shots.** One shot per voice line. For each, write: `id`, `speaker`, `text` (exactly what the MP3 says), `audio` path, `place` (rooftop, office, street, bedroom, sky, or add a new `drawPlace` branch), `camera` (`static|push|pull|pan` + `amount`), `cast`, `slots` (left/center/right), and `fx`. Save as `shots.json` (schema in `scripts/build_timeline.py`). Show the shot list and let the user correct it if it is ambiguous.
3. **Characters.** Define each in `characters` (name, skin, hair, hairStyle spiky|bob|long|short, eye, outfit, accent, `color` = that character's caption colour). They must be original, never copied from existing anime. Anime look: big eyes with highlight, thick clean outlines, flat colour plus exactly one shade. Render the character sheet and keep the JSON as the single source of truth so characters never change between shots:
   `node scripts/render_frames.mjs sheet timeline.json landscape out/character_sheet.png`
4. **Align and lip-sync.** `python3 scripts/build_timeline.py shots.json timeline.json work` runs MMS_FA per line (word timings) and builds each character's mouth-open curve from the loudness of that voice line (`mouth.py`). Blinks and idle motion are built into the engine. For non-English lines, `align.py` romanises with uroman while keeping the original spelling on screen.
5. **Anime tricks on key lines** (add to `fx`, times in seconds from shot start): `speedlines` on reveals and determination, `impact` (white/black flash frames, with `sfx: "impact"`) on the biggest line, `shake` on hits, `chibi` (+ `who`, `sfx: "pop"`) on comic reactions. Use sparingly: roughly one trick per 2-3 shots, so they still land.
6. **Captions and sound.** 2-3 word captions are already handled by the engine, with the spoken word in the speaker's colour and the rest white. Sound effects sit on the same cues as the visual hits (via `fx[].sfx`). Mix with `python3 scripts/build_audio.py timeline.json work/audio.wav`.
7. **Contact sheet, then approval.** Pick ~12 times spread across the episode, including every trick:
   `node scripts/render_frames.mjs stills timeline.json portrait work/stills 0.5,2.0,...`
   `bash scripts/contact_sheet.sh work/stills out/contact_sheet.png`
   Show `out/contact_sheet.png` and stop. Do not render the full video until the user approves.
8. **Render** after approval, for each orientation (`portrait` = 1080x1920, `landscape` = 1920x1080, 30 fps):
   `node scripts/render_frames.mjs video timeline.json portrait work/reel_silent.mp4`
   `bash scripts/encode.sh work/reel_silent.mp4 work/audio.wav out/<name>_9x16`
   Repeat for landscape with `out/<name>_16x9`. `encode.sh` muxes AAC 256k audio and writes the `_720p.mp4` copy (target ~17 MB, always under 20 MB). Check sizes with `ls -lh` and report them.

## Render settings
- 9:16: 1080x1920, 30 fps, H.264 yuv420p, CRF 16, AAC 256k, faststart. 16:9: 1920x1080, same settings.
- WhatsApp copy: 720p (720x1280 or 1280x720), bitrate computed from duration to stay under 20 MB.
- Captions sit at ~68% height in 9:16 (clear of the top 250 px and bottom 420 px app UI) and ~86% in 16:9.

## Files in this skill
- `engine/engine.js`, `engine/index.html`: canvas engine (characters, places, speed lines, impact frames, shake, chibi, captions, badge).
- `scripts/`: `check_setup.sh`, `sfx.sh`, `align.py`, `mouth.py`, `build_timeline.py`, `build_audio.py`, `render_frames.mjs`, `contact_sheet.sh`, `encode.sh`.
