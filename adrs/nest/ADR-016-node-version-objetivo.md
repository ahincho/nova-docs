# ADR-016: Node.js 24 como Versión Objetivo

## Estado

Aceptada
**Scope:** `nest`
Reemplaza el placeholder que apuntaba a Node.js 22 LTS.

## Fecha

2026-09-05

## Contexto

El placeholder anterior apuntaba a Node 22 LTS por la razón habitual: es la versión conservadora
y la que corría en los servicios existentes. Al implementar las sondas de salud sobre
`@nestjs/terminus` apareció una restricción que decidió el asunto sola.

**terminus 12 se publica sólo como ESM.** Declara `"type": "module"` y no trae build CommonJS.
Los tres consumos que importan se comportan distinto:

| Quién lo carga          | Node 22                                 | Node 24.9+                               |
| ----------------------- | --------------------------------------- | ---------------------------------------- |
| el servicio, en runtime | funciona desde 22.12 con `require(esm)` | funciona                                 |
| `tsc`, al compilar      | funciona                                | funciona                                 |
| **Jest, en los tests**  | **falla**                               | funciona con `--experimental-vm-modules` |

El fallo de Jest es el que cierra la puerta: `Must use import to load ES Module`. La primera
ejecución de CI en Node 22 lo mostró. Node 24.9 es la primera versión desde la que `require(esm)`
funciona dentro del entorno de módulos de Jest con esa bandera.

Quedarse en terminus 11, que sí es CommonJS, tampoco servía: no tiene la API
`attempt().withTimeout()` sobre la que está escrito el adaptador de chequeos, que es lo que le
pone fecha límite a cada indicador.

Los servicios de destino ya corren Node 24 en sus imágenes, así que subir el piso no obliga a
mover nada desplegado.

## Decisión

**Node 24 como versión objetivo, con `engines: ">=24.9"`** en los paquetes de la plataforma y en
los servicios que la consumen. CI y el workflow de publicación corren en Node 24.

El `>=24.9` no es cosmético: es exactamente el piso que Jest necesita, y ponerlo en `engines`
hace que la instalación avise, en vez de que los tests fallen con un error que no nombra la
causa.

## Consecuencias

### Positivas

- El runtime, el compilador y el runner de tests coinciden en una sola versión.
- `require(esm)` deja de ser un problema para cualquier dependencia futura que se publique sólo
  como ESM, que es hacia donde va el ecosistema.
- Node 24 es LTS, así que no es una apuesta por una versión de línea impar.

### Negativas

- **La bandera `--experimental-vm-modules` viaja hasta el consumidor.** Los scripts de test de
  cada servicio dejan de ser `jest` y pasan a invocar Jest a través de `node`. Es feo y es
  provisional: existe sólo mientras el runner sea Jest.
- Un servicio que por el motivo que sea siga en Node 22 no puede correr la suite de la
  plataforma, aunque sí puede ejecutarla en producción.

### Concern abierto

Esta decisión es **una consecuencia del runner, no una preferencia de plataforma**. Si el stack
pasa a Vitest, que es ESM nativo, la bandera desaparece y el piso podría bajar. La decisión del
runner sigue abierta en [ADR-021](ADR-021-jest-testing.md); mientras no se resuelva, ésta queda
como está.

## Referencias

- [ADR-021: Framework de Testing](ADR-021-jest-testing.md), la decisión de la que ésta depende
- `ahincho/nova-nestjs`, PR #3: salud sobre terminus, donde apareció la restricción
- `nodejs/node`, `require(esm)`: sin bandera desde 22.12; utilizable por Jest desde 24.9
