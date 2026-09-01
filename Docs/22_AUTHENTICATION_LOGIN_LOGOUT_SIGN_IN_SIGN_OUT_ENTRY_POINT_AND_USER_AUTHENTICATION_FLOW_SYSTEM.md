# VoltStack Authentication System

## 22 — Login, Logout, Sign-In, Sign-Out, Entry Point and User Authentication Flow System

- **Archivo:** `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de entrada, continuación y salida de Authentication.
- **Objetivo principal:** definir cómo una aplicación VoltStack inicia, coordina, continúa, completa y termina los flujos de Authentication sin concentrar Password, MFA, Passkeys, Federation, Risk, Device Trust, Session o Recovery dentro de controladores monolíticos.

---

## 1. Propósito

Hasta este punto, VoltStack dispone conceptualmente de subsistemas especializados para:

- Identity
- Credentials
- Passwords
- Sessions
- Remember-Me
- Bearer Tokens
- MFA
- Passkeys
- OAuth2 / OIDC
- Account Recovery
- Abuse Protection
- Risk
- Device Trust

Ahora necesitamos responder:
¿Cómo interactúan todos estos subsistemas cuando un usuario realmente inicia o cierra sesión?

Este documento define esa capa de coordinación.

## 2. Alcance

El sistema deberá cubrir:

```text
Login
Logout
Sign-In
Sign-Out

Authentication Entry Points
Authentication Flow Coordinator
Authentication Flow State
Authentication Continuation

Form Login
JSON Login
SPA Login
Passkey Login
Federated Login
Remember-Me Restoration

MFA Continuation
Step-Up Authentication
Fresh Authentication
Recovery Entry Points

Authentication Success
Authentication Failure
Authentication Challenge

Session Establishment
Session Rotation
Remember-Me Issuance
Device Trust Issuance

Redirect Handling
Intended Destination
Safe Redirects

Current Session Logout
Device Logout
All Sessions Logout
Global Logout

Federated Logout Boundaries

CSRF Protection
Logout Security

Events
Hooks
Audit
Observability
Extensibility
FrankenPHP Safety
```

## 3. Terminología

VoltStack podrá utilizar:

- Sign-In
- Authentication
- Login

como términos de UX/API.
Arquitectónicamente, el término preferido será:

- Authentication Flow
- porque un proceso puede necesitar múltiples pasos.

## 4. Login no siempre es una operación única

El modelo tradicional:

```text
POST /login
    ↓
password correct?
    ↓
session
es insuficiente.
```

Un flujo moderno puede ser:

```text
Identifier
    ↓
Password
    ↓
Risk Evaluation
    ↓
MFA
    ↓
Passkey
    ↓
Device Trust Decision
    ↓
Session
```

## 5. Principio fundamental

El Login Controller no implementará Authentication.

El controller será un adapter de transporte.

## 6. Segunda regla fundamental

Authentication Flow Coordinator orquestará subsistemas; no reimplementará sus responsabilidades.

## 1. Tercera regla fundamental

Un flujo de Authentication podrá ser multi-request, multi-step y multi-protocol.

## 2. Arquitectura general

HTTP / SPA / API / CLI
│
▼
Authentication Entry Point
│
▼
Authentication Flow Coordinator
│
▼
Authentication Attempt
│
├── Authenticator Resolution
├── Credential Verification
├── Identity Eligibility
├── Abuse Protection
├── Risk Assessment
├── MFA / Step-Up
├── Device Trust
└── Authentication Policy
│
▼
Authentication Flow Result
│
┌────┼───────────┬────────────┐
▼    ▼           ▼            ▼
SUCCESS CHALLENGE CONTINUE     FAILURE
│        │         │            │
▼        ▼         ▼            ▼
Session   MFA/      Next        Failure
Creation Passkey    Step        Handler
│
▼
Success Handler
│
▼
Redirect / JSON / SPA Response

## 3. AuthenticationEntryPoint

Un Entry Point define cómo comienza un Authentication Flow.

```php
Contrato conceptual:
interface AuthenticationEntryPointInterface
{
    public function start(
        AuthenticationEntryRequest $request
    ): AuthenticationFlowResponse;
}
```

## 4. Entry Points built-in

VoltStack podrá incluir:

- FormLoginEntryPoint
- JsonLoginEntryPoint
- SpaLoginEntryPoint
- PasskeyEntryPoint
- FederatedLoginEntryPoint
- RememberMeEntryPoint
- StepUpEntryPoint
- RecoveryEntryPoint

## 5. Entry Point != Authenticator

Ejemplo:
FormLoginEntryPoint
puede iniciar un flujo que termine utilizando:

- PasswordAuthenticator
- TOTPAuthenticator
- PasskeyAuthenticator

## 6. Entry Point != Controller

El controller puede delegar a un Entry Point.

```php
public function login(LoginRequest $request)
{
    return $this->auth->entry('login')->start($request);
}
```

## 7. AuthenticationFlowCoordinator

Componente central:

```php
interface AuthenticationFlowCoordinatorInterface
{
    public function start(
        AuthenticationFlowRequest $request
    ): AuthenticationFlowResult;

    public function continue(
        AuthenticationFlowContinuation $continuation
    ): AuthenticationFlowResult;
}
```

## 8. Responsabilidades

El Coordinator deberá:

- resolve Authentication context
- create flow
- invoke AuthenticationManager
- evaluate intermediate results
- request additional evidence
- coordinate Step-Up
- finalize successful Authentication
- route failures

## 9. Lo que NO deberá hacer

No implementará directamente:

- password hashing
- TOTP verification
- WebAuthn verification
- OIDC token validation
- session storage
- risk scoring
- device fingerprinting
- rate limiting

## 10. AuthenticationFlowId

Cada flujo multi-step tendrá un identificador opaco:
AuthenticationFlowId

## 11. Propiedades

Debe ser:

- unguessable
- non-semantic
- short-lived
- safe for correlation

## 12. AuthenticationFlow

Conceptualmente:

```php
final readonly class AuthenticationFlow
{
    public function __construct(
        public AuthenticationFlowId $id,
        public AuthenticationFlowType $type,
        public AuthenticationFlowStatus $status,
        public AuthenticationPurpose $purpose,
        public \DateTimeImmutable $startedAt,
        public \DateTimeImmutable $expiresAt,
    ) {}
}
```

## 13. AuthenticationFlowType

Ejemplos:

- INTERACTIVE_LOGIN
- PASSKEY_LOGIN
- FEDERATED_LOGIN
- REMEMBER_ME_RESTORATION
- STEP_UP
- FRESH_AUTHENTICATION
- RECOVERY

## 14. AuthenticationFlowStatus

STARTED
IDENTITY_RESOLVED
EVIDENCE_PENDING
CHALLENGE_PENDING
STEP_UP_REQUIRED
COMPLETED
FAILED
EXPIRED
CANCELLED

## 15. Flow State

Un flujo multi-request necesita conservar estado.

```text
Pero:
Nunca deberá persistir secrets de Authentication innecesariamente.
```

## 16. AuthenticationFlowState

Puede conservar:

- FlowId
- IdentityReference
- TenantReference
- Firewall
- Purpose
- Completed Steps
- Required Evidence
- Assurance State
- Risk Summary
- Intended Destination Reference
- Expiration

## 17. Nunca conservar

raw password
raw OTP
private key
raw bearer token
recovery secret

## 18. FlowStateStore

Contrato:

```php
interface AuthenticationFlowStateStoreInterface
{
    public function load(
        AuthenticationFlowId $id
    ): ?AuthenticationFlowState;

    public function save(
        AuthenticationFlowState $state
    ): void;

