# COPYCAT ENGINE — System Spec

A pipeline that finds proven short-form video, measures it to the pixel and
the tenth of a second, and emits a build sheet Brie can execute with a HeyGen
avatar without ever standing in front of a camera.

---

## 0. The one correction to Sabrina's rule

Sabrina says: *copy the 1M-view video, not the 10K-view video.*

She's approximating the right thing with a blunt instrument. Raw views measure
**channel size**, not **format strength**. A 1M-view video from a 2M-follower
account is a *below-average* post. A 200K-view video from a 5K account is a
format that broke containment — and that is the one worth cloning, because the
format is doing the work, not the audience.

So the engine ranks on three numbers, not one:

```
outlier_multiple = views / median(views of that creator's last 30 posts)
reach_ratio      = views / follower_count
absolute_views   = views
```

A video qualifies when `outlier_multiple >= 3.0` **and** `absolute_views >= 100,000`.
Then it sorts by `outlier_multiple`, not by views.

This is not deviating from the method. It is the method, with the noise removed.
Everything downstream — copy the container exactly, replace the filling — is
Sabrina's instruction followed to the letter.

---

## 1. What gets copied, and what does not

This is the hard line the whole system runs on. "Exact and perfect, no artistic
liberty" applies to the **container**. The container is format, and format is
not owned by anyone.

### COPY EXACTLY — the container
- Hook wording, verbatim, word for word, for the first 8–12 seconds
- Title / caption first line, verbatim
- Total duration, to within ±2 seconds (if the source is 0:43, yours is 0:43)
- Words per minute, to within ±5%
- Cut rhythm — where every hard cut lands, in seconds
- Thumbnail **geometry**: text block position and size as a % of frame, face
  position and size as a % of frame, number of words, line breaks
- Color palette, sampled as hex
- Font *class* and weight (condensed grotesque / geometric sans / heavy slab),
  plus the size as a % of frame height
- Caption style: burned-in vs native, position, size, highlight color
- Where the CTA lands, in seconds
- The *shape* of the payload: "7 tools" stays 7 items, not 5

### REPLACE ENTIRELY — the filling
- Their face → her HeyGen avatar, at the same coordinates and the same scale
- Their photography, screenshots, b-roll → hers
- Their logo, wordmark, watermark → hers, same corner, same size
- Everything said after the hook — the actual list, the actual tools, the
  actual commentary, the actual examples
- Their niche framing → her angle

The reason this is not a compromise: what makes a thumbnail perform is the
contrast ratio, the text-to-frame ratio and the face scale. Not whose face it
is. Her face at their coordinates beats their face on her channel, every time,
because the audience has to recognize *her* for the follow to happen.

The engine enforces this split mechanically — the REMIX agent is handed the
container as structured JSON and never sees their video file at all.

---

## 2. Agents

Six agents. Each one writes a file and exits. No agent holds state in context.

### `scout` — find the outliers
- Input: `targets.yaml`
- Pulls last 30 posts per creator per platform
- Computes the three scores in §0
- Writes `data/sweeps/YYYY-MM-DD.json`
- Writes `data/candidates/<video_id>.json` for anything that qualifies
- **Hard cap: 6 candidates per sweep.** More than six and she ships zero.

Sources:
- YouTube → Data API v3 via `playlistItems.list` on the uploads playlist
  (1 quota unit) then `videos.list` batched 50-at-a-time (1 unit).
  **Never `search.list`** — it costs 100 units and will burn the 10,000/day cap
  in 100 calls. yt-dlp `--dump-json --skip-download` as the fallback and for
  thumbnails.
- Instagram → Apify `apify/instagram-reel-scraper`, by username. Returns views,
  likes, comments, duration, caption, displayUrl, videoUrl, transcript.
- TikTok → Apify `clockworks/tiktok-scraper`, by profile. Returns plays, duration,
  cover image, subtitles.

### `forensics` — measure it
The agent that makes this whole thing different from a swipe file. For each candidate:

**Frames**
- `ffmpeg` extract at t = 0.0, 0.5, 1.0, 1.5, 2.0, 3.0s, plus the cover frame
- Scene-change detection (`ffmpeg -vf "select='gt(scene,0.3)'"`) → the cut map,
  every hard cut in seconds

**Thumbnail / cover forensics** — two passes, both required:
1. *Measurement pass* (OpenCV + PaddleOCR): text bounding boxes as % of frame
   height and width, dominant hex colors via k-means (k=5) with % coverage,
   face bbox via Haar/YuNet as % of frame, WCAG contrast ratio of text against
   the pixels directly behind it, rule-of-thirds cell the face occupies.
2. *Judgment pass* (Claude vision on the frame): typeface class, stroke weight,
   outline/shadow treatment, emotional register of the expression, what the eye
   hits first, second, third. Vision gets the numbers from pass 1 in its prompt
   so it describes rather than guesses.

**Audio / script forensics**
- Transcript with word-level timestamps (platform subtitles where available,
  `whisper` locally where not)
- WPM overall, and WPM for the first 10s separately — the hook is almost always
  faster than the body, and that delta is part of the container
