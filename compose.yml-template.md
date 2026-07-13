# Docker Compose Service Template

This template provides a consistent starting point for self-hosted Docker Compose projects. It is designed for persistent services, rootless-friendly deployments, Caddy-based reverse proxying, shared external database networks, explicit image pinning, and optional container hardening.

The template is intentionally verbose. Remove unused options rather than leaving empty keys in the final Compose file.

---

## Directory layout

A typical project should look like this:

```text
[directory-name]/
├── compose.yml
├── .env
├── config.example.env
└── data/
    └── [service-name]/
```

Recommended conventions:

- The directory name, Compose project name, and primary service name should normally match the application.
- Store project-specific persistent data under `./data/`.
- Keep `.env` private.
- Commit `config.example.env` with safe placeholder values.
- Do not commit application data, credentials, tokens, private keys, or generated configuration containing credentials.

---

## Compose filename convention

Use `compose.yml` as the standard filename for each project.

Modern Docker Compose recognizes `compose.yml` automatically, so commands can be run without specifying `-f`:

```bash
docker compose up -d
docker compose pull
docker compose logs -f
```

Using the same filename across every project keeps directory layouts predictable, simplifies documentation and scripts, and avoids mixing older conventions such as `docker-compose.yml` with newer Compose usage.

Use an alternate filename only when the project intentionally contains multiple Compose configurations, such as:

```text
compose.yml
compose.override.yml
compose.production.yml
```

---

## Compose template

```yaml
# [directory-name] compose # this comment is required by this template

name: [directory-name] # required by this template; explicitly fixes the Compose project name

services:
  [service-name]: # use image-name
    image: [image-name]:[specific-version]@sha256:[digest] # pin both a human-readable version tag and its digest; the digest determines the exact image content; update both together; avoid the latest tag

    # container general config
    container_name: [service-name]

    init: true # optional; useful for applications that do not properly reap child processes

    depends_on: # optional; only used when the service requires another service and that service has a healthcheck
      [service]: # example
        condition: service_healthy
        restart: true # requires Compose 2.17+; restart this service when Compose explicitly updates the dependency; this is not crash monitoring

    restart: unless-stopped # required by this template; default for persistent services; use restart: "no" for one-shot services

    # command: # optional; remove when unused

    env_file: # optional; remove when unused
      - ./config.env # example only; normally use a service-specific environment file if direct container injection is needed

    environment: # optional; remove when unused
      # Project .env or --env-file supplies Compose interpolation values.
      # Service env_file injects variables directly into the container.
      # Values under environment override matching values from env_file.
      #
      # Structure:
      # VARIABLE_NAME: ${VARIABLE_NAME}
      #
      # Do not place passwords, tokens, or private keys directly in compose.yml.
      # Reference values and commonly reused variables from .env.
      TZ: ${TZ:?TZ must be set} # example; use only when supported by the image
      PUID: ${PUID:?PUID must be set} # optional; use only when documented by the image
      PGID: ${PGID:?PGID must be set} # optional; use only when documented by the image

    # container security
    # Optional: do not enable these settings wholesale.
    # Test each setting against the image and retain only those the application supports.
    read_only: true

    tmpfs:
      - /tmp:size=128m,mode=1777
      # - /run:size=16m,mode=755 # add only when required

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

    # cap_add:
    #   - NET_BIND_SERVICE # add only when documented and genuinely required

    # storage config
    # Optional.
    # Prefer bind mounts when direct host access, backup, or migration is useful.
    # Prefer named volumes when the data should be managed entirely by Docker.
    #
    # Default host location:
    # ./data/[service-name]
    volumes:
      - ./data/[service-name]:/dir/to/app # example
      - ./data/[service-name]/read-only-file.ext:/dir/to/app/read-only-file.ext:ro # example

    # network config
    # Specifying networks replaces automatic attachment to the implicit default network.
    # Include "default" explicitly when project-local service communication is required.
    networks: # optional; remove when the implicit default network is sufficient
      - default # include when project-local service communication is required
      - caddy # required when Caddy reverse-proxies this service
      - postgres18 # required when intentionally using a private database hosted in another Compose project

    ports: # optional; only include when host publishing is genuinely required
      - target: "65533"
        published: "65533"
        host_ip: "127.0.0.1" # IPv4 loopback only; publish separately on ::1 if required
        protocol: "tcp"

    labels: # required only when consumed by Caddy or another integration
      caddy: "[hostname].domain.tld"
      caddy.reverse_proxy: "{{upstreams [portnumber]}}"
      caddy_ingress_network: caddy # tells Caddy which shared Docker network to use
      # add other integration-specific labels here

    # healthcheck
    # Only use a documented, meaningful, side-effect-free readiness check.
    healthcheck:
      test: ["CMD-SHELL", "documented-command || exit 1"] # CMD is preferred; CMD-SHELL is required for shell expressions
      start_interval: 5s # requires Docker Engine 25+ / API 1.44+ and a sufficiently recent Compose plugin
      start_period: 30s
      interval: 30s
      timeout: 10s
      retries: 3

networks: # keep the default Compose network above external network declarations
  default:

  # Declare external networks only when a service requires cross-project access.
  # External networks must already exist and are not managed by this project.
  # Keep external network declarations alphabetized.
  caddy:
    external: true

  postgres18:
    external: true
```

