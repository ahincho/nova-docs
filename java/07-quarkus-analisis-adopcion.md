# Analisis de Quarkus: adopcion como framework alternativo en Nova Platform

## 1. Contexto y motivacion

Se ha priorizado **Quarkus como framework alternativo** a Spring Boot dentro del meta-framework Nova Platform. Este documento evalua:

1. Cuanto esfuerzo implica soportar Quarkus en Nova (estado actual, gaps, plan).
2. Como generar proyectos nuevos basados en Quarkus (`code.quarkus.io` oficial vs `quarkus-hexagonal-archetype` local).
3. Cuales son las piezas compartidas con Spring Boot y cuales son framework-specific.

**Alcance:** este doc NO implementa nada. Es el analisis de nivel macro. La implementacion concreta se divide en:

- **Doc 08**: libreria pura DDD + implementaciones del Bus (Spring + Quarkus).
- **Doc 09**: estrategia de scaffolding (archetypes, code.quarkus.io, codestarts).

**Audiencia:** desarrollador que va a tomar la decision de que hacer primero.

---

## 2. Estado actual del meta-framework respecto a Quarkus

### 2.1. Lo que YA existe (deuda tecnica previa)

| Pieza | Estado | Ubicacion |
|---|---|---|
| `nova-quarkus-bom` | Existe como **placeholder vacio** publicado en `1.0.0` | `java/nova-bom/nova-quarkus-bom/pom.xml` |
| `nova-micronaut-bom` | Existe como **placeholder vacio** publicado en `1.0.0` (mismo nivel) | `java/nova-bom/nova-micronaut-bom/pom.xml` |
| Convencion de naming | Documentada: `nova-java-<rol>-quarkus-extension` | `06-semantic-versioning-en-java.md` seccion 0.1 |
| Quarkus 3.15.1 + Hexagonal archetype (legacy) | 102 archivos Java, 139 totales — disponible localmente | `examples/archetypes/java-projects/quarkus-hexagonal-archetype/` |
| Quarkus 3.37.2 + Hello World (current) | Single-module Gradle Kotlin DSL + `quarkus-rest` + `quarkus-arc` | `instances/nova-java-quarkus-example/` (instancia del meta-framework) |
| Decision Maven Central | Descartada para todo el meta-framework; GitHub Packages es el unico registry | `06-semantic-versioning-en-java.md` seccion 3.5 + 11.9.29 |
| `nova-java-api-standard` (libreria pura, framework-agnostic) | Publicada en `1.0.0` — **lista para consumirse desde Quarkus tal cual** | `java/nova-java-api-standard/` |

**Hallazgo clave sobre `nova-java-api-standard`:** las clases viven en `pe.edu.nova.java.libs.api.standard.*` y NO tienen ninguna importacion de Spring Boot, Quarkus, ni de ningun framework. Son POJOs/enums puros (`ApiResponse`, `ApiError`, `PageInfo`, `RequestContext`, `ResponseBuilder`, `UserAgentParser`, etc.).

**Conclusion:** `api-standard` es **directamente consumible** desde un proyecto Quarkus sin ningun cambio. La integracion de Quarkus con `api-standard` no requiere fork ni adapter.

### 2.2. Lo que NO existe (gaps)

| Gap | Impacto | Esfuerzo estimado |
|---|---|---|
| `nova-quarkus-bom` con contenido real (Quarkus 3.37.x + extensions) | Bloqueante para consumir Quarkus desde Nova | Bajo (1 PR, `pom.xml`) |
| `nova-java-api-standard-quarkus-extension` (extension que aporta un `ExceptionMapper` JAX-RS nativo para `ApiError`) | Necesario para que `ApiResponse`/`ApiError` se serialicen correctamente via `quarkus-rest` | Bajo (1 PR, ~50 lineas) |
| `reusable-publish-gradle.yml` necesita verificar el artefacto `*-runner.jar` | El uber-jar de Quarkus se publica con `runner` classifier; el pipeline actual espera `*.jar` | Bajo (ajustar el `publish` task) |
| Composite action `nova-setup-java` con `build-tool: gradle` | Funciona generico; Quarkus usa `id("io.quarkus")` ademas del `java` plugin | Ninguno (no requiere cambios) |
| Sin `nova-java-<rol>-quarkus-extension` repo todavia | No hay extensions Quarkus propias todavia | Bloqueante conceptual — se resuelve con el primer extension |
| Sin GitHub Actions reusable workflow especifico para Quarkus CI | Reutiliza OWASP, SBOM, release-please — **sin cambios** | Ninguno (la infra existente cubre Quarkus) |
| `quarkus-hexagonal-archetype` no consume libs Nova | Esta aislado del meta-framework (es codigo de tu compañero, no parte de Nova todavia) | Alto — discutido en doc 09 |
| `nova-java-quarkus-example` no consume libs Nova todavia | Es solo un Hello World | Bajo (adaptar para consumir `nova-bom`) |
| Sin codestart publico de Nova en Quarkus Hub | Developer externo no puede hacer `quarkus create app pe.edu.nova:something` | Bajo (roadmap, no bloqueante) |

### 2.3. Lo que SI esta hecho y se reutiliza tal cual

