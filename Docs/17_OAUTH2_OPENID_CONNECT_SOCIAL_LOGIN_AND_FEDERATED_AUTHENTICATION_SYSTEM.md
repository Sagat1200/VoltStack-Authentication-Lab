# VoltStack Authentication System

## 17 — OAuth 2.x, OpenID Connect, Social Login and Federated Authentication System

- **Archivo:** `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de autenticación federada mediante OAuth 2.x, OpenID Connect y proveedores externos de identidad

**Depende especialmente de:**

- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`

---

## 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack podrá delegar Authentication a proveedores externos sin delegar ciegamente el control de su modelo interno de Identity, Security State, Authentication Assurance o Authorization.
El sistema deberá cubrir:

- OAuth 2.x Client
- OpenID Connect Relying Party
- Social Login
- Enterprise SSO
- External Identity Providers
- Federated Authentication
- Authorization Code Flow
- PKCE
- state
- nonce
- ID Tokens
- Access Tokens
- Refresh Tokens
- UserInfo
- Provider Discovery
- JWKS
- Federated Identity Mapping
- Account Linking
- Account Unlinking
- Federated Assurance
- Tenant-specific Identity Providers

La arquitectura deberá soportar proveedores como:

- Google
- Microsoft
- GitHub
- Apple
- Okta
- Auth0
- Keycloak
- Entra ID
- enterprise OIDC providers
- custom compliant providers

sin acoplar el Core a ninguno de ellos.

## 2. Principio fundamental

VoltStack deberá distinguir:

```text
OAuth
    authorization delegation protocol

OpenID Connect
    identity/authentication layer over OAuth

Social Login
    product UX built on federation

Federated Authentication
    local authentication established from
    verified external authentication evidence
```

Por tanto:
OAuth por sí solo no es un protocolo de Authentication.

La autenticación federada mediante OIDC deberá basarse en evidencia de identidad definida por el protocolo y validada correctamente, no simplemente en obtener un Access Token.

## 3. OAuth vs OpenID Connect

Modelo conceptual:

```text
OAuth
    Client wants delegated access
    to Resource Server

OpenID Connect
    Client additionally wants
    authenticated subject information
```

## 4. Access Token vs ID Token

No deberán confundirse.

```text
Access Token
    credential for accessing a resource/API

ID Token
    signed authentication assertion
    about an end-user authentication event
```

## 5. Regla crítica

Nunca:

```text
received OAuth access token
    ↓
therefore user authenticated
```

sin un protocolo/provider contract que defina explícitamente ese comportamiento.

## 6. Roles del sistema

VoltStack podrá actuar como:

- OAuth Client
- OIDC Relying Party

Este documento no convierte a VoltStack automáticamente en:

- OAuth Authorization Server
- OpenID Provider

Esos roles pertenecerán a subsistemas separados si se implementan.

## 7. Modelo global

User
↓
VoltStack Application
↓
Federated Login Initiation
↓
Authorization Request
↓
External Identity Provider
↓
User Authentication
↓
Authorization Response
↓
VoltStack Callback
↓
Code Validation
↓
Token Endpoint
↓
ID Token
↓
OIDC Verification
↓
FederatedSubject
↓
Identity Mapping
↓
Identity Security State
↓
Authentication Evidence
↓
AuthenticationContext
↓
Session / Stateless continuation

## 8. Federación no sustituye Identity Model

Aunque el proveedor autentique a:
subject = abc123
VoltStack seguirá trabajando con:

```text
FederatedSubject
    ↓
FederatedIdentityMapping
    ↓
IdentityReference
    ↓
Identity
```

## 9. External subject

La identidad federada deberá usar como clave estable preferida:
issuer + subject
No:

- email
- display name
- username

## 10. OAuthProvider vs FederationConnection

Debe distinguirse:

```text
Provider Definition
    protocol/server metadata

Federation Connection
    configured trust relationship
    used by a VoltStack application/tenant
```

## 11. FederationConnection

Conceptualmente:

```php
final readonly class FederationConnection
{
    public function __construct(
        public FederationConnectionId $id,
        public string $name,
        public FederationProtocol $protocol,
        public TrustedIssuer $issuer,
        public ClientIdentifier $clientId,
        public RedirectUriSet $redirectUris,
        public FederationTrustPolicy $trustPolicy,
        public FederatedIdentityMappingPolicy $mappingPolicy,
    ) {}
}
```

## 12. Connection registry

VoltStack deberá disponer de:
FederationConnectionRegistry

## 13. Registry responsibilities

register connections
resolve by trusted ID
resolve by tenant
resolve by firewall
resolve by provider alias
validate bootstrap configuration
compile discovery metadata

## 14. No client-controlled provider URLs

Nunca construir una conexión desde:

```php
issuer = request input
authorization_endpoint = request input
token_endpoint = request input
```

## 15. Connection trust

Una conexión deberá existir previamente en configuración/administration controlada.

## 16. Social Login

Social Login será una experiencia construida sobre FederationConnection.
Ejemplo:

- Continue with Google
- Continue with Microsoft
- Continue with GitHub

## 17. Provider adapter

Los proveedores con particularidades podrán implementar:
FederationProviderAdapter
pero el dominio principal deberá utilizar estándares siempre que sea posible.

## 18. OIDC first-class

OpenID Connect deberá ser el protocolo de federación principal para Authentication moderna.

## 19. OAuth-only providers

Algunos providers pueden exponer identidad mediante APIs propietarias después de OAuth.
Esto deberá considerarse:

- provider-specific identity federation adapter
- no OIDC.

## 20. Authorization Code Flow

Para browser-based federated Authentication deberá favorecerse:
Authorization Code Flow

## 21. Flujo general

VoltStack
↓
Authorization Endpoint
↓
User authenticates at IdP
↓
redirect with authorization code
↓
VoltStack callback
↓
Token Endpoint
↓
ID Token / Access Token

## 22. Authorization code

El code deberá ser:

- short-lived
- single-use

bound to redirect URI/client
según el proveedor/protocolo.
VoltStack no deberá tratarlo como ID Token.

## 23. PKCE

VoltStack deberá soportar y favorecer:
Proof Key for Code Exchange

## 24. PKCE values

code_verifier
code_challenge
code_challenge_method

## 25. Code verifier

Debe ser:

- high entropy
- random
- transaction-bound
- short-lived
- secret

## 26. PKCE challenge

Preferir:

- S256
- cuando el proveedor lo soporte.

## 27. PKCE no sustituye state

Son protecciones diferentes.

## 28. State parameter

state deberá proteger:

- request/callback correlation
- CSRF/login CSRF
- transaction binding

## 29. State requirements

Debe ser:

- unguessable
- single-use
- short-lived
- transaction-bound

## 30. No application data raw in state

Evitar:
state=user_id=15&redirect=/admin

## 31. State reference

Preferir:
opaque transaction reference

## 32. Redirect destination

El intended destination deberá almacenarse server-side y validarse.

## 33. Open redirect prevention

Nunca aceptar después del callback:

```php
redirect=request.redirect
sin allowlist/internal validation.
```

## 34. Nonce

OIDC Authentication deberá soportar:

- nonce
- para ligar ID Token a la Authentication Transaction.

## 35. State vs Nonce

state
binds authorization request/response

nonce
binds OIDC ID Token
to authentication request

## 36. Nonce validation

Si se envía nonce:

- ID Token nonce
- deberá coincidir exactamente.

## 37. Nonce replay

La transaction completada deberá consumirse.

## 38. FederationAuthenticationTransaction

Será fundamental para flows multi-request.

