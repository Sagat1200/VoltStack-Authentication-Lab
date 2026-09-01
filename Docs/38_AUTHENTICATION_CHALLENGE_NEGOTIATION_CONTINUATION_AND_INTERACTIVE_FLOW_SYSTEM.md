# VoltStack Authentication System

## 38 — Authentication Challenge Negotiation, Continuation and Interactive Flow System

- **Archivo:** `38_AUTHENTICATION_CHALLENGE_NEGOTIATION_CONTINUATION_AND_INTERACTIVE_FLOW_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Nivel:** Core / Security-Critical / Authentication Orchestration
- **Dependencias principales:** documentos 01–37, especialmente 03, 04, 06, 07, 08, 15, 16, 17, 18, 20, 22, 23, 25, 27, 29, 32, 34, 36 y 37.

---

## 1. Propósito

Este documento define el subsistema responsable de convertir una necesidad de Authentication en un flujo interactivo seguro, continuable y negociable.
Su responsabilidad principal será resolver:

```text
Current Authentication Context
            │
            ▼
Authentication Requirement
            │
            ▼
Requirement not satisfied
            │
            ▼
Challenge Negotiation
            │
            ▼
Challenge Presentation
            │
            ▼
```

User / Principal Interaction
│
▼
Evidence
│
▼
Authentication Context Re-evaluation
│
┌────┼─────┐
▼    ▼     ▼
SUCCESS NEXT FAILURE
VoltStack no deberá implementar login, MFA, reauthentication, federation, passkeys y recovery como flujos independientes sin una arquitectura común.
Todos deberán utilizar un modelo compartido de:

- Authentication Transaction
- Challenge
- Negotiation
- Continuation
- Interaction
- Evidence
- Transition
- Completion

## 2. Problema arquitectónico

Los sistemas tradicionales suelen implementar:

- /login
- /mfa
- /passkey
- /oauth/callback
- /password/confirm
- /recovery
- como controladores separados.

Esto genera lógica duplicada para:

- redirects
- session state
- error handling
- expiration
- continuation
- CSRF
- retry
- step-up
- fallback
- SPA responses
- API responses

VoltStack deberá evitar este modelo fragmentado.

## 3. Principio fundamental

Authentication es una transacción compuesta por estados y challenges, no una colección de formularios independientes.

## 4. Authentication Transaction

Toda interacción Authentication compleja deberá pertenecer a una:
AuthenticationTransaction

## 5. Ejemplo

Transaction
│
├── Identify Principal
│
├── Password Challenge
│
├── Risk Evaluation
│
├── Passkey Step-Up
│
└── Complete Authentication

## 6. AuthenticationTransaction

final readonly class AuthenticationTransaction
{
public function __construct(
public AuthenticationTransactionId $id,
public AuthenticationFlowType $type,
public AuthenticationTransactionStatus $status,
public AuthenticationContextSnapshot $context,
public AuthenticationRequirementSet $requirements,
public AuthenticationContinuation $continuation,
public DateTimeImmutable $createdAt,
public DateTimeImmutable $expiresAt,
) {}
}

## 7. Transaction ID

Debe ser:

- opaque
- unguessable
- high entropy
- non-semantic

Nunca:

- /auth/transaction/123
- si 123 permite enumeración significativa.

## 8. Authentication Flow Types

enum AuthenticationFlowType: string
{
case Login = 'login';
case Reauthentication = 'reauthentication';
case StepUp = 'step_up';
case Recovery = 'recovery';
case Enrollment = 'enrollment';
case Linking = 'linking';
case Privileged = 'privileged';
case BreakGlass = 'break_glass';
case Federation = 'federation';
}

## 9. Transaction State Machine

Modelo base:

```text
CREATED
   │
   ▼
EVALUATING
   │
   ▼
NEGOTIATING
   │
   ▼
CHALLENGE_REQUIRED
   │
   ▼
CHALLENGE_PRESENTED
   │
   ▼
AWAITING_RESPONSE
   │
   ▼
VERIFYING
   │
   ├───────────────┐
   │               │
   ▼               ▼
REQUIREMENTS     FAILED
REMAIN
   │
   ▼
NEGOTIATING
   │
   ▼
...
   │
   ▼
SATISFIED
   │
   ▼
COMPLETING
   │
   ▼
COMPLETED
```

Estados terminales adicionales:

- FAILED
- EXPIRED
- CANCELLED
- REVOKED
- ABORTED

## 10. AuthenticationTransactionStatus

enum AuthenticationTransactionStatus: string
{
case Created = 'created';
case Evaluating = 'evaluating';
case Negotiating = 'negotiating';
case ChallengeRequired = 'challenge_required';
case AwaitingResponse = 'awaiting_response';
case Verifying = 'verifying';
case Satisfied = 'satisfied';
case Completing = 'completing';
case Completed = 'completed';

case Failed = 'failed';
case Expired = 'expired';
case Cancelled = 'cancelled';
case Revoked = 'revoked';
case Aborted = 'aborted';
}

## 11. Transaction != Session

Una Authentication Transaction no es una Session.

```text
Authentication Transaction
=
temporary authentication process

Session
=

post-authentication security context
```

## 12. Transaction Without Session

Debe funcionar en:

- stateless API
- OAuth callback
- CLI
- device flow
- machine flow
- initial login

## 13. Transaction Lifetime

Será corta.
Ejemplo conceptual:

- 5–15 minutes
- pero configurable por flujo.

## 14. Flow-Specific Lifetime

Ejemplo:

- login              10m
- reauthentication    5m
- passkey challenge   3m
- recovery            15m
- break-glass          5m

## 15. Expiration Is Mandatory

Ninguna Authentication Transaction interactiva deberá permanecer indefinidamente activa.

## 16. Authentication Requirement Input

El sistema recibe requirements desde documento 36.

- Ejemplo:
- Minimum Assurance: HIGH
- MFA: required
- Phishing Resistance: required

Freshness: <= 5 minutes

## 17. Current Context

Documento 37 aporta:

- Current Assurance: STANDARD
- Password: verified

Password Freshness: 2 minutes
Phishing Resistant: false

## 18. Requirement Gap

El sistema calculará:

```text
Requirement
-

Current Context
=

Authentication Gap
```

## 19. AuthenticationRequirementGap

final readonly class AuthenticationRequirementGap
{
public function __construct(
public AuthenticationRequirementSet $required,
public AuthenticationAssuranceProfile $current,
public array $missingCapabilities,
public array $freshnessViolations,
public array $additionalConstraints,
) {}
}

## 20. Ejemplo

Requirement:

```text
HIGH
MFA
PHISHING_RESISTANT
```

Current:

```text
STANDARD
Password
```

Gap:

- POSSESSION FACTOR
- PHISHING RESISTANCE
- HIGH assurance

## 21. Challenge Negotiation

VoltStack deberá determinar qué challenges pueden cerrar ese gap.

## 22. Challenge Candidate

Ejemplo:

- Passkey
- Security Key
- TOTP
- Password
- Federated Reauthentication
- Client Certificate
- Recovery Method

## 23. ChallengeDescriptor

final readonly class AuthenticationChallengeDescriptor
{
public function __construct(
public AuthenticationChallengeType $type,
public AuthenticationMethod $method,
public AuthenticationCapabilitySet $potentialCapabilities,
public InteractionMode $interactionMode,
public ChallengePriority $priority,
) {}
}

## 24. Potential Capability

Importante:

- potential capability
- ≠
- verified capability

El challenge solo indica qué podría demostrar.
Documento 37 determinará lo realmente demostrado.

## 25. AuthenticationChallengeType

enum AuthenticationChallengeType: string
{
case Password = 'password';
case Passkey = 'passkey';
case WebAuthnSecurityKey = 'webauthn_security_key';
case Totp = 'totp';
case EmailOtp = 'email_otp';
case SmsOtp = 'sms_otp';
case RecoveryCode = 'recovery_code';
case Federated = 'federated';
case ClientCertificate = 'client_certificate';
case DeviceApproval = 'device_approval';
case Custom = 'custom';
}

## 26. Challenge Negotiator

interface AuthenticationChallengeNegotiatorInterface
{
public function negotiate(
AuthenticationNegotiationContext $context
): AuthenticationChallengePlan;
}

## 27. Negotiation Inputs

El negotiator podrá considerar:

- requirement gap
- principal
- available methods
- enrolled credentials
- tenant
- realm
- application
- device capabilities
- client capabilities
- risk
- security posture
- authentication history
- method policy
- provider availability
- user preference

## 28. Security Before Preference

El orden será:

```text
Security Eligibility
       ↓
Requirement Satisfaction
       ↓
Availability
       ↓
Compatibility
       ↓
