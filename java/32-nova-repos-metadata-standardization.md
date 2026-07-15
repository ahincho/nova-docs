# Nova Platform - Repository Metadata Standardization Plan

This file proposes standardization for 32 Nova Platform + demo repositories:
- 4 cross-cutting (bom, devops, docs, infrastructure)
- 21 nova-java-*
- 4 nova-nestjs-*
- 3 demo-*

Target language: **English** (all).
Target label scheme: **Scheme B (modern)** with `nova-*` and `framework/*` namespaces.

---

## Section 1: Repository Descriptions (32)

| # | Repo | Current Description | Proposed (English) |
|---|---|---|---|
| 1 | nova-bom | Bill of Materials (BOM) raiz del meta-framework Nova Platform. Centraliza versiones para Java, NestJS y futuros stacks. | Root Bill of Materials (BOM) for the Nova Platform meta-framework. Centralizes dependency versions for Java, NestJS and future stacks. |
| 2 | nova-devops | Workflows reutilizables de GitHub Actions para CI/CD del meta-framework Nova Platform (build, quality, publish para Maven y Gradle). | Reusable GitHub Actions workflows for CI/CD of the Nova Platform meta-framework (build, quality, publish for Maven and Gradle). |
| 3 | nova-docs | Documentacion del meta-framework Nova Platform: ADRs (shared, java, nest), guias tecnicas (semantic versioning, evaluacion de madurez, comparativa de archetypes) y scripts de automatizacion operativa. | Nova Platform meta-framework documentation: ADRs (shared, java, nest), technical guides (semantic versioning, maturity evaluation, archetype comparison) and operational automation scripts. |
| 4 | nova-infrastructure | Infraestructura como codigo (Docker Compose) del stack de observabilidad Nova Platform: OpenTelemetry Collector, Tempo, Loki, Mimir, Pyroscope y Grafana. | Infrastructure as code (Docker Compose) for the Nova Platform observability stack: OpenTelemetry Collector, Tempo, Loki, Mimir, Pyroscope and Grafana. |
| 5 | nova-java-api-standard | Libreria pura Java de estandares API: ApiResponse/ApiError, HATEOAS links, PageInfo, FilterCriteria, RateLimitInfo, HttpStatusCode y UserAgentParser. | Pure-Java library of API standards: ApiResponse/ApiError, HATEOAS links, PageInfo, FilterCriteria, RateLimitInfo, HttpStatusCode and UserAgentParser. Framework-agnostic. |
| 6 | nova-java-api-standard-quarkus-extension | Quarkus 'extension coloquial' that integrates nova-api-standard: ApiExceptionMapper + ApiObjectMapperCustomizer auto-wired. | Quarkus extension that integrates nova-api-standard: ApiExceptionMapper + ApiObjectMapperCustomizer auto-wired. |
| 7 | nova-java-commons-spring-boot-starter | Starter Spring Boot que re-exporta las libs puras de Nova (api-standard, mask-utils) como dependencias auto-configuradas para una aplicacion Spring Boot. | Spring Boot starter that re-exports Nova pure libraries (api-standard, mask-utils) as auto-configured dependencies for Spring Boot applications. |
| 8 | nova-java-date-utils | Libreria pura Java de utilidades de fechas: formateo, parseo, calculo relativo y helpers de zona horaria. Sin dependencias de Spring. | Pure-Java date utilities library: formatting, parsing, relative date calculation and timezone helpers. No Spring dependency. |
| 9 | nova-java-example | Instancia/demo del meta-framework Nova Java. Muestra uso real de las libs puras + starters de Nova. | Nova Java meta-framework instance/demo. Shows real usage of Nova pure libraries and starters. |
| 10 | nova-java-mapper-utils | Libreria pura Java de mapeo entre objetos (MapStruct-like) y helpers de conversion. Sin dependencias de Spring. | Pure-Java object mapping library (MapStruct-like) and conversion helpers. No Spring dependency. |
| 11 | nova-java-mask-utils | Libreria pura Java de enmascaramiento de datos sensibles (tarjetas de credito, emails, telefonos). Sin dependencias de Spring. | Pure-Java library for sensitive data masking (credit cards, emails, phones). No Spring dependency. |
| 12 | nova-java-notifications | Nova Notifications core library: pure-Java (no framework), framework-agnostic facade for Email/SMS/Push/Slack with Resilience4j-style retry+circuit-breaker+rate-limit. Published to GitHub Packages. | (already English, no change) |
| 13 | nova-java-notifications-micronaut-module | Micronaut 5 module for Nova Notifications. @Factory + @ConfigurationProperties under nova.notifications.* prefix; supports Micronaut AOT and Shadow JAR. | (already English, no change) |
| 14 | nova-java-notifications-quarkus-extension | Quarkus 3.33 LTS extension for Nova Notifications. CDI @Singleton + SmallRye Config @ConfigMapping under nova.notifications.* prefix; ships META-INF/jandex.idx so Quarkus build-time scan discovers beans. | (already English, no change) |
| 15 | nova-java-notifications-spring-boot-starter | Spring Boot 4.1 auto-configuration starter for Nova Notifications. Wires NotificationFacade, RestClient-based REST exposure, and @ConfigurationProperties under nova.notifications.* prefix. | (already English, no change) |
| 16 | nova-java-observability-spring-boot-starter | Starter Spring Boot de observabilidad: Four Golden Signals (latency, traffic, errors, saturation), trazas distribuidas con OpenTelemetry y auto-configuracion para Spring Boot Actuator. | Spring Boot observability starter: Four Golden Signals (latency, traffic, errors, saturation), distributed tracing with OpenTelemetry and Spring Boot Actuator auto-configuration. |
| 17 | nova-java-observability-utils | Libreria pura Java de utilidades de observabilidad: metricas, trazas y logs sin acoplamiento a Spring. Helpers para OpenTelemetry SDK. | Pure-Java observability utilities library: metrics, traces and logs without Spring coupling. OpenTelemetry SDK helpers. |
| 18 | nova-java-quarkus-archetype | Nova Platform Quarkus Maven archetype. Generates a multi-module (boot/product/shared) microservice skeleton on Java 25 + Quarkus 3.33.2.1 LTS. | (already English, no change) |
| 19 | nova-java-quarkus-example | Instancia Quarkus del meta-framework Nova Platform. Consume nova-api-standard + nova-api-api-standard-quarkus-extension (gemela de ahincho/nova-java-example, que es Spring Boot). | Quarkus instance of the Nova Platform meta-framework. Consumes nova-api-standard + nova-api-standard-quarkus-extension (twin of ahincho/nova-java-example, which is Spring Boot). FIX: removes typo "nova-api-api-standard" |
| 20 | nova-java-quarkus-parent | Parent POM for Nova Platform Quarkus microservice instances. Centralizes Java 25 + Quarkus 3.33.2.1 LTS + plugins + nova-notifications-quarkus-extension dependency. | (already English, no change) |
| 21 | nova-java-quarkus-template | Gradle template for microservice instances built on the Nova Platform meta-framework with Quarkus 3.33.x LTS. Multi-module (shared + product + boot), Java 25, wired with nova-notifications-quarkus-extension. | (already English, no change) |
| 22 | nova-java-spring-boot-archetype | Maven archetype para generar un nuevo proyecto Spring Boot con las convenciones y dependencias del meta-framework Nova Platform. | Maven archetype for generating a new Spring Boot project with Nova Platform meta-framework conventions and dependencies. |
| 23 | nova-java-spring-boot-gradle-plugin | Plugin Gradle de Nova Platform para proyectos Spring Boot: aplica convenciones de build, configura Java toolchain y Spring Boot plugin automaticamente. | Nova Platform Gradle plugin for Spring Boot projects: applies build conventions, configures Java toolchain and Spring Boot plugin automatically. |
| 24 | nova-java-spring-boot-parent | Parent POM de Maven para proyectos Spring Boot del meta-framework Nova Platform: dependencias gestionadas, plugins y propiedades centralizadas. | Parent POM for Spring Boot projects in the Nova Platform meta-framework: managed dependencies, plugins and centralized properties. |
| 25 | nova-java-spring-boot-starter | Meta-starter Spring Boot de Nova Platform: incluye todos los starters Nova (commons, observability) y configura la aplicacion para usar el meta-framework. | Nova Platform Spring Boot meta-starter: bundles all Nova starters (commons, observability) and configures the application to use the meta-framework. |
| 26 | nova-nestjs-commons | Monorepo Turborepo con paquetes NestJS comunes: nestjs-mask, nestjs-api-standard y nestjs-observability. | Turborepo monorepo of common NestJS packages: nestjs-mask, nestjs-api-standard and nestjs-observability. |
| 27 | nova-nestjs-observability-starter | Modulo dinamico NestJS de observabilidad con OpenTelemetry: Four Golden Signals, trazas distribuidas, correlacion de logs y exportadores OTLP. | Dynamic NestJS observability module with OpenTelemetry: Four Golden Signals, distributed tracing, log correlation and OTLP exporters. |
| 28 | nova-nestjs-parent | Configuracion compartida (TypeScript, ESLint, Prettier, Jest, TypeDoc) para proyectos NestJS del meta-framework Nova Platform. | Shared configuration (TypeScript, ESLint, Prettier, Jest, TypeDoc) for NestJS projects in the Nova Platform meta-framework. |
| 29 | nova-nestjs-starter | Meta-framework NestJS: factorÃ­a de arranque que re-exporta libs puras y modulos NestJS del ecosistema Nova Platform. | NestJS meta-framework: bootstrap factory that re-exports pure libraries and NestJS modules from the Nova Platform ecosystem. FIX: removes encoding bug "factorÃ­a" -> "factory" |
| 30 | demo-notifications-micronaut | Nova Notifications + Micronaut 5 demo. Controller-based example + @MicronautTest integration test that overrides the starter's NotificationConfiguration bean. | (already English, no change) |
| 31 | demo-notifications-quarkus | Nova Notifications + Quarkus 3.33 LTS demo. JAX-RS resource example with notification.sendWelcomeEmail() and JUnit5 integration test for the Quarkus extension. | (already English, no change) |
| 32 | demo-notifications-spring-boot | Nova Notifications + Spring Boot 4.1 demo. End-to-end REST example with RestClient-based service + integration test that wires the Spring Boot starter. | (already English, no change) |