```php
Conceptualmente:
final readonly class FederationAuthenticationTransaction
{
    public function __construct(
        public FederationTransactionId $id,
        public FederationConnectionId $connection,
        public string $state,
        public ?string $nonce,
        public ?string $pkceVerifier,
        public string $firewall,
        public ?TenantIdentifier $tenant,
        public \DateTimeImmutable $createdAt,
        public \DateTimeImmutable $expiresAt,
    ) {}
}
```

## 39. Transaction repository

Podrá usar:

- Redis
- database
- distributed KV

## 40. Transaction TTL

Deberá ser corto.

## 41. Transaction single-use

Después de completar o fallar de forma terminal:
transaction = consumed

## 42. Authorization Request Builder

Componente:

- AuthorizationRequestBuilder
- deberá construir requests OIDC/OAuth.

## 43. Authorization Request parameters

Podrá incluir:

- client_id
- redirect_uri
- response_type
- scope
- state
- nonce
- code_challenge
- code_challenge_method
- prompt
- max_age
- login_hint
- acr_values
- según protocolo/policy.

## 44. response_type

Para Authorization Code Flow:
code

## 45. OIDC scope

Como mínimo para OIDC:
openid

## 46. Additional scopes

Ejemplos:

- profile
- email

solo cuando realmente sean necesarios.

## 47. Scope minimization

No solicitar:

- contacts
- calendar
- files

solo para iniciar sesión.

## 48. Consent minimization

La federación de Authentication deberá solicitar el mínimo acceso externo necesario.

## 49. prompt

Podrá controlar comportamientos como:

- login
- consent
- none
- select_account
- según provider.

## 50. prompt=none

Es útil para determinados SSO flows, pero deberá manejar respuestas de interacción requerida.

## 51. max_age

Podrá exigir que el IdP haya autenticado al usuario recientemente.

## 52. auth_time

Cuando se utilice max_age, deberá validarse el auth_time conforme al protocolo/provider profile.

## 53. Fresh federated authentication

Esto puede contribuir al FreshAuthenticationRequirement de VoltStack.

## 54. login_hint

Solo es una ayuda de UX.
No establece Identity.

## 55. acr_values

Podrá solicitar assurance específico al IdP.

## 56. Requested assurance is not proof

Solicitar:

```php
acr_values=...
no significa que el IdP realmente lo haya satisfecho.
```

Debe verificarse response claims.

## 57. Authorization endpoint

Debe provenir de metadata confiable.

## 58. Redirect URI

Debe ser:

- pre-registered
- exact
- trusted

## 59. Dynamic redirect URI danger

Nunca permitir que el request elija cualquier callback URL.

## 60. Callback route

Deberá quedar vinculada a:

- FederationConnection
- Firewall
- Application

## 61. Callback parameter handling

Podrá recibir:

- code
- state
- error
- error_description

u otros parámetros definidos.

## 62. Error response

Debe mapearse a un:

- FederatedAuthenticationFailure
- seguro.

## 63. Provider error content

No confiar en error_description para UI sin sanitización.

## 64. State validation first

Antes de utilizar code/provider information:

- state
- deberá verificarse y resolver la transaction.

## 65. Unknown state

Fail closed.

## 66. Expired transaction

Fail closed.

## 67. Used transaction

Debe detectarse replay.

## 68. Authorization Code exchange

Debe realizarse mediante:
TokenEndpointClient

## 69. Token endpoint request

Podrá incluir:

```php
grant_type=authorization_code
code
redirect_uri
client_id
```

client_secret or client authentication
code_verifier
según client profile.

## 70. Client authentication

Métodos posibles:

- client_secret_basic
- client_secret_post
- private_key_jwt
- none/public client
- según provider/profile.

## 71. Client secret protection

Nunca almacenar client secret dentro de transaction/client-side state.

## 72. Secret Provider

Usar:

- FederationClientSecretProvider
- o Secret Management abstraction.

## 73. Client secret rotation

Debe ser soportable.

## 74. Private-key client authentication

Podrá soportarse para perfiles enterprise/high-security.

## 75. Token endpoint trust

La URL deberá provenir únicamente de trusted provider metadata.

## 76. SSRF safety

No enviar requests a URLs derivadas directamente de claims o input del cliente.

## 77. Token Endpoint Response

Podrá contener:

- access_token
- token_type
- expires_in
- refresh_token
- id_token
- scope

## 78. ID Token required

Para OIDC Authentication estándar deberá esperarse:

- id_token
- según flow/profile.

## 79. ID Token verifier

Deberá reutilizar o extender el sistema JWT del documento 14.

## 80. ID Token validation

Como mínimo deberá considerar:

- signature
- issuer
- audience

authorized party where relevant
expiration
issued-at
nonce
token type/profile
auth_time when required

## 81. Issuer validation

Debe coincidir con la conexión confiable.

## 82. Audience

El Client ID de VoltStack deberá estar entre audiencias válidas.

## 83. Multiple audiences

Cuando existan múltiples aud, deberá manejarse la semántica de:

- azp
- según OIDC.

## 84. azp

Authorized Party deberá validarse cuando el protocolo lo exija.

## 85. Subject

sub debe existir y ser estable dentro del issuer.

## 86. Subject length/format

Debe validarse estructuralmente y almacenarse como valor opaco.

## 87. Email

Puede venir como:

- email
- email_verified

## 88. Email is attribute, not federated primary key

La clave seguirá siendo:
issuer + sub

## 89. email_verified

Tampoco implica automáticamente que sea seguro enlazar con una cuenta local existente.

## 90. Email verification semantics

Diferentes providers pueden dar significados distintos.

- VoltStack deberá utilizar:
- ProviderTrustProfile
- para interpretar claims.

## 91. UserInfo Endpoint

Puede utilizarse después de token verification para obtener atributos adicionales.

## 92. UserInfo is optional

No siempre es necesario si ID Token contiene suficiente información.

## 93. UserInfo Access Token

El Access Token deberá ser apropiado para ese endpoint.

## 94. UserInfo subject binding

El sub de UserInfo deberá coincidir con el subject del ID Token cuando el protocolo lo requiera.

## 95. Subject mismatch

Fail closed.

## 96. UserInfo mapper

No copiar todas las claims.
Usar:
FederatedAttributeMapper

## 97. Standard claims

Pueden incluir:

- name
- given_name
- family_name
- preferred_username
- email
- locale
- picture

pero su persistencia/local authority será configurable.

## 98. Provider Discovery

OIDC puede publicar metadata en:
.well-known/openid-configuration

## 99. Discovery

VoltStack podrá soportarla mediante:
OidcDiscoveryProvider

## 100. Discovery trust boundary

La discovery URL deberá derivarse de un issuer previamente confiable/configurado.
No de input arbitrario.

## 101. Metadata fields

Podrán incluir:

- issuer
- authorization_endpoint
- token_endpoint
- userinfo_endpoint
- jwks_uri
- scopes_supported
- response_types_supported
- id_token_signing_alg_values_supported

## 102. Issuer consistency

El issuer devuelto por metadata deberá coincidir exactamente con el trusted issuer esperado conforme a las reglas del protocolo.

## 103. Discovery cache

Debe soportar:

- TTL
- refresh
- validation
- stale policy

## 104. Discovery outage

Una conexión ya configurada puede utilizar metadata cacheada dentro de límites seguros.

## 105. First discovery failure

No deberá inventar endpoints.

## 106. JWKS

Las public keys deberán obtenerse desde:

- jwks_uri
- confiable de metadata validada.

## 107. JWKS cache

Reutilizar conceptos del documento 14:

- bounded refresh
- kid handling
- key rotation
- negative lookup

## 108. Unknown key

Puede disparar un refresh controlado.

## 109. Key rollover

Debe soportarse sin interrumpir Authentication durante rotación normal.

## 110. Provider registry

