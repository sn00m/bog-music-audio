# Bog Music Sampler — Pure Data Patch Documentation

An OSC-triggered sample player built in Pure Data. Each incoming trigger
picks one of a set of 12-second `.wav` files, plays it through a per-voice
ADSR envelope, and outputs it to the audio interface. Triggers arrive over
the network as OSC and are converted internally into the same note/velocity
representation a MIDI note-on would produce, so the whole voice-allocation
and envelope system underneath behaves exactly like a MIDI instrument —
there's just no physical MIDI input anymore, since OSC is the only trigger
source in use.

## Files

There are two parallel sets of patches. They share the same design; the
only difference is how many samples each covers.

| Files | Samples covered | Note numbers (internal) | Voices |
|---|---|---|---|
| `sampler.pd` + `sampler-voice.pd` | all 15 samples | 60–74 | 8 |
| `cluster1.pd` + `cluster-voice1.pd` | 01_C3, 02_D3, 03_Eb3 | 60–62 | 4 |
| `cluster2.pd` + `cluster-voice2.pd` | 04_F3, 05_G3, 06_Ab3 | 63–65 | 4 |
| `cluster3.pd` + `cluster-voice3.pd` | 07_Bb3, 08_C4, 09_D4 | 66–68 | 4 |
| `cluster4.pd` + `cluster-voice4.pd` | 10_Eb4, 11_F4, 12_G4 | 69–71 | 4 |
| `cluster5.pd` + `cluster-voice5.pd` | 13_Ab4, 14_Bb4, 15_C5 | 72–74 | 4 |

The `sampler.pd` / `sampler-voice.pd` pair is the full 15-sample instrument.
The five `clusterN.pd` patches are self-contained thirds of the same
instrument — each one only knows about 3 of the 15 samples, so they can run
as separate Pd instances (e.g. on separate machines) while every note keeps
the exact pitch mapping it has in the full patch. A `clusterN.pd` main
patch always pairs with its own `cluster-voiceN.pd` — the number must match,
since that's how Pd resolves the abstraction name to a file.

Every `...N.pd` main patch is generated from the same template — same ADSR
behavior, choke gate, declick fade, note-off handling, and OSC input — just
parameterized by which 3 samples/notes it owns, and which OSC port it
listens on.

## Signal flow (main patch)

```
OSC in ──► [OSC-to-note conversion] ──► [choke gate] ──► poly N 1 ──► pack f f f ──► route 1..N ──► per-voice sample player (×N)
                                                                                                            │
                                                                                                            ▼
                                                                                               *~ 0.2 ──► dac~ (stereo)
```

1. **Input.** An OSC message arrives over the network and is converted into
   a note/velocity pair (see "OSC input" below) — this is the only trigger
   source; there's no physical MIDI input (`notein`) in these patches.
2. **Choke gate.** The converted trigger passes through a rate-limiting gate
   before reaching `poly`, so a burst of near-simultaneous triggers doesn't
   overload the voices (see "Choke gate" below).
3. **Voice allocation.** `poly N 1` assigns each note to one of N voices,
   stealing the oldest voice if all are busy. `poly`'s three outlets are, in
   order: **voice number, note, velocity** — this order matters (see the
   `pack`/`route` note below).
4. **`pack f f f`** combines `[voice#, note, velocity]` into a single list,
   with voice# first so `route` can dispatch on it.
5. **`route 1..N`** sends each voice's `[note, velocity]` pair to its own
   `sampler-voice` / `cluster-voiceN` instance.
6. Each voice independently decides which sample to play and runs its own
   ADSR envelope (see "Per-voice architecture" below).
7. All voices sum into `*~ 0.2` (fixed output trim) and out to `dac~ 1 2`
   (both stereo channels carry the same mono-summed signal).

### Why `poly`'s outlet order matters

