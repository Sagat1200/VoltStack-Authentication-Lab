# VoltStack Authentication System

## 39 — Authentication Transaction State, Nonce, Replay Protection, CSRF Binding and Cryptographic Continuation Security System

- **Archivo:** `39_AUTHENTICATION_TRANSACTION_STATE_NONCE_REPLAY_PROTECTION_CSRF_BINDING_AND_CRYPTOGRAPHIC_CONTINUATION_SECURITY_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Cryptographic Protocol Infrastructure
- **Dependencias principales:** 08, 16, 17, 19, 22, 25, 27, 29, 31, 32, 34, 36, 37, 38.

---

## 1. Propósito

Este documento define la infraestructura de seguridad que protege el estado transitorio utilizado durante Authentication.
El documento 38 estableció:

```text
Authentication Requirement
        ↓
Authentication Transaction
        ↓
Challenge
        ↓
Interaction
        ↓
Evidence
        ↓
Continuation
```

El presente sistema deberá garantizar que ninguno de esos elementos pueda ser:

- forged
- replayed
- swapped
- stolen and reused
- cross-bound
- cross-tenant reused
- cross-realm reused
- cross-operation reused
- silently modified

resumed outside its context
La arquitectura resultante será:

```text
Authentication Transaction
          │
          ▼
Security Envelope
          │
          ├── State Binding
          ├── Nonce
          ├── Anti-Replay
          ├── CSRF Binding
          ├── Flow Purpose
          ├── Principal Binding
          ├── Session Binding
          ├── Tenant Binding
          ├── Realm Binding
          ├── Client Binding
          ├── Operation Binding
          ├── Payload Binding
          ├── Expiration
          ├── Single-Use Semantics
          └── Cryptographic Integrity
```

## 2. Principio fundamental

Ningún valor controlado por el cliente deberá convertirse en Authentication State únicamente porque el cliente lo devuelve sin modificaciones aparentes.

VoltStack deberá poder demostrar:

- Who created it?
- For which transaction?
- For which purpose?
- For which principal?
- For which session?
- For which tenant?
- For which realm?
- For which operation?

When was it created?
Has it expired?
Has it already been used?
Has it been modified?

## 3. Problema

Un flujo aparentemente correcto:

```text
Login
  ↓
redirect IdP
  ↓
callback
  ↓
continue
```

puede ser vulnerable si no existe binding criptográfico.

```text
Ejemplo:
Attacker starts OAuth login
        ↓
obtains callback/state
        ↓
```

injects callback into victim browser
↓
victim session becomes associated
with attacker's external identity
El mismo problema aparece en:

- OAuth/OIDC
- Passkeys
- MFA
- Password confirmation
- Account linking
- Recovery
- Step-Up
- Privileged authentication
- Device approval
- Magic links
- Email verification
- CLI device flows
- Break-glass workflows

## 4. Security State Model

VoltStack utilizará un modelo explícito:

```text
AuthenticationTransaction
        │
        ├── Transaction State
        ├── Challenge State
        ├── Security Binding
        ├── Nonce Set
        ├── Replay State
        ├── Continuation
        └── Cryptographic Envelope
```

## 5. Transaction State

AuthenticationTransaction continuará siendo el objeto lógico definido en 38.
Este documento añade su estado de seguridad.

## 6. AuthenticationTransactionSecurityState

final readonly class AuthenticationTransactionSecurityState
{
public function __construct(
public AuthenticationTransactionId $transactionId,
public AuthenticationFlowPurpose $purpose,
public AuthenticationSecurityBinding $binding,
public AuthenticationNonceSet $nonces,
public AuthenticationReplayState $replay,
public AuthenticationStateVersion $version,
public DateTimeImmutable $issuedAt,
public DateTimeImmutable $expiresAt,
) {}
}

## 7. State != Browser Session

El estado de Authentication no deberá depender exclusivamente de:
$_SESSION['auth_state'];
Una browser Session puede participar como binding, pero:

- Authentication Transaction State
- ≠
- PHP Session State

## 8. Security Binding

Concepto central:

```php
final readonly class AuthenticationSecurityBinding
{
    public function __construct(
        public PrincipalBinding|null $principal,
        public SessionBinding|null $session,
        public TenantBinding|null $tenant,
        public RealmBinding $realm,
        public ClientBinding|null $client,
        public OperationBinding|null $operation,
        public FlowBinding $flow,
    ) {}
}
```

## 9. Binding Dimensions

Una transaction podrá quedar vinculada a:

- principal
- session
- tenant
- realm
- application
- flow purpose
- challenge
- client
- device
- origin
- operation
- resource
- payload
- provider
- redirect target

No todos aplican a todos los flows.

## 10. Binding Strength

VoltStack no deberá utilizar un único boolean:

```php
bound = true
Debe conocer a qué está ligado el estado.
```

## 11. Flow Purpose

Toda transaction deberá tener purpose explícito.

```php
enum AuthenticationFlowPurpose: string
{
    case Login = 'login';
    case Reauthenticate = 'reauthenticate';
    case StepUp = 'step_up';
    case Recovery = 'recovery';
    case CredentialEnrollment = 'credential_enrollment';
    case CredentialRemoval = 'credential_removal';
    case AccountLinking = 'account_linking';
    case Privileged = 'privileged';
    case BreakGlass = 'break_glass';
    case Federation = 'federation';
}
```

## 12. Purpose Confusion

Debe impedirse:

```text
LOGIN state
     ↓
ACCOUNT LINKING callback
```

o:

```text
RECOVERY proof
     ↓
PRIVILEGED reauthentication
```

## 13. Fundamental Rule

Proof valid for Purpose A is not automatically valid for Purpose B.

## 1. FlowBinding

final readonly class FlowBinding
{
public function __construct(
public AuthenticationFlowPurpose $purpose,
public AuthenticationFlowDefinitionVersion $version,
) {}
}

## 2. Principal Binding

Cuando la identity ya se conoce:

```php
final readonly class PrincipalBinding
{
    public function __construct(
        public PrincipalId $principalId,
        public PrincipalType $type,
        public int|string|null $securityEpoch,
    ) {}
}
```

## 3. Anonymous Transactions

Inicialmente:

```php
principal = unknown
puede ser válido.
```

Después de identity resolution:

```text
transaction
    ↓
bind principal
```

## 4. Principal Binding Immutability

Una vez ligada a:
Alice
no podrá transformarse silenciosamente en:
Bob

## 5. Session Binding

Reauthentication, step-up y operaciones sensibles normalmente deberán ligarse a Session.
final readonly class SessionBinding
{
public function __construct(
public SessionId $sessionId,
public int|string|null $sessionEpoch,
) {}
}

## 6. Session ID Exposure

La referencia utilizada en binding puede ser interna.
No es necesario revelar el Session ID real al browser.

## 7. Session Revocation

Si Session vinculada es revocada:

```text
reauthentication transaction
        ↓
invalid
por defecto.
```

## 8. Session Rotation

Cuando se rota Session ID después de login:
pre-authentication binding
y:

- post-authentication session
- deben manejarse explícitamente.

No copiar ciegamente el binding anterior.

## 9. Tenant Binding

final readonly class TenantBinding
{
public function __construct(
public TenantId $tenantId,
public int|string|null $securityEpoch,
) {}
}

## 10. Cross-Tenant Replay

Un proof emitido para:
Tenant A
no podrá autorizar:

- Tenant B
- salvo flujo explícitamente global.

## 11. Realm Binding

final readonly class RealmBinding
{
public function __construct(
public AuthenticationRealmId $realm,
) {}
}

## 12. Cross-Realm Replay

Especialmente:

- USER_REALM
- ADMIN_REALM
- PLATFORM_ADMIN_REALM
- SECURITY_ADMIN_REALM
- deberán permanecer aislados.

## 13. User → Admin Escalation

No:

```text
successful user login proof
        ↓
