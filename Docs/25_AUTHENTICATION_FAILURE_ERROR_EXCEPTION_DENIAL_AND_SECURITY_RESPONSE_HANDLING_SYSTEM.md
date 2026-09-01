# VoltStack Authentication System

## 25 — Authentication Failure, Error, Exception, Denial and Security Response Handling System

- **Archivo:** `25_AUTHENTICATION_FAILURE_ERROR_EXCEPTION_DENIAL_AND_SECURITY_RESPONSE_HANDLING_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de fallos, errores, excepciones, denegaciones, challenges y respuestas seguras de Authentication.

---

## 1. Propósito

Este documento define cómo VoltStack representará, clasificará, propagará, transformará y responderá ante cualquier resultado no exitoso producido durante Authentication.
El sistema deberá distinguir explícitamente entre:

- Authentication Failure
- Credential Failure
- Identity Failure
- Authentication Denial
- Security Denial
- Authentication Challenge
- Step-Up Requirement
- Flow Failure
- Flow Expiration
- Provider Failure
- Infrastructure Failure
- Configuration Failure
- Protocol Failure
- Authentication Exception
- Authentication Error
- Safe Public Response
- Internal Diagnostic Response
- Retryable Failure
- Terminal Failure

El objetivo principal es evitar que todos los problemas de Authentication terminen representados mediante una única excepción genérica.

## 2. Problema arquitectónico

Un diseño simplista suele terminar haciendo:

```php
try {
    $user = $auth->authenticate($request);
} catch (\Throwable $e) {
    return response('Authentication failed', 401);
}
```

Esto mezcla problemas completamente distintos:

- Password incorrecto
- Usuario suspendido
- MFA requerido

Risk Engine rechazó el acceso
Rate Limiter bloqueó el intento
Sesión expirada
OIDC Provider no disponible
Redis caído
Configuración inválida
Authenticator defectuoso
Flow expirado
CSRF inválido
Todos podrían acabar convertidos incorrectamente en:

- 401 Unauthorized
- VoltStack no deberá utilizar ese modelo.

## 3. Principio fundamental

Un resultado de Authentication no exitoso no implica necesariamente una excepción.

Por ejemplo:

- Invalid Password
- MFA Required
- Passkey Required
- Authentication Denied
- Session Expired
- Rate Limited

son resultados esperables del dominio.

## 4. Segunda regla fundamental

Las excepciones representan condiciones excepcionales; los resultados normales del protocolo de Authentication deberán representarse mediante objetos de dominio.

## 1. Taxonomía principal

VoltStack utilizará cinco categorías fundamentales:

- SUCCESS
- FAILURE
- DENIAL
- CHALLENGE
- ERROR

## 2. SUCCESS

Authentication alcanzó el estado requerido.
AuthenticationSuccess
Ejemplo:

- Password valid
- Identity eligible
- Risk acceptable
- MFA satisfied
- Session established

## 3. FAILURE

La evidencia presentada no pudo satisfacer Authentication.

- Ejemplo:
- INVALID_CREDENTIAL
- INVALID_PASSKEY_ASSERTION
- INVALID_OTP
- INVALID_BEARER_TOKEN

Esto normalmente no es una excepción.

## 4. DENIAL

La identidad o evidencia puede ser válida, pero una política impide continuar.
Ejemplo:

- ACCOUNT_DISABLED
- ACCOUNT_LOCKED
- RISK_DENIED
- DEVICE_BLOCKED
- AUTHENTICATION_NOT_ALLOWED

## 5. CHALLENGE

Authentication todavía no terminó.

- Se necesita evidencia adicional.
- MFA_REQUIRED
- PASSKEY_REQUIRED
- FRESH_AUTH_REQUIRED
- PASSWORD_CHANGE_REQUIRED

## 6. ERROR

Authentication no pudo evaluarse correctamente debido a un problema técnico.
Ejemplo:

- IDENTITY_PROVIDER_UNAVAILABLE
- SESSION_STORE_UNAVAILABLE
- RISK_PROVIDER_TIMEOUT
- CONFIGURATION_ERROR

## 7. Modelo conceptual

Authentication Attempt
│
▼
Authentication Processing
│
▼
Authentication Outcome
│
┌──────┼───────┬───────────┬───────────┐
▼      ▼       ▼           ▼           ▼
SUCCESS FAILURE DENIAL    CHALLENGE     ERROR

## 8. AuthenticationOutcome

Contrato conceptual:

```php
interface AuthenticationOutcome
{
    public function type(): AuthenticationOutcomeType;
}
```

## 9. AuthenticationOutcomeType

enum AuthenticationOutcomeType: string
{
case Success = 'success';
case Failure = 'failure';
case Denial = 'denial';
case Challenge = 'challenge';
case Error = 'error';
}

## 10. AuthenticationFailure

final readonly class AuthenticationFailure implements AuthenticationOutcome
{
public function __construct(
public AuthenticationFailureCode $code,
public AuthenticationReasonSet $reasons,
public AuthenticationFailureContext $context,
) {}
}

## 11. AuthenticationDenial

final readonly class AuthenticationDenial implements AuthenticationOutcome
{
public function __construct(
public AuthenticationDenialCode $code,
public AuthenticationReasonSet $reasons,
public AuthenticationDenialContext $context,
) {}
}

## 12. AuthenticationChallenge

final readonly class AuthenticationChallenge implements AuthenticationOutcome
{
public function __construct(
public AuthenticationChallengeType $type,
public AuthenticationRequirementSet $requirements,
public AuthenticationContinuation $continuation,
) {}
}

## 13. AuthenticationError

final readonly class AuthenticationError implements AuthenticationOutcome
{
public function __construct(
public AuthenticationErrorCode $code,
public AuthenticationErrorCategory $category,
public bool $retryable,
public ?AuthenticationDiagnosticReference $diagnostic = null,
) {}
}

## 14. Failure Taxonomy

Las fallas deberán dividirse al menos en:

- CREDENTIAL_FAILURE
- IDENTITY_FAILURE
- PROTOCOL_FAILURE
- FLOW_FAILURE
- SESSION_FAILURE
- TOKEN_FAILURE
- FACTOR_FAILURE
- FEDERATION_FAILURE
- RECOVERY_FAILURE

## 15. Credential Failure

Ejemplos:

- INVALID_PASSWORD
- INVALID_PASSKEY
- INVALID_OTP
- INVALID_RECOVERY_CODE
- INVALID_BEARER_TOKEN
- INVALID_REMEMBER_ME_CREDENTIAL

## 16. Identity Failure

Ejemplos:

- IDENTITY_NOT_FOUND
- IDENTITY_UNRESOLVABLE
- IDENTITY_MAPPING_FAILED

Sin embargo, esta información no necesariamente deberá exponerse al cliente.

## 17. Account Enumeration Resistance

Externamente:

- IDENTITY_NOT_FOUND
- INVALID_PASSWORD

podrán convertirse ambos en:
INVALID_CREDENTIALS

## 18. Internal vs Public Failure

Internamente:
IDENTITY_NOT_FOUND
Externamente:

- AUTHENTICATION_FAILED
- Esta separación será obligatoria.

## 19. AuthenticationFailureCode

Ejemplo:

```php
enum AuthenticationFailureCode: string
{
    case InvalidCredentials = 'invalid_credentials';
    case InvalidPassword = 'invalid_password';
    case InvalidOtp = 'invalid_otp';
    case InvalidPasskey = 'invalid_passkey';
    case InvalidToken = 'invalid_token';
    case ExpiredCredential = 'expired_credential';
}
```

## 20. Denial Taxonomy

IDENTITY_DENIAL
SECURITY_POLICY_DENIAL
RISK_DENIAL
DEVICE_DENIAL
ABUSE_DENIAL
TENANT_DENIAL
ASSURANCE_DENIAL

## 21. Identity Denial

Ejemplos:

- ACCOUNT_DISABLED
- ACCOUNT_SUSPENDED
- ACCOUNT_LOCKED
- ACCOUNT_EXPIRED
- ACCOUNT_NOT_YET_ACTIVE

## 22. Security Policy Denial

Ejemplos:

- AUTHENTICATION_METHOD_NOT_ALLOWED
- LOGIN_WINDOW_RESTRICTED
- NETWORK_NOT_ALLOWED
- SECURITY_HOLD_ACTIVE

## 23. Risk Denial

RISK_TOO_HIGH
KNOWN_COMPROMISED_DEVICE
IMPOSSIBLE_TRAVEL
COMPROMISED_CREDENTIAL

## 24. Device Denial

DEVICE_REVOKED
DEVICE_COMPROMISED
DEVICE_BLOCKED
DEVICE_CREDENTIAL_REPLAY

## 25. Abuse Denial

RATE_LIMITED
BRUTE_FORCE_BLOCKED
CREDENTIAL_STUFFING_BLOCKED
PASSWORD_SPRAYING_BLOCKED
AUTOMATION_BLOCKED

## 26. Assurance Denial

REQUIRED_ASSURANCE_UNAVAILABLE
PHISHING_RESISTANT_FACTOR_UNAVAILABLE
REQUIRED_FACTOR_NOT_ENROLLED

## 27. AuthenticationDenialCode

enum AuthenticationDenialCode: string
{
case AccountDisabled = 'account_disabled';
case AccountLocked = 'account_locked';
case SecurityPolicy = 'security_policy';
case RiskTooHigh = 'risk_too_high';
case DeviceBlocked = 'device_blocked';
case RateLimited = 'rate_limited';
case InsufficientAssurance = 'insufficient_assurance';
}

## 28. Failure vs Denial

Esta distinción será importante:

```text
Password incorrecto
    ↓
