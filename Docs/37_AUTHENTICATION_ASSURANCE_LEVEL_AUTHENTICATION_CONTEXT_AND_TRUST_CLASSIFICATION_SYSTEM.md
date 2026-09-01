# VoltStack Authentication System

## 37 — Authentication Assurance Level, Authentication Context and Trust Classification System

- **Archivo:** `37_AUTHENTICATION_ASSURANCE_LEVEL_AUTHENTICATION_CONTEXT_AND_TRUST_CLASSIFICATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Nivel:** Core / Security-Critical / Trust Model
- **Dependencias principales:** 01–36, especialmente 08, 10, 11, 12, 14, 15, 16, 17, 20, 21, 24, 29, 31, 32, 33, 34, 35 y 36.

---

## 1. Propósito

Este documento define el modelo formal mediante el cual VoltStack representará:

- Authentication Assurance
- Authentication Context
- Evidence Strength
- Authenticator Capabilities
- Factor Independence
- Trust Classification
- Authentication Freshness
- Session Assurance
- Federated Assurance
- Machine/Workload Assurance
- Step-Up
- Step-Down
- Assurance Degradation

El objetivo es eliminar conceptos ambiguos como:

- secure login
- strong authentication
- trusted login
- high security
- verified user
- strong MFA

y reemplazarlos por propiedades explícitas y evaluables.

## 2. Problema

Una aplicación tradicional puede almacenar:

```php
$user->authenticated = true;
VoltStack considera insuficiente ese modelo.
```

Dos sesiones pueden estar autenticadas y tener propiedades radicalmente distintas:

- Session A
- ─────────
- Password

Authenticated 12 hours ago
Unknown device
Elevated risk

Session B
─────────
Hardware-backed passkey
User verification
Authenticated 30 seconds ago
Managed device
Low risk
Ambas son:

```php
authenticated = true
pero no poseen la misma garantía.
```

## 3. Principio fundamental

Authentication es un estado multidimensional, no un booleano.

## 1. Segundo principio

Assurance representa qué tan confiable es la evidencia de Authentication; no representa qué puede hacer el principal.

Por tanto:

```text
Authentication Assurance
        ≠
Authorization Privilege
```

## 5. Ejemplo

Alice
Role: Administrator
Authentication Assurance: LOW

Bob
Role: User
Authentication Assurance: HIGH
Alice puede tener más permisos potenciales.
Bob puede haber demostrado su identidad con mayor garantía.

## 6. Modelo conceptual

AUTHENTICATION EVENT
│
▼
Authentication Evidence
│
┌──────────────┼──────────────┐
▼              ▼              ▼
Method          Factor       Capabilities
│              │              │
└──────────────┼──────────────┘
▼
Evidence Assessment
│
▼
Authentication Context
│
┌──────────────┼──────────────┐
▼              ▼              ▼
Freshness         Risk         Device Trust
│              │              │
└──────────────┼──────────────┘
▼
Assurance Evaluator
│
▼
Authentication Assurance
│
▼
Trust Classification
│
▼
Policy Engine (36)

## 7. Authentication Assurance Level

VoltStack deberá proporcionar niveles internos de assurance.

```php
Ejemplo base:
enum AuthenticationAssuranceLevel: int
{
    case None       = 0;
    case Low        = 10;
    case Standard   = 20;
    case Strong     = 30;
    case High       = 40;
    case Privileged = 50;
}
```

## 8. Los números no son seguridad

Los valores:

- 10
- 20
- 30
- 40
- 50

solo sirven para ordenamiento interno cuando sea semánticamente válido.
No significan:
High = 40% secure

## 9. Assurance Level != Authentication Method

No establecer:

```php
Password = LOW
TOTP = STANDARD
Passkey = HIGH
de forma universal.
```

El assurance dependerá de:

- method
- configuration
- verification
- credential properties
- factor combination
- freshness
- binding
- context

## 10. Ejemplo

Dos passkeys pueden diferir:

```text
Passkey A
─────────
User verification: yes
Hardware backed: yes
Attestation verified: yes

Passkey B
─────────
User verification: no
Hardware backing: unknown
Attestation: unavailable
```

No deben necesariamente producir el mismo assurance.

## 11. Assurance como resultado derivado

VoltStack calculará:

```text
Evidence
   +
Capabilities
   +
Factor Composition
   +
Freshness
   +
Credential State
   +
Authentication Context
   =

Authentication Assurance
```

## 12. AssuranceProfile

Además del nivel agregado, deberá conservarse un perfil multidimensional.
final readonly class AuthenticationAssuranceProfile
{
public function __construct(
public AuthenticationAssuranceLevel $level,
public AuthenticationCapabilitySet $capabilities,
public AuthenticationFreshness $freshness,
public FactorComposition $factors,
public TrustClassification $trust,
) {}
}

## 13. Por qué conservar Profile

Porque:
HIGH
por sí solo no responde:

- Was it phishing resistant?
- Was user verification performed?

Was hardware backing verified?
Was it MFA?
Was the device managed?
How old is the authentication?

## 14. Authentication Capability

Capabilities expresan propiedades demostradas.

```php
enum AuthenticationCapability: string
{
    case IdentityProof = 'identity_proof';

    case KnowledgeFactor = 'knowledge_factor';
    case PossessionFactor = 'possession_factor';
    case InherenceFactor = 'inherence_factor';

    case MultiFactor = 'multi_factor';

    case UserPresence = 'user_presence';
    case UserVerification = 'user_verification';

    case PhishingResistant = 'phishing_resistant';
    case ReplayResistant = 'replay_resistant';

    case DeviceBound = 'device_bound';
    case HardwareBacked = 'hardware_backed';

    case ChannelBound = 'channel_bound';

    case FederationBacked = 'federation_backed';

    case WorkloadAttested = 'workload_attested';
}
```

## 15. Capability Verification

Una capability deberá tener estado.

- VERIFIED
- ASSERTED
- INFERRED
- UNKNOWN
- UNSUPPORTED

## 16. VERIFIED

VoltStack tiene evidencia suficiente.

## 17. ASSERTED

Un tercero confiable la declara.
Ejemplo:

```php
OIDC Provider:
amr = ["pwd", "otp"]
```

## 18. INFERRED

VoltStack la deriva de propiedades conocidas.
Debe utilizarse con cautela.

## 19. UNKNOWN

No existe información suficiente.

## 20. UNSUPPORTED

El authenticator/protocol no puede proporcionar esa propiedad.

## 21. Regla

UNKNOWN nunca equivale a VERIFIED.

## 1. CapabilityEvidence

final readonly class AuthenticationCapabilityEvidence
{
public function __construct(
public AuthenticationCapability $capability,
public CapabilityVerificationState $state,
public EvidenceSource $source,
) {}
}

## 2. Evidence Strength

Cada evidencia podrá ser evaluada individualmente.

```php
final readonly class AuthenticationEvidenceAssessment
{
    public function __construct(
        public AuthenticationEvidenceReference $evidence,
        public AuthenticationMethod $method,
        public AuthenticationCapabilitySet $capabilities,
        public EvidenceStrength $strength,
        public AuthenticationInstant $verifiedAt,
    ) {}
}
```

## 3. Evidence Strength

Ejemplo conceptual:

- WEAK
- MODERATE
- STRONG
- VERY_STRONG

No sustituye al Assurance Level global.

## 4. Password Evidence

Puede aportar:

- KNOWLEDGE_FACTOR
- IDENTITY_PROOF

pero normalmente no:

- PHISHING_RESISTANT
- HARDWARE_BACKED

## 5. TOTP Evidence

Puede aportar:

- POSSESSION_FACTOR
- dependiendo del modelo.

Pero:

- TOTP
- ≠
- phishing resistant

## 6. SMS OTP

No deberá clasificarse automáticamente igual que un authenticator criptográfico fuerte.

## 7. Passkey

Puede aportar:

- POSSESSION_FACTOR
- PHISHING_RESISTANT
- REPLAY_RESISTANT
- USER_PRESENCE
- USER_VERIFICATION
- DEVICE_BOUND
- HARDWARE_BACKED

pero únicamente las propiedades realmente demostradas.

## 8. Client Certificate

Puede aportar:

- POSSESSION_FACTOR
- CRYPTOGRAPHIC_PROOF
- CHANNEL_BOUND
- según protocolo/configuración.

## 9. Federated Authentication

Puede aportar assurance mediante:

- issuer trust
- signature validation
- acr
- amr
- authentication time
- federation policy

## 10. Workload Identity

Puede aportar:

- WORKLOAD_ATTESTED
- CRYPTOGRAPHIC_PROOF
- SHORT_LIVED_CREDENTIAL
- AUDIENCE_BOUND
- según documento 33.

## 11. Factor Model

VoltStack deberá modelar factores independientemente de los métodos.

## 12. AuthenticationFactorClass

enum AuthenticationFactorClass: string
{
case Knowledge = 'knowledge';
case Possession = 'possession';
case Inherence = 'inherence';

case Cryptographic = 'cryptographic';
case Device = 'device';
case Federated = 'federated';
case Recovery = 'recovery';
case Workload = 'workload';
}

## 13. Factor Independence

Dos métodos distintos no implican dos factores independientes.

## 14. Ejemplo

Password

+

PIN derived from same password
no debe considerarse automáticamente MFA.

## 36. FactorDependencyGraph

VoltStack podrá modelar dependencias:

```text
Credential A ─────┐
                  ├── Same Root Secret
