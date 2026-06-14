# Jrawler Production Deployment

This directory contains the Docker Compose deployment used by Jenkins for a
DigitalOcean Droplet.

## One-time Droplet setup

1. Install Docker and the Docker Compose plugin.
2. Create the deployment directory:

   ```bash
   mkdir -p /opt/jrawler
   ```

3. Copy `deploy/.env.prod.example` to `/opt/jrawler/.env` and fill secrets:

   ```bash
   cp deploy/.env.prod.example /opt/jrawler/.env
   ```

4. Make sure the Jenkins SSH key can connect to the Droplet user configured in
   `Jenkinsfile`.

## Jenkins credentials

Create these Jenkins credentials:

- `digitalocean-registry`: username/password credential for Docker registry
  login. For DigitalOcean Container Registry, use a token-capable login.
- `digitalocean-droplet-ssh`: SSH private key credential for the Droplet.

## Jenkins agent requirements

The pipeline runs Maven and Node inside Docker containers, so the Jenkins agent
does not need local Java, Maven, Node, or npm installations. It does need:

- Docker CLI and access to the Docker daemon
- Docker Compose plugin available on the Droplet
- network access to the Docker registry, Maven Central, and npm registry

## Jenkinsfile values to change

Update these values before the first run:

- `REGISTRY`, for example `registry.digitalocean.com/my-registry`
- `DEPLOY_HOST`, the Droplet IP or DNS name
- `DEPLOY_USER`, usually `root` or a deploy user
- credential IDs if your Jenkins uses different names

On every `main` build Jenkins will test, build both images, push versioned and
`latest` tags, upload the compose file, and restart the stack.
