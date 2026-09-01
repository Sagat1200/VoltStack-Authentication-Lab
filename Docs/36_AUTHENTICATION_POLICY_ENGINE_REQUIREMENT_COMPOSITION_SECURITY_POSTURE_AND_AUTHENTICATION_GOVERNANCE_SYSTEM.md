# VoltStack Authentication System

## 36 — Authentication Policy Engine, Requirement Composition, Security Posture and Authentication Governance System

- **Archivo:** `36_AUTHENTICATION_POLICY_ENGINE_REQUIREMENT_COMPOSITION_SECURITY_POSTURE_AND_AUTHENTICATION_GOVERNANCE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Nivel:** Core / Security-Critical / Governance
- **Dependencias principales:** 01–35, especialmente 05, 06, 08, 10, 11, 12, 14, 15, 16, 17, 19, 20, 21, 29, 31, 32, 33, 34 y 35.

---

## 1. Propósito

Este documento define el sistema central de gobierno de Authentication de VoltStack.
Hasta este punto, Authentication contiene múltiples políticas especializadas:

- Password Policy
- Session Policy
- Remember-Me Policy
- Token Policy
- MFA Policy
- Passkey Policy
- Federation Policy
- Recovery Policy
- Throttling Policy
- Risk Policy
- Device Trust Policy
- Tenant Authentication Policy
- Cryptographic Policy
- Privileged Authentication Policy
- Machine Authentication Policy
- Credential Management Policy
- Security Center Policy

Estas políticas no deberán evolucionar como sistemas independientes sin coordinación.
VoltStack deberá proporcionar un:
AUTHENTICATION POLICY ENGINE
capaz de determinar, para una operación concreta:

```text
WHO
is authenticating

WHERE
the authentication occurs

WHAT
```

operation is being requested

WITH WHAT
authentication evidence

UNDER WHICH
tenant / realm / application / security context

WITH WHICH
risk signals

AND WHAT
authentication assurance is required
para producir una decisión coherente.

## 2. Objetivo fundamental

El sistema deberá responder una pregunta central:
¿Qué debe demostrar este principal, en este contexto concreto, para que VoltStack considere suficiente su autenticación?

No simplemente:
Is user logged in?
sino:
What authentication assurance is required here?

## 3. Problema arquitectónico

Sin un Policy Engine central pueden aparecer reglas como:

```php
if ($tenant->requiresMfa()) {
    // ...
}

if ($user->isAdmin()) {
    // ...
}

if ($risk > 70) {
    // ...
}

if ($device->isUnknown()) {
    // ...
}

if ($route->requiresPasskey()) {
    // ...
}
```

distribuidas entre:

- Controllers
- Middleware
- Authenticators
- Listeners
- Services
- Routes
- Security Center
- Applications

Esto produce:

- duplicación
- inconsistencia
- policy drift
- bypass
- dificultad de auditoría
- dificultad de testing

VoltStack deberá centralizar la composición.

## 4. Principio arquitectónico

Los mecanismos de Authentication producen evidencia; las políticas determinan qué evidencia es necesaria.

Por tanto:

- Authenticator
- ≠
- Policy Engine

## 5. Ejemplo

Un Passkey Authenticator puede demostrar:

- passkey
- user verification
- phishing resistance

pero no decidir:

- "Todos los administradores del Tenant A deben usar passkey."
- Eso pertenece al Policy Engine.

## 6. Segundo principio

Authentication Policy determina requisitos; Authorization determina permisos.

## 1. Authentication vs Authorization

Authentication responde:

- Who are you?
- How strongly have you proven it?

Is that proof acceptable now?
Authorization responde:
Are you allowed to perform this operation?

## 2. Ejemplo

User
↓
Password authentication
↓
Authenticated
↓
Requests: Delete Production Database
↓
Authentication Policy:

```text
Passkey + recent authentication required
   ↓
Step-Up
   ↓
```

Authorization:

- Does user have database.delete permission?
- Ambos sistemas participan, pero no son iguales.

## 3. Arquitectura general

AUTHENTICATION REQUEST
│
▼
Authentication Context
│
┌────────────────┼─────────────────┐
▼                ▼                 ▼
Principal          Resource          Operation
│                │                 │
└────────────────┼─────────────────┘
▼
POLICY ENGINE
│
┌───────────────────┼───────────────────┐
▼                   ▼                   ▼
Platform Policy       Tenant Policy       Realm Policy
│                   │                   │
├──────────────┬────┴─────┬─────────────┤
▼              ▼          ▼             ▼
Identity Policy   Risk Policy Device Policy Operation Policy
│              │          │             │
└──────────────┴─────┬────┴─────────────┘
▼
REQUIREMENT COMPOSITION
│
▼
EFFECTIVE AUTH REQUIREMENT
│
┌─────────┴─────────┐
▼                   ▼
Evidence             Assurance
Available            Required
│                   │
└─────────┬─────────┘
▼
POLICY DECISION

## 4. Core abstraction

interface AuthenticationPolicyEngineInterface
{
public function evaluate(
AuthenticationPolicyContext $context
): AuthenticationPolicyDecision;
}

## 5. AuthenticationPolicyContext

Representará todo el contexto relevante para una decisión.

```php
final readonly class AuthenticationPolicyContext
{
    public function __construct(
        public ?PrincipalReference $principal,
        public AuthenticationOperation $operation,
        public AuthenticationEvidenceSet $evidence,
        public AuthenticationSecurityContext $security,
        public AuthenticationPolicyScope $scope,
        public RiskAssessment $risk,
    ) {}
}
```

## 6. Context debe ser explícito

Evitar que policies lean arbitrariamente:

- globals
- current request singleton
- static Auth state
- environment mutable

## 7. Context immutability

AuthenticationPolicyContext deberá ser:

- immutable
- request/fiber scoped
- serializable when safe
- secret-free

## 8. Policy Sources

VoltStack podrá obtener reglas desde:

- Framework Security Floor
- Platform
- Application
- Realm
- Tenant
- Organization
- Identity Classification
- Operation
- Resource Classification
- Risk Engine
- Device Trust
- Authentication Method

## 9. Policy Hierarchy

Conceptualmente:

```text
Framework Security Floor
          │
          ▼
Platform Policy
          │
          ▼
Application Policy
          │
          ▼
Realm Policy
          │
          ▼
Tenant Policy
          │
          ▼
```

Identity / Principal Policy
│
▼
Resource / Operation Policy
│
▼
Risk Adaptive Policy
│
▼
Effective Authentication Requirement

## 10. No simple override

La jerarquía no deberá significar:
child policy replaces parent policy
porque una policy inferior podría debilitar accidentalmente una garantía superior.

## 11. Security Floor

VoltStack deberá definir:

- minimum security requirements
- que policies inferiores no puedan debilitar.

## 12. Ejemplo

Framework:
Privileged operations require fresh authentication.
Tenant intenta:
No reauthentication required.
Resultado:
Framework floor wins.

## 13. Policy Composition

La composición deberá ser:

- monotonic by default
- respecto a seguridad.

## 14. Monotonic Security Composition

Agregar una policy más restrictiva:

- may strengthen requirements
- pero no reducir automáticamente requisitos existentes.

## 15. Example

Platform:
MFA required

Tenant:
Passkey required

Risk:
Fresh authentication required
Resultado:

- MFA
- +
- Passkey
- +
- Fresh Authentication

si son requisitos compatibles.

## 22. Policy Types

VoltStack deberá distinguir:

- REQUIREMENT_POLICY
- ELIGIBILITY_POLICY
- METHOD_POLICY
- RISK_POLICY
- SESSION_POLICY
- DEVICE_POLICY
- CREDENTIAL_POLICY
- RECOVERY_POLICY
- GOVERNANCE_POLICY

## 23. Requirement Policy

Determina qué debe demostrarse.
Ejemplo:
MFA required

## 24. Eligibility Policy

Determina si Authentication puede proceder.

```text
Ejemplo:
Account suspended
→ authentication prohibited
```

## 25. Method Policy

Determina métodos permitidos/prohibidos.
Ejemplo:
SMS OTP prohibited for privileged authentication

## 26. Risk Policy

Transforma risk assessment en requisitos.

```text
Ejemplo:
HIGH RISK
→ phishing-resistant MFA required
```

## 27. Session Policy

Determina propiedades de sesión.
Ejemplo:
Privileged session max age = 15 minutes

## 28. Device Policy

Ejemplo:
Managed corporate device required

## 29. Credential Policy

Ejemplo:
Password credential must satisfy current hashing policy

## 30. Recovery Policy

Ejemplo:
Recovery-authenticated session cannot perform credential export.

## 31. Governance Policy

Controla cómo pueden definirse, aprobarse y cambiarse otras policies.

## 32. AuthenticationRequirement

Abstracción central:

```php
interface AuthenticationRequirementInterface
{
    public function type(): string;
}
```

## 33. Requirement Types

VoltStack podrá incluir:

- IdentityRequirement
- AuthenticationMethodRequirement
- FactorRequirement
- MfaRequirement
- PasskeyRequirement
- PhishingResistanceRequirement
- UserVerificationRequirement
- FreshAuthenticationRequirement
- DeviceTrustRequirement
- ManagedDeviceRequirement
- NetworkRequirement
- CertificateRequirement
- FederationRequirement
- RecoveryRestrictionRequirement
- RiskRequirement
- AssuranceRequirement

## 34. Requirement Set

final readonly class AuthenticationRequirementSet
{
public function __construct(
public array $requirements
) {}
}

## 35. Requirement Composition Engine

interface AuthenticationRequirementComposerInterface
{
public function compose(
iterable $requirements,
AuthenticationPolicyContext $context
): EffectiveAuthenticationRequirement;
}

## 36. EffectiveAuthenticationRequirement

Será el resultado normalizado de todas las políticas aplicables.

## 37. Ejemplo

Entrada:

```text
Tenant:
MFA
```

Operation:
Fresh Auth <= 5 minutes

Risk:
Passkey

Device:
Trusted Device
Salida:

```text
EffectiveAuthenticationRequirement
├── MFA
├── Passkey
├── Freshness <= 5 min
└── Trusted Device
```

## 38. Requirement Normalization

El Composer deberá eliminar:

- duplicates
- redundancies
- weaker superseded requirements

## 39. Ejemplo

Fresh Auth <= 30 min
Fresh Auth <= 10 min
Fresh Auth <= 5 min
Resultado:
Fresh Auth <= 5 min

## 40. Requirement Strength

Algunos requisitos deberán tener relaciones de fuerza.

## 41. Example

Conceptualmente:

```text
Any MFA
    <
