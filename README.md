# Suantechs/.github — CI/CD central

Reusable workflows (`workflow_call`) para estandarizar el CI/CD de toda la
plataforma. En vez de copiar `build-image.yml` + `deploy.yml` en cada repo, cada
proyecto los **llama** con ~10 líneas y `secrets: inherit`.

> Ticket: **STP-140**. Arquetipos soportados: **sitio Astro** (sin migraciones) y
> **servicio NestJS** (migraciones + paquetes privados `@suantechs/*`).

## Secrets (a nivel organización)

Heredados con `secrets: inherit` — **no** hay que ponerlos por repo:

| Secret | Uso |
|---|---|
| `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY` | SSH al VPS para el deploy |
| `GHCR_USER`, `GHCR_TOKEN` | login a GHCR en el VPS (pull de imágenes privadas) |
| `DISCORD_WEBHOOK_URL` | avisos de deploy (iniciado/ok/fallido) |
| `NODE_AUTH_TOKEN` | **por repo** — solo arquetipo NestJS con paquetes privados |

## Workflows

### `build-image.yml`
Construye y publica `ghcr.io/suantechs/<image_name>` con tags `latest` y `<sha>`.
El Dockerfile (que corre el build) es el gate de CI.

| Input | Default | Descripción |
|---|---|---|
| `image_name` | — (req) | nombre de la imagen sin registry/tag |
| `context` | `.` | contexto de build |
| `dockerfile` | `./Dockerfile` | ruta al Dockerfile |
| `build_args` | `''` | build-args multilínea |
| `needs_node_auth` | `false` | inyecta `node_auth_token` (paquetes privados) |

### `deploy.yml`
SSH al VPS: login GHCR → pull (o retag de `<sha>` para rollback) → migraciones
opcionales → `compose up` → smoke check → avisos a Discord.

| Input | Default | Descripción |
|---|---|---|
| `image_name` | — (req) | imagen en `ghcr.io/suantechs/<image_name>` |
| `app_dir` | — (req) | dir del stack en el VPS (ej. `/opt/apps/enki`) |
| `compose_file` | `docker-compose.prod.yml` | archivo compose |
| `image_tag` | `latest` | `latest` o un `<sha>` para rollback |
| `service` | `app` | servicio del compose a migrar/actualizar |
| `run_migrations` | `false` | corre migraciones antes de levantar (NestJS) |
| `migration_cmd` | `npm run migration:run:prod` | comando de migración |
| `migration_deps` | `postgres` | servicios a levantar antes de migrar |
| `smoke_url` | `''` | URL a verificar tras el deploy (vacío = sin smoke) |
| `app_label` | `image_name` | nombre legible para Discord |

## Uso desde un repo consumidor

`.github/workflows/build-image.yml`:

```yaml
name: Build image
on:
  push: { branches: [master] }
  workflow_dispatch:
jobs:
  build:
    # Obligatorio: el caller concede los permisos del GITHUB_TOKEN al reusable.
    # Sin packages: write, el run falla al arrancar (startup_failure).
    permissions:
      contents: read
      packages: write
    uses: Suantechs/.github/.github/workflows/build-image.yml@master
    with:
      image_name: enki
    secrets: inherit
```

`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  workflow_dispatch:
    inputs:
      image_tag: { description: 'latest o <sha> para rollback', default: latest, required: false }
  workflow_run:
    workflows: ['Build image']
    types: [completed]
    branches: [master]
jobs:
  deploy:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: Suantechs/.github/.github/workflows/deploy.yml@master
    with:
      image_name: enki
      app_dir: /opt/apps/enki
      smoke_url: https://enkipanama.com/
      image_tag: ${{ github.event.inputs.image_tag || 'latest' }}
    secrets: inherit
```

### Arquetipo NestJS

Añadir en el build `needs_node_auth: true` y en el deploy
`run_migrations: true`, `smoke_url: http://localhost:<puerto>/api/version`.

## Rollback

`Actions → Deploy → Run workflow → image_tag = <sha>` de un build previo. El
deploy baja ese `<sha>`, lo retagea como `:latest` local y reinicia.
