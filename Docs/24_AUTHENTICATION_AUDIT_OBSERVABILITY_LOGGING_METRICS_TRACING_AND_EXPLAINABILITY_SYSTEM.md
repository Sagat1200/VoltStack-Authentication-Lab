# VoltStack Authentication System

## 24 — Authentication Audit, Observability, Logging, Metrics, Tracing and Explainability System

- **Archivo:** `24_AUTHENTICATION_AUDIT_OBSERVABILITY_LOGGING_METRICS_TRACING_AND_EXPLAINABILITY_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema transversal de auditoría, observabilidad, logging, métricas, tracing, correlación y explicabilidad de Authentication.

**Depende especialmente de:**

- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`
- `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md`
- `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md`
- `18_ACCOUNT_RECOVERY_PASSWORD_RESET_IDENTITY_RECOVERY_AND_CREDENTIAL_REESTABLISHMENT_SYSTEM.md`
- `19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`
- `20_AUTHENTICATION_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL_SYSTEM.md`
- `21_AUTHENTICATION_DEVICE_TRUST_DEVICE_IDENTITY_AND_TRUSTED_DEVICE_CREDENTIAL_SYSTEM.md`
- `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
- `23_AUTHENTICATION_EVENTS_HOOKS_LISTENERS_SUBSCRIBERS_AND_EXTENSION_LIFECYCLE_SYSTEM.md`

---

## 1. Propósito

Este documento define cómo VoltStack hará observable, auditable y explicable el comportamiento completo del sistema Authentication.
El subsistema deberá cubrir:

- Authentication Audit
- Security Audit Trail
- Structured Logging
- Operational Logging
- Security Logging
- Metrics
- Counters
- Gauges
- Histograms
- Tracing
- Distributed Tracing
- Correlation
- Authentication Flow Correlation
- Attempt Correlation
- Session Correlation
- Recovery Correlation
- Risk Explainability
- Authentication Decision Explainability
- Failure Explainability
- Security Decision Reasoning
- Secret Redaction
- PII Protection
- Audit Integrity
- Audit Retention
- Tenant Isolation
- Compliance Export
- SIEM Integration
- Alerting
- Dashboards
- Sampling
- High-cardinality Protection
- FrankenPHP Context Isolation

## 2. Problema arquitectónico

Sin una arquitectura formal de observabilidad, el sistema puede terminar con:

```php
Log::info('login failed');
Log::info('login success');
Log::error($exception);
```

Eso es insuficiente para responder preguntas como:

- ¿Por qué fue rechazado este login?
- ¿Qué factor elevó el riesgo?
- ¿Qué Authenticator decidió?
- ¿Qué sesión se creó?
- ¿Qué dispositivo estaba involucrado?
- ¿Hubo Step-Up?
- ¿Se disparó Recovery?
- ¿Quién revocó las sesiones?
- ¿Fue ataque o error técnico?
- ¿Qué policy estaba activa?
- ¿Qué tenant estaba involucrado?
- ¿Qué versión del Risk Model tomó la decisión?

## 3. Principio fundamental

VoltStack distinguirá explícitamente:

```text
EVENT
    comunica un hecho

AUDIT
    conserva evidencia histórica relevante

LOG
    ayuda a diagnosticar ejecución

METRIC
    describe comportamiento agregado

TRACE
    reconstruye una operación distribuida

EXPLANATION
    describe por qué se tomó una decisión
```

## 4. Regla central

No todo Event es Audit, no todo Log es Security Evidence y no toda decisión debe depender de lo que se registre posteriormente.

## 1. Segunda regla

Observability nunca deberá convertirse en un canal accidental de exfiltración de credenciales o secretos.

## 2. Arquitectura general

Authentication Subsystems
│
▼
Authentication Events
│
▼
Observability Router
│
┌──────┼────────┬─────────┬─────────────┐
▼      ▼        ▼         ▼             ▼
Audit   Logs   Metrics    Traces     Explainability
│       │       │          │             │
▼       ▼       ▼          ▼             ▼
Store   Sink   Backend    Collector    Decision Record
│       │       │          │             │
└───────┴───────┴──────────┴─────────────┘
│
▼
Security Operations / SIEM

## 3. AuthenticationObservabilityManager

Podrá existir como facade/orquestador:

```php
interface AuthenticationObservabilityManagerInterface
{
    public function record(
        AuthenticationObservabilityEvent $event
    ): void;
}
```

## 4. No God Object

Internamente delegará a:

- AuthenticationAuditRecorder
- AuthenticationLogger
- AuthenticationMetricRecorder
- AuthenticationTracer
- AuthenticationExplanationRecorder

## 5. AuthenticationObservabilityContext

Contexto común:

```php
final readonly class AuthenticationObservabilityContext
{
    public function __construct(
        public ?AuthenticationAttemptId $attempt,
        public ?AuthenticationFlowId $flow,
        public ?IdentityReference $identity,
        public ?TenantReference $tenant,
        public ?SessionReference $session,
        public ?DeviceReference $device,
        public ?TraceContext $trace,
        public \DateTimeImmutable $occurredAt,
    ) {}
}
```

## 6. Contexto seguro

No deberá contener:

- password
- OTP
- Bearer Token
- Refresh Token
- Recovery Token
- Passkey private material
- client secret
- session cookie value
- Remember-Me secret

## 7. Correlation identifiers

VoltStack deberá disponer de identificadores distintos para distintos niveles.
RequestId
TraceId
AuthenticationFlowId
AuthenticationAttemptId
SessionPublicReference
RecoveryTransactionId
FederationTransactionId
DevicePublicReference

## 8. Flow vs Attempt

Ejemplo:

```text
Flow F-100
    Attempt A-1 Password
    Attempt A-2 TOTP
    Attempt A-3 Passkey
```

## 9. Correlation no será Identity

Los identificadores de correlación no deberán revelar:

- email
- username
- database primary key
- token value

## 10. Authentication Audit

Audit será el registro duradero de hechos de seguridad relevantes.

## 11. Audit != Debug Log

Un Audit Record deberá responder:

- qué ocurrió
- cuándo
- sobre qué Identity
- en qué Tenant
- quién/qué lo provocó

qué acción se tomó
con qué resultado

## 12. AuthenticationAuditRecord

Conceptualmente:

```php
final readonly class AuthenticationAuditRecord
{
    public function __construct(
        public AuditRecordId $id,
        public AuthenticationAuditEventType $type,
        public AuditActor $actor,
        public ?IdentityReference $subject,
        public ?TenantReference $tenant,
        public AuthenticationAuditOutcome $outcome,
        public AuditReasonSet $reasons,
        public \DateTimeImmutable $occurredAt,
        public AuditCorrelation $correlation,
        public array $metadata = [],
    ) {}
}
```

## 13. Audit Actor vs Subject

Importante.
Ejemplo:

```text
Actor:
    Administrator Alice
```

Subject:
Identity Bob

Action:

```text
    Administrative Recovery
No deben confundirse.
```

## 18. AuditActor

Tipos:

- SELF
- ADMINISTRATOR
- SYSTEM
- SECURITY_AUTOMATION
- SERVICE
- FEDERATED_PROVIDER
- UNKNOWN_EXTERNAL

## 19. Audit Outcome

SUCCESS
FAILURE
DENIED
CANCELLED
EXPIRED
PARTIAL

## 20. Eventos auditables

Al menos:

```text
AuthenticationSucceeded
AuthenticationFailed
AuthenticationStepUpRequired
AuthenticationFinalized

PasswordChanged
PasswordResetCompleted

MfaFactorEnrolled
MfaFactorRevoked
MfaRecoveryCompleted

PasskeyRegistered
PasskeyRevoked

FederatedIdentityLinked
FederatedIdentityUnlinked

RecoveryStarted
RecoveryCompleted

DeviceTrusted
DeviceRevoked
DeviceMarkedLost
DeviceMarkedCompromised

SessionRevoked
GlobalLogoutCompleted

SecurityHoldCreated
SecurityHoldReleased

