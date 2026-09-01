# VoltStack Authentication System

## 26 — Authentication Testing, Verification, Security Assurance and Conformance System

- **Archivo:** `26_AUTHENTICATION_TESTING_VERIFICATION_SECURITY_ASSURANCE_AND_CONFORMANCE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth` + Testing
- **Estado:** Especificación arquitectónica del subsistema de pruebas, verificación, assurance de seguridad, conformidad y regresión del sistema Authentication.

---

## 1. Propósito

Este documento define cómo VoltStack verificará que el sistema Authentication:

- funciona correctamente
- cumple sus contratos

mantiene invariantes de seguridad
resiste ataques conocidos
no introduce regresiones
es consistente entre runtimes
es seguro bajo concurrencia
es compatible con multi-tenancy
cumple protocolos externos
La intención no será limitarse a:

- "login works"
- "wrong password fails"

sino construir una infraestructura capaz de demostrar que las garantías arquitectónicas definidas entre los documentos 00 y 25 siguen siendo ciertas a medida que evoluciona el framework.

## 2. Objetivo principal

El sistema deberá permitir verificar automáticamente:

- Authentication Architecture
- Authenticator Contracts
- Identity Resolution
- Credential Verification
- Password Security
- Session Security
- Remember-Me
- Bearer/API Tokens
- MFA
- Passkeys / WebAuthn
- OAuth2 / OIDC
- Account Recovery
- Abuse Protection
- Risk Engine
- Device Trust
- Authentication Flows
- Events / Hooks
- Observability
- Failure Handling
- Multi-Tenant Isolation
- Concurrency
- FrankenPHP Worker Reuse
- Security Invariants
- Protocol Conformance

## 3. Principio fundamental

Una propiedad de seguridad que no puede verificarse repetidamente terminará convirtiéndose en una suposición.

Por ello las invariantes:
AUTH-...
definidas en documentos anteriores deberán poder mapearse a pruebas automatizadas.

## 4. Segunda regla fundamental

Testing de Authentication no deberá comprobar solamente resultados funcionales; deberá verificar propiedades negativas, aislamiento, ausencia de leakage y resistencia a estados inesperados.

## 1. Tercera regla fundamental

Los tests de Authentication deberán probar tanto el happy path como las condiciones que el framework nunca debe permitir.

## 2. Arquitectura general

Authentication Specifications
│
▼
Security Invariants Registry
│
▼
Authentication Test Harness
│
┌──────┼──────────────────────────────────┐
▼      ▼           ▼          ▼           ▼
Unit  Contract   Integration  Security   Protocol
Tests  Tests       Tests       Tests     Conformance
│      │           │          │           │
└──────┴───────────┴──────────┴───────────┘
│
▼
Verification Report
│
┌─────┼─────┐
▼     ▼     ▼
PASS   FAIL  WARNING

## 3. Testing Layers

VoltStack distinguirá:

- Unit Tests
- Contract Tests
- Component Integration Tests
- Authentication Flow Tests
- Protocol Conformance Tests
- Security Property Tests
- Attack Simulation Tests
- Concurrency Tests
- Runtime Isolation Tests
- Fault Injection Tests
- Regression Tests
- Performance Security Tests

## 4. Unit Testing

Verifica componentes individuales.

- Ejemplo:
- PasswordPolicy
- RiskRule
- RateLimitKey
- SessionExpirationPolicy
- DeviceTrustPolicy

## 5. Contract Testing

Verifica que una implementación cumpla un contrato del framework.
Ejemplo:

- CustomAuthenticator
- CustomIdentityProvider
- CustomRiskProvider
- CustomRateLimitStore

## 6. Integration Testing

Verifica cooperación entre subsistemas.

```text
Ejemplo:
Password
    ↓
Risk
    ↓
MFA
    ↓
Session
```

## 7. Flow Testing

Verifica Authentication completa a través de múltiples pasos.

## 8. Protocol Conformance

Valida implementaciones contra protocolos externos como:

- OAuth2
- OpenID Connect
- WebAuthn

FIDO2 profiles where applicable
Bearer authentication semantics

## 9. Security Property Testing

Busca demostrar invariantes como:

- invalid credential never authenticates
- revoked session never restores

tenant A state never appears in tenant B

## 10. Attack Simulation

Simula:

- brute force
- credential stuffing
- password spraying
- OTP guessing
- replay
- flow manipulation
- session fixation

## 11. Fault Injection

Simula fallos técnicos:

- Redis unavailable
- Database timeout
- OIDC provider timeout
- Risk provider failure
- Session store failure

## 12. AuthenticationTestHarness

Será la infraestructura central.

```php
interface AuthenticationTestHarnessInterface
{
    public function scenario(
        AuthenticationTestScenario $scenario
    ): AuthenticationTestExecution;
}
```

## 13. AuthenticationTestScenario

Conceptualmente:

```php
final readonly class AuthenticationTestScenario
{
    public function __construct(
        public TestIdentitySet $identities,
        public TestCredentialSet $credentials,
        public AuthenticationTestEnvironment $environment,
        public AuthenticationTestSteps $steps,
        public AuthenticationExpectationSet $expectations,
    ) {}
}
```

## 14. Scenario DSL

VoltStack podrá ofrecer una API expresiva.

```php
AuthTest::scenario('password + mfa')
    ->identity('alice')
    ->withPassword('correct-password')
    ->requireMfa()
    ->submitTotp('valid')
    ->expectAuthenticated();
```

## 15. Secrets en Testing

Los fixtures de pruebas podrán contener secretos falsos.

- Pero tooling deberá evitar que terminen accidentalmente en:
- snapshots
- failure reports
- CI logs
- traces

## 16. Authentication Test Environment

Deberá poder controlar:

- Clock
- Randomness
- Network
- Storage
- Session Store
- Rate Limit Store
- Risk Providers
- Device Context
- Tenant Context
- OIDC Providers

## 17. Deterministic Clock

Authentication depende fuertemente del tiempo.
VoltStack deberá utilizar:
ClockInterface
para poder probar:

- expiration
- freshness
- cooldowns
- token TTL
- session timeout
- risk decay

## 18. Frozen Clock

Ejemplo:
$clock->freeze('2026-08-23 10:00:00');

## 19. Time Travel

$clock->advance(minutes: 10);
deberá ser posible en tests.

## 20. Deterministic Randomness

Algunos componentes podrán aceptar:

- RandomSourceInterface
- en tests.

## 21. No weakening production crypto

Nunca permitir que deterministic test randomness pueda activarse accidentalmente en producción.

## 22. Test Service Provider

Podrá registrar:

- FakeClock
- FakeMailer
- FakeRiskProvider
- FakeIdentityProvider
- InMemorySessionStore
- InMemoryRateLimitStore

solo en testing environment.