| Pieza | Reusable para Quarkus? | Notas |
|---|---|---|
| `reusable-owasp-check.yml` | ✅ Si | El plugin `org.owasp.dependencycheck` funciona identico |
| `reusable-sbom.yml` (CycloneDX) | ✅ Si | CycloneDX es framework-agnostic |
| `reusable-build-matrix.yml` (Java 21 + 25) | ✅ Si | Quarkus 3.33 LTS soporta Java 21; 3.37+ soporta Java 25 |
| `reusable-release-please.yml` | ✅ Si | Convention commits son framework-agnostic |
| `reusable-changelog.yml` | ✅ Si | |
| `reusable-commitlint.yml` | ✅ Si | |
| `reusable-publish-gradle.yml` | ✅ Si con ajustes menores | Necesita confirmar comportamiento del artefacto `*-runner.jar` (ver seccion 5) |
| Local Build Cache + Configuration Cache | ⚠️ Parcial | Quarkus + config cache tiene bugs reportados (workaround documentado en doc 09) |
| `release-please` con `component` + `skip-snapshot` + `last-release-sha` | ✅ Si | Patron canonico ya validado en los 9 repos Spring Boot — copiar tal cual |
| FP suppression registry (§11.9.33) | ✅ Si | Aplica a cualquier framework Java |
| `nova-devops` mirror NVD | ✅ Si | Cache compartido entre todos los repos Java |

→ La conclusion: **~80% de la infraestructura CI/CD del meta-framework es framework-agnostic y se reutiliza para Quarkus sin cambios**. Los gaps estan concentrados en ~20% que es framework-specific (BOM, ExceptionMapper JAX-RS, uber-jar packaging).

---

## 3. Analisis de Quarkus 3.x vs el meta-framework

### 3.1. Version compatibility (estado al 2026-07-14)

| Quarkus | Java minimo | Java maximo | LTS? | Compatible con Nova meta-framework? |
|---|---|---|---|---|
| 3.15 LTS (archetype local legacy) | 17 | 21 | Si (LTS) | ✅ Si — coincide con Java 21 que ya soporta Nova |
| 3.20 LTS | 17 | 21 | Si | ✅ Si |
| 3.27 LTS | 17 | 21 | Si | ✅ Si |
| 3.33 LTS (LTS actual) | 17 | 21 | Si (LTS vigente) | ✅ Si |
| **3.37.x (latest)** | 17 | 25 | No (latest) | ✅ Si — soporta Java 25, coincide con la build matrix actual de Nova |
| 4.x | N/A | N/A | N/A | **No existe todavia** (no hay plan oficial) |

**Sources:** `quarkus.io/blog/tag/lts/` y `github.com/quarkusio/quarkus/releases` (consultados 2026-07-14). La ultima release estable es **3.37.2 (released 2026-07-08)**.

**Recomendacion:** arrancar con **Quarkus 3.37.x** (la que usa la instancia `nova-java-quarkus-example`). Coincide con Java 25 que el meta-framework ya soporta en matrix build. No atarse a un LTS especifico en la primera POC — eso es para fase de estabilizacion, no para validar infra.

### 3.2. Plugin y modelo de build

| Aspecto | Spring Boot 4.x (lo que usa Nova) | Quarkus 3.x |
|---|---|---|
| Plugin Gradle | `id("org.springframework.boot")` | `id("io.quarkus")` (configura `quarkusPlatformVersion`) |
| Bootstrapping | Reflection + classpath scan en startup | **Build-time augmentation** (codigo generado al compilar) |
| Startup time JVM | ~1-2s | ~0.5-1s |
| Native build | No built-in (GraalVM separado) | **Built-in** via `./gradlew quarkusBuild --native` |
| Output artefacto JVM | `myapp-1.0.0.jar` (fat jar, ~50MB) | `myapp-1.0.0-runner.jar` (uber-jar, ~10MB optimizado) |
| Output artefacto native | N/A | `myapp-1.0.0-runner` (binario ELF sin JVM) |
| `maven-publish` | Compatible | Compatible (Quarkus genera POM correcto) |
| Configuration cache | Soporte parcial con bugs | Mejor soporte (Quarkus prioriza esto) |
| OWASP plugin | Funciona identico | Funciona identico |
| CycloneDX SBOM | Funciona identico | Funciona identico |
| Conventional commits | Funciona identico | Funciona identico |
| `release-please` | Funciona identico | Funciona identico |

**Conclusion:** el modelo de build es muy similar (Gradle + plugins). La pieza framework-specific es el `quarkus` plugin que reemplaza al de Spring Boot. Diferencia **superficial** — el resto de la cadena CI no se entera del framework.

### 3.3. Equivalencias conceptuales Spring Boot ↔ Quarkus

| Spring Boot | Quarkus | Comentario |
|---|---|---|
| `spring-boot-starter-web` | `quarkus-rest` (o `quarkus-resteasy-reactive`) | JAX-RS, NO Spring MVC |
| `spring-boot-starter-actuator` | `quarkus-smallrye-health` + `quarkus-micrometer-registry-prometheus` | MicroProfile Health en vez de Actuator |
| `spring-boot-starter-test` | `quarkus-junit` | `@QuarkusTest` en vez de `@SpringBootTest` |
| `spring-boot-devtools` | `quarkus-dev` (live reload via `quarkusDev` task) | |
| `@SpringBootApplication` | `@QuarkusMain` + clase con `static void main` | Quarkus detecta el main automaticamente |
| `@RestController` | `@Path` + `@GET/POST/...` (JAX-RS) | Sin `@RequestBody`; `@QueryParam`, `@PathParam`, etc. |
| `@Service` / `@Component` | `@ApplicationScoped` / `@Singleton` (CDI) | Jakarta CDI, no Spring |
| `@Autowired` | `@Inject` (CDI) | |
| `@RestControllerAdvice` + `@ExceptionHandler` | `@Provider` + `@ServerExceptionMapper` | Patron oficial de Quarkus |
| `application.yml` / `application.properties` | `application.properties` (YAML via `quarkus-config-yaml`) | Mismo formato que Spring Boot |
| `spring.profiles.active` | `quarkus.profile` | |
| Spring Boot Starter / AutoConfiguration | **Extension Quarkus** | Mismo concepto, distinto mecanismo |

→ **Las equivalencias son a nivel conceptual, no 1:1 a nivel API**. El codigo de las apps debe ser distinto (JAX-RS vs Spring MVC, CDI vs Spring DI). Lo que el meta-framework aporta es **compartible**: las librerias puras (Nivel 1) son las mismas.

