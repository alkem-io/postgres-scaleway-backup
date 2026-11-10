# Postgres backup image

The container image that Alkemio's Postgres backup and restore jobs run in. It is published to Docker Hub as [`alkemio/postgres-scaleway-backup`](https://hub.docker.com/r/alkemio/postgres-scaleway-backup).

The image is a toolbox. It holds the tools; the backup logic lives in a script that each job mounts from a ConfigMap in its own cluster.

## What is in the image

Built from [`Dockerfile-Postgres`](Dockerfile-Postgres), on the official `postgres` image:

| Tool | Version | Used for |
| --- | --- | --- |
| `pg_dump`, `pg_restore`, `psql` | `POSTGRES_VERSION` in the Dockerfile | Dumping and restoring databases |
| `aws` (AWS CLI) | Latest at build time | Uploading to and downloading from S3 |
| `s3cmd` | Latest at build time | S3 access |
| `age` | `AGE_VERSION` in the Dockerfile | Optional encryption of dumps |
| `kubectl` | Latest stable at build time | Cluster access from a job |
| `gzip` | From the base image | Compression |

The AWS CLI is preconfigured for Scaleway Object Storage in `nl-ams` through [`scaleway/config`](scaleway/config). Credentials are not in the image: jobs pass them as environment variables.

## The Postgres version rule

`pg_dump` refuses to dump a server of a newer major version than itself. It can dump older servers.

So the image must move to a new Postgres major **before** any server does. Release the image first, point the backup jobs at it, check a backup, and only then upgrade the server.

## How jobs use it

A job overrides the image's command with its own script:

```yaml
containers:
  - name: postgres-backup
    image: alkemio/postgres-scaleway-backup:v1.1.0
    args:
      - /bin/bash
      - /scripts/do-pg-backup.sh
    volumeMounts:
      - name: backup-scripts-volume
        mountPath: /scripts
volumes:
  - name: backup-scripts-volume
    configMap:
      name: postgres-backup-conf
```

The jobs that use this image:

| Environment | Repository | Manifests |
| --- | --- | --- |
| Sandbox | [alkem-io/dev-orchestration](https://github.com/alkem-io/dev-orchestration) | `01-alkemio-platform/.../third-party/storage/postgres/` |
| Acceptance, production | [alkem-io/infrastructure-operations](https://github.com/alkem-io/infrastructure-operations) | `orchestration/.../third-party/storage/postgres/` |

Do not rely on the image's default command. It is [`resources/perform-backup.sh`](resources/perform-backup.sh), a MySQL script inherited from the upstream project, and it does nothing useful against Postgres.

## Releasing a new image

1. Change `POSTGRES_VERSION` in `Dockerfile-Postgres` (and `AGE_VERSION` if needed) and merge to `master`.
2. Publish a GitHub release with a tag of the form `v1.2.3`. The tag becomes the image tag.
3. The [release workflow](.github/workflows/build-release-postgres-docker-hub.yml) builds the image, checks that `pg_dump` matches `POSTGRES_VERSION` and that the other tools run, then pushes `alkemio/postgres-scaleway-backup:<tag>`.
4. Change the image tag in the job manifests listed above.

The workflow can also be run by hand from the Actions tab with a tag.

A published tag is never replaced: the workflow fails if the tag already exists. Jobs pin tags, so an image must not change underneath them. There is no `latest` tag.

To build locally:

```bash
docker build -f Dockerfile-Postgres -t postgres-scaleway-backup:local .
```

## Versions

| Tag | Postgres client | Built by |
| --- | --- | --- |
| `alkemio/postgres-scaleway-backup:v1.1.0` | 18.6 | Release workflow |
| `svetoslavpetkov23/postgres-scaleway-backup:v1.0.3` | 18.6 | By hand |
| `svetoslavpetkov23/postgres-scaleway-backup:v1.0.2` | 17.5 | By hand |
| `alkemio/postgres-scaleway-backup:v1.0.1` | 12 | By hand |

Use `v1.1.0` or later. The two images in the personal namespace exist only because nothing built this image from CI before `v1.1.0`.

## Inherited from upstream

This repository is a fork of [kubernetes-cloud-mysql-backup](https://github.com/benjamin-maynard/kubernetes-cloud-mysql-backup). These parts came with it and are not used by Alkemio:

- `Dockerfile-MySQL` and `resources/perform-backup.sh`: the MySQL backup image. No workflow builds it and no environment runs it. Its environment variables are described in the upstream README.
- `tests/` and `.github/workflows/run-pr-tests.yaml`: the upstream test setup. It targets infrastructure this repository does not have and does not run.
