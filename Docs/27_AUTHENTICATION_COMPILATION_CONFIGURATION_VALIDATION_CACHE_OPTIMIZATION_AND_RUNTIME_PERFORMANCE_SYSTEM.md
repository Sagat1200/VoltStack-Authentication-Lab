# VoltStack Authentication System

## 27 — Authentication Compilation, Configuration Validation, Cache Optimization and Runtime Performance System

- **Archivo:** `27_AUTHENTICATION_COMPILATION_CONFIGURATION_VALIDATION_CACHE_OPTIMIZATION_AND_RUNTIME_PERFORMANCE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema encargado de validar, compilar, optimizar y preparar la configuración de Authentication para ejecución eficiente y segura.

---

## 1. Propósito

Este documento define cómo VoltStack transformará la configuración declarativa del sistema Authentication en una representación:

- validated
- normalized
- compiled
- immutable
- optimized
- cacheable
- runtime-safe

capaz de utilizarse eficientemente durante cada request.
El objetivo fundamental será evitar que cada Authentication request tenga que resolver repetidamente:

- firewalls
- guards
- providers
- authenticators
- policies
- risk rules
- MFA requirements
- device rules
- events
- hooks
- rate limits
- entry points
- failure mappings
- desde configuración cruda.

## 2. Principio fundamental

VoltStack separará dos mundos:

```text
BOOTSTRAP / COMPILE TIME

REQUEST / RUNTIME
```

La mayor cantidad posible de trabajo estático deberá ocurrir durante:

- bootstrap
- container compilation
- cache warmup
- deployment build

y no durante cada Authentication attempt.

## 3. Regla central

Todo aquello que pueda determinarse antes de recibir el request deberá resolverse, validarse o compilarse antes del request.

## 1. Ejemplo

No queremos ejecutar cada vez:

```php
$config = config('authentication');

foreach ($config['firewalls'] as $firewall) {
    foreach ($firewall['authenticators'] as $authenticator) {
        // resolve services
        // validate priorities
        // validate providers
        // compile policies
    }
}
```

El runtime deberá recibir algo semejante a:

- CompiledAuthenticationRuntime
- ya preparado.

## 5. Arquitectura general

Authentication Configuration
│
▼
Configuration Loader
│
▼
Schema Validation
│
▼
Semantic Validation
│
▼
Dependency Resolution
│
▼
Authentication Compiler
│
┌──────┼──────────────────────────────┐
▼      ▼         ▼         ▼          ▼
Firewall Auth   Provider   Policy    Event/Hook
Compiler Registry Registry Compiler  Compiler
│      │         │         │          │
└──────┴─────────┴─────────┴──────────┘
│
▼
CompiledAuthenticationRuntime
│
▼
Runtime Cache
│
▼
Authentication Requests

## 6. Configuration Layers

La configuración efectiva podrá construirse desde:

- framework defaults
- application configuration
- environment overrides
- firewall configuration
- tenant policy
- package extensions
- runtime-safe dynamic policy

## 7. Distinción crítica

No toda configuración será compilable de la misma manera.

- VoltStack distinguirá:
- STATIC CONFIGURATION
- DYNAMIC CONFIGURATION
- REQUEST CONTEXT

## 8. Static Configuration

Ejemplos:

- registered authenticators
- firewall names
- service IDs
- provider types
- event listeners
- hook definitions
- policy structures

route entry point types
rate-limit algorithms
Puede compilarse.

## 9. Dynamic Configuration

Ejemplos:

- tenant-specific security policy
- feature flags
- temporary attack mode
- security emergency policy

Puede requerir resolución runtime.

## 10. Request Context

Nunca se compila globalmente:

- current Identity
- current Tenant
- current Session
- current Device
- current Risk signals
- current credentials

## 11. AuthenticationConfiguration

Objeto conceptual:

```php
final readonly class AuthenticationConfiguration
{
    public function __construct(
        public FirewallConfigurationSet $firewalls,
        public AuthenticatorConfigurationSet $authenticators,
        public ProviderConfigurationSet $providers,
        public AuthenticationPolicyConfiguration $policies,
        public AuthenticationExtensionConfiguration $extensions,
    ) {}
}
```

## 12. Configuration Loader

Contrato:

```php
interface AuthenticationConfigurationLoaderInterface
{
    public function load(): AuthenticationConfiguration;
}
```

## 13. Configuration Sources

Podrán existir:

- PHP arrays
- configuration files
- environment variables
- compiled configuration
- extension registrations

## 14. No runtime arbitrary configuration mutation

Evitar:

```php
config([
    'auth.firewall' => $currentTenant,
]);
en workers persistentes.
```

## 15. Configuration Normalization

Antes de validar deberá normalizarse.
Ejemplo:
'limit' => '10/minute'
puede convertirse en:

```php
RateLimit(
    capacity = 10,
    interval = 60 seconds
)
```

## 16. AuthenticationConfigurationNormalizer

interface AuthenticationConfigurationNormalizerInterface
{
public function normalize(
RawAuthenticationConfiguration $configuration
): AuthenticationConfiguration;
}

## 17. Schema Validation

La primera fase comprobará estructura.
Ejemplo:

- authentication.firewalls must be map
- authenticator.priority must be integer
- provider must exist

rate limit must be valid duration

## 18. Schema vs Semantic Validation

Diferencia importante.
Schema:
priority is integer
Semantic:
two authenticators cannot have conflicting exclusive priority

## 19. Semantic Validation

Validará relaciones entre componentes.

## 20. Ejemplos

Firewall references unknown Authenticator
Authenticator references missing Provider
OIDC Authenticator has no issuer
Passkey RP ID missing
Remember-Me enabled without persistent credential store
Risk policy references unknown signal
Event listener references unknown service

## 21. AuthenticationConfigurationValidator

interface AuthenticationConfigurationValidatorInterface
{
public function validate(
AuthenticationConfiguration $configuration
): AuthenticationConfigurationValidationResult;
}

## 22. ValidationResult

Podrá contener:

- errors
- warnings
- deprecations
- optimization hints

## 23. Errors

Impiden bootstrap.

## 24. Warnings

Ejemplos:

- Remember-Me lifetime > recommended threshold
- Password login enabled without MFA on admin firewall

Rate limit store is memory under multi-node deployment

## 25. Warnings no deben ser silenciosas

Podrán aparecer en:

- CLI
- development bootstrap
- deployment diagnostics

## 26. Production strict mode

VoltStack podrá soportar:

- authentication.strict_configuration = true
- para convertir ciertos warnings en errores.

## 27. Security Semantic Validation

Esta fase será especialmente importante.

## 28. Ejemplo — Password

Si:
password authenticator enabled
deberá existir:

- password hasher policy
- identity provider
- failure policy

## 29. Ejemplo — MFA

Si:
admin requires AAL2
pero ningún factor puede satisfacer AAL2:
CONFIGURATION ERROR

## 30. Ejemplo — Passkey

Si:
passkey authenticator
pero:

- RP ID invalid
- origin list empty
- challenge store unavailable
- debe fallar configuración.

## 31. Ejemplo — OIDC

Debe validar:

- issuer
- client ID
- redirect URI
- allowed algorithms
- JWKS strategy
- state protection

## 32. Ejemplo — Sessions

Si firewall es stateful:
SessionStore required

## 33. Ejemplo — Stateless

Si firewall es stateless:

- session authenticator restoration disabled
- por default.

## 34. Ejemplo — Remember-Me

No deberá activarse en:

- strict stateless API firewall
- sin diseño explícito.

## 35. Ejemplo — Multi-Tenant

Si un provider requiere Tenant:
TenantResolver must be available before Identity resolution

## 36. Dependency Graph

Authentication tiene muchas dependencias.
VoltStack deberá construir:
AuthenticationDependencyGraph

## 37. Ejemplo

Firewall
├── Authenticator
│    └── IdentityProvider
│         └── Database
├── RiskEngine
├── RateLimiter
├── SessionManager
└── EventDispatcher

## 38. Dependency Cycle Detection

Debe detectar:

```text
RiskEngine → AuthManager → RiskEngine
u otros ciclos inválidos.
```

## 39. AuthenticationDependencyGraphValidator

interface AuthenticationDependencyGraphValidatorInterface
{
public function validate(
AuthenticationDependencyGraph $graph
): void;
}

## 40. Compiler

Componente central:

```php
interface AuthenticationCompilerInterface
{
    public function compile(
        AuthenticationConfiguration $configuration
    ): CompiledAuthenticationRuntime;
}
```

## 41. CompiledAuthenticationRuntime

Debe ser:

- immutable
- optimized
- safe for sharing
- read-only

## 42. Contenido conceptual

final readonly class CompiledAuthenticationRuntime
{
public function__construct(
public CompiledFirewallMap $firewalls,
public CompiledAuthenticatorRegistry $authenticators,
public CompiledProviderRegistry $providers,
public CompiledPolicyRegistry $policies,
public CompiledEntryPointRegistry $entryPoints,
public CompiledEventRegistry $events,
public CompiledHookRegistry $hooks,
) {}
}

## 43. Runtime no contiene estado actual

Nunca:

- current user
- current request
- current tenant
- current session
- current device
- current risk

## 44. Firewall Compilation

Cada firewall deberá convertirse a:
CompiledFirewall

## 45. CompiledFirewall

Podrá contener:

- matcher
- state mode
- authenticator chain
- provider mapping
- entry point
- policy profile
- session profile
- risk profile
- failure handler

## 46. Firewall Matching

Match rules podrán compilarse.

- Ejemplos:
- path prefix
- host
- request type
- route metadata

## 47. Runtime Match

Idealmente:

```php
O(1)
o recorrido mínimo.
```

## 48. No regex recompilation

Regex configuradas deberán precompilarse cuando sea posible.

## 49. Authenticator Registry Compilation

Se resolverán:

- service IDs
- priorities
- supported transports
- credential types
- firewall availability

## 50. Authenticator Resolution Table

Ejemplo:

```text
PasswordCredential
    → PasswordAuthenticator

