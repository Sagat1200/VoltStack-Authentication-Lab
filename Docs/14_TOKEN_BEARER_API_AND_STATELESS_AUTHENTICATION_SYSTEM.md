# VoltStack Authentication System

## 14 — Token, Bearer, API and Stateless Authentication System

- **Archivo:** `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del subsistema de autenticación stateless basada en tokens

**Depende especialmente de:**

- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de autenticar requests mediante credenciales transportadas explícitamente en cada operación, sin depender de una `AuthenticationSession` de navegador.

El sistema deberá cubrir:

```text
Bearer Tokens
Opaque API Tokens
Personal Access Tokens
Machine Tokens
Service Tokens
Integration Tokens
Signed Tokens
JWT-based Credentials
Reference Tokens
Client Credentials
```

bajo una arquitectura común.

El flujo principal será:

```text
Request
   ↓
Explicit Token Credential
   ↓
Bearer/API Authenticator
   ↓
Token Verification
   ↓
Identity Resolution
   ↓
Identity Security Validation
   ↓
AuthenticationEvidence
   ↓
AuthenticationContext
```

---

## 2. Principio fundamental

VoltStack distinguirá estrictamente:

```text
TOKEN
    presented credential

VERIFIED TOKEN
    trusted result of token validation

IDENTITY
    canonical authenticated subject

TOKEN CLAIMS / ABILITIES
    authenticated attributes

AUTHORIZATION
    decision about allowed actions
```

Por tanto:

> **La presencia de un Bearer token no autentica por sí misma una request.**

Y:

> **Un token válido puede autenticar una Identity sin necesariamente autorizar la operación solicitada.**

---

## 3. Stateful vs Stateless Authentication

VoltStack deberá soportar ambos modelos:

```text
Stateful
    AuthenticationSession
    Remember-Me
    browser cookies

Stateless
    Bearer Token
    API Token
    signed assertion
    machine/service credential
```

---

## 4. Qué significa Stateless

En este documento `stateless` significa principalmente:

> La autenticación de la request no depende de una AuthenticationSession previamente creada en el servidor.

No significa necesariamente:

```text
no server lookup
no token repository
no revocation store
no cache
```

Un opaque token puede ser stateless desde la perspectiva de la request y aun requerir consulta a almacenamiento.

---

## 5. Stateless no significa self-contained

Deberán distinguirse:

```text
Reference / Opaque Token
    server resolves record

Self-Contained Token
    token carries signed claims
```

Ambos pueden utilizar:

```text
Authorization: Bearer ...
```

---

## 6. Objetivos

El subsistema deberá soportar:

```text
Authorization header extraction
Bearer scheme
opaque tokens
personal access tokens
service tokens
machine credentials
JWT verification
signed claims
token type validation
issuer validation
audience validation
expiration
not-before
revocation
rotation
token hashing
token families
scopes / abilities
tenant binding
identity binding
firewall binding
token provenance
SecurityVersion
credential versioning
risk signals
distributed verification
cache
audit
observability
FrankenPHP safety
```

---

## 7. No objetivos

No deberá sustituir:

```text
OAuth Authorization Server
OIDC Provider
Authorization Engine
Session System
Remember-Me
MFA
Credential enrollment
```

Aunque VoltStack podrá construir módulos adicionales sobre esta arquitectura.

---

## 8. Modelo general

```text
HTTP Request
     │
     ▼
Authorization Header
     │
     ▼
BearerTokenAuthenticator
     │
     ▼
BearerTokenCredential
     │
     ▼
TokenVerifierResolver
     │
     ├── OpaqueTokenVerifier
     ├── JwtTokenVerifier
     ├── PersonalAccessTokenVerifier
     └── ServiceTokenVerifier
     │
     ▼
VerifiedTokenCredential
     │
     ▼
Identity Resolution
     │
     ▼
Identity Security Validation
     │
     ▼
AuthenticationEvidence
     │
     ▼
AuthenticationContext
```

---

## 9. BearerTokenAuthenticator

Será responsable únicamente de:

```text
detect Authorization header
recognize Bearer scheme
validate structural format
extract raw token
create BearerTokenCredential
```

No deberá verificar:

```text
signature
database token
issuer
audience
expiration
scopes
identity
```

---

## 10. Authorization Header

Formato esperado:

```http
Authorization: Bearer <credential>
```

---

## 11. Header parsing

El parser deberá controlar:

```text
missing header
multiple Authorization headers
unsupported scheme
empty credential
oversized credential
unexpected whitespace
malformed encoding
```

---

## 12. Multiple Authorization headers

No deberá elegirse uno arbitrariamente.

Resultado:

```text
AMBIGUOUS
```

o:

```text
MALFORMED
```

según el adapter HTTP.

---

## 13. Unsupported schemes

Ejemplo:

```http
Authorization: Basic ...
```

en un Firewall Bearer-only.

Resultado:

```text
UNSUPPORTED_AUTHENTICATION_SCHEME
```

No deberá ignorarse para probar una sesión implícita salvo policy explícita.

---

## 14. Bearer credential semantics

Un bearer token sigue el principio:

```text
whoever possesses the token
can present it
```

Por tanto, su confidencialidad es crítica.

---

## 15. BearerTokenCredential

Conceptualmente:

```php
final readonly class BearerTokenCredential
    implements SensitiveCredentialInterface
{
    public function __construct(
        #[\SensitiveParameter]
        private string $token,
    ) {}
}
```

---

## 16. Token length limits

Antes de parseo/verificación costosos deberá existir:

```text
maximum token size
```

para reducir riesgos de DoS.

---

## 17. TokenType

VoltStack deberá modelar tipos diferentes.

Ejemplos:

```text
OPAQUE_API_TOKEN
PERSONAL_ACCESS_TOKEN
ACCESS_TOKEN
SERVICE_TOKEN
MACHINE_TOKEN
JWT_ACCESS_TOKEN
SIGNED_ASSERTION
INTERNAL_TOKEN
```

---

## 18. Token type no debe inferirse solo del contenido

No confiar únicamente en:

```text
token looks like JWT
```

para decidir su semántica.

Un string con tres segmentos Base64URL no significa automáticamente:

```text
trusted JWT access token
```

---

## 19. Token discrimination

La selección puede derivarse de:

```text
Firewall
Authenticator configuration
token prefix
issuer profile
explicit token registry
protocol metadata
```

---

## 20. Token prefix

Opaque tokens podrán usar prefixes como:

```text
vst_pat_
vst_api_
vst_srv_
```

para:

```text
operational identification
safe parser routing
secret scanning
developer ergonomics
```

pero el prefix no es prueba de validez.

---

## 21. Token prefix no contiene privilegios

Evitar:

```text
admin_token_
superuser_
```

como semántica de seguridad.

---

## 22. Opaque Token

Un opaque token no contiene claims interpretables por el cliente.

Ejemplo:

```text
vst_pat_xxxxx.yyyyy
```

El servidor lo resuelve contra almacenamiento confiable.

---

## 23. Opaque Token model

Preferible:

```text
selector.secret
```

similar a persistent credentials.

---

## 24. Selector

Permite lookup eficiente.

---

## 25. Secret

Debe ser:

```text
high entropy
CSPRNG generated
non-guessable
```

---

## 26. Token storage

Nunca almacenar raw opaque token en DB.

Preferir:

```text
selector
secret_digest
```

---

## 27. OpaqueTokenRecord

Conceptualmente:

```php
final readonly class OpaqueTokenRecord
{
    public function __construct(
        public TokenId $id,
        public TokenSelector $selector,
        public SecretDigest $secretDigest,
        public IdentityReference $identity,
        public TokenType $type,
        public TokenStatus $status,
        public TokenPolicySnapshot $policy,
        public \DateTimeImmutable $issuedAt,
        public ?\DateTimeImmutable $expiresAt,
    ) {}
}
```

---

## 28. TokenStatus

Estados posibles:

```text
ACTIVE
EXPIRED
REVOKED
COMPROMISED
ROTATED
SUPERSEDED
```

---

## 29. TokenId vs secret

El `TokenId` será una referencia segura para:

```text
audit
revocation
management
provenance
```

No será el bearer secret.

---

## 30. Public Token ID

Puede existir:

```text
TokenPublicId
```

para UI/API de administración sin revelar selector/secret.

---

## 31. TokenRepository

Contrato conceptual:

```php
interface TokenRepositoryInterface
{
    public function findBySelector(
        TokenSelector $selector
    ): TokenLookupResult;

