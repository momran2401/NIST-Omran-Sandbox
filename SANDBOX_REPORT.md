# NIST-Omran-Sandbox Report

All times are local Mountain time unless noted. Secrets are intentionally omitted.

## Phase 0 - Discovery

### 2026-07-02T12:56:52-06:00

Commands run:

- Local: `Get-Content .\SANDBOX_AGENT_PROMPT.md`
- Local: `Get-Location`, `Get-ChildItem -Force`, `git status --short`, `git remote -v` in `C:\Users\mao8\radio-sandbox-agent`
- Local: `git ls-remote https://github.com/momran2401/NIST-Omran-Sandbox HEAD`
- Radio read-only: `ssh -o BatchMode=yes sensor@24.128.57.203 "printf 'ssh_ok\n'; hostname; id; pwd"`
- Radio read-only discovery script covering original/sandbox directories, pixi python candidates, service units, cloudflared configs, listening ports, Tailscale status, disk, GitHub reachability, and port 8001 availability.
- Radio read-only grep/stat script covering the web server port/static/auth references, `/etc/radio-web.env` metadata, and pixi-run python resolution.
- Local: cloned `https://github.com/momran2401/NIST-Omran-Sandbox` to `C:\Users\mao8\NIST-Omran-Sandbox`, then inspected status, log, `.gitignore`, and file list.

Note: one Phase 0 smoke command briefly wrote and removed two temporary curl output files under `/tmp` on the radio. No persistent file remained, but this did not match the strict no-write discovery constraint.

Findings:

- SSH key auth works for `sensor@24.128.57.203`; host is `radio05`.
- Non-interactive sudo is not currently available for `sensor` (`sudo -n true` failed). Later systemd changes may block unless another authorized path exists.
- Original project directory exists at `/home/sensor/NIST-Omran`.
- Sandbox runtime directory does not yet exist on the radio: `/home/sensor/NIST-Omran-Sandbox` was missing.
- Original live files include `live/striqt_web_server.py`, `live/striqt_standalone.py`, and static files under `live/web/`.
- Original project is a git repo with origin `https://github.com/momran2401/NIST-Omran`.
- The pixi interpreter path is `/home/sensor/aggregate-directivity-acquisition/.pixi/envs/default/bin/python`.
- `striqt`, `fastapi`, and `uvicorn` import successfully under the pixi environment.
- The live service is `radio-web.service`.
- The cloudflared service is `cloudflared.service`; `cloudflared-update.service` also exists.
- `radio-web.service` has `WorkingDirectory=/home/sensor/NIST-Omran`, `EnvironmentFile=/etc/radio-web.env`, and starts the web server through `/home/sensor/.pixi/bin/pixi run --manifest-path /home/sensor/aggregate-directivity-acquisition/pixi.toml python /home/sensor/NIST-Omran/live/striqt_web_server.py`.
- `/etc/radio-web.env` exists but is root-only (`-rw------- 1 root root`), so the sandbox unit should reference it rather than copying values.
- The web server default port is `8000` in `live/striqt_web_server.py`.
- The static file directory is `live/web`, mounted by `StaticFiles(directory=str(WEB_DIR), html=True)`.
- Cloudflared config paths found: `/home/sensor/.cloudflared/config.yml` and `/etc/cloudflared/config.yml`.
- Current ingress maps `radio.mustafaomran.com` to `http://localhost:8000`, followed by `http_status:404`.
- Port `8000` is listening; port `8001` is free.
- Tailscale command exists but is logged out. No Tailscale IP is currently available.
- Free disk on `/home/sensor`: about `5.9G` available on a `28G` filesystem.
- Radio outbound internet to `https://github.com` works.
- `momran2401/NIST-Omran-Sandbox` is readable without credentials, so it appears public.
- The local sandbox checkout already contained a tracked `striqt/` tree. The prompt says to exclude `striqt` if it lives inside the project, but the hard guardrail says never modify or delete `striqt`; therefore it is being left untouched.

Phase 0 checks:

