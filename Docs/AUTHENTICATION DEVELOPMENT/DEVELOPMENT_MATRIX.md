# DEVELOPMENT_MATRIX

## Proposito

Esta matriz controla el estado real del desarrollo del subsistema `Quantum/Auth` de VoltStack frente a la documentacion arquitectonica ubicada en `vendor/voltstack/authentication-lab/Docs`.

El criterio del corte es conservador y se basa en evidencia visible en:

- `vendor/voltstack/framework/src/Quantum/Auth`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/src/Helper/helpers.php`
- `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- piezas de integracion adyacente en `vendor/voltstack/framework/src/Quantum/Controllers/Security`

## Leyenda

- `Operativo`: existe implementacion usable dentro de `Quantum/Auth` o integracion directa del framework con evidencia clara en runtime y pruebas.
- `Parcial`: existe base de codigo, helper, binding o integracion adyacente, pero falta el cierre funcional del bloque segun la arquitectura objetivo.
- `Pendiente`: no hay evidencia suficiente en este corte para considerarlo implementado.

## Nota de alcance

La documentacion `00_AUTHENTICATION_PROJECT_CONTEXT.md` se usa como contexto base y no se contabiliza como bloque de implementacion en esta matriz.

## Resumen del corte

| Estado           | Cantidad |
| ---------------- | -------: |
| Operativo        |        0 |
| Parcial          |       21 |
| Pendiente        |       29 |
| Total documentos |       50 |

## Matriz 01-50

