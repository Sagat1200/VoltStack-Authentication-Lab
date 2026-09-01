# VoltStack Authentication System

## 01 — Authentication Architecture

- **Archivo:** `01_AUTHENTICATION_ARCHITECTURE.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Arquitectura técnica base  
- **Depende de:** `00_AUTHENTICATION_PROJECT_CONTEXT.md`

---

## 1. Propósito

Este documento define la arquitectura técnica del **VoltStack Authentication System**.

Su objetivo es transformar los principios establecidos en `00_AUTHENTICATION_PROJECT_CONTEXT.md` en una estructura concreta de:

- capas;
- componentes;
- contratos;
- dependencias;
- responsabilidades;
- flujos;
- límites;
- estados;
- puntos de extensión;
- integración con otros subsistemas de VoltStack.

La arquitectura deberá permitir que un caso sencillo conserve una API simple:

```php
Auth::attempt([
    'email' => $email,
    'password' => $password,
]);

Auth::user();

Auth::logout();
```

mientras internamente pueda ejecutar flujos como:

```text
Request
   ↓
Tenant Resolution
   ↓
Authentication Firewall
   ↓
Authenticator Selection
   ↓
Passport
   ↓
Identity Resolution
   ↓
Credential Verification
   ↓
Factor Verification
   ↓
Risk Evaluation
   ↓
Policy Evaluation
   ↓
Challenge / Step-Up
   ↓
Authentication Decision
   ↓
Authentication Context
   ↓
Session / Token
   ↓
Authorization
```

---

## 2. Objetivos arquitectónicos

La arquitectura deberá cumplir los siguientes objetivos.

### 2.1 Separación de responsabilidades

Ningún componente deberá concentrar simultáneamente:

```text
request matching
identity loading
credential verification
MFA
session persistence
risk evaluation
authorization
response generation
```

Cada responsabilidad deberá vivir en una abstracción independiente.

---

### 2.2 Independencia de transporte

Authentication no deberá depender exclusivamente de HTTP.

La misma arquitectura deberá funcionar en:

```text
HTTP
SPA
API
WebSocket
SSE
CLI
Queue Worker
Scheduled Job
RPC
Internal Service
```

Por ello, el núcleo utilizará objetos de contexto propios en lugar de depender directamente de `Request`.

---

### 2.3 Independencia de almacenamiento

El sistema no deberá asumir que las identidades están almacenadas en:

```text
SQL
Eloquent
Doctrine
```

Podrán provenir de:

```text
Database
LDAP
Active Directory
Remote API
OIDC
SAML
Memory
Custom Provider
```

---

### 2.4 Independencia del mecanismo de autenticación

El Core no deberá conocer los detalles de:

```text
password
JWT
OAuth
passkeys
API keys
certificates
magic links
```

Estos se implementarán mediante componentes extensibles.

---

### 2.5 Runtime safety

Todo estado relacionado con una request deberá estar explícitamente aislado.

Especialmente bajo:

```text
FrankenPHP
workers
long-lived processes
async runtimes
```

---

## 3. Arquitectura por capas

El sistema se dividirá conceptualmente en siete capas.

```text
┌────────────────────────────────────┐
│  7. Integration / Public API       │
├────────────────────────────────────┤
│  6. Context / State / Persistence  │
├────────────────────────────────────┤
│  5. Decision / Policy / Risk       │
├────────────────────────────────────┤
│  4. Factors / Credentials          │
├────────────────────────────────────┤
│  3. Identity                       │
├────────────────────────────────────┤
│  2. Authentication Orchestration   │
├────────────────────────────────────┤
│  1. Transport Adaptation           │
└────────────────────────────────────┘
```

---

## 4. Capa 1 — Transport Adaptation

Esta capa adapta un entorno externo al modelo interno de Authentication.

Podrán existir adapters para:

```text
HTTP
SPA
CLI
WebSocket
Queue
Service-to-Service
```

Su responsabilidad será crear un:

```php
AuthenticationRequest
```

o equivalente.

Ejemplo conceptual:

```php
final class AuthenticationRequest
{
    public function __construct(
        public readonly AuthenticationTransport $transport,
        public readonly array $headers,
        public readonly array $cookies,
        public readonly array $attributes,
        public readonly ?string $route,
        public readonly ?string $host,
        public readonly ?string $tenantId,
    ) {}
}
```

Esta capa no deberá verificar credenciales.

---

## 5. Authentication Request

`AuthenticationRequest` será una representación normalizada del contexto necesario para autenticar.

Deberá poder contener:

```text
transport
request identifiers
route
host
headers
cookies
client metadata
tenant metadata
remote address
authentication hints
runtime context
```

No deberá convertirse en una copia completa del HTTP Request.

Su función será proporcionar únicamente la información necesaria para Authentication.

---

## 6. Capa 2 — Authentication Orchestration

Esta será la capa central.

Sus componentes principales serán:

```text
AuthenticationManager
AuthenticationFirewallResolver
AuthenticatorResolver
AuthenticationProcessor
AuthenticationTransactionManager
```

La responsabilidad general será:

```text
coordinar
        ↓
resolver
        ↓
ejecutar
        ↓
obtener decisión
```

---

## 7. Authentication Manager

El:

```php
AuthenticationManagerInterface
```

será el principal punto de entrada interno.

Ejemplo conceptual:

```php
interface AuthenticationManagerInterface
{
    public function authenticate(
        AuthenticationRequest $request
    ): AuthenticationResult;
}
```

Podrá existir una implementación:

```php
final class AuthenticationManager
    implements AuthenticationManagerInterface
{
}
```

No deberá implementar directamente cada mecanismo.

Coordinará los demás componentes.

---

## 8. Responsabilidades del Authentication Manager

El Manager podrá:

1. recibir el contexto de autenticación;
2. resolver el firewall;
3. resolver authenticators compatibles;
4. ejecutar el authenticator seleccionado;
5. procesar el Passport;
6. resolver identidad;
7. verificar credenciales;
8. evaluar factores;
9. evaluar políticas;
10. ejecutar risk evaluation;
11. generar challenge cuando sea necesario;
12. establecer AuthenticationContext;
13. persistir estado cuando corresponda;
14. generar AuthenticationResult.

No deberá:

```text
consultar directamente la DB
comparar passwords directamente
crear cookies directamente
autorizar recursos
renderizar responses HTML
```

---

## 9. Authentication Firewall Layer

El firewall determina qué configuración aplica.

Abstracciones propuestas:

```text
AuthenticationFirewall
AuthenticationFirewallCollection
AuthenticationFirewallResolver
AuthenticationFirewallMatcher
```

Ejemplo:

```php
interface AuthenticationFirewallMatcherInterface
{
    public function matches(
        AuthenticationRequest $request
    ): bool;
}
```

---

## 10. Firewall Definition

Una definición conceptual:

```php
final class AuthenticationFirewall
{
    public function __construct(
        public readonly string $name,
        public readonly array $authenticators,
        public readonly string $identityProvider,
        public readonly AuthenticationStateMode $stateMode,
        public readonly array $policies = [],
    ) {}
}
```

Podrá representar:

```text
web
admin
api
internal
tenant
service
```

---

## 11. Firewall Resolution

Ejemplo:

```text
Request: /admin/users
Host: app.example.com
Tenant: acme

        ↓

FirewallResolver

        ↓

admin firewall
```

El matching podrá considerar:

```text
path
route name
host
subdomain
tenant
HTTP method
transport
custom metadata
```

---

## 12. Configuración compilable

Las reglas de firewall podrán compilarse durante bootstrap.

De:

```text
dynamic definitions
```

a:

```text
compiled matcher graph
```

para reducir trabajo durante runtime.

---

## 13. Authenticator Resolver

Una vez seleccionado el firewall, se resolverán authenticators.

Ejemplo:

```text
api firewall

Authenticators:
    BearerTokenAuthenticator
    ApiKeyAuthenticator
```

El resolver determinará cuál soporta el request.

Contrato conceptual:

```php
interface AuthenticatorResolverInterface
{
    public function resolve(
        AuthenticationRequest $request,
        AuthenticationFirewall $firewall
    ): AuthenticatorInterface;
}
```

---

## 14. Authenticator

Cada Authenticator tendrá una responsabilidad limitada.

```php
interface AuthenticatorInterface
{
    public function supports(
        AuthenticationRequest $request
    ): bool;

    public function createPassport(
        AuthenticationRequest $request
    ): AuthenticationPassport;
}
```

El Authenticator:

```text
detecta
extrae
normaliza
construye Passport
```

No necesariamente valida toda la autenticación.

---

## 15. Ejemplo PasswordAuthenticator

```text
POST /login
    ↓
PasswordAuthenticator
    ↓
extract email/password
    ↓
AuthenticationPassport
```

Conceptualmente:

```php
new AuthenticationPassport(
    identity: new IdentityClaim('email', $email),
    credentials: [
        new PasswordCredential($password),
    ],
);
```

---

## 16. Ejemplo BearerTokenAuthenticator

```text
Authorization: Bearer ey...
        ↓
BearerTokenAuthenticator
        ↓
AccessTokenCredential
        ↓
AuthenticationPassport
```

La verificación del token podrá delegarse posteriormente a un verifier.

---

## 17. Authentication Passport Layer

El Passport será la unidad de entrada al motor de verificación.

Podrá contener:

```text
identity claims
credentials
factors
badges
requirements
authentication method
metadata
```

Ejemplo conceptual:

```php
final class AuthenticationPassport
{
    public function __construct(
        public readonly IdentityClaim $identity,
        public readonly array $credentials,
        public readonly array $factors = [],
        public readonly array $requirements = [],
        public readonly array $metadata = [],
    ) {}
}
```

---

## 18. Passport inmutable

El Passport deberá ser preferentemente inmutable.

Los resultados derivados deberán almacenarse en objetos separados.

Esto evita estados ambiguos como:

```text
credential:
    verified = false

más tarde:

credential:
    verified = true
```

En lugar de mutar el Passport, se producirán objetos de evidencia verificada.

---

## 19. Capa 3 — Identity

La arquitectura de identidad incluirá:

```text
IdentityInterface
IdentityIdentifier
IdentityType
IdentityProvider
IdentityResolver
IdentityRepository abstraction
```

---

## 20. IdentityInterface

Contrato base conceptual:

```php
interface IdentityInterface
{
    public function identifier(): IdentityIdentifier;

    public function type(): IdentityType;
}
```

El contrato base deberá ser mínimo.

No deberán incluirse obligatoriamente:

```text
password
email
roles
permissions
tenant
```

---

## 21. Identity Identifier

Los identificadores deberán tratarse como value objects.

Ejemplos:

```text
UserId
UUID
ULID
ServiceId
ExternalSubject
ClientId
```

No se deberá asumir que el identificador siempre es un entero.

---

## 22. Identity Provider

Contrato conceptual:

```php
interface IdentityProviderInterface
{
    public function resolve(
        IdentityClaim $claim
    ): ?IdentityInterface;
}
```

Implementaciones posibles:

```text
DatabaseIdentityProvider
OrmIdentityProvider
LdapIdentityProvider
OidcIdentityProvider
RemoteIdentityProvider
MemoryIdentityProvider
```

---

## 23. Identity Claim

La identificación presentada no necesariamente será un ID interno.

Podrá ser:

```text
email
username
phone
OIDC subject
certificate subject
API client id
service id
```

Por ello se utilizará un:

```text
IdentityClaim
```

Ejemplo:

```php
new IdentityClaim(
    type: 'email',
    value: 'user@example.com',
);
```

---

## 24. Identity Resolution

Flujo:

```text
AuthenticationPassport
        │
        ▼
IdentityClaim
        │
        ▼
IdentityProvider
        │
        ▼
Identity
```

El provider podrá aplicar estrategias específicas de normalización.

---

## 25. Capa 4 — Credentials and Factors

Esta capa manejará la verificación de evidencia.

Componentes:

```text
Credential
CredentialVerifier
CredentialVerifierRegistry
Factor
FactorVerifier
Challenge
```

---

## 26. Credential Interface

Contrato conceptual:

```php
interface CredentialInterface
{
    public function type(): string;
}
```

Ejemplos:

```text
PasswordCredential
ApiKeyCredential
AccessTokenCredential
CertificateCredential
SignedAssertionCredential
PasskeyCredential
```

---

## 27. Credential Verifier

Cada tipo deberá tener un verifier específico.

```php
interface CredentialVerifierInterface
{
    public function supports(
        CredentialInterface $credential
    ): bool;

    public function verify(
        IdentityInterface $identity,
        CredentialInterface $credential,
        AuthenticationContextInput $context
    ): CredentialVerificationResult;
}
```

---

## 28. Credential Verification Result

El resultado deberá ser explícito.

Ejemplo:

```text
VALID
INVALID
EXPIRED
REVOKED
UNSUPPORTED
ERROR
```

Errores internos no deberán tratarse como credenciales válidas.

---

## 29. Factor Architecture

Un factor adicional podrá representarse como:

```php
AuthenticationFactorInterface
```

Ejemplos:

```text
TotpFactor
PasskeyFactor
RecoveryCodeFactor
PushFactor
TrustedDeviceFactor
```

---

## 30. Factor Requirement

El sistema deberá separar:

```text
factor available
```

de:

```text
factor required
```

Una Authentication Policy podrá declarar:

```text
require AAL2
```

y el sistema determinar qué factores pueden elevar el assurance.

---

## 31. Challenges

Cuando falta un factor, se generará un:

```text
AuthenticationChallenge
```

Ejemplo:

```text
TotpChallenge
PasskeyChallenge
RecoveryChallenge
```

En lugar de marcar simplemente la autenticación como fallida.

---

## 32. Capa 5 — Decision, Policy and Risk

Esta capa determinará si la evidencia presentada es suficiente.

Componentes:

```text
AuthenticationPolicyEngine
RiskEvaluator
AssuranceCalculator
AuthenticationDecisionEngine
```

---

## 33. Authentication Policy

Las policies de Authentication no deberán confundirse con Authorization Policies.

Aquí una policy responde a:

> ¿Qué requisitos deben cumplirse para considerar válida esta autenticación?

Ejemplos:

```text
password required
AAL2 required
tenant must exist
device verification required
MFA required for administrators
passkey required for privileged identities
```

---

## 34. AuthenticationPolicyInterface

Conceptualmente:

```php
interface AuthenticationPolicyInterface
{
    public function evaluate(
        AuthenticationEvidence $evidence,
        AuthenticationEnvironment $environment
    ): AuthenticationPolicyResult;
}
```

---

## 35. Risk Evaluation

El Risk Engine podrá producir:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

o un score numérico.

Input potencial:

```text
IP
device
identity
history
tenant
authentication method
failed attempts
geolocation signals
client characteristics
```

El Risk Engine no deberá decidir permisos de negocio.

---

## 36. Risk Result

Ejemplo conceptual:

```php
final class AuthenticationRiskResult
{
    public function __construct(
        public readonly int $score,
        public readonly AuthenticationRiskLevel $level,
        public readonly array $signals,
    ) {}
}
```

---

## 37. Assurance Calculator

El sistema calculará el nivel de assurance logrado.

Input:

```text
method
credentials
factors
device binding
cryptographic properties
provider
risk context
```

Output:

```text
AuthenticationAssuranceLevel
```

---

## 38. Authentication Decision

La decisión final tendrá estados explícitos.

```text
AUTHENTICATED
CHALLENGE_REQUIRED
STEP_UP_REQUIRED
REJECTED
ERROR
```

No deberá limitarse a un booleano.

---

## 39. AuthenticationDecision Object

Ejemplo:

```php
final class AuthenticationDecision
{
    public function __construct(
        public readonly AuthenticationDecisionStatus $status,
        public readonly ?AuthenticationChallenge $challenge,
        public readonly array $reasons = [],
    ) {}
}
```

---

## 40. Capa 6 — Context, State and Persistence

Una vez autenticada la identidad, se creará un:

```text
AuthenticationContext
```

y, cuando corresponda:

```text
Session
Token
Context Storage
```

---

## 41. Authentication Context

El Context deberá ser el objeto canónico que representa la autenticación activa.

Ejemplo:

```php
final class AuthenticationContext
{
    public function __construct(
        public readonly IdentityInterface $identity,
        public readonly AuthenticationMethod $method,
        public readonly AuthenticationAssuranceLevel $assurance,
        public readonly DateTimeImmutable $authenticatedAt,
        public readonly ?TenantIdentifier $tenant,
        public readonly ?DeviceIdentifier $device,
        public readonly array $attributes = [],
    ) {}
}
```

---

## 42. Request-Scoped Context

El AuthenticationContext activo deberá vivir en un scope explícito.

No deberá almacenarse permanentemente en un singleton mutable.

Correcto:

```text
Request
   ↓
RequestScope
   ↓
AuthenticationContext
```

Incorrecto:

```text
global AuthManager
    └── currentUser = Alice
```

---

## 43. Authentication Context Storage

Contrato:

```php
interface AuthenticationContextStorageInterface
{
    public function get(): ?AuthenticationContext;

    public function set(
        AuthenticationContext $context
    ): void;

    public function clear(): void;
}
```

La implementación deberá respetar request scoping.

---

## 44. Session Persistence

Para stateful auth:

```text
AuthenticationContext
        ↓
SessionAuthenticationState
        ↓
Session Store
        ↓
Session Identifier
```

No será necesario serializar siempre el objeto Identity completo.

Podrá persistirse:

```text
identity id
provider
authentication method
assurance
timestamps
session metadata
```

y reconstruir posteriormente el contexto.

---

## 45. Token Persistence

Para auth stateless:

```text
AuthenticationContext
        ↓
Token Issuer
        ↓
Signed / opaque token
```

La arquitectura soportará tanto:

```text
opaque tokens
signed tokens
reference tokens
JWT-like tokens
```

sin convertir JWT en el modelo base.

---

## 46. Capa 7 — Public API and Integration

Esta capa presentará una API cómoda a desarrolladores.

Componentes:

```text
Auth Facade
Guards
Middleware
Helpers
Testing utilities
Framework integrations
```

---

## 47. Auth Facade

Ejemplo:

```php
Auth::check();

Auth::identity();

Auth::user();

Auth::id();

Auth::context();

Auth::guard('web');

Auth::logout();
```

La facade delegará a servicios internos.

---

## 48. Guard como API pública

VoltStack conservará el concepto de Guard principalmente por ergonomía.

Un Guard podrá representar:

```text
named authentication configuration
```

Ejemplo:

```php
Auth::guard('admin')->check();
```

Pero internamente:

```text
Guard
   ↓
Firewall / Authentication Configuration
   ↓
Authentication Manager
```

Guard no deberá convertirse en el centro arquitectónico.

---

## 49. GuardInterface

Conceptualmente:

```php
interface GuardInterface
{
    public function check(): bool;

    public function identity(): ?IdentityInterface;

    public function context(): ?AuthenticationContext;

    public function logout(): void;
}
```

Métodos adicionales podrán existir según estrategia.

---

## 50. Authentication Result

El resultado general del Manager deberá ser estructurado.

Posibles estados:

```text
AuthenticatedResult
UnauthenticatedResult
ChallengeRequiredResult
StepUpRequiredResult
RejectedResult
AuthenticationErrorResult
```

Esto facilitará su adaptación a:

```text
HTTP
SPA
CLI
API
```

---

## 51. Success Handler

La autenticación exitosa podrá activar un:

```text
AuthenticationSuccessHandlerInterface
```

Responsabilidades posibles:

```text
session creation
token issuance
event dispatch
audit
redirect metadata
SPA response metadata
```

No deberá ejecutar Authorization.

---

## 52. Failure Handler

La falla podrá ser tratada por:

```text
AuthenticationFailureHandlerInterface
```

Deberá transformar fallos internos en respuestas apropiadas al contexto.

Ejemplo:

```text
InvalidCredentialsException
        ↓
generic login failure response
```

sin filtrar información innecesaria.

---

## 53. Challenge Handler

Los challenges tendrán manejo independiente.

```text
AuthenticationChallenge
        ↓
ChallengeHandler
        ↓
Transport-specific representation
```

Ejemplo SPA:

```json
{
    "authentication": {
        "status": "challenge_required",
        "type": "totp"
    }
}
```

Ejemplo web:

```text
redirect /mfa
```

---

## 54. Authentication Transaction

Los flujos multi-step requerirán un estado temporal independiente de la sesión final.

Objeto:

```text
AuthenticationTransaction
```

Podrá contener:

```text
transaction id
identity claim
verified evidence
pending requirements
challenge state
creation time
expiration time
tenant
client metadata
```

---

## 55. Transaction Store

El almacenamiento podrá ser:

```text
session
cache
database
distributed store
signed temporary token
```

mediante:

```php
AuthenticationTransactionStoreInterface
```

---

## 56. Idempotencia

Los pasos sensibles deberán diseñarse para prevenir ejecución múltiple accidental.

Ejemplos:

```text
magic link consumption
recovery code usage
token exchange
OAuth callback
passkey challenge completion
```

Los componentes deberán soportar conceptos de:

```text
one-time use
replay detection
nonce
transaction state
```

---

## 57. Dependencias entre capas

La regla principal será:

```text
higher-level layers
        ↓
depend on interfaces
        ↓
lower-level capabilities
```

No deberán existir dependencias circulares.

---

## 58. Grafo conceptual de dependencias

```text
Public API
   │
   ▼
Authentication Manager
   │
   ├── Firewall Resolver
   ├── Authenticator Resolver
   ├── Passport Processor
   │
   ├── Identity Provider
   ├── Credential Verifiers
   ├── Factor Verifiers
   │
   ├── Risk Evaluator
   ├── Authentication Policy Engine
   ├── Assurance Calculator
   │
   └── Context Factory
           │
           ├── Session Strategy
           └── Token Strategy
```

---

## 59. Passport Processor

Para evitar que el Manager se vuelva monolítico, el procesamiento del Passport podrá delegarse a:

```php
AuthenticationPassportProcessorInterface
```

Responsabilidades:

```text
resolve identity
verify credentials
verify available factors
construct evidence
```

---

## 60. Authentication Evidence

Una vez procesado el Passport se generará evidencia verificada.

Ejemplo:

```text
AuthenticationEvidence
│
├── ResolvedIdentity
├── VerifiedCredentials
├── VerifiedFactors
├── ProviderEvidence
├── DeviceEvidence
└── Metadata
```

Este objeto será diferente del Passport original.

---

## 61. Evidencia verificable

Cada elemento deberá registrar de manera estructurada:

```text
what was verified
by which verifier
at what time
using which method
```

sin almacenar secretos sensibles.

Esto permitirá explicar posteriormente el assurance de autenticación.

---

## 62. State Machine

Authentication podrá modelarse como máquina de estados.

```text
UNAUTHENTICATED
      │
      ▼
IDENTITY_PENDING
      │
      ▼
CREDENTIAL_PENDING
      │
      ▼
FACTOR_PENDING
      │
      ├───────────────┐
      ▼               │
CHALLENGE_REQUIRED    │
      │               │
      └───────────────┘
      │
      ▼
EVIDENCE_VERIFIED
      │
      ▼
POLICY_EVALUATION
      │
      ▼
AUTHENTICATED
```

Estados terminales alternativos:

```text
REJECTED
EXPIRED
CANCELLED
ERROR
```

---

## 63. Estados no ambiguos

No deberán mezclarse:

```text
not authenticated yet
```

con:

```text
authentication failed
```

ni:

```text
challenge required
```

con:

```text
authentication rejected
```

Esto será especialmente importante para MFA y SPA.

---

## 64. Error Architecture

Las excepciones internas podrán agruparse jerárquicamente.

```text
AuthenticationException
│
├── IdentityResolutionException
├── CredentialVerificationException
├── InvalidCredentialException
├── AuthenticationPolicyException
├── AuthenticationChallengeException
├── TokenException
├── SessionAuthenticationException
└── ProviderException
```

La capa de transporte decidirá cuánto exponer.

---

## 65. Fail Closed

Cualquier error inesperado deberá resultar en:

```text
NOT AUTHENTICATED
```

Nunca:

```text
authentication assumed valid
```

Ejemplo:

```text
identity provider timeout
        ↓
authentication error
        ↓
no authenticated context
```

---

## 66. Security Boundary

El Authentication System constituye una frontera de seguridad.

Datos de entrada deberán considerarse no confiables:

```text
cookies
headers
tokens
credentials
client metadata
OAuth assertions
tenant identifiers
device metadata
```

Nada deberá convertirse en `AuthenticationContext` sin validación apropiada.

---

## 67. Secret Handling

Secrets deberán minimizar su tiempo de vida.

Ejemplo:

```text
PasswordCredential
    ↓
Verifier
    ↓
verification
    ↓
discard
```

No deberán copiarse a:

```text
AuthenticationContext
logs
events
exceptions
cache
session
```

---

## 68. Logging Boundary

Los logs podrán incluir:

```text
identity identifier
provider
authenticator
result
risk
assurance
session id hash
request correlation id
```

pero nunca:

```text
raw password
raw bearer token
MFA secret
private key
full recovery code
```

---

## 69. Events Architecture

El sistema emitirá eventos en puntos definidos.

```text
AuthenticationStarted
FirewallResolved
AuthenticatorSelected
PassportCreated
IdentityResolved
CredentialVerified
FactorVerified
ChallengeRequired
AuthenticationSucceeded
AuthenticationFailed
AuthenticationContextCreated
LogoutCompleted
```

Los listeners no deberán alterar de forma insegura el estado principal.

---

## 70. Pre/Post Hooks

Cuando sea necesaria extensión síncrona se usarán contratos explícitos.

Ejemplo:

```text
BeforeAuthenticationProcessor
AfterCredentialVerificationProcessor
BeforeContextCreationProcessor
```

No se dependerá exclusivamente de eventos para lógica crítica.

---

## 71. Extension Registries

Podrán existir:

```text
AuthenticatorRegistry
IdentityProviderRegistry
CredentialVerifierRegistry
FactorRegistry
ChallengeHandlerRegistry
RiskEvaluatorRegistry
AuthenticationPolicyRegistry
```

Los registries deberán congelarse o compilarse después del bootstrap cuando sea posible.

---

## 72. Provider Discovery

Los providers podrán registrarse mediante:

```php
Auth::extend(...);
```

o configuración:

```php
'providers' => [
    'users' => [
        'driver' => 'database',
    ],
],
```

La API exacta se definirá posteriormente.

---

## 73. Integración con Container

El Container resolverá las implementaciones.

Ejemplo:

```text
AuthenticatorInterface
      ↓
concrete authenticator
```

Registries y factories podrán utilizar service identifiers.

---

## 74. Integración con Configuration

La configuración deberá ser declarativa.

Ejemplo conceptual:

```php
return [
    'defaults' => [
        'guard' => 'web',
    ],

    'firewalls' => [
        'web' => [
            'stateful' => true,
            'provider' => 'users',
            'authenticators' => [
                'session',
                'password',
            ],
        ],

        'api' => [
            'stateful' => false,
            'provider' => 'users',
            'authenticators' => [
                'bearer',
            ],
        ],
    ],
];
```

---

## 75. Integración con Routing

Routing podrá adjuntar metadata como:

```text
requires authentication
required guard
required assurance level
authentication firewall hint
```

Authentication no deberá depender directamente del router concreto.

Consumirá metadata normalizada.

---

## 76. Integración con Middleware

Middleware será un adapter.

Ejemplo:

```text
HTTP Request
    ↓
Authenticate Middleware
    ↓
Authentication Manager
    ↓
AuthenticationResult
```

El middleware no deberá implementar verificaciones de password o tokens por sí mismo.

---

## 77. Integración con Authorization

La única dirección válida será:

```text
Authentication
      ↓
AuthenticationContext
      ↓
Authorization
```

Nunca:

```text
Authorization
      ↓
required to establish Identity
```

excepto en casos de políticas de aplicación posteriores a la autenticación.

---

## 78. Separación de Authentication Policy

Debe distinguirse:

```text
Authentication Policy
```

de:

```text
Authorization Policy
```

Authentication Policy:

```text
Does this login satisfy the required authentication guarantees?
```

Authorization Policy:

```text
May this identity perform this action on this resource?
```

---

## 79. Integración con Session System

Authentication dependerá de contratos de sesión.

No deberá implementar el storage de sesiones general.

```text
Quantum/Auth
    ↓
Session Contract
    ↓
Quantum/Session
```

---

## 80. Integración con Cache

Cache podrá utilizarse para:

```text
OIDC metadata
public keys
JWKS
provider metadata
compiled auth configuration
rate-limit state
transaction state
```

Nunca como repositorio inseguro de secretos sin protección.

---

## 81. Integración con Crypto

Operaciones criptográficas deberán delegarse a primitivas bien definidas.

Ejemplos:

```text
password hashing
token signing
constant-time comparison
random bytes
nonce generation
key derivation
```

Authentication no deberá reimplementar criptografía de bajo nivel.

---

## 82. Integración con Rate Limiter

El Authentication System enviará contexto suficiente para aplicar límites.

Ejemplo:

```text
AuthenticationThrottleKey
│
├── IP
├── identity claim
├── tenant
├── authenticator
└── device
```

---

## 83. Integración con Observability

Cada etapa podrá producir spans.

Ejemplo:

```text
auth.authenticate
 ├── auth.firewall.resolve
 ├── auth.authenticator.resolve
 ├── auth.identity.resolve
 ├── auth.credential.verify
 ├── auth.risk.evaluate
 └── auth.context.create
```

La telemetría deberá evitar secretos.

---

## 84. Integración con Audit

El sistema de auditoría recibirá eventos estructurados.

Ejemplo:

```text
AuthenticationAuditRecord
```

con:

```text
event
identity
tenant
authenticator
outcome
timestamp
correlation id
```

---

## 85. Scope Architecture

Se distinguirán al menos:

```text
Application Scope
Request Scope
Authentication Transaction Scope
Session Scope
```

---

## 86. Application Scope

Podrá contener:

```text
configuration
compiled firewall definitions
registries
stateless factories
providers
immutable metadata
```

---

## 87. Request Scope

Podrá contener:

```text
AuthenticationRequest
AuthenticationResult
AuthenticationContext
request memoization
```

Deberá destruirse al terminar la request.

---

## 88. Authentication Transaction Scope

Podrá sobrevivir varias requests.

Ejemplo:

```text
login request
    ↓
MFA request
    ↓
challenge completion
```

pero deberá tener:

```text
expiration
ownership
replay protection
cleanup
```

---

## 89. Session Scope

Podrá sobrevivir múltiples requests y representar autenticación persistente.

No deberá confundirse con request scope.

---

## 90. Lazy Authentication

El sistema podrá soportar autenticación lazy.

Ejemplo:

```text
Request
   ↓
No component asks for identity
   ↓
session identity not loaded
```

Cuando:

```php
Auth::user();
```

se invoque:

```text
load session auth state
      ↓
resolve identity
      ↓
create request context
```

Esto reducirá queries innecesarias.

---

## 91. Eager Authentication

Determinadas rutas podrán requerir autenticación temprana.

Ejemplo:

```text
protected admin route
```

Entonces middleware o routing metadata podrá forzar:

```text
authenticate before controller
```

---

## 92. Request Memoization

Dentro de una misma request:

```php
Auth::user();
Auth::user();
Auth::user();
```

no deberá provocar tres consultas al provider.

El contexto resuelto se memoizará por request.

---

## 93. Identity Refresh

La arquitectura deberá definir cuándo refrescar una Identity.

Posibles estrategias:

```text
every request
session snapshot
version-aware
time-based refresh
security-sensitive refresh
```

La estrategia exacta se documentará posteriormente.

---

## 94. Context Versioning

El estado persistido podrá incorporar versión.

Ejemplo:

```text
identity_version
credential_version
security_version
```

para invalidar sesiones antiguas cuando ocurra:

```text
password change
account disable
security reset
credential revocation
```

---

## 95. Revocation Architecture

La revocación deberá poder afectar:

```text
specific session
all sessions
specific token
all tokens
device
identity
credential
service account
```

El sistema deberá disponer de abstracciones para ello.

---

## 96. Authentication Context Factory

La creación del Context deberá centralizarse.

```php
AuthenticationContextFactoryInterface
```

Input:

```text
verified evidence
assurance
environment
risk
```

Output:

```text
AuthenticationContext
```

---

## 97. Context Integrity

Una vez creado, el contexto deberá ser preferentemente inmutable.

Cambios importantes deberán producir un nuevo contexto.

Ejemplo:

```text
AAL1 Context
    ↓
Step-Up
    ↓
new AAL2 Context
```

No simplemente modificar silenciosamente el anterior.

---

## 98. Authentication Method

Se utilizará un value object o enum extensible:

```text
PASSWORD
SESSION
PASSKEY
ACCESS_TOKEN
API_KEY
OIDC
SAML
CERTIFICATE
SERVICE_CREDENTIAL
```

Podrá representar cadenas de métodos si hay varios factores.

---

## 99. Authentication Provenance

El Context deberá poder indicar de dónde provino la autenticación.

Ejemplo:

```text
provider: enterprise_oidc
authenticator: oidc
issuer: corporate-idp
```

Esto será útil para auditoría y Authorization contextual.

---

## 100. Tenant Context

Cuando la aplicación sea multi-tenant:

```text
TenantContext
```

deberá resolverse antes o durante Authentication según estrategia.

El `AuthenticationContext` deberá asociarse inequívocamente al tenant cuando corresponda.

---

## 101. Cross-Tenant Protection

Una sesión creada en:

```text
tenant A
```

no deberá utilizarse automáticamente en:

```text
tenant B
```

salvo configuración explícita de identidad global.

---

## 102. Device Context

El sistema podrá recibir:

```text
DeviceContext
```

como input.

Podrá incluir:

```text
device id
trust state
first seen
last seen
client characteristics
binding information
```

---

## 103. Service Authentication

Machine-to-machine deberá utilizar el mismo pipeline central.

```text
Service Request
      ↓
ServiceAuthenticator
      ↓
Service Credential
      ↓
ServiceIdentity
      ↓
AuthenticationContext
```

No será un sistema paralelo.

---

## 104. CLI Authentication

CLI podrá establecer contextos mediante:

```text
explicit service identity
interactive identity
temporary token
operator identity
```

sin depender de cookies HTTP.

---

## 105. Queue Authentication

Jobs podrán transportar una referencia autenticada segura cuando sea necesario.

No deberá serializarse indiscriminadamente todo el `AuthenticationContext`.

Se deberá utilizar un modelo específico de:

```text
delegated authentication context
```

o identidad de ejecución.

---

## 106. WebSocket Authentication

La autenticación inicial podrá establecer un contexto asociado a la conexión.

Deberán contemplarse:

```text
connection lifetime
token expiration
reauthentication
tenant isolation
context refresh
```

---

## 107. SPA Architecture

SPA podrá recibir estados estructurados.

```text
AUTHENTICATED
UNAUTHENTICATED
CHALLENGE_REQUIRED
STEP_UP_REQUIRED
SESSION_EXPIRED
```

El frontend no deberá depender de detectar HTML inesperado para entender la autenticación.

---

## 108. API Architecture

Las APIs utilizarán respuestas deterministas.

Ejemplo:

```text
401
    unauthenticated

403
    authenticated but unauthorized

authentication challenge metadata
    when applicable
```

La separación entre 401 y 403 deberá conservarse.

---

## 109. Testing Architecture

El sistema deberá permitir reemplazar fácilmente:

```text
IdentityProvider
CredentialVerifier
RiskEvaluator
SessionStore
TokenVerifier
Clock
Random generator
```

mediante dependency injection.

---

## 110. Test Authentication Context

Testing podrá establecer contextos directamente.

Ejemplo:

```php
$this->actingAs($user);
```

Internamente:

```text
TestAuthenticationContextFactory
        ↓
request-scoped AuthenticationContext
```

sin ejecutar login real.

---

## 111. Clock Abstraction

Todo comportamiento temporal deberá depender de una abstracción de reloj.

Ejemplos:

```text
token expiration
session expiration
challenge expiration
authentication time
risk windows
```

Esto facilita testing determinista.

---

## 112. Randomness Abstraction

Operaciones que requieran aleatoriedad criptográfica deberán pasar por un servicio especializado.

Ejemplos:

```text
session identifiers
nonce
recovery tokens
challenge identifiers
```

No deberán utilizar generadores débiles.

---

## 113. Serialization Boundary

Objetos sensibles no deberán ser serializables automáticamente.

Especialmente:

```text
PasswordCredential
RawTokenCredential
PrivateKeyCredential
MfaSecret
```

La persistencia deberá utilizar representaciones específicas y mínimas.

---

## 114. Compilation Architecture

Durante bootstrap podrán compilarse:

```text
firewall matcher graph
authenticator maps
provider maps
policy definitions
route authentication metadata
factor configuration
```

Resultado:

```text
AuthenticationCompiledConfiguration
```

---

## 115. Runtime Optimization

En producción se buscará:

```text
O(1) or near-O(1) registry lookup
minimal provider queries
lazy identity resolution
single credential verification
compiled request matching
request memoization
pre-resolved configuration
```

---

## 116. No Security Downgrade

Ninguna optimización podrá omitir:

```text
credential verification
signature verification
revocation checks required by policy
tenant validation
context isolation
```

---

## 117. Suggested Package Structure

```text
src/
└── Quantum/
    └── Auth/
        ├── Contracts/
        │   ├── AuthenticationManagerInterface.php
        │   ├── AuthenticatorInterface.php
        │   ├── IdentityInterface.php
        │   ├── IdentityProviderInterface.php
        │   ├── CredentialInterface.php
        │   ├── CredentialVerifierInterface.php
        │   ├── AuthenticationPolicyInterface.php
        │   └── AuthenticationContextStorageInterface.php
        │
        ├── Authentication/
        │   ├── AuthenticationManager.php
        │   ├── AuthenticationProcessor.php
        │   ├── AuthenticationResult.php
        │   └── AuthenticationDecision.php
        │
        ├── Firewall/
        ├── Authenticator/
        ├── Passport/
        ├── Identity/
        ├── Credentials/
        ├── Factors/
        ├── Challenge/
        ├── Evidence/
        ├── Risk/
        ├── Policy/
        ├── Assurance/
        ├── Context/
        ├── Sessions/
        ├── Tokens/
        ├── Transactions/
        ├── Guards/
        ├── Middleware/
        ├── Events/
        ├── Exceptions/
        ├── Audit/
        ├── Observability/
        ├── Compilation/
        ├── Testing/
        └── Support/
```

---

## 118. Public vs Internal API

Se distinguirán claramente:

```text
Public API
```

de:

```text
Internal SPI
```

Public API:

```text
Auth facade
Guard contracts
AuthenticationContext
Identity contracts
selected extension interfaces
```

Internal SPI:

```text
pipeline processors
compiler internals
runtime state handlers
internal result objects
```

Esto permitirá evolucionar internamente sin romper aplicaciones.

---

## 119. Stability Levels

Los componentes podrán clasificarse como:

```text
Public Stable
Extension Stable
Internal
Experimental
```

Ejemplo:

```text
IdentityInterface
    Public Stable

AuthenticatorInterface
    Extension Stable

CompiledFirewallGraph
    Internal
```

---

## 120. Architectural Invariants

Las siguientes reglas deberán mantenerse siempre.

### AUTH-ARCH-01

Una `Identity` no implica automáticamente que esté autenticada.

---

#### AUTH-ARCH-02

Un `AuthenticationPassport` no representa autenticación exitosa.

---

#### AUTH-ARCH-03

Solo evidencia verificada podrá producir un `AuthenticationContext`.

---

#### AUTH-ARCH-04

Un `AuthenticationContext` activo deberá estar asociado a un scope explícito.

---

#### AUTH-ARCH-05

Los secretos no deberán persistirse dentro del contexto.

---

#### AUTH-ARCH-06

Authentication no deberá evaluar permisos de recursos.

---

#### AUTH-ARCH-07

Authorization no deberá verificar credenciales.

---

#### AUTH-ARCH-08

Los mecanismos de autenticación deberán ser extensibles sin modificar Core.

---

#### AUTH-ARCH-09

Los errores deberán fallar de forma cerrada.

---

#### AUTH-ARCH-10

Las rutas stateful y stateless deberán compartir el mismo dominio de Authentication.

---

#### AUTH-ARCH-11

Los flujos multi-step deberán utilizar transacciones explícitas.

---

#### AUTH-ARCH-12

El estado de request no deberá sobrevivir accidentalmente bajo runtimes persistentes.

---

## 121. Arquitectura global

```text
┌──────────────────────────────────────────────────────────────┐
│                       VOLTSTACK RUNTIME                      │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Transport Adapter   │
                    └─────────┬───────────┘
                              │
                              ▼
                 ┌──────────────────────────┐
                 │ AuthenticationRequest    │
                 └────────────┬─────────────┘
                              │
                              ▼
                 ┌──────────────────────────┐
                 │ AuthenticationManager    │
                 └────────────┬─────────────┘
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
        Firewall         Authenticator     Transaction
        Resolver          Resolver          Manager
             │                │
             └────────┬───────┘
                      ▼
               Authenticator
                      │
                      ▼
          AuthenticationPassport
                      │
                      ▼
            Passport Processor
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
    Identity       Credential      Factor
    Provider        Verifiers      Verifiers
        │             │              │
        └─────────────┼──────────────┘
                      ▼
            AuthenticationEvidence
                      │
            ┌─────────┼─────────┐
            ▼         ▼         ▼
          Risk      Policy   Assurance
        Evaluator   Engine   Calculator
            │         │         │
            └─────────┼─────────┘
                      ▼
            AuthenticationDecision
                      │
         ┌────────────┼─────────────┐
         │            │             │
         ▼            ▼             ▼
   Authenticated   Challenge      Rejected
         │
         ▼
 AuthenticationContextFactory
         │
         ▼
 AuthenticationContext
         │
    ┌────┴─────────┐
    ▼              ▼
 Session         Token
 State           State
    │              │
    └──────┬───────┘
           ▼
 AuthenticationContextStorage
           │
    ┌──────┼──────────────┐
    ▼      ▼              ▼
 Controllers Services Authorization
```

---

## 122. Flujo resumido de una autenticación por password

```text
POST /login
    ↓
HTTP Adapter
    ↓
AuthenticationRequest
    ↓
web Firewall
    ↓
PasswordAuthenticator
    ↓
AuthenticationPassport
    ↓
IdentityProvider
    ↓
UserIdentity
    ↓
PasswordVerifier
    ↓
VerifiedCredential
    ↓
Policy Engine
    ↓
AAL1
    ↓
AUTHENTICATED
    ↓
AuthenticationContext
    ↓
Session State
    ↓
Auth::user()
```

---

## 123. Flujo resumido con MFA

```text
Password verified
      ↓
AuthenticationEvidence
      ↓
Policy requires AAL2
      ↓
Current assurance AAL1
      ↓
ChallengeRequired
      ↓
AuthenticationTransaction stored
      ↓
TOTP Challenge
      ↓
TOTP verified
      ↓
Evidence extended
      ↓
AAL2
      ↓
AuthenticationContext
```

---

## 124. Flujo resumido stateless

```text
Bearer Token
     ↓
BearerTokenAuthenticator
     ↓
Token Credential
     ↓
Token Verifier
     ↓
Identity Resolution
     ↓
Verified Evidence
     ↓
AuthenticationContext
     ↓
request-scoped only
```

No será necesario crear sesión.

---

## 125. Flujo resumido Passkey

```text
Passkey Assertion
      ↓
PasskeyAuthenticator
      ↓
AuthenticationPassport
      ↓
Challenge Validation
      ↓
Credential Public Key
      ↓
Signature Verification
      ↓
Identity Resolution
      ↓
AuthenticationEvidence
      ↓
Assurance Evaluation
      ↓
AuthenticationContext
```

---

## 126. Flujo resumido OIDC

```text
OIDC Callback
     ↓
OidcAuthenticator
     ↓
AuthenticationTransaction
     ↓
Authorization Code Verification
     ↓
Token Exchange
     ↓
ID Token Verification
     ↓
Federated Identity Resolution
     ↓
AuthenticationEvidence
     ↓
AuthenticationContext
```

---

## 127. Responsabilidad final por componente

```text
AuthenticationRequest
    normaliza entrada

Firewall
    selecciona configuración

Authenticator
    interpreta mecanismo

Passport
    representa intento

IdentityProvider
    resuelve identidad

CredentialVerifier
    verifica credencial

FactorVerifier
    verifica factores adicionales

RiskEvaluator
    analiza riesgo

AuthenticationPolicy
    define requisitos

AssuranceCalculator
    calcula confianza

AuthenticationDecision
    determina resultado

AuthenticationContext
    representa identidad autenticada

Session/Token Strategy
    mantiene estado

Auth Facade
    ofrece ergonomía
```

---

## 128. Criterios de aceptación

La arquitectura será considerada válida cuando permita:

1. múltiples firewalls;
2. múltiples authenticators por firewall;
3. múltiples identity providers;
4. credenciales extensibles;
5. MFA y step-up;
6. sesiones y tokens;
7. identidades humanas y de máquina;
8. multi-tenancy;
9. flows multi-step;
10. Passkeys;
11. OIDC/SAML/LDAP mediante extensiones;
12. request-scoped context;
13. lazy authentication;
14. observabilidad;
15. auditoría;
16. testing aislado;
17. compilación;
18. FrankenPHP-safe runtime;
19. integración limpia con Authorization;
20. API pública sencilla.

---

## 129. Regla arquitectónica final

El Authentication System deberá preservar siempre esta secuencia conceptual:

```text
INPUT
  ↓
INTERPRETATION
  ↓
IDENTITY
  ↓
EVIDENCE
  ↓
VERIFICATION
  ↓
POLICY
  ↓
DECISION
  ↓
AUTHENTICATION CONTEXT
  ↓
STATE
  ↓
APPLICATION
```

Ningún acceso a la identidad autenticada deberá saltarse la generación de un estado de autenticación válido.

La arquitectura interna podrá evolucionar, pero esta separación de responsabilidades deberá mantenerse estable.

---

## 130. Próximo documento

El siguiente documento será:

```text
02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md
```

Su responsabilidad será definir formalmente el modelo de dominio de Authentication, incluyendo:

```text
Identity
IdentityClaim
Credential
Factor
Authenticator
Passport
Evidence
Challenge
Transaction
Decision
AuthenticationContext
AuthenticationToken
Session
Assurance
Risk
AuthenticationMethod
AuthenticationResult
```

y establecer las relaciones, invariantes y estados válidos entre estos objetos.