Preference / UX
Nunca al revés.
```

## 29. Challenge Plan

final readonly class AuthenticationChallengePlan
{
public function__construct(
public array $candidates,
public AuthenticationChallengeSelectionStrategy $strategy,
public AuthenticationRequirementGap $gap,
) {}
}

## 30. Candidate Set

Ejemplo:

```text
Requirement:
Phishing Resistant
```

Candidates:

1. Platform Passkey
2. Security Key
3. Enterprise IdP reauthentication

Excluded:

- TOTP
- SMS
- Email OTP

## 31. Negotiation Is Not Downgrade

Si policy exige:
PHISHING_RESISTANT
y el usuario selecciona:

- SMS OTP
- VoltStack no deberá reducir el requirement para acomodarlo.

## 32. No Suitable Challenge

Resultado explícito:
AUTHENTICATION_REQUIREMENT_UNSATISFIABLE

## 33. Unsatisfiable Requirement

Puede ocurrir porque:

- no compatible credential
- method disabled
- provider unavailable
- device unsupported
- credential revoked
- tenant policy
- realm policy
- required assurance impossible

## 34. Challenge Selection Strategies

enum AuthenticationChallengeSelectionStrategy: string
{
case Automatic = 'automatic';
case UserChoice = 'user_choice';
case PolicyPreferred = 'policy_preferred';
case Sequential = 'sequential';
case Parallel = 'parallel';
}

## 35. Automatic

VoltStack selecciona el mejor challenge elegible.

## 36. User Choice

Usuario recibe alternativas válidas.
Ejemplo:

```text
Verify with:

• Passkey
• Security Key
• Authenticator App
```

## 37. Policy Preferred

Policy define orden.

- Passkey
- Security Key
- TOTP

## 38. Sequential

Se necesitan varios challenges:

```text
Password
   ↓
TOTP
   ↓
Complete
```

## 39. Parallel

Casos especiales podrían permitir challenges independientes simultáneos.
Ejemplo enterprise:

```text
Administrator Authentication
         +
Manager Approval
```

aunque approval pertenece conceptualmente también al sistema de autorización/workflow.

## 40. Challenge Ranking

No usar únicamente:

```php
priority = 1
como modelo de seguridad.
```

## 41. Ranking Dimensions

Puede considerar:

- security suitability
- requirement coverage
- phishing resistance
- assurance potential
- user friction
- availability
- provider health
- credential state

## 42. Security Dominance

Un método de menor fricción nunca debe superar a otro si no satisface requirement.

## 43. Challenge Coverage

Cada candidate puede indicar qué parte del gap puede cubrir.

## 44. Example

Gap:

```text
MFA
Phishing Resistant
Freshness <= 5m
```

Passkey:
covers all

TOTP:

- covers MFA + freshness
- does NOT cover phishing resistance

## 45. Minimal Challenge Principle

Si el contexto ya satisface parte del requirement, no pedir evidencia innecesaria.

## 46. Example

Current:
Password verified 1 minute ago
Requirement:
Password + Passkey
Solicitar solo:

- Passkey
- no password otra vez salvo policy explícita.

## 47. Reauthentication vs Step-Up

No son equivalentes.

```text
Reauthentication
=

prove again

Step-Up
=

increase assurance/capabilities
```

## 48. Example Reauthentication

Passkey HIGH
authenticated 4 hours ago

Requirement:
HIGH <= 5 minutes

Action:
Passkey again

## 49. Example Step-Up

Password STANDARD

Requirement:
HIGH + phishing resistance

Action:
Passkey

## 50. Challenge Instance

Descriptor describe un tipo.
Instance representa un challenge concreto.

## 51. AuthenticationChallenge

final readonly class AuthenticationChallenge
{
public function __construct(
public AuthenticationChallengeId $id,
public AuthenticationTransactionId $transactionId,
public AuthenticationChallengeDescriptor $descriptor,
public AuthenticationChallengeStatus $status,
public DateTimeImmutable $createdAt,
public DateTimeImmutable $expiresAt,
public int $attempt,
) {}
}

## 52. Challenge ID

Debe ser:

- opaque
- unguessable
- unique

## 53. Challenge State Machine

CREATED
│
▼
PRESENTED
│
▼
AWAITING_RESPONSE
│
▼
RECEIVED
│
▼
VERIFYING
│
├───────┐
▼       ▼
VERIFIED  FAILED
│
▼
CONSUMED
Alternativas:

- EXPIRED
- CANCELLED
- REVOKED
- SUPERSEDED

## 54. AuthenticationChallengeStatus

enum AuthenticationChallengeStatus: string
{
case Created = 'created';
case Presented = 'presented';
case AwaitingResponse = 'awaiting_response';
case Received = 'received';
case Verifying = 'verifying';
case Verified = 'verified';
case Consumed = 'consumed';

case Failed = 'failed';
case Expired = 'expired';
case Cancelled = 'cancelled';
case Revoked = 'revoked';
case Superseded = 'superseded';
}

## 55. Verified != Consumed

Una challenge proof puede verificarse y después consumirse al avanzar la transaction.
Esto ayuda a evitar reutilización.

## 56. One-Time Challenge

Por defecto:
Un challenge interactivo es de un solo uso.

## 1. Challenge Binding

Todo challenge deberá ligarse como mínimo a:

- transaction
- flow
- realm
- tenant where applicable
- principal when known
- requirement

## 2. Additional Binding

Dependiendo del método:

- session
- client
- device
- origin
- RP ID
- redirect URI
- provider
- requested operation

## 3. Challenge Binding Purpose

Evitar que challenge generado para:
change profile
sea reutilizado para:
delete tenant

## 4. Sensitive Operation Binding

Documento 32.

- Challenge podrá incluir:
- operation_id
- intent_id
- payload_digest
- cuando sea necesario.

## 5. Challenge Payload

El payload específico del authenticator deberá encapsularse.

## 6. AuthenticationChallengePayload

interface AuthenticationChallengePayloadInterface
{
public function type(): AuthenticationChallengeType;
}

## 7. Password Challenge Payload

Puede ser mínimo.
No debe contener password esperado ni hash.

## 8. WebAuthn Challenge Payload

Puede contener:

- challenge
- rpId
- allowedCredentials
- userVerification
- timeout
- extensions
- según protocolo.

## 9. TOTP Challenge

Normalmente no requiere enviar secreto.

## 10. Federated Challenge

Puede contener referencia segura a:

- provider
- authorization request
- state
- PKCE context
- nonce
- redirect target

## 11. Recovery Challenge

Debe utilizar requisitos más restrictivos y contexto RECOVERY.

## 12. Challenge Factory

interface AuthenticationChallengeFactoryInterface
{
public function create(
AuthenticationChallengeDescriptor $descriptor,
AuthenticationTransaction $transaction
): AuthenticationChallenge;
}

## 13. Challenge Presenter

Separar challenge de representación.

## 14. AuthenticationChallengePresenterInterface

interface AuthenticationChallengePresenterInterface
{
public function present(
AuthenticationChallenge $challenge,
AuthenticationPresentationContext $context
): AuthenticationChallengePresentation;
}

## 15. Presentation != Challenge

El mismo challenge lógico puede presentarse mediante:

- HTML
- SPA JSON
- mobile API
- CLI prompt
- native application

## 16. Presentation Layer

Nunca debe decidir assurance.

## 17. HTML Presentation

Ejemplo:

```php
GET /auth/challenge/{opaque-reference}
produce formulario apropiado.
```

## 18. SPA Presentation

Ejemplo conceptual:

```php
{
  "type": "authentication.challenge",
  "transaction": "...",
  "challenge": "...",
  "method": "passkey",
  "interaction": {
    "mode": "webauthn"
  }
}
```

## 19. SPA Security

Frontend no decide:

- which methods are trusted
- whether requirement is satisfied
- assurance level
- challenge validity

Todo eso pertenece al servidor.

## 20. Client Capabilities

Frontend puede declarar capacidades técnicas.

- Ejemplo:
- WebAuthn supported
- platform authenticator available
- redirect capable

## 21. Client Capability != Security Evidence

Que navegador diga:

- WebAuthn supported
- no demuestra que exista una credential válida.

## 22. ClientCapabilitySet

final readonly class AuthenticationClientCapabilitySet
{
public function __construct(
public bool $webauthn,
public bool $redirect,
public bool $popup,
public bool $interactive,
) {}
}

## 23. Progressive Enhancement

Login debe poder funcionar con diferentes clients sin duplicar dominio.

## 24. Interactive Flow Engine

Componente central:

```php
interface AuthenticationFlowEngineInterface
{
    public function advance(
        AuthenticationTransactionId $transaction,
        AuthenticationFlowInput $input
    ): AuthenticationFlowResult;
}
```

## 25. Flow Engine Responsibilities

load transaction
validate state
process response
verify evidence
update transaction
re-evaluate context
calculate remaining gap
negotiate next challenge
complete or continue

## 26. Flow Engine Does Not Verify Every Credential Directly

Delegará a:

- Authenticator Resolver
- Authenticators
- Evidence Validators
- Federation subsystem
- ya definidos previamente.

## 27. Flow Result

sealed interface AuthenticationFlowResult {}
Conceptualmente:

- ChallengeRequired
- RedirectRequired
- Waiting
- Completed
- Failed
- Cancelled

## 28. ChallengeRequired Result

final readonly class ChallengeRequired implements AuthenticationFlowResult
{
public function __construct(
public AuthenticationChallengePresentation $challenge,
) {}
}

## 29. Completed Result

final readonly class AuthenticationCompleted implements AuthenticationFlowResult
{
public function __construct(
public AuthenticationContext $context,
public AuthenticationCompletion $completion,
) {}
}

## 30. Authentication Continuation

Uno de los conceptos centrales de este documento.

```text
Después de Authentication:
¿Qué debe ocurrir?
```

## 31. Examples

continue login
retry sensitive operation
return to checkout
finish account linking
return to OAuth authorization
continue admin operation
resume SPA navigation

## 32. AuthenticationContinuation

interface AuthenticationContinuationInterface
{
public function type(): AuthenticationContinuationType;
}

## 33. Continuation Types

enum AuthenticationContinuationType: string
{
case Redirect = 'redirect';
case Route = 'route';
case Operation = 'operation';
case SpaNavigation = 'spa_navigation';
case OAuthAuthorization = 'oauth_authorization';
case AccountLinking = 'account_linking';
case Enrollment = 'enrollment';
case ApiResponse = 'api_response';
}

## 34. Continuation != Arbitrary URL

Nunca almacenar y ejecutar ciegamente:
?return=<https://attacker.example>

## 35. Open Redirect Prevention

Continuation targets deberán validarse.

## 36. Preferred Model

Utilizar:

- route identifier
- operation identifier
- internal intent
- validated URI

en lugar de arbitrary URL.

## 37. Continuation Resolver

interface AuthenticationContinuationResolverInterface
{
public function resolve(
AuthenticationContinuation $continuation,
AuthenticationContext $context
): AuthenticationContinuationResult;
}

## 38. Continuation Binding

Debe estar vinculada a la transaction.

## 39. Continuation Integrity

Documento 39 profundizará en:

- state protection
- nonce
- anti-replay
- signed continuation
- cryptographic binding

Este documento establece la semántica.

## 40. Original Intent

Para operaciones sensibles:

- Authentication
- debe continuar el intent original, no una reconstrucción arbitraria.

## 41. AuthenticationIntent

final readonly class AuthenticationIntent
{
public function __construct(
public AuthenticationIntentId $id,
public OperationReference $operation,
public ResourceReference|null $resource,
public PayloadDigest|null $payloadDigest,
) {}
}

## 42. Intent Binding

Ejemplo:
Delete Tenant 42
no debe convertirse durante step-up en:
Delete Tenant 43

## 43. WYSIWYS

Arquitectura compatible con:

- What You See Is What You Sign
- para operaciones de alta sensibilidad.

## 44. Safe Retry

Después de step-up:

- original operation
- puede reintentarse solo si es seguro.

## 45. Unsafe Automatic Replay

No repetir automáticamente:

- financial transfer
- delete operation
- external side effect
- non-idempotent POST

sin modelo explícito de intent/idempotency.

## 46. SPA Flow

Ejemplo:

```text
SPA
 │
 ├── POST /tenant/42/delete
 │
 ▼