Phishing-resistant MFA
    <
Hardware-backed phishing-resistant MFA
```

cuando la policy así lo defina.

## 42. No Universal Ordering

No todos los métodos pueden ordenarse linealmente.

- Ejemplo:
- Client Certificate
- Passkey
- Enterprise Federation

pueden tener propiedades distintas.

## 43. Capability-Based Requirements

Por ello, preferir:

- requires phishing resistance
- requires user verification
- requires hardware backing

sobre:
method X > method Y

## 44. Authentication Capability

enum AuthenticationCapability: string
{
case MultiFactor = 'multi_factor';

case PhishingResistant = 'phishing_resistant';

case UserVerification = 'user_verification';

case HardwareBacked = 'hardware_backed';

case DeviceBound = 'device_bound';

case FederationBacked = 'federation_backed';
}

## 45. Evidence Capabilities

Cada evidencia podrá declarar propiedades verificadas.

## 46. Example

Passkey:

- PHISHING_RESISTANT
- USER_VERIFICATION
- DEVICE_BOUND?
- HARDWARE_BACKED?

No asumir propiedades no demostradas.

## 47. Requirement Satisfaction

interface AuthenticationRequirementSatisfactionEvaluatorInterface
{
public function evaluate(
EffectiveAuthenticationRequirement $requirement,
AuthenticationEvidenceSet $evidence,
AuthenticationPolicyContext $context
): AuthenticationRequirementSatisfaction;
}

## 48. Satisfaction States

SATISFIED
PARTIALLY_SATISFIED
UNSATISFIED
IMPOSSIBLE
INDETERMINATE

## 49. SATISFIED

Toda evidencia necesaria ya existe.

## 50. PARTIALLY_SATISFIED

Ejemplo:

- Password completed
- MFA still required

## 51. UNSATISFIED

Falta evidencia pero puede obtenerse.

## 52. IMPOSSIBLE

No existe un método permitido disponible.

- Ejemplo:
- Passkey required
- +

Identity has no passkeys
+
Enrollment prohibited in this flow

## 53. INDETERMINATE

Dependencia externa no puede evaluarse.
Ejemplo:
Device compliance provider unavailable

## 54. Policy Decision

final readonly class AuthenticationPolicyDecision
{
public function __construct(
public AuthenticationPolicyDecisionType $type,
public EffectiveAuthenticationRequirement $requirement,
public AuthenticationRequirementSatisfaction $satisfaction,
public AuthenticationDecisionExplanation $explanation,
) {}
}

## 55. Decision Types

ALLOW
CHALLENGE
DENY
RESTRICT
DEFER
ERROR

## 56. ALLOW

Authentication actual satisface requirement.

## 57. CHALLENGE

Se requiere evidencia adicional.

## 58. DENY

Authentication no puede proceder bajo la policy actual.

## 59. RESTRICT

Principal puede autenticarse, pero bajo un estado restringido.
Muy importante para:

- Recovery
- Compromise
- Expired credential
- Security Review

## 60. DEFER

La decisión depende de otro componente.
Debe usarse cuidadosamente.

## 61. ERROR

Policy no pudo evaluarse correctamente.
No equivale a DENY internamente, aunque externamente pueda producir fail-closed.

## 62. Fail Closed

Para decisiones security-critical:

```text
policy evaluation failure
→ do not silently allow
```

## 63. Failure Strategy

Debe configurarse por policy class:

- FAIL_CLOSED
- FAIL_RESTRICTED
- FAIL_DEFERRED

Nunca:

- FAIL_OPEN
- por defecto en seguridad crítica.

## 64. Authentication Operation

Policies deberán evaluar operaciones semánticas.

## 65. Examples

SIGN_IN
CREATE_SESSION
REFRESH_SESSION
ISSUE_TOKEN
LINK_AUTH_METHOD
REMOVE_AUTH_METHOD
CHANGE_PASSWORD
ADD_PASSKEY
REMOVE_PASSKEY
TRUST_DEVICE
RECOVER_ACCOUNT
VIEW_SECURITY_CENTER
REVOKE_SESSION
LOGOUT_EVERYWHERE
PERFORM_PRIVILEGED_OPERATION

## 66. Typed Operations

Evitar strings arbitrarios:

- 'delete-important-stuff'
- Preferir objetos/enums registrados.

## 67. Operation Metadata

Puede incluir:

- sensitivity
- privilege
- data classification
- destructive
- financial
- administrative

## 68. Operation Security Classification

NORMAL
SENSITIVE
PRIVILEGED
CRITICAL
BREAK_GLASS

## 69. Default Requirements

Ejemplo:

```php
NORMAL
→ authenticated session

SENSITIVE
→ fresh authentication

PRIVILEGED
→ strong MFA

CRITICAL
→ phishing-resistant fresh MFA

BREAK_GLASS
→ dedicated emergency policy
No hardcodear necesariamente estos valores; son policy defaults.
```

## 70. Authentication Freshness

Documento 32.
Debe ser un requisito first-class.

## 71. FreshnessRequirement

final readonly class FreshAuthenticationRequirement
{
public function __construct(
public DateInterval $maximumAge
) {}
}

## 72. Freshness Composition

La ventana más estricta gana.

## 73. Example

Platform: 30 minutes
Tenant: 15 minutes
Operation: 5 minutes
Resultado:
5 minutes

## 74. Method Freshness

Puede distinguirse:

- overall authentication freshness
- specific factor freshness

## 75. Example

Password authenticated 20 min ago
Passkey authenticated 2 min ago
Policy puede requerir:
fresh phishing-resistant factor <= 5 min

## 76. Authentication Evidence Age

Cada evidencia deberá conservar:

- issued_at
- verified_at
- method
- capabilities
- context

## 77. Security Posture

Documento 35 introdujo IdentitySecurityPosture.
Aquí se formaliza como entrada y salida de governance.

## 78. Security Posture Dimensions

No deberá limitarse a un único estado.

- Puede considerar:
- Identity State
- Credential State
- Session State
- Device State
- Risk State
- Recovery State
- Compromise State
- Compliance State

## 79. SecurityPostureSnapshot

final readonly class AuthenticationSecurityPosture
{
public function __construct(
public IdentityPosture $identity,
public CredentialPosture $credentials,
public SessionPosture $session,
public DevicePosture $device,
public RiskPosture $risk,
public RecoveryPosture $recovery,
) {}
}

## 80. Security Posture != Score

No reducir por defecto todo a:
72/100

## 81. Typed Posture

Preferir:

```php
identity = NORMAL
device = UNKNOWN
risk = ELEVATED
credential = HEALTHY
recovery = NORMAL
```

## 82. Effective Posture

Policy puede derivar:

- NORMAL
- ELEVATED
- RESTRICTED
- ACTION_REQUIRED
- COMPROMISED
- BLOCKED

## 83. Posture Transition

Ejemplo:

```text
NORMAL
  ↓ suspicious login
ELEVATED
  ↓ compromise confirmed
COMPROMISED
  ↓ restricted recovery
RESTRICTED
  ↓ security review complete