BearerCredential
    → BearerTokenAuthenticator

WebAuthnAssertion
    → PasskeyAuthenticator
```

## 51. Priority Sorting

Ordenar una vez durante compile time.

```php
No:
usort(...)
en cada request.
```

## 52. Authenticator Conflict Detection

Detectar:

- two exclusive authenticators
- same credential type
- same priority
- same firewall

si semántica resulta ambigua.

## 53. Provider Registry Compilation

Resolver:

- provider ID
- identity type
- tenant requirements
- cache policy
- capabilities

## 54. Provider Capability Map

Ejemplo:

```text
database_users:
    PASSWORD_LOOKUP
    IDENTITY_BY_ID
    TENANT_SCOPED
```

oidc_mapping:
FEDERATED_IDENTITY

## 55. Provider Selection

Runtime deberá evitar búsqueda lineal innecesaria.

## 56. Policy Compilation

Authentication Policies podrán contener reglas estáticas.

## 57. Policy AST

Una policy declarativa podría compilarse a:
AuthenticationPolicyAst

## 58. Ejemplo declarativo

'admin' => [
'required_assurance' => 'aal2',
'risk' => [
'high' => 'require_passkey',
],
]

## 59. Compiled form

CompiledPolicy
├── MinimumAssurance(AAL2)
└── RiskRule(HIGH → PhishingResistant)

## 60. Runtime policy evaluation

Solo evalúa datos dinámicos.

## 61. No parsing strings runtime

Evitar:

```php
Duration::parse('30 minutes');
en cada request.
```

## 62. MFA Policy Compilation

Podrá precomputar:

- required factors
- acceptable factor combinations
- fallback constraints
- freshness rules

## 63. Step-Up Requirement Graph

Ejemplo:

```text
AAL1
  ↓
