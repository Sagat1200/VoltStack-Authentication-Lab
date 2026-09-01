# VoltStack Authentication System

## 02 — Authentication Domain Model and Core Concepts

- **Archivo:** `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Modelo de dominio base  
- **Depende de:**  
- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`

---

## 1. Propósito

Este documento define el **modelo de dominio central** del sistema de autenticación de VoltStack.

Su objetivo es establecer con precisión qué significa cada uno de los conceptos fundamentales del sistema, cuáles son sus responsabilidades y cómo se relacionan entre sí.

El sistema utilizará un vocabulario explícito para evitar confusiones entre conceptos cercanos como:

```text
Identity
IdentityClaim
Credential
Factor
Authenticator
Passport
Evidence
Challenge
Transaction
AuthenticationDecision
AuthenticationContext
AuthenticationToken
Session
Assurance
Risk
```

La consistencia semántica será considerada una parte fundamental de la arquitectura.

---

## 2. Principio del modelo de dominio

Authentication se modelará como un proceso de transformación:

```text
Untrusted Authentication Input
        ↓
Authentication Passport
        ↓
Identity Resolution
        ↓
Credential / Factor Verification
        ↓
Verified Authentication Evidence
        ↓
Authentication Policy
        ↓
Authentication Decision
        ↓
Authentication Context
```

Cada objeto representa un estado diferente del proceso.

No deberán intercambiarse responsabilidades entre ellos.

---

## 3. Modelo conceptual global

```text
AuthenticationRequest
        │
        ▼
Authenticator
        │
        ▼
AuthenticationPassport
        │
        ├── IdentityClaim
        ├── Credential
        ├── Factor Evidence
        ├── Requirements
        └── Metadata
        │
        ▼
IdentityProvider
        │
        ▼
Identity
        │
        ▼
CredentialVerifier
        │
        ▼
VerifiedCredential
        │
        ▼
FactorVerifier
        │
        ▼
VerifiedFactor
        │
        ▼
AuthenticationEvidence
        │
        ├── Identity
        ├── Verified Credentials
        ├── Verified Factors
        └── Verification Metadata
        │
        ▼
Risk Evaluation
        │
        ▼
Authentication Policy
        │
        ▼
Assurance Calculation
        │
        ▼
AuthenticationDecision
        │
   ┌────┼───────────┐
   ▼    ▼           ▼
Success Challenge  Reject
   │
   ▼
AuthenticationContext
        │
   ┌────┴──────┐
   ▼           ▼
Session      Token
```

---

## 4. Aggregate boundaries

El dominio deberá evitar la creación de un único objeto `Authentication` que contenga todas las responsabilidades.

Se distinguirán al menos los siguientes grupos conceptuales:

```text
Identity Domain
Credential Domain
Authentication Attempt Domain
Verification Domain
Challenge Domain
Decision Domain
Authenticated State Domain
Persistence Domain
```

---

## 5. Identity

Una `Identity` representa una entidad cuya identidad puede ser establecida por el sistema.

Conceptualmente:

> Identity describe **quién es una entidad**, pero no implica que la entidad esté actualmente autenticada.

Ejemplos:

```text
UserIdentity
AdministratorIdentity
EmployeeIdentity
CustomerIdentity
ServiceIdentity
MachineIdentity
DeviceIdentity
ApiClientIdentity
FederatedIdentity
```

---

## 6. Identity no significa authenticated

La siguiente relación es inválida:

```text
Identity exists
     ↓
Authenticated
```

Una identidad puede existir en el sistema sin que exista una autenticación activa.

La relación correcta será:

```text
Identity
   +
Verified Authentication Evidence
   +
Authentication Decision
        ↓
AuthenticationContext
```

---

## 7. IdentityInterface

Contrato mínimo conceptual:

```php
interface IdentityInterface
{
    public function identifier(): IdentityIdentifier;

    public function type(): IdentityType;
}
```

El contrato base deberá permanecer deliberadamente pequeño.

No deberá exigir:

```text
password
email
username
roles
permissions
tenant
session
```

---

## 8. IdentityIdentifier

`IdentityIdentifier` será un value object que representa el identificador canónico interno de una identidad.

Podrá encapsular:

```text
integer
UUID
ULID
string
external identifier
compound identifier
```

Ejemplo conceptual:

```php
final readonly class IdentityIdentifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

## 9. IdentityType

El tipo de identidad permitirá distinguir dominios distintos.

Ejemplos:

```text
human
administrator
service
machine
device
client
external
temporary
```

No deberá utilizarse automáticamente como mecanismo de Authorization.

Su función es describir el dominio de identidad.

---

## 10. Identity Attributes

Una identidad podrá exponer atributos adicionales mediante contratos específicos.

Ejemplos:

```text
display name
email
phone
external subject
organization
profile metadata
```

Pero estos atributos no formarán necesariamente parte de `IdentityInterface`.

Podrán utilizarse contratos especializados:

```php
interface EmailAwareIdentityInterface
{
    public function email(): ?string;
}
```

Esto evita convertir el contrato central en una interfaz monolítica.

---

## 11. IdentityClaim

Una `IdentityClaim` representa información presentada para localizar una identidad.

Ejemplos:

```text
email = user@example.com
username = francisco
phone = +52...
subject = oidc-subject
client_id = service-01
certificate_subject = ...
```

Una claim no representa todavía una identidad validada.

---

## 12. IdentityClaim vs IdentityIdentifier

Deben diferenciarse estrictamente.

```text
IdentityClaim
    información presentada para encontrar identidad

IdentityIdentifier
    identificador canónico de la identidad resuelta
```

Ejemplo:

```text
IdentityClaim:
    email = john@example.com

IdentityIdentifier:
    user:01JXYZ...
```

---

## 13. IdentityClaim model

Modelo conceptual:

```php
final readonly class IdentityClaim
{
    public function __construct(
        public string $type,
        public string $value,
        public array $attributes = [],
    ) {}
}
```

---

## 14. IdentityProvider

Un `IdentityProvider` resuelve una claim y produce una Identity.

```text
IdentityClaim
      ↓
IdentityProvider
      ↓