Credential B ─────┘
```

## 37. Factor Independence Evaluator

interface FactorIndependenceEvaluatorInterface
{
public function evaluate(
AuthenticationEvidenceSet $evidence
): FactorIndependenceAssessment;
}

## 38. Independence States

INDEPENDENT
PARTIALLY_DEPENDENT
DEPENDENT
UNKNOWN

## 39. MFA Definition

VoltStack considerará MFA válido cuando la policy requerida determine que existe una combinación suficiente de factores independientes.

## 40. MFA != Two Methods

method_count >= 2
no es definición suficiente.

## 41. Authentication Context

El contexto será un snapshot explícito del estado Authentication relevante.

## 42. AuthenticationContext

final readonly class AuthenticationContext
{
public function __construct(
public PrincipalReference $principal,
public PrincipalType $principalType,
public AuthenticationEvidenceSet $evidence,
public AuthenticationAssuranceProfile $assurance,
public AuthenticationFreshness $freshness,
public AuthenticationSecurityPosture $posture,
public AuthenticationSessionContext $session,
public AuthenticationDeviceContext $device,
public AuthenticationRiskContext $risk,
public AuthenticationTenantContext $tenant,
public AuthenticationRealmContext $realm,
) {}
}

## 43. AuthenticationContext no es Request

No deberá contener indiscriminadamente todo HTTP Request.

## 44. Context Projection

Solo incluir security-relevant data.

## 45. Context Immutability

Una vez emitido para una operación:

- immutable snapshot
- salvo nueva evaluación.

## 46. Request Scoped

Especialmente importante con FrankenPHP.
Nunca:

```php
Auth::$currentContext
como estado mutable global.
```

## 47. Context Version

Podrá incluir:

- context_version
- para compatibilidad y auditoría.

## 48. Context Provenance

Cada propiedad relevante debería poder indicar su fuente.

```php
Ejemplo:
device.trust
source = corporate_mdm
```

## 49. Authentication Time

Distinguir:

- authentication_started_at
- authentication_completed_at
- evidence_verified_at
- session_issued_at
- last_step_up_at

## 50. No single auth timestamp

Una sola columna:

- authenticated_at
- puede resultar insuficiente.

## 51. Authentication Freshness

Freshness deberá modelarse por evidencia y contexto.

## 52. Freshness Model

final readonly class AuthenticationFreshness
{
public function __construct(
public DateTimeImmutable $lastAuthenticationAt,
public array $factorFreshness,
) {}
}

## 53. Factor-Specific Freshness

Ejemplo:

```text
Password:
45 minutes
```

Passkey:
2 minutes

## 54. Requirement

phishing-resistant authentication <= 5 minutes
puede satisfacerse aunque el password sea antiguo.

## 55. Freshness Is Not Session Age

session age
≠
authentication freshness

## 56. Ejemplo

Session creada hace:
8 hours
pero passkey step-up ocurrió hace:
30 seconds

## 57. Assurance Freshness

Assurance podrá degradarse cuando la evidencia envejece.

## 58. Ejemplo conceptual

t0
Passkey authentication
HIGH assurance

+15 min
HIGH but not fresh enough for CRITICAL operation

+8 h
session still valid
but effective assurance for sensitive operation reduced

## 59. Assurance Degradation

Debe ser explícita.

## 60. Degradation Causes

time
risk increase
device posture change
credential revocation
session state change
identity state change
policy change
security epoch change

## 61. No Permanent High Assurance

Una sesión no deberá conservar:

- HIGH
- para siempre solo porque una vez realizó strong authentication.

## 62. Base Assurance vs Effective Assurance

Distinguir:
ACHIEVED_ASSURANCE
de:
EFFECTIVE_ASSURANCE

## 63. Achieved Assurance

Lo demostrado originalmente.

## 64. Effective Assurance

Lo que sigue siendo aceptable ahora bajo contexto actual.

## 65. Example

Achieved: HIGH

Current device:
COMPROMISED

Effective:
RESTRICTED / insufficient

## 66. AssuranceEvaluator

interface AuthenticationAssuranceEvaluatorInterface
{
public function evaluate(
AuthenticationEvidenceSet $evidence,
AuthenticationAssuranceContext $context
): AuthenticationAssuranceProfile;
}

## 67. EffectiveAssuranceEvaluator

interface EffectiveAuthenticationAssuranceEvaluatorInterface
{
public function evaluate(
AuthenticationAssuranceProfile $achieved,
AuthenticationContext $context
): EffectiveAuthenticationAssurance;
}

## 68. Assurance Rules

El evaluator deberá utilizar reglas versionadas.

## 69. Assurance Policy Version

Importante para auditoría:
assurance_model_version = 4

## 70. Why

Una organización puede cambiar qué considera:

- HIGH
- sin cambiar el método utilizado históricamente.

## 71. Trust Classification

Trust será una clasificación contextual.
No será sinónimo de assurance.

## 72. Assurance vs Trust

ASSURANCE
How strongly was identity authenticated?

TRUST
How much confidence should this context receive now?

## 73. Example

Strong passkey authentication
+
Compromised device
Puede producir:

- Authentication Assurance = HIGH
- Context Trust = UNTRUSTED

## 74. TrustClassification

enum TrustClassification: string
{
case Unknown = 'unknown';
case Untrusted = 'untrusted';
case Limited = 'limited';
case Normal = 'normal';
case Elevated = 'elevated';
case Trusted = 'trusted';
case HighlyTrusted = 'highly_trusted';
}

## 75. "Trusted" no significa seguro absoluto

Es una clasificación contextual relativa a policies.

## 76. Trust Inputs

Puede considerar:

- authentication assurance
- device posture
- risk
- network
- identity state
- credential state
- session state
- tenant policy
- realm

## 77. TrustClassifier

interface AuthenticationTrustClassifierInterface
{
public function classify(
AuthenticationTrustContext $context
): AuthenticationTrustAssessment;
}

## 78. TrustAssessment

final readonly class AuthenticationTrustAssessment
{
public function __construct(
public TrustClassification $classification,
public array $signals,
public array $reasonCodes,
) {}
}

## 79. Trust Signals

Ejemplos:

```text
STRONG_AUTHENTICATION
PHISHING_RESISTANT
KNOWN_DEVICE
MANAGED_DEVICE
LOW_RISK

UNKNOWN_DEVICE
TOR_EXIT_NODE
CREDENTIAL_RECENTLY_RESET
RECOVERY_AUTHENTICATION
COMPROMISED_DEVICE
```

## 80. Positive signals cannot erase hard failures

Ejemplo:

- hardware passkey
- +
- account suspended

no produce acceso simplemente por strong authentication.

## 81. Trust Composition

No deberá ser:

- positive points - negative points
- como única lógica.

## 82. Hard Constraints

Algunas señales son dominantes.

```php
Ejemplo:
Identity = BLOCKED
→ context cannot become trusted
```

## 83. Security Posture vs Trust

Diferencia:

```text
Security Posture
= state of security-relevant entities

