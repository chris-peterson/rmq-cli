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
   ghcr.io/chris-peterson/rmq-cli:main rmqa list queues
```

### Export Configuration

```bash
docker run -e RABBIT_HOST=rabbitmqserver \
   -e RABBIT_USER=admin -e RABBIT_PASSWORD=p@ssw0rd \
   ghcr.io/chris-peterson/rmq-cli:main rmqa definitions export --file config.json
```

### Show Overview

```bash
docker run -e RABBIT_HOST=rabbitmqserver \
   -e RABBIT_USER=admin -e RABBIT_PASSWORD=p@ssw0rd \
   ghcr.io/chris-peterson/rmq-cli:main rmqa show overview
```

### GitLab CI

```yaml
stages:
- snapshot
- deploy
- rollback

image: ghcr.io/chris-peterson/rmq-cli:main

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

To read the SBOM instead of verifying the signature:

```bash
docker buildx imagetools inspect ghcr.io/chris-peterson/rmq-cli:4.3.5 \
   --format '{{ json .SBOM }}'
```
