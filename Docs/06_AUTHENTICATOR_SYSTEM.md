# VoltStack Authentication System

## 06 — Authenticator System

- **Archivo:** `06_AUTHENTICATOR_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del sistema de Authenticators  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`

---

## 1. Propósito

Este documento define el modelo de **Authenticators** de VoltStack.

Un Authenticator será el componente encargado de:

> **Reconocer un mecanismo de autenticación, extraer y normalizar la evidencia presentada por el cliente, y construir un `AuthenticationPassport` que pueda ser procesado por el Authentication System.**

El Authenticator no será responsable por sí solo de completar todo el proceso de autenticación.

Su función principal será actuar como puente entre:

```text
transport / protocol input
        ↓
authentication-domain input
```

La arquitectura deberá permitir incorporar nuevos mecanismos sin modificar el Core.

---

## 2. Objetivo arquitectónico

VoltStack deberá soportar Authenticators para:

```text
Password
Session
Bearer Token
API Key
JWT
Passkey / WebAuthn
Magic Link
Email OTP
OAuth / OIDC
SAML
LDAP-backed flows
Client Certificate
mTLS
Service Account
Workload Identity
Signed Assertion
Recovery Credential
Custom Protocols
```

sin crear caminos de autenticación paralelos.

Todos deberán terminar produciendo:

```text
AuthenticationPassport
```

o una estructura equivalente compatible con el pipeline central.

---

## 3. Principio fundamental

La regla será:

> **Authenticator interpreta; Verifier verifica.**

Por tanto:

```text
Authenticator
    detects mechanism
    extracts data
    validates basic structure
    creates Passport

CredentialVerifier
    verifies cryptographic/authentication proof

IdentityProvider
    resolves Identity

Policy Engine
    determines requirements

Decision Engine
    decides outcome
```

---

## 4. Authenticator no es Guard

Un `Guard` representa una API de acceso a Authentication.

Un `Authenticator` representa un mecanismo concreto para iniciar o recuperar autenticación.

Ejemplo:

```text
Guard:
    web

Authenticators:
    session
    password
    passkey
```

---

## 5. Authenticator no es Firewall

El Firewall responde:

> ¿Qué configuración debe aplicarse?

El Authenticator responde:

> ¿Qué mecanismo está intentando utilizar esta request?

Relación:

```text
AuthenticationRequest
        ↓
Firewall
        ↓
allowed authenticators
        ↓
Authenticator Resolver
        ↓
Authenticator
```

---

## 6. Authenticator no es CredentialVerifier

Ejemplo password:

```text
PasswordAuthenticator
    extracts:
        email
        password

PasswordCredentialVerifier
    verifies:
        password hash
```

Ejemplo bearer:

```text
BearerTokenAuthenticator
    extracts:
        raw bearer token

TokenVerifier
    verifies:
        signature
        issuer
        audience
        expiration
        revocation
```

---

## 7. AuthenticatorInterface

Contrato conceptual:

```php
interface AuthenticatorInterface
{
    public function supports(
        AuthenticationRequest $request
    ): bool;

    public function authenticate(
        AuthenticationRequest $request
    ): AuthenticationPassport;
}
```

Sin embargo, VoltStack podrá separar todavía más:

```php
interface AuthenticatorInterface
{
    public function supports(
        AuthenticationRequest $request
    ): AuthenticatorSupportResult;

    public function createPassport(
        AuthenticationRequest $request
    ): AuthenticationPassport;
}
```

Esta segunda forma es preferible porque evita que `authenticate()` sugiera que el Authenticator completa todo el proceso.

---

## 8. Nombre recomendado del método

La API interna debería preferir:

```php
createPassport()
```

sobre:

```php
authenticate()
```

porque el Authenticator no autentica completamente.

El término `authenticate()` podrá reservarse para APIs públicas del Manager.

---

## 9. AuthenticatorSupportResult

En lugar de limitarse a `bool`, podría existir:

```text
SUPPORTED
NOT_SUPPORTED
MALFORMED
AMBIGUOUS
ERROR
```

Esto permite diferenciar:

```text
no bearer token present
```

de:

```text
Authorization header claims Bearer but is malformed
```

---

## 10. supports() debe ser barato

El método `supports()` deberá ser:

```text
fast
side-effect free
non-destructive
non-blocking where possible
```

No deberá:

```text
query database
verify password
call remote IdP
consume OTP
validate signature
create challenge
```

---

## 11. supports() examples

Password:

```text
POST /login
+
expected login payload
    → SUPPORTED
```

Bearer:

```text
Authorization: Bearer ...
    → SUPPORTED
```

Session:

```text
session cookie present
    → SUPPORTED
```

Passkey:

```text
passkey assertion payload
    → SUPPORTED
```

---

## 12. Malformed input

Ejemplo:

```text
Authorization: Bearer
```

podrá producir:

```text
MALFORMED
```

en lugar de:

```text
NOT_SUPPORTED
```

para evitar que otro Authenticator interprete accidentalmente la misma entrada.

---

## 13. AuthenticatorDescriptor

Cada Authenticator deberá tener metadata independiente de la instancia.

Ejemplo:

```text
AuthenticatorDescriptor
│
├── name
├── service id
├── authentication method
├── priority
├── supported transports
├── interactive
├── stateful capability
├── challenge capability
├── protocol
└── security metadata
```

---

## 14. Descriptor benefits

Permite:

```text
lazy instantiation
compiled resolution
configuration validation
debugging
capability discovery
```

sin crear todos los Authenticators en cada request.

---

## 15. AuthenticatorRegistry

VoltStack tendrá un:

```text
AuthenticatorRegistry
```

que administre Authenticators registrados.

Conceptualmente:

```php
interface AuthenticatorRegistryInterface
{
    public function get(string $name): AuthenticatorDescriptor;

    public function has(string $name): bool;

    public function all(): iterable;
}
```

---

## 16. Registry lifecycle

Durante bootstrap:

```text
registration
    ↓
validation
    ↓
compilation
    ↓
freeze
```

En producción no deberá cambiar durante requests normales.

---

## 17. Authenticator registration

Podrá soportar:

```php
Auth::extendAuthenticator(
    'custom_token',
    CustomTokenAuthenticator::class
);
```

o registration mediante service provider.

La API definitiva deberá integrarse con el Container.

---

## 18. Named Authenticators

Cada mecanismo tendrá nombre estable.

Ejemplos:

```text
session
password
bearer_token
api_key
passkey
magic_link
oidc
saml
service_account
client_certificate
```

---

## 19. AuthenticationMethod mapping

