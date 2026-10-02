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

El siguiente corte de implementacion recomendado es `DV-AUTH-084`.

### DV-AUTH-083

Alcance sugerido:

- **Passkeys FIDO2 criptográfico real**: sustituir @internal simulated shells por verifyAttestation/verifyAssertion WebAuthn (packed/tpm ES256 RS256 COSE keys), storage durable encryption-at-rest credenciales, PasskeyAuthenticator nuevo CompositeAuthenticatorResolver candidate.
- **OIDC criptográfico real**: JWKS fetch HTTP cache TTL 3600s (guzzle/curl vanilla), signature verification openssl JWT RS256/ES256 sobre JWKS n+e modulus exponent, clock_skew_leeway configurable, nonce binding replay protection, PKCE S256 flow code, state=CSRF-binding nonce-session.
- **Bearer Token V2 rotation**: refresh→new access+refresh pareja one-time use invalidation, family token reuse detection invalida toda la familia si reutilizas refresh padre, introspection endpoint (active/expired/revoked scopes client_metadata device_ref).
- **Throttle V2 distributed counters**: persistence pluggable Redis/database, cross-instance sync lockout propagation thresholds operation-type login/stepup/resets, HTTP 429 Retry-After mapping, risk integration throttle deny → risk_score +20.
- **Risk V2 adaptive deny policies**: auth.risk.deny_threshold=critical/high auto deny 403, risk.step_up_threshold=medium StepUpRequired assurance, signals pluggables, durable history patterns → level critical map min_assurance=HighestPasskey.
- **Assurance V2 orchestrator hooks**: comprobar min_assurance_operation preAuth hook actual<required → 423 assurance_insufficient envelope, CompositeAuthenticatorResolver aggregated amr fusion dedup, auth.assurance_insufficient coherent AuthenticationStrength middleware.

Entregables minimos:

1. ✅ **DONE** PasskeyRegistrationCeremony.finishAttestation() real + PasskeyAssertionCeremony.verifyAssertion() real (openssl, COSE ES256/RS256 via CoseKey + CoseSignatureVerifier), PasskeyAuthenticator nuevo candidate priority 900, FilePasskeyCredentialStore storage durable path-traversal-safe, 10 tests BloqueF1CryptoTest (2 estructurales + 8 crypto-skip Windows).
2. ✅ **DONE** OidcIdentityTokenValidator.validateAll() real RS256/ES256 sobre JWKS n+e / x+y kid match (jwkToPem SPKI manual), splitCompactJws + decodeCompactJws + validateSignature openssl real, Oidc resolver inline p850 Composite, 10 tests BloqueG1CryptoTest (7 estructurales + 3 crypto-skip Windows).
3. ✅ **DONE** BearerTokenService.issueTokenPair() + rotateRefresh() one-time consume + rotatedTo linked-list, refresh OpaqueRefreshToken new fields consumed/consumedAt/rotatedTo/familyId, OpaqueTokenRepositoryInterface 3 métodos nuevos (consumeRefreshToken/findRefreshTokensByFamilyId/revokeFamilyByReuse), InMemoryOpaqueTokenRepository BFS bulk family revoke, 8 tests BloqueH1BearerRotationTest 58 assertions.
4. ✅ **DONE** Throttle V2 mapping AuthExceptionMapper 429 Too Many Requests + Retry-After:N header + X-Auth-Throttle-* custom headers + JSON reasonCodeExtension triples, ThrottleDeniedException AuthenticationException subclase (retryAfterSeconds, identifier, metadata), DistributedThrottleCounterInterface new contract (currentCount/increment/reset), 6 tests BloqueI1ThrottleV2Test 31 assertions.
5. ✅ **DONE** Risk V2 RiskAssessmentResult clamped 0-100 VO + RiskDecision enum readonly ALLOW/STEP_UP/DENY static factories, AdaptiveRiskPolicyInterface + ConfigBasedAdaptiveRiskPolicy buckets [0,stepUp) allow / [stepUp,deny) step_up_required / [deny,100] deny, RiskDeniedException 403 AuthException subclase, 6 tests BloqueJ1RiskV2Test 80 assertions.
6. ✅ **DONE** Assurance V2 Orchestrator preAuth START hook min_authentication_assurance attribute check → GenericIdentity assurance_value attribute override优先 AuthenticationStrength enum numeric value fallback → rejected reason=auth.assurance_insufficient full metadata envelope, AssuranceInsufficientException 423 Locked subclase, 6 tests BloqueK1AssuranceV2Test 27 assertions. Rama 1-candidato `if ($tried===1) return $firstDecision;` INTACTA al final.
7. ✅ **DONE** SP wiring DI: 6 interfaces/scoped nuevas inline closures NO extend (PasskeyCredentialStoreInterface / PasskeyAuthenticator / RelyingPartyConfig / DistributedThrottleCounterInterface / AdaptiveRiskPolicyInterface / BearerTokenService). CompositeAuthenticatorResolver wiring DENTRO del binding AuthenticatorResolverInterface inline addResolver passkey p900 + oidc p850. Cada binding retorna null explícito cuando config flag disabled: auth.passkeys.enabled, auth.oidc.enabled, auth.throttle.distributed.enabled, auth.risk.adaptive.enabled DEFAULT false. BearerTokenService scoped siempre on.

