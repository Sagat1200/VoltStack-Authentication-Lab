# VoltStack Authentication System

## 15 — Multi-Factor Authentication, Factor Orchestration and Step-Up System

- **Archivo:** `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del subsistema de factores, MFA, challenge orchestration y step-up authentication

**Depende especialmente de:**

- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema encargado de combinar múltiples pruebas de autenticación, determinar cuándo son necesarias, coordinar challenges adicionales y elevar el nivel de confianza de un `AuthenticationContext` ya existente.

El sistema deberá cubrir:

```text
Multi-Factor Authentication
Two-Factor Authentication
Step-Up Authentication
Factor Enrollment
Factor Verification
Factor Selection
Factor Policies
Authentication Assurance
Fresh Authentication
Adaptive MFA
Risk-triggered MFA
Trusted Devices
Recovery Factors
Factor Lifecycle
```

La meta arquitectónica será evitar implementar MFA como una condición aislada:

```php
if ($user->two_factor_secret) {
    // ask OTP
}
```

y convertirlo en un sistema general de composición de evidencia.

---

## 2. Principio fundamental

VoltStack distinguirá:

```text
Authentication Method
Authentication Factor
Authentication Evidence
Authentication Assurance
Authentication Requirement
Authentication Challenge
Step-Up Authentication
```

Por tanto:

> **MFA no será un Authenticator concreto; será una política de composición de evidencia y assurance.**

---

## 3. Modelo general

```text
Initial Authentication
        │
        ▼
Primary Evidence
        │
        ▼
Authentication Requirement Evaluation
        │
   ┌────┴────┐
   │         │
SATISFIED   MORE EVIDENCE REQUIRED
   │         │
   │         ▼
   │   Factor Orchestrator
   │         │
   │         ▼
   │     Challenge
   │         │
   │         ▼
   │   Factor Verification
   │         │
   │         ▼
   │     Factor Evidence
   │         │
   └─────────┴─────────┐
                       ▼
              Evidence Composition
                       │
                       ▼
             Assurance Calculation
                       │
                       ▼
             AuthenticationDecision
```

---

## 4. Qué es un Factor

Un `AuthenticationFactor` representa una categoría de evidencia utilizada para demostrar control o presencia de una Identity.

Categorías clásicas:

```text
KNOWLEDGE
POSSESSION
INHERENCE
```

VoltStack podrá extenderlas con atributos más precisos.

---

## 5. Knowledge Factor

Ejemplos:

```text
password
PIN
memorized secret
```

---

## 6. Possession Factor

Ejemplos:

```text
TOTP seed/device
hardware key
passkey authenticator
registered mobile device
smart card
```

---

## 7. Inherence Factor

Ejemplos:

```text
biometric evidence
```

VoltStack no deberá asumir que toda biometría es gestionada directamente por el framework.

Una Passkey, por ejemplo, puede indicar que el authenticator local realizó user verification biométrica sin que VoltStack reciba el dato biométrico.

---

## 8. FactorCategory

Conceptualmente:

```php
enum FactorCategory: string
{
    case KNOWLEDGE = 'knowledge';
    case POSSESSION = 'possession';
    case INHERENCE = 'inherence';
}
```

Podrá usarse un Value Object extensible si se necesita plugin extensibility.

---

## 9. Factor no equivale a Credential

Una `Credential` es el material presentado.

Un `Factor` describe qué dimensión de confianza aporta la verificación.

Ejemplo:

```text
PasswordCredential
        ↓
VerifiedCredential
        ↓
Knowledge Factor Evidence
```

---

## 10. FactorEvidence

Se recomienda modelar:

```text
VerifiedFactor
```

independientemente de `VerifiedCredential`.

---

## 11. VerifiedFactor model

Conceptualmente:

```php
final readonly class VerifiedFactor
{
    public function __construct(
        public FactorType $type,
        public FactorCategory $category,
        public \DateTimeImmutable $verifiedAt,
        public FactorStrength $strength,
        public FactorBinding $binding,
        public array $attributes = [],
    ) {}
}
```

---

## 12. FactorType

Tipos iniciales:

```text
password
totp
hotp
passkey
security_key
email_otp
sms_otp
push_approval
recovery_code
federated_mfa
trusted_device
client_certificate
```

Debe ser extensible.

---

## 13. FactorCategory vs FactorType

Ejemplo:

```text
type = password
category = knowledge
```

```text
type = passkey
category = possession
```

y dependiendo de metadata:

```text
user_verified = true
```

podría aportar propiedades adicionales de assurance.

---

## 14. FactorStrength

No deberá reducirse a:

```text
weak
strong
```

Podrá modelar propiedades:

```text
phishing_resistant
hardware_backed
device_bound
user_verified
one_time
out_of_band
shared_secret
federated
```

---

## 15. Factor independence

Uno de los problemas principales de MFA es asumir que:

```text
two credentials
=
two independent factors
```

Esto es falso.

---

## 16. Ejemplo de no independencia

```text
password
+
security question
```

ambos son:

```text
knowledge
```

y no constituyen necesariamente MFA independiente.

---

## 17. Otro ejemplo

```text
password
+
email OTP
```

si el email account puede recuperarse usando la misma password comprometida, la independencia real puede ser menor.

---

## 18. FactorIndependencePolicy

VoltStack deberá poder evaluar:

```text
factor categories
credential origins
device bindings
provider relationships
recovery dependencies
```

---

## 19. Independent Factor Requirement

Una policy podría exigir:

```text
at least 2 distinct FactorCategories
```

---

## 20. Pero no siempre

Passkeys modernas pueden proporcionar propiedades fuertes que hacen simplista contar únicamente categorías tradicionales.

Por eso VoltStack necesitará un:

```text
AuthenticationAssuranceCalculator
```

más expresivo.

---

## 21. Authentication Assurance

Representará el grado de confianza alcanzado por una cadena de autenticación.

---

## 22. Assurance no será simplemente “MFA true/false”

Evitar:

```php
$context->mfa = true;
```

como modelo completo.

Preferir:

```text
AuthenticationAssurance
```

con:

```text
level
properties
verified factors
freshness
provenance
```

---

## 23. AssuranceLevel

Puede modelarse conceptualmente como:

```text
AAL0
AAL1
AAL2
AAL3
```

o nombres propios.

---

## 24. No hardcodear estándares externos

VoltStack podrá mapear perfiles internos a estándares externos, pero no deberá asumir que cualquier combinación satisface automáticamente NIST/otro estándar sin policy formal.

---

## 25. AuthenticationAssurance model

Conceptualmente:

```php
final readonly class AuthenticationAssurance
{
    public function __construct(
        public AssuranceLevel $level,
        public AssurancePropertySet $properties,
        public \DateTimeImmutable $establishedAt,
    ) {}
}
```

---

## 26. Assurance Properties

Ejemplos:

```text
multi_factor
phishing_resistant
hardware_backed
device_bound
fresh
federated
user_verified
holder_of_key
```

---

## 27. Assurance is derived

Nunca deberá venir directamente del cliente:

```text
aal = 3
```

ni aceptarse ciegamente desde claims no verificadas.

---

## 28. AuthenticationAssuranceCalculator

Contrato:

```php
interface AuthenticationAssuranceCalculatorInterface
{
    public function calculate(
        AuthenticationEvidence $evidence,
        AssuranceCalculationContext $context
    ): AuthenticationAssurance;
}
```

---

## 29. Inputs

Podrá considerar:

```text
verified credentials
verified factors
factor independence
factor strength
provider trust
device binding
authentication freshness
token provenance
federated assurance
risk-adjustment policy
```

---

## 30. Calculator no autoriza

No deberá decidir:

```text
may transfer money
```

Solo produce características de Authentication.

---

## 31. Authentication Requirements

Una operación podrá exigir:

```text
minimum assurance
specific factor
specific property
fresh factor
factor category
factor count
```

---

## 32. AuthenticationRequirement

Conceptualmente:

```php
interface AuthenticationRequirementInterface
{
    public function evaluate(
        AuthenticationContext $context,
        AuthenticationRequirementContext $requirement
    ): AuthenticationRequirementResult;
}
```

---

## 33. Requirements examples

```text
MinimumAssuranceRequirement
FreshAuthenticationRequirement
SpecificFactorRequirement
PhishingResistantRequirement
HardwareBackedRequirement
MultiFactorRequirement
UserVerificationRequirement
```

---

## 34. Requirement composition

Podrán componerse:

```text
AND
OR
ANY_OF
ALL_OF
AT_LEAST_N
```

---

## 35. Ejemplo

Una operación crítica:

```text
minimum AAL2
AND
fresh within 5 minutes
AND
phishing_resistant
```

---

## 36. Step-Up Authentication

Step-Up significa elevar un `AuthenticationContext` existente para satisfacer requisitos adicionales.

---

## 37. Ejemplo

Context actual:

```text
password
AAL1
authenticated 30 minutes ago
```

Operación requiere:

```text
AAL2
```

Resultado:

```text
STEP_UP_REQUIRED
```

---

## 38. Step-up flow

```text
Existing AuthenticationContext
        ↓