    public function consume(
        AuthenticationFlowId $id
    ): void;
}
```

## 19. Implementaciones

Podrán existir:

- SessionAuthenticationFlowStore
- CacheAuthenticationFlowStore
- DatabaseAuthenticationFlowStore
- SignedClientFlowState

## 20. Signed client state

Solo para información que sea seguro entregar al cliente.
No secrets ni authoritative mutable security state sin protección adecuada.

## 21. Flow expiration

Todo flujo deberá expirar.
Ejemplo:

- Login Flow       10 min
- MFA Continuation 5 min
- Step-Up          5 min
- Configurable.

## 22. Flow replay

Un flow completado deberá quedar:

- CONSUMED
- o equivalente.

## 23. AuthenticationFlowRequest

Conceptualmente:

```php
final readonly class AuthenticationFlowRequest
{
    public function __construct(
        public AuthenticationFlowType $type,
        public AuthenticationPurpose $purpose,
        public AuthenticationTransport $transport,
        public AuthenticationInput $input,
        public AuthenticationRequestContext $context,
    ) {}
}
```

## 24. AuthenticationTransport

HTTP_FORM
HTTP_JSON
SPA
API
CLI
INTERNAL

## 25. Transport independence

El Core no deberá depender de:

- HTML redirect
- JSON response
- Vue
- React
- Svelte

## 26. AuthenticationFlowResult

Será transport-agnostic.

```php
interface AuthenticationFlowResult
{
}
```

## 27. Resultados principales

AuthenticationSucceeded
AuthenticationFailed
AuthenticationChallengeRequired
AuthenticationContinuationRequired
AuthenticationRedirectRequired
AuthenticationRecoveryRequired

## 28. AuthenticationSucceeded

Podrá contener:

- AuthenticationContext
- AuthenticationAssurance
- RiskAssessment
- SessionCreationRecommendation
- PersistentLoginRecommendation
- DeviceTrustRecommendation

## 29. Success todavía no significa HTTP Response

El transport adapter decidirá cómo representarlo.

## 30. AuthenticationFailed

Podrá contener:

- failure category
- public failure code
- internal reason
- retry metadata
- audit context

## 31. Public vs internal failure

Importante.

- Internamente:
- IDENTITY_NOT_FOUND
- PASSWORD_INVALID
- ACCOUNT_DISABLED

Externamente puede convertirse en:

- INVALID_CREDENTIALS
- para evitar enumeration.

## 32. AuthenticationChallengeRequired

Ejemplo:

- MFA_REQUIRED
- PASSKEY_REQUIRED
- FRESH_AUTH_REQUIRED
- DEVICE_VERIFICATION_REQUIRED

## 33. Challenge

Debe incluir solo información necesaria para continuar.

## 34. AuthenticationContinuation

Representa:
the flow is not finished

## 35. Continuation Token

Puede utilizarse:
opaque flow continuation reference

## 36. Continuation token security

Debe ser:

- short-lived
- single-purpose
- bound to flow

bound to security context where applicable

## 37. No Authentication state in query strings

Evitar:

- /login/mfa?password=...
- obviamente.

## 38. Form Login

Flujo clásico:

```text
GET /login
    ↓
Login Form
    ↓
POST /login
    ↓
FormLoginEntryPoint
    ↓
AuthenticationFlowCoordinator
```

## 39. Form input

Puede contener:

- identifier
- password
- remember_me
- trust_device

## 40. Input DTO

final readonly class PasswordLoginInput
{
public function __construct(
public string $identifier,
public SensitiveString $password,
public bool $rememberMe,
public bool $trustDevice,
) {}
}

## 41. SensitiveString

Password deberá ser tratado como material sensible.

## 42. Lifecycle

receive
↓
verify
↓
discard
No conservar más tiempo del necesario.

## 43. Form Login completo

POST /login
↓
CSRF
↓
Abuse Protection
↓
Identifier Resolution
↓
Password Authenticator
↓
Identity Eligibility
↓
Risk Engine
↓
Adaptive Authentication
↓
MFA?
┌──┴──┐
NO    YES
│       │
│       ▼
│   MFA Challenge
│       ↓
│   Continuation
│       ↓
│   Factor verified
│       │
└───────┤
▼
Authentication Success
↓
Session Rotation/Creation
↓
Remember-Me?
↓
Trusted Device?
↓
Success Handler

## 44. JSON Login

Entrada:

```php
{
  "email": "<user@example.com>",
  "password": "..."
}
```

Core flow será esencialmente el mismo.

## 45. Diferencia

El adapter produce:

- HTTP JSON
- en lugar de redirects.

## 46. Example success response conceptual

{
"status": "authenticated"
}

## 47. MFA response

Podrá ser:

```php
{
  "status": "challenge_required",
  "challenge": "mfa"
}
```

sin exponer detalles innecesarios.

## 48. SPA Authentication

VoltStack tendrá soporte explícito para SPA.

## 49. SPA no significa necesariamente bearer token

Una SPA same-origin puede utilizar:

```text
secure server-side session

+

CSRF protection
```

## 56. SPA flow

SPA Runtime
↓
POST /auth/login
↓
Authentication Flow
↓
challenge?
↓
SPA receives structured protocol message
↓
renders next step
↓
continues Flow

## 57. SPA Authentication Protocol

Podrá definir estados:

- AUTH_SUCCESS
- AUTH_FAILURE
- AUTH_CHALLENGE
- AUTH_CONTINUE
- AUTH_REDIRECT
- AUTH_RECOVERY

## 58. SPA response envelope

Conceptualmente:

```php
{
  "type": "AUTH_CHALLENGE",
  "flow": "...",
  "challenge": {
    "type": "mfa"
  }
}
```

## 59. No framework coupling

El mismo protocolo deberá poder ser consumido por:

- VoltStack Runtime
- Vue
- React
- Svelte
- vanilla JavaScript

## 60. Passkey Login

No requiere password.

## 61. Flujo

Passkey Login Entry
↓
WebAuthn Authentication Options
↓
Client Credential
↓
PasskeyAuthenticator
↓
Identity Resolution
↓
Risk
↓
Authentication Policy
↓
Session

## 62. Passkey discoverable credentials

Pueden permitir:
identifier-less login

## 63. Identity resolution

Ocurre después de validar correctamente la credential según el modelo WebAuthn.

## 64. Passkey challenge state

Debe estar ligado al Authentication Flow.

## 65. Passkey flow replay

Challenge consumido tras uso.

## 66. Federated Login

Ejemplo:

- Continue with Google
- Continue with Microsoft
- Enterprise SSO

## 67. Inicio

FederatedLoginEntryPoint
↓
Federation State
↓
Redirect to IdP

## 68. Callback

IdP Callback
↓
state validation
↓
OIDC/OAuth validation
↓
Federated Identity Mapping
↓
Local Identity
↓
Risk
↓
Local Authentication Policy

## 69. External success != automatic local login

El IdP puede autenticar correctamente pero VoltStack todavía deberá evaluar:

- Identity eligibility
- Tenant mapping
- Federated assurance
- Risk
- Local step-up requirements

## 70. Federated flow state

Deberá preservar:

- state
- nonce where applicable

PKCE context where applicable
FlowId
intended destination reference

## 71. Federated state expiration

Obligatoria.

## 72. Remember-Me Restoration

No debe confundirse con login explícito.

## 73. Flujo

No active Session
↓
Remember-Me Credential detected
↓
RememberMeEntryPoint
↓
Credential verification
↓
Identity eligibility
↓
Device Trust
↓
Risk
↓
Restoration Policy
↓
Session

## 74. Risk puede requerir Step-Up

Ejemplo:

```text
Remember-Me valid
+
new device
+
new country
    ↓
STEP_UP_REQUIRED
```

## 75. Restored authentication provenance

Debe conservar:
REMEMBER_ME

## 76. Assurance

No deberá inflarse artificialmente al assurance original.

## 77. MFA Continuation

Después de password:

```text
Password valid
    ↓
