# ADR-012: Estandares de Calidad y Testing

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Solo 1 de 10 modulos tiene tests (`mask-utils` con 26 archivos de test, 7/10 de madurez). El resto tiene 0 tests. No hay enforcement de coverage ni analisis estatico uniforme. Madurez general del meta-framework: 3.2/10.

## Decision
Establecer estandares de calidad obligatorios:

### Testing

`mask-utils` es la **implementacion de referencia**. Todos los modulos Nivel 1 deben replicar su estructura de tests:
- `unit/` — JUnit 5 (tests clasicos)
- `property/` — jqwik (property-based testing)
- `fuzz/` — jqwik (fuzzing)

**Object Mother pattern** para factories de test data: directorio `src/test/java/.../mother/`, naming `{Entity}Mother.java`.

Testing es el **deficit critico P0** del proyecto.

### Coverage

**JaCoCo** como herramienta de coverage. Configurado en Parent POM y Gradle Plugin.

- Minimo global: **80% line coverage** a nivel BUNDLE.
- Targets por capa (inspirado en Quarkus Hexagonal archetype):
  - Domain: 90%
  - Application: 85%
  - Infrastructure: 60%
- Build **falla** si coverage cae por debajo del umbral.

### Analisis estatico

| Herramienta | Proposito | Estado |
|---|---|---|
| Checkstyle | Estilo de codigo | Implementado (config compartida `checkstyle.xml`) |
| SpotBugs | Deteccion de bugs | Pendiente |
| PITest | Mutation testing | Implementado en 3 repos (mask-utils, date-utils, mapper-utils) |
| Error Prone | Errores en compilacion | Pendiente |

### ArchUnit para validacion de arquitectura

- Dependencia `archunit-junit5` en Parent POM.
- Test obligatorio: verificar que Nivel 1 no importa de `org.springframework.*`, `io.quarkus.*`, `io.micronaut.*`.
- Archetype-generated projects incluyen ArchUnit tests.

## Consecuencias
### Positivas
- Calidad medible y enforceable mediante umbrales de coverage
- Regresiones detectadas automaticamente en CI
- Architecture tests como documentacion ejecutable
- Estructura de tests uniforme facilita onboarding de nuevos contribuidores

### Negativas
- Costo inicial alto para subir cobertura de 0% a 80% en 9 modulos
- ArchUnit agrega tiempo de ejecucion en CI
- PITest (mutation testing) puede ser lento en modulos grandes

## Referencias
- `docs/java/02-evaluacion-madurez-java.md` Seccion 2-3
- `docs/java/03-siguientes-pasos-y-mejoras.md` Seccion P0 y 10
- `docs/java/05-adopcion-puntos-fuertes-archetype.md` Seccion 3, 9, 11