Cada Authenticator deberá declarar el `AuthenticationMethod` que produce principalmente.

Ejemplo:

```text
PasswordAuthenticator
    → PASSWORD

PasskeyAuthenticator
    → PASSKEY

OidcAuthenticator
    → OIDC
```

---

## 20. Interactive vs Non-Interactive

VoltStack distinguirá:

```text
InteractiveAuthenticator
NonInteractiveAuthenticator
```

---

## 21. Interactive Authenticator

Requiere participación del usuario.

Ejemplos:

```text
Password
Passkey
TOTP flow
Magic Link
OIDC redirect login
SAML SSO
```

Puede producir:

```text
challenge
redirect
continuation transaction
```

---

## 22. Non-Interactive Authenticator

Puede procesarse completamente en una request.

Ejemplos:

```text
Bearer Token
API Key
Client Certificate
Service Credential
Session Recovery
```

---

## 23. Challenge-capable Authenticator

Algunos protocolos necesitan challenge inicial.

Ejemplo:

```text
Passkey
OIDC
SAML
Magic Link
```

Podrán implementar:

```php
interface ChallengeCapableAuthenticatorInterface
{
    public function begin(
        AuthenticationInitiationContext $context
    ): AuthenticationChallengeResult;
}
```

---

## 24. Inicio vs finalización

Para flujos multi-step conviene separar:

```text
begin authentication
continue authentication
```

Ejemplo Passkey:

```text
begin
    ↓
challenge generated
    ↓
client response
    ↓
continue
```

---

## 25. ProtocolAuthenticatorInterface

Authenticators de protocolos complejos podrán implementar contratos adicionales:

```php
interface ProtocolAuthenticatorInterface
{
    public function protocol(): AuthenticationProtocol;
}
```

Ejemplos:

```text
OIDC
SAML
WebAuthn
```

---

## 26. Stateful Authenticator

Un Authenticator puede depender de estado previo.

Ejemplo:

```text
SessionAuthenticator
```

pero el estado deberá estar encapsulado mediante servicios específicos.

No deberá almacenar estado mutable directamente en la instancia.

---

## 27. Stateless Authenticator

Ejemplo:

```text
BearerTokenAuthenticator
ApiKeyAuthenticator
```

deberá poder funcionar únicamente con la request actual y servicios externos requeridos.

---

## 28. SessionAuthenticator

Responsabilidad:

```text
detect session authentication state
extract session reference
create Passport / recovery request
```

No deberá:

```text
own session storage
decide permissions
```

La reconstrucción final puede delegarse al Recovery Manager.

---

## 29. PasswordAuthenticator

Responsabilidad:

```text
recognize login payload
extract identity claim
extract password credential
normalize input
construct Passport
```

---

## 30. PasswordAuthenticator example

```php
final class PasswordAuthenticator implements AuthenticatorInterface
{
    public function supports(
        AuthenticationRequest $request
    ): AuthenticatorSupportResult {
        // inspect request shape
    }

    public function createPassport(
        AuthenticationRequest $request
    ): AuthenticationPassport {
        return new AuthenticationPassport(
            identityClaim: new IdentityClaim(
                'email',
                $request->input('email')
            ),
            method: AuthenticationMethod::password(),
            credentials: [
                new PasswordCredential(
                    $request->input('password')
                ),
            ],
        );
    }
}
```

---

## 31. PasswordAuthenticator no verifica hash

Prohibido:

```php
if (password_verify(...)) {
    return $user;
}
```

dentro del Authenticator.

El flujo correcto:

```text
Authenticator
    ↓
PasswordCredential
    ↓
CredentialVerifier
```

---

## 32. BearerTokenAuthenticator

Responsabilidad:

```text
detect Authorization header
parse scheme
extract raw token
validate structural limits
construct token credential
```

---

## 33. Bearer token parsing

Deberá validar:

```text
scheme
presence
length limits
encoding constraints
forbidden whitespace
multiple authorization headers
```

antes de crear Credential.

---

## 34. Multiple Authorization headers

Si aparecen múltiples credenciales:

```text
Authorization: Bearer tokenA
Authorization: Bearer tokenB
```

podrá considerarse:

```text
AMBIGUOUS
```

o inválido.

Nunca escoger uno arbitrariamente.

---

## 35. APIKeyAuthenticator

Podrá extraer API keys desde:

```text
Authorization header
dedicated header
signed request
```

según configuración.

Deberá evitar múltiples ubicaciones ambiguas por defecto.

---

## 36. API key location policy

Ejemplo:

```text
X-API-Key
```

puede habilitarse.

Pero permitir simultáneamente:

```text
query string
header
cookie
```

incrementa ambigüedad y exposición.

VoltStack deberá favorecer ubicaciones seguras.

---

## 37. Query-string credentials

Las credenciales en URL deberán desaconsejarse fuertemente porque pueden filtrarse en:

```text
logs
browser history
proxies
analytics
referrers
```

---

## 38. JwtAuthenticator

VoltStack podrá implementar JWT como tipo de token, no como modelo completo de Authentication.

El Authenticator:

```text
extract JWT
create token credential
```

El TokenVerifier:

```text
verify JWT
```

---

## 39. PasskeyAuthenticator

Responsabilidad:

```text
recognize WebAuthn assertion
parse credential response
bind transaction/challenge reference
construct PasskeyAssertionCredential
```

No deberá verificar directamente la firma salvo que la implementación delegada encapsule correctamente esa responsabilidad.

---

## 40. WebAuthn structural validation

Antes del verifier podrá comprobar:

```text
required fields present
size limits
credential id format
client data encoding
transaction reference
```

---

## 41. MagicLinkAuthenticator

Podrá procesar:

```text
one-time login token
```

desde una URL segura.

Debe producir:

```text
MagicLinkCredential
```

y asociar:

```text
transaction
nonce
purpose
```

---

## 42. Magic link consumption

El Authenticator no deberá marcar el token como consumido simplemente al parsearlo.

El consumo deberá ocurrir dentro de una operación atómica de verificación/commit.

---

## 43. OidcAuthenticator

Un OIDC Authenticator podrá tener dos fases:

```text
begin()
callback()
```

---

## 44. OIDC begin

Responsabilidades:

```text
create authentication transaction
generate state
generate nonce
generate PKCE when required
construct authorization request
```

---

## 45. OIDC callback

Responsabilidades:

```text
recognize callback
extract code/state/error
restore transaction
construct protocol Passport
```

La verificación completa de tokens deberá delegarse a componentes OIDC especializados.

---

## 46. OAuth/OIDC separation

VoltStack deberá evitar confundir:

```text
OAuth authorization
```