---

## Image pinning

Use both a readable version tag and an immutable digest:

```yaml
image: example/application:1.2.3@sha256:0123456789abcdef...
```

The tag communicates the intended application version. The digest determines the exact image content Docker will run.

When updating an image:

1. Select the new version tag.
2. Pull or inspect the image to obtain its digest.
3. Update both the tag and digest together.
4. Review upstream release notes.
5. Validate the rendered Compose configuration.
6. Recreate the service and verify its health.

Avoid `latest` because it does not identify a predictable release.

---

## `.env` and `config.example.env`

Docker Compose uses environment files in two distinct ways.

### Project `.env`

A file named `.env` beside `compose.yml` is normally used for Compose interpolation:

```yaml
environment:
  TZ: ${TZ:?TZ must be set}
  PUID: ${PUID:?PUID must be set}
```

Example private `.env`:

```dotenv
TZ=America/Chicago
PUID=1001
PGID=989

POSTGRES_HOST=postgres18
POSTGRES_PORT=5432
POSTGRES_DATABASE=application
POSTGRES_USER=application
POSTGRES_PASSWORD=replace-with-private-value
```

Compose reads these values before it creates the container and substitutes them into `compose.yml`.

The `.env` file is not automatically injected wholesale into the container. Only variables referenced by the Compose file, or passed through another supported mechanism, are provided to the service.

You can use a different interpolation file explicitly:

```bash
docker compose --env-file ./production.env config
docker compose --env-file ./production.env up -d
```

Use the same `--env-file` option consistently for validation and deployment.

### Service `env_file`

A service-level `env_file` injects the listed variables directly into that container:

```yaml
services:
  application:
    env_file:
      - ./application.env
```

Example `application.env`:

```dotenv
APP_LOG_LEVEL=info
APP_FEATURE_ENABLED=true
APP_WORKERS=4
```

This is different from the project `.env` file:

- Project `.env`: primarily supplies values used while Compose renders the configuration.
- Service `env_file`: sends variables directly into the container.
- `environment`: explicitly defines container environment variables and overrides matching values from `env_file`.

Prefer `environment` when you want the Compose file to document exactly which values are passed to the container. Prefer `env_file` when an application has a large, application-specific variable set that would make the Compose file difficult to read.

### `config.example.env`

Commit an `config.example.env` file that documents every required variable without containing usable credentials.

Example:

```dotenv
# Host and locale
TZ=America/Chicago
PUID=1001
PGID=989

# Database connection
POSTGRES_HOST=postgres18
POSTGRES_PORT=5432
POSTGRES_DATABASE=application
POSTGRES_USER=application
POSTGRES_PASSWORD=CHANGE_ME

# Application
APP_LOG_LEVEL=info
APP_PUBLIC_URL=https://application.domain.tld
```

Good `config.example.env` practices:

- Include every variable required to deploy the project.
- Use safe placeholders such as `CHANGE_ME`.
- Include comments explaining non-obvious values.
- Keep variable names and ordering synchronized with `.env`.
- Do not include real passwords, tokens, private keys, internal recovery codes, or copied production values.
- Use realistic non-sensitive defaults where appropriate.
- Clearly identify optional variables.
- Update `config.example.env` whenever the Compose file gains or removes a variable.

A new deployment can then be initialized with:

```bash
cp example.env .env
chmod 600 .env
```

Edit `.env` before starting the project.

### Required and optional interpolation

Require a value and provide an error message:

```yaml
environment:
  TZ: ${TZ:?TZ must be set}
```

Use a default value:

```yaml
environment:
  LOG_LEVEL: ${LOG_LEVEL:-info}
```

Pass an empty value when unset:

```yaml
environment:
  OPTIONAL_SETTING: ${OPTIONAL_SETTING-}
```

For important deployment settings, required interpolation is preferable because Compose fails early instead of silently starting with an incomplete configuration.

---

## `.gitignore`

A project containing private environment files and bind-mounted application data should normally include:

```gitignore
.env
*.env
!config.example.env

data/
backup/
backups/

*.log
```

If the project intentionally tracks a non-sensitive service environment file, add a specific exception instead of broadly committing all `.env` files.

---

## Networking conventions

### Default network

Every Compose project normally receives an implicit default network. Once a service has an explicit `networks` list, automatic attachment is replaced.

Include `default` when the service still needs to communicate with other services in the same Compose project:

```yaml
networks:
  - default
  - caddy
```

### Caddy network

Attach a service to the external `caddy` network when Caddy needs to reach it directly:

```yaml
networks:
  - caddy
```

Use the ingress label when multiple networks are attached:

```yaml
labels:
  caddy_ingress_network: caddy
```

This prevents the proxy integration from choosing the wrong network address.

### Database networks

A shared, external database network is appropriate when a private database service is hosted in a separate Compose project:

```yaml
networks:
  - postgres18
```

The versioned PostgreSQL network name makes major-version migrations and parallel operation clearer:

```text
postgres17
postgres18
```

The application should normally connect using the database service name or a documented network alias, not a container IP address.

### Keycloak and other externally available identity providers

Applications using OIDC or OAuth generally do not need a shared Docker network with Keycloak when Keycloak is already available through the reverse proxy.

Use the externally valid issuer URL:

```dotenv
OIDC_ISSUER_URL=https://keycloak.domain.tld/realms/example
```

This keeps discovery, redirect URLs, token validation, and issuer matching aligned with the public endpoint.

---

## Ports

Do not publish a port merely because the application listens on it.

When Caddy reaches the service over a shared Docker network, `ports` is normally unnecessary.

Use loopback publishing for host-only administration or debugging:

```yaml
ports:
  - target: "8080"
    published: "8080"
    host_ip: "127.0.0.1"
    protocol: "tcp"
```

This exposes the service only on IPv4 loopback. Add a separate `::1` mapping if IPv6 loopback access is required and supported by the deployment.

---

## Security options

These controls are application-dependent:

```yaml
read_only: true

tmpfs:
  - /tmp:size=128m,mode=1777

security_opt:
  - no-new-privileges:true

cap_drop:
  - ALL
```

Do not apply them blindly.

An application may require writable paths such as:

- `/tmp`
- `/run`
- `/var/run`
- `/var/cache/...`
- an application-specific configuration or state directory

Mount only the paths that genuinely need to be writable.

If an application requires a Linux capability, add back only that capability:

```yaml
cap_drop:
  - ALL

cap_add:
  - NET_BIND_SERVICE
```

Confirm the requirement from the image documentation or through testing.

---

## Healthchecks and startup dependencies

Use healthchecks to represent actual service readiness rather than simple process existence.

A useful healthcheck should be:

- Documented or based on a stable application endpoint.
- Side-effect-free.
- Available inside the container.
- Fast enough to run repeatedly.
- Able to distinguish startup from readiness.

Prefer exec-form `CMD` when no shell logic is needed:

```yaml
healthcheck:
  test: ["CMD", "application", "healthcheck"]
```

Use `CMD-SHELL` only for shell operators, pipes, substitutions, or compound expressions:

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:8080/health || exit 1"]
```

A dependency can wait for a healthy service:

```yaml
depends_on:
  database:
    condition: service_healthy
    restart: true
```

The `restart: true` dependency option tells Compose to restart the dependent service after Compose explicitly updates or restarts the dependency. It is not a runtime crash-monitoring mechanism.

---

## Validation and deployment

Validate the Compose file before deployment:

```bash
docker compose config --quiet
```

Inspect the fully rendered configuration:

```bash
docker compose config
```

Review the resolved image reference:

```bash
docker compose config --images
```

Pull images without starting services:

```bash
docker compose pull
```

Start or update the project:

```bash
docker compose up -d
```

Review status:

```bash
docker compose ps
```

Follow logs:

```bash
docker compose logs -f
```

For rootless Docker, run these commands as the user who owns the rootless Docker daemon and project files.

---

## Pre-deployment checklist

- Replace every bracketed placeholder.
- Remove unused keys and example sections.
- Confirm the image tag and digest correspond to the same release.
- Review upstream image documentation.
- Confirm every required `.env` variable is present.
- Ensure `config.example.env` contains no private values.
- Verify bind-mount directory ownership and permissions.
- Confirm external networks already exist.
- Attach only the networks the service genuinely needs.
- Avoid publishing ports that Caddy can reach internally.
- Test security options individually.
- Use a meaningful readiness healthcheck.
- Run `docker compose config --quiet`.
- Inspect `docker compose config` before deployment.
- Back up persistent data before major application or database upgrades.