## 23. Authentication Test Identity

Podrá crearse mediante builder:

```php
$alice = AuthTestIdentity::make()
    ->active()
    ->withPassword()
    ->withTotp()
    ->withPasskey();
```

## 24. Identity States

Fixtures deberán soportar:

- ACTIVE
- DISABLED
- SUSPENDED
- LOCKED
- EXPIRED
- RECOVERY_PENDING
- SECURITY_HOLD

## 25. Credential Fixtures

Deberán permitir:

- valid
- expired
- revoked
- compromised
- rotated
- malformed

## 26. Authenticator Contract Tests

Todo Authenticator deberá pasar un suite común.

## 27. AuthenticatorContractTestSuite

Verificará:

- supports correct credential type
- rejects unsupported credentials
- returns normalized outcome

never leaks raw credential
respects cancellation
does not mutate unrelated state

## 28. Contract ejemplo

abstract class AuthenticatorContractTestCase
{
abstract protected function authenticator(): AuthenticatorInterface;

final public function test_invalid_credentials_do_not_authenticate(): void
{
// ...

```php
    }
}
```

## 29. Custom Authenticators

Packages externos podrán ejecutar el mismo suite para certificar compatibilidad.

## 30. Identity Provider Contract Tests

Verificarán:

- identity lookup
- canonicalization
- tenant scope
- unknown identity behavior
- error normalization

## 31. Multi-Tenant Provider Test

Obligatorio:

- Identity A in Tenant A
- cannot be resolved from Tenant B

## 32. Password Testing

Deberá incluir:

- correct password
- incorrect password
- unknown identity
- rehash required
- expired password
- compromised password
- disabled identity

## 33. Password Hash Tests

Verificar:

- current algorithm
- policy parameters
- rehash detection
- constant-time verification primitive
- dummy verification path

## 34. Password Rehash

Caso:

```text
old valid hash
    ↓
successful authentication
    ↓
rehash scheduled/performed
```

sin cambiar resultado de Authentication.

## 35. Invalid Password

Nunca deberá provocar:

- session creation
- Remember-Me issuance
- trusted device enrollment

## 36. Unknown Identity Timing Path

Verificar que el path utilice dummy verification cuando policy lo exija.

## 37. Secret Leakage Test

Password deberá estar ausente de:

- logs
- events
- exceptions
- traces
- audit
- response

## 38. Session Testing

Deberá cubrir:

- creation
- rotation
- restoration
- expiration
- idle timeout
- absolute timeout
- revocation
- security version mismatch

## 39. Session Fixation Test

Escenario:

- anonymous session ID = X
- login

authenticated session ID must != X

## 40. Session Restore after Revocation

Debe fallar.

## 41. Session Security Version

session security version = 10
identity security version = 11
Resultado:
session invalid

## 42. Concurrent Session Tests

Ejemplo:

- session revoked on Node A
- request reaches Node B

Debe respetar la revocación según consistencia configurada.

## 43. Remember-Me Testing

Casos:

- valid credential
- expired credential
- revoked credential
- rotation
- replay
- Identity disabled
- risk elevation

## 44. Replay Test

Remember-Me generation 1
↓
rotated to generation 2
↓
generation 1 used again
debe producir señal/reacción definida.

## 45. Token Authentication Testing

Debe cubrir:

- valid bearer
- invalid bearer
- expired bearer
- revoked bearer
- wrong audience
- wrong issuer
- wrong tenant
- wrong scope

## 46. Token Parsing Fuzzing

Entradas:

- empty
- huge
- malformed
- invalid encoding
- duplicate scheme
- unexpected Unicode

no deberán provocar crashes.

## 47. Token Constant Behavior

Unknown token lookup no debe provocar consultas no acotadas.

## 48. MFA Testing

Deberá cubrir:

- factor enrollment
- factor verification
- invalid OTP
- expired OTP
- challenge exhaustion
- factor revocation
- backup code use
- step-up

## 49. OTP Attempt Limits

Verificar:
attempts <= policy
y después:
CHALLENGE_EXHAUSTED

## 50. New Challenge Bypass

Crear nuevo challenge no deberá resetear límites globales abusivamente.

## 51. Backup Codes

Cada código deberá ser:

- single-use
- revocable

## 52. MFA Fatigue Tests

Simular múltiples push prompts.

- Debe producir:
- throttling
- signal
- challenge restriction
- según policy.

## 53. Step-Up Tests

Validar:

```text
AAL1
    ↓
route requires AAL2
    ↓
Step-Up
    ↓
session assurance updated
```

## 54. Freshness Tests

Si requisito:

- fresh <= 5 min
- una autenticación de 10 minutos deberá requerir reauthentication.

## 55. Passkey / WebAuthn Tests

Deberán cubrir:

- registration
- authentication
- challenge generation
- challenge expiration
- challenge replay
- origin validation
- RP ID validation
- credential revocation

## 56. WebAuthn Conformance

Vectores oficiales/estándar compatibles deberán utilizarse cuando sea posible.

## 57. Malformed WebAuthn Payload

Fuzz testing especialmente importante.

## 58. Challenge Replay

Una assertion contra challenge ya consumido deberá fallar.

## 59. Wrong Origin

Debe fallar siempre.

## 60. Wrong RP ID

Debe fallar.

## 61. User Verification Requirements

Tests para:

- preferred
- required
- discouraged
- según policy.

## 62. Passkey Counter Anomaly

Debe producir el resultado/security signal definido.

## 63. Synced Passkeys

Tests no deberán asumir:
one passkey == one physical device

## 64. OAuth2 / OIDC Tests

Deberán cubrir:

- state
- nonce
- PKCE
- issuer
- audience
- signature
- expiration
- authorized party
- callback replay

## 65. State Test

Callback con state diferente:
reject

## 66. Nonce Test

OIDC ID Token con nonce incorrecto:
reject

## 67. PKCE

Verificar challenge/verifier.

## 68. Provider Substitution

Flow iniciado con Provider A no debe poder completarse mediante Provider B.

## 69. Issuer Confusion

Token de issuer no esperado debe rechazarse.

## 70. Audience Confusion

Token válido para otra aplicación:
reject

## 71. Federated Mapping Tests

known external identity
unknown external identity
mapping collision
tenant mismatch

## 72. External Provider Failure

Simular:

- timeout
- HTTP 500
- invalid metadata
- JWKS stale

## 73. No Silent Downgrade

Si federated assurance requerido no puede verificarse:
local session not established

## 74. Recovery Testing

Deberá cubrir:

- recovery initiation
- token validation
- OTP validation
- expiry
- replay
- rate limiting
- identity enumeration
- credential re-establishment

## 75. Recovery Enumeration

Known y unknown Identity deberán producir comportamiento público compatible con policy.

## 76. Recovery Email Flood

Simular múltiples requests a una misma Identity.

## 77. Recovery Token Replay

Después de consumo:
second use rejected

## 78. Recovery Finalization

Deberá verificar:

- old sessions revoked if policy
- security version incremented if required
- recovery state cleared

## 79. Abuse Protection Testing

Debe incluir:

- brute force
- credential stuffing
- password spraying
- OTP guessing
- token guessing
- recovery flooding

## 80. Brute Force Scenario

same Identity
same source
many password failures

## 81. Credential Stuffing Scenario

many identities
same/few sources
moderate failures each

## 82. Password Spraying Scenario

many identities
low rate
long window

## 83. Distributed Attack

Usar múltiples fake source IPs.

## 84. IP-only Protection Test

Demostrar que un ataque distribuido no evade toda protección.

## 85. Account Lockout DoS Test

Simular ataques contra una víctima.
Debe comprobar:

- attacker cannot permanently disable legitimate Identity
- por intentos inválidos normales.

## 86. Rate Limiter Contract Tests

Todo store/algorithm deberá verificar:

- limit
- TTL
- window
- refill
- atomicity
- concurrency

## 87. Redis Atomicity Test

Múltiples concurrent consumers no deberán superar el límite por race.

## 88. Counter Cardinality Test

Generar:

- 100k random identity claims
- y verificar límites de memoria/storage.

## 89. Risk Engine Testing

Deberá cubrir:

- risk scoring
- rules
- correlation
- signal confidence
- signal decay
- provider failure
- adaptive decisions

## 90. New Device

Debe elevar riesgo según policy, no automáticamente denegar.

## 91. Impossible Travel

Casos:

- real impossible travel
- VPN
- same device
- low-confidence geolocation

## 92. Signal Deduplication

Mismo hecho recibido de dos providers no deberá duplicar score de forma incorrecta.

## 93. Hard Rule Override

Ejemplo:

- credential compromised
- debe producir acción definida aunque score agregado sea bajo.

## 94. Risk Decay

Avanzar clock y verificar reducción/expiración.

## 95. Provider Timeout

Debe cumplir failure mode configurado.

## 96. Risk/Assurance Separation

Debe probarse explícitamente:

- Assurance HIGH
- Risk HIGH
- es estado válido.

## 97. Device Trust Testing

Deberá cubrir:

- unknown
- known
- trusted
- managed
- lost
- revoked
- compromised

## 98. Known != Trusted

Test obligatorio.

## 99. Shared Device

Alice trusts browser
Bob uses same browser
Bob no hereda trust.

## 100. Tenant-scoped Trust

Device trusted en Tenant A no debe ser trusted automáticamente en Tenant B.

## 101. Trusted Device MFA Test

Trusted device puede reducir un challenge normal según policy.
Pero una operación sensible deberá seguir pudiendo requerir fresh factor.

## 102. Device Credential Replay

Debe producir:

- signal
- risk increase
- revocation
- según policy.

## 103. Device Cardinality

Anonymous requests no crean registros ilimitados.

## 104. Authentication Flow Testing

Los flows deberán modelarse como state machines.

## 105. Basic Flow

STARTED
→ EVIDENCE
→ POLICY
→ FINALIZATION
→ COMPLETED

## 106. Challenge Flow

STARTED
→ EVIDENCE
→ CHALLENGE
→ CONTINUATION
→ FINALIZATION

## 107. Expired Flow

Una continuación posterior a expiry debe fallar.

## 108. Consumed Flow

Una continuación posterior a completion debe fallar.

## 109. Flow Security Version Change

Password verified
waiting MFA
Identity disabled
submit MFA
Finalization debe rechazar.

## 110. Cross-Tenant Flow

Flow Tenant A no puede continuar en Tenant B.

## 111. Cross-Realm Flow

Consumer flow no debe satisfacer admin realm sin requirements adicionales.

## 112. Challenge Swapping

Challenge de Flow A no puede utilizarse en Flow B.

## 113. Concurrent Finalization

Dos requests intentan completar el mismo flow.
Solo una finalización válida.

## 114. Idempotency

Retry del mismo network request no deberá crear múltiples sesiones accidentales.

## 115. Failure Handling Tests

Deben verificar:

- FAILURE
- DENIAL
- CHALLENGE
- ERROR
- por separado.

## 116. Invalid Password

FAILURE

## 117. Disabled Identity

DENIAL
internamente.

## 118. MFA Required

CHALLENGE

## 119. Redis Unavailable

ERROR
o degraded result explícito.

## 120. Public Error Mapping

Internamente diferentes razones pueden mapearse a un mismo public code.

## 121. Vendor Exception Isolation

Simular excepciones de drivers.
Nunca deben llegar directamente al response.

## 122. Retry Tests

Verificar que:

- invalid password
- no se reintenta.

Mientras:

- temporary provider timeout
- puede reintentarse si policy lo permite.

## 123. Downgrade Resistance Tests

Críticos.

## 124. Scenario

Policy requires Passkey
Passkey provider fails
Password succeeds
Resultado obligatorio:
not authenticated

## 125. Another Scenario

AAL2 required
only AAL1 available
No downgrade.

## 126. Enumeration Testing

Deberá cubrir:

- response body
- status
- headers
- response size
- timing distribution
- challenge behavior

## 127. Timing Tests

No comparar milisegundos exactos.
Usar distribuciones/umbrales amplios.

## 128. Statistical Timing Harness

Opcionalmente:

- N repeated attempts
- compare distributions

flag material timing divergence

## 129. Warning

Timing tests pueden ser flaky en CI.

- Por ello deberán utilizar:
- controlled environment
- broad thresholds
- statistical analysis

## 130. Secret Leakage Testing

Será una categoría propia.

## 131. Canary Secrets

Ejemplo:

- VOLTSTACK_TEST_PASSWORD_CANARY
- VOLTSTACK_TEST_TOKEN_CANARY
- VOLTSTACK_TEST_OTP_CANARY

## 132. Search surfaces

Después del scenario revisar:

- logs
- audit
- traces
- events
- exceptions
- responses
- queues
- snapshots

## 133. Expected

zero occurrence

## 134. Sensitive Parameter Test

También verificar stack traces de exceptions.

## 135. Multi-Tenant Isolation Testing

Obligatorio para cualquier componente tenant-aware.

## 136. Isolation Matrix

Identity
Session
Remember-Me
Token
MFA
Passkey mapping
Recovery
Device Trust
Risk History
Audit
Flow
todos deben probarse cross-tenant.

## 137. Tenant Switch Attack

Intentar modificar:

- tenant identifier
- durante flow.
- Debe rechazarse.

## 138. Shared Storage Isolation

Cuando tenants compartan Redis/DB:

- keys
- prefixes
- queries
- indexes
- deben conservar boundary.

## 139. Tenant Cache Poisoning

Cache de Tenant A no debe resolver Identity de Tenant B.

## 140. Security Realm Isolation

Además de tenant.

- Ejemplo:
- web
- admin
- api

## 141. Runtime Isolation Tests

Especialmente importante por FrankenPHP.

## 142. Worker Reuse Test

Request 1:
Alice authenticated

Request 2:
anonymous

Request 3:

- Bob
- El segundo debe ser realmente anonymous.

## 147. Mutable Singleton Detection

Tests podrán buscar leakage en services compartidos.

## 148. Fiber Concurrency Tests

Ejecutar dos Authenticaciones concurrentes:

- Alice
- Bob

con distintos:

- Tenant
- Flow
- Risk
- Device
- Session

## 149. Expected

Ningún estado cruzado.

## 150. Parallel Requests

Simular races en:

- session rotation
- credential rotation
- flow finalization
- OTP attempts
- device trust issuance

## 151. Race Testing

Puede ejecutarse repetidamente para aumentar probabilidad de detectar errores.

## 152. FrankenPHP Long-Run Test

Ejecutar cientos/miles de requests sobre workers reutilizados.

## 153. Memory Growth

Monitorizar:

- request count
- memory usage
- retained Authentication contexts

## 154. Expected

No crecimiento lineal por leakage.

## 155. Chaos / Fault Injection

Authentication deberá poder probarse bajo fallos parciales.

## 156. FaultInjector

Conceptualmente:

```php
$faults->failNext(
    AuthenticationDependency::SessionStore
);
```

## 157. Failure Modes

timeout
exception
corrupt response
stale response
connection reset
partial write

## 158. Database Failure

Debe verificarse failure policy.

## 159. Redis Failure

Especialmente:

- rate limiting
- session
- flow

## 160. Risk Provider Failure

No debe convertirse automáticamente en LOW risk.

## 161. OIDC Failure

No debe crear una sesión parcial.

## 162. Audit Backend Failure

Debe obedecer policy:

- fail closed
- buffer
- continue + alert

## 163. Event Listener Failure

Critical y best-effort deben comportarse distinto.

## 164. Fault Combination Tests

Ejemplo:

- OIDC login
- +;
- Risk Provider unavailable
- +;
- Audit degraded
- para validar composición.

## 165. Security Invariant Registry

VoltStack debería registrar formalmente invariantes.

## 166. AuthenticationInvariant

Conceptualmente:

```php
final readonly class AuthenticationInvariant
{
    public function __construct(
        public string $id,
        public string $description,
        public AuthenticationInvariantCategory $category,
    ) {}
}
```

## 167. IDs

Ejemplo:

- AUTH-ABUSE-01
- AUTH-RISK-03
- AUTH-DEVICE-TRUST-05
- AUTH-FLOW-05
- AUTH-OBS-SECRET-01
- AUTH-FAIL-01

## 168. Mapping

Cada invariant podrá mapear a uno o múltiples test IDs.

## 169. Conformance Report

Ejemplo:

- AUTH-ABUSE-01      PASS
- AUTH-RISK-03       PASS
- AUTH-DEVICE-RT-03  PASS
- AUTH-OBS-SECRET-01 PASS

## 170. Missing Coverage

Si una invariant no tiene test:

- UNVERIFIED
- No deberá marcarse como PASS.

## 171. Security Assurance Levels for Tests

Podrán existir perfiles:

- BASIC
- STANDARD
- HARDENED
- ENTERPRISE

## 172. BASIC

core unit
integration
basic security invariants

## 173. STANDARD

Añade:

- fuzzing
- property tests
- runtime isolation
- protocol validation

## 174. HARDENED

Añade:

- attack simulations
- fault injection
- timing tests
- concurrency stress
- mutation testing

## 175. ENTERPRISE

Puede añadir:

- external conformance
- compliance profiles

independent security test packs

## 176. AuthenticationConformanceProfile

Ejemplo:

- VOLTSTACK_AUTH_CORE_V1
- VOLTSTACK_AUTH_WEB_V1
- VOLTSTACK_AUTH_API_V1
- VOLTSTACK_AUTH_ENTERPRISE_V1

## 177. Core Profile

Podrá requerir:

- Identity
- Password
- Session
- Failure handling
- Testing invariants

## 178. Web Profile

Añade:

- CSRF
- Remember-Me
- Device Trust
- HTML flows

## 179. API Profile

Añade:

- Bearer
- stateless auth
- audience/issuer
- rate limiting

## 180. Enterprise Profile

Añade:

- MFA
- Passkeys
- Federation
- Risk
- Device policies
- Audit
- SIEM

## 181. Conformance Runner

interface AuthenticationConformanceRunnerInterface
{
public function run(
AuthenticationConformanceProfile $profile
): AuthenticationConformanceReport;
}

## 182. Plugin Certification

Un paquete externo podrá ejecutar:

- auth:conformance
- para demostrar que cumple contratos mínimos.

## 183. Authenticator Certification

Ejemplo:

- Custom SAML Authenticator
- podría ejecutar el suite de Authenticator.

## 184. Provider Certification

Igual para:

- Risk Provider
- Session Store
- Device Attestation Provider

## 185. Property-Based Testing

Muy útil para Authentication.

## 186. Ejemplo — Session

Propiedad:
Una Session revocada nunca vuelve a ser válida sin crear una Session nueva.

## 1. Ejemplo — Security Requirements

Propiedad:
Añadir un requisito de seguridad no puede reducir el assurance mínimo efectivo.

## 2. Ejemplo — Risk

Propiedad:
Un hard deny signal no puede transformarse en allow por añadir una señal neutral.

## 3. Ejemplo — Tenant

Propiedad:
Cambiar Tenant en una referencia no puede producir acceso a una Identity equivalente de otro tenant.

## 4. Generators

VoltStack Testing podrá proporcionar:

- IdentityGenerator
- CredentialGenerator
- SessionGenerator
- RiskSignalGenerator
- FlowGenerator
- TenantGenerator

## 5. Fuzz Testing

Objetivos principales:

- credential parsers
- tokens
- OAuth callbacks
- WebAuthn payloads
- flow tokens
- recovery inputs
- headers

## 6. Fuzz invariant

Para cualquier input arbitrario:

- no crash
- no secret disclosure
- no authentication bypass
- bounded resource consumption

## 7. Mutation Testing

Importante para security-sensitive code.

## 8. Example mutation

Cambiar:
if (!$valid)
por:

```php
if ($valid)
debe ser detectado por tests.
```

## 9. Targets

Especialmente:

- signature verification
- expiration checks
- tenant checks
- risk hard rules
- session revocation
- challenge replay

## 10. Mutation Score

Podrá incluirse en security quality gates.

## 11. Static Analysis

El Testing System también puede integrar:

- PHPStan/Psalm-style analysis
- taint analysis
- dead-code checks
- forbidden API checks
- según tooling.

## 12. Forbidden APIs

Ejemplo:
random_int for secret generation?
según contexto, o:

- md5
- sha1
- para password storage.

## 13. Architectural Static Rules

Podrán verificar:

- Authenticator cannot depend on HTTP Response
- Domain cannot depend on Laravel/Symfony adapters

Core cannot call session global directly

## 14. Dependency Rule Tests

Ejemplo:

```text
Quantum\Auth\Domain
    must not depend on
Quantum\Http\Response
```

## 15. Namespace Architecture Testing

Puede verificar la arquitectura documentada.

## 16. Security Regression Suite

Cada vulnerabilidad descubierta deberá producir:

- Regression Test
- antes o junto con la corrección.

## 17. Vulnerability Regression ID

Ejemplo:
AUTH-REG-2026-001

## 18. Regression Metadata

affected subsystem
attack type
fixed version
test reference

## 19. Never delete security regression lightly

Aunque implementación cambie.

## 20. Historical Attack Corpus

VoltStack podrá mantener casos para:

- OAuth state confusion
- JWT confusion
- session fixation
- remember-me replay
- passkey replay
- tenant crossover

## 21. Protocol Test Vectors

Deben preferirse vectores conocidos cuando sea posible.

## 22. JWT Test Corpus

Casos como:

- invalid signature
- wrong alg
- expired
- wrong issuer
- wrong audience
- malformed encoding

## 23. Algorithm Confusion

Debe existir test específico.

## 24. alg=none

Si formato/protocolo aplica:
reject

## 25. Key Confusion

Tests para impedir usar tipos de clave incorrectos.

## 26. OIDC Mix-Up

Si arquitectura lo soporta, deberá existir suite contra provider mix-up.

## 27. CSRF Testing

Form login/logout y OAuth flows.

## 28. Login CSRF

Simular request cross-site.

## 29. Logout CSRF

GET/logout o cross-site POST no autorizado debe ser rechazado según policy.

## 30. Open Redirect Testing

Payloads:

- <https://evil.example>
- //evil.example
- /%2F%2Fevil.example

javascript:
encoded variants

## 31. Expected

Solo destinos permitidos.

## 32. Header Injection Testing

Authentication responses no deberán reflejar valores inseguros en headers.

## 33. Cache-Control Testing

Páginas/respuestas sensibles deben tener policy apropiada.

## 34. Browser Cookie Testing

Verificar:

- Secure
- HttpOnly
- SameSite
- Path
- Domain
- expiration

para:

- Session
- Remember-Me
- Trusted Device
- Flow
- según diseño.

## 35. Cookie Isolation

Cookies de admin realm pueden requerir nombres/path distintos.

## 36. Cookie Prefixes

Si se utilizan:

- __Host-
- __Secure-

deben validarse según requisitos.

## 37. Session Cookie Leakage

Nunca aparecer en:

- URLs
- logs
- HTML
- trace

## 38. Performance Security Testing

No es benchmark general.
Busca detectar ataques de agotamiento.

## 39. Password Hashing DoS

Simular múltiples password attempts.

- Medir:
- CPU
- memory
- hash concurrency
- response behavior

## 40. Admission Control Test

Debe activar límites antes de saturar hashing.

## 41. Large Token Attack

Entradas gigantes no deben provocar consumo descontrolado.

## 42. Flow Cardinality Attack

Crear miles de flows.

- Debe existir:
- TTL
- rate limits
- storage bounds

## 43. Challenge Cardinality

Igual para:

- MFA
- Passkey
- Recovery
- Federation

## 44. Resource Bound Assertions

Tests podrán definir:

- max generated keys
- max memory growth
- max DB queries

## 45. Query Count Testing

Authentication hot path puede tener budgets.

## 46. Example

Password Login
<= N Identity queries
según implementation profile.

## 47. N no debe ser universal hardcode

Puede depender del provider.
Pero test profile puede definirlo.

## 48. Security Assurance Report

El runner podrá generar:

- passed invariants
- failed invariants
- unverified invariants
- protocol conformance
- mutation score
- fuzz status
- runtime isolation status

## 49. Machine-readable Report

Formato:

- JSON
- para CI.

## 50. Human-readable Report

Para developers/security reviewers.

## 51. CI Quality Gates

Ejemplo:

- 0 failed critical invariants
- 0 secret leakage failures
- 100% required conformance tests

mutation score >= configured threshold

## 52. Critical Invariant

Algunas pruebas serán:
BLOCKING

## 53. Example Blocking

revoked token authenticates
tenant isolation fails
password appears in logs
session fixation possible

## 54. Warning Tests

Ejemplo:

- timing divergence above advisory threshold
- puede ser warning hasta investigar.

## 55. Test Classification

CRITICAL
HIGH
STANDARD
ADVISORY

## 56. AuthenticationSecurityTestCase

Base class:

```php
abstract class AuthenticationSecurityTestCase extends TestCase
{
    protected AuthenticationTestHarness $auth;
    protected FakeClock $clock;
    protected AuthenticationLeakDetector $leaks;
}
```

## 57. AuthenticationLeakDetector

Componente importante.

## 58. LeakDetector surfaces

Logs
Audit
Traces
Events
Exceptions
HTTP Responses
Queues

## 59. API conceptual

$this->leaks
->assertSecretNotObserved($password);

## 60. AuthenticationSpy

Podrá observar:

- events
- metrics
- audit
- security signals
- en tests.

## 61. Example

AuthTest::spy()
->assertEventDispatched(AuthenticationFailed::class)
->assertNoSessionCreated();

## 62. Fake Authenticators

Podrán producir:

- verified
- failure
- challenge
- error
- de forma controlada.

## 63. Fake Identity Provider

Configurable:

- known identity
- unknown
- timeout
- error
- tenant mismatch

## 64. Fake Risk Provider

Podrá devolver:

- LOW
- HIGH
- CRITICAL
- TIMEOUT

## 65. Fake OIDC Provider

Debe poder simular:

- valid callback
- wrong state
- wrong nonce
- wrong issuer
- expired token
- timeout

## 66. Fake WebAuthn Client

Podrá producir test assertions sin depender de hardware real.

## 67. Real Browser Integration Tests

Además de fake clients, deberán existir algunos tests end-to-end con browser cuando sea posible.

## 68. Browser E2E

Validar:

- cookies
- redirects
- CSRF
- SPA protocol
- passkey browser integration

## 69. Hardware-dependent Tests

Security key / platform authenticator real podrá pertenecer a:

- optional hardware test suite
- no al CI básico.

## 70. Test Matrix

VoltStack deberá poder ejecutar matrices:

- PHP versions
- FrankenPHP
- PHP-FPM
- Session backends
- Redis
- Database drivers

## 71. Runtime Matrix

Al menos:

- traditional request lifecycle
- long-lived worker

## 72. Database Matrix

Cuando subsystem aplique:

- MySQL
- PostgreSQL
- MariaDB
- SQLite

## 73. Store Matrix

in-memory
database
Redis
para sessions/rate limits según soporte.

## 74. Authentication Protocol Version Matrix

Adapters externos podrán probar múltiples versiones compatibles.

## 75. Backward Compatibility Tests

Public Authentication contracts no deberán romperse accidentalmente.

## 76. Serialized Event Compatibility

Si existen events versionados:

- v1 event
- debe seguir siendo deserializable mientras support policy lo exija.

## 77. Public Error Contract Tests

Códigos como:

- AUTHENTICATION_FAILED
- ADDITIONAL_VERIFICATION_REQUIRED
- deben mantenerse estables.

## 78. Snapshot Testing

Puede usarse para:

- public JSON schemas
- problem details
- event envelopes

## 79. No snapshots de secrets

Los snapshots deberán pasar LeakDetector.

## 80. State Machine Model Testing

Ideal para:

- Authentication Flow
- Session
- Remember-Me Rotation
- Device Credential
- Recovery

## 81. Model-Based Testing

Generar secuencias arbitrarias:

- start
- challenge
- retry
- expire
- cancel
- resume

y verificar estados válidos.

## 82. Invalid Transition

Ejemplo:

```text
COMPLETED
→ CONTINUE
debe ser imposible.
```

## 83. Session Model

ACTIVE
→ REVOKED
no puede volver a:

- ACTIVE
- sin nueva Session.

## 84. Credential Rotation Model

ACTIVE gen1
→ SUPERSEDED
→ replay detected

## 85. Authentication Security Assertions

VoltStack podrá proporcionar helpers como:

```php
assertAuthenticated();
assertUnauthenticated();
assertAuthenticationDenied();
assertAuthenticationChallengeRequired();
assertAssuranceLevel();
assertRiskLevel();
assertSessionRotated();
assertNoSecretLeak();
```

## 86. Tenant Assertions

assertAuthenticatedInTenant($tenant);
assertNotAuthenticatedInTenant($otherTenant);

## 87. Device Assertions

assertDeviceTrusted();
assertDeviceNotTrusted();
assertDeviceRevoked();

## 88. Flow Assertions

assertFlowState(AuthenticationFlowStatus::ChallengePending);

## 89. Security Event Assertions

assertSecuritySignal(
SecuritySignalType::CredentialStuffingSuspected
);

## 90. Framework Developer Ergonomics

La intención será que un developer pueda escribir:

```php
AuthTest::actingAs($alice)
    ->withAssurance('aal2')
    ->withTrustedDevice()
    ->request('/billing')
    ->assertAllowed();
```

sin perder semántica real.

## 91. actingAs() Caveat

Debe distinguir:
test shortcut
de:
real authentication flow

## 92. Two Testing Modes

VoltStack deberá ofrecer:

- AUTHENTICATED_CONTEXT_MODE
- FULL_AUTHENTICATION_FLOW_MODE

## 93. Fast Context Mode

Para probar controllers/Authorization sin ejecutar login completo.

## 94. Full Flow Mode

Para probar Authentication real.

## 95. Important

Un test de Authentication nunca deberá usar solamente:

```php
actingAs($user)
para afirmar que Login funciona.
```

## 96. Security Test Tagging

Tests podrán etiquetarse:

- auth
- security
- conformance
- fuzz
- chaos
- slow
- frankenphp

## 97. CI Stages

Ejemplo:

```text
PR:
    unit + contract + integration
```

main:
security + conformance

nightly:
fuzz + mutation + chaos + stress

## 284. Long-Running Fuzzing

Podrá ejecutarse fuera del PR path.

## 285. Security Release Gate

Antes de releases mayores:

- full hardened suite
- deberá ejecutarse.

## 286. Threat Model Traceability

Cada amenaza documentada debería mapear a tests.
Ejemplo:

```text
Threat:
Credential Stuffing
```

Controls:

```text
Rate Limiting
Risk
Step-Up
```

Tests:
AUTH-ATTACK-STUFFING-001...

## 287. Threat-to-Test Matrix

Tooling futuro podrá generar:

- Threat
- Control
- Invariant
- Test
- Status

## 288. Security Requirement Traceability

También:

- Requirement
- Implementation component
- Test

## 289. Documentation Conformance

Los documentos Authentication podrán incluir IDs.
El runner podrá verificar cobertura de esos IDs.

## 290. Test Documentation

Security-sensitive tests deberán explicar:

- threat
- expected invariant
- failure impact

## 291. Example

```text
/**
 * Invariant: AUTH-FLOW-05
 *
 * A flow must revalidate critical security state before finalization.
 */
```

## 1. External Security Testing

Automated suite no sustituye:

- code review
- penetration testing
- cryptographic review
- external audit

## 2. Principle

Passing all framework tests is evidence of assurance, not mathematical proof of security.

## 3. Penetration Test Profile

VoltStack podrá facilitar testing externo mediante:

- test environment
- seed identities
- security headers
- audit correlation

## 4. Do not add insecure backdoors

No crear:

- /test-login-bypass
- en production code.

## 5. Dedicated Test Adapter

Si se necesita bypass en tests:

- TestAuthenticationProvider
- registrado exclusivamente en testing environment.

## 6. Environment Guard

Debe fallar bootstrap si test provider se intenta habilitar en producción.

## 7. Security Test Helpers

Deben respetar mismo principio.

## 8. Test Credentials

No utilizar credenciales reales.

## 9. CI Secret Hygiene

CI logs tampoco deberán mostrar fixtures sensibles aunque sean falsos si podrían enseñar patrones inseguros.

## 10. Authentication Test Data Factory

Namespace sugerido:
VoltStack\Testing\Auth\Factory

## 11. Example Factories

IdentityFactory
PasswordCredentialFactory
PasskeyCredentialFactory
SessionFactory
MfaFactorFactory
DeviceFactory

## 12. Test Scenario Library

VoltStack podrá incluir escenarios predefinidos:

- PasswordLoginScenario
- PasswordMfaScenario
- PasskeyLoginScenario
- FederatedLoginScenario
- RecoveryScenario
- CredentialStuffingScenario
- SessionHijackScenario

## 13. Reusable Security Scenarios

Packages podrán ejecutar los mismos escenarios contra sus adapters.

## 14. AuthenticationConformanceKit

Podrá distribuirse como:
VoltStack\Testing\Auth\Conformance

## 15. Namespace sugerido

VoltStack\Testing\Auth
VoltStack\Testing\Auth\Contracts
VoltStack\Testing\Auth\Harness
VoltStack\Testing\Auth\Scenario
VoltStack\Testing\Auth\Factory
VoltStack\Testing\Auth\Fake
VoltStack\Testing\Auth\Assertion
VoltStack\Testing\Auth\Security
VoltStack\Testing\Auth\Conformance
VoltStack\Testing\Auth\Fuzz
VoltStack\Testing\Auth\Chaos
VoltStack\Testing\Auth\Runtime

## 16. Estructura sugerida

src/Testing/Auth/
├── Contracts/
│   ├── AuthenticationTestHarnessInterface.php
│   └── AuthenticationConformanceRunnerInterface.php
│
├── Harness/
│   ├── AuthenticationTestHarness.php
│   ├── AuthenticationTestEnvironment.php
│   └── AuthenticationTestExecution.php
│
├── Scenario/
│   ├── AuthenticationTestScenario.php
│   ├── PasswordLoginScenario.php
│   ├── PasswordMfaScenario.php
│   ├── PasskeyLoginScenario.php
│   ├── FederatedLoginScenario.php
│   └── RecoveryScenario.php
│
├── Factory/
│   ├── IdentityFactory.php
│   ├── PasswordCredentialFactory.php
│   ├── SessionFactory.php
│   ├── DeviceFactory.php
│   └── MfaFactorFactory.php
│
├── Fake/
│   ├── FakeClock.php
│   ├── FakeAuthenticator.php
│   ├── FakeIdentityProvider.php
│   ├── FakeRiskProvider.php
│   ├── FakeOidcProvider.php
│   └── FakeWebAuthnClient.php
│
├── Assertion/
│   ├── AuthenticationAssertions.php
│   ├── FlowAssertions.php
│   ├── SessionAssertions.php
│   ├── RiskAssertions.php
│   └── TenantIsolationAssertions.php
│
├── Security/
│   ├── AuthenticationLeakDetector.php
│   ├── AuthenticationInvariantRegistry.php
│   ├── AuthenticationSecurityTestCase.php
│   └── SecurityRegressionRegistry.php
│
├── Conformance/
│   ├── AuthenticationConformanceProfile.php
│   ├── AuthenticationConformanceRunner.php
│   ├── AuthenticationConformanceReport.php
│   ├── AuthenticatorContractTestSuite.php
│   ├── IdentityProviderContractTestSuite.php
│   └── RateLimitStoreContractTestSuite.php
│
├── Fuzz/
│   ├── CredentialFuzzer.php
│   ├── TokenFuzzer.php
│   ├── WebAuthnFuzzer.php
│   └── FederationFuzzer.php
│
├── Chaos/
│   ├── AuthenticationFaultInjector.php
│   └── AuthenticationChaosScenario.php
│
└── Runtime/
├── FrankenPhpWorkerIsolationTest.php
├── FiberAuthenticationIsolationTest.php
└── DistributedAuthenticationTestHarness.php

## 17. Testing configuration

return [

'authentication_testing' => [

'clock' => 'fake',

'stores' => [
'session' => 'memory',
'rate_limit' => 'memory',
'flow' => 'memory',
],

'leak_detection' => true,

'conformance_profile' => 'standard',

],

];

## 309. Production Guard

Config de testing deberá estar deshabilitada en producción.

## 310. Example Full Test

public function test_high_risk_password_login_requires_passkey(): void
{
$alice = $this->identity()
->active()
->withPassword('secret')
->withPasskey();

$this->risk->returnHighRisk();

$result = $this->auth->login(
identity: $alice,
password: 'secret',
);

$this->assertAuthenticationChallengeRequired($result);
$this->assertPasskeyRequired($result);
$this->assertNoSessionCreated();
}

## 311. Secret Leakage Test Example

public function test_password_never_leaks_to_observability(): void
{
$password = 'VOLTSTACK_TEST_SECRET_CANARY';

$this->attemptInvalidLogin($password);

$this->leaks->assertSecretNotObserved($password);
}

## 312. Tenant Isolation Example

public function test_flow_cannot_cross_tenants(): void
{
$flow = $this->startLoginInTenant('tenant-a');

$result = $this->continueFlow(
$flow,
tenant: 'tenant-b',
);

$this->assertAuthenticationDenied($result);
}

## 313. FrankenPHP Example

Worker #1

Request A
Identity Alice
Tenant A
Flow F1

Request B
Anonymous
Tenant B

Request C
Identity Bob
Tenant B
Flow F2
Assertions:

- B has no Alice state
- C has no Alice/F1 state

A audit never appears as B context

## 314. Complete Verification Pipeline

Source Code
↓
Static Analysis
↓
Unit Tests
↓
Contract Tests
↓
Integration Tests
↓
Security Invariant Tests
↓
Protocol Conformance
↓
Fuzz / Property Tests
↓
Concurrency / FrankenPHP
↓
Fault Injection
↓
Mutation Testing
↓
Conformance Report

## 315. Security Invariants — Testing System

AUTH-TEST-01
Every critical Authentication subsystem has automated tests.
AUTH-TEST-02
Every critical documented security invariant is mapped to at least one automated verification.
AUTH-TEST-03
Unverified invariants are never reported as passed.

- AUTH-TEST-04
- Test shortcuts cannot be enabled in production.
- AUTH-TEST-05

Authentication testing itself does not introduce credential leakage.

## 316. Security Invariants — Credentials

AUTH-TEST-CRED-01
Invalid credentials never produce authenticated context.

- AUTH-TEST-CRED-02
- Revoked credentials never authenticate.
- AUTH-TEST-CRED-03

Expired credentials never authenticate after expiry.

- AUTH-TEST-CRED-04
- Credential test suites include secret leakage verification.
- AUTH-TEST-CRED-05

Custom Authenticators must pass common contract tests.

## 317. Security Invariants — Flow

AUTH-TEST-FLOW-01
Challenges are bound to their originating Flow.

- AUTH-TEST-FLOW-02
- Expired flows cannot continue.
- AUTH-TEST-FLOW-03

Completed flows cannot be replayed.

- AUTH-TEST-FLOW-04
- Critical security state is revalidated before finalization.
- AUTH-TEST-FLOW-05

Concurrent finalization cannot create inconsistent Authentication state.

## 318. Security Invariants — Tenant

AUTH-TEST-TENANT-01
Authentication state never crosses Tenant boundaries.

- AUTH-TEST-TENANT-02
- Session stores preserve Tenant isolation.
- AUTH-TEST-TENANT-03

Device Trust does not automatically cross Tenant boundaries.

- AUTH-TEST-TENANT-04
- Risk history is isolated by Tenant.
- AUTH-TEST-TENANT-05

Flow continuation validates Tenant binding.

## 319. Security Invariants — Runtime

AUTH-TEST-RT-01
FrankenPHP worker reuse does not leak Authentication state.

- AUTH-TEST-RT-02
- Concurrent fibers maintain isolated Authentication contexts.
- AUTH-TEST-RT-03

Shared singleton services do not retain mutable current Identity state.
AUTH-TEST-RT-04
Distributed nodes respect authoritative revocation state.
AUTH-TEST-RT-05
Long-running stress tests do not show unbounded Authentication context retention.

## 320. Security Invariants — Failure Injection

AUTH-TEST-FAULT-01
Critical dependency failure never silently lowers security requirements.
AUTH-TEST-FAULT-02
Risk provider failure never automatically becomes low risk.

- AUTH-TEST-FAULT-03
- Session establishment failure never produces finalized Authentication.
- AUTH-TEST-FAULT-04

External provider timeout follows explicit retry/failure policy.
AUTH-TEST-FAULT-05
Audit failure follows configured audit failure semantics.

## 321. Anti-pattern — Solo probar 200 OK

Incorrecto.

## 322. Anti-pattern — Authentication tests usando siempre actingAs()

No valida Authentication real.

## 323. Anti-pattern — No probar negative paths

Peligroso.

## 324. Anti-pattern — No tests de concurrency

Inaceptable para Session, Flow y credential rotation.

## 325. Anti-pattern — No tests con FrankenPHP

Contradice uno de los objetivos centrales de VoltStack.

## 326. Anti-pattern — Timing exact assertions

Generan tests frágiles.

## 327. Anti-pattern — Fuzzing sin resource bounds

El fuzzer mismo puede colgar CI.

## 328. Anti-pattern — Test secrets impresos en output

No.

## 329. Anti-pattern — Mockear todo

Puede hacer que Integration y Security Tests no detecten errores reales.

## 330. Anti-pattern — Solo Integration Tests

Sin contract/property tests se pierden garantías.

## 331. Anti-pattern — Conformance declarado manualmente

No.
Debe derivarse del runner.

## 332. Anti-pattern — Mutation testing ignorado en crypto/security checks

Debería utilizarse en áreas críticas.

## 333. Anti-pattern — Vulnerabilidad corregida sin Regression Test

Evitar.

## 334. Anti-pattern — Test Provider disponible en producción

Nunca.

## 335. Relación con Laravel y Symfony

Laravel ofrece una excelente experiencia de testing mediante:

- HTTP tests
- actingAs
- assertAuthenticated
- assertGuest
- fakes
- events
- queues
- time travel

Symfony aporta herramientas sólidas mediante:

- KernelTestCase
- WebTestCase
- security test clients
- service container replacement
- HTTP client testing
- event testing

VoltStack conservará esa ergonomía, pero añadirá una capa específica para seguridad de Authentication:

- Authentication Test Harness
- Authenticator Contract Tests
- Security Invariant Registry
- Conformance Profiles
- Secret Leak Detection

Flow State Machine Testing
FrankenPHP Worker Isolation
Tenant Isolation Suites
Attack Simulation
Protocol Conformance

## 336. Diferenciador VoltStack

El objetivo no será solamente permitir:
$this->assertAuthenticated();
sino poder demostrar:

```php
Authenticated = true
Assurance = phishing-resistant
Risk = LOW
Flow = correctly finalized
Session = rotated
Tenant = correct
```

Device Trust = scoped correctly
No credentials leaked
No security invariant violated

## 337. Quality Gate recomendado

Antes de declarar estable el sistema Authentication V1:

- All CRITICAL invariants      PASS
- All core contract suites     PASS

Tenant isolation suite       PASS
FrankenPHP isolation suite   PASS
Secret leakage suite         PASS
Protocol conformance         PASS
Critical mutation tests      PASS
Security regression suite    PASS

## 338. Regla arquitectónica final

VoltStack deberá preservar:

```text
AUTHENTICATION SPECIFICATION
            │
            ▼
     SECURITY INVARIANTS
            │
            ▼
   AUTOMATED TEST MAPPING
            │
   ┌────────┼───────────┬───────────┐
   ▼        ▼           ▼           ▼
```

CONTRACT  SECURITY   CONFORMANCE  RUNTIME
TESTS      TESTS       TESTS       TESTS
│        │           │           │
└────────┴───────────┴───────────┘
│
▼
ASSURANCE REPORT
│
┌─────────┼─────────┐
▼         ▼         ▼
PASS      FAILED   UNVERIFIED
La primera regla central será:
VoltStack no considerará una garantía de Authentication como verificada simplemente porque la implementación parece correcta; las invariantes críticas deberán disponer de pruebas automáticas repetibles.

La segunda:
Los tests deberán demostrar tanto que las operaciones válidas funcionan como que estados inválidos, ataques, replays, fallos parciales, condiciones de carrera y cruces de tenant no pueden producir Authentication indebida.

La tercera:
El entorno de testing deberá poder controlar tiempo, dependencias, riesgo, fallos de infraestructura, concurrencia y contexto de dispositivo sin introducir primitives inseguras en producción.

La cuarta:
FrankenPHP, multi-tenancy, concurrency y secret leakage serán dimensiones obligatorias de verificación del Authentication Core, no suites opcionales añadidas al final.

La quinta:
La conformidad del sistema deberá expresarse mediante evidencia verificable —tests, invariants, reports y regression suites— y no únicamente mediante documentación.

Siguiente documento recomendado
La secuencia natural puede continuar con:
`27_AUTHENTICATION_COMPILATION_CONFIGURATION_VALIDATION_CACHE_OPTIMIZATION_AND_RUNTIME_PERFORMANCE_SYSTEM.md`
Ese documento nos permitiría formalizar cómo toda la arquitectura que hemos construido se prepara para producción sin resolver cientos de decisiones dinámicamente en cada request:

- Authentication Configuration
- Configuration Schema
- Configuration Validation
- Semantic Validation
- Firewall Compilation
- Guard Compilation
- Authenticator Registry Compilation
- Provider Registry Compilation
- Authentication Policy Compilation
- Risk Policy Compilation
- MFA Policy Compilation
- Flow Policy Compilation

Event / Hook Registry Compilation
Dependency Graph Validation
Conflict Detection
Authentication Cache
Compiled Metadata
Precomputed Resolvers
Fast Paths
Warmup
Cold Start
FrankenPHP Bootstrap
Immutable Runtime State
Request-local State
Memory Safety
Performance Budgets
Authentication Profiling
Optimization
Cache Invalidation
Development vs Production Modes
La idea central será separar:

```text
BOOTSTRAP / COMPILE TIME
    resolver todo lo estático

REQUEST TIME
    resolver únicamente Identity, Credential,
    Risk, Device, Tenant y contexto dinámico
```

Esto será especialmente importante para que toda la riqueza del sistema de Authentication que llevamos construida no convierta cada login en una cadena innecesariamente costosa de resolución de configuración.