    public function revoke(
        TokenId $id,
        TokenRevocationReason $reason
    ): void;
}
```

---

## 32. OpaqueTokenVerifier

Proceso:

```text
parse token
    ↓
lookup selector
    ↓
validate record status
    ↓
verify secret digest
    ↓
validate expiration
    ↓
validate binding
    ↓
VerifiedTokenCredential
```

---

## 33. Secret hashing

Como el token es aleatorio y de alta entropía, normalmente podrá utilizarse:

```text
secure digest
HMAC
keyed digest
```

sin necesidad de Argon2id.

---

## 34. PasswordHasher must not be reused blindly

Password y API token tienen threat models distintos.

---

## 35. Personal Access Token

Un Personal Access Token, PAT, representará una credential creada para una Identity humana o administrativa para acceder a APIs.

---

## 36. PAT is not a Session

No deberá utilizar browser-session semantics.

---

## 37. PAT model

Podrá contener server-side:

```text
identity
name
abilities
tenant
issuedAt
expiresAt
lastUsedAt
status
origin
```

---

## 38. Token names

Un nombre como:

```text
GitHub Integration
CLI Laptop
CI Token
```

es metadata informativa.

No forma parte del secret.

---

## 39. PAT creation

Deberá ser una operación sensible que normalmente requiere:

```text
authenticated Identity
fresh authentication
appropriate Authorization
```

---

## 40. Token displayed once

Al crear un opaque token:

```text
raw token
```

deberá mostrarse normalmente una única vez.

---

## 41. After creation

El sistema deberá almacenar solo digest.

Por tanto:

> **VoltStack no deberá ofrecer posteriormente una operación “mostrar mi token completo”.**

Solo:

```text
rotate
revoke
create replacement
```

---

## 42. PAT lifecycle

```text
CREATED
   ↓
ACTIVE
   ├──→ EXPIRED
   ├──→ REVOKED
   ├──→ COMPROMISED
   └──→ ROTATED/SUPERSEDED
```

---

## 43. API Token vs Personal Access Token

```text
API Token
    generic token concept

PAT
    user-controlled API credential
```

---

## 44. Service Token

Representa una credential para:

```text
ServiceIdentity
WorkloadIdentity
Application Client
```

---

## 45. Service tokens should not require fake users

Nunca obligar a crear:

```text
users.email = billing-service@example.com
```

para autenticar un servicio.

---

## 46. ServiceIdentity

Debe ser una Identity de primera clase.

---

## 47. Machine Token

Puede autenticar:

```text
device
node
agent
worker
machine
```

según arquitectura.

---

## 48. Token ownership

Cada token deberá estar ligado a una:

```text
IdentityReference
```

o, para ciertos protocolos, producirla tras verificación.

---

## 49. TokenBinding

Podrá incluir:

```text
Identity
Tenant
Firewall
Audience
Client
Device
Service
Purpose
```

---

## 50. Tenant binding

Token:

```text
tenant = ACME
```

no deberá autenticar request bajo:

```text
tenant = Globex
```

---

## 51. Global tokens

Podrán existir, pero deberán declararse explícitamente como tales.

---

## 52. Firewall binding

Un PAT para:

```text
api
```

no deberá autenticar:

```text
admin_internal
```

salvo realm compartido explícitamente.

---

## 53. Audience

Todos los tokens destinados a varios servicios deberán tener un concepto de:

```text
audience
```

aunque opaque tokens lo representen server-side.

---

## 54. Purpose

También podrá declararse:

```text
api_access
artifact_upload
webhook_signing
service_rpc
```

---

## 55. Purpose mismatch

Un token de:

```text
artifact_upload
```

no deberá utilizarse como token de login general.

---

## 56. Token scopes / abilities

VoltStack podrá asociar:

```text
abilities
scopes
capabilities
```

a un token.

---

## 57. Authentication vs scopes

El Token Verifier deberá confirmar que los scopes almacenados/firmados pertenecen realmente al token.

Pero decidir si permiten una acción corresponde a Authorization.

---

## 58. Example

Token válido:

```text
identity = Alice
abilities = ["invoice:read"]
```

Authentication:

```text
SUCCESS
```

Intento:

```text
DELETE invoice
```

puede resultar:

```text
AUTHORIZATION DENIED
```

sin invalidar el token.

---

## 59. TokenAbilitySet

Podrá ser un Value Object:

```php
final readonly class TokenAbilitySet
{
    // canonical authenticated token abilities
}
```

---

## 60. Ability strings

Deberán tener:

```text
canonical naming
bounded length
validated syntax
```

---

## 61. Wildcards

Si se soportan:

```text
*
invoice:*
```

su semántica deberá pertenecer al Authorization mapping, no a improvisación dentro del Verifier.

---

## 62. `*` token

No deberá implicar automáticamente superadministrador fuera del dominio definido.

---

## 63. JWT Authentication

VoltStack deberá soportar tokens JWT cuando sean apropiados.

Pero:

> **JWT es un formato de token, no un sistema de Authentication completo.**

---

## 64. JWT structure

```text
header.payload.signature
```

Parsear estos segmentos no establece confianza.

---

## 65. JWT verification flow

```text
raw JWT
   ↓
safe parser
   ↓
unverified header/payload
   ↓
issuer profile resolution
   ↓
algorithm policy
   ↓
key resolution
   ↓
signature verification
   ↓
claim validation
   ↓
token type validation
   ↓