VoltStack podrá registrar provider templates:

- google
- microsoft
- github
- apple
- generic_oidc

## 111. Provider template

No será la trust connection real.

- Solo define:
- default endpoints/profile quirks
- claim conventions
- supported capabilities

## 112. Connection owns trust

La aplicación/tenant decide:

- client ID
- issuer
- secrets
- redirect URIs
- mapping
- policy

## 113. Generic OIDC provider

Debe ser first-class para evitar depender de adapters propietarios.

## 114. Enterprise federation

Un tenant puede tener una conexión:
ACME Microsoft Entra
otro:
Globex Okta

## 115. TenantFederationConnection

Cada conexión podrá estar ligada a:

- tenant
- organization
- realm

## 116. Home Realm Discovery

Antes de redirigir, el sistema puede decidir qué IdP usar.

## 117. Inputs permitidos

tenant context
verified organization configuration
email domain mapping
subdomain
explicit provider selection from allowlisted UI

## 118. Email domain routing

Ejemplo:

```php
@acme.com
    → ACME Entra connection
```

## 119. Domain mapping must be trusted

No utilizar el dominio del email para construir un issuer dinámicamente.

## 120. Tenant custom IdP

Un tenant podrá configurar OIDC enterprise mediante administración segura.

## 121. Configuration onboarding

Deberá validar:

- issuer
- discovery
- supported flows
- redirect URI
- JWKS
- client configuration

domain ownership where applicable

## 122. Provider connection testing

Tooling administrativo podrá probar:

- metadata fetch
- key fetch
- authorization endpoint
- token endpoint configuration
- sin exponer secrets.

## 123. FederatedSubject

Después de verificar ID Token:

- issuer
- +;
- subject

se convierte en:
FederatedSubject

## 124. Federated mapping

FederatedSubject
↓
FederatedIdentityMapper
↓
FederatedIdentityMapping
↓
IdentityReference

## 125. Existing mapping

Es el caso más seguro y sencillo.

## 126. Unknown mapping

Puede producir:

- NOT_MAPPED
- AUTO_PROVISION_REQUIRED
- LINK_REQUIRED
- DENY
- según policy.

## 127. JIT Provisioning

VoltStack podrá crear Identity en primer login si la conexión lo permite.

## 128. JIT policy

Podrá exigir:

- trusted issuer
- allowed tenant
- email verified
- allowed domain
- invitation
- group/claim condition
- organization match

## 129. Auto-provision transaction

Debe crear atómicamente cuando sea posible:

- Identity
- +;
- FederatedIdentityMapping

## 130. No auto-provision from arbitrary provider

Obligatorio.

## 131. Identity profile mapping

Podrán mapearse:

- displayName
- email
- avatar
- locale
- según authority policy.

## 132. Attribute authority

Ejemplo:

```text
corporate employee display name
    external authoritative

application preferences
    local authoritative
```

## 133. Attribute refresh

Puede ejecutarse:

- ON_LOGIN
- NEVER
- SELECTIVE
- AUTHORITATIVE

## 134. Account linking

Conectar una cuenta federada a una Identity existente será una operación sensible.

## 135. Unsafe linking

Nunca:

```text
external email == local email
    ↓
automatic link
como regla universal.
```

## 136. Safe linking

Ejemplo:

```text
existing authenticated local Identity
        +
fresh authentication
        +
successful federated authentication
        +
explicit user confirmation
        ↓
create link
```

## 137. Linking without existing session

Puede permitirse mediante flows de account ownership suficientemente fuertes, pero deberá ser policy explícita.

## 138. Email-based suggested linking

VoltStack puede decir:

- An account with this email may already exist.
- Sign in to link it.
- sin realizar auto-link.

## 139. Link conflict

Si:
issuer + subject
ya pertenece a otra Identity:
CONFLICT

## 140. Federated account takeover

Un mapping incorrecto puede permitir tomar control de una cuenta local.
Por ello linking será tratado como Credential Lifecycle sensible.

## 141. Account unlinking

También deberá ser explícito.

## 142. Unlink policy

Antes de eliminar:

- ensure another viable authentication method exists
- si la aplicación quiere evitar lockout.

## 143. Federation-only Identity

Una cuenta puede tener únicamente:

- OIDC connection
- sin password local.

## 144. Unlink last method

Puede rechazarse o derivar a recovery/enrollment.

## 145. Linking events

FederatedIdentityLinked
FederatedIdentityUnlinked
FederatedIdentityLinkRejected

## 146. Authentication Evidence

Una federated Authentication exitosa deberá generar:
FederatedAuthenticationEvidence

## 147. Evidence fields

Podrá contener:

- connection reference
- issuer
- subject reference
- authentication time
- federated assurance
- AMR
- ACR
- provider trust profile
- verified claims summary

sin almacenar raw ID Token.

## 148. Raw tokens and assertions

No deberán entrar al AuthenticationContext.

## 149. ID Token storage

Normalmente no será necesario persistir el ID Token completo después de completar login.

## 150. Access Token storage

Solo debe persistirse si la aplicación necesita llamar APIs del proveedor después del login.
Eso pertenece a:

- External Access Token Management
- no necesariamente al estado de Authentication principal.

## 151. Refresh Tokens

Igualmente:

- external provider credential
- separado de AuthenticationSession.

## 152. Token vault

Si se necesitan Access/Refresh Tokens, deberán almacenarse en un:

- FederatedTokenVault
- o subsistema de secrets/connected accounts.

## 153. Never put provider refresh token in browser session payload

Debe permanecer server-side y protegido.

## 154. Social Login without API access

Caso simple:

```text
OIDC login
    ↓
verify Identity
    ↓
```

discard provider tokens after required verification
cuando no se necesiten APIs externas.

## 155. Provider API scopes

No solicitar scopes de API solo por querer login.

## 156. offline_access

Solo cuando realmente se necesita refresh token.

## 157. Refresh token security

Debe tratarse como long-lived high-value credential.

## 158. Refresh token compromise

Puede no comprometer directamente la password local, pero sí el connected account.

## 159. Provider token lifecycle

Podrá ser subsistema separado futuro.

## 160. Federated Assurance

El proveedor puede informar cómo autenticó al subject.

## 161. acr

Puede indicar Authentication Context Class.

## 162. amr

Puede indicar Authentication Methods References.

## 163. Trust mapping

Nunca:

```text
amr contains "mfa"
    ↓
VoltStack MFA=true
sin issuer-specific policy.
```

## 164. FederatedAssuranceMapper

Contrato:

```php
interface FederatedAssuranceMapperInterface
{
    public function map(
        FederationConnection $connection,
        VerifiedOidcAuthentication $authentication
    ): AuthenticationAssuranceContribution;
}
```

## 165. Provider-specific trust

Ejemplo:

```php
Connection ACME-Entra
acr=company-high
    → phishing_resistant + AAL2
```

si la organización controla semántica.

## 166. Unknown ACR

No asignar assurance elevado.

## 167. AMR double-counting

El assertion OIDC representa un único provenance graph.

- No crear artificialmente:
- OIDC factor
- +;
- password factor
- +;
- OTP factor

como tres proofs independientes si solo tenemos una assertion que afirma los dos últimos.

## 168. FederatedFactorEvidence

Puede representar:
External Authentication
properties:

```text
        multi_factor
        password
        otp
```

sin fingir que VoltStack verificó directamente cada credential.

## 169. Freshness

auth_time puede contribuir a:

- FreshAuthentication
- si es validado y trusted.

## 170. max_age

Puede forzar nueva Authentication en IdP.

## 171. Local step-up after federation

Incluso si IdP autenticó, VoltStack puede requerir factor local adicional.

## 172. Example

