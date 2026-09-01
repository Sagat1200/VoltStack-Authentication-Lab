# VoltStack Authentication System

## 40 — Authentication Security Notification, Alerting, Compromise Detection, Incident Response and Account Protection System

- **Archivo:** `40_AUTHENTICATION_SECURITY_NOTIFICATION_ALERTING_COMPROMISE_DETECTION_INCIDENT_RESPONSE_AND_ACCOUNT_PROTECTION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Identity Protection / Incident Response
- **Dependencias principales:** 12, 13, 15, 16, 17, 18, 19, 20, 21, 23, 24, 25, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39.

---

## 1. Propósito

Este documento define el subsistema encargado de detectar, clasificar, contener, comunicar y responder ante eventos que indiquen que una identidad, sesión, dispositivo, credencial o método de autenticación podría estar comprometido.
Hasta este punto VoltStack dispone de mecanismos para:

```text
Authenticate
     │
     ├── verify credentials
     ├── establish sessions
     ├── evaluate risk
     ├── manage devices
     ├── perform MFA
     ├── use passkeys
     ├── federate identities
     ├── recover accounts
     ├── perform reauthentication
     ├── manage credentials
     ├── inspect security state
     ├── compose Authentication Policy
     ├── classify Assurance
     ├── negotiate Challenges
     └── secure Authentication Transactions
```

Sin embargo, un sistema empresarial necesita responder también a:
What happens when Authentication looks wrong?
La arquitectura propuesta será:

```text
Authentication Signals
        │
        ▼
Compromise Detection
        │
        ▼
Security Finding
        │
        ▼
Correlation
        │
        ▼
Security Alert
        │
        ▼
Authentication Incident
        │
        ▼
Response Policy
        │
   ┌────┼──────────┬─────────────┐
   ▼    ▼          ▼             ▼
Notify Contain   Restrict      Investigate
│
▼
Account Protection
│
▼
Recovery / Security Review
│
▼
Normal Security State
```

## 2. Principio fundamental

Authentication Security no termina cuando las credenciales son válidas. Debe continuar evaluando si la actividad autenticada sigue siendo coherente, legítima y segura.

Una autenticación técnicamente válida puede haber sido realizada por:

- stolen password
- stolen session
- stolen refresh token
- compromised passkey device
- compromised OAuth account
- malicious recovery
- stolen API credential
- malware
- phishing
- session replay
- credential stuffing
- SIM swap
- insider abuse

Por ello:

```text
Valid Authentication
        ≠
Legitimate Authentication
```

## 3. Objetivos

El sistema deberá proporcionar:

1. Security Signals.
2. Compromise Detection.
3. Security Findings.
4. Signal correlation.
5. Security Alerts.
6. Authentication Incidents.
7. Incident severity.
8. Incident lifecycle.
9. Automated containment.
10. Account restriction.
11. Session containment.
12. Credential containment.
13. Device containment.
14. Authentication method protection.
15. Security notifications.
16. User confirmation.
17. Security review.
18. Recovery integration.
19. Administrative investigation.
20. Distributed incident propagation.
21. Multi-tenant isolation.
22. Machine identity incident response.
23. Auditability.
24. Explainability.
25. Extensibility.
26. Lo que este sistema no es

No deberá convertirse en:

- SIEM completo
- SOC platform
- general fraud engine
- EDR
- network IDS
- WAF
- Authorization engine
- generic notification framework

VoltStack podrá integrarse con esos sistemas.

```text
Su responsabilidad será:
Authentication-centric identity compromise detection and response.
```

## 27. Separación de dominios

Debe conservarse:

```text
Risk Engine
    │
    │ evaluates risk
    ▼
Risk Signal

Compromise Detection
    │
    │ determines security concern
    ▼
Security Finding

Alerting
    │
    │ determines attention required
    ▼
Security Alert

Incident Response
    │
    │ coordinates response
    ▼
Authentication Incident

Account Protection
    │
    │ changes security state
    ▼
```

Protected / Restricted Identity

## 6. Risk Signal != Security Finding

Documento 20 produce señales como:

- new_device
- new_country
- unusual_time
- high_velocity
- impossible_travel
- anonymous_network
- failed_attempts

Una señal individual no implica necesariamente compromiso.

## 7. Security Finding

Representa una conclusión de seguridad más fuerte:

```php
final readonly class AuthenticationSecurityFinding
{
    public function __construct(
        public AuthenticationSecurityFindingId $id,
        public AuthenticationSecurityFindingType $type,
        public AuthenticationSecuritySeverity $severity,
        public PrincipalReference $principal,
        public array $evidence,
        public DateTimeImmutable $detectedAt,
        public AuthenticationSecurityConfidence $confidence,
    ) {}
}
```

## 8. Ejemplo

Signal:
New Country

Signal:
Unknown Device

Signal:
Password Login

Signal:

```text
Impossible Travel

            ↓
```

Finding:
PROBABLE_ACCOUNT_TAKEOVER

## 9. Finding != Incident

Un finding puede:

- be ignored
- be correlated
- produce alert
- increase risk
- open incident
- según policy.

## 10. Security Signal

Modelo común:

```php
interface AuthenticationSecuritySignalInterface
{
    public function type(): AuthenticationSecuritySignalType;

    public function occurredAt(): DateTimeImmutable;

    public function subject(): SecuritySignalSubject;

    public function attributes(): array;
}
```

## 11. Signal Sources

Podrán provenir de:

- Authentication
- Sessions
- Remember-Me
- MFA
- Passkeys
- OAuth/OIDC
- Recovery
- Devices
- Risk Engine
- Transaction Security
- Credential Management
- Security Center
- Machine Authentication
- Policy Engine
- External Security Providers
- Threat Intelligence
- Administrator Actions
- User Reports

## 12. Tipos de señales

Ejemplos:

```text
LOGIN_SUCCESS
LOGIN_FAILURE

NEW_DEVICE
NEW_LOCATION
NEW_COUNTRY

IMPOSSIBLE_TRAVEL

PASSWORD_FAILURE_BURST
PASSWORD_SPRAY_PATTERN
CREDENTIAL_STUFFING_PATTERN

MFA_FAILURE
MFA_FATIGUE_PATTERN

RECOVERY_STARTED
RECOVERY_COMPLETED

PASSWORD_CHANGED
MFA_REMOVED
PASSKEY_REMOVED

EXTERNAL_ACCOUNT_LINKED
EXTERNAL_ACCOUNT_UNLINKED

SESSION_REPLAY
NONCE_REPLAY
AUTH_STATE_REPLAY

SESSION_FROM_NEW_NETWORK

DEVICE_TRUST_CHANGED

API_CREDENTIAL_USED

MACHINE_CREDENTIAL_ANOMALY

SECURITY_EPOCH_CHANGED
```

## 13. Signal Classification

Las señales deberán clasificarse por dominio:

- Identity
- Credential
- Session
- Device
- Network
- Location
- Authentication Method
- Recovery
- Federation
- Machine Identity
- Transaction Security
- Administrative

## 14. Signal Subject

final readonly class SecuritySignalSubject
{
public function __construct(
public PrincipalReference|null $principal,
public SessionReference|null $session,
public DeviceReference|null $device,
public CredentialReference|null $credential,
public TenantId|null $tenant,
public AuthenticationRealmId|null $realm,
) {}
}

## 15. Anonymous Signals

Algunas señales ocurren antes de conocer identidad.

- Ejemplo:
- credential stuffing
- password spray
- enumeration attempt

Por ello principal puede ser inicialmente desconocido.

## 16. Correlation

Después puede correlacionarse con:

- credential identifier
- email hash
- account lookup
- network
- device
- provider
- tenant
- respetando privacidad.

## 17. Signal Normalization

Diferentes proveedores pueden producir:

- "new_device"
- "unknown-device"
- "DEVICE_NEW"

El Core deberá normalizarlos.

## 18. Signal Normalizer

interface AuthenticationSecuritySignalNormalizerInterface
{
public function normalize(
mixed $signal
): AuthenticationSecuritySignalInterface;
}

## 19. Confidence

enum AuthenticationSecurityConfidence: string
{
case Low = 'low';
case Medium = 'medium';
case High = 'high';
case VeryHigh = 'very_high';
case Confirmed = 'confirmed';
}

## 20. Severity

Confidence y Severity no son iguales.

## 21. Severity Model

enum AuthenticationSecuritySeverity: string
{
case Informational = 'informational';
case Low = 'low';
case Medium = 'medium';
case High = 'high';
case Critical = 'critical';
}

## 22. Ejemplo

Finding:
User logged in from new browser

Confidence:
CONFIRMED

Severity:
LOW
Mientras:

```text
Finding:
Probable stolen session
```

Confidence:
HIGH

Severity:
CRITICAL

## 23. Finding Types

VoltStack deberá proporcionar inicialmente:

- SUSPICIOUS_LOGIN
- PROBABLE_ACCOUNT_TAKEOVER
- CREDENTIAL_COMPROMISE
- SESSION_COMPROMISE
- DEVICE_COMPROMISE
- RECOVERY_ABUSE
- MFA_ABUSE
- MFA_FATIGUE
- FEDERATED_IDENTITY_COMPROMISE
- AUTHENTICATION_REPLAY
- AUTHENTICATION_BINDING_ATTACK
- BRUTE_FORCE
- CREDENTIAL_STUFFING
- PASSWORD_SPRAY
- UNEXPECTED_SECURITY_CHANGE
- PRIVILEGED_ACCOUNT_ANOMALY
- MACHINE_IDENTITY_COMPROMISE
- SERVICE_CREDENTIAL_COMPROMISE

## 24. Compromise Detection Engine

interface AuthenticationCompromiseDetectionEngineInterface
{
public function analyze(
AuthenticationSecuritySignalSet $signals,
AuthenticationCompromiseContext $context
): AuthenticationSecurityFindingSet;
}

## 25. Detection Pipeline

Raw Security Events
↓
Normalize
↓
Enrich
↓
Correlate
↓
Evaluate Detection Rules
↓
Calculate Confidence
↓
Classify Severity
↓
Produce Findings

## 26. Detection Rules

Reglas podrán ser:

- deterministic
- threshold-based
- temporal
- correlation-based
- risk-based
- provider-based
- behavioral
- plugin-defined

## 27. Deterministic Rule

Ejemplo:

- valid session token
- +
- session already revoked
- +
- reuse detected

puede producir:

- SESSION_REPLAY
- con confianza alta.

## 28. Threshold Rule

20 password failures
within 2 minutes
for same identity
→ posible brute force.

## 29. Temporal Rule

Password Changed
↓ 30 seconds
Recovery Codes Regenerated
↓ 15 seconds
All Passkeys Removed
puede ser altamente sospechoso.

## 30. Correlation Rule

New Device
+
New Country
+
Password Login
+
MFA Reset
puede elevarse a:
PROBABLE_ACCOUNT_TAKEOVER

## 31. Behavioral Detection

Podrá existir como extensión.
El Core no deberá depender de Machine Learning.

## 32. Detection Rule Contract

interface AuthenticationCompromiseDetectionRuleInterface
{
public function evaluate(
AuthenticationCompromiseContext $context
): AuthenticationSecurityFindingSet;
}

## 33. Rule Metadata

final readonly class AuthenticationDetectionRuleDefinition
{
public function __construct(
public AuthenticationDetectionRuleId $id,
public string $version,
public AuthenticationSecuritySeverity $defaultSeverity,
public bool $enabled,
) {}
}

## 34. Detection Rules Versioning

Importante para investigación:
Why was this incident created?
Debe poder responderse:

```text
Detection Rule:
ATO-IMPOSSIBLE-TRAVEL-03
```

Version:
7

## 35. Finding Evidence

Evidence puede contener referencias seguras a:

- events
- sessions
- devices
- credentials
- locations
- providers
- risk signals
- transaction attacks

## 36. Evidence != Secrets

Nunca:

- password
- raw session token
- TOTP secret
- recovery code
- private key
- raw API token
- OAuth refresh token

## 37. Evidence Snapshot

Puede guardar metadata necesaria para investigación incluso si el objeto original posteriormente cambia.

## 38. Evidence Integrity

Incidentes críticos pueden requerir integridad reforzada.
Integración con documento 31.

## 39. Security Alert

Un Alert significa:
Existe una situación de Authentication Security que requiere conocimiento o acción.

## 1. Alert Model

final readonly class AuthenticationSecurityAlert
{
public function __construct(
public AuthenticationSecurityAlertId $id,
public AuthenticationSecurityAlertType $type,
public AuthenticationSecuritySeverity $severity,
public PrincipalReference|null $principal,
public AuthenticationSecurityFindingId|null $finding,
public AuthenticationSecurityAlertStatus $status,
public DateTimeImmutable $createdAt,
) {}
}

## 2. Alert Status

enum AuthenticationSecurityAlertStatus: string
{
case Open = 'open';
case Acknowledged = 'acknowledged';
case Investigating = 'investigating';
case Resolved = 'resolved';
case Dismissed = 'dismissed';
}

## 3. Alert != Notification

Muy importante:

- Alert
- es security state.
- Notification
- es delivery.

## 4. Ejemplo

Un único Alert:
Suspicious login detected
puede producir:

- Email
- Push Notification
- Security Center Banner
- Admin Console Alert
- Webhook
- SIEM Event

## 5. Alert Deduplication

Evitar:

```text
500 failed login signals
→ 500 user alerts
```

## 6. Alert Correlation

Podrán agruparse:

- same identity
- same incident
- same finding type
- same device
- same time window

## 7. Alert Fingerprint

final readonly class AuthenticationAlertFingerprint
{
public function __construct(
public string $value,
) {}
}

## 8. Fingerprint Purpose

Permite:

- deduplication
- aggregation
- suppression
- correlation

sin depender del mensaje visual.

## 9. Alert Fatigue

VoltStack deberá evitar alert fatigue.

## 10. Severity-Based Notification

Ejemplo conceptual:

```text
INFO
→ Security Center only