CredentialCompromiseDetected
HighRiskAuthenticationDetected
```

## 21. No auditar todo

Ejemplo:

- AuthenticationFlow internal state changed from X to Y
- puede ser trace/log, no necesariamente audit.

## 22. AuditProjectionPolicy

Contrato:

```php
interface AuthenticationAuditProjectionPolicyInterface
{
    public function shouldAudit(
        AuthenticationEventInterface $event
    ): bool;
}
```

## 23. Event → Audit projection

Authentication Event
↓
AuditProjectionPolicy
↓
AuthenticationAuditRecord

## 24. Audit append-only

Idealmente el audit principal deberá comportarse como:
append-only

## 25. No update history

No debería hacerse:

- UPDATE audit SET outcome='success'
- para cambiar eventos previos.

Mejor:

- Audit Record 1: Recovery Started
- Audit Record 2: Recovery Completed

## 26. Correction model

Si es necesario corregir:

- AuditCorrectionRecord
- referenciando el original.

## 27. Audit Integrity

En escenarios avanzados VoltStack podrá soportar:

- record hashing
- hash chains
- signed audit batches
- immutable storage
- WORM storage
- external SIEM replication

## 28. Hash Chain conceptual

Record 1 Hash
↓
Record 2 includes previous hash
↓
Record 3 includes previous hash

## 29. No guarantee universal

Esto será opcional según deployment.

## 30. AuditIntegrityProvider

Contrato:

```php
interface AuditIntegrityProviderInterface
{
    public function protect(
        AuthenticationAuditRecord $record
    ): ProtectedAuditRecord;
}
```

## 31. Audit storage implementations

DatabaseAuthenticationAuditStore
AppendOnlyAuthenticationAuditStore
FileAuthenticationAuditStore
ExternalSiemAuditStore
CompositeAuditStore

## 32. AuthenticationAuditStore

interface AuthenticationAuditStoreInterface
{
public function append(
AuthenticationAuditRecord $record
): void;
}

## 33. Append semantics

No:

```php
saveOrUpdate()
para audit records normales.
```

## 34. Audit Store Failure Policy

Podrá ser:

- FAIL_CLOSED
- BUFFER
- CONTINUE_AND_ALERT
- BEST_EFFORT

## 35. Critical operations

Ejemplos donde FAIL_CLOSED puede ser razonable:

- Administrative Recovery
- Credential export
- Security hold override
- High-security privileged authentication

## 36. Ordinary consumer login

Puede usar:

- BUFFER / CONTINUE_AND_ALERT
- según policy.

## 37. Audit buffering

Debe tener límites.
No crear:
unbounded memory queue

## 38. Outbox integration

Audit records críticos pueden utilizar:
transactional outbox

## 39. Authentication Logging

Logging se utilizará principalmente para diagnóstico.

## 40. Log categories

AUTHENTICATION
AUTHENTICATOR
SESSION
MFA
PASSKEY
FEDERATION
RECOVERY
RISK
ABUSE
DEVICE
EVENT
EXTENSION

## 41. Structured Logs

VoltStack deberá favorecer:

```php
{
  "event": "authentication.failed",
  "authenticator": "password",
  "reason": "invalid_credentials",
  "flow_id": "F...",
  "attempt_id": "A...",
  "risk": "moderate"
}
```

## 42. No free-form only logging

Evitar depender exclusivamente de:
"User could not log in because something went wrong"

## 43. Log Level Policy

Ejemplo:

```text
DEBUG
    internal lifecycle

INFO
    normal security lifecycle summaries

NOTICE
    unusual but expected security behavior

WARNING
    suspicious events

ERROR
    operational failures

CRITICAL
    severe security/infrastructure failure
```

## 44. Invalid password no es ERROR

Un password incorrecto normalmente será:

- INFO / NOTICE / security telemetry
- no error técnico.

## 45. Database unavailable

Sí puede ser:
ERROR

## 46. Credential stuffing suspected

Puede ser:

- WARNING
- o security event.

## 47. Secret Redaction

Logging deberá aplicar redaction estricta.

## 48. AuthenticationRedactionPolicy

interface AuthenticationRedactionPolicyInterface
{
public function redact(
array $context
): array;
}

## 49. Campos siempre prohibidos

password
password_confirmation
authorization header
access_token
refresh_token
id_token
recovery_token
mfa_code
otp
webauthn raw response where sensitive
client_secret
session cookie
remember_me token
trusted_device secret

## 50. Pattern Redaction

Además de keys conocidas, podrá existir protección contra:

- Bearer ...
- JWT-looking values

known secret object types
SensitiveParameter

## 51. Sensitive Value Objects

Valores sensibles deberán implementar una abstracción como:

- SensitiveValue
- que nunca se serializa accidentalmente a string.

## 52. Ejemplo

final class SensitiveString
{
public function __toString(): string
{
return '[REDACTED]';
}
}
Conceptualmente.

## 53. Stack traces

Deberán evitar parameters sensibles mediante mecanismos como:

```php
# [\SensitiveParameter]
cuando corresponda.
```

## 54. PII Protection

Authentication utiliza datos potencialmente personales.

- Ejemplos:
- email
- username
- IP
- device metadata
- location
- external provider identifiers

## 55. PII Classification

VoltStack podrá clasificar metadata:

- NON_IDENTIFYING
- PSEUDONYMOUS
- PERSONAL
- SENSITIVE_SECURITY
- SECRET

## 56. Logging policy by classification

Ejemplo:

```text
SECRET
    never log

SENSITIVE_SECURITY
    restricted sink

PERSONAL
    minimized

PSEUDONYMOUS
    preferred for long-term telemetry
