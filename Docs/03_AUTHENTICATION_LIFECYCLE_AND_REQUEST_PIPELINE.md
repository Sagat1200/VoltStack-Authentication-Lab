# VoltStack Authentication System

## 03 — Authentication Lifecycle and Request Pipeline

- **Archivo:** `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación del ciclo de vida y pipeline  
- **Depende de:**  
- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`

---

## 1. Propósito

Este documento define el **ciclo de vida completo de una autenticación** dentro de VoltStack.

Su objetivo es especificar cómo una entrada potencialmente no confiable atraviesa el sistema hasta producir uno de los siguientes resultados:

```text
AUTHENTICATED
CHALLENGE_REQUIRED
STEP_UP_REQUIRED
REJECTED
UNAUTHENTICATED
ERROR
```

El lifecycle deberá ser consistente en:

```text
HTTP
SPA
API
WebSocket
CLI
Queues
Workers
Internal Services
Machine-to-Machine
```

aunque cada transporte utilice adapters diferentes.

---

## 2. Principio fundamental del lifecycle

Una autenticación nunca deberá saltar directamente desde:

```text
Request
   ↓
User
```

El flujo correcto será:

```text
Input
   ↓
Normalization
   ↓
Authentication Discovery
   ↓
Firewall Resolution
   ↓
Authenticator Resolution
   ↓
Passport Creation
   ↓
Identity Resolution
   ↓
Credential Verification
   ↓
Factor Verification
   ↓
Evidence Assembly
   ↓
Risk Evaluation
   ↓
Authentication Policy
   ↓
Assurance Calculation
   ↓
Decision
   ↓
Authentication Context
   ↓
State Establishment
```

Cada etapa deberá tener responsabilidades claras.

---

## 3. Lifecycle global

```text
┌─────────────────────────────────────┐
│          Runtime Request            │
└──────────────────┬──────────────────┘
                   │
                   ▼
          Transport Adaptation
                   │
                   ▼
        AuthenticationRequest
                   │
                   ▼
        Authentication Discovery
                   │
                   ▼
          Firewall Resolution
                   │
                   ▼
       Authenticator Resolution
                   │
                   ▼
          Passport Creation
                   │
                   ▼
          Passport Processing
                   │
        ┌──────────┼───────────┐
        ▼          ▼           ▼
     Identity   Credentials   Factors
        │          │           │
        └──────────┼───────────┘
                   ▼
        AuthenticationEvidence
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
       Risk      Policy    Assurance
        │          │          │
        └──────────┼──────────┘
                   ▼
       AuthenticationDecision
                   │
        ┌──────────┼─────────────┐
        ▼          ▼             ▼
Authenticated   Challenge      Rejected
        │
        ▼
 AuthenticationContext
        │
   ┌────┴─────┐
   ▼          ▼
Session      Token
```

---

## 4. Fases del pipeline

El pipeline se divide en trece fases principales:

```text
Phase 01 — Runtime Entry
Phase 02 — Transport Normalization
Phase 03 — Existing Authentication Recovery
Phase 04 — Firewall Resolution
Phase 05 — Authenticator Discovery
Phase 06 — Passport Creation
Phase 07 — Identity Resolution
Phase 08 — Credential and Factor Verification
Phase 09 — Evidence Construction
Phase 10 — Risk, Policy and Assurance
Phase 11 — Authentication Decision
Phase 12 — Context Establishment
Phase 13 — Persistence and Completion
```

---

## 5. Phase 01 — Runtime Entry

El lifecycle comienza cuando algún runtime necesita conocer o establecer autenticación.

Ejemplos:

```text
incoming HTTP request
SPA action
API request
WebSocket connection
CLI command
queued job
service request
```

El Authentication System no deberá asumir que toda ejecución proviene de HTTP.

---

## 6. Entradas posibles

La autenticación podrá comenzar por distintas razones.

### 6.1 Protected request

```text
GET /dashboard
```

requiere identidad autenticada.

---

### 6.2 Explicit login

```text
POST /login
```

intenta establecer una nueva autenticación.

---

### 6.3 Lazy identity access

```php
Auth::user();
```

puede disparar la restauración lazy de autenticación existente.

---

### 6.4 API credential

```text
Authorization: Bearer ...
```

puede disparar autenticación stateless.

---

### 6.5 Multi-step continuation

```text
POST /auth/mfa/verify
```

continúa una `AuthenticationTransaction`.

---

## 7. Request lifecycle ownership

Cada ejecución tendrá un scope explícito.

```text
RequestScope
│
├── AuthenticationRequest
├── AuthenticationResult
├── AuthenticationContext
├── request memoization
└── correlation metadata
```

Este estado deberá destruirse al terminar la unidad de trabajo correspondiente.

---

## 8. Persistent runtime rule

Bajo FrankenPHP u otros runtimes persistentes:

```text
Request A
     ↓
request-scoped auth state
     ↓
cleanup
     ↓
Request B
```

No deberá quedar estado mutable de A disponible para B.

---

## 9. Phase 02 — Transport Normalization

Cada transporte deberá convertir su entrada en un modelo normalizado.

Ejemplo HTTP:

```text
HTTP Request
   ↓
HttpAuthenticationAdapter
   ↓
AuthenticationRequest
```

Ejemplo CLI:

```text
CLI Input
   ↓
CliAuthenticationAdapter
   ↓
AuthenticationRequest
```

---

## 10. AuthenticationRequest

El objeto normalizado podrá contener:

```text
transport
route metadata
host
method
headers relevant to auth
cookies relevant to auth
tenant hint
client metadata
device metadata
network metadata
runtime metadata
authentication hints
correlation id
```

No deberá contener información irrelevante del transporte.

---

## 11. Untrusted boundary

Todo dato proveniente del transporte deberá considerarse:

```text
UNTRUSTED
```

incluyendo:

```text
cookies
bearer tokens
API keys
usernames
emails
tenant hints
device identifiers
headers
client claims
```

hasta ser verificado o normalizado.

---

## 12. Input normalization

La normalización podrá incluir:

```text
header canonicalization
credential extraction hints
string normalization
identifier trimming
transport metadata extraction
route metadata translation
```

pero nunca:

```text
credential verification
identity trust
authorization
```

---

## 13. Phase 03 — Existing Authentication Recovery

Antes de iniciar una nueva autenticación, el sistema podrá comprobar si ya existe estado válido.

Ejemplo stateful:

```text
Session Cookie
    ↓
Session Store
    ↓
AuthenticationSession
    ↓
Context Reconstruction
```

Ejemplo stateless:

```text
Bearer Token
    ↓
Token Validation
    ↓
AuthenticationContext
```

---

## 14. Recovery strategies

Podrán existir:

```text
SessionRecoveryStrategy
TokenRecoveryStrategy
DelegatedContextRecoveryStrategy
ConnectionContextRecoveryStrategy
```

Cada una deberá producir un resultado explícito.

---

## 15. Recovery result

Posibles estados:

```text
NO_STATE
VALID_STATE
EXPIRED_STATE
REVOKED_STATE
INVALID_STATE
ERROR
```

---

## 16. Existing state short-circuit

Si se recupera un estado válido y la request no exige reautenticación:

```text
VALID_STATE
   ↓
AuthenticationContext
   ↓
pipeline may short-circuit
```

Esto evita repetir autenticación completa.

---

## 17. No unsafe short-circuit

No deberá realizarse short-circuit si:

```text
authentication expired
security version changed
session revoked
tenant mismatch
fresh authentication required
step-up required
token invalid
device binding invalid
```

---

## 18. Lazy recovery

En rutas públicas podrá diferirse la recuperación hasta que algún componente solicite identidad.

Ejemplo:

```text
public page
   ↓
no Auth::user()
   ↓
no identity provider query
```

---

## 19. Eager recovery

Rutas protegidas podrán exigir autenticación temprana.

```text
protected route
    ↓
authentication middleware
    ↓
recover or authenticate
```

---

## 20. Phase 04 — Firewall Resolution

El `AuthenticationFirewallResolver` determina qué configuración se aplica.

Inputs potenciales:

```text
route
path
host
tenant
transport
application
protocol
metadata
```

---

## 21. Firewall resolution flow

```text
AuthenticationRequest
        ↓
Compiled Firewall Matcher
        ↓
Matching Firewall
        ↓
Authentication Configuration
```

---

## 22. No matching firewall

Si ninguna definición coincide, la política deberá ser explícita.

Opciones posibles:

```text
public/no-auth context
default firewall
configuration error
```

Nunca deberá seleccionarse un firewall incorrecto por accidente.

---

## 23. Multiple firewall matches

La resolución deberá ser determinista.

Podrá usar:

```text
priority
specificity
declaration order
compiled precedence
```

La política exacta deberá ser documentada y testeada.

---

## 24. Firewall output

El firewall podrá definir:

```text
identity provider
authenticators
state mode
session strategy
token strategy
authentication policies
risk profile
MFA requirements
failure handlers
success handlers
```

---

## 25. Phase 05 — Authenticator Discovery

El sistema consultará los authenticators registrados para el firewall.

Ejemplo:

```text
web firewall

SessionAuthenticator
PasswordAuthenticator
PasskeyAuthenticator
```

---

## 26. supports()

Cada authenticator deberá poder declarar si soporta la entrada.

```php
$authenticator->supports($request);
```

El método deberá:

```text
be side-effect free
not verify credentials
not mutate request state
not query unnecessary external systems
```

---

## 27. Authenticator candidates

Resultado conceptual:

```text
AuthenticatorCandidateSet
```

que puede contener:

```text
0
1
N
```

candidatos.

---

## 28. Zero candidates

Si no existe authenticator compatible:

```text
public route
    → unauthenticated allowed

protected route
    → authentication required result
```

La decisión final depende del integration layer.

---

## 29. Multiple candidates

Cuando varios soportan la request deberán existir reglas deterministas.

Ejemplo:

```text
Bearer token present
Session cookie present
```

El sistema podrá:

```text
prioritize explicit credential
use configured precedence
reject ambiguous credential combinations
```

según política.

---

## 30. Ambiguous authentication

Determinadas combinaciones deberán considerarse inseguras.

Ejemplo conceptual:

```text
two incompatible credentials
        ↓
ambiguous identity source
        ↓
REJECT / policy-defined behavior
```

No deberá seleccionarse silenciosamente una identidad inesperada.

---

## 31. Authenticator selection metadata

La selección podrá producir:

```text
authenticator name
selection reason
priority
firewall
```

para observabilidad.

---

## 32. Phase 06 — Passport Creation

El authenticator seleccionado extraerá datos y creará un:

```text
AuthenticationPassport
```

---

## 33. Passport creation

Ejemplo password:

```text
email + password
      ↓
PasswordAuthenticator
      ↓
AuthenticationPassport
```

Ejemplo token:

```text
Bearer header
      ↓
BearerTokenAuthenticator
      ↓
AuthenticationPassport
```

---

## 34. Passport validation

Antes de procesarlo, deberá validarse estructuralmente.

Ejemplos:

```text
missing identity claim
malformed credential
unsupported method
invalid transaction reference
missing required protocol state
```

---

## 35. Structural rejection

Datos mal formados deberán poder producir:

```text
REJECTED
```

sin llegar a verificadores innecesarios.

---

## 36. Passport creation events

Podrán emitirse:

```text
AuthenticatorSelected
AuthenticationPassportCreated
AuthenticationPassportRejected
```

sin incluir secretos.

---

## 37. Phase 07 — Identity Resolution

El `IdentityResolver` utiliza la claim y el provider configurado.

```text
IdentityClaim
    ↓
IdentityResolver
    ↓
IdentityProvider
    ↓
Identity
```

---

## 38. Identity normalization

Antes de buscar podrá realizarse normalización controlada.

Ejemplos:

```text
case normalization where appropriate
email normalization
Unicode normalization
provider-specific canonicalization
```

Nunca deberá aplicarse una normalización que cambie semántica sin política explícita.

---

## 39. Tenant-aware identity lookup

En multi-tenancy:

```text
IdentityClaim
   +
TenantContext
    ↓
IdentityProvider
```

La consulta deberá estar ligada al tenant correcto.

---

## 40. Identity not found

El dominio podrá registrar internamente:

```text
IDENTITY_NOT_FOUND
```

pero la respuesta externa podrá ser genérica:

```text
INVALID_CREDENTIALS
```

---

## 41. Enumeration resistance

Cuando se utilice password authentication, el sistema podrá ejecutar un hash dummy o estrategia equivalente para reducir diferencias temporales entre:

```text
user exists
```

y:

```text
user does not exist
```

si el diseño concreto lo requiere.

---

## 42. Provider failure

Si el provider falla:

```text
database unavailable
LDAP timeout
remote IdP failure
```

el resultado deberá ser:

```text
ERROR
```

y nunca una autenticación válida.

---

## 43. Identity status checks

Después de resolver identidad podrán ejecutarse validaciones de autenticación como:

```text
identity active
identity authentication enabled
credential use allowed
tenant membership valid for auth
```

sin evaluar permisos de negocio.

---

## 44. Phase 08 — Credential Verification

Cada credencial será enviada al verifier adecuado.

```text
Credential
   ↓
CredentialVerifierRegistry
   ↓
CredentialVerifier
   ↓
CredentialVerificationResult
```

---

## 45. Verifier resolution

La resolución deberá ser determinista por:

```text
credential type
authenticator
provider
configuration
```