OIDC AAL1
↓
Admin route requires phishing-resistant AAL2
↓
local Passkey Step-Up

## 173. Federated MFA vs Local MFA

Pueden combinarse dentro del mismo Assurance system.

## 174. Provider selection

Una aplicación puede tener:

- Google
- Microsoft
- Corporate OIDC
- simultáneamente.

## 175. Explicit selection

UI deberá usar:

- provider public identifier
- que mapea a connection confiable.

## 176. No arbitrary provider callback

Callback deberá saber qué connection originó transaction mediante state.

## 177. Mix-Up Attack

OAuth/OIDC pueden sufrir ataques donde respuestas de un Authorization Server se confunden con otro.
VoltStack deberá prevenirlos mediante binding estricto de transaction a:

- issuer
- authorization endpoint
- token endpoint
- client
- redirect URI

## 178. Provider mix-up prevention

La callback no debe inferir provider desde parámetros no confiables.

## 179. Transaction chooses provider

state
↓
FederationTransaction
↓
connection
No:
callback.provider query param

## 180. Authorization Server mix-up

Todos los endpoints utilizados deben provenir de la misma trust configuration validada.

## 181. Issuer identifier in authorization response

Si perfiles modernos lo proporcionan, puede validarse como defensa adicional.

## 182. Confused Deputy

VoltStack deberá impedir que tokens/responses destinados a otro client/application se utilicen aquí.

## 183. Defensas

audience
client ID
redirect URI
issuer
nonce
transaction binding
token type

## 184. Login CSRF

Un atacante podría intentar iniciar una federated session con su propia cuenta en el navegador de la víctima.
state y transaction binding son fundamentales.

## 185. Session fixation

Después de federated Authentication exitosa:

- rotate session ID
- igual que con Password/Passkey.

## 186. Existing authenticated session

Un federated callback puede representar:

- login
- account linking
- step-up
- provider connection

Debe estar ligado a un FederationPurpose.

## 187. FederationPurpose

Ejemplos:

- LOGIN
- LINK_ACCOUNT
- STEP_UP
- REAUTHENTICATE
- CONNECT_PROVIDER_API

## 188. Purpose binding

Un callback iniciado para:
CONNECT_PROVIDER_API
no deberá convertirse accidentalmente en:
LOGIN

## 189. Linking flow transaction

Debe almacenar:

```php
base Identity
base AuthenticationContext
purpose = LINK_ACCOUNT
```

## 190. Identity mismatch during linking

El external subject no debe estar ya vinculado a otra Identity.

## 191. Step-up federation

Una organización puede permitir:
Reauthenticate with Corporate SSO

## 192. Step-up max_age

Puede solicitar fresh IdP login.

## 193. Step-up completion

Debe extender Evidence del Context existente.

## 194. Session created after normal federated login

Una vez completada Authentication:

```text
AuthenticationContext
    ↓
AuthenticationSession
según el Firewall.
```

## 195. Stateless federated login

En APIs, federation suele terminar en emisión de una credential/token específica.
Eso deberá ser un flow explícito distinto del browser login.

## 196. No provider access token as VoltStack session

Evitar tratar el Access Token del IdP como session ID local.

## 197. Identity Security State

Aunque OIDC sea válido:
Local Identity DISABLED
debe resultar:
Authentication denied

## 198. Tenant Security State

También.

## 199. SecurityVersion

Puede invalidar sessions locales creadas desde federation.

## 200. External account deactivation

VoltStack puede enterarse:

- on next login
- SCIM
- provider webhook
- directory sync
- introspection
- dependiendo de integración.

## 201. No assumption of instant federation deprovisioning

Un OIDC ID Token verificado una vez no informa necesariamente futuros cambios del IdP.

## 202. Enterprise deprovisioning

Idealmente se integra con:

- SCIM
- directory sync
- identity lifecycle provisioning
- como subsistema futuro.

## 203. Single Sign-On

Federated login habilita SSO porque el IdP puede conservar su propia AuthenticationSession.

## 204. VoltStack SSO session

Aun así, VoltStack crea su propio:
AuthenticationSession

## 205. Two independent sessions

IdP session
VoltStack session

## 206. Local logout

Cerrar sesión en VoltStack no necesariamente termina sesión del IdP.

## 207. Single Logout boundary

OIDC/SAML logout federation es un problema separado y provider-dependent.

## 208. Logout modes

Podrán existir:

- LOCAL_ONLY
- PROVIDER_LOGOUT_ATTEMPT
- FEDERATED_LOGOUT

si el provider/profile lo soporta.

## 209. Default recommendation

Logout local deberá ser siempre posible sin depender de disponibilidad del IdP.

## 210. RP-initiated logout

Podrá soportarse donde el proveedor implemente estándares compatibles.

## 211. id_token_hint

Si se usa, debe provenir de state confiable, no del request arbitrario.

## 212. Post-logout redirect

Debe estar preconfigurado/validado.

## 213. Provider logout failure

No deberá impedir destruir la AuthenticationSession local.

## 214. Account switching

El usuario puede querer:
switch external account

## 215. prompt=select_account

Puede utilizarse cuando el provider lo soporte.

## 216. No implicit account switch attack

Una existing local session no deberá cambiar de Identity solo porque callback trae otro external subject, salvo un nuevo Authentication flow explícito.

## 217. Session identity mismatch

Si login flow comenzó mientras otra Identity ya estaba autenticada, policy deberá decidir:

- replace session
- reject
- link flow
- de forma explícita.

## 218. Enterprise tenant routing

El IdP puede emitir organization/tenant claims.

## 219. External organization mapping

Usar:
ExternalOrganizationMapper

## 220. Do not trust tenant claim directly

Nunca:

```php
tenant = idToken['org']
sin mapping local.
```

## 221. Organization mapping

external org id
↓
trusted mapping
↓
TenantIdentifier

## 222. Tenant mismatch

Si transaction era para Tenant A y assertion corresponde a external org mapped to Tenant B:

- reject
- salvo cross-tenant policy explícita.

## 223. Group Claims

Enterprise IdPs pueden proporcionar:

- groups
- roles
- directory memberships

## 224. Authentication boundary

Estas son authenticated claims, pero no Roles de VoltStack automáticamente.

## 225. Authorization mapper

Un sistema separado podrá convertir:

```text
external group
    ↓
local authorization relation
```

## 226. No admin by claim alone

Nunca:

```text
groups contains admin
    ↓
VoltStack super admin
```

sin trusted mapping policy.

## 227. Claim overage

Algunos providers no incluyen todos los grupos y requieren API lookup.
El Auth core no deberá depender de grupos completos para identidad básica.

## 228. Claims normalization

Usar:

- FederatedClaimNormalizer
- por provider/profile.

## 229. Claim schema

Debe declarar:

- required claims
- optional claims
- mapped claims
- ignored claims

## 230. Unknown claims

Ignorar o conservar solo en diagnostic scope controlado.

## 231. Privacy

No persistir todo el ID Token/UserInfo “por si acaso”.

## 232. Data minimization

Almacenar únicamente atributos necesarios.

## 233. Provider picture/avatar URLs

Son external URLs y no deben tratarse como trusted content sin políticas frontend/security apropiadas.

## 234. Provider username

Puede cambiar.
No usarlo como federated primary key.

## 235. Provider email change

No rompe mapping basado en issuer + sub.

## 236. Subject change

No deberá mapearse automáticamente por email.

## 237. Subject collision

Dentro del mismo issuer debe existir mapping único.

## 238. Connection migration

Si se cambia issuer, no asumir que los sub mantienen semántica.

## 239. Alias issuers

Solo soportar si el provider/protocolo lo define y la conexión lo configura explícitamente.

## 240. Account consolidation