Identity
```

Ejemplos:

```text
DatabaseIdentityProvider
OrmIdentityProvider
LdapIdentityProvider
OidcIdentityProvider
RemoteApiIdentityProvider
MemoryIdentityProvider
```

---

## 15. IdentityProvider no autentica

Un provider no deberá decidir si las credenciales son válidas.

Su responsabilidad será:

```text
find
resolve
load
refresh
```

No:

```text
verify password
validate MFA
authorize action
```

---

## 16. Credential

Una `Credential` representa material utilizado para demostrar una identidad.

Ejemplos:

```text
PasswordCredential
ApiKeyCredential
AccessTokenCredential
PasskeyAssertionCredential
ClientSecretCredential
CertificateCredential
SignedAssertionCredential
RecoveryCodeCredential
```

---

## 17. Credential como dato no confiable

Toda Credential proveniente de una request deberá considerarse:

```text
UNVERIFIED
```

hasta ser procesada por un verifier.

Nunca:

```text
Credential object exists
        ↓
credential valid
```

---

## 18. CredentialInterface

Contrato conceptual:

```php
interface CredentialInterface
{
    public function type(): CredentialType;
}
```

Las credenciales concretas podrán ofrecer métodos especializados.

Ejemplo:

```php
final readonly class PasswordCredential
    implements CredentialInterface
{
    public function __construct(
        #[SensitiveParameter]
        public string $password,
    ) {}

    public function type(): CredentialType
    {
        return CredentialType::PASSWORD;
    }
}
```

---

## 19. Sensitive credentials

Credenciales sensibles deberán:

```text
tener vida útil mínima
no persistirse accidentalmente
no serializarse por defecto
no aparecer en logs
no aparecer en exceptions
no incorporarse al AuthenticationContext
```

---

## 20. CredentialType

El sistema utilizará tipos explícitos.

Ejemplos:

```text
PASSWORD
API_KEY
ACCESS_TOKEN
REFRESH_TOKEN
PASSKEY_ASSERTION
CLIENT_SECRET
CERTIFICATE
SIGNED_ASSERTION
RECOVERY_CODE
```

La representación deberá permitir tipos personalizados.

---

## 21. CredentialVerifier

Un `CredentialVerifier` verifica una Credential.

```text
Identity
   +
Credential
   +
Authentication Environment
        ↓
CredentialVerifier
        ↓
CredentialVerificationResult
```

---

## 22. VerifiedCredential

Cuando una credencial es válida, deberá generarse una representación separada:

```text
VerifiedCredential
```

Nunca se deberá mutar la credencial original para marcar:

```text
verified = true
```

---

## 23. VerifiedCredential model

Ejemplo conceptual:

```php
final readonly class VerifiedCredential
{
    public function __construct(
        public CredentialType $type,
        public string $verifier,
        public DateTimeImmutable $verifiedAt,
        public array $attributes = [],
    ) {}
}
```

Este objeto no deberá contener el secreto original.

---

## 24. Authentication Factor

Un `AuthenticationFactor` representa una categoría o evidencia adicional utilizada para elevar la confianza de autenticación.

Ejemplos:

```text
Password
Passkey
TOTP
Hardware Security Key
Recovery Code
Trusted Device
Certificate
External IdP
```

---

## 25. Credential vs Factor

No son conceptos idénticos.

Una Credential representa material concreto de verificación.

Un Factor representa una dimensión de autenticación.

Ejemplo:

```text
PasswordCredential
      ↓ verified as
Knowledge Factor
```

Otro ejemplo:

```text
PasskeyAssertionCredential
      ↓ verified as
Possession / Cryptographic Factor
```

---

## 26. Factor Categories

VoltStack podrá reconocer categorías como:

```text
KNOWLEDGE
POSSESSION
INHERENCE
CRYPTOGRAPHIC
FEDERATED
DEVICE
CONTEXTUAL
RECOVERY
```

La clasificación será extensible.

---

## 27. AuthenticationFactor

Modelo conceptual:

```php
final readonly class AuthenticationFactor
{
    public function __construct(
        public FactorType $type,
        public FactorCategory $category,
        public array $attributes = [],
    ) {}
}
```

---

## 28. VerifiedFactor

Una vez verificado:

```text
AuthenticationFactor
      ↓
FactorVerifier
      ↓
VerifiedFactor
```

Ejemplo:

```php
final readonly class VerifiedFactor
{
    public function __construct(
        public FactorType $type,
        public FactorCategory $category,
        public string $verifier,
        public DateTimeImmutable $verifiedAt,
        public array $attributes = [],
    ) {}
}
```

---

## 29. Authenticator

Un `Authenticator` interpreta un mecanismo concreto de autenticación.

Ejemplos:

```text
PasswordAuthenticator
BearerTokenAuthenticator
ApiKeyAuthenticator
PasskeyAuthenticator
MagicLinkAuthenticator
OidcAuthenticator
ServiceAccountAuthenticator
```

Su principal resultado será:

```text
AuthenticationPassport
```

---

## 30. Authenticator no es CredentialVerifier

Diferencia:

```text
Authenticator
    entiende la entrada y construye el intento

CredentialVerifier
    verifica evidencia concreta
```

Ejemplo:

```text
PasswordAuthenticator
    extrae email + password

PasswordCredentialVerifier
    verifica el password
```

---

## 31. AuthenticationPassport

`AuthenticationPassport` representa un intento estructurado de autenticación.

Debe responder:

```text
qué identidad se intenta resolver
qué credenciales se presentaron
qué método se está usando
qué factores están disponibles
qué metadata acompaña el intento
```

---

## 32. Passport no representa éxito

Invariante:

```text
AuthenticationPassport
        ≠
Authenticated Identity
```

Puede contener:

```text
datos inválidos
identity inexistente
credenciales incorrectas
factores incompletos
```

---

## 33. Passport model

Ejemplo conceptual:

```php
final readonly class AuthenticationPassport
{
    public function __construct(
        public IdentityClaim $identityClaim,
        public AuthenticationMethod $method,
        public array $credentials = [],
        public array $factorInputs = [],
        public array $requirements = [],
        public array $metadata = [],
    ) {}
}
```

---

## 34. Authentication Method

`AuthenticationMethod` identifica el mecanismo principal utilizado.

Ejemplos:

```text
PASSWORD
SESSION
BEARER_TOKEN
API_KEY
PASSKEY
MAGIC_LINK
OIDC
SAML
CLIENT_CERTIFICATE
SERVICE_CREDENTIAL
```

---

## 35. Method vs Factor

Un método describe el flujo principal.

Un factor describe evidencia utilizada dentro del flujo.

Ejemplo:

```text
Method:
    PASSWORD

Factors:
    KNOWLEDGE(password)
    POSSESSION(TOTP)