FAILURE
```

Password correcto + cuenta suspendida
↓
DENIAL

## 33. Challenge Taxonomy

VoltStack deberá reconocer:

- PASSWORD_CHALLENGE
- MFA_CHALLENGE
- PASSKEY_CHALLENGE
- FEDERATED_CHALLENGE
- STEP_UP_CHALLENGE
- FRESH_AUTH_CHALLENGE
- PASSWORD_CHANGE_CHALLENGE
- RECOVERY_CHALLENGE
- CONSENT_CHALLENGE

## 34. Challenge no es Failure

Ejemplo:

```text
Password valid
    ↓
MFA required
```

No significa:
Authentication failed
Significa:
Authentication incomplete

## 35. Step-Up

Resultado:
STEP_UP_REQUIRED
deberá incluir:

- required assurance
- allowed factors
- flow reference
- expiration

## 36. AuthenticationContinuation

Representará la posibilidad de continuar el flow.

```php
final readonly class AuthenticationContinuation
{
    public function __construct(
        public AuthenticationFlowReference $flow,
        public AuthenticationStepReference $step,
        public \DateTimeImmutable $expiresAt,
    ) {}
}
```

## 37. Continuation != Credential

La referencia de continuación no deberá convertirse automáticamente en evidencia suficiente para Authentication.

## 38. Flow Failures

Ejemplos:

- FLOW_NOT_FOUND
- FLOW_EXPIRED
- FLOW_ALREADY_COMPLETED
- FLOW_CANCELLED
- FLOW_STATE_INVALID
- FLOW_STEP_INVALID
- FLOW_REPLAY_DETECTED

## 39. Expired Flow

No necesariamente es:
401
Puede requerir:
restart authentication

## 40. Replay Detection

Un flow ya consumido que vuelve a presentarse deberá generar:

- FLOW_REPLAY_DETECTED
- y potencialmente un Security Event.

## 41. Session Failures

SESSION_NOT_FOUND
SESSION_EXPIRED
SESSION_REVOKED
SESSION_INVALID
SESSION_CONTEXT_MISMATCH
SESSION_ROTATION_FAILED

## 42. Session Expired

En aplicación web:
redirect to login
En API:

```php
401
La semántica del dominio permanece igual; cambia la representación.
```

## 43. Token Failures

TOKEN_MISSING
TOKEN_MALFORMED
TOKEN_INVALID
TOKEN_EXPIRED
TOKEN_REVOKED
TOKEN_NOT_YET_VALID
TOKEN_AUDIENCE_INVALID
TOKEN_ISSUER_INVALID
TOKEN_SIGNATURE_INVALID
TOKEN_SCOPE_INVALID

## 44. Token Error vs Token Failure

Firma inválida:

```text
TOKEN_SIGNATURE_INVALID
    → FAILURE
```

JWK endpoint caído:

```text
TOKEN_VERIFICATION_PROVIDER_UNAVAILABLE
    → ERROR
```

## 45. MFA Failures

MFA_CODE_INVALID
MFA_CODE_EXPIRED
MFA_CHALLENGE_EXPIRED
MFA_ATTEMPTS_EXCEEDED
MFA_FACTOR_REVOKED

## 46. Passkey Failures

PASSKEY_ASSERTION_INVALID
PASSKEY_CHALLENGE_INVALID
PASSKEY_CHALLENGE_EXPIRED
PASSKEY_ORIGIN_INVALID
PASSKEY_RP_ID_INVALID
PASSKEY_COUNTER_ANOMALY
PASSKEY_CREDENTIAL_REVOKED

## 47. Passkey Counter Anomaly

Puede convertirse en:

- DENIAL
- +;
- SECURITY SIGNAL
- dependiendo de policy.

## 48. Federation Failures

OAUTH_STATE_INVALID
OAUTH_STATE_EXPIRED
OIDC_NONCE_INVALID
OIDC_TOKEN_INVALID
OIDC_ISSUER_INVALID
OIDC_AUDIENCE_INVALID
FEDERATED_IDENTITY_MAPPING_FAILED
FEDERATED_ACCOUNT_NOT_LINKED

## 49. Federation Errors

Separados:

- OIDC_PROVIDER_UNAVAILABLE
- OAUTH_TOKEN_ENDPOINT_TIMEOUT
- JWKS_ENDPOINT_UNAVAILABLE
- FEDERATION_METADATA_UNAVAILABLE

## 50. Recovery Failures

RECOVERY_TOKEN_INVALID
RECOVERY_TOKEN_EXPIRED
RECOVERY_TRANSACTION_EXPIRED
RECOVERY_EVIDENCE_INVALID
RECOVERY_REPLAY_DETECTED

## 51. Recovery Denials

RECOVERY_NOT_ALLOWED
SECURITY_HOLD_ACTIVE
RECOVERY_RISK_TOO_HIGH
INSUFFICIENT_RECOVERY_ASSURANCE

## 52. Error Taxonomy

Los errores técnicos deberán dividirse en:

- INFRASTRUCTURE_ERROR
- PROVIDER_ERROR
- CONFIGURATION_ERROR
- PROGRAMMING_ERROR
- PROTOCOL_ERROR
- STORAGE_ERROR
- CRYPTOGRAPHIC_ERROR
- TIMEOUT_ERROR
- DEPENDENCY_ERROR

## 53. Infrastructure Error

Ejemplos:

- Database unavailable
- Redis unavailable
- Session backend unavailable
- Queue unavailable

## 54. Provider Error

Risk Provider unavailable
OIDC Provider unavailable
External Identity Provider unavailable

## 55. Configuration Error

Ejemplos:

- Authenticator referenced but not registered
- Missing signing key
- Invalid firewall configuration
- Unsupported authentication method

## 56. Programming Error

Ejemplo:

- Unexpected invariant violation
- Invalid internal state
- Unsupported result object

Estos sí deberán tender a exceptions.

## 57. Protocol Error

Ejemplo:

- Malformed OAuth callback
- Invalid WebAuthn response structure

Invalid authentication continuation payload

## 58. Cryptographic Error

Distinguir:
cryptographic verification returned invalid
de:

- cryptographic provider malfunctioned
- El primero puede ser Failure.
- El segundo Error.

## 59. Authentication Exception Hierarchy

Se propone:

```text
AuthenticationException
│
├── AuthenticationInfrastructureException
├── AuthenticationProviderException
├── AuthenticationConfigurationException
├── AuthenticationProtocolException
├── AuthenticationStorageException
├── AuthenticationCryptographicException
├── AuthenticationTimeoutException
└── AuthenticationInvariantViolationException
```

## 60. No InvalidCredentialsException por default

VoltStack deberá evitar usar exceptions como control de flujo normal:
throw new InvalidCredentialsException();
preferiblemente:
return AuthenticationFailure::invalidCredentials();

## 61. Exceptions en adapters

Adapters externos sí pueden lanzar excepciones.

```php
Ejemplo:
try {
    $provider->verify(...);
} catch (ProviderTimeoutException $e) {
    // translate
}
```

## 62. Exception Translation Boundary

Toda excepción de infraestructura deberá traducirse antes de llegar al dominio.
RedisException
↓
SessionStoreUnavailable
↓
AuthenticationError

## 63. No vendor exceptions

El Core no deberá depender de:

- PredisException
- PDOException
- GuzzleException
- Symfony HttpClient exception

## 64. AuthenticationExceptionTranslator

interface AuthenticationExceptionTranslatorInterface
{
public function translate(
\Throwable $exception,
AuthenticationExecutionContext $context
): AuthenticationError;
}

## 65. Exception Mapping

Ejemplo:

```text
PDOException
    ↓
