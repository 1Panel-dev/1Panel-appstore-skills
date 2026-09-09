# Review Checklist

Use this before finalizing a generated app package.

## Required Files

- Root `data.yml`
- Root `README.md`
- Root `README_en.md` when the upstream app or docs support English
- Root `logo.png`
- Version `data.yml`
- Version `docker-compose.yml`

## README

- `README.md` follows the existing appstore style: `产品介绍` and optional `主要功能`.
- `README_en.md` follows the existing appstore style: `Introduction` and optional `Features`.
- README content is based on official repository README, official website, or official docs.
- README files do not include generation evidence, local test paths, Docker internals, or long installation notes.

## Metadata

- App key matches folder name.
- Version folder does not start with `v`.
- Root `additionalProperties.key`, `name`, `tags`, `type`, `description`, and `limit` exist.
- Official website, GitHub, and docs links are included when known.
- Architecture list matches the image manifest when known.
- Root `additionalProperties.description` includes `en`, `es-es`, `ja`, `ms`, `pt-br`, `ru`, `ko`, `zh-Hant`, `zh`, and `tr`.

## Compose

- Main service uses `${CONTAINER_NAME}`.
- Services use `restart: always`.
- Services join `1panel-network`.
- `networks.1panel-network.external` is `true`.
- Services have `labels.createdBy: "Apps"`.
- Every public port uses a `PANEL_APP_PORT_*` variable.
- Every compose variable is declared in version `data.yml`, except 1Panel-provided variables.
- Persistent paths use relative mounts such as `./data`.
- Persistent relative mounts were checked against the official container runtime user.
- Third-party images are used only when official sources confirm no usable public image or source-build-only installation, the user accepted the image, and the package records the third-party source.
- Every version form field includes a full multilingual `label` map.

## Lifecycle Scripts

- Required hooks match the target 1Panel version and calling workflow described in `appstore-format.md`.
- `<version>/scripts/` is omitted when no scripts or helper files are needed; an app may need upgrade or uninstall scripts without an init script.
- Manually added hooks use the documented filenames, Bash/LF format, executable permissions, and installation-relative paths; `bash -n` passes for each script.
- Every lifecycle action has official application evidence in the spec and script comments. Migrations account for supported source versions; cleanup is limited to application-owned resources.
- Start/stop/restart hooks have a verified caller that enables lifecycle-script mode. Ordinary parameter updates are not assumed to run `init.sh` or `restart.sh`.
- Regeneration preserves manual scripts and helpers; changes to generated `init.sh` are recorded in the spec.
- If a non-root container user needs write access to a persisted host directory, `<version>/scripts/init.sh` exists.
- `init.sh` uses UID/GID values confirmed from official Dockerfile, Compose, or image docs.
- `init.sh` fixes only package-relative persisted paths, for example `chown -R 1000:1000 data`.
- Other `init.sh` commands are limited to official preflight requirements.
- Expected persisted directories exist in the package tree with `.gitkeep`; `init.sh` is not the only source of those directories.

## Security

- No unexpected `privileged: true`.
- No host network unless justified.
- No dangerous host mounts such as `/`, `/etc`, or `/var/run/docker.sock` unless explicitly required.
- Avoid unpinned `:latest` images unless the upstream application only publishes `latest`.
- Secrets are form fields with `type: password`.

## Local Test

- Run `python3 scripts/validate_app_package.py apps/<app-key>` from the skill directory for basic package checks; this does not validate shell syntax or lifecycle behavior.
- Copy the generated app folder to `/opt/1panel/resource/apps/local`.
- Refresh local apps in 1Panel.
- Install, start, stop, restart, uninstall, and reinstall.
- When applicable, test upgrade from supported source versions and parameter changes, including the resulting files and container configuration.
- Confirm exposed port, data persistence, logs, and health checks.
