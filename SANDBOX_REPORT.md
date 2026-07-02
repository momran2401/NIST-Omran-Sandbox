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