```

## 57. Email logging

Preferible no registrar email completo en logs generales.

## 58. Identity Reference

Utilizar:

- opaque IdentityPublicReference
- cuando sea suficiente.

## 59. IP logging

Puede ser necesario para seguridad.

- Pero deberá tener:
- retention policy
- access controls
- purpose

## 60. IP pseudonymization

Para analytics de largo plazo podrá utilizarse:

- keyed hash
- network aggregation
- coarse prefix

cuando no se necesite raw IP.

## 61. Authentication Metrics

Metrics describen comportamiento agregado.

## 62. Core metrics

auth_attempts_total
auth_success_total
auth_failure_total
auth_challenge_total
auth_stepup_total
auth_session_established_total
auth_logout_total

## 63. Authenticator metrics

auth_password_attempt_total
auth_passkey_attempt_total
auth_federated_attempt_total
auth_bearer_attempt_total

## 64. MFA metrics

auth_mfa_challenge_total
auth_mfa_success_total
auth_mfa_failure_total
auth_mfa_recovery_total

## 65. Risk metrics

auth_risk_assessment_total
auth_risk_stepup_total
auth_risk_denial_total

## 66. Abuse metrics

auth_throttle_total
auth_bruteforce_suspected_total
auth_credential_stuffing_suspected_total
auth_password_spraying_suspected_total

## 67. Recovery metrics

auth_recovery_started_total
auth_recovery_completed_total
auth_recovery_failed_total

## 68. Session metrics

auth_session_created_total
auth_session_restored_total
auth_session_revoked_total
auth_session_expired_total

## 69. Device metrics

auth_device_trusted_total
auth_device_revoked_total
auth_device_replay_total

## 70. Histograms

Ejemplos:

- auth_authentication_duration_seconds
- auth_password_verification_duration_seconds
- auth_risk_evaluation_duration_seconds
- auth_session_creation_duration_seconds
- auth_federation_exchange_duration_seconds
- auth_event_listener_duration_seconds

## 71. Metrics cardinality

Crítico.

- Nunca labels con:
- IdentityId
- email
- username
- IP
- SessionId
- FlowId
- AttemptId
- DeviceId
- token

## 72. Safe labels

Ejemplos:

- authenticator
- firewall
- result
- risk_level
- factor_type
- tenant_tier
- transport
- failure_category

## 73. Tenant labels

Usar Tenant ID directamente puede generar alta cardinalidad.

- Preferible:
- tenant class
- plan
- region
- deployment partition

si se necesita agregación.

## 74. Per-tenant observability

Para tenants individuales, usar:

- logs
- audit queries
- dedicated dashboards

en lugar de metric labels sin límite.

## 75. Metric backend independence

Core no deberá acoplarse a:

- Prometheus
- Datadog
- New Relic
- OpenTelemetry backend

## 76. AuthenticationMetricRecorder

interface AuthenticationMetricRecorderInterface
{
public function increment(
AuthenticationMetric $metric,
array $labels = []
): void;

public function observe(
AuthenticationHistogram $metric,
float $value,
array $labels = []
): void;
}

## 77. OpenTelemetry

Puede existir un adapter first-class.

## 78. Tracing

Tracing deberá reconstruir el recorrido de una Authentication.

## 79. Trace conceptual

HTTP Request
↓
auth.flow.start
↓
auth.abuse.evaluate
↓
auth.authenticator.password
↓
auth.identity.resolve
↓
auth.risk.assess
↓
auth.mfa.challenge
↓
auth.session.establish
↓
auth.flow.finalize

## 80. Trace Span Names

Convención:

- auth.flow.start
- auth.flow.continue
- auth.authenticator.verify
- auth.identity.resolve
- auth.eligibility.evaluate
- auth.abuse.evaluate
- auth.risk.assess
- auth.mfa.verify
- auth.passkey.verify
- auth.federation.exchange
- auth.recovery.verify
- auth.device.trust
- auth.session.establish
- auth.session.restore
- auth.logout

## 81. Span Attributes

Safe:

- auth.method=password
- auth.result=success
- auth.risk=moderate
- auth.stepup=true
- auth.transport=html

## 82. Unsafe attributes

Nunca:

- auth.password
- auth.token
- auth.email
- auth.authorization_header
- auth.otp

## 83. Trace Context

El trace deberá poder propagarse entre:

- request
- queue
- outbox
- security event processing

## 84. Trace != Security Trust

Un Trace ID recibido del cliente no deberá ser una Authentication credential.

## 85. Trace sampling

No todos los successful logins necesitan full trace en producción.

## 86. Sampling strategy

Podrá favorecer:

- 100% failures
- 100% high-risk
- 100% critical security events
- sample normal successes

## 87. Security critical traces

Pueden tener sampling elevado, pero cuidando PII.

## 88. Authentication Trace Correlation

Trace podrá correlacionar:

- FlowId
- AttemptId
- RecoveryTransaction
- SessionPublicReference
- internamente.

## 89. Explainability

VoltStack deberá poder explicar por qué tomó una decisión.

## 90. Explainability != Chain of Thought

El sistema expondrá:

- deterministic reason codes
- policy results
- signal summaries
- assurance requirements

no razonamiento interno libre.

## 91. AuthenticationDecisionExplanation

Conceptualmente:

```php
final readonly class AuthenticationDecisionExplanation
{
    public function __construct(
        public AuthenticationDecision $decision,
        public AuthenticationReasonCodeSet $reasons,
        public AuthenticationRequirementSet $requirements,
        public ?RiskExplanation $risk,
        public ?AssuranceExplanation $assurance,
        public PolicyReferenceSet $policies,
    ) {}
}
```

## 92. Reason codes

Ejemplos:

- INVALID_CREDENTIAL
- IDENTITY_DISABLED
- MFA_REQUIRED
- PASSKEY_REQUIRED
- FRESH_AUTH_REQUIRED
- HIGH_RISK
- NEW_DEVICE
- RECENT_RECOVERY
- INSUFFICIENT_ASSURANCE
- DEVICE_NOT_TRUSTED
- RATE_LIMITED
- FEDERATED_ASSURANCE_INSUFFICIENT
- SESSION_EXPIRED

## 93. Explainability layers

VoltStack deberá distinguir:

- INTERNAL
- DEVELOPER
- SECURITY_OPERATOR
- USER_SAFE

## 94. Internal explanation

Puede contener:

- policy IDs
- signal codes
- Authenticator result
- risk reason codes
- security state

## 95. Developer explanation

Útil en local/test:

- Password valid, but policy admin.high_security requires
- phishing-resistant AAL2 and current context is AAL1.

## 96. Security operator explanation

Puede agregar:

- risk source
- provider
- network classification
- recent security history

## 97. User-safe explanation

Ejemplo:
Additional verification is required to continue.

## 98. No leakage

Nunca decir al atacante:

- Your password was correct but TOTP is missing.
- si eso ayuda account enumeration o credential validation.

## 99. ExplanationDisclosurePolicy

interface AuthenticationExplanationDisclosurePolicyInterface
{
public function disclose(
AuthenticationDecisionExplanation $explanation,
AuthenticationDisclosureContext $context
): PublicAuthenticationExplanation;
}

## 100. Context-dependent disclosure

Un usuario ya autenticado haciendo Step-Up puede recibir más detalle que un anonymous login.

## 101. Risk Explainability

RiskExplanation podrá contener:

- risk level
- score if used
- reason codes
- signal categories
- policy version
- model version

## 102. Example

Risk: HIGH

Reasons:

```text
- NEW_DEVICE
- NEW_COUNTRY
- RECENT_ACCOUNT_RECOVERY
```

Policy:
admin-risk-v4

Decision:
REQUIRE_PHISHING_RESISTANT_STEP_UP

## 103. No raw third-party provider data

No incluir toda la respuesta de threat intelligence.

## 104. Assurance Explainability

Podrá mostrar internamente:

```text
Current:
    password
    AAL1
```

Required:

```text
    phishing-resistant possession
    fresh <= 5m
```

Missing:
phishing-resistant evidence

## 105. AuthenticationDecisionRecord

Para decisiones importantes podrá persistirse un resumen explicable.

## 106. Diferencia frente a Audit

Audit registra:
Authentication denied
Decision Record puede registrar:
por qué y con qué policies

## 107. Decision Record retention

Probablemente menor que audit principal.

## 108. Policy references

Toda decision importante debería poder registrar:

- policy ID
- policy version
- model version
- configuration profile

## 109. Reproducibility

Esto permite investigar:
¿Por qué el usuario necesitó Passkey el martes pero no el lunes?

## 110. Authentication Decision Provenance

Puede incluir:

- Authenticator
- Evidence types
- Assurance
- Risk level
- Device trust
- Effective policy
- sin secrets.

## 111. Failure Classification

Debe diferenciar:

- CREDENTIAL_FAILURE
- IDENTITY_FAILURE
- SECURITY_POLICY_FAILURE
- RISK_REJECTION
- ABUSE_REJECTION
- FLOW_FAILURE
- PROVIDER_FAILURE
- TECHNICAL_FAILURE

## 112. Operational vs Security failure

Ejemplo:

```text
Password incorrect
    security/credential failure

Redis unavailable
    operational failure
