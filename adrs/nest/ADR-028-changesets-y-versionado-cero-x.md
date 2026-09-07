# ADR-028: Changesets y Versionado `0.x` para el Stack NestJS

## Estado

Aceptada (implementada)
**Scope:** `nest`
**Relación con [ADR-007](../shared/ADR-007-release-please-para-automatizacion.md):** lo acota.
release-please queda como decisión del stack Java; NestJS nunca lo usó.
**Relación con [ADR-018](../versioning/ADR-018-politica-de-versioning-y-bump.md):** es el ADR
que ese documento anunció y que no se había escrito.

## Fecha

2026-09-07

## Contexto

ADR-006 (Conventional Commits), ADR-007 (release-please) y ADR-018 (política de bump) se
escribieron en julio, cuando el único stack vivo era Java. El stack NestJS entró en alcance el
2026-09-05 y desde su primera publicación usa **Changesets** y numeración **`0.x`**. Ninguna de
las dos cosas estaba escrita en ningún lado.

El resultado es que hoy hay dos ADRs aceptados que describen mal lo que existe:

1. **ADR-007 declara alcance `shared` (Java + NestJS)** y detalla un flujo -release-please abre
   un PR de release, al merge crea el tag `vX.Y.Z` y un GitHub Release- que en
   `ahincho/nova-nestjs` no ocurre nunca.
2. **ADR-018 fija `1.0.0` como primera versión de todo repositorio** y admite `0.x` sólo como
   excepción que requiere aprobación explícita. La plataforma NestJS va por la 0.13.0, y no como
   excepción tramitada sino como decisión de fondo.

Un ADR aceptado que describe mal la realidad es peor que su ausencia: se lee como norma. Alguien
que entre al repositorio de NestJS buscando el flujo de release encuentra release-please
documentado y nada de eso en el código.

## Decisión

**El stack NestJS versiona y publica con Changesets, en `0.x`, con disparador humano.** El stack
Java no se toca: sigue con release-please y con ADR-018.

### Por qué Changesets y no release-please en NestJS

**Tres paquetes con una sola versión.** ADR-025 los ata con el grupo `fixed` de changesets:
`@ahincho/nova-nestjs`, `-toolchain` y `-schematics` suben juntos siempre. release-please en
monorepo hace lo contrario -deriva el bump de cada paquete de los paths que el commit tocó-, que
es exactamente lo que ADR-025 decidió no tener.

**La nota del cambio la escribe una persona, no el asunto del commit.** Un changeset es un
archivo en prosa, revisado en el mismo PR que el cambio. Es la diferencia entre `feat: add docker
command` y poder explicar en tres párrafos que la CLI de schematics enciende el modo dry-run sola
y sale con código 0 sin haber escrito nada. Los changesets de esta plataforma llevan ese tipo de
contenido, y no cabe en 72 caracteres.

**Publicar es irreversible y el disparador es una persona.** El workflow de release corre por
`workflow_dispatch`. En GitHub Packages una versión no se sobrescribe ni se borra sin romper a
quien ya la instaló; que el último paso sea alguien decidiendo, y no un merge, es deliberado.

**Java se queda donde está.** Ahí el bump sale del commit, no hay grupo fijo y
`net.nemerosa.versioning` ya está atado a release-please. Cambiarlo sería costo sin beneficio.

**Conventional Commits (ADR-006) sigue aplicando a los dos stacks.** En NestJS gobierna el
mensaje del commit; lo que no gobierna es el bump, que sale del changeset.

### Por qué `0.x` y no `1.0.0`

ADR-018 razona sobre librerías Java que nacen con una API que se considera pública. La plataforma
NestJS lleva veinte versiones publicadas en cinco semanas, y varias rompieron a propósito:

| Versión | Cambio incompatible                                |
| ------- | -------------------------------------------------- |
| 0.2.0   | los once paquetes colapsaron a tres (ADR-025)      |
| 0.6.0   | Jest salió, entró Vitest (ADR-021)                 |
| 0.7.0   | ESLint salió, entró oxlint (ADR-022)               |
| 0.8.1   | NestJS 11 subió a 12 (ADR-024)                     |
| 0.11.0  | `main.ts` dejó de copiarse al servicio generado    |

Con `1.0.0` de entrada, eso son cinco majors en cinco semanas. Un número que sube así no comunica
estabilidad: comunica que el número no significa nada, que es peor que no tenerlo.

**El compromiso es concreto: `1.0.0` cuando un servicio de A303 esté en producción sobre la
plataforma.** Ese es el momento en que la API deja de moverse sola, porque a partir de ahí hay
alguien a quien romper.

### La mecánica, que tiene una trampa

```bash
pnpm changeset          # escribe la nota del cambio, viaja en el PR que lo trae
pnpm version-packages   # changeset version: aplica los bumps y el changelog
                        # ESTO SE COMMITEA
pnpm release            # changeset publish: publica lo que dicen los manifiestos
```

**`changeset publish` no consume changesets: publica las versiones que los manifiestos ya
llevan.** Si `changeset version` no corrió y no se commiteó, un `workflow_dispatch` de release
termina en verde y no sube absolutamente nada. Ocurre sin un solo mensaje de error, y es la forma
más fácil que hay de creer que se publicó algo que no existe.

## Consecuencias

### Positivas

- Los tres paquetes suben juntos: ningún consumidor resuelve una matriz de compatibilidad que
  nadie escribió.
- El `CHANGELOG` se lee como prosa y explica el porqué, no sólo el qué.
- Nada se publica por un merge accidental.
- Los dos ADRs que describían mal el stack quedan acotados en lugar de contradichos en silencio.

### Negativas

- **Dos herramientas de release en el meta-framework, una por stack.** Quien trabaje en los dos
  tiene que recordar cuál. Es el precio de no forzar una sola sobre dos ecosistemas que resuelven
  el problema de forma distinta.
- **Un release puede salir en verde sin publicar nada**, por la trampa de arriba. Se mitiga
  verificando que el commit de `version-packages` esté en la rama antes de disparar.
- **En `0.x`, el caret no cruza el minor.** `^0.13.0` resuelve `>=0.13.0 <0.14.0`, así que un
  consumidor anclado con `^` **no** recibe la 0.14.0: hay que subirla a mano en cada servicio.
  Es protección y fricción a la vez, y explica por qué el servicio de ejemplo quedó atrasado dos
  veces seguidas sin que nada fallara. A partir de `1.0.0` el caret vuelve a arrastrar minors y
  el problema desaparece solo.
- **`0.x` avisa que un minor puede romper, pero no lo impide.** El README de la plataforma lo
  dice; nada mecánico lo verifica.

## Referencias

- [ADR-006: Conventional Commits y Semantic Versioning](../shared/ADR-006-conventional-commits-y-semantic-versioning.md)
- [ADR-007: release-please para Automatizacion de Releases](../shared/ADR-007-release-please-para-automatizacion.md)
- [ADR-018: Politica de Versioning y Bump](../versioning/ADR-018-politica-de-versioning-y-bump.md)
- [ADR-025: Tres Paquetes NestJS en Lugar de Once](ADR-025-tres-paquetes-en-lugar-de-once.md)
- `ahincho/nova-nestjs`, `README.md`, secciones "Versionado" y "Estado"
- `ahincho/nova-nestjs`, `.changeset/config.json`: el grupo `fixed` con los tres paquetes
