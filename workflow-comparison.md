# TCI Workflow Comparison — Implementation Differences

This document focuses on **how** key operations work in each TCI implementation, not just whether they exist. The biggest deviations from the reference (ExpertSDR3) are highlighted.

Implementations covered:
- **ExpertSDR3** — reference / origin of the protocol
- **deskHPSDR** (dl1bz) — C implementation for ANAN/Hermes Lite 2
- **Thetis** (ramdor / mi0bot) — C# for ANAN hardware (OpenHPSDR)
- **NereusSDR** (boydsoftprez) — C++ SDR software
- **Lyra-SDR-cpp** (N8SDR1) — C++ SDR, WDSP-based
- **OpenHPSDR Zeus** (brianbruff) — .NET 10, full TCI server, archived 2026-07-17
- **AnanCore** (ChasingCoffee) — Zeus fork; TCI likely inherited
- **station-engine** (Zeus-SDR) — Zeus fork; TCI likely inherited
- **Bolt SDR** (pe5jw) — Zeus-derived; TCI likely inherited
- **Rigflow** (dbourgoyne) — TCI *client* only (digital mode bridge)
- **Kainos** (PNWHam-K7JUS) — skeleton/early stage, TCI not confirmed

> **Note on OpenHPSDR Zeus forks:** Zeus has a complete TCI server implementation (~2100 lines C#). AnanCore, station-engine, and Bolt SDR are all forks of Zeus and likely inherit its TCI server. Earlier analysis incorrectly stated these had no TCI — that was wrong. Zeus uses protocol name `"ExpertSDR3"` version `"2.0"` in its handshake.

---

## 1. Audio Streaming: `audio_start` / `audio_stop`

Audio streaming delivers decoded RX audio (float32 stereo PCM) to a client over the WebSocket connection in binary frames (frame_type = 1 for RX, 2 for TX).

### Command Syntax

| Operation | ExpertSDR3 (reference) | deskHPSDR | Thetis | NereusSDR | Zeus |
|---|---|---|---|---|---|
| Start RX audio | `audio_start:trx,ch;` | `audio_start:rx;` | `audio_start:rx;` (stub) | `audio_start:rx;` (stub) | `audio_start:rx[,bool];` |
| Stop RX audio | `audio_stop:trx,ch;` | `audio_stop:rx;` | `audio_stop:rx;` (stub) | `audio_stop:rx;` (stub) | `audio_stop:rx;` |
| Sample rate query | `audio_samplerate;` | `audio_samplerate;` | `audio_samplerate;` | `audio_samplerate;` | `audio_samplerate;` |
| Set sample rate | `audio_samplerate:48000;` | `audio_samplerate:48000;` | `audio_samplerate:48000;` | `audio_samplerate:48000;` | `audio_samplerate:48000;` (range 8000–48000) |

### Key Differences

**Argument count for `audio_start`:**
- ExpertSDR3: `audio_start:trx,ch;` — 2 arguments (TRX index, channel index)
- deskHPSDR: `audio_start:rx;` — 1 argument only (receiver index); rejects any other arg count
- Thetis: `audio_start:rx;` — 1 argument; currently a **stub** (no audio frames sent yet; command accepted silently)
- NereusSDR: `audio_start:rx;` — 1 argument; also a **stub** pending Phase 16 implementation

**Binary frame format (all implementations):**
```
Bytes 0–63:  64-byte header
  [0–3]   frame_type (uint32 LE): 1 = RX audio, 2 = TX audio
  [4–7]   receiver_index (uint32 LE)
  [8–11]  sample_rate (uint32 LE)
  [12–15] format (uint32 LE): 0 = float32
  [16–19] channels (uint32 LE): 1 or 2
  [20–23] sample_count (uint32 LE)
Bytes 64+:   float32 samples (interleaved L/R for stereo)
```

**deskHPSDR audio stream configuration commands** (not in reference spec):
```
audio_stream_sample_type:float32;       # always float32 in practice
audio_stream_channels:2;                # 1 or 2; default 2
audio_stream_samples:2048;              # block size; max 2048; clamped if over
tx_stream_audio_buffering:true;         # enable TX audio buffering
```

**Thetis audio parameters** (same config commands, confirmed working):
```
audio_samplerate:48000;                 # only valid rate
audio_stream_sample_type:float32;       # confirmed accepted
audio_stream_channels:2;               # stereo confirmed
audio_stream_samples:2048;             # clamped at 2048 max
```

**Server echo behavior:**
- deskHPSDR: echoes `audio_start:<rx>;` / `audio_stop:<rx>;` to the calling client
- Thetis: no echo (stub, silently accepted)
- ExpertSDR3: unknown/unconfirmed

---

## 2. IQ Streaming: `iq_start` / `iq_stop`

IQ streaming delivers raw complex baseband samples (float32 interleaved I/Q) in binary frames (frame_type = 0).

### Command Syntax

| Operation | ExpertSDR3 (reference) | deskHPSDR | Thetis | NereusSDR | Zeus |
|---|---|---|---|---|---|
| Start IQ stream | `iq_start:trx;` | `iq_start:rx;` or `iq_start;` | `iq_start:rx;` (stub) | `iq_start:rx;` (stub) | `iq_start:rx[,bool];` |
| Stop IQ stream | `iq_stop:trx;` | `iq_stop:rx;` or `iq_stop;` | `iq_stop:rx;` (stub) | `iq_stop:rx;` (stub) | `iq_stop:rx;` |
| Set IQ rate | `iq_samplerate:rate;` | `iq_samplerate:rate;` | `iq_samplerate:rate;` | `iq_samplerate:rate;` | `iq_samplerate:rate;` (echoes hardware rate) |
| Query IQ rate | `iq_samplerate;` | `iq_samplerate;` | `iq_samplerate;` | `iq_samplerate;` | `iq_samplerate;` |

### Key Differences

**deskHPSDR — Exclusive IQ Ownership Model:**

This is one of the most significant deviations. deskHPSDR implements *exclusive* IQ stream ownership:

1. The **first client** to send `iq_start` becomes the IQ stream owner and sets the sample rate for all clients
2. If a second client sends `iq_start` at a different rate, it gets the *owner's rate* — it cannot change it
3. If the **owner** sends `iq_stop`, ALL IQ streams are stopped for ALL clients (not just the owner)
4. Non-owner clients sending `iq_stop` only stop their own stream

```
Client A: iq_samplerate:192000;    → sets rate to 192000 Hz
Client A: iq_start:0;              → Client A becomes owner at 192000 Hz
Client B: iq_start:0;              → Client B gets 192000 Hz (owner's rate, not configurable)
Client A: iq_stop:0;               → ALL clients' IQ streams are stopped!
```

Valid IQ sample rates in deskHPSDR: **48000, 96000, 192000, 384000** Hz (others rejected)

**Thetis IQ sample rate behavior:**
- Accepts **any integer** value (no rate validation); just stores and echoes back
- `iq_start` / `iq_stop` are **stubs** pending Phase 18 (binary IQ pipeline)
- Commands accepted silently, no frames sent

**NereusSDR:**
- IQ start/stop are stubs
- Has `SETUP:TciAlwaysStreamIq=True;` compat flag to auto-start IQ streaming on connect

**deskHPSDR server echo:**
- `iq_start:<rx>;` echoed to the calling client on success
- `iq_stop:<rx>;` echoed to the calling client

---

## 3. PTT / TX Control: `trx`

The `trx` command switches a receiver into transmit mode (PTT). This is where **TX audio injection** and TX ownership are controlled.

### Command Syntax

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR | Zeus |
|---|---|---|---|---|---|
| PTT on | `trx:0,true;` | `trx:0,true;` | `trx:0,true;` | `trx:0,true;` | `trx:0,true;` |
| PTT off | `trx:0,false;` | `trx:0,false;` | `trx:0,false;` | `trx:0,false;` | `trx:0,false;` |
| PTT + TX audio | (unknown) | `trx:0,true,tci;` | not supported | not supported | `trx:0,true,tci;` |
| Query PTT state | `trx:0;` | `trx:0;` | `trx:0;` | `trx:0;` | `trx:0;` |

### Key Differences

**deskHPSDR — 3rd argument `tci` for TX audio session:**

This is a deskHPSDR-specific extension:
```
trx:0,true,tci;    # PTT on AND register as TX audio client
trx:0,false;       # PTT off (3rd arg not needed for off)
```

The `tci` parameter:
- Marks the calling WebSocket client as the TX audio provider
- TX ownership is **exclusive** — refused if another client or local hardware controls TX
- Enables the binary TX audio frame pipeline for this client (frame_type = 2)
- Without `tci`, PTT works but no TX audio can be sent over WebSocket

**Broadcast behavior differences:**
- ExpertSDR3: `trx` state changes broadcast to all connected clients
- deskHPSDR: `trx` state changes broadcast to all clients
- **Thetis: NO broadcast** — only the requesting client receives the response; other clients do NOT hear PTT state changes via TCI
- NereusSDR: same as Thetis (no broadcast for `trx`)

```
# Thetis/NereusSDR — only the sender sees:
Client A sends: trx:0,true;
Client A receives: trx:0,true;   ← only A hears this
Client B sees: (nothing)          ← B is not notified!
```

This means multi-client applications that rely on PTT state sync will **not work** with Thetis or NereusSDR without polling `trx:0;` periodically.

---

## 4. Frequency / VFO: `vfo`

### Command Syntax

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR |
|---|---|---|---|---|
| Set VFO-A | `vfo:0,0,14250000;` | `vfo:0,0,14250000;` | `vfo:0,0,14250000;` | `vfo:0,0,14250000;` |
| Set VFO-B | `vfo:0,1,7100000;` | `vfo:0,1,7100000;` | `vfo:0,1,7100000;` | `vfo:0,1,7100000;` |
| Query VFO-A | `vfo:0,0;` | `vfo:0,0;` | `vfo:0,0;` | `vfo:0,0;` |
| Response | `vfo:0,0,14250000;` | `vfo:0,0,14250000;` | `vfo:0,0,14250000;` | `vfo:0,0,14250000;` |

The `vfo` command format is largely **consistent** across implementations. The parameters are:
- `rx` — receiver index (0-based)
- `ch` — channel: 0 = VFO-A, 1 = VFO-B
- `freq` — frequency in Hz (integer)

**VFO lock** uses a separate command with 3 args: `vfo_lock:rx,ch,bool;`

**Split operation** is enabled separately: `split_enable:rx,bool;`

---

## 5. Modulation Mode

### Command Syntax

| Mode | ExpertSDR3 sends | deskHPSDR sends | Thetis sends | NereusSDR | Zeus sends |
|---|---|---|---|---|---|
| Upper Sideband | `modulation:0,USB;` | `modulation:0,USB;` | `modulation:0,USB;` | accepts `usb` → echoes `USB` | `modulation:0,USB;` |
| Lower Sideband | `modulation:0,LSB;` | `modulation:0,LSB;` | `modulation:0,LSB;` | accepts `lsb` → echoes `LSB` | `modulation:0,LSB;` |
| CW (upper) | `modulation:0,CWU;` | `modulation:0,CW;` ⚠️ | `modulation:0,CWU;` | accepts `cwu` → echoes `CWU` | `modulation:0,CWU;` ✓ |
| CW (lower) | `modulation:0,CWL;` | `modulation:0,CW;` ⚠️ | `modulation:0,CWL;` | accepts `cwl` → echoes `CWL` | `modulation:0,CWL;` ✓ |
| Narrow FM | `modulation:0,NFM;` | `modulation:0,FM;` ⚠️ | `modulation:0,NFM;` | accepts `nfm` or `fm` → echoes `FM` | `modulation:0,FM;` (accepts `NFM` alias) |
| AM | `modulation:0,AM;` | `modulation:0,AM;` | `modulation:0,AM;` | accepts `am` → echoes `AM` | `modulation:0,AM;` |
| Digital USB | `modulation:0,DIGU;` | `modulation:0,DIGU;` | `modulation:0,DIGU;` | standard | `modulation:0,DIGU;` |
| Digital LSB | `modulation:0,DIGL;` | `modulation:0,DIGL;` | `modulation:0,DIGL;` | standard | `modulation:0,DIGL;` |

### Key Differences

**deskHPSDR — CWL and CWU are both sent as `CW`:**

deskHPSDR maps both CW modes to the single string `"CW"`:
```
# deskHPSDR sends (regardless of CWL vs CWU):
modulation:0,CW;

# Standard / Thetis sends:
modulation:0,CWL;    # or CWU
```

A TCI client reading deskHPSDR cannot distinguish CWL from CWU from the modulation command alone. The actual pitch/sideband is controlled by the hardware.

**deskHPSDR — NFM is sent as `FM`:**
```
# deskHPSDR sends:
modulation:0,FM;

# Standard sends:
modulation:0,NFM;
```

**Thetis — case-insensitive input, uppercase output:**
- Accepts: `usb`, `USB`, `Usb` (any case)
- Always echoes/broadcasts in uppercase: `USB`, `CWL`, `NFM`

**NereusSDR — NFM alias:**
- Accepts both `nfm` and `fm` as input
- Always echoes `FM` (not `NFM`)
- Case-insensitive input

**NereusSDR compat flag for CW handling:**
```
SETUP:TciCwluBecomesCw=True;           # make Thetis send CW like deskHPSDR
SETUP:TciCwBecomesCwuAbove10mhz=True;  # map CW → CWU above 10 MHz
```

---

## 6. AGC (Automatic Gain Control)

### Command Syntax

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR | Zeus |
|---|---|---|---|---|---|
| Set AGC mode | `agc_auto_ex:rx,ch,bool;` | `agc_mode:rx,mode;` ⚠️ | `agc_mode:rx,mode;` | `agc_mode:rx,mode;` | `agc_mode:rx,mode;` |
| AGC gain | `agc_gain:rx,gain;` | `agc_gain:rx,gain;` | `agc_gain:rx,gain;` | `agc_gain:rx,gain;` | `agc_gain:rx,gain;` |
| Query AGC mode | `agc_mode:rx;` | `agc_mode:rx;` | `agc_mode:rx;` | `agc_mode:rx;` | `agc_mode:rx;` |

### Key Differences

**ExpertSDR3 vs deskHPSDR/Thetis/NereusSDR — different command name:**
```
# ExpertSDR3 (reference spec):
agc_auto_ex:0,0,true;    # enable auto AGC on rx=0, ch=0

# deskHPSDR, Thetis, NereusSDR:
agc_mode:0,fast;         # set AGC mode: off / fast / normal
```

The `agc_auto_ex` command from the ExpertSDR3 reference is **not implemented** in deskHPSDR, Thetis, or NereusSDR. They use `agc_mode` with a named mode instead.

**AGC mode values:**
| Mode string | Meaning |
|---|---|
| `off` | AGC disabled |
| `fast` | Fast attack/decay |
| `normal` | Normal (also: `med` alias in Thetis/NereusSDR) |
| `slow` | (ExpertSDR3 only?) |
| `long` | (ExpertSDR3 only?) |
| `auto` | (ExpertSDR3 only?) |

**Thetis/NereusSDR `med` alias:**
```
agc_mode:0,med;    → echoes agc_mode:0,normal;   # "med" maps to "normal"
agc_mode:0,bogus;  → echoes agc_mode:0,normal;   # unknown mode → default normal
```

**AGC gain clamping (Thetis/NereusSDR):**
```
agc_gain:0,200;    → agc_gain:0,120;    # clamped to max 120
agc_gain:0,-100;   → agc_gain:0,-20;    # clamped to min -20
```
Range: -20 to +120 dB

---

## 7. DSP Options: NR / NB / ANF (WDSP)

These are noise reduction and filtering options, implemented via WDSP (Warren Pratt's DSP library) in deskHPSDR, Thetis, NereusSDR, and Lyra-SDR-cpp.

### Command Syntax

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR |
|---|---|---|---|---|
| Noise Blanker | `rx_nb_enable:rx,bool;` | `rx_nb_enable:rx,bool;` | `rx_nb_enable:rx,bool;` | `rx_nb_enable:rx,bool;` |
| Noise Reduction | `rx_nr_enable:rx,bool;` | `rx_nr_enable:rx,bool;` | `rx_nr_enable:rx,bool;` | `rx_nr_enable:rx,bool;` |
| NR extended | `rx_nr_enable_ex:rx,bool,idx;` | not confirmed | `rx_nr_enable_ex:rx,bool,idx;` | `rx_nr_enable_ex:rx,bool,idx;` |
| Auto Notch Filter | `rx_anf_enable:rx,bool;` | `rx_anf_enable:rx,bool;` | `rx_anf_enable:rx,bool;` | `rx_anf_enable:rx,bool;` |
| Auto Peak Filter | `rx_apf_enable:rx,bool;` | `rx_apf_enable:rx,bool;` | not confirmed | not confirmed |
| Noise Floor | `rx_nf_enable:rx,bool;` | `rx_nf_enable:rx,bool;` | not confirmed | not confirmed |
| BIN mode | `rx_bin_enable:rx,bool;` | `rx_bin_enable:rx,bool;` | not confirmed | not confirmed |

### Key Differences

**`rx_nr_enable_ex` — Thetis dual-emit behavior:**

When Thetis processes `rx_nr_enable_ex`, it emits **two notifications**:
1. The basic form `rx_nr_enable:rx,bool;`
2. The extended form `rx_nr_enable_ex:rx,bool,idx;`

```
Client sends: rx_nr_enable_ex:0,true,2;
Server emits (broadcast):
  rx_nr_enable:0,true;          ← basic form first
  rx_nr_enable_ex:0,true,2;     ← extended form second
```

A client listening for both events will receive two notifications for one command. Clients that only know the basic form still get notified.

**NR index range (Thetis/NereusSDR):**
- Valid `nr_index` values: 1 to 4
- Index 2 = WDSP NR2 (spectral subtraction, typically best quality)

**deskHPSDR NR/NB commands in dispatch table:**
- `rx_nb_enable`, `rx_nr_enable`, `rx_anf_enable`, `rx_apf_enable`, `rx_nf_enable`, `rx_bin_enable` all confirmed present
- Direct WDSP calls; exact behavior matches reference syntax

---

## 8. RX Filter Band

### Command Syntax

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR |
|---|---|---|---|---|
| Set filter | `rx_filter_band:rx,lo,hi;` | `rx_filter_band:rx,lo,hi;` | `rx_filter_band:rx,lo,hi;` | `rx_filter_band:rx,lo,hi;` |
| Query filter | `rx_filter_band:rx;` | `rx_filter_band:rx;` | `rx_filter_band:rx;` | `rx_filter_band:rx;` |
| Response | `rx_filter_band:rx,lo,hi;` | `rx_filter_band:rx,lo,hi;` | `rx_filter_band:rx,lo,hi;` | `rx_filter_band:rx,lo,hi;` |

Filter values are in Hz, relative to the carrier (e.g., `rx_filter_band:0,200,2900;` = 200–2900 Hz passband for USB).

This command is **consistent** across all implementations.

---

## 9. Squelch

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR |
|---|---|---|---|---|
| Enable | `sql_enable:rx,bool;` | `sql_enable:rx,bool;` | `sql_enable:rx,bool;` | `sql_enable:rx,bool;` |
| Set level | `sql_level:rx,dBm;` | `sql_level:rx,dBm;` | `sql_level:rx,dBm;` | `sql_level:rx,dBm;` |
| Query level | `sql_level:rx;` | `sql_level:rx;` | `sql_level:rx;` | `sql_level:rx;` |

**Thetis/NereusSDR level clamping:**
- Range: -140 to 0 dBm
- Values outside range are clamped: `sql_level:0,100;` → `sql_level:0,0;`
- Values below: `sql_level:0,-200;` → `sql_level:0,-140;`

---

## 10. Volume and Monitor

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR |
|---|---|---|---|---|
| Master volume | `volume:val;` | `volume:val;` | `volume:val;` | `volume:val;` |
| Per-RX volume | `rx_volume:rx,ch,val;` | `rx_volume:rx,ch,val;` | `rx_volume:rx,ch,val;` | standard |
| TX monitor vol | `mon_volume:val;` | `mon_volume:val;` | standard | standard |
| TX monitor on/off | `mon_enable:bool;` | `mon_enable:bool;` | standard | standard |
| Mute master | `mute:bool;` | `mute:bool;` | `mute:bool;` | `mute:bool;` |
| Query mute | `mute;` | `mute;` | `mute;` | `mute;` |

These commands are **consistent** across implementations.

---

## 11. NereusSDR-Specific: SETUP: Compatibility Flags

NereusSDR supports non-standard `SETUP:` commands for compatibility with different client expectations:

```
SETUP:TciEmulateExpertSDR3Protocol=True;    # emulate ExpertSDR3 init sequence
SETUP:TciEmulateSunSDR2Pro=True;            # emulate SunSDR2 Pro responses
SETUP:TciCwluBecomesCw=True;               # send CW instead of CWL/CWU (like deskHPSDR)
SETUP:TciCwBecomesCwuAbove10mhz=True;      # map CW→CWU above 10 MHz
SETUP:TciIqSwap=True;                       # swap I and Q in IQ stream
SETUP:TciAlwaysStreamIq=True;              # auto-start IQ streaming on connect
SETUP:TciSendInitialFrequencyStateOnConnect=True;  # resend all VFO state on connect
SETUP:TciRateLimitMs=10;                   # throttle command processing (ms)
SETUP:TciForgetRx2VfoBOnDisconnect=True;   # reset RX2 VFO-B on client disconnect
SETUP:TciUseRx1VfoaForRx2Vfoa=True;       # link RX1 and RX2 VFO-A
SETUP:TciCopyRx2VfobToVfoa=True;          # copy RX2 VFO-B value to VFO-A
```

---

## 12. RIT / XIT Offsets

| Operation | ExpertSDR3 | deskHPSDR | Thetis | NereusSDR |
|---|---|---|---|---|
| Enable RIT | `rit_enable:rx,bool;` | `rit_enable:rx,bool;` | standard | standard |
| Set RIT offset | `rit_offset:rx,hz;` | `rit_offset:rx,hz;` | standard | standard |
| Enable XIT | `xit_enable:rx,bool;` | `xit_enable:rx,bool;` | standard | standard |
| Set XIT offset | `xit_offset:rx,hz;` | `xit_offset:rx,hz;` | standard | standard |

These are consistent across implementations.

---

## Summary: Major Deviations by Implementation

### deskHPSDR (dl1bz)
| Command | Deviation |
|---|---|
| `audio_start` | 1 arg only (`rx`), not 2 (`trx,ch`) |
| `iq_start` / `iq_stop` | **Exclusive ownership** — first client owns rate; owner's `iq_stop` kills all streams |
| `trx` | Optional 3rd arg `tci` enables TX audio injection |
| `modulation` | CWL/CWU → `CW`; NFM → `FM` |
| `agc_mode` | Uses `agc_mode` with `off/fast/normal`; does NOT have `agc_auto_ex` |
| IQ rates | Only 48k/96k/192k/384k accepted; others rejected |
| Config cmds | `audio_stream_sample_type`, `audio_stream_channels`, `audio_stream_samples`, `tx_stream_audio_buffering` |
| RTTY/CW | Native RTTY and CW macro extensions (must negotiate first) |

### Thetis (ramdor / mi0bot)
| Command | Deviation |
|---|---|
| `audio_start` / `audio_stop` | **Stub** — silently accepted, no audio frames sent yet |
| `iq_start` / `iq_stop` | **Stub** — silently accepted, no IQ frames sent yet |
| `trx` | **No broadcast** — only the requesting client hears state changes |
| `modulation` | Case-insensitive input; always echoes uppercase (`USB`, `CWL`, etc.) |
| `agc_mode` | `med` alias → echoes `normal`; unknown → `normal` |
| `agc_gain` | Clamped: range -20 to +120 dB |
| `rx_nr_enable_ex` | Dual-emit: sends basic `rx_nr_enable` AND extended form on state change |
| `sql_level` | Clamped: range -140 to 0 dBm |

### NereusSDR (boydsoftprez)
| Command | Deviation |
|---|---|
| `audio_start` / `audio_stop` | **Stub** — Phase 16, pending implementation |
| `iq_start` / `iq_stop` | **Stub** — Phase 18, pending implementation |
| `trx` | **No broadcast** — only the requesting client hears state changes |
| `modulation` | NFM and FM both → `FM`; case-insensitive; echoes uppercase |
| `agc_mode` | `med` alias → `normal`; unknown → `normal`; gain clamped -20..+120 |
| `sql_level` | Clamped: range -140 to 0 dBm |
| `SETUP:` | Unique compat flags to emulate other SDR software behavior |
| `iq_samplerate` | Accepts any integer (no rate validation) |

### OpenHPSDR Zeus (brianbruff)
| Command | Behavior |
|---|---|
| `audio_start` | `audio_start:rx[,bool];` — 1 or 2 args; bool defaults to true; per-session subscription (not exclusive) |
| `iq_start` | `iq_start:rx[,bool];` — per-session subscription; echoes `iq_start:rx,bool;` |
| `iq_samplerate` | Accepts any rate, echoes the **hardware's actual rate** (not the requested rate) |
| `trx` | Supports `trx:rx,bool,signal_source;` — source ∈ {`tci`, `mic1`, `mic2`, `micPC`, `ecoder2`} |
| `modulation` | Sends `CWL`/`CWU` correctly (not collapsed); sends `FM` (accepts `NFM` as alias); uppercase output |
| `agc_mode` | Uses `FIXED/LONG/SLOW/MEDIUM/FAST/CUSTOM`; `OFF`→`FIXED`; `NORMAL`/`MED`→`MEDIUM` |
| Broadcast | Full state broadcast to all clients; PTT changes ARE broadcast (unlike Thetis/NereusSDR) |
| Protocol ident | `protocol:ExpertSDR3,2.0;` + `device:Zeus;` in init block |

### AnanCore, station-engine, Bolt SDR (Zeus forks)
All are derived from OpenHPSDR Zeus and likely inherit its TCI server implementation. Not independently verified.

### Lyra-SDR-cpp, Kainos
- **Lyra-SDR-cpp**: Uses WDSP; standard TCI interface expected; full analysis pending
- **Kainos**: Early stage; TCI interface not yet confirmed functional

---

## Typical Workflow: Starting Audio Reception

### ExpertSDR3 (reference)
```
# On connect, server sends init block ending with:
ready

# Configure audio
audio_samplerate:48000;

# Subscribe to RX0 audio stream (2 args: trx, channel)
audio_start:0,0;

# Server now sends binary audio frames (frame_type=1)
```

### deskHPSDR
```
# On connect, server sends init block ending with:
ready

# Optional: configure stream format
audio_stream_sample_type:float32;
audio_stream_channels:2;
audio_stream_samples:2048;

# Subscribe to RX0 audio stream (1 arg: rx only)
audio_start:0;
# Server echoes: audio_start:0;

# Server now sends binary audio frames (frame_type=1)
```

### Thetis / NereusSDR
```
# On connect, server sends init block ending with:
ready

# Configure audio
audio_samplerate:48000;

# Subscribe (1 arg; command accepted but NO frames sent — stub)
audio_start:0;

# NOTE: no audio frames will arrive — binary pipeline not yet implemented
```

---

## Typical Workflow: TX with Audio Injection

### deskHPSDR (only implementation with TX audio over TCI)
```
# Ensure TX audio buffering is on
tx_stream_audio_buffering:true;

# PTT on + register as TX audio provider (3rd arg "tci" is essential)
trx:0,true,tci;

# Now send TX audio as binary frames (frame_type=2):
# Header: frame_type=2, receiver_index=0, sample_rate=48000,
#         format=0 (float32), channels=2, sample_count=N
# Payload: float32 stereo PCM samples

# PTT off
trx:0,false;
```

### Thetis / NereusSDR / ExpertSDR3
```
# No TX audio injection over TCI (not implemented / not documented)
# PTT still works:
trx:0,true;
# But audio must come from the local system (mic/VAC/virtual cable)
trx:0,false;
```

---

## Data Sources

| Implementation | Source analyzed |
|---|---|
| deskHPSDR | `tci.c` (dl1bz/deskhpsdr, 6203 lines, full dispatch table + all handlers) |
| Thetis (ramdor/mi0bot) | NereusSDR test matrix (cites `TCIServer.cs` line numbers for v2.10.3.13–15) |
| NereusSDR | `nereus_matrix.csv` (official TCI verification test matrix, 17 KB) |
| ExpertSDR3 | TCI protocol reference, on7off/TCI-Remote reference docs |
| OpenHPSDR Zeus | `TciSession.cs` (2096 lines) + `TciProtocol.cs` (244 lines) + `TciHandshake.cs` — direct source analysis |
| Lyra-SDR-cpp | Source scan (WDSP-based; standard TCI expected) |
| AnanCore, station-engine, Bolt SDR | Zeus forks — TCI inherited; not independently analyzed |
| Kainos | Skeleton only — TCI not confirmed |