AUTH_STORAGE_UNAVAILABLE

TimeoutException
    ↓
AUTH_PROVIDER_TIMEOUT

InvalidConfigurationException
    ↓
AUTH_CONFIGURATION_ERROR
```

## 66. Unknown Exception

Debe convertirse en:

- AUTH_INTERNAL_ERROR
- externamente.

Internamente conservará correlation/diagnostic information.

## 67. Throwable boundary

Debe existir un boundary final alrededor del Authentication Pipeline.
Request
↓
Authentication Pipeline
↓
Authentication Exception Boundary
↓
Outcome

## 68. AuthenticationFailureBoundary

Conceptualmente:

```php
final class AuthenticationFailureBoundary
{
    public function execute(
        AuthenticationRequest $request
    ): AuthenticationOutcome {
        try {
            return $this->pipeline->authenticate($request);
        } catch (\Throwable $e) {
            return $this->translator->translate($e);
        }
    }
}
```

## 69. No catch(Throwable) disperso

Debe centralizarse en boundaries apropiados.

## 70. Fail-Open vs Fail-Closed

Cada dependencia de seguridad deberá definir explícitamente su estrategia.

## 71. Fail-Closed

Si no puede comprobarse una condición crítica:
deny authentication

## 72. Fail-Open

Solo permitido cuando una policy explícita determine que la dependencia no es crítica.

## 73. Ejemplo Risk Provider

Para administrador:

```text
Risk Provider unavailable
    ↓
FAIL_CLOSED
```

Para aplicación pública de bajo riesgo:

```text
Risk Provider unavailable
    ↓
DEGRADED_POLICY
podría ser configurable.
```

## 74. Nunca fail-open implícito

No:

```php
try {
    return $risk->evaluate();
} catch (\Throwable) {
    return Risk::low();
}
Esto sería peligroso.
```

## 75. Dependency Failure Policy

enum AuthenticationDependencyFailureMode: string
{
case FailClosed = 'fail_closed';
case FailOpen = 'fail_open';
case Degraded = 'degraded';
case Retry = 'retry';
}

## 76. Degraded Authentication

VoltStack podrá representar:

- AUTHENTICATION_DEGRADED
- internamente.

## 77. Degraded != Success

Debe conservar información de que alguna capacidad estuvo indisponible.

## 78. Retryability

Todo Error podrá indicar:
retryable=true|false

## 79. Retryable

Ejemplos:

- PROVIDER_TIMEOUT
- SESSION_STORE_TEMPORARILY_UNAVAILABLE
- RATE_LIMITED
- FEDERATION_PROVIDER_UNAVAILABLE

## 80. Non-Retryable

Ejemplos:

- INVALID_CREDENTIALS
- ACCOUNT_DISABLED
- TOKEN_REVOKED
- PASSKEY_REVOKED

## 81. Retry Policy

Debe incluir:

- maximum attempts
- backoff
- jitter
- timeout budget

## 82. No retry de passwords

VoltStack no deberá reintentar automáticamente credenciales inválidas.

## 83. Infrastructure retry

Sí puede reintentar:

- temporary network failure
- cuando sea seguro.

## 84. AuthenticationRetryPolicy

interface AuthenticationRetryPolicyInterface
{
public function decide(
AuthenticationError $error,
AuthenticationExecutionContext $context
): AuthenticationRetryDecision;
}

## 85. Retry Budget

Evitar:
infinite retry

## 86. Error Cascades

Un provider caído no debe generar miles de retries simultáneos.
Podrán utilizarse:

- Circuit Breaker
- Retry Budget
- Backoff
- Bulkhead

## 87. Circuit Breaker

Especialmente útil para:

- OIDC Provider
- Risk Provider
- Remote Identity Provider

## 88. Authentication Security Response

Después de obtener un Outcome:

```text
Outcome
    ↓
Security Response Mapper
    ↓
Transport-specific Response
```

## 89. AuthenticationResponseMapper

interface AuthenticationResponseMapperInterface
{
public function map(
AuthenticationOutcome $outcome,
AuthenticationResponseContext $context
): AuthenticationResponse;
}

## 90. Response Context

Puede contener:

- transport
- route
- firewall
- client type
- content negotiation
- SPA mode
- API mode
- tenant
- localization

## 91. Response Types

RedirectAuthenticationResponse
JsonAuthenticationResponse
ProblemDetailsAuthenticationResponse
SpaAuthenticationResponse
ChallengeAuthenticationResponse
EmptyAuthenticationResponse

## 92. Domain != HTTP

El dominio no deberá devolver:
new JsonResponse(...)

## 93. HTTP Adapter

Será responsabilidad de:

- Authentication HTTP Transport
- convertir Outcomes.

## 94. HTTP Status Mapping

Mapeo típico:

```text
Invalid credentials
    → 401

Missing authentication
    → 401

Authenticated but forbidden
    → Authorization system / 403

Rate limited
    → 429

Malformed request
    → 400

Expired flow
    → 400 / 401 depending protocol

Provider unavailable
    → 503

Internal failure
    → 500
```

## 95. Authentication vs Authorization

Crítico:

```text
Authentication Failure
    ≠
Authorization Denial
```

## 96. 401 vs 403

Conceptualmente:

```text
401
    Authentication missing/invalid

403
    Identity authenticated but not authorized
```

El sistema Authentication no deberá apropiarse de las decisiones del sistema Authorization.

## 97. Account Disabled

Puede representarse externamente de forma deliberadamente genérica.
No necesariamente:

- 403 ACCOUNT_DISABLED
- porque podría revelar existencia de la cuenta.

## 98. Enumeration Safe Mapping

Contexto anonymous:

- UNKNOWN_IDENTITY
- INVALID_PASSWORD
- ACCOUNT_DISABLED

pueden producir:
Unable to authenticate with the provided credentials.

## 99. Authenticated Context

Después de autenticación parcial/fuerte, puede exponerse más información cuando sea seguro.

## 100. AuthenticationDisclosurePolicy

interface AuthenticationFailureDisclosurePolicyInterface
{
public function disclose(
AuthenticationOutcome $outcome,
AuthenticationDisclosureContext $context
): PublicAuthenticationFailure;
}

## 101. Disclosure Levels

ANONYMOUS
PARTIALLY_AUTHENTICATED
AUTHENTICATED
DEVELOPER
SECURITY_OPERATOR

## 102. Anonymous

Mínimo detalle.

## 103. Partially Authenticated

Puede indicar:
Additional verification required.

## 104. Authenticated

Puede mostrar:

- Your session expired.
- Please authenticate again.

## 105. Developer

Puede mostrar:

```php
Authenticator=password
Failure=INVALID_PASSWORD
Policy=default-auth-v3
```

solo en entorno autorizado.

## 106. Security Operator

Puede acceder a diagnóstico completo redacted.

## 107. Public Error Codes

VoltStack deberá tener códigos estables:

- AUTHENTICATION_FAILED
- AUTHENTICATION_REQUIRED
- ADDITIONAL_VERIFICATION_REQUIRED
- AUTHENTICATION_EXPIRED
- AUTHENTICATION_RATE_LIMITED
- AUTHENTICATION_TEMPORARILY_UNAVAILABLE

## 108. Internal Error Codes

Más específicos:

- AUTH-CRED-001
- AUTH-FLOW-004
- AUTH-RISK-008
- AUTH-SESSION-003
- AUTH-OIDC-014
- AUTH-WEBAUTHN-006

## 109. Public != Diagnostic

Nunca deberán ser equivalentes automáticamente.

## 110. Diagnostic Reference

Cliente puede recibir:

```text
reference:
    ERR-7F83...
