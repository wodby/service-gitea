# Gitea on Wodby

What Wodby sets up for Gitea on this service. Gitea runs from the official rootless image `gitea/gitea:<version>-rootless` as user and group 1000. Configuration is passed as `GITEA__<section>__<KEY>` environment variables, which the image writes to `app.ini` on every start; a variable with an empty section name (`GITEA____APP_NAME`) sets a key at the top of the file.

## Linked services

| Link | Variables |
| --- | --- |
| PostgreSQL (required) | `GITEA__database__DB_TYPE`, `GITEA__database__HOST`, `GITEA__database__NAME`, `GITEA__database__USER`, `GITEA__database__PASSWD`, `GITEA__database__SSL_MODE` |
| SMTP server (optional) | `GITEA__mailer__ENABLED`, `GITEA__mailer__PROTOCOL`, `GITEA__mailer__SMTP_ADDR`, `GITEA__mailer__SMTP_PORT` |

`GITEA__database__SSL_MODE` is `prefer`: the connection uses TLS when the server offers it, as managed databases do, and plain TCP to the bundled PostgreSQL service. The certificate is not verified.

Mail goes to the linked mail service without authentication or TLS. The mail variables are present only while the link exists. The sender is `noreply@<site domain>` unless `GITEA__mailer__FROM` is set on the service.

## What the manifest sets

- `GITEA__server__DOMAIN`, `GITEA__server__SSH_DOMAIN` and `GITEA__server__ROOT_URL` follow the service's primary route. A route change marks the service for redeploy.
- `GITEA____APP_NAME` is the app's title.
- `GITEA__security__INSTALL_LOCK` is `true`, so the web installer never runs.
- Generated tokens provide `GITEA__security__SECRET_KEY`, `GITEA__security__INTERNAL_TOKEN`, `GITEA__oauth2__JWT_SECRET` and `GITEA__server__LFS_JWT_SECRET`. They are stable across deployments. Changing `SECRET_KEY` makes stored two-factor secrets and other encrypted values unreadable.
- `GITEA__service__DISABLE_REGISTRATION` is `true`: accounts are created by an administrator. Set it to `false` to open sign-up.
- `GITEA__security__REVERSE_PROXY_TRUSTED_PROXIES` lists private networks, so Gitea takes client addresses from the cluster gateway's `X-Forwarded-For` header.
- Git LFS is on (`GITEA__server__LFS_START_SERVER`).
- `GITEA_APP_INI` is `/var/lib/gitea/custom/conf/app.ini`, on the data volume.

## Administrator

The first start creates an administrator from the service settings Admin username (default `gitea_admin`), Admin password (required, at least 8 characters) and Admin email (default `<username>@<site domain>`), only when Gitea has no administrator. The container stops with an error when the password is missing or too short. Later changes to these settings are not applied; change the password in Gitea.

## Git over SSH

The `gitea` endpoint has an `http` port (3000) and an `ssh` port (2222, Gitea's built-in SSH server). Git over SSH needs the `ssh` port published on the app's Ports page. Publishing redeploys the service: `GITEA_SSH_PUBLIC_PORT` gets the public port, the startup script sets `GITEA__server__SSH_PORT` to it and turns SSH on. Until then `GITEA__server__DISABLE_SSH` is `true` and Gitea shows only HTTPS clone addresses. The SSH user is `git`.

## Startup

The container runs a script after the image's entrypoint has written `app.ini`: it applies the SSH and mail sender values, runs `gitea migrate`, creates the administrator when needed, then starts `gitea web`. The health endpoint is `/api/healthz`; the startup probe allows 15 minutes for database migrations after an upgrade.

## Changing configuration

Change Gitea settings by adding or editing `GITEA__<section>__<KEY>` environment variables on the service, then deploy. Removing a variable doesn't remove its key from `app.ini`; set the value you want instead. Settings in Gitea's admin panel that are stored in the database, such as users and organizations, are not affected.

## Data

One `data` volume is mounted at `/var/lib/gitea`. It holds repositories (`git/repositories`), LFS objects, packages, attachments, avatars, Actions logs and artifacts, the SSH host keys and `app.ini`. Everything else is in the linked PostgreSQL database. Back up the volume and the database together. The manifest declares no backup, import or action for this service.

Gitea Actions is available, but no runner is included: register an external runner to run workflows. The service runs one replica and can't be scaled.