con:

```text
OIDC authentication
```

Un OAuth access token por sí solo no siempre representa identidad autenticada.

---

## 47. SamlAuthenticator

Podrá:

```text
recognize SAML response
extract assertion envelope
restore request state
create SignedAssertionCredential
```

La validación criptográfica y semántica deberá realizarla un SAML verifier especializado.

---

## 48. ClientCertificateAuthenticator

Podrá obtener una referencia a certificado cliente desde un adapter confiable.

Importante:

```text
certificate metadata from trusted server/TLS layer
```

no de headers arbitrarios enviados por el cliente.

---

## 49. Proxy TLS termination

Si TLS termina en reverse proxy, headers de certificado solo podrán aceptarse si:

```text
trusted proxy configured
header stripping guaranteed
authenticated proxy channel
```

---

## 50. ServiceAccountAuthenticator

Podrá manejar:

```text
client id
client secret
signed service assertion
service token
```

y producir `ServiceIdentity` posteriormente.

---

## 51. WorkloadIdentityAuthenticator

Diseñado para:

```text
container
VM
cloud workload
internal runtime identity
```

Podrá integrarse con mecanismos externos sin crear usuarios ficticios.

---

## 52. RecoveryAuthenticator

Account recovery deberá usar Authenticators específicos.

Ejemplo:

```text
RecoveryCodeAuthenticator
RecoveryLinkAuthenticator
```

pero con policies más estrictas.

---

## 53. RememberMeAuthenticator

Podrá procesar credenciales persistentes.

Debe distinguir su provenance:

```text
REMEMBER_ME
```

de una sesión fresca.

---

## 54. Remember-me assurance

La autenticación recuperada mediante remember-me podrá recibir menor assurance.

Ejemplo:

```text
fresh password + MFA → AAL2

remember-me → AAL1
```

según policy.

---

## 55. AnonymousAuthenticator

VoltStack no necesitará un Authenticator anónimo por defecto.

La ausencia de autenticación se representará preferentemente como:

```text
no AuthenticationContext
```

---

## 56. PreAuthenticatedAuthenticator

Podrá existir para integraciones confiables donde otra infraestructura ya autenticó.

Ejemplos:

```text
trusted reverse proxy identity
enterprise gateway
internal signed delegation
```

pero deberá requerir una fuente de confianza explícita.

---

## 57. Pre-authentication risk

El patrón:

```text
X-User: admin
```

nunca deberá considerarse confiable por sí mismo.

Debe validarse:

```text
trusted proxy
channel
signature
network boundary
header stripping
```

---

## 58. Authenticator capability model

Cada descriptor podrá declarar capacidades:

```text
INTERACTIVE
NON_INTERACTIVE
STATEFUL
STATELESS
CHALLENGE_CAPABLE
PASSWORDLESS
MFA_CAPABLE
RECOVERY
FEDERATED
MACHINE
USER
TOKEN_BASED
```

---

## 59. Capabilities are metadata

Estas capabilities sirven para:

```text
configuration validation
selection
debugging
tooling
```

No reemplazan security policies.

---

## 60. Transport support

Un Authenticator podrá declarar:

```text
HTTP
SPA
CLI
WEBSOCKET
QUEUE
INTERNAL
```

---

## 61. Transport adapter boundary

Authenticators no deberían depender directamente de clases concretas HTTP cuando sea posible.

Preferir:

```text
AuthenticationRequest
```

normalizada.

---

## 62. HTTP-specific extension

Cuando sea necesario, un Authenticator podrá acceder a un adapter especializado, pero deberá mantenerse fuera del dominio central.

---

## 63. Credential extraction

La extracción deberá:

```text
validate size
validate multiplicity
normalize representation
preserve secrets minimally
```

---

## 64. Input size limits

Antes de operaciones costosas deberán existir límites.

Ejemplos:

```text
password max accepted transport size
token max size
SAML response max size
WebAuthn payload max size
certificate chain max size
```

Esto ayuda contra DoS.

---

## 65. No arbitrary serialization

El Authenticator no deberá deserializar objetos PHP proporcionados por el cliente.

Preferir:

```text
JSON
form data
protocol-specific safe parsing
```

con schemas definidos.

---

## 66. AuthenticationPassport creation

Un Authenticator exitoso debe crear un Passport estructurado.

Ejemplo:

```text
IdentityClaim
AuthenticationMethod
Credential[]
FactorInput[]
ProtocolState
Metadata
```

---

## 67. Passport metadata

Metadata segura puede incluir:

```text
authenticator name
protocol
client id
requested realm
transaction id
```

No:

```text
raw password
raw tokens duplicated
MFA secret
private key
```

---

## 68. Authenticator provenance

Cada Passport deberá registrar qué Authenticator lo creó.

Esto permite:

```text
audit
debugging
policy decisions
assurance calculation
```

---

## 69. AuthenticatorContext

Podrá existir un objeto interno:

```text
AuthenticatorContext
```

con:

```text
firewall
tenant
transport
transaction
route metadata
client metadata
```

para evitar que el Authenticator conozca todo el OperationContext.

---

## 70. Authenticator Context contract

Conceptualmente:

```php
final readonly class AuthenticatorContext
{
    public function __construct(
        public AuthenticationRequest $request,
        public ResolvedAuthenticationFirewall $firewall,
        public ?AuthenticationTransaction $transaction,
    ) {}
}
```

---

## 71. AuthenticatorResolver

Su responsabilidad será seleccionar Authenticator(s) permitidos por Firewall.

Input:

```text
AuthenticationRequest
Resolved Firewall
Authenticator Descriptors
```

Output:

```text
AuthenticatorResolution
```

---

## 72. Resolution algorithm

Conceptualmente:

```text
firewall allowed authenticators
        ↓
descriptor filtering
        ↓
supports() evaluation
        ↓
candidate set
        ↓
priority/conflict policy
        ↓
selected authenticator
```

---

## 73. No scanning all system authenticators

Si el sistema tiene 50 Authenticators pero el Firewall permite 3:

```text
only evaluate those 3
```

Esto mejora rendimiento y reduce superficie de ataque.

---

## 74. Priority

Cada Authenticator podrá tener:

```text
priority
```

pero no deberá ser la única regla de resolución.

---

## 75. Explicit credentials first

Una política razonable puede priorizar:

```text
explicit credential
    >
implicit persistent state
```

Ejemplo:

```text
Bearer token
    >
session cookie
```

en un Firewall hybrid.

---

## 76. However, precedence is configurable

Algunas aplicaciones podrían preferir:

```text
session
    >
bearer
```

