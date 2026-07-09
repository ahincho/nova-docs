# ADR-013: Firma GPG Preparada y Diferida

## Estado
Propuesta (documentada, diferida)
**Scope:** `java` (Java stack)

## Fecha
2026-07-08

## Contexto
Maven Central requiere firma GPG obligatoria para publicar artefactos. GitHub Packages no lo requiere. El proyecto actualmente publica solo a GitHub Packages.

## Decision
La firma GPG esta **completamente documentada pero NO generada**:

### Que esta listo
- Guia de generacion de claves (RSA 4096, 2 anos de expiracion)
- Composite action `nova-setup-gpg` implementada (skip graceful si no hay secrets)
- Workflow `reusable-publish-maven-central.yml` disenado
- Configuracion Gradle (`signing { useInMemoryPgpKeys(...) }`) documentada
- Troubleshooting de 7 errores comunes documentado

### Que falta (NOVA-SEMVER-29, backlog)
1. Generar par de claves GPG
2. Subir clave publica a `keys.openpgp.org`
3. Configurar 3 secrets en GitHub: `GPG_SIGNING_KEY_ID`, `GPG_SIGNING_KEY`, `GPG_SIGNING_PASSWORD`
4. Registrar namespace `pe.edu.nova` en Sonatype
5. Activar workflow de publish a Maven Central

### Parametros definidos para cuando se genere
- RSA 4096 bits
- Identidad: `Nova Platform <ahincho@users.noreply.github.com>`
- Keyserver: `keys.openpgp.org`
- Expiracion: 2 anos
- Certificado de revocacion guardado en vault seguro

### Alternativa futura
Sigstore/Cosign (CNCF graduated) para firma keyless. Maven Central aun NO lo acepta.

## Consecuencias
### Positivas
- Publicacion a GitHub Packages funciona sin GPG, sin bloquear el desarrollo actual
- Documentacion lista para cuando se necesite publicar a Maven Central
- La composite action `nova-setup-gpg` ya maneja el skip graceful, evitando fallos en CI

### Negativas
- Publicacion a Maven Central bloqueada hasta ejecutar NOVA-SEMVER-29
- Riesgo de que la documentacion quede desactualizada si se difiere demasiado tiempo

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 10.3.0 hasta 10.3.9
