# FIXLOG.md — AUDIT_REPORT.md fixes applied in the sandbox

One entry per fix (one commit each). Each entry: what changed + the **Verify** step
(copied verbatim from the finding), tagged `[demo]` (checkable with `--demo` on port 8001)
or `[hardware]` (needs the real AIR8201B SDR). Human runs the demo tests on the radio.

---

## Cluster 1 — Cadence (the slideshow fix)

### LV-W1 — Fix `rowsForWindow`: track radio nfft separately from header bins; hop-aware window→rows
**File:** `live/web/app.js`
**Changed:** Added module-level `radioNfft` (the requested radio FFT size), updated **only** by
the `#nfft-sel` change handler and never from frame headers. Renamed `curNfft`→`curBins`
(per-frame block bin count from the header) at every site set from a frame header;
`buildFreqsMHz`/`updateMeta`(FFT label)/PNG-caption keep `curBins`, while
`rowsForWindow`/`computeDisplayDepth`/`rowsForCurrentSettings` now use `radioNfft`. Rewrote
`rowsForWindow(fs, radioNfft, windowMs, hopFrac)` → `rows = windowMs/1000·fs/(radioNfft·hopFrac)`
clamped `[1,300]`, added `backendHopFrac()` (=1 for quicklook, 15/28 otherwise), and fixed
`updateMeta`'s `winMs = depthRows·radioNfft·hopFrac/curFs·1e3`. Threaded the new args through
the connect, rate-sel, nfft-sel, and win/mode call sites.

**Verify [demo]/[hardware]:** run `--demo --backend quicklook`; open dev tools → WS frames: the
first control message after connect must carry `rows` = `round(0.02·15.36e6/1024)=300` for
quicklook but ≈ 160 (hop-aware) for calibrated with the same settings; server log
`[config] rows: 12 -> N` shows the new N. On hardware: `[radio] IQ …` cadence and frame rate in
the meta line should jump from ~0.2 fps to multiple fps (with LV-W3).

### LV-W3 — Snap calibrated nfft to smooth 28-multiples {252, 504, 1008, 2016, 4032}
**File:** `live/striqt_web_server.py`
**Changed:** Replaced `aligned_nfft`'s `round(n/28)·28` (which produced slow non-power-of-2 sizes
like 1036=2²·7·37 and 2044=2²·7·73) with a nearest-by-absolute-distance snap to
`ALIGNED_NFFTS = (252, 504, 1008, 2016, 4032)` — all ÷28 (satisfies the 15/28 window-fill
integrality), all ÷12 (so `averaging_factor` returns 12 at every FFT setting — restores Dan's
consistent bin-averaging), all 7-smooth (2^a·3^b·7 → fast FFT). `averaging_factor` and the
`Spectrogram` spec construction are unchanged. Unit-checked locally: every aligned size is
divisible by 28 and 12; requested {256,512,1024,2048,4096} → {252,504,1008,2016,4032}, all with
averaging factor 12.

**Verify [demo]/[hardware]:** unit-check in a REPL: `all(n % 28 == 0 and n % 12 == 0 for n in
ALIGNED_NFFTS)` (done here, passes); run `--demo --backend calibrated` and confirm header `nfft`
(bins) is 83 at FFT 1024 and the meta fps rises; on hardware time one `calibrated_spectrogram`
call before/after (wrap with `time.perf_counter` temporarily) — expect ≥3× faster at 1008 vs
1036 for the same rows.

### LV-W2 — Right-size STFT rows/samples (stop computing ~1.87× the rows we display)
**File:** `live/striqt_web_server.py`
**Changed:** Added helper `calibrated_sample_count(nfft, rows) = rows·hop + (nfft-hop)`,
`hop = nfft·15//28`. `samples_needed` now uses it for the `calibrated`/`ssb` base (was
`nfft·rows`); quicklook stays `nfft·rows`; the SSB discovery-period `max()` floor is unchanged.
**Also right-sized `calibrated_spectrogram`'s internal `needed`** from `rows·nfft` to the same
helper — required, and beyond the samples-only handoff: `calibrated_spectrogram` re-derives
`needed = rows·nfft` and pads/truncates to it, so cutting only `samples_needed` would have
zero-padded the newest rows into garbage (the finding's own risk note assumes the reduced count
reaches `evaluate_spectrogram`). Proven locally against striqt's real striding row count
(`sliding_window_view(x, nfft)` stepped by `hop`): `floor((N-nfft)/hop)+1` == `rows` **exactly**
for every aligned nfft at rows ∈ {12,60,300}, with ~46% fewer samples/FFTs at 300 rows — so no
front-pad and frames are unchanged.

**Verify [demo]/[hardware]:** demo: displayed rows unchanged (header `rows` identical), server
CPU per frame drops (log timestamps); assert in a REPL that `evaluate_spectrogram` on the new
sample count returns ≥ rows STFT rows (it returns exactly `int((28/15)(N/nfft−1))+1`; with N as
above that is ≥ rows). [demo] confirms frames identical + header rows unchanged; [hardware]
confirms the per-frame CPU drop.

---

## Cluster 2 — Fidelity & labeling