- Exact python path: `/home/sensor/aggregate-directivity-acquisition/.pixi/envs/default/bin/python`.
- Exact original live port: `8000`; sandbox target port `8001` is free.
- Service names: `radio-web.service`, `cloudflared.service`.
- Static-file dir: `/home/sensor/NIST-Omran/live/web`.
- Cloudflared config path for active unit: `/etc/cloudflared/config.yml`.
- Tailscale IP: unavailable because Tailscale is logged out.
- Persistent remote state: no intended modifications made in Phase 0.

## Rollback

If the sandbox service is installed later:

```bash
sudo systemctl disable --now radio-web-sandbox.service
sudo rm -f /etc/systemd/system/radio-web-sandbox.service
sudo systemctl daemon-reload
```

If Cloudflare public exposure is changed later, restore `/etc/cloudflared/config.yml` from the backup created in Phase 5b, then validate and reload cloudflared.

The original `/home/sensor/NIST-Omran` project and original `radio-web.service` / `cloudflared.service` are treated as read-only except for the explicit Phase 4.5 live-service validation window.

## Phase 1 - Sandbox and Git Flow

### 2026-07-02T13:05:00-06:00

Commands run:

- Local: `git push origin main` to publish the Phase 0 report.
- Local/radio read-only source stream: `ssh sensor@24.128.57.203 "tar -czf - -C /home/sensor/NIST-Omran ..."` piped to local `tar -xzf - -C C:/Users/mao8/NIST-Omran-Sandbox`.
- Local: `git status --short`, `git diff --stat`, `git diff --name-only`, `git diff --numstat`, `git diff --check`, `git status --porcelain=v2`.
- Local: `rg` secret scan excluding `striqt/**`, `SANDBOX_REPORT.md`, `*.pyc`, and `.git/**`.
- Local: updated `.gitignore` to ignore pixi envs, caches, capture/data outputs, env/credential material, Cloudflare tunnel material, and future `striqt/` additions.

Findings and decisions:

- The sandbox repository already had a history and a `main` branch, so it was cloned rather than reinitialized.
- The radio-to-local seed produced no non-`striqt` content differences beyond `.gitignore`; timestamp/line-ending noise was restored before commit.
- The secret scan found only source/documentation references and one example dev credential string already documented as non-deployed. No private keys, token assignments, Cloudflare credentials, or env files were found in the non-`striqt` tree.
- The existing tracked `striqt/` tree remains untouched due to the hard guardrail.

### 2026-07-02T13:16:00-06:00

Commands run:

- Radio write inside sandbox: `git clone https://github.com/momran2401/NIST-Omran-Sandbox /home/sensor/NIST-Omran-Sandbox`.
- Radio: started the unmodified sandbox web server under pixi in `--demo` mode on `127.0.0.1:8001` with temporary non-secret auth values.
- Radio: `curl` checks against `http://127.0.0.1:8001/`.
- Radio: short pixi Python WebSocket client using a Basic Auth header, decoding the binary frame header.
- Radio: `ss -ltnp` check confirmed port `8001` was released after baseline.

Baseline results:

- Runtime clone exists at `/home/sensor/NIST-Omran-Sandbox`, commit `18c7a8f`.
- Default `--demo` / calibrated backend starts and serves HTTP, but emits no frames because SciPy fails to load against the system `libstdc++`: missing `GLIBCXX_3.4.29`.
- The unmodified server's built-in `--backend quicklook` mode passed the Phase 1 stream baseline on port `8001`:
  - `server_ready=1`
  - no-auth HTTP: `401`
  - auth HTTP with temporary test creds: `200`
  - WebSocket frames in about 5 seconds: `41`
  - first frame header: `demo=true`, `channels=[0,1]`, `shape=[12,1024]`, `nfft=1024`, `rows=12`, `fs=15360000.0`
- Port `8001` was free after the baseline process stopped.

Phase 1 checks:

- Sandbox server starts on `8001` under the pixi environment.
- Auth gate works with env-provided credentials.
- WebSocket receives demo frames in unmodified quicklook mode.
- Original `/home/sensor/NIST-Omran` was not modified.
- Known issue carried into Phase 3: the current calibrated demo backend fails due `libstdc++`/SciPy linkage before producing frames.

## Phase 2 - striqt API Read-Out

### 2026-07-02T13:38:00-06:00

Commands run:

