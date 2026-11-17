---
name: dark-deco-video-studio-v3
description: >
  Free-tools-only Dark Deco noir comic video production pipeline for Google Colab (T4 GPU).
  Turns a narration script (plus optional voice-over) into a 9:16 Short or 16:9 Long video
  featuring one locked recurring lead character ("The Father / The Mentor"), sprite+LoRA
  identity lock, Silver-Age comic balloons with face-aware tails, Depth Anything V2 parallax,
  MusicGen/SFX audio mix (-14 LUFS), 3 high-contrast silhouette thumbnails, and upload metadata.
---

# Dark Deco Video Studio v3

## Core Architecture Rules
1. **Free Tools Only**: Runs on free Google Colab T4 (16 GB VRAM, 12 GB RAM). Open-weight models only; zero paid APIs or required keys.
2. **Staged VRAM Lifecycle**: Never hold SDXL + ControlNet + Depth + Whisper simultaneously. Load one stage, flush (`gc.collect()`, `torch.cuda.empty_cache()`), cache artifacts to Drive (`/content/drive/MyDrive/dark_deco_v3`) with `/content` zip fallback.
3. **Self-Contained Notebooks**: `scripts/build_notebooks.py` inlines all `scripts/engine/*.py` modules directly into the generated notebooks (`00_character_studio.ipynb`, `01_episode_builder.ipynb`) so the user never depends on external skill folders at runtime.
4. **Locked Lead Identity**: "The Father / The Mentor" (45yo man, sharp angular jawline, swept-back dark hair with grey temples, thick dark-rimmed glasses, charcoal three-piece suit, trigger `ohwxlead`). Every episode closes with the black wolf silhouette (glowing white eyes) behind him.

---

## Mandatory 6-Step Workflow

### STEP 0: Ask Only What Is Missing
Before generating files, inspect the user's message and ask **only** unanswered items from this list (keep questions brief and structured):
1. **Format**: Short `9:16` (1080x1920) or Long `16:9` (1920x1080)?
2. **Thumbnail Title**: Exact 2–5 word title text (even if wording differs from spoken script, e.g., `"COMPARISON IS KILLER OF JOY"`).
3. **Thumbnail Prop**: What object does the lead's black silhouette hold (e.g., `"silver medal"`)?
4. **Frame Setting**: Default hybrid (`home_study` + episode venue in `mono_noir`), or custom setting?
5. **Character Studio Status**: Is the Character Pack (`series/lead/lora.safetensors` + `sprites/`) already on Google Drive? If not, generate `00_character_studio.ipynb` alongside `01_episode_builder.ipynb`.
6. **Voice-Over & Music**: User-uploaded audio file or free TTS (`kokoro-onnx` / `piper-tts` / `edge-tts`), plus music mood preference?

---

### STEP 1: Script Preprocessing
Run `python3 scripts/preprocess_script.py` on the raw transcript:
- **Strip non-spoken cues** (`[applause]`, `[laughter]`, `[music]`, `(sighs)`) from spoken text and convert them into timed `sfx` cues (`crowd_applause`, `laughter`, etc.).
- **Flag suspicious auto-transcript typos** (e.g., `"Mley"` / `"Mammon"` -> `"Momand"`, `"wells"` -> `"wealth"`, `"medlers"` -> `"medalists"`). Present a table of flagged tokens and ask the user to confirm before altering spoken text.
- Maintain a `spoken_to_display_map` so subtitles and speech bubbles match confirmed spelling while alignment tracks the actual audio tokens.

---

### STEP 2: Series Bible (`series_bible.json`)
Load or initialize from `references/series_bible_template.json`:
- **Lead Character (`lead`)**: Locked appearance tokens, trigger token `ohwxlead`, sprite index (`30 poses x 8 emotions`), voice profile.
- **Recurring Cast**: Episode-specific recurring characters (e.g., `tito` son, `leo` rival) with fixed appearance descriptors.
- **Locations & Palettes**: Frame scenes use `mono_noir`; flashbacks use `teal_noir`, `night_blue`, or `sepia_archive`; thumbnails use `bat_dusk`.
- **Brand Motifs**: Wolf shadow outro (`wolf_silhouette`), lead silhouette thumbnail.

---

### STEP 3: Storyboard Generation (`storyboard.json`)
Follow `references/storyboard_schema_v3.md`, `references/shot_grammar.md`, and `references/prompt_rules.md`:
- **Verbatim Coverage**: Concatenated `narration_segment` across all shots must match the preprocessed spoken script 100% verbatim.
- **High Visual Density**: 1.5–4.0 s per shot for Shorts (at ~2.5 words/sec) plus `inserts` (prop/detail cutaways) on key nouns.
- **Semantic Shot Fields Only**: Specify `shot_type`, `location_id`, `characters` (with `pose`, `emotion`, `screen_pos`), `scene_description` (<= 35 words), `must_show` (>= 1 item), and `must_not_show`. Never free-type character appearances; the engine injects locked tokens from `series_bible.json`.
- **Zero In-Image Text**: Never ask SDXL to draw words or signs. All text is rendered by `engine/comic.py`.
- **IP / Trademark Hygiene**: Never include copyrighted hero names, studios, brands, living celebrities, or sports leagues. Use generic descriptions (`"1992 summer games podium"`, `"1913 newspaper comic strip"`).

---

### STEP 4: Automated Validation
Execute `python3 scripts/validate_storyboard.py storyboard.json series_bible.json script.txt`:
- Verifies 100% verbatim narration coverage and shot duration bounds (1.5–4.0 s Shorts / 3.0–6.0 s Long).
- Checks every character ID against `series_bible.json` and ensures every named person/place/object in narration is represented in `must_show`.
- Enforces IP deny-list, `bubbles[].text` <= 12 words, valid `at_word` tokens, frame vs. flashback character isolation (unless `imagined: true`), and CLIP token budget via chunked Compel encoding.
- Fix all errors before building notebooks.

---

### STEP 5: Build Self-Contained Colab Notebooks & Metadata
Run `python3 scripts/build_notebooks.py --storyboard storyboard.json --bible series_bible.json --output-dir output/`:
- Produces `00_character_studio.ipynb` (if needed), `01_episode_builder.ipynb`, `storyboard.json`, `script.txt`, and `metadata.json` (3 titles, timestamped description, 15 tags, hashtags, pinned comment).
- Run `python3 -m pytest scripts/tests/` to verify AST syntax of every notebook code cell, JSON schema validity, and CPU visual proofs.

---

### STEP 6: Final User Message
Keep your response practical and concise:
1. **Run Guide**: Step-by-step Colab instructions (`Cell 0` setup -> `Cell 1` audio -> `Cell 2` 2-shot self-test -> full pipeline -> contact sheet `REGEN` -> download zip).
2. **T4 Runtime Estimates**: Character Studio (~75–105 min once per channel); Episode Builder (~18–28 min for a 2.5-min Short on T4, 0 min for cached stages on resume).
3. **Verification Honesty**: State clearly what passed on CPU (validator, alignment, comic renderer, face-aware tails, layout collision, notebook AST) vs. what requires the Colab T4 GPU (SDXL, ControlNet, Depth Anything V2, Whisper, MusicGen).
4. **Editorial Fact-Check Note**: For historical, psychological, health, or financial claims in the script, append a brief note flagging contested details (e.g., Arthur R. Momand spelled "Momand" vs transcript "Mley/Mammon") and offer 1–2 safer wordings.
