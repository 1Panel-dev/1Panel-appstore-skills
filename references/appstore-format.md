# 1Panel Appstore Format

Generated packages should follow the local app shape accepted by 1Panel.

```text
apps/<app-key>/
  logo.png
  README.md
  README_en.md
  data.yml
  <version>/
    data.yml
    docker-compose.yml
    <persistent-dir>/
      .gitkeep
    scripts/
      init.sh          # Optional; generated from init fields
      upgrade.sh       # Optional; added manually
      uninstall.sh     # Optional; added manually
      start.sh         # Only for a verified lifecycle-script caller
      stop.sh          # Only for a verified lifecycle-script caller
      restart.sh       # Only for a verified lifecycle-script caller
```

`README_en.md`, persisted directories, and `scripts/init.sh` are generated only when needed. Add other scripts manually when the target 1Panel workflow and official application requirements call for them. Omit `scripts/` when no scripts or helper files are needed.

## Root `data.yml`

Required fields:

- `name`
- `tags`
- `title`
- `description`
- `additionalProperties.key`
- `additionalProperties.name`
- `additionalProperties.tags`
- `additionalProperties.shortDescZh`
- `additionalProperties.shortDescEn`
- `additionalProperties.description`
- `additionalProperties.type`
- `additionalProperties.crossVersionUpdate`
- `additionalProperties.limit`
- `additionalProperties.architectures`

`additionalProperties.description` must include:

```text
en, es-es, ja, ms, pt-br, ru, ko, zh-Hant, zh, tr
```

## Version `data.yml`

Use `additionalProperties.formFields` for user-editable values.

- Every public port uses `rule: paramPort` and `type: number`.
- Secrets use `type: password`.
- Every form field `label` includes the full appstore language set.
- Every `${...}` variable used in Compose must be declared here, except known 1Panel-provided variables such as `${CONTAINER_NAME}`.

## `docker-compose.yml`

- Primary service uses `container_name: ${CONTAINER_NAME}`.
- Secondary services use `container_name: ${CONTAINER_NAME}-<service>`.
- Every service joins `1panel-network`.
- `1panel-network` is external.
- Every service includes `labels.createdBy: Apps`.
- Public ports use `PANEL_APP_PORT_*`.
- Persisted host paths are relative, such as `./data:/app/data`.

## README Files

Keep app README files concise:

- Chinese: `## 产品介绍`, optional `## 主要功能`.
- English: `## Introduction`, optional `## Features`.

Do not include source evidence, generated-package diagnostics, or local testing notes in app README files.

## Lifecycle Scripts

Hook availability depends on the target version and calling workflow. [1Panel v2.2.5](https://github.com/1Panel-dev/1Panel/blob/v2.2.5/agent/app/service/app_utils.go#L992-L1017) dispatches `init`, `upgrade`, and `uninstall`. The following table describes [dev-v2 at commit 5ad12c6](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/service/app_utils.go#L998-L1033); verify the target release before using the additional hooks.

| File under `<version>/scripts/` | Trigger and execution order | Script timeout |
| --- | --- | --- |
| `init.sh` | Installation initialization, before container startup. | 10 minutes |
| `upgrade.sh` | Upgrade, after stopping the old containers and installing the target scripts, before starting the new containers. | 10 minutes |
| `uninstall.sh` | Uninstall, after Compose teardown in the ordinary app workflow. In lifecycle-script mode, it also takes responsibility for teardown. | 10 minutes |
| `start.sh` | Start, including installation startup, only when the caller enables lifecycle-script mode. | 1 hour |
| `stop.sh` | Stop, only when the caller enables lifecycle-script mode. | 10 minutes |
| `restart.sh` | Restart or parameter update, only when the caller enables lifecycle-script mode. | 1 hour |

Execution rules:

- 1Panel runs `bash <installation-directory>/scripts/<hook>.sh` on the host, with the application installation directory as the working directory. Package-relative paths such as `data` and `.env` resolve there. Read `.env` explicitly when script logic needs its values.
- A missing hook is skipped. Before execution, 1Panel calls its recursive chmod helper on the selected script path. A nonzero script exit is returned to the calling task.
- Ordinary start, stop, and restart operations use Compose. The internal `UseLifecycleScripts` mode makes the corresponding script responsible for the operation; Compose is not also executed as a fallback. Merely adding these files does not enable this mode.
- Parameter updates do not rerun `init.sh`. Ordinary updates rebuild the containers; internal lifecycle-script mode calls `restart.sh` instead. `UseLifecycleScripts` is an internal field marked `json:"-"`, not a checkbox in the ordinary app parameter editor.

Call-site evidence: [installation](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/service/app.go#L560-L583), [upgrade](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/service/app_upgrade.go#L407-L432), [uninstall](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/service/app_utils.go#L348-L369), [start/stop/restart](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/service/app_install.go#L258-L277), [parameter update](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/service/app_install.go#L504-L513), and [internal field](https://github.com/1Panel-dev/1Panel/blob/5ad12c6fe4a63e5e08406283edd27f06c2e8e430/agent/app/dto/request/app.go#L54-L56).

### Generated `init.sh`

Generate `<version>/scripts/init.sh` only when official sources require host-side initialization.

Common case:

- A container runs as a non-root user.
- A host-mounted persistent directory must be writable by that user.
- Official Dockerfile, Compose, or image docs provide the numeric UID/GID.

If no init action is needed, do not generate `init.sh`. Other required scripts may still use `scripts/`.

### Manually Added Scripts

The spec and generator currently support only `init.sh`; add other hooks after generation. Use `upgrade.sh` for application-required migrations or configuration adjustments that fit the upgrade timing above. Use `uninstall.sh` only for cleanup of application-owned resources beyond normal 1Panel removal. Derive commands from official application sources and record the supporting URLs in script comments and the spec's `source_evidence`.

- Use the exact hook filename, a `#!/bin/bash` shebang, LF line endings, and executable permissions (for example, `chmod 755 scripts/upgrade.sh` from the version directory).
- Use paths relative to the installation directory. Keep helper files under `scripts/` as needed; helper filenames are not automatically dispatched as hooks.
- Make changes safe to retry, stop on failed prerequisites, and return nonzero on failure. A migration must account for supported source versions; cleanup must stay within the application's resources.
- Do not assume parameter updates rerun installation initialization, or that lifecycle-script mode is available to ordinary app packages.
- Run `bash -n` on each added script, then test the relevant lifecycle operation in the target 1Panel environment. The package validator does not check shell syntax or runtime behavior.

When regenerating with `--force`, the generator writes `init.sh` when init fields require it. When they no longer require it, it removes only a regular `init.sh` containing its `# Generated from official installation evidence:` marker line, and removes `scripts/` only if empty. This cleanup skips a symbolic link at `scripts/` or `init.sh`, preserving the link and its target. Other scripts and helper files are preserved. An unmarked manual `init.sh` is preserved when init fields are absent; keep edits to a generated `init.sh` in the spec because that file is managed by the generator.
