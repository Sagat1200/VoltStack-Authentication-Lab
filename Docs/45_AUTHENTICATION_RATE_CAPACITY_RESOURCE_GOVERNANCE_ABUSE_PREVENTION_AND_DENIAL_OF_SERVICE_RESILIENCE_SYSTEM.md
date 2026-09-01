# VoltStack Authentication System

## 45 — Authentication Rate, Capacity, Resource Governance, Abuse Prevention and Denial-of-Service Resilience System

- **Archivo:** `45_AUTHENTICATION_RATE_CAPACITY_RESOURCE_GOVERNANCE_ABUSE_PREVENTION_AND_DENIAL_OF_SERVICE_RESILIENCE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Availability / Abuse Prevention / Resource Governance / Distributed Security
- **Dependencias principales:** 19, 20, 24, 25, 27, 29, 30, 31, 32, 33, 36, 37, 38, 39, 40, 41, 43, 44.

---

## 1. Propósito

Este documento define la arquitectura responsable de proteger la capacidad operativa del sistema de Authentication frente a:
Brute Force
Credential Stuffing
Password Spraying
Account Enumeration
OTP Flooding
SMS Pumping
Email Bombing
Recovery Abuse
Passkey/WebAuthn Abuse
Federation Abuse
Challenge Flooding
Transaction Flooding
Session Creation Flooding
Cryptographic Resource Exhaustion
Password Hashing Exhaustion
Database Exhaustion
Redis Exhaustion
KMS/HSM Exhaustion
Queue Saturation
External Provider Exhaustion
Multi-Tenant Noisy Neighbors
Bot Traffic
Application-Layer DoS
Distributed Denial of Service
Cost Amplification Attacks
Resource Starvation
Security-Control Starvation
El sistema no se limitará a:
"X intentos por minuto"
sino que gobernará la capacidad completa de Authentication.

## 2. Problema fundamental

Authentication es una superficie especialmente atractiva para ataques de agotamiento de recursos.
Una request aparentemente pequeña:
POST /login
puede provocar:
HTTP parsing
    ↓
Tenant resolution
    ↓
Identity lookup
    ↓
Password hash verification
    ↓
Risk analysis
    ↓
Session storage
    ↓
Audit
    ↓
Metrics
    ↓
Notification
    ↓
External provider calls
El atacante puede intentar convertir:
1 cheap request
en:
N expensive operations

## 3. Principio de asimetría

VoltStack deberá minimizar la asimetría:
Attacker Cost << Server Cost
especialmente antes de que la identidad del cliente haya sido establecida.

## 4. Objetivo

Buscar:
Attacker Cost ≈ Controlled Server Cost
o al menos:
Server Cost <= Explicit Security Budget

## 5. Authentication Availability Is Security

La disponibilidad del sistema de Authentication forma parte del modelo de seguridad.
Si un atacante puede impedir que usuarios legítimos:
login
reauthenticate
recover accounts
perform MFA
access administrative systems
revoke compromised sessions
entonces existe un problema de seguridad aunque ninguna credencial haya sido robada.

## 6. Seguridad vs disponibilidad

VoltStack no tratará:
Confidentiality
Integrity
Availability
como dimensiones completamente independientes.
Authentication necesita las tres.

## 7. Relación con documento 19

19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE...
responde principalmente:
¿Cuándo debemos limitar intentos de autenticación?

45 responde:
¿Cómo protegemos la capacidad total del sistema de Authentication y de sus dependencias?

1. Separación
19 — Attempt Security
        │
        ▼
45 — Resource Governance
2. Rate Limiting != Resource Governance
Un sistema puede respetar:
5 requests / minute / IP
y aun así sufrir DoS mediante:
millions of IPs
expensive password hashing
many tenants
distributed bots
external API amplification
queue flooding
3. Resource Governance Model
Authentication Request
        │
        ▼
Admission Control
        │
        ▼
Abuse Evaluation
        │
        ▼
Rate / Quota Evaluation
        │
        ▼
Capacity Evaluation
        │
        ▼
Resource Budget Reservation
        │
        ▼
Authentication Processing
        │
        ▼
Resource Accounting
4. Core Components
VoltStack deberá definir:
AuthenticationAdmissionController
AuthenticationRateLimiter
AuthenticationQuotaManager
AuthenticationResourceGovernor
AuthenticationCapacityManager
AuthenticationAbuseDetector
AuthenticationCostEstimator
AuthenticationBudgetManager
AuthenticationConcurrencyGovernor
AuthenticationLoadShedder
AuthenticationDependencyProtector
AuthenticationFairnessScheduler
AuthenticationDegradationManager
5. Admission Control
Toda operación costosa de Authentication podrá pasar por:
interface AuthenticationAdmissionControllerInterface
{
    public function decide(
        AuthenticationAdmissionRequest $request
    ): AuthenticationAdmissionDecision;
}
6. Admission Request
final readonly class AuthenticationAdmissionRequest
{
    public function __construct(
        public AuthenticationOperationType $operation,
        public AuthenticationResourceScope $scope,
        public AuthenticationCostEstimate $estimatedCost,
        public AuthenticationRequestSignals $signals,
    ) {}
}
7. Admission Decision
enum AuthenticationAdmissionDecisionType: string
{
    case Allow = 'allow';
    case AllowDegraded = 'allow_degraded';
    case Delay = 'delay';
    case Challenge = 'challenge';
    case RateLimited = 'rate_limited';
    case CapacityRejected = 'capacity_rejected';
    case AbuseRejected = 'abuse_rejected';
    case DependencyUnavailable = 'dependency_unavailable';
}

## 8. Admission != Authentication Decision

Admission responde:
"¿Puede esta operación consumir recursos ahora?"
Authentication responde:
"¿Ha demostrado el principal la identidad requerida?"

## 9. Nunca

high server load
→ consider user authenticated

## 10. Degraded Authentication

ALLOW_DEGRADED solo podrá degradar funcionalidades no esenciales.
Nunca requisitos de seguridad.

## 11. Ejemplo permitido

Login succeeds
↓
Security Center projection update delayed

## 12. Ejemplo prohibido

WebAuthn service overloaded
↓
skip MFA

## 13. Authentication Operation Type

enum AuthenticationOperationType: string
{
    case Identify = 'identify';
    case PasswordVerify = 'password_verify';
    case PasskeyVerify = 'passkey_verify';
    case MfaVerify = 'mfa_verify';
    case OtpIssue = 'otp_issue';
    case OtpVerify = 'otp_verify';
    case RecoveryStart = 'recovery_start';
    case RecoveryVerify = 'recovery_verify';
    case FederationStart = 'federation_start';
    case FederationCallback = 'federation_callback';
    case SessionCreate = 'session_create';
    case SessionRefresh = 'session_refresh';
    case Reauthentication = 'reauthentication';
    case StepUp = 'step_up';
    case CredentialEnroll = 'credential_enroll';
    case CredentialRotate = 'credential_rotate';
    case SecurityNotification = 'security_notification';
}

## 14. Different Costs

No todas las operaciones cuestan igual.
Identity lookup          → low/moderate
Password Argon2 verify   → high CPU + memory
WebAuthn verify          → crypto
SMS OTP                  → external cost
OIDC federation          → network/provider
Recovery                 → high security sensitivity
Key operation            → KMS/HSM capacity

## 15. Cost Model

final readonly class AuthenticationCostEstimate
{
    public function __construct(
        public int $cpuUnits,
        public int $memoryUnits,
        public int $databaseUnits,
        public int $cacheUnits,
        public int $cryptoUnits,
        public int $networkUnits,
        public int $externalProviderUnits,
        public int $financialCostUnits,
    ) {}
}

## 16. Cost Units

No necesitan representar directamente:
milliseconds
megabytes
dollars
Pueden ser unidades abstractas normalizadas.

## 17. Cost Estimator

interface AuthenticationCostEstimatorInterface
{
    public function estimate(
        AuthenticationOperationType $operation,
        AuthenticationCostContext $context
    ): AuthenticationCostEstimate;
}

## 18. Cost-Based Admission

Ejemplo:
Server has:
100 crypto units available

Password verify costs:
5 units

→ maximum ~20 concurrent equivalent operations
conceptualmente.

## 26. Resource Dimensions

VoltStack deberá considerar independientemente:
CPU
Memory
Database Connections
Database Query Capacity
Cache/Redis Connections
Cache Operations
Network
Cryptographic Operations
Password Hashing
KMS Operations
HSM Operations
External IdP Calls
SMS Provider Capacity
Email Provider Capacity
Queue Capacity
Worker Capacity
File Descriptors
Sockets
Application Threads/Fibers

## 27. Resource Budget

final readonly class AuthenticationResourceBudget
{
    public function __construct(
        public AuthenticationResourceBudgetId $id,
        public AuthenticationResourceScope $scope,
        public AuthenticationResourceLimits $limits,
    ) {}
}

## 28. Scope

enum AuthenticationResourceScopeType: string
{
    case Platform = 'platform';
    case Environment = 'environment';
    case Region = 'region';
    case Realm = 'realm';
    case Tenant = 'tenant';
    case Application = 'application';
    case Identity = 'identity';
    case Network = 'network';
    case Client = 'client';
    case Provider = 'provider';
}

## 29. Hierarchical Budgets

Platform Budget
      ↓
Region Budget
      ↓
Realm Budget
      ↓
Tenant Budget
      ↓
Application Budget

## 30. Hardening Principle

Un child scope no puede concederse más capacidad crítica de la permitida por su parent.

## 31. Budget Composition

Ejemplo:
Platform:
1000 password verifications/sec

Region:
400/sec

Tenant:
50/sec

Identity:
5/sec

## 32. Effective Limit

Será el límite más restrictivo aplicable según dimensión.

## 33. Rate Limit Dimensions

No limitar únicamente por IP.

## 34. Candidate Keys

IP
Network Prefix
Identity
Username Hash
Email Hash
Tenant
Realm
Application
Device
Session
Credential
Provider
ASN
Geographic Region
Operation
Client ID
Machine Identity
Workload Identity

## 35. Sensitive Identifier Protection

Rate-limit storage no debe requerir guardar emails/usernames en plaintext.

## 36. Identifier Fingerprint

Puede utilizarse:
HMAC(server-secret, normalized_identifier)
para buckets internos.

## 37. Why HMAC

Hash simple de emails conocidos puede ser enumerable.

## 38. Key Rotation

Rate-limit fingerprint key tendrá lifecycle gobernado por documento 31.

## 39. Rate Limit Key

final readonly class AuthenticationRateLimitKey
{
    public function __construct(
        public AuthenticationRateLimitDimension $dimension,
        public string $opaqueValue,
    ) {}
}

## 40. Composite Limits

Ejemplo:
IP + operation
Identity + operation
Tenant + operation
IP + identity
Provider + tenant

## 41. Layered Throttling

Global
  ↓
Region
  ↓
Tenant
  ↓
Network
  ↓
Identity
  ↓
Operation

## 42. Distributed Botnet Problem

IP-only protection falla ante:
100,000 IP addresses
×
1 attempt

## 43. Identity-Centric Defense

Password spraying puede requerir:
attempts per identity
attempts per identity across IPs
attempts across many identities from same network

## 44. Password Spraying Detection

Patrón:
1 IP
↓
many accounts
↓
few attempts each

## 45. Credential Stuffing Detection

Patrón:
many IPs
↓
many accounts
↓
known credential pairs

## 46. Distributed Attack Detection

Puede requerir agregaciones globales/regionales.

## 47. Abuse Signal

final readonly class AuthenticationAbuseSignal
{
    public function__construct(
        public AuthenticationAbuseSignalType $type,
        public float $confidence,
        public AuthenticationAbuseScope $scope,
        public DateTimeImmutable $observedAt,
    ) {}
}

## 48. Abuse Types

BRUTE_FORCE
PASSWORD_SPRAYING
CREDENTIAL_STUFFING
OTP_FLOOD
SMS_PUMPING
EMAIL_BOMBING
RECOVERY_ABUSE
CHALLENGE_FLOOD
TRANSACTION_FLOOD
FEDERATION_FLOOD
BOT_AUTOMATION
RESOURCE_AMPLIFICATION
ACCOUNT_ENUMERATION
SESSION_FLOOD
TOKEN_REFRESH_FLOOD

## 49. Abuse Detector

interface AuthenticationAbuseDetectorInterface
{
    public function evaluate(
        AuthenticationAbuseContext $context
    ): AuthenticationAbuseAssessment;
}

## 50. Abuse Assessment

final readonly class AuthenticationAbuseAssessment
{
    public function __construct(
        public AuthenticationAbuseLevel $level,
        public AuthenticationAbuseSignalSet $signals,
        public AuthenticationAbuseRecommendationSet $recommendations,
    ) {}
}

## 51. Abuse Levels

enum AuthenticationAbuseLevel: string
{
    case Normal = 'normal';
    case Elevated = 'elevated';
    case High = 'high';
    case Severe = 'severe';
    case Emergency = 'emergency';
}

## 52. Abuse != Risk

Documento 20 analiza principalmente riesgo de Authentication relacionado con un intento/principal/contexto.
Documento 45 analiza también:
platform abuse
resource abuse
cost abuse
capacity attacks

## 53. Relationship

Risk Engine
   │
   ├── Is this authentication suspicious?
   │
Abuse Engine
   │
   └── Is someone abusing Authentication infrastructure?

## 54. Shared Signals

Ambos pueden compartir signals.
Pero mantienen decisiones separadas.

## 55. Brute-Force Response

Opciones:
delay
throttle
challenge
temporary attempt restriction
require stronger method
network restriction
block
alert

## 56. Avoid Permanent Account Lockout

Atacante no debe poder bloquear permanentemente a la víctima mediante intentos fallidos.

## 57. Lockout DoS

Attacker
↓
5 wrong passwords
↓
Victim locked forever
es un anti-pattern.

## 58. Progressive Response

Preferir:
normal
↓
small delay
↓
larger delay
↓
stronger challenge
↓
temporary restriction
↓
incident response
según contexto.

## 59. Adaptive Delay

interface AuthenticationDelayStrategyInterface
{
    public function delay(
        AuthenticationAbuseAssessment $assessment
    ): AuthenticationDelay;
}

## 60. Delay Bounds

No crear delays que mantengan PHP workers bloqueados innecesariamente.

## 61. Async/Protocol-Friendly Delay

Preferir respuestas:
retry_after
o scheduling apropiado.

## 62. Sleep Anti-Pattern

sleep(60);
dentro del HTTP worker.

## 63. Token Bucket

VoltStack podrá soportar:
Token Bucket
Leaky Bucket
Fixed Window
Sliding Window
GCRA-like algorithms
Concurrency Semaphore
Quota Counter

## 64. Algorithm Independence

Core Authentication no dependerá de un único algoritmo.

## 65. Rate Limiter Contract

interface AuthenticationRateLimiterInterface
{
    public function consume(
        AuthenticationRateLimitRequest $request
    ): AuthenticationRateLimitDecision;
}

## 66. Atomicity

Distributed counters deberán ser atómicos.

## 67. Redis

Puede utilizar:
Lua
atomic commands
transactions
según adapter.

## 68. Database

Puede utilizar:
atomic update
row locking
advisory mechanisms
según driver.

## 69. Local Memory Limiter

Solo adecuado para:
single process
development
testing
supplementary node-local protection
No como única defensa distribuida.

## 70. Fail-Open vs Fail-Closed

Si rate-limit backend falla, no existe una respuesta universal.

## 71. Policy

enum AuthenticationLimiterFailureMode: string
{
    case FailClosed = 'fail_closed';
    case FailOpen = 'fail_open';
    case LocalFallback = 'local_fallback';
    case DegradedAdmission = 'degraded_admission';
}

## 72. Example

Admin privileged login:
limiter unavailable
→ potentially fail closed

## 73. Example

Normal low-risk login:
distributed limiter unavailable
→ local emergency limiter
puede ser policy aceptable.

## 74. Never Unlimited Fail-Open

Redis down
→ unlimited password hashing
prohibido.

## 75. Emergency Local Limiter

Cada node puede mantener un conservative local budget.

## 76. Conservative Budget

Si normalmente cluster permite:
1000 verifies/sec
un node aislado no asumirá los 1000.

## 77. Capacity Manager

interface AuthenticationCapacityManagerInterface
{
    public function snapshot(): AuthenticationCapacitySnapshot;
}

## 78. Capacity Snapshot

final readonly class AuthenticationCapacitySnapshot
{
    public function __construct(
        public AuthenticationCapacityState $state,
        public AuthenticationResourceAvailability $resources,
        public DateTimeImmutable $measuredAt,
    ) {}
}

## 79. Capacity States

enum AuthenticationCapacityState: string
{
    case Normal = 'normal';
    case Elevated = 'elevated';
    case Constrained = 'constrained';
    case Critical = 'critical';
    case Emergency = 'emergency';
}

## 80. Capacity Signals

CPU utilization
memory pressure
active workers
request queue depth
DB pool utilization
Redis latency
KMS latency
HSM queue depth
external provider latency
queue backlog
password hashing concurrency

## 81. Capacity Data Is Operational

No debe exponerse detalladamente a attackers.

## 82. Generic Client Response

En lugar de:
Argon2 worker pool exhausted
usar respuesta controlada.

## 83. Resource Reservation

interface AuthenticationResourceGovernorInterface
{
    public function reserve(
        AuthenticationResourceReservationRequest $request
    ): AuthenticationResourceReservation;
}

## 84. Reservation

interface AuthenticationResourceReservation
{
    public function acquired(): bool;

    public function release(): void;
}

## 85. RAII-Like Usage

Conceptualmente:
$reservation = $governor->reserve($request);

try {
    // expensive authentication work
} finally {
    $reservation->release();
}

## 86. Lease-Based Reservation

Distributed reservation podrá expirar para evitar capacity leaks por worker crash.

## 87. Concurrency Limits

Algunas operaciones necesitan:
maximum concurrent operations
más que requests/sec.

## 88. Password Hashing

Ejemplo crítico.

## 89. Password Hashing Cost

Argon2 está diseñado para consumir recursos deliberadamente.
Esto es bueno contra password cracking.
También crea una superficie DoS.

## 90. Password Verification Pool

VoltStack deberá poder limitar:
concurrent password verifications

## 91. Password Hash Governor

interface AuthenticationPasswordHashCapacityGovernorInterface
{
    public function acquire(
        PasswordHashOperation $operation
    ): PasswordHashCapacityLease;
}

## 92. Password Hashing Budget

Puede considerar:
algorithm
memory cost
time cost
parallelism
server memory
CPU
current load

## 93. Password Hash Policy

Security parameters no deben reducirse automáticamente bajo load.

## 94. Prohibido

server overloaded
→ lower Argon2 memory cost
para nuevas verificaciones.

## 95. Correct Response

server overloaded
→ admit fewer password verification operations

## 96. Hash Parameter Governance

Documento 11 define política criptográfica.
Documento 45 gobierna capacidad para ejecutarla.

## 97. Memory Exhaustion

Si:
Argon2 = 64 MB
1000 concurrent verifies
podría requerir decenas de GB.
Concurrency deberá limitarse.

## 98. Fake Password Hashing

Para prevenir enumeration, algunos sistemas ejecutan dummy hash cuando identity no existe.
Esto también puede amplificar DoS.

## 99. Dummy Verification Governance

VoltStack deberá equilibrar:
enumeration resistance
vs
resource exhaustion

## 100. Dummy Hash Pool

Dummy verification seguirá bajo mismo resource governor.

## 101. Unknown Identity Flood

No permitirá bypass de capacity controls mediante usernames inexistentes.

## 102. Enumeration-Safe Responses

No requieren que cada unknown identity consuma costo ilimitado.

## 103. Tiered Work

Puede utilizarse:
cheap admission checks
↓
rate/abuse checks
↓
resource reservation
↓
expensive hash

## 104. Expensive Work Last

Principio:
Realizar validaciones baratas y seguras antes de operaciones criptográficas costosas siempre que no introduzca canales laterales peligrosos.

  1. Timing Side Channels
La optimización no debe hacer trivialmente distinguible:
existing account
vs
non-existing account
  2. Response Equalization
Puede usar:
bounded timing normalization
dummy verification
generic responses
adaptive techniques
sin sleeps enormes.
  3. WebAuthn Capacity
WebAuthn verification consume menos memoria que Argon2 pero requiere:
signature verification
credential lookup
origin/RP validation
counter/state operations
  4. WebAuthn Flooding
Challenge creation debe estar limitada antes de almacenar millones de challenges.
  5. Challenge Budget
max active challenges per:
identity
session
client
IP/network
tenant
platform
  6. Challenge Replacement
En algunos flows puede preferirse:
new challenge supersedes old challenge
en lugar de acumular infinitamente.
  7. Transaction Budget
Documento 38.
max active authentication transactions
por scopes apropiados.
  8. Transaction Flood
Atacante no autenticado podría intentar crear:
millions of login transactions
  9. Anonymous Transaction Budget
Necesita límites por:
client
network
device/browser binding
application
tenant
global capacity
 10. Transaction State Size
Debe estar limitado.
 11. Payload Size
Documento 39.
Authentication state/challenge responses tendrán:
maximum token size
maximum parameter size
maximum nested depth
maximum field count
 12. Parser DoS
Antes de Authentication puede existir:
huge JSON
huge headers
huge cookies
huge form data
HTTP layer deberá limitarlo.
 13. Integration with HTTP
Quantum\Http y HttpKernel deberán aplicar límites antes de Quantum\Auth.
 14. Defense Layers
Reverse Proxy
      ↓
HTTP Server / FrankenPHP
      ↓
HTTP Kernel
      ↓
Authentication Admission
      ↓
Authentication Runtime
 15. Edge Protection
VoltStack puede integrarse con:
CDN
WAF
reverse proxy
load balancer
API gateway
pero no depender exclusivamente de ellos.
 16. Trusted Proxy Boundary
Client IP solo será confiable después de aplicar configuración explícita de trusted proxies.
 17. X-Forwarded-For
Nunca confiar ciegamente.
 18. IP Chain Normalization
Debe existir una única política central.
 19. IPv6
Rate limiting debe manejar correctamente IPv6.
 20. Prefix Aggregation
En algunos ataques puede limitarse por:
IPv4 /24-like network
IPv6 prefix
configurable.
 21. NAT Fairness
Muchos usuarios legítimos pueden compartir IP.
 22. Therefore
IP hard blocking agresivo puede causar collateral DoS.
 23. Multi-Dimensional Decisions
Preferir combinación:
IP

+

identity
+
device
+
tenant
+
behavior
+
risk

## 128. Device Signals

Documento 21.
Trusted device puede influir en fairness/risk.
Pero:
Trusted Device no otorga capacidad ilimitada.

  1. Session Signals
Authenticated session puede recibir budgets distintos de anonymous clients.
  2. Privileged Operations
Documento 32.
Podrán tener:
lower rate limits
stronger admission
reserved capacity
  3. Why Reserved Capacity
Durante ataque general, administradores deben poder:
investigate
revoke
freeze
respond
  4. Security Control Starvation
Ataque no debe consumir todos los recursos e impedir las operaciones necesarias para detenerlo.
  5. Reserved Security Capacity
final readonly class AuthenticationReservedCapacity
{
    public function __construct(
        public AuthenticationResourcePoolId $pool,
        public AuthenticationResourceLimits $minimumReserved,
    ) {}
}
  6. Reserved Operations
Ejemplos:
session revocation
account freeze
security epoch increment
incident response
break-glass login
security admin reauthentication
credential compromise response
  7. Break-Glass Capacity
Documento 32.
Puede disponer de pool reservado muy pequeño y altamente protegido.
  8. Break-Glass Abuse
Reservar capacidad no significa endpoint públicamente ilimitado.
  9. Service Accounts
Documento 33.
Machine Authentication requiere budgets propios.
 10. Machine Rate Keys
service identity
workload identity
credential ID
audience
tenant
source network
 11. Machine Traffic
No aplicar exactamente políticas humanas.
 12. Service-to-Service Capacity
Puede requerir:
requests/sec
concurrent requests
token issuance rate
signature verification rate
 13. Token Issuance Abuse
Machine client comprometido podría solicitar millones de tokens.
 14. Issuance Quotas
per client
per workload
per tenant
per audience
 15. Short-Lived Tokens
No justifican issuance ilimitada.
 16. Token Refresh Flood
Refresh endpoint es superficie DoS.
 17. Refresh Coalescing
Clientes oficiales pueden evitar múltiples simultaneous refreshes.
Server igualmente debe proteger endpoint.
 18. OTP Governance
OTP issuance es particularmente vulnerable.
 19. OTP Abuse Dimensions
per identity
per destination
per IP
per device
per tenant
per provider
global
 20. Destination Fingerprint
Email/teléfono deberá representarse mediante fingerprint protegido cuando sea posible.
 21. SMS Pumping
Atacante puede intentar generar cargos SMS.
 22. Financial Cost Budget
final readonly class AuthenticationFinancialCostBudget
{
    public function __construct(
        public AuthenticationBudgetScope $scope,
        public int $maximumUnits,
        public DateInterval $window,
    ) {}
}
 23. Provider Cost Protection
Ejemplo:
Tenant A:
max 1,000 SMS OTP/day
con emergency/security exceptions gobernadas.
 24. SMS Is Not Free Resource
Capacity model deberá contemplar:
money
provider quota
delivery capacity
fraud risk
 25. OTP Cooldown
No permitir:
Send code
Send code
Send code
Send code
cada segundo.
 26. OTP Resend
Puede reutilizar challenge lógico o generar nuevo código según policy.
 27. Old OTP
Si nuevo OTP invalida anterior:
atomic supersession
 28. OTP Verification Attempts
Limitados independientemente de issuance.
 29. OTP Guessing
Código de 6 dígitos requiere fuerte attempt limiting.
 30. TOTP
No tiene external delivery cost, pero sí:
guessing risk
verification load
 31. Email Bombing
Recovery/login-link endpoints no deben permitir usar VoltStack para inundar correo.
 32. Communication Governance
Documento 43.
45 decide capacidad/abuse.
43 decide comunicación/delivery.
 33. Notification Suppression
Repeated identical security notifications pueden agregarse/suprimirse cuando sea seguro.
 34. Never Suppress Critical Evidence
Audit permanece aunque comunicación al usuario se agregue.
 35. Recovery Abuse
Recovery es una superficie especialmente crítica.
 36. Recovery Start Budget
per identity
per destination
per network
per device
per tenant
 37. Recovery Enumeration
Rate responses no deben revelar:
account exists
recovery email exists
MFA exists
 38. Recovery Resource Isolation
Recovery traffic puede tener pool separado.
 39. Recovery Attack
No debe bloquear completamente login legítimo del usuario salvo policy de incidente.
 40. Federation Governance
OAuth/OIDC federation puede generar:
redirects
state records
PKCE state
nonce records
provider calls
callback verification
JWK refresh
 41. Federation Start Rate
Limitado.
 42. Callback Flood
Attacker puede llamar callback con basura.
 43. Cheap Rejection First
Antes de network calls:
validate parameter size
state structure/reference
transaction existence
provider binding
expiration
cuando sea seguro.
 44. JWK Refresh Attack
Atacante puede enviar tokens con miles de kid desconocidos.
 45. Never
unknown kid
→ fetch remote JWK every time
 46. Unknown kid Governance
Documento 31/33/39.
Debe usar:
trusted issuer
configured JWKS URL
refresh cooldown
negative cache
single-flight refresh
rate limits
circuit breaker
 47. Single-Flight
100 requests unknown kid
      ↓
1 controlled metadata refresh
no 100.
 48. SSRF Boundary
Token nunca proporciona una URL arbitraria para descargar keys.
 49. IdP Outage
Federation provider outage no debe causar request pile-up ilimitado.
 50. Circuit Breaker
interface AuthenticationDependencyCircuitBreakerInterface
{
    public function allow(
        AuthenticationDependencyId $dependency
    ): AuthenticationCircuitDecision;
}
 51. Circuit States
CLOSED
OPEN
HALF_OPEN
 52. Security Rule
Circuit breaker puede evitar llamadas.
No puede:
skip signature verification
skip issuer validation
accept unverified token
 53. Dependency Bulkheads
Cada external dependency puede tener pool propio.
 54. Bulkhead Pattern
Google OIDC Pool
Microsoft OIDC Pool
SMS Pool
Email Pool
KMS Pool
 55. Benefit
Falla de SMS provider no consume toda la capacidad de Authentication.
 56. KMS/HSM Governance
Operaciones criptográficas remotas pueden ser limitadas por:
QPS
latency
financial cost
hardware slots
 57. Key Operations
sign
verify
decrypt
encrypt
unwrap
attest
pueden tener budgets distintos.
 58. Local Verification
Cuando architecture lo permita:
public-key verification locally
puede reducir dependencia de KMS.
 59. Private-Key Operations
Permanecen protegidas.
 60. KMS Outage
No se sustituirá por:
hardcoded local private key
como fallback improvisado.
 61. Crypto Pool
interface AuthenticationCryptoCapacityGovernorInterface
{
    public function reserve(
        AuthenticationCryptoOperation $operation
    ): AuthenticationCryptoCapacityReservation;
}
 62. Database Governance
Authentication puede ser DB-intensive.
 63. Query Budgets
Operación de login no debe ejecutar queries sin límite.
 64. N+1 Security Problem
Authentication pipeline con plugins no debe generar:
100 DB queries per login
accidentalmente.
 65. Compiled Authentication
Documento 27.
Precompilar metadata reduce resource consumption.
 66. Query Budget Instrumentation
Development/testing puede verificar:
max queries
max query duration
por operation.
 67. DB Connection Pool
Authentication puede reservar pool mínimo para critical operations.
 68. Database Saturation
Admission control deberá reaccionar antes de:
100% connection pool exhaustion
 69. Redis Governance
Redis puede almacenar:
sessions
rate limits
nonces
replay markers
challenges
transactions
security epochs
 70. Redis Failure Blast Radius
No diseñar todas las defensas con idéntico failure mode si puede evitarse.
 71. Namespace Isolation
Keys deberán estar:
purpose-separated
tenant-aware where needed
TTL-bound
 72. Redis Memory DoS
Attacker puede intentar crear millones de:
rate buckets
transactions
challenges
nonce records
 73. Cardinality Governance
No crear un permanent bucket por cada random username enviado.
 74. Ephemeral Bucket TTL
Unknown identifier buckets tendrán TTL.
 75. Cardinality Cap
Puede limitarse número de new buckets por network/time window.
 76. Approximate Structures
Para ciertos abuse analytics pueden utilizarse:
probabilistic counters
sketches
Bloom-like structures
cuando false-positive semantics sean aceptables.
Nunca para authoritative credential validity.
 77. Session Creation Governance
Successful login también consume recursos.
 78. Session Flood
Cuenta comprometida podría generar millones de sessions.
 79. Session Limits
Puede existir:
maximum active sessions per identity
según policy.
 80. Session Limit Strategy
reject new
revoke oldest
require user action
allow with alert
según tenant/security policy.
 81. Privileged Sessions
Límites más estrictos.
 82. Remember-Me Credentials
No generar ilimitadamente.
 83. Passkeys
Enrollment también limitado.
 84. Credential Enrollment Flood
Atacante con compromised session no debe poder registrar miles de credentials.
 85. Method Inventory Limit
Documento 34.
Puede existir:
max passkeys
max TOTP methods
max federated links
max API credentials
 86. Limit Is Policy
No hardcode universal.
 87. Multi-Tenant Noisy Neighbor
Tenant A no debe consumir toda la capacidad de Tenant B.
 88. Tenant Budget
final readonly class AuthenticationTenantResourceBudget
{
    public function __construct(
        public TenantId $tenantId,
        public AuthenticationResourceLimits $limits,
        public AuthenticationBurstAllowance $burst,
    ) {}
}
 89. Tenant Baseline
Cada tenant puede recibir capacidad mínima.
 90. Burst Capacity
Capacidad ociosa puede compartirse.
 91. Borrowing
Tenant A idle
Tenant B busy
→ B may borrow capacity
sin consumir reserved critical capacity.
 92. Fairness
interface AuthenticationFairnessStrategyInterface
{
    public function allocate(
        AuthenticationCapacitySnapshot $capacity,
        AuthenticationDemandSnapshot $demand
    ): AuthenticationCapacityAllocation;
}
 93. Fairness Strategies
equal share
weighted share
plan-based
priority-based
reserved minimum + burst
 94. Security Floors
Plan comercial no debe permitir eliminar security floors.
 95. Premium Capacity
Puede ofrecer mayor legitimate capacity.
No menor seguridad.
 96. Tenant Attack Isolation
Si Tenant A sufre credential stuffing:
Tenant A → constrained
Tenant B → normal
idealmente.
 97. Shared IP Across Tenants
No bloquear toda plataforma únicamente por IP sin considerar contexto.
 98. Platform Attack
Si ataque es global, global controls prevalecen.
 99. Hierarchical Rate Evaluation
Global Limit
   │
   ├── Region
   │     ├── Tenant A
   │     └── Tenant B
   │
   └── Region
100. Quotas
Rate = velocidad.
Quota = cantidad acumulada.
101. Examples
10 attempts/minute
rate.
10,000 SMS/month
quota.
102. Quota Manager
interface AuthenticationQuotaManagerInterface
{
    public function consume(
        AuthenticationQuotaRequest $request
    ): AuthenticationQuotaDecision;
}
103. Quota Windows
hour
day
billing period
calendar month
rolling period
104. Security Operations and Quotas
No permitir que una billing quota impida:
critical compromise notification
sin una emergency policy.
105. Reserved Security Quota
Puede existir.
106. Capacity vs Quota
Capacity
→ what system can handle now

Quota
→ what scope may consume over period

## 235. Backpressure

Cuando downstream está saturado:
producer must slow down

## 236. Background Integration

Documento 44.
Authentication background queues deberán reportar:
queue depth
oldest message age
worker utilization
retry rate

## 237. Queue Saturation

Puede provocar:
stop non-essential task creation
coalesce projection jobs
aggregate notifications
defer cleanup

## 238. Cannot Defer Critical Security Truth

security epoch increment
no se posterga si es necesario para correctness.

## 239. Load Shedding

interface AuthenticationLoadShedderInterface
{
    public function decide(
        AuthenticationLoadSheddingContext $context
    ): AuthenticationLoadSheddingDecision;
}

## 240. Shed Order

Ejemplo conceptual:

1. Telemetry enrichment
2. Non-critical projections
3. Optional notifications
4. Maintenance
5. Expensive UX enhancements
6. Low-priority Authentication traffic
antes de tocar operaciones críticas.
7. Never Shed
Sin política explícita:
credential revocation
incident response
security epoch update
critical administrative recovery
8. Brownout Mode
VoltStack podrá entrar en modo degradado.
9. Brownout
Mantiene funciones esenciales reduciendo opcionales.
10. Brownout Example
Normal:
login

+ rich risk enrichment
+ device geolocation enrichment
+ projection refresh
+ notification enrichment

Brownout:
login

+ mandatory risk/security checks
  1. Mandatory vs Optional Dependency
Toda integración deberá declarar:
enum AuthenticationDependencyRequirement: string
{
    case Mandatory = 'mandatory';
    case SecurityCritical = 'security_critical';
    case Optional = 'optional';
    case ObservabilityOnly = 'observability_only';
}
  2. Dependency Registry
final readonly class AuthenticationDependencyDefinition
{
    public function __construct(
        public AuthenticationDependencyId $id,
        public AuthenticationDependencyRequirement $requirement,
        public AuthenticationResourcePoolId $pool,
    ) {}
}
  3. Degradation Matrix
Ejemplo:
Dependencia Falla Comportamiento
Audit authoritative store crítica fail according to security policy
SMS OTP SMS unavailable alternative only if policy permits
Security Center projection no crítica defer
SIEM export degradable outbox/retry
KMS required for signing crítica fail closed
Geolocation enrichment opcional omit
Distributed limiter crítica/degradable según flow local conservative fallback

  4. No Silent Fallback
Todo fallback debe estar definido por policy.
  5. Method Downgrade
Dependencia caída no justifica usar método débil.
  6. Example
Passkey provider unavailable
en WebAuthn normalmente no implica external provider; pero si alguna dependencia necesaria falla:
→ error / alternative only if requirement remains satisfied
  7. Security Requirement First
Documento 36 continúa siendo autoridad.
  8. Capacity Cannot Rewrite Requirement
require phishing resistant

+

system constrained
no se convierte en:
password only

## 253. Challenge Negotiation

Documento 38 puede negociar otra alternativa solo si:
alternative satisfies same effective requirement

## 254. Proof-of-Work

VoltStack puede permitir plugins de computational challenge para casos especializados.
Pero no será defensa base.

## 255. Why

Proof-of-work puede perjudicar:
mobile devices
accessibility
battery
legitimate low-power clients
y botnets aún pueden resolverlo.

## 256. CAPTCHA

Podrá integrarse como abuse mitigation.

## 257. CAPTCHA != Authentication Factor

Resolver CAPTCHA:
does not prove identity

## 258. CAPTCHA Trust

Es señal anti-automation.
No aumenta automáticamente Authentication Assurance Level.

## 259. CAPTCHA Provider Outage

No debe convertirse automáticamente en bypass.
Policy decide alternative.

## 260. Privacy

Abuse prevention puede recolectar:
IP
device signals
network patterns
behavior
Documento 42 gobierna minimización/retention.

## 261. Data Minimization

No almacenar más información anti-abuse de la necesaria.

## 262. Fingerprinting

Browser fingerprinting invasivo no será requisito arquitectónico.

## 263. Pluggable Signals

Tenants podrán elegir señales compatibles con:
privacy
regulation
risk appetite
dentro de platform floors.

## 264. Consent

Cuando legalmente corresponda, documento 42.

## 265. Retention

Rate-limit counters normalmente necesitan retención corta.

## 266. Abuse Intelligence

Datos agregados pueden tener retención distinta.

## 267. Raw IP Retention

Debe estar explícitamente gobernada.

## 268. Security Metadata

No convertirse en sistema de vigilancia general.

## 269. Account Enumeration via Rate Limits

Responses distintas pueden filtrar existencia.
Ejemplo incorrecto:
known account:
429 after 5 attempts

unknown account:
always 401

## 270. Enumeration-Resistant Limiting

Diseñar buckets/responses para reducir diferencias observables.

## 271. HTTP 429

Puede utilizarse cuando apropiado.
Pero no siempre debe revelar exactamente qué dimensión activó el límite.

## 272. Retry-After

Puede exponerse de forma controlada.

## 273. Precise Internal Reason

Audit puede registrar:
IDENTITY_PASSWORD_VERIFY_LIMIT
mientras cliente recibe mensaje genérico.

## 274. Failure Taxonomy Integration

Documento 25.

## 275. Error Codes

AUTH_RATE_LIMITED
AUTH_RATE_LIMIT_BACKEND_UNAVAILABLE
AUTH_RATE_LIMIT_LOCAL_FALLBACK_ACTIVE

AUTH_QUOTA_EXCEEDED
AUTH_QUOTA_BACKEND_UNAVAILABLE

AUTH_CAPACITY_CONSTRAINED
AUTH_CAPACITY_CRITICAL
AUTH_CAPACITY_RESERVATION_FAILED

AUTH_RESOURCE_CPU_EXHAUSTED
AUTH_RESOURCE_MEMORY_EXHAUSTED
AUTH_RESOURCE_DATABASE_EXHAUSTED
AUTH_RESOURCE_CACHE_EXHAUSTED
AUTH_RESOURCE_CRYPTO_EXHAUSTED
AUTH_RESOURCE_KMS_EXHAUSTED
AUTH_RESOURCE_QUEUE_EXHAUSTED
AUTH_RESOURCE_PROVIDER_EXHAUSTED

AUTH_PASSWORD_HASH_CAPACITY_EXCEEDED
AUTH_CRYPTO_CAPACITY_EXCEEDED

AUTH_CHALLENGE_CAPACITY_EXCEEDED
AUTH_TRANSACTION_CAPACITY_EXCEEDED
AUTH_SESSION_CAPACITY_EXCEEDED

AUTH_ABUSE_DETECTED
AUTH_BRUTE_FORCE_DETECTED
AUTH_PASSWORD_SPRAYING_DETECTED
AUTH_CREDENTIAL_STUFFING_DETECTED
AUTH_OTP_FLOOD_DETECTED
AUTH_SMS_PUMPING_DETECTED
AUTH_EMAIL_BOMBING_DETECTED
AUTH_RECOVERY_ABUSE_DETECTED
AUTH_BOT_AUTOMATION_DETECTED
AUTH_RESOURCE_AMPLIFICATION_DETECTED

AUTH_DEPENDENCY_CIRCUIT_OPEN
AUTH_DEPENDENCY_CAPACITY_EXCEEDED
AUTH_DEPENDENCY_DEGRADED

AUTH_TENANT_CAPACITY_EXCEEDED
AUTH_TENANT_QUOTA_EXCEEDED
AUTH_GLOBAL_CAPACITY_EXCEEDED

AUTH_LOAD_SHED
AUTH_BROWNOUT_ACTIVE
AUTH_ADMISSION_DENIED

## 276. Error Normalization

No exponer:
KMS has 97% utilization
a cliente.

## 277. Audit

Eventos:
AuthenticationRateLimitTriggered
AuthenticationQuotaExceeded
AuthenticationCapacityConstrained
AuthenticationAdmissionRejected
AuthenticationAbuseDetected
AuthenticationCredentialStuffingDetected
AuthenticationPasswordSprayingDetected
AuthenticationOtpFloodDetected
AuthenticationSmsPumpingDetected
AuthenticationLoadSheddingActivated
AuthenticationBrownoutActivated
AuthenticationDependencyCircuitOpened
AuthenticationReservedCapacityUsed
AuthenticationTenantCapacityConstrained

## 278. Audit Volume

No registrar individualmente millones de bot requests si esto crea un segundo DoS.

## 279. Audit Aggregation

Puede utilizar:
count
window
scope fingerprint
attack type
severity

## 280. Security Event Sampling

Sampling puede aplicarse a repetitive low-value events.

## 281. Never Sample Away

critical incident transitions
configuration changes
manual overrides
break-glass usage

## 282. Metrics

auth_admission_total
auth_admission_rejected_total
auth_rate_limit_total
auth_quota_exceeded_total

auth_capacity_state
auth_capacity_reservation_total
auth_capacity_reservation_failed_total

auth_password_hash_active
auth_password_hash_wait_seconds
auth_password_hash_capacity_rejected_total

auth_crypto_active
auth_crypto_capacity_rejected_total

auth_challenge_active
auth_transaction_active
auth_session_creation_total

auth_abuse_detected_total
auth_brute_force_detected_total
auth_password_spraying_detected_total
auth_credential_stuffing_detected_total
auth_otp_flood_detected_total

auth_dependency_circuit_open
auth_dependency_rejection_total

auth_load_shed_total
auth_brownout_state

auth_tenant_capacity_rejected_total

## 283. Metric Cardinality

Labels controlados:
operation
resource
decision
abuse_type
capacity_state
dependency_class
realm_class

## 284. Prohibited Labels

email
username
identity_id
IP
session_id
tenant_id
transaction_id
credential_id
salvo sistemas especializados que explícitamente manejen cardinalidad fuera de metrics estándar.

## 285. Tracing

Spans:
auth.admission.evaluate
auth.rate_limit.evaluate
auth.quota.evaluate
auth.capacity.reserve
auth.abuse.evaluate
auth.password_hash.reserve
auth.crypto.reserve
auth.dependency.protect
auth.load_shed.evaluate

## 286. Trace Safety

No incluir raw credentials.

## 287. Performance

El propio sistema anti-abuse no puede ser más caro que la operación que protege.

## 288. Fast Path

Normal traffic debería recorrer:
compiled policies
fast counters
local snapshots
bounded distributed operations

## 289. Slow Abuse Analytics

Análisis complejo puede ejecutarse async.
Documento 44.

## 290. Immediate Defense

No esperar análisis async para aplicar basic rate/capacity controls.

## 291. Two-Tier Architecture

Fast Protection Plane
       │
       ├── admission
       ├── counters
       ├── capacity
       └── concurrency
       │
       ▼
Authentication

Async Intelligence Plane
       │
       ├── correlation
       ├── attack detection
       ├── anomaly analysis
       └── recommendations

## 292. Intelligence Feedback

Async signals pueden alimentar futuros decisions.

## 293. Eventual Abuse Detection

No modifica retroactivamente Authentication Context histórico.

## 294. Incident Escalation

Severe abuse puede activar documento 40.

## 295. Example

Credential stuffing detected
      ↓
Abuse Level = Severe
      ↓
Incident created
      ↓
Tenant protections hardened
      ↓
notifications
      ↓
possible forced reauthentication

## 296. Dynamic Security Overlay

Documento 36 puede recibir temporary policy overlay.

## 297. Example

Attack ongoing
→ require phishing-resistant MFA for admin realm
si policy lo determina.

## 298. Abuse Overlay != Arbitrary Rule

Debe ser:
versioned
audited
scoped
expiring
explainable

## 299. Emergency Controls

Security operators podrán activar:
tenant authentication restriction
provider disablement
network restriction
method restriction
capacity reserve
brownout

## 300. Privileged Administration

Cambios requieren:
Authorization
Fresh Authentication
Privileged Context
Audit
según documento 32.

## 301. Emergency Rate Override

Debe tener:
reason
actor
scope
expiry
maximum bounds

## 302. No Permanent Emergency Flag

Overrides expiran.

## 303. Configuration

Ejemplo conceptual:
return [

    'resource_governance' => [

        'admission' => [
            'enabled' => true,
        ],

        'password_hashing' => [
            'max_concurrent' => 32,
            'reserved_admin' => 4,
        ],

        'transactions' => [
            'max_active_per_identity' => 10,
            'max_active_per_network' => 100,
        ],

        'otp' => [
            'issue' => [
                'identity' => '3/10m',
                'destination' => '5/1h',
                'network' => '30/10m',
            ],
        ],

        'recovery' => [
            'identity' => '3/1h',
            'network' => '20/1h',
        ],

        'brownout' => [
            'enabled' => true,
        ],
    ],
];

## 304. Configuration Is Conceptual

Valores reales dependen de:
hardware
traffic
hash policy
tenant model
risk profile
infrastructure

## 305. No Universal Magic Numbers

VoltStack no deberá asumir que:
5 attempts
es universalmente correcto.

## 306. Policy Registry

interface AuthenticationResourceGovernancePolicyRegistryInterface
{
    public function effectivePolicy(
        AuthenticationResourceGovernanceContext $context
    ): EffectiveAuthenticationResourceGovernancePolicy;
}

## 307. Policy Hierarchy

Framework Safety Floor
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
Operation

## 308. Tenant Customization

Tenant podrá endurecer:
login attempts
recovery limits
session limits
OTP quotas

## 309. Tenant Cannot

Desactivar platform safety floors.

## 310. Compiled Governance

Static policies podrán compilarse.

## 311. Dynamic Inputs

current load
attack level
dependency health
queue depth
risk
se evalúan runtime.

## 312. Compiled Policy

final readonly class CompiledAuthenticationResourcePolicy
{
    public function __construct(
        public AuthenticationCompiledRateLimitSet $rateLimits,
        public AuthenticationCompiledQuotaSet $quotas,
        public AuthenticationCompiledCapacityRules $capacity,
        public AuthenticationCompiledDegradationRules $degradation,
    ) {}
}

## 313. FrankenPHP

Persistent workers aumentan la importancia del resource governance.

## 314. Worker Memory

Authentication request no debe dejar:
rate limit context
identity
tenant
capacity lease
abuse signals
en globals.

## 315. Reservation Cleanup

Toda reservation deberá liberarse en:
finally
o mediante lease expiration.

## 316. Worker Crash

Distributed reservation no deberá quedar eterna.

## 317. Per-Worker Capacity

Puede existir:
local concurrency budget
además del global.

## 318. Two-Level Capacity

Global Cluster Budget
       +
Local Worker Budget

## 319. Why

Evita que un node acepte más trabajo del que puede procesar aunque cluster tenga capacidad teórica.

## 320. FrankenPHP Request Concurrency

Debe gobernarse antes de ejecutar expensive authentication operations.

## 321. Fibers

Capacity leases no se almacenarán en static properties.

## 322. Context

final readonly class AuthenticationResourceExecutionContext
{
    public function __construct(
        public AuthenticationAdmissionDecision $admission,
        public AuthenticationResourceReservationSet $reservations,
    ) {}
}

## 323. Cancellation

Si request se cancela:
release reservations
cancel unnecessary downstream work
cuando sea seguro.

## 324. Client Disconnect

No necesariamente debe cancelar security mutation ya comprometida.

## 325. Example

Si password change ya fue committed:
browser disconnected
no revertirlo.

## 326. Testing — Rate Limit

Validar:
single node
multi-node
race conditions
window boundaries
TTL
clock skew

## 327. Testing — Botnet

Miles de IPs contra una identidad.
Identity-centric control responde.

## 328. Testing — Password Spray

Una IP contra miles de identities.
Network/account-spread detection responde.

## 329. Testing — NAT

Miles de legitimate users tras misma IP.
No producir lockout masivo injustificado.

## 330. Testing — Unknown Identities

Flood de random usernames no crea cardinalidad infinita.

## 331. Testing — Argon2 Exhaustion

Concurrency nunca supera configured memory-safe budget.

## 332. Testing — Capacity Lease Crash

Worker crash libera capacidad mediante TTL/fencing.

## 333. Testing — Redis Failure

Fallback policy aplicada.
Nunca unlimited expensive verification.

## 334. Testing — Database Saturation

Admission comienza a shed/defer antes de pool exhaustion total.

## 335. Testing — KMS Outage

No crypto bypass.

## 336. Testing — Unknown kid

10,000 unknown kid requests no producen 10,000 JWKS downloads.

## 337. Testing — SMS Pumping

Financial quota limita abuso.

## 338. Testing — Email Bombing

Recovery endpoint no genera unlimited mail.

## 339. Testing — OTP Guessing

Verification attempts bounded.

## 340. Testing — Challenge Flood

Active challenge limits respetados.

## 341. Testing — Transaction Flood

Anonymous transaction capacity bounded.

## 342. Testing — Session Flood

Identity session policy aplicada.

## 343. Testing — Tenant Isolation

Tenant atacante no agota Tenant B.

## 344. Testing — Global Attack

Platform capacity protection prevalece.

## 345. Testing — Reserved Capacity

Critical revocation sigue disponible durante overload.

## 346. Testing — Brownout

Solo optional functionality desaparece.

## 347. Testing — Security Requirement

Brownout nunca reduce MFA/assurance requirement.

## 348. Testing — Circuit Breaker

External dependency failure queda aislada.

## 349. Testing — Half Open

Solo bounded probes alcanzan dependency.

## 350. Testing — Multi-Region

Regional limits + global signals funcionan sin depender de una única hot key global para cada request.

## 351. Testing — Counter Failure

No produce privilege/security downgrade.

## 352. Testing — Enumeration

Rate-limit responses no hacen trivial descubrir accounts.

## 353. Testing — Privacy

Raw identifiers no aparecen en rate keys cuando fingerprinting protegido sea requerido.

## 354. Testing — Metrics Cardinality

Ataque con millones de usernames no crea millones de metric series.

## 355. Testing — Audit DoS

Ataque no llena audit synchronously hasta derribar sistema.

## 356. Testing — FrankenPHP

100,000 sequential requests no filtran:
tenant
capacity reservation
rate context
abuse assessment

## 357. Testing — Fibers

Concurrent Tenant A/B permanecen aislados.

## 358. Security Invariants — Admission

AUTH-CAP-ADM-01
Toda operación Authentication costosa podrá ser sometida a admission control.
AUTH-CAP-ADM-02
Admission no equivaldrá a Authentication.
AUTH-CAP-ADM-03
Capacity pressure nunca otorgará Authentication.
AUTH-CAP-ADM-04
Degraded mode nunca reducirá requisitos obligatorios de seguridad.
AUTH-CAP-ADM-05
Expensive work deberá ejecutarse únicamente después de controles baratos apropiados.

## 359. Security Invariants — Rate Limiting

AUTH-CAP-RATE-01
IP no será la única dimensión disponible.
AUTH-CAP-RATE-02
Distributed attacks deberán poder detectarse mediante otras dimensiones.
AUTH-CAP-RATE-03
Rate-limit counters distribuidos serán atómicos.
AUTH-CAP-RATE-04
Limiter failure no producirá capacidad ilimitada.
AUTH-CAP-RATE-05
Permanent victim lockout no será la defensa por defecto.
AUTH-CAP-RATE-06
Responses evitarán account enumeration innecesaria.

## 360. Security Invariants — Resources

AUTH-CAP-RES-01
Password hashing tendrá concurrency governance.
AUTH-CAP-RES-02
Load no reducirá automáticamente password hashing security parameters.
AUTH-CAP-RES-03
Cryptographic capacity será explícitamente gobernable.
AUTH-CAP-RES-04
External providers tendrán bulkheads/capacity policies.
AUTH-CAP-RES-05
Resource reservations tendrán release/expiry semantics.
AUTH-CAP-RES-06
Worker crash no mantendrá reservation infinita.

## 361. Security Invariants — OTP

AUTH-CAP-OTP-01
OTP issuance tendrá rate limits.
AUTH-CAP-OTP-02
OTP verification tendrá límites independientes.
AUTH-CAP-OTP-03
SMS/email cost abuse estará gobernado.
AUTH-CAP-OTP-04
Destination identifiers podrán protegerse mediante fingerprints.
AUTH-CAP-OTP-05
OTP flood no generará communication amplification ilimitada.

## 362. Security Invariants — Federation

AUTH-CAP-FED-01
Unknown kid no provocará remote fetch ilimitado.
AUTH-CAP-FED-02
JWK refresh estará single-flight/rate-limited.
AUTH-CAP-FED-03
Token input no controlará arbitrariamente key URLs.
AUTH-CAP-FED-04
Provider outage no provocará security downgrade.
AUTH-CAP-FED-05
Callbacks malformados serán rechazados antes de operaciones costosas cuando sea posible.

## 363. Security Invariants — Multi-Tenant

AUTH-CAP-TENANT-01
Un tenant no deberá monopolizar capacidad compartida.
AUTH-CAP-TENANT-02
Tenant budgets estarán scope-bound.
AUTH-CAP-TENANT-03
Tenant custom policy no podrá debilitar platform safety floors.
AUTH-CAP-TENANT-04
Capacity borrowing no consumirá reservas críticas protegidas.
AUTH-CAP-TENANT-05
Rate-limit state no cruzará tenant boundaries incorrectamente.

## 364. Security Invariants — Availability

AUTH-CAP-AVL-01
Critical security controls tendrán capacidad protegida cuando la arquitectura lo requiera.
AUTH-CAP-AVL-02
Attack traffic no deberá bloquear revocation/incident response por starvation evitable.
AUTH-CAP-AVL-03
Load shedding tendrá orden explícito.
AUTH-CAP-AVL-04
Optional work será eliminado antes que security-critical work.
AUTH-CAP-AVL-05
Brownout nunca será equivalente a “security off”.

## 365. Security Invariants — Privacy

AUTH-CAP-PRIV-01
Anti-abuse metadata estará sujeto al documento 42.
AUTH-CAP-PRIV-02
Rate-limit storage minimizará PII.
AUTH-CAP-PRIV-03
Metric labels no incluirán identifiers de cardinalidad no controlada.
AUTH-CAP-PRIV-04
Abuse prevention no se convertirá implícitamente en tracking permanente.

## 366. Anti-Pattern

if ($serverLoad > 90) {
    $requireMfa = false;
}

## 367. Anti-Pattern

if ($serverLoad > 90) {
    $argonMemory = 1024;
}

## 368. Anti-Pattern

Redis unavailable
→ rate limiting disabled

## 369. Anti-Pattern

5 failed attempts
→ permanent account lock

## 370. Anti-Pattern

rate limit only by IP

## 371. Anti-Pattern

unknown username
→ create permanent Redis key

## 372. Anti-Pattern

unknown kid
→ fetch JWK immediately

## 373. Anti-Pattern

CAPTCHA solved
→ assurance = HIGH

## 374. Anti-Pattern

SMS provider unavailable
→ skip MFA

## 375. Anti-Pattern

one resource pool
→ login + bulk emails + telemetry + revocations

## 376. Anti-Pattern

tenant premium plan
→ unlimited Authentication

## 377. Anti-Pattern

sleep(60)
para throttle dentro de persistent HTTP worker.

## 378. Anti-Pattern

metrics label = username

## 379. Anti-Pattern

attack detected
→ log every request synchronously
hasta agotar disco.

## 380. Anti-Pattern

brownout
→ accept unsigned token

## 381. Anti-Pattern

DB replica says account exists
→ perform critical decision without authoritative revalidation

## 382. Namespace

Namespace recomendado:
VoltStack\Quantum\Auth\Governance
Subnamespace:
VoltStack\Quantum\Auth\Governance\Resource

## 383. Estructura sugerida

src/Quantum/Auth/Governance/
├── Admission/
│   ├── AuthenticationAdmissionController.php
│   ├── AuthenticationAdmissionRequest.php
│   ├── AuthenticationAdmissionDecision.php
│   └── AuthenticationAdmissionDecisionType.php
│
├── RateLimit/
│   ├── AuthenticationRateLimiter.php
│   ├── AuthenticationRateLimitKey.php
│   ├── AuthenticationRateLimitRequest.php
│   ├── AuthenticationRateLimitDecision.php
│   ├── AuthenticationRateLimitDimension.php
│   ├── AuthenticationLimiterFailureMode.php
│   └── AuthenticationIdentifierFingerprinter.php
│
├── Quota/
│   ├── AuthenticationQuotaManager.php
│   ├── AuthenticationQuotaRequest.php
│   ├── AuthenticationQuotaDecision.php
│   └── AuthenticationFinancialCostBudget.php
│
├── Capacity/
│   ├── AuthenticationCapacityManager.php
│   ├── AuthenticationCapacitySnapshot.php
│   ├── AuthenticationCapacityState.php
│   ├── AuthenticationCapacityAllocation.php
│   └── AuthenticationReservedCapacity.php
│
├── Resource/
│   ├── AuthenticationResourceGovernor.php
│   ├── AuthenticationResourceBudget.php
│   ├── AuthenticationResourceLimits.php
│   ├── AuthenticationResourceScope.php
│   ├── AuthenticationResourceReservation.php
│   └── AuthenticationResourceExecutionContext.php
│
├── Cost/
│   ├── AuthenticationCostEstimator.php
│   ├── AuthenticationCostEstimate.php
│   └── AuthenticationCostContext.php
│
├── Password/
│   ├── AuthenticationPasswordHashCapacityGovernor.php
│   └── PasswordHashCapacityLease.php
│
├── Crypto/
│   ├── AuthenticationCryptoCapacityGovernor.php
│   └── AuthenticationCryptoCapacityReservation.php
│
├── Abuse/
│   ├── AuthenticationAbuseDetector.php
│   ├── AuthenticationAbuseAssessment.php
│   ├── AuthenticationAbuseSignal.php
│   ├── AuthenticationAbuseLevel.php
│   └── AuthenticationAbuseSignalType.php
│
├── Dependency/
│   ├── AuthenticationDependencyDefinition.php
│   ├── AuthenticationDependencyRequirement.php
│   ├── AuthenticationDependencyCircuitBreaker.php
│   ├── AuthenticationDependencyBulkhead.php
│   └── AuthenticationDependencyHealth.php
│
├── Fairness/
│   ├── AuthenticationFairnessStrategy.php
│   ├── AuthenticationTenantResourceBudget.php
│   └── AuthenticationCapacityAllocation.php
│
├── Degradation/
│   ├── AuthenticationLoadShedder.php
│   ├── AuthenticationBrownoutManager.php
│   ├── AuthenticationDegradationPolicy.php
│   └── AuthenticationLoadSheddingDecision.php
│
├── Policy/
│   ├── AuthenticationResourceGovernancePolicyRegistry.php
│   ├── EffectiveAuthenticationResourceGovernancePolicy.php
│   ├── CompiledAuthenticationResourcePolicy.php
│   └── AuthenticationGovernancePolicyCompiler.php
│
├── Runtime/
│   ├── AuthenticationResourceRuntime.php
│   ├── AuthenticationResourceRuntimeResetter.php
│   └── AuthenticationLocalEmergencyLimiter.php
│
├── Events/
├── Metrics/
├── Exceptions/
└── Testing/

## 384. Integration Architecture

                     HTTP / SPA / API / CLI
                              │
                              ▼
                    HTTP Resource Limits
                              │
                              ▼
                 Authentication Entry Point
                              │
                              ▼
                ┌─────────────────────────┐
                │ Admission Controller    │
                └────────────┬────────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
       Rate Limits       Abuse Engine     Capacity
            │                │                │
            └────────────────┼────────────────┘
                             ▼
                     Resource Governor
                             │
                             ▼
                    Authentication Flow
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          Password        WebAuthn      Federation
          Capacity         Crypto         Provider
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                   Authentication Result
                             │
                             ▼
                 Background Processing 44

## 385. Control Plane vs Data Plane

VoltStack deberá distinguir:
Authentication Data Plane
que procesa Authentication real,
de:
Authentication Governance Control Plane
que administra:
limits
budgets
capacity policies
emergency modes
dependency state
abuse overlays

## 386. Control Plane Security

Cambiar:
password hash concurrency
OTP limits
reserved capacity
brownout rules
provider limits
es operación administrativa sensible.

## 387. Audit Configuration Changes

Toda modificación importante deberá registrar:
actor
old value
new value
scope
reason
timestamp
policy version

## 388. Configuration Validation

VoltStack deberá detectar configuraciones peligrosas.
Ejemplo:
password_hash.max_concurrent = 10000
con hardware insuficiente.

## 389. Resource Safety Validator

interface AuthenticationResourceSafetyValidatorInterface
{
    public function validate(
        AuthenticationResourceGovernanceConfiguration $configuration,
        AuthenticationRuntimeResourceProfile $runtime
    ): AuthenticationResourceSafetyReport;
}

## 390. Startup Validation

Puede advertir/fallar ante:
impossible memory budget
zero critical reserve
invalid limiter backend
unsafe fail-open policy
unbounded OTP quota
unbounded transaction capacity
según environment.

## 391. Hardware-Aware Defaults

VoltStack podrá calcular defaults conservadores usando:
available memory
CPU cores
worker count
Argon2 parameters
DB pool
Redis pool

## 392. Explicit Override

Production puede requerir confirmación explícita para valores fuera de safety bounds.

## 393. Capacity Planning

El sistema deberá producir información para estimar:
maximum password verifies/sec
maximum concurrent logins
OTP provider requirements
KMS capacity
Redis operations
DB pool demand

## 394. Benchmark Integration

VoltStack Testing/CLI podría incluir:
auth:capacity:benchmark
para entorno controlado.

## 395. Benchmark Safety

Nunca ejecutar benchmark pesado accidentalmente en production.

## 396. Example Output

Password Algorithm: Argon2id
Memory / Verification: 64 MiB
Safe Concurrent Verifications: 24
Measured P95: 185 ms
Recommended Worker Reserve: 4
Recommended Global Admission Limit: 20
conceptualmente.

## 397. Capacity Profiles

development
testing
small-production
production
high-throughput
custom
pueden existir como starting profiles, no como security truth universal.

## 398. Laravel Comparison

Laravel proporciona excelentes primitives:
RateLimiter
Cache
Queues
Middleware
Horizon
Session
Events
y permite implementar throttling de login fácilmente.
Pero la aplicación normalmente debe diseñar por separado:
password hashing capacity
resource budgets
multi-dimensional abuse detection
KMS/HSM capacity
OTP financial abuse
multi-tenant fairness
reserved incident-response capacity
brownout
dependency bulkheads
Authentication-specific load shedding
distributed attack correlation
VoltStack incorpora estos conceptos directamente al subsistema Authentication.

## 399. Symfony Comparison

Symfony dispone de:
RateLimiter
Lock
Messenger
Cache
Security
HttpKernel
y buenas primitives para infraestructura resiliente.
VoltStack añade una capa semántica específica para Authentication:
Authentication Admission
Authentication Cost Model
Authentication Resource Budget
Password Hash Capacity
Challenge/Transaction Capacity
Abuse Classification
OTP Cost Governance
Tenant Fairness
Security Reserved Capacity
Authentication Brownout
Security-Safe Dependency Degradation

## 400. Diferenciador VoltStack

Laravel-like Rate Limiting DX
+
Symfony-like Infrastructure Contracts
+
Authentication-Specific Admission Control
+
Multi-Dimensional Rate Limits
+
Hierarchical Resource Budgets
+
Password Hashing Capacity Governance
+
Cryptographic Capacity Governance
+
OTP/SMS Cost Protection
+
Challenge/Transaction Capacity
+
Credential Stuffing Detection
+
Password Spraying Detection
+
Bot/Abuse Resistance
+
Dependency Bulkheads
+
Circuit Breakers
+
Multi-Tenant Fairness
+
Noisy-Neighbor Isolation
+
Reserved Security Capacity
+
Security-Safe Load Shedding
+
Authentication Brownout
+
Distributed Capacity Control
+
FrankenPHP-Safe Runtime

## 401. Arquitectura final del sistema

                        Internet
                           │
                           ▼
                 Edge / Proxy Controls
                           │
                           ▼
                    HTTP Limits
                           │
                           ▼
             Authentication Admission
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
      Rate/Quota        Abuse            Capacity
          │             Detection           │
          └────────────────┼─────────────────┘
                           ▼
                  Cost / Budget Engine
                           │
                           ▼
                  Resource Reservation
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
     CPU / Memory       Crypto/KMS      DB / Redis
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                 Authentication Runtime
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
             SUCCESS               FAILURE
                 │
                 ▼
            Async System 44

## 402. Relación 19, 20, 36, 38, 44, 45

La separación definitiva será:
19
│
├── How frequently may Authentication attempts occur?
│
20
│
├── How risky is this Authentication?
│
36
│
├── What Authentication is required?
│
38
│
├── How do we obtain the missing evidence?
│
44
│
├── What security work can run asynchronously?
│
45
│
└── Can the Authentication infrastructure safely
    afford and admit this operation now?

## 403. Regla de prioridad

Cuando disponibilidad y Authentication requirement entren en tensión:
Security Requirement
        >
Convenience
El sistema deberá:
reject
delay
defer
use equivalent secure alternative
antes que reducir silenciosamente la seguridad.

## 404. Regla de capacidad

VoltStack deberá limitar la cantidad de trabajo que acepta antes de permitir que una cantidad ilimitada de trabajo degrade todo el sistema.

  1. Regla de recursos costosos
Las operaciones diseñadas deliberadamente para ser costosas —como password hashing— deberán estar protegidas por límites de concurrencia y capacidad, no debilitadas criptográficamente durante períodos de carga.

  2. Regla de dependencias
La falla de una dependencia de Authentication debe degradar únicamente las capacidades cuya semántica permita degradación; nunca deberá convertir una verificación obligatoria en opcional.

  3. Regla multi-tenant
Un tenant puede consumir su capacidad, pero no la capacidad de seguridad de toda la plataforma.

  4. Regla anti-abuso
Rate limiting no debe entenderse únicamente como “contar IPs”; VoltStack deberá correlacionar identidad, red, tenant, dispositivo, operación, comportamiento, costo y capacidad para resistir ataques distribuidos.

  5. Regla de disponibilidad defensiva
El propio mecanismo utilizado para defender Authentication no deberá convertirse fácilmente en un nuevo vector de DoS.

Esto aplica especialmente a:
password dummy hashing
CAPTCHA
audit
metrics
rate-limit cardinality
JWK refresh
notifications
incident generation

## 410. Principio arquitectónico final

VoltStack deberá considerar Authentication como un sistema con recursos finitos.
Por tanto:
Authentication Security
=

Identity Verification
+
Credential Security
+
Policy Enforcement
+
Integrity
+
Availability
+
Capacity Governance
+
Abuse Resistance
El modelo final será:
Untrusted Demand
      │
      ▼
Measure
      │
      ▼
Classify
      │
      ▼
Limit
      │
      ▼
Reserve
      │
      ▼
Authenticate
      │
      ▼
Account
      │
      ▼
Adapt
De esta forma, un atacante no deberá poder transformar fácilmente:
cheap malicious traffic
en:
unbounded expensive Authentication work
y, aun bajo ataque, VoltStack deberá intentar preservar primero la capacidad necesaria para:
legitimate authentication
strong authentication
session revocation
credential revocation
incident response
administrative security
break-glass recovery
antes que funcionalidades secundarias.

## 411. Siguiente documento

El siguiente documento recomendado es:
46_AUTHENTICATION_MIGRATION_LEGACY_CREDENTIAL_IMPORT_BACKWARD_COMPATIBILITY_AND_PROGRESSIVE_SECURITY_UPGRADE_SYSTEM.md
Este sistema cerrará uno de los problemas más importantes para la adopción real de VoltStack: permitir que una aplicación existente migre desde Laravel, Symfony, sistemas propios o plataformas legacy sin obligar a resetear inmediatamente todas las credenciales.
Deberá cubrir:
Legacy Identity Import
Legacy Password Hash Recognition
Password Hash Progressive Upgrade
Credential Import
Session Migration Boundaries
Remember-Me Migration
MFA Migration
TOTP Secret Migration
Passkey/WebAuthn Import Constraints
Recovery Credential Migration
OAuth/OIDC Identity Migration
Legacy Provider Mapping
API Credential Migration
Machine Credential Migration
Credential Provenance
Migration Trust Levels
Compatibility Authentication
Dual Authentication Periods
Progressive Cutover
Security Upgrade on Successful Login
Legacy Algorithm Deprecation
Forced Migration Deadlines
Migration Security Epochs
Credential Rebinding
Identity Deduplication
Migration Conflict Resolution
Rollback Boundaries
Migration Audit
Migration Metrics
Bulk Import
Streaming Import
Dry-Run Validation
Migration Checkpoints
Idempotent Migration
Distributed Migration
Multi-Tenant Migration
Zero-Downtime Migration
Legacy System Retirement
FrankenPHP-Safe Migration Runtime
La idea central deberá ser:
Legacy Authentication
        │
        ▼
Compatibility Boundary
        │
        ▼
Verify Existing Credential
        │
        ▼
Create VoltStack Evidence
        │
        ▼
Apply Current VoltStack Policy
        │
        ▼
Progressively Upgrade Credential
        │
        ▼
Native VoltStack Authentication
sin permitir que la compatibilidad legacy se convierta en una excepción permanente a las reglas modernas de seguridad.
