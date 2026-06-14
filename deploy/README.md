# Jrawler Production Deployment

This directory contains the Docker Compose deployment used by Jenkins on the
same DigitalOcean instance that runs Nexus and the application stack.

## One-time host setup

1. Install Docker and the Docker Compose plugin.
2. Create a Nexus Docker hosted repository, for example exposed as
   `nexus.example.com:8082`.
3. If Nexus uses plain HTTP or a self-signed certificate, configure the Docker
   daemon on the host to trust it before running the pipeline.
4. Create the deployment directory and make it writable by the Jenkins user:

   ```bash
   sudo mkdir -p /opt/jrawler
   sudo chown jenkins:jenkins /opt/jrawler
   ```

5. Copy `deploy/.env.prod.example` to `/opt/jrawler/.env` and fill secrets:

   ```bash
   cp deploy/.env.prod.example /opt/jrawler/.env
   ```

## Jenkins credentials

Create these Jenkins credentials:

- `nexus-docker-registry`: username/password credential for Nexus Docker
  registry login.

## Jenkins agent requirements

The pipeline runs Maven and Node inside Docker containers, so the Jenkins agent
does not need local Java, Maven, Node, or npm installations. It does need:

- Docker CLI and access to the Docker daemon
- Docker Compose plugin
- network access to Nexus, Maven Central, and npm registry

## Jenkinsfile values to change

Update these values before the first run:

- `NEXUS_REGISTRY`, for example `nexus.example.com:8082`
- `DEPLOY_DIR`, if you do not want `/opt/jrawler`
- credential IDs if your Jenkins uses different names

On every `master` build Jenkins will test, build both images, push versioned and
`latest` tags to Nexus, copy the compose file into `DEPLOY_DIR`, and restart the
local Docker Compose stack.

For a classic Jenkins Pipeline job, enable GitHub webhook triggering. For a
Multibranch Pipeline, configure the GitHub webhook/scan trigger so pushes to
`master` run this Jenkinsfile.
