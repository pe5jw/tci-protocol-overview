# TCI Commands Reference — With Code Examples

Complete reference for all TCI commands. Each entry shows the canonical wire format, followed by implementation variants where they differ.

Code examples use Python with `websockets`, but the wire strings are identical in any language.

```python
# Connection setup (used in all examples below)
import asyncio
import websockets

async def connect():
    uri = "ws://localhost:50001"
    async with websockets.connect(uri) as ws:
        # wait for init block + ready
        while True:
            msg = await ws.recv()
            if msg == "ready":
                break
        # now send commands ...
```

---

## Table of Contents

1. [Connection & Init](#connection--init)
2. [Frequency / VFO](#frequency--vfo)
3. [Modulation & Filters](#modulation--filters)
4. [RIT / XIT](#rit--xit)
5. [DSP — Noise Reduction & Filtering](#dsp--noise-reduction--filtering)
6. [AGC & Squelch](#agc--squelch)
7. [PTT / TX](#ptt--tx)
8. [Audio Control](#audio-control)
9. [IQ & Audio Streaming](#iq--audio-streaming)
10. [Sensors](#sensors)
11. [TX Profiles](#tx-profiles)
12. [CAT Pass-through](#cat-pass-through)
13. [Keepalive](#keepalive)
14. [Implementation Variant Summary](#implementation-variant-summary)

---

## Connection & Init

### Handshake — server sends first

After connecting, the server immediately sends an init block followed by `ready`.

```
protocol:2;
device:ExpertSDR2;
receive_only:false;
trx_count:1;
channels_count:2;
vfo_limits:100000,450000000;
if_limits:-96000,96000;
modulations_list:AM,SAM,DSB,LSB,USB,CWL,CWU,NFM,DIGL,DIGU,SPEC,DRM;
iq_samplerate:192000;
audio_samplerate:48000;
tx_profiles_ex:Default,Contest,DX;
ready
```

```python
async def read_init(ws):
    """Read and parse the server init block."""
    params = {}
    async for msg in ws:
        if msg == "ready":
            break
        if ":" in msg:
            key, _, val = msg.rstrip(";").partition(":")
            params[key] = val
    return params

# Example result:
# {
#   "device": "ExpertSDR2",
#   "trx_count": "1",
#   "iq_samplerate": "192000",
#   ...
# }
```

### `start` / `stop`

Client sends `start` to begin streaming; `stop` to end.

```
start
stop
```

```python
await ws.send("start")
await ws.send("stop")
```

**Variants:**
- Thetis: `start` / `stop` also toggle radio power (hardware on/off)
- All other implementations: streaming control only

### `ready`

Server-only. Marks the end of the init block. No colon, no semicolon.

---

## Frequency / VFO

### `vfo` — Set VFO frequency

```
vfo:trx,channel,freq_hz;
```

| Param | Type | Description |
|---|---|---|
| `trx` | int | Transceiver index (0-based) |
| `channel` | int | RX channel (0 = VFO A, 1 = VFO B) |
| `freq_hz` | int | Frequency in Hz |

```python
# Set TRX 0, channel 0 (VFO A) to 14.074 MHz
await ws.send("vfo:0,0,14074000;")

# Query current frequency (omit value)
await ws.send("vfo:0,0;")
# Server replies: vfo:0,0,14074000;
```

No known variants — all implementations use identical syntax.

---

### `dds` — Set DDS (center) frequency

```
dds:trx,freq_hz;
```

Sets the center frequency of the spectrum display. The VFO offset from center is preserved.

```python
# Center spectrum on 14.1 MHz
await ws.send("dds:0,14100000;")

# Query
await ws.send("dds:0;")
```

---

### `if` — Set IF frequency

```
if:trx,channel,freq_hz;
```

Sets the intermediate frequency offset within the passband.

```python
# Set IF offset
await ws.send("if:0,0,3000;")

# Query
await ws.send("if:0,0;")
```

---

### `vfo_lock` — Lock individual VFO

```
vfo_lock:trx,channel,true|false;
```

```python
await ws.send("vfo_lock:0,0,true;")   # lock VFO A
await ws.send("vfo_lock:0,0,false;")  # unlock VFO A
```

---

### `lock` — Lock all VFOs on TRX

```
lock:trx,true|false;
```

```python
await ws.send("lock:0,true;")
```

---

### `split_enable` — Enable split operation

```
split_enable:trx,true|false;
```

```python
await ws.send("split_enable:0,true;")   # TX on VFO B freq
await ws.send("split_enable:0,false;")  # TX on VFO A freq

# Query
await ws.send("split_enable:0;")
```

**Variants:**
- Thetis / Kainos: uses numeric `0` for query instead of omitting the value:
  ```python
  # Thetis query form
  await ws.send("split_enable:0,0;")
  ```

---

### `vfo_swap_ex` — Swap VFO A/B

```
vfo_swap_ex:trx;
```

```python
await ws.send("vfo_swap_ex:0;")
```

---

### `rx_enable` — Enable/disable RX channel

```
rx_enable:trx,channel,true|false;
```

```python
await ws.send("rx_enable:0,1,true;")   # enable second RX channel
await ws.send("rx_enable:0,1,false;")  # disable second RX channel
```

---

## Modulation & Filters

### `modulation` — Set modulation mode

```
modulation:trx,mode;
```

Standard mode values: `USB`, `LSB`, `AM`, `SAM`, `DSB`, `CWL`, `CWU`, `NFM`, `DIGL`, `DIGU`, `SPEC`, `DRM`

```python
await ws.send("modulation:0,USB;")
await ws.send("modulation:0,CWU;")
await ws.send("modulation:0,NFM;")

# Query
await ws.send("modulation:0;")
# Server replies: modulation:0,USB;
```

**Variants:**

| Implementation | CW modes | FM mode | Notes |
|---|---|---|---|
| ExpertSDR / Thetis / NereusSDR / Zeus | `CWL`, `CWU` | `NFM` | Standard |
| deskHPSDR | `CW` (only) | `FM` | Collapses CWL/CWU → CW, NFM → FM |
| Zeus | `CWL`, `CWU` | accepts `NFM` and `FM` | Case-insensitive input, uppercase output |

```python
# Universal modulation handler — normalize on receive
def normalize_mode(mode: str) -> str:
    mode = mode.upper()
    if mode == "CW":
        return "CWU"       # assume upper sideband
    if mode == "FM":
        return "NFM"
    return mode

# Send: always use CWU — accepted everywhere
await ws.send("modulation:0,CWU;")
```

---

### `rx_filter_band` — Set RX passband filter

```
rx_filter_band:trx,low_hz,high_hz;
```

Filter edges relative to carrier frequency in Hz.

```python
# USB: 300–2700 Hz
await ws.send("rx_filter_band:0,300,2700;")

# CW narrow: -250 to +250 Hz
await ws.send("rx_filter_band:0,-250,250;")

# AM wide: -4000 to +4000 Hz
await ws.send("rx_filter_band:0,-4000,4000;")

# Query
await ws.send("rx_filter_band:0;")
```

No known variants.

---

### `tx_filter_band_ex` — Set TX passband filter

```
tx_filter_band_ex:trx,low_hz,high_hz;
```

```python
await ws.send("tx_filter_band_ex:0,100,2900;")
```

---

## RIT / XIT

### `rit_enable` — Enable RIT

```
rit_enable:trx,true|false;
```

```python
await ws.send("rit_enable:0,true;")
await ws.send("rit_enable:0,false;")
```

---

### `rit_offset` — Set RIT offset

```
rit_offset:trx,offset_hz;
```

```python
await ws.send("rit_offset:0,400;")    # +400 Hz
await ws.send("rit_offset:0,-200;")   # -200 Hz
await ws.send("rit_offset:0,0;")      # clear
```

---

### `xit_enable` — Enable XIT

```
xit_enable:trx,true|false;
```

```python
await ws.send("xit_enable:0,true;")
```

---

### `xit_offset` — Set XIT offset

```
xit_offset:trx,offset_hz;
```

```python
await ws.send("xit_offset:0,600;")
```

---

## DSP — Noise Reduction & Filtering

### `rx_nb_enable` — Noise Blanker 1

```
rx_nb_enable:trx,channel,true|false;
```

```python
await ws.send("rx_nb_enable:0,0,true;")
```

**Variants:**
- Zeus: uses `rx_nb_enable` (standard)
- deskHPSDR: supports indexed form via `rx_nb_enable_ex`

---

### `rx_nb2_enable` — Noise Blanker 2

```
rx_nb2_enable:trx,channel,true|false;
```

```python
await ws.send("rx_nb2_enable:0,0,true;")
```

---

### `rx_nb_enable_ex` — Extended Noise Blanker (indexed)

```
rx_nb_enable_ex:trx,channel,nb_idx,true|false;
```

```python
await ws.send("rx_nb_enable_ex:0,0,0,true;")   # NB1
await ws.send("rx_nb_enable_ex:0,0,1,true;")   # NB2
```

**Variants:**
- ExpertSDR / deskHPSDR / NereusSDR: supported
- Thetis: not documented; use `rx_nb_enable` / `rx_nb2_enable`

---

### `rx_nr_enable` — Noise Reduction

```
rx_nr_enable:trx,channel,true|false;
```

```python
await ws.send("rx_nr_enable:0,0,true;")
```

---

### `rx_nr_enable_ex` — Extended Noise Reduction (indexed)

```
rx_nr_enable_ex:trx,channel,nr_idx,true|false;
```

`nr_idx` values: `0`–`4` (0 = NR1, 1 = NR2, …)

```python
await ws.send("rx_nr_enable_ex:0,0,0,true;")   # NR1 on
await ws.send("rx_nr_enable_ex:0,0,1,true;")   # NR2 on
```

**Variants:**

| Implementation | NR labels | Notes |
|---|---|---|
| ExpertSDR | `0`–`4` | Indexed |
| deskHPSDR | NR1, NR2, NR3, NR4, NNR | Labeled differently in UI; still indexed on wire |
| NereusSDR | WDSP NR1–NR4 | Full WDSP support |
| SDROxide | 5 NR engines | Client-side |

---

### `rx_anf_enable` — Auto Notch Filter

```
rx_anf_enable:trx,channel,true|false;
```

```python
await ws.send("rx_anf_enable:0,0,true;")
```

---

### `rx_bin_enable` — Binaural Audio

```
rx_bin_enable:trx,channel,true|false;
```

```python
await ws.send("rx_bin_enable:0,0,true;")
```

---

## AGC & Squelch

### `agc_mode` — AGC mode

```
agc_mode:trx,channel,mode;
```

This is the command used by all open-source implementations. ExpertSDR3 uses `agc_auto_ex` instead (see variant below).

```python
await ws.send("agc_mode:0,0,fast;")
await ws.send("agc_mode:0,0,normal;")
await ws.send("agc_mode:0,0,off;")
```

**Mode values by implementation:**

| Mode | ExpertSDR | deskHPSDR | Thetis / NereusSDR | Zeus |
|---|---|---|---|---|
| Auto on | `agc_auto_ex:0,0,true;` | `normal` | `normal` | `MEDIUM` |
| Auto off | `agc_auto_ex:0,0,false;` | `off` | `off` | `FIXED` |
| Fast | — | `fast` | `fast` | `FAST` |
| Slow | — | — | — | `SLOW` |
| Medium | — | — | `med` | `MEDIUM` |
| Custom | — | — | — | `CUSTOM` |

```python
# Universal AGC handler
def send_agc(ws, trx, channel, mode):
    """mode: 'off', 'fast', 'normal'"""
    # All open-source implementations
    return f"agc_mode:{trx},{channel},{mode};"

# deskHPSDR / Thetis / NereusSDR / Zeus
await ws.send(send_agc(0, 0, "fast"))

# ExpertSDR3 only
await ws.send("agc_auto_ex:0,0,true;")   # auto on
await ws.send("agc_auto_ex:0,0,false;")  # auto off (manual)
```

---

### `agc_auto_ex` — AGC auto mode (ExpertSDR3)

```
agc_auto_ex:trx,channel,true|false;
```

```python
# ExpertSDR3 reference implementation only
await ws.send("agc_auto_ex:0,0,true;")
await ws.send("agc_auto_ex:0,0,false;")
```

**Note:** All open-source servers (deskHPSDR, Thetis, NereusSDR, Zeus) use `agc_mode` instead.

---

### `agc_gain` — Manual AGC gain

```
agc_gain:trx,channel,gain_db;
```

Only effective when AGC is in manual/off mode.

```python
await ws.send("agc_gain:0,0,60;")    # 60 dB gain
await ws.send("agc_gain:0,0,-10;")   # low gain
```

---

### `sql_enable` — Squelch enable

```
sql_enable:trx,channel,true|false;
```

```python
await ws.send("sql_enable:0,0,true;")
await ws.send("sql_enable:0,0,false;")
```

---

### `sql_level` — Squelch threshold

```
sql_level:trx,channel,level_db;
```

```python
await ws.send("sql_level:0,0,-80;")   # open at -80 dBm
await ws.send("sql_level:0,0,-60;")   # tighter squelch
```

---

## PTT / TX

### `trx` — PTT (TX/RX toggle)

```
trx:trx,true|false;
```

```python
await ws.send("trx:0,true;")    # PTT on (TX)
await ws.send("trx:0,false;")   # PTT off (RX)

# Query
await ws.send("trx:0;")
```

**Variants:**

Zeus supports an optional third argument `signal_source`:

```
trx:trx,true|false,signal_source;
```

Valid `signal_source` values: `tci`, `mic1`, `mic2`, `micPC`, `ecoder2`

```python
# Zeus: TX with audio via WebSocket (TCI audio source)
await ws.send("trx:0,true,tci;")

# deskHPSDR: also supports 3-arg form
await ws.send("trx:0,true,tci;")
```

**PTT broadcast behavior:**

| Implementation | PTT echoed to all clients? |
|---|---|
| ExpertSDR / deskHPSDR / Zeus | ✓ broadcast to all |
| Thetis / NereusSDR | ✗ reply to requesting client only |

```python
# Safe approach for multi-client: actively query PTT state
async def get_ptt_state(ws, trx=0):
    await ws.send(f"trx:{trx};")
    async for msg in ws:
        if msg.startswith(f"trx:{trx},"):
            return msg.rstrip(";").split(",")[1] == "true"
```

---

### `tune` — Carrier tune

```
tune:trx,true|false;
```

Sends a carrier for antenna tuner adjustment.

```python
await ws.send("tune:0,true;")
await ws.send("tune:0,false;")
```

---

### `drive` — TX drive level

```
drive:trx,level_pct;
```

Range: 0–100 (percent of maximum power).

```python
await ws.send("drive:0,50;")    # 50% power
await ws.send("drive:0,100;")   # full power
```

---

### `tune_drive` — Tune drive level

```
tune_drive:trx,level_pct;
```

Separate drive level used during `tune` mode.

```python
await ws.send("tune_drive:0,25;")   # 25% power during tune
```

---

### `tx_antenna` — Select TX antenna

```
tx_antenna:trx,ant_idx;
```

```python
await ws.send("tx_antenna:0,0;")   # antenna 1
await ws.send("tx_antenna:0,1;")   # antenna 2
```

**Variants:**
- ExpertSDR / deskHPSDR / Zeus: direct command
- Thetis / Kainos: uses CAT pass-through with `ZZOC`:
  ```python
  await ws.send("run_cat_ex:0,ZZOC01;")   # antenna index 01
  ```

---

### `rx_antenna` — Select RX antenna

```
rx_antenna:trx,channel,ant_idx;
```

```python
await ws.send("rx_antenna:0,0,2;")   # RX antenna 3 on channel 0
```

**Variants:**
- ExpertSDR / deskHPSDR / Zeus: direct command
- Thetis / Kainos: uses CAT pass-through with `ZZOA`:
  ```python
  await ws.send("run_cat_ex:0,ZZOA02;")   # RX antenna index 02
  ```

---

## Audio Control

### `rx_volume` — RX audio volume

```
rx_volume:trx,channel,volume_db;
```

```python
await ws.send("rx_volume:0,0,-6;")    # -6 dB
await ws.send("rx_volume:0,0,0;")     # unity gain
await ws.send("rx_volume:0,0,-40;")   # near mute
```

---

### `mute` — Mute audio

```
mute:trx,true|false;
```

```python
await ws.send("mute:0,true;")
await ws.send("mute:0,false;")
```

---

### `mon_enable` — TX monitor enable

```
mon_enable:trx,true|false;
```

Enables sidetone / TX audio monitor in headphones.

```python
await ws.send("mon_enable:0,true;")
```

---

### `mon_volume` — TX monitor volume

```
mon_volume:trx,volume_db;
```

```python
await ws.send("mon_volume:0,-10;")
```

---

### `line_in` — Line-in level

```
line_in:trx,level_db;
```

```python
await ws.send("line_in:0,-6;")
```

---

## IQ & Audio Streaming

### `iq_start` / `iq_stop` — Start/stop IQ stream

```
iq_start:trx;
iq_stop:trx;
```

```python
await ws.send("iq_start:0;")
# server now sends binary IQ frames
await ws.send("iq_stop:0;")
```

**Variants:**

Zeus also supports a 2-argument form:

```
iq_start:trx,bool;
```

```python
# Zeus extended form
await ws.send("iq_start:0,true;")
await ws.send("iq_start:0,false;")   # equivalent to iq_stop
```

**IQ ownership model:**

| Implementation | Model |
|---|---|
| deskHPSDR | **Exclusive** — first client locks the sample rate for all; `iq_stop` from that client stops all IQ |
| Thetis / NereusSDR | Stub — `iq_start` accepted but IQ data not streamed |
| ExpertSDR / Zeus | Per-session subscription |

```python
# Safe approach for deskHPSDR: set rate before iq_start
await ws.send("iq_samplerate:192000;")
await ws.send("iq_start:0;")
```

---

### `audio_start` / `audio_stop` — Start/stop RX audio stream

```
audio_start:trx;          ← standard (1 argument)
audio_stop:trx;
```

```python
await ws.send("audio_start:0;")
# server now sends binary audio frames
await ws.send("audio_stop:0;")
```

**Variants:**

ExpertSDR3 reference spec uses 2 arguments `audio_start:trx,channel;`. All open-source implementations use 1 argument. **Always use 1 argument.**

```python
# ExpertSDR3 reference spec only (avoid — breaks other servers):
await ws.send("audio_start:0,0;")

# All others + safe universal form:
await ws.send("audio_start:0;")
```

| Implementation | `audio_start` args | Behavior |
|---|---|---|
| ExpertSDR3 | 2 (`trx,ch`) | Reference spec |
| deskHPSDR | 1 (`trx`) | Rejects 2 args |
| Thetis / NereusSDR | 1 (`trx`) | Stub — accepts but no audio |
| Zeus | 1 (`trx`) | Active |

---

### `audio_stream_sample_type` — Audio sample format

```
audio_stream_sample_type:trx,float32|int16;
```

```python
await ws.send("audio_stream_sample_type:0,float32;")
await ws.send("audio_stream_sample_type:0,int16;")
```

---

### `audio_stream_channels` — Mono or stereo

```
audio_stream_channels:trx,1|2;
```

```python
await ws.send("audio_stream_channels:0,2;")   # stereo
await ws.send("audio_stream_channels:0,1;")   # mono
```

---

### `audio_stream_samples` — Samples per frame

```
audio_stream_samples:trx,count;
```

```python
await ws.send("audio_stream_samples:0,480;")    # 10 ms @ 48 kHz
await ws.send("audio_stream_samples:0,1920;")   # 40 ms @ 48 kHz
```

---

### `audio_samplerate` — Audio sample rate

```
audio_samplerate:trx,rate_hz;
```

```python
await ws.send("audio_samplerate:0,48000;")
```

NereusSDR supports multiple rates via WDSP resampling: 8000, 12000, 16000, 44100, 48000 Hz.

---

### `tx_stream_audio_buffering` — TX audio buffering

```
tx_stream_audio_buffering:trx,true|false;
```

```python
await ws.send("tx_stream_audio_buffering:0,true;")
```

---

### Reading binary frames (Python example)

```python
import struct

FRAME_TYPE_IQ    = 0x00
FRAME_TYPE_RX_AUDIO = 0x01
FRAME_TYPE_TX_AUDIO = 0x02

async def read_binary_frames(ws):
    async for msg in ws:
        if isinstance(msg, bytes):
            # 64-byte header
            header = msg[:64]
            payload = msg[64:]

            flags        = header[0]
            sample_rate  = int.from_bytes(header[1:4], "little")
            sample_count = int.from_bytes(header[12:16], "little")
            frame_type   = header[16]

            if frame_type == FRAME_TYPE_IQ:
                # interleaved float32: I0, Q0, I1, Q1, ...
                n = len(payload) // 4
                samples = struct.unpack(f"<{n}f", payload)
                i_samples = samples[0::2]
                q_samples = samples[1::2]

            elif frame_type == FRAME_TYPE_RX_AUDIO:
                # float32 stereo PCM
                n = len(payload) // 4
                samples = struct.unpack(f"<{n}f", payload)
                left  = samples[0::2]
                right = samples[1::2]
```

---

## Sensors

Sensor data flows **server → client** as periodic broadcasts when enabled.

### `rx_sensors_enable` / `tx_sensors_enable` — Enable sensor streams

```
rx_sensors_enable:trx,true|false;
tx_sensors_enable:trx,true|false;
```

```python
await ws.send("rx_sensors_enable:0,true;")
await ws.send("tx_sensors_enable:0,true;")
```

---

### `rx_smeter` — S-meter reading

Server sends periodically when `rx_sensors_enable` is active.

```
rx_smeter:trx,channel,level_dbm;
```

```python
# Receive loop
async for msg in ws:
    if msg.startswith("rx_smeter:"):
        parts = msg.rstrip(";").split(":")
        trx, ch, dbm = parts[1].split(",")
        print(f"S-meter TRX{trx} ch{ch}: {dbm} dBm")
```

**Aliases accepted by some implementations:**
- `s_meter` → `rx_smeter` (deskHPSDR)
- `smeter` → `rx_smeter` (older implementations)

```python
def parse_smeter(msg: str) -> float | None:
    for prefix in ("rx_smeter:", "s_meter:", "smeter:"):
        if msg.startswith(prefix):
            return float(msg.rstrip(";").split(",")[-1])
    return None
```

---

### `tx_sensors` — TX sensor readout

```
tx_sensors:trx,mic_db,fwd_pwr,rev_pwr,swr;
```

```python
async for msg in ws:
    if msg.startswith("tx_sensors:"):
        _, vals = msg.rstrip(";").split(":")
        trx, mic, fwd, rev, swr = vals.split(",")
```

---

### `tx_forward_power` — TX forward power

```
tx_forward_power:trx,watts;
```

Alias: `tx_power` (some implementations).

```python
async for msg in ws:
    if msg.startswith("tx_forward_power:") or msg.startswith("tx_power:"):
        watts = float(msg.rstrip(";").split(",")[-1])
```

---

### `tx_swr` — SWR reading

```
tx_swr:trx,swr;
```

Alias: `swr` (some implementations).

```python
async for msg in ws:
    if msg.startswith("tx_swr:") or msg.startswith("swr:"):
        swr = float(msg.rstrip(";").split(",")[-1])
```

---

## TX Profiles

### `tx_profiles_ex` — Profile list (init only)

Sent by server during init. Read-only; lists available TX profile names.

```
tx_profiles_ex:Default,Contest,DX;
```

```python
# Parse during init
if key == "tx_profiles_ex":
    profiles = val.split(",")
    # ['Default', 'Contest', 'DX']
```

---

### `tx_profile_ex` — Select TX profile

```
tx_profile_ex:trx,profile_name;
```

```python
await ws.send("tx_profile_ex:0,Contest;")
await ws.send("tx_profile_ex:0,Default;")
```

---

## CAT Pass-through

### `run_cat_ex` — Send CAT command to radio

```
run_cat_ex:trx,cat_string;
```

Passes a raw CAT command string to the underlying hardware.

```python
await ws.send("run_cat_ex:0,FA014074000;")   # Kenwood FA (VFO A freq)
```

**Thetis / Kainos — ZZ dialect:**

Thetis uses extended Kenwood `ZZ` commands for functions not available in standard TCI:

```python
# Antenna selection
await ws.send("run_cat_ex:0,ZZOA00;")   # RX antenna 0
await ws.send("run_cat_ex:0,ZZOC00;")   # TX antenna 0

# Squelch source
await ws.send("run_cat_ex:0,ZZSO0;")

# Query
await ws.send("run_cat_ex:0,ZZOA;")
# Server replies: run_cat_ex:0,ZZOA00;
```

---

## Keepalive

### `keepalive` — Keep connection alive

```
keepalive;
```

No reply from server. Send periodically to prevent connection timeout.

```python
import asyncio

async def keepalive_loop(ws, interval=30):
    while True:
        await asyncio.sleep(interval)
        await ws.send("keepalive;")
```

---

## Implementation Variant Summary

Quick reference: which implementations differ from the standard.

| Command | Standard | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR | Zeus |
|---|---|---|---|---|---|---|
| `audio_start` | `trx` (1 arg) | `trx,ch` (2 args) | `trx` (rejects 2) | `trx` (stub) | `trx` (stub) | `trx` |
| `agc_mode` | `agc_mode:trx,ch,mode;` | `agc_auto_ex:trx,ch,bool;` | `agc_mode` | `agc_mode` | `agc_mode` | `agc_mode` |
| CW mode | `CWL` / `CWU` | `CWL` / `CWU` | `CW` only | `CWL` / `CWU` | `CWL` / `CWU` | `CWL` / `CWU` |
| FM mode | `NFM` | `NFM` | `FM` only | `NFM` | `NFM` | `FM` (alias `NFM`) |
| PTT broadcast | broadcast all | broadcast all | broadcast all | requesting client only | requesting client only | broadcast all |
| IQ ownership | per-session | per-session | exclusive | stub | stub | per-session |
| `trx` 3rd arg | — | — | `tci` | — | — | `tci,mic1,mic2,…` |
| Antenna select | `tx_antenna` / `rx_antenna` | direct | direct | ZZ CAT | ? | direct |
| S-meter | `rx_smeter` | `rx_smeter` | `s_meter` alias | `rx_smeter` | `rx_smeter` | `rx_smeter` |

---

### Universal client — minimal safe command set

```python
async def universal_tci_client(host="localhost", port=50001):
    uri = f"ws://{host}:{port}"
    async with websockets.connect(uri) as ws:

        # 1. Read init block
        params = {}
        async for msg in ws:
            if msg == "ready":
                break
            if ":" in msg:
                key, _, val = msg.rstrip(";").partition(":")
                params[key] = val

        device = params.get("device", "unknown")
        print(f"Connected to: {device}")

        # 2. Detect implementation
        is_deskhpsdr = "deskhpsdr" in device.lower()
        is_thetis    = device == "ExpertSDR2"  # Thetis masquerades
        is_zeus      = device == "Zeus"
        is_expert    = "expertsd" in device.lower() and not is_thetis

        # 3. Start streams — 1 arg, safe for all
        await ws.send("audio_start:0;")

        # Only start IQ if deskHPSDR — set rate first (exclusive ownership)
        if is_deskhpsdr:
            await ws.send("iq_samplerate:192000;")
        await ws.send("iq_start:0;")

        # 4. Set mode — CWU works everywhere
        await ws.send("modulation:0,CWU;")

        # 5. AGC — agc_mode works everywhere except ExpertSDR3
        if is_expert:
            await ws.send("agc_auto_ex:0,0,true;")
        else:
            await ws.send("agc_mode:0,0,fast;")

        # 6. PTT — 2-arg form works everywhere
        await ws.send("trx:0,true;")   # TX on
        await ws.send("trx:0,false;")  # TX off

        # 7. Poll PTT in Thetis (no broadcast)
        if is_thetis:
            await ws.send("trx:0;")
```