LOW
→ Security Center

MEDIUM
→ Security Center + email

HIGH
→ Security Center + email + push

CRITICAL
→ immediate multi-channel + admin/security alert
configurable.
```

## 50. Notification System

interface AuthenticationSecurityNotificationDispatcherInterface
{
public function dispatch(
AuthenticationSecurityNotification $notification
): AuthenticationSecurityNotificationResult;
}

## 51. Notification Channels

Adapters podrán soportar:

- Email
- SMS
- Push
- Web Push
- In-App
- Security Center
- Webhook
- Slack
- Teams
- PagerDuty
- SIEM
- SOC Platform
- Custom

El Core no dependerá de ellos.

## 52. Security Notification Types

NEW_LOGIN
NEW_DEVICE
PASSWORD_CHANGED
PASSWORD_RESET
MFA_ENABLED
MFA_DISABLED
PASSKEY_ADDED
PASSKEY_REMOVED
RECOVERY_USED
RECOVERY_CODES_REGENERATED
EXTERNAL_ACCOUNT_LINKED
EXTERNAL_ACCOUNT_UNLINKED
SESSION_REVOKED
GLOBAL_LOGOUT
API_CREDENTIAL_CREATED
API_CREDENTIAL_REVOKED
SUSPICIOUS_LOGIN
ACCOUNT_RESTRICTED
ACCOUNT_LOCKED
SECURITY_FREEZE
COMPROMISE_DETECTED
SECURITY_REVIEW_REQUIRED

## 53. Mandatory Notifications

Algunas notificaciones críticas no deberían poder desactivarse por usuario.
Ejemplos:

- password changed
- recovery method changed
- MFA disabled
- passkey removed
- external identity linked
- account recovery completed
- security freeze

## 54. Notification Policy

interface AuthenticationSecurityNotificationPolicyInterface
{
public function channelsFor(
AuthenticationSecurityNotification $notification,
AuthenticationSecurityNotificationContext $context
): AuthenticationSecurityNotificationPlan;
}

## 55. Notification Delivery Failure

Regla:
Un fallo al enviar una notificación no debe revertir una mutación de seguridad ya completada.

Ejemplo:

```text
Password changed successfully
        ↓
Email service unavailable
```

No:
rollback password change

## 56. Delivery Retry

Utilizar:

- Outbox
- Queue
- Retry
- Dead-letter
- cuando corresponda.

## 57. Notification Outbox

Security Mutation
│
├── DB Commit
│
└── Notification Outbox
│
▼
Dispatcher

## 58. Notification Content

Nunca incluir:

- password
- full recovery codes
- raw API tokens
- session tokens
- TOTP secrets
- private keys

## 59. Security Links

Links dentro de notificaciones deberán usar el sistema del documento 39.

## 60. Example

"Review this login"
→ short-lived purpose-bound security continuation.

## 61. Notification Link Purpose

SECURITY_ALERT_REVIEW
no deberá reutilizarse como:

- LOGIN
- PASSWORD_RESET
- ACCOUNT_LINKING

## 62. Authentication Incident

Un Incident agrupa uno o varios findings relacionados que representan una investigación/respuesta de seguridad.

## 63. Incident Model

final readonly class AuthenticationSecurityIncident
{
public function __construct(
public AuthenticationSecurityIncidentId $id,
public AuthenticationSecurityIncidentType $type,
public AuthenticationSecuritySeverity $severity,
public AuthenticationIncidentStatus $status,
public PrincipalReference|null $principal,
public TenantId|null $tenant,
public DateTimeImmutable $openedAt,
public DateTimeImmutable|null $resolvedAt,
) {}
}

## 64. Incident Status

enum AuthenticationIncidentStatus: string
{
case Open = 'open';
case Triaged = 'triaged';
case Containing = 'containing';
case Contained = 'contained';
case Investigating = 'investigating';
case Recovering = 'recovering';
case Resolved = 'resolved';
case Closed = 'closed';
}

## 65. Incident Lifecycle

DETECTED
↓
OPEN
↓
TRIAGED
↓
CONTAINING
↓
CONTAINED
↓
INVESTIGATING
↓
RECOVERING
↓
RESOLVED
↓
CLOSED
No todos los incidentes necesitan todas las fases.

## 66. Incident Type

Inicialmente:

- ACCOUNT_TAKEOVER
- SESSION_THEFT
- CREDENTIAL_COMPROMISE
- DEVICE_COMPROMISE
- RECOVERY_COMPROMISE
- FEDERATED_IDENTITY_COMPROMISE
- PRIVILEGED_ACCOUNT_COMPROMISE
- MACHINE_IDENTITY_COMPROMISE
- AUTHENTICATION_REPLAY_ATTACK
- AUTHENTICATION_INFRASTRUCTURE_ATTACK

## 67. Incident Correlation

Un incident puede contener:

- 5 suspicious logins
- 2 unknown devices
- 1 password change
- 1 MFA removal

## 68. Incident Correlation Engine

interface AuthenticationIncidentCorrelationEngineInterface
{
public function correlate(
AuthenticationSecurityFinding $finding
): AuthenticationIncidentCorrelationResult;
}

## 69. Existing Incident

Puede devolver:
ATTACH_TO_EXISTING

## 70. New Incident

O:
OPEN_NEW_INCIDENT

## 71. No Incident

O:
NO_INCIDENT_REQUIRED

## 72. Incident Severity Escalation

MEDIUM incident
+
confirmed session replay
=

CRITICAL incident

## 73. Severity Downgrade

Deberá ser explícito/auditable.
No silencioso.

## 74. Incident Timeline

Cada incident deberá tener timeline.

## 75. Timeline Entry

final readonly class AuthenticationIncidentTimelineEntry
{
public function __construct(
public DateTimeImmutable $occurredAt,
public AuthenticationIncidentTimelineEventType $type,
public ActorReference|null $actor,
public array $metadata,
) {}
}

## 76. Timeline Examples

Finding detected
Alert opened
User notified
User reported "not me"
Session revoked
Identity restricted
Password reset
Passkeys reviewed
Security review completed
Incident resolved

## 77. Automated Response

VoltStack deberá soportar respuesta automática.

## 78. Response Policy

interface AuthenticationIncidentResponsePolicyInterface
{
public function plan(
AuthenticationSecurityIncident $incident,
AuthenticationIncidentResponseContext $context
): AuthenticationIncidentResponsePlan;
}

## 79. Response Plan

Puede contener:

- notify
- step-up
- reauthenticate
- revoke session
- revoke sessions
- revoke remember-me
- revoke device trust
- revoke credential
- increment security epoch
- restrict identity
- freeze security changes
- require recovery
- require security review
- disable machine credential
- notify administrators
- emit SIEM event

## 80. Response Actions

enum AuthenticationIncidentResponseActionType: string
{
case Notify = 'notify';
case RequireReauthentication = 'require_reauthentication';
case RequireStepUp = 'require_step_up';
case RevokeSession = 'revoke_session';
case RevokeAllSessions = 'revoke_all_sessions';
case RevokeCredential = 'revoke_credential';
case RevokeDeviceTrust = 'revoke_device_trust';
case IncrementSecurityEpoch = 'increment_security_epoch';
case RestrictIdentity = 'restrict_identity';
case ApplySecurityFreeze = 'apply_security_freeze';
case RequireRecovery = 'require_recovery';
case RequireSecurityReview = 'require_security_review';
}

## 81. Containment

Objetivo:
Limitar rápidamente el poder del posible atacante.

## 1. Containment Levels

enum AuthenticationContainmentLevel: string
{
case None = 'none';
case Observe = 'observe';
case Soft = 'soft';
case Strong = 'strong';
case Full = 'full';
}

## 2. Observe

record
alert
increase monitoring
sin modificar Authentication.

## 3. Soft Containment

Ejemplo:

- require fresh authentication
- disable remember-me restoration
- require MFA

## 4. Strong Containment

Ejemplo:

- revoke suspicious session
- revoke device trust
- invalidate selected credentials

## 5. Full Containment

Ejemplo:

- revoke all sessions
- increment security epoch
- disable persistent login
- restrict account
- freeze credential changes
- require recovery

## 6. Containment Scope

Puede aplicarse a:

- session
- device
- credential
- authentication method
- tenant
- identity
- realm
- machine identity

## 7. Least-Destructive Containment

No revocar todo automáticamente si puede aislarse el artefacto comprometido con suficiente confianza.

## 8. Example

single API token leaked
preferiblemente:
revoke token
no necesariamente:
lock entire human account

## 9. But Critical Compromise

Si attacker modificó:

- password
- MFA
- recovery
- passkeys
- federated accounts

puede requerirse full containment.

## 10. Account Protection State

Documento 35 introdujo Security Posture/Freeze.
Este documento formaliza protección reactiva.

## 11. Identity Protection State

enum IdentityProtectionState: string
{
case Normal = 'normal';
case Monitoring = 'monitoring';
case Challenged = 'challenged';
case Restricted = 'restricted';
case Frozen = 'frozen';
case RecoveryRequired = 'recovery_required';
case SecurityReviewRequired = 'security_review_required';
}

## 12. Protection State != Account Status

No confundir:

- ACTIVE
- SUSPENDED
- DELETED

con:

- RESTRICTED
- FROZEN
- RECOVERY_REQUIRED

## 13. Restricted Authentication

Puede permitir:

- security center
- recovery
- support contact
- logout
- security review

y bloquear:

- financial actions
- credential creation
- tenant administration
- API token creation
- privileged operations

## 14. Authorization Interaction

Authentication Protection State será input de Authorization cuando corresponda.
Pero Authentication no decidirá permisos de recursos.

## 15. Security Freeze

Puede bloquear temporalmente:

- new sessions
- new credentials
- method linking
- method removal
- device trust
- recovery changes
- API credential creation

## 16. Freeze Existing Sessions

Policy decide si:

```text
freeze