Trust Classification
= contextual interpretation of those states
```

## 84. Example

Posture:

```text
Identity NORMAL
Credential HEALTHY
Device UNKNOWN
Risk ELEVATED
```

Trust:
LIMITED

## 85. Trust Is Operation-Relative

Una sesión puede ser suficientemente confiable para:
view profile
pero no para:
rotate production credentials

## 86. Therefore

Policy Engine 36 debe comparar:
Effective Assurance / Trust
contra:
Operation Requirement

## 87. Authentication Context Class

VoltStack deberá clasificar contextos semánticos.

## 88. Context Classes

ANONYMOUS
IDENTIFIED
AUTHENTICATED
RESTRICTED
STRONG_AUTHENTICATED
PRIVILEGED
RECOVERY
IMPERSONATED
BREAK_GLASS
MACHINE
WORKLOAD

## 89. Context Class != Assurance

Por ejemplo:

- MACHINE
- describe tipo de contexto, no su fuerza.

## 90. Restricted Context

Puede existir:

- AUTHENTICATED
- +
- RESTRICTED
- +
- HIGH ASSURANCE

si la restricción proviene de otro estado.

## 91. AuthenticationContextClass

enum AuthenticationContextClass: string
{
case Anonymous = 'anonymous';
case Identified = 'identified';
case Authenticated = 'authenticated';
case Restricted = 'restricted';
case Privileged = 'privileged';
case Recovery = 'recovery';
case Impersonated = 'impersonated';
case BreakGlass = 'break_glass';
case Machine = 'machine';
case Workload = 'workload';
}

## 92. Multiple Classifications

Puede ser mejor modelarlas como flags/tags cuando no sean mutuamente excluyentes.

## 93. Example

AUTHENTICATED
PRIVILEGED
IMPERSONATED
simultáneamente.

## 94. Authentication Context Attributes

Ejemplo:

- principal_type
- realm
- tenant
- authentication_methods
- factor_classes
- assurance_level
- capabilities
- freshness
- trust
- device_posture
- risk_level
- session_type
- recovery_state
- impersonation_state

## 95. Authentication Context Fingerprint

Para ciertos usos puede generarse una representación/hash de propiedades security-relevant.

## 96. No secrets

Fingerprint nunca deberá contener material secreto.

## 97. Context Serialization

Si se serializa:

- versioned
- minimal

signed/integrity protected when required

## 98. Context Is Not a Bearer Credential

Un serialized AuthenticationContext no deberá convertirse accidentalmente en token reutilizable.

## 99. Session Assurance

Cada Session deberá registrar el assurance obtenido al establecerse.

## 100. SessionAuthenticationState

final readonly class SessionAuthenticationState
{
public function __construct(
public AuthenticationAssuranceProfile $achievedAssurance,
public AuthenticationFreshness $freshness,
public AuthenticationCapabilitySet $capabilities,
public AuthenticationSecurityEpoch $securityEpoch,
) {}
}

## 101. Session Assurance Snapshot

Registrar:

- achieved level
- methods
- capabilities
- factor composition
- timestamps
- model version

## 102. Session Re-evaluation

La sesión deberá reevaluarse cuando sea necesario.

## 103. Step-Up Authentication

Documento 32.
Step-up significa obtener evidencia adicional para satisfacer requirement superior.

## 104. Example

Current:
STANDARD

Operation:

```text
HIGH required

          ↓

Step-Up Challenge

          ↓

Passkey

          ↓

HIGH
```

## 105. Step-Up Is Incremental

No necesariamente debe destruir la sesión existente.

## 106. Step-Up Context

Debe preservar:

- original session
- requested operation
- tenant
- realm
- challenge binding

## 107. Step-Up Proof

Puede aumentar:

- freshness
- capabilities
- assurance

## 108. Step-Down

VoltStack también deberá soportar reducción efectiva de assurance.

## 109. Step-Down Causes

timeout
risk escalation
device compromise
credential compromise
policy update
logout from privileged mode
impersonation start
recovery state

## 110. Step-Down != Logout

Puede conservar sesión pero reducir capacidades.

## 111. Example

PRIVILEGED context
↓ timeout
STANDARD context

## 112. Privileged Assurance

Documento 32.
No deberá equivaler simplemente a:

```php
level = 50
Debe requerir propiedades explícitas.
```

## 113. Example Privileged Profile

level: PRIVILEGED
freshness: <= 5 min
phishing_resistant: VERIFIED
user_verification: VERIFIED
device_trust: MANAGED
según policy.

## 114. Privileged Role Is Separate

role=admin
no produce:
assurance=privileged

## 115. Break-Glass Assurance

Break-glass tendrá classification específica.

## 116. Break-Glass Is Not "Maximum Trust"

Importante.
Una emergency identity puede estar:
strongly authenticated
pero su contexto deberá clasificarse:

- BREAK_GLASS
- y recibir controles adicionales.

## 117. Example

Assurance:
HIGH

Context:
BREAK_GLASS

Trust:
LIMITED_FOR_NORMAL_OPERATIONS

Authorization scope:
EMERGENCY_ONLY

## 118. Recovery Assurance

Recovery evidence no deberá heredar automáticamente assurance normal.

## 119. Recovery Context

Context Class: RECOVERY
Posture: RESTRICTED
hasta completar requisitos.

## 120. Account Linking

Documento 34.
Vincular un nuevo Authentication Method deberá requerir assurance suficiente.

## 121. Example

Current Authentication:
password-only / old session

Operation:
link new passkey

Policy:
fresh strong authentication required

## 122. Credential Binding

La nueva credential puede registrar:

- binding assurance
- binding context
- binding timestamp

## 123. Assurance at Enrollment

Importante distinguir:
Credential Strength
de:
Assurance used to enroll credential

## 124. Example

Una passkey fuerte enrolada durante una sesión comprometida es un problema distinto.

## 125. EnrollmentAssurance

final readonly class CredentialEnrollmentAssurance
{
public function __construct(
public AuthenticationAssuranceProfile $profile,
public DateTimeImmutable $boundAt,
) {}
}

## 126. Credential Trust

Una credential podrá clasificarse según:

- origin
- enrollment assurance
- age
- hardware properties
- attestation
- compromise status
- last use

## 127. CredentialTrustClassification

No debe confundirse con Context Trust.

## 128. Device Assurance

Device Trust 21 tampoco equivale a Authentication Assurance.

## 129. Example

Managed Device
+
No user authentication
no significa:
Authenticated User

## 130. Device Attestation

Puede aumentar confianza contextual, pero no sustituye Identity Proof salvo modelo explícito de workload/device principal.

## 131. Federation Assurance

Documento 17.
VoltStack deberá traducir assurance externo a su modelo interno.

## 132. OIDC ACR

Un proveedor puede emitir:
acr

## 133. OIDC AMR

Puede emitir:
amr
por ejemplo:

- pwd
- otp
- mfa
- hwk
- fpt
- dependiendo del proveedor.

## 134. Never Blindly Trust ACR/AMR

Primero verificar:

- issuer
- signature
- audience
- nonce
- federation policy
- claim semantics
- provider configuration

## 135. Assurance Mapping

interface FederatedAssuranceMapperInterface
{
public function map(
FederatedAuthenticationAssertion $assertion,
FederationTrustContext $context
): AuthenticationAssuranceProfile;
}

## 136. Provider-Specific Mapping

Distintos IdPs pueden utilizar valores ACR diferentes.

## 137. Mapping Registry

Issuer
↓
Assurance Mapping Profile
↓
VoltStack Capabilities

## 138. Example

External ACR:

```text
urn:company:aal3

        ↓ trusted mapping