- Local read-only: `rg` over `striqt/src/striqt` for `evaluate_spectrogram`, `json_schema`, `cellular_5g_ssb_spectrogram`, and relevant config keys.
- Local read-only: numbered excerpts from `striqt/src/striqt/analysis/measurements/shared.py`, `_spectrogram.py`, `_cellular_5g_ssb_spectrogram.py`, `analysis/specs/structs.py`, `analysis/specs/helpers.py`, `analysis/lib/register.py`, `sensor/specs/structs.py`, `sensor/specs/helpers.py`, `sensor/bindings.py`, and `sensor/lib/bindings.py`.
- Radio read-only: pixi Python introspection of installed module paths, installed call signatures, and `json_schema(bindings.air8201b.sweep_spec)` shape.
- Radio read-only: grep excerpts from installed `site-packages/striqt/sensor/lib/bindings.py`, `sensor/bindings.py`, and `analysis/specs/helpers.py` where the installed binding API differs from the checked-out `striqt/` tree.

Spectrogram API facts:

- The lower-level calibrated spectrogram routine is `striqt.analysis.measurements.shared.evaluate_spectrogram(iq, capture, spec, *, dtype='float32', limit_digits=None, dB=True) -> tuple[array, dict]`; it directly returns the spectrogram array plus attrs. Citation: `striqt/src/striqt/analysis/measurements/shared.py:122-148`.
- `evaluate_spectrogram` validates `sample_rate / frequency_resolution`, computes `nfft`, overlap, zero-fill, optional frequency-bin averaging, optional time-bin averaging, optional LO nulling, optional stopband trim, and ENBW/noise-bandwidth attrs. Citations: `shared.py:162-242`.
- The current server already calls this lower-level function in `calibrated_spectrogram`, but only with `window='hann'` and `frequency_resolution=sample_rate/nfft`; it does not enable the 5G/SSB averaging parameters. Citations: `live/striqt_web_server.py:559-610`.
- The public `spectrogram` measurement constructs `specs.Spectrogram` from kwargs and calls `shared.evaluate_spectrogram(..., dB=True, limit_digits=2, dtype='float16')`. Citation: `striqt/src/striqt/analysis/measurements/_spectrogram.py:63-84`.
- The `as_xarray=False` knob is not an argument to `shared.evaluate_spectrogram`; it is added by the measurement registry wrapper. The wrapper accepts `as_xarray` and returns `(data, attrs)` when false. Citation: `striqt/src/striqt/analysis/lib/register.py:341-365`.
- Dan's "use `evaluate_spectrogram` with `as_xarray=False`" is therefore slightly imprecise: use `as_xarray=False` on registered measurements, or call `shared.evaluate_spectrogram` directly without `as_xarray`.

Symbol-aligned / binned spectrogram facts:

- `cellular_5g_ssb_spectrogram` builds a `specs.Spectrogram` with `frequency_resolution=subcarrier_spacing/2`, `fractional_overlap=13/28`, `window_fill=15/28`, `integration_bandwidth=subcarrier_spacing`, and `trim_stopband=False`, then calls `shared.evaluate_spectrogram`. Citations: `striqt/src/striqt/analysis/measurements/_cellular_5g_ssb_spectrogram.py:99-116`.
- It computes `symbol_count = round(28 * subcarrier_spacing / 15e3)`, masks to the first two slots of each discovery period, truncates frequency to the SSB sample rate, and reshapes into `(channels, ssb_index, symbol, frequency)`. Citations: `_cellular_5g_ssb_spectrogram.py:101-132`.
- Frequency-bin averaging is driven by `integration_bandwidth / frequency_resolution` and performed with `sw.binned_mean(..., axis=2, fft=True)`, then scaled from mean to sum. Citations: `shared.py:180-188` and `shared.py:223-228`.
- Time averaging is driven by `time_aperture / hop_period` for the generic spectrogram path. Citations: `shared.py:191-200` and `shared.py:229-230`.
- Relevant schema/config keys are defined on `FrequencyAnalysisSpecBase`: `window`, `frequency_resolution`, `fractional_overlap`, `window_fill`, `integration_bandwidth`, `trim_stopband`, and `lo_bandstop`. Citation: `striqt/src/striqt/analysis/specs/structs.py:140-166`.
- The SSB-specific schema keys include `subcarrier_spacing`, `sample_rate`, `discovery_periodicity`, `frequency_offset`, `max_block_count`, `window`, and `lo_bandstop`. Citation: `striqt/src/striqt/analysis/specs/structs.py:169-190`.