```

needs one possession factor
↓
TOTP OR Passkey

## 64. Passkey Requirement Optimization

Si policy exige:

- phishing_resistant
- resolver lista de factores compatibles durante compilation.

## 65. Risk Policy Compilation

Precompilar:

- thresholds
- signal weights
- correlation rules
- hard denies
- decay policies

## 66. Risk Runtime

Solo recibe:

- SecuritySignalSet
- RiskContext

## 67. Abuse Protection Compilation

Compilar:

- rate limit profiles
- dimensions
- algorithms
- stores
- actions

## 68. RateLimitDefinition

Strings como:

- 5/min
- deberán normalizarse.

## 69. Composite Keys

La estructura deberá predefinirse.

- Ejemplo:
- IP
- Identity
- IP + Identity
- Tenant + IP

## 70. Device Policy Compilation

Precomputar:

- trust requirements
- lifetimes
- binding rules
- management rules

## 71. Flow Policy Compilation

Precomputar:

- flow TTL
- available methods
- entry point mapping
- challenge behavior
- continuation constraints

## 72. Failure Mapping Compilation

Documento 25 definió:

```text
internal code
→ public code
→ transport response
```

Este mapping deberá precompilarse.

## 73. Example

INVALID_PASSWORD
UNKNOWN_IDENTITY
podrán compartir:

- AUTHENTICATION_FAILED
- sin resolver reglas dinámicamente.

## 74. Event Registry Compilation

Documento 23.

```text
Precomputar:
EventClass → ordered listeners
```

## 75. Hook Registry Compilation

HookPoint → ordered handlers

## 76. Listener Metadata

Precompilar:

- priority
- dispatch mode
- failure policy
- projection

## 77. No reflection en hot path

Attributes deberán escanearse durante:

- bootstrap
- cache warmup
- build

## 78. Attribute Metadata Cache

Podrá existir:
AuthenticationAttributeMetadataCache

## 79. Route Metadata Compilation

Authentication requirements declarados en routes podrán compilarse.
Ejemplo:
->requiresAuth('aal2')

## 80. Compiled Route Requirement

Route ID
→ AuthenticationRequirementSet

## 81. Runtime lookup

O(1) por route ID.

## 82. Entry Point Compilation

Resolver por:

- Firewall
- Transport
- Purpose

## 83. Example table

web + HTML
→ FormLoginEntryPoint

web + SPA
→ SpaLoginEntryPoint

api + JSON
→ ApiAuthenticationEntryPoint

## 84. Authentication Runtime Resolver

Será el bridge entre compiled state y request state.

## 85. Conceptual

interface AuthenticationRuntimeResolverInterface
{
public function resolve(
AuthenticationRequestContext $request
): RuntimeAuthenticationContext;
}

## 86. Request Runtime Context

Puede contener:

- CompiledFirewall
- Tenant
- Request metadata
- Transport
- Security Realm

## 87. Fast Path

VoltStack deberá favorecer fast paths para casos comunes.

## 88. Example — Active Session

Request
↓
Compiled Firewall lookup
↓
Session cookie exists
↓
Session restore
↓
Identity/security version valid
↓
Authenticated
sin ejecutar:

- PasswordAuthenticator
- Federation
- MFA
- Risk full evaluation

si no son necesarios.

## 89. Bearer Fast Path

Authorization header
↓
Bearer Firewall
↓
Token parser
↓
Token verifier
sin cargar Session subsystem.

## 90. Anonymous Fast Path

Una route pública puede marcar:

- authentication optional
- y evitar trabajo innecesario.

## 91. Lazy Subsystems

Risk Engine no deberá inicializar providers externos si no se requiere.

## 92. Lazy Resolution

Sin embargo, dependency map estará validado previamente.

## 93. Authentication Pipeline Compilation

En lugar de construir middleware/pipeline cada request:
CompiledAuthenticationPipeline

## 94. Example

Firewall:
admin

Pipeline:

- 1 Network Context
- 2 Abuse Protection
- 3 Authentication
- 4 Risk
- 5 Assurance
- 6 Session

## 95. Conditional Stages

Podrán precompilar condiciones:

```php
if no session → authenticator chain
if successful → risk
if risk high → step-up
```

## 96. Avoid dynamic service discovery

No:

```php
$container->get($nameFromRequest);
sin registry validado.
```

## 97. Service References

Compiled runtime puede contener:

- service IDs
- indexed references

resolved immutable service handles
según container design.

## 98. Container Integration

VoltStack Quantum\Container deberá colaborar con Authentication compiler.

## 99. Compiler Passes

Puede usar fases:

- Load
- Normalize
- Validate
- Resolve
- Compile
- Optimize
- Freeze
- Cache

## 100. AuthenticationCompilerPass

Contrato:

```php
interface AuthenticationCompilerPassInterface
{
    public function process(
        AuthenticationCompilationContext $context
    ): void;
}
```

## 101. Passes built-in

Podrán existir:

- FirewallCompilerPass
- AuthenticatorCompilerPass
- ProviderCompilerPass
- PolicyCompilerPass
- RiskCompilerPass
- MfaCompilerPass
- EventCompilerPass
- HookCompilerPass
- FailureMappingCompilerPass

## 102. Compiler Pass Ordering

Determinista.

## 103. Extension Compiler Passes

Plugins podrán registrar passes en puntos controlados.

## 104. Security concern

Un compiler pass de plugin es código confiable y puede alterar arquitectura.
Debe registrarse durante bootstrap, no runtime.

## 105. Compilation Context

Podrá contener:

- normalized config
- service graph
- registries
- diagnostics
- compiled metadata

## 106. Freeze Phase

Después de compilation:
registries immutable

## 107. Runtime Registration

Por default no permitir:

```php
Auth::registerAuthenticator(...)
después de freeze en producción.
```

## 108. Development Mode

Podrá permitir hot reload.

## 109. Production Mode

Preferir:
immutable compiled runtime

## 110. Authentication Cache

Debemos distinguir varios tipos.

- CONFIGURATION CACHE
- METADATA CACHE
- POLICY CACHE
- IDENTITY CACHE
- RUNTIME DECISION CACHE
- No son equivalentes.

## 111. Configuration Cache

Seguro para:
compiled static config

## 112. Metadata Cache

Ejemplo:

- attributes
- route requirements
- authenticator metadata
- provider capabilities

## 113. Policy Cache

Puede contener:
compiled policies

## 114. Identity Cache

Mucho más sensible.

- Debe considerar:
- security state
- version
- tenant
- revocation

## 115. Decision Cache

Especialmente riesgoso.
No cachear indiscriminadamente:
Authentication allowed = true

## 116. Regla crítica

Cachear configuración y metadata es generalmente seguro; cachear decisiones de seguridad requiere invalidación explícita y contexto completo.

## 1. CompiledAuthenticationCache

Contrato:

```php
interface CompiledAuthenticationCacheInterface
{
    public function load(
        AuthenticationCacheKey $key
    ): ?CompiledAuthenticationRuntime;

    public function store(
        AuthenticationCacheKey $key,
        CompiledAuthenticationRuntime $runtime
    ): void;
}
```

## 118. Cache Key

Debe depender de:

- framework version
- authentication schema version
- application config hash
- extension set
- environment

## 119. Example

auth:v1:
framework-2.0:

- config-a79f:
- extensions-b42e

## 120. Cache Invalidation

Debe invalidarse cuando cambien:

- Authentication configuration
- Service registrations
- Authenticator package version
- Policy files
- Extension registrations
- Route Authentication metadata

## 121. File mtimes

Podrán utilizarse en development.

## 122. Production build hash

Preferible hash determinista.

## 123. Configuration Fingerprint

AuthenticationConfigurationFingerprint

## 124. Runtime Cache Integrity

Compiled cache deberá poder detectar:

- corrupt cache
- wrong version
- incompatible schema

## 125. No unserialize inseguro

Si se serializa compiled metadata, deberá utilizar formato seguro/controlado.

## 126. Generated PHP Cache

Una opción eficiente:
generated PHP arrays/classes

## 127. Benefits

OPcache compatible
fast load
no generic serialization attack surface

## 128. Authentication Cache Warmup

Durante deployment:

- php volt auth:cache
- conceptualmente.

## 129. Warmup tasks

compile configuration
validate policies
scan attributes
build registries
compile routes
validate dependency graph
write cache

## 130. Clear command

php volt auth:clear
conceptualmente.

## 131. Verify command

php volt auth:validate

## 132. Inspect command

php volt auth:inspect
podría mostrar:

- firewalls
- authenticators
- providers
- entry points
- policies
- listeners
- hooks
- sin secrets.

## 133. Explain compiled config

Tooling podrá responder:

- Why is Passkey required on admin?
- mostrando policy resolution estática.

## 134. Secret Safety in Config Dump

Nunca mostrar:

- OIDC client secret
- private signing key
- token encryption key

## 135. Secret References

Configuración compilada deberá contener:

- SecretReference
- no secret material cuando sea posible.

## 136. Runtime Secret Resolver

Recupera secret solo cuando se necesita.

## 137. Secret caching

Si se permite:

- memory-protected
- bounded
- immutable
- y nunca debug-dumped.

## 138. Environment Variables

No deben convertirse automáticamente en generated cache visible si son secretos.

## 139. Secrets Manager Integration

Puede soportar adapters para:

- Vault
- AWS Secrets Manager
- cloud secret providers
- environment
- encrypted local store
- en otros subsistemas.

## 140. Dynamic Tenant Policies

No toda policy tenant puede compilarse globalmente.

## 141. Strategy

Compilar:
Policy Template
y runtime aplica:
Tenant Policy Overlay

## 142. Tenant Policy Resolver

Contrato:

```php
interface TenantAuthenticationPolicyResolverInterface
{
    public function resolve(
        TenantReference $tenant,
        CompiledAuthenticationPolicy $base
    ): EffectiveAuthenticationPolicy;
}
```

## 143. Tenant Policy Cache

Puede existir con:

- TenantPolicyVersion
- TTL
- explicit invalidation

## 144. Security floor

Tenant overlay nunca podrá reducir:
FrameworkSecurityFloor

## 145. Policy Merge Compilation

Precompilar merge semantics:

```text
minimum assurance → max
limits → more restrictive
```

required factors → union/constraint composition
denials → additive

## 146. No arbitrary array merge

No usar:

```php
array_merge(...)
para security policy.
```

## 147. Typed Policy Composition

Cada policy type define su operator.

## 148. Example

Assurance:

```text
AAL1 + AAL2
    → AAL2