Requirement Evaluation
        ↓
Insufficient
        ↓
StepUpPlanner
        ↓
Available Factor Selection
        ↓
AuthenticationTransaction
        ↓
Challenge
        ↓
Factor Verification
        ↓
Evidence Extension
        ↓
Assurance Recalculation
        ↓
New AuthenticationContext
```

---

## 39. Step-up does not discard identity

El sistema ya conoce:

```text
Identity
```

pero necesita más prueba.

---

## 40. Step-up is not a new login necessarily

Puede preservar:

```text
Identity
session lineage
tenant
base AuthenticationContext
```

mientras añade Evidence.

---

## 41. StepUpRequirementResolver

Deberá calcular qué falta.

Ejemplo:

```text
Current:
    password
    AAL1

Required:
    AAL2 + possession

Missing:
    possession factor
```

---

## 42. StepUpPlan

Conceptualmente:

```php
final readonly class StepUpPlan
{
    public function __construct(
        public AuthenticationRequirementSet $requirements,
        public FactorSelectionSet $acceptableFactors,
        public \DateTimeImmutable $expiresAt,
    ) {}
}
```

---

## 43. Factor Orchestrator

Será uno de los componentes centrales.

---

## 44. FactorOrchestrator responsibilities

```text
resolve available factors
evaluate factor policy
select acceptable factors
create challenge
manage transaction
process verification
compose factor evidence
handle retries
handle fallback
calculate remaining requirements
complete step-up
```

---

## 45. FactorRegistry

Registrará tipos y handlers.

---

## 46. FactorDescriptor

Podrá contener:

```text
type
category
verifier
challenge handler
enrollment handler
capabilities
strength properties
transport support
```

---

## 47. FactorVerifier

Contrato conceptual:

```php
interface FactorVerifierInterface
{
    public function verify(
        FactorVerificationRequest $request
    ): FactorVerificationResult;
}
```

---

## 48. FactorVerificationResult

```text
VERIFIED
INVALID
EXPIRED
REPLAYED
UNAVAILABLE
LOCKED
ERROR
```

---

## 49. Verified Factor

Solo `VERIFIED` produce:

```text
VerifiedFactor
```

---

## 50. Factor Challenge

Algunos factores necesitan challenge previo.

Ejemplos:

```text
WebAuthn
Push approval
Email OTP
SMS OTP
HOTP/TOTP prompt transaction
```

---

## 51. AuthenticationChallenge

Conceptualmente:

```php
final readonly class AuthenticationChallenge
{
    public function __construct(
        public AuthenticationChallengeId $id,
        public FactorType $factor,
        public AuthenticationTransactionId $transaction,
        public \DateTimeImmutable $expiresAt,
        public ChallengePayload $payload,
    ) {}
}
```

---

## 52. ChallengePayload

Solo deberá contener información segura para presentar al cliente.

No secrets server-side.

---

## 53. Challenge State

El estado privado se almacenará en:

```text
AuthenticationTransaction
```

o challenge repository.

---

## 54. Challenge TTL

Todo challenge deberá tener expiración adecuada.

---

## 55. Challenge one-time semantics

Cuando aplique:

```text
challenge consumed
```

no deberá reutilizarse.

---

## 56. Challenge binding

Debe vincularse a:

```text
Identity
AuthenticationTransaction
Factor
Tenant
Purpose
AuthenticationContext lineage
```

según necesidad.

---

## 57. Challenge mismatch

Cualquier mismatch deberá producir:

```text
INVALID_CHALLENGE
```

---

## 58. MFA Transaction

El flow multi-request vivirá en:

```text
AuthenticationTransaction
```

---

## 59. MFA transaction contains

```text
identity reference
base evidence snapshot
requirements
completed factors
pending factors
challenge references
expiration
attempt state
tenant
firewall
```

---

## 60. No raw password in transaction

Nunca.

---

## 61. No raw OTP persistence unless strictly required

Preferir referencias/hash/ephemeral challenge state.

---

## 62. Factor selection

Cuando varios métodos están disponibles:

```text
TOTP
Passkey
Security Key
Recovery Code
```

el usuario o policy puede elegir.

---

## 63. FactorSelectionPolicy

Podrá determinar:

```text
preferred factor
allowed factors
forbidden factors
fallback order
risk restrictions
```

---

## 64. Preferred factor

Ejemplo:

```text
Passkey
    >
TOTP
    >
Recovery Code
```

no porque “priority” equivalga a assurance universal, sino por policy.

---

## 65. User choice

En ciertos flows se permitirá:

```text
Choose another method
```

---

## 66. Security restriction

No todos los factores deberán poder sustituir a todos los demás.

Ejemplo:

```text
operation requires phishing-resistant
```

Entonces:

```text
TOTP
SMS
Recovery Code
```

no satisfacen requirement.

---

## 67. Factor fallback

Debe ser explícito.

---

## 68. Invalid factor attempt

Ejemplo:

```text
TOTP selected
TOTP invalid
```

No deberá probar automáticamente Recovery Code en el mismo payload.

---

## 69. User-selected new factor

Puede iniciar una nueva attempt dentro de la misma transaction si policy lo permite.

---

## 70. TOTP

VoltStack deberá soportar Time-Based One-Time Password.

---

## 71. TOTP factor record

Podrá contener:

```text
credential id
identity
encrypted/protected secret reference
status
algorithm/profile
digits
period
createdAt
lastUsedCounter/time-step
```

---

## 72. TOTP secret storage

A diferencia de password hashes, el servidor necesita acceder al shared secret para verificar códigos.

Por tanto deberá almacenarse:

```text
encrypted/protected
```

no hasheado irreversiblemente.

---

## 73. Secret storage boundary

Usar:

```text
SecretProtector
Key Management
encrypted credential store
```

---

## 74. TOTPVerifier

Validará:

```text
code format
time step
allowed skew/window
secret
replay state
credential status
transaction binding where applicable
```

---

## 75. TOTP replay

Un código válido no debería aceptarse múltiples veces dentro de la misma ventana si el sistema puede evitarlo.

---

## 76. LastAcceptedTimeStep

Podrá persistirse:

```text
last accepted TOTP step
```

para replay resistance.

---

## 77. Concurrent TOTP verification

Debe utilizar actualización atómica para impedir doble uso.

---

## 78. TOTP window

El número de pasos aceptados alrededor del clock actual deberá ser configurable y reducido.

---

## 79. Clock

Usar:

```text
ClockInterface
```

---

## 80. TOTP enrollment

Flow:

```text
Authenticated Identity
    ↓
Fresh Authentication
    ↓
Generate TOTP Secret
    ↓
Provision QR / URI
    ↓
User submits code
    ↓
Verify ownership/configuration
    ↓
