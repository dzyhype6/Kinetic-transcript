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

**Serve it.** Don't open the file directly.

```sh
git clone <this repo> && cd kinetic-type
npx serve .            # then open the 127.0.0.1 address it prints
# or, with no node:
python3 -m http.server 8000
```

Opening `index.html` off the disk still works for the type, the plates and the
export, but two things break on a `file://` page and neither fails obviously:
**transcription**, because browsers refuse to load a module from a `file://`
origin, and **Spotify**, which will not redirect back to one. Serving the
folder costs one command and avoids both.

Chrome or Edge on desktop for the full set. Firefox has no tab-audio capture.
Safari records from 17.4 — as MP4 rather than WebM — and offline render needs
WebCodecs, which means Chrome, Edge, Safari 16.4+, or a recent Firefox;
without it the app falls back to real-time recording on its own.

## Phone and laptop

The same file, the same code path, two very different shapes.

On a laptop it is the two-column deck it always was: stage on the left,
panel on the right, every keyboard shortcut live.

On a phone the preview is **pinned** to the top of the screen and only the
panel scrolls underneath it, because a lyric editor where moving a slider
scrolls the picture out of sight is not an editor. The `⤢` button in the
transport cycles the preview through three sizes — working, large, and out of
the way entirely — since judging the picture and writing the script want
opposite halves of the screen. Turn the phone sideways and it goes back to two
columns.

What changes with a touchscreen, keyed off `pointer: coarse` rather than the
window width, so a touchscreen laptop gets it too:

- every control is at least 44px, and every text field at least 16px — below
  that, iOS Safari zooms the whole page when an input takes focus
- timeline blocks carry an 11px grab margin, and the stretch handle on the
  right edge grows to match, capped at a third of the block so short cards
  stay draggable
- **while `Tap to time` is armed, the picture itself is the tap pad** — your
  thumb is already there, and it is the one target big enough to hit without
  looking. `↶ Undo tap` and `Done` appear beside it, because `Z` and `Esc`
  are not reachable
- the diagnostics strip collapses to its one-line verdict and opens on demand
- the preview renders at half resolution; the recorder forces full resolution
  back on, so the export is unchanged

Two things a phone genuinely cannot do, and the app says so rather than
failing quietly: tab audio capture (`getDisplayMedia` is desktop-only in
Chrome and Edge) and Spotify's redirect back to a `file://` page. Everything
else — local file, mic, beat detection, recording — works.

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

### One card, different

A deck where every card animates identically is a deck, not an edit. So a row
can carry a fourth field of `key=value` overrides that apply to that card
alone:

```
12.40 | 2.10 | we run the block | anim=glitch size=200 color=#FF4A17
14.50 | 1.90 | nobody knows      | font=bebas align=left hit=all
```

`anim` `font` `size` `posy` `align` `case` `treat` `hit` `color` `accent`
`lh` `track` `stagger` `indur` `outdur`. Anything else — an unknown name, or a
value out of range — makes the whole field text again, so a lyric that happens
to read `cost=12` survives being typed.

The seven most-used live in a panel under **Lines**, where empty means "as the
deck". Cards carrying an override are marked in the list and striped blue on
the timeline. The panel and the fourth field are two views of one thing.

## Transcribing a track

You do not have to type the words in. **Lines → Transcribe** runs Whisper over
the loaded file and writes a timed script from it, in Swahili, English, or any
of the ~99 languages Whisper knows — pick one, or let it detect.

It runs **on this machine**. The model is fetched once from jsDelivr and cached
by the browser; every run after the first works offline, and the audio is never
uploaded anywhere. No API key, no account, no server — the same deal as the
rest of the app.

| Model | Size | Use it when |
|---|---|---|
| Tiny | ~40 MB | roughing out timings, or a slow machine |
| Base | ~80 MB | the usual choice |
| Small | ~250 MB | the words matter and you can wait |

WebGPU is used where the browser has it and WASM everywhere else, which is the
difference between a couple of minutes and rather more.

**Be realistic about the words.** Whisper was trained on speech. Sung vocals
over a full mix are a harder problem than speech, and harder again for a lower
resource language — Swahili included. On a vocal stem or an a cappella it is
very good. On a dense master it gives you a draft to correct, and it will
sometimes loop a phrase through an instrumental break (repeats three deep are
dropped on the way in).

The **timings** are the better half of what you get either way, because they
land the cards where the words actually are — which is the part that takes the
longest by hand.

Audio is fed through in 30 second windows with a 3 second overlap, so a word
across a boundary is heard whole by one of them. That is also what makes the
progress bar mean something and **Stop** stop: it ends after the window it is
in and keeps everything heard so far.

## Timing a track