`poly`'s outlets are **voice-number, note, velocity** — not note/velocity/voice
as you might expect by analogy with `notein`. `pack` fires as soon as its
hot (leftmost) inlet receives a value, using whatever is currently in the
other inlets — and Pd delivers a multi-outlet object's outputs right-to-left.
So `poly`'s outlets must be wired to `pack` in the same order they arrive
(voice# → pack inlet 0, note → inlet 1, velocity → inlet 2) or `pack` fires
before the note/velocity have actually arrived, producing garbage. This was
a real bug in an earlier version of this patch — see "Known-fixed issues"
below for the full list of subtleties this design already accounts for.

## Per-voice architecture (`sampler-voice.pd` / `cluster-voiceN.pd`)

Each voice is an abstraction instantiated multiple times (8× in the full
patch, 4× per cluster). It receives `[note velocity]` at its inlet and:

1. **Note-off detection.** Real MIDI note-off is velocity = 0 on the *same*
   note number (not note = 0, a common mistake). The voice checks velocity,
   not note, to decide whether this is a release.
   - **Note-on** (velocity ≠ 0): proceeds to the declick fade, then the
     attack/decay/sustain sequence below.
   - **Note-off** (velocity = 0): stops any pending attack timer and starts
     the release ramp using the current release-time setting.
2. **Declick fade.** Every note-on first ramps the voice's output to
   silence over 5ms, *then* sets up the new sample and starts the attack.
   This exists because `poly` steals voices under load — with only a
   handful of voices holding 12-second samples, playing enough notes in a
   short window guarantees stealing eventually. Without the fade, a stolen
   voice would jump straight from mid-playback at full volume to a brand
   new sample, an audible discontinuity (click). The 5ms silent gap makes
   every retrigger — stolen or not — start from silence.
3. **Sample selection.** After the fade, the note number is matched against
   this voice's list of valid notes (`select 60 61 62` etc.). A match sends
   `set <samplename>` to `tabplay~` (binding it to that array) followed by a
   `bang` (start playback from the top). Both are required — `tabplay~`
   doesn't accept a bare array name.
4. **ADSR envelope**, driven by four shared `[value]` globals set from the
   floatatoms at the top of the main patch:
   - **Attack**: ramp from 0 to `velocity/127` over the attack time.
   - **Decay**: ramp from the attack peak down to `sustain level × velocity/127`
     over the decay time.
   - **Sustain**: holds at that level until note-off (or until the sample
     naturally finishes playing — see below).
   - **Release**: on note-off, ramp to 0 over the release time.
   The envelope is a `line~` object multiplying `tabplay~`'s signal output
   via `*~`.
5. Voice output goes back to the main patch through the abstraction's
   `outlet~`.

### Why release is set to 12500ms

All 15 samples are exactly 12.0 seconds long. Release defaults to 12500ms so
that even if a note-off arrives the instant a note starts, the release ramp
still outlasts the sample — the sample always plays to completion rather
than being cut short. If you use different-length samples, adjust this
default (or the floatatom at runtime) accordingly.

## Controls (top of each main patch)

- **Attack / Decay / Sustain / Release** floatatoms (ms, ms, 0–1, ms).
  Defaults: 10 / 300 / 0.6 / 12500. These write to shared `[value]` globals
  that every voice abstraction reads — one set of knobs controls all voices.
- **Choke (ms)** floatatom, default 100. See below.

## Choke gate

Rate-limits *note-on* triggers only: any note-on arriving sooner than the
choke time after the last *accepted* note-on is silently dropped. Note-offs
always pass through unconditionally, so nothing gets stuck sounding.

Implementation: a `spigot` sits between the input source and `poly`'s note
inlet. A small state machine (`select 0` on velocity, an `f` flag, a `del`
timer) opens the spigot immediately for any note-off, and for note-ons only
opens it if a flag says "not currently choked" — in which case it also sets
the flag and schedules it to clear after the configured choke time.

## OSC input

The sole trigger source. A custom networked controller sends OSC messages;
the patch converts each one into a note/velocity pair and feeds it into
`poly`'s inlets — the same inlets a MIDI `notein` object would drive, so
everything downstream (choke gate, voice allocation, envelopes) behaves
identically to a MIDI instrument even though no MIDI is actually involved.

Chain: `netreceive -u -b <port>` → `oscparse` → `list trim` → `route fb1` →
`unpack f f` → convert sample index to note, forward velocity → `poly`.

- **Message format expected:** OSC address `/fb1`, two integer arguments:
  `<sample index> <velocity>`, e.g. `/fb1 8 100`.
  - **Sample index** is **1–15**, matching the existing sample numbering
    (1 = 01_C3 … 15 = 15_C5). Converted internally via `note = value + 59`.
  - **Velocity** is **0–127**, same range and meaning as MIDI velocity —
    it's forwarded straight into the same `poly`/ADSR amplitude scaling a
    real MIDI note-on would use, so it directly controls playback amplitude
    (`velocity/127` is the attack peak — see "Per-voice architecture" above).
  - `unpack f f`'s two outlets fire right-to-left, so velocity (outlet 1)
    is always set on `poly`'s velocity inlet *before* the note (outlet 0)
    triggers it — same "cold value before hot trigger" ordering used
    throughout this patch.
- **No note-off** is sent or expected — matches the controller's current
  behavior of firing note-on only. This already works correctly because of
  the long release time above: the sample just plays to completion.
- **`list trim` is required** before `route` — `oscparse` wraps its output
  in Pd's generic `list` selector, which `route` can't match an address
  symbol against directly. `list trim` strips that wrapper so `route fb1`
  sees `fb1` as the actual selector.
- Each cluster patch listens on its own port so all six patches can run
  simultaneously without conflicting:

  | Patch | Port |
  |---|---|
  | sampler.pd | 8000 |
  | cluster1.pd | 8001 |
  | cluster2.pd | 8002 |
  | cluster3.pd | 8003 |
  | cluster4.pd | 8004 |
  | cluster5.pd | 8005 |

  A cluster patch will simply ignore OSC values outside its own 3-sample
  range — the converted note number just won't match that voice's `select`
  list.

## TD activity ping (outgoing OSC)

These patches run alongside a TouchDesigner project with two relevant
sub-patches: an ambient background patch, and an idle-mode patch that should
start playing automatically whenever nothing has been triggered on any of
the Pd patches for 1 minute, and reset that 1-minute timer on any new
trigger. **Idle-mode's actual timer and trigger logic lives entirely on the
TD side** — Pd's only job is to notify TD that a trigger happened. Each of
the 6 patches sends its own ping independently; TD is responsible for
treating a ping from *any* of them as "not idle."

Chain (added once per patch, right off `route fb1` — so it fires on every
valid controller message, independent of whether the choke gate goes on to
accept or drop it): `route fb1` → `t b` → **[`1` → `oscformat /activity`] and
[`del 50` → `0` → the same `oscformat /activity`]** → `netsend -u -b`.

- **Destination**: each patch has a message box near the OSC-input objects
  reading `connect 192.168.1.100 9000` — **this is a placeholder IP**, since
  TD's machine address wasn't known yet when this was built. Edit it (Pd
  edit mode, double-click the message box, retype the IP) in **all 6
  patches** once you know TD's real address, then click the message box
  once (in run mode) to reconnect — it also fires automatically via
  `loadbang` on patch open, so after editing it correctly once and
  resaving, no manual click is needed on future opens. Port **9000** is
  fixed (doesn't collide with the 8000–8005 incoming-trigger ports).
- **Message sent**: OSC address `/activity`, sent **twice per trigger**:
  once with argument `1`, then again 50ms later with argument `0`. The
  value itself is a dummy — only its arrival matters — but the *pair* is
  required. On the TD side this is built to feed a Trigger CHOP that was
  originally designed around MIDI, where a note naturally returns to 0
  between hits; an OSC In CHOP just holds whatever value it last received
  indefinitely rather than decaying back down. Sending only `1` made the
  channel sit permanently "on" after the very first ping, so a
  rising-edge/threshold-based Trigger CHOP downstream never saw a second
  edge to fire on for any later ping. Sending `1` then `0` recreates a real
  on/off pulse, matching what a MIDI note-on/note-off pair would look like
  — confirmed working against a live TD Trigger CHOP + Speed CHOP idle
  timer built this way.
- **What the TD side needs to implement**: listen for OSC on UDP port 9000
  (all patches send there); on receiving `/activity` from *any* patch,
  (re)start a 60-second countdown; if it ever elapses without being reset,
  start idle-mode playback. TD will also need its own logic for what
  happens to idle-mode once activity resumes (e.g. stopping it again) —
  that's not something Pd signals for; only the "went idle" edge is
  Pd's concern here.
- `netsend`/`netreceive` (not `udpsend`/`udpreceive`) are the correct
  vanilla Pd object names for this — and both need the `-b` (binary) flag
  to carry real OSC bytes; without it, Pd's own FUDI text encoding gets
  sent instead and `oscparse` on the receiving end will reject it.

## Sample loading & arrays

On patch load, a `loadbang` fires a separate `read -resize <path> <arrayname>`
message per sample, each wired independently to `soundfiler`. (An earlier
version chained all reads through one comma-separated message box —
`soundfiler` silently fails on every read after the first when they're
chained that way, so each sample gets its own message box and its own
connection.) Each sample has a matching `[array]` declared in a subpatch,
shown as a small waveform graph in the patch.

## DSP autostart

A `loadbang` sends `; pd dsp 1` on patch load, so audio is enabled
automatically — you don't need to turn on DSP manually via Pd's Media menu
every time you open a patch.

## Computer-keyboard test triggers

`sampler.pd` (the full 15-sample patch only, not the clusters) has a
QWERTY-row test input: `Q W E R T Y U I O P` then `A S D F G` trigger the 15
samples low-to-high, with velocity fixed at 100, for testing without the OSC
controller connected. This feeds `poly` directly, same as the OSC path,
using the same 60–74 internal note range. It was
dropped from the cluster patches since it doesn't generalize cleanly to
arbitrary 3-note subsets.

## Known-fixed issues

Several non-obvious bugs were found and fixed during development; worth
knowing about if you're modifying the patch:

- **`pack`/`route` ordering** — see "Why poly's outlet order matters" above.
- **`unpack`'s right-to-left firing order** inside each voice: velocity
  arrives before note, but some logic needs the note stored *before* it's
  read back out. Fixed with explicit `t b f` sequencing rather than relying
  on incoming connection order.
- **`tabplay~` needs `set <name>` then a separate `bang`** — a bare array
  name sent to it is not a valid message.
- **Comma-in-text bug**: any unescaped comma inside a `.pd` file's object or
  message text (including plain comments) gets treated as a message
  separator by Pd's file parser, splitting the rest of the line into stray
  top-level commands and printing `canvas: no method for '...'` errors at
  load time. Text labels in this patch use `/` instead of `,` for this
  reason, and multi-message chains (like the old sample-loading message) are
  split into separate message boxes instead.
- **`$1` in a message box inside an argument-less abstraction** resolves
  against the abstraction's (nonexistent) creation argument, not the
  incoming message — it silently evaluates to 0 every time. The release
  message that used to read `0 $1` was replaced with a `pack`-based
  construction that doesn't rely on `$`-substitution.
