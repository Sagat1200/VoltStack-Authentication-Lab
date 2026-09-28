# EXECUTIVE_IMPLEMENTATION_PLAN

## Proposito

Este documento traduce la arquitectura del sistema Authentication a un plan ejecutivo de implementacion para el paquete:

- `vendor/voltstack/framework/src/Quantum/Auth`

Su objetivo es convertir la documentacion `00-50` en una secuencia de desarrollo operable, incremental y compatible con el estado actual del framework.

## Objetivo del primer cierre real

El primer cierre util del subsistema `Quantum/Auth` no debe intentar cubrir toda la plataforma descrita en `Docs/00-50`.

El objetivo inmediato debe ser entregar un stack minimo, seguro y usable con estas capacidades:

1. principal autenticado request-scoped,
2. identidad canonica,
3. password authentication,
4. session authentication,
5. login/logout reales,
6. `AuthenticationContext` confiable,
7. facade/helper ergonomicos,
8. integracion con runtime y Controllers Security.

## Regla de alcance

No abrir en la primera fase:

- MFA,
- passkeys,
- OIDC,
- risk engine,
- abuse protection avanzado,
- distributed session coordination,
- machine identities,
- policy engine completo.

Esos bloques dependen de un core que todavia no existe.

## Estado actual de partida

Hoy ya existe una base minima ampliada:

- `Quantum\Auth\AuthManager`
- helper `auth()`
- binding scoped en `Application.php`
- `AuthenticationManagerInterface`
- `AuthenticationOrchestratorInterface`
- `AuthenticationContext`
- `AuthenticationDecision`
- `AuthenticationContextAccessor`
- `AuthenticationOrchestrator`
- `IdentityProviderInterface`
- `LocalIdentityProvider`
- `AuthenticatorInterface`
- `PasswordAuthenticator`
- `PasswordCredentials`
- `AuthManager::attempt()`
- `AuthenticationSession`
- `AuthenticationSessionRepositoryInterface`
- `InMemoryAuthenticationSessionRepository`
- `SessionAuthenticator`
- `AuthManager::login()`
- `AuthenticatorResolverInterface`
- `DefaultAuthenticatorResolver`
- facade `Auth`
- `config/auth.php`
- `AuthenticationServiceProvider`
- `AuthenticationSessionRecoveryReason`
- `AuthenticationSessionPublicId`
- `touch()` de session para refresh de metadata
- middleware alias `auth`
- middleware alias `guest`
- middleware alias `mfa`
- `IdentitySecurityState`
- excepciones propias de Authentication
- `attemptOrFail()`
- `FileAuthenticationSessionRepository`
- `PasswordPolicy`
- upgrade persistente inicial de hash usando `storage_path`
- purga, rotacion y revocacion basica de session
- tombstones minimos de recovery para sesiones revocadas/expiradas
- inventory seguro por `session_public_id`
- metadata reducida de inventory y refresh server-side de `last_activity`
- denial explicito `auth.revoked_session`
- pruebas unitarias y feature del lenguaje base y del request scope
- pruebas del flujo minimo de password authentication
- pruebas del flujo minimo de session authentication
- pruebas de facade, resolver y expiracion minima de session
- pruebas de elegibilidad y driver file de session
- pruebas de password policy, rotation y revocation

Por tanto, el plan debe preservar compatibilidad con:

- `auth()->user()`
- `auth()->check()`
- `auth()->guest()`
- `auth()->id()`
- `auth()->logout()`

## Arquitectura minima objetivo para V1

La V1 operativa de `Quantum/Auth` deberia quedar compuesta por estos bloques:

```text
Auth Facade / Helper
        |
        v
AuthenticationManager
        |
        v
AuthenticationOrchestrator
        |
        +--> Firewall Resolver
        +--> Authenticator Resolver
        +--> Authenticator
        +--> Identity Provider
        +--> Credential Verifier
        +--> Decision Engine
        +--> Context Factory
        +--> Session Persistence
```

## Estructura recomendada de namespaces

La estructura inicial recomendada es esta:

```text
Quantum/Auth
    Contracts/
    Context/
    Identity/
    Credentials/
    Authenticators/
    Sessions/
    Decisions/
    Support/
    Runtime/
    Exceptions/
    Facades/
```

## Layout inicial sugerido

```text
Quantum/Auth
    AuthManager.php
    Contracts/
        AuthenticationManagerInterface.php
        AuthenticationOrchestratorInterface.php
        AuthenticatorInterface.php
        AuthenticatorResolverInterface.php
        IdentityProviderInterface.php
        CredentialVerifierInterface.php
        AuthenticationSessionRepositoryInterface.php
    Context/
        AuthenticationRequest.php
        AuthenticationContext.php
        AuthenticationContextAccessor.php
    Identity/
        IdentityInterface.php
        IdentityIdentifier.php
        IdentityReference.php
    Credentials/
        PasswordCredentials.php
        VerifiedCredential.php
    Authenticators/
        PasswordAuthenticator.php
        SessionAuthenticator.php
        DefaultAuthenticatorResolver.php
    Sessions/
        AuthenticationSession.php
        AuthenticationSessionId.php
        AuthenticationSessionFactory.php
        AuthenticationSessionRepository.php
    Decisions/
        AuthenticationDecision.php
        AuthenticationDecisionStatus.php
        AuthenticationContextFactory.php
    Runtime/
        AuthenticationOrchestrator.php
        AuthenticationOperationContext.php
    Exceptions/
        AuthenticationException.php
        InvalidCredentialsException.php
        AuthenticationRequiredException.php
    Facades/
        Auth.php
```