MFA required
```

no deberá reiniciarse todo el login desde cero.

## 78. Continuation

Se crea:

- AuthenticationFlowState
- con evidencia ya verificada.

## 79. Password no se almacena

Solo se conserva:

- password evidence verified
- o equivalente.

## 80. MFA continuation flow

Flow #123
↓
PasswordEvidence = VERIFIED
↓
Required = SECOND_FACTOR
↓
TOTP submitted
↓
TOTP verified
↓
Assurance recomputed
↓
Risk re-evaluated if necessary
↓
Success

## 81. Step-Up Entry Point

Puede iniciarse desde una sesión ya autenticada.

## 82. Ejemplo

User logged in
↓
Open Billing Settings
↓
Route requires fresh phishing-resistant auth
↓
StepUpEntryPoint
↓
Passkey
↓
Session assurance upgraded
↓
Return to Billing Settings

## 83. Step-Up != new full session necessarily

Puede actualizar:

- AuthenticationContext
- Assurance Snapshot
- Freshness

de la sesión existente.

## 84. Fresh Authentication

Diferente de Step-Up.
Puede exigir:

- repeat existing factor
- para demostrar presencia reciente.

## 85. Example

Password authenticated 6 hours ago
↓
change password
↓
fresh authentication required

## 86. AuthenticationPurpose

Cada flujo tendrá purpose.

- Ejemplos:
- LOGIN
- SESSION_RESTORATION
- STEP_UP
- CHANGE_PASSWORD
- ADD_PASSKEY
- REMOVE_MFA
- TRUST_DEVICE
- ACCOUNT_RECOVERY
- CREATE_API_TOKEN
- SENSITIVE_OPERATION

## 87. Purpose binding

Una evidencia obtenida para:
ACCOUNT_RECOVERY
no deberá reutilizarse automáticamente para:
CREATE_API_TOKEN

## 88. Entry Point Resolver

interface AuthenticationEntryPointResolverInterface
{
public function resolve(
AuthenticationEntryPointContext $context
): AuthenticationEntryPointInterface;
}

## 89. Resolution dimensions

Firewall
Transport
Route
Authentication Purpose
Requested Method
Tenant

## 90. Multiple entry points

Un firewall podrá soportar:

- password
- passkey
- federated
- simultáneamente.

## 91. Login Method Discovery

La UI podrá consultar métodos disponibles.

## 92. AuthenticationMethodDescriptor

Ejemplo:

- PASSWORD
- PASSKEY
- GOOGLE
- MICROSOFT
- ENTERPRISE_SSO

## 93. Enumeration protection

No revelar métodos asociados a una Identity específica antes de Authentication cuando eso permita account discovery.

## 94. Authentication Success Pipeline

Una Authentication exitosa no deberá saltar directamente a redirect.

## 95. Pipeline conceptual

AuthenticationSucceeded
↓
Final Security Validation
↓
Session Establishment
↓
Session ID Rotation
↓
AuthenticationContext Persistence
↓
Remember-Me Issuance
↓
Device Trust Issuance
↓
Security History Update
↓
Audit/Event Dispatch
↓
Success Response

## 96. Ordering

El orden es security-sensitive.

## 97. Final validation

Antes de session establishment:

- Identity still eligible?
- Credential security state still valid?
- Flow not expired?
- Risk requirements satisfied?

## 98. Race conditions

Una Identity puede ser disabled entre:
password verification
y:
session creation

## 99. Therefore

Finalization deberá revalidar invariantes críticas.

## 100. AuthenticationFinalizer

Contrato:

```php
interface AuthenticationFinalizerInterface
{
    public function finalize(
        AuthenticationSucceeded $result,
        AuthenticationFinalizationContext $context
    ): FinalizedAuthentication;
}
```

## 101. Session fixation protection

Login deberá:

- rotate session identifier
- o crear una nueva sesión segura.

## 102. Existing anonymous session

Datos legítimos podrán migrarse según Session subsystem.

## 103. Security identity

No reutilizar session authentication state anterior.

## 104. Session establishment

Será responsabilidad del Session subsystem.

## 105. Remember-Me issuance

Solo si:

- requested
- policy allows
- risk allows
- Identity eligible

## 106. Device Trust issuance

Solo si:

- requested/policy allows
- minimum assurance reached
- risk acceptable
- device eligible

## 107. Authentication Success Handler

Contrato:

```php
interface AuthenticationSuccessHandlerInterface
{
    public function handle(
        FinalizedAuthentication $authentication,
        AuthenticationResponseContext $context
    ): AuthenticationResponse;
}
```

## 108. Success handlers

RedirectAuthenticationSuccessHandler
JsonAuthenticationSuccessHandler
SpaAuthenticationSuccessHandler
ApiAuthenticationSuccessHandler

## 109. Authentication Failure Pipeline

Authentication Failure
↓
Failure Classification
↓
Abuse Protection Feedback
↓
Audit / Telemetry
↓
Public Failure Mapping
↓
Failure Handler

## 110. Failure handler

interface AuthenticationFailureHandlerInterface
{
public function handle(
AuthenticationFailure $failure,
AuthenticationResponseContext $context
): AuthenticationResponse;
}

## 111. Enumeration protection

Ejemplo interno:

- IDENTITY_NOT_FOUND
- PASSWORD_INVALID

pueden producir externamente:
INVALID_CREDENTIALS

## 112. Timing side channels

El flujo deberá colaborar con los mecanismos definidos anteriormente para minimizar diferencias observables.

## 113. Failure response

No revelar:

- email exists
- password correct but MFA disabled
- account has passkey
- account is administrator

salvo contexto autenticado donde sea seguro.

## 114. Account disabled

UX puede requerir mensajes específicos.
Eso deberá estar definido por:
AuthenticationFailureDisclosurePolicy

## 115. FailureDisclosurePolicy

Decide qué información puede exponerse.

## 116. Authentication Challenge Pipeline

Cuando falta evidencia:

```text
AuthenticationChallengeRequired
        ↓
Challenge Planner
        ↓
Available Factors
        ↓
Challenge
        ↓
Continuation Response
```

## 117. Challenge Planner

Pertenece principalmente a MFA/Authenticator orchestration.
Flow system solo coordina.

## 118. Challenge Response

Puede ser:

- HTML
- JSON
- SPA protocol
- redirect
- WebAuthn options

## 119. Intended Destination

Después de login puede existir un destino original.
Ejemplo:
/admin/reports

## 120. IntendedDestination

Nunca debe ser una URL arbitraria confiada ciegamente.

## 121. Open Redirect Protection

Obligatoria.

## 122. Safe destinations

Preferir:

- internal route reference
- signed destination
- validated relative path

## 123. Unsafe

`<https://evil.example>`
//evil.example
javascript:...

## 124. IntendedDestinationResolver

Contrato:

```php
interface IntendedDestinationResolverInterface
{
    public function resolve(
        AuthenticationFlowContext $context
    ): SafeDestination;
}
```

## 125. SafeDestination

Puede representar:

- RouteName
- InternalPath
- ApplicationDestination

## 126. Destination binding

Debe poder ligarse al Flow.

## 127. Step-Up destination

Especialmente importante:

```php
sensitive page
    ↓
step-up
    ↓
return safely
```

## 128. SPA destination

Puede representarse mediante route/state interno, no necesariamente URL completa.

## 129. Login CSRF

Session-based form login deberá protegerse contra CSRF cuando el modelo de aplicación lo requiera.

## 130. Login CSRF matters

Un atacante puede intentar forzar al navegador de la víctima a iniciar sesión en una cuenta controlada por el atacante.

## 131. CSRF policy

Debe integrarse con el Security/HTTP subsystem.

## 132. SPA CSRF

Same-origin session SPA deberá utilizar el mecanismo CSRF correspondiente.

## 133. Bearer API Authentication

Normalmente no utiliza login endpoint para cada request.

## 134. Token issuance

Si existe:

- POST /tokens
- debe ser tratado como operación sensible separada.

## 135. Password Grant

VoltStack no deberá diseñar APIs modernas alrededor del legado OAuth password grant.

## 136. Logout

Logout tampoco será simplemente:
session_destroy();

## 137. LogoutScope

VoltStack deberá soportar scopes explícitos.

## 138. Scopes

CURRENT_SESSION
CURRENT_DEVICE
ALL_SESSIONS
GLOBAL_IDENTITY

## 139. Current Session Logout

invalidate current session
clear current session cookie
clear request authentication context

## 140. Remember-Me behavior

Policy deberá definir si logout actual:

- revokes current Remember-Me credential
- Recomendado para sign-out explícito.

## 141. Trusted Device behavior

Logout no deberá necesariamente:
forget trusted device

## 142. Distinción

Sign Out
y:

- Forget This Device
- son acciones distintas.

## 143. Logout Current Device

Puede significar:
terminate all sessions associated with DeviceReference

## 144. Puede además

Según policy:
revoke Remember-Me credentials on device

## 145. No necesariamente revoca Device Trust

Debe ser opción separada.

## 146. All Sessions Logout

terminate all sessions for Identity
excepto opcionalmente la actual.

## 147. Common UX

