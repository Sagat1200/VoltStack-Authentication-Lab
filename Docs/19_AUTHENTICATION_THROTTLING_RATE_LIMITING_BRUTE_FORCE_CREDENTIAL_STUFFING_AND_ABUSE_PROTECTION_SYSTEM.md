# VoltStack Authentication System

## 19 — Authentication Throttling, Rate Limiting, Brute Force, Credential Stuffing and Abuse Protection System

- **Archivo:** `19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema transversal de protección contra abuso de Authentication.

**Depende especialmente de:**

- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`
- `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md`
- `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md`
- `18_ACCOUNT_RECOVERY_PASSWORD_RESET_IDENTITY_RECOVERY_AND_CREDENTIAL_REESTABLISHMENT_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema transversal encargado de proteger Authentication contra ataques automatizados, abuso deliberado, consumo excesivo de recursos y ataques distribuidos.
El sistema cubrirá:

- Authentication Throttling
- Rate Limiting
- Brute Force Protection
- Credential Stuffing Protection
- Password Spraying Protection
- Credential Guessing Protection
- Username Enumeration Protection
- Password Guessing
- OTP Guessing
- Recovery Token Guessing
- API Token Guessing
- MFA Challenge Abuse
- Recovery Abuse
- Passkey Challenge Abuse
- OAuth/OIDC Initiation Abuse
- Authentication Resource Exhaustion
- Distributed Authentication Attacks
- Adaptive Throttling
- Risk-Based Throttling
- Progressive Delay
- Security Cooldowns
- CAPTCHA Integration
- Distributed Counters
- Abuse Telemetry

El objetivo no será solamente limitar requests.
VoltStack deberá determinar:
Quién está intentando autenticar, contra qué Identity o credential, mediante qué mecanismo, desde qué contexto y con qué patrón de comportamiento.

## 2. Problema fundamental

Un Authentication endpoint es una superficie particularmente atractiva para atacantes.
Ejemplo:
POST /login
puede utilizarse para:

- password guessing
- credential stuffing
- password spraying
- account enumeration
- CPU exhaustion
- hashing exhaustion
- database exhaustion
- session exhaustion
- email flooding
- MFA flooding

Por tanto:

- HTTP Rate Limiting
- por sí solo no será suficiente.

## 3. Principio arquitectónico

VoltStack separará:
HTTP Rate Limiting
de:

- Authentication Abuse Protection
- El primero protege infraestructura HTTP.

El segundo entiende semántica de Authentication.

## 4. Ejemplo

Un HTTP limiter puede observar:
100 requests from IP X
El Authentication Abuse Protection System podrá observar:

- 100 login attempts
- from IP X

against 93 different identities
using PasswordAuthenticator
with mostly invalid passwords
Esto sugiere:
Credential Stuffing
o:
Password Spraying

## 5. Regla central

Rate Limiting será una primitive del sistema; Abuse Protection será la capa de decisión de seguridad construida sobre múltiples primitives y señales.

## 6. Arquitectura general

Authentication Request
│
▼
Authentication Manager
│
▼
Authenticator Resolution
│
▼
Abuse Protection Pre-Check
│
├── Source
├── Identity Claim
├── Tenant
├── Authenticator
├── Credential Type
├── Device
├── Network
└── Risk Signals
│
▼
Rate / Abuse Evaluation
│
├── ALLOW
├── MONITOR
├── THROTTLE
├── CHALLENGE
├── STEP_UP
├── COOLDOWN
└── DENY
│
▼
Authenticator
│
▼
Authentication Result
│
▼
Abuse Protection Post-Check
│
▼
Counters + Signals + Audit

## 7. AbuseProtectionManager

Será el orquestador principal.

```php
Conceptualmente:
interface AuthenticationAbuseProtectionManagerInterface
{
    public function evaluate(
        AuthenticationAttemptContext $context
    ): AbuseProtectionDecision;

    public function record(
        AuthenticationAttemptResult $result
    ): void;
}
```

## 8. Separación pre/post authentication

El sistema tendrá dos fases.
PRE_AUTHENTICATION
POST_AUTHENTICATION

## 9. Pre-authentication

Antes de operaciones costosas podrá decidir:

- ALLOW
- DELAY
- CHALLENGE
- DENY

Esto protege especialmente:

- password hashing
- database lookups
- external IdP calls
- MFA delivery

## 10. Post-authentication

Después del intento registrará:

- success
- invalid credential
- unknown identity
- invalid OTP
- invalid token
- expired credential
- risk rejection

## 11. AuthenticationAttemptContext

Conceptualmente:

```php
final readonly class AuthenticationAttemptContext
{
    public function __construct(
        public FirewallId $firewall,
        public AuthenticatorId $authenticator,
        public ?IdentityClaim $identityClaim,
        public ?IdentityReference $identity,
        public ?TenantReference $tenant,
        public NetworkContext $network,
        public DeviceContext $device,
        public AuthenticationPurpose $purpose,
        public AuthenticationRiskContext $risk,
    ) {}
}
```

## 12. No almacenar secrets

AuthenticationAttemptContext no deberá contener persistentemente:

- password
- OTP
- Bearer Token
- Recovery Token
- Passkey assertion secrets

## 13. Dimensiones de Rate Limiting

VoltStack soportará múltiples dimensiones simultáneamente.

- Source IP
- Identity
- Identity Claim
- Credential
- Authenticator
- Device
- Session
- Tenant
- Firewall
- Network
- ASN
- Recovery Destination
- Global Application

## 14. IP Limiting

Ejemplo:

- auth:ip:203.0.113.10
- Protege contra un único origen agresivo.

## 15. Limitación de IP no es suficiente

Atacantes pueden utilizar:

- botnets
- proxies
- residential proxies
- cloud infrastructure
- IPv6 pools

## 16. Identity Limiting

Ejemplo:

```php
auth:identity:{opaque-id}
Detecta múltiples ataques contra una misma Identity.
```

## 17. Problema de Identity Lockout

Si simplemente hacemos:

```text
5 failures
    ↓
lock account
```

un atacante puede bloquear cualquier cuenta conocida.

## 18. Regla

Los fallos provocados por actores no confiables no deberán otorgarles una primitive barata de denial-of-service contra una Identity legítima.

## 19. Identity Claim Limiting

Antes de resolver Identity:

```php
auth:claim:{digest}
Podrá utilizarse un digest keyed de la forma normalizada.
```

## 20. No almacenar email plaintext

Preferible:

```php
HMAC(rate_limit_key, normalized_claim)
en counters.
```

## 21. Credential Limiting

Cuando exista identificador seguro de credential:
auth:credential:{credential-id}

## 22. Authenticator Limiting

Ejemplo:

- auth:authenticator:password
- auth:authenticator:totp
- auth:authenticator:bearer
- auth:authenticator:recovery

## 23. Tenant Limiting

En multi-tenancy:

```php
auth:tenant:{tenant-id}
evita que abuso contra un tenant degrade a todos.
```

## 24. Global Limiting

Protección global contra:

- authentication storms
- bot attacks
- resource exhaustion

## 25. Hierarchical Limits

Un intento puede consumir simultáneamente:

```text
Global Budget
    ↓
Tenant Budget
    ↓
Firewall Budget
    ↓
Authenticator Budget
    ↓
IP Budget
    ↓
Identity Budget
```

## 26. No todos los budgets tienen igual consecuencia

Ejemplo:

```text
IP exceeded
    → throttle IP

Identity exceeded
    → require stronger challenge

Global exceeded
    → shed abusive traffic