NORMAL
```

## 84. Posture as Policy Input

Ejemplo:

```text
COMPROMISED
→ password-only authentication prohibited
```

## 85. Posture as Policy Output

Policy también puede decidir:

- ALLOW authentication
- but mark session RESTRICTED

## 86. Restricted Authentication

Debe ser first-class.

## 87. Example

Después de recovery:

- Authenticated Identity
- +
- Restricted Session

puede:

- view security center
- change credentials
- review devices

pero no:

- transfer money
- create API token
- change tenant owner

Authorization integrará esta información.

## 88. Authentication Assurance

El documento 37 profundizará en assurance levels.
Este documento define el punto de integración.

## 89. Assurance Requirement

Policy podrá solicitar:

- minimum assurance
- sin especificar método exacto.

## 90. Example

Requirement:

- Assurance >= HIGH
- Podría satisfacerse mediante distintas combinaciones permitidas.

## 91. Method-specific Policy

Solo cuando exista razón real:
Passkey explicitly required

## 92. Prefer Capability Requirements

Siempre que sea posible:
phishing-resistant
es preferible a:
must use WebAuthn authenticator X

## 93. MFA Composition

MFA no deberá significar simplemente:
two authentication events

## 94. Factor Independence

Policy puede exigir:
independent factors

## 95. Example Invalid MFA

Password
+
Password again
No constituye MFA.

## 96. Factor Categories

Conceptualmente:

- KNOWLEDGE
- POSSESSION
- INHERENCE
- FEDERATED_ASSERTION
- DEVICE_ATTESTATION
- CERTIFICATE

## 97. Factor Semantics

El documento 15 seguirá siendo authority del MFA orchestration.
Policy Engine define requisitos.

## 98. Device Requirements

Policy puede solicitar:

- KNOWN_DEVICE
- TRUSTED_DEVICE
- MANAGED_DEVICE
- COMPLIANT_DEVICE
- DEVICE_BOUND_CREDENTIAL

## 99. Device Trust Is Not Binary

Documento 21.

- Ejemplo:
- UNKNOWN
- OBSERVED
- KNOWN
- TRUSTED
- MANAGED
- COMPROMISED

## 100. Device Policy Example

Tenant A:
administrators require MANAGED_DEVICE

Risk:
COMPROMISED_DEVICE → DENY

## 101. Network Context

Policy podrá considerar:

- network zone
- VPN status
- private network
- country
- region
- ASN risk
- cuando estén disponibles.

## 102. Network Is Weak Evidence

IP/network no deberá tratarse automáticamente como Identity proof.

## 103. Geo Policy

Puede existir:

- block authentication from prohibited jurisdictions
- según aplicación/compliance.

## 104. Geo Failures

Geolocation imprecisa debe considerarse.

- Policy puede:
- challenge
- restrict
- deny
- según criticidad.

## 105. Risk Integration

Documento 20.
Risk Engine produce:

- RiskAssessment
- Policy Engine consume ese resultado.

## 106. Separation

Risk Engine:
"This authentication has risk 0.82 / HIGH"

Policy Engine:
"HIGH risk requires phishing-resistant step-up"

## 107. Risk Does Not Directly Authenticate

Risk no sustituye evidencia.

## 108. Adaptive Authentication

Pipeline:

```text
Base Requirements
      │
      ▼
Risk Assessment
      │
      ▼
Adaptive Policy
      │
      ▼