```

reuse as admin reauthentication
sin policy explícita.

## 14. Application Binding

Cuando Authentication sirve varias aplicaciones:

- App A
- App B
- Admin Console
- Mobile
- CLI

el estado puede ligarse a application_id.

## 15. Client Binding

final readonly class ClientBinding
{
public function __construct(
public ClientId|null $clientId,
public ClientInstanceId|null $instanceId,
) {}
}

## 16. Client Binding Limitations

No depender de:

- User-Agent
- IP

como binding criptográfico fuerte.

## 17. IP Address

Puede participar en Risk.
No debe ser Authentication Proof.

## 18. User-Agent

Puede servir para observabilidad/risk.
Es controlado por el cliente.

## 19. Device Binding

Trusted Device del documento 21 puede participar.

- Pero:
- device trust
- ≠
- transaction integrity

## 20. Operation Binding

Documento 32 introdujo sensitive operations.

```php
final readonly class OperationBinding
{
    public function __construct(
        public OperationId $operation,
        public ResourceReference|null $resource,
        public PayloadDigest|null $payloadDigest,
    ) {}
}
```

## 21. Example

Operation:
tenant.delete

Resource:
tenant:42

Payload:
confirmation=DELETE
El step-up result deberá quedar ligado a ese intent cuando policy así lo exija.

## 35. Payload Binding

Para operaciones críticas:

- authentication proof
- puede ligarse al contenido concreto que el usuario autorizó.

## 36. PayloadDigest

final readonly class PayloadDigest
{
public function __construct(
public string $algorithm,
public string $digest,
public string $canonicalizationVersion,
) {}
}

## 37. Do Not Hash Raw Arbitrarily

No:

```php
hash('sha256', json_encode($request->all()));
como contrato de seguridad sin canonicalización definida.
```

## 38. Canonicalization

Debe especificar:

- field ordering
- encoding
- normalization
- included fields
- excluded fields
- numeric representation
- Unicode handling
- null semantics

## 39. Canonical Payload

Ejemplo conceptual:

```php
operation=tenant.delete
tenant=42
confirmation=DELETE
version=1
```

## 40. WYSIWYS

Permite aproximarse a:
What You See Is What You Sign.

```text
Especialmente útil para:
payments
key rotation
credential deletion
tenant deletion
privilege escalation
security configuration
```

## 41. Nonce

Un nonce será un valor de alta entropía usado para ligar o identificar una ejecución única.

## 42. Nonce != Token

Nonce no necesariamente concede autoridad.

## 43. Nonce Requirements

Debe ser:

- cryptographically random
- sufficient entropy
- purpose-specific
- short-lived
- single-context

## 44. Nonce Generator

interface AuthenticationNonceGeneratorInterface
{
public function generate(
AuthenticationNoncePurpose $purpose
): AuthenticationNonce;
}

## 45. AuthenticationNonce

final readonly class AuthenticationNonce
{
public function __construct(
public AuthenticationNonceId $id,
public AuthenticationNoncePurpose $purpose,
public string $value,
public DateTimeImmutable $issuedAt,
public DateTimeImmutable $expiresAt,
) {}
}

## 46. Nonce Purpose

enum AuthenticationNoncePurpose: string
{
case Transaction = 'transaction';
case Challenge = 'challenge';
case Oidc = 'oidc';
case OAuthState = 'oauth_state';
case Csrf = 'csrf';
case Continuation = 'continuation';
case Resume = 'resume';
case Operation = 'operation';
case DeviceFlow = 'device_flow';
case SignedRequest = 'signed_request';
}

## 47. Domain Separation

Nonces de distintos propósitos no deberán reutilizarse.

```text
No:
same random value
→ OAuth state
→ OIDC nonce
→ CSRF token
```

## 48. AuthenticationNonceSet

final readonly class AuthenticationNonceSet
{
public function __construct(
public AuthenticationNonce|null $transaction,
public AuthenticationNonce|null $challenge,
public AuthenticationNonce|null $oidc,
public AuthenticationNonce|null $csrf,
public AuthenticationNonce|null $continuation,
) {}
}

## 49. Cryptographic Randomness

Utilizar CSPRNG.
En PHP:

```php
random_bytes();
es una base apropiada.
```

## 50. No Predictable Nonces

Nunca:
$nonce = time() . $userId;

## 51. No UUID Assumption

No asumir que cualquier UUID es automáticamente un nonce seguro.
Depende de su versión/generación.

## 52. Entropy Policy

Centralizar requerimientos de longitud/entropy.

## 53. AuthenticationNoncePolicy

final readonly class AuthenticationNoncePolicy
{
public function __construct(
public int $minimumBits,
public DateInterval $maximumLifetime,
public bool $singleUse,
) {}
}

## 54. Replay Attack

Replay:

```text
Attacker captures valid proof
           ↓
```

uses same proof again
↓
server accepts again

## 55. Replay Protection Layers

VoltStack utilizará defensa en profundidad:

- nonce
- +
- expiration
- +
- transaction state
- +
- challenge state
- +
- single-use consumption
- +
- context binding
- +
- security epoch
- +
- idempotency
- +
- protocol-specific replay defense

## 56. AuthenticationReplayState

final readonly class AuthenticationReplayState
{
public function__construct(
public bool $consumed,
public DateTimeImmutable|null $consumedAt,
public ReplayIdentifier|null $identifier,
) {}
}

## 57. Replay Identifier

Puede derivarse de:

- challenge ID
- nonce ID
- jti
- authorization code
- assertion identifier
- operation proof ID
- dependiendo del protocolo.

## 58. Replay Store

interface AuthenticationReplayStoreInterface
{
public function consume(
ReplayIdentifier $identifier,
DateTimeImmutable $expiresAt
): ReplayConsumptionResult;
}

## 59. Atomic Consume

Semántica requerida:

```text
first caller:
CONSUMED
```

second caller:
ALREADY_CONSUMED

## 60. Distributed Atomicity

Debe funcionar entre:

- Node A
- Node B
- Node C

## 61. Redis Example

Conceptualmente:

```php
SET replay:{id} consumed NX EX <ttl>
aunque implementación concreta puede variar.
```

## 62. Database Example

Constraint única o conditional atomic update.

## 63. Race Attack

Dos requests simultáneos con mismo challenge:

```text
Request A ──┐
            ├── same proof
Request B ──┘
```

solo uno deberá producir transición válida cuando el protocolo sea single-use.

## 64. Replay Window

No esperar hasta expiration para detectar replay.

## 65. Challenge Consumption

Documento 38:

```text
VERIFIED
   ↓
CONSUMED
```

Este documento exige que CONSUMED sea atomic security state.

## 66. Evidence Replay

Aunque challenge haya sido consumido, el authenticator deberá aplicar defensas específicas.
Ejemplos:

- TOTP replay
- OIDC token replay
- signed request replay
- WebAuthn challenge replay
- OAuth code reuse

## 67. Protocol Layering

Generic Replay Protection
+
Protocol-Specific Replay Protection

## 68. TOTP

Dependiendo de policy:
same valid TOTP timestep
puede rechazarse después del primer uso para la misma security context.

## 69. WebAuthn

El challenge debe ser:

- fresh
- random

origin-bound by protocol validation
RP-bound
single-use
transaction-bound

## 70. OAuth Authorization Code

Debe tratarse como single-use según protocolo/provider.
VoltStack no deberá depender únicamente del provider para su transaction-level replay protection.

## 71. OIDC Nonce

OIDC nonce tiene purpose específico:

```text
authorization request
        ↓
ID Token
        ↓
nonce claim validation
```

## 72. OIDC Nonce != OAuth State

Ambos protegen amenazas diferentes.

## 73. OAuth State

state deberá ligar:

```text
outbound authorization request
          ↓
callback
```

y proteger contra request/callback confusion y CSRF según flow.

## 74. State Requirements

state deberá ser:

- unguessable
- transaction-bound
- purpose-bound
- provider-bound
- short-lived
- single-use where possible

## 75. State Payload

Preferentemente no colocar datos sensibles directamente.

## 76. Opaque State

Modelo recomendado:

```php
state = random opaque reference
que apunta a server-side transaction state.
```

## 77. Signed State

También puede soportarse:

- signed/encrypted state envelope
- para casos apropiados.

## 78. Stateful vs Stateless

VoltStack soportará ambos modelos:

- Opaque Stateful State
- Cryptographically Protected Stateless State

## 79. Default Preference

Para Authentication sensible:
Stateful opaque references serán preferidas cuando revocación, single-use y concurrency control sean importantes.

## 1. Stateless State Advantages

less shared storage
easy horizontal scaling

## 2. Stateless State Risks

harder revocation
larger tokens
data disclosure if only signed
replay control still needs state
key rotation complexity

## 3. Signed != Encrypted

Una firma protege:

- integrity
- authenticity

pero no:
confidentiality

## 4. Encrypted State

Si contiene información confidencial:

- authenticated encryption
- deberá utilizarse.

## 5. Never Custom Crypto

VoltStack no deberá inventar:

- XOR encryption
- custom cipher
- home-grown MAC

## 6. Cryptographic Service

Integración con documento 31.

```php
interface AuthenticationStateProtectorInterface
{
    public function protect(
        AuthenticationProtectedState $state
    ): ProtectedAuthenticationState;

    public function unprotect(
        ProtectedAuthenticationState $protected
    ): AuthenticationProtectedState;
}
```

## 7. Protection Purpose

La cryptographic key deberá estar separada por purpose.

- Ejemplos:
- AUTH_TRANSACTION_STATE
- AUTH_CONTINUATION_STATE
- AUTH_RESUME_TOKEN
- AUTH_OAUTH_STATE
- AUTH_DEVICE_FLOW_STATE

## 8. Key Purpose Separation

No usar la misma key indiscriminadamente para:

- session cookies
- OAuth state
- password reset
- API tokens
- continuations

## 9. Key Management

Documento 31 gobierna:

- generation
- storage
- rotation
- revocation
- KMS
- HSM
- versioning
- retirement

## 10. Protected State Header

Puede incluir metadata no sensible:

- version
- key id
- algorithm suite
- purpose

## 11. Algorithm Agility

No hardcodear la arquitectura a un único algoritmo.

## 12. Algorithm Confusion

El servidor deberá determinar qué algoritmos son permitidos.

- Nunca confiar en:
- alg supplied by attacker
- sin allowlist estricta.

## 13. Key ID

kid puede seleccionar key conocida.

- No puede permitir:
- arbitrary URL
- filesystem path
- remote key source

## 14. Unknown Key ID

Debe producir:

- AUTH_STATE_UNKNOWN_KEY
- con refresh controlado si aplica.

## 15. State Version

final readonly class AuthenticationStateVersion
{
public function __construct(
public int $value,
) {}
}

## 16. Versioning Purpose

Permite evolucionar:

- serialization
- bindings
- cryptographic envelope
- canonicalization

## 17. Old State

Puede:

- remain temporarily valid
- require migration
- be revoked
- según security policy.

## 18. Critical Security Migration

Una vulnerabilidad puede exigir:

```php
minimum_state_version = 4
invalidando versiones anteriores.
```

## 19. CSRF

CSRF aplica especialmente cuando Authentication utiliza ambient authority como cookies.

## 20. Important Distinction

Authentication flow security no se resuelve únicamente con CSRF token.
Necesita:

```text
CSRF

+