```

## 27. RateLimitKey

Objeto conceptual:

```php
final readonly class RateLimitKey
{
    public function __construct(
        public RateLimitDimension $dimension,
        public string $opaqueKey,
    ) {}
}
```

## 28. RateLimitDimension

GLOBAL
TENANT
FIREWALL
AUTHENTICATOR
IP
NETWORK
DEVICE
IDENTITY
IDENTITY_CLAIM
CREDENTIAL
SESSION
RECOVERY_DESTINATION

## 29. RateLimitPolicy

interface RateLimitPolicyInterface
{
public function limitsFor(
AuthenticationAttemptContext $context
): RateLimitSet;
}

## 30. RateLimit

Conceptualmente:

```php
final readonly class RateLimit
{
    public function __construct(
        public RateLimitKey $key,
        public int $capacity,
        public RateLimitWindow $window,
        public RateLimitAction $action,
    ) {}
}
```

## 31. Algoritmos

VoltStack deberá permitir diferentes estrategias.

- Fixed Window
- Sliding Window
- Sliding Log
- Token Bucket
- Leaky Bucket

Generic Cell Rate Algorithm
Adaptive Limiting

## 32. Fixed Window

Ejemplo:
10 attempts / minute
Ventajas:

- simple
- fast
- low storage

Desventaja:
boundary bursts

## 33. Sliding Window

Permite comportamiento más uniforme.

```text
now - 60 seconds
        ↓
count attempts
```

## 34. Sliding Log

Mayor precisión, pero mayor consumo de storage.
No será necesariamente el default.

## 35. Token Bucket

Modelo:

```php
bucket capacity = 10
refill = 1 token / 6 sec
```

Cada intento consume un token.

## 36. Ventajas Token Bucket

Permite:

- small legitimate bursts
- controlled sustained rate

## 37. Leaky Bucket

Útil para suavizar flujos.

## 38. Algoritmo configurable

No habrá un único algoritmo obligatorio para todas las dimensiones.

## 39. Recommended V1

Para VoltStack V1:

- Token Bucket
- +;
- Fixed/Sliding Window counters

cubrirán la mayoría de necesidades.

## 40. RateLimiter

Contrato:

```php
interface RateLimiterInterface
{
    public function consume(
        RateLimitKey $key,
        RateLimitPolicy $policy,
        int $cost = 1
    ): RateLimitResult;
}
```

## 41. Atomicity

consume() deberá ser atómico.

## 42. Problema

No:

```text
GET counter = 4
GET counter = 4
```

Request A increments → 5
Request B increments → 5
permitiendo ambos incorrectamente.

## 43. Redis

Para instalaciones distribuidas será una implementación natural.

## 44. Redis operations

Podrán utilizar:

- INCR
- EXPIRE
- Lua scripts
- transactions
- sorted sets
- según algoritmo.

## 45. Lua / atomic scripts

Útiles para:

- read
- evaluate
- increment
- set expiration

en una sola operación lógica.

## 46. RateLimitStore

Contrato:

```php
interface RateLimitStoreInterface
{
    public function consume(
        RateLimitOperation $operation
    ): RateLimitResult;
}
```

## 47. Implementaciones

RedisRateLimitStore
MemoryRateLimitStore
DatabaseRateLimitStore
DistributedRateLimitStore

## 48. Memory store

Solo apropiado para:

- tests
- development
- single-process environments

## 49. FrankenPHP

No deberá utilizarse estado mutable process-global como authoritative limiter.

## 50. Ejemplo incorrecto

static $attempts = [];
Crítico bajo:

- FrankenPHP workers
- long-lived workers
- horizontal scaling

## 51. Brute Force Attack

Modelo:

- one identity
- many candidate passwords

## 52. Detección

Señales:

- many failures
- same identity
- different passwords
- one/few sources
- short interval

El sistema no necesita almacenar passwords para detectarlo.

## 53. Credential Stuffing

Modelo:

- many stolen username/password pairs
- tested across accounts

Patrón:

- many identities
- many attempts

moderate attempts per identity

## 54. Problema

Un limiter solo por Identity puede no detectarlo.

## 55. Necesidad multidimensional

Debe correlacionarse:

- source
- network
- tenant
- authenticator
- failure ratio
- identity cardinality

## 56. Password Spraying

Modelo:

- same/few passwords
- against many identities

VoltStack no deberá almacenar los passwords introducidos para comparar cuáles se repiten.

## 57. Detección segura

Puede basarse en:

- many identities
- same source/network

low attempts per identity
high failure ratio
temporal pattern
sin conservar password material.

## 58. Credential Guessing

Incluye:

- passwords
- API tokens
- recovery tokens
- OTP
- backup codes

Cada tipo tendrá policies diferentes.

## 59. Cost-aware limiting

No todos los intentos cuestan igual.

```text
Ejemplo:
unknown API token lookup
    cheap

Argon2 password verification
    expensive

external OIDC request
    network expensive

SMS OTP issuance
    monetary cost
```

## 60. AuthenticationCost

Podrá modelarse:

- LOW
- MEDIUM
- HIGH
- VERY_HIGH
- EXTERNAL_COST

## 61. Cost-based budget

Un attempt podrá consumir:

- 1 token
- 5 tokens
- 10 tokens
- dependiendo de costo.

## 62. Password Hashing DoS

Password hashing está diseñado para ser costoso.
Eso también crea una superficie DoS.

## 63. Ataque

thousands of login attempts
↓
thousands of Argon2 computations
↓
CPU/memory exhaustion

## 64. Pre-hash protection

Antes de verificar password deberán aplicarse límites baratos.

## 65. Pipeline

Request
↓
cheap network checks
↓
cheap rate-limit checks
↓
identity resolution
↓
identity-aware rate limit
↓
expensive password hash

## 66. Dummy Password Verification

Para reducir enumeration timing puede ser necesario verificar un dummy hash para Identity inexistente.

## 67. Resource protection tension

Dummy hashing mejora timing uniformity pero consume recursos.

## 68. VoltStack deberá balancear

enumeration resistance
vs
resource exhaustion resistance
mediante pre-hash limiting.

## 69. Dummy Hash

Debe ser:

- valid current algorithm hash
- y administrado por Password subsystem.

## 70. No dummy hash generation per request

Nunca:

```php
password_hash(random_bytes(...));
por request.
```

## 71. Precomputed dummy hash

Será reutilizable para verification work.

## 72. Progressive Throttling

VoltStack podrá incrementar delays según patrón.

```text
Ejemplo conceptual:
failure 1 → normal
```

failure 2 → normal
failure 3 → 250ms
failure 4 → 500ms
failure 5 → 1s
failure 6 → 2s
...

## 73. No worker sleeping blindly

En runtimes concurrentes:

```php
sleep(30);
puede ser una mala estrategia.
```

## 74. Preferible

Responder:

- 429 Too Many Requests
- Retry-After

o utilizar mecanismos async/runtime apropiados.

## 75. Server-side delay

Podrá existir solo para pequeños jitter/delays controlados.

## 76. Exponential Backoff

Puede utilizarse con límites máximos.

## 77. Unbounded exponential backoff

No.

## 78. Maximum cooldown

Toda escalation tendrá cap.

## 79. Progressive policy

Ejemplo:

```text
NORMAL
   ↓
OBSERVED
   ↓
THROTTLED
   ↓
CHALLENGE_REQUIRED
   ↓