---

## 4. Glosario fundamental: Extension / Aggregator / Codestart

> **Proposito de esta seccion:** fijar vocabulario preciso para que cuando hablemos de "extension" o "starter" no haya ambiguedad. La terminologia de Quarkus tiene capas que la documentacion oficial mezcla, lo cual genera confusion.

### 4.1. Tabla maestra (las 4 categorias que importan)

| # | Concepto | Trae codigo de runtime? | Genera codigo en build-time? | Declara dependencias? | Auto-cargable? | Ejemplo real en Quarkus | Ejemplo Nova correspondiente |
|---|---|---|---|---|---|---|---|
| 1 | **BOM / Aggregator** | ❌ No | ❌ No | Solo versiones (`<dependencyManagement>`); NO las incluye transitivamente | N/A (no se "usa" como dependencia, se importa como BOM) | `io.quarkus.platform:quarkus-bom:3.37.2` | `nova-quarkus-bom`, `nova-spring-boot-bom`, `nova-bom` |
| 2 | **Extension coloquial** | ✅ Si (clases CDI + JAX-RS providers) | ❌ No | ✅ Si (`api` o `implementation`) | ✅ Si (CDI escanea el classpath + Jandex indiza en build-time) | `quarkus-arc` (CDI puro, sin `@BuildStep`), `quarkus-config-yaml` | `nova-java-api-standard-quarkus-extension` (Fase 0), `nova-java-commons-spring-boot-starter` (su equivalente Spring) |
| 3 | **Extension real (con `@BuildStep`)** | ✅ Si | ✅ Si (genera proxies, registra classes para reflection, etc.) | ✅ Si | ✅ Si (Jandex indiza + build steps ejecutan en augmentation) | `quarkus-rest` (genera proxies JAX-RS), `quarkus-hibernate-orm-panache` (genera metodos `find*`), `quarkus-smallrye-health` | (no planeado en Nova; seria overkill para serializar un `ApiError`) |
| 4 | **Codestart** | ❌ No (es solo estructura de archivos + templates Mustache) | ❌ No | Define deps en su POM interno, pero el proyecto **generado** es independiente | N/A (no es una dependencia; es un template que el scaffolder descarga) | El template "default" de `code.quarkus.io`, Quarkiverse Hub extensions | (roadmap: empaquetar `nova-java-quarkus-template/` como codestart oficial) |

### 4.2. Definicion formal de cada uno

#### 4.2.1. BOM / Aggregator

**Que es:** un archivo POM (o `<dependencies>` block en Gradle con `enforcedPlatform`) cuyo unico proposito es declarar versiones de artefactos de terceros en `<dependencyManagement>`. NO aporta codigo. NO aporta dependencias transitivas al consumidor (solo fija versiones cuando el consumidor declara esas deps).

**Como se reconoce:**
- Su `pom.xml` tiene `<packaging>pom</packaging>` y solo contiene `<dependencyManagement>` + (opcional) `<dependencies>` con `<scope>import</scope>`.
- No tiene `src/main/java`.
- Su tamano en el registry es < 5 KB (solo el POM + el `pom.properties`).

**En Quarkus oficial:** el `io.quarkus.platform:quarkus-bom` (tambien `quarkus-bom-test`, `quarkus-camel-bom`, etc.). Es la fuente de verdad para "que version de cada artefacto Quarkus existe en una release dada".

**En Nova:** `nova-quarkus-bom` (placeholder hoy), `nova-spring-boot-bom`, `nova-micronaut-bom`, `nova-bom` (raiz).

#### 4.2.2. Extension coloquial

**Que es:** un modulo Java normal que aporta **codigo de runtime** (clases CDI, JAX-RS providers, Jackson customizers, factories, etc.) pero **NO aporta build steps**. Quarkus lo descubre automaticamente cuando esta en el classpath porque CDI escanea `META-INF/services/*` y Jandex indiza las anotaciones Jakarta en build-time.

**Como se reconoce:**
- Tiene `src/main/java/` con clases que usan anotaciones CDI/JAX-RS.
- NO tiene ninguna clase que implemente `@BuildStep` (no hay `*BuildItem` producers).
- Su `pom.xml` es el de una libreria Java normal (`<packaging>jar</packaging>`).
- En el JAR compilado no hay archivos `META-INF/quarkus-extension.properties` (Quarkus las genera **automaticamente** al detectar dependencias en una app, pero la extension misma no las necesita).

**Ejemplo real de Quarkus oficial:** `quarkus-arc` (la implementacion de CDI de Quarkus). Tiene cientos de clases CDI pero cero `@BuildStep` — es codigo de runtime puro. Otro ejemplo: `quarkus-config-yaml` (parser de YAML) — aporta un `ConfigSource` que CDI descubre al startup.

**En Nova:** `nova-java-api-standard-quarkus-extension` (Fase 0). Aporta:
- `@Provider` con `@ServerExceptionMapper` para mapear excepciones a `ApiError`.
- `@Singleton` con `ObjectMapperCustomizer` (SmallRye) para configurar Jackson.
- Nada de `@BuildStep`.

#### 4.2.3. Extension real (con `@BuildStep`)

**Que es:** un modulo Java que aporta codigo de runtime **Y ademas** aporta uno o mas `@BuildStep` que ejecutan durante la fase de **augmentation** de Quarkus (entre la compilacion de Java y el empaquetado final). Los build steps pueden:
- Generar codigo fuente Java adicional (que luego se compila).
- Producir `BuildItem`s que otras extensions consumen.
- Registrar clases para reflection (necesario para native).
- Configurar el classpath del uber-jar.

