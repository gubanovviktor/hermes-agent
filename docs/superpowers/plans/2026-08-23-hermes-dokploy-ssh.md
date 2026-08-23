# Hermes Dokploy SSH Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Hermes Agent as an isolated Dokploy Docker Compose service with persistent state and key-only direct SSH into the container.

**Architecture:** Clone the upstream Hermes source into this workspace, then add the smallest Dokploy-specific runtime layer directly to that source tree: `openssh-server`, an s6-supervised SSH service, a cont-init script that installs the authorized key from an environment variable, and a Compose file that publishes only SSH port `2222`. The deployment keeps upstream Hermes `/init` and s6 behavior intact, mounts a named `/opt/data` volume, and avoids host networking.

**Tech Stack:** Dokploy Docker Compose, Docker, Debian 13-based Hermes image, s6-overlay, OpenSSH server, shell scripts.

**Spec:** `docs/superpowers/specs/2026-08-23-hermes-dokploy-ssh-design.md`

## Global Constraints

- The service must not use `network_mode: host`.
- Only host port `2222` to container port `22` is published initially.
- SSH is key-only: password login disabled and root login disabled.
- Hermes state persists in a named Docker volume mounted at `/opt/data`.
- No Docker socket is mounted into the Hermes container.
- No host filesystem paths are mounted into the Hermes container.
- No public Hermes dashboard or public Hermes API server is exposed in the initial deployment.
- No API keys or private SSH keys are baked into the image.
- Preserve upstream Hermes entrypoint behavior: `/opt/hermes/docker/entrypoint-dispatch.sh` remains the image `ENTRYPOINT`.
- If host port `2222` is occupied on Dokploy, stop and choose the nearest clear high port instead of replacing another service.

---

## File Structure

- `README.md`: upstream Hermes documentation after clone; do not rewrite for this deployment.
- `Dockerfile`: upstream Hermes Dockerfile, modified minimally to install `openssh-server`, copy SSH configuration, and copy the authorized-key init script.
- `docker-compose.yml`: Dokploy deployment compose file, replacing upstream host-network compose with an isolated one-service stack.
- `docker/sshd_config`: OpenSSH server hardening config.
- `docker/cont-init.d/10-hermes-ssh-authorized-key`: s6 cont-init script that writes `HERMES_SSH_AUTHORIZED_KEY` into `/opt/data/.ssh/authorized_keys`.
- `docker/s6-rc.d/hermes-sshd/type`: s6 service type declaration.
- `docker/s6-rc.d/hermes-sshd/run`: s6 run script for `sshd`.
- `docker/s6-rc.d/user/contents.d/hermes-sshd`: empty marker that adds `hermes-sshd` to the upstream `user` bundle.
- `docs/deploy/dokploy-ssh.md`: operator notes for Dokploy env vars, SSH command, dashboard tunneling, and verification.

---

### Task 1: Materialize Upstream Source

**Files:**
- Create/Populate: repository files from `https://github.com/NousResearch/hermes-agent`
- Preserve: `docs/superpowers/specs/2026-08-23-hermes-dokploy-ssh-design.md`
- Preserve: `docs/superpowers/plans/2026-08-23-hermes-dokploy-ssh.md`

**Interfaces:**
- Consumes: empty local workspace plus the saved spec and plan.
- Produces: a git checkout of upstream Hermes in `/Users/victorgubanov/Projects/hermes-agent` with the spec and plan restored.

- [ ] **Step 1: Save local planning docs outside the soon-to-be cloned repo**

Run:

```bash
mkdir -p /tmp/hermes-agent-planning-backup
cp -R docs/superpowers /tmp/hermes-agent-planning-backup/
```

Expected: `/tmp/hermes-agent-planning-backup/superpowers/specs/2026-08-23-hermes-dokploy-ssh-design.md` and `/tmp/hermes-agent-planning-backup/superpowers/plans/2026-08-23-hermes-dokploy-ssh.md` exist.