Sign out of all other sessions

## 148. Global Identity Logout

Scope más fuerte.

- Puede:
- revoke sessions
- revoke Remember-Me credentials
- invalidate authentication continuations

optionally revoke selected persistent auth credentials

## 149. Global logout != credential reset

No revocar automáticamente:

- Passkeys
- Passwords
- OAuth connections

salvo operación de security reset distinta.

## 150. LogoutCommand

Conceptualmente:

```php
final readonly class LogoutCommand
{
    public function __construct(
        public LogoutScope $scope,
        public LogoutReason $reason,
        public bool $forgetDevice = false,
    ) {}
}
```

## 151. LogoutReason

USER_REQUEST
SECURITY_ACTION
ADMINISTRATIVE
SESSION_EXPIRED
IDENTITY_DISABLED
CREDENTIAL_COMPROMISED

## 152. LogoutManager

interface LogoutManagerInterface
{
public function logout(
LogoutCommand $command,
LogoutContext $context
): LogoutResult;
}

## 153. Logout pipeline

Logout Request
↓
CSRF / Request Validation
↓
Authentication Context
↓
Logout Policy
↓
Session Revocation
↓
Persistent Credential Cleanup
↓
Optional Device Action
↓
Audit
↓
Response

## 154. Logout via GET

No recomendado para state-changing browser logout.

## 155. Preferir

POST /logout
con protección CSRF apropiada.

## 156. Why

Evita:

- image tags
- prefetching
- crawlers
- cross-site links
- causando logout accidental.

## 157. Logout idempotency

Logout deberá ser razonablemente idempotente.

## 158. Example

Dos requests:

- POST /logout
- POST /logout

no deben provocar error de seguridad inesperado.

## 159. Logout Response

Puede ser:

- redirect
- 204
- JSON
- SPA protocol event

## 160. LogoutSuccessHandler

interface LogoutSuccessHandlerInterface
{
public function handle(
LogoutResult $result,
LogoutResponseContext $context
): AuthenticationResponse;
}

## 161. Federated Logout

Complejo.

## 162. Local logout

Cerrar sesión VoltStack no implica necesariamente cerrar sesión en:

- Google
- Microsoft
- enterprise IdP

## 163. IdP logout

Tampoco debe asumirse universalmente.

## 164. FederatedLogoutPolicy

Decide:

- LOCAL_ONLY
- LOCAL_AND_PROVIDER
- PROVIDER_IF_SUPPORTED

## 165. OIDC RP-Initiated Logout

Podrá implementarse mediante provider adapter cuando sea soportado.

## 166. Security boundary

VoltStack no deberá prometer:

- global logout from external identity provider
- si el provider/protocol no lo garantiza.

## 167. Logout callback

Debe validar:

- state
- destination
- provider
- cuando corresponda.

## 168. Session Cookie Clearing

Debe usar mismos atributos relevantes con los que fue creada.

## 169. Persistent cookie cleanup

Incluye:

- Session
- Remember-Me
- Flow/Continuation cookies
- según operación.

## 170. Trusted Device Cookie

Solo eliminar si:

```php
forgetDevice = true
o security policy lo requiere.
```

## 171. Logout and cached AuthenticationContext

Debe invalidarse inmediatamente en request scope.

## 172. Long-running runtimes

Crítico bajo FrankenPHP.

## 173. No stale identity after logout

Nunca:

```php
logout()
Auth::user() // old user
```

dentro del mismo lifecycle lógico.

## 174. AuthenticationContextHolder

Deberá permitir:

- set
- replace
- clear
- de forma request-scoped.

## 175. Entry Point Configuration

Conceptualmente:

```php
'authentication' => [

    'entry_points' => [

        'web' => [
            'form',
            'passkey',
            'federated',
        ],

        'spa' => [
            'json',
            'passkey',
            'federated',
        ],

    ],

];
```

## 176. Login Route Configuration

Ejemplo conceptual:

```php
Auth::routes()
    ->login('/login')
    ->logout('/logout')
    ->mfa('/auth/challenge')
    ->passkey('/auth/passkey')
    ->callback('/auth/{provider}/callback');
```

## 177. Routes optional

Core no deberá obligar a registrar routes mágicas.

## 178. Application-defined routes

También:
Route::post('/sign-in', LoginController::class);

## 179. Headless Authentication

Debe soportarse.

## 180. Headless mode

VoltStack proporciona:

- services
- protocol
- DTOs
- contracts
- sin views.

## 181. Optional UI scaffolding

Puede existir como paquete/tooling separado.

## 182. LoginController minimalista

Ideal:

```php
final class LoginController
{
    public function __invoke(
        LoginRequest $request,
        AuthenticationEntryPoint $auth
    ) {
        return $auth->start($request);
    }
}
```

## 183. No business authentication logic

Evitar:

```php
if ($user->password...) {
    if ($user->mfa...) {
        if ($ip...) {
            ...
        }
    }
}
```

## 184. Flow Policies

Podrán decidir:

- allowed login methods
- required assurance
- allowed federated providers
- session persistence
- device trust
- redirect behavior

## 185. AuthenticationFlowPolicyResolver

interface AuthenticationFlowPolicyResolverInterface
{
public function resolve(
AuthenticationFlowContext $context
): EffectiveAuthenticationFlowPolicy;
}

## 186. Policy hierarchy

Framework Security Floor
↓
Application
↓
Firewall
↓
Tenant
↓
Identity Type
↓
Authentication Purpose
↓
Route/Resource Requirement

## 187. Security floor

Lower policies nunca deben debilitar requisitos no-negociables.

## 188. Flow method selection

Ejemplo:

```text
Admin:
    Passkey OR
    Password + phishing-resistant MFA
```

## 189. Consumer

Password
Passkey
Google
Microsoft

## 190. Authentication Method Choice

Cuando varios métodos sean válidos, el usuario puede elegir.

## 191. Method negotiation

Debe estar representado explícitamente.

## 192. AuthenticationMethodSelection

Puede contener:

- available methods
- preferred method
- required properties

## 193. Do not expose account-specific methods prematurely

Enumeration nuevamente.

## 194. Passwordless Flow

Passkey
↓
Identity
↓
Risk
↓
Session
first-class.

## 195. Mixed Flow

Password
↓
Risk HIGH
↓
Passkey
también first-class.

## 196. Federated + Local Step-Up

OIDC
↓
local Identity
↓
local Risk
↓
Passkey required
también.

## 197. Recovery-to-Login Transition

Recovery completado no debe necesariamente crear una full trusted session.

## 198. Policy options

Recovery completed
↓
require normal login

or

Recovery completed
↓
limited authentication context
↓
Step-Up

## 199. Prefer explicit transition

No mezclar Recovery y Login implícitamente.

## 200. Authentication Completion

Un flow está completo únicamente cuando:

- required evidence satisfied
- Identity eligible
- risk requirements satisfied
- security policy satisfied
- finalization succeeds

## 201. Authentication Atomicity

No todo puede ser una única DB transaction.

## 202. Razón

Puede involucrar:

- external IdP
- WebAuthn
- cache
- session store
- database
- events

## 203. Therefore

Usar:

- state machine
- idempotency
- compensating cleanup

en vez de fingir atomicidad global.

## 204. Finalization failure

Si session creation falla después de Authentication:
do not report authenticated success

## 205. Persistent credential issuance failure

Policy decidirá si:

- login succeeds without Remember-Me
- o si high-security flow falla.

## 206. Device trust issuance failure

Normalmente:

- Authentication can succeed
- device remains untrusted

## 207. Audit failure

Según security profile:

- best effort
- buffered
- fail closed
- para eventos críticos.

## 208. Event Model

Eventos posibles:

- AuthenticationFlowStarted
- AuthenticationFlowContinued
- AuthenticationMethodSelected
- AuthenticationChallengeIssued
- AuthenticationStepUpRequired
- AuthenticationSucceeded
- AuthenticationFailed
- AuthenticationFinalized
- LoginCompleted
- LogoutStarted
- LogoutCompleted
- GlobalLogoutCompleted

## 209. Events no deben contener secrets

Nunca:

- password
- OTP
- raw token
- WebAuthn private material

## 210. Event ordering

Debe definirse.
Ejemplo:

- AuthenticationSucceeded
- puede significar evidence/policy success.

Mientras:

- AuthenticationFinalized
- significa session/context successfully established.

## 211. LoginCompleted

Transport-level user sign-in completion.

## 212. Event listeners

No deberán poder convertir silenciosamente failure en success.

## 213. Hooks

Extensibility deberá usar puntos controlados.

## 214. BeforeAuthentication

Puede enriquecer context, no declarar credentials válidas arbitrariamente salvo authenticator/provider autorizado.

## 215. AfterAuthentication

Puede:

- audit
- analytics
- notifications

## 216. Authentication Flow Middleware

Podrá existir para:

- tenant resolution
- request metadata
- CSRF
- locale
- transport context

## 217. Middleware order

Security-sensitive.

## 218. Login route middleware

No deberá ejecutar middleware que requiera Authentication completa antes del propio login, salvo diseño explícito.

## 219. Entry Point from Protected Resource

Cuando usuario no autenticado accede a recurso protegido:

```text
AuthenticationRequired
        ↓
EntryPointResolver
        ↓
```

Login / Federation / JSON 401

## 220. Browser

Puede producir:
redirect to /login

## 221. JSON API

Debe producir:

- 401
- no redirect HTML.

## 222. SPA

Puede producir protocolo:
AUTH_REQUIRED

## 223. AuthenticationRequiredHandler

Separa esto del Authenticator.

## 224. Unauthorized vs Unauthenticated

Authentication:
401 / authentication required
Authorization:
403 / authenticated but forbidden

## 225. Fundamental boundary

El Entry Point responde a:
No existe Authentication suficiente.

```text
No a:
El usuario no tiene permiso.
```

## 1. Step-Up Required

Puede ser Authentication insufficiency aunque exista Identity autenticada.

## 2. HTTP mapping

Dependiendo de protocolo puede ser:

- 401
- redirect
- challenge response
- SPA continuation

## 3. Authentication Error Taxonomy

Categorías:

- INPUT
- CREDENTIAL
- IDENTITY
- ELIGIBILITY
- ABUSE
- RISK
- CHALLENGE
- FLOW
- SESSION
- PROVIDER
- SYSTEM

## 4. Stable error codes

Ejemplos:

- AUTH_INVALID_CREDENTIALS
- AUTH_FLOW_EXPIRED
- AUTH_CHALLENGE_REQUIRED
- AUTH_ACCOUNT_UNAVAILABLE
- AUTH_RATE_LIMITED
- AUTH_PROVIDER_UNAVAILABLE
- AUTH_SESSION_ESTABLISHMENT_FAILED

## 5. Internal subcodes

Más detallados para logs/audit.

## 6. User-facing messages

Localizables.

## 7. No exception-driven normal control flow

MFA required
no debería necesariamente ser una exception.
Es un:
AuthenticationChallengeRequired

## 8. Exceptional failures

Ejemplo:

- session store unavailable
- invalid framework configuration
- cryptographic provider failure

sí pueden generar exceptions controladas.

## 9. Authentication Response Mapping

Domain Result
↓
Transport Mapper
↓
HTTP/SPA/API Response

## 10. Response mapper

interface AuthenticationResponseMapperInterface
{
public function map(
AuthenticationFlowResult $result,
AuthenticationTransportContext $context
): mixed;
}

## 11. HTML mapper

SUCCESS → redirect
FAILURE → form error
CHALLENGE → challenge page

## 12. JSON mapper

SUCCESS → 200/204
FAILURE → 401/appropriate status
CHALLENGE → structured challenge

## 13. SPA mapper

SUCCESS → AUTH_SUCCESS
FAILURE → AUTH_FAILURE
CHALLENGE → AUTH_CHALLENGE

## 14. Security Headers

Authentication responses deberán colaborar con HTTP security subsystem.
Especialmente:

- Cache-Control
- Referrer-Policy
- CSP
- según páginas sensibles.

## 15. Login pages and caching

Credenciales/form state no deberán terminar en shared caches.

## 16. Browser autofill

VoltStack no deberá romper password managers innecesariamente.

## 17. Standard field semantics

UI scaffolding debería usar:

```php
autocomplete="username"
autocomplete="current-password"
autocomplete="one-time-code"
cuando corresponda.
```

## 18. Password Managers

Son aliados de seguridad.

## 19. Login identifier normalization

Debe ocurrir en Identity subsystem según identifier type.

```php
No hacer:
strtolower($everything);
universalmente.
```

## 20. Login Attempt ID

Separado de FlowId.

## 21. Attempt

Representa una evaluación concreta.

## 22. Flow

Puede contener múltiples attempts/challenges.

## 23. Example

Flow #F1

Attempt #A1 password success
Attempt #A2 TOTP failure
Attempt #A3 TOTP success

## 249. Audit correlation

FlowId
AttemptId
SessionPublicId
permiten tracing sin secrets.

## 250. Rate limiting

Debe poder operar sobre:

- attempt
- flow
- identity candidate
- network
- device
- según documento 19.

## 251. Restarting flows

No debe ser mecanismo para evadir throttling.

## 252. Flow creation rate limit

Puede existir.

## 253. MFA challenge rate limit

También.

## 254. Flow invalidation

Puede ocurrir por:

- timeout
- too many failures

Identity security version change
password reset
account disable
manual cancellation

## 255. SecurityVersion binding

Un Flow puede registrar:
IdentitySecurityVersion

## 256. If changed

Flow puede requerir:
restart

## 257. Example

Password verified
↓
waiting MFA
↓
administrator disables account
↓
MFA submitted
↓
final validation rejects flow

## 258. Critical invariant

Verified evidence does not guarantee indefinite eligibility.

## 259. Parallel login flows

Permitidos según policy.

## 260. Cross-flow evidence reuse

No por default.

## 261. Why

Flow binding previene:

- challenge swapping
- state confusion
- replay

## 262. MFA challenge belongs to Flow

Obligatorio.

## 263. WebAuthn challenge belongs to Flow

Obligatorio.

## 264. Federated state belongs to Flow

Obligatorio.

## 265. Recovery challenge belongs to Recovery Flow

No mezclar.

## 266. Authentication Flow State Machine

┌──────────────┐
│   STARTED    │
└──────┬───────┘
▼
┌─────────────────┐
│ IDENTITY/EVIDENCE│
└────────┬────────┘
▼
┌────────────────────┐
│ POLICY EVALUATION  │
└───────┬───────┬────┘
│       │
satisfied  more evidence
│       │
│       ▼
│  ┌───────────────┐
│  │   CHALLENGE   │
│  └───────┬───────┘
│          │
│          ▼
│    CONTINUATION
│          │
└─────┬────┘
▼
FINAL VALIDATION
│
┌───────┴────────┐
▼                ▼
COMPLETED          FAILED

## 267. Cancelled Flow

Usuario puede cancelar:

- MFA
- Passkey
- Federated login

## 268. Cancellation

No equivale necesariamente a credential failure.

## 269. Audit distinction

CANCELLED
vs:
FAILED

## 270. Browser Back Button

Flows deberán manejar:

- stale forms
- consumed challenges
- duplicate submissions
- sin inconsistencias.

## 271. Duplicate form submission

Idempotency/flow state debe impedir doble session finalization peligrosa.

## 272. Finalization lock

Puede existir:

- FlowFinalizationLock
- o atomic consume.

## 273. Exactly-once illusion

No siempre posible distribuido.
Diseñar operaciones idempotentes.

## 274. Session creation idempotency

Flow finalization deberá evitar múltiples sesiones accidentales cuando policy lo requiera.

## 275. Redirect after POST

HTML flow debería preferir:

```text
POST
↓
Redirect
↓
GET
cuando corresponda.
```

## 276. Authentication UI independence

Core no deberá generar HTML directamente.

## 277. UI package

Podrá proporcionar:

- Login Form
- MFA Form
- Passkey UI
- Federated Buttons
- Recovery UI
- Device Trust Prompt
- en otro módulo.

## 278. Framework APIs

Ejemplos conceptuales:

```php
Auth::login()->start($credentials);

Auth::flow($id)->continue($evidence);

Auth::logout()->current();

Auth::logout()->device();

Auth::logout()->all();
```

## 279. Lower-level API

