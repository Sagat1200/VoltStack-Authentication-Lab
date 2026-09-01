# VoltStack Authentication System

## 29 — Authentication Multi-Tenancy, Security Realms, Cross-Tenant Isolation and Tenant Authentication Policy System

- **Archivo:** `29_AUTHENTICATION_MULTI_TENANCY_SECURITY_REALMS_CROSS_TENANT_ISOLATION_AND_TENANT_AUTHENTICATION_POLICY_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema responsable de autenticación multi-tenant, resolución de tenants, Security Realms, aislamiento entre tenants, políticas de autenticación específicas por tenant y composición de políticas globales, de plataforma y tenant.

---

## 1. Propósito

Este documento define cómo VoltStack deberá ejecutar Authentication en aplicaciones:

- Single Tenant
- Multi-Tenant
- Multi-Organization
- Multi-Workspace
- Multi-Domain
- Multi-Realm
- B2B SaaS
- Enterprise SaaS
- Platform SaaS

sin permitir que una identidad, sesión, credential, token, factor o contexto autenticado perteneciente a un tenant sea aceptado accidentalmente dentro de otro.
El subsistema deberá coordinar:

- Tenant Resolution
- Security Realm Resolution
- Tenant Authentication Context
- Identity Scoping
- Tenant Membership
- Tenant Identity Providers
- Tenant Authentication Policies
- Tenant MFA Policies
- Tenant Federation
- Tenant Sessions
- Tenant Tokens
- Tenant Remember-Me
- Tenant Passkeys
- Tenant Device Trust
- Tenant Risk
- Tenant Recovery
- Tenant Rate Limits
- Tenant Extensions
- Tenant Audit
- Cross-Tenant Isolation
- Tenant Switching

## 2. Principio fundamental

VoltStack deberá mantener como conceptos diferentes:

```text
Global Identity
        ≠
Tenant Membership
        ≠
Tenant Authentication Context
        ≠
Security Realm
        ≠
Authorization Context
```

Confundir estos conceptos produciría vulnerabilidades graves.

## 3. Ejemplo

Un usuario puede ser:

```text
Identity:
    user-100
```

Memberships:

```text
    tenant-A → ADMIN
    tenant-B → VIEWER
```

La autenticación:
user-100 authenticated in tenant-A
no implica automáticamente:

- user-100 authenticated in tenant-B
- aunque la misma identidad tenga membresía en ambos.

## 4. Authentication vs Authorization

Este documento únicamente determina:

- WHO authenticated?
- WHERE did authentication occur?
- UNDER WHICH TENANT?
- UNDER WHICH REALM?
- WITH WHICH CREDENTIALS?
- WITH WHICH ASSURANCE?

Authorization posteriormente determina:
WHAT may this identity do?

## 5. Regla de aislamiento

Toda Authentication deberá estar ligada explícita o implícitamente a un Security Realm y, cuando corresponda, a un Tenant Context.

## 1. Regla de no transferencia

Una Authentication válida dentro de un tenant o realm no será transferible automáticamente a otro.

## 2. Regla de composición

La política efectiva será conceptualmente:

```text
Framework Security Floor
        +
Platform Authentication Policy
        +
Security Realm Policy
        +
Tenant Authentication Policy
        +
Runtime Emergency Policy
```

La composición deberá producir la política:

- most restrictive applicable policy
- según las reglas semánticas definidas para cada propiedad.

## 3. Arquitectura general

REQUEST
│
▼
Firewall Resolution
│
▼
Realm Resolver
│
▼
Tenant Resolver
│
▼
Authentication Context
│
┌───────────────┼───────────────┐
▼               ▼               ▼
Realm           Tenant          Platform
Policy          Policy           Policy
│               │               │
└───────────────┼───────────────┘
▼
Effective Auth Policy
│
▼
Identity Provider
│
▼
Authentication
│
▼
Membership Validation
│
▼
Security State
│
▼
Authentication
Result

## 4. Conceptos principales

VoltStack definirá como mínimo:

```text
Tenant
TenantReference
TenantContext
TenantMembership
TenantAuthenticationContext

SecurityRealm
SecurityRealmId
SecurityRealmContext

TenantAuthenticationPolicy
PlatformAuthenticationPolicy
RealmAuthenticationPolicy
EffectiveAuthenticationPolicy
```

## 10. Tenant

Un Tenant representa una frontera lógica de aislamiento dentro de la aplicación.
Puede corresponder a:

- Company
- Organization
- Customer
- Account
- Institution
- Business Unit
- SaaS Customer

## 11. TenantReference

Authentication no deberá requerir cargar siempre toda la entidad Tenant.
Podrá utilizar:

```php
final readonly class TenantReference
{
    public function __construct(
        public TenantId $id,
    ) {}
}
```

## 12. TenantContext

Contendrá únicamente información necesaria para la operación actual.
final readonly class TenantContext
{
public function __construct(
public TenantReference $tenant,
public SecurityRealmId $realm,
) {}
}

## 13. Tenant Authentication Context

Será más específico:

```php
final readonly class TenantAuthenticationContext
{
    public function __construct(
        public TenantReference $tenant,
        public SecurityRealmId $realm,
        public TenantAuthenticationPolicy $policy,
    ) {}
}
```

## 14. Security Realm

Un Security Realm representa una frontera dentro de la cual una Authentication posee significado.
Ejemplos:

- public-web
- customer-portal
- admin-panel
- partner-portal
- internal-api
- machine-api
- support-console

## 15. Realm no equivale a Tenant

Ejemplo:

```text
Realm:
    customer-portal
```

Tenants:

```text
    acme
    globex
    wayne-enterprises
```

## 16. Tenant puede tener múltiples Realms

Ejemplo:

```text
Tenant ACME
 ├── customer
 ├── administration
 ├── api
 └── machine
```

## 17. Realm puede abarcar múltiples tenants

Ejemplo:

```text
customer-portal
    ├── Tenant A
    ├── Tenant B
    └── Tenant C
```

## 18. Realm ID

Debe ser estable:

```php
final readonly class SecurityRealmId
{
    public function __construct(
        public string $value,
    ) {}
}
```

## 19. Realm Context

final readonly class SecurityRealmContext
{
public function __construct(
public SecurityRealmId $id,
public AuthenticationRequirementSet $requirements,
) {}
}

## 20. Realm Resolver

Contrato:

```php
interface SecurityRealmResolverInterface
{
    public function resolve(
        AuthenticationRequestContext $request
    ): SecurityRealmResolution;
}
```

## 21. Tenant Resolver

Contrato:

```php
interface AuthenticationTenantResolverInterface
{
    public function resolve(
        AuthenticationRequestContext $request,
        SecurityRealmContext $realm
    ): TenantResolution;
}
```

## 22. Tenant Resolution Sources

Podrá utilizar:

- Host
- Subdomain
- Custom Domain
- Route Metadata
- Path Prefix
- Signed Request Metadata
- Trusted Gateway Metadata
- Explicit Login Context

## 23. Ejemplo — Subdomain

acme.example.com
puede resolver:
Tenant = ACME

## 24. Ejemplo — Custom Domain

portal.acme.com
puede mapearse a:

- Tenant ACME
- mediante TenantDomainRegistry.

## 25. Ejemplo — Path

/app/acme/login
podría resolver tenant mediante route metadata.

## 26. No confiar ciegamente en input

Nunca:

```php
$tenant = $_GET['tenant'];
y utilizarlo directamente como contexto de seguridad.
```

## 27. Tenant Resolution

Debe producir:

- Resolved
- NotFound
- Ambiguous
- Invalid
- Unavailable

## 28. Tenant Canonicalization

Aliases deberán convertirse a:

- Canonical TenantId
- antes de Authentication.

## 29. Tenant Resolution Precedence

Debe ser explícita.

```text
Ejemplo:
Trusted Route Metadata
        >
Verified Custom Domain
        >
Subdomain Mapping
        >