Activate factor
```

---

## 81. Do not activate unconfirmed TOTP

El secret generado no deberá convertirse en factor activo hasta una verificación inicial exitosa.

---

## 82. Pending factor

Estado:

```text
PENDING_ENROLLMENT
```

---

## 83. TOTP recovery implications

Eliminar TOTP es una operación sensible.

Puede requerir:

```text
fresh authentication
another factor
recovery process
```

---

## 84. HOTP

Podrá soportarse como factor adicional.

Diferencia:

```text
counter-based
```

en lugar de time-based.

---

## 85. HOTP state

Necesita counter sincronizado y replay prevention.

---

## 86. Email OTP

VoltStack podrá soportarlo, pero deberá modelarlo con menor assurance que factores resistentes a phishing.

---

## 87. Email OTP is not equivalent to strong possession automatically

Depende del security model del email account.

---

## 88. EmailOtpChallenge

Generará:

```text
random one-time code
expiry
attempt limit
transaction binding
```

---

## 89. OTP storage

No almacenar código en plaintext si no es necesario.

Podrá almacenarse digest para verificación.

---

## 90. Email code lifetime

Debe ser breve y configurable.

---

## 91. Email OTP rate limiting

Controlar:

```text
send frequency
verification attempts
resend abuse
address enumeration
```

---

## 92. SMS OTP

Podrá soportarse, pero deberá etiquetarse con propiedades de assurance apropiadas.

---

## 93. SMS risks

Incluyen:

```text
SIM swap
SS7 weaknesses
number recycling
social engineering
```

Por ello no deberá tratarse como equivalente a hardware-backed/passkey.

---

## 94. SMS factor policy

Aplicaciones de alta seguridad podrán deshabilitarlo.

---

## 95. Passkeys

Passkeys pueden actuar como:

```text
primary authentication
step-up factor
phishing-resistant possession factor
```

según flow.

---

## 96. Passkey integration

El factor system deberá reutilizar Evidence creada por `PasskeyAuthenticator` cuando ya exista.

No volver a verificar innecesariamente.

---

## 97. Passkey properties

Podrán aportar:

```text
phishing_resistant
user_verified
device_bound / synced
hardware characteristics where known
```

---

## 98. Security Keys

Una WebAuthn security key puede ofrecer un factor de alta assurance.

---

## 99. User Verification

Debe distinguirse:

```text
user presence
```

de:

```text
user verification
```

---

## 100. Passkey Requirement

Una operación podrá exigir:

```text
WebAuthn
AND
user_verified = true
```

---

## 101. Recovery Codes

Recovery Codes deberán modelarse como:

```text
recovery factor
```

no como MFA normal de alta confianza.

---

## 102. Recovery factor semantics

Un Recovery Code sirve para recuperar acceso cuando el factor normal no está disponible.

Por ello su uso puede:

```text
reduce assurance
invalidate trusted-device state
require factor re-enrollment
trigger notifications
```

---

## 103. Recovery Code one-time

Debe consumirse atómicamente.

---

## 104. Recovery code storage

Preferir hashes seguros apropiados a su entropía.

---

## 105. Recovery code generation

Generar suficientes bits de entropía y formato usable.

---

## 106. Recovery code display

Mostrar una sola vez o permitir regeneración, no recuperación plaintext desde DB.

---

## 107. Recovery Code use

Después de usar:

```text
code = CONSUMED
```

---

## 108. Recovery Code impact

Policy recomendada:

```text
mark authentication as recovery-origin
require enrollment/reconfirmation
reduce long-lived trust
```

---

## 109. Push Approval

Podrá soportarse mediante provider/plugin.

---

## 110. Push challenge

```text
server creates challenge
mobile app receives
user approves
signed response returns
```

---

## 111. Push fatigue

Simple approve/deny puede ser vulnerable a push bombing.

VoltStack deberá permitir métodos stronger:

```text
number matching
transaction details
cryptographic challenge
```

---

## 112. Factor Provider SPI

Plugins podrán agregar:

```text
Duo
custom hardware
smart cards
enterprise push
biometric provider
```

---

## 113. Federated MFA

Un IdP externo puede afirmar que realizó MFA.

---

## 114. Federated MFA must be verified semantically

VoltStack deberá conocer:

```text
issuer trusted?
claim authentic?
authentication methods reference trusted?
fresh enough?
mapped assurance profile?
```

---

## 115. No blind `amr` trust

Claims como:

```text
amr
acr
```

solo deben usarse después de token/assertion verification y mapping policy.

---

## 116. FederatedAssuranceMapper

Podrá convertir:

```text
issuer-specific assurance
```

a:

```text
VoltStack AuthenticationAssurance
```

---

## 117. Provider-specific mapping

Ejemplo:

```text
corporate IdP acr=high
    → AAL2 + managed_identity
```

solo si configurado.

---

## 118. Factor double counting

Si un OIDC assertion ya representa:

```text
password + TOTP
```

VoltStack no deberá contar el OIDC assertion y TOTP como factores independientes adicionales sin comprender provenance.

---

## 119. Evidence provenance graph

En sistemas avanzados podrá ser necesario representar:

```text
Federated Assertion
    derived from
        Password
        TOTP
```

para evitar double counting.

---

## 120. FactorEvidenceSource

Podrá identificar:

```text
LOCAL
FEDERATED
DEVICE
INFRASTRUCTURE
RECOVERY
```

---

## 121. MFA Requirement Policies

Podrán aplicarse por:

```text
Firewall
Route
Operation
Identity Type
Tenant
Risk
Resource sensitivity
```

---

## 122. Ejemplo Firewall

```text
admin
    minimum assurance = AAL2
```

---

## 123. Ejemplo Route

```text
/settings/profile
    AAL1

/settings/security
    fresh AAL2

/payments/wire
    fresh phishing-resistant AAL2
```

---

## 124. AuthenticationRequirementResolver

Combinará requirements desde distintas capas.

---

## 125. Requirement composition security

Una regla menos restrictiva no debe borrar otra más fuerte.

Ejemplo:

```text
Firewall: AAL2
Route: AAL1
```

Effective:

```text
AAL2
```

---

## 126. Tenant requirement

Tenant enterprise:

```text
all interactive users require MFA
```

---

## 127. Identity-specific requirement

High-risk admin:

```text
phishing-resistant factor required
```

---

## 128. Risk-triggered MFA

Risk Engine podrá producir:

```text
STEP_UP_REQUIRED
```

aunque la route normalmente permita AAL1.

---

## 129. Adaptive MFA

Factores requeridos pueden depender de:

```text
new device
unusual network
location change
impossible travel
credential compromise signal
sensitive transaction
```

---

## 130. Risk signal is not proof

Risk Engine solo altera requirements.

---

## 131. Risk cannot downgrade required assurance

Si route exige:

```text
AAL2
```

un risk score bajo no podrá bajar a AAL1 salvo policy explícita extremadamente controlada.

---

## 132. Trusted Devices

VoltStack podrá soportar el concepto:

```text
TrustedDeviceCredential
```

para reducir frecuencia de MFA.

---

## 133. Trusted device is not “MFA bypass”

Debe modelarse como una credencial/evidencia con propiedades limitadas.

---

## 134. Trusted device flow

```text
User completes strong MFA
        ↓
Policy allows trust
        ↓
Issue TrustedDeviceCredential
        ↓
future login
        ↓
device credential verified
        ↓
MFA requirement may be partially satisfied
```

---

## 135. TrustedDeviceCredential

Debe ser:

```text
revocable
rotatable
expiring
device-bound where possible
security-version-aware
```

---

## 136. Trusted Device vs Remember-Me

No son lo mismo.

```text
Remember-Me
    helps establish base Authentication

Trusted Device
    helps satisfy/reduce secondary-factor requirement
```

---

## 137. Both may coexist

Ejemplo:

```text
Remember-Me
    → base AAL1 session

TrustedDevice
    → secondary trusted-device evidence

Policy
    → may still require fresh password/passkey for sensitive action
```

---

## 138. Trusted device assurance

No deberá equivaler automáticamente a possession factor fuerte si solo es una bearer cookie.

---

## 139. Device credential theft

Debe considerarse en risk/rotation policy.

---

## 140. TrustedDevicePolicy

Podrá definir:

```text
lifetime
rotation
allowed origin factors
device binding
risk reaction
revocation
assurance contribution
```

---

## 141. Factor Enrollment

Registrar un factor es una operación sensible.

---

## 142. Enrollment requirements

Podrán incluir:

```text
existing AuthenticationContext
fresh authentication
minimum assurance
Authorization/self ownership
no unresolved compromise
```

---

## 143. FactorEnrollmentTransaction

Debe separar:

```text
factor preparation
verification
activation
```

---

## 144. Enrollment flow

```text
Authenticated Identity
      ↓
Freshness Check
      ↓
Enrollment Transaction
      ↓
Create pending FactorCredential
      ↓
User proves possession/control
      ↓
Activate factor
      ↓
CredentialVersion / FactorVersion update
      ↓
Audit
```

---

## 145. Factor status

Estados posibles:

```text
PENDING
ACTIVE
DISABLED
REVOKED
COMPROMISED
REPLACED
```

---

## 146. FactorCredential

Representa el material registrado para validar un factor.

Ejemplos:

```text
TOTP secret
WebAuthn public credential
trusted device record
push device key
```

---

## 147. FactorCredentialRepository

Contrato general opcional:

```php
interface FactorCredentialRepositoryInterface
{
    public function findAvailableFor(
        IdentityReference $identity
    ): FactorCredentialSet;
}
```

---

## 148. Different stores

No todos los factores tienen que usar el mismo repository físico.

---

## 149. Factor discovery

El Orchestrator deberá saber qué factores activos tiene la Identity.

---

## 150. FactorAvailability

Podrá representar:

```text
AVAILABLE
UNAVAILABLE
TEMPORARILY_UNAVAILABLE
REQUIRES_ENROLLMENT
DISABLED
```

---

## 151. Enrollment requirement

Una Identity puede autenticarse con password pero tener:

```text
MFA_ENROLLMENT_REQUIRED
```

por `IdentitySecurityState`.

---

## 152. MFA enrollment bootstrapping

No se deberá crear una sesión de privilegio normal antes de completar enrollment cuando policy lo prohíba.

---

## 153. Enrollment-only transaction

Puede permitir acceso únicamente a:

```text
factor enrollment flow
logout
recovery
```

sin usar Authorization normal.

---

## 154. Factor removal

Operación sensible.

---

## 155. Removal requirements

Por ejemplo:

```text
fresh AAL2
another remaining factor
recovery verification
```

---

## 156. Last factor removal

Si MFA es obligatorio:

```text
cannot remove last compliant factor
```

sin pasar a recovery/admin flow.

---

## 157. Factor reset

Más sensible que simple removal.

Puede requerir:

```text
SecurityVersion++
revoke trusted devices
revoke sessions
recovery state
```

---

## 158. Compromised factor

Un factor individual puede marcarse:

```text
COMPROMISED
```

---

## 159. Factor compromise impact

Puede invalidar:

```text
sessions derived from factor
trusted-device credentials issued from it
persistent assurance
```

según provenance.

---

## 160. FactorVersion

Puede existir una versión específica del conjunto MFA.

---

## 161. MfaVersion

Ejemplo:

```text
MfaVersion
```

para invalidar sessions/remembered devices cuando cambie configuración MFA.

---

## 162. Simplicidad

V1 puede utilizar:

```text
SecurityVersion
```

para cambios críticos y añadir `FactorVersion` si realmente mejora invalidación selectiva.

---

## 163. Fresh Authentication

Freshness debe tratarse como dimensión independiente.

---

## 164. FreshAuthenticationRequirement

Ejemplo:

```text
primary authentication <= 10 minutes
```

---

## 165. FactorFreshnessRequirement

Puede exigir:

```text
passkey verified within 5 minutes
```

---

## 166. Freshness timestamps

Context puede conservar:

```text
authenticatedAt
primaryVerifiedAt
factorVerifiedAt[type]
lastStepUpAt
```

---

## 167. Activity is not freshness

Una sesión usada durante horas no mantiene un factor “fresco”.

---

## 168. Step-Up freshness

Completar step-up actualiza únicamente la evidencia correspondiente.

---

## 169. Session update

Después de step-up:

```text
AuthenticationContext V1
    ↓