- [ ] **Step 2: Clone upstream into the empty workspace**

Run from `/Users/victorgubanov/Projects`:

```bash
rmdir hermes-agent
git clone https://github.com/NousResearch/hermes-agent.git hermes-agent
```

Expected: `/Users/victorgubanov/Projects/hermes-agent/.git` exists and `git status --short` prints nothing.

- [ ] **Step 3: Restore planning docs into the cloned repository**

Run:

```bash
cd /Users/victorgubanov/Projects/hermes-agent
mkdir -p docs/superpowers
cp -R /tmp/hermes-agent-planning-backup/superpowers/* docs/superpowers/
```

Expected: both planning docs exist inside the cloned repo.

- [ ] **Step 4: Capture upstream baseline**

Run:

```bash
cd /Users/victorgubanov/Projects/hermes-agent
git rev-parse --short HEAD
git status --short
```

Expected: first command prints the upstream commit; second command shows only `docs/superpowers/...` as untracked or modified.

- [ ] **Step 5: Commit planning docs**

Run:

```bash
git add docs/superpowers/specs/2026-08-23-hermes-dokploy-ssh-design.md docs/superpowers/plans/2026-08-23-hermes-dokploy-ssh.md
git commit -m "docs: plan hermes dokploy ssh deployment"
```

Expected: commit succeeds.

---

### Task 2: Add SSH Runtime Files

**Files:**
- Create: `docker/sshd_config`
- Create: `docker/cont-init.d/10-hermes-ssh-authorized-key`
- Create: `docker/s6-rc.d/hermes-sshd/type`
- Create: `docker/s6-rc.d/hermes-sshd/run`
- Create: `docker/s6-rc.d/user/contents.d/hermes-sshd`

**Interfaces:**
- Consumes: upstream s6 service tree under `docker/s6-rc.d/` and upstream `/opt/data` Hermes home.
- Produces: an s6-supervised `hermes-sshd` service and an init-time authorized-key installer.

- [ ] **Step 1: Create the SSH daemon config**

Create `docker/sshd_config` with exactly:

```sshconfig
Port 22
Protocol 2
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
PidFile /run/sshd.pid
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
AllowTcpForwarding yes
AllowAgentForwarding no
PermitTunnel no
UsePAM no
PrintMotd no
ClientAliveInterval 60
ClientAliveCountMax 3
AllowUsers hermes
Subsystem sftp internal-sftp
```

- [ ] **Step 2: Create the authorized-key init script**

Create `docker/cont-init.d/10-hermes-ssh-authorized-key` with exactly:

```sh
#!/command/with-contenv sh
set -eu

if [ -z "${HERMES_SSH_AUTHORIZED_KEY:-}" ]; then
    echo "[hermes-ssh] HERMES_SSH_AUTHORIZED_KEY is required" >&2
    exit 1
fi

case "${HERMES_SSH_AUTHORIZED_KEY}" in
    ssh-ed25519\ *|ssh-rsa\ *|ecdsa-sha2-nistp256\ *|ecdsa-sha2-nistp384\ *|ecdsa-sha2-nistp521\ *)
        ;;
    *)
        echo "[hermes-ssh] HERMES_SSH_AUTHORIZED_KEY does not look like a supported public key" >&2
        exit 1
        ;;
esac

install -d -m 0700 -o hermes -g hermes /opt/data/.ssh
printf '%s\n' "${HERMES_SSH_AUTHORIZED_KEY}" > /opt/data/.ssh/authorized_keys
chown hermes:hermes /opt/data/.ssh/authorized_keys
chmod 0600 /opt/data/.ssh/authorized_keys
```

- [ ] **Step 3: Create the s6 service type**

Create `docker/s6-rc.d/hermes-sshd/type` with exactly:

```text
longrun
```

- [ ] **Step 4: Create the s6 SSH run script**

Create `docker/s6-rc.d/hermes-sshd/run` with exactly:

```sh
#!/command/with-contenv sh
set -eu

mkdir -p /run/sshd
chmod 0755 /run/sshd

if [ ! -f /etc/ssh/ssh_host_ed25519_key ]; then
    ssh-keygen -A
fi

exec /usr/sbin/sshd -D -e -f /etc/ssh/sshd_config
```

- [ ] **Step 5: Add the service to the s6 user bundle**

Create an empty marker file:

```bash
mkdir -p docker/s6-rc.d/user/contents.d
: > docker/s6-rc.d/user/contents.d/hermes-sshd
```

Expected: `find docker/s6-rc.d/hermes-sshd docker/s6-rc.d/user/contents.d/hermes-sshd -maxdepth 1 -type f -print` lists the three new service files.

- [ ] **Step 6: Make scripts executable**

Run:

```bash
chmod 0755 docker/cont-init.d/10-hermes-ssh-authorized-key docker/s6-rc.d/hermes-sshd/run
```

Expected: `test -x docker/cont-init.d/10-hermes-ssh-authorized-key && test -x docker/s6-rc.d/hermes-sshd/run` succeeds.

- [ ] **Step 7: Commit SSH runtime files**

Run:

```bash
git add docker/sshd_config docker/cont-init.d/10-hermes-ssh-authorized-key docker/s6-rc.d/hermes-sshd docker/s6-rc.d/user/contents.d/hermes-sshd
git commit -m "feat: add supervised ssh access for hermes container"
```

Expected: commit succeeds.

---

### Task 3: Patch Dockerfile for SSH

**Files:**
- Modify: `Dockerfile`

**Interfaces:**
- Consumes: files from Task 2.
- Produces: a Hermes image that contains `openssh-server`, the hardened SSH config, and the authorized-key init hook while preserving the upstream entrypoint.

- [ ] **Step 1: Add `openssh-server` to the existing apt install list**

In `Dockerfile`, find the existing package list that includes:

```dockerfile
openssh-client docker-cli xz-utils
```

Change it to:

```dockerfile
openssh-client openssh-server docker-cli xz-utils
```

Expected: `rg -n "openssh-client openssh-server docker-cli xz-utils" Dockerfile` finds one line.

- [ ] **Step 2: Copy the SSH config and init script**

In `Dockerfile`, near the existing `COPY --chmod=0755 docker/cont-init.d/... /etc/cont-init.d/...` lines, add:

```dockerfile
COPY docker/sshd_config /etc/ssh/sshd_config
COPY --chmod=0755 docker/cont-init.d/10-hermes-ssh-authorized-key /etc/cont-init.d/10-hermes-ssh-authorized-key
```

Expected: `rg -n "10-hermes-ssh-authorized-key|sshd_config" Dockerfile` finds both copy lines.

- [ ] **Step 3: Verify the upstream entrypoint remains unchanged**

Run:

```bash
tail -n 5 Dockerfile
```

Expected output includes:

```dockerfile
ENTRYPOINT [ "/opt/hermes/docker/entrypoint-dispatch.sh" ]
CMD [ ]
```

- [ ] **Step 4: Commit Dockerfile changes**

Run:

```bash
git add Dockerfile
git commit -m "build: install ssh server in hermes image"
```

Expected: commit succeeds.

---

### Task 4: Replace Compose with Dokploy-Isolated Compose

**Files:**
- Modify: `docker-compose.yml`
- Create: `.env.dokploy.example`

**Interfaces:**
- Consumes: Dockerfile changes from Task 3.
- Produces: a Dokploy-ready Compose stack with a single isolated service and named volume.

- [ ] **Step 1: Replace `docker-compose.yml`**

Replace the file with exactly:

```yaml
services:
  hermes:
    build:
      context: .
      dockerfile: Dockerfile
    image: hermes-agent-dokploy-ssh:latest
    restart: unless-stopped
    ports:
      - "${HERMES_SSH_HOST_PORT:-2222}:22"
    volumes:
      - hermes-data:/opt/data
    environment:
      HERMES_UID: "${HERMES_UID:-10000}"
      HERMES_GID: "${HERMES_GID:-10000}"
      HERMES_SSH_AUTHORIZED_KEY: "${HERMES_SSH_AUTHORIZED_KEY}"
    command: ["sleep", "infinity"]

volumes:
  hermes-data:
```