```

VoltStack:

- HIGH
- PHISHING_RESISTANT
- USER_VERIFICATION

solo si el contrato de confianza lo garantiza.

## 139. Unknown ACR

No convertir a HIGH automáticamente.

## 140. Federation Freshness

Debe utilizar:

- auth_time
- cuando sea confiable y requerido.

## 141. Federation Reauthentication

Puede requerir:

```php
prompt=login
max_age
```

u otro mecanismo del protocolo/proveedor.

## 142. Local Step-Up

Alternativamente VoltStack puede añadir factor local según policy.

## 143. Mixed Assurance

Ejemplo:

- Federated Password
- +
- Local Passkey

puede producir assurance compuesto.

## 144. Provenance

Debe conservarse qué parte provino de:

- external IdP
- local authenticator
- device provider
- risk engine

## 145. Machine Assurance

Documento 33.
No utilizar conceptos humanos inapropiados.

## 146. Machine Authentication Profile

Puede considerar:

- credential type
- key protection
- attestation
- issuer
- token lifetime
- audience binding
- certificate validation
- workload identity
- replay resistance

## 147. Machine Assurance Levels

Pueden compartir enum general si semánticamente funciona, pero conservar profile específico.

## 148. Example

Static API Key
→ LOW/MODERATE machine assurance

Short-lived workload identity

+ attestation
+ audience binding
+ cryptographic proof
→ HIGH machine assurance
según policy.

## 1. Machine MFA

No inventar:

- machine entered OTP
- para imitar MFA humano.

## 2. Workload Attestation

Puede funcionar como propiedad de confianza de workload.

## 3. Machine Freshness

Se deriva de:

- credential issuance time
- token age
- attestation time
- certificate validity

## 4. Service-to-Service Context

final readonly class MachineAuthenticationContext
{
public function __construct(
public ServicePrincipalReference $principal,
public WorkloadIdentity $workload,
public AuthenticationEvidenceSet $evidence,
public AuthenticationAssuranceProfile $assurance,
public Audience $audience,
public TrustDomain $trustDomain,
) {}
}

## 5. Actor Chains

Delegation/impersonation debe conservar:

- actor
- subject
- delegation chain

## 6. Assurance Through Delegation

No asumir:
subject assurance = actor assurance

## 7. Example

Human Admin
HIGH assurance

↓ delegates

Automation Service
La service identity debe tener su propio Authentication Context.

## 156. On-Behalf-Of

Context puede registrar:

- actor_assurance
- subject_context
- delegation_assurance

## 157. Impersonation

Documento 32.
Impersonation debe estar visible en Context.

## 158. Impersonation Trust

Puede reducir trust para determinadas operaciones aunque actor tenga high assurance.

## 159. Context Chains

Original Actor
│
▼
Delegation
│
▼
Effective Subject
│
▼
Authentication Context Chain

## 160. Assurance Provenance

VoltStack deberá poder responder:
¿Por qué este contexto fue clasificado como HIGH?

## 1. AssuranceExplanation

final readonly class AuthenticationAssuranceExplanation
{
public function __construct(
public AuthenticationAssuranceLevel $level,
public array $evidence,
public array $capabilities,
public array $reasonCodes,
public AssuranceModelVersion $modelVersion,
) {}
}

## 2. Example reason codes

AUTH_ASSURANCE_PASSWORD_VERIFIED
AUTH_ASSURANCE_INDEPENDENT_POSSESSION_FACTOR
AUTH_ASSURANCE_PHISHING_RESISTANCE_VERIFIED
AUTH_ASSURANCE_USER_VERIFICATION_VERIFIED
AUTH_ASSURANCE_HARDWARE_BACKING_UNKNOWN

## 3. Internal vs Public Explanation

Igual que documento 36.
No revelar detalles explotables.

## 4. Assurance Policy

VoltStack deberá separar:
Assurance Evaluation Rules
de:
Operation Requirements

## 5. Example

Assurance system:
This evidence profile qualifies as HIGH.
Policy system:
This operation requires HIGH.

## 6. Separation Diagram

Evidence
│
▼
ASSURANCE SYSTEM
│
▼
Assurance Profile
│
│
├───────────────┐
│               │
▼               ▼
Trust Classifier  Policy Engine
│
▼
Requirement
│
▼
Decision

## 7. Assurance Model Registry

interface AuthenticationAssuranceModelRegistryInterface
{
public function current(): AuthenticationAssuranceModel;

public function get(
AssuranceModelVersion $version
): AuthenticationAssuranceModel;
}

## 168. Versioned Models

Permite reproducir decisiones históricas.

## 169. Assurance Model Migration

Cambiar modelo no debe reescribir silenciosamente eventos históricos.

## 170. Historical Truth

Conservar:

- At time T,
- under assurance model V3,

this authentication was classified HIGH.

## 171. Current Re-Evaluation

Separadamente:

- Under current model V5,
- this session is now insufficient.

## 172. Assurance Floors

Framework puede definir mínimos para niveles.

## 173. Example

HIGH podría requerir siempre ciertas propiedades mínimas.
Pero aplicaciones pueden definir requisitos adicionales.

## 174. Do Not Let Tenants Redefine Semantics Arbitrarily

Tenant no debería poder decir:

- Password-only = PRIVILEGED
- si viola framework assurance floor.

## 175. Custom Assurance Levels

Enterprise extension puede añadir:

- CORPORATE_HIGH
- REGULATED

pero preferentemente como profiles/classifications sobre un core estable.

## 176. Interoperability

Mantener core assurance suficientemente neutral para integrarse con estándares externos.

## 177. NIST-like Mapping

VoltStack podrá permitir mappings hacia esquemas externos como:

- AAL1
- AAL2
- AAL3

sin hacerlos necesariamente idénticos a sus niveles internos.

## 178. Important

No afirmar:

- VoltStack HIGH == external AAL3
- sin mapping explícito.

## 179. ExternalAssuranceMapping

interface ExternalAssuranceMappingInterface
{
public function map(
AuthenticationAssuranceProfile $profile
): ExternalAssuranceReference;
}

## 180. Compliance Profiles

Podrán existir adaptadores:

- NIST profile
- enterprise profile
- banking profile
- government profile
- sin contaminar Core.

## 181. Trust Domain

Para federation/workloads:

- TrustDomain
- debe ser first-class.

## 182. Trust Domain != Tenant

Puede coincidir o cruzar tenants dependiendo de arquitectura.

## 183. Trust Boundary

Toda transición entre trust domains debe reevaluarse.

## 184. Cross-Domain Assurance

No conservar assurance ciegamente al cruzar dominios.

## 185. Example

Internal Corporate Trust Domain
↓
External Partner Domain
puede requerir:

- assurance remapping
- audience validation
- policy re-evaluation

## 186. Realm Assurance

Realm también puede imponer constraints.

## 187. Example

customer realm
→ STANDARD

admin realm
→ HIGH + phishing resistant

## 188. Tenant Assurance

Tenant puede endurecer.

## 189. Security Floor

Documento 36.
Tenant no puede debilitar mínimos.

## 190. Trust Boundary Events

realm change
tenant change
privilege elevation
delegation
impersonation
federation
service exchange
deben ser visibles.

## 191. Continuous Re-Evaluation

Authentication Context no implica vigilancia biométrica continua.
Significa reevaluar señales cuando existan triggers relevantes.

## 192. Re-Evaluation Triggers

new request classification
risk update
device state update
credential revocation
security epoch
policy bundle update
session age threshold
privilege request
tenant/realm transition

## 193. ContextReevaluationService

interface AuthenticationContextReevaluationServiceInterface
{
public function reevaluate(
AuthenticationContext $context,
AuthenticationReevaluationTrigger $trigger
): AuthenticationContext;
}

## 194. Re-evaluation Output

Puede producir:

- unchanged
- step-up required
- step-down
- restricted
- revoked

## 195. Revocation

Documento 40 profundizará.

## 196. Security Epoch

Context deberá registrar epoch.

## 197. Example

Session epoch = 17
Identity security epoch = 18
Resultado:

- session requires invalidation/re-evaluation
- según policy.

## 198. Context Expiration

Authentication Context snapshots pueden tener vida limitada.

## 199. Context Refresh

No reutilizar snapshots indefinidamente.

## 200. Risk Snapshot

Debe registrar:

- risk level
- assessment timestamp
- model/version

## 201. Device Snapshot

Debe registrar:

- device trust
- assessment timestamp
- provider

## 202. Stale Context Inputs

Una policy podrá declarar edad máxima de ciertos inputs.

## 203. Example

device compliance must be <= 15 minutes old

## 204. Context Input Freshness

No solo Authentication Evidence tiene freshness.

- También:
- risk assessment
- device compliance
- federation assertion
- workload attestation

## 205. Trust Signal Freshness

Debe modelarse cuando sea relevante.

## 206. Authentication Context Builder

interface AuthenticationContextBuilderInterface
{
public function build(
AuthenticationContextInput $input
): AuthenticationContext;
}

## 207. Builder Responsibilities

collect verified inputs
normalize evidence
evaluate factor independence
calculate achieved assurance
calculate effective assurance
classify trust
attach provenance

## 208. Builder Should Not Authorize

No:
if role admin allow

## 209. AuthenticationContextResolver

Para runtime:

```php
interface AuthenticationContextResolverInterface
{
    public function current(): AuthenticationContext;
}
```

## 210. Request-local resolver

En FrankenPHP deberá ser request/fiber scoped.

## 211. Context Storage

Session puede almacenar datos necesarios para reconstrucción.
Preferible no serializar objetos runtime completos sin versionado.

## 212. Minimal Session Representation

Ejemplo:

- principal id
- session id
- auth events
- assurance snapshot
- capabilities
- security epoch
- policy/model versions

## 213. Reconstruct Current Context

Risk/device/posture dinámicos pueden reevaluarse.

## 214. Token Context

Stateless token puede contener subset.

## 215. Token Claims

Nunca aceptar claims de assurance sin:

- signature validation
- issuer trust
- audience validation
- token validity
- claim mapping

## 216. Assurance Claim

Podría existir:

- auth_assurance
- pero deberá tratarse como issuer assertion.

## 217. Context Binding

Authentication Context puede estar ligado a:

- session
- token
- client
- device
- tenant
- realm

## 218. Context Transfer

No transferir ciegamente entre:

```text
browser → API token
tenant A → tenant B
```

user realm → admin realm

## 219. Assurance Translation

Cada transición puede requerir mapping.

## 220. Browser → API Token

Token emitido desde sesión puede registrar:

- issuance_assurance
- pero sus capabilities se determinan separadamente.

## 221. API Token Lifetime

Token puede sobrevivir más que freshness de sesión.
No debe significar que freshness humana se mantiene.

## 222. Sensitive API Operations

Pueden requerir token emitido bajo cierto assurance o mecanismo adicional.

## 223. Authentication Method Management

Documento 34.
Removal/enrollment decisions usarán Context.

## 224. Security Center

Documento 35.

- Podrá mostrar una versión human-friendly:
- Strong authentication enabled
- 2 passkeys
- MFA active
- 3 trusted devices

## 225. Do Not Expose Internal Scores

Security Center no necesita mostrar:

```php
trust_score=0.837263
si no aporta valor.
```

## 226. User-Facing Trust Language

Evitar promesas absolutas:
Your account is 100% secure.
Preferir:
Your account has strong authentication protections enabled.

## 227. Governance

Cambios en Assurance Model son security-critical.

## 228. Assurance Model Lifecycle

DRAFT
↓
VALIDATED
↓
SIMULATED
↓
APPROVED
↓
ACTIVE
↓
SUPERSEDED
↓
RETIRED

## 229. Assurance Simulation

Antes de cambiar clasificación:

- How many current sessions become insufficient?
- How many users need step-up?

How many tenants lose compatible methods?

## 230. Shadow Evaluation

Nuevo assurance model puede ejecutarse en shadow.

## 231. Example

Current model:
HIGH

Candidate model:

```text
STANDARD