Fusionar mappings de distintos issuers es una operación de Identity management.

## 241. FederationConnectionStatus

Estados:

- ACTIVE
- DISABLED
- MISCONFIGURED
- DEGRADED
- REVOKED

## 242. Disabled connection

No debe aceptar nuevos logins.

## 243. Existing local sessions

No necesariamente se invalidan por deshabilitar un provider.
Policy puede decidir.

## 244. Provider outage

Debe distinguirse:

- USER_DENIED
- INVALID_RESPONSE
- PROVIDER_UNAVAILABLE
- TOKEN_ENDPOINT_ERROR
- JWKS_UNAVAILABLE
- DISCOVERY_UNAVAILABLE

## 245. External failure mapping

El usuario podrá recibir un error seguro como:
Unable to sign in with this provider.

## 246. No automatic fallback

Si usuario eligió:
Corporate SSO
y falla:

- do not automatically authenticate by local password
- sin nueva acción explícita.

## 247. Break-glass access

Puede existir un Authenticator local separado para administración de emergencia.
Debe ser policy explícita.

## 248. Provider outage fallback risk

Un fallback automático puede destruir el boundary empresarial de SSO.

## 249. Token endpoint retries

Podrán existir retries limitados solo cuando sea seguro.

## 250. Authorization code replay

No reintentar indiscriminadamente un code que puede ser single-use si ya existe posibilidad de consumo.

## 251. Idempotency

Callback processing deberá manejar doble envío de browser de forma segura.

## 252. Consumed transaction

Segundo callback:
REPLAYED_TRANSACTION

## 253. Network failure after code exchange

Puede existir situación donde token endpoint consumió code pero VoltStack perdió response.
El flow probablemente requerirá restart, no reusar code indefinidamente.

## 254. Callback operation boundaries

No crear local session antes de completar:

- ID Token verification
- Identity mapping
- Identity Security State
- Assurance evaluation
- commit-time validation

## 255. Commit sequence

callback
↓
state verified
↓
code exchanged
↓
ID Token verified
↓
FederatedSubject resolved
↓
Identity mapped
↓
Identity state validated
↓
Assurance calculated
↓
AuthenticationContext created
↓
session ID rotated
↓
AuthenticationSession persisted

## 256. Session persistence failure

No dejar request parcialmente autenticada.

## 257. FederationAuthenticator

Será un Authenticator estándar.

## 258. OidcAuthenticator

Podrá ser implementación especializada:

```php
final class OidcAuthenticator implements AuthenticatorInterface
{
}
```

## 259. Responsibilities

recognize callback operation
resolve FederationTransaction
delegate code exchange
delegate ID Token verification
create FederatedSubject
delegate Identity mapping
create Evidence

## 260. No crypto inside Authenticator

JWT/JWS validation deberá delegarse.

## 261. FederationManager

Podrá orquestar conexiones y flows.

## 262. FederationManager responsibilities

begin login
begin linking
begin step-up
resolve connection
complete callback

## 263. No God Object

Delegará a:

- ConnectionRegistry
- TransactionRepository
- AuthorizationRequestBuilder
- TokenEndpointClient
- OidcVerifier
- IdentityMapper
- AssuranceMapper

## 264. Provider-specific adapters

Ejemplo:

- GoogleFederationAdapter
- MicrosoftFederationAdapter
- GitHubOAuthIdentityAdapter
- AppleFederationAdapter
- GenericOidcAdapter

## 265. Adapter limitations

Un adapter no podrá saltarse las security invariants del Core.

## 266. GitHub-style OAuth login

Si un provider no usa OIDC para ese producto:

```text
Authorization Code
    ↓
Access Token
    ↓
Provider User API
    ↓
```

provider-specific verified external subject

## 267. OAuth Identity Adapter

Deberá declarar cómo obtiene una identidad estable y qué trust semantics tiene.

## 268. Access Token source

Solo usar Access Token obtenido directamente mediante el code exchange de la transaction.

## 269. User endpoint

URL preconfigurada.

## 270. Identity response

Debe validarse estructuralmente.

## 271. External numeric IDs

Podrán utilizarse como subject opaco bajo el issuer/provider namespace.

## 272. Provider email

Sigue sin ser primary mapping key.

## 273. OAuth provider identity evidence

Debe marcarse como:

- provider_specific_federation
- para no fingir OIDC semantics.

## 274. Security profile

Cada adapter deberá declarar su assurance/trust mapping.

## 275. OAuth Access Token verification

Si solo se utiliza contra el provider User API, no es necesario que VoltStack pueda verificarlo localmente.

## 276. Token confidentiality

Sigue siendo sensible durante el flow.

## 277. Temporary Access Token lifetime

No mantenerlo más tiempo de lo necesario si no se usará después.

## 278. Login providers UI

Puede listar conexiones:

- Google
- Microsoft
- Corporate SSO
- basadas en FederationConnectionRegistry.

## 279. Public Connection ID

El frontend utiliza un ID seguro:

- google
- corporate-acme

que resuelve a configuración server-side.

## 280. Provider logo/name

Solo UX.

## 281. Connection enumeration

No mostrar conexiones privadas de un tenant a usuarios de otros tenants.

## 282. Enterprise discovery privacy

Una aplicación puede decidir no revelar públicamente nombres de IdPs corporativos antes de conocer tenant/domain.

## 283. Rate limiting

Debe aplicarse a:

- federation initiation
- callback failures
- unknown states
- link attempts
- JIT provisioning

## 284. Authorization endpoint redirect abuse

No permitir generar infinite transactions sin límites.

## 285. Transaction store quotas

Podrán ser por:

- IP
- session
- tenant
- connection

## 286. PKCE verifier handling

Nunca loguearlo.

## 287. state logging

Aunque no sea credential principal, tratarlo como ephemeral security token y evitar logs completos.

## 288. Nonce logging

Igualmente.

## 289. Authorization code logging

Nunca.

## 290. ID Token logging

Nunca completo.

## 291. Access Token logging

Nunca.

## 292. Refresh Token logging

Nunca.

## 293. Client secret logging

Nunca.

## 294. Redaction

HTTP client instrumentation deberá ocultar campos sensibles.

## 295. Observability spans

Ejemplos:

- auth.federation.begin
- auth.federation.callback
- auth.oidc.discovery
- auth.oidc.jwks
- auth.oidc.token_exchange
- auth.oidc.id_token.verify
- auth.oidc.userinfo
- auth.federation.identity_map
- auth.federation.link
- auth.federation.provision

## 296. Metrics

auth_federation_started_total
auth_federation_success_total
auth_federation_failure_total
auth_federation_state_failure_total
auth_federation_token_exchange_failure_total
auth_federation_mapping_total
auth_federation_jit_total
auth_federation_link_total
auth_oidc_verification_latency

## 297. Safe labels

connection
provider_type
firewall
result
failure_category
si connection cardinality es controlada.

## 298. Avoid labels

subject
email
authorization code
state
tenant arbitrary id

## 299. Audit events

Especialmente:

- FederatedLoginSucceeded
- FederatedLoginRejected
- FederatedIdentityProvisioned
- FederatedIdentityLinked
- FederatedIdentityUnlinked
- FederationConnectionDisabled
- FederationMappingConflict
- FederatedAssuranceApplied

## 300. High-volume success

Cada federated login sí es un security event razonablemente importante y puede auditarse según deployment.

## 301. Provider failure monitoring

Podrá alertar por:

- JWKS failure spike
- token endpoint failures
- invalid state spike
- issuer mismatch
- signature failures

## 302. Mix-up detection metrics

Registrar categorías específicas.

## 303. Testing — Authorization initiation

Casos:

- valid connection
- unknown connection
- disabled connection
- tenant mismatch
- PKCE generated
- state generated
- nonce generated
- redirect URI exact
- scope minimization

## 304. Testing — callback

valid state
missing state
wrong state
expired state
replayed state
provider error
missing code

## 305. Testing — token exchange

success
provider timeout
invalid grant
client auth failure
malformed response
missing ID token

## 306. Testing — ID Token

valid
bad signature
wrong issuer
wrong audience
wrong azp
expired
future iat beyond tolerance
nonce mismatch
missing sub
wrong algorithm
unknown kid

## 307. Testing — UserInfo

matching subject
subject mismatch
malformed response
provider unavailable

## 308. Testing — Mapping

existing mapping
unknown mapping
JIT allowed
JIT denied
mapping conflict
duplicate subject
tenant mismatch

## 309. Testing — Linking

authenticated local user
fresh auth
successful external auth
link
existing link conflict
attempted email auto-link
unlink
last-method protection

## 310. Testing — Mix-Up

start with Provider A
callback attempts Provider B endpoints
issuer mismatch
token from Provider B
Debe fallar.

## 311. Testing — Open Redirect

Validar:

- external return URL rejected
- protocol-relative URL rejected
- untrusted host rejected

## 312. Testing — Multi-Tenant

Tenant A connection
Tenant B connection
cross-tenant callback
domain routing
external org mapping

## 313. Testing — Assurance

trusted acr
unknown acr
amr mfa
fresh auth_time
stale auth_time
local step-up after federated login

## 314. Testing — Provider outage

discovery unavailable
JWKS unavailable
token endpoint unavailable
userinfo unavailable

## 315. Testing — Persistent Runtime

Request A:

- Connection Google
- Identity Alice

Request B:

- Connection Enterprise
- Identity Bob
- No state sharing.

## 316. Fuzz testing

Especialmente:

- callback query parser
- ID Token parser
- provider JSON
- discovery metadata
- JWKS
- UserInfo responses

## 317. Property-based testing

Útil para:

- state uniqueness
- transaction lifecycle
- issuer normalization
- redirect URI matching
- mapping uniqueness

## 318. Security invariant — OAuth/OIDC transaction

AUTH-FED-TXN-01
Every federation flow has an explicit server-side transaction.
AUTH-FED-TXN-02
state is unpredictable, short-lived and single-use.

- AUTH-FED-TXN-03
- PKCE verifier is transaction-bound.
- AUTH-FED-TXN-04

OIDC nonce is transaction-bound and verified.

- AUTH-FED-TXN-05
- Callback cannot choose the provider/connection.
- AUTH-FED-TXN-06

Transaction purpose cannot change during callback.
AUTH-FED-TXN-07
Expired or consumed transactions fail closed.

## 319. Security invariant — Provider Trust

AUTH-FED-TRUST-01
Providers are selected from trusted configuration.

- AUTH-FED-TRUST-02
- Dynamic arbitrary issuer discovery is prohibited by default.
- AUTH-FED-TRUST-03

Discovery metadata issuer is verified.

- AUTH-FED-TRUST-04
- Authorization, token and JWKS endpoints belong to trusted metadata.
- AUTH-FED-TRUST-05

External redirects are not constructed from arbitrary claims/input.
AUTH-FED-TRUST-06
Provider adapters cannot weaken Core trust rules.

## 320. Security invariant — ID Token

AUTH-OIDC-ID-01
ID Token signature is verified before claims are trusted.

- AUTH-OIDC-ID-02
- Issuer is verified.
- AUTH-OIDC-ID-03
- Audience is verified.
- AUTH-OIDC-ID-04

Authorized party is verified when required.

- AUTH-OIDC-ID-05
- Expiration is verified.
- AUTH-OIDC-ID-06

Nonce is verified when used.

- AUTH-OIDC-ID-07
- Subject is required and treated as opaque.
- AUTH-OIDC-ID-08

ID Token cannot be confused with Access/Refresh Tokens.

## 321. Security invariant — Mapping

AUTH-FED-MAP-01
Federated identity key is based on issuer + subject.

- AUTH-FED-MAP-02
- Email is not the default federated primary key.
- AUTH-FED-MAP-03

Automatic linking by email is prohibited by default.

- AUTH-FED-MAP-04
- Mapping conflicts fail closed.
- AUTH-FED-MAP-05

JIT provisioning is policy-controlled.

- AUTH-FED-MAP-06
- Tenant mappings are explicit.
- AUTH-FED-MAP-07

Group claims do not automatically become local roles.

## 322. Security invariant — Assurance

AUTH-FED-ASSURANCE-01
Federated assurance is provider/connection mapped.

- AUTH-FED-ASSURANCE-02
- Unknown acr/amr values do not increase assurance.
- AUTH-FED-ASSURANCE-03

External evidence is not double-counted.

- AUTH-FED-ASSURANCE-04
- Freshness depends on trusted authentication timestamps/policies.
- AUTH-FED-ASSURANCE-05

Local step-up can augment federated Authentication.

## 323. Security invariant — Secrets

AUTH-FED-SECRET-01
Authorization codes are never logged.

- AUTH-FED-SECRET-02
- Access Tokens are never logged.
- AUTH-FED-SECRET-03

Refresh Tokens are never logged.

- AUTH-FED-SECRET-04
- PKCE verifiers are never logged.
- AUTH-FED-SECRET-05

Client secrets are never exposed through runtime domain objects unnecessarily.
AUTH-FED-SECRET-06
Full ID Tokens are not persisted as AuthenticationContext state.

## 324. Security invariant — Runtime

AUTH-FED-RT-01
Federation transaction state is never process-global.

- AUTH-FED-RT-02
- Shared connection/providers are immutable or stateless.
- AUTH-FED-RT-03

Request-local current provider/Identity state is isolated.

- AUTH-FED-RT-04
- Concurrent federation flows cannot share state.
- AUTH-FED-RT-05

Distributed workers use authoritative shared transaction state.

## 325. Anti-pattern — OAuth token equals login

Incorrecto.

## 326. Anti-pattern — email equals federated identity

Incorrecto como default.

## 327. Anti-pattern — arbitrary issuer input

Riesgo de SSRF y trust injection.

## 328. Anti-pattern — callback provider query param

No debe controlar qué connection valida el response.

## 329. Anti-pattern — missing state

Nunca aceptar callback sin transaction correlation.

## 330. Anti-pattern — state contains local user ID

No como diseño base.

## 331. Anti-pattern — state alone replaces nonce

No son equivalentes.

## 332. Anti-pattern — ID Token used as API Access Token

Token type confusion.

## 333. Anti-pattern — Access Token used as ID Token

Igualmente.

## 334. Anti-pattern — claims read before signature verification

Nunca.

## 335. Anti-pattern — provider groups become admin role

Nunca sin Authorization mapping explícito.

## 336. Anti-pattern — auto-link verified email

Incluso email_verified=true no basta universalmente.

## 337. Anti-pattern — refresh token in browser session

No.

## 338. Anti-pattern — unlimited social scopes

Authentication debe solicitar mínimo acceso.

## 339. Anti-pattern — SSO provider outage triggers password fallback

No automáticamente.

## 340. Anti-pattern — local logout depends on provider availability

Nunca.

## 341. Anti-pattern — shared mutable current connection

Crítico bajo FrankenPHP.

## 342. Anti-pattern — arbitrary post-login redirect

Open redirect.

## 343. Componentes principales

FederationManager
FederationConnection
FederationConnectionId
FederationConnectionRegistry
FederationConnectionResolver
FederationTrustPolicy

FederationAuthenticationTransaction
FederationTransactionRepository
FederationPurpose

AuthorizationRequestBuilder
FederationCallbackHandler

