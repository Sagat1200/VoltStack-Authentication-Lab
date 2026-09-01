# VoltStack Authentication System

## 04 — Authentication Manager and Orchestration System

- **Archivo:** `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del coordinador central  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`

---

## 1. Propósito

Este documento define el **Authentication Manager y el sistema de orquestación** de VoltStack.

El Authentication Manager será el punto de entrada principal para coordinar una autenticación, pero **no contendrá toda la lógica del sistema**.

Su responsabilidad será organizar y ejecutar componentes especializados:

```text
AuthenticationManager
        │
        ▼
AuthenticationOrchestrator
        │
        ├── Context Recovery
        ├── Firewall Resolver
        ├── Authenticator Resolver
        ├── Passport Processor
        ├── Identity Resolver
        ├── Credential Verifiers
        ├── Factor Verifiers
        ├── Evidence Builder
        ├── Risk Engine
        ├── Assurance Calculator
        ├── Authentication Policy Engine
        ├── Decision Engine
        ├── Challenge Manager
        ├── Context Factory
        └── Persistence Strategy
```

El objetivo es evitar una implementación monolítica similar a:

```php
final class AuthenticationManager
{
    public function authenticate($request)
    {
        // 2,000+ lines of authentication logic
    }
}
```

VoltStack utilizará una arquitectura de **orquestación explícita, componible, observable y extensible**.

---

## 2. Principio arquitectónico

La regla fundamental será:

> **El Manager coordina; los servicios especializados deciden y ejecutan.**

Por tanto:

```text
AuthenticationManager
    ≠ Password Verifier

AuthenticationManager
    ≠ Identity Provider

AuthenticationManager
    ≠ Risk Engine

AuthenticationManager
    ≠ Policy Engine

AuthenticationManager
    ≠ Session Manager

AuthenticationManager
    ≠ Authorization Engine
```

---

## 3. Separación Manager / Orchestrator

VoltStack distinguirá dos responsabilidades.

```text
AuthenticationManager
        │
        │ Public API
        ▼
AuthenticationOrchestrator
        │
        │ Internal coordination
        ▼
Authentication Pipeline
```

El Manager representa la API de alto nivel.

El Orchestrator representa la coordinación interna.

---

## 4. AuthenticationManager

El `AuthenticationManager` será responsable de:

```text
accept authentication operation
resolve request scope
prevent invalid reentrancy
invoke orchestrator
store final result
expose authenticated context
coordinate lifecycle boundaries
```

No deberá conocer los detalles internos de cada método de autenticación.

---

## 5. API conceptual

```php
interface AuthenticationManagerInterface
{
    public function authenticate(
        AuthenticationRequest $request
    ): AuthenticationResult;

    public function context(): ?AuthenticationContext;

    public function authenticated(): bool;

    public function clear(): void;
}
```

La API final podrá dividirse en contratos más pequeños.

---

## 6. Authentication operations

El Manager deberá distinguir operaciones explícitas.

Ejemplos:

```text
authenticate
recover
reauthenticate
stepUp
continueChallenge
logout
```

No es recomendable que todas se oculten detrás de un único método ambiguo.

---

## 7. AuthenticationOperation

Podrá existir:

```text
AuthenticationOperation
```

con tipos como:

```text
AUTHENTICATE
RECOVER
REAUTHENTICATE
STEP_UP
CONTINUE_CHALLENGE
LOGOUT
```

Esto permitirá adaptar el pipeline sin inferir intención de forma insegura.

---

## 8. AuthenticationOrchestrator

El `AuthenticationOrchestrator` coordinará el procesamiento completo.

Ejemplo conceptual:

```php
interface AuthenticationOrchestratorInterface
{
    public function execute(
        AuthenticationOperationContext $context
    ): AuthenticationResult;
}
```

---

## 9. Orchestration context

El orchestrator no deberá pasar decenas de argumentos entre stages.

Se utilizará un contexto interno:

```text
AuthenticationOperationContext
```

que podrá contener:

```text
operation
request
firewall
authenticator
passport
identity
verified credentials
verified factors
evidence
risk
assurance
decision
transaction
authentication context
result
```

---

## 10. OperationContext no es AuthenticationContext

Diferencia fundamental:

```text
AuthenticationOperationContext
    estado interno mutable del pipeline

AuthenticationContext
    resultado autenticado confiable
```

El primero jamás deberá exponerse como identidad autenticada.

---

## 11. Trust boundaries del OperationContext

Cada propiedad tendrá un estado de confianza distinto.

Ejemplo:

```text
request
    UNTRUSTED

passport
    STRUCTURALLY VALIDATED

identity
    RESOLVED

verifiedCredentials
    VERIFIED

evidence
    TRUSTED EVIDENCE

decision
    TRUSTED DECISION

authenticationContext
    AUTHENTICATED STATE
```

---

## 12. Stage pipeline

El orchestrator ejecutará una secuencia de stages.

```text
AuthenticationOperationContext
        ↓
Stage 1
        ↓
Stage 2
        ↓
Stage 3
        ↓
...
        ↓
AuthenticationResult
```

---

## 13. AuthenticationStageInterface

Contrato conceptual:

```php
interface AuthenticationStageInterface
{
    public function process(
        AuthenticationOperationContext $context,
        AuthenticationStageNext $next
    ): AuthenticationResult;
}
```

---

## 14. Pipeline alternativo

También podrá implementarse mediante stages que devuelvan instrucciones explícitas:

```php
interface AuthenticationStageInterface
{
    public function process(
        AuthenticationOperationContext $context
    ): StageResult;
}
```

El orchestrator interpreta posteriormente:

```text
CONTINUE
SHORT_CIRCUIT
CHALLENGE
REJECT
ERROR
```

Este modelo puede ofrecer mayor control y seguridad.

---

## 15. StageResult

Modelo conceptual:

```text
StageResult
│
├── CONTINUE
├── COMPLETE
├── SHORT_CIRCUIT
├── CHALLENGE_REQUIRED
├── STEP_UP_REQUIRED
├── REJECTED
└── ERROR
```

---

## 16. Pipeline base

Orden recomendado:

```text
01 InitializeOperation
02 RecoverExistingAuthentication
03 ResolveFirewall
04 ResolveAuthenticator
05 CreatePassport
06 ResolveIdentity
07 VerifyCredentials
08 VerifyAvailableFactors
09 BuildEvidence
10 EvaluateRisk
11 CalculateAssurance
12 EvaluateAuthenticationPolicies
13 MakeAuthenticationDecision
14 ManageChallenge
15 CreateAuthenticationContext
16 PersistAuthenticationState
17 CommitAuthentication
18 CompleteAuthentication
```

No todas las operaciones ejecutarán todos los stages.

---

## 17. Pipeline por operación

Ejemplo:

```text
AUTHENTICATE
    full pipeline

RECOVER
    recovery-focused pipeline

CONTINUE_CHALLENGE
    transaction restoration
    factor verification
    evidence continuation
    policy/assurance
    decision

STEP_UP
    existing context
    new requirements
    challenge/factor
    new context

LOGOUT
    revocation
    storage cleanup
```

---

## 18. AuthenticationPipelineRegistry

Los pipelines podrán registrarse por operación.

```text
AuthenticationPipelineRegistry
│
├── authenticate
├── recover
├── continue_challenge
├── step_up
├── reauthenticate
└── logout
```

---

## 19. Pipeline definition

Ejemplo conceptual:

```php
$pipeline->define(AuthenticationOperation::AUTHENTICATE, [
    RecoverExistingAuthenticationStage::class,
    ResolveFirewallStage::class,
    ResolveAuthenticatorStage::class,
    CreatePassportStage::class,
    ResolveIdentityStage::class,
    VerifyCredentialsStage::class,
    BuildEvidenceStage::class,
    EvaluateRiskStage::class,
    CalculateAssuranceStage::class,
    EvaluatePolicyStage::class,
    DecideAuthenticationStage::class,
    EstablishContextStage::class,
    PersistAuthenticationStage::class,
]);
```

---

## 20. Compiled pipeline

En producción la configuración podrá compilarse.

```text
Configuration
     ↓
AuthenticationCompiler
     ↓
CompiledAuthenticationPipeline
```

Esto evita reconstruir arrays, resolver aliases y validar dependencias en cada request.

---

## 21. Stage descriptors

En lugar de almacenar únicamente clases:

```text
StageDescriptor
│
├── stage id
├── service id
├── priority
├── operation
├── conditions
├── critical
└── metadata
```

---

## 22. Fixed stages

Determinadas etapas serán estructuralmente obligatorias.

Ejemplos:

```text
credential verification
before evidence trust

decision
before context creation

context creation
before authenticated persistence
```

No podrán reordenarse arbitrariamente.

---

## 23. Extension stages

Plugins podrán incorporar etapas en zonas seguras.

Ejemplo:

```text
after_identity_resolution
before_risk_evaluation
after_risk_evaluation
before_decision
after_authentication_success
```

---

## 24. Security stage graph

En vez de permitir un orden totalmente libre, VoltStack podrá definir un DAG conceptual:

```text
Passport
   ↓
Identity
   ↓
Credential Verification
   ↓
Evidence
   ↓
Risk
   ↓
Assurance
   ↓
Policy
   ↓
Decision
   ↓
Context
```

Las extensiones solo podrán insertarse donde las dependencias lo permitan.

---

## 25. Stage dependencies

Cada stage podrá declarar:

```text
requires
provides
```

Ejemplo:

```text
VerifyCredentialsStage

requires:
    passport
    identity

provides:
    verified_credentials
```

---

## 26. Pipeline validation

Durante bootstrap/compilación deberá verificarse:

```text
missing dependencies
cyclic dependencies
invalid ordering
duplicate critical stage
unknown stage
unsupported operation
```

Los errores deberán detectarse antes de recibir tráfico cuando sea posible.

---

## 27. AuthenticationProcessor

Además del orchestrator podrá existir un `AuthenticationProcessor`.

Su responsabilidad será procesar el núcleo de un Passport:

```text
Passport
    ↓
Identity
    ↓
Credentials
    ↓
Factors
    ↓
Evidence
    ↓
Risk / Assurance / Policy
    ↓
Decision
```

---

## 28. Manager vs Orchestrator vs Processor

```text
AuthenticationManager
    public system entry point

AuthenticationOrchestrator
    lifecycle coordinator

AuthenticationProcessor
    authentication-domain processing
```

Esto permite que distintos transportes compartan el mismo Processor.

---

## 29. Context recovery manager

La restauración de autenticación podrá delegarse a:

```text
AuthenticationRecoveryManager
```

que coordine:

```text
session recovery
token recovery
remember-me recovery
delegated context recovery
```

---

## 30. Recovery strategy interface

```php
interface AuthenticationRecoveryStrategyInterface
{
    public function supports(
        AuthenticationRequest $request
    ): bool;

    public function recover(
        AuthenticationRequest $request
    ): AuthenticationRecoveryResult;
}
```

---

## 31. Recovery ordering

Las strategies tendrán precedencia explícita.

Ejemplo:

```text
1. explicitly supplied bearer token
2. authentication session
3. remember-me credential
```

según firewall.

Nunca deberá depender del orden accidental del container.

---

## 32. Recovery conflicts

Si varias strategies producen identidades incompatibles:

```text
Bearer → Identity A
Session → Identity B
```

el sistema deberá aplicar una política explícita.

Posibles respuestas:

```text
reject ambiguous authentication
prefer explicit credential
firewall-specific precedence
```

---

## 33. AuthenticatorResolver

El orchestrator utilizará:

```text
AuthenticatorResolver
```

para descubrir mecanismos compatibles.

```php
interface AuthenticatorResolverInterface
{
    public function resolve(
        AuthenticationRequest $request,
        AuthenticationFirewall $firewall
    ): AuthenticatorResolution;
}
```

---

## 34. AuthenticatorResolution

Podrá representar:

```text
NONE
SINGLE
MULTIPLE_RESOLVED
AMBIGUOUS
ERROR
```

---

## 35. AuthenticatorRegistry

Los authenticators estarán registrados mediante:

```text
AuthenticatorRegistry
```

con metadata:

```text
name
service
method
priority
firewalls
transport support
capabilities
```

---

## 36. Capability-based selection

Un authenticator podrá declarar capacidades:

```text
PASSWORD
BEARER_TOKEN
PASSKEY
OIDC
API_KEY
SESSION
```

El resolver podrá usar esta metadata sin instanciar todos los servicios.

---

## 37. Lazy authenticator instantiation

Para rendimiento:

```text
Registry Metadata
      ↓
candidate resolution
      ↓
instantiate selected authenticator only
```

Esto será importante cuando existan muchos plugins.

---

## 38. PassportProcessor

El `PassportProcessor` podrá encapsular:

```text
passport structural validation
identity resolution
credential verification
factor verification
evidence construction
```

sin asumir decisiones de transporte.

---

## 39. IdentityResolver orchestration

El Identity Resolver podrá coordinar varios providers.

```text
IdentityClaim
      ↓
ProviderResolver
      ↓
IdentityProvider
      ↓
Identity
```

---

## 40. ProviderResolver

Seleccionará provider según:

```text
firewall
identity type
claim type
tenant
authenticator
configuration
```

---

## 41. Multiple providers

Un firewall podría permitir:

```text
database users
LDAP employees
federated identities
service accounts
```

pero la selección deberá ser determinista.

---

## 42. Provider fallback

Fallback entre providers deberá ser explícito.

Ejemplo seguro:

```text
corporate LDAP
   ↓ unavailable
authentication ERROR
```

No:

```text
LDAP unavailable
   ↓
silently try local admin database
```

salvo que una política específica lo defina.

---

## 43. CredentialVerificationManager

Coordinará verificadores.

```text
Credential
      ↓
CredentialVerifierRegistry
      ↓
CredentialVerifier
      ↓
CredentialVerificationResult
```

---

## 44. VerificationResult

Podrá representar:

```text
VERIFIED
INVALID
EXPIRED
REVOKED
UNSUPPORTED
ERROR
```

---

## 45. Secret lifecycle orchestration

El Manager deberá procurar que:

```text
credential extracted
     ↓
verified
     ↓
secret no longer required
     ↓
reference released
```

No deberá copiar secrets entre stages innecesariamente.

---

## 46. SensitiveParameter

Métodos que reciben secretos deberán aprovechar mecanismos como:

```php
#[\SensitiveParameter]
```

cuando corresponda.

Esto reduce exposición accidental en stack traces.

---

## 47. FactorVerificationManager

Coordinará:

```text
TOTP
passkeys
recovery codes
push approvals
hardware keys
other factors
```

---

## 48. EvidenceBuilder orchestration

Después de verification:

```text
Identity
+
VerifiedCredentials
+
VerifiedFactors
+
Provenance
        ↓
AuthenticationEvidenceBuilder
        ↓
AuthenticationEvidence
```

---

## 49. Evidence replacement

El OperationContext podrá reemplazar:

```text
evidence V1
```

por:

```text
evidence V2
```

después de un factor adicional.

No deberá mutar Evidence si este se define como immutable.

---

## 50. Risk orchestration

El `AuthenticationRiskManager` podrá ejecutar múltiples evaluators.

```text
Evidence + Environment
        ↓
RiskEvaluator A
RiskEvaluator B
RiskEvaluator C
        ↓
RiskAggregator
        ↓
AuthenticationRisk
```

---

## 51. Risk evaluator failures

Cada evaluator deberá declarar comportamiento:

```text
critical
optional
advisory
```

Pero ningún fallo podrá incrementar artificialmente la confianza.

---

## 52. Risk aggregation

El aggregator podrá usar:

```text
highest risk
weighted score
rule-based aggregation
custom profile
```

según configuración.

---

## 53. Assurance orchestration

El:

```text
AuthenticationAssuranceCalculator
```

recibirá evidencia validada.

No deberá depender de claims sin verificar.

---

## 54. Policy orchestration

El:

```text
AuthenticationPolicyManager
```

coordinará políticas aplicables.

```text
global policies
firewall policies
identity-type policies
tenant policies
risk-driven policies
operation policies
```

---

## 55. Policy resolution

Primero:

```text
PolicyResolver
```

determina cuáles aplicar.

Después:

```text
PolicyEvaluator
```

las ejecuta.

---

## 56. Policy evaluation result

```text
AuthenticationPolicyResult
│
├── SATISFIED
├── CHALLENGE_REQUIRED
├── STEP_UP_REQUIRED
├── REJECTED
└── ERROR
```

---

## 57. DecisionEngine

El Decision Engine deberá convertir los resultados del dominio en una decisión única.

Entrada:

```text
Evidence
Risk
Assurance
PolicyResult
Operation
```

Salida:

```text
AuthenticationDecision
```

---

## 58. Decision consistency

El Decision Engine será responsable de evitar estados contradictorios como:

```text
policy = REJECTED
decision = AUTHENTICATED
```

---

## 59. Decision precedence

Una posible precedencia:

```text
ERROR
   >
REJECTED
   >
CHALLENGE / STEP_UP
   >
AUTHENTICATED
```

La especificación definitiva deberá ser determinista.

---

## 60. ChallengeManager

Cuando se necesite evidencia adicional:

```text
AuthenticationDecision
      ↓
ChallengeManager
      ↓
AuthenticationTransaction
      +
AuthenticationChallenge
```

---

## 61. ChallengeManager responsibilities

```text
create challenge
persist transaction
bind challenge
validate continuation
consume challenge
expire challenge
cancel challenge
resume authentication
```

---

## 62. ChallengeFactoryRegistry

Podrá existir:

```text
ChallengeFactoryRegistry
```

para:

```text
TOTP
Passkey
Email OTP
Push
Recovery
Reauthentication
```

---

## 63. TransactionCoordinator

El:

```text
AuthenticationTransactionCoordinator
```

será responsable de las transiciones atómicas de transactions.

---

## 64. Transaction versioning

Para evitar carreras:

```text
transaction version = 3
```

al actualizar:

```text
expected version = 3
new version = 4
```

Si otra request ya la cambió:

```text
CONCURRENT_MODIFICATION
```

---

## 65. ContextFactory

Solo deberá ejecutarse cuando:

```text
decision == AUTHENTICATED
```

---

## 66. ContextFactory invariant

Debe rechazar:

```text
missing identity
missing evidence
insufficient assurance
non-authenticated decision
```

aunque el orchestrator tenga un bug.

Esto introduce defensa en profundidad.

---

## 67. ContextStorage

El Context creado deberá almacenarse en un scope específico.

Contrato conceptual:

```php
interface AuthenticationContextStorageInterface
{
    public function get(): ?AuthenticationContext;

    public function set(AuthenticationContext $context): void;

    public function clear(): void;
}
```

---

## 68. RequestScopedContextStorage

Para HTTP/SPA:

```text
RequestScopedAuthenticationContextStorage
```

---

## 69. ConnectionScopedContextStorage

Para WebSocket:

```text
ConnectionScopedAuthenticationContextStorage
```

con reglas propias de expiración.

---

## 70. ExecutionScopedContextStorage

Para CLI/queues:

```text
ExecutionScopedAuthenticationContextStorage
```

---

## 71. No global mutable context

Prohibido:

```php
AuthenticationManager::$currentUser
```

o equivalentes globales mutables.

Esto sería especialmente peligroso con FrankenPHP.

---

## 72. AuthenticationScope

VoltStack podrá definir:

```text
AuthenticationScope
```

con identificador propio.

Ejemplos:

```text
request
connection
job
command
```

---

## 73. Scope lifecycle

```text
scope created
    ↓
authentication context installed
    ↓
application execution
    ↓
scope cleanup
```

---

## 74. ScopeResetter

```php
interface AuthenticationScopeResetterInterface
{
    public function reset(AuthenticationScope $scope): void;
}
```

---

## 75. Reentrancy Guard

El Manager deberá detectar autenticación recursiva.

```text
IDLE
  ↓
AUTHENTICATING
```

Si durante el proceso se invoca nuevamente:

```text
authenticate()
```

sin operación permitida:

```text
AuthenticationReentrancyException
```

o resultado interno equivalente.

---

## 76. Reentrancy states

```text
IDLE
RECOVERING
AUTHENTICATING
CONTINUING_CHALLENGE
STEPPING_UP
LOGGING_OUT
```

---

## 77. Allowed nested operations

No toda operación anidada será inválida.

Por ejemplo, ciertos componentes podrán consultar:

```text
current context
```

durante step-up.

Pero no deberán iniciar un login independiente.

---

## 78. Partial state isolation

Durante:

```text
AUTHENTICATING
```

el ContextStorage seguirá conteniendo:

```text
previous valid context
```

o:

```text
null
```

Nunca el `OperationContext` parcial.

---

## 79. AuthenticationOperationLock

Para determinados scopes podrá existir un lock lógico:

```text
AuthenticationOperationLock
```

para evitar operaciones incompatibles simultáneas dentro del mismo scope.

---

## 80. Result propagation

Cada stage deberá propagar resultados explícitos.

No se deberá usar:

```text
null
```

para significar simultáneamente:

```text
continue
unsupported
failure
unauthenticated
```

---

## 81. Typed results

Preferencia:

```text
ContinueStageResult
CompleteStageResult
ChallengeStageResult
RejectedStageResult
ErrorStageResult
```

o una representación equivalente.

---

## 82. Exceptions

Las exceptions estarán reservadas principalmente para:

```text
programming errors
configuration errors
invariant violations
infrastructure failures where appropriate
```

No para flujos esperados como:

```text
wrong password
MFA required
expired login attempt
```

---

## 83. Exception boundary

El Manager será una frontera principal.

```text
Pipeline
   ↓ exception
AuthenticationManager
   ↓
classification
   ↓
AuthenticationErrorResult
```

---

## 84. Fatal invariant violation

Ciertas excepciones no deberán convertirse silenciosamente en fallos normales.

Ejemplo:

```text
AuthenticationContext created from rejected decision
```

deberá producir:

```text
critical security error
```

además de fail-closed.

---

## 85. Commit Coordinator

La activación de autenticación stateful podrá ser coordinada por:

```text
AuthenticationCommitCoordinator
```

---

## 86. Commit sequence

```text
Decision AUTHENTICATED
      ↓
ContextFactory
      ↓
prepare persistence
      ↓
persist required state
      ↓
activate ContextStorage
      ↓
mark transaction complete
      ↓
commit
      ↓
emit AuthenticationSucceeded
```

---

## 87. Context activation timing

Idealmente el contexto no deberá hacerse visible hasta que las operaciones críticas necesarias hayan terminado.

Evitar:

```text
ContextStorage::set()
       ↓
session persistence fails
```

dejando una request parcialmente autenticada.

---

## 88. Critical persistence

Ejemplos:

```text
session creation
session ID rotation
one-time credential consumption
transaction completion
security version recording
```

---

## 89. Commit rollback

Cuando sea posible:

```text
prepare
   ↓
failure
   ↓
rollback provisional state
```

---

## 90. Distributed commit limitations

VoltStack no deberá pretender garantizar ACID global entre:

```text
database
Redis
remote IdP
external risk service
```

cuando no sea posible.

Se usarán estrategias como:

```text
ordering
idempotency
compensation
short-lived provisional state
atomic store operations
```

---

## 91. SuccessHandlerManager

Después del commit:

```text
SuccessHandlerManager
```

podrá coordinar:

```text
redirect
token response
SPA response metadata
audit
telemetry
last-login update
```

---

## 92. Critical vs non-critical success handlers

Cada handler deberá clasificarse.

```text
CRITICAL
NON_CRITICAL
```

Ejemplo:

```text
session persistence
    CRITICAL

analytics event
    NON_CRITICAL
```

---

## 93. FailureHandlerManager

Coordinará:

```text
safe error mapping
challenge response
redirect
JSON response
audit
rate-limit feedback
```

pero el dominio no dependerá de HTTP.

---

## 94. Transport result adapters

Ejemplos:

```text
HttpAuthenticationResultAdapter
SpaAuthenticationResultAdapter
ApiAuthenticationResultAdapter
CliAuthenticationResultAdapter
WebSocketAuthenticationResultAdapter
```

---

## 95. Manager does not return Response

Preferencia arquitectónica:

```php
$result = $auth->authenticate($request);
$response = $adapter->toResponse($result);
```

en lugar de:

```php
$response = $auth->authenticate($request);
```

Esto mantiene Authentication independiente del transporte.

---

## 96. Authentication facade

La facade pública podrá ofrecer:

```php
Auth::check();

Auth::guest();

Auth::identity();

Auth::context();

Auth::assurance();
```

y APIs explícitas para operaciones.

---

## 97. Auth::user()

Por compatibilidad con el ecosistema PHP/Laravel podrá existir:

```php
Auth::user();
```

como alias conveniente de la identidad humana actual cuando sea aplicable.

Pero internamente VoltStack preferirá:

```php
Auth::identity();
```

porque Authentication soportará:

```text
users
services
machines
devices
clients
```

---

## 98. Auth::attempt()

Podrá existir una API ergonómica:

```php
Auth::attempt([
    'email' => $email,
    'password' => $password,
]);
```

pero deberá traducirse internamente a:

```text
AuthenticationRequest
    ↓
AuthenticationManager
```

y no crear un camino alternativo inseguro.

---

## 99. Auth::login()

Si se permite:

```php
Auth::login($identity);
```

deberá tener semántica cuidadosamente limitada.

No deberá significar automáticamente:

> "Confía en cualquier objeto Identity que la aplicación entregue."

Podría requerir un `TrustedAuthenticationGrant` interno.

---

## 100. TrustedAuthenticationGrant

Para casos donde el framework necesita establecer identidad sin credenciales convencionales:

```text
TrustedAuthenticationGrant
```

deberá representar explícitamente la fuente de confianza.

Ejemplos:

```text
test environment
trusted SSO bridge
internal signed delegation
system bootstrap
```

---

## 101. Programmatic authentication safety

No deberá existir una API demasiado fácil como:

```php
Auth::setUser($user);
```

que convierta cualquier objeto arbitrario en autenticación válida.

---

## 102. Testing authentication

El Testing package podrá proporcionar:

```php
actingAs($identity)
```

pero implementado mediante:

```text
TestAuthenticationGrant
```

y disponible exclusivamente en entornos/test utilities apropiados.

---

## 103. Guard compatibility layer

Para facilitar migraciones desde Laravel, VoltStack podría ofrecer:

```php
Auth::guard('web');
```

como adapter.

Sin embargo, internamente el concepto preferido será:

```text
Firewall
+
Authentication Context
+
Authenticator
```

en lugar de convertir `Guard` en el núcleo arquitectónico.

---

## 104. Firewall-specific manager

Podrá obtenerse:

```php
$auth->forFirewall('api');
```

retornando un facade/context configurado, no un segundo sistema de autenticación independiente.

---

## 105. Default firewall

El Manager podrá resolver automáticamente el firewall activo desde la request.

Por ello:

```php
Auth::check();
```

normalmente no necesitará especificarlo.

---

## 106. Manager configuration

Ejemplo conceptual:

```php
'authentication' => [
    'default_firewall' => 'web',

    'firewalls' => [
        'web' => [
            'stateful' => true,
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

La configuración final se definirá en documentos especializados.

---

## 107. ManagerFactory

Si existen múltiples application contexts podrá existir:

```text
AuthenticationManagerFactory
```

pero no deberá crear managers completos por request si pueden reutilizarse de forma segura.

---

## 108. Immutable services

Los servicios compartidos del Authentication System deberán ser preferentemente:

```text
stateless
immutable
request-independent
```

para poder permanecer en memoria con FrankenPHP.

---

## 109. Mutable state location

El estado mutable deberá vivir en:

```text
AuthenticationOperationContext
AuthenticationScope
AuthenticationTransaction
explicit persistence stores
```

No en singletons compartidos.

---

## 110. FrankenPHP model

```text
Worker
│
├── immutable AuthenticationManager
├── immutable registries
├── compiled firewall config
├── compiled pipelines
│
├── Request A Scope
│      └── AuthenticationContext A
│
└── Request B Scope
       └── AuthenticationContext B
```

---

## 111. Request leakage prevention

Al terminar A:

```text
Scope A
    ↓
reset
```

antes de B.

---

## 112. Runtime reset integration

Authentication deberá registrarse con el runtime reset system.

Ejemplo:

```text
RuntimeResetManager
│
├── ContainerScopeResetter
├── AuthenticationScopeResetter
├── EventScopeResetter
└── RequestMemoizationResetter
```

---

## 113. Memoization

Durante una request podrán memoizarse:

```text
firewall resolution
authenticator resolution
identity resolution where safe
context recovery
policy resolution
```

---

## 114. No cross-request memoization of identity state

No deberá almacenarse globalmente:

```text
Identity user:123 is active
```

sin estrategia de invalidación adecuada.

---

## 115. Registry compilation

Registries estáticos podrán compilarse:

```text
AuthenticatorRegistry
CredentialVerifierRegistry
FactorVerifierRegistry
PolicyRegistry
ChallengeFactoryRegistry
```

---

## 116. Container integration

Los registries podrán almacenar:

```text
service identifiers
factories
lazy references
metadata
```

en lugar de instancias obligatoriamente.

---

## 117. Dependency cycles

El sistema deberá prevenir ciclos como:

```text
AuthenticationManager
   ↓
IdentityProvider
   ↓
Auth facade
   ↓
AuthenticationManager
```

---

## 118. Provider dependency rule

Un IdentityProvider no deberá depender del estado de autenticación que está intentando crear.

Si necesita tenant:

```text
TenantContext
```

deberá proporcionarse explícitamente.

---

## 119. Authenticator dependency rule

Authenticators podrán depender de:

```text
credential parsers
protocol clients
factories
configuration
```

pero no deberán modificar directamente `AuthenticationContextStorage`.

---

## 120. ContextFactory ownership

Solo el orchestration/commit layer deberá instalar el contexto.

Esto evita que un authenticator haga:

```php
Auth::setUser($user);
```

durante credential verification.

---

## 121. Stage tracing

Cada stage podrá generar spans:

```text
auth.stage.resolve_firewall
auth.stage.resolve_identity
auth.stage.verify_credentials
auth.stage.evaluate_risk
auth.stage.evaluate_policy
auth.stage.commit
```

---

## 122. Stage timing

El orchestrator podrá medir:

```text
start
duration
outcome
```

sin incluir secretos.

---

## 123. Stage diagnostics

En desarrollo podrá ofrecerse una traza:

```text
Firewall: web
Authenticator: password
IdentityProvider: users
CredentialVerifier: password
Factors: password
Assurance: AAL1
Policy: MFA required
Decision: CHALLENGE_REQUIRED
```

---

## 124. Production diagnostics

En producción la información deberá limitarse según seguridad.

No deberá exponer detalles internos al cliente.

---

## 125. Authentication explainability

Para debugging autorizado podrá existir:

```text
AuthenticationDecisionExplanation
```

que describa:

```text
why authenticator selected
which policies applied
why challenge required
why assurance level obtained
```

---

## 126. Explanation security

La explicación detallada no deberá enviarse automáticamente al usuario final.

Podría revelar:

```text
account existence
internal policies
risk signals
provider topology
```

---

## 127. Event orchestration

El orchestrator podrá emitir:

```text
AuthenticationStarted
FirewallResolved
AuthenticatorResolved
IdentityResolved
CredentialVerified
FactorVerified
RiskEvaluated
AuthenticationPolicyEvaluated
AuthenticationDecisionMade
AuthenticationContextCreated
AuthenticationCommitted
AuthenticationSucceeded
AuthenticationRejected
AuthenticationErrored
```

---

## 128. Events are immutable observations

Preferentemente:

```text
event
    describes what happened
```

y no:

```text
event listener
    secretly changes decision
```

---

## 129. Security-critical interceptors

Si una extensión necesita participar en decisiones deberá usar:

```text
Policy
RiskEvaluator
CredentialVerifier
AuthenticationStage
```

según el caso.

No listeners genéricos.

---

## 130. Audit orchestration

Los eventos relevantes podrán transformarse en:

```text
AuthenticationAuditRecord
```

mediante un subscriber especializado.

---

## 131. Audit failure policy

El sistema deberá definir qué ocurre si el audit sink falla.

Para entornos regulados podrá configurarse:

```text
audit_required = true
```

haciendo ciertas operaciones fail-closed.

Para otros:

```text
buffer / retry
```

---

## 132. Rate limiter orchestration

El Manager podrá coordinar puntos de rate limiting mediante:

```text
AuthenticationRateLimitCoordinator
```

---

## 133. Rate limit dimensions

Podrán incluir:

```text
identity claim
IP/network
device
tenant
authenticator
credential identifier
transaction
```

sin depender únicamente de IP.

---

## 134. Rate limit stages

Ejemplo:

```text
pre_identity
pre_expensive_verification
post_failure
challenge_verification
recovery
```

---

## 135. Password hashing resource governance

El orchestrator deberá permitir limitar concurrencia de verificaciones costosas.

Esto ayuda a prevenir:

```text
CPU exhaustion
memory exhaustion
hashing DoS
```

---

## 136. Resource budget

Podrá existir:

```text
AuthenticationResourceBudget
```

para controlar:

```text
maximum external calls
maximum hashing operations
maximum challenge attempts
deadline
```

---

## 137. Authentication deadline

Toda operación podrá tener:

```text
AuthenticationDeadline
```

para impedir pipelines indefinidos.

---

## 138. Cancellation token

En runtimes compatibles podrá propagarse:

```text
CancellationToken
```

a servicios remotos.

---

## 139. Retry coordinator

Retries deberán ser gestionados centralmente o mediante resiliencia de infraestructura.

Nunca cada stage deberá implementar loops arbitrarios.

---

## 140. Idempotency coordination

Operaciones como:

```text
OIDC callback
magic-link consumption
refresh-token rotation
challenge completion
```

podrán recibir:

```text
AuthenticationIdempotencyKey
```

cuando sea apropiado.

---

## 141. Replay detection

El orchestrator podrá integrar:

```text
ReplayProtectionService
```

en operaciones one-time.

---

## 142. Authentication state machine enforcement

Podrá existir:

```text
AuthenticationStateTransitionGuard
```

que valide:

```text
current state
requested transition
```

---

## 143. Transition example

```text
CREDENTIALS_VERIFIED
       ↓
EVIDENCE_READY
```

válido.

```text
PASSPORT_CREATED
       ↓
AUTHENTICATED
```

inválido.

---

## 144. OperationContext state

Podrá mantener:

```text
AuthenticationLifecycleState
```

para debugging y protección de invariantes.

---

## 145. State transitions

```text
INITIALIZED
NORMALIZED
RECOVERED
FIREWALL_RESOLVED
AUTHENTICATOR_RESOLVED
PASSPORT_CREATED
IDENTITY_RESOLVED
CREDENTIALS_VERIFIED
FACTORS_VERIFIED
EVIDENCE_READY
RISK_EVALUATED
ASSURANCE_CALCULATED
POLICY_EVALUATED
DECIDED
CONTEXT_CREATED
PERSISTED
COMMITTED
COMPLETED
```

---

## 146. Terminal states

```text
COMPLETED
REJECTED
CHALLENGE_PENDING
STEP_UP_PENDING
ERROR
CANCELLED
```

---

## 147. Orchestration result

Al terminar:

```text
AuthenticationOrchestrationResult
```

podrá incluir internamente:

```text
authentication result
correlation id
final lifecycle state
timing
safe diagnostics
```

---

## 148. Public result minimization

La API pública normalmente devolverá:

```text
AuthenticationResult
```

sin exponer todo el orchestration result.

---

## 149. Manager caching

El Manager podrá cachear únicamente configuración/metadata segura:

```text
compiled pipelines
compiled firewalls
registry metadata
policy descriptors
```

No:

```text
current user
current request
current passport
current transaction
```

globalmente.

---

## 150. Hot reload

En desarrollo:

```text
config change
    ↓
invalidate compiled auth metadata
    ↓
recompile
```

En producción:

```text
precompiled immutable metadata
```

---

## 151. Boot validation

Durante bootstrap:

```text
validate firewall references
validate provider references
validate authenticator names
validate verifier dependencies
validate pipeline graph
validate policy references
validate challenge factories
```

---

## 152. Fail-fast configuration

Una configuración imposible deberá impedir iniciar la aplicación cuando sea razonable.

Ejemplo:

```text
firewall requires authenticator "passkey"
but authenticator is not registered
```

---

## 153. Runtime failures remain fail-closed

Aunque bootstrap valide configuración, un provider remoto puede fallar posteriormente.

Resultado:

```text
ERROR
```

no bypass.

---

## 154. Manager concurrency model

Un mismo Manager podrá reutilizarse concurrentemente siempre que:

```text
manager = stateless
operation state = scoped
registries = immutable/thread-safe
context storage = scope-aware
```

---

## 155. Fiber/coroutine safety

Si VoltStack soporta fibers/coroutines, el ContextStorage deberá evitar que contextos se mezclen entre ejecuciones concurrentes.

No deberá depender únicamente de globals PHP tradicionales.

---

## 156. Context propagation

Cuando una ejecución hija necesite identidad:

```text
parent context
     ↓
explicit propagation
     ↓
child execution
```

No deberá heredarse implícitamente en todos los casos.

---

## 157. Service-to-service propagation

El Manager no deberá simplemente serializar:

```text
AuthenticationContext
```

y enviarlo a otro servicio.

Deberá utilizar:

```text
delegation token
signed assertion
service credential
```

según arquitectura.

---

## 158. Manager security boundaries

El Manager será responsable de preservar:

```text
untrusted input boundary
verification boundary
decision boundary
context establishment boundary
persistence boundary
scope boundary
```

---

## 159. Orchestration invariant 01

Un stage no podrá declarar autenticación completa sin pasar por Decision Engine y Context Factory.

---

## 160. Orchestration invariant 02

Solo evidencia verificada podrá pasar al Assurance Calculator como evidencia confiable.

---

## 161. Orchestration invariant 03

El ContextStorage nunca contendrá estado parcial.

---

## 162. Orchestration invariant 04

Una excepción inesperada antes del commit produce fail-closed.

---

## 163. Orchestration invariant 05

El Manager nunca deberá confiar en orden accidental del Service Container.

---

## 164. Orchestration invariant 06

Los pipelines críticos serán validados antes de ejecución.

---

## 165. Orchestration invariant 07

Los plugins no podrán saltarse las fronteras obligatorias de verificación.

---

## 166. Orchestration invariant 08

Los scopes deberán limpiarse siempre.

---

## 167. Orchestration invariant 09

Una operación de step-up no reemplaza el contexto existente hasta completar exitosamente el nuevo nivel.

---

## 168. Orchestration invariant 10

Los failures esperados deberán representarse mediante resultados tipados, no excepciones genéricas.

---

## 169. Orchestration invariant 11

Las exceptions de infraestructura nunca producirán fallback de seguridad implícito.

---

## 170. Orchestration invariant 12

Los eventos observacionales no modificarán decisiones críticas.

---

## 171. Orchestration invariant 13

Los secrets tendrán propagación mínima.

---

## 172. Orchestration invariant 14

El Manager no ejecutará Authorization.

---

## 173. Orchestration invariant 15

Toda autenticación exitosa tendrá provenance suficiente para auditoría.

---

## 174. Ejemplo completo — Password

```text
Application
    ↓
AuthenticationManager::authenticate()
    ↓
OperationContext created
    ↓
ReentrancyGuard
    ↓
AuthenticationOrchestrator
    ↓
ResolveFirewallStage
    ↓
AuthenticatorResolver
    ↓
PasswordAuthenticator
    ↓
AuthenticationPassport
    ↓
IdentityResolver
    ↓
UserIdentity
    ↓
CredentialVerificationManager
    ↓
VerifiedPasswordCredential
    ↓
EvidenceBuilder
    ↓
AuthenticationEvidence
    ↓
RiskManager
    ↓
LOW
    ↓
AssuranceCalculator
    ↓
AAL1
    ↓
PolicyManager
    ↓
SATISFIED
    ↓
DecisionEngine
    ↓
AUTHENTICATED
    ↓
ContextFactory
    ↓
AuthenticationContext
    ↓
CommitCoordinator
    ↓
AuthenticationSession
    ↓
ContextStorage
    ↓
AuthenticationSucceeded
```

---

## 175. Ejemplo completo — MFA

```text
AuthenticationManager
    ↓
Password pipeline
    ↓
Evidence(password)
    ↓
AAL1
    ↓
Policy requires AAL2
    ↓
Decision = CHALLENGE_REQUIRED
    ↓
ChallengeManager
    ↓
TransactionCoordinator
    ↓
TotpChallenge
    ↓
ChallengeRequiredResult

Next Request
    ↓
AuthenticationManager::continueChallenge()
    ↓
Transaction restored
    ↓
FactorVerificationManager
    ↓
VerifiedFactor(TOTP)
    ↓
Evidence V2
    ↓
Assurance AAL2
    ↓
Policy SATISFIED
    ↓
Decision AUTHENTICATED
    ↓
ContextFactory
    ↓
CommitCoordinator
```

---

## 176. Ejemplo — Existing session

```text
AuthenticationManager
    ↓
RecoveryManager
    ↓
SessionRecoveryStrategy
    ↓
AuthenticationSession
    ↓
validity checks
    ↓
security version
    ↓
tenant binding
    ↓
Context reconstruction
    ↓
AuthenticationContext
    ↓
short-circuit
```

---

## 177. Ejemplo — Step-up

```text
Current Context
AAL1
    ↓
AuthenticationManager::stepUp()
    ↓
OperationContext(previous context)
    ↓
Policy requirements
    ↓
Passkey Challenge
    ↓
Passkey verified
    ↓
Evidence extended
    ↓
AAL2
    ↓
new AuthenticationContext
    ↓
atomic replacement
```

---

## 178. Ejemplo — Error

```text
AuthenticationManager
    ↓
IdentityResolver
    ↓
Database unavailable
    ↓
Infrastructure Exception
    ↓
Manager Exception Boundary
    ↓
AuthenticationError
    ↓
audit + telemetry
    ↓
ERROR Result
```

Nunca:

```text
Database unavailable
    ↓
skip authentication
    ↓
allow request
```

---

## 179. Component map

```text
AuthenticationManager
│
├── AuthenticationOrchestrator
│   ├── AuthenticationPipelineRegistry
│   ├── AuthenticationStateTransitionGuard
│   └── AuthenticationCommitCoordinator
│
├── AuthenticationRecoveryManager
│   └── RecoveryStrategyRegistry
│
├── FirewallResolver
│
├── AuthenticatorResolver
│   └── AuthenticatorRegistry
│
├── AuthenticationProcessor
│   ├── PassportProcessor
│   ├── IdentityResolver
│   ├── CredentialVerificationManager
│   ├── FactorVerificationManager
│   └── EvidenceBuilder
│
├── AuthenticationRiskManager
├── AuthenticationAssuranceCalculator
├── AuthenticationPolicyManager
├── AuthenticationDecisionEngine
├── AuthenticationChallengeManager
├── AuthenticationTransactionCoordinator
├── AuthenticationContextFactory
├── AuthenticationContextStorage
│
├── SuccessHandlerManager
├── FailureHandlerManager
│
├── AuthenticationEventDispatcher
├── AuthenticationAuditManager
├── AuthenticationRateLimitCoordinator
└── AuthenticationScopeResetter
```

---

## 180. Namespace propuesto

```text
VoltStack\Quantum\Auth\Manager
VoltStack\Quantum\Auth\Orchestration
VoltStack\Quantum\Auth\Pipeline
VoltStack\Quantum\Auth\Pipeline\Stage
VoltStack\Quantum\Auth\Recovery
VoltStack\Quantum\Auth\Authenticator
VoltStack\Quantum\Auth\Identity
VoltStack\Quantum\Auth\Credentials
VoltStack\Quantum\Auth\Factors
VoltStack\Quantum\Auth\Evidence
VoltStack\Quantum\Auth\Risk
VoltStack\Quantum\Auth\Assurance
VoltStack\Quantum\Auth\Policy
VoltStack\Quantum\Auth\Decision
VoltStack\Quantum\Auth\Challenge
VoltStack\Quantum\Auth\Transaction
VoltStack\Quantum\Auth\Context
VoltStack\Quantum\Auth\Persistence
VoltStack\Quantum\Auth\Events
VoltStack\Quantum\Auth\Audit
```

---

## 181. Estructura interna sugerida

```text
src/Quantum/Auth/
│
├── Manager/
│   ├── AuthenticationManager.php
│   └── AuthenticationManagerInterface.php
│
├── Orchestration/
│   ├── AuthenticationOrchestrator.php
│   ├── AuthenticationOperation.php
│   ├── AuthenticationOperationContext.php
│   ├── AuthenticationCommitCoordinator.php
│   └── AuthenticationStateTransitionGuard.php
│
├── Pipeline/
│   ├── AuthenticationPipeline.php
│   ├── AuthenticationPipelineRegistry.php
│   ├── AuthenticationStageInterface.php
│   ├── StageResult.php
│   └── Stage/
│
├── Recovery/
├── Authenticator/
├── Identity/
├── Credentials/
├── Factors/
├── Evidence/
├── Risk/
├── Assurance/
├── Policy/
├── Decision/
├── Challenge/
├── Transaction/
├── Context/
├── Persistence/
├── Events/
└── Audit/
```

---

## 182. Design goal — Laravel ergonomics

La capa pública deberá poder ofrecer una experiencia sencilla:

```php
if (Auth::check()) {
    $identity = Auth::identity();
}
```

o:

```php
$result = Auth::attempt([
    'email' => $email,
    'password' => $password,
]);
```

sin sacrificar la arquitectura interna.

---

## 183. Design goal — Symfony composability

Internamente deberá conservar la composición mediante:

```text
authenticators
providers
passports
verifiers
policies
events
firewalls
```

sin que el desarrollador quede atado a una implementación única.

---

## 184. Design goal — VoltStack evolution

VoltStack añadirá como principios centrales:

```text
typed authentication results
explicit assurance
risk-aware authentication
multi-step transactions
step-up authentication
service/machine identities
scope isolation
persistent-runtime safety
compiled orchestration
security-constrained extension points
```

---

## 185. Qué NO deberá hacer el AuthenticationManager

El Manager no deberá convertirse en:

```text
password hasher
database repository
JWT library
HTTP middleware
authorization engine
session store
OAuth client
TOTP implementation
WebAuthn implementation
rate limiter
audit database
```

Coordinará esos subsistemas mediante contratos.

---

## 186. Criterios de aceptación

El sistema de Manager y Orchestration será considerado correcto cuando:

1. exista un punto de entrada coherente para Authentication;
2. Manager y Orchestrator tengan responsabilidades distintas;
3. el Manager permanezca pequeño;
4. el pipeline sea explícito;
5. las etapas críticas tengan orden protegido;
6. existan extension points controlados;
7. los pipelines puedan validarse durante bootstrap;
8. puedan compilarse para producción;
9. exista request/execution scoping;
10. sea seguro con FrankenPHP;
11. detecte reentrancy;
12. no exponga estados parciales;
13. soporte recovery;
14. soporte authentication inicial;
15. soporte challenge continuation;
16. soporte step-up;
17. soporte reauthentication;
18. soporte logout;
19. gestione commit coherentemente;
20. distinga critical y non-critical handlers;
21. soporte observabilidad;
22. soporte auditoría;
23. permita plugins;
24. minimice propagación de secrets;
25. mantenga Authentication separado de Authorization.

---

## 187. Regla arquitectónica final

La relación principal será:

```text
                    PUBLIC API
                        │
                        ▼
              AuthenticationManager
                        │
                        ▼
             AuthenticationOrchestrator
                        │
                        ▼
              Validated Auth Pipeline
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
 Authentication     Security         Infrastructure
   Domain           Services           Adapters
        │               │                │
        └───────────────┼────────────────┘
                        ▼
              AuthenticationDecision
                        │
                        ▼
              AuthenticationContext
```

El principio central será:

> **VoltStack deberá ofrecer una API de autenticación tan sencilla como la de Laravel, una arquitectura interna tan componible como la de Symfony, pero con un modelo explícito de evidencia, assurance, riesgo, transacciones, step-up, aislamiento de runtime y orquestación segura.**

---

## 188. Próximo documento

El siguiente documento recomendado será:

```text
05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md
```

Su responsabilidad será definir el subsistema que determina **qué entorno de Authentication se aplica a cada ejecución**, incluyendo:

```text
Authentication Firewall
Guard compatibility
firewall matching
request matching
host/path/method/transport matching
firewall precedence
stateful/stateless/hybrid modes
context resolution
firewall-specific providers
firewall-specific authenticators
lazy authentication
anonymous/public execution
firewall isolation
multi-firewall applications
tenant-aware firewalls
compiled firewall matcher
firewall cache
persistent-runtime behavior
```

Este documento será especialmente importante porque permitirá tomar la ergonomía de los **guards de Laravel** y la potencia de los **firewalls de Symfony**, sin hacer que ninguno de los dos conceptos controle por sí solo toda la arquitectura de Authentication.
