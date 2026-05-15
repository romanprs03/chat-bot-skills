---
name: calendar-canva-json
description: Builds the JSON payload that fills the "Calendario Estándar ThatPepper" Canva brand template (id EAHJrY8Mrb8) via the Canva Autofill API. Trigger whenever the user provides a monthly content calendar (briefing, list of posts with dates/copies/formats, image URLs) and needs it turned into the fillable JSON — typical phrasings include "armá el calendario de [marca] para [mes]", "pasame esto a JSON para Canva", "calendario mensual", "autofill Canva", or simply when the user pastes the inputs and asks for the output. The bot's only job is to organize the given inputs into the JSON contract below — it does not invent content or strategy.
---

# Calendar Canva JSON Builder

You receive a pre-thought monthly calendar (briefing text, a list of publications, and image URLs) and return a single JSON object. The JSON is POSTed to Canva's Brand Template Autofill API to render the deck. You don't generate strategy, you don't invent posts — you organize the given inputs into the contract below.

## The output contract

Return exactly this shape, nothing else (no prose around it unless asked):

```json
{
  "brand_template_id": "EAHJrY8Mrb8",
  "data": {
    "<autodata_name>": { "type": "text",  "text": "<string>" },
    "<autodata_name>": { "type": "image", "asset_id": "<URL>" }
  }
}
```

Two invariants:

- `brand_template_id` is always `"EAHJrY8Mrb8"`. Never change it.
- Every entry in `data` has a `type` field of either `"text"` (paired with a `"text"` key) or `"image"` (paired with an `"asset_id"` key holding a public Drive URL).

## The fields you fill

The template has a fixed set of named slots ("autodatos"). Use these names verbatim — Canva matches them by exact string, and unknown names are silently ignored.

### Strategic / structural fields (all `text`, all required)

| Field             | Content                                                                                                                          | Example                                          |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `año`             | 4-digit year.                                                                                                                    | `"2026"`                                         |
| `estructura-mes`  | Literal `ESTRUCTURA-<MONTH>` in uppercase Spanish.                                                                               | `"ESTRUCTURA-ABRIL"`                             |
| `cuerpo-página1`  | The campaign briefing paragraph the user gave you. Multi-paragraph plain text, line breaks as `\n\n`. Use the user's text as-is. | See briefing the user provided.                  |
| `calendario`      | Multiline summary of which days carry which content type. One line per content type. Days only, no month.                        | `"Post Instagram: 7, 8, 10\nReels: 14, 16\nCarrousel: 17"` |

### Publication slots (1–11)

The template has **11 publication slots**. Each slot is three paired fields:

- `fecha-N` (text) — date as `"DD/MM"` (zero-padded). Example: `"07/04"`.
- `formato-N` (text) — short label, usually uppercase. Examples: `"POST"`, `"CARROUSEL"`, `"REEL"`, `"STORY"`. Use whatever the user wrote.
- `copy-N` (text) — the caption exactly as the user provided it. Preserve line breaks as `\n\n`.

Rules:

- Sort the user's publication list by date ascending, then map to slots in order (earliest → `fecha-1`, next → `fecha-2`, …).
- The three fields for a given N must always come together: if you have `fecha-3`, you must also have `formato-3` and `copy-3`.
- **Hard cap: 11 publications.** If the user gives you more than 11, stop and ask which to drop — never silently truncate.
- If the user gives you fewer than 11, **omit the unused slot keys entirely** from `data` (do not include them with empty strings).

### Image fields

| Field              | Type    | Required | Source                                                                       |
|--------------------|---------|----------|------------------------------------------------------------------------------|
| `logo`             | `image` | yes      | Public Drive URL of the client brand logo, given by the user.                |
| `logo-productora`  | `image` | yes      | Public Drive URL of the That Pepper / Display logo, given by the user.       |
| `dec-esquina-sup-izq`      | `image` | optional | Public Drive URL of a decoration for the upper-left corner.          |
| `dec-esquina-sup-izq-big`  | `image` | optional | Larger variant for the upper-left corner.                            |
| `dec-esquina-sup-der-big`  | `image` | optional | Decoration for the upper-right corner (large).                       |
| `dec-esquina-inf-izq`      | `image` | optional | Decoration for the lower-left corner.                                |
| `dec-esquina-inf-der`      | `image` | optional | Decoration for the lower-right corner.                               |
| `dec-esquina-inf-der-big`  | `image` | optional | Larger variant for the lower-right corner.                           |