```

## 113. No misclassification

No aumentar brute force counter por:
database timeout

## 114. Authentication Diagnostic Code

Podrá existir:

- AUTH-DIAG-...
- para soporte interno.

## 115. Stable public error

Separado de diagnostic code.

## 116. Example

Interno:
AUTH-DIAG-RISK-0042
Externo:
AUTH_ADDITIONAL_VERIFICATION_REQUIRED

## 117. Operational Diagnostics

VoltStack deberá poder inspeccionar:

- active Authenticator profile
- effective policies
- session backend
- rate limiter backend
- risk providers
- event listeners
- audit backend
- sin mostrar secrets.

## 118. Diagnostic Snapshot

Puede existir:

- AuthenticationDiagnosticSnapshot
- solo en tooling autorizado.

## 119. Debug Mode

En desarrollo puede mostrar explicaciones más detalladas.

## 120. Production

Nunca habilitar automáticamente exposición detallada solo porque hubo exception.

## 121. Authentication Audit Query

Podrá ofrecer API para consultar:

- by Identity
- by time range
- by event type
- by security severity
- by device
- by session reference

## 122. Authorization

Consultar audit requiere Authorization explícita.

## 123. Tenant Isolation

Tenant admin solo debe ver:

- its tenant audit
- según policy.

## 124. Platform security staff

Puede tener vista cross-tenant separada.

## 125. Audit Query Service

interface AuthenticationAuditQueryServiceInterface
{
public function search(
AuthenticationAuditQuery $query
): AuthenticationAuditPage;
}

## 126. Pagination

Obligatoria.
No cargar millones de registros.

## 127. Time-bound queries

Recomendadas.

## 128. Indexes

Audit storage deberá considerar:

- occurred_at
- tenant
- identity reference
- event type
- severity
- según backend.

## 129. Audit retention

Debe existir:
AuthenticationAuditRetentionPolicy

## 130. Retention examples

normal login successes:
shorter retention

security changes:
longer retention

account recovery:
longer retention

admin actions:
longest retention

## 131. Legal/compliance requirements

Deben ser configurables por deployment/tenant, no hardcoded.

## 132. Right to deletion vs security audit

La aplicación deberá resolver según regulación aplicable.
VoltStack proporcionará primitives, no decisiones legales universales.

## 133. Pseudonymization after retention threshold

Puede soportarse:

```text
IdentityReference → pseudonymous historical subject
cuando policy lo permita.
```

## 134. Audit archival

Puede mover records antiguos a:

- cold storage
- immutable archive
- SIEM archive

## 135. Audit Lifecycle

HOT
↓
WARM
↓
ARCHIVED
↓
EXPIRED / PURGED

## 136. Purge process

Debe ser auditable para records regulados.

## 137. SIEM Integration

VoltStack deberá permitir exportar eventos de seguridad hacia:

- Splunk
- Elastic
- Microsoft Sentinel
- Datadog
- custom SIEM
- OpenTelemetry-compatible pipeline
- mediante adapters.

## 138. SIEM Event Projection

No enviar eventos internos completos.
Usar:
SecurityEventProjection

## 139. SIEM schema

Podrá incluir:

- timestamp
- event type
- severity
- tenant reference
- identity pseudonym
- network summary
- risk
- result
- reason codes
- correlation

## 140. Secret-free invariant

Obligatorio.

## 141. Common security schema

VoltStack podrá mapear en el futuro a estándares como:

- ECS
- OCSF
- OpenTelemetry semantic conventions
- mediante adapters.

## 142. Native internal model

Seguirá siendo independiente.

## 143. Alerts

Observability deberá permitir condiciones como:

- credential stuffing spike
- recovery abuse spike
- high-risk logins
- device credential replay
- audit store failure
- session revocation failure
- OIDC verification failures
- Passkey counter anomalies

## 144. Alerting system boundary

Auth produce:

- Security Signals
- Metrics
- Security Events

El sistema de alertas decide cómo notificar.

## 145. No SMTP inside Risk Engine

Naturalmente.

## 146. Dashboard concepts

Puede haber dashboards para:

- Authentication success rate
- Failure rate
- MFA completion rate
- Passkey adoption
- Federated login ratio
- Risk distribution
- Abuse trends
- Recovery trends
- Device trust activity
- Session revocation

## 147. Security Dashboard

Además:

- high-risk authentication
- credential stuffing
- password spraying
- MFA fatigue
- recovery replay
- device compromise

## 148. Operational Dashboard

auth latency
Redis limiter latency
session store latency
risk provider latency
OIDC provider errors
event listener latency

## 149. SLOs

VoltStack podrá facilitar definir SLOs como:

- Authentication availability
- Authentication p95 latency
- Session establishment success

Federated provider success rate

## 150. Security SLOs

También pueden existir:

- security event delivery latency
- audit persistence reliability
- session revocation propagation

## 151. Telemetry Sampling

No todos los logs/traces tienen igual valor.

## 152. Sampling examples

normal success:
sampled

invalid password:
aggregated / sampled depending on volume

high-risk:
retain

security compromise:
retain

technical error:
retain

## 153. Audit is not sampled arbitrarily

Si policy dice que un hecho es auditable:

- must record it
- independientemente del sampling de telemetry.

## 154. Log Flood Protection

Un atacante puede provocar:

- millions of invalid logins
- y convertir logging en DoS.

## 155. Strategies

sampling
aggregation
rate-limited repetitive logs
metrics instead of per-request logs
security event summaries

## 156. Never hide critical unique security events

La agregación deberá ser cuidadosa.

## 157. Log Injection

User-controlled values deberán estructurarse.

```php
No construir:
Log::warning("Login failed for ".$input);
sin sanitation.
```

## 158. Structured fields

Mejor:

```php
$logger->warning('authentication.failed', [
    'identity_ref' => $safeRef,
]);
```

## 159. Newline/control character safety

Log backend adapter deberá manejarlo.

## 160. Metric Flood Protection

Dynamic labels controlados.

## 161. Trace Flood Protection

Sampling + span limits.

## 162. Event Metadata Size Limits

Eventos deberán tener:

- maximum metadata size
- bounded arrays
- bounded reason codes

## 163. External Provider Payloads

Nunca guardar responses completas por default.

## 164. OIDC claims

Persistir solo:
necessary normalized claims / audit summary

## 165. WebAuthn

No guardar complete binary assertions en observability.

## 166. Recovery

Nunca URL/token completo.

## 167. Session IDs

No registrar cookie/session token real.

## 168. SessionPublicReference

Usar un ID de observabilidad distinto.

## 169. Token Fingerprints

Para algunos tokens se puede guardar:

- non-secret public identifier
- pero nunca hash simple de un low-entropy secret.

## 170. Credential Public Reference

Cada credential puede tener:

- CredentialPublicId
- para audit.

## 171. AuthenticationOutcomeSummary

Objeto común:

- SUCCESS
- FAILURE
- CHALLENGE
- DENIED
- CANCELLED
- EXPIRED

## 172. Observability severity

Separada del resultado.

```text
Ejemplo:
Failure + expected invalid password
    low severity