SECURITY_COOLDOWN
```

## 80. AbuseProtectionDecision

Conceptualmente:

```php
final readonly class AbuseProtectionDecision
{
    public function __construct(
        public AbuseProtectionAction $action,
        public ?\DateTimeImmutable $retryAt,
        public SecuritySignalSet $signals,
        public AbuseDecisionReasonSet $reasons,
    ) {}
}
```

## 81. AbuseProtectionAction

ALLOW
ALLOW_AND_MONITOR
THROTTLE
REQUIRE_CHALLENGE
REQUIRE_STEP_UP
TEMPORARY_COOLDOWN
DENY

## 82. THROTTLE vs DENY

THROTTLE:
try later
DENY:

- security policy rejects attempt
- No son equivalentes.

## 83. CAPTCHA Integration

CAPTCHA podrá utilizarse para reducir automatización.

## 84. CAPTCHA no es Authentication

Nunca deberá producir:
IdentityEvidence

## 85. CAPTCHA representa

AutomationResistanceEvidence
o señal equivalente.

## 86. CAPTCHA boundary

VoltStack deberá definir interfaces, no acoplar Core a un proveedor específico.

## 87. BotChallengeProvider

interface BotChallengeProviderInterface
{
public function challenge(
BotChallengeContext $context
): BotChallenge;

public function verify(
BotChallengeResponse $response
): BotChallengeResult;
}

## 88. Providers

Podrán existir adapters para distintos proveedores externos.

## 89. Privacy

El uso de servicios CAPTCHA externos deberá ser explícito/configurable.

## 90. Challenge Escalation

Ejemplo:

```text
normal login
   ↓
suspicious automation
   ↓
CAPTCHA
   ↓
login continues
```

## 91. CAPTCHA cannot rescue invalid credentials

Resolver CAPTCHA no convierte password incorrecto en válido.

## 92. Account Lockout

VoltStack evitará permanent account lockout como default.

## 93. Hard Lock

Solo deberá ocurrir por:

- administrative action
- security incident
- Identity Security State
- explicit high-security policy

## 94. Failed attempts

Normalmente producirán:

- throttling
- cooldowns
- additional challenges
- risk elevation
- no permanent disablement.

## 95. Identity Lockout State

Si existe:
TEMPORARILY_AUTHENTICATION_RESTRICTED
deberá tener:

- reason
- expiresAt
- source

## 96. Attack-driven lockout prevention

No permitir:

```text
attacker knows email
        ↓
5 bad passwords
        ↓
victim locked indefinitely
```

## 97. Successful Authentication

Puede reducir algunos counters.

## 98. No universal reset

Un successful login desde IP A no necesariamente debe borrar evidencia de ataque desde IP B.

## 99. Counter scopes

Por tanto:

- source counters
- identity counters
- source+identity counters
- tendrán lifecycle independiente.

## 100. Composite Rate Keys

Ejemplo:

- IP
- Identity
- IP + Identity
- Tenant + IP
- Authenticator + IP
- Tenant + Identity

## 101. Composite key

auth:limit:{dimension}:{opaque-digest}

## 102. Key privacy

No incluir directamente:

- email
- username
- phone
- Bearer token

## 103. Secret-derived keys

Nunca usar:

```php
sha256(password)
para análisis de password spraying.
```

## 104. IP Normalization

IPv4 e IPv6 deberán manejarse correctamente.

## 105. IPv6

Limitar exclusivamente dirección completa puede ser insuficiente.
Puede ser útil considerar:

- network prefix
- según policy.

## 106. Proxy awareness

La IP del cliente deberá provenir del subsistema trusted proxy.

## 107. X-Forwarded-For

Nunca confiar ciegamente en headers enviados por cliente.

## 108. Trusted Proxy Integration

Authentication consume:

- TrustedClientNetworkContext
- del HTTP layer.

## 109. Network signals

Podrán incluir:

- client IP
- network prefix
- proxy status
- ASN
- country
- hosting-provider classification
- known anonymizer

cuando exista proveedor de intelligence.

## 110. Network intelligence is optional

Core no dependerá de servicios comerciales.

## 111. Risk integration

Abuse Protection podrá consumir:
AuthenticationRiskAssessment

## 112. Risk signals

Ejemplos:

- new device
- unusual location
- impossible travel
- known malicious network
- automation suspicion
- credential compromise signal

## 113. Risk may modify limits

Ejemplo:

```text
normal:
10 attempts/min
```

high risk:
3 attempts/min

## 114. Risk does not validate credential

Nunca.

## 115. Adaptive Rate Limiting

Los límites podrán cambiar según:

- system load
- attack level
- tenant risk
- network reputation
- authentication cost

## 116. Security floor

Adaptive limiting podrá endurecer.
No deberá debilitar automáticamente mínimos de seguridad.

## 117. Attack Mode

Podrá existir:

- NORMAL
- ELEVATED
- UNDER_ATTACK
- EMERGENCY

## 118. AttackModeProvider

Podrá derivarse de:

- security operations
- metrics
- automated detection

## 119. Under Attack

Puede reducir:

- anonymous authentication budgets
- recovery issuance
- OTP delivery

## 120. Legitimate users

La estrategia deberá minimizar daño a usuarios legítimos.

## 121. Credential Stuffing Detector

Podrá existir:

```php
interface CredentialStuffingDetectorInterface
{
    public function evaluate(
        AuthenticationBehaviorWindow $window
    ): CredentialStuffingAssessment;
}
```

## 122. Password Spraying Detector

Igualmente:

- PasswordSprayingDetector
- sin inspeccionar passwords.

## 123. BruteForceDetector

BruteForceDetector

## 124. Abuse detectors are signal producers

No deberán modificar Authentication directamente.
Producen:
SecuritySignal

## 125. SecuritySignal

Conceptualmente:

```php
final readonly class SecuritySignal
{
    public function __construct(
        public SecuritySignalType $type,
        public SecuritySignalSeverity $severity,
        public float $confidence,
        public \DateTimeImmutable $observedAt,
    ) {}
}
```

## 126. Signal Types

HIGH_FAILURE_RATE
IDENTITY_TARGETING
MULTI_IDENTITY_ATTACK
CREDENTIAL_STUFFING_SUSPECTED
PASSWORD_SPRAYING_SUSPECTED
OTP_GUESSING
TOKEN_GUESSING
RECOVERY_ABUSE
MFA_FLOODING
AUTOMATION_SUSPECTED
RESOURCE_EXHAUSTION

## 127. Confidence

No todo patrón es certeza.

- Por eso:
- confidence
- será útil.

## 128. Signal Aggregator

AuthenticationSecuritySignalAggregator
podrá combinar múltiples señales.

## 129. Password Authentication Protection

Deberá incluir:

- pre-hash limits
- identity limits
- source limits
- source+identity limits
- hashing resource budget
- failed attempt recording

## 130. Password Failure

No revelar externamente:

- unknown user
- wrong password
- disabled account

cuando ello genere enumeration.

## 131. Generic response

Ejemplo:
Invalid credentials.

## 132. Internal failure

Internamente:

- IDENTITY_NOT_FOUND
- PASSWORD_MISMATCH
- IDENTITY_DISABLED
- para audit/risk.

## 133. Password Spraying

Ataques lentos deberán detectarse mediante ventanas mayores.

- Ejemplo:
- 1 attempt/account/hour
- against 10,000 accounts

## 134. Multi-window analysis

Puede utilizar:

- 1 minute
- 15 minutes
- 1 hour
- 24 hours
- según signal.

## 135. Long-window counters

No todos necesitan precisión exacta.

## 136. Approximate counters

Para detección global pueden utilizarse estructuras eficientes en futuras versiones.

## 137. MFA OTP Protection

Los OTP tienen entropy limitada.
Necesitan controles fuertes.

## 138. OTP limits

attempts per challenge
attempts per Identity
attempts per source
challenge TTL

## 139. OTP challenge attempt counter

Debe ser authoritative.

## 140. Example

challenge allows 5 attempts
Después:
CHALLENGE_EXHAUSTED

## 141. New challenge

No deberá permitir resetear ilimitadamente el counter.

## 142. Challenge issuance limiter

Necesario.

## 143. MFA Flooding

Ataque:

```text
valid/stolen password
        ↓
