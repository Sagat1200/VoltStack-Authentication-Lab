# VoltStack Authentication System

## 23 — Authentication Events, Hooks, Listeners, Subscribers and Extension Lifecycle System

- **Archivo:** `23_AUTHENTICATION_EVENTS_HOOKS_LISTENERS_SUBSCRIBERS_AND_EXTENSION_LIFECYCLE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de eventos, hooks, listeners, subscribers y extensibilidad del lifecycle de Authentication.

**Depende especialmente de:**

- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`
- `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md`
- `18_ACCOUNT_RECOVERY_PASSWORD_RESET_IDENTITY_RECOVERY_AND_CREDENTIAL_REESTABLISHMENT_SYSTEM.md`
- `19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`
- `20_AUTHENTICATION_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL_SYSTEM.md`
- `21_AUTHENTICATION_DEVICE_TRUST_DEVICE_IDENTITY_AND_TRUSTED_DEVICE_CREDENTIAL_SYSTEM.md`
- `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`

---

## 1. Propósito

Este documento define cómo VoltStack expondrá el lifecycle del sistema Authentication a:

- application code
- plugins
- packages
- security modules
- audit systems
- notification systems
- telemetry
- analytics
- tenant extensions
- enterprise integrations

sin acoplar directamente los componentes Core.

- El sistema deberá proporcionar:
- Authentication Events
- Security Events
- Domain Events
- Lifecycle Hooks
- Event Listeners
- Event Subscribers
- Prioritized Handlers
- Synchronous Events
- Deferred Events
- Transactional Events
- Critical Security Hooks
- Extension Points
- Plugin Integration
- Event Ordering
- Event Versioning
- Event Payload Security
- Secret Redaction
- Listener Failure Policies
- Tenant-aware Events
- Request-scoped Hooks
- Persistent Runtime Safety

## 2. Problema arquitectónico

Sin un sistema de eventos bien definido, distintos componentes pueden terminar haciendo cosas como:

```php
$mailer->send(...);

$audit->log(...);

$analytics->track(...);

$plugin->run(...);

$securityMonitor->notify(...);
```

directamente dentro de:

- PasswordAuthenticator
- SessionManager
- RecoveryManager
- RiskEngine
- PasskeyAuthenticator
- LogoutManager

Esto produciría:

- tight coupling
- poor testability
- circular dependencies
- unpredictable ordering
- security side effects
- plugin fragility

## 3. Principio fundamental

Los componentes Core deberán emitir hechos del dominio; no deberán conocer todos los consumidores de esos hechos.

## 1. Events vs Hooks

VoltStack distinguirá:

```text
EVENT
    notifies that something happened

HOOK
    permits controlled participation
    in a defined lifecycle point
```

## 5. Ejemplo Event

AuthenticationSucceeded
significa:

```text
La Authentication produjo un resultado exitoso.

Listeners pueden reaccionar.
```

## 6. Ejemplo Hook

BeforeAuthenticationFinalization
permite que extensiones autorizadas:

- add constraints
- collect additional context

request a security action
antes de completar el proceso.

## 7. Regla importante

Un Event no deberá permitir reescribir arbitrariamente un hecho que ya ocurrió.

## 1. Ejemplo

Después de:
AuthenticationSucceeded
un listener ordinario no deberá cambiar el resultado a:
INVALID_PASSWORD

## 2. Security Hook

Si se necesita impedir finalización, deberá existir un Hook explícito anterior:
BeforeAuthenticationFinalization

## 3. Categorías

VoltStack tendrá al menos:

- Domain Events
- Security Events
- Lifecycle Events
- Operational Events
- Hooks

## 4. Domain Events

Representan hechos importantes del dominio.

- Ejemplos:
- IdentityAuthenticated
- AuthenticationFailed
- SessionEstablished
- DeviceTrusted
- RecoveryCompleted

## 5. Security Events

Representan hechos relevantes para seguridad.

- Ejemplos:
- CredentialCompromiseDetected
- AuthenticationRiskElevated
- CredentialStuffingSuspected
- DeviceCredentialReplayDetected
- RecoveryReplayDetected

## 6. Lifecycle Events

Representan transiciones del pipeline.

- Ejemplo:
- AuthenticationFlowStarted
- AuthenticationEvidenceVerified
- AuthenticationFinalizationStarted

## 7. Operational Events

Más orientados a infraestructura.

- Ejemplo:
- AuthenticationProviderUnavailable
- RateLimitStoreFailed
- RiskProviderTimedOut

## 8. Event Base Contract

Conceptualmente:

```php
interface AuthenticationEventInterface
{
    public function eventId(): EventId;

    public function occurredAt(): \DateTimeImmutable;

    public function context(): AuthenticationEventContext;
}
```

## 9. EventId

Debe ser:

- unique
- opaque
- traceable

## 10. AuthenticationEventContext

Podrá contener referencias seguras a:

- AuthenticationAttemptId
- AuthenticationFlowId
- IdentityReference
- TenantReference
- FirewallId
- SessionReference
- DeviceReference
- según el evento.

## 11. No raw secrets

Nunca deberá contener:

- password
- OTP
- Bearer Token
- Refresh Token
- Recovery Token
- client secret
- private key
- raw WebAuthn assertion

## 12. Event Payload Security

Todo evento deberá clasificarse.

## 13. EventSensitivity

Ejemplo:

- PUBLIC_INTERNAL
- SECURITY_INTERNAL
- SENSITIVE_SECURITY
- RESTRICTED

## 14. Default

Los eventos de Authentication serán:

- internal
- no serializables públicamente por defecto.

## 15. Secret Redaction

Antes de dispatch, payloads deberán pasar por:

- AuthenticationEventRedactor
- cuando sea necesario.

## 16. Redactor

Contrato:

```php
interface AuthenticationEventRedactorInterface
{
    public function redact(
        AuthenticationEventInterface $event
    ): AuthenticationEventInterface;
}
```

## 17. Better design

Preferible construir eventos seguros desde el inicio en vez de incluir secrets y después eliminarlos.

## 18. Event Immutability

Eventos deberán ser:

- immutable
- readonly where possible

## 19. Razón

Un listener no deberá modificar el evento para afectar listeners posteriores.

## 20. Ejemplo

final readonly class AuthenticationSucceeded
implements AuthenticationEventInterface
{
public function __construct(
public EventId $eventId,
public AuthenticationAttemptId $attempt,
public IdentityReference $identity,
public AuthenticationAssurance $assurance,
public \DateTimeImmutable $occurredAt,
) {}
}

## 21. Listener

Contrato conceptual:

```php
interface AuthenticationEventListenerInterface
{
    public function handle(
        AuthenticationEventInterface $event
    ): void;
}
```

## 22. Typed listeners

Preferible:

```php
final class RecordAuthenticationAudit
{
    public function __invoke(
        AuthenticationSucceeded $event
    ): void {
        // ...
    }
}
```

## 23. Listener Registry

VoltStack mantendrá:
AuthenticationEventListenerRegistry

## 24. Registro

Podrá configurarse mediante:

- Service Provider
- Attributes
- Configuration
- Plugin registration
- Subscriber

## 25. Subscriber

Agrupa múltiples listeners relacionados.

```php
Conceptualmente:
interface AuthenticationEventSubscriberInterface
{
    public static function subscribedEvents(): array;
}
```

## 26. Ejemplo

final class AuthenticationAuditSubscriber
{
public static function subscribedEvents(): array
{
return [
AuthenticationSucceeded::class => 'onSuccess',
AuthenticationFailed::class => 'onFailure',
LogoutCompleted::class => 'onLogout',
];
}
}