+

revoke all sessions
ocurre conjuntamente.
```

## 98. Freeze Reason

Debe registrarse.

```php
enum AuthenticationSecurityFreezeReason: string
{
    case SuspectedCompromise = 'suspected_compromise';
    case ConfirmedCompromise = 'confirmed_compromise';
    case UserRequested = 'user_requested';
    case AdministratorRequested = 'administrator_requested';
    case IncidentResponse = 'incident_response';
}
```

## 99. Security Epoch

Documento 39.
Full containment normalmente puede:
identity_security_epoch++

## 100. Why Epoch

Permite invalidar:

- sessions
- reauth proofs
- continuations
- pending authentication transactions
- recovery transactions
- según sus bindings.

## 101. Session Revocation

Puede ser:

- single session
- suspicious sessions
- all other sessions
- all sessions

## 102. Session Selection

No confiar únicamente en IP para determinar cuáles son maliciosas.

## 103. Credential Containment

Puede afectar:

- password
- passkey
- TOTP
- recovery credential
- API token
- device credential
- machine credential
- federated credential

## 104. Password Compromise

Respuesta posible:

- mark password compromised
- disable password auth temporarily
- require stronger method
- require password replacement
- revoke sessions

## 105. Password Compromised != Identity Compromised

Si attacker nunca logró login, puede bastar con:

- force password rotation
- dependiendo de policy.

## 106. Passkey Compromise

Puede revocarse credential específica.

## 107. Lost Device

Documento 35.

```text
Lost device
    ↓
revoke device trust
    ↓
revoke device-bound credentials
    ↓
revoke sessions
según cascade policy.
```

## 108. TOTP Compromise

Puede:

- revoke TOTP factor
- require alternate strong authentication
- re-enroll MFA

## 109. Recovery Compromise

Muy sensible.
Puede requerir:

- invalidate recovery codes
- remove compromised recovery channel
- security review
- cooldown

## 110. Federated Compromise

Si Google/Microsoft/enterprise IdP identity comprometida:

- disable provider method
- require alternate authentication

unlink only after safe verification

## 111. Provider-Wide Compromise

Documento 39 permite:
provider_security_epoch++

## 112. Machine Identity Compromise

Documento 33.

- Response puede:
- revoke machine credential
- rotate signing key
- revoke certificate
- disable workload identity

increment machine security epoch
quarantine service identity

## 113. Machine Containment != Human Lockout

No intentar:
send MFA challenge to Kubernetes workload

## 114. Machine Incident Policy

Debe ser específica para PrincipalType.

## 115. User Confirmation

Security Center podrá preguntar:
Was this you?

## 116. Confirmation Model

enum AuthenticationActivityConfirmation: string
{
case ConfirmedByUser = 'confirmed_by_user';
case DeniedByUser = 'denied_by_user';
case Unknown = 'unknown';
}

## 117. "This Was Me"

Debe producir señal positiva.
No eliminar historial.

## 118. "This Wasn't Me"

Debe considerarse señal de alta confianza.

## 119. Compromise Response Command

final readonly class IdentityCompromiseResponseCommand
{
public function __construct(
public PrincipalReference $principal,
public SecurityActivityReference $activity,
public AuthenticationActivityConfirmation $confirmation,
public SessionReference $currentSession,
) {}
}

## 120. Denied Activity Flow

User:

```text
"This wasn't me"
        │
        ▼
Validate current identity
        │
        ▼
```

Fresh Authentication if appropriate
│
▼
Open/Escalate Incident
│
▼
Contain suspicious artifacts
│
▼
Increase Security Epoch
│
▼
Restrict Identity
│
▼
Security Review

## 121. Compromised Current Session Problem

No siempre puede confiarse en la sesión desde la que se reporta.

## 122. Adaptive Response

Policy puede requerir:

- alternate factor
- recovery method
- passkey
- support-assisted recovery

antes de ejecutar algunas acciones.

## 123. But Emergency Lock

Debe poder existir una operación:

- LOCK MY ACCOUNT
- con threat model específico.

## 124. Self-Lock

Puede priorizar containment sobre continuidad.

## 125. Self-Lock Flow

User suspects compromise
↓
Lock Account
↓
revoke sessions
↓
increment security epoch
↓
freeze authentication changes
↓
require recovery

## 126. Anti-DoS

No permitir que atacante no autenticado bloquee arbitrariamente cuentas con facilidad.

## 127. Account Lock vs Login Throttle

Son conceptos distintos.

## 128. Brute Force Lockout

VoltStack debe evitar diseños que permitan:

```text
attacker submits wrong password repeatedly
→ victim permanently locked
```

## 129. Progressive Protection

Preferir combinaciones como:

- rate limit
- challenge
- risk escalation
- temporary delay
- MFA requirement
- IP/network controls

## 130. Incident Response Orchestrator

interface AuthenticationIncidentResponseOrchestratorInterface
{
public function respond(
AuthenticationSecurityIncident $incident
): AuthenticationIncidentResponseResult;
}

## 131. Orchestration

Incident
↓
Resolve Response Policy
↓
Build Response Plan
↓
Validate Plan
↓
Execute Containment
↓
Persist Results
↓
Audit
↓
Events
↓
Notifications

## 132. Response Plan Validation

Debe evitar acciones contradictorias.

- Ejemplo:
- require current session step-up
- +

revoke current session first
requiere orden explícito.

## 133. Response Ordering

Ejemplo:

1. Establish recovery authority
2. Freeze credential mutation
3. Revoke attacker sessions
4. Increment epoch
5. Require security review
6. Action Dependencies

final readonly class AuthenticationIncidentResponseStep
{
public function __construct(
public AuthenticationIncidentResponseStepId $id,
public AuthenticationIncidentResponseActionType $action,
public array $dependsOn,
) {}
}

## 7. Response Execution State

PENDING
RUNNING
SUCCEEDED
FAILED
SKIPPED
COMPENSATED

## 8. Partial Failure

Ejemplo:

- sessions revoked ✓
- device trust revoked ✓
- notification failed ✗

Containment sigue siendo exitoso parcialmente.

## 9. Critical Failure

Ejemplo:
security epoch increment failed
puede requerir:

- incident escalation
- fail closed
- manual intervention

## 10. Idempotency

Incident response actions deberán ser idempotentes cuando sea posible.

## 11. Example

revoke session X
ejecutado dos veces:

- REVOKED
- ALREADY_REVOKED
- no error destructivo.

## 12. Response Operation ID

final readonly class AuthenticationIncidentResponseOperationId
{
public function __construct(
public string $value,
) {}
}

## 13. Distributed Response

Un incident puede requerir:

```text
Node A → revoke session
Region B → invalidate cache
```

Region C → disable credential

## 14. Distributed Security Epoch

Debe propagarse conforme documento 30/39.

## 15. Revocation Authority

Caches locales nunca serán authority.

## 16. Incident Event Bus

interface AuthenticationIncidentEventBusInterface
{
public function dispatch(
AuthenticationIncidentEvent $event
): void;
}

## 17. Important Events

AuthenticationSecurityFindingCreated
AuthenticationSecurityAlertOpened
AuthenticationSecurityAlertAcknowledged

AuthenticationIncidentOpened
AuthenticationIncidentSeverityChanged
AuthenticationIncidentContained
AuthenticationIncidentResolved

AuthenticationContainmentStarted
AuthenticationContainmentCompleted
AuthenticationContainmentFailed

IdentityProtectionStateChanged
IdentityRestricted
IdentitySecurityFreezeApplied
IdentitySecurityFreezeReleased

IdentityCompromiseConfirmed
IdentityCompromiseDismissed

AuthenticationSecurityReviewRequired
AuthenticationSecurityReviewCompleted

## 146. Audit Requirements

Toda mutación de incident response deberá registrar:

- Actor
- Target Principal
- Tenant
- Realm
- Incident
- Finding
- Action
- Reason
- Policy
- Authentication Context
- Authorization Decision
- Timestamp
- Outcome

## 147. Actor Types

Puede ser:

- USER
- ADMINISTRATOR
- SECURITY_OPERATOR
- SYSTEM
- AUTOMATED_POLICY
- MACHINE

## 148. Automated Action

Debe quedar claro:

```php
Actor = SYSTEM
Reason = INCIDENT_POLICY
```

## 149. Human Override

Un Security Operator puede cambiar response plan si Authorization/Authentication policy lo permite.

## 150. Override Audit

Debe registrar:

- previous plan
- new plan
- actor
- reason
- ticket
- timestamp

## 151. Dual Control

Para incidentes críticos puede requerirse:

- Security Operator A
- +
- Security Operator B

## 152. Separation of Duties

Integración con Authorization system.

## 153. Incident Resolution

Resolver no significa borrar evidencia.

## 154. Resolution Model

final readonly class AuthenticationIncidentResolution
{
public function __construct(
public AuthenticationIncidentResolutionType $type,
public ActorReference $resolvedBy,
public string|null $reasonCode,
public DateTimeImmutable $resolvedAt,
) {}
}

## 155. Resolution Types

CONFIRMED_COMPROMISE
FALSE_POSITIVE
USER_CONFIRMED_ACTIVITY
ADMINISTRATIVE_ACTIVITY
RECOVERED
MITIGATED
EXTERNAL_PROVIDER_RESOLVED
UNKNOWN

## 156. Closed Incident

No deberá reabrirse silenciosamente.
Puede:
reopen explicitly
o:
open related incident

## 157. Security Review

Documento 35 introdujo IdentitySecurityReview.
Aquí se convierte en recovery workflow posterior a incident.

## 158. Review Checklist

Puede incluir:

- Review active sessions
- Review remembered devices
- Review trusted devices
- Review passkeys
- Review MFA methods
- Review linked providers
- Review recovery methods
- Review API tokens
- Review machine delegations
- Change password
- Regenerate recovery codes
- Confirm contact methods

## 159. Review Requirement

final readonly class AuthenticationSecurityReviewRequirement
{
public function__construct(
public array $requiredSteps,
public AuthenticationAssuranceRequirement $completionAssurance,
) {}
}

## 160. Review State

REQUIRED
IN_PROGRESS
BLOCKED
COMPLETED
EXPIRED

## 161. Review Completion

No simplemente:

- clicked "done"
- Debe verificar que pasos obligatorios realmente se completaron.

## 162. Recovery → Review

Flujo recomendado:

```text
Account Compromise
       ↓
