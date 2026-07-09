# ADR-015: Librerias Puras sin Dependencias de Framework

## Estado
Aceptada
**Scope:** `java` (Java stack)

## Fecha
2026-07-08

## Contexto
El meta-framework debe soportar Spring Boot, Quarkus y Micronaut. Si las librerias core (mask-utils, date-utils, etc.) importan de `org.springframework.*`, se vuelven inutilizables en Quarkus/Micronaut.

## Decision
Las librerias de **Nivel 1 deben tener ZERO dependencias de framework**:

### Regla

- No import de `org.springframework.*`
- No import de `io.quarkus.*`
- No import de `io.micronaut.*`
- No import de `jakarta.inject.*` (CDI)
- Solo Java puro + dependencias utilitarias (ej: Jackson para JSON, si es necesario)

### Enforcement automatico con ArchUnit

```java
@ArchTest
static final ArchRule no_framework_dependencies =
    noClasses()
        .should().dependOnClassesThat()
        .resideInAnyPackage(
            "org.springframework..",
            "io.quarkus..",
            "io.micronaut.."
        );
```

### Librerias Nivel 1 actuales

| Libreria | Deps externas | Framework-free |
|---|---|---|
| `mask-utils` | Zero | Si |
| `date-utils` | Zero | Si |
| `mapper-utils` | Zero | Si |
| `api-standard` | Zero | Si |
| `observability-utils` | Zero | Si |

### Librerias futuras planificadas

- `ddd-utils`: Pure-Java DDD building blocks (AggregateRoot, Value Objects, Domain Events, CQRS buses). Framework-specific implementations (CDI buses, Spring buses) van en Nivel 2.

### BOM multi-framework

El `nova-bom` tiene 3 sub-modules:
- `nova-spring-boot-bom` (activo)
- `nova-quarkus-bom` (placeholder)
- `nova-micronaut-bom` (placeholder)

Cada sub-BOM agrega las dependencias especificas del framework sobre las librerias puras.

## Consecuencias
### Positivas
- Reutilizacion real multi-framework: las mismas librerias funcionan en Spring Boot, Quarkus y Micronaut
- Testing independiente sin necesidad de levantar contenedor de aplicacion
- Menor superficie de dependencias reduce conflictos de versiones y vulnerabilidades

### Negativas
- Algunas funcionalidades requieren abstracciones adicionales (ej: logging sin SLF4J facade puede ser limitante)
- Duplicacion potencial de adaptadores en Nivel 2 para cada framework soportado
- Restriccion estricta puede forzar disenos mas complejos para evitar dependencias de framework

## Referencias
- `docs/java/01-conceptos-tecnicos-meta-frameworks.md` Seccion 2 (Nivel 1)
- `docs/java/05-adopcion-puntos-fuertes-archetype.md` Seccion 3