| Doc | Area | Estado | Evidencia visible | Gap principal |
| --: | ---- | ------ | ----------------- | ------------- |
| 01 | Authentication Architecture | Parcial | `Quantum/Auth/AuthManager.php`, `AuthenticationServiceProvider.php`, `Contracts/*`, `Context/*`, `Runtime/*`, `Authenticators/*`, `Identity/*`, `Sessions/*`, `Quantum/Middlewares/AuthMiddleware.php`, helper `auth()`, facade `Auth`, `config/auth.php` | Falta governance del sistema, politica de fallos mas rica y coordinacion distribuida |
| 02 | Domain Model And Core Concepts | Parcial | `IdentityInterface`, `IdentityIdentifier`, `IdentityReference`, `GenericIdentity`, `AuthenticationContext`, `AuthenticationDecision`, `AuthDomainModelTest.php` | Falta evidence model, claims, factors y separacion completa de aggregates del dominio |
| 03 | Lifecycle And Request Pipeline | Parcial | `AuthenticationRequest`, `AuthenticationContextAccessor`, `AuthenticationOrchestrator`, `DefaultAuthenticatorResolver`, `PasswordAuthenticator`, `SessionAuthenticator`, `AuthenticationResponseDecorator`, autenticacion y recovery entre requests en `AuthManagerTest.php` | Falta pipeline completo con firewall, continuation state y eventos del lifecycle |
| 04 | Manager And Orchestration System | Parcial | `AuthenticationManagerInterface`, `AuthenticationOrchestratorInterface`, `AuthenticationOperationContext`, `AuthenticationOrchestrator`, `AuthManager` con `attempt()`, `login()` y `logout()` persistido | Falta stage pipeline formal, `login()` mas expresivo y operaciones avanzadas de governance |
| 05 | Firewall Guard And Context Resolution | Parcial | `AuthenticationContextAccessor` resuelve y limpia contexto autenticado request-scoped | Falta resolver real de firewall y guard, ademas de recovery formal desde session/token |
| 06 | Authenticator System | Parcial | `Contracts/AuthenticatorInterface.php`, `Contracts/MultiFactorIdentityProviderInterface.php`, `Authenticators/PasswordAuthenticator.php`, `Authenticators/SessionAuthenticator.php` | Faltan extension points avanzados del lifecycle y authenticators adicionales |
| 07 | Authenticator Resolution Selection And Priority | Parcial | `Contracts/AuthenticatorResolverInterface.php`, `Runtime/DefaultAuthenticatorResolver.php`, `DefaultAuthenticatorResolverTest.php` | Falta prioridad configurable, composicion dinamica y manejo de ambiguedad |
| 08 | Passport Credential And Evidence System | Parcial | `Credentials/PasswordCredentials.php` tipa el ingreso de credenciales para password auth | Falta `AuthenticationPassport`, evidencia verificada, claims y ensamblado formal de evidence |
| 09 | Identity Model Provider Resolution And Federated Mapping | Parcial | `Contracts/IdentityProviderInterface.php`, `Identity/LocalIdentityProvider.php`, `IdentityInterface`, `GenericIdentity` | Falta resolver multiples providers, mapping federado y referencias externas canonicas |
| 10 | Identity Security State Account Status And Authentication Eligibility | Parcial | `Identity/IdentitySecurityState.php`, `IdentityProviderInterface::securityStateFor()`, `LocalIdentityProvider`, rechazo por identidad inelegible en `PasswordAuthenticator`, tests unitarios y feature | Falta versionado de seguridad, lockout policies y reglas mas ricas de elegibilidad |
| 11 | Password Authentication Hashing Policy And Credential Lifecycle | Parcial | `Contracts/PasswordPolicyInterface.php`, `Contracts/PasswordRehashingIdentityProviderInterface.php`, `Contracts/MultiFactorIdentityProviderInterface.php`, `Passwords/PasswordPolicy.php`, `Authenticators/PasswordAuthenticator.php`, `Identity/LocalIdentityProvider.php`, `Credentials/PasswordCredentials.php`, `config/auth.php`, tests unitarios y feature de policy/rehash/MFA local | Falta expiracion y ciclo de vida formal de credenciales mas alla del rehash inicial |
| 12 | Session Authentication Persistence Context Restoration And Session Lifecycle | Parcial | `Sessions/AuthenticationSession.php`, `AuthenticationSessionId.php`, `InMemoryAuthenticationSessionRepository.php`, `FileAuthenticationSessionRepository.php`, `Contracts/AuthenticationSessionRepositoryInterface.php`, `Authenticators/SessionAuthenticator.php`, `Support/AuthenticationAssurance.php`, expiracion configurable, `rotate_on_recover`, `revoke_others_on_login`, purga de expiradas y tests feature de restauracion/logout/rotation/revocation/MFA recovery | Faltan stores distribuidos, revocacion remota y coordinacion multi-nodo |
| 13 | Remember Me Persistent Login And Long Lived Authentication Credential | Pendiente | Sin evidencia suficiente | Falta modelo de credencial persistente independiente de session |
| 14 | Token Bearer API And Stateless Authentication | Pendiente | Solo existe infraestructura adyacente de `AuthenticationStrength` en Controllers Security | Falta autenticacion stateless real en `Quantum/Auth`, verificacion de tokens y resolucion de identidad |
| 15 | Multi Factor Authentication Factor Orchestration And Step Up | Pendiente | Solo hay semantica adyacente de `AuthenticationStrength::MultiFactor` | Falta sistema MFA, factor orchestration, challenge y step-up |
| 16 | Passkey WebAuthn FIDO2 And Phishing Resistant Authentication | Pendiente | Sin evidencia suficiente | Faltan credenciales WebAuthn, verifier, RP config y ceremonies |
| 17 | OAuth2 OpenID Connect Social Login And Federated Authentication | Pendiente | Sin evidencia suficiente | Falta federation layer, OIDC verification, mapping y sessions derivadas |
| 18 | Account Recovery Password Reset Identity Recovery And Credential Reestablishment | Pendiente | Sin evidencia suficiente | Faltan recovery flows, tokens, governance y reestablecimiento seguro |
| 19 | Throttling Rate Limiting Brute Force Credential Stuffing And Abuse Protection | Pendiente | Sin evidencia suficiente | Falta abuse protection manager, counters y decisiones pre/post authentication |
| 20 | Risk Engine Adaptive Authentication And Security Signal | Pendiente | Sin evidencia suficiente | Faltan signal providers, risk engine y politicas adaptativas |
| 21 | Device Trust Device Identity And Trusted Device Credential | Pendiente | Sin evidencia suficiente | Faltan device references, trusted device credentials y evaluacion de confianza |
| 22 | Login Logout Sign In Sign Out Entry Point And User Authentication Flow | Parcial | `AuthManager` soporta `attempt()`, `attemptOrFail()`, `login()`, `logout()`, `setUser()` y `context()`; facade `Auth`; `Quantum/Middlewares/AuthMiddleware.php`; `Quantum/Middlewares/GuestMiddleware.php`; `AuthManagerTest.php` cubre login valido, invalido, policy, restauracion, expiracion, rotacion, revocacion, logout, ruta protegida, ruta guest-only, denial stale-session, assurance insuficiente por metadata, assurance explicito en contexto y login MFA local | Faltan entry points adicionales y politicas de sign-in/sign-out mas ricas |
| 23 | Events Hooks Listeners Subscribers And Extension Lifecycle | Pendiente | Sin evidencia suficiente | Faltan eventos propios de Authentication y puntos de extension del lifecycle |
| 24 | Audit Observability Logging Metrics Tracing And Explainability | Pendiente | Sin evidencia suficiente dentro de `Quantum/Auth` | Faltan audit trail, logs estructurados, metricas y explicabilidad de decisiones |
| 25 | Failure Error Exception Denial And Security Response Handling | Parcial | `Exceptions/AuthenticationException.php`, `InvalidCredentialsException.php`, `IdentityNotEligibleException.php`, `AuthenticationRequiredException.php`, `GuestOnlyException.php`, `StaleAuthenticationSessionException.php`, `SecondFactorRequiredException.php`, `InvalidSecondFactorException.php`, `AuthExceptionMapper.php`, `attemptOrFail()`, `Quantum/Middlewares/AuthMiddleware.php`, `Quantum/Middlewares/GuestMiddleware.php`, integracion en `Quantum/Exceptions/ExceptionHandler.php` con `401/403`, `WWW-Authenticate`, `X-Volt-Error-Code` y reuse de `controller.security.authentication.authentication_strength_insufficient` | Falta mapa mas completo de errores, denial policies diferenciadas y respuestas especificas por mecanismo |
| 26 | Testing Verification Security Assurance And Conformance | Parcial | `tests/Feature/AuthManagerTest.php`, `tests/Unit/AuthDomainModelTest.php`, `tests/Unit/LocalIdentityProviderTest.php`, `tests/Unit/AuthenticationSessionRepositoryTest.php`, `tests/Unit/DefaultAuthenticatorResolverTest.php`, `tests/Unit/FileAuthenticationSessionRepositoryTest.php`, `tests/Unit/PasswordPolicyTest.php` validan scope por request, facade, provider local, elegibilidad, policy, resolver, persistencia, rotation y revocation | Falta harness formal, concurrencia, endurecimiento y conformance |
| 27 | Compilation Configuration Validation Cache Optimization And Runtime Performance | Parcial | existe `config/auth.php` con carga por `Bootstrapper::loadConfiguration()` y uso operativo en Auth/session | Faltan validacion, caches y optimizacion/metadata del subsistema |
| 28 | Extensibility Plugin Provider Custom Authenticator And Integration | Pendiente | Sin evidencia suficiente | Faltan registries, extension points y providers customizables |
| 29 | Multi Tenancy Security Realms Cross Tenant Isolation And Tenant Authentication Policy | Pendiente | Sin evidencia suficiente | Faltan tenant context, realm resolution y aislamiento cross-tenant |
| 30 | Distributed System Cluster Session Coordination Revocation Consistency And Multi Node Runtime | Pendiente | Sin evidencia suficiente | Faltan stores distribuidos, consistency policies y coordinacion multi-node |
| 31 | Cryptographic Key Secret Certificate Trust And Key Lifecycle Management | Pendiente | Sin evidencia suficiente | Faltan key stores, trust policies, certificados y rotacion |
| 32 | Privileged Administrative Break Glass Sensitive Operation And Reauthentication | Pendiente | Sin evidencia suficiente | Faltan flows privilegiados, break-glass y reautenticacion reforzada |
| 33 | Service Workload Machine To Machine And Non Human Identity | Pendiente | Sin evidencia suficiente | Faltan principals no humanos, workload identity y machine credentials |
| 34 | Identity Linking Account Linking Credential Binding And Authentication Method Management | Pendiente | Sin evidencia suficiente | Faltan account linking, credential binding y gestion de metodos |
| 35 | Session Device Credential Inventory Security Center And User Security Management | Pendiente | Sin evidencia suficiente | Faltan inventarios de sesion, dispositivo y centro de seguridad del usuario |
| 36 | Policy Engine Requirement Composition Security Posture And Authentication Governance | Pendiente | Sin evidencia suficiente | Falta `AuthenticationPolicyEngine` y composicion formal de requisitos |
| 37 | Assurance Level Authentication Context And Trust Classification | Parcial | `Support/AuthenticationAssurance.php`, `AuthenticationContext::authenticationStrength()`, `AuthenticationContext::authenticationAssuranceProfile()`, `Quantum/Middlewares/AuthMiddleware.php` interpreta `minimum_strength` desde metadata de ruta `auth`, reutiliza `AuthenticationStrength` + `AuthenticationRequiredException` de Controllers Security y conserva `amr` MFA local | Falta persistencia de evidencias ricas y step-up real |
| 38 | Challenge Negotiation Continuation And Interactive Flow | Pendiente | Sin evidencia suficiente | Faltan transactions, continuation state y challenge negotiation |
| 39 | Transaction State Nonce Replay Protection CSRF Binding And Cryptographic Continuation Security | Pendiente | Sin evidencia suficiente | Faltan nonce store, replay protection y seguridad criptografica de continuaciones |
| 40 | Security Notification Alerting Compromise Detection Incident Response And Account Protection | Pendiente | Sin evidencia suficiente | Faltan alertas, compromise detection y respuestas de proteccion de cuenta |
| 41 | Identity Lifecycle Account State Suspension Lockout Deactivation Deletion And Reactivation | Pendiente | Sin evidencia suficiente | Faltan lifecycle states y reglas de suspension/reactivacion |
| 42 | Privacy Data Minimization Retention Consent And Security Metadata Governance | Pendiente | Sin evidencia suficiente | Faltan politicas de minimizacion, retencion y gobierno de metadatos sensibles |
| 43 | Notification Security Communication Out Of Band Channel And Delivery Governance | Pendiente | Sin evidencia suficiente | Faltan canales out-of-band, delivery governance y contratos de notificacion |
| 44 | Authentication Background Processing Async Security Task Maintenance Cleanup And Scheduled Operation | Pendiente | Sin evidencia suficiente | Faltan jobs de mantenimiento, limpieza y tareas asincronas del subsistema |
| 45 | Rate Capacity Resource Governance Abuse Prevention And Denial Of Service Resilience | Pendiente | Sin evidencia suficiente | Faltan budgets de recurso, control de capacidad y resistencia DoS especifica |
| 46 | Migration Legacy Credential Import Backward Compatibility And Progressive Security Upgrade | Pendiente | Sin evidencia suficiente | Faltan migracion de credenciales legadas y upgrade progresivo |
| 47 | Developer Experience Facade Helper Configuration Bootstrap And Application Integration | Parcial | helper `auth()`, facade `Quantum/Facades/Auth.php`, `config/auth.php`, `AuthenticationServiceProvider.php`, `Quantum/Middlewares/AuthMiddleware.php`, `Quantum/Middlewares/GuestMiddleware.php`, `Application.php` delega el wiring del subsistema al provider, `auth` respeta `minimum_strength` y el contexto expone assurance profile | Falta una API publica todavia mas coherente y helpers/aliases complementarios |
| 48 | Administration Operational Tooling Diagnostics Security Operations And Production Management | Pendiente | Sin evidencia suficiente | Faltan comandos, diagnosticos y tooling operacional del sistema Auth |
| 49 | Reference Implementation Default Components Secure Defaults And Framework Integration | Parcial | Existe base integrada con `AuthManager`, `AuthenticationServiceProvider`, `AuthenticationContextAccessor`, `AuthenticationOrchestrator`, `DefaultAuthenticatorResolver`, `LocalIdentityProvider`, `PasswordAuthenticator`, `SessionAuthenticator`, `PasswordPolicy`, `Support/AuthenticationAssurance.php`, `AuthExceptionMapper.php`, `Quantum/Middlewares/AuthMiddleware.php`, `Quantum/Middlewares/GuestMiddleware.php`, `config/auth.php`, repositorio de session en memoria/archivo y errores propios de Authentication, incluyendo `stale_session`, strength insufficiente y MFA local | Faltan stores mas robustos, aliases complementarios y testing utilities |
| 50 | System Integration And Final Architecture | Parcial | Hay integracion minima entre contracts, contextos, authenticators, sessions, `AuthenticationServiceProvider`, kernel HTTP, middlewares `auth/guest`, denials `guest_only/stale_session`, assurance por metadata de ruta, assurance profile explicito, MFA local y pruebas del subsistema Auth | Falta el cierre end-to-end del sistema `Quantum/Auth` como plataforma coherente de autenticacion |