Contain
       ↓
Recovery
       ↓
Restricted Authentication
       ↓
Security Review
       ↓
Strong Authentication
       ↓
Restore Normal State
```

## 163. Recovery Does Not Automatically Clear Incident

Muy importante.

## 164. Recovery Success

Solo demuestra suficiente authority para recovery.
No demuestra que:

- all attacker sessions removed
- all malicious credentials removed

all linked accounts safe

## 165. Post-Recovery Cooldown

Puede existir:

- 24h no recovery changes
- 12h no payout change
- 1h no privileged credential creation
- según aplicación/policy.

## 166. Authentication Cooldown

Core puede expresar cooldown security state.

## 167. Cooldown Model

final readonly class AuthenticationSecurityCooldown
{
public function __construct(
public AuthenticationSecurityCooldownType $type,
public DateTimeImmutable $startsAt,
public DateTimeImmutable $endsAt,
) {}
}

## 168. Cooldown != Authorization Rule

Authentication produce state/context.
Authorization/application policy decide qué acciones restringir.

## 169. Security Center Integration

Documento 35 mostrará:

- Active Alerts
- Open Incidents
- Protection State
- Security Review
- Recent Security Actions
- Recommended Actions

## 170. User-Facing Incident View

Debe ser más simple que SOC view.

## 171. User View

Ejemplo:
We detected unusual activity.

```text
Device:
Windows / Chrome
```

Approximate Location:
Monterrey, Mexico

Time:

```php
12:45 AM

Was this you?

[Yes, this was me]
[No, secure my account]
```

## 172. Privacy

No mostrar ubicación con precisión innecesaria.

## 173. IP Addresses

Pueden mostrarse parcialmente o según policy.

## 174. Admin Incident View

Puede contener más metadata:

```php
risk score
finding evidence
session IDs (management references)
credential references
rule versions
correlation
response timeline
sin secretos.
```

## 175. Security Operator View

Puede añadir:

- raw security event references
- SIEM correlation IDs
- incident notes
- containment actions

## 176. Viewer-Aware Projection

interface AuthenticationIncidentProjectionPolicyInterface
{
public function project(
AuthenticationSecurityIncident $incident,
ViewerSecurityContext $viewer
): AuthenticationIncidentProjection;
}

## 177. User != Admin Projection

No cargar todo y esconderlo únicamente en frontend.

## 178. Multi-Tenant Incident Isolation

Tenant A no podrá ver incidentes de Tenant B.

## 179. Global Identity

Una identidad global puede tener:

- global incident
- tenant-specific incident
- realm-specific incident

## 180. Tenant Scope

enum AuthenticationIncidentScope: string
{
case Global = 'global';
case Tenant = 'tenant';
case Realm = 'realm';
case Application = 'application';
}

## 181. Tenant Admin

Puede administrar incidentes dentro de su authority.

- No implica authority sobre:
- global credentials
- other tenants
- platform admin realm

## 182. Platform Security Incident

Puede afectar múltiples tenants.

## 183. Example

Federated provider signing key compromise
puede producir:
Platform Authentication Incident

## 184. Incident Fan-Out

Platform incident puede generar findings relacionados por tenant/identity.

## 185. Avoid Incident Explosion

No crear automáticamente millones de incident rows si un provider global falla.

## 186. Parent Incident

Modelo:

```text
Platform Incident
       │
       ├── Tenant Impact A
       ├── Tenant Impact B
       └── Tenant Impact C
```

## 187. Incident Relationship

final readonly class AuthenticationIncidentRelationship
{
public function __construct(
public AuthenticationSecurityIncidentId $parent,
public AuthenticationSecurityIncidentId $child,
public AuthenticationIncidentRelationshipType $type,
) {}
}

## 188. Compromised Provider

Response:

- disable affected authentication method
- increment provider epoch
- reject new tokens
- require alternate authentication
- notify impacted users

## 189. Signing Key Compromise

Documento 31.

- Puede requerir:
- revoke key
- rotate keys
- invalidate protected states
- open infrastructure incident

## 190. Authentication Infrastructure Incident

Ejemplos:

- session signing key compromised
- replay store corrupted
- MFA provider compromised
- authentication database leak

OAuth client secret leak
certificate authority compromise

## 191. Infrastructure Incident Scope

No necesariamente asociado a un único principal.

## 192. Infrastructure Response

Puede afectar:

- all sessions
- all tokens
- specific provider
- specific realm
- specific tenant
- specific key generation

## 193. Incident Blast Radius

final readonly class AuthenticationIncidentBlastRadius
{
public function __construct(
public AuthenticationIncidentScope $scope,
public array $affectedRealms,
public array $affectedTenants,
public array $affectedCredentialTypes,
) {}
}

## 194. Blast Radius Calculation

Debe evitar asumir:

```text
key compromised
→ everything compromised
```

si key purpose separation del documento 31 limita impacto.

## 195. Cryptographic Purpose Separation Benefit

Ejemplo:
OAuth State Key compromised
no debería implicar automáticamente:
Session Signing Key compromised

## 196. Detection from Document 39

Eventos como:

- nonce replay
- state replay
- binding mismatch
- cross-tenant callback
- cross-realm continuation
- invalid cryptographic state
- alimentarán este sistema.

## 197. Invalid Token Noise

No todo token inválido abre incident.

## 198. Correlation Required

Ejemplo:

```text
1 malformed state
→ noise

500 valid-MAC states with binding mismatch
→ highly suspicious
```

## 199. Credential Stuffing

Detection puede usar:

- many accounts
- same network
- common passwords
- failure velocity

known compromised credential indicators
sin almacenar raw passwords.

## 200. Password Spray

Patrón:

- same password candidate
- across many accounts

El Core no deberá guardar contraseña para detectarlo.

## 201. Safe Password Fingerprinting

Cualquier técnica para correlacionar password attempts debe ser diseñada extremadamente cuidadosamente.
Default:
No correlacionar raw candidate passwords dentro del generic Authentication incident system.

## 1. MFA Fatigue

Patrón:

- many push requests
- user repeatedly denies
- eventually approves
- puede indicar ataque.

## 2. Push MFA

Si se soporta:

- number matching
- transaction details
- rate limiting
- challenge binding

deben preferirse sobre simple Approve/Deny cuando sea posible.

## 3. Recovery Abuse

Señales:

- many recovery attempts
- new recovery method
- immediate password reset
- immediate MFA removal

## 4. Account Takeover Pattern

Ejemplo:

```text
New Country
    ↓
Recovery
    ↓
Password Change
    ↓
MFA Removed
    ↓
Passkey Added
    ↓