---

## 46. Password verification

Ejemplo:

```text
PasswordCredential
      ↓
PasswordCredentialVerifier
      ↓
PasswordHasher
      ↓
valid / invalid
```

El plaintext deberá descartarse tan pronto como sea posible.

---

## 47. Token verification

Ejemplo:

```text
TokenCredential
   ↓
TokenVerifier
   ↓
signature
issuer
audience
expiration
revocation
   ↓
verified claims
```

---

## 48. API key verification

Preferencia conceptual:

```text
key identifier
     +
secret
```

para permitir lookup eficiente sin almacenar la API key completa en plaintext.

---

## 49. Credential verification short-circuit

Una credencial inválida normalmente deberá detener el procesamiento del intento.

```text
INVALID_CREDENTIAL
        ↓
REJECTED
```

No deberán verificarse factores secundarios innecesarios.

---

## 50. Credential verifier error

Debe distinguirse:

```text
INVALID
```

de:

```text
ERROR
```

Ejemplo:

```text
wrong password
    INVALID

hashing service failure
    ERROR
```

---

## 51. Credential rehash

Una verificación exitosa podrá producir metadata:

```text
needsRehash = true
```

La actualización del hash deberá ocurrir mediante un proceso seguro y controlado.

---

## 52. Phase 08B — Factor Verification

Factores adicionales podrán estar ya presentes o requerirse posteriormente.

Ejemplo:

```text
password
   +
TOTP
```

---

## 53. Factors available in same request

En algunos protocolos:

```text
credential + second factor
```

pueden llegar simultáneamente.

Entonces ambos podrán verificarse dentro del mismo pipeline.

---

## 54. Factors requiring challenge

Si falta un factor requerido:

```text
current evidence
    ↓
policy / assurance evaluation
    ↓
CHALLENGE_REQUIRED
```

---

## 55. Factor replay protection

Factores one-time deberán marcarse como usados cuando corresponda.

Ejemplos:

```text
recovery codes
magic links
OTP
one-time challenge tokens
```

---

## 56. Phase 09 — Evidence Construction

Solo después de verificaciones exitosas se construirá:

```text
AuthenticationEvidence
```

---

## 57. Evidence contents

Podrá incluir:

```text
resolved identity
authentication method
verified credentials
verified factors
provider provenance
verification timestamps
safe metadata
```

---

## 58. Evidence builder

El builder deberá garantizar:

```text
no raw secrets
no unverified claims
no duplicate evidence
consistent timestamps
correct identity binding
```

---

## 59. Evidence immutability

Cada nueva etapa podrá producir nueva evidencia.

Ejemplo:

```text
Evidence V1:
password verified

        ↓ TOTP

Evidence V2:
password + TOTP verified
```

---

## 60. Phase 10 — Risk Evaluation

El Risk Engine analiza el contexto del intento.

```text
AuthenticationEvidence
        +
AuthenticationEnvironment
        ↓
RiskEvaluator
        ↓
AuthenticationRisk
```

---

## 61. Risk input

Puede incluir:

```text
new device
known device
network
IP reputation
authentication history
failed attempts
tenant
identity type
authentication method
protocol anomalies
```

---

## 62. Risk output

Ejemplo:

```text
score = 70
level = HIGH

signals:
    NEW_DEVICE
    UNUSUAL_NETWORK
```

---

## 63. Risk must not expose secrets

La telemetría de riesgo no deberá almacenar:

```text
password
OTP
full bearer token
private keys
```

---

## 64. Risk engine failure

Según política, un error del Risk Engine podrá:

```text
fail closed
require step-up
degrade to conservative policy
```

pero nunca elevar confianza.

La opción exacta deberá ser explícita.

---

## 65. Authentication Policy Evaluation

Las Authentication Policies determinan requisitos del login.

Inputs:

```text
identity
evidence
risk
environment
firewall
authentication requirements
```

---

## 66. Policy examples

```text
administrator requires AAL2
new device requires MFA
service identities require certificate
tenant admins require passkey
high risk requires step-up
```

---

## 67. Policy composition

Podrán combinarse mediante:

```text
ALL
ANY
NOT
threshold
custom policy chain
```

aunque el sistema deberá evitar un lenguaje excesivamente complejo si no es necesario.

---

## 68. Assurance Calculation

El sistema calcula el nivel de confianza alcanzado.

```text
AuthenticationEvidence
        +
risk/context
        ↓
AssuranceCalculator
        ↓
AuthenticationAssurance
```

---

## 69. Assurance inputs

Podrán considerar:

```text
factor count
factor diversity
cryptographic strength
hardware backing
freshness
provider trust
device binding
```

---

## 70. Assurance cannot be client-provided

El cliente nunca deberá declarar confiablemente:

```text
assurance = AAL2
```

El servidor deberá derivarlo.

---

## 71. Policy and assurance ordering

El pipeline recomendado será:

```text
evidence
   ↓
risk evaluation
   ↓
assurance calculation
   ↓
policy evaluation
```

aunque algunas políticas podrán influir en qué señales adicionales deben evaluarse.

El `AuthenticationProcessor` coordinará este orden.

---

## 72. Phase 11 — Authentication Decision

El resultado se representa mediante:

```text
AuthenticationDecision
```

Estados:

```text
AUTHENTICATED
CHALLENGE_REQUIRED
STEP_UP_REQUIRED
REJECTED
ERROR
```

---

## 73. AUTHENTICATED

Se produce cuando:

```text
identity resolved
credentials valid
required factors valid
risk policy satisfied
assurance sufficient
authentication policies satisfied
```

---

## 74. CHALLENGE_REQUIRED

Se produce cuando:

```text
authentication can continue
        +
additional evidence required
```

Ejemplo:

```text
TOTP required
```

---

## 75. STEP_UP_REQUIRED

Se produce cuando existe autenticación previa válida pero insuficiente.

Ejemplo:

```text
existing AAL1 session
        ↓
sensitive action requires AAL2
        ↓
STEP_UP_REQUIRED
```

---

## 76. REJECTED

Se produce cuando la autenticación no debe continuar.

Ejemplos:

```text
invalid credentials
revoked token
expired magic link
policy rejection
invalid challenge
```

---

## 77. ERROR

Representa falla del sistema.

Ejemplos:

```text
identity provider unavailable
key resolver failure
misconfiguration
session store unavailable during required operation
```

---

## 78. Decision reasons

Internamente podrán conservarse razones detalladas:

```text
INVALID_PASSWORD
TOKEN_REVOKED
ASSURANCE_INSUFFICIENT
DEVICE_NOT_TRUSTED
PROVIDER_ERROR
```

La capa externa decidirá cuáles son seguras de exponer.

---

## 79. Phase 11B — Challenge Creation

Si la decisión es:

```text
CHALLENGE_REQUIRED
```

se deberá crear o actualizar una:

```text
AuthenticationTransaction
```

---

## 80. AuthenticationTransaction creation

La transaction deberá incluir:

```text
transaction id
identity reference
verified evidence
pending requirements
challenge type
tenant
client binding
created at
expires at
```

sin raw secrets.

---

## 81. Challenge generation

Ejemplo:

```text
AuthenticationDecision
    CHALLENGE_REQUIRED
          ↓
ChallengeFactory
          ↓
TotpChallenge
```

---

## 82. Challenge persistence

Podrá persistirse mediante:

```text
session
cache
database
distributed store
signed temporary state
```

según strategy.

---

## 83. Challenge binding

El challenge deberá ligarse cuando corresponda a:

```text
transaction
identity
tenant
client
device
protocol nonce
```

para prevenir sustitución o replay.

---

## 84. Challenge response

La integración podrá convertirlo en:

```text
HTTP redirect
SPA structured response
API JSON response
CLI prompt
WebSocket message
```

sin cambiar el dominio.

---

## 85. Challenge continuation

La siguiente request deberá:

```text
load AuthenticationTransaction
        ↓
validate transaction
        ↓
validate challenge
        ↓
verify factor
        ↓
extend evidence
        ↓
resume pipeline
```

No deberá reiniciar obligatoriamente todo el login desde cero.

---

## 86. Expired transaction

```text
transaction expired
      ↓
REJECTED / restart required
```

Nunca deberá restaurarse evidencia antigua más allá de su ventana de validez.

---

## 87. Phase 12 — Authentication Context Establishment

Solo una decisión:

```text
AUTHENTICATED
```

podrá producir un `AuthenticationContext` normal.

---

## 88. ContextFactory input

El factory recibirá:

```text
Identity
AuthenticationEvidence
AuthenticationAssurance
Risk Result
AuthenticationEnvironment
Tenant
Device
Provenance
```

---

## 89. Context contents

Ejemplo:

```text
identity = user:123
method = password
factors = password + TOTP
assurance = AAL2
tenant = acme
device = device:98
authenticatedAt = ...
risk = LOW
```

---

## 90. Secret-free context

El Context no deberá incluir:

```text
raw password
raw OTP
raw token
MFA secret
client secret
private key
```

---

## 91. Context installation

Una vez creado:

```text
AuthenticationContext
        ↓
AuthenticationContextStorage
        ↓
RequestScope
```

A partir de ese momento:

```php
Auth::check();
Auth::identity();
Auth::context();
```

podrán resolver el mismo contexto.

---

## 92. Request memoization

El Context instalado será reutilizado durante la unidad de trabajo.

No deberán repetirse:

```text
identity lookup
token verification
session reconstruction
```

sin causa explícita.

---

## 93. Context replacement

Un contexto podrá reemplazarse solo mediante operaciones explícitas.

Ejemplo:

```text
AAL1 Context
   ↓
successful step-up
   ↓
AAL2 Context
```

---

## 94. Phase 13 — Persistence

Después de establecer contexto, la estrategia dependerá del modo del firewall.

```text
STATEFUL
STATELESS
HYBRID
```

---

## 95. Stateful completion

```text
AuthenticationContext
        ↓
SessionAuthenticationStrategy
        ↓
AuthenticationSession
        ↓
Session Store
```

---

## 96. Session creation

Al autenticarse por primera vez deberá contemplarse:

```text
session id regeneration
fixation protection
authentication metadata
security version
tenant binding
device binding
timestamps
```

---

## 97. Session regeneration

Después de login exitoso:

```text
old session id
     ↓
regenerate
     ↓
new authenticated session
```

para prevenir session fixation.

---

## 98. Stateless completion

```text
AuthenticationContext
        ↓
request scope only
```

o:

```text
AuthenticationContext
        ↓
Token Issuer
        ↓
issued token
```

según el flujo.

---

## 99. Hybrid completion

Una SPA puede:

```text
use session cookie
      +
structured API responses
```

sin requerir bearer token.

Otro sistema podrá:

```text
web session
      +
personal API token
```

para distintos canales.

---

## 100. Token issuance

Emitir un token no deberá ser un efecto automático de toda autenticación.

Debe depender de:

```text
firewall
flow
client
policy
requested grant
```

---

## 101. Success handlers

Después del establecimiento del estado podrán ejecutarse:

```text
session persistence
events
audit
telemetry
response metadata
redirect hints
token issuance where configured
```

mediante handlers explícitos.

---

## 102. Success event order

Orden conceptual recomendado:

```text
AuthenticationDecision = AUTHENTICATED
        ↓
AuthenticationContextCreated
        ↓
AuthenticationStatePersisted
        ↓
AuthenticationSucceeded
```

La semántica exacta deberá mantenerse consistente para listeners.

---

## 103. Failure handling pipeline

Un fallo deberá atravesar:

```text
Internal Failure
      ↓
Failure Classification
      ↓
Audit / Metrics
      ↓
Safe External Mapping
      ↓
AuthenticationResult
```

---

## 104. No information leakage

Ejemplo:

Internamente:

```text
IDENTITY_NOT_FOUND
```

Externamente:

```text
Invalid credentials
```

cuando la política de seguridad lo requiera.

---

## 105. Cleanup phase

Toda ejecución deberá tener una fase de cleanup.

Debe liberar:

```text
raw credentials
temporary sensitive buffers
request auth state
non-persisted transaction state
request-scoped references
```

---

## 106. Cleanup under exceptions

El cleanup deberá ocurrir también cuando exista:

```text
exception
timeout
handler failure
provider failure
```

mediante `finally` o lifecycle equivalente.

---

## 107. Request completion

Al finalizar:

```text
RequestScope
   ↓
destroy
```

El estado persistente válido permanece únicamente mediante:

```text
session
token
transaction store
other explicit persistence
```

---

## 108. Pipeline short-circuit rules

El pipeline podrá detenerse en puntos definidos.

### 108.1 Existing valid context

```text
valid session restored
      ↓
return authenticated
```

---

### 108.2 Unsupported authentication

```text
no auth mechanism
      ↓
guest/public result
```

según route policy.

---

### 108.3 Invalid passport

```text
malformed input
      ↓
REJECTED
```

---

### 108.4 Identity resolution failure

```text
identity unavailable
      ↓
REJECTED or ERROR
```

según causa.

---

### 108.5 Credential failure

```text
invalid credential
      ↓
REJECTED
```

---

### 108.6 Challenge required

```text
evidence insufficient but resumable
      ↓
CHALLENGE_REQUIRED
```

---

### 108.7 Policy rejection

```text
authentication policy rejects
      ↓
REJECTED
```

---

## 109. Pipeline processor architecture

Para evitar un `AuthenticationManager` gigantesco, podrán existir stages.