Evidence + new factor
    ↓
AuthenticationContext V2
```

---

## 170. Context immutability

Preferir crear nuevo Context/snapshot, no mutar silenciosamente el existente.

---

## 171. Session ID rotation after step-up

Recomendado para cambios importantes de authentication privilege/assurance.

---

## 172. Step-up lifetime

El assurance elevado podrá tener un tiempo más corto que la sesión base.

---

## 173. Example

```text
Session base:
    valid 8 hours

Wire-transfer step-up:
    valid 5 minutes
```

---

## 174. Scoped Step-Up

En sistemas avanzados, un step-up puede vincularse a:

```text
specific operation
transaction
resource
purpose
```

---

## 175. Transaction-bound assurance

Ejemplo:

```text
approve transfer #123
```

No necesariamente debe elevar toda la sesión para cualquier operación durante 5 minutos.

---

## 176. ScopedAuthenticationEvidence

Podrá representar evidencia ligada a:

```text
purpose
resource
transaction
```

---

## 177. Global vs scoped step-up

Policies:

```text
SESSION_WIDE
PURPOSE_BOUND
TRANSACTION_BOUND
```

---

## 178. High-security default

Para operaciones financieras extremadamente sensibles puede preferirse:

```text
TRANSACTION_BOUND
```

---

## 179. Challenge replay

Challenges exitosos deben quedar consumidos.

---

## 180. Challenge ID entropy

Debe ser impredecible.

---

## 181. Challenge client payload

No deberá contener secrets server-side.

---

## 182. Challenge origin binding

WebAuthn y otros protocolos deben validar origin/RP context.

---

## 183. Challenge session binding

Puede vincularse a:

```text
AuthenticationSessionId/public lineage
```

sin usar el raw session secret como payload.

---

## 184. Challenge transaction hijacking

Un challenge para Alice no deberá poder completarse en una transaction de Bob.

---

## 185. Multi-tab/browser concurrency

MFA flow debe tolerar:

```text
multiple tabs
parallel challenges
expired previous challenge
```

sin confusión.

---

## 186. Challenge supersession

Policy puede determinar que nuevo challenge:

```text
supersedes previous
```

---

## 187. OTP resend

Al reenviar email/SMS OTP:

```text
old code invalidated
```

será una estrategia recomendada.

---

## 188. ChallengeAttemptTracker

Podrá limitar:

```text
verification attempts
resends
challenge creations
```

---

## 189. Brute force protection

Especialmente importante para:

```text
6-digit OTP
recovery codes
PIN-like factors
```

---

## 190. Attempt scope

Puede incluir:

```text
transaction
identity
factor credential
IP/network
device
```

---

## 191. Lockout caution

No permitir que un atacante bloquee permanentemente MFA de otra persona con pocos intentos.

---

## 192. Progressive restrictions

Preferir:

```text
throttling
challenge regeneration
temporary cooldown
risk escalation
```

---

## 193. TOTP code cardinality

No almacenar cada código usado indefinidamente.

Mantener state suficiente para replay prevention.

---

## 194. Recovery Factors

Deberán separarse conceptualmente de normal factors.

---

## 195. FactorPurpose

Podrá ser:

```text
PRIMARY
SECONDARY
RECOVERY
DEVICE_TRUST
INFRASTRUCTURE
```

---

## 196. Recovery factor cannot automatically satisfy normal MFA policy

A menos que policy lo permita explícitamente.

---

## 197. Recovery authentication result

Podrá producir:

```text
RECOVERY_AUTHENTICATED
```

y exigir acciones posteriores.

---

## 198. Recovery code + password

Puede recuperar cuenta, pero no necesariamente producir el mismo assurance que Passkey + hardware key.

---

## 199. Factor reset after recovery

Recomendado:

```text
re-enroll MFA
revoke trusted devices
increment SecurityVersion
```

---

## 200. Backup factor

Diferente de Recovery factor si sigue siendo un factor normal.

Ejemplo:

```text
second registered security key
```

es backup pero sigue siendo factor fuerte.

---

## 201. Factor policy terminology

Distinguir:

```text
primary factor
secondary factor
alternate factor
backup factor
recovery factor
```

---

## 202. Multi-factor combination

El system deberá soportar:

```text
ALL_REQUIRED
AT_LEAST_N
DISTINCT_CATEGORIES
PROPERTY_REQUIREMENTS
CUSTOM_POLICY
```

---

## 203. Example distinct categories

```text
password + TOTP
```

satisface:

```text
knowledge + possession
```

---

## 204. Example same category

```text
password + PIN
```

puede no satisfacer `DISTINCT_CATEGORIES`.

---

## 205. Example property requirement

```text
password + SMS OTP
```

puede satisfacer multi-factor, pero no:

```text
phishing_resistant
```

---

## 206. Passkey-only high assurance

Dependiendo de authenticator properties/policy, una single authentication ceremony podría alcanzar properties fuertes sin usar dos prompts separados.

VoltStack no deberá confundir:

```text
number of prompts
```

con:

```text
assurance quality
```

---

## 207. MFA terminology caution

UI puede decir `Two-factor authentication`, pero domain model debe permanecer más general.

---

## 208. Policy engine

`FactorRequirementPolicy` podrá ser configurable.

---

## 209. Example policy DSL conceptual

```php
Mfa::require(
    Assurance::atLeast('AAL2')
        ->and(FactorProperty::phishingResistant())
        ->freshWithin('5 minutes')
);
```

---

## 210. Configuration style

También podrá declararse:

```php
'authentication' => [
    'requirements' => [
        'admin' => [
            'minimum_assurance' => 'aal2',
            'freshness' => '15 minutes',
        ],
    ],
];
```

---

## 211. Factor policy by route metadata

Routing system podrá adjuntar:

```text
auth.assurance = aal2
auth.fresh = 5m
auth.factor = phishing_resistant
```

---

## 212. Middleware integration

Un middleware podrá evaluar requisitos y devolver:

```text
continue
step_up_required
unauthenticated
```

---

## 213. Middleware must not verify factors itself

Delegará al Authentication subsystem.

---

## 214. Challenge response protocol

SPA/API podrá recibir:

```json
{
  "authentication": {
    "status": "challenge_required",
    "transaction": "txn_public_...",
    "requirements": [
      "phishing_resistant"
    ],
    "available_methods": [
      "passkey",
      "security_key"
    ]
  }
}
```

---

## 215. Safe challenge data

No incluir:

```text
TOTP secret
recovery code values
private credential metadata
```

---

## 216. Browser form flow

Puede redirigir a:

```text
/auth/challenge
```

manteniendo la misma `AuthenticationTransaction`.

---

## 217. Challenge EntryPoint

Podrá existir:

```text
AuthenticationChallengeEntryPoint
```

similar a otros entry points del Auth system.

---

## 218. Resume destination

Después de completar challenge, podrá regresar a la operación original si es seguro.

---

## 219. Open redirect protection

El intended destination deberá ser validado/internalizado.

---

## 220. Transaction continuation token

El frontend puede manejar una referencia opaca:

```text
txn_public_x
```

no el state interno completo.

---

## 221. Transaction hijack protection

Referencia deberá ser:

```text
unguessable
purpose-bound
expiring
```

---

## 222. Transaction ownership

Una base authenticated session puede quedar vinculada a la transaction.

---

## 223. Session loss during challenge

Policy deberá decidir:

```text
restart authentication
```

en lugar de completar step-up sobre una base session inexistente.

---

## 224. Base Context Version

La MFA transaction podrá guardar:

```text
base security version
base session reference
base authentication lineage
```

---

## 225. Commit-time revalidation

Antes de completar step-up:

```text
Identity still eligible?
Session still valid?
SecurityVersion unchanged?
Requirements still current?
```

---

## 226. TOCTOU protection

Evita:

```text
challenge started
    ↓