All Sessions Revoked
```

Este patrón puede indicar que attacker está expulsando al usuario legítimo.

## 5. Defensive Response

Puede bloquear temporalmente:

- removing last trusted method
- adding new recovery method
- changing security email

hasta completar stronger verification.

## 6. Security Mutation Velocity

El número/velocidad de security mutations puede ser una señal.

## 7. Mutation Types

password
MFA
passkeys
recovery
federation
devices
API tokens

## 8. Credential Takeover Sequence Detection

Debe ser extensible mediante temporal correlation rules.

## 9. Session Theft Detection

Señales posibles:

- same session from incompatible locations
- same session concurrent impossible device profile
- revoked session reused
- session binding anomalies

## 10. Device Fingerprinting Caution

Browser fingerprint no debe tratarse como verdad criptográfica.

## 11. Session Anomaly Confidence

Device/network signals deberán producir probabilistic findings, salvo evidencia criptográfica.

## 12. Confirmed Replay

Documento 39 puede producir evidencia mucho más fuerte.

## 13. Trusted Device Abuse

Trusted device no es permanentemente trusted.

## 14. Device Trust Revocation

Incident response puede eliminar trust sin eliminar device record.

## 15. New Credential After Compromise

Puede ser maliciosa.

- Por ello incident timeline debe relacionar:
- credential enrollment
- con compromise window.

## 16. Compromise Window

final readonly class AuthenticationCompromiseWindow
{
public function __construct(
public DateTimeImmutable|null $suspectedStart,
public DateTimeImmutable|null $confirmedAt,
public DateTimeImmutable|null $containedAt,
) {}
}

## 17. Why

Permite identificar:

- sessions created during compromise
- credentials created during compromise

devices trusted during compromise
providers linked during compromise

## 18. Retroactive Review

Security Review puede destacar estos objetos.

## 19. Retroactive Revocation

Policy puede recomendar o ejecutar revocación de artifacts creados dentro de compromise window.

## 20. False Positives

Sistema debe asumir que existen.

## 21. False Positive Handling

Debe permitir:

- dismiss finding
- resolve incident
- mark activity legitimate
- adjust detection

sin borrar evidencia histórica.

## 22. User Confirmation Is Evidence

No es verdad absoluta.
Un attacker con session comprometida también puede presionar:
"This was me"

## 23. Confirmation Confidence

Dependerá del Authentication Context usado para confirmar.

## 24. Example

Confirmation realizada después de:

- fresh phishing-resistant passkey
- puede tener mayor confianza.

## 25. Confirmation Context

final readonly class AuthenticationSecurityConfirmationContext
{
public function __construct(
public AuthenticationContext $authentication,
public AuthenticationAssuranceProfile $assurance,
public SessionReference $session,
) {}
}

## 26. High-Risk Confirmation

Puede exigir alternate independent method.

## 27. Security Alert Acknowledgement

Acknowledged no significa:

- legitimate
- resolved
- false positive

Solo:
viewer saw it

## 28. Incident Notes

Administradores podrán añadir notas.

## 29. Notes Security

No permitir secrets.

## 30. Sensitive Incident Data

Puede requerir:

- encryption at rest
- restricted authorization
- audit reads
- retention policy

## 31. Retention

Security findings/incidents podrán tener mayor retención que normal application logs.

## 32. Privacy

Pero no conservar indefinidamente:

- IP
- location
- device metadata
- sin política.

## 33. Data Minimization

Guardar únicamente lo necesario para:

- security
- investigation
- compliance
- response

## 34. Data Classification

PUBLIC
INTERNAL
SECURITY_SENSITIVE
SECRET
Incident data normalmente:
SECURITY_SENSITIVE

## 35. Export to SIEM

Contrato:

```php
interface AuthenticationSecurityEventExporterInterface
{
    public function export(
        AuthenticationSecurityExportEvent $event
    ): void;
}
```

## 36. SIEM Payload

Debe ser:

- structured
- versioned
- redacted
- correlatable

## 37. Example Fields

event_type
timestamp
severity
tenant
realm
principal pseudonymous reference
incident
finding
rule
outcome

## 38. OpenTelemetry

Puede emitir spans/events.

## 39. Trace Names

auth.security.signal.process
auth.security.compromise.detect
auth.security.finding.create
auth.security.alert.open
auth.security.incident.correlate
auth.security.incident.respond
auth.security.containment.execute
auth.security.notification.dispatch

## 40. Metrics

Ejemplos:

- auth_security_signals_total
- auth_security_findings_total
- auth_security_alerts_total
- auth_security_incidents_open_total
- auth_security_incidents_resolved_total
- auth_security_containment_total
- auth_security_notifications_total
- auth_security_notification_failures_total
- auth_security_compromise_confirmed_total
- auth_security_false_positive_total

## 41. Metric Labels

Permitidos con cardinalidad controlada:

- signal_type
- finding_type
- incident_type
- severity
- result
- channel

## 42. Forbidden Metric Labels

No:

- user_id
- email
- session_id
- IP
- device_id
- incident_id

## 43. Detection Latency

Métrica importante:

- time from signal
- to finding

## 44. Containment Latency

time from finding
to containment

## 45. Recovery Latency

time from containment
to recovery

## 46. MTTD / MTTC / MTTR

Authentication security puede calcular:

- Mean Time To Detect
- Mean Time To Contain

Mean Time To Recover

## 47. Policy Engine Integration

Documento 36 determinará:

```text
finding severity
→ required response
cuando corresponda.
```

## 48. Example Policy

IF
finding = AUTHENTICATION_REPLAY
AND
confidence >= HIGH

THEN
revoke session
increment security epoch
require security review

## 250. Static vs Dynamic Policy

Response policy puede combinar:

- platform policy
- tenant policy
- realm policy
- incident type
- risk
- identity type
- assurance

## 251. Tenant Hardening

Tenant puede exigir respuesta más fuerte.

## 252. Tenant Weakening

Tenant no podrá reducir framework/platform security floor.

## 253. Incident Policy Version

Toda response debe registrar policy version.

## 254. Explainability

Debe poder responder:

- Why was my account restricted?
- de forma segura.

## 255. Safe Explanation

Ejemplo:

- We detected a sign-in from a new device
- followed by changes to your security settings.

## 256. Unsafe Explanation

Evitar revelar:

- our impossible-travel threshold is 734 km/h
- our bot detector score was 0.8121

si ayuda a evadir detección.

## 257. Internal Explanation

Puede ser mucho más detallada.

## 258. Detection Explainability

final readonly class AuthenticationCompromiseExplanation
{
public function __construct(
public array $reasonCodes,
public array $supportingSignals,
public AuthenticationSecurityConfidence $confidence,
) {}
}

## 259. Reason Codes

Ejemplos:

- NEW_DEVICE_AFTER_RECOVERY
- SESSION_REPLAY_CONFIRMED
- MFA_REMOVAL_AFTER_NEW_LOGIN
- MULTIPLE_SECURITY_MUTATIONS
- IMPOSSIBLE_TRAVEL_HIGH_CONFIDENCE

## 260. External Threat Intelligence

VoltStack podrá integrar:

- compromised credential feeds
- malicious IP intelligence
- known proxy/Tor intelligence
- device reputation
- breach intelligence

## 261. Threat Intelligence Is Untrusted Input

Debe normalizarse y calificarse.

## 262. Provider Confidence

Cada source puede tener:

- trust level
- freshness
- version

## 263. Provider Failure

Threat intelligence failure no deberá romper login por defecto salvo policy explícita.

## 264. Critical Internal Signals

Por contraste:

- confirmed nonce replay
- no depende de external provider.

## 265. Security Signal Bus

interface AuthenticationSecuritySignalBusInterface
{
public function publish(
AuthenticationSecuritySignalInterface $signal
): void;
}

## 266. Synchronous Detection

Algunas detecciones deben ejecutarse inline.
Ejemplo:
replay detected

## 267. Asynchronous Detection

Otras pueden ejecutarse posteriormente.
Ejemplo:
behavioral anomaly correlation

## 268. Inline vs Async

Clasificación:

- Preventive Detection
- Reactive Detection
- Analytical Detection

## 269. Preventive

Puede bloquear request actual.

## 270. Reactive

Puede contener inmediatamente después.

## 271. Analytical

Puede generar incident posteriormente.

## 272. Async Race

Un attacker puede actuar antes de que detection async termine.
Por ello señales críticas deben procesarse inline cuando sea necesario.

## 273. Event Ordering

Distributed events pueden llegar fuera de orden.

## 274. Event Timestamp

Conservar:

- occurredAt
- receivedAt
- processedAt

## 275. Event ID

Cada security event debe tener ID idempotente.

## 276. Duplicate Event

No crear dos findings por delivery duplicate sin policy.

## 277. Event Deduplication

interface AuthenticationSecurityEventDeduplicatorInterface
{
public function firstSeen(
AuthenticationSecurityEventId $id
): bool;
}

## 278. Transactional Outbox

Authentication mutations críticas deberán preferir outbox para eventos.

## 279. Example

Password Changed
│
├── password commit
└── security event outbox

## 280. Lost Events

No depender únicamente de:

- dispatch after commit in process memory
- para critical security events.

## 281. Message Integrity

Cross-service security events pueden requerir:

- authenticated transport
- message signing
- trusted broker

## 282. Event Forgery

Un plugin no confiable no debería poder crear arbitrariamente:

- CONFIRMED_COMPROMISE
- sin authority.

## 283. Signal Authority

Signals pueden declarar provenance.

## 284. Signal Provenance

final readonly class AuthenticationSecuritySignalProvenance
{
public function __construct(
public AuthenticationSecuritySignalSource $source,
public string $provider,
public string $version,
public AuthenticationSecurityTrustLevel $trust,
) {}
}

## 285. Trust Level

UNTRUSTED
LOW
NORMAL
HIGH
AUTHORITATIVE

## 286. Plugin Signal

Por defecto no authoritative.

## 287. Core Replay Detector

Puede ser authoritative para replay que él mismo observó.

## 288. User Report

"This wasn't me" puede tener HIGH confidence, pero authority depende del authentication context.

## 289. Admin Report

También requiere Authentication + Authorization.

## 290. Incident Manual Creation

Security operators pueden crear incident manualmente.

## 291. Manual Incident

Debe registrar:

- actor
- reason
- ticket
- evidence

## 292. Privileged Operation

Crear/escalar/cerrar critical incident puede ser privileged operation del documento 32.

## 293. Incident Suppression

Puede existir para known benign behavior.

## 294. Suppression Rule

Debe ser:

- scoped
- time-limited
- audited
- versioned

## 295. No Permanent Ignore

Evitar:

- ignore impossible travel for user forever
- como default.

## 296. Suppression Scope

Puede ser:

- tenant
- service
- known migration
- security test
- penetration test

## 297. Maintenance Windows

Pueden reducir ruido, no desactivar framework security invariants.

## 298. Shadow Detection

Nuevas detection rules pueden ejecutarse:

- SHADOW
- sin incident response.

## 299. Shadow Mode

Útil para:

- measure false positives
- tune thresholds
- validate rollout

## 300. Mandatory Detection

No poner en shadow:

- cryptographically confirmed replay
- si security floor exige bloquearlo.

## 301. Detection Rule Lifecycle

DRAFT
SHADOW
ACTIVE
DEPRECATED
RETIRED

## 302. Canary Rollout

Puede aplicarse a:

- specific tenants
- percentage
- environment
- realm

## 303. Rollback

Detection rule debe poder revertirse rápidamente.

## 304. Response Rule Lifecycle

Separado del Detection Rule.

## 305. Detection != Response

Ejemplo:
detect suspicious login
puede mantenerse igual mientras response cambia de:
notify
a:
require step-up

## 306. Development Environment

No enviar accidentalmente production security notifications.

## 307. Environment Isolation

development
staging
production
deben tener incident namespaces/config separados.

## 308. Test Events

Deben marcarse:
synthetic = true

## 309. Security Drills

VoltStack deberá permitir ejercicios controlados.

## 310. Incident Drill

Ejemplo:

- simulate compromised session
- sin comprometer una real.

## 311. Drill Events

Deben estar claramente clasificados.

## 312. Break-Glass Integration

Documento 32.

- Uso de break-glass debe generar:
- high-priority security event
- aunque sea legítimo.

## 313. Break-Glass Incident

No necesariamente compromiso.
Pero debe ser altamente observable.

## 314. Break-Glass Notification

Puede notificar inmediatamente a Security Team.

## 315. Break-Glass Abuse

Uso fuera de policy puede abrir critical incident.

## 316. Impersonation

Admin impersonation también debe producir security signals.

## 317. Actor vs Subject

Durante impersonation:

```php
Actor = Admin
Effective Subject = User
```

debe conservarse en incident evidence.

## 318. Compromise During Impersonation

No atribuir automáticamente al usuario efectivo.

## 319. Machine Identity Incidents

Modelo puede reutilizar:

- Finding
- Alert
- Incident
- Response

pero no human workflows.

## 320. Machine Recovery

Puede ser:

- rotate credential
- reissue certificate
- redeploy workload
- change trust policy

## 321. Service Quarantine

Puede existir:
MachineIdentityProtectionState::Quarantined

## 322. Workload Identity

Ephemeral workloads pueden simplemente dejar de recibir credentials.

## 323. Certificate Incident

Si client certificate comprometido:

- revoke certificate
- update trust/revocation
- rotate key
- restart workload

## 324. API Credential Leak

Response:

- revoke token
- notify owner
- identify usage

review actions during compromise window

## 325. Delegated Credentials

Revoke delegation without necessarily revoking parent identity.

## 326. Incident Forensics

VoltStack no será forensic suite completa.
Pero debe preservar suficientes referencias.

## 327. Forensic References

audit event IDs
trace IDs
session management IDs
device management IDs
credential management IDs
transaction IDs
provider event IDs
SIEM IDs

## 328. No Secret Forensics

Nunca almacenar secrets solo "por si acaso".

## 329. Evidence Chain

Critical deployments podrán integrar immutable/WORM audit storage.

## 330. Evidence Hashing

Puede integrarse con documento 31 para integrity proofs.

## 331. Security Notification Localization

User notifications podrán localizarse.

## 332. Security Meaning Must Remain Stable

Traducción no debe alterar severity/meaning.

## 333. Notification Templates

Deben estar versionados.

## 334. Template Injection

No insertar raw:

- User-Agent
- device name
- provider claim
- IP metadata
- sin escaping.

## 335. Phishing-Resistant Notification Design

Emails no deberían entrenar al usuario a introducir contraseña desde cualquier link.

## 336. Recommended Pattern

Security email:

```text
"Review activity in your account"
→ safe application security center.
```

## 337. Sensitive Recovery Links

Solo cuando realmente sean necesarios y protegidos por 39.

## 338. Notification Authenticity

Applications pueden proporcionar:

- known sender
- consistent domain
- security center
- para reducir phishing.

## 339. SMS

No debe considerarse canal confidencial fuerte.

## 340. Push

Puede ofrecer richer context, pero sigue necesitando secure application binding.

## 341. Webhook

Tenant webhooks deben:

- sign payload
- retry
- timestamp
- event ID

## 342. Webhook Replay

Consumer podrá verificar:

- timestamp
- signature
- event ID

## 343. Webhook Secret Rotation

Integración documento 31.

## 344. Security Center Alert Action

Acciones:

- THIS_WAS_ME
- THIS_WAS_NOT_ME
- REVIEW_SESSION
- REVOKE_SESSION
- LOCK_ACCOUNT
- START_SECURITY_REVIEW

## 345. UI Is Not Authority

Backend vuelve a evaluar:

- identity
- session
- tenant
- realm
- authorization
- authentication requirement
- incident state

## 346. Concurrency

Dos responders pueden actuar simultáneamente.

## 347. Incident Version

final readonly class AuthenticationIncidentVersion
{
public function __construct(
public int $value,
) {}
}

## 348. Optimistic Concurrency

Mutaciones pueden utilizar compare-and-swap.

## 349. Example

Security Operator A → resolve
User → reports compromise
No perder actualización.

## 350. State Transition Guard

interface AuthenticationIncidentStateMachineInterface
{
public function transition(
AuthenticationSecurityIncident $incident,
AuthenticationIncidentTransition $transition
): AuthenticationSecurityIncident;
}

## 351. Invalid Transition

Ejemplo:

```text
CLOSED
→ CONTAINING
requiere explicit reopen.
```

## 352. Incident Repository

interface AuthenticationSecurityIncidentRepositoryInterface
{
public function get(
AuthenticationSecurityIncidentId $id
): AuthenticationSecurityIncident;

public function save(
AuthenticationSecurityIncident $incident
): void;
}

## 353. Alert Repository

Separado.

## 354. Finding Repository

Separado.

## 355. Signal Store

Puede ser opcional/retention-specific.

## 356. CQRS

Query projections pueden separarse de command models.

## 357. Incident Query

interface AuthenticationIncidentQueryInterface
{
public function search(
AuthenticationIncidentQueryCriteria $criteria
): AuthenticationIncidentPage;
}

## 358. Pagination

Cursor-based para grandes volúmenes.

## 359. Filtering

Por:

- status
- severity
- type
- tenant
- realm
- principal
- date
- con Authorization.

## 360. Indexing

No indexar secrets.

## 361. Distributed Caching

Incident views pueden cachearse.

## 362. Command Authority

Containment nunca depende de cached incident projection.

## 363. Security Version

Cambios críticos pueden incrementar:

- incident_version
- identity_security_epoch
- session_version
- credential_version
- según caso.

## 364. Cache Key

Debe incluir:

- viewer
- tenant
- realm
- incident version
- authority scope

## 365. Admin Cache Leakage

No:

```php
incident:{id}
compartido entre user/admin projections.
```

## 366. FrankenPHP

No almacenar:

```php
static $currentIncident;
static $currentFinding;
static $currentPrincipalRisk;
```

## 367. Request Scope

Todo contexto de incident response será request/fiber scoped.

## 368. Worker Reset

Limpiar:

- current security signal context
- finding context
- incident context
- response plan
- viewer context
- temporary evidence

## 369. Fiber Safety

Dos incidentes concurrentes no deberán compartir:

- principal
- tenant
- severity
- response plan

## 370. Async Worker Safety

Queue workers también deben resetear context entre jobs.

## 371. Long-Running Consumer

Misma regla que FrankenPHP.

## 372. Sensitive Memory

No conservar evidence sensible más tiempo del necesario.

## 373. Failure Taxonomy

AUTH_SECURITY_SIGNAL_INVALID
AUTH_SECURITY_SIGNAL_UNTRUSTED
AUTH_SECURITY_SIGNAL_DUPLICATE

AUTH_SECURITY_FINDING_INVALID
AUTH_SECURITY_FINDING_CONFLICT

AUTH_SECURITY_ALERT_NOT_FOUND
AUTH_SECURITY_ALERT_ALREADY_RESOLVED
AUTH_SECURITY_ALERT_ACCESS_DENIED

AUTH_INCIDENT_NOT_FOUND
AUTH_INCIDENT_ACCESS_DENIED
AUTH_INCIDENT_INVALID_TRANSITION
AUTH_INCIDENT_VERSION_CONFLICT
AUTH_INCIDENT_CORRELATION_FAILED

AUTH_INCIDENT_RESPONSE_FAILED
AUTH_INCIDENT_RESPONSE_PARTIAL
AUTH_INCIDENT_RESPONSE_POLICY_UNAVAILABLE

AUTH_CONTAINMENT_FAILED
AUTH_CONTAINMENT_PARTIAL
AUTH_CONTAINMENT_STATE_CONFLICT

AUTH_IDENTITY_RESTRICTION_FAILED
AUTH_SECURITY_FREEZE_FAILED

AUTH_SECURITY_NOTIFICATION_FAILED
AUTH_SECURITY_NOTIFICATION_CHANNEL_UNAVAILABLE

AUTH_SECURITY_REVIEW_REQUIRED
AUTH_SECURITY_REVIEW_INCOMPLETE

AUTH_COMPROMISE_CONFIRMATION_INVALID
AUTH_COMPROMISE_ALREADY_CONTAINED

## 374. Public Error Normalization

No revelar:
incident exists for another tenant

## 375. Internal Error

Audit puede conservar detalle exacto.

## 376. Security Invariants — Detection

AUTH-INC-DETECT-01
Security Signal no equivale automáticamente a compromiso.

- AUTH-INC-DETECT-02
- Findings deberán incluir provenance.
- AUTH-INC-DETECT-03

Detection rules serán versionadas.

- AUTH-INC-DETECT-04
- Raw credentials nunca forman parte de evidence.
- AUTH-INC-DETECT-05

Critical cryptographic replay podrá producir finding inmediato.
AUTH-INC-DETECT-06
External intelligence no será tratada automáticamente como authoritative.

## 377. Security Invariants — Alerts

AUTH-INC-ALERT-01
Alert != Notification.

- AUTH-INC-ALERT-02
- Acknowledged != Resolved.
- AUTH-INC-ALERT-03
- Alerts soportarán deduplication.
- AUTH-INC-ALERT-04

Critical alerts no dependerán de un único delivery channel.
AUTH-INC-ALERT-05
User-visible alerts nunca expondrán secretos.

## 378. Security Invariants — Incidents

AUTH-INC-01
Incident tendrá lifecycle explícito.

- AUTH-INC-02
- Incident resolution no elimina evidence.
- AUTH-INC-03

Severity changes serán auditable.

- AUTH-INC-04
- Response actions serán idempotentes cuando sea posible.
- AUTH-INC-05

Containment crítico utilizará authoritative stores.
AUTH-INC-06
Closed incident no cambiará silenciosamente de estado.

## 379. Security Invariants — Account Protection

AUTH-PROTECT-01
Restricted != Suspended.

- AUTH-PROTECT-02
- Security Freeze != Account Deletion.
- AUTH-PROTECT-03

Recovery success no implica incident resolved.

- AUTH-PROTECT-04
- Security Review puede ser obligatoria post-recovery.
- AUTH-PROTECT-05

Compromise puede incrementar Security Epoch.
AUTH-PROTECT-06
Account protection state deberá ser explícito.

## 380. Security Invariants — Multi-Tenant

AUTH-INC-TENANT-01
Tenant A no ve incidentes de Tenant B.

- AUTH-INC-TENANT-02
- Tenant admin no obtiene authority global implícita.
- AUTH-INC-TENANT-03

Platform incident puede tener child impacts.
AUTH-INC-TENANT-04
Tenant policy puede endurecer response floor, no debilitarlo.

## 381. Security Invariants — Runtime

AUTH-INC-RT-01
No mutable global incident context.

- AUTH-INC-RT-02
- No cross-request finding leakage.
- AUTH-INC-RT-03

No cross-fiber tenant leakage.

- AUTH-INC-RT-04
- Queue workers limpian security context.
- AUTH-INC-RT-05

Cached projections nunca son containment authority.

## 382. Anti-Pattern

risk score > 80
→ account compromised
sin contexto.

## 383. Anti-Pattern

new IP
→ revoke everything

## 384. Anti-Pattern

user clicked "This was me"
→ erase incident

## 385. Anti-Pattern

password reset succeeded
→ account safe again

## 386. Anti-Pattern

send email failed
→ rollback security containment

## 387. Anti-Pattern

admin can view incident
→ admin can view raw credentials

## 388. Anti-Pattern

500 failed logins
→ send 500 emails

## 389. Anti-Pattern

temporary detection outage
→ allow confirmed replay

## 390. Anti-Pattern

tenant admin
→ disable platform compromise detection

## 391. Anti-Pattern

false positive
→ delete all audit history

## 392. Anti-Pattern

security freeze
→ delete account

## 393. Anti-Pattern

machine identity compromised
→ ask machine for MFA

## 394. Anti-Pattern

IP + User-Agent
→ definitive attacker identity

## 395. Anti-Pattern

security event
→ raw token included for investigation

## 396. Componentes principales

AuthenticationSecuritySignal
AuthenticationSecuritySignalBus
AuthenticationSecuritySignalNormalizer
AuthenticationSecuritySignalProvenance

AuthenticationCompromiseDetectionEngine
AuthenticationCompromiseDetectionRule
AuthenticationSecurityFinding

AuthenticationSecurityAlert
AuthenticationAlertFingerprint

AuthenticationSecurityIncident
AuthenticationIncidentCorrelationEngine
AuthenticationIncidentStateMachine

AuthenticationIncidentResponsePolicy
AuthenticationIncidentResponsePlan
AuthenticationIncidentResponseOrchestrator

AuthenticationContainmentLevel
IdentityProtectionState
AuthenticationSecurityFreeze

AuthenticationSecurityNotification
AuthenticationSecurityNotificationPolicy
AuthenticationSecurityNotificationDispatcher

AuthenticationSecurityReviewRequirement
AuthenticationCompromiseWindow

AuthenticationSecurityEventExporter

## 397. Namespace sugerido

VoltStack\Quantum\Auth\SecurityIncident

## 398. Estructura sugerida

src/Quantum/Auth/SecurityIncident/
├── Contracts/
│   ├── AuthenticationSecuritySignalBusInterface.php
│   ├── AuthenticationSecuritySignalNormalizerInterface.php
│   ├── AuthenticationCompromiseDetectionEngineInterface.php
│   ├── AuthenticationCompromiseDetectionRuleInterface.php
│   ├── AuthenticationIncidentCorrelationEngineInterface.php
│   ├── AuthenticationIncidentResponsePolicyInterface.php
│   ├── AuthenticationIncidentResponseOrchestratorInterface.php
│   ├── AuthenticationSecurityNotificationDispatcherInterface.php
│   ├── AuthenticationSecurityNotificationPolicyInterface.php
│   ├── AuthenticationSecurityEventExporterInterface.php
│   └── AuthenticationIncidentProjectionPolicyInterface.php
│
├── Signal/
│   ├── AuthenticationSecuritySignal.php
│   ├── AuthenticationSecuritySignalType.php
│   ├── AuthenticationSecuritySignalSet.php
│   ├── AuthenticationSecuritySignalProvenance.php
│   └── SecuritySignalSubject.php
│
├── Detection/
│   ├── AuthenticationCompromiseDetectionEngine.php
│   ├── AuthenticationDetectionRuleDefinition.php
│   ├── AuthenticationDetectionRuleRegistry.php
│   └── AuthenticationCompromiseExplanation.php
│
├── Finding/
│   ├── AuthenticationSecurityFinding.php
│   ├── AuthenticationSecurityFindingId.php
│   ├── AuthenticationSecurityFindingType.php
│   ├── AuthenticationSecurityFindingSet.php
│   ├── AuthenticationSecurityConfidence.php
│   └── AuthenticationSecuritySeverity.php
│
├── Alert/
│   ├── AuthenticationSecurityAlert.php
│   ├── AuthenticationSecurityAlertId.php
│   ├── AuthenticationSecurityAlertStatus.php
│   └── AuthenticationAlertFingerprint.php
│
├── Incident/
│   ├── AuthenticationSecurityIncident.php
│   ├── AuthenticationSecurityIncidentId.php
│   ├── AuthenticationIncidentStatus.php
│   ├── AuthenticationIncidentType.php
│   ├── AuthenticationIncidentVersion.php
│   ├── AuthenticationIncidentTimelineEntry.php
│   ├── AuthenticationIncidentRelationship.php
│   └── AuthenticationIncidentBlastRadius.php
│
├── Response/
│   ├── AuthenticationIncidentResponsePlan.php
│   ├── AuthenticationIncidentResponseStep.php
│   ├── AuthenticationIncidentResponseActionType.php
│   ├── AuthenticationIncidentResponseOperationId.php
│   └── AuthenticationContainmentLevel.php
│
├── Protection/
│   ├── IdentityProtectionState.php
│   ├── AuthenticationSecurityFreeze.php
│   ├── AuthenticationSecurityFreezeReason.php
│   └── AuthenticationSecurityCooldown.php
│
├── Review/
│   ├── AuthenticationSecurityReviewRequirement.php
│   ├── AuthenticationCompromiseWindow.php
│   └── AuthenticationSecurityReviewManager.php
│
├── Notification/
│   ├── AuthenticationSecurityNotification.php
│   ├── AuthenticationSecurityNotificationPlan.php
│   ├── AuthenticationSecurityNotificationDispatcher.php
│   └── AuthenticationSecurityNotificationOutbox.php
│
├── Projection/
│   ├── AuthenticationIncidentProjection.php
│   └── AuthenticationIncidentQuery.php
│
├── Export/
│   ├── AuthenticationSecurityExportEvent.php
│   └── AuthenticationSecurityEventExporter.php
│
├── Events/
│   └── ...

```text
│
├── Runtime/
│   ├── AuthenticationSecurityIncidentContext.php
│   └── AuthenticationSecurityIncidentResetter.php
│
└── Exceptions/
    └── ...
