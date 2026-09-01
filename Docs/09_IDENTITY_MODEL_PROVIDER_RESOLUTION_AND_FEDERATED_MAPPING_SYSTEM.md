# VoltStack Authentication System

## 09 — Identity Model, Provider Resolution and Federated Mapping System

- **Archivo:** `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del modelo de identidad, providers y resolución federada  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de representar, localizar, resolver, refrescar y mapear las **identidades autenticables** dentro de VoltStack.

El objetivo es construir un modelo independiente del concepto tradicional:

```text
User
+
email
+
password
+
database
```

VoltStack deberá poder autenticar:

```text
Human Users
Administrators
Employees
Customers
Service Accounts
API Clients
Machines
Devices
Workloads
Federated Identities
External Subjects
Temporary Identities
```

utilizando múltiples fuentes de identidad.

---

## 2. Principio fundamental

La arquitectura utilizará:

```text
Identity
```

como concepto general.

Por tanto:

```text
User
    is an Identity

Service
    is an Identity

Machine
    is an Identity

API Client
    is an Identity
```

pero:

```text
Identity
    is not necessarily a User
```

---

## 3. Separación conceptual

Deberán distinguirse:

```text
IdentityClaim
IdentityReference
IdentityIdentifier
Identity
IdentityProvider
IdentityResolver
FederatedSubject
FederatedIdentityMapping
```

Relación general:

```text
Untrusted Claim
      ↓
Claim Normalization
      ↓
Identity Provider Resolution
      ↓
Identity Provider
      ↓
Canonical Identity
```

o:

```text
Verified External Assertion
      ↓
Issuer + Subject
      ↓
Federated Identity Mapper
      ↓
Canonical Identity
```

---

## 4. Identity

Una `Identity` representa una entidad canónica reconocida por la aplicación o por un dominio confiable de identidad.

No representa por sí misma:

```text
authenticated state
session
permissions
roles
authorization decision
```

Una Identity existe independientemente de que actualmente esté autenticada.

---

## 5. IdentityInterface

Contrato mínimo recomendado:

```php
interface IdentityInterface
{
    public function identifier(): IdentityIdentifier;

    public function type(): IdentityType;
}
```

El contrato central deberá mantenerse deliberadamente pequeño.

---

## 6. Qué NO deberá incluir IdentityInterface

No deberá exigir:

```php
password();
email();
roles();
permissions();
tenant();
session();
token();
```

porque no todas las identidades poseen estos conceptos.

---

## 7. IdentityIdentifier

Será el identificador canónico interno de una Identity.

Ejemplos:

```text
UUID
ULID
integer
string
compound reference
```

VoltStack no deberá asumir:

```text
int $id
```

como formato universal.

---

## 8. IdentityIdentifier como Value Object

Conceptualmente:

```php
final readonly class IdentityIdentifier
{
    public function __construct(
        public string $value,
    ) {}
}
```

Podrá incorporar además namespace/type si resulta necesario.

---

## 9. Global uniqueness

VoltStack deberá decidir si:

```text
IdentityIdentifier
```

es globalmente único o únicamente dentro de un provider/realm.

La recomendación es utilizar una referencia completa para comparaciones cross-provider.

---

## 10. IdentityReference

Podrá existir:

```text
IdentityReference
```

compuesta por:

```text
provider
realm
identity type
identifier
tenant scope
```

Ejemplo:

```text
provider: users
type: human
id: 01JX...
tenant: acme
```

---

## 11. IdentityReference model

Conceptualmente:

```php
final readonly class IdentityReference
{
    public function __construct(
        public string $provider,
        public IdentityType $type,
        public IdentityIdentifier $identifier,
        public ?TenantIdentifier $tenant = null,
        public ?string $realm = null,
    ) {}
}
```

---

## 12. Identity equality

Para comparaciones de seguridad deberá preferirse:

```text
IdentityReference equality
```

sobre:

```text
raw id equality
```

Ejemplo:

```text
provider A / user 15
```

no equivale necesariamente a:

```text
provider B / user 15
```

---

## 13. Tenant-aware equality

Igualmente:

```text
tenant A / user 15
```

no necesariamente equivale a:

```text
tenant B / user 15
```

---

## 14. IdentityType

Tipos iniciales posibles:

```text
human
administrator
customer
employee
service
machine
workload
device
client
federated
temporary
```

Debe ser extensible.

---

## 15. IdentityType no es Authorization Role

Debe mantenerse:

```text
IdentityType = administrator
```

como clasificación de identidad cuando corresponda.

No deberá confundirse con:

```text
Role = administrator
```

del sistema de Authorization.

---

## 16. Specialized Identity interfaces

Para capacidades opcionales podrán existir contratos adicionales:

```php
interface EmailAwareIdentityInterface
{
    public function email(): ?string;
}
```

```php
interface DisplayNameAwareIdentityInterface
{
    public function displayName(): ?string;
}
```

---

## 17. Capability interfaces

Otros posibles:

```text
TenantAwareIdentityInterface
SecurityVersionAwareIdentityInterface
CredentialOwnerInterface
FederatedIdentityAwareInterface
DeviceAwareIdentityInterface
```

Solo cuando exista valor arquitectónico real.

---

## 18. Identity attributes

Metadata adicional deberá mantenerse fuera del contrato mínimo.

Podrá utilizarse:

```text
IdentityAttributeBag
```

o adapters específicos.

---

## 19. Attribute trust

Los atributos de una Identity cargada desde un provider confiable se consideran parte de esa representación.

Sin embargo, no todos deberán propagarse automáticamente a:

```text
AuthenticationContext
AuthorizationContext
token claims
```

---

## 20. IdentityClaim

Una `IdentityClaim` representa una afirmación utilizada para intentar encontrar una Identity.

Ejemplos:

```text
email
username
phone
employee number
OIDC subject
client id
certificate subject
service name
external identifier
```

---

## 21. Claim no es Identity

```text
IdentityClaim('email', 'a@example.com')
```

solo significa:

> buscar una Identity asociada a este identificador.

No significa que esa identidad exista ni que esté autenticada.

---

## 22. IdentityClaim model

Conceptualmente:

```php
final readonly class IdentityClaim
{
    public function __construct(
        public IdentityClaimType $type,
        public string $value,
        public array $attributes = [],
    ) {}
}
```

---

## 23. IdentityClaimType

Será extensible.

Ejemplos:

```text
email
username
phone
external_subject
client_id
service_id
certificate_subject
employee_number
```

---

## 24. Identity claim namespaces

Para evitar colisiones puede resultar útil:

```text
email
username
oidc:subject
saml:name_id
service:client_id
```

aunque la API pública deberá mantenerse clara.

---

## 25. IdentityClaimNormalizer

Antes de resolver una claim podrá utilizarse:

```text
IdentityClaimNormalizer
```

---

## 26. Normalización depende del claim type

Ejemplo email:

```text
trim outer transport whitespace
domain canonicalization when appropriate
Unicode normalization
```

Ejemplo username:

```text
application-defined case handling
```

No deberá existir una normalización destructiva universal.

---

## 27. No password-style normalization

Claims y credentials deben permanecer separados.

La normalización de email no implica modificar password.

---

## 28. Unicode normalization

VoltStack deberá permitir políticas que reduzcan problemas de:

```text
confusable characters
multiple Unicode representations
case inconsistencies
```

especialmente para usernames.

---

## 29. Claim canonicalization

Podrá producirse:

```text
Presented Claim
      ↓