Server
 │
 └── AUTHENTICATION_REQUIRED
          │
          ▼
     Challenge Plan
          │
          ▼
      Passkey UI
          │
          ▼
     Evidence Response
          │
          ▼
     Context Re-evaluation
          │
          ▼
       Satisfied
          │
          ▼
     Resume Intent
```

## 47. SPA Response Contract

Conceptualmente:

```php
{
  "status": "authentication_required",
  "transaction": "...",
  "challenge": {
    "type": "passkey"
  },
  "continuation": {
    "type": "operation"
  }
}
```

## 48. SPA Navigation

Authentication challenge no debe destruir necesariamente SPA state.

## 49. Modal Authentication

Puede soportarse:

```text
Sensitive action
      ↓
Auth modal
      ↓
Passkey
      ↓
Continue
```

## 50. Full Page Authentication

También:

```php
Sensitive action
      ↓
/auth/challenge
      ↓
Complete
      ↓
return
```

## 51. UX Is Transport Concern

Ambos usan la misma transaction.

## 52. Browser Back Button

Debe manejarse correctamente.
Un challenge superseded/consumed no vuelve a ser válido por navegar atrás.

## 53. Multi-Tab

Dos tabs pueden interactuar con misma Session.

## 54. Transaction Isolation

Cada sensitive intent deberá poder tener transaction independiente.

## 55. Example

Tab A:
Change password

Tab B:

- Delete API key
- No deben mezclar challenges.

## 112. Session-Wide Step-Up

Policy puede permitir que successful step-up eleve Session Assurance temporalmente.

## 113. Operation-Bound Step-Up

Para mayor seguridad:
proof valid only for operation X

## 114. Policy Decides Scope

Documento 32/36.

## 115. Challenge Choice UI

Si existen alternativas:

- Use a passkey
- Use security key
- Use authenticator app

## 116. Do Not Reveal Sensitive Inventory

Evitar:

- Your YubiKey serial 123456 is registered
- si no es necesario.

## 117. Credential Enumeration

Challenge negotiation no deberá permitir descubrir credentials de otras identities.

## 118. Unknown User Protection

Initial login debe evitar diferencias excesivas que permitan:

- username enumeration
- email enumeration
- credential enumeration

## 119. Identity Discovery

Puede ocurrir antes o durante transaction.

## 120. Anonymous Transaction

Inicialmente:

```php
principal = unknown
es válido.
```

## 121. Identified Transaction

Después:
principal = resolved

## 122. Principal Switching

Una transaction no debe cambiar silenciosamente:
Alice → Bob

## 123. Identity Binding

Cuando se resuelve principal, transaction queda vinculada.

## 124. Restart on Identity Change

Si usuario quiere cambiar cuenta:

- cancel transaction
- start new transaction
- preferentemente.

## 125. Password Flow

Identify
↓
Password Challenge
↓
Password Authenticator
↓
Evidence
↓
Context
↓
Requirement Evaluation

## 126. Password Failure

No necesariamente termina transaction inmediatamente.
Puede permitir retry bajo throttling.

## 127. Attempts

Challenge podrá tener:

- attempt_count
- pero brute-force policy pertenece al sistema de throttling 19.

## 128. Maximum Attempts

Challenge puede expirar/revocarse después de policy threshold.

## 129. TOTP Flow

Password Evidence
↓
Requirement still incomplete
↓
TOTP Challenge
↓
TOTP Verification
↓
Independent factor evaluation
↓
Context re-evaluation

## 130. TOTP Replay

El flow engine no debe aceptar un código previamente consumido cuando el authenticator/replay policy lo prohíba.
Documento 39 complementará anti-replay general.

## 131. Passkey Flow

Requirement Gap
↓
Passkey Candidate
↓
WebAuthn Challenge
↓
Browser Ceremony
↓
Assertion
↓
WebAuthn Authenticator
↓
Evidence
↓
Capability Assessment

## 132. Passkey Cancellation

Usuario puede cancelar browser ceremony.
Resultado:

- challenge cancelled
- o volver a negociación según UX/policy.

## 133. Passkey Failure

No fallback automático a método débil si requirement exige phishing resistance.

## 134. Federation Flow

Transaction
↓
Federated Challenge
↓
Authorization Request
↓
External IdP
↓
Callback
↓
Federated Evidence
↓
Context Re-evaluation

## 135. Federation Continuation

Callback deberá recuperar transaction correcta.

## 136. Login CSRF

Federation linking/login deberá impedir que callback de una transaction atacante se inserte en transaction de víctima.
Protecciones criptográficas detalladas en 39.

## 137. Provider Selection

Puede negociarse:

- Google
- Microsoft
- Enterprise SSO
- GitHub

solo entre providers habilitados.

## 138. Provider Discovery

Tenant/realm puede restringir.

```php
Ejemplo:
@company.com
→ Enterprise IdP
```

pero discovery no debe introducir account enumeration.

## 139. Enterprise Forced Federation

Policy puede indicar:

- password unavailable
- enterprise SSO required

## 140. Federation Failure

Provider outage no implica fallback automático a password.
Fallback debe estar autorizado por policy.

## 141. Recovery Flow

Recovery no es simplemente:
choose another method

## 142. Recovery Mode

Debe cambiar transaction type/context:
RECOVERY

## 143. Recovery Restrictions

Puede producir:

- restricted authentication context
- aunque recovery challenge tenga éxito.

## 144. Recovery Continuation

Ejemplo:

```text
Recovery proof
    ↓
Restricted context
    ↓
Bind new passkey
    ↓
Revoke compromised credentials
    ↓
Restore normal access
```

## 145. Account Linking Flow

Documento 34.

```text
Authenticated identity
       ↓
Fresh Authentication
       ↓
Create Linking Transaction
       ↓
External/credential challenge
       ↓
Verify ownership
       ↓
Conflict detection
       ↓
Bind method
```

## 146. Linking Transaction Isolation

Login transaction y linking transaction no son intercambiables.

## 147. Critical Rule

Authentication of an external identity for login does not automatically authorize linking it to the currently authenticated account.

## 1. Enrollment Flow

Para TOTP/passkey/etc.:

```text
authenticate
    ↓
authorize enrollment
    ↓
create enrollment transaction
    ↓
credential ceremony
    ↓
proof
    ↓
activate credential
```

## 2. Credential Pending State

Credential no deberá quedar Active antes de completar verification requerida.

## 3. Privileged Flow

Documento 32.

```text
Normal Session
     ↓