## Fases ejecutivas

## Fase 0 - Refactor de compatibilidad

### Objetivo

Preservar el `AuthManager` actual, pero convertirlo en una fachada fina sobre una infraestructura nueva.

### Entregables

1. Mantener `AuthManager` como punto de entrada publico actual.
2. Introducir `AuthenticationContextAccessor`.
3. Mover el acceso a `RuntimeContext` a una pieza dedicada.
4. Conservar compatibilidad del helper `auth()`.

### Criterio de cierre

- el test actual sigue pasando,
- el acceso a `auth.user` deja de estar embebido directamente en toda la logica futura.

## Fase 1 - Dominio base

### Objetivo

Construir el lenguaje minimo del sistema.

### Entregables

1. `IdentityInterface`
2. `IdentityIdentifier`
3. `IdentityReference`
4. `AuthenticationRequest`
5. `AuthenticationDecision`
6. `AuthenticationDecisionStatus`
7. `AuthenticationContext`

### Reglas

- `AuthenticationContext` debe ser inmutable.
- `AuthenticationDecision` no es lo mismo que `AuthenticationContext`.
- `IdentityInterface` no debe asumir email, password ni roles.

### Tests minimos

1. value objects y enums,
2. invariantes de inmutabilidad,
3. `AuthenticationContext` solo nace desde decision autenticada.

## Fase 2 - Orquestacion minima

### Objetivo

Crear el pipeline interno del sistema.

### Entregables

1. `AuthenticationManagerInterface`
2. `AuthenticationOrchestratorInterface`
3. `AuthenticationOperationContext`
4. `DefaultAuthenticationManager`
5. `AuthenticationOrchestrator`

### Operaciones minimas

- `authenticate`
- `recover`
- `logout`

### Regla de diseno

El manager coordina; los servicios especializados deciden y ejecutan.

### Tests minimos

1. flujo feliz de manager -> orchestrator,
2. propagacion de decision,
3. limpieza de contexto al terminar request.

## Fase 3 - Password authentication

### Objetivo

Entregar el primer mecanismo real de autenticacion.

### Entregables

1. `PasswordCredentials`
2. `AuthenticatorInterface`
3. `PasswordAuthenticator`
4. `IdentityProviderInterface`
5. `CredentialVerifierInterface`
6. implementacion default para verificacion de password

### Alcance minimo recomendado

- login por identificador + password,
- resolucion de identidad local,
- verificacion de password,
- decision `AUTHENTICATED` o `REJECTED`.

### Fuera de alcance en esta fase

- reset de password,
- password upgrade policy avanzada,
- throttling avanzado,
- MFA,
- step-up.

### Tests minimos

1. credencial valida autentica,
2. credencial invalida rechaza,
3. identidad inexistente rechaza sin leakage innecesario,
4. no se exponen secretos en errores.

### Estado actual del corte

Parcialmente implementada:

- existe `IdentityProviderInterface`,
- existe `LocalIdentityProvider`,
- existe `AuthenticatorInterface`,
- existe `PasswordAuthenticator`,
- existe `PasswordCredentials`,
- y `AuthManager::attempt()` ya autentica contra el provider local configurado.

Falta en esta fase:

- policy formal de password,
- ciclo de vida de credenciales,
- y endurecimiento adicional del flujo.

## Fase 4 - Session authentication

### Objetivo

Persistir y restaurar autenticacion de forma segura.

### Entregables

1. `AuthenticationSession`
2. `AuthenticationSessionId`
3. `AuthenticationSessionFactory`
4. `AuthenticationSessionRepositoryInterface`
5. `SessionAuthenticator`
6. restauracion de contexto desde session validada

### Reglas

- `AuthenticationSession` no es la application session completa.
- no almacenar passwords ni secretos raw.
- validar expiracion y estado antes de restaurar contexto.
- preparar la forma para futura rotacion y revocacion.

### Tests minimos

1. session valida restaura contexto,
2. session expirada no restaura,
3. logout invalida el estado restaurable,
4. aislamiento entre requests consecutivos.

### Estado actual del corte

Parcialmente implementada:

- existe `AuthenticationSession`,
- existe `AuthenticationSessionId`,
- existe `AuthenticationSessionRepositoryInterface`,
- existe `InMemoryAuthenticationSessionRepository`,
- existe `SessionAuthenticator`,
- `AuthManager::login()` y `AuthManager::logout()` ya interactuan con la session,
- y el kernel HTTP ya emite `Set-Cookie` / `X-Auth-Session`.

Falta en esta fase:

- rotacion de session,
- storage persistente,
- y reglas mas fuertes de revocacion.

## Fase 5 - DX e integracion de framework

### Objetivo

Convertir el core en experiencia de desarrollo usable.