Expected: no `network_mode: host` appears in the file.

- [ ] **Step 2: Add an environment example**

Create `.env.dokploy.example` with exactly:

```dotenv
HERMES_SSH_HOST_PORT=2222
HERMES_UID=10000
HERMES_GID=10000
HERMES_SSH_AUTHORIZED_KEY=ssh-ed25519 AAAA_REPLACE_WITH_YOUR_PUBLIC_KEY victor@example
```

- [ ] **Step 3: Validate Compose syntax**

Run:

```bash
HERMES_SSH_AUTHORIZED_KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFakeKeyForComposeValidationOnly user@example" docker compose config >/tmp/hermes-compose-config.yml
```

Expected: command exits 0 and `/tmp/hermes-compose-config.yml` contains `2222:22` or the equivalent expanded port mapping.

- [ ] **Step 4: Commit Compose files**

Run:

```bash
git add docker-compose.yml .env.dokploy.example
git commit -m "deploy: add isolated dokploy compose stack"
```

Expected: commit succeeds.

---

### Task 5: Add Operator Documentation

**Files:**
- Create: `docs/deploy/dokploy-ssh.md`

**Interfaces:**
- Consumes: Compose env names and SSH behavior from Tasks 2-4.
- Produces: deployment and operations guide.

- [ ] **Step 1: Create deployment docs**

Create `docs/deploy/dokploy-ssh.md` with exactly:

```markdown
# Dokploy SSH Deployment

This deployment runs Hermes Agent as one isolated Docker Compose service on Dokploy and exposes only SSH.

## Dokploy Settings

- Deployment type: Docker Compose
- Compose file: `docker-compose.yml`
- Published TCP port: `${HERMES_SSH_HOST_PORT:-2222}` to container port `22`
- Persistent volume: `hermes-data:/opt/data`

## Required Environment

Set these in Dokploy:

```dotenv
HERMES_SSH_HOST_PORT=2222
HERMES_UID=10000
HERMES_GID=10000
HERMES_SSH_AUTHORIZED_KEY=<your public SSH key>
```

Use a public key, not a private key. A valid value starts with `ssh-ed25519`, `ssh-rsa`, or `ecdsa-sha2-*`.

## SSH

```bash
ssh -p 2222 hermes@<dokploy-host-or-domain>
```

Inside the container:

```bash
hermes doctor
hermes setup
hermes
```

## Dashboard Tunnel

The dashboard is not exposed publicly. If you enable it later inside the container, access it with a tunnel:

```bash
ssh -p 2222 -L 9119:127.0.0.1:9119 hermes@<dokploy-host-or-domain>
```

Then open `http://127.0.0.1:9119` on your local machine.

## Security Checks

After deployment:

```bash
ssh -p 2222 hermes@<dokploy-host-or-domain> 'whoami && hermes doctor'
ssh -p 2222 root@<dokploy-host-or-domain>
```

