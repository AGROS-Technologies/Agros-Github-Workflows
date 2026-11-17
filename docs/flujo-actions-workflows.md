# Guía de funcionamiento de workflows y actions

Este documento describe los flujos reutilizables y las acciones centralizadas del repositorio, qué trabajo realizan, qué servicios o herramientas utilizan y qué efectos producen en el repositorio o en la infraestructura.

## 1. Vista general

El repositorio centraliza automatizaciones para que otros repositorios las invoquen:

1. Un workflow del repositorio de aplicación decide cuándo ejecutar una tarea y llama a uno de los workflows de este repositorio.
2. El workflow central recibe opciones y secretos, y organiza los pasos en un runner de GitHub Actions.
3. Las acciones compuestas de `.github/actions/` realizan el trabajo detallado: validaciones, pruebas, análisis, publicación de imágenes o cambios en infraestructura.

Los 13 workflows disponibles usan `workflow_call`. Por tanto, no se disparan por sí solos ante un `push`, una solicitud de cambios o un horario: deben ser llamados por un workflow consumidor. Las acciones compuestas tampoco tienen disparadores propios.

Los workflows de CI comparten esta secuencia general:

1. Descargar el repositorio de aplicación.
2. Ejecutar Gitleaks y Semgrep.
3. Instalar o compilar el proyecto, auditar sus dependencias y ejecutar las pruebas.
4. Si se indicó una imagen, construirla, escanearla con Trivy y, si el evento de origen es `push`, publicarla en GHCR.

Las automatizaciones de ramas, releases, despliegue y retirada operan sobre GitHub, ClickUp, GHCR, una VPS y/o Hostinger, según el flujo.

## 2. Workflows reutilizables

| Workflow | Cuándo se usa | Trabajo principal |
| --- | --- | --- |
| [`ci-python.yml`](../.github/workflows/ci-python.yml) | CI de proyectos Python | Gitleaks, Semgrep, instalación de dependencias, `pip-audit`, pruebas y publicación opcional de imagen Docker. |
| [`ci-node.yml`](../.github/workflows/ci-node.yml) | CI de proyectos Node.js | Gitleaks, Semgrep, `npm ci`, `npm audit`, pruebas y publicación opcional de imagen Docker. |
| [`ci-java.yml`](../.github/workflows/ci-java.yml) | CI de proyectos Java | Gitleaks, Semgrep, compilación/pruebas Maven o Gradle, OWASP Dependency-Check y publicación opcional de imagen Docker. |
| [`dast.yml`](../.github/workflows/dast.yml) | Pruebas dinámicas de una aplicación ya desplegada | Ejecuta el análisis pasivo OWASP ZAP Baseline sobre la URL indicada. |
| [`cd.yml`](../.github/workflows/cd.yml) | Despliegue de Hub Auth | Despliega la aplicación empaquetada con Docker Compose, ejecuta migraciones, comprueba salud y puede hacer rollback. |
| [`deploy-branch.yml`](../.github/workflows/deploy-branch.yml) | Despliegue de una rama en DEV, TEST o PROD | Publica o actualiza DNS y despliega una imagen GHCR en una VPS detrás de Nginx y HTTPS. |
| [`teardown-branch.yml`](../.github/workflows/teardown-branch.yml) | Retirada de un despliegue de rama | Elimina el DNS y los recursos de esa rama en la VPS; no elimina la rama GitHub. |
| [`update-env.yml`](../.github/workflows/update-env.yml) | Actualización de configuración de una rama desplegada | Inserta o actualiza variables en el archivo `.env` correspondiente de la VPS. |
| [`pr-validate.yml`](../.github/workflows/pr-validate.yml) | Validación previa de una solicitud de cambios | Comprueba compatibilidad entre ramas, ramas de resolución pendientes y, para bugfix, alineación de versión. |
| [`pr-merged.yml`](../.github/workflows/pr-merged.yml) | Procesamiento posterior a una solicitud de cambios fusionada | Elimina ramas temporales, incrementa versiones de release y propaga cambios hacia `dev`. |
| [`succesed-merge.yml`](../.github/workflows/succesed-merge.yml) | Análisis de seguridad tras una fusión | Ejecuta Semgrep en la rama protegida. El nombre del archivo está escrito `succesed-merge`; la acción correspondiente se llama `success-merge`. |
| [`create-branch.yml`](../.github/workflows/create-branch.yml) | Creación controlada de ramas de trabajo | Valida una tarea ClickUp y crea una rama `feature/` o `bugfix/` desde la rama base elegida. |
| [`create-release.yml`](../.github/workflows/create-release.yml) | Inicio de una línea de release | Crea y publica una rama `release/`, una etiqueta Git y una imagen GHCR; luego avanza la versión de la rama por defecto. |