```

Success + compromised credential
critical severity

## 173. SecuritySeverity

INFO
LOW
MEDIUM
HIGH
CRITICAL

## 174. Event → severity mapping

Debe ser policy/configurable.

## 175. AuthenticationExplainabilityService

Contrato:

```php
interface AuthenticationExplainabilityServiceInterface
{
    public function explain(
        AuthenticationDecisionContext $context
    ): AuthenticationDecisionExplanation;
}
```

## 176. Explainability sources

Authenticator results
Identity eligibility
Authentication requirements
Assurance
Risk
Device trust
Abuse protection
Policy evaluation
Flow state

## 177. Explanation building

No debería depender de parsing logs después.

## 178. Decision artifacts

Cada subsystem deberá producir:

- structured reason codes
- en tiempo real.

## 179. Example

Risk engine:

- HIGH_RISK
- NEW_DEVICE
- RECENT_RECOVERY

MFA:
PHISHING_RESISTANT_REQUIRED
Finalizer:
SESSION_ESTABLISHED

## 180. Final explanation

Authentication succeeded after phishing-resistant Step-Up
because the login originated from a new device shortly after account recovery.
para operator/developer view.

## 181. Explainability without exposing attack surface

User-safe:
For your security, additional verification was required.

## 182. Debug trace explanation

En testing puede generar:

- PasswordAuthenticator -> VERIFIED
- Eligibility -> ACTIVE
- Risk -> HIGH

Rule RiskHighRequiresPhishingResistant -> MATCH
Passkey -> VERIFIED
Assurance -> REQUIREMENT_SATISFIED
Finalization -> SUCCESS

## 183. Testing utility

Muy útil para pruebas de policies.

## 184. AuthenticationDecisionTrace

Podrá existir como estructura de desarrollo/testing.

## 185. Production storage

No necesariamente persistir full decision trace en cada request.

## 186. Policy Explainability

Cada policy/rule deberá tener:

- PolicyId
- PolicyVersion
- RuleId

## 187. Example

Policy:
admin.authentication.v4

Rule:
RISK_HIGH_REQUIRE_PASSKEY

Matched:
true

## 188. Custom extensions

También deberán declarar stable reason codes.

## 189. No arbitrary user-facing strings in Core

Core produce:

- reason codes
- UI/localization produce mensajes.

## 190. Observability Extension Points

VoltStack deberá permitir:

- custom Audit Store
- custom Log Sink
- custom Metric Exporter
- custom Trace Exporter
- custom SIEM Projection
- custom Explainability Formatter
- custom Retention Policy

## 191. No custom observer may receive secrets by default

Explicit security projection required.

## 192. AuthenticationTelemetryProjection

Podrá producir datos mínimos:

- method
- result
- duration
- risk level
- challenge type

## 193. Audit Projection

Más rico:

- actor
- subject
- tenant
- reason
- security operation
- correlation

## 194. Security Projection

Puede incluir:

- network risk summary
- device status
- risk signals

con controles de acceso.

## 195. Projection Architecture

Internal Authentication Event
↓
Projection Policy
├── AuditProjection
├── TelemetryProjection
├── SecurityProjection
└── ExternalProjection

## 196. Access Control

Observability data no es públicamente accesible.

## 197. Audit Viewer

Requiere:

- Authorization
- Tenant scope
- Security role

## 198. Trace Viewer

Similar.

## 199. Risk explanations

Pueden contener información altamente sensible para attackers.

## 200. Security Operations Boundary

Debe existir una vista/servicio separado.

## 201. Audit Immutability vs Privacy

Si ciertos campos deben modificarse por regulación, podría aplicarse:

- cryptographic erasure
- pseudonymization
- external subject mapping

sin alterar el hecho histórico.

## 202. Cryptographic erasure

Opcional:

- PII encrypted with per-subject key
- key deleted later
- audit event remains

## 203. Advanced feature

No necesario V1, pero arquitectura no deberá impedirlo.

## 204. Multi-tenant Audit Stores

Podrán existir:

- shared store + tenant partition
- per-tenant database
- external tenant SIEM

## 205. Tenant Routing

AuthenticationAuditStoreResolver podrá decidir backend.

## 206. Tenant cannot route into another tenant store

Invariante.

## 207. Global platform audit

Puede existir separado del tenant audit.

## 208. Dual-write

Algunos eventos críticos pueden ir a:

- tenant audit
- +;
- platform security audit

## 209. Avoid inconsistent transaction assumptions

Si dual-write externo falla, aplicar explicit delivery policy.

## 210. Audit Export

Podrá soportar:

- JSON
- NDJSON
- CSV
- SIEM stream
- según tooling.

## 211. Export security

Debe:

- authorize
- scope tenant
- redact
- paginate/stream

audit the export itself

## 212. Audit of Audit

Acciones como:

- view sensitive audit
- export audit
- change retention
- purge audit

también pueden ser auditadas.

## 213. Meta-audit

Importante para compliance enterprise.

## 214. Authentication Diagnostic Events

Pueden incluir:

- AuthConfigurationLoaded
- AuthenticatorRegistryCompiled
- RiskProviderUnavailable
- EventListenerFailed
- AuditStoreUnavailable

## 215. Startup Diagnostics

VoltStack podrá validar:

- audit backend
- metrics exporter
- trace exporter
- redaction config
- durante bootstrap.

## 216. No secrets in startup diagnostics

Naturalmente.

## 217. Observability Health

Podrá exponer:

- HEALTHY
- DEGRADED
- UNAVAILABLE
- por backend.

## 218. Authentication Health Report

Podrá incluir:

- session store
- rate limiter store
- audit store
- risk providers
- federation metadata
- observability exporters

## 219. Health != secret config dump

Nunca.

## 220. Health and Authentication Policy

High-security deployments podrán decidir:

```text
audit unavailable
    → reject privileged authentication
```

## 221. Normal consumer deployment

Puede:
continue + alert

## 222. Failure Mode Registry

Cada observability backend tendrá:

- criticality
- failure mode
- buffer policy
- retry policy

## 223. Audit retries

Deben ser idempotentes.

## 224. AuditRecordId

Permite deduplication.

## 225. External sink retries

No generar records duplicados indiscriminadamente.

## 226. At-least-once assumption

External systems deben tolerarlo.

## 227. Observability Backpressure

Si exporter está lento:

- do not indefinitely block Authentication
- salvo fail-closed policy explícita.

## 228. Queue bounds

Toda cola local deberá tener:

- capacity
- drop policy
- critical event path

## 229. Critical vs non-critical telemetry

Separar canales.

## 230. Example

normal trace
droppable under pressure

credential compromise audit
not droppable under same policy

## 231. FrankenPHP

Todo contexto de observabilidad request-specific deberá limpiarse.

## 232. Prohibido

static $currentFlowId;
static $currentIdentity;
static $currentTrace;
como state mutable shared.

## 233. Execution Context

Usar:

- RequestScope
- FiberLocal
- ExecutionContext
- según runtime abstraction.

## 234. Logger singleton

Puede compartirse si es stateless.

## 235. Metric exporter singleton

También.

## 236. Trace provider singleton

También, siempre que maneje context propagation correctamente.

## 237. Audit writer

Puede ser singleton si no retiene current Identity/request mutable.

## 238. Worker reset

Debe limpiar:

- current trace context
- temporary correlation
- per-request buffers
- decision traces

## 239. Fiber Safety

Dos Authenticaciones concurrentes no deberán mezclar:

- FlowId
- AttemptId
- Identity
- Tenant
- Trace
- Explanation

## 240. Distributed tracing

Node A puede iniciar Flow y Node B continuar.

## 241. Authentication Flow Correlation

FlowId permitirá correlacionar incluso si TraceId cambia entre requests.

## 242. Important distinction

Trace
one distributed operation/request chain

Authentication Flow
may span multiple independent requests

## 243. Therefore

No usar TraceId como FlowId.

## 244. Session Correlation

SessionPublicReference permite investigar:

- created
- rotated
- restored
- revoked
- expired

## 245. Credential Correlation

CredentialPublicReference para:

- issued
- rotated
- revoked
- compromised
- sin guardar secret.

## 246. Recovery Correlation

RecoveryTransactionPublicReference para toda recuperación.

## 247. Security Incident Correlation

Futuro:
SecurityIncidentId
puede agrupar:

- compromised credential
- session revocations
- recovery
- device compromise

## 248. Alert deduplication

Correlation ayuda a no generar:

- 50 alerts
- por un mismo incidente.

## 249. User Notifications

Observability no envía directamente notifications.
Pero sus Events pueden alimentar notification subsystem.

## 250. Audit notification relationship

Ambos pueden consumir mismo domain event.

## 251. Authentication Reports

Tooling futuro podrá generar:

- login history
- security activity
- active sessions
- device history
- recovery history
- risk activity

## 252. User-visible login history

Debe usar:
safe projection

## 253. Example

August 21
Chrome on Windows
Monterrey area
Passkey
Successful
sin revelar sensitive backend details.

## 254. Location uncertainty

Debe expresarse como aproximada.

## 255. User-visible activity

No debe mostrar:

- risk score 89
- threat feed X
- internal policy ID

## 256. Security operator view

Sí puede mostrar información más rica.

## 257. Development Tooling

Podrá incluir comandos:

- auth:audit:tail
- auth:audit:search
- auth:metrics:list
- auth:trace:explain
- auth:decision:explain
- auth:observability:health
- futuros.

## 258. Auth Decision Explain CLI

Ejemplo:

- Flow F-123
- Decision: STEP_UP_REQUIRED

Reasons:

```text
- NEW_DEVICE
- RISK_HIGH
```

Missing Requirement:

- phishing_resistant

## 1. No raw secrets in CLI output

Obligatorio.

## 2. Configuration conceptual

return [

'authentication' => [

'observability' => [

'audit' => [
'enabled' => true,
'store' => 'database',
],

'logging' => [
'enabled' => true,
'channel' => 'security',
],

'metrics' => [
'enabled' => true,
],

'tracing' => [
'enabled' => true,
'sampling' => 'adaptive',
],

'explainability' => [
'enabled' => true,
],

],

],

];

## 261. Audit configuration

'audit' => [

'retention' => [
'authentication_success' => '90 days',
'security_changes' => '2 years',
'administrative_actions' => '5 years',
],

'integrity' => [
'enabled' => false,
],

];
Valores ilustrativos.

## 262. Logging configuration

'logging' => [

'channel' => 'authentication',

'redact' => [
'authorization',
'password',
'token',
'otp',
],

];

## 263. Metrics config

'metrics' => [

'provider' => 'opentelemetry',

'high_cardinality_labels' => false,

];

## 264. Tracing config

'tracing' => [

'provider' => 'opentelemetry',

'sample' => [
'success' => 0.05,
'failure' => 0.25,
'high_risk' => 1.0,
'critical' => 1.0,
],

];
Solo ejemplo conceptual.

## 265. Explainability config

'explainability' => [

'persist_decisions' => false,

'developer_mode' => env('APP_DEBUG', false),

];

## 266. Audit Record example

Event:
DEVICE_TRUSTED

Actor:
SELF

Subject:
Identity usr_public_...

```text
Tenant:
    tenant_public_...