Schema API facts:

- The schema helper is `striqt.analysis.specs.helpers.json_schema(cls)`, which calls `msgspec.json.schema(cls, schema_hook=_schema_hook)`. Citation: `striqt/src/striqt/analysis/specs/helpers.py:230-240`; installed citation: `site-packages/striqt/analysis/specs/helpers.py:226-236`.
- The active installed binding API differs from the checked-out source: installed `bindings.air8201b` is a `SensorBinding` with `.sweep_spec`; the radio should call `json_schema(bindings.air8201b.sweep_spec)`. Installed citations: `site-packages/striqt/sensor/lib/bindings.py:77-91`, `:113-138`, and `site-packages/striqt/sensor/bindings.py:157-164`.
- Installed `json_schema(bindings.air8201b.sweep_spec)` returns a top-level schema with `'$ref'` and `'$defs'`; the concrete object schema is under `$defs.air8201b`.
- `$defs.air8201b` includes `sensor_binding`, `source`, `captures`, `analysis`, `extensions`, `description`, `loops`, `adjust_captures`, `peripherals`, `sink`, `options`, and `mock_source`; `sensor_binding` is the only required top-level field in the raw schema.
- `$defs.Air8201BSourceSpec` includes source fields such as `master_clock_rate`, `trigger_strobe`, `signal_trigger`, `array_backend`, `calibration`, `time_source`, `time_sync_at`, `clock_source`, `receive_retries`, `adc_overload_limit`, `if_overload_limit`, and `gapless`.
- `$defs.SoapyCapture` requires `port`, `center_frequency`, and `gain`, and has defaults for `duration`, `sample_rate`, `analysis_bandwidth`, `lo_shift`, `host_resample`, `backend_sample_rate`, and `adjust_analysis`.
- `$defs.Analysis` exposes optional measurement configs including `spectrogram`, `power_spectral_density`, `cellular_5g_pss_sync`, and `cellular_5g_ssb_spectrogram`.

Phase 3 wiring plan:

- Keep the current Acquirer/Computer split unchanged; only replace `calibrated_spectrogram` and frame metadata/control plumbing as needed.
- Add an SSB/averaged spectrogram compute path using installed `striqt.analysis.measurements.cellular_5g_ssb_spectrogram(..., as_xarray=False, subcarrier_spacing=30000.0, sample_rate=7680000.0, discovery_periodicity=0.02, frequency_offset=0.0, max_block_count=None, window='blackmanharris', lo_bandstop=120000.0)`.
- Collapse the SSB output's `(ssb_index, symbol)` axes into display rows and pad/crop to the existing `(channels, rows, nfft)` frame contract; if the SSB frequency-bin count differs from the UI `nfft`, update the frame header `nfft` to the actual width or resample only if necessary.
- Keep `quicklook` as fallback and preserve `--quantize`.
- Add a minimal analysis selector in the existing UI/control channel to request spectrogram, PSD-derived display, or SSB spectrogram, but make the improved calibrated/SSB path the default target for `calibrated`.

Phase 4 wiring plan:

- Add `GET /schema` that returns `json_schema(bindings.air8201b.sweep_spec)` from the installed pixi `striqt`.
- Render the existing dashboard form from `$defs.air8201b`, resolving `$ref`s needed for `source`, `captures[0]`, and selected `analysis` configs.
- Validate/clamp server-side using the schema-derived field metadata and the current conservative operational bounds.
- Accept uploaded sweep JSON in the Appendix A shape, seed visible form fields, and preserve hidden lower-level values in client state.
- Send dynamic capture/source updates over the existing WebSocket control path; capture settings retune live, source settings require reconnect.

Phase 2 checks:

- Real spectrogram and schema APIs are documented with citations.
- Dan's `as_xarray` statement is fact-checked against code.
- Concrete plans for Phase 3 and Phase 4 are recorded.