## 27. Subscribers no son global God Objects

Deberán agrupar concerns coherentes.
Ejemplo válido:
AuthenticationAuditSubscriber
Ejemplo malo:
EverythingAuthenticationSubscriber

## 28. Event Dispatcher

Contrato:

```php
interface AuthenticationEventDispatcherInterface
{
    public function dispatch(
        AuthenticationEventInterface $event
    ): void;
}
```

## 29. Generic Event Dispatcher

VoltStack probablemente dispondrá de:

- Quantum\Event
- Authentication podrá utilizarlo mediante adapter.

## 30. Recommended architecture

Quantum\Event
generic infrastructure

Quantum\Auth\Event
Authentication-specific events

## 38. Event Priority

Listeners podrán declarar:
priority

## 39. Priority semantics

Ejemplo:

- 1000 critical security
- 500 audit
- 100 notifications
- 0 default
- -100 analytics
- Solo ilustrativo.

## 40. Priority does not define security authority

Un analytics listener con priority alto no se convierte en security control.

## 41. Critical Handlers

Algunos listeners/hooks pueden clasificarse:

- CRITICAL
- IMPORTANT
- BEST_EFFORT

## 42. Failure Policy

Cada extension point deberá definir qué ocurre si falla un handler.

## 43. Ejemplo crítico

Si:
SecuritySessionRevocationListener
falla durante account compromise:

- fail closed
- puede ser correcto.

## 44. Ejemplo no crítico

Si:
AuthenticationAnalyticsListener
falla:

- Authentication should continue
- normalmente.

## 45. ListenerFailurePolicy

Valores:

- FAIL_CLOSED
- FAIL_OPERATION
- CONTINUE_AND_REPORT
- IGNORE
- DEFER_FAILURE

## 46. IGNORE

Solo apropiado para listeners realmente no importantes.

## 47. Event dispatch error isolation

Un listener defectuoso no debe romper todos los listeners automáticamente.

## 48. Dispatcher strategy

Podrá:

- execute listener
- catch failure

apply listener failure policy
continue or abort

## 49. Hook System

Hooks son distintos.

```php
Contrato conceptual:
interface AuthenticationHookInterface
{
    public function execute(
        AuthenticationHookContext $context
    ): AuthenticationHookResult;
}
```

## 50. AuthenticationHookResult

Podrá expresar:

- CONTINUE
- MODIFY_CONTEXT
- ADD_REQUIREMENT
- REQUEST_STEP_UP
- REJECT

pero solo en hooks cuyo contrato permita esas acciones.

## 51. Hook Capability

Cada Hook definirá qué puede hacer.

```text
Ejemplo:
BeforeRiskAssessment
    may contribute signals

BeforeFinalization
    may add security requirement

AfterAuthenticationSucceeded
    cannot invalidate past credential verification directly
```

## 52. Hook sandbox conceptual

Extensiones no deberán tener autoridad ilimitada.

## 53. Hook Permissions

Podrán clasificarse:

- READ_CONTEXT
- ADD_METADATA
- ADD_SECURITY_SIGNAL
- ADD_AUTHENTICATION_REQUIREMENT
- DENY_FINALIZATION
- MODIFY_RESPONSE

## 54. Plugin manifests

Podrán declarar qué hook capabilities necesitan.

## 55. Principle of least privilege

Plugins no deberían obtener automáticamente:

- DENY_FINALIZATION
- si solo necesitan analytics.

## 56. Authentication Lifecycle Hooks

VoltStack podrá definir puntos como:

```text
BeforeAuthenticationFlowStart
AfterAuthenticationFlowStart

BeforeAuthenticatorResolution
AfterAuthenticatorResolution

BeforeCredentialVerification
AfterCredentialVerification

BeforeIdentityEligibilityCheck
AfterIdentityEligibilityCheck

BeforeRiskAssessment
AfterRiskAssessment

BeforeAdaptiveAuthenticationDecision
AfterAdaptiveAuthenticationDecision

BeforeStepUp
AfterStepUp

BeforeAuthenticationFinalization
AfterAuthenticationFinalization

BeforeSessionEstablishment
AfterSessionEstablishment

BeforeLogout
AfterLogout
```

## 57. Not all hooks should exist in V1

Exponer demasiados hooks congela arquitectura demasiado pronto.

## 58. Recommended V1 hooks

Más conservador:

- AuthenticationFlowStarting
- AuthenticationContextEnrichment
- AuthenticationSecurityRequirements
- AuthenticationFinalizationGuard
- AuthenticationFinalized
- LogoutGuard
- LogoutCompleted

## 59. Events can be numerous

Los Events son menos peligrosos para compatibility que mutating hooks.

## 60. Event List Initial Set

VoltStack deberá definir al menos:

```text
AuthenticationFlowStarted
AuthenticationFlowExpired
AuthenticationFlowCancelled

AuthenticationAttemptStarted
AuthenticationMethodSelected
AuthenticationCredentialVerified
AuthenticationCredentialRejected

AuthenticationIdentityResolved
AuthenticationIdentityRejected

AuthenticationRiskAssessed
AuthenticationStepUpRequired
AuthenticationChallengeIssued
AuthenticationChallengeCompleted

AuthenticationSucceeded
AuthenticationFailed
AuthenticationFinalizationStarted
AuthenticationFinalized

AuthenticationSessionEstablished
AuthenticationSessionRevoked

RememberMeCredentialIssued
RememberMeCredentialRotated
RememberMeCredentialRevoked

DeviceTrusted
DeviceRevoked
DeviceCompromised

RecoveryStarted
RecoveryAuthorized
RecoveryCompleted
RecoveryRejected

LogoutStarted
LogoutCompleted
```

## 61. Security Events Initial Set

BruteForceSuspected
CredentialStuffingSuspected
PasswordSprayingSuspected
OtpGuessingSuspected
TokenGuessingSuspected
MfaFatigueSuspected

CredentialCompromiseDetected
SessionHijackSuspected
DeviceCredentialReplayDetected
PasskeyCounterAnomalyDetected
RecoveryTokenReplayDetected

ImpossibleTravelDetected
HighRiskAuthenticationDetected
AuthenticationSecurityHoldRequested

## 62. AuthenticationAttemptStarted

Debe ocurrir antes de credential verification.

- Payload seguro:
- AttemptId
- FlowId
- AuthenticatorId
- Tenant
- Firewall
- timestamp
- No credential secret.

## 63. AuthenticationCredentialVerified

Puede contener:

- credential type
- credential public reference
- authentication method
- evidence properties

## 64. No secret values

Nunca:

- actual token
- password
- OTP

## 65. AuthenticationCredentialRejected

Internamente podrá incluir:

- failure classification
- pero deberá respetar redaction.

## 66. AuthenticationIdentityResolved

Payload:

- IdentityReference
- IdentityType
- TenantReference

## 67. AuthenticationIdentityRejected

Razones:

- DISABLED
- SUSPENDED
- EXPIRED
- TENANT_DISABLED
- según disclosure interna.

## 68. AuthenticationRiskAssessed

Podrá contener:

- RiskLevel
- reason codes
- signal summary
- model version
- policy version

## 69. Avoid raw intelligence payloads

No incluir información completa de proveedores externos.

## 70. AuthenticationStepUpRequired

Payload:

- required assurance properties
- purpose
- freshness

## 71. AuthenticationChallengeIssued

Puede señalar:

- MFA
- PASSKEY
- FEDERATION
- RECOVERY

pero no challenge secret.

## 72. AuthenticationSucceeded

Debe representar:

- authentication requirements satisfied
- pero todavía puede ser anterior a Session establishment.