$flow = $authenticationFlows->start(
AuthenticationFlowRequest::login(...)
);

## 280. Facade convenience

No deberá ocultar security semantics.

## 281. Programmatic Login

Debe existir pero con cuidado.

## 282. Example

Auth::establish($identity, $evidence);
no debería aceptar arbitrariamente:

```php
Auth::login($user);
sin provenance.
```

## 283. Trusted programmatic authentication

Puede requerir:
TrustedAuthenticationGrant

## 284. Why

Evita que cualquier application code cree una full-assurance session sin evidencia.

## 285. Compatibility API

VoltStack puede ofrecer convenience similar a frameworks conocidos, pero internamente deberá crear provenance explícita.

## 286. Impersonation

No deberá modelarse como login normal.
Pertenece al sistema de Authorization/Delegation/Impersonation correspondiente.

## 287. Testing — Form Login

valid credentials
invalid credentials
unknown Identity
disabled Identity
expired password

## 288. Testing — MFA

password valid + MFA required
MFA failure
MFA success
flow expiration
challenge replay

## 289. Testing — Passkey

identifier-first
discoverable credential
invalid assertion
expired challenge
replayed challenge

## 290. Testing — Federation

state mismatch
nonce mismatch
callback replay
unknown provider
mapping failure
local step-up required

## 291. Testing — Remember-Me

valid restoration
revoked credential
high-risk restoration
Identity disabled

## 292. Testing — Risk

low risk login
step-up required
critical deny
risk changes during flow

## 293. Testing — Device Trust

trust requested
insufficient assurance
high risk
successful issuance
issuance failure

## 294. Testing — Session

session rotation
session creation failure
anonymous session migration
fixation protection

## 295. Testing — Redirect

safe internal path
external URL
protocol-relative URL
encoded bypass attempts
step-up return path

## 296. Testing — Enumeration

Compare externally observable behavior for:

- unknown identity
- wrong password

## 297. Testing — Logout

current session
current device
all sessions
global
forget device

## 298. Testing — Logout CSRF

Cross-site requests must not trigger browser logout where CSRF protection is required.

## 299. Testing — Federated Logout

local only
provider supported
provider failure
callback state mismatch

## 300. Testing — Concurrent Finalization

Two submissions of same successful Flow.
Solo resultado válido según idempotency policy.

## 301. Testing — Security State Race

Disable Identity during MFA continuation.

## 302. Testing — Flow Restart Abuse

Restarting flow does not reset abuse counters improperly.

## 303. Testing — Multi-Tenant

Flow Tenant A
cannot continue under Tenant B

## 304. Testing — Security Realm

Admin flow no puede reutilizar consumer flow evidence incorrectamente.

## 305. Testing — SPA

AUTH_SUCCESS
AUTH_FAILURE
AUTH_CHALLENGE
AUTH_CONTINUE
AUTH_REQUIRED

## 306. Testing — JSON

Verify stable machine-readable errors.

## 307. Testing — Headless

Core works without views/routes scaffolding.

## 308. Testing — FrankenPHP

Sequential requests:

```text
Request 1:
Alice login
```

Request 2:
Anonymous

Request 3:

- Bob login
- No state leakage.

## 309. Testing — Concurrent FrankenPHP

Flow Alice
Flow Bob
Flow Charlie
simultáneos y aislados.

## 310. Testing — Fiber Safety

Cada flow context separado.

## 311. Testing — Distributed Runtime

Flow iniciado en Node A puede continuar en Node B cuando store configurado sea distribuido.

## 312. Testing — Failure Injection

Session store down
Flow store down
Risk provider timeout
Audit unavailable
Remember-Me issuance failure

## 313. Property-based Testing

Útil para:

- flow state transitions
- destination validation
- logout scopes
- challenge consumption
- continuation expiry

## 314. Fuzz Testing

Especialmente:

- redirect destinations
- continuation tokens
- federated callback parameters
- JSON login payloads
- flow identifiers

## 315. Security invariants — Flow

AUTH-FLOW-01
Authentication flows are explicit state machines.
AUTH-FLOW-02
Verified credentials are not persisted as raw secrets in Flow State.
AUTH-FLOW-03
Flow continuations are short-lived and purpose-bound.

- AUTH-FLOW-04
- Completed challenges cannot be replayed.
- AUTH-FLOW-05

Flow completion requires final security validation.
AUTH-FLOW-06
Security-state changes may invalidate an in-progress flow.

## 316. Security invariants — Entry Points

AUTH-ENTRY-01
Entry Points initiate Authentication but do not validate credentials themselves unless delegating to the proper Authenticator.
AUTH-ENTRY-02
Entry Point selection respects Firewall, Transport and Authentication Purpose.
AUTH-ENTRY-03
Unauthenticated and Unauthorized states remain distinct.
AUTH-ENTRY-04
Transport-specific behavior does not leak into Authentication Core.

## 317. Security invariants — Login

AUTH-LOGIN-01
Successful credential verification alone does not establish a Session.
AUTH-LOGIN-02
Identity eligibility is revalidated before finalization where required.
AUTH-LOGIN-03
Interactive Session establishment prevents session fixation.

- AUTH-LOGIN-04
- Remember-Me and Device Trust issuance require independent policy approval.
- AUTH-LOGIN-05

Authentication success preserves evidence provenance and assurance.

## 318. Security invariants — Continuation

AUTH-CONT-01
MFA continuation never requires storing the original raw password.
AUTH-CONT-02
Challenges are bound to their originating Flow.
AUTH-CONT-03
Continuation cannot cross Tenant/Security Realm boundaries unless explicitly designed.
AUTH-CONT-04
Expired flows cannot be silently revived.
AUTH-CONT-05
Parallel flow evidence is not interchangeable by default.

## 319. Security invariants — Redirect

AUTH-REDIRECT-01
User-controlled destinations are never trusted directly.

- AUTH-REDIRECT-02
- Authentication redirects prevent open redirect attacks.
- AUTH-REDIRECT-03

Sensitive continuation state is not encoded in arbitrary destination URLs.

## 320. Security invariants — Logout

AUTH-LOGOUT-01
Logout scope is explicit.

- AUTH-LOGOUT-02
- Browser state-changing logout uses an appropriate non-GET method by default.
- AUTH-LOGOUT-03

Logout invalidates AuthenticationContext immediately.
AUTH-LOGOUT-04
Sign-Out does not implicitly mean Forget Device unless policy requests it.
AUTH-LOGOUT-05
Global logout does not silently revoke unrelated permanent Identity credentials.
AUTH-LOGOUT-06
Logout is idempotent where practical.

## 321. Security invariants — SPA/API

AUTH-TRANSPORT-01
SPA, JSON and HTML transports consume the same domain-level Authentication results.
AUTH-TRANSPORT-02
Session-based SPA authentication preserves CSRF protections.
AUTH-TRANSPORT-03
API clients are not redirected to HTML login unless explicitly configured.
AUTH-TRANSPORT-04
Machine-readable Authentication errors remain stable and do not expose secrets.

## 322. Security invariants — Runtime

AUTH-FLOW-RT-01
Current Authentication Flow context is request-scoped.

- AUTH-FLOW-RT-02
- Flow coordinators do not retain mutable Identity state across requests.
- AUTH-FLOW-RT-03

FrankenPHP workers do not retain previous Authentication state.
AUTH-FLOW-RT-04
Concurrent fibers have isolated Flow contexts.
AUTH-FLOW-RT-05
Distributed Flow state uses authoritative shared storage where cross-node continuation is required.

## 323. Anti-pattern — Giant LoginController

Evitar:

```php
if ($password) {
    if ($mfa) {
        if ($risk) {
            if ($remember) {
                ...
            }
        }
    }
}
```

## 324. Anti-pattern — Password verification creates session directly

No.

## 325. Anti-pattern — MFA required represented only as exception

No como única abstracción.

## 326. Anti-pattern — Raw password stored in session between MFA steps

Nunca.

## 327. Anti-pattern — Redirect URL trusted from request

Nunca.

## 328. Anti-pattern — GET /logout

No como default.

## 329. Anti-pattern — Logout always deletes trusted device

No.

## 330. Anti-pattern — External IdP success means unconditional local session

No.

## 331. Anti-pattern — Remember-Me equals full fresh login

