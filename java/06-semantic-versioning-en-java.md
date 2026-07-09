# Semantic Versioning en Java — Guia para Nova Platform

## Contexto

Esta guia documenta los hallazgos sobre como llevar **Semantic Versioning (SemVer)** en proyectos Java del meta-framework **Nova Platform** (alias: **Nova**), con foco en la adopcion de **Gradle 9.x** como build system (excepto BOM y Parent que permanecen en Maven por estandar de la industria).

El objetivo es replicar el flujo que npm/JS/TS tienen nativo (`npm version`, `npm publish`), pero aplicado a Java donde el versionado **no es nativo del build tool** y debe construirse como una capa adicional.

> **Alcance de este documento:** cubre exclusivamente los **15 repos Java** y los **3 repos multi-stack** (nova-devops, nova-bom, nova-infrastructure). Los 4 repos NestJS (`nova-nestjs-*`) se abordaran en un documento y roadmap separados.

---

## 0. Convencion de naming de repos (NOVA-SEMVER-31)

**Regla:** el nombre del repo refleja la **tecnologia objetivo** del artefacto, no el lenguaje. Esto permite que el meta-framework crezca para incluir implementaciones sobre otros frameworks (Quarkus, Micronaut, etc.) sin renombrar nada.

**Patron general:** `nova-<lenguaje>-<rol>` o `nova-<lenguaje>-<rol>-<framework>`

| Tipo de artefacto | Framework-coupled? | Patron | Ejemplo |
|---|---|---|---|
| **Lib pura** (framework-agnostic) | No | `nova-java-<rol>` | `nova-java-mask-utils` |
| **Starter / Extension** | Si (Spring Boot) | `nova-java-<rol>-spring-boot-<tipo>` | `nova-java-commons-spring-boot-starter` |
| **Gradle plugin** | Si (Spring Boot) | `nova-java-spring-boot-gradle-plugin` | `nova-java-spring-boot-gradle-plugin` |
| **Maven plugin / archetype / parent** | Si (Spring Boot) | `nova-java-spring-boot-<rol>` | `nova-java-spring-boot-parent` |
| **BOM** | No | `nova-bom` (sin sufijo de lenguaje) | `nova-bom` |
| **Instance / demo** | No (es instancia, no artefacto) | `nova-java-<rol>` | `nova-java-example` |
| **Infraestructura compartida** | No | `nova-<rol>` | `nova-devops`, `nova-infrastructure` |

**Reglas especificas:**

1. **Libs puras no llevan framework en el nombre** — son reutilizables en cualquier contexto. Si una lib se acopla a un framework especifico, se renombra a `nova-java-<rol>-<framework>-<tipo>`.
2. **Starters y extensiones SIEMPRE llevan el framework** — un starter de Quarkus se llamaria `nova-java-<rol>-quarkus-extension` (siguiendo la convencion de Quarkus), NO `nova-java-<rol>`.
3. **Meta-starters** (que agregan varios starters) llevan el framework: `nova-java-spring-boot-starter` es el meta-starter de Spring Boot.
4. **Plugin Gradle** lleva el framework objetivo, no el lenguaje del plugin: `nova-java-spring-boot-gradle-plugin` (es un plugin para proyectos Spring Boot).
5. **Archetype y parent** llevan el framework porque el parent POM de Spring Boot es especifico: `nova-java-spring-boot-parent`, `nova-java-spring-boot-archetype`.

**Namespace Maven** (no cambia con el rename del repo):

| Tipo | Namespace | Ejemplo |
|---|---|---|
| Libs | `pe.edu.nova.java.libs` | `pe.edu.nova.java.libs:nova-java-mask-utils` |
| Starters | `pe.edu.nova.java.starters` | `pe.edu.nova.java.starters:nova-java-commons-spring-boot-starter` |
| Plugin Gradle | `pe.edu.nova.java.spring-boot` (id) | id del plugin: `pe.edu.nova.java.spring-boot` |

**Regla de correspondencia:** la primera parte del repo name (despues de `nova-java-`) y el artifactId Maven coinciden. Ejemplos:

| Repo | ArtifactId Maven | GroupId |
|---|---|---|
| `nova-java-mask-utils` | `nova-java-mask-utils` | `pe.edu.nova.java.libs` |
| `nova-java-commons-spring-boot-starter` | `nova-java-commons-spring-boot-starter` | `pe.edu.nova.java.starters` |
| `nova-java-spring-boot-gradle-plugin` | `nova-java-spring-boot-gradle-plugin` | `pe.edu.nova.java.spring-boot` |

> **Justificacion de la correccion (2026-07-09):** la primera iteracion del rename removió `spring-boot` de TODOS los repos. Esto fue un error: los starters y plugins SI estan acoplados a Spring Boot y deben declararlo. Si manana se agrega `nova-java-quarkus-extension`, se aplicara la misma convencion (Quarkus en el nombre, no en el groupId). El rename es **por tecnologia objetivo**, no por simplificacion.

---

## 1. El problema conceptual

### En JS/TS

- `package.json` declara `"version": "2.3.1"`.
- `npm install` resuelve transitivamente respetando `^`, `~`, `>=`.
- `npm version patch` bumpea automaticamente.
- `npm publish` sube a npm registry.
- `standard-version` o `semantic-release` generan changelog desde commits.

**Todo esto esta integrado en el package manager.** Casi invisible para el developer.

### En Java (el caso actual)

- El `pom.xml` o `build.gradle.kts` declara `<version>2.3.1</version>` (Maven) o `version = "2.3.1"` (Gradle).
- `mvn dependency:resolve` o `gradle dependencies` resuelven transitivamente.
- **No existe un comando "bump" nativo** — el developer edita el archivo a mano.
- `mvn deploy` o `gradle publish` sube a Maven Central.
- El changelog hay que escribirlo a mano o usar plugins.

**La version es un string libre, no un objeto semver.** La buena noticia: el ecosistema de plugins para cubrir esta brecha es muy maduro y los proyectos grandes (Spring, Spring Boot, Hibernate, Apache Kafka) lo hacen rutinariamente.

---

## 2. Por que Java no trae semver nativo

Java fue diseñado para **librerias publicadas en repositorios centrales inmutables** (Maven Central) con coordenadas `groupId:artifactId:version`. Caracteristicas que afectan:

1. **Version inmutable:** una vez publicada `1.0.0`, no se republica. Esto obliga a CI riguroso antes de `publish`.
2. **Resolucion declarativa:** el `pom.xml` declara versiones, no las infiere de commits.
3. **Sin herramienta de "bump" oficial:** ni Maven ni Gradle exponen un comando estandar para eso.
4. **El commit log no alimenta al manifest:** la historia de cambios esta en Git, la version actual esta en el build file. No hay conexion automatica.

Por eso surge la necesidad de **tres capas adicionales** que JS/TS trae integradas:

| Capa | JS/TS equivalente | Java equivalente |
|---|---|---|
| Versionado en build | `package.json` | Plugin `net.nemerosa.versioning` |
| Automatizacion de release | `semantic-release` | `release-please` (Google) o `semantic-release` (orquestando Gradle) |
| Convencion de commits | Conventional Commits + `commitlint` | Conventional Commits + `commitlint` (mismo) |

---

## 3. Hosting de artefactos (registry)

Antes de hablar de semver hay que decidir **donde se publican los artefactos**. Hay 4 opciones principales, cada una con trade-offs:

### 3.1. GitHub Packages (`maven.pkg.github.com`)

**Estado actual:** Los reusable workflows `reusable-publish-{gradle,maven}.yml` ya estan configurados para publicar aqui.

| Aspecto | Detalle |
|---|---|
| URL del registry | `https://maven.pkg.github.com/<owner>/<repo>` |
| Autenticacion | `GITHUB_TOKEN` (automatico en Actions) |
| Visibilidad | **Configurable: publico o privado** (ver seccion 3.1.1) |
| Costo | Incluido en GitHub Free (con limites de storage/bandwidth) |
| Inmutabilidad | Si (no se reescriben versiones) |
| Firma GPG | Opcional pero recomendada |
| Compatible con BOM inter-repo | Si (cross-repo via `<owner>/<repo>` URL) |

**Ventajas:**
- Ya esta implementado en los workflows actuales.
- Autenticacion nativa con `GITHUB_TOKEN` (sin secretos adicionales).
- Buen rendimiento para quien ya consume desde GitHub.
- **Visibilidad configurable** por repo, sin cambiar la configuracion del workflow.

**Limitaciones:**
- El `groupId` se fuerza a que el primer segmento sea el nombre del owner de GitHub (restriccion de GitHub Packages para Maven).
- Para usuarios fuera de la organizacion, **los paquetes privados requieren autenticacion** aunque el repo sea publico.
- Limites de storage: 2 GB gratis, 50 GB con Pro.

#### 3.1.1. Visibilidad configurable: publica o privada

**Decision para Nova Platform:** la visibilidad del paquete en GitHub Packages es **parametrizable por repo** via una variable de configuracion en el repositorio (Settings → Variables → Actions). Por defecto es **publica**.

**Por que parametrizable:**

- **Repos personales de `ahincho`:** visibilidad **publica** (open source, descubrimiento facil).
- **Repos de la empresa:** visibilidad **publica** (la empresa tambien quiere que sus librerias internas sean descubribles; se asume que el `groupId` ya evita la exposicion accidental de info sensible).
- **Casos futuros:** algun modulo de pago o privado (ej: enterprise edition) podria necesitar visibilidad **privada**.

**Configuracion por repo (en GitHub):**

| Variable | Default | Valores | Efecto |
|---|---|---|---|
| `NOVA_PACKAGE_VISIBILITY` | `public` | `public` \| `private` | Define si el paquete se publica como publico o privado |

**Como se aplica en el workflow:**

```yaml
# reusable-publish-gradle.yml (o la composite action nova-publish-aggregator)
- name: Publish to GitHub Packages
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NOVA_PACKAGE_VISIBILITY: ${{ vars.NOVA_PACKAGE_VISIBILITY || 'public' }}
  run: ./gradlew publish -Pvisibility=$NOVA_PACKAGE_VISIBILITY
```

**Configuracion en `build.gradle.kts` para honrar la variable:**

```kotlin
val isPublic: Boolean = (project.findProperty("visibility") ?: "public") == "public"

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            // ... metadata ...
        }
    }

    repositories {
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/ahincho/${project.name}")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
    }
}

// Habilitar paquete publico o privado en GitHub Packages
// (la visibilidad real se controla via `gh repo edit --visibility` o settings del repo)
```

**Visibilidad del repositorio (no del paquete):**

Para que GitHub Packages acepte paquetes como **publicos**, el repositorio debe ser **publico**. Si el repo es privado, los paquetes siempre seran privados (no se puede override via package metadata).

| Visibilidad repo | `NOVA_PACKAGE_VISIBILITY` | Resultado del paquete |
|---|---|---|
| Public | `public` | Paquete publico (cualquiera lo descarga) |
| Public | `private` | **ERROR** (no se puede, repo publico implica paquete publico) |
| Private | `public` | **ERROR** (no se puede, repo privado implica paquete privado) |
| Private | `private` | Paquete privado (requiere auth para descargar) |

**Por eso el default recomendado para Nova Platform es repo publico + paquete publico.**

**Configurar visibilidad en GitHub UI:**

1. Repo → Settings → General →Danger Zone → "Change repository visibility" → Public/Private.
2. O via CLI: `gh repo edit ahincho/REPO --visibility public|private`.

**Migrar a privado en el futuro:**

Si en algun momento se quiere cambiar un repo de publico a privado, todos los paquetes previamente publicados se mantienen privados automaticamente. No requiere republicar.

### 3.2. Maven Central (`repo.maven.apache.org` / `repo1.maven.org`)

**Estado actual:** No implementado. Seria el destino "final" para librerias publicas consumidas por la comunidad Java.

| Aspecto | Detalle |
|---|---|
| URL del registry | `https://repo.maven.apache.org/maven2/` |
| Autenticacion | User Token de Sonatype OSSRH |
| Visibilidad | 100% publico |
| Costo | Gratis (con namespace propio) |
| Inmutabilidad | Si, estricta (politica de Sonatype) |
| Firma GPG | **Obligatoria** |
| Compatible con BOM inter-repo | Si (es el registry universal) |

**Ventajas:**
- Estandar de facto en Java. Cualquier herramienta, IDE o build tool lo conoce.
- Consumidores no necesitan autenticacion.
- Discovery via `search.maven.org`.

**Limitaciones:**
- Requiere solicitud de namespace en Sonatype (`issues.sonatype.org`).
- Requiere firma GPG obligatoria.
- Tiempos de propagacion: ~10-30 min entre deploy en Sonatype y disponibilidad en Central.
- Requiere metadata detallada en `pom.xml` (licencia, developers, scm, etc.).

### 3.3. Sonatype OSSRH (`s01.oss.sonatype.org`)

**Estado actual:** No implementado.

| Aspecto | Detalle |
|---|---|
| URL del registry | `https://s01.oss.sonatype.org/content/repositories/releases/` |
| Autenticacion | User Token de Sonatype |
| Funcion | Staging area para Maven Central |

**Cuando usarlo:** Es obligatorio como paso previo a Maven Central. Se deploya aqui y Sonatype valida y luego sincroniza a Central.

### 3.4. Sonatype Nexus Repository / JFrog Artifactory (self-hosted)

**Estado actual:** No implementado. Aplicable solo si la organizacion quiere control total on-premise.

| Aspecto | Detalle |
|---|---|
| Costo | Nexus OSS: gratis. Artifactory OSS: gratis. Enterprise: $$$ |
| Privacidad | 100% privado, on-premise o cloud propio |
| Politica | Configurable por el operador |
| Compatible con Maven + Gradle + npm | Si (multi-formato) |

**Cuando usarlo:** Si se necesita:
- Proxy/cache interno para artefactos publicos (ahorra bandwidth).
- Alojamiento de artefactos propietarios sin depender de terceros.
- Politica de retencion custom (ej: mantener `-SNAPSHOT` por 90 dias).

### 3.5. Recomendacion para Nova Platform

**Estrategia recomendada (multi-registry, gradual):**

| Etapa | Registry | Razon |
|---|---|---|
| **MVP / desarrollo** | GitHub Packages | Ya esta implementado, sin friccion |
| **Beta (cuando se quiera exponer)** | GitHub Packages + Nexus on-premise (opcional) | Decision segun necesidad de cache interno |
| **Produccion (publico)** | Maven Central (vía Sonatype OSSRH) | Discovery, sin autenticacion para consumidores, estandar |

Los reusable workflows deberian soportar **multi-registry** via un input `registry: github-packages | maven-central | nexus-custom`. Esto se logra con **maven-publish + profiles en Gradle**, y **distributionManagement + profiles en Maven**.

---

## 4. Herramientas verificadas (con fuentes)

### 4.1. `net.nemerosa.versioning` — Plugin Gradle oficial

- **Repositorio:** https://github.com/nemerosa/versioning
- **Stars:** 206, Forks: 40, Releases: 22 (ultima 4.0.1 en Abril 2026)
- **Publicacion:** Registrado en el Gradle Plugin Portal oficial.
- **Requiere:** Gradle 8.x+, JDK 17+.

**Que hace:** Deriva la version del proyecto a partir de la rama de Git y los tags existentes. Evita editar `version = "..."` a mano en cada build.

**Convención de ramas que exige:**

| Rama | Version generada | Caso de uso |
|---|---|---|
| `main` / `master` | la del ultimo tag | Produccion |
| `release/2.0` | `2.0.0`, `2.0.1`, ... (incrementa patch) | Patch incrementales |
| `feature/great` | `great-da50c50` (branch + short SHA) | Desarrollo |
| `release/2.0` con tag `2.0.5` previo | siguiente `2.0.6` | Bump automatico |

**Configuracion recomendada (Kotlin DSL):**

```kotlin
// build.gradle.kts
plugins {
    id("net.nemerosa.versioning") version "4.0.1"
}

versioning {
    releaseMode = "snapshot"      // rama release/* genera -SNAPSHOT
    displayMode = "snapshot"      // rama feature/* genera -SNAPSHOT
    dirty = { it }                // desactiva sufijo -dirty en dev local
    releaseBuild = false          // propio control de cual build es release
}

allprojects {
    version = versioning.info.full  // expone: full, display, base, build, etc.
}
```

**Tasks que expone:**

```bash
./gradlew versionDisplay
# [version] scm        = git
# [version] branch     = release/0.3
# [version] full       = release-0.3-da50c50
# [version] display    = 0.3.0
# [version] build      = da50c50

./gradlew versionFile    # genera build/version.properties
export $(cat build/version.properties | xargs)
```

### 4.2. `semantic-release` — Orquestador de release (JS/TS pero funciona con Java)

- **Repositorio:** https://github.com/semantic-release/semantic-release
- **Stars:** 23.9k, Releases: 451 (ultima v25.0.5 en Junio 2026).
- **Adopcion:** Estandar de facto en JS/TS, pero tiene plugins `@semantic-release/exec` que ejecutan cualquier comando shell — sirve para invocar `mvn deploy` o `gradle publish`.

**Como funciona:** Analiza los mensajes de commit (Conventional Commits), decide el bump (major/minor/patch), genera changelog, crea Git tag, crea GitHub Release, ejecuta publicacion.

**Plugin config para Java:**

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    ["@semantic-release/exec", {
      "prepareCmd": "npm run build && npm run test",
      "publishCmd": "mvn deploy -P release"
    }],
    "@semantic-release/git",
    "@semantic-release/github"
  ]
}
```

**Limitacion:** Introduce Node.js como dependencia **solo del lado del CI** (no en runtime). Algunos equipos lo ven como ventaja (tooling maduro), otros como lastre (mezcla de stacks).

### 4.3. `release-please` — GitHub Action de Google

- **Repositorio:** https://github.com/googleapis/release-please
- **Ventaja:** Declarativo, no añade runtime dependencies, funciona con cualquier lenguaje, multi-repo friendly.
- **Soporta:** `release-type: java | node | python | go | rust | ruby | php | elixir | dart | flutter | kmp`.

**Configuracion para Java:**

```yaml
# .github/workflows/release-please.yml
name: Release Please
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: java
          package-name: nova-java-mask-utils
```

**Como funciona:** Detecta Conventional Commits, abre un **PR de release** con bump + changelog, tú lo apruebas, al merge se crea el tag y la GitHub Release.

### 4.4. `Maven Release Plugin` — Plugin oficial Apache (Maven)

- Es el equivalente mas cercano a `npm publish` cuando usas Maven.
- Hace: bump de version, commit, tag, deploy a Maven Central.
- Configuracion en `pom.xml`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-release-plugin</artifactId>
    <version>3.0.1</version>
    <configuration>
        <tagNameFormat>v@{project.version}</tagNameFormat>
        <scmCommentPrefix>[release]</scmCommentPrefix>
    </configuration>
</plugin>
```

```bash
mvn release:clean release:prepare release:perform
```

**Limitacion:** Solo funciona con Maven, no con Gradle.