para soporte.
```

## 111. Reference != stack trace

Nunca exponer:

- file
- line
- class internals
- SQL
- secret
- stack

## 112. RFC 7807 / Problem Details

API podrá responder conceptualmente:

```php
{
  "type": "authentication-required",
  "title": "Authentication required",
  "status": 401,
  "code": "AUTHENTICATION_REQUIRED"
}
```

## 113. Invalid Credentials Response

{
"type": "authentication-failed",
"title": "Authentication failed",
"status": 401,
"code": "AUTHENTICATION_FAILED"
}
Sin decir:

- user exists
- password incorrect

## 114. MFA Challenge Response

{
"type": "authentication-challenge",
"status": 401,
"code": "ADDITIONAL_VERIFICATION_REQUIRED",
"challenge": "mfa"
}

## 115. SPA Response

VoltStack SPA Runtime podrá recibir:

- AUTH_CHALLENGE
- AUTH_REDIRECT
- AUTH_RETRY
- AUTH_SESSION_EXPIRED
- AUTH_FAILURE

## 116. SPA Protocol

Ejemplo conceptual:

```php
{
  "auth": {
    "state": "challenge",
    "challenge": "passkey",
    "flow": "opaque-reference"
  }
}
```

## 117. SPA Security

Nunca incluir:

- raw challenge internals
- server secrets
- policy internals
- risk score

salvo datos protocolariamente requeridos.

## 118. Redirect Response

Aplicación tradicional:

```text
Failure
    ↓
Redirect /login
    ↓
Flash safe error
```

## 119. Flash Data

Nunca almacenar:

- password
- OTP
- token

## 120. Old Input

Authentication forms deberán excluir campos sensibles de:

- old input
- session flash
- validation dumps

## 121. Password Validation Errors

Errores de formato durante registration/password change son distintos de Authentication Failure.

## 122. Authentication Form Validation

Antes del Authenticator:

```text
Request Validation
    ↓
Authentication Request
```

## 123. Malformed Input

Ejemplo:
password missing
Puede ser:

- 400 / validation response
- no necesariamente invalid credentials.

## 124. Enumeration Timing

Las respuestas deberán reducir diferencias observables entre:

- unknown identity
- known identity + invalid password

## 125. Constant-Time Principles

Cuando sea viable:

- dummy password verification
- uniform response structure
- similar processing path

## 126. No prometer tiempo idéntico

Red, DB, cache y runtime hacen imposible garantizar tiempos exactamente iguales.
El objetivo es reducir canales obvios.

## 127. Dummy Credential Verification

Si Identity no existe:

- perform dummy password hash verification
- cuando Password Authentication esté activo.

## 128. Dummy Hash

Deberá ser:

- valid hash using current/representative algorithm
- y preconfigurado/gestionado eficientemente.

## 129. No generar dummy hash por request

Sería costoso y podría facilitar DoS.

## 130. Error Message Localization

Los códigos internos serán independientes del idioma.
AUTHENTICATION_FAILED
podrá traducirse.

## 131. Translation Layer

Public Error Code
↓
Translator
↓
Localized Message

## 132. No translated strings in Core

Core produce códigos, no textos finales.

## 133. AuthenticationFailureHandler

Podrá existir como orchestrator:

```php
interface AuthenticationFailureHandlerInterface
{
    public function handle(
        AuthenticationOutcome $outcome,
        AuthenticationExecutionContext $context
    ): AuthenticationFailureHandlingResult;
}
```

## 134. Responsibilities

Podrá coordinar:

- classification
- security event emission
- audit
- metrics
- retry decision
- disclosure
- response mapping

sin convertirse en God Object.

## 135. Pipeline

AuthenticationOutcome
↓
FailureClassifier
↓
SecurityReaction
↓
Audit / Events / Metrics
↓
DisclosurePolicy
↓
ResponseMapper
↓
Transport Response

## 136. Failure Classifier

interface AuthenticationFailureClassifierInterface
{
public function classify(
AuthenticationOutcome $outcome
): AuthenticationFailureClassification;
}

## 137. Classification Dimensions

Podrán incluir:

- category
- severity
- retryability
- security relevance
- audit relevance
- user disclosure level
- transport semantics

## 138. Security Reaction

Un Failure puede provocar acciones adicionales.

```text
Ejemplo:
invalid password
    ↓
increment abuse signal
```

## 139. Compromised Credential

Puede provocar:

- deny
- revoke sessions
- require recovery
- emit security event
- notify user

## 140. Device Replay

Puede provocar:

- revoke device trust
- raise risk
- invalidate session
- audit

## 141. AuthenticationSecurityReaction

interface AuthenticationSecurityReactionInterface
{
public function react(
AuthenticationOutcome $outcome,
AuthenticationSecurityContext $context
): void;
}

## 142. Reaction Idempotency

Debe evitar repetir acciones destructivas si el mismo evento se procesa dos veces.

## 143. Security Reaction != Response

Ejemplo:

```php
revoke sessions
es reacción.
return 401
es response.
```

## 144. Failure Events

El sistema podrá emitir:

- AuthenticationFailed
- AuthenticationDenied
- AuthenticationChallengeRequired
- AuthenticationErrored
- AuthenticationFlowExpired
- AuthenticationRateLimited
- AuthenticationProviderUnavailable

## 145. Event Payloads

No deberán contener:

- password
- OTP
- token
- session secret
- recovery secret

## 146. Failure Audit

Debe integrarse con documento 24.

## 147. Audit Example

Event:
AUTHENTICATION_DENIED

Reason:
RISK_TOO_HIGH

Authenticator:
PASSWORD

Credential:
VERIFIED

Result:
DENIED

Risk:

```text
    HIGH
Sin almacenar password.
```

## 148. Failure Metrics

Ejemplos:

- auth_failure_total
- auth_denial_total
- auth_challenge_total
- auth_error_total

## 149. Labels seguros

method
category
reason_family
transport

## 150. No Identity labels

Nunca:

- email
- user_id
- IP
- session_id

como labels de métricas.

## 151. Failure Tracing

Span podrá registrar:

- auth.result=failure
- auth.failure.category=credential

## 152. No secret trace attributes

Obligatorio.

## 153. Error Severity

Ejemplo:

```text
INVALID_PASSWORD
    INFO

RATE_LIMITED
    NOTICE/WARNING

RISK_DENIED
    WARNING

SESSION_STORE_UNAVAILABLE
    ERROR

AUTHENTICATION_INVARIANT_VIOLATION
    CRITICAL
```

## 154. Severity != HTTP Status

Un:

- 401
- puede ser INFO.

Un:

- 503
- puede ser ERROR.

## 155. Authentication Failure State Machine

PENDING
│
├──> SUCCESS
│
├──> FAILURE
│
├──> DENIAL
│
├──> CHALLENGE ──> PENDING
│
└──> ERROR

## 156. Challenge continuation

CHALLENGE
↓
new evidence
↓
PENDING

## 157. Terminal Outcomes

Normalmente:

- SUCCESS
- FAILURE
- DENIAL
- ERROR

son terminales para ese Attempt.

## 158. Flow puede continuar

Un nuevo Attempt puede pertenecer al mismo Flow.

## 159. Attempt vs Flow Failure

Ejemplo:

```text
Attempt 1:
    invalid OTP
```

Flow:

```text
    still active
No confundir.
```

## 160. Failure Budget per Flow

Puede limitar:

- OTP attempts
- Passkey failures
- Recovery evidence attempts

## 161. Exceeded Attempts

Puede transformar:

```text
FAILURE
    ↓
DENIAL
```

Ejemplo:
MFA_ATTEMPTS_EXCEEDED

## 162. Rate Limit Response

Deberá poder incluir:

- Retry-After
- cuando sea seguro.

## 163. Retry-After Privacy

No revelar detalles internos del bucket/rate limiter.

## 164. Brute Force Response

Puede seguir devolviendo:

- generic authentication failure
- aunque internamente exista bloqueo.

Esto reduce información al atacante.

## 165. Credential Stuffing

Igual.

## 166. CAPTCHA / Additional Challenge

Si VoltStack integra mecanismos anti-abuse futuros:

- ABUSE_CHALLENGE_REQUIRED
- puede ser un Challenge, no Failure.

## 167. Provider Failure Strategy

Ejemplo OIDC:

```text
Provider unavailable
        ↓