CanonicalIdentityClaim
```

antes de lookup.

---

## 30. CanonicalIdentityClaim

La representación canonical deberá registrar:

```text
type
canonical value
normalizer
```

sin conservar innecesariamente variaciones sensibles.

---

## 31. IdentityProvider

Un `IdentityProvider` representa una fuente confiable de Identity.

Ejemplos:

```text
DatabaseIdentityProvider
OrmIdentityProvider
LdapIdentityProvider
ActiveDirectoryIdentityProvider
RemoteApiIdentityProvider
FederatedIdentityProvider
MemoryIdentityProvider
ServiceIdentityProvider
```

---

## 32. IdentityProviderInterface

Contrato conceptual:

```php
interface IdentityProviderInterface
{
    public function resolve(
        IdentityResolutionRequest $request
    ): IdentityResolutionResult;
}
```

Esto es preferible a un simple:

```php
findByIdentifier(string $id)
```

porque permite contexto adicional.

---

## 33. IdentityResolutionRequest

Podrá contener:

```text
claim
identity reference
tenant
realm
authenticator
firewall
verified external subject
resolution mode
```

---

## 34. Resolution modes

Podrán existir:

```text
BY_CLAIM
BY_REFERENCE
BY_FEDERATED_SUBJECT
BY_CREDENTIAL_DERIVED_SUBJECT
REFRESH
```

---

## 35. IdentityResolutionResult

Estados:

```text
RESOLVED
NOT_FOUND
AMBIGUOUS
DISABLED
UNAVAILABLE
ERROR
```

---

## 36. RESOLVED

Produce una Identity canónica.

---

## 37. NOT_FOUND

No existe Identity coincidente.

Externamente puede mapearse de forma genérica para prevenir enumeration.

---

## 38. AMBIGUOUS

Una claim produjo más de una Identity donde se esperaba una sola.

Ejemplo:

```text
duplicate email
```

en un provider que exige unicidad.

Esto deberá fallar de forma cerrada.

---

## 39. DISABLED

La Identity existe, pero no puede participar en Authentication según estado.

---

## 40. UNAVAILABLE

El provider no está disponible.

Ejemplos:

```text
LDAP timeout
database unavailable
remote API failure
```

---

## 41. ERROR

Fallo interno o configuración inválida.

---

## 42. Provider no verifica Credential

Un IdentityProvider deberá resolver:

```text
who
```

No verificar necesariamente:

```text
proof
```

Ejemplo:

```text
DatabaseIdentityProvider
    loads User
```

Mientras:

```text
PasswordVerifier
    checks password
```

---

## 43. Provider exceptions

Un provider puede necesitar exponer material de autenticación mediante repositorios especializados, pero no deberá convertirse en un megaobjeto.

---

## 44. Credential storage separation

Preferencia:

```text
IdentityProvider
    resolves Identity

CredentialRepository
    resolves credential records
```

en vez de:

```text
IdentityProvider
    handles everything
```

---

## 45. DatabaseIdentityProvider

Caso tradicional:

```text
IdentityClaim(email)
       ↓
database query
       ↓
UserIdentity
```

---

## 46. ORM independence

El contrato no deberá depender directamente de:

```text
Eloquent
Doctrine
VoltStack ORM
```

Podrán existir adapters específicos.

---

## 47. VoltStackOrmIdentityProvider

Un adapter podrá usar el ORM nativo.

Ejemplo conceptual:

```php
final class VoltStackOrmIdentityProvider
    implements IdentityProviderInterface
{
}
```

---

## 48. Eloquent-compatible adapter

Si VoltStack ofrece compatibilidad/interoperabilidad, podrá existir un provider que adapte modelos estilo Laravel.

Pero el Core no dependerá de Eloquent.

---

## 49. LDAP Identity Provider

Podrá resolver mediante:

```text
username
DN
employee id
email
```

según configuración.

---

## 50. LDAP lookup vs LDAP credential validation

Deberán separarse cuando sea posible:

```text
LDAP Identity Provider
    resolve entry

LDAP Credential Verifier
    bind/verify credential
```

aunque algunos servidores integren ambas operaciones.

---

## 51. Active Directory

Será una especialización/configuración de provider LDAP o package dedicado.

Podrá mapear:

```text
UPN
sAMAccountName
objectGUID
SID
```

a una IdentityReference canónica.

---

## 52. Remote API Identity Provider

Podrá resolver Identity mediante servicio externo.

Debe soportar:

```text
timeouts
circuit breaker
retry where safe
caching where safe
```

---

## 53. Provider failures nunca autentican

Si un servicio remoto no responde:

```text
UNAVAILABLE
```

No:

```text
assume cached identity valid indefinitely
```

salvo una estrategia explícita y segura.

---

## 54. MemoryIdentityProvider

Útil para:

```text
testing
development
small system identities
```

No será necesariamente recomendado para producción general.

---

## 55. ServiceIdentityProvider

Resolverá:

```text
ServiceIdentity
MachineIdentity
WorkloadIdentity
APIClientIdentity
```

sin obligar a almacenarlas en tabla `users`.

---

## 56. IdentityProviderDescriptor

Cada provider tendrá metadata:

```text
name
service
supported claim types
supported identity types
tenant awareness
federation awareness
refresh capability
priority
realm
```

---

## 57. IdentityProviderRegistry

VoltStack tendrá:

```text
IdentityProviderRegistry
```

---

## 58. Registry contract

Conceptualmente:

```php
interface IdentityProviderRegistryInterface
{
    public function get(string $name): IdentityProviderDescriptor;

    public function has(string $name): bool;
}
```

---

## 59. Registry compilation

Durante bootstrap:

```text
registration
    ↓
validation
    ↓
provider map compilation
    ↓
freeze
```

---

## 60. ProviderResolver

Cuando más de un provider pueda manejar una operación:

```text
IdentityProviderResolver
```

seleccionará el correcto.

---

## 61. Provider resolution inputs

Podrá considerar:

```text
firewall
authenticator
claim type
identity type
tenant
realm
federated issuer
authentication operation
explicit provider binding
```

---

## 62. Provider selection must be deterministic

No deberá utilizar:

```text
first provider that returns a user
```

como estrategia general.

---

## 63. Por qué "try every provider" es peligroso

Supongamos:

```text
Provider A:
    corporate employees

Provider B:
    local administrators
```

Buscar una misma claim en ambos hasta encontrar algo puede generar:

```text
identity confusion
realm crossing
privilege boundary mistakes
timing leakage
```

---

## 64. Provider explicit binding

Recomendación común:

```text
Firewall web
    provider = users

Firewall admin
    provider = administrators

Firewall service
    provider = services
```

---

## 65. Multi-provider Firewall

Cuando sea necesario:

```text
enterprise
    providers:
        ldap
        local_breakglass