VerifiedTokenCredential
```

---

## 66. Signature first trust boundary

Claims como:

```text
sub
scope
role
tenant
```

no deberán utilizarse como hechos confiables antes de verificación criptográfica y semántica.

---

## 67. `alg`

Nunca confiar en el algoritmo solicitado por el token fuera de una allowlist del servidor.

---

## 68. Algorithm policy

Cada issuer/token profile deberá declarar:

```text
allowed algorithms
required key types
minimum key strength
```

---

## 69. Algorithm confusion

VoltStack deberá impedir ataques donde un token fuerce un algoritmo no previsto.

---

## 70. `none`

No deberá aceptarse salvo un escenario explícito, aislado y excepcional que normalmente no formará parte del Auth estándar.

---

## 71. `kid`

Se utilizará únicamente para seleccionar una clave dentro de un key source confiable.

---

## 72. `kid` security

Nunca utilizarlo directamente como:

```text
filesystem path
URL
SQL fragment
class name
```

---

## 73. JWKS

Para issuers externos podrá existir:

```text
JwkSetProvider
```

---

## 74. JWKS caching

Deberá soportar:

```text
cache
rotation
TTL
refresh
bounded key lookup
```

---

## 75. Unknown kid

Puede provocar refresh controlado del key set.

No deberá provocar requests arbitrarias a URLs del token.

---

## 76. Issuer

El token deberá validarse contra:

```text
configured trusted issuer
```

---

## 77. Dynamic issuer prohibited by default

Nunca:

```php
$issuer = $unverifiedClaims['iss'];
fetch($issuer . '/.well-known/...');
```

sin una conexión previamente registrada y policy SSRF-safe.

---

## 78. Audience validation

Debe comprobarse cuando corresponda.

---

## 79. Subject

`sub` deberá convertirse en una:

```text
VerifiedIdentityClaim
```

solo después de validación completa.

---

## 80. Expiration

Claims temporales comunes:

```text
exp
nbf
iat
```

deberán validarse con `ClockInterface`.

---

## 81. Clock skew

Debe ser:

```text
explicit
small
profile-specific
```

---

## 82. Token Type

Un JWT válido criptográficamente puede seguir siendo incorrecto para la operación.

Ejemplos:

```text
ID Token
Refresh Token
Access Token
Magic Link Token
Email Verification Token
```

---

## 83. Token type confusion defense

Un:

```text
OIDC ID Token
```

no deberá aceptarse automáticamente como API access token.

---

## 84. `typ`

El header `typ` puede ayudar, pero no deberá ser la única defensa.

Se deberán validar:

```text
issuer profile
audience
purpose
claims
protocol context
```

---

## 85. JWT claim profile

Cada token profile deberá declarar:

```text
required claims
optional claims
forbidden claims where useful
mapping rules
```

---

## 86. JWT custom claims

No copiar automáticamente todos a AuthenticationContext.

---

## 87. Claim mapper

Podrá existir:

```text
VerifiedTokenClaimMapper
```

---

## 88. Self-contained token revocation

Este es uno de los principales trade-offs de JWT.

Si no hay lookup server-side:

```text
signed valid token
```

puede seguir siendo aceptado hasta expirar.

---

## 89. Revocation strategies

Podrán existir:

```text
SHORT_LIVED_ONLY
DENYLIST
SECURITY_VERSION
TOKEN_VERSION
INTROSPECTION
KEY_ROTATION
```

---

## 90. SHORT_LIVED_ONLY

Reduce ventana de revocación.

---

## 91. DENYLIST

Mantiene:

```text
jti / token fingerprint
```

revocado hasta expiration.

---

## 92. SECURITY_VERSION

Token incluye o referencia:

```text
identity security version
```

que se comprueba server-side.

Ya no es completamente lookup-free.

---

## 93. TOKEN_VERSION

La Identity/Client puede tener:

```text
tokenVersion
```

para invalidar familias de tokens.

---

## 94. INTROSPECTION

El token se valida contra authority central.

---

## 95. KEY_ROTATION

Puede invalidar grupos muy amplios, pero no sirve como mecanismo normal de revocación individual.

---

## 96. Statelessness spectrum

VoltStack deberá reconocer un continuo:

```text
Fully self-contained
        ↓
self-contained + denylist
        ↓
security-version checked
        ↓
introspected
        ↓
opaque reference token
```

No presentar "stateless" como una elección binaria.

---

## 97. Recommended defaults

Para first-party PAT/API credentials:

```text
opaque reference tokens
```

suelen ofrecer excelente revocabilidad y simplicidad.

Para protocolos distribuidos/federados:

```text
signed short-lived tokens
```

pueden ser apropiados.

---

## 98. No JWT-by-default dogma

VoltStack no deberá usar JWT automáticamente para toda API solo por ser popular.

---

## 99. TokenIssuer

VoltStack podrá incluir un servicio para tokens first-party.

```php
interface TokenIssuerInterface
{
    public function issue(
        TokenIssuanceRequest $request
    ): TokenIssuanceResult;
}
```

---

## 100. Token issuance is sensitive

Crear tokens requiere:

```text
authenticated actor
fresh auth when appropriate
Authorization
token policy
```

---

## 101. TokenIssuanceRequest

Podrá contener:

```text
subject IdentityReference
token type
name
abilities
tenant
audience
expiration
requested policy
```

---

## 102. Server decides final policy

El cliente no podrá exigir:

```text
expires = never
abilities = *
audience = everything
```

sin que policy/Authorization lo permita.

---

## 103. EffectiveTokenPolicy

Se resolverá desde:

```text
Framework security floor
Application policy
Firewall policy
Identity type policy
Tenant policy
Requested restrictions
```

---

## 104. Requested policy can narrow

Una solicitud podrá pedir:

```text
read-only
expires in 1 hour
```

y hacer el token más restrictivo.

---

## 105. Requested policy cannot broaden

No podrá superar la policy efectiva.

---

## 106. TokenPolicy

Podrá incluir:

```text
type
expiration
abilities
audiences
tenant binding
rotation
max lifetime
security version behavior
revocation model
```

---

## 107. TokenIssueResult

Para opaque tokens:

```text
raw token shown once
+
safe token metadata
```

---

## 108. Raw token boundary

Después de entregarlo:

```text
TokenIssuer
```

no deberá conservar raw token.

---

## 109. Signed token issuance

Para JWT:

```text
TokenSigner
```

generará una representación firmada.

---

## 110. Signing keys

No deberán estar acopladas al TokenIssuer directamente.

Preferir:

```text
SigningKeyProvider
SigningKeyResolver
```

---

## 111. Private key isolation

Keys podrán residir en:

```text
secret manager
HSM
KMS
secure filesystem
```

según deployment.

---

## 112. Signing algorithm policy

Debe ser explícita y versionable.

---

## 113. Key rotation

Tokens firmados deberán soportar:

```text
current signing key
previous verification keys
key ids
rotation schedule
```

---

## 114. Old signing keys

Podrán seguir verificando tokens no expirados durante transición.

---

## 115. Key compromise

Debe existir un incidente distinto de rotación normal.

Puede requerir:

```text
immediate key removal
token revocation
security alerts
forced credential renewal
```

---

## 116. TokenVerifierInterface

Contrato:

```php
interface TokenVerifierInterface
{
    public function verify(
        TokenVerificationRequest $request
    ): TokenVerificationResult;
}
```

---

## 117. TokenVerificationRequest

Podrá contener:

```text
TokenCredential
Firewall
Tenant
Expected Audience
Expected Token Type
AuthenticationOperation
Issuer Profile
```

---

## 118. TokenVerificationResult

Estados:

```text
VERIFIED
INVALID
EXPIRED
NOT_YET_VALID
REVOKED
COMPROMISED
WRONG_TYPE
WRONG_AUDIENCE
WRONG_ISSUER
BINDING_MISMATCH
REPLAYED
ERROR
```

---

## 119. VerifiedTokenCredential

Conceptualmente:

```php
final readonly class VerifiedTokenCredential
{
    public function __construct(
        public TokenType $type,
        public TokenReference $token,
        public AuthenticationProvenance $provenance,
        public VerifiedTokenClaimSet $claims,
        public TokenAbilitySet $abilities,
        public \DateTimeImmutable $verifiedAt,
    ) {}
}
```

---

## 120. No raw token in verified object

Nunca.

---

## 121. TokenReference

Puede contener:

```text
token id
issuer
jti
family
public id
```

según tipo.

---

## 122. VerifiedTokenClaimSet

Solo incluirá claims:

```text
validated
mapped
allowed
```

---

## 123. Identity derivation

Para opaque token:

```text
record → IdentityReference
```

Para JWT:

```text
verified issuer + sub
    ↓