The first command should log in as `hermes`. The second command should fail because root login is disabled.
```

- [ ] **Step 2: Commit docs**

Run:

```bash
git add docs/deploy/dokploy-ssh.md
git commit -m "docs: document dokploy ssh deployment"
```

Expected: commit succeeds.

---

### Task 6: Local Build and Runtime Verification

**Files:**
- Test only: `Dockerfile`
- Test only: `docker-compose.yml`
- Test only: `docker/sshd_config`
- Test only: `docker/cont-init.d/10-hermes-ssh-authorized-key`
- Test only: `docker/s6-rc.d/hermes-sshd/run`

**Interfaces:**
- Consumes: implementation from Tasks 2-4.
- Produces: verified local image and container behavior before Dokploy deployment.

- [ ] **Step 1: Generate a throwaway SSH key for verification**

Run:

```bash
rm -f /tmp/hermes_dokploy_test_key /tmp/hermes_dokploy_test_key.pub
ssh-keygen -t ed25519 -N "" -f /tmp/hermes_dokploy_test_key -C hermes-dokploy-test
```

Expected: `/tmp/hermes_dokploy_test_key.pub` exists.

- [ ] **Step 2: Validate `sshd_config` using a Debian container**

Run:

```bash
docker run --rm -v "$PWD/docker/sshd_config:/tmp/sshd_config:ro" debian:13.4 sh -lc 'apt-get update >/dev/null && apt-get install -y --no-install-recommends openssh-server >/dev/null && sshd -t -f /tmp/sshd_config'
```

Expected: command exits 0.

- [ ] **Step 3: Build the image**

Run:

```bash
HERMES_SSH_AUTHORIZED_KEY="$(cat /tmp/hermes_dokploy_test_key.pub)" docker compose build hermes
```

Expected: build exits 0.

- [ ] **Step 4: Start the container locally**

Run:

```bash
HERMES_SSH_HOST_PORT=2222 HERMES_SSH_AUTHORIZED_KEY="$(cat /tmp/hermes_dokploy_test_key.pub)" docker compose up -d hermes
```

Expected: `docker compose ps hermes` shows the service running.

- [ ] **Step 5: Confirm only the intended port is published**

Run:

```bash
docker compose port hermes 22
docker inspect "$(docker compose ps -q hermes)" --format '{{json .HostConfig.NetworkMode}} {{json .HostConfig.Binds}}'
```

Expected: first command prints a host binding for port `2222`; second command does not print `host` as the network mode and does not list host filesystem binds.

- [ ] **Step 6: Verify SSH login works**

Run:

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/tmp/hermes_known_hosts -i /tmp/hermes_dokploy_test_key -p 2222 hermes@127.0.0.1 'whoami && test "$HOME" = "/opt/data" && hermes doctor || true'
```

Expected: output starts with `hermes`. `hermes doctor` may report configuration warnings before setup, but the binary must execute.

- [ ] **Step 7: Verify root SSH login fails**

Run:

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/tmp/hermes_known_hosts -i /tmp/hermes_dokploy_test_key -p 2222 root@127.0.0.1 'true'
```

Expected: command fails.

- [ ] **Step 8: Verify password SSH login is unavailable**

Run:

```bash
ssh -o BatchMode=yes -o PubkeyAuthentication=no -o PreferredAuthentications=password -p 2222 hermes@127.0.0.1 'true'
```

Expected: command fails without prompting for a password.

- [ ] **Step 9: Verify state persists after restart**

Run:

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/tmp/hermes_known_hosts -i /tmp/hermes_dokploy_test_key -p 2222 hermes@127.0.0.1 'printf persistent > /opt/data/persist-check.txt'
docker compose restart hermes
sleep 10
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/tmp/hermes_known_hosts -i /tmp/hermes_dokploy_test_key -p 2222 hermes@127.0.0.1 'cat /opt/data/persist-check.txt'
```

Expected: final output is `persistent`.

- [ ] **Step 10: Stop local test container**

Run:

```bash
docker compose down
```

Expected: service stops. The named volume may remain for further testing.

- [ ] **Step 11: Commit verification fixes if any were required**

If any implementation fixes were needed during verification, run:

```bash
git add Dockerfile docker-compose.yml .env.dokploy.example docker/sshd_config docker/cont-init.d/10-hermes-ssh-authorized-key docker/s6-rc.d/hermes-sshd docker/s6-rc.d/user/contents.d/hermes-sshd docs/deploy/dokploy-ssh.md
git commit -m "fix: verify hermes ssh container runtime"
```

Expected: commit is created only if files changed.

---

### Task 7: Deploy to Dokploy

**Files:**
- Remote configuration: Dokploy project/application settings
- Remote runtime: Dokploy Docker Compose deployment