Privileged Requirement
     ↓
Privileged Authentication Transaction
     ↓
Strong Challenge
     ↓
Privileged Context
```

## 4. Break-Glass Flow

Debe ser explícito.
Nunca:

```php
if emergency:
    skip authentication
```

## 5. Break-Glass Transaction

type = BREAK_GLASS
con:

- reason
- incident reference
- approval where required
- special challenge policy
- short lifetime
- audit

## 6. Break-Glass Challenge Selection

Solo emergency-capable credentials.

## 7. No Normal Fallback

Si emergency flow requiere hardware credential:

- forgot password
- no debe ofrecer password reset normal como bypass.

## 8. Device Approval

Puede existir:

```text
Login on Device A
      ↓
```

Approve on Device B

## 9. Out-of-Band Challenge

Challenge response puede llegar por otro channel.

## 10. Transaction Waiting State

Para ello:

- WAITING_EXTERNAL
- puede añadirse.

## 11. AuthenticationTransactionStatus Extension

case WaitingExternal = 'waiting_external';

## 12. Polling

Client puede consultar transaction.

## 13. Polling Security

No revelar más información de la necesaria.

## 14. Push Completion

Arquitectura puede permitir:

- WebSocket
- SSE
- push notification

sin depender de ellos.

## 15. Polling Frequency

Debe limitarse.

## 16. External Approval Timeout

Challenge expira normalmente.

## 17. QR Authentication

Puede modelarse como out-of-band challenge.

## 18. Device Code Flow

Para CLI/TV/device:

```text
CLI
 │
 ├── receives device code
 │
 ▼
User Browser
 │
 ├── authenticates
 │
 ▼
Transaction completed
 │
 ▼
CLI receives credential
```

## 19. Device Code != Authentication Credential

Es una referencia temporal a transaction/challenge.

## 20. User Code

Debe ser:

- short enough for human entry
- high enough entropy for threat model
- rate limited
- short-lived

## 21. CLI Authentication

VoltStack deberá soportar interactive CLI sin acoplar flow a HTML.

## 22. CLI Challenge Presentation

Ejemplo:
Authentication required.

```text
Open:
<verified application URL>
```

Enter code:
ABCD-EFGH

## 170. CLI Password Prompt

Si se soporta:

- hidden input
- no command-line argument
- no shell history

## 171. Machine Authentication

Documento 33.
La mayoría de machine flows no serán interactive.

## 172. Machine Challenge

Puede existir conceptualmente:

- mTLS proof
- signed nonce
- workload attestation
- token exchange

pero no requiere UI.

## 173. Unified Flow Model

Interactive y non-interactive pueden compartir:

- transaction
- challenge
- evidence
- verification
- completion
- cuando sea útil.

## 174. Do Not Force Everything Interactive

Un service-to-service token exchange no debe fingir formulario.

## 175. InteractionMode

enum InteractionMode: string
{
case Form = 'form';
case BrowserNative = 'browser_native';
case Redirect = 'redirect';
case OutOfBand = 'out_of_band';
case DeviceCode = 'device_code';
case Cli = 'cli';
case NonInteractive = 'non_interactive';
}

## 176. Authentication Flow Graph

Flows complejos deben representarse como graph/state machine.

## 177. Example

┌── Passkey ───────────┐
│                      │
Identify ── Password ── Risk ── Need MFA?
│                      │
└── Federation ────────┘
│
▼
Complete

## 178. Graph Nodes

Evaluate
Challenge
Verify
Branch
Wait
Complete
Abort

## 179. AuthenticationFlowNode

interface AuthenticationFlowNodeInterface
{
public function execute(
AuthenticationFlowExecutionContext $context
): AuthenticationFlowTransition;
}

## 180. Static vs Dynamic Flow

No diseñar login como graph completamente estático.
Risk/policy/context pueden cambiar camino dinámicamente.

## 181. Policy-Driven Flow

Ejemplo:

```text
Known Device + Low Risk
→ Password
```

Unknown Device + Elevated Risk
→ Password + Passkey

## 182. Flow Definition

Policy define requirements.
Negotiator selecciona mecanismo.
Flow Engine orquesta.

## 183. Separation

Policy Engine
"What is required?"

Assurance System
"What is currently proven?"

Challenge Negotiator
"How can the gap be satisfied?"

Flow Engine
"How do we execute it?"

## 184. Architectural Boundary

Esta separación es obligatoria.

## 185. Challenge Provider

Cada mecanismo podrá registrar provider.

```php
interface AuthenticationChallengeProviderInterface
{
    public function supports(
        AuthenticationChallengeType $type
    ): bool;

    public function create(
        AuthenticationChallengeCreationContext $context
    ): AuthenticationChallenge;
}
```

## 186. Built-In Providers

PasswordChallengeProvider
TotpChallengeProvider
PasskeyChallengeProvider
FederatedChallengeProvider
RecoveryChallengeProvider
DeviceApprovalChallengeProvider

## 187. Challenge Handler

Procesa respuesta.

```php
interface AuthenticationChallengeHandlerInterface
{
    public function handle(
        AuthenticationChallenge $challenge,
        AuthenticationChallengeResponse $response
    ): AuthenticationChallengeResult;
}
```

## 188. Response Objects

Nunca pasar arrays arbitrarios por todo Core.

## 189. Example

final readonly class TotpChallengeResponse
implements AuthenticationChallengeResponse
{
public function __construct(
public string $code,
) {}
}

## 190. Sensitive Data Lifetime

Password/OTP/etc. deberán existir en memoria el mínimo tiempo práctico.

## 191. No Persistence

No almacenar raw:

- password
- OTP

WebAuthn assertion beyond necessary audit-safe metadata
OAuth authorization code
client secret
en transaction store.

## 192. Challenge Result

interface AuthenticationChallengeResultInterface {}
Implementaciones:

- Verified
- Rejected
- RetryableFailure
- TerminalFailure
- Cancelled
- Expired

## 193. Retryable Failure

Ejemplo:

- wrong TOTP
- si policy permite retry.

## 194. Terminal Failure

Ejemplo:

- credential revoked
- transaction compromised
- challenge binding mismatch

## 195. Error Semantics

Documento 25 proporciona taxonomy general.
Este sistema añadirá errores específicos.

## 196. Failure Taxonomy

AUTH_TRANSACTION_NOT_FOUND
AUTH_TRANSACTION_EXPIRED
AUTH_TRANSACTION_CANCELLED
AUTH_TRANSACTION_REVOKED
AUTH_TRANSACTION_INVALID_STATE
AUTH_TRANSACTION_CONTEXT_MISMATCH

AUTH_REQUIREMENT_UNSATISFIABLE

AUTH_CHALLENGE_NOT_FOUND
AUTH_CHALLENGE_EXPIRED
AUTH_CHALLENGE_CANCELLED
AUTH_CHALLENGE_REVOKED
AUTH_CHALLENGE_ALREADY_CONSUMED
AUTH_CHALLENGE_SUPERSEDED
AUTH_CHALLENGE_INVALID_STATE
AUTH_CHALLENGE_BINDING_MISMATCH

AUTH_CHALLENGE_METHOD_UNAVAILABLE
AUTH_CHALLENGE_METHOD_NOT_ALLOWED
AUTH_CHALLENGE_RESPONSE_INVALID

AUTH_CONTINUATION_INVALID
AUTH_CONTINUATION_EXPIRED
AUTH_CONTINUATION_NOT_ALLOWED

AUTH_FLOW_ABORTED
AUTH_FLOW_MAX_TRANSITIONS_EXCEEDED
AUTH_FLOW_NO_VALID_TRANSITION

## 197. Generic External Errors

No exponer:

- User exists but has no passkey.
- cuando permita enumeration.

## 198. Internal Diagnostics

Audit puede conservar reason code apropiado.

## 199. Flow Transition Limit

Debe existir protección contra loops.

## 200. Example

password
→ negotiate
→ password
→ negotiate
→ password
...
debe detectarse.

## 201. Maximum Transitions

Configurable por flow.

## 202. Challenge Supersession

Si nuevo challenge reemplaza uno anterior:
old challenge → SUPERSEDED

## 203. Example

Usuario selecciona:
TOTP
luego cambia a:

- Passkey
- TOTP challenge puede invalidarse.

## 204. Multiple Active Challenges

Permitido solo si flow lo declara.

## 205. Race Conditions

Ejemplo:

- Tab A verifies Passkey
- Tab B submits old TOTP
- Debe resolverse atómicamente.

## 206. Transaction Version

Usar optimistic concurrency:
transaction_version

## 207. Compare-and-Swap

Conceptualmente:

- UPDATE transaction
- WHERE id = X

AND version = 7

## 208. Distributed Lock

Alternativamente para transiciones críticas.

## 209. Preferred Approach

Minimizar locks largos.

- Usar:
- atomic state transition
- version check
- single-use consumption

## 210. Transaction Repository

interface AuthenticationTransactionRepositoryInterface
{
public function get(
AuthenticationTransactionId $id
): ?AuthenticationTransaction;

public function save(
AuthenticationTransaction $transaction
): void;
}

## 211. Atomic Repository

Necesitará operaciones como:

```php
public function transition(
    AuthenticationTransactionId $id,
    AuthenticationTransactionVersion $expected,
    AuthenticationTransactionTransition $transition
): AuthenticationTransaction;
```

## 212. Challenge Repository

interface AuthenticationChallengeRepositoryInterface
{
public function store(AuthenticationChallenge $challenge): void;

public function consume(
AuthenticationChallengeId $id
): AuthenticationChallenge;
}

## 213. Atomic Consumption

consume() debe ser atómico.

## 214. Storage Backends

Podrán ser:

- Redis
- database
- distributed cache

in-memory for tests/single-process dev

## 215. Production Distributed Runtime

En múltiples nodes:

- Node A creates challenge
- Node B receives response
- debe funcionar.

## 216. Sticky Sessions

No deberán ser requisito arquitectónico.

## 217. Transaction State Distribution

State compartido cuando el flow lo requiera.

## 218. Stateless Transactions

Algunos componentes podrían serializarse criptográficamente.
Documento 39 analizará cuándo.

## 219. Stateful Preferred for Sensitive Flows

Para:

- break-glass
- credential changes
- account recovery
- admin step-up

stateful suele facilitar:

- revocation
- single-use
- audit
- concurrency control

## 220. Expiration Cleanup

Transaction store deberá soportar TTL.

## 221. Garbage Collection

Estados terminales podrán conservar metadata audit-safe por política.

## 222. Transaction Data Retention

No confundir:
runtime transaction state
con:
audit record

## 223. Audit Record

Puede sobrevivir más tiempo sin conservar secretos.

## 224. Authentication Flow Context

final readonly class AuthenticationFlowExecutionContext
{
public function __construct(
public AuthenticationTransaction $transaction,
public AuthenticationContext $authentication,
public AuthenticationRequirementSet $requirements,
public AuthenticationRiskContext $risk,
public AuthenticationDeviceContext $device,
public AuthenticationClientCapabilitySet $client,
) {}
}

## 225. Context Re-Evaluation

Después de cada successful evidence:

```php
Evidence
   ↓