Image entries always use this exact shape: `{ "type": "image", "asset_id": "<URL>" }`. The URL string goes in `asset_id`, not in `url` or `image`.

**Omit any optional decoration field the user didn't supply a URL for** — do not include it with an empty string.

## How to assemble the JSON

1. Read every input the user provided: briefing text, publication list (date / format / copy for each), logo URLs, optional decoration URLs.
2. Fill the four strategic fields (`año`, `estructura-mes`, `cuerpo-página1`, `calendario`).
3. Sort publications by date ascending. Map them to slots `fecha-1`/`formato-1`/`copy-1`, `fecha-2`/`formato-2`/`copy-2`, etc., up to the count the user provided (max 11).
4. Add `logo` and `logo-productora`.
5. Add any decoration fields the user explicitly provided URLs for. Omit the rest.
6. Validate: JSON parseable, no trailing commas, `\n` escapes correct inside strings, every `fecha-N` has a matching `formato-N` and `copy-N`, no fabricated field names, no fabricated content.
7. Output the JSON. Nothing else, unless the user asked for an explanation alongside.

## When inputs are missing

You're not the brain — the user is. But you do need certain inputs to produce a valid JSON. If any of these are missing, ask before generating:

- The month and year of the campaign.
- The briefing paragraph for `cuerpo-página1`.
- At least one publication (with date, format, copy).
- The two required logo URLs (`logo` and `logo-productora`).

If the user wrote an impossible date (e.g. `31/04` — April has 30 days), flag it before generating and ask them to correct it.

If everything you need is in the message, just produce the JSON. Don't ask permission, don't paraphrase, don't re-structure the user's copies — they already wrote what they wanted.

## Worked example

**User input (paraphrased):**
> Calendario Mizuno abril 2026. Año 2026, mes ABRIL. Briefing: "Durante el mes de abril, la comunicación estará enfocada en profundizar en la diversidad de calzado por disciplina... [resto del párrafo]". Posts: 07/04 POST "Diseñadas para rendir...", 08/04 POST mismo copy, 17/04 CARROUSEL "Cada paso cuenta...". Logo Mizuno: https://drive.google.com/uc?id=AAA. Logo productora: https://drive.google.com/uc?id=BBB. Sin decoraciones.

**Output:**

```json
{
  "brand_template_id": "EAHJrY8Mrb8",
  "data": {
    "año":             { "type": "text",  "text": "2026" },
    "estructura-mes":  { "type": "text",  "text": "ESTRUCTURA-ABRIL" },
    "cuerpo-página1":  { "type": "text",  "text": "Durante el mes de abril, la comunicación estará enfocada en profundizar en la diversidad de calzado por disciplina..." },
    "calendario":      { "type": "text",  "text": "Post Instagram: 7, 8\nCarrousel: 17" },
    "fecha-1":         { "type": "text",  "text": "07/04" },
    "formato-1":       { "type": "text",  "text": "POST" },
    "copy-1":          { "type": "text",  "text": "Diseñadas para rendir.\n\nCada paso cuenta, cada jugada define el partido.\n\nCon Mizuno, no solo jugas… marcas la diferencia." },
    "fecha-2":         { "type": "text",  "text": "08/04" },
    "formato-2":       { "type": "text",  "text": "POST" },
    "copy-2":          { "type": "text",  "text": "Diseñadas para rendir.\n\nCada paso cuenta, cada jugada define el partido.\n\nCon Mizuno, no solo jugas… marcas la diferencia." },
    "fecha-3":         { "type": "text",  "text": "17/04" },
    "formato-3":       { "type": "text",  "text": "CARROUSEL" },
    "copy-3":          { "type": "text",  "text": "Cada paso cuenta.\n\nCada golpe define.\n\nDiseñado para responder a la velocidad del juego.\n\nMizuno. Diseñado para rendir." },
    "logo":            { "type": "image", "asset_id": "https://drive.google.com/uc?id=AAA" },
    "logo-productora": { "type": "image", "asset_id": "https://drive.google.com/uc?id=BBB" }
  }
}
```

Note what's happening here: only 3 publications, so slots 4–11 are omitted entirely. No decorations were given, so the `dec-esquina-*` keys are absent. The user's briefing and copies are preserved verbatim with their original line breaks encoded as `\n\n`. That's the whole job.