transaction binding
+
purpose binding
+
challenge binding
+
continuation integrity
```

## 100. CSRF Binding

final readonly class AuthenticationCsrfBinding
{
public function __construct(
public CsrfTokenId $tokenId,
public AuthenticationTransactionId $transaction,
public AuthenticationFlowPurpose $purpose,
) {}
}

## 101. Generic Application CSRF

VoltStack puede reutilizar el CSRF subsystem general.
Pero deberá permitir transaction-level binding para operaciones Authentication sensibles.

## 102. Login CSRF

Login también puede sufrir CSRF.

```text
Ataque conceptual:
attacker authenticates own account
        ↓
```

forces victim browser to complete login
↓
victim unknowingly operates attacker account

## 103. Login CSRF Protection

Login transaction deberá estar ligada al browser/client interaction apropiado.

## 104. Account Linking CSRF

Más crítico:

```text
victim logged in
       ↓
```

attacker causes external linking callback
↓
attacker identity linked to victim account
Debe impedirse.

## 105. Linking State

Debe contener o referenciar:

```php
purpose = ACCOUNT_LINKING
current authenticated identity
session
provider
transaction
nonce
expiration
```

## 106. Login State Cannot Link

Aunque callback sea criptográficamente válido para:
LOGIN
debe rechazarse para:
ACCOUNT_LINKING

## 107. Reauthentication CSRF

Reauthentication transaction deberá estar ligada al sensitive intent que la originó cuando sea necesario.

## 108. SameSite Cookies

Podrán ser parte de defense-in-depth.
No sustituyen CSRF architecture.

## 109. Origin Validation

Para browser flows, validar Origin cuando sea apropiado.

## 110. Referer

Puede ser defense-in-depth.
No deberá ser única defensa.

## 111. Cross-Origin Authentication

Federated redirects requieren cruces de origen legítimos.
Por ello, las reglas deberán distinguir:
application form submission
de:
federation callback

## 112. PKCE

PKCE será obligatorio/recomendado según protocolo y client type, particularmente OAuth/OIDC authorization flows.

## 113. PKCE Model

code_verifier
↓
transformation
↓
code_challenge
↓
authorization request
↓
authorization code
↓
token exchange + verifier

## 114. PKCE State

code_verifier es secreto temporal.

## 115. PKCE Storage

Debe almacenarse:
server-side
o dentro de un envelope confidencialmente protegido cuando la arquitectura lo permita.

## 116. Never Expose Verifier

No:

- query string
- logs
- metrics
- traces
- browser-visible state
- innecesariamente.

## 117. PKCE Binding

Verifier deberá estar ligado a:

- transaction
- provider
- authorization request
- redirect URI
- client
- purpose
- según contexto.

## 118. S256

Cuando protocolo/provider soporte estándares modernos, preferir challenge transformation segura como S256, no downgrade silencioso.

## 119. PKCE != State

PKCE no sustituye OAuth state.

## 120. State != Nonce

OIDC nonce tampoco sustituye state.

## 121. Defense Composition

OAuth/OIDC flow

Transaction
+
State
+
PKCE
+
OIDC Nonce
+
Redirect URI validation
+
Issuer validation
+
Audience validation
+
Code single-use
+
Continuation binding

## 122. Redirect URI

Debe estar pre-registered/configured.

## 123. Dynamic Redirects

No aceptar:

- redirect_uri supplied by arbitrary request
- sin allowlist/resolution segura.

## 124. Post-Authentication Continuation

Separar:
OAuth redirect URI
de:
application continuation after authentication

## 125. Open Redirect

Nunca:

```text
/auth/login?next=<https://evil.example>
→ redirect automático.
```

## 126. Continuation Security

Documento 38 definió AuthenticationContinuation.
Este sistema añade integridad.

## 127. ProtectedAuthenticationContinuation

final readonly class ProtectedAuthenticationContinuation
{
public function __construct(
public ContinuationId $id,
public AuthenticationTransactionId $transaction,
public AuthenticationContinuationType $type,
public ContinuationTarget $target,
public AuthenticationSecurityBinding $binding,
public DateTimeImmutable $issuedAt,
public DateTimeImmutable $expiresAt,
) {}
}

## 128. Continuation Target

Preferencia:

- RouteReference
- OperationReference
- InternalUriReference
- SpaIntentReference
- OAuthAuthorizationReference

en vez de string URL genérico.

## 129. RouteReference

final readonly class RouteContinuationTarget
{
public function __construct(
public RouteName $route,
public array $parameters,
) {}
}

## 130. Route Validation

Resolver route server-side.

## 131. Internal URI

Si se permiten URIs:

- scheme
- host
- port
- path
- deben validarse.

## 132. Relative URLs

Pueden ser preferibles para browser application continuation.

## 133. Continuation Lifetime

Nunca mayor que la transaction salvo razón explícita.

## 134. Continuation Single-Use

Sensitive continuation deberá ser consumible una vez.

## 135. Continuation Replay

complete step-up
↓
capture continuation
↓
replay later
debe fallar si single-use.

## 136. Continuation Store

interface AuthenticationContinuationStoreInterface
{
public function consume(
ContinuationId $id,
AuthenticationContinuationConsumptionContext $context
): AuthenticationContinuation;
}

## 137. Continuation Validation

Antes de consumir:

- exists
- not expired
- not consumed
- transaction valid
- purpose matches
- principal matches
- session matches
- tenant matches
- realm matches
- operation matches
- security epochs valid

## 138. Cryptographic Continuation Token

Puede existir como:
opaque reference
o:
protected token

## 139. Stateless Continuation

Si stateless:

- signature/AEAD
- expiration
- purpose
- binding
- son obligatorios.

## 140. Stateless Replay Problem

Firma válida no impide replay.
Para single-use sensitive continuation seguirá necesitándose replay state.

## 141. Resume Reference

Out-of-band flows necesitan reanudar transaction.

## 142. AuthenticationResumeReference

final readonly class AuthenticationResumeReference
{
public function __construct(
public ResumeReferenceId $id,
public AuthenticationTransactionId $transaction,
public AuthenticationChallengeId|null $challenge,
public AuthenticationFlowPurpose $purpose,
public DateTimeImmutable $expiresAt,
) {}
}

## 143. Resume Reference != Transaction ID

No exponer Transaction ID y asumir que basta para reanudar.

## 144. Resume Capability

Una resume reference puede ser capability temporal.
Debe protegerse acorde al poder que concede.

## 145. Resume Binding

Puede ligarse a:

- client
- device
- browser interaction
- channel
- user code

## 146. Device Flow

Modelo:

```text
Device
  │
  ├── device_code
  └── user_code

Browser
  │
  └── user enters user_code
```

## 147. Device Code

Debe tener alta entropía.

## 148. User Code

Puede tener menor longitud por usabilidad, compensado con:

- short TTL
- rate limiting
- polling limits
- attempt limits
- binding

## 149. User Code Enumeration

Debe protegerse contra brute force.

## 150. Device Flow Binding

Cuando browser aprueba:

```text
user_code
    ↓
transaction
    ↓
specific device request
```

## 151. Approval Confusion

UI deberá mostrar suficiente información para evitar aprobar dispositivo equivocado.

## 152. Out-of-Band Approval

Para:

- push approval
- email link
- mobile confirmation

usar references específicas y short-lived.

## 153. Magic Links

Si VoltStack soporta magic links:

- link possession
- es Authentication Evidence de un tipo determinado.

No asumir automáticamente assurance alta.

## 154. Magic Link Token

Debe ser:

- random
- short-lived
- single-use
- purpose-bound
- principal-bound
- flow-bound

## 155. Magic Link URL Leakage

Considerar:

- browser history
- Referer
- email scanners
- proxy logs
- analytics

## 156. Email Scanner Problem

Security scanners pueden abrir links automáticamente.
No diseñar destructive completion únicamente por GET.

## 157. Recommended Pattern

GET magic link
↓
validate token
↓
show confirmation
↓
POST consume
cuando threat model lo requiera.

## 158. Recovery Links

Deben utilizar namespace/purpose separado de login magic links.

## 159. Password Reset Token

Aunque recovery subsystem tenga su propio modelo, deberá obedecer:

- purpose
- principal
- expiration
- single-use
- security epoch

## 160. Security Epoch

Uno de los mecanismos de invalidación global.

## 161. Epoch Types

Podrán existir:

- identity_security_epoch
- session_epoch
- credential_epoch
- tenant_security_epoch
- provider_security_epoch
- realm_security_epoch

## 162. Epoch Snapshot

Transaction puede capturar epochs relevantes.

## 163. Example

Transaction created:

```text
identity epoch = 17
session epoch  = 8
```

tenant epoch   = 3
Después:

- password compromised
- identity epoch = 18

La transaction puede quedar inválida.

## 164. Epoch Policy

No todo cambio exige invalidar todo.
Debe definirse por security event.

## 165. Epoch Resolver

interface AuthenticationSecurityEpochResolverInterface
{
public function current(
AuthenticationSecurityBinding $binding
): AuthenticationSecurityEpochSet;
}

## 166. Epoch Validation

Antes de critical completion:

```text
captured epochs
        vs