```

## 149. Rate Limit

10/min + 5/min
→ 5/min
si representan same dimension/security scope.

## 150. Factor Requirements

MFA
+
phishing-resistant
no significa que uno sustituya al otro.
Debe componerse semánticamente.

## 151. Optimization Pass

Después de compilation podrán aplicarse optimizaciones.

## 152. Dead Authenticator Elimination

Si ningún firewall referencia un Authenticator:

- do not include in runtime registry
- salvo dynamic use explícito.

## 153. Dead Listener Elimination

Internal listeners desactivados por feature config pueden omitirse.

## 154. Constant Folding

Ejemplo:

- Firewall admin always requires AAL2
- puede precomputarse.

## 155. Policy Rule Indexing

En lugar de evaluar todas las reglas:

- index by:
- risk level
- purpose
- firewall
- authenticator

## 156. Risk Rule Index

Ejemplo:

```php
HIGH
→ [rule 7, rule 9]

LOW
→ [rule 1]
```

## 157. Event Listener Lookup

Hash map:

```text
EventClass
→ listeners
```

## 158. Hook Lookup

HookPoint
→ handlers

## 159. Provider Lookup

ProviderId
→ service reference

## 160. Authenticator Lookup

CredentialType
→ candidate chain

## 161. Entry Point Lookup

Firewall + Transport
→ EntryPoint

## 162. Runtime Allocation Reduction

Authentication hot paths deberán minimizar objetos temporales innecesarios.

## 163. But not at cost of safety

No reutilizar objetos request-scoped mutables.

## 164. Object Pools

No recomendados para:

- AuthenticationContext
- Identity
- Credentials
- RiskContext

debido a riesgo de leakage.

## 165. Immutable Value Objects

Favorecer.

## 166. Shared immutable metadata

Sí puede reutilizarse entre requests.

## 167. Request-local mutable state

Debe permanecer aislado.

## 168. FrankenPHP Runtime Model

VoltStack deberá asumir:
worker lives for many requests

## 169. Shared State Allowed

Ejemplos:

- compiled firewall metadata
- immutable policies
- event registry
- service definitions
- static routing tables

## 170. Shared State Forbidden

current Identity
current Session
current Tenant
current Flow
current Risk
current Authentication Result
current Device

## 171. Request Reset

Al finalizar request:

- AuthenticationExecutionContext cleared
- Fiber local state cleared

Request event buffers cleared
temporary security context cleared

## 172. AuthenticationRuntimeResetter

Contrato:

```php
interface AuthenticationRuntimeResetterInterface
{
    public function reset(): void;
}
```

## 173. Reset integration

Debe integrarse con FrankenPHP worker lifecycle.

## 174. Reset Exceptions

Aunque request falle con exception:

```php
finally {
    reset();
}
conceptualmente.
```

## 175. Fiber-local Authentication Context

Necesario si runtime concurrente.

## 176. No static global Auth::user

La Facade podrá aparentar:
Auth::user();
pero internamente resolverá:
ExecutionContext

## 177. Facade Cache

No cachear current user en propiedad static.

## 178. Performance Budgets

VoltStack podrá definir budgets por etapa.

- Ejemplo conceptual:
- Firewall resolution
- Authenticator selection
- Identity lookup
- Password verification
- Risk evaluation
- Session creation

## 179. AuthenticationPerformanceBudget

final readonly class AuthenticationPerformanceBudget
{
public function __construct(
public Duration $total,
public Duration $risk,
public Duration $externalProviders,
) {}
}

## 180. Deadline Propagation

Si total:
2 seconds
y password tomó:

- 400ms
- providers externos reciben budget restante.

## 181. Authentication Execution Deadline

Podrá propagarse a:

- OIDC
- Risk
- Remote Identity Provider
- Device Attestation

## 182. No indefinite dependency waits

Obligatorio.

## 183. Performance ≠ weakening security

No omitir:

- signature verification
- revocation checks
- security version checks
- por rapidez.

## 184. Fast Security Paths

En cambio:

- precomputed keys
- indexed lookups
- local metadata
- efficient crypto libraries

## 185. Password Hashing

Será deliberadamente caro.
No optimizar reduciendo parámetros por load runtime de manera insegura.

## 186. Resource Governor Integration

Documento 19.

- Puede limitar concurrencia de:
- Argon2
- external providers
- challenge creation

## 187. Password Verification Pool

No object pool de secrets.
Puede haber:
concurrency semaphore

## 188. Cache de Identity

Si se utiliza, debe ser cuidadoso.

## 189. Identity Cache Key

Debe incluir:

- Tenant
- IdentityProvider
- Canonical Identity key

## 190. Identity Cache Value

Puede incluir:

- identity summary
- security version
- status

## 191. Identity Cache TTL

Debe ser corto/configurable para security-sensitive state.

## 192. Immediate invalidation

Al:

- disable account
- security hold
- password reset
- credential compromise
- debe invalidarse.

## 193. Versioned Cache

Una estrategia mejor:
IdentitySecurityVersion

## 194. Session Restoration Optimization

Puede comparar security version eficiente.

## 195. Revocation Cache

Puede existir:
revocation version

## 196. Stale Security Cache

Debe tratarse como riesgo serio.

## 197. Cache Failure Policy

Si cache falla:

- fallback to authoritative store
- normalmente.

## 198. Cache is not source of truth

Salvo componentes donde store explícitamente sea cache/distributed authoritative system.

## 199. Authentication Metadata Cache

Sí puede ser authoritative para compiled static metadata.

## 200. Cache Stampede

Al expirar compiled/dynamic policy caches, múltiples requests podrían recomputar.

## 201. Stampede Protection

Podrá usar:

- single-flight
- locks
- stale-while-revalidate

solo cuando seguridad lo permita.

## 202. Never stale critical security policy indiscriminately

Un policy change de emergencia debe propagarse rápidamente.

## 203. Policy Version

Cada policy dinámica deberá tener:
version

## 204. Emergency Invalidation

Security operations podrán:

- invalidate auth policy cache
- global/tenant.

## 205. Attack Mode

Documento 19.
Debe poder actualizarse sin recompilar toda aplicación.

## 206. Static vs Emergency Policy

Separar:

- Compiled Base Policy
- +;
- Runtime Emergency Overlay

## 207. Emergency Overlay

Puede:

- lower rate limits
- require stronger factors
- deny vulnerable authenticator

## 208. Security monotonicity

Emergency overlay podrá endurecer.
No debilitar framework floor.

## 209. Observability Performance

Documento 24.

- No bloquear request innecesariamente por:
- metrics
- non-critical tracing
- analytics

## 210. Audit Criticality

Audit crítico puede ser síncrono según policy.

## 211. Event Dispatch Performance

Listener registry precompilado.

## 212. Deferred Listeners

Sacarlos del hot path cuando sea posible.

## 213. Trace Sampling

Aplicar antes de crear spans costosos cuando sea posible.

## 214. Logging Sampling

Evitar flood de invalid credentials.

## 215. Authentication Profiler

Development tooling podrá medir etapas.

## 216. Example

Authentication Profile

Firewall resolution       0.04 ms
Session restoration       0.23 ms
Identity lookup           1.4 ms
Password verification   165.0 ms
Risk assessment           8.7 ms
Session creation          0.8 ms
Total                   176.2 ms

## 217. AuthenticationProfiler

Contrato:

```php
interface AuthenticationProfilerInterface
{
    public function begin(string $stage): AuthenticationProfileSpan;
}
```

## 218. Production overhead

Profiler detallado podrá desactivarse/sampling.

## 219. Performance Regression Testing

Documento 26 deberá poder verificar budgets.

## 220. Example

Compiled firewall lookup
must not regress from O(1)

## 221. Query Count Regression

Authentication pipelines pueden tener expected query budgets.

## 222. Memory Regression

FrankenPHP long-run tests deberán detectar retained objects.

## 223. Cold Start

Especialmente importante para:

- CLI
- serverless-like environments
- development

## 224. Warm Runtime

Especialmente FrankenPHP.

## 225. Dual optimization

VoltStack deberá equilibrar:

- fast bootstrap
- fast request runtime

## 226. Preloading

Generated Authentication classes/metadata podrán beneficiarse de:

- PHP OPcache
- preloading

cuando deployment lo soporte.

## 227. No mandatory preload

Será opcional.

## 228. Cache Format Version

Debe existir:
AuthenticationCompiledCacheVersion

## 229. Framework upgrade

Si version incompatible:
cache ignored/rebuilt

## 230. No mysterious stale compiled config

Bootstrap deberá verificar fingerprint.

## 231. Configuration Diagnostics

Errores deberán indicar:

- path
- component
- reason
- suggested correction

## 232. Example

authentication.firewalls.admin.authenticators.passkey

Error:
PasskeyAuthenticator requires at least one allowed origin.

## 233. Secret-safe diagnostics

No mostrar secret values.

## 234. Cross-Configuration Validation

Ejemplo:

- Admin route requires phishing-resistant auth
- but admin firewall has no compatible authenticator.

Debe detectarse antes de runtime cuando sea estático.

## 235. Route-to-Firewall Validation

Podrá detectar rutas protegidas incompatibles.

## 236. Authentication Coverage Diagnostics

Tooling podrá listar:

- Firewall
- Routes
- Entry Point
- Authenticators
- Required Assurance

## 237. Ambiguous Firewall Detection

Dos firewalls match mismo request con misma prioridad.

- Debe ser:
- error
- o resolución explícita.

## 238. Firewall Priority

Precompilada.

## 239. Default Firewall

Debe ser explícito cuando necesario.

## 240. Shadowed Firewall Warning

Ejemplo:
/admin/**
nunca se alcanza porque:

- /**
- tiene mayor prioridad.

## 241. Authenticator Shadowing

Un Authenticator puede nunca seleccionarse.
Warning.

## 242. Policy Shadowing

Una regla más general puede hacer otra inalcanzable.
Tooling puede detectarlo parcialmente.

## 243. Dead Configuration Analysis

Advanced, pero útil.

## 244. Configuration Schema Versioning

Authentication config tendrá:
schema version

## 245. Migrations

VoltStack podrá ofrecer:

- deprecation
- automatic migration hints

## 246. Backward Compatibility

Old keys podrán mapearse durante transición.

## 247. Deprecation Warnings

Durante compile.
No cada request.

## 248. Strict V2 mode futuro

Puede remover aliases viejos.

## 249. Package Configuration Extensions

Packages podrán registrar schema fragments.

## 250. Schema Collision

Debe detectarse.

## 251. Extension Configuration Namespace

Ejemplo:
authentication.extensions.vendor_package

## 252. Extension Compiler

Podrá convertir config plugin a metadata.

## 253. Plugin Version Compatibility

Verificar:

- Authentication Extension API version
- antes de compile.

## 254. Compilation Failure

Debe detener bootstrap si afecta seguridad.

## 255. No partial compiled runtime

Nunca ejecutar con:
half compiled authentication configuration

## 256. Atomic Cache Build

Generar archivo temporal:

- auth.cache.tmp
- validar y luego rename atómico.

## 257. Existing valid cache

No sobrescribir con cache corrupto.

## 258. Distributed Deployments

Cada node deberá usar:

- same config fingerprint
- idealmente.

## 259. Configuration Drift Detection

VoltStack podrá exponer:

- AuthenticationConfigurationFingerprint
- en health diagnostics.

## 260. Node mismatch

Security operations pueden detectar:
Node A auth config hash != Node B

## 261. Rolling Deployments

Durante transición pueden coexistir versiones.

## 262. Compatibility Window

Session/flow/event schemas deberán tolerar versiones soportadas.

## 263. Flow State Versioning

Importante para multi-step Authentication durante deployments.

## 264. AuthenticationFlowStateVersion

Un flow creado por versión N podría llegar a node N+1.

## 265. Strategy

compatible read
migrate
or safely restart flow
266. Never deserialize incompatible state silently
267. Session State Versioning

Igual.

## 268. Remember-Me Credential Compatibility

Deployment nuevo debe decidir si credenciales antiguas siguen siendo válidas.

## 269. Credential Version

Puede formar parte del formato.

## 270. Cache Security

Compiled configuration cache puede revelar estructura sensible.

## 271. File permissions

Cache local deberá tener permisos apropiados.

## 272. No raw secrets

Reiteración importante.

## 273. Cache poisoning

No aceptar cache generado por request/user input.

## 274. Cache Integrity

Opcionalmente:

- hash/signature
- si deployment necesita.

## 275. Build provenance

Cache metadata podrá incluir:

- framework version
- build ID
- configuration fingerprint
- createdAt

## 276. Runtime Compatibility Check

Antes de usar cache:

- version compatible?
- fingerprint valid?

## 277. No network compile-time dependencies by default

Authentication compile no debería necesitar contactar:

- OIDC provider
- risk provider

salvo explicit validation mode.

## 278. Offline Compilation

Debe ser posible para deployment reproducible.

## 279. Online Validation Mode

Opcional:

- verify OIDC discovery
- verify JWKS connectivity

como deployment health check.

## 280. Distinction

CONFIGURATION VALID
no significa:
EXTERNAL PROVIDER CURRENTLY AVAILABLE

## 281. Runtime Health Checks

Separados.

## 282. Authentication Warmup

Al iniciar worker FrankenPHP:

- load compiled runtime
- resolve immutable service graph
- warm cryptographic metadata

## 283. Avoid per-worker duplicated huge state

Cuando posible, metadata compacta.

## 284. PHP Memory Model

Cada worker tendrá memoria propia, por lo que compiled structures deberán ser razonables.

## 285. OPcache Sharing

Generated PHP metadata puede reducir duplicación a nivel opcode.

## 286. Large Tenant Counts

No compilar políticas completas para millones de tenants.

## 287. Base + Overlay model

Fundamental.

## 288. Tenant Policy LRU

Puede cachear tenant overlays con límite.

## 289. Bounded caches

Todo cache in-process dinámico debe tener:

- max entries
- TTL
- eviction

## 290. No unbounded associative array

Especialmente:
static $tenantPolicies = [];

## 291. FrankenPHP Memory Leak Prevention

Cualquier cache local dinámico deberá ser:

- bounded
- observable
- resettable when needed

## 292. Per-request Identity Memoization

Puede ser útil.
Ejemplo:
Identity loaded once during login flow request

## 293. Request Memoization

Seguro porque request-scoped.

## 294. Cross-request Identity Memoization

Mucho más complejo y debe usar cache/versioning explícito.

## 295. AuthenticationExecutionMemo

Podrá almacenar request-local:

- resolved tenant
- resolved Identity
- effective policy
- risk provider results

## 296. No secret caching

Passwords/OTPs no.

## 297. Credential Verification Memoization

No reutilizar:

- password valid
- arbitrariamente entre requests.

## 298. Within one flow

Puede persistirse evidence representation firmada/authoritative según documentos previos.

## 299. Runtime Performance Pipeline

REQUEST
│
▼
O(1) Firewall Lookup
│
▼
Compiled Entry Point
│
▼
Compiled Authenticator Candidate Set
│
▼
Request-Local Identity/Context
│
▼
Compiled Policy Evaluation
│
▼
Dynamic Security Evaluation
│
▼
Authentication Result

## 300. Performance Target

El overhead propio del framework alrededor de operaciones esenciales deberá ser pequeño respecto a:

- password hashing
- network calls
- database I/O
- WebAuthn crypto

## 301. Authentication Runtime Metrics

Documento 24 podrá exponer:

- auth_compile_cache_hit
- auth_firewall_resolution_duration
- auth_policy_resolution_duration
- auth_authenticator_resolution_duration
- auth_bootstrap_duration

## 302. Cache Metrics

auth_compiled_cache_hit_total
auth_compiled_cache_miss_total
auth_policy_cache_hit_total
auth_tenant_policy_cache_eviction_total

## 303. Diagnostics

Si cache miss inesperado en producción:
warning

## 304. Profiling Mode

Development podrá registrar:

- resolver calls
- compiled lookups
- cache misses
- provider initialization

## 305. No profiling secrets

Obligatorio.

## 306. Testing — Configuration Validation

Debe cubrir:

- valid configuration
- missing provider
- duplicate firewall
- invalid priority
- unknown Authenticator
- invalid policy

## 307. Testing — Semantic Validation

Ejemplo:

- AAL2 required
- but no compatible factor
- debe fallar.

## 308. Testing — Dependency Cycles

Detectar ciclos.

## 309. Testing — Compiler Determinism

Misma configuración:
same compiled representation/fingerprint

## 310. Testing — Cache Invalidation

Cambiar:

- authenticator config
- debe cambiar fingerprint.

## 311. Testing — Secret-Free Cache

Canary secret no debe aparecer en:

- compiled cache
- diagnostic dump
- generated PHP metadata

cuando se utilice secret reference.

## 312. Testing — Production Freeze

Intentar registrar Authenticator runtime:

- must fail
- en modo frozen.

## 313. Testing — Development Hot Reload

Puede recompilar correctamente.

## 314. Testing — Firewall Resolution

Miles/millones de requests simulados.
Debe producir misma selección.

## 315. Testing — Ambiguous Firewall

Debe detectarse.

## 316. Testing — Policy Composition

Framework floor + app + tenant.
Verificar composición restrictiva.

## 317. Testing — Security Monotonicity

Tenant nunca reduce security floor.

## 318. Testing — Event Registry

Orden precompilado correcto.

## 319. Testing — Hook Registry

Priority/capabilities correctas.

## 320. Testing — Route Metadata

Requirement correcto por route.

## 321. Testing — Distributed Version Compatibility

Flow old version → new node.

## 322. Testing — Cache Corruption

Debe ignorar/rebuild.
No ejecutar metadata corrupta.

## 323. Testing — FrankenPHP Worker Reuse

Compiled state se comparte.
Request state no.

## 324. Testing — Memory

Muchos tenant overlays.
Cache bounded.

## 325. Testing — Concurrency

Concurrent cache load/build no produce archivos parciales.

## 326. Testing — Atomic Cache Write

Crash durante build no rompe cache anterior.

## 327. Testing — Performance Regression

Comparar:

- firewall lookup
- policy lookup
- registry resolution
- contra budgets.

## 328. Testing — Fault Injection

Compiled cache unreadable.
Debe:

- recompile
- or fail bootstrap safely

## 329. Testing — Configuration Drift

Nodes distintos deben poder detectarse.

## 330. Testing — Extension Compatibility

Plugin incompatible:
bootstrap failure

## 331. Security invariants — Configuration

AUTH-COMP-CONFIG-01
Invalid Authentication configuration never reaches runtime.
AUTH-COMP-CONFIG-02
Security-sensitive configuration is semantically validated, not only schema validated.
AUTH-COMP-CONFIG-03
Configuration diagnostics never expose secrets.

- AUTH-COMP-CONFIG-04
- Ambiguous security configuration is rejected or explicitly resolved.
- AUTH-COMP-CONFIG-05

Tenant overlays cannot weaken framework security floors.

## 332. Security invariants — Compilation

AUTH-COMP-01
Compiled Authentication runtime is immutable after freeze.

- AUTH-COMP-02
- Request-specific Authentication state is never stored in compiled runtime.
- AUTH-COMP-03

Compiler output is deterministic for equivalent configuration.
AUTH-COMP-04
Incomplete compilation never produces a usable runtime.
AUTH-COMP-05
Dependency cycles are detected before runtime.

## 333. Security invariants — Cache

AUTH-CACHE-01
Authentication compiled cache is versioned.

- AUTH-CACHE-02
- Stale/incompatible compiled cache is rejected.
- AUTH-CACHE-03

Compiled cache does not contain raw authentication secrets by default.
AUTH-CACHE-04
Security-state caches have explicit invalidation/version semantics.
AUTH-CACHE-05
Caches are not silently treated as authoritative where authoritative state is required.

## 334. Security invariants — Performance

AUTH-PERF-01
Performance optimizations cannot skip mandatory security checks.
AUTH-PERF-02
Password hashing parameters are not weakened dynamically for load reduction.
AUTH-PERF-03
Hot-path registries use precomputed resolution where possible.
AUTH-PERF-04
Dynamic local caches are bounded.
AUTH-PERF-05
External dependency calls have deadlines/timeouts.

## 335. Security invariants — Tenant

AUTH-COMP-TENANT-01
Tenant policy caches are tenant-keyed.

- AUTH-COMP-TENANT-02
- Tenant policy data cannot leak across requests.
- AUTH-COMP-TENANT-03

Large tenant counts do not require compiling all tenant policy instances globally.
AUTH-COMP-TENANT-04
Tenant dynamic policy versions participate in cache invalidation.

## 336. Security invariants — Runtime

AUTH-COMP-RT-01
Shared runtime metadata is immutable.

- AUTH-COMP-RT-02
- Current Identity, Tenant, Session, Device and Risk are execution-scoped.
- AUTH-COMP-RT-03

FrankenPHP request reset clears all Authentication execution state.
AUTH-COMP-RT-04
Fiber-local Authentication contexts do not share mutable state.
AUTH-COMP-RT-05
In-process dynamic caches are bounded and observable.

## 337. Anti-pattern — Parse config every request

No.

## 338. Anti-pattern — Reflection scanning every login

No en producción.

## 339. Anti-pattern — Mutable compiled registries

No.

## 340. Anti-pattern — Current User inside AuthManager singleton

Nunca.

## 341. Anti-pattern — Secrets inside generated config cache

No.

## 342. Anti-pattern — array_merge for security policies

No.

## 343. Anti-pattern — Tenant policy cached globally without tenant key

Crítico.

## 344. Anti-pattern — Unbounded static tenant cache

Especialmente peligroso con FrankenPHP.

## 345. Anti-pattern — Risk result cached as allow=true

Sin contexto/versioning, no.

## 346. Anti-pattern — SecurityVersion ignored for cached Identity

No.

## 347. Anti-pattern — Compile by calling external IdP as requirement

No por default.

## 348. Anti-pattern — Silent fallback to raw config after compile failure

Nunca.

## 349. Anti-pattern — Dynamic authenticator registration in production request

No.

## 350. Anti-pattern — Security checks skipped for performance

Nunca.

## 351. Componentes principales

AuthenticationConfiguration
AuthenticationConfigurationLoader
AuthenticationConfigurationNormalizer
AuthenticationConfigurationValidator
AuthenticationConfigurationValidationResult

AuthenticationCompiler
AuthenticationCompilerPass
AuthenticationCompilationContext
CompiledAuthenticationRuntime

AuthenticationDependencyGraph
AuthenticationDependencyGraphValidator

## 352. Compiled Registries

CompiledFirewallMap
CompiledAuthenticatorRegistry
CompiledProviderRegistry
CompiledPolicyRegistry
CompiledEntryPointRegistry
CompiledEventRegistry
CompiledHookRegistry
CompiledFailureMappingRegistry

## 353. Cache Components

CompiledAuthenticationCache
AuthenticationCacheKey
AuthenticationConfigurationFingerprint
AuthenticationCompiledCacheVersion
AuthenticationCacheInvalidator
AuthenticationCacheWarmer

## 354. Runtime Components

AuthenticationRuntimeResolver
RuntimeAuthenticationContext
AuthenticationExecutionContext
AuthenticationExecutionMemo
AuthenticationRuntimeResetter
AuthenticationExecutionDeadline

## 355. Performance Components

AuthenticationPerformanceBudget
AuthenticationProfiler
AuthenticationProfileSpan
AuthenticationOptimizationPass
AuthenticationRuntimeMetrics

## 356. Tenant Components

TenantAuthenticationPolicyResolver
CompiledTenantPolicyTemplate
EffectiveTenantAuthenticationPolicy
TenantPolicyVersion
TenantAuthenticationPolicyCache

## 357. Namespace sugerido

VoltStack\Quantum\Auth\Compilation
VoltStack\Quantum\Auth\Compilation\Contracts
VoltStack\Quantum\Auth\Compilation\Config
VoltStack\Quantum\Auth\Compilation\Compiler
VoltStack\Quantum\Auth\Compilation\Dependency
VoltStack\Quantum\Auth\Compilation\Registry
VoltStack\Quantum\Auth\Compilation\Cache
VoltStack\Quantum\Auth\Compilation\Runtime
VoltStack\Quantum\Auth\Compilation\Performance
VoltStack\Quantum\Auth\Compilation\Tenant

## 358. Estructura sugerida

src/Quantum/Auth/Compilation/
├── Contracts/
│   ├── AuthenticationConfigurationLoaderInterface.php
│   ├── AuthenticationConfigurationNormalizerInterface.php
│   ├── AuthenticationConfigurationValidatorInterface.php
│   ├── AuthenticationCompilerInterface.php
│   ├── AuthenticationCompilerPassInterface.php
│   ├── CompiledAuthenticationCacheInterface.php
│   └── AuthenticationRuntimeResolverInterface.php
│
├── Config/
│   ├── AuthenticationConfiguration.php
│   ├── AuthenticationConfigurationNormalizer.php
│   ├── AuthenticationConfigurationValidator.php
│   ├── AuthenticationConfigurationValidationResult.php
│   └── AuthenticationConfigurationFingerprint.php
│
├── Compiler/
│   ├── AuthenticationCompiler.php
│   ├── AuthenticationCompilationContext.php
│   ├── FirewallCompilerPass.php
│   ├── AuthenticatorCompilerPass.php
│   ├── ProviderCompilerPass.php
│   ├── PolicyCompilerPass.php
│   ├── RiskCompilerPass.php
│   ├── MfaCompilerPass.php
│   ├── EventCompilerPass.php
│   └── HookCompilerPass.php
│
├── Dependency/
│   ├── AuthenticationDependencyGraph.php
│   └── AuthenticationDependencyGraphValidator.php
│
├── Registry/
│   ├── CompiledFirewallMap.php
│   ├── CompiledAuthenticatorRegistry.php
│   ├── CompiledProviderRegistry.php
│   ├── CompiledPolicyRegistry.php
│   ├── CompiledEventRegistry.php
│   └── CompiledHookRegistry.php
│
├── Cache/
│   ├── CompiledAuthenticationCache.php
│   ├── AuthenticationCacheKey.php
│   ├── AuthenticationCompiledCacheVersion.php
│   ├── AuthenticationCacheWarmer.php
│   └── AuthenticationCacheInvalidator.php
│
├── Runtime/
│   ├── CompiledAuthenticationRuntime.php
│   ├── AuthenticationRuntimeResolver.php
│   ├── RuntimeAuthenticationContext.php
│   ├── AuthenticationExecutionContext.php
│   ├── AuthenticationExecutionMemo.php
│   └── AuthenticationRuntimeResetter.php
│
├── Performance/
│   ├── AuthenticationPerformanceBudget.php
│   ├── AuthenticationProfiler.php
│   ├── AuthenticationProfileSpan.php
│   └── AuthenticationOptimizationPass.php
│
└── Tenant/
├── TenantAuthenticationPolicyResolver.php
├── CompiledTenantPolicyTemplate.php
├── EffectiveTenantAuthenticationPolicy.php
├── TenantPolicyVersion.php
└── TenantAuthenticationPolicyCache.php

## 359. Flujo de compilación completo

Raw Authentication Config
│
▼
LOAD
│
▼
NORMALIZE
│
▼
SCHEMA VALIDATION
│
▼
SEMANTIC VALIDATION
│
▼
DEPENDENCY GRAPH
│
▼
COMPILER
│
┌────────┼────────────────────────────┐
▼        ▼        ▼        ▼          ▼
Firewall Auth   Provider   Policy    Event/Hook
│
▼
OPTIMIZATION
│
▼
FREEZE
│
▼
Configuration Fingerprint
│
▼
CACHE GENERATION
│
▼
CompiledAuthenticationRuntime

## 360. Runtime Flow

REQUEST
│
▼
Compiled Runtime
│
▼
Firewall Lookup
│
▼
Runtime Tenant Context
│
▼
Effective Policy
│
▼
Authenticator Candidate
│
▼
Authentication Pipeline
│
▼
Dynamic Identity / Risk / Device
│
▼
Authentication Result
│
▼
Request Context Reset

## 361. FrankenPHP Flow

WORKER BOOT
│
▼
Load CompiledAuthenticationRuntime
│
▼
Immutable shared metadata
│
├──────────── Request A
│                 │
│                 ▼
│         ExecutionContext A
│                 │
│                 ▼
│               RESET
│
├──────────── Request B
│                 │
│                 ▼
│         ExecutionContext B
│                 │
│                 ▼
│               RESET
│
└──────────── ...

## 362. Configuration Example

return [

'authentication' => [

'compile' => [
'enabled' => true,
'strict' => true,
],

'cache' => [
'enabled' => true,
'driver' => 'generated_php',
],

'runtime' => [
'freeze' => true,
],

],

];

## 363. Tenant Cache Example

'tenant_policy_cache' => [

'enabled' => true,

'max_entries' => 1000,

'ttl' => '10 minutes',

'versioned' => true,

];

## 364. Performance Config

'performance' => [

'profiling' => false,

'deadlines' => [
'total' => '2 seconds',
'external_provider' => '500 milliseconds',
],

];
Valores únicamente ilustrativos.

## 365. Developer CLI conceptual

php volt auth:validate
php volt auth:compile
php volt auth:cache
php volt auth:clear
php volt auth:inspect
php volt auth:profile

## 366. auth:validate

Debe mostrar:

- configuration errors
- semantic conflicts
- missing dependencies
- security warnings
- deprecated configuration

## 367. auth:compile

Genera representación compilada.

## 368. auth:inspect

Ejemplo:
Firewall: admin

State:
stateful

Authenticators:

```text
    passkey
    password
