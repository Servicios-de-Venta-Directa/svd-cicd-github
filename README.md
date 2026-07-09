# svd-cicd-github

Acciones GitHub reutilizables para centralizar la lógica CI/CD de los repositorios de aplicaciones SVD.

## Propósito

Este repositorio público contiene **composite actions** (acciones reutilizables) que se comparten entre múltiples repositorios de aplicaciones. Facilita la implementación de un pipeline CI/CD consistente: construir imágenes Docker, enviar a registro, actualizar valores de ArgoCD, y promover a producción.

## Acciones Incluidas

### 1. `docker-build`

Construye una imagen Docker, la etiqueta con timestamp (UTC-3) y la envía a Docker Hub.

- **Entradas:** Docker credentials, nombre imagen, ruta Dockerfile, rama Git
- **Salidas:** `image_tag` (timestamp), `image_name_tag`, `image_with_repo` (full path)
- **Etiquetado:**
  - Main: `YYYYMMDDHHmmss`
  - Feature/bugfix: `YYYYMMDDHHmmss-branch-normalized`

### 2. `argocd-update`

Actualiza `argocd-applications/values.yaml` en el repo svd-cicd con la nueva etiqueta y rama, luego commitea y empuja.

- **Entradas:** GitHub token, nombre app, image tag, nombre rama
- **Actualiza:** `.tag` y `.branch` de la app en values.yaml
- **Commit:** Automático con mensaje e identidad `github-actions[bot]`

### 3. `retag-image`

Descarga la imagen dev, la re-etiqueta eliminando el sufijo de rama, y la envía como versión prod.

- **Entradas:** Docker credentials, nombre imagen, dev image tag (con rama)
- **Salidas:** `prod_tag` (timestamp sin rama)
- **Lógica:** `20260709124530-develop` → `20260709124530`

## Uso

En tu repositorio, crea `.github/workflows/docker-image-ci.yml`:

```yaml
name: Docker Image CI

on:
  push:
    branches:
      - develop
      - "SVD-*"
      - "bugfix/**"
  pull_request:
    branches:
      - main
  workflow_dispatch: {}

jobs:
  build:
    if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.docker_build.outputs.image_tag }}
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        id: docker_build
        uses: Servicios-de-Venta-Directa/svd-cicd-github/.github/actions/docker-build@main
        with:
          docker_repo_user: ${{ secrets.DOCKER_REPO_USER }}
          docker_repo_password: ${{ secrets.DOCKER_REPO_PASSWORD }}
          image_name: tu-imagen
          branch_name: ${{ github.ref_name }}

  deploy-dev:
    if: github.ref_name == 'develop' || startsWith(github.ref_name, 'SVD-') || startsWith(github.ref_name, 'bugfix/')
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Update ArgoCD (dev)
        uses: Servicios-de-Venta-Directa/svd-cicd-github/.github/actions/argocd-update@main
        with:
          gitops_token: ${{ secrets.GITOPS_TOKEN }}
          gitops_app_name: tu-app-dev
          image_tag: ${{ needs.build.outputs.image_tag }}
          branch_name: ${{ github.ref_name }}

  retag-and-deploy-prod:
    if: github.event.pull_request.merged == true && github.event.pull_request.base.ref == 'main'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Retag image for production
        id: retag
        uses: Servicios-de-Venta-Directa/svd-cicd-github/.github/actions/retag-image@main
        with:
          docker_repo_user: ${{ secrets.DOCKER_REPO_USER }}
          docker_repo_password: ${{ secrets.DOCKER_REPO_PASSWORD }}
          image_name: tu-imagen
          dev_image_tag: ${{ needs.build.outputs.image_tag }}

      - name: Update ArgoCD (prod)
        uses: Servicios-de-Venta-Directa/svd-cicd-github/.github/actions/argocd-update@main
        with:
          gitops_token: ${{ secrets.GITOPS_TOKEN }}
          gitops_app_name: tu-app-prod
          image_tag: ${{ steps.retag.outputs.prod_tag }}
          branch_name: ${{ github.ref_name }}
```

## Requisitos

Cada repositorio debe tener estos **secrets** configurados:

- `DOCKER_REPO_USER` — Usuario Docker Hub
- `DOCKER_REPO_PASSWORD` — Contraseña Docker Hub
- `GITOPS_TOKEN` — Token GitHub con acceso a `Servicios-de-Venta-Directa/svd-cicd`

## Compatibilidad

- ✅ GitHub Free (composite actions no requieren organización-level secrets)
- ✅ Linux runners
- ✅ Múltiples repositorios sin duplicación

## Licencia

Apache 2.0. Ver [LICENSE](LICENSE).

## Mantenimiento

Cambios a estas acciones se replican automáticamente a todos los repos que las usen (consumidores siempre hacen referencia a `@main`).
