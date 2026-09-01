# VoltStack Authentication System

## 07 — Authenticator Resolution, Selection and Priority System

- **Archivo:** `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del sistema de resolución, selección, precedencia y conflictos de Authenticators  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema encargado de responder:

> **¿Qué Authenticator debe procesar una operación concreta cuando un Firewall permite múltiples mecanismos de autenticación?**

La existencia de varios Authenticators introduce problemas que no deberán resolverse mediante un simple:

```php
foreach ($authenticators as $authenticator) {
    if ($authenticator->supports($request)) {
        return $authenticator;
    }
}
```

Ese enfoque hace que la seguridad dependa accidentalmente del orden de registro.

VoltStack utilizará un modelo explícito de:

```text
Candidate Discovery
        ↓
Support Evaluation
        ↓
Credential Classification
        ↓
Conflict Detection
        ↓
Precedence Policy
        ↓
Authenticator Selection
        ↓
Selection Lock
```

---

## 2. Objetivos

El sistema deberá proporcionar:

```text
deterministic selection
explicit priority
credential precedence
explicit/implicit distinction
ambiguity detection
conflict detection
downgrade prevention
firewall-specific policies
multi-credential handling
compiled resolution
fast candidate discovery
safe fallback semantics
debug explainability
persistent-runtime safety
```

---

## 3. Principio fundamental

La regla principal será:

> **La selección de Authenticator es una decisión de seguridad, no un detalle de iteración.**

Por tanto:

```text
registration order
service discovery order
container order
plugin loading order
```

nunca deberán determinar accidentalmente qué mecanismo gana.

---

## 4. Separación de responsabilidades

```text
AuthenticatorRegistry
    conoce los Authenticators existentes

Firewall
    define cuáles están permitidos

AuthenticatorResolver
    descubre candidatos

AuthenticatorSelectionPolicy
    decide precedencia

AuthenticatorConflictDetector
    detecta combinaciones inseguras

AuthenticationManager
    ejecuta el Authenticator seleccionado
```

---

## 5. AuthenticatorResolver

Contrato conceptual:

```php
interface AuthenticatorResolverInterface
{
    public function resolve(
        AuthenticatorResolutionRequest $request
    ): AuthenticatorResolution;
}
```

---

## 6. AuthenticatorResolutionRequest

Podrá contener:

```php
final readonly class AuthenticatorResolutionRequest
{
    public function __construct(
        public AuthenticationRequest $request,
        public ResolvedAuthenticationFirewall $firewall,
        public AuthenticationOperation $operation,
        public ?AuthenticationTransaction $transaction = null,
    ) {}
}
```

---

## 7. AuthenticatorResolution

No deberá retornar simplemente:

```text
Authenticator|null
```

porque perdería información crítica.

Se utilizará un resultado estructurado.

---

## 8. Resolution outcomes

Posibles estados:

```text
SELECTED
NONE
MULTIPLE
AMBIGUOUS
CONFLICT
MALFORMED
UNSUPPORTED
LOCKED
ERROR
```

---

## 9. Resultado conceptual

```php
final readonly class AuthenticatorResolution
{
    public function __construct(
        public AuthenticatorResolutionStatus $status,
        public ?AuthenticatorDescriptor $selected,
        public array $candidates,
        public array $supportResults,
        public ?AuthenticatorResolutionReason $reason,
    ) {}
}
```

---

## 10. Pipeline general

```text
AuthenticationRequest
        │
        ▼
Resolved Firewall
        │
        ▼
Allowed Authenticator Set
        │
        ▼
Candidate Discovery
        │
        ▼
Support Evaluation
        │
        ▼
Credential Classification
        │
        ▼
Malformed Input Detection
        │
        ▼
Conflict Detection
        │
        ▼
Selection Policy
        │
        ▼
Priority / Specificity
        │
        ▼
Selected Authenticator
        │
        ▼
Selection Lock
```

---

## 11. Firewall como primera frontera

El Resolver solo considerará Authenticators permitidos por el Firewall.

Ejemplo:

```text
Registry:
    session
    password
    bearer
    api_key
    passkey
    oidc

Firewall web:
    session
    password
    passkey
```

Entonces solo se evaluarán:

```text
session
password
passkey
```

---

## 12. No global fallback

Si ningún Authenticator permitido soporta la request:

```text
NONE
```

No deberá buscarse automáticamente en todo el Registry.

---

## 13. Razón de seguridad

Si el Firewall `api` permite:

```text
bearer
api_key
```

y llega una session cookie válida, el sistema no deberá pensar:

```text
"SessionAuthenticator existe en el framework,
probemos con él."
```

La session está fuera de la frontera de seguridad de ese Firewall.

---

## 14. Candidate Discovery

La primera fase identifica cuáles Authenticators podrían ser relevantes sin ejecutar todavía `supports()` sobre todos.

---

## 15. Candidate hints

Podrán utilizarse señales como:

```text
Authorization header
session cookie
route
HTTP method
content type
authentication transaction
protocol callback
client certificate
dedicated API-key header
request attributes
transport
```

---

## 16. Candidate hints no son decisiones

Ejemplo:

```text
Authorization header exists
```

puede indicar:

```text
BearerTokenAuthenticator
```

pero todavía deberá verificarse:

```text
scheme = Bearer
structure valid
firewall allows bearer
```

---

## 17. CandidateDiscoveryInterface

Conceptualmente:

```php
interface AuthenticatorCandidateDiscoveryInterface
{
    public function discover(
        AuthenticatorResolutionRequest $request
    ): AuthenticatorCandidateSet;
}
```

---

## 18. CandidateSet

Podrá contener:

```text
descriptors
discovery reasons
credential channels detected
protocol hints
```

---

## 19. Compiled candidate indexes

VoltStack podrá compilar índices:

```text
header:authorization
    → bearer

cookie:session
    → session

route:login
    → password

route:webauthn.callback
    → passkey

route:oidc.callback
    → oidc

tls:client_certificate
    → client_certificate