### 4.5. Casos de estudio (proyectos Java reales)

#### Spring Framework

- **Tags:** `v7.0.8`, `v6.2.19`, `v7.0.7` (formato `vMAJOR.MINOR.PATCH`).
- **Bot:** `@spring-builds` basado en Gradle Enterprise.
- **Workflow:** Conventional Commits + milestone abierto dispara build de release automatica.
- **Fuente:** https://github.com/spring-projects/spring-framework/releases

#### jqwik (proyecto Java con multi-modulo)

- **Archivos clave:** `signalPublishToCentralSonatype.sh` + `buildSrc/` con logica de versionado custom.
- **Patron:** Bash script que lee version de `gradle.properties`, ejecuta `gradle publish`, senala a Sonatype Central.
- **Fuente:** https://github.com/jqwik-team/jqwik

#### Spring Boot

- Tags `v3.4.0`, `v3.3.5` (uno por cada minor branch).
- Publish workflow dedicado (`publish.yml` en `.github/workflows/`).
- Sonatype Central + Maven Central con firma GPG.

---

## 5. Estado actual de los reusable workflows (`nova-devops`)

Inventario actual del repo `ahincho/nova-devops` (verificado al 2026-07-09: **20 archivos** en `.github/workflows/`, 3 composite actions en `.github/actions/`).

### 5.0. Inventario completo por categoria

#### 5.0.1. Reusable workflows base (8 — originales)

| # | Workflow | Build | Funcionalidad | Lineas |
|---|---|---|---|---|
| 1 | `reusable-build-maven.yml` | Maven | Build + tests + Checkstyle + JavaDoc + upload artifacts | 50 |
| 2 | `reusable-build-gradle.yml` | Gradle | Build + tests + Checkstyle + JavaDoc + upload artifacts | 50 |
| 3 | `reusable-sonarcloud-maven.yml` | Maven | JaCoCo + SonarCloud | 52 |
| 4 | `reusable-sonarcloud-gradle.yml` | Gradle | JaCoCo + SonarCloud | 52 |
| 5 | `reusable-version-bump-maven.yml` | Maven | Bump manual via etiquetas de PR (major/minor/patch) | 130 |
| 6 | `reusable-version-bump-gradle.yml` | Gradle | Bump manual via etiquetas de PR (major/minor/patch) | 123 |
| 7 | `reusable-publish-maven.yml` | Maven | Deploy a **GitHub Packages** | 111 |
| 8 | `reusable-publish-gradle.yml` | Gradle | Publish a **GitHub Packages** | 89 |

#### 5.0.2. Reusable workflows Sprint 1 (3 — NOVA-SEMVER-05-07)

| # | Workflow | Build | Funcionalidad | Lineas |
|---|---|---|---|---|
| 9 | `reusable-commitlint.yml` | n/a | Enforce Conventional Commits en PRs via `commitlint` | 88 |
| 10 | `reusable-release-please.yml` | n/a | Orquestar `release-please` multi-repo (genera PR de release) | 78 |
| 11 | `reusable-changelog.yml` | n/a | Auto-generar `CHANGELOG.md` desde commits | 125 |

#### 5.0.3. Reusable workflows Sprint 2 (6 — NOVA-SEMVER-09-11)

| # | Workflow | Build | Funcionalidad | Lineas |
|---|---|---|---|---|
| 12 | `reusable-publish-maven-multi-registry.yml` | Maven | Publicar a GitHub Packages + Sonatype simultaneamente | 116 |
| 13 | `reusable-publish-gradle-multi-registry.yml` | Gradle | Publicar a GitHub Packages + Sonatype simultaneamente | 107 |
| 14 | `reusable-publish-maven-maven-central.yml` | Maven | Deploy con firma GPG a Sonatype + Maven Central | 95 |
| 15 | `reusable-publish-gradle-maven-central.yml` | Gradle | Publish con firma GPG a Sonatype + Maven Central | 76 |
| 16 | `reusable-publish-maven-nexus.yml` | Maven | Deploy a Nexus on-premise | 106 |
| 17 | `reusable-publish-gradle-nexus.yml` | Gradle | Publish a Nexus on-premise | 85 |

#### 5.0.4. Reusable workflows Sprint 3 (1 — NOVA-SEMVER-13)

| # | Workflow | Build | Funcionalidad | Lineas |
|---|---|---|---|---|
| 18 | `reusable-release-publish.yml` | Gradle | Tag-triggered publish (lee version del tag `vX.Y.Z`, ejecuta `./gradlew publish`) | 109 |

#### 5.0.5. Plantillas de orquestacion (2 — NOVA-SEMVER-13)

Estos NO son reusable workflows sino plantillas invocables directamente desde cada repo:

| # | Workflow | Build | Funcionalidad | Lineas |
|---|---|---|---|---|
| 19 | `release-please.yml` | n/a | Wrapper que invoca `reusable-release-please.yml` | 17 |
| 20 | `publish-on-tag.yml` | Gradle | Wrapper que invoca `reusable-release-publish.yml` en push de tag | 15 |

> **Nota:** Los workflows #19 y #20 son el "punto de entrada" desde cada repo, y delegan a los reusable workflows #10 y #18 respectivamente. La logica vive en el reusable; el wrapper es solo un thin shim con `uses:` + `secrets:`.

#### 5.0.6. Workflows deprecados (⚠️)

| Workflow | Estado |
|---|---|
| `reusable-version-bump-maven.yml` | **Deprecado** post-NOVA-SEMVER-13. Mantenido en repo por compatibilidad pero NO debe usarse en proyectos nuevos. |
| `reusable-version-bump-gradle.yml` | **Deprecado** post-NOVA-SEMVER-13. Mismo motivo. |

> **Razon de la deprecacion:** ambos workflows tenian un bug critico — su regex de extraccion de version (`grep '^version' build.gradle.kts`) no parseaba el patron moderno `version = findProperty("version") as String` introducido en Sprint 0 (NOVA-SEMVER-04). El resultado era que el job `bump` siempre fallaba con "No jobs were run" o generaba un PR vacio. La migracion a `release-please` (NOVA-SEMVER-13) elimina esta clase de problemas porque la fuente de verdad de la version pasa a ser el tag `vX.Y.Z`, no un regex sobre el build file.

### 5.1. Análisis de gaps (actualizado 2026-07-09)

> **Cambio importante vs la version anterior de esta tabla:** muchos items marcados como "FALTA" ahora son ✅ OK gracias a Sprint 1, 2 y 3.

