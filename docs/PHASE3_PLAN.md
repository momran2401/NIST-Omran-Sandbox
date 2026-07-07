# PHASE3_PLAN.md — Multi-SDR support for the live web viewer

Design consultation + execution plan, authored 2026-07-07 after a full read of
`live/striqt_web_server.py`, `live/web/`, `live/pluto_standalone.py`, and
`striqt/src/striqt/sensor/` (read-only). This document is authoritative for Phase 3
scope, approach, phasing, and the do-NOT list.

---

## 1. Verdict & recommended scope

**Worth doing — but only at the narrow scope: AIR8201B + PlutoSDR + demo.**

The honest read: the codebase is *closer* to multi-device than it looks on the server
side (nearly everything already sizes on `len(CHANNELS)`), and *further* than it looks
on the frontend (every render path hardcodes two channels). The deciding constraint is
striqt itself: **striqt has real-hardware bindings only for Deepwave AirStack radios**
(`air7101b`/`air7201b`/`air8201b` → `Airstack1Source`). There is no Pluto, RTL-SDR,
Lime, HackRF, or UHD binding. Every non-Deepwave device therefore rides on the
subclass-and-override pattern that `live/pluto_standalone.py` already proved — viable
for one or two devices, not a foundation for "the whole SoapySDR zoo."

**Build:**
- Launch-time device selection: `--device {air8201b, pluto, demo, auto}`.
- A `DEVICE_PROFILES` table (channels, label, defaults, capability envelope, source factory).
- Fully dynamic channel count, server and frontend (covers 1-ch Pluto and 2-ch AIR-T,
  and incidentally 3–4 ch future devices).
- Device capability envelope (freq/gain/rate ranges) feeding the **existing** freedom-model
  tier-1 clamps, reported through the existing ack — no parallel validator.
- Device identity in the UI (title, labels, channel count) instead of hardcoded "AIR8201".

**Defer (do not build in Phase 3):**
- In-browser device menu / runtime hot-swap between physical radios.
- Profiles for RTL-SDR / LimeSDR / HackRF / USRP / AirSpy.
- Per-device calibration (striqt calibration files exist only for AirStack).
- Unifying the standalone scripts into the web server.
- Hotplug / disconnect detection beyond the existing `Acquirer._recover` retry loop.

## 2. Device selection: launch flag + opt-in auto-detect (no browser menu)

**Decision: CLI `--device`, default `air8201b`, with `auto` as an opt-in convenience.
No in-browser device menu.**