La decisión deberá estar configurada en Firewall.

---

## 77. Authenticator conflict

Si:

```text
PasswordAuthenticator supports
MagicLinkAuthenticator supports
```

simultáneamente en una forma ambigua:

```text
CONFLICT
```

en vez de adivinar.

---

## 78. Credential identity conflict

Aunque se seleccione más de un mecanismo, si producen identidades diferentes:

```text
REJECT
```

salvo un flow formal de actor/subject/delegation.

---

## 79. Composite Authenticator

VoltStack deberá evitar Authenticators que mezclen varios mecanismos arbitrariamente.

En lugar de:

```text
PasswordAndTotpAuthenticator
```

preferir:

```text
PasswordAuthenticator
+
Factor Verification Pipeline
```

---

## 80. Exception — atomic protocols

Un protocolo puede contener internamente múltiples pruebas inseparables.

Ejemplo:

```text
WebAuthn assertion
```

podrá producir múltiples propiedades de assurance sin dividir artificialmente el Authenticator.

---

## 81. Multi-factor coordination

El Authenticator no decidirá:

```text
MFA required
```

Esto corresponde a Policy/Assurance.

---

## 82. Authenticator can indicate available evidence

Sí podrá crear:

```text
FactorInput
```

si la request ya contiene un segundo factor.

---

## 83. FactorInput

Ejemplo:

```text
TotpInput
RecoveryCodeInput
PasskeyInput
```

que luego será verificado.

---

## 84. Challenge initiation

Para mecanismos challenge-first:

```text
Passkey
OIDC
Magic Link
```

el Authenticator System necesitará un initiation service.

---

## 85. AuthenticationInitiator

Podrá existir:

```php
interface AuthenticationInitiatorInterface
{
    public function begin(
        AuthenticationInitiationRequest $request
    ): AuthenticationInitiationResult;
}
```

---

## 86. InitiationResult

Puede producir:

```text
ChallengeCreated
RedirectRequired
ClientActionRequired
Rejected
Error
```

---

## 87. Authenticator may implement initiation

Ejemplo:

```php
interface InitiatingAuthenticatorInterface
    extends AuthenticatorInterface
{
    public function begin(
        AuthenticationInitiationContext $context
    ): AuthenticationInitiationResult;
}
```

---

## 88. Protocol transaction binding

OIDC, WebAuthn y Magic Link deberán asociarse a:

```text
AuthenticationTransaction
```

cuando sean multi-step.

---

## 89. State parameter

Protocolos con `state` deberán:

```text
generate unpredictable value
bind to transaction
validate on return
consume/revoke when complete
```

---

## 90. Nonce

Protocol-specific nonce deberá tener lifecycle separado y no reutilizar identifiers predecibles.

---

## 91. PKCE

Cuando un protocolo lo requiera o recomiende, la arquitectura deberá permitir:

```text
code verifier
code challenge
```

sin almacenarlo en contextos públicos.

---

## 92. Redirect Authenticator

Podrá existir una capability:

```text
REDIRECT_BASED
```

para OIDC/SAML.

La generación del redirect deberá usar destinations configuradas y validadas.

---

## 93. Callback matching

Un callback Authenticator deberá reconocer:

```text
route
transaction
provider
protocol state
```

y no solo un query parameter.

---

## 94. Callback errors

Protocolos externos pueden devolver errores esperados:

```text
access_denied
login_required
consent_required
```

Estos deberán mapearse a resultados de Authentication tipados.

---

## 95. Authenticator errors

Taxonomía:

```text
AuthenticatorUnsupported
AuthenticatorMalformedInput
AuthenticatorConflict
AuthenticatorProtocolError
AuthenticatorConfigurationError
AuthenticatorInfrastructureError
```

---

## 96. Invalid credentials are not Authenticator errors

Ejemplo:

```text
wrong password
```

no es un error del PasswordAuthenticator.

Es un:

```text
CredentialVerificationFailure
```

---

## 97. Authenticator malformed input

Ejemplo:

```text
Bearer scheme with empty token
```

sí pertenece al Authenticator.

---

## 98. Authenticator protocol error

Ejemplo:

```text
OIDC callback missing required state
```

podrá ser un error/rejection protocol-specific.

---

## 99. Exception policy

Authenticators deberán retornar resultados tipados cuando la condición sea esperada.

Exceptions:

```text
programming error
configuration invalid
unexpected infrastructure failure
```

---

## 100. Fail-closed rule

Cualquier condición ambigua deberá tender a:

```text
REJECT / ERROR
```

no:

```text
try weaker authenticator
```

---

## 101. Fallback between authenticators

Fallback solo deberá ocurrir cuando:

```text
previous authenticator = NOT_SUPPORTED
```

No cuando:

```text
credential invalid
protocol malformed
verification failed
```

---

## 102. Why this matters

Ejemplo peligroso:

```text
Bearer token invalid
    ↓
fallback to session
    ↓
authenticated
```

Esto puede ocultar errores y crear credential confusion.

---

## 103. Credential downgrade protection

Si un cliente presenta explícitamente un mecanismo fuerte pero inválido:

```text
invalid passkey assertion
```

VoltStack no deberá automáticamente:

```text
try password
```

en la misma operación salvo flow explícito de retry.

---

## 104. Authenticator status

Podrá existir:

```text
AuthenticatorExecutionStatus
```

con:

```text
SUPPORTED
PASSPORT_CREATED
REJECTED
MALFORMED
ERROR
```

---

## 105. AuthenticatorResult

Aunque el resultado principal sea Passport, internamente podría existir:

```text
AuthenticationPassportResult
```

con metadata de ejecución.

---

## 106. Authenticator ordering

El orden deberá estar definido por configuración compilada.

No por:

```text
service registration order
container hash order
plugin load accident
```

---

## 107. Compilation

Durante bootstrap:

```text
Authenticator definitions
       ↓
validation
       ↓
descriptor generation
       ↓
firewall mapping
       ↓
compiled resolver index
```

---

## 108. Compiled Resolver Index

Ejemplo conceptual:

```text
web:
    session
    password
    passkey

api:
    bearer_token
    api_key

service:
    client_certificate
    service_token
```

---

## 109. Resolution hints

Para acelerar resolución se podrán compilar hints.

Ejemplo:

```text
Authorization header present
    candidate = bearer

session cookie present
    candidate = session

POST /login
    candidate = password
```

---

## 110. Hint is not trust

Un hint solo reduce candidatos.

No valida credenciales.

---

## 111. Structural pre-parser

Para protocolos complejos puede existir:

```text
AuthenticatorInputParser
```

separado del Authenticator.