```

la selección deberá usar reglas explícitas.

---

## 66. ProviderSelectionPolicy

Podrá existir:

```php
interface IdentityProviderSelectionPolicyInterface
{
    public function resolve(
        IdentityProviderResolutionContext $context
    ): IdentityProviderResolution;
}
```

---

## 67. Provider precedence

Podrá declararse:

```text
federated
ldap
local
```

pero una falla de autenticación en uno no deberá provocar downgrade silencioso a otro.

---

## 68. Provider fallback semantics

Debe distinguirse:

```text
provider NOT_APPLICABLE
```

de:

```text
provider RESOLVED identity but credential failed
```

El segundo no permitirá probar otra identidad equivalentemente nombrada en otro provider.

---

## 69. Provider lock

Una vez resuelta una Identity:

```text
provider selection
```

deberá quedar bloqueada para esa operación salvo flow formal.

---

## 70. Realm

VoltStack podrá utilizar el concepto:

```text
IdentityRealm
```

para representar un namespace lógico de identidades.

Ejemplos:

```text
customers
employees
administrators
services
external-partners
```

---

## 71. Realm vs Tenant

No son equivalentes.

```text
Realm
    identity namespace / authority domain

Tenant
    application/business isolation scope
```

---

## 72. Realm example

```text
Realm employees
    LDAP provider

Realm customers
    database provider
```

pueden coexistir dentro del mismo tenant.

---

## 73. Realm identity uniqueness

Una claim puede ser única dentro de un Realm pero no globalmente.

---

## 74. IdentityAuthority

Podrá existir conceptualmente:

```text
IdentityAuthority
```

para indicar quién tiene autoridad sobre la Identity.

Ejemplo:

```text
local application
corporate LDAP
external OIDC issuer
cloud workload platform
```

---

## 75. Authority vs Provider

```text
Provider
    implementation used to retrieve/map identity

Authority
    source considered authoritative for identity facts
```

Pueden coincidir, pero no siempre.

---

## 76. Federated Identity

Una identidad federada proviene de una autoridad externa.

Ejemplo:

```text
OIDC
SAML
enterprise SSO
```

VoltStack no deberá usar directamente un email externo como ID local confiable.

---

## 77. FederatedSubject

Se deberá modelar formalmente:

```text
FederatedSubject
```

---

## 78. FederatedSubject model

Conceptualmente:

```php
final readonly class FederatedSubject
{
    public function __construct(
        public string $issuer,
        public string $subject,
        public string $protocol,
    ) {}
}
```

---

## 79. Issuer + Subject

La identidad externa canónica se representa normalmente por:

```text
issuer
+
subject
```

No solo:

```text
email
```

---

## 80. Por qué email no es suficiente

Un email puede:

```text
change
be recycled
appear in multiple providers
be unverified
be controlled by claim mapping
```

Por tanto, el binding federado deberá usar un identificador estable del issuer.

---

## 81. FederatedIdentityMapping

VoltStack deberá mapear:

```text
FederatedSubject
      ↓
FederatedIdentityMapper
      ↓
Local IdentityReference
```

---

## 82. Mapping model

Podrá persistirse:

```text
issuer
subject
provider
local identity id
tenant/realm
created at
last seen
metadata
```

---

## 83. Mapping uniqueness

Debe garantizarse unicidad segura:

```text
issuer + subject
```

dentro del ámbito configurado.

---

## 84. FederatedIdentityMapperInterface

Conceptualmente:

```php
interface FederatedIdentityMapperInterface
{
    public function resolve(
        FederatedSubject $subject,
        FederatedIdentityMappingContext $context
    ): FederatedIdentityMappingResult;
}
```

---

## 85. Mapping outcomes

```text
MAPPED
NOT_MAPPED
AUTO_PROVISION_REQUIRED
CONFLICT
DISABLED
ERROR
```

---

## 86. Existing mapping

Caso ideal:

```text
verified OIDC issuer+sub
    ↓
existing mapping
    ↓
local Identity
```

---

## 87. Auto-provisioning

VoltStack podrá soportar creación automática de Identity.

Sin embargo, esto deberá ser una policy explícita.

---

## 88. Auto-provisioning risks

Debe controlar:

```text
issuer allowlist
tenant invitation
domain restrictions
email verification claims
realm assignment
default identity type
duplicate linking
```

---

## 89. Never auto-provision arbitrary issuer

Input externo no podrá elegir libremente:

```text
issuer = attacker.example
```

y provocar creación de cuenta.

---

## 90. Just-in-Time Provisioning

Podrá existir:

```text
FederatedIdentityProvisioner
```

para JIT provisioning.

---

## 91. Provisioning transaction

La creación de Identity deberá ser atómica respecto al mapping cuando sea posible.

Evitar:

```text
create user
    ↓
mapping fails
    ↓
orphan identity
```

---

## 92. Linking existing identities

Una identidad local existente podrá enlazarse con un provider externo.

Esto es una operación sensible distinta de login.

---

## 93. Identity linking

No deberá realizarse automáticamente solo porque:

```text
external email == local email
```

salvo política explícita muy controlada.

---

## 94. Account linking attacks

Un issuer malicioso o cuenta externa comprometida podría afirmar:

```text
victim@example.com
```

Por ello linking por email requiere condiciones estrictas.

---

## 95. Safe linking flow

Ejemplo recomendado:

```text
authenticated local session
        +
fresh reauthentication
        +
verified external login
        ↓
explicit link
```

---

## 96. Admin-driven linking

También puede existir en entornos enterprise, con auditoría.

---

## 97. IdentityLink

Podrá representar una relación entre:

```text
local Identity
```

y:

```text
FederatedSubject
```

---

## 98. Multiple federated links

Una Identity puede tener:

```text
Google
Microsoft
Corporate OIDC
SAML
```

simultáneamente.

---

## 99. Unlinking

Eliminar un provider externo deberá verificar que la cuenta conserve mecanismos de recuperación/autenticación suficientes cuando la policy lo exija.

---

## 100. Federated identity conflict

Si:

```text
issuer+subject
```

ya está vinculado a otra Identity:

```text
CONFLICT
```

Nunca reasignar silenciosamente.

---

## 101. Subject reuse

Un issuer bien diseñado no debería reciclar `sub`, pero VoltStack deberá conservar mapping estable y auditado.

---

## 102. Issuer canonicalization

El issuer deberá compararse usando reglas específicas del protocolo.

No aplicar transformaciones arbitrarias.

---

## 103. OIDC mapping

Flujo:

```text
OIDC ID Token verified
        ↓
issuer
subject
        ↓
FederatedSubject
        ↓
FederatedIdentityMapper
        ↓
IdentityReference
        ↓
IdentityProvider
        ↓
Identity
```

---

## 104. SAML mapping

Podrá utilizar:

```text
issuer/entity ID
NameID
persistent identifier
```

según configuración.

---

## 105. SAML email mapping

Si el IdP solo ofrece email, deberá reconocerse que el binding puede ser menos estable y requerir políticas adicionales.

---

## 106. External attribute mapping

Un IdP puede proporcionar:

```text
name
email
department
groups
locale
```

Estos datos deberán pasar por:

```text
FederatedAttributeMapper
```

---

## 107. Attribute Mapper

Será responsable de mapear claims externos a atributos locales permitidos.

---

## 108. Attribute allowlist

No deberá copiar:

```text
all claims
```

automáticamente a Identity.

---

## 109. External group claims

Pueden utilizarse posteriormente por provisioning o Authorization, pero deberán manejarse explícitamente.

No convertir grupos externos automáticamente en Roles administrativos salvo configuración segura.

---

## 110. Federated identity refresh

En cada login externo, VoltStack podrá actualizar determinados atributos.

Políticas:

```text
NEVER
ON_LOGIN
IF_CHANGED
AUTHORITATIVE
SELECTIVE
```

---

## 111. Local vs external authority

Ejemplo:

```text
display name
    external authoritative