```

En un login MFA el método principal puede seguir siendo `PASSWORD`, aunque la autenticación termine utilizando múltiples factores.

---

## 36. AuthenticationRequirement

Representa un requisito que deberá cumplirse antes de considerar válida la autenticación.

Ejemplos:

```text
MinimumAssuranceLevel
RequiredFactor
RequiredVerifiedEmail
RequiredTrustedDevice
RequiredTenantBinding
FreshAuthenticationRequired
```

---

## 37. Requirements no son Authorization rules

Ejemplo correcto:

```text
Require AAL2 before completing authentication
```

Ejemplo que pertenece a Authorization:

```text
User may delete invoice
```

---

## 38. Authentication Evidence

`AuthenticationEvidence` representa el resultado estructurado de las verificaciones realizadas.

Será uno de los conceptos centrales.

Conceptualmente:

```text
AuthenticationEvidence
│
├── Resolved Identity
├── Verified Credentials
├── Verified Factors
├── Provider Evidence
├── Authentication Method
├── Verification Time
└── Metadata
```

---

## 39. Evidence vs Passport

```text
Passport
    contiene afirmaciones y material presentado

Evidence
    contiene resultados ya verificados
```

Transformación:

```text
AuthenticationPassport
        ↓
verification
        ↓
AuthenticationEvidence
```

---

## 40. AuthenticationEvidence model

Ejemplo conceptual:

```php
final readonly class AuthenticationEvidence
{
    public function __construct(
        public IdentityInterface $identity,
        public AuthenticationMethod $method,
        public array $verifiedCredentials,
        public array $verifiedFactors,
        public DateTimeImmutable $verifiedAt,
        public array $attributes = [],
    ) {}
}
```

---

## 41. Evidence must be secret-free

El Evidence nunca deberá contener:

```text
raw password
raw OTP
private key
raw recovery code
raw bearer token
client secret
```

Solo resultados derivados y metadata segura.

---

## 42. Authentication Environment

El sistema deberá representar las condiciones externas del intento.

Ejemplo:

```text
AuthenticationEnvironment
│
├── tenant
├── transport
├── client
├── network
├── device
├── route metadata
├── runtime
└── timestamps
```

---

## 43. Environment vs Context

Deben diferenciarse.

```text
AuthenticationEnvironment
    contexto de entrada utilizado durante la autenticación

AuthenticationContext
    resultado autenticado producido al finalizar
```

---

## 44. Authentication Risk

`AuthenticationRisk` representa una evaluación de riesgo asociada al intento.

Podrá contener:

```text
score
level
signals
evaluator metadata
```

---

## 45. AuthenticationRiskLevel

Ejemplo:

```text
LOW
MODERATE
HIGH
CRITICAL
```

Estos nombres podrán configurarse o extenderse.

---

## 46. RiskSignal

Un `RiskSignal` representa una observación individual.

Ejemplos:

```text
NEW_DEVICE
UNKNOWN_IP
IMPOSSIBLE_TRAVEL
FAILED_ATTEMPTS
TOKEN_REUSE
UNUSUAL_CLIENT
HIGH_VALUE_OPERATION
```

No todos los signals implican rechazo automático.

---

## 47. Authentication Assurance

`AuthenticationAssurance` representa el nivel de confianza obtenido.

Podrá adoptar niveles como:

```text
AAL0
AAL1
AAL2
AAL3
```

donde `AAL0` puede representar ausencia de autenticación utilizable.

---

## 48. Assurance no es Role

Assurance responde:

> ¿Qué tan fuerte fue el proceso que estableció esta identidad?

No:

> ¿Qué privilegios tiene?

Por lo tanto:

```text
AAL2
```

no significa:

```text
administrator
```

---

## 49. Assurance Calculation

El assurance podrá derivarse de:

```text
authentication method
credential strength
number of factors
factor categories
hardware backing
provider trust
freshness
risk state
device binding
```

---

## 50. Authentication Policy

Una `AuthenticationPolicy` evalúa si la evidencia y el assurance cumplen los requisitos del flujo.

Ejemplo:

```text
Identity type = administrator
        ↓
require AAL2
```

---

## 51. Authentication Policy Result

Posibles resultados:

```text
SATISFIED
ADDITIONAL_FACTOR_REQUIRED
STEP_UP_REQUIRED
REJECTED
ERROR
```

---

## 52. Authentication Decision

`AuthenticationDecision` representa la decisión final del motor.

Estados centrales:

```text
AUTHENTICATED
CHALLENGE_REQUIRED
STEP_UP_REQUIRED
REJECTED
ERROR
```

---

## 53. AuthenticationDecisionStatus

Conceptualmente:

```php
enum AuthenticationDecisionStatus
{
    case AUTHENTICATED;
    case CHALLENGE_REQUIRED;
    case STEP_UP_REQUIRED;
    case REJECTED;
    case ERROR;
}
```

---

## 54. Decision is not exception

Un challenge necesario es un resultado normal:

```text
CHALLENGE_REQUIRED
```

No debería representarse mediante excepción general.

Del mismo modo:

```text
wrong password
```

puede ser un rechazo esperado y no un error de infraestructura.

---

## 55. Authentication Challenge

Un `AuthenticationChallenge` representa una acción pendiente necesaria para continuar.

Ejemplos:

```text
TotpChallenge
PasskeyChallenge
EmailOtpChallenge
PushChallenge
RecoveryCodeChallenge
ReauthenticationChallenge
```

---

## 56. Challenge model

Ejemplo conceptual:

```php
interface AuthenticationChallengeInterface
{
    public function type(): ChallengeType;

    public function identifier(): ChallengeIdentifier;

    public function expiresAt(): DateTimeImmutable;
}
```

---

## 57. Challenge Identifier

Cada challenge deberá tener un identificador suficientemente impredecible o estar asociado de manera segura a una transaction.

Nunca deberá confiarse únicamente en información proporcionada por el cliente.

---

## 58. Challenge State

Estados posibles:

```text
CREATED
PENDING
COMPLETED
FAILED
EXPIRED
CANCELLED
CONSUMED
```

---

## 59. One-time challenge

Cuando corresponda, un challenge completado deberá pasar a:

```text
CONSUMED
```

para prevenir replay.

Esto aplica especialmente a:

```text
magic links
OTP
passkey challenges
recovery codes
one-time authentication links
```

---

## 60. Authentication Transaction

Una `AuthenticationTransaction` representa el ciclo de vida de un flujo de autenticación que puede atravesar múltiples pasos o requests.

Ejemplo:

```text
login start
    ↓
password
    ↓
MFA challenge
    ↓
TOTP
    ↓
authenticated
```

---

## 61. Transaction vs Session

No son equivalentes.

```text
AuthenticationTransaction
    estado temporal durante el proceso de autenticación