OidcAuthenticator
OauthFederatedAuthenticator

## 344. Componentes OAuth/OIDC

OAuthAuthorizationRequest
OAuthAuthorizationResponse
PkceGenerator
StateGenerator
NonceGenerator

TokenEndpointClient
OidcDiscoveryProvider
OidcProviderMetadata
OidcIdTokenVerifier
OidcUserInfoClient

## 345. Componentes Identity

FederatedSubject
FederatedIdentityMapper
FederatedIdentityMapping
FederatedIdentityProvisioner
FederatedAttributeMapper
ExternalOrganizationMapper

## 346. Componentes Assurance

FederatedAssuranceMapper
FederatedAuthenticationEvidence
FederatedFactorEvidence
ProviderTrustProfile

## 347. Componentes de secretos externos

FederationClientSecretProvider
FederatedTokenVault
ExternalAccessTokenRecord
ExternalRefreshTokenRecord
estos últimos podrán vivir en otro paquete si la arquitectura final lo prefiere.

## 348. Namespace sugerido

VoltStack\Quantum\Auth\Federation
VoltStack\Quantum\Auth\Federation\Contracts
VoltStack\Quantum\Auth\Federation\Connection
VoltStack\Quantum\Auth\Federation\Transaction
VoltStack\Quantum\Auth\Federation\OAuth
VoltStack\Quantum\Auth\Federation\Oidc
VoltStack\Quantum\Auth\Federation\Provider
VoltStack\Quantum\Auth\Federation\Mapping
VoltStack\Quantum\Auth\Federation\Provisioning
VoltStack\Quantum\Auth\Federation\Assurance
VoltStack\Quantum\Auth\Federation\Token

## 349. Estructura sugerida

src/Quantum/Auth/Federation/
├── Contracts/
│   ├── FederationConnectionResolverInterface.php
│   ├── FederationTransactionRepositoryInterface.php
│   ├── FederatedIdentityMapperInterface.php
│   └── FederatedAssuranceMapperInterface.php
│
├── Connection/
│   ├── FederationConnection.php
│   ├── FederationConnectionId.php
│   ├── FederationConnectionRegistry.php
│   ├── FederationConnectionResolver.php
│   └── FederationTrustPolicy.php
│
├── Transaction/
│   ├── FederationAuthenticationTransaction.php
│   ├── FederationTransactionId.php
│   ├── FederationPurpose.php
│   └── FederationTransactionRepository.php
│
├── OAuth/
│   ├── AuthorizationRequestBuilder.php
│   ├── OAuthAuthorizationRequest.php
│   ├── OAuthCallbackResponse.php
│   ├── PkceGenerator.php
│   ├── StateGenerator.php
│   └── TokenEndpointClient.php
│
├── Oidc/
│   ├── OidcAuthenticator.php
│   ├── OidcDiscoveryProvider.php
│   ├── OidcProviderMetadata.php
│   ├── OidcIdTokenVerifier.php
│   ├── OidcUserInfoClient.php
│   └── NonceGenerator.php
│
├── Mapping/
│   ├── FederatedSubject.php
│   ├── FederatedIdentityMapping.php
│   ├── FederatedIdentityMapper.php
│   ├── FederatedAttributeMapper.php
│   └── ExternalOrganizationMapper.php
│
├── Provisioning/
│   ├── FederatedIdentityProvisioner.php
│   ├── FederatedProvisioningPolicy.php
│   └── FederatedProvisioningResult.php
│
├── Assurance/
│   ├── FederatedAuthenticationEvidence.php
│   ├── FederatedAssuranceMapper.php
│   ├── FederatedFactorEvidence.php
│   └── ProviderTrustProfile.php
│
└── Provider/
├── GenericOidcProviderAdapter.php
├── GoogleFederationAdapter.php
├── MicrosoftFederationAdapter.php
├── GitHubFederationAdapter.php
└── AppleFederationAdapter.php

## 350. Configuración conceptual — OIDC

return [

'authentication' => [

'federation' => [

'connections' => [

'corporate' => [
'protocol' => 'oidc',
'issuer' => 'https://identity.example.com',
'client_id' => env('CORPORATE_OIDC_CLIENT_ID'),
'client_secret' => env('CORPORATE_OIDC_CLIENT_SECRET'),

'redirect_uri' => '/auth/federation/corporate/callback',

'scopes' => [
'openid',
'profile',
'email',
],

'pkce' => true,

'mapping' => [
'mode' => 'federated_subject',
'jit' => true,
],
],

],

],

],

];

## 351. Configuración conceptual — Social Providers

'connections' => [

'google' => [
'provider' => 'google',
'client_id' => env('GOOGLE_CLIENT_ID'),
'client_secret' => env('GOOGLE_CLIENT_SECRET'),
'scopes' => [
'openid',
'profile',
'email',
],
],

'microsoft' => [
'provider' => 'microsoft',
'client_id' => env('MICROSOFT_CLIENT_ID'),
'client_secret' => env('MICROSOFT_CLIENT_SECRET'),
],

];

## 352. Configuración tenant enterprise

'tenants' => [

'federation' => [
'enabled' => true,

'allow_custom_oidc' => true,

'home_realm_discovery' => true,

'jit_provisioning' => false,
],

];

## 353. Flujo OIDC completo

User clicks "Corporate SSO"
↓
FederationConnectionRegistry
↓
Connection = Corporate
↓
Create FederationTransaction
├── state
├── nonce
├── PKCE verifier
├── tenant
└── purpose=LOGIN
↓
Build Authorization Request
↓
Redirect to IdP
↓
IdP authenticates user
↓
Callback code + state
↓
Validate state
↓
Resolve transaction/connection
↓
Exchange code + PKCE verifier
↓
Receive ID Token
↓
Verify signature
↓
Verify issuer
↓
Verify audience/azp
↓
Verify expiration
↓
Verify nonce
↓
Create FederatedSubject
↓
FederatedIdentityMapper
↓
IdentityReference
↓
Identity Security State
↓
FederatedAssuranceMapper
↓
AuthenticationEvidence
↓
AuthenticationContext
↓
Rotate Session ID
↓
AuthenticationSession

## 354. Flujo JIT

Verified FederatedSubject
↓
No existing mapping
↓
FederatedProvisioningPolicy
↓
Issuer trusted?
Tenant allowed?
Claims acceptable?
Invitation/domain policy?
↓
YES
↓
Create local Identity
↓
Create mapping issuer+sub
↓
Identity Security State
↓
AuthenticationContext

## 355. Flujo linking seguro

Alice already authenticated locally
↓
Fresh Authentication
↓
Select "Link Microsoft"
↓
FederationTransaction:

```php
    purpose = LINK_ACCOUNT
    base Identity = Alice
        ↓
Microsoft Authentication
        ↓
Verified FederatedSubject
        ↓
Check mapping conflict
        ↓
Explicit Confirmation
        ↓
Create mapping
        ↓
Audit
```

## 356. Flujo inseguro prohibido

Microsoft says:

```php
    email = <alice@example.com>
        ↓
```

Local Alice has same email
↓
AUTOMATIC LINK
No deberá ser default.

## 357. Flujo federated Step-Up

Current session:

```text
    AAL1
        ↓
```

Sensitive action requires:

```text
    fresh corporate authentication
        ↓
```

FederationTransaction:

```php
    purpose = STEP_UP
    max_age = 0 / policy value
        ↓
Corporate IdP
        ↓
```

Verified ID Token + auth_time
↓
Federated Assurance Mapping
↓
Evidence extended
↓
AuthenticationContext upgraded

## 358. Flujo wrong issuer

Transaction:

```php
    Connection A
    issuer = <https://idp-a.example>

Callback code exchanged
        ↓
```