```

repeated MFA push requests
↓
victim fatigue
↓
victim approves

## 144. MFA Push Protection

Debe limitar:

- push issuance
- repeated prompts
- parallel challenges

## 145. MFA fatigue signal

MFA_PROMPT_FATIGUE_SUSPECTED

## 146. MFA challenge replacement

Crear nuevo challenge no debe evadir límites anteriores.

## 147. Recovery Abuse

Documento 18 se integrará con este sistema.

## 148. Recovery initiation limiting

Dimensiones:

- source
- Identity claim
- Identity
- destination
- tenant

## 149. Email Flooding

Un atacante no debe poder enviar miles de:

- Password Reset Email
- a una víctima.

## 150. Destination limiter

Ejemplo:
recovery:destination:{opaque-digest}

## 151. Unknown Identity

Debe participar en source limits para evitar que enumeration de millones de emails sea gratis.

## 152. Recovery token guessing

Debe limitarse por:

- source
- selector misses
- transaction

aunque tokens sean high entropy.

## 153. Recovery OTP

Necesita límites aún más estrictos.

## 154. Recovery replay

Replays repetidos generan security signals.

## 155. Bearer Token Abuse

Bearer tokens suelen tener alta entropía.

- Pero endpoints pueden sufrir:
- random token flooding
- database/cache exhaustion

## 156. Token lookup architecture

Preferir identificadores/selectors que permitan lookup eficiente cuando diseño de token lo soporte.

## 157. Unknown token limiting

Antes de operaciones caras.

## 158. API authentication budgets

Pueden diferir de browser login.

## 159. Machine-to-machine

Puede necesitar:

- client-specific limits
- credential-specific limits
- network-specific limits

## 160. Passkey Abuse

Passkeys son resistentes a password guessing, pero endpoints WebAuthn aún pueden abusarse.

## 161. Passkey challenge generation

Debe limitarse.

## 162. WebAuthn challenge storage

Ataques podrían intentar crear millones de challenges.

## 163. Challenge creation budget

Debe existir:
passkey challenge issuance limiter

## 164. Assertion verification

También podrá limitarse por:

- source
- Identity
- credential

## 165. Invalid credential IDs

No deberán permitir un amplification/storage attack.

## 166. Federation Abuse

OAuth/OIDC endpoints pueden sufrir:

- login initiation flooding
- state creation flooding
- callback flooding
- provider request amplification

## 167. Federation initiation limits

Antes de crear:

- state
- nonce
- PKCE material
- aplicar limits.

## 168. Callback limits

Aplicar límites a:

- invalid state
- invalid code
- provider errors

## 169. External provider protection

No utilizar IdP como amplification target.

## 170. Provider-specific budgets

oidc:provider:{provider-id}

## 171. Authentication Challenge Budget

Todos los challenge-based systems podrán compartir abstraction:
ChallengeBudget

## 172. Challenge types

MFA
Passkey
Recovery
Federation
Email Verification

## 173. Challenge lifecycle abuse

Deberá proteger:

- create
- verify
- resend
- replace
- cancel

## 174. AuthenticationResourceGovernor

VoltStack podrá introducir:

- AuthenticationResourceGovernor
- para proteger recursos caros.

## 175. Resource classes

PASSWORD_HASH_CPU
PASSWORD_HASH_MEMORY
DATABASE_LOOKUP
EXTERNAL_IDP
EMAIL_DELIVERY
SMS_DELIVERY
CHALLENGE_STORAGE
SESSION_CREATION

## 176. Resource Budget

Conceptualmente:
ResourceBudget

## 177. Password hashing concurrency

Podrá limitarse número simultáneo de hashes.

## 178. Hashing semaphore

Una implementación podría utilizar:

- bounded semaphore
- local por worker/process además de distributed rate limits.

## 179. Local resource protection

Aquí sí puede existir state local para proteger recursos locales.

## 180. Diferencia crítica

Authoritative Security Counter
→ distributed/shared

Local CPU Semaphore
→ process-local acceptable

## 181. FrankenPHP Worker Protection

Cada worker podrá tener:
max concurrent expensive auth operations

## 182. Fiber Safety

Semaphores/counters locales deberán ser concurrency-safe.

## 183. No Identity state in static properties

Nunca.

## 184. Authentication Admission Control

Antes de trabajo caro:

- AdmissionController
- puede decidir si el request entra.

## 185. Admission result

ADMITTED
DEFERRED
REJECTED

## 186. Load shedding

Durante ataque extremo:

- anonymous expensive authentication
- puede reducirse antes de afectar usuarios ya autenticados.

## 187. Authenticated traffic separation

Authentication budgets no deberían consumir todo el presupuesto de tráfico de sesiones válidas.

## 188. Tenant isolation

Un tenant atacado no deberá agotar todo el Authentication capacity.

## 189. Per-tenant budgets

Muy importante para SaaS.

## 190. Tenant burst allowance

Puede configurarse por plan/capacidad, pero seguridad mínima será global.

## 191. Trusted Network

Enterprise applications podrán tener policies especiales.

## 192. Trusted does not mean unlimited

Una corporate network comprometida también puede atacar.

## 193. Trusted network may modify

capacity
challenge policy
risk score
pero nunca desactivar toda protección.

## 194. Device Signals

Un device conocido puede influir.

## 195. Device ID is not proof

Nunca considerar client-provided device identifier como fuerte por sí solo.

## 196. Device limiter

Puede complementar:
IP + Identity + Device

## 197. Privacy-preserving identifiers

Device fingerprints deberán manejarse cuidadosamente y ser opcionales.

## 198. Authentication Failure Taxonomy

Internamente:

- INVALID_CREDENTIAL
- UNKNOWN_IDENTITY
- INVALID_OTP
- INVALID_TOKEN
- EXPIRED_CREDENTIAL
- REVOKED_CREDENTIAL
- INELIGIBLE_IDENTITY
- ABUSE_THROTTLED
- RISK_REJECTED
- CHALLENGE_FAILED

## 199. External taxonomy

Más reducida:

- AUTHENTICATION_FAILED
- TOO_MANY_ATTEMPTS
- ADDITIONAL_VERIFICATION_REQUIRED
- TEMPORARILY_UNAVAILABLE

## 200. Enumeration resistance

No solo afecta mensajes.

- También:
- status codes
- response times
- response sizes
- rate-limit headers
- challenge availability
- redirects

## 201. Rate-limit header leakage

Ejemplo:

- X-RateLimit-Remaining: 2
- solo para cuentas existentes puede revelar existencia.

## 202. Public headers

Deberán diseñarse sin crear side channels.

## 203. Retry-After

Puede utilizarse para throttling genérico.

## 204. Timing jitter

Podrá utilizarse moderadamente.
No sustituye arquitectura correcta.

## 205. Response size normalization

Opcional para perfiles de alta seguridad.

## 206. Authentication attempt identifier

Cada intento podrá tener:
AuthenticationAttemptId

## 207. Purpose

Permite correlacionar:

- logs
- traces
- security signals

sin usar Identity PII.

## 208. Abuse Event

final readonly class AuthenticationAbuseEvent
{
public function __construct(
public AuthenticationAttemptId $attempt,
public AbuseEventType $type,
public SecuritySignalSet $signals,
public \DateTimeImmutable $occurredAt,
) {}
}

## 209. Audit events

AuthenticationThrottled
AuthenticationCooldownApplied
BruteForceSuspected
CredentialStuffingSuspected
PasswordSprayingSuspected
OtpGuessingSuspected
RecoveryAbuseSuspected
MfaFloodingSuspected
TokenGuessingSuspected
AuthenticationResourceLimited
AttackModeChanged

## 210. Audit privacy

Nunca:

- password
- OTP
- Bearer token
- Recovery token
- full authorization header

## 211. IP logging

Dependerá de privacy policy y regulación.

- Puede utilizarse:
- raw short-retention security log
- pseudonymized long-term metrics

## 212. Metrics

Ejemplos:

- auth_attempts_total
- auth_failures_total
- auth_throttled_total
- auth_challenges_required_total
- auth_bruteforce_suspected_total
- auth_credential_stuffing_suspected_total
- auth_password_spraying_suspected_total
- auth_otp_guessing_total
- auth_recovery_abuse_total
- auth_resource_rejected_total

## 213. Rate limiter metrics

auth_rate_limit_consumed_total
auth_rate_limit_rejected_total
auth_rate_limit_store_errors_total
auth_rate_limit_latency

## 214. Safe labels

authenticator
firewall
tenant class
action
reason category
credential type

## 215. Avoid high cardinality

No usar:

- identity_id
- email
- IP
- attempt_id
- token_id
- como metric labels.

## 216. Tracing

Spans:

- auth.abuse.evaluate
- auth.rate_limit.consume
- auth.abuse.signals
- auth.abuse.decision
- auth.resource.admission
- auth.abuse.record_result

## 217. Trace secrets

Nunca incluir credentials.

## 218. Security telemetry pipeline

Authentication Attempts
↓
Security Signals
↓
Abuse Detection
↓
Metrics / Audit
↓
Security Operations

## 219. Event-driven analysis

Advanced detectors podrán consumir events asincrónicamente.

## 220. Synchronous vs asynchronous

Synchronous:
must protect current request
Asynchronous:
detect broader attack patterns

## 221. Example

Synchronous:

```text
IP exceeded token bucket
    → throttle