billing settings
    local authoritative
```

El mapper deberá respetarlo.

---

## 112. Identity State

Una Identity podrá tener estado relativo a Authentication.

Ejemplos:

```text
ACTIVE
DISABLED
LOCKED
SUSPENDED
PENDING
DELETED
```

---

## 113. IdentityStatus

No necesariamente forma parte de `IdentityInterface`.

Podrá obtenerse mediante:

```text
IdentityAuthenticationStateProvider
```

o capability.

---

## 114. Disabled Identity

Aunque una credential sea correcta:

```text
identity disabled
    ↓
authentication rejected
```

---

## 115. Locked Identity

El significado deberá diferenciarse de rate limiting.

Podría representar bloqueo administrativo o temporal.

---

## 116. Avoid permanent lockout as brute-force default

VoltStack deberá preferir rate limiting y risk controls antes que facilitar ataques de lockout contra usuarios.

---

## 117. Pending Identity

Ejemplo:

```text
registration created
email not verified
```

Puede existir Identity pero con restricciones de Authentication.

---

## 118. Deleted Identity

Una session antigua no deberá reconstruir una Identity eliminada.

---

## 119. Identity security version

VoltStack podrá soportar:

```text
securityVersion
```

para invalidar Authentication state.

---

## 120. Security version cases

Incrementar tras:

```text
password compromise
account recovery
forced logout
MFA reset
security incident
global credential rotation
```

---

## 121. IdentityVersion vs SecurityVersion

Diferenciar:

```text
IdentityVersion
    cambios generales

SecurityVersion
    cambios que afectan autenticación
```

---

## 122. Session reconstruction

La session puede guardar:

```text
identity reference
security version
```

Luego:

```text
stored version
    vs
current version
```

determina si puede reconstruirse Context.

---

## 123. Identity refresh

Una Identity cargada al iniciar sesión puede necesitar refresh en requests posteriores.

---

## 124. RefreshStrategy

Posibles:

```text
ALWAYS
NEVER
VERSION_BASED
INTERVAL_BASED
EVENT_INVALIDATED
PROVIDER_SPECIFIC
```

---

## 125. ALWAYS

Carga provider cada request.

Ventaja:

```text
fresh state
```

Costo:

```text
database/provider query
```

---

## 126. NEVER

Usa snapshot persistido.

Solo apropiado para casos concretos y con mecanismos de invalidación suficientes.

---

## 127. VERSION_BASED

Comprueba versión ligera.

---

## 128. INTERVAL_BASED

Refresca cada cierto tiempo.

---

## 129. EVENT_INVALIDATED

Utiliza cache/session hasta recibir invalidación.

Más complejo en sistemas distribuidos.

---

## 130. Provider-specific refresh

LDAP/OIDC/remote systems pueden necesitar estrategias distintas.

---

## 131. IdentityRefresher

Podrá existir:

```php
interface IdentityRefresherInterface
{
    public function refresh(
        IdentityReference $identity
    ): IdentityRefreshResult;
}
```

---

## 132. Refresh outcomes

```text
REFRESHED
UNCHANGED
NOT_FOUND
DISABLED
STALE
ERROR
```

---

## 133. Stale Identity

Puede ocurrir si los datos cargados ya no cumplen condiciones necesarias.

---

## 134. Request memoization

Dentro de una request:

```text
IdentityReference user:10
```

deberá resolverse una sola vez cuando sea seguro.

---

## 135. Identity Map integration

Si el sistema ORM ya implementa Identity Map, Auth podrá aprovecharla a través del provider, pero no depender directamente de su implementación.

---

## 136. Cross-request caching

Metadata de Identity puede cachearse si:

```text
versioning
TTL
invalidation
tenant scoping
```

están correctamente definidos.

---

## 137. Authentication-sensitive cache

Campos como:

```text
disabled
security version
credential status
```

requieren freshness más estricta que:

```text
display name
avatar
locale
```

---

## 138. Split identity metadata

Puede ser útil distinguir:

```text
IdentityCore
IdentitySecurityState
IdentityProfile
```

para optimizar cargas.

---

## 139. IdentityCore

Podría contener:

```text
identifier
type
provider
realm
tenant scope
```

---

## 140. IdentitySecurityState

Podría contener:

```text
authentication enabled
security version
account status
credential policy version
```

---

## 141. IdentityProfile

Podría contener atributos no críticos:

```text
display name
avatar
locale
```

---

## 142. Lazy profile loading

Authentication no necesita cargar todo el profile para autenticar.

---

## 143. IdentityProvider capabilities

Descriptor podrá declarar:

```text
CLAIM_LOOKUP
REFERENCE_LOOKUP
REFRESH
FEDERATED_MAPPING
TENANT_AWARE
SERVICE_IDENTITIES
BULK_LOOKUP
```

---

## 144. Claim support

Ejemplo:

```text
users provider:
    email
    username

services provider:
    client_id

LDAP:
    username
    employee_number
```

---

## 145. Unsupported Claim

Si un provider no soporta una claim:

```text
NOT_APPLICABLE
```

puede utilizarse en provider resolution antes de selección final.

---

## 146. ProviderSelectionResult

Podrá distinguir:

```text
SELECTED
NONE
AMBIGUOUS
CONFLICT
ERROR
```

---

## 147. Provider ambiguity

Ejemplo:

```text
two providers claim authoritative ownership
of same Realm + claim type
```

deberá detectarse en bootstrap si es posible.

---

## 148. Multi-provider discovery

Para algunos sistemas enterprise podrá ser necesario elegir provider según:

```text
email domain
tenant
organization
realm
explicit enterprise connection
```

---

## 149. Domain-based provider routing

Ejemplo:

```text
@corp.example
    → corporate LDAP

@partner.example
    → partner OIDC
```

Esto deberá basarse en configuración confiable.

---

## 150. Domain discovery privacy

Un login público no deberá necesariamente revelar qué provider utiliza una organización si eso representa information leakage significativa.

---

## 151. Home Realm Discovery

VoltStack podrá soportar un:

```text
HomeRealmDiscoveryService
```

para enterprise SSO.

---

## 152. Home Realm Discovery input

Podrá considerar:

```text
tenant
email domain
explicit organization
subdomain
configured connection
```

---

## 153. Home Realm Discovery output

```text
IdentityRealm
Provider
Federation Connection
```

---

## 154. Home Realm security

No permitir que un dominio arbitrario produzca un issuer/URL remoto dinámico sin configuración previa.

---

## 155. Multi-tenancy

Identity resolution deberá ser tenant-aware desde el inicio.

---

## 156. Tenant scoped Identity

Ejemplo:

```text
Tenant A
    user@example.com → Identity 100

Tenant B
    user@example.com → Identity 205
```

Esto es válido en modelos tenant-isolated.

---

## 157. Global Identity

Otro modelo:

```text
global Identity 100
    memberships:
        tenant A
        tenant B