current epochs
```

## 167. Credential Version

Puede utilizarse para invalidar proofs asociados a una credential específica.

## 168. Provider Security Epoch

Si un federated provider es comprometido:

- provider epoch++
- puede invalidar transactions relacionadas.

## 169. Clock

Expiration depende del tiempo.

## 170. Clock Interface

interface AuthenticationClockInterface
{
public function now(): DateTimeImmutable;
}

## 171. Testability

Nunca dispersar:

```php
new DateTimeImmutable();
time();
```

por todo el Core.

## 172. Clock Skew

Protocolos distribuidos pueden permitir skew limitado.

## 173. Skew Policy

final readonly class AuthenticationClockSkewPolicy
{
public function __construct(
public DateInterval $maximumPastSkew,
public DateInterval $maximumFutureSkew,
) {}
}

## 174. Future-Dated State

State con iat excesivamente futuro debe rechazarse.

## 175. Expired State

Siempre reject.

## 176. Boundary Conditions

Definir claramente:

```php
now == expiresAt
como expired o no.
```

Preferencia:

```text
now >= expiresAt
→ expired
```

## 177. Token Identifiers

Protected state podrá tener:

- jti
- o equivalente.

## 178. JTI

Sirve como unique identifier/replay reference.
No sustituye expiration ni signature.

## 179. Token Audience

Protected state deberá tener audience cuando pueda cruzar componentes.

## 180. Example

aud = voltstack-auth-continuation

## 181. Issuer

Puede declarar:

```php
iss = auth.example.internal
según arquitectura.
```

## 182. Subject

Solo cuando semánticamente corresponda.

## 183. Purpose Claim

Más importante que reutilizar ambiguamente sub.
Ejemplo:
purpose = account_linking

## 184. Protected State Claims

Modelo conceptual:

- version
- purpose
- transaction_id
- binding_reference
- iat
- nbf
- exp
- jti
- audience
- key_id

## 185. Minimal State

No colocar:

- full user profile
- permissions
- roles
- secrets
- password hashes
- TOTP secret
- raw credential data

## 186. Serialization

Debe ser determinista cuando forme parte de MAC/signature input.

## 187. Serializer Interface

interface AuthenticationStateSerializerInterface
{
public function serialize(
AuthenticationProtectedState $state
): string;

public function deserialize(
string $payload
): AuthenticationProtectedState;
}

## 188. Schema Validation

Después de decrypt/verify:

```text
parse
    ↓
strict schema validation
    ↓
semantic validation
```

## 189. Cryptographic Validation != Semantic Validation

Un token correctamente firmado todavía puede ser inválido porque:

- expired
- wrong purpose
- wrong tenant
- wrong realm
- wrong session
- already consumed
- wrong operation

## 190. Validation Pipeline

Input
↓
Syntax Validation
↓
Cryptographic Validation
↓
Version Validation
↓
Purpose Validation
↓
Expiration Validation
↓
Binding Validation
↓
Epoch Validation
↓
Replay Validation
↓
Flow-State Validation
↓
Accept

## 191. Ordering

Evitar expensive/dynamic operations antes de validaciones baratas cuando sea seguro hacerlo.

## 192. Error Oracle

No devolver al atacante diferencias excesivamente detalladas entre:

- bad MAC
- unknown transaction
- expired token
- wrong user

cuando eso facilite probing.

## 193. Internal Error Detail

Internamente sí conservar reason code.

## 194. Constant-Time Comparison

Secret/MAC/token verifier comparisons deberán utilizar mecanismos constant-time apropiados.
En PHP, cuando corresponda:
hash_equals();

## 195. Do Not Compare Secrets Normally

No:

```php
if ($expected === $provided)
para valores donde timing leakage sea relevante.
```

## 196. Token Hashing

Opaque bearer-like references almacenadas server-side pueden persistirse hashed.

## 197. Example

Browser recibe:
resume_token = random secret
Servidor guarda:
hash(resume_token)

## 198. Database Leak Resistance

Así, database leak no necesariamente revela tokens activos.

## 199. Token Prefix

Puede existir public identifier + secret.
resume_abc123.<secret>

## 200. Public ID

Permite lookup eficiente sin almacenar secret en claro.

## 201. Token Parsing

Debe ser strict.

## 202. No Ambiguous Encoding

Elegir encoding canónico:

- base64url
- hex
- según contrato.

## 203. Padding

Definir semántica.

## 204. Unicode

Security tokens deberían utilizar alphabet restringido cuando sea posible.

## 205. URL Safety

Tokens en URLs deberán ser URL-safe.

## 206. Query String Risks

URLs pueden filtrarse a:

- logs
- history
- analytics
- Referer
- screenshots

## 207. Fragment

No asumir que URL fragment resuelve automáticamente toda amenaza.

## 208. Cookie Storage

Algunos state references pueden almacenarse en cookies.

## 209. Cookie Security

Aplicar según caso:

- Secure
- HttpOnly
- SameSite
- Path
- Domain
- Max-Age

## 210. Cookie Scope

No utilizar:

```php
Domain=.example.com
innecesariamente para Authentication state.
```

## 211. Host Prefix

Cuando plataforma/browser lo permita, patrones endurecidos de cookies pueden ser utilizados.

## 212. Cookie Fixation

Pre-authentication cookie identifiers deberán rotarse o gestionarse para impedir fixation.

## 213. Session Fixation

Successful authentication deberá integrar la estrategia de session regeneration definida en el subsystem correspondiente.

## 214. Transaction Fixation

No aceptar transaction ID elegido por el cliente.

## 215. Challenge Fixation

No aceptar challenge arbitrario creado por el cliente.

## 216. OAuth State Fixation

No reutilizar state proporcionado externamente.

## 217. Nonce Fixation

Servidor genera nonce.

## 218. Double Submit CSRF

Puede soportarse en infraestructura general.
Pero para critical Authentication flows, server-bound transaction state es preferible cuando sea posible.

## 219. CSRF Token Rotation

Puede rotarse por:

- session
- authentication event
- security policy

sin romper innecesariamente transactions legítimas.

## 220. Multiple Tabs

Cada transaction puede tener su propio CSRF binding.

## 221. Tab A / Tab B

Tab A → password change transaction
Tab B → provider linking transaction
No deben intercambiar tokens.

## 222. Browser Navigation Back

Token consumido continúa consumido aunque página quede en history.

## 223. Browser Retry

Retry de POST puede ocurrir.
Idempotency deberá distinguir:
safe duplicate transport retry
de:
security replay

## 224. Idempotency vs Replay

No son lo mismo.

## 225. Idempotency

Objetivo:

```text
same legitimate operation retried
→ same logical result
```

## 226. Replay Protection

Objetivo:

```text
same security proof reused
→ not grant authority twice
```

## 227. Idempotency Key

Puede ligarse a operation intent.

## 228. Authentication Proof Consumption

Aunque operation completion sea idempotente, proof puede quedar consumed.

## 229. Example

Delete credential request
↓
reauth proof consumed
↓
network timeout
↓
client retries with same idempotency key
Sistema puede devolver resultado previo sin reutilizar proof como nueva autoridad.

## 230. Security Action Record

Para ello puede existir:

```php
final readonly class AuthenticatedOperationExecution
{
    public function __construct(
        public AuthenticationIntentId $intent,
        public AuthenticationProofId $proof,
        public IdempotencyKey $idempotencyKey,
        public AuthenticationOperationExecutionStatus $status,
    ) {}
}
```

## 231. Continuation Transactionality

Critical sequence:

- consume proof
- +
- validate operation
- +
- record execution
- +
- perform security mutation

deberá diseñarse con atomicity adecuada.

## 232. Distributed Transactions

No siempre habrá ACID global.

## 233. Saga / Outbox

Podrán utilizarse:

- transactional outbox
- idempotent consumers
- operation records
- compensating action

## 234. But Authentication Grant Must Be Precise

No permitir que eventual consistency otorgue múltiples security grants.

## 235. Fail Closed

Si replay store crítico no está disponible:

```text
high-risk operation
→ reject/defer
```

## 236. Availability Trade-Off

Policy puede variar por flow.

## 237. Login Replay Store Failure

Puede existir degraded strategy si threat model lo permite.

## 238. Break-Glass

Cuidado:
El emergency system no debe depender innecesariamente del mismo datastore cuya caída intenta resolver.

## 239. Break-Glass Replay Protection

Puede requerir infraestructura separada/resiliente:

- dedicated store
- hardware-backed one-time credential
- offline counter
- separate trust domain

## 240. Cryptographic Domain Separation

Toda operación criptográfica deberá incorporar purpose.

- Conceptualmente:
- VoltStack/Auth/TransactionState/v1
- VoltStack/Auth/Continuation/v1
- VoltStack/Auth/OAuthState/v1
- VoltStack/Auth/Resume/v1

## 241. Why

Evita usar ciphertext/MAC válido de un protocolo dentro de otro.

## 242. Associated Data

Si se utiliza AEAD, authenticated associated data puede contener:

- purpose
- version
- application
- realm
- cuando sea apropiado.

## 243. Context Confusion

Ciphertext válido para:
auth continuation
no debe ser aceptado como:
password reset token

## 244. Key Derivation

Documento 31 puede permitir subkeys derivadas por purpose.

## 245. No Manual KDF Without Design

Implementación deberá usar primitives/bibliotecas adecuadas.

## 246. Key Rotation

Protected states existentes deberán indicar key version/ID.

## 247. Rotation Window

Modelo:

```text
K1 ACTIVE
K2 introduced
```

new tokens → K2
existing K1 tokens → temporarily verify

## 248. Key Revocation

Si K1 comprometida:

- stop verifying K1
- aunque transactions legítimas fallen.
- Security domina continuidad.

## 249. Key Retirement

Después de máximo lifetime de todos los states dependientes:
K1 → RETIRED

## 250. Maximum State Lifetime

Debe ayudar a determinar key retention.

## 251. Cryptographic Inventory

Documento 31 deberá poder identificar qué keys protegen:

- transaction state
- continuations
- resume tokens
- OAuth state

## 252. Secret Logging

Prohibido registrar:

- state token
- nonce raw value
- PKCE verifier
- resume token
- magic-link token
- CSRF token
- authorization code

## 253. Safe Logging

Registrar:

- token fingerprint
- transaction ID
- nonce ID
- key ID
- purpose
- result
- cuando sea útil.

## 254. Token Fingerprint

Derivado no reversible y truncado apropiadamente para correlación.

## 255. Tracing

Nunca poner raw token en span attributes.

## 256. Metrics

Nunca usar token IDs o transaction IDs como labels.

## 257. Audit

Puede registrar:

- transaction
- purpose
- binding dimensions
- nonce identifier
- state version
- key ID
- validation result
- replay detected
- CSRF failure
- continuation rejection
- sin secretos.

## 258. Security Events

Eventos:

```text
AuthenticationStateCreated
AuthenticationStateValidated
AuthenticationStateRejected
AuthenticationStateExpired