Ejemplo:

```php
interface AuthenticationPipelineStageInterface
{
    public function process(
        AuthenticationPipelineContext $context,
        AuthenticationPipelineNext $next
    ): AuthenticationResult;
}
```

---

## 110. Pipeline stages

Posibles implementaciones:

```text
RecoverExistingAuthenticationStage
ResolveFirewallStage
ResolveAuthenticatorStage
CreatePassportStage
ResolveIdentityStage
VerifyCredentialsStage
VerifyFactorsStage
BuildEvidenceStage
EvaluateRiskStage
CalculateAssuranceStage
EvaluatePolicyStage
CreateDecisionStage
EstablishContextStage
PersistAuthenticationStage
```

---

## 111. Pipeline extensibility

Plugins podrán incorporar stages únicamente en extension points definidos.

No deberá permitirse:

```text
arbitrary reorder of critical verification stages
```

sin controles.

---

## 112. Fixed security ordering

Determinadas relaciones deberán ser invariantes.

Por ejemplo:

```text
Credential verification
    before
AuthenticationContext creation
```

y:

```text
Token signature verification
    before
token claims trusted
```

---

## 113. Extension points

Puntos razonables:

```text
before firewall resolution
after authenticator selection
after identity resolution
after credential verification
before risk evaluation
after risk evaluation
before context creation
after successful authentication
```

---

## 114. Hook safety

Los hooks no deberán poder convertir:

```text
invalid credential
```

en:

```text
authenticated
```

sin utilizar una extensión de autenticación formal.

---

## 115. Events vs pipeline control

Los eventos serán principalmente observacionales.

Ejemplo:

```text
CredentialVerified
```

No deberán utilizarse para alterar silenciosamente la decisión de seguridad.

La lógica crítica utilizará contratos síncronos explícitos.

---

## 116. Lifecycle state machine

```text
NEW
 │
 ▼
NORMALIZED
 │
 ▼
FIREWALL_RESOLVED
 │
 ▼
AUTHENTICATOR_RESOLVED
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
EVIDENCE_READY
 │
 ▼
RISK_EVALUATED
 │
 ▼
POLICY_EVALUATED
 │
 ├─────────────► REJECTED
 │
 ├─────────────► ERROR
 │
 ├─────────────► CHALLENGE_PENDING
 │                  │
 │                  ▼
 │             FACTOR_VERIFIED
 │                  │
 │                  └───────┐
 │                          │
 ▼                          │
AUTHENTICATED ◄─────────────┘
 │
 ▼
CONTEXT_ESTABLISHED
 │
 ▼
STATE_PERSISTED
 │
 ▼
COMPLETED
```

---

## 117. Invalid state transitions

Se deberán impedir:

```text
NEW → AUTHENTICATED

PASSPORT_CREATED → CONTEXT_ESTABLISHED

CREDENTIALS_VERIFIED → SESSION_CREATED
without decision/context

CHALLENGE_PENDING → CONTEXT_ESTABLISHED
without challenge completion
```

---

## 118. Reauthentication lifecycle

Una identity ya autenticada podrá requerir confirmar credenciales nuevamente.

```text
Existing Context
     ↓
Reauthentication Requirement
     ↓
AuthenticationTransaction
     ↓
Credential / Factor Verification
     ↓
Fresh AuthenticationContext
```

---

## 119. Step-up lifecycle

```text
Existing Context AAL1
        ↓
Operation requires AAL2
        ↓
STEP_UP_REQUIRED
        ↓
Challenge
        ↓
Factor verified
        ↓
Context AAL2
```

El contexto original podrá mantenerse mientras el challenge está pendiente, pero no deberá considerarse suficiente para la operación sensible.

---

## 120. Fresh authentication

El sistema deberá distinguir:

```text
authenticated
```

de:

```text
recently authenticated
```

Ejemplo:

```text
authentication age = 8 hours
```

puede no satisfacer una policy:

```text
fresh authentication within 5 minutes
```

---

## 121. Logout lifecycle

Logout será un lifecycle explícito.

```text
Current AuthenticationContext
        ↓
Logout Request
        ↓
Session/Token Revocation
        ↓
Context Storage Clear
        ↓
AuthenticationLogoutEvent
```

---

## 122. Logout stateful

Podrá implicar:

```text
session invalidation
session cookie expiration
remember-me revocation
device token cleanup
```

---

## 123. Logout stateless

Para tokens completamente stateless puede no existir invalidación inmediata sin un mecanismo adicional.

Las estrategias posibles incluyen:

```text
short expiration
revocation list
token family revocation
security version
reference token
```

---

## 124. Logout all devices

Debe existir una operación diferente de logout simple.

```text
Identity
   ↓
Revoke All Sessions
   ↓
Increment Security Version
   ↓
Invalidate Refresh Tokens
```

según la arquitectura final.

---

## 125. Session recovery lifecycle

Request posterior:

```text
Session Cookie
     ↓
Session Infrastructure
     ↓
Authentication Session Record
     ↓
Validity Check
     ↓
Security Version Check
     ↓
Tenant Binding Check
     ↓
Identity Refresh if required
     ↓
AuthenticationContext
```

---

## 126. Token recovery lifecycle

```text
Authorization Header
     ↓
TokenCredential
     ↓
TokenAuthenticator
     ↓
TokenVerifier
     ↓
Claims Validation
     ↓
Revocation / Version Check
     ↓
Identity Resolution
     ↓
AuthenticationContext
```

---

## 127. Remember-me lifecycle

```text
No active session
    +
Remember-me credential
        ↓
PersistentAuthenticator
        ↓
token verification
        ↓
Identity resolution
        ↓
reduced or configured assurance
        ↓
AuthenticationContext
        ↓
new session
```

Remember-me no deberá asumir automáticamente el mismo assurance que autenticación fresca.

---

## 128. Password reset effect

Un cambio de password podrá disparar:

```text
credential version increment
security version increment
session invalidation
remember-me revocation
refresh-token revocation
```

según política.

---

## 129. Account recovery effect

Account recovery deberá considerarse operación de alta sensibilidad.

Después de una recuperación podrán aplicarse:

```text
revoke sessions
revoke tokens
rotate recovery credentials
require fresh MFA enrollment
raise audit severity
```

---

## 130. Concurrent requests

Dos requests simultáneas podrían intentar:

```text
refresh session
consume OTP
consume magic link
rotate token
```

El lifecycle deberá soportar atomicidad donde corresponda.

---

## 131. One-time credential atomicity

Ejemplo:

```text
Recovery Code
```

deberá consumirse atómicamente.

```text
verify
  +
mark consumed
```

no deberá permitir doble uso por carrera.

---

## 132. Session rotation concurrency