account disabled
    ↓
challenge completes
    ↓
context upgraded incorrectly
```

---

## 227. Requirements may change mid-flow

Ejemplo:

```text
tenant changes policy to require passkey
```

durante transaction.

La commit phase deberá reevaluar effective requirements.

---

## 228. If requirement became stronger

Puede continuar challenge adicional.

---

## 229. If identity became ineligible

Flow termina.

---

## 230. Factor policy compilation

Configuraciones estáticas podrán compilarse por:

```text
firewall
route metadata
tenant profile
identity type
```

para reducir hot-path overhead.

---

## 231. Dynamic factors

Risk puede añadir requirements en runtime.

---

## 232. Factor resolver performance

No cargar todos los registros de factores si solo se necesita saber:

```text
has phishing-resistant factor?
```

Podrán existir capability indexes.

---

## 233. FactorCredentialSummary

Una carga ligera puede contener:

```text
type
status
properties
credential public id
```

sin secret material.

---

## 234. Lazy credential material loading

TOTP secret solo se carga cuando TOTP realmente se selecciona.

---

## 235. Passkey public keys

Pueden cargarse cuando el WebAuthn flow lo requiera.

---

## 236. Secret cache

No cachear TOTP shared secrets globalmente sin necesidad.

---

## 237. FactorRepository cache

Metadata puede cachearse con invalidation por factor lifecycle.

---

## 238. Factor changes and sessions

Agregar/remover factores puede afectar contexts existentes.

---

## 239. Enrollment change policy

Ejemplo:

```text
adding extra factor
    no invalidation

removing last strong factor
    SecurityVersion++
```

---

## 240. Factor compromise policy

```text
mark factor compromised
    ↓
revoke derived trusted devices
    ↓
invalidate relevant contexts
```

---

## 241. Session provenance requirement

Para invalidación selectiva, Session snapshot debe conocer factors relevantes.

---

## 242. Simple V1

Puede invalidar todas las sessions ante cambios MFA críticos.

---

## 243. Trusted Device lifecycle

Estados:

```text
ACTIVE
EXPIRED
REVOKED
COMPROMISED
ROTATED
```

---

## 244. Trusted device token storage

Similar a persistent credential:

```text
selector + high-entropy secret
server-side digest
```

si se implementa como bearer cookie.

---

## 245. Device token rotation

Recomendado.

---

## 246. Trusted device cookie

Debe utilizar:

```text
HttpOnly
Secure
appropriate SameSite
```

---

## 247. Distinct cookie

No reutilizar:

```text
Session ID
Remember-Me token
```

como trusted-device credential.

---

## 248. Independent revocation

Debe ser posible:

```text
forget MFA trust for device
```

sin necesariamente cerrar sesión base.

---

## 249. Risk invalidation

Cambio fuerte de device/network puede hacer que trusted-device evidence deje de satisfacer policy.

---

## 250. Factor secret encryption

Para factores cuyos secrets deben recuperarse:

```text
TOTP
certain push credentials
```

usar:

```text
FactorSecretProtector
```

---

## 251. FactorSecretProtector

Contrato conceptual:

```php
interface FactorSecretProtectorInterface
{
    public function protect(
        SensitiveFactorSecret $secret
    ): ProtectedFactorSecret;

    public function reveal(
        ProtectedFactorSecret $secret
    ): SensitiveFactorSecret;
}
```

---

## 252. Key rotation

Protected factor secrets deberán soportar rotation de encryption keys cuando sea posible.

---

## 253. Secret exposure

La ventana de plaintext TOTP secret debe ser mínima.

---

## 254. Factor repository breach

Encrypted secrets ofrecen defensa adicional, pero encryption key separation es esencial.

---

## 255. WebAuthn advantage

Passkeys/security keys almacenan:

```text
public key
```

server-side, eliminando shared secret.

---

## 256. Enrollment provisioning URI

TOTP provisioning URI contiene secret.

Debe tratarse como sensible.

---

## 257. QR code

No almacenar QR en logs/caches públicos.

---

## 258. TOTP enrollment page

Debe usar no-store/cache controls cuando corresponda.

---

## 259. Recovery codes UI

También debe impedir caching/logging accidental.

---

## 260. Notification events

Cambios MFA podrán generar notificaciones:

```text
new factor enrolled
factor removed
recovery code used
trusted device added
factor reset
```

---

## 261. Notification system boundary

Auth emite eventos.

Messaging/notification subsystem decide entrega.

---

## 262. Audit events

Importantes:

```text
FactorEnrollmentStarted
FactorEnrolled
FactorRemoved
FactorDisabled
FactorCompromised
FactorVerificationSucceeded
FactorVerificationFailed
StepUpRequired
StepUpCompleted
RecoveryFactorUsed
TrustedDeviceIssued
TrustedDeviceRevoked
MfaReset
```

---

## 263. High-volume failures

Pueden ir a security telemetry/metrics en lugar de audit persistente uno por uno.

---

## 264. Observability spans

```text
auth.factor.resolve
auth.factor.challenge
auth.factor.verify
auth.assurance.calculate
auth.stepup.plan
auth.stepup.complete
auth.factor.enroll
auth.factor.remove
```

---

## 265. Metrics

```text
auth_factor_verification_total
auth_factor_failure_total
auth_mfa_challenge_total
auth_mfa_success_total
auth_stepup_required_total
auth_stepup_success_total
auth_factor_enrollment_total
auth_recovery_factor_used_total
```

---

## 266. Safe labels

```text
factor_type
factor_category
result
firewall
challenge_type
```

No:

```text
identity id
phone
email
credential id
```

como labels de alta cardinalidad.

---

## 267. MFA abandonment

Métrica útil:

```text
challenge_started
vs
challenge_completed
```

para UX y security analysis.

---

## 268. Factor failure reason

Internamente:

```text
INVALID
EXPIRED
REPLAYED
THROTTLED
UNAVAILABLE
```

---

## 269. External response

No revelar detalles excesivos.

---

## 270. Factor availability disclosure

Después de primary authentication puede ser seguro mostrar opciones.

Antes, revelar:

```text
Alice has TOTP and Passkey
```

puede facilitar enumeration.

---

## 271. Pre-auth MFA disclosure

Debe evitarse por defecto.

---

## 272. Login enumeration resistance

No responder antes de demostrar primary credential:

```text
Enter code sent to +52...
```

para una cuenta cuyo password ni siquiera fue verificado, salvo passwordless flows diseñados así.

---

## 273. Passwordless authentication

Factor system también debe soportar flows donde:

```text
Passkey
```

sea el único/primary evidence.

---

## 274. MFA is not password-dependent

Este será un principio clave.

---

## 275. Step-up from token auth

API token Context puede requerir:

```text
additional proof
```

en algunos interactive delegated scenarios.

Pero machine APIs normalmente usarán otras políticas.

---

## 276. Service factors

Service Authentication puede combinar:

```text
mTLS
service token
workload identity
```

como multi-proof Authentication.

---

## 277. Human MFA vs machine multi-proof

Pueden compartir Evidence composition sin usar la misma UX/challenge layer.

---

## 278. Factor purpose profiles

```text
HUMAN_INTERACTIVE
MACHINE
RECOVERY
DEVICE_TRUST
```

---

## 279. Assurance calculator extensibility

Plugins podrán añadir nuevas properties/factor mappings.

---

## 280. Custom Factor

Registro conceptual:

```php
Auth::factor(
    'hardware_badge',
    HardwareBadgeFactorProvider::class
);
```

---

## 281. Custom factor requirements

Debe declarar:

```text
category
strength properties
credential type
verification handler
enrollment support
challenge support
```

---

## 282. Plugins cannot claim arbitrary assurance unchecked

Un plugin no debería decir simplemente:

```text
AAL3
```

sin pasar por mapping/policy del framework/application.

---

## 283. Assurance Trust Policy

Puede limitar qué properties de custom/federated factors son aceptadas.

---

## 284. Factor Policy Security Floor

Framework podrá impedir que un tenant convierta:

```text
email OTP
```

en equivalente a:

```text
phishing-resistant hardware factor
```

por simple configuración.

---

## 285. Tenant customization

Sí podrá:

```text
require TOTP
disable SMS
require MFA for admins
disable trusted devices
```

---

## 286. More restrictive wins

En composición:

```text
framework
application
firewall
tenant
route
risk
```

las restricciones críticas tenderán a combinarse monotónicamente.

---

## 287. Example effective requirement

```text
Framework:
    minimum AAL1