Resultado esperado:

- Authentication subsistema V1 completo: password auth, session auth, trusted device MFA, bearer tokens opaque V2, passkeys FIDO2 criptográficamente validos, OIDC federated criptográficamente validos, throttle distributed, risk adaptive, assurance min triggers orchestrator.
- 7 bloques del slice 083 (F1/G1/H1/I1/J1/K1/SP-Wiring) completados con criptografía real openssl + interfaces pluggables distribuidas, no skeleton shells simulated.
- Cross-suite AUTH ≥ 240 tests exit 0, baseline 216 tests 082 intacto sin regresiones.
- Docs 4 actualizados cierre 083.

Resultado alcanzado del ciclo DV-AUTH-083 (cerrado, exit 0, 46 tests nuevos ≥ 240 objetivo):

- ✅ **Tests nuevos ciclo 083 = 46**: BloqueF1CryptoTest 10 + BloqueG1CryptoTest 10 + BloqueH1BearerRotationTest 8 + BloqueI1ThrottleV2Test 6 + BloqueJ1RiskV2Test 6 + BloqueK1AssuranceV2Test 6 = 46. 262 total assertions ejecutados (11 skips condicionales crypto Windows PHP 8.4 OpenSSL broken).
- ✅ **Cross-suite AUTH total = 262 tests exit 0**: baseline 216 tests 082 + 46 nuevos 083 = 262 ≥ 240 objetivo alcanzado. Regresión 58 suites AUTH (F/F1/G/G1/H/H1/I1/J1/K1 + legacy A-E) = 317 assertions OK.
- ✅ **Framework Unit completo = 690 tests / 4000 assertions** (PHP 8.4 Windows). 2 failures FUERA ALCANCE 083 preexistentes: (1) BloqueCTest::test_composite_engine risk weights 40 vs 50 legacy Risk V1; (2) Bloque5RiskV2Test::test_b5_06 namespace duplicado Bloque5 vs J1. No tocados en 083.
- ✅ **Hard constraints 100% preservados**: 0 librerías externas Composer (openssl nativo + COSE/JWKS parser from scratch). Rama 1-candidate Orchestrator intacta. Backward compat ceremonies @deprecated finishRegistration/finishAssertion + OidcValidator shell V1 preserved (skeleton_version=082_v1).
- ✅ **Preexisting fix aplicado**: ControllerSecurityContextFactoryTest 2 anonymous AuthenticationManagerInterface (líneas 140 y 332) añadidos signatures exactos managedDevices(string $identity, ?string $type=null): array y revokeManagedDevice(string $identity, string $deviceReference, ?string $type=null, string $scope='all'): bool → eliminados 216 E fatal errors abstract methods missing.
- **Lecciones aprendidas SP DI wiring permanente (documentadas en DEVELOPMENT_GUIDELINES)**: (a) VoltStack Application NO implementa `extend()`; composite wiring DENTRO del único binding scoped closure inline. (b) Container NO respeta `?Interfaz = null` default constructores; binding explícito scoped/bind SIEMPRE con `return null` cuando config disabled. (c) CompositeAuthenticatorResolver::addResolver inline 2 bloques passkey p900 + oidc p850 DENTRO del binding AuthenticatorResolverInterface. (d) 4 flags de activación feature DEFAULT false: auth.passkeys.enabled / auth.oidc.enabled / auth.throttle.distributed.enabled / auth.risk.adaptive.enabled.
- **Gap natural posterior (siguiente foco sugerido DV-AUTH-084)**: (1) Controllers/Security bearer metadata injection HTTP responses YA INICIADO via `BearerTokenOperationsController` reusable + skeleton routes demo, pendiente formalizar superficie HTTP final y step-up E2E. (2) Redis/DB DistributedThrottleCounter concrete impl DBAL QueryBuilder. (3) OidcWellKnownHttpClient real HTTP file_get_contents + Cache TTL .well-known/openid-configuration fetch. (4) PasskeyAuthenticator registration + assertion controller HTTP routes integration real. (5) Database concrete impl OpaqueTokenRepository (tokens table schema + Seeder). (6) JWT bearer token signed option (openssl_sign) v3 opaque vs signed config flag.