```

Explicit Authentication Flow Binding

## 30. Ambiguous Tenant

Si dos fuentes confiables indican tenants diferentes:

- DENY
- por default.

## 31. Ejemplo

Host:
acme.example.com
Flow:
tenant = globex
Resultado:
TENANT_CONTEXT_MISMATCH

## 32. Tenant Binding

Una vez iniciado un Authentication Flow, deberá quedar ligado al tenant.
AuthenticationFlow
tenant = ACME

## 33. Flow continuation

Cada paso deberá verificar:

```text
Current Tenant
    ==
Flow Tenant
```

## 34. MFA Binding

Challenge MFA también deberá quedar ligado a:

- Identity
- Tenant
- Realm
- Authentication Flow
- Purpose

## 35. Recovery Binding

Recovery deberá estar tenant-aware cuando la cuenta lo requiera.

## 36. Federation Binding

OAuth/OIDC state deberá conservar contexto suficiente para recuperar de forma segura:

- Tenant
- Realm
- Flow
- Provider

## 37. Identity Models

VoltStack deberá soportar al menos tres estrategias.

## 38. Model A — Tenant-local Identity

Tenant A
└── Identity 100

Tenant B
└── Identity 100
Son identities completamente distintas.

## 39. Model B — Global Identity + Membership

Global Identity
│
├── Membership Tenant A
└── Membership Tenant B
Especialmente útil en SaaS.

## 40. Model C — Hybrid Identity

Puede existir:

- Platform Identity
- Tenant-local Identity
- Federated Identity
- simultáneamente.

## 41. VoltStack no impondrá un único modelo

Pero deberá representar explícitamente cuál se utiliza.

## 42. GlobalIdentity

Representa persona/cuenta a nivel plataforma.

## 43. TenantMembership

Relaciona:

- Identity
- Tenant

## 44. Membership no es Authorization completa

Membership responde principalmente:

- Does this Identity belong to this Tenant?
- Los roles/permisos pertenecen al sistema Authorization.

## 45. Membership Status

Puede ser:

- ACTIVE
- INVITED
- SUSPENDED
- DISABLED
- LEFT
- EXPIRED

## 46. Authentication Eligibility

Para autenticarse dentro del tenant:

- Identity eligible
- AND
- Membership eligible
- AND
- Tenant eligible

## 47. Tenant Status

Podrá ser:

- ACTIVE
- SUSPENDED
- DISABLED
- LOCKED
- TERMINATED
- MAINTENANCE

## 48. Tenant suspended

Por default:
new Authentication denied

## 49. Existing Sessions

Policy puede determinar:

- revoke immediately
- expire naturally
- restricted maintenance access

## 50. TenantSecurityVersion

Cada tenant podrá tener:
TenantSecurityVersion

## 51. Uso

Al cambiar estado crítico:
TenantSecurityVersion++

## 52. Session binding

Session puede contener:
tenant_security_version_at_authentication

## 53. Fast revocation

Si:
session version != current tenant version
se requiere:

- revalidation
- o revocación.

## 54. MembershipSecurityVersion

Igualmente:
MembershipSecurityVersion

## 55. Cambio de membership

Por ejemplo:

```text
ACTIVE → SUSPENDED
deberá invalidar contextos pertinentes.
```

## 56. Identity Security Version

Ya definido previamente.

- Por tanto una Session multi-tenant puede depender de:
- IdentitySecurityVersion
- MembershipSecurityVersion
- TenantSecurityVersion
- AuthenticationPolicyVersion

## 57. Session Context

Conceptualmente:

```php
final readonly class TenantBoundAuthenticationSession
{
    public function __construct(
        public IdentityId $identity,
        public TenantId $tenant,
        public SecurityRealmId $realm,
        public AuthenticationAssurance $assurance,
        public SecurityVersionVector $versions,
    ) {}
}
```

## 58. Security Version Vector

Puede representar:

```php
identity = 12
membership = 4
tenant = 8
policy = 19
```

## 59. Session Revalidation

Puede comparar versiones eficientemente.

## 60. Session Cookie Scope

Debe configurarse cuidadosamente.

## 61. Subdomain SaaS

Ejemplo:

- acme.example.com
- globex.example.com

Compartir cookie:

- .example.com
- puede ser peligroso.

## 62. Default

Preferir cookie scoped al host cuando el diseño lo permita.

## 63. Shared SSO

Si se desea cross-tenant SSO:
debe ser explícito

## 64. Cross-Tenant SSO

No significa compartir directamente Session.

## 65. Modelo seguro

Platform SSO Session
│
▼
Tenant Entry Request
│
▼
Tenant Membership Check
│
▼
Tenant Policy Evaluation
│
▼
Tenant Authentication Context

## 66. Platform Authentication

Puede demostrar:
this is Identity X
pero Tenant todavía decide:
is Identity X eligible here?

## 67. Tenant Assurance

Tenant puede requerir mayor assurance.
Ejemplo:

- Platform Session = AAL1
- Tenant Admin = AAL2

Resultado:
STEP-UP REQUIRED

## 68. Cross-Realm Authentication

Misma regla.

## 69. Realm Trust Relationships

VoltStack podrá definir:
RealmTrustPolicy

## 70. Ejemplo

customer → admin
no confía automáticamente.

## 71. Trust Direction

Puede ser direccional:

```text
admin authentication
    may satisfy customer realm

customer authentication
    does NOT satisfy admin realm
```

## 72. RealmTrustRelationship

final readonly class RealmTrustRelationship
{
public function __construct(
public SecurityRealmId $source,
public SecurityRealmId $target,
public AuthenticationRequirementSet $requirements,
) {}
}

## 73. No implicit realm trust

Default:
NONE

## 74. Authentication Provenance

Para transferir Authentication entre realms se deberá evaluar:

- source realm
- methods
- assurance
- freshness
- tenant
- device
- risk

## 75. Tenant Switching

Un usuario con múltiples memberships puede cambiar de tenant.

## 76. Incorrecto

session.tenant = requestedTenant
sin más validaciones.

## 77. Correcto

Tenant Switch Request
↓
Target Tenant Resolve
↓
Membership Check
↓
Tenant Status
↓
Target Tenant Policy
↓
Current Assurance Evaluation
↓
Risk Evaluation
↓
Step-Up if required
↓
New Tenant Authentication Context

## 78. TenantSwitchService

interface TenantAuthenticationSwitchServiceInterface
{
public function switch(
AuthenticatedIdentity $identity,
TenantReference $target,
TenantSwitchContext $context
): TenantSwitchResult;
}

## 79. Session strategy

Tenant switching puede:

- rotate same session
- create new tenant session

issue tenant context token
según deployment.

## 80. Default security preference

Rotar identificadores relevantes.

## 81. Session fixation

Tenant switch no debe permitir fixation.

## 82. Tenant History

Session puede mantener tenant actual.
No confiar en tenant ID proveniente únicamente del frontend.

## 83. SPA

Frontend podrá solicitar:

- switch tenant
- pero backend realiza resolución completa.

## 84. Tenant Context Token

Opcionalmente podría existir token interno firmado:

- Identity
- Tenant
- Realm
- Session
- Expiry
- Security Versions

## 85. No bearer long-lived por default

Debe ser corto y ligado al contexto.

## 86. Tenant Authentication Policy

Contrato conceptual:

```php
final readonly class TenantAuthenticationPolicy
{
    public function__construct(
        public AuthenticationAssuranceRequirement $assurance,
        public AuthenticationMethodPolicy $methods,
        public MfaPolicy $mfa,
        public SessionPolicy $sessions,
        public RecoveryPolicy $recovery,
        public RiskPolicy $risk,
    ) {}
}
```

## 87. Tenant Configurable Properties

Según plataforma, Tenant puede controlar:

- allowed login methods
- MFA requirement
- passkey requirement
- session lifetime
- remember-me
- federation providers
- trusted device duration
- recovery methods
- IP restrictions
- risk sensitivity

## 88. No tenant unlimited control

Platform definirá:
TenantPolicyCapabilitySet

## 89. Example

Tenant puede:
require MFA
pero no:
disable platform mandatory MFA

## 90. Framework Security Floor

Ejemplo:

- password minimum = secure baseline
- recovery minimum = platform baseline

session fixation protection = mandatory
credential redaction = mandatory

## 91. Platform Policy

Puede establecer:
all enterprise tenants require AAL2

## 92. Realm Policy

Puede establecer:
admin realm requires phishing-resistant factor

## 93. Tenant Policy

Puede agregar:
ACME requires passkey

## 94. Emergency Policy

Puede agregar:
password login temporarily disabled

## 95. Effective Policy

Resultado:

- AAL2
- Passkey required
- Password disabled
- Phishing-resistant required

## 96. Policy Composition Engine

interface TenantAuthenticationPolicyComposerInterface
{
public function compose(
AuthenticationPolicyLayerSet $layers
): EffectiveAuthenticationPolicy;
}

## 97. Typed composition

Como documento 27:

```text
assurance → maximum
session lifetime → minimum
```

required factors → composition
allowed methods → intersection
denied methods → union
rate limits → most restrictive compatible

## 98. Allowed Methods

Ejemplo:

```text
Platform:
    password, passkey, oidc
