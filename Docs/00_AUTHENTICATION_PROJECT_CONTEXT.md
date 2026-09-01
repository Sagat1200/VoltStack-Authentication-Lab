# VoltStack Authentication System

## 00 — Authentication Project Context

- **Archivo:** `00_AUTHENTICATION_PROJECT_CONTEXT.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica base  
- **Versión objetivo:** VoltStack 1.x+

---

## 1. Introducción

El **VoltStack Authentication System** es el subsistema responsable de establecer, verificar, mantener y representar la identidad autenticada de usuarios, servicios, dispositivos y otras entidades capaces de interactuar con una aplicación VoltStack.

Authentication responde fundamentalmente a la pregunta:

> **¿Quién está realizando esta operación y mediante qué evidencia podemos confiar en esa identidad?**

El sistema debe proporcionar una arquitectura moderna que combine:

- la ergonomía y productividad del sistema de autenticación de Laravel;
- la separación de responsabilidades y extensibilidad del componente Security de Symfony;
- capacidades modernas de identidad diseñadas específicamente para VoltStack;
- compatibilidad con aplicaciones tradicionales, SPA, APIs y servicios;
- ejecución eficiente sobre PHP tradicional y runtimes persistentes como FrankenPHP;
- soporte para arquitecturas multi-tenant;
- autenticación humana y machine-to-machine;
- mecanismos passwordless;
- autenticación multifactor;
- identidad federada;
- evaluación contextual y adaptativa.

Authentication será un subsistema independiente de Authorization.

La frontera fundamental será:

```text
Authentication
      │
      │ establishes
      ▼
Authenticated Identity
      │
      ▼
AuthenticationContext
      │
      │ consumed by
      ▼
Authorization
      │
      ▼
ALLOW / DENY
```

Authentication determina la identidad.

Authorization determina las capacidades de esa identidad.

---

## 2. Objetivo del sistema

El objetivo principal es proporcionar una infraestructura universal de autenticación que pueda ser utilizada consistentemente por todos los componentes de VoltStack.

El sistema deberá permitir:

```text
Web Applications
SPA Applications
Mobile APIs
REST APIs
GraphQL APIs
CLI
Workers
Queues
WebSockets
SSE
Microservices
Internal Services
Scheduled Tasks
Enterprise Applications
Multi-Tenant Applications
Machine-to-Machine Communication
```

sin que cada entorno tenga que implementar un modelo de autenticación independiente.

La arquitectura deberá ser suficientemente sencilla para permitir:

```php
Auth::attempt($credentials);

Auth::check();

Auth::user();

Auth::id();

Auth::logout();
```

pero internamente deberá soportar pipelines considerablemente más sofisticados.

---

## 3. Principio fundamental

La autenticación no debe estar acoplada exclusivamente al concepto de `User`.

VoltStack utilizará el concepto general de:

```php
IdentityInterface
```

Una identidad podrá representar:

```text
Human User
Administrator
Customer
Employee
Service Account
Machine
Application
API Client
Worker
Device
Federated Identity
Temporary Identity
```

Por lo tanto:

```text
User
```

será únicamente una posible implementación de:

```text
Identity
```

Esto permite que el sistema pueda evolucionar sin depender del modelo tradicional:

```text
users table
      +
email
      +
password
```

---

## 4. Alcance

El Authentication System cubrirá todo el ciclo de vida relacionado con establecimiento y mantenimiento de identidad.

Incluye:

### 4.1 Identity

Responsable de representar quién está autenticado.

```text
Identity
Identity Identifier
Identity Type
Identity Provider
Identity Resolver
Identity Loader
Identity Metadata
```

---

### 4.2 Credentials

Representación y verificación de evidencia secreta o criptográfica.

```text
Password
Secret
API Key
Access Token
Certificate
Signed Assertion
Passkey
Security Key
Recovery Credential
```

---

### 4.3 Authenticators

Los Authenticators implementarán mecanismos concretos de autenticación.

Ejemplos:

```text
PasswordAuthenticator
SessionAuthenticator
AccessTokenAuthenticator
ApiKeyAuthenticator
JwtAuthenticator
PasskeyAuthenticator
MagicLinkAuthenticator
OAuthAuthenticator
OidcAuthenticator
SamlAuthenticator
ClientCertificateAuthenticator
ServiceAccountAuthenticator
```

---

## 5. Authentication Pipeline

El sistema utilizará un pipeline explícito.

Modelo conceptual:

```text
Request
   │
   ▼
Authentication Firewall
   │
   ▼
Authenticator Resolver
   │
   ▼
Authenticator
   │
   ▼
Authentication Passport
   │
   ▼
Identity Resolution
   │
   ▼
Credential Verification
   │
   ▼
Authentication Factors
   │
   ▼
Authentication Policy
   │
   ▼
Risk Evaluation
   │
   ▼
Authentication Decision
   │
   ▼
Authentication Context
   │
   ▼
Session / Token / Security Context
```

Cada etapa deberá ser reemplazable o extensible mediante contratos.

---

## 6. Authentication Firewall

Inspirado conceptualmente por Symfony, VoltStack introducirá un:

```text
AuthenticationFirewall
```

Su función será determinar qué configuración de autenticación corresponde a una request o contexto de ejecución.

Podrá considerar:

```text
route
path
host
domain
protocol
tenant
request type
API scope
runtime
transport
application context
```

Ejemplo conceptual:

```text
/admin/*
      ↓
admin firewall

/api/*
      ↓
api firewall

/*
      ↓
web firewall
```

Cada firewall podrá definir:

```text
authenticators
identity provider
stateful/stateless
session strategy
failure handler
success handler
authentication policy
MFA requirements
tenant requirements
```

---

## 7. Authenticator

Un Authenticator será una implementación de:

```php
AuthenticatorInterface
```

Conceptualmente:

```php
interface AuthenticatorInterface
{
    public function supports(AuthenticationRequest $request): bool;

    public function authenticate(
        AuthenticationRequest $request
    ): AuthenticationPassport;
}
```

El Authenticator no deberá convertirse en un objeto monolítico responsable de todo el proceso.

Su responsabilidad principal será:

```text
recognize authentication mechanism
            +
extract authentication evidence
            +
construct authentication passport
```

La verificación real podrá ser delegada a otros componentes.

---

## 8. Authentication Passport

VoltStack adoptará y extenderá el concepto de Passport.

Un Passport representa:

> La evidencia presentada y los requisitos necesarios para intentar establecer una identidad autenticada.

Modelo conceptual:

```text
AuthenticationPassport
│
├── IdentityEvidence
│
├── Credentials
│
├── Factors
│
├── Badges
│
├── Challenges
│
├── Metadata
└── AuthenticationRequirements
```

Un Passport todavía **no representa una autenticación exitosa**.

Representa un intento estructurado de autenticación.

---

## 9. Authentication Factors

VoltStack distinguirá explícitamente factores de autenticación.

Categorías:

```text
Knowledge
Possession
Inherence
Federated
Contextual
Device
Cryptographic
```

Ejemplos:

```text
Password
PIN
TOTP
Passkey
Security Key
Certificate
Biometric-backed credential
Recovery Code
External Identity Provider
Trusted Device
```

Esto permitirá construir MFA de forma nativa.

---

## 10. Authentication Assurance

No todas las autenticaciones proporcionan el mismo nivel de confianza.

VoltStack deberá representar explícitamente el nivel de assurance alcanzado.

Conceptualmente:

```text
AAL1
AAL2
AAL3
```

o mediante una representación extensible equivalente.

Ejemplo:

```text
Password
      ↓
AAL1

Password + TOTP
      ↓
AAL2

Hardware-backed Passkey
      ↓
AAL2/AAL3 según política y contexto
```

Authorization podrá utilizar posteriormente este nivel.

Ejemplo:

```text
view invoice
      ↓
AAL1 sufficient

change bank account
      ↓
AAL2 required
```

La decisión de acceso pertenece a Authorization.

El nivel de autenticación pertenece a Authentication.

---

## 11. Authentication Context

Una autenticación exitosa producirá un:

```php
AuthenticationContext
```

Este será uno de los objetos centrales del sistema.

Podrá contener:

```text
Identity
Identity Provider
Authentication Method
Authentication Factors
Authentication Time
Authentication Assurance Level
Tenant
Session
Device
Client
Network Context
Authentication Metadata
Risk Metadata
```

Ejemplo conceptual:

```text
AuthenticationContext

identity:
    user:483

provider:
    database

method:
    passkey

assurance:
    AAL2

tenant:
    tenant:82

device:
    device:153

authenticated_at:
    2026-08-19T20:30:00
```

Este contexto podrá ser consumido por:

```text
Authorization
Controllers
Middleware
Auditing
Observability
Application Services
Security Policies
```

---

## 12. Authentication Token

VoltStack distinguirá entre:

```text
AuthenticationContext
```

y:

```text
AuthenticationToken
```

El Context representa el estado lógico de autenticación.

El Token podrá representar una forma serializable, transportable o persistible de ese estado.

Podrán existir:

```text
SessionAuthenticationToken
BearerAuthenticationToken
JwtAuthenticationToken
ServiceAuthenticationToken
TemporaryAuthenticationToken
FederatedAuthenticationToken
```

No todos los Authentication Context deberán necesariamente estar respaldados por un token transportable.

---

## 13. Stateful Authentication

VoltStack soportará autenticación stateful como capacidad de primera clase.

Principalmente:

```text
Browser
    ↓
Session Cookie
    ↓
Session
    ↓
Authentication State
    ↓
AuthenticationContext
```

El sistema deberá manejar:

```text
session creation
session regeneration
session rotation
session expiration
session invalidation
session revocation
concurrent sessions
device sessions
idle timeout
absolute timeout
```

---

## 14. Stateless Authentication

También deberá soportarse autenticación completamente stateless.

Ejemplo:

```text
Request
   │
Authorization: Bearer ...
   │
   ▼
AccessTokenAuthenticator
   │
   ▼
Token Verification
   │
   ▼
Identity
   │
   ▼
AuthenticationContext
```

Casos principales:

```text
REST API
GraphQL API
Mobile Application
Microservices
Service-to-Service
CLI integrations
```

---

## 15. Hybrid Authentication

VoltStack permitirá combinar mecanismos stateful y stateless.

Ejemplo:

```text
SPA
 │
 ├── Session Cookie
 │
 ├── CSRF protection
 │
 └── API requests
```

o:

```text
Web Application
 │
 ├── Session authentication
 │
 └── Personal access token API
```

La arquitectura no deberá asumir que una aplicación utiliza exclusivamente un modelo.

---

## 16. Password Authentication

Las contraseñas continuarán siendo soportadas, pero serán tratadas como un mecanismo entre múltiples mecanismos posibles.

El sistema deberá proporcionar:

```text
PasswordHasher
PasswordVerifier
PasswordPolicy
PasswordRehash
PasswordMigration
CompromisedPasswordChecker
PasswordReset
PasswordHistory
```

Algoritmos modernos deberán ser configurables.

Preferencia inicial:

```text
Argon2id
```

con soporte de compatibilidad para:

```text
bcrypt
```

y migración progresiva de hashes.

---

## 17. Password Hash Migration

El sistema deberá permitir migrar hashes automáticamente.

Ejemplo:

```text
bcrypt hash
     │
successful login
     ▼
needsRehash()
     │
     ▼
Argon2id
     │
     ▼
new hash persisted
```

Esto permitirá actualizar políticas criptográficas sin obligar a los usuarios a restablecer sus contraseñas.

---

## 18. Multi-Factor Authentication

MFA será una capacidad arquitectónica nativa.

No deberá depender de implementar lógica especial dentro de controllers.

Pipeline:

```text
Primary Authentication
        │
        ▼
Factor Requirements
        │
        ▼
Additional Challenge
        │
        ▼
Factor Verification
        │
        ▼
Authentication Assurance
        │
        ▼
AuthenticationContext
```

Factores inicialmente contemplados:

```text
TOTP
HOTP
Recovery Codes
WebAuthn
Passkeys
Security Keys
Email OTP
SMS OTP
Push
Custom Factor
```

Los factores débiles podrán ser deshabilitados mediante políticas de aplicación.

---

## 19. Passkeys y WebAuthn

Passkeys deberán ser ciudadanos de primera clase dentro del sistema.

VoltStack deberá permitir:

```text
password authentication
password + MFA
passwordless passkey
passkey + step-up
hardware security key
```

sin alterar el modelo fundamental de Authentication.

---

## 20. Passwordless Authentication

El framework deberá soportar mecanismos passwordless.

Ejemplos:

```text
Passkey
Magic Link
One-Time Login
Federated Login
Hardware Credential
```

La arquitectura no deberá considerar `password` como requisito obligatorio de una Identity.

---

## 21. Step-Up Authentication

Una sesión autenticada podrá requerir autenticación adicional para determinadas operaciones.

Ejemplo:

```text
User authenticated
      │
      │ AAL1
      ▼
Change payment information
      │
      ▼
Step-up required
      │
      ▼
Passkey / TOTP
      │
      ▼
AAL2
      │
      ▼
continue
```

Authentication será responsable de realizar el step-up.

Authorization será responsable de determinar cuándo un nivel superior es necesario.

---

## 22. Federated Identity

La arquitectura deberá permitir integrar proveedores externos.

Protocolos contemplados:

```text
OAuth 2.x
OpenID Connect
SAML
LDAP
Active Directory
Enterprise Identity Providers
Social Identity Providers
```

Los protocolos específicos podrán vivir en paquetes Quantum separados.

Por ejemplo:

```text
Quantum/Auth
Quantum/OAuth
Quantum/Oidc
Quantum/Saml
Quantum/Ldap
Quantum/WebAuthn
```

Core deberá proporcionar los contratos necesarios sin incorporar obligatoriamente todas las implementaciones.

---

## 23. Machine Identity

Authentication no estará limitado a humanos.

VoltStack deberá soportar:

```text
Service Account
Machine Identity
Workload Identity
Application Identity
API Client
Worker Identity
Device Identity
```

Esto permitirá autenticar comunicaciones como:

```text
Service A
   │
   ▼
Service B
```

sin crear usuarios ficticios para representar procesos automáticos.

---

## 24. Device Identity

Los dispositivos podrán formar parte del Authentication Context.

Conceptualmente:

```text
Identity
   +
Device
   +
Session
   +
Authentication Method
```

Esto permitirá implementar:

```text
trusted devices
known devices
new device detection
device revocation
session-per-device
device-bound authentication
risk evaluation
```

---

## 25. Multi-Tenant Authentication

Multi-tenancy será contemplado desde la arquitectura inicial.

Authentication podrá operar bajo:

```text
Global Identity
Tenant Identity
Tenant-Scoped Identity
Federated Tenant Identity
```

Ejemplo:

```text
Request
   ↓
Tenant Resolution
   ↓
Authentication Firewall
   ↓
Tenant Identity Provider
   ↓
Authentication
```

El sistema deberá prevenir:

```text
cross-tenant identity leakage
cross-tenant session reuse
incorrect provider resolution
tenant context confusion
```

---

## 26. Adaptive Authentication

VoltStack permitirá modificar los requisitos de autenticación según contexto y riesgo.

Factores potenciales:

```text
new device
unusual IP
location change
impossible travel
credential anomalies
session anomalies
authentication failures
sensitive operation
device trust
client characteristics
```

Ejemplo:

```text
Normal Login
    ↓
Risk Low
    ↓
Password sufficient

Unusual Login
    ↓
Risk Elevated
    ↓
MFA required
```

El Risk Engine deberá ser extensible.

---

## 27. Authentication Challenges

El sistema representará explícitamente desafíos de autenticación.

Ejemplos:

```text
EnterPasswordChallenge
TotpChallenge
PasskeyChallenge
EmailOtpChallenge
RecoveryCodeChallenge
ReauthenticationChallenge
```

Esto será especialmente importante para SPA.

El backend podrá comunicar:

```text
AUTHENTICATION_REQUIRED
```

o:

```text
AUTHENTICATION_CHALLENGE_REQUIRED
```

sin depender de redirects HTML.

---

## 28. SPA Authentication

El sistema estará diseñado para integrarse directamente con el runtime SPA de VoltStack.

Ejemplo:

```text
SPA Request
     │
     ▼
Authentication Pipeline
     │
     ├── authenticated
     │
     ├── unauthenticated
     │
     ├── challenge required
     │
     └── step-up required
```

El protocolo podrá transportar estados estructurados.

Ejemplo conceptual:

```json
{
    "authentication": {
        "status": "challenge_required",
        "challenge": "totp",
        "transaction": "..."
    }
}
```

El frontend no deberá inferir estados de autenticación a partir únicamente de redirects.

---

## 29. Authentication Transactions

Procesos de autenticación multi-step deberán representarse mediante:

```text
AuthenticationTransaction
```

Ejemplo:

```text
Login Started
      ↓
Password Verified
      ↓
MFA Required
      ↓
TOTP Verified
      ↓
Authentication Completed
```

La transaction permitirá conservar de forma segura el estado intermedio.

Esto será utilizado por:

```text
MFA
OAuth
OIDC
Passkeys
Magic Links
Account Recovery
Step-Up
```

---

## 30. Events

Authentication emitirá eventos durante su ciclo de vida.

Ejemplos:

```text
AuthenticationStarted
IdentityResolved
CredentialsVerified
AuthenticationChallengeCreated
AuthenticationFactorVerified
AuthenticationSucceeded
AuthenticationFailed
AuthenticationContextCreated
SessionCreated
SessionRotated
SessionRevoked
LogoutCompleted
RiskDetected
StepUpRequired
```

Los eventos deberán integrarse con el Event System de VoltStack.

---

## 31. Observabilidad

Authentication deberá proporcionar observabilidad nativa.

Se deberán poder registrar:

```text
authentication attempts
successes
failures
latency
provider latency
credential verification time
MFA challenges
session creation
token validation
risk evaluation
```

Deberá existir soporte para:

```text
metrics
structured logs
tracing
security events
audit events
```

---

## 32. Auditoría

Los eventos sensibles deberán poder generar registros de auditoría.

Ejemplo:

```text
AUTH_LOGIN_SUCCESS
AUTH_LOGIN_FAILURE
AUTH_MFA_SUCCESS
AUTH_MFA_FAILURE
AUTH_SESSION_CREATED
AUTH_SESSION_REVOKED
AUTH_PASSWORD_CHANGED
AUTH_RECOVERY_STARTED
AUTH_RECOVERY_COMPLETED
AUTH_DEVICE_REGISTERED
AUTH_DEVICE_REVOKED
```

Nunca deberán registrarse:

```text
raw passwords
raw access tokens
private keys
recovery secrets
MFA secrets
sensitive credential material
```

---

## 33. Security by Default

Authentication seguirá una política:

```text
Secure by Default
Fail Closed
Least Privilege
Explicit Trust
Minimal Credential Exposure
Defense in Depth
```

Ante estados ambiguos:

```text
UNKNOWN
INVALID
EXPIRED
UNVERIFIED
INCOMPLETE
```

el resultado por defecto deberá ser:

```text
UNAUTHENTICATED
```

Nunca deberá inferirse autenticación válida ante un error interno.

---

## 34. Timing Attacks

La implementación deberá reducir filtraciones temporales durante verificaciones sensibles.

Particularmente:

```text
password verification
token comparison
API keys
recovery codes
signatures
authentication identifiers
```

Las comparaciones criptográficas deberán utilizar operaciones adecuadas para evitar timing attacks cuando corresponda.

---

## 35. User Enumeration

Los flujos públicos deberán minimizar la posibilidad de descubrir identidades registradas.

Por ejemplo:

```text
Incorrect:

"Email does not exist."

Better:

"Unable to authenticate with the supplied credentials."
```

Esto aplica especialmente a:

```text
login
password reset
magic links
account recovery
verification
```

---

## 36. Rate Limiting

Authentication deberá integrarse con el sistema de Rate Limiting.

Podrán establecerse límites por:

```text
IP
Identity
Credential Identifier
Device
Tenant
Route
Authenticator
Authentication Transaction
```

La estrategia deberá soportar:

```text
fixed window
sliding window
token bucket
progressive delay
adaptive throttling
```

---

## 37. Brute Force Protection

El sistema deberá proporcionar infraestructura para detectar y responder a ataques de fuerza bruta.

Sin depender exclusivamente del bloqueo permanente de cuentas.

Podrán utilizarse:

```text
rate limiting
progressive delays
risk scoring
temporary challenges
MFA escalation
IP reputation
device signals
security telemetry
```

---

## 38. Authentication Failure

Los fallos deberán representarse mediante objetos estructurados.

Ejemplos internos:

```text
InvalidCredentials
IdentityNotFound
CredentialExpired
AuthenticationExpired
FactorRequired
FactorInvalid
TokenInvalid
TokenExpired
AccountUnavailable
ProviderUnavailable
RiskRejected
AuthenticationPolicyRejected
```

La respuesta externa no deberá necesariamente revelar la causa interna exacta.

---

## 39. Extensibilidad

El sistema deberá permitir registrar:

```text
Authenticators
Identity Providers
Credential Verifiers
Factors
Challenge Handlers
Token Types
Session Strategies
Risk Evaluators
Authentication Policies
Success Handlers
Failure Handlers
```

mediante contratos y registries.

No deberán requerirse modificaciones al Core para incorporar mecanismos nuevos.

---

## 40. Driver Model

Cuando resulte conveniente, VoltStack proporcionará un modelo familiar para desarrolladores Laravel:

```php
Auth::driver('session');

Auth::guard('web');
```

Sin embargo, `Guard` será principalmente una API de conveniencia.

Internamente:

```text
Guard
    ↓
Authentication Manager
    ↓
Firewall
    ↓
Authenticator Pipeline
```

Esto permitirá conservar ergonomía sin sacrificar arquitectura.

---

## 41. Facade

VoltStack proporcionará:

```php
VoltStack\Facades\Auth
```

API inicial esperada:

```php
Auth::check();

Auth::guest();

Auth::user();

Auth::identity();

Auth::id();

Auth::context();

Auth::attempt($credentials);

Auth::login($identity);

Auth::logout();

Auth::guard('web');

Auth::session();

Auth::assuranceLevel();
```

La Facade no contendrá lógica de seguridad.

Delegará siempre al Authentication System.

---

## 42. Contratos fundamentales

La arquitectura probablemente definirá contratos equivalentes a:

```text
AuthenticationManagerInterface
AuthenticatorInterface
IdentityInterface
IdentityProviderInterface
CredentialInterface
CredentialVerifierInterface
AuthenticationPassportInterface
AuthenticationContextInterface
AuthenticationTokenInterface
AuthenticationFactorInterface
AuthenticationPolicyInterface
AuthenticationFirewallInterface
AuthenticationContextStorageInterface
AuthenticationSessionInterface
RiskEvaluatorInterface
ChallengeInterface
```

Los contratos exactos serán definidos en documentos posteriores.

---

## 43. Estructura inicial

Estructura conceptual:

```text
src/
└── Quantum/
    └── Auth/
        ├── Contracts/
        ├── Authentication/
        ├── Authenticator/
        ├── Context/
        ├── Credentials/
        ├── Identity/
        ├── Providers/
        ├── Passport/
        ├── Factors/
        ├── Challenges/
        ├── Firewall/
        ├── Guards/
        ├── Sessions/
        ├── Tokens/
        ├── Password/
        ├── MFA/
        ├── Passkeys/
        ├── Devices/
        ├── Risk/
        ├── Policies/
        ├── Events/
        ├── Exceptions/
        ├── Middleware/
        ├── Observability/
        ├── Audit/
        └── Support/
```

La estructura definitiva será especificada posteriormente y no deberá considerarse congelada por este documento.

---

## 44. Integración con Container

Todos los componentes principales deberán resolverse mediante el Dependency Injection Container de VoltStack.

Ejemplo:

```text
AuthenticationManagerInterface
        ↓
AuthenticationManager
```

Los registries y providers podrán registrarse durante bootstrap.

Esto permitirá:

```text
dependency injection
testing
replacement
decorators
instrumentation
plugins
```

---

## 45. Integración con Routing

Routing podrá declarar requisitos relacionados con Authentication.

Ejemplo conceptual:

```php
Route::get('/profile', ProfileController::class)
    ->auth();
```

o:

```php
Route::post('/billing', BillingController::class)
    ->auth()
    ->assurance('AAL2');
```

Routing no ejecutará directamente la autenticación.

Declarará metadata consumida por los sistemas correspondientes.

---

## 46. Integración con Controllers

Controllers podrán acceder al contexto:

```php
Auth::identity();

Auth::context();
```

o mediante dependency injection:

```php
public function __invoke(
    AuthenticationContext $authentication
) {
    // ...
}
```

El Controller System no deberá implementar lógica de autenticación propia.

---

## 47. Integración con Authorization

Authentication producirá información.

Authorization la consumirá.

```text
AuthenticationContext
        │
        ▼
AuthorizationContext
        │
        ▼
Authorization Engine
```

Por ejemplo:

```text
Authentication:

identity = user:100
tenant = company:8
assurance = AAL2
device = trusted
```

Authorization podrá evaluar:

```text
identity
roles
permissions
resource ownership
tenant
assurance
device
context
risk
```

sin que Authentication conozca Policies, Roles o Permissions.

---

## 48. Separación Authentication / Authorization

No deberán introducirse en Authentication conceptos como:

```text
Role
Permission
Policy
Ability
Resource Ownership
RBAC
ABAC
ReBAC
Access Decision
```

Estos pertenecen al Authorization System.

Del mismo modo, Authorization no deberá implementar:

```text
password verification
sessions
login
MFA
passkeys
credential validation
identity providers
```

La frontera deberá mantenerse estricta.

---

## 49. FrankenPHP y runtimes persistentes

El sistema deberá estar diseñado desde el principio para runtimes persistentes.

Esto significa evitar estado mutable global que pueda sobrevivir incorrectamente entre requests.

Nunca deberá ocurrir:

```text
Request A
User Alice
    ↓
shared mutable singleton
    ↓
Request B
receives Alice
```

Los datos de autenticación asociados a una request deberán tener un lifecycle explícito.

Conceptualmente:

```text
Application Scope
     │
     ├── immutable authentication configuration
     ├── registries
     └── factories

Request Scope
     │
     ├── AuthenticationContext
     ├── AuthenticationTransaction
     └── request authentication state
```

Esto será crítico para FrankenPHP worker mode.

---

## 50. Performance

El diseño deberá evitar trabajo innecesario.

Objetivos:

```text
lazy identity resolution
lazy session loading
compiled firewall matching
cached authentication configuration
efficient token validation
request-scoped memoization
minimal provider queries
no repeated password verification
no duplicated context reconstruction
```

Las optimizaciones nunca deberán reducir garantías de seguridad.

---

## 51. Compilación

Determinadas configuraciones podrán compilarse durante bootstrap o deployment.

Ejemplos:

```text
firewall definitions
authenticator mappings
provider mappings
authentication policies
route authentication metadata
factor definitions
```

Resultado conceptual:

```text
Authentication Configuration
           ↓
Authentication Compiler
           ↓
Optimized Authentication Graph
           ↓
Runtime
```

Esto reducirá resolución dinámica durante cada request.

---

## 52. Cache

La caché podrá utilizarse para:

```text
configuration
provider metadata
public cryptographic keys
OIDC discovery metadata
JWKS
compiled firewall mappings
non-sensitive identity metadata
```

No deberá utilizarse irresponsablemente para:

```text
raw credentials
plaintext secrets
raw passwords
unprotected authentication tokens
private authentication material
```

---

## 53. Testing

Todos los componentes deberán ser testeables independientemente.

Se contemplarán:

```text
Unit Tests
Integration Tests
Functional Tests
Authentication Flow Tests
Security Tests
Property-Based Tests
Fuzz Tests
Concurrency Tests
Timing Tests
Multi-Tenant Isolation Tests
Persistent Runtime Tests
```

VoltStack Testing deberá proporcionar utilidades como:

```php
actingAs($identity);

actingAsGuest();

withAuthenticationContext(...);

withAssuranceLevel(...);
```

sin requerir autenticación real en cada test cuando no sea el objetivo de la prueba.

---

## 54. Compatibilidad conceptual con Laravel

VoltStack deberá resultar familiar para desarrolladores Laravel.

Ejemplos:

```php
Auth::user();

Auth::check();

Auth::attempt($credentials);

Auth::logout();

Auth::guard('web');
```

Sin embargo:

```text
Laravel API familiarity
          ≠
Laravel internal architecture
```

VoltStack no deberá copiar las limitaciones internas de Laravel únicamente para mantener compatibilidad conceptual.

---

## 55. Influencia de Symfony

Se adoptarán ideas arquitectónicas de Symfony como:

```text
Firewall
Authenticator
Passport
Badges
Authentication Token
User Provider
Token Storage
Success Handler
Failure Handler
```

pero adaptadas al modelo y necesidades de VoltStack.

No se pretende reproducir Symfony Security.

El objetivo es utilizar sus mejores principios de separación de responsabilidades.

---

## 56. Principios de diseño

El Authentication System seguirá los siguientes principios.

### AUTH-01 — Authentication y Authorization son sistemas diferentes

No mezclar responsabilidades.

#### AUTH-02 — Identity sobre User

El sistema autentica identidades, no únicamente modelos `User`.

#### AUTH-03 — Credentials are replaceable

Las contraseñas no son obligatorias.

#### AUTH-04 — Authentication is contextual

Una autenticación incluye método, contexto, assurance y metadata.

#### AUTH-05 — MFA is native

MFA forma parte del modelo fundamental.

#### AUTH-06 — Passwordless is native

Passkeys y mecanismos passwordless no serán adaptaciones secundarias.

#### AUTH-07 — Stateful and stateless are equal citizens

Sessions y tokens reciben soporte arquitectónico equivalente.

#### AUTH-08 — Multi-tenancy is architectural

No será una extensión posterior.

#### AUTH-09 — Fail closed

Los errores no deberán producir autenticación implícita.

#### AUTH-10 — Secure defaults

La configuración inicial deberá favorecer seguridad.

#### AUTH-11 — Extensibility without core modification

Los nuevos mecanismos se incorporarán mediante contratos.

#### AUTH-12 — Runtime isolation

Compatible de forma segura con FrankenPHP y procesos persistentes.

#### AUTH-13 — Observable security

Los eventos importantes deberán ser auditables y observables.

#### AUTH-14 — Ergonomic public API

La complejidad interna no deberá trasladarse innecesariamente al desarrollador.

#### AUTH-15 — Authentication evidence is explicit

Toda autenticación deberá poder explicar qué evidencia permitió establecer la identidad.

---

## 57. Objetivo arquitectónico final

VoltStack deberá permitir que un caso sencillo permanezca sencillo:

```php
if (Auth::attempt([
    'email' => $email,
    'password' => $password,
])) {
    return redirect('/dashboard');
}
```

mientras el mismo sistema pueda manejar:

```text
Enterprise Request
        │
        ▼
Tenant Resolution
        │
        ▼
Authentication Firewall
        │
        ▼
OIDC Authenticator
        │
        ▼
Federated Identity Provider
        │
        ▼
Authentication Passport
        │
        ▼
Identity Resolution
        │
        ▼
Cryptographic Verification
        │
        ▼
Device Evaluation
        │
        ▼
Risk Evaluation
        │
        ▼
Step-Up Requirement
        │
        ▼
Passkey Challenge
        │
        ▼
AAL2 Authentication
        │
        ▼
Authentication Context
        │
        ▼
Session / Token
        │
        ▼
Authorization
```

sin sustituir el núcleo del framework.

---

## 58. Arquitectura conceptual global

```text
                        VOLTSTACK REQUEST
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Tenant / App Context│
                    └──────────┬──────────┘
                               │
                               ▼
                 ┌──────────────────────────┐
                 │ Authentication Firewall  │
                 └─────────────┬────────────┘
                               │
                               ▼
                 ┌──────────────────────────┐
                 │ Authenticator Resolver   │
                 └─────────────┬────────────┘
                               │
                               ▼
                    ┌────────────────────┐
                    │   Authenticator    │
                    └─────────┬──────────┘
                              │
                              ▼
                 ┌─────────────────────────┐
                 │ Authentication Passport │
                 └────────────┬────────────┘
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
        Identity         Credentials        Factors
        Resolution       Verification       / Badges
              │               │                │
              └───────────────┼────────────────┘
                              │
                              ▼
                   Authentication Policy
                              │
                              ▼
                       Risk Evaluation
                              │
                              ▼
                    Challenge / Step-Up
                              │
                              ▼
                   Authentication Decision
                              │
                              ▼
                 ┌─────────────────────────┐
                 │ Authentication Context  │
                 └────────────┬────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
             Session                     Token
                 │                         │
                 └────────────┬────────────┘
                              │
                              ▼
                      Security Context
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         Controllers     Application      Authorization
                           Services
```

---

## 59. Fuera de alcance

Este sistema no será responsable directamente de:

```text
Roles
Permissions
Policies
Access Control Lists
Resource Ownership
RBAC
ABAC
ReBAC
Authorization Decisions
Business Permissions
```

Estos pertenecen al:

```text
VoltStack Authorization System
```

Tampoco deberá absorber responsabilidades de:

```text
Routing
Controllers
Sessions infrastructure
HTTP transport
Database ORM
Cryptographic primitives
Rate Limiter
Cache
Events
Observability
```

Authentication utilizará esos subsistemas mediante contratos bien definidos.

---

## 60. Criterio de éxito

El Authentication System será considerado arquitectónicamente satisfactorio cuando:

1. pueda autenticar identidades humanas y no humanas;
2. soporte sesiones y autenticación stateless;
3. no dependa obligatoriamente de passwords;
4. soporte MFA y step-up authentication;
5. soporte Passkeys/WebAuthn;
6. pueda integrar OAuth/OIDC/SAML/LDAP sin modificar Core;
7. funcione correctamente en aplicaciones multi-tenant;
8. sea seguro bajo runtimes persistentes;
9. pueda proporcionar un `AuthenticationContext` consistente;
10. se integre con Authorization sin acoplamiento circular;
11. permita instrumentación, auditoría y testing;
12. pueda extenderse mediante providers, authenticators, factors y policies;
13. mantenga una API sencilla para casos comunes;
14. pueda optimizarse y compilarse para producción;
15. mantenga una política fail-closed ante estados inesperados.

---

## 61. Regla arquitectónica final

La arquitectura completa deberá conservar permanentemente la siguiente separación:

```text
IDENTITY
   │
   ▼
AUTHENTICATION
   │
   │ proves
   ▼
AUTHENTICATION CONTEXT
   │
   │ informs
   ▼
AUTHORIZATION
   │
   │ decides
   ▼
APPLICATION ACTION
```

Por tanto:

> **Authentication prueba una identidad.**
> **Authentication prueba una identidad.**
> **AuthenticationContext describe cómo y bajo qué condiciones fue establecida esa identidad.**
> **Authorization determina qué puede hacer esa identidad.**
> **AuthenticationContext describe cómo y bajo qué condiciones fue establecida esa identidad.**
> **Authorization determina qué puede hacer esa identidad.**

Ninguno de estos tres conceptos deberá confundirse o fusionarse dentro de VoltStack.

---

## 62. Próximo documento

El siguiente documento del sistema será:

```text
01_AUTHENTICATION_ARCHITECTURE.md
```

Su responsabilidad será transformar los principios establecidos en este Project Context en la arquitectura técnica formal del Authentication System, definiendo:

```text
layers
components
contracts
boundaries
dependencies
data flow
runtime flow
state ownership
extension points
integration points
```

y establecer la estructura arquitectónica sobre la cual se construirán los demás subsistemas de Authentication.