Rationale: the server runs *on the radio's own host* — the Jetson inside the AIR-T, a
Raspberry Pi next to a Pluto. A given host effectively has one radio. The realistic
framing is **"one codebase configured per host at launch,"** not "one server that
hot-swaps local SDRs" and not "connect to remote radios." A browser device menu would
imply runtime source teardown/rebuild (a whole class of wedge states the freedom model
was built to avoid) for a scenario — two different radios on one host — that doesn't
exist in this deployment. Aric's menu idea is satisfied at the right layer: `--device
auto` enumerates SoapySDR, picks the single supported device, and errors with the full
enumeration listing when there are zero or several.

Default `air8201b` (not `auto`) so the existing Jetson deployment is byte-identical to
today with no flag changes. `--demo` is kept as an alias for `--device demo`.
DAN/ARIC are unaffected: both modes are pure frontend CSS gating; device identity just
becomes a label they display.

## 3. Channel count: fully dynamic, set once at launch

The 2→N conversion is cheaper than feared on the server and a genuine (but mechanical)
refactor on the frontend:

- **Server:** `CHANNELS` stays a module global, assigned once in `main()` from the
  device profile *before* any thread or `SharedConfig` is constructed. All existing
  usage sites (`make_capture`, ring buffers, read buffers, `build_header`, stream
  shims) are runtime reads of `len(CHANNELS)` — they need zero edits. The only
  hardcoded-2 site is `DemoAcquirer` (generates exactly ch0/ch1 tone sets).
- **Frontend:** the real work. `app.js` hardcodes `{0,1}` buffer maps, `wf0`/`wf1`
  canvases, a fixed 9-series uPlot layout (RX1/RX2 × mean/max/hold/min + RX1−RX2),
  `[0,1]` loops in the server-PSD renderer, band monitor, CSV/PNG export, and peak
  markers. The fix: derive the channel list from the frame header's `channels` field
  (already shipped by the server, already honored by the block parser), template-clone
  the waterfall panes, and generate the uPlot series set from the channel list — the
  plot is already rebuilt lazily on tuning changes (`uplotKind`), so this slots in
  cleanly. The RX1−RX2 diff trace and the band monitor's Δ/Q segments exist only when
  exactly 2 channels are present. 1-channel degradation is clean: one full-width pane,
  single PSD pair, no diff — not invasive once rendering is list-driven.

Compatibility note (verified): the *current* frontend degrades gracefully against a
1-channel header (second pane freezes) but **crashes** on 3+ channels
(`wfCanvas[2]` is undefined). The phasing below therefore caps the demo `--channels`
override at 2 until the frontend is channel-dynamic, then lifts it.

## 4. The striqt-binding finding (deciding constraint), stated plainly

- `striqt/src/striqt/sensor/bindings.py` registers real hardware **only** for Deepwave
  AirStack (`air7101b`/`air7201b`/`air8201b`), plus null/function/file sources.
- The SoapySDR driver string is **not a spec field** — it is hardcoded
  (`driver='SoapyAIRT'`) in `Airstack1Source.__init__`
  (`striqt/src/striqt/sensor/lib/sources/deepwave.py`), which also calls
  `_set_jesd_sysref_delay()` — an FPGA register write that **crashes any non-Deepwave
  device**, plus `get_id()` reading the Jetson `eth0` MAC and `read_peripherals()`
  reading an AirStack-only temperature sensor.
- Consequence: any non-Deepwave device must subclass and override. The
  `PlutoSource` POC in `live/pluto_standalone.py` does exactly and only this:
  `__init__` → `SoapySource.__init__(spec, driver='plutosdr')` (skips the FPGA write),
  `get_id()` → `getHardwareKey()` or `'pluto'`, `read_peripherals()` → `{}`. Phase 3
  ports it verbatim into the web server. `striqt/` is never edited.
- The **analysis pipeline is device-agnostic** — `evaluate_spectrogram` is pure
  IQ→spectrogram and never touches the device. So Pluto gets the full analysis suite.
- **Calibration is AirStack-only** (cal-file path on the Soapy spec + Y-factor tooling
  bound only to the three AirStack radios). The live viewer never loads a cal file —
  its "calibrated" backend is a grid/window convention, not absolute power cal — so
  Pluto runs it fine, just uncalibrated in the absolute-power sense. Acceptable;
  labeled, not hidden.

## 5. Capability envelope: query the device, feed the existing freedom model

SoapySDR exposes `getFrequencyRange` / `getGainRange` / `getSampleRateRange`. Phase 3
adds a per-profile **envelope** `{freq_min, freq_max, gain_min, gain_max, rate_min,
rate_max}` with two sources: a static per-profile fallback, and (when the profile sets
`query_envelope=True`) a live query after the radio opens. The envelope lives on
`SharedConfig` and is consumed by the **existing tier-1 clamps** in
`SharedConfig.update` — replacing the hardcoded `300e6–6e9` center clamp, `−60..10`
gain clamp, and rate bounds — and is published additively via `GET /config`. Tier-2
(scratch-validate) and tier-3 (compute backstop) are untouched and remain the safety
net. Acks (`rounded`/`rejected`) work unchanged because the clamp sites are unchanged —
only their bounds become device-derived.

Deliberate conservatism: **`query_envelope=False` for `air8201b` and `demo`**, whose
fallback envelopes are today's exact hardcoded numbers. The AIR-T's `−60..10` gain
range is a striqt calibrated-gain convention, not SoapyAIRT's raw reported range;
querying it live could shift legal bounds on the existing deployment. Pluto queries
live (fallback: 325 MHz–3.8 GHz, gain 0–73 dB, 0.52–61.44 MS/s).

The `RATES_HZ` LTE grid (3.84/7.68/15.36/30.72 MS/s) stays — it is domain logic, not a
device property — but is intersected with the envelope (falling back to the full grid
if the intersection is empty). The SSB-grid rate bypass is unchanged.

## 6. Phased implementation plan

Each phase = one commit (`P3-n: title`) + a `docs/FIXLOG.md` entry with a
`[demo]`/`[hardware]` Verify. After every commit: `python -m py_compile
live/striqt_web_server.py` and (when touched) `node --check live/web/app.js`.
The tree is demo-green after every commit; default launch behavior never changes.

- **P3-0** — this document.
- **P3-1 — Device profiles + `--device` CLI (server only).**
  `live/striqt_web_server.py`: add `SoapySource` to the guarded striqt import; port
  `PlutoSource` from the POC (under `if _SENSOR_OK:`); `DEVICE_PROFILES` (channels,
  label, defaults — Pluto default rate 3.84 MS/s for USB headroom —, envelope +
  `query_envelope`, `make_source` factory); `DEVICE`/`DEVICE_LABEL` globals;
  `make_source()` becomes a profile dispatch; `_resolve_auto_device()` via
  `SoapySDR.Device.enumerate()`; `main()` gains `--device`, maps `--demo`, sets
  `CHANNELS` from the profile before `SharedConfig()`; RadioConfig seeds from profile
  defaults (identical values for air8201b/demo).
- **P3-2 — N-channel DemoAcquirer + `--channels` (demo-only, 1|2) + device metadata.**
  `DEMO_TONES` table cycled per channel (2-ch output identical to today);
  `build_header` adds `"device": DEVICE_LABEL` (additive); `/config` adds
  `device: {name, label, channels}`.
- **P3-3 — Capability envelope.** `query_device_envelope(source)` (per-key
  try/except); `SharedConfig._envelope` + lock-guarded accessors, set from
  `Acquirer.open_radio`; tier-1 clamp sites + `_effective_radio` read the envelope;
  `allowed_rates(env)` = grid ∩ envelope; `/config` publishes the envelope.
- **P3-4 — Channel-dynamic frontend** (+ lift `--channels` cap to 1–4).
  `ensureChannels()` rebuilds panes from a `<template>` on channel-signature change;
  `chColors(i)` palette (indices 0/1 keep today's exact colors); uPlot series
  generated from the channel list (diff trace iff 2 ch); loops replace `[0,1]` in
  updatePSD/psdSeries/initUplotPsdStats/renderServerPsd/peak markers/band monitor
  (Δ/Q iff 2 ch)/CSV/PNG.
- **P3-5 — Device identity in the UI.** Neutral static title fallbacks; `document.title`,
  h1, subtitle, meta bar set from `header.device` + channel count; envelope min/max
  hints on the DAN capture form (display only — server clamps stay authoritative).
- **P3-6 — Docs + hardware checklist.** `run_web.sh` header notes `--device`
  passthrough (it already forwards `"$@"`); this file gains the results/checklist
  updates; FIXLOG completed.

Demo mode survives every phase (it is just the `demo` profile; `DemoAcquirer` keeps its
own compute loop and tier-3 backstop). DAN/ARIC survive every phase (CSS-gated; the
generated panes reproduce the existing markup classes exactly).

## 7. Risks, open questions, and on-hardware tests

Risk concentration, in order: (1) the frontend refactor (most surface, mitigated by
pixel-identical 2-ch behavior as the acceptance bar); (2) Pluto streaming stability at
higher rates over USB (mitigated: default 3.84 MS/s); (3) SoapyPlutoSDR quirks in
range-object APIs and `setMasterClockRate` behavior (mitigated: per-key try/except +
profile fallbacks; the spec's 125 MHz master clock is an AIR-T value the Pluto driver
is expected to ignore — verify on hardware).

**`[hardware]` checklist — AIR8201B (regression):** default launch (no flags) behaves
identically: same clamps/acks, same frames, same banner except label; `--device auto`
resolves to air8201b; envelope in `/config` equals the old hardcoded numbers.

**`[hardware]` checklist — PlutoSDR:** `--device pluto` opens/arms/streams; 1-channel
UI (single pane, single PSD set, no diff/Δ/Q); envelope query returns sane numbers
(gain 0–73, freq ~0.325–3.8 GHz) and the capture form shows them; gain/center/rate acks
clamp to Pluto ranges; sustained streaming at 3.84 MS/s, behavior at 15.36 MS/s
(overflow logging, `_recover` after USB unplug/replug); `read_peripherals() == {}`
doesn't break anything; `--device auto` resolves to pluto on the Pi.

**Open questions:** exact SoapyPlutoSDR range-object API shape (`.minimum()`/
`.maximum()` vs tuple) — handled defensively; whether Pluto's AD9361 needs
`host_resample` for non-native rates (deferred — the LTE grid rates are all
AD9361-legal); whether the 61.44 MS/s grid point should ever be offered (it is outside
`RATES_HZ` today; unchanged).

## 8. Execution results (P3-6, 2026-07-07) and hardware validation checklist

All phases P3-0…P3-6 are implemented and committed. What was verified on the
dev box (`[demo]`, no radio, no striqt install): py_compile + `node --check`
green after every commit; `--device demo`, `--demo`, and `--channels 1|3`
boot and stream; `/config` carries `device` + `envelope`; the WS clamp
round-trip (`center 1e6 → 300e6`, `gain 99 → 10`) is byte-identical to
Phase 2b; frame headers carry `device` and the true channel list.

**Remaining human checks, in order:**

1. `[demo]` browser pass (any machine with a browser): default 2-ch view
   pixel-identical (colors/legend order/labels/diff trace); `--channels 1`
   single full-width pane with no diff/Δ/Q; `--channels 3` three panes with
   12 PSD series; CSV/PNG exports match the channel count; DAN/ARIC toggle
   and all four Analysis modes in each; title reads "Demo (synthetic IQ)".
2. `[hardware]` AIR8201B regression (Jetson): bare launch identical to
   Phase 2b (banner label aside); retune/gain acks unchanged; `/config`
   envelope equals the old hardcoded numbers (no live query by design);
   `--device auto` resolves to air8201b.
3. `[hardware]` PlutoSDR bring-up (Pi/laptop with SoapyPlutoSDR):
   `--device pluto` opens/arms/streams; log prints
   `[device] capability envelope updated: {...}` with sane values
   (freq ≈ 0.325–3.8 GHz, gain 0–73); 1-ch UI; capture-form tooltips show
   the queried ranges; sustained 3.84 MS/s, then try 15.36 MS/s (watch
   overflow logging); unplug/replug USB → `_recover` loop reopens;
   `--device auto` resolves to pluto.

## 9. Do NOT do

- Do **not** edit anything under `striqt/` — read-only vendored dependency.
- Do **not** build a second validator — the envelope feeds the existing tier-1 clamps;
  tier-2/tier-3 stay as-is; the existing ack is the only reporting channel.
- Do **not** add an in-browser device menu, runtime device switching, or hotplug logic.
- Do **not** add profiles beyond `air8201b`/`pluto`/`demo` (however tempting the zoo is).
- Do **not** attempt per-device calibration or Y-factor work.
- Do **not** merge the standalone scripts into the web server.
- Do **not** change the frame-header contract other than additively.
- Do **not** change default launch behavior — bare `python3 live/striqt_web_server.py`
  on the Jetson must be byte-identical to Phase 2b.