## 73. AuthenticationFinalized

Representa:
usable AuthenticationContext established

## 74. Diferencia crítica

AuthenticationSucceeded
proof/policy succeeded

AuthenticationFinalized
resulting authentication state committed

## 75. SessionEstablished

Puede incluir:

- SessionPublicReference
- IdentityReference
- assurance
- provenance

## 76. LogoutStarted

Payload:

- Identity
- scope
- reason
- current session reference

## 77. LogoutCompleted

Incluye:

- scope
- sessions revoked count summary
- device action summary
- sin secrets.

## 78. Synchronous Events

Algunos eventos pueden ejecutarse inmediatamente.
Ejemplo:

- AuthenticationFinalized
- para actualizar request-scoped state.

## 79. Deferred Events

Otros pueden enviarse después.

- Ejemplo:
- Send login notification
- Analytics event
- SIEM export

## 80. Deferred Dispatch

Podrá usar:

- queue
- event bus
- outbox
- background worker

## 81. Authentication should not wait unnecessarily

No bloquear login por:

- marketing analytics
- non-critical email
- external BI pipeline

## 82. Critical notification

Incluso una security notification puede ser deferred si su delivery no altera la validez inmediata del Authentication result.

## 83. Transactional Events

Problema:

```text
DB transaction
    ↓
event dispatched
    ↓
listener sees state
    ↓
DB rollback
```

## 84. Solution

Eventos dependientes de commit deberán emitirse:
AFTER_COMMIT

## 85. DispatchMode

Podrá ser:

- IMMEDIATE
- AFTER_COMMIT
- DEFERRED

## 86. Event metadata

Cada event class podrá declarar default dispatch mode.

## 87. Example

PasswordCredentialReplaced
puede requerir:

- AFTER_COMMIT
- para notificaciones/audit downstream.

## 88. Outbox pattern

Para eventos críticos distribuidos, VoltStack podrá soportar:
Transactional Outbox

## 89. Why

Evita:

- DB committed
- event lost

## 90. Example

Account compromise recovery:

```text
credentials revoked
SecurityVersion++
    ↓
commit
    ↓
event outbox
    ↓
session revocation workers
security notifications
```

## 91. Not all events need outbox

Sería demasiado caro.

## 92. Event Delivery Guarantees

Podrán clasificarse:

- BEST_EFFORT
- AT_LEAST_ONCE
- COMMIT_COUPLED

## 93. Exactly once

No deberá prometerse en distributed systems.

## 94. Listener idempotency

Deferred consumers deberán ser idempotentes cuando delivery sea at-least-once.

## 95. EventId supports idempotency

Consumers podrán guardar:

- processed EventId
- si es necesario.

## 96. Event Ordering

Dentro del mismo Flow se deberá definir orden lógico.

## 97. Example

AuthenticationFlowStarted
AuthenticationAttemptStarted
AuthenticationCredentialVerified
AuthenticationIdentityResolved
AuthenticationRiskAssessed
AuthenticationSucceeded
AuthenticationFinalizationStarted
AuthenticationSessionEstablished
AuthenticationFinalized

## 98. Ordering across distributed consumers

No deberá asumirse globalmente salvo que infraestructura lo garantice.

## 99. Correlation sequence

Event metadata podrá incluir:

- flowSequence
- opcionalmente.

## 100. Sequence scope

Solo dentro de:

- FlowId
- Session lifecycle
- RecoveryTransaction

## 101. Event Versioning

Importantísimo para plugins y queues.

## 102. Event Schema Version

Cada serializable event deberá tener:

- eventType
- schemaVersion

## 103. Example

auth.authentication.finalized
version = 1

## 104. Breaking changes

Nueva versión de event schema.

## 105. Internal PHP event classes

Pueden evolucionar más libremente si nunca salen del proceso.

## 106. Externalizable Events

Los que crucen:

- queue
- event bus
- plugin boundary
- webhook future

deberán tener schemas estables.

## 107. AuthenticationEventEnvelope

Conceptualmente:

```php
final readonly class AuthenticationEventEnvelope
{
    public function __construct(
        public EventId $id,
        public string $type,
        public int $version,
        public \DateTimeImmutable $occurredAt,
        public array $payload,
        public AuthenticationCorrelation $correlation,
    ) {}
}
```

## 108. Correlation

Puede incluir:

- TraceId
- AttemptId
- FlowId
- TenantReference
- según privacy.

## 109. Tenant-aware Events

Todo evento multi-tenant deberá incluir contexto de tenant adecuado.

## 110. No tenant inference downstream

Un listener no debería tener que adivinar tenant desde Identity.

## 111. TenantEventContext

Podrá contener:

- TenantReference
- SecurityRealm

## 112. Cross-tenant listener safety

Listener deberá ejecutarse con tenant context explícito cuando lo necesite.

## 113. Never leak current tenant singleton

Especialmente bajo FrankenPHP.

## 114. Request-scoped context

Tenant y Identity actuales nunca deberán derivarse de mutable static globals.

## 115. Listener Context

El dispatcher podrá proporcionar:
AuthenticationListenerExecutionContext

## 116. Listener context can contain services

Preferible usar dependency injection en listener en vez de service locator.

## 117. Event payload remains data

No deberá convertirse en container accessor.

## 118. Hooks and Dependency Injection

Hooks serán services registrados.

## 119. HookRegistry

interface AuthenticationHookRegistryInterface
{
public function handlersFor(
AuthenticationHookPoint $point
): iterable;
}

## 120. HookPoint

Value Object/enum:

- FLOW_START
- CONTEXT_ENRICHMENT
- SECURITY_REQUIREMENTS
- FINALIZATION_GUARD
- LOGOUT_GUARD

## 121. Hook ordering

Priority + deterministic registration order.

## 122. Deterministic ordering

Dos deployments con misma configuración deberán producir mismo orden.

## 123. Dynamic runtime registration

Debe limitarse.

## 124. Production preference

Compile Event/Hook registry durante bootstrap.

## 125. Benefits

performance
determinism
configuration validation
long-running runtime safety

## 126. AuthenticationExtension

Podrá existir una abstracción superior:

```php
interface AuthenticationExtensionInterface
{
    public function register(
        AuthenticationExtensionRegistry $registry
    ): void;
}
```

## 127. Extension Registry

Permitirá registrar:

- Authenticators
- Signal Providers
- Risk Rules
- Event Listeners
- Hooks
- Recovery Methods
- Device Providers
- Federation Adapters

## 128. Pero este documento se centra en lifecycle extensions

No sustituirá registries especializados.

## 129. Extension Manifest

Conceptualmente:

- name
- version

required Auth API version
listeners
subscribers
hooks
required capabilities

## 130. Compatibility

Plugin podrá declarar:
auth_extension_api >= 1 < 2

## 131. Extension API Version

Debe existir separada de Framework version.

## 132. Razón

VoltStack puede evolucionar sin romper todos los plugins de Authentication.

## 133. Hook Capability Model

Ejemplo:

```text
audit plugin:
    EVENT_READ
```

enterprise risk plugin:
ADD_SECURITY_SIGNAL

policy plugin:
ADD_REQUIREMENT

security enforcement plugin:
FINALIZATION_VETO

## 134. FINALIZATION_VETO

Alta autoridad.
No deberá otorgarse automáticamente a cualquier plugin.

## 135. Plugin Security

Un plugin Auth corre dentro del proceso de aplicación y técnicamente puede hacer mucho.
Aun así, API design debe minimizar poder accidental.

## 136. Extension Isolation

Conceptual, no sandbox fuerte en PHP.

## 137. Plugins externos