```

Asynchronous:

```text
50,000 identities targeted across 2 hours
    → credential stuffing campaign detected
```

## 222. Feedback loop

Asynchronous detector podrá publicar:

- NetworkRiskSignal
- AttackMode
- TemporarySecurityRule

## 223. Feedback safety

Debe evitarse un detector defectuoso bloqueando globalmente Authentication sin límites/control.

## 224. Security rule TTL

Reglas automáticas temporales deberán expirar.

## 225. Manual override

Security operators podrán elevar protección.

## 226. Emergency Mode

Ejemplo:

```text
password authentication
    ↓
CAPTCHA required
    ↓
lower rate limits
```

sin necesariamente deshabilitar Passkeys.

## 227. Prefer stronger authenticators under attack

Passkeys pueden recibir políticas más permisivas que Password por su resistencia al credential stuffing.

## 228. Authenticator-aware protection

Importante.

## 229. Password

Mayor exposición a:

- guessing
- stuffing
- spraying
- hashing DoS

## 230. Passkey

Mayor exposición a:

- challenge flooding
- storage abuse
- malformed assertion abuse

pero no password guessing.

## 231. Bearer

Mayor exposición a:

- token guessing
- random lookup flooding
- stolen token replay

## 232. Recovery

Mayor exposición a:

- email flooding
- enumeration
- token guessing
- account takeover

## 233. MFA

Mayor exposición a:

- OTP guessing
- push fatigue
- challenge flooding

## 234. Federation

Mayor exposición a:

- state flooding
- callback flooding
- provider amplification

## 235. Policy per Authenticator

'authenticators' => [

'password' => [
'profile' => 'password_interactive',
],

'passkey' => [
'profile' => 'passkey_interactive',
],

'bearer' => [
'profile' => 'api_token',
],

];

## 236. AbuseProtectionProfile

Podrá definir:

- rate limits
- cost
- challenge escalation
- cooldowns
- resource budget
- detectors

## 237. Configuration example

'authentication' => [

'abuse_protection' => [

'enabled' => true,

'store' => 'redis',

'profiles' => [

'password_interactive' => [

'limits' => [
'ip' => '30/minute',
'identity' => '10/15 minutes',
'ip_identity' => '5/minute',
],

'progressive_throttling' => true,

'bot_challenge' => 'adaptive',
],

],

],

];
Los valores son ilustrativos, no defaults normativos.

## 238. Global security floor

El framework podrá imponer límites máximos razonables para determinadas operaciones críticas.

## 239. Application override

La aplicación podrá endurecer.

## 240. Tenant override

Puede endurecer.

## 241. Tenant cannot disable security floor

No:

```php
rate_limiting = false
para un mecanismo donde el framework exige protección mínima.
```

## 242. Policy hierarchy

Framework Security Floor
↓
Application Policy
↓
Firewall Policy
↓
Authenticator Policy
↓
Tenant Policy
↓
Risk Adjustment
↓
Attack Mode Adjustment

## 243. More restrictive composition

Como regla general:
more restrictive effective limit wins

## 244. Exception

Budgets jerárquicos no necesariamente se sustituyen.
Pueden aplicarse todos.

## 245. EffectiveAbuseProtectionPolicy

Objeto compilado para un intento.

## 246. Policy compilation

Configuraciones estáticas podrán compilarse durante bootstrap.

## 247. Request-time resolution

Solo resolver:

- Identity
- tenant
- network
- risk
- device
- dinámicamente.

## 248. Performance

Abuse Protection debe ser más barato que el trabajo que intenta proteger.

## 249. Regla

No diseñar un sistema anti-abuso cuyo costo por request sea mayor que ejecutar Authentication directamente.

## 1. Fast path

Para requests legítimos:

- few cache lookups
- atomic counter operations
- minimal allocations

## 2. Counter batching

Puede considerarse en telemetry.
No para authoritative enforcement cuando reduzca seguridad.

## 3. Store failure

Pregunta crítica:
What happens if Redis is unavailable?

## 4. Fail-open vs fail-closed

No existe una única respuesta para todos los contextos.

## 5. Password login

Una policy podría usar:

- degraded local limiter
- temporalmente.

## 6. High-security admin authentication

Podría:

- fail closed
- si authoritative abuse protection no está disponible.

## 7. Recovery

Puede ser más apropiado fail closed para operaciones sensibles.

## 8. AbuseStoreFailurePolicy

FAIL_CLOSED
DEGRADED_LOCAL_PROTECTION
FAIL_OPEN_WITH_ALERT

## 9. FAIL_OPEN_WITH_ALERT

Solo deberá permitirse explícitamente donde riesgo lo tolere.

## 10. Local degraded limiter

Podrá proteger al menos el worker contra resource exhaustion.

## 11. Distributed consistency

Counters no necesitan siempre serializabilidad global perfecta.
Pero:

- challenge attempt limits
- OTP attempts
- single-use semantics

sí requieren mayor consistencia.

## 12. Distinción

Approximate Abuse Counter
vs
Authoritative Credential Attempt Counter

## 13. OTP challenge counter

Authoritative.

## 14. Global credential stuffing estimate

Puede ser approximate.

## 15. Rate limit expiration

Todas las keys temporales deberán tener TTL.

## 16. Key leak prevention

No crear counters sin expiración indefinidamente.

## 17. Cardinality attack

Un atacante puede enviar millones de usernames aleatorios.

## 18. Riesgo

Si cada claim genera:
Redis key
por horas/días:
memory exhaustion

## 19. Counter cardinality defense

Podrá utilizar:

- short TTL
- source-first limiting
- bounded claim tracking
- probabilistic structures
- aggregation

## 20. Unknown claim strategy

No necesariamente crear expensive long-lived Identity counter para cada username inexistente.

## 21. Source-first protection

Muy importante.

## 22. Database enumeration DoS

No realizar múltiples queries complejas antes de source limits.

## 23. Authentication query budgets

Podrán existir para proteger Identity Provider/DB.

## 24. Provider-level limiter

identity-provider:{provider-id}

## 25. Multi-database tenants

Un tenant atacado no deberá saturar todos los pools.

## 26. Circuit Breaker integration

Provider failures repetidos pueden integrarse con resilience subsystem.

## 27. Abuse vs outage

No confundir:
provider failing
con:
attacker failing authentication

## 28. Security Events

Failures técnicos no deberán incrementar Identity credential-failure counters como si fueran passwords incorrectos.

## 29. Example

database timeout
no significa:
INVALID_PASSWORD

## 30. AuthenticationAttemptOutcome

SUCCESS
CREDENTIAL_FAILURE
SECURITY_REJECTION
TECHNICAL_FAILURE
CANCELLED

## 31. Counter behavior

Cada outcome afecta counters de forma diferente.

## 32. Technical failure

Puede afectar:

- system health
- provider circuit breaker

pero no necesariamente:
Identity brute-force score

## 33. Successful Password Authentication

Puede producir señal positiva.

## 34. Trusted success

Puede disminuir progresivamente algunos risk scores.

## 35. No instant forgiveness

Un atacante que logra acertar password después de 1000 intentos es más sospechoso, no menos.

## 36. Success after attack

Debe generar:

- SUCCESS_AFTER_HIGH_FAILURE_RATE
- y posiblemente step-up.

## 37. Credential Stuffing Success

Especialmente peligroso.

## 38. Risk escalation

correct password

+;

credential stuffing campaign
↓
MFA / Passkey Step-Up

## 288. Abuse Protection can require Step-Up

Sí.

## 289. Boundary with MFA

Abuse system dice:

- additional assurance required
- MFA system ejecuta el factor.

## 290. Boundary with Authorization

Abuse Protection protege Authentication.
No decide:
can transfer money

## 291. Boundary with WAF

WAF protege:
HTTP/network attack patterns
Auth Abuse Protection protege:

- Authentication semantics
- Se complementan.

## 292. Boundary with generic RateLimiter

VoltStack podrá tener un Quantum\RateLimiter.
Authentication podrá consumirlo como primitive.

## 293. Recommended separation

Quantum\RateLimiter
generic algorithm/storage

Quantum\Auth\Abuse
Authentication-specific policies

## 294. Boundary with Risk Engine

Risk Engine
evaluates contextual risk

Abuse Protection
evaluates abusive behavior/rates

Authentication Manager
orchestrates final flow

## 295. Boundary with Identity Security State

Abuse Protection puede producir señales que provoquen:

- security restriction
- pero las transiciones durables pertenecen al Identity Security subsystem.

## 296. Security Hold

Ataques graves pueden solicitar:
IdentitySecurityHoldRequested

## 297. No direct arbitrary mutation

Detector no debería hacer:
$user->disabled = true;

## 298. Extensibility

VoltStack deberá permitir:

- custom RateLimitPolicy
- custom SecuritySignalProvider
- custom AbuseDetector
- custom BotChallengeProvider
- custom RateLimitStore
- custom AttackModeProvider
- custom ResourceGovernor

## 299. AbuseDetector interface

interface AuthenticationAbuseDetectorInterface
{
public function detect(
AuthenticationBehaviorContext $context
): SecuritySignalSet;
}

## 300. Provider priority

Detectors podrán ejecutarse según:

- priority
- cost
- scope

## 301. Cheap detectors first

Ejemplo:

```text
rate counter
    ↓