### Entregables

1. `Auth` facade
2. metodos en `AuthManager`:
   - `attempt()`
   - `login()`
   - `logout()`
   - `context()`
   - `principal()`
3. configuracion `config/auth.php`
4. `AuthenticationServiceProvider`
5. bindings de contratos default

### Integraciones esperadas

1. helper `auth()` sigue funcionando,
2. `Application.php` delega a provider dedicado,
3. Controllers Security puede consumir `AuthenticationContext` real,
4. ruta o middleware `auth` ya puede apoyarse directamente sobre este core.

### Tests minimos

1. `Auth::attempt()` autentica,
2. `Auth::logout()` limpia session y contexto,
3. facade no filtra estado entre requests,
4. bootstrap registra todos los servicios necesarios.

### Estado actual del corte

Parcialmente implementada:

- existe facade `Auth`,
- existe `config/auth.php`,
- `AuthManager` ya expone `attempt()`, `login()`, `logout()` y `context()`,
- existe `DefaultAuthenticatorResolver`,
- existe `AuthenticationServiceProvider`,
- existe middleware alias `auth`,
- existe middleware alias `guest`,
- y la session minima usa configuracion de cookie y expiracion.

Falta en esta fase:

- entry points adicionales mas alla de `auth/guest`,
- una API publica aun mas pulida para adopcion de framework,
- y alineacion mas profunda con Controllers Security.

## Fase 6 - Endurecimiento minimo para declarar V1

### Objetivo

Cerrar el primer release realmente util.

### Entregables

1. mapa de excepciones de Authentication
2. responses `401` coherentes
3. pruebas feature del flujo completo
4. documentacion de configuracion inicial
5. actualizacion de `DEVELOPMENT_MATRIX` y `DEVELOPMENT_VERSIONS`

### Estado actual del corte

Parcialmente implementada:

- existen errores propios de Authentication,
- existe elegibilidad minima de identidad,
- `attemptOrFail()` ya permite flujos con excepcion,
- el storage de session puede ser `memory` o `file`,
- y existe upgrade persistente inicial de hash para provider local con `storage_path`.

Falta en esta fase:

- revocacion distribuida,
- un modelo mas completo de errores por mecanismo,
- entry points complementarios,
- y stores mas robustos para despliegues reales.

### Criterio de cierre de V1

Se puede considerar cerrada la primera version del sistema cuando exista:

1. login por password,
2. restauracion por session,
3. logout,
4. `AuthenticationContext`,
5. facade/helper estables,
6. tests unitarios y feature,
7. integracion usable en una app VoltStack nueva.

## Mapa de clases prioritarias

## Prioridad P0

Estas son las clases que mas valor destraban al inicio:

1. `IdentityInterface`
2. `IdentityIdentifier`
3. `AuthenticationRequest`
4. `AuthenticationDecision`
5. `AuthenticationContext`
6. `AuthenticationManagerInterface`
7. `AuthenticationOrchestratorInterface`
8. `AuthenticationOperationContext`
9. `AuthenticationContextAccessor`

## Prioridad P1

1. `AuthenticatorInterface`
2. `PasswordAuthenticator`
3. `IdentityProviderInterface`
4. `CredentialVerifierInterface`
5. `AuthenticationSession`
6. `AuthenticationSessionRepositoryInterface`
7. `SessionAuthenticator`

## Prioridad P2

1. `Auth` facade
2. `AuthenticationServiceProvider`
3. `config/auth.php`
4. excepciones especificas
5. adaptacion a Controllers Security

## Orden recomendado de implementacion

1. `Context/`
2. `Identity/`
3. `Decisions/`
4. `Contracts/`
5. `Runtime/`
6. `Authenticators/PasswordAuthenticator`
7. `Sessions/`
8. `Facades/ + provider + config`
9. integracion con seguridad existente

## Integraciones del framework que deben tocarse

La implementacion de `Quantum/Auth` probablemente requerira tocar estas piezas:

1. `vendor/voltstack/framework/src/Platform/Application.php`
2. `vendor/voltstack/framework/src/Helper/helpers.php`
3. `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php`
4. `vendor/voltstack/framework/src/Quantum/Controllers/Security/Engine/ControllerSecurityManager.php`
5. `vendor/voltstack/framework/src/Quantum/Controllers/Security/Exceptions/ControllerSecurityExceptionMapper.php`
6. `config/` para introducir configuracion de auth

## Riesgos a vigilar

### 1. Acoplar todo a arrays

El `AuthManager` actual admite arrays y objetos arbitrarios.
Eso es util como compatibilidad temporal, pero la nueva arquitectura debe migrar progresivamente hacia identidades tipadas.

### 2. Saltarse el core por ergonomia

No permitir que:

- `Auth::login($identity)`

termine siendo solo:

- guardar un objeto en contexto o session

sin pasar por decision, context factory y persistencia.

### 3. Repetir semantica entre Auth y Controllers Security

Hay que alinear:

- `AuthenticationContext`
- `AuthenticationStrength`
- `AuthenticationRequiredException`

para evitar dos modelos paralelos de autenticacion dentro del framework.