Admin Firewall:
    AAL2

Tenant:
    phishing-resistant

Route:
    fresh 5m

Risk:
    require device-bound

Effective:
    AAL2
    phishing-resistant
    fresh <= 5m
    device-bound
```

---

## 288. Requirement conflicts

Si no existe factor capaz de satisfacer:

```text
UNSATISFIABLE_AUTHENTICATION_REQUIREMENT
```

---

## 289. No silent weakening

Nunca:

```text
require passkey
user has no passkey
    ↓
accept SMS instead
```

sin policy explícita.

---

## 290. Unsatisfiable requirement handling

Puede conducir a:

```text
enrollment required
admin recovery
access denied
```

según contexto.

---

## 291. Enrollment during step-up

En algunos casos se puede permitir:

```text
no compliant factor
    ↓
fresh base auth
    ↓
enroll new factor
    ↓
verify
    ↓
continue
```

pero deberá ser flow explícito.

---

## 292. Circular security problem

No permitir enrollment de un factor fuerte usando únicamente una evidencia que la propia policy considera insuficiente para cambios de seguridad.

---

## 293. Factor enrollment assurance policy

Ejemplo:

```text
to enroll new passkey:
    fresh password + existing factor
```

si ya hay MFA.

---

## 294. First factor bootstrap

Si es la primera configuración MFA:

```text
fresh primary auth
```

puede ser suficiente.

---

## 295. Recovery when all factors lost

Deberá delegarse al Account Recovery System, no crear bypass escondido dentro de FactorOrchestrator.

---

## 296. MFA reset admin operation

No debe simplemente:

```text
delete all factors
```

sin:

```text
audit
security version invalidation
session handling
re-enrollment policy
```

---

## 297. Factor migration

TOTP provider migration, WebAuthn metadata changes, etc., deberán respetar credential lifecycle.

---

## 298. Factor metadata versioning

Podrá existir schema version.

---

## 299. Challenge serializer

Client-visible challenge payload deberá ser versionable.

---

## 300. Persistent runtime safety

Todos los orchestrators/verifiers compartidos deberán ser stateless.

---

## 301. Prohibido

```php
final class FactorOrchestrator
{
    private ?Identity $currentIdentity;
    private array $completedFactors;
}
```

si vive como singleton.

---

## 302. Operation scope

Debe contener:

```text
transaction
current challenges
factor attempts
partial evidence
requirements
```

---

## 303. FrankenPHP cleanup

Al finalizar request:

```text
FactorChallengeContext cleared
StepUpContext cleared
AuthenticationContext cleared
```

---

## 304. Fiber isolation

Dos challenges concurrentes no deben compartir:

```text
Identity
TOTP attempt
WebAuthn challenge
```

---

## 305. Distributed challenges

En múltiples servidores:

```text
Challenge Repository
AuthenticationTransaction Repository
```

deben ser compartidos o replicados apropiadamente.

---

## 306. Sticky sessions not required

Si el transaction store es compartido.

---

## 307. Challenge Repository

Puede usar:

```text
Redis
database
distributed KV
```

---

## 308. Ephemeral data

Redis TTL es especialmente apropiado para challenges.

---

## 309. Durable factor credentials

TOTP/passkey/recovery records requieren store durable.

---

## 310. Challenge store outage

No se podrá verificar challenge de forma confiable.

Resultado:

```text
ERROR
```

y fail closed.

---

## 311. Factor provider outage

Ejemplo push provider no disponible.

Resultado puede ser:

```text
UNAVAILABLE
```

---

## 312. Alternate factor

El usuario podrá seleccionar otro factor permitido.

---

## 313. No automatic downgrade

Si una policy exige phishing resistance y passkey provider falla:

```text
do not silently downgrade to SMS
```

---

## 314. Availability policy

Aplicación puede declarar factores alternativos equivalentes si realmente satisfacen requirements.

---

## 315. FactorVerifier timeout

External providers deberán tener:

```text
timeout
circuit breaker
bounded retry
```

---

## 316. Push retry

Debe ser idempotente/challenge-aware.

---

## 317. TOTP verification local

No requiere remote calls.

---

## 318. Security invariant — Factor

### AUTH-MFA-01

A factor is not equivalent to a credential.

#### AUTH-MFA-02

Only verified factor evidence contributes to assurance.

#### AUTH-MFA-03

Multiple credentials do not automatically mean independent factors.

#### AUTH-MFA-04

Factor strength and independence are policy-derived.

#### AUTH-MFA-05

Factor evidence is identity-bound.

#### AUTH-MFA-06

Factor evidence has explicit verification time.

#### AUTH-MFA-07

Recovery factors are distinguishable from normal factors.

#### AUTH-MFA-08

Factor lifecycle changes are auditable.

---

## 319. Security invariant — Assurance

### AUTH-ASSURANCE-01

Assurance is derived, never client-provided.

#### AUTH-ASSURANCE-02

Assurance is not a boolean MFA flag.

#### AUTH-ASSURANCE-03

Assurance properties may decay through freshness.

#### AUTH-ASSURANCE-04

A strong historical authentication does not imply indefinitely fresh assurance.

#### AUTH-ASSURANCE-05

Federated assurance requires trusted mapping.

#### AUTH-ASSURANCE-06

Evidence cannot be double-counted.

#### AUTH-ASSURANCE-07

Risk may raise requirements but cannot silently weaken hard requirements.

---

## 320. Security invariant — Step-Up

### AUTH-STEPUP-01

Step-up extends an existing authenticated context.

#### AUTH-STEPUP-02

Step-up transaction is bound to the base Identity/context.

#### AUTH-STEPUP-03

The base Authentication state is revalidated before completion.

#### AUTH-STEPUP-04

Identity ineligibility cancels step-up.

#### AUTH-STEPUP-05

Step-up evidence may be session-wide or purpose-bound.

#### AUTH-STEPUP-06

Step-up does not automatically create unrestricted long-lived trust.

#### AUTH-STEPUP-07

Session rotation may accompany meaningful assurance elevation.

---

## 321. Security invariant — Challenge

### AUTH-CHALLENGE-01

Challenges are unpredictable and short-lived.

#### AUTH-CHALLENGE-02

Challenges are purpose-bound.

#### AUTH-CHALLENGE-03

Challenges are transaction-bound.

#### AUTH-CHALLENGE-04

Consumed challenges cannot be replayed.

#### AUTH-CHALLENGE-05

Challenge state is not trusted from client input.

#### AUTH-CHALLENGE-06

Challenge attempt counts are bounded.

#### AUTH-CHALLENGE-07

Resend/regeneration semantics are explicit.

---

## 322. Security invariant — TOTP

### AUTH-TOTP-01

TOTP shared secrets are protected at rest.

#### AUTH-TOTP-02

TOTP codes are never persisted in plaintext unnecessarily.

#### AUTH-TOTP-03

Clock windows are explicit and bounded.

#### AUTH-TOTP-04

Replay within accepted time steps is prevented where feasible.

#### AUTH-TOTP-05

Pending enrollment does not authenticate.

#### AUTH-TOTP-06

TOTP secrets never enter logs/events/metrics.

---

## 323. Security invariant — Recovery

### AUTH-RECOVERY-FACTOR-01

Recovery codes are one-time.

#### AUTH-RECOVERY-FACTOR-02

Recovery factors do not automatically equal normal MFA assurance.

#### AUTH-RECOVERY-FACTOR-03

Recovery use can trigger factor re-enrollment and session invalidation.

#### AUTH-RECOVERY-FACTOR-04

Recovery secrets are never recoverable in plaintext after creation.

---

## 324. Security invariant — Runtime

### AUTH-MFA-RT-01

Factor challenge state is operation/transaction scoped.

#### AUTH-MFA-RT-02

Shared factor services are stateless.

#### AUTH-MFA-RT-03

No current factor state survives between FrankenPHP requests.

#### AUTH-MFA-RT-04

Concurrent fibers have isolated challenge/evidence state.

#### AUTH-MFA-RT-05

Distributed flows use a shared transaction/challenge store when required.

---

## 325. Anti-pattern — MFA boolean

Evitar:

```php
$user->mfa_enabled = true;
```

como modelo completo.

Puede existir como summary cache, pero no como arquitectura central.

---

## 326. Anti-pattern — Two OTPs = two factors

No necesariamente.

---

## 327. Anti-pattern — SMS equals passkey

No deberá tener mismas properties por defecto.

---

## 328. Anti-pattern — recovery code = normal second factor

Debe tener semántica propia.

---

## 329. Anti-pattern — trusted device bypass

No saltarse requirements fuertes sin evaluar su evidence.

---

## 330. Anti-pattern — TOTP secret plaintext

Nunca guardar sin protección adecuada.

---

## 331. Anti-pattern — challenge in process memory only

En distributed deployments puede perderse/cambiar de worker.

---

## 332. Anti-pattern — auto fallback after invalid factor

Un fallo no debe convertirse automáticamente en otra prueba más débil.

---

## 333. Anti-pattern — factor enrollment from stale weak session

Cambios de seguridad deben exigir freshness apropiada.

---

## 334. Anti-pattern — factor claims assurance directly

Un plugin no decide unilateralmente el assurance final.

---

## 335. Anti-pattern — double count federation

No contar assertion y sus factores internos como evidencias independientes duplicadas.

---

## 336. Anti-pattern — activity equals fresh auth

No.

---

## 337. Anti-pattern — account disable during challenge ignored

Commit-time revalidation obligatoria.

---

## 338. Anti-pattern — MFA as Authorization

MFA no decide si el usuario tiene permiso; solo si el Context satisface nivel de autenticación requerido.

---

## 339. Componentes principales

```text
AuthenticationAssurance
AssuranceLevel
AssurancePropertySet
AuthenticationAssuranceCalculator

