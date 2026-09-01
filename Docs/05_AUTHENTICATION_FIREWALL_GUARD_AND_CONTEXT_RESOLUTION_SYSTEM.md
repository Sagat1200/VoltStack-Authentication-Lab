# VoltStack Authentication System

## 05 — Authentication Firewall, Guard and Context Resolution System

- **Archivo:** `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del sistema de firewall, guards y resolución de contexto  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de determinar:

> **Qué configuración de Authentication debe aplicarse a una ejecución concreta y cómo se expone ese contexto al desarrollador.**

VoltStack combinará tres conceptos diferentes:

```text
Authentication Firewall
Guard
Authentication Context Resolution
```

Cada uno tendrá una función distinta.

La arquitectura deberá evitar confundirlos.

---

## 2. Separación conceptual

La relación será:

```text
Request / Execution
        │
        ▼
Authentication Firewall Resolver
        │
        ▼
Authentication Firewall
        │
        ├── Identity Provider
        ├── Authenticators
        ├── State Mode
        ├── Policies
        ├── Session Strategy
        └── Token Strategy
        │
        ▼
Authentication Manager
        │
        ▼
AuthenticationContext
        │
        ▼
Guard / Auth Facade
```

Por tanto:

```text
Firewall
    define qué configuración de Authentication aplica

Guard
    ofrece una API de acceso conveniente

AuthenticationContext
    representa la autenticación actual
```

---

## 3. Influencia arquitectónica

VoltStack tomará:

### 3.1 De Laravel

```text
guards con nombre
Auth::guard()
Auth::user()
Auth::check()
Auth::guest()
ergonomía
```

### 3.2 De Symfony

```text
security firewalls
request matching
multiple authenticators
stateful/stateless configuration
provider selection
firewall-specific behavior
```

### 3.3 Propio de VoltStack

```text
firewall + guard separation
transport-independent matching
tenant-aware firewall resolution
compiled match graph
context-scoped guards
persistent-runtime isolation
hybrid state mode
authentication assurance policies
machine/service firewall support
```

---

## 4. Authentication Firewall

Un `AuthenticationFirewall` representa una configuración coherente de autenticación aplicable a un determinado contexto de ejecución.

No es un firewall de red.

No bloquea paquetes IP.

Representa una:

> **frontera lógica de Authentication.**

---

## 5. Responsabilidades del Firewall

Un firewall podrá definir:

```text
name
matcher
priority
identity provider
authenticators
authentication mode
state mode
session strategy
token strategy
remember-me strategy
policies
assurance requirements
risk profile
challenge strategy
success handlers
failure handlers
tenant behavior
```

---

## 6. FirewallInterface

Contrato conceptual:

```php
interface AuthenticationFirewallInterface
{
    public function name(): string;

    public function stateMode(): AuthenticationStateMode;

    public function identityProvider(): string;

    public function authenticators(): array;