Ejemplo:

```text
SAML parser
WebAuthn response parser
JWT structural parser
```

---

## 112. Parser responsibilities

```text
safe decode
size limits
schema validation
syntax validation
```

No:

```text
cryptographic trust
identity resolution
authorization
```

---

## 113. Decoding limits

Base64, JSON y XML deberán tener límites contra:

```text
memory exhaustion
deep nesting
entity expansion
huge tokens
```

---

## 114. XML safety

SAML processing deberá proteger contra problemas de XML parser.

El Authenticator no deberá utilizar configuraciones inseguras de entidades externas.

---

## 115. JSON safety

Payloads gigantes o deeply nested deberán rechazarse antes de verificaciones costosas.

---

## 116. Credential redaction

Cualquier objeto de debugging deberá mostrar:

```text
BearerTokenCredential([REDACTED])
PasswordCredential([REDACTED])
```

y nunca el secreto real.

---

## 117. toString restrictions

Credentials no deberían implementar `__toString()` retornando secretos.

---

## 118. Serialization restrictions

Credenciales sensibles deberán bloquear o controlar:

```text
serialize()
json_encode()
debug dump
```

cuando sea posible.

---

## 119. Authenticator logging

Puede registrar:

```text
authenticator name
method
support result
protocol
duration
```

No:

```text
credentials
assertions
full tokens
```

---

## 120. Authentication tracing

Span:

```text
auth.authenticator.resolve
```

y opcional:

```text
auth.authenticator.execute
```

---

## 121. Metrics

Ejemplos:

```text
authenticator_selected_total
authenticator_malformed_input_total
authenticator_conflict_total
authenticator_resolution_latency
passport_creation_latency
```

---

## 122. Cardinality controls

No deberán usarse como labels métricos:

```text
user id
email
token id
transaction id
```

si generan alta cardinalidad.

---

## 123. Authenticator auditing

Normalmente el Authenticator seleccionado será metadata dentro del evento de autenticación.

No es necesario un Audit Record separado por `supports()`.

---

## 124. Security-sensitive Authenticator events

Sí podrá auditarse:

```text
pre-authenticated gateway identity
service account authentication
client certificate identity
federated enterprise login
credential conflict
```

---

## 125. Testing interface

Todos los Authenticators deberán poder probarse independientemente.

Casos:

```text
supported input
unsupported input
malformed input
boundary sizes
missing fields
duplicate credentials
transport mismatch
```

---

## 126. PasswordAuthenticator tests

```text
POST login with valid shape
missing username
missing password
multiple password values
oversized identity claim
Unicode normalization
```

sin necesitar verificar hashes.

---

## 127. BearerAuthenticator tests

```text
missing header
wrong scheme
empty token
multiple headers
oversized token
weird whitespace
case handling
```

---

## 128. PasskeyAuthenticator tests

```text
missing transaction
malformed assertion
invalid encoding
oversized clientDataJSON
missing credential id
```

La validación criptográfica pertenece al verifier.

---

## 129. Fuzz testing

Parsers de:

```text
Authorization
JWT
SAML
WebAuthn
API keys
```

son candidatos a fuzzing.

---

## 130. Property-based tests

Especialmente para:

```text
header parsing
token extraction
normalization
conflict detection
```

---

## 131. Authenticator isolation tests

Un Authenticator no deberá:

```text
modify ContextStorage
write session unexpectedly
dispatch authorization decisions
```

---

## 132. Persistent-runtime tests

Una misma instancia de Authenticator podrá reutilizarse entre requests solo si es stateless.

Test:

```text
Request A → credential A
Request B → credential B
```

sin leakage.

---

## 133. Mutable authenticator state prohibited

Evitar:

```php
class PasswordAuthenticator
{
    private ?string $email = null;
}
```

cuando la instancia sea compartida.

---

## 134. Request-local parser state

Si se requiere estado temporal:

```text
local variables
operation context
request-scoped helper
```

no properties persistentes.

---

## 135. Dependency injection

Authenticators podrán depender de:

```text
input parsers
protocol configuration
transaction manager
URL generator
clock
crypto helpers
```

pero no de servicios innecesarios.

---

## 136. Minimal dependency principle

Un BearerTokenAuthenticator no debería depender de:

```text
SessionManager
PasswordHasher
AuthorizationEngine
```

---

## 137. Custom Authenticator API

Desarrolladores podrán crear:

```php
final class SignedWebhookAuthenticator
    implements AuthenticatorInterface
{
    public function supports(...): AuthenticatorSupportResult
    {
        // ...
    }

    public function createPassport(...): AuthenticationPassport
    {
        // ...
    }
}
```

---

## 138. Registration example

Conceptualmente:

```php
Auth::authenticator(
    'signed_webhook',
    SignedWebhookAuthenticator::class
);
```

y:

```php
'firewalls' => [
    'webhooks' => [
        'authenticators' => [
            'signed_webhook',
        ],
    ],
];
```

---

## 139. Custom Credential requirement

Un custom Authenticator normalmente deberá crear:

```text
CustomCredential
```

más un:

```text
CredentialVerifier
```

correspondiente.

Esto conserva separación de responsabilidades.

---

## 140. Plugin contract stability

VoltStack deberá mantener estable al menos:

```text
AuthenticatorInterface
AuthenticatorSupportResult
AuthenticationPassport builder API
AuthenticatorDescriptor registration
```

para ecosistema de plugins.

---

## 141. Internal SPI

Podrán permanecer internos:

```text
CompiledAuthenticatorIndex
AuthenticatorResolutionGraph
ProtocolParserCache
ResolutionHintCompiler
```

---

## 142. Authenticator versioning

Plugins podrán declarar compatibilidad:

```text
authenticator contract version
```

si el framework evoluciona de forma significativa.

---

## 143. Deprecation

Authenticators del Core deberán poder marcarse deprecated sin romper el sistema inmediatamente.

Ejemplo:

```text
legacy_token
```

---

## 144. Deprecated mechanism policy

VoltStack podrá permitir:

```text
warn
deny in production
allow only migration mode
```

para mecanismos inseguros.

---

## 145. Passwordless first-class support

La arquitectura no deberá asumir que `PasswordAuthenticator` es obligatorio.

Un Firewall puede ser:

```text
passkey only
OIDC only
certificate only
```

---

## 146. Password disabled identity

Una Identity puede no tener password registrado y seguir autenticándose mediante Passkey/OIDC.

---

## 147. MFA-first architecture

El Authenticator System no deberá necesitar clases combinadas para cada combinación:

```text
password + TOTP
password + passkey
OIDC + TOTP
OIDC + passkey
```