```

## 399. Flujo general

AUTHENTICATION
│
▼
Security Signals
│
▼
┌─────────────────┐
│ Detection Engine│
└────────┬────────┘
│
▼
Security Finding
│
┌──────┴──────┐
▼             ▼
Risk Update    Correlation
│
▼
Security Alert
│
▼
Security Incident
│
▼
Response Policy
│
┌────────────────────┼─────────────────────┐
▼                    ▼                     ▼
Notify              Containment          Investigation
│
┌──────────────────┼──────────────────┐
▼                  ▼                  ▼
Revoke Session    Revoke Credential   Security Freeze
│                  │                  │
└──────────────────┼──────────────────┘
▼
Security Epoch Update
│
▼
Restricted Identity
│
▼
Recovery
│
▼
Security Review
│
▼
NORMAL SECURITY STATE

## 400. Ejemplo completo — Account Takeover

12:01
Successful password login
New device
New country

12:02
Recovery email changed

12:03
TOTP removed

12:04
New passkey added

12:05
All previous sessions revoked
El Detection Engine correlaciona:

- NEW_DEVICE
- +
- NEW_COUNTRY
- +
- RECOVERY_CHANGE
- +
- MFA_REMOVAL
- +
- PASSKEY_ENROLLMENT
- +
- SESSION_REVOCATION

Resultado:

```text
Finding:
PROBABLE_ACCOUNT_TAKEOVER
```

Confidence:
VERY_HIGH

Severity:
CRITICAL
Se abre:

- Incident:
- ACCOUNT_TAKEOVER

Response Policy:

1. Freeze security mutations
2. Increment Identity Security Epoch
3. Revoke active sessions
4. Disable recently-added authentication methods
5. Require account recovery
6. Notify trusted security channels
7. Require Security Review

Después:

```text
Recovery
   ↓
