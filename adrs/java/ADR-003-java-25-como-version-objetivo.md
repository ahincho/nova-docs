# ADR-003: Java 25 como Version Objetivo

## Estado
Aceptada (con concern abierto)
**Scope:** `java` (Java stack)

## Fecha
2026-07-08

## Contexto
Necesitamos definir la version minima de Java para el meta-framework. Java 25 es la version mas reciente. Java 21 es la LTS vigente. La eleccion impacta directamente en que features del lenguaje pueden usarse en las librerias y en que entornos de produccion podran consumirse.

## Decision
Java 25 como target en todos los repos. Toolchain configurado en `build.gradle.kts` y Parent POM:

```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
}
```

**Concern abierto:** El doc `03-siguientes-pasos-y-mejoras.md` nota que esto es "muy restrictivo" y sugiere soportar Java 21 LTS como minimo. La tarea NOVA-SEMVER-19 (`reusable-build-matrix.yml`) propone testear contra Java 21 + 25 para validar compatibilidad.

## Consecuencias
### Positivas
- Acceso a features mas recientes del lenguaje (pattern matching, virtual threads, structured concurrency).
- Posibilidad de usar preview features para validacion temprana.

### Negativas
- Excluye consumidores que usan Java 21 LTS en produccion, limitando la adopcion.
- Requiere build matrix para validar compatibilidad con Java 21, agregando complejidad al CI.

## Referencias
- `docs/java/02-evaluacion-madurez-java.md` Seccion 4
- `docs/java/03-siguientes-pasos-y-mejoras.md` Decision Pendiente #2