### 2.1 CI por lenguaje

Los tres workflows de CI aceptan el directorio del proyecto y las opciones propias del stack. `image_name` es opcional; si se omite, el flujo no construye ni publica una imagen.

| Stack | Valores predeterminados y requisitos | Criterio de fallo de auditoría |
| --- | --- | --- |
| Python | Python 3.11, `requirements.txt`, `pytest -q`. | `pip-audit` falla si encuentra vulnerabilidades reportadas. |
| Node.js | Node.js 22, `package-lock.json`, script `test`. | `npm audit` bloquea hallazgos HIGH y CRITICAL; los reportes se resumen y se guardan como artefacto. |
| Java | Java 21 y Maven; Gradle también está disponible y requiere `./gradlew`. | OWASP Dependency-Check usa por defecto `--failOnCVSS 7.0`; el reporte HTML se guarda como artefacto. |

En cada stack, Gitleaks examina secretos y Semgrep realiza SAST. Si se configura `image_name`, la imagen se construye y se analiza con Trivy antes de publicarse; el paso bloqueante falla ante vulnerabilidades CRITICAL y HIGH con corrección disponible, mientras que las que no tienen corrección se ignoran en ese paso y quedan incluidas en el reporte completo. En eventos `push`, se publican etiquetas basadas en SHA y rama; desde `main` o `release/*` también se intenta publicar la etiqueta SemVer tomada del archivo `VERSION`, si existe y aún no está publicada.

El workflow Java describe `dependency_check_args` como argumentos adicionales, aunque el valor predeterminado ya establece el umbral. La imagen de Trivy y las reglas Semgrep se pueden personalizar en los tres stacks.

### 2.2 Despliegues y seguridad dinámica

- **`deploy-branch.yml`:** recibe la rama, el ambiente y credenciales GHCR/VPS/Hostinger. Valida la pareja rama-ambiente, comprueba que la rama y la imagen existan, actualiza DNS, y prepara el contenedor, Nginx y Certbot en la VPS. PROD solo permite `main` o `release/*` y usa `VERSION` como etiqueta de imagen; DEV/TEST permiten `dev`, `test`, `feature/*`, `bugfix/*` y `hotfix/*`.
- **`cd.yml`:** flujo específico de Hub Auth, no es un despliegue genérico. Traduce `dev`, `test` y `main` a desarrollo, staging y producción, elige el secreto de base de datos, obtiene una etiqueta de imagen explícita o usa `sha-<7 caracteres>`, despliega con Docker Compose, ejecuta `alembic upgrade head` y prueba `http://localhost:8000/api/health`. Si falla la comprobación y hay una versión previamente registrada, intenta volver a ella. Al completar correctamente actualiza la etiqueta Git `deployed/<ambiente>`.
- **`dast.yml`:** recibe una URL de una aplicación ya desplegada y, opcionalmente, una imagen Docker de ZAP. Solo acepta URL `http://` o `https://`; el análisis Baseline es pasivo. Guarda reportes y falla ante alertas de riesgo alto.
- Para ejecutar DAST después de un despliegue, el workflow consumidor debe ordenar ambos trabajos de forma que DAST dependa del despliegue y pasarle la URL resultante.

### 2.3 Ramas y ciclo de pull request