```

Tenant:
passkey, oidc
Resultado:
passkey, oidc

## 99. Denied Methods

Emergency:
deny oidc
Resultado:
passkey

## 100. No empty authentication set silently

Si composición produce:
no usable authentication methods
deberá generar:
POLICY_UNSATISFIABLE

## 101. Compile-time validation

Si policies son estáticas, detectar antes.

## 102. Runtime tenant policy

Si dinámica, detectar al resolver.

## 103. Tenant Policy Version

Cada cambio deberá incrementar:
TenantAuthenticationPolicyVersion

## 104. Cache

Cache key:

- TenantId
- RealmId
- PolicyVersion
- PlatformPolicyVersion

## 105. Tenant Policy Cache Isolation

Nunca:

```text
admin-policy → cached globally
si depende del tenant.
```

## 106. Correct

tenant:acme:realm:admin:policy:v18
conceptualmente.

## 107. Tenant Policy Cache

Debe ser:

- bounded
- versioned
- observable

## 108. Large SaaS

No cargar políticas de todos los tenants durante bootstrap.

## 109. Base + Overlay

Compiled Platform Policy
+
Compiled Realm Policy
+
Tenant Policy Overlay

## 110. Tenant-specific Identity Provider

Tenant puede usar provider distinto.

```text
Ejemplo:
ACME → Azure/OIDC
```

Globex → Local Password
Wayne → Enterprise IdP

## 111. Provider Resolver

interface TenantIdentityProviderResolverInterface
{
public function resolve(
TenantReference $tenant,
AuthenticationPurpose $purpose
): IdentityProviderSet;
}

## 112. Provider IDs

Deben ser tenant-scoped cuando corresponda.

## 113. OIDC Tenant Configuration

Puede incluir:

- issuer
- client ID
- secret reference
- allowed domains
- claim mapping

## 114. Secrets

No almacenar raw secrets dentro de:
TenantAuthenticationPolicy
Preferir:
SecretReference

## 115. Tenant Federation Registry

Conceptualmente:

```text
TenantId
    ↓
FederationProviderSet
```

## 116. OIDC callback

No confiar en tenant proporcionado por callback query.
Debe recuperarse desde:
signed/bound Authentication Flow

## 117. Federation Account Collision

Dos tenants pueden recibir:

```php
sub = 12345
del mismo/diferente IdP.
```

Clave deberá incluir suficiente namespace:

- Tenant
- Issuer
- Subject

cuando mapping sea tenant-local.

## 118. Federated Identity Key

issuer + subject
puede ser global si arquitectura lo define.
Pero Tenant Membership sigue separado.

## 119. Email no es identidad federada estable

No utilizar únicamente:

- email
- para vincular cuentas automáticamente salvo policy explícita y segura.

## 120. Tenant Domain Claim

Un tenant puede configurar:
@acme.com
pero esto no convierte cualquier cuenta con ese email automáticamente en miembro.

## 121. Tenant-specific Password Policy

Tenant puede endurecer:

- password length
- breach checks

rotation under enterprise policy
pero no debilitar baseline.

## 122. Tenant Password Credentials

Dependiendo del modelo pueden ser:

- global
- tenant-local

## 123. Global Password

Una credential autentica Global Identity.
Luego:

- Membership + Tenant Policy
- establecen Tenant Authentication Context.

## 124. Tenant-local Password

Credential deberá quedar ligada a:
Tenant + Identity

## 125. Credential Namespace

Debe impedir collisions.

## 126. Passkeys

También deben definir scope.

## 127. Global Passkey

Puede autenticar Global Identity.

## 128. Tenant-specific Passkey

Puede pertenecer a:
Tenant Credential Namespace

## 129. RP ID

WebAuthn tiene implicaciones importantes en subdomains/custom domains.

## 130. RP configuration

Debe derivarse de configuración confiable del Realm/Tenant.
Nunca de Host arbitrario sin validación.

## 131. Custom Domains

TenantDomainRegistry deberá confirmar que el dominio pertenece al tenant.

## 132. Domain Verification

Podrá existir estado:

- PENDING
- VERIFIED
- REVOKED

## 133. Authentication solo usa VERIFIED domains

## 134. Domain Takeover Prevention

Al liberar custom domain:

- sessions/flows bound to previous mapping
- deberán tratarse cuidadosamente.

## 135. Domain Mapping Version

Podrá existir:
TenantDomainVersion

## 136. Tenant Sessions

Session deberá tener scope claro:

- platform
- tenant
- realm

## 137. SessionScope

enum AuthenticationSessionScope
{
case PLATFORM;
case TENANT;
case REALM;
}
Conceptualmente.

## 138. Tenant Session

Debe contener:

- TenantId
- obligatoriamente.

## 139. Realm Session

Debe contener:
RealmId

## 140. Session Lookup

No debe cargar Session y después confiar en que coincide.
Debe validar binding antes de utilizar Authentication state.

## 141. Session Key Namespace

Distributed store:

```php
auth:tenant:{tenant}:session:{id}
puede utilizarse conceptualmente.
```

## 142. Session IDs

Deben continuar siendo impredecibles aunque tengan namespace interno.

## 143. Logout

Debe distinguir:

- logout current realm
- logout current tenant
- logout all tenants
- logout platform-wide

## 144. API

Conceptualmente:

```php
Auth::logoutCurrentRealm();
Auth::logoutTenant();
Auth::logoutEverywhere();
```

## 145. Tenant Logout

No debe necesariamente cerrar sesiones de otros tenants.

## 146. Global Logout

Sí puede revocar todas.

## 147. Remember-Me

Persistent credential deberá incluir scope.

## 148. Tenant Remember-Me

Identity
Tenant
Realm
Credential Family

## 149. Cross-Tenant Remember-Me

Deshabilitado por default.

## 150. Platform Remember-Me

Si se permite, deberá volver a validar Membership y Tenant Policy al entrar a cada tenant.

## 151. API Tokens

Tokens multi-tenant requieren especial cuidado.

## 152. Tenant-bound token

Claims/metadata:

- subject
- tenant
- realm/audience
- issuer
- expiry

## 153. Global API Token

No deberá implicar acceso a todos los tenants.
Authorization/membership seguirá siendo necesario.

## 154. Token Tenant Claim

Si request intenta:
Tenant B
pero token dice:
Tenant A
Resultado:
TOKEN_TENANT_MISMATCH

## 155. Token Audience

Security Realm puede mapearse a audience.

## 156. Service-to-Service

Machine identities pueden pertenecer a:

- Tenant
- Platform
- Realm

## 157. Tenant Machine Identity

Credential deberá quedar scoped.

## 158. Platform Machine Identity

Puede requerir explicit delegation para actuar sobre tenant.

## 159. Impersonation

No pertenece exclusivamente a Authentication, pero Authentication debe preservar:

- actor
- subject
- tenant
- realm

## 160. Support Impersonation

Ejemplo:

```text
Platform Support Identity
        ↓
Approved impersonation
        ↓
Tenant ACME
        ↓