No enforcement yet.
```

## 232. Model Rollout

Debe coordinarse con Policy Engine 36.

## 233. Atomicity

Policy bundle y assurance model deben ser compatibles.

## 234. Compatibility Metadata

Policy bundle podrá declarar:
minimum_assurance_model_version

## 235. Incompatible Versions

Runtime deberá:

- fail closed
- para decisiones críticas.

## 236. Distributed Runtime

Todos los nodes deberán conocer:

- policy bundle version
- assurance model version
- security epoch
- cuando corresponda.

## 237. Drift Detection

Ejemplo:

```text
Node A:
Assurance Model V7
```

Node B:

- Assurance Model V6
- debe ser observable.

## 238. Cache

Puede cachearse:

- method capability metadata
- assurance rule graph
- federation mappings

## 239. Do Not Cache Blindly

No cachear:

- effective assurance forever
- ignorando contexto dinámico.

## 240. Cache Key

Cuando corresponda:

- assurance model version
- evidence fingerprint
- context classification
- security epoch

## 241. Evidence Fingerprint

Debe utilizar IDs/hashes seguros, no material secreto.

## 242. Performance

El assurance evaluator estará en hot path.
Debe optimizarse.

## 243. Compilation

Rules estáticas podrán compilarse.

## 244. Capability Bitset

Internamente capabilities podrían representarse eficientemente mediante:

- bitsets
- sin exponer esa implementación al dominio.

## 245. Precomputed Authenticator Metadata

Ejemplo:

```text
AuthenticatorDefinition
├── possible capabilities
├── factor class
├── phishing resistance support
└── hardware metadata support
```

## 246. Possible != Verified

Precomputed metadata indica:
can provide capability
no:
did provide capability

## 247. FrankenPHP

Servicios long-lived permitidos:

- AssuranceModelRegistry
- CompiledAssuranceEvaluator
- CapabilityRegistry
- FederationMappingRegistry
- si son inmutables/stateless.

## 248. Request-local

AuthenticationContext
EvidenceSet
RiskContext
DeviceContext
TrustAssessment

## 249. Atomic Model Reload

Assurance Model V5
↓
atomic swap
Assurance Model V6

## 250. No Partial Reload

Una evaluación utiliza exactamente una versión.

## 251. Events

VoltStack deberá considerar:

```text
AuthenticationAssuranceEvaluated
AuthenticationAssuranceChanged
AuthenticationAssuranceDegraded
AuthenticationStepUpRequired
AuthenticationStepUpCompleted
AuthenticationStepDownApplied

AuthenticationTrustClassified
AuthenticationTrustChanged

AuthenticationContextCreated
AuthenticationContextReevaluated
AuthenticationContextRestricted

AuthenticationFactorIndependenceEvaluated