Durante rotación de sesiones deberá evitarse invalidar accidentalmente requests legítimas concurrentes sin estrategia definida.

Podrán aplicarse:

```text
grace period
token family
rotation id
previous id allowance
```

cuando sea necesario.

---

## 133. Authentication transaction concurrency

Una transaction deberá impedir:

```text
two independent completions
duplicate MFA consumption
replayed callback
```

mediante:

```text
versioning
atomic state transition
nonce
one-time completion
```

---

## 134. Pipeline idempotency

Ciertas fases podrán ser idempotentes.

Ejemplos:

```text
firewall resolution
authenticator discovery
safe identity lookup
```

Otras no:

```text
consume magic link
rotate refresh token
consume recovery code
```

El sistema deberá distinguirlas.

---

## 135. Correlation lifecycle

Cada intento podrá recibir:

```text
AuthenticationCorrelationId
```

desde el inicio.

Se propagará a:

```text
logs
traces
audit records
events
challenge transactions
```

sin convertirse en secreto.

---

## 136. Observability spans

Ejemplo:

```text
auth.lifecycle
├── auth.recover
├── auth.firewall.resolve
├── auth.authenticator.resolve
├── auth.passport.create
├── auth.identity.resolve
├── auth.credentials.verify
├── auth.factors.verify
├── auth.risk.evaluate
├── auth.policy.evaluate
├── auth.assurance.calculate
├── auth.context.create
└── auth.state.persist
```

---

## 137. Metrics

Métricas posibles:

```text
authentication_attempt_total
authentication_success_total
authentication_failure_total
authentication_challenge_total
authentication_step_up_total
authentication_latency
identity_provider_latency
credential_verify_latency
session_recovery_latency
```

---

## 138. Security metrics

También:

```text
invalid_credentials_total
token_reuse_detected_total
expired_session_total
revoked_token_total
mfa_failure_total
cross_tenant_auth_reject_total
```

---

## 139. Audit points

Deberán auditarse al menos:

```text
login success
login failure
MFA challenge
MFA success
MFA failure
step-up
session creation
session revocation
token issuance
token revocation
logout
account recovery
```

---

## 140. Sensitive values and lifecycle logging

Nunca deberán registrarse:

```text
password
OTP
full API key
full bearer token
recovery code
private key
passkey private material
```

---

## 141. Timing behavior

El lifecycle deberá evitar diferencias temporales innecesarias que revelen:

```text
identity existence
credential storage details
specific failure causes
```

especialmente en login y recovery.

---

## 142. Rate limiting placement

El Rate Limiter podrá intervenir en varios puntos.

```text
before expensive identity lookup
before password hashing
before OTP verification
before recovery flow
before token exchange
```

según estrategia.

---

## 143. Rate-limit short-circuit

Cuando el límite sea excedido:

```text
THROTTLED
```

podrá mapearse a un resultado específico de integración sin ejecutar verificaciones costosas.

---

## 144. Progressive delay

El sistema podrá aplicar:

```text
increasing delays
```

pero evitando bloquear threads/workers de manera ineficiente cuando exista una solución de scheduling o rate limiting mejor.

---

## 145. Cancellation

Flujos multi-step deberán poder cancelarse.

```text
AuthenticationTransaction
      ↓
CANCELLED
```

El estado intermedio deberá invalidarse.

---

## 146. Timeout handling

Cada fase que dependa de infraestructura externa deberá tener límites.

Ejemplos:

```text
LDAP
OIDC metadata
JWKS
remote identity provider
risk service
```

Un timeout deberá producir:

```text
ERROR / conservative fallback
```

según política explícita.

---

## 147. Circuit breaker integration

Proveedores remotos podrán integrarse con resiliencia:

```text
timeout
retry where safe
circuit breaker
fallback provider only if explicitly configured
```

Nunca deberá utilizarse un fallback que reduzca seguridad silenciosamente.

---

## 148. Retry safety

No se deberán reintentar automáticamente operaciones no idempotentes como:

```text
one-time token consumption
magic link consumption
refresh token rotation
```

sin mecanismos apropiados.

---

## 149. HTTP lifecycle example — Password login

```text
POST /login
    ↓
HttpAuthenticationAdapter
    ↓
AuthenticationRequest
    ↓
web Firewall
    ↓
PasswordAuthenticator
    ↓
Passport(email,password)
    ↓
IdentityProvider
    ↓
UserIdentity
    ↓
PasswordVerifier
    ↓
VerifiedCredential
    ↓
Evidence
    ↓
Risk LOW
    ↓
Assurance AAL1
    ↓
Policy SATISFIED
    ↓
AUTHENTICATED
    ↓
AuthenticationContext
    ↓
Session Regeneration
    ↓
AuthenticationSession
    ↓
Success Handler
```

---

## 150. HTTP lifecycle example — Password + MFA

```text
POST /login
    ↓
Password verified
    ↓
Evidence = Password
    ↓
AAL1
    ↓
Policy requires AAL2
    ↓
CHALLENGE_REQUIRED
    ↓
AuthenticationTransaction
    ↓
TotpChallenge
    ↓
SPA/Web response

POST /auth/mfa
    ↓
Transaction restored
    ↓
TOTP verified
    ↓
Evidence extended
    ↓
AAL2
    ↓
AUTHENTICATED
    ↓
Context + Session
```

---

## 151. API lifecycle example — Bearer token

```text
GET /api/orders
Authorization: Bearer ...
        ↓
api Firewall
        ↓
BearerTokenAuthenticator
        ↓
TokenCredential
        ↓
Signature/Token Verification
        ↓
Claims Verification
        ↓
Identity Resolution
        ↓
AuthenticationEvidence
        ↓
AuthenticationContext
        ↓
request-scoped
```

---

## 152. Passkey lifecycle example

```text
Begin Login
   ↓
AuthenticationTransaction
   ↓
Passkey Challenge
   ↓
Client Assertion
   ↓
PasskeyAuthenticator
   ↓
Challenge Validation
   ↓
RP / Origin Validation
   ↓
Signature Verification
   ↓
Identity Resolution
   ↓
Evidence
   ↓
Assurance
   ↓
AUTHENTICATED
```

---

## 153. OIDC lifecycle example

```text
Login with Enterprise IdP
        ↓
AuthenticationTransaction
        ↓
state + nonce created
        ↓
redirect
        ↓
callback
        ↓
transaction validation
        ↓
code exchange
        ↓
ID token signature
issuer
audience
nonce
expiration
        ↓
federated identity mapping
        ↓
AuthenticationEvidence
        ↓
Policy / Assurance
        ↓
AuthenticationContext
```

---

## 154. Machine-to-machine lifecycle

```text
Service A Request
      ↓
mTLS / Service Credential
      ↓
ServiceAuthenticator
      ↓
ServiceIdentity
      ↓
Credential Verification
      ↓
AuthenticationEvidence
      ↓
Service Authentication Policy
      ↓
AuthenticationContext
```