Session
    estado persistente posterior a una autenticación exitosa
```

---

## 62. Transaction model

Ejemplo conceptual:

```php
final class AuthenticationTransaction
{
    public function __construct(
        public readonly AuthenticationTransactionId $id,
        public readonly DateTimeImmutable $createdAt,
        public readonly DateTimeImmutable $expiresAt,
        public AuthenticationTransactionState $state,
    ) {}
}
```

La mutabilidad interna exacta deberá estudiarse cuidadosamente.

---

## 63. Transaction states

Estados posibles:

```text
CREATED
IDENTITY_RESOLVED
PRIMARY_CREDENTIAL_VERIFIED
CHALLENGE_PENDING
FACTOR_VERIFIED
POLICY_PENDING
AUTHENTICATED
REJECTED
EXPIRED
CANCELLED
```

---

## 64. Transaction invariants

Una transaction:

```text
must expire
must belong to an authentication flow
must be replay resistant
must not contain raw secrets
must not outlive its intended purpose
must not become an authenticated session implicitly
```

---

## 65. Authentication Result

`AuthenticationResult` será el resultado de alto nivel utilizado por integración y transporte.

Podrá ser una jerarquía cerrada:

```text
AuthenticatedResult
ChallengeRequiredResult
StepUpRequiredResult
RejectedResult
AuthenticationErrorResult
```

---

## 66. Result vs Decision

La `AuthenticationDecision` pertenece al dominio de decisión.

El `AuthenticationResult` pertenece a la capa de salida del proceso.

Ejemplo:

```text
Decision:
    CHALLENGE_REQUIRED

Result:
    ChallengeRequiredResult(
        transaction,
        challenge,
        metadata
    )
```

---

## 67. AuthenticatedResult

Contendrá como mínimo:

```text
AuthenticationContext
```

y opcionalmente:

```text
session state
issued token metadata
response hints
```

---

## 68. AuthenticationContext

`AuthenticationContext` es la representación canónica de una autenticación activa y válida.

Este objeto deberá responder:

```text
quién fue autenticado
cómo fue autenticado
cuándo
con qué nivel de assurance
bajo qué tenant
desde qué contexto relevante
```

---

## 69. Context model

Ejemplo conceptual:

```php
final readonly class AuthenticationContext
{
    public function __construct(
        public IdentityInterface $identity,
        public AuthenticationMethod $method,
        public AuthenticationAssuranceLevel $assurance,
        public DateTimeImmutable $authenticatedAt,
        public ?TenantIdentifier $tenant = null,
        public ?DeviceIdentifier $device = null,
        public array $attributes = [],
    ) {}
}
```

---

## 70. Context is immutable

El `AuthenticationContext` deberá ser preferentemente inmutable.

Ejemplo:

```text
AAL1 context
     ↓
step-up
     ↓
new AAL2 context
```

No:

```text
same context mutated in-place
```

---

## 71. Context freshness

El contexto deberá poder expresar cuándo ocurrió la autenticación.

Esto permitirá preguntas como:

```text
authentication happened 30 seconds ago
authentication happened 10 hours ago
```

y soportar:

```text
fresh authentication required
```

---

## 72. Authentication Age

Podrá existir un value object:

```text
AuthenticationAge
```

derivado de:

```text
current time - authenticatedAt
```

No deberá persistirse necesariamente como un valor mutable.

---

## 73. Authentication Provenance

El contexto podrá contener provenance:

```text
authenticator
identity provider
external issuer
authentication method
factor chain
```

Ejemplo:

```text
method = OIDC
provider = microsoft-enterprise
issuer = https://...
```

---

## 74. Authentication Chain

En algunos casos una autenticación puede derivar de varias etapas.

Ejemplo:

```text
OIDC
  +
Passkey step-up
```

Podrá existir:

```text
AuthenticationChain
```

que describa las etapas sin almacenar secretos.

---

## 75. AuthenticationToken

Un `AuthenticationToken` representa una forma de transportar, referenciar o persistir un estado de autenticación.

Puede ser:

```text
opaque token
signed token
JWT
reference token
session-backed token
temporary authentication token
service token
```

---

## 76. Token vs Context

Diferencia fundamental:

```text
AuthenticationToken
    material o referencia transportable

AuthenticationContext
    estado lógico autenticado dentro de VoltStack
```

Un token válido puede producir un Context.

El Context no deberá depender de que exista un token.

---

## 77. Raw Token Credential

Cuando llega un token desde el cliente, inicialmente será una:

```text
TokenCredential
```

y no un `AuthenticationToken` confiable.

Flujo:

```text
raw token
    ↓
TokenCredential
    ↓
verification
    ↓
Verified Token Claims
    ↓
AuthenticationContext
```

---

## 78. Token Claims

Los tokens firmados podrán contener claims como:

```text
subject
issuer
audience
issued_at
expires_at
not_before
session reference
tenant
assurance
scope metadata
```

Pero los claims deberán considerarse no confiables hasta verificar:

```text
signature
issuer
audience
expiration
revocation where required
```

---

## 79. Authentication Session

Una `AuthenticationSession` representa autenticación persistente stateful.

Podrá contener referencias a:

```text
identity
authentication method
assurance
creation time
last activity
expiration
device
tenant
security version
```

---

## 80. Session vs generic Session system

El Auth System no deberá reemplazar al sistema general de sesiones.

La relación será:

```text
AuthenticationSession
        ↓ persisted through
Session Infrastructure
```

---

## 81. AuthenticationSessionId

Será un value object independiente.

La sesión de autenticación podrá coincidir con un session ID general o utilizar una referencia especializada según la implementación.

---

## 82. Session state

Posibles estados:

```text
ACTIVE
IDLE
EXPIRED
REVOKED
TERMINATED
COMPROMISED
```

---

## 83. Session expiration

Se distinguirán al menos:

```text
idle timeout
absolute timeout
credential-driven invalidation
security-version invalidation
manual revocation
```

---

## 84. Device

`Device` representa un dispositivo conocido dentro del contexto de Authentication.

No implica confianza automática.

Estados posibles:

```text
UNKNOWN
KNOWN
TRUSTED
REVOKED
COMPROMISED
```

---

## 85. DeviceIdentifier

Deberá ser diferente de fingerprints inseguros generados únicamente desde datos del navegador.

El sistema no deberá asumir que:

```text
User-Agent + IP = secure device identity
```

---

## 86. Trusted Device

La confianza de dispositivo deberá producirse mediante evidencia específica.

Ejemplos:

```text
device-bound token
cryptographic device key
explicit trusted-device enrollment
verified secure credential
```

---

## 87. Tenant

Authentication deberá reconocer un:

```text
TenantIdentifier
```

cuando la autenticación sea tenant-aware.

La identidad podrá ser:

```text
global
tenant-scoped
tenant-bound
```

---

## 88. Identity + Tenant binding

La misma claim:

```text
email = user@example.com
```

puede resolver a identidades distintas bajo diferentes tenants.

Por tanto:

```text
IdentityClaim
   +
