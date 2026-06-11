# FFmpeg audio/video pipelines — the traps that waste hours

_Last updated: 2026-06-11_

## Why this exists

Building the RANT Squad Clip Maker (hooks, section removal, voice disguise, auto speaker detection) surfaced a set of FFmpeg + media-pipeline bugs that all looked fine until you actually watched/listened to the output. A render that "completes" with exit code 0 can still be silently broken — wrong duration, drifted lip-sync, mistimed audio. These are the lessons. They apply to any project that cuts, joins, pitches, or re-encodes audio/video.

The golden rule: **for media, "the command succeeded" is not "the output is correct." Always verify the artifact.**

---

## 1. The pitch-shift sample-rate trap (this one cost the most)

To pitch-shift audio while keeping its duration, the classic chain is:

```
asetrate=SR*r, aresample=SR, atempo=1/r
```

`asetrate` does NOT resample — it *relabels* the stream's sample rate. So the value of `SR` you use **must equal the input's real sample rate**, or the math is wrong.

The bug: I hardcoded `SR=48000`. Most podcast/phone/screen-recorder audio is **44100 Hz**. On a 44100 source, `asetrate=48000*r` mis-times everything — the pitched audio came out ~8% short (`44100/48000 = 0.91875`) and progressively earlier, so **the lips drifted off the sound** over a long video.

**Fix — always normalize the rate FIRST, for any source:**

```
aresample=48000, asetrate=48000*r, aresample=48000, atempo=1/r
```

The leading `aresample=48000` converts whatever comes in (44.1k, 48k, 32k, 16k, mono) to a known rate, so the pitch math is exact regardless of source. Verified across 48k/44.1k/32k/16k/22.05k: durations match within ~30 ms and a marker beep stays at its original timestamp.

**General rule:** any filter that *reinterprets* rather than *resamples* (`asetrate`, raw PTS math) must be fed a known, normalized rate. Resample up front.

---

## 2. Time-selective audio effects: gate the volume, don't cut-and-stitch

First instinct for "disguise the audio only during these 130 windows" was: `asplit` → `atrim` each interval → process some → `concat` back together. **It drifts.** Every `atrim`/`concat` joint carries a sub-millisecond timing error, and across 130 joints on a 100-minute video those errors accumulate into seconds of A/V desync. The video stream is copied (exact); the rebuilt audio is not — so they slide apart.

**Better pattern — keep continuous, full-length tracks and gate them by time:**

```
[0:a]aresample=48000,asplit=K[base0][base1]...;
[base0]volume=0:enable='between(t,a1,b1)+between(t,a2,b2)+...'[anorm];   // untouched, muted INSIDE the windows
[base1]<pitch chain>,volume=0:enable='lt(between(t,a1,b1)+...,1)'[ap0];  // pitched, muted OUTSIDE its windows
[anorm][ap0]...amix=inputs=K:normalize=0:dropout_transition=0[outa]
```

Why it's correct: no segment boundaries in the audio timeline, so nothing drifts. The untouched track carries the timeline (exact, video-aligned); each effect track is full-length and only *audible* inside its windows; exactly one track is non-silent at any instant, so `amix:normalize=0` sums them cleanly. Scales to hundreds of windows with just `1 + (#distinct effects)` branches — and the host's audio stays perfectly in sync because it's never touched.

