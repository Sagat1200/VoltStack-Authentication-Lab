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
| Parcial          |       28 |
| Pendiente        |       22 |
| Total documentos |       50 |

## Matriz 01-50

| Doc | Area | Estado | Evidencia visible | Gap principal |
| --: | ---- | ------ | ----------------- | ------------- |
| 01 | Authentication Architecture | Parcial | `Quantum/Auth/AuthManager.php`, `AuthenticationServiceProvider.php`, `Contracts/*`, `Context/*`, `Runtime/*`, `Authenticators/*`, `Identity/*`, `Sessions/*`, `Quantum/Middlewares/AuthMiddleware.php`, helper `auth()`, facade `Auth`, `config/auth.php` | Falta governance del sistema, politica de fallos mas rica y coordinacion distribuida |
| 02 | Domain Model And Core Concepts | Parcial | `IdentityInterface`, `IdentityIdentifier`, `IdentityReference`, `GenericIdentity`, `AuthenticationContext`, `AuthenticationDecision`, `AuthDomainModelTest.php`; `AuthenticationContext` ya expone claims administrativas compartidas (`managementAuthority`, `managementOwnershipProof`, `managementScopes`) para self-service remoto coherente entre Auth y Controllers Security, gobierna claims privilegiadas (`managementClaimsSource`, `managementPrivilegeLevel`) con fallback explicito a atributos de identidad, centraliza una decision compartida de management gobernado (`hasGovernedManagementClaims`, `canAdministrativelyManageDevices`, `managementAuthorizationMode`, `managementAuthorizationReasonCode`) reutilizada por el tooling operativo, y ahora distingue autorizacion por alcance con `canAdministrativelyManageDeviceSessions()`, `managementSessionAuthorizationMode()`, `managementSessionAuthorizationReasonCode()`, `canAdministrativelyManageTrustedDevices()`, `managementTrustedDeviceAuthorizationMode()` y `managementTrustedDeviceAuthorizationReasonCode()` | Falta evidence model mas rico, claims privilegiadas multi-actor, factors y separacion completa de aggregates del dominio |
| 03 | Lifecycle And Request Pipeline | Parcial | `AuthenticationRequest`, `AuthenticationContextAccessor`, `AuthenticationOrchestrator`, `DefaultAuthenticatorResolver`, `PasswordAuthenticator`, `SessionAuthenticator`, `AuthenticationResponseDecorator`, autenticacion y recovery entre requests en `AuthManagerTest.php` | Falta pipeline completo con firewall, continuation state y eventos del lifecycle |
| 04 | Manager And Orchestration System | Parcial | `AuthenticationManagerInterface`, `AuthenticationOrchestratorInterface`, `AuthenticationOperationContext`, `AuthenticationOrchestrator`, `AuthManager` con `attempt()`, `login()` y `logout()` persistido | Falta stage pipeline formal, `login()` mas expresivo y operaciones avanzadas de governance |
| 05 | Firewall Guard And Context Resolution | Parcial | `AuthenticationContextAccessor` resuelve y limpia contexto autenticado request-scoped; `Quantum/Middlewares/AuthMiddleware.php` y `Quantum/Middlewares/MfaMiddleware.php` exponen entry points HTTP para auth y elevation | Falta resolver real de firewall y guard, ademas de recovery formal desde session/token |
| 06 | Authenticator System | Parcial | `Contracts/AuthenticatorInterface.php`, `Contracts/MultiFactorIdentityProviderInterface.php`, `Authenticators/PasswordAuthenticator.php`, `Authenticators/SessionAuthenticator.php`, `Runtime/DefaultAuthenticatorResolver.php` soportando `authenticate`, `recover` y `step_up` | Faltan extension points avanzados del lifecycle y authenticators adicionales |
| 07 | Authenticator Resolution Selection And Priority | Parcial | `Contracts/AuthenticatorResolverInterface.php`, `Runtime/DefaultAuthenticatorResolver.php`, `DefaultAuthenticatorResolverTest.php` | Falta prioridad configurable, composicion dinamica y manejo de ambiguedad |
| 08 | Passport Credential And Evidence System | Parcial | `Credentials/PasswordCredentials.php` tipa el ingreso de credenciales para password auth y `second_factor`/`otp` en login y `step_up` | Falta `AuthenticationPassport`, evidencia verificada, claims y ensamblado formal de evidence |
| 09 | Identity Model Provider Resolution And Federated Mapping | Parcial | `Contracts/IdentityProviderInterface.php`, `Identity/LocalIdentityProvider.php`, `IdentityInterface`, `GenericIdentity` | Falta resolver multiples providers, mapping federado y referencias externas canonicas |
| 10 | Identity Security State Account Status And Authentication Eligibility | Parcial | `Identity/IdentitySecurityState.php`, `IdentityProviderInterface::securityStateFor()`, `LocalIdentityProvider`, rechazo por identidad inelegible en `PasswordAuthenticator`, tests unitarios y feature | Falta versionado de seguridad, lockout policies y reglas mas ricas de elegibilidad |
| 11 | Password Authentication Hashing Policy And Credential Lifecycle | Parcial | `Contracts/PasswordPolicyInterface.php`, `Contracts/PasswordRehashingIdentityProviderInterface.php`, `Contracts/MultiFactorIdentityProviderInterface.php`, `Passwords/PasswordPolicy.php`, `Authenticators/PasswordAuthenticator.php`, `Identity/LocalIdentityProvider.php`, `Credentials/PasswordCredentials.php`, `config/auth.php`, tests unitarios y feature de policy/rehash/MFA local y `step_up` | Falta expiracion y ciclo de vida formal de credenciales mas alla del rehash inicial |
| 12 | Session Authentication Persistence Context Restoration And Session Lifecycle | Parcial | `Sessions/AuthenticationSession.php`, `AuthenticationSessionId.php`, `AuthenticationSessionPublicId.php`, `AuthenticationSessionRecoveryReason.php`, `AuthenticationSessionSummary.php`, `InMemoryAuthenticationSessionRepository.php`, `FileAuthenticationSessionRepository.php`, `Contracts/AuthenticationSessionRepositoryInterface.php`, `Authenticators/SessionAuthenticator.php`, `Support/AuthenticationAssurance.php`, expiracion configurable, `rotate_on_recover`, `revoke_others_on_login`, `session_public_id`, `touch()` de sesion, tombstones minimos de recovery, retencion configurable de tombstones, inventory con metadata reducida de session/device, enumeracion global `all()` para tooling/reconciliacion, `authentication_fresh_at` y tests feature/unit de restauracion/logout/rotation/revocation/MFA recovery/step-up recovery | Faltan stores distribuidos reales, policy/authorization mas rica, retencion gobernada mas completa y coordinacion multi-nodo |
| 13 | Remember Me Persistent Login And Long Lived Authentication Credential | Pendiente | Sin evidencia suficiente | Falta modelo de credencial persistente independiente de session |
| 14 | Token Bearer API And Stateless Authentication | Pendiente | Solo existe infraestructura adyacente de `AuthenticationStrength` en Controllers Security | Falta autenticacion stateless real en `Quantum/Auth`, verificacion de tokens y resolucion de identidad |
| 15 | Multi Factor Authentication Factor Orchestration And Step Up | Pendiente | Solo hay semantica adyacente de `AuthenticationStrength::MultiFactor` | Falta sistema MFA, factor orchestration, challenge y step-up |
| 16 | Passkey WebAuthn FIDO2 And Phishing Resistant Authentication | Pendiente | Sin evidencia suficiente | Faltan credenciales WebAuthn, verifier, RP config y ceremonies |
| 17 | OAuth2 OpenID Connect Social Login And Federated Authentication | Pendiente | Sin evidencia suficiente | Falta federation layer, OIDC verification, mapping y sessions derivadas |
| 18 | Account Recovery Password Reset Identity Recovery And Credential Reestablishment | Pendiente | Sin evidencia suficiente | Faltan recovery flows, tokens, governance y reestablecimiento seguro |
| 19 | Throttling Rate Limiting Brute Force Credential Stuffing And Abuse Protection | Pendiente | Sin evidencia suficiente | Falta abuse protection manager, counters y decisiones pre/post authentication |
| 20 | Risk Engine Adaptive Authentication And Security Signal | Pendiente | Sin evidencia suficiente | Faltan signal providers, risk engine y politicas adaptativas |
| 21 | Device Trust Device Identity And Trusted Device Credential | Parcial | `AuthManager` deriva metadata reducida de device (`client_platform`, `device_kind`), `device_reference` pseudonimizado, trusted-device records persistentes via `TrustedDeviceRepositoryInterface`, emite una credencial cliente `publicId.secret` con `credential_hash` persistido server-side, la rota cuando reduce el challenge MFA, conserva `previous_credential_hash` para replay detection, valida esa credencial en runtime mediante `Devices/TrustedDeviceCredentialValidator.php` y refleja `trusted_device_public_id` / `trusted_device_credential_present`; `PasswordAuthenticator.php` usa ese reconocimiento para challenge reduction y `AuthManagerTest.php` valida emision simultanea de cookies, alta/olvido de trusted devices con MFA, rotacion por challenge reduction, replay revocation y rechazo por fingerprint distinto | Faltan referencias mas fuertes, evaluacion/policy de confianza mas rica y governance multi-actor del posture de device |
| 22 | Login Logout Sign In Sign Out Entry Point And User Authentication Flow | Parcial | `AuthManager` soporta `attempt()`, `attemptOrFail()`, `stepUp()`, `stepUpOrFail()`, `login()`, `logout()`, `setUser()`, `context()`, `currentSession()`, `sessions()`, `trustedDevices()`, `devices()`, `managedDevices()`, `trustCurrentDevice()`, `forgetTrustedDevice()`, `revokeDevice()`, `revokeOtherDevices()`, `revokeManagedDevice()`, `revokeSession()` y `revokeOtherSessions()`; facade `Auth`; middlewares `AuthMiddleware`, `GuestMiddleware`, `MfaMiddleware`; `AuthManagerTest.php` cubre casos de uso core, inventory seguro, revocación coordinada, gestión de dispositivos confiables, enforcement de alcance para delegación parcial, y proyección de claims administrativos del actor actual y objetivos gobernados | Faltan entry points adicionales, policy/authorization multi-actor mas profunda en runtime y sign-in/sign-out governance mas completa |
| 23 | Events Hooks Listeners Subscribers And Extension Lifecycle | Pendiente | Sin evidencia suficiente | Faltan eventos propios de Authentication y puntos de extension del lifecycle |
| 24 | Audit Observability Logging Metrics Tracing And Explainability | Parcial | `Quantum/Console/Commands/AuthSecurityCenterReportCommand.php` ya expone reporting operativo seguro del security center, resumen de agregados, conteo de management gobernado, distincion entre `direct_admin` y `delegated_admin`, ahora tambien `management_authorized` y `management_authorization_reason_code` para actores gobernados, y snapshots JSONL durables opcionales via `--export-log`; `Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php` ya puede anexar audit trail JSONL durable via `--audit-log` con motivo de rechazo del actor cuando aplica; `AuthSecurityCenterReportCommandTest.php` y `AuthSecurityCenterRevokeDeviceCommandTest.php` validan detalle operativo, export persistente y trazabilidad durable minima | Faltan logs estructurados mas amplios, metricas, tracing y gobierno mas rico del export/audit operativo |
| 25 | Failure Error Exception Denial And Security Response Handling | Parcial | `Exceptions/AuthenticationException.php`, `InvalidCredentialsException.php`, `IdentityNotEligibleException.php`, `AuthenticationRequiredException.php`, `GuestOnlyException.php`, `StaleAuthenticationSessionException.php`, `RevokedAuthenticationSessionException.php`, `FreshAuthenticationRequiredException.php`, `SecondFactorRequiredException.php`, `InvalidSecondFactorException.php`, `StepUpAuthenticationRequiredException.php`, `SecondFactorNotAvailableException.php`, `StepUpRequiredException.php`, `AuthExceptionMapper.php`, `attemptOrFail()`, `stepUpOrFail()`, `Quantum/Middlewares/AuthMiddleware.php`, `Quantum/Middlewares/GuestMiddleware.php`, `Quantum/Middlewares/MfaMiddleware.php`, integracion en `Quantum/Exceptions/ExceptionHandler.php` con `401/403`, `WWW-Authenticate`, `X-Volt-Error-Code` y denials propios `auth.step_up_required` / `auth.revoked_session` / `auth.fresh_authentication_required` | Falta mapa mas completo de errores, policies de respuesta por mecanismo y challenges mas ricos |
| 26 | Testing Verification Security Assurance And Conformance | Parcial | `tests/Feature/AuthManagerTest.php`, `tests/Feature/SkeletonSecuritySmokeTest.php`, `tests/Unit/AuthDomainModelTest.php`, `tests/Unit/LocalIdentityProviderTest.php`, `tests/Unit/AuthenticationSessionRepositoryTest.php`, `tests/Unit/DefaultAuthenticatorResolverTest.php`, `tests/Unit/FileAuthenticationSessionRepositoryTest.php`, `tests/Unit/PasswordPolicyTest.php`, `tests/Unit/TrustedDeviceRepositoryTest.php`, `tests/Unit/AuthSessionsCleanupCommandTest.php`, `tests/Unit/AuthDevicesReconcileCommandTest.php`, `tests/Unit/AuthSecurityCenterReportCommandTest.php`, `tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`, `tests/Unit/ConsoleApplicationTest.php` y `tests/Unit/ControllerSecurityContextFactoryTest.php` validan scope por request, facade, provider local, elegibilidad, policy, resolver, persistencia, rotation/revocation de session, replay-hardening de trusted-device credentials, lectura de inventory agregado sobre stores compartidos, revocacion coordinada por dispositivo, cleanup operativo extendido, reconciliacion sobre store compartido, reporting operativo seguro del security center, export de actores administrativos gobernados, snapshots JSONL durables del reporte, mutacion administrativa operacional por `identity + device_reference`, gobierno por actor explicito + session publica valida, decision compartida de autorizacion administrativa con motivo de rechazo, proyeccion de hints y claims del actor actual sobre `auth()->devices()`, bypass gobernado de `fresh-auth` para mutaciones remotas por dispositivo agregado, revocacion remota multi-identidad en runtime principal con actor gobernado y actor rechazado, discovery gobernado del target via `managedDevices(identity, type?)`, scope parcial `sessions|trusted-devices|all` para esa mutacion, ownership target explicito (`management_target_identity`, `management_target_type`, `management_target_matches_current_identity`), claims administrativas crudas del actor (`management_actor_authority`, `management_actor_ownership_proof`, `management_actor_claims_source`, `management_actor_privilege_level`, `management_actor_scopes`) y relacion actor-target (`management_actor_target_relation`, `management_actor_target_reason_code`) tanto en inventory propio como gobernado, y ahora delegacion parcial por alcance con validacion de actor pleno, actor solo de `sessions`, actor solo de `trusted-devices` y rechazo operacional del comando cuando el actor no cubre el `scope` pedido | Falta harness formal, concurrencia, endurecimiento mas amplio y conformance |