TenantContext
      ↓
IdentityProvider
```

podrá ser necesario.

---

## 89. Authentication Principal

Podrá utilizarse el término:

```text
Principal
```

como representación de una identidad autenticada dentro de determinados adapters.

Sin embargo, el modelo central preferirá:

```text
Identity
AuthenticationContext
```

para evitar ambigüedad.

---

## 90. Subject

En protocolos federados el término:

```text
subject
```

representará el identificador otorgado por un issuer.

Ejemplo:

```text
OIDC issuer
      +
subject
      ↓
Federated Identity Mapping
```

No deberá confundirse automáticamente con el ID interno de VoltStack.

---

## 91. Federated Identity

Podrá representar:

```text
issuer
subject
provider
local identity mapping
attributes
```

La autenticación federada deberá terminar igualmente en un:

```text
AuthenticationContext
```

---

## 92. Service Identity

Una `ServiceIdentity` representa una entidad software.

Ejemplo:

```text
billing-service
email-worker
reporting-engine
deployment-agent
```

No deberá utilizar un usuario humano ficticio.

---

## 93. Machine Identity

Podrá representar una máquina, nodo o workload.

Métodos posibles:

```text
mTLS
signed workload identity
service credential
cloud workload token
```

---

## 94. Anonymous Identity

VoltStack deberá decidir cuidadosamente si una request guest tiene:

```text
null AuthenticationContext
```

o una identidad especial `AnonymousIdentity`.

Como regla base recomendada:

```text
unauthenticated request
    → no AuthenticationContext
```

Esto evita confundir anonimato con autenticación.

Podrán existir adapters que representen anonymous principals si un protocolo lo requiere.

---

## 95. Guest

`guest` será principalmente una conveniencia de API.

Ejemplo:

```php
Auth::guest();
```

equivaldrá conceptualmente a:

```text
AuthenticationContext == null
```

---

## 96. Impersonation

Impersonation no deberá modelarse simplemente reemplazando Identity.

Si se soporta, un contexto deberá conservar:

```text
actor identity
effective identity
delegation reason
started at
assurance
audit metadata
```

Aunque su especificación profunda pertenecerá a otros documentos.

---

## 97. Delegated Identity

Para sistemas service-to-service podrán existir contexts delegados.

Ejemplo:

```text
Human User
    ↓
Frontend Service
    ↓
Backend Service
```

El backend podría conocer:

```text
actor = frontend-service
subject = user-123
```

Estos conceptos deberán permanecer explícitos.

---

## 98. Actor vs Subject

En flows delegados:

```text
Actor
    entidad que ejecuta técnicamente la request

Subject
    entidad en cuyo nombre se ejecuta
```

No deberán mezclarse.

---

## 99. Authentication Failure

Un fallo de autenticación deberá representarse mediante una taxonomía precisa.

Categorías:

```text
Credential Failure
Identity Resolution Failure
Policy Rejection
Challenge Failure
Token Failure
Provider Failure
Infrastructure Failure
```

---

## 100. Failure reason exposure

El dominio podrá conocer:

```text
IDENTITY_NOT_FOUND
INVALID_PASSWORD
TOKEN_REVOKED
```

pero la capa externa podrá responder simplemente:

```text
INVALID_CREDENTIALS
```

para prevenir enumeration y filtración de información.

---

## 101. Expected failure vs system error

Diferencia:

```text
wrong password
    expected authentication failure

database unavailable
    authentication infrastructure error
```

Ambos deberán terminar fail-closed, pero sus métricas, logs y handlers serán diferentes.

---

## 102. AuthenticationError

Podrá representar errores operacionales.

Ejemplos:

```text
ProviderUnavailable
KeyResolutionFailed
SessionStoreUnavailable
AuthenticationConfigurationError
```

No deberá producir un Context autenticado.

---

## 103. Value objects centrales

El dominio deberá preferir value objects para conceptos sensibles.

Posibles:

```text
IdentityIdentifier
IdentityClaim
AuthenticationTransactionId
AuthenticationSessionId
AuthenticationChallengeId
AuthenticationMethod
AuthenticationAssuranceLevel
AuthenticationRiskLevel
TenantIdentifier
DeviceIdentifier
CredentialType
FactorType
```

---

## 104. Enums vs extensibilidad

Los enums PHP deberán utilizarse únicamente cuando el conjunto de valores sea realmente cerrado.

Conceptos que deban admitir plugins podrán requerir value objects basados en strings.

Ejemplo:

```text
AuthenticationMethod
```

probablemente deberá ser extensible.

---

## 105. Equality semantics

Los value objects deberán definir igualdad por valor.

Ejemplo:

```text
IdentityIdentifier('abc')
    ==
IdentityIdentifier('abc')
```

sin depender de identidad de objeto PHP.

---

## 106. Domain invariants

El dominio tendrá las siguientes invariantes principales.

### AUTH-DOMAIN-01

Una `IdentityClaim` no es una `Identity`.

#### AUTH-DOMAIN-02

Una `Credential` no es una credencial verificada.

#### AUTH-DOMAIN-03

Un `AuthenticationPassport` no es una autenticación exitosa.

#### AUTH-DOMAIN-04

Solo un `IdentityProvider` o mecanismo equivalente autorizado puede resolver una Identity.

#### AUTH-DOMAIN-05

Solo evidencia verificada puede formar parte de `AuthenticationEvidence`.

#### AUTH-DOMAIN-06

`AuthenticationEvidence` no deberá contener secretos originales.

#### AUTH-DOMAIN-07

Un Challenge pendiente no es un fallo necesariamente.

#### AUTH-DOMAIN-08

Una Transaction no es una Session.

#### AUTH-DOMAIN-09

Solo una decisión `AUTHENTICATED` puede generar un AuthenticationContext normal.

#### AUTH-DOMAIN-10

Un AuthenticationContext no otorga permisos por sí mismo.

#### AUTH-DOMAIN-11

Un Token no es confiable hasta ser validado.

#### AUTH-DOMAIN-12

Un Device conocido no implica Device confiable.

#### AUTH-DOMAIN-13

Una Identity existente no implica Authentication activa.

#### AUTH-DOMAIN-14

El estado guest debe distinguirse explícitamente de authenticated.

---

## 107. State transitions

Modelo simplificado:

```text
NO_ATTEMPT
    │
    ▼