```

Authentication deberá poder soportar ambos.

---

## 158. TenantIdentityMode

Podrá existir:

```text
TENANT_SCOPED
GLOBAL
HYBRID
EXTERNAL
```

---

## 159. TenantScoped provider

Consulta:

```text
tenant_id + canonical claim
```

---

## 160. Global provider

Consulta Identity global y luego establece tenant context por otra relación.

---

## 161. Authentication vs tenant membership

Que una Identity global exista no significa que deba acceder a un tenant.

La pertenencia y capacidad final pertenecen principalmente a Authorization / tenant policy.

---

## 162. Authentication-level tenant validity

Authentication sí deberá impedir estados incoherentes como:

```text
session cryptographically bound to tenant A
used under tenant B
```

---

## 163. Tenant claim from IdP

Un OIDC provider puede afirmar organization/tenant.

Esto no deberá aceptarse automáticamente como tenant local sin mapping.

---

## 164. TenantMapping

Podrá existir:

```text
ExternalOrganizationMapper
```

para convertir external org IDs en `TenantIdentifier`.

---

## 165. Tenant mapping conflict

Un external tenant/org desconocido deberá fallar o pasar por provisioning controlado.

---

## 166. Service identities multi-tenant

Una ServiceIdentity puede ser:

```text
global platform service
tenant service
tenant integration client
```

Debe quedar explícito en IdentityReference.

---

## 167. API Client identity

No deberá necesariamente mapearse a un User.

Ejemplo:

```text
client: accounting-integration
```

---

## 168. Client vs Service

Podrán diferenciarse:

```text
API Client
    external application

Service Identity
    internal service/workload
```

según dominio.

---

## 169. Machine Identity

Puede representar:

```text
server
node
VM
container host
appliance
```

---

## 170. Workload Identity

Representa una instancia lógica de software:

```text
Kubernetes workload
cloud function
queue worker
deployment task
```

---

## 171. Device Identity

Puede ser objeto autenticable independiente o metadata ligada a User Authentication.

No deberán confundirse ambos usos.

---

## 172. Device-as-Identity

Ejemplo:

```text
IoT device
```

es la propia Identity.

---

## 173. Device-as-Context

Ejemplo:

```text
user logging in from trusted laptop
```

el device es parte del AuthenticationContext, no la Identity principal.

---

## 174. Composite identities

VoltStack deberá evitar modelar arbitrariamente:

```text
user + device
```

como una nueva Identity si basta con Context binding.

---

## 175. Temporary Identity

Podrá existir para:

```text
guest invitation
temporary operator
one-time access
ephemeral workload
```

con expiración clara.

---

## 176. Identity expiration

Algunas Identity pueden tener:

```text
validFrom
validUntil
```

además de credential expiration.

---

## 177. Expired Identity

No deberá autenticarse aunque una credential siga siendo criptográficamente válida.

---

## 178. IdentityAuthenticationStateChecker

Se recomienda separar:

```text
Identity resolution
```

de:

```text
authentication eligibility
```

mediante un:

```text
IdentityAuthenticationStateChecker
```

---

## 179. StateChecker responsibilities

Podrá evaluar:

```text
active
disabled
deleted
validity window
security state
tenant-level authentication restrictions
```

---

## 180. StateChecker no ejecuta Authorization

No decidirá:

```text
may view dashboard
may delete invoice
```

---

## 181. Pre-auth checks

Antes de credential verification pueden realizarse ciertos checks.

Pero debe considerarse timing leakage.

---

## 182. Post-auth checks

Algunos checks podrán realizarse después de credential verification para ocultar account state externamente.

Ejemplo:

```text
disabled account
```

puede responder igual que invalid credentials públicamente.

---

## 183. PreAuthenticationIdentityChecker

Puede existir para restricciones técnicas.

---

## 184. PostAuthenticationIdentityChecker

Puede evaluar estado después de demostrar credencial.

Esto recuerda conceptos de UserChecker de Symfony, pero con separación más explícita.

---

## 185. Enumeration safety

Mensajes públicos deberán evitar:

```text
account exists but disabled
email exists
provider found account
```

salvo UX/policy específica.

---

## 186. Identity resolution timing

Lookup de identity inexistente puede ser más rápido que credencial real.

El Password subsystem podrá compensar con dummy verification.

---

## 187. Provider query limits

Claims públicas no deberán permitir queries excesivamente costosas.

Ejemplo:

```text
wildcard username search
```

no deberá formar parte de login normal.

---

## 188. Exact lookup

Authentication providers deberán favorecer:

```text
exact canonical lookup
```

con índices apropiados.

---

## 189. Identity uniqueness constraints

Fields de login como email/username deberán tener constraints acordes al scope:

```text
unique globally
unique per tenant
unique per realm
```

---

## 190. Database constraints

La seguridad no deberá depender únicamente de validación de aplicación.

Cuando corresponda, usar constraints DB para unicidad.

---

## 191. Race during provisioning

Dos logins federados concurrentes podrían intentar crear la misma Identity.

Debe utilizarse:

```text
unique mapping constraint
transaction
retry safe resolution
```

---

## 192. Federated mapping atomicity

Ideal:

```text
create identity
+
create mapping
```

en una transacción cuando comparten storage.

---

## 193. Cross-store mapping

Si provider/mapping viven en distintos stores, usar:

```text
idempotency
compensation
unique correlation
```

---

## 194. Identity lifecycle events

Podrán existir:

```text
IdentityResolved
IdentityResolutionFailed
IdentityRefreshed
IdentityDisabledDetected
FederatedIdentityMapped
FederatedIdentityProvisioned
FederatedIdentityLinked
FederatedIdentityUnlinked
```

---

## 195. Authentication events vs Identity lifecycle

No todo cambio de Identity pertenece al Auth module.

Profile management puede vivir en otro dominio.

Authentication solo deberá emitir eventos relevantes a su proceso.

---

## 196. Provider observability

Spans:

```text
auth.identity.resolve
auth.identity.refresh
auth.identity.federated_map
```

---

## 197. Metrics

Ejemplos:

```text
identity_resolution_total
identity_resolution_not_found_total
identity_resolution_ambiguous_total
identity_resolution_duration
identity_provider_error_total
federated_mapping_total
federated_provisioning_total
```

---

## 198. Metric labels

Seguros y de baja cardinalidad:

```text
provider
claim_type
identity_type
status
realm
```

No:

```text
email
user id
subject
```

como labels globales.

---

## 199. Audit

Eventos sensibles:

```text
federated link created
federated link removed
JIT identity provisioned
identity disabled login attempt
provider conflict
cross-tenant identity mismatch
```

---

## 200. Provider logs

Nunca deberán registrar de forma indiscriminada:

```text
LDAP password
OIDC token
SAML assertion
private credential material
```

---

## 201. Claim logging

Incluso emails/usernames pueden ser PII.

Logging deberá usar política de privacidad/configuración, no imprimir claims por defecto en todos los traces.

---

## 202. Identity fingerprints

Para correlación puede usarse una referencia interna segura cuando corresponda.

---

## 203. Provider cache

Podrá cachearse:

```text
provider metadata
identity core
federated mapping
public federation metadata
```

con políticas adecuadas.

---

## 204. Federated mapping cache

`issuer + subject → IdentityReference` puede ser buen candidato si existe invalidación tras unlink.

---

## 205. Negative mapping cache

Debe tener TTL corto y considerar provisioning concurrente.

---

## 206. Security-state caching

Requiere TTL/invalidation más estricto.

---

## 207. LDAP caching

Deberá evitar conservar passwords/bind credentials en caches de aplicación.

---

## 208. Remote provider cache outage

Una cache miss no debe transformar un provider failure en identity absence sin distinguirlos.

---

## 209. Identity Provider resilience

Remote providers podrán utilizar:

```text
timeout
circuit breaker
bounded retry
connection pooling
```

---

## 210. Retry safety

Lookup read-only puede reintentarse cuando sea seguro.

Provisioning/linking requiere idempotencia.

---

## 211. Provider resource budgets

Authentication podrá imponer:

```text
maximum provider calls per operation
deadline
```

---

## 212. No provider fan-out by default

No consultar 20 providers para cada login si puede resolverse mediante routing previo.

---

## 213. Compiled Provider Resolution

Durante bootstrap podrá compilarse:

```text
firewall + authenticator + claim type
        ↓