Deben considerarse parte del trusted computing base si ejecutan código in-process.

## 138. Documentation requirement

VoltStack deberá dejar claro:
Instalar un plugin Authentication equivale a confiar código con acceso potencial a security-sensitive application context.

## 1. Safe Event Subscribers

Analytics plugins deberían recibir payload limitado.

## 2. Event Projection

Puede existir:

- AuthenticationEventProjection
- para producir una versión reducida del event.

## 3. Example

Internal event:

- AuthenticationFailed
- IdentityReference
- NetworkContext
- RiskAssessment

Analytics projection:

```php
authenticator=password
result=failure
risk=high
sin Identity.
```

## 4. Projection Profiles

AUDIT
ANALYTICS
TELEMETRY
PLUGIN
EXTERNAL_BUS

## 5. Benefits

Reduce accidental data exposure.

## 6. Listener scopes

Listeners podrán declararse:

- SYNC
- DEFERRED
- AFTER_COMMIT

## 7. Sync listener budget

Debe ser pequeño.

## 8. Slow listener detection

Observability puede medir:
listener latency

## 9. Listener timeout

Para external/network listeners, deberían preferirse deferred jobs.

## 10. No HTTP calls in critical synchronous path by default

Ejemplo malo:

```text
AuthenticationSucceeded
    ↓
```

call external analytics API
↓
wait 4 seconds

## 11. Asynchronous subscriber

Mejor:

```text
AuthenticationSucceeded
    ↓
enqueue event
```

## 12. Deferred listener and secrets

Queue payload sigue sin poder contener secrets.

## 13. Serialization safety

Event serialization debe ser explícita.

## 14. No PHP object serialization for external event bus

Preferir:
versioned structured payload

## 15. Generic JSON compatibility

Cuando se necesite externalización.

## 16. Events and Audit

Audit subsystem será un consumer importante.

## 17. Audit does not equal event log

No todos los Events deberán persistirse como Audit Records.

## 18. Example

AuthenticationFlowStarted
puede ser telemetry.

## 19. Mientras

PasswordResetCompleted
DeviceCompromised
GlobalLogoutCompleted
probablemente audit.

## 20. AuditProjectionPolicy

Decidirá qué eventos producen audit records.

## 21. Events and Notifications

Notifications pueden consumir:

- NewDeviceTrusted
- PasswordResetCompleted
- AccountRecoveryCompleted
- HighRiskAuthenticationDetected

## 22. Notification listeners

Deben ser deferred por default.

## 23. Events and Session Revocation

Algunos security events pueden requerir acciones críticas.

## 24. Example

CredentialCompromiseDetected
puede activar:
SessionRevocationCommand

## 25. Recommended architecture

Listener produce un command explícito:

- SecurityActionDispatcher
- en lugar de modificar stores arbitrariamente.

## 26. Event → Command distinction

Event:
something happened

Command:
perform an action

## 165. No command disguised as event

Evitar:
PleaseRevokeSessionsEvent
Mejor:

```text
CredentialCompromiseDetected
    ↓
SecurityPolicy
    ↓
RevokeSessionsCommand
```

## 166. Hooks and Commands

Un Hook puede retornar:
SecurityActionRecommendation
pero no ejecutar arbitrariamente múltiples subsistemas si no es su responsabilidad.

## 167. Authentication Context Enrichment Hook

Puede añadir metadata segura.

- Ejemplo:
- organization context
- enterprise device context
- application-specific login context

## 168. Cannot add fake verified evidence

Nunca deberá permitir:

```php
context.addVerifiedPassword()
desde generic hook.
```

## 169. Evidence creation privilege

Solo:

- Authenticator
- trusted Evidence Provider

explicit authentication extension capability

## 170. Security Requirement Hook

Puede añadir:

- RequireMfa
- RequirePasskey
- RequireFreshAuthentication

## 171. Monotonic security

Hooks genéricos podrán:
strengthen requirements
pero no:
remove framework security floor

## 172. Requirement composition

Debe ser monotónica para security floors.

## 173. Example

Framework:
AAL2 required
plugin intenta:
AAL1
Effective:
AAL2

## 174. Finalization Guard Hook

Puede rechazar finalization.

## 175. Use cases

enterprise licensing disabled
compliance hold
external fraud hard deny
maintenance security restriction

## 176. Boundary

No deberá reimplementar Authorization de recursos.

## 177. Logout Guard

Puede impedir scopes peligrosos?
Normalmente logout debería ser fácil.

## 178. Recommended

Hooks no deberían impedir:

- CURRENT_SESSION logout
- salvo necesidades extraordinarias.

## 179. Why

Usuario siempre debería poder cerrar su sesión local.

## 180. Global logout guard

Puede exigir:
fresh authentication
antes de:
ALL_SESSIONS / GLOBAL

## 181. LogoutCompleted event

Nunca deberá depender de external provider availability para existir.

## 182. Hook Re-entrancy

Listeners/hooks pueden generar otros Events.
Debe evitarse recursión infinita.

## 183. Event recursion protection

Dispatcher podrá mantener:

- dispatch depth
- event lineage
- request-scoped.

## 184. Maximum depth

Puede existir para prevenir loops accidentales.

## 185. Example loop

AuthenticationFailed
↓
Listener emits AuthenticationFailed
↓
...
debe detectarse.

## 186. Event lineage

Puede registrar:

- causationEventId
- correlationId

## 187. Correlation ID

Útil para:

- authentication flow
- recovery
- security incident

## 188. Causation

Ejemplo:

```text
CredentialCompromiseDetected
    ↓
causes SessionRevoked
```

## 189. Distributed tracing

Event envelope puede transportar:

- trace context
- según observability policy.

## 190. Do not trust incoming trace headers blindly for security decisions

Tracing metadata no es Authentication evidence.

## 191. Error Handling

Listener failures deberán convertirse en:

- AuthenticationExtensionFailure
- o equivalente.

## 192. Error categories

LISTENER_FAILURE
HOOK_FAILURE
SUBSCRIBER_FAILURE
SERIALIZATION_FAILURE
DEFERRED_DISPATCH_FAILURE
OUTBOX_FAILURE

## 193. Security critical failures

Podrán abortar operación.

## 194. Example

FinalizationGuard fails
Si policy:

- FAIL_CLOSED
- Authentication no se finaliza.

## 195. Analytics failure

No bloquea.

## 196. Hook Result Aggregation

Múltiples hooks pueden devolver resultados.

## 197. Aggregation rule

Para security decisions:

- DENY > ADD_REQUIREMENT > CONTINUE
- conceptualmente.

## 198. Cannot let later hook undo deny

A menos que architecture defina explicit override authority, lo cual no se recomienda.

## 199. Monotonic Hook Aggregation

Preferido.

## 200. Example

Hook A:
Require MFA
Hook B:
Continue
resultado:
Require MFA

## 201. Hook Exception

No debe convertirse accidentalmente en CONTINUE.
Policy explícita.

## 202. HookTimeoutPolicy

Para hooks síncronos externalizados:

- FAIL_CLOSED
- FAIL_OPERATION
- CONTINUE_WITH_SIGNAL

## 203. External network hooks

No recomendados en hot path.
Mejor integrar como provider especializado con timeout/circuit breaker.

## 204. Bootstrap Compilation

Registries podrán compilarse:

```text
event → ordered listeners
hook → ordered handlers
```

## 205. Compile-time validation

Detectar:

- duplicate registration
- unknown event
- invalid method
- missing service
- incompatible API version
- illegal capability

## 206. Production optimization

Permite:
precomputed dispatch map

## 207. Cache

El dispatch map puede formar parte de framework cache.