---

## 155. Queue lifecycle

Si un job necesita identidad delegada:

```text
HTTP Request
    ↓
Delegation Context Created
    ↓
safe serialized reference
    ↓
Queue
    ↓
Job Runtime
    ↓
Delegated Authentication Context Reconstruction
```

No deberá serializarse automáticamente el contexto HTTP completo.

---

## 156. WebSocket lifecycle

```text
Connection Handshake
      ↓
Authentication
      ↓
Connection AuthenticationContext
      ↓
messages
      ↓
token/session expiration check
      ↓
reauthentication if required
```

---

## 157. SPA lifecycle states

El backend deberá poder comunicar estados estructurados:

```text
authenticated
unauthenticated
challenge_required
step_up_required
session_expired
authentication_rejected
```

---

## 158. SPA challenge example

```json
{
    "authentication": {
        "status": "challenge_required",
        "type": "totp",
        "transaction": "txn_..."
    }
}
```

La transaction ID no deberá ser suficiente por sí sola para completar el challenge sin validaciones adicionales.

---

## 159. API status mapping

Conceptualmente:

```text
No authentication
    → 401

Authenticated but unauthorized
    → 403

Invalid login submission
    → transport/application-specific response

MFA challenge
    → 401/structured flow or protocol-specific response
```

La semántica exacta dependerá del endpoint, pero Authentication y Authorization deberán permanecer diferenciados.

---

## 160. Redirect safety

Cuando handlers generen redirects después de login deberán prevenir:

```text
open redirect
untrusted return URL
external redirect injection
```

mediante destinos validados.

---

## 161. Authentication middleware lifecycle

Middleware típico:

```text
Request
  ↓
Authentication Middleware
  ↓
AuthenticationManager
  ↓
AuthenticationResult
  ├── Authenticated → next()
  ├── Challenge → handler
  ├── Unauthenticated → handler
  └── Error → secure response
```

---

## 162. Controller lifecycle

Cuando la request llega al controller:

```text
AuthenticationContext
```

ya deberá estar disponible si la ruta requería autenticación.

El controller no deberá repetir:

```text
password checks
token parsing
session validation
```

---

## 163. Authorization handoff

Después de Authentication:

```text
AuthenticationContext
        ↓
Authorization Context Builder
        ↓
Authorization Engine
```

Authentication no deberá tomar decisiones como:

```text
can edit invoice
can delete user
```

---

## 164. Runtime isolation lifecycle

Al final de request:

```text
Auth Context Storage
     ↓
clear()

Authentication Request State
     ↓
discard

Credential objects
     ↓
release
```

Esto será obligatorio en workers persistentes.

---

## 165. Request scope reset

Podrá existir un contrato:

```php
interface AuthenticationRequestScopeResetterInterface
{
    public function reset(): void;
}
```

integrado con el lifecycle general del runtime.

---

## 166. Exception safety

Todo pipeline deberá asegurar:

```text
exception
   ↓
no AuthenticationContext established
   ↓
cleanup
   ↓
safe failure response
```

---

## 167. Partial state rollback

Si falla después de crear session state provisional:

```text
session created
    ↓
subsequent required persistence fails
```

el sistema deberá evitar dejar un estado parcialmente autenticado.

---

## 168. Commit point

Conviene definir conceptualmente un:

```text
Authentication Commit Point
```

antes del cual el estado aún no está activo y después del cual la autenticación se considera establecida.

Ejemplo:

```text
Context built
    ↓
state persistence successful
    ↓
COMMIT
    ↓
AuthenticationSucceeded
```

---

## 169. Authentication atomicity

La creación de una autenticación stateful debería aproximarse a:

```text
verify
  +
create context
  +
persist session state
  +
activate request context
```

como una transición coherente.

---

## 170. Failure before commit

Si ocurre un error antes del commit:

```text
no active authenticated state
```

---

## 171. Failure after commit

Si un listener observacional falla después del commit, deberá definirse si:

```text
authentication remains valid
listener failure logged
```

o si ciertos listeners críticos forman parte del commit.

La distinción entre critical hooks y observability listeners será obligatoria.

---

## 172. Critical hooks

Ejemplos potenciales:

```text
session persistence
security state version check
one-time credential consumption
```

Estos sí podrán bloquear el commit.

---

## 173. Non-critical listeners

Ejemplos:

```text
analytics
non-security telemetry
secondary notifications
```

Su fallo no debería invalidar necesariamente una autenticación ya establecida.

---

## 174. Authentication lifecycle invariants

### AUTH-LIFE-01

No se crea `AuthenticationContext` antes de verificar la evidencia requerida.

#### AUTH-LIFE-02

Un challenge no equivale a un fallo.

#### AUTH-LIFE-03

Una transaction pendiente no equivale a una session autenticada.

#### AUTH-LIFE-04

Todo estado recibido del cliente se considera no confiable hasta validación.

#### AUTH-LIFE-05

Los secretos no sobreviven más de lo necesario.

#### AUTH-LIFE-06

Toda request tiene aislamiento de Authentication explícito.

#### AUTH-LIFE-07

Los errores inesperados fallan de forma cerrada.

#### AUTH-LIFE-08

El pipeline no evalúa permisos de negocio.

#### AUTH-LIFE-09

Una autenticación restaurada debe revalidar las condiciones exigidas por la estrategia.

#### AUTH-LIFE-10

El pipeline debe ser determinista ante múltiples authenticators.

#### AUTH-LIFE-11

Los one-time credentials deben consumirse de forma segura y resistente a replay.

#### AUTH-LIFE-12

Step-up produce nuevo estado de autenticación, no privilegios directamente.

#### AUTH-LIFE-13

El estado request-scoped debe limpiarse al final de la ejecución.

#### AUTH-LIFE-14

Las decisiones críticas deben producirse mediante contratos síncronos explícitos.

#### AUTH-LIFE-15

La observabilidad nunca deberá modificar silenciosamente el resultado de seguridad.

---

## 175. Pipeline result matrix

| Estado | Contexto autenticado nuevo | Transaction | Puede continuar inmediatamente |
| --- | --- | --- | --- |
| `AUTHENTICATED` | Sí | Puede cerrarse | Sí |
| `CHALLENGE_REQUIRED` | No | Sí | No |
| `STEP_UP_REQUIRED` | No nuevo todavía | Sí | No para operación protegida |
| `REJECTED` | No | Debe cerrarse/inutilizarse | No |
| `UNAUTHENTICATED` | No | No necesariamente | Solo si recurso público |
| `ERROR` | No | Debe manejarse de forma segura | No |

---

## 176. Stage responsibility matrix