provider
```

---

## 214. Example compiled map

```text
web + password + email
    → users

admin + password + email
    → admins

enterprise + oidc + external_subject
    → federated

internal + service_token + client_id
    → services
```

---

## 215. Runtime dynamic inputs

Tenant/realm pueden añadir una segunda etapa de resolución usando profiles precompilados.

---

## 216. ProviderResolutionGraph

Podrá existir internamente:

```text
ProviderResolutionGraph
```

para aplicaciones complejas.

---

## 217. Graph safety

No deberá permitir ciclos como:

```text
provider A fallback B
provider B fallback A
```

---

## 218. Provider fallback graph validation

Deberá detectarse en bootstrap.

---

## 219. Provider registration example

Conceptualmente:

```php
Auth::identityProvider(
    'users',
    DatabaseIdentityProvider::class
);
```

---

## 220. Configuration example

```php
'providers' => [
    'users' => [
        'driver' => 'orm',
        'identity' => App\Auth\UserIdentity::class,
        'claim' => 'email',
    ],

    'employees' => [
        'driver' => 'ldap',
        'realm' => 'employees',
    ],

    'services' => [
        'driver' => 'database',
        'identity_type' => 'service',
    ],
],
```

---

## 221. Federated provider configuration

```php
'federation' => [
    'corporate' => [
        'protocol' => 'oidc',
        'issuer' => 'configured-trusted-issuer',
        'identity_provider' => 'users',
        'mapping' => 'federated_subject',
        'provision' => true,
    ],
],
```

---

## 222. No arbitrary issuer discovery

No permitir:

```php
'issuer' => $request->input('issuer');
```

para realizar discovery remoto sin allowlist.

---

## 223. FederationConnection

Podrá existir:

```text
FederationConnection
```

que encapsule:

```text
name
protocol
issuer/entity ID
client
provider
realm
tenant mapping
attribute mapping
provisioning policy
```

---

## 224. Connection Registry

```text
FederationConnectionRegistry
```

podrá almacenar conexiones confiables.

---

## 225. Tenant federation connections

Un tenant enterprise puede seleccionar de una lista administrada de conexiones.

La conexión no deberá construirse desde URL arbitraria del login request.

---

## 226. Domain ownership verification

Si un tenant configura un dominio para home realm discovery, el sistema empresarial podrá exigir prueba de propiedad fuera del login flow.

---

## 227. Federated mapping key

Preferible:

```text
connection_id + issuer + subject
```

o estructura equivalente segura.

---

## 228. Email changes

Si el IdP cambia email pero mantiene `sub`:

```text
same mapping
```

deberá conservarse.

---

## 229. Subject changes

Si cambia `sub`, no deberá asumirse misma identidad solo por email.

---

## 230. Identity merge

Fusionar dos identidades es una operación administrativa/de cuenta extremadamente sensible.

No forma parte del login automático.

---

## 231. Identity split

Igualmente, cambios de mappings deberán ser explícitos y auditables.

---

## 232. Local password + federation

Una Identity puede soportar ambos:

```text
local password
OIDC
passkey
```

La Identity no debe almacenar lógica específica de uno.

---

## 233. Federation-only Identity

También puede no tener password local.

---

## 234. Service identities without profile

Una ServiceIdentity puede contener únicamente:

```text
identifier
type
security state
```

sin email/nombre.

---

## 235. Identity serialization

La Identity runtime no deberá serializarse indiscriminadamente en Session.

Preferir:

```text
IdentityReference
```

más snapshot mínimo.

---

## 236. Persistent Identity snapshot

Si se utiliza:

```text
identity reference
display-safe fields
security version
```

con schema explícito.

---

## 237. IdentityResolver

Será el domain service principal que coordina providers.

Contrato conceptual:

```php
interface IdentityResolverInterface
{
    public function resolve(
        IdentityResolutionRequest $request
    ): IdentityResolutionResult;
}
```

---

## 238. IdentityResolver responsibilities

```text
normalize claim
resolve provider
perform lookup
validate uniqueness
apply tenant/realm binding
run identity state checks
produce IdentityReference
memoize per operation
```

---

## 239. IdentityResolver no verifica password

Esto seguirá siendo responsabilidad de Credential Verification.

---

## 240. Credential-derived identity resolution

Cuando un token produce un subject confiable:

```text
VerifiedCredential
      ↓
DerivedIdentityClaim
      ↓