Restricted Authentication
   ↓
Review Sessions
   ↓
Review Devices
   ↓
Review Passkeys
   ↓
Review MFA
   ↓
Review Recovery
   ↓
Strong Authentication
   ↓
Incident Resolution
   ↓
Normal State
```

## 8. Ejemplo — Session Theft

Session S1
│
├── Monterrey
│
└── New York 30 seconds later
Esto por sí solo puede ser probabilístico.

```text
Pero posteriormente:
Session S1 revoked
       ↓
```

S1 reused successfully at protocol boundary
Documento 39 genera:
SESSION_REPLAY
Este sistema produce:

```text
Finding:
SESSION_COMPROMISE
```

Confidence:
CONFIRMED

Severity:
CRITICAL
Respuesta:

- revoke session family
- increment session/identity epoch
- require reauthentication
- open security review
- notify user

## 402. Ejemplo — Machine Credential

CI/CD workload
│
└── signing credential K7
Security system detecta:

- credential used
- from unauthorized workload
- +
- invalid expected attestation

Response:

```text
Incident:
MACHINE_IDENTITY_COMPROMISE
```

Actions:

- revoke K7
- increment machine credential version
- disable token issuance
- quarantine workload identity
- notify platform security
- rotate credential

No existe:

- "Enter your MFA code"
- porque se trata de identidad no humana.

## 403. Relación con Laravel

Laravel ofrece primitives que pueden participar:

- Auth Events
- Notifications
- Sessions
- Fortify
- Sanctum
- Password Reset
- Rate Limiting
- Queues
- Events
- Listeners
- Notifications

Una aplicación Laravel puede construir sistemas de protección sobre estas piezas.
VoltStack propone convertir el dominio completo en infraestructura de framework:

- Signals
- +
- Findings
- +
- Detection
- +
- Alerts
- +
- Incident Correlation
- +
- Containment
- +
- Identity Protection State
- +
- Security Review
- +
- Notification Policy
- +
- Distributed Response

## 404. Relación con Symfony

Symfony proporciona piezas sólidas mediante:

- Security
- Authenticators
- Events
- RateLimiter
- Notifier
- Messenger
- Cache
- Lock
- Workflow

que pueden utilizarse para implementar arquitecturas similares.
VoltStack formalizará un modelo Authentication-specific para que incident response no dependa de lógica dispersa entre:

- event subscribers
- controllers
- listeners
- notification handlers
- security voters
- database flags

## 405. Diferenciador VoltStack

Laravel-like Developer Experience
+
Symfony-like Contracts
+
Authentication Security Signal Bus
+
Compromise Detection Engine
+
Versioned Detection Rules
+
Security Findings
+
Alert Correlation
+
Authentication Incident Lifecycle
+
Automated Containment
+
Identity Protection States
+
Security Freeze
+
Security Epoch Invalidation
+
User Compromise Confirmation
+
Post-Recovery Security Review
+
Machine Identity Incident Response
+
Multi-Tenant Incident Isolation
+
Distributed Containment
+
SIEM Integration
+
FrankenPHP/Fiber-Safe Runtime

## 406. Decisiones arquitectónicas definitivas

VoltStack adoptará:

1. Authentication Security continuará después del login.
2. Risk Signal != Security Finding.
3. Finding != Alert.
4. Alert != Notification.
5. Finding != Incident.
6. Incident agrupará findings relacionados.
7. Signals tendrán provenance.
8. Detection rules serán versionadas.
9. Confidence y Severity serán conceptos distintos.
10. Detection podrá ser synchronous o asynchronous.
11. Cryptographically confirmed replay podrá procesarse inline.
12. Behavioral detection podrá ser async.
13. Core no dependerá de Machine Learning.
14. Alerts soportarán deduplication.
15. Alert acknowledgement no resolverá incident.
16. Notifications serán delivery, no security state.
17. Critical notifications podrán ser obligatorias.
18. Notification failure no revertirá security mutation.
19. Notification delivery podrá usar Outbox.
20. Security links usarán documento 39.
21. Incidents tendrán state machine.
22. Severity changes serán auditable.
23. Incident response será policy-driven.
24. Response plans tendrán pasos explícitos.
25. Response actions serán idempotentes cuando sea posible.
26. Containment podrá ser granular.
27. Least-destructive containment será preferido cuando sea seguro.
28. Full containment estará disponible para confirmed compromise.
29. Identity Protection State será separado de Account Status.
30. Restricted Authentication será first-class.
31. Security Freeze será first-class.
32. Recovery no resolverá automáticamente incident.
33. Post-Recovery Security Review podrá ser obligatoria.
34. Security Cooldown podrá existir.
35. "This wasn't me" será formal security signal.
36. "This was me" no eliminará evidence.
37. Confirmation strength dependerá del Authentication Context.
38. Self-Lock será first-class.
39. Self-Lock tendrá anti-DoS protections.
40. Credential compromise podrá aislarse sin bloquear toda identity.
41. Compromise Window será first-class.
42. Artifacts creados durante compromise podrán revisarse retroactivamente.
43. Machine identities utilizarán response específico.
44. Machine incidents no usarán human MFA semantics.
45. Provider-wide incidents serán soportados.
46. Infrastructure Authentication incidents serán soportados.
47. Blast Radius será explícito.
48. Cryptographic key purpose separation limitará blast radius.
49. Multi-tenant incident isolation será obligatoria.
50. Platform incidents podrán tener tenant impacts.
51. Tenant policy podrá endurecer response.
52. Tenant policy no podrá debilitar platform floor.
53. Incident projections serán viewer-aware.
54. Admin visibility no implica secret visibility.
55. SIEM integration será first-class.
56. Security events serán structured/versioned/redacted.
57. Metrics evitarán high-cardinality identity labels.
58. Detection/containment latency será observable.
59. Detection rules soportarán shadow/canary.
60. Mandatory security invariants no podrán desactivarse mediante shadow mode.
61. Security drills serán soportados.
62. Break-glass será altamente observable.
63. Impersonation conservará Actor y Subject.
64. Critical response utilizará authoritative stores.
65. Cached projections nunca serán containment authority.
66. Distributed nodes recibirán revocation/security epochs.
67. FrankenPHP no retendrá incident context.
68. Fiber isolation será obligatoria.
69. Queue workers limpiarán security context.
70. Incident evidence nunca contendrá raw Authentication secrets.
71. Criterios de aceptación

El sistema estará completo cuando existan, como mínimo:
72. Security Signal model.
73. Signal provenance.
74. Signal normalization.
75. Signal bus.
76. Signal deduplication.
77. Compromise Detection Engine.
78. Detection Rule contract.
79. Detection Rule Registry.
80. Detection Rule versioning.
81. Shadow detection.
82. Canary detection.
83. Security Findings.
84. Finding confidence.
85. Finding severity.
86. Finding evidence.
87. Finding correlation.
88. Security Alerts.
89. Alert lifecycle.
90. Alert fingerprint.
91. Alert deduplication.
92. Alert aggregation.
93. Notification model.
94. Notification policies.
95. Mandatory notification support.
96. Multi-channel delivery.
97. Notification Outbox.
98. Notification retry.
99. Secure notification links.
100. Authentication Incidents.
101. Incident lifecycle.
102. Incident state machine.
103. Incident correlation.
104. Incident severity escalation.
105. Incident timeline.
106. Incident relationships.
107. Parent platform incidents.
108. Blast Radius.
109. Response Policy.
110. Response Plan.
111. Response Orchestrator.
112. Response dependencies.
113. Partial failure handling.
114. Idempotent response actions.
115. Containment levels.
116. Session containment.
117. Credential containment.
118. Device containment.
119. Authentication method containment.
120. Machine identity containment.
121. Identity Protection State.
122. Restricted Authentication.
123. Security Freeze.
124. Self-Lock.
125. Security Epoch integration.
126. User activity confirmation.
127. "This wasn't me" workflow.
128. "This was me" workflow.
129. Compromise Window.
130. Retroactive artifact review.
131. Security Review.
132. Post-Recovery Review.
133. Security Cooldown.
134. Multi-tenant isolation.
135. Realm isolation.
136. Platform incidents.
137. Machine incidents.
138. Provider compromise handling.
139. Authentication infrastructure incidents.
140. Audit.
141. Explainability.
142. SIEM export.
143. OpenTelemetry tracing.
144. Security metrics.
145. MTTD metrics.
146. MTTC metrics.
147. MTTR metrics.
148. Distributed response.
149. Multi-region propagation.
150. Authoritative revocation.
151. FrankenPHP isolation.
152. Fiber isolation.
153. Queue worker isolation.
154. Secret redaction.
155. Privacy/retention policy.
156. False-positive handling.
157. Incident drills.
158. Break-glass monitoring.
159. Impersonation monitoring.
160. Optimistic concurrency.
161. Incident versioning.
162. Arquitectura final del subsistema

┌───────────────────────────────────────────────────────────────┐
│                AUTHENTICATION SECURITY                        │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│ Authentication Events                                         │
│ Risk Signals                                                  │
│ Session Events                                                │
│ Device Events                                                 │
│ Credential Events                                             │
│ Transaction Security Events                                   │
│ External Threat Intelligence                                  │
│ User Reports                                                  │
│        │                                                      │
│        ▼                                                      │
│ ┌─────────────────────────┐                                   │
│ │ Security Signal Layer   │                                   │
│ └────────────┬────────────┘                                   │
│              ▼                                                │
│ ┌─────────────────────────┐                                   │
│ │ Compromise Detection    │                                   │
│ └────────────┬────────────┘                                   │
│              ▼                                                │
│       Security Findings                                       │
│              │                                                │
│              ▼                                                │
│ ┌─────────────────────────┐                                   │
│ │ Correlation Engine      │                                   │
│ └────────────┬────────────┘                                   │
│              ▼                                                │
│      Alerts / Incidents                                       │
│              │                                                │
│              ▼                                                │
│ ┌─────────────────────────┐                                   │
│ │ Response Policy         │                                   │
│ └────────────┬────────────┘                                   │
│              ▼                                                │
│ ┌─────────────────────────┐                                   │
│ │ Response Orchestrator   │                                   │
│ └────────────┬────────────┘                                   │
│              │                                                │
│     ┌────────┼────────┬─────────────┐                          │
│     ▼        ▼        ▼             ▼                          │
│  Notify   Revoke   Restrict       Freeze                       │
│     │        │        │             │                          │
│     └────────┴────────┴──────┬──────┘                          │
│                              ▼                                 │
│                    Account Protection                          │
│                              │                                 │
│                              ▼                                 │
│                         Recovery                               │
│                              │                                 │
│                              ▼                                 │
│                      Security Review                           │
│                              │                                 │
│                              ▼                                 │
│                      Normal Security State                     │
│                                                               │
└───────────────────────────────────────────────────────────────┘

## 163. Posición de los documentos 36–40

Con este documento aparece una secuencia arquitectónica muy sólida:

```text
36 AUTHENTICATION POLICY ENGINE
        │
        │ ¿Qué seguridad exige VoltStack?
        ▼