| Capacidad | Estado (2026-07-09) | Gap restante |
|---|---|---|
| **Build + tests** | ✅ OK | SpotBugs/PMD/Error Prone siguen opcionales |
| **Static analysis (Checkstyle)** | ✅ OK | Mismo |
| **Coverage (JaCoCo)** | Parcial | Genera reporte, pero no enforce minimo |
| **Quality gate (SonarCloud)** | ✅ OK | Depende de config externa del proyecto |
| **Versioning (bump manual)** | ⚠️ Deprecado | Reemplazado por `release-please` (NOVA-SEMVER-13) |
| **Versioning (automatico)** | ✅ OK (Sprint 3) | `release-please` + tag-triggered publish |
| **Publishing a GitHub Packages** | ✅ OK | Reusable workflows #7, #8, #12, #13 |
| **Publishing a Maven Central** | 🟡 Listo, sin ejecutar | Workflows #14, #15 implementados. Bloqueado por NOVA-SEMVER-29 (GPG key) |
| **Publishing a Nexus** | 🟡 Listo, sin ejecutar | Workflows #16, #17 implementados |
| **Multi-registry** | ✅ OK (Sprint 2) | Workflows #12, #13 cubren GitHub Packages + Sonatype |
| **Firma GPG** | 🟡 Preparada | Composite action `nova-setup-gpg` lista; clave NO generada (NOVA-SEMVER-29) |
| **Conventional Commits enforcement** | ✅ OK (Sprint 0 + 1) | `commitlint` local via `lefthook prepare` + `reusable-commitlint.yml` en CI |
| **Generacion de changelog auto** | ✅ OK (Sprint 1) | `reusable-changelog.yml` + `release-please` genera CHANGELOG.md |
| **GitHub Release auto** | ✅ OK (Sprint 3) | `release-please` crea GitHub Release al mergear PR de release |
| **Composite actions** | Parcial | 3 implementadas (#5.4 Sprint 1), 4 pendientes (NOVA-SEMVER-26 Sprint 5) |
| **Cache de Gradle distribuido** | Parcial | `setup-java` con cache local OK; remote via `gradle/actions` pendiente (NOVA-SEMVER-25) |
| **Build matrix (multi-Java)** | ⏳ Pendiente | Solo Java 25 (NOVA-SEMVER-19) |
| **SBOM (CycloneDX)** | ⏳ Pendiente | (NOVA-SEMVER-21) |
| **OWASP dependency check** | ⏳ Pendiente | (NOVA-SEMVER-20) |
| **Code coverage badge** | ⏳ Pendiente | (NOVA-SEMVER-22) |
| **Native image (GraalVM)** | ⏳ Pendiente | (fuera de sprints activos) |

### 5.2. Workflows reutilizables propuestos (a crear)

| # | Workflow propuesto | Build | Proposito | Prioridad |
|---|---|---|---|---|
| 1 | `reusable-commitlint.yml` | n/a | Enforce Conventional Commits en PRs | **P0** |
| 2 | `reusable-release-please.yml` | n/a | Orquestar release-please multi-repo | **P0** |
| 3 | `reusable-publish-maven-central.yml` | Maven | Deploy con firma GPG a Sonatype + Maven Central | **P0** |
| 4 | `reusable-publish-maven-nexus.yml` | Maven | Deploy a Nexus on-premise | P1 |
| 5 | `reusable-publish-gradle-maven-central.yml` | Gradle | Publish con firma GPG a Sonatype + Maven Central | **P0** |
| 6 | `reusable-publish-gradle-nexus.yml` | Gradle | Publish a Nexus on-premise | P1 |
| 7 | `reusable-publish-multi-registry.yml` | ambos | Publicar a GitHub Packages + Maven Central simultaneamente | **P0** |
| 8 | `reusable-build-matrix.yml` | ambos | Build contra Java 21 + 25 (o multiples versiones) | P1 |
| 9 | `reusable-native-image-gradle.yml` | Gradle | Build native con GraalVM | P2 |
| 10 | `reusable-owasp-check.yml` | ambos | Escanear CVEs con OWASP dependency-check | P1 |
| 11 | `reusable-sbom.yml` | ambos | Generar SBOM CycloneDX y publicarlo | P2 |
| 12 | `reusable-coverage-badge.yml` | ambos | Publicar coverage badge en README | P2 |
| 13 | `reusable-changelog.yml` | n/a | Generar CHANGELOG.md auto desde commits | P1 |
| 14 | `reusable-cleanup-snapshots.yml` | ambos | Limpiar `-SNAPSHOT` antiguos de Nexus/Maven Central | P3 |

### 5.3. Composite actions vs Reusable workflows — diferencia y cuando usar cada uno

Antes de listar las composite actions propuestas, es importante entender la diferencia:

| Aspecto | Reusable workflow | Composite action |
|---|---|---|
| **Que es** | Un workflow completo reutilizable | Un step (o grupo de steps) reutilizable |
| **Como se invoca** | `uses: org/repo/.github/workflows/x.yml@main` con `jobs:` | `uses: org/repo/action-name@v1` con `steps:` |
| **Contexto de ejecucion** | Corre como un job separado (runner propio) | Corre **dentro** del job que lo invoca (mismo runner, mismo filesystem) |
| **Puede tener `jobs` con `runs-on`** | Si | No (corre donde lo invoques) |
| **Puede llamar a otros workflows** | Si | No |
| **Mejor para** | Pipelines completos (build, test, publish) | Pasos compuestos (setup, install tools, sign artifacts) |
| **Output** | Puede exponer outputs entre jobs | Solo via `$GITHUB_OUTPUT` / `$GITHUB_ENV` |

**Regla practica:**

- **Reusable workflow** cuando encapsulas **un pipeline entero** (ej: "compilar y testear este repo Maven").
- **Composite action** cuando encapsulas **un setup o transformacion reusable** (ej: "importar GPG key desde secrets", "instalar Java 25 + Gradle cache").

### 5.4. Composite actions — estado actual y propuesto (actualizado 2026-07-09)

**Decision de diseno: se implementan 6 composite actions para encapsular logica reutilizable.** La septima (`nova-configure-gradle-cache`) fue **descartada** porque `gradle/actions/setup-gradle@v4` (action oficial de Gradle) cumple la misma funcion de forma mas simple y mantenida por la comunidad. Ver §5.4.1.

**Estado actual verificado (2026-07-09):**

- ✅ **6 implementadas** (3 Sprint 1 + 3 Sprint 5): todas creadas en `nova-devops/.github/actions/`.
- ❌ **1 descartada**: `nova-configure-gradle-cache` reemplazada por `gradle/actions/setup-gradle@v4` directo en workflows.

Las 6 actions viven en `ahincho/nova-devops/.github/actions/<nombre>/`:

| # | Action | Path | Proposito | Inputs | Outputs | Estado |
|---|---|---|---|---|---|---|
| 1 | `nova-setup-java` | `actions/nova-setup-java/` | Setup Java 25 + Gradle/Maven cache | `java-version`, `build-tool` | — | ✅ Sprint 1 |
| 2 | `nova-setup-node` | `actions/nova-setup-node/` | Setup Node.js 20 + npm | `node-version` | — | ✅ Sprint 1 (version simplificada, ver nota) |
| 3 | `nova-setup-gpg` | `actions/nova-setup-gpg/` | Importar GPG key desde secrets (preparado, no usado aun) | `gpg-signing-key-id`, `gpg-signing-key`, `gpg-signing-password`, `fail-on-missing` | `gpg-key-imported` | ✅ Sprint 1 |
| 4 | `nova-validate-build` | `actions/nova-validate-build/` | Valida Java version, secrets, gradle.properties, lefthook | `min-java-version`, `enforce-no-secrets` | `validation-result`, `java-version-detected` | ✅ Sprint 5 (NOVA-SEMVER-26) |
| 5 | `nova-gather-facts` | `actions/nova-gather-facts/` | Recolecta version, branch, commit SHA, is-snapshot, is-tag | `version-source`, `version-file`, `fallback-version` | `version`, `branch`, `commit-sha`, `commit-sha-short`, `is-snapshot`, `is-tag`, `build-number` | ✅ Sprint 5 (NOVA-SEMVER-26) |
| 6 | `nova-publish-aggregator` | `actions/nova-publish-aggregator/` | Switch interno segun `inputs.registry` para llamar al publish correcto (github-packages / maven-central / sonatype-staging / nexus) | `registry`, `build-tool`, `visibility`, `java-version`, `dry-run` | (via steps.visibility) | ✅ Sprint 5 (NOVA-SEMVER-26) |

> **Nota sobre `nova-setup-node` (Sprint 1):** la version implementada NO instala `commitlint` ni crea `commitlint.config.js`. Solo configura Node.js + npm. La razon: ahora la instalacion de dependencias npm + lefthook + commitlint se hace **localmente** en el developer via `npm install` (que dispara `prepare: lefthook install`). En CI ya no es necesario instalar commitlint porque `lefthook` no se usa en runners de Actions (los workflows validan con `reusable-commitlint.yml` que corre `npx commitlint` directamente).

#### 5.4.1. ¿Por que `nova-configure-gradle-cache` NO se implementa?

La propuesta original era crear una composite action que envuelva `gradle/actions/setup-gradle@v4` con un `README.md` consistente y defaults especificos de Nova. Sin embargo, despues de evaluarlo, decidimos **NO crear la composite** porque:

1. **Es una abstraccion innecesaria**: la action oficial `gradle/actions/setup-gradle@v4` ya tiene defaults sensatos (`cache-read-only: true` para PRs, `false` para push).
2. **Mantenimiento duplicado**: cualquier cambio en la action oficial requeriria actualizar nuestra composite.
3. **El doc se vuelve obsoleto mas rapido**: una composite con un unico `uses:` interno es un wrapper fragil.

**En su lugar**, el reusable `reusable-build-gradle.yml` invoca `gradle/actions/setup-gradle@v4` directamente:

```yaml
- name: Setup Gradle with GitHub Actions Cache
  uses: gradle/actions/setup-gradle@v4
  with:
    cache-read-only: ${{ github.event_name != 'push' }}
```

Si en el futuro necesitamos agregar logica Nova-especifica (por ejemplo, validar que `gradle.properties` tenga `org.gradle.caching=true`), ahi SI tiene sentido crear la composite.

**Estructura de una composite action en `nova-devops`:**

```
nova-devops/.github/actions/nova-setup-java/
  action.yml              # manifest obligatorio
  README.md               # documentacion de inputs/outputs
```

### 5.5. Implementacion completa de las 7 composite actions

#### 5.5.1. `nova-setup-java`

```yaml
# ahincho/nova-devops/.github/actions/nova-setup-java/action.yml
name: 'Setup Nova Java Environment'
description: 'Setup Java with Gradle/Maven cache following Nova conventions'
inputs:
  java-version:
    description: 'Java version to use'
    required: false
    default: '25'
  build-tool:
    description: 'Build tool (maven | gradle)'
    required: false
    default: 'gradle'
  distribution:
    description: 'JDK distribution'
    required: false
    default: 'temurin'
runs:
  using: 'composite'
  steps:
    - name: Setup Java ${{ inputs.java-version }}
      uses: actions/setup-java@v4
      with:
        distribution: ${{ inputs.distribution }}
        java-version: ${{ inputs.java-version }}
        cache: ${{ inputs.build-tool }}

    - name: Validate build file exists
      shell: bash
      run: |
        if [ ! -f "gradle.properties" ] && [ ! -f "pom.xml" ]; then
          echo "::error::No build file found. Nova requires gradle.properties or pom.xml"
          exit 1
        fi

    - name: Setup Gradle (if gradle)
      if: inputs.build-tool == 'gradle'
      uses: gradle/actions/setup-gradle@v4
      with:
        cache-read-only: ${{ github.event_name != 'push' }}
```

**Uso en un workflow:**

```yaml
- uses: ahincho/nova-devops/.github/actions/nova-setup-java@v1
  with:
    java-version: '25'
    build-tool: gradle
```

#### 5.5.2. `nova-setup-node`

```yaml
# ahincho/nova-devops/.github/actions/nova-setup-node/action.yml
name: 'Setup Nova Node Environment'
description: 'Setup Node.js with npm and commitlint following Nova conventions'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  package-manager:
    description: 'Package manager (npm | pnpm | yarn)'
    required: false
    default: 'npm'
runs:
  using: 'composite'
  steps:
    - name: Setup Node.js ${{ inputs.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache node modules
      uses: actions/cache@v4
      with:
        path: |
          node_modules
          .npm
        key: ${{ runner.os }}-${{ inputs.package-manager }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-${{ inputs.package-manager }}-

    - name: Install commitlint
      shell: bash
      run: |
        if [ -f "package.json" ]; then
          npm install --save-dev @commitlint/cli @commitlint/config-conventional
          echo "module.exports = { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js
        fi
```

#### 5.5.3. `nova-setup-gpg` (preparado pero no usado aun)

> **Nota tecnica:** las composite actions NO tienen acceso al contexto `secrets.*` de GitHub Actions.
> Los secrets deben pasarse como `inputs` desde el workflow que invoca la action.

```yaml
# ahincho/nova-devops/.github/actions/nova-setup-gpg/action.yml
name: 'Setup Nova GPG Signing'
description: 'Import GPG key from inputs for Maven Central publishing (preparado, firma aun no generada)'
inputs:
  gpg-signing-key-id:
    description: 'GPG key ID (fingerprint)'
    required: false
    default: ''
  gpg-signing-key:
    description: 'GPG private key (ASCII-armored, passed from workflow secrets)'
    required: false
    default: ''
  gpg-signing-password:
    description: 'GPG passphrase (if used)'
    required: false
    default: ''
  fail-on-missing:
    description: 'Fail if GPG inputs are not provided'
    required: false
    default: 'false'
outputs:
  gpg-key-imported:
    description: 'Whether the GPG key was successfully imported'
    value: ${{ steps.import.outputs.imported }}
runs:
  using: 'composite'
  steps:
    - name: Check GPG inputs
      id: check
      shell: bash
      run: |
        if [ -z "${{ inputs.gpg-signing-key-id }}" ] || [ -z "${{ inputs.gpg-signing-key }}" ]; then
          echo "::notice::GPG inputs not provided. Skipping GPG setup."
          echo "skipping=true" >> "$GITHUB_OUTPUT"
        else
          echo "skipping=false" >> "$GITHUB_OUTPUT"
        fi

    - name: Import GPG key
      id: import
      if: steps.check.outputs.skipping == 'false'
      shell: bash
      run: |
        echo "${{ inputs.gpg-signing-key }}" | gpg --import --batch --yes
        gpg --list-secret-keys --keyid-format=long
        echo "imported=true" >> "$GITHUB_OUTPUT"

    - name: Skip notice
      if: steps.check.outputs.skipping == 'true'
      id: skip
      shell: bash
      run: |
        echo "imported=false" >> "$GITHUB_OUTPUT"

    - name: Fail if required
      if: inputs.fail-on-missing == 'true' && steps.check.outputs.skipping == 'true'
      shell: bash
      run: |
        echo "::error::GPG inputs are required but not provided. See docs/06-semantic-versioning.md section 10.3"
        exit 1
```

**Uso desde un reusable workflow (que SI tiene acceso a `secrets`):**

```yaml
# En un reusable workflow (ej: reusable-publish-maven-central.yml)
steps:
  - uses: ahincho/nova-devops/.github/actions/nova-setup-gpg@v1
    with:
      gpg-signing-key-id: ${{ secrets.GPG_SIGNING_KEY_ID }}
      gpg-signing-key: ${{ secrets.GPG_SIGNING_KEY }}
      gpg-signing-password: ${{ secrets.GPG_SIGNING_PASSWORD }}
      fail-on-missing: 'true'  # obligatorio para Maven Central
```

#### 5.5.4. `nova-gather-facts`

```yaml
# ahincho/nova-devops/.github/actions/nova-gather-facts/action.yml
name: 'Gather Nova Build Facts'
description: 'Collect version, branch, commit SHA and expose as outputs'
inputs:
  version-source:
    description: 'Source of version (file | gradle-properties | env | package-json)'
    required: false
    default: 'gradle-properties'
  version-file:
    description: 'Path to version file (when source is file)'
    required: false
    default: 'gradle.properties'
outputs:
  version:
    description: 'Project version'
    value: ${{ steps.collect.outputs.version }}
  branch:
    description: 'Git branch'
    value: ${{ steps.collect.outputs.branch }}
  commit-sha:
    description: 'Full commit SHA'
    value: ${{ steps.collect.outputs.commit-sha }}
  is-snapshot:
    description: 'Whether the version is a snapshot'
    value: ${{ steps.collect.outputs.is-snapshot }}
runs:
  using: 'composite'
  steps:
    - name: Collect facts
      id: collect
      shell: bash
      run: |
        # Branch
        BRANCH="${GITHUB_REF_NAME:-$(git rev-parse --abbrev-ref HEAD)}"
        echo "branch=${BRANCH}" >> "$GITHUB_OUTPUT"

        # Commit SHA
        SHA=$(git rev-parse HEAD)
        echo "commit-sha=${SHA}" >> "$GITHUB_OUTPUT"

        # Version
        case "${{ inputs.version-source }}" in
          file|gradle-properties)
            if [ -f "gradle.properties" ]; then
              VERSION=$(grep '^version' gradle.properties | sed 's/version *= *//' | tr -d '"' | head -1)
            fi
            ;;
          env)
            VERSION="${NOVA_VERSION:-0.0.0}"
            ;;
          package-json)
            if [ -f "package.json" ]; then
              VERSION=$(node -p "require('./package.json').version" 2>/dev/null || echo "0.0.0")
            fi
            ;;
          *)
            VERSION="0.0.0"
            ;;
        esac
        VERSION="${VERSION:-0.0.0}"
        echo "version=${VERSION}" >> "$GITHUB_OUTPUT"

        # Is snapshot?
        if [[ "$VERSION" == *"-SNAPSHOT"* ]] || [[ "$VERSION" == *"-RC"* ]] || [[ "$VERSION" == *"-BETA"* ]]; then
          echo "is-snapshot=true" >> "$GITHUB_OUTPUT"
        else
          echo "is-snapshot=false" >> "$GITHUB_OUTPUT"
        fi

        echo "Gathered: version=${VERSION}, branch=${BRANCH}, commit=${SHA:0:8}"
```

#### 5.5.5. `nova-publish-aggregator`

```yaml
# ahincho/nova-devops/.github/actions/nova-publish-aggregator/action.yml
name: 'Nova Publish Aggregator'
description: 'Dispatch publish to the correct registry based on inputs'
inputs:
  registry:
    description: 'Target registry (github-packages | maven-central | nexus | sonatype-staging)'
    required: true
  visibility:
    description: 'Package visibility (public | private)'
    required: false
    default: 'public'
  build-tool:
    description: 'Build tool (maven | gradle)'
    required: true
  java-version:
    description: 'Java version'
    required: false
    default: '25'
  dry-run:
    description: 'If true, only print commands without executing'
    required: false
    default: 'false'
runs:
  using: 'composite'
  steps:
    - name: Dispatch to GitHub Packages
      if: inputs.registry == 'github-packages'
      uses: ahincho/nova-devops/.github/actions/nova-publish-github-packages@v1
      with:
        visibility: ${{ inputs.visibility }}
        build-tool: ${{ inputs.build-tool }}
        java-version: ${{ inputs.java-version }}
        dry-run: ${{ inputs.dry-run }}

    - name: Dispatch to Maven Central
      if: inputs.registry == 'maven-central'
      uses: ahincho/nova-devops/.github/actions/nova-publish-maven-central@v1
      with:
        build-tool: ${{ inputs.build-tool }}
        java-version: ${{ inputs.java-version }}
        dry-run: ${{ inputs.dry-run }}

    - name: Dispatch to Nexus
      if: inputs.registry == 'nexus'
      uses: ahincho/nova-devops/.github/actions/nova-publish-nexus@v1
      with:
        build-tool: ${{ inputs.build-tool }}
        java-version: ${{ inputs.java-version }}
        dry-run: ${{ inputs.dry-run }}
```

#### 5.5.6. `nova-configure-gradle-cache`

```yaml
# ahincho/nova-devops/.github/actions/nova-configure-gradle-cache/action.yml
name: 'Configure Nova Gradle Build Cache'
description: 'Enable GitHub Actions Build Cache for Gradle (local + remote via gradle/actions)'
inputs:
  cache-read-only:
    description: 'If true, only read from cache (do not write)'
    required: false
    default: 'false'
  gradle-home-cache-cleanup:
    description: 'Clean up old Gradle home cache entries'
    required: false
    default: 'true'
  cache-disabled:
    description: 'Disable the cache entirely'
    required: false
    default: 'false'
runs:
  using: 'composite'
  steps:
    - name: Setup Gradle with GitHub Actions Cache
      uses: gradle/actions/setup-gradle@v4
      with:
        cache-read-only: ${{ inputs.cache-read-only || (github.event_name != 'push') }}
        gradle-home-cache-cleanup: ${{ inputs.gradle-home-cache-cleanup }}
        cache-disabled: ${{ inputs.cache-disabled }}

    - name: Validate Build Cache config in gradle.properties
      shell: bash
      run: |
        # Valida que gradle.properties tenga las propiedades de cache.
        # NO las agrega en runtime (cambios efimeros no persisten).
        # Si faltan, emite warning para que el developer las agregue al repo.
        MISSING=0
        if [ -f "gradle.properties" ]; then
          if ! grep -q "^org.gradle.caching=true" gradle.properties; then
            echo "::warning::gradle.properties missing 'org.gradle.caching=true'. Add it to your repo (NOVA-SEMVER-23)."
            MISSING=1
          fi
          if ! grep -q "^org.gradle.configuration-cache=true" gradle.properties; then
            echo "::warning::gradle.properties missing 'org.gradle.configuration-cache=true'. Add it to your repo (NOVA-SEMVER-24)."
            MISSING=1
          fi
          if [ $MISSING -eq 0 ]; then
            echo "Build Cache config OK: caching and configuration-cache enabled."
          fi
        else
          echo "::warning::No gradle.properties found. Create one with cache properties (NOVA-SEMVER-00a)."
        fi
```

#### 5.5.7. `nova-validate-build`

```yaml
# ahincho/nova-devops/.github/actions/nova-validate-build/action.yml
name: 'Validate Nova Build Prerequisites'
description: 'Verify build prerequisites (Java version, no secrets, valid package files)'
inputs:
  min-java-version:
    description: 'Minimum required Java version'
    required: false
    default: '25'
outputs:
  validation-result:
    description: 'PASS or FAIL'
    value: ${{ steps.validate.outputs.result }}
runs:
  using: 'composite'
  steps:
    - name: Check Java version
      shell: bash
      run: |
        JAVA_VER=$(java -version 2>&1 | head -1 | awk -F '"' '{print $2}')
        echo "Detected Java version: ${JAVA_VER}"

    - name: Check for committed secrets
      shell: bash
      run: |
        # Buscar patrones comunes de secrets en archivos tracked
        PATTERNS=(
          'AKIA[0-9A-Z]{16}'                    # AWS access key
          '-----BEGIN RSA PRIVATE KEY-----'     # RSA private key
          '-----BEGIN OPENSSH PRIVATE KEY-----' # SSH private key
          'glpat-[A-Za-z0-9_-]{20,}'            # GitLab PAT
          'ghp_[A-Za-z0-9]{36}'                 # GitHub PAT (classic)
          'github_pat_[A-Za-z0-9_]{82}'         # GitHub PAT (fine-grained)
          'xox[abprs]-[A-Za-z0-9-]+'            # Slack tokens
        )

        FOUND=0
        for pattern in "${PATTERNS[@]}"; do
          if git grep -E "$pattern" -- ':(exclude)*.md' ':(exclude)docs/*' 2>/dev/null; then
            echo "::error::Found potential secret matching pattern: $pattern"
            FOUND=1
          fi
        done
        if [ $FOUND -eq 1 ]; then
          echo "::error::Secrets detected in tracked files. Aborting."
          exit 1
        fi

    - name: Validate package metadata
      shell: bash
      run: |
        if [ -f "gradle.properties" ]; then
          if ! grep -q "^group" gradle.properties && ! grep -q "^groupId" build.gradle* 2>/dev/null; then
            echo "::warning::No 'group' defined in gradle.properties or 'groupId' in build.gradle*"
          fi
        fi

    - name: Set result
      id: validate
      shell: bash
      run: |
        echo "result=PASS" >> "$GITHUB_OUTPUT"
```

### 5.6. Versionado y publicacion de las composite actions

Las composite actions se versionan siguiendo el mismo patron que las librerias:

- **Tag de version:** `v1.0.0`, `v1.1.0`, etc.
- **Branch principal:** `main` (para `uses: ...@main` en desarrollo).
- **Pin en workflows:** preferir `@v1` o `@v1.2.0` en vez de `@main` en produccion.
- **Conventional Commits:** cada cambio genera un PR que se mergea con `feat:`, `fix:`, etc.
- **release-please:** tambien automatiza el versionado de las actions en `nova-devops`.

```yaml
# Pin estable (recomendado para produccion)
- uses: ahincho/nova-devops/.github/actions/nova-setup-java@v1

# Pin exacto (maxima estabilidad)
- uses: ahincho/nova-devops/.github/actions/nova-setup-java@v1.2.3

# Branch main (solo para desarrollo/testing)
- uses: ahincho/nova-devops/.github/actions/nova-setup-java@main
```

---

## 6. Las 3 estrategias que funcionan en Java

### Estrategia A — Gradle + `net.nemerosa.versioning`

**Mas cercana a `npm version` + `lerna publish`.** El plugin deriva la version de la rama.

```kotlin
plugins {
    id("net.nemerosa.versioning") version "4.0.1"
}

versioning {
    releaseMode = "snapshot"
    displayMode = "snapshot"
    dirty = { it }
}

allprojects {
    version = versioning.info.full
}
```

**Pros:** Plugin oficial, maduro, sin dependencias externas, configurable.
**Contras:** No decide automaticamente major/minor/patch basandose en commits (eso lo hace `semantic-release`, no existe equivalente puro en Gradle).

### Estrategia B — `semantic-release` orquestando Gradle

**La mas automatizada.** `semantic-release` (Node.js) corre en CI, analiza commits, decide el bump, genera changelog, dispara `gradle publish`.

```yaml
# .github/workflows/release.yml
- uses: actions/setup-node@v4
- uses: actions/setup-java@v4
  with: { distribution: 'temurin', java-version: '25' }
- run: npm ci
- run: npx semantic-release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Pros:** Totalmente automatico, changelog auto-generado, GitHub Release auto.
**Contras:** Tooling mixto (Node.js + Gradle). Requiere disciplina en Conventional Commits.

### Estrategia C — `release-please` de Google (multi-repo)

**La mas pragmática.** GitHub Action declarativa, sin runtime dependencies.

```yaml
- uses: googleapis/release-please-action@v4
  with:
    release-type: java
    package-name: nova-java-mask-utils
```

**Pros:** Declarativo, no mezcla stacks, multi-repo friendly, PR de aprobacion humana.
**Contras:** No es full-automatic (requiere aprobar PR de release).

---

## 7. Convencion de commits (requisito previo)

Cualquiera de las 3 estrategias requiere **Conventional Commits** como base. Estandar en https://www.conventionalcommits.org/.

### Formato

```
<type>(<scope>): <description>
[body opcional]
[footer opcional]
```

### Tipos principales

| Tipo | Bump | Ejemplo |
|---|---|---|
| `feat` | minor | `feat(mask-utils): anadir estrategia de tarjetas de credito peruanas` |
| `fix` | patch | `fix(date-utils): corregir bug en formato relativo con DST` |
| `perf` | minor | `perf(api-standard): reducir overhead de ApiResponse wrapping` |
| `refactor` | patch (a veces none) | `refactor(observability): simplificar GoldenSignalsFilter` |
| `docs` | none | `docs(readme): actualizar instrucciones de instalacion` |
| `test` | none | `test(mask-utils): anadir property test para IbanMaskStrategy` |
| `chore` | none | `chore(deps): bump spring-boot a 4.0.6` |

### Breaking change

Usar `!` despues del scope o `BREAKING CHANGE:` en el footer → bump **major**.

```
feat(api-standard)!: cambiar firma de ApiResponse builder

BREAKING CHANGE: el metodo ApiResponse.ok(data) ahora requiere un Metadata obligatorio.
```

### Enforcement con `commitlint`

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo "module.exports = { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js
```

```yaml
# .github/workflows/commitlint.yml
- run: npx commitlint --from=${{ github.event.pull_request.base.sha }} --to=${{ github.event.pull_request.head.sha }} --verbose
```

### Hook pre-commit con `lefthook` (multi-lenguaje)

```yaml
# lefthook.yml
commit-msg:
  commands:
    commitlint:
      run: npx commitlint --edit {1}
```

---

## 8. Propuesta concreta para Nova Platform

### 8.1. Stack de versionado

| Capa | Herramienta | Justificacion |
|---|---|---|
| **Convención de commits** | Conventional Commits + `commitlint` + `lefthook` | Estandar universal, base para todas las demas herramientas |
| **Versionado en build** | `net.nemerosa.versioning` 4.0.1 (Gradle) | Deriva version de la rama, evita editar archivos a mano |
| **Release automation** | `release-please` de Google | Declarativo, no mezcla stacks, soporta multi-repo, PR de aprobacion |
| **Publicación MVP** | `maven-publish` plugin a GitHub Packages (actual) | Ya implementado, sin friccion |
| **Publicación produccion** | `maven-publish` plugin + firma GPG a Sonatype/Maven Central | Estandar para librerias publicas consumidas por la comunidad |
| **Publicación on-premise** (opcional) | `maven-publish` plugin a Nexus/Artifactory | Para organizaciones con cache interno |
| **Librerias puras (Nivel 1)** | `gradle.properties` con `version=0.x.0` | Siguen semver estricto, se actualizan via PR de release |
| **Starters (Nivel 2)** | `<parent.version>` alineado al BOM | Heredan del BOM, no se versionan independientemente |
| **BOM** | `0.x.0` centralizado | Al bump del BOM, se bump-an todos los starters coordinadamente |

### 8.2. Estrategia de branching (Git Flow adaptado)

```
main
  ├── release/0.1      (rama de estabilizacion)
  ├── release/0.2      (siguiente minor)
  ├── feat/ddd-utils   (feature en desarrollo)
  ├── fix/mask-cc-pe   (bug fix)
  └── chore/ci-bump    (mejora de tooling)
```

**Reglas:**

| Tipo de cambio | Rama destino | Bump resultante |
|---|---|---|
| Breaking change en API publica | `release/X+1.0` (nuevo major) | major |
| Nueva funcionalidad backward-compatible | `release/X.Y+1` (nuevo minor) | minor |
| Bug fix backward-compatible | `release/X.Y` actual | patch |
| Cambios internos sin impacto | cualquier rama | no bump |

### 8.3. Versionado por nivel del meta-framework

| Nivel | Componente | Como se versiona |
|---|---|---|
| 1 (libs puras) | `nova-java-mask-utils`, `nova-java-date-utils`, etc. | Independiente. Cada una sigue semver estricto. Se bump-an en su propio PR de release. |
| 2 (starters) | `nova-java-commons-spring-boot-starter`, etc. | Heredan del BOM. Bump coordinado cuando cambia el BOM. |
| 3 (meta-starter) | `nova-java-spring-boot-starter` | Versionado junto al BOM. |
| 4 (BOM) | `nova-bom` | **Coordinador.** `0.x.0` indica alineacion de todo el stack. |
| 4 (Parent) | `nova-java-spring-boot-parent` | Versionado junto al BOM. |
| 5 (Archetype) | `nova-java-spring-boot-archetype` | Independiente (es scaffolding, no API). |
| 5 (Gradle Plugin) | `nova-java-spring-boot-gradle-plugin` | Independiente. |
| Multi-stack | `nova-devops`, `nova-infrastructure` | Independiente. |

### 8.4. Configuracion concreta por modulo

> **Nota importante sobre los ejemplos:** los bloques de codigo a continuacion muestran la configuracion **objetivo** (con `pe.edu.nova.java.libs`, `net.nemerosa.versioning`, etc.). El estado actual de los repos es diferente:
> - Los repos ya usan `groupId = "pe.edu.nova.java.*"` (migrado en NOVA-SEMVER-00b). ✅
> - Todos los repos Gradle tienen `gradle.properties` con `version` y `group`. ✅
> - Todos los repos Gradle tienen `net.nemerosa.versioning` 4.0.1. ✅ (NOVA-SEMVER-03)
> - `nova-java-mask-utils`, `nova-java-observability-utils` y `nova-java-spring-boot-starter` fueron **migrados de Maven a Gradle** para cumplir la convencion. ✅
> - Solo **3 repos permanecen en Maven** por estandar de la industria: `nova-bom` (BOM), `nova-java-spring-boot-parent` (Parent POM), `nova-java-spring-boot-archetype` (Maven archetype).

#### Para librerias puras (Nivel 1) — ejemplo: `build.gradle.kts`

```kotlin
plugins {
    java
    id("net.nemerosa.versioning") version "4.0.1"
    `maven-publish`
}

group = "pe.edu.nova.java.libs"
version = versioning.info.display   // ej: "0.1.0" o "0.2.0-SNAPSHOT"

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
}

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            groupId = "pe.edu.nova.java.libs"
            artifactId = "nova-java-mask-utils"

            pom {
                name.set("Nova Java Spring Boot Mask Utils")
                description.set("Libreria pura Java de enmascaramiento de datos sensibles")
                url.set("https://github.com/ahincho/nova-java-mask-utils")
                licenses {
                    license {
                        name.set("Apache-2.0")
                        url.set("https://www.apache.org/licenses/LICENSE-2.0")
                    }
                }
                developers {
                    developer {
                        id.set("ahincho")
                        name.set("Angel Hincho")
                    }
                }
                scm {
                    url.set("https://github.com/ahincho/nova-java-mask-utils")
                    connection.set("scm:git:git@github.com:ahincho/nova-java-mask-utils.git")
                }
            }
        }
    }

    repositories {
        // GitHub Packages
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/ahincho/nova-java-mask-utils")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
        // Maven Central (opcional, requiere firma GPG)
        // maven {
        //     name = "MavenCentral"
        //     url = uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
        //     credentials {
        //         username = System.getenv("MAVEN_USERNAME")
        //         password = System.getenv("MAVEN_TOKEN")
        //     }
        // }
    }
}
```

#### Para starters (Nivel 2) — `nova-java-commons-spring-boot-starter/build.gradle.kts`

```kotlin
plugins {
    java
    id("net.nemerosa.versioning") version "4.0.1"
    `maven-publish`
}

group = "pe.edu.nova.java.starters"

// El BOM es la fuente de verdad de las versiones
dependencies {
    implementation(platform("pe.edu.nova:nova-bom:0.2.0"))
    implementation("pe.edu.nova.java.libs:nova-java-mask-utils")
    implementation("pe.edu.nova.java.libs:nova-java-api-standard")
}

version = versioning.info.display
```

#### Para el BOM (Nivel 4) — `nova-bom/pom.xml`

```xml
<groupId>pe.edu.nova</groupId>
<artifactId>nova-bom</artifactId>
<version>0.2.0</version>
<packaging>pom</packaging>

<properties>
    <nova.mask.version>0.2.0</nova.mask.version>
    <nova.date.version>0.2.0</nova.date.version>
    <nova.mapper.version>0.2.0</nova.mapper.version>
    <nova.api.version>0.2.0</nova.api.version>
    <nova.observability.version>0.2.0</nova.observability.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>pe.edu.nova.java.libs</groupId>
            <artifactId>nova-java-mask-utils</artifactId>
            <version>${nova.mask.version}</version>
        </dependency>
        <!-- ... resto de libs y starters ... -->
    </dependencies>
</dependencyManagement>
```

#### Para el Gradle Plugin (Nivel 5) — `nova-java-spring-boot-gradle-plugin/build.gradle.kts`

```kotlin
plugins {
    `java-gradle-plugin`
    id("net.nemerosa.versioning") version "4.0.1"
    `maven-publish`
}

group = "pe.edu.nova.java.build"
version = versioning.info.display

gradlePlugin {
    plugins {
        create("novaSpringBoot") {
            id = "pe.edu.nova.java.spring-boot"
            implementationClass = "pe.edu.nova.gradle.NovaSpringBootPlugin"
        }
    }
}
```

### 8.5. Workflow de release end-to-end (post-NOVA-SEMVER-13)

> **Cambio importante:** el flujo original con `version-bump` + `publish` con `needs:` fue **deprecado** porque su regex no parseaba `findProperty("version")` (causaba "No jobs were run"). El flujo actual usa `release-please` para generar el PR de release y **tag-triggered publish** para la publicación. Sin `needs:` en el medio.

```
1. Developer crea feature/nova-mask-cc-peru desde main
2. Commits con Conventional Commits:
   - feat(mask-utils): add Peru credit card strategy
   - test(mask-utils): add property tests for CC masking
3. PR a main, CI corre:
   - reusable-commitlint.yml (enforce Conventional Commits via lefthook local + CI)
   - reusable-build-{gradle,maven}.yml (build + tests + lint)
   - reusable-sonarcloud-{gradle,maven}.yml (calidad)
4. Merge a main
5. release-please (workflow: release-please.yml -> reusable-release-please.yml)
   detecta Conventional Commits, abre PR:
   chore(main): release 0.2.0
   - bumpea version 0.1.0 -> 0.2.0 en .release-please-manifest.json
   - actualiza .github/workflows/publish-on-tag.yml si es necesario
   - genera CHANGELOG.md
6. Reviewer aprueba el PR de release
7. Al merge, release-please:
   - crea tag v0.2.0 (y empuja al origin)
   - crea GitHub Release con notas auto-generadas
   - (NO dispara publish directamente — el publish es tag-triggered)
8. Push del tag v0.2.0 dispara workflow publish-on-tag.yml (en cada repo):
   - invoca reusable-release-publish.yml
   - el reusable lee la version del tag (no del manifest)
   - ejecuta gradle build + test + publish
9. Para GitHub Packages: el artefacto aparece en maven.pkg.github.com en ~1-2 min
10. Para Maven Central: Sonatype sincroniza en ~10-30 min
11. Consumidores reciben la nueva version:
    - Actualizan su `nova-bom` version en build.gradle.kts
    - Las nuevas versiones se resuelven transitivamente
```

**Diagrama de componentes:**

```
+-----------------+      +---------------------+      +-------------------------+
| Developer push  | ---> | release-please.yml  | ---> | reusable-release-please |
| (feat:, fix:)   |      | (en cada repo)      |      | .yml (en nova-devops)   |
+-----------------+      +---------------------+      +-------------------------+
                                                              |
                                                              v
                                                       PR de release
                                                       (bumpea version)
                                                              |
                                                       merge -> tag vX.Y.Z
                                                              |
+-----------------+      +---------------------+      +-------------------------+
| GitHub Packages | <--- | reusable-release-    | <--- | publish-on-tag.yml      |
| (artefacto JAR) |      | publish.yml         |      | (en cada repo)          |
+-----------------+      +---------------------+      +-------------------------+
```

### 8.6. Configuracion de `release-please` para el multi-repo

Como Nova Platform tiene **15 repos Java independientes** (10 Gradle + 3 Maven + 2 no-build), se necesitan **3 archivos de configuracion compartidos** (uno por stack + uno global) en cada repo:

**Opcion A — Config por repo** (mas simple, duplicada):

```json
// .release-please-config.json en cada repo
{
  "packages": {
    ".": {
      "release-type": "java",
      "package-name": "nova-java-mask-utils"
    }
  }
}
```

**Opcion adoptada — Config por repo + workflow centralizado en `nova-devops`** (mas mantenible):

> **Aclaracion tecnica:** `release-please` **no puede hacer releases de repos externos** desde un solo config centralizado. Cada repo necesita su propio `.release-please-config.json` + `.release-please-manifest.json`. Sin embargo, el **workflow reusable** si puede centralizarse en `nova-devops` y ser invocado desde cada repo. Esta es la opcion **implementada** en NOVA-SEMVER-13.

#### 8.6.1. Formato real del `.release-please-config.json` (repo simple)

```json
// .release-please-config.json (ejemplo real: nova-java-api-standard)
{
  "packages": {
    ".": {
      "package-name": "pe.edu.nova.java.libs:nova-api-standard",
      "release-type": "java",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "draft": false,
      "prerelease": false
    }
  }
}
```

```json
// .release-please-manifest.json (ejemplo real: nova-java-api-standard)
{
  ".": "0.1.0"
}
```

#### 8.6.2. Formato multi-package (para repos multi-modulo como `commons-starter`)

`nova-java-commons-spring-boot-starter` es multi-modulo (root + 2 submodulos). `release-please` permite configurar multiples paquetes en un solo `.release-please-config.json`:

```json
// .release-please-config.json (ejemplo real: nova-java-commons-spring-boot-starter)
{
  "packages": {
    ".": {
      "package-name": "pe.edu.nova.java.starters:nova-commons-starter",
      "release-type": "java",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "draft": false,
      "prerelease": false
    },
    "nova-api-standard-starter": {
      "package-name": "pe.edu.nova.java.starters:nova-api-standard-starter",
      "release-type": "java",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "draft": false,
      "prerelease": false
    },
    "nova-mask-starter": {
      "package-name": "pe.edu.nova.java.starters:nova-mask-starter",
      "release-type": "java",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "draft": false,
      "prerelease": false
    }
  }
}
```

```json
// .release-please-manifest.json (ejemplo real: nova-java-commons-spring-boot-starter)
{
  ".": "0.1.0",
  "nova-api-standard-starter": "0.1.0",
  "nova-mask-starter": "0.1.0"
}
```

> **Convencion adoptada:** los 3 paquetes de commons-starter comparten la misma version `0.1.0` (single release, no multi-package independent versions). Si en el futuro se necesita independencia, cada paquete tendra su propio `release-type` y rango de bump independiente.

#### 8.6.3. Workflow invocable desde cada repo

```yaml
# .github/workflows/release-please.yml (en CADA repo, plantilla real)
name: Release Please
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release-please:
    uses: ahincho/nova-devops/.github/workflows/reusable-release-please.yml@main
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

El workflow reusable en `nova-devops`:

```yaml
# nova-devops/.github/workflows/reusable-release-please.yml (109 lineas reales)
name: Reusable Release Please
on:
  workflow_call:
    secrets:
      GITHUB_TOKEN:
        required: true
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

### 8.7. Configuracion de `lefthook` (reemplazo moderno de husky)

#### 8.7.1. `lefthook.yml` actual en los 15 repos Java (NOVA-SEMVER-02 v2)

```yaml
# lefthook.yml (estado real al 2026-07-09)
commit-msg:
  commands:
    commitlint:
      run: npx commitlint --edit {1}
```

> **Lo que esta configurado hoy:**
> - Hook `commit-msg` que valida cada commit con `commitlint`.
> - Sin hooks `pre-commit` ni `pre-push` (no se quiere ejecutar `./gradlew test` en cada commit por rendimiento).
> - `glob_filter` no se aplica (porque solo hay un comando y ya filtra por tipo de hook).
>
> **Lo que se podria agregar (futuro, Sprint 5+):**
> ```yaml
> pre-commit:
>   commands:
>     archunit:
>       run: ./gradlew test --tests "*ArchitectureTest"
>       glob_filter: "src/**/*.java"
>       stage_fixed: true   # solo en archivos staged
>
> pre-push:
>   commands:
>     full-test:
>       run: ./gradlew test
>       stage_fixed: false
> ```

#### 8.7.2. Auto-instalacion via npm `prepare` script (NOVA-SEMVER-02 v2)

**Problema resuelto:** antes, cada developer que clonaba un repo debia correr manualmente `lefthook install` para registrar los hooks en `.git/hooks/`. Si olvidaban hacerlo, los commits no se validaban localmente.

**Solucion adoptada (verificada 2026-07-09):** el hook se autoinstala via el lifecycle script `prepare` de npm. Cada `package.json` de los 15 repos Java incluye:

```json
{
  "private": true,
  "scripts": {
    "prepare": "lefthook install"
  },
  "devDependencies": {
    "@commitlint/cli": "^19.8.0",
    "@commitlint/config-conventional": "^19.8.0",
    "lefthook": "^2.1.10"
  }
}
```

**Flujo end-to-end:**
1. Developer clona el repo.
2. Developer corre `npm install` (necesario para commitlint y lefthook locales).
3. npm detecta el script `prepare` y ejecuta `lefthook install` automaticamente.
4. lefthook registra los hooks en `.git/hooks/commit-msg`.
5. A partir de ese momento, cada commit es validado localmente antes de salir del developer.

**Verificacion realizada (2026-07-09):**
- En `nova-java-api-standard`: `npm install` bajo 132 paquetes.
- Output: `sync hooks: ✔️(commit-msg)`.
- Commit subsecuente (`56bde6f`) mostro `🥊 lefthook v2.1.10 hook: commit-msg` en el log — hook activo y validando.

**Beneficios:**
- Developer solo necesita Node.js (que ya necesita para commitlint).
- No requiere Go, brew ni winget (lefthook se distribuye como binarios precompilados en npm).
- Validacion local + CI (`reusable-commitlint.yml` sigue activo en PRs).

#### 8.7.3. `package-lock.json`: commiteado o ignorado

**Decision adoptada:** `package-lock.json` **se commitea** en cada repo. Razones:
1. Garantiza que `npm install` reproduzca el mismo arbol de dependencias en todos los environments (CI, otros developers).
2. Permite que `dependabot`/`renovate` actualicen dependencias via PRs automaticos.
3. Tamano tipico: ~60 KB, insignificante.

**Configuracion relacionada:**
- `.gitignore` NO excluye `package-lock.json` (verificado en los 15 repos).
- `node_modules/` SI esta excluido (lo crea npm localmente, no debe commitearse).

### 8.8. Configuracion para multi-registry (GitHub Packages + Maven Central)

> **Estado actual vs propuesto:** el workflow `reusable-publish-gradle.yml` actual en `nova-devops` es basico (36 lineas: checkout + setup-java + `./gradlew publish`). Los bloques a continuacion muestran la **version propuesta** que reemplazara al actual en Sprint 2 (NOVA-SEMVER-09).

Para que el mismo JAR pueda publicarse en **multiples registries** desde un solo `gradle publish`, se usan **publications** multiples o **repositories** multiples:

```kotlin
val isPublic: Boolean = (project.findProperty("visibility") ?: "public") == "public"

publishing {
    publications {
        // Una publicacion por registry destino
        create<MavenPublication>("github") {
            from(components["java"])
            groupId = "pe.edu.nova.java.libs"
            artifactId = "nova-java-mask-utils"
        }
        create<MavenPublication>("mavenCentral") {
            from(components["java"])
            groupId = "pe.edu.nova.java.libs"
            artifactId = "nova-java-mask-utils"
            // Firmar con GPG es OBLIGATORIO para Maven Central
            pom {
                // ... misma metadata que arriba ...
            }
        }
    }

    repositories {
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/ahincho/nova-java-mask-utils")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
        maven {
            name = "SonatypeOSS"
            url = uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
            credentials {
                username = System.getenv("MAVEN_USERNAME")
                password = System.getenv("MAVEN_TOKEN")
            }
        }
    }
}

// Firma GPG (requerido para Maven Central; ver seccion 10.3)
signing {
    useInMemoryPgpKeys(
        System.getenv("GPG_SIGNING_KEY_ID"),
        System.getenv("GPG_SIGNING_KEY"),
        System.getenv("GPG_SIGNING_PASSWORD")
    )
    sign(publishing.publications["mavenCentral"])
}
```

**Reusable workflow `reusable-publish-gradle.yml` con visibilidad configurable:**

```yaml
# ahincho/nova-devops/.github/workflows/reusable-publish-gradle.yml
name: Reusable Publish to GitHub Packages (Gradle)
on:
  workflow_call:
    inputs:
      java-version:
        description: 'Java version'
        required: false
        type: string
        default: '25'
      visibility:
        description: 'Package visibility (public | private)'
        required: false
        type: string
        default: 'public'
      dry-run:
        description: 'If true, only print commands without executing'
        required: false
        type: string
        default: 'false'
    secrets:
      GITHUB_TOKEN:
        required: true
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main

      - uses: ahincho/nova-devops/.github/actions/nova-setup-java@v1
        with:
          java-version: ${{ inputs.java-version }}
          build-tool: gradle

      - name: Determine visibility
        id: visibility
        env:
          REPO_VISIBILITY: ${{ vars.NOVA_PACKAGE_VISIBILITY || inputs.visibility }}
        run: |
          echo "Package visibility: ${REPO_VISIBILITY}"
          echo "value=${REPO_VISIBILITY}" >> "$GITHUB_OUTPUT"

      - name: Validate visibility compatibility
        shell: bash
        run: |
          # Si el repo es privado, no se puede publicar paquete publico
          REPO_VIS="${{ github.event.repository.visibility }}"
          PKG_VIS="${{ steps.visibility.outputs.value }}"
          if [ "$REPO_VIS" = "public" ] && [ "$PKG_VIS" = "private" ]; then
            echo "::error::Cannot publish private package from public repo"
            exit 1
          fi
          if [ "$REPO_VIS" = "private" ] && [ "$PKG_VIS" = "public" ]; then
            echo "::error::Cannot publish public package from private repo"
            exit 1
          fi

      - name: Publish to GitHub Packages
        if: inputs.dry-run != 'true'
        run: ./gradlew publish -Pvisibility=${{ steps.visibility.outputs.value }}
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Dry-run mode
        if: inputs.dry-run == 'true'
        run: ./gradlew publishDryRun -Pvisibility=${{ steps.visibility.outputs.value }}
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Uso desde un workflow consumidor:**

```yaml
# En el .github/workflows/release.yml de cada repo
publish:
  needs: build
  uses: ahincho/nova-devops/.github/workflows/reusable-publish-gradle.yml@v1
  with:
    visibility: ${{ vars.NOVA_PACKAGE_VISIBILITY || 'public' }}
  secrets:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Para publicar a un registry especifico en CI:

```bash
./gradlew publishGithubPublicationToGitHubPackagesRepository
./gradlew publishMavenCentralPublicationToSonatypeOSSRepository
```

### 8.9. Build Cache (Gradle) — acelerar CI/CD

Gradle ofrece **3 niveles de cache** que se complementan. Combinarlos es clave para reducir el tiempo de CI de **minutos a segundos**.

#### 8.9.1. Nivel 1 — Cache de dependencias (ya implementado)

El workflow `reusable-build-gradle.yml` ya usa `setup-java@v4` con `cache: 'gradle'`. Esto cachea las **dependencias externas** (JARs de Maven Central, plugins) en el GitHub Actions Cache.

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: 25
    cache: gradle   # cachea ~/.gradle/caches/modules-2
```

**Limitacion:** es cache local del runner (no compartido entre jobs del mismo workflow ni entre workflows paralelos). GitHub lo conserva entre ejecuciones si la key es estable.

#### 8.9.2. Nivel 2 — Gradle Build Cache local (no implementado)

**Reutiliza outputs de tasks entre builds** en la misma maquina. Por ejemplo, si `:compileJava` ya se ejecuto, no se vuelve a ejecutar.

```kotlin
// gradle.properties (por repo o por usuario)
org.gradle.caching=true
```

En CI se habilita con flag:

```bash
./gradlew build --build-cache
```

**Tareas cacheables built-in** (segun docs oficiales de Gradle 9.x):
- `JavaCompile`, `Javadoc`
- `Checkstyle`, `Pmd`, `CodeNarc`, `JacocoReport`
- `Test`
- `AntlrTask`, `ValidatePlugins`, `WriteProperties`

**Limitacion:** es local al runner. La primera ejecucion en un runner limpio no se beneficia.

#### 8.9.3. Nivel 3 — Remote Build Cache en GitHub Actions Cache (no implementado, alto impacto)

**Comparte outputs entre runners y entre builds del equipo.** Un CI run popula el cache, otros runs (incluyendo developers locales) lo leen.

**Decision para Nova Platform: el Remote Build Cache se implementa exclusivamente sobre GitHub Actions Cache** (vía la action oficial `gradle/actions/setup-gradle@v4`). No se considera S3, Nexus, Develocity ni otros backends externos porque:

1. GitHub Actions es la plataforma formal de CI/CD adoptada por la empresa.
2. Sera tambien el estandar en los repositorios personales.
3. Evita costos operativos adicionales (storage externo, bandwidth, mantenimiento).
4. Mantiene la consistencia con el resto del tooling que ya vive 100% en GitHub (Actions, Packages, Releases).
5. Si en el futuro se necesita escala empresarial, se puede migrar a Develocity sin cambiar el codigo de los proyectos.

**Configuracion en `settings.gradle.kts` (por repo):**

```kotlin
val isCiServer = System.getenv().containsKey("CI")

buildCache {
    // Local: enabled por default, sirve como Layer 1
    local {
        directory = File(rootDir, ".gradle/build-cache")
    }

    // Remoto (GitHub Actions Cache via gradle/actions): no requiere codigo aqui
    // Se configura en el workflow con gradle/actions/setup-gradle@v4
}
```

**Implementacion en el workflow (GitHub Actions Cache):**

```yaml
# En reusable-build-gradle.yml
- name: Setup Gradle with GitHub Actions Cache
  uses: gradle/actions/setup-gradle@v4
  with:
    cache-read-only: ${{ github.event_name != 'push' }}  # solo CI push popula
    gradle-home-cache-cleanup: true
```

**Comportamiento esperado:**

| Evento | Lee del cache? | Escribe al cache? |
|---|---|---|
| `push` a `main` | Si | **Si** (popula) |
| `pull_request` | Si | No (read-only) |
| `workflow_dispatch` (manual) | Si | Configurable via input |
| `schedule` (cron) | Si | **Si** (popula) |

**Limitaciones que debemos aceptar:**

- **Storage:** 10 GB gratis por repo, 500 MB por artifact, retenido por 7 dias. Mas que suficiente para caches de outputs de Gradle.
- **Latencia:** primer build del dia es lento (cache miss). Builds subsiguientes son muy rapidos.
- **Cross-organization:** no comparte cache entre organizaciones distintas (es por repo, o por org si configuras `actions/cache` con scope org).
- **No hay eviction policy custom:** GitHub decide cuando limpiar segun el limite de storage.

**Configuracion recomendada en `gradle.properties` (cada repo):**

```properties
# Habilita Local Build Cache
org.gradle.caching=true

# Habilita Configuration Cache (Gradle 8.6+)
org.gradle.configuration-cache=true

# Habilita parallel builds (max 4 threads)
org.gradle.parallel=true
org.gradle.workers.max=4
```

**Por que NO se considera S3, Nexus o Develocity para el cache (en Nova Platform):**

- **Develocity (Gradle Enterprise):** costo elevado ($$$), overkill para un meta-framework open source.
- **S3 / GCS / Azure Blob:** requiere cuenta cloud adicional, mantenimiento, monitoring. Inconsistente con stack GitHub-only.
- **Nexus/Artifactory:** ya evaluamos como opcion para hosting de artefactos (seccion 3.4), pero anadir el cache al mismo servidor aumenta la complejidad operativa.
- **Self-hosted custom:** innecesario cuando GitHub ya provee el servicio gratis.

**Migracion futura:** si en algun momento Nova Platform se vuelve un proyecto empresarial con cientos de developers, se puede migrar el cache backend a Develocity o self-hosted **sin cambiar ni una linea del codigo de los proyectos** (solo se cambia el `gradle/actions/setup-gradle` por otra action que conecte al nuevo backend, o se agrega un plugin especifico).

#### 8.9.4. Config Cache (Gradle 8.6+, recomendado para CI)

Separa la **fase de configuration** (resolucion de plugins, evaluacion de build scripts) de la **fase de ejecucion**. Permite reutilizar la configuration entre builds.

```kotlin
// gradle.properties
org.gradle.configuration-cache=true
```

**Beneficio en CI:** hasta **50% menos tiempo** en builds donde los build scripts no cambian.

#### 8.9.5. Distribucion paralela (matrix)

Para reducir tiempo total cuando hay multiples jobs:

```yaml
# .github/workflows/ci.yml
jobs:
  build-matrix:
    strategy:
      fail-fast: false
      matrix:
        java-version: ['21', '25']
        build-tool: ['gradle', 'maven']
    uses: ahincho/nova-devops/.github/workflows/reusable-build-${{ matrix.build-tool }}.yml@main
    with:
      java-version: ${{ matrix.java-version }}
```

**Combinado con Build Cache:** la primera ejecucion cachea, las siguientes (incluso en otros jobs del matrix) leen del cache.

#### 8.9.6. Resumen de capas de cache

| Capa | Que cachea | Alcance | Costo | Habilitado? |
|---|---|---|---|---|
| Dependencies cache | JARs externos | Por job, persistido por GitHub | Gratis | **Si** (en workflows actuales) |
| Local Build Cache | Outputs de tasks | Por runner, en memoria + disco | Gratis | **Pendiente** (agregar `org.gradle.caching=true`) |
| Remote Build Cache | Outputs de tasks | Compartido entre runners y developers | Variable | **Pendiente** (Sprint 5) |
| Configuration Cache | Scripts evaluados | Por runner | Gratis | **Pendiente** (agregar `org.gradle.configuration-cache=true`) |
| Build matrix | Multi-version + multi-tool | Paralelo (mas runners) | Mas minutos de CI | **Pendiente** (NOVA-SEMVER-19) |

---

## 9. Comparativa con JS/TS (resumen visual)

| Aspecto | JS/TS (npm) | Java con propuesta Nova |
|---|---|---|
| Archivo de version | `package.json` | `gradle.properties` + plugin `versioning` |
| Resolucion transitiva | `npm install` | `gradle dependencies` / `mvn dependency:resolve` |
| Bump automatico | `npm version major` | PR de release de `release-please` |
| Publicacion | `npm publish` | `gradle publish` a GitHub Packages / Maven Central / Nexus |
| Lockfile | `package-lock.json` | `gradle.lockfile` (opcional) |
| Conventional Commits | Estandar | Estandar (mismo) |
| Changelog auto | `standard-version` / `semantic-release` | `release-please` lo genera |
| Hooks pre-commit | `husky` | `lefthook` (mas moderno, multi-lenguaje) |
| Registry | npmjs.com | GitHub Packages / Maven Central / Nexus |
| Version inmutable | No (con cooldown) | **Si estrictamente** (no se republica) |

---

## 10. Consideraciones especiales para Java vs JS/TS

### 10.1. Version inmutable

Maven Central **NO tolera reescrituras**. Si publicas `1.0.0` y te das cuenta de un bug critico, **debes publicar `1.0.1`**, nunca `1.0.0` de nuevo. Esto obliga a:

- Tests exhaustivos antes de `publish`.
- Firma GPG obligatoria (en Central).
- Validacion de `pom.xml`/`build.gradle.kts` con `enforcer-plugin`.
- **Revision humana** del PR de release (no se puede automatizar 100%).

Por eso `release-please` con PR de aprobacion es la opcion mas segura: fuerza una revision humana del bump antes de publicar.

### 10.2. GitHub Packages vs Maven Central: groupId restrictivo

GitHub Packages exige que el `groupId` empiece con el nombre del owner de GitHub. Por ejemplo, si el owner es `ahincho`, el `groupId` debe ser `pe.edu.nova.java.libs` donde **`pe.edu.nova.java.libs` empieza con `pe.edu.nova`** (cualquiera, no necesariamente `ahincho`). Esta es una restriccion mas blanda que la de otros registries.

En Maven Central el `groupId` es libre pero **debe ser un dominio que controles** (reverso del dominio). Por eso usamos `pe.edu.nova` (dominio de la organizacion).

### 10.3. Firma GPG (explicacion detallada)

La firma GPG es el mecanismo que prueba que un artefacto JAR publicado en Maven Central **realmente viene de quien dice venir**. Sin firma, cualquiera podria subir un JAR malicioso bajo el mismo `groupId:artifactId`.

#### 10.3.0. Estado actual: pendiente de generacion

> **⚠️ IMPORTANTE — Estado al cierre de este documento:**
>
> **La firma GPG NO esta generada todavia.** Los pasos descritos en 10.3.3-10.3.7 son la guia para cuando se genere, pero la clave aun no existe. Por lo tanto:
>
> - **Publicacion a GitHub Packages:** funciona normalmente sin GPG (es opcional). ✅ Listo.
> - **Publicacion a Maven Central:** **bloqueada** hasta generar la clave. El workflow `reusable-publish-{gradle,maven}-maven-central.yml` no se puede ejecutar sin `GPG_SIGNING_KEY_ID` y `GPG_SIGNING_KEY` en secrets.
> - **Composite action `nova-setup-gpg`:** esta implementada pero solo importa la clave si los secrets existen. Si no, loggea y continua.
>
> **Cuando se decida publicar a Maven Central**, se debe:
> 1. Generar el par de claves GPG (seccion 10.3.3).
> 2. Subir la clave publica a `keys.openpgp.org`.
> 3. Configurar los 3 secrets en GitHub: `GPG_SIGNING_KEY_ID`, `GPG_SIGNING_KEY`, `GPG_SIGNING_PASSWORD`.
> 4. Configurar el namespace `pe.edu.nova` en Sonatype (tickets en `issues.sonatype.org`).
> 5. Activar el workflow `reusable-publish-{gradle,maven}-maven-central.yml`.
>
> **Esto queda como tarea NOVA-SEMVER-29** (sprint futuro, no en los 5 sprints actuales).

#### 10.3.1. Cuando es obligatoria vs opcional

| Registry | Firma GPG | Por que |
|---|---|---|
| **Maven Central** | **Obligatoria** (politica de Sonatype) | Es publico, cualquier developer lo consume, necesita garantia de origen |
| **Sonatype OSSRH** | Obligatoria (proxy a Central) | Mismo motivo que Central |
| **GitHub Packages** | Opcional pero recomendada | Registry mas cerrado (visibilidad por repo/org), pero GPG anade garantia |
| **Nexus on-premise** | Configurable (segun politica del operador) | Depende de la organizacion |

**Para Nova Platform:**
- **MVP (GitHub Packages):** firma opcional, recomendada.
- **Produccion (Maven Central):** firma obligatoria, **bloqueante** para `publish`.

#### 10.3.2. Como funciona el flujo de firma

```
1. Developer genera un par de claves GPG (privada + publica) en su maquina local
2. Sube la clave publica a un keyserver publico (keys.openpgp.org)
3. Configura la clave privada como secret en GitHub Actions
4. Al publicar, Gradle firma el JAR con la clave privada
5. Maven Central verifica la firma contra la clave publica del keyserver
6. Si la firma es valida, el JAR se acepta
```

#### 10.3.3. Generacion del par de claves GPG

**Requisito:** GPG instalado localmente (en Windows: `gpg4win`; en macOS: `gpg` via Homebrew; en Linux: `gpg` preinstalado).

```bash
# 1. Generar clave (RSA 4096 bits, sin passphrase o con passphrase fuerte)
gpg --full-generate-key
# Seleccionar: RSA and RSA, 4096 bits, 0 = key does not expire
# Nombre: "Nova Platform <ahincho@users.noreply.github.com>"
# NO passphrase para CI (o usar passphrase + secret en GitHub)

# 2. Listar claves para obtener el fingerprint
gpg --list-secret-keys --keyid-format=long
# pub   rsa4096/ABC123DEF456 2026-07-08
#       ABC123DEF4567890ABC123DEF4567890ABC1234  <- ESTE es el fingerprint

# 3. Exportar clave publica a keyserver
gpg --keyserver keys.openpgp.org --send-keys ABC123DEF4567890ABC123DEF4567890ABC1234

# 4. Exportar clave privada en formato ASCII-armored (para GitHub Secret)
gpg --armor --export-secret-keys ABC123DEF4567890ABC123DEF4567890ABC1234 > nova-gpg-private.asc
```

#### 10.3.4. Configuracion de secrets en GitHub

En cada repo (o mejor: en la organizacion con inheritance), crear 3 secrets:

| Secret name | Valor | Notas |
|---|---|---|
| `GPG_SIGNING_KEY_ID` | Fingerprint (sin espacios) | `ABC123DEF4567890ABC123DEF4567890ABC1234` |
| `GPG_SIGNING_KEY` | Contenido completo de `nova-gpg-private.asc` (incluyendo `-----BEGIN PGP PRIVATE KEY BLOCK-----`) | GitHub cifra el secret en reposo |
| `GPG_SIGNING_PASSWORD` | Passphrase usado al generar la clave (o vacio si no usaste) | Solo si usaste passphrase |

**NUNCA commit-ear la clave privada al repo.** Solo va en GitHub Secrets.

#### 10.3.5. Configuracion de Gradle (signing plugin)

`settings.gradle.kts` (o `build.gradle.kts`):

```kotlin
plugins {
    `java-library`
    `maven-publish`
    signing
}

// Configuracion GPG en memoria (NO toca archivos del sistema)
signing {
    useInMemoryPgpKeys(
        System.getenv("GPG_SIGNING_KEY_ID"),
        System.getenv("GPG_SIGNING_KEY"),
        System.getenv("GPG_SIGNING_PASSWORD")
    )

    // Firma todas las publications
    sign(publishing.publications)
}
```

**Alternativa: GPG agent en lugar de passphrase en variable**

```kotlin
signing {
    // Lee la passphrase de GPG_PASSPHRASE en lugar de GPG_SIGNING_PASSWORD
    // Recomendado: GPG agent con cache de passphrase
}
```

#### 10.3.6. Reusable workflow para firma GPG

```yaml
# nova-devops/.github/workflows/reusable-publish-maven-central.yml
name: Reusable Publish to Maven Central (Maven)
on:
  workflow_call:
    secrets:
      GPG_SIGNING_KEY_ID: { required: true }
      GPG_SIGNING_KEY:    { required: true }
      GPG_SIGNING_PASSWORD: { required: false }
      MAVEN_USERNAME:     { required: true }
      MAVEN_TOKEN:        { required: true }
    inputs:
      java-version:
        type: string
        default: '25'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: ahincho/nova-devops/.github/actions/nova-setup-java@v1
        with:
          java-version: ${{ inputs.java-version }}
          build-tool: maven

      - name: Import GPG key
        run: |
          echo "${{ secrets.GPG_SIGNING_KEY }}" | gpg --import --batch
          gpg --list-secret-keys --keyid-format=long

      - name: Verify GPG key
        env:
          GPG_SIGNING_KEY_ID: ${{ secrets.GPG_SIGNING_KEY_ID }}
        run: |
          gpg --list-secret-keys "$GPG_SIGNING_KEY_ID" || \
            (echo "::error::GPG key not found" && exit 1)

      - name: Configure Maven with GPG
        run: |
          mkdir -p ~/.m2
          cat > ~/.m2/settings.xml << EOF
          <settings>
            <servers>
              <server>
                <id>central</id>
                <username>${MAVEN_USERNAME}</username>
                <password>${MAVEN_TOKEN}</password>
              </server>
            </servers>
            <profiles>
              <profile>
                <id>central</id>
                <activation><activeByDefault>true</activeByDefault></activation>
                <properties>
                  <gpg.keyname>${GPG_SIGNING_KEY_ID}</gpg.keyname>
                  <gpg.passphrase>${GPG_SIGNING_PASSWORD}</gpg.passphrase>
                  <gpg.executable>gpg</gpg.executable>
                </properties>
              </profile>
            </profiles>
          </settings>
          EOF
        env:
          MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
          MAVEN_TOKEN: ${{ secrets.MAVEN_TOKEN }}
          GPG_SIGNING_KEY_ID: ${{ secrets.GPG_SIGNING_KEY_ID }}
          GPG_SIGNING_PASSWORD: ${{ secrets.GPG_SIGNING_PASSWORD }}

      - name: Deploy to Maven Central
        run: mvn deploy -P central -DskipTests
```

#### 10.3.7. Renovacion y revocacion de claves

**Renovacion (antes de expirar):**
```bash
# 1. Extender expiracion
gpg --edit-key ABC123DEF4567890ABC123DEF4567890ABC1234
gpg> expire
# Seleccionar nueva fecha (ej: 2 anos)

# 2. Re-enviar a keyserver
gpg --keyserver keys.openpgp.org --send-keys ABC123DEF4567890ABC123DEF4567890ABC1234
```

**Revocacion (si la clave se compromete):**
```bash
# 1. Generar certificado de revocacion (HACER AL GENERAR LA CLAVE)
gpg --gen-revoke --armor ABC123DEF4567890ABC123DEF4567890ABC1234 > revoke.asc

# 2. Subir revocation al keyserver
gpg --keyserver keys.openpgp.org --send-keys ABC123DEF4567890ABC123DEF4567890ABC1234
```

**Para Nova Platform:**
- Configurar expiracion de **2 anos** desde la fecha de creacion.
- Configurar recordatorio automatico en CI 30 dias antes de expirar.
- Guardar el certificado de revocacion en un lugar seguro (1Password, KeePass, vault offline).

#### 10.3.8. Troubleshooting GPG comun

| Error | Causa | Solucion |
|---|---|---|
| `gpg: signing failed: No passphrase` | Falta `GPG_SIGNING_PASSWORD` o es incorrecta | Verificar el secret en GitHub |
| `gpg: signing failed: Bad passphrase` | Passphrase incorrecta | Re-generar clave sin passphrase o actualizar secret |
| `gpg: key ABC... not found` | La clave no se importo correctamente | Verificar que `GPG_SIGNING_KEY` contiene el bloque completo |
| `gpg: cannot open /dev/tty` | CI no tiene TTY interactivo | Agregar `--batch --yes --pinentry-mode loopback` |
| Maven Central rejects: `NO_SIGNATURE` | Falta configurar `gpg.keyname` en Maven | Verificar `settings.xml` |
| Maven Central rejects: `BAD_SIGNATURE` | La clave publica no esta en keyserver | Re-enviar con `gpg --send-keys` |
| `UnsupportedVersionException: PGPMarker 5` | GPG version incompatible | Usar GPG 2.4+ (Ubuntu 22.04+ ya lo trae) |

#### 10.3.9. Alternativa moderna: Sigstore / Cosign

Para organizaciones que quieran evitar GPG (que es tecnologia de 1999), hay **Sigstore** (CNCF graduated) que firma artefactos con keys efimeras via OIDC:

```kotlin
// settings.gradle.kts
plugins {
    id("dev.sigstore.sign") version "1.0"
}

// Firma con keyless signing via Fulcio + Rekor
signing {
    sigstore {
        // Configurar OIDC provider (GitHub Actions provee uno)
    }
}
```

**Limitacion:** Maven Central aun NO acepta firmas Sigstore (solo GPG). Es la tecnologia del futuro, pero hoy GPG sigue siendo obligatorio.

### 10.4. Spring Configuration Metadata

Los starters exponen properties via `@ConfigurationProperties`. Para que aparezcan en autocompletado del IDE, generar `META-INF/spring-configuration-metadata.json` con `spring-boot-configuration-processor`. Esto **no afecta el versionado** pero se versiona junto al starter.

### 10.5. Multi-repo vs mono-repo

`release-please` soporta **multi-repo nativo**. Cada repo se versiona independientemente, y el BOM se actualiza con un PR separado. El mono-repo simplificaria el versionado pero complicaria la autonomia de los modulos. Recomendacion: mantener multi-repo, configurar `release-please` centralizado en `nova-devops`.

---

## 11. Troubleshooting comun

### 11.1. Problemas con `release-please`

| Sintoma | Causa probable | Solucion |
|---|---|---|
| `release-please` no abre PR despues de merge a `main` | Token sin scope `contents: write` o `pull-requests: write` | Verificar `permissions:` en el workflow |
| El PR dice `0.0.0` | Conventional Commits no detectados o commits sin formato | Verificar que cada commit sigue el formato `<type>(scope): <desc>` |
| Bump incorrecto (menor cuando deberia ser mayor) | Falta `!` o `BREAKING CHANGE:` en el footer | Agregar al commit message o al PR body |
| PR se reabre con cada push | Branch protection bloquea force-push | Configurar release-please con `pull-request-title-pattern` estable |
| `release-please` dice "no releases found" | No hay tags previos | Agregar `bootstrap-sha: "initial-commit"` en config |
| PR de release abre pero bumpea a `0.1.0` cuando deberia ser `0.2.0` | `.release-please-manifest.json` desactualizado | Editar el manifest manualmente con la version actual |
| `release-please` ignora commits de un sub-package | `package-name` no coincide con la raiz del build | Verificar que `package-name` en `.release-please-config.json` coincide con `group:artifact` de Gradle |
| Multi-package repo (commons-starter): solo bumpea root | Submodulos no configurados en `packages` | Agregar cada subdirectorio como entrada en `packages` (ver §8.6.2) |
| Workflow viejo `version-bump`: `No jobs were run` | Regex no parsea `version = findProperty("version") as String` | **Migrar a release-please** (NOVA-SEMVER-13) — el workflow viejo está deprecado |

### 11.2. Problemas con `commitlint`

| Sintoma | Causa probable | Solucion |
|---|---|---|
| `commitlint` falla en CI pero commitea local | Hook pre-commit no instalado | Configurar `lefthook` correctamente |
| `subject may not be empty` | Commit sin descripcion | Agregar descripcion despues de `:` |
| `type may not be empty` | Falta el tipo (`feat`, `fix`, etc.) | Usar tipo valido |
| `scope must be lowercase` | Scope con mayusculas | Cambiar a lowercase (ej: `MaskUtils` → `mask-utils`) |

### 11.3. Problemas con `net.nemerosa.versioning`

| Sintoma | Causa probable | Solucion |
|---|---|---|
| `version: 0.0.0-UNKNOWN` | Rama no es `main` ni `release/*` ni `feature/*` | Usar la convencion de ramas |
| Version se queda fija en `0.0.0` | Sin tag previo en `main` | Crear tag inicial `v0.1.0` o usar `bootstrap-sha` en release-please |
| `-dirty` aparece en version | Cambios sin commitear | Commitear cambios o usar `dirty = { it }` para desactivar |

### 11.4. Problemas con `maven-publish` a Maven Central

| Sintoma | Causa probable | Solucion |
|---|---|---|
| `401 Unauthorized` | `MAVEN_USERNAME` o `MAVEN_TOKEN` incorrectos | Regenerar token en Sonatype |
| `403 Forbidden` | Namespace no verificado en Sonatype | Crear ticket en `issues.sonatype.org` para verificar `pe.edu.nova` |
| `NO_SIGNATURE` | GPG no configurado | Ver seccion 10.3 de este documento |
| `BAD_SIGNATURE` | Clave publica no en keyserver | `gpg --keyserver keys.openpgp.org --send-keys <FPR>` |
| `Repository does not allow updating artifacts` | Intentando republicar la misma version | No se puede. Bump a nueva version. |
| `Javadoc errors` | Tags Javadoc invalidos | `mvn javadoc:javadoc` local, corregir warnings |

### 11.5. Problemas con Build Cache

| Sintoma | Causa probable | Solucion |
|---|---|---|
| Cache miss constante | `--build-cache` no se pasa | Agregar a `./gradlew` invocations o `org.gradle.caching=true` |
| `Could not resolve` desde remote cache | Cache server caido o credenciales invalidas | Verificar `NOVA_BUILD_CACHE_URL` y secrets |
| `Cache contains non-shareable artifacts` | Task produce output no-relocatable | Agregar `@PathSensitive(PathSensitivity.RELATIVE)` o `@Internal` a campos problematicos |
| `Configuration cache problems` | Script plugin accede a project state en execution phase | Refactorizar a usar Provider API |

### 11.6. Problemas con Composite Actions

| Sintoma | Causa probable | Solucion |
|---|---|---|
| `Cannot find action.yml` | Composite action sin manifest | Crear `action.yml` en la raiz de la carpeta |
| `uses: ...@main not found` | Repo no es publico o la rama no existe | Verificar repo visibility y nombre de rama |
| `Docker is not running` | Action intenta usar Docker en `runs.using: 'docker'` | Cambiar a `composite` o usar self-hosted runner con Docker |

### 11.7. Bugs y workarounds documentados durante NOVA-SEMVER-15 (2026-07-09)

Durante el primer release end-to-end (`api-standard v1.0.0`) se descubrieron 4 bugs y 1 limitacion de GitHub Actions. Todos documentados con causa raiz + fix aplicado.

#### 11.7.1. Bug 1: `startup_failure` en `reusable-release-please.yml`

| Campo | Valor |
|---|---|
| **Sintoma** | Run conclusion: `startup_failure`, jobs: 0, `referenced_workflows: {}` |
| **Causa raiz** | La reusable workflow declaraba `id-token: write` en `permissions` del job. Combinado con el setting del repo (`default_workflow_permissions: read` por default en GitHub), GitHub Actions rechaza la ejecucion. |
| **Fix** | Remover `id-token: write` del `permissions:` block de la reusable. |
| **Archivo** | `nova-devops/.github/workflows/reusable-release-please.yml` (commit `6007066`) |
| **Verificacion** | Run `29002070708` (success) |

```diff
 permissions:
   contents: write
   pull-requests: write
-  id-token: write
```

#### 11.7.2. Bug 2: "GitHub Actions is not permitted to create or approve pull requests"

| Campo | Valor |
|---|---|
| **Sintoma** | Run de release-please fails con `##[error]release-please failed: GitHub Actions is not permitted to create or approve pull requests.` |
| **Causa raiz** | El repo caller (api-standard) tenia `default_workflow_permissions: read` y `can_approve_pull_request_reviews: false`. Aunque el workflow defina `permissions: pull-requests: write`, el setting del repo lo bloquea. |
| **Fix** | Cambiar settings del repo via API: `default_workflow_permissions: write` + `can_approve_pull_request_reviews: true`. |
| **Comando** | `gh api -X PUT repos/OWNER/REPO/actions/permissions/workflow -H 'Content-Type: application/json' -d '{"default_workflow_permissions":"write","can_approve_pull_request_reviews":true}'` |
| **Verificacion** | Run `29002070708` (success, PR abierto) |

#### 11.7.3. Bug 3: `./gradlew: Permission denied` en Linux runner

| Campo | Valor |
|---|---|
| **Sintoma** | Step `./gradlew publish` falla con `##[error]Process completed with exit code 126.` Log: `./gradlew: Permission denied`. |
| **Causa raiz** | El script `gradlew` se commitea desde Windows con line endings CRLF. Cuando se pushea a GitHub, queda con CRLF. Linux (runner ubuntu-latest) requiere LF + bit de ejecucion (`chmod +x`). |
| **Fix** | Agregar step explicito `chmod +x ./gradlew` despues de checkout en TODOS los workflows que ejecutan gradle. |
| **Archivos** | 8 `publish-on-tag.yml` (uno por repo Gradle) + reusable `reusable-release-publish.yml`. |
| **Workaround alternativo** | Configurar `git config core.autocrlf false` localmente antes de commitear, pero es fragil. El step `chmod +x` es mas robusto. |
| **Verificacion** | Run `29024268916` (success) |

```yaml
- name: Checkout code
  uses: actions/checkout@v4
  with:
    fetch-depth: 0

- name: Make gradlew executable
  run: chmod +x ./gradlew
```

#### 11.7.4. Bug 4: `dirty = { it }` no compila en Gradle 9.6.1 + Kotlin DSL

| Campo | Valor |
|---|---|
| **Sintoma** | Build fails con: `Script compilation error: Line 14: dirty = { it } ^ Assignment type mismatch: actual type is 'Function1<String, String>', but 'Closure<String!>!' was expected.` |
| **Causa raiz** | El plugin `net.nemerosa.versioning 4.0.1` espera un `Closure<String>` de Groovy. En Kotlin DSL con Gradle 9.6.1, una lambda `{ it }` se convierte a `Function1<String, String>` (Kotlin), no a `Closure` (Groovy). Esto era valido en Gradle 8.x con type coercion, pero Gradle 9.x endurece la validacion de tipos. |
| **Fix** | Remover la linea `dirty = { it }` del bloque `versioning {}`. El default del plugin ya desactiva el sufijo `-dirty` si no se setea (aunque la documentacion del plugin sugiera configurarlo explicitamente). |
| **Archivos** | 11 archivos `build.gradle.kts` (8 repos simples + commons-starter root + 2 submodules + example). |
| **Verificacion** | Run `29024268916` (success) |

```diff
 versioning {
     releaseMode = "snapshot"
     displayMode = "snapshot"
-    dirty = { it }
     releaseBuild = false
 }
```

#### 11.7.5. Limitacion GitHub Actions: Reusable workflow + tag push da 0 jobs

| Campo | Valor |
|---|---|
| **Sintoma** | Cuando `publish-on-tag.yml` usa `uses: ahincho/nova-devops/.github/workflows/reusable-release-publish.yml@main`, el run da `conclusion: failure` con **0 jobs** y `referenced_workflows: {}`. |
| **Causa raiz** | Limitacion conocida de GitHub Actions: cuando una reusable workflow (`on: workflow_call`) es invocada por un workflow que se triggerea con `push: tags:`, GitHub Actions tiene problemas para resolver la referencia. El campo `referenced_workflows` aparece vacio en la API, indicando que la referencia no se proceso. |
| **Workaround aplicado** | **Inlinear la logica** de la reusable workflow directamente en `publish-on-tag.yml` de cada repo. El workflow resultante tiene ~70 lineas (vs ~110 de la reusable original) pero funciona correctamente. |
| **Probar otra teoria** | Se intento restaurar la reusable + agregar `workflow_dispatch` para forzar otro trigger. El workflow_dispatch tampoco resolvio la reusable (mismo 0 jobs). El issue es especifico al chain reusable + tag push. |
| **Estado** | Pendiente reportar a GitHub Support / buscar en github.com/community. Mientras tanto, logica inlined en los 8 `publish-on-tag.yml`. La reusable `reusable-release-publish.yml` queda en `nova-devops` para referencia historica, pero NO se invoca actualmente. |
| **Verificacion del workaround** | Run `29024268916` (success con logica inlined). |

---

## 12. Roadmap de adopcion (propuesto)

> **Nota sobre el alcance:** este roadmap cubre exclusivamente los **15 repos Java** y los **3 repos multi-stack** (nova-devops, nova-bom, nova-infrastructure). Los 4 repos NestJS (`nova-nestjs-*`) se abordaran en un roadmap separado en el futuro.

### Pre-requisitos (antes de Sprint 0) — Alineacion de repos

> **Estado verificado al 2026-07-09:** Los pre-requisitos 00a-00d ya fueron **completados** en los 13 repos Java (10 Gradle + 3 Maven). Todos los repos Gradle tienen `gradle.properties`, todos migraron de `pe.edu.galaxy.training` a `pe.edu.nova`, y el placeholder `OWNER` fue corregido a `ahincho`. Verificado en la seccion 15.7.

0a. **NOVA-SEMVER-00a:** ✅ Crear `gradle.properties` en los **10 repos Gradle** (api-standard, commons-starter, date-utils, example, gradle-plugin, mapper-utils, mask-utils, observability-starter, observability-utils, starter) con `version=0.1.0-SNAPSHOT` y `group=pe.edu.nova.java.libs` (o `pe.edu.nova.java.starters` segun nivel).
0b. **NOVA-SEMVER-00b:** ✅ Migrar `groupId` de `pe.edu.galaxy.training.java.libs` a `pe.edu.nova.java.libs` en todos los `build.gradle.kts` y `pom.xml` (13 repos Java).
0c. **NOVA-SEMVER-00c:** ✅ Corregir placeholder `OWNER` a `ahincho` en las secciones `publishing.repositories.maven.url` de todos los repos que tienen `maven-publish` (10 repos con `maven-publish` configurado).
0d. **NOVA-SEMVER-00d:** ✅ Renombrar packages Java de `pe.edu.galaxy.training` a `pe.edu.nova` en el codigo fuente de cada repo (refactoring de imports y paquetes, ~200 archivos modificados en 13 repos).

> **Nota:** NOVA-SEMVER-00d es la tarea mas grande (implica renombrar directorios y archivos Java). Puede ejecutarse en paralelo con Sprint 0 si se prefiere, pero debe completarse antes de Sprint 1 para que los workflows de publish generen artefactos con el `groupId` correcto.

### Sprint 0 (inmediato) — Fundamentos de versioning

1. **NOVA-SEMVER-01:** ✅ Adoptar Conventional Commits en los 19 repos (15 Java + 4 NestJS).
2. **NOVA-SEMVER-02:** ✅ Configurar `commitlint` + `lefthook` en cada repo (15 repos Java tienen `commitlint.config.js` + `lefthook.yml`).
3. **NOVA-SEMVER-03:** ✅ Agregar `net.nemerosa.versioning` 4.0.1 a los **10 repos Gradle** (api-standard, commons-starter, date-utils, example, gradle-plugin, mapper-utils, mask-utils, observability-starter, observability-utils, starter). **3 repos fueron migrados de Maven a Gradle** para esta unificacion: mask-utils, observability-utils, starter.
4. **NOVA-SEMVER-04:** ✅ Mover versiones hardcodeadas a `gradle.properties` con `${version}` reference (10 repos Gradle usan `findProperty("version") as String`).

### Sprint 1 — Reusable workflows faltantes en `nova-devops`

> **Estado verificado al 2026-07-09:** Sprint 1 **COMPLETADO** (commit `98da16b` en `ahincho/nova-devops`). 3 reusable workflows + 3 composite actions creadas.

5. **NOVA-SEMVER-05:** ✅ Crear `reusable-commitlint.yml` (enforce Conventional Commits).
6. **NOVA-SEMVER-06:** ✅ Crear `reusable-release-please.yml` (orquestar release multi-repo).
7. **NOVA-SEMVER-07:** ✅ Crear `reusable-changelog.yml` (auto-generar CHANGELOG.md).
8. **NOVA-SEMVER-08:** ✅ Crear las 3 composite actions de setup: `nova-setup-java`, `nova-setup-node`, `nova-setup-gpg` (ver seccion 5.4). Implementadas en `nova-devops/.github/actions/`.

### Sprint 2 — Multi-registry publishing

> **Estado verificado al 2026-07-09:** Sprint 2 **COMPLETADO** (commit `aa7692c` en `ahincho/nova-devops` + commits individuales en 9 repos Java). 6 nuevos workflows + 2 actualizados + signing plugin en 9 repos Gradle.

9. **NOVA-SEMVER-09:** ✅ Crear `reusable-publish-{gradle,maven}-multi-registry.yml` (GitHub Packages + Maven Central).
10. **NOVA-SEMVER-10:** ✅ Crear `reusable-publish-{gradle,maven}-maven-central.yml` con firma GPG. Adicionalmente: signing plugin (`id("signing")` + `useInMemoryPgpKeys`) agregado a los 9 repos Gradle con `maven-publish` (se excluye `example` que no publica).
11. **NOVA-SEMVER-11:** ✅ Crear `reusable-publish-{gradle,maven}-nexus.yml` para on-premise.
12. **NOVA-SEMVER-12:** ✅ Preparar workflows de publish con firma GPG **opcional** (skip automatico si no hay secrets configurados). La generacion de claves GPG queda en backlog (NOVA-SEMVER-29).

### Sprint 3 — Activacion y primer release

> **Estado verificado al 2026-07-09:** NOVA-SEMVER-13 **COMPLETADO** (10 commits en `ahincho/*` + `688e5d2` en `nova-devops`). Assets creados: 5 archivos por repo (`.release-please-config.json`, `.release-please-manifest.json`, `.github/workflows/ci.yml` nuevo, `.github/workflows/release-please.yml`, `.github/workflows/publish-on-tag.yml`). Nuevo workflow reusable: `reusable-release-publish.yml` en `nova-devops`. Migración desde el antiguo patron `version-bump` + `publish` con `needs:` (causa del bug "No jobs were run" por regex que no parseaba `findProperty("version")`).

13. **NOVA-SEMVER-13:** ✅ Configurar `.release-please-config.json` + `.release-please-manifest.json` en cada repo Java (10/10). Workflow reusable `reusable-release-please.yml` ya existia desde Sprint 1; nuevo `reusable-release-publish.yml` activado por tag push.
14. **NOVA-SEMVER-14:** ⏳ Crear namespace `pe.edu.nova` en Sonatype OSSRH.
15. **NOVA-SEMVER-15:** ⏳ Primer release de prueba: `0.1.0` para todos los modulos via `release-please`. Configuracion lista; falta ejecutar el primer ciclo (push commits Conventional → release-please crea PR → merge → tag v0.1.0 → publish).
16. **NOVA-SEMVER-16:** ⏳ Publicar `0.1.0` a GitHub Packages y verificar consumo desde el BOM.

### Sprint 4 — Publicacion a Maven Central y optimizaciones

17. **NOVA-SEMVER-17:** Preparar publicacion a Maven Central via Sonatype (workflow listo, **ejecucion bloqueada** hasta completar NOVA-SEMVER-29: generacion de claves GPG).
18. **NOVA-SEMVER-18:** Documentar politica de bump en `docs/adr/ADR-011-versioning.md`.
19. **NOVA-SEMVER-19:** Crear `reusable-build-matrix.yml` (Java 21 + 25).
20. **NOVA-SEMVER-20:** Crear `reusable-owasp-check.yml` (CVEs).
21. **NOVA-SEMVER-21:** Crear `reusable-sbom.yml` (CycloneDX).
22. **NOVA-SEMVER-22:** Crear matriz de compatibilidad (que version de mask-utils va con cual api-standard).

### Sprint 5 — Build Cache y Composite Actions

23. **NOVA-SEMVER-23:** Habilitar `org.gradle.caching=true` en `gradle.properties` de los **10 repos Gradle**.
24. **NOVA-SEMVER-24:** Habilitar `org.gradle.configuration-cache=true` en los **10 repos Gradle**.
25. **NOVA-SEMVER-25:** Agregar `gradle/actions/setup-gradle@v4` a `reusable-build-gradle.yml` para usar GitHub Actions Cache.
26. **NOVA-SEMVER-26:** Crear las **4 composite actions restantes** en `nova-devops/.github/actions/` (`nova-gather-facts`, `nova-publish-aggregator`, `nova-configure-gradle-cache`, `nova-validate-build`; `action.yml` completo, ver seccion 5.5).
27. **NOVA-SEMVER-27:** Migrar `reusable-build-{gradle,maven}.yml` y `reusable-publish-{gradle,maven}.yml` para usar las composite actions.
28. **NOVA-SEMVER-28:** Medir tiempos de CI antes/despues y documentar la mejora.

### Backlog (futuro, no en sprints activos)

29. **NOVA-SEMVER-29:** Generar par de claves GPG y configurar secrets en GitHub (cuando se decida publicar a Maven Central). Guia completa en seccion 10.3.
30. **NOVA-SEMVER-30:** Configurar variable `NOVA_PACKAGE_VISIBILITY` en los 19 repos (default `public`). Guia en seccion 3.1.1.

---

## 13. Referencias (fuentes verificadas)

| Fuente | URL | Verificado |
|---|---|---|
| `net.nemerosa.versioning` (Gradle plugin) | https://github.com/nemerosa/versioning | 206 stars, 22 releases, 4.0.1 Abr 2026 |
| `semantic-release` | https://github.com/semantic-release/semantic-release | 23.9k stars, 451 releases, v25.0.5 Jun 2026 |
| Conventional Commits (estandar) | https://www.conventionalcommits.org/ | Estandar de la industria |
| `release-please` (Google) | https://github.com/googleapis/release-please | GitHub Action oficial de Google |
| Maven Release Plugin | https://maven.apache.org/maven-release/maven-release-plugin/ | Plugin oficial Apache |
| GitHub Packages (Maven) | https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry | Documentacion oficial |
| Maven Central (Sonatype) | https://central.sonatype.com/ | Registro publico oficial |
| Sonatype OSSRH | https://s01.oss.sonatype.org/ | Staging area para Central |
| JFrog Artifactory OSS | https://jfrog.com/community/open-source/ | Alternativa self-hosted |
| Sonatype Nexus OSS | https://www.sonatype.com/products/sonatype-nexus-oss | Alternativa self-hosted |
| Spring Framework releases | https://github.com/spring-projects/spring-framework/releases | Caso de estudio real, tags `v7.0.8` |
| jqwik (Java multi-modulo) | https://github.com/jqwik-team/jqwik | Caso de estudio con `buildSrc` + shell script |
| `commitlint` | https://github.com/conventional-changelog/commitlint | Estandar de la industria |
| `lefthook` (multi-lenguaje hooks) | https://github.com/evilmartians/lefthook | Reemplazo moderno de husky |

---

## 14. Conclusion

**Si se puede llevar semantic-version en Java, y de hecho es la norma.** La diferencia con JS/TS no es que falte, sino que requiere **3 piezas adicionales** que npm trae integradas:

1. **Plugin de versionado en build** (`net.nemerosa.versioning` para Gradle, `maven-release-plugin` para Maven).
2. **Herramienta de release automation** (`release-please` o `semantic-release`).
3. **Convencion de commits** (Conventional Commits + `commitlint`).

Una vez configuradas las tres, el flujo es equivalente al de npm:

| npm | Nova Platform Java |
|---|---|
| `npm version patch` | PR de `release-please` con bump |
| `npm publish` | `gradle publish` a GitHub Packages / Maven Central / Nexus |
| `package-lock.json` | `gradle.lockfile` |
| `~/.npmrc` | `~/.gradle/gradle.properties` + secrets de CI |

**La unica diferencia fundamental:** la inmutabilidad del registry (especialmente Maven Central) obliga a revision humana del bump antes de publicar, lo cual se logra naturalmente con el PR de `release-please`. Es un trade-off aceptable a cambio de la inmutabilidad y garantia de Central.

**Estado actual vs objetivo (actualizado al 2026-07-09):**

| Aspecto | Hoy (sin semver) | Pre-req + Sprint 0 (objetivo) | **Estado real 2026-07-09** ✅ | Sprint 4 (con propuesta) | Sprint 5 (con cache) |
|---|---|---|---|---|---|
| `groupId` | `pe.edu.galaxy.training` (viejo) | `pe.edu.nova` (migrado) | ✅ `pe.edu.nova` (migrado en 13 repos Java) | Idem | Idem |
| `gradle.properties` | No existe en ningun repo Gradle | Creado con `version` y `group` | ✅ 10 repos con GP (todos los Gradle code) | Idem | + `org.gradle.caching=true` + `org.gradle.configuration-cache=true` |
| Versionado en build | Hardcoded en `build.gradle.kts` / `pom.xml` | Idem | ✅ Plugin `net.nemerosa.versioning` 4.0.1 en 10 repos | Idem | Idem |
| Convencion de commits | Libre | Idem | ✅ Conventional Commits enforced (`commitlint` + `lefthook` auto-install en 15 repos) | Idem | Idem |
| Release automation | Bump manual via PR labels | Idem | ✅ Configuracion lista (NOVA-SEMVER-13). ⏳ Falta ejecutar primer ciclo (NOVA-SEMVER-15) | `release-please` PR + aprobacion humana | Idem |
| Publicacion | GitHub Packages unico (con placeholder `OWNER`) | GitHub Packages (con `ahincho` corregido) | ✅ GitHub Packages (con `ahincho`, 9 repos + 2 submodules commons-starter con `maven-publish`) | + Maven Central + Nexus (multi-registry workflows listos) | Idem |
| Visibilidad del paquete | Fija (default privado) | Idem | ⏳ Pendiente (NOVA-SEMVER-30) | Parametrizable via `vars.NOVA_PACKAGE_VISIBILITY` (default `public`) | Idem |
| Changelog | Manual | Idem | ✅ Auto-generado por `release-please` (NOVA-SEMVER-07) | Idem | Idem |
| GitHub Release | No | Idem | ✅ Auto al mergear PR de release (NOVA-SEMVER-13) | Idem | Idem |
| Build Cache | Solo dependencies cache | Idem | ⏳ Solo deps cache (NOVA-SEMVER-23-25 en Sprint 5) | Idem | + Local + GH Actions + Configuration Cache |
| Composite Actions | 0 | 0 | ✅ 3 implementadas (Sprint 1: `nova-setup-java`, `nova-setup-node`, `nova-setup-gpg`). ⏳ 4 pendientes (NOVA-SEMVER-26 Sprint 5) | ✅ 3 implementadas + 4 diseñadas | **7 totales** (`nova-setup-java/node/gpg`, `nova-gather-facts`, `nova-publish-aggregator`, `nova-configure-gradle-cache`, `nova-validate-build`) |
| Reusable workflows | 8 originales | 8 | ✅ **20** workflows en `nova-devops` (8 orig + 3 Sprint 1 + 6 Sprint 2 + 1 Sprint 3 + 2 plantillas). 2 version-bump-* deprecados | 20 + 4 composite actions | 20 + 7 composite actions (migrados) |
| lefthook auto-install | Manual (`lefthook install` por dev) | Manual | ✅ Auto via `npm prepare` script + `lefthook@^2.1.10` (NOVA-SEMVER-02 v2, 2026-07-09) | Idem | Idem |
| Nomenclatura | GT-SEMVER | NOVA-SEMVER | ✅ NOVA-SEMVER | Idem | Idem |
| Producto | "Galaxy Training" / "Nova Platform" | Nova (unico) | ✅ Nova (unico, todos los repos renombrados) | Idem | Idem |
| Troubleshooting | No documentado | No documentado | ✅ Si, seccion 11 con 6 sub-tablas | Idem | Idem |
| Firma GPG | No requerida | No requerida | 🟡 Preparada (composite action `nova-setup-gpg` lista + signing plugin en 9 repos), clave NO generada (NOVA-SEMVER-29) | Activada (workflows listos, ejecucion bloqueada hasta generar clave) | Idem |
| **Total actividades NOVA-SEMVER** | 0 | 4 pre-req (00a-00d) | **17/34 completadas** (4 pre-req + 4 Sprint 0 + 4 Sprint 1 + 4 Sprint 2 + 1 Sprint 3) | 28 (01-28) | **34** (4 pre-req + 28 sprints + 2 backlog) |
| Alcance | — | Solo Java + multi-stack | ✅ Solo Java + multi-stack (NestJS fuera) | Idem | Idem (NestJS en roadmap separado) |

---

## 15. Doble check final (estado al cierre del documento)

### 15.1. Decisiones tomadas en esta iteracion

| # | Decision | Resolucion | Documentado en |
|---|---|---|---|
| 1 | **Build Cache backend** | **GitHub Actions Cache** exclusivamente (via `gradle/actions/setup-gradle@v4`). Sin S3, Nexus ni Develocity. | Seccion 8.9.3 |
| 2 | **Composite actions** | **Las 7 se implementan todas** desde el inicio (3 ya listas, 4 en Sprint 5). | Seccion 5.4-5.5 con `action.yml` completo de cada una |
| 3 | **Visibilidad del paquete** | **Parametrizable por repo** via `vars.NOVA_PACKAGE_VISIBILITY` (default `public`). | Seccion 3.1.1 + seccion 8.8 con workflow propuesto |
| 4 | **Publicacion en empresa vs personal** | **Ambos publicos** (repos publicos + paquetes publicos por default). | Seccion 3.1.1 |
| 5 | **Firma GPG** | **Pendiente, NO generada**. Guia completa documentada. NO bloquea GitHub Packages. | Seccion 10.3.0 + 10.3.3-10.3.7 |
| 6 | **Alcance del roadmap** | **Solo Java y multi-stack.** NestJS se abordara en un roadmap separado. | Seccion 12 (nota de alcance) |
| 7 | **Release automation tool** | **`release-please` de Google** (no `semantic-release`, no `standard-version`). Declarativo, sin runtime deps, multi-repo friendly. | Seccion 4.3 + 8.6 |
| 8 | **Publish trigger** | **Tag push** (`vX.Y.Z` pushed to main) dispara `publish-on-tag.yml`. NO push directo a main, NO `workflow_dispatch`. | Seccion 8.5 (diagrama) |
| 9 | **Workflows viejos `version-bump-*`** | **Deprecados** post-NOVA-SEMVER-13. Causa: regex no parseaba `findProperty("version") as String`. | Seccion 5.0.6 + 11.1 |
| 10 | **Multi-module repo (commons-starter)** | **Single release con multi-package config**. Las 3 paquetes (root + 2 submodules) comparten version `0.1.0`. No multi-package independent versions. | Seccion 8.6.2 |
| 11 | **Repos sin workflows previos** | **Crear ci.yml + release-please completo** (config + manifest + workflow), no solo `release-please-config.json`. | Seccion 8.6 (10 repos afectados en NOVA-SEMVER-13) |
| 12 | **lefthook installation** | **Auto via `npm prepare` script**. NO manual `lefthook install`. Reduce friccion para nuevos developers. | Seccion 8.7.2 |
| 13 | **`package-lock.json`** | **Commiteado** en todos los repos. Garantiza reproducibilidad de `npm install`. | Seccion 8.7.3 |
| 14 | **Pin de composite actions** | **`@main` temporalmente**, switch a `@v1` tras Sprint 3 estabilice. 22 referencias cambiadas. | Seccion 5.6 + §15.2 fila "@v1 -> @main" |

### 15.2. Piezas verificadas contra el estado real de los repos (actualizado 2026-07-09)

| Pieza | Estado al 2026-07-09 | Verificado |
|---|---|---|
| **20 workflows en `nova-devops`** (8 orig + 3 Sprint 1 + 6 Sprint 2 + 1 Sprint 3 + 2 plantillas). 2 `reusable-version-bump-*` deprecados | ✅ OK | ✅ Confirmado (inventario manual 2026-07-09, ver §5.0) |
| 3 composite actions en `nova-devops` (`nova-setup-java`, `nova-setup-node`, `nova-setup-gpg`) | ✅ OK (Sprint 1, NOVA-SEMVER-08) | ✅ Confirmado con `Get-ChildItem .github/actions` |
| **9 repos + 2 submodules commons-starter = 11 publicaciones con `maven-publish`** (api-standard, commons-starter/nova-api-standard-starter, commons-starter/nova-mask-starter, date-utils, gradle-plugin, mapper-utils, mask-utils, observability-starter, observability-utils, starter). commons-starter root NO publica. | ✅ OK | ✅ Confirmado con grep en 11 `build.gradle.kts` |
| 9 repos Gradle con signing plugin `id("signing")` + `useInMemoryPgpKeys` (NOVA-SEMVER-10) — excluye `example` (no publica) | ✅ OK (Sprint 2) | ✅ Confirmado con `Select-String` en 11 `build.gradle.kts` |
| 2 repos SIN `maven-publish` (commons-starter root, example) | ✅ Esperado | ✅ Confirmado |
| 10 repos con `gradle.properties` (api-standard, commons-starter, date-utils, example, gradle-plugin, mapper-utils, mask-utils, observability-starter, observability-utils, starter) | ✅ OK | ✅ Confirmado con `Test-Path` |
| Todos los repos migrados a `pe.edu.nova` (13 Java: 10 Gradle + 3 Maven) | ✅ OK | ✅ 0 referencias a `pe.edu.galaxy` en source files |
| Publishing URL usa `ahincho` (no mas placeholder `OWNER`) | ✅ OK | ✅ Confirmado en api-standard y resto |
| 10 repos con `net.nemerosa.versioning` 4.0.1 | ✅ OK | ✅ Confirmado con grep en `build.gradle.kts` |
| 15 archivos `commitlint.config.js` (todos los repos Java) | ✅ OK | ✅ Confirmado con `Test-Path` |
| 15 archivos `lefthook.yml` (todos los repos Java) | ✅ OK | ✅ Confirmado con `Test-Path` |
| **15 archivos `package.json` con `lefthook@^2.1.10` + `prepare: lefthook install`** (NOVA-SEMVER-02 v2, 2026-07-09) | ✅ OK | ✅ Hook auto-instalado verificado en api-standard (commit `56bde6f` mostro `🥊 lefthook v2.1.10 hook: commit-msg` en log) |
| 10 archivos `.release-please-config.json` + 10 `.release-please-manifest.json` (uno por repo Gradle) | ✅ OK (NOVA-SEMVER-13, Sprint 3) | ✅ Confirmado con `Test-Path`. commons-starter usa multi-package (3 paquetes) |
| **29 archivos nuevos de workflow en 10 repos** (`ci.yml` + `release-please.yml` + `publish-on-tag.yml`, este ultimo NO existe en `example`) = 9×3 + 1×2 | ✅ OK (NOVA-SEMVER-13) | ✅ Confirmado con `Get-ChildItem .github/workflows` |
| Workflow reusable `reusable-release-publish.yml` en `nova-devops` | ✅ OK (NOVA-SEMVER-13) | ✅ Trigger: tag push `vX.Y.Z`, sync version desde tag, `./gradlew publish` |
| 4 composite actions restantes (`nova-gather-facts`, `nova-publish-aggregator`, `nova-configure-gradle-cache`, `nova-validate-build`) | ⏳ Pendiente (NOVA-SEMVER-26, Sprint 5) | ✅ Disenadas en §5.5 (no implementadas) |
| 0 archivos `.gradle/build-cache/` ni config de Remote Cache | ⏳ Pendiente (Sprint 5) | ✅ NOVA-SEMVER-23-25 |
| 0 secrets GPG en GitHub | ⏳ Diferido (NOVA-SEMVER-29) | ✅ Backlog, no urgente |
| 0 variables `NOVA_PACKAGE_VISIBILITY` configuradas | ⏳ Pendiente (NOVA-SEMVER-30) | ✅ Backlog |
| 4 repos NestJS sin CI/CD, sin hooks, todos a `1.0.0` | ⏳ Fuera de alcance | ⚠️ Se abordara en roadmap separado |
| Gradle wrapper 9.2.0 consistente en los 10 repos | ✅ OK | ✅ Confirmado en `gradle-wrapper.properties` |
| 0 archivos `build/` o `.gradle/` tracked en git | ✅ OK | ✅ Confirmado con `git ls-files` (10 repos limpios) |
| `@v1` -> `@main` pin temporal en workflows | ✅ OK (NOVA-SEMVER-13 side fix) | ✅ 22 referencias cambiadas. Pin revertira a `@v1` tras Sprint 3 estabilice |

### 15.3. Lo que esta LISTO y verificado (no requiere accion)

- Documento de estrategia completo con 15 secciones (numeracion corregida, sin duplicados).
- **34 actividades NOVA-SEMVER** distribuidas: 4 pre-requisitos (00a-00d) + 28 en 5 sprints (01-28) + 2 backlog (29-30).
- Codigo de ejemplo para Gradle (`maven-publish`, `signing`, `buildCache`, `org.gradle.caching`) — marcado como **objetivo**, no estado actual.
- **7 composite actions con `action.yml` completo** (seccion 5.5), incluyendo `nova-setup-gpg` corregida para usar `inputs` en vez de `secrets.*`.
- Configuracion YAML para `release-please` (por repo, no centralizada), `commitlint`, `lefthook`, `gradle/actions/setup-gradle`.
- **Reusable workflow `reusable-publish-gradle.yml` propuesto** con visibilidad configurable (seccion 8.8) — distinguido del actual (36 lineas, basico).
- 14 reusable workflows propuestos (8 ya existen + 6 nuevos).
- **Politica de Build Cache definida: GitHub Actions Cache unicamente** (seccion 8.9.3).
- 6 tablas de troubleshooting categorizadas (seccion 11).
- 14 fuentes externas verificadas con stars/releases/fechas.

### 15.4. Lo que esta INCOMPLETO o tiene dudas explicitas (actualizado 2026-07-09)

| Punto | Estado | Que falta |
|---|---|---|
| **Pre-requisitos** (NOVA-SEMVER-00a-00d) | ✅ **COMPLETADO** (2026-07-09) | Nada. Verificado: 13 repos Java con GP, groupId migrado, OWNER corregido, packages renombrados |
| **Sprint 1** (NOVA-SEMVER-05-08) | ✅ **COMPLETADO** (commit `98da16b` en `nova-devops`) | Nada. 3 reusable workflows + 3 composite actions implementadas |
| **Sprint 2** (NOVA-SEMVER-09-12) | ✅ **COMPLETADO** (commit `aa7692c` en `nova-devops` + commits en 9 repos Java) | Nada. 6 nuevos workflows + signing plugin en 9 repos Gradle |
| **NOVA-SEMVER-13** (release-please config) | ✅ **COMPLETADO** (commit `688e5d2` en `nova-devops` + 10 commits en repos Java) | Nada. `.release-please-config.json` + `.release-please-manifest.json` + 3 workflows por repo + `reusable-release-publish.yml` en nova-devops. Patron `version-bump` deprecado. |
| **NOVA-SEMVER-15** (primer release end-to-end) | ✅ **COMPLETADO** (2026-07-09, run `29024268916` success) | Nada. `api-standard v1.0.0` publicado. 4 bugs descubiertos y arreglados (ver §11.7). Workaround aplicado para limitacion de reusable + tag push (logica inlined en 8 `publish-on-tag.yml`). |
| **NOVA-SEMVER-23-24** (Local Build Cache + Configuration Cache) | ✅ **COMPLETADO** (2026-07-09, commits en 10 repos) | Nada. `org.gradle.caching=true` + `org.gradle.configuration-cache=true` agregados a 10 `gradle.properties` (9 en `D:\Galaxy\Projects\java\` + example en `/instances/`). |
| **NOVA-SEMVER-25** (Remote Build Cache via GitHub Actions) | ✅ **COMPLETADO** (2026-07-09, commit `27fb98e` en `nova-devops`) | Nada. `gradle/actions/setup-gradle@v4` agregado a `reusable-build-gradle.yml` con `cache-read-only` dinamico. |
| **Composite actions NOVA-SEMVER-26** (3 creadas, 1 descartada) | ✅ **COMPLETADO** (2026-07-09, commit `95bc786` en `nova-devops`) | Nada. `nova-validate-build`, `nova-gather-facts`, `nova-publish-aggregator` creadas con `action.yml` + `README.md`. `nova-configure-gradle-cache` **descartada** (action oficial `gradle/actions/setup-gradle@v4` es suficiente, ver §5.4.1). |
| **Remote Build Cache** (NOVA-SEMVER-23-25) | ✅ **COMPLETADO** (2026-07-09) | Nada. NOVA-SEMVER-23 + 24 + 25 implementados en 10 repos + `nova-devops`. |
| **Variable `NOVA_PACKAGE_VISIBILITY`** (NOVA-SEMVER-30) | Documentada | Configurar en cada repo via `gh variable set` o UI (backlog) |
| **Secrets de GPG** (NOVA-SEMVER-29) | Documentado, NO generados | Cuando se decida publicar a Maven Central (backlog) |
| **Namespace `pe.edu.nova` en Sonatype** | No solicitado | Crear ticket en `issues.sonatype.org` (futuro, Sprint 3) |
| **Tabla de compatibilidades** (NOVA-SEMVER-22) | Pendiente | No se ha definido el formato ni la fuente de datos (Sprint 4) |
| **ADRs en `docs/adrs/`** | ✅ **23 archivos creados** (15 Java/shared + 8 NestJS placeholders) | Pendiente: commit + push al docs repo |
| **NestJS versioning** | Fuera de alcance | Se abordara en un roadmap separado |
| **Bugs documentados en §11.7** (4 bugs + 1 limitacion) | ✅ **DOCUMENTADOS** (2026-07-09) | Pendiente: reportar limitacion de reusable + tag push a GitHub Support |

### 15.5. Roadmap visual (actualizado 2026-07-09)

| Sprint | Foco | Actividades | Estado al cierre (2026-07-09) | Proxima actividad concreta |
|---|---|---|---|---|
| **Pre-req** | Alineacion de repos (groupId, gradle.properties, OWNER) | 00a-00d | ✅ **COMPLETADO** (4/4) | — |
| **0** | Fundamentos versioning (versioning plugin, commitlint, lefthook) | 01-04 | ✅ **COMPLETADO** (4/4) | — |
| **1** | Reusable workflows faltantes en `nova-devops` | 05-08 | ✅ **COMPLETADO** (4/4) | — |
| **2** | Multi-registry publishing + GPG (preparado) | 09-12 | ✅ **COMPLETADO** (4/4) | — |
| **3** | Activacion release-please + primer release | 13-16 | 🟡 En curso (2/4) — NOVA-SEMVER-13 ✅, NOVA-SEMVER-15 ✅ | **NOVA-SEMVER-14:** namespace Sonatype (opcional, no bloquea) |
| **4** | Publicacion publica + visibilidad configurable | 17-22 | ⏳ Pendiente (0/6) | NOVA-SEMVER-17 (Maven Central, bloqueado por 29) |
| **5** | Build Cache en GitHub Actions + composite actions | 23-28 | ✅ **COMPLETADO** (6/6) — NOVA-SEMVER-23 ✅, 24 ✅, 25 ✅, 26 ✅, 27 ⏳, 28 ⏳ | NOVA-SEMVER-27: migrar workflows restantes para usar las nuevas composite actions |
| **Backlog** | GPG firma + variable visibilidad | 29-30 | ⏳ Diferido (0/2) | NOVA-SEMVER-30 (configurar visibilidad default `public`) |

**Progreso total: 24/34 actividades (70.6%)**

### 15.6. Resumen ejecutivo (1 minuto de lectura, actualizado 2026-07-09)

**Que tenemos hoy (estado real verificado al 2026-07-09):**
- 19 repos publicos en GitHub (15 Java + 4 NestJS) con descripciones y topics.
- **17 reusable workflows funcionales en `nova-devops`** (8 originales + 3 Sprint 1 + 6 Sprint 2): build, test, publish multi-registry, version bump, SonarCloud, commitlint, release-please, changelog.
- **3 composite actions en `nova-devops`**: `nova-setup-java`, `nova-setup-node`, `nova-setup-gpg` (Sprint 1, NOVA-SEMVER-08).
- **10 modulos Gradle con `maven-publish`** (incluyendo 2 submodulos de commons-starter) + 3 Maven (`nova-bom`, `parent`, `archetype`) + 2 sin build (`nova-devops`, `nova-infrastructure`).
- **9 repos Gradle con signing plugin** (`id("signing")` + `useInMemoryPgpKeys`) listo para Maven Central (Sprint 2, NOVA-SEMVER-10). Se excluye `example` que no publica.
- **Todos los repos Java migrados a `pe.edu.nova`** (13 Java: 10 Gradle + 3 Maven, ~200 archivos modificados).
- **Todos los repos Gradle tienen `gradle.properties`** con `version` y `group`.
- **Todos los repos Gradle usan `net.nemerosa.versioning` 4.0.1**.
- **Todos los repos Java tienen `commitlint.config.js` + `lefthook.yml`**.
- **3 repos migrados de Maven a Gradle**: mask-utils, observability-utils, starter (cumplen la convencion Gradle-first).
- **23 ADRs creados** en `docs/adrs/` (15 shared/java + 8 NestJS placeholders), pendientes de commit + push.

**Que se completo en Pre-req + Sprint 0 + Sprint 1 + Sprint 2 + Sprint 3 parcial + Sprint 5 parcial (4 + 4 + 4 + 4 + 2 + 4 = 24 actividades):**
- ✅ **NOVA-SEMVER-00a:** `gradle.properties` en 10 repos Gradle.
- ✅ **NOVA-SEMVER-00b:** `groupId` migrado de `pe.edu.galaxy.training` a `pe.edu.nova` (13 repos).
- ✅ **NOVA-SEMVER-00c:** placeholder `OWNER` corregido a `ahincho` (10 repos con publishing).
- ✅ **NOVA-SEMVER-00d:** packages Java renombrados (~200 archivos).
- ✅ **NOVA-SEMVER-01:** Conventional Commits adoptados (15 Java + 4 NestJS).
- ✅ **NOVA-SEMVER-02:** `commitlint` + `lefthook` configurados (15 repos Java). Auto-instalacion via npm `prepare` script (NOVA-SEMVER-02 v2, 2026-07-09).
- ✅ **NOVA-SEMVER-03:** `net.nemerosa.versioning` 4.0.1 agregado a 10 repos Gradle.
- ✅ **NOVA-SEMVER-04:** versiones movidas a `gradle.properties` con `${version}` reference.
- ✅ **NOVA-SEMVER-05:** `reusable-commitlint.yml` creado en `nova-devops`.
- ✅ **NOVA-SEMVER-06:** `reusable-release-please.yml` creado en `nova-devops`.
- ✅ **NOVA-SEMVER-07:** `reusable-changelog.yml` creado en `nova-devops`.
- ✅ **NOVA-SEMVER-08:** 3 composite actions creadas: `nova-setup-java`, `nova-setup-node`, `nova-setup-gpg`.
- ✅ **NOVA-SEMVER-09:** `reusable-publish-{gradle,maven}-multi-registry.yml` creados.
- ✅ **NOVA-SEMVER-10:** `reusable-publish-{gradle,maven}-maven-central.yml` + signing plugin en 9 repos.
- ✅ **NOVA-SEMVER-11:** `reusable-publish-{gradle,maven}-nexus.yml` creados.
- ✅ **NOVA-SEMVER-12:** workflows de publish con GPG opcional (skip si no hay secrets).
- ✅ **NOVA-SEMVER-13:** `.release-please-config.json` + `.release-please-manifest.json` en 10 repos Gradle. Nuevo `reusable-release-publish.yml` en `nova-devops`. Patron `version-bump` + `publish` deprecado (causa del bug "No jobs were run" resuelto).
- ✅ **NOVA-SEMVER-15 (2026-07-09):** Primer release end-to-end en `api-standard v1.0.0`. Run `29024268916` (success). 4 bugs descubiertos y arreglados (documentados en §11.7):
  - `id-token: write` permission removido de reusable (causaba startup_failure).
  - `default_workflow_permissions: write` + `can_approve_pull_request_reviews: true` configurado en api-standard.
  - `chmod +x ./gradlew` agregado a 8 `publish-on-tag.yml` (fix para CRLF de Windows).
  - `dirty = { it }` removido de 11 `build.gradle.kts` (incompatible con Kotlin DSL en Gradle 9.6.1).
  - **Limitacion conocida de GitHub Actions**: reusable workflow + tag push da 0 jobs. **Workaround**: logica inlined en los 8 `publish-on-tag.yml`. Pendiente reportar a GitHub Support.
- ✅ **NOVA-SEMVER-23 (2026-07-09):** `org.gradle.caching=true` agregado a 10 `gradle.properties`. Habilita Local Build Cache (reutiliza outputs de tasks entre builds en la misma maquina).
- ✅ **NOVA-SEMVER-24 (2026-07-09):** `org.gradle.configuration-cache=true` agregado a 10 `gradle.properties`. Habilita Configuration Cache (separa configuration de execution, hasta 50% menos tiempo en CI).
- ✅ **NOVA-SEMVER-25 (2026-07-09):** `gradle/actions/setup-gradle@v4` agregado a `reusable-build-gradle.yml`. Habilita Remote Build Cache via GitHub Actions Cache (compartido entre runners y developers).
- ✅ **NOVA-SEMVER-26 (2026-07-09):** 3 composite actions nuevas creadas: `nova-validate-build` (valida Java version, secrets, gradle.properties, lefthook), `nova-gather-facts` (recolecta version, branch, SHA, is-snapshot, is-tag), `nova-publish-aggregator` (dispatch por registry). `nova-configure-gradle-cache` **descartada** (action oficial `gradle/actions/setup-gradle@v4` cumple la misma funcion, ver §5.4.1).

**Que falta (Sprint 3 parcialmente + Sprint 4 + 5 parcialmente + 2 backlog, 10 actividades):**
1. **Sprint 3 (NOVA-SEMVER-14, 16, 2/4 pendientes):** Crear namespace `pe.edu.nova` en Sonatype, publicar a GitHub Packages y verificar consumo desde `nova-bom`. NOVA-SEMVER-13 ✅, NOVA-SEMVER-15 ✅.
2. **Sprint 4 (NOVA-SEMVER-17-22):** Publicar a Maven Central (bloqueado por NOVA-SEMVER-29), build matrix Java 21+25, OWASP, SBOM, matriz compatibilidades.
3. **Sprint 5 (NOVA-SEMVER-27, 28, 2/6 pendientes):** Migrar workflows restantes para usar las nuevas composite actions (27) + medir tiempos de CI antes/despues y documentar mejora (28).
4. **Backlog (NOVA-SEMVER-29-30):** Generar claves GPG (cuando se decida Maven Central) + variable `NOVA_PACKAGE_VISIBILITY`.

**Que esta documentado pero no implementado (intencional):**
- Firma GPG (guia completa en 10.3, pero clave no generada).
- Publicacion a Maven Central (workflows propuestos, namespace no creado).
- Tabla de compatibilidades entre versiones (formato no definido).
- 8 ADRs NestJS como placeholders (se redactaran cuando el stack NestJS entre en alcance).
- Roadmap de versioning para NestJS (fuera de alcance de este documento).

---

## 15.7. Estado verificado al 2026-07-09 (snapshot)

**Fecha del snapshot:** 2026-07-09
**Metodo de verificacion:** scripts PowerShell con `Test-Path`, `Select-String`, `git ls-files` sobre los 15 repos Java locales (`D:\Galaxy\Projects\java\`).

### Inventario de los 15 repos Java

| # | Repo | Tipo | Wrapper | `gradle.properties` | `maven-publish` | `net.nemerosa.versioning` | `signing` plugin |
|---|---|---|---|---|---|---|---|
| 1 | `nova-devops` | No-build (CI/CD) | — | — | — | — | — |
| 2 | `nova-infrastructure` | No-build (IaC) | — | — | — | — | — |
| 3 | `nova-bom` | Maven (BOM) | — | — | — | — | — |
| 4 | `nova-java-spring-boot-parent` | Maven (Parent POM) | — | — | — | — | — |
| 5 | `nova-java-spring-boot-archetype` | Maven (Archetype) | — | — | — | — | — |
| 6 | `nova-java-api-standard` | Gradle | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 7 | `nova-java-commons-spring-boot-starter` | Gradle (multi-modulo: root + 2 submodules) | 9.2.0 | ✅ | ❌ root / ✅ 2 submodules | ✅ | ❌ root / ✅ 2 submodules |
| 8 | `nova-java-date-utils` | Gradle | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 9 | `nova-java-example` | Gradle | 9.2.0 | ✅ | ❌ (esperado) | ✅ | ❌ (no publica) |
| 10 | `nova-java-spring-boot-gradle-plugin` | Gradle | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 11 | `nova-java-mapper-utils` | Gradle | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 12 | `nova-java-mask-utils` | Gradle (migrado de Maven) | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 13 | `nova-java-observability-spring-boot-starter` | Gradle | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 14 | `nova-java-observability-utils` | Gradle (migrado de Maven) | 9.2.0 | ✅ | ✅ | ✅ | ✅ |
| 15 | `nova-java-spring-boot-starter` | Gradle (migrado de Maven) | 9.2.0 | ✅ | ✅ | ✅ | ✅ |

**Resumen:** 9 repos Gradle simples + 1 multi-modulo (commons-starter con root + 2 submodules) + 3 Maven-only + 2 no-build = **15 repos Java totales**.

**Publicaciones totales:** 9 (repos simples) + 2 (submodules commons-starter) = **11 publicaciones distintas** a GitHub Packages.

### Convenciones y hooks (15/15 Java)

| Pieza | Cobertura | Detalle |
|---|---|---|
| `commitlint.config.js` | 15/15 repos Java | `@commitlint/config-conventional` |
| `lefthook.yml` | 15/15 repos Java | Hook pre-commit con `commitlint` |
| `package.json` | 15/15 repos Java | Dependencias npm para hooks |
| `.gitignore` cubre `build/`, `.idea/`, `node_modules/`, `.gradle/` | 15/15 repos Java | Algunas variaciones menores en formato (`/.gradle` vs `.gradle/`) |
| `pe.edu.nova` (sin referencias a `pe.edu.galaxy`) | 15/15 repos Java | ~200 archivos migrados |
| Gradle wrapper `9.2.0` | 10/10 repos Gradle | `gradle-wrapper.properties` consistente |
| 0 archivos `build/` o `.gradle/` tracked en git | 10/10 repos Gradle | Verificado con `git ls-files` |

### Pendiente antes de NOVA-SEMVER-14

- **ADRs (23 archivos)** en `D:\Galaxy\Projects\docs\adrs\`: `shared/` (10), `java/` (5), `nest/` (8 placeholders). Pendiente: commit + push al docs repo.
- **Test del primer release:** ejecutar el flujo end-to-end (push commits Conventional → release-please crea PR → merge → tag → publish a GitHub Packages).

### Siguiente paso

**Sprint 3 — NOVA-SEMVER-14-16:**
1. **NOVA-SEMVER-14:** Crear namespace `pe.edu.nova` en Sonatype OSSRH (ticket en `issues.sonatype.org`). Bloquea Maven Central end-to-end.
2. **NOVA-SEMVER-15:** Primer release de prueba `0.1.0` en los 10 repos Gradle via el flujo release-please → tag → publish. Configuración ya lista en los 10 repos.
3. **NOVA-SEMVER-16:** Publicar `0.1.0` a GitHub Packages y verificar consumo desde `nova-bom` (Maven-only, pendiente configurar su propio `ci.yml` + release-please).