```

Can alternative authenticator be used?
│
┌───┴───┐
▼       ▼
YES      NO
│       │
▼       ▼
Fallback   Error Response

## 168. Authenticator Fallback

Debe ser policy-driven.

## 169. No silent downgrade

Si Passkey es requerido:

- Passkey provider unavailable
- no deberá degradar silenciosamente a password.

## 170. Assurance Preservation

Fallback solo es válido si mantiene:

- required assurance
- required factor properties
- security policy

## 171. Downgrade Attack Resistance

Toda fallback decision deberá verificar:

- assurance
- factor class
- risk policy
- tenant policy

## 172. Failure During Step-Up

Si password ya fue verificado pero Passkey falla:

- Authentication no está completa
- No crear sesión final accidentalmente.

## 173. Partial Authentication

Podrá existir:
PartialAuthenticationContext

## 174. Partial Context Security

Debe tener:

- short TTL
- limited capabilities
- flow binding
- replay protection

## 175. Partial Authentication != authenticated session

No deberá autorizar recursos normales.

## 176. Failure During Session Creation

Caso importante:

- Credentials VERIFIED
- Identity ELIGIBLE
- MFA VERIFIED
- Session Store FAILED

Resultado:
ERROR
No:
SUCCESS

## 177. Success Atomicity

Authentication Success deberá emitirse según el punto definido por lifecycle.
No antes de completar operaciones críticas.

## 178. Failure During Audit

Dependerá del AuditFailurePolicy definido en documento 24.

## 179. Failure During Event Listener

Dependerá de criticidad.

- critical synchronous listener
- non-critical listener
- async listener

no deberán tener el mismo comportamiento.

## 180. Listener Failure Isolation

Un listener de analytics no debería impedir login.

## 181. Critical Security Listener

Un listener que aplica una revocación requerida puede ser crítico.

## 182. Failure Policy Metadata

Listeners/extensions podrán declarar:

- critical
- failure_mode
- timeout
- retryability

## 183. Extension Exception Isolation

Plugins no confiables no deberán romper arbitrariamente todo Authentication.

## 184. Extension Boundary

Extension
↓
Extension Exception Boundary
↓
Extension Failure Policy

## 185. Custom Authenticators

Deben retornar Outcomes definidos.
No arbitrary arrays.

## 186. Invalid Authenticator Result

Si un custom Authenticator devuelve un tipo imposible:
AUTHENTICATOR_CONTRACT_VIOLATION

## 187. Contract Violation

Debe considerarse:
PROGRAMMING ERROR

## 188. Error Recovery

Algunos errores podrán producir una Recovery Action:

- RETRY
- RESTART_FLOW
- REAUTHENTICATE
- USE_ALTERNATIVE_FACTOR
- CONTACT_SUPPORT
- WAIT

## 189. AuthenticationRecoveryAction

No confundir con Account Recovery.
Es una instrucción de recuperación del error.

## 190. Example

SESSION_EXPIRED
↓
REAUTHENTICATE

## 191. Example

OIDC_PROVIDER_UNAVAILABLE
↓
USE_ALTERNATIVE_FACTOR
si policy lo permite.

## 192. Example

RATE_LIMITED
↓
WAIT

## 193. Safe Client Actions

API podrá devolver:

```text
action:
    reauthenticate
sin revelar internals.
```

## 194. Response Contracts

Frontend no deberá parsear textos.
No:
if (error.message === 'Your session expired') {}

## 195. Correct

if (error.code === 'AUTHENTICATION_EXPIRED') {}

## 196. Stable Public Contract

Public codes deberán ser versionables y estables.

## 197. Internal Codes

Podrán evolucionar más rápidamente.

## 198. Versioning

SPA/API protocol podrá incluir:
authentication response schema version

## 199. Security Headers

Responses de Authentication podrán requerir:

- Cache-Control: no-store
- Pragma: no-cache

según tipo de response.

## 200. Sensitive Responses

Especialmente:

- login
- MFA
- recovery
- OAuth callback
- session refresh

## 201. WWW-Authenticate

Bearer/API authentication podrá utilizar:

- WWW-Authenticate
- conforme al protocolo correspondiente.

## 202. No sensitive details in header

No incluir raw token/reasons sensibles.

## 203. Redirect Security

Redirects deberán validar:

- target
- origin
- allowed host
- scheme

## 204. Open Redirect Prevention

Nunca:

- /login?next=<https://attacker.example>
- sin validación.

## 205. AuthenticationEntryPoint

Podrá decidir respuesta cuando Authentication sea requerida.

## 206. Entry Point != Failure Handler

Entry Point responde a:
authentication required
Failure Handler responde a:
authentication attempt failed

## 207. AccessDenied vs AuthenticationRequired

Authorization puede determinar:

- resource requires authenticated identity
- y delegar al Authentication Entry Point.

## 208. Clean subsystem boundary

Authorization
↓
Unauthenticated?
↓
Authentication Entry Point

Authenticated but forbidden?
↓
Authorization Denial

## 209. Login Failure Handler

Para flows HTML podrá existir:

- LoginFailureHandler
- como adapter especializado.

## 210. Logout Failure

Logout también puede fallar técnicamente.
Ejemplo:
session revocation backend unavailable

## 211. Logout Semantics

Debe distinguir:

- local client cleanup
- server-side revocation
- global revocation

## 212. Logout Response Safety

Incluso si revocación remota falla, la aplicación puede necesitar limpiar cookie local y alertar internamente.
Policy-dependent.

## 213. Global Logout

Si solo 7 de 8 sesiones se revocan:

- PARTIAL_FAILURE
- puede ser necesario.

## 214. Partial Failure

VoltStack deberá soportar:

- PARTIAL_SUCCESS
- PARTIAL_FAILURE

para operaciones compuestas de gestión Authentication.

## 215. Login normal

No deberá terminar en partial success.

- Authentication final es:
- success
- or not success

## 216. Administrative Operations

Sí pueden ser parciales.

## 217. AuthenticationOperationResult

Para operaciones administrativas:

```php
final readonly class AuthenticationOperationResult
{
    public function __construct(
        public AuthenticationOperationStatus $status,
        public array $completed,
        public array $failed,
    ) {}
}
```

## 218. Timeouts

Cada provider externo deberá tener timeout.

## 219. No unbounded wait

Authentication no debe bloquear indefinidamente.

## 220. Timeout Classification

TIMEOUT
↓
ERROR
↓
retry/fallback/fail-closed
según policy.

## 221. Overall Authentication Deadline

Además de timeout por dependencia, podrá existir:
AuthenticationExecutionDeadline

## 222. Deadline propagation

Providers reciben remaining budget.

## 223. Cancellation

Si request se cancela, trabajo costoso podrá cancelarse cuando runtime lo permita.

## 224. Cancellation != Failure

Puede ser:

- CANCELLED
- como estado operacional.

## 225. AuthenticationOutcome Extended

Modelo completo puede ser:

- SUCCESS
- FAILURE
- DENIAL
- CHALLENGE
- ERROR
- CANCELLED

## 226. Expiration

Puede representarse como Failure/Denial especializado según objeto.
Pero los reason codes deberán conservar:
EXPIRED

## 227. Security Error Normalization

Externamente varios errores internos pueden mapearse al mismo response.

## 228. Example

Internamente:

- Risk provider timeout
- Session provider timeout
- Identity provider timeout

Externamente:
AUTHENTICATION_TEMPORARILY_UNAVAILABLE

## 229. No infrastructure topology leakage

Cliente no necesita saber:

- Redis failed
- PostgreSQL failed
- Datadog timeout

## 230. Debug Mode

En desarrollo podrá exponer mayor información, pero:
never secrets

## 231. APP_DEBUG

Incluso con debug:

- password
- token
- OTP
- cookie
- client secret
- nunca deberán aparecer.

## 232. Exception Rendering

Authentication-specific renderer deberá ejecutar redaction antes de cualquier debug representation.

## 233. AuthenticationExceptionRenderer

interface AuthenticationExceptionRendererInterface
{
public function render(
AuthenticationError $error,
AuthenticationResponseContext $context
): AuthenticationResponse;
}

## 234. Generic Framework Exception Handler

Podrá delegar errores Auth al renderer especializado.

## 235. Integration

Global Exception Handler
↓
Authentication Exception?
↓
Authentication Exception Translator
↓
Authentication Error
↓
Authentication Response Mapper

## 236. Never render vendor exception directly

Obligatorio.

## 237. Correlation

Todo Error técnico importante deberá tener:

- RequestId
- TraceId
- AuthenticationFlowId
- DiagnosticReference
- cuando estén disponibles.

## 238. Public correlation

Normalmente solo:

- DiagnosticReference
- deberá exponerse.

## 239. Failure Observability

Integración con documento 24:

```text
Outcome
   ├── Audit
   ├── Log
   ├── Metric
   ├── Trace
   └── Explainability