```

Outcome:
SUCCESS

Reason:
STRONG_AUTHENTICATION_COMPLETED

Correlation:

```text
    Flow F-...
    Session S-...
```

Time:
2026-08-21T...

## 267. Failed Login audit

Event:
AUTHENTICATION_FAILED

Authenticator:
PASSWORD

Outcome:
FAILURE

External Reason:
INVALID_CREDENTIALS

Internal Category:
CREDENTIAL_FAILURE

Risk:
MODERATE

## 268. High-risk login audit

Event:
AUTHENTICATION_STEP_UP_REQUIRED

Reasons:

```text
    NEW_DEVICE
    NEW_COUNTRY
    RECENT_ACCOUNT_RECOVERY
```

Requirement:
PHISHING_RESISTANT

Policy:
admin-auth-v4

## 269. Global logout audit

Event:
GLOBAL_LOGOUT_COMPLETED

Actor:
SELF

Subject:
same Identity

Result:

```text
    8 sessions revoked
    3 Remember-Me credentials revoked
```

Reason:
USER_REQUEST

## 270. Recovery audit

Recovery Started
↓
Recovery Evidence Verified
↓
Recovery Authorized
↓
Credentials Re-established
↓
Sessions Revoked
↓
Recovery Completed

## 271. Full observability flow

Authentication Request
│
▼
Trace Started
│
▼
Authentication Flow
│
├── Structured Logs
├── Metrics
├── Domain Events
└── Decision Reasons
│
▼
Authentication Result
│
├── Audit Projection
├── Security Projection
├── Telemetry Projection
└── Explainability Record
│
▼
External Backends

## 272. Architecture overview

AUTHENTICATION CORE
│
▼
AUTHENTICATION EVENTS
│
┌────────────────┼──────────────────┐
▼                ▼                  ▼
AUDIT ROUTER     TELEMETRY ROUTER    EXPLANATION
│                │                  │
┌───────┼──────┐    ┌────┼─────┐            │
▼       ▼      ▼    ▼    ▼     ▼            ▼
DATABASE  SIEM   ARCHIVE LOG METRIC TRACE  DECISION RECORD
│       │      │    │    │     │            │
└───────┴──────┴────┴────┴─────┴────────────┘
│
▼
SECURITY OPERATIONS

## 273. Testing — Secret Redaction

Debe probar explícitamente que nunca aparecen en:

- logs
- audit
- metrics
- traces
- events
- exceptions
- queues

valores como:

- password
- OTP
- Bearer Token
- Recovery Token
- Refresh Token
- Client Secret

## 274. Canary Secret Testing

Muy útil.
Crear credential fake:
SECRET_CANARY_XYZ
ejecutar flows y verificar que no aparezca en ningún output de observability.

## 275. Testing — Audit

Casos:

- successful append
- failure
- buffer
- retry
- duplicate delivery
- ordering
- retention

## 276. Testing — Actor/Subject

Especialmente administrative recovery y impersonation-adjacent operations.

## 277. Testing — Tenant Isolation

Tenant A no puede consultar audit de Tenant B.

## 278. Testing — Metric Cardinality

Verificar que Identity IDs no aparezcan como labels.

## 279. Testing — Tracing

Verificar:

- correct parent/child spans
- FlowId correlation
- no secret attributes

## 280. Testing — Explainability

Dada una policy conocida, explanation debe producir:

- correct reason codes
- correct requirements
- correct policy reference

## 281. Testing — Disclosure

Anonymous user debe recibir explicación más limitada que Security Operator.

## 282. Testing — Sampling

High-risk/critical events no deben perderse según policy.

## 283. Testing — Log Flood

Simular:

- 100k invalid login attempts
- y verificar que logging no cause resource exhaustion.

## 284. Testing — Outbox

commit succeeds
dispatch fails
retry succeeds
sin duplicar audit semánticamente.

## 285. Testing — Integrity Chain

Si habilitada:

- record tampering
- debe detectarse.

## 286. Testing — Retention

Records expiran según policy.

## 287. Testing — Pseudonymization

Si se habilita, debe preservar agregación permitida sin revelar Identity original.

## 288. Testing — Store Failure

audit unavailable
metric backend unavailable
trace exporter unavailable
SIEM unavailable
con cada failure policy.

## 289. Testing — FrankenPHP

Request A:

- Flow Alice
- Trace A
- Tenant A

Request B:

- Flow Bob
- Trace B
- Tenant B

No se mezclan contexts.

## 290. Testing — Fiber Concurrency

Dos traces concurrentes conservan correlation correcta.

## 291. Testing — Distributed Flow

Flow inicia Node A, continúa Node B.
Audit debe seguir correlacionado por FlowId.

## 292. Testing — Event Replay

Reprocesar evento audit no debe recrear Authentication state.

## 293. Fuzz testing

Especialmente:

- metadata
- provider errors
- external claims
- reason payloads
- structured log fields

## 294. Property-based testing

Útil para:

- redaction
- reason composition
- retention rules
- audit append ordering
- metric label validation

## 295. Security invariants — Observability

AUTH-OBS-01
Observability is not an Authentication authority.

- AUTH-OBS-02
- Failure of telemetry does not silently alter credential validity.
- AUTH-OBS-03

Observability contexts are request/execution scoped.

- AUTH-OBS-04
- Correlation identifiers are non-secret.
- AUTH-OBS-05

Logging, metrics and tracing never become the source of truth for Authentication state.

## 296. Security invariants — Secrets

AUTH-OBS-SECRET-01
Raw credentials never enter logs.

- AUTH-OBS-SECRET-02
- Raw credentials never enter metric labels.
- AUTH-OBS-SECRET-03

Raw credentials never enter trace attributes.

- AUTH-OBS-SECRET-04
- Raw credentials never enter audit records.
- AUTH-OBS-SECRET-05

Raw credentials never enter external SIEM payloads.
AUTH-OBS-SECRET-06
Session cookies and persistent login secrets are never logged.

## 297. Security invariants — Audit

AUTH-AUDIT-01
Authentication audit records are append-oriented.

- AUTH-AUDIT-02
- Actor and subject are represented separately.
- AUTH-AUDIT-03

Tenant context is explicit.

- AUTH-AUDIT-04
- Audit access is Authorization-protected.
- AUTH-AUDIT-05

Critical audit store failure follows an explicit policy.

- AUTH-AUDIT-06
- Audit records contain reason codes rather than secrets.
- AUTH-AUDIT-07

Audit retention is policy-driven.

## 298. Security invariants — Metrics

AUTH-METRIC-01
Metrics avoid unbounded cardinality.

- AUTH-METRIC-02
- Identity, email, token and session identifiers are not metric labels.
- AUTH-METRIC-03

Security-critical counters remain distinguishable from operational counters.
AUTH-METRIC-04
Metric exporter failure does not silently fail Authentication unless explicitly required by policy.

## 299. Security invariants — Tracing

AUTH-TRACE-01
Trace context is not Authentication evidence.

- AUTH-TRACE-02
- Trace attributes are secret-free.
- AUTH-TRACE-03

Trace and Authentication Flow identifiers remain distinct.

- AUTH-TRACE-04
- Distributed trace propagation cannot cross tenant boundaries incorrectly.
- AUTH-TRACE-05

Trace sampling does not determine whether mandatory Audit is recorded.

## 300. Security invariants — Explainability

AUTH-EXPLAIN-01
Authentication decisions expose structured reason codes.

- AUTH-EXPLAIN-02
- User-facing explanations are filtered through disclosure policy.
- AUTH-EXPLAIN-03

Explainability does not reveal credential validity unnecessarily.
AUTH-EXPLAIN-04
Risk explanations expose policy/model versions where appropriate.
AUTH-EXPLAIN-05
Explainability is derived from structured decision artifacts rather than inferred from log text.

## 301. Security invariants — Privacy

AUTH-OBS-PRIV-01
Observability follows data minimization.

- AUTH-OBS-PRIV-02
- PII retention is explicit.
- AUTH-OBS-PRIV-03

Long-term telemetry should prefer pseudonymous or aggregate data.
AUTH-OBS-PRIV-04
Tenant observability data remains isolated.
AUTH-OBS-PRIV-05
Sensitive security intelligence is not exposed to ordinary users.

## 302. Security invariants — Runtime

AUTH-OBS-RT-01
Current correlation state is request/fiber scoped.

- AUTH-OBS-RT-02
- Shared loggers/exporters do not retain mutable Identity state.
- AUTH-OBS-RT-03

FrankenPHP workers clear request-local observability state.

- AUTH-OBS-RT-04
- Concurrent fibers maintain isolated traces and explanations.
- AUTH-OBS-RT-05

Distributed flows use explicit correlation identifiers.

## 303. Anti-pattern — Log::info($request->all())

Nunca en Authentication.

## 304. Anti-pattern — Authorization header in error log

Nunca.

## 305. Anti-pattern — Email as metric label

Alta cardinalidad + privacy leak.

## 306. Anti-pattern — Session token as trace attribute

Nunca.

## 307. Anti-pattern — Audit == logs

No.

## 308. Anti-pattern — Traces are permanent audit

No.

## 309. Anti-pattern — Every successful login stored forever

No necesariamente.

## 310. Anti-pattern — Audit mutable rows

Evitar.

## 311. Anti-pattern — Risk score exposed to anonymous user

No.

## 312. Anti-pattern — Full third-party threat feed response persisted

No.

## 313. Anti-pattern — Technical exception counted as invalid credential

No.

## 314. Anti-pattern — All logs synchronous

Puede provocar DoS.

## 315. Anti-pattern — All security events sampled

No si son audit-critical.

## 316. Anti-pattern — TraceId reused as FlowId

No.

## 317. Anti-pattern — Global mutable current trace context

Crítico bajo FrankenPHP.

## 318. Componentes principales

AuthenticationObservabilityManager
AuthenticationObservabilityContext

AuthenticationAuditRecorder
AuthenticationAuditRecord
AuthenticationAuditStore
AuthenticationAuditQueryService
AuthenticationAuditProjectionPolicy
AuthenticationAuditRetentionPolicy

AuthenticationLogger
AuthenticationRedactionPolicy

AuthenticationMetricRecorder
AuthenticationMetric
AuthenticationHistogram

AuthenticationTracer
AuthenticationTraceContext
AuthenticationSpan

AuthenticationExplainabilityService
AuthenticationDecisionExplanation
AuthenticationExplanationDisclosurePolicy

## 319. Correlation components

AuthenticationCorrelation
RequestId
TraceId
AuthenticationFlowId
AuthenticationAttemptId
SessionPublicReference
CredentialPublicReference
RecoveryTransactionPublicReference

## 320. Projection components

AuditEventProjection
TelemetryEventProjection
SecurityEventProjection
SiemEventProjection
UserSecurityActivityProjection

## 321. Integrity components

AuditIntegrityProvider
AuditHashChainProvider
ProtectedAuditRecord
AuditIntegrityVerifier

## 322. Storage/export components

DatabaseAuthenticationAuditStore
AppendOnlyAuthenticationAuditStore
CompositeAuthenticationAuditStore

AuthenticationSiemExporter
AuthenticationTelemetryExporter
AuthenticationTraceExporter

## 323. Namespace sugerido

VoltStack\Quantum\Auth\Observability
VoltStack\Quantum\Auth\Observability\Contracts
VoltStack\Quantum\Auth\Observability\Audit
VoltStack\Quantum\Auth\Observability\Logging
VoltStack\Quantum\Auth\Observability\Metrics
VoltStack\Quantum\Auth\Observability\Tracing
VoltStack\Quantum\Auth\Observability\Explainability
VoltStack\Quantum\Auth\Observability\Projection
VoltStack\Quantum\Auth\Observability\Redaction
VoltStack\Quantum\Auth\Observability\Correlation
VoltStack\Quantum\Auth\Observability\Export

## 324. Estructura sugerida

src/Quantum/Auth/Observability/
├── Contracts/
│   ├── AuthenticationObservabilityManagerInterface.php
│   ├── AuthenticationAuditStoreInterface.php
│   ├── AuthenticationMetricRecorderInterface.php
│   ├── AuthenticationTracerInterface.php
│   └── AuthenticationExplainabilityServiceInterface.php
│
├── Audit/
│   ├── AuthenticationAuditRecord.php
│   ├── AuditRecordId.php
│   ├── AuditActor.php
│   ├── AuthenticationAuditOutcome.php
│   ├── AuthenticationAuditRecorder.php
│   ├── AuthenticationAuditQuery.php
│   ├── AuthenticationAuditQueryService.php
│   ├── AuthenticationAuditProjectionPolicy.php
│   ├── AuthenticationAuditRetentionPolicy.php
│   └── AuditIntegrityProvider.php
│
├── Logging/
│   ├── AuthenticationLogger.php
│   ├── AuthenticationLogContext.php
│   ├── AuthenticationLogLevelPolicy.php
│   └── AuthenticationRedactionPolicy.php
│
├── Metrics/
│   ├── AuthenticationMetric.php
│   ├── AuthenticationHistogram.php
│   ├── AuthenticationMetricRecorder.php
│   └── AuthenticationMetricLabelPolicy.php
│
├── Tracing/
│   ├── AuthenticationTracer.php
│   ├── AuthenticationTraceContext.php
│   ├── AuthenticationSpan.php
│   └── AuthenticationTraceSamplingPolicy.php
│
├── Explainability/
│   ├── AuthenticationDecisionExplanation.php
│   ├── AuthenticationExplainabilityService.php
│   ├── AuthenticationReasonCode.php
│   ├── AuthenticationExplanationDisclosurePolicy.php
│   ├── RiskExplanation.php
│   └── AssuranceExplanation.php
│
├── Projection/
│   ├── AuditEventProjection.php
│   ├── TelemetryEventProjection.php
│   ├── SecurityEventProjection.php
│   ├── SiemEventProjection.php
│   └── UserSecurityActivityProjection.php
│
├── Correlation/
│   ├── AuthenticationCorrelation.php
│   ├── RequestId.php
│   ├── TraceId.php
│   └── SessionPublicReference.php
│
├── Export/
│   ├── AuthenticationSiemExporter.php
│   ├── AuthenticationAuditExporter.php
│   └── AuthenticationTelemetryExporter.php
│
└── AuthenticationObservabilityManager.php

## 325. Integración con sistema de eventos

La relación recomendada será:

```text
Domain Event
    ↓
