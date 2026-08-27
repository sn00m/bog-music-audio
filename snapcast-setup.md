# Running the Pd Patches Alongside a Snapcast Client

How to make the Pure Data patches in this repo and a `snapclient` synced
stream play **at the same time** on a Raspberry Pi, out of the same speakers.

## The problem

On a fresh Raspberry Pi OS install the Pd patches and Snapcast will not play
together. The usual symptom: `snapclient` plays fine on its own, then the
moment a Pd patch opens its audio device, the Snapcast stream goes silent.
Closing Pd brings Snapcast back.

This is the classic Linux **exclusive-ALSA** collision. Whichever process
opens the raw sound card first (`hw:0`) locks it, and the second process to
ask for it gets silence (or a "Device or resource busy" error). Neither app
wins permanently — they simply cannot both hold a bare hardware device.

The fix is to make **both** programs go through the software mixing layer
instead of touching `hw:` directly.

## The mixing layer: PipeWire

Current Raspberry Pi OS (Bookworm) ships **PipeWire** as the audio server,
and PipeWire *is* the mixer. It accepts an unlimited number of simultaneous
clients and mixes them automatically.

**There is nothing to configure on the mixing layer itself.** The entire job
is making sure neither `snapclient` nor Pd bypasses PipeWire by opening
`hw:0` directly. On a machine where this is broken, one of them — almost
always Pd, via its Audio Settings dialog — is still pointed at the raw card.

### Check what you have

```bash
pactl info 2>/dev/null | grep 'Server Name'   # expect "PipeWire"
ps aux | grep -E 'pipewire|pulseaudio' | grep -v grep
aplay -L | grep -E '^(default|pulse|pipewire|dmix)'
cat /etc/default/snapclient                    # how snapclient opens audio
pd -listdev                                     # what Pd sees / is using
```

If `Server Name` comes back as `PulseAudio` (older Bullseye images) the same
steps below still apply — the `pulse` player and the `default` ALSA device
both route into PulseAudio the same way they route into PipeWire.

## Step 1 — snapclient into PipeWire

Edit `/etc/default/snapclient`:

```bash
SNAPCLIENT_OPTS="--player pulse"
```

```bash
sudo systemctl restart snapclient
```

`pulse` here means PipeWire's built-in PulseAudio-compatibility player. It is
the correct choice on Bookworm — there is no separate `pipewire` player
name. Do **not** use `--player alsa --soundcard hw:0`, which is what grabs
the card exclusively.

## Step 2 — Pd into PipeWire

Make sure the ALSA-to-PipeWire bridge is installed (it usually already is on
Bookworm):

```bash
apt policy pipewire-alsa      # should show "Installed:" with a version, not "none"
sudo apt install pipewire-alsa
```

Then, in Pd's **Media ▸ Audio Settings**:

- Set **Output Device** to `default` (it may instead be listed as `pipewire`
  or `pulse`).
- Do **not** pick anything ending in `(HW)` or shown as `hw:0`.

With `pipewire-alsa` present, the ALSA `default` device *is* PipeWire, so Pd
just becomes another client and mixes with everything else.

If you launch Pd from a script or terminal instead of the GUI:

```bash
pd -alsa -r 48000 -audiobuf 60 audio/sampler.pd
```

Run `pd -listdev` first. If `-alsa` on its own still grabs the hardware, add
`-audiooutdev N` with the index `-listdev` reports for `default`.

Because these patches call `; pd dsp 1` from `loadbang` (see the README's
"DSP autostart" section), Pd opens its audio device immediately on patch
load — so the device setting has to be correct *before* you open a patch,
not after.

## Step 3 — verify both are connected

With `snapclient` playing and a Pd patch open:

```bash
wpctl status
```

Under **Audio ▸ Streams** you should see **both**:

- `Snapcast` (or `snapclient`)
- `pd` / `Pure Data` (may appear as `ALSA plug-in [pd]`)

listed as playback streams on the same sink. `pw-top` will likewise show two
active output streams. If both are there, they are mixing — you are done.

## If it still cuts out

- **Sample rate.** Snapcast streams at 48 kHz by default. Run Pd at 48000 as
  well (`-r 48000`, or the Sample Rate field in Audio Settings) so PipeWire
  is not forced to resample one stream under load. All the samples in this
  repo are plain `.wav` files and will resample cleanly to whatever rate Pd
  runs at, so matching Pd to 48 kHz has no downside here.
- **Pd's device dropdown shows only `hw:` entries.** `pipewire-alsa` is not
  active. Reinstall it and reboot, or run Pd as `pw-jack pd ...` after
  `sudo apt install pipewire-jack`.
- **`wpctl status` shows only one stream.** The missing app is still on
  `hw:`. Recheck that app's device setting — `/etc/default/snapclient` for
  Snapcast, Audio Settings for Pd.
- **Multiple cluster patches at once.** Running `cluster1.pd` … `cluster5.pd`
  as separate Pd instances (as the README describes) means five Pd clients
  into PipeWire plus Snapcast. That is fine — PipeWire mixes them all — but
  every one of those instances must be set to `default`, not `hw:0`, or the
  first one to load will lock the card against the rest.

## The one rule

Nothing in the chain may reference `hw:0` directly — not `snapclient`, not
Pd, not any `~/.asoundrc`. Everything goes through `default` / `pulse` /
`pipewire`, and PipeWire does the mixing.
