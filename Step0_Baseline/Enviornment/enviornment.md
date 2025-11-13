# Environments — HSTrader

Short reference for preparing environment files and running services locally. This document purposely does not link or expose any committed .env files or secrets — it gives samples and points to compose/CI files to inspect.

## Purpose
- Standardize how services receive configuration (env vars).
- Describe common workflows: full-Docker and hybrid (docker infra + local services).
- Point to important compose and CI definitions to customize runtime behavior.

## Where to look (important repo files)
Inspect these files for runtime configuration and CI build/deploy logic:
- Docker compose / stacks:
  - vfxserver/stack.yaml
  - vfxserver/docker-compose.yaml
  - vfxcore/stack.yaml
  - vfxmarket/stack.yaml
- Service helper scripts and Makefiles:
  - vfxserver/Makefile
  - vfxcore/Makefile
  - vfxmarket/Makefile
  - vfxcore/export.sh
  - vfxmarket/export.sh
  - vfxlp/export.sh
- CI pipelines:
  - vfxmarket/.gitlab-ci.yml
  - vfxweb/.gitlab-ci.yml(artificat build)
  - vfxserver/.gitlab-ci.yml
  - vfxcore/.gitlab-ci.yml
  - vfxlp/.gitlab-ci.yml
  - vfxdata/ .gitlab-ci.yml

Do not commit secrets. CI pipelines should reference variables in the CI/CD secret store.

## Recommended workflows

1) Full Docker (all services in containers)
- Copy or create an env file locally (do not commit).
- From repo root (or service folder):
  - sudo docker compose -f vfxserver/docker-compose.yaml up -d
  - or adjust stack file and run: sudo docker compose -f vfxserver/stack.yaml up -d

2) Hybrid (recommended when developing Go services)
- Start infra only:
  - sudo docker compose -f vfxserver/stack.yaml up -d
- In each service folder:
  - source export.sh (if present) or export required env vars
  - make run  (or go run ./cmd)

## Minimal sample env (use only locally; replace placeholders)
- Do not paste secrets into repo. Save locally as `.env` or export per-shell.

```env
# Basic DB
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=vfxuser
MYSQL_PASSWORD=changeme
MYSQL_DB=vfxcore

# Cache / Broker
REDIS_URL=127.0.0.1:6379
NATS_HOST=nats://127.0.0.1:4222

# Influx
INFLUX_HOST=127.0.0.1
INFLUX_PORT=8086
INFLUX_Token=<token>

# Service ports
HTTP_PORT=8888
GRPC_PORT=3000

# Volumes (used by compose via variable substitution)
MYSQL_DATA=~/volumes/vfx/mysql
REDIS_DATA=~/volumes/vfx/redis
INFLUXDB_DATA=~/volumes/vfx/influx
LOGS_DATA=~/volumes/vfx/logs
```

## Helpful commands
- Start infra stack:
  - sudo docker compose -f vfxserver/stack.yaml up -d
- Run a single service (example):
  - cd vfxmarket
  - source export.sh   # or export vars manually
  - make run
- Build binary and image:
  - go build -o ./cmd/vfxserver cmd/*.go
  - docker build -t registry.gitlab.com/hscore/vfxserver:dev .
  - docker push registry.gitlab.com/hscore/vfxserver:dev

## CI / deployment notes
- CI pipelines bake runtime config using CI variables (do not store secrets in repo).
- Inspect service .gitlab-ci.yml files to see build, artifact, and registry steps.
- Compose files reference env vars for volume mounts and endpoints — set those locally before running compose.

## Troubleshooting pointers
- DB connection errors: verify MYSQL_* values and that the MySQL service is healthy in the compose stack.
- Port conflicts: check HTTP_PORT and GRPC_PORT values.
- Missing provider configs: check stack.yaml for mounts expected by LP/FIX services.

## Next steps
1. Choose workflow (Full Docker vs Hybrid).
2. Create a local `.env` from the sample above (or use export.sh per service).
3. Start infra via the stack YAML and run services as needed.
4. Inspect the listed compose and CI files to adapt to your