**Como se reconoce:**
- Tiene `src/main/java/` con al menos una clase que contiene un metodo anotado `@BuildStep`.
- Cada `@BuildStep` declara que `BuildItem`s produce y/o consume (e.g., `@BuildStep public FeatureBuildItem feature() { ... }`).
- En el JAR compilado hay archivos `META-INF/quarkus-extension.properties` y posiblemente generated sources en `target/generated-sources/`.
- El descriptor del modulo en `pom.xml` declara `<quarkus.build.parent>` apuntando al BOM de Quarkus.

**Ejemplo real de Quarkus oficial:** `quarkus-rest`. Sus build steps escanean las clases `@Path` en build-time y **generan proxies pre-compilados** que reemplazan la reflection en runtime. Esto es la razon por la que Quarkus arranca ~50x mas rapido que Spring Boot en la primera request.

**Cuando es necesario en Nova:** **casi nunca** para nuestro caso de uso. Las piezas que tenemos (`api-standard`, futuros `ddd-utils`, `bus-quarkus`) son logica de aplicacion que no necesita generar codigo. La unica excepcion futura seria si quisieramos hacer algo tipo "Nova-style active record" (generar `find*` methods al build-time, como Panache), lo cual no esta planeado.

#### 4.2.4. Codestart

**Que es:** un **template ZIP** (estructura de archivos + Mustache templates + un descriptor `codestart.yml`) que el scaffolder de Quarkus (`quarkus create app` o `code.quarkus.io`) descarga, rellena con variables (groupId, artifactId, version, etc.), y descomprime como proyecto inicial. NO es una dependencia Maven/Gradle; es un template offline-first.

**Como se reconoce:**
- Es un directorio (o ZIP) con la estructura completa de un proyecto Quarkus listo para abrir en IDE.
- Contiene `codestart.yml` (descriptor que el scaffolder lee para saber que variables pedir al usuario).
- Contiene archivos `.tpl.qute` o `.mustache` que el scaffolder rellena con las variables.
- Si lo distribuyes via Quarkiverse Hub, debe cumplir el formato `quarkus-codestarts-...` (ver guia oficial).

**Ejemplo real de Quarkus oficial:** "Default project" en `code.quarkus.io` (el que te genera `quarkus-rest` + `quarkus-arc` + Dockerfile + tests por default). Otro ejemplo: cada Quarkiverse extension puede publicar su propio codestart (e.g., `quarkus-camel` agrega el codestart `camel-basic`).

**En Nova:** roadmap. Empaquetar `examples/archetypes/java-projects/nova-java-quarkus-template/` como codestart oficial y publicarlo en `hub.quarkiverse.io` para que developers externos puedan hacer `quarkus create app pe.edu.nova:something`.

### 4.3. Por que esta distincion importa

La confusion tipica que vemos en el equipo (y que se refleja en este hilo de chat):

> "Una extension es el equivalente de un starter de spring boot verdad? Con autoconfiguracion y demas?"

**Respuesta precisa:**

| Pregunta | Respuesta |
|---|---|
| ¿Una **extension coloquial** Quarkus es equivalente a un Spring Boot starter con `@AutoConfiguration`? | **Si, exactamente.** Mismo proposito, distinto mecanismo de auto-carga. |
| ¿Una **extension real** (con `@BuildStep`) es equivalente a un Spring Boot starter? | **No.** Es algo que Spring Boot **no tiene equivalente directo** — es codigo generado al build-time, lo cual no existe en el ecosistema Spring Boot. |
| ¿Un **aggregator/BOM** Quarkus es equivalente a un Spring Boot starter? | **No.** El BOM no aporta codigo ni dependencias; solo fija versiones. Es un `spring-boot-dependencies` (el BOM que importa `spring-boot-starter-parent`). |
| ¿Un **codestart** es equivalente a un Spring Boot starter? | **No.** Es un template ZIP que genera un proyecto nuevo, no una dependencia. Es mas cercano a un Spring Initializr (`start.spring.io`) response ZIP. |

### 4.4. ¿Cual es lo mas adecuado para nuestros casos?

#### 4.4.1. Casos presentes (lo que vamos a hacer en Sprint 4-5)

| Pieza Nova | Tipo correcto | Justificacion |
|---|---|---|
| `nova-quarkus-bom` (Fase 0, doc 07 §7) | **BOM / Aggregator** | Solo declara versiones de Quarkus 3.37.x + extensions Nova. NO aporta codigo. |
| `nova-java-api-standard-quarkus-extension` (Fase 0, doc 07 §7) | **Extension coloquial** | Aporta `@Provider` + `@ServerExceptionMapper` + `ObjectMapperCustomizer` para serializar `ApiResponse`/`ApiError`. No necesita generar codigo en build-time. |
| `nova-java-ddd-utils` (Fase 1, doc 08 §3.1) | Libreria pura (Nivel 1, NO es extension) | Codigo Java puro sin CDI ni JAX-RS. Consumible desde cualquier framework. |
| `nova-java-bus-api` (Fase 1, doc 08 §3.2) | Libreria pura (Nivel 1, NO es extension) | Solo interfaces (`CommandBus`, etc.). Consumible desde cualquier framework. |
| `nova-java-bus-spring` (Fase 1, doc 08 §3.3) | Libreria Spring (NO es extension Quarkus) | Tiene `@Service` + `@Component`. Es extension **Spring**, no Quarkus. |
| `nova-java-bus-quarkus` (Fase 1, doc 08 §3.4) | **Extension coloquial** | Tiene `@ApplicationScoped` + CDI `Instance<T>`. No genera codigo. |
| `examples/archetypes/java-projects/nova-java-quarkus-template/` (Fase 0.5, doc 09 §5) | **Codestart** (futuro) o directorio de referencia (presente) | Es un template de proyecto, no una dependencia. Roadmap: empaquetarlo como codestart oficial en Quarkiverse Hub. |
| `instances/nova-java-quarkus-example/` (Fase 0, doc 07 §7) | Instancia del meta-framework (app real que CONSUME Nova, NO se publica como dependencia) | Es la gemela Quarkus de `instances/nova-java-example/` (Spring Boot). Sirve como integration test vivo del extension `nova-java-api-standard-quarkus-extension`. |

