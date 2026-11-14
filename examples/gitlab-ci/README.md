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
| `SCANNER_REPO_URL`     | `source-*` | This repository's Git URL, without embedded credentials                                                                  |
| `SCANNER_REF`          | `source-*` | Branch or tag to build; `git clone --branch` does not accept a commit SHA                                                |
| `SCANNER_GIT_TOKEN`    | no         | Masked. Read-only access or deploy token for a private scanner project; omit to clone with `CI_JOB_TOKEN`                |
| `SCANNER_GIT_USER`     | no         | User name for `SCANNER_GIT_TOKEN`, e.g. `gitlab+deploy-token-42`. Default `oauth2`                                       |
| `SCANNER_IMAGE`        | `image-*`  | Scanner image reference                                                                                                  |

Do not export `OPENAI_API_KEY` or `CODEX_API_KEY` to these jobs. A `local` scan
uses only the local provider's credential.

## Private scanner repository

The runner injects credentials only for the project the pipeline runs in, so a
`source-*` job that clones a second, private project fails with
`fatal: could not read Username for '...': No such device or address`. Those files
authenticate the clone with `CI_JOB_TOKEN`. On the scanner project, open
**Settings > CI/CD > Job token permissions**, add the project that runs the
pipeline (or its group), and save. GitLab denies job-token access to other
projects until that entry exists.

If the instance restricts job-token permissions, create a project or group
**access token** (or a **deploy token**) with the `read_repository` scope only,
store it as the masked variable `SCANNER_GIT_TOKEN`, and set `SCANNER_GIT_USER`
when the token has its own user name (`gitlab+deploy-token-42` for a deploy
token; `oauth2` is the default).

Credentials are added only to `http://` and `https://` URLs; `git@host:...` and
`file://` URLs reach `git clone` unchanged. Prefer `https://`: a token in an
`http://` URL crosses the network in clear text, and a redirect from `http` to
`https` can drop it. Set `SCANNER_REPO_URL` to the URL the server finally serves.

## Runner requirements

Codex confines source inspection with Bubblewrap, so the scan needs a runner that
supports unprivileged user namespaces. Docker's built-in seccomp profile blocks the
namespace syscalls Bubblewrap needs, so register the runner with this repository's
profile:

```bash
# on the host that runs the Docker daemon
sudo install -D -m 0644 docker/codex-security-seccomp.json \
  /etc/codex-security/codex-security-seccomp.json
```

```toml
[[runners]]
  [runners.docker]
    security_opt = [
      "seccomp=/etc/codex-security/codex-security-seccomp.json",
      "no-new-privileges:true",
    ]
    cap_drop = ["ALL"]
```

These settings apply to every job the runner executes, so register the runner for
these jobs alone and select it with `tags: [codex-security]`. The Docker daemon
resolves that path, not the job container: install the profile on the host that runs
the daemon, and bind-mount it at the same path into the runner container when the
runner itself runs in one. When the daemon cannot read that host path, as with
Docker Desktop's Linux VM, no `security_opt` path works; run the scan through the
CLI's own `docker run --security-opt seccomp=<path>`, which reads the file locally,
or run these jobs on a Linux runner.

Check a runner before spending a scan on it:

```yaml
runner-preflight:
  stage: security
  image:
    name: "$SCANNER_IMAGE" # node:24-bookworm-slim for the source-* files
    entrypoint: [""]
  script:
    - unshare -Ur true && echo "userns available"
```

Without the profile the check and every scan fail with
`bwrap: No permissions to create a new namespace`, and the scan exits 2.

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

A job that keeps `USER 10001` must point `SCAN_TARGET` and `REPORT_DIR` at writable
paths. The state directory stays outside the checkout by default
(`/output/.codex-security-state` in the scan images,
`$HOME/.codex/state/plugins/codex-security` under `node:24-bookworm-slim`), and the
CLI rejects one that resolves inside the scanned repository.

A GitLab-hosted copy of this repository builds the same image from its own
`.gitlab-ci.yml`. The `publish-image` job runs the `scanner` target, adds `curl`
in a derived image, and pushes `<registry>/risp:<commit sha>` and `:latest`
without credentials. It runs `docker` against the socket the runner binds in from
the host, so it needs neither a `docker:dind` service nor a privileged runner,
and a registry that serves plain HTTP must be trusted by that daemon. It runs on
the default branch, on tags, and on manual runs.

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

An exit 2 with `bwrap: No permissions to create a new namespace` means the runner
lacks the seccomp profile; see [Runner requirements](#runner-requirements).

`--diff` requires a clean checkout without untracked files, which is why the
`*-diff` files write nothing into `$CI_PROJECT_DIR` before the scan and set
`GIT_DEPTH: "0"`. Publishing the SARIF file to the GitLab code scanning view is
Ultimate-only and only ingests jobs that succeed, so it stays commented out.

## Trust boundaries

- Restrict who can edit these files and use the variables. Treat scanned code and
  model output as untrusted input.
- Never enable `set -x` in the job script: `RIS_P_ENDPOINT_TOKEN` is part of the
  request URL and `RIS_P_SECRET` is part of a header.
- The `source-*` clone URL carries `CI_JOB_TOKEN` or `SCANNER_GIT_TOKEN` for the
  duration of the job. Keep that token read-only and scoped to `read_repository`.
- Scan reports can contain source excerpts and vulnerability details. Restrict
  artifact access and set a retention policy that matches your data handling.
- The `source-*` files install dependencies with `--frozen-lockfile` from the
  pinned ref; pin `SCANNER_REF` to a reviewed commit or tag rather than a moving
  branch.