## Bloque actualmente visible en codigo

### 1. Base minima de Auth

- `Quantum\Auth\AuthManager`
- helper global `auth()`
- `AuthenticationServiceProvider.php`
- prueba feature del scope por request

Capacidad actual:

- almacenar un usuario arbitrario en el `RuntimeContext`,
- consultar `user/check/guest/id`,
- limpiar el estado con `logout()`.

### 2. Lenguaje base del subsistema

- `Contracts/AuthenticationManagerInterface.php`
- `Contracts/AuthenticationOrchestratorInterface.php`
- `Identity/IdentityInterface.php`
- `Identity/IdentityIdentifier.php`
- `Identity/IdentityReference.php`
- `Identity/GenericIdentity.php`
- `Context/AuthenticationRequest.php`
- `Context/AuthenticationContext.php`
- `Context/AuthenticationContextAccessor.php`
- `Decisions/AuthenticationDecision.php`
- `Decisions/AuthenticationDecisionStatus.php`
- `Runtime/AuthenticationOperationContext.php`
- `Runtime/AuthenticationOrchestrator.php`
- `tests/Unit/AuthDomainModelTest.php`

Capacidad actual:

- representar identidad canonica minima,
- representar `AuthenticationContext` y `AuthenticationDecision`,
- resolver contexto autenticado request-scoped,
- mantener compatibilidad con la API previa de `AuthManager`.