- **`create-branch.yml`:** recibe nombre, tipo (`feature` o `bugfix`), estados ClickUp permitidos y token de ClickUp. Opcionalmente crea un token de GitHub App; si no, usa `GITHUB_TOKEN`.
- **`pr-validate.yml`:** permite `feature/*` hacia `dev`; `bugfix/*` y `hotfix/*` hacia `test`, `main` o `release/*`; y `fix-merge/*` únicamente hacia `dev`. Una rama `fix-merge/*` pendiente bloquea las otras categorías. Para `bugfix/*`, compara el archivo `VERSION` de origen y destino. La ejecución se serializa por número de PR y la ejecución anterior puede cancelarse.
- **`pr-merged.yml`:** solo continúa si el evento indica que el PR fue fusionado. Al fusionar `fix-merge/*`, elimina esa rama. En destinos `main` o `release/*`, incrementa PATCH en `VERSION`, crea un commit y una etiqueta Git, y publica los cambios. Después intenta propagar el destino hacia `dev`; si hay conflicto, crea una rama `fix-merge/*` y abre un PR para resolverlo. Las ejecuciones se serializan por rama destino y no se cancelan entre sí.
- **`succesed-merge.yml`:** invoca Semgrep en el repositorio después de una fusión exitosa. Permite elegir las reglas (por defecto `p/default`) y pasar un token Semgrep opcional.
- **`create-release.yml`:** valida que la ejecución provenga de la rama por defecto y que exista más de un commit. Usa `VERSION` (si falta, inicia en `1.0.0`) para crear `release/<major>.<minor>` y la etiqueta `<major>.<minor>.<patch>`, publica una imagen GHCR con esa etiqueta y aumenta la versión de la rama por defecto al siguiente minor terminado en `.0`. No crea un objeto de “GitHub Release”; crea una rama y una etiqueta Git y publica una imagen.

### 2.4 Administración de despliegues de ramas

- **`update-env.yml`:** necesita la rama, un objeto JSON de variables y acceso SSH a la VPS. Actualiza el archivo `.env` de esa rama. `PORT` queda reservado para el proceso de despliegue y se ignora en esta actualización.
- **`teardown-branch.yml`:** necesita rama, ambiente, SSH y credenciales Hostinger. Intenta borrar el registro DNS y, en la VPS, detener el Compose, quitar configuración Nginx y certificado, y eliminar el directorio de esa rama. La rama remota de GitHub no se toca.

## 3. Acciones compuestas

Las acciones de esta sección son los pasos reutilizables llamados por los workflows. También pueden consumirse directamente, aunque las interfaces de workflow suelen aportar una orquestación más completa.

### Calidad, pruebas y seguridad

| Acción | Funcionamiento y herramientas |
| --- | --- |
| [`security-scan`](../.github/actions/security-scan/action.yml) | Descarga Gitleaks, analiza el historial Git con salida redactada y ejecuta Semgrep con el conjunto de reglas elegido. Un hallazgo bloqueante hace fallar el paso. |
| [`python-ci`](../.github/actions/python-ci/action.yml) | Configura Python con `actions/setup-python`, usa caché pip, instala `requirements.txt` y `pip-audit`, audita dependencias y ejecuta el comando de pruebas (por defecto `pytest -q`). |
| [`node-ci`](../.github/actions/node-ci/action.yml) | Configura Node.js con `actions/setup-node`, instala versiones bloqueadas con `npm ci`, ejecuta `npm audit`, resume vulnerabilidades y guarda el JSON como artefacto. Falla por HIGH/CRITICAL y ejecuta el script npm indicado. |
| [`java-ci`](../.github/actions/java-ci/action.yml) | Configura Temurin, valida Maven/Gradle, compila y prueba con `mvn verify` o `./gradlew build`, ejecuta OWASP Dependency-Check y guarda su reporte HTML como artefacto. |
| [`docker-build-scan-publish`](../.github/actions/docker-build-scan-publish/action.yml) | Calcula etiquetas SHA/rama/versión, construye una imagen local con Docker Buildx y caché de GitHub Actions, genera un reporte JSON completo de Trivy con resumen por severidad, lo sube como artefacto por 30 días y luego ejecuta el escaneo bloqueante; solo en `push` inicia sesión en GHCR y publica. La etiqueta SemVer no se sobrescribe si ya existe. |
| [`dast-zap`](../.github/actions/dast-zap/action.yml) | Ejecuta ZAP Baseline en Docker en modo pasivo, crea reportes JSON/HTML/Markdown, resume riesgos y falla si hay alertas de riesgo alto. Sube los reportes como artefacto incluso si el análisis falla. |

### Ramas, ClickUp y releases