AssuranceModelActivated
AssuranceModelSuperseded
AssuranceModelVersionDriftDetected
```

## 252. Audit

Para operaciones sensibles podrá registrar:

- principal reference
- operation
- achieved assurance
- effective assurance
- required assurance
- capabilities
- context class
- trust classification
- model version
- reason codes

## 253. No Secret Audit

Nunca:

- password
- OTP
- private key
- raw token
- WebAuthn secrets

## 254. Metrics

auth_assurance_evaluation_total
auth_assurance_level_total
auth_assurance_degradation_total

auth_step_up_required_total
auth_step_up_completed_total
auth_step_down_total

auth_trust_classification_total
auth_context_re_evaluation_total

auth_factor_independence_unknown_total

auth_assurance_model_version
auth_assurance_model_drift_total

## 255. Cardinality

Labels aceptables:

- assurance_level
- principal_type
- context_class
- trust_class
- factor_class

## 256. Avoid

user_id
email
session_id
credential_id
como labels.

## 257. Tracing

Spans:

- auth.assurance.evaluate
- auth.assurance.capabilities
- auth.assurance.factor_independence
- auth.assurance.effective
- auth.trust.classify
- auth.context.build
- auth.context.reevaluate
- auth.federation.assurance_map

## 258. Failure Taxonomy

AUTH_ASSURANCE_EVALUATION_FAILED
AUTH_ASSURANCE_MODEL_NOT_FOUND
AUTH_ASSURANCE_MODEL_INVALID
AUTH_ASSURANCE_MODEL_VERSION_MISMATCH

AUTH_CAPABILITY_UNKNOWN
AUTH_CAPABILITY_CONFLICT
AUTH_CAPABILITY_UNVERIFIED

AUTH_FACTOR_INDEPENDENCE_UNKNOWN
AUTH_FACTOR_DEPENDENCY_CONFLICT

AUTH_CONTEXT_INVALID
AUTH_CONTEXT_STALE
AUTH_CONTEXT_VERSION_UNSUPPORTED

AUTH_TRUST_CLASSIFICATION_FAILED

AUTH_FEDERATED_ASSURANCE_MAPPING_FAILED
AUTH_FEDERATED_ASSURANCE_UNTRUSTED

AUTH_EFFECTIVE_ASSURANCE_INSUFFICIENT
AUTH_ASSURANCE_REEVALUATION_REQUIRED

## 259. Fail Closed

Si una operación exige HIGH y evaluator no puede determinar assurance:
UNKNOWN
no debe interpretarse como:
HIGH

## 260. Unknown Semantics

Regla general:
En Authentication Assurance, ausencia de evidencia no constituye evidencia positiva.

## 1. Testing Strategy

Debe cubrir:

- unit
- composition
- property-based
- integration
- federation
- machine identity
- session
- distributed
- FrankenPHP
- security
- performance

## 2. Password Tests

Password no obtiene phishing resistance.

## 3. OTP Tests

OTP no recibe propiedades no verificadas.

## 4. Passkey Tests

Passkey recibe únicamente capabilities demostradas por ceremony/result.

## 5. Hardware Tests

Hardware backing UNKNOWN no equivale a VERIFIED.

## 6. MFA Tests

Factor independence se verifica.

## 7. Duplicate Evidence Tests

La misma evidencia no se cuenta dos veces.

## 8. Replay Tests

Reutilizar una proof consumida no aumenta assurance.

## 9. Freshness Tests

Assurance-sensitive operations respetan factor freshness.

## 10. Session Age Tests

Session age no se confunde con factor freshness.

## 11. Step-Up Tests

Nueva evidencia aumenta capabilities correctamente.

## 12. Step-Down Tests

Timeout/risk puede reducir effective assurance.

## 13. Recovery Tests

Recovery context no obtiene full assurance automáticamente.

## 14. Break-Glass Tests

Break-glass no se clasifica automáticamente HighlyTrusted.

## 15. Device Tests

Managed device no sustituye user authentication.

## 16. Compromised Device Tests

Strong Authentication puede coexistir con low trust.

## 17. Federation Tests

ACR/AMR no confiables no elevan assurance.

## 18. Unknown ACR Tests

No produce HIGH.

## 19. Federation Freshness Tests

auth_time correctamente validado.

## 20. Machine Tests

No aplica reglas MFA humanas inapropiadas.

## 21. Workload Tests

Attestation freshness participa correctamente.

## 22. Delegation Tests

Actor assurance no se copia automáticamente al subject.

## 23. Tenant Tests

Tenant puede endurecer requisitos, no redefinir security floor.

## 24. Realm Tests

Cambios de realm disparan reevaluación cuando corresponde.

## 25. Model Version Tests

Historical assurance conserva model version original.

## 26. Re-evaluation Tests

Modelo actual puede declarar session histórica insuficiente.

## 27. Cache Tests

Effective assurance no se reutiliza con security epoch incompatible.

## 28. Distributed Tests

Version drift detectado.

## 29. FrankenPHP Tests

No hay leakage de context entre requests.

## 30. Fiber Tests

Dos AuthenticationContexts concurrentes permanecen aislados.

## 31. Determinism Tests

Mismo:

- evidence
- context snapshot
- model version
- clock instant
- produce mismo assessment.

## 32. Property-Based Tests

Especialmente:

- factor combinations
- capability composition
- freshness
- step-up
- degradation

## 33. Security Invariants — Assurance

AUTH-ASSURANCE-01
Authentication Assurance nunca se reduce conceptualmente a authenticated=true.
AUTH-ASSURANCE-02
Assurance no equivale a Authorization privilege.

- AUTH-ASSURANCE-03
- Authentication Method no determina por sí solo el Assurance Level.
- AUTH-ASSURANCE-04

Assurance se deriva de evidence y propiedades verificadas.
AUTH-ASSURANCE-05
Los niveles agregados conservan un perfil multidimensional.

## 34. Security Invariants — Capabilities

AUTH-CAP-01
UNKNOWN != VERIFIED.
AUTH-CAP-02
Una capability solo satisface requirement si su estado de verificación es aceptable.
AUTH-CAP-03
El soporte potencial de una capability no demuestra que ocurrió.
AUTH-CAP-04
Passkey no implica automáticamente hardware backing.
AUTH-CAP-05
MFA no implica automáticamente phishing resistance.

## 35. Security Invariants — Factors

AUTH-FACTOR-01
Dos métodos no implican automáticamente dos factores independientes.
AUTH-FACTOR-02
La misma evidencia nunca se contabiliza múltiples veces para elevar assurance.
AUTH-FACTOR-03
Dependencias entre factores pueden reducir la garantía de la combinación.
AUTH-FACTOR-04
Factor independence desconocida no se trata como independencia verificada en operaciones críticas.

## 36. Security Invariants — Freshness

AUTH-FRESH-01
Session Age y Authentication Freshness son conceptos distintos.
AUTH-FRESH-02
Freshness puede evaluarse por factor/capability.

- AUTH-FRESH-03
- High Assurance histórico no implica Fresh High Assurance actual.
- AUTH-FRESH-04

Assurance puede degradarse con el tiempo.

## 37. Security Invariants — Trust

AUTH-TRUST-01
Trust Classification y Authentication Assurance son distintos.
AUTH-TRUST-02
Strong Authentication no neutraliza un estado de identidad bloqueado.
AUTH-TRUST-03
Managed Device no equivale a authenticated user.

- AUTH-TRUST-04
- Break-glass no equivale a maximum trust.
- AUTH-TRUST-05

Trust puede ser relativo a operación/contexto.

## 38. Security Invariants — Federation

AUTH-FED-ASSURANCE-01
ACR/AMR externos no se aceptan sin issuer trust y validación criptográfica.
AUTH-FED-ASSURANCE-02
Mappings de assurance son explícitos y versionados.

- AUTH-FED-ASSURANCE-03
- Unknown external assurance no se eleva automáticamente.
- AUTH-FED-ASSURANCE-04

Assurance externo conserva provenance.

## 39. Security Invariants — Machine

AUTH-MACHINE-ASSURANCE-01
Machine identities no emulan MFA humano artificialmente.

- AUTH-MACHINE-ASSURANCE-02
- Machine assurance utiliza propiedades apropiadas para workloads.
- AUTH-MACHINE-ASSURANCE-03

Actor assurance no se transfiere automáticamente mediante delegation.
AUTH-MACHINE-ASSURANCE-04
Workload attestation stale no se considera current assurance.

## 40. Security Invariants — Runtime

AUTH-CONTEXT-RT-01
AuthenticationContext es request/fiber scoped.

- AUTH-CONTEXT-RT-02
- Context snapshots son inmutables.
- AUTH-CONTEXT-RT-03

Cada evaluation usa una versión coherente del Assurance Model.
AUTH-CONTEXT-RT-04
Context cache respeta Security Epoch.
AUTH-CONTEXT-RT-05
FrankenPHP workers no conservan Identity Context entre requests.

## 41. Anti-Patterns

No implementar:

```php
authenticated=true
→ all authenticated sessions equivalent
```

## 42. Anti-Pattern

admin role
→ high authentication assurance

## 43. Anti-Pattern

passkey
→ hardware-backed
sin evidence.

## 44. Anti-Pattern

MFA
→ phishing resistant

## 45. Anti-Pattern

two methods
→ independent MFA

## 46. Anti-Pattern

trusted device
→ authenticated identity

## 47. Anti-Pattern

low risk
→ high assurance
Risk y assurance son dimensiones diferentes.

## 48. Anti-Pattern

high assurance
→ low risk
Tampoco necesariamente.

## 49. Anti-Pattern

session age
=
auth freshness

## 50. Anti-Pattern

once HIGH
=
always HIGH

## 51. Anti-Pattern

recovery success
=
normal security context

## 52. Anti-Pattern

break-glass
=
most trusted context

## 53. Anti-Pattern

OIDC acr
→ blindly trust

## 54. Anti-Pattern

actor assurance
→ copy to delegated service

## 55. Anti-Pattern

trust score 93
→ secure
sin semántica.

## 56. Anti-Pattern

AuthenticationContext singleton mutable
especialmente bajo FrankenPHP.

## 57. Componentes principales

AuthenticationAssuranceLevel
AuthenticationAssuranceProfile
AuthenticationAssuranceEvaluator
EffectiveAuthenticationAssuranceEvaluator

AuthenticationCapability
AuthenticationCapabilitySet
AuthenticationCapabilityEvidence

AuthenticationEvidenceAssessment
EvidenceStrength

AuthenticationFactorClass
FactorComposition
FactorIndependenceEvaluator

AuthenticationFreshness

AuthenticationContext
AuthenticationContextBuilder
AuthenticationContextResolver
AuthenticationContextReevaluationService

TrustClassification
AuthenticationTrustAssessment
AuthenticationTrustClassifier

FederatedAssuranceMapper
AuthenticationAssuranceModel
AuthenticationAssuranceModelRegistry
AuthenticationAssuranceExplanation

## 318. Namespace sugerido

VoltStack\Quantum\Auth\Assurance

VoltStack\Quantum\Auth\Assurance\Contracts
VoltStack\Quantum\Auth\Assurance\Level
VoltStack\Quantum\Auth\Assurance\Capability
VoltStack\Quantum\Auth\Assurance\Evidence
VoltStack\Quantum\Auth\Assurance\Factor
VoltStack\Quantum\Auth\Assurance\Freshness
VoltStack\Quantum\Auth\Assurance\Trust
VoltStack\Quantum\Auth\Assurance\Context
VoltStack\Quantum\Auth\Assurance\Federation
VoltStack\Quantum\Auth\Assurance\Machine
VoltStack\Quantum\Auth\Assurance\Model
VoltStack\Quantum\Auth\Assurance\Runtime

## 319. Estructura sugerida

src/Quantum/Auth/Assurance/
├── Contracts/
│   ├── AuthenticationAssuranceEvaluatorInterface.php
│   ├── EffectiveAuthenticationAssuranceEvaluatorInterface.php
│   ├── AuthenticationTrustClassifierInterface.php
│   ├── FactorIndependenceEvaluatorInterface.php
│   ├── AuthenticationContextBuilderInterface.php
│   ├── AuthenticationContextResolverInterface.php
│   ├── AuthenticationContextReevaluationServiceInterface.php
│   └── FederatedAssuranceMapperInterface.php
│
├── Level/
│   ├── AuthenticationAssuranceLevel.php
│   ├── AuthenticationAssuranceProfile.php
│   ├── EffectiveAuthenticationAssurance.php
│   └── AuthenticationAssuranceExplanation.php
│
├── Capability/
│   ├── AuthenticationCapability.php
│   ├── AuthenticationCapabilitySet.php
│   ├── AuthenticationCapabilityEvidence.php
│   └── CapabilityVerificationState.php
│
├── Evidence/
│   ├── AuthenticationEvidenceAssessment.php
│   ├── EvidenceStrength.php
│   └── EvidenceSource.php
│
├── Factor/
│   ├── AuthenticationFactorClass.php
│   ├── FactorComposition.php
│   ├── FactorIndependenceAssessment.php
│   ├── FactorDependencyGraph.php
│   └── FactorIndependenceEvaluator.php
│
├── Freshness/
│   ├── AuthenticationFreshness.php
│   ├── FactorFreshness.php
│   └── ContextSignalFreshness.php
│
├── Trust/
│   ├── TrustClassification.php
│   ├── AuthenticationTrustAssessment.php
│   ├── AuthenticationTrustContext.php
│   └── AuthenticationTrustClassifier.php
│
├── Context/
│   ├── AuthenticationContext.php
│   ├── AuthenticationContextClass.php
│   ├── AuthenticationContextBuilder.php
│   ├── AuthenticationContextResolver.php
│   ├── AuthenticationContextInput.php
│   └── AuthenticationContextReevaluationService.php
│
├── Federation/
│   ├── FederatedAssuranceMapper.php
│   ├── FederationAssuranceMappingProfile.php
│   └── ExternalAssuranceReference.php
│
├── Machine/
│   ├── MachineAuthenticationContext.php
│   └── WorkloadAssuranceProfile.php
│
├── Model/
│   ├── AuthenticationAssuranceModel.php
│   ├── AuthenticationAssuranceModelRegistry.php
│   ├── AssuranceModelVersion.php
│   └── CompiledAssuranceModel.php
│
└── Runtime/
├── DefaultAuthenticationAssuranceEvaluator.php
├── DefaultEffectiveAuthenticationAssuranceEvaluator.php
└── AuthenticationAssuranceRuntimeResetter.php

## 320. Configuración conceptual

return [

'authentication' => [

'assurance' => [

'model' => 'default',

'levels' => [
'low',
'standard',
'strong',
'high',
'privileged',
],

'capabilities' => [
'phishing_resistant',
'user_verification',
'replay_resistant',
'hardware_backed',
'device_bound',
],

'freshness' => [
'track_per_factor' => true,
],

'trust' => [
'classification' => true,
],

'federation' => [
'require_explicit_mapping' => true,
],

'runtime' => [
'compiled' => true,
'cache' => true,
],

],

],

];

## 321. Developer Experience

Ejemplo:

```php
$context = Auth::context();