AuthenticationRequirement
AuthenticationRequirementSet
AuthenticationRequirementResolver

FactorType
FactorCategory
FactorStrength
FactorDescriptor
FactorRegistry

VerifiedFactor
VerifiedFactorSet
FactorVerifier
FactorVerificationResult

FactorOrchestrator
FactorSelectionPolicy
FactorAvailabilityResolver
```

---

## 340. Componentes Step-Up

```text
StepUpPlanner
StepUpPlan
StepUpTransaction
StepUpResult
AuthenticationChallenge
AuthenticationChallengeId
ChallengeRepository
ChallengeAttemptTracker
```

---

## 341. Componentes factor lifecycle

```text
FactorCredential
FactorCredentialRepository
FactorCredentialStatus
FactorEnrollmentManager
FactorRemovalManager
FactorCompromiseManager
FactorSecretProtector
```

---

## 342. Built-in factor modules

```text
PasswordFactorAdapter
TotpFactor
HotpFactor
EmailOtpFactor
SmsOtpFactor
PasskeyFactorAdapter
RecoveryCodeFactor
TrustedDeviceFactor
FederatedFactorAdapter
```

---

## 343. Namespace sugerido

```text
VoltStack\Quantum\Auth\Factor
VoltStack\Quantum\Auth\Factor\Contracts
VoltStack\Quantum\Auth\Factor\Evidence
VoltStack\Quantum\Auth\Factor\Registry
VoltStack\Quantum\Auth\Factor\Verification
VoltStack\Quantum\Auth\Factor\Enrollment
VoltStack\Quantum\Auth\Factor\Challenge
VoltStack\Quantum\Auth\Factor\Totp
VoltStack\Quantum\Auth\Factor\Recovery
VoltStack\Quantum\Auth\Factor\TrustedDevice

VoltStack\Quantum\Auth\Assurance
VoltStack\Quantum\Auth\StepUp
```

---

## 344. Estructura sugerida

```text
src/Quantum/Auth/
├── Assurance/
│   ├── AuthenticationAssurance.php
│   ├── AssuranceLevel.php
│   ├── AssurancePropertySet.php
│   ├── AuthenticationAssuranceCalculator.php
│   ├── AuthenticationRequirement.php
│   ├── AuthenticationRequirementSet.php
│   └── AuthenticationRequirementResolver.php
│
├── Factor/
│   ├── Contracts/
│   │   ├── FactorVerifierInterface.php
│   │   ├── FactorCredentialRepositoryInterface.php
│   │   └── FactorSecretProtectorInterface.php
│   │
│   ├── FactorType.php
│   ├── FactorCategory.php
│   ├── FactorStrength.php
│   ├── FactorDescriptor.php
│   ├── FactorRegistry.php
│   ├── FactorOrchestrator.php
│   ├── FactorSelectionPolicy.php
│   │
│   ├── Evidence/
│   │   ├── VerifiedFactor.php
│   │   └── VerifiedFactorSet.php
│   │
│   ├── Verification/
│   │   ├── FactorVerificationRequest.php
│   │   ├── FactorVerificationResult.php
│   │   └── FactorVerificationStatus.php
│   │
│   ├── Enrollment/
│   │   ├── FactorCredential.php
│   │   ├── FactorCredentialStatus.php
│   │   ├── FactorEnrollmentManager.php
│   │   └── FactorRemovalManager.php
│   │
│   ├── Challenge/
│   │   ├── AuthenticationChallenge.php
│   │   ├── AuthenticationChallengeId.php
│   │   ├── ChallengeRepository.php
│   │   └── ChallengeAttemptTracker.php
│   │
│   ├── Totp/
│   │   ├── TotpCredential.php
│   │   ├── TotpVerifier.php
│   │   ├── TotpEnrollmentManager.php
│   │   └── TotpPolicy.php
│   │
│   ├── Recovery/
│   │   ├── RecoveryCodeCredential.php
│   │   ├── RecoveryCodeVerifier.php
│   │   └── RecoveryCodeManager.php
│   │
│   └── TrustedDevice/
│       ├── TrustedDeviceCredential.php
│       ├── TrustedDeviceVerifier.php
│       ├── TrustedDeviceIssuer.php
│       └── TrustedDevicePolicy.php
│
└── StepUp/
    ├── StepUpPlanner.php
    ├── StepUpPlan.php
    ├── StepUpTransaction.php
    ├── StepUpCoordinator.php
    └── StepUpResult.php
```

---

## 345. Configuración conceptual

```php
return [

    'authentication' => [

        'assurance' => [

            'profiles' => [

                'standard' => [
                    'minimum' => 'aal1',
                ],

                'admin' => [
                    'minimum' => 'aal2',
                ],

                'critical' => [
                    'minimum' => 'aal2',
                    'properties' => [
                        'phishing_resistant',
                    ],
                    'fresh_within' => '5 minutes',
                ],

            ],

        ],

        'factors' => [

            'totp' => [
                'enabled' => true,
            ],

            'passkey' => [
                'enabled' => true,
            ],

            'sms' => [
                'enabled' => false,
            ],

            'trusted_device' => [
                'enabled' => true,
                'lifetime' => '30 days',
            ],

        ],

    ],

];
```

Los valores son ilustrativos.

---

## 346. Configuración por Firewall

```php
'firewalls' => [

    'web' => [
        'assurance' => [
            'minimum' => 'aal1',
        ],
    ],

    'admin' => [
        'assurance' => [
            'minimum' => 'aal2',
            'accepted_factors' => [
                'passkey',
                'totp',
            ],
        ],
    ],

];
```

---

## 347. Route requirement conceptual

```php
Route::post('/payments/{payment}/approve', ...)
    ->auth()
    ->assurance('aal2')
    ->freshAuthentication('5 minutes')
    ->requireFactorProperty('phishing_resistant');
```

La API concreta dependerá del Routing metadata system.

---

## 348. Flujo Password + TOTP

```text
POST /login
    ↓
PasswordAuthenticator
    ↓
Password verified
    ↓
Evidence:
    knowledge=password
    ↓
Assurance Calculator
    ↓
AAL1
    ↓
Firewall requires AAL2
    ↓
FactorOrchestrator
    ↓
TOTP available
    ↓
AuthenticationTransaction
    ↓
TOTP challenge
    ↓
TOTP verified
    ↓
Evidence:
    knowledge=password
    possession=totp
    ↓
Assurance = AAL2
    ↓
AuthenticationContext
    ↓
AuthenticationSession
```

---

## 349. Flujo Passkey Primary

```text
Passkey Authentication
    ↓
signature verified
user presence verified
user verification verified
    ↓
VerifiedFactor:
    possession
    phishing_resistant
    user_verified
    ↓
AssuranceCalculator
    ↓
effective assurance
```

Puede no requerir password adicional si policy considera suficiente la evidencia.

---

## 350. Flujo Step-Up

```text
Existing Session
    ↓
AuthenticationContext:
    password
    AAL1
    ↓
Route requires:
    AAL2
    ↓
RequirementEvaluator
    ↓
STEP_UP_REQUIRED
    ↓
Available:
    Passkey
    TOTP
    ↓
User chooses Passkey
    ↓
WebAuthn Challenge
    ↓
Passkey verified
    ↓
Evidence extended
    ↓
Assurance recalculated
    ↓
AAL2
    ↓
Session ID rotated
    ↓
new Context snapshot persisted
    ↓
original operation resumes
```

---

## 351. Flujo fresh authentication

```text
Session:
    AAL2
    last step-up 2 hours ago

Operation:
    AAL2
    fresh <= 5 minutes

        ↓

Assurance sufficient
Freshness insufficient
        ↓
REAUTHENTICATION_REQUIRED
        ↓
Passkey / selected factor
        ↓
fresh evidence
        ↓