```

Required Assurance:
AAL2

Risk Profile:
high_security

Entry Point:
admin_login

Session:
admin_session

Events:
7 listeners

Hooks:
2 security guards

## 369. auth:profile

Podrá ejecutar test request/scenario y mostrar bottlenecks.

## 370. Relación con Laravel

Laravel utiliza optimizaciones importantes como:

- config cache
- route cache
- container/service provider bootstrap
- cached events
- OPcache

y mantiene una experiencia extremadamente simple para el developer.
VoltStack deberá conservar esta filosofía:

- configure once
- cache once
- execute fast

## 371. Relación con Symfony

Symfony aporta un modelo especialmente interesante mediante:

- Container Compilation
- Compiler Passes
- Compiled Service Container
- Configuration Trees
- Cache Warmers
- Service Tags
- Dependency Validation

Esta arquitectura encaja especialmente bien con el objetivo de VoltStack de construir un sistema Authentication sofisticado pero eficiente.

## 372. Modelo VoltStack

VoltStack combinará:

- Laravel-like configuration ergonomics
- +;
- Symfony-like compilation
- +;

VoltStack immutable FrankenPHP runtime

## 373. Diferenciador

La configuración developer-friendly:

```php
'admin' => [
    'authenticators' => [
        'password',
        'passkey',
    ],
]
```

no deberá significar que runtime tenga que interpretar strings, descubrir servicios, ordenar authenticators y reconstruir policies en cada request.
Se transformará en:

```php
CompiledFirewall(admin)
    │
    ├── AuthenticatorRef #4
    ├── AuthenticatorRef #7
    ├── PolicyRef #2
    ├── RiskProfile #1
    └── EntryPointRef #3
