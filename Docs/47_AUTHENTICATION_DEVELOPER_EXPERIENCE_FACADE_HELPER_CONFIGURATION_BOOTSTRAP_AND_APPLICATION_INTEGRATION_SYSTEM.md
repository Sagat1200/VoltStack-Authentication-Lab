# VoltStack Authentication System

## 47 — Authentication Developer Experience, Facade, Helper, Configuration, Bootstrap and Application Integration System

- **Archivo:** `47_AUTHENTICATION_DEVELOPER_EXPERIENCE_FACADE_HELPER_CONFIGURATION_BOOTSTRAP_AND_APPLICATION_INTEGRATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Developer Experience / Framework Integration / Security-Critical API Surface
- **Dependencias:** 01–46 AUTHENTICATION_*
- **Objetivo:** convertir toda la arquitectura interna de Authentication en una experiencia de desarrollo coherente, segura, expresiva y compatible con aplicaciones HTTP, SPA, API, CLI, workers, FrankenPHP y entornos distribuidos.

---

## 1. Propósito

Los documentos 01–46 definen una arquitectura Authentication extremadamente amplia:
Identity
Credentials
Authenticators
Evidence
Sessions
Remember-Me
Tokens
MFA
Passkeys
Federation
Recovery
Risk
Device Trust
Policies
Assurance
Challenges
Transactions
Replay Protection
Security Incidents
Identity Lifecycle
Privacy
Notifications
Background Tasks
Rate Governance
Migration
Sin embargo, una arquitectura completa no garantiza una buena experiencia de desarrollo.
El desarrollador no debería necesitar conocer 40 subsistemas para realizar:
Auth::user();
ni escribir veinte servicios para:
Auth::attempt($credentials);
El propósito de este documento es definir la capa de experiencia de desarrollo de Authentication.

## 2. Objetivo principal

VoltStack deberá proporcionar una superficie simple:
Auth::check();

Auth::guest();

Auth::user();

Auth::id();

Auth::attempt([
    'email' => $email,
    'password' => $password,
]);

Auth::logout();
mientras internamente puede ejecutar:
Identity Resolution
       ↓
Authenticator Resolution
       ↓
Credential Verification
       ↓
Authentication Evidence
       ↓
Eligibility
       ↓
Risk
       ↓
Policy
       ↓
Assurance
       ↓
Challenge Negotiation
       ↓
Session Establishment
       ↓
Events / Audit

## 3. Principio fundamental

Simple API does not mean simple security model.

La complejidad deberá estar encapsulada, no eliminada.

## 4. Segundo principio

Developer Experience nunca podrá convertirse en un bypass del Authentication Core.

Una facade:
Auth::login($user);
no deberá significar:
$_SESSION['user_id'] = $user->id;

## 5. Tercer principio

Convenience API
      ↓
Application Authentication API
      ↓
Authentication Orchestrator
      ↓
Security Core
Nunca:
Convenience API
      ↓
Session storage directly

## 6. Cuarto principio

La experiencia deberá ser:
Laravel-like simplicity
        +
Symfony-like explicit architecture
        +
VoltStack security semantics

## 7. Capas de Developer Experience

┌───────────────────────────────────────┐
│ Application Code                      │
│ Controllers / Actions / Components    │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Helpers / Facades / Attributes        │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Application Authentication API        │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Authentication Manager / Orchestrator │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Authentication Security Core          │
└───────────────────────────────────────┘

## 8. Public API Surface

La API pública deberá dividirse entre:
Facade API
Helper API
Contract API
Attribute API
Middleware API
Fluent API
Context API
CLI API
Testing API

## 9. Authentication Facade

Namespace sugerido:
VoltStack\Facades\Auth
o internamente:
VoltStack\Quantum\Auth\Facades\Auth
con alias público:
use VoltStack\Facades\Auth;

## 10. Facade Responsibility

La facade será exclusivamente un proxy hacia servicios request-scoped.
No contendrá:
current user
current session
credentials
mutable authentication state

## 11. FrankenPHP Critical Rule

Esto sería incorrecto:
class Auth
{
    private static ?User $user = null;
}
En persistent workers podría producir:
Request A → User A
worker reused
Request B → accidentally sees User A

## 12. Correct Facade Model

Auth Facade
    ↓
Container
    ↓
Current Request Scope
    ↓
AuthenticationContextAccessor

## 13. Core Facade Methods

La primera superficie podría incluir:
Auth::check();

Auth::guest();

Auth::user();

Auth::id();

Auth::context();

Auth::session();

Auth::methods();

Auth::assurance();

Auth::attempt();

Auth::login();

Auth::logout();

Auth::logoutOtherSessions();

Auth::reauthenticate();

Auth::require();

Auth::security();

## 14. Auth::check()

if (Auth::check()) {
    // authenticated
}
Semántica:
Existe un principal autenticado válido dentro del contexto actual.

No significa:
authorized
privileged
high assurance
fresh authentication
trusted device

## 15. Auth::guest()

if (Auth::guest()) {
    // unauthenticated
}
Equivalente lógico:
! Auth::check()
pero puede exponerse por ergonomía.

## 16. Auth::user()

$user = Auth::user();
Retorna el principal humano/application-facing cuando corresponda.
Conceptualmente:
?AuthenticatablePrincipal

## 17. Auth::principal()

VoltStack debería ofrecer además:
$principal = Auth::principal();
porque no toda identidad será un usuario humano.
Puede representar:
Human
Service
Workload
Application
Device
Automation

## 18. Auth::user() vs Auth::principal()

Auth::principal()
    ↓
Any authenticated principal

Auth::user()
    ↓
Human/application user projection
Esto evita forzar machine identities a convertirse en usuarios falsos.

## 19. Auth::id()

$id = Auth::id();
deberá devolver:
?IdentityId
o una representación application-friendly configurable.

## 20. Typed Identity

Internamente se recomienda:
IdentityId
no strings arbitrarios.

## 21. Auth::context()

Una de las APIs más importantes.
$context = Auth::context();
Retorna:
AuthenticationContext
definido por el sistema de assurance/context.

## 22. Example

$context = Auth::context();

$context->identity();
$context->assurance();
$context->authenticatedAt();
$context->methods();
$context->tenant();
$context->realm();
$context->deviceTrust();

## 23. Context Immutability

El desarrollador no podrá hacer:
Auth::context()->setAssurance(HIGH);

## 24. Context Is Evidence-Derived

Authentication Context será producido únicamente por el Authentication Core.

## 25. Auth::assurance()

Convenience:
$level = Auth::assurance();
podría devolver:
AuthenticationAssuranceLevel::Strong

## 26. Assurance Convenience Checks

Auth::hasAssurance(AuthenticationAssuranceLevel::Strong);

## 27. But

No deberá recomendarse:
if (Auth::hasAssurance(HIGH)) {
    deleteTenant();
}
porque falta Authorization.

## 28. Correct Model

Gate::authorize('tenant.delete', $tenant);

Auth::require(
    AuthenticationRequirement::highAssurance()
);
o una integración declarativa.

## 29. Auth::attempt()

Experiencia estilo Laravel:
$result = Auth::attempt([
    'email' => $email,
    'password' => $password,
]);
Pero el resultado debería ser más expresivo que un simple boolean.

## 30. Authentication Attempt Result

interface AuthenticationAttemptResult
{
}
Implementaciones:
Authenticated
ChallengeRequired
ReauthenticationRequired
UpgradeRequired
RecoveryRequired
Denied
Locked
Suspended
RateLimited
Failed

## 31. Why Not Boolean Only?

Porque:
true / false
no puede expresar:
password valid but MFA required
password valid but passkey step-up required
identity valid but account locked
migration upgrade required
device approval required
rate limited
recovery required

## 32. Ergonomic Shortcut

Puede existir:
Auth::attemptCredentials($credentials)->successful();

## 33. Simple Login Controller

$result = Auth::attempt([
    'email' => $request->email,
    'password' => $request->password,
]);

return match (true) {
    $result->authenticated() =>
        redirect()->intended('/dashboard'),

    $result->challengeRequired() =>
        Auth::challenges()->respond($result),

    default =>
        back()->withErrors([
            'email' => 'Unable to authenticate.',
        ]),
};

## 34. Higher-Level Flow API

VoltStack puede simplificar todavía más:
return Auth::flow()
    ->attempt($request)
    ->respond();

## 35. Authentication Flow Facade

Auth::flow();
retorna:
AuthenticationFlowApplicationInterface

## 36. Responsibility

Traducir resultados internos a:
HTTP Response
SPA Response
JSON Response
Redirect
Challenge Page
sin mezclar Presentation con Security Decision.

## 37. Auth::login()

Debe manejarse cuidadosamente.
Laravel-like:
Auth::login($user);
es muy conveniente.
Pero VoltStack deberá darle semántica explícita.

## 38. Direct Login Is Dangerous

Esto:
Auth::login($user);
no prueba identidad.
Por tanto no debería crear arbitrariamente un Authentication Context fuerte.

## 39. Trusted Login

Debe existir un concepto:
Auth::establish(
    principal: $user,
    evidence: $trustedEvidence
);
para integraciones legítimas.

## 40. Auth::login() Policy

Puede ser alias restringido para:
trusted application-established authentication
y requerir un AuthenticationEstablishmentAuthority.

## 41. Recommended

Auth::loginUsingEvidence(
    $principal,
    $evidence
);

## 42. Manual Authentication Evidence

Solo componentes autorizados podrán crear evidence confiable.

## 43. Never

Auth::login($user, assurance: 'privileged');

## 44. Authentication Establishment Authority

interface AuthenticationEstablishmentAuthorityInterface
{
    public function establish(
        PrincipalInterface $principal,
        AuthenticationEvidenceSet $evidence,
        AuthenticationEstablishmentContext $context,
    ): AuthenticationEstablishmentResult;
}

## 45. Auth::once()

Puede existir para Authentication request-only:
Auth::once($credentials);
pero sin session persistence.

## 46. Use Cases

internal operation
one-request compatibility
CLI
special middleware
testing

## 47. Auth::logout()

Auth::logout();
deberá:
invalidate current session
invalidate request auth context
rotate/clear session state
emit events
audit as required
clear relevant cookies

## 48. Logout != Cookie Deletion

El backend authoritative state deberá actualizarse.

## 49. Global Logout

Auth::logoutEverywhere();
podrá:
revoke sessions
revoke remember-me
increment relevant security epoch
según policy.

## 50. Logout Other Sessions

Auth::logoutOtherSessions();
deberá pasar por:
Authorization
Authentication Requirement
Security Policy
cuando corresponda.

## 51. Session API

Auth::sessions();
retorna un application service.

## 52. Example

$sessions = Auth::sessions()->all();

Auth::sessions()->revoke($sessionId);

Auth::sessions()->revokeOthers();

Auth::sessions()->revokeAll();

## 53. Session ID Safety

La API nunca expondrá el bearer session token.
Usará:
Public Session Management ID
del documento 35.

## 54. Authentication Methods API

Auth::methods();

## 55. Example

Auth::methods()->all();

Auth::methods()->password();

Auth::methods()->passkeys();

Auth::methods()->mfa();

Auth::methods()->federated();

Auth::methods()->remove($methodId);

## 56. Removal Is Security-Sensitive

Auth::methods()->remove($id);
no elimina directamente un DB row.
Pasa por el command layer del documento 34.

## 57. Passkey DX

Auth::passkeys()->beginRegistration();

Auth::passkeys()->completeRegistration($response);

Auth::passkeys()->all();

Auth::passkeys()->remove($id);

## 58. MFA DX

Auth::mfa()->methods();

Auth::mfa()->enroll('totp');

Auth::mfa()->disable($methodId);

## 59. Security Center DX

Auth::security();

## 60. Example

$snapshot = Auth::security()->snapshot();

$snapshot->sessions();
$snapshot->devices();
$snapshot->methods();
$snapshot->alerts();
$snapshot->recommendations();

## 61. Security Commands

Auth::security()->reportUnknownSession($session);

Auth::security()->reportSuspiciousLogin($activity);

Auth::security()->secureAccount();

Auth::security()->beginReview();

## 62. Risk API

Risk no debería exponerse como:
Auth::risk()->setScore(0);

## 63. Read-Only Application Surface

Puede ofrecer:
Auth::risk()->current();
cuando policy permita al application layer conocerlo.

## 64. Sensitive Internal Risk Details

No necesariamente serán visibles a controllers.

## 65. Requirement API

Uno de los componentes fundamentales.
Auth::require(
    AuthenticationRequirement::fresh(
        maximumAge: new DateInterval('PT10M')
    )
);

## 66. Example

Auth::require(
    AuthenticationRequirement::highAssurance()
);

## 67. Fluent Requirement Builder

$requirement = AuthRequirement::make()
    ->minimumAssurance(AuthenticationAssuranceLevel::High)
    ->freshWithin(minutes: 5)
    ->requireMfa()
    ->requirePhishingResistance()
    ->requireUserVerification();

## 68. Then

$result = Auth::require($requirement);

## 69. Result

Satisfied
ChallengeRequired
ReauthenticationRequired
StepUpRequired
Denied

## 70. Automatic Challenge

Puede existir:
return Auth::require($requirement)
    ->respond();

## 71. But

El framework deberá evitar auto-replay inseguro de operaciones destructivas.
Documento 38/39.

## 72. Sensitive Operation DX

Preferible:
Auth::forOperation('tenant.delete')
    ->require();

## 73. Operation Registry Integration

Auth::forOperation('auth.mfa.disable');
resuelve requirements del Policy Engine.

## 74. Example

Auth::forOperation('billing.payment_method.change')
    ->require();

## 75. Authorization Still Separate

Gate::authorize('update-payment-method', $account);

Auth::forOperation('billing.payment_method.change')
    ->require();

## 76. Combined Framework Integration

VoltStack podrá ofrecer middleware/attributes que ejecuten ambas capas correctamente.

## 77. Attribute-Based Authentication

\# [RequiresAuthentication]
public function dashboard()
{
}

## 78. Assurance Attribute

\# [RequiresAuthentication(
    assurance: AuthenticationAssuranceLevel::Strong
)]
public function securitySettings()
{
}

## 79. Fresh Authentication Attribute

\# [RequiresFreshAuthentication(
    maximumAge: 'PT10M'
)]
public function changePassword()
{
}

## 80. MFA Attribute

\# [RequiresMfa]
public function manageApiKeys()
{
}

## 81. Phishing-Resistant Attribute

\# [RequiresPhishingResistantAuthentication]
public function privilegedOperation()
{
}

## 82. Operation Attribute

Más recomendable para enterprise:
\# [SensitiveOperation('tenant.delete')]
public function destroy(Tenant $tenant)
{
}

## 83. Why Better?

La policy puede evolucionar sin modificar controller.
Hoy:
fresh password
Mañana:
passkey + trusted device

## 84. Declarative Security

Controller
   ↓
Metadata
   ↓
Compiled Authentication Policy
   ↓
Runtime Requirement

## 85. Route Integration

Route::get('/dashboard', DashboardController::class)
    ->auth();

## 86. Route Assurance

Route::get('/security', SecurityController::class)
    ->auth()
    ->assurance('strong');

## 87. Sensitive Operation

Route::delete('/tenant/{tenant}', DeleteTenantController::class)
    ->sensitive('tenant.delete');

## 88. Middleware API

Aliases conceptuales:
auth
guest
auth.session
auth.fresh
auth.mfa
auth.assurance
auth.sensitive
auth.realm
auth.tenant

## 89. Minimal Middleware

->middleware('auth');

## 90. Named Auth Context

Puede existir:
->middleware('auth:admin');
pero VoltStack debería modelarlo como realm/security context, no solo "guard string".

## 91. Realm DX

->realm('admin');
o:
\# [AuthenticationRealm('admin')]

## 92. Guard Compatibility

Para facilitar migración desde Laravel podría soportarse:
Auth::guard('web');
pero internamente "guard" no debe limitar la arquitectura.

## 93. Preferred VoltStack Concept

Authentication Context
+
Realm
+
Authenticator Resolver
+
Session Strategy

## 94. Compatibility API

Auth::guard('web');
puede existir como façade de compatibilidad.

## 95. Native API

Podría preferirse:
Auth::realm('user');

## 96. Multi-Tenant DX

Auth::tenant();
retorna el tenant del Authentication Context.

## 97. Never Mutable

No:
Auth::tenant($tenant);
para cambiar arbitrariamente security scope.

## 98. Tenant Context Establishment

Debe pasar por Tenant Context Resolver.

## 99. Cross-Tenant Operations

Auth::withinTenant($tenant, function () {
    // ...
});
sería peligroso si cambia Authentication authority.

## 100. Better

Diferenciar:
Application Tenant Context
Authentication Tenant Binding

## 101. Tenant Binding Immutable

Durante una Authentication Transaction, tenant binding no se cambia silenciosamente.

## 102. SPA Integration

VoltStack tiene runtime SPA propio.
Authentication deberá integrarse nativamente.

## 103. SPA Authentication State

Frontend puede recibir:
{
  "authenticated": true,
  "principal": {
    "id": "..."
  },
  "security": {
    "assurance": "strong",
    "reauthentication_required": false
  }
}

## 104. Never Send

session token
password hash
TOTP secret
recovery codes
internal risk rules
security incident evidence

## 105. Frontend Auth Manifest

Podría existir:
Authentication Frontend Manifest
con información segura.

## 106. Example

{
  "authenticated": true,
  "realm": "user",
  "capabilities": {
    "can_reauthenticate": true,
    "passkeys_available": true
  }
}

## 107. Capabilities != Authorization

No enviar:
{
  "is_admin": true
}
como sustituto del Authorization backend.

## 108. SPA Challenge Protocol

SPA Action
   ↓
Authentication Requirement Not Met
   ↓
409/428 Authentication Challenge
   ↓
Frontend opens challenge UI
   ↓
Challenge completed
   ↓
Operation continuation

## 109. Dedicated Status

VoltStack puede definir protocolo propio.
Ejemplo conceptual:
428 Authentication Requirement
aunque el transporte exacto deberá definirse cuidadosamente.

## 110. Challenge Payload

{
  "type": "authentication_challenge",
  "transaction": "...",
  "challenge": {
    "type": "passkey"
  }
}

## 111. Transaction Reference

Debe ser opaque, short-lived y protegido según doc39.

## 112. SPA Replay

No repetir automáticamente:
DELETE /tenant
sin safe intent protocol.

## 113. Hydration Integration

Authentication context puede formar parte de la initial hydration segura.

## 114. Example

$hydration->share(
    'auth',
    AuthFrontendProjection::fromContext(Auth::context())
);

## 115. Authentication Projection

Debe ser explícita.
No serializar directamente:
AuthenticationContext

## 116. Why?

El contexto interno puede contener metadata que frontend no necesita.

## 117. API Authentication

Para APIs:
Auth::principal();
Auth::context();
deberán funcionar igual independientemente de si authentication vino de:
Bearer token
mTLS
signed request
OAuth
machine credential
session

## 118. Transport Independence

Controller no debería necesitar:
if ($request->hasBearerToken()) { ... }
para conocer principal.

## 119. Machine Identity DX

$principal = Auth::principal();

if ($principal instanceof MachinePrincipal) {
    // ...
}

## 120. Better

Usar typed principal capabilities.

## 121. CLI Integration

Authentication también deberá funcionar fuera de HTTP.

## 122. CLI Context

CLI Command
   ↓
Authentication Entry Point
   ↓
Interactive / Machine Authentication
   ↓
Authentication Context

## 123. Example

voltstack auth:login
puede establecer una CLI authentication session local segura.

## 124. Privileged CLI

voltstack tenant:delete acme
puede requerir:
Authorization
+
Fresh Authentication
+
MFA/Passkey

## 125. CLI Helper

Internamente:
Auth::forOperation('tenant.delete')
    ->require();

## 126. Queue Integration

Background jobs no deberán llamar:
Auth::user()
esperando que el usuario original siga presente.

## 127. Job Security Context

Deberá existir explícitamente.
final readonly class JobAuthenticationContext
{
    // machine actor
    // optional original initiator
    // delegation
}

## 128. Actor vs Initiator

Human User
    ↓ queues job
Worker Machine Identity
    ↓ executes job
Entonces:
Initiator = Human
Actor = Worker

## 129. No Serialized Session

Nunca:
dispatch(new Job(
    sessionToken: Auth::session()->token()
));

## 130. Delegation API

Puede existir:
$delegation = Auth::delegation()
    ->forOperation('report.generate')
    ->forResource($report)
    ->expiresIn(minutes: 10)
    ->issue();

## 131. Background Job

GenerateReport::dispatch(
    delegation: $delegation->reference()
);

## 132. Delegation Downscoped

Nunca hereda toda la autoridad del usuario.

## 133. WebSocket Integration

Persistent connections necesitan Authentication Context explícito.

## 134. Connection Authentication

Connection Establishment
      ↓
Authenticate
      ↓
Connection Security Context
      ↓
Periodic/Re-event Validation

## 135. Session Revocation

Una conexión WebSocket no debe permanecer autenticada eternamente después de session revocation.

## 136. Authentication Epoch

Connection puede almacenar:
identity security epoch
session version
y revalidar.

## 137. Configuration System

Archivo conceptual:
config/auth.php

## 138. Configuration Goals

Debe ser:
secure by default
typed
validated
cacheable
environment-aware
tenant-aware where appropriate
secret-free

## 139. Example Top-Level Configuration

return [

    'defaults' => [
        'realm' => 'user',
        'session' => 'web',
    ],

    'realms' => [
        // ...
    ],

    'authenticators' => [
        // ...
    ],

    'sessions' => [
        // ...
    ],

    'passwords' => [
        // ...
    ],

    'mfa' => [
        // ...
    ],

    'passkeys' => [
        // ...
    ],

    'federation' => [
        // ...
    ],

    'recovery' => [
        // ...
    ],

    'policy' => [
        // ...
    ],

    'security' => [
        // ...
    ],
];

## 140. Realm Configuration

'realms' => [

    'user' => [
        'principal' => App\Models\User::class,
        'session' => 'web',
        'authenticators' => [
            'password',
            'passkey',
            'oidc',
        ],
    ],

    'admin' => [
        'principal' => App\Models\Admin::class,
        'session' => 'admin',
        'authenticators' => [
            'password',
            'passkey',
        ],
        'policy' => 'admin',
    ],
],

## 141. Admin Realm

Podría requerir:
'policy' => [
    'minimum_assurance' => 'strong',
    'mfa' => true,
    'phishing_resistant' => true,
    'remember_me' => false,
],

## 142. Configuration Is Not Runtime State

Nunca:
config('auth.current_user')

## 143. Environment Variables

Pueden referenciar:
provider URLs
client IDs
feature flags
pero secrets deberían resolverse mediante Secret Provider.

## 144. Secret References

'client_secret' => secret('oidc.google.client_secret'),
conceptualmente.

## 145. Configuration Validation

Durante bootstrap:
Load
 ↓
Normalize
 ↓
Validate
 ↓
Resolve references
 ↓
Compile
 ↓
Freeze

## 146. Invalid Configuration

Debe fallar temprano.

## 147. Examples

unknown authenticator
duplicate realm
missing session strategy
admin realm allows remember-me against platform floor
passkey configured without RP ID
OIDC provider missing issuer
policy contradiction

## 148. Configuration Compiler

interface AuthenticationConfigurationCompilerInterface
{
    public function compile(
        AuthenticationRawConfiguration $configuration
    ): CompiledAuthenticationConfiguration;
}

## 149. Compiled Configuration

final readonly class CompiledAuthenticationConfiguration
{
    public function __construct(
        public CompiledRealmRegistry $realms,
        public CompiledAuthenticatorRegistry $authenticators,
        public CompiledSessionRegistry $sessions,
        public CompiledAuthenticationPolicySet $policies,
        public AuthenticationConfigurationVersion $version,
    ) {}
}

## 150. Compiled Config Is Immutable

Especialmente importante para FrankenPHP.

## 151. Config Cache

VoltStack podrá soportar:
voltstack auth:config:compile
o integrarlo en:
voltstack optimize

## 152. Config Cache Must Be Secret-Free

No serializar secrets resueltos permanentemente.

## 153. Bootstrap System

Authentication deberá registrarse en el framework bootstrap.

## 154. Bootstrap Phases

Kernel Bootstrap
      ↓
Auth Package Discovery
      ↓
Auth Configuration
      ↓
Contract Registration
      ↓
Provider Registration
      ↓
Static Registry Compilation
      ↓
Runtime Factory Registration
      ↓
Middleware Registration
      ↓
Route Metadata Registration
      ↓
Worker Reset Hooks
      ↓
READY

## 155. Authentication Service Provider

final class AuthenticationServiceProvider
    extends ServiceProvider
{
    public function register(): void
    {
        // contracts
        // factories
        // managers
    }

    public function boot(): void
    {
        // middleware
        // attributes
        // routes
        // reset hooks
    }
}

## 156. Register vs Boot

register():
container bindings
factories
registries
boot():
runtime integration
middleware aliases
routes
metadata
listeners

## 157. Service Lifetime Classification

Cada service deberá declarar lifetime.
Singleton Immutable
Request Scoped
Transaction Scoped
Transient
Worker Scoped

## 158. Singleton Candidates

Solo componentes inmutables:
Compiled Configuration
Compiled Policy Registry
Authenticator Definitions
Algorithm Registry
Metadata Registry

## 159. Request-Scoped

AuthenticationContextAccessor
CurrentAuthenticationTransaction
Request Security Context
Current Tenant Binding
Current Realm Context

## 160. Transient

Challenge processors
Credential verification operations
Proof validators
según implementación.

## 161. Never Singleton

Current User
Current Session
Current Authentication Context
Current Challenge
Current Transaction

## 162. Container Contracts

Ejemplo:
$container->scoped(
    AuthenticationContextAccessorInterface::class,
    AuthenticationContextAccessor::class,
);

## 163. Request Scope

Al inicio:
AuthenticationContext = Anonymous

## 164. Authentication Middleware

Resuelve Authentication Context.

## 165. Request End

clear context
clear transaction-local secrets
reset scoped services

## 166. FrankenPHP Worker Reset

VoltStack deberá registrar:
AuthenticationWorkerResetter

## 167. Reset Contract

interface AuthenticationRuntimeResetterInterface
{
    public function reset(): void;
}

## 168. Reset Targets

context accessor
temporary credential buffers
challenge context
tenant binding
realm binding
request-local memoization
temporary provider state

## 169. Facade Resolution

Auth::user()
   ↓
Facade Proxy
   ↓
Container
   ↓
Request-Scoped AuthenticationContextAccessor
   ↓
Current AuthenticationContext

## 170. Helper Functions

VoltStack puede proporcionar helpers opcionales.

## 171. auth()

auth();
retorna:
AuthenticationApplicationInterface

## 172. auth()->user()

$user = auth()->user();

## 173. auth()->check()

if (auth()->check()) {
}

## 174. Helper Design

Helpers serán thin wrappers.
Nunca implementarán lógica.

## 175. No Global Mutable State

El helper siempre resuelve desde container/runtime context.

## 176. auth_user()

No es necesario si:
auth()->user()
ya es suficientemente claro.
Evitar proliferación de helpers.

## 177. Recommended Public Surface

Mantener:
Auth facade
auth() helper
Contracts
Attributes
Middleware
como superficie principal.

## 178. Contract-First DX

Aplicaciones desacopladas pueden usar:
final class DashboardController
{
    public function __construct(
        private AuthenticationContextAccessorInterface $auth,
    ) {}
}

## 179. Example

public function __invoke()
{
    return view('dashboard', [
        'user' => $this->auth->principal(),
    ]);
}

## 180. Facade vs DI

VoltStack soportará ambas.
Facade
→ convenience

Dependency Injection
→ explicit architecture/testability

## 181. Neither Is Less Secure

Ambas terminan en el mismo Core.

## 182. Facade Must Be Mockable Carefully

Testing facade no deberá cambiar production security semantics.

## 183. Application Authentication Interface

interface AuthenticationApplicationInterface
{
    public function check(): bool;

    public function principal(): ?PrincipalInterface;

    public function context(): AuthenticationContext;

    public function attempt(
        AuthenticationAttemptInput $input
    ): AuthenticationAttemptResult;

    public function logout(): AuthenticationLogoutResult;
}

## 184. Credentials Array

La API simple puede aceptar:
Auth::attempt([
    'email' => $email,
    'password' => $password,
]);
pero internamente deberá convertirse a typed input.

## 185. Typed Input

final readonly class PasswordAuthenticationAttempt
{
    public function __construct(
        public AuthenticationIdentifier $identifier,
        public SensitivePassword $password,
        public AuthenticationAttemptOptions $options,
    ) {}
}

## 186. SensitivePassword

Debe intentar limitar accidental logging/serialization.

## 187. Example

final class SensitivePassword
{
    public function __toString(): string
    {
        throw new SensitiveValueExposureException();
    }
}
conceptualmente.

## 188. Credential Bags

Evitar pasar arrays sin control por todo el sistema.

## 189. Boundary Normalization

Controller Array
      ↓
Application Input Factory
      ↓
Typed Sensitive Objects
      ↓
Core

## 190. Request Integration

Puede existir:
Auth::attemptFrom($request);
pero debe requerir un named input mapping.

## 191. Avoid Magic Field Assumptions

No asumir siempre:
email
password

## 192. Authentication Form Definition

Auth::flow('login')
    ->from($request);
puede resolver configuración.

## 193. Login Identifier Types

email
username
phone
employee ID
tenant username
custom identifier

## 194. Identifier Resolver

El Core ya deberá manejar esta abstracción.
DX no debe hardcodear email.

## 195. Redirect Integration

Después de login:
return Auth::redirectIntended('/dashboard');

## 196. Intended Destination

Debe usar secure continuation del doc39.

## 197. No Arbitrary Redirect

Nunca confiar directamente en:
?redirect=https://evil.example

## 198. Guest Middleware

guest
puede redirigir usuarios autenticados.

## 199. But

Realm-aware:
authenticated in user realm
no necesariamente significa:
authenticated in admin realm

## 200. Realm-Aware Guest

\# [GuestOnly(realm: 'admin')]

## 201. Exception Integration

Authentication exceptions deberán convertirse en respuestas apropiadas.

## 202. Exception Boundary

Controllers no deberían necesitar manejar:
ReplayStoreUnavailableException
KeyResolutionException
AuthenticatorInfrastructureException
directamente.

## 203. Error Normalization

Internal Failure
      ↓
Authentication Failure Taxonomy
      ↓
Application Result
      ↓
Safe HTTP/SPA/API Response

## 204. Enumeration Protection

Public errors:
Invalid credentials.
en lugar de:
User exists but password incorrect.
cuando policy lo requiera.

## 205. Internal Explainability

Audit/security operators sí pueden recibir:
IDENTITY_SUSPENDED
PASSWORD_INVALID
CREDENTIAL_REVOKED
según permisos.

## 206. Response Factory

interface AuthenticationResponseFactoryInterface
{
    public function from(
        AuthenticationApplicationResult $result,
        AuthenticationTransportContext $transport,
    ): ResponseInterface;
}

## 207. Transport Types

HTML
SPA
JSON API
CLI
WebSocket
Mobile

## 208. Localization

Authentication messages deben usar translation system.

## 209. But

Security reason codes no deben depender de strings traducidos.

## 210. Example

Internal:
AUTH_CREDENTIAL_INVALID
UI:
"Las credenciales proporcionadas no son válidas."

## 211. Event Integration

Developer podrá escuchar:
\# [Listener]
public function onLogin(AuthenticationSucceeded $event): void
{
}

## 212. But Event Listeners Cannot Rewrite Authentication

Un listener no debería:
$event->setAuthenticated(true);

## 213. Events Are Facts

Preferir eventos inmutables.

## 214. Hooks

Para extensibilidad que sí interviene en decisiones se usarán contratos específicos:
Policy Contributor
Authenticator
Risk Contributor
Eligibility Checker
Challenge Provider
no event listeners arbitrarios.

## 215. Application Hooks

Puede haber hooks seguros:
after login
after logout
after credential enrollment

## 216. No Secret Event Payloads

Eventos no deben transportar plaintext credentials salvo un contrato síncrono extremadamente controlado que explícitamente lo requiera.
Preferiblemente nunca.

## 217. Model Integration

Un modelo application-facing podría implementar:
interface Authenticatable
{
    public function authenticationIdentityId(): IdentityId;
}

## 218. Identity != ORM Model

VoltStack no deberá hacer que Authentication dependa completamente del ORM.

## 219. Identity Provider

interface IdentityProviderInterface
{
    public function find(
        IdentityReference $reference
    ): ?IdentityInterface;
}

## 220. Eloquent-Like Integration

Puede existir adapter para el ORM de VoltStack.

## 221. Symfony-Style Provider

También puede existir provider explícito.

## 222. Flexible Architecture

Database Model
LDAP
External Identity Store
API
In-Memory Testing
pueden implementar Identity Provider.

## 223. Authentication Model Trait

Puede existir convenience trait:
use AuthenticatableIdentity;

## 224. But Trait Is Optional

Core debe funcionar sin él.

## 225. Password Access

Modelo no debería exponer:
$user->password
como requirement del Core.

## 226. Credential Repository

Password credential pertenece al Authentication Credential subsystem.

## 227. Better Separation

Identity
   │
   ├── Profile
   │
   └── Authentication Methods
          ↓
      Credential Store

## 228. Legacy-Friendly Adapter

Para Laravel-style tables:
users.password
podrá existir adapter.

## 229. Native VoltStack

Preferirá separar credenciales del application model cuando arquitectura lo permita.

## 230. Registration Integration

Authentication no necesariamente posee todo el proceso de user registration.

## 231. Registration Flow

Application Registration
       ↓
Identity Provisioning
       ↓
Authentication Method Enrollment
       ↓
Verification
       ↓
Activation

## 232. DX

Puede existir:
Auth::registration()
    ->begin($data);

## 233. But

Registration es una application workflow construida sobre Identity Lifecycle + Credential Enrollment.

## 234. Password Reset DX

Auth::recovery()
    ->request($identifier);

## 235. Complete Recovery

Auth::recovery()
    ->complete($transaction, $evidence);

## 236. Do Not Call It Only Password Reset

Porque passwordless users pueden recuperar identidad sin password.

## 237. Federation DX

return Auth::federation('google')->redirect();

## 238. Callback

return Auth::federation('google')
    ->callback($request);

## 239. Flow Purpose

La API deberá saber si se trata de:
Login
Link
Reauthenticate
Recovery

## 240. Explicit Purpose

Preferible:
Auth::federation('google')
    ->forLogin()
    ->redirect();

## 241. Linking

Auth::federation('google')
    ->forLinking()
    ->redirect();

## 242. Prevent Flow Confusion

Un callback iniciado para login no podrá reutilizarse para linking.
Documento 39.

## 243. Passkey Browser Integration

Backend DX:
$options = Auth::passkeys()
    ->authenticationOptions();
Frontend obtiene safe options.

## 244. Verification

$result = Auth::passkeys()
    ->authenticate($credentialResponse);

## 245. Frontend Package

VoltStack podría proporcionar:
@voltstack/auth
como pequeño runtime JS integrado al Frontend Runtime.

## 246. Responsibilities

passkey browser API
challenge UI protocol
SPA continuation
security notification UI hooks
auth state refresh

## 247. JS Runtime Cannot Decide Security

Frontend nunca determina:
assurance
authentication success
authorization
challenge satisfaction

## 248. Server Is Authority

Siempre.

## 249. Configuration DSL

Además de arrays, podría existir DSL programático.

## 250. Example

Authentication::configure()
    ->realm('user')
        ->session('web')
        ->password()
        ->passkeys()
        ->oidc('google')
    ->realm('admin')
        ->session('admin')
        ->password()
        ->passkeys()
        ->requireMfa()
        ->requirePhishingResistance();

## 251. DSL Compiles to Same Model

PHP Config
Attributes
DSL
Package Contributions
        ↓
Normalized Authentication Configuration
        ↓
Compiler

## 252. One Semantic Model

Evitar que cada configuración tenga comportamiento distinto.

## 253. Package Auto-Discovery

Quantum packages podrán registrar Authentication extensions.

## 254. Example Package

Quantum/Auth/WebAuthn
podría registrar:
PasskeyAuthenticator
PasskeyChallengeProvider
PasskeyMethodDefinition

## 255. Plugin Registration

Debe pasar por Authentication Registry.

## 256. Package Cannot Override Security Floor

Incluso si auto-discovered.

## 257. Bootstrap Ordering

Authentication puede depender de:
Config
Container
Events
Cache
Crypto
Database
HTTP
Session
Routing
Telemetry
Queue

## 258. Dependency Graph

Config
  ↓
Container
  ↓
Crypto / Storage
  ↓
Authentication Core
  ↓
Authentication Runtime
  ↓
HTTP / Routing / SPA Integration

## 259. Avoid Circular Dependency

Ejemplo peligroso:
Routing requires Auth
Auth bootstrap requires Routing

## 260. Solution

Separar:
Auth Core Registration
Auth Runtime Registration
Auth Routing Integration

## 261. Bootstrap Phases Detailed

Phase 1 — Definitions
Carga contracts y immutable definitions.
Phase 2 — Configuration
Normaliza auth.php.
Phase 3 — Compilation
Compila policies, realms, authenticators.
Phase 4 — Runtime Bindings
Registra request-scoped services.
Phase 5 — Transport Integration
HTTP, SPA, CLI.
Phase 6 — Operational Integration
Telemetry, queues, reset hooks.

## 262. Application Boot Hooks

Aplicaciones pueden extender:
public function bootAuthentication(
    AuthenticationRegistry $auth
): void {
}

## 263. But

No debería ser necesario para uso estándar.

## 264. Zero-Configuration Goal

Aplicación estándar:
install VoltStack
create User identity model
configure database
run migrations
y disponer de Authentication básica segura.

## 265. Secure Defaults

Por ejemplo:
Argon2id/password_hash policy
Secure cookies
HttpOnly
SameSite
CSRF
session rotation
rate limiting
generic login errors
recovery token single-use
modern crypto

## 266. No "Quick Start Insecure Mode"

VoltStack no debería enseñar ejemplos inseguros para luego decir:
"don't do this in production"

## 267. Development Environment

Puede simplificar providers, pero no desactivar invariantes críticas.

## 268. Example

Dev puede usar:
Mail notification → log transport
pero no:
disable CSRF
accept any password
disable nonce validation
como default.

## 269. Authentication Scaffolding

Puede existir:
voltstack make:auth

## 270. Scaffolding Options

web
spa
api
admin
passkeys
mfa
federation

## 271. Example

voltstack make:auth --spa --passkeys

## 272. Generated Artifacts

Podrían incluir:
Login Action
Logout Action
Recovery Routes
Passkey Enrollment Component
Security Center Component
Auth configuration
Tests

## 273. Scaffolding Philosophy

Generar application/UI layer.
No copiar Authentication Core al proyecto.

## 274. No Vendor Forking

Security fixes del framework deben actualizar Core sin regenerar application code.

## 275. Default Routes

VoltStack podría registrar opcionalmente:
/login
/logout
/auth/challenge
/auth/continue
/recovery
/security
/passkeys

## 276. But

Route registration deberá ser configurable.

## 277. Headless Mode

'ui' => false,
para APIs/SPAs custom.

## 278. Headless Authentication

Core sigue completo.
Solo no genera presentation routes/views.

## 279. Controller Integration

Controller parameter injection:
public function__invoke(
    AuthenticationContext $auth
) {
}

## 280. Parameter Resolver

Controller system de VoltStack puede resolver automáticamente el current immutable context.

## 281. Principal Injection

public function __invoke(
    #[CurrentPrincipal] User $user
) {
}

## 282. Missing Authentication

Resolver produce Authentication Required.
No:
null passed unexpectedly

## 283. Current Identity

\# [CurrentIdentity]
IdentityId $identity
podría ser soportado.

## 284. Security Context Injection

public function __invoke(
    AuthenticationContext $authentication,
    AuthorizationContext $authorization,
) {
}
manteniendo separación.

## 285. Action Integration

VoltStack Actions podrán declarar:
\# [SensitiveOperation('auth.password.change')]
final class ChangePasswordAction
{
}

## 286. Framework Compiler

Puede precompilar metadata de:
Routes
Controllers
Actions
Components
Commands

## 287. Compile-Time Security Metadata

Reduce reflexión runtime.

## 288. Component Integration

Componentes reactivos pueden usar:
\# [RequiresAuthentication]
final class SecurityCenterComponent
{
}

## 289. Component Actions

\# [SensitiveOperation('auth.mfa.disable')]
public function disableMfa(string $method)
{
}

## 290. Hydration Security

No confiar en serialized component state para Authentication Context.

## 291. Every Request Rebinds Context

Incluso SPA/reactive requests.

## 292. Long-Lived Component

No conserva:
"authenticated = true"
como autoridad.

## 293. State System Integration

Puede exponer:
$this->auth->user()
pero la fuente es runtime context.

## 294. Template Integration

VoltStack directives podrían ofrecer:
@auth
@guest

## 295. Example

@auth
    Welcome, {{ auth()->user()->name }}
@endauth

## 296. Realm

@auth(realm: 'admin')
    ...
@endauth

## 297. Assurance UI

Podría existir:
@authAssurance('strong')
pero solo para presentation.

## 298. Critical Rule

Template directives:
hide/show UI
No:
enforce backend security

## 299. Same for Frontend

Ocultar botón no sustituye Authorization/Authentication enforcement.

## 300. Testing DX

VoltStack deberá proporcionar una excelente Testing API.

## 301. actingAs()

$this->actingAs($user);
estilo Laravel.

## 302. But

Debe crear un Authentication Context explícito.

## 303. Default Test Assurance

No debería ser accidentalmente PRIVILEGED.

## 304. Example

$this->actingAs(
    $user,
    assurance: AuthenticationAssuranceLevel::Standard
);

## 305. MFA Test

$this->actingAs($user)
    ->withMfa()
    ->withAssurance(AuthenticationAssuranceLevel::Strong);

## 306. Fresh Authentication

$this->actingAs($user)
    ->authenticatedAgo(minutes: 2);

## 307. Expired Freshness

$this->actingAs($user)
    ->authenticatedAgo(hours: 4);

## 308. Device Trust

$this->actingAs($user)
    ->fromTrustedDevice();

## 309. Risk

$this->actingAs($user)
    ->withRisk(AuthenticationRiskLevel::High);

## 310. Suspended Identity

$this->actingAs($user)
    ->suspended();
pero deberá modelar lifecycle correctamente.

## 311. Security Incident

$this->actingAs($user)
    ->withProtectionState(
        IdentityProtectionState::Restricted
    );

## 312. Testing Challenges

$this->assertAuthenticationChallengeRequired();

$this->assertAuthenticationChallengeType('passkey');

## 313. Assert Authenticated

$this->assertAuthenticated();

$this->assertAuthenticatedAs($user);

$this->assertGuest();

## 314. Assert Assurance

$this->assertAuthenticationAssurance(
    AuthenticationAssuranceLevel::Strong
);

## 315. Assert Session Revoked

$this->assertSessionRevoked($sessionId);

## 316. Testing Does Not Weaken Production

Test helpers deberán usar dedicated test providers.
No insertar backdoors en Core.

## 317. Auth::fake()

Podría existir exclusivamente bajo Testing package.

## 318. Production Guard

El Testing authentication provider no podrá registrarse accidentalmente en production.

## 319. Environment Assertion

Bootstrap deberá fallar si:
TestAuthenticator enabled in production

## 320. Static Analysis

VoltStack podría proporcionar annotations/types para mejorar IDE.

## 321. Generic User Type

Conceptualmente:
/** @return User|null */
Auth::user();
puede inferirse desde realm configuration.

## 322. Typed Realm

Auth::realm('admin')->user();
podría inferir:
Admin|null
mediante tooling.

## 323. IDE Metadata

Compiler puede generar:
storage/framework/auth-metadata.php
para tooling.

## 324. No Security Dependency on IDE Metadata

Solo DX.

## 325. Deprecation Strategy

Public Auth APIs deberán tener estabilidad.

## 326. Semantic Versioning

Cambios en:
Facade
Contracts
Attributes
Config schema
Result types
requieren política de compatibilidad.

## 327. Deprecated API

Ejemplo:
Auth::guard('web');
podría mantenerse como compatibility API mientras native realm API madura.

## 328. Deprecation Warning

Nunca deberá incluir secretos.

## 329. Application Integration Contracts

Se recomienda dividir:
AuthenticationReaderInterface
AuthenticationCommandInterface
AuthenticationFlowInterface
AuthenticationSecurityInterface
en lugar de un God Object.

## 330. Facade Can Aggregate

Auth
puede ser cómoda, pero internamente delegará.

## 331. Internal Delegation

Auth
 ├── Reader
 ├── Login
 ├── Session
 ├── Methods
 ├── Recovery
 ├── Security
 ├── Challenge
 └── Delegation

## 332. Avoid God Service

No crear:
class AuthManager
{
    // 300 public methods
}

## 333. Facade Subsystems

Auth::sessions();

Auth::methods();

Auth::passkeys();

Auth::mfa();

Auth::federation();

Auth::recovery();

Auth::security();

Auth::delegation();

Auth::flow();

## 334. Read vs Command Separation

Auth::sessions()->all();
es query.
Auth::sessions()->revoke($id);
es security command.
Internamente deberán usar pipelines distintos.

## 335. Security Commands Revalidate

Aunque UI previamente haya consultado:
canRevoke = true
el command deberá revalidar.

## 336. TOCTOU Protection

Documento 32/39.

## 337. Facade Error Semantics

Convenience API no debería lanzar excepciones por outcomes normales.

## 338. Normal Outcome

Invalid credentials
MFA required
Reauthentication required
→ result object.

## 339. Exceptional Conditions

misconfiguration
corrupt security state
infrastructure failure
programmer contract violation
→ exceptions.

## 340. Result-Oriented API

Esto mejora:
HTTP
SPA
CLI
testing

## 341. Application Integration Boundary

El Core no deberá depender de:
Controller
Blade/View
SPA Component
ORM model
CLI terminal

## 342. Adapters Depend on Core

HTTP Adapter ─┐
SPA Adapter ──┤
CLI Adapter ──┼→ Application Auth API → Core
Queue Adapter ┤
Test Adapter ─┘

## 343. Clean Architecture Direction

Application Layer
      ↓
Auth Application Contracts
      ↓
Domain/Core

## 344. Framework Convenience

La facade pertenece al outer framework layer.

## 345. Authentication Core Remains Testable

Sin HTTP kernel.

## 346. Telemetry Integration

Developer API deberá emitir telemetry desde Core/orchestrator.
No exigir:
Log::info('login');
en cada controller.

## 347. Automatic Telemetry

Auth::attempt()
    ↓
Auth Core
    ↓
events / audit / metrics / tracing

## 348. Correlation

Authentication Transaction ID puede correlacionar:
HTTP request
challenge
session creation
audit
notification
sin exponer secrets.

## 349. Developer Debugging

En development puede existir:
voltstack auth:diagnose

## 350. Diagnostic Output

Default realm: user
Session strategy: web
Authenticators:
  password     READY
  passkey      READY
  google_oidc  READY

Policy:
  compiled     YES

Crypto:
  password hashing   READY
  session signing    READY
  state protection   READY

Runtime:
  FrankenPHP reset hooks READY

## 351. No Secrets in Diagnose

Nunca imprimir:
client secret
signing private key
TOTP secret
password pepper

## 352. Config Explain

voltstack auth:config:explain admin
podría mostrar:
Realm: admin
Session: admin
Minimum assurance: STRONG
MFA: required
Phishing resistance: required
Remember-me: disabled

Sources:
Platform floor
  +
Admin realm policy

## 353. Policy Explain

voltstack auth:policy:explain tenant.delete

## 354. Example

Operation: tenant.delete

Minimum Assurance: HIGH
Freshness: 5 minutes
MFA: required
Phishing Resistance: required
Interactive Authentication: required

Sources:
Platform Security Floor
Tenant Administration Policy
Sensitive Operation Registry

## 355. Excellent DX Principle

Security debe ser explicable, no mágica.

## 356. Authentication Debug Toolbar

VoltStack Developer Debug Toolbar podría mostrar en development:
Principal
Realm
Authentication method
Assurance
Authentication age
Session type
Device trust
Policy requirements

## 357. But Never

password
token
cookie value
OTP
nonce
PKCE verifier
private keys

## 358. Production Debugging

Debug toolbar deshabilitada.

## 359. Facade Security Invariants

AUTH-DX-FACADE-001
Facade no almacena current user estáticamente.
AUTH-DX-FACADE-002
Facade siempre delega al Authentication Core.
AUTH-DX-FACADE-003
Facade no puede elevar assurance.
AUTH-DX-FACADE-004
Facade no puede crear arbitrary Authentication Evidence.
AUTH-DX-FACADE-005
Facade no puede saltarse Authorization.

## 360. Configuration Invariants

AUTH-DX-CONFIG-001
Configuration se valida antes de runtime.
AUTH-DX-CONFIG-002
Compiled configuration es inmutable.
AUTH-DX-CONFIG-003
Compiled cache no contiene secrets.
AUTH-DX-CONFIG-004
Tenant policy no debilita platform floor.
AUTH-DX-CONFIG-005
Contradictory policies fallan explícitamente.

## 361. Bootstrap Invariants

AUTH-DX-BOOT-001
Current Authentication Context nunca es singleton.
AUTH-DX-BOOT-002
Request context se limpia al finalizar.
AUTH-DX-BOOT-003
Worker context se limpia entre requests/jobs.
AUTH-DX-BOOT-004
Testing authenticators no pueden activarse en production.
AUTH-DX-BOOT-005
Authentication Core puede bootstrapped sin presentation layer.

## 362. Application Integration Invariants

AUTH-DX-APP-001
Frontend state nunca es autoridad de Authentication.
AUTH-DX-APP-002
Template directives solo controlan presentation.
AUTH-DX-APP-003
Background jobs no transportan reusable human session tokens.
AUTH-DX-APP-004
SPA destructive operations no se auto-replay sin safe continuation.
AUTH-DX-APP-005
Machine identities no se convierten en fake users.
AUTH-DX-APP-006
Realm/tenant binding no se cambia mediante convenience helpers.

## 363. Testing Invariants

AUTH-DX-TEST-001
actingAs() crea context explícito.
AUTH-DX-TEST-002
Default test assurance no es privileged.
AUTH-DX-TEST-003
Test bypasses existen únicamente dentro de Testing package.
AUTH-DX-TEST-004
Production bootstrap rechaza test providers.

## 364. Anti-Pattern — Static User

Auth::$user = $user;

## 365. Anti-Pattern — Direct Session Mutation

session(['user_id' => $user->id]);
como Authentication.

## 366. Anti-Pattern — Role as Authentication

if ($user->isAdmin()) {
    Auth::setAssurance('high');
}

## 367. Anti-Pattern — Frontend Authority

if (window.auth.isAdmin) {
    deleteTenant();
}

## 368. Anti-Pattern — Auth Context in Job

dispatch(new Job(
    user: Auth::user(),
    session: Auth::session()
));

## 369. Anti-Pattern — Raw Credential Arrays Everywhere

$credentials['password']
pasando por múltiples layers.

## 370. Anti-Pattern — Direct Method Deletion

DB::table('auth_methods')->delete($id);

## 371. Anti-Pattern — Guard Explosion

web
api
admin
partner
tenant
mobile
worker
internal
cada uno implementando Authentication completamente diferente.

## 372. Better

Realms
Authenticators
Session Strategies
Policies
Principal Types
componibles.

## 373. Anti-Pattern — Config Secrets

'google_client_secret' => 'abc123'
en repository.

## 374. Anti-Pattern — Boolean Attempt Everywhere

if (!Auth::attempt(...)) {
    // impossible to distinguish challenge/recovery/rate limit
}

## 375. Anti-Pattern — Convenience Bypass

Auth::forceLogin($user);
disponible en production application code.

## 376. Anti-Pattern — User Model Owns Everything

class User
{
    public $password;
    public $totp_secret;
    public $recovery_codes;
    public $sessions;
    public $oauth_tokens;
}

## 377. Recommended Separation

User/Profile Domain
       │
       ▼
Identity Reference
       │
       ▼
Authentication Domain
 ├── Methods
 ├── Credentials
 ├── Sessions
 ├── Devices
 └── Security State

## 378. Suggested Namespaces

VoltStack\Quantum\Auth\Application
VoltStack\Quantum\Auth\Configuration
VoltStack\Quantum\Auth\Bootstrap
VoltStack\Quantum\Auth\Runtime
VoltStack\Quantum\Auth\Http
VoltStack\Quantum\Auth\Spa
VoltStack\Quantum\Auth\Console
VoltStack\Quantum\Auth\Attributes
VoltStack\Quantum\Auth\Middleware
VoltStack\Quantum\Auth\Testing
VoltStack\Facades

## 379. Suggested Directory Structure

src/
├── Facades/
│   └── Auth.php
│
└── Quantum/
    └── Auth/
        ├── Application/
        │   ├── Contracts/
        │   │   ├── AuthenticationApplicationInterface.php
        │   │   ├── AuthenticationReaderInterface.php
        │   │   ├── AuthenticationCommandInterface.php
        │   │   ├── AuthenticationFlowInterface.php
        │   │   └── AuthenticationSecurityInterface.php
        │   │
        │   ├── AuthenticationApplication.php
        │   ├── AuthenticationReader.php
        │   ├── AuthenticationCommandBus.php
        │   └── AuthenticationFlowApplication.php
        │
        ├── Facades/
        │   └── Auth.php
        │
        ├── Helpers/
        │   └── auth.php
        │
        ├── Configuration/
        │   ├── AuthenticationConfiguration.php
        │   ├── AuthenticationConfigurationLoader.php
        │   ├── AuthenticationConfigurationValidator.php
        │   ├── AuthenticationConfigurationCompiler.php
        │   └── CompiledAuthenticationConfiguration.php
        │
        ├── Bootstrap/
        │   ├── AuthenticationServiceProvider.php
        │   ├── AuthenticationBootstrapper.php
        │   ├── AuthenticationRuntimeBootstrapper.php
        │   └── AuthenticationWorkerResetter.php
        │
        ├── Runtime/
        │   ├── AuthenticationContextAccessor.php
        │   ├── AuthenticationRuntimeContext.php
        │   └── AuthenticationRuntimeResetter.php
        │
        ├── Attributes/
        │   ├── RequiresAuthentication.php
        │   ├── RequiresFreshAuthentication.php
        │   ├── RequiresMfa.php
        │   ├── RequiresPhishingResistantAuthentication.php
        │   ├── AuthenticationRealm.php
        │   ├── SensitiveOperation.php
        │   ├── CurrentPrincipal.php
        │   └── CurrentIdentity.php
        │
        ├── Middleware/
        │   ├── Authenticate.php
        │   ├── RedirectIfAuthenticated.php
        │   ├── RequireFreshAuthentication.php
        │   ├── RequireAssurance.php
        │   ├── RequireMfa.php
        │   └── RequireSensitiveOperationAuthentication.php
        │
        ├── Http/
        │   ├── AuthenticationResponseFactory.php
        │   ├── AuthenticationRequestFactory.php
        │   └── AuthenticationHttpExceptionMapper.php
        │
        ├── Spa/
        │   ├── AuthenticationFrontendProjection.php
        │   ├── AuthenticationSpaResponseFactory.php
        │   ├── AuthenticationChallengeResponse.php
        │   └── AuthenticationHydrationContributor.php
        │
        ├── Console/
        │   ├── AuthenticationConsoleManager.php
        │   ├── Commands/
        │   └── AuthenticationConsoleResponseFactory.php
        │
        ├── Queue/
        │   ├── JobAuthenticationContext.php
        │   ├── AuthenticationDelegationIssuer.php
        │   └── AuthenticationDelegationResolver.php
        │
        ├── Components/
        │   └── AuthenticationComponentIntegration.php
        │
        ├── Testing/
        │   ├── InteractsWithAuthentication.php
        │   ├── AuthenticationTestContextBuilder.php
        │   ├── FakeAuthenticator.php
        │   └── AuthenticationAssertions.php
        │
        └── Exceptions/
            ├── AuthenticationConfigurationException.php
            ├── AuthenticationRuntimeException.php
            └── SensitiveValueExposureException.php

## 380. Recommended Auth Surface

La facade inicial no debería intentar exponer cada subsystem.
Una primera API coherente:
Auth::check();
Auth::guest();

Auth::id();
Auth::user();
Auth::principal();
Auth::context();

Auth::attempt(...);
Auth::once(...);
Auth::logout();
Auth::logoutEverywhere();

Auth::assurance();
Auth::require(...);
Auth::forOperation(...);

Auth::sessions();
Auth::methods();
Auth::mfa();
Auth::passkeys();
Auth::federation();
Auth::recovery();
Auth::security();
Auth::delegation();
Auth::flow();

## 381. Layered Developer Experience

VoltStack ofrecerá tres niveles.
Nivel 1 — Simple
Auth::user();

Auth::attempt($credentials);

Auth::logout();
Nivel 2 — Security-aware
Auth::forOperation('tenant.delete')->require();

Auth::security()->snapshot();

Auth::passkeys()->beginRegistration();
Nivel 3 — Architectural
AuthenticationPolicyEngineInterface
AuthenticationFlowEngineInterface
AuthenticationContextAccessorInterface
AuthenticationEvidenceVerifierInterface

## 382. Why Three Levels?

Un desarrollador creando un blog no necesita interactuar directamente con:
AuthenticationTransactionSecurityState
Pero un paquete enterprise sí puede necesitarlo.

## 383. Progressive Disclosure

La complejidad debe aparecer cuando se necesita.
Simple Application
       ↓
Simple API

Advanced Application
       ↓
Advanced API

Framework Extension
       ↓
Core Contracts

## 384. Laravel Comparison

Laravel sobresale en DX:
Auth::user();

Auth::check();

Auth::attempt($credentials);

Auth::logout();

auth()->user();

$this->actingAs($user);
Esta simplicidad es una referencia importante para VoltStack.
VoltStack debe conservar esa sensación de productividad.
La diferencia estará en que debajo de esa superficie habrá conceptos formalizados para:
Assurance
Authentication Evidence
Step-Up
Reauthentication
Passkeys
Risk
Device Trust
Identity Lifecycle
Security Incidents
Machine Identities
Multi-Tenant Authentication
Distributed Runtime

## 385. Symfony Comparison

Symfony favorece una arquitectura más explícita basada en:
Security
Authenticators
User Providers
Password Hashers
Firewalls
Access Control
Events
Dependency Injection
Esto facilita construir sistemas complejos y desacoplados.
VoltStack adopta esa disciplina mediante:
Contracts
Typed Value Objects
Registries
Resolvers
Compilers
Immutable Contexts
Explicit Lifetimes
pero proporciona una capa de convenience API más integrada.

## 386. VoltStack Position

Laravel
   ↓
Excellent convenience

Symfony
   ↓
Excellent explicit architecture

VoltStack
   ↓
Convenience
+
Explicit Architecture
+
Assurance-Aware Authentication
+
Persistent Runtime Safety

## 387. FrankenPHP Differentiator

La facade parecerá tradicional:
Auth::user();
pero no dependerá de globals tradicionales.
Internamente:
Auth Facade
    ↓
Container
    ↓
Fiber/Request Scope
    ↓
Immutable Authentication Context
permitiendo:
FrankenPHP
long-lived workers
concurrent requests
async execution
sin contaminación entre usuarios.

## 388. Example Complete Controller

final class DeleteTenantController
{
    public function __invoke(Tenant $tenant): Response
    {
        Gate::authorize('delete', $tenant);

        $requirement = Auth::forOperation('tenant.delete')
            ->forResource($tenant)
            ->evaluate();

        if (!$requirement->satisfied()) {
            return $requirement->respond();
        }

        $tenant->delete();

        return redirect('/tenants');
    }
}

## 389. Better Declarative Version

\# [RequiresAuthentication]
\# [SensitiveOperation('tenant.delete')]
final class DeleteTenantController
{
    public function __invoke(Tenant $tenant): Response
    {
        Gate::authorize('delete', $tenant);

        $tenant->delete();

        return redirect('/tenants');
    }
}

## 390. Framework Pipeline

Entonces VoltStack ejecuta:
Route Match
   ↓
Tenant Context
   ↓
Authentication Context Resolution
   ↓
Authorization Metadata
   ↓
Sensitive Operation Metadata
   ↓
Authentication Policy
   ↓
Current Assurance
   ↓
Requirement Gap
   ↓
Challenge if needed
   ↓
Authorization final check
   ↓
Controller

## 391. Authentication + Authorization Integration

Los sistemas siguen siendo independientes:
Authentication
    ↓
Who are you?
How were you authenticated?
How fresh is authentication?
What assurance exists?

Authorization
    ↓
May this principal perform this action?
Pero el framework puede coordinarlos.

## 392. Combined Sensitive Action

Authenticated?
      ↓
Identity eligible?
      ↓
Authorized?
      ↓
Authentication requirement satisfied?
      ↓
Current risk acceptable?
      ↓
Final Authorization recheck
      ↓
Execute

## 393. Critical Rule

Una API cómoda puede coordinar Authentication y Authorization, pero nunca debe confundirlos conceptualmente.

  1. Acceptance Criteria
Este sistema se considerará completo cuando VoltStack permita:

- consultar el principal autenticado mediante facade, helper o DI;
- distinguir user() de principal();
- obtener un AuthenticationContext inmutable;
- consultar assurance sin permitir modificarlo;
- ejecutar login mediante una API sencilla;
- representar MFA/step-up/recovery mediante resultados tipados;
- cerrar sesiones de manera authoritative;
- gestionar sessions mediante public management IDs;
- gestionar métodos Authentication sin manipular DB directamente;
- registrar y administrar passkeys;
- administrar MFA;
- iniciar recuperación;
- iniciar federation;
- exigir fresh Authentication;
- exigir assurance;
- resolver requirements por sensitive operation;
- declarar requisitos mediante attributes;
- declarar requisitos mediante routes;
- integrar Authentication con controllers/actions/components;
- integrar Authentication con el SPA runtime;
- proyectar únicamente información segura al frontend;
- soportar APIs y machine identities;
- soportar CLI Authentication;
- soportar delegation segura para background jobs;
- operar con WebSockets;
- compilar configuración;
- validar configuration security;
- resolver secrets fuera del config cache;
- bootstrapped Authentication mediante service providers;
- clasificar correctamente service lifetimes;
- limpiar runtime state entre requests;
- operar bajo FrankenPHP;
- proporcionar testing helpers;
- evitar test backdoors en production;
- ofrecer diagnostics seguros;
- mantener Authentication y Authorization separados.
  1. Arquitectura Final del Developer Experience
                     APPLICATION
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
      Facade           Helper            DI
    Auth::...          auth()          Contracts
        │                │                │
        └────────────────┼────────────────┘
                         ▼
              Authentication Application API
                         │
          ┌──────────────┼───────────────┐
          │              │               │
          ▼              ▼               ▼
       Reader          Commands         Flows
          │              │               │
          └──────────────┼───────────────┘
                         ▼
                Authentication Core
                         │
       ┌─────────────────┼──────────────────┐
       ▼                 ▼                  ▼
   Identity          Credentials        Sessions
       │                 │                  │
       ├──────────────┬──┴───────┬──────────┤
       ▼              ▼          ▼          ▼
     Policy        Assurance   Challenge   Risk
       │              │          │          │
       └──────────────┼──────────┼──────────┘
                      ▼
              Security Decision
                      │
                      ▼
              Runtime Integration
       ┌──────────────┼───────────────┐
       ▼              ▼               ▼
      HTTP            SPA             CLI
       │              │               │
       └──────────────┼───────────────┘
                      ▼
              Events / Audit / Telemetry
  2. Regla arquitectónica final
La meta de este sistema puede resumirse en:
Easy things
should be easy.

Advanced things
should be possible.

Dangerous things
should be difficult.

Security bypasses
should not exist.
Por ello, VoltStack deberá permitir:
Auth::user();
sin sacrificar la arquitectura necesaria para soportar:
MFA
Passkeys
Step-Up
Risk
Reauthentication
Federation
Machine Identity
Security Incidents
Identity Lifecycle
Multi-Tenancy
Distributed Authentication
FrankenPHP
La regla definitiva será:
La Developer Experience de VoltStack Authentication simplifica el acceso al sistema de seguridad; nunca simplifica ni elimina sus garantías.

  1. Estado de la arquitectura Authentication
Con este documento hemos alcanzado un punto importante.
Los documentos 01–46 definen principalmente qué es y cómo funciona internamente Authentication.
El documento 47 define:
Authentication Core
       ↓
Framework API
       ↓
Application Developer
Ahora podemos construir las dos capas finales de la arquitectura.
  2. Siguiente documento recomendado
El siguiente documento es:
48_AUTHENTICATION_ADMINISTRATION_OPERATIONAL_TOOLING_DIAGNOSTICS_SECURITY_OPERATIONS_AND_PRODUCTION_MANAGEMENT_SYSTEM.md
Su objetivo será diseñar la capa operacional completa:
Authentication System
        ↓
Operational Control Plane
        ↓
┌──────────────────────────────────┐
│ Admin CLI                        │
│ Security Operations              │
│ Diagnostics                      │
│ Health Checks                    │
│ Credential Administration       │
│ Session Administration          │
│ Incident Operations              │
│ Policy Inspection                │
│ Key/Certificate Operations       │
│ Migration Operations             │
│ Tenant Security Administration   │
│ Runtime Inspection               │
│ Production Troubleshooting       │
└──────────────────────────────────┘
Después de 48, la secuencia final recomendada queda:
48  Administration / Operational Tooling
                 ↓
49  Reference Implementation / Default Components
                 ↓
50  System Integration / Final Architecture
Con 50 podremos cerrar formalmente el sistema completo de Authentication de VoltStack y representar cómo todos los subsistemas 01–50 forman una única arquitectura coherente.
