# ADR-016: Node.js 24 como Versión Objetivo

## Estado

Aceptada
**Scope:** `nest`
Reemplaza el placeholder que apuntaba a Node.js 22 LTS.
**Revisada el 2026-09-06**, cuando [ADR-021](ADR-021-jest-testing.md) se cerró en Vitest: el
concern abierto que tenía al pie quedó resuelto y el piso bajó de `>=24.9` a `>=24`.

## Fecha

2026-09-05. Revisión: 2026-09-06.

## Contexto

El placeholder anterior apuntaba a Node 22 LTS por la razón habitual: es la versión conservadora
y la que corría en los servicios existentes. Al implementar las sondas de salud sobre
`@nestjs/terminus` apareció una restricción que decidió el asunto sola.

**terminus 12 se publica sólo como ESM.** Declara `"type": "module"` y no trae build CommonJS.
Los tres consumos que importan se comportaban distinto:

| Quién lo carga          | Node 22                                 | Node 24.9+                               |
| ----------------------- | --------------------------------------- | ---------------------------------------- |
| el servicio, en runtime | funciona desde 22.12 con `require(esm)` | funciona                                 |
| `tsc`, al compilar      | funciona                                | funciona                                 |
| **Jest, en los tests**  | **falla**                               | funciona con `--experimental-vm-modules` |

El fallo de Jest es el que cerraba la puerta: `Must use import to load ES Module`. La primera
ejecución de CI en Node 22 lo mostró. Node 24.9 es la primera versión desde la que `require(esm)`
funciona dentro del entorno de módulos de Jest con esa bandera.

Quedarse en terminus 11, que sí es CommonJS, tampoco servía: no tiene la API
`attempt().withTimeout()` sobre la que está escrito el adaptador de chequeos, que es lo que le
pone fecha límite a cada indicador.

Los servicios de destino ya corren Node 24 en sus imágenes, así que subir el piso no obligaba a
mover nada desplegado.

### Lo que cambió el 2026-09-06

**Jest ya no está.** [ADR-021](ADR-021-jest-testing.md) se cerró en Vitest, que es ESM nativo, y
la fila que decidía esta tabla desapareció junto con la bandera. Eso obliga a rehacer la
pregunta desde cero: si Jest no lo pide, **¿quién pide Node 24?**

Los pisos que declaran las dependencias hoy, leídos de sus `package.json` instalados y no de la
documentación:

| Paquete           | `engines.node`                          |
| ----------------- | --------------------------------------- |
| `vitest` 5.0.0    | `^22.12.0 \|\| ^24.0.0 \|\| >=26.0.0`   |
| `@nestjs/terminus` 12 | `^20.19.0 \|\| ^22.12.0 \|\| >=24.0.0` |
| `eslint` 10       | `^20.19.0 \|\| ^22.13.0 \|\| >=24`      |
| `@nestjs/core` 11 | `>= 20`                                 |
| NestJS 12         | 20.19+ o 22.12+                         |

**Ninguno llega a 24.9.** El `.9` era exclusivamente de Jest. Dentro de la línea 24, el piso real
que imponen las dependencias es **24.0.0**.

Nada de esto empuja a bajar de línea. Node 24 se sostiene solo por tres razones que no dependen
del runner: es LTS, es lo que corren las imágenes de los servicios de destino, y es lo que corre
A303 hoy (24.18.0). Hay además un dato que lo refuerza: **`vitest` 5 excluye la línea 25**
(`^24.0.0 || >=26.0.0`), así que el próximo salto no es a la siguiente versión sino a la
siguiente par.

## Decisión

**Node 24 como versión objetivo, con `engines: ">=24"`** en los paquetes de la plataforma y en
los servicios que la consumen. CI y el workflow de publicación corren en Node 24.

El `>=24.9` original **no era cosmético: era exactamente el piso que Jest necesitaba**, y por eso
estaba en `engines`, para que avisara la instalación en vez de fallar los tests con un error que
no nombra la causa. Retirado Jest, ese número perdió su referente y se relaja a `>=24`, que es lo
que las dependencias piden de verdad.

Un `engines` cuyo motivo ya no existe es peor que uno flojo: el siguiente que lo lea va a suponer
que hay una razón y no la va a encontrar.

## Consecuencias

### Positivas

- El runtime, el compilador y el runner de tests coinciden en una sola versión.
- `require(esm)` deja de ser un problema para cualquier dependencia futura que se publique sólo
  como ESM, que es hacia donde va el ecosistema. NestJS 12 ya publica su núcleo así.
- Node 24 es LTS, así que no es una apuesta por una versión de línea impar.
- **El `engines` vuelve a decir la verdad.** Cada número que quede ahí tiene que poder
  justificarse contra el `engines` de alguna dependencia.

### Negativas

- Un servicio que por el motivo que sea siga en Node 22 no puede correr la suite de la
  plataforma, aunque sí puede ejecutarla en producción. Esto **ya no es una restricción técnica
  sino una elección**: Vitest y NestJS 12 funcionan desde 22.12, y bajar el piso a
  `^22.12 || >=24` es posible el día que algún servicio lo necesite. No se hace hoy porque
  ninguno lo pide y una matriz de dos líneas de Node cuesta mantenerla.

### Concern abierto — resuelto

> *Original, del 2026-09-05:* «Esta decisión es una consecuencia del runner, no una preferencia
> de plataforma. Si el stack pasa a Vitest, que es ESM nativo, la bandera desaparece y el piso
> podría bajar.»

Pasó exactamente eso. La bandera desapareció y el piso bajó de `>=24.9` a `>=24`. Lo que queda
de esta decisión ya no depende de ADR-021: **Node 24 se sostiene por sí solo.**

## Referencias

- [ADR-021: Framework de Testing](ADR-021-jest-testing.md), de la que ésta dependía hasta el
  2026-09-06
- `ahincho/nova-nestjs`, PR #3: salud sobre terminus, donde apareció la restricción
- `nodejs/node`, `require(esm)`: sin bandera desde 22.12; utilizable por Jest desde 24.9
- `engines` leídos del árbol instalado de `ahincho/nova-nestjs`, 2026-09-06
