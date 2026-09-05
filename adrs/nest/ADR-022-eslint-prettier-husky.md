# ADR-022: Linter y Formateador

## Estado

**Propuesta** — el linter está acordado, la ejecución pendiente. El formateador sigue abierto.

**Scope:** `nest`

## Fecha

Medición: 2026-09-04. Ejecución: pendiente.

## Contexto

Hoy los servicios usan **ESLint con `typescript-eslint` y Prettier**, que es lo que trae el
arquetipo de NestJS. Frontend ya migró a Biome, así que la pregunta natural era si backend
debía seguirlo. Medido en frío sobre un servicio real de 70 archivos:

| Etapa                 | Hoy             | Alternativa               | Diferencia |
| --------------------- | --------------- | ------------------------- | ---------- |
| lint con tipos        | ESLint 14.4 s   | oxlint + tsgolint 0.75 s  | 19x        |
| formato               | Prettier 2.7 s  | oxfmt 0.37 s              | 7x         |

**Biome quedó descartado para backend**, y por corrección y no por velocidad:

- No tiene la familia `no-unsafe-*`. Usa inferencia de tipos propia y parcial, así que no puede
  reemplazarlas. En un BFF esas reglas son el gate que impide que un `any` viaje desde la
  respuesta de un upstream hasta el controlador sin que nadie lo note.
- Rechaza los decoradores de parámetro -`@Res()`, `@Inject()`- salvo que se active
  `unsafeParameterDecoratorsEnabled`, y eso es todo NestJS.

**oxlint sí las tiene**, porque su motor con tipos (`oxlint-tsgolint`) está construido sobre
TypeScript 7. Sobre un archivo de prueba dio **los mismos 10 hallazgos** que ESLint. Además
`typescript-eslint` declara que no soporta TypeScript 7, así que el camino actual tiene fecha de
vencimiento propia.

## Decisión

**oxlint como linter.** Acordado; falta escribir el preset y aplicarlo.

**El formateador queda abierto** entre seguir en Prettier y pasar a oxfmt. Difieren en 2 de 71
archivos, y en los dos casos son tipos unión largos; Biome formatea igual que oxfmt. La
diferencia es real pero pequeña, y cambiar de formateador ensucia el historial de todo archivo
que toque.

**Husky no se adopta.** Un hook que comprueba tipos falla en un runner por razones que no tienen
que ver con el código -ya pasó en frontend, con `TS2307` porque los tipos de los módulos
hermanos sólo existen con los servidores de desarrollo levantados- y termina apagándose con una
variable de entorno. El gate va en CI, que es donde no se puede saltar con `--no-verify`.

## Consecuencias

### Positivas

- El lint deja de ser el paso lento de CI.
- Las reglas con tipos sobreviven a TypeScript 7.

### Negativas

- **`@oxlint/migrate` descarta las reglas con tipos en silencio.** La conversión automática
  produce una configuración que parece completa y no lo es: las 23 reglas que importan hay que
  escribirlas a mano. Es la trampa principal de esta migración.
- oxlint tiene menos plugins de terceros que ESLint. Hoy no usamos ninguno, pero acota el futuro.

## Referencias

- [ADR-021: Framework de Testing](ADR-021-jest-testing.md), la otra mitad del toolchain
- Medición sobre `A303-Nova_A303-14-backend-bff-courses`, 2026-09-04, ejecuciones en frío
- `@utpxpedition/orbit`: el toolchain equivalente de frontend, sobre Biome