1. Load the file, press play, hit **Tap to time**, tap once per card.
   `Z` takes back the last tap, `Esc` stops.
2. Or **Detect beats** — an RMS envelope at ~86 Hz, positive first difference,
   adaptive-threshold peak picking, autocorrelation for the tempo. Then
   **Snap to beats** pulls every card onto the nearest onset within 320 ms.
3. Or tap the tempo by hand and lay a grid from the playhead.

Then fix it on the timeline: drag a block to move it, drag its right edge to
stretch it, `[` and `]` nudge 50 ms.

### Zooming the timeline

A three minute song across a 370px phone gives a 2.4s card under five pixels
of width — not something you can select, let alone drag. So the timeline shows
a window onto the track rather than always the whole of it:

- **pinch** to zoom, or roll the **wheel**; both zoom about the point under
  your fingers, so whatever you were looking at stays put
- **shift-wheel** pans, and `-` / `=` zoom from the keyboard
- **double-tap or double-click**, or press `0`, to fit the whole track again
- while a zoomed track plays, the window pages along to keep the playhead in
  frame, and a three-pixel bar along the top shows where you are in the track

Starting a pinch cancels whatever drag the first finger had begun, so zooming
never leaves a card dragged halfway across the song.

## Export

Two ways out, and the first is the default.

**Offline render** draws every frame on demand and hands it straight to a
`VideoEncoder`. No clock is involved, so no frame can be dropped, and the tab
can sit in the background while it works — go do something else. This is what
`render(t)` being a pure function from time to pixels was always for.

Speed depends on the machine. A hardware VP9 encoder beats real time
comfortably; software VP9 at 1080×1920 is roughly level with it — measured at
1.1× on a CPU-only box, where drawing the frames took 17ms of 3.3s and the
encoder took the rest. It asks for hardware first. The panel reports which one
it got and how fast it went, so you learn what your machine does.

The container is muxed here, longhand, in about 150 lines of EBML. Export is
the one button everything else leads to; it is not allowed to break because
somebody's network blocks a script host. That risk is worth taking for an
opt-in extra like transcription and not for this.

**Real-time recording** is still there behind the toggle, and it is the right
choice in one case: when the sound is coming from tab capture or the mic.
Only a loaded file can be re-read offline, so an offline render of a
tab-captured edit would be silent. Real time uses `MediaRecorder` — WebM
(VP9 + Opus) where the browser has it, MP4 (H.264 + AAC) on Safari and iOS.

Either way: up to 16 Mbps, full resolution regardless of the preview quality
setting.

```sh
ffmpeg -i edit.webm -c:v libx264 -crf 18 -pix_fmt yuv420p edit.mp4
```

Two things the offline path gets right that filming the preview cannot. Video
plates are **seeked** to the instant being drawn rather than left wherever
they happened to be playing. And beat pulse and shake read an envelope
measured off the decoded audio, so the motion the preview promised is the
motion in the file — there is no live meter running during an offline render
to read instead.

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

On touch, `?` in the title bar prints the equivalents: the stage is the tap
pad, `↶ Undo tap` and `Done` stand in for `Z` and `Esc`, and `◀ −50ms` /
`+50ms ▶` under Lines stand in for `[` and `]`.

## Saved edits

Look and timings go to `localStorage`. The plates and the track go to
**IndexedDB** beside them, so loading a saved edit brings its media back with
it instead of asking you to re-attach everything — `localStorage` holds strings
and about five megabytes of them, which is why it never could.

The project record keeps ids, not bytes. A vault row that nothing points at any
more — a deleted project, a discarded autosave — is swept on the next boot, and
the `stored` line in the diagnostics panel says how many files are down there
and how close to the browser's quota they are.

Export a `.json` to move an edit between machines. That carries the edit but not
the bytes, so re-attach the media on the other side; if a plate is missing when
an edit loads, the panel names the file rather than leaving a silent gap.

### Autosave

Nothing here touches a server, so the only copy of an edit is the tab it is in
— and iOS evicts a backgrounded tab whenever it feels like it. Switch apps to
find the lyrics, come back, and the deck is empty.

So the project blob is written to `localStorage` on a lazy timer and again on
`pagehide` and on the way to the background. On the next boot, if there is an
edit there that is not what is already on screen, a bar offers it back —
**Restore** or **Discard**. It is never applied silently: quietly replacing
what somebody just typed is worse than losing it.

Same caveat as a saved edit: the look and the timings come back, the media
does not.

## The diagnostics panel

It exists because a blank stage should never be a mystery. Every subsystem
reports its state, and every failure names its own cause and the way out —
which codec was refused, which scope is missing, why the canvas is tainted,
why a card got scaled down. If something looks wrong, read the bottom right
before reading the code.