#### 4.4.2. Casos futuros (roadmap, NO inmediato)

| Pieza futura | Tipo correcto | Cuando |
|---|---|---|
| `nova-quarkus-commons-extension` (meta-equivalente a `nova-java-commons-spring-boot-starter`) | **Extension coloquial** | Fase 1+. Bundle de deps + CDI beans reutilizables. |
| Una extension que genere `find*` methods automaticamente estilo Panache | **Extension real** (con `@BuildStep`) | **NO planeado**. Solo si en el futuro se quiere hacer active record sobre JPA. |
| Codestart oficial en `hub.quarkiverse.io` | **Codestart** | Fase 2 (opcional). Permite `quarkus create app pe.edu.nova:something`. |
| `nova-quarkus-test-utils` (utilidades para tests, e.g., `Testcontainers` wrappers) | **Extension coloquial** | Fase 1+. Aporta `@QuarkusTestResource` lifecycle beans. |
| `nova-quarkus-messaging-extension` (wrapper sobre `quarkus-smallrye-reactive-messaging`) | **Extension coloquial** | Solo si se necesita. Por ahora SmallRye funciona directo. |
| `nova-quarkus-opentelemetry-extension` (wrapper sobre `quarkus-opentelemetry`) | **Extension coloquial** | Solo si se necesita. Por ahora OTel funciona directo. |

#### 4.4.3. Regla de decision rapida (cheatsheet)

Cuando estes a punto de crear una pieza nueva para Quarkus, hazte estas preguntas:

```
1. ¿Trae codigo de runtime (clases CDI, JAX-RS providers, factories)?
   ├── NO → es un BOM/Aggregator. Naming: nova-java-<rol>-bom o nova-<framework>-bom.
   └── SI → continua.

2. ¿Necesita generar codigo fuente Java al build-time (@BuildStep)?
   ├── NO → es una Extension COLOQUIAL. Naming: nova-java-<rol>-quarkus-extension.
   └── SI → es una Extension REAL. Naming: misma convencion, pero documentar
            explicitamente que tiene build steps y por que. Effort: 3-5 dias vs 1-2 horas.

3. ¿Es un template ZIP para scaffolders (code.quarkus.io)?
   ├── NO → no es codestart; es extension o BOM.
   └── SI → es Codestart. Naming del repo: nova-java-quarkus-template.
            Distribuir via Quarkiverse Hub si quieres descubrimiento oficial.
```

### 4.5. Equivalencias con Spring Boot (resumen)

Para que quede claro cuando alguien viene del mundo Spring Boot:

| Spring Boot | Quarkus | Tipo (segun §4.1) |
|---|---|---|
| `spring-boot-dependencies` (BOM) | `io.quarkus.platform:quarkus-bom` | BOM / Aggregator |
| `spring-boot-starter-data-jpa` (sin AutoConfiguration custom) | `quarkus-hibernate-orm-panache` (sin `@BuildStep` custom) | Extension coloquial |
| `spring-boot-starter-data-jpa` con AutoConfiguration custom + Hibernate | `quarkus-hibernate-orm-panache` con `@BuildStep` que genera codigo | Extension real |
| Spring Initializr ZIP | `code.quarkus.io` ZIP / Quarkiverse codestart | Codestart |
| `@Configuration` + `@Bean` | `@ApplicationScoped` + `@Produces` | (mecanismo de extension coloquial) |
| `spring.factories` / `AutoConfiguration.imports` | `META-INF/services/*` + Jandex indiza en build-time | (mecanismo de auto-carga) |

---

## 5. Estrategia de adopcion: que pieza creamos primero

### 5.1. Respuesta a la pregunta del usuario

> "para Quarkus, como primer avance debemos por lo menos integrarlo con el api-standard"

**Confirmado y propuesto asi:**

**Paso 1 (esta sesion):** crear `nova-java-api-standard-quarkus-extension` — una **extension coloquial** (NO real con `@BuildStep`) que provee:

1. Un `Provider` con `@ServerExceptionMapper` para mapear excepciones genericas a `ApiError` (serializado como JSON estandar con la misma forma que `api-standard`).
2. Un `ObjectMapperCustomizer` (SmallRye) que registra modulos Jackson compatibles con la serializacion de `ApiResponse`/`ApiMetadata`/`PageInfo`.
3. Consumir `pe.edu.nova.java.libs:nova-java-api-standard:1.0.0` como dependencia `implementation` directa (sin cambios en `api-standard`).
4. Tests con `@QuarkusTest` que validen que el contrato HTTP de respuesta es el mismo que en Spring Boot.

**Paso 2 (siguiente sprint):** segun lo decidido en el doc 08, crear `nova-java-ddd-utils` y `nova-java-bus-{spring,quarkus}`.

**Paso 3 (siguiente sprint):** adaptar `instances/nova-java-quarkus-example` para que consuma `nova-bom` + `nova-java-api-standard-quarkus-extension` y validar la cadena CI/CD completa.

### 5.2. Que NO se hace en este paso

- NO se modifica `nova-java-api-standard` (sigue siendo framework-agnostic).
- NO se modifica el `quarkus-hexagonal-archetype` de tu compañero (es codigo aislado, lo abordamos en doc 09).
- NO se publica a Maven Central (consistente con la politica del meta-framework §3.5).
- NO se implementa native build (es opcional, Fase posterior).
- NO se crea codestart oficial en Quarkiverse Hub (roadmap, Fase 2+).
- NO se hace una extension Quarkus "real" con `@BuildStep` (overkill, no aporta valor).

---

## 6. Compatibilidad con `reusable-publish-gradle.yml`