PASSPORT_CREATED
    │
    ▼
IDENTITY_RESOLVED
    │
    ▼
CREDENTIALS_VERIFIED
    │
    ▼
FACTORS_EVALUATED
    │
    ├───────────────┐
    ▼               │
CHALLENGE_PENDING   │
    │               │
    └───────────────┘
    │
    ▼
EVIDENCE_READY
    │
    ▼
POLICY_EVALUATED
    │
    ├──────────────► REJECTED
    │
    ├──────────────► STEP_UP_REQUIRED
    │
    ▼
AUTHENTICATED
    │
    ▼
CONTEXT_CREATED
```

---

## 108. Invalid transitions

Ejemplos que deberán impedirse:

```text
PASSPORT_CREATED
    → AuthenticationContext

Credential received
    → VerifiedCredential

Challenge created
    → Authenticated

Token parsed
    → AuthenticationContext
```

sin ejecutar las verificaciones correspondientes.

---

## 109. Context reconstruction

Cuando existe una sesión válida:

```text
Stored Authentication State
      ↓
integrity / validity checks
      ↓
Identity reload or snapshot validation
      ↓
AuthenticationContext
```

No deberá asumirse que un estado persistido antiguo sigue siendo válido indefinidamente.

---

## 110. Security Version

Una Identity podrá estar asociada a una versión de seguridad.

Ejemplo:

```text
securityVersion = 7
```

Una sesión creada con:

```text
securityVersion = 6
```

podrá ser invalidada.

Casos:

```text
password changed
MFA reset
account recovery
administrator forces logout
credential compromise
```

---

## 111. Credential Version

También podrá existir una versión específica de credenciales.

Esto permitirá distinguir:

```text
identity profile updated
```

de:

```text
authentication-sensitive state changed
```

---

## 112. Authentication Freshness

Podrá modelarse mediante:

```text
authenticatedAt
lastVerifiedAt
lastStepUpAt
```

según necesidades futuras.

Ejemplo:

```text
AAL2 reached yesterday
```

puede no satisfacer una operación que requiera:

```text
fresh AAL2 within 5 minutes
```

---

## 113. Step-Up Context

Un step-up exitoso deberá producir un contexto nuevo que conserve provenance.

Ejemplo:

```text
Context V1
method: password
assurance: AAL1

        ↓ passkey

Context V2
method chain:
    password
    passkey
assurance:
    AAL2
```

---

## 114. Authentication Chain model

Podría modelarse:

```php
final readonly class AuthenticationChain
{
    /** @param AuthenticationStep[] $steps */
    public function __construct(
        public array $steps,
    ) {}
}
```

Cada `AuthenticationStep` podría contener:

```text
method
verified factors
timestamp
provider
assurance contribution
```

---

## 115. Authentication Context Attributes

Los atributos serán metadata extensible.

Ejemplos:

```text
external issuer
session reference
risk result id
device trust
authentication transaction id
```

No deberán utilizarse para almacenar cualquier dato arbitrario sin gobernanza.

---

## 116. Typed attributes

A largo plazo será preferible utilizar atributos tipados o namespaced:

```text
voltstack.auth.device_trust
voltstack.auth.oidc.issuer
application.authentication.source
```

para evitar colisiones.

---

## 117. Core objects mutability policy

Preferencia general:

```text
Value Objects
    immutable

Identity
    implementation-defined but preferably stable

Passport
    immutable

VerifiedCredential
    immutable

VerifiedFactor
    immutable

AuthenticationEvidence
    immutable

AuthenticationDecision
    immutable

AuthenticationContext
    immutable

AuthenticationResult
    immutable
```

`AuthenticationTransaction` puede requerir evolución de estado, pero deberá controlarse mediante métodos específicos o reconstrucción.

---

## 118. Domain services

No toda lógica deberá residir en entities/value objects.

Domain services previstos:

```text
IdentityResolver
CredentialVerificationManager
FactorVerificationManager
AuthenticationEvidenceBuilder
AuthenticationAssuranceCalculator
AuthenticationPolicyEngine
AuthenticationDecisionEngine
AuthenticationContextFactory
```

---

## 119. Identity Resolver

Será responsable de coordinar:

```text
claim normalization
provider selection
identity lookup
tenant-aware resolution
provider errors
```

---

## 120. Evidence Builder

Deberá producir:

```text
AuthenticationEvidence
```

exclusivamente a partir de resultados verificados.

Podrá garantizar:

```text
no duplicate factors
no raw credentials
consistent timestamps
verified identity
```

---

## 121. Assurance Calculator

Será un domain service independiente.

Entrada:

```text
AuthenticationEvidence
AuthenticationEnvironment
Risk Result
```

Salida:

```text
AuthenticationAssurance
```

---

## 122. Decision Engine

Combinará:

```text
AuthenticationEvidence
Assurance
Risk
AuthenticationRequirements
AuthenticationPolicies
```

para producir:

```text
AuthenticationDecision
```

---

## 123. Context Factory

Será la única vía normal para producir un `AuthenticationContext`.

Esto permitirá centralizar:

```text
provenance
timestamps
tenant binding
device binding
security version
assurance
context attributes
```

---

## 124. Domain events

El modelo podrá emitir conceptos de evento como:

```text
AuthenticationAttempted
IdentityResolved
CredentialVerified
FactorVerified
ChallengeCreated
ChallengeCompleted
AuthenticationRejected
AuthenticationSucceeded
AuthenticationContextEstablished
AuthenticationSessionCreated
AuthenticationSessionRevoked
```

---

## 125. Domain events vs audit events

No deberán confundirse.

```text
Domain Event
    comunica algo ocurrido dentro del dominio

Audit Event
    registro de seguridad destinado a trazabilidad
