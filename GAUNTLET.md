# GAUNTLET — paste this into Claude Code

> Drop `SPEC.md`, `targets.yaml` and `templates/brief.md` in an empty repo
> alongside this file, then paste everything below the line into Claude Code.

---

You are building the **Copycat Engine**: a pipeline that finds proven short-form
video, measures it to the pixel and the tenth of a second, and emits a build
sheet that gets produced with a HeyGen avatar. The operator never records.

Read `SPEC.md` first. It is the contract. `targets.yaml` is the roster.
`templates/brief.md` is the exact output shape. Do not redesign any of them —
if you believe one is wrong, write your objection to `OBJECTIONS.md` and build
it as specified anyway.

## Already provisioned — do not set any of this up

The operator's accounts, subscriptions and API keys are **already live**. Read
`.env`, use what's there, and never walk her through creating an account, a key,
a plan or a token. If a key is missing, say which one and stop — don't guide.

Confirmed and ready:
- Google Cloud project `copycat-engine`; YouTube Data API v3 enabled; key
  restricted to that API. `YOUTUBE_API_KEY`
- Airtable base **Copycat Engine**, base id from your `.env`; token scoped to
  that base only, with data.records read/write and schema.bases read/write.
  `AIRTABLE_TOKEN` / `AIRTABLE_BASE_ID`
- Apify (paid plan, pay-as-you-go active) `APIFY_TOKEN`
- HeyGen (API credits active) `HEYGEN_API_KEY`; avatar/look and voice choices
  come from the canonical operator registry and runtime configuration
- Anthropic `ANTHROPIC_API_KEY`
- Blotato `BLOTATO_API_KEY`

Two standing constraints:

1. **Her own channels do not exist yet, and that blocks nothing.** Every account
   in `targets.yaml` belongs to someone else — the pipeline only ever reads. Do
   not check for, ask about, or wait on her handles. Phase 7 builds to disk.
2. **Nothing renders or posts without her explicit say-so.** `producer` writes
   the HeyGen request payload and the thumbnail spec to `out/` and stops.
   Actually calling HeyGen requires `--approve` on the command line, and the
   Blotato scheduling leg is written but left switched off behind
   `PUBLISH_ENABLED=false` in `.env`. Never flip it yourself.

## Render rules — non-negotiable, read before Phase 7

The operator's canonical avatar and voice registry is the single source of truth
for render configuration. `producer` must read that registry at the start of
every run and fail loudly if it is missing. Never copy provider identifiers or
credential-like values into this repository; copied values drift and may expose
sensitive configuration.

The registry's operational rules bind this pipeline:

1. **Her face always speaks in her own cloned voice.** Generate the approved
   audio with the provider and settings named in the registry, then pass that
   audio to HeyGen for lip-sync. Never substitute or guess a voice identifier;
   a blank or unavailable registry value means stop and ask.
2. **Rotate looks, never repeat back-to-back.** Use the ordered look rotation
   defined by the registry/runtime configuration and record which look each
   brief used in `data/results/`.
3. **Use the registry's required framing and style.** Do not infer or silently
   replace them from stale repository notes.
4. Apply every per-look duration, crop, format, stabilization, and eligibility
   constraint from the registry. If the registry and this repository disagree,
   the registry wins.
5. Use the trained portrait orientation specified by the registry; do not
   silently switch to landscape.

## Run protocol

This is a gauntlet: **eight phases, each with a gate.** A gate is a shell
command. You may not begin phase N+1 until phase N's gate command exits 0.
You run continuously through all eight without pausing for approval, except
where a phase says STOP.

On every gate failure: read the actual error, fix the actual cause, re-run.
Three consecutive failures on the same gate → write the symptom, the three
attempts and your best theory to `BLOCKED.md`, skip to the next phase, and
list it in the final report. Do not spin. Do not stub a passing test to clear
a gate — a gate cleared by weakening its own assertion is a failed gate.

After each gate passes, append one line to `PROGRESS.md`:
`<phase> | <gate cmd> | PASS | <what exists now that didn't before>`

Commit after every passing gate. Conventional commits.

## Laws

1. **Container vs filling.** SPEC §1 is absolute. The `remix` agent is passed
   structured container JSON and is architecturally denied access to source
   video files, source transcripts past the hook, and source image assets.
   Enforce this in code — separate function signatures, not a comment.
2. **No invented measurements.** Every number in a brief traces to a measurement
   or an API field. If a value could not be measured, the field is `null` and
   the brief prints `UNMEASURED`. A plausible guess in a measurement field is
   the worst possible failure of this system.