AuthenticationNonceCreated
AuthenticationNonceConsumed
AuthenticationNonceReplayDetected

AuthenticationCsrfValidationFailed

AuthenticationReplayDetected

AuthenticationContinuationProtected
AuthenticationContinuationConsumed
AuthenticationContinuationRejected

AuthenticationResumeReferenceCreated
AuthenticationResumeReferenceConsumed

AuthenticationBindingMismatchDetected
AuthenticationSecurityEpochMismatchDetected
```

## 259. High-Signal Events

Especialmente:

- replay
- binding mismatch
- cross-tenant mismatch
- cross-realm mismatch
- invalid MAC/signature
- unexpected purpose

deben poder alimentar Risk/Incident systems.

## 260. DoS Consideration

No generar alertas costosas por cada token basura de Internet.

## 261. Event Classification

Distinguir:

- malformed noise
- probable attack
- confirmed replay
- internal inconsistency

## 262. Rate Limiting

Validation endpoints deben integrar documento 19.

## 263. Replay Cache Exhaustion

Atacante no deberá poder llenar replay store con IDs arbitrarios.

## 264. Consume Only Valid Material

Preferentemente:

```text
cryptographic/basic validation
        ↓
```

then replay state allocation
cuando sea seguro.

## 265. Bloom Filters

Podrían utilizarse como optimization en contextos concretos.
Nunca como única authoritative replay protection si false positives/negatives son inaceptables.

## 266. Cleanup

Replay records pueden expirar después del máximo período en que replay sería relevante.

## 267. Replay Tombstones

Algunos critical proofs pueden conservar tombstone más tiempo.

## 268. Retention

Debe balancear:

- security
- storage
- privacy
- compliance

## 269. Multi-Region

Replay protection debe considerar:

- Region A
- Region B

## 270. Eventual Replication Problem

proof consumed in A
immediately replayed in B
before replication

## 271. Critical Flows

Requerir:

- strongly consistent consume
- home region
- global coordination

region affinity for transaction
según arquitectura.

## 272. Region Affinity != Sticky Session

Puede existir:

- transaction home region
- sin depender de un application worker concreto.

## 273. Home Region

final readonly class AuthenticationTransactionPlacement
{
public function __construct(
public RegionId $homeRegion,
) {}
}

## 274. Cross-Region Callback

Puede reenrutarse internamente al authoritative region.

## 275. Regional Failure

Policy definirá si transaction:

- fails
- restarts
- uses replicated authority

## 276. No Split-Brain Authentication

Dos regiones no deberán completar la misma single-use transaction independientemente.

## 277. Persistent Workers

FrankenPHP requiere aislamiento estricto.

## 278. Never Store

static $currentNonce;
static $oauthState;
static $pkceVerifier;
static $continuation;

## 279. Secret Lifetime in Memory

Reducir lifetime de:

- PKCE verifier
- raw CSRF secret
- magic link token
- state plaintext

## 280. Immutable Runtime Objects

Preferir:

- final readonly class ...
- para state snapshots.

## 281. Request Context

Security state deberá pasar explícitamente por context/resolver.

## 282. Fiber Local Context

Si VoltStack proporciona context-local storage, deberá ser fiber-safe y limpiarse siempre.

## 283. Worker Reset

Después de cada request:

- clear auth flow context
- clear transient secrets
- clear transaction references
- clear continuation references

## 284. Serialization Cache

No cachear accidentalmente decrypted protected state entre requests.

## 285. Exception Objects

No incluir raw secrets en exception messages.

## 286. Debug Toolbar

Documento de developer tooling deberá redacted automáticamente estos valores.

## 287. Development Environment

APP_DEBUG=true nunca deberá imprimir:

- PKCE verifier
- nonce
- CSRF secret
- state plaintext
- magic link
- recovery token

## 288. Error Pages

Misma regla.

## 289. Browser DevTools

Algunos valores inevitablemente existen en browser.
Minimizar aquellos que no necesitan estar allí.

## 290. Frontend State Stores

No colocar long-lived Authentication capabilities en:

- localStorage
- Redux persisted state
- IndexedDB
- global JS variables
- sin necesidad.

## 291. HttpOnly

Cuando frontend JavaScript no necesita acceder al valor, preferir HttpOnly storage apropiado.

## 292. SPA Architecture

SPA podrá manejar:

- transaction reference
- challenge presentation
- interaction state

pero security truth permanece server-side.

## 293. SPA Continuation

Ejemplo:

```text
POST /api/tenant/42/delete
        ↓
authentication_required
        ↓
transaction reference
        ↓
passkey challenge
        ↓
proof
        ↓
```

server marks transaction satisfied
↓
continuation capability

## 294. SPA Automatic Replay

No automático para destructive actions salvo explicit safe intent protocol.

## 295. SPA Intent Token

Puede existir:

- opaque operation intent reference
- server-side.

## 296. Intent Store

interface AuthenticationIntentStoreInterface
{
public function create(
AuthenticationIntent $intent
): AuthenticationIntentReference;

public function consume(
AuthenticationIntentReference $reference
): AuthenticationIntent;
}

## 297. Request Body

Para operaciones sensibles, almacenar:

- canonical digest
- en vez de body completo cuando sea suficiente.

## 298. Sensitive Payload Storage

Si necesita almacenar payload para continuation:

- encrypt
- short TTL
- least data
- access control

## 299. File Uploads

No intentar serializar uploads arbitrarios dentro de continuation token.

## 300. Upload Continuation

Usar:

- temporary object reference
- +
- digest
- +
- ownership binding
- +
- TTL

## 301. Large Payloads

Mismo principio.

## 302. Authorization Recheck

Authentication continuation no congela Authorization.

## 303. Example

user begins delete
↓
step-up
↓
role removed by admin
↓
continuation
Debe volver a evaluar Authorization.

## 304. Authentication Recheck

También:

- account suspended
- session revoked
- risk escalated
- antes de completion.

## 305. Transaction Proof != Authorization Grant

Regla crítica.

## 306. Transaction Proof != Permanent Authentication

Operation-bound proof no necesariamente eleva toda Session.

## 307. AuthenticationProof

final readonly class AuthenticationProof
{
public function __construct(
public AuthenticationProofId $id,
public PrincipalId $principal,
public AuthenticationAssuranceProfile $assurance,
public AuthenticationSecurityBinding $binding,
public DateTimeImmutable $issuedAt,
public DateTimeImmutable $expiresAt,
) {}
}

## 308. Proof Scope

Puede ser:

- session-scoped
- operation-scoped
- transaction-scoped
- credential-management-scoped

## 309. AuthenticationProofScope

enum AuthenticationProofScope: string
{
case Session = 'session';
case Operation = 'operation';
case Transaction = 'transaction';
case CredentialManagement = 'credential_management';
}

## 310. Proof Store

High-risk proofs preferiblemente server-side/revocable.

## 311. Stateless Proof

Puede permitirse para lower-risk contexts si policy acepta limitaciones.

## 312. Proof Revocation

Puede ocurrir por:

- logout
- session revoke
- password change
- credential revoke
- security epoch
- risk escalation
- tenant suspension

## 313. Proof Replay

Operation-bound proof puede consumirse.
Session-scoped proof puede ser reusable dentro de lifetime según policy.

## 314. Explicit Semantics

Nunca inferir reuse simplemente porque el token aún no expiró.

## 315. Cryptographic Binding Validator

interface AuthenticationSecurityBindingValidatorInterface
{
public function validate(
AuthenticationSecurityBinding $expected,
AuthenticationSecurityContext $actual
): AuthenticationBindingValidationResult;
}

## 316. Validation Must Be Exact Where Required

No:

```text
tenant mismatch
→ ignore because same user
```

## 317. Optional Binding

Cada binding dimension deberá declarar si es:

- required
- optional
- not applicable

## 318. Binding Policy

final readonly class AuthenticationBindingPolicy
{
public function__construct(
public BindingRequirement $principal,
public BindingRequirement $session,
public BindingRequirement $tenant,
public BindingRequirement $realm,
public BindingRequirement $operation,
public BindingRequirement $client,
) {}
}

## 319. BindingRequirement

enum BindingRequirement: string
{
case Required = 'required';
case Optional = 'optional';
case NotApplicable = 'not_applicable';
}

## 320. Policy Compilation

Static binding policies pueden compilarse.
Dynamic values nunca.

## 321. Fail Closed on Missing Required Binding

Si operation binding required y falta:
reject

## 322. Null != Wildcard

Muy importante:
tenant = null
no debe significar automáticamente:
all tenants

## 323. Explicit Global Scope

Usar:

- GLOBAL
- como scope semántico explícito.

## 324. Wildcards

Evitar wildcards en security bindings.

## 325. Provider Binding

Federation state deberá estar ligado al provider esperado.

## 326. Provider Mix-Up

Callback de Provider B no debe completar request iniciado con Provider A.

## 327. Issuer Binding

OIDC issuer esperado deberá validarse.

## 328. Multi-Issuer Systems

Issuer es parte de identity/trust boundary.

## 329. Redirect Binding

Callback path/redirect URI deberá coincidir con configuración esperada.

## 330. OAuth Mix-Up Protection

Architecture deberá considerar provider/issuer mix-up además de state CSRF.

## 331. Account Linking

External identity reference:

- provider
- issuer
- subject

solo se vincula después de validar transaction purpose y current local identity binding.

## 332. Callback Does Not Choose Local Identity

Nunca:

```php
callback query:
local_user_id=123
```

como autoridad para linking.

## 333. Local Identity Source

Debe venir de protected transaction state.

## 334. Password Confirmation

Un simple:

```php
session['password_confirmed_at']
puede ser insuficiente para architecture avanzada.
```

## 335. Fresh Authentication Proof

Usar:
AuthenticationProof
con:

- method
- assurance
- freshness
- session binding
- purpose/scope

## 336. MFA Confirmation

Misma regla.

## 337. Step-Up Proof

Puede quedar operation-bound.

## 338. Security Center Integration

Documento 35 puede utilizar estos mecanismos para:

- revoke session
- remove passkey
- unlink provider
- regenerate recovery codes
- sign out everywhere

## 339. Credential Removal Example

Security Center
↓
Remove Passkey P1
↓
Requirement: fresh phishing-resistant auth
↓
Transaction
↓
Passkey P2 challenge
↓
Proof bound to:

```php
identity
session
operation=passkey.remove
resource=P1
      ↓