Stronger Requirements
```

## 109. Risk Cannot Weaken Floor

Un risk score bajo no debe automáticamente eliminar un security floor obligatorio.

## 110. Example

Platform:
Admins always require MFA.
Risk:
LOW
Resultado sigue siendo:
MFA required.

## 111. Trusted Device Cannot Bypass Mandatory MFA

Salvo policy explícita que permita equivalencia y dentro del security floor.

## 112. Policy Provenance

Cada requisito deberá saber de dónde provino.

## 113. RequirementProvenance

final readonly class AuthenticationRequirementProvenance
{
public function __construct(
public string $policyId,
public AuthenticationPolicyScope $scope,
public string $reasonCode,
) {}
}

## 114. Why Provenance Matters

Permite explicar:
Why is passkey required?
Respuesta interna:

- Tenant Policy T-34
- +
- Operation CRITICAL
- +
- Risk HIGH

## 115. Explainability

Documento 24.
Policy Engine deberá producir explicación estructurada.

## 116. AuthenticationDecisionExplanation

No necesariamente texto humano.

- Preferir:
- reason codes
- policy references
- requirement provenance

## 117. Example

{
"decision": "challenge",
"reasons": [
"AUTH_TENANT_MFA_REQUIRED",
"AUTH_HIGH_RISK_PHISHING_RESISTANCE_REQUIRED"
]
}

## 118. Public Explanation

No deberá revelar:

- exact fraud threshold
- internal risk model
- secret policy exceptions

## 119. Internal Explanation

Audit puede contener mayor detalle.

## 120. Policy IDs

Cada policy deberá tener identificador estable.

- Ejemplo:
- auth.platform.minimum_mfa
- auth.tenant.admin_passkey
- auth.operation.payment.fresh_auth

## 121. Policy Version

Toda policy deberá poder versionarse.

## 122. Why

Para responder:

- Which policy produced this decision?
- en un incidente meses después.

## 123. Policy Definition

Conceptualmente:

```php
final readonly class AuthenticationPolicyDefinition
{
    public function __construct(
        public AuthenticationPolicyId $id,
        public AuthenticationPolicyVersion $version,
        public AuthenticationPolicyScope $scope,
        public AuthenticationPolicyPriority $priority,
        public AuthenticationPolicyCondition $condition,
        public AuthenticationPolicyEffect $effect,
    ) {}
}
```

## 124. Policy Registry

interface AuthenticationPolicyRegistryInterface
{
public function policiesFor(
AuthenticationPolicyContext $context
): iterable;
}

## 125. Registry Sources

Puede cargar policies desde:

- compiled configuration
- PHP providers
- tenant configuration
- database
- remote governance service
- plugins

## 126. Runtime Database Policies

Permitidas, pero deberán validarse antes de activarse.

## 127. Policy Compilation

Documento 27.
Policies estáticas podrán compilarse.

## 128. Compiled Policy Graph

Configuration
↓
Validation
↓
Normalization
↓
Policy Graph
↓
Compilation
↓
Optimized Runtime Representation

## 129. Policy Graph

Permitirá detectar:

- conflicts
- cycles
- impossible requirements
- shadowed rules
- redundancies

## 130. Policy Conflict

Ejemplo:

```text
Policy A:
Passkey required
```

Policy B:

- Passkey prohibited
- para el mismo contexto.

Esto no debe resolverse silenciosamente.

## 131. Conflict Resolver

interface AuthenticationPolicyConflictResolverInterface
{
public function resolve(
AuthenticationPolicyConflict $conflict
): AuthenticationPolicyConflictResolution;
}

## 132. Default Conflict Strategy

Security-critical conflict:
FAIL_CLOSED
y producir:

- configuration error
- audit
- observability signal

## 133. Explicit Precedence

Algunos conflictos pueden resolverse mediante:

- security floor
- scope
- priority
- specificity
- explicit override authorization

## 134. Priority

No deberá convertirse en:
highest integer wins everything

## 135. Security Semantics First

La composición semántica deberá tener prioridad sobre orden arbitrario.

## 136. Policy Specificity

Ejemplo:

```text
Platform:
MFA required for privileged operations
```

Tenant:
Passkey required for privileged operations
No conflicto:

- Passkey may satisfy/strengthen MFA
- según capabilities.

## 137. Impossible Policy Detection

Ejemplo:

- Require hardware-backed passkey
- +

No registered hardware-backed credentials
+
Enrollment forbidden
Puede detectarse como:
IMPOSSIBLE

## 138. Lockout Prevention

Governance deberá detectar policies capaces de bloquear a todos los administradores.

## 139. Policy Safety Validation

Antes de activar una policy:

- syntax validation
- semantic validation
- security floor validation
- lockout analysis
- method availability analysis
- tenant scope validation
- break-glass validation

## 140. Break-Glass Governance

Documento 32.
Critical policy changes deberán preservar emergency access controlado.

## 141. Break-Glass Is Not Policy Bypass

Debe tener policy propia:

- dedicated identities
- strong authentication
- limited duration
- mandatory audit
- reason required
- alerting

## 142. Policy Change Authorization

Modificar Authentication Policy es una operación privilegiada.

## 143. Authentication Policy Administration

No confundir con autenticación del usuario.

- Authorization System decidirá quién puede:
- create
- edit
- approve
- activate
- deactivate
- rollback
- policies.

## 144. Policy Lifecycle

DRAFT
↓
VALIDATED
↓
REVIEWED
↓
APPROVED
↓
ACTIVE
↓
SUPERSEDED
↓
RETIRED

## 145. Optional Emergency State

EMERGENCY_DISABLED
solo mediante proceso privilegiado.

## 146. Draft Policy

No afecta runtime.

## 147. Validated Policy

Pasó validaciones sintácticas/semánticas.

## 148. Approved Policy

Autorizada para activación.

## 149. Active Policy

Participa en decisiones.

## 150. Superseded

Sustituida por versión posterior.

## 151. Retired

Conservada para audit/history.

## 152. Policy Approval

Enterprise deployments podrán requerir:
four-eyes approval

## 153. Separation of Duties

Ejemplo:

- Author
- ≠
- Approver
- para policies críticas.

## 154. Policy Change Set

final readonly class AuthenticationPolicyChangeSet
{
public function __construct(
public array $changes,
public AuthenticationPolicyChangeReason $reason,
) {}
}

## 155. Dry Run

VoltStack deberá soportar evaluar una policy antes de activarla.

## 156. Policy Simulation

interface AuthenticationPolicySimulatorInterface
{
public function simulate(
AuthenticationPolicyDefinition $candidate,
iterable $scenarios
): AuthenticationPolicySimulationReport;
}

## 157. Simulation Scenarios

Ejemplos:

- normal user login
- admin login
- new device
- high-risk login
- account recovery
- machine authentication
- tenant administrator
- break-glass access

## 158. Simulation Result

Podrá detectar:

- new challenges
- new denials
- lockouts
- weakened requirements
- unreachable flows

## 159. Shadow Mode

Nueva policy podrá ejecutarse sin enforcement.
SHADOW

## 160. Shadow Policy

Produce:

- would_allow
- would_challenge
- would_deny

pero no cambia decisión.

## 161. Shadow Use Cases

Útil para:

- MFA rollout
- Passkey rollout
- new risk thresholds
- device compliance

## 162. Shadow Security

No deberá utilizarse para políticas necesarias para corregir vulnerabilidad crítica si enforcement inmediato es necesario.

## 163. Policy Rollout

Podrá soportar:

- percentage rollout
- tenant rollout
- organization rollout
- identity cohort rollout

## 164. Rollout Must Be Deterministic

Un principal no deberá saltar aleatoriamente entre policies dentro de una misma operación.

## 165. Policy Snapshot

Cada Authentication Transaction deberá poder fijar:

- policy snapshot/version
- para mantener coherencia durante el flujo.

## 166. Why

Si una policy cambia entre:
password step
y:

- MFA step
- el flujo podría volverse inconsistente.

## 167. Transaction Policy Snapshot

Documento 39 profundizará esto.

## 168. Long-Running Flows

Policy podrá exigir re-evaluation si:

- flow exceeds maximum age
- critical policy changed
- risk materially changed
- identity state changed

## 169. Policy Re-Evaluation

No asumir que una decisión inicial permanece válida indefinidamente.

## 170. Re-Evaluation Triggers

risk change
device change
identity suspension
credential revocation
tenant policy version change
security epoch change
privilege escalation
operation change

## 171. Session Policy Binding

Una sesión podrá registrar:

- authentication policy version
- assurance achieved
- methods used

security posture at issuance

## 172. Session Continuation

En cada request no será necesario repetir toda Authentication.
Pero policies pueden decidir si la sesión sigue siendo suficiente.

## 173. Session Sufficiency Evaluation

interface SessionAuthenticationSufficiencyEvaluatorInterface
{
public function evaluate(
AuthenticationSession $session,
EffectiveAuthenticationRequirement $requirement,
AuthenticationPolicyContext $context
): AuthenticationRequirementSatisfaction;
}

## 174. Example

Session:

- Password + TOTP
- authenticated 2 hours ago

Operation:
Delete tenant
Policy:

- Passkey
- fresh <= 5 min

Resultado:
CHALLENGE

## 175. Token Policy Binding

API token podrá tener:

- issued assurance
- credential type
- client identity
- policy version

## 176. Token Cannot Magically Upgrade

Token emitido bajo assurance bajo no deberá satisfacer una operación que exige assurance mayor sin mecanismo explícito.

## 177. Stateless Authentication

Documento 14.
Policy Engine deberá funcionar también sin Session.

## 178. Machine Authentication

Documento 33.

- Policy deberá soportar:
- human principal
- service principal
- workload principal
- device principal

## 179. Principal Type Policy

Ejemplo:

```text
Human:
Passkey allowed
```

Machine:

- Passkey invalid
- Certificate/workload identity required

## 180. Workload Policy

Puede requerir:

- mTLS
- short-lived token
- workload attestation
- trusted issuer
- specific audience

## 181. Machine Freshness

Puede significar:

- token age
- certificate validity
- attestation freshness

en lugar de interactive reauthentication.

## 182. Federation Policy

Documento 17.

- Policy podrá restringir:
- allowed identity providers
- required issuer
- required claims
- required authentication context
- required federation assurance

## 183. Example

Tenant Enterprise:
Only corporate OIDC provider allowed.

## 184. Federation Claim Trust

Policy no deberá confiar en claims no validados por federation subsystem.

## 185. Authentication Context Mapping

External IdP:

- acr
- amr

podrán mapearse a capabilities internas.
El documento 37 profundizará esto.

## 186. Password Policy Integration

Documento 11.
Policy Engine no reemplaza hashing/password lifecycle.

- Puede determinar:
- password method allowed?
- password expired?
- password reauthentication sufficient?

## 187. Passkey Policy Integration

Documento 16.

- Puede exigir:
- user verification
- resident credential
- attestation class
- hardware-backed

cuando esté soportado y justificado.

## 188. Recovery Policy Integration

Documento 18.
Recovery evidence podrá tener menor assurance.

## 189. Example

Recovered account
→ authenticated
→ restricted posture
→ credential review required

## 190. Security Center Integration

Documento 35.

- Security Center podrá consultar:
- effective requirements
- security posture
- recommendations
- action requirements

## 191. Example

Botón:
Remove Last Passkey
Policy Engine:
DENY
reason:
would leave identity without required phishing-resistant method

## 192. Credential Enrollment Governance

Policy deberá gobernar no solo login sino enrollment.

## 193. Enrollment Operations

ENROLL_PASSWORD
ENROLL_PASSKEY
ENROLL_MFA
LINK_IDENTITY_PROVIDER
REGISTER_DEVICE
ISSUE_API_CREDENTIAL

## 194. Example

Tenant:

- SMS enrollment prohibited
- aunque SMS authenticator exista.

## 195. Credential Removal Governance

También:

- REMOVE_PASSWORD
- REMOVE_PASSKEY
- REMOVE_MFA
- UNLINK_PROVIDER
- DELETE_RECOVERY_METHOD

## 196. Minimum Credential Coverage

Policy puede exigir:

- at least one recovery-capable method
- at least two passkeys

at least one phishing-resistant method

## 197. Avoid Credential Dead Ends

No permitir:

- remove final authentication method
- si deja Identity inutilizable, salvo workflows explícitos.

## 198. Authentication Governance

Governance comprende:

- definition
- validation
- approval
- deployment
- versioning
- simulation
- monitoring
- rollback
- retirement
- audit

## 199. Governance Plane vs Runtime Plane

Separar:
GOVERNANCE PLANE
de:
AUTHENTICATION RUNTIME PLANE

## 200. Governance Plane

Policy Authoring
Policy Validation
Policy Approval
Policy Simulation
Policy Deployment
Policy Audit

## 201. Runtime Plane

Policy Lookup
Context Evaluation
Requirement Composition
Evidence Satisfaction
Decision

## 202. Architecture

GOVERNANCE PLANE

Policy Author
│
▼
Policy Definition
│
▼
Validation
│
▼
Simulation
│
▼
Approval
│
▼
Compilation
│
▼
Versioned Policy Artifact
│
▼
Deployment
│
▼
───────────────────────────────────────
RUNTIME PLANE
│
▼
Policy Registry
│
▼
Policy Engine
│
▼
Requirement Composer
│
▼
Satisfaction Evaluator
│
▼
Authentication Decision

## 203. Runtime Must Not Depend on Authoring DB

Idealmente policies compiladas deberán poder ejecutarse aunque:

- policy administration database
- esté temporalmente indisponible.

## 204. Deployment Artifact

Podrá existir:
AuthenticationPolicyBundle

## 205. AuthenticationPolicyBundle

Contiene:

- bundle version
- policies
- compiled graph
- security floor version
- signature/checksum
- created_at

## 206. Integrity

Policy artifacts deberán protegerse contra modificación no autorizada.

## 207. Policy Signing

En deployments de alta seguridad podrá utilizarse:
signed policy bundles

## 208. Cryptographic Integration

Documento 31.

## 209. Policy Rollback

Debe poder volver a una versión previa válida.

## 210. Rollback Security

No permitir rollback a una policy conocida como vulnerable sin autorización especial.

## 211. Emergency Policy Deployment

Puede existir un canal privilegiado para:

- immediate credential revocation requirement
- disable compromised authenticator
- force MFA

## 212. Emergency Changes

Deben:

- audit
- notify
- version
- expire/review

## 213. Policy Expiration

Policies temporales podrán tener:

- valid_from
- valid_until

## 214. Time-Based Policies

Ejemplo:

- During incident:
- require passkey for all admins.

## 215. Clock Source

Utilizar clock abstraction confiable.
No:

```php
new DateTime()
distribuido arbitrariamente por policies.
```

## 216. Policy Conditions

Podrán considerar:

- principal type
- identity attributes
- tenant
- realm
- operation
- resource classification
- device posture
- network context
- risk
- time
- authentication evidence
- session properties

## 217. Attribute Safety

No todas las attributes deberán ser confiables.

## 218. Trusted Attribute Sources

Policy debe saber procedencia:

- verified identity attribute
- tenant configuration
- validated IdP claim
- untrusted request metadata

## 219. Untrusted Metadata

No permitir:

- HTTP header says admin=true
- como policy input confiable.

## 220. Policy Data Classification

Inputs podrán clasificarse:

- TRUSTED
- VERIFIED
- DERIVED
- UNTRUSTED

## 221. Policy DSL

VoltStack podría proporcionar una DSL declarativa.

```php
Ejemplo conceptual:
AuthPolicy::for('tenant.admin')
    ->when(Operation::Privileged)
    ->require(
        MFA::phishingResistant(),
        FreshAuth::within(minutes: 5),
    );