IdentityResolver
```

---

## 241. Derived Claim marker

Podría existir:

```text
VerifiedIdentityClaim
```

para diferenciarla de una claim presentada directamente por el cliente.

---

## 242. VerifiedIdentityClaim

Contendrá:

```text
claim type
value
source verifier
issuer/provenance
```

---

## 243. Verified Claim still needs mapping

Que el claim sea auténtico no significa que el mapping local sea automático.

---

## 244. Provider-issued direct identity

En algunos casos el provider mismo representa la autoridad y la Identity puede ser externa.

VoltStack deberá permitir:

```text
FederatedIdentity
```

sin obligar siempre a crear un User local.

---

## 245. Local vs external Identity mode

Podrán existir:

```text
LOCAL
FEDERATED_LOCAL_MAPPING
FEDERATED_EXTERNAL
SERVICE_EXTERNAL
```

según aplicación.

---

## 246. External Identity Context

Si no hay User local, Authorization podrá trabajar con una Identity externa canónica siempre que exista un modelo estable.

---

## 247. External profile lifetime

No deberá asumirse que atributos externos permanecen válidos indefinidamente.

---

## 248. Provider trust profile

Cada provider/conexión podrá tener:

```text
IdentityProviderTrustProfile
```

para aportar metadata al Assurance System.

---

## 249. Trust profile examples

```text
local_database
corporate_managed
federated_standard
high_assurance_enterprise
machine_workload
```

---

## 250. Trust profile no autentica

Solo aporta contexto sobre la autoridad.

---

## 251. Assurance integration

Evidence podrá incluir:

```text
identity provider provenance
federation issuer
provider trust profile
```

para que `AuthenticationAssuranceCalculator` los evalúe.

---

## 252. Risk integration

Risk Engine podrá considerar:

```text
new federated provider
unexpected realm
JIT provisioned account
provider anomalies
```

---

## 253. Authorization integration

Authorization podrá consumir:

```text
Identity
IdentityType
Tenant
Authentication provenance
verified external attributes
```

pero Roles/Permissions no deberán cargarse dentro de Identity Core por necesidad de Auth.

---

## 254. Roles from external provider

Si se decide mapear grupos externos a Authorization roles, esto deberá pasar por un sistema explícito de mapping de Authorization.

No por el IdentityProvider automáticamente.

---

## 255. Identity ownership

El Auth subsystem deberá distinguir entre:

```text
who owns identity facts
```

y:

```text
who owns authorization facts
```

---

## 256. Testing IdentityProvider

Casos mínimos:

```text
claim resolved
not found
duplicate identities
disabled identity
tenant mismatch
provider unavailable
canonicalization
refresh
```

---

## 257. Federated mapper tests

```text
known issuer+subject
unknown mapping
JIT provisioning
duplicate mapping
issuer mismatch
subject collision
tenant mapping conflict
linking conflict
```

---

## 258. Multi-provider tests

```text
explicit provider
realm routing
domain routing
ambiguous provider
fallback not applicable
provider outage
no insecure downgrade
```

---

## 259. Security version tests

```text
same version
changed version
disabled identity
deleted identity
```

---

## 260. Concurrency tests

Especialmente:

```text
simultaneous federated provisioning
simultaneous account linking
mapping uniqueness
identity refresh
```

---

## 261. FrankenPHP tests

Una provider/resolver instance compartida no deberá conservar:

```text
last identity
last tenant
last claim
last provider result
```

entre requests.

---

## 262. Provider services

Deben ser preferentemente:

```text
stateless
immutable
request-independent
```

---

## 263. Provider request state

Vive en:

```text
IdentityResolutionRequest
OperationContext
request memoization
```

---

## 264. Scope-safe memoization

Puede almacenar temporalmente:

```text
IdentityReference → Identity
```

dentro de una request.

---

## 265. Negative request memoization

También puede recordar:

```text
claim X → NOT_FOUND
```

solo durante la request para evitar consultas duplicadas.

---

## 266. Logging safety tests

Deberá comprobarse que exceptions/logs no filtren:

```text
federation assertions
tokens
LDAP bind secrets
private identity attributes
```

---

## 267. Property-based testing

Útil para:

```text
IdentityReference equality
claim canonicalization
tenant scoping
provider resolution
mapping uniqueness
```

---

## 268. Domain invariants

### AUTH-ID-01

Una `IdentityClaim` no es una `Identity`.

#### AUTH-ID-02

Una Identity no implica Authentication activa.

#### AUTH-ID-03

Raw IDs no deben compararse cross-provider sin contexto.

#### AUTH-ID-04

La resolución de provider debe ser determinista.

#### AUTH-ID-05

Una credential fallida en un provider no activa fallback silencioso a otro.

#### AUTH-ID-06

Tenant-scoped identities deben incluir tenant binding.

#### AUTH-ID-07

`issuer + subject` constituye el identificador federado primario recomendado.

#### AUTH-ID-08

Email no deberá utilizarse como enlace federado automático inseguro.

#### AUTH-ID-09

El linking entre identidades debe ser explícito y auditable.

#### AUTH-ID-10

Un provider remoto en ERROR no equivale a Identity inexistente.

#### AUTH-ID-11

Una Identity disabled no puede producir autenticación válida.

#### AUTH-ID-12

La Identity no contiene Roles/Permissions obligatoriamente.

#### AUTH-ID-13

Service/Machine identities son ciudadanos de primera clase.

#### AUTH-ID-14

Una session debe reconstruir Identity bajo su provider/tenant correctos.

#### AUTH-ID-15

SecurityVersion puede invalidar Authentication persistida.

#### AUTH-ID-16

Federated claims deben provenir de assertions ya verificadas.

#### AUTH-ID-17

Federated mapping no debe confiar únicamente en atributos mutables como email.

#### AUTH-ID-18

Identity provider state no deberá filtrarse entre requests persistentes.

---

## 269. Anti-pattern — User-only core

Evitar:

```php
interface AuthContract
{
    public function user(): User;
}
```

como fundamento del sistema.

---

## 270. Anti-pattern — email equals identity

Evitar:

```text
email string
    =
canonical identity
```

---

## 271. Anti-pattern — try every provider

Evitar:

```php
foreach ($providers as $provider) {
    if ($identity = $provider->find($email)) {
        return $identity;
    }
}
```

sin realm/provider selection formal.

---

## 272. Anti-pattern — provider authenticates everything

Evitar:

```text
Provider
    loads identity
    verifies password
    creates session
    maps roles
```

---

## 273. Anti-pattern — external email auto-link

Evitar:

```text
OIDC email matches local email
    ↓
automatically link
```

sin una policy segura.

---

## 274. Anti-pattern — external groups become admin

Evitar:

```text
claim groups = ["admin"]
    ↓
local admin role
```

sin explicit Authorization mapping.

---

## 275. Anti-pattern — tenant omitted from reference

En tenant-scoped systems:

```text
user:10
```

sin tenant puede resultar ambiguo.

---

## 276. Anti-pattern — session stores full mutable model

Evitar depender de serializar la entidad ORM completa dentro de la sesión.

---

## 277. Anti-pattern — provider outage means user not found

Debe distinguirse:

```text
NOT_FOUND
```

de:

```text
UNAVAILABLE
```

---

## 278. Anti-pattern — dynamic remote issuer

No permitir que input del cliente determine directamente un IdP remoto arbitrario.

---

## 279. Anti-pattern — stale federated attributes

No considerar eternamente válidos atributos externos capturados años antes si son relevantes a seguridad.

---

## 280. Core components

```text
IdentityInterface
IdentityIdentifier
IdentityReference
IdentityType
IdentityClaim
IdentityClaimType
IdentityClaimNormalizer
IdentityProviderInterface
IdentityProviderRegistry
IdentityProviderResolver
IdentityResolver
IdentityResolutionRequest
IdentityResolutionResult
IdentityAuthenticationStateChecker
IdentityRefresher
IdentitySecurityVersion
```

---

## 281. Federation components

```text
FederatedSubject
FederationConnection
FederationConnectionRegistry
FederatedIdentityMapper
FederatedIdentityMapping
FederatedIdentityProvisioner
FederatedAttributeMapper
ExternalOrganizationMapper
HomeRealmDiscoveryService
```

---

## 282. Namespace sugerido

```text
VoltStack\Quantum\Auth\Identity
VoltStack\Quantum\Auth\Identity\Contracts
VoltStack\Quantum\Auth\Identity\Provider
VoltStack\Quantum\Auth\Identity\Resolution
VoltStack\Quantum\Auth\Identity\State
VoltStack\Quantum\Auth\Identity\Federation
VoltStack\Quantum\Auth\Identity\Refresh
```

---

## 283. Estructura sugerida

```text
src/Quantum/Auth/
└── Identity/
    ├── Contracts/
    │   ├── IdentityInterface.php
    │   ├── IdentityProviderInterface.php
    │   ├── IdentityResolverInterface.php
    │   └── IdentityRefresherInterface.php
    │
    ├── IdentityIdentifier.php
    ├── IdentityReference.php
    ├── IdentityType.php
    ├── IdentityClaim.php
    ├── IdentityClaimType.php
    ├── IdentityClaimNormalizer.php
    │
    ├── Provider/
    │   ├── IdentityProviderRegistry.php
    │   ├── IdentityProviderDescriptor.php
    │   ├── IdentityProviderResolver.php
    │   ├── DatabaseIdentityProvider.php
    │   ├── MemoryIdentityProvider.php
    │   └── ServiceIdentityProvider.php
    │
    ├── Resolution/
    │   ├── IdentityResolutionRequest.php
    │   ├── IdentityResolutionResult.php
    │   ├── IdentityResolutionStatus.php
    │   ├── IdentityResolver.php
    │   └── VerifiedIdentityClaim.php
    │
    ├── State/
    │   ├── IdentityAuthenticationState.php
    │   ├── IdentityAuthenticationStateChecker.php
    │   ├── IdentitySecurityVersion.php
    │   └── IdentityStatus.php
    │
    ├── Refresh/
    │   ├── IdentityRefresher.php
    │   ├── IdentityRefreshStrategy.php
    │   └── IdentityRefreshResult.php
    │
    └── Federation/
        ├── FederatedSubject.php
        ├── FederatedIdentityMapping.php
        ├── FederatedIdentityMapper.php
        ├── FederatedIdentityProvisioner.php
        ├── FederatedAttributeMapper.php
        ├── FederationConnection.php
        ├── FederationConnectionRegistry.php
        └── HomeRealmDiscoveryService.php