### 3. Primer flujo real de password authentication

- `Contracts/AuthenticatorInterface.php`
- `Contracts/IdentityProviderInterface.php`
- `Credentials/PasswordCredentials.php`
- `Authenticators/PasswordAuthenticator.php`
- `Identity/LocalIdentityProvider.php`
- `AuthManager::attempt()`
- `tests/Unit/LocalIdentityProviderTest.php`
- nuevos casos en `tests/Feature/AuthManagerTest.php`

Capacidad actual:

- autenticar por `identifier/email/username + password`,
- resolver identidad local desde configuracion,
- verificar `password_hash` con `password_verify()`,
- poblar `AuthenticationContext` autentico cuando el login es valido.

### 4. Session authentication minima

- `Sessions/AuthenticationSession.php`
- `Sessions/AuthenticationSessionId.php`
- `Sessions/InMemoryAuthenticationSessionRepository.php`
- `Contracts/AuthenticationSessionRepositoryInterface.php`
- `Authenticators/SessionAuthenticator.php`
- `Runtime/AuthenticationResponseDecorator.php`
- soporte de cookies en `Quantum/Http/Request.php`
- pruebas de restauracion y logout en `tests/Feature/AuthManagerTest.php`

Capacidad actual:

- persistir un login autenticado entre requests usando session id,
- restaurar `AuthenticationContext` desde cookie/header de session,
- invalidar la session en `logout()`,
- emitir `Set-Cookie` y `X-Auth-Session` en el ciclo HTTP.