Authentication Context Builder (37)
   ↓
Policy Requirement Evaluation (36)
```

## 226. Do Not Assume Success Ends Flow

Password verified puede ser correcto y aun así:
MFA required

## 227. Example

Password VERIFIED

Transaction:
not completed

Reason:
PHISHING_RESISTANT capability missing

## 228. Challenge Chaining

Challenge A
↓
Evidence A
↓
Re-evaluate
↓
Challenge B

## 229. Evidence Accumulation

Transaction puede acumular references a evidence verificadas.

## 230. Evidence Lifetime

Solo evidencia suficientemente fresca y válida cuenta.

## 231. Evidence Expiration During Flow

Ejemplo extremo:

- Password verified
- user waits too long
- Passkey verified

password freshness now expired
Policy puede requerir password otra vez.

## 232. Dynamic Requirements

Risk puede cambiar durante flow.

## 233. Example

Initial:
STANDARD required

During flow:
risk engine detects anomaly

New requirement:
HIGH + phishing resistant

## 234. Requirement Re-Evaluation

Antes de completar transaction, recalcular requirements cuando policy lo exija.

## 235. TOCTOU

Evitar:

```text
requirements evaluated at start
↓
20 minutes later
↓
complete without recheck
```

## 236. Final Authentication Check

Antes de COMPLETED:

- principal state
- credential state
- session state
- tenant state
- realm
- security epoch
- risk
- requirements
- assurance

pueden reevaluarse según sensibilidad.

## 237. Completion Service

interface AuthenticationCompletionServiceInterface
{
public function complete(
AuthenticationTransaction $transaction,
AuthenticationContext $context
): AuthenticationCompletion;
}

## 238. Completion Depends on Flow

Login:
issue session
Step-up:
upgrade context / issue proof
Recovery:
restricted recovery state
Linking:
bind credential

## 239. Completion Is Atomic

No:

```text
mark completed
↓
fail session creation
dejando estado inconsistente.
```

## 240. Completion Idempotency

Retry de completion no deberá crear:

- multiple sessions
- duplicate credentials
- duplicate account links

## 241. Completion Record

Puede existir:

```php
final readonly class AuthenticationCompletion
{
    public function __construct(
        public AuthenticationCompletionId $id,
        public AuthenticationTransactionId $transactionId,
        public AuthenticationCompletionType $type,
        public DateTimeImmutable $completedAt,
    ) {}
}
```

## 242. Cancellation

Usuario puede cancelar.

## 243. Cancellation Semantics

challenge cancel
puede:
return to method selection
mientras:

- transaction cancel
- termina todo flow.

## 244. Abort

Sistema puede abortar por:

- security event
- identity disabled
- credential compromise
- policy change
- tenant suspension
- session revocation

## 245. Revocation

Security subsystem puede revocar transaction externamente.

## 246. Transaction Security Epoch

Podrá capturar epochs relevantes.

## 247. Example

transaction created:
identity epoch = 9

password changed elsewhere:

- identity epoch = 10
- Transaction debe reevaluarse o invalidarse.

## 248. Session Binding

Reauthentication transaction normalmente estará ligada a Session.

## 249. Session Change

Si Session termina:

- reauth transaction
- debe invalidarse.

## 250. Login Transaction

No necesita Session previa.

## 251. Tenant Binding

Transaction iniciada para:

- Tenant A
- no deberá utilizarse para Tenant B.

## 252. Realm Binding

Especialmente:

- user realm
- admin realm
- platform realm

## 253. Cross-Realm Continuation

Debe requerir explicit transition.

## 254. Admin Realm

Login normal no debe elevarse automáticamente a admin realm.

## 255. Authentication Entry Points

Documento 22.
Entry Point inicia transaction apropiada.

## 256. Example

Protected Endpoint
↓
Authentication Entry Point
↓
Transaction Factory
↓
Challenge Negotiation

## 257. Transaction Factory

interface AuthenticationTransactionFactoryInterface
{
public function create(
AuthenticationTransactionRequest $request
): AuthenticationTransaction;
}

## 258. Transaction Request

final readonly class AuthenticationTransactionRequest
{
public function __construct(
public AuthenticationFlowType $flow,
public AuthenticationRequirementSet $requirements,
public AuthenticationContinuation $continuation,
public PrincipalReference|null $principal,
public TenantReference|null $tenant,
public RealmReference $realm,
) {}
}

## 259. Middleware Integration

Middleware podrá detectar:

- Unauthenticated
- Insufficient Assurance
- Reauthentication Required
- y crear/usar transaction.

## 260. Middleware Does Not Render Forms

Debe devolver:

- AuthenticationFlowResult
- adaptado por transport layer.

## 261. HTTP Adapter

AuthenticationFlowResult
↓
HTTP Authentication Adapter
↓
HTML / Redirect / JSON

## 262. CLI Adapter

AuthenticationFlowResult
↓
CLI Adapter
↓
Prompt / Device Code

## 263. SPA Adapter

AuthenticationFlowResult
↓
SPA Protocol Adapter
↓
Structured challenge response

## 264. Mobile Adapter

Puede reutilizar API representation.

## 265. Content Negotiation

No usar únicamente:

- Accept: application/json
- para decisiones de seguridad.
- Solo para representación.

## 266. API Clients

Para APIs no interactivas:

- 401
- puede incluir structured requirement metadata.

## 267. Example

{
"error": "authentication_required",
"requirement": "strong_authentication"
}
sin revelar detalles sensibles.

## 268. WWW-Authenticate

Cuando protocolos estándar lo permitan, usar correctamente headers estándar.

## 269. Interactive API

Una app móvil puede optar por iniciar transaction.

## 270. Method Discovery Endpoint

Si existe, debe estar protegido contra enumeration.

## 271. Example

No:
POST /auth/methods?email=<victim@example.com>

Response:

```php
["passkey", "totp", "google"]
sin mitigaciones.
```

## 272. Generic Discovery

Puede responder:

- Continue authentication
- y revelar métodos solo después de prueba/contexto apropiado.

## 273. Authentication Choice Persistence

Preferencia de usuario:

- prefer passkey
- puede almacenarse.

## 274. Preference Is Not Policy

Usuario no puede seleccionar:

- always SMS
- para evadir phishing-resistant requirement.

## 275. Remember Last Method

Puede mejorar UX.

## 276. Privacy

No compartir last-used provider en contexto donde revele información sensible innecesariamente.

## 277. Automatic Passkey First

Puede intentarse si:

- client supports it
- credential discovery model supports it
- policy allows

## 278. Conditional UI

WebAuthn conditional mediation podrá integrarse como presentation strategy.

## 279. Domain Independence

Core no dependerá de APIs JavaScript específicas.

## 280. Browser Adapter

Frontend runtime implementará ceremony.

## 281. Authentication UI Contract

Debe ser versionado.

## 282. Example Version

auth_protocol_version = 1

## 283. Protocol Evolution

Permitir:

- V1 client
- V2 server
- con compatibilidad definida.

## 284. Unknown Challenge Type

Client antiguo puede recibir:

- unsupported_challenge
- y solicitar alternative candidate si policy permite.

## 285. Fallback Negotiation

Debe volver al servidor.
No:

```php
if passkey fails:
    use sms