```

## 240. Failure Reason Codes

Los mismos reason codes deberán poder alimentar:

- Risk
- Audit
- Explainability
- Testing
- Security Operations

sin acoplarse a mensajes UI.

## 241. AuthenticationFailureReason

Value Object:

```php
final readonly class AuthenticationFailureReason
{
    public function __construct(
        public string $code,
        public AuthenticationFailureCategory $category,
        public AuthenticationSecuritySeverity $severity,
    ) {}
}
```

## 242. No arbitrary exceptions as reason

No:
'reason' => $e->getMessage()

## 243. Safe reason normalization

Sí:
'reason' => AuthenticationReasonCode::ProviderUnavailable

## 244. Authentication Failure Registry

Puede existir:
AuthenticationFailureRegistry
para registrar:

- code
- category
- default severity
- retryability
- disclosure policy

## 245. Registry compiled

En producción podrá compilarse/cachearse.

## 246. Extension Codes

Custom packages deberán usar namespace.
Ejemplo:
vendor.package.auth.custom_failure

## 247. Collision Protection

Registry rechazará códigos duplicados incompatibles.

## 248. Authentication Error Catalog

Documentación/tooling podrá listar todos los códigos registrados.

## 249. Developer Experience

CLI futuro:
php volt auth:errors
podría mostrar:

- Code
- Category
- Public mapping
- Retryable
- Severity

## 250. Error Documentation

Cada código deberá definir:

- meaning
- origin
- security relevance
- retryability
- public mapping
- expected reaction

## 251. Testing — Failure vs Error

Debe verificarse:

```text
invalid password
    → FAILURE

database unavailable
    → ERROR