### Summary of description changes
- 18 descriptions to translate to English
- 14 already in English (no change)
- 1 typo fix: `nova-api-api-standard` -> `nova-api-standard`
- 1 encoding fix: `factorÃ­a` -> `factory`

---

## Section 2: Topics (5 repos missing them)

| # | Repo | Proposed Topics |
|---|---|---|
| 1 | nova-java-api-standard-quarkus-extension | java, quarkus, quarkus-extension, nova-platform, library, framework-integration |
| 2 | nova-java-quarkus-archetype | java, maven, archetype, quarkus, nova-platform, microservice-template |
| 3 | nova-java-quarkus-example | java, quarkus, nova-platform, demo, example, microservice-instance |
| 4 | nova-java-quarkus-parent | java, maven, parent-pom, quarkus, nova-platform, microservice-parent |
| 5 | nova-java-quarkus-template | java, gradle, template, quarkus, nova-platform, microservice-template |

---

## Section 3: Label Scheme B (to apply to all 32 repos)

### Standard labels (applied to ALL repos)

**Namespace (org-wide):**
- `nova-platform` (cyan): belongs to Nova Platform

**Type (one per repo):**
- `type:library` (blue): pure library
- `type:extension` (purple): framework extension
- `type:starter` (purple): Spring Boot/Micronaut starter
- `type:module` (purple): framework module
- `type:archetype` (purple): Maven archetype
- `type:template` (purple): project template
- `type:parent-pom` (gray): parent POM
- `type:plugin` (purple): Gradle plugin
- `type:instance` (blue): microservice instance
- `type:meta-starter` (purple): meta-starter that bundles other starters
- `type:demo` (yellow): demo / example
- `type:docs` (gray): documentation
- `type:infra` (gray): infrastructure as code