3. **No guessed handles.** A handle with `verified: false` is never passed to a
   scraper. Unresolved handles are skipped and reported.
4. **Fail loud.** Empty scraper results raise. They never return `[]` silently —
   that is how a bad handle becomes "no outliers this week."
5. **Cache everything.** Every API response is written to `data/raw/` before
   parsing. Re-runs read cache. You will iterate on parsing far more often than
   the data changes, and every uncached re-run costs real money.
6. **Budget guard.** `lib/budget.py` tracks estimated spend per run. Any single
   run projected over **$5** halts and asks. Hard monthly ceiling **$25** on
   scraping, tracked in `data/spend.json`.
7. `uv` for deps. `ruff` + `pytest`. Type hints on every public function.
   No secrets in git; `.env.example` lists every key with a comment on where to get it.

---

## PHASE 0 — Skeleton and handle resolution

Scaffold the repo per SPEC §5. Then resolve every `verified: false` handle in
`targets.yaml`.

For each unresolved creator: search the web for their Instagram, TikTok and
YouTube handles. Confirm by fetching the profile and checking that the bio or
content actually matches the described person — a name match alone is not
confirmation. Write findings back to `targets.yaml` with `verified: true` and a
`resolved_via:` URL, or leave `false` with a `resolution_note:`.

If `targets.yaml` has a `seed_video`, resolve who posted it first. It is the
proof case: a dated post with a known result, and it becomes the first full
teardown. If one creator appears to have two accounts, confirm they are the same
person before sweeping both.

**GATE 0:** `uv run pytest tests/test_targets.py -q && uv run python -m agents.resolve --check`

`--check` exits 0 only when **every tier_1 creator has at least two verified
platform handles** and **the seed video, if any, is resolved to its creator**. Tier_2 may remain partially unresolved — report which.

## PHASE 1 — Sources

`lib/sources/youtube.py`, `instagram.py`, `tiktok.py`. Each exposes
`fetch_recent(handle: str, n: int) -> list[Post]` against one normalized `Post`
dataclass: `platform, video_id, permalink, posted_at, views, likes, comments,
duration_sec, caption, thumbnail_url, video_url, transcript_url, follower_count`.

YouTube: `playlistItems.list` on the uploads playlist (1 unit) → `videos.list`
batched 50 IDs per call (1 unit). **`search.list` is forbidden** — 100 units per
call against a 10,000/day cap. yt-dlp `--dump-json --skip-download` is the
fallback and the thumbnail source.

Instagram: Apify `apify/instagram-reel-scraper`. TikTok: Apify
`clockworks/tiktok-scraper`. Both by profile/username input.

Every response cached to `data/raw/<platform>/<handle>/<date>.json` first.

**GATE 1:** `uv run pytest tests/test_sources.py -q` — recorded-fixture tests for
each platform's parser, plus a test asserting an empty result raises rather than
returns `[]`, plus a test that fails if `search.list` appears anywhere in the
YouTube module.

## PHASE 2 — Scout

`lib/scoring.py` implements SPEC §0 exactly: `outlier_multiple`, `reach_ratio`,
`absolute_views`. Qualify on `outlier_multiple >= 3.0 AND absolute_views >= 100_000`.
**Sort by `outlier_multiple`, never by raw views.** Cap at 6 candidates.

`agents/scout.py` sweeps the roster, writes `data/sweeps/` and `data/candidates/`.

**GATE 2:** `uv run pytest tests/test_scoring.py -q` — must include the case
that proves the design: a 1M-view post from a 2M-follower creator whose median
is 1.2M scores *below* a 200K-view post from a 5K-follower creator whose median
is 40K. If that test passes with the ordering reversed, the scoring is wrong.

## PHASE 3 — Forensics: vision

`lib/vision.py`. Two passes, both required, per SPEC §2.

Measurement pass — OpenCV + PaddleOCR: text bounding boxes as % of frame height
and width, cap height as % of frame height, line count, word count, k-means
(k=5) dominant colors as hex with % coverage, face bbox via YuNet as % of frame,
face center as % coordinates, rule-of-thirds cell, WCAG contrast ratio of text
fill against the pixels directly behind the glyphs (not the global background —
sample the actual backing pixels).

Judgment pass — Claude vision on the frame, **with the measurement-pass numbers
supplied in the prompt** so it describes rather than estimates: typeface class,
stroke weight, case, tracking, outline/shadow treatment, three named font
substitutes available in Canva, visual hierarchy order.

Output must convert cleanly to px at both 1080×1920 and 1280×720.

**GATE 3:** `uv run pytest tests/test_vision.py -q` — run against three checked-in
fixture thumbnails with hand-measured ground truth. Assert every ratio within
±3% and every hex within ΔE 10. Assert `null` is returned, not a default, when
no face or no text is found.