```

## 252. Testing — Denial

valid password + disabled account
→ DENIAL

## 253. Testing — Challenge

valid password + MFA required
→ CHALLENGE

## 254. Testing — Enumeration

Comparar respuestas para:

- unknown identity
- known identity + invalid password
- disabled identity

La respuesta pública deberá ser consistente según policy.

## 255. Testing — Timing

No exigir tiempos matemáticamente idénticos.
Sí verificar que se ejecute dummy verification cuando corresponda.

## 256. Testing — Exception Translation

Simular:

- PDOException
- RedisException
- HTTP timeout

y verificar traducción correcta.

## 257. Testing — Vendor Isolation

Ninguna excepción vendor deberá llegar al transport response.

## 258. Testing — Secret Redaction

Usar canary:
SECRET_AUTH_CANARY
y comprobar ausencia en:

- response
- logs
- audit
- trace
- exception output
- events

## 259. Testing — Retry

Verificar:
retryable provider timeout
pero:

- invalid password
- no se reintenta.

## 260. Testing — Retry Exhaustion

Al superar budget:

- AUTHENTICATION_TEMPORARILY_UNAVAILABLE
- o resultado configurado.

## 261. Testing — Circuit Breaker

Provider persistentemente caído deberá abrir circuito.

## 262. Testing — Fail Closed

Security dependency crítica indisponible:
authentication not finalized

## 263. Testing — No Silent Downgrade

Passkey requerida + provider indisponible:

- password-only success
- debe ser imposible.

## 264. Testing — Session Creation Failure

Credenciales válidas + session backend caído:

- ERROR
- No emitir final success.

## 265. Testing — SPA Mapping

Cada Outcome debe generar protocolo SPA válido.

## 266. Testing — HTTP Mapping

Verificar:

- 401
- 429
- 400
- 500
- 503
- según caso.

## 267. Testing — Redirect

Open redirect payload deberá rechazarse.

## 268. Testing — Flash

Passwords/OTP nunca deberán almacenarse como old input.

## 269. Testing — Localization

Cambiar idioma no cambia:

- public error code
- solo mensaje.

## 270. Testing — Flow Expiration

Flow expirado no puede continuar.

## 271. Testing — Replay

Flow consumido no puede reutilizarse.

## 272. Testing — Partial Authentication

Partial context no permite acceder a recursos autenticados normales.

## 273. Testing — Fiber Concurrency

Failure de Request A no deberá alterar Response de Request B.

## 274. Testing — FrankenPHP

Después de:
Authentication failure for Alice
el worker reutilizado no deberá conservar:

- Alice Identity
- Failure reason
- Flow
- Trace
- Partial Authentication
- para Bob.

## 275. Testing — Property Based

Útil para:

- exception mapping
- public disclosure
- failure classification
- status mapping
- redaction

## 276. Testing — Fuzzing

Especialmente:

- OAuth callback
- OIDC tokens
- WebAuthn payloads
- Bearer headers
- Continuation payloads
- Recovery inputs

## 277. Security Invariants — Outcomes

AUTH-FAIL-01
Invalid credentials are domain failures, not infrastructure exceptions.
AUTH-FAIL-02
Challenges are not Authentication failures.

- AUTH-FAIL-03
- Denials remain distinguishable from credential failures internally.
- AUTH-FAIL-04

Technical errors remain distinguishable from security denials.
AUTH-FAIL-05
Authentication cannot finalize from an Error outcome.

## 278. Security Invariants — Exceptions

AUTH-EXC-01
Vendor exceptions never cross the Authentication Core boundary.
AUTH-EXC-02
Expected credential failures do not require exceptions for control flow.
AUTH-EXC-03
Unknown exceptions are normalized before public rendering.

- AUTH-EXC-04
- Stack traces never become production Authentication responses.
- AUTH-EXC-05

Exception rendering never exposes Authentication secrets.

## 279. Security Invariants — Disclosure

AUTH-DISC-01
Internal failure codes and public failure codes are separate.
AUTH-DISC-02
Anonymous responses resist account enumeration.

- AUTH-DISC-03
- Risk intelligence is not exposed to anonymous clients.
- AUTH-DISC-04

Diagnostic references do not reveal internal topology.
AUTH-DISC-05
Localization does not alter security semantics.

## 280. Security Invariants — Response

AUTH-RESP-01
Domain Authentication components do not construct transport responses.
AUTH-RESP-02
Transport responses derive from normalized Authentication Outcomes.
AUTH-RESP-03
Sensitive Authentication responses are non-cacheable when required.
AUTH-RESP-04
Redirect targets are validated.
AUTH-RESP-05
Authentication failures are not confused with Authorization denials.

## 281. Security Invariants — Retry

AUTH-RETRY-01
Invalid credentials are never automatically retried.

- AUTH-RETRY-02
- Retries have bounded budgets.
- AUTH-RETRY-03

Fallback cannot reduce required assurance.

- AUTH-RETRY-04
- Fail-open behavior requires explicit policy.
- AUTH-RETRY-05

Security-critical dependency failures default toward safe behavior.

## 282. Security Invariants — Flow

AUTH-FLOW-FAIL-01
Expired flows cannot continue.

- AUTH-FLOW-FAIL-02
- Completed flows cannot be replayed.
- AUTH-FLOW-FAIL-03

Attempt failure does not automatically invalidate the entire Flow unless policy requires it.
AUTH-FLOW-FAIL-04
Partial Authentication does not grant full authenticated capabilities.
AUTH-FLOW-FAIL-05
Step-Up failure cannot accidentally finalize the original Authentication.

## 283. Security Invariants — Runtime

AUTH-FAIL-RT-01
Failure context is request/fiber scoped.

- AUTH-FAIL-RT-02
- Partial Authentication state is request/flow scoped.
- AUTH-FAIL-RT-03

Long-running workers clear failure state after each request.

- AUTH-FAIL-RT-04
- Concurrent Authentication flows cannot share mutable exception context.
- AUTH-FAIL-RT-05

Diagnostic context cannot leak across tenants.

## 284. Anti-pattern — Exception for every bad password

throw new AuthenticationException('Wrong password');
Evitar como modelo central.

## 285. Anti-pattern — Catch everything and return 401

catch (\Throwable) {
return response('', 401);
}
No.

## 286. Anti-pattern — Return 500 for invalid credentials

No.

## 287. Anti-pattern — Return 401 for database outage

No.

## 288. Anti-pattern — Return 403 for every Authentication denial

No necesariamente.

## 289. Anti-pattern — Reveal account status

Your account exists but is disabled.
a un cliente anonymous.

## 290. Anti-pattern — Reveal password correctness

Password correct, but MFA failed.
cuando el contexto no permite esa información.

## 291. Anti-pattern — Retry invalid password automatically

No.

## 292. Anti-pattern — Silent factor downgrade

Passkey failed → use password → success
cuando Passkey era requisito.

## 293. Anti-pattern — Vendor exception to frontend

Nunca.

## 294. Anti-pattern — Exception message as API contract

No.

## 295. Anti-pattern — Parse UI error strings

No.

## 296. Anti-pattern — Store password in old input

Nunca.

## 297. Anti-pattern — Authentication success before session establishment

No cuando la sesión sea parte obligatoria del lifecycle.

## 298. Anti-pattern — Risk provider exception means LOW risk

Crítico.

## 299. Anti-pattern — Static current exception

Peligroso bajo FrankenPHP.

## 300. Componentes principales

AuthenticationOutcome
AuthenticationOutcomeType

AuthenticationSuccess
AuthenticationFailure
AuthenticationDenial
AuthenticationChallenge
AuthenticationError

AuthenticationFailureCode
AuthenticationDenialCode
AuthenticationErrorCode
AuthenticationReasonCode

AuthenticationFailureClassifier
AuthenticationFailureHandler
AuthenticationFailureBoundary

AuthenticationExceptionTranslator
AuthenticationRetryPolicy
AuthenticationDependencyFailurePolicy

AuthenticationFailureDisclosurePolicy
AuthenticationResponseMapper
AuthenticationExceptionRenderer

AuthenticationSecurityReaction
AuthenticationRecoveryAction

## 301. Componentes de transporte

AuthenticationResponse
RedirectAuthenticationResponse
JsonAuthenticationResponse
ProblemDetailsAuthenticationResponse
SpaAuthenticationResponse
ChallengeAuthenticationResponse

AuthenticationEntryPoint
LoginFailureHandler
AuthenticationHttpStatusMapper

## 302. Componentes de diagnóstico

AuthenticationDiagnosticReference
AuthenticationFailureClassification
AuthenticationErrorCatalog
AuthenticationFailureRegistry
AuthenticationFailureContext
AuthenticationErrorContext

## 303. Namespace sugerido

VoltStack\Quantum\Auth\Failure
VoltStack\Quantum\Auth\Failure\Contracts
VoltStack\Quantum\Auth\Failure\Outcome
VoltStack\Quantum\Auth\Failure\Exception
VoltStack\Quantum\Auth\Failure\Classification
VoltStack\Quantum\Auth\Failure\Disclosure
VoltStack\Quantum\Auth\Failure\Response
VoltStack\Quantum\Auth\Failure\Retry
VoltStack\Quantum\Auth\Failure\Security
VoltStack\Quantum\Auth\Failure\Diagnostics

## 304. Estructura sugerida

src/Quantum/Auth/Failure/
├── Contracts/
│   ├── AuthenticationOutcomeInterface.php
│   ├── AuthenticationFailureHandlerInterface.php
│   ├── AuthenticationFailureClassifierInterface.php
│   ├── AuthenticationExceptionTranslatorInterface.php
│   ├── AuthenticationRetryPolicyInterface.php
│   ├── AuthenticationFailureDisclosurePolicyInterface.php
│   └── AuthenticationResponseMapperInterface.php
│
├── Outcome/
│   ├── AuthenticationSuccess.php
│   ├── AuthenticationFailure.php
│   ├── AuthenticationDenial.php
│   ├── AuthenticationChallenge.php
│   ├── AuthenticationError.php
│   └── AuthenticationOutcomeType.php
│
├── Classification/
│   ├── AuthenticationFailureCode.php
│   ├── AuthenticationDenialCode.php
│   ├── AuthenticationErrorCode.php
│   ├── AuthenticationReasonCode.php
│   ├── AuthenticationFailureCategory.php
│   └── AuthenticationFailureClassification.php
│
├── Exception/
│   ├── AuthenticationException.php
│   ├── AuthenticationInfrastructureException.php
│   ├── AuthenticationProviderException.php
│   ├── AuthenticationConfigurationException.php
│   ├── AuthenticationProtocolException.php
│   ├── AuthenticationCryptographicException.php
│   ├── AuthenticationTimeoutException.php
│   └── AuthenticationInvariantViolationException.php
│
├── Retry/
│   ├── AuthenticationRetryPolicy.php
│   ├── AuthenticationRetryDecision.php
│   ├── AuthenticationRetryBudget.php
│   └── AuthenticationDependencyFailureMode.php
│
├── Disclosure/
│   ├── AuthenticationFailureDisclosurePolicy.php
│   ├── AuthenticationDisclosureContext.php
│   └── PublicAuthenticationFailure.php
│
├── Response/
│   ├── AuthenticationResponseMapper.php
│   ├── AuthenticationExceptionRenderer.php
│   ├── AuthenticationHttpStatusMapper.php
│   ├── JsonAuthenticationResponse.php
│   ├── SpaAuthenticationResponse.php
│   └── RedirectAuthenticationResponse.php
│
├── Security/
│   ├── AuthenticationSecurityReaction.php
│   └── AuthenticationRecoveryAction.php
│
├── Diagnostics/
│   ├── AuthenticationDiagnosticReference.php
│   ├── AuthenticationErrorCatalog.php
│   └── AuthenticationFailureRegistry.php
│
├── AuthenticationFailureBoundary.php
└── AuthenticationFailureHandler.php

## 305. Flujo completo

AUTHENTICATION PIPELINE
│
▼
AuthenticationOutcome
│
┌──────────────────┼──────────────────┐
│                  │                  │
▼                  ▼                  ▼
CLASSIFY          SECURITY REACTION   OBSERVABILITY
│                  │                  │
│                  │          ┌───────┼───────┐
│                  │          ▼       ▼       ▼
│                  │        Audit   Metric   Trace
│                  │
└──────────────────┼──────────────────┘
│
▼
DISCLOSURE POLICY
│
▼
PUBLIC OUTCOME
│
▼
RESPONSE MAPPER
│
┌─────────────┼─────────────┐
▼             ▼             ▼
HTML           SPA           API

## 306. Exception Flow

External Dependency
│
▼
Vendor Exception
│
▼
Adapter Boundary
│
▼
Authentication Exception
│
▼
Exception Translator
│
▼
AuthenticationError
│
▼
Failure Handling Pipeline

## 307. Credential Failure Flow

PasswordAuthenticator
↓
Password INVALID
↓
AuthenticationFailure
↓
INVALID_CREDENTIALS
↓
Abuse Signal
↓
Audit / Metrics
↓
Disclosure Policy
↓
Generic Public Failure

## 308. Denial Flow

Password VALID
↓
Identity ACTIVE
↓
Risk HIGH
↓
Policy DENY
↓
AuthenticationDenial
↓
RISK_TOO_HIGH
↓
Security Event / Audit
↓
Safe Public Response

## 309. Challenge Flow

Password VALID
↓
Assurance INSUFFICIENT
↓
AuthenticationChallenge
↓
PASSKEY_REQUIRED
↓
Continuation Created
↓
Challenge Response
↓
Passkey Evidence
↓
Authentication Flow Continues

## 310. Infrastructure Error Flow

Password VALID
↓
Risk Provider
↓
TIMEOUT
↓
AuthenticationProviderException
↓
Exception Translator
↓
AuthenticationError
↓
Failure Policy
↓
Retry / Degraded / Fail Closed

## 311. Relación con Laravel

Laravel proporciona ideas como:

```php
AuthenticationException
unauthenticated()
Guards
Middleware
Exception Handler
RateLimiter
Redirect responses
Sanctum/Passport token responses
```

Su ergonomía es excelente, pero VoltStack deberá evitar centralizar toda la semántica de Authentication alrededor de una única AuthenticationException.

## 312. Relación con Symfony

Symfony aporta conceptos especialmente valiosos:

- AuthenticationException hierarchy
- AuthenticationEntryPointInterface
- AuthenticationFailureHandlerInterface
- AccessDeniedHandler
- Passport/Authenticator model
- Security events
- Exception normalization

VoltStack tomará especialmente la separación conceptual entre:

- Authenticator
- Entry Point
- Failure Handler
- Security Exception

pero la ampliará con Outcomes explícitos.

## 313. Diferenciador VoltStack

VoltStack utilizará una semántica más rica:

```text
                    AUTHENTICATION OUTCOME
                            │
       ┌──────────┬─────────┼─────────┬─────────┐
       ▼          ▼         ▼         ▼         ▼
    SUCCESS    FAILURE    DENIAL   CHALLENGE   ERROR
                  │         │         │          │
                  ▼         ▼         ▼          ▼
             credential   policy   continue   technical
              rejected    reject     flow      problem