## 208. Dynamic plugin changes

Requieren invalidar/recompilar registry.

## 209. Hot reload

Development mode puede permitirlo.

## 210. Persistent Runtime

Event Dispatcher y Hook Registry podrán ser singleton si son:

- immutable
- stateless
- compiled

## 211. Prohibido

final class AuthenticationDispatcher
{
private ?Identity $currentUser;
private array $currentEvents;
}
como shared singleton.

## 212. Per-request dispatch state

Como:

- recursion depth
- correlation
- temporary listeners

deberá vivir en:

- RequestScope
- FiberLocal
- ExecutionContext

## 213. FrankenPHP Safety

Obligatorio limpiar:

- temporary listener state
- hook aggregation state
- current correlation state
- event dispatch stack

al final del request.

## 214. Fiber Safety

Dos flows simultáneos no deberán compartir:

- current event
- hook result
- current Identity

## 215. Temporary listeners

Si se soportan, deberán ser request-scoped.

## 216. Global listener registration

Solo durante bootstrap.

## 217. Testing — Event Ordering

Validar orden esperado.

## 218. Testing — Listener Priority

higher priority
before lower priority
de forma determinista.

## 219. Testing — Listener Failure

Casos:

- critical listener fails
- best-effort listener fails

multiple listeners one fails

## 220. Testing — Hook Aggregation

CONTINUE + REQUIRE_MFA
REQUIRE_MFA + DENY
multiple requirements

## 221. Testing — Secret Redaction

Events nunca deben contener:

- password
- OTP
- raw bearer
- recovery token

## 222. Testing — Tenant Isolation

Tenant A event listeners nunca deben recibir accidentalmente request context de Tenant B.

## 223. Testing — Deferred Events

Verificar:

- serialized payload
- schema version
- idempotency

## 224. Testing — After Commit

transaction commit
→ dispatch

transaction rollback
→ no after-commit event

## 225. Testing — Outbox

DB commit
event pending
worker retry
duplicate delivery
Consumer idempotente.

## 226. Testing — Event Recursion

Detect loops.

## 227. Testing — Plugin Compatibility

supported extension API
unsupported extension API
missing capability

## 228. Testing — Finalization Guard

Debe poder impedir finalization sin crear session parcial.

## 229. Testing — Monotonic Security

Extension no puede reducir framework security floor.

## 230. Testing — Async Failure

Deferred analytics failure no rompe Authentication.

## 231. Testing — FrankenPHP

Request A emite eventos para Alice.
Request B para Bob.
Nunca:
Bob event sees Alice context

## 232. Testing — Fiber Concurrency

Listeners concurrentes deben recibir event correcto.

## 233. Testing — Performance

Medir:

- dispatch overhead
- listener lookup
- hook aggregation
- allocation count

## 234. Event fast path

Con cero listeners:
minimal overhead

## 235. Compiled registry lookup

Idealmente O(1) o equivalente.

## 236. Testing — Listener Explosion

Miles de plugin listeners no deben degradar silenciosamente sin observability.

## 237. Listener limits

No necesariamente hard limit, pero tooling deberá permitir inspección.

## 238. Event Registry Inspection

Development tooling podrá mostrar:

- Event
- Listeners
- Priorities
- Dispatch Modes
- Failure Policies

## 239. Hook Registry Inspection

Igualmente:

- Hook Point
- Handlers
- Capabilities
- Priority

## 240. Debugging

Puede existir:

- auth:event:list
- auth:hook:list

en CLI tooling futuro.

## 241. No secrets in debug tools

Obligatorio.

## 242. Observability

Spans:

- auth.event.dispatch
- auth.event.listener
- auth.hook.execute
- auth.hook.aggregate
- auth.event.defer
- auth.event.outbox

## 243. Metrics

auth_event_dispatched_total
auth_event_listener_total
auth_event_listener_failure_total
auth_hook_execution_total
auth_hook_rejection_total
auth_event_deferred_total
auth_event_outbox_failure_total

## 244. Labels

Safe:

- event_type
- hook_type
- listener_class category
- result
- dispatch_mode

## 245. Avoid

IdentityId
FlowId
email
token
como metric labels.

## 246. Listener latency

Histogram:
auth_event_listener_duration

## 247. Slow listener warning

Development/observability puede detectar listeners que bloquean login.

## 248. Event Sampling

High-volume lifecycle events podrán samplearse para telemetry.

## 249. Security events

No deberían perderse por sampling cuando son audit-critical.

## 250. Audit vs Telemetry routing

Debe estar explícito.

## 251. Events from Authentication Subsystems

Password
PasswordAuthenticated
PasswordAuthenticationFailed
PasswordRehashed
PasswordCredentialChanged
PasswordCredentialCompromised

## 252. Session

AuthenticationSessionCreated
AuthenticationSessionRestored
AuthenticationSessionRotated
AuthenticationSessionRevoked
AuthenticationSessionExpired

## 253. Remember-Me

RememberMeIssued
RememberMeRotated
RememberMeRestored
RememberMeRevoked
RememberMeReplayDetected

## 254. MFA

MfaChallengeStarted
MfaFactorVerified
MfaFactorRejected
MfaStepUpCompleted
MfaRecoveryStarted

## 255. Passkeys

PasskeyRegistrationStarted
PasskeyRegistered
PasskeyAuthenticationSucceeded
PasskeyAuthenticationRejected
PasskeyRevoked
PasskeyCounterAnomalyDetected

## 256. Federation

FederatedLoginStarted
FederatedAuthenticationSucceeded
FederatedAuthenticationRejected
FederatedIdentityLinked
FederatedIdentityUnlinked
FederationProviderUnavailable

## 257. Recovery

RecoveryRequested
RecoveryEvidenceVerified
RecoveryAuthorized
RecoveryCompleted
RecoveryRejected
RecoveryReplayDetected

## 258. Abuse

AuthenticationThrottled
BruteForceSuspected
CredentialStuffingSuspected
PasswordSprayingSuspected
MfaFloodingSuspected

## 259. Risk

AuthenticationRiskAssessed
HighRiskAuthenticationDetected
ImpossibleTravelDetected
StepUpRequiredByRisk
SecurityHoldRecommended

## 260. Device

DeviceObserved
DeviceEnrolled
DeviceTrusted
DeviceTrustExpired
DeviceRevoked
DeviceMarkedLost
DeviceMarkedCompromised

## 261. Flow

AuthenticationFlowStarted
AuthenticationFlowContinued
AuthenticationFlowCompleted
AuthenticationFlowExpired
AuthenticationFlowCancelled

## 262. Logout

LogoutStarted
CurrentSessionLoggedOut
DeviceSessionsLoggedOut
AllSessionsLoggedOut
GlobalLogoutCompleted

## 263. Event Naming Convention

Preferir:

- Past Tense
- para Events.

Ejemplo:

- AuthenticationSucceeded
- DeviceTrusted
- SessionRevoked

## 264. Hooks

Preferir:

- Before...
- After...
- ...Guard
- ...Enricher
- según semántica.

## 265. Avoid vague names

No:

- AuthenticationEvent
- AuthChanged
- SecurityHook

## 266. Public vs Internal Events

Algunos eventos podrán ser:
PUBLIC_EXTENSION_API
otros:
INTERNAL_IMPLEMENTATION

## 267. Why

No queremos congelar cada detalle interno como API pública.

## 268. Public Event Registry

VoltStack deberá documentar qué Events son estables para plugins.

## 269. Internal Events

Pueden cambiar entre minor versions según compatibility policy.

## 270. Stable V1 public events

Podrían ser:

- AuthenticationSucceeded
- AuthenticationFailed
- AuthenticationFinalized
- LogoutCompleted
- SessionEstablished
- SessionRevoked
- DeviceTrusted
- DeviceRevoked
- RecoveryCompleted
- HighRiskAuthenticationDetected

## 271. Internal lifecycle events

Ejemplos:

- AuthenticatorResolutionStarted
- RiskProviderBatchCompleted
- pueden permanecer internos.

## 272. Extension lifecycle

Plugins themselves pueden tener:

- REGISTERED
- BOOTED
- ACTIVE
- DISABLED
- FAILED

## 273. AuthenticationExtensionManager

Podrá gestionar:

- registration
- compatibility
- capabilities
- boot
- shutdown

## 274. Extension registration phase

Durante container/bootstrap.

## 275. Extension boot phase

Después de que registries y dependencies principales estén disponibles.

## 276. No request-time boot

Producción no debería bootstrappear plugins en cada login.

## 277. Extension shutdown

Long-running runtime puede necesitar cleanup, pero shared extension objects deberán evitar mutable request state.

## 278. Extension Configuration

Cada extension podrá tener namespace propio.
Ejemplo:
'authentication.extensions.enterprise_risk'

## 279. Tenant plugin configuration

Si se soporta, debe resolverse por request sin mutar global extension configuration.

## 280. No singleton config mutation

Nunca:

```php
$plugin->tenant = $currentTenant;
en singleton.
```

## 281. Extension Security Floor

Plugins no pueden desactivar:

- credential validation
- security state checks
- risk hard rules
- framework mandatory invariants
- mediante listener común.

## 282. Trusted Core Extensions

VoltStack podrá designar algunas extensions como:

- CORE_SECURITY_EXTENSION
- con más capacidades.

## 283. Third-party extensions

Por default:

- limited lifecycle capabilities
- cuando la API lo permita.

## 284. Event Replay

Deferred systems pueden re-procesar events.
Listeners deberán manejar idempotency.

## 285. Replay does not mean Authentication replay

Importante.
Reprocesar:

- AuthenticationFinalized event
- no debe recrear una nueva Session.

## 286. Side-effect listener safety

Un listener de Session creation no debería depender de event replay.
Session creation pertenece al synchronous core pipeline.

## 287. Events observe committed state

Arquitectura clave.

## 288. Commands create state

Events describe state changes.

## 289. Security action consumers

Cuando un Event genera una acción:

```text
HighRiskAuthenticationDetected
    ↓
policy
    ↓
CreateSecurityHoldCommand
```

la command debe ser idempotente.

## 290. Event bridges

VoltStack podrá ofrecer bridges a:

- PSR-compatible dispatchers
- Symfony EventDispatcher-style adapters
- Laravel event-style adapters
- message buses
- sin acoplar core.

## 291. Native event model remains authoritative

Adapters traducen.

## 292. Symfony influence

Symfony destaca por:

- EventDispatcher
- listeners
- subscribers
- priorities
- security events

VoltStack podrá aprovechar esa claridad.

## 293. Laravel influence

Laravel ofrece ergonomía mediante:

- events
- listeners
- queued listeners
- subscribers
- service providers

VoltStack deberá mantener experiencia sencilla.

## 294. VoltStack addition

Añadirá separación explícita entre:

- Event
- Hook
- Security Hook
- Deferred Event
- Transactional Event
- Extension Capability
- Event Projection

## 295. Developer API

Ejemplo conceptual:

```php
Auth::events()->listen(
    AuthenticationFinalized::class,
    RecordLoginAnalytics::class,
);
```

## 296. Subscriber API

Auth::events()->subscribe(
AuthenticationAuditSubscriber::class
);

## 297. Hook API

Auth::hooks()->add(
AuthenticationHookPoint::FINALIZATION_GUARD,
CorporateSecurityGuard::class,
);

## 298. Attribute-based registration

Opcional:

```php
# [ListenTo(AuthenticationFinalized::class)]
final class RecordLogin
{
}
```

## 299. Hook attribute

Podría existir:

```php
# [AuthenticationHook(
    point: AuthenticationHookPoint::SECURITY_REQUIREMENTS,
    priority: 100
)]
```

## 300. Compile attributes

Durante build/cache.

## 301. No runtime reflection hot path

En producción preferir metadata compilada.

## 302. Event Configuration Example

return [

'authentication' => [

'events' => [

'listeners' => [

AuthenticationFinalized::class => [
RecordAuthenticationAudit::class,
SendLoginNotification::class,
],

AuthenticationFailed::class => [
RecordFailedAuthentication::class,
],

],

],

],

];

## 303. Listener metadata example

AuthenticationFinalized::class => [

[
'listener' => RecordAuthenticationAudit::class,
'priority' => 500,
'mode' => 'sync',
'failure' => 'fail_operation',
],

[
'listener' => SendLoginNotification::class,
'priority' => 0,
'mode' => 'deferred',
'failure' => 'continue_and_report',
],

],

## 304. Hook config example

'hooks' => [

'finalization_guard' => [
CorporateSecurityGuard::class,
ComplianceGuard::class,
],

],

## 305. Complete flow with Events

Authentication Request
↓
AuthenticationFlowStarted
↓
AuthenticationAttemptStarted
↓
Authenticator
↓
AuthenticationCredentialVerified
↓
AuthenticationIdentityResolved
↓
AuthenticationRiskAssessed
↓
AuthenticationSecurityRequirements Hook
↓
Step-Up if required
↓
AuthenticationSucceeded
↓
FinalizationGuard Hook
↓
AuthenticationFinalizationStarted
↓
Session Established
↓
AuthenticationSessionEstablished
↓
AuthenticationFinalized
↓
Deferred Events
├── Audit
├── Notification
├── Analytics
└── Security Telemetry

## 306. Complete failure flow

AuthenticationAttemptStarted
↓
Credential Rejected
↓
AuthenticationCredentialRejected
↓
AbuseProtection records failure
↓
AuthenticationFailed
↓
Audit / Risk / Telemetry listeners
↓
Public Failure Mapping

## 307. Security Event flow

Credential Stuffing Detector
↓
CredentialStuffingSuspected
↓
Security Policy Listener
↓
TemporarySecurityRule Command
↓
Attack Mode Elevated
↓
AttackModeChanged

## 308. Recovery event flow

RecoveryCompleted
↓
Security Policy
↓
SessionRevocationCommand
↓
Sessions Revoked
↓
AuthenticationSessionRevoked
↓
Security Notification Listener

## 309. Device event flow

DeviceCredentialReplayDetected
↓
Risk Engine signal
↓
HighRiskAuthenticationDetected
↓
DeviceRevocationCommand
↓
DeviceRevoked

## 310. Architecture overview

AUTHENTICATION CORE
│
┌─────────────────┼─────────────────┐
▼                 ▼                 ▼
EVENTS             HOOKS          COMMANDS
│                 │                 │
▼                 ▼                 ▼
Notification     Security Guard    Security Action
Audit            Context Enricher  Session Revocation
Analytics        Requirement       Device Revocation
Telemetry        Provider          Recovery Action
│                 │                 │
└───────────┬─────┴──────┬──────────┘
▼            ▼
EXTENSION REGISTRY
│
▼
PLUGINS / PACKAGES

## 311. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Events describe facts; Hooks participate in controlled lifecycle points

## 2. Events are immutable

## 3. Raw Authentication secrets never enter events

## 4. Security-sensitive hooks have explicit capabilities

## 5. Security requirements compose monotonically

## 6. Generic listeners cannot weaken framework security floors

## 7. Deferred and transactional event delivery are first-class