No.

## 332. Anti-pattern — SPA requires JWT

No.

## 333. Anti-pattern — HTML semantics inside Core

No.

## 334. Anti-pattern — Auth::login($user) creates arbitrary high-assurance session

No sin provenance/security contract.

## 335. Anti-pattern — Flow state stored in singleton

Especialmente peligroso bajo FrankenPHP.

## 336. Componentes principales

AuthenticationFlow
AuthenticationFlowId
AuthenticationFlowType
AuthenticationFlowStatus
AuthenticationFlowState

AuthenticationFlowRequest
AuthenticationFlowResult
AuthenticationFlowCoordinator
AuthenticationFlowStateStore
AuthenticationFlowPolicyResolver

## 337. Entry Point components

AuthenticationEntryPoint
AuthenticationEntryPointResolver

FormLoginEntryPoint
JsonLoginEntryPoint
SpaLoginEntryPoint
PasskeyEntryPoint
FederatedLoginEntryPoint
RememberMeEntryPoint
StepUpEntryPoint
RecoveryEntryPoint

## 338. Result components

AuthenticationSucceeded
AuthenticationFailed
AuthenticationChallengeRequired
AuthenticationContinuationRequired
AuthenticationRedirectRequired
AuthenticationRecoveryRequired

## 339. Finalization components

AuthenticationFinalizer
FinalizedAuthentication

AuthenticationSuccessHandler
AuthenticationFailureHandler
AuthenticationRequiredHandler
AuthenticationFailureDisclosurePolicy

## 340. Redirect components

IntendedDestination
SafeDestination
IntendedDestinationResolver
AuthenticationRedirectPolicy

## 341. Logout components

LogoutManager
LogoutCommand
LogoutScope
LogoutReason
LogoutResult
LogoutSuccessHandler
FederatedLogoutPolicy

## 342. Transport components

AuthenticationTransport
AuthenticationResponseMapper

HtmlAuthenticationResponseMapper
JsonAuthenticationResponseMapper
SpaAuthenticationResponseMapper

## 343. Namespace sugerido

VoltStack\Quantum\Auth\Flow
VoltStack\Quantum\Auth\Flow\Contracts
VoltStack\Quantum\Auth\Flow\EntryPoint
VoltStack\Quantum\Auth\Flow\Continuation
VoltStack\Quantum\Auth\Flow\Result
VoltStack\Quantum\Auth\Flow\Finalization
VoltStack\Quantum\Auth\Flow\Response
VoltStack\Quantum\Auth\Flow\Redirect
VoltStack\Quantum\Auth\Logout

## 344. Estructura sugerida

src/Quantum/Auth/
├── Flow/
│   ├── Contracts/
│   │   ├── AuthenticationFlowCoordinatorInterface.php
│   │   ├── AuthenticationFlowStateStoreInterface.php
│   │   ├── AuthenticationEntryPointInterface.php
│   │   ├── AuthenticationEntryPointResolverInterface.php
│   │   └── AuthenticationFlowPolicyResolverInterface.php
│   │
│   ├── EntryPoint/
│   │   ├── FormLoginEntryPoint.php
│   │   ├── JsonLoginEntryPoint.php
│   │   ├── SpaLoginEntryPoint.php
│   │   ├── PasskeyEntryPoint.php
│   │   ├── FederatedLoginEntryPoint.php
│   │   ├── RememberMeEntryPoint.php
│   │   └── StepUpEntryPoint.php
│   │
│   ├── Continuation/
│   │   ├── AuthenticationFlowContinuation.php
│   │   ├── AuthenticationContinuationToken.php
│   │   └── AuthenticationChallengeState.php
│   │
│   ├── Result/
│   │   ├── AuthenticationSucceeded.php
│   │   ├── AuthenticationFailed.php
│   │   ├── AuthenticationChallengeRequired.php
│   │   └── AuthenticationContinuationRequired.php
│   │
│   ├── Finalization/
│   │   ├── AuthenticationFinalizer.php
│   │   └── FinalizedAuthentication.php
│   │
│   ├── Redirect/
│   │   ├── IntendedDestination.php
│   │   ├── SafeDestination.php
│   │   └── IntendedDestinationResolver.php
│   │
│   ├── Response/
│   │   ├── AuthenticationSuccessHandler.php
│   │   ├── AuthenticationFailureHandler.php
│   │   ├── HtmlAuthenticationResponseMapper.php
│   │   ├── JsonAuthenticationResponseMapper.php
│   │   └── SpaAuthenticationResponseMapper.php
│   │
│   ├── AuthenticationFlow.php
│   ├── AuthenticationFlowId.php
│   ├── AuthenticationFlowState.php
│   ├── AuthenticationFlowStatus.php
│   └── AuthenticationFlowCoordinator.php
│
└── Logout/
├── LogoutManager.php
├── LogoutCommand.php
├── LogoutScope.php
├── LogoutReason.php
├── LogoutResult.php
└── FederatedLogoutPolicy.php

## 345. Flujo completo Password + MFA

POST /login
│
▼
FormLoginEntryPoint
│
▼
AuthenticationFlowCoordinator
│
▼
PasswordAuthenticator
│
▼
PasswordEvidence VERIFIED
│
▼
Identity Eligibility
│
▼
Risk Engine
│
▼
MFA REQUIRED
│
▼
AuthenticationChallengeRequired
│
▼
Flow State persisted
│
▼
MFA UI
│
▼
POST challenge
│
▼
Flow resumed
│
▼
MFA Authenticator
│
▼
Evidence VERIFIED
│
▼
Final Risk / Policy Evaluation
│
▼
AuthenticationSucceeded
│
▼
AuthenticationFinalizer
│
├── Session
├── Remember-Me
├── Device Trust
└── History
│
▼
RedirectAuthenticationSuccessHandler

## 346. Flujo Passkey passwordless

Sign In
↓
Passkey Entry Point
↓
WebAuthn Challenge
↓
Browser Authenticator
↓
Assertion
↓
PasskeyAuthenticator
↓
Identity resolved
↓
Risk
↓
Policy
↓
Authentication Finalization
↓
Session

## 347. Flujo Federated + Step-Up

Continue with Enterprise SSO
↓
OIDC Redirect
↓
IdP Authentication
↓
OIDC Callback
↓
Local Identity Mapping
↓
Federated Assurance = MEDIUM
↓
Admin Policy requires
phishing-resistant local evidence
↓
Passkey Challenge
↓
Passkey VERIFIED
↓
Authentication Finalization

## 348. Flujo Remember-Me + Risk

Request
↓
No Session
↓
Remember-Me found
↓
Credential VERIFIED
↓
Identity eligible
↓
Device UNKNOWN
↓
Risk HIGH
↓
Step-Up
↓
Passkey
↓
Session established

## 349. Flujo Step-Up

Authenticated Session
↓
Sensitive Operation
↓
Authentication Requirement:

```text
fresh phishing-resistant
        ↓
StepUpEntryPoint
        ↓
Passkey
        ↓
Verified Evidence
        ↓
```

Session Assurance Snapshot updated
↓
Safe Intended Destination
↓
Sensitive Operation

## 350. Flujo Logout

POST /logout
↓
CSRF Validation
↓
LogoutManager
↓
Scope = CURRENT_SESSION
↓
Session revoked
↓
Remember-Me credential revoked
↓
AuthenticationContext cleared
↓
Audit
↓
Redirect

## 351. Flujo Global Logout

Security Settings
↓
Sign out everywhere
↓
Fresh Authentication
↓
LogoutManager
↓
GLOBAL_IDENTITY
│
├── revoke sessions
├── revoke Remember-Me
├── invalidate active flows
└── clear current context
│
▼
Audit
↓
Login Required

## 352. Arquitectura global