### 4. Abrir demasiadas features antes de cerrar password + session

Si se abren temprano:

- tokens,
- MFA,
- federation,
- passkeys,

el riesgo es multiplicar contratos sin cerrar ningun flujo real.

## Definition of Done por fase

Una fase se considera realmente cerrada solo si:

1. existe codigo fuente identificable,
2. existe al menos un test unitario o feature representativo,
3. existe integracion real con runtime o bootstrap,
4. no se introduce estado global mutable,
5. se actualizan los artefactos de `AUTHENTICATION DEVELOPMENT`.

## Siguiente corte recomendado

El siguiente corte de implementacion recomendado es `DV-AUTH-083`.

### DV-AUTH-083

Alcance sugerido:

- **Passkeys FIDO2 criptográfico real**: sustituir @internal simulated shells por verifyAttestation/verifyAssertion WebAuthn (packed/tpm ES256 RS256 COSE keys), storage durable encryption-at-rest credenciales, PasskeyAuthenticator nuevo CompositeAuthenticatorResolver candidate.
- **OIDC criptográfico real**: JWKS fetch HTTP cache TTL 3600s (guzzle/curl vanilla), signature verification openssl JWT RS256/ES256 sobre JWKS n+e modulus exponent, clock_skew_leeway configurable, nonce binding replay protection, PKCE S256 flow code, state=CSRF-binding nonce-session.
- **Bearer Token V2 rotation**: refresh→new access+refresh pareja one-time use invalidation, family token reuse detection invalida toda la familia si reutilizas refresh padre, introspection endpoint (active/expired/revoked scopes client_metadata device_ref).
- **Throttle V2 distributed counters**: persistence pluggable Redis/database, cross-instance sync lockout propagation thresholds operation-type login/stepup/resets, HTTP 429 Retry-After mapping, risk integration throttle deny → risk_score +20.
- **Risk V2 adaptive deny policies**: auth.risk.deny_threshold=critical/high auto deny 403, risk.step_up_threshold=medium StepUpRequired assurance, signals pluggables, durable history patterns → level critical map min_assurance=HighestPasskey.
- **Assurance V2 orchestrator hooks**: comprobar min_assurance_operation preAuth hook actual<required → 423 assurance_insufficient envelope, CompositeAuthenticatorResolver aggregated amr fusion dedup, auth.assurance_insufficient coherent AuthenticationStrength middleware.

Entregables minimos:

1. PasskeyRegistrationCeremony.verifyAttestation() real + PasskeyAssertionCeremony.verifyAssertion() real (openssl, COSE ES256/RS256), PasskeyAuthenticator nuevo candidate priority 900, storage durable credenciales, 10 tests passkeys new cripto.
2. OidcIdentityTokenValidator.validateAllSignature() real RS256/ES256 sobre JWKS n+e kid match, fetch HTTP real JWKS con cache TTL, OidcAuthenticator nuevo candidate authorization code callback flow, 10 tests OIDC new signature validation.
3. BearerTokenService.rotateRefresh() emit new pareja, refresh one-time consume flag + family_id reuse detection invalidar descendencia bulk, 8 tests bearer rotation + reuse detection.
4. Throttle V2 mapping 429 Retry-After en AuthExceptionMapper, storage pluggable distributed counters, 6 tests Throttle V2 HTTP 429 response.
5. Risk V2 adaptive deny threshold configurable + mapping StepUp/assurance min, 6 tests Risk V2 threshold deny.
6. Assurance V2 Orchestrator hooks: min_assurance_operation check preAuth, return StepUpRequired/423 envelope, auth.assurance_insufficient middleware, 6 tests Assurance V2 triggers.
7. SP wiring DI: 3 interfaces nuevas passkey/oidc/throttleV2 bindings con default return null config disabled flags (auth.passkeys.enabled default false; auth.oidc.enabled default false; auth.throttle.distributed.enabled default false).

Resultado esperado:

- Authentication subsistema V1 completo: password auth, session auth, trusted device MFA, bearer tokens opaque V2, passkeys FIDO2 criptográficamente validos, OIDC federated criptográficamente validos, throttle distributed, risk adaptive, assurance min triggers orchestrator.
- 7 bloques del slice 082 (A-G) completados con criptografía real, no skeleton shells simulated.
- Cross-suite AUTH ≥ 240 tests exit 0, baseline 216 tests 082 intacto sin regresiones.
- Docs 4 actualizados cierre 083.

## Cortes recomendados previos

### DV-AUTH-082

Alcance sugerido:

- Bloque A gobernanza distribuida passwords: DistributedPasswordGovernanceProviderInterface, PasswordRotationReceipt, RetentionTieredEnforcer 3-tier, LocalIdentityProvider implements via updateEntry(), 3 gates PasswordAuthenticator.
- Bloque B Throttling V1: AbuseProtectionThrottleInterface decide/recordAttempt, BruteForceCounter sliding 1m/5m/15m, CredentialStuffingBloomFilter 40 passwords, ThrottleEngineV1 umbrales identifier×1 / device×1.5 / ip×2, Orchestrator preAuth hook.
- Bloque C Risk Signal V1: RiskScore enum low/medium/high/critical cap100, 4 signals (NewDevice +25 / IpDrift +15 / ImpossibleTravel +40 / IrregularTime +10), CompositeRiskSignalEngine suma cap100, Orchestrator postAuth metadata SIN deny V1.
- Bloque D Assurance composable: AuthenticationMethodReferenceList dedup amr, AssuranceProfile int enum 0-7 (Lowest→HighestPasskey), StepUpRequirement required/not VO, AssuranceStepUpEvaluator min_assurance operation, AuthenticationAssurance composeFromAmr / meetsMinimumAssurance sin modificar código existente.
- Bloque E Nonce + CSRF Binding: TransactionNonceStoreInterface issue/validate, NonceRecord/NonceValidationResult VOs, CsrfChallengeBinder HKDF deterministic, InMemoryTransactionNonceStore one-time correct purge order (check exist → check individual expiry → purge global distingue expired vs consumed), Orchestrator preAuth Nonce validate ANTES throttle, postAuth issue nonce+CSRF metadata.
- Bloque F Passkeys Skeleton V1 simulated: RelyingPartyConfig/PasskeyCredentialRecord fromArray/toArray, PasskeyCredentialStoreInterface/InMemoryPasskeyCredentialStore, PasskeyRegistrationCeremony @internal beginChallenge simulated 64hex, PasskeyAssertionCeremony @internal verifyAssertion simulated sin crypto.
- Bloque G OIDC Skeleton V1 simulated: OidcWellKnownClientInterface/OidcJwksCacheInterface contracts, OidcProviderMetadata + InMemoryMockOidcWellKnownClient hardcodeado sin HTTP fetch, InMemoryOidcJwksCache TTL naive, OidcIdentityTokenValidator shell 6 structural checks (iss/aud/exp/nonce/azp/at_hash) SIN crypto, FederatedClaimsMapper email_verified=true→Active / false→Suspended heurística.
- H1 cross-suite completa ≥ 192 tests AUTH exit 0; H2 actualizar 4 docs AUTHENTICATION DEVELOPMENT.

Resultado alcanzado (exit 0, 216 tests ≥ 192 objetivo, 2713 assertions, baseline 081 intacto):

- Bloques A-G implementados + unit tests 10+10+8+8+8+6+6 = 56 unit nuevos verified 100% BloqueXTest exit 0 cada uno.
- Unit AUTH total 143 tests + Feature AuthManagerTest 61/61 + SkeletonSecuritySmokeTest 12/12 = 216 tests AUTH exit 0, baseline 136 tests 081 intacto sin regresiones.
- Orchestrator rama 1-candidate 100% intacta (if ($tried === 1) return $firstDecision; SIN mods), hooks preAuth (Nonce→Throttle) y postAuth (Risk→Nonce+CSRF) envueltos FUERA de la rama backward compat.
- Backward compat layered OPT-IN 100%: 4 interfaces nuevas = instanceof checks; config flags throttle/risk/transaction.nonce enabled = default false; constructor Orchestrator 4 args nuevos nullable = null default; metadata nueva = array_filter eliminando nulls.
- SP DI FIX crítico: eliminar bound() inexistente → bindings DIRECTOS scoped interfaces 2 (AbuseProtectionThrottleInterface / TransactionNonceStoreInterface) retorna null cuando config disabled; VoltStack Container NO respeta = null default type-hints interfaces, binding DEBE existir siempre.
- Reglas permanentes STANDING RULE preservadas: actor_target_scope_relation clave no índice posicional; distributed_guard_denied matched_resources>0 affected=0; delegated-target-protected middleware('auth') 401 post-revocación.

Gap natural posterior (083):

- reemplazar skeletons simulated shells con validación criptográfica real (passkeys attestation/assertion WebAuthn openssl COSE, OIDC JWT signature JWKS fetch HTTP real),
- bearer token V2 rotation pareja one-time + family reuse detection,
- throttle V2 distributed counters persistence pluggable Redis/database + HTTP 429 mapping Retry-After,
- risk V2 adaptive deny thresholds configurable auto deny 403 / step_up_required,
- assurance V2 orchestrator hooks preAuth comprobar min_assurance_operation actual < required → 423 assurance_insufficient envelope.

### DV-AUTH-080

Alcance sugerido:

- introducir aserciones de test explicitas sobre el nuevo envelope estructurado de rechazo (correlation_id, operation_id, result, reason_code, 12-clave resource_coverage) en todas las ramas de validacion temprana y authorization_failed, tanto en salida JSON como en eventos JSONL de --audit-log
- ampliar cobertura E2E del runtime principal con taxonomias actor-target-scope administrativas adicionales y escenarios de degradacion parcial distribuida variada en managedDevices() y revokeManagedDevice()
- construir un harness formal de snapshots/export long-form para validar --export-log y --audit-log con trazabilidad de envelopes completos a traves del tiempo, correlacionando correlation_id y operation_id entre reporte, auditoria y mutacion operativa

Entregables minimos:

1. extender AuthSecurityCenterRevokeDeviceCommandTest con casos explicitos que validen presencia y forma correcta de correlation_id, operation_id, result, reason_code y las 12 claves de resource_coverage en la salida JSON Y en los eventos JSONL de --audit-log, cubriendo las 6 ramas de rechazo (5 validaciones tempranas + authorization_failed).
2. ampliar AuthManagerTest con casos feature adicionales que fijen taxonomias actor-target-scope administrativas mas variadas (mas overlays delegated_admin_*, direct_admin_*, self_governed_*) bajo combinaciones distintas de drift operativo (recent_lag, partial_visibility, concentrated_activity), escenarios de degradacion parcial distribuida y coverage de scope sessions|trusted-devices|all en managedDevices() y revokeManagedDevice().
3. construir un harness de snapshots/export long-form que valide --export-log y --audit-log con trazabilidad completa: correlacion cruzada de correlation_id y operation_id entre reporte, auditoria durable y salida de mutacion; preservacion de resource_coverage completo con matched_resources vs affected_resources a traves del tiempo; y conservacion de actor_aware_profiles / mutation_actor_profiles con ordenacion estable desacoplada de indice posicional.
4. mantener un policy/runtime compartido mas expresivo entre AuthenticationContext, AuthManager, report y revoke-device; conservar nomenclatura unificada sin regresar a claves legacy observed_affected_*; y mantener AuthManagerTest verde sin regresiones (59 tests / 732 assertions minimo).
5. ampliar pruebas de integracion y hardening sobre escenarios delegados, directos, multi-nodo, ordenacion estable de perfiles y degradaciones graduales adicionales; introducir variaciones de provider local con enriquecimiento persistente para validar escenarios multi-fuente.
6. seguir endureciendo limpieza y retencion gobernada de tombstones de recovery; explorar retencion segmentada por tipo de outcome operativo; y seguir evolucionando el provider local hacia fuentes persistentes mas ricas con enrichment local durable para actor-aware profiles.

Resultado real (evidencia):

- Suite `AuthSecurityCenterRevokeDeviceCommandTest` verde: **27 tests / 811 assertions**. Incluye 2 tests authorization_failed extendidos (L1077-1257) y 5 tests validation_failed NUEVOS (L1259-1631: missing_identity, missing_device_reference, missing_actor_identity, missing_actor_session_public_id, invalid_scope), cada uno con envelope JSON + JSONL de 12 claves resource_coverage + correlation_id/operation_id cross-check.
- Suite `AuthManagerTest` verde: **60 tests / 773 assertions**. Incluye 1 test E2E NUEVO `test_auth_manager_managed_devices_handles_multiple_target_devices_and_scopes_for_delegated_admin` (L4863-L5123, 41 assertions): 3 identidades (target-A MFA 2 devices, target-B MFA 1 device trusted, delegated_admin full_scope), 4 logins, inventario multi-identidad, 3 operaciones revoke por scope granular (sessions / trusted-devices / all) y 4 endpoints protegidos validando 401/200 post-revocation.
- Suite `AuthSecurityCenterReportCommandTest` verde: **12 tests / 524 assertions**. Incluye 1 harness NUEVO snapshots/export long-form (L994-L1451, 81 assertions): 6 eventos custom audit-log con 6 outcomes distintos (executed sessions/trusted/all + validation_failed + authorization_failed + distributed_guard_denied), correlation_id cross report ↔ export ↔ audit source, matched_total=10 sessions=6 trusted=4, affected_total=5 sessions=3 trusted=2, matched_resources propagados por time_windows / store_time_windows / store_cohorts / mutation_actor_profiles / mutation_scope_profiles con perfiles filtrados por clave (no indice posicional).
- Suites combinadas framework: **99 tests / 2108 assertions exit 0** sin regresiones.
- El contrato unificado de envelopes estructurados de rechazo queda fijado mediante aserciones explicitas, la taxonomia actor-target-scope administrativa multi-identity multi-login multi-scope queda mejor cubierta y el harness longitudinal habilita validacion durable reproducible.

### DV-AUTH-079

Alcance sugerido:

- normalizar `resource_coverage` y envelopes de subconjunto afectado en todas las ramas JSON y JSONL de rechazo, decision y ejecucion operativa
- reforzar la trazabilidad cruzada entre `report`, export, audit trail durable y `revoke-device` cuando un mismo subconjunto de recursos atraviesa outcomes distintos
- seguir ampliando la cobertura end-to-end del runtime principal para taxonomias actor-target-scope administrativas adicionales y degradaciones parciales distribuidas

Entregables minimos:

1. extender `revoke-device`, `--audit-log` y envelopes de rechazo para publicar siempre el mismo lenguaje de `resource_coverage`, recursos candidatos y recursos efectivamente afectados.
2. reforzar `longitudinal_metrics`, `store_cohorts`, export y auditoria durable para correlacionar mejor outcomes distintos sobre un mismo subconjunto de recursos y stores degradados.
3. ampliar `revoke-device` y la suite feature del runtime principal con escenarios adicionales que fijen esas taxonomias actor-target-scope, outcomes y degradaciones parciales en `managedDevices()` y `revokeManagedDevice()`.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios delegados, directos, multi-nodo, ordenacion estable de perfiles y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con una correlacion mas rica entre drift, subconjuntos afectados, cohortes distribuidas, mutacion y outcomes operativos compartidos entre reporte y auditoria,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-078

