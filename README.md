# Copycat Engine

A spec-first build kit for an agent that finds short-form posts that outperformed their creator's own baseline, measures *why* (hook, duration, pacing, thumbnail geometry), and writes a build sheet for your own version, in your voice, with your face. Nothing renders or posts without the operator's approval.

## Start here

Four files. Put them in an empty repo and paste `GAUNTLET.md` into Claude Code.

| File | What it is |
|---|---|
| **GAUNTLET.md** | The build prompt. Eight phases, eight hard gates. Paste everything below the `---`. |
| **SPEC.md** | The architecture contract the prompt refuses to deviate from. Read §0 and §1 before anything else. |
| **targets.yaml** | The roster. Handles marked `verified: false` get resolved in Phase 0. |
| **templates/brief.md** | The output. Every field is a measurement or `UNMEASURED`. |

## Keys needed before the first live run

```
YOUTUBE_API_KEY=      # console.cloud.google.com → YouTube Data API v3 → free, 10k units/day
APIFY_TOKEN=          # apify.com → Settings → Integrations. Starter plan $19/mo required.
ANTHROPIC_API_KEY=    # vision pass on thumbnails
HEYGEN_API_KEY=       # app.heygen.com → Settings → API. Pay-as-you-go from $5.
AIRTABLE_TOKEN=
AIRTABLE_BASE_ID=
BLOTATO_API_KEY=      # optional, for scheduling
```

Apify's Free tier hard-blocks at $5 with no overage billing, which will kill a
sweep mid-run. Starter at $19/mo is the floor. Actual scraping usage at this
roster size runs ~$4.50/mo against that $19 credit.

## First week

1. Paste `GAUNTLET.md`. Let it run all eight phases. Read `REPORT.md`.
2. Approve the six candidates from the dry-run sweep.
3. Full teardown on the seed video first: a documented, dated post with a
   known result is the highest-value artifact in the roster.
4. Produce 3/day for six days. Don't batch. Read the 72h scorecard before
   picking the next day's three.

## The two things that decide whether this works

**Container vs filling** (SPEC §1). Copy the hook verbatim, the title verbatim,
the duration to the second, the WPM, the cut timing, and the thumbnail geometry
as ratios. Replace the face, the assets, the logo and everything after the hook.
That is exactly the copy that matters — a thumbnail performs on contrast ratio,
text-to-frame ratio and face scale, not on whose face it is. Her face at their
coordinates beats their face on her channel, because the follow requires
recognizing *her*.

**Outlier multiple, not raw views** (SPEC §0). "Copy the 1M video not the 10K
video" is a proxy for the real signal. A 1M-view post from a 2M-follower account
is below average. A 200K-view post from a 5K account is a format that broke
containment. Rank on the multiple against each creator's own median.

## Credits

The container-vs-filling idea comes from a public talk by Sabrina Ramonov on copying viral formats ethically. SPEC §0 replaces her "copy the 1M-view video" rule with an outlier multiple against each creator's own median.

## License

MIT, see [`LICENSE`](LICENSE).