$context->assurance()->level();
$context->assurance()->capabilities();
$context->freshness();
$context->trust();
```

## 322. Capability Query

if ($context->assurance()->has(
AuthenticationCapability::PhishingResistant
)) {
// ...
}
Para lógica de seguridad de aplicación, preferir normalmente Policy Engine en vez de condiciones manuales.

## 323. Requirement

AuthPolicy::for('tenant.delete')
->require(
Assurance::high(),
Capability::phishingResistant(),
FreshAuth::within(minutes: 5),
);

## 324. Context Example

Principal:
Human/User #451

Context:
AUTHENTICATED

Methods:

```text
Password
Passkey
```

Factors:

```text
Knowledge
Possession
```

Capabilities:

```text
MultiFactor             VERIFIED
UserVerification        VERIFIED
PhishingResistant       VERIFIED
ReplayResistant         VERIFIED
HardwareBacked          UNKNOWN
```

Achieved Assurance:
HIGH

Freshness:

```text
Passkey 2m
Password 18m
```

Device:
KNOWN

Risk:
LOW

Trust:
TRUSTED

## 325. Example con contexto degradado

Achieved Assurance:
HIGH

Device:
COMPROMISED

Risk:
HIGH

Effective Trust:
UNTRUSTED

Effective Authentication:
RESTRICTED

Required Action:

```php
Step-up alone is insufficient;
device/security remediation required.
```

Esto demuestra por qué:

- Assurance
- Trust
- Risk
- Security Posture

no deben convertirse en un único número.

## 326. Integración final con Policy Engine

EVIDENCE
│
▼
ASSURANCE EVALUATOR
│
▼
Assurance Profile
│
┌──────────────┼──────────────┐
▼              ▼              ▼
Freshness         Risk          Posture
│              │              │
└──────────────┼──────────────┘
▼
TRUST CLASSIFIER
│
▼
Authentication Context
│
▼
POLICY ENGINE (36)
│
▼
Effective Requirement
│
▼
Satisfaction Evaluation
│
┌──────────────┼──────────────┐
▼              ▼              ▼
ALLOW         STEP-UP          DENY
│
▼
NEW EVIDENCE
│
└──────► Assurance Re-evaluation

## 327. Comparación conceptual con Laravel

Laravel proporciona una excelente abstracción pragmática de:

- authenticated user
- guards
- sessions
- password confirmation
- MFA through ecosystem/Fortify

VoltStack conservará una DX similar, pero añadirá un modelo formal para representar:

- how
- when
- with what
- how strongly
- under what context

fue autenticado el principal.
En vez de depender únicamente de:
Auth::check();
VoltStack podrá conceptualmente responder:
Auth::context()->assurance();
mientras las decisiones reales de seguridad permanecerán centralizadas en Policy Engine.

## 328. Comparación conceptual con Symfony

Symfony ofrece piezas muy adecuadas para representar información de Authentication mediante:

- Authenticators
- Passports
- Badges
- Tokens
- Firewalls

VoltStack toma esa filosofía de composición y añade un modelo explícito de:

- Evidence
- Capabilities
- Factor Independence
- Assurance Profiles
- Freshness
- Trust Classification
- Context Re-evaluation

que podrá ser compartido consistentemente por todo el framework.

## 329. Diferenciador VoltStack

La arquitectura resultante será:

- Laravel-like Developer Experience
- +
- Symfony-like Authentication Composition
- +
- Explicit Evidence Model
- +
- Capability-Based Assurance
- +
- Factor Independence
- +
- Per-Factor Freshness
- +

Achieved vs Effective Assurance
+
Trust Classification
+
Security Posture
+
Federated Assurance Mapping
+
Human + Machine Assurance
+
Step-Up / Step-Down
+
Versioned Assurance Models
+
Distributed Consistency
+
FrankenPHP-safe Runtime

## 330. Decisiones arquitectónicas definitivas

VoltStack adoptará las siguientes reglas:

## 01. Authentication no será representada únicamente como boolean.

## 1. Authentication Assurance será independiente de Authorization.

## 2. Roles nunca determinarán Assurance Level.

## 3. Authentication Methods no tendrán un nivel universal fijo.

## 4. Assurance se derivará de evidence verificable.

## 5. Los niveles agregados conservarán un Assurance Profile.

## 6. Capabilities serán first-class.

## 7. UNKNOWN nunca equivaldrá a VERIFIED.

## 8. Potential capability no equivaldrá a demonstrated capability.

## 9. MFA no significará simplemente dos métodos.

## 10. Factor independence será evaluable.

## 11. Phishing resistance será propiedad explícita.

## 12. User Verification será propiedad explícita.

## 13. Hardware backing será propiedad explícita.

## 14. Authentication Freshness será first-class.

## 15. Freshness podrá existir por factor/capability.

## 16. Session Age y Authentication Freshness serán distintos.

## 17. Achieved Assurance y Effective Assurance serán distintos.

## 18. Assurance podrá degradarse.

## 19. High Assurance no será permanente.

## 20. Trust Classification será distinta de Assurance.

## 21. Security Posture será distinta de Trust.

## 22. Risk será distinto de Assurance.

## 23. Strong Authentication podrá coexistir con High Risk.

## 24. Strong Authentication podrá coexistir con Untrusted Context.

## 25. Device Trust no sustituirá Identity Authentication.

## 26. Authentication Context será un snapshot explícito.

## 27. Authentication Context será immutable.

## 28. Authentication Context será request/fiber scoped.

## 29. Context tendrá provenance cuando sea relevante.

## 30. Context podrá contener múltiples classifications.

## 31. Restricted Authentication será first-class.

## 32. Recovery Context no heredará automáticamente normal trust.

## 33. Break-Glass no significará maximum trust.

## 34. Privileged Role no significará privileged assurance.

## 35. Step-Up añadirá nueva evidence.

## 36. Step-Down podrá reducir effective assurance sin logout.

## 37. Credential enrollment conservará binding assurance.

## 38. Federation assurance requerirá mappings explícitos.

## 39. ACR/AMR externos nunca se confiarán ciegamente.

## 40. External Assurance conservará provenance.

## 41. Machine Assurance utilizará propiedades de workloads.

## 42. Machines no emularán artificialmente MFA humano.

## 43. Delegation no transferirá assurance automáticamente.

## 44. Trust-domain transitions podrán requerir remapping.

## 45. Assurance Models serán versionados.

## 46. Historical assessments conservarán su model version.

## 47. Current policy podrá reevaluar sesiones históricas.

## 48. Model deployment será atómico.

## 49. Distributed version drift será observable.

## 50. Cache respetará model version y Security Epoch.

## 51. FrankenPHP nunca conservará AuthenticationContext mutable

entre requests.

## 52. Assurance evaluation será auditable y explicable.

53. Absence of evidence nunca se interpretará como positive proof.
54. Criterios de aceptación

El sistema se considerará completo cuando soporte:
55. Authentication Assurance Levels;
56. Assurance Profiles;
57. achieved assurance;
58. effective assurance;
59. Authentication Capabilities;
60. capability verification states;
61. evidence assessments;
62. evidence strength;
63. factor classes;
64. factor independence;
65. dependency detection;
66. MFA composition;
67. phishing resistance;
68. replay resistance;
69. user presence;
70. user verification;
71. hardware backing;
72. device binding;
73. per-factor freshness;
74. assurance degradation;
75. step-up;
76. step-down;
77. Authentication Context;
78. context classifications;
79. context provenance;
80. Security Posture integration;
81. Risk integration;
82. Device Trust integration;
83. Trust Classification;
84. restricted contexts;
85. recovery contexts;
86. privileged contexts;
87. break-glass contexts;
88. impersonation contexts;
89. federation assurance mapping;
90. ACR/AMR mapping;
91. machine assurance;
92. workload assurance;
93. delegation-aware contexts;
94. credential enrollment assurance;
95. Assurance Model versioning;
96. simulation;
97. shadow evaluation;
98. model deployment;
99. atomic hot reload;
100. distributed consistency;
101. drift detection;
102. caching;
103. observability;
104. audit;
105. FrankenPHP isolation;
106. Fiber safety;
107. deterministic evaluation;
108. security testing;
109. performance testing.
110. Regla arquitectónica final

VoltStack deberá mantener esta separación:

```text
Authentication Method
        │
        ▼