### 5. DX, resolver y configuracion inicial

- `Contracts/AuthenticatorResolverInterface.php`
- `Runtime/DefaultAuthenticatorResolver.php`
- `Quantum/Facades/Auth.php`
- `config/auth.php`
- `AuthenticationServiceProvider.php`
- expiracion configurable de session en `AuthManager`
- pruebas `DefaultAuthenticatorResolverTest.php` y casos facade/expiry en `AuthManagerTest.php`

Capacidad actual:

- separar la seleccion de authenticators del orchestrator,
- exponer API estatica ergonomica via `Auth`,
- cargar configuracion inicial de auth desde `config/auth.php`,
- endurecer la session con expiracion configurable y limpieza de cookie al expirar,
- y delegar el wiring del subsistema a un provider dedicado.

### 6. Elegibilidad, errores y storage configurable

- `Identity/IdentitySecurityState.php`
- `Exceptions/AuthenticationException.php`
- `Exceptions/InvalidCredentialsException.php`
- `Exceptions/IdentityNotEligibleException.php`
- `Sessions/FileAuthenticationSessionRepository.php`
- `IdentityProviderInterface::securityStateFor()`
- integracion en `Quantum/Exceptions/ExceptionHandler.php`
- `attemptOrFail()` en `AuthManager`