37 AUTHENTICATION ASSURANCE
        │
        │ ¿Qué nivel de confianza tenemos?
        ▼
38 AUTHENTICATION CHALLENGE
        │
        │ ¿Cómo obtenemos evidencia adicional?
        ▼
39 AUTHENTICATION TRANSACTION SECURITY
        │
        │ ¿La transacción es íntegra,
        │ fresca, ligada y no reutilizada?
        ▼
40 AUTHENTICATION SECURITY INCIDENT
        │
        │ ¿Qué hacemos cuando detectamos
        │ actividad posiblemente comprometida?
        ▼
   CONTAINMENT / RECOVERY
```

Esto separa cinco problemas que no deberían terminar concentrados dentro de un único AuthManager.

## 164. Regla arquitectónica principal

La regla fundamental será:
VoltStack nunca asumirá que una identidad continúa siendo segura únicamente porque alguna vez se autenticó correctamente.

El estado de seguridad será dinámico:

```text
Authenticated
     │
     ▼
Observed
     │
     ▼
Risk Evaluated
     │
     ▼
Potentially Suspicious
     │
     ▼
Finding
     │
     ▼
Incident
     │
     ▼
Containment
     │
     ▼
Recovery
     │
     ▼
Security Review
     │
     ▼
Trusted Again
```

Y especialmente:

```text
Authentication Success
        ≠
Permanent Trust
```

## 411. Siguiente documento recomendado

El siguiente sistema que conviene formalizar es:
`41_AUTHENTICATION_IDENTITY_LIFECYCLE_ACCOUNT_STATE_SUSPENSION_LOCKOUT_DEACTIVATION_DELETION_AND_REACTIVATION_SYSTEM.md`
El motivo es que ya hemos utilizado conceptos como:

- ACTIVE
- DISABLED
- SUSPENDED
- LOCKED
- RESTRICTED
- FROZEN
- COMPROMISED
- RECOVERY_REQUIRED
- DEACTIVATED
- DELETED

pero todavía debemos establecer formalmente cuáles pertenecen a la identidad, cuáles a Authentication Security y cuáles son temporales.
El 41 debería definir:

```text
                 IDENTITY
                    │
                    ▼
               PROVISIONED
                    │
                    ▼
                 ACTIVE
              ┌─────┼─────┐
              ▼     ▼     ▼
           LOCKED SUSPENDED DEACTIVATED
              │     │          │
              ▼     ▼          ▼
           ACTIVE ACTIVE    REACTIVATED
                               │
                               ▼
                             ACTIVE
                               │
                               ▼
                        DELETION_PENDING
                               │
                               ▼
                            DELETED
```

manteniendo separado:

```text
Identity Lifecycle State
        ≠
Authentication Security Posture
        ≠
Session State
        ≠
Credential State
        ≠
Authorization State
```

Esa separación será especialmente importante para multi-tenancy, offboarding, suspensión administrativa, cierre de cuentas, retención, reactivación y FrankenPHP/distributed authentication.