ID Token issuer =
<https://idp-b.example>
↓
WRONG_ISSUER
↓
Authentication rejected

## 359. Flujo mix-up

Login initiated at Provider A
↓
Attacker attempts response from Provider B
↓
state resolves Connection A
↓
token/provider validation mismatch
↓
reject

## 360. Flujo logout

VoltStack Session
↓
LOCAL LOGOUT
↓
AuthenticationSession terminated
↓
cookie cleared

Optional:

```text
      ↓
Provider logout redirect
```

El primer paso no dependerá del segundo.

## 361. Arquitectura global

FEDERATED AUTHENTICATION
│
USER / BROWSER
│
▼
FederationManager
│
▼
FederationConnectionRegistry
│
▼
FederationAuthenticationTransaction
│
┌─────────────────────┼──────────────────────┐
▼                     ▼                      ▼
STATE                  NONCE                   PKCE
│                     │                      │
└─────────────────────┼──────────────────────┘
▼
AUTHORIZATION REQUEST
│
▼
EXTERNAL IDENTITY
PROVIDER
│
▼
AUTHORIZATION RESPONSE
│
▼
CALLBACK HANDLER
│
▼
CODE EXCHANGE
│
▼
VERIFIED OIDC TOKENS
│
┌──────────────┴──────────────┐
▼                             ▼
FederatedSubject              Assurance Claims
│                             │
▼                             ▼
Identity Mapping              Assurance Mapper
│                             │
└──────────────┬──────────────┘
▼
Local Identity
│
▼
Identity Security State
│
▼
FederatedAuthenticationEvidence
│
▼
AuthenticationContext
│
▼
AuthenticationSession

## 362. Decisiones arquitectónicas principales

VoltStack adoptará:

1. OAuth and Authentication are not synonyms.
2. OpenID Connect is the preferred federation protocol.
3. Authorization Code + PKCE is the default browser federation flow.
4. Every flow is transaction-based.
5. state, nonce and PKCE serve distinct purposes.
6. Provider trust comes from preconfigured connections.
7. issuer + subject is the canonical federated key.
8. Email is an attribute, not the default account-link key.
9. Federation does not override local Identity Security State.
10. Federated assurance is explicitly mapped.
11. External provider tokens are not local sessions.
12. Social Login, Enterprise SSO, Linking and Step-Up share the same federation primitives.
13. Provider-specific adapters cannot bypass Core validation.
14. Federation state is safe under distributed and persistent runtimes.
15. Comparación conceptual con Laravel y Symfony

De Laravel, VoltStack deberá conservar la simplicidad de experiencias similares a:

- Socialite
- provider redirects
- provider callbacks
- simple external-user retrieval

pero no limitará el modelo a:

```php
driver
redirect()
user()
```

De Symfony conservará la separación de:

- Authenticator
- Firewall
- OAuth/OIDC bundle integration
- Security Context
- Identity mapping

VoltStack añadirá una capa Core propia para:

- FederationConnection
- FederationTransaction
- PKCE
- state
- nonce
- OIDC verification
- issuer trust
- mix-up protection
- federated mapping
- JIT provisioning
- safe linking
- federated assurance
- tenant federation
- enterprise SSO
- persistent-runtime isolation

De esta forma, paquetes o adapters de Google/Microsoft/GitHub serán solamente implementaciones sobre un dominio federado estable.

## 16. Criterios de aceptación

El subsistema será considerado completo cuando:
17. soporte OAuth Authorization Code;
18. soporte OpenID Connect;
19. soporte PKCE;
20. soporte state;
21. soporte nonce;
22. tenga FederationTransaction;
23. soporte connection registry;
24. permita multiple providers;
25. soporte generic OIDC;
26. permita adapters sociales;
27. valide redirect URIs;
28. evite open redirects;
29. valide ID Token signatures;
30. valide issuer;
31. valide audience;
32. valide azp cuando corresponda;
33. valide expiration;
34. valide nonce;
35. soporte JWKS;
36. soporte discovery;
37. sea SSRF-safe;
38. soporte UserInfo;
39. valide UserInfo subject;
40. soporte issuer+subject mapping;
41. soporte JIT provisioning;
42. soporte account linking;
43. soporte unlinking;
44. evite auto-link inseguro por email;
45. soporte federated assurance;
46. interprete acr/amr mediante policy;
47. soporte fresh federated auth;
48. soporte step-up;
49. soporte tenant IdPs;
50. soporte home realm discovery;
51. soporte organization mapping;
52. evite group→role automático;
53. prevenga mix-up attacks;
54. prevenga login CSRF;
55. soporte local logout independiente;
56. tenga boundaries claros para provider logout;
57. maneje provider outages;
58. sea observable;
59. sea auditable;
60. sea seguro con FrankenPHP;
61. sea fiber-safe;
62. mantenga Federation separada de Authorization.
63. Regla arquitectónica final

VoltStack deberá preservar:

```text
TRUSTED FEDERATION CONNECTION
        ↓
SERVER-SIDE AUTHENTICATION TRANSACTION
        ↓
```

STATE + NONCE + PKCE
↓
EXTERNAL PROVIDER AUTHENTICATION
↓
AUTHORIZATION CODE
↓
TRUSTED TOKEN ENDPOINT
↓
VERIFIED ID TOKEN / PROVIDER EVIDENCE
↓
ISSUER + SUBJECT
↓
FEDERATED IDENTITY MAPPING
↓
LOCAL IDENTITY
↓
LOCAL SECURITY STATE
↓
FEDERATED ASSURANCE MAPPING
↓
AUTHENTICATION EVIDENCE
↓
AUTHENTICATION CONTEXT
La primera regla central será:
VoltStack delegará la prueba de identidad al proveedor externo, pero nunca delegará automáticamente el control sobre su Identity local, Tenant, Authorization o Security State.

La segunda será:
La confianza federada estará anclada en una FederationConnection configurada y en una transacción iniciada por VoltStack; nunca en URLs, issuers, callbacks o claims arbitrarios enviados por el cliente.

La tercera:
El identificador federado estable será el sujeto dentro de su autoridad emisora —issuer + subject— mientras que email, username, grupos y demás claims serán atributos que requieren políticas explícitas de mapping.

## 1. Próximo documento recomendado

El siguiente documento será:
`18_ACCOUNT_RECOVERY_PASSWORD_RESET_IDENTITY_RECOVERY_AND_CREDENTIAL_REESTABLISHMENT_SYSTEM.md`
Su responsabilidad será definir el sistema completo de recuperación de acceso, que hasta ahora solo hemos mencionado como dependencia de Password, MFA, Passkeys y Federation:

- Account Recovery
- Password Reset
- Lost MFA Factors
- Lost Passkeys
- Recovery Transactions
- Recovery Tokens
- Recovery Codes
- Email Recovery
- Recovery Links
- Identity Verification
- Credential Re-establishment
- Recovery Assurance
- Recovery Escalation
- Recovery Abuse Prevention
- Recovery Enumeration Resistance
- Recovery Token Hashing
- Token Expiration
- Single-use Recovery
- SecurityVersion changes
- Session Revocation
- Persistent Credential Revocation
- API Token Revocation
- MFA Re-enrollment
- Passkey Re-enrollment
- Federated Recovery
- Administrative Recovery
- High-risk Recovery
- Recovery Cooldowns
- Recovery Notifications
- Audit
- Observability
- Distributed State
- FrankenPHP safety

Este documento será crítico porque un sistema de Authentication solo es tan fuerte como su mecanismo de recuperación más débil. El objetivo será que VoltStack no implemente “Forgot Password” como un endpoint aislado, sino como un verdadero Identity Recovery and Credential Re-establishment System.
