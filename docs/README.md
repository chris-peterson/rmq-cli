# <img src="favicon.svg" alt="rmq-cli" width="64" height="64" style="vertical-align: middle"> rmq-cli

A Docker image for administration of RabbitMQ using `rabbitmqadmin` and/or `rabbitmqctl`.

The container includes:

* [rabbitmqadmin](https://www.rabbitmq.com/management-cli.html)
* [rabbitmqctl](https://www.rabbitmq.com/rabbitmqctl.8.html)
* [jq](https://stedolan.github.io/jq/) — command line JSON processor
* [python](https://www.python.org/) — needed to support `rabbitmqadmin`

The default `CMD` is `rmq` which invokes `rabbitmqctl` honoring [environment variables](#environment-variables).

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
   ghcr.io/chris-peterson/rmq-cli:main rmqa export config.json
```

### Show Overview

```bash
docker run -e RABBIT_HOST=rabbitmqserver \
   -e RABBIT_USER=admin -e RABBIT_PASSWORD=p@ssw0rd \
   ghcr.io/chris-peterson/rmq-cli:main rmqa show overview --format=pretty_json
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
  script: rmqa export $RABBIT_HOST.config
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
  script: rmqa import $RABBIT_HOST.config
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