```

---

## 20. Candidate discovery performance

Con muchos Authenticators, el objetivo será aproximarse a:

```text
O(1)
```

para casos comunes.

---

## 21. Support Evaluation

Después de descubrir candidatos:

```text
Authenticator::supports()
```

determinará si cada uno realmente reconoce la operación.

---

## 22. Support states

Se recomienda:

```text
SUPPORTED
NOT_SUPPORTED
MALFORMED
AMBIGUOUS
ERROR
```

---

## 23. NOT_SUPPORTED

Significa:

> El input no intenta utilizar este Authenticator.

Esto permite continuar evaluando otros candidatos.

---

## 24. MALFORMED

Significa:

> El input parece intentar utilizar este mecanismo, pero su estructura es inválida.

Ejemplo:

```text
Authorization: Bearer
```

Esto no deberá convertirse en:

```text
NOT_SUPPORTED
```

---

## 25. Por qué MALFORMED bloquea fallback

Si se tratara como `NOT_SUPPORTED`:

```text
Malformed Bearer
       ↓
ignored
       ↓
SessionAuthenticator
       ↓
valid session
       ↓
authenticated
```

Esto crea un downgrade implícito.

---

## 26. ERROR

Representa un problema inesperado en la evaluación.

Normalmente:

```text
fail closed
```

para operaciones protegidas.

---

## 27. AMBIGUOUS support

Un Authenticator podrá detectar que el mismo canal contiene múltiples interpretaciones válidas o contradictorias.

Ejemplo:

```text
duplicate protocol parameters
```

---

## 28. Credential classification

Los mecanismos deberán clasificarse según cómo aparecen en la request.

Categorías recomendadas:

```text
EXPLICIT
IMPLICIT
CONTINUATION
INFRASTRUCTURE
RECOVERY
```

---

## 29. Explicit Credential

Una credencial que el cliente presenta intencionalmente en la operación.

Ejemplos:

```text
Authorization: Bearer ...
X-API-Key: ...
password login payload
passkey assertion
OTP
```

---

## 30. Implicit Credential

Estado que acompaña automáticamente a la request.

Ejemplos:

```text
session cookie
remember-me cookie
ambient browser credential
```

---

## 31. Continuation Credential

Pertenece a un flow iniciado previamente.

Ejemplos:

```text
OIDC callback
SAML response
WebAuthn assertion transaction
magic-link transaction
```

---

## 32. Infrastructure Credential

Proviene de una frontera de infraestructura confiable.

Ejemplos:

```text
mTLS certificate
workload identity
trusted gateway assertion
```

---

## 33. Recovery Credential

Permite recuperar acceso.

Ejemplos:

```text
recovery code
recovery link
```

Deberá tratarse con policies específicas.

---

## 34. Explicit vs implicit precedence

Una política común será:

```text
EXPLICIT
    >
IMPLICIT
```

Ejemplo:

```text
Bearer Token
    >
Session Cookie
```

cuando ambos estén permitidos.

---

## 35. Esto no será universal

El Firewall deberá poder definir la política.

Ejemplo:

```php
'credential_precedence' => [
    'bearer_token',
    'session',
],
```

---

## 36. CredentialChannel

Será útil modelar el lugar donde aparece la credencial.

Ejemplos:

```text
AUTHORIZATION_HEADER
SESSION_COOKIE
API_KEY_HEADER
LOGIN_BODY
TLS_CLIENT_CERTIFICATE
PROTOCOL_CALLBACK
QUERY_PARAMETER
```

---

## 37. Channel security

El canal podrá influir en la policy.

Ejemplo:

```text
API key in header
```

puede permitirse.

```text
API key in query
```

puede rechazarse aunque el Authenticator técnicamente pudiera parsearla.

---

## 38. CredentialPresence

Antes de seleccionar podrá construirse un mapa:

```text
Authorization Bearer → present
Session Cookie       → present
API Key              → absent
Client Certificate   → absent
```

---

## 39. Multiple credentials

La presencia simultánea de varias credenciales no será automáticamente válida.

El Firewall deberá definir una:

```text
MultipleCredentialPolicy
```

---

## 40. Policies recomendadas

```text
REJECT_MULTIPLE
PREFER_EXPLICIT
PREFER_CONFIGURED_ORDER
ALLOW_COMPATIBLE
REQUIRE_SAME_IDENTITY
```

---

## 41. REJECT_MULTIPLE

Ejemplo:

```text
session + bearer
```

produce:

```text
AUTHENTICATOR_CONFLICT
```

antes de verificar cualquiera.

Adecuado para APIs estrictas.

---

## 42. PREFER_EXPLICIT

Ejemplo:

```text
bearer + session
```

selecciona:

```text
bearer
```

La session queda ignorada para esa operación.

---

## 43. PREFER_CONFIGURED_ORDER

Ejemplo:

```text
precedence:
    client_certificate
    bearer
    api_key