**Lifecycle (one per repo):**
- `lifecycle:stable` (brightgreen): production-ready
- `lifecycle:beta` (yellow): usable but evolving
- `lifecycle:experimental` (orange): under development

**Framework (one or more):**
- `framework:java-pure` (blue): pure Java, no Spring/Quarkus/etc
- `framework:spring-boot` (green): Spring Boot
- `framework:quarkus` (blue): Quarkus
- `framework:micronaut` (blue): Micronaut
- `framework:nestjs` (red): NestJS
- `framework:gradle` (gray): built with Gradle
- `framework:maven` (gray): built with Maven

**Area (one or more, repo-specific):**
- `area:api-standard` (lightblue)
- `area:notifications` (lightblue)
- `area:observability` (lightblue)
- `area:common` (lightblue)
- `area:bom` (lightblue)
- `area:devops` (lightblue)
- `area:infrastructure` (lightblue)
- `area:documentation` (lightblue)

**Priority:**
- `priority:critical` (darkred)
- `priority:high` (red)
- `priority:medium` (yellow)
- `priority:low` (lightgray)

**Status / process:**
- `breaking-change` (red)
- `security` (red)
- `dependencies` (lightgray)
- `wontfix` (white)
- `duplicate` (lightgray)
- `invalid` (yellow)
- `question` (pink)
- `good first issue` (purple)
- `help wanted` (teal)
- `autorelease: pending` (lightgray)
- `autorelease: tagged` (lightgray)