Alcance sugerido:

- extender `resource_coverage`, `affected_resource_kinds` y la nocion de subconjunto afectado al `--export-log`, `--audit-log` y `revoke-device`
- reforzar la trazabilidad cruzada entre `report`, export, audit trail durable y `revoke-device` cuando distintos stores sostienen distintos subconjuntos del impacto esperado
- seguir ampliando la cobertura end-to-end del runtime principal para taxonomias actor-target-scope administrativas adicionales y degradaciones parciales distribuidas

Entregables minimos:

1. extender `report`, `--export-log` y `--audit-log` para publicar el mismo lenguaje de `resource_coverage`, `affected_resource_kinds` y subconjuntos afectados.
2. reforzar `longitudinal_metrics`, `store_cohorts`, export y auditoria durable para correlacionar mejor `mutation_actor_profiles`, `target_store_assessments`, recursos afectados y drift segun la mutacion pedida.
3. ampliar `revoke-device` y la suite feature del runtime principal con escenarios adicionales que fijen esas taxonomias actor-target-scope y degradaciones parciales en `managedDevices()` y `revokeManagedDevice()`.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios delegados, directos, multi-nodo, ordenacion estable de perfiles y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con una correlacion mas rica entre drift, subconjuntos afectados, cohortes distribuidas, mutacion y decision operativa compartida entre reporte y auditoria,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-077

Alcance sugerido:

- correlacionar mejor `mutation_scope_profiles`, `mutation_actor_profiles` y `target_store_assessments` cuando una misma policy actor-aware cubre subconjuntos distintos de recursos afectados
- extender la trazabilidad cruzada entre `report`, export, audit trail durable y `revoke-device` para degradaciones parciales multi-store mas accionables
- seguir ampliando la cobertura end-to-end del runtime principal para taxonomias actor-target-scope administrativas adicionales y degradaciones parciales distribuidas

Entregables minimos:

1. extender `report` para distinguir mejor que cohortes, stores o recursos afectados sostienen cada `policy_source`, `policy_reason_code` y denial actor-aware cuando el drift es parcial.
2. reforzar `longitudinal_metrics`, `store_cohorts`, export y auditoria durable para correlacionar mejor `mutation_actor_profiles`, `target_store_assessments`, recursos afectados y drift segun la mutacion pedida.
3. ampliar la suite feature del runtime principal con escenarios adicionales que fijen esas taxonomias actor-target-scope y degradaciones parciales en `managedDevices()` y `revokeManagedDevice()`.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios delegados, directos, multi-nodo, ordenacion estable de perfiles y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con una correlacion mas rica entre drift, recursos afectados, cohortes distribuidas, mutacion y decision operativa,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-076

Alcance sugerido:

- enriquecer `mutation_scope_profiles` con vistas mas actor-aware para explicar mejor `policy_source`, `policy_reason_code` y `reason_code` por mutacion
- profundizar la correlacion por cohorte/store entre drift, audit trail durable, export operativo y decisiones distribuidas reales
- seguir ampliando la cobertura end-to-end del runtime principal para taxonomias actor-target-scope administrativas adicionales y sus degradaciones distribuidas

Entregables minimos:

1. extender `report` para proyectar perfiles de mutacion mas actor-aware sin depender exclusivamente del contrato global de `operational_response`.
2. reforzar `longitudinal_metrics`, `store_cohorts`, export y auditoria durable para correlacionar mejor `mutation_kinds`, `policy_source`, `policy_reason_code`, `reason_code`, `target_store_assessments` y drift segun la mutacion pedida.
3. ampliar la suite feature del runtime principal con escenarios adicionales que fijen esas taxonomias actor-target-scope en `managedDevices()` y `revokeManagedDevice()`.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios delegados, directos, multi-nodo y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con una correlacion mas rica entre drift, cohortes distribuidas, mutacion y decision operativa,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-075

Alcance sugerido:

- extender overlays graduales equivalentes sobre `delegated_admin_full_scope_target` y otros casos administrativos plenos que aun resuelven en policies demasiado generales
- enriquecer la correlacion entre drift, `mutation_scope_profiles` y audit/export operativo por tipo de mutacion
- seguir ampliando la cobertura end-to-end del runtime principal para taxonomias actor-target-scope adicionales y sus degradaciones distribuidas

Entregables minimos:

1. extender `target_scope_relation_policies` y `distributed_guard_scope_decision` a mas combinaciones `delegated_admin_full_scope_target` y overlays administrativos plenos equivalentes.
2. reforzar `report`, export y auditoria durable para correlacionar mejor `mutation_scope_profiles`, `target_store_assessments`, drift, `policy_source` y `reason_code` segun la mutacion pedida.
3. ampliar la suite feature del runtime principal con escenarios adicionales que fijen esas taxonomias actor-target-scope en `managedDevices()` y `revokeManagedDevice()`.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios delegados plenos, multi-nodo y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con contratos mas completos tambien para delegacion administrativa plena,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-072

Alcance sugerido:

- hardening adicional de guardia distribuida sobre perfiles multi-store intermedios y `store_assessments`
- observabilidad operativa mas rica para degradaciones actor-target-scope
- trazabilidad gobernada mas fuerte sobre mutaciones administrativas distribuidas y ownership proyectado

Entregables minimos:

1. enriquecer `operational_response` y `distributed_guard_scope_decision` con perfiles adicionales derivados de `coordination_profile`, `store_assessments`, `store_cohorts`, `time_windows` y `store_time_windows`.
2. ampliar la trazabilidad operativa para explicar mejor que scopes y targets administrativos quedan degradados en cada drift.
3. mantener un policy/runtime compartido mas expresivo para actores administrativos sobre inventory agregado, mutaciones remotas y tooling de consola.
4. ampliar pruebas de integracion y hardening sobre escenarios delegados, distribuidos, multi-nodo y visibilidad irregular entre stores.
5. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.
6. sentar base para observabilidad administrativa mas util en produccion.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy administrativa, management de dispositivos, governance multi-actor, coordinacion distribuida, deteccion de drift, escalacion operativa, denials distribuidos y observabilidad operativa,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-073

Alcance sugerido:

- consolidar overlays graduales adicionales sobre `target_scope_relation_policies` y `distributed_guard_scope_decision`
- ampliar cobertura end-to-end del runtime principal para degradaciones actor-target-scope ya publicadas por el tooling operativo
- enriquecer la explicabilidad para que el reporte resuma mejor que stores y targets sostienen cada degradacion parcial

Entregables minimos:

1. extender las combinaciones `actor-target-scope` cubiertas por la guardia distribuida, especialmente sobre perfiles graduales ya presentes como `recent_lag` y `concentrated_activity`.
2. ampliar la suite feature del runtime principal para fijar como se proyectan y respetan esas degradaciones en `managedDevices()` y `revokeManagedDevice()`.
3. reforzar `report` para que resuma mejor la relacion entre `store_assessments`, `target_store_fingerprints`, `degraded_scope_profiles` y el drift observado.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios delegados, multi-nodo y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con contratos mas completos entre runtime principal y tooling operativo,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-074

Alcance sugerido:

- profundizar overlays graduales equivalentes sobre targets `direct_admin_*` y combinaciones adicionales de `delegated_admin_full_scope_target`
- enriquecer la correlacion operativa entre `target_store_fingerprints`, `target_store_assessments`, drift y denials por tipo de mutacion
- seguir tendiendo puentes entre la semantica distribuida del tooling operativo y los paths publicos del runtime principal

Entregables minimos:

1. extender `target_scope_relation_policies` y `distributed_guard_scope_decision` a mas combinaciones `direct_admin_*` y casos graduales adicionales sobre targets administrados plenos.
2. reforzar `report` para resumir mejor que stores explican cada denial o degradacion parcial segun el tipo de mutacion pedida.
3. ampliar la suite feature del runtime principal con escenarios adicionales que fijen esas taxonomias actor-target-scope en `managedDevices()` y `revokeManagedDevice()`.
4. mantener un policy/runtime compartido mas expresivo entre `AuthenticationContext`, `AuthManager`, `report` y `revoke-device`.
5. ampliar pruebas de integracion y hardening sobre escenarios directos, delegados, multi-nodo y degradaciones parciales adicionales.
6. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva la semantica gradual ya introducida, pero con overlays mas completos para actores directos y delegados,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-021

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- metadata de device mas rica y policy/authorization mas expresiva
- limpieza/retencion gobernada de tombstones

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. metadata de device y actividad mas util para security center.
3. policy/authorization mas rica para revocacion administrativa.
4. limpieza/retencion de tombstones y recovery coordinado.
5. alineacion con Controllers Security y `AuthenticationContext`.
6. pruebas de integracion y hardening adicionales.
7. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy y entry points,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-022

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- tooling/cleanup de tombstones y retention minima operativa
- policy/authorization mas expresiva para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. cleanup/retention minima de tombstones y metadata derivada.
3. policy/authorization mas rica para revocacion administrativa.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup y entry points,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-023

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- policy/authorization mas expresiva para revocacion administrativa
- trusted devices y metadata de device mas estable

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. policy/authorization mas rica para revocacion administrativa y escenarios multi-actor.
3. trusted devices o referencias de device mas estables.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup y device posture,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-024

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- trusted devices reales y credentials duraderas
- policy/authorization multi-actor para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. trusted device records o credentials mas formales.
3. policy/authorization mas rica para revocacion administrativa y multi-actor.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup, trusted devices y device posture,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-025

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- trusted-device credentials cliente duraderas
- policy/authorization multi-actor para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. trusted-device credentials cliente mas formales y su validacion.
3. policy/authorization mas rica para revocacion administrativa y multi-actor.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup, trusted-device posture y challenge reduction,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-026

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- rotacion y replay hardening de trusted-device credentials cliente
- policy/authorization multi-actor para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. rotacion, invalidacion rica y replay hardening de trusted-device credentials.
3. policy/authorization mas rica para revocacion administrativa y multi-actor.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva challenge reduction para dispositivos reconocidos, pero endurece rotacion, replay protection y revocacion de la credencial cliente,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.