Remove P1
```

## 340. Self-Destructive Credential Proof

Policy puede impedir autenticar con la misma credential que se intenta eliminar para ciertas operaciones.

## 341. Credential Binding

Credential enrollment transaction debe ligar:

- new credential challenge
- current identity
- current authenticated session
- enrollment purpose

## 342. Credential Substitution Attack

No permitir que attacker cambie:

- credential being enrolled
- entre challenge generation y completion.

## 343. WebAuthn Registration

Challenge ligado a:

- identity
- RP
- origin
- transaction
- enrollment purpose

## 344. TOTP Enrollment

Pending secret ligado a enrollment transaction.

## 345. TOTP Secret Substitution

Confirmation code debe verificar exactamente el pending secret correspondiente.

## 346. Recovery Regeneration

New recovery set generation debe ser operation-bound.

## 347. Privileged Authentication

Proof puede incluir:

```php
realm = ADMIN
scope = privileged
maximum age
phishing resistant
```

## 348. Break-Glass

Purpose separation estricta.

## 349. Break-Glass Token

Nunca aceptado por normal login flow si no está explícitamente diseñado.

## 350. Machine Identity

Documento 33.

- Non-human authentication también necesita:
- nonce
- audience
- replay protection
- signed request freshness
- jti
- operation binding

## 351. Signed Requests

Modelo:

- method
- URI
- canonical headers
- body digest
- timestamp
- nonce
- audience
- firmados.

## 352. Signed Request Canonicalization

Debe versionarse.

## 353. Signed Request Replay Store

Nonce/jti debe consumirse.

## 354. Timestamp Alone Is Insufficient

Ventana de:

- ±5 minutes
- sin nonce permite replay durante cinco minutos.

## 355. Machine Nonce Scope

Puede ligarse a:

- machine identity
- credential
- audience
- operation

## 356. API Personal Tokens

No necesariamente utilizan transaction state en cada request.
Pero credential creation/revocation sí.

## 357. Security Posture Integration

Documento 36/37 puede exigir stronger binding según risk.

## 358. Risk-Based Binding

Ejemplo:

```text
normal profile edit
→ session-bound reauth

banking transfer
→ session + operation + payload bound
```

## 359. Binding Strength Escalation

Policy Engine podrá producir requirements como:

```php
requireOperationBinding = true
requirePayloadBinding = true
requireSingleUseProof = true
```

## 360. AuthenticationSecurityRequirement Extension

Conceptualmente:

```php
final readonly class AuthenticationTransactionSecurityRequirement
{
    public function __construct(
        public bool $requireSessionBinding,
        public bool $requireOperationBinding,
        public bool $requirePayloadBinding,
        public bool $requireSingleUse,
        public bool $requireServerSideState,
    ) {}
}
```

## 361. Security Level != Cryptographic Algorithm

No diseñar:

```text
HIGH → AES
LOW → no signature
```

Todos los protected states requieren baseline robusto.

- Security level afecta:
- binding
- lifetime
- single-use
- revocability
- required statefulness

más que inventar diferentes primitive strengths.

## 362. Security Profiles

Ejemplo:

- STANDARD_TRANSACTION
- SENSITIVE_OPERATION
- PRIVILEGED_OPERATION
- RECOVERY
- BREAK_GLASS
- MACHINE_SIGNED_REQUEST

## 363. Standard Transaction

Puede requerir:

- transaction binding
- purpose
- expiration
- CSRF

## 364. Sensitive Operation

Además:

- session
- operation
- single-use
- authoritative replay store

## 365. Critical Operation

Además:

- payload digest
- shorter TTL
- final recheck
- stateful proof

## 366. Privileged

Además:

- admin realm
- high assurance
- security epoch
- stronger audit

## 367. Recovery

Además:

- restricted completion
- cooldown
- recovery-specific purpose

## 368. Break-Glass

Además:

- separate trust domain
- emergency purpose
- incident reference
- special replay strategy

## 369. Failure Taxonomy

Errores específicos:

```text
AUTH_STATE_INVALID
AUTH_STATE_MALFORMED
AUTH_STATE_EXPIRED
AUTH_STATE_NOT_YET_VALID
AUTH_STATE_UNSUPPORTED_VERSION
AUTH_STATE_UNKNOWN_KEY
AUTH_STATE_CRYPTOGRAPHIC_VALIDATION_FAILED
AUTH_STATE_PURPOSE_MISMATCH

AUTH_NONCE_INVALID
AUTH_NONCE_EXPIRED
AUTH_NONCE_REPLAYED
AUTH_NONCE_PURPOSE_MISMATCH

AUTH_CSRF_INVALID
AUTH_CSRF_BINDING_MISMATCH

AUTH_REPLAY_DETECTED
AUTH_REPLAY_STORE_UNAVAILABLE

AUTH_BINDING_PRINCIPAL_MISMATCH
AUTH_BINDING_SESSION_MISMATCH
AUTH_BINDING_TENANT_MISMATCH
AUTH_BINDING_REALM_MISMATCH
AUTH_BINDING_CLIENT_MISMATCH
AUTH_BINDING_OPERATION_MISMATCH
AUTH_BINDING_PAYLOAD_MISMATCH
AUTH_BINDING_PROVIDER_MISMATCH

AUTH_SECURITY_EPOCH_MISMATCH

AUTH_CONTINUATION_INVALID
AUTH_CONTINUATION_EXPIRED
AUTH_CONTINUATION_ALREADY_CONSUMED
AUTH_CONTINUATION_TARGET_INVALID
AUTH_CONTINUATION_BINDING_MISMATCH

AUTH_RESUME_REFERENCE_INVALID
AUTH_RESUME_REFERENCE_EXPIRED
AUTH_RESUME_REFERENCE_CONSUMED

AUTH_PKCE_VERIFIER_MISSING
AUTH_PKCE_VERIFIER_INVALID