### 6.1. Comportamiento esperado

El pipeline actual (`reusable-publish-gradle.yml` en `nova-devops`) hace:

1. `actions/checkout@v4` con `ref: main`.
2. `nova-setup-java@main` (Java 25 + Gradle wrapper).
3. `nova-validate-build@main` (verifica que compile).
4. `nova-gather-facts@main` (lee `gradle.properties` para extraer la version).
5. `nova-publish-aggregator@main` con `registry: github-packages`, `build-tool: gradle`. Internamente ejecuta `./gradlew publish` con `version-source: gradle-properties`.

### 6.2. Que pasa con Quarkus

El plugin `maven-publish` de Quarkus genera el POM correctamente y publica tanto `myapp-1.0.0.jar` (jar comun) como `myapp-1.0.0-runner.jar` (uber-jar ejecutable) en GitHub Packages.

**NO requiere cambios en `reusable-publish-gradle.yml`**. La pieza que SI requiere verificacion es `nova-publish-aggregator` (composite action) — especificamente la logica que extrae `version-source: gradle-properties`. Hay que confirmar que el `gradle.properties` del proyecto Quarkus tenga `version=...` en una sola linea (mismo formato que los 9 repos Spring Boot).

**Verificacion empirica recomendada (Fase 0):**
1. Crear `nova-java-api-standard-quarkus-extension` con `gradle.properties` que tenga `version=1.0.0`.
2. Lanzar `reusable-publish-gradle.yml` con `dry-run: true` para verificar que `./gradlew publishToMavenLocal` genera el POM + `*-runner.jar` correctamente.
3. Si falla, ajustar `nova-publish-aggregator` para que publique el `runner` artifact explicitamente.

### 6.3. Sugerencia concreta al usuario

**Mi sugerencia:** arrancar la POC reutilizando `reusable-publish-gradle.yml` tal cual. Si algo falla, ajustar `nova-publish-aggregator` (composite action) — son ~30 lineas de cambio, no requiere tocar el workflow reusable.

---

## 7. Compatibilidad con el patron `release-please`

El patron validado en los 9 repos Spring Boot (§11.9.34) tiene 3 piezas:

1. `release-please.yml` con `with:` block pasando `release-type`, `path`, `config-file`, `manifest-file`.
2. `.release-please-config.json` con `packages: { ".": { component, skip-snapshot, include-component-in-tag, ... } }` y `last-release-sha` top-level.
3. `.release-please-manifest.json` con `{ ".": "1.0.1" }`.

**Confirmado:** este patron es **framework-agnostic**. Funciona identico para Quarkus. Lo unico que cambia entre repos es:
- El `path:` (`.` para single-module, `path/submodule` para multi-module).
- El `package-name:` (e.g., `nova-java-api-standard-quarkus-extension`).
- El `release-type:` (siempre `java` para nuestros proyectos).

**Conclusion:** copiar el patron tal cual al primer repo Quarkus. Sin sorpresas.

---

## 8. Plan de adopcion por fases

### Fase 0: Validacion tecnica (1-2 dias)

**Objetivo:** confirmar que Quarkus funciona end-to-end con la infra de Nova.

| Actividad | Esfuerzo | Output |
|---|---|---|
| Crear `nova-java-api-standard-quarkus-extension` (extension coloquial para Quarkus, ver §4) | 1 dia | Extension que provee `ServerExceptionMapper` + `ObjectMapperCustomizer` |
| Configurar con la infra completa (matrix Java 21+25, OWASP, SBOM, release-please, publish-on-tag) | 0.5 dia | PR mergeado, todos los workflows verdes |
| Verificar que `release-please` + `publish-on-tag` publican a GitHub Packages | 0.25 dia | `nova-java-api-standard-quarkus-extension:1.0.0` visible |
| Adaptar `instances/nova-java-quarkus-example` para consumir `nova-bom` + la nueva extension | 0.5 dia | `nova-java-quarkus-example` arranca con Quarkus 3.37 + `nova-bom` |

**Output:** un repo Quarkus production-ready + una instancia del meta-framework (`instances/nova-java-quarkus-example` → `ahincho/nova-java-quarkus-example`) que consume `nova-bom` y demuestra que la cadena CI/CD completa funciona. **Si esto falla, no se justifica seguir.**

#### Fase 0: Estado de cierre (2026-07-14)

**Cerrada.** La implementacion concreta de Fase 0 se realizo en las siguientes piezas:

| Pieza | Repo | Version | Estado |
|---|---|---|---|
| Extension coloquial `ApiExceptionMapper` + `ApiObjectMapperCustomizer` | `ahincho/nova-java-api-standard-quarkus-extension` | 1.1.1 | ✅ Publicado en GitHub Packages |
| Tests unitarios del extension (10 tests) | mismo repo | — | ✅ Verde (10/10 passing) |
| Workflows CI del extension (build + matrix Java 21+25 + OWASP + SBOM + sonar) | mismo repo | — | ✅ Verde en PR + push a main |
| Workflow `release-please.yml` + `publish-on-tag.yml` | mismo repo | — | ✅ Funciona (release 1.1.0 y 1.1.1 publicados) |
| Instancia Quarkus del meta-framework (resource JAX-RS con `ApiResponse<Greeting>`) | `ahincho/nova-java-quarkus-example` | 0.1.0-SNAPSHOT | ✅ Repositorio creado, codigo pusheado |
| Tests `@QuarkusTest` integration test vivo | mismo repo | — | ✅ Escritos (4 tests cubriendo happy path + exception mapping + subclass + path validation) |
| Workflows CI del ejemplo | mismo repo | — | ✅ Configurados (build + matrix + owasp + sbom + sonar + quarkus-it) |
| Documentacion del ejemplo | mismo repo | — | ✅ README + SECRETS_SETUP.md |