```

El primer mecanismo presente y soportado gana.

---

## 44. ALLOW_COMPATIBLE

Permite evaluar múltiples mecanismos cuando forman una combinación prevista.

Ejemplo:

```text
mTLS
+
service token
```

si la policy exige ambos.

Esto ya se acerca a autenticación compuesta.

---

## 45. REQUIRE_SAME_IDENTITY

Puede aceptar varios mecanismos únicamente si todos se verifican y representan la misma identidad.

Ejemplo:

```text
mTLS service certificate
+
signed service token
```

ambos deben representar:

```text
Service A
```

---

## 46. Multi-authenticator execution

Cuando una policy requiere más de un Authenticator:

```text
AuthenticatorResolution
```

podrá devolver un:

```text
AuthenticatorSelectionPlan
```

en lugar de un único Authenticator.

---

## 47. SelectionPlan

Conceptualmente:

```php
final readonly class AuthenticatorSelectionPlan
{
    public function __construct(
        public array $authenticators,
        public AuthenticatorCombinationMode $mode,
    ) {}
}
```

---

## 48. Combination modes

Podrán existir:

```text
SINGLE
ALL_REQUIRED
ANY_ONE
SAME_IDENTITY
ORDERED_CHAIN
```

---

## 49. SINGLE

Caso habitual:

```text
Bearer only
Password only
Session only
```

---

## 50. ALL_REQUIRED

Ejemplo machine-to-machine:

```text
mTLS
AND
signed token
```

---

## 51. ANY_ONE

Puede utilizarse para mecanismos alternativos explícitamente equivalentes.

Sin embargo, deberá evitarse usarlo como fallback después de credential failure.

---

## 52. SAME_IDENTITY

Todos los mecanismos seleccionados deberán verificar la misma identidad.

---

## 53. ORDERED_CHAIN

Representa protocolos donde una evidencia depende de otra fase.

No deberá utilizarse para modelar MFA general si el Assurance System puede hacerlo mejor.

---

## 54. Priority

Cada Authenticator podrá tener prioridad.

Ejemplo:

```text
client_certificate = 300
bearer_token       = 200
api_key            = 100
session             = 50
```

---

## 55. Priority no equivale a trust

Un número mayor no significa necesariamente que el mecanismo sea criptográficamente más fuerte.

Priority significa:

> **precedencia de selección dentro de un contexto configurado.**

---

## 56. Assurance separado de Priority

Ejemplo:

```text
SessionAuthenticator priority = 300
PasskeyAuthenticator priority = 100
```

no implica:

```text
session > passkey security
```

Assurance se calcula en otro subsistema.

---

## 57. Global vs Firewall priority

Podrá existir:

```text
default authenticator priority
```

pero el Firewall deberá poder sobrescribirla.

---

## 58. Ejemplo

```php
'api' => [
    'authenticators' => [
        'bearer_token' => [
            'priority' => 200,
        ],

        'api_key' => [
            'priority' => 100,
        ],
    ],
];
```

---

## 59. Specificity

Cuando dos Authenticators tienen igual priority, podrá considerarse especificidad.

Ejemplo:

```text
GenericBearerAuthenticator
VendorSignedBearerAuthenticator
```

Si la request contiene una marca inequívoca del segundo, este puede ser más específico.

---

## 60. Specificity contract

Podrá representarse mediante:

```text
AuthenticatorMatchSpecificity
```

con niveles:

```text
EXACT
STRONG
GENERIC
FALLBACK
```

---

## 61. Evitar heurísticas ocultas

La especificidad deberá derivarse de reglas explícitas, no de inferencias difíciles de auditar.

---

## 62. Selection ranking

Un ranking conceptual podría usar:

```text
1. transaction lock
2. malformed/conflict checks
3. credential policy
4. explicit precedence
5. configured priority
6. specificity
7. deterministic declaration index
```

El último punto solo deberá usarse como tie-breaker seguro cuando no exista ambigüedad semántica.

---

## 63. Ambiguity

Si dos Authenticators siguen siendo equivalentes después de aplicar todas las reglas:

```text
AMBIGUOUS
```

será preferible a escoger uno arbitrariamente.

---

## 64. Ejemplo de ambigüedad

```text
Authenticator A:
    header X-Credential

Authenticator B:
    header X-Credential

same priority
same specificity
same firewall
```

Resultado:

```text
AUTHENTICATOR_AMBIGUOUS
```

---

## 65. Compile-time ambiguity detection

Muchas ambigüedades deberán detectarse al compilar.

Ejemplo:

```text
same trigger
same channel
same priority
same specificity
```

---

## 66. Runtime ambiguity

Algunas solo pueden detectarse con input concreto.

Ejemplo:

```text
two protocol formats overlapping
```

Estas deberán fallar de manera cerrada.

---

## 67. Conflict vs Ambiguity

No son iguales.

```text
AMBIGUITY:
    system cannot determine intended mechanism

CONFLICT:
    multiple mechanisms are clearly present
    but policy says they cannot coexist
```

---

## 68. Ejemplo de Ambiguity

```text
X-Token: abc
```

puede corresponder a dos Authenticators.

---

## 69. Ejemplo de Conflict

```text
Authorization: Bearer abc
Cookie: session=xyz
```

ambos mecanismos están claramente identificados, pero `REJECT_MULTIPLE` los prohíbe.

---

## 70. Malformed vs Conflict

También distintos.

```text
MALFORMED:
    one attempted mechanism has invalid structure

CONFLICT:
    multiple validly identifiable mechanisms collide
```

---

## 71. Selection policy object

Se recomienda:

```php
interface AuthenticatorSelectionPolicyInterface
{
    public function select(
        AuthenticatorCandidateSet $candidates,
        AuthenticatorSelectionContext $context
    ): AuthenticatorResolution;
}
```

---

## 72. SelectionContext

Podrá contener:

```text
firewall
operation
transport
tenant
transaction
credential presence
support results
```

---

## 73. Firewall-specific policy

Cada Firewall podrá referenciar:

```text
authenticator_selection_policy
```

Ejemplo:

```php
'selection' => [
    'multiple_credentials' => 'prefer_explicit',
    'precedence' => [
        'passkey',
        'password',
        'session',
    ],
],
```

---

## 74. Default policies por tipo de Firewall

VoltStack podrá ofrecer defaults seguros.

Ejemplo conceptual:

```text
web stateful:
    explicit > session > remember-me

api stateless:
    reject ambiguous multiple explicit credentials

service:
    require configured service credential policy
```

---

## 75. Defaults deberán ser visibles

No deberán existir reglas críticas ocultas en código.

Tooling deberá poder mostrar:

```text
Effective Authenticator Selection Policy
```

---

## 76. Credential downgrade

Downgrade ocurre cuando el sistema abandona una credencial presentada y termina autenticando mediante un mecanismo menos exigente.

Ejemplo:

```text
invalid bearer
    ↓
valid session
    ↓
authenticated
```

---

## 77. Regla anti-downgrade

Una vez que una credencial explícita ha sido seleccionada:

```text
verification failure
```

deberá terminar esa tentativa.

No se reabre la selección automáticamente.

---

## 78. Selection Lock

VoltStack introducirá conceptualmente:

```text
AuthenticatorSelectionLock
```

una vez elegido el mecanismo.

---

## 79. Selection Lock purpose

Impide:

```text
Authenticator A selected
    ↓