**Interfaces:**
- Consumes: verified repository from Task 6 and Dokploy credentials available through the existing local Dokploy helper or Dokploy UI.
- Produces: running Dokploy service accessible by direct SSH.

- [ ] **Step 1: Check whether host port 2222 is free on the Dokploy host**

Use the Dokploy host shell or Dokploy helper to run:

```bash
ss -tlnp | awk '$4 ~ /:2222$/ { print }'
```

Expected: no output. If there is output, choose the nearest clear high port, update `HERMES_SSH_HOST_PORT`, `docs/deploy/dokploy-ssh.md`, and the spec port note before deploying.

- [ ] **Step 2: Create or update the Dokploy project**

Create a Dokploy Docker Compose project named:

```text
hermes-agent-ssh
```

Expected: Dokploy project exists and points at this repository/branch or uploaded source.

- [ ] **Step 3: Set Dokploy environment variables**

Set:

```dotenv
HERMES_SSH_HOST_PORT=2222
HERMES_UID=10000
HERMES_GID=10000
HERMES_SSH_AUTHORIZED_KEY=<the user's public SSH key>
```

Expected: Dokploy stores the env vars. The private key is never uploaded.

- [ ] **Step 4: Deploy**

Trigger a Dokploy deployment for the Compose stack.

Expected: build succeeds and one `hermes` service is running.

- [ ] **Step 5: Inspect runtime network and mounts**

From the host, run the equivalent of:

```bash
docker ps --filter name=hermes --format 'table {{.Names}}\t{{.Ports}}'
docker inspect <hermes-container-id> --format '{{json .HostConfig.NetworkMode}} {{json .HostConfig.Binds}}'
```

Expected: SSH port is published; network mode is not `host`; no Docker socket or host path bind is present.

- [ ] **Step 6: Verify direct SSH**

From the local machine:

```bash
ssh -p 2222 hermes@<dokploy-host-or-domain> 'whoami && hermes doctor || true'
```

Expected: output starts with `hermes`; `hermes doctor` executes.

- [ ] **Step 7: Verify denied root SSH**

From the local machine:

```bash
ssh -o BatchMode=yes -p 2222 root@<dokploy-host-or-domain> 'true'
```

Expected: command fails.

- [ ] **Step 8: Run Hermes setup**

From the local machine:

```bash
ssh -p 2222 hermes@<dokploy-host-or-domain>
```

Inside the SSH session, run one of:

```bash
hermes setup
```

or:

```bash
hermes setup --portal
```

Expected: Hermes setup completes and writes state under `/opt/data`.

- [ ] **Step 9: Verify persistence on Dokploy**

Inside SSH:

```bash
printf dokploy-persistent > /opt/data/dokploy-persist-check.txt
```

Restart the service in Dokploy, then run:

```bash
ssh -p 2222 hermes@<dokploy-host-or-domain> 'cat /opt/data/dokploy-persist-check.txt'
```

Expected: output is `dokploy-persistent`.

- [ ] **Step 10: Record final access details**

Update `docs/deploy/dokploy-ssh.md` if the deployed SSH port is not `2222`.

Run:

```bash
git status --short
```

Expected: clean working tree, unless a port documentation change was needed. Commit any needed docs change with:

```bash
git add docs/deploy/dokploy-ssh.md docs/superpowers/specs/2026-08-23-hermes-dokploy-ssh-design.md
git commit -m "docs: record hermes dokploy ssh port"
```

---

## Self-Review Notes

- Spec coverage: isolated container, direct SSH, persistent `/opt/data`, key-only auth, no host network, no Docker socket, no public dashboard/API, and Dokploy Compose deployment are all mapped to tasks.
- Placeholder scan: no `TBD`, `TODO`, or vague implementation steps remain.
- Interface consistency: `HERMES_SSH_AUTHORIZED_KEY`, `HERMES_SSH_HOST_PORT`, `/opt/data`, and `hermes-sshd` names are used consistently across Dockerfile, Compose, docs, and verification.
