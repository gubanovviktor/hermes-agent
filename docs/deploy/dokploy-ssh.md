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

## Dashboard Tunnel

The dashboard is not exposed publicly. If you enable it later inside the container, access it with a tunnel:

```bash
ssh -p 2223 -L 9119:127.0.0.1:9119 hermes@<dokploy-host-or-domain>
```

Then open `http://127.0.0.1:9119` on your local machine.

## Security Checks

After deployment:

```bash
ssh -p 2223 hermes@<dokploy-host-or-domain> 'whoami && hermes doctor'
ssh -p 2223 root@<dokploy-host-or-domain>
```

The first command should log in as `hermes`. The second command should fail because root login is disabled.