credential invalid
    ↓
resolver invoked again
    ↓
Authenticator B succeeds
```

---

## 80. Lock scope

El lock pertenecerá a:

```text
AuthenticationOperation
```

o:

```text
AuthenticationTransaction
```

según el flow.

---

## 81. SelectionLock data

Podrá registrar:

```text
selected authenticator
selection policy
credential channel
timestamp
operation id
transaction id
```

sin secretos.

---

## 82. Retry

Un retry explícito sí podrá crear una nueva operación.

Ejemplo:

```text
password failed
user chooses passkey
```

Eso es distinto de fallback silencioso.

---

## 83. Retry semantics

```text
Operation 1:
    password
    → failure

Operation 2:
    passkey
    → new explicit user choice
```

---

## 84. Challenge continuation lock

Un flow OIDC iniciado con:

```text
OidcAuthenticator:microsoft
```

deberá regresar al mismo Authenticator/provider.

No volver a resolver arbitrariamente.

---

## 85. Transaction-bound selection

`AuthenticationTransaction` podrá guardar:

```text
authenticator = oidc
provider = microsoft
```

---

## 86. WebAuthn continuation

Igualmente:

```text
begin passkey
    ↓
transaction locked to passkey
    ↓
assertion callback
```

---

## 87. Magic-link continuation

El link deberá referenciar una transaction/purpose concreto.

No deberá permitir cambiar de Authenticator durante consumo.

---

## 88. OIDC provider confusion

Si se inició:

```text
provider = Microsoft
```

y callback intenta declarar:

```text
provider = Google
```

deberá rechazarse.

---

## 89. Identity conflict

Puede detectarse después de verification.

Ejemplo:

```text
mTLS → Service A
token → Service B
```

Si la policy exige misma identidad:

```text
IDENTITY_CONFLICT
```

---

## 90. IdentityConflictDetector

Podrá existir:

```php
interface AuthenticationIdentityConflictDetectorInterface
{
    public function compare(
        array $verifiedEvidences
    ): IdentityCompatibilityResult;
}
```

---

## 91. Identity equality

No deberá compararse únicamente:

```text
id = 123
```

si existen providers o realms distintos.

Podrá usarse:

```text
IdentityReference
```

con:

```text
provider
realm
type
identifier
tenant scope
```

---

## 92. Cross-tenant identity conflict

```text
tenant A / user 5
```

no equivale necesariamente a:

```text
tenant B / user 5
```

---

## 93. Compatible evidence

Múltiples evidencias pueden representar la misma identidad:

```text
certificate subject → service:billing
token subject       → service:billing
```

y ser combinables.

---

## 94. Authentication evidence composition

Cuando varios mecanismos son requeridos:

```text
Evidence A
+
Evidence B
    ↓
Evidence Set
```

El Assurance System calculará el resultado final.

---

## 95. Resolver no calcula Assurance

Aunque conozca tipos de Authenticator, no deberá decidir:

```text
AAL2
```

Eso corresponde al subsistema de Assurance.

---

## 96. Resolver tampoco autoriza

No deberá considerar:

```text
role
permission
resource ownership
business policy
```

---

## 97. Operation intent

La selección puede depender de:

```text
LOGIN
RECOVER
STEP_UP
REAUTHENTICATE
SESSION_RECOVERY
SERVICE_AUTHENTICATION
```

---

## 98. Ejemplo

En `STEP_UP`:

```text
session
```

no debería seleccionarse como nueva prueba simplemente porque está presente.

Podrán permitirse:

```text
passkey
totp
hardware key
```

---

## 99. Intent-aware filtering

Los AuthenticatorDescriptor podrán declarar:

```text
supported intents
```

---

## 100. Session recovery intent

Para:

```text
SESSION_RECOVERY
```

puede evaluarse únicamente:

```text
session
remember_me
```

según Firewall.

---

## 101. Login intent

Para:

```text
LOGIN
```

pueden evaluarse:

```text
password
passkey
oidc
saml
```

---

## 102. Service intent

Para:

```text
SERVICE_AUTHENTICATION
```

pueden evaluarse:

```text
mTLS
service token
workload identity
```

---

## 103. Transport filtering

Antes de `supports()` se excluirán Authenticators incompatibles con el transport.

Ejemplo:

```text
CLI
```

no necesita evaluar:

```text
browser session cookie
```

salvo configuración especial.

---

## 104. Tenant filtering

Un Authenticator podrá estar disponible solo para ciertos tenant profiles.

Ejemplo:

```text
Tenant A:
    password
    passkey

Tenant B:
    enterprise OIDC only
```

---

## 105. Tenant policy no debe crear resolución arbitraria

La configuración efectiva deberá compilarse o resolverse mediante perfiles conocidos.

Evitar:

```text
arbitrary tenant-controlled class name
```

---

## 106. Provider-aware filtering

Un Authenticator podrá requerir determinado provider capability.

Ejemplo:

```text
PasswordAuthenticator
```

requiere provider capaz de resolver password-authenticable identities.

---

## 107. Provider filtering no verifica credenciales

Solo determina compatibilidad estructural/configuracional.

---

## 108. Route-aware selection

Rutas especializadas pueden limitar mecanismos.

Ejemplo:

```text
/login/password
    → password

/login/passkey
    → passkey

/login/sso
    → oidc
