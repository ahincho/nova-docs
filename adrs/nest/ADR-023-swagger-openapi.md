# ADR-023: Swagger/OpenAPI para Documentación de API

## Estado

Aceptada (implementada)
**Scope:** `nest`
Reemplaza el placeholder que estaba sin redactar.

## Fecha

2026-09-07. Publicado en `@ahincho/nova-nestjs` 0.12.0.

## Contexto

Los once servicios NestJS de A303 -siete BFF y cuatro ACL- no publican ningún contrato. Quien
quiere consumir uno lee su código, y quien quiere generar un cliente lo escribe a mano. Es la
clase de deuda que no duele hasta que hay más de un equipo del otro lado.

`@nestjs/swagger` resuelve la mitad fácil: decora, y genera. La mitad difícil es propia de esta
plataforma, y es la razón por la que esto no puede ser «agreguen el paquete en cada repo».

**El interceptor de respuesta envuelve lo que el controlador devolvió.** Un documento generado
a partir del tipo de retorno describe el método, no el cable:

```jsonc
// lo que devuelve el controlador     // lo que recibe el cliente
{ "id": "1", "name": "Calculo I" }    { "success": true, "status": 200,
                                        "data": { "id": "1", ... },
                                        "errors": [] }
```

Un cliente generado del primer documento no compila contra el servicio. Y el error no se
descubre generando: se descubre en la primera integración, que es el peor momento posible.

## Decisión

**`@nestjs/swagger` 12 como dependencia directa de `@ahincho/nova-nestjs`, con el sobre
documentado por la plataforma.**

Tres partes:

1. **`bootstrap({ openapi })`** monta el documento y la interfaz. Como CORS y como `auth`,
   **omitir la opción no publica nada**: exponer la documentación es una decisión de quien
   despliega.
2. **`ApiEnvelope(Dto)` y `ApiErrors(404)`** describen la respuesta real, sobre incluido. El
   código de error de cada fallo sale de `statusToErrorCode`, la misma función que usa el filtro
   de excepciones en ejecución, así que documento y comportamiento no se pueden separar.
3. **El requisito del token va en la raíz del documento**, no operación por operación, porque el
   guard de `NovaAuthModule` también es global. Un decorador por método invertiría el default:
   quedaría documentado como abierto todo lo que alguien olvidó anotar.

### Dependencia directa, no peer opcional

El patrón de `nestjs-pino` -declarar la forma y no importar el paquete- no sirve acá. pino se
configura con un objeto plano; Swagger necesita el runtime para generar y **necesita que los
decoradores existan** para que un DTO pueda anotarse. Un peer opcional obligaría a cada servicio
a instalarlo, que es justo lo que los tres paquetes existen para evitar.

### La documentación queda fuera del `globalPrefix`

Por defecto `useGlobalPrefix: false`. El prefijo versiona la API y la documentación no es parte
de lo versionado: heredarlo haría que pasar de `v1` a `v2` mueva el enlace que la gente tiene
guardado. Se puede invertir con una opción.

Las sondas de salud sí salen en el documento. Son rutas que el servicio atiende, y esconderlas
sería mentir por omisión.

## Consecuencias

### La imagen crece, y se paga aunque esté apagada

`@nestjs/swagger` arrastra `swagger-ui-dist`: unos megabytes de assets dentro del contenedor.
`enabled: false` no lo evita, porque la dependencia se instala igual.

Es el costo aceptado a cambio de que los decoradores funcionen sin que nadie instale nada. Si
alguna vez pesa de verdad, la salida es un build multi-stage que descarte los assets, no volver
el paquete opcional.

### `@scarf/scarf` entra al árbol, y rompe el install si nadie lo decide

`swagger-ui-dist` depende de `@scarf/scarf`, cuyo script de instalación es telemetría: le reporta
al mantenedor que alguien instaló el paquete. pnpm 10+ **aborta el install** cuando hay un script
sin decidir, así que sin esto un `pnpm install` limpio falla antes de compilar nada:

```yaml
allowBuilds:
  '@scarf/scarf': false
```

Va en el `pnpm-workspace.yaml` de la plataforma y en el que genera el schematic. Apagarlo no le
quita nada: la documentación se sirve igual.

Vale la pena notar cómo apareció. `@nestjs/terminus` ya declaraba `@nestjs/swagger` como peer
**opcional**; mientras estuvo sin cumplir, `swagger-ui-dist` no existía en el árbol. Instalarlo
lo cumplió, y con eso entró una rama entera de dependencias que nadie pidió.

### El servicio generado nace documentado

El schematic pone `openapi` en su `main.ts`, con `enabled` leído de `OPENAPI_ENABLED` y
`bearerAuth: false` -nace sin `auth`-, y su test de punta a punta pide `/docs/json` y verifica que
responde. Eso hace que un `@ApiProperty` mal puesto reviente en CI y no en la primera visita a la
interfaz.

### Lo que no resuelve

No hay generación de clientes, ni publicación del documento en ningún lado, ni validación de que
el documento no cambie de forma incompatible entre versiones. Son pasos siguientes, y ninguno
tiene sentido antes de que exista el documento.

## Referencias

- `packages/core/src/openapi/` y `packages/core/docs/openapi.md`
- ADR-012, que fija el estándar del sobre que acá se documenta
- ADR-024, por la versión 12 y su condición de ESM puro
