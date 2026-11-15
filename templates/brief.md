---
brief_id: {{slug}}
generated: {{iso_date}}
status: draft            # draft | approved | built | posted | scored
pattern: {{pattern_name}}
target_platform: [reels, tiktok, shorts]
---

# {{slug}}

## 1 · SOURCE — the exact piece

| | |
|---|---|
| **Direct link** | {{permalink}} |
| Creator | {{creator}} ({{handle}}) |
| Posted | {{posted_date}} |
| Views | {{views}} |
| **Outlier multiple** | **{{outlier_multiple}}×** their 30-post median |
| Reach ratio | {{reach_ratio}} views per follower |
| Duration | **{{duration}}** ({{duration_sec}}s) |
| Hook WPM | {{hook_wpm}} |
| Body WPM | {{body_wpm}} |
| Local copy | `data/forensics/{{video_id}}/` |

> Direct per-item link, always. Never a profile or search page.

## 2 · TITLE / CAPTION — copy verbatim

```
{{source_title}}
```

**Yours (proper nouns + domain swapped only):**
```
{{remix_title}}
```
Changed: {{title_diff}}

## 3 · HOOK — copy verbatim, 0:00–{{hook_end}}

**Theirs, word for word:**
```
{{source_hook_verbatim}}
```

**Yours:**
| Timecode | Line | Delivery |
|---|---|---|
{{#hook_lines}}| {{tc}} | {{line}} | {{delivery_note}} |
{{/hook_lines}}

Words in hook: {{hook_word_count}} · Target: {{hook_word_target}} (source WPM held)

## 4 · THUMBNAIL / COVER FRAME — build sheet

Source cover: `data/forensics/{{video_id}}/cover.jpg`

### Measured geometry — reproduce at these ratios
| Element | Measurement | At 1080×1920 | At 1280×720 |
|---|---|---|---|
| Text block top edge | {{text_top_pct}}% of height | {{text_top_1080}} px | {{text_top_720}} px |
| Text block left edge | {{text_left_pct}}% of width | {{text_left_1080}} px | {{text_left_720}} px |
| **Cap height** | **{{cap_height_pct}}% of frame height** | **{{font_px_1080}} px** | **{{font_px_720}} px** |
| Line count | {{line_count}} | | |
| Words on frame | {{word_count}} | | |
| Face bbox height | {{face_h_pct}}% of frame height | {{face_h_1080}} px | {{face_h_720}} px |
| Face center | x {{face_x_pct}}%, y {{face_y_pct}}% | | |
| Thirds cell | {{thirds_cell}} | | |

### Type
- Class: **{{font_class}}** · Weight: **{{font_weight}}** · Case: {{font_case}}
- Tracking: {{tracking}} · Treatment: {{text_treatment}}
- Substitutes: {{font_substitutes}}

### Color — sampled hex, with coverage
| Role | Hex | Coverage |
|---|---|---|
| Background dominant | `{{bg_hex}}` | {{bg_pct}}% |
| Text fill | `{{text_hex}}` | — |
| Accent / highlight word | `{{accent_hex}}` | {{accent_pct}}% |
| Outline / shadow | `{{outline_hex}}` | — |

Text-on-background contrast: **{{contrast_ratio}}:1** — hold this or beat it.

### Yours
- Same coordinates, same ratios, same palette
- Her face replaces theirs at the identical bbox and scale
- Copy on frame: `{{remix_thumb_text}}`
- Build: {{thumb_build_route}}

## 5 · STRUCTURE — the cut map

Total: **{{duration_sec}}s.** Yours lands within ±2s or it gets rebuilt.

| Time | Beat | On screen | Their content | Yours |
|---|---|---|---|---|
{{#beats}}| {{tc}} | {{beat_type}} | {{visual}} | {{their_line}} | {{your_line}} |
{{/beats}}

Hard cuts at: {{cut_timecodes}}
Pattern interrupts: {{interrupt_inventory}}
CTA lands at: {{cta_tc}} — {{cta_type}}

## 6 · FULL SCRIPT — timed

Target: **{{target_word_count}} words** = {{duration_sec}}s × {{body_wpm}} WPM.
Off by more than 5% and the pacing breaks.

{{#script_lines}}
**{{tc}}** — {{line}}
{{/script_lines}}

Actual: {{actual_word_count}} words · Delta: {{word_delta}}

## 7 · HEYGEN BUILD

```json
{
  "avatar_id": "{{avatar_id}}",
  "voice_id": "{{voice_id}}",
  "speed": {{voice_speed}},
  "dimension": {"width": 1080, "height": 1920},
  "script": "{{heygen_script}}"
}
```
Pauses at: {{pause_markers}} · Est. cost: ${{heygen_cost}} · Est. render: {{render_eta}}

## 8 · B-ROLL / SCREEN SHOTS
{{#broll}}
- **{{tc}}** — {{description}} · source: {{source_type}}
{{/broll}}

## 9 · THE THREE DELTAS — what makes it hers
1. **{{delta_1}}**
2. **{{delta_2}}**
3. **{{delta_3}}**

## 10 · VOICE CHECK
- [ ] Zero banned words (journey, amazing, tribe, hack, transformation, gentle reminder, in essence, at the end of the day, real talk)
- [ ] Opens already in motion
- [ ] Specific beats general — named tools, real numbers
- [ ] No hedged opinions
- [ ] Dry, not jokey
- [ ] Never performs vulnerability

## 11 · AFTER 72H
| | Source | Mine |
|---|---|---|
| Views | {{views}} | |
| Outlier multiple | {{outlier_multiple}}× | |
| Follows | — | |
Verdict: ☐ run it again  ☐ adjust  ☐ retire pattern