Identity mapping
```

---

## 124. Token without local Identity

VoltStack podrá permitir external/federated Identity si el sistema está configurado para ello.

---

## 125. Service token identity

Puede producir directamente:

```text
ServiceIdentity
```

---

## 126. Client identity vs user identity

OAuth-style delegated tokens pueden involucrar:

```text
client identity
subject identity
```

Estos conceptos deberán preservarse, no fusionarse.

---

## 127. Actor / Subject

Podrá existir:

```text
ActorIdentity
SubjectIdentity
```

en escenarios delegados.

---

## 128. Simple first-party PAT

Normalmente:

```text
actor == subject
```

---

## 129. Delegated token

Puede ser:

```text
client application acting for user
```

La futura arquitectura de delegation deberá modelarlo formalmente.

---

## 130. Token scopes and delegation

No convertir scopes en sustituto de Actor/Subject.

---

## 131. AuthenticationEvidence

Token válido producirá:

```text
TokenAuthenticationEvidence
```

---

## 132. Evidence fields

Podrá incluir:

```text
token type
token reference
issuer
subject
client
audience
abilities
verification time
issued time
expiration
binding properties
```

---

## 133. Evidence does not grant permissions

Aunque contenga abilities, Authorization sigue siendo quien decide.

---

## 134. SecurityVersion integration

Opaque token records podrán capturar:

```text
SecurityVersion
```

al emitirse.

---

## 135. Verification

```text
token.securityVersion
        ==
identity.securityVersion
```

cuando la policy lo requiera.

---

## 136. Effect

Permite:

```text
forced logout/token invalidation
account recovery
compromise response
```

sin enumerar todos los tokens.

---

## 137. PAT password reset policy

Puede ser configurable:

```text
KEEP
REVOKE_USER_TOKENS
SECURITY_VERSION_INVALIDATE
REVOKE_ONLY_PASSWORD_DERIVED
```

---

## 138. Recommended high-security behavior

Account recovery debería poder invalidar tokens persistentes asociados a la Identity.

---

## 139. Service tokens and human password resets

No deben quedar acoplados accidentalmente.

Un ServiceIdentity no tiene por qué verse afectado por password state humano.

---

## 140. TokenVersion

Puede utilizarse separadamente de SecurityVersion.

---

## 141. TokenVersion use

```text
revoke all API tokens
```

sin invalidar browser sessions.

---

## 142. Version dimensions

Un modelo avanzado podría tener:

```text
SecurityVersion
SessionVersion
PersistentCredentialVersion
ApiTokenVersion
```

según necesidades.

---

## 143. Simplicidad V1

No proliferar versiones sin necesidad.

Podrá utilizarse:

```text
SecurityVersion
+
explicit token revocation
```

como base.

---

## 144. Token rotation

No todos los API tokens necesitan rotación automática por request.

---

## 145. PAT rotation

Normalmente será:

```text
manual/API-triggered
```

o periódica.

---

## 146. Refresh-token-style rotation

Pertenece más naturalmente a OAuth/token lifecycle especializado.

Puede construirse sobre primitives de token family.

---

## 147. Token replacement

Operación:

```text
Token A
    ↓
issue Token B
    ↓
revoke/supersede Token A
```

---

## 148. Graceful rotation

Para integraciones externas puede ser útil permitir:

```text
old and new token overlap
```

por un periodo corto.

---

## 149. RotationWindowPolicy

Podrá definir:

```text
NO_OVERLAP
SHORT_OVERLAP
SCHEDULED_CUTOVER
```

---

## 150. Service credential rotation

Especialmente útil para:

```text
CI/CD
external integrations
machine-to-machine
```

---

## 151. Token family

Podrá existir para credenciales rotatorias:

```text
TokenFamily
```

pero no deberá ser obligatorio para todos los PAT.

---

## 152. Token compromise

Un usuario/administrador podrá marcar:

```text
COMPROMISED
```

---

## 153. Compromise response

Podrá:

```text
revoke token
revoke family
revoke sessions derived from it
increment token/security version
raise risk signal
audit
```

---

## 154. Token usage metadata

Podrá registrar:

```text
last_used_at
last_used_ip summary
last_used_region
last_used_client
```

con data minimization.

---

## 155. `last_used_at`

No deberá actualizarse necesariamente sin límite en cada request si genera write amplification.

---

## 156. Touch interval

Podrá utilizarse:

```text
update at most once every N minutes
```

para metadata no crítica.

---

## 157. Revocation remains immediate

La optimización de `last_used_at` no deberá afectar status/revocation checks.

---

## 158. Token usage tracking

Puede ser:

```text
NONE
BASIC
SECURITY
DETAILED
```

según privacidad/riesgo.

---

## 159. API authentication hot path

Debe ser eficiente.

Opaque token típico:

```text
parse
lookup by selector
digest verify
status/expiry
Identity state
Context build
```

---

## 160. Token lookup caching

Puede utilizarse con cuidado.

---

## 161. Revocation-safe cache

Debe respetar:

```text
short TTL
invalidation
versioning
```

---

## 162. High-security APIs

Pueden deshabilitar cache de credential records.

---

## 163. JWT verification caching

Puede cachearse:

```text
issuer configuration
JWKS/public keys
parsed immutable policy
```

No hace falta cachear el raw token globalmente.

---

## 164. Verified token cache

Puede ser útil dentro de una request.

Cross-request caching de token verification requiere diseño cuidadoso porque puede retrasar revocación.

---

## 165. Request memoization

Sí:

```text
same token
same operation
same request
```

debe verificarse una vez.

---

## 166. Token fingerprint

Para memoización/telemetría podrá utilizarse un fingerprint no reversible.

---

## 167. Do not fingerprint low-entropy tokens insecurely

Los tokens generados por VoltStack deberán ser high-entropy.

Para credentials externas, deberá evaluarse el threat model.

---

## 168. AuthenticationRateLimiter

Token Authentication podrá integrarse con rate limiting, especialmente para:

```text
bad token floods
unknown selector attacks
signature verification DoS
introspection abuse
```

---

## 169. JWT DoS

Antes de criptografía costosa:

```text
size limits
segment count
encoding validation
header limits
claim depth limits
```

---

## 170. Key lookup DoS

Un atacante no deberá poder generar infinitos `kid` y provocar requests remotas ilimitadas.

---

## 171. JWKS refresh throttling

Debe existir:

```text
bounded refresh
negative key cache
cooldown
```

---

## 172. Introspection

Tokens externos opaque pueden validarse mediante:

```text
TokenIntrospectionClient
```

---

## 173. Introspection result

Debe considerarse información confiable únicamente si:

```text
endpoint trusted
TLS valid
client authenticated
response valid
issuer profile configured
```

---

## 174. Introspection outage

Resultado:

```text
ERROR
```

No:

```text
assume active
```

---

## 175. Introspection caching

Puede utilizarse hasta un límite corto derivado de:

```text
token expiry
revocation requirements
risk profile
```

---

## 176. Negative introspection caching

Debe ser corto y cuidadoso.

---

## 177. Service-to-service Authentication

VoltStack deberá soportar APIs internas sin browser/session assumptions.

Ejemplo:

```text
Service A
    ↓
Authorization: Bearer service-token
    ↓