```

Aquí la selección puede ser casi directa.

---

## 109. Generic login endpoint

También podrá existir:

```text
/login
```

que soporte varios mecanismos.

En ese caso deberá existir un discriminator explícito o payload inequívoco.

---

## 110. Authentication method discriminator

Ejemplo:

```json
{
    "method": "password",
    "email": "...",
    "password": "..."
}
```

El campo `method` podrá actuar como hint fuerte.

---

## 111. Discriminator validation

El cliente no podrá seleccionar un Authenticator que el Firewall no permita.

```text
method = internal_super_admin_auth
```

no habilita nada por sí mismo.

---

## 112. Discriminator conflict

Si:

```text
method = password
```

pero el payload contiene únicamente una passkey assertion:

```text
MALFORMED / CONFLICT
```

---

## 113. Header scheme selection

`Authorization` puede funcionar como discriminator:

```text
Bearer
Basic
DPoP
Custom
```

pero solo schemes configurados deberán aceptarse.

---

## 114. Unsupported explicit scheme

Ejemplo:

```text
Authorization: Basic ...
```

en un Firewall que solo admite Bearer.

Deberá producir:

```text
UNSUPPORTED_AUTHENTICATION_SCHEME
```

no ignorarse como si no hubiera credential.

---

## 115. Why unsupported explicit credentials matter

Ignorarlo podría permitir:

```text
unsupported explicit credential
    ↓
session fallback
```

creando comportamiento inesperado.

---

## 116. Unknown Authorization scheme

Deberá existir policy:

```text
REJECT
IGNORE_ON_PUBLIC_ROUTE
```

según contexto.

Para rutas protegidas, la opción segura por defecto será `REJECT`.

---

## 117. Public routes

Una ruta pública puede recibir una credencial inválida.

VoltStack deberá definir si:

```text
ignore invalid optional authentication
```

o:

```text
reject malformed credential
```

---

## 118. Recomendación para malformed explicit credential

Incluso en rutas públicas, si el cliente intenta autenticarse explícitamente con una credencial malformada, es preferible no tratarla silenciosamente como autenticación ausente en APIs sensibles.

---

## 119. Optional Authentication Policy

Podrá existir:

```text
OptionalAuthenticationFailurePolicy
```

con:

```text
IGNORE_ABSENT
REJECT_MALFORMED
REJECT_INVALID
ALLOW_GUEST_ON_INVALID
```

La última deberá utilizarse con cautela.

---

## 120. Browser UX distinction

Una página pública web puede permitir:

```text
expired session
    ↓
guest page
```

mientras una API con bearer inválido debería responder:

```text
authentication failure
```

Por tanto, la policy depende del mecanismo y transport.

---

## 121. Expired session is not identical to invalid explicit token

La resolución deberá preservar esta distinción.

---

## 122. Remember-me precedence

Configuración típica:

```text
explicit login
    >
session
    >
remember_me
```

---

## 123. Session + remember-me

Si existe session válida, no es necesario intentar remember-me.

---

## 124. Invalid session + remember-me

La policy deberá decidir explícitamente si remember-me puede recuperar autenticación después de session expirada/inválida.

No asumirlo automáticamente.

---

## 125. Session invalidation semantics

Casos como:

```text
session revoked
```

pueden requerir que remember-me también sea invalidado.

El Resolver deberá respetar el resultado del Session Security subsystem.

---

## 126. Selection policy phases

Se recomienda dividir:

```text
Phase 1 — Eligibility
Phase 2 — Support
Phase 3 — Structural validity
Phase 4 — Conflict
Phase 5 — Precedence
Phase 6 — Selection
Phase 7 — Lock
```

---

## 127. Phase 1 — Eligibility

Filtra por:

```text
firewall
transport
intent
tenant profile
protocol transaction
capabilities
```

---

## 128. Phase 2 — Support

Ejecuta:

```text
supports()
```

solo sobre candidatos elegibles.

---

## 129. Phase 3 — Structural validity

Cualquier:

```text
MALFORMED
```

relevante deberá procesarse antes de fallback.

---

## 130. Phase 4 — Conflict

Detecta:

```text
multiple credentials
channel conflicts
transaction mismatch
duplicate mechanisms
```

---

## 131. Phase 5 — Precedence

Aplica:

```text
credential category
configured order
priority
specificity
```

---

## 132. Phase 6 — Selection

Produce:

```text
AuthenticatorSelectionPlan
```

---

## 133. Phase 7 — Lock

Fija la selección dentro de la operación.

---

## 134. Deterministic tie-breaking

La resolución final deberá ser reproducible con:

```text
same config
same request
same operation
    ↓
same selection
```

---

## 135. Declaration order

Podrá usarse como último tie-breaker solo cuando el usuario haya definido explícitamente que el orden de la lista representa precedencia.

Ejemplo:

```php
'precedence' => [
    'bearer',
    'api_key',
];
```

No deberá inferirse del orden accidental del Container.

---

## 136. Priority ranges

No será necesario imponer semántica rígida, pero podría recomendarse:

```text
300+ explicit infrastructure
200  explicit request credentials
100  session
50   remember-me
```

solo como convención.

---

## 137. Prefer symbolic precedence

Para configuración pública puede ser más claro:

```php
'precedence' => [
    'client_certificate',
    'bearer',
    'session',
],
```

que números mágicos.

---

## 138. Internal compiled priority

La configuración simbólica podrá compilarse a números internos.

---

## 139. Policy presets

VoltStack podrá proporcionar:

```text
strict
browser
api
hybrid
service
```

---

## 140. Strict preset

Podría significar:

```text
reject multiple credentials
reject ambiguity
reject malformed
no implicit fallback
```

---

## 141. Browser preset

Podría significar:

```text
explicit login > session > remember-me
expired optional session may become guest
malformed explicit login rejects
```

---

## 142. API preset

Podría significar:

```text
explicit credentials only
reject multiple explicit credentials
no session fallback unless hybrid configured
```

---

## 143. Hybrid preset

Podría significar:

```text
explicit bearer > session
invalid bearer does not fallback
```

---

## 144. Service preset

Podría significar:

```text
configured machine credentials
optionally ALL_REQUIRED
strict identity compatibility
```

---

## 145. Custom policy

Aplicaciones podrán registrar:

```php
Auth::selectionPolicy(
    'custom',
    CustomAuthenticatorSelectionPolicy::class
);
```

---

## 146. Custom policies and security

Las custom policies deberán operar sobre descriptors/resultados estructurados.

No deberían recibir secretos innecesariamente.

---

## 147. Policy purity

Idealmente:

```text
selection policy
```

será determinista y side-effect free.

---

## 148. No remote calls in selection policy

No deberá:

```text
query database
call IdP
verify token
consume OTP
```

---

## 149. Selection before expensive verification

Una ventaja importante:

```text
resolve one mechanism first
    ↓
