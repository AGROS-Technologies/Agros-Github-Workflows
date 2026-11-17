# Arquitectura de CI/CD y seguridad por stack

## Capas de reutilización

1. **Proyecto consumidor:** su workflow invoca el workflow reutilizable de este repositorio indicando el stack, las opciones del proyecto y los secretos necesarios.
2. **Workflows centrales:** `.github/workflows/ci-python.yml`, `ci-node.yml`, `ci-java.yml` y `dast.yml` orquestan el orden del pipeline; no implementan los pasos de compilación o escaneo.
3. **Actions centrales:** `.github/actions/security-scan`, `python-ci`, `node-ci`, `java-ci`, `docker-build-scan-publish` y `dast-zap` encapsulan los pasos. Dentro de cada action se integran las herramientas de terceros (Gitleaks, Semgrep, pip-audit, npm audit, OWASP Dependency-Check, Trivy y ZAP).

Las actions por stack también pueden consumirse desde otros workflows, pero la interfaz recomendada para los repositorios de aplicaciones es el workflow correspondiente. Los workflows centrales son independientes de una aplicación; la configuración específica de cada proyecto debe permanecer en el repositorio consumidor.

Todos los workflows y composite actions de este repositorio usan referencias centrales `agrofusion-mdu-prototipo/Ops-Github-Workflows/...@main`. Cambia `main` por un tag o SHA cuando se requiera fijar una versión de la interfaz reutilizable.

## CI

Usa una entrada de CI acorde con el stack de la aplicación:

| Workflow | Stack y SCA | Requisitos del proyecto |
| --- | --- | --- |
| `ci-python.yml` | Python con `pip-audit` | `requirements.txt` (o configurar `requirements_file`) y pruebas ejecutables con `pytest` por defecto |
| `ci-node.yml` | Node.js/TypeScript con `npm audit` | `package-lock.json` y script de npm llamado `test` por defecto |
| `ci-java.yml` | Java con OWASP Dependency-Check | Maven o Gradle; Gradle requiere `./gradlew` |

Ejemplo para Java:

```yaml
permissions:
  contents: read
  packages: write

jobs:
  ci:
    uses: agrofusion-mdu-prototipo/Ops-Github-Workflows/.github/workflows/ci-java.yml@main
    with:
      image_name: ghcr.io/mi-organizacion/mi-aplicacion
      java_version: "21"
      build_tool: maven
    secrets:
      ghcr_token: ${{ secrets.GHCR_TOKEN }}
```

Omite `image_name` y el secreto GHCR si el proyecto solo requiere los controles, el SCA y las pruebas, sin construir ni publicar una imagen. Si se publica una imagen, el llamador debe conceder `packages: write` al `GITHUB_TOKEN` o proporcionar un `ghcr_token` con ese permiso.

Los tres workflows ejecutan Gitleaks y Semgrep. Si `image_name` se proporciona, construyen la imagen, bloquean vulnerabilidades CRITICAL/HIGH con corrección disponible en Trivy y solo después la publican. npm audit bloquea HIGH/CRITICAL; Dependency-Check usa CVSS 7.0 como umbral. `pip-audit` no ofrece un filtro de severidad equivalente y por eso bloquea cualquier vulnerabilidad reportada, que es un criterio más estricto. En Node.js se puede elegir el script npm de pruebas mediante `test_script`. Los pipelines con imagen publican etiquetas SHA y de rama, y una etiqueta SemVer cuando existe `VERSION`.

Para cubrir la revisión semanal de SCA indicada en la matriz, el repositorio consumidor debe invocar su workflow de CI tanto en `pull_request` como en un evento `schedule`. Por ejemplo, agrega `schedule` junto a los triggers del workflow llamador:

```yaml
on:
  pull_request:
  schedule:
    - cron: "0 9 * * 1"
```

Las ejecuciones programadas hacen los análisis y las pruebas, pero no publican imágenes.

## CD y DAST

`cd.yml` conserva el despliegue específico de Hub Auth: ejecuta Alembic y comprueba `/api/health`. Para aplicaciones Java, Node.js u otras aplicaciones empaquetadas en Docker, usa el workflow `deploy-branch.yml`, que despliega la imagen de GHCR sin depender del lenguaje. Para despliegues de producción, ese flujo espera que exista `VERSION` en el repositorio.

`dast.yml` ejecuta OWASP ZAP Baseline de forma pasiva contra una URL ya desplegada. Bloquea alertas de riesgo alto (ZAP no define una categoría separada de riesgo crítico) y resume las alertas medias y bajas. Llámalo desde el workflow consumidor después del job de despliegue y pasa únicamente una URL de un entorno que el proyecto esté autorizado a probar:

```yaml
  dast:
    needs: deploy
    uses: agrofusion-mdu-prototipo/Ops-Github-Workflows/.github/workflows/dast.yml@main
    with:
      target_url: https://mi-aplicacion.dev.example.com
```
