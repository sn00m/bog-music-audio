# Streaming `soundscape.pd` from the Mac mini (Snapcast source)

The Mac mini runs `soundscape.pd` (ambient bed + idle mode) and is the
**source** for the Snapcast stream. Its audio has to get into `snapserver`,
which then streams to every Pi. The Pi side — `snapclient` mixing with the
local `clusterN.pd` — is in [`snapcast-setup.md`](snapcast-setup.md).

`snapserver` on macOS can't capture a sound device directly, so the path is:

```
soundscape.pd ──► BlackHole (virtual device) ──► sox ──► /tmp/snapfifo ──► snapserver ──► network
```

Everything runs at **48 kHz stereo** end to end — the soundscape WAVs are
48 kHz, so nothing should resample.

## Step 1 — virtual audio device (BlackHole)

```bash
brew install blackhole-2ch
```

This adds a CoreAudio device called **BlackHole 2ch** (2 in / 2 out). Log
out/in or reboot if it doesn't show up. (Loopback.app works identically if
you already own it — just use its device name in place of `BlackHole 2ch`
everywhere below.)

## Step 2 — point `soundscape.pd`'s output at BlackHole

In Pd: **Media ▸ Audio Settings**

- **Output Device:** `BlackHole 2ch`
- **Sample rate:** `48000`
- **Channels:** `2`

`soundscape.pd`'s `dac~ 1 2` now feeds BlackHole instead of speakers. If you
launch Pd from the terminal, `-r 48000` and pick the BlackHole device index
from `pd -listdev`.

**Monitoring on the Mac (optional).** If you want to hear the soundscape on
the Mac mini too, open *Audio MIDI Setup*, create a **Multi-Output Device**
containing both `BlackHole 2ch` and `Built-in Output`, and set Pd's output
to that instead. The Pis don't need this.

## Step 3 — capture BlackHole into a pipe

```bash
mkfifo /tmp/snapfifo
```

Confirm this `sox` build has CoreAudio support:

```bash
sox --help 2>&1 | grep -i coreaudio
```

It should list `coreaudio` under the audio device drivers. If it prints
nothing, the Homebrew build was made without it — install `sox` from
MacPorts, or build it with CoreAudio enabled.

Create `feed-snapcast.sh` (the device name is exactly what Audio MIDI Setup
shows for BlackHole):

```sh
#!/bin/sh
DEV="BlackHole 2ch"
while true; do
  sox -q -t coreaudio "$DEV" \
    -t raw -b 16 -e signed-integer -c 2 -r 48000 -L - > /tmp/snapfifo
  echo "sox exited — restarting in 1s" >&2
  sleep 1
done
```

```bash
chmod +x feed-snapcast.sh
```

`-L` forces little-endian output to match snapserver's `sampleformat`. Set
`BlackHole 2ch` to **48 kHz** in *Audio MIDI Setup* so sox isn't resampling.
The `while` loop keeps the feed alive if sox ever drops; for a permanent
install wrap it in a `launchd` LaunchAgent so it starts at login.

## Step 4 — snapserver

```bash
brew install snapcast
```

Edit the snapserver config (`/opt/homebrew/etc/snapserver.conf` on Apple
Silicon, `/usr/local/etc/snapserver.conf` on Intel):

```ini
[stream]
source = pipe:///tmp/snapfifo?name=Soundscape&sampleformat=48000:16:2&codec=flac
```

Start everything (order doesn't strictly matter — snapserver retries the
pipe):

```bash
brew services start snapcast     # runs snapserver
./feed-snapcast.sh &             # or via launchd
```

Then open `soundscape.pd` in Pd.

### Alternative: let snapserver run sox itself

Instead of the pipe + script, snapserver can supervise the capture with a
`process` source:

```ini
source = process:///opt/homebrew/bin/sox?name=Soundscape&sampleformat=48000:16:2&params=-q -t coreaudio BlackHole%202ch -t raw -b 16 -e signed-integer -c 2 -r 48000 -L -
```

snapserver restarts sox if it dies. The `%20` is the space in the device
name.

## Verify

- `journalctl`-style logs: `brew services info snapcast`, or run
  `snapserver` in the foreground to watch it.
- On a Pi, `snapclient` should now play the soundscape. `wpctl status` on
  the Pi shows the `Snapcast` stream (and, once a cluster patch is open,
  the `pd` stream alongside it — see [`snapcast-setup.md`](snapcast-setup.md)).
- The Snapcast web UI (`http://<mac-mini>:1780`) shows the `Soundscape`
  stream as **playing** once sox is feeding the pipe.

## Notes

- **Latency.** Snapcast buffers ~1 s so all Pis play in lockstep. The
  soundscape is therefore ~1 s behind real time. That's irrelevant here —
  the ambient bed and idle mode aren't synced to anything, and the sampler
  hits are triggered independently over OSC on each Pi.
- **Silence handling.** sox keeps writing samples even when `soundscape.pd`
  is producing near-silence between the ambient bed and an idle fade, so the
  pipe never runs dry and snapserver won't drop the stream. (The ambient bed
  always plays, so there's real signal anyway.)
- **Sample rate mismatches** are the usual cause of clicks or pitch errors:
  check Pd, the `BlackHole 2ch` rate in Audio MIDI Setup, sox's `-r`, the
  `sampleformat`, and the snapclients are all 48000.
- **DSP must be on.** `soundscape.pd` sends `; pd dsp 1` from `loadbang`, so
  this is automatic on patch open — but if you see a dead stream, confirm
  Pd's DSP is running and its meters move.