El pipeline general deberá componer factores.

---

## 148. Authentication chain

Un Authenticator puede iniciar un método y un step-up posterior añadir otro.

Ejemplo:

```text
OIDC Authenticator
        ↓
AuthenticationContext AAL1
        ↓
Passkey step-up
        ↓
AuthenticationContext AAL2
```

---

## 149. Authenticator does not own assurance

El Authenticator podrá aportar metadata sobre método, pero:

```text
AssuranceCalculator
```

determina el nivel final.

---

## 150. Hardware-backed hints

Un Passkey verifier puede producir evidencia:

```text
hardware_backed = true
user_verification = true
```

y el Assurance Calculator la interpreta.

No el Authenticator.

---

## 151. Authentication freshness

El Authenticator aporta:

```text
this proof verified now
```

pero la Policy define si cumple freshness requerido.

---

## 152. Identity-first vs Credential-first

Algunos mecanismos resuelven Identity primero:

```text
password
```

Otros validan credential primero:

```text
signed token with subject claim
```

El Authenticator System deberá soportar ambos sin romper el modelo.

---

## 153. Deferred identity resolution

Un Passport podrá contener:

```text
credential only
```

si el verifier produce luego un subject confiable.

Ejemplo:

```text
Bearer Token
```

---

## 154. IdentityClaim optionality

Por tanto, `AuthenticationPassport` no debería exigir siempre una IdentityClaim inicial.

Puede admitir:

```text
direct claim
deferred claim
credential-derived identity
```

---

## 155. IdentityResolutionStrategy

Podrá existir metadata:

```text
PRE_CREDENTIAL
POST_CREDENTIAL
PROVIDER_ASSERTION
DIRECT_TRUSTED_BINDING
```

---

## 156. Password flow

```text
claim
    ↓
identity
    ↓
password verify
```

---

## 157. Token flow

```text
token
    ↓
token verify
    ↓
trusted subject
    ↓
identity
```

---

## 158. Certificate flow

```text
certificate
    ↓
certificate verify
    ↓
subject
    ↓
service/user identity
```

---

## 159. Authenticator must declare flow needs

El descriptor podrá indicar:

```text
identity_resolution_phase
```

para que el Processor organice correctamente el pipeline.

---

## 160. Passport variants

Podrían existir:

```text
ClaimBasedAuthenticationPassport
CredentialDerivedIdentityPassport
FederatedAuthenticationPassport
ChallengeContinuationPassport
```

pero deberá evitarse una jerarquía excesiva si un modelo composable funciona mejor.

---

## 161. Recommended model

Preferible:

```text
AuthenticationPassport
+
IdentityResolutionHint
+
Credential[]
+
ProtocolState
```

sobre demasiadas subclases.

---

## 162. AuthenticationIntent

Un Passport podrá indicar:

```text
LOGIN
RECOVER_SESSION
STEP_UP
REAUTHENTICATE
CONTINUE_CHALLENGE
SERVICE_AUTHENTICATION
```

aunque la operación principal ya exista en Orchestration.

Debe evitarse duplicación de estado.

---

## 163. Authenticator-specific options

Configuración:

```php
'password' => [
    'identity_claim' => 'email',
    'password_field' => 'password',
],
```

podrá compilarse en un descriptor.

---

## 164. Configuration should not leak into runtime arrays

Preferir:

```text
CompiledPasswordAuthenticatorConfiguration
```

o value objects sobre leer config arrays constantemente.

---

## 165. Multi-provider authenticator

Un Authenticator no debería decidir providers arbitrariamente.

Podrá proporcionar:

```text
provider hint
```

si el protocolo lo requiere.

El `IdentityProviderResolver` toma la decisión final.

---

## 166. Federation provider hint

Ejemplo OIDC:

```text
provider = microsoft
issuer = ...
```

podrá formar parte de Passport metadata validada estructuralmente.

---

## 167. Dynamic provider selection security

No deberá aceptarse:

```text
?provider=arbitrary-url
```

como issuer dinámico sin allowlist.

---

## 168. SSRF protection

Authenticators que trabajen con remote issuers no deberán resolver URLs arbitrarias controladas por el cliente.

Configuración externa deberá estar pre-registrada o validada.

---

## 169. Key lookup security

Un token `kid` puede seleccionar una clave dentro de un issuer confiable.

No deberá convertirse en path/URL arbitraria.

---

## 170. Algorithm confusion

Token Authenticators/Verifiers deberán impedir que input no confiable seleccione algoritmos inseguros fuera de policy.

---

## 171. Protocol strictness

El Authenticator deberá rechazar variantes ambiguas de protocolo en lugar de ser excesivamente tolerante.

---

## 172. Duplicate parameter handling

OIDC/SAML/login payloads con parámetros duplicados relevantes deberán tener reglas explícitas.

Ejemplo:

```text
state=a&state=b
```

no deberá resolverse arbitrariamente.

---

## 173. Canonicalization

La normalización deberá ser mínima y protocol-aware.

No modificar secretos de forma que cambie su significado.

---

## 174. Password normalization

El framework no deberá trimpear passwords automáticamente salvo una policy explícita.

```text
" password "
```

puede ser una contraseña válida distinta.

---

## 175. Identity claim normalization

Emails/usernames sí pueden tener estrategias definidas por provider.

---

## 176. Unicode considerations

Claims deberán usar una normalización consistente para evitar cuentas visualmente ambiguas cuando la aplicación así lo requiera.

---

## 177. Credential lifetime

Credentials creadas por Authenticators deberán vivir únicamente durante la operación.

---

## 178. Credential cleanup

El Operation Context deberá liberar referencias cuando deje de necesitarlas.

---

## 179. Copy minimization

Evitar:

```text
request password
→ DTO password
→ passport copy
→ verifier copy
→ log context copy
```

Idealmente mantener referencias mínimas.

---

## 180. Secret zeroization limitations

PHP no permite garantizar zeroization completa de strings en memoria.

VoltStack deberá ser transparente sobre ello y minimizar lifetime/copies en lugar de prometer garantías imposibles.

---

## 181. Authenticator security invariants

### AUTH-AUTHN-01

`supports()` no verifica credenciales.

#### AUTH-AUTHN-02

Un Authenticator no crea `AuthenticationContext`.

#### AUTH-AUTHN-03

Un Authenticator no ejecuta Authorization.

#### AUTH-AUTHN-04

Un Authenticator no almacena secretos después de la operación.

#### AUTH-AUTHN-05

Input malformado no se convierte en `NOT_SUPPORTED` cuando podría permitir downgrade.

