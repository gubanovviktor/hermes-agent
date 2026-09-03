# Dokploy SSH Deployment

This deployment runs Hermes Agent as one isolated Docker Compose service on Dokploy and exposes only SSH.

## Dokploy Settings

- Deployment type: Docker Compose
- Compose file: `docker-compose.yml`
- Published TCP port: `${HERMES_SSH_HOST_PORT:-2223}` to container port `22`
- Persistent volume: `hermes-data:/opt/data`

## Required Environment

Set these in Dokploy:

```dotenv
HERMES_SSH_HOST_PORT=2223
HERMES_UID=10000
HERMES_GID=10000
HERMES_SSH_AUTHORIZED_KEY=<your public SSH key>
```

Use a public key, not a private key. A valid value starts with `ssh-ed25519`, `ssh-rsa`, or `ecdsa-sha2-*`.

## SSH

```bash
ssh -p 2223 hermes@<dokploy-host-or-domain>
```

Inside the container:

```bash
hermes doctor
hermes setup
hermes
```

## Dashboard

The web dashboard (config editor, API-key manager, session browser) is **not exposed
publicly** by this deployment — only SSH is. Reach it through an SSH tunnel.

### Why the supervised service stays down

The image ships an s6 `dashboard` service, but in this Dokploy deployment it does not
start on its own:

1. Dokploy's env-panel variables are only used for `${...}` interpolation in
   `docker-compose.yml`. `HERMES_DASHBOARD*` are not referenced in the compose
   `environment:` block, so they never reach the container and the service's `run`
   script exits early (`s6-svstat /run/service/dashboard` shows `down`).
2. Even when the flag is passed, the `run` script calls `hermes dashboard` without
   `--skip-build`, and the sealed image cannot `npm install` at runtime. The SPA
   bundle is already prebuilt at `/opt/hermes/hermes_cli/web_dist/`, so `--skip-build`
   is required.

### Working access — SSH config alias (recommended)

Add to your local `~/.ssh/config`:

```sshconfig
Host hermes-dash
  HostName <dokploy-host>
  Port 2223
  User hermes
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  LocalForward 9119 127.0.0.1:9119
  RemoteCommand hermes dashboard --host 127.0.0.1 --port 9119 --no-open --skip-build
```

Then:

```bash
ssh hermes-dash
```

Wait ~10 s for the `HERMES_DASHBOARD_READY port=9119` line, then open
<http://localhost:9119>. `Ctrl-C` stops the dashboard and closes the tunnel.

### One-off equivalent

```bash
ssh -p 2223 -i ~/.ssh/id_ed25519 -L 9119:127.0.0.1:9119 hermes@<dokploy-host> \
  "hermes dashboard --host 127.0.0.1 --port 9119 --no-open --skip-build"
```

### Making it a real supervised service

To have s6 keep the dashboard up across restarts, two source changes are needed on the
deployment branch:

- `docker-compose.yml` → add to `environment:`:
  ```yaml
  HERMES_DASHBOARD: "${HERMES_DASHBOARD:-1}"
  HERMES_DASHBOARD_HOST: "${HERMES_DASHBOARD_HOST:-127.0.0.1}"
  HERMES_DASHBOARD_PORT: "${HERMES_DASHBOARD_PORT:-9119}"
  ```
- `docker/s6-rc.d/dashboard/run` → append `--skip-build` to the final
  `exec … hermes dashboard …` line.

Keep `HERMES_DASHBOARD_HOST=127.0.0.1` so the bind stays loopback-only (the non-loopback
auth gate never engages); the SSH tunnel remains the security boundary.

## Browser tools

The agent's `browser_*` tools need a real Chromium. The published `--only-shell`
Playwright install is not enough, and the sealed image cannot install a browser at
runtime (`/opt/hermes` is read-only, `HERMES_DISABLE_LAZY_INSTALLS=1`). This branch
bakes what's needed at build time:

- `Dockerfile` — full `npx playwright install --with-deps chromium` (no `--only-shell`)
  plus `npm install -g agent-browser@^0.26.0` so the CLI resolves offline.
- `docker-compose.yml` `command:` — on every start:
  - `hermes config set browser.backend off` — drive the baked Chromium through the
    built-in `browser_*` tools instead of the Browser Use CLI (which would try to
    fetch itself at runtime).
  - `hermes config set browser.record_sessions true` — auto-record each browser
    session to a WebM under `/opt/data/browser_recordings/` (72 h retention).

`docker/stage2-hook.sh` discovers the Chromium binary under
`$PLAYWRIGHT_BROWSERS_PATH` at boot and exports `AGENT_BROWSER_EXECUTABLE_PATH`.

Screenshots (`browser_vision`, `screenshot`) and WebM session recording both work
headless — no display needed. `computer_use` (full desktop control / desktop screen
recording) additionally needs Xvfb + a window manager, which this image does not ship.

Verify after deployment:

```bash
ssh -p 2223 hermes@<dokploy-host-or-domain> \
  'hermes doctor | grep -iA2 -E "playwright|browser tools"; hermes config get browser'
```

## Security Checks

After deployment:

```bash
ssh -p 2223 hermes@<dokploy-host-or-domain> 'whoami && hermes doctor'
ssh -p 2223 root@<dokploy-host-or-domain>
```

The first command should log in as `hermes`. The second command should fail because root login is disabled.
