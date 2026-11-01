# my-floci

A minimal Docker Compose setup for running [floci](https://github.com/floci-io/floci), a free, MIT-licensed AWS emulator that runs on `localhost:4566`.

## What is floci?

floci is a local, drop-in replacement for AWS (and for LocalStack). Point your AWS SDK, CLI, Terraform, or test suite at `http://localhost:4566` and it responds like real AWS — no cloud account, no auth token, no bill.

A few things that make it interesting:

- **Fast and tiny** — built with Quarkus Native, it starts in ~24 ms and idles around ~13 MiB of RAM (vs ~3.3 s and ~143 MiB for LocalStack).
- **126 AWS services** on a single port — S3, SQS, DynamoDB, Lambda, RDS, IAM, and more.
- **Real Docker under the hood** — Lambda, ECS, EC2, RDS, ElastiCache, and friends run as actual containers instead of shallow mocks, which is why the Docker socket is mounted (see below).
- **LocalStack-compatible** — same port 4566, same wire protocols, same SDK calls. Existing `endpoint-url` overrides keep working.

## Why this compose file?

`docker-compose.yml` runs floci as a long-lived local service with **persistent storage**, so buckets, tables, queues, etc. survive container restarts.

| Setting | Purpose |
| --- | --- |
| `ports: 4566:4566` | Exposes the AWS endpoint on `localhost:4566`. |
| `FLOCI_STORAGE_MODE=persistent` | Enables durable on-disk storage instead of in-memory-only. |
| `FLOCI_STORAGE_PERSISTENT_PATH=/app/data` | Where floci writes its data inside the container. |
| `floci-data:/app/data` | Named volume that keeps the data on the host across `docker compose down/up`. |
| `/var/run/docker.sock` | Lets floci spawn real containers for Lambda/ECS/EC2/RDS and other Docker-backed services. |
| `restart: unless-stopped` | Restarts the container automatically (e.g. after a reboot) unless you stop it yourself. |

## Usage

```bash
# start in the background
docker compose up -d

# check that it's up
curl http://localhost:4566/_localstack/health   # served for LocalStack parity

# follow logs
docker compose logs -f

# stop (data in the floci-data volume is preserved)
docker compose down
```

Then point any AWS client at it:

```bash
aws --endpoint-url http://localhost:4566 s3 mb s3://my-bucket
aws --endpoint-url http://localhost:4566 s3 ls
```

Any credentials/region work — e.g. `AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test`.

## Alternative: plain `docker run`

It can still be launched without Compose:

```bash
docker run -d --name floci \
  -p 4566:4566 \
  -e DOCKER_HOST=unix:///var/run/docker.sock \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v floci_data:/app/data \
  floci/floci:latest
```

Note: the `docker run` variant doesn't set `FLOCI_STORAGE_MODE=persistent` — add `-e FLOCI_STORAGE_MODE=persistent` if you want the same durable-storage behavior as the Compose setup.