Authentication Evidence
        │
        ▼
Evidence Capabilities
        │
        ▼
Factor Composition
        │
        ▼
Achieved Assurance
        │
        ├──────────────┐
        │              │
        ▼              ▼
    Freshness       Security Posture
        │              │
        ├──────┬───────┤
        │      │       │
        ▼      ▼       ▼
      Risk   Device   Identity
        │      │       │
        └──────┼───────┘
               ▼
        Effective Assurance
               │
               ▼
       Trust Classification
               │
               ▼
      Authentication Context
               │
               ▼
       Authentication Policy
               │
               ▼
      Operation Requirement
               │
        ┌──────┼───────┐
        ▼      ▼       ▼
      ALLOW  STEP-UP   DENY
```

La regla fundamental será:
VoltStack nunca preguntará únicamente si un principal está autenticado; podrá determinar qué evidencia presentó, qué propiedades fueron verificadas, cuándo ocurrió, qué assurance produjo, cuál sigue siendo efectivo y qué nivel de confianza merece ese contexto para la operación solicitada.

Con esto, 36 y 37 forman una pareja arquitectónica fundamental:

```text
36
"What authentication is required?"

37
"What authentication has actually been demonstrated?"
```

y la decisión final surge de comparar ambas respuestas.
Siguiente documento
El siguiente apartado contemplado es:
`38_AUTHENTICATION_CHALLENGE_NEGOTIATION_CONTINUATION_AND_INTERACTIVE_FLOW_SYSTEM.md`
Su función será formalizar cómo VoltStack pasa de:

```text
CURRENT AUTHENTICATION CONTEXT
            │
            ▼
   Requirement Unsatisfied
            │
            ▼
        CHALLENGE
```

a seleccionar y negociar de manera segura:

- Password
- Passkey
- TOTP
- Federated Reauthentication
- Recovery
- Step-Up
- Alternative Method

manteniendo una Authentication Transaction coherente a través de HTTP, SPA, APIs y FrankenPHP, sin mezclar todavía la protección criptográfica del estado, nonces y anti-replay, que corresponderá específicamente al documento 39.
