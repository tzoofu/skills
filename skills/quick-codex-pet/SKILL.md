---
name: quick-codex-pet
description: Create a Codex custom pet package quickly from a person or character reference image. Use when asked to make, repair, or package a Codex pet with pet.json and spritesheet.webp, especially when the user wants a clean 8x9 animated atlas, transparent background, and fast iteration from a reference photo.
---

# Quick Codex Pet

## Goal

Create a Codex custom pet package:

```text
<PetName>/
├── pet.json
└── spritesheet.webp
```

The atlas must be `1536x1872`, arranged as `8` columns by `9` rows, with `192x208` cells and transparent unused cells.

## Fast Workflow

1. Identify the subject from the reference image.
   - Write down only the stable visual traits that must survive at pet size.
   - For a person, capture hair shape, eyes, smile/expression, clothing colors, signature accessories, and overall vibe.

2. Generate a neutral canonical base sprite.
   - Use `imagegen`.
   - Use a flat chroma background such as `#FF00FF`.
   - Make the normal pose neutral: arms down, eyes open, relaxed stance.
   - Do not put waving, pointing, saluting, closed eyes, or dramatic expressions in the base.

3. If a wave is needed, generate or preserve a separate waving source.
   - The raised hand belongs only in the `waving` row.
   - Do not use a raised-hand image as the base for idle/running/review states.

4. Remove the chroma key locally.
   - Use the installed helper:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/imagegen/scripts/remove_chroma_key.py" \
  --input work/<pet>-base-chroma.png \
  --out work/<pet>-base-cutout.png \
  --auto-key border \
  --soft-matte \
  --transparent-threshold 12 \
  --opaque-threshold 220 \
  --despill
```

5. Build the atlas deterministically with Pillow.
   - Use the neutral base for every row except `waving`.
   - Use the waving source only for row 3.
   - Keep eyes open unless the generated source itself has a clean blink. Avoid drawing scripted closed-eye overlays.
   - Use subtle body transforms for animation: small shifts, rotations, squash/stretch, and flips.

6. Validate the atlas.

```bash
python3 /Users/tzoof/.codex/vendor_imports/skills/skills/.curated/hatch-pet/scripts/validate_atlas.py \
  outputs/<PetName>/spritesheet.webp \
  --json-out work/qa/<pet>-validation.json
```

7. Inspect a contact sheet.
   - Confirm the idle row is not waving.
   - Confirm closed-eye frames do not look weird.
   - Confirm the wave appears only in row 3.
   - Confirm unused cells are blank and the sprite is not clipped.

8. Package the files.

```json
{
  "id": "pet-id",
  "displayName": "Pet Name",
  "description": "One short sentence.",
  "spritesheetPath": "spritesheet.webp"
}
```

## Row Contract

| Row | State | Used columns |
| --- | --- | ---: |
| 0 | idle | 6 |
| 1 | running-right | 8 |
| 2 | running-left | 8 |
| 3 | waving | 4 |
| 4 | jumping | 5 |
| 5 | failed | 8 |
| 6 | waiting | 6 |
| 7 | running | 6 |
| 8 | review | 6 |

Unused columns after the used columns must be fully transparent.

## Base Prompt Template

```text
Create a compact full-body Codex pet mascot based on the person in the reference image.

Preserve these traits: <hair, eyes, smile, accessories, clothing, colors>.
Style: polished sticker-like chibi game sprite, full body, readable at 192x208, crisp edges.
Pose: neutral relaxed standing pose, both arms relaxed down by their sides or gently resting near the clothing. Hands must not be raised. The character must not be waving, saluting, pointing, reaching, or holding any lifted-hand pose. Eyes open.
Background: perfectly flat solid #FF00FF chroma-key background, uniform only, no shadows, no gradients, no floor plane, no texture, no reflection.
Constraints: no text, no logo, no scenery, no watermark. Do not use #FF00FF in the subject.
```

## Waving Prompt Template

```text
Create the same Codex pet character as a compact full-body chibi sprite.

Preserve the same face, hair, accessories, clothing, palette, and proportions.
Pose: friendly waving pose with one hand raised. This pose is for the waving animation only.
Background: perfectly flat solid #FF00FF chroma-key background, uniform only.
Constraints: no text, no logo, no scenery, no watermark, no wave marks or floating effects.
```

## Faster Implementation Notes

- Reuse the current project script as the starter: `work/build_mypet.py`.
- Change the path constants, pet name, display name, and description.
- Keep `BASE_PATH` as the neutral cutout.
- Keep `WAVE_BASE_PATH` as the waving cutout.
- If the pet should never wave, remove the separate wave source and use the neutral base everywhere.
- For left/right movement, mirror `running-right` transforms for `running-left`.
- Do not manually paint closed eyes unless the user explicitly asks for blinking and it has been visually checked.

## QA Traps From This Run

- Do not prompt the base with "one hand raised"; it makes idle look permanently mid-wave.
- Do not use a waving pose as the canonical base.
- Do not add fake blink overlays on tiny faces; they can look uncanny.
- Do not let failed/review/waiting overlays cover the eyes unless the user likes that style.
- Do not accept a WebP unless `validate_atlas.py` reports `ok: true`.
- Do not forget to update the zip after rebuilding the package.

## Final Deliverables

Save final user-facing files under `outputs/`:

```text
outputs/<PetName>/pet.json
outputs/<PetName>/spritesheet.webp
outputs/<PetName>.zip
```

For Codex app installation, place the whole folder under:

```text
${CODEX_HOME:-$HOME/.codex}/pets/<PetName>/
```