    public function policies(): array;
}
```

La definición real podrá dividirse en objetos especializados.

---

## 7. FirewallDefinition

Será preferible representar configuración mediante un objeto immutable:

```php
final readonly class AuthenticationFirewallDefinition
{
    public function __construct(
        public string $name,
        public AuthenticationFirewallMatcherInterface $matcher,
        public AuthenticationStateMode $stateMode,
        public string $identityProvider,
        public array $authenticators,
        public array $policies = [],
        public int $priority = 0,
    ) {}
}
```

---

## 8. Firewall Matching

La selección deberá basarse en metadata normalizada.

Podrá considerar:

```text
transport
route
path
host
subdomain
HTTP method
tenant
application
protocol
execution type
route metadata
custom attributes
```

---

## 9. Matchers básicos

VoltStack podrá proporcionar:

```text
PathMatcher
RouteMatcher
HostMatcher
MethodMatcher
TransportMatcher
TenantMatcher
ApplicationMatcher
AttributeMatcher
CompositeMatcher
```

---

## 10. Composite Matching

Las reglas podrán combinarse.

Ejemplo:

```text
host = admin.example.com
AND
path startsWith /admin
AND
transport = HTTP
```

---

## 11. Matcher contract

```php
interface AuthenticationFirewallMatcherInterface
{
    public function matches(
        AuthenticationRequest $request
    ): bool;
}
```

El matcher deberá ser:

```text
deterministic
side-effect free
fast
secret-free
```

---

## 12. No authentication inside matcher

Un matcher no deberá:

```text
query user
verify password
validate token
create session
perform authorization
```

Su única responsabilidad es determinar si una definición aplica.

---

## 13. FirewallResolver

El:

```text
AuthenticationFirewallResolver
```

será responsable de elegir la definición correcta.

Contrato conceptual:

```php
interface AuthenticationFirewallResolverInterface
{
    public function resolve(
        AuthenticationRequest $request
    ): AuthenticationFirewallResolution;
}
```

---

## 14. FirewallResolution

Podrá representar:

```text
MATCHED
DEFAULT
NONE
AMBIGUOUS
ERROR
```

---

## 15. Deterministic precedence

Cuando varias definiciones coincidan, VoltStack deberá utilizar reglas deterministas.

Posible orden:

```text
1. explicit priority
2. specificity
3. route-specific match
4. host-specific match
5. path-specific match
6. declaration order as final tie-breaker
```

La implementación deberá documentar el criterio exacto.

---

## 16. Specificity

Ejemplo:

```text
Firewall A:
    path = /*

Firewall B:
    path = /admin/*
```

Para `/admin/users`:

```text
Firewall B
```

deberá ser más específico.

---

## 17. Explicit priority

Configuración conceptual:

```php
'firewalls' => [
    'admin' => [
        'priority' => 100,
    ],

    'web' => [
        'priority' => 0,
    ],
];
```

La prioridad deberá utilizarse solo cuando sea necesaria.

---

## 18. Ambiguous match

Si dos firewalls presentan la misma prioridad y especificidad incompatible:

```text
AMBIGUOUS_FIREWALL
```

deberá poder producir:

```text
configuration error
```

en vez de seleccionar uno aleatoriamente.

---

## 19. Compile-time ambiguity detection

Muchas ambigüedades podrán detectarse durante compilación.

Ejemplo:

```text
same host
same path
same transport
same priority
```

VoltStack deberá advertir o rechazar la configuración.

---

## 20. Default Firewall

Podrá existir un firewall por defecto.

Ejemplo:

```php
'default' => 'web',
```

Este se utilizará cuando:

```text
no more specific firewall matches
```

si la configuración lo permite.

---

## 21. Public execution

También deberá ser posible que una request:

```text
no firewall required
```

o:

```text
public firewall
```

sea procesada sin exigir identidad.

---

## 22. Public does not mean disabled security

Un recurso público aún podrá utilizar Authentication de forma lazy.

Ejemplo:

```text
home page

guest:
    anonymous content

authenticated:
    personalized content
```

---

## 23. Firewall Modes

Un firewall podrá trabajar en:

```text
STATEFUL
STATELESS
HYBRID
```

---

## 24. STATEFUL

Ejemplo típico:

```text
browser web application
```

Utiliza:

```text
session
cookies
remember-me
context recovery
```

---

## 25. STATELESS

Ejemplo:

```text
REST API
```

Cada request deberá establecer autenticación nuevamente mediante:

```text
bearer token
API key
mTLS
service credential
```

No se mantiene Session Authentication entre requests.

---

## 26. HYBRID

Puede aceptar:

```text
session
+
bearer token
```

o distintos mecanismos según canal.

Ejemplo:

```text
VoltStack SPA
```

puede usar:

```text
HTTP session
+
structured SPA authentication protocol
```

---

## 27. Hybrid ambiguity rules

Si un firewall acepta múltiples mecanismos:

```text
Session
Bearer
API Key
```

deberá existir una precedencia explícita.

Ejemplo:

```text
explicit bearer credential
    >
session
```

o política distinta.

---

## 28. Credential conflict

Supongamos:

```text
session → Alice
bearer  → Bob
```

VoltStack nunca deberá fusionarlos silenciosamente.

El resultado podría ser:

```text
AUTHENTICATION_CREDENTIAL_CONFLICT
```

según política.

---

## 29. Firewall Identity Provider

Cada firewall podrá seleccionar un provider por defecto:

```text
web
    provider = users

admin
    provider = administrators

service
    provider = services
```

---

## 30. Multiple Identity Providers

También podrá permitir más de uno:

```text
corporate LDAP
database users
OIDC federated identity
```

pero la selección deberá delegarse a un `IdentityProviderResolver`.

---

## 31. Firewall Authenticators

Ejemplo:

```php
'web' => [
    'authenticators' => [
        'session',
        'password',
        'passkey',
    ],
];
```

Otro:

```php
'api' => [
    'authenticators' => [
        'bearer_token',
        'api_key',
    ],
];
```

---

## 32. Firewall Policies

Podrán aplicarse policies específicas.

Ejemplo:

```text
admin firewall
    require minimum AAL2

service firewall
    require certificate-backed credential
```

---

## 33. Authentication policy inheritance

VoltStack podrá combinar:

```text
global policies
+
firewall policies
+
tenant policies
+
operation policies
```

La resolución final deberá ser explícita.

---

## 34. Firewall Risk Profile

Un firewall podrá seleccionar:

```text
default
strict
enterprise
machine
high-security
```

como profile de riesgo.

---

## 35. Firewall Challenge Strategy

Ejemplo:

```text
web
    redirect or SPA challenge

api
    structured API challenge

CLI
    interactive prompt

service
    reject
```

---

## 36. Firewall Success Strategy

Ejemplo:

```text
web
    establish session

api
    request-scoped context

login endpoint
    establish session + rotate id

oauth exchange endpoint
    issue token
```

---

## 37. Guard

El concepto de `Guard` se mantendrá como API de conveniencia y compatibilidad conceptual.

Un Guard representa:

> una vista nombrada de una configuración y contexto de Authentication.

---

## 38. Guard no será el núcleo

En Laravel, el Guard tiene una responsabilidad central.

VoltStack no reproducirá esto literalmente.

La relación será:

```text
Guard
  ↓
AuthenticationManager
  ↓
Firewall
  ↓
AuthenticationContext
```

---

## 39. GuardInterface

Conceptualmente:

```php
interface GuardInterface
{
    public function check(): bool;

    public function guest(): bool;

    public function identity(): ?IdentityInterface;

    public function context(): ?AuthenticationContext;

    public function logout(): AuthenticationResult;
}
```

---

## 40. Extended Guard APIs

Algunas implementaciones podrán incluir:

```text
attempt()
login()
reauthenticate()
stepUp()
session()
```

pero no todos los Guard deberán soportar todas las capacidades.

---

## 41. Capability-aware guards

Ejemplo:

```text
SessionGuardAdapter
    supports login/logout/session

TokenGuardAdapter
    supports token-based authentication

ServiceGuardAdapter
    may not support interactive login
```

Podrá existir:

```text
GuardCapability
```

para describir las operaciones permitidas.

---

## 42. Named Guards

Configuración:

```php
'guards' => [
    'web' => [
        'firewall' => 'web',
    ],

    'admin' => [
        'firewall' => 'admin',
    ],

    'api' => [
        'firewall' => 'api',
    ],
];
```

---

## 43. Guard aliases

Podrá ser posible:

```text
guard web
    maps to firewall web
```

pero no es obligatorio que exista una relación uno-a-uno.

---

## 44. Guard configuration profiles

Por ejemplo:

```text
Guard:
    admin

Firewall:
    shared-web

Additional guard configuration:
    provider = administrators
```

No obstante, debe evitarse complejidad innecesaria.

La recomendación inicial será:

```text
named guard → named firewall profile
```

cuando resulte posible.

---

## 45. Auth::guard()

API:

```php
Auth::guard('admin');
```

deberá retornar un objeto `GuardInterface`.

---

## 46. Default guard

```php
Auth::check();
```

utilizará el:

```text
active/default Guard
```

según contexto.

---

## 47. Active Guard Resolution

El Guard activo podrá determinarse desde:

```text
resolved firewall
route metadata
explicit developer selection
default configuration
```

---

## 48. Explicit guard selection

Ejemplo:

```php
Auth::guard('admin')->identity();
```

No deberá cambiar silenciosamente el contexto global de la request.

---

## 49. Guard scoping

El objeto:

```php
$admin = Auth::guard('admin');
```

deberá ser una vista scoped/configurada.

No:

```text
set global current guard = admin
```

salvo operación explícita y controlada.

---

## 50. Multiple guards in same request

VoltStack podrá permitir consultar varios guards.

Ejemplo:

```php
Auth::guard('web')->check();

Auth::guard('service')->check();
```

pero cada uno deberá mantener contextos claramente separados si representan autenticaciones distintas.

---

## 51. Primary Authentication Context

Una request normalmente tendrá un:

```text
PrimaryAuthenticationContext
```

utilizado por:

```php
Auth::identity();
```

---

## 52. Secondary contexts

Casos avanzados podrían requerir:

```text
primary user context
+
service context
+
delegated context
```

Estos no deberán mezclarse.

---

## 53. ContextKey

Podrá existir:

```text
AuthenticationContextKey
```

por ejemplo:

```text
primary
guard:web
guard:admin
delegated
service
```

---

## 54. AuthenticationContextRegistry

En lugar de almacenar un único contexto podrá existir internamente:

```text
AuthenticationContextRegistry
```

con:

```text
context key → AuthenticationContext
```

---

## 55. ContextRegistry safety

El registry deberá ser:

```text
request/execution scoped
```

Nunca global al worker.

---

## 56. ContextStorage contract

Conceptualmente:

```php
interface AuthenticationContextStorageInterface
{
    public function get(
        AuthenticationContextKey $key
    ): ?AuthenticationContext;

    public function set(
        AuthenticationContextKey $key,
        AuthenticationContext $context
    ): void;

    public function remove(
        AuthenticationContextKey $key
    ): void;

    public function clear(): void;
}
```

---

## 57. Primary context alias

`Auth::context()` podrá equivaler a:

```text
context registry
    key = primary
```

---

## 58. Context Resolver

El:

```text
AuthenticationContextResolver
```

deberá determinar qué contexto usar para una operación.

Inputs:

```text
explicit guard
active firewall
route metadata
execution metadata
context key
```

---

## 59. ContextResolution

Posibles resultados:

```text
FOUND
NOT_FOUND
MULTIPLE
CONFLICT
EXPIRED
INVALID
```

---

## 60. Lazy Context Resolution

`Auth::identity()` podrá disparar recuperación lazy si todavía no se resolvió el contexto.

Flujo:

```text
Auth::identity()
      ↓
ContextResolver
      ↓
no request context
      ↓
AuthenticationRecoveryManager
      ↓
session/token recovery
      ↓
AuthenticationContext
```

---

## 61. Lazy recovery memoization

Después:

```php
Auth::identity();
Auth::identity();
Auth::identity();
```

no deberá repetir el recovery.

---

## 62. Lazy failure memoization

Si no existe autenticación:

```text
NO_AUTHENTICATION
```

también podrá memoizarse durante la request para evitar comprobaciones repetidas.

---

## 63. Eager Context Resolution

Middleware podrá forzar:

```text
Auth context resolve before controller
```

para rutas protegidas.

---

## 64. AuthenticationRequired metadata

Routing podrá adjuntar:

```text
authentication.required = true
```

---

## 65. Guard metadata

También:

```text
authentication.guard = admin
```

---

## 66. Firewall hint metadata

Opcionalmente:

```text
authentication.firewall = admin
```

pero la configuración deberá evitar duplicación/conflictos con matching.

---

## 67. Assurance metadata

Ejemplo:

```text
authentication.assurance = AAL2
```

Este dato no selecciona por sí solo permisos; indica requisito de Authentication.

---

## 68. Route example

Conceptualmente:

```php
Route::get('/admin', AdminController::class)
    ->auth('admin')
    ->assurance('AAL2');
```

Internamente:

```text
route metadata
     ↓
authentication integration
```

---

## 69. Firewall vs route auth middleware

No deberán ser equivalentes.

El firewall responde:

> ¿Qué configuración de Authentication se aplica?

El middleware responde:

> ¿Debe esta request requerir Authentication antes de continuar?

---

## 70. Public route under authenticated firewall

Ejemplo:

```text
firewall = web
route = /
authentication required = false
```

Aun así, el sistema puede recuperar identidad lazy.

---

## 71. Protected route under firewall

```text
firewall = web
route = /dashboard
authentication required = true
```

requiere contexto válido.

---

## 72. Firewall state isolation

Dos firewalls no deberán compartir estado accidentalmente.

Ejemplo:

```text
admin
web
```

Una sesión `web` no deberá autenticar automáticamente `admin` salvo configuración explícita.

---

## 73. Session namespace

Podrán usarse claves específicas:

```text
auth.web
auth.admin
```

dentro de la sesión.

---

## 74. Session cookie isolation

Aplicaciones de alta seguridad podrán usar:

```text
different cookies
different cookie paths
different domains
different SameSite policies
```

por firewall.

---

## 75. Shared session configuration

También podrá configurarse:

```text
web + admin share authentication session
```

pero será una decisión explícita.

---

## 76. Authentication Realm

Podrá introducirse el concepto:

```text
AuthenticationRealm
```

para representar un dominio lógico de identidad/autenticación compartido.

Ejemplo:

```text
Realm: employees
Firewalls:
    intranet
    admin
```

---

## 77. Realm benefits

Un realm podría centralizar:

```text
identity provider
session namespace
credential policies
SSO scope
```

aunque deberá evitar duplicar innecesariamente `Firewall`.

---

## 78. Realm vs Firewall

```text
Realm
    who / identity domain

Firewall
    where / execution boundary
```

Este concepto podrá reservarse para una evolución posterior si no es necesario en V1.

---

## 79. Multi-Tenant Firewall Resolution

VoltStack deberá soportar:

```text
tenant-aware matching
```

Ejemplo:

```text
tenant = acme
host = acme.app.com
```

---

## 80. Tenant resolution ordering

El sistema deberá decidir si:

```text
Tenant Resolution
    precedes
Firewall Resolution
```

o si el firewall ayuda a resolver tenant.

La recomendación base será:

```text
minimal tenant/application context resolution
       ↓
firewall resolution
       ↓
full authentication
```

---

## 81. Tenant hint is untrusted

Host/subdomain/headers pueden proporcionar:

```text
tenant hint
```

pero el tenant efectivo deberá validarse.

---

## 82. Tenant-bound firewall

Ejemplo:

```text
tenant-admin
```

podrá requerir:

```text
tenant context exists
identity provider scoped to tenant
session bound to tenant
```

---

## 83. Cross-tenant session protection

Si una AuthenticationSession contiene:

```text
tenant = acme
```

y la request resuelve:

```text
tenant = globex
```

entonces:

```text
session must not authenticate
```

salvo identidad global explícitamente soportada.

---

## 84. Global identities

Podrán existir:

```text
platform administrators
system services
```

capaces de operar en múltiples tenants.

Aun así, deberá distinguirse:

```text
identity scope
```

de:

```text
current tenant context
```

---

## 85. Tenant switching

Cambiar tenant no deberá modificar silenciosamente:

```text
AuthenticationContext
```

si el contexto está tenant-bound.

Puede requerir:

```text
new context
session rebinding
step-up
explicit tenant switch flow
```

---

## 86. Domain/Host Firewall

Ejemplo:

```text
app.example.com
    web

admin.example.com
    admin

api.example.com
    api
```

---

## 87. Path Firewall

Ejemplo:

```text
/admin/*
    admin

/api/*
    api

/*
    web
```

---

## 88. Route-name Firewall

Más robusto en algunas aplicaciones:

```text
route prefix = admin.
```

en lugar de depender solo de path.

---

## 89. Transport Firewall

Ejemplo:

```text
HTTP
WebSocket
CLI
Queue
```

pueden tener perfiles distintos.

---

## 90. WebSocket firewall

Podrá seleccionar:

```text
websocket
```

y definir:

```text
token authenticator
connection-scoped context
reauthentication policy
```

---

## 91. CLI firewall

Ejemplo:

```text
console
```

podrá usar:

```text
service identity
operator identity
development interactive auth
```

---

## 92. Queue firewall

Ejemplo:

```text
queue
```

podrá definir:

```text
worker service identity
delegated subject validation
```

---

## 93. Internal service firewall

```text
internal
```

podrá requerir:

```text
mTLS
service token
workload identity
```

---

## 94. API firewall

Recomendación habitual:

```text
stateless = true
authenticators:
    bearer
    api_key
```

---

## 95. SPA firewall

Puede ser:

```text
hybrid
session-based
CSRF-aware
structured challenge responses
```

---

## 96. Admin firewall

Podrá configurar:

```text
stateful
provider = administrators
minimum AAL2
passkey or MFA
strict risk policy
shorter session timeout
```

---

## 97. FirewallConfig object

La resolución debería entregar un objeto listo para runtime:

```text
ResolvedAuthenticationFirewall
```

en lugar de consultar config arrays repetidamente.

---

## 98. Resolved Firewall

Podrá contener referencias ya resueltas a:

```text
provider descriptor
authenticator descriptors
policy descriptors
state strategy
challenge strategy
```

---

## 99. Firewall compilation

Pipeline:

```text
Raw Configuration
      ↓
Firewall Definition Builder
      ↓
Validation
      ↓
Matcher Compilation
      ↓
Resolved Service References
      ↓
CompiledAuthenticationFirewallRegistry
```

---

## 100. Compiled Matcher Graph

Podrá optimizar matching mediante:

```text
host index
route index
path trie
transport index
priority table
```

en lugar de iterar todos los firewalls.

---

## 101. Matching complexity

Objetivo:

```text
near O(1)
or
O(log n)
```

para rutas comunes cuando existan muchos firewalls.

---

## 102. Avoid regex overuse

Un sistema basado exclusivamente en regex puede resultar:

```text
hard to optimize
hard to debug
easy to misconfigure
```

VoltStack deberá soportar matchers estructurados.

---

## 103. Regex Matcher

Podrá existir para casos avanzados, pero no será necesariamente la opción recomendada.

---

## 104. Firewall Debugger

En desarrollo podrá existir una herramienta que explique:

```text
request:
    /admin/users

matched:
    admin

reason:
    host matched
    path prefix matched
    priority 100

other candidates:
    web
```

---

## 105. Debug safety

Los diagnostics no deberán revelar:

```text
secrets
raw credentials
token values
```

---

## 106. Guard Resolver

Podrá existir:

```text
GuardResolver
```

responsable de convertir:

```text
guard name
```

en:

```text
Guard instance / adapter
```

---

## 107. GuardRegistry

```text
GuardRegistry
│
├── web
├── admin
├── api
└── service
```

---

## 108. Guard factories

Guards podrán ser creados mediante factories:

```text
SessionGuardAdapterFactory
StatelessGuardAdapterFactory
GenericGuardFactory
```

aunque debe evitarse replicar la complejidad interna de Laravel sin necesidad.

---

## 109. Generic Guard recommendation

La implementación preferida puede ser:

```text
ConfiguredGuard
```

con:

```text
guard name
firewall name
context key
manager
```

evitando numerosas clases de Guard.

---

## 110. ConfiguredGuard

Conceptualmente:

```php
final class ConfiguredGuard implements GuardInterface
{
    public function __construct(
        private readonly string $name,
        private readonly string $firewall,
        private readonly AuthenticationManagerInterface $manager,
        private readonly AuthenticationContextResolverInterface $contexts,
    ) {}
}
```

---

## 111. Guard identity resolution

```php
Auth::guard('admin')->identity();
```

flujo:

```text
Guard
    ↓
ContextResolver(admin)
    ↓
existing context?
    ├── yes → identity
    └── no
         ↓
       recover admin authentication
```

---

## 112. Guard check()

```php
Auth::guard('web')->check();
```

deberá significar:

```text
valid AuthenticationContext exists for guard
```

No simplemente:

```text
session contains user_id
```

---

## 113. Guard guest()

```php
Auth::guard('web')->guest();
```

equivale a:

```text
!check()
```

salvo semántica especializada.

---

## 114. Guard id()

Podrá existir:

```php
Auth::guard('web')->id();
```

que devolverá:

```text
IdentityIdentifier|null
```

o valor normalizado equivalente.

---

## 115. Guard user()

Para familiaridad:

```php
Auth::guard('web')->user();
```

será válido cuando la identidad actual represente un usuario humano.

---

## 116. Non-user identity handling

Si:

```text
ServiceIdentity
```

es la identidad actual, `user()` deberá:

```text
return null
```

o lanzar una excepción tipada según diseño.

La API recomendada general será:

```php
identity()
```

---

## 117. Guard context()

```php
Auth::guard('admin')->context();
```

devuelve el `AuthenticationContext` asociado.

---

## 118. Guard assurance()

Podrá exponer:

```php
Auth::guard('admin')->assurance();
```

para leer el nivel alcanzado.

---

## 119. Guard authenticate operation

Un Guard podrá iniciar un flow explícito:

```php
Auth::guard('web')->attempt($credentials);
```

internamente:

```text
Guard
   ↓
AuthenticationOperation bound to firewall web
   ↓
Manager
```

---

## 120. Explicit firewall binding

Esto evita que el Manager vuelva a resolver un firewall diferente.

La operación deberá indicar:

```text
requestedFirewall = web
```

y validar que el contexto permite utilizarlo.

---

## 121. Guard privilege confusion prevention

Una request bajo firewall `web` no deberá poder autenticarse inadvertidamente contra `admin` solo porque la aplicación invocó mal:

```php
Auth::guard('admin')->attempt(...)
```

Podrán existir políticas de:

```text
allowed explicit guard switching
```

según entorno.

---

## 122. Programmatic guard use

Fuera de HTTP, la selección explícita sí puede ser necesaria:

```text
CLI
testing
workers
```

Por ello no deberá prohibirse globalmente.

---

## 123. Context Resolution Order

Orden recomendado:

```text
1. explicit context/guard requested
2. execution-scoped primary context
3. active firewall mapping
4. route metadata
5. configured default
```

---

## 124. No accidental fallback after failure

Si el firewall `admin` encuentra una sesión inválida:

```text
admin authentication failed
```

VoltStack no deberá hacer:

```text
fallback to web
```

y considerar autenticado al usuario sin una política explícita.

---

## 125. Fallback rules

Fallback solo deberá utilizarse para:

```text
resolution
```

no para reducir requisitos de seguridad.

Ejemplo permitido:

```text
no route-specific firewall
    ↓
default web firewall
```

Ejemplo peligroso:

```text
admin MFA failed
    ↓
fallback web authentication
```

---

## 126. Anonymous/public context

Como regla general:

```text
unauthenticated
    =
no AuthenticationContext
```

No se creará automáticamente una `AnonymousIdentity`.

---

## 127. Optional anonymous adapter

Si algún subsystem necesita principal anónimo:

```text
AnonymousPrincipalAdapter
```

podrá generarlo sin afirmar autenticación real.

---

## 128. Context status

Podrá existir un estado:

```text
AuthenticationContextStatus
```

con:

```text
ACTIVE
EXPIRED
REVOKED
STALE
INVALID
```

aunque un Context inmutable activo normalmente será eliminado cuando deje de ser válido.

---

## 129. Stale Context

Ejemplo:

```text
session restored
identity security version changed
```

El contexto reconstruido deberá considerarse:

```text
STALE / INVALID
```

y no instalarse.

---

## 130. Context freshness resolution

Antes de utilizar una operación sensible podrá comprobarse:

```text
authentication age
last step-up
current assurance
```

Esto puede producir:

```text
STEP_UP_REQUIRED
```

sin invalidar la sesión base.

---

## 131. Context Validation

Podrá existir:

```text
AuthenticationContextValidator
```

responsable de verificar:

```text
expiration
security version
tenant binding
device binding
session validity
credential invalidation
```

según estrategia.

---

## 132. Context Resolver vs Validator

```text
Resolver
    finds the relevant context

Validator
    determines whether it remains valid
```

---

## 133. Context recovery

Si no hay contexto activo, Resolver podrá coordinar:

```text
session recovery
token recovery
remember-me recovery
```

---

## 134. Recovery policy per firewall

Ejemplo:

```text
web:
    session
    remember-me

api:
    bearer

service:
    mTLS
```

---

## 135. Remember-Me Context

Remember-me podrá producir un contexto con:

```text
lower freshness
lower assurance
persistent authentication provenance
```

según política.

---

## 136. Authentication source

El Context deberá registrar origen:

```text
session
remember_me
bearer_token
passkey
OIDC
service_credential
```

---

## 137. Context reconstruction optimization

Una sesión puede almacenar:

```text
identity id
provider id
assurance
method
security version
tenant
timestamps
```

evitando reconstruir el login completo.

---

## 138. Identity refresh strategy

Cada firewall podrá definir:

```text
always
never
version_based
interval_based
security_sensitive
```

para refrescar Identity.

---

## 139. Session snapshot

Una implementación podría guardar atributos suficientes para evitar DB lookup en cada request.

Sin embargo, deberá considerar:

```text
account disabled
security version changed
tenant revoked
```

---

## 140. Identity version checks

Una estrategia eficiente:

```text
session.security_version
        vs
identity.security_version
```

permite invalidación rápida.

---

## 141. Context cache

Solo podrá cachearse de manera segura dentro del scope de request.

Cross-request context caching requerirá controles estrictos y probablemente no será necesario en V1.

---

## 142. Context inheritance

Una ejecución hija no deberá heredar automáticamente toda autenticación.

Ejemplo:

```text
async task
```

debe recibir una delegación explícita si necesita identidad.

---

## 143. Context stack

Para impersonation o delegation podría existir internamente:

```text
AuthenticationContextStack
```

Ejemplo:

```text
base actor
    ↓
impersonated subject
```

pero no deberá utilizarse para resolver guards comunes.

---

## 144. Actor Context

Puede ser distinto de:

```text
Effective Subject Context
```

cuando exista impersonation.

Esto se desarrollará en documentación específica.

---

## 145. Context binding to request

Un contexto podrá asociarse a:

```text
request id
execution id
connection id
```

cuando sea útil para detectar uso fuera de scope.

---

## 146. Persistent runtime safeguards

Un ContextStorage deberá poder comprobar:

```text
current scope id
```

antes de devolver estado.

---

## 147. Example persistent-runtime bug

Incorrecto:

```text
Worker singleton:

currentContext = Alice

Request completes

currentContext remains Alice

next request by Bob
```

Este patrón estará explícitamente prohibido.

---

## 148. Correct persistent-runtime model

```text
Singleton Manager
      │
      ├── immutable config
      └── scope accessor
              │
              ▼
        Request Scope
              │
              ▼
      AuthenticationContext
```

---

## 149. Scope accessor

Podrá existir:

```text
AuthenticationScopeAccessor
```

para localizar el scope actual de forma segura.

---

## 150. Thread/fiber isolation

Si existen ejecuciones concurrentes:

```text
Fiber A → Alice
Fiber B → Bob
```

cada una deberá resolver su propio Context.

---

## 151. Firewall Registry

El:

```text
AuthenticationFirewallRegistry
```

almacenará todas las definiciones compiladas.

---

## 152. Registry interface

```php
interface AuthenticationFirewallRegistryInterface
{
    public function get(string $name): AuthenticationFirewallDefinition;

    public function has(string $name): bool;
}
```

---

## 153. Immutable registry

Después del bootstrap:

```text
registry frozen
```

en producción.

Esto mejora:

```text
predictability
performance
worker safety
```

---

## 154. Dynamic registration

En desarrollo/plugins podrán registrarse firewalls durante bootstrap.

No deberán añadirse arbitrariamente después de procesar requests salvo entorno explícitamente dinámico.

---

## 155. Guard Registry lifecycle

Igualmente:

```text
GuardRegistry
```

podrá congelarse tras bootstrap.

---

## 156. Firewall aliases

Podrán existir aliases:

```text
default_web → web
```

pero deberán resolverse durante compilación.

---

## 157. Configuration validation

Se deberá validar:

```text
unique firewall names
valid guard references
provider exists
authenticator exists
state strategy valid
session strategy compatible
matcher valid
policy references valid
```

---

## 158. Stateful consistency validation

Ejemplo inválido:

```text
state_mode = STATELESS
session_strategy = required
```

salvo una semántica híbrida explícita.

---

## 159. Authenticator compatibility validation

Ejemplo:

```text
CLI firewall
authenticator = browser_session_cookie
```

podrá advertirse como incompatible.

---

## 160. Transport capability metadata

Authenticators podrán declarar:

```text
supported transports
```

para detectar errores de configuración.

---

## 161. Firewall capability metadata

Ejemplo:

```text
supports_session
supports_challenge
supports_interactive_login
supports_token_issuance
supports_remember_me
```

---

## 162. Compiled Guard Mapping

En producción:

```text
guard:web → firewall:web
guard:admin → firewall:admin
```

puede convertirse en lookup directo.

---

## 163. Context lookup optimization

`Auth::check()` debería aproximarse a:

```text
scope lookup
   ↓
primary context
```

después de la resolución inicial.

No deberá recorrer firewalls repetidamente.

---

## 164. Firewall resolution memoization

Durante una request:

```text
FirewallResolver
```

se ejecutará una sola vez salvo que el contexto de routing cambie explícitamente.

---

## 165. Route finalization

Para HTTP, firewall resolution definitivo deberá ocurrir cuando exista suficiente metadata de route si esta forma parte del matcher.

Puede haber dos fases:

```text
early host/path hint
       ↓
routing
       ↓
final firewall resolution
```

si la arquitectura HTTP lo requiere.

---

## 166. Early authentication constraints

Bearer token processing antes del routing puede resultar útil, pero no deberá ignorar policies dependientes de route.

La arquitectura deberá permitir:

```text
credential discovery early
final policy after route resolution
```

---

## 167. Firewall pre-resolution

Podrá existir un:

```text
PreAuthenticationFirewallHint
```

basado en host/path.

Luego:

```text
FinalAuthenticationFirewallResolution
```

con route metadata.

---

## 168. Consistency check

Si pre-resolution y final resolution producen firewalls incompatibles:

```text
fail closed
```

o reiniciar el flujo de manera controlada.

Nunca reutilizar contexto bajo un firewall incorrecto.

---

## 169. Firewall transitions

Durante una misma request, cambiar firewall después de establecer AuthenticationContext debería estar prohibido normalmente.

---

## 170. Subrequests

Si VoltStack soporta subrequests:

```text
parent request
   ↓
subrequest
```

deberá definirse si:

```text
inherits context
re-resolves firewall
```

La recomendación:

```text
inherit trusted parent context explicitly
+
validate subrequest requirements
```

---

## 171. Internal forwarding

Un internal forward de ruta pública a admin no deberá conservar automáticamente una autenticación insuficiente.

---

## 172. Firewall assurance requirements

Cada firewall podrá definir:

```text
minimum_assurance
```

Ejemplo:

```text
admin:
    minimum AAL2
```

---

## 173. Route higher assurance

Una ruta puede requerir más que el firewall.

Ejemplo:

```text
admin firewall = AAL2
financial operation = fresh AAL3
```

---

## 174. Effective requirement

Será combinación de:

```text
global
firewall
route
operation
risk
```

usando la política más estricta aplicable según reglas definidas.

---

## 175. Guard assurance semantics

```php
Auth::guard('admin')->check();
```

deberá indicar autenticación válida para ese Guard.

Si el Context existe pero no cumple el assurance requerido por el Guard, podrá devolver:

```text
false
```

o un estado más rico mediante:

```php
status()
```

La semántica deberá ser consistente.

---

## 176. GuardStatus

Podría existir:

```text
AUTHENTICATED
UNAUTHENTICATED
STEP_UP_REQUIRED
EXPIRED
INVALID
```

---

## 177. check() simplification

Para ergonomía:

```php
check(): bool
```

solo devuelve `true` cuando el Guard considera el contexto usable.

Para detalles:

```php
status()
context()
```

---

## 178. Firewall Status Debugging

Podrá exponerse en tooling de desarrollo:

```text
selected firewall
selected guard
state mode
provider
authenticator
current context
assurance
```

sin secrets.

---

## 179. Auth Facade behavior

La facade deberá resolver:

```text
current scope
current primary guard
current context
```

de forma lazy.

---

## 180. Facade does not store state

`Auth` será un facade/proxy.

No contendrá:

```text
static current user
static session
static token
```

---

## 181. Auth::identity()

Flujo:

```text
Facade
  ↓
AuthenticationManager
  ↓
Primary Context Resolver
  ↓
Identity
```

---

## 182. Auth::context()

Retorna:

```text
AuthenticationContext|null
```

---

## 183. Auth::check()

Equivale conceptualmente a:

```text
resolved valid primary context exists
```

---

## 184. Auth::guest()

Equivale a:

```text
!Auth::check()
```

---

## 185. Auth::guard()

Crea o recupera un proxy configurado:

```text
ConfiguredGuard
```

---

## 186. Guard object caching

Los Guard proxies pueden ser reutilizados porque deberán ser stateless respecto al usuario actual.

El contexto actual se consulta siempre mediante Scope/ContextResolver.

---

## 187. Guard singleton safety

Un `ConfiguredGuard` singleton es seguro solo si almacena:

```text
configuration
service references
guard name
```

y no:

```text
current identity
current context
current session
```

---

## 188. Testing guards

Testing podrá usar:

```php
$this->actingAs($user, guard: 'admin');
```

Esto instalará un Context de test en:

```text
AuthenticationContextRegistry
key = guard:admin
```

y, si corresponde:

```text
primary
```

---

## 189. Testing firewall resolution

Deberán existir tests para:

```text
host matching
path matching
route matching
transport matching
priority
ambiguity
default firewall
tenant binding
```

---

## 190. Property-based matching tests

El matcher es candidato ideal para:

```text
property-based testing
```

por combinaciones extensas de:

```text
host/path/method/route
```

---

## 191. Persistent runtime tests

Deberá comprobarse:

```text
Request A → user A
reset
Request B → guest
```

y:

```text
Request A → admin guard
Request B → web guard
```

sin leakage.

---

## 192. Security tests

Casos:

```text
admin session used on web
web session used on admin
tenant A session on tenant B
conflicting bearer/session identity
ambiguous firewall match
stale context
revoked session
```

---

## 193. Performance tests

Medir:

```text
firewall resolution latency
context lookup latency
lazy recovery latency
number of matcher evaluations
number of provider calls
```

---

## 194. Firewall observability

Span recomendado:

```text
auth.firewall.resolve
```

atributos seguros:

```text
firewall.name
match.type
state.mode
transport
```

---

## 195. Guard observability

No será necesario crear span por cada:

```php
Auth::check()
```

si solo hace lookup local.

Esto evitará ruido.

---

## 196. Context recovery observability

Sí será útil:

```text
auth.context.recover
```

con:

```text
source = session/token/remember_me
outcome
duration
```

---

## 197. Audit implications

La selección normal de firewall no necesita siempre un Audit Record.

Pero eventos relevantes sí:

```text
cross-firewall credential conflict
cross-tenant context rejection
admin authentication
service identity authentication
```

---

## 198. Firewall error categories

Podrán existir:

```text
FirewallNotFound
FirewallAmbiguous
FirewallConfigurationInvalid
GuardNotFound
GuardFirewallMismatch
AuthenticationContextConflict
AuthenticationContextInvalid
```

---

## 199. Safe production mapping

Estos errores de infraestructura/configuración no deberán exponerse con detalles internos al cliente.

---

## 200. Fail-closed rule

Ante:

```text
unknown firewall
ambiguous critical firewall
context mismatch
tenant mismatch
guard conflict
```

si la ruta es protegida:

```text
do not authenticate
```

---

## 201. Unknown guard

```php
Auth::guard('missing');
```

deberá producir una excepción clara para desarrolladores:

```text
GuardNotFoundException
```

No fallback silencioso al default.

---

## 202. Unknown firewall config

También:

```text
configured guard references missing firewall
```

debe fallar durante bootstrap si es posible.

---

## 203. Dynamic tenant firewall configuration

VoltStack deberá evitar crear un firewall completo por tenant si miles de tenants usan la misma política.

Preferir:

```text
shared firewall definition
+
tenant-aware context/policy
```

---

## 204. Per-tenant policy override

Puede existir:

```text
tenant authentication profile
```

sin duplicar matchers.

---

## 205. Firewall templates

Podrán definirse profiles:

```text
web
enterprise-web
strict-admin
machine-api
```

que generen definiciones compiladas.

---

## 206. Configuration inheritance

Si se soporta herencia:

```text
admin extends web
```

deberá resolverse durante compilación.

Runtime no deberá recorrer cadenas de herencia.

---

## 207. Inheritance safety

Las propiedades sensibles no deberán debilitarse accidentalmente.

Ejemplo:

```text
parent requires AAL2
child removes requirement
```

podrá requerir override explícito.

---

## 208. Configuration example

```php
return [
    'default_guard' => 'web',

    'guards' => [
        'web' => [
            'firewall' => 'web',
        ],

        'admin' => [
            'firewall' => 'admin',
        ],

        'api' => [
            'firewall' => 'api',
        ],
    ],

    'firewalls' => [
        'admin' => [
            'priority' => 100,
            'match' => [
                'path' => '/admin/*',
            ],
            'state' => 'stateful',
            'provider' => 'admins',
            'authenticators' => [
                'session',
                'password',
                'passkey',
            ],
            'assurance' => 'AAL2',
        ],

        'api' => [
            'priority' => 90,
            'match' => [
                'path' => '/api/*',
            ],
            'state' => 'stateless',
            'provider' => 'users',
            'authenticators' => [
                'bearer',
            ],
        ],

        'web' => [
            'priority' => 0,
            'match' => [
                'path' => '/*',
            ],
            'state' => 'stateful',
            'provider' => 'users',
            'authenticators' => [
                'session',
                'password',
                'passkey',
            ],
        ],
    ],
];
```

---

## 209. Resolution example

Request:

```text
GET /admin/users
```

Pipeline:

```text
AuthenticationRequest
       ↓
FirewallResolver
       ↓
admin
       ↓
GuardResolver
       ↓
guard admin
       ↓
ContextResolver
       ↓
session recovery
       ↓
AuthenticationContext
```

---

## 210. API example

```text
GET /api/users
Authorization: Bearer ...
```

Resolution:

```text
path /api/*
     ↓
api firewall
     ↓
stateless
     ↓
BearerTokenAuthenticator
     ↓
AuthenticationContext
```

---

## 211. Public route example

```text
GET /
```

Resolution:

```text
web firewall
    ↓
authentication optional
    ↓
lazy session recovery if needed
```

---

## 212. Admin step-up example

```text
admin firewall requires AAL2

existing admin context = AAL1
        ↓
ContextValidator
        ↓
STEP_UP_REQUIRED
```

No deberá devolver:

```text
guest
```

porque existe autenticación base válida, pero insuficiente.

---

## 213. Context conflict example

```text
web session = Alice
bearer token = Bob
hybrid firewall
```

Dependiendo de policy:

```text
explicit bearer wins
```

o:

```text
reject identity conflict
```

pero nunca:

```text
merge Alice + Bob
```

---

## 214. FirewallContext

Podrá existir un objeto:

```text
AuthenticationFirewallContext
```

que agrupe:

```text
resolved firewall
resolved guard
tenant
state mode
provider profile
authenticator profile
policy profile
```

---

## 215. FirewallContext is not AuthenticationContext

```text
AuthenticationFirewallContext
    configuration/runtime routing context

AuthenticationContext
    trusted authenticated identity state
```

---

## 216. Context resolution pipeline

```text
Execution
   ↓
Firewall Resolution
   ↓
Guard Mapping
   ↓
Context Key Resolution
   ↓
Context Registry Lookup
   │
   ├── valid context
   │       ↓
   │     return
   │
   └── absent
           ↓
       Recovery Strategy
           ↓
       Context Validation
           ↓
       Context Installation
```

---

## 217. ContextResolver contract

```php
interface AuthenticationContextResolverInterface
{
    public function resolve(
        AuthenticationContextResolutionRequest $request
    ): AuthenticationContextResolution;
}
```

---

## 218. ResolutionRequest

Podrá contener:

```text
guard
firewall
context key
allow lazy recovery
required assurance
tenant
```

---

## 219. Resolution result

```text
AuthenticationContextResolution
│
├── RESOLVED
├── ABSENT
├── STEP_UP_REQUIRED
├── EXPIRED
├── INVALID
├── CONFLICT
└── ERROR
```

---

## 220. Context resolution should not authorize

Aunque reciba:

```text
required assurance
```

no deberá evaluar:

```text
resource permissions
roles
policies de Authorization
```

---

## 221. Integration with Authorization

Authorization recibirá:

```text
AuthenticationContext
```

junto con:

```text
resource
action
environment
```

No resolverá Guards por sí mismo salvo adapter de framework.

---

## 222. AuthorizationContextBuilder

Puede consumir:

```text
primary AuthenticationContext
```

o un contexto explícito seleccionado.

---

## 223. Multiple context authorization

En delegated execution, Authorization podrá necesitar:

```text
actor
subject
```

pero eso se resolverá mediante un modelo específico, no usando cualquier Guard arbitrario.

---

## 224. Security invariant — Firewall

### AUTH-FW-01

La selección de firewall deberá ser determinista.

#### AUTH-FW-02

Un firewall no autentica por sí mismo; configura el proceso.

#### AUTH-FW-03

Matchers no verifican credenciales.

#### AUTH-FW-04

Ambigüedades críticas fallan de forma cerrada.

#### AUTH-FW-05

El estado de un firewall no deberá filtrarse accidentalmente a otro.

#### AUTH-FW-06

La identidad tenant-bound deberá respetar el tenant actual.

#### AUTH-FW-07

Una autenticación inválida no deberá provocar fallback a una política más débil.

#### AUTH-FW-08

El matching debe poder compilarse y validarse.

---

## 225. Security invariant — Guard

### AUTH-GUARD-01

Guard es una API de conveniencia, no fuente de confianza.

#### AUTH-GUARD-02

Guard no almacena directamente el usuario actual en estado global.

#### AUTH-GUARD-03

`check()` implica Context válido, no mera presencia de identificador de sesión.

#### AUTH-GUARD-04

Un Guard desconocido no debe caer al default silenciosamente.

#### AUTH-GUARD-05

Consultar otro Guard no debe cambiar el Guard primario global.

#### AUTH-GUARD-06

La API general deberá preferir `identity()` sobre `user()`.

---

## 226. Security invariant — Context

### AUTH-CTX-01

AuthenticationContext siempre pertenece a un scope explícito.

#### AUTH-CTX-02

No se expone un contexto parcial.

#### AUTH-CTX-03

Un contexto inválido o expirado no se devuelve como autenticado.

#### AUTH-CTX-04

Contextos multi-tenant deben validar binding.

#### AUTH-CTX-05

La resolución lazy debe memoizarse por scope.

#### AUTH-CTX-06

El Context nunca se almacena en static mutable state.

#### AUTH-CTX-07

Step-up insuficiente debe diferenciarse de guest.

#### AUTH-CTX-08

Cross-request leakage debe ser estructuralmente imposible.

---

## 227. Arquitectura recomendada

```text
                  AuthenticationRequest
                           │
                           ▼
               AuthenticationFirewallResolver
                           │
                           ▼
                ResolvedAuthenticationFirewall
                           │
             ┌─────────────┼──────────────┐
             ▼             ▼              ▼
          Provider     Authenticators   Policies
             │             │              │
             └─────────────┼──────────────┘
                           ▼
                AuthenticationManager
                           │
                           ▼
                AuthenticationContext
                           │
                           ▼
               AuthenticationContextRegistry
                           │
              ┌────────────┼─────────────┐
              ▼            ▼             ▼
          primary       guard:web     guard:admin
              │
              ▼
           Auth Facade
              │
        ┌─────┼──────┐
        ▼     ▼      ▼
      check identity context
```

---

## 228. Estructura sugerida

```text
src/Quantum/Auth/
│
├── Firewall/
│   ├── AuthenticationFirewallDefinition.php
│   ├── AuthenticationFirewallRegistry.php
│   ├── AuthenticationFirewallResolver.php
│   ├── AuthenticationFirewallResolution.php
│   ├── AuthenticationStateMode.php
│   │
│   └── Matcher/
│       ├── AuthenticationFirewallMatcherInterface.php
│       ├── PathMatcher.php
│       ├── HostMatcher.php
│       ├── RouteMatcher.php
│       ├── TransportMatcher.php
│       ├── TenantMatcher.php
│       └── CompositeMatcher.php
│
├── Guard/
│   ├── GuardInterface.php
│   ├── ConfiguredGuard.php
│   ├── GuardRegistry.php
│   ├── GuardResolver.php
│   └── GuardStatus.php
│
├── Context/
│   ├── AuthenticationContext.php
│   ├── AuthenticationContextKey.php
│   ├── AuthenticationContextRegistry.php
│   ├── AuthenticationContextResolver.php
│   ├── AuthenticationContextValidator.php
│   ├── AuthenticationContextResolution.php
│   └── AuthenticationScopeAccessor.php
│
└── Compilation/
    ├── AuthenticationFirewallCompiler.php
    ├── CompiledFirewallRegistry.php
    └── CompiledFirewallMatcher.php
```

---

## 229. Public API objetivo

VoltStack deberá poder ofrecer:

```php
Auth::check();

Auth::guest();

Auth::identity();

Auth::user();

Auth::id();

Auth::context();

Auth::assurance();
```

Y:

```php
Auth::guard('admin')->check();

Auth::guard('admin')->identity();

Auth::guard('api')->context();
```

sin exponer al desarrollador toda la complejidad interna.

---

## 230. Arquitectura interna objetivo

La simplicidad pública deberá traducirse internamente a:

```text
Auth
 ↓
Guard / Primary Context
 ↓
Context Resolver
 ↓
Firewall
 ↓
Recovery / Authentication Manager
 ↓
AuthenticationContext
```

---

## 231. Anti-patterns prohibidos

### 231.1 Guard contains current user

Evitar:

```php
class Guard
{
    private ?User $user;
}
```

en instancias compartidas entre requests.

---

### 231.2 Firewall verifies credentials

Evitar:

```text
Firewall
    verify password
```

---

### 231.3 Silent guard fallback

Evitar:

```text
unknown guard
    ↓
default guard
```

---

### 231.4 Silent firewall downgrade

Evitar:

```text
admin fails
    ↓
web succeeds
```

como fallback implícito.

---

### 231.5 Context based only on session ID

Evitar:

```text
session exists
    ↓
authenticated
```

sin validación del Authentication State.

---

### 231.6 Tenant-unaware context reuse

Evitar:

```text
context tenant A
    reused in tenant B
```

---

### 231.7 Global current guard

Evitar:

```php
Auth::$currentGuard = 'admin';
```

en runtimes persistentes.

---

## 232. Criterios de aceptación

El sistema será considerado correcto cuando:

1. soporte múltiples firewalls;
2. soporte múltiples guards;
3. Firewall y Guard tengan responsabilidades distintas;
4. los matchers sean composables;
5. la resolución sea determinista;
6. detecte ambigüedades;
7. permita default firewall;
8. soporte rutas públicas;
9. soporte stateful;
10. soporte stateless;
11. soporte hybrid;
12. soporte providers por firewall;
13. soporte authenticators por firewall;
14. soporte policies por firewall;
15. soporte assurance mínimo;
16. soporte tenant-aware resolution;
17. prevenga cross-tenant context reuse;
18. permita múltiples contextos en casos avanzados;
19. tenga un contexto primario;
20. permita lazy recovery;
21. permita eager recovery;
22. memoice resolución por request;
23. sea seguro con FrankenPHP;
24. permita compilación;
25. permita debugging seguro;
26. preserve ergonomía estilo Laravel;
27. preserve potencia conceptual estilo Symfony;
28. no convierta Guards en el núcleo de Authentication;
29. no permita downgrade implícito;
30. mantenga separación estricta con Authorization.

---

## 233. Regla arquitectónica final

La arquitectura deberá preservar la siguiente separación:

```text
WHERE does authentication apply?
        ↓
FIREWALL

HOW should application code access it?
        ↓
GUARD / AUTH FACADE

WHO is currently authenticated and under what guarantees?
        ↓
AUTHENTICATION CONTEXT
```

En consecuencia:

> **El Firewall selecciona la política y configuración de autenticación.**
> **El Guard ofrece una interfaz de acceso conveniente a esa configuración.**
> **El AuthenticationContext representa la identidad autenticada real y sus garantías.**

Ninguno de los tres conceptos deberá sustituir a los demás.

---

## 234. Próximo documento

El siguiente documento recomendado será:

```text
06_AUTHENTICATOR_SYSTEM.md
```

Su responsabilidad será definir completamente el modelo de Authenticators de VoltStack, incluyendo:

```text
AuthenticatorInterface
authenticator capabilities
supports()
request credential extraction
passport creation
interactive vs non-interactive authenticators
stateful vs stateless authenticators
password authenticator
session authenticator
bearer authenticator
API key authenticator
passkey authenticator
magic-link authenticator
OIDC/SAML adapters
service authenticators
authenticator metadata
authenticator registries
priority
conflict resolution
protocol state
error semantics
security boundaries
extension APIs
compilation
performance
testing
```

Ese documento establecerá cómo VoltStack podrá incorporar nuevos mecanismos de Authentication sin modificar el núcleo del framework.