network reputation cache
    ↓
```

expensive external risk provider

## 302. Short-circuit

Si IP ya está severamente throttled:
do not call expensive external detector

## 303. Security Signal Provider failures

No deberán automáticamente aprobar request.

## 304. Policy decides fallback

Igual que otros providers.

## 305. Cache

Network/risk intelligence podrá cachearse.

## 306. Never cache raw credentials

Obvio pero normativo.

## 307. Testing — Rate Limiter

Debe cubrir:

- below limit
- exact limit
- over limit
- window expiration
- refill
- concurrency
- atomic consume
- TTL

## 308. Testing — Brute Force

one Identity
many failures
one source
multiple sources
success after failures

## 309. Testing — Credential Stuffing

many identities
same source
distributed sources
low attempts per identity
high aggregate failures

## 310. Testing — Password Spraying

low rate
many identities
long window
distributed timing
sin almacenar password samples.

## 311. Testing — Account DoS

Verificar que:

- attacker cannot permanently lock victim
- mediante fallos baratos.

## 312. Testing — Enumeration

Comparar:

- existing Identity
- unknown Identity
- disabled Identity

en:

- message
- status
- headers
- timing profile
- challenge behavior

## 313. Testing — OTP

attempt exhaustion
new challenge
challenge replacement
parallel attempts
replay

## 314. Testing — MFA Flooding

many push challenges
resend
parallel requests

## 315. Testing — Recovery Abuse

email flooding
unknown Identity
known Identity
destination limits
token guessing

## 316. Testing — Passkey

challenge flooding
invalid credential IDs
malformed assertions
storage exhaustion

## 317. Testing — Federation

state flooding
callback flooding
provider outage
invalid states

## 318. Testing — Redis failure

FAIL_CLOSED
DEGRADED_LOCAL_PROTECTION
FAIL_OPEN_WITH_ALERT

## 319. Testing — Distributed concurrency

Múltiples application nodes deberán respetar effective limit.

## 320. Testing — FrankenPHP

Request A:
Alice
Request B:
Bob
Request C:
Tenant B
No debe existir leakage de:

- Identity
- attempt state
- decision
- risk signals

## 321. Testing — Fiber concurrency

Dos fibers consumiendo mismo local resource budget deberán ser safe.

## 322. Testing — Cardinality Attack

Generar gran número de:

- random usernames
- random tokens
- random challenge IDs

y verificar memory/storage bounds.

## 323. Testing — Resource Exhaustion

Medir:

- CPU
- memory
- Redis operations
- DB queries
- password hash concurrency
- durante attack simulation.

## 324. Testing — Adaptive Policy

NORMAL
↓
UNDER_ATTACK
↓
NORMAL
sin dejar stale state.

## 325. Testing — Tenant Isolation

Ataque contra Tenant A no debe bloquear Tenant B injustificadamente.

## 326. Testing — Clock

Rate limiting depende del tiempo.
Usar:

- ClockInterface
- en lugar de llamadas dispersas al reloj.

## 327. Testing — Clock skew

Distributed stores deberán evitar dependencia peligrosa de clocks inconsistentes cuando sea posible.

## 328. Security invariants — General

AUTH-ABUSE-01
Todo Authenticator expuesto deberá participar en Abuse Protection.
AUTH-ABUSE-02
Los límites deberán aplicarse antes de operaciones criptográficas o externas costosas cuando sea posible.
AUTH-ABUSE-03
Ningún limiter deberá almacenar credentials plaintext.
AUTH-ABUSE-04
Los counters distribuidos utilizados para enforcement deberán soportar operaciones atómicas.
AUTH-ABUSE-05
Abuse Protection no sustituye validación criptográfica.

- AUTH-ABUSE-06
- CAPTCHA no constituye Identity Authentication Evidence.
- AUTH-ABUSE-07

Los límites deberán tener TTL/bounds apropiados.
AUTH-ABUSE-08
El sistema deberá resistir counter-cardinality attacks.

## 329. Security invariants — Identity

AUTH-ABUSE-ID-01
Un atacante remoto no deberá poder bloquear permanentemente una Identity mediante intentos inválidos.
AUTH-ABUSE-ID-02
Identity counters no serán la única dimensión de protección.

- AUTH-ABUSE-ID-03
- Successful Authentication no elimina automáticamente evidencia global de ataque.
- AUTH-ABUSE-ID-04

Unknown identities también consumen budgets apropiados.
AUTH-ABUSE-ID-05
Identity existence no deberá revelarse mediante diferencias triviales de rate limiting.

## 330. Security invariants — Password

AUTH-ABUSE-PWD-01
Password hashing deberá estar protegido mediante admission/rate limiting.
AUTH-ABUSE-PWD-02
Dummy hash verification no deberá ocurrir antes de controles baratos contra resource exhaustion.
AUTH-ABUSE-PWD-03
Passwords nunca serán utilizados como rate-limit keys.

- AUTH-ABUSE-PWD-04
- Password spraying detection no deberá almacenar candidate passwords.
- AUTH-ABUSE-PWD-05

Success después de patrones altamente sospechosos podrá requerir step-up.

## 331. Security invariants — MFA

AUTH-ABUSE-MFA-01
OTP challenges tendrán límites de intentos.

- AUTH-ABUSE-MFA-02
- Challenge regeneration no reseteará ilimitadamente los límites efectivos.
- AUTH-ABUSE-MFA-03

MFA push issuance estará limitado.
AUTH-ABUSE-MFA-04
Repeated MFA prompts deberán poder producir fatigue signals.

## 332. Security invariants — Recovery

AUTH-ABUSE-REC-01
Recovery initiation estará limitado.

- AUTH-ABUSE-REC-02
- Recovery destination flooding estará limitado.
- AUTH-ABUSE-REC-03

Unknown Identity recovery requests participarán en abuse protection.
AUTH-ABUSE-REC-04
Recovery token verification estará protegido incluso con tokens de alta entropía.

## 333. Security invariants — Distributed Runtime

AUTH-ABUSE-RT-01
Authoritative counters no vivirán exclusivamente en memoria de un FrankenPHP worker.
AUTH-ABUSE-RT-02
Request-scoped abuse context nunca será process-global.

- AUTH-ABUSE-RT-03
- Local resource governors podrán ser worker-local.
- AUTH-ABUSE-RT-04

Distributed counter operations utilizadas para enforcement serán atomic-safe.
AUTH-ABUSE-RT-05
Rate-limit store failures tendrán política explícita.

## 334. Anti-pattern — cinco intentos y bloquear cuenta

5 failures
↓
account disabled
No como estrategia general.

## 335. Anti-pattern — IP only

100 requests/IP
es insuficiente contra ataques distribuidos.

## 336. Anti-pattern — Identity only

Permite:
distributed account lockout

## 337. Anti-pattern — Rate limit después del hash

Argon2
↓
Rate Limiter
demasiado tarde para prevenir hashing DoS.

## 338. Anti-pattern — sleep(60)

Mantener workers bloqueados como throttling principal.

## 339. Anti-pattern — CAPTCHA siempre

Deteriora UX y accesibilidad innecesariamente.
Preferir adaptive challenge.

## 340. Anti-pattern — CAPTCHA = authenticated

Nunca.

## 341. Anti-pattern — almacenar passwords para detectar spraying

Nunca.

## 342. Anti-pattern — static array limiter

Especialmente incorrecto con FrankenPHP.

## 343. Anti-pattern — unlimited Redis keys

Un attacker puede provocar memory exhaustion.

## 344. Anti-pattern — reset all counters on success

Puede borrar evidencia importante de ataque.

## 345. Anti-pattern — technical failure counted as wrong password

Incorrecto.

## 346. Anti-pattern — rate limiter fail-open silently

Nunca sin policy y observabilidad.

## 347. Anti-pattern — one universal policy

Password, Passkey, MFA, Bearer y Recovery tienen amenazas diferentes.

## 348. Componentes principales

AuthenticationAbuseProtectionManager
AuthenticationAttemptContext
AuthenticationAttemptResult
AbuseProtectionDecision
AbuseProtectionAction

RateLimiter
RateLimitStore
RateLimitPolicy
RateLimitSet
RateLimitKey
RateLimitDimension
RateLimitResult

AuthenticationAbuseDetector
SecuritySignal
SecuritySignalSet
SecuritySignalAggregator

## 349. Detectors

BruteForceDetector
CredentialStuffingDetector
PasswordSprayingDetector
OtpGuessingDetector
TokenGuessingDetector
RecoveryAbuseDetector
MfaFatigueDetector
AutomationDetector

## 350. Resource Protection

AuthenticationResourceGovernor
AuthenticationAdmissionController
ResourceBudget
ResourceCost
HashingConcurrencyGovernor
ChallengeBudget

## 351. Adaptive Protection

AttackMode
AttackModeProvider
AdaptiveAbusePolicy
RiskBasedRateLimitPolicy
TemporarySecurityRule

## 352. Bot Protection

BotChallengeProvider
BotChallenge
BotChallengeContext
BotChallengeResult

## 353. Storage

RedisRateLimitStore
MemoryRateLimitStore
DatabaseRateLimitStore

## 354. Namespace sugerido

VoltStack\Quantum\Auth\Abuse
VoltStack\Quantum\Auth\Abuse\Contracts
VoltStack\Quantum\Auth\Abuse\RateLimit
VoltStack\Quantum\Auth\Abuse\Detection
VoltStack\Quantum\Auth\Abuse\Signal
VoltStack\Quantum\Auth\Abuse\Resource
VoltStack\Quantum\Auth\Abuse\Challenge
VoltStack\Quantum\Auth\Abuse\Policy
VoltStack\Quantum\Auth\Abuse\Storage

## 355. Estructura sugerida

src/Quantum/Auth/Abuse/
├── Contracts/
│   ├── AuthenticationAbuseProtectionManagerInterface.php
│   ├── AuthenticationAbuseDetectorInterface.php
│   ├── RateLimiterInterface.php
│   ├── RateLimitStoreInterface.php
│   ├── RateLimitPolicyInterface.php
│   └── BotChallengeProviderInterface.php
│
├── RateLimit/
│   ├── RateLimit.php
│   ├── RateLimitSet.php
│   ├── RateLimitKey.php
│   ├── RateLimitDimension.php
│   ├── RateLimitResult.php
│   ├── TokenBucketLimiter.php
│   ├── FixedWindowLimiter.php
│   └── SlidingWindowLimiter.php
│
├── Detection/
│   ├── BruteForceDetector.php
│   ├── CredentialStuffingDetector.php
│   ├── PasswordSprayingDetector.php
│   ├── OtpGuessingDetector.php
│   ├── TokenGuessingDetector.php
│   ├── RecoveryAbuseDetector.php
│   └── MfaFatigueDetector.php
│
├── Signal/
│   ├── SecuritySignal.php
│   ├── SecuritySignalSet.php
│   ├── SecuritySignalType.php
│   └── SecuritySignalAggregator.php
│
├── Resource/
│   ├── AuthenticationResourceGovernor.php
│   ├── AuthenticationAdmissionController.php
│   ├── ResourceBudget.php
│   ├── ResourceCost.php
│   └── HashingConcurrencyGovernor.php
│
├── Challenge/
│   ├── BotChallenge.php
│   ├── BotChallengeContext.php
│   └── BotChallengeResult.php
│
├── Policy/
│   ├── AbuseProtectionPolicy.php
│   ├── EffectiveAbuseProtectionPolicy.php
│   ├── AdaptiveAbusePolicy.php
│   └── RiskBasedRateLimitPolicy.php
│
├── Storage/
│   ├── RedisRateLimitStore.php
│   ├── MemoryRateLimitStore.php
│   └── DatabaseRateLimitStore.php
│
├── AuthenticationAttemptContext.php
├── AuthenticationAttemptResult.php
├── AbuseProtectionDecision.php
├── AttackMode.php
└── AuthenticationAbuseProtectionManager.php

## 356. Integración global

HTTP REQUEST
│
▼
Firewall
│
▼
AuthenticationManager
│
▼
AuthenticatorResolver
│
▼
AbuseProtectionManager
│
┌─────────────────┼──────────────────┐
▼                 ▼                  ▼
Rate Limits      Abuse Detectors    Resource Governor
│                 │                  │
└─────────────────┼──────────────────┘
▼
Security Signal Set
│
▼
Effective Abuse Policy
│
▼
Protection Decision
│
┌──────────┬───────┼───────┬──────────┐
▼          ▼       ▼       ▼          ▼
ALLOW    THROTTLE  CAPTCHA  STEP-UP    DENY
│
▼
Authenticator
│
┌────────────────┼────────────────┐
▼                ▼                ▼
Password          Passkey           Token
│                │                │
└────────────────┼────────────────┘
▼
Authentication Result
│
▼
Abuse Post-Processing
│
┌────────────┼────────────┐
▼            ▼            ▼
Counters      Signals      Audit
│
▼
Risk / Telemetry

## 357. Relación con Laravel y Symfony

Laravel ofrece primitives muy útiles alrededor de:

- RateLimiter
- ThrottleRequests
- Login throttling
- cache-backed counters

y permite construir fácilmente límites application-level.

- Symfony dispone de componentes y mecanismos como:
- RateLimiter Component
- Login throttling
- limiter factories
- fixed window
- sliding window
- token bucket

VoltStack conservará la facilidad de configuración de ambos mundos, pero Authentication no dependerá únicamente de un limiter genérico.
La arquitectura será:

```text
Generic Rate Limiter
        ↓