Service B
```

---

## 178. Service credentials should be least privilege

Token policy podrá limitar:

```text
audience
abilities
tenant
purpose
expiration
```

---

## 179. Long-lived service tokens

Deberán evitarse cuando infraestructura soporte better workload identity.

Pero VoltStack deberá soportarlos para compatibilidad.

---

## 180. Workload identity preference

En cloud environments, pueden preferirse:

```text
short-lived federated workload credentials
mTLS
cloud workload identity
```

sobre secrets estáticos.

---

## 181. Static service token warning

Tooling podrá advertir:

```text
non-expiring service token
broad abilities
no audience
```

---

## 182. Token expiration

Todo token deberá tener una policy explícita:

```text
required expiration
optional expiration
non-expiring allowed
```

---

## 183. Non-expiring tokens

Si se permiten:

```text
must be explicitly configured
revocable
auditable
restricted
```

---

## 184. PAT default

Es preferible favorecer expiración configurable en lugar de credenciales eternas silenciosas.

---

## 185. Token `issuedAt`

Debe registrarse server-side.

---

## 186. Token `lastUsedAt`

Metadata operacional.

---

## 187. Token `expiresAt`

Fuente autoritativa para opaque tokens.

---

## 188. JWT `exp`

Debe comprobarse contra clock del servidor.

---

## 189. Not-before

`nbf` o equivalente deberá respetarse.

---

## 190. Token clock source

Siempre:

```text
ClockInterface
```

---

## 191. Time precision

No depender de timezones de presentación.

---

## 192. RevocationReason

Ejemplos:

```text
USER_REVOKED
ADMIN_REVOKED
ROTATED
COMPROMISED
ACCOUNT_RECOVERY
IDENTITY_DISABLED
TENANT_REVOKED
SECURITY_VERSION_CHANGED
POLICY_CHANGED
```

---

## 193. Token revocation service

```php
interface TokenRevocationServiceInterface
{
    public function revoke(
        TokenReference $token,
        TokenRevocationContext $context
    ): TokenRevocationResult;
}
```

---

## 194. Revoke by identity

Deberá soportarse:

```text
revoke all tokens for Identity
```

---

## 195. Revoke by tenant

```text
identity + tenant
```

---

## 196. Revoke by token type

Ejemplo:

```text
revoke PATs
keep service delegation tokens
```

según dominio.

---

## 197. Revoke by family

Para credenciales rotatorias.

---

## 198. Token management API

Conceptualmente:

```php
Auth::tokens()->create(...);
Auth::tokens()->all();
Auth::tokens()->revoke($token);
Auth::tokens()->rotate($token);
```

---

## 199. Security boundary

Estas APIs deberán pasar por Authorization.

Authentication solo ofrece primitives.

---

## 200. Token list output

Nunca mostrar:

```text
raw secret
secret digest
full JWT
```

---

## 201. Safe token metadata

Sí:

```text
public id
name
abilities
createdAt
expiresAt
lastUsedAt
status
```

---

## 202. Token display hint

Podrá conservarse:

```text
last four characters
```

solo si no reduce seguridad de forma relevante y sirve para UX.

---

## 203. Raw token recovery

No disponible.

---

## 204. Token reissue

Si se perdió:

```text
create replacement
```

---

## 205. API Key Authenticator

Además de Bearer, VoltStack podrá soportar:

```text
X-API-Key
```

u otros channels configurados.

---

## 206. Recommended transport

Preferir:

```text
Authorization header
dedicated secure header
```

sobre query string.

---

## 207. Query tokens

Deberán estar deshabilitados por defecto.

---

## 208. Por qué

URLs pueden filtrarse en:

```text
logs
analytics
browser history
proxy logs
referrers
screenshots
```

---

## 209. WebSocket authentication

El sistema podrá adaptar tokens a WebSocket handshake.

---

## 210. Long-lived WebSocket

Una vez autenticada la conexión, deberá existir policy para:

```text
token expiry during connection
revocation
SecurityVersion changes
reauthentication
```

---

## 211. Queue/job authentication

Un job interno puede utilizar:

```text
delegated identity proof
service token
signed execution context
```

pero no deberá reutilizar un bearer token arbitrariamente si no está diseñado para ello.

---

## 212. CLI authentication

PATs pueden ser útiles para:

```text
VoltStack CLI
developer tooling
automation
```

---

## 213. CLI token storage

La seguridad local del token pertenece parcialmente al cliente, pero documentación deberá recomendar secret stores apropiados.

---

## 214. Token Transport Adapter

Deberá abstraer:

```text
HTTP header
WebSocket handshake
CLI environment/credential store
internal RPC metadata
```

sin modificar el dominio del token.

---

## 215. AuthenticationContext

Para stateless auth, el Context normalmente vive únicamente durante la request.

---

## 216. No automatic Session creation

Un API Firewall stateless no deberá crear:

```text
AuthenticationSession
```

automáticamente tras Bearer Authentication.

---

## 217. Hybrid Firewall

Puede existir:

```text
session + bearer
```

pero selección deberá obedecer `07_AUTHENTICATOR_RESOLUTION...`.

---

## 218. Bearer precedence

Un hybrid Firewall puede definir:

```text
explicit bearer > implicit session
```

---

## 219. Invalid Bearer

Si Bearer fue seleccionado y falla:

```text
do not fallback to session
```

---

## 220. API stateless Context lifecycle

```text
Request begins
    ↓
Token Authentication
    ↓
AuthenticationContext
    ↓
Authorization/Application
    ↓
Request ends
    ↓