Capacidad actual:

- rechazar identidades bloqueadas, suspendidas o deshabilitadas,
- emitir errores propios de Authentication con `401` y `WWW-Authenticate`,
- exponer `attemptOrFail()` para flujos que necesitan semantica de excepcion,
- y elegir storage de session `memory` o `file` por configuracion.

### 7. Password policy y lifecycle hardening

- `Contracts/PasswordPolicyInterface.php`
- `Contracts/PasswordRehashingIdentityProviderInterface.php`
- `Passwords/PasswordPolicy.php`
- `Identity/LocalIdentityProvider.php`
- extensiones de `AuthenticationSessionRepositoryInterface`
- `purgeExpired()` y `deleteForIdentity()` en repositorios `memory/file`
- `rotate_on_recover` y `revoke_others_on_login` en `config/auth.php`
- rotacion y revocacion integradas en `AuthManager`

Capacidad actual:

- validar credenciales de password contra una policy explicita,
- rehashear y persistir hash actualizado cuando el provider local usa `storage_path`,
- purgar sesiones expiradas de forma proactiva,
- rotar la session al recuperarla si la configuracion lo exige,
- y revocar otras sesiones del mismo identity al hacer login.

### 8. Provider y middleware de integracion

- `AuthenticationServiceProvider.php`
- `Quantum/Middlewares/AuthMiddleware.php`
- `Quantum/Middlewares/GuestMiddleware.php`
- alias `auth` resuelto por `MiddlewareAliasRegistry`
- alias `guest` resuelto por `MiddlewareAliasRegistry`
- pruebas feature de ruta protegida en `AuthManagerTest.php`

Capacidad actual:

- registrar el subsistema Authentication desde un provider dedicado,
- proteger rutas HTTP con los aliases `auth` y `guest`,
- distinguir `guest_only` y `stale_session` en entry points HTTP,
- distinguir assurance insuficiente con `minimum_strength` en metadata de ruta,
- exponer assurance profile explicito desde `AuthenticationContext`,
- y mantener el wiring del framework alineado con el patron de providers existente.

### 9. Infraestructura adyacente ya presente en el framework

- `AuthenticationRequired` en Controllers Security
- `AuthenticationStrength`
- `AuthenticationRequiredException`
- mapeo de errores de seguridad con `401` y `WWW-Authenticate`

Estas piezas son utiles para integracion futura, pero no sustituyen el subsistema `Quantum/Auth`.

## Faltantes de mayor impacto

### Prioridad alta

1. `02`, `03`, `04`, `05`, `06`, `07`, `08`, `09`
2. `10`, `11`, `12`, `22`, `25`
3. `47`, `49`

Motivo:

- ya existe lenguaje base del subsistema,
- ya existe autenticacion real minima por password sobre provider local,
- ya existe session auth minima entre requests,
- ya existe resolver, facade y configuracion inicial,
- ya existe elegibilidad minima, failure handling basico y driver file,
- ya existe password policy y hardening basico del lifecycle de session,
- pero aun faltan stores distribuidos, lifecycle completo de credenciales y endurecimiento adicional.

### Prioridad media

1. `25`, `26`, `36`, `37`
2. `13`, `14`, `15`, `19`, `20`, `21`
3. `29`, `30`

### Prioridad posterior

1. `16`, `17`, `18`
2. `31` a `35`
3. `38` a `46`
4. `48`, `50`

## Orden recomendado para seguir desarrollando

1. consolidar `11 + 12 + 22 + 49`,
2. despues abrir `26 + 36 + 37`,
3. continuar con `13 + 14 + 15 + 19 + 20 + 21`,
4. y finalmente expandir federation, passkeys, distributed runtime y operacion avanzada.

## Regla de mantenimiento

Cada vez que se cierre un bloque relevante del subsistema Authentication, esta matriz debe actualizar:

1. el estado del documento impactado,
2. la evidencia visible en codigo y tests,
3. el gap principal restante,
4. la prioridad de trabajo posterior.
