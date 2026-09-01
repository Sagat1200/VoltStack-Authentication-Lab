# VoltStack Authentication System

## 49 — Authentication Reference Implementation, Default Components, Secure Defaults and Framework Integration System

- **Archivo:** `49_AUTHENTICATION_REFERENCE_IMPLEMENTATION_DEFAULT_COMPONENTS_SECURE_DEFAULTS_AND_FRAMEWORK_INTEGRATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica de implementación de referencia
- **Clasificación:** Reference Implementation / Secure Defaults / Framework Integration
- **Dependencias:** 01–48 AUTHENTICATION_*
- **Siguiente documento:** `50_AUTHENTICATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md`

---

## 1. Propósito

Los documentos 01–48 han definido la arquitectura conceptual, contratos, límites de seguridad, políticas, lifecycle, runtime y operación del sistema Authentication.
El documento 49 responde una pregunta distinta:
¿Qué implementación concreta proporciona VoltStack para que Authentication funcione de forma segura inmediatamente después de instalar el framework?

La arquitectura no deberá obligar al desarrollador a implementar manualmente:
PasswordHasher
SessionAuthenticator
AuthenticatorResolver
AuthenticationManager
AuthenticationPolicyEngine
NonceStore
ReplayProtection
MFA
Passkeys
Recovery
Security Events
Rate Limiting
Operational Tooling
para poder crear una aplicación segura.
VoltStack deberá proporcionar una Reference Authentication Implementation completa.

## 2. Objetivo

La implementación predeterminada deberá buscar simultáneamente:
Secure by Default
        +
Production Ready
        +
Laravel-like DX
        +
Symfony-like Architecture
        +
FrankenPHP Safe
        +
Multi-Tenant Ready
        +
Extensible
        +
Observable
        +
Testable

## 3. Principio fundamental

El camino más sencillo para utilizar Authentication en VoltStack deberá ser también un camino seguro.

No:
Easy Mode
    ↓
Insecure Authentication
sino:
Default Mode
    ↓
Secure Authentication

## 4. Reference Implementation != Architecture

VoltStack distinguirá:
Authentication Contracts
        │
        ▼
Architecture
de:
Default Implementations
        │
        ▼
Reference Implementation
Esto permite reemplazar componentes sin reemplazar el modelo conceptual.

## 5. Arquitectura

Application
    │
    ▼
VoltStack Authentication API
    │
    ▼
Authentication Contracts
    │
    ▼
┌─────────────────────────────────────┐
│ Default Reference Implementation    │
├─────────────────────────────────────┤
│ Manager                             │
│ Resolver                            │
│ Authenticators                      │
│ Policy Engine                       │
│ Assurance                           │
│ Challenge Engine                    │
│ Flow Engine                         │
│ Sessions                            │
│ Credentials                         │
│ MFA                                 │
│ Passkeys                            │
│ Recovery                            │
│ Federation                          │
│ Risk                                │
│ Security Incidents                  │
│ Notifications                       │
│ Rate Governance                     │
│ Operational Tooling                 │
└─────────────────────────────────────┘

## 6. Tres niveles de extensibilidad

La implementación se organizará en:
Contracts
    ↓
Default Components
    ↓
Custom Components
Ejemplo:
PasswordHasherInterface
        │
        ├── Argon2idPasswordHasher
        │
        └── CustomEnterprisePasswordHasher

## 7. No Hard-Coded Implementations

El Core no deberá depender directamente de:
new Argon2idPasswordHasher();
sino de:
PasswordHasherInterface
resuelto mediante el Container.

## 8. Default Authentication Stack

Una aplicación VoltStack nueva debería obtener:
Identity authentication
Password authentication
Session authentication
Logout
Remember-me optional
CSRF protection
Rate limiting
Secure password hashing
Authentication events
Audit integration
Authentication context
Policy evaluation
Session rotation
Security headers integration
FrankenPHP isolation
Testing utilities
sin paquetes adicionales.

## 9. Optional Advanced Stack

Podrán habilitarse:
TOTP MFA
Passkeys / WebAuthn
OAuth/OIDC
Enterprise SSO
Recovery
Trusted devices
Adaptive risk
Machine authentication
Security Center
Security incident response
sin sustituir el Authentication Core.

## 10. Default Package

Namespace raíz:
VoltStack\Quantum\Auth
Implementaciones predeterminadas:
VoltStack\Quantum\Auth\Default

## 11. Authentication Service Provider

El framework deberá registrar Authentication durante Bootstrap.
final class AuthenticationServiceProvider
{
    public function register(Container $container): void
    {
        // Authentication services.
    }

    public function boot(AuthenticationRuntime $runtime): void
    {
        // Runtime integrations.
    }
}

## 12. Bootstrap Sequence

Framework Bootstrap
      ↓
Configuration
      ↓
Container
      ↓
AuthenticationServiceProvider
      ↓
Register Contracts
      ↓
Register Default Implementations
      ↓
Compile Authentication Metadata
      ↓
Compile Policies
      ↓
Register Middleware
      ↓
Register Runtime Reset Hooks
      ↓
Authentication Ready

## 13. Compile Before Runtime

Todo lo estático que pueda compilarse deberá resolverse antes del request.
Ejemplos:
authenticator registry
method definitions
policy definitions
route requirements
sensitive operations
realm definitions
provider definitions
operational capabilities

## 14. Runtime State

En cambio:
current identity
current session
current tenant
current realm
current authentication transaction
current risk
current assurance
será request/fiber scoped.

## 15. Default Authentication Manager

Implementación:
DefaultAuthenticationManager
Contrato:
interface AuthenticationManagerInterface
{
    public function authenticate(
        AuthenticationRequest $request
    ): AuthenticationResult;
}

## 16. Responsabilidad

El Manager orquesta:
Request
   ↓
Context Resolution
   ↓
Authenticator Resolution
   ↓
Evidence Verification
   ↓
Identity Resolution
   ↓
Eligibility
   ↓
Assurance
   ↓
Policy
   ↓
Session / Result
No implementará criptografía directamente.

## 17. Default Authenticator Resolver

final class DefaultAuthenticatorResolver
    implements AuthenticatorResolverInterface
{
}
Resolverá autenticadores mediante metadata compilada.

## 18. Resolver Priority

Ejemplo conceptual:
Explicit Route Authenticator
        ↓
Realm Requirement
        ↓
Request Credential Type
        ↓
Configured Priority
        ↓
Applicable Authenticator

## 19. No Guessing Dangerous Credentials

Si múltiples autenticadores reclaman ambiguamente la misma request:
AMBIGUOUS_AUTHENTICATOR
en lugar de seleccionar arbitrariamente.

## 20. Default Authenticators

VoltStack proporcionará:
PasswordAuthenticator
SessionAuthenticator
RememberMeAuthenticator
BearerTokenAuthenticator
PasskeyAuthenticator
TotpAuthenticator
FederatedAuthenticator
RecoveryAuthenticator
MutualTlsAuthenticator
ApiKeyAuthenticator
PrivateKeyJwtAuthenticator
WorkloadIdentityAuthenticator
según capacidades habilitadas.

## 21. Password Authenticator

final class PasswordAuthenticator
    implements AuthenticatorInterface
{
}
Pipeline:
Identifier
    ↓
Identity Lookup
    ↓
Enumeration-Safe Handling
    ↓
Credential Lookup
    ↓
Password Verification
    ↓
Credential Upgrade?
    ↓
Eligibility
    ↓
Evidence

## 22. Default Password Algorithm

La política predeterminada deberá preferir:
Argon2id
cuando esté disponible.

## 23. Password Hashing Policy

No deberá codificarse un único costo eternamente.
interface PasswordHashingPolicyInterface
{
    public function parameters(): PasswordHashParameters;

    public function needsRehash(
        PasswordHashMetadata $metadata
    ): bool;
}

## 24. Progressive Password Upgrade

Legacy Hash
    ↓
Successful Authentication
    ↓
Verify Legacy Hash
    ↓
Generate Argon2id Hash
    ↓
Atomic Credential Replacement

## 25. Never Downgrade Hashes

Si una credencial ya utiliza una política superior:
Argon2id v5
un nodo con configuración antigua no deberá convertirla a:
bcrypt v2

## 26. Dummy Password Verification

Para reducir enumeration/timing leakage:
unknown identity
      ↓
dummy password verification
cuando sea apropiado.

## 27. Password Normalization

VoltStack no deberá realizar transformaciones silenciosas peligrosas sobre passwords.
No:
trim()
strtolower()

## 28. Password Length

Deberá existir:
maximum input length
para proteger recursos ante inputs abusivos.

## 29. Password Policy

La implementación default puede soportar:
minimum length
maximum length
compromised password checking adapter
tenant requirements
administrative requirements
pero evitar reglas obsoletas arbitrarias como:
must contain exactly:
1 uppercase
1 lowercase
1 symbol
1 digit
como única estrategia universal.

## 30. Default Session Authentication

Implementación:
DefaultSessionAuthenticator

## 31. Default Session Architecture

Browser
   ↓
Opaque Session Credential
   ↓
Session Resolver
   ↓
Server-Side Session Record
   ↓
Identity
   ↓
Authentication Context

## 32. Opaque Session Identifier

El cookie no deberá contener toda la identidad serializada.
Preferencia:
opaque high-entropy session credential

## 33. Session Storage

Default:
Database / Cache-backed abstraction
mediante:
SessionRepositoryInterface

## 34. Production Recommendation

Para deployments distribuidos:
Redis-compatible distributed session store
podrá ser recomendado.
Pero Authentication no dependerá directamente de Redis.

## 35. Session Cookie Defaults

Default:
HttpOnly = true
Secure = true in HTTPS production
SameSite = Lax
Path = /
con configuración por realm.

## 36. Admin Realm

Puede utilizar:
separate cookie name
shorter TTL
SameSite policy
no remember-me
stronger assurance

## 37. Session Fixation Protection

Después de Authentication:
anonymous session
      ↓
successful authentication
      ↓
rotate session identifier

## 38. Privilege Change Rotation

También podrá rotarse ante:
reauthentication
privileged elevation
account recovery
major credential change

## 39. Default Session Lifetime

No deberá existir un TTL universal oculto.
Se resolverá mediante policy.
Ejemplo:
user realm:
8h idle
7d absolute

admin realm:
15m idle
4h absolute
configurable.

## 40. Remember-Me

Deshabilitado por defecto para:
admin
privileged
break-glass

## 41. Remember-Me Architecture

Persistent Credential
        ↓
Lookup
        ↓
Verification
        ↓
Rotation
        ↓
Normal Authentication Context

## 42. Remember-Me != Full Assurance

Restaurar remember-me no implica:
fresh authentication
MFA
HIGH assurance
PRIVILEGED context

## 43. Default Logout

Auth::logout();
deberá:
revoke current session
clear authentication context
invalidate/rotate relevant session state
clear cookie
emit event
audit when appropriate

## 44. Global Logout

Auth::logoutEverywhere();
deberá utilizar el authoritative session revocation system.

## 45. Default Bearer Authentication

VoltStack podrá soportar tokens mediante contratos separados.
No deberá asumir que:
Bearer token = JWT

## 46. Token Formats

Pueden existir:
opaque access token
JWT
custom signed token
machine token

## 47. Default API Token

Para aplicaciones first-party simples, la implementación segura recomendada podrá utilizar:
opaque high-entropy token
+
public token ID
+
hashed verifier server-side

## 48. JWT

JWT será soportado cuando el deployment lo requiera.

## 49. JWT Validation

Default validation:
signature
algorithm policy
issuer
audience
expiration
not-before
token purpose
tenant/realm
security epoch/version

## 50. Algorithm Confusion

Nunca aceptar algoritmo del token sin policy.

## 51. none

alg = none
deberá rechazarse.

## 52. Default Identity Provider

Implementación:
DatabaseIdentityProvider
mediante:
IdentityProviderInterface

## 53. Identity Lookup

Soportará identificadores configurables:
email
username
external subject
internal identity ID
sin convertir todos ellos en el Identity.

## 54. Stable Identity ID

Default:
opaque immutable identity identifier

## 55. Email != Identity ID

Cambiar email no cambia Identity.

## 56. Identity Repository

interface IdentityRepositoryInterface
{
    public function find(
        IdentityLookup $lookup
    ): ?Identity;
}

## 57. Default Authentication Context

final readonly class DefaultAuthenticationContext
    implements AuthenticationContextInterface
{
}
Incluirá:
identity
principal type
methods
factors
assurance
authentication time
freshness
session
device trust
tenant
realm
risk
security posture

## 58. Context Is Immutable

Para step-up:
Context V1
    ↓
new evidence
    ↓
Context V2
No mutar el objeto compartido.

## 59. Default Context Accessor

DX:
Auth::user();
Auth::identity();
Auth::context();
Auth::check();
Auth::guest();

## 60. Request Scope

El accessor obtiene el contexto del request/fiber actual.
Nunca de:
static $currentUser;

## 61. Default Authentication Policy Engine

Implementación:
DefaultAuthenticationPolicyEngine

## 62. Sources

Default engine combinará:
Framework Security Floor
Platform Configuration
Realm Configuration
Tenant Policy
Application Policy
Route Metadata
Operation Metadata
Identity Security State
Dynamic Risk Overlay

## 63. Default Merge Semantics

Hardening-first:
minimum assurance → strongest
maximum age       → shortest
required MFA      → OR
phishing resistant→ OR
allowed methods   → intersection
forbidden methods → union
minimum factors   → maximum

## 64. Unsatisfiable Policy

Resultado explícito:
AUTH_POLICY_UNSATISFIABLE

## 65. No Silent Downgrade

Si policy exige:
PASSKEY
y usuario sólo tiene SMS:
DENIED / ENROLLMENT REQUIRED
según flow.
No:
use SMS anyway

## 66. Default Assurance Resolver

DefaultAuthenticationAssuranceResolver

## 67. Assurance Inputs

verified evidence
method properties
factor independence
user verification
user presence
phishing resistance
hardware backing
device/workload binding
freshness
risk

## 68. Roles Never Raise Assurance

ROLE_ADMIN
no convierte:
password-only
en:
HIGH assurance

## 69. Default Assurance Levels

enum AuthenticationAssuranceLevel: int
{
    case Low = 10;
    case Standard = 20;
    case Strong = 30;
    case High = 40;
    case Privileged = 50;
}

## 70. Default Challenge Negotiator

Implementación:
DefaultAuthenticationChallengeNegotiator

## 71. Selection Pipeline

Requirement Gap
      ↓
Security Eligibility
      ↓
Available Methods
      ↓
Client Compatibility
      ↓
Tenant / Realm Policy
      ↓
Risk
      ↓
Preference
      ↓
Challenge Plan

## 72. Default Preference

Para strong authentication, cuando estén disponibles:
Passkey/WebAuthn
      ↓
strong MFA
      ↓
TOTP
      ↓
lower-assurance channels
según policy.

## 73. SMS Is Not Default Strongest Method

SMS podrá soportarse, pero no se clasificará automáticamente como phishing-resistant.

## 74. Default Flow Engine

DefaultAuthenticationFlowEngine

## 75. Unified Flows

Mismo engine para:
login
reauthentication
step-up
recovery
credential enrollment
account linking
privileged authentication
federation

## 76. Default Transaction Repository

Contrato:
AuthenticationTransactionRepositoryInterface
Implementaciones:
DatabaseAuthenticationTransactionRepository
CacheAuthenticationTransactionRepository

## 77. Default Production Strategy

Para aplicaciones distribuidas:
shared transactional/cache store
y no memoria local del worker.

## 78. No In-Memory Production Transactions

static array $transactions = [];
queda prohibido para runtime distribuido.

## 79. Default Transaction TTL

Dependerá del flow.
Ejemplos:
login             10m
step-up            5m
credential bind   10m
recovery          15m
configurable.

## 80. Default Nonce Generator

final class SecureAuthenticationNonceGenerator
{
    public function generate(): AuthenticationNonce
    {
        return AuthenticationNonce::fromBytes(
            random_bytes(32)
        );
    }
}
conceptualmente.

## 81. CSPRNG Only

Nunca:
rand()
mt_rand()
uniqid()
timestamp

## 82. Default Replay Store

Contrato:
AuthenticationReplayStoreInterface

## 83. Atomic Consume

Debe garantizar:
first consume  → success
second consume → replay detected

## 84. Default Database Implementation

Puede utilizar unique constraints / atomic updates.

## 85. Default Distributed Implementation

Redis-compatible:
SET NX
o mecanismo atómico equivalente.

## 86. Default CSRF Integration

Authentication utilizará el sistema CSRF del framework.

## 87. Authentication-Specific CSRF

Especial atención a:
login CSRF
account linking
credential enrollment
credential removal
recovery
reauthentication

## 88. SameSite Is Defense-in-Depth

No sustituye token/binding donde sea necesario.

## 89. Default Protected State

OAuth state, continuation y otros artifacts utilizarán:
AuthenticationStateProtectorInterface

## 90. Cryptographic Keys

Key purposes separados:
SESSION_SIGNING
STATE_PROTECTION
CONTINUATION_PROTECTION
ADMIN_SESSION_SIGNING
MACHINE_TOKEN_SIGNING
WEBHOOK_SIGNING

## 91. No Universal Application Secret

Evitar:
APP_KEY
como única clave para todas las funciones criptográficas de Authentication.

## 92. Key Derivation

Cuando sea apropiado podrán derivarse subkeys por purpose mediante mecanismos criptográficos establecidos.

## 93. Default MFA

VoltStack proporcionará TOTP como MFA estándar opcional.

## 94. TOTP Secret Generation

CSPRNG.

## 95. TOTP Storage

Secret cifrado/protegido mediante secret protection infrastructure.
Nunca plaintext accidental en:
logs
cache dumps
debug output

## 96. TOTP Enrollment

Begin Enrollment
      ↓
Generate Secret
      ↓
Present QR / Manual Key
      ↓
User Submits TOTP
      ↓
Verify
      ↓
Activate Method

## 97. Unverified TOTP

No cuenta como Authentication Method activo.

## 98. TOTP Replay

Cuando sea requerido por security level, podrá impedirse reutilizar el mismo timestep exitoso.

## 99. Default Recovery Codes

Generados criptográficamente.

## 100. Recovery Code Storage

code
 ↓
hash/verifier
 ↓
database
No plaintext.

## 101. Show Once

Los códigos se muestran únicamente durante generación.

## 102. Recovery Code Consumption

Atomic single-use.

## 103. Default Passkeys/WebAuthn

VoltStack deberá incluir una implementación WebAuthn compatible con estándares mediante una capa dedicada.

## 104. WebAuthn Components

WebAuthnRegistrationManager
WebAuthnAuthenticationManager
WebAuthnChallengeRepository
WebAuthnCredentialRepository
WebAuthnRpPolicy
WebAuthnOriginPolicy

## 105. RP Configuration

'passkeys' => [
    'rp_id' => env('AUTH_WEBAUTHN_RP_ID'),
    'rp_name' => env('APP_NAME'),
],

## 106. Origin Validation

Nunca confiar únicamente en:
Host header
para determinar origins válidos.

## 107. Explicit Allowed Origins

Production:
'allowed_origins' => [
    'https://example.com',
];

## 108. User Verification

Configurable:
required
preferred
discouraged
con required para operaciones de mayor assurance cuando corresponda.

## 109. Passkey Credential Storage

Guardar sólo información server-side necesaria:
credential ID
public key
counter metadata
transports
backup eligibility/state
user handle
timestamps
status

## 110. Private Key Never Reaches Server

La implementación jamás deberá esperar private key del authenticator.

## 111. Default Federation

VoltStack proporcionará una implementación OIDC estándar.

## 112. OAuth2 vs OIDC

Para Authentication de usuarios:
OIDC
será preferido cuando el proveedor lo soporte.

## 113. Default OIDC Components

OidcProviderRegistry
OidcDiscoveryResolver
OidcAuthorizationRequestFactory
OidcCallbackHandler
OidcTokenValidator
OidcJwksResolver
OidcIdentityMapper

## 114. OIDC Defaults

state required
PKCE S256
nonce required where applicable
issuer validation
audience validation
signature validation
redirect URI validation
short-lived transaction

## 115. External Email Linking

Deshabilitado por defecto.
No:
Google email == local email
    ↓
automatically same identity

## 116. Explicit Linking

Debe pasar por doc34.

## 117. Default Recovery System

El sistema default soportará recuperación mediante mecanismos configurados.

## 118. Recovery Is Not Normal Login

Tendrá:
separate flow purpose
separate policy
separate transaction
separate rate limits
separate security events

## 119. Recovery Context

El resultado puede ser:
RESTRICTED_AUTHENTICATION
hasta completar security review.

## 120. Default Trusted Device System

Deshabilitado hasta que la aplicación lo configure explícitamente.

## 121. Why

"Remember this device" tiene implicaciones:
security
privacy
device lifecycle
risk

## 122. Device Trust Credential

Cuando se habilite:
opaque credential
rotatable
revocable
device-bound where possible
tenant/realm scoped

## 123. Device Trust Never Replaces Identity

No:
trusted device
   =

authenticated user

## 124. Default Risk Engine

VoltStack deberá proporcionar un engine básico determinista.
No necesita ML.

## 125. Default Signals

Puede considerar:
authentication failures
new device
new location approximation
credential changes
recovery activity
session anomalies
rate-limit state
security incidents
provider compromise

## 126. Risk Levels

LOW
MEDIUM
HIGH
CRITICAL

## 127. Risk Does Not Directly Grant Access

Risk produce contexto/requirements.
Authorization sigue independiente.

## 128. Default Security Incident Engine

Proporcionará reglas base para:
brute force
credential stuffing indicators
MFA fatigue
replay
credential compromise
suspicious security changes
probable takeover

## 129. Automated Containment

Conservative by default.
Ejemplo:
confirmed replay
    ↓
invalidate transaction
pero no necesariamente:
delete account

## 130. Default Protection Actions

require reauthentication
require step-up
revoke session
increment security epoch
restrict identity
security review
notify user
según policy.

## 131. Default Communication System

VoltStack incluirá:
EmailAuthenticationCommunicationProvider
si el framework Mail está disponible/configurado.
Otros adapters:
SMS
Push
WebPush
Webhook
serán intercambiables.

## 132. Notification Failure

Nunca deberá revertir una security mutation ya confirmada.

## 133. Example

Password changed successfully
       ↓
notification provider fails
Resultado:
PASSWORD_CHANGED
NOTIFICATION_PENDING/FAILED
No:
ROLLBACK PASSWORD CHANGE

## 134. Default Security Messages

Plantillas para:
new login
password changed
MFA changed
passkey added
passkey removed
recovery initiated
recovery completed
email changed
suspicious authentication
session revoked
account restricted

## 135. Secure Template Rules

Nunca incluir:
password
TOTP seed
recovery codes
session credential
raw bearer token

## 136. Default Rate Governance

Implementación:
DefaultAuthenticationRateGovernor

## 137. Default Dimensions

source network
identity candidate
resolved identity
tenant
realm
credential
device
purpose
provider
según disponibilidad.

## 138. Enumeration Protection

Los límites no deberán revelar fácilmente si una identidad existe.

## 139. Progressive Defense

normal
   ↓
delay
   ↓
stronger challenge
   ↓
temporary restriction
   ↓
incident response
según señales.

## 140. No Default Permanent Lockout

Un atacante no debería poder bloquear permanentemente una cuenta sólo enviando passwords incorrectos.

## 141. Password Authentication Default Limits

Valores exactos serán configurables y deployment-dependent.
El framework deberá proporcionar valores conservadores iniciales, pero la arquitectura no deberá convertirlos en constantes universales.

## 142. Default Background Tasks

VoltStack registrará tareas como:
auth.transactions.expire
auth.nonces.cleanup
auth.sessions.cleanup
auth.remember_me.cleanup
auth.recovery.cleanup
auth.security_events.retention
auth.provider_metadata.refresh
auth.credentials.expiration_check
auth.certificates.expiration_check
auth.notifications.cleanup
auth.incidents.maintenance
auth.privacy.retention

## 143. Scheduler Integration

VoltStack Scheduler
       ↓
Authentication Maintenance Registry
       ↓
Task Handlers

## 144. Same Handler for CLI

voltstack auth:maintenance:run transactions-expire
utiliza el mismo handler.

## 145. Default Queue Integration

Security-critical asynchronous tasks deberán usar colas diferenciables.
Ejemplo:
auth-security-critical
auth-notifications
auth-maintenance
auth-migration

## 146. Default Database Schema

La implementación de referencia deberá proporcionar migrations oficiales.

## 147. Core Tables

Conceptualmente:
auth_identities
auth_identity_identifiers
auth_authentication_methods
auth_credentials
auth_sessions
auth_remember_credentials
auth_security_epochs

## 148. Transaction Tables

auth_transactions
auth_challenges
auth_replay_records
auth_continuations

## 149. MFA / Passkey

auth_mfa_methods
auth_passkeys
auth_recovery_credentials

## 150. Federation

auth_external_identities
auth_federation_transactions

## 151. Device

auth_devices
auth_device_credentials
auth_device_trust

## 152. Security

auth_security_events
auth_security_findings
auth_security_alerts
auth_security_incidents
auth_incident_actions

## 153. Communication

auth_contact_points
auth_contact_verifications
auth_communications
auth_delivery_attempts

## 154. Operations

auth_operational_records
auth_operational_approvals

## 155. Policy

Cuando policies dinámicas se almacenen en DB:
auth_policy_sets
auth_policy_versions
auth_policy_assignments

## 156. Privacy

Cuando se necesiten:
auth_consent_records
auth_retention_holds
auth_erasure_records

## 157. Not Every App Needs Every Table

La instalación podrá utilizar migration capabilities/modules.
Ejemplo:
core
sessions
mfa
passkeys
federation
security-center
operations
machine-auth

## 158. Schema Ownership

Cada tabla pertenece a un subdominio.
No crear:
users
con 150 columnas Authentication.

## 159. Application User Model

VoltStack deberá permitir:
final class User
{
    public IdentityId $identityId;
}
o asociación equivalente.

## 160. Identity != Domain User

Authentication Identity representa seguridad.
Application User representa dominio de negocio.

## 161. Default Integration Strategy

Para aplicaciones sencillas, VoltStack puede proporcionar un modelo integrado conveniente.
Pero la arquitectura debe mantener separables:
Authentication Identity
Application User/Profile

## 162. Authentication Middleware

Default middleware:
Authenticate
RequireGuest
RequireAuthenticationAssurance
RequireFreshAuthentication
RequireMfa
RequirePhishingResistantAuthentication
RequireTrustedDevice
ResolveAuthenticationContext

## 163. Example

Route::get('/dashboard', DashboardController::class)
    ->middleware('auth');

## 164. Strong Authentication

Route::post('/billing/payment-method', UpdatePaymentMethod::class)
    ->middleware([
        'auth',
        'auth.assurance:strong',
    ]);

## 165. Preferred Metadata API

Más expresivo:
\# [Authenticated]
\# [RequiresAssurance(AuthenticationAssuranceLevel::Strong)]
final class UpdatePaymentMethod
{
}

## 166. Sensitive Operation

\# [SensitiveOperation('tenant.delete')]
final class DeleteTenantController
{
}
Policy engine obtiene el requirement.

## 167. Avoid Middleware Explosion

No será necesario crear middleware para cada combinación.
Preferencia:
metadata
    ↓
policy engine

## 168. Controller Parameter Injection

public function __invoke(
    AuthenticatedIdentity $identity,
    AuthenticationContext $auth
): Response {
}

## 169. Invalid Injection

Si endpoint requiere identity pero request es guest:
AuthenticationRequired
antes de invocar controller.

## 170. Route Compilation

Authentication metadata se integrará con Route Compiler.
Route
   ↓
Attributes
   ↓
Authentication Metadata
   ↓
Compiled Route

## 171. No Runtime Reflection When Avoidable

Production cache deberá contener metadata compilada.

## 172. Authentication Facade

VoltStack proporcionará:
Auth
como facade de DX.

## 173. Core Methods

Auth::check();

Auth::guest();

Auth::identity();

Auth::user();

Auth::context();

Auth::session();

Auth::assurance();

Auth::login(...);

Auth::logout();

Auth::logoutEverywhere();

Auth::require(...);

## 174. Facade Is Thin

No deberá contener business/security logic.

## 175. Example

if (Auth::check()) {
    $identity = Auth::identity();
}

## 176. Explicit Dependency Injection Preferred Internally

Framework internals deberán preferir:
AuthenticationContextInterface
sobre facade.
Facade está orientada principalmente a application DX.

## 177. Helpers

Opcionales:
auth()
authenticated()
guest()

## 178. auth()

auth()->identity();
auth()->context();
auth()->check();

## 179. Helper Safety

Nunca devolver un mutable global singleton.

## 180. Login DX

Ejemplo simple:
$result = Auth::attempt([
    'email' => $request->email,
    'password' => $request->password,
]);

## 181. But Internally

Auth::attempt()
     ↓
AuthenticationRequest
     ↓
AuthenticationManager
     ↓
Authenticator Resolver
     ↓
Password Authenticator
     ↓
Policy / Assurance / Session

## 182. No Shortcut Security Path

La facade no tendrá un pipeline alternativo simplificado.

## 183. Login Result

No sólo true/false.
$result = Auth::attempt(...);

if ($result->completed()) {
}

if ($result->challengeRequired()) {
}

## 184. Typed Results

Ejemplos:
Authenticated
ChallengeRequired
RedirectRequired
RecoveryRequired
EnrollmentRequired
Denied
Failed

## 185. Boolean Convenience

Podrá existir:
Auth::validateCredentials(...)
pero no deberá ocultar flows interactivos cuando sean necesarios.

## 186. SPA Integration

VoltStack SPA runtime deberá entender Authentication flow responses.

## 187. Example

{
  "type": "authentication.challenge_required",
  "transaction": "atx_...",
  "challenge": {
    "type": "passkey"
  }
}

## 188. SPA Runtime

Application Request
      ↓
Authentication Requirement
      ↓
Challenge Required
      ↓
SPA Runtime
      ↓
Authentication UI
      ↓
Challenge Response
      ↓
Transaction Resume
      ↓
Original Intent

## 189. Unsafe Automatic Replay

POST destructivo no se reenvía ciegamente.

## 190. Intent Binding

Se utilizará doc39:
operation
resource
payload digest
continuation

## 191. Blade/View Integration

Conceptualmente:
@auth
    ...
@endauth

@guest
    ...
@endguest

## 192. Assurance Directives

Podrán existir:
@authAssurance('strong')
    ...
@endAuthAssurance
pero sólo para presentación.

## 193. UI Is Not Authorization

Ocultar botón:
!=
protect endpoint

## 194. Reactive Component Integration

VoltStack components deberán recibir Authentication context de forma segura.

## 195. Hydration

Nunca confiar en:
client-hydrated identity
como authoritative Authentication state.

## 196. Server Rehydrates Security Context

Cada interacción relevante obtiene authoritative server context.

## 197. Client Manifest

Puede conocer:
authenticated: true
assurance: strong
para UX.
Pero backend siempre revalida.

## 198. Authentication State Changes

Cuando cambie:
login
logout
step-up
session revoked
security restriction
SPA runtime deberá poder actualizar estado.

## 199. Default Login Routes

VoltStack puede registrar opcionalmente:
/login
/logout
/auth/challenge/*
/auth/recovery/*
/auth/passkeys/*

## 200. Headless Mode

Debe ser posible:
'auth' => [
    'routes' => false,
];

## 201. UI Optional

Authentication Core no dependerá de templates.

## 202. Default Controllers

Cuando routes/UI estén habilitados:
LoginController
LogoutController
ChallengeController
RecoveryController
PasskeyController
SecurityCenterController
serán adapters finos.

## 203. No Business Logic in Controllers

Controller
   ↓
Authentication Application Service
   ↓
Domain/Core

## 204. Configuration

Archivo conceptual:
config/auth.php

## 205. Example

return [

    'default_realm' => 'user',

    'identity' => [
        'provider' => 'database',
    ],

    'passwords' => [
        'algorithm' => 'argon2id',
    ],

    'sessions' => [
        'driver' => 'database',
        'rotate_on_login' => true,
    ],

    'mfa' => [
        'totp' => true,
    ],

    'passkeys' => [
        'enabled' => true,
    ],

    'federation' => [
        'enabled' => false,
    ],

];

## 206. Secure Production Validation

Si:
APP_ENV=production
y:
Secure cookies disabled
VoltStack deberá:
fail configuration validation
cuando la deployment topology requiera HTTPS.

## 207. Configuration Compiler

AuthenticationConfigurationCompiler
validará configuración antes del tráfico.

## 208. Configuration Errors

Ejemplos:
AUTH_CONFIG_SESSION_COOKIE_INSECURE
AUTH_CONFIG_WEBAUTHN_RP_ID_MISSING
AUTH_CONFIG_WEBAUTHN_ORIGIN_INVALID
AUTH_CONFIG_OIDC_REDIRECT_URI_INVALID
AUTH_CONFIG_KEY_PURPOSE_MISSING
AUTH_CONFIG_POLICY_UNSATISFIABLE

## 209. No Silent Production Fallback

Si KMS configurado falla:
No:
fallback to hard-coded local key

## 210. Development Defaults

Development puede proporcionar:
local key provider
database sessions
local mail capture
pero deberá quedar claramente clasificado como development infrastructure.

## 211. Environment-Aware Defaults

local
testing
staging
production
pueden tener defaults diferentes.

## 212. Testing Environment

Testing deberá ser rápido y determinista sin debilitar contratos.

## 213. Test Password Hasher

Puede existir:
FastTestingPasswordHasher
pero únicamente en testing.

## 214. Production Guard

El container compiler deberá impedir:
FastTestingPasswordHasher
en producción.

## 215. Authentication Testing Toolkit

Namespace:
VoltStack\Testing\Auth

## 216. Test Helpers

actingAs($identity);

actingAsAdmin($identity);

withAssurance(
    AuthenticationAssuranceLevel::Strong
);

withTrustedDevice();

withAuthenticationMethod(...);

withoutAuthentication();

## 217. Example

$this
    ->actingAs($user)
    ->withAssurance(AuthenticationAssuranceLevel::Strong)
    ->post('/settings/security')
    ->assertOk();

## 218. Test Context Is Explicitly Synthetic

actingAs() no deberá ejecutar fake production login internamente.
Construye Authentication testing context controlado.

## 219. Integration Authentication Tests

También podrá ejecutarse:
$this->authenticateUsingPassword(...);
para probar el pipeline real.

## 220. Passkey Testing

Utilities para crear:
synthetic authenticator
synthetic credential
valid assertion
invalid assertion
replayed assertion
wrong origin
wrong RP ID

## 221. OIDC Testing

Fake provider:
FakeOidcProvider
con:
issuer
JWKS
authorization response
token response
claims
key rotation
failure modes

## 222. Time Testing

$this->freezeAuthenticationTime(...);
$this->travelAuthenticationTime(...);
mediante AuthenticationClockInterface.

## 223. Replay Testing

$proof = $this->createAuthenticationProof();

$this->consume($proof)->assertSuccess();
$this->consume($proof)->assertReplayDetected();

## 224. FrankenPHP Testing

Reference implementation deberá incluir pruebas:
Request A → User A
Request B → User B
Worker reused
y verificar que B jamás recibe A.

## 225. Concurrency Testing

También:
two fibers
two authentication contexts
one worker

## 226. Default Events

Ejemplos:
AuthenticationAttempted
AuthenticationSucceeded
AuthenticationFailed
AuthenticationChallengeCreated
AuthenticationChallengeVerified
AuthenticationSessionCreated
AuthenticationSessionRevoked
AuthenticationLoggedOut
AuthenticationMethodAdded
AuthenticationMethodRemoved
AuthenticationCredentialRotated
AuthenticationRiskChanged
AuthenticationIncidentOpened

## 227. Event Payload Rules

No secrets.

## 228. Default Audit Integration

Security-sensitive events serán enviados al Authentication Audit system.

## 229. Event != Audit

No todos los eventos necesitan audit durable.

## 230. Audit Examples

Sí:
password changed
MFA disabled
passkey removed
admin session revoked
identity suspended
break-glass activated

## 231. Ordinary Telemetry

Por ejemplo:
authenticator execution duration
es telemetry, no necesariamente audit.

## 232. Default Metrics

auth_attempts_total
auth_success_total
auth_failures_total
auth_challenges_total
auth_sessions_created_total
auth_sessions_revoked_total
auth_step_up_total
auth_recovery_total

## 233. Metrics Dimensions

Permitidas:
authenticator type
realm
result category
assurance category
controlando cardinalidad.

## 234. Never

email
identity_id
session_id
token_id
como labels normales.

## 235. Default Tracing

Spans:
auth.authenticate
auth.resolve_authenticator
auth.verify_evidence
auth.resolve_identity
auth.evaluate_policy
auth.evaluate_assurance
auth.create_session

## 236. Trace Sanitization

No credential values.

## 237. Default Exception Mapping

Authentication failures deberán convertirse según transporte.

## 238. Browser

redirect/challenge
cuando corresponda.

## 239. API

{
  "error": "authentication_required"
}

## 240. SPA

Typed protocol response.

## 241. CLI

Human-readable safe message + stable error code.

## 242. Enumeration-Safe Failure

No diferenciar públicamente:
email not found
password wrong
cuando eso revele existencia.

## 243. Internal Reason

Internamente sí pueden distinguirse:
IDENTITY_NOT_FOUND
CREDENTIAL_INVALID
para security analytics, sujeto a privacy policy.

## 244. Default Security Center

VoltStack deberá proporcionar backend de referencia para:
sessions
devices
methods
passkeys
federated identities
recovery readiness
security activity
alerts

## 245. UI Package Separation

El Core devuelve:
IdentitySecuritySnapshot
La UI decide presentación.

## 246. Default Security Center Routes

Opcionales:
/settings/security
/settings/security/sessions
/settings/security/passkeys
/settings/security/devices
/settings/security/activity

## 247. Every Mutation Revalidates

Aunque UI diga:
canRemovePasskey = true
backend revalida al ejecutar.

## 248. Default Machine Authentication

Para service-to-service, VoltStack proporcionará componentes para:
API keys
mTLS
private_key_jwt
workload identity

## 249. Machine Realm

Recomendación:
machine
separado de:
user
admin

## 250. Machine Sessions

No usar PHP/browser session.
Preferencia:
short-lived credentials

## 251. API Key Default Format

Conceptualmente:
vsk_live_<public-id>_<secret>
sin fijar necesariamente esa sintaxis.

## 252. Public ID

Permite localizar verifier sin almacenar secret.

## 253. Secret

CSPRNG de alta entropía.

## 254. Storage

public ID
hash/verifier
metadata
status

## 255. Environment Prefix

Puede ayudar a prevenir:
test key used in production
pero no sustituye validación server-side.

## 256. Default mTLS

Validará:
chain
trust anchor
validity
revocation policy
EKU
identity mapping
tenant/realm

## 257. Reverse Proxy

Client certificate headers sólo se aceptarán desde proxies explícitamente confiables.

## 258. Never Trust Arbitrary Header

X-Client-Cert
desde Internet.

## 259. Default Workload Identity

Proporcionará contracts/adapters para:
Kubernetes
cloud identity
CI OIDC
SPIFFE-like identities
sin acoplar el Core a un proveedor.

## 260. Multi-Tenant Defaults

Cada Authentication Context deberá resolver explícitamente:
tenant
realm
cuando la aplicación sea multi-tenant.

## 261. Tenant Resolution Before Authentication

En algunos deployments:
Host
 ↓
Tenant
 ↓
Authentication

## 262. Identity Before Tenant

Otros pueden utilizar:
Authentication
 ↓
Identity memberships
 ↓
Tenant selection

## 263. Both Supported

Pero el flow deberá declarar cuál estrategia utiliza.

## 264. Cross-Tenant Session Safety

Session deberá contener/bind:
tenant context
cuando sea tenant-specific.

## 265. Tenant Switch

Cambiar tenant puede requerir:
new scoped context
session rotation
policy reevaluation
authorization reevaluation

## 266. No Tenant Mutable Singleton

Crítico para FrankenPHP.

## 267. Default Realm Configuration

Ejemplo:
'realms' => [

    'user' => [
        'session' => true,
        'remember_me' => true,
    ],

    'admin' => [
        'session' => true,
        'remember_me' => false,
        'minimum_assurance' => 'strong',
    ],

    'machine' => [
        'session' => false,
    ],

];

## 268. Realm != Role

Admin realm no significa automáticamente:
ROLE_ADMIN

## 269. Realm Is Authentication Boundary

Authorization decide roles/permissions.

## 270. Default Admin Security Floor

Recomendado:
MFA required
shorter session
no remember-me
fresh authentication for sensitive operations
phishing-resistant method preferred/required where configured

## 271. Default Break-Glass

Deshabilitado salvo configuración explícita.

## 272. Enabling Break-Glass

Debe requerir configuración consciente.
No generar automáticamente:
admin/admin

## 273. Default Operational Tooling

Documento 48 tendrá implementación oficial.

## 274. Commands

Incluyendo:
auth:diagnose
auth:health:status
auth:identity:*
auth:session:*
auth:credential:*
auth:policy:*
auth:incident:*
auth:key:*
auth:migration:*
auth:maintenance:*
auth:runtime:*

## 275. Tooling Uses Same Core

No duplicate security logic.

## 276. Default Health Checks

configuration
container
session store
transaction store
replay store
policy
keys
provider metadata
runtime isolation
scheduler
queue

## 277. Production Health Endpoint

Minimal public output.
Detailed health requiere operational authority.

## 278. Default Diagnostics

voltstack auth:diagnose
deberá funcionar inmediatamente.

## 279. Secure Defaults Matrix

Componente Default
Password hashing Argon2id
Session fixation protection Enabled
Session cookie HttpOnly Enabled
Session cookie Secure Production HTTPS
CSRF Enabled for browser flows
Login throttling Enabled
Credential enumeration protection Enabled
Session rotation Enabled
Replay protection Enabled
OAuth state Required
OIDC nonce Required where applicable
PKCE S256
Passkey origin validation Strict
MFA secrets in logs Forbidden
Recovery codes plaintext storage Forbidden
Remember-me admin realm Disabled
Break-glass Disabled by default
External account auto-link by email Disabled
Production debug secret exposure Forbidden
Authentication runtime mutable globals Forbidden

  1. Secure Defaults Must Be Versioned
Un default seguro en 2026 puede dejar de serlo.
  2. Security Profile Version
AuthenticationSecurityProfileVersion
Ejemplo conceptual:
voltstack-auth-security-profile-v1
  3. Why Version Defaults
Una aplicación existente no debería cambiar comportamiento crítico silenciosamente sólo por una actualización patch.
  4. Security Profile Upgrade
voltstack auth:security-profile:status
  5. Example
Current profile:
v1

Available:
v2

Changes:

- stronger Argon2id parameters
- admin session maximum reduced
- OAuth PKCE enforcement tightened
- deprecated SHA-1 certificate policy removed
  1. Progressive Security Upgrade
Integración directa con documento 46.
  2. Framework Upgrade
Composer update puede instalar soporte para nuevos defaults.
Pero activación de cambios incompatibles deberá gobernarse.
  3. Security Critical Emergency Changes
VoltStack podrá marcar ciertos defaults como:
mandatory security floor
cuando mantener comportamiento anterior sea inseguro.
  4. Compatibility vs Security
Regla:
Backward compatibility nunca debe obligar a aceptar indefinidamente una vulnerabilidad conocida.

  5. Deprecation
Authentication methods débiles podrán pasar:
SUPPORTED
    ↓
DEPRECATED
    ↓
RESTRICTED
    ↓
DISABLED
    ↓
REMOVED
  6. Example
Un algoritmo legacy puede:
verify existing credential
pero no:
create new credential
  7. Default Component Registry
interface AuthenticationDefaultComponentRegistryInterface
{
    public function components(): iterable;
}
  8. Component Metadata
Cada implementación podrá declarar:
contract
implementation
security profile
environment support
capabilities
deprecated?
  9. Default Binding Example
$container->bind(
    PasswordHasherInterface::class,
    Argon2idPasswordHasher::class
);
 10. Override
Application:
$container->bind(
    PasswordHasherInterface::class,
    EnterprisePasswordHasher::class
);
 11. Replacement Validation
El Container Compiler podrá verificar que custom implementation declare capacidades requeridas.
 12. Security Capability Contract
Ejemplo:
interface AuthenticationSecurityCapabilityProviderInterface
{
    public function capabilities(): AuthenticationSecurityCapabilities;
}
 13. Why
Un plugin custom no debería afirmar:
phishing-resistant = true
sin cumplir contrato correspondiente.
 14. Trust Boundary
Plugins Authentication son security-sensitive code.
 15. Plugin Registration
Debe ser explícita.
No auto-cargar cualquier clase encontrada en vendor como authenticator.
 16. Authenticator Metadata
\# [Authenticator(
    id: 'password',
    credential: PasswordCredential::class,
    priority: 100
)]
final class PasswordAuthenticator
{
}
 17. Compile Registry
Discovery
   ↓
Validation
   ↓
Conflict Detection
   ↓
Compile
   ↓
Immutable Authenticator Registry
 18. Duplicate Authenticator ID
Build/configuration error.
 19. Unknown Dependency
Build error cuando sea posible.
 20. Runtime Plugin Failure
Debe aislarse según criticality.
 21. No Fallback to Insecure Authenticator
Si passkey plugin falla y policy exige phishing resistance:
authentication unavailable
no:
fallback password
 22. Framework Event Integration
Authentication se conectará con VoltStack Event System.
 23. Framework Cache Integration
Sólo metadata segura y cacheable.
 24. Never Cache
raw password
TOTP input
recovery code
PKCE verifier in generic cache
private key
session cookie
 25. Dedicated Sensitive Stores
Artifacts sensibles usan repositories con lifecycle explícito.
 26. Framework Database Integration
Authentication utilizará el Database System para:
transactions
locking
optimistic concurrency
migrations
repositories
outbox
 27. No ORM Magic Required
Authentication contracts no deberán depender exclusivamente de Active Record.
 28. Framework Queue Integration
Para:
notifications
cleanup
security enrichment
migration
incident processing
 29. Framework Scheduler Integration
Para maintenance.
 30. Framework Telemetry Integration
Con:
Logging
Metrics
Tracing
Profiling
OpenTelemetry
Prometheus
según Telemetry System.
 31. Framework Authorization Integration
Boundary:
Authentication
    ↓
Authenticated Principal
    ↓
Authorization
 32. Authentication Never Calls isAdmin() as Security Shortcut
Authorization resuelve permisos.
 33. Authorization Can Require Authentication Context
Ejemplo:
Can delete tenant?
        ↓
Authorization says YES
        ↓
Authentication requirement says HIGH + FRESH
        ↓
Step-up
 34. Bidirectional Coordination Without Domain Collapse
Authentication
   │
   ├── provides principal/context
   │
Authorization
   │
   └── determines action permission
Policy coordination puede ocurrir, pero siguen siendo sistemas distintos.
 35. Framework Controller Integration
Controller Resolver podrá inyectar:
AuthenticatedIdentity
AuthenticationContext
AuthenticationSession
 36. Framework Routing Integration
Route metadata puede declarar Authentication requirements.
 37. Framework Middleware Integration
Authentication ejecutará en posición definida del HTTP pipeline.
 38. Suggested Pipeline
Request
  ↓
Trusted Proxy Resolution
  ↓
Tenant Resolution
  ↓
Request Context
  ↓
Authentication Context Resolution
  ↓
Authentication Requirement
  ↓
Authorization
  ↓
Controller
El orden exacto podrá variar por tenant strategy.
 39. Exception Pipeline
Authentication exceptions se integran con global exception handling.
 40. SPA Integration
First-class, no add-on.
 41. FrankenPHP Integration
First-class, no compatibility patch.
 42. Persistent Worker Architecture
Clasificar servicios:
IMMUTABLE_SHARED
REQUEST_SCOPED
TRANSACTION_SCOPED
JOB_SCOPED
 43. Shared Services
Ejemplos:
compiled policy registry
compiled authenticator registry
password hashing policy
immutable configuration
 44. Request-Scoped
AuthenticationContext
CurrentIdentity
CurrentSession
CurrentTenant
CurrentRealm
CurrentRiskContext
 45. Transaction-Scoped
AuthenticationTransaction
Challenge Context
Continuation Context
Temporary Evidence
 46. Job-Scoped
Operational delegation
Tenant context
Incident processing context
 47. Worker Reset
Al terminar request:
AuthenticationContext
Identity Context
Session Context
Tenant Context
Realm Context
Risk Context
Temporary Evidence
Challenge Context
deberán quedar inaccesibles.
 48. Secret Lifetime
Temporary secrets deben vivir el mínimo tiempo posible.
 49. No Secret Properties in Long-Lived Services
Incorrecto:
final class AuthenticationManager
{
    private string $currentPassword;
}
 50. Fiber Isolation
Context storage deberá ser fiber-aware.
 51. Concurrency
Dos requests concurrentes en un worker:
Request A → Alice
Request B → Bob
jamás deberán compartir:
identity
session
tenant
realm
challenge
 52. Default Runtime Guard
Development/testing podrá detectar acceso a Authentication context fuera de scope.
 53. Example
AuthenticationContextNotAvailableOutsideRequestScope
 54. CLI Authentication Context
CLI no debe fingir browser Authentication.
Operational commands utilizan AuthenticationOperatorContext.
 55. Queue Authentication Context
Queue jobs no restauran automáticamente user session.
 56. Original Human Actor
Cuando sea necesario:
initiator identity
viaja como metadata/delegation segura, no como reusable bearer credential.
 57. Configuration Cache
voltstack auth:cache
puede compilar:
authenticators
policies
realms
routes
operations
method definitions
 58. Cache Contains No Secrets
Crítico.
 59. Key IDs Are Okay
Puede contener:
key reference
provider reference
pero no private key.
 60. Config Validation Command
voltstack auth:config:validate
 61. Security Audit Command
voltstack auth:security:check
 62. Example Findings
[CRITICAL] Admin realm allows remember-me
[ERROR] WebAuthn origin uses HTTP in production
[ERROR] OAuth provider has PKCE disabled
[WARN] Password security profile is deprecated
[WARN] 428 identities have no recovery method
[INFO] 82% of administrators have passkeys
 63. Configuration Linter
Debe detectar anti-patterns antes de producción.
 64. Install Experience
Ideal:
composer create-project voltstack/app
y Authentication Core ya está disponible.
 65. Optional Setup
voltstack auth:install
puede:
publish config
install migrations
register optional routes
create security profile
validate environment
 66. Install Must Not Create Default Admin Password
Nunca:
<admin@example.com>
password
 67. Admin Bootstrap
Si se necesita primer administrador:
voltstack auth:admin:create
deberá utilizar un ceremony seguro.
 68. Preferred First Admin
Puede:
create identity
      ↓
issue one-time enrollment invitation
      ↓
administrator establishes passkey/MFA
      ↓
activate admin identity
 69. Avoid Password in Shell History
CLI no deberá recomendar:
--password=Secret123
 70. Interactive Secret Input
Si una contraseña debe introducirse:
hidden terminal input
y nunca command history.
 71. Better
Permitir passkey/federated/bootstrap token enrollment.
 72. Bootstrap Token
Si se utiliza:
high entropy
short TTL
single use
purpose-bound
admin-enrollment-only
hashed at rest
 73. Default Data Seeder
No deberá generar credenciales production conocidas.
 74. Development Seeder
Puede generar synthetic accounts sólo en:
local
testing
con guard explícito.
 75. Production Guard
AUTH_DEV_SEEDER_FORBIDDEN_IN_PRODUCTION
 76. Default Failure Taxonomy
La reference implementation deberá mapear errores a códigos estables.
Ejemplos:
AUTHENTICATION_REQUIRED
AUTHENTICATION_FAILED
AUTHENTICATION_METHOD_UNAVAILABLE
AUTHENTICATION_POLICY_UNSATISFIABLE
AUTHENTICATION_CHALLENGE_REQUIRED
AUTHENTICATION_REAUTHENTICATION_REQUIRED
AUTHENTICATION_STEP_UP_REQUIRED
AUTHENTICATION_ACCOUNT_RESTRICTED
AUTHENTICATION_SESSION_REVOKED
AUTHENTICATION_REPLAY_DETECTED
 77. Internal vs Public Failure
Interno:
PASSWORD_HASH_INVALID
Público:
AUTHENTICATION_FAILED
cuando sea necesario evitar enumeration.
 78. Reference Implementation Invariants
AUTH-REF-001
Todo contrato crítico posee una implementación default o queda explícitamente marcado como capability opcional.
AUTH-REF-002
El default path no requiere desactivar controles de seguridad.
AUTH-REF-003
No existen credenciales default conocidas.
AUTH-REF-004
No existen secret keys hard-coded.
AUTH-REF-005
No existe Authentication state mutable global.
 79. Password Invariants
AUTH-REF-PASSWORD-001
Passwords nunca se registran.
AUTH-REF-PASSWORD-002
Argon2id es el default cuando el runtime lo soporta.
AUTH-REF-PASSWORD-003
Legacy hashes sólo se crean si policy explícita lo permite.
AUTH-REF-PASSWORD-004
Rehash ocurre después de verificación válida.
AUTH-REF-PASSWORD-005
Password normalization peligrosa está prohibida.
 80. Session Invariants
AUTH-REF-SESSION-001
Session ID rota después del login.
AUTH-REF-SESSION-002
Cookies browser son HttpOnly por defecto.
AUTH-REF-SESSION-003
Producción HTTPS utiliza Secure cookies.
AUTH-REF-SESSION-004
Remember-me no produce privileged assurance.
AUTH-REF-SESSION-005
Admin remember-me está deshabilitado por defecto.
 81. Transaction Invariants
AUTH-REF-TX-001
Nonces son CSPRNG.
AUTH-REF-TX-002
Security artifacts tienen TTL.
AUTH-REF-TX-003
Replay-sensitive artifacts son single-use.
AUTH-REF-TX-004
OAuth state y OIDC nonce son conceptos distintos.
AUTH-REF-TX-005
PKCE utiliza S256 por defecto.
 82. MFA Invariants
AUTH-REF-MFA-001
TOTP secret nunca se registra.
AUTH-REF-MFA-002
Un TOTP no verificado no se activa.
AUTH-REF-MFA-003
Recovery codes se almacenan como verifier/hash.
AUTH-REF-MFA-004
Recovery code consumption es atómico.
AUTH-REF-MFA-005
SMS no se clasifica como phishing-resistant.
 83. Passkey Invariants
AUTH-REF-PASSKEY-001
RP ID es explícito.
AUTH-REF-PASSKEY-002
Origins son validados.
AUTH-REF-PASSKEY-003
Private key nunca llega al servidor.
AUTH-REF-PASSKEY-004
Challenges son transaction-bound.
AUTH-REF-PASSKEY-005
User verification requirements provienen de policy.
 84. Federation Invariants
AUTH-REF-OIDC-001
State obligatorio.
AUTH-REF-OIDC-002
PKCE obligatorio por default.
AUTH-REF-OIDC-003
Issuer validado.
AUTH-REF-OIDC-004
Audience validada.
AUTH-REF-OIDC-005
Email no auto-vincula identities.
 85. Runtime Invariants
AUTH-REF-RUNTIME-001
Current Identity es request/fiber scoped.
AUTH-REF-RUNTIME-002
Current Tenant es request/fiber scoped.
AUTH-REF-RUNTIME-003
Current Realm es request/fiber scoped.
AUTH-REF-RUNTIME-004
Workers limpian contexto después del request.
AUTH-REF-RUNTIME-005
Queue jobs no heredan browser sessions.
 86. Operational Invariants
AUTH-REF-OPS-001
CLI usa el mismo Core.
AUTH-REF-OPS-002
Admin UI usa el mismo Core.
AUTH-REF-OPS-003
No existe bypass administrativo implícito.
AUTH-REF-OPS-004
Secrets no aparecen en diagnostics.
AUTH-REF-OPS-005
Critical operations son auditables.
 87. Anti-Pattern — Minimal Core, Security as Plugins
No diseñar:
VoltStack Auth Core
    =
email + password + session
y obligar a instalar paquetes externos para tener:
rate limiting
CSRF
session rotation
audit
Estos controles básicos deben ser parte del sistema.
 88. Anti-Pattern — One Giant Auth Service
class AuthService
{
    login();
    logout();
    hashPassword();
    validateJwt();
    verifyTotp();
    verifyWebAuthn();
    sendEmail();
    rotateKeys();
}
Prohibido arquitectónicamente.
 89. Anti-Pattern — User as Entire Security Model
users
 ├── password
 ├── remember_token
 ├── google_id
 ├── totp_secret
 ├── passkey
 ├── api_token
 ├── device_id
 └── is_locked
No representa adecuadamente el dominio.
 90. Anti-Pattern — Session Serialized User
No almacenar una copia completa del User como authoritative Authentication state durante semanas.
 91. Anti-Pattern — is_admin
if ($user->is_admin) {
    $assurance = HIGH;
}
Incorrecto.
 92. Anti-Pattern — Fallback Security Downgrade
Passkey failed
   ↓
SMS
sin policy.
 93. Anti-Pattern — Universal APP_KEY
No reutilizar una sola clave para:
sessions
OAuth state
webhooks
machine tokens
recovery
sin purpose separation.
 94. Anti-Pattern — Production In-Memory Store
No:
ArraySessionRepository
ArrayReplayStore
ArrayTransactionRepository
en producción distribuida.
 95. Anti-Pattern — Auto-Link by Email
Especialmente peligroso con múltiples providers.
 96. Anti-Pattern — Hidden Security Defaults
Todos los defaults importantes deben ser:
documented
inspectable
versioned
 97. Anti-Pattern — Test Hasher in Production
El framework deberá detectarlo.
 98. Anti-Pattern — FrankenPHP Static User
Auth::$user
prohibido.
 99. Anti-Pattern — Security Through Frontend
if (user.assurance === 'high') {
   allowDeleteTenant();
}
Frontend sólo controla UX.
100. Suggested Complete Directory Structure
src/
└── Quantum/
    └── Auth/
        ├── Contracts/
        ├── Core/
        ├── Context/
        ├── Identity/
        ├── Credential/
        ├── Method/
        ├── Authenticator/
        │   ├── Password/
        │   ├── Session/
        │   ├── Bearer/
        │   ├── Passkey/
        │   ├── Totp/
        │   ├── Federation/
        │   └── Machine/
        ├── Session/
        ├── RememberMe/
        ├── Policy/
        ├── Assurance/
        ├── Challenge/
        ├── Flow/
        ├── Transaction/
        ├── TransactionSecurity/
        ├── Mfa/
        ├── Passkey/
        ├── Federation/
        ├── Recovery/
        ├── Device/
        ├── Risk/
        ├── SecurityIncident/
        ├── IdentityLifecycle/
        ├── Privacy/
        ├── Communication/
        ├── Background/
        ├── Rate/
        ├── Migration/
        ├── Operations/
        ├── Machine/
        ├── Crypto/
        ├── Audit/
        ├── Events/
        ├── Telemetry/
        ├── Middleware/
        ├── Attributes/
        ├── Http/
        ├── Spa/
        ├── Console/
        ├── Configuration/
        ├── Compiler/
        ├── Runtime/
        ├── Default/
        ├── Exceptions/
        └── Testing/
101. Default/
No deberá convertirse en un monolito.
Puede contener factories/composition:
Default/
├── DefaultAuthenticationManager.php
├── DefaultAuthenticationStackFactory.php
├── DefaultAuthenticationSecurityProfile.php
├── DefaultAuthenticationBindings.php
└── DefaultAuthenticationConfiguration.php
Las implementaciones específicas viven en sus dominios correspondientes.
102. Facade
src/
└── Facades/
    └── Auth.php
103. Support
src/
└── Support/
    └── Authentication/
para componentes transversales realmente generales.
104. Testing
src/
└── Testing/
    └── Authentication/
o integración equivalente con Quantum\Auth\Testing.
105. Framework Integration Map
                       VOLTSTACK
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
     Routing          Controllers          HTTP
        │                  │                  │
        └────────────┬─────┴──────────────┬──┘
                     ▼                    ▼
               Authentication       Middleware
                     │
        ┌────────────┼──────────────┐
        ▼            ▼              ▼
   Authorization   Events       Telemetry
        │            │              │
        ├────────────┼──────────────┤
        ▼            ▼              ▼
    Database       Queue          Cache
        │            │              │
        ├────────────┼──────────────┤
        ▼            ▼              ▼
    Scheduler     Crypto         Runtime
                                   │
                                   ▼
                              FrankenPHP
106. Reference Request Flow
Una request autenticada normal:
HTTP Request
    ↓
Trusted Proxy Processing
    ↓
Tenant Resolution
    ↓
Realm Resolution
    ↓
Authentication Middleware
    ↓
Session Credential Extraction
    ↓
Session Authenticator
    ↓
Session Repository
    ↓
Identity Provider
    ↓
Eligibility Resolver
    ↓
Authentication Context
    ↓
Assurance Resolver
    ↓
Authentication Requirement
    ↓
Satisfied?
 ┌──┴───┐
 │      │
YES     NO
 │       ↓
 │   Challenge Negotiator
 │       ↓
 │   Authentication Flow
 │
 ▼
Authorization
    ↓
Controller
107. Password Login Flow
POST /login
     ↓
CSRF
     ↓
Rate Governance
     ↓
Authentication Transaction
     ↓
Password Authenticator
     ↓
Identity Resolution
     ↓
Password Verification
     ↓
Eligibility
     ↓
Risk
     ↓
Policy
     ↓
MFA Required?
 ┌───┴────┐
NO       YES
 │         ↓
 │      Challenge
 │         ↓
 │      TOTP / Passkey
 │         ↓
 └─────────┤
           ▼
Authentication Context
           ↓
Session Rotation
           ↓
Session Creation
           ↓
Events / Audit
           ↓
Continuation
108. Passkey Login Flow
Login Intent
    ↓
Transaction
    ↓
WebAuthn Challenge
    ↓
Browser Authenticator
    ↓
Assertion
    ↓
Origin Validation
    ↓
RP ID Validation
    ↓
Challenge Validation
    ↓
Signature Verification
    ↓
Credential Status
    ↓
Identity
    ↓
Assurance
    ↓
Policy
    ↓
Session
109. OIDC Flow
Login
  ↓
Authentication Transaction
  ↓
State + Nonce + PKCE
  ↓
OIDC Provider
  ↓
Callback
  ↓
State Validation
  ↓
Code Exchange
  ↓
ID Token Validation
  ↓
Issuer / Audience / Nonce
  ↓
External Identity Mapping
  ↓
Local Identity
  ↓
Policy / Assurance
  ↓
Session
110. Sensitive Operation Flow
Authenticated Session
       ↓
Delete Tenant
       ↓
Authorization
       ↓
Authentication Requirement
       ↓
Current Assurance Insufficient
       ↓
Step-Up Transaction
       ↓
Passkey Challenge
       ↓
Verified Evidence
       ↓
HIGH + FRESH Context
       ↓
Operation-Bound Proof
       ↓
Authorization Revalidation
       ↓
Lifecycle / Risk Revalidation
       ↓
Delete Tenant
111. Security Incident Flow
Authentication Event
      ↓
Signal
      ↓
Risk / Detection
      ↓
Finding
      ↓
Incident
      ↓
Response Policy
      ↓
Session Revocation
Identity Restriction
Credential Revocation
Security Epoch
      ↓
Notification
      ↓
Security Center
      ↓
Recovery / Review
112. Reference Implementation Philosophy
La implementación default deberá ser:
Opinionated enough to be safe
        +
Modular enough to be replaced
113. Developer Experience Target
Para una aplicación sencilla:
Route::get('/dashboard', DashboardController::class)
    ->middleware('auth');
y:
$user = Auth::user();
debe ser suficiente.
114. Enterprise Target
La misma arquitectura deberá permitir:
multiple realms
multiple tenants
passkeys
enterprise OIDC
machine identities
risk-based authentication
distributed sessions
KMS
HSM
security incidents
SOC integration
break-glass
multi-region deployment
sin reemplazar el Authentication Core.
115. Acceptance Criteria
El documento 49 se considerará implementado cuando VoltStack proporcione una referencia funcional que incluya:

- DefaultAuthenticationManager;
- authenticator resolver;
- password authenticator;
- Argon2id password hasher;
- progressive hash upgrade;
- session authenticator;
- secure session rotation;
- server-side session repository abstraction;
- secure browser cookie defaults;
- logout y global logout;
- remember-me opcional;
- bearer authentication abstraction;
- opaque API credentials;
- JWT validation cuando se habilite;
- stable Identity IDs;
- database Identity provider;
- immutable Authentication Context;
- request/fiber scoped context accessor;
- default Authentication Policy Engine;
- default Assurance Resolver;
- default Challenge Negotiator;
- default Flow Engine;
- transaction repository;
- CSPRNG nonce generation;
- atomic replay protection;
- CSRF integration;
- protected OAuth/continuation state;
- purpose-separated cryptographic keys;
- TOTP;
- hashed recovery codes;
- WebAuthn/passkeys;
- strict RP/origin validation;
- OIDC;
- state + PKCE + nonce;
- explicit external identity linking;
- recovery flows;
- optional trusted-device system;
- deterministic baseline risk engine;
- baseline incident detection;
- security notifications;
- rate/abuse governance;
- maintenance tasks;
- queue integration;
- database migrations;
- middleware;
- route metadata;
- attributes;
- controller injection;
- Auth facade;
- helpers;
- SPA Authentication protocol;
- optional UI adapters;
- Security Center backend;
- machine Authentication;
- multi-tenant contexts;
- realm support;
- operational tooling;
- diagnostics;
- health checks;
- testing utilities;
- telemetry;
- audit;
- configuration compiler;
- security-profile versioning;
- progressive security upgrades;
- plugin validation;
- FrankenPHP-safe service lifetimes;
- worker reset;
- fiber isolation;
- production configuration guards.
  1. Regla final de implementación
La instalación básica de VoltStack no deberá producir:
Authentication Skeleton
que el desarrollador tenga que convertir posteriormente en un sistema seguro.
Deberá producir:
Secure Authentication Foundation
sobre la cual pueda construirse.
  2. Relación entre arquitectura y referencia
Los documentos anteriores definen:
WHAT Authentication must guarantee
El documento 49 define:
WHAT VoltStack ships by default
Y el documento 50 definirá:
HOW EVERYTHING FITS TOGETHER
  3. Arquitectura resultante hasta 49
                         VOLTSTACK AUTHENTICATION
                                  │
       ┌──────────────────────────┼───────────────────────────┐
       │                          │                           │
       ▼                          ▼                           ▼
  Architecture                Runtime                  Reference
   Contracts                 Security                Implementation
       │                          │                           │
       └──────────────────────────┼───────────────────────────┘
                                  ▼
                        Authentication Platform
                                  │
             ┌────────────────────┼─────────────────────┐
             ▼                    ▼                     ▼
         Human Auth          Machine Auth          Operations
             │                    │                     │
             ├────────────────────┼─────────────────────┤
             ▼                    ▼                     ▼
         Policies             Assurance             Incidents
             │                    │                     │
             ├────────────────────┼─────────────────────┤
             ▼                    ▼                     ▼
         Sessions            Credentials            Security
             │                    │                 Governance
             └────────────────────┼─────────────────────┘
                                  ▼
                         VoltStack Runtime
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
                 HTTP/SPA      Queue/CLI    FrankenPHP
  4. Decisión arquitectónica
VoltStack utilizará el patrón:
Contract
   ↓
Default Secure Implementation
   ↓
Application Override
en lugar de:
Framework Hard-Coding
o:
Empty Contract With Everything Left To User
Esto ofrece un equilibrio entre:
Convention
Security
Extensibility
Developer Experience
Enterprise Architecture
  5. Secure-by-Default Rule
Una configuración default deberá preferir:
Reject
antes que:
Silently Downgrade Security
cuando una garantía requerida no pueda cumplirse.
  6. Fail-Secure Matrix
Ejemplos:
Required passkey unavailable
    → do not silently use SMS

Replay store unavailable for critical flow
    → do not ignore replay protection

Unknown OIDC signing key
    → do not trust token

Invalid tenant binding
    → reject

Expired reauthentication proof
    → require new proof

Stale privileged context
    → reauthenticate

Unknown credential status
    → do not assume active

Invalid policy compilation
    → do not silently remove requirement

## 406. Reference Implementation Boundary

La implementación de referencia no significa que VoltStack deba reinventar primitivas criptográficas.
VoltStack podrá utilizar bibliotecas maduras para:
WebAuthn
JWT/JWS/JWE
OIDC
TOTP
cryptographic primitives
pero dichas bibliotecas estarán detrás de contratos VoltStack.

## 407. Security Library Principle

VoltStack owns:
architecture
policy
state
lifecycle
binding
integration
security invariants

Cryptographic library owns:
correct primitive implementation

## 408. No Custom Crypto

VoltStack nunca deberá implementar informalmente:
hash('sha256', $secret . $key);
como sustituto improvisado de protocolos criptográficos establecidos.

## 409. Framework Integration Principle

Authentication no será un paquete aislado del resto del framework.
Será un Quantum profundamente integrado con:
Container
Config
Bootstrap
HTTP
HttpKernel
Routing
Middleware
Controllers
Database
Cache
Events
Queue
Scheduler
Telemetry
Authorization
Testing
Runtime
FrankenPHP
SPA Runtime
pero cada integración deberá pasar por contratos definidos.

## 410. Resultado final del documento 49

Con este documento, VoltStack ya dispone conceptualmente de:
Authentication Architecture
        +
Authentication Security Model
        +
Authentication Runtime Model
        +
Authentication Governance
        +
Authentication Operational Model
        +
Authentication Reference Implementation
Por tanto, el sistema ya está preparado para su cierre arquitectónico.

## 411. Siguiente y último documento

El siguiente documento debe ser:
50_AUTHENTICATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md
Este será el documento final del sistema Authentication de VoltStack.
No deberá añadir simplemente otro subsistema. Su función será integrar formalmente los 49 documentos anteriores en una sola arquitectura.
Deberá definir, entre otros:
Complete Authentication Domain Map
Complete Component Map
Subsystem Dependency Graph
Allowed Dependency Directions
Authentication Kernel
Full Request Lifecycle
Login Lifecycle
Session Restoration Lifecycle
MFA / Step-Up Lifecycle
Passkey Lifecycle
Federation Lifecycle
Recovery Lifecycle
Machine Authentication Lifecycle
Sensitive Operation Lifecycle
Security Incident Lifecycle
Identity Lifecycle
Credential Lifecycle
Administrative Lifecycle
Background Processing Lifecycle
Distributed Authentication Architecture
Multi-Tenant Architecture
Realm Architecture
Authentication ↔ Authorization Boundary
Authentication ↔ Routing
Authentication ↔ Controllers
Authentication ↔ Database
Authentication ↔ Cache
Authentication ↔ Events
Authentication ↔ Queue
Authentication ↔ Telemetry
Authentication ↔ SPA
Authentication ↔ FrankenPHP
Control Plane vs Data Plane
Compile-Time vs Runtime
Request-Scoped vs Worker-Scoped State
Security Trust Boundaries
Complete Failure Model
Complete Event Model
Complete Security Invariants
Final Namespace Architecture
Final Directory Structure
Final Dependency Rules
Deployment Topologies
Single-Node Deployment
Distributed Deployment
Multi-Region Deployment
Enterprise Deployment
Performance Model
Security Model
Testing Architecture
Extension Model
Backward Compatibility Strategy
Final Acceptance Criteria
Y deberá cerrar la arquitectura con una visión unificada:
                         VOLTSTACK
                            │
                            ▼
                AUTHENTICATION SYSTEM
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       Identity         Authentication      Security
                         Runtime            Governance
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                    Trusted Principal
                            │
                            ▼
                      Authorization
                            │
                            ▼
                       Application
Con 50 quedará formalmente completado el diseño del Authentication System de VoltStack.