## 8. Event schemas are versioned when crossing process boundaries

## 9. At-least-once consumers must be idempotent

## 10. Tenant context is explicit

## 11. Listener failure behavior is policy-driven

## 12. Event and Hook registries are compiled where possible

## 13. Authentication extensions are versioned through an extension API

## 14. Persistent workers never retain request-local event context

## 15. Events are not substitutes for Commands

## 16. Security invariants — Events

AUTH-EVENT-01
Authentication Events never contain raw Authentication credentials.
AUTH-EVENT-02
Events are immutable after creation.

- AUTH-EVENT-03
- Events represent facts and cannot be mutated to rewrite history.
- AUTH-EVENT-04

Externalizable events have explicit schema versions.

- AUTH-EVENT-05
- Sensitive payloads are minimized and classified.
- AUTH-EVENT-06

Tenant context is explicit when applicable.

## 17. Security invariants — Hooks

AUTH-HOOK-01
Hooks have explicit lifecycle points.

- AUTH-HOOK-02
- Hook capabilities are bounded.
- AUTH-HOOK-03

Generic hooks cannot fabricate verified Authentication Evidence.
AUTH-HOOK-04
Hooks cannot weaken framework security floors.

- AUTH-HOOK-05
- Security requirement aggregation is monotonic by default.
- AUTH-HOOK-06

Hook failures follow explicit failure policy.

## 18. Security invariants — Listeners

AUTH-LISTENER-01
Best-effort listener failure does not silently corrupt Authentication state.
AUTH-LISTENER-02
Critical listeners are explicitly classified.

- AUTH-LISTENER-03
- Deferred listeners must tolerate duplicate delivery where applicable.
- AUTH-LISTENER-04

Listeners must not depend on process-global current Identity state.
AUTH-LISTENER-05
Listener execution order is deterministic for equal configuration.

## 19. Security invariants — Transactions

AUTH-EVENT-TX-01
After-commit events are not emitted after transaction rollback.
AUTH-EVENT-TX-02
Critical distributed events may use an Outbox.

- AUTH-EVENT-TX-03
- Exactly-once delivery is not assumed.
- AUTH-EVENT-TX-04

State-changing operations remain Commands/Core operations rather than replayable events.

## 20. Security invariants — Extensions

AUTH-EXT-01
Authentication extensions declare compatibility.

- AUTH-EXT-02
- Extension capabilities follow least privilege.
- AUTH-EXT-03

Third-party extensions cannot silently bypass mandatory credential verification.
AUTH-EXT-04
In-process Authentication plugins are considered trusted application code.
AUTH-EXT-05
Public extension events are distinguished from internal implementation events.

## 21. Security invariants — Runtime

AUTH-EVENT-RT-01
Event dispatch request state is execution-scoped.

- AUTH-EVENT-RT-02
- Shared registries and dispatchers are immutable/stateless where possible.
- AUTH-EVENT-RT-03

No previous request Identity/Event state survives FrankenPHP boundaries.
AUTH-EVENT-RT-04
Concurrent fibers have independent dispatch stacks.
AUTH-EVENT-RT-05
Temporary listeners are never global mutable state.

## 22. Anti-pattern — Mailer inside Authenticator

Evitar:

```php
if ($passwordIsValid) {
    $mailer->send(...);
}
```

El Authenticator produce evidencia/evento.

## 23. Anti-pattern — Mutable Event

No:
$event->authenticated = false;

## 24. Anti-pattern — Every listener may veto login

No.

## 25. Anti-pattern — Analytics listener fail-closes Authentication

No por default.

## 26. Anti-pattern — Security hook fail-opens silently

Nunca.

## 27. Anti-pattern — Raw password in AuthenticationFailed

Nunca.

## 28. Anti-pattern — Token in queue event

Nunca.

## 29. Anti-pattern — Current Tenant from singleton

Crítico con FrankenPHP.

## 30. Anti-pattern — Event performs command semantics

Evitar:

- RevokeSessionsEvent
- cuando realmente es una orden.

## 31. Anti-pattern — Plugin can lower Assurance requirement

No.

## 32. Anti-pattern — Events are persisted forever automatically

No.

## 33. Anti-pattern — One global subscriber with every concern

Evitar.

## 34. Anti-pattern — Reflection scanning every Authentication request

No en producción.

## 35. Anti-pattern — Event bus outage breaks all logins by default

Solo si esos eventos son explícitamente critical.

## 36. Componentes principales

AuthenticationEvent
AuthenticationEventContext
AuthenticationEventEnvelope
AuthenticationEventDispatcher
AuthenticationEventRedactor

AuthenticationEventListener
AuthenticationEventSubscriber
AuthenticationEventListenerRegistry

AuthenticationHook
AuthenticationHookPoint
AuthenticationHookResult
AuthenticationHookRegistry
AuthenticationHookAggregator

AuthenticationExtension
AuthenticationExtensionRegistry
AuthenticationExtensionManager
AuthenticationExtensionManifest
AuthenticationExtensionApiVersion

## 333. Delivery components

AuthenticationEventDeliveryMode
AuthenticationDeferredEventDispatcher
AuthenticationAfterCommitDispatcher
AuthenticationEventOutbox
AuthenticationEventSerializer
AuthenticationEventProjection

## 334. Failure components

AuthenticationListenerFailurePolicy
AuthenticationHookFailurePolicy
AuthenticationExtensionFailure
AuthenticationEventDispatchException
AuthenticationHookExecutionException

## 335. Namespace sugerido

VoltStack\Quantum\Auth\Event
VoltStack\Quantum\Auth\Event\Contracts
VoltStack\Quantum\Auth\Event\Lifecycle
VoltStack\Quantum\Auth\Event\Security
VoltStack\Quantum\Auth\Event\Listener
VoltStack\Quantum\Auth\Event\Subscriber
VoltStack\Quantum\Auth\Event\Dispatch
VoltStack\Quantum\Auth\Event\Projection

VoltStack\Quantum\Auth\Hook
VoltStack\Quantum\Auth\Hook\Contracts
VoltStack\Quantum\Auth\Hook\Security

VoltStack\Quantum\Auth\Extension
VoltStack\Quantum\Auth\Extension\Contracts

## 336. Estructura sugerida

src/Quantum/Auth/
├── Event/
│   ├── Contracts/
│   │   ├── AuthenticationEventInterface.php
│   │   ├── AuthenticationEventDispatcherInterface.php
│   │   ├── AuthenticationEventListenerInterface.php
│   │   └── AuthenticationEventSubscriberInterface.php
│   │
│   ├── Lifecycle/
│   │   ├── AuthenticationFlowStarted.php
│   │   ├── AuthenticationAttemptStarted.php
│   │   ├── AuthenticationCredentialVerified.php
│   │   ├── AuthenticationSucceeded.php
│   │   ├── AuthenticationFailed.php
│   │   ├── AuthenticationFinalized.php
│   │   └── LogoutCompleted.php
│   │
│   ├── Security/
│   │   ├── CredentialStuffingSuspected.php
│   │   ├── HighRiskAuthenticationDetected.php
│   │   ├── DeviceCredentialReplayDetected.php
│   │   └── RecoveryReplayDetected.php
│   │
│   ├── Listener/
│   │   ├── AuthenticationEventListenerRegistry.php
│   │   └── AuthenticationListenerFailurePolicy.php
│   │
│   ├── Subscriber/
│   │   └── AuthenticationSubscriberRegistry.php
│   │
│   ├── Dispatch/
│   │   ├── AuthenticationEventDispatcher.php
│   │   ├── DeferredAuthenticationEventDispatcher.php
│   │   ├── AfterCommitAuthenticationEventDispatcher.php
│   │   └── AuthenticationEventOutbox.php
│   │
│   └── Projection/
│       ├── AuthenticationEventProjection.php
│       ├── AuditEventProjection.php
│       └── TelemetryEventProjection.php
│
├── Hook/
│   ├── Contracts/
│   │   └── AuthenticationHookInterface.php
│   ├── AuthenticationHookPoint.php
│   ├── AuthenticationHookResult.php
│   ├── AuthenticationHookRegistry.php
│   ├── AuthenticationHookAggregator.php
│   └── Security/
│       └── AuthenticationFinalizationGuard.php
│
└── Extension/
├── Contracts/
│   └── AuthenticationExtensionInterface.php
├── AuthenticationExtensionRegistry.php
├── AuthenticationExtensionManager.php
├── AuthenticationExtensionManifest.php
└── AuthenticationExtensionApiVersion.php

