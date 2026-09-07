# ADR-017: pnpm como Package Manager

## Estado

Aceptada (implementada)
**Scope:** `nest`
Reemplaza el placeholder que estaba sin redactar.

## Fecha

2026-09-07. En uso desde el primer commit de `nova-nestjs`; se escribe ahora porque las
consecuencias dejaron de ser teóricas.

## Contexto

El meta-framework promete algo concreto: **un servicio declara tres paquetes y nada más**. Nada
de versiones de NestJS, de vitest, de oxlint ni de TypeScript en su `package.json`.

Esa promesa se puede escribir con cualquier gestor. Lo que decide si es verdad es qué pasa
cuando alguien importa algo que no declaró.

Con npm y con yarn, `node_modules` es plano: un paquete transitivo se importa sin problema. La
promesa se cumple el primer día y se erosiona sola, sin que nada avise, hasta que subir una
versión del framework rompe un servicio que dependía de un transitivo que nadie sabía que usaba.

## Decisión

**pnpm 11.24.0, fijado con `packageManager` en el `package.json` de la raíz.**

La razón es el aislamiento: en pnpm, `node_modules` sólo expone lo declarado. Un
`import { Module } from '@nestjs/common'` en un servicio que no lo declara **no compila**. La
promesa deja de ser una convención y pasa a ser una propiedad verificable.

Lo demás -workspaces, catálogos, velocidad, el store con enlaces duros- es bienvenido, pero no
es lo que decide.

## Consecuencias

### `publicHoistPattern` es la contrapartida, y hay que escribirla

Si un servicio no declara `@nestjs/common`, no lo puede importar. Pero **tiene que** importarlo:
escribe controladores. La salida es declarar qué expone la plataforma a través de sus paquetes:

```yaml
publicHoistPattern:
  - '@nestjs/*'
  - '@types/*'
  - rxjs
  - reflect-metadata
  - class-validator
  - class-transformer
  - typescript
  - vitest
  - supertest
```

La lista es sólo de lo que se **importa** o lo que resuelve el `tsconfig`. Las herramientas que
únicamente se ejecutan -oxlint, prettier, dependency-cruiser- no están: el comando `nova` las
resuelve desde el toolchain, no desde el árbol del servicio.

### El monorepo no puede ver un conflicto de peers

**Es la consecuencia más cara, y costó una versión publicada rota.**

Cada paquete de un workspace resuelve su propio árbol, así que dos dependencias incompatibles
entre paquetes distintos conviven sin molestarse. Un servicio consumidor las aplana en un solo
árbol y ahí el install corta.

Así salió la 0.8.0: `@nestjs/cli` 12 traía `chokidar` 5 y los schematics pedían Angular DevKit
20, cuyo peer es `chokidar` ^4. `pnpm peers check` sobre el monorepo pasaba en verde. Verificado
en los dos sentidos: exit 0 en el monorepo, exit 1 en el consumidor.

De ahí sale el chequeo de consumidor en CI, que empaqueta los tres paquetes y los instala en un
servicio de verdad. **No es una prueba de integración de más: es la única que ve el árbol que ve
quien nos usa.**

### `--frozen-lockfile` no revisa los peers

Y CI lo usa. Por eso el workflow corre `pnpm peers check` como paso aparte: sin él, una
combinación imposible pasa el install de CI y falla en la máquina de quien instale de cero.

### Los scripts de instalación hay que decidirlos, uno por uno

pnpm 10+ bloquea los scripts de instalación y **aborta el install** si nadie decidió qué hacer
con ellos. No es una molestia: es la defensa contra un `postinstall` que hace lo que quiere en la
máquina de quien instala.

Se decide en `allowBuilds`, y las dos decisiones tomadas van en `false`: `unrs-resolver` -binario
nativo opcional que cae a JavaScript- y `@scarf/scarf`, cuyo script es telemetría y que entró al
árbol con `@nestjs/swagger`.

### `minimumReleaseAge`, con una excepción por scope

pnpm rechaza por defecto una versión publicada hace muy poco, que es la defensa contra un
mantenedor comprometido. La política cuida de terceros, y los paquetes de la plataforma son
propios: `minimumReleaseAgeExclude: ['@ahincho/nova-*']` permite probar un parche el mismo día en
que se publica. La exclusión es por scope y no por versión, para que no haya que editarla en cada
release.

### Lo que se comparte por catálogo es casi nada

Sólo `@types/node` y `rimraf`. Las versiones de las herramientas viven en las `dependencies` del
toolchain, que es donde tienen que estar: el toolchain las instala y el comando `nova` las
ejecuta, así que ningún otro paquete las nombra.

`@types/node` sigue declarado en cada paquete porque el `types: ["node"]` del tsconfig lo resuelve
desde el paquete, y pnpm no expone un transitivo -que es, otra vez, el punto entero de esta
decisión-.

## Referencias

- `pnpm-workspace.yaml` de la plataforma y el que genera el schematic
- `.github/actions/consumer-check/` y `scripts/rewire-consumer.mjs`
- ADR-025, sobre por qué son tres paquetes y no once
