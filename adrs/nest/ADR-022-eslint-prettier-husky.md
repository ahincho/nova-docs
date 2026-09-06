# ADR-022: Linter y Formateador

## Estado

**Parcialmente aceptada.** El linter está decidido e implementado: oxlint, publicado en
`@ahincho/nova-nestjs` 0.7.0. **El formateador sigue abierto** entre Prettier y oxfmt.

**Scope:** `nest`

> El archivo conserva el nombre `ADR-022-eslint-prettier-husky.md` porque es el destino de los
> enlaces que le apuntan desde [ADR-021](ADR-021-jest-testing.md) y desde el índice. Un ADR se
> identifica por su número, no por su nombre de archivo.

## Fecha

Medición: 2026-09-04 y 2026-09-06. Linter decidido e implementado: 2026-09-06. Formateador:
pendiente.

## Contexto

Los servicios usaban **ESLint con `typescript-eslint` y Prettier**, que es lo que trae el
arquetipo de NestJS. Frontend ya migró a Biome, así que la pregunta natural era si backend
debía seguirlo. Medido en frío sobre un servicio real de 70 archivos:

| Etapa          | Hoy            | Alternativa              | Diferencia |
| -------------- | -------------- | ------------------------ | ---------- |
| lint con tipos | ESLint 14.4 s  | oxlint + tsgolint 0.75 s | 19x        |
| formato        | Prettier 2.7 s | oxfmt 0.37 s             | 7x         |

**Biome quedó descartado para backend**, y por corrección y no por velocidad:

- No tiene la familia `no-unsafe-*`. Usa inferencia de tipos propia y parcial, así que no puede
  reemplazarlas. En un BFF esas reglas son el gate que impide que un `any` viaje desde la
  respuesta de un upstream hasta el controlador sin que nadie lo note.
- Rechaza los decoradores de parámetro -`@Res()`, `@Inject()`- salvo que se active
  `unsafeParameterDecoratorsEnabled`, y eso es todo NestJS.

**oxlint sí las tiene**, porque su motor con tipos (`oxlint-tsgolint`) está construido sobre
TypeScript 7. Sobre un archivo de prueba dio **los mismos 10 hallazgos** que ESLint. Además
`typescript-eslint` declara que no soporta TypeScript 7, así que el camino actual tenía fecha de
vencimiento propia.

## Decisión

### Linter: oxlint, implementado

El toolchain publica `oxlint/oxlintrc.json` en vez del preset plano de ESLint, y ya no instala
`eslint` ni `typescript-eslint`.

```json
// .oxlintrc.json del servicio
{ "extends": ["./node_modules/@ahincho/nova-nestjs-toolchain/oxlint/oxlintrc.json"] }
```

```json
// package.json
{ "scripts": { "lint": "oxlint --type-aware" } }
```

### Formateador: sigue abierto

Entre seguir en Prettier y pasar a oxfmt. Vuelto a medir sobre el repositorio de la plataforma,
143 archivos:

|                | Tiempo | Difieren de Prettier            | Markdown             | YAML   |
| -------------- | ------ | ------------------------------- | -------------------- | ------ |
| Prettier 3.9   | 1.41 s | -                               | sí                   | sí     |
| oxfmt 0.66     | 0.75 s | **1** (un tipo unión largo)     | sí, byte a byte      | sí     |
| Biome 2.5.12   | 0.88 s | 5 (unión, arreglos JSON juntos) | **no, los ignora**   | **no** |

**Biome no formatea Markdown ni YAML**: los reporta como «rutas provistas pero ignoradas».
Comprobado sobre un `.md` deliberadamente desordenado, donde Prettier y oxfmt producen la misma
salida y Biome deja el archivo intacto. Para un repositorio cuya documentación son catorce
archivos Markdown, eso solo ya lo descarta.

La recomendación sobre la mesa es **quedarse en Prettier y pasar a oxfmt cuando llegue a 1.0**.
Hoy está en 0.66.0, beta desde el 2026-02-24, y la ganancia a este tamaño de repositorio es de
0.66 segundos. Cambiar de formateador ensucia el historial de todo archivo que toque, así que no
conviene hacerlo dos veces.

### Husky no se adopta