Subject User
```

No debe transformarse simplemente en:
currentUser = subject

## 161. Actor Chain

Authentication Context deberá conservar:

- Actor
- Effective Subject
- Tenant
- Realm
- para Authorization/Audit.

## 162. Tenant Recovery

Recovery deberá considerar:

- global identity recovery
- tenant membership recovery
- tenant-local credential recovery
- como operaciones diferentes.

## 163. Example

Restablecer password global no reactiva:
SUSPENDED tenant membership

## 164. Tenant Admin Recovery

Un administrador de tenant podría ayudar a recuperar acceso.

- Pero no debería poder recuperar:
- platform account
- salvo policy explícita.

## 165. Recovery Authority Boundaries

Tenant Admin
authority limited to Tenant

## 166. Tenant MFA

Tenant puede exigir:

- MFA
- Passkey
- Hardware-backed
- Phishing-resistant

## 167. MFA Enrollment

Enrollment puede ser:

- global
- tenant-specific

## 168. Factor Reuse

Un factor global puede reutilizarse entre tenants si policy lo permite.

## 169. Tenant Factor

No debe aparecer en otro tenant.

## 170. Factor Identifier Namespace

Debe incluir scope.

## 171. Trusted Devices

Trust puede ser:

- platform-wide
- tenant-specific
- realm-specific

## 172. Default

Preferir scope restringido.

## 173. Example

Trusted en:
Tenant A customer realm
no significa trusted en:
Tenant A admin realm

## 174. Device Trust Key

Puede depender de:

- Identity
- Tenant
- Realm
- Device

## 175. Risk

Risk debe recibir tenant context.

## 176. Tenant Risk Baseline

Comportamiento normal puede variar por tenant.

## 177. Cross-Tenant Risk

Una señal crítica global puede afectar todos los tenants.
Ejemplo:
credential compromise

## 178. Tenant-local signal

Ejemplo:

- unusual login for ACME
- puede permanecer tenant-scoped.

## 179. Signal Scope

GLOBAL
TENANT
REALM
SESSION
DEVICE

## 180. Risk aggregation

Debe respetar scope.

## 181. Abuse Protection

Rate limiting deberá incluir tenant cuando corresponda.

## 182. Example

Tenant + IP
Tenant + Identity
Tenant + Username

## 183. Global abuse limit

También puede existir para impedir que atacante distribuya ataques entre tenants.

## 184. Composite defense

Global IP limit
+
Tenant IP limit
+
Identity limit

## 185. Tenant enumeration

Errores no deberán revelar:

- tenant exists
- user belongs to tenant
- innecesariamente.

## 186. Login Error

Preferir respuestas públicas normalizadas.

## 187. Custom Domain Enumeration

Puede ser inevitable que dominio exista por routing, pero Authentication no deberá agregar información innecesaria.

## 188. Audit

Cada evento tenant-aware deberá incluir:

- TenantId
- RealmId
- cuando aplique.

## 189. Example

AuthenticationSucceeded
identity = X
tenant = ACME
realm = admin

## 190. Cross-Tenant Audit Query

Security operators autorizados pueden investigar globalmente.
Tenant administrators solo su tenant.
Authorization decide acceso a logs.

## 191. Audit Integrity

Tenant no debe poder alterar TenantId registrado.

## 192. Observability

Metrics deberán evitar high-cardinality uncontrolled tenant labels.

## 193. Tenant metrics

Podrán agregarse selectivamente.

## 194. PII

No usar email como metric label.

## 195. Tenant Extension Policy

Documento 28.
Tenant puede habilitar ciertos Authentication providers/extensions.

## 196. Platform allowlist

Primero:
Platform allowed extensions

## 197. Tenant subset

Después:
Tenant enabled extensions

## 198. Effective Extensions

PlatformAllowed ∩ TenantEnabled

## 199. Tenant no instala código PHP

En SaaS normal, tenant solo habilita/configura extensiones previamente instaladas por plataforma.

## 200. Critical distinction

Plugin Installation
es operación platform-level.
Plugin Activation
puede ser tenant-level.

## 201. Tenant Authenticator Registry

Puede construirse como vista filtrada:

```text
Global Compiled Registry
        +
Tenant Activation Policy
        =
Effective Tenant Registry
```

## 202. No per-tenant full compiler

Evitar para grandes cantidades de tenants.

## 203. Tenant Feature Cache

Puede cachear effective registry.
Debe incluir:
TenantExtensionPolicyVersion

## 204. Tenant Federation Secrets

Secrets tenant-specific deberán almacenarse en secret storage seguro.

## 205. Encryption

Si persisten en DB:

- encrypted at rest
- además de controles de acceso.

## 206. Secret Retrieval

Solo provider correspondiente deberá solicitar secret.

## 207. No Tenant object dumps

Diagnostics nunca deben imprimir secretos.

## 208. Data Isolation

Authentication repositories deberán aceptar Tenant context explícito cuando sean tenant-scoped.

## 209. Correct

$repository->findCredential(
tenant: $tenant,
identity: $identity
);

## 210. Risky

$repository->findCredential($identity);
si credential es tenant-scoped.

## 211. Tenant-aware repository contract

interface TenantCredentialRepositoryInterface
{
public function find(
TenantReference $tenant,
CredentialId $credential
): ?CredentialRecord;
}

## 212. Database isolation

Puede utilizar:

- tenant_id columns
- schema-per-tenant
- database-per-tenant

Authentication no deberá asumir una única estrategia.

## 213. Storage Isolation Adapter

Database subsystem resolverá detalles.
Authentication exige:
TenantBoundary preserved

## 214. Cache Isolation

Cache keys deberán incluir tenant.

## 215. Queue Isolation

Authentication async jobs deberán transportar tenant context firmado/validado.

## 216. Never infer Tenant from stale worker state

Especialmente FrankenPHP/queue workers.

## 217. Event Isolation

Tenant-aware events incluyen immutable TenantReference.

## 218. Listener context

No depender de:

```php
Tenant::current()
si event ya puede llevar tenant explícito.
```

## 219. Cross-Tenant Operations

Algunas operaciones platform-level necesitan trabajar entre tenants.

## 220. Explicit Cross-Tenant Capability

Deberán utilizar:

- CrossTenantAuthenticationContext
- o service especializado.

## 221. No accidental bypass

Un repository tenant-aware no deberá permitir:

```php
tenant = null
para significar "todos".
```

## 222. Platform repository

Debe ser contrato distinto.

## 223. Example

TenantSessionRepository
PlatformSessionAdministrationRepository

## 224. Cross-Tenant Authentication Administration

Puede incluir:

- global logout
- incident response
- identity compromise
- platform suspension

## 225. Audit requirement

Toda operación cross-tenant administrativa deberá auditarse.

## 226. Tenant Isolation Guard

Componente central:

```php
interface AuthenticationTenantIsolationGuardInterface
{
    public function assert(
        AuthenticationTenantBoundary $expected,
        AuthenticationTenantBoundary $actual
    ): void;
}
```

## 227. Uso

Antes de reutilizar:

- Session
- Flow
- Credential
- Token
- Factor
- Device Trust
- Recovery Challenge
- validar boundary.

## 228. TenantBoundary

final readonly class AuthenticationTenantBoundary
{
public function __construct(
public ?TenantId $tenant,
public SecurityRealmId $realm,
) {}
}

## 229. Global objects

tenant = null solo será válido para tipos explícitamente globales.

## 230. No wildcard semantics

Nunca interpretar:
tenant = null
como:
any tenant

## 231. Critical invariant

NULL TENANT ≠ ALL TENANTS

## 232. Tenant mismatch

Debe producir internal reason:
AUTH_TENANT_BOUNDARY_MISMATCH

## 233. Public response

Normalizada para evitar disclosure.

## 234. Security Realm Isolation Guard

Análogo:
AUTH_REALM_BOUNDARY_MISMATCH

## 235. Cross-Realm reuse

Solo permitido mediante:
RealmTrustPolicy

## 236. Tenant + Realm Boundary

La frontera real suele ser:
(TenantId, RealmId)

## 237. Authentication Scope

Puede formalizarse:

```php
final readonly class AuthenticationScope
{
    public function __construct(
        public ?TenantId $tenant,
        public SecurityRealmId $realm,
    ) {}
}
```

## 238. Scope Equality

Debe utilizar comparación tipada.

## 239. No loose string comparison

Especialmente IDs.

## 240. Security Context Resolution

Orden recomendado:

```text
Request
  ↓
