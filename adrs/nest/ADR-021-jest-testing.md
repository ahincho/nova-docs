# ADR-021: Framework de Testing

## Estado

**Propuesta** — decisión abierta, pendiente de acordar. La evidencia está medida; falta elegir.

**Scope:** `nest`

## Fecha

Medición: 2026-09-04. Decisión: pendiente.

## Contexto

Hoy la plataforma y los servicios corren **Jest con `ts-jest`**, que es lo que trae el
arquetipo de NestJS. Dos cosas obligan a revisarlo.

**La primera es velocidad.** Medido en frío sobre un servicio real de 70 archivos TypeScript y
82 tests:

| Runner            | Tiempo |
| ----------------- | ------ |
| Jest + `ts-jest`  | 17.3 s |
| Jest + `@swc/jest` | 6.4 s |
| Vitest            | 3.8 s  |
| rstest            | 3.6 s  |

**La segunda es ESM, y es la que aprieta.** `@nestjs/terminus` 12 se publica sólo como módulo
ESM. Jest sólo lo carga con `--experimental-vm-modules` y Node 24.9 o superior, y esa bandera
termina en el script `test` de **cada servicio consumidor**. Ver
[ADR-016](ADR-016-node-version-objetivo.md), que hoy depende de esta decisión.

## Opciones

### A. Seguir en Jest con la bandera

Es lo que está implementado. Funciona y no cuesta migración.

- El script de test de cada consumidor deja de ser `jest` y pasa a invocar Jest a través de
  `node` con una bandera marcada como experimental.
- El piso de Node queda clavado en 24.9 por una razón que no es del producto.
- Cada dependencia nueva que se publique sólo como ESM vuelve a plantear lo mismo.

### B. Jest con `@swc/jest`

Baja el tiempo a 6.4 s y mantiene la API de Jest, así que la migración es cambiar el
`transform` del preset.

- **No resuelve el problema de ESM.** La transformación de `ts-jest` sobre `node_modules` ya
  falló en un archivo interno de terminus; habría que depurar si SWC lo pasa.
- Suma una herramienta más al toolchain para un problema que la opción C elimina.

### C. Vitest

ESM nativo: la bandera y el piso de Node dejan de tener motivo. Corrió los 82 tests del
servicio de prueba **sin ningún plugin específico de NestJS**, porque Oxc emite los metadatos de
decoradores.

Costo de migración, contado sobre el código real: 18 `jest.fn`, 5 `jest.Mock`, 0 `jest.mock` y
un archivo end-to-end. Es mecánico.

- La API de aserciones es compatible casi por completo, pero no del todo: los dobles y los
  temporizadores cambian de nombre.
- Sale del arquetipo oficial de NestJS, así que un desarrollador nuevo encuentra la
  documentación de Nest escrita para Jest.

### D. rstest

El más rápido y el que ya usa el equipo de frontend, lo que valdría por consistencia.

- Pasó los 82 tests **sólo con tres ajustes simultáneos**: `source.decorators.version: 'legacy'`,
  `tools.rspack.externalsType: 'commonjs'` y una expresión regular en `output.externals` para
  los especificadores desnudos. Sin ellos: `joi.string is not a function` y `Unexpected token @`.
- Es ESM primero, y los BFF y ACL son CommonJS.
- Es el proyecto más joven de los cuatro.

## Decisión

Pendiente. La recomendación sobre la mesa es **C, Vitest**, y hacerlo antes de que la bandera de
la opción A llegue a los servicios consumidores, para no tener que retirarla después de haberla
repartido.

## Consecuencias

Sin decidir. Lo que sí está decidido es que **el umbral de cobertura del 80 % vive en el preset
compartido** y no en cada repositorio, para que el número signifique lo mismo en todos. Eso se
mantiene con cualquiera de las cuatro opciones.

## Referencias

- [ADR-016: Node.js 24 como Versión Objetivo](ADR-016-node-version-objetivo.md)
- [ADR-022: Linter y Formateador](ADR-022-eslint-prettier-husky.md), la otra mitad del toolchain
- Medición sobre `A303-Nova_A303-14-backend-bff-courses`, 2026-09-04, ejecuciones en frío