## 337. Configuración conceptual

return [

'authentication' => [

'events' => [

'enabled' => true,

'listeners' => [

AuthenticationFinalized::class => [
RecordAuthenticationAudit::class,
SendAuthenticationNotification::class,
],

AuthenticationFailed::class => [
RecordFailedAuthentication::class,
],

],

],

],

];

## 338. Subscriber configuration

'subscribers' => [
AuthenticationAuditSubscriber::class,
AuthenticationTelemetrySubscriber::class,
];

## 339. Hook configuration

'hooks' => [

'security_requirements' => [
EnterpriseAuthenticationRequirements::class,
],

'finalization_guard' => [
ComplianceAuthenticationGuard::class,
],

];

## 340. Extension configuration

'extensions' => [

CorporateRiskExtension::class => [
'enabled' => true,
],

];

## 341. Arquitectura global

AUTHENTICATION CORE
│
▼
DOMAIN / SECURITY FACT
│
┌─────────┴─────────┐
▼                   ▼
EVENTS               HOOKS
│                   │
┌──────────┼──────────┐        │
▼          ▼          ▼        ▼
AUDIT   NOTIFICATION  TELEMETRY SECURITY
│          │          │       EXTENSIONS
└──────────┴──────────┼────────┘
▼
EXTENSION SYSTEM
│
▼
APPLICATION / PLUGINS

## 342. Comparación conceptual con Laravel y Symfony

Laravel aporta una experiencia muy práctica mediante:

- Events
- Listeners
- Subscribers
- Queued Listeners
- Service Providers

Symfony aporta una separación muy sólida mediante:

- EventDispatcher
- Event Subscribers
- Listener Priorities
- Security Events

VoltStack deberá tomar ambas fortalezas.

- Pero añadirá primitives específicas para Authentication:
- Security-sensitive Events
- Controlled Authentication Hooks
- Hook Capabilities
- Monotonic Security Requirements
- Listener Failure Policies
- Transactional Authentication Events
- Event Projections
- Extension API Versioning
- FrankenPHP-safe Execution Context

El resultado será:

- Laravel-like ergonomics
- +;
- Symfony-like event decomposition
- +;
- security-aware extension lifecycle

## 343. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte Authentication Events;
2. soporte Security Events;
3. soporte Lifecycle Events;
4. soporte immutable events;
5. soporte EventId;
6. soporte explicit event context;
7. evite raw secrets;
8. soporte sensitivity classification;
9. soporte listeners;
10. soporte subscribers;
11. soporte listener priorities;
12. soporte deterministic ordering;
13. soporte listener failure policies;
14. soporte synchronous events;
15. soporte deferred events;
16. soporte after-commit events;
17. soporte transactional outbox cuando sea requerido;
18. soporte event schema versioning;
19. soporte idempotent deferred consumers;
20. soporte tenant-aware events;
21. soporte event projections;
22. soporte audit projections;
23. soporte telemetry projections;
24. soporte Authentication Hooks;
25. soporte explicit hook points;
26. soporte hook capabilities;
27. soporte monotonic requirement composition;
28. soporte finalization guards;
29. soporte context enrichment;
30. soporte hook failure policies;
31. soporte extension registry;
32. soporte extension API versioning;
33. soporte plugin compatibility checks;
34. soporte compile-time registry validation;
35. soporte event correlation;
36. soporte causation metadata;
37. soporte recursion protection;
38. soporte observability;
39. soporte listener latency metrics;
40. soporte event registry inspection;
41. soporte multi-tenancy;
42. soporte event delivery across queues;
43. soporte extensibility without Core coupling;
44. sea fiber-safe;
45. sea seguro con FrankenPHP;
46. mantenga Events separados de Commands;
47. impida que generic listeners fabriquen Authentication Evidence;
48. impida que extensions debiliten security floors.
49. Regla arquitectónica final

VoltStack deberá preservar:

```text
AUTHENTICATION CORE ACTION
          │
          ▼
```

DOMAIN / SECURITY FACT
│
▼
IMMUTABLE AUTH EVENT
│
┌────┼─────────────┐
▼    ▼             ▼
AUDIT NOTIFICATION TELEMETRY
│
▼
DEFERRED DELIVERY
y para intervención controlada:

```text
AUTHENTICATION LIFECYCLE
          │
          ▼
      HOOK POINT
          │
          ▼
AUTHORIZED EXTENSIONS
          │
    ┌─────┼──────────┐
    ▼     ▼          ▼
 CONTINUE REQUIRE   DENY
          MORE
        SECURITY
          │
          ▼
EFFECTIVE AUTHENTICATION POLICY
```

La primera regla central será:
Events describen lo que ocurrió; Hooks son puntos explícitos donde extensiones autorizadas pueden influir en lo que todavía está por ocurrir.

La segunda:
Ningún listener genérico podrá fabricar evidencia de Authentication ni reducir requisitos mínimos de seguridad.

La tercera:
VoltStack tratará los eventos de Authentication como datos potencialmente sensibles: los secrets nunca entrarán en payloads, queues, logs o sistemas externos por defecto.

La cuarta:
La extensibilidad del sistema no deberá comprometer determinismo, aislamiento multi-tenant, concurrencia ni seguridad bajo FrankenPHP.

Siguiente documento recomendado
La secuencia natural continúa con:
`24_AUTHENTICATION_AUDIT_OBSERVABILITY_LOGGING_METRICS_TRACING_AND_EXPLAINABILITY_SYSTEM.md`
Este documento deberá formalizar transversalmente todo lo que hasta ahora hemos mencionado de forma distribuida:

- Authentication Audit
- Security Audit Trail
- Authentication Logging
- Structured Logs
- Security Event Logging
- Secret Redaction
- PII Protection
- Authentication Metrics
- Counters
- Histograms
- Tracing
- Trace Correlation
- Flow Correlation
- Attempt Correlation
- Session Correlation
- Risk Explainability
- Authentication Decision Explainability
- Failure Reason Classification
- Operational Diagnostics
- Security Diagnostics
- SIEM Integration
- Audit Retention
- Tamper Resistance
- Audit Integrity
- Tenant Audit Isolation
- Compliance Exports
- Telemetry Sampling
- High-cardinality Protection
- Alerting
- Dashboards
- FrankenPHP Observability Context

Con ello podremos separar definitivamente:

```php
EVENT SYSTEM
    comunica hechos

AUDIT SYSTEM
    conserva evidencia histórica de seguridad

LOGGING
    ayuda al diagnóstico

METRICS
    describen comportamiento agregado

TRACING
    explica un request/flow distribuido

EXPLAINABILITY
    explica por qué Authentication tomó una decisión
y evitar que todos esos conceptos terminen tratados simplemente como Log::info(...).
```
