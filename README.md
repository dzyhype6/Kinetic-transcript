# Kinetic Type

A single-file lyric-video deck. Drop in a track, type the words, tap the timing,
put a picture behind it, record a WebM. No build step, no dependencies, no server
— `index.html` is the whole program.

## Why it renders to canvas

The first version animated DOM nodes with CSS keyframes. That looks fine going
forwards and falls apart everywhere else: scrubbing backwards replays nothing,
a paused animation holds a frame the timeline disagrees with, and there is no
way to capture it except a screen recording.

Everything here is `render(t)` — one pure function from time to pixels. Which
buys:

- scrubbing that matches playback exactly, in both directions
- frames rendered off-clock (that is how the contact sheet works)
- `canvas.captureStream()` into `MediaRecorder`, so the export is the real thing

## Audio, honestly

| Source | Clock | Waveform & beats | Sound in the export |
|---|---|---|---|
| Local file | yes, seekable | yes, offline analysis | yes |
| Tab capture (`getDisplayMedia`) | wall clock | yes, live | yes |
| Microphone | wall clock | yes, live | yes, plus your room |
| Spotify Web API | yes, polled + interpolated | no | no |
| Nothing | free-running | no | video only |

### On Spotify

Spotify's stream is Widevine-encrypted. The Web Playback SDK will play a track
in a page, but the element it creates is opaque: no `MediaElementSource`, no
sample access, no recording. That is the entire purpose of the DRM and there is
no way around it from a browser.

What the Web API *does* hand over is `is_playing`, `progress_ms` and the track
object. So this app uses Spotify as a **clock** and an **artwork source**, and
pairs it with tab capture when you also want the waveform and sound on the
export. Position is polled every 900 ms and interpolated between polls; drift
over 350 ms re-anchors hard, smaller errors are eased in so cards never jump.

Auth is PKCE — no client secret, nothing to leak.

1. Make a free app at [developer.spotify.com/dashboard](https://developer.spotify.com/dashboard)
2. Add the redirect URI the Audio tab prints, exactly as shown
3. Paste the Client ID in and hit Connect

Scopes used: `user-read-playback-state`, `user-read-currently-playing`,
`user-modify-playback-state`. Reading the position works on free accounts;
the play/pause/seek buttons need Premium.

**Spotify will not redirect back to a `file://` page.** Serve the folder first:

```sh
npx serve .        # then open the 127.0.0.1 address it prints
```

## Running it

```sh
git clone <this repo> && cd kinetic-type
open index.html                  # everything except Spotify
# or
npx serve .                      # everything
```

Chrome or Edge on desktop for the full set. Firefox records but cannot share
tab audio. Safari has no `MediaRecorder`, so no export.

## Script format

One card per row:

```
12.40 | 2.10 | we run the block at *midnight*
14.50 | 1.90 | nobody knows our names
```

Or just the words, and let the timing come later. `start | duration | text`.

- `[Chorus]` and `(intro)` rows are dropped
- `*stars*` mark the accent word when the hit mode is set to honour them
- `//` forces a line break inside a card
- `.lrc` imports and exports, so timings move between tools

## Timing a track

1. Load the file, press play, hit **Tap to time**, tap once per card.
   `Z` takes back the last tap, `Esc` stops.
2. Or **Detect beats** — an RMS envelope at ~86 Hz, positive first difference,
   adaptive-threshold peak picking, autocorrelation for the tempo. Then
   **Snap to beats** pulls every card onto the nearest onset within 320 ms.
3. Or tap the tempo by hand and lay a grid from the playhead.

Then fix it on the timeline: drag a block to move it, drag its right edge to
stretch it, `[` and `]` nudge 50 ms.

## Export

WebM, VP9 + Opus, up to 16 Mbps, recorded in real time at full resolution
regardless of the preview quality setting. Keep the tab in front — a
backgrounded tab throttles `requestAnimationFrame` and the render stutters.

```sh
ffmpeg -i edit.webm -c:v libx264 -crf 18 -pix_fmt yuv420p edit.mp4
```

A cross-origin image taints the canvas and the browser then refuses to hand
over its pixels, which kills recording *and* stills. Spotify cover art is the
usual culprit — the app says so in the diagnostics panel rather than failing
quietly. Save the cover and drop it in as a file to get the export back.

## Keyboard

```
Space      play / pause
← →        seek 1s        (Shift: 5s)
, .        step one frame
T          arm or fire a tap
Z          take back the last tap
[ ]        nudge selected card 50ms
G          safe-area guides
Ctrl+Z     undo
Esc        stop tapping
```

## Saved edits

Look and timings go to `localStorage`; media does not. Export a `.json` to move
an edit between machines, then re-attach the track and the plates.

## The diagnostics panel

It exists because a blank stage should never be a mystery. Every subsystem
reports its state, and every failure names its own cause and the way out —
which codec was refused, which scope is missing, why the canvas is tainted,
why a card got scaled down. If something looks wrong, read the bottom right
before reading the code.