Notes:
- `volume`'s `enable=` is a timeline expression: filter is ACTIVE (applies `volume=0`) when the expr is non-zero, bypassed (full volume) when zero.
- Commas inside `between(t,a,b)` survive the `-filter_complex` parser when wrapped in single quotes: `enable='between(t,1,2)+between(t,3,4)'`. (Passed via argv, no shell — the quotes are interpreted by ffmpeg's filtergraph parser.)
- Mixed sample rates break `amix`. Resample every branch to one rate before mixing (the `aresample` before `asplit` covers the untouched branch too).

---

## 3. Verify the artifact with ffprobe — and test on REAL input

The disguise render "completed" (exit 0, status complete) but the audio was **74 seconds shorter than the video**. Nothing in the pipeline errored. The only way I caught it:

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=duration -of csv=p=0 out.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=duration -of csv=p=0 out.mp4
# audio duration ≈ video duration? if not, it's broken regardless of exit code.
```

For media work, after any render: **probe the output's video vs audio stream durations.** A gap = drift/truncation.

Two sub-lessons that hid the bug:
- **Test on real input, not just synthetic.** My synthetic test used clean audio and *masked* the bug (a different ffmpeg version + `amix:duration=longest` papered over the short branch). The real 44100 file exposed it. Always reproduce on a file with the actual characteristics (real sample rate, real codec, real length).
- **A tell-tale "exact pre-fix value."** When the staging output was *identically* 842.176 s after my fix, that exact match to the pre-fix value meant the **deploy wasn't live yet** — old code was still serving. (See `DEBUGGING_DEPLOYMENTS.md` — confirm the new code is actually running before concluding a fix failed. On Railway, an env-var change OR a push triggers a redeploy that takes ~60–90 s.)

Timing/alignment check beyond duration: put a marker (a 1 s tone inside a target window), run the effect, and use `silencedetect` to confirm the marker is still at its original time:

```bash
ffmpeg -i out.mp4 -af silencedetect=noise=-40dB:d=0.2 -f null - 2>&1 | grep silence_
```

---

## 4. Static ffmpeg/ffprobe for local debugging (when the box doesn't have them)

Don't debug media bugs only by re-rendering on the server. Grab static macOS binaries and reproduce the exact filtergraph locally in seconds:

```bash
curl -sL -o ffprobe.zip https://evermeet.cx/ffmpeg/getrelease/ffprobe/zip && unzip -oq ffprobe.zip
curl -sL -o ffmpeg.zip  https://evermeet.cx/ffmpeg/getrelease/ffmpeg/zip  && unzip -oq ffmpeg.zip
```

This turned a slow "push → wait for deploy → render → guess" loop into a fast local repro that pinpointed the 44100 issue. Reproduce the *exact* `-filter_complex` string the server uses.

---

## 5. Heavy re-encodes must run in the background, not in a synchronous request

A full re-encode (normalize-to-CFR on import, long render) inside a synchronous HTTP request **times out** — the server keeps working and finishes, but the browser/edge gives up after a couple of minutes and shows **"Failed to fetch."** The user thinks it failed; it didn't.

Pattern: return a job id immediately, do the heavy work in the background, have the frontend **poll a status endpoint** (`step`/`progress`, done-signal, error). Reuse one status channel for prepare/render/normalize/detect. Anything that can exceed ~30–60 s of wall-clock does NOT belong in a request the browser is awaiting.

---

## 6. Timelines are not portable between encodings (YouTube/VFR)

"My timestamps don't match YouTube" is usually because two different *encodings* of the same video have two different clocks — dominated by **variable frame rate** (phone/screen-recorder files) plus container edit-lists. YouTube re-encodes to constant frame rate, which shifts every timestamp; the drift grows the deeper into the video.

- The only *exact* fix is to use the **same bytes the reference uses** — e.g. download the video from YouTube (`yt-dlp`) and work on that copy, so the transcript timestamps line up by construction. Raise any download length/size caps for long-form.
- Re-encoding a local file to CFR (`-vsync cfr -r <fps>`, reset PTS) is a *best-effort* approximation, not a guarantee — it won't perfectly reproduce another service's encoder. Offer it, but be honest it's approximate.
- Symptom triage: gap **tiny-at-start, growing** = frame-rate/timebase drift; **constant offset** = an intro/trim difference or edit-list.

---

## 7. Speaker diarization ("who spoke when") — API choice

When you need to label speakers (e.g. to disguise one guest):

- **Use AssemblyAI.** More accurate at separating *similar* voices (diarization is its focus), simpler API, and cheaper. `POST /v2/transcript` with `speaker_labels:true` (+ `language_detection:true`), poll the transcript id, read `utterances[]` (`speaker`, `start`/`end` in ms, `text`). ~**$0.15/hr** of audio (~$0.26 for a 1h45m podcast). Free starter credits, no card needed to begin.
- **Deepgram** is faster/real-time but more often mixes up similar voices and costs more (~$0.46/hr base + ~$0.12/hr diarization). Only wins if you need live transcription.
- **Diarization is language-agnostic** — it's based on voice characteristics, not words. It works on Bengali etc. even when you don't need (or trust) the transcript text. If you already have transcripts elsewhere, you only need the speaker time-ranges.
- It bills per hour of audio and is a flat one-time cost per file → **persist the result** (and the user's selections) so you never re-charge for the same video.

---

## 8. Persist user choices, recover ephemeral results — don't re-charge

In-memory job state (detected speakers, prepared sources) is fragile — a redeploy or idle restart wipes it. Two rules:

- **Auto-save the user's actual selections** (marked segments, chosen voices) to durable storage, keyed to the project — same as any other editor state. Then a refresh/reopen restores their work for free.
- **Recover paid/expensive results from the live status endpoint** when you can, instead of re-running them. After a refresh, re-reading the still-in-memory result off `/status` saved the user from paying for a second diarization. (`updateJob` that merges — `{...current, ...updates}` — preserves fields like `speakers` across a restore, which made this possible.)

---

## Quick checklist for any audio/video feature

- [ ] Resample to a known rate before any `asetrate`/PTS math.
- [ ] Time-selective audio effects → volume-gated continuous tracks, never cut-and-stitch.
- [ ] After render, `ffprobe` video vs audio stream durations (must match).
- [ ] Reproduce on a REAL input file (real sample rate/codec/length), not just synthetic.
- [ ] Heavy re-encode → background job + status polling (never a synchronous request).
- [ ] Confirm the deploy is actually live before judging a "fix didn't work."
- [ ] Persist user selections; recover expensive/in-memory results instead of re-running.