```

Esto permite que el framework sepa exactamente:

- qué ocurrió
- por qué ocurrió
- si es esperado
- si es sospechoso
- si debe auditarse
- si puede reintentarse

si el flow puede continuar
si debe crearse un challenge
qué puede conocer el cliente
qué HTTP response corresponde
qué reacción de seguridad ejecutar

## 314. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Authentication Outcomes instead of exception-driven control flow

## 2. SUCCESS, FAILURE, DENIAL, CHALLENGE and ERROR remain distinct

## 3. Invalid credentials are domain failures

## 4. Security policy rejection is a denial

## 5. Additional evidence requirements are challenges

## 6. Infrastructure failures are errors

## 7. Vendor exceptions are translated at boundaries

## 8. Internal and public error codes are separated

## 9. Disclosure is context-sensitive

## 10. Anonymous responses resist account enumeration

## 11. Transport mapping is separate from domain semantics

## 12. Authentication and Authorization denials remain separate

## 13. Retry behavior is explicit and bounded

## 14. Fail-open behavior is never implicit

## 15. Authentication fallback cannot reduce required assurance

## 16. Partial Authentication never grants full authenticated capabilities

## 17. Security reactions are separate from transport responses

## 18. Error handling integrates with Audit, Metrics, Tracing and Explainability

## 19. Secrets never appear in errors, exceptions or responses

## 20. Failure state is isolated under FrankenPHP and concurrent runtimes

## 21. Criterios de aceptación

El subsistema será considerado completo cuando:
22. represente Success;
23. represente Failure;
24. represente Denial;
25. represente Challenge;
26. represente Error;
27. diferencie Failure y Exception;
28. diferencie Failure y Denial;
29. diferencie Challenge y Failure;
30. diferencie Authentication y Authorization denial;
31. soporte Credential failures;
32. soporte Identity failures;
33. soporte Flow failures;
34. soporte Session failures;
35. soporte Token failures;
36. soporte MFA failures;
37. soporte Passkey failures;
38. soporte Federation failures;
39. soporte Recovery failures;
40. soporte Infrastructure errors;
41. soporte Provider errors;
42. soporte Configuration errors;
43. soporte Protocol errors;
44. soporte Cryptographic errors;
45. traduzca vendor exceptions;
46. soporte unknown exception normalization;
47. soporte stable public codes;
48. soporte detailed internal codes;
49. soporte diagnostic references;
50. soporte disclosure policies;
51. resista account enumeration;
52. soporte dummy credential verification;
53. soporte HTML responses;
54. soporte JSON/API responses;
55. soporte SPA responses;
56. soporte Problem Details;
57. soporte Authentication Entry Points;
58. soporte retry policies;
59. soporte retry budgets;
60. soporte backoff;
61. soporte circuit breakers;
62. soporte fail-closed;
63. soporte explicit fail-open;
64. soporte degraded operation;
65. impida silent security downgrade;
66. preserve assurance requirements;
67. soporte security reactions;
68. soporte safe recovery actions;
69. soporte flow continuation;
70. soporte partial Authentication;
71. soporte flow expiration;
72. soporte replay detection;
73. soporte rate-limit responses;
74. soporte localization;
75. soporte error catalog;
76. soporte extension failure codes;
77. integre Audit;
78. integre Logging;
79. integre Metrics;
80. integre Tracing;
81. integre Explainability;
82. proteja secretos;
83. proteja PII;
84. sea tenant-aware;
85. sea fiber-safe;
86. sea seguro bajo FrankenPHP.
87. Regla arquitectónica final

VoltStack deberá preservar siempre:

```text
                 AUTHENTICATION EXECUTION
                           │
                           ▼
                  NORMALIZED OUTCOME
                           │
       ┌─────────┬─────────┼─────────┬─────────┐
       ▼         ▼         ▼         ▼         ▼
    SUCCESS   FAILURE    DENIAL   CHALLENGE   ERROR
       │         │         │         │         │
       └─────────┴─────────┼─────────┴─────────┘
                           │
                           ▼
                 SECURITY CLASSIFICATION
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
            REACTION   OBSERVABILITY DISCLOSURE
                                      │
                                      ▼
                                SAFE OUTCOME
                                      │
                                      ▼
                               RESPONSE MAPPER
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                        HTML         SPA          API
```

La primera regla será:
Una credencial incorrecta es un resultado esperado de Authentication; no es por sí misma una excepción del sistema.

La segunda:
Una identidad correctamente identificada y una credencial correctamente verificadas todavía pueden terminar en DENIAL si las políticas de seguridad, riesgo, dispositivo, tenant o assurance impiden completar Authentication.

La tercera:
Un CHALLENGE significa que Authentication continúa; nunca deberá confundirse con FAILURE.

La cuarta:
Los errores técnicos deberán normalizarse y mantenerse separados de los fallos de credenciales para evitar decisiones incorrectas de seguridad, métricas, rate limiting y auditoría.

La quinta:
Ninguna excepción, mensaje de error o respuesta pública deberá revelar passwords, tokens, OTPs, secretos de sesión, Recovery Credentials, información sensible de Risk o detalles que faciliten account enumeration.

La sexta:
VoltStack nunca deberá degradar silenciosamente el nivel de seguridad requerido para convertir un fallo técnico en un login exitoso.

Siguiente documento recomendado
La secuencia natural continúa con:
`26_AUTHENTICATION_TESTING_VERIFICATION_SECURITY_ASSURANCE_AND_CONFORMANCE_SYSTEM.md`
Aquí deberíamos concentrar todo el modelo formal de pruebas del sistema Authentication, incluyendo:

- Unit Testing
- Integration Testing
- Authentication Flow Testing
- Authenticator Contract Testing
- Password Testing
- Session Testing
- Token Testing
- MFA Testing
- Passkey/WebAuthn Testing
- OAuth2/OIDC Testing
- Recovery Testing
- Risk Engine Testing
- Device Trust Testing
- Abuse Protection Testing
- Failure/Denial Testing
- Multi-Tenant Isolation Testing
- Security Invariant Testing
- Property-Based Testing
- Fuzz Testing
- Mutation Testing
- Concurrency Testing

FrankenPHP Worker Reuse Testing
Timing/Enumeration Testing
Replay Testing
Downgrade Resistance Testing
Secret Leakage Testing
Chaos/Fault Injection Testing
Protocol Conformance Testing
Security Regression Suites
Authentication Test Harness
Fake Authenticators
Fake Identity Providers
Fake Risk Providers
Authentication Flow Simulator
Compliance/Conformance Profiles
Este documento sería especialmente importante porque permitiría convertir todas las invariantes AUTH-* que hemos ido definiendo desde 00 hasta 25 en contratos verificables automáticamente, en vez de dejarlas únicamente como reglas arquitectónicas.