decidido unilateralmente.
```

## 286. Server Renegotiation

Client cannot handle Passkey
↓
Report capability limitation
↓
Server renegotiates
↓
Alternative eligible challenge

## 287. Client Claims Are Untrusted

webauthn=false puede afectar UX, pero nunca otorgar bypass.

## 288. Accessibility

Challenge presentation debe permitir alternativas accesibles cuando security policy lo permita.

## 289. Localization

Mensajes y labels pertenecen a presentation layer.

## 290. Domain Error Codes

Core devuelve códigos estables.

## 291. Human-Friendly Message

Adapter traduce:

- AUTH_CHALLENGE_EXPIRED
- a mensaje localizado.

## 292. Events

Eventos sugeridos:

```text
AuthenticationTransactionCreated
AuthenticationTransactionStarted
AuthenticationTransactionExpired
AuthenticationTransactionCancelled
AuthenticationTransactionRevoked
AuthenticationTransactionCompleted
AuthenticationTransactionFailed

AuthenticationChallengeNegotiationStarted
AuthenticationChallengeNegotiated
AuthenticationChallengeCreated
AuthenticationChallengePresented
AuthenticationChallengeResponseReceived
AuthenticationChallengeVerified
AuthenticationChallengeFailed
AuthenticationChallengeConsumed
AuthenticationChallengeExpired
AuthenticationChallengeCancelled
AuthenticationChallengeSuperseded

AuthenticationContinuationCreated
AuthenticationContinuationResolved
AuthenticationContinuationRejected

AuthenticationFlowAdvanced
AuthenticationFlowRequirementChanged
AuthenticationFlowCompleted
AuthenticationFlowAborted
```

## 293. Event Payload Security

No incluir:

- password
- OTP
- raw WebAuthn response
- OAuth authorization code
- access token
- refresh token
- recovery code

## 294. Audit

Registrar cuando corresponda:

- transaction ID
- flow type
- principal reference
- tenant
- realm
- challenge type
- method
- result
- failure reason
- risk classification
- assurance before
- assurance after
- continuation type
- timestamps

## 295. Audit Correlation

Transaction ID será excelente correlation identifier.

## 296. But Not Secret

No asumir que correlation ID por sí mismo autoriza acceso a transaction.

## 297. Metrics

auth_transaction_created_total
auth_transaction_completed_total
auth_transaction_failed_total
auth_transaction_expired_total

auth_challenge_created_total
auth_challenge_completed_total
auth_challenge_failed_total
auth_challenge_cancelled_total

auth_challenge_negotiation_total
auth_requirement_unsatisfiable_total

auth_flow_duration_seconds
auth_challenge_duration_seconds
auth_flow_transition_total

## 298. Useful Dimensions

flow_type
challenge_type
realm
result
failure_category
con control de cardinalidad.

## 299. Do Not Label by User

No:

- email
- user_id
- transaction_id
- credential_id
- como metric labels.

## 300. Tracing

Spans:

- auth.transaction.create
- auth.transaction.advance
- auth.challenge.negotiate
- auth.challenge.create
- auth.challenge.present
- auth.challenge.verify
- auth.context.reevaluate
- auth.continuation.resolve
- auth.transaction.complete

## 301. Trace Security

No incluir credential material.

## 302. Performance

Flow Engine no debe ejecutar todos los authenticators.

## 303. Authenticator Routing

Usar resolver/registry compilado.

## 304. Challenge Candidate Index

Puede precompilarse por:

- realm
- method
- capability
- flow type

## 305. Dynamic Filtering

Después aplicar:

- principal credentials
- risk
- device
- tenant
- availability

## 306. Provider Health

Negotiator puede considerar provider outage.

## 307. But Security Policy Dominates

Si único método permitido está caído:

- fail closed
- salvo emergency/fallback policy explícita.

## 308. Circuit Breaker

Federated provider adapters pueden usar resilience mechanisms.

## 309. No Silent Security Downgrade

Circuit breaker no significa:
IdP down → skip MFA

## 310. FrankenPHP

El Flow Engine puede ser long-lived si es stateless.

## 311. Safe Long-Lived Components

ChallengeProviderRegistry
CompiledFlowDefinitions
NegotiationStrategyRegistry
PresentationAdapterRegistry

## 312. Request/Transaction Scoped

AuthenticationTransaction
AuthenticationChallenge
AuthenticationFlowExecutionContext
ChallengeResponse
AuthenticationContext

## 313. Never

static $currentTransaction;
static $currentChallenge;
static $pendingUser;

## 314. Worker Reset

Toda referencia request-scoped deberá liberarse.

## 315. Fiber Safety

Concurrent requests dentro de mismo worker nunca compartirán transaction context mutable.

## 316. Async Authenticators

Flow Engine deberá poder manejar authenticators/providers asíncronos conceptualmente.

## 317. Waiting State

Ejemplo:

```text
send push approval
      ↓