- Hook boundary: the timestamp where the promise stops and the payload starts
- Beat map: every sentence with its start timecode
- Pattern-interrupt inventory: every zoom, text pop, sfx, b-roll swap, in seconds

Writes `data/forensics/<video_id>.json`.

### `teardown` — describe the container
Turns forensics JSON into a reusable pattern. Names the format
("numbered-list-with-countdown-reveal", "contrarian-claim-then-proof"), states
the promise the hook makes, and writes the reusable skeleton with the specifics
stripped out. Writes `data/patterns/<pattern_name>.md`, appends to `PATTERNS.md`.
Patterns accumulate. After 20 teardowns she owns a format library nobody else has.

### `remix` — write her version
Input: the container (structured), her domain brief, `brie-voice` skill, banned
word list. **Never sees the source video.**

Output is the build sheet in `templates/brief.md`. Non-negotiables:
- First 10s must match the source hook word-for-word, with only proper nouns
  and the domain swapped
- Script must be timed to land within ±2s of source duration at source WPM.
  The agent computes `target_words = (duration_sec / 60) * source_wpm` and
  writes to that word count. This is arithmetic, not vibes.
- Every line carries a timecode
- Exactly 3 "make it mine" deltas, no more

### `producer` — build it without a camera
- HeyGen API: avatar video from the script. Pay-as-you-go from $5;
  Avatar III digital twin ≈ $1/min, Avatar IV photo avatar ≈ $3/min.
  A 45-second clip on Avatar III is about $0.75.
- Thumbnail: render from the build sheet spec. Canva via her existing template,
  or Adobe Express, or a Pillow script for full pixel control.
- Captions: burn in at the measured position/size/color
- Assembly: `ffmpeg` — avatar track + b-roll cuts at the measured cut timecodes
- Hands off to Blotato for scheduling (she already runs it, and it is Sabrina's
  own tool, so platform quirks are already solved)

### `scorecard` — close the loop
72 hours after each post: pull her metrics, compute her own outlier multiple
against her own median, write to `data/results/`. Monthly, correlate: which
patterns, which hook types, which durations, which thumbnail configurations.

This is the agent most people skip. It is the only one that compounds.

---

## 3. Cadence — and why not to batch

Sabrina's instruction: 3x/day short form, don't batch, learn from yesterday
before you post today.

The engine honors that with a split schedule:

| When | What runs | Human time |
|---|---|---|
| Sunday 06:00 | `scout` → `forensics` → `teardown` → `remix` on 6 candidates | 20 min review |
| Mon–Sat 05:30 | `producer` builds the day's 3 clips from approved briefs | 10 min approve |
| Daily 09:00 | `scorecard` on posts from 72h ago | 0 min, read the digest |

So the *research* batches weekly — that's just data collection, batching it
costs nothing. The *production* stays daily, which is the part Sabrina is
actually protecting: yesterday's numbers inform today's pick. The queue holds
6 briefs and she pulls 3/day from it; when a brief scores, `remix` is told to
generate two more in that pattern.

---

## 4. Cost floor — the honest number

She asked for the minimum that actually works. It is smaller than expected,
because the roster is small on purpose.

| Line | Volume | Cost |
|---|---|---|
| Apify Starter plan | required for pay-as-you-go; Free tier hard-blocks at $5 with no overage | **$19/mo** |
| Instagram reels scraped | ~10 creators × 30 posts × 4 sweeps = 1,200/mo @ $2.60/1k | ~$3.12 |
| TikTok videos scraped | ~800/mo @ $1.70/1k | ~$1.36 |
| YouTube | Data API v3, 10,000 units/day free — a full sweep costs ~30 units | $0 |
| HeyGen | 90 clips/mo × 45s on Avatar III @ ~$1/min | ~$68 |
| Whisper | runs locally | $0 |

**Research: ~$19/mo** (the $19 plan fee covers the ~$4.50 of actual usage with
huge headroom — she will not approach the $19 credit). **Production: ~$68/mo**
at full 3x/day volume, and that is the line to watch, not the scraping.

Her instinct was right. A handful of tracked creators is the correct design.
The reason to scan 30 posts each rather than 3 is that the median is what makes
the outlier detectable — you need the baseline to know what a spike is.

---

## 5. Repo layout

```
copycat-engine/
├── targets.yaml
├── PATTERNS.md                 # grows with every teardown
├── agents/
│   ├── scout.py
│   ├── forensics.py
│   ├── teardown.py
│   ├── remix.py
│   ├── producer.py
│   └── scorecard.py
├── lib/
│   ├── sources/{youtube,instagram,tiktok}.py
│   ├── vision.py               # OCR + color + face measurement
│   ├── audio.py                # transcript, WPM, beat map
│   └── scoring.py
├── data/
│   ├── sweeps/ candidates/ forensics/ patterns/ briefs/ results/
├── templates/brief.md
├── out/<date>/<slug>/          # briefs + thumbnails + rendered mp4
└── .env.example
```

Markdown briefs in `data/briefs/` are the source of truth and are git-committed.
`scout` and `scorecard` also sync to Airtable so the kanban view stays live.
If Airtable and git disagree, **git wins** — Airtable is a view, not a database.