```

## 222. Declarative Preferred

Preferir:

- policy declarations
- sobre callbacks arbitrarios cuando sea posible.

## 223. Custom Policies

Debe permitirse PHP custom:

```php
interface AuthenticationPolicyInterface
{
    public function evaluate(
        AuthenticationPolicyContext $context
    ): AuthenticationPolicyEffect;
}
```

## 224. Custom Policy Sandbox

No necesariamente sandbox técnico, pero deberá tener contratos estrictos.

## 225. Custom Policy Side Effects

Policies deberán ser:
pure or effectively side-effect free

## 226. No Policy Side Effects

No:

- send email
- delete session
- write user record
- durante evaluación.

## 227. Why

Policy evaluation puede ejecutarse:

- multiple times
- simulation
- shadow mode
- retry
- distributed nodes

## 228. Decision Effects

Mutaciones posteriores se ejecutan fuera del Policy Engine.

## 229. Determinism

Con mismo:

- policy snapshot
- context
- evidence
- risk assessment
- clock instant

debería producirse la misma decisión.

## 230. External Dependencies

Policies que necesiten:

- device compliance
- external risk
- remote tenant policy

deberán recibir resultados resueltos previamente o mediante providers controlados.

## 231. Avoid Arbitrary Network Calls

No hacer:

- HTTP request inside every policy
- sin gobernanza.

## 232. Policy Evaluation Budget

Debe existir límite de:

- execution time
- number of policies
- external lookups
- memory

## 233. Performance

Hot path de Authentication no debe evaluar cientos de policies innecesarias.

## 234. Policy Index

Registry podrá indexar por:

- operation
- tenant
- realm
- principal type
- security classification

## 235. Compiled Decision Tree

Policies estáticas podrán transformarse en:
optimized decision tree

## 236. Short Circuit

Puede utilizarse cuando semánticamente sea seguro.

```text
Ejemplo:
Identity permanently blocked
→ DENY
```

sin evaluar recomendaciones posteriores.

## 237. No Unsafe Short Circuit

No omitir policies necesarias para:

- audit explanation
- security floor
- mandatory governance

## 238. Policy Cache

Puede cachearse:

- resolved applicable policy set
- compiled policy graph
- normalized requirements
- según contexto.

## 239. Cache Key

Debe incluir cuando corresponda:

- policy bundle version
- tenant
- realm
- operation
- principal classification
- security posture
- risk class

## 240. Dynamic Risk

No cachear una decisión final ignorando risk cambiante.

## 241. Dynamic Identity State

Tampoco ignorar:

- suspension
- credential revocation
- security epoch

## 242. Cache Invalidation

Cambios en:

- policy version
- tenant policy
- identity state
- security posture

deben invalidar resultados afectados.

## 243. FrankenPHP

Policy Engine puede ser long-lived si es:

- stateless
- immutable

## 244. Request State

Debe vivir en:

- AuthenticationPolicyContext
- no en propiedades mutables del singleton.

## 245. Incorrecto

class PolicyEngine
{
private ?User $currentUser;
}
en servicio singleton.

## 246. Correcto

$engine->evaluate($context);

## 247. Fiber Safety

Cada evaluación conserva su propio context.

## 248. Policy Bundle Hot Reload

FrankenPHP puede mantener bundle compilado.

```text
Cambio:
Bundle V12
    ↓ atomic swap