Projection
    ↓
```

Audit / Logging / Metrics / Trace
No todos los subsistemas deberán llamar manualmente a cada backend.

## 326. Ejemplo

PasskeyRegistered
│
├── AuditProjection
│      ↓
│   Audit Store
│
├── MetricProjection
│      ↓
│   auth_passkey_registered_total
│
└── SecurityActivityProjection
↓
User Security History

## 327. Authentication Failure Flow observable

POST /login
↓
Trace
↓
AttemptId
↓
PasswordAuthenticator
↓
INVALID
↓
AuthenticationFailed Event
├── Metric increment
├── Abuse detector
├── Structured security log
└── Audit according to policy

## 328. Authentication Success Flow observable

AuthenticationSucceeded
↓
Finalization
↓
SessionEstablished
↓
AuthenticationFinalized
├── Audit
├── Metrics
├── Trace completion
├── Login history
└── Optional notification

## 329. High-risk flow observable

Password VALID
↓
Risk HIGH
↓
RiskExplanation:

```text
    NEW_DEVICE
    NEW_COUNTRY
        ↓
Step-Up Required
        ↓
Passkey VERIFIED
        ↓
Authentication Finalized
        ↓
```

Audit:
high-risk authentication completed with step-up

## 330. Recovery flow observable

RecoveryStarted
↓
RecoveryEvidenceVerified
↓
RecoveryAuthorized
↓
CredentialReestablished
↓
SessionsRevoked
↓
RecoveryCompleted
↓
Audit / Notification / Security History

## 331. Relación con Laravel y Symfony

Laravel ofrece primitives útiles mediante:

- logging
- events
- Monolog
- metrics integrations
- Telescope
- Pulse
- custom audit packages

Symfony ofrece una arquitectura sólida mediante:

- Monolog
- EventDispatcher
- Profiler
- Stopwatch
- Security events
- Messenger
- OpenTelemetry ecosystem

VoltStack deberá aprovechar esas ideas, pero formalizar un modelo específico de Authentication donde:

- Audit
- Logs
- Metrics
- Tracing
- Explainability
- sean subsistemas diferentes.

## 332. Diferenciador VoltStack

En lugar de un sistema que solo diga:
Login failed.
VoltStack podrá reconstruir internamente:

```text
Flow:
    F-123