Authentication Rate Policies
        ↓
Behavior Detection
        ↓
Security Signals
        ↓
Risk-aware Abuse Protection
        ↓
Authentication Decision
```

Por tanto:

- Laravel/Symfony-style rate limiting
- será una primitive dentro de un sistema mayor.

## 358. Diferenciador VoltStack

El objetivo será evolucionar desde:
Too many requests from this IP.

- hacia:
- This source is targeting many identities,
- using Password Authentication,

with an abnormal failure ratio,
while the application is experiencing
elevated authentication traffic.
sin almacenar secrets ni introducir un sistema excesivamente costoso.

## 359. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte throttling de Authentication;
2. soporte rate limiting multidimensional;
3. soporte límites por IP;
4. soporte límites por Identity;
5. soporte límites por Identity Claim;
6. soporte límites por credential;
7. soporte límites por authenticator;
8. soporte límites por tenant;
9. soporte límites globales;
10. soporte composite keys;
11. soporte Fixed Window;
12. soporte Sliding Window;
13. soporte Token Bucket;
14. soporte stores distribuidos;
15. soporte Redis;
16. garantice atomic consume;
17. soporte TTL;
18. resista counter-cardinality attacks;
19. proteja password hashing;
20. soporte resource budgets;
21. soporte local concurrency governors;
22. detecte brute force;
23. detecte credential stuffing;
24. detecte password spraying;
25. detecte OTP guessing;
26. detecte token guessing;
27. detecte recovery abuse;
28. detecte MFA flooding;
29. proteja Passkey challenges;
30. proteja OAuth/OIDC flows;
31. soporte progressive throttling;
32. soporte cooldown;
33. soporte CAPTCHA adapters;
34. soporte risk integration;
35. soporte adaptive limits;
36. soporte Attack Mode;
37. evite permanent attacker-driven account lockout;
38. preserve enumeration resistance;
39. diferencie credential failures de technical failures;
40. soporte audit;
41. soporte metrics;
42. soporte tracing;
43. soporte custom detectors;
44. soporte custom stores;
45. soporte custom policies;
46. soporte multi-tenancy;
47. sea seguro bajo FrankenPHP;
48. sea seguro bajo concurrencia/fibers;
49. soporte store failure policies;
50. mantenga Authentication secrets fuera de counters, logs y telemetry.
51. Regla arquitectónica final

VoltStack deberá preservar el siguiente flujo:

```text
AUTHENTICATION ATTEMPT
        │
        ▼