operation allowed to continue
```

---

## 352. Flujo risk-triggered MFA

```text
Login succeeds with Password
        ↓
Risk Engine:
    new device
    unusual network
        ↓
Dynamic Requirement:
    possession factor
        ↓
FactorOrchestrator
        ↓
Passkey/TOTP
        ↓
successful factor
        ↓
AuthenticationContext created
```

---

## 353. Flujo TOTP enrollment

```text
Authenticated Identity
    ↓
Fresh Authentication
    ↓
Generate TOTP secret
    ↓
Store Pending Protected Secret
    ↓
Present QR
    ↓
User submits OTP
    ↓
Verify
    ↓
Activate TOTP credential
    ↓
Audit
    ↓
Optionally generate Recovery Codes
```

---

## 354. Flujo Recovery Code

```text
MFA challenge
    ↓
User cannot access normal factor
    ↓
Select Recovery Code
    ↓
RecoveryCodeVerifier
    ↓
atomic consume
    ↓
Recovery Factor Evidence
    ↓
Recovery policy
    ↓
restricted Authentication / account recovery
    ↓
require factor re-enrollment
```

---

## 355. Flujo Trusted Device

```text
Primary + MFA Authentication
        ↓
User selects "Trust this device"
        ↓
Policy allows
        ↓
TrustedDeviceCredential issued
        ↓
later login
        ↓
Password / Remember-Me base evidence
        +
TrustedDeviceCredential
        ↓
Requirement evaluation
        ↓
may satisfy secondary challenge
        ↓
normal AuthenticationContext
```

---

## 356. Flujo factor compromise

```text
Passkey/TOTP marked compromised
        ↓
FactorCompromiseManager
        ↓
factor status = COMPROMISED
        ↓
SecurityVersion / FactorVersion change
        ↓
derived trusted devices revoked
        ↓
affected sessions invalidated
        ↓
future Authentication:
    factor ignored/rejected
```

---

## 357. Flujo admin AAL

```text
Web Context:
    Password
    AAL1

User enters /admin
        ↓
Admin Firewall:
    AAL2 required
        ↓
same Identity recognized
        ↓
STEP_UP_REQUIRED
        ↓
Passkey
        ↓
AAL2 Context for admin realm
```

Esto permite compartir Identity sin compartir ciegamente assurance.

---

## 358. Arquitectura global

```text
                  AUTHENTICATION EVIDENCE
                           │
                           ▼
                 ASSURANCE CALCULATOR
                           │
                           ▼
                AuthenticationAssurance
                           │
                           ▼
               REQUIREMENT RESOLUTION
                           │
          ┌────────────────┴────────────────┐
          │                                 │
          ▼                                 ▼
     SATISFIED                         INSUFFICIENT
          │                                 │
          │                                 ▼
          │                          STEP-UP PLANNER
          │                                 │
          │                                 ▼
          │                        FACTOR ORCHESTRATOR
          │                                 │
          │             ┌───────────────────┼────────────────┐
          │             ▼                   ▼                ▼
          │           TOTP               PASSKEY          RECOVERY
          │             │                   │                │
          │             └───────────────────┼────────────────┘
          │                                 ▼
          │                         VERIFIED FACTOR
          │                                 │
          └─────────────────┬───────────────┘
                            ▼
                     EVIDENCE EXTENSION
                            │
                            ▼
                  ASSURANCE RECALCULATION
                            │
                            ▼
                  AuthenticationContext
```

---

## 359. Decisiones arquitectónicas principales

VoltStack adoptará:

```text
1. MFA is evidence composition, not a special login mode.
2. Factors and credentials are distinct concepts.
3. Assurance is richer than an MFA boolean.
4. Step-up extends an existing authenticated context.
5. Freshness is independent from session activity.
6. Recovery factors have different semantics from normal factors.
7. Passkeys can be primary or step-up factors.
8. Federated MFA requires explicit trust mapping.
9. Trusted devices are credentials/evidence, not hidden MFA bypasses.
10. Factor state and challenge state are request/transaction scoped under persistent runtimes.
```

---

## 360. Comparación conceptual con Laravel y Symfony

Laravel ofrece una experiencia muy productiva alrededor de autenticación y, mediante paquetes como Fortify/Jetstream, patrones de 2FA basados principalmente en TOTP, recovery codes y confirmación de password.

Symfony aporta una separación más formal mediante authenticators, badges, passports, authentication tokens y mecanismos extensibles de security.

VoltStack tomará esas ventajas, pero construirá MFA como una capa transversal:

```text
Authenticator
    produces primary Evidence

Factor System
    extends Evidence

Assurance System
    interprets Evidence

Requirement System
    determines sufficiency

Step-Up System
    obtains missing Evidence
```

Así se evita acoplar MFA a:

```text
User
password
TOTP
```

y se permite combinar:

```text
Password
Passkey
TOTP
Federated MFA
Hardware Key
Trusted Device
Client Certificate
future factors
```

bajo un modelo coherente.

---

## 361. Criterios de aceptación

El subsistema será considerado completo cuando:

1. distinga Factor de Credential;
2. soporte FactorCategory;
3. soporte FactorType;
4. soporte VerifiedFactor;
5. soporte FactorRegistry;
6. soporte FactorVerifier;
7. soporte FactorOrchestrator;
8. soporte AuthenticationAssurance;
9. soporte AssuranceCalculator;
10. soporte Assurance properties;
11. soporte AuthenticationRequirements;
12. soporte Requirement composition;
13. soporte MFA;
14. soporte step-up;
15. soporte fresh authentication;
16. soporte scoped step-up;
17. soporte TOTP;
18. soporte HOTP extensible;
19. soporte Passkey integration;
20. soporte Email OTP;
21. soporte SMS OTP opcional;
22. soporte Recovery Codes;
23. soporte Trusted Devices;
24. soporte Federated MFA mapping;
25. detecte factor independence;
26. evite double counting;
27. soporte enrollment;
28. soporte factor removal;
29. soporte factor compromise;
30. soporte factor reset;
31. proteja TOTP secrets;
32. prevenga OTP replay;
33. soporte challenge lifecycle;
34. soporte challenge expiration;
35. soporte attempt limiting;
36. soporte risk-triggered MFA;
37. soporte tenant policies;
38. soporte Firewall policies;
39. soporte route requirements;
40. soporte commit-time revalidation;
41. prevenga TOCTOU;
42. sea observable;
43. sea auditable;
44. sea seguro con FrankenPHP;
45. sea fiber-safe;
46. mantenga MFA separado de Authorization.

---

## 362. Regla arquitectónica final

VoltStack deberá preservar:

```text
VERIFIED CREDENTIALS
        ↓
VERIFIED FACTORS
        ↓
AUTHENTICATION EVIDENCE
        ↓
ASSURANCE CALCULATION
        ↓
CURRENT ASSURANCE
        ↓
AUTHENTICATION REQUIREMENTS
        ↓
        ├── SATISFIED
        │      ↓
        │  CONTINUE
        │
        └── INSUFFICIENT
               ↓
           STEP-UP PLAN
               ↓
           FACTOR CHALLENGE
               ↓
           VERIFIED FACTOR
               ↓
           EVIDENCE EXTENSION
               ↓
           NEW ASSURANCE
```

La regla central será:

> **VoltStack no preguntará simplemente “¿tiene MFA activado?”, sino “¿qué evidencia autenticada posee este contexto, qué propiedades de assurance demuestra y qué evidencia adicional necesita esta operación?”**

Esta diferencia permitirá construir un sistema capaz de manejar tanto:

```text
Password + TOTP
```

como:

```text
Passkeys
hardware security keys
federated MFA
adaptive MFA
trusted devices
machine multi-proof authentication
transaction-bound step-up
```

sin crear un pipeline independiente para cada combinación.

---

## 363. Próximo documento recomendado

El siguiente documento será:

```text
16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md
```

Este documento deberá profundizar específicamente en la autenticación moderna mediante Passkeys/WebAuthn:

```text
WebAuthn architecture
Passkeys
FIDO2
PublicKeyCredential
Relying Party
RP ID
Origins
registration ceremony
authentication ceremony
challenge generation
challenge persistence
credential creation options
credential request options
credential IDs
public keys
user handles
discoverable credentials
non-discoverable credentials
resident keys
user presence
user verification
authenticator attachment
platform authenticators
cross-platform authenticators
conditional UI
passkey autofill
attestation
attestation policies
authenticator data
clientDataJSON
signature verification
signature counters
backup eligibility
backup state
synced passkeys
credential lifecycle
multiple passkeys
credential naming
credential revocation
credential recovery
step-up integration
MFA integration
tenant/RP isolation
origin validation
replay protection
audit
observability
testing
FrankenPHP safety
```

Con esto VoltStack tendrá un subsistema específico para uno de los mecanismos que deberá convertirse en pieza central de su autenticación moderna y phishing-resistant.