Route/Firewall
  ↓
Realm
  ↓
Tenant
  ↓
Scope
  ↓
Policy
  ↓
Authentication
```

## 241. Why Realm before Tenant?

Porque Realm puede determinar:

- whether tenant is required
- how tenant is resolved

which resolver is valid

## 242. Platform Realm

Puede no requerir Tenant.
Ejemplo:
platform-account

## 243. Tenant Realm

Sí requiere.
Ejemplo:
tenant-admin

## 244. Machine Realm

Puede utilizar Tenant opcional/obligatorio según service identity.

## 245. Scope Requirement

Cada Realm declara:

- TENANT_REQUIRED
- TENANT_OPTIONAL
- TENANT_FORBIDDEN

## 246. Tenant forbidden

Ejemplo:

- platform-superadmin
- puede prohibir accidental Tenant context para evitar confusión.

## 247. RealmDescriptor

final readonly class SecurityRealmDescriptor
{
public function __construct(
public SecurityRealmId $id,
public TenantRequirement $tenantRequirement,
public AuthenticationRequirementSet $requirements,
) {}
}

## 248. Realm Registry

Compilado en documento 27.

## 249. Realm Resolution O(1)

Cuando route metadata permita.

## 250. Tenant Domain Registry

Puede ser dinámico.

## 251. Domain Cache

Debe incluir mapping version.

## 252. Negative Cache

Puede cachear domain not found brevemente para protección.

## 253. Domain changes

Invalidar inmediatamente cuando sea posible.

## 254. Tenant Resolution DoS

Attacker puede generar muchos subdomains aleatorios.

## 255. Protection

bounded negative cache
rate limiting
efficient lookup

## 256. No unbounded domain cache

FrankenPHP.

## 257. Tenant Policy DoS

Igualmente muchos tenant IDs falsos no deben llenar caches.
258. Resolve canonical tenant before caching policy
259. Tenant Authentication Entry Point

Puede variar.

```text
Ejemplo:
ACME → SSO only
```

Globex → Password + Passkey

## 260. Entry Point Resolver

interface TenantAuthenticationEntryPointResolverInterface
{
public function resolve(
TenantAuthenticationContext $context
): AuthenticationEntryPoint;
}

## 261. Login UI

Puede mostrar métodos permitidos por effective policy.

## 262. No security solely in UI

Backend valida nuevamente.

## 263. Tenant Branding

Puede personalizar login.
Pero branding metadata deberá mantenerse separada de security policy.

## 264. Login Discovery

Una plataforma puede preguntar email primero y determinar tenant/IdP.

## 265. Security concern

Esto puede permitir:

- account enumeration
- tenant enumeration

## 266. Discovery Policy

Deberá definir:
what information can be disclosed

## 267. Home Realm Discovery

Para enterprise federation puede existir:

```text
email/domain
    ↓