Bundle V13
```

## 249. Atomic Bundle Replacement

Requests existentes pueden terminar con V12.
Nuevos requests usan V13.

## 250. No Partial Reload

Nunca:

- half old policies
- half new policies
- en una evaluación.

## 251. Distributed Policy Consistency

Documento 30.
Nodes deberán conocer:
active policy bundle version

## 252. Version Drift

Observability deberá detectar:

```text
Node A → V15
Node B → V14
```

## 253. Critical Policy Rollout

Puede requerir barrier/coordination antes de aceptar determinadas operaciones.

## 254. Eventual Consistency

Aceptable para algunas policies no críticas.
No necesariamente para:
disable compromised authentication method

## 255. Security Epoch Integration

Documento 40 profundizará.

- Policy changes críticos pueden incrementar:
- Authentication Security Epoch
- para forzar re-evaluation.

## 256. Events

AuthenticationPolicyEvaluated
AuthenticationPolicyDecisionProduced
AuthenticationRequirementComposed
AuthenticationRequirementUnsatisfied

AuthenticationPolicyCreated
AuthenticationPolicyValidated
AuthenticationPolicyApproved
AuthenticationPolicyActivated
AuthenticationPolicySuperseded
AuthenticationPolicyRetired
AuthenticationPolicyRolledBack

AuthenticationPolicyConflictDetected
AuthenticationPolicySimulationCompleted
AuthenticationPolicyBundleDeployed
AuthenticationPolicyVersionDriftDetected

## 257. Event Volume

AuthenticationPolicyEvaluated puede ser muy frecuente.
Debe configurarse observability sampling.

## 258. Audit

Cambios de policy deberán registrar:

- actor
- policy id
- old version
- new version
- scope
- reason
- approval
- deployment
- timestamp

## 259. Runtime Decision Audit

Para decisiones sensibles:

- operation
- principal
- effective requirements
- decision
- policy versions
- risk class
- assurance achieved
- sin secrets.

## 260. Policy Diff

Governance deberá poder producir diferencias.
Ejemplo:

```text
V4:
MFA required
```

V5:
Phishing-resistant MFA required

## 261. Human Review

Diff debe ser comprensible antes de aprobación.

## 262. Observability Metrics

auth_policy_evaluation_total
auth_policy_decision_total
auth_policy_challenge_total
auth_policy_deny_total
auth_policy_restricted_total

auth_policy_conflict_total
auth_policy_evaluation_error_total

auth_policy_bundle_version
auth_policy_bundle_reload_total
auth_policy_bundle_drift_total

auth_policy_simulation_total
auth_policy_rollout_total

## 263. Labels

Cardinalidad controlada:

- decision
- operation_class
- policy_scope
- risk_class
- outcome

## 264. No Labels

No:

- user_id
- email
- session_id
- token_id

## 265. Tracing

Spans:

- auth.policy.evaluate
- auth.policy.resolve
- auth.policy.compose_requirements
- auth.policy.evaluate_satisfaction
- auth.policy.simulate
- auth.policy.compile

## 266. Explainability Trace

Puede registrar:

- number of applicable policies
- requirement count
- decision type
- bundle version

## 267. Security Logging

No registrar:

- password
- OTP
- token
- private claims
- secret credential

## 268. Failure Taxonomy

AUTH_POLICY_NOT_FOUND
AUTH_POLICY_INVALID
AUTH_POLICY_CONFLICT
AUTH_POLICY_EVALUATION_FAILED

AUTH_POLICY_BUNDLE_INVALID
AUTH_POLICY_BUNDLE_VERSION_MISMATCH
AUTH_POLICY_BUNDLE_INTEGRITY_FAILURE

AUTH_REQUIREMENT_UNSATISFIED
AUTH_REQUIREMENT_IMPOSSIBLE
AUTH_REQUIREMENT_CONFLICT

AUTH_POLICY_SECURITY_FLOOR_VIOLATION
AUTH_POLICY_LOCKOUT_RISK
AUTH_POLICY_APPROVAL_REQUIRED

AUTH_POLICY_CONTEXT_INVALID
AUTH_POLICY_INPUT_UNTRUSTED

AUTH_POLICY_VERSION_DRIFT

## 269. Public Failure Mapping

No revelar detalles internos innecesarios.
Externamente:

- Additional authentication required.
- Authentication method not permitted.
- Authentication unavailable.

## 270. Testing Strategy

Debe cubrir:

- unit
- composition
- property-based
- integration
- simulation
- distributed
- security
- performance
- FrankenPHP

## 271. Unit Tests

Cada policy individual.

## 272. Composition Tests

Verificar combinación de policies.

## 273. Monotonicity Tests

Agregar una policy restrictiva no reduce requirement accidentalmente.

## 274. Security Floor Tests

Tenant/application no pueden debilitar floor.

## 275. Freshness Tests

Ventana más estricta gana.

## 276. Capability Tests

Evidence satisface solo capabilities realmente demostradas.

## 277. MFA Tests

Dos evidencias del mismo factor no se cuentan incorrectamente como MFA.

## 278. Passkey Tests

Phishing resistance no se asume si evidence no la demuestra.

## 279. Risk Tests

Risk HIGH aumenta requirements según policy.

## 280. Low Risk Tests

LOW risk no elimina mandatory requirements.

## 281. Device Tests

Trusted device no sustituye automáticamente MFA.

## 282. Recovery Tests

Recovered session puede quedar restricted.

## 283. Compromise Tests

Compromised posture produce requisitos/restricciones correctos.

## 284. Session Sufficiency Tests

Session vieja produce step-up.

## 285. Token Tests

Token de assurance bajo no obtiene assurance alto.

## 286. Machine Tests

Machine principal no recibe interactive human requirements inválidos.

## 287. Federation Tests

External assurance se mapea correctamente.

## 288. Conflict Tests

Contradicciones se detectan.

## 289. Impossible Requirement Tests

Configuraciones imposibles no se silencian.

## 290. Lockout Tests

Policy deployment detecta bloqueo total de admins.

## 291. Break-Glass Tests

Emergency path permanece controlado y auditado.

## 292. Simulation Tests

Candidate policy produce expected impact report.

## 293. Shadow Mode Tests

Shadow policy no cambia enforcement.

## 294. Rollout Tests

Cohort selection determinista.

## 295. Version Tests

Decisión registra bundle correcto.

## 296. Hot Reload Tests

No existe mixed bundle durante evaluación.

## 297. Distributed Tests

Nodes convergen a active bundle.

## 298. Drift Tests

Version drift genera alerta/metric.

## 299. Cache Tests

No reutilizar decisiones con risk/posture incompatible.

## 300. Tenant Isolation Tests

Policy de Tenant A nunca afecta Tenant B salvo scope explícito.

## 301. Realm Tests

Realm policies permanecen aisladas.

## 302. FrankenPHP Tests

User/context anterior no sobrevive.

## 303. Fiber Tests

Evaluaciones concurrentes no comparten estado.

## 304. Side-Effect Tests

Policy evaluation no muta sistema.

## 305. Determinism Tests

Mismo input produce misma decisión.

## 306. Property-Based Tests

Especialmente útiles para:

- composition
- precedence
- monotonicity
- conflict detection

## 307. Fuzzing

Policy parser/DSL deberá fuzzearse.

## 308. Performance Tests

Medir:

- P50
- P95
- P99
- de evaluation.

## 309. Policy Count Tests

Evaluar comportamiento con:

- 10
- 100
- 1,000
- 10,000

policies registradas, asegurando indexación adecuada.

## 310. Security Invariants — Core

AUTH-POLICY-CORE-01
Authentication mechanisms producen evidencia; Policy Engine determina requisitos.
AUTH-POLICY-CORE-02
Authentication Policy y Authorization permanecen separados.

- AUTH-POLICY-CORE-03
- Toda decisión se evalúa sobre un contexto explícito.
- AUTH-POLICY-CORE-04

El contexto de una evaluación no vive en estado global mutable.
AUTH-POLICY-CORE-05
Una policy no realiza side effects durante evaluación.

## 311. Security Invariants — Composition

AUTH-POLICY-COMP-01
Security floor no puede ser debilitado por scopes inferiores.
AUTH-POLICY-COMP-02
La composición es monotónica por defecto respecto a requisitos de seguridad.
AUTH-POLICY-COMP-03
Requisitos redundantes se normalizan sin perder seguridad.

- AUTH-POLICY-COMP-04
- Cuando dos freshness requirements se combinan, prevalece el más estricto.
- AUTH-POLICY-COMP-05

Conflictos security-critical nunca se resuelven silenciosamente.

## 312. Security Invariants — Evidence

AUTH-POLICY-EVID-01
Solo capabilities verificadas pueden satisfacer requirements.
AUTH-POLICY-EVID-02
Un método no recibe propiedades de seguridad por nombre o tipo sin evidence correspondiente.
AUTH-POLICY-EVID-03
Dos evidencias equivalentes no se convierten automáticamente en MFA.
AUTH-POLICY-EVID-04
Evidence freshness se evalúa explícitamente.
AUTH-POLICY-EVID-05
Un token/session no puede incrementar su assurance sin nueva evidencia válida.

## 313. Security Invariants — Risk

AUTH-POLICY-RISK-01
Risk puede endurecer requirements.

- AUTH-POLICY-RISK-02
- Risk bajo no debilita security floor obligatorio.
- AUTH-POLICY-RISK-03

Risk score no sustituye Authentication Evidence.
AUTH-POLICY-RISK-04
Trusted Device no elimina mandatory requirements salvo policy explícita válida.

## 314. Security Invariants — Posture

AUTH-POLICY-POSTURE-01
Security Posture no se reduce obligatoriamente a un score.
AUTH-POLICY-POSTURE-02
Restricted Authentication es diferente de anonymous y de fully trusted authentication.
AUTH-POLICY-POSTURE-03
Compromised/Recovery state puede reducir las operaciones disponibles aun después de Authentication exitosa.
AUTH-POLICY-POSTURE-04
Posture transitions son auditables cuando afectan seguridad.

## 315. Security Invariants — Governance

AUTH-POLICY-GOV-01
Toda policy activa tiene ID y versión.

- AUTH-POLICY-GOV-02
- Cambios de policy son auditables.
- AUTH-POLICY-GOV-03

Policies críticas pasan validación antes de activación.

- AUTH-POLICY-GOV-04
- Governance puede detectar configuraciones capaces de provocar lockout.
- AUTH-POLICY-GOV-05

Policy rollback no debe convertirse en bypass de security fixes.
AUTH-POLICY-GOV-06
Emergency policy changes requieren governance explícita.

## 316. Security Invariants — Runtime

AUTH-POLICY-RT-01
Una evaluación utiliza una versión coherente del policy bundle.
AUTH-POLICY-RT-02
Nunca se mezclan parcialmente dos bundles durante una decisión.
AUTH-POLICY-RT-03
FrankenPHP no comparte Policy Context entre requests.

- AUTH-POLICY-RT-04
- Fiber executions permanecen aisladas.
- AUTH-POLICY-RT-05

Cache no ignora policy version ni security-relevant context.

## 317. Security Invariants — Multi-Tenant

AUTH-POLICY-TENANT-01
Tenant policy solo afecta su scope.

- AUTH-POLICY-TENANT-02
- Tenant policy no puede debilitar framework security floor.
- AUTH-POLICY-TENANT-03

Cross-tenant policies requieren scope explícito y autoridad administrativa.
AUTH-POLICY-TENANT-04
Policy caches incluyen Tenant/Realm cuando afectan la decisión.

## 318. Anti-Pattern

Controller decides MFA policy

## 319. Anti-Pattern

Authenticator decides tenant security policy

## 320. Anti-Pattern

Authorization permission
=

Authentication strength

## 321. Anti-Pattern

Risk LOW
=

skip mandatory MFA

## 322. Anti-Pattern

Trusted Device
=

automatically fully trusted identity

## 323. Anti-Pattern

Passkey
=

always hardware-backed

## 324. Anti-Pattern

two login attempts
=

MFA

## 325. Anti-Pattern

highest policy priority integer wins

## 326. Anti-Pattern

tenant override
=

can disable platform security floor

## 327. Anti-Pattern

policy conflict
=

choose first one

## 328. Anti-Pattern

policy exception
=

silent bypass

## 329. Anti-Pattern

policy evaluation
→ modifies database

## 330. Anti-Pattern

policy evaluation
→ sends notification

## 331. Anti-Pattern

policy callback
→ arbitrary HTTP requests

## 332. Anti-Pattern

cache decision by user ID only

## 333. Anti-Pattern

policy bundle reload
→ mutate live array gradually

## 334. Anti-Pattern

policy version change
→ sessions magically inherit higher assurance

## 335. Anti-Pattern

recovery completed
=

full security posture automatically

## 336. Anti-Pattern

policy score 100
=

authentication guaranteed secure

## 337. Componentes principales

AuthenticationPolicyEngine
AuthenticationPolicyRegistry
AuthenticationPolicyContext
AuthenticationPolicyDefinition
AuthenticationPolicyDecision

AuthenticationRequirement
AuthenticationRequirementSet
EffectiveAuthenticationRequirement
AuthenticationRequirementComposer
AuthenticationRequirementSatisfactionEvaluator

AuthenticationSecurityPosture
AuthenticationPolicyConflictResolver
AuthenticationDecisionExplanation

AuthenticationPolicyBundle
AuthenticationPolicySimulator
AuthenticationPolicyChangeSet

## 338. Governance Components

AuthenticationPolicyValidator
AuthenticationPolicyCompiler
AuthenticationPolicyApprover
AuthenticationPolicyDeploymentManager
AuthenticationPolicyRollbackManager
AuthenticationPolicyDiff
AuthenticationPolicySimulationReport

## 339. Runtime Components

CompiledAuthenticationPolicyRegistry
AuthenticationPolicyEvaluator
AuthenticationRequirementComposer
AuthenticationRequirementNormalizer
AuthenticationRequirementSatisfactionEvaluator
SessionAuthenticationSufficiencyEvaluator
AuthenticationPolicyRuntimeContext

## 340. Namespace sugerido

VoltStack\Quantum\Auth\Policy
VoltStack\Quantum\Auth\Policy\Contracts
VoltStack\Quantum\Auth\Policy\Context
VoltStack\Quantum\Auth\Policy\Definition
VoltStack\Quantum\Auth\Policy\Requirement
VoltStack\Quantum\Auth\Policy\Evidence
VoltStack\Quantum\Auth\Policy\Posture
VoltStack\Quantum\Auth\Policy\Decision
VoltStack\Quantum\Auth\Policy\Governance
VoltStack\Quantum\Auth\Policy\Compilation
VoltStack\Quantum\Auth\Policy\Simulation
VoltStack\Quantum\Auth\Policy\Runtime

## 341. Estructura sugerida

src/Quantum/Auth/Policy/
├── Contracts/
│   ├── AuthenticationPolicyEngineInterface.php
│   ├── AuthenticationPolicyInterface.php
│   ├── AuthenticationPolicyRegistryInterface.php
│   ├── AuthenticationRequirementComposerInterface.php
│   ├── AuthenticationRequirementSatisfactionEvaluatorInterface.php
│   ├── AuthenticationPolicyConflictResolverInterface.php
│   ├── AuthenticationPolicySimulatorInterface.php
│   └── SessionAuthenticationSufficiencyEvaluatorInterface.php
│
├── Context/
│   ├── AuthenticationPolicyContext.php
│   ├── AuthenticationPolicyScope.php
│   ├── AuthenticationOperation.php
│   └── AuthenticationOperationClassification.php
│
├── Definition/
│   ├── AuthenticationPolicyDefinition.php
│   ├── AuthenticationPolicyId.php
│   ├── AuthenticationPolicyVersion.php
│   ├── AuthenticationPolicyCondition.php
│   └── AuthenticationPolicyEffect.php
│
├── Requirement/
│   ├── AuthenticationRequirement.php
│   ├── AuthenticationRequirementSet.php
│   ├── EffectiveAuthenticationRequirement.php
│   ├── FreshAuthenticationRequirement.php
│   ├── MfaRequirement.php
│   ├── PasskeyRequirement.php
│   ├── CapabilityRequirement.php
│   ├── DeviceRequirement.php
│   └── AssuranceRequirement.php
│
├── Evidence/
│   ├── AuthenticationCapability.php
│   ├── AuthenticationRequirementSatisfaction.php
│   └── AuthenticationRequirementSatisfactionEvaluator.php
│
├── Posture/
│   ├── AuthenticationSecurityPosture.php
│   ├── IdentityPosture.php
│   ├── CredentialPosture.php
│   ├── SessionPosture.php
│   ├── DevicePosture.php
│   ├── RiskPosture.php
│   └── RecoveryPosture.php
│
├── Decision/
│   ├── AuthenticationPolicyDecision.php
│   ├── AuthenticationPolicyDecisionType.php
│   ├── AuthenticationDecisionExplanation.php
│   └── AuthenticationRequirementProvenance.php
│
├── Governance/
│   ├── AuthenticationPolicyValidator.php
│   ├── AuthenticationPolicyChangeSet.php
│   ├── AuthenticationPolicyLifecycle.php
│   ├── AuthenticationPolicyDiff.php
│   ├── AuthenticationPolicyApprover.php
│   ├── AuthenticationPolicyDeploymentManager.php
│   └── AuthenticationPolicyRollbackManager.php
│
├── Compilation/
│   ├── AuthenticationPolicyCompiler.php
│   ├── AuthenticationPolicyBundle.php
│   ├── CompiledAuthenticationPolicyRegistry.php
│   └── AuthenticationPolicyGraph.php
│
├── Simulation/
│   ├── AuthenticationPolicySimulator.php
│   ├── AuthenticationPolicySimulationScenario.php
│   └── AuthenticationPolicySimulationReport.php
│
└── Runtime/
├── AuthenticationPolicyEngine.php
├── AuthenticationRequirementComposer.php
├── AuthenticationRequirementNormalizer.php
├── SessionAuthenticationSufficiencyEvaluator.php
└── AuthenticationPolicyRuntimeResetter.php

## 342. Configuración conceptual

return [

'authentication' => [

'policy' => [

'security_floor' => true,

'fail_strategy' => 'closed',

'compilation' => true,

'simulation' => true,

'shadow_mode' => true,

'governance' => [
'versioning' => true,
'approval' => true,
'audit' => true,
],

'runtime' => [
'cache' => true,
'bundle_hot_reload' => true,
],

],

],

];

## 343. Developer Experience

Ejemplo declarativo:

```php
AuthPolicy::define('tenant.admin.authentication')
    ->forPrincipal('administrator')
    ->when(Operation::Privileged)
    ->require(
        Mfa::phishingResistant(),
        FreshAuth::within(minutes: 5),
    );