WAITING_EXTERNAL
```

## 318. Resume

Cuando llega evento externo:

- load transaction
- validate state
- apply evidence
- advance

## 319. Resume Token

Documento 39 definirá protección criptográfica de referencias de resume.

## 320. Idempotency

External callbacks pueden llegar múltiples veces.

## 321. Callback Handling

Debe ser:

- idempotent
- single-use aware
- state validated

## 322. Duplicate Callback

No genera dos sesiones.

## 323. Delayed Callback

Si transaction expiró:
reject

## 324. Callback After Cancellation

Reject.

## 325. Callback After Completion

No reabre transaction.

## 326. Callback Ordering

En distributed systems callbacks pueden llegar fuera de orden.
State machine debe rechazarlos correctamente.

## 327. Flow Version

Transaction debe conocer:
flow_definition_version

## 328. Why

Deployment puede cambiar flow mientras transaction sigue activa.

## 329. Existing Transactions

Preferiblemente continúan bajo versión compatible original durante corta vida.

## 330. Critical Security Update

Puede revocar versiones anteriores.

## 331. FlowDefinitionVersion

final readonly class AuthenticationFlowDefinitionVersion
{
public function __construct(
public string $value,
) {}
}

## 332. Compiled Flow Definition

final readonly class CompiledAuthenticationFlowDefinition
{
public function __construct(
public AuthenticationFlowDefinitionVersion $version,
public array $nodes,
public array $transitions,
) {}
}

## 333. Configuration Example

return [

'authentication' => [

'flows' => [

'login' => [
'ttl' => '10 minutes',
'max_transitions' => 12,
],

'reauthentication' => [
'ttl' => '5 minutes',
],

'step_up' => [
'ttl' => '5 minutes',
'operation_binding' => true,
],

'recovery' => [
'ttl' => '15 minutes',
'restricted_completion' => true,
],

],

'challenges' => [

'passkey' => [
'enabled' => true,
'priority' => 100,
],

'totp' => [
'enabled' => true,
'priority' => 80,
],

'password' => [
'enabled' => true,
'priority' => 50,
],

],

],

];

## 334. Configuration Is Not Security Truth Alone

Effective policy viene de documento 36.

## 335. Extensibility

Plugins podrán registrar:

- Challenge Provider
- Challenge Handler
- Challenge Presenter
- Negotiation Contributor
- Flow Node
- Continuation Handler
- Interaction Adapter

## 336. Custom Challenge Contract

interface CustomAuthenticationChallengeProviderInterface
extends AuthenticationChallengeProviderInterface
{
}
No necesita privilegios especiales.

## 337. Plugin Security

Plugin no puede declarar:

- challenge verified
- sin producir Authentication Evidence válida.

## 338. Evidence Boundary

Challenge subsystem produce/verifica interacción.
Authenticator subsystem produce evidence.
Assurance subsystem evalúa evidence.

## 339. Extensibility Isolation

Plugin debe declarar capabilities que potencialmente soporta.
Documento 37 verifica resultado real.

## 340. Testing — Transaction

Cubrir:

- creation
- expiration
- cancellation
- revocation
- completion
- invalid transitions
- concurrent transitions
- version conflicts

## 341. Testing — Negotiation

Cubrir:

- single candidate
- multiple candidates
- no candidate
- disabled method
- policy restrictions
- client incompatibility
- provider outage
- user preference

## 342. Testing — Downgrade

Requirement fuerte nunca produce fallback débil.

## 343. Testing — Password

success
failure
retry
lockout interaction
expiration

## 344. Testing — TOTP

success
wrong code
replay
expired transaction

## 345. Testing — Passkey

successful assertion
user cancellation
wrong RP
wrong origin
invalid challenge
credential mismatch
user verification requirement

## 346. Testing — Federation

valid callback
wrong transaction
wrong provider
expired callback
duplicate callback
cancelled transaction
provider error

## 347. Testing — Continuation

valid route
invalid external redirect
operation mismatch
payload mismatch
expired continuation

## 348. Testing — SPA

challenge response
modal flow
navigation
cancel
retry
stale challenge

## 349. Testing — Multi-Tab

Challenges no se mezclan.

## 350. Testing — Concurrency

Dos successful submissions simultáneos:
exactly one consumption

## 351. Testing — Distributed

Node A create
Node B verify
Node C complete

## 352. Testing — FrankenPHP

Context no se filtra entre requests.

## 353. Testing — Fibers

Transactions concurrentes permanecen aisladas.

## 354. Testing — Dynamic Risk

Requirement puede aumentar durante flow.

## 355. Testing — TOCTOU

Identity suspendida antes de completion bloquea finalización.

## 356. Testing — Security Epoch

Cambio de epoch invalida/revalida transaction correctamente.

## 357. Testing — Flow Upgrade

Transaction V1 no se corrompe al desplegar Flow V2.

## 358. Testing — Recovery

Recovery completion queda restricted cuando corresponde.

## 359. Testing — Break-Glass

No existe fallback normal accidental.

## 360. Security Invariants — Transaction

AUTH-FLOW-01
Toda interacción Authentication compleja pertenece a una transaction identificable.
AUTH-FLOW-02
Transactions son temporales.

- AUTH-FLOW-03
- Transactions tienen estados explícitos.
- AUTH-FLOW-04

Transiciones inválidas se rechazan.

- AUTH-FLOW-05
- Transaction no equivale a Session.
- AUTH-FLOW-06

Una transaction completada no puede reabrirse.

## 361. Security Invariants — Challenge

AUTH-CHALLENGE-01
Todo challenge pertenece a una transaction.

- AUTH-CHALLENGE-02
- Challenge IDs son opacos e impredecibles.
- AUTH-CHALLENGE-03
- Challenges expiran.
- AUTH-CHALLENGE-04

Challenges son single-use por defecto.

- AUTH-CHALLENGE-05
- Challenge consumido no vuelve a verificarse.
- AUTH-CHALLENGE-06

Challenge está ligado a su contexto relevante.
AUTH-CHALLENGE-07
Challenge no puede reutilizarse entre tenants/realms/operations incompatibles.

## 362. Security Invariants — Negotiation

AUTH-NEG-01
Challenge Negotiation nunca debilita requirements.

- AUTH-NEG-02
- User preference no supera Security Policy.
- AUTH-NEG-03

Client capability no constituye Authentication Evidence.

- AUTH-NEG-04
- Fallback solo usa métodos que satisfacen requirements.
- AUTH-NEG-05

Provider outage no produce security downgrade implícito.

## 363. Security Invariants — Continuation

AUTH-CONT-01
Continuation está ligada a transaction.

- AUTH-CONT-02
- Continuation no acepta arbitrary redirects sin validación.
- AUTH-CONT-03

Sensitive operation continuation conserva original intent.

- AUTH-CONT-04
- Unsafe operations no se reejecutan automáticamente sin modelo explícito.
- AUTH-CONT-05

Authentication completion no autoriza cambiar el intent original.

## 364. Security Invariants — Identity

AUTH-FLOW-ID-01
Transaction no cambia silenciosamente de principal.

- AUTH-FLOW-ID-02
- Unknown principal no revela credential inventory.
- AUTH-FLOW-ID-03

Linking y login utilizan semánticas de transaction distintas.
AUTH-FLOW-ID-04
External identity authentication no implica account-link authorization.

## 365. Security Invariants — Runtime

AUTH-FLOW-RT-01
Transaction context nunca vive en static mutable global state.
AUTH-FLOW-RT-02
FrankenPHP workers aíslan transactions.

- AUTH-FLOW-RT-03
- Fiber concurrency no comparte challenge state.
- AUTH-FLOW-RT-04

Challenge consumption es atómico.
AUTH-FLOW-RT-05
Distributed execution no depende de sticky sessions.

## 366. Security Invariants — Completion

AUTH-COMPLETE-01
Requirements se verifican antes de completion.

- AUTH-COMPLETE-02
- Final security state puede reevaluarse.
- AUTH-COMPLETE-03
- Completion es idempotente.
- AUTH-COMPLETE-04

Successful challenge no implica successful transaction.
AUTH-COMPLETE-05
Successful transaction produce exactamente el tipo de completion autorizado por su flow.

## 367. Anti-Pattern

/login controller
contains all auth logic

## 368. Anti-Pattern

if MFA:

```php
redirect('/totp')
sin transaction.
```

## 369. Anti-Pattern

session['pending_user'] = $user
como único modelo de flujo.

## 370. Anti-Pattern

return_url from query string
→ redirect directly

## 371. Anti-Pattern

Passkey failed
→ automatically use SMS
sin policy negotiation.

## 372. Anti-Pattern

password correct
→ login complete
sin reevaluar requirements.

## 373. Anti-Pattern

challenge verified
→ transaction completed

## 374. Anti-Pattern

same challenge usable twice

## 375. Anti-Pattern

same reauth proof
→ any sensitive operation

## 376. Anti-Pattern

frontend chooses assurance level

## 377. Anti-Pattern

frontend says WebAuthn unavailable
→ server bypasses requirement

## 378. Anti-Pattern

IdP unavailable
→ disable MFA temporarily

## 379. Anti-Pattern

successful recovery
→ unrestricted admin session

## 380. Anti-Pattern

currentTransaction static property
bajo FrankenPHP.

## 381. Anti-Pattern

all callbacks accepted after transaction completion

## 382. Anti-Pattern

challenge state stored only in browser
sin integridad/protección adecuada.

## 383. Componentes principales

AuthenticationTransaction
AuthenticationTransactionFactory
AuthenticationTransactionRepository

AuthenticationRequirementGap

AuthenticationChallenge
AuthenticationChallengeDescriptor
AuthenticationChallengeProvider
AuthenticationChallengeHandler
AuthenticationChallengeRepository

AuthenticationChallengeNegotiator
AuthenticationChallengePlan

AuthenticationFlowEngine
AuthenticationFlowDefinition
AuthenticationFlowNode
AuthenticationFlowTransition

AuthenticationContinuation
AuthenticationContinuationResolver

AuthenticationIntent

AuthenticationCompletion
AuthenticationCompletionService

AuthenticationChallengePresenter
AuthenticationTransportAdapter

## 384. Namespace sugerido

VoltStack\Quantum\Auth\Flow
Subnamespaces:

- Contracts
- Transaction
- Challenge
- Negotiation
- Continuation
- Intent
- Completion
- Interaction
- Presentation
- Transport
- Runtime
- Storage
- Events
- Exceptions

## 385. Estructura sugerida

src/Quantum/Auth/Flow/
├── Contracts/
│   ├── AuthenticationFlowEngineInterface.php
│   ├── AuthenticationTransactionFactoryInterface.php
│   ├── AuthenticationTransactionRepositoryInterface.php
│   ├── AuthenticationChallengeNegotiatorInterface.php
│   ├── AuthenticationChallengeProviderInterface.php
│   ├── AuthenticationChallengeHandlerInterface.php
│   ├── AuthenticationChallengeRepositoryInterface.php
│   ├── AuthenticationChallengePresenterInterface.php
│   ├── AuthenticationContinuationResolverInterface.php
│   └── AuthenticationCompletionServiceInterface.php
│
├── Transaction/
│   ├── AuthenticationTransaction.php
│   ├── AuthenticationTransactionId.php
│   ├── AuthenticationTransactionStatus.php
│   ├── AuthenticationTransactionVersion.php
│   ├── AuthenticationTransactionRequest.php
│   └── AuthenticationFlowType.php
│
├── Challenge/
│   ├── AuthenticationChallenge.php
│   ├── AuthenticationChallengeId.php
│   ├── AuthenticationChallengeType.php
│   ├── AuthenticationChallengeStatus.php
│   ├── AuthenticationChallengeDescriptor.php
│   ├── AuthenticationChallengePayloadInterface.php
│   └── AuthenticationChallengeResponse.php
│
├── Negotiation/
│   ├── AuthenticationRequirementGap.php
│   ├── AuthenticationNegotiationContext.php
│   ├── AuthenticationChallengePlan.php
│   └── AuthenticationChallengeSelectionStrategy.php
│
├── Continuation/
│   ├── AuthenticationContinuation.php
│   ├── AuthenticationContinuationType.php
│   └── AuthenticationContinuationResolver.php
│
├── Intent/
│   ├── AuthenticationIntent.php
│   ├── AuthenticationIntentId.php
│   └── PayloadDigest.php
│
├── Completion/
│   ├── AuthenticationCompletion.php
│   ├── AuthenticationCompletionId.php
│   └── AuthenticationCompletionService.php
│
├── Interaction/
│   ├── InteractionMode.php
│   ├── AuthenticationClientCapabilitySet.php
│   └── AuthenticationFlowExecutionContext.php
│
├── Presentation/
│   ├── HtmlChallengePresenter.php
│   ├── SpaChallengePresenter.php
│   ├── ApiChallengePresenter.php
│   └── CliChallengePresenter.php
│
├── Runtime/
│   ├── AuthenticationFlowEngine.php
│   ├── AuthenticationContextReevaluationStep.php
│   └── AuthenticationFlowRuntimeResetter.php
│
├── Storage/
│   ├── DatabaseAuthenticationTransactionRepository.php
│   └── RedisAuthenticationTransactionRepository.php
│
├── Events/
│   └── ...

```text
│
└── Exceptions/
    └── ...