Context destroyed
```

---

## 221. FrankenPHP

Esta propiedad es especialmente importante.

Nunca almacenar:

```text
current token
current identity
current abilities
```

en singleton mutable.

---

## 222. Shared verifier services

Sí pueden compartirse si son:

```text
stateless
immutable
request-independent
```

---

## 223. Request scope

Debe contener:

```text
raw token credential
verification result
Identity
AuthenticationContext
```

---

## 224. Fiber safety

Concurrent requests deben tener contexts separados.

---

## 225. Token secret lifetime

El raw token deberá descartarse tan pronto como sea posible después de verification.

---

## 226. PHP zeroization limitation

VoltStack minimizará copies/lifetime sin afirmar borrado garantizado de RAM.

---

## 227. Logging

Nunca registrar:

```text
Authorization header
raw token
secret digest
private signing keys
full assertion
```

---

## 228. Redaction

Ejemplo:

```text
Authorization: Bearer [REDACTED]
```

---

## 229. Error rendering

Nunca incluir token en stack trace contextual.

---

## 230. Audit

Eventos sensibles:

```text
TokenIssued
TokenRevoked
TokenRotated
TokenCompromised
TokenAuthenticationSucceeded
TokenAuthenticationRejected
TokenReplayDetected
SigningKeyRotated
```

---

## 231. High-volume token success

No todas las requests API exitosas necesitan un audit record persistente.

Usar:

```text
metrics
tracing
access logs with safe references
```

---

## 232. Token issuance and revocation should be audited

Sí, por tratarse de lifecycle de credentials.

---

## 233. Observability spans

```text
auth.token.extract
auth.token.resolve_verifier
auth.token.verify
auth.token.lookup
auth.token.introspect
auth.token.identity_map
auth.token.issue
auth.token.revoke
```

---

## 234. Metrics

Ejemplos:

```text
auth_token_verification_total
auth_token_failure_total
auth_token_expired_total
auth_token_revoked_total
auth_token_wrong_audience_total
auth_token_wrong_type_total
auth_token_issue_total
auth_token_revoke_total
auth_token_lookup_latency
auth_token_signature_verification_latency
auth_token_introspection_latency
```

---

## 235. Safe labels

```text
token_type
verifier
firewall
status
issuer_profile
```

Evitar:

```text
token id
jti
user id
subject
```

como labels sin control.

---

## 236. Security telemetry

Detectar:

```text
many invalid tokens
repeated revoked token
unknown selectors
unknown kids
audience probing
issuer probing
token reuse anomalies
```

---

## 237. Enumeration

Externamente varios failures pueden mapearse a:

```text
invalid authentication credentials
```

---

## 238. HTTP semantics

Para un token inválido/ausente en recurso protegido:

```text
401 Unauthorized
```

según integración HTTP.

---

## 239. Authorization failure

Token válido pero permiso insuficiente:

```text
403 Forbidden
```

según policy/protocolo.

---

## 240. `WWW-Authenticate`

El HTTP adapter podrá emitir:

```http
WWW-Authenticate: Bearer
```

y parámetros seguros según protocolo.

---

## 241. Error detail

No deberá revelar:

```text
token belongs to Alice but expired
```

a un cliente no confiable.

---

## 242. Token introspection diagnostics

Internamente sí deberá conservarse classification.

---

## 243. Token parser testing

Casos:

```text
missing
empty
oversized
multiple headers
bad scheme
malformed prefix
invalid encoding
```

---

## 244. Opaque token tests

```text
valid selector/secret
unknown selector
wrong secret
expired
revoked
tenant mismatch
firewall mismatch
SecurityVersion mismatch
```

---

## 245. PAT tests

```text
issue
display once
list metadata
revoke
rotate
abilities
expiration
last-used tracking
```

---

## 246. JWT tests

```text
valid signature
bad signature
wrong alg
alg none
wrong key type
unknown kid
wrong issuer
wrong audience
expired
nbf
wrong token type
malformed claims
oversized payload
```

---

## 247. Key rotation tests

```text
new key signs
old key verifies old tokens
retired key rejected after allowed window
```

---

## 248. Revocation tests

```text
individual token
family
identity-wide
tenant-bound
security version
```

---

## 249. Hybrid Firewall tests

```text
bearer only
session only
bearer + session
invalid bearer + valid session
```

El último deberá respetar anti-downgrade.

---

## 250. Concurrency tests

```text
token revoke during verification
token rotate concurrent requests
last-used touch races
SecurityVersion change before Context commit
```

---

## 251. Persistent runtime tests

Request A:

```text
Token Alice
```

Request B:

```text
Token Bob
```

Request C:

```text
No token
```

Cada una deberá permanecer aislada.

---

## 252. Fuzz testing

Especialmente:

```text
Authorization parser
JWT parser
Base64URL decoder
claim parser
token prefix parser
```

---

## 253. Property-based testing

Útil para:

```text
token state machine
expiration boundaries
ability canonicalization
audience matching
binding invariants
```

---

## 254. Token state machine

```text
CREATED
   ↓
ACTIVE
   ├──► EXPIRED
   ├──► REVOKED
   ├──► COMPROMISED
   └──► SUPERSEDED / ROTATED
```

Los estados terminales no deberán regresar a `ACTIVE`.

---

## 255. Security invariant — Token

### AUTH-TOKEN-01

Raw tokens are always untrusted credentials.

#### AUTH-TOKEN-02

Token parsing never implies token verification.

#### AUTH-TOKEN-03

Opaque raw secrets are not persisted server-side.

#### AUTH-TOKEN-04

Raw bearer credentials never enter logs, events or metrics.

#### AUTH-TOKEN-05

Every token has an explicit semantic type.

#### AUTH-TOKEN-06

A cryptographically valid token with wrong purpose/type is invalid for the operation.

#### AUTH-TOKEN-07

Token validity does not imply Authorization success.

#### AUTH-TOKEN-08

Tenant/Firewall/Audience bindings are validated explicitly.

#### AUTH-TOKEN-09

Identity Security State can invalidate otherwise valid credentials.

#### AUTH-TOKEN-10

Token verification failure never downgrades automatically to another Authenticator.

---

## 256. Security invariant — JWT

### AUTH-JWT-01

Unverified claims are never trusted.

#### AUTH-JWT-02

Allowed algorithms are server-configured.

#### AUTH-JWT-03

`kid` cannot trigger arbitrary file/network access.

#### AUTH-JWT-04

Issuer is validated against trusted profiles.

#### AUTH-JWT-05

Audience is validated where required.

#### AUTH-JWT-06

Expiration and not-before are validated using server clock.

#### AUTH-JWT-07

Token type confusion is explicitly prevented.

#### AUTH-JWT-08

Unknown or unsupported algorithms fail closed.

#### AUTH-JWT-09

JWKS refresh is bounded.

#### AUTH-JWT-10

Self-contained tokens have explicit revocation semantics.

---

## 257. Security invariant — Opaque Tokens

### AUTH-OPAQUE-01

Selectors contain no authentication authority.

#### AUTH-OPAQUE-02

Secrets are high-entropy and CSPRNG-generated.

#### AUTH-OPAQUE-03

Stored secret representations are one-way.

#### AUTH-OPAQUE-04

Secret comparison uses safe cryptographic primitives.

#### AUTH-OPAQUE-05

Unknown token store state never authenticates.

#### AUTH-OPAQUE-06

Revocation is checked before Context creation.

---

## 258. Security invariant — Issuance

### AUTH-TOKEN-ISSUE-01

Token issuance is a security-sensitive operation.

#### AUTH-TOKEN-ISSUE-02

Client-requested abilities cannot exceed server policy.

#### AUTH-TOKEN-ISSUE-03

Raw opaque tokens are revealed only at issuance.

#### AUTH-TOKEN-ISSUE-04

Non-expiring credentials require explicit policy.

#### AUTH-TOKEN-ISSUE-05

Token creation is auditable.

#### AUTH-TOKEN-ISSUE-06

Private signing keys are not exposed through Authentication domain objects.

---

## 259. Security invariant — Runtime

### AUTH-TOKEN-RT-01

Current token state is request-scoped.

#### AUTH-TOKEN-RT-02

Shared token verifiers remain stateless.

#### AUTH-TOKEN-RT-03

AuthenticationContext is cleared after every request.

#### AUTH-TOKEN-RT-04

Concurrent fibers cannot share token/identity state.

#### AUTH-TOKEN-RT-05

Cross-request verification caches cannot bypass revocation policy.

---

## 260. Anti-pattern — JWT means authenticated

Evitar:

```php
$payload = base64_decode($token);
$userId = $payload['sub'];
```

sin verification.

---

## 261. Anti-pattern — JWT everywhere

No utilizar JWT para toda API únicamente para evitar una DB query.

---

## 262. Anti-pattern — PAT stored plaintext

Nunca.

---

## 263. Anti-pattern — query-string API key

Deshabilitado por defecto.

---

## 264. Anti-pattern — roles in token become Authorization truth automatically

Claims autenticadas aún necesitan policy de Authorization.

---

## 265. Anti-pattern — token type guessing

No decidir semántica solo porque el string contiene puntos.

---

## 266. Anti-pattern — arbitrary `iss` discovery

Riesgo SSRF/trust confusion.

---

## 267. Anti-pattern — arbitrary `kid` lookup

Nunca filesystem/network dinámico no controlado.

---

## 268. Anti-pattern — bearer fallback

```text
invalid bearer
    ↓