```

Un mismo hecho podrá generar ambos.

---

## 126. Serialization policy

Objetos permitidos para persistencia deberán tener serializers específicos.

Nunca deberá utilizarse como estrategia de seguridad:

```php
serialize($entireAuthenticationObject);
```

especialmente cuando contenga objetos sensibles.

---

## 127. Persistence representations

Podrán existir DTOs específicos:

```text
PersistedAuthenticationSession
PersistedAuthenticationTransaction
PersistedChallenge
PersistedTokenRecord
```

separados del modelo runtime.

---

## 128. Domain object IDs

Entidades temporales relevantes deberán tener IDs propios:

```text
AuthenticationTransactionId
AuthenticationChallengeId
AuthenticationSessionId
TokenId
DeviceId
```

Esto mejora auditoría y evita utilizar secrets como identificadores.

---

## 129. Authentication correlation

Además podrá existir:

```text
AuthenticationCorrelationId
```

para seguir el flujo completo:

```text
request
login
MFA
session creation
audit events
```

sin reutilizar identificadores sensibles.

---

## 130. Authentication attempt

Un `AuthenticationAttempt` podrá utilizarse como concepto observacional que represente una ejecución completa.

No deberá necesariamente convertirse en una entidad persistente del dominio.

Puede incluir:

```text
correlation id
authenticator
started at
completed at
result
```

---

## 131. Secret vs identifier

Debe mantenerse una separación estricta.

Incorrecto:

```text
API key used as database primary identifier
```

Preferible:

```text
API key id
   +
secret
```

El secreto puede rotarse sin cambiar la identidad lógica del credential record.

---

## 132. Credential record

Para credenciales persistidas podrá existir un modelo:

```text
CredentialRecord
│
├── credential id
├── identity id
├── type
├── verifier material
├── created at
├── expires at
├── status
└── metadata
```

Nunca deberá almacenar plaintext cuando exista una representación segura alternativa.

---

## 133. Credential status

Estados posibles:

```text
ACTIVE
EXPIRED
REVOKED
COMPROMISED
PENDING
DISABLED
```

---

## 134. Passkey model

Una passkey persistida conceptualmente podrá contener:

```text
credential id
public key
identity reference
sign counter / authenticator metadata
transports
device metadata
created at
last used at
```

Nunca la private key.

---

## 135. Password credential model

La representación persistida será:

```text
PasswordCredentialRecord
│
├── identity
├── password hash
├── algorithm metadata
├── updated at
└── security version
```

El password plaintext jamás será parte de este record.

---

## 136. Recovery credential

Recovery codes deberán tratarse como credenciales de alta sensibilidad y normalmente de un solo uso.

Estado:

```text
ACTIVE
CONSUMED
REVOKED
```

---

## 137. Authentication capability discovery

Una Identity podrá tener asociados mecanismos disponibles.

Ejemplo:

```text
password
passkey
TOTP
recovery codes
federated login
```

Esto no significa que deban exponerse todos públicamente sin controles.

---

## 138. Registered Factor

Podrá diferenciarse:

```text
RegisteredFactor
```

de:

```text
VerifiedFactor
```

Ejemplo:

```text
User has TOTP configured
```

no significa:

```text
TOTP was verified in this authentication
```

---

## 139. Factor enrollment

El alta de un factor es un proceso separado del uso de ese factor.

```text
Factor Enrollment
      ≠
Factor Verification
```

Su seguridad requerirá documentación específica posterior.

---

## 140. Authentication capability vs authorization ability

Nunca deberán confundirse:

```text
hasPasskey()
```

con:

```text
canDeleteUsers()
```

El primero pertenece a Authentication.

El segundo a Authorization.

---

## 141. Public domain contracts

Los contratos que probablemente formarán parte de la API estable incluyen:

```text
IdentityInterface
IdentityProviderInterface
AuthenticatorInterface
CredentialInterface
CredentialVerifierInterface
AuthenticationContextInterface
AuthenticationChallengeInterface
AuthenticationPolicyInterface
```

Su definición final podrá evolucionar en documentos especializados.

---

## 142. Internal domain types

Tipos que probablemente permanecerán internos:

```text
AuthenticationEvidenceBuilder
CompiledRequirementSet
InternalCredentialVerificationPipeline
AuthenticationStateTransitionGuard
```

Esto evita congelar detalles de implementación prematuramente.

---

## 143. Naming conventions

Se utilizarán términos completos en conceptos de dominio importantes.

Preferible:

```text
AuthenticationContext
AuthenticationTransaction
CredentialVerificationResult
```

Evitar:

```text
AuthCtx
Txn
CredResult
```

en API pública.

---

## 144. Namespace strategy

Posible distribución:

```text
VoltStack\Quantum\Auth\Identity
VoltStack\Quantum\Auth\Credentials
VoltStack\Quantum\Auth\Authenticator
VoltStack\Quantum\Auth\Passport
VoltStack\Quantum\Auth\Evidence
VoltStack\Quantum\Auth\Factors
VoltStack\Quantum\Auth\Challenge
VoltStack\Quantum\Auth\Transaction
VoltStack\Quantum\Auth\Decision
VoltStack\Quantum\Auth\Context
VoltStack\Quantum\Auth\Session
VoltStack\Quantum\Auth\Token
VoltStack\Quantum\Auth\Risk
VoltStack\Quantum\Auth\Assurance
```

---

## 145. Core relationship matrix

| Concepto | Entrada | Produce | ¿Confiable inicialmente? |
| --- | --- | --- | --- |
| IdentityClaim | request/input | búsqueda de Identity | No |
| Identity | provider | sujeto resuelto | Sí como identidad, no como autenticación |
| Credential | request/input | verificación | No |
| VerifiedCredential | verifier | evidencia | Sí |
| AuthenticationFactor | configuración/input | factor verificable | Depende |
| VerifiedFactor | verifier | evidencia | Sí |
| Passport | authenticator | intento estructurado | No |
| AuthenticationEvidence | verification pipeline | evidencia consolidada | Sí |
| Risk | evaluators | señales de riesgo | Sí según evaluator |
| Assurance | calculator | nivel de confianza | Sí |
| Decision | policy engine | resultado | Sí |
| Challenge | decision engine | paso pendiente | Sí |
| Transaction | authentication flow | estado temporal | Sí, con integridad |
| AuthenticationContext | context factory | estado autenticado | Sí |
| AuthenticationToken | issuer / transport | transporte de auth | Solo tras validación |
| AuthenticationSession | session subsystem | persistencia stateful | Sí mientras siga válida |

---

## 146. Core lifecycle example

Ejemplo de password + TOTP:

```text
1. IdentityClaim(email)

2. PasswordCredential(raw secret)

3. PasswordAuthenticator
      ↓
   AuthenticationPassport

4. IdentityProvider
      ↓
   UserIdentity

5. PasswordCredentialVerifier
      ↓
   VerifiedCredential(password)

6. AuthenticationEvidence
      ↓
   current assurance = AAL1

7. Policy
      ↓
   AAL2 required

