# ADR-029: Sin Reintentos ni Corte de Circuito en el Cliente HTTP

## Estado

Aceptada
**Scope:** `nest`
**Disparador de revisión:** el primer incidente en el que un upstream inestable degrade a un BFF
de forma que un reintento habría evitado, o el primer servicio que necesite hablar con un
upstream que declare `Retry-After`.

## Fecha

2026-09-07

## Contexto

`HttpClientService` es por donde todo servicio de Nova llama a sus upstreams. Hoy hace cuatro
cosas sobre `fetch`: un timeout que siempre está puesto, las cabeceras de correlación que viajan
solas, una traducción del fallo del upstream que impide que llegue literal al cliente -502 o 504,
o `UpstreamHttpError` si quien llama pide `forwardError`-, y una línea de log que nunca lleva el
cuerpo de la respuesta.

**Lo que no hace es reintentar, ni abrir un circuito.** Eso nunca se decidió: simplemente no se
escribió, y la ausencia no está documentada en ningún lado. Un revisor que abra el módulo no
puede saber si falta o si se descartó, y esa ambigüedad es el problema que este ADR cierra.

Es una pregunta legítima. Los siete BFF de A303 existen para agregar llamadas a la capa de
orquestación, y esa capa a su vez llama a los microservicios Quarkus. Una cadena de tres saltos
es exactamente donde un reintento parece obvio.

## Decisión

**No se agregan reintentos ni corte de circuito al cliente HTTP.** Se documenta la ausencia como
deliberada, y se deja el disparador de revisión escrito arriba.

Tres razones, en orden de peso:

**Un reintento automático amplifica el incidente que pretende cubrir.** El caso en que ayuda es
el fallo transitorio aislado. El caso en que aparece de verdad es el upstream saturado, y ahí
multiplicar por dos o por tres el tráfico entrante es la diferencia entre un servicio lento y uno
caído. Sin control de concurrencia y sin presupuesto de reintentos -que es bastante más que un
bucle con backoff-, la versión ingenua empeora el peor caso.

**No se puede reintentar sin saber si la operación es idempotente, y el cliente no lo sabe.**
`HttpClientService` recibe un método y una ruta. Reintentar un `GET` suele ser seguro; reintentar
un `POST` que ya llegó y cuya respuesta se perdió duplica un efecto. Un cliente genérico que
reintenta por método está adivinando el contrato del upstream, y los upstreams de A303 no
declaran idempotencia en ningún lado.

**El lugar donde esto se resuelve mejor no es la aplicación.** Un reintento con presupuesto, un
corte de circuito y una detección de outliers son cosas que un service mesh o el propio balanceador
hacen con visión de toda la flota, no un proceso que sólo ve sus propias llamadas. Meterlo en la
librería fija una política por servicio y la vuelve invisible desde la infraestructura.

### Lo que sí queda cubierto

El fallo no se traga: un upstream caído produce un 502 y uno lento un 504, los dos con su línea de
log y su id de correlación, y `forwardError` deja que quien llama mapee la semántica del upstream
cuando la necesita. **Un servicio que hoy necesite reintentar una llamada concreta puede hacerlo
en su propio caso de uso**, donde sí sabe si esa operación es idempotente. Lo que no hay es una
política automática y global.

## Consecuencias

### Positivas

- El comportamiento del cliente es predecible: una llamada es una llamada, y lo que se ve en los
  logs del upstream es lo que el servicio pidió.
- Un incidente no se amplifica desde la capa de aplicación.
- La decisión queda escrita, que es lo que faltaba: la ausencia ya no se lee como un olvido.

### Negativas

- **Un fallo transitorio de un salto se propaga hasta el cliente.** Es el costo real y hay que
  nombrarlo: un 502 que un reintento habría evitado llega al navegador del alumno.
- **Cada servicio que necesite reintentar lo escribe por su cuenta**, y con eso se pierde la
  uniformidad que da tener una sola implementación. Si esto pasa dos o tres veces, es señal de
  que la decisión hay que revisarla.
- **Se apoya en infraestructura que hoy no existe.** El argumento del service mesh es correcto y
  no está implementado: en A303 no hay malla, así que por ahora nadie reintenta en ninguna capa.

## Referencias

- [ADR-014: Observabilidad - Four Golden Signals](../shared/ADR-014-observabilidad-four-golden-signals.md)
- [ADR-025: Tres Paquetes NestJS en Lugar de Once](ADR-025-tres-paquetes-en-lugar-de-once.md)
- `packages/core/src/http/http-client.service.ts`: el cliente, con su timeout y su traducción
- `packages/core/docs/http.md`: la documentación del módulo