#### AUTH-AUTHN-06

Fallos de credential verification no provocan fallback automático a otro Authenticator.

#### AUTH-AUTHN-07

La selección entre Authenticators es determinista.

#### AUTH-AUTHN-08

Authenticators compartidos deben ser stateless.

#### AUTH-AUTHN-09

Protocol state debe ligarse a AuthenticationTransaction cuando corresponda.

#### AUTH-AUTHN-10

Un Authenticator no decide assurance final.

#### AUTH-AUTHN-11

Un Authenticator no decide MFA requerido.

#### AUTH-AUTHN-12

Claims derivadas de tokens/assertions no son confiables antes de verification.

#### AUTH-AUTHN-13

Los secrets nunca se registran.

#### AUTH-AUTHN-14

El Core debe poder incorporar Authenticators externos sin modificación.

---

## 182. Core Authenticators V1

Se recomienda incluir inicialmente:

```text
SessionAuthenticator
PasswordAuthenticator
BearerTokenAuthenticator
ApiKeyAuthenticator
RememberMeAuthenticator
```

como Core mínimo.

---

## 183. Core / optional boundary

Capacidades más complejas podrían vivir en Quantum packages:

```text
Quantum/WebAuthn
Quantum/Oidc
Quantum/Saml
Quantum/Ldap
```

integrándose mediante `AuthenticatorInterface`.

---

## 184. Recommended package split

```text
Quantum/Auth
    core auth architecture

Quantum/AuthPassword
    optional if separated

Quantum/WebAuthn
    PasskeyAuthenticator
    WebAuthn verifier

Quantum/Oidc
    OidcAuthenticator
    protocol client

Quantum/Saml
    SamlAuthenticator

Quantum/Ldap
    identity provider integration
```

La fragmentación exacta podrá decidirse posteriormente.

---

## 185. SessionAuthenticator flow

```text
Request
  ↓
session indicator
  ↓
SessionAuthenticator
  ↓
SessionReferenceCredential
  ↓
Recovery/Session validation
  ↓
AuthenticationContext
```

---

## 186. PasswordAuthenticator flow

```text
Login Request
   ↓
PasswordAuthenticator
   ↓
IdentityClaim + PasswordCredential
   ↓
IdentityProvider
   ↓
PasswordVerifier
   ↓
AuthenticationEvidence
```

---

## 187. BearerAuthenticator flow

```text
Authorization Header
   ↓
BearerTokenAuthenticator
   ↓
TokenCredential
   ↓
TokenVerifier
   ↓
Verified Token Claims
   ↓
Identity Resolution
```

---

## 188. ApiKeyAuthenticator flow

```text
API key header
   ↓
ApiKeyAuthenticator
   ↓
ApiKeyCredential
   ↓
ApiKeyVerifier
   ↓
Client/Service/User Identity
```

---

## 189. PasskeyAuthenticator flow

```text
Assertion
   ↓
PasskeyAuthenticator
   ↓
PasskeyAssertionCredential
   ↓
WebAuthnVerifier
   ↓
Verified cryptographic evidence
   ↓
Identity
```

---

## 190. OIDC Authenticator flow

```text
Begin
   ↓
transaction + state + nonce + PKCE
   ↓
external IdP
   ↓
callback
   ↓
OidcAuthenticator
   ↓
code/assertion Passport
   ↓
OIDC verification
   ↓
Federated Identity
```

---

## 191. ServiceAuthenticator flow

```text
Service request
   ↓
ServiceCredentialAuthenticator
   ↓
credential
   ↓
verifier
   ↓
ServiceIdentity
```

---

## 192. Authentication resolution example

Firewall:

```text
web:
    session
    password
    passkey
```

Request:

```text
POST /login
email/password body
```

Resolver:

```text
SessionAuthenticator → NOT_SUPPORTED
PasswordAuthenticator → SUPPORTED
PasskeyAuthenticator → NOT_SUPPORTED

Selected:
    PasswordAuthenticator
```

---

## 193. Conflict example

Request:

```text
Authorization: Bearer invalid-token
+
valid session cookie
```

Hybrid policy:

```text
explicit bearer has precedence
```

Entonces:

```text
Bearer selected
    ↓
verification fails
    ↓
REJECTED
```

No:

```text
fallback to session
```

---

## 194. Alternative conflict policy

Otra aplicación puede configurar:

```text
multiple credentials prohibited
```

Entonces:

```text
session + bearer
    ↓
AUTHENTICATION_CONFLICT
```

antes de verification.

---

## 195. Authenticator diagnostics example

Desarrollo:

```text
Firewall:
    api

Allowed Authenticators:
    bearer_token
    api_key

Support Results:
    bearer_token → SUPPORTED
    api_key → NOT_SUPPORTED

Selected:
    bearer_token
```

---

## 196. Authenticator health

Protocol Authenticators remotos podrán exponer health metadata mediante tooling separado.

Ejemplo:

```text
OIDC discovery available
JWKS cached
SAML metadata valid
```

Pero `supports()` no deberá hacer health checks remotos.

---

## 197. Dependency failure handling

Si selected Authenticator depende de un service unavailable:

```text
ERROR
```

No fallback automático a mecanismo más débil.

---

## 198. Authentication protocol profiles

VoltStack podrá definir profiles:

```text
web_password
spa_session
api_bearer
enterprise_oidc
service_mtls
```

que preconfiguren Authenticators.

---

## 199. Profiles are configuration only

No crean clases especiales ni rutas alternativas en el Core.

---

## 200. Authenticator factories

Instancias podrán construirse mediante:

```text
AuthenticatorFactoryInterface
```

para configuración específica.

---

## 201. Factory contract

Conceptualmente:

```php
interface AuthenticatorFactoryInterface
{
    public function create(
        AuthenticatorDescriptor $descriptor
    ): AuthenticatorInterface;
}
```

---

## 202. Lazy service proxies

En Container DI, el Registry podrá resolver un lazy proxy en lugar de factory manual.

La arquitectura no deberá imponer una técnica concreta.

---

## 203. Authenticator compilation validator

Deberá verificar:

```text
duplicate names
missing service
unsupported transport
invalid firewall binding
missing credential verifier
missing protocol dependency
invalid challenge capability
```

---

## 204. Verifier compatibility validation

Ejemplo:

```text
ApiKeyAuthenticator
    produces ApiKeyCredential

No ApiKeyCredentialVerifier registered
```

debe detectarse durante bootstrap cuando sea posible.

---

## 205. Protocol capability validation

Ejemplo:

```text
PasskeyAuthenticator
challenge_capable = true

No ChallengeStore configured
```

