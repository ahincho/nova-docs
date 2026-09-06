# ADR-021: Framework de Testing

## Estado

Aceptada (implementada)
**Scope:** `nest`
Reemplaza a Jest, que era lo que traía el arquetipo de NestJS.

> El archivo conserva el nombre `ADR-021-jest-testing.md` porque es el destino de los enlaces
> que le apuntan desde [ADR-016](ADR-016-node-version-objetivo.md), desde
> [ADR-022](ADR-022-eslint-prettier-husky.md) y desde el índice. Un ADR se identifica por su
> número, no por su nombre de archivo.

## Fecha

Medición: 2026-09-04 y 2026-09-06. Decisión e implementación: 2026-09-06, publicada en
`@ahincho/nova-nestjs` 0.6.0.

## Contexto

La plataforma y los servicios corrían **Jest con `ts-jest`**, que es lo que trae el arquetipo de
NestJS. Dos cosas obligaron a revisarlo.

**La primera es ESM, y es la que aprieta.** `@nestjs/terminus` 12 se publica sólo como módulo
ESM. Jest sólo lo carga con `--experimental-vm-modules` y Node 24.9 o superior, y esa bandera
termina en el script `test` de **cada servicio consumidor**.

**NestJS 12, publicado el 2026-08-28, publica su núcleo como ESM.** Eso convierte la bandera de
un parche para una dependencia en un parche para el framework entero, y es lo que volvió urgente
una decisión que parecía cómoda de postergar.

**La segunda es costo de ejecución.** Medido en frío sobre un servicio real de 70 archivos
TypeScript y 82 tests:

| Runner             | Tiempo |
| ------------------ | ------ |
| Jest + `ts-jest`   | 17.3 s |
| Jest + `@swc/jest` | 6.4 s  |
| Vitest             | 3.8 s  |
| rstest             | 3.6 s  |

Esa tabla exagera la diferencia, y conviene decirlo. Vuelta a medir sobre el propio `core` de la
plataforma, 282 tests en 27 archivos, la velocidad se empareja y **la diferencia real está en
memoria**:

| Escenario                                          | Jest + `ts-jest`        | Vitest              |
| -------------------------------------------------- | ----------------------- | ------------------- |
| tests, caché tibia                                 | 4.3-4.9 s / 1690 MB     | 4.1-4.6 s / 1030 MB |
| tests, caché fría                                  | 5.1 s / 1783 MB         | 4.1 s / 1030 MB     |
| **con cobertura, en frío, que es lo que corre CI** | **7.6-8.4 s / 2300 MB** | **4.7 s / 1050 MB** |

Los 4.5x del primer servicio venían de `ts-jest` transformando un árbol de fuentes más grande,
no de una ventaja general del runner. En una suite chica los dos tardan lo mismo y Vitest usa
alrededor de 55 % menos memoria.

## Opciones

### A. Seguir en Jest con la bandera

Es lo que estaba implementado. Funcionaba y no costaba migración.

- El script de test de cada consumidor deja de ser `jest` y pasa a invocar Jest a través de
  `node` con una bandera marcada como experimental.
- El piso de Node queda clavado en 24.9 por una razón que no es del producto.
- Cada dependencia nueva que se publique sólo como ESM vuelve a plantear lo mismo, y NestJS 12
  ya lo planteó.

### B. Jest con `@swc/jest`

Baja el tiempo a 6.4 s y mantiene la API de Jest, así que la migración es cambiar el
`transform` del preset.

- **No resuelve el problema de ESM**, que es el que decide.
- Suma una herramienta más al toolchain para un problema que la opción C elimina.

### C. Vitest

ESM nativo: la bandera y el piso de Node dejan de tener motivo.

- La API de aserciones es compatible casi por completo, pero no del todo: los dobles y los
  temporizadores cambian de nombre.
- Sale del arquetipo oficial de NestJS para proyectos CommonJS, así que quien busque
  documentación de Nest la va a encontrar escrita para Jest. Con un matiz: **los proyectos ESM
  de NestJS 12 traen Vitest por defecto**, así que la divergencia se está cerrando desde el otro
  lado.

### D. rstest

El más rápido y el que ya usa el equipo de frontend, lo que valdría por consistencia.

- Pasó los 82 tests **sólo con tres ajustes simultáneos**: `source.decorators.version: 'legacy'`,
  `tools.rspack.externalsType: 'commonjs'` y una expresión regular en `output.externals` para
  los especificadores desnudos. Sin ellos: `joi.string is not a function` y `Unexpected token @`.
- Es ESM primero, y los BFF y ACL son CommonJS.
- Es el proyecto más joven de los cuatro.

## Decisión

**C: Vitest**, implementado y publicado en 0.6.0.

El toolchain deja de traer `jest`, `ts-jest` y `@types/jest`, y el preset
`@ahincho/nova-nestjs-toolchain/jest` se reemplaza por
`@ahincho/nova-nestjs-toolchain/vitest/index.mjs`, una fábrica `novaVitestConfig(opciones)`:

```js
// vitest.config.mjs
import { novaVitestConfig } from '@ahincho/nova-nestjs-toolchain/vitest/index.mjs';
export default novaVitestConfig();
```

El archivo va en `.mjs` y no en `.ts` porque es configuración, no código del servicio: como
`.ts` entraría al `include` del `tsconfig` y habría que declararle tipos que no aportan nada. Es
la misma decisión que ya toma `eslint.config.mjs`.

## Consecuencias

### El umbral de cobertura sigue viviendo en el preset

Es lo que estaba decidido de antes y no cambia: el 80 % vive en el preset compartido y no en
cada repositorio, para que el número signifique lo mismo en todos.

Lo que sí cambia es **quién lo calcula**. De Istanbul a v8, y los números se mueven un poco. En
`core`: sentencias 98.57 -> 98.15, ramas 92.51 -> 94.93. Sigue muy por encima del umbral, pero
no son comparables contra un histórico anterior a esta fecha.

### La migración de un servicio, exactamente

Medida sobre `core`: 48 puntos de llamada en 24 archivos, todos mecánicos, 283 de 283 en verde.
Un servicio tiene que:

- borrar su `jest.config.js` y escribir `vitest.config.mjs`;
- cambiar `"test"` a `vitest run` y `"test:cov"` a `vitest run --coverage`;
- poner `"types": ["node", "vitest/globals"]` en su `tsconfig.json`;
- reemplazar `jest` por `vitest` en su `publicHoistPattern`;
- en los specs, `jest.fn` -> `vi.fn`, y `jest.Mock` / `jest.SpyInstance` -> `Mock` /
  `MockInstance` **importados de `'vitest'`**, porque `vitest/globals` declara las funciones y
  no los tipos.

Tres trampas que sólo aparecen corriéndolo:

1. **`jest` a veces queda solo al final de una línea** con `.spyOn` en la siguiente, y una
   expresión regular de una sola línea no lo ve.
2. **`mockImplementation()` sin argumentos** es válido en Jest y error de tipos en Vitest. No
   falla el test, falla el `tsc`, así que sobrevive a una suite verde.
3. **`@types/jest` conserva el `jest.Mock<Retorno, Argumentos>` de dos parámetros**; el `Mock`
   de Vitest recibe un tipo de función, así que pasa a ser
   `Mock<(...args: Argumentos) => Retorno>`.

### La metadata de decoradores no la fija el preset del runner

Vitest 5 transpila con **Oxc**, no con esbuild, y Oxc lee el `tsconfig.json` del proyecto.
Declarar `emitDecoratorMetadata` dentro del preset del runner **no sirve**: se comprobó
poniéndolo en `false` y la metadata se siguió emitiendo. O sea que lo que sostiene la inyección
por constructor de NestJS es `tsconfig/nestjs.json`, del mismo toolchain.

Como el fallo, cuando ocurre, se lee como un token de inyección indefinido y manda a buscar al
lugar equivocado, `core` lleva un `decorator-metadata.spec.ts` de una sola aserción que lo
vigila.

### La exclusión de los `index.ts` de la cobertura se retiró

El preset de Jest excluía `**/index.ts` porque Istanbul contaba cada reexport como una función
sin cubrir, y con seis módulos reexportados eso bajaba `core` del 98 % al 74 %. Con v8 un barril
puro aporta **cero sentencias**, así que la exclusión dejó de hacer falta.

Y no era inocua: se estaba llevando por delante `feature/index.ts` en los schematics, que no es
un barril sino las reglas mismas. Lo que sí conviene saber es que **v8 sí cuenta los archivos
que ningún test importa**, siempre que `coverage.include` esté declarado.

### Negativas

- Un desarrollador que venga de la documentación de NestJS para CommonJS va a encontrar Jest.
- Los servicios de A303 que ya tengan suites en Jest tienen que pasar por la migración de
  arriba antes de adoptar la plataforma.

### Lo que esta decisión desbloquea

- **[ADR-016](ADR-016-node-version-objetivo.md) deja de depender de ésta.** La bandera
  `--experimental-vm-modules` desapareció y con ella la razón del piso `>=24.9`.
- **NestJS 12 deja de tener un bloqueo de runner.** Migrar el núcleo a ESM ya no obliga a
  arrastrar una bandera experimental.

## Referencias

- [ADR-016: Node.js 24 como Versión Objetivo](ADR-016-node-version-objetivo.md), que dependía de
  ésta
- [ADR-022: Linter y Formateador](ADR-022-eslint-prettier-husky.md), la otra mitad del toolchain
- Medición sobre `A303-Nova_A303-14-backend-bff-courses`, 2026-09-04, ejecuciones en frío
- Medición sobre `ahincho/nova-nestjs` `packages/core`, 2026-09-06, 282 tests en 27 archivos
- `ahincho/nova-nestjs` 0.6.0, la implementación
