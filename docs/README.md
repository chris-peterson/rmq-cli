# <img src="favicon.svg" alt="rmq-cli" width="64" height="64" style="vertical-align: middle"> rmq-cli

A Docker image for administration of RabbitMQ using `rabbitmqadmin` and/or `rabbitmqctl`.

The container includes:

* [rabbitmqadmin](https://www.rabbitmq.com/docs/management-cli)
* [rabbitmqctl](https://www.rabbitmq.com/rabbitmqctl.8.html)
* [jq](https://stedolan.github.io/jq/) — command line JSON processor

The default `CMD` is `rmq` which invokes `rabbitmqctl` honoring [environment variables](#environment-variables).

`rabbitmqadmin` is v2, whose command syntax differs from v1: named arguments are
`--snake-case` flags rather than `key=value`, and result sorting, column selection,
and JSON/CSV output for most commands are gone. See the
[breaking changes](https://github.com/rabbitmq/rabbitmqadmin-ng#breaking-or-potentially-breaking-changes).

## Environment Variables

| name | default | description |
| --- | --- | --- |
| `RABBIT_HOST` | `127.0.0.1` | IP or FQDN of the RabbitMQ server |
| `RABBIT_PORT` | `15672` | Management port of the RabbitMQ server |
| `RABBIT_USER`/`RABBIT_PASSWORD` | `guest` | Administrator username/password (used by rmqa) |
| `RABBIT_ERLANG_COOKIE` | `<unset>` | Erlang cookie (used by rmq) |

## Examples

### List Queues

```bash
docker run -e RABBIT_HOST=rabbitmqserver \
   -e RABBIT_USER=admin -e RABBIT_PASSWORD=p@ssw0rd \
   ghcr.io/chris-peterson/rmq-cli:4.3 rmqa list queues
```

### Export Configuration

```bash
docker run -e RABBIT_HOST=rabbitmqserver \
   -e RABBIT_USER=admin -e RABBIT_PASSWORD=p@ssw0rd \
   ghcr.io/chris-peterson/rmq-cli:4.3 rmqa definitions export --file config.json
```

### Show Overview

```bash
docker run -e RABBIT_HOST=rabbitmqserver \
   -e RABBIT_USER=admin -e RABBIT_PASSWORD=p@ssw0rd \
   ghcr.io/chris-peterson/rmq-cli:4.3 rmqa show overview
```

### GitLab CI

```yaml
stages:
- snapshot
- deploy
- rollback

image: ghcr.io/chris-peterson/rmq-cli:4.3

variables:
  RABBIT_USER: admin
  RABBIT_PASSWORD: p@ssw0rd

.snapshot:
  stage: snapshot
  script: rmqa definitions export --file $RABBIT_HOST.config
  artifacts:
    paths:
    - $RABBIT_HOST.config
    expire_in: 1 week

.deploy:
  stage: deploy
  script: scripts/deploy.sh
  when: manual

.rollback:
  stage: rollback
  script: rmqa definitions import --file $RABBIT_HOST.config
  when: manual

snapshot:test:
  extends: .snapshot
  environment: test
  variables:
    RABBIT_HOST: test-rabbitmqserver

deploy:test:
  extends: .deploy
  environment: test
  variables:
    RABBIT_HOST: test-rabbitmqserver

rollback:test:
  extends: .rollback
  environment: test
  variables:
    RABBIT_HOST: test-rabbitmqserver
```

## Image tags

Tags follow the RabbitMQ base image version, in two forms:

| Tag | Moves? | Use it when |
| --- | --- | --- |
| `4.3` | to the newest `4.3.x` | **recommended** — patch updates within a minor line, so the `rabbitmqadmin` CLI grammar stays put |
| `4.3.5` | never | you need a byte-identical image on every pull |

### Older versions

Still resolvable, no longer built:

| Tag | RabbitMQ | Notes |
| --- | --- | --- |
| `4.1.1` | 4.1.1 | last release before the base image switched to `rabbitmqadmin` v2 |
| `4.x` | 4.1.1 | no longer produced, in favor of the `X.Y` and `X.Y.Z` forms [above](#image-tags) |
| `3.13` | 3.13.3 | RabbitMQ 3.x, `rabbitmqadmin` v1 grammar |

`4.1.1` and earlier ship the Python `rabbitmqadmin` v1, where definitions export is
`export <file>` rather than `definitions export --file <file>`.

## Verifying the image

Published images carry a [SLSA build provenance](https://slsa.dev/spec/v1.0/provenance)
attestation and per-platform SBOMs, signed by Sigstore during the build. The
provenance binds the image digest to the workflow run that produced it, so you can
confirm an image came from this repository rather than from someone who pushed a
similarly-named tag.

```bash
gh attestation verify oci://ghcr.io/chris-peterson/rmq-cli:4.3.5 \
   --repo chris-peterson/rmq-cli
```

```text
Loaded digest sha256:a0fb14f79d4bfc9b78c36c8eb0f2a5d81c37317e128be8768bf7ccaa696beb93 for oci://ghcr.io/chris-peterson/rmq-cli:4.3.5
Loaded 1 attestation from GitHub API

The following policy criteria will be enforced:
- Predicate type must match:................ https://slsa.dev/provenance/v1
- Source Repository Owner URI must match:... https://github.com/chris-peterson
- Source Repository URI must match:......... https://github.com/chris-peterson/rmq-cli
- Subject Alternative Name must match regex: (?i)^https://github\.com/chris-peterson/rmq-cli/
- OIDC Issuer must match:................... https://token.actions.githubusercontent.com

✓ Verification succeeded!

The following 1 attestation matched the policy criteria

- Attestation #1
  - Build repo:..... chris-peterson/rmq-cli
  - Build workflow:. .github/workflows/docker-publish.yml@refs/tags/4.3.5
  - Signer repo:.... chris-peterson/rmq-cli
  - Signer workflow: .github/workflows/docker-publish.yml@refs/tags/4.3.5
```

**In a pipeline, gate on the exit code, not the output.** That report is only
printed to a terminal; with stdout redirected or piped, `gh attestation verify`
prints nothing at all and signals solely through its exit status. A silent success
is easily misread as a failed command:

```bash
if gh attestation verify oci://ghcr.io/chris-peterson/rmq-cli:4.3 \
     --repo chris-peterson/rmq-cli >/dev/null 2>&1; then
   echo "provenance ok"
fi
```

## Reading the SBOM

Each image carries an SPDX SBOM per platform, so you can answer "does this
CVE affect me" without pulling and scanning the image yourself:

```bash
docker buildx imagetools inspect ghcr.io/chris-peterson/rmq-cli:4.3.5 \
   --format '{{ json .SBOM }}' \
   | jq -r '."linux/amd64".SPDX.packages[] | "\(.name) \(.versionInfo)"' | sort
```

```text
adduser 3.137ubuntu1
apt 2.8.3
base-files 13ubuntu10.4
base-passwd 3.6.3build1
bash 5.2.21-2ubuntu4
bsdutils 1:2.39.3-9ubuntu6.5
ca-certificates 20260601~24.04.1
coreutils 9.4-3ubuntu6.2
...
```

Swap `linux/amd64` for `linux/arm64` to read the other platform; both carry 124
packages for 4.3.5. The signed SBOM attestations are separate from these, one per
platform, and are what `gh attestation verify` checks.
