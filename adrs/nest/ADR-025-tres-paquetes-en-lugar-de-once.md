# ADR-025: Tres Paquetes NestJS en Lugar de Once

## Estado

Aceptada
**Scope:** `nest`
**Relación con [ADR-001](../shared/ADR-001-arquitectura-meta-framework-cinco-niveles.md):** lo
matiza para NestJS. Los cinco niveles siguen describiendo el diseño; lo que cambia es cuántos
artefactos publicados hacen falta para expresarlos.

## Fecha

2026-09-05

## Contexto

Hasta la versión 0.1 el stack NestJS publicaba **once paquetes**, siguiendo el ADR-001 al pie
de la letra: una librería pura, cinco conectores de NestJS, un agregador, tres presets de
herramientas y los schematics. Repartidos, además, en cuatro repositorios.

Después de un ciclo completo de uso aparecieron tres hechos que el diseño no había previsto:

1. **Nadie instalaba las piezas sueltas.** El único consumidor real, el servicio de ejemplo,
   dependía del agregador. Los cinco conectores existían como artefactos publicados que ningún
   `package.json` nombraba.
2. **Cada cambio arrastraba una cascada.** Tocar `api-standard` obligaba a publicar
   `nestjs-api-standard`, después el agregador, después el ejemplo. Tres publicaciones y tres
   revisiones de changelog para una corrección de una línea.
3. **La frontera que importaba era otra.** Los niveles separan lo que tiene distinto consumidor
   o distinto ciclo de vida. En NestJS hay exactamente dos de esas fronteras: lo que una
   aplicación importa en tiempo de ejecución, y lo que sólo corre en desarrollo.

El caso de Java es distinto y por eso el ADR-001 no se toca allá. Ahí los niveles separan
frameworks de verdad -Spring Boot, Quarkus, Micronaut-, y una librería de Nivel 1 tiene tres
consumidores que evolucionan por separado. En NestJS el framework es uno solo.

## Decisión

Publicar **tres paquetes** desde un único monorepo, con **un solo número de versión** para los
tres, como hace `@nestjs/*`.

| Paquete                             | Qué contiene                                                                                   |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `@ahincho/nova-nestjs`              | el runtime: `api-standard`, `api`, `auth`, `config`, `http`, `observability`, `health`, `bootstrap()` |
| `@ahincho/nova-nestjs-toolchain`    | los presets de desarrollo: TypeScript, linter y runner de tests                                 |
| `@ahincho/nova-nestjs-schematics`   | los generadores `feature` y `upstream`                                                          |

Los schematics quedan aparte por una razón mecánica y no de diseño: Nest CLI resuelve una
colección **por nombre de paquete**, así que no pueden vivir dentro de otro.

**Dentro de `core` cada módulo conserva su carpeta y su `index.ts`.** La frontera del ADR-001
sigue visible en el código y sigue siendo verificable: un módulo que importara de otro se ve en
la revisión. Lo que desapareció no es la separación, es el costo de publicarla.

La versión compartida se fija con el grupo `fixed` de changesets. Es deliberado: tres paquetes
que sólo se usan juntos y que versionan por separado obligan a cada consumidor a resolver una
matriz de compatibilidad que nadie escribió.

## Consecuencias

### Positivas

- Un cambio en el runtime es **una publicación**, no tres en cascada.
- Un consumidor declara una dependencia y no siete, y no tiene que averiguar cuáles combinan.
- El `CHANGELOG` de la plataforma se lee en un solo archivo.
- Los cuatro repositorios anteriores quedaron archivados, no borrados: el historial se conserva.

### Negativas

- **Un paquete que crece.** `core` reúne siete módulos y su superficie pública es grande. Se
  mitiga con un documento por módulo en `packages/core/docs/`, pero la disciplina de no mezclar
  responsabilidades ahora es una convención de carpetas y no una frontera de publicación.
- **Se paga lo que no se usa.** Un servicio que sólo quiere el sobre de respuesta instala
  también el cliente HTTP y las sondas. En NestJS el costo es de bytes en la imagen, no de
  arranque, porque los módulos que no se importan no se instancian.
- **Un `major` de cualquier módulo es un `major` de los tres paquetes.** Es el precio de la
  versión compartida y es el mismo que paga `@nestjs/*`.
- **Volver atrás cuesta.** Si algún día un módulo necesita su propio ciclo -por ejemplo, porque
  lo consuma algo que no es NestJS-, extraerlo es un cambio incompatible para todos.

## Referencias

- [ADR-001: Arquitectura de Meta-Framework en Cinco Niveles](../shared/ADR-001-arquitectura-meta-framework-cinco-niveles.md)
- `ahincho/nova-nestjs`, PR #1: el colapso, con la lista de los once paquetes anteriores
- `ahincho/nova-nestjs`, `README.md`, sección "Por qué tres y no once"
- `docs/nest/02-evaluacion-madurez-nestjs.md`: la evaluación del diseño anterior, ya histórica