CHEAP PRE-AUTH CHECKS
        │
        ▼
MULTI-DIMENSIONAL RATE LIMITS
        │
        ▼
ABUSE SIGNAL DETECTION
        │
        ▼
RESOURCE ADMISSION CONTROL
        │
        ▼
EFFECTIVE SECURITY POLICY
        │
        ├──────────── ALLOW
        │
        ├──────────── THROTTLE
        │
        ├──────────── CHALLENGE
        │
        ├──────────── STEP-UP
        │
        ├──────────── COOLDOWN
        │
        └──────────── DENY
                       │
                       ▼
                 AUTHENTICATOR
                       │
                       ▼
              AUTHENTICATION RESULT
                       │
                       ▼
               BEHAVIOR RECORDING
                       │
                       ▼
               SECURITY SIGNALS
                       │
                       ▼
             FUTURE ATTEMPT POLICY
```

La primera regla central será:
Authentication Abuse Protection no deberá depender de una sola dimensión como IP o Identity. VoltStack deberá combinar límites de origen, Identity, credential, tenant, authenticator, red y recursos para resistir ataques distribuidos.

La segunda:
Un atacante nunca deberá obtener una primitive barata para bloquear permanentemente una cuenta legítima simplemente enviando credenciales incorrectas.

La tercera:
Las operaciones criptográficamente costosas, especialmente Password Hashing, deberán estar detrás de controles baratos de admisión y rate limiting para impedir que las propias garantías criptográficas de VoltStack se conviertan en un vector de denial-of-service.

Y finalmente:
El sistema de protección contra abuso deberá ser transversal a Password, MFA, Passkeys, Bearer Tokens, Federation y Account Recovery, pero cada mecanismo conservará políticas adaptadas a su propio modelo de amenazas.

Siguiente documento
La secuencia puede continuar con:
`20_AUTHENTICATION_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL_SYSTEM.md`
Este documento permitirá separar formalmente abuso de riesgo.
El documento 19 responde principalmente:

- ¿Está este actor abusando o atacando
- el sistema de Authentication?

Mientras que el documento 20 responderá:

- Aunque las credenciales sean válidas,
- ¿qué tan confiable es este intento

de Authentication dentro de su contexto?
Ahí podremos diseñar:

```text
Authentication Risk Engine
│
├── Risk Assessment
├── Risk Score
├── Security Signals
├── Signal Providers
├── Device Risk
├── Network Risk
├── Geographic Risk
├── Behavioral Risk
├── Credential Risk
├── Identity Risk
├── Session Risk
├── Impossible Travel
├── New Device Detection
├── Known Device Recognition
├── Credential Compromise Signals
├── IP / Network Reputation
├── Authentication History
├── Adaptive Authentication
├── Dynamic MFA
├── Step-Up Decisions
├── Risk Thresholds
├── Risk Policies
├── Risk Decay
├── Signal Confidence
├── Explainability
├── False Positive Management
├── Privacy Boundaries
├── External Risk Providers
├── Audit
├── Observability
└── FrankenPHP / Distributed Runtime Safety
```

Con ello tendremos una separación arquitectónica muy importante:

```text
19 — ABUSE PROTECTION
        │
        │ "¿están atacando?"
        ▼
20 — RISK ENGINE
        │
        │ "¿qué tan confiable es este intento?"
        ▼
15 — MFA / STEP-UP
        │
        │ "¿necesitamos más evidencia?"
        ▼
AUTHENTICATION DECISION
```

Eso nos permitirá construir en VoltStack una capa de Adaptive Authentication mucho más avanzada que un simple sistema de login + rate limiter.