### LV-F1 — Ship the true frequency axis in the header; use it in the client
**Files:** `live/striqt_web_server.py`, `live/web/app.js`
**Changed:** The calibrated axis was invented client-side (`(g−73.5)/147·fs`), wrong by up to
~103 kHz because striqt's DC-centered bin grouping drops edge bins. Now the server ships the true
axis. Backend functions (`db_spectrogram`/`calibrated_spectrogram`/`ssb_spectrogram`) return
`(blocks, meta)` with `meta = {fft_nfft, bin_avg}`; `compute_blocks` returns `(blocks, meta)`
(adds executed `backend`); new `build_header(cfg, blocks, meta, demo)` centralizes the header and
emits `fft_nfft`, `bin_avg`, `freqs_hz_step = bin_avg·fs/fft_nfft`, and `freqs_hz_f0` (=`-fs/2` for
quicklook's fftshift convention, `-(bins-1)/2·step` for the DC-centered calibrated/ssb grid).
`Acquirer.publish`/`DemoAcquirer._publish` and their call sites thread `meta`. Client:
`buildFreqsMHz` prefers header `f0`/`step` (falls back to the old formula for old servers); axis
rebuilds when `freqs_hz_step` changes. Verified locally: calibrated DC bin sits exactly at center;
+2.5 MHz demo tone lands within ½-step (60 kHz < 91 kHz) of 2.5 MHz; quicklook axis byte-identical
to the old formula (stays exact).

**Verify [demo]:** `--demo` (demo tones at exactly +2.5 MHz / −1.8 MHz on ch0): before, the
calibrated peak marker read ≈ 2.45 MHz offset; after, 2.5 MHz ± ½ step. Browser console: the
DC bin `freqsMHz[(curBins-1)/2]*1e6 - curCenter` must be ~0 in calibrated mode (with LV-W3's
83-bin grid the DC bin is index 41); quicklook stays exact.

### LV-F2 — Honest backend reporting (SSB fallback flag; disable SSB at incompatible rates)
**Files:** `live/striqt_web_server.py`, `live/web/app.js`
**Changed:** `compute_blocks` now pre-checks `ssb_grid_compatible(cfg.sample_rate)` when the
requested backend is `ssb`; when false it skips the doomed striqt call (which used to raise +
fall back every frame) and runs calibrated directly, recording `meta["backend"]="calibrated"`
and `meta["backend_requested"]="ssb"`. `build_header` (from LV-F1) already emits
`backend_requested` only when it differs from the executed `backend`. Client: `onFrame` warns
once per change (`setStatus(..., "warn")` + log) when `backend !== backend_requested`; the meta
"analysis" field now renders the executed `curBackend` (was the lying dropdown value); added
`ssbGridCompatible(fs)` (mirrors the server test) + `updateSsbOption()` which disables the SSB
`<option>` with a 420 kHz-grid tooltip when the current rate can't deliver it (called on tuning
change and at bootstrap). Confirmed locally: all four selectable rates (3.84/7.68/15.36/30.72)
are incompatible → SSB disabled + fallback reported; the 420 kHz rates (15.12/30.24) would be
compatible but aren't selectable.

**Verify [demo]:** `--demo --backend ssb`: header decodes to `backend:"calibrated"` with
`backend_requested:"ssb"`, the meta shows `calibrated` + a warning status line; switching rates
keeps it consistent; no striqt `ValueError` spam in the server output.

### LV-F3 — PSD "Mean" trace: average in linear power, not dB
**File:** `live/web/app.js`
**Changed:** `psdSeries` accumulated the mean trace as an arithmetic mean of dB values (biased
low for fluctuating signals). Now accumulates `10**(v/10)` per row and converts back with
`10·log10(max(mean/depth, 1e-20))`, mirroring the band monitor's (already-correct) linear
convention. Max/min traces are order statistics — left unchanged.
**Deferred (optional in the finding):** the same one-line change in the Qt viewers
(`striqt_standalone.py:1544`, `striqt_frontend_TCP.py:807`, `pluto_standalone.py`) — Qt-only,
not exercisable on port 8001. Ask if you want it applied.

**Verify [demo]:** band monitor "RX1 x.x dB" over a dragged band equals the average of the
fixed mean trace over the same band within ~0.5 dB (previously the mean trace read lower).

### LV-F4 — Truthful labels: FFT vs bins, hop-aware window, units on PSD/CSV/PNG
**File:** `live/web/app.js` (no `index.html` change needed — the y-axis label is JS-driven)
**Changed:** Added client state `curFftNfft`/`curBinAvg` (from the LV-F1 header fields).
(1) `updateMeta` prints FFT as `${radioNfft}→${curFftNfft} (${curBins} bins × ${curBinAvg})` for
calibrated/ssb and plain `${radioNfft}` for quicklook; the window ms is already the LV-W1
hop-aware value. (2) New `psdYLabel()` sets the uPlot y-axis to "Integrated power (dB rel. FS)"
for calibrated/ssb vs "Power (dB rel. FS / bin)" for quicklook (re-applied when `initUplot`
rebuilds on backend/tuning change). (3) `savePsdCsv` prepends `# backend=`, `# fft_nfft=`,
`# bin_avg=`, `# units=dB (uncalibrated, band-integrated|per-bin)` comment lines. (4) `exportPng`
caption uses the real FFT size and the true hop-aware window ms.

**Verify [demo]:** switch backends in `--demo` and confirm the label changes and the ~8.5 dB
level shift is now explained on-screen; export CSV and check the header comment lines.