```

## 344. Tenant Policy

AuthPolicy::tenant($tenant)
->for(Operation::SignIn)
->require(
Mfa::required()
);

## 345. Risk-Adaptive Policy

AuthPolicy::whenRisk(RiskLevel::High)
->require(
Capability::phishingResistant()
);

## 346. Device Policy

AuthPolicy::for(Operation::Critical)
->require(
Device::managed()
);

## 347. Policy Evaluation

Conceptualmente:

```php
$decision = Auth::policy()->evaluate(
    $context
);
```

## 348. Decision

match ($decision->type) {
AuthenticationPolicyDecisionType::Allow =>
$this->continue(),

AuthenticationPolicyDecisionType::Challenge =>
$this->challenge($decision->requirement),

AuthenticationPolicyDecisionType::Restrict =>
$this->createRestrictedContext(),

AuthenticationPolicyDecisionType::Deny =>
$this->deny(),
};

## 349. Flujo completo

Authentication Request
│
▼
Resolve Principal / Tenant / Realm
│
▼
Build AuthenticationPolicyContext
│
▼
Resolve Applicable Policies
│
▼
Apply Framework Security Floor
│
▼
Compose Requirements
│
▼
Normalize Requirements
│
▼
Evaluate Risk / Posture
│
▼
Evaluate Available Evidence
│
├───────────────┬─────────────────┐
▼               ▼                 ▼
SATISFIED       UNSATISFIED        IMPOSSIBLE
│               │                 │
▼               ▼                 ▼
ALLOW          CHALLENGE           DENY
│
▼
Session / Token / Authentication Context

## 350. Governance Flow

Policy Draft
│
▼
Syntax Validation
│
▼
Semantic Validation
│
▼
Security Floor Validation
│
▼
Lockout Analysis
│
▼
Simulation
│
▼
Review / Approval
│
▼
Compilation
│
▼
Versioned Bundle
│
▼
Deployment
│
▼
Runtime
│
▼
Monitoring
│
├── Rollback
└── Supersede

## 351. Relación con Laravel

Laravel ofrece mecanismos potentes y simples para:

- guards
- middleware
- password confirmation
- authentication events
- rate limiting
- Fortify
- Sanctum

y las aplicaciones pueden añadir reglas alrededor de ellos.
VoltStack conservará esa simplicidad de uso, pero convertirá los requisitos de Authentication en un modelo explícito y componible.
La intención no será reemplazar:

- Policies/Gates
- de Authorization.

Será resolver un dominio diferente:
Authentication Assurance Governance

## 352. Relación con Symfony

Symfony aporta un modelo especialmente sólido de:

- firewalls
- authenticators
- badges
- passports
- access control
- user checkers
- events

que demuestra el valor de separar mecanismos.
VoltStack extenderá esa separación mediante un Policy Engine formal que gobierne:

- authentication requirements
- assurance
- freshness
- risk adaptation
- device posture
- credential eligibility
- tenant rules
- security posture

## 353. Diferenciador VoltStack

VoltStack buscará combinar:

- Laravel-like DX
- +
- Symfony-like Security Contracts
- +

Explicit Authentication Policy Engine
+
Requirement Composition
+
Capability-Based Authentication
+
Adaptive Authentication
+
Security Posture
+
Tenant Governance
+
Policy Versioning
+
Policy Simulation
+
Shadow Deployment
+
Lockout Analysis
+
Distributed Policy Bundles
+
FrankenPHP-safe Runtime

## 354. Decisiones arquitectónicas principales

VoltStack adoptará:

## 01. Authentication Policy será distinta de Authorization.

## 1. Authenticators producirán evidence; no definirán governance.

## 2. Existirá un Authentication Policy Engine central.

## 3. Toda evaluación recibirá un Policy Context explícito.

## 4. Policies serán side-effect free.

## 5. Existirá un Framework Security Floor.

## 6. Tenant/Application policies no podrán debilitar ese floor.

## 7. Requirement composition será monotónica por defecto.

## 8. Requirements se normalizarán semánticamente.

## 9. Capabilities serán preferibles a hardcoding de métodos.

## 10. Evidence solo satisfará capabilities verificadas.

## 11. Authentication freshness será first-class.

## 12. Risk podrá endurecer requisitos.

## 13. Risk bajo no eliminará requisitos obligatorios.

## 14. Device Trust no será Authentication Evidence universal.

## 15. Security Posture será multidimensional.

## 16. Restricted Authentication será first-class.

## 17. Policies podrán gobernar login, enrollment, removal,

recovery, sessions, tokens y operaciones sensibles.

## 18. Policies tendrán IDs y versiones estables.

## 19. Requirement provenance será conservada.

## 20. Decisiones serán explicables mediante reason codes.

## 21. Public explanations ocultarán detalles sensibles.

## 22. Policy conflicts serán detectados explícitamente.

## 23. Security-critical conflicts serán fail-closed.

## 24. Policies imposibles serán detectables.

## 25. Governance analizará lockout risk.

## 26. Policy changes serán operaciones privilegiadas.

## 27. Policies críticas podrán requerir aprobación.

## 28. Governance y Runtime serán planos separados.

## 29. Policies podrán simularse antes de deployment.

## 30. Shadow mode será soportado.

## 31. Rollouts podrán ser graduales y deterministas.

## 32. Authentication transactions podrán fijar policy snapshot.

## 33. Security-relevant changes podrán disparar re-evaluation.

## 34. Sessions/tokens conservarán assurance y policy metadata.

## 35. Tokens no podrán incrementar assurance implícitamente.

## 36. Human y Machine principals compartirán Policy Engine

sin compartir necesariamente requirements.

## 37. Policies compilables serán optimizadas.

## 38. Runtime no dependerá necesariamente del authoring database.

## 39. Policy bundles serán versionados.

## 40. Policy bundle replacement será atómico.

## 41. Distributed nodes detectarán version drift.

## 42. Policy cache será context/version aware.

## 43. Policy Engine podrá ser long-lived bajo FrankenPHP

solo si permanece stateless.

## 44. Request/Fiber contexts permanecerán aislados.

## 45. Audit registrará policy changes y decisiones sensibles.

## 46. Policy evaluation será observable.

## 47. Plugins podrán registrar policies mediante contratos.

## 48. Custom policies no podrán usar side effects como parte

de su decisión.

49. El sistema será extensible sin sacrificar security floors.
50. Criterios de aceptación

El sistema será considerado completo cuando soporte:
51. AuthenticationPolicyEngine;
52. AuthenticationPolicyContext;
53. Policy Registry;
54. Policy IDs;
55. Policy Versions;
56. Policy scopes;
57. Framework Security Floor;
58. Platform policies;
59. Application policies;
60. Realm policies;
61. Tenant policies;
62. Principal policies;
63. Operation policies;
64. Risk policies;
65. Device policies;
66. requirement composition;
67. requirement normalization;
68. capability-based requirements;
69. MFA requirements;
70. passkey requirements;
71. freshness requirements;
72. device requirements;
73. assurance requirements;
74. evidence satisfaction;
75. partial satisfaction;
76. impossible requirements;
77. ALLOW;
78. CHALLENGE;
79. DENY;
80. RESTRICT;
81. Security Posture;
82. restricted authentication;
83. requirement provenance;
84. structured explainability;
85. conflict detection;
86. lockout analysis;
87. policy validation;
88. policy lifecycle;
89. policy approval;
90. policy versioning;
91. policy diff;
92. policy simulation;
93. shadow mode;
94. gradual rollout;
95. policy compilation;
96. versioned policy bundles;
97. atomic bundle deployment;
98. rollback;
99. emergency policy changes;
100. session sufficiency evaluation;
101. token policy integration;
102. federation integration;
103. machine authentication policies;
104. Security Center integration;
105. multi-tenancy;
106. distributed consistency;
107. version drift detection;
108. caching;
109. observability;
110. audit;
111. FrankenPHP isolation;
112. Fiber safety;
113. deterministic evaluation;
114. property-based testing;
115. performance testing.
116. Regla arquitectónica final

VoltStack deberá considerar Authentication Policy como una capa transversal:

```text
                    AUTHENTICATION MECHANISMS
                              │
                              ▼
                           EVIDENCE
                              │
                              ▼
                    AUTHENTICATION CONTEXT
                              │
                              ▼
                 ┌─────────────────────────┐
                 │  AUTHENTICATION POLICY  │
                 │         ENGINE          │
                 └────────────┬────────────┘
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
```

SECURITY FLOOR          TENANT/REALM             RISK
│                      │                      │
└──────────────────────┼──────────────────────┘
▼
REQUIREMENT COMPOSER
│
▼
EFFECTIVE AUTH REQUIREMENT
│
▼
SATISFACTION EVALUATOR
│
┌────────────────┼────────────────┐
▼                ▼                ▼
ALLOW          CHALLENGE         DENY
│
└──────────────┬─────────────────┘
▼
SECURITY POSTURE
│
▼
AUTHENTICATION CONTEXT
La primera regla será:
Authentication Policy nunca sustituirá Authorization; determinará la suficiencia y las condiciones de Authentication.

La segunda:
Ninguna policy de Tenant, aplicación, plugin o usuario podrá reducir silenciosamente las garantías mínimas impuestas por el Framework Security Floor.

La tercera:
VoltStack compondrá requisitos por propiedades de seguridad verificables siempre que sea posible, en lugar de acoplar la arquitectura a nombres concretos de autenticadores.

La cuarta:
Un Authentication Method solo aportará las capacidades que la evidencia concreta haya demostrado; el nombre del método no constituirá por sí mismo una garantía.

La quinta:
Risk podrá aumentar la exigencia de Authentication, pero una evaluación de bajo riesgo no eliminará requisitos obligatorios definidos por políticas superiores.

La sexta:
Una Authentication exitosa podrá producir un contexto restringido cuando Identity, Recovery, Credential, Device o Compromise Posture así lo requieran.

La séptima:
Toda policy activa será identificable, versionada, auditable y explicable.

La octava:
Toda modificación crítica de Authentication Governance deberá poder validarse, simularse y analizarse antes de afectar el runtime.

La novena:
Una Authentication Transaction utilizará una visión coherente de las policies y nunca una mezcla parcial de versiones.

La décima:

```php
Bajo FrankenPHP y ejecución distribuida, el Policy Engine podrá mantenerse long-lived únicamente como infraestructura inmutable/stateless; Identity, Tenant, Evidence, Risk y Request State permanecerán aislados por contexto.

Siguiente documento
```

La siguiente pieza es:
`37_AUTHENTICATION_ASSURANCE_LEVEL_AUTHENTICATION_CONTEXT_AND_TRUST_CLASSIFICATION_SYSTEM.md`
Aquí formalizaremos algo que el documento 36 ya necesita pero deliberadamente no debe resolver por completo:

```text
What does "strong authentication" actually mean?
        │
        ├── Authentication Assurance Levels
        ├── Authenticator Assurance
        ├── Evidence Strength
        ├── Factor Independence
        ├── Phishing Resistance
        ├── Hardware Backing
        ├── User Verification
        ├── Authentication Freshness
        ├── Session Assurance
        ├── Federation ACR/AMR Mapping
        ├── Step-Up / Step-Down
        └── Trust Classification
```

Ese documento permitirá que VoltStack deje de manejar conceptos ambiguos como strong, secure o trusted y los convierta en propiedades formales de Authentication Assurance.