```

## 374. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Authentication configuration is normalized before runtime

## 2. Schema validation and semantic validation are separate

## 3. Security conflicts fail before serving requests

## 4. Authentication dependency graphs are validated at bootstrap

## 5. Static Authentication metadata is compiled

## 6. Runtime registries are immutable

## 7. Authenticator/provider/policy resolution is precomputed where possible

## 8. Reflection and metadata scanning are removed from the production hot path

## 9. Configuration cache is versioned and fingerprinted

## 10. Secrets are represented by references rather than dumped into compiled caches

## 11. Tenant policy uses base-policy + runtime overlay

## 12. Tenant caches are bounded and versioned

## 13. Security policy composition is typed rather than generic array merging

## 14. Performance optimizations cannot bypass mandatory checks

## 15. Compiled runtime is safe to share across FrankenPHP requests

## 16. Current Authentication state is always execution-scoped

## 17. Authentication worker state is reset after every request

## 18. Cache failure and stale-state behavior are explicit

## 19. Compilation and cache build are atomic

## 20. Runtime profiling and observability are available without secret disclosure

## 21. Criterios de aceptación

El subsistema será considerado completo cuando:
22. soporte AuthenticationConfiguration;
23. soporte configuration normalization;
24. soporte schema validation;
25. soporte semantic validation;
26. soporte strict configuration mode;
27. soporte warnings/deprecations;
28. soporte dependency graph validation;
29. soporte cycle detection;
30. soporte AuthenticationCompiler;
31. soporte compiler passes;
32. compile Firewalls;
33. compile Authenticator registries;
34. compile Provider registries;
35. compile Authentication Policies;
36. compile MFA policies;
37. compile Risk policies;
38. compile Abuse policies;
39. compile Device policies;
40. compile Entry Points;
41. compile Failure mappings;
42. compile Event registries;
43. compile Hook registries;
44. compile route authentication metadata;
45. soporte immutable CompiledAuthenticationRuntime;
46. soporte configuration fingerprint;
47. soporte versioned cache;
48. soporte cache warmup;
49. soporte cache invalidation;
50. soporte atomic cache generation;
51. soporte stale cache detection;
52. soporte generated PHP cache;
53. evite raw secrets en compiled cache;
54. soporte runtime secret references;
55. soporte tenant policy overlays;
56. soporte tenant policy versioning;
57. soporte bounded tenant caches;
58. preserve framework security floors;
59. soporte fast firewall resolution;
60. soporte precomputed Authenticator selection;
61. soporte lazy dynamic subsystems;
62. soporte request-local memoization;
63. evite unsafe cross-request decision caching;
64. soporte runtime deadlines;
65. soporte performance budgets;
66. soporte Authentication Profiler;
67. soporte configuration inspection;
68. soporte deployment validation;
69. soporte rolling deployment compatibility;
70. soporte Flow state versioning;
71. soporte Session state compatibility;
72. soporte configuration drift detection;
73. soporte development hot reload;
74. soporte production freeze;
75. sea fiber-safe;
76. sea seguro bajo FrankenPHP;
77. limpie request state siempre;
78. no permita dynamic global state leakage;
79. mantenga optimizaciones separadas de security correctness.
80. Regla arquitectónica final

VoltStack deberá mantener esta separación:

```php
              BOOTSTRAP / BUILD
                     │
                     ▼
             RAW CONFIGURATION
                     │
                     ▼
                 NORMALIZE
                     │
                     ▼
                  VALIDATE
                     │
                     ▼
                  COMPILE
                     │
                     ▼
                  OPTIMIZE
                     │
                     ▼
                   FREEZE
                     │
                     ▼
            COMPILED AUTH RUNTIME
                     │