Un hook que comprueba tipos falla en un runner por razones que no tienen que ver con el código
-ya pasó en frontend, con `TS2307` porque los tipos de los módulos hermanos sólo existen con los
servidores de desarrollo levantados- y termina apagándose con una variable de entorno. El gate va
en CI, que es donde no se puede saltar con `--no-verify`.

## Consecuencias

### `--type-aware` no es opcional, y su ausencia no avisa

Las 23 reglas que necesitan tipos **sólo corren con `oxlint --type-aware`**, que a su vez
necesita `oxlint-tsgolint` instalado. Sin la bandera oxlint no falla ni advierte: simplemente no
las evalúa, y el reporte sale verde con la mitad del análisis sin hacer.

Es la consecuencia más peligrosa de esta decisión, porque el modo degradado es indistinguible del
modo correcto. Por eso el script se llama `oxlint --type-aware` y no `oxlint` a secas, y por eso
esconderlo detrás de un comando propio -`nova lint`- pasa de ser comodidad a ser una defensa.

### `@oxlint/migrate` descarta las reglas con tipos en silencio

La conversión automática produce una configuración que parece completa y no lo es: las 23 reglas
que importan hay que escribirlas a mano. Están listadas una por una en el preset, bajo su propio
comentario, para que se note si alguna se cae.

Es la misma familia de trampa que `biome migrate prettier` poniendo `semicolons: "asNeeded"` y
borrando todos los punto y coma. **Toda migración automática de configuración hay que diffearla.**

### `extends` de oxlint resuelve rutas, no paquetes

No se puede escribir `"extends": ["@ahincho/nova-nestjs-toolchain/oxlint"]`. La ruta va relativa
al archivo de configuración y entra a `node_modules`. Funciona de forma estable porque el
toolchain es una dependencia directa del servicio y pnpm le deja un enlace real en la raíz;
verificado instalando el paquete empaquetado, no en el monorepo.

Es una diferencia con ESLint y con Vitest, que sí resuelven especificadores de paquete, y es la
razón por la que la línea del consumidor se ve más fea de lo que uno esperaría.

### El monorepo ahora se lintea a sí mismo

Antes no lo hacía: publicaba un preset de linter que nunca corría sobre su propio código. La
primera pasada sobre 82 archivos tardó 1.1 s y encontró **siete hallazgos reales**, todos
corregidos:

- Un `Array.isArray` sobre un `readonly string[]` en `NovaConfigModule.forRoot`. Estrecha a
  `any[]`, así que la rama que parecía la segura era justo la que metía un `any` en el
  `envFilePath` que se le pasa a `@nestjs/config`. Es el que justifica el ejercicio.
- Cinco aserciones de tipo que no cambiaban nada.
- Un `async` sin `await` en un test de concurrencia, que al arreglarlo dejó a la vista que el
  `Promise.all` mezclaba una promesa con un valor sincrónico.

También hizo falta publicar `vitest/index.d.mts`: sin declaración de tipos, el `vitest.config.mjs`
de un consumidor importa un valor sin tipar y `no-unsafe-call` lo marca. Un preset sin tipos deja
de ser sólo incómodo cuando el linter con tipos lo mira.

### Positivas

- El lint deja de ser el paso lento de CI: 14.4 s a 0.75 s en un servicio, 1.1 s en el monorepo.
- Las reglas con tipos sobreviven a TypeScript 7.
- La plataforma corre sobre sí misma lo que le exige a sus consumidores.

### Negativas

- oxlint tiene menos plugins de terceros que ESLint. Hoy no usamos ninguno, pero acota el futuro.
- Cada servicio de A303 que ya tenga ESLint tiene que borrar su `eslint.config.mjs`, escribir el
  `.oxlintrc.json` y cambiar dos entradas de su `publicHoistPattern`.

## Referencias

- [ADR-021: Framework de Testing](ADR-021-jest-testing.md), la otra mitad del toolchain
- Medición sobre `A303-Nova_A303-14-backend-bff-courses`, 2026-09-04, ejecuciones en frío
- Medición del formateador sobre `ahincho/nova-nestjs`, 143 archivos, 2026-09-06
- `ahincho/nova-nestjs` 0.7.0, la implementación del linter
- `@utpxpedition/orbit`: el toolchain equivalente de frontend, sobre Biome