**Bug critico encontrado y corregido durante Fase 0:**

El extension original usaba `implements ExceptionMapper<Throwable>` que en Quarkus REST (resteasy-reactive) **NO se invoca para excepciones lanzadas desde resource methods**. El `QuarkusErrorHandler` intercepta antes y devuelve un JSON default con stack trace, derrotando el proposito del mapper. La forma idiomatica de Quarkus es `@ServerExceptionMapper`, que se registra en build-time augmentation. Fix incluido en la version 1.1.1 del extension (commit `37bd083`).

**Hallazgos colaterales durante el cierre:**

1. **URL del repo en `build.gradle.kts`** (extension): apuntaba al repo del extension en GitHub Packages, pero `nova-api-standard` NO esta publicado alli — esta en su propio repo (`nova-java-api-standard`). Fix incluido en 1.1.1.

2. **`GITHUB_TOKEN` vs `NOVA_PACKAGES_READ_TOKEN`**: el `GITHUB_TOKEN` automatico NO puede leer packages de OTROS repos (solo del repo actual). Se necesita un fine-grained PAT con scope explicito `read:packages` para cross-repo reads. Workaround: `NOVA_PACKAGES_READ_TOKEN` env var leida por el `build.gradle.kts`.

3. **`reusable-build-gradle.yml` ejecuta `./gradlew checkstyleMain`**: el extension necesitaba aplicar el plugin `checkstyle` y referenciar `config/checkstyle/checkstyle.xml`. Sin esto el task fallaba con `Task 'checkstyleMain' not found`.

**Pendiente del lado del developer:**

- Configurar el secret `NOVA_PACKAGES_READ_TOKEN` en `https://github.com/ahincho/nova-java-quarkus-example/settings/secrets/actions` con scope `read:packages` sobre los repos `ahincho/nova-java-api-standard-quarkus-extension` y `ahincho/nova-java-api-standard`. Sin este secret, el workflow CI de la instancia falla con `401 Unauthorized` al resolver la dep. Ver `.github/SECRETS_SETUP.md` del repo de la instancia.

**Causa raiz del publish fantasma (descubierto 2026-07-15) — REABRE FASE 0:**

Las versiones `1.1.0`, `1.1.1`, etc. del extension se publicaron como **paquetes fantasma**: la API REST de GitHub Packages muestra el package + la version, pero el `maven.pkg.github.com` retorna `404 Not Found` para `.jar`/`.pom`/`.module` al hacer GET. Solo `maven-metadata.xml` se actualiza. **El plugin `maven-publish` de Gradle reporta `BUILD SUCCESSFUL` aunque ningún archivo de artefacto se persiste.**

Diagnostico (orden de descartes):

1. **No es la version de Gradle (9.5.1):** el repo `ahincho/nova-java-api-standard` (lib pura) usa Gradle 9.5.1 desde local y publica artefactos completos sin problema (`.pom` 794b, `.jar` 45323b, `.module` 1992b).
2. **No es el content-type:** el plugin `maven-publish` envia `.jar` y `.pom` con `Content-Type: application/octet-stream`. PUTs manuales con `application/octet-stream` al mismo package con los mismos archivos (4462 bytes JAR + 1593 bytes POM) se persisten correctamente (200 OK + GET 200 con el tamano exacto). Es decir, **el server acepta el body, el problema es del lado del cliente**.
3. **No es el NTLM de Windows ni Basic Auth forzado:** el log de Gradle muestra `Using Credentials ... NTLM`, pero al forzar Basic Auth via `JAVA_TOOL_OPTIONS=-Dhttp.auth.preference=Basic` el resultado es identico (200 OK al hacer PUT pero archivos no persistidos).
4. **No es el estado stale del package:** tras borrar las 11 ghost versions y el package entero (auto-delete al borrar la ultima version), republicar `v2.0.0` desde cero reproduce el mismo fantasma.
5. **No es la longitud del artifactId per se:** PUTs manuales a artifactIds de 16 a 46 caracteres funcionan. **Pero el PUT de Gradle solo se persiste para artifactIds cortos.** El artifactId original `nova-java-api-standard-quarkus-extension` (39 chars) es rechazado por el cliente Gradle cuando combina con GH Packages Maven, mientras que PUTs manuales al mismo path con los mismos archivos si funcionan.

**Conclusion:** bug de la combinacion GitHub Packages Maven + plugin `maven-publish` de Gradle, sensible a la longitud del artifactId. **Workaround aplicado:** renombrar el artifactId a `nova-quarkus-api-ext` (20 chars). El repo de GitHub sigue siendo `ahincho/nova-java-api-standard-quarkus-extension` (solo cambia el artifactId, no la URL del repo). El path Maven completo queda: `pe/edu/nova/java/starters/nova-quarkus-api-ext/{version}/...`.

**Acciones aplicadas:**

- `settings.gradle.kts` del extension: `rootProject.name = "nova-quarkus-api-ext"`.
- `.release-please-config.json`: `component` y `package-name` actualizados.
- `README.md` del extension: titulo, tabla de metadata, nota de naming y ejemplo de dependencia actualizados.
- `build.gradle.kts` de la instancia: URL del repo Maven sigue igual (es el GH repo, no el artifactId), pero la coordenada Maven ahora es `pe.edu.nova.java.starters:nova-quarkus-api-ext:1.0.0`.
- README de la instancia: titulo y referencia al extension actualizados.
- Limpiados 27 packages basura generados durante el diagnostico (nombres `nova-quarkus-ext`, `nova-api-std-quarkus-ext`, etc.) via API DELETE.

**Pendiente:** tag `v1.0.0` en el repo del extension (con el nuevo artifactId) y verificacion end-to-end via el workflow `quarkus-it` de la instancia.