| Acción | Funcionamiento y herramientas |
| --- | --- |
| [`fetch-clickup`](../.github/actions/fetch-clickup/action.yml) | Consulta la API v2 de ClickUp con `curl`, obtiene nombre y estado de una tarea y los expone como salidas. Acepta un ID de 7 a 9 caracteres, opcionalmente con prefijo `CU-` o `#`. |
| [`create-branch`](../.github/actions/create-branch/action.yml) | Valida el nombre, extrae el ID ClickUp, consulta su estado por API y lo compara sin distinguir mayúsculas/minúsculas con los estados permitidos. Para `feature` parte de la rama por defecto; para `bugfix` busca la etiqueta más reciente y exige su rama `release/<major>.<minor>`. Verifica que la rama no exista y la crea y publica con Git. |
| [`pr-validate`](../.github/actions/pr-validate/action.yml) | Aplica las reglas de compatibilidad de ramas descritas arriba, busca ramas `fix-merge/*` pendientes con Git y compara versiones `VERSION` para los bugfix. |
| [`pr-merged`](../.github/actions/pr-merged/action.yml) | Configura Git y el token, identifica el tipo de destino, elimina ramas `fix-merge/*`, incrementa PATCH en destinos de release y etiqueta/publica los cambios. Intenta fusionar el destino en `dev`; si hay conflicto, usa GitHub CLI (`gh`) para crear un PR de resolución. |
| [`success-merge`](../.github/actions/success-merge/action.yml) | Descarga el repositorio con historial completo (`actions/checkout`) y ejecuta Semgrep para análisis SAST en la rama fusionada. |
| [`create-release`](../.github/actions/create-release/action.yml) | Verifica rama y versión, genera rama/tag de release, publica ambos con Git y construye/publica la imagen con Docker Buildx y GHCR. Finalmente incrementa MINOR en la rama por defecto y publica el cambio. |

### Publicación y despliegue

| Acción | Funcionamiento y herramientas |
| --- | --- |
| [`ghcr-publish`](../.github/actions/ghcr-publish/action.yml) | Calcula o recibe el nombre de imagen, inicia sesión en GitHub Container Registry, comprueba si la etiqueta existe y construye y publica una imagen Docker. Devuelve el nombre completo y si la etiqueta existía. No la invocan los workflows actuales. |
| [`cd`](../.github/actions/cd/action.yml) | Acción de despliegue específica de Hub Auth: checkout del SHA, selección de ambiente y secretos, login GHCR, Docker Compose para base de datos y aplicación, migración Alembic, healthcheck, rollback mediante la etiqueta `deployed/<ambiente>` y registro del SHA desplegado. |
| [`deploy-branch`](../.github/actions/deploy-branch/action.yml) | Comprueba rama/ambiente e imagen, obtiene la versión y nombre DNS, administra el registro A vía API de Hostinger y se conecta por SSH. En la VPS instala dependencias del sistema si faltan, inicia sesión en GHCR, asigna o reutiliza un puerto, genera Compose y configuración Nginx, descarga la imagen, inicia el contenedor y solicita HTTPS mediante Certbot. |
| [`teardown-branch`](../.github/actions/teardown-branch/action.yml) | Calcula el subdominio, solicita a Hostinger eliminar su registro DNS y entra por SSH para detener/eliminar los recursos Compose, retirar el sitio Nginx y el certificado Certbot, y borrar solo el directorio de la rama. El borrado DNS y algunos pasos de limpieza son de mejor esfuerzo. |
| [`update-env`](../.github/actions/update-env/action.yml) | Se conecta por SSH, valida que `env-vars` sea JSON con `jq` y actualiza o añade pares clave/valor en el `.env` de la rama. Ignora `PORT`. |

## 4. Herramientas y servicios externos

| Herramienta o servicio | Uso en estos flujos |
| --- | --- |
| GitHub Actions y GitHub API | Ejecución en `ubuntu-latest`, checkout, tokens de GitHub App, permisos, artefactos, ramas y PR. |
| Git y GitHub CLI (`gh`) | Validación de historial, etiquetas, ramas, commits, propagación y PR automático de resolución de conflictos. |
| Gitleaks y Semgrep | Detección de secretos y análisis estático (SAST). |
| `pip-audit`, `npm audit`, OWASP Dependency-Check | Análisis de dependencias (SCA) según el lenguaje. |
| Docker, Docker Buildx y GHCR | Construcción, análisis, almacenamiento y despliegue de imágenes de contenedor. |
| Trivy | Análisis de vulnerabilidades de imágenes Docker. |
| OWASP ZAP Baseline | DAST pasivo de una aplicación desplegada; genera reportes para revisar. |
| ClickUp API v2 | Obtención del estado y nombre de tareas y validación antes de crear ramas. |
| SSH (`appleboy/ssh-action`) | Acceso remoto a la VPS para despliegue, actualización de variables y retirada de ramas. |
| Hostinger DNS API | Creación/actualización y eliminación de registros DNS. |
| Docker Compose, Nginx, Certbot y `systemctl` | Ejecución de aplicaciones, proxy inverso, HTTPS y recarga del servicio en la VPS. |
| Python, Node.js y Java Temurin | Configuración de runtimes en los runners para las pruebas de cada stack. |