perform expensive verification only when needed
```

---

## 150. DoS considerations

Un atacante no debería poder provocar verificaciones criptográficas costosas para todos los Authenticators simultáneamente si la policy solo necesita uno.

---

## 151. Candidate budget

Podrá existir un límite:

```text
max_authenticator_candidates
```

como protección contra configuraciones/plugins excesivos.

---

## 152. supports() budget

El Resolver podrá instrumentar:

```text
support evaluation count
duration
```

para detectar Authenticators costosos.

---

## 153. Slow supports() warning

En desarrollo:

```text
Authenticator X supports() took 50ms
```

podrá generar warning porque `supports()` debería ser barato.

---

## 154. Resolution caching

Dentro de una operación, el resultado deberá memoizarse.

---

## 155. Request memoization

Para recovery:

```text
Auth::check()
Auth::identity()
Auth::context()
```

no deberán resolver Authenticators tres veces.

---

## 156. Cache key

La memoización podrá depender de:

```text
operation id
firewall
intent
transaction
```

---

## 157. No unsafe cross-request cache

No se almacenará:

```text
last selected authenticator
```

globalmente en el worker.

---

## 158. FrankenPHP safety

El Resolver podrá ser singleton si contiene únicamente:

```text
compiled immutable indexes
policy registry
service references
```

y todo estado de resolución vive en scope/operation.

---

## 159. Concurrent fibers

Dos requests concurrentes:

```text
Fiber A → bearer
Fiber B → session
```

deberán mantener locks y resolution results independientes.

---

## 160. Resolver observability

Span recomendado:

```text
auth.authenticator.resolve
```

---

## 161. Trace attributes

Seguros:

```text
firewall.name
candidate.count
selected.authenticator
selection.policy
credential.category
resolution.status
```

---

## 162. No secret trace attributes

Nunca:

```text
bearer token
password
API key
OIDC code
SAML assertion
```

---

## 163. Metrics

Ejemplos:

```text
authenticator_resolution_total
authenticator_resolution_conflict_total
authenticator_resolution_ambiguous_total
authenticator_selection_total
authenticator_candidate_count
authenticator_resolution_duration
```

---

## 164. Low-cardinality labels

Permitidos:

```text
firewall
authenticator
status
policy
```

si el conjunto está controlado.

---

## 165. Audit events

Podrán auditarse:

```text
multiple credential conflict
credential downgrade prevented
identity conflict
transaction authenticator mismatch
unexpected authentication scheme
```

---

## 166. Explainability

En modo desarrollo:

```text
Request authentication resolution

Firewall:
    api

Eligible:
    bearer
    api_key

Detected:
    Authorization: Bearer → bearer
    X-API-Key → absent

Support:
    bearer → SUPPORTED

Policy:
    api-strict

Selected:
    bearer

Reason:
    only supported explicit credential
```

---

## 167. Conflict explainability

Ejemplo:

```text
Detected:
    bearer → present
    session → present

Policy:
    reject_multiple

Result:
    CONFLICT

Reason:
    multiple credential channels prohibited
```

---

## 168. Redaction in debugger

El debugger podrá mostrar:

```text
Authorization: Bearer [REDACTED]
```

Nunca el valor.

---

## 169. Compile-time validation

El compiler deberá detectar:

```text
duplicate authenticator names
unknown firewall authenticator
duplicate precedence entries
impossible priorities
unsupported policy
ambiguous trigger mappings
invalid combination modes
```

---

## 170. Missing precedence

Si un Firewall permite mecanismos potencialmente conflictivos:

```text
session
bearer
```

y no existe policy explícita, VoltStack podrá aplicar un default seguro o exigir configuración.

---

## 171. Strict configuration mode

Podrá existir:

```text
auth.strict_configuration = true
```

que obligue a declarar reglas para combinaciones ambiguas.

---

## 172. Configuration example — Web

```php
'web' => [
    'authenticators' => [
        'password',
        'passkey',
        'session',
        'remember_me',
    ],

    'selection' => [
        'policy' => 'browser',
        'precedence' => [
            'passkey',
            'password',
            'session',
            'remember_me',
        ],
    ],
];
```

---

## 173. Configuration example — API

```php
'api' => [
    'authenticators' => [
        'bearer_token',
        'api_key',
    ],

    'selection' => [
        'policy' => 'strict',
        'multiple_credentials' => 'reject',
    ],
];
```

---

## 174. Configuration example — Hybrid SPA

```php
'spa' => [
    'authenticators' => [
        'bearer_token',
        'session',
    ],

    'selection' => [
        'policy' => 'hybrid',
        'precedence' => [
            'bearer_token',
            'session',
        ],
        'fallback_after_failure' => false,
    ],
];
```

---

## 175. Configuration example — Service

```php
'internal_service' => [
    'authenticators' => [
        'client_certificate',
        'service_token',
    ],

    'selection' => [
        'combination' => 'all_required',
        'identity_policy' => 'same_identity',
    ],
];
```

---

## 176. Example — password vs session

Request:

```text
POST /login
valid existing session
password credentials supplied
```

Browser policy:

```text
password = explicit
session = implicit
```

Resultado:

```text
PasswordAuthenticator selected
```

porque el usuario está iniciando explícitamente una nueva autenticación.

---

## 177. Example — invalid password + valid session

En `/login`:

```text
password selected
verification fails
```

El sistema no deberá convertir automáticamente la operación en:

```text
session authentication success
```

El login explícito falló.

La sesión existente puede seguir existiendo como contexto separado dependiendo de la aplicación, pero no convierte el intento fallido en éxito.

---

## 178. Example — API key + bearer

API strict:

```text
Bearer present
API key present
```

Resultado:

```text
CONFLICT
```

si `REJECT_MULTIPLE`.

---

## 179. Example — bearer + session

Hybrid:

```text
bearer present
session present
precedence = bearer > session
```

Resultado:

```text
Bearer selected
```

---

## 180. Example — invalid bearer + session

```text
Bearer selected
    ↓