try session
```

prohibido salvo nueva operación explícita.

---

## 269. Anti-pattern — endless API token

No debe ser el default silencioso.

---

## 270. Anti-pattern — token abilities bypass Authorization

Nunca.

---

## 271. Anti-pattern — fake users for services

Utilizar `ServiceIdentity`.

---

## 272. Anti-pattern — raw token in telemetry

Nunca.

---

## 273. Anti-pattern — process-global current token

Crítico bajo FrankenPHP.

---

## 274. Anti-pattern — token revocation means delete only

Puede ser útil conservar tombstone/status para audit y race handling.

---

## 275. Anti-pattern — stale cache authenticates revoked token

Cross-request cache debe respetar revocation.

---

## 276. Componentes principales

```text
BearerTokenAuthenticator
ApiKeyAuthenticator

BearerTokenCredential
ApiKeyCredential
OpaqueTokenCredential
JwtTokenCredential

TokenType
TokenId
TokenPublicId
TokenSelector
TokenStatus
TokenReference

TokenVerifier
TokenVerifierRegistry
TokenVerifierResolver

OpaqueTokenVerifier
JwtTokenVerifier
PersonalAccessTokenVerifier
ServiceTokenVerifier

VerifiedTokenCredential
VerifiedTokenClaimSet
TokenAbilitySet
```

---

## 277. Componentes lifecycle

```text
TokenIssuer
TokenRepository
TokenRevocationService
TokenRotationService
TokenFamily
TokenPolicy
TokenPolicyResolver
TokenUsageTracker
TokenPruner
```

---

## 278. Componentes JWT

```text
JwtParser
JwtVerifier
JwtIssuerProfile
JwtClaimPolicy
SigningKeyProvider
VerificationKeyProvider
JwkSetProvider
JwkSetCache
VerifiedTokenClaimMapper
```

---

## 279. Componentes externos

```text
TokenIntrospectionClient
ExternalTokenIssuerProfile
FederatedTokenIdentityMapper
```

---

## 280. Namespace sugerido

```text
VoltStack\Quantum\Auth\Token
VoltStack\Quantum\Auth\Token\Contracts
VoltStack\Quantum\Auth\Token\Credential
VoltStack\Quantum\Auth\Token\Verification
VoltStack\Quantum\Auth\Token\Opaque
VoltStack\Quantum\Auth\Token\Jwt
VoltStack\Quantum\Auth\Token\Issuance
VoltStack\Quantum\Auth\Token\Lifecycle
VoltStack\Quantum\Auth\Token\Policy
VoltStack\Quantum\Auth\Token\Store
```

---

## 281. Estructura sugerida

```text
src/Quantum/Auth/Token/
├── Contracts/
│   ├── TokenVerifierInterface.php
│   ├── TokenRepositoryInterface.php
│   ├── TokenIssuerInterface.php
│   └── TokenRevocationServiceInterface.php
│
├── Credential/
│   ├── BearerTokenCredential.php
│   ├── ApiKeyCredential.php
│   ├── OpaqueTokenCredential.php
│   └── JwtTokenCredential.php
│
├── Verification/
│   ├── TokenVerificationRequest.php
│   ├── TokenVerificationResult.php
│   ├── TokenVerificationStatus.php
│   ├── TokenVerifierRegistry.php
│   ├── TokenVerifierResolver.php
│   └── VerifiedTokenCredential.php
│
├── Opaque/
│   ├── OpaqueTokenRecord.php
│   ├── OpaqueTokenVerifier.php
│   ├── TokenSelector.php
│   ├── TokenSecretHasher.php
│   └── OpaqueTokenGenerator.php
│
├── Jwt/
│   ├── JwtParser.php
│   ├── JwtTokenVerifier.php
│   ├── JwtIssuerProfile.php
│   ├── JwtClaimPolicy.php
│   ├── SigningKeyProvider.php
│   ├── VerificationKeyProvider.php
│   ├── JwkSetProvider.php
│   └── JwkSetCache.php
│
├── Issuance/
│   ├── TokenIssuer.php
│   ├── TokenIssuanceRequest.php
│   ├── TokenIssuanceResult.php
│   └── TokenPolicyResolver.php
│
├── Lifecycle/
│   ├── TokenRevocationService.php
│   ├── TokenRotationService.php
│   ├── TokenFamily.php
│   ├── TokenUsageTracker.php
│   └── TokenPruner.php
│
└── Policy/
    ├── TokenPolicy.php
    ├── TokenAbilitySet.php
    ├── TokenAudienceSet.php
    └── TokenBindingPolicy.php
```

---

## 282. Configuración conceptual — API opaque tokens

```php
'firewalls' => [

    'api' => [
        'state' => 'stateless',

        'authenticators' => [
            'bearer_token',
        ],

        'tokens' => [
            'profile' => 'first_party_api',
            'type' => 'opaque',
            'security_version_validation' => true,
        ],
    ],

];
```

---

## 283. Configuración conceptual — PAT

```php
'token_profiles' => [

    'personal_access' => [
        'type' => 'opaque',
        'expires' => '90 days',
        'abilities' => true,
        'tenant_binding' => true,
        'revocable' => true,
    ],

];
```

---

## 284. Configuración conceptual — JWT

```php
'token_profiles' => [

    'external_oidc_access' => [
        'type' => 'jwt',
        'issuer' => 'corporate',
        'allowed_algorithms' => [
            'RS256',
        ],
        'audience' => 'voltstack-api',
        'token_type' => 'access_token',
    ],

];
```

Los nombres/algoritmos son ilustrativos; la configuración final deberá expresarse mediante perfiles criptográficos seguros.

---

## 285. Configuración conceptual — service token

```php
'token_profiles' => [

    'internal_service' => [
        'type' => 'opaque',
        'identity_type' => 'service',
        'audience_required' => true,
        'default_lifetime' => '30 days',
        'rotation' => 'manual',
    ],

];
```

---

## 286. Flujo PAT completo

```text
CLI
  ↓
Authorization: Bearer vst_pat_selector.secret
  ↓
BearerTokenAuthenticator
  ↓
OpaqueTokenVerifier
  ↓
selector lookup
  ↓
secret digest verification
  ↓
status/expiration
  ↓
tenant/firewall
  ↓
IdentityReference
  ↓
Identity Security State
  ↓
TokenAuthenticationEvidence
  ↓
AuthenticationContext
  ↓
API Authorization
```

---

## 287. Flujo JWT completo

```text
Authorization: Bearer eyJ...
        ↓
BearerTokenAuthenticator
        ↓
JwtTokenCredential
        ↓
JwtParser
        ↓
Unverified header/claims
        ↓
Trusted Issuer Profile
        ↓
Allowed Algorithm
        ↓
Verification Key
        ↓
Signature Verification
        ↓
Issuer + Audience + Type + Time Validation
        ↓
Verified Claims
        ↓
issuer + subject
        ↓
Identity Mapping
        ↓
Identity Security State
        ↓
AuthenticationEvidence
        ↓
AuthenticationContext
```

---

## 288. Flujo token revocado

```text
Bearer token
    ↓
lookup / verify
    ↓
Token status = REVOKED
    ↓
Authentication rejected
    ↓
safe 401 response
    ↓
security telemetry
```

---

## 289. Flujo wrong audience

```text
Valid signature
      ↓
Issuer valid
      ↓
Audience != current API
      ↓
WRONG_AUDIENCE
      ↓
no AuthenticationContext
```

---

## 290. Flujo wrong token type

```text
Valid OIDC ID Token
      ↓
presented to REST API
      ↓
expected ACCESS_TOKEN
      ↓
WRONG_TYPE
      ↓
reject
```

---

## 291. Flujo SecurityVersion

```text
PAT:
    SecurityVersion = 5

Identity:
    SecurityVersion = 6

        ↓

TokenVerifier
        ↓
STALE
        ↓