AUTH_OAUTH_STATE_INVALID
AUTH_OIDC_NONCE_INVALID
```

## 370. Public Failure

No devolver necesariamente reason exacto.

```php
Ejemplo:
{
  "error": "authentication_flow_invalid"
}
```

## 371. Internal Failure

Internamente:

- AUTH_BINDING_TENANT_MISMATCH
- para auditoría/investigación.

## 372. Security Invariants — State

AUTH-STATE-01
Todo protected Authentication State tiene purpose.

- AUTH-STATE-02
- Todo state tiene lifetime limitado.
- AUTH-STATE-03

Todo state tiene versión.

- AUTH-STATE-04
- Client-controlled state no es trusted sin validación.
- AUTH-STATE-05

Cryptographic validity no implica semantic validity.
AUTH-STATE-06
State de un purpose no puede utilizarse en otro.

## 373. Security Invariants — Nonces

AUTH-NONCE-01
Nonces se generan con CSPRNG.

- AUTH-NONCE-02
- Nonces poseen purpose.
- AUTH-NONCE-03

Nonces no se reutilizan entre protocolos.

- AUTH-NONCE-04
- Nonces tienen TTL.
- AUTH-NONCE-05

Single-use nonces se consumen atómicamente.

## 374. Security Invariants — Replay

AUTH-REPLAY-01
Single-use proof solo puede consumirse una vez.

- AUTH-REPLAY-02
- Concurrent replay produce máximo una transición autorizada.
- AUTH-REPLAY-03

Replay protection funciona entre nodes.

- AUTH-REPLAY-04
- Timestamp no sustituye nonce.
- AUTH-REPLAY-05

Signature no sustituye replay state.
AUTH-REPLAY-06
Idempotency no sustituye replay protection.

## 375. Security Invariants — CSRF

AUTH-CSRF-01
Cookie-based sensitive Authentication flows reciben CSRF protection apropiada.
AUTH-CSRF-02
Login puede requerir protección contra login CSRF.

- AUTH-CSRF-03
- Account linking debe estar ligado a current authenticated identity/session.
- AUTH-CSRF-04

SameSite no es única defensa.
AUTH-CSRF-05
Login callback no puede completar linking transaction.

## 376. Security Invariants — Binding

AUTH-BIND-01
Required binding ausente produce rechazo.

- AUTH-BIND-02
- null nunca significa wildcard implícito.
- AUTH-BIND-03

Tenant binding no cruza tenants.

- AUTH-BIND-04
- Realm binding no cruza realms.
- AUTH-BIND-05

Principal binding no cambia silenciosamente.

- AUTH-BIND-06
- Operation-bound proof no autoriza otra operación.
- AUTH-BIND-07

Payload-bound proof no autoriza payload modificado.

## 377. Security Invariants — Continuation

AUTH-CONT-SEC-01
Continuation está protegida contra modificación.

- AUTH-CONT-SEC-02
- Continuation tiene expiration.
- AUTH-CONT-SEC-03

Sensitive continuation puede ser single-use.

- AUTH-CONT-SEC-04
- Continuation target es validado.
- AUTH-CONT-SEC-05

Open redirects están prohibidos.
AUTH-CONT-SEC-06
Continuation no congela Authorization.

## 378. Security Invariants — Crypto

AUTH-CRYPTO-01
No custom cryptography.

- AUTH-CRYPTO-02
- Keys están separadas por purpose lógico.
- AUTH-CRYPTO-03

Algorithms están allowlisted server-side.

- AUTH-CRYPTO-04
- Unknown kid no controla arbitrary key retrieval.
- AUTH-CRYPTO-05

Key compromise permite invalidación.
AUTH-CRYPTO-06
Sensitive state requiere confidentiality cuando corresponda, no solo signature.

## 379. Security Invariants — Runtime

AUTH-STATE-RT-01
No existe mutable global transaction security state.

- AUTH-STATE-RT-02
- FrankenPHP workers no comparten nonces entre requests accidentalmente.
- AUTH-STATE-RT-03

PKCE verifier no permanece en worker global.

- AUTH-STATE-RT-04
- Fiber execution mantiene aislamiento.
- AUTH-STATE-RT-05

Decrypted state no se cachea entre principals.

## 380. Anti-Pattern

state = base64(user_id)

## 381. Anti-Pattern

state = encrypt(user_id)
sin:

- purpose
- transaction
- expiration
- replay protection

## 382. Anti-Pattern

OAuth state == OIDC nonce

## 383. Anti-Pattern

CSRF token == OAuth state == transaction token

## 384. Anti-Pattern

valid signature
→ accept
sin semantic validation.

## 385. Anti-Pattern

JWT exp valid
→ replay impossible

## 386. Anti-Pattern

timestamp valid
→ signed request cannot replay

## 387. Anti-Pattern

session user == transaction user
sin transaction binding para account linking.

## 388. Anti-Pattern

?next=<https://whatever.example>

## 389. Anti-Pattern

POST destructive operation
→ reauth
→ browser automatically POSTs again
sin intent/idempotency model.

## 390. Anti-Pattern

tenant = null
→ valid for all tenants

## 391. Anti-Pattern

User-Agent fingerprint
→ cryptographic device binding

## 392. Anti-Pattern

IP unchanged
→ same client

## 393. Anti-Pattern

PKCE enabled
→ no state needed

## 394. Anti-Pattern

SameSite=Lax
→ CSRF solved

## 395. Anti-Pattern

signed continuation
→ no replay store needed
para single-use operations.

## 396. Anti-Pattern

static $pkceVerifier;
bajo FrankenPHP.

## 397. Anti-Pattern

debug log:

```php
oauth_state=...
pkce_verifier=...
```

## 398. Componentes principales

AuthenticationTransactionSecurityState
AuthenticationSecurityBinding
AuthenticationBindingPolicy
AuthenticationSecurityBindingValidator

AuthenticationNonce
AuthenticationNonceSet
AuthenticationNoncePolicy
AuthenticationNonceGenerator

AuthenticationReplayState
AuthenticationReplayStore

AuthenticationStateProtector
AuthenticationStateSerializer
AuthenticationProtectedState

AuthenticationCsrfBinding

ProtectedAuthenticationContinuation
AuthenticationContinuationStore

AuthenticationResumeReference

AuthenticationSecurityEpochSet
AuthenticationSecurityEpochResolver

AuthenticationProof
AuthenticationProofScope

AuthenticationIntentStore
AuthenticatedOperationExecution

## 399. Namespace sugerido

VoltStack\Quantum\Auth\TransactionSecurity

## 400. Estructura sugerida

src/Quantum/Auth/TransactionSecurity/
├── Contracts/
│   ├── AuthenticationStateProtectorInterface.php
│   ├── AuthenticationStateSerializerInterface.php
│   ├── AuthenticationNonceGeneratorInterface.php
│   ├── AuthenticationReplayStoreInterface.php
│   ├── AuthenticationSecurityBindingValidatorInterface.php
│   ├── AuthenticationSecurityEpochResolverInterface.php
│   ├── AuthenticationContinuationStoreInterface.php
│   └── AuthenticationIntentStoreInterface.php
│
├── State/
│   ├── AuthenticationTransactionSecurityState.php
│   ├── AuthenticationProtectedState.php
│   ├── ProtectedAuthenticationState.php
│   └── AuthenticationStateVersion.php
│
├── Binding/
│   ├── AuthenticationSecurityBinding.php
│   ├── AuthenticationBindingPolicy.php
│   ├── BindingRequirement.php
│   ├── PrincipalBinding.php
│   ├── SessionBinding.php
│   ├── TenantBinding.php
│   ├── RealmBinding.php
│   ├── ClientBinding.php
│   ├── FlowBinding.php
│   └── OperationBinding.php
│
├── Nonce/
│   ├── AuthenticationNonce.php
│   ├── AuthenticationNonceId.php
│   ├── AuthenticationNonceSet.php
│   ├── AuthenticationNoncePurpose.php
│   └── AuthenticationNoncePolicy.php
│
├── Replay/
│   ├── AuthenticationReplayState.php
│   ├── ReplayIdentifier.php
│   ├── ReplayConsumptionResult.php
│   ├── RedisReplayStore.php
│   └── DatabaseReplayStore.php
│
├── Csrf/
│   └── AuthenticationCsrfBinding.php
│
├── Continuation/
│   ├── ProtectedAuthenticationContinuation.php
│   ├── ContinuationId.php
│   └── AuthenticationContinuationConsumptionContext.php
│
├── Resume/
│   ├── AuthenticationResumeReference.php
│   └── ResumeReferenceId.php
│
├── Proof/
│   ├── AuthenticationProof.php
│   ├── AuthenticationProofId.php
│   └── AuthenticationProofScope.php
│
├── Epoch/
│   ├── AuthenticationSecurityEpochSet.php
│   └── AuthenticationSecurityEpochValidator.php
│
├── Intent/
│   ├── PayloadDigest.php
│   ├── AuthenticationIntentStore.php
│   └── AuthenticatedOperationExecution.php
│
├── Crypto/
│   ├── AuthenticationStateProtector.php
│   ├── AuthenticationStateSerializer.php
│   ├── AuthenticationCryptographicPurpose.php
│   └── AuthenticationStateKeyResolver.php
│
├── Runtime/
│   ├── AuthenticationTransactionSecurityManager.php
│   └── AuthenticationTransactionSecurityResetter.php
│
├── Events/
│   └── ...

```text
│
└── Exceptions/
    └── ...