**Default GitHub labels** (already exist, no change): `bug`, `documentation`, `enhancement`.

### Area per repo (assignment)

| Repo | Type | Lifecycle | Framework | Area |
|---|---|---|---|---|
| nova-bom | parent-pom | stable | maven | bom |
| nova-devops | docs/infra | stable | none (CI tooling) | devops |
| nova-docs | docs | stable | none | documentation |
| nova-infrastructure | infra | beta | none | infrastructure |
| nova-java-api-standard | library | stable | java-pure | api-standard |
| nova-java-api-standard-quarkus-extension | extension | beta | quarkus | api-standard |
| nova-java-commons-spring-boot-starter | starter | stable | spring-boot | common, api-standard |
| nova-java-date-utils | library | stable | java-pure | common |
| nova-java-example | instance | stable | spring-boot | (none specific) |
| nova-java-mapper-utils | library | stable | java-pure | common |
| nova-java-mask-utils | library | stable | java-pure | common |
| nova-java-notifications | library | stable | java-pure, gradle | notifications |
| nova-java-notifications-micronaut-module | module | beta | micronaut, gradle | notifications |
| nova-java-notifications-quarkus-extension | extension | beta | quarkus, gradle | notifications |
| nova-java-notifications-spring-boot-starter | starter | beta | spring-boot, gradle | notifications |
| nova-java-observability-spring-boot-starter | starter | stable | spring-boot | observability |
| nova-java-observability-utils | library | stable | java-pure | observability |
| nova-java-quarkus-archetype | archetype | stable | quarkus, maven | (none specific) |
| nova-java-quarkus-example | instance | stable | quarkus | (none specific) |
| nova-java-quarkus-parent | parent-pom | stable | quarkus, maven | (none specific) |
| nova-java-quarkus-template | template | stable | quarkus, gradle | (none specific) |
| nova-java-spring-boot-archetype | archetype | stable | spring-boot, maven | (none specific) |
| nova-java-spring-boot-gradle-plugin | plugin | stable | spring-boot, gradle | (none specific) |
| nova-java-spring-boot-parent | parent-pom | stable | spring-boot, maven | (none specific) |
| nova-java-spring-boot-starter | meta-starter | stable | spring-boot | common |
| nova-nestjs-commons | library | beta | nestjs, gradle (turborepo) | common, api-standard, observability |
| nova-nestjs-observability-starter | starter | beta | nestjs | observability |
| nova-nestjs-parent | docs/config | stable | nestjs | (none specific) |
| nova-nestjs-starter | meta-starter | beta | nestjs | common |
| demo-notifications-micronaut | demo | stable | micronaut, gradle | notifications |
| demo-notifications-quarkus | demo | stable | quarkus, gradle | notifications |
| demo-notifications-spring-boot | demo | stable | spring-boot, gradle | notifications |

### Migration strategy
- Each repo currently has SOME labels (Scheme A: nova:semver, nova:docs, nova:workflow, breaking-change, priority:high, priority:medium, etc.) or Scheme B (autorelease: pending, nova-platform, nova-core, nova-starter, nova-demo, framework/spring-boot).
- Strategy: rename Scheme A labels to Scheme B (e.g., `nova:semver` -> `priority:medium` or `breaking-change`); add missing Scheme B labels; leave alone if already Scheme B.

---

## Section 4: Execution order

1. Apply descriptions via `gh repo edit --description "..."` per repo (idempotent).
2. Apply topics via `gh repo edit --add-topic ...` (replaces full list each time).
3. Apply labels:
   - For each repo, generate list of Scheme B labels (by category).
   - For Scheme A labels present, delete them after mapping.
   - For Scheme B labels already present, leave alone.
   - For new Scheme B labels, create with proper colors.

---

## Status

PENDING USER APPROVAL.