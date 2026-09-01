# VoltStack Authentication System

## 41 — Authentication Identity Lifecycle, Account State, Suspension, Lockout, Deactivation, Deletion and Reactivation System

- **Archivo:** `41_AUTHENTICATION_IDENTITY_LIFECYCLE_ACCOUNT_STATE_SUSPENSION_LOCKOUT_DEACTIVATION_DELETION_AND_REACTIVATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Identity Lifecycle / Account State Governance
- **Dependencias principales:** 03, 09, 10, 12, 13, 18, 20, 21, 23, 24, 25, 29, 30, 32, 33, 34, 35, 36, 37, 38, 39, 40.

---

## 1. Propósito

Este documento define el sistema formal responsable del ciclo de vida de las identidades autenticables dentro de VoltStack.
Su responsabilidad principal será responder:
¿Existe esta identidad?

¿Está activa?

¿Puede autenticarse?

¿Está bloqueada temporalmente?

¿Está suspendida administrativamente?

¿Fue desactivada?

¿Está pendiente de eliminación?

¿Fue eliminada?

¿Puede ser reactivada?

¿Qué ocurre con sus sesiones?

¿Qué ocurre con sus credenciales?

¿Qué ocurre con sus dispositivos?

¿Qué ocurre con sus identidades federadas?

¿Qué ocurre dentro de cada tenant?

¿Qué ocurre con sus datos después de eliminarla?
El sistema deberá evitar que estas respuestas terminen distribuidas entre:
users.active
users.deleted_at
users.blocked
sessions
password resets
tenant memberships
middleware
controllers
authorization policies
security incident flags
sin un modelo arquitectónico coherente.

## 2. Problema fundamental

Un sistema de Authentication no puede modelar una cuenta únicamente mediante:
$user->active === true;
En sistemas reales una identidad puede encontrarse en situaciones como:
ACTIVE

LOCKED

SUSPENDED

DEACTIVATED

DELETION_PENDING

DELETED

REACTIVATION_PENDING
mientras simultáneamente puede tener:
Security Posture = COMPROMISED

Tenant Membership = SUSPENDED

Session = REVOKED

Password = ACTIVE

Passkey = ACTIVE

Device = LOST
Estos estados pertenecen a dominios diferentes.

## 3. Principio fundamental

La existencia de una identidad, su capacidad para autenticarse, su estado de seguridad, su pertenencia a tenants y sus permisos son dimensiones diferentes.

Por tanto:
Identity Lifecycle State
        ≠
Authentication Security Posture
        ≠
Authentication Eligibility
        ≠
Credential State
        ≠
Session State
        ≠
Tenant Membership State
        ≠
Authorization State

## 4. Arquitectura conceptual

                        Identity
                           │
             ┌─────────────┼───────────────┐
             │             │               │
             ▼             ▼               ▼
       Lifecycle State  Security State  Memberships
             │             │               │
             │             │               ▼
             │             │          Tenant State
             │             │
             ▼             ▼
     Authentication     Security
       Eligibility      Posture
             │
             ▼
       Authentication
             │
             ▼
       Authorization

## 5. Objetivos

El sistema deberá proporcionar:

1. Identity lifecycle explícito.
2. Account states tipados.
3. Authentication eligibility.
4. Temporary lockout.
5. Administrative suspension.

## 6. Security suspension.

## 7. Voluntary deactivation.

## 8. Administrative deactivation.

## 9. Deletion requests.

## 10. Delayed deletion.

## 11. Deletion cancellation.

## 12. Logical deletion.

## 13. Physical erasure orchestration.

## 14. Anonymization.

## 15. Retention holds.

## 16. Reactivation.

## 17. Session invalidation.

## 18. Credential lifecycle integration.

## 19. Device lifecycle integration.

## 20. Federation lifecycle integration.

## 21. Recovery lifecycle integration.

## 22. Multi-tenant lifecycle.

## 23. Machine identity lifecycle.

## 24. Distributed state propagation.

## 25. Auditability.

## 26. Explainability.

## 27. Extensibility.

## 28. FrankenPHP-safe runtime.

## 29. Lo que este sistema no es

No será responsable directamente de:
resource permissions
roles
RBAC
ABAC
business subscription state
billing suspension
employee HR state
data retention engine completo
generic workflow engine
Podrá consumir estos estados mediante policies/adapters.

## 30. Identity

La identidad lógica deberá permanecer separada de las credenciales.
final readonly class Identity
{
    public function __construct(
        public IdentityId $id,
        public IdentityType $type,
        public IdentityLifecycleState $lifecycleState,
        public IdentityVersion $version,
        public DateTimeImmutable $createdAt,
    ) {}
}

## 31. Identity != User Record

Una implementación puede persistirla en:
users
identities
accounts
principals
pero el dominio no deberá depender del nombre de tabla.

## 32. Identity Types

enum IdentityType: string
{
    case Human = 'human';
    case Service = 'service';
    case Workload = 'workload';
    case Application = 'application';
    case Device = 'device';
    case Automation = 'automation';
}

## 33. Human y Machine Lifecycles

Podrán compartir infraestructura de lifecycle, pero no necesariamente las mismas transiciones.
Ejemplo:
Human:
ACTIVE → DEACTIVATED

Machine:
ACTIVE → RETIRED
11. Identity Lifecycle State
Modelo inicial:
enum IdentityLifecycleState: string
{
    case Provisioning = 'provisioning';
    case PendingActivation = 'pending_activation';
    case Active = 'active';
    case Locked = 'locked';
    case Suspended = 'suspended';
    case Deactivated = 'deactivated';
    case DeletionPending = 'deletion_pending';
    case Deleted = 'deleted';
    case ReactivationPending = 'reactivation_pending';
    case Retired = 'retired';
}
12. State Machine principal
                  PROVISIONING
                       │
                       ▼
              PENDING_ACTIVATION
                       │
                       ▼
                     ACTIVE
             ┌─────────┼───────────┐
             │         │           │
             ▼         ▼           ▼
          LOCKED   SUSPENDED   DEACTIVATED
             │         │           │
             └────┬────┘           │
                  ▼                │
                ACTIVE             │
                                   ▼
                         REACTIVATION_PENDING
                                   │
                                   ▼
                                 ACTIVE

ACTIVE / SUSPENDED / DEACTIVATED
               │
               ▼
        DELETION_PENDING
               │
        ┌──────┴──────┐
        ▼             ▼
      ACTIVE        DELETED
No todas las transiciones estarán disponibles para todos los Identity Types.
13. Provisioning
PROVISIONING representa una identidad cuya creación todavía no terminó.
Ejemplos:
enterprise provisioning
SCIM import
invitation workflow
service account creation
external identity bootstrap
14. Provisioning no implica Authentication
Por defecto:
PROVISIONING
→ authentication denied
15. Pending Activation
Representa una identidad creada pero que todavía debe completar una ceremonia.
Ejemplos:
verify email
accept invitation
establish first credential
complete enterprise onboarding
administrator approval
16. Pending Activation != Suspended
Una identidad pendiente nunca estuvo necesariamente activa.
17. Active
ACTIVE significa:
El lifecycle de la identidad permite potencialmente Authentication.

No significa:
authentication automatically succeeds
18. Active != Eligible
Una identidad ACTIVE todavía puede ser Authentication-ineligible debido a:
security freeze
compromise
tenant restrictions
credential state
realm policy
risk
authentication policy
19. Authentication Eligibility
Por ello se introduce:
interface AuthenticationEligibilityResolverInterface
{
    public function resolve(
        IdentityReference $identity,
        AuthenticationEligibilityContext $context
    ): AuthenticationEligibilityDecision;
}
20. Eligibility Decision
enum AuthenticationEligibilityDecisionType: string
{
    case Eligible = 'eligible';
    case ActivationRequired = 'activation_required';
    case Locked = 'locked';
    case Suspended = 'suspended';
    case Deactivated = 'deactivated';
    case DeletionPending = 'deletion_pending';
    case Deleted = 'deleted';
    case SecurityRestricted = 'security_restricted';
    case RecoveryRequired = 'recovery_required';
    case PolicyDenied = 'policy_denied';
}
21. Lifecycle State != Eligibility Decision
Una decisión puede depender de más dimensiones.
Ejemplo:
Lifecycle = ACTIVE
Security Posture = COMPROMISED
Protection State = RECOVERY_REQUIRED

↓

Eligibility = RECOVERY_REQUIRED
22. Locked
LOCKED será un bloqueo temporal de Authentication.
23. Lockout
Lockout podrá producirse por:
too many authentication failures
temporary administrative protection
security incident
user self-lock
automated defense
24. Lockout Model
final readonly class IdentityLockout
{
    public function __construct(
        public IdentityLockoutId $id,
        public IdentityReference $identity,
        public IdentityLockoutReason $reason,
        public DateTimeImmutable $startedAt,
        public DateTimeImmutable|null $expiresAt,
        public ActorReference|null $actor,
    ) {}
}
25. Lockout Reason
enum IdentityLockoutReason: string
{
    case AuthenticationFailures = 'authentication_failures';
    case SecurityIncident = 'security_incident';
    case UserRequested = 'user_requested';
    case AdministratorRequested = 'administrator_requested';
    case AutomatedProtection = 'automated_protection';
}
26. Temporary Lockout
Normalmente:
LOCKED
   │
   │ expires
   ▼
ACTIVE
27. Lockout Expiration
No deberá requerir necesariamente un cron para desbloquear físicamente la fila.
Puede calcularse:
lockedUntil <= now
y resolver estado efectivo.
28. Lockout Anti-DoS
Regla crítica:
Un atacante no debe poder causar fácilmente un bloqueo permanente de otra identidad mediante intentos fallidos.

 1. Progressive Authentication Defense
Preferir:
rate limiting
delay
challenge escalation
risk increase
temporary lock
network throttling
antes que bloqueo permanente.
 2. Account Enumeration
Public Authentication responses no deberán revelar claramente:
account exists but is locked
si ello facilita enumeración.
 3. Internal Reason
Internamente sí deberá conservarse:
AUTH_IDENTITY_LOCKED
 4. Lockout Scope
Un bloqueo puede aplicarse a:
identity
authentication method
realm
tenant
network interaction
 5. Method Lockout
Ejemplo:
password temporarily locked
passkey still usable
No debe requerir bloquear toda la identidad.
 6. Identity Lock vs Method Lock
Identity Lock
    ≠
Credential Lock
 7. Realm Lock
Ejemplo:
Admin Realm locked
User Realm still available
si policy lo permite.
 8. Suspension
SUSPENDED representa una decisión explícita que impide o restringe Authentication por una causa administrativa, contractual, security-related o governance-related.
 9. Suspension != Lockout
LOCKOUT
→ normalmente temporal/automático

SUSPENSION
→ normalmente explícita/policy-driven

## 38. Suspension Model

final readonly class IdentitySuspension
{
    public function __construct(
        public IdentitySuspensionId $id,
        public IdentityReference $identity,
        public IdentitySuspensionReason $reason,
        public DateTimeImmutable $startedAt,
        public DateTimeImmutable|null $expiresAt,
        public ActorReference $actor,
        public string|null $reference,
    ) {}
}

## 39. Suspension Reasons

enum IdentitySuspensionReason: string
{
    case Security = 'security';
    case Administrative = 'administrative';
    case Compliance = 'compliance';
    case PolicyViolation = 'policy_violation';
    case OrganizationRequest = 'organization_request';
    case Legal = 'legal';
    case Custom = 'custom';
}

## 40. Business Suspension

VoltStack Authentication no debería conocer directamente conceptos como:
subscription unpaid
employee terminated
customer delinquent
La aplicación puede traducirlos a una lifecycle action mediante policy.

## 41. Suspension Authority

Toda suspensión deberá registrar quién la originó:
SYSTEM
USER
TENANT_ADMIN
PLATFORM_ADMIN
SECURITY_OPERATOR
EXTERNAL_PROVISIONER

## 42. Suspension Scope

Podrá ser:
enum IdentityLifecycleScope: string
{
    case Global = 'global';
    case Tenant = 'tenant';
    case Realm = 'realm';
    case Application = 'application';
}

## 43. Global Suspension

Global Identity
      │
      ▼
SUSPENDED
bloquea Authentication global conforme policy.

## 44. Tenant Suspension

No necesariamente modifica lifecycle global.
Ejemplo:
Identity = ACTIVE

Tenant A Membership = ACTIVE
Tenant B Membership = SUSPENDED

## 45. Multi-Tenant Separation

Esto es esencial.
No deberá hacerse:
Tenant B suspends user
        ↓
users.active = false
        ↓
user loses Tenant A
salvo que la acción tenga authority global explícita.

## 46. Tenant Membership Lifecycle

Debe existir modelo separado:
enum TenantIdentityMembershipState: string
{
    case Invited = 'invited';
    case Active = 'active';
    case Suspended = 'suspended';
    case Deactivated = 'deactivated';
    case Removed = 'removed';
}

## 47. Identity State vs Membership State

Identity ACTIVE
   │
   ├── Tenant A ACTIVE
   ├── Tenant B SUSPENDED
   └── Tenant C REMOVED

## 48. Authentication Flow Multi-Tenant

Resolve Identity
      ↓
Check Global Lifecycle
      ↓
Check Realm Lifecycle
      ↓
Check Tenant Membership
      ↓
Check Security State
      ↓
Check Authentication Policy
      ↓
Authenticate

## 49. Suspension and Existing Sessions

Policy deberá decidir qué ocurre.

## 50. Default

Para suspensión de seguridad/global:
SUSPEND
   ↓
increment security epoch
   ↓
revoke active sessions
   ↓
disable persistent login
será comportamiento recomendado.

## 51. Administrative Suspension

Puede configurablemente:
revoke immediately
o:
prevent renewal/new authentication
dependiendo del dominio.

## 52. Session Must Revalidate State

Una sesión válida criptográficamente no debe ignorar suspensión posterior.

## 53. Distributed State

Cluster:
Node A
User session active

Node B
Admin suspends identity

Node C
Next request
Node C deberá detectar suspensión.

## 54. Identity Security Epoch

Puede incrementarse para invalidar artefactos existentes.

## 55. Lifecycle Version

Además se introduce:
final readonly class IdentityLifecycleVersion
{
    public function __construct(
        public int $value
    ) {}
}

## 56. Purpose

Permite:
cache invalidation
optimistic concurrency
distributed propagation
audit correlation

## 57. Lifecycle Epoch vs Security Epoch

No necesariamente son iguales.
Lifecycle Version
→ lifecycle mutation ordering

Security Epoch
→ invalidate authentication artifacts
Una transición puede actualizar ambos.

## 58. Deactivation

DEACTIVATED representa una identidad conservada pero voluntaria o administrativamente retirada del uso normal.

## 59. Deactivation != Suspension

SUSPENSION
→ enforcement

DEACTIVATION
→ lifecycle choice/state

## 60. Examples

user closes account temporarily
employee account archived
unused service identity disabled
application account deactivated

## 61. Deactivation Sources

enum IdentityDeactivationReason: string
{
    case UserRequested = 'user_requested';
    case AdministratorRequested = 'administrator_requested';
    case Inactivity = 'inactivity';
    case OrganizationOffboarding = 'organization_offboarding';
    case Migration = 'migration';
    case Custom = 'custom';
}

## 62. Deactivation Consequences

Default:
prevent new authentication
revoke sessions
revoke remember-me
preserve credentials securely
preserve audit history

## 63. Credential Preservation

Deactivation no implica necesariamente eliminar:
password
passkeys
federated links
porque puede existir reactivation.

## 64. Credential Policy

Puede elegir:
PRESERVE
SUSPEND
REVOKE
ROTATE_ON_REACTIVATION

## 65. Deactivation and API Credentials

Human account deactivation debería normalmente revocar o suspend:
personal access tokens
application passwords
delegated credentials

## 66. Deactivation and Machine Credentials

Depende de Machine Identity lifecycle.

## 67. User-Initiated Deactivation

Debe requerir Authentication adecuada.

## 68. Sensitive Operation

Desactivar cuenta será operación sensible del documento 32.
Puede requerir:
fresh authentication
step-up
explicit confirmation

## 69. Deactivation Confirmation

No depender únicamente de:
POST /account/deactivate
sin ceremonia.

## 70. Deactivation Intent

Documento 39 puede bindear:
identity
session
operation
tenant
nonce
payload

## 71. Deactivation Cooldown

Puede existir:
DEACTIVATED
    │
    │ 30 days
    ▼
DELETION_PENDING
según aplicación.

## 72. Deactivation != Deletion Request

No deberán ser sinónimos.

## 73. Deletion Request

Una identidad puede solicitar eliminación.
final readonly class IdentityDeletionRequest
{
    public function __construct(
        public IdentityDeletionRequestId $id,
        public IdentityReference $identity,
        public ActorReference $requestedBy,
        public DateTimeImmutable $requestedAt,
        public DateTimeImmutable|null $scheduledFor,
        public IdentityDeletionReason $reason,
    ) {}
}

## 74. Deletion Lifecycle

ACTIVE
   │
   ▼
DELETION REQUESTED
   │
   ▼
DELETION_PENDING
   │
   ├───────────────┐
   │               │
   ▼               ▼
CANCELLED        EXECUTED
   │               │
   ▼               ▼
ACTIVE           DELETED

## 75. Deletion Grace Period

VoltStack deberá soportar:
7 days
30 days
90 days
custom
sin imponer duración universal.

## 76. Why Grace Period

Permite:
accidental deletion recovery
fraud investigation
dependency cleanup
legal checks
subscription cleanup
tenant ownership transfer

## 77. Deletion Pending Authentication

Policy decide si puede autenticarse.
Default recomendado:
normal application access = denied

account restoration/security center = allowed

## 78. Restricted Deletion Session

Puede existir una Authentication Context restringida exclusivamente a:
cancel deletion
download allowed data
security review
logout
support

## 79. Cancellation

Cancelar eliminación será también security-sensitive.

## 80. Deletion Cancellation Attack

Un attacker no debe poder mantener indefinidamente una cuenta que el propietario intenta eliminar.

## 81. Cancellation Requirement

Puede requerir:
strong authentication
recovery authority
fresh passkey
dependiendo de security state.

## 82. Deletion Preconditions

Antes de eliminar puede requerirse verificar:
legal hold
tenant ownership
pending financial records
regulatory retention
active security incident
administrative dependencies
machine ownership

## 83. Deletion Policy

interface IdentityDeletionPolicyInterface
{
    public function evaluate(
        IdentityReference $identity,
        IdentityDeletionContext $context
    ): IdentityDeletionDecision;
}

## 84. Deletion Decision

enum IdentityDeletionDecisionType: string
{
    case Allowed = 'allowed';
    case Delayed = 'delayed';
    case BlockedByRetention = 'blocked_by_retention';
    case BlockedByLegalHold = 'blocked_by_legal_hold';
    case OwnershipTransferRequired = 'ownership_transfer_required';
    case SecurityReviewRequired = 'security_review_required';
    case Denied = 'denied';
}

## 85. Legal Hold

Debe ser first-class.
final readonly class IdentityRetentionHold
{
    public function __construct(
        public IdentityRetentionHoldId $id,
        public IdentityReference $identity,
        public IdentityRetentionHoldType $type,
        public DateTimeImmutable $createdAt,
        public DateTimeImmutable|null $expiresAt,
    ) {}
}

## 86. Retention Hold != Active Account

Una cuenta puede estar:
DELETED
mientras ciertos registros legalmente necesarios permanecen retenidos.

## 87. Logical Deletion

La eliminación lógica de Authentication deberá significar:
identity no longer authenticatable
no necesariamente:
every database row physically erased immediately

## 88. Physical Erasure

Debe ser una fase separada.

## 89. Deletion Architecture

Deletion Approved
      ↓
Disable Authentication
      ↓
Revoke Sessions
      ↓
Revoke Credentials
      ↓
Mark Identity Deleted
      ↓
Emit Deletion Event
      ↓
Data Lifecycle Orchestrator
      ↓
Anonymize / Erase / Retain

## 90. Authentication Responsibility

Auth es responsable de:
authentication identity
authentication credentials
authentication sessions
authentication devices
authentication security metadata
Otros módulos son responsables de sus propios datos.

## 91. Cross-Domain Deletion

Ejemplo:
Orders
Invoices
Messages
Files
Audit
Analytics
no deben ser eliminados directamente por AuthManager.

## 92. Deletion Event

IdentityDeletionApproved
IdentityAuthenticationDisabled
IdentityDeleted
IdentityErasureRequested
permitirán coordinación.

## 93. Saga / Orchestration

En sistemas complejos:
Identity Deletion
      │
      ├── Auth
      ├── Billing
      ├── Storage
      ├── CRM
      ├── Analytics
      └── Tenant
puede utilizar orchestration/saga.

## 94. Deletion Must Be Idempotent

Ejecutar:
delete identity X
dos veces no deberá corromper estado.

## 95. Deleted

DELETED representa:
La identidad ya no participa en Authentication normal.

1. Deleted Identity Login
Siempre:
Authentication denied
salvo mecanismos especiales de restoration expresamente soportados.
2. Deleted Identity Credentials
No deben seguir siendo utilizables.
3. Credential Revocation
Deletion deberá provocar:
password disabled
passkeys revoked
MFA revoked
remember-me revoked
API tokens revoked
device credentials revoked
federated authentication links disabled
machine delegations revoked
según identity type.
4. Session Revocation
Todas las sesiones deberán invalidarse.
5. Pending Transactions
También:
login transactions
reauth transactions
recovery transactions
linking transactions
OAuth states
continuations
deben invalidarse.
6. Security Epoch
Deletion deberá incrementar/invalidate security epoch.
7. Reactivation After Deleted
Por defecto:
DELETED
→ ACTIVE
no será transición normal.
8. Why
Una identidad eliminada puede haber sufrido:
credential destruction
PII anonymization
external cleanup
retention actions
por lo que "undelete" puede ser imposible o inseguro.
9. Restore vs Reactivate
Separar:
Reactivate
→ DEACTIVATED identity

Restore
→ DELETION_PENDING identity

Recover Deleted Identity
→ exceptional policy-specific process

## 105. Reactivation

Normalmente:
DEACTIVATED
        ↓
REACTIVATION_PENDING
        ↓
Identity Verification
        ↓
Credential Review
        ↓
Policy Validation
        ↓
ACTIVE

## 106. Reactivation Is Not a Boolean

No:
$user->active = true;
como única operación.

## 107. Reactivation Ceremony

Puede requerir:
verify identity
verify contact method
fresh credential
password rotation
passkey verification
MFA re-enrollment
admin approval
tenant approval
security review

## 108. Reactivation Requirement

final readonly class IdentityReactivationRequirement
{
    public function __construct(
        public AuthenticationRequirement $authentication,
        public bool $requireSecurityReview,
        public bool $requireCredentialReview,
        public bool $requireAdministrativeApproval,
    ) {}
}

## 109. Reactivation Policy

interface IdentityReactivationPolicyInterface
{
    public function requirements(
        IdentityReference $identity,
        IdentityReactivationContext $context
    ): IdentityReactivationRequirement;
}

## 110. Reactivation Pending

Durante esta fase no debe existir acceso normal.

## 111. Restricted Context

Puede permitirse únicamente:
complete verification
configure credential
review security
cancel reactivation

## 112. Credential Reactivation

Credenciales antiguas no deben reactivarse automáticamente siempre.

## 113. Credential Age

Ejemplo:
account deactivated 4 years
puede requerir:
password reset
new passkey
MFA re-enrollment

## 114. External Identity Reactivation

Debe revalidar:
issuer
subject
provider status
tenant policy
federation configuration

## 115. Provider Reuse Risk

No asumir que email sigue representando misma persona.

## 116. Reactivation and Recovery

Si no existe credential viable:
Reactivation
    ↓
Recovery
puede ser necesario.

## 117. Reactivation and Security Incident

Si la identidad fue desactivada por compromise:
security incident resolution
+
security review
+
strong authentication
pueden ser requisitos.

## 118. Security Posture

Documento 35/40.
Ejemplo:
Lifecycle = DEACTIVATED
Security Posture = COMPROMISED
Al solicitar reactivation:
REACTIVATION_PENDING
+
RECOVERY_REQUIRED

## 119. Lifecycle Transition

Toda transición deberá ser explícita.

## 120. Transition Model

final readonly class IdentityLifecycleTransition
{
    public function __construct(
        public IdentityLifecycleState $from,
        public IdentityLifecycleState $to,
        public IdentityLifecycleTransitionReason $reason,
        public ActorReference $actor,
        public DateTimeImmutable $occurredAt,
    ) {}
}

## 121. State Machine Contract

interface IdentityLifecycleStateMachineInterface
{
    public function transition(
        Identity $identity,
        IdentityLifecycleTransitionRequest $request
    ): IdentityLifecycleTransitionResult;
}

## 122. Transition Guard

interface IdentityLifecycleTransitionGuardInterface
{
    public function evaluate(
        Identity $identity,
        IdentityLifecycleTransitionRequest $request
    ): IdentityLifecycleTransitionGuardResult;
}

## 123. Transition Guards

Pueden verificar:
Authentication Requirement
Authorization
Security Posture
Tenant Scope
Legal Hold
Ownership
Incident State
Credential Viability
Policy

## 124. Invalid Transition

Ejemplo:
DELETED
→ LOCKED
debe rechazarse.

## 125. Valid Transition Matrix

Ejemplo conceptual:
From To Normal
Provisioning PendingActivation ✓
PendingActivation Active ✓
Active Locked ✓
Locked Active ✓
Active Suspended ✓
Suspended Active ✓
Active Deactivated ✓
Deactivated ReactivationPending ✓
ReactivationPending Active ✓
Active DeletionPending ✓
Deactivated DeletionPending ✓
DeletionPending Active Policy
DeletionPending Deleted ✓
Deleted Active ✗
Retired Active ✗ default

  1. Transition Side Effects
No deberán estar escondidos arbitrariamente dentro del Entity.
Ejemplo:
ACTIVE
→ SUSPENDED
puede necesitar:
revoke sessions
increment epoch
emit event
audit
notify
  2. Lifecycle Orchestrator
interface IdentityLifecycleOrchestratorInterface
{
    public function execute(
        IdentityLifecycleCommand $command
    ): IdentityLifecycleResult;
}
  3. Orchestration Pipeline
Lifecycle Command
       ↓
Resolve Identity
       ↓
Resolve Actor
       ↓
Authorization
       ↓
Authentication Requirement
       ↓
Validate Current State
       ↓
Evaluate Transition Guards
       ↓
Resolve Side-Effect Plan
       ↓
Atomic Lifecycle Mutation
       ↓
Security Epoch / Versions
       ↓
Outbox
       ↓
Audit
       ↓
Async Side Effects
  4. Authorization
Authentication lifecycle management does not replace Authorization.
Ejemplo:
Admin authenticated strongly
no implica:
may suspend any identity
  5. Authentication Requirement
Acciones como:
deactivate own account
cancel deletion
reactivate
delete
suspend administrator
pueden requerir documento 32/36.
  6. Self-Service vs Administrative
enum IdentityLifecycleOperationMode: string
{
    case SelfService = 'self_service';
    case Administrative = 'administrative';
    case SecurityOperations = 'security_operations';
    case Automated = 'automated';
    case Provisioning = 'provisioning';
}
  7. Actor vs Target
Siempre:
Actor
≠
Target Identity
como posibilidad.
  8. Self-Service
Actor = Target
  9. Administrative
Actor = Administrator
Target = User
 10. Automated
Actor = SYSTEM
Target = Identity
 11. External Provisioning
SCIM/HR/enterprise directory puede solicitar lifecycle changes.
 12. External Authority
Debe ser explícita.
No cualquier external event podrá suspender una identidad.
 13. Provisioning Provider
interface ExternalIdentityLifecycleProviderInterface
{
    public function lifecycleAuthority(
        ExternalIdentityReference $identity
    ): ExternalLifecycleAuthority;
}
 14. Source of Truth
Puede configurarse:
LOCAL
EXTERNAL
HYBRID
 15. External-Managed Identity
Ejemplo:
Corporate SSO + SCIM
La aplicación puede prohibir local reactivation si directorio externo indica:
DEACTIVATED
 16. External State Reconciliation
Local ACTIVE
External DISABLED
debe producir reconciliation policy.
 17. Reconciliation
No utilizar automáticamente:
last writer wins
 18. Authority Precedence
Debe declararse.
Ejemplo:
Platform Security Suspension
        >
   >
Tenant State
        >
External Provisioning
        >
Self-Service
dependiendo de operación.

## 144. Lifecycle Policy Engine

interface IdentityLifecyclePolicyEngineInterface
{
    public function evaluate(
        IdentityLifecycleOperation $operation,
        IdentityLifecycleContext $context
    ): IdentityLifecyclePolicyDecision;
}

## 145. Policy Hierarchy

Puede seguir:
Framework Floor
      ↓
Platform
      ↓
Environment
      ↓
Realm
      ↓
Tenant
      ↓
Application
      ↓
Identity Type
      ↓
Operation

## 146. Hardening

Tenant podrá endurecer lifecycle requirements.

## 147. Tenant Cannot Weaken Platform Floor

Ejemplo:
Platform:
admin deletion requires passkey

Tenant:
password only
resultado:
passkey requirement remains

## 148. Lifecycle Reason

Toda transición deberá tener reason code.

## 149. Reason != Free Text

Preferir:
reasonCode
+
optional human note

## 150. Audit

Toda transición deberá generar audit record.

## 151. Audit Record

Debe incluir:
Actor
Target Identity
Previous State
New State
Scope
Tenant
Realm
Reason
Policy Version
Authentication Context
Authorization Result
Timestamp
Operation ID

## 152. Lifecycle History

interface IdentityLifecycleHistoryRepositoryInterface
{
    public function history(
        IdentityReference $identity
    ): IdentityLifecycleHistory;
}

## 153. History Is Append-Oriented

No sobrescribir:
suspended_at
como único historial.

## 154. Example History

2026-01-03 ACTIVE
2026-03-10 SUSPENDED
2026-03-12 ACTIVE
2026-07-01 DEACTIVATED
2026-07-20 REACTIVATION_PENDING
2026-07-21 ACTIVE

## 155. Temporal Queries

Puede responder:
What was this identity's state at time T?
si retention/audit policy lo permite.

## 156. Event Model

Eventos principales:
IdentityProvisioningStarted
IdentityProvisioned
IdentityActivationRequired
IdentityActivated

IdentityLocked
IdentityLockExpired
IdentityUnlocked

IdentitySuspended
IdentitySuspensionExpired
IdentityReinstated

IdentityDeactivated
IdentityReactivationRequested
IdentityReactivated

IdentityDeletionRequested
IdentityDeletionScheduled
IdentityDeletionCancelled
IdentityDeletionStarted
IdentityDeleted
IdentityErasureRequested
IdentityErasureCompleted

IdentityRetired

## 157. Events vs Commands

SuspendIdentity
es Command.
IdentitySuspended
es Event.

## 158. Event Outbox

Lifecycle mutation y critical event deberán persistirse coherentemente.
DB Transaction
   │
   ├── lifecycle state
   ├── lifecycle version
   └── outbox event

## 159. Notification

Cambios importantes pueden generar:
account suspended
account reactivated
deletion requested
deletion cancelled
account deleted

## 160. Notification Failure

No revierte lifecycle mutation.

## 161. Security Notification Integration

Documento 40 determinará canales/alerting.

## 162. Deletion Notification

Especialmente importante antes de grace-period expiration.
Ejemplo:
Deletion requested
Deletion in 7 days
Deletion in 24 hours
Deletion completed
configurable.

## 163. Suspicious Lifecycle Change

Una transición puede producir security signal.
Ejemplo:
account deletion requested
immediately after suspicious recovery

## 164. Lifecycle → Security Signals

IdentityDeactivationRequested
IdentityDeletionRequested
IdentityReactivationRequested
IdentitySuspensionChanged
pueden alimentar documento 40.

## 165. Security Incident → Lifecycle

También dirección inversa:
Confirmed Account Takeover
        ↓
SUSPENDED / RESTRICTED
según policy.

## 166. Avoid Circular Logic

Debe existir orchestration explícita para evitar:
incident suspends identity
→ suspension creates incident
→ incident suspends identity
→ ...

## 167. Causation ID

Eventos deberán poder conservar:
eventId
correlationId
causationId
operationId

## 168. Idempotency

Lifecycle commands deberán ser idempotentes cuando tenga sentido.

## 169. Example

Suspend Identity X
repetido:
SUSPENDED
ALREADY_SUSPENDED

## 170. Command ID

final readonly class IdentityLifecycleOperationId
{
    public function __construct(
        public string $value
    ) {}
}

## 171. Optimistic Concurrency

Debe evitar:
Admin A → Suspend
User → Deactivate
Admin B → Reactivate
perdiendo cambios.

## 172. Compare-and-Swap

expected lifecycle version = 17
si actual:
18
→ conflict.

## 173. Lifecycle Conflict

AUTH_IDENTITY_LIFECYCLE_VERSION_CONFLICT

## 174. Distributed Propagation

Cambio:
ACTIVE → SUSPENDED
deberá propagarse a:
session validators
authentication nodes
API gateways
workers
websocket servers
SPA runtimes
según arquitectura.

## 175. Eventual Consistency

Puede permitirse para UI.
No necesariamente para security-critical eligibility.

## 176. Critical Eligibility

Debe consultar:
authoritative state
o una réplica con guarantees adecuadas.

## 177. Stale Cache

No deberá permitir que una identidad suspendida continúe autenticándose indefinidamente.

## 178. Cache Strategy

Lifecycle Version
Security Epoch
bounded TTL
distributed invalidation

## 179. Cache Key

identity_lifecycle:{identity}:{version}
conceptualmente.

## 180. Negative Cache

Puede cachear identidad inexistente con TTL corto.

## 181. Enumeration Safety

No usar diferencias de cache timing para revelar cuentas cuando pueda evitarse.

## 182. Session Integration

Session deberá conservar referencia suficiente para detectar lifecycle changes.

## 183. Session Authentication Snapshot

Puede contener:
identityId
identityLifecycleVersion
identitySecurityEpoch

## 184. Snapshot Is Not Authority

Si versiones cambian:
revalidate

## 185. Remember-Me

Una identidad:
SUSPENDED
DEACTIVATED
DELETED
no deberá poder restaurar Authentication mediante remember-me.

## 186. Recovery

Recovery no debe bypass lifecycle.
Ejemplo:
SUSPENDED
no se convierte automáticamente en:
ACTIVE
porque password reset fue exitoso.

## 187. Recovery Purpose

Recovery recupera Authentication authority.
No cambia administrative lifecycle salvo policy explícita.

## 188. Passkeys

Passkey válida para identidad suspendida:
cryptographic verification = valid
authentication eligibility = denied

## 189. Federation

OIDC token válido:
signature = valid
issuer = trusted
subject = mapped
pero identity:
DEACTIVATED
→ no login normal.

## 190. Authentication Success Ordering

Debe ser:
Credential Evidence
       ↓
Identity Resolution
       ↓
Lifecycle Eligibility
       ↓
Security State
       ↓
Policy
       ↓
Authentication Success
con optimizaciones seguras cuando proceda.

## 191. Information Leakage

El orden interno no obliga a revelar motivo al cliente.

## 192. Credential Verification Cost

Para ciertos flujos puede ser necesario equilibrar:
enumeration resistance
password hashing cost
disabled accounts
DoS resistance

## 193. Dummy Verification

Password authenticators pueden utilizar dummy password hash verification para identidades inexistentes/ineligibles cuando sea apropiado.

## 194. Suspension Does Not Expose Password Validity

Una respuesta pública no deberá indicar:
correct password but suspended
salvo producto/policy explícito y seguro.

## 195. Lifecycle Snapshot

final readonly class IdentityLifecycleSnapshot
{
    public function __construct(
        public IdentityReference $identity,
        public IdentityLifecycleState $state,
        public IdentityLifecycleVersion $version,
        public DateTimeImmutable $capturedAt,
    ) {}
}

## 196. Snapshot Immutability

Especialmente importante en FrankenPHP.

## 197. No Mutable User Singleton

No:
Auth::$currentUser->state = ...
compartido entre requests.

## 198. Request Context

Lifecycle context deberá ser request/fiber scoped.

## 199. FrankenPHP

Nunca mantener:
current identity state
current tenant lifecycle
current suspension
current deletion request
en mutable singleton de worker.

## 200. Worker Reset

Limpiar entre requests:
IdentityLifecycleContext
EligibilityDecision
LifecycleSnapshot
TransitionContext
TenantMembershipSnapshot

## 201. Fiber Safety

Dos requests concurrentes:
Fiber A → User 100 ACTIVE
Fiber B → User 200 SUSPENDED
no pueden contaminarse.

## 202. Queue Workers

Misma regla.

## 203. Long-Running Commands

CLI workers deben resetear contexto entre operations.

## 204. Machine Identity Lifecycle

Máquinas requieren semantics particulares.

## 205. Machine States

Podría utilizarse:
enum MachineIdentityLifecycleState: string
{
    case Provisioning = 'provisioning';
    case Active = 'active';
    case Suspended = 'suspended';
    case Quarantined = 'quarantined';
    case Deactivated = 'deactivated';
    case Retired = 'retired';
    case Deleted = 'deleted';
}

## 206. Retired

RETIRED es especialmente útil para máquinas.
Significa:
La identidad ya no debe volver a utilizarse, pero su historial debe preservarse.

  1. Retired != Deleted
Puede conservar:
audit identity
certificate history
deployment history
incident history
sin aceptar Authentication.
  2. Machine Retirement
Ejemplo:
service replaced
workload architecture migrated
CI agent permanently removed
  3. Machine Reactivation
Por defecto:
RETIRED
→ ACTIVE
debería prohibirse.
Crear nueva machine identity suele ser más seguro.
  4. Machine Suspension
Puede permitir remediation.
ACTIVE
→ SUSPENDED
→ ACTIVE
  5. Machine Quarantine
Documento 40.
ACTIVE
→ QUARANTINED
por security incident.
  6. Workload Identity
Identidades efímeras pueden expirar naturalmente en lugar de deactivation tradicional.
  7. Identity Expiration
VoltStack podrá soportar:
final readonly class IdentityExpiration
{
    public function __construct(
        public DateTimeImmutable $expiresAt
    ) {}
}
  8. Temporary Accounts
Ejemplos:
contractor
guest
temporary admin
ephemeral integration
  9. Expiration
Al alcanzar fecha:
ACTIVE
→ DEACTIVATED
o:
ACTIVE
→ EXPIRED
si se decide añadir estado explícito.
 10. Prefer Reason Over Excessive States
No convertir cada causa en lifecycle state.
Por ejemplo:
DEACTIVATED
reason = EXPIRATION
puede ser mejor que añadir:
EXPIRED
si no cambia comportamiento.
 11. State Explosion
Evitar:
SUSPENDED_SECURITY
SUSPENDED_ADMIN
SUSPENDED_BILLING
SUSPENDED_LEGAL
...
 12. State + Reason
Preferir:
State = SUSPENDED
Reason = SECURITY
 13. Effective State
Con múltiples scopes puede necesitarse calcular:
interface EffectiveIdentityLifecycleResolverInterface
{
    public function resolve(
        IdentityReference $identity,
        IdentityLifecycleResolutionContext $context
    ): EffectiveIdentityLifecycle;
}
 14. Example
Global = ACTIVE
Realm = ACTIVE
Tenant = SUSPENDED
en Tenant X:
Effective = SUSPENDED
 15. Another Example
Global = SUSPENDED
Tenant = ACTIVE
resultado:
Effective = SUSPENDED
Tenant no puede override global suspension.
 16. Composition
Lifecycle scope composition debe ser monotónica respecto a restricciones.
 17. Restriction Ordering
Conceptualmente:
ACTIVE
<
LOCKED
<
SUSPENDED
<
DEACTIVATED
<
DELETED
pero no debe asumirse que todos los estados forman una simple escala.
 18. Why Not Simple Enum Max
DELETION_PENDING tiene semantics diferentes a SUSPENDED.
Por tanto usar resolver formal.
 19. Lifecycle Constraint
Puede modelarse:
final readonly class IdentityLifecycleConstraint
{
    public function __construct(
        public IdentityLifecycleScope $scope,
        public IdentityLifecycleConstraintType $type,
        public IdentityLifecycleState $state,
    ) {}
}
 20. Effective Lifecycle
final readonly class EffectiveIdentityLifecycle
{
    public function __construct(
        public IdentityLifecycleState $state,
        public array $contributingConstraints,
        public IdentityLifecycleVersion $version,
    ) {}
}
 21. Explainability
Debe responder internamente:
Why is authentication denied?
Ejemplo:
Global Identity = ACTIVE
Tenant Membership = SUSPENDED
Reason = ORGANIZATION_REQUEST
 22. Public Explanation
Puede mostrar:
Your access to this organization is currently unavailable.
sin revelar detalles internos.
 23. Admin Explanation
Puede mostrar:
Suspended by tenant administrator
Reason: OFFBOARDING
Ticket: HR-2034
según Authorization.
 24. Security Explanation
Security operator puede obtener mayor detalle.
 25. Deletion Privacy
Una identidad eliminada no deberá seguir apareciendo innecesariamente en:
user search
autocomplete
admin selectors
member lists
 26. Historical References
Pero registros históricos pueden necesitar mostrar:
Deleted User
o pseudonymous identifier.
 27. Tombstone
Puede utilizarse:
final readonly class DeletedIdentityTombstone
{
    public function __construct(
        public IdentityId $identityId,
        public DateTimeImmutable $deletedAt,
        public IdentityDeletionReference $deletionReference,
    ) {}
}
 28. Tombstone Purpose
Permite:
prevent credential resurrection
maintain referential history
deduplicate deletion
audit
 29. Tombstone Data Minimization
No debe convertirse en copia permanente del perfil eliminado.
 30. Identifier Reuse
Tema crítico.
 31. Email Reuse
Si cuenta A elimina:
<alice@example.com>
y después una persona registra el mismo email, no debe heredarse automáticamente:
old sessions
old OAuth links
old passkeys
old tenant memberships
old audit authority
 32. Stable Identity ID
Nueva cuenta:
Identity ID = NEW
aunque identifier sea reutilizado.
 33. External Subject Reuse
OIDC:
issuer + subject
debe tratarse con reglas de provider específicas.
 34. Identity Resurrection Attack
Nunca:
email matches deleted account
→ restore deleted identity
automáticamente.
 35. Username Reuse
Debe ser configurable.
 36. Reserved Identifiers
Después de deletion pueden mantenerse temporalmente reservados sin conservar más PII de la necesaria.
 37. Hashed Reservation
Puede utilizarse con cautela según threat model.
 38. GDPR / Privacy Regulations
El Core deberá ofrecer primitives para:
erasure
retention
anonymization
legal holds
sin asumir una jurisdicción universal.
 39. Compliance Adapter
interface IdentityDataRetentionPolicyInterface
{
    public function plan(
        IdentityReference $identity,
        IdentityDeletionContext $context
    ): IdentityDataRetentionPlan;
}
 40. Retention Plan
Puede clasificar datos:
ERASE
ANONYMIZE
RETAIN
RETAIN_UNTIL
TRANSFER
 41. Authentication Secrets
Después de deletion deberían normalmente:
REVOKE + ERASE
cuando retention/legal constraints no exijan otra cosa.
 42. Password Hash
No existe razón normal para conservarlo después de account deletion completa.
 43. Passkey Public Keys
Pueden eliminarse tras revocation/retention processing.
 44. Audit Records
Pueden requerir retención.
Pero deben minimizar PII.
 45. Security Incident Records
Documento 40 puede tener retention independiente.
 46. Anonymization
Debe ser irreversible cuando se declare completa.
 47. Pseudonymization != Anonymization
No tratarlas como equivalentes.
 48. Deletion Completion
No debería declararse:
all user data deleted
si solo Auth terminó su parte.
 49. Scoped Completion
Preferir:
Authentication identity deletion completed.
 50. Global Erasure Completion
Solo Data Lifecycle Orchestrator puede afirmar global completion.
 51. Ownership Transfer
Antes de deletion puede ser necesario transferir:
tenant ownership
API applications
service accounts
billing ownership
security administrator responsibilities
 52. Last Tenant Owner
No permitir eliminar una identidad si deja tenant sin owner cuando policy lo prohíba.
 53. Authorization Integration
Esta verificación pertenece principalmente al tenant/business domain, pero Auth lifecycle puede exigir un precondition provider.
 54. Deletion Precondition Provider
interface IdentityDeletionPreconditionProviderInterface
{
    public function evaluate(
        IdentityReference $identity
    ): IdentityDeletionPreconditionSet;
}
 55. Plugin Preconditions
Packages podrán añadir:
transfer project ownership
remove legal hold
resolve outstanding security incident
 56. Plugin Security
Un plugin no podrá bypass:
platform deletion floor
authorization
reauthentication
audit
retention hold
 57. Lifecycle Extension
Plugins pueden añadir:
transition guards
preconditions
side-effect contributors
notification contributors
retention contributors
 58. Custom States
Se recomienda evitar custom lifecycle states indiscriminados.
 59. Prefer Custom Constraints
En muchos casos:
ACTIVE + application constraint
es mejor que modificar core enum.
 60. State Extensibility
Si se permiten custom states deberán declarar:
authentication eligibility
allowed transitions
terminal status
reactivation semantics
composition semantics
 61. Compile-Time Validation
Documento 27/36.
Lifecycle definitions configurables deberán validarse.
 62. Invalid Lifecycle Graph
Debe rechazarse:
DELETED → ACTIVE
si contradice framework floor.
 63. Cycles
Algunos ciclos son válidos:
ACTIVE ↔ SUSPENDED
ACTIVE ↔ LOCKED
otros no.
 64. Terminal States
DELETED
RETIRED
normalmente terminales.
 65. Terminal State Contract
interface TerminalIdentityLifecycleStateInterface
{
    public function permitsAuthentication(): bool;
}
conceptualmente.
 66. Lifecycle Compiler
interface IdentityLifecycleCompilerInterface
{
    public function compile(
        IdentityLifecycleDefinitionSet $definitions
    ): CompiledIdentityLifecycle;
}
 67. Compilation
Puede precomputar:
transition matrix
guards
side effects
requirements
terminal states
scope composition
 68. Runtime Performance
No parsear configuración completa en cada login.
 69. Compiled Lifecycle
Debe ser:
immutable
versioned
cacheable
secret-free
 70. Lifecycle Definition Version
final readonly class IdentityLifecycleDefinitionVersion
{
    public function __construct(
        public string $value
    ) {}
}
 71. Deployment Compatibility
Cambiar lifecycle graph requiere considerar identities existentes.
 72. Migration Validation
Ejemplo:
old version has state TEMP_DISABLED
new version removes it
debe existir migration strategy.
 73. Lifecycle Schema Migration
Puede requerir:
TEMP_DISABLED → SUSPENDED
antes de activar nueva definición.
 74. Operational Tooling
CLI conceptual:
volt auth:identity:status {identity}
volt auth:identity:suspend {identity}
volt auth:identity:reinstate {identity}
volt auth:identity:deactivate {identity}
volt auth:identity:reactivate {identity}
volt auth:identity:delete {identity}
volt auth:identity:history {identity}
 75. CLI Security
CLI no implica bypass.
Debe conservar:
actor
authorization
privileged authentication
reason
audit
cuando corresponda.
 76. Emergency Operations
Break-glass puede permitir lifecycle operations extraordinarias.
 77. Break-Glass Does Not Erase Audit
Nunca.
 78. Bulk Suspension
Enterprise use case:
suspend 10,000 identities
 79. Bulk Operations
Necesitan:
batch operation ID
per-item outcome
idempotency
rate governance
audit summary
distributed processing
 80. Bulk Command
final readonly class BulkIdentityLifecycleCommand
{
    public function __construct(
        public IdentityLifecycleOperationId $operationId,
        public array $identities,
        public IdentityLifecycleOperation $operation,
    ) {}
}
 81. Partial Bulk Failure
Debe reportarse:
9,998 suspended
1 already suspended
1 denied due to legal state
 82. Bulk Atomicity
No exigir transacción global gigantesca.
 83. SCIM
Enterprise adapters podrán mapear:
SCIM active=false
a lifecycle semantics configuradas.
 84. SCIM active=false
No asumir universalmente:
DELETE
Normalmente podría significar:
DEACTIVATED
 85. Directory Reconciliation
Debe evitar recrear cuentas eliminadas accidentalmente.
 86. Deprovisioning
Enterprise flow:
Directory disables user
       ↓
VoltStack DEACTIVATED
       ↓
revoke sessions
       ↓
disable credentials
       ↓
remove tenant access
según authority.
 87. Reprovisioning
No necesariamente:
active=true
→ reactivate instantly
si security policy exige review.
 88. Invitation Lifecycle
Tenant invitation no debe requerir Identity ACTIVE antes de aceptar.
 89. Invitation vs Identity
Invitation
    ≠
Identity Lifecycle
 90. Identity Linking
Documento 34.
No permitir linking hacia:
DELETED
SUSPENDED
sin policy explícita.
 91. Merge
Identity merge deberá considerar lifecycle states.
 92. Merge Example
Identity A = ACTIVE
Identity B = DEACTIVATED
no simplemente:
merge rows
 93. Merge Lifecycle Plan
Debe determinar:
surviving identity
credentials
sessions
tenant memberships
audit history
deleted identity tombstone
 94. Merge and Deleted Identity
No resucitar deleted identity.
 95. Identity Security Center
Documento 35 deberá mostrar lifecycle state cuando sea relevante.
 96. Self-Service View
Ejemplo:
Account status: Active
o:
Account scheduled for deletion: September 15
 97. Admin View
Puede mostrar:
Lifecycle
Suspension reason
Deletion request
Reactivation state
History
según authority.
 98. Security Center Is Projection
No será source of truth del lifecycle.
 99. Commands
UI deberá enviar commands al Lifecycle Orchestrator.
No:
UPDATE users SET active = 0
100. Application Integration
Conceptual DX:
Auth::identity($user)->deactivate();

Auth::identity($user)->requestDeletion();

Auth::identity($user)->cancelDeletion();

Auth::identity($user)->requestReactivation();

## 307. Administrative DX

Auth::identity($user)->suspend(
    reason: IdentitySuspensionReason::Administrative
);

## 308. Query DX

$status = Auth::identity($user)->lifecycle();

if ($status->isActive()) {
    // ...
}

## 309. Eligibility DX

$decision = Auth::identity($user)
    ->authenticationEligibility($context);

## 310. Avoid Convenience Bypass

No ofrecer:
$user->forceActivate();
como public generic API.

## 311. Internal Emergency API

Si existe deberá ser:
explicit
privileged
audited
scoped

## 312. Persistence Model

Una implementación posible:
identities
identity_lifecycle_transitions
identity_suspensions
identity_lockouts
identity_deletion_requests
identity_retention_holds
identity_reactivation_requests
identity_tombstones

## 313. Current State Projection

identities.lifecycle_state puede existir como current projection.

## 314. History

Pero transitions deberán conservar historial.

## 315. Event Sourcing

No será obligatorio.

## 316. Event-Sourced Adapter

Podrá implementarse mediante contracts.

## 317. Relational Adapter

Será implementación default razonable.

## 318. Transaction Boundary

Current state + transition history + outbox deberán idealmente persistirse en misma transacción local.

## 319. Cross-System Side Effects

Se ejecutan después mediante eventos/saga.

## 320. Delete Saga Failure

Ejemplo:
Auth deleted ✓
Storage cleanup failed ✗
No reactivar automáticamente Authentication.

## 321. Erasure Retry

Storage cleanup puede reintentarse.

## 322. Deletion Operation State

Puede existir:
REQUESTED
SCHEDULED
AUTH_DISABLED
ERASURE_IN_PROGRESS
PARTIALLY_COMPLETED
COMPLETED
FAILED
separado del Identity Lifecycle.

## 323. Lifecycle State Simplicity

Esto evita llenar IdentityLifecycleState con estados técnicos de background processing.

## 324. State Machine vs Workflow

Identity Lifecycle State Machine
modela estado de identidad.
Deletion Workflow
modela proceso operativo.

## 325. Reactivation Workflow

Misma separación.

## 326. Suspension Workflow

Puede incluir approval sin añadir:
SUSPENSION_APPROVAL_PENDING
al lifecycle core.

## 327. Workflow Reference

Lifecycle transition puede guardar:
workflowId

## 328. Administrative Approval

Algunas organizaciones pueden requerir dual control.

## 329. Example

Suspend Platform Administrator
puede requerir dos security administrators.

## 330. Approval Is Authorization Governance

Integrar sistema de Authorization approvals.

## 331. Lifecycle Security Boundary

Auth lifecycle nunca debe confiar únicamente en UI role.

## 332. Database Administrator

Acceso directo DB puede bypass framework; operational security debe tratarlo aparte.

## 333. Integrity Constraints

DB puede reforzar:
valid enum/state
unique active deletion request
version increment
foreign keys
cuando sea compatible.

## 334. State Transition Enforcement

No depender únicamente de DB trigger.
Core debe validar domain transition.

## 335. Testing

El sistema requerirá pruebas unitarias, integración, distribución, seguridad y concurrencia.

## 336. Unit Tests

Cubrir:
valid transitions
invalid transitions
lock expiration
suspension
deactivation
reactivation
deletion
tenant composition

## 337. Security Tests

Cubrir:
suspended passkey cannot login
deleted password cannot login
remember-me cannot restore deactivated identity
recovery cannot bypass suspension
tenant admin cannot globally suspend
deleted email does not resurrect identity

## 338. Concurrency Tests

suspend vs reactivate
delete vs cancel deletion
lock expiry vs admin suspension
external deprovision vs local reactivation

## 339. Distributed Tests

Node A authenticates
Node B suspends
Node C rejects next request

## 340. FrankenPHP Tests

Request A ACTIVE identity
Request B SUSPENDED identity
sin leakage.

## 341. Fiber Tests

Múltiples lifecycle contexts simultáneos.

## 342. Deletion Tests

Verificar:
sessions revoked
credentials revoked
pending auth transactions invalidated
tombstone created
outbox generated

## 343. Reactivation Tests

Verificar que credenciales antiguas no se restauren automáticamente cuando policy exige renovación.

## 344. Multi-Tenant Tests

Tenant A suspension
no afecta Tenant B salvo global lifecycle state.

## 345. Machine Tests

retired service identity
no puede obtener nuevos tokens.

## 346. External Provisioning Tests

SCIM disable/re-enable respetando authority hierarchy.

## 347. Property-Based Tests

Especialmente útiles para state machine.
Invariante:
No valid transition path from DELETED to ACTIVE
bajo default policy.

## 348. Model Checking

Para configuraciones enterprise complejas podría utilizarse validación formal del transition graph.

## 349. Failure Taxonomy

AUTH_IDENTITY_NOT_FOUND
AUTH_IDENTITY_LIFECYCLE_INVALID_STATE
AUTH_IDENTITY_LIFECYCLE_INVALID_TRANSITION
AUTH_IDENTITY_LIFECYCLE_VERSION_CONFLICT

AUTH_IDENTITY_ACTIVATION_REQUIRED
AUTH_IDENTITY_LOCKED
AUTH_IDENTITY_SUSPENDED
AUTH_IDENTITY_DEACTIVATED
AUTH_IDENTITY_DELETION_PENDING
AUTH_IDENTITY_DELETED
AUTH_IDENTITY_RETIRED

AUTH_IDENTITY_SUSPENSION_DENIED
AUTH_IDENTITY_REINSTATEMENT_DENIED

AUTH_IDENTITY_DEACTIVATION_DENIED
AUTH_IDENTITY_REACTIVATION_DENIED
AUTH_IDENTITY_REACTIVATION_REQUIREMENT_FAILED

AUTH_IDENTITY_DELETION_DENIED
AUTH_IDENTITY_DELETION_BLOCKED
AUTH_IDENTITY_DELETION_LEGAL_HOLD
AUTH_IDENTITY_DELETION_OWNERSHIP_TRANSFER_REQUIRED
AUTH_IDENTITY_DELETION_ALREADY_PENDING

AUTH_IDENTITY_DELETION_CANCELLATION_DENIED

AUTH_IDENTITY_EXTERNAL_AUTHORITY_CONFLICT
AUTH_IDENTITY_LIFECYCLE_POLICY_UNAVAILABLE
AUTH_IDENTITY_LIFECYCLE_STATE_STALE

## 350. Public Error Normalization

Login puede responder genéricamente:
Authentication failed.
mientras internamente:
AUTH_IDENTITY_SUSPENDED

## 351. Security Invariants — Lifecycle

AUTH-LIFE-01
Cada identidad tendrá un lifecycle state explícito.
AUTH-LIFE-02
Lifecycle state no será equivalente a Authentication eligibility.
AUTH-LIFE-03
Transitions deberán pasar por state machine.
AUTH-LIFE-04
Invalid transitions deberán rechazarse.
AUTH-LIFE-05
DELETED será terminal por defecto.
AUTH-LIFE-06
RETIRED será terminal por defecto para machine identities.
AUTH-LIFE-07
Cada transición tendrá reason y actor.
AUTH-LIFE-08
Lifecycle history será auditable.

## 352. Security Invariants — Lockout

AUTH-LOCK-01
Lockout no equivaldrá a suspension.
AUTH-LOCK-02
Credential lockout podrá existir sin identity lockout.
AUTH-LOCK-03
Temporary lockout deberá tener expiry semantics claras.
AUTH-LOCK-04
Failed login attacks no deberán permitir permanent account DoS fácilmente.
AUTH-LOCK-05
Public errors minimizarán account enumeration.

## 353. Security Invariants — Suspension

AUTH-SUSP-01
Suspended identity no podrá autenticarse normalmente.
AUTH-SUSP-02
Recovery no eliminará administrative suspension automáticamente.
AUTH-SUSP-03
Valid credential no bypassará suspension.
AUTH-SUSP-04
Tenant suspension no se convertirá en global suspension implícita.
AUTH-SUSP-05
Suspension authority deberá registrarse.

## 354. Security Invariants — Deactivation

AUTH-DEACT-01
Deactivation != deletion.
AUTH-DEACT-02
Deactivated identities no podrán crear sesiones normales.
AUTH-DEACT-03
Remember-me no restaurará deactivated identity.
AUTH-DEACT-04
Reactivation será ceremony, no boolean update.
AUTH-DEACT-05
Credential restoration dependerá de policy.

## 355. Security Invariants — Deletion

AUTH-DEL-01
Deletion revocará Authentication authority.
AUTH-DEL-02
Deleted credentials nunca deberán autenticar.
AUTH-DEL-03
Deleted sessions nunca deberán restaurarse.
AUTH-DEL-04
Pending Authentication transactions deberán invalidarse.
AUTH-DEL-05
Deletion no implicará borrar datos de otros dominios directamente.
AUTH-DEL-06
Physical erasure será proceso separado.
AUTH-DEL-07
Legal retention podrá sobrevivir account deletion.
AUTH-DEL-08
Deleted identifier reuse no resucitará antigua identity.
AUTH-DEL-09
Tombstones no conservarán PII innecesaria.
AUTH-DEL-10
Deletion será idempotente.

## 356. Security Invariants — Reactivation

AUTH-REACT-01
Reactivation normal aplicará a DEACTIVATED, no DELETED.
AUTH-REACT-02
Reactivation podrá exigir fresh Authentication.
AUTH-REACT-03
Reactivation podrá exigir credential review.
AUTH-REACT-04
Reactivation no limpiará security incident automáticamente.
AUTH-REACT-05
External-managed identities respetarán external authority.

## 357. Security Invariants — Multi-Tenant

AUTH-LIFE-TENANT-01
Global Identity State y Tenant Membership State serán distintos.
AUTH-LIFE-TENANT-02
Tenant A no podrá modificar lifecycle de Tenant B.
AUTH-LIFE-TENANT-03
Tenant admin no obtendrá global lifecycle authority implícita.
AUTH-LIFE-TENANT-04
Global restriction tendrá precedencia sobre local permissiveness.
AUTH-LIFE-TENANT-05
Tenant policy podrá endurecer pero no debilitar platform security floor.

## 358. Security Invariants — Runtime

AUTH-LIFE-RT-01
No habrá mutable global current lifecycle state.
AUTH-LIFE-RT-02
Lifecycle snapshots serán request-scoped.
AUTH-LIFE-RT-03
Fiber contexts estarán aislados.
AUTH-LIFE-RT-04
Queue workers resetearán context.
AUTH-LIFE-RT-05
Stale cache no será authority para critical lifecycle decision.
AUTH-LIFE-RT-06
Lifecycle mutations serán versionadas.

## 359. Anti-Pattern

$user->active = false;
$user->save();
como sistema completo de suspensión.

## 360. Anti-Pattern

password reset successful
→ suspended account active

## 361. Anti-Pattern

tenant suspended user
→ global user disabled

## 362. Anti-Pattern

deleted_at != null
como única semántica de todo el lifecycle.

## 363. Anti-Pattern

soft delete
→ sessions remain valid

## 364. Anti-Pattern

deactivate
→ delete every credential immediately
sin considerar reactivation policy.

## 365. Anti-Pattern

reactivate
→ old sessions become valid again

## 366. Anti-Pattern

email reused
→ attach new person to old identity

## 367. Anti-Pattern

SCIM active=true
→ bypass security freeze

## 368. Anti-Pattern

admin panel checkbox
→ bypass Authorization + reauthentication

## 369. Anti-Pattern

account deletion
→ Auth module directly deletes invoices

## 370. Anti-Pattern

deleted account
→ restore all OAuth links automatically

## 371. Anti-Pattern

cached ACTIVE state
→ accept indefinitely after suspension

## 372. Anti-Pattern

static $currentIdentityLifecycle;
en FrankenPHP.

## 373. Componentes principales

Identity

IdentityLifecycleState
IdentityLifecycleVersion
IdentityLifecycleSnapshot

IdentityLifecycleTransition
IdentityLifecycleStateMachine
IdentityLifecycleTransitionGuard

IdentityLifecyclePolicyEngine
IdentityLifecycleOrchestrator

AuthenticationEligibilityResolver
AuthenticationEligibilityDecision

IdentityLockout
IdentitySuspension

IdentityDeletionRequest
IdentityDeletionPolicy
IdentityDeletionPrecondition
IdentityRetentionHold
IdentityDataRetentionPlan

IdentityReactivationRequirement
IdentityReactivationPolicy

TenantIdentityMembershipState
EffectiveIdentityLifecycleResolver

DeletedIdentityTombstone

IdentityLifecycleHistory
IdentityLifecycleOperationId

## 374. Namespace sugerido

VoltStack\Quantum\Auth\IdentityLifecycle

## 375. Estructura sugerida

src/Quantum/Auth/IdentityLifecycle/
├── Contracts/
│   ├── IdentityLifecycleStateMachineInterface.php
│   ├── IdentityLifecycleTransitionGuardInterface.php
│   ├── IdentityLifecycleOrchestratorInterface.php
│   ├── IdentityLifecyclePolicyEngineInterface.php
│   ├── AuthenticationEligibilityResolverInterface.php
│   ├── EffectiveIdentityLifecycleResolverInterface.php
│   ├── IdentityDeletionPolicyInterface.php
│   ├── IdentityDeletionPreconditionProviderInterface.php
│   ├── IdentityReactivationPolicyInterface.php
│   ├── IdentityDataRetentionPolicyInterface.php
│   ├── IdentityLifecycleHistoryRepositoryInterface.php
│   └── ExternalIdentityLifecycleProviderInterface.php
│
├── Model/
│   ├── Identity.php
│   ├── IdentityId.php
│   ├── IdentityType.php
│   ├── IdentityVersion.php
│   └── IdentityReference.php
│
├── State/
│   ├── IdentityLifecycleState.php
│   ├── IdentityLifecycleVersion.php
│   ├── IdentityLifecycleSnapshot.php
│   ├── EffectiveIdentityLifecycle.php
│   └── IdentityLifecycleConstraint.php
│
├── Transition/
│   ├── IdentityLifecycleTransition.php
│   ├── IdentityLifecycleTransitionRequest.php
│   ├── IdentityLifecycleTransitionResult.php
│   ├── IdentityLifecycleTransitionReason.php
│   ├── IdentityLifecycleStateMachine.php
│   └── IdentityLifecycleTransitionGuard.php
│
├── Eligibility/
│   ├── AuthenticationEligibilityDecision.php
│   ├── AuthenticationEligibilityDecisionType.php
│   ├── AuthenticationEligibilityContext.php
│   └── AuthenticationEligibilityResolver.php
│
├── Lockout/
│   ├── IdentityLockout.php
│   ├── IdentityLockoutId.php
│   ├── IdentityLockoutReason.php
│   └── IdentityLockoutManager.php
│
├── Suspension/
│   ├── IdentitySuspension.php
│   ├── IdentitySuspensionId.php
│   ├── IdentitySuspensionReason.php
│   └── IdentitySuspensionManager.php
│
├── Deactivation/
│   ├── IdentityDeactivationReason.php
│   └── IdentityDeactivationManager.php
│
├── Deletion/
│   ├── IdentityDeletionRequest.php
│   ├── IdentityDeletionRequestId.php
│   ├── IdentityDeletionDecision.php
│   ├── IdentityDeletionDecisionType.php
│   ├── IdentityDeletionPolicy.php
│   ├── IdentityDeletionPrecondition.php
│   ├── IdentityRetentionHold.php
│   ├── IdentityRetentionHoldId.php
│   ├── DeletedIdentityTombstone.php
│   └── IdentityDeletionOrchestrator.php
│
├── Reactivation/
│   ├── IdentityReactivationRequirement.php
│   ├── IdentityReactivationContext.php
│   ├── IdentityReactivationPolicy.php
│   └── IdentityReactivationOrchestrator.php
│
├── Tenant/
│   ├── TenantIdentityMembershipState.php
│   ├── IdentityLifecycleScope.php
│   └── EffectiveIdentityLifecycleResolver.php
│
├── Retention/
│   ├── IdentityDataRetentionPlan.php
│   └── IdentityDataRetentionPolicy.php
│
├── History/
│   ├── IdentityLifecycleHistory.php
│   └── IdentityLifecycleHistoryRepository.php
│
├── Policy/
│   ├── IdentityLifecyclePolicyEngine.php
│   └── IdentityLifecyclePolicyDecision.php
│
├── Compiler/
│   ├── IdentityLifecycleCompiler.php
│   ├── IdentityLifecycleDefinitionSet.php
│   └── CompiledIdentityLifecycle.php
│
├── Events/
│   └── ...
│
├── Runtime/
│   ├── IdentityLifecycleContext.php
│   ├── IdentityLifecycleContextStorage.php
│   └── IdentityLifecycleResetter.php
│
└── Exceptions/
    └── ...

## 376. Configuración conceptual

return [

    'identity_lifecycle' => [

        'deactivation' => [
            'revoke_sessions' => true,
            'revoke_remember_me' => true,
            'preserve_authentication_methods' => true,
        ],

        'deletion' => [
            'grace_period' => '30 days',
            'allow_cancellation' => true,
            'revoke_sessions_immediately' => true,
            'invalidate_pending_transactions' => true,
            'create_tombstone' => true,
        ],

        'reactivation' => [
            'require_fresh_authentication' => true,
            'require_credential_review' => true,
        ],

        'lockout' => [
            'permanent_failure_lockout' => false,
        ],

    ],

];
Los valores son ilustrativos, no defaults normativos definitivos.

## 377. Ejemplo — suspensión administrativa

Tenant Administrator
        ↓
Suspend Membership
        ↓
Authorization
        ↓
Fresh Authentication if required
        ↓
Tenant Lifecycle Policy
        ↓
Tenant Membership
ACTIVE → SUSPENDED
        ↓
Revoke Tenant Sessions
        ↓
Audit + Event
La identidad global permanece:
ACTIVE

## 378. Ejemplo — suspensión global

Platform Security Operator
        ↓
Suspend Identity
        ↓
Privileged Authentication
        ↓
Authorization
        ↓
Global Lifecycle Policy
        ↓
ACTIVE → SUSPENDED
        ↓
Increment Security Epoch
        ↓
Revoke Sessions
        ↓
Revoke Persistent Login
        ↓
Distributed Invalidation
        ↓
Audit / Notification

## 379. Ejemplo — eliminación

User
  ↓
Request Account Deletion
  ↓
Fresh Authentication
  ↓
Deletion Policy
  ↓
Check:
  ├── legal hold
  ├── tenant ownership
  ├── security incident
  └── retention
  ↓
DELETION_PENDING
  ↓
Revoke normal Authentication
  ↓
Grace Period
  │
  ├── Cancel → restoration workflow
  │
  └── Expire
        ↓
     DELETED
        ↓
Revoke Authentication Artifacts
        ↓
Create Tombstone
        ↓
IdentityErasureRequested
        ↓
Cross-domain data lifecycle

## 380. Ejemplo — reactivation

DEACTIVATED Identity
        ↓
Reactivation Requested
        ↓
REACTIVATION_PENDING
        ↓
Verify Identity
        ↓
Check Security Incidents
        ↓
Check External Authority
        ↓
Review Credentials
        ↓
Rotate Password if required
        ↓
Verify Passkey/MFA
        ↓
Policy Approval
        ↓
ACTIVE
        ↓
Create Fresh Session
No se recuperan sesiones antiguas.

## 381. Ejemplo — credential válida, account inválida

Passkey Verification
       │
       ▼
Cryptographic Signature VALID
       │
       ▼
Identity Resolution
       │
       ▼
Identity = SUSPENDED
       │
       ▼
Authentication DENIED
Esto demuestra una regla central:
Valid Credential Evidence does not override Identity Lifecycle.

  1. Ejemplo — Multi-Tenant
Identity
Global State = ACTIVE

        │
        ├── Tenant A
        │     Membership = ACTIVE
        │
        ├── Tenant B
        │     Membership = SUSPENDED
        │
        └── Tenant C
              Membership = ACTIVE
Resultado:
Tenant A → Authentication eligible
Tenant B → Tenant authentication denied
Tenant C → Authentication eligible
sujeto a los demás controles.
  2. Ejemplo — Account Takeover + Lifecycle
Documento 40 detecta:
CONFIRMED_ACCOUNT_TAKEOVER
Response policy:
Identity Lifecycle = SUSPENDED
Security Posture = COMPROMISED
Protection State = RECOVERY_REQUIRED
Después:
Recovery
   ↓
Security Review
   ↓
Incident Resolved
   ↓
Lifecycle Reinstatement
   ↓
ACTIVE
Obsérvese que cada dimensión cambia mediante su propio subsistema.
  3. Laravel
Laravel proporciona primitives útiles como:
Authenticatable
SoftDeletes
Authentication Events
Sessions
Password Reset
Fortify
Sanctum
Notifications
Policies
Jobs
pero una aplicación suele definir manualmente estados como:
active
blocked
suspended_at
deleted_at
y middleware propio.
VoltStack formalizará estos conceptos en un Identity Lifecycle System reutilizable y coherente.
  4. Symfony
Symfony ofrece:
Security
UserInterface
UserCheckerInterface
Authenticators
Voters
Workflow
Messenger
EventDispatcher
Lock
UserCheckerInterface permite comprobar estados del usuario y Workflow puede modelar ciclos de vida.
VoltStack tomará esa rigurosidad, pero hará el lifecycle de Authentication un subsistema explícito conectado nativamente con:
Sessions
Credentials
Security Epochs
Multi-Tenancy
Recovery
Security Incidents
Distributed Runtime
FrankenPHP
  5. Diferenciador VoltStack
Laravel-like Account Management DX

+

Symfony-like Contracts and State Machines
+
Explicit Identity Lifecycle
+
Authentication Eligibility
+
Temporary Lockout
+
Administrative Suspension
+
Voluntary Deactivation
+
Deletion Grace Period
+
Deletion Orchestration
+
Retention Holds
+
Identity Tombstones
+
Secure Reactivation
+
Tenant Membership Lifecycle
+
External Provisioning Authority
+
Security Incident Integration
+
Credential/Session Invalidation
+
Distributed Lifecycle Versions
+
FrankenPHP/Fiber Safety

## 387. Decisiones arquitectónicas definitivas

VoltStack adoptará:

1. Identity Lifecycle será first-class.
2. Identity Lifecycle != Authentication Eligibility.
3. Lifecycle != Security Posture.
4. Lifecycle != Authorization.
5. Lifecycle != Tenant Membership.
6. Lifecycle != Credential State.
7. Lifecycle != Session State.
8. Lifecycle utilizará state machine.
9. Todas las transiciones serán explícitas.
10. Todas las transiciones tendrán actor.
11. Todas las transiciones tendrán reason.
12. Lifecycle history será auditable.
13. ACTIVE no garantizará Authentication.
14. LOCKED será diferente de SUSPENDED.
15. Credential lock podrá existir sin Identity lock.
16. Lockout automático será resistente a account DoS.
17. Suspension podrá ser global, tenant o realm scoped.
18. Tenant suspension no desactivará global identity.
19. Global suspension tendrá precedencia.
20. Deactivation != Suspension.
21. Deactivation != Deletion.
22. Deactivated identity podrá ser reactivable.
23. Deactivation revocará sesiones por default.
24. Deactivation no destruirá necesariamente todos los métodos.
25. Reactivation será security ceremony.
26. Reactivation no restaurará sesiones antiguas.
27. Reactivation podrá exigir credential rotation.
28. Deletion tendrá request explícito.
29. Deletion podrá tener grace period.
30. Deletion cancellation será security-sensitive.
31. Deletion tendrá preconditions.
32. Legal Hold será first-class.
33. Deletion lógica y physical erasure serán separadas.
34. Auth no eliminará directamente datos de otros dominios.
35. Cross-domain erasure será orchestrated.
36. DELETED será terminal por default.
37. Deleted credentials nunca autenticarán.
38. Deleted sessions nunca serán restauradas.
39. Pending Authentication transactions se invalidarán.
40. Security Epoch podrá incrementarse en lifecycle transitions.
41. Lifecycle Version será independiente de Security Epoch.
42. Identifier reuse no resucitará identity.
43. Nueva cuenta recibirá nuevo Identity ID.
44. Tombstones serán minimalistas.
45. External provisioning tendrá authority explícita.
46. External source no utilizará last-write-wins ciego.
47. SCIM disable no equivaldrá automáticamente a deletion.
48. Security Freeze no podrá ser eliminado por external reactivation.
49. Machine identities podrán utilizar RETIRED.
50. Retired machine identities no se reactivarán normalmente.
51. Effective Lifecycle resolverá global/tenant/realm constraints.
52. Restricciones superiores no podrán debilitarse localmente.
53. Lifecycle config podrá compilarse.
54. Compiled lifecycle será immutable.
55. Invalid lifecycle graphs serán rechazados.
56. Current state + history + outbox se persistirán coherentemente.
57. Commands críticos serán idempotentes.
58. Optimistic concurrency será soportada.
59. Bulk lifecycle operations serán first-class.
60. Lifecycle state crítico no dependerá indefinidamente de cache stale.
61. Distributed invalidation será soportada.
62. UI será projection, no authority.
63. Security Center no mutará DB directamente.
64. Public errors minimizarán enumeration.
65. Recovery no bypassará suspension.
66. Valid passkey no bypassará lifecycle.
67. Valid OIDC token no bypassará lifecycle.
68. Lifecycle changes podrán producir security signals.
69. Security incidents podrán producir lifecycle transitions.
70. Causation/correlation evitarán loops.
71. FrankenPHP context será request scoped.
72. Fiber isolation será obligatoria.
73. Queue workers resetearán lifecycle context.
74. No existirán mutable global lifecycle singletons.
75. Plugins no podrán bypass security floors.
76. Criterios de aceptación
El sistema estará arquitectónicamente completo cuando implemente al menos:
77. Identity.
78. IdentityType.
79. IdentityLifecycleState.
80. IdentityLifecycleVersion.
81. IdentityLifecycleSnapshot.
82. Lifecycle state machine.
83. Transition request.
84. Transition result.
85. Transition guards.
86. Transition reason.
87. Lifecycle history.
88. Lifecycle orchestrator.
89. Lifecycle policy engine.
90. Authentication Eligibility resolver.
91. Eligibility decisions.
92. Lockout model.
93. Temporary lock expiration.
94. Lockout anti-DoS.
95. Method-specific lockout.
96. Suspension model.
97. Suspension reasons.
98. Global suspension.
99. Tenant suspension.
100. Realm suspension.
101. Tenant Membership state.
102. Effective Lifecycle resolver.
103. Deactivation.
104. Deactivation reasons.
105. Session revocation on deactivation.
106. Persistent login revocation.
107. Credential preservation policies.
108. Deletion Request.
109. Deletion grace period.
110. Deletion cancellation.
111. Deletion policy.
112. Deletion preconditions.
113. Legal/Retention Hold.
114. Ownership transfer precondition.
115. Logical deletion.
116. Physical erasure orchestration.
117. Cross-domain deletion events.
118. Deletion idempotency.
119. Session invalidation.
120. Credential invalidation.
121. Device invalidation.
122. Pending transaction invalidation.
123. Security Epoch integration.
124. Deleted Identity Tombstone.
125. Identifier reuse protection.
126. Reactivation Request.
127. Reactivation Pending.
128. Reactivation Policy.
129. Reactivation Requirements.
130. Credential review.
131. Security Review integration.
132. Recovery integration.
133. External federation revalidation.
134. External lifecycle authority.
135. SCIM integration contract.
136. External reconciliation.
137. Machine lifecycle.
138. Machine retirement.
139. Machine quarantine integration.
140. Bulk operations.
141. Optimistic concurrency.
142. Operation IDs.
143. Transactional Outbox.
144. Lifecycle events.
145. Notifications.
146. Security signals.
147. Audit.
148. Explainability.
149. Privacy/data minimization.
150. Retention planning.
151. Compiled lifecycle definitions.
152. Lifecycle definition versioning.
153. Distributed invalidation.
154. Cache versioning.
155. Multi-region consistency strategy.
156. Public error normalization.
157. Unit tests.
158. Security tests.
159. Multi-tenant tests.
160. Distributed tests.
161. Concurrency tests.
162. Property-based state-machine tests.
163. Machine identity tests.
164. External provisioning tests.
165. FrankenPHP isolation tests.
166. Fiber isolation tests.
167. Arquitectura final
┌──────────────────────────────────────────────────────────────────┐
│                    IDENTITY LIFECYCLE                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│                        Identity                                  │
│                           │                                      │
│                           ▼                                      │
│                 Lifecycle State Machine                          │
│                           │                                      │
│          ┌────────────────┼───────────────────┐                   │
│          ▼                ▼                   ▼                   │
│       Lockout         Suspension         Deactivation             │
│          │                │                   │                   │
│          └────────────────┼───────────────────┘                   │
│                           │                                      │
│                           ▼                                      │
│               Authentication Eligibility                         │
│                           │                                      │
│        ┌──────────────────┼────────────────────┐                  │
│        ▼                  ▼                    ▼                  │
│     Sessions          Credentials          Devices               │
│        │                  │                    │                  │
│        └──────────────────┼────────────────────┘                  │
│                           ▼                                      │
│                    Security Epoch                                │
│                                                                  │
│  ──────────────────────────────────────────────────────────────  │
│                                                                  │
│                  ACCOUNT TERMINATION                              │
│                                                                  │
│      Deactivation                                                 │
│           │                                                       │
│           ├──────────► Reactivation                               │
│           │                                                       │
│           ▼                                                       │
│     Deletion Request                                              │
│           │                                                       │
│           ▼                                                       │
│    Deletion Pending                                               │
│       ┌───┴────┐                                                  │
│       ▼        ▼                                                  │
│    Cancel    Delete                                               │
│       │        │                                                  │
│       ▼        ▼                                                  │
│    Restore   DELETED                                              │
│                │                                                  │
│                ▼                                                  │
│        Data Lifecycle Events                                      │
│                │                                                  │
│      ┌─────────┼──────────┐                                       │
│      ▼         ▼          ▼                                       │
│    Erase    Anonymize   Retain                                    │
│                                                                  │
│  ──────────────────────────────────────────────────────────────  │
│                                                                  │
│                  MULTI-TENANT LIFECYCLE                           │
│                                                                  │
│ Global Identity = ACTIVE                                          │
│       │                                                           │
│       ├── Tenant A = ACTIVE                                       │
│       ├── Tenant B = SUSPENDED                                    │
│       └── Tenant C = REMOVED                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
168. Regla arquitectónica final
La regla central del sistema será:
VoltStack nunca utilizará un único booleano para representar el ciclo de vida y la seguridad de una identidad.

En su lugar:
Identity
   │
   ├── Lifecycle
   ├── Authentication Eligibility
   ├── Security Posture
   ├── Protection State
   ├── Tenant Membership
   ├── Authentication Methods
   ├── Credentials
   ├── Sessions
   └── Devices
serán dimensiones coordinadas pero independientes.
Esto permite distinguir correctamente situaciones como:
ACTIVE + SECURE
ACTIVE + COMPROMISED
ACTIVE + TENANT_SUSPENDED
ACTIVE + PASSWORD_LOCKED
SUSPENDED + RECOVERY_AVAILABLE
DEACTIVATED + REACTIVATABLE
DELETION_PENDING + RESTORABLE
DELETED + RETENTION_HOLD
RETIRED_MACHINE + AUDIT_RETAINED
sin convertir Auth en una colección de banderas contradictorias.

## 391. Posición dentro de la arquitectura 36–41

La secuencia queda:
36 Authentication Policy Engine
          │
          │ ¿Qué requisitos aplican?
          ▼
37 Authentication Assurance
          │
          │ ¿Cuánta confianza tenemos?
          ▼
38 Authentication Challenge
          │
          │ ¿Cómo obtenemos más evidencia?
          ▼
39 Authentication Transaction Security
          │
          │ ¿La ceremonia es íntegra y fresca?
          ▼
40 Authentication Security Incident
          │
          │ ¿Qué hacemos ante compromiso?
          ▼
41 Identity Lifecycle
          │
          │ ¿Puede esta identidad seguir existiendo
          │ y participar en Authentication?
          ▼
      Authentication Eligibility
Esto crea una separación importante:
POLICY
   ↓
ASSURANCE
   ↓
CHALLENGE
   ↓
TRANSACTION SECURITY
   ↓
INCIDENT RESPONSE
   ↓
IDENTITY LIFECYCLE
sin mezclar estas responsabilidades dentro de Authenticator, Guard, UserProvider o AuthManager.

## 392. Siguiente documento recomendado

Siguiendo la etapa transversal del sistema, el siguiente documento debería ser:
42_AUTHENTICATION_PRIVACY_DATA_MINIMIZATION_RETENTION_CONSENT_AND_SECURITY_METADATA_GOVERNANCE_SYSTEM.md
El 41 ya introdujo problemas que hacen necesario formalizarlo:
IP addresses
device metadata
approximate locations
authentication history
security events
incident evidence
federated identity metadata
credential metadata
deleted identity tombstones
retention holds
audit records
risk signals
El 42 deberá definir claramente:
Authentication Data Classification
            ↓
Collection Policy
            ↓
Purpose Limitation
            ↓
Data Minimization
            ↓
Privacy Controls
            ↓
Retention Policy
            ↓
Anonymization / Pseudonymization
            ↓
Erasure
            ↓
Legal / Security Retention
            ↓
Governance
y, sobre todo, resolver una tensión fundamental del sistema de Authentication:
Conservar suficiente información para seguridad, fraude, auditoría e incident response sin convertir Authentication en un almacén indefinido de información personal y telemetría sensible.
