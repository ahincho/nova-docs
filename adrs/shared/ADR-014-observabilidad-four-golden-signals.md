# ADR-014: Observabilidad con Four Golden Signals

## Estado
Aceptada
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Un meta-framework empresarial necesita observabilidad transparente integrada. Los 4 Golden Signals (latency, traffic, errors, saturation) son el estandar de Google SRE.

## Decision
Implementar observabilidad como ciudadano de primera clase:

### Stack

| Pilar | Herramienta | Integracion |
|---|---|---|
| Metricas | Micrometer + Prometheus | Four Golden Signals via `GoldenSignalsFilter` |
| Tracing distribuido | OpenTelemetry SDK | Bridge `micrometer-tracing-bridge-otel` |
| Correlacion de logs | MDC (Mapped Diagnostic Context) | Trace ID + Span ID en cada log entry |
| Profiling continuo | Pyroscope (futuro) | Runtime profiling sin overhead significativo |
| Collector | OpenTelemetry Collector | Estandar CNCF (no Jaeger directo) |

### Implementacion en el meta-framework

- `observability-utils` (Nivel 1): utilidades puras Java de observabilidad.
- `observability-starter` (Nivel 2): auto-configuracion Spring Boot que registra `GoldenSignalsFilter`, metricas, tracing.
- Madurez actual: 5/10 (parcialmente implementado).

### Caracteristicas del `GoldenSignalsFilter`

- Mide latencia (p50, p95, p99), throughput (req/s), error rate, saturation.
- Se activa condicionalmente (`@ConditionalOnClass`, `@ConditionalOnProperty`).
- Properties configurables: `nova.observability.golden-signals.enabled=true|false`.

## Consecuencias
### Positivas
- Observabilidad out-of-the-box para cualquier proyecto que use el starter
- Compatible con el ecosistema Grafana/Prometheus sin configuracion adicional
- Estandar OpenTelemetry garantiza interoperabilidad con multiples backends
- Correlacion automatica de logs con traces reduce tiempo de debugging

### Negativas
- Dependencia en OpenTelemetry BOM agrega complejidad al arbol de dependencias
- Overhead minimo pero existente en cada request (filtro intercepta todas las peticiones)
- Pyroscope (profiling continuo) aun no implementado, stack incompleto temporalmente

## Referencias
- `docs/java/01-conceptos-tecnicos-meta-frameworks.md` Seccion 3.5
- `docs/java/04-comparativa-archetypes.md` Seccion 3.6
