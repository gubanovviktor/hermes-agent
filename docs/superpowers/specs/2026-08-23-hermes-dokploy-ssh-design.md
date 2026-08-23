# Hermes Agent on Dokploy with Direct SSH

## Goal

Run Hermes Agent as a separate, isolated Dokploy deployment, with persistent Hermes state and direct SSH access into the Hermes container. The container must not use host networking and must not expose Dokploy's other services or host filesystem.

## Source Context

- Upstream project: `https://github.com/NousResearch/hermes-agent`
- Upstream provides a production `Dockerfile` and `docker-compose.yml`.
- Upstream Docker Compose uses persistent Hermes data under `/opt/data` and runs Hermes via the gateway command.
- Upstream warns that the dashboard should stay localhost-only and be accessed through SSH tunneling if remote access is needed.
- Dokploy supports Docker Compose deployments with explicit volumes and published ports, which fits this service better than a simple HTTP application deployment.

## Chosen Approach

Create a small Dokploy-specific wrapper around the upstream Hermes source:

- Build from the upstream GitHub source so the deployment does not depend on a separately published runtime image.
- Add `openssh-server`.
- Create or reuse a non-root `hermes` runtime user.
- Store all Hermes state in a dedicated Docker volume mounted at `/opt/data`.
- Publish a single external SSH port: `2223:22`.
- Run Hermes and `sshd` in the same container under a controlled entrypoint that preserves the upstream `/init` / s6 startup path.

This gives direct SSH into the Hermes environment while keeping the deployment isolated from the Dokploy host and other projects.

## Architecture

The Dokploy project will contain one Docker Compose stack:

- `hermes`: the only service.
- `hermes-data`: named Docker volume for `/opt/data`.
- Private Docker network created by Dokploy/Compose.
- External published TCP port for SSH only.

The service must not use `network_mode: host`. If Hermes dashboard or API access is needed later, it must be added deliberately behind authentication or reached through an SSH tunnel.

## SSH Access Model

SSH access will be key-only:

- Password login disabled.
- Root login disabled.
- Public key injected through an environment variable or mounted authorized-keys file.
- `sshd` listens on container port `22`.
- Dokploy publishes host port `2223`.

Expected connection shape:

```bash
ssh -p 2223 hermes@<dokploy-host-or-domain>
```

Inside the SSH session, the user lands in the Hermes environment and can run:

```bash
hermes
hermes gateway
hermes doctor
```

## Container Runtime

The container must preserve upstream Hermes initialization because the official Dockerfile uses s6 overlay and setup hooks for UID/GID remapping, profile reconciliation, dashboard toggles, and service supervision.

Implementation must avoid replacing upstream startup blindly. The wrapper must either:

- add `sshd` as an additional s6-supervised service, or
- use a tiny entrypoint that starts `sshd` and then execs the upstream `/init` command exactly as expected.

The first option is cleaner if the upstream s6 layout is easy to extend. The second option is acceptable if tested carefully.

## Persistent State

Persist `/opt/data` through a named Docker volume:

```yaml
volumes:
  hermes-data:
```

The container environment sets:

```yaml
HERMES_UID: "10000"
HERMES_GID: "10000"
```

If file ownership issues appear, use the upstream UID/GID remapping mechanism rather than ad hoc `chmod 777`.

## Secrets and Configuration

No API keys should be baked into the image.

Secrets are provided through Dokploy environment variables or secret management:

- model provider keys, if not using Hermes setup interactively
- messaging gateway credentials, if Telegram/Discord/etc. are enabled later
- optional `API_SERVER_KEY`, only if the Hermes API server is deliberately exposed
- SSH authorized public key, or a mounted authorized-keys file

The SSH private key stays only on the client machine.

## Network Exposure

Initial exposure:

- Publish SSH only: host port `2223` to container port `22`.

Do not expose initially:

- Hermes dashboard
- Hermes OpenAI-compatible API server
- arbitrary container ports
- Docker socket
- host filesystem paths

If dashboard access is needed, use:

```bash
ssh -p 2223 -L 9119:127.0.0.1:9119 hermes@<host>
```

Then open `http://127.0.0.1:9119` locally, assuming the dashboard is running inside the container on localhost.

## Deployment Steps

1. Prepare a minimal deployment repository or Dokploy Git source containing:
   - `Dockerfile.dokploy`
   - `docker-compose.yml`
   - `sshd_config`
   - s6 service files when extending upstream supervision, or an entrypoint wrapper when that route proves safer
2. Configure Dokploy as a Docker Compose deployment.
3. Set environment variables in Dokploy.
4. Deploy and verify that the container starts.
5. SSH into the container with key auth.
6. Run `hermes doctor`.
7. Run `hermes setup` or `hermes setup --portal`.
8. Confirm Hermes state survives container restart.

## Verification

Minimum verification before calling the deployment done:

- `docker compose config` succeeds locally or in CI.
- The image builds.
- The container starts cleanly.
- `sshd -t` succeeds.
- SSH key login works.
- Password SSH login fails.
- Root SSH login fails.
- `hermes doctor` runs inside the SSH session.
- `/opt/data` persists after restart.
- No host network mode is used.
- Only the intended SSH port is published.

## Risks

- Upstream startup is non-trivial because Hermes uses s6 overlay. The wrapper must preserve it.
- Running SSH inside an app container increases exposed surface area, so SSH must be locked down to key-only login.
- Publishing a raw TCP port through Dokploy may need explicit host firewall rules outside HTTP/Traefik routing.
- If the upstream Docker image is not published in a stable registry, the Dokploy image may need to build from GitHub source, which can be slower.

## Non-Goals

- No host-level SSH workflow.
- No access to other Dokploy projects.
- No Docker socket mounted into the Hermes container.
- No public dashboard or public Hermes API server in the initial deployment.
- No automatic setup of Telegram/Discord/Slack gateways unless requested separately.

## Port Rule

Use SSH host port `2223`. Port `2222` was occupied on the Dokploy host during implementation, so the deployment moved to the nearest clear candidate instead of replacing another service.
