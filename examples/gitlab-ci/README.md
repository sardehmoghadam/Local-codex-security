# GitLab CI

Four copyable pipelines that run Codex Security in GitLab CI and POST the sealed
findings JSON to a remote RIS-P endpoint. Copy one file to the target repository
as `.gitlab-ci.yml`, or `include:` it from an existing pipeline, then set the
variables below.

| File                                 | Scanner                                       | Scope                 |
| ------------------------------------ | --------------------------------------------- | --------------------- |
| [`source-full.yml`](source-full.yml) | builds the CLI from a pinned ref of this repo | full repository       |
| [`source-diff.yml`](source-diff.yml) | same                                          | merge request changes |
| [`image-full.yml`](image-full.yml)   | prebuilt scanner image                        | full repository       |
| [`image-diff.yml`](image-diff.yml)   | same                                          | merge request changes |

The `source-*` files clone this repository next to the checkout and build the CLI
there, so they need no registry and no DinD service. Build output never lands in
the scan target, which also keeps it out of the inventory. The `image-*` files
only need an image reference and start faster, but require a runner that can pull
it. The `*-diff` files scan `--diff $CI_MERGE_REQUEST_DIFF_BASE_SHA` on merge
request pipelines and fall back to the full repository on branch, scheduled, and
web pipelines.

## Variables

Set these in **Settings > CI/CD > Variables**. Mark the credentials **Masked**;
mark the scan credentials **Protected** as well if only protected refs may run
scans.

| Variable               | Required   | Notes                                                                                                                    |
| ---------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------ |
| `LOCAL_API_BASE_URL`   | yes        | OpenAI-compatible endpoint, e.g. `http://host.docker.internal:1234/v1`                                                   |
| `LOCAL_API_KEY`        | no         | Set it when the gateway requires a key. Without it, `--auth auto` resolves to anonymous access for the `local` provider. |
| `RISQ_URL`             | yes        | Endpoint root without a trailing slash                                                                                   |
| `RIS_P_ENDPOINT_TOKEN` | yes        | Masked. It travels in the request URL, so it also appears in that URL                                                    |
| `RIS_P_SECRET`         | yes        | Masked                                                                                                                   |
| `SCAN_MODEL`           | no         | Default `qwen3.5-35b-a3b`                                                                                                |
| `SCAN_EFFORT`          | no         | Default `high`, passed through to Codex unchanged                                                                        |
| `SCAN_MODE`            | no         | `standard` or `deep`                                                                                                     |
| `FAIL_ON_SEVERITY`     | no         | `none`, `critical`, `high`, `medium`, or `low`                                                                           |
| `SCAN_TARGET`          | no         | Default `$CI_PROJECT_DIR`                                                                                                |
| `SCANNER_REPO_URL`     | `source-*` | This repository's Git URL                                                                                                |
| `SCANNER_REF`          | `source-*` | Branch, tag, or commit to build                                                                                          |
| `SCANNER_IMAGE`        | `image-*`  | Scanner image reference                                                                                                  |

Do not export `OPENAI_API_KEY` or `CODEX_API_KEY` to these jobs. A `local` scan
uses only the local provider's credential.

## Runner requirements

Codex confines source inspection with Bubblewrap, so the scan needs a runner that
supports unprivileged user namespaces. Register the runner with the repository's
seccomp profile:

```toml
[[runners]]
  [runners.docker]
    security_opt = ["seccomp=/path/to/codex-security/docker/codex-security-seccomp.json"]
```

On hosts that restrict unprivileged namespaces, also load
`docker/codex-security.apparmor` and add
`security_opt = ["apparmor=codex-security-container"]`. Do not disable sandboxing
to work around a blocked scan.

When the inference gateway runs on the runner host, reach it from the job
container:

```toml
[[runners]]
  [runners.docker]
    extra_hosts = ["host.docker.internal:host-gateway"]
```

and set `LOCAL_API_BASE_URL=http://host.docker.internal:1234/v1`.

## Build the scanner image

The `image-*` files expect an image published once from this repository:

```bash
docker build --target scanner -t registry.example.com/security/codex-security:0.8.2 .
docker push registry.example.com/security/codex-security:0.8.2
```

`scanner` runs as uid 10001, ships a CLI entrypoint, and installs `python3`, `git`,
and `ca-certificates` but not `curl`. Publish an overlay for GitLab instead of
tagging that stage directly:

```dockerfile
FROM registry.example.com/security/codex-security:0.8.2
USER root
RUN apt-get update \
    && apt-get install --no-install-recommends --yes curl \
    && rm -rf /var/lib/apt/lists/*
```

A job that keeps `USER 10001` must point `SCAN_TARGET`, `REPORT_DIR`, and
`CODEX_SECURITY_STATE_DIR` at writable paths.

## Results and failures

Each job writes to `security-results/` and uploads it as an artifact:

- `result.json` - the scan's `--format json` stdout
- `findings.json`, `report.md`, `scan-manifest.json` when the scan produced them
- `ris-p-findings.json` - `export --export-format json`, byte-identical to the
  sealed `findings.json` and the body of the upload
- `results.sarif` - `export --export-format sarif` with source-line fingerprints

Add `security-results/` to the target repository's `.gitignore`.

| Scan exit         | Job behavior                                                                                              |
| ----------------- | --------------------------------------------------------------------------------------------------------- |
| `0`               | Export reports, upload the findings JSON, job passes.                                                     |
| `1`               | Severity policy failed. Reports are still exported and uploaded; `allow_failure` marks the job a warning. |
| `2`, `130`, `143` | Execution failed or was interrupted. No export or upload; the job fails.                                  |

The upload runs `curl --fail-with-body`, so a 4xx or 5xx response fails the job and
prints the server's response body. `--retry` replays the POST on connection errors
and 5xx responses, and the endpoint's `pipeline_id` and `job` parameters make those
replays idempotent. An upload failure fails the job only when the scan itself
succeeded.

`--diff` requires a clean checkout without untracked files, which is why the
`*-diff` files write nothing into `$CI_PROJECT_DIR` before the scan and set
`GIT_DEPTH: "0"`. Publishing the SARIF file to the GitLab code scanning view is
Ultimate-only and only ingests jobs that succeed, so it stays commented out.

## Trust boundaries

- Restrict who can edit these files and use the variables. Treat scanned code and
  model output as untrusted input.
- Never enable `set -x` in the job script: `RIS_P_ENDPOINT_TOKEN` is part of the
  request URL and `RIS_P_SECRET` is part of a header.
- Scan reports can contain source excerpts and vulnerability details. Restrict
  artifact access and set a retention policy that matches your data handling.
- The `source-*` files install dependencies with `--frozen-lockfile` from the
  pinned ref; pin `SCANNER_REF` to a reviewed commit or tag rather than a moving
  branch.
