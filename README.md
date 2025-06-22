# 📦 Guía de CI/CD Automático y versionado semántico

![GitLab pipeline](https://img.shields.io/badge/pipeline-gitlab-blue?logo=gitlab)
![Semantic Versioning](https://img.shields.io/badge/semver-2.0.0-green)

> **CHANGELOG\_VERSION.txt** es la fuente de verdad para cada despliegue: contiene la versión completa construida por el pipeline, huella de commit y metadatos del artefacto.

---

## 🧑‍💼 Introducción

**Sentinel Deploy** es la estrategia implementada en este repositorio para lograr despliegues automáticos, seguros y trazables mediante GitLab CI/CD. El ciclo completo `build → test → analysis → deploy` se ejecuta sin intervención humana para cambios menores y parches, garantizando agilidad. No obstante, cualquier cambio crítico (*breaking changes*) activa un mecanismo de control que requiere aprobación manual, gestionado mediante *Protected Tags*.

Esta arquitectura combina versionado semántico automatizado, control de integridad y transparencia total en cada release.

> ⚠️ **Salvaguardas recomendadas:** deshabilitar builds en tags innecesarios, definir claramente `BREAKING CHANGE:` en los commits o MRs, y habilitar notas informativas por release para mantener trazabilidad y control.

---

## 🧑‍💼 Tabla de contenidos

1. [Introducción](#introducción)
2. [Estrategia de versionado](#estrategia-de-versionado)
3. [Flujo de CI/CD](#flujo-de-cicd)
4. [Etapas del pipeline](#etapas-del-pipeline)
5. [¿Cómo se calcula la versión?](#cómo-se-calcula-la-versión)
6. [Protección de versiones Major](#protección-de-versiones-major)
7. [CHANGELOG\_VERSION.txt](#changelog_versiontxt)
8. [Ejemplos de commits y su impacto](#ejemplos-de-commits-y-su-impacto-en-el-versionado)
9. [Guía rápida de uso](#guía-rápida-de-uso)
10. [Contribución](#contribución)

---

## Estrategia de versionado

Esta estrategia sigue los principios de [Semantic Versioning 2.0.0](https://semver.org/lang/es/) y los adapta al entorno CI/CD moderno con automatización inteligente.

| Componente | Fuente                                                                               | Ejemplo |
| ---------- | ------------------------------------------------------------------------------------ | ------- |
| **MAJOR**  | `BREAKING CHANGE:` en MR – requiere tag protegido y validación manual                | `2`     |
| **MINOR**  | Commits tipo `feat:` que introducen nuevas funcionalidades sin romper compatibilidad | `5`     |
| **PATCH**  | Commits tipo `fix:` que corrigen errores sin cambiar interfaces públicas             | `7`     |
| **BUILD**  | `CI_PIPELINE_IID` que identifica de forma única el pipeline de construcción          | `123`   |

> Formato final de versión: **`v<MAJOR>.<MINOR>.<PATCH>.<BUILD>`** → `v2.5.7.123`

Esta versión se genera automáticamente mediante:

* `semantic-release`, herramienta que analiza commits y decide el tipo de incremento.
* Commit messages que siguen la convención de [Conventional Commits](https://www.conventionalcommits.org).

Asegura:

* Automatización total en cambios menores o correcciones.
* Supervisión humana en cambios estructurales.
* Reflejo de versión en `CHANGELOG_VERSION.txt` y cada release/tag.

---

## Flujo de CI/CD

### Descripción

El proceso se divide en dos fases:

1. **CI (rama)**:

   * Se ejecuta en cada commit o merge request.
   * Etapas: `lint`, `test`, `analysis`, `versioning`.
   * `semantic-release` analiza commits desde el último tag:

     * Si encuentra `feat:` o `fix:`, genera una nueva versión **minor** o **patch**, crea el tag y se inicia el pipeline **CD**.
     * Si detecta `BREAKING CHANGE:`, **no crea el tag** automáticamente.

       * Se activa un job condicional (`manual`) que deja en espera la ejecución.
       * Un usuario autorizado debe crear manualmente un **tag protegido** MAJOR.

2. **CD (tag)**:

   * Se activa al crearse un tag `vX.Y.Z.N`.
   * Etapas: `docker-build`, `docker-scan`, `deploy`.
   * Se ejecuta siempre:

     * Tags **minor/patch** creados automáticamente.
     * Tags **major** creados manualmente por el Release Manager.

### Representación secuencial

```
CI (en ramas)
├── verifyFiles
├── install-deps
├── lint_python & dockerfile-lint
├── test_api
├── sonarcloud-check & zap_dast
└── semantic-release
    ├── Minor/Patch: crea tag automático → inicia CD
    └── MAJOR: job manual → requiere tag protegido

CD (por tag)
├── build-docker
├── docker-security-scan & python-deps-scan
└── rollout-status
```

---

## Etapas del pipeline

| Stage          | Descripción breve                                      |
| -------------- | ------------------------------------------------------ |
| `verifyFiles`  | Verifica archivos requeridos en el repositorio         |
| `install-deps` | Instalación y caché de dependencias                    |
| `lint`         | `flake8` + `dockerfile-lint`                           |
| `test`         | `pytest` + reporte de cobertura                        |
| `analysis`     | `sonar-scanner` (SAST) + `nuclei` (DAST)               |
| `versioning`   | `semantic-release` genera tag / CHANGELOG\_VERSION.txt |
| `docker-build` | Construcción y push de la imagen                       |
| `docker-scan`  | Trivy sobre imagen y árbol de dependencias             |
| `deploy`       | `kubectl rollout` + (opcional) `argo-wait`             |

---

## 🧠 ¿Cómo se calcula la versión?

El stage `versioning` usa [`semantic-release`](https://github.com/semantic-release/semantic-release):

### Funciones:

1. Analiza commits desde el último tag.
2. Clasifica usando [Conventional Commits](https://www.conventionalcommits.org).
3. Decide el incremento:

   * `fix:` → PATCH
   * `feat:` → MINOR
   * `BREAKING CHANGE:` → MAJOR (requiere tag manual)
4. Genera nuevo tag si aplica y activa el pipeline CD.

### Ejemplo `.gitlab-ci.yml`

```yaml
versioning:
  stage: versioning
  image: node:18
  script:
    - npm install -g semantic-release \
        @semantic-release/changelog \
        @semantic-release/git \
        @semantic-release/gitlab
    - semantic-release --branches main --ci
  only:
    - main
```

**Salida esperada:**

```
🔍 Analyzing commit messages...
✔ Determined next version: 1.4.0 (minor)
✔ Created tag v1.4.0.123
```

O si hay cambio mayor:

```
🔍 Analyzing commit messages...
⚠ Detected BREAKING CHANGE – skipping tag creation
🚫 Requires manual tag protected by Release Manager
```

---

## Protección de versiones Major

* Solo el grupo **Release Managers** puede crear tags `v*.*.*.*` para MAJOR.
* Si `semantic-release` detecta `BREAKING CHANGE`, crea un job `manual` que espera intervención.
* El tag debe ser protegido y creado manualmente.

> ⚡️ **Ejemplo de commit estructural:**
>
> ```
> feat(auth): actualizar sistema de autenticación
>
> BREAKING CHANGE: se eliminó el endpoint /v1/login y se reemplazó por /v2/auth
> ```

---

## CHANGELOG\_VERSION.txt

Archivo generado durante `docker-build`:

```
App: barberia_calendario
Tag: v2.5.7.123
Commit: a1b2c3d
Branch: main
Fecha: 2025-06-22T17:04:18Z
Pipeline: https://gitlab.com/…/pipelines/1234
Autor: joneshys
```

---

## 📘 Ejemplos de commits y su impacto en el versionado

| Commit                                                                 | Resultado                      |
| ---------------------------------------------------------------------- | ------------------------------ |
| `feat(api): nuevo endpoint /users`                                     | MINOR                          |
| `fix(auth): error en login`                                            | PATCH                          |
| `refactor(core): reordenar lógica`                                     | No cambia versión              |
| `feat(core): nueva API` + `BREAKING CHANGE: elimina endpoint anterior` | MAJOR (requiere tag protegido) |



| Commit                                            | Tipo      | Resultado en versión semántica |
| ------------------------------------------------- | --------- | ------------------------------ |
| `feat(auth): agregar autenticación con JWT`       | **Minor** | `1.1.0`                        |
| `feat(api/user): endpoint para actualizar perfil` | **Minor** | `1.2.0`                        |
| `feat(cart): cálculo automático de descuentos`    | **Minor** | `1.3.0`                        |
| `feat(ui/header): nuevo botón de cerrar sesión`   | **Minor** | `1.4.0`                        |
| `feat(db): migración para historial de pagos`     | **Minor** | `1.5.0`                        |
| `fix: corregir error de fechas`                   | **Patch** | `1.0.1`                        |
| `docs: mejorar README`                            | *Otro*    | No cambia la versión           |
| `refactor: cambiar lógica de auth`                | *Otro*    | No cambia la versión           |
| `feat(auth): nueva API /v2`                       |           |                                |
| `BREAKING CHANGE: elimina /v1`                    | **MAJOR** | Requiere tag manual `2.0.0`    |

> 💡 Los `BREAKING CHANGE:` deben ir en el cuerpo del commit (después del mensaje principal) o en la descripción del *Merge Request* si estás usando squash.


---

## 🍪 Guía rápida de uso

1. Commitea usando Conventional Commits (`feat:`, `fix:`, etc).
2. Crea Merge Request → se ejecuta pipeline CI.
3. Si no hay `BREAKING CHANGE`, se genera y publica versión minor/patch.
4. El tag inicia pipeline CD (build, escaneo, despliegue).
5. Para cambios estructurales, incluye `BREAKING CHANGE:` en cuerpo del commit o descripción del MR. 

## Documentación adicional para los Tag´s
### 📚 Recursos adicionales
* [Git – Tag Documentation](https://git-scm.com/book/en/v2/Git-Basics-Tagging)
* [GitLab – Protected Tags](https://docs.gitlab.com/user/project/protected_tags/)
* [Semantic Release Docs](https://semantic-release.gitbook.io/semantic-release)
---

## Contribución

* Ejecuta `pre-commit install` para validar linting local.
* Consulta la [Guía de commits](docs/COMMITS.md).
* Propón mejoras vía MR; los bots de IA harán code review automáticamente.

---

> © 2025 Joneshys – Distribuido bajo licencia MIT.