IdP discovery
```

## 268. Verified Mapping

No confiar simplemente en dominio escrito por usuario.
Usar configuration mapping.

## 269. Multiple Tenant Memberships

Email puede pertenecer a múltiples tenants.
No seleccionar automáticamente sin policy.

## 270. Tenant Chooser

Puede mostrarse después de Platform Authentication.

## 271. Chooser data

Solo listar memberships de la Identity autenticada.

## 272. No arbitrary tenant IDs

Frontend no debe recibir catálogo completo.

## 273. Tenant Invitation

INVITED membership puede permitir special onboarding Authentication Flow.

## 274. Invitation Token

Debe estar ligado a:

- Tenant
- Invitation

Intended Identity/email where applicable
Expiry
Purpose

## 275. Invitation acceptance

No equivale automáticamente a authenticated tenant session hasta completar policy.

## 276. Tenant Offboarding

Al eliminar membership:

- MembershipSecurityVersion++
- y revocar contexts correspondientes.

## 277. Tenant Deletion

Debe invalidar:

- sessions
- tokens
- remember-me credentials
- flows
- recovery challenges
- device trust
- tenant-specific credentials
- según retention policies.

## 278. Soft Delete

Authentication deberá tratar tenant eliminado/suspended como ineligible.

## 279. Race Conditions

Ejemplo:

- Authentication succeeds
- membership revoked concurrently
- session created

## 280. Finalization Revalidation

Antes de emitir Session:
revalidate critical versions/status

## 281. Optimistic Security Version Check

Puede detectar cambios concurrentes.

## 282. Transaction boundary

Cuando storage lo permita, finalization puede usar transaction.

## 283. Distributed race

Version checking seguirá siendo importante.

## 284. Tenant Policy changed mid-flow

Flow comenzó con:
AAL1
Tenant cambia a:

- AAL2
- antes de finalizar.

## 285. Finalization

Debe usar:

- current effective policy
- o policy version rules explícitas.

## 286. Default

Security-increasing changes deben aplicarse antes de finalization.

## 287. Security-decreasing change

No necesita reducir requirement de flow ya iniciado.

## 288. Monotonic Flow Security

Durante un flow:

- requirements may increase
- but should not silently decrease

## 289. Policy Snapshot

Flow puede guardar:

- policy version
- requirements

## 290. Final requirement

Conceptualmente:

```php
max(
    flow security requirement,
    current security requirement
)
semánticamente, no numéricamente.
```

## 291. Tenant Emergency Policy

Puede responder a incidente específico.
Ejemplo:
Tenant ACME compromised

## 292. Emergency actions

disable password
require passkey
revoke all sessions
disable federation

## 293. Platform Emergency Policy

Puede afectar:
all tenants

## 294. Emergency versioning

Debe propagarse rápidamente.

## 295. Authentication Cache

No usar stale permissive policy durante emergencia.

## 296. Fail-safe

Si policy critical no puede resolverse:

- fail closed
- para realms sensibles.

## 297. Availability tradeoff

Puede configurarse según realm.

## 298. Admin Realm

Preferir fail closed.

## 299. Low-risk public realm

Puede tener política distinta, sin violar security floor.

## 300. Tenant Policy Source

Puede provenir de:

- database
- configuration service
- distributed policy store

## 301. TenantAuthenticationPolicyRepository

interface TenantAuthenticationPolicyRepositoryInterface
{
public function get(
TenantReference $tenant,
SecurityRealmId $realm
): TenantAuthenticationPolicyRecord;
}

## 302. Policy repository failure

Debe distinguir:

- NOT_FOUND
- UNAVAILABLE
- CORRUPT
- VERSION_CONFLICT

## 303. Default policy

Solo utilizar si explícitamente definido.

## 304. No accidental permissive default

Nunca:

```php
return new TenantPolicy();
con valores permisivos porque DB falló.
```

## 305. Tenant Configuration Validation

Cuando tenant cambia policy:

- validate
- normalize
- compose
- check satisfiability
- persist
- version++
- invalidate cache

## 306. Tenant cannot save impossible policy

Ejemplo:

- Require SmartCard
- pero tenant no tiene SmartCard provider habilitado.

## 307. Policy activation

Puede utilizar:

- DRAFT
- ACTIVE
- SCHEDULED

## 308. Scheduled policy

Enterprise puede programar:
MFA mandatory starting Monday
309. Authentication deberá usar active version
310. Policy History

Mantener versiones ayuda a:

- audit
- incident investigation
- rollback

## 311. Rollback

No debe saltarse security floor actual.

## 312. Tenant-specific Rate Limits

Puede variar dentro de límites platform.

## 313. Tenant cannot raise above platform maximum exposure

Ejemplo:

- Platform:
- max 10 password attempts/min

Tenant no puede elegir:

- 1000/min
- si platform lo prohíbe.

## 314. Tenant can tighten

Ejemplo:
5/min

## 315. Tenant-specific Session Lifetime

Platform:
maximum 30 days
Tenant:
8 hours
Resultado:
8 hours

## 316. Tenant-specific Remember-Me

Puede deshabilitarlo.

## 317. Tenant-specific Passwordless

Puede exigir:

- passkey only
- si capabilities disponibles.

## 318. Tenant-specific Federation Only

Puede exigir:
enterprise OIDC only

## 319. Break-glass accounts

Enterprise puede necesitar fallback.
Debe ser explícito y altamente auditado.

## 320. BreakGlassAuthenticationPolicy

Debe definir:

- who
- when
- assurance
- audit
- expiration

## 321. Break-glass no bypass universal

## 322. Platform Superadmin

No deberá autenticarse accidentalmente como tenant user por compartir email.

## 323. Separate Realm

Recomendado:
platform-admin

## 324. Separate Credentials

Preferible para roles críticos.

## 325. Tenant Admin Realm

Puede ser:

- tenant-admin
- con tenant obligatorio.

## 326. Realm Confusion Attack

Ejemplo:

- Token emitido para:
- customer

usado en:

- admin
- Debe fallar.

## 327. Tenant Confusion Attack

Token emitido para:
Tenant A
usado en:

- Tenant B
- Debe fallar.

## 328. Identity Confusion Attack

Federated account del Tenant A no debe vincularse automáticamente a local account Tenant B.

## 329. Session Confusion Attack

Session platform no debe tratarse como tenant-admin Session sin evaluation.

## 330. Cache Confusion Attack

Cached Identity Tenant A no debe devolverse para lookup Tenant B.

## 331. Device Trust Confusion

Trusted Device Tenant A no debe elevar Tenant B accidentalmente.

## 332. Recovery Confusion

Recovery token Tenant A no debe resetear credential Tenant B.

## 333. MFA Challenge Confusion

Challenge de Realm customer no debe completar admin Authentication.

## 334. Flow Confusion

Authentication Flow ID siempre deberá estar bound a Scope.

## 335. Scope fingerprint

Podrá utilizar:

- tenant + realm + purpose
- dentro del binding.

## 336. Cross-Tenant Data Structures

Todo registro security-sensitive deberá declarar scope.

- Ejemplo:
- AuthenticationSession
- AuthenticationFlow
- Credential
- RecoveryChallenge
- RememberMeCredential
- DeviceTrustRecord
- FederatedMapping
- RiskRecord

## 337. AuthenticationScopeAware

Podrá existir:

```php
interface AuthenticationScopeAwareInterface
{
    public function authenticationScope(): AuthenticationScope;
}
```

## 338. Isolation Guard Generic

interface AuthenticationScopeGuardInterface
{
public function assertCompatible(
AuthenticationScope $expected,
AuthenticationScope $actual
): void;
}

## 339. Security Invariants — Tenant Resolution

AUTH-TENANT-RES-01
Tenant resolution occurs before tenant-scoped Authentication state is consumed.
AUTH-TENANT-RES-02
Tenant identifiers from untrusted input are never treated as authoritative without resolution.
AUTH-TENANT-RES-03
Conflicting trusted Tenant signals fail safely.

- AUTH-TENANT-RES-04
- Authentication Flows are bound to their resolved Tenant.
- AUTH-TENANT-RES-05

Tenant resolution state never leaks between FrankenPHP requests.

## 340. Security Invariants — Realms

AUTH-REALM-01
Every Authentication belongs to an explicit Security Realm.

- AUTH-REALM-02
- Realm Authentication is not transferable without explicit trust policy.
- AUTH-REALM-03

Realm trust relationships are directional and explicit.

- AUTH-REALM-04
- Realm requirements participate in Effective Authentication Policy.
- AUTH-REALM-05

Realm mismatch never silently falls back to another realm.

## 341. Security Invariants — Isolation

AUTH-TENANT-ISO-01
Authentication state belonging to Tenant A cannot be consumed as Tenant B state.
AUTH-TENANT-ISO-02
null Tenant never means all tenants.

- AUTH-TENANT-ISO-03
- Tenant-scoped cache keys contain canonical Tenant identity.
- AUTH-TENANT-ISO-04

Tenant-scoped repositories require explicit Tenant context.
AUTH-TENANT-ISO-05
Cross-Tenant administrative operations use dedicated privileged contracts.

## 342. Security Invariants — Identity

AUTH-TENANT-ID-01
Global Identity and Tenant Membership are distinct concepts.

- AUTH-TENANT-ID-02
- Global Authentication does not imply Tenant eligibility.
- AUTH-TENANT-ID-03

Tenant Authentication verifies Membership eligibility when applicable.
AUTH-TENANT-ID-04
Membership suspension invalidates applicable Authentication contexts.
AUTH-TENANT-ID-05
Federated mappings preserve Tenant/Issuer/Subject boundaries according to configured identity model.

## 343. Security Invariants — Sessions

AUTH-TENANT-SESSION-01
Tenant Sessions are explicitly Tenant-bound.

- AUTH-TENANT-SESSION-02
- Realm Sessions are explicitly Realm-bound.
- AUTH-TENANT-SESSION-03

Tenant switching cannot mutate Authentication scope without validation.
AUTH-TENANT-SESSION-04
Cross-Tenant SSO revalidates target Tenant eligibility and policy.
AUTH-TENANT-SESSION-05
Security Version changes can invalidate stale Tenant Sessions.

## 344. Security Invariants — Credentials

AUTH-TENANT-CRED-01
Tenant-local credentials are stored and resolved within Tenant scope.
AUTH-TENANT-CRED-02
Tenant Remember-Me credentials are not reusable across tenants.
AUTH-TENANT-CRED-03
Tenant-bound tokens reject Tenant mismatches.

- AUTH-TENANT-CRED-04
- MFA challenges preserve Tenant and Realm bindings.
- AUTH-TENANT-CRED-05

Recovery credentials preserve their intended Authentication scope.

## 345. Security Invariants — Policy

AUTH-TENANT-POL-01
Tenant policy cannot weaken Framework Security Floor.

- AUTH-TENANT-POL-02
- Tenant policy composition uses typed security semantics.
- AUTH-TENANT-POL-03

Unsatisfiable Authentication policies fail explicitly.

- AUTH-TENANT-POL-04
- Tenant policy caches are versioned and isolated.
- AUTH-TENANT-POL-05

Security-increasing policy changes are considered before Authentication finalization.

## 346. Security Invariants — Runtime

AUTH-TENANT-RT-01
Current Tenant is execution-scoped.

- AUTH-TENANT-RT-02
- Current Realm is execution-scoped.
- AUTH-TENANT-RT-03

Tenant state is reset after every request.

- AUTH-TENANT-RT-04
- Fiber execution contexts do not share Tenant state.
- AUTH-TENANT-RT-05

Dynamic Tenant caches are bounded.

## 347. Anti-pattern — Global Tenant::current() mutable singleton

No.

## 348. Anti-pattern — Tenant from query parameter used directly

No.

## 349. Anti-pattern — Session authenticated globally means authenticated everywhere

No.

## 350. Anti-pattern — tenant_id = null means all tenants

Nunca.

## 351. Anti-pattern — Shared cookie automatically means cross-tenant trust

No.

## 352. Anti-pattern — Email determines tenant membership

No.

## 353. Anti-pattern — Federation callback chooses tenant

No.

## 354. Anti-pattern — Cache Identity by email only

No en modelos tenant-aware.

## 355. Anti-pattern — Tenant policy uses generic array_merge

No.

## 356. Anti-pattern — Tenant can disable mandatory security control

Nunca.

## 357. Anti-pattern — Tenant switch modifies only session variable

No.

## 358. Anti-pattern — Passkey RP ID derived blindly from request Host

No.

## 359. Anti-pattern — MFA challenge reusable in another realm

No.

## 360. Anti-pattern — Recovery token reusable cross-tenant

No.

## 361. Anti-pattern — Plugin activation bypasses platform allowlist

No.

## 362. Anti-pattern — Tenant context retained in FrankenPHP singleton

Crítico.

## 363. Componentes principales

TenantReference
TenantContext
TenantAuthenticationContext
TenantMembership
TenantMembershipStatus

SecurityRealm
SecurityRealmId
SecurityRealmContext
SecurityRealmDescriptor

AuthenticationScope
AuthenticationTenantBoundary
SecurityVersionVector

## 364. Resolution Components

SecurityRealmResolver
AuthenticationTenantResolver
TenantDomainRegistry
TenantDomainResolver
TenantCanonicalizer
AuthenticationScopeResolver

## 365. Policy Components

TenantAuthenticationPolicy
PlatformAuthenticationPolicy
RealmAuthenticationPolicy
EffectiveAuthenticationPolicy
TenantAuthenticationPolicyComposer
TenantAuthenticationPolicyRepository
TenantAuthenticationPolicyVersion
TenantPolicyCapabilitySet

## 366. Isolation Components

AuthenticationTenantIsolationGuard
AuthenticationScopeGuard
SecurityRealmIsolationGuard
TenantCredentialRepository
TenantSessionRepository
TenantFlowRepository

## 367. Tenant Switching Components

TenantAuthenticationSwitchService
TenantSwitchContext
TenantSwitchResult
RealmTrustPolicy
RealmTrustRelationship

## 368. Federation Components

TenantFederationRegistry
TenantFederationProviderResolver
TenantFederatedIdentityMapper
TenantFederationConfiguration

## 369. Runtime Components

TenantAuthenticationExecutionContext
TenantAuthenticationContextResolver
TenantAuthenticationRuntimeResetter
TenantAuthenticationPolicyCache
TenantExtensionPolicyResolver

## 370. Namespace sugerido

VoltStack\Quantum\Auth\Tenant
VoltStack\Quantum\Auth\Tenant\Contracts
VoltStack\Quantum\Auth\Tenant\Context
VoltStack\Quantum\Auth\Tenant\Resolution
VoltStack\Quantum\Auth\Tenant\Realm
VoltStack\Quantum\Auth\Tenant\Policy
VoltStack\Quantum\Auth\Tenant\Identity
VoltStack\Quantum\Auth\Tenant\Session
VoltStack\Quantum\Auth\Tenant\Credential
VoltStack\Quantum\Auth\Tenant\Federation
VoltStack\Quantum\Auth\Tenant\Isolation
VoltStack\Quantum\Auth\Tenant\Runtime

## 371. Estructura sugerida

src/Quantum/Auth/Tenant/
├── Contracts/
│   ├── AuthenticationTenantResolverInterface.php
│   ├── SecurityRealmResolverInterface.php
│   ├── TenantAuthenticationPolicyRepositoryInterface.php
│   ├── TenantAuthenticationPolicyComposerInterface.php
│   ├── TenantIdentityProviderResolverInterface.php
│   ├── TenantAuthenticationSwitchServiceInterface.php
│   └── AuthenticationScopeGuardInterface.php
│
├── Context/
│   ├── TenantReference.php
│   ├── TenantContext.php
│   ├── TenantAuthenticationContext.php
│   ├── AuthenticationScope.php
│   └── SecurityVersionVector.php
│
├── Resolution/
│   ├── AuthenticationTenantResolver.php
│   ├── TenantResolution.php
│   ├── TenantCanonicalizer.php
│   ├── TenantDomainResolver.php
│   └── AuthenticationScopeResolver.php
│
├── Realm/
│   ├── SecurityRealm.php
│   ├── SecurityRealmId.php
│   ├── SecurityRealmContext.php
│   ├── SecurityRealmDescriptor.php
│   ├── SecurityRealmResolver.php
│   ├── RealmTrustPolicy.php
│   └── RealmTrustRelationship.php
│
├── Policy/
│   ├── TenantAuthenticationPolicy.php
│   ├── PlatformAuthenticationPolicy.php
│   ├── RealmAuthenticationPolicy.php
│   ├── EffectiveAuthenticationPolicy.php
│   ├── TenantAuthenticationPolicyComposer.php
│   ├── TenantAuthenticationPolicyVersion.php
│   └── TenantPolicyCapabilitySet.php
│
├── Identity/
│   ├── TenantMembership.php
│   ├── TenantMembershipStatus.php
│   ├── MembershipSecurityVersion.php
│   └── TenantIdentityProviderResolver.php
│
├── Session/
│   ├── TenantBoundAuthenticationSession.php
│   ├── AuthenticationSessionScope.php
│   ├── TenantAuthenticationSwitchService.php
│   ├── TenantSwitchContext.php
│   └── TenantSwitchResult.php
│
├── Federation/
│   ├── TenantFederationRegistry.php
│   ├── TenantFederationConfiguration.php
│   └── TenantFederatedIdentityMapper.php
│
├── Isolation/
│   ├── AuthenticationTenantBoundary.php
│   ├── AuthenticationTenantIsolationGuard.php
│   ├── AuthenticationScopeGuard.php
│   └── SecurityRealmIsolationGuard.php
│
└── Runtime/
├── TenantAuthenticationExecutionContext.php
├── TenantAuthenticationContextResolver.php
├── TenantAuthenticationRuntimeResetter.php
└── TenantAuthenticationPolicyCache.php

## 372. Configuración conceptual

return [

'authentication' => [

'tenancy' => [

'enabled' => true,

'resolver' => 'domain',

'strict_isolation' => true,

],

'realms' => [

'customer' => [
'tenant' => 'required',
],

'tenant-admin' => [
'tenant' => 'required',
'assurance' => 'aal2',
],

'platform-admin' => [
'tenant' => 'forbidden',
'assurance' => 'aal2',
],

],

],

];

## 373. Tenant policy conceptual

[
'tenant' => 'acme',

'authentication' => [

'methods' => [
'passkey',
'oidc',
],

'mfa' => [
'required' => true,
],

'remember_me' => false,

'session' => [
'lifetime' => '8 hours',
],

],
];

## 374. Effective policy

Supongamos:

```text
Framework:
    Password allowed
    AAL1 minimum