Una vez configurado, el workflow `quarkus-it` (job custom en `.github/workflows/ci.yml`) ejecuta los 4 tests `@QuarkusTest` y valida end-to-end que:
- `GET /hello` retorna 200 con `ApiResponse<Greeting>` JSON
- `GET /hello/error` retorna 400 con `ApiResponse` conteniendo `ApiError(code=BAD_REQUEST)`
- `GET /hello/World` retorna 200 con saludo personalizado
- `Instant` en `ApiMetadata.generatedAt` se serializa como ISO-8601 (confirma que `ApiObjectMapperCustomizer` funciona)

### Fase 1: Adoptar DDD + Bus (doc 08, 1-2 semanas)

Documentada en detalle en el **doc 08**. Resumen: `nova-java-ddd-utils` (lib pura Nivel 1), `nova-java-bus-api` (lib pura Nivel 1), `nova-java-bus-spring` (Nivel 2), `nova-java-bus-quarkus` (extension coloquial Nivel 2).

### Fase 2: Scaffolding oficial (opcional, doc 09, 1 semana)

Documentada en detalle en el **doc 09**. Resumen: fork del `quarkus-hexagonal-archetype` de tu compañero a `nova-java-quarkus-template`, integrar con CI de Nova, roadmap de codestart oficial en Quarkiverse Hub.

### Fase 3: Migracion de proyectos existentes (ongoing, no scope definido)

Por proyecto, segun necesidad. Native build es opcional y se aborda caso por caso.

---

## 9. Riesgos identificados

| Riesgo | Probabilidad | Impacto | Mitigacion |
|---|---|---|---|
| Quarkus native build requiere GraalVM (~2GB en CI) | Baja (no se hace en Fase 0) | Alto | Hacer native build opcional, no en cada PR |
| Quarkus 3.x cambia API entre minor versions | Baja | Alto | Pin a 3.37.x al inicio, migrar a 3.x LTS cuando se estabilice |
| `code.quarkus.io` no soporta codestarts custom privados | Alta | Bajo | Roadmap: publicar en Quarkiverse Hub (publico) o self-host |
| Quarkus build-time augmentation no es compatible con Configuration Cache de Gradle en algunos casos | Media | Medio | Documentar workaround (`org.gradle.configuration-cache=false` solo para Quarkus) |
| Spring Boot y Quarkus tienen dependencias conflictivas si se mezclan | Baja | Alto | Documentar: cada repo es UN framework (regla del meta-framework §0) |
| OWASP data accuracy issues especificos de Quarkus ecosystem | Baja | Bajo | Reutilizar el FP registry §11.9.33 |
| Quarkus uber-jar `*-runner.jar` no se publica correctamente via `maven-publish` Gradle | Baja | Medio | Verificar en Fase 0 con `dry-run: true`; ajustar `nova-publish-aggregator` si falla |
| Confundir "extension coloquial" con "extension real" y hacer `@BuildStep` innecesariamente | Media | Bajo | Documentar la diferencia (ver §4); preguntar "¿necesita generar codigo?" antes de implementar |
| Quarkus 4.x no anunciado todavia | N/A | N/A | No aplica — usar 3.37.x latest estable |

---

## 10. Recomendacion al usuario

**Arrancar la Fase 0 esta semana con `nova-java-api-standard-quarkus-extension`** — es la pieza minima viable para demostrar que Quarkus se integra nativamente con las libs Nova sin tocar `api-standard`.

**Orden de trabajo propuesto:**

1. **Hoy:** crear repo `nova-java-api-standard-quarkus-extension` con la extension coloquial (ExceptionMapper + ObjectMapperCustomizer para `ApiResponse`/`ApiError`).
2. **Manana:** configurar con la infra completa (matrix, OWASP, SBOM, release-please, publish-on-tag).
3. **Pasado manana:** mergear PR, validar release end-to-end, confirmar `1.0.0` visible en GitHub Packages.
4. **Mismo dia:** adaptar `instances/nova-java-quarkus-example` para que consuma `nova-bom` + la nueva extension. Validar que arranca con `mvn quarkus:dev` o `./gradlew quarkusDev`.
5. **Siguiente sprint:** Fase 1 (DDD + Bus, doc 08) y Fase 2 (scaffolding, doc 09) en paralelo.

**Si la Fase 0 falla:** reconsiderar. No invertir en Fase 1/2 hasta entender que fallo.

---

## 11. Preguntas que quedan abiertas (revisar despues de Fase 0)

1. **Quarkus LTS vs latest para produccion:** ¿empezamos con 3.37.x y migramos a 3.x LTS cuando se estabilice, o esperamos al proximo LTS? (asume respuesta: 3.37.x hasta Fase 1, evaluar LTS en estabilizacion).
2. **Native build:** ¿es requisito o nice-to-have? (asume respuesta: nice-to-have por ahora, no en Fase 0/1).
3. **Codestart oficial:** ¿publicar en Quarkiverse Hub o self-host? (asume respuesta: postergar a Fase 2+).
4. **Naming del extension:** confirmado `nova-java-api-standard-quarkus-extension` (extension, NO starter — alineado con terminologia Quarkus, ver §4).
5. **Bus implementations:** detalle en doc 08.

---

## 12. Referencias oficiales consultadas

- https://quarkus.io/guides/writing-extensions — guia oficial para escribir extensions Quarkus (incluye seccion sobre build steps).
- https://quarkus.io/blog/tag/lts/ — releases LTS.
- https://github.com/quarkusio/quarkus/releases — releases completas (3.37.2 es la ultima al 2026-07-14).
- https://quarkus.io/guides/writing-extensions#buildstep — patron `@BuildStep`.
- https://quarkus.io/guides/cdi-reference — referencia CDI (SmallRye).
- https://quarkus.io/guides/rest — guia de `quarkus-rest` (JAX-RS).
- https://hub.quarkiverse.io — registry de extensions de comunidad (donde se publican codestarts).