verification failed
```

Resultado:

```text
AUTHENTICATION_FAILED
```

No:

```text
try session
```

---

## 181. Example — no bearer + valid session

Hybrid:

```text
bearer → NOT_SUPPORTED
session → SUPPORTED
```

Resultado:

```text
SessionAuthenticator selected
```

Esto sí es fallback válido porque bearer nunca fue intentado.

---

## 182. Diferencia fundamental

```text
NOT_SUPPORTED
    → another authenticator may be selected

SELECTED + INVALID CREDENTIAL
    → authentication attempt ends
```

Esta será una de las invariantes más importantes del sistema.

---

## 183. Example — malformed bearer

```text
Authorization: Bearer
```

Resultado:

```text
MALFORMED
```

No deberá probar session aunque exista.

---

## 184. Example — unsupported scheme

```text
Authorization: Basic ...
```

API Bearer-only:

```text
UNSUPPORTED_AUTHENTICATION_SCHEME
```

---

## 185. Example — OIDC transaction

Transaction:

```text
authenticator = oidc
provider = microsoft
```

Callback:

```text
resolver sees transaction lock
```

Resultado:

```text
OIDC Microsoft selected directly
```

sin reevaluar password/session.

---

## 186. Example — wrong protocol callback

Transaction:

```text
passkey
```

Request parece:

```text
OIDC callback
```

Resultado:

```text
TRANSACTION_AUTHENTICATOR_MISMATCH
```

---

## 187. Example — service dual credential

```text
mTLS certificate → Service A
token            → Service A
```

Policy:

```text
ALL_REQUIRED
SAME_IDENTITY
```

Resultado:

```text
success candidate plan
```

---

## 188. Example — service identity conflict

```text
mTLS → Service A
token → Service B
```

Resultado posterior:

```text
IDENTITY_CONFLICT
```

---

## 189. Error taxonomy

Se recomienda:

```text
AuthenticatorResolutionException
AuthenticatorAmbiguityException
AuthenticatorConflictException
UnsupportedAuthenticationSchemeException
MalformedAuthenticationInputException
AuthenticatorSelectionLockedException
AuthenticationTransactionMismatchException
AuthenticationIdentityConflictException
```

---

## 190. Expected failures vs exceptions

Resultados esperados:

```text
NONE
MALFORMED
CONFLICT
AMBIGUOUS
UNSUPPORTED
```

pueden representarse mediante value objects.

Exceptions se reservarán para:

```text
invalid configuration
programming errors
unexpected infrastructure failures
```

---

## 191. Integration with Authentication Manager

El Manager realizará:

```text
resolve firewall
    ↓
resolve authenticator
    ↓
lock selection
    ↓
create passport
    ↓
process passport
```

---

## 192. Manager must not bypass Resolver

No deberá existir:

```php
$authenticator = $container->get('bearer');
$authenticator->createPassport(...);
```

en flows normales internos ignorando la policy.

---

## 193. Explicit programmatic authentication

APIs avanzadas podrán seleccionar un Authenticator explícitamente:

```php
Auth::using('passkey')->authenticate(...);
```

pero deberá validarse:

```text
allowed by firewall
allowed by operation
compatible with transport
```

---

## 194. Explicit selection as highest precedence

Si la aplicación inicia deliberadamente:

```text
using('passkey')
```

esa selección podrá convertirse en:

```text
EXPLICIT_SELECTION
```

y bloquear el Resolver a ese mecanismo.

---

## 195. Explicit selection cannot bypass security

No podrá habilitar un Authenticator desactivado.

---

## 196. Route-bound Authenticator

Una ruta podrá indicar:

```text
authentication.authenticator = password
```

para endpoints especializados.

Esto actúa como selection hint/constraint.

---

## 197. Route constraint mismatch

Si la ruta exige:

```text
password
```

pero recibe passkey:

```text
UNSUPPORTED_FOR_ROUTE
```

en vez de resolver passkey.

---

## 198. Authentication transaction has precedence over route hint

Para continuations, la transaction original deberá prevalecer para evitar protocol confusion, siempre que la ruta sea compatible.

---

## 199. Security invariants

### AUTH-RES-01

Solo Authenticators permitidos por el Firewall pueden ser seleccionados.

#### AUTH-RES-02

La selección debe ser determinista.

#### AUTH-RES-03

El orden accidental de servicios no determina precedencia.

#### AUTH-RES-04

`NOT_SUPPORTED` permite continuar; credential failure no.

#### AUTH-RES-05

Input malformado no debe convertirse en fallback silencioso.

#### AUTH-RES-06

Una credencial explícita seleccionada bloquea fallback automático.

#### AUTH-RES-07

Múltiples credenciales deben someterse a una policy explícita.

#### AUTH-RES-08

Ambigüedad crítica falla de forma cerrada.

#### AUTH-RES-09

Selection Priority no equivale a Authentication Assurance.

#### AUTH-RES-10

La selección no ejecuta Authorization.

#### AUTH-RES-11

Protocol continuations permanecen ligados al Authenticator original.

#### AUTH-RES-12

Identidades de múltiples evidencias no se fusionan silenciosamente.

#### AUTH-RES-13

El estado de resolución pertenece al scope de operación.

#### AUTH-RES-14

No existe fallback global fuera del Firewall.

#### AUTH-RES-15

El Resolver no verifica secretos.

#### AUTH-RES-16

La selección puede compilarse y explicarse.

---

## 200. Anti-pattern — first supports wins

Prohibido:

```php
foreach ($authenticators as $authenticator) {
    if ($authenticator->supports($request)) {
        return $authenticator;
    }
}
```

sin política de conflicto y precedencia.

---

## 201. Anti-pattern — catch and try next

Prohibido:

```php
foreach ($authenticators as $authenticator) {
    try {
        return $authenticator->authenticate($request);
    } catch (AuthenticationFailed $e) {
        continue;
    }
}
```

Esto convierte credenciales inválidas en fallback.

---

## 202. Anti-pattern — priority as assurance

Prohibido:

```text
higher priority
=
more secure authentication
```

---

## 203. Anti-pattern — hidden precedence

Evitar reglas internas que el desarrollador no pueda inspeccionar.

---

## 204. Anti-pattern — multi-credential merge

Prohibido:

```text
session says Alice
token says Bob