```

Platform Enterprise:
MFA required

Admin Realm:

```text
    AAL2
    phishing-resistant required
```

ACME:
Passkey or OIDC

Emergency:
OIDC disabled
Resultado:
Effective ACME Admin Policy

Methods:
Passkey

Assurance:
AAL2

MFA:
Required

Phishing Resistance:
Required

OIDC:
Disabled

Remember-Me:
Depends on stricter applicable policy

## 375. Authentication Flow completo

REQUEST
│
▼
Firewall Resolution
│
▼
Security Realm
│
▼
Tenant Resolution
│
▼
Authentication Scope
(Tenant + Realm)
│
▼
Tenant Status
│
▼
Policy Composition
│
▼
Effective Authentication Policy
│
▼
Available Authenticator Set
│
▼
Credential Verification
│
▼
Global/Tenant Identity
│
▼
Membership Eligibility
│
▼
MFA / Risk / Device
│
▼
Final Security Version Check
│
▼
Tenant-bound Authentication Result
│
▼
Session / Token

## 376. Cross-Tenant SSO Flow

Platform Authentication
│
▼
Global Identity
│
▼
User selects Tenant B
│
▼
Resolve Tenant B
│
▼
Verify Membership B
│
▼
Resolve Realm B
│
▼
Compose Tenant B Policy
│
▼
Is current Authentication sufficient?
│
┌┴───────────────┐
▼                ▼
YES               NO
│                │
│             STEP-UP
│                │
└───────┬────────┘
▼
Tenant B Authentication Context

## 377. Tenant Switch Flow

Authenticated:

```text
Identity X
Tenant A
AAL1

        │
        ▼
```

Switch → Tenant B

│
▼
Membership B?
│
▼
Tenant B Active?
│
▼
Tenant B Policy:

```text
AAL2
        │
        ▼
Current AAL1 insufficient
        │
        ▼
Step-Up
        │
        ▼
AAL2 achieved
        │
        ▼
Rotate Session Context
        │
        ▼