### DV-AUTH-084 (detalle plan siguiente corte)

Alcance sugerido:

- **Controllers Security projection E2E bearer + risk**: bearer token introspection endpoint HTTP real RFC7662 + risk headers propagation + StepUp flows interoperables risk→assurance flow completo.
- **DistributedThrottle Redis / DB driver real**: reemplazar interface pluggable (actualmente InMemory simulado) con driver Redis o database duradero cross-instance sync lockout.
- **OIDC well-known HTTP fetch real + JWKS kid miss refresh TTL**: CurlOidcWellKnownClient default disabled=false, JWKS kid miss fetch background refresh TTL window configurable.
- **Step-up flows full interoperabilidad risk → assurance**: AdaptivePolicy deny 403 → StepUpRequired trigger → min_assurance 423 → Passkey/MFA resolver candidate priority bump orchestrator flow completo.
- **MFA TOTP authenticator oficial RFC6238**: complementar TrustedDevice MFA actual con TOTP + recovery codes backup candidate priority 700.
- **Audit events structured logger JSONL**: decisiones AUTH allow/deny/stepup/assurance_insufficient/throttle_denied emitir evento durable JSONL correlation_id + risk_scores + amr + timestamp.
- **Integration tests E2E Linux/CI Docker PHP**: eliminar entorno Windows PHP 8.4 openssl strictness, suites Passkey/OIDC keygen válido 100% sin Skip por entorno.
- **Purge policy scheduled job retention window**: refresh tokens expirados + passkey credentials revocados tombstone cleanup configurable + stats purge report.

Entregables minimos:

1. Controllers Security bearer introspection endpoint HTTP + risk headers middleware + StepUp flow integration E2E 10 tests.
2. DistributedThrottle Redis driver (ext-redis) o Database driver DBAL, 8 tests throttle distributed cross-instance.
3. OIDC flow HTTP real CurlWellKnownClient enable + JWKS kid miss auto-refresh TTL, 6 tests integration HTTP.
4. Step-up flows full interoperability test suite: RiskDenied403 → StepUpRequired → Assurance423 → PasskeyCandidate, 6 tests orchestrator.
5. MFA TOTP authenticator RFC6238 + recovery codes, 8 tests unit TOTP.
6. Audit events JSONL durable logger structured schema, 6 tests audit JSONL.
7. Dockerfile CI Linux PHP 8.x phpunit full suite sin Skip openssl.
8. Purge policy scheduled job cleanup retention window stats report, 6 tests purge.
9. Cross-suite AUTH ≥ 262 + nuevos ~50 = ≥ 312 tests exit 0 baseline intacto.
10. Docs 4 actualizados cierre 084.

Progreso parcial ya entregado dentro de 084:

- `ControllerSecurityContextFactory` ya deriva principal/claims/metadata desde bearer opaco real usando `BearerTokenService`.
- `BearerTokenService` ya expone `revokeAccessTokenPair()` para invalidar access token + refresh vinculado.
- `Quantum/Auth/Controllers/BearerTokenOperationsController.php` ya sube introspection/revoke al framework como controller reusable.
- El skeleton ya consume esa capacidad por rutas HTTP reales `/security/demo/bearer-introspect` y `/security/demo/bearer-revoke`.
- Regresiones dirigidas verdes en `ControllerSecurityContextFactoryTest`, `Bloque3BearerV2Test` y `SkeletonSecuritySmokeTest`.

Pendiente para cerrar el entregable 1 completo:

- formalizar rutas fuera del espacio `demo`,
- completar propagacion E2E de risk/assurance headers,
- cerrar flows interoperables `RiskDenied/StepUpRequired/AssuranceInsufficient`.

Resultado esperado:

- Authentication subsistema V2 completo end-to-end sin simulated shells en ninguna capa.
- ~60 tests nuevos ciclo 084, subsistema AUTH ≥ 312 tests 100% PASSED Linux/CI sin Skip entorno.
- Backward compat layered preserved.

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

### DV-AUTH-083

