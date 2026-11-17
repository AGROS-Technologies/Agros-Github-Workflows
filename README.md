# Agros-Github-Workflows

Workflows reutilizables y acciones compuestas para estandarizar **CI/CD, seguridad y despliegues** en los repositorios de AGROS Technologies.

> Este repositorio contiene automatizaciones centrales: los workflows se ejecutan cuando un repositorio consumidor los invoca.

## ¿Qué incluye?

| Área | Capacidades |
| --- | --- |
| **Integración continua** | Pipelines para Python, Node.js y Java: análisis de secretos y SAST, auditoría de dependencias, pruebas y construcción/publicación opcional de imágenes Docker. |
| **Seguridad de aplicaciones** | Gitleaks, Semgrep, `pip-audit`, `npm audit`, OWASP Dependency-Check, Trivy y análisis DAST pasivo con OWASP ZAP. |
| **Ramas y releases** | Creación de ramas vinculadas a ClickUp, validaciones de pull requests, propagación tras fusiones y automatización de versiones/release. |
| **Despliegue** | Publicación de imágenes en GHCR; despliegues y retirada de ramas en VPS con Docker Compose, Nginx, DNS de Hostinger y Certbot. Incluye CD específico de Hub Auth. |

## Inicio rápido

Desde el repositorio de la aplicación, crea un workflow que llame al workflow reutilizable apropiado. Por ejemplo, para CI de Node.js:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  ci:
    uses: AGROS-Technologies/Agros-Github-Workflows/.github/workflows/ci-node.yml@main
    with:
      image_name: ghcr.io/<organizacion>/<imagen>
    secrets:
      ghcr_token: ${{ secrets.GITHUB_TOKEN }}
```

El ejemplo supone que el proyecto tiene `package-lock.json`, un script `test` en `package.json` y un Dockerfile. Si no necesitas construir y publicar una imagen, omite `image_name`; si se publica en GHCR, el llamador debe conceder `packages: write`.

Los workflows de este repositorio usan `workflow_call`: no se disparan por un `push` propio. Los eventos (`pull_request`, `push`, `schedule`, etc.) se configuran en el workflow llamador.

## Workflows disponibles

| Necesidad | Workflow |
| --- | --- |
| CI Python, Node.js o Java | [`ci-python.yml`](.github/workflows/ci-python.yml), [`ci-node.yml`](.github/workflows/ci-node.yml), [`ci-java.yml`](.github/workflows/ci-java.yml) |
| Análisis dinámico de una aplicación desplegada | [`dast.yml`](.github/workflows/dast.yml) |
| Despliegue general de una rama en VPS | [`deploy-branch.yml`](.github/workflows/deploy-branch.yml) |
| Despliegue de Hub Auth | [`cd.yml`](.github/workflows/cd.yml) |
| Crear o retirar ramas de despliegue, actualizar variables | [`create-branch.yml`](.github/workflows/create-branch.yml), [`teardown-branch.yml`](.github/workflows/teardown-branch.yml), [`update-env.yml`](.github/workflows/update-env.yml) |
| Validar y procesar pull requests | [`pr-validate.yml`](.github/workflows/pr-validate.yml), [`pr-merged.yml`](.github/workflows/pr-merged.yml), [`succesed-merge.yml`](.github/workflows/succesed-merge.yml) |
| Crear una rama de release | [`create-release.yml`](.github/workflows/create-release.yml) |

## Documentación

- [Guía de funcionamiento de workflows y actions](docs/flujo-actions-workflows.md): inventario, secuencias, herramientas, secretos y consideraciones operativas.
- [CI/CD y seguridad por stack](docs/ci-security-stacks.md): configuración e integración recomendada de CI, SCA y DAST.
- [Workflows](.github/workflows/) y [actions compuestas](.github/actions/): definiciones disponibles para reutilizar.

## Uso y versionado

Las referencias de ejemplo usan `@main`, lo que permite consumir la versión más reciente, pero también hace que los cambios en `main` puedan afectar a los consumidores. Para fijar una versión reproducible, usa una etiqueta aprobada o un SHA de commit.

Configura en el repositorio consumidor solo los secretos requeridos por el workflow elegido. No incluyas credenciales en archivos del repositorio.

## Copyright and Usage

Copyright © 2026 AGROS Technologies. All rights reserved.

This repository is publicly available for educational, academic, and inspection purposes.

No license is granted to use, reproduce, modify, distribute, or commercially exploit this software without prior written authorization from the respective copyright holders.

Contributions and third-party components remain subject to their respective ownership and licensing terms.