```

## 386. Integración global

REQUEST
│
▼
Authentication Entry Point
│
▼
POLICY ENGINE (36)
│
▼
Authentication Requirement
│
▼
ASSURANCE SYSTEM (37)
│
▼
Current Auth Context
│
▼
Requirement Satisfied?
│          │
YES         NO
│          │
▼          ▼
Continue   REQUIREMENT GAP
│
▼
CHALLENGE NEGOTIATOR
│
▼
Challenge Plan
│
▼
FLOW ENGINE (38)
│
▼
Challenge Presentation
│
┌─────────────┼─────────────┐
▼             ▼             ▼
HTML          SPA           API
│             │             │
└─────────────┼─────────────┘
▼
User Interaction
│
▼
Authenticator
│
▼
Authentication Evidence
│
▼
ASSURANCE SYSTEM (37)
│
▼
Updated Authentication Context
│
▼
POLICY ENGINE (36)
│
┌────────┼─────────┐
▼        ▼         ▼
COMPLETE   NEXT     DENY
│
└──► Challenge

## 387. Laravel comparison

Laravel proporciona mecanismos excelentes para:

- Authentication Guards
- Fortify
- password confirmation
- two-factor challenges
- Socialite
- redirect intended
- middleware

pero normalmente cada feature posee su propio flow.

- VoltStack pretende unificar:
- Login
- MFA
- Passkeys
- Federation
- Reauthentication
- Step-Up
- Recovery
- Credential Enrollment
- Account Linking
- Privileged Authentication
- Break-Glass

bajo la misma:

- Authentication Transaction Architecture
- sin perder una DX sencilla.

## 388. Symfony comparison

Symfony proporciona una arquitectura sólida mediante:

- Authenticators
- Passports
- Badges
- AuthenticationEntryPoint
- Security Tokens
- User Providers

VoltStack extenderá esa composición hacia un motor explícito de:

- Transactions
- Requirement Gaps
- Challenge Negotiation
- Interactive Continuations
- Dynamic Step-Up
- Operation Binding
- Multi-Transport Presentation
- Distributed Challenge State

## 389. Diferenciador VoltStack

Laravel-like Authentication DX
+
Symfony-like Authenticator Composition
+
Policy-Driven Requirements
+
Assurance-Aware Authentication
+
Unified Authentication Transactions
+
Challenge Negotiation
+
Dynamic Multi-Step Flows
+
Safe Continuations
+
Operation-Bound Step-Up
+
SPA-Native Authentication
+
HTML/API/CLI Transport Independence
+
Passkey/Federation/Recovery Integration
+
Distributed Transactions
+
Atomic Challenge Consumption
+
FrankenPHP-safe Runtime

## 390. Decisiones arquitectónicas definitivas

VoltStack adoptará:

1. Authentication flows complejos serán transactions.
2. Transaction y Session serán conceptos separados.
3. Transactions tendrán TTL.
4. Transactions tendrán state machine explícita.
5. Challenges pertenecerán a transactions.
6. Challenges serán opacos.
7. Challenges expirarán.
8. Challenges serán single-use por defecto.
9. Challenge verification y consumption serán estados distintos.
10. Challenge consumption será atómico.
11. Requirements procederán del Policy Engine.
12. Current Assurance procederá del Assurance System.
13. Negotiator calculará cómo cerrar el requirement gap.
14. Negotiation nunca reducirá requirements.
15. User preference será secundaria a security policy.
16. Client capabilities no serán Authentication Evidence.
17. Challenge selection será server-controlled.
18. Unsupported client capability podrá causar renegotiation, no bypass.
19. Successful challenge no implicará successful transaction.
20. Después de cada evidence podrá reevaluarse Authentication Context.
21. Dynamic Risk podrá modificar requirements durante flow.
22. Final requirements podrán reevaluarse antes de completion.
23. Continuations serán first-class.
24. Continuation nunca será arbitrary redirect sin validación.
25. Sensitive continuations podrán estar ligadas a operation intent.
26. Unsafe requests no se reproducirán automáticamente.
27. Login, step-up, recovery y linking tendrán flow types distintos.
28. Federation callbacks estarán ligados a transaction.
29. Recovery podrá producir restricted context.
30. Break-glass utilizará flow explícito.
31. Out-of-band authentication será soportada.
32. Device-code/CLI flows serán posibles.
33. Flow domain será independiente de HTML.
34. SPA será transport nativo, no workaround.
35. Multi-tab transactions estarán aisladas.
36. Transaction transitions serán concurrency-safe.
37. Distributed runtime no dependerá de sticky sessions.
38. Completion será idempotente.
39. Existing transactions conservarán flow version.
40. Critical security changes podrán revocar flows antiguos.
41. Transaction runtime state y audit state estarán separados.
42. Secrets no se persistirán dentro de transaction state.
43. Plugins producirán evidence; no podrán declarar assurance arbitrariamente.
44. FrankenPHP no conservará transaction context global mutable.
45. Fiber/request isolation será obligatorio.
46. Flow Engine no duplicará Authenticator logic.
47. Flow Engine no duplicará Policy Engine.
48. Flow Engine no calculará Assurance por su cuenta.
49. Challenge Presentation no tomará decisiones de seguridad.
50. Cryptographic state protection se centralizará en el sistema 39.
51. Criterios de aceptación

El sistema estará completo cuando soporte:
52. Authentication Transactions;
53. transaction IDs;
54. transaction TTL;
55. transaction state machine;
56. flow types;
57. requirement gaps;
58. challenge descriptors;
59. challenge instances;
60. challenge states;
61. challenge expiration;
62. challenge cancellation;
63. challenge supersession;
64. atomic consumption;
65. challenge binding;
66. challenge negotiation;
67. multiple candidates;
68. user choice;
69. policy preference;
70. sequential challenges;
71. requirement coverage;
72. no-valid-challenge handling;
73. reauthentication;
74. step-up;
75. password flows;
76. TOTP flows;
77. passkey flows;
78. federation flows;
79. recovery flows;
80. linking flows;
81. enrollment flows;
82. privileged flows;
83. break-glass flows;
84. out-of-band flows;
85. device-code flows;
86. CLI flows;
87. non-interactive compatibility;
88. continuations;
89. redirect validation;
90. operation intents;
91. payload digest binding;
92. safe retries;
93. SPA challenge protocol;
94. HTML presentation;
95. API presentation;
96. CLI presentation;
97. client capability negotiation;
98. fallback renegotiation;
99. multi-tab isolation;
100. optimistic concurrency;
101. distributed storage;
102. idempotent callbacks;
103. idempotent completion;
104. dynamic requirement reevaluation;
105. TOCTOU protection;
106. security epoch integration;
107. flow versioning;
108. events;
109. audit;
110. metrics;
111. tracing;
112. extensibility;
113. FrankenPHP safety;
114. Fiber safety;
115. security testing;
116. concurrency testing;
117. distributed testing.
118. Regla arquitectónica final

La arquitectura completa deberá mantener:

```text
Authentication Requirement
          │
          ▼
 Current Authentication Context
          │
          ▼
    Requirement Gap
          │
          ▼
 Challenge Negotiation
          │
          ▼
 Authentication Transaction
          │
          ▼
       Challenge
          │
          ▼
      Interaction
          │
          ▼
      Authenticator
          │
          ▼
 Authentication Evidence
          │
          ▼
 Assurance Re-Evaluation
          │
          ▼
 Requirement Re-Evaluation
          │
     ┌────┼─────┐
     ▼    ▼     ▼
   NEXT COMPLETE FAIL
     │
     ▼
Challenge
```

La regla fundamental será:
VoltStack no deberá pensar en Authentication como “mostrar login y validar credenciales”, sino como una transacción de seguridad capaz de negociar, presentar, verificar y encadenar challenges hasta demostrar exactamente la evidencia exigida por la política vigente.

Esto deja una separación especialmente importante:

```text
36 — Policy Engine
¿Qué se necesita?

37 — Assurance System
¿Qué se ha demostrado?

38 — Challenge & Flow System
¿Cómo conseguimos de forma interactiva
```

lo que todavía falta?
Siguiente documento
El siguiente apartado contemplado es:
`39_AUTHENTICATION_TRANSACTION_STATE_NONCE_REPLAY_PROTECTION_CSRF_BINDING_AND_CRYPTOGRAPHIC_CONTINUATION_SECURITY_SYSTEM.md`
Será la capa criptográfica que protegerá la infraestructura creada en 38:

```php
Authentication Transaction
          │
          ├── Transaction State
          ├── Challenge State
          ├── Nonces
          ├── CSRF Binding
          ├── PKCE
          ├── Federation State
          ├── Continuation State
          ├── Operation Intent
          └── Resume References
                   │
                   ▼
          CRYPTOGRAPHIC PROTECTION
El 38 define cómo funciona la transacción; el 39 definirá cómo impedir que esa transacción sea falsificada, intercambiada, repetida, secuestrada o reutilizada por un atacante.
```