## 5. Entradas, secretos y permisos que debe aportar el consumidor

- **CI:** indicar el stack y, cuando corresponda, directorio, versión del runtime, comando de prueba y nombre de imagen. Para publicar en GHCR, otorgar permiso `packages: write` al token o entregar un `ghcr_token` con ese permiso.
- **DAST:** suministrar la URL de una aplicación ya desplegada y autorizada para pruebas.
- **Ramas/PR/releases:** `GITHUB_TOKEN` o, opcionalmente, credenciales de una GitHub App con permisos adecuados para escribir ramas, etiquetas y PR. La creación de ramas también necesita `CLICKUP_API_TOKEN`.
- **VPS:** los flujos de despliegue/retirada/configuración necesitan IP, usuario y clave SSH. Despliegue y retirada requieren además credenciales de Hostinger; despliegue necesita token GHCR con lectura de la imagen.
- **CD de Hub Auth:** proporcionar el token GHCR, el secreto JWT y la contraseña de la base de datos correspondiente al ambiente (`dev`, `test` o `prod`).
- Los workflows declaran permisos a nivel de workflow o job. El repositorio consumidor también debe permitir que el `GITHUB_TOKEN` tenga los permisos requeridos por la operación; la credencial con la que se llama no eleva por sí sola permisos ausentes.

## 6. Consideraciones operativas

1. **Referencias centrales:** los consumidores y varias acciones apuntan a este repositorio con `@main`. Una actualización de `main` puede cambiar el comportamiento sin que el consumidor modifique su referencia. Para reproducibilidad, conviene fijar una etiqueta versionada o SHA aprobado.
2. **Recursos con estado:** despliegue, DNS, Nginx, certificados, archivos `.env`, ramas y etiquetas modifican sistemas persistentes. Para un teardown, el nombre de rama y el ambiente deben coincidir con el despliegue que se desea retirar.
3. **Etiquetas de imagen:** CI genera una etiqueta SHA y una de rama en `push`; para `main`/`release/*` puede generar además una etiqueta `VERSION`. El despliegue genérico de PROD selecciona la etiqueta de `VERSION`, por lo que el proyecto debe mantener ese archivo alineado con la imagen publicada.
4. **Distintos modelos de release:** `create-release` crea la línea `release/<major>.<minor>` y luego avanza el minor de desarrollo. `pr-merged` incrementa PATCH al fusionar hacia `main` o `release/*`. Son automatizaciones complementarias, pero tienen responsabilidades diferentes.
5. **Herramientas de seguridad:** los criterios no son iguales: `pip-audit` bloquea cualquier vulnerabilidad reportada, `npm audit` bloquea HIGH/CRITICAL, Java utiliza el umbral CVSS configurado y Trivy bloquea CRITICAL/HIGH con corrección disponible; las no corregibles se ignoran en el paso de bloqueo. ZAP bloquea alertas de riesgo alto.
6. **Artefactos de diagnóstico:** Node.js, Java, Trivy y ZAP suben reportes como artefactos para facilitar su consulta; deben revisarse en la ejecución de GitHub Actions.
7. **Alcance del teardown:** se elimina el recurso de despliegue de una rama en la VPS, pero no la rama de GitHub. La eliminación de DNS se trata como mejor esfuerzo.
8. **Acción no conectada a un workflow actual:** `ghcr-publish` está disponible para consumo, pero ninguno de los 13 workflows listados la llama actualmente.

## 7. Inventario

- **Workflows:** 13 en `.github/workflows/`.
- **Actions compuestas:** 17 en `.github/actions/`.
- **Familias funcionales:** CI/SCA, SAST/DAST, ramas y PR, releases, publicación de contenedores, CD de Hub Auth y operaciones de despliegue en VPS.