8. AuthenticationDecision
      ↓
   CHALLENGE_REQUIRED

9. AuthenticationTransaction

10. TotpChallenge

11. TotpCredential

12. TotpVerifier
      ↓
    VerifiedFactor(TOTP)

13. New AuthenticationEvidence

14. AssuranceCalculator
      ↓
    AAL2

15. AuthenticationDecision
      ↓
    AUTHENTICATED

16. AuthenticationContext

17. AuthenticationSession
```

---

## 147. Stateless token lifecycle example

```text
Bearer Header
     ↓
TokenCredential
     ↓
BearerTokenAuthenticator
     ↓
AuthenticationPassport
     ↓
TokenVerifier
     ↓
Verified Token Evidence
     ↓
Identity Resolution
     ↓
AuthenticationEvidence
     ↓
Policy / Assurance
     ↓
AuthenticationContext
```

No es obligatorio crear:

```text
AuthenticationSession
```

---

## 148. Passkey lifecycle example

```text
Passkey assertion
      ↓
PasskeyCredential
      ↓
PasskeyAuthenticator
      ↓
AuthenticationPassport
      ↓
challenge verification
origin / RP verification
signature verification
      ↓
VerifiedCredential
      ↓
Identity
      ↓
AuthenticationEvidence
      ↓
Assurance
      ↓
AuthenticationContext
```

---

## 149. Federated identity lifecycle

```text
OIDC callback
     ↓
Authorization code
     ↓
OIDC Authenticator
     ↓
Authentication Transaction validation
     ↓
Token exchange
     ↓
ID token verification
     ↓
issuer + subject
     ↓
Federated Identity mapping
     ↓
AuthenticationEvidence
     ↓
AuthenticationContext
```

---

## 150. Domain security properties

El modelo deberá garantizar:

```text
Explicit Trust
Explicit Verification
Immutable Results
Minimal Secret Propagation
Replay Resistance
Scope Isolation
Tenant Binding
Traceability
Fail Closed
Extensible Identity Model
```

---

## 151. Anti-patterns prohibidos

### 151.1 User object as authentication state

Evitar:

```php
$currentUser = $user;
```

como única representación de autenticación.

Debe existir contexto.

---

### 151.2 Boolean-only authentication

Evitar limitar el resultado a:

```php
true / false
```

porque existen:

```text
challenge required
step-up required
expired
rejected
infrastructure error
```

---

### 151.3 Credential self-verification

Evitar:

```php
$passwordCredential->verify();
```

si esto acopla secreto, identidad, hasher y storage.

Preferir verifier especializado.

---

### 151.4 Mutable Passport

Evitar convertir progresivamente:

```text
Passport
   verified = true
   identity = ...
   factor = ...
```

hasta que represente múltiples estados semánticos.

---

### 151.5 Token equals identity

Un token no deberá convertirse directamente en Identity sin validación.

---

### 151.6 Session equals user

Una sesión es estado de autenticación persistente, no una entidad User.

---

### 151.7 Role inside Identity core contract

Roles pertenecen al sistema de Authorization.

---

## 152. Aggregate conceptual final

```text
                       ┌─────────────────────┐
                       │ AuthenticationInput │
                       └──────────┬──────────┘
                                  │
                                  ▼
                         IdentityClaim
                                  │
                                  ▼
                           Authenticator
                                  │
                                  ▼
                    AuthenticationPassport
                                  │
                 ┌────────────────┼────────────────┐
                 │                │                │
                 ▼                ▼                ▼
            IdentityClaim     Credentials     Factor Inputs
                 │                │                │
                 ▼                ▼                ▼
          IdentityProvider     Verifiers        Verifiers
                 │                │                │
                 ▼                ▼                ▼
              Identity      VerifiedCredential VerifiedFactor
                 │                │                │
                 └────────────────┼────────────────┘
                                  ▼
                    AuthenticationEvidence
                                  │
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
              Risk             Policy           Assurance
                │                 │                 │
                └─────────────────┼─────────────────┘
                                  ▼
                    AuthenticationDecision
                                  │
               ┌──────────────────┼──────────────────┐
               │                  │                  │
               ▼                  ▼                  ▼
        AUTHENTICATED      CHALLENGE_REQUIRED      REJECTED
               │                  │
               │                  ▼
               │         AuthenticationTransaction
               │                  │
               │                  ▼
               │             Challenge
               │
               ▼
       AuthenticationContext
               │
       ┌───────┴─────────┐
       ▼                 ▼
AuthenticationSession AuthenticationToken
```

---

## 153. Regla conceptual principal

VoltStack deberá conservar permanentemente esta distinción:

```text
Claim
   identifies a candidate

Credential
   presents proof

Verifier
   validates proof

Evidence
   records verified proof

Policy
   determines requirements

Assurance
   measures confidence

Decision
   determines authentication outcome

Context
   represents active authentication

Session / Token
   persist or transport authentication
```

Cada concepto existe por una razón distinta y no deberá absorber las responsabilidades de los demás.

---

## 154. Criterios de aceptación

El modelo de dominio será considerado correcto cuando:

1. pueda representar identidades humanas y no humanas;
2. no dependa obligatoriamente de `User`;
3. distinga claims de identities;
4. distinga credentials de verified credentials;
5. distinga Passport de Evidence;
6. soporte MFA de forma natural;
7. soporte challenges multi-step;
8. soporte transactions;
9. soporte assurance;
10. soporte risk evaluation;
11. soporte sesiones y tokens sin mezclarlos;
12. represente stateful y stateless authentication;
13. pueda modelar Passkeys;
14. pueda modelar identidad federada;
15. pueda representar service identities;
16. pueda soportar multi-tenancy;
17. no almacene secretos en AuthenticationContext;
18. preserve estados no ambiguos;
19. funcione con runtimes persistentes;
20. permanezca independiente de Authorization.

---

## 155. Próximo documento

El siguiente documento será:

```text
03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md
```

Su responsabilidad será especificar en profundidad el ciclo completo de autenticación:

```text
request entry
        ↓
authentication discovery
        ↓
firewall resolution
        ↓
authenticator selection
        ↓
passport creation
        ↓
identity resolution
        ↓
credential verification
        ↓
factor verification
        ↓
risk evaluation
        ↓
policy evaluation
        ↓
challenge / step-up
        ↓
decision
        ↓
context creation
        ↓
session/token establishment
        ↓
request completion
```

incluyendo estados, short-circuiting, errores, hooks, eventos, seguridad y comportamiento bajo runtimes persistentes.