═════════════════════╪═════════════════════
                     │
               REQUEST RUNTIME
                     │
                     ▼
             STATIC O(1) LOOKUPS
                     │
                     ▼
         DYNAMIC SECURITY CONTEXT
                     │
            ┌────────┼────────┐
            ▼        ▼        ▼
         Identity   Risk    Device
            │        │        │
            └────────┼────────┘
                     ▼
          AUTHENTICATION DECISION
                     │
                     ▼
              REQUEST RESET
```

La primera regla central será:
VoltStack deberá pagar el costo de comprender su arquitectura de Authentication una vez durante compilation/bootstrap, no repetidamente durante cada request.

La segunda:
La representación compilada podrá compartirse entre workers y requests únicamente porque será inmutable y no contendrá Identity, Session, Tenant, Device, Risk ni ningún otro estado de Authentication actual.

La tercera:
Cache y optimización nunca serán excusa para usar información de seguridad obsoleta; cualquier cache de Identity, policy, session o security state deberá contar con versionado, invalidación o fallback al estado autoritativo.

La cuarta:
La configuración declarativa podrá ser sencilla para el developer, pero deberá convertirse antes de producción en una arquitectura tipada, validada, determinista y libre de ambigüedad.

La quinta:
FrankenPHP será tratado como runtime de primera clase: immutable shared state para metadata y execution-scoped state para toda Authentication actual.

Siguiente documento recomendado
La secuencia natural continúa con:
`28_AUTHENTICATION_EXTENSIBILITY_PLUGIN_PROVIDER_CUSTOM_AUTHENTICATOR_AND_INTEGRATION_SYSTEM.md`
Aquí convendría consolidar toda la extensibilidad que hemos ido introduciendo de manera distribuida:

- Custom Authenticators
- Custom Identity Providers
- Custom Credential Types
- Custom Password Hashers
- Custom Token Providers
- Custom MFA Factors
- Custom Passkey Providers
- Custom Federation Providers
- Custom Recovery Methods
- Custom Risk Providers
- Custom Abuse Detectors

Custom Device Trust Providers
Custom Session Stores
Custom Flow Stores
Custom Rate Limit Stores
Authentication Extensions
Plugin Manifests
Extension Capabilities
Service Registration
Compiler Integration
Configuration Schemas
Extension API Versions
Extension Isolation
Conflict Resolution
Extension Dependencies
Feature Discovery
Package Compatibility
Testing Contracts
Conformance Suites
Security Certification
FrankenPHP-safe Extensions
Ese documento sería importante porque, después de los 00–27, ya tenemos casi todas las piezas del Authentication Core. El siguiente paso lógico es formalizar cómo terceros amplían esas piezas sin romper las invariantes de seguridad del framework.