APPLICATION
│
┌───────────────┼───────────────┐
▼               ▼               ▼
HTML             SPA             API
│               │               │
└───────────────┼───────────────┘
▼
AUTHENTICATION ENTRY POINT
│
▼
AUTHENTICATION FLOW SYSTEM
│
┌──────────────────┼───────────────────┐
▼                  ▼                   ▼
AUTHENTICATORS       FLOW STATE          POLICIES
│                                      │
├───────────────┬──────────────────────┤
▼               ▼                      ▼
IDENTITY           RISK                  MFA
│               │                      │
├───────────────┼──────────────────────┤
▼               ▼                      ▼
PASSKEY          DEVICE TRUST          FEDERATION
│               │                      │
└───────────────┼──────────────────────┘
▼
AUTHENTICATION RESULT
│
┌─────────┴──────────┐
▼                    ▼
SUCCESS              FAILURE
│
▼
AUTHENTICATION FINALIZER
│
┌───────┼────────┬──────────┐
▼       ▼        ▼          ▼
SESSION REMEMBER  DEVICE     HISTORY
ME       TRUST
│
▼
RESPONSE MAPPER
│
┌────┼─────┐
▼    ▼     ▼
HTML  SPA   JSON

## 353. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Login/Logout are orchestration flows, not Authenticator responsibilities

## 2. Authentication flows are explicit state machines

## 3. Controllers and transport adapters remain thin

## 4. Multi-step Authentication uses secure continuation state

## 5. Raw credentials are never persisted between flow steps

## 6. Password, MFA, Passkey, Federation, Risk, Device and Session remain

independent subsystems.

## 7. Authentication finalization revalidates critical security invariants

## 8. Session fixation protection is mandatory for interactive login

## 9. Remember-Me and Device Trust issuance are independent decisions

## 10. HTML, SPA and JSON share the same domain Authentication pipeline

## 11. Intended destinations are validated against open redirects

## 12. Logout has explicit scopes

## 13. Sign-Out and Forget Device are separate operations

## 14. Federated Authentication success does not automatically imply local success

## 15. Authentication Flow state is safe for FrankenPHP, fibers and

distributed runtimes.

## 16. Diferencias importantes frente al modelo tradicional

Modelo simplificado tradicional:

```php
Controller
    ↓
Auth::attempt()
    ↓
Session
    ↓
Redirect
```

VoltStack evolucionará hacia:

```php
Entry Point
    ↓
Flow Coordinator
    ↓
Authenticator(s)
    ↓
Evidence
    ↓
Eligibility
    ↓
Risk
    ↓
Adaptive Requirements
    ↓
```

MFA / Passkey / Federation
↓
Final Validation
↓
Authentication Finalizer
↓
Session / Persistent Credentials / Device
↓
Transport Response
La intención no es hacer más complejo el uso cotidiano.
La API de alto nivel podrá seguir siendo sencilla:

```php
return Auth::login($request);
pero internamente conservará todas las garantías del pipeline completo.
```

## 17. Relación con Laravel y Symfony

VoltStack puede aprovechar ideas de ambos enfoques.
Del modelo de Laravel resulta valiosa la ergonomía de conceptos como:

- guards
- providers
- attempt
- login
- logout
- intended destination
- session regeneration
- middleware

Del modelo de Symfony resulta especialmente valiosa la separación entre:

- firewall
- authenticator
- passport
- entry point
- success handler
- failure handler
- remember-me
- authentication events

VoltStack llevará esta separación más lejos mediante:

- AuthenticationFlow
- AuthenticationFlowCoordinator
- AuthenticationContinuation
- AuthenticationFinalizer
- Risk-aware Flow
- MFA-aware Flow
- Passkey-native Flow
- SPA Authentication Protocol
- Explicit Logout Scopes
- Device-aware Authentication

La meta será conservar:

```text
Laravel-like developer ergonomics

+

Symfony-like security decomposition
+
```

VoltStack-native adaptive authentication architecture

## 356. Criterios de aceptación

El sistema será considerado completo cuando:

1. soporte Authentication Entry Points;
2. soporte Flow Coordinator;
3. soporte Flow IDs;
4. soporte Flow State;
5. soporte Flow expiration;
6. soporte continuation;
7. soporte challenge state;
8. soporte Form Login;
9. soporte JSON Login;
10. soporte SPA Login;
11. soporte Passkey Login;
12. soporte Federated Login;
13. soporte Remember-Me restoration;
14. soporte MFA continuation;
15. soporte Step-Up;
16. soporte Fresh Authentication;
17. soporte Authentication Purpose;
18. soporte Success/Failure/Challenge results;
19. soporte Authentication Finalization;
20. revalide Identity eligibility;
21. prevenga session fixation;
22. soporte Remember-Me issuance;
23. soporte Device Trust issuance;
24. soporte Authentication History update;
25. soporte Success Handlers;
26. soporte Failure Handlers;
27. soporte safe intended destinations;
28. prevenga open redirects;
29. soporte CSRF-safe browser login/logout;
30. soporte CURRENT_SESSION logout;
31. soporte CURRENT_DEVICE logout;
32. soporte ALL_SESSIONS logout;
33. soporte GLOBAL_IDENTITY logout;
34. distinga Sign-Out y Forget Device;
35. soporte Federated Logout policy;
36. soporte HTML response mapping;
37. soporte JSON mapping;
38. soporte SPA protocol mapping;
39. soporte stable error taxonomy;
40. soporte audit;
41. soporte events;
42. soporte metrics/tracing;
43. soporte multi-tenancy;
44. soporte Security Realm isolation;
45. soporte distributed Flow State;
46. soporte idempotent finalization;
47. sea fiber-safe;
48. sea seguro bajo FrankenPHP.
49. Regla arquitectónica final

VoltStack deberá preservar:

```text
ENTRY POINT
     │
     ▼
AUTHENTICATION FLOW
     │
     ▼
AUTHENTICATOR RESOLUTION
     │
     ▼
VERIFIED EVIDENCE
     │
     ▼
IDENTITY + ELIGIBILITY
     │
     ▼
```

RISK + AUTHENTICATION POLICY
│
├───────────────┐
│               │
▼               ▼
SUFFICIENT      MORE EVIDENCE
│               │
│          MFA / PASSKEY
│               │
└───────┬───────┘
▼
FINAL VALIDATION
│
▼
AUTHENTICATION FINALIZER
│
┌───────┼────────┬─────────┐
▼       ▼        ▼         ▼
SESSION REMEMBER  DEVICE    HISTORY
ME       TRUST
│
▼
TRANSPORT RESPONSE
La primera regla será:
VoltStack no tendrá un LoginController que sea, en realidad, todo el sistema de Authentication disfrazado de controller.

La segunda:
Sign-In será un flujo explícito que transforma evidencia verificable en un AuthenticationContext únicamente después de satisfacer Identity, Risk, Assurance y Authentication Policy.

La tercera:
Sign-Out será una operación de seguridad con alcance explícito, capaz de distinguir entre cerrar una sesión, cerrar las sesiones de un dispositivo, cerrar todas las sesiones y ejecutar una invalidación global de Authentication.

La cuarta:
HTML, SPA, JSON y futuros transports serán representaciones diferentes del mismo modelo de Authentication, no implementaciones diferentes de sus reglas de seguridad.

Siguiente documento recomendado
La secuencia natural puede continuar con:
`23_AUTHENTICATION_EVENTS_HOOKS_LISTENERS_SUBSCRIBERS_AND_EXTENSION_LIFECYCLE_SYSTEM.md`
Aquí conviene formalizar el sistema de extensión antes de avanzar hacia observabilidad y testing, porque ya tenemos numerosos puntos donde otros módulos necesitarán reaccionar:

- AuthenticationStarted
- AuthenticationAttempted
- IdentityResolved
- CredentialVerified
- AuthenticationFailed
- ChallengeRequired
- MFACompleted
- RiskAssessed
- StepUpRequired
- AuthenticationSucceeded
- AuthenticationFinalized
- SessionEstablished
- RememberMeIssued
- DeviceTrusted
- LogoutStarted
- LogoutCompleted
- AuthenticationEvents
- Security Events
- Domain Events
- Hooks
- Listeners
- Subscribers
- Priorities
- Synchronous Listeners
- Deferred Listeners
- Transactional Events
- Event Ordering
- Event Payload Security
- Secret Redaction
- Listener Failure Policy
- Critical Security Listeners
- Extension Points
- Plugin Integration
- Tenant-aware Events
- Observability Integration
- FrankenPHP Safety

Esto nos permitirá evitar otro problema arquitectónico importante: que cada subsistema de Authentication termine llamando directamente a correo, auditoría, notificaciones, analytics, plugins o lógica de aplicación.