| Stage | Responsabilidad principal |
| --- | --- |
| Transport Adapter | Normalizar entrada |
| Recovery | Restaurar estado existente |
| Firewall Resolver | Seleccionar configuración |
| Authenticator Resolver | Seleccionar mecanismo |
| Authenticator | Crear Passport |
| Identity Resolver | Resolver Identity |
| Credential Verifier | Verificar credenciales |
| Factor Verifier | Verificar factores |
| Evidence Builder | Consolidar evidencia verificada |
| Risk Evaluator | Evaluar riesgo |
| Assurance Calculator | Calcular assurance |
| Authentication Policy Engine | Validar requisitos |
| Decision Engine | Determinar resultado |
| Challenge Manager | Gestionar continuación |
| Context Factory | Crear estado autenticado |
| Persistence Strategy | Persistir session/token |
| Context Storage | Exponer autenticación a la request |
| Success/Failure Handler | Adaptar resultado al transporte |

---

## 177. Pipeline architecture recommendation

La implementación recomendada será una combinación de:

```text
AuthenticationManager
        +
explicit domain services
        +
ordered internal pipeline stages
```

El Manager será el coordinador visible, pero no concentrará la lógica.

---

## 178. Suggested internal flow

```php
$result = $authenticationManager->authenticate($request);
```

Internamente:

```text
normalize
recover
resolve firewall
resolve authenticator
create passport
process passport
evaluate risk
calculate assurance
evaluate policies
decide
establish context
persist
complete
```

---

## 179. No recursive authentication

El pipeline deberá protegerse contra llamadas recursivas accidentales.

Ejemplo:

```text
IdentityProvider
    ↓
calls Auth::user()
    ↓
starts Authentication again
```

Esto puede causar:

```text
recursion
deadlock
inconsistent state
```

Se deberán detectar estados `AUTHENTICATION_IN_PROGRESS`.

---

## 180. Reentrant access

Durante el pipeline, componentes que consulten el contexto actual deberán obtener:

```text
existing prior context
```

o:

```text
no active context
```

pero no un estado parcialmente construido.

---

## 181. Partial context prohibition

Nunca deberá exponerse algo como:

```text
identity resolved
credential not verified
```

a través de:

```php
Auth::user();
```

como si fuera autenticación válida.

---

## 182. Lifecycle security boundary

El punto exacto donde una identidad pasa de:

```text
candidate
```

a:

```text
authenticated identity
```

será la creación y activación exitosa de:

```text
AuthenticationContext
```

después de una decisión `AUTHENTICATED`.

---

## 183. Final lifecycle architecture

```text
                        EXTERNAL INPUT
                              │
                              ▼
                    Transport Normalization
                              │
                              ▼
                    AuthenticationRequest
                              │
                              ▼
                    Existing State Recovery
                              │
               ┌──────────────┴───────────────┐
               │                              │
          valid context                    no context
               │                              │
               │                              ▼
               │                     Firewall Resolution
               │                              │
               │                              ▼
               │                    Authenticator Resolver
               │                              │
               │                              ▼
               │                      Authenticator
               │                              │
               │                              ▼
               │                 AuthenticationPassport
               │                              │
               │                              ▼
               │                    Identity Resolution
               │                              │
               │                              ▼
               │                  Credential Verification
               │                              │
               │                              ▼
               │                    Factor Verification
               │                              │
               │                              ▼
               │                 AuthenticationEvidence
               │                              │
               │                  ┌───────────┼───────────┐
               │                  ▼           ▼           ▼
               │                Risk        Policy     Assurance
               │                  └───────────┼───────────┘
               │                              ▼
               │                  AuthenticationDecision
               │                              │
               │            ┌─────────────────┼─────────────────┐
               │            ▼                 ▼                 ▼
               │      AUTHENTICATED      CHALLENGE          REJECTED
               │            │                 │
               │            │                 ▼
               │            │         AuthenticationTransaction
               │            │                 │
               │            │                 ▼
               │            │              Challenge
               │            │
               └────────────┼─────────────────────────────┐
                            ▼                             │
                  AuthenticationContext                  │
                            │                             │
                  ┌─────────┴─────────┐                   │
                  ▼                   ▼                   │
               Session              Token                 │
                  │                   │                   │
                  └─────────┬─────────┘                   │
                            ▼                             │
                 Authentication Commit                    │
                            │                             │
                            ▼                             │
                  Request Context Storage                 │
                            │                             │
                            ▼                             │
                  Application / Authorization             │
                                                          │
Challenge continuation ───────────────────────────────────┘
```

---

## 184. Criterios de aceptación

El lifecycle será considerado correcto cuando:

1. pueda funcionar sin dependencia directa de HTTP;
2. restaure sesiones de forma segura;
3. soporte lazy y eager authentication;
4. resuelva firewalls determinísticamente;
5. resuelva múltiples authenticators;
6. construya Passports sin confiar en input;
7. resuelva identities de forma tenant-aware;
8. verifique credentials mediante componentes independientes;
9. soporte MFA y challenges multi-step;
10. construya Evidence sin secretos;
11. evalúe risk;
12. calcule assurance;
13. evalúe Authentication Policies;
14. produzca decisiones explícitas;
15. soporte step-up;
16. cree AuthenticationContext únicamente tras éxito;
17. soporte stateful, stateless e hybrid auth;
18. defina un commit point;
19. sea seguro ante errores parciales;
20. sea seguro en runtimes persistentes;
21. soporte observabilidad;
22. soporte auditoría;
23. prevenga replay;
24. maneje concurrencia;
25. preserve la separación con Authorization.

---

## 185. Regla final del lifecycle

La regla principal de todo el pipeline será:

```text
INPUT IS UNTRUSTED
        ↓
PROOF MUST BE VERIFIED
        ↓
VERIFIED PROOF BECOMES EVIDENCE
        ↓
EVIDENCE IS EVALUATED
        ↓
ONLY AN AUTHENTICATED DECISION
MAY CREATE AUTHENTICATION CONTEXT
```

En forma resumida:

> **Ninguna identidad será considerada autenticada por haber sido encontrada, por poseer una sesión, por presentar un token o por construir un Passport.**
> **La identidad se considera autenticada únicamente cuando el pipeline ha verificado la evidencia necesaria, satisfecho las políticas de Authentication y establecido un AuthenticationContext válido.**

---

## 186. Próximo documento

El siguiente documento será:

```text
04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md
```

Su responsabilidad será definir en profundidad el componente coordinador central del sistema, incluyendo:

```text
AuthenticationManager
AuthenticationProcessor
pipeline orchestration
stage execution
short-circuit rules
result propagation
request scoping
reentrancy
transaction coordination
context establishment
failure boundaries
extension points
performance
compilation integration
```

y establecer cómo VoltStack coordinará todo el Authentication System sin convertir al Manager en un componente monolítico.