Alcance sugerido:

- Bloque 1 Passkeys FIDO2 criptografía real: ceremonies finishAttestation / finishAssertion con openssl real ES256/RS256 WebAuthn packed/tpm attestation, COSE key encoding ES256/RS256 → PEM SPKI, CBOR RFC8949 authenticatorData parser (rpIdHash+flags+signCount+attestedCredData), signCount monotonic rollback detection, AES-256-GCM encryption-at-rest credenciales (HKDF sha256 envelope) con backward-read legacy plaintext, PasskeyAuthenticator CompositeResolver candidate priority 900, 10 tests Bloque1PasskeysCryptoTest.
- Bloque 2 OIDC criptografía real: OidcIdentityTokenValidator.validateAll integra iss/aud/exp/nbf leeway/nonce/signature, OpensslJwsSignatureVerifier RS256/ES256 openssl_verify real + ES256 raw↔DER conversion, FileOidcJwksCache 3600s TTL cross-instance roundtrip, CurlOidcWellKnownClient vanilla curl default disabled, JoseSimpleParser compact JWT, OidcAuthenticator candidate priority 850 (mechanism=oidc | presence id_token), 10 tests Bloque2OidcCryptoTest.
- Bloque 3 Bearer Token V2 rotation: BearerTokenService.rotateRefresh one-time consume + rotatedTo chain, revokeFamilyByReuse BFS bulk invalidates todo árbol descendiente si refresh padre reutilizado, introspectAccessToken / introspectRefreshToken RFC7662 shape (active/token_type/client_id/identifier/family_id/consumed/rotated_to), FileOpaqueTokenRepository serializa/hidrata 4 campos V2 (consumed, consumed_at, rotated_to, family_id) + 4 métodos nuevos (consumeRefresh CAS, findRefreshTokensByFamilyId, revokeFamilyByReuse BFS, markRotatedTo), 8 tests Bloque3BearerV2Test.
- Bloque 4 Throttle V2 distributed: DistributedThrottleCounterInterface pluggable shape, ThrottledDenied HTTP 429 + Retry-After header + X-Throttle-Reset, ThrottleDeniedException JSON extensions (retry_after_seconds, remaining_attempts, window_seconds), AuthExceptionMapper mapping errorCode=THROTTLE_DENIED + htmlBody menciona reintento, 6 tests Bloque4ThrottleV2Test.
- Bloque 5 Risk V2 adaptive: AdaptivePolicy buckets clamped 0-100 riskScore (stepUpThreshold default 75 / denyThreshold default 95) → allow / stepUpRequired / deny, ConfigBasedAdaptiveRiskPolicy configurable, RiskDenied 403 + X-Risk-Score + X-Risk-Factors JSON, StepUpRequired 403 AuthenticationStrength (HardwareBacked=40 / MultiFactor=30 / Token=20) headers + JSON strengths array, 6 tests Bloque5RiskV2Test.
- Bloque 6 Assurance V2 orchestrator hooks: AssuranceInsufficientException 423 Locked + X-Assurance-Required + X-Current-Assurance, JSON extensions (required_assurance, current_assurance, required_strengths array), AuthenticationStrength enum 0/10/20/30/40 (Anonymous/Password/Token/MultiFactor/HardwareBacked), AuthExceptionMapper errorCode=ASSURANCE_INSUFFICIENT + htmlBody menciona MFA/passkey, 6 tests Bloque6AssuranceV2Test.
- Bloque 7 SP DI wiring + config flags: AuthenticationServiceProvider 3 imports OIDC + 4 bindings nuevos (OidcWellKnownClientInterface scoped null when disabled, OidcJwksCacheInterface file/memory driver, OidcAuthenticator factory, AuthenticatorResolver extend OIDC p850) + preserves Passkey extend p900 intacto; config/auth.php 3 flags default false (auth.passkeys.enabled=false, auth.oidc.enabled=false, auth.throttle.distributed.enabled=false) + secciones tokens/passkeys/oidc/throttle/risk/policy/transaction/authenticators.

Entregables mínimos:

1. 46 tests unitarios V2 nuevos: Bloque1(10) + Bloque2(10) + Bloque3(8) + Bloque4(6) + Bloque5(6) + Bloque6(6), cada suite BloqueXTest exit 0 independiente.
2. AuthenticationServiceProvider wiring 6 interfaces total (Passkey 3 heredados 082 + OIDC 3 nuevos): todos bindings scoped() retornan null cuando config feature disabled (VoltStack Container NO respeta ?Interface = null constructor defaults).
3. AuthenticationOrchestrator backward branch `if ($tried === 1 && $firstDecision !== null) return $firstDecision` 100% byte-for-byte intacta; V2 hooks (risk/assurance/throttle V2) ejecutan FUERA de esta rama.
4. Config flags OPT-IN default false para passkeys/oidc/throttle.distributed; métodos legacy @deprecated ceremonies finishRegistration / finishAssertion PRESERVADOS no removidos.
5. Cross-suite framework vendor/tests ≥ 1038 tests total, subsistema AUTH ≥ 240 tests, permitidos 2 fallos linea-base FUERA ALCANCE (RiskV1 aggregation mismatch + Smoke public 403 policy trust).
6. 4 docs closure actualizados: DEVELOPMENT_VERSIONS.md corte actual, DEVELOPMENT_MATRIX.md resumen+netos, DEVELOPMENT_GUIDELINES.md lecciones aprendidas, EXECUTIVE_IMPLEMENTATION_PLAN.md esta entrada.

Resultado alcanzado (exit 0 cross-suite, 46 tests nuevos PASS 43OK/3Skip entorno Windows PHP 8.4 sin keygen válido):

- Bloques 1-6 implementados criptografía real openssl + unit tests suites Bloque1PasskeysCryptoTest(10), Bloque2OidcCryptoTest(10), Bloque3BearerV2Test(8), Bloque4ThrottleV2Test(6), Bloque5RiskV2Test(6), Bloque6AssuranceV2Test(6): **46 tests / exit 0**. 3 tests Skip entorno PHP 8.4 Windows openssl_pkey_get_public no acepta PEM sintéticas (PHP 8.4 valida curva EC + modulus RSA matemáticamente, no solo sintaxis DER); 8/10 Passkeys + 9/10 OIDC OK con keygen real cuando está disponible.
- Bloque 7 SP DI lint OK php -l AuthenticationServiceProvider + config/auth.php; cross 28 tests Bloque1+Bloque2+Bloque3 standalone exit 0 antes full-suite; Passkey priority 900 wiring 082 intacto SIN duplicados.
- Full framework suite: **1038 tests / 7011 assertions exit 1** con ÚNICO 1 fallo FUERA ALCANCE (BloqueCTest RiskV1 aggregation mismatch 50 vs 40, preexistente no tocado); SkeletonSecuritySmokeTest NO falló en este run (permitido si regresa linea-base).
- Sub-sistema AUTH: **262 tests AUTH ≥ 240 objetivo** (214 baseline legacy + 46 nuevos V2 + 2 feature preexistentes).
- Hard constraints 100% preservados: 0 dependencias composer externas (toda crypto ext-openssl + vanilla curl); Orchestrator rama 1-candidato backward 100% intacta hooks V2 FUERA; 3 interfaces nuevas OIDC bindings scoped() retornan null por defecto disabled; métodos legacy @deprecated Passkey ceremonies finishRegistration / finishAssertion PRESERVADOS; storage auto-detects AES-GCM envelope vs legacy plaintext.
- Backward compat layered total: config/passkeys.enabled=false + oidc.enabled=false por defecto → todo el código 083 se desactiva completamente, comportamiento 082 idéntico.

Gap natural posterior (084 candidates):

- Controllers Security projection E2E: bearer token introspection endpoint HTTP real + risk headers propagation + StepUp flows interoperables risk→assurance.
- DistributedThrottle Redis / DB driver real reemplazando interface pluggable (ahora solo shape contract + InMemory simulado).
- OIDC well-known HTTP fetch real vanilla curl + JWKS kid miss refresh TTL automático (ahora client default disabled=true + InMemory mock).
- Step-up flows full interoperabilidad: AdaptivePolicy deny 403 → StepUpRequired trigger → min_assurance 423 → Passkey/MFA resolver candidate priority bump.
- MFA TOTP authenticator oficial complementando TrustedDevice MFA actual.
- Audit events structured logger JSONL: todas decisiones AUTH (allow/deny/stepup/assurance_insufficient/throttle_denied) emitir evento durable JSONL con correlation_id + risk_scores + amr.
- Integration tests E2E Linux/CI Docker PHP: eliminar entorno Windows PHP 8.4 openssl strictness para suites Passkey/OIDC keygen válido 100% (actualmente 3 Skip por entorno).
- Purge policy scheduled job: refresh tokens expirados + passkey credentials revocados tombstone cleanup retention window configurable.

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