```

## 401. Flujo criptográfico general

Authentication Flow (38)
│
▼
Create Transaction State
│
▼
Determine Security Policy
│
┌─────────────┼──────────────┐
▼             ▼              ▼
Bindings       Nonces        Continuation
│             │              │
└─────────────┼──────────────┘
▼
Protect / Persist State
│
▼
CLIENT
│
▼
Interaction
│
▼
State / Proof Returned
│
▼
Syntax Validation
│
▼
Cryptographic Validation
│
▼
Purpose Validation
│
▼
Binding Validation
│
▼
Epoch Validation
│
▼
Replay Validation
│
▼
Flow State Validation
│
▼
ACCEPT
│
▼
Authentication Flow (38)

## 402. OAuth/OIDC example

VoltStack
│
├── create Authentication Transaction
├── generate state
├── generate OIDC nonce
├── generate PKCE verifier
├── derive PKCE challenge
├── bind provider
├── bind purpose
└── persist protected state
│
▼
Identity Provider
│
▼
Callback
│
├── validate state
├── consume state
├── validate provider
├── validate issuer
├── exchange code using PKCE verifier
├── validate ID Token
├── validate OIDC nonce
├── validate transaction
└── produce Authentication Evidence

## 403. Sensitive operation example

DELETE TENANT 42
│
▼
Authentication Requirement
│
▼
Create Step-Up Transaction
│
├── principal = user-123
├── session = session-A
├── tenant = tenant-42
├── realm = tenant-admin
├── operation = tenant.delete
├── resource = tenant-42
└── payload digest = XYZ
│
▼
Passkey Challenge
│
▼
Evidence
│
▼
Authentication Proof
│
▼
Validate all bindings
│
▼
Consume proof
│
▼
Recheck Authorization
│
▼
Recheck Tenant State
│
▼
Delete Tenant 42
El proof no puede utilizarse para:

- delete Tenant 43
- change password
- create API key
- assign admin role

## 404. Laravel comparison

Laravel dispone de piezas relacionadas mediante:

- CSRF middleware
- sessions
- signed URLs
- temporary signed URLs
- password confirmation
- Socialite state handling
- session regeneration
- Sanctum
- Fortify

Estas piezas resuelven problemas importantes, pero no constituyen necesariamente un único dominio de:
Authentication Transaction Security
VoltStack formalizará una infraestructura común para:

- login
- MFA
- passkeys
- OAuth/OIDC
- reauthentication
- step-up
- recovery
- credential management
- account linking
- device flows
- privileged operations

## 405. Symfony comparison

Symfony aporta primitivas maduras alrededor de:

- CSRF
- Security
- Authenticators
- Login links
- sessions
- remember-me
- signed URIs

OAuth integrations through ecosystem packages
VoltStack añadirá una capa explícita que conecte:

- Transaction
- +
- Purpose
- +
- Nonce
- +
- Replay State
- +
- Security Binding
- +
- Continuation
- +
- Operation Intent
- +
- Security Epoch
- +
- Cryptographic Protection

como un solo subsistema.

## 406. Diferenciador VoltStack

Laravel-like DX
+
Symfony-like Security Rigor
+
Explicit Authentication Transactions
+
Purpose-Bound State
+
Principal/Session/Tenant/Realm Binding
+
Operation-Bound Reauthentication
+
Payload-Bound Sensitive Authentication
+
Nonce Domain Separation
+
Distributed Replay Protection
+
CSRF-Aware Authentication Flows
+
OAuth State + PKCE + OIDC Nonce Composition
+
Cryptographically Protected Continuations
+
Single-Use Resume Capabilities
+
Security Epoch Invalidation
+
Multi-Region Replay Coordination
+
FrankenPHP/Fiber-Safe Runtime

## 407. Decisiones arquitectónicas definitivas

VoltStack adoptará las siguientes reglas:

1. Authentication Transaction State será un dominio explícito.
2. State no dependerá exclusivamente de PHP Session.
3. Todo state tendrá purpose.
4. Todo state tendrá versión.
5. Todo state tendrá lifetime.
6. State del cliente será untrusted hasta validarse.
7. Cryptographic validity y semantic validity serán independientes.
8. Transactions podrán ligarse a principal.
9. Transactions podrán ligarse a Session.
10. Transactions podrán ligarse a Tenant.
11. Transactions estarán ligadas a Realm.
12. Sensitive transactions podrán ligarse a operation.
13. Critical transactions podrán ligarse a payload digest.
14. null no significará wildcard.
15. Global scope será explícito.
16. Nonces usarán CSPRNG.
17. Nonces serán purpose-specific.
18. OAuth state y OIDC nonce serán diferentes.
19. CSRF token y OAuth state no serán el mismo primitive.
20. PKCE no sustituirá state.
21. State no sustituirá PKCE.
22. OIDC nonce no sustituirá state.
23. Single-use state tendrá atomic consumption.
24. Replay protection será distribuida cuando corresponda.
25. Signature no será considerada replay protection.
26. Timestamp no será considerado replay protection.
27. Idempotency no será considerada replay protection.
28. Continuations serán protegidas.
29. Sensitive continuations podrán ser single-use.
30. Continuation target será validado.
31. Arbitrary open redirects estarán prohibidos.
32. Operation intent sobrevivirá reauthentication sin poder ser modificado.
33. Authentication continuation volverá a evaluar Authorization.
34. Authentication continuation podrá volver a evaluar Risk.
35. Security Epoch permitirá invalidación masiva.
36. Provider Security Epoch permitirá invalidar federation state.
37. State protection utilizará keys administradas por 31.
38. Keys estarán separadas por purpose.
39. Custom cryptography estará prohibida.
40. Algorithms estarán server-allowlisted.
41. kid no permitirá arbitrary remote key retrieval.
42. Sensitive state tendrá confidentiality cuando sea necesaria.
43. Stateful opaque state será preferido para high-risk operations.
44. Stateless protected state podrá existir cuando threat model lo permita.
45. Stateless single-use state seguirá necesitando replay control.
46. PKCE verifier será secreto temporal.
47. Login CSRF será parte del threat model.
48. Account Linking tendrá strict session/identity/purpose binding.
49. Login callback nunca completará Linking transaction.
50. Magic/recovery links serán purpose-separated.
51. Browser GET no realizará destructive security mutation cuando scanners puedan activarlo.
52. Device codes tendrán replay/rate-limit protection.
53. Signed machine requests usarán nonce además de timestamp.
54. Critical replay stores fallarán cerrados.
55. Multi-region design impedirá double completion.
56. Debugging nunca expondrá transient secrets.
57. Logs/traces nunca almacenarán raw Authentication state secrets.
58. FrankenPHP no conservará transaction secrets globalmente.
59. Fiber isolation será obligatoria.
60. Authentication security proofs tendrán scope explícito.
61. Criterios de aceptación

El subsistema estará completo cuando VoltStack pueda demostrar soporte para:
62. transaction security state;
63. state versioning;
64. state purpose;
65. state expiration;
66. principal binding;
67. session binding;
68. tenant binding;
69. realm binding;
70. client binding;
71. operation binding;
72. resource binding;
73. payload digest binding;
74. canonical payload representation;
75. CSPRNG nonces;
76. nonce purposes;
77. nonce TTL;
78. nonce atomic consumption;
79. replay detection;
80. distributed replay store;
81. concurrent replay protection;
82. challenge replay protection;
83. TOTP replay integration;
84. WebAuthn challenge binding;
85. OAuth state;
86. OIDC nonce;
87. PKCE;
88. PKCE verifier protection;
89. provider binding;
90. issuer binding;
91. redirect URI validation;
92. login CSRF protection;
93. account-linking CSRF protection;
94. reauthentication CSRF binding;
95. protected continuations;
96. continuation TTL;
97. continuation single-use;
98. safe continuation targets;
99. open-redirect prevention;
100. resume references;
101. device-flow references;
102. magic-link security;
103. recovery-link purpose separation;
104. security epochs;
105. credential versions;
106. provider security epochs;
107. state signing/authentication;
108. confidential state protection;
109. cryptographic purpose separation;
110. key rotation;
111. key revocation;
112. algorithm allowlisting;
113. strict serialization;
114. semantic validation;
115. constant-time secret comparisons;
116. hashed opaque token storage;
117. idempotent operation integration;
118. operation proof consumption;
119. Authorization recheck;
120. risk recheck;
121. distributed state;
122. multi-region replay control;
123. fail-closed critical operations;
124. audit;
125. security events;
126. metrics;
127. tracing;
128. secret redaction;
129. debug redaction;
130. FrankenPHP isolation;
131. Fiber isolation;
132. OAuth/OIDC tests;
133. replay-race tests;
134. CSRF tests;
135. cross-tenant tests;
136. cross-realm tests;
137. purpose-confusion tests;
138. continuation manipulation tests;
139. key-rotation tests;
140. epoch-invalidation tests;
141. multi-region tests.
142. Reglas finales

La seguridad completa deberá obedecer:

```text
          CLIENT INPUT
               │
               ▼
        UNTRUSTED STATE
               │
               ▼
       Syntax Validation
               │
               ▼
    Cryptographic Validation
               │
               ▼
       Purpose Validation
               │
               ▼
       Lifetime Validation
               │
               ▼
       Binding Validation
               │
               ▼
        Epoch Validation
               │
               ▼
       Replay Validation
               │
               ▼
       Flow Validation
               │
               ▼
       AUTHENTIC STATE
```

Y deberá mantenerse la separación:

```text
36 — Policy Engine
¿Qué exige la política?

37 — Assurance System
¿Qué evidencia y confianza poseemos?

38 — Challenge / Interactive Flow
¿Cómo obtenemos la evidencia faltante?

39 — Transaction Security
¿Cómo garantizamos que el estado,
```

challenge, proof y continuation
no fueron falsificados, intercambiados
o reutilizados?
La regla principal de este documento será:
Una Authentication Transaction no es segura simplemente porque sus credenciales sean correctas. También debe demostrarse que cada challenge, callback, nonce, proof y continuation pertenece exactamente al flujo, principal, contexto, propósito y operación para los que fue creado, dentro de su tiempo de vida y sin haber sido reutilizado.

## 1. Posición dentro de la arquitectura global

Con los documentos 36–39, VoltStack ya posee cuatro capas claramente separadas:

```text
┌───────────────────────────────────────────────────────────┐
│                    AUTHENTICATION                         │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  36  GOVERNANCE / POLICY                                  │
│       What must be proven?                                │
│                         │                                 │
│                         ▼                                 │
│  37  ASSURANCE / TRUST                                    │
│       What has been proven?                               │
│                         │                                 │
│                         ▼                                 │
│  38  CHALLENGE / FLOW                                     │
```

│       How do we obtain what is missing?                   │
│                         │                                 │
│                         ▼                                 │
│  39  TRANSACTION SECURITY                                 │
│       Is the transaction itself authentic, fresh,         │
│       bound and non-replayed?                             │
│                                                           │
└───────────────────────────────────────────────────────────┘
Esto evita mezclar cuatro problemas que muchos frameworks terminan concentrando dentro de un AuthManager, middleware o controller.

## 2. Siguiente documento contemplado

El siguiente documento que recomiendo para continuar la arquitectura es:
`40_AUTHENTICATION_SECURITY_NOTIFICATION_ALERTING_COMPROMISE_DETECTION_INCIDENT_RESPONSE_AND_ACCOUNT_PROTECTION_SYSTEM.md`
El cambio es importante: los documentos anteriores protegen el proceso de Authentication; el 40 deberá encargarse de detectar y responder cuando existan indicios de que la identidad, una Session, una Credential o un Authentication Method ya pudo haber sido comprometido.
La arquitectura sería:

```php
Authentication Events
        │
        ├── Failed Login
        ├── Credential Replay
        ├── Impossible Travel
        ├── New Device
        ├── Credential Compromise
        ├── Session Anomaly
        ├── Recovery Activity
        ├── MFA Change
        ├── Passkey Removal
        ├── Provider Compromise
        └── Security Binding Attack
                 │
                 ▼
        COMPROMISE DETECTION
                 │
                 ▼
          Incident Evaluation
                 │
       ┌─────────┼───────────┐
       ▼         ▼           ▼
    Notify    Contain     Investigate
       │         │           │
       │         ├── Revoke Session
       │         ├── Revoke Credential
       │         ├── Increment Epoch
       │         ├── Restrict Account
       │         └── Force Reauth
       │
       ▼
   Security Center (35)
```

De esta manera, el 40 cerraría una pieza crítica que todavía falta en Authentication: no solo impedir ataques durante el login, sino responder coordinadamente cuando el sistema sospecha que una cuenta ya está bajo ataque o comprometida.