deberá advertirse o fallar.

---

## 206. Security linting

Tooling podrá advertir:

```text
API key accepted in query string
password login without rate limiter
OIDC without PKCE where expected
SAML issuer wildcard
pre-auth header without trusted proxy configuration
```

---

## 207. Authenticator introspection

CLI futura:

```text
volt auth:authenticators
```

podría mostrar:

```text
name
method
firewalls
transport
priority
capabilities
status
```

---

## 208. Firewall introspection

```text
volt auth:firewall api
```

podría indicar qué Authenticators son resolubles.

---

## 209. Core public contracts

Probablemente estables:

```text
AuthenticatorInterface
AuthenticatorSupportResult
AuthenticatorDescriptor
AuthenticatorRegistryInterface
AuthenticationPassport
AuthenticationRequest
```

---

## 210. Internal components

Probablemente internos:

```text
CompiledAuthenticatorResolver
AuthenticatorHintIndex
AuthenticatorResolutionGraph
AuthenticatorPipelineProfiler
```

---

## 211. Namespace sugerido

```text
VoltStack\Quantum\Auth\Authenticator
VoltStack\Quantum\Auth\Authenticator\Contracts
VoltStack\Quantum\Auth\Authenticator\Registry
VoltStack\Quantum\Auth\Authenticator\Resolution
VoltStack\Quantum\Auth\Authenticator\Builtin
VoltStack\Quantum\Auth\Authenticator\Protocol
```

---

## 212. Estructura sugerida

```text
src/Quantum/Auth/
└── Authenticator/
    ├── Contracts/
    │   ├── AuthenticatorInterface.php
    │   ├── InitiatingAuthenticatorInterface.php
    │   └── AuthenticatorRegistryInterface.php
    │
    ├── AuthenticatorDescriptor.php
    ├── AuthenticatorSupportResult.php
    ├── AuthenticatorCapability.php
    ├── AuthenticatorRegistry.php
    ├── AuthenticatorResolver.php
    ├── AuthenticatorResolution.php
    │
    ├── Builtin/
    │   ├── SessionAuthenticator.php
    │   ├── PasswordAuthenticator.php
    │   ├── BearerTokenAuthenticator.php
    │   ├── ApiKeyAuthenticator.php
    │   └── RememberMeAuthenticator.php
    │
    ├── Parsing/
    │   ├── AuthorizationHeaderParser.php
    │   ├── CredentialInputParser.php
    │   └── ProtocolInputParser.php
    │
    └── Compilation/
        ├── AuthenticatorCompiler.php
        └── CompiledAuthenticatorIndex.php
```

---

## 213. Anti-pattern — Fat Authenticator

Evitar:

```php
final class LoginAuthenticator
{
    public function authenticate()
    {
        // read request
        // query DB
        // verify password
        // verify TOTP
        // calculate risk
        // create session
        // authorize route
        // redirect
    }
}
```

Esto contradice toda la arquitectura.

---

## 214. Anti-pattern — Authenticator returns User

Evitar:

```php
public function authenticate(): User
```

Preferir:

```php
public function createPassport(): AuthenticationPassport
```

---

## 215. Anti-pattern — supports() remote call

Evitar:

```php
supports()
    → call OIDC discovery endpoint
```

---

## 216. Anti-pattern — fallback after invalid credential

Evitar:

```text
Bearer invalid
    ↓
try API key
    ↓
try session
```

como comportamiento automático.

---

## 217. Anti-pattern — Authenticator state leakage

Evitar properties con request data en servicios singleton.

---

## 218. Anti-pattern — trusting parsed claims

Evitar:

```text
decode JWT
    ↓
read sub
    ↓
authenticate user
```

sin signature/claim verification.

---

## 219. Anti-pattern — Pre-auth header trust

Evitar:

```text
X-Authenticated-User
    ↓
Identity
```

sin trusted boundary.

---

## 220. Anti-pattern — embedded Authorization

Evitar:

```text
PasswordAuthenticator
    if user is admin...
```

Roles y permisos no pertenecen al Authenticator.

---

## 221. Criterios de aceptación

El Authenticator System será considerado correcto cuando:

1. Authenticators tengan responsabilidad limitada;
2. soporten múltiples transportes;
3. produzcan Passports;
4. no verifiquen directamente Authorization;
5. no creen AuthenticationContext;
6. no almacenen estado mutable request-specific en servicios compartidos;
7. exista AuthenticatorRegistry;
8. exista resolución determinista;
9. se detecten conflictos;
10. se eviten fallbacks inseguros;
11. soporte password;
12. soporte session recovery;
13. soporte bearer token;
14. soporte API keys;
15. permita Passkeys;
16. permita OIDC;
17. permita SAML;
18. permita machine identities;
19. soporte interactive y non-interactive auth;
20. soporte challenge initiation;
21. soporte flujos multi-step;
22. sea extensible mediante plugins;
23. permita compilación;
24. sea observable;
25. sea testeable aisladamente;
26. minimice exposición de secretos;
27. sea seguro bajo FrankenPHP;
28. permita credential-derived identity;
29. mantenga separación con CredentialVerifier;
30. mantenga separación con Authorization.

---

## 222. Regla arquitectónica final

La secuencia deberá permanecer:

```text
UNTRUSTED INPUT
      ↓
AUTHENTICATOR
      ↓
STRUCTURED PASSPORT
      ↓
IDENTITY / CREDENTIAL VERIFICATION
      ↓
AUTHENTICATION EVIDENCE
      ↓
POLICY / ASSURANCE / DECISION
      ↓
AUTHENTICATION CONTEXT
```

En consecuencia:

> **El Authenticator no prueba por sí mismo que una identidad sea válida.**
> **El Authenticator únicamente traduce un mecanismo o protocolo de autenticación a un formato estructurado que el motor de Authentication pueda verificar.**

Esta separación permitirá que VoltStack tenga una API de autenticación sencilla y, al mismo tiempo, una arquitectura extensible capaz de incorporar mecanismos futuros sin reescribir el núcleo.

---

## 223. Próximo documento

El siguiente documento será:

```text
07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md
```

Su responsabilidad será profundizar específicamente en:

```text
Authenticator Resolver
candidate discovery
support evaluation
priority
specificity
explicit vs implicit credentials
multiple credential handling
credential conflict
identity conflict
fallback rules
downgrade prevention
selection policies
firewall-specific precedence
compiled resolution
resolution hints
ambiguity detection
performance
debugging
security invariants
```

separando la definición de los Authenticators del algoritmo encargado de seleccionar cuál debe ejecutarse para cada intento.
