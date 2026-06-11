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

## 2. Time-selective audio effects: concat with EXACT-length pieces

Goal: apply an effect (pitch-shift a guest's voice) only during N specific windows. Two tempting approaches, **both wrong in different ways** — and the fix is the third:

**Approach A — naive cut-and-stitch (DRIFTS).** `asplit` → `atrim` each interval → process some → `concat`. Each pitched piece is time-stretched by `atempo`, which is *not* perfectly duration-exact, so every joint carries a sub-millisecond error. Across ~130 joints on a 100-minute video those errors accumulate into noticeable A/V desync (the video is copied = exact; the rebuilt audio slides).

**Approach B — volume-gated continuous tracks (SYNC-PERFECT but OOMs at scale).** Keep full-length tracks, mute by time, sum with `amix`:
```
[base0]volume=0:enable='between(t,a1,b1)+between(t,a2,b2)+...'[anorm];    // untouched, muted INSIDE windows
[base1]<pitch>,volume=0:enable='lt(between(t,a1,b1)+...,1)'[ap0];          // pitched, muted OUTSIDE its windows
[anorm][ap0]...amix=inputs=K:normalize=0:dropout_transition=0[outa]
```
The timeline never moves, so sync is perfect — but the `enable=` expression has one `between()` term **per window**, and at ~130 windows ffmpeg dies with **`Error initializing complex filters. Cannot allocate memory`**. The render then produces no file. Great for a handful of windows, unusable at scale.

**Approach C — concat, but force each piece to its EXACT length (the answer).** Cut-and-stitch like A, but after pitching a piece, pin it to its precise target duration `D = end - start` with `apad` + `atrim`, so the joined timeline can't drift:
```
[ai]atrim=START:END,asetpts=PTS-STARTPTS,<pitch chain>,apad=whole_dur=D,atrim=0:D[si]
[s0][s1]...[sN]concat=n=N:v=0:a=1[outa]
```
Normal (un-pitched) pieces are already sample-exact from `atrim`; pitched pieces are forced exact by `apad`(pad if short)+`atrim`(trim if long). Verified at **130 windows on real 44.1k audio: A/V within ~33 ms, a marked beep stays at its original time, peak memory ~120 MB** (vs the OOM in B). Pass the (large) filtergraph via `-filter_complex_script <file>` to dodge any arg-length limit.

Summary: **A drifts, B doesn't scale, C is both sync-safe and scalable.** Default to C for any "effect on selected windows" job.

Notes:
- Still resample to a known rate before the pitch chain (§1).
- `apad=whole_dur=D` pads silence up to D; `atrim=0:D` caps at D → exactly D. The ±few-ms of content trimmed/padded at a window edge is inaudible; the sync win is worth it.
- `volume`'s `enable=` (used in B) is a timeline expr: filter ACTIVE when non-zero, bypassed when zero. Fine for a few windows; don't build one with 100+ terms.

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

## 9. Fail loudly — never mark a broken render "complete", never silently skip a privacy effect

A multi-step render (disguise → cut → stitch → upload) had a `.catch(() => null)` on the disguise step "so it wouldn't break the whole render." When the disguise OOM'd, it fell through, the stitch *also* failed, and the job was marked **complete with a dead download link**. Two distinct failures of judgement:

- **A failed step must not be swallowed into a fake success.** After the final stitch, `fs.pathExists(outputPath)` — if it's missing, `throw` and set the job to **error**, don't report complete with a URL that 404s. "The user couldn't find their render" was the symptom of a phantom-complete.
- **A safety/privacy effect that fails must ABORT, not fall through.** If voice-disguise (the whole point: hide a shy guest) fails, you must NOT continue and ship a render with the guest's *real* voice. Re-throw and error the job. Silently producing the un-protected version is worse than failing. Same logic for any redaction/blur/mute step.

General rule for media pipelines: prefer **fail-closed**. A loud error the user can retry beats a silent wrong output they trust.

---

## Quick checklist for any audio/video feature

- [ ] Resample to a known rate before any `asetrate`/PTS math.
- [ ] Time-selective audio effects → concat with **exact-length pieces** (`apad`+`atrim`); not naive stitch (drifts), not volume-gated `amix` at scale (OOMs ~130 windows).
- [ ] After render, `ffprobe` video vs audio stream durations (must match).
- [ ] Reproduce on a REAL input file (real sample rate/codec/length), not just synthetic — and at REAL scale (130+ windows), not 3.
- [ ] Heavy re-encode → background job + status polling (never a synchronous request).
- [ ] Confirm the deploy is actually live before judging a "fix didn't work."
- [ ] Persist user selections; recover expensive/in-memory results instead of re-running.
- [ ] Fail closed: verify the output file exists before "complete"; abort (don't fall through) if a privacy/safety effect fails.