Identity X
Tenant B
AAL2
```

## 378. Cross-Tenant Isolation Matrix

Artifact Tenant Bound Realm Bound
Tenant Session Yes Usually
Authentication Flow Yes when tenant realm Yes
MFA Challenge Yes when applicable Yes
Recovery Challenge Yes when tenant-specific Yes
Remember-Me Yes for tenant credential Yes/Policy
API Token According to token scope Yes/Audience
Device Trust Configurable, restricted by default Configurable
Risk Signal Scope-dependent Scope-dependent
Federated Mapping Model-dependent Provider-dependent
Tenant Credential Yes Credential-dependent

## 1. Laravel influence

Laravel aporta simplicidad mediante:

- guards
- providers
- middleware
- authentication contracts
- session-based authentication

pero un sistema multi-tenant avanzado suele requerir que aplicaciones/paquetes construyan varias de estas garantías por encima del Authentication estándar.
VoltStack incorporará estas fronteras como conceptos nativos.

## 2. Symfony influence

Symfony aporta:

- firewalls
- user providers
- access contexts
- authenticators
- stateless/stateful boundaries

que ofrecen una base conceptual sólida para Security Realms.

- VoltStack ampliará este modelo incorporando explícitamente:
- Tenant
- Realm
- Authentication Scope
- Membership
- Tenant Policy
- Security Version Vector
- Cross-Tenant Isolation

## 3. Diferenciador VoltStack

La ecuación será:

```text
Laravel-like developer ergonomics

+

Symfony-like firewall/security boundaries
+
native SaaS multi-tenancy
+
typed Authentication scopes
+
tenant policy composition
+
FrankenPHP-safe execution isolation
```

## 382. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Global Identity and Tenant Membership are separate

## 2. Tenant and Security Realm are separate

## 3. Every Authentication has an explicit Realm

## 4. Tenant-required Realms resolve Tenant before consuming tenant state

## 5. Authentication artifacts carry explicit scope

## 6. Null Tenant never means all tenants

## 7. Cross-Tenant Authentication reuse is denied by default

## 8. Cross-Realm Authentication reuse is denied by default

## 9. Realm trust is explicit and directional

## 10. Tenant switching is a new security evaluation, not a variable mutation

## 11. Platform SSO may identify the user but does not automatically authenticate the user into every Tenant

## 12. Tenant policy can strengthen but cannot weaken platform security floors

## 13. Policy composition uses typed security semantics

## 14. Tenant Sessions, Flows, MFA and Recovery preserve scope bindings

## 15. Security versions support fast invalidation

## 16. Federation callbacks recover Tenant from trusted Flow state

## 17. Tenant-specific caches always use canonical Tenant scope

## 18. Cross-Tenant administrative operations use separate privileged contracts

## 19. Current Tenant/Realm are execution-scoped

## 20. FrankenPHP workers never retain current Tenant between requests

## 21. Criterios de aceptación

El subsistema será considerado completo cuando:
22. soporte TenantReference;
23. soporte TenantContext;
24. soporte TenantAuthenticationContext;
25. soporte SecurityRealm;
26. soporte Realm Registry;
27. soporte Realm Resolution;
28. soporte Tenant Resolution;
29. soporte Tenant canonicalization;
30. soporte domain/subdomain resolution;
31. detecte conflicting Tenant signals;
32. soporte AuthenticationScope;
33. soporte Tenant Isolation Guard;
34. soporte Realm Isolation Guard;
35. distinga Global Identity y Tenant Membership;
36. soporte membership eligibility;
37. soporte Tenant status;
38. soporte Tenant Security Version;
39. soporte Membership Security Version;
40. soporte Security Version Vector;
41. soporte tenant-bound Sessions;
42. soporte realm-bound Sessions;
43. soporte Tenant Switching;
44. soporte Cross-Tenant SSO seguro;
45. soporte Realm Trust Policies;
46. soporte directional Realm trust;
47. soporte Tenant Authentication Policies;
48. soporte Platform Authentication Policies;
49. soporte Realm Policies;
50. soporte Emergency Policies;
51. soporte typed policy composition;
52. detecte unsatisfiable policies;
53. soporte Tenant Policy Versioning;
54. soporte Tenant Policy Cache;
55. soporte Tenant-specific Identity Providers;
56. soporte Tenant Federation;
57. soporte Tenant OIDC configuration;
58. preserve Federation Tenant binding;
59. soporte Tenant-local/global credentials;
60. soporte Tenant Passkeys;
61. soporte Tenant MFA;
62. soporte Tenant Remember-Me;
63. soporte Tenant API Tokens;
64. soporte Tenant Device Trust;
65. soporte Tenant Risk;
66. soporte Tenant Recovery;
67. soporte Tenant Rate Limits;
68. soporte Tenant Extension Policies;
69. soporte Tenant Audit context;
70. preserve scope en async operations;
71. preserve scope en Events;
72. soporte Cross-Tenant administration explícita;
73. soporte policy changes mid-flow;
74. soporte finalization revalidation;
75. soporte Tenant emergency policy;
76. evite cache contamination;
77. evite realm confusion;
78. evite tenant confusion;
79. sea seguro bajo FrankenPHP y fibers.
80. Regla arquitectónica final

La arquitectura deberá mantener:

```php
                    GLOBAL IDENTITY
                          │
                          ▼
                  TENANT MEMBERSHIP
                          │
                          ▼
                     TENANT
                          │
                          ▼
                   SECURITY REALM
                          │
                          ▼
                AUTHENTICATION SCOPE
                  (Tenant + Realm)
                          │
                          ▼
             EFFECTIVE AUTHENTICATION
                      POLICY
                          │
                          ▼
                    AUTHENTICATE
                          │
                          ▼
             TENANT-BOUND SECURITY
                       CONTEXT
                          │
                          ▼
                    AUTHORIZATION
```

La primera regla será:
VoltStack nunca deberá confundir la identidad global de una persona con su pertenencia a un tenant ni con el contexto dentro del cual fue autenticada.

La segunda:
Una Session, Token, Flow, MFA Challenge, Recovery Credential, Remember-Me Credential o Device Trust Record no podrá cruzar una frontera Tenant/Realm salvo que exista un mecanismo explícito diseñado para ello.

La tercera:
Cross-Tenant SSO significará reutilizar evidencia de identidad bajo una nueva evaluación de Membership, Realm, Policy, Risk y Assurance; nunca significará compartir indiscriminadamente el mismo contexto autenticado.

La cuarta:
Las políticas tenant podrán endurecer la seguridad de VoltStack, pero nunca reducir las garantías obligatorias establecidas por Framework, Platform o Security Realm.

La quinta:
Todo estado Tenant y Realm actual será execution-scoped; ningún singleton, facade, provider, cache local o worker FrankenPHP podrá conservar implícitamente el contexto del request anterior.

La sexta, especialmente importante para toda la arquitectura SaaS de VoltStack:
Global Identity ≠ Tenant Membership ≠ Authentication Scope ≠ Authorization Context.

Esta separación deberá permanecer como una de las invariantes fundamentales de todo el framework.
Siguiente documento
La continuación recomendable es:
`30_AUTHENTICATION_DISTRIBUTED_SYSTEM_CLUSTER_SESSION_COORDINATION_REVOCATION_CONSISTENCY_AND_MULTI_NODE_RUNTIME_SYSTEM.md`
Este documento llevaría Authentication al escenario de producción distribuida y formalizaría:

- Multi-Node Authentication
- Distributed Sessions
- Distributed Authentication Flows
- Distributed Rate Limiting
- Global Logout
- Session Revocation Propagation
- Token Revocation
- Credential Revocation
- Security Version Propagation
- Policy Version Propagation
- Tenant Security Changes
- Cache Consistency
- Node Configuration Consistency
- Distributed Locks
- Single-Use Challenge Coordination

Replay Protection Across Nodes
Race Conditions
Eventual vs Strong Consistency
Failure Modes
Network Partitions
Node Failover
Redis/Distributed Stores
Rolling Deployments
FrankenPHP Worker Coordination
Multi-Region Authentication
Con 29 ya queda definida la frontera lógica de seguridad. El 30 definiría cómo mantener esa frontera cuando VoltStack deje de ejecutarse en un solo servidor y Authentication opere simultáneamente en múltiples workers, nodos y regiones.