## PHASE 4 — Forensics: audio and structure

`lib/audio.py` + `agents/forensics.py`.

- Transcript with word-level timestamps: platform subtitles where available,
  local `whisper` where not
- **Hook WPM and body WPM computed separately** — the hook is nearly always
  faster and that delta is part of the container
- Hook boundary: the timestamp where the promise stops and the payload starts
- Beat map: every sentence with start timecode
- Cut map: `ffmpeg -vf "select='gt(scene,0.3)',showinfo"` → every hard cut in seconds
- Frame grabs at 0.0/0.5/1.0/1.5/2.0/3.0s
- Pattern-interrupt inventory: zooms, text pops, sfx, b-roll swaps, in seconds

Writes the complete `data/forensics/<video_id>.json`.

**GATE 4:** `uv run python -m agents.forensics --video-id <fixture> --assert-complete`
— exits non-zero if any required field is missing *or* if any field is populated
with a default rather than a measurement.

## PHASE 5 — Teardown and Remix

`agents/teardown.py`: forensics JSON → named pattern + reusable skeleton →
`data/patterns/`, appended to `PATTERNS.md`.

`agents/remix.py`: **enforce Law 1 in the type system.** `remix()` accepts
`Container` and `DomainBrief`. It has no parameter that can carry a source
video path, a full source transcript, or a source image. Add a test that fails
if such a parameter is ever added.

Remix rules, all mechanically checked:
- First 10s matches source hook verbatim, proper nouns and domain swapped only
- `target_words = (duration_sec / 60) * body_wpm`; written output within 5%
- Every line carries a timecode
- Exactly 3 deltas
- Zero banned words: journey, amazing, tribe, hack, transformation, gentle
  reminder, in essence, at the end of the day, real talk
- Voice per the operator's `brie-voice` skill if present in the environment

Renders `templates/brief.md`. Writes `data/briefs/` and `out/<date>/<slug>/`.

**GATE 5:** `uv run pytest tests/test_remix.py -q` — asserts the banned-word
list is empty in output, the word count lands within 5% of target, exactly 3
deltas exist, every template field is filled or explicitly `UNMEASURED`, and
the signature-isolation test passes.

## PHASE 6 — Airtable sync

Base `Copycat Engine`, table `Briefs`. One row per brief: slug, status (single
select: draft/approved/built/posted/scored), pattern, source permalink, outlier
multiple, duration, platform, brief markdown path, thumbnail attachment, posted
URL, 72h results. Kanban grouped by status.

**Sync is one-way from git.** Git is truth; Airtable is a view. Status changes
made in Airtable are pulled back as the *only* field allowed to flow upstream.

**GATE 6:** `uv run python -m agents.sync --dry-run` prints the exact row diff
and exits 0 with zero destructive operations in the plan.

## PHASE 7 — Producer, Scorecard, Scheduling

`agents/producer.py`: **dry by default** — renders the HeyGen request payload,
the thumbnail spec and the cut list to `out/` and exits. With `--approve` it then
does HeyGen video generation (poll until complete, download),
thumbnail render from the build sheet spec, caption burn-in at measured
position/size/color, `ffmpeg` assembly against the measured cut map, Blotato
handoff for scheduling. Cost estimate before every HeyGen call; respect Law 6.

`agents/scorecard.py`: 72h post-mortem, her own outlier multiple against her own
median, `data/results/`, monthly pattern correlation.

Schedule per SPEC §3: Sunday 06:00 research sweep, Mon–Sat 05:30 production,
daily 09:00 scorecard. Ship as `cron/` entries plus a `justfile`; do not install
anything into the operator's crontab without asking.

**GATE 7:** `just smoke` — end-to-end on one checked-in fixture video with all
paid APIs mocked. Produces a complete brief, a rendered thumbnail PNG, and a
HeyGen request payload, and asserts total real spend for the run is $0.00.

---

## STOP

Now stop and write `REPORT.md`:

1. What runs end-to-end, verified by which gate
2. What is in `BLOCKED.md` and what you think is actually wrong
3. Handles still unresolved, and why
4. The three cheapest next improvements, ranked by impact per hour of work
5. Any key in `.env` that is missing or rejected by its API — name it and stop;
   do not explain how to obtain it
6. Projected first-month spend, itemized, against SPEC §4

Then run one live sweep in dry-run mode against the verified tier-1 creators
only, and show the six candidates it would select — with their direct per-item
links, their outlier multiples, and what each would cost to produce. Do not
produce anything. Wait.