| 27 | Compilation Configuration Validation Cache Optimization And Runtime Performance | Parcial | existe `config/auth.php` con carga por `Bootstrapper::loadConfiguration()` y uso operativo en Auth/session | Faltan validacion, caches y optimizacion/metadata del subsistema |
| 28 | Extensibility Plugin Provider Custom Authenticator And Integration | Pendiente | Sin evidencia suficiente | Faltan registries, extension points y providers customizables |
| 29 | Multi Tenancy Security Realms Cross Tenant Isolation And Tenant Authentication Policy | Pendiente | Sin evidencia suficiente | Faltan tenant context, realm resolution y aislamiento cross-tenant |
| 30 | Distributed System Cluster Session Coordination Revocation Consistency And Multi Node Runtime | Parcial | `FileAuthenticationSessionRepository.php` conserva tombstones minimos de revocacion/expiracion en storage compartido, `AuthenticationSessionRecoveryReason.php` normaliza recovery outcomes, `AuthManager::devices()` agrega sessions y trusted devices por `device_reference` con hints de ownership (`management_authority`, `management_ownership_proof`), `AuthManager::revokeDevice()` y `AuthManager::revokeOtherDevices()` coordinan mutaciones sobre ese agregado, los repositorios ya exponen `all()` y `AuthDevicesReconcileCommand.php` reconcilia el posture de sesiones frente a trusted devices sobre stores compartidos | Faltan stores distribuidos reales, consistency policies, ownership multi-actor y coordinacion multi-node mas fuerte |
| 31 | Cryptographic Key Secret Certificate Trust And Key Lifecycle Management | Pendiente | Sin evidencia suficiente | Faltan key stores, trust policies, certificados y rotacion |
| 32 | Privileged Administrative Break Glass Sensitive Operation And Reauthentication | Parcial | `FreshAuthenticationRequiredException.php`, `AuthExceptionMapper.php`, `AuthManager` exige freshness configurable para revocacion remota de sesiones, trusted devices y dispositivos agregados via `revokeDevice()` en self-service, pero ahora permite el caso gobernado cuando el actor actual puede administrar dispositivos remotamente via `admin_device_management`; `QuantumExceptionHandlerTest.php` / `AuthManagerTest.php` validan tanto el bloqueo HTTP de operaciones sensibles en self-service como el bypass gobernado sobre mutacion agregada por dispositivo | Faltan flows privilegiados, break-glass, proofs de reauth y reautenticacion reforzada mas general |
| 33 | Service Workload Machine To Machine And Non Human Identity | Pendiente | Sin evidencia suficiente | Faltan principals no humanos, workload identity y machine credentials |
| 34 | Identity Linking Account Linking Credential Binding And Authentication Method Management | Pendiente | Sin evidencia suficiente | Faltan account linking, credential binding y gestion de metodos |
| 35 | Session Device Credential Inventory Security Center And User Security Management | Parcial | `AuthenticationSessionPublicId.php`, `AuthenticationSessionSummary.php`, `TrustedDeviceSummary.php`, `Devices/DeviceInventorySummary.php`, `AuthenticationContext::sessionPublicId()` / `freshAuthenticationAt()` / `deviceReference()` / `deviceTrustState()` / `trustedDeviceCredentialPresent()` / `trustedDevicePublicId()`, `AuthenticationContext::hasGovernedManagementClaims()` / `canAdministrativelyManageDevices()` / `managementAuthorizationMode()` / `managementAuthorizationReasonCode()` / `managementActorTargetRelation()` / `managementActorTargetReasonCode()` / `canAdministrativelyManageDeviceSessions()` / `managementSessionAuthorizationMode()` / `managementSessionAuthorizationReasonCode()` / `canAdministrativelyManageTrustedDevices()` / `managementTrustedDeviceAuthorizationMode()` / `managementTrustedDeviceAuthorizationReasonCode()`, `AuthManager::currentSession()`, `AuthManager::sessions()`, `AuthManager::trustedDevices()`, `AuthManager::devices()`, `AuthManager::managedDevices()`, `AuthManager::trustCurrentDevice()`, `AuthManager::forgetTrustedDevice()`, `AuthManager::revokeDevice()`, `AuthManager::revokeOtherDevices()`, `AuthManager::revokeManagedDevice()`, metadata reducida `client_family/client_platform/device_kind/device_reference/device_trust_state/ip_prefix/label/last_activity_at`, hints `can_revoke/requires_reauthentication/revocation_scope/revocation_mode` para sessions, `can_forget/requires_reauthentication/revocation_scope/revocation_mode` para trusted devices, un agregado por dispositivo con `session_count`, `session_public_ids`, `trusted_device_public_id`, `management_sensitivity`, `management_reason_code`, `management_authority`, `management_ownership_proof` y hints/claims del actor actual (`management_actor_governed`, `management_actor_authorized`, `management_actor_authorization_mode`, `management_actor_authorization_reason_code`, `management_actor_authority`, `management_actor_ownership_proof`, `management_actor_claims_source`, `management_actor_privilege_level`, `management_actor_scopes`, `management_actor_target_relation`, `management_actor_target_reason_code`, `management_actor_can_manage_sessions`, `management_actor_session_authorization_mode`, `management_actor_session_authorization_reason_code`, `management_actor_can_manage_trusted_devices`, `management_actor_trusted_device_authorization_mode`, `management_actor_trusted_device_authorization_reason_code`); ese agregado ahora puede omitir `fresh-auth` remoto para mutaciones por dispositivo cuando el actor actual esta gobernado y autorizado, y el runtime principal ya soporta tanto discovery multi-identidad via `managedDevices(identity, type?)` como mutacion remota multi-identidad sobre stores compartidos via `revokeManagedDevice(identity, device_reference, type?, scope)` con revocacion parcial y autorizacion parcial real de `sessions` o `trusted-devices`; ademas, el summary ya explicita el target del ownership proyectado mediante `management_target_identity`, `management_target_type` y `management_target_matches_current_identity`, junto con revocacion coordinada y bulk revoke remoto, reporting operativo via `auth:security-center:report`, export explicito de actores administrativos gobernados via `--management-actors`, snapshot durable opcional del reporte via `--export-log` y mutacion administrativa operacional via `auth:security-center:revoke-device` gobernada por actor explicito, decision compartida de autorizacion, decision por alcance y modo `direct_admin/delegated_admin` | Faltan metadata de dispositivo mas fuerte, authorization/policy multi-actor mas profunda y mutaciones distribuidas administrativas mas ricas del security center |
| 36 | Policy Engine Requirement Composition Security Posture And Authentication Governance | Pendiente | Sin evidencia suficiente | Falta `AuthenticationPolicyEngine` y composicion formal de requisitos |
| 37 | Assurance Level Authentication Context And Trust Classification | Parcial | `Support/AuthenticationAssurance.php`, `AuthenticationContext::authenticationStrength()`, `AuthenticationContext::authenticationAssuranceProfile()`, `Quantum/Middlewares/AuthMiddleware.php` interpreta `minimum_strength` y `mfa`, `Quantum/Middlewares/MfaMiddleware.php` ofrece entry point dedicado, se conserva `amr` MFA local y `step_up`, y `ControllerSecurityContextFactory.php` ya puede reutilizar `AuthenticationStrength`, el perfil de assurance, claims administrativas compartidas y governance de claims privilegiadas de una sesion Auth real dentro de Controllers Security | Falta persistencia de evidencias ricas y denials/evidencias mas ricas |
| 38 | Challenge Negotiation Continuation And Interactive Flow | Pendiente | Sin evidencia suficiente | Faltan transactions, continuation state y challenge negotiation |
| 39 | Transaction State Nonce Replay Protection CSRF Binding And Cryptographic Continuation Security | Parcial | `Devices/TrustedDeviceCredentialValidator.php` detecta replay del credential anterior inmediato mediante `previous_credential_hash`, `AuthManager.php` limpia el cookie y elimina el estado `trusted` del contexto recuperado, y `AuthManagerTest.php` valida la revocacion del trusted-device record al reutilizar el cookie previo | Faltan nonce store, replay protection mas general, binding CSRF/continuation y seguridad criptografica fuera del flujo trusted-device |
| 40 | Security Notification Alerting Compromise Detection Incident Response And Account Protection | Pendiente | Sin evidencia suficiente | Faltan alertas, compromise detection y respuestas de proteccion de cuenta |
| 41 | Identity Lifecycle Account State Suspension Lockout Deactivation Deletion And Reactivation | Pendiente | Sin evidencia suficiente | Faltan lifecycle states y reglas de suspension/reactivacion |
| 42 | Privacy Data Minimization Retention Consent And Security Metadata Governance | Parcial | `AuthManager` reduce `User-Agent` a `client_family`, `client_platform`, `device_kind` y `device_reference` pseudonimizado, reduce IP a `ip_prefix`, evita exponer secretos bearer en inventory, persiste solo `credential_hash` y `previous_credential_hash` del trusted-device cookie y actualiza `last_activity_at` server-side; `config/auth.php` define `auth.session.cleanup.tombstone_retention` y reglas de rotacion/replay para `trusted_devices`; `AuthManagerTest.php`, `AuthSessionsCleanupCommandTest.php` y repositorios validan minimizacion, emision de cookies, rotacion, retencion minima y purga operativa de trusted devices expirados | Faltan politicas de retencion mas ricas, cleanup gobernado y gobierno explicito de metadata sensible |
| 43 | Notification Security Communication Out Of Band Channel And Delivery Governance | Pendiente | Sin evidencia suficiente | Faltan canales out-of-band, delivery governance y contratos de notificacion |
| 44 | Authentication Background Processing Async Security Task Maintenance Cleanup And Scheduled Operation | Parcial | `AuthSessionsCleanupCommand.php` expone cleanup operativo bajo demanda para sesiones expiradas, trusted devices vencidos y tombstones; `AuthDevicesReconcileCommand.php` expone reconciliacion bajo demanda del posture trusted/untrusted de sesiones sobre stores compartidos; los repositorios soportan `purgeRecoveryReasons()` y enumeracion global `all()` | Faltan jobs/scheduler, cleanup asincrono y reconciliacion multi-store automatizada |
| 45 | Rate Capacity Resource Governance Abuse Prevention And Denial Of Service Resilience | Pendiente | Sin evidencia suficiente | Faltan budgets de recurso, control de capacidad y resistencia DoS especifica |
| 46 | Migration Legacy Credential Import Backward Compatibility And Progressive Security Upgrade | Pendiente | Sin evidencia suficiente | Faltan migracion de credenciales legadas y upgrade progresivo |
| 47 | Developer Experience Facade Helper Configuration Bootstrap And Application Integration | Parcial | helper `auth()`, facade `Quantum/Facades/Auth.php`, `config/auth.php`, `AuthenticationServiceProvider.php`, `Quantum/Middlewares/AuthMiddleware.php`, `Quantum/Middlewares/GuestMiddleware.php`, `Quantum/Middlewares/MfaMiddleware.php`, `Route::mfa()`, `Application.php` delega el wiring del subsistema al provider, `auth` respeta `minimum_strength`/`mfa`, la API publica ya expone inventory/revocacion de sesiones propias, trusted-device records, trusted-device credential cliente validada, `device_reference` derivado, challenge reduction con rotacion del cookie, hints de management en `trustedDevices()`, un security center agregado mediante `devices()` con `management_sensitivity/management_reason_code`, `management_authority/management_ownership_proof` y hints del actor actual proyectados desde el contexto, mutaciones coordinadas via `revokeDevice()` / `revokeOtherDevices()` con bypass gobernado de `fresh-auth` para revocacion remota por dispositivo agregado, y ahora un par coherente de discovery + mutacion multi-identidad via `managedDevices(identity, type?)` y `revokeManagedDevice(identity, device_reference, type?, scope)` con alcance parcial compatible con consola; ese par ya proyecta tambien metadata target explicita (`management_target_identity`, `management_target_type`, `management_target_matches_current_identity`), claims administrativas crudas del actor (`management_actor_authority`, `management_actor_ownership_proof`, `management_actor_claims_source`, `management_actor_privilege_level`, `management_actor_scopes`), relacion actor-target (`management_actor_target_relation`, `management_actor_target_reason_code`) y capacidad administrativa por alcance (`management_actor_can_manage_sessions`, `management_actor_session_authorization_mode`, `management_actor_session_authorization_reason_code`, `management_actor_can_manage_trusted_devices`, `management_actor_trusted_device_authorization_mode`, `management_actor_trusted_device_authorization_reason_code`) para que la API publica sea mas honesta sobre el ownership administrado, tooling operativo accesible en consola via `auth:security-center:report` y `auth:security-center:revoke-device`, y Controllers Security puede consumir una sesion `Quantum\Auth` real con claims administrativas compartidas, governance de claims privilegiadas y una decision administrativa compartida reutilizable sin requerir bearer token artificial | Falta una API publica todavia mas coherente, policy/authorization multi-actor mas profunda y helpers/aliases complementarios |
| 48 | Administration, Operational Tooling, Diagnostics, Security Operations And Production Management | Parcial | `Quantum/Console/Commands/AuthSessionsCleanupCommand.php`, `Quantum/Console/Commands/AuthDevicesReconcileCommand.php`, `Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`, `Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`, `ConsoleApplication.php`, `AuthSessionsCleanupCommandTest.php`, `AuthDevicesReconcileCommandTest.php`, `AuthSecurityCenterReportCommandTest.php` y `AuthSecurityCenterRevokeDeviceCommandTest.php` aportan cleanup, reconciliacion operativa, reporting seguro del security center, conteo de management gobernado, export explicito de actores administrativos gobernados con distincion `direct_admin/delegated_admin`, decision compartida de autorizacion con `management_authorized/management_authorization_reason_code`, snapshot JSONL durable opcional del reporte via `--export-log`, revocacion operacional coordinada por `identity + device_reference` gobernada por actor + `actor_session_public_id`, y ahora autorizacion operacional por alcance `all|sessions|trusted-devices` alineada con `AuthenticationContext`, con rechazo explicito cuando el actor delegado no cubre el `scope`; el runtime principal ya converge parcialmente con ese modelo permitiendo bypass gobernado de `fresh-auth` sobre revocacion remota por dispositivo agregado y semantica compartida entre runtime y consola | Faltan diagnosticos mas profundos, policy administrativa multi-actor mas profunda y tooling mas amplio del sistema Auth |
| 49 | Reference Implementation Default Components Secure Defaults And Framework Integration | Parcial | Existe base integrada con `AuthManager`, `AuthenticationServiceProvider`, `AuthenticationContextAccessor`, `AuthenticationOrchestrator`, `DefaultAuthenticatorResolver`, `LocalIdentityProvider`, `PasswordAuthenticator`, `SessionAuthenticator`, `PasswordPolicy`, `Support/AuthenticationAssurance.php`, `AuthExceptionMapper.php`, `Quantum/Middlewares/AuthMiddleware.php`, `Quantum/Middlewares/GuestMiddleware.php`, `Quantum/Middlewares/MfaMiddleware.php`, `config/auth.php`, repositorio de session en memoria/archivo, repositorio de trusted devices en memoria/archivo, enumeracion global `all()` para tooling, `session_public_id`, inventory seguro con metadata reducida de session/device, `device_reference` derivado, hints de accion/policy para sessions y trusted devices, trusted-device records con MFA, trusted-device credential cliente validada, challenge reduction con rotacion y replay revocation, freshness configurable para revocacion remota de sesiones y trusted devices en self-service, inventory agregado `devices()` sobre store compartido con `management_sensitivity/management_reason_code`, ownership `management_authority/management_ownership_proof` y hints/claims del actor actual proyectados desde la decision compartida de management, mutaciones coordinadas `revokeDevice()` / `revokeOtherDevices()` con bypass gobernado para revocacion remota por dispositivo agregado, un par coherente de inventory + mutacion multi-identidad via `managedDevices(identity, type?)` y `revokeManagedDevice(identity, device_reference, type?, scope)` con `all|sessions|trusted-devices`, ahora complementado con metadata target explicita (`management_target_identity`, `management_target_type`, `management_target_matches_current_identity`), claims administrativas crudas del actor (`management_actor_authority`, `management_actor_ownership_proof`, `management_actor_claims_source`, `management_actor_privilege_level`, `management_actor_scopes`), relacion actor-target (`management_actor_target_relation`, `management_actor_target_reason_code`) y autorizacion delegada por alcance (`management_actor_can_manage_sessions`, `management_actor_session_authorization_mode`, `management_actor_session_authorization_reason_code`, `management_actor_can_manage_trusted_devices`, `management_actor_trusted_device_authorization_mode`, `management_actor_trusted_device_authorization_reason_code`) sobre el ownership proyectado, reconciliacion operativa `auth:devices:reconcile`, reporting operativo `auth:security-center:report` con snapshot JSONL durable opcional via `--export-log` y decision compartida de autorizacion administrativa, mutacion administrativa operacional `auth:security-center:revoke-device` gobernada por actor explicito y sesion publica valida con audit trail JSONL durable opcional, distincion operativa entre `direct_admin` y `delegated_admin`, Controllers Security derivando principal/claims/strength, claims administrativas compartidas y governance de claims privilegiadas desde sesion Auth, retencion minima de tombstones y trusted devices expirados, multiples `Set-Cookie` y comando `auth:sessions:cleanup`, incluyendo errores `stale_session`, `revoked_session`, `fresh_authentication_required`, strength insufficiente, MFA local, `step_up` y alias `mfa` | Faltan stores mas robustos, policy/authorization multi-actor mas profunda y testing utilities mas amplias |

| 50 | System Integration And Final Architecture | Parcial | Hay integracion minima entre contracts, contextos, authenticators, sessions, `AuthenticationServiceProvider`, kernel HTTP, middlewares `auth/guest/mfa`, denials `guest_only/stale_session/step_up_required`, assurance por metadata de ruta, assurance profile explicito, MFA local, `step_up`, integracion de Controllers Security con sesiones Auth reales, claims administrativas compartidas, governance de claims privilegiadas y pruebas del subsistema Auth | Falta el cierre end-to-end del sistema `Quantum/Auth` como plataforma coherente de autenticacion |

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
- ya existe password policy, hardening basico del lifecycle de session y tombstones minimos de recovery,
- pero aun faltan stores distribuidos reales, lifecycle completo de credenciales y endurecimiento adicional.

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