```

---

## 284. Flujo password local

```text
IdentityClaim(email)
        ↓
ClaimNormalizer
        ↓
ProviderResolver
        ↓
users provider
        ↓
IdentityResolver
        ↓
UserIdentity
        ↓
Identity State Check
        ↓
PasswordVerifier
```

---

## 285. Flujo tenant-aware

```text
TenantContext(acme)
       +
IdentityClaim(email)
        ↓
Tenant-aware Provider
        ↓
IdentityReference(
    tenant = acme
)
        ↓
Identity
```

---

## 286. Flujo token credential-first

```text
BearerTokenCredential
        ↓
TokenVerifier
        ↓
VerifiedIdentityClaim(
    issuer,
    subject
)
        ↓
IdentityProviderResolver
        ↓
Federated/Local Mapper
        ↓
Identity
```

---

## 287. Flujo OIDC existente

```text
OIDC ID Token
    verified
        ↓
issuer + subject
        ↓
FederatedSubject
        ↓
Mapping Lookup
        ↓
Local IdentityReference
        ↓
IdentityProvider
        ↓
Identity
```

---

## 288. Flujo OIDC JIT provisioning

```text
Verified FederatedSubject
        ↓
No mapping
        ↓
Provisioning Policy
        ↓
allowed
        ↓
FederatedAttributeMapper
        ↓
Create Identity
        ↓
Create Federated Mapping
        ↓
Identity
```

---

## 289. Flujo de linking

```text
Authenticated Local Identity
        ↓
Fresh Authentication
        ↓
External Federated Authentication
        ↓
Verified FederatedSubject
        ↓
Conflict Check
        ↓
Explicit Link
        ↓
Audit
```

---

## 290. Flujo service identity

```text
Service Credential
        ↓
Credential Verification
        ↓
Verified client/service claim
        ↓
ServiceIdentityProvider
        ↓
ServiceIdentity
```

---

## 291. Flujo session reconstruction

```text
AuthenticationSession
        ↓
IdentityReference
        ↓
SecurityVersion Check
        ↓
IdentityRefresher / Provider
        ↓
Current Identity
        ↓
AuthenticationContext reconstruction
```

---

## 292. Flujo invalidado por SecurityVersion

```text
Session:
    securityVersion = 4

Identity:
    securityVersion = 5

        ↓

STALE AUTHENTICATION
        ↓
session invalidation / reauthentication
```

---

## 293. Arquitectura global

```text
                        IDENTITY INPUT
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
          Direct Claim              Verified External
                                         Subject
                │                           │
                ▼                           ▼
       Claim Normalization          FederatedSubject
                │                           │
                ▼                           ▼
         Provider Resolver         Federation Mapper
                │                           │
                └─────────────┬─────────────┘
                              ▼
                      IdentityReference
                              │
                              ▼
                      IdentityProvider
                              │
                              ▼
                          Identity
                              │
                              ▼
                 Authentication State Check
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
               ACTIVE                  INVALID
                 │                         │
                 ▼                         ▼
      Credential/Evidence Pipeline       Reject
```

---

## 294. Criterios de aceptación

El subsistema será considerado correcto cuando:

1. `Identity` no dependa de `User`;
2. `IdentityInterface` sea mínimo;
3. soporte UUID/ULID/string IDs;
4. exista `IdentityReference`;
5. distinga raw ID de provider-aware identity;
6. soporte IdentityClaim;
7. soporte normalización tipada;
8. exista IdentityProvider;
9. exista ProviderRegistry;
10. exista ProviderResolver;
11. provider selection sea determinista;
12. no exista fallback inseguro entre providers;
13. soporte múltiples realms;
14. soporte tenant-scoped identities;
15. soporte global identities;
16. soporte human identities;
17. soporte service identities;
18. soporte machine/workload identities;
19. soporte credential-first identity resolution;
20. soporte federated identities;
21. use issuer + subject para federation;
22. soporte mapping local;
23. soporte JIT provisioning controlado;
24. soporte identity linking seguro;
25. detecte mapping conflicts;
26. soporte identity status;
27. soporte SecurityVersion;
28. soporte identity refresh;
29. permita cache segura;
30. permita provider resilience;
31. sea observable;
32. sea auditable;
33. sea seguro con FrankenPHP;
34. sea extensible mediante plugins;
35. permanezca separado de Authorization.

---

## 295. Regla arquitectónica final

VoltStack deberá preservar:

```text
CLAIM
    locates

PROVIDER
    resolves

IDENTITY
    represents

FEDERATED MAPPING
    binds external subject to identity

SECURITY STATE
    determines authentication eligibility

AUTHENTICATION
    proves current control of identity

AUTHORIZATION
    decides allowed actions
```

En consecuencia:

> **Una Identity describe quién es la entidad; no demuestra por sí sola que la entidad esté actualmente autenticada.**
> **Un IdentityProvider determina de dónde obtiene VoltStack esa identidad; no debe concentrar todas las responsabilidades de Authentication.**
> **Una identidad federada deberá estar vinculada mediante identificadores estables y verificables —principalmente issuer + subject— y no mediante atributos mutables como el email salvo políticas explícitas.**

Esta separación permitirá que VoltStack utilice identidades locales, empresariales, federadas, multi-tenant y machine-to-machine bajo un único modelo de Authentication.

---

## 296. Próximo documento

El siguiente documento recomendado será:

```text
10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md
```

Su responsabilidad será profundizar específicamente en:

```text
IdentityAuthenticationState
account enabled/disabled state
locked/suspended identities
pending identities
deleted identities
authentication eligibility
pre-authentication checks
post-authentication checks
security version
credential version
forced logout
security reset
compromise state
temporary access restrictions
tenant-level authentication restrictions
identity expiry
account recovery effects
state transitions
state persistence
distributed invalidation
session/token invalidation
race conditions
audit
observability
testing
```

Este documento establecerá la diferencia formal entre:

```text
Identity exists
```

y:

```text
Identity is currently eligible to authenticate
```

que será fundamental antes de desarrollar en profundidad passwords, sessions, MFA y mecanismos modernos de autenticación.