```

Attempt:
A-456

Authenticator:
Password

Credential:
VERIFIED

Identity:
ACTIVE

Risk:
HIGH

Reasons:

```text
    NEW_DEVICE
    RECENT_RECOVERY
```

Requirement:
PHISHING_RESISTANT

Step-Up:
Passkey VERIFIED

Result:
AUTHENTICATION_FINALIZED

Session:
S-public-789
sin conservar el password, OTP, token o cualquier secret utilizado en el proceso.

## 333. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Audit, Logging, Metrics, Tracing and Explainability are separate concerns

## 2. Authentication secrets never enter observability payloads

## 3. Audit is append-oriented and security-focused

## 4. Logs are structured and diagnostic

## 5. Metrics are aggregate and low-cardinality

## 6. Traces reconstruct execution but are not durable security records

## 7. Explainability uses structured reason codes

## 8. Authentication Flow, Attempt, Trace and Session identifiers remain distinct

## 9. Observability projections minimize PII

## 10. Tenant boundaries apply to audit and observability

## 11. Sampling never replaces mandatory Audit

## 12. Security-critical observability failures have explicit policies

## 13. Long-running runtimes isolate correlation state per request/fiber

## 14. Decision provenance includes policy/model versions when relevant

## 15. SIEM and external systems receive projected, redacted payloads

## 16. Criterios de aceptación

El subsistema será considerado completo cuando:
17. soporte Authentication Audit;
18. soporte append-oriented audit;
19. distinga Actor y Subject;
20. soporte Tenant-aware Audit;
21. soporte Audit Retention;
22. soporte Audit Query;
23. soporte Audit Integrity extensible;
24. soporte Audit Store failure policies;
25. soporte structured logging;
26. soporte secret redaction;
27. soporte PII classification;
28. soporte Authentication metrics;
29. evite high-cardinality labels;
30. soporte counters;
31. soporte histograms;
32. soporte tracing;
33. soporte distributed trace correlation;
34. distinga TraceId y FlowId;
35. soporte Attempt correlation;
36. soporte Session correlation;
37. soporte Recovery correlation;
38. soporte explainability;
39. soporte structured reason codes;
40. soporte Risk explanations;
41. soporte Assurance explanations;
42. soporte disclosure policies;
43. soporte policy/model version provenance;
44. soporte external SIEM projection;
45. soporte telemetry projections;
46. soporte user security activity projections;
47. soporte sampling;
48. soporte log flood protection;
49. soporte event payload size bounds;
50. soporte audit export;
51. audite sensitive audit access/export;
52. soporte retention/purge lifecycle;
53. soporte observability health;
54. soporte backend failure policies;
55. soporte multi-tenancy;
56. soporte privacy controls;
57. soporte audit/telemetry separation;
58. soporte OpenTelemetry adapters;
59. soporte custom exporters;
60. soporte custom Audit Stores;
61. sea fiber-safe;
62. sea seguro con FrankenPHP;
63. nunca use observability como Authentication authority;
64. nunca persista raw credentials.
65. Regla arquitectónica final

VoltStack deberá preservar:

```text
AUTHENTICATION DECISION
        │
        ├──────────────────────────────┐
        │                              │
        ▼                              ▼
```

STRUCTURED SECURITY FACT          DECISION REASONS
│                              │
▼                              ▼
EVENT PROJECTION                EXPLAINABILITY
│
┌──────┼────────┬─────────┐
▼      ▼        ▼         ▼
AUDIT   LOG    METRIC     TRACE
│       │       │          │
▼       ▼       ▼          ▼
STORE   SINK   BACKEND    COLLECTOR
La primera regla central será:
VoltStack deberá poder reconstruir qué ocurrió durante una Authentication sin almacenar las credenciales que permitieron ejecutarla.

La segunda:
Audit conservará evidencia histórica; Logging diagnosticará; Metrics medirán; Tracing correlacionará ejecución; Explainability describirá por qué se tomó una decisión. Ninguna de estas capas sustituirá a las demás.

La tercera:
Toda decisión relevante de Authentication deberá poder expresarse mediante reason codes, requisitos, assurance, risk y referencias de policy suficientemente estructuradas para ser investigable y verificable.

La cuarta:
Observabilidad nunca deberá comprometer la confidencialidad de Passwords, Tokens, OTP, Recovery Credentials, Session Secrets, Remember-Me Credentials, Passkeys ni secretos federados.

Siguiente documento recomendado
La secuencia natural continúa con:
`25_AUTHENTICATION_FAILURE_ERROR_EXCEPTION_DENIAL_AND_SECURITY_RESPONSE_HANDLING_SYSTEM.md`
Este documento debería formalizar una capa que ya hemos tocado parcialmente, pero que merece existir como sistema independiente:

- Authentication Failure
- Credential Failure
- Identity Failure
- Authentication Denial
- Challenge Required
- Step-Up Required
- Flow Expiration
- Provider Failure
- Infrastructure Failure
- Security Policy Failure
- Risk Denial
- Abuse Throttling
- Authentication Exceptions
- Exception Taxonomy
- Failure Classification
- Public Error Mapping
- Internal Error Mapping
- HTTP Status Mapping
- SPA Error Mapping
- JSON/API Mapping
- Redirect Failures
- Retryability
- Retry-After
- Safe User Messages
- Localization
- Account Enumeration Resistance
- Secret Redaction
- Failure Correlation
- Failure Audit
- Failure Metrics
- Failure Recovery Paths

Fail-Open / Fail-Closed Policies
FrankenPHP Exception Isolation
La separación sería importante:

```text
FAILURE
    resultado esperado del dominio

DENIAL
    política de seguridad rechaza Authentication

CHALLENGE
    aún falta evidencia

EXCEPTION
    fallo inesperado/técnico

ERROR RESPONSE
    representación segura del resultado hacia el cliente
```

Eso evitará que VoltStack caiga en el patrón típico de convertir absolutamente todo en una AuthenticationException.
