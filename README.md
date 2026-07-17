# Hermes Agent Railway Template

Deploy [Hermes Agent](https://github.com/NousResearch/hermes-agent) on Railway using the official Nous Research image, dashboard, and s6 process supervision. A minimal compatibility entrypoint maps variables from existing template deployments before handing control to the upstream `/init` process.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/hermes-agent)

## Railway configuration

The Dockerfile is pinned to this official image:

```text
nousresearch/hermes-agent:main@sha256:3fb7724c7ccf85d2bd64813fbc506868d5e190aa6cff170d3c8c7eb8e5d8a2cf
```

This immutable `main` digest includes upstream fix [#61178](https://github.com/NousResearch/hermes-agent/pull/61178), which keeps password-only dashboard authentication on the login form. The latest tagged release, `v2026.7.7.2`, predates that fix and crashes by routing `BasicAuthProvider` through the OAuth handler. Return to a tagged image once Nous Research publishes a release containing the fix.

Use these service settings:

| Setting | Value |
|---|---|
| Start command | `gateway run` |
| Volume mount | `/data` |
| Public port | `8080` |
| Health-check path | `/api/status` |
| Replicas | `1` |

Railway continues building this GitHub repository so existing template consumers receive update notifications. The compatibility entrypoint only translates legacy environment variables, then executes the official image's `/init`; s6-overlay still supervises the Hermes gateway and dashboard.

### Variables

```dotenv
PORT=8080
ADMIN_USERNAME=admin
ADMIN_PASSWORD=<generated-password>
```

The compatibility entrypoint maps the existing `ADMIN_USERNAME` and sealed `ADMIN_PASSWORD` values to the official dashboard variables before starting `/init`. Existing deployments therefore keep the same login without adding, copying, revealing, or re-entering credentials. It does not modify `config.yaml`. If `ADMIN_PASSWORD` is absent, it generates and logs a password as the previous template did.

For zero-interaction migration, a stable 32-byte dashboard session-signing secret is derived from `ADMIN_PASSWORD`. Derivation supports legacy passwords shorter than Hermes's 16-byte minimum signing-key length while preserving the existing login. After migration, operators may add `HERMES_DASHBOARD_BASIC_AUTH_SECRET` as an independent sealed value containing at least 32 random bytes. Rotating only this secret logs dashboard users out; it does not change the dashboard password or Hermes data. Neither the password nor the derived secret is written to `config.yaml`.

The Dockerfile supplies the remaining official variables:

```dotenv
HERMES_HOME=/data/.hermes
HERMES_WRITE_SAFE_ROOT=/data/.hermes
HERMES_LAZY_INSTALL_TARGET=/data/.hermes/lazy-packages
HERMES_DASHBOARD=1
HERMES_DASHBOARD_HOST=0.0.0.0
HERMES_GATEWAY_BOOTSTRAP_STATE=running
```

The entrypoint maps `PORT` to `HERMES_DASHBOARD_PORT` at runtime.

Provider credentials, messaging channels, models, skills, profiles, and gateway state are managed through the official dashboard and persisted under `/data/.hermes`.

## Migrating an existing deployment

The previous template already stores Hermes data under `/data/.hermes`. Keep the existing Railway service, domain, volume, and `/data` mount unchanged.

Before changing the service:

1. Stop the gateway and create a manual Railway volume backup.
2. Record the current source commit and service variables for rollback.
3. Apply the template update. Railway rebuilds this repository against the pinned official image.
4. Verify that the start command is `gateway run` and the health check is `/api/status`; both are declared in `railway.toml`.
5. Deploy and verify the dashboard, gateway, configured channels, sessions, memories, and skills. Existing `ADMIN_*` credentials and provider or messaging configuration in `/data/.hermes/.env` remain in place.

Do not change the volume mount to `/opt/data` and do not move the existing data. To roll back, redeploy the previous repository commit; the old runtime will read the same untouched `/data/.hermes` directory.

## Local verification

The Compose configuration builds the same compatibility image Railway uses and preserves the `/data/.hermes` layout. Existing `ADMIN_USERNAME` and `ADMIN_PASSWORD` values are forwarded to the official dashboard variables. Without an `.env` file, the local credentials default to `admin` / `change-me`.

To override the local credentials, create an environment file:

```bash
cp .env.example .env
```

Change the dashboard password and signing secret in `.env`, then run:

```bash
docker compose up -d
```

Open <http://localhost:8080>. Check container health and logs with:

```bash
docker compose ps
docker compose logs -f hermes
```

Stop the service without deleting its persistent data:

```bash
docker compose down
```

Hermes state is stored in the ignored local `./data/.hermes` directory and survives `docker compose down`. Back up or remove `./data` manually when you want to preserve or reset the local test state.

### Migrating the previous local named volume

Earlier versions of this repository stored local state in the Docker volume `hermes-agent-railway-template_hermes-data`. Switching to the `./data` bind mount does not copy that volume automatically. To preserve the old state, stop Hermes, retain the newly created directory as a backup, and copy the old volume into a fresh `./data` directory:

```bash
docker compose down
mv data "data.fresh-backup-$(date +%Y%m%d%H%M%S)"
mkdir data
docker run --rm \
  --entrypoint /bin/sh \
  -v hermes-agent-railway-template_hermes-data:/source:ro \
  -v "$PWD/data:/destination" \
  hermes-agent-railway-template:local \
  -c 'cp -a /source/. /destination/'
docker compose up --build
```

The source volume is mounted read-only and is not deleted. Verify the migrated sessions, configuration, provider keys, and messaging channels before removing either the old Docker volume or the timestamped backup directory.

## Upgrading Hermes

Update the pinned release and digest in `Dockerfile` deliberately after reviewing the upstream [release notes](https://github.com/NousResearch/hermes-agent/releases) and validating the new image. Do not use `latest`. Because the template remains GitHub-backed, merging an upgrade to the default branch notifies existing template consumers.

See the official [Hermes Docker documentation](https://hermes-agent.nousresearch.com/docs/user-guide/docker) for image behavior and configuration details.