reject
```

---

## 292. Flujo service token

```text
Service A
   ↓
service token
   ↓
ServiceTokenVerifier
   ↓
ServiceIdentityReference
   ↓
ServiceIdentityProvider
   ↓
ServiceIdentity
   ↓
AuthenticationContext
   ↓
Authorization
```

---

## 293. Flujo hybrid

```text
Request:
    session cookie
    bearer token

Firewall:
    bearer > session

        ↓

Bearer selected
        ↓
Token verification fails
        ↓
AUTHENTICATION FAILED
```

No:

```text
fallback to session
```

---

## 294. Flujo token issuance

```text
Authenticated Identity
        ↓
Fresh Auth Check
        ↓
Authorization:
    may create token?
        ↓
Requested abilities/lifetime
        ↓
TokenPolicyResolver
        ↓
EffectiveTokenPolicy
        ↓
Generate opaque secret
        ↓
Store digest + metadata
        ↓
Return raw token ONCE
```

---

## 295. Flujo token revocation

```text
Current authenticated actor
        ↓
Token Public ID
        ↓
Authorization
        ↓
TokenRevocationService
        ↓
Token = REVOKED
        ↓
cache invalidation
        ↓
audit
```

---

## 296. Flujo JWT key rotation

```text
Signing Key K1
        ↓
tokens issued
        ↓
K2 introduced as current
        ↓
new tokens signed K2
        ↓
K1 remains verification-only
        ↓
K1 tokens expire
        ↓
K1 retired
```

---

## 297. Arquitectura global

```text
                    STATELESS AUTHENTICATION
                             │
                 Explicit Request Credential
                             │
                             ▼
                  Bearer/API Authenticator
                             │
                             ▼
                       Token Credential
                             │
             ┌───────────────┴────────────────┐
             ▼                                ▼
         Opaque Token                    Signed Token
             │                                │
             ▼                                ▼
       Token Repository                   JWT Parser
             │                                │
             ▼                                ▼
       Secret Verification             Signature Verification
             │                                │
             └──────────────┬─────────────────┘
                            ▼
                    Verified Token
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
           Identity       Binding       Abilities
           Mapping        Checks        / Claims
              │             │              │
              └─────────────┼──────────────┘
                            ▼
                 Identity Security State
                            │
                            ▼
                 Authentication Evidence
                            │
                            ▼
                 AuthenticationContext
                            │
                            ▼
                      Authorization
```

---

## 298. Decisiones arquitectónicas

VoltStack adoptará:

```text
1. Bearer transport and token format are separate concepts.
2. Token parsing and token verification are separate phases.
3. Opaque tokens are first-class, not fallback tokens.
4. JWT is optional, not mandatory for APIs.
5. PAT secrets are shown once and stored only as digests.
6. Service identities do not require User models.
7. Token type confusion is explicitly prevented.
8. Token abilities are authenticated data, not final Authorization decisions.
9. Token revocation semantics are explicit for every token profile.
10. Stateless Authentication remains request-scoped under FrankenPHP.
```

---

## 299. Comparación conceptual con Laravel

VoltStack podrá conservar la ergonomía de mecanismos como:

```text
Sanctum Personal Access Tokens
Bearer token authentication
token abilities
simple token issuance/revocation
```

pero no acoplará el sistema a:

```text
User model
HasApiTokens trait
single token table semantics
```

El modelo general será:

```text
Identity
+
TokenCredential
+
TokenVerifier
+
TokenRepository
+
TokenPolicy
+
AuthenticationEvidence
```

---

## 300. Comparación conceptual con Symfony

VoltStack aprovechará ideas presentes en:

```text
AccessTokenAuthenticator
AccessTokenHandler
firewall-specific Authentication
custom token handlers
```

pero formalizará adicionalmente:

```text
opaque token lifecycle
PAT issuance
token type semantics
SecurityVersion
token abilities
machine/service identities
JWT profiles
revocation models
key rotation
persistent-runtime safety
```

---

## 301. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte Bearer Authentication;
2. soporte Authorization header seguro;
3. soporte API-key adapters;
4. soporte opaque tokens;
5. soporte PATs;
6. soporte Service Tokens;
7. soporte Machine Tokens;
8. soporte JWT;
9. separe parsing de verification;
10. soporte TokenType;
11. evite token type confusion;
12. soporte token hashing/digest;
13. nunca almacene opaque raw secrets;
14. muestre raw PAT una sola vez;
15. soporte expiration;
16. soporte revocation;
17. soporte rotation;
18. permita token families;
19. soporte abilities;
20. soporte audiences;
21. soporte purpose binding;
22. soporte tenant binding;
23. soporte firewall binding;
24. soporte Identity binding;
25. soporte SecurityVersion;
26. valide issuer;
27. valide audience;
28. valide temporal claims;
29. controle algorithms;
30. controle `kid`;
31. soporte JWKS cache;
32. soporte external introspection;
33. soporte key rotation;
34. soporte key compromise response;
35. soporte request memoization;
36. sea revocation-aware;
37. sea observable;
38. sea auditable;
39. soporte APIs distribuidas;
40. soporte service-to-service;
41. sea seguro con FrankenPHP;
42. sea fiber-safe;
43. mantenga Token Authentication separado de Authorization.

---

## 302. Regla arquitectónica final

VoltStack deberá preservar:

```text
EXPLICIT TOKEN
      ↓
UNTRUSTED CREDENTIAL
      ↓
TOKEN PROFILE RESOLUTION
      ↓
CRYPTOGRAPHIC / SERVER-SIDE VERIFICATION
      ↓
TYPE + ISSUER + AUDIENCE + TIME + BINDING VALIDATION
      ↓
VERIFIED TOKEN
      ↓
IDENTITY RESOLUTION
      ↓
IDENTITY SECURITY STATE
      ↓
AUTHENTICATION EVIDENCE
      ↓
REQUEST-SCOPED AUTHENTICATION CONTEXT
      ↓
AUTHORIZATION
```

La primera regla central será:

> **Un token es una prueba transportada por la request; no es una Identity ni una decisión de acceso.**

La segunda será:

> **VoltStack no utilizará “JWT” y “Stateless Authentication” como sinónimos. Los tokens opacos, reference tokens, signed tokens y credentials de servicio serán estrategias distintas bajo una misma arquitectura.**

La tercera será:

> **Ninguna claim, scope, subject o role derivado de un token podrá considerarse confiable hasta que el token completo haya superado sus verificaciones criptográficas, semánticas y contextuales.**

---

## 303. Próximo documento

El siguiente documento recomendado será:

```text
15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md
```

Este documento deberá definir una de las capas más importantes de todo Authentication:

```text
Authentication Factors
knowledge factors
possession factors
inherence/external assurance signals
TOTP
HOTP
email OTP boundaries
SMS OTP boundaries
Passkeys as factors
Recovery Codes
factor enrollment
factor verification
FactorRegistry
FactorVerifier
FactorEvidence
multi-factor composition
factor independence
MFA requirements
step-up authentication
freshness
factor reuse
trusted devices
challenge transactions
MFA recovery
factor reset
factor lifecycle
tenant policies
risk-triggered MFA
adaptive MFA
audit
observability
FrankenPHP safety
```

Con ese documento VoltStack podrá dejar de tratar MFA como:

```text
password
+
if ($user->two_factor_secret) ...
```

y convertirlo en un verdadero **Factor Orchestration System** capaz de combinar Password, Passkey, TOTP, federación, dispositivos y futuros métodos de autenticación sin crear pipelines especiales para cada combinación.