result:
    Alice/Bob combined principal
```

---

## 205. Anti-pattern — arbitrary tenant selection

Un tenant no deberá poder proporcionar directamente:

```text
authenticator class
```

desde input no confiable.

---

## 206. Anti-pattern — repeated resolution

No volver a ejecutar Resolver después de un verification failure para buscar otro mecanismo.

---

## 207. Anti-pattern — transaction switching

Un callback no podrá cambiar de OIDC a SAML o Passkey porque otro Authenticator también haga `supports()`.

---

## 208. Estructura sugerida

```text
src/Quantum/Auth/
└── Authenticator/
    └── Resolution/
        ├── AuthenticatorResolverInterface.php
        ├── AuthenticatorResolver.php
        ├── AuthenticatorResolutionRequest.php
        ├── AuthenticatorResolution.php
        ├── AuthenticatorResolutionStatus.php
        ├── AuthenticatorResolutionReason.php
        ├── AuthenticatorCandidateSet.php
        ├── AuthenticatorCandidateDiscovery.php
        ├── AuthenticatorSelectionPlan.php
        ├── AuthenticatorSelectionLock.php
        │
        ├── Policy/
        │   ├── AuthenticatorSelectionPolicyInterface.php
        │   ├── StrictSelectionPolicy.php
        │   ├── BrowserSelectionPolicy.php
        │   ├── ApiSelectionPolicy.php
        │   ├── HybridSelectionPolicy.php
        │   └── ServiceSelectionPolicy.php
        │
        ├── Conflict/
        │   ├── AuthenticatorConflictDetector.php
        │   ├── CredentialConflict.php
        │   └── IdentityConflictDetector.php
        │
        └── Compilation/
            ├── AuthenticatorResolutionCompiler.php
            ├── CompiledCandidateIndex.php
            └── CompiledSelectionPolicy.php
```

---

## 209. Arquitectura final

```text
                    AuthenticationRequest
                             │
                             ▼
                    Resolved Firewall
                             │
                             ▼
                 Allowed Authenticators
                             │
                             ▼
                   Candidate Discovery
                             │
                             ▼
                    Eligibility Filter
                             │
                             ▼
                    supports() Results
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        MALFORMED        CONFLICT       SUPPORTED
              │              │              │
              ▼              ▼              ▼
            Reject          Reject      Selection Policy
                                              │
                                  ┌───────────┼───────────┐
                                  ▼           ▼           ▼
                             precedence    priority   specificity
                                  │           │           │
                                  └───────────┼───────────┘
                                              ▼
                                      Selection Plan
                                              │
                                              ▼
                                       Selection Lock
                                              │
                                              ▼
                                        Authenticator
                                              │
                                              ▼
                                    AuthenticationPassport
```

---

## 210. Criterios de aceptación

El sistema será considerado completo cuando:

1. resuelva únicamente Authenticators permitidos;
2. soporte candidate discovery;
3. soporte compiled hints;
4. distinga `SUPPORTED` y `NOT_SUPPORTED`;
5. distinga `MALFORMED`;
6. detecte ambigüedad;
7. detecte conflictos;
8. distinga credenciales explícitas e implícitas;
9. soporte precedence configurable;
10. soporte priority;
11. soporte specificity;
12. soporte múltiples credenciales;
13. soporte políticas estrictas;
14. soporte browser policies;
15. soporte API policies;
16. soporte hybrid policies;
17. soporte service policies;
18. impida credential downgrade;
19. implemente selection locking;
20. soporte protocol continuation locking;
21. soporte múltiples Authenticators requeridos;
22. detecte identity conflicts;
23. sea tenant-aware;
24. sea intent-aware;
25. sea transport-aware;
26. permita compilación;
27. permita memoization;
28. sea seguro con FrankenPHP;
29. tenga observabilidad;
30. tenga explainability;
31. sea extensible;
32. no verifique credenciales;
33. no calcule Authorization;
34. no calcule Assurance;
35. no dependa del orden accidental de registro.

---

## 211. Regla arquitectónica final

La resolución deberá preservar siempre:

```text
DETECT
  ↓
CLASSIFY
  ↓
VALIDATE STRUCTURE
  ↓
DETECT CONFLICT
  ↓
SELECT
  ↓
LOCK
  ↓
VERIFY
```

y nunca:

```text
TRY
 ↓
FAIL
 ↓
TRY SOMETHING WEAKER
 ↓
SUCCESS
```

La distinción crítica será:

```text
NOT_SUPPORTED
        ≠
AUTHENTICATION_FAILED
```

Por tanto:

> **Un Authenticator que no aplica permite buscar otro mecanismo.**
> **Un Authenticator seleccionado cuya credencial falla termina esa tentativa de autenticación.**

Esta regla impedirá que la flexibilidad multi-Authenticator de VoltStack se convierta en una vía accidental de degradación de seguridad.

---

## 212. Próximo documento

El siguiente documento recomendado será:

```text
08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md
```

Este documento deberá definir la capa central que conecta los Authenticators con la verificación real:

```text
AuthenticationPassport
IdentityClaim
Credential
CredentialSet
SecretCredential
PasswordCredential
TokenCredential
ApiKeyCredential
PasskeyAssertionCredential
SignedAssertionCredential
SessionReferenceCredential
CredentialVerifier
CredentialVerifierRegistry
VerifiedCredential
AuthenticationEvidence
EvidenceSet
credential lifecycle
credential redaction
credential-derived identity
verification ordering
verification result
evidence composition
security boundaries
secret handling
persistent-runtime safety
extension model
```

Con ese documento quedará formalizado el punto exacto donde los datos no confiables producidos por los Authenticators comienzan a convertirse en **evidencia de autenticación verificada**.
