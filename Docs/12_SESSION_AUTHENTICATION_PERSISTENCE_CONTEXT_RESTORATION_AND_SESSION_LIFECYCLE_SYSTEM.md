# VoltStack Authentication System

## 12 — Session Authentication, Persistence, Context Restoration and Session Lifecycle System

- **Archivo:** `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del subsistema de autenticación persistente mediante sesiones

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
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de convertir una Authentication válida en un estado persistente reutilizable entre múltiples ejecuciones.

Su función principal será transformar:

```text
AuthenticationDecision::AUTHENTICATED
        ↓
AuthenticationContext
        ↓
AuthenticationSession
        ↓
Session Store
        ↓
future request
        ↓
Session Authentication Recovery
        ↓
AuthenticationContext
```

de manera segura, revocable, aislada y compatible con:

```text
HTTP
SPA
FrankenPHP
distributed applications
multiple devices
multiple tenants
multiple firewalls
```

---

## 2. Principio fundamental

Una sesión no deberá interpretarse como:

```text
session exists
    =
user authenticated
```

La relación correcta será:

```text
Session Identifier
      ↓
Session State
      ↓
AuthenticationSession
      ↓
Integrity / Validity Checks
      ↓
Identity Security Validation
      ↓
Context Reconstruction
      ↓
AuthenticationContext
```

Por tanto:

> **La presencia de una cookie o identificador de sesión nunca será suficiente para establecer Authentication.**

---

## 3. Objetivos

El subsistema deberá soportar:

```text
session creation
session restoration
session ID rotation
session fixation protection
session revocation
logout
forced logout
idle timeout
absolute timeout
session renewal
multiple sessions
session limits
tenant binding
firewall binding
device metadata
security version validation
authentication provenance
assurance restoration
distributed stores
safe context serialization
SPA authentication
persistent-runtime isolation
```

---

## 4. No objetivos

Este sistema no deberá encargarse directamente de:

```text
password verification
MFA verification
Authorization
CSRF token generation
remember-me persistence
OAuth refresh tokens
API token validation
```

aunque deberá integrarse con esos subsistemas.

---

## 5. Arquitectura general

```text
AuthenticationContext
        │
        ▼
SessionAuthenticationStrategy
        │
        ▼
AuthenticationSessionFactory
        │
        ▼
AuthenticationSession
        │
        ▼
AuthenticationSessionRepository
        │
        ▼
Session Store
        │
        ▼
Session Identifier / Cookie
```

Request posterior:

```text
Session Identifier
        │
        ▼
SessionAuthenticator
        │
        ▼
AuthenticationSessionRepository
        │
        ▼
AuthenticationSession
        │
        ▼
SessionValidator
        │
        ├── expiration
        ├── revocation
        ├── firewall
        ├── tenant
        ├── security version
        └── optional device binding
        │
        ▼
AuthenticationContextRestorer
        │
        ▼
AuthenticationContext
```

---

## 6. Session vs AuthenticationSession

VoltStack deberá distinguir:

```text
Application Session
```

de:

```text
AuthenticationSession
```

La primera puede contener:

```text
flash messages
cart
UI state
CSRF state
application data
```

La segunda contiene exclusivamente información relacionada con Authentication.

---

## 7. Razón de esta separación

Una aplicación puede tener una sesión anónima antes del login.

Ejemplo:

```text
anonymous application session
        ↓
user authenticates
        ↓
same application session container
        +
new AuthenticationSession
```

La autenticación deberá añadir estado autenticado sin confundir ambos conceptos.

---

## 8. AuthenticationSession

Representará el estado persistido necesario para reconstruir una Authentication válida.

---

## 9. AuthenticationSession model

Conceptualmente:

```php
final readonly class AuthenticationSession
{
    public function __construct(
        public AuthenticationSessionId $id,
        public IdentityReference $identity,
        public string $firewall,
        public AuthenticationMethod $method,
        public AuthenticationAssurance $assurance,
        public SecurityVersion $securityVersion,
        public ?TenantIdentifier $tenant,
        public \DateTimeImmutable $authenticatedAt,
        public \DateTimeImmutable $createdAt,
        public \DateTimeImmutable $lastActivityAt,
        public ?\DateTimeImmutable $expiresAt,
        public AuthenticationSessionStatus $status,
        public AuthenticationSessionProvenance $provenance,
    ) {}
}
```

---

## 10. No raw credentials

Una `AuthenticationSession` nunca deberá almacenar:

```text
password
OTP
raw bearer token
API key secret
passkey private material
recovery code
```

---

## 11. Session identity binding

Toda AuthenticationSession deberá estar ligada a una:

```text
IdentityReference
```

y no solamente a:

```text
user_id
```

Esto permite soportar:

```text
multiple providers
multiple realms
service identities
tenant-scoped identities
```

---

## 12. AuthenticationSessionId

Deberá ser:

```text
unpredictable
high entropy
opaque
non-semantic
```

No deberá derivarse de:

```text
user ID
email
timestamp
incremental sequence
```

---

## 13. Session ID exposure

El cliente normalmente recibe únicamente:

```text
Session Identifier
```

mediante cookie.

La información completa de Authentication permanecerá en el servidor si se utiliza una sesión server-side.

---

## 14. Stateful Session Model

Modelo recomendado para aplicaciones web:

```text
Client
   │
   └── session id
          ↓
Session Store
          ↓
AuthenticationSession
```

Ventajas:

```text
central revocation
easy rotation
server-side invalidation
small cookies
security state control
```

---

## 15. Cookie-backed serialized session

VoltStack podría soportar sesiones client-side protegidas criptográficamente.

Pero esta estrategia deberá considerarse distinta.

Requiere:

```text
confidentiality if sensitive metadata
integrity
key rotation
expiration
size limits
revocation strategy
```

---

## 16. Recomendación

Para Authentication state principal de VoltStack V1:

```text
server-side session state
```

será la opción preferida.

---

## 17. AuthenticationSessionFactory

Solo deberá ejecutarse después de:

```text
AuthenticationDecision = AUTHENTICATED
```

y una `AuthenticationContext` válida.

---

## 18. Factory contract

Conceptualmente:

```php
interface AuthenticationSessionFactoryInterface
{
    public function create(
        AuthenticationContext $context,
        SessionCreationContext $creation
    ): AuthenticationSession;
}
```

---

## 19. SessionCreationContext

Podrá incluir:

```text
firewall
tenant
device reference
client metadata
session policy
request correlation
```

---

## 20. Session Authentication Strategy

Cada Firewall stateful podrá definir:

```text
SessionAuthenticationStrategy
```

responsable de decidir cómo se establece persistencia después del login.

---

## 21. Strategy responsibilities

Podrá decidir:

```text
regenerate application session ID
create AuthenticationSession
replace previous authentication state
retain selected application session data
invalidate old auth state
store provenance
```

---

## 22. Login session fixation

Una amenaza crítica será:

```text
attacker obtains/sets session id
        ↓
victim logs in
        ↓
same session id remains
        ↓
attacker reuses it
```

---

## 23. Session fixation protection

Después de Authentication exitosa:

```text
pre-auth session id
        ↓
regenerate
        ↓
new session id
```

deberá ser el comportamiento por defecto.

---

## 24. Session ID rotation point

La rotación deberá ocurrir como parte del Authentication Commit.

No antes de tener Authentication válida.

---

## 25. Migration of application session data

Al regenerar el ID se podrá conservar estado seguro como:

```text
cart
locale
flash data
intended destination
CSRF state depending on strategy
```

---

## 26. No migration of insecure auth state

No deberá migrarse:

```text
old authentication context
old identity binding
stale privilege metadata
```

sin validación explícita.

---

## 27. SessionMigrationPolicy

Podrá controlar:

```text
MIGRATE_SAFE_STATE
DROP_ALL
CUSTOM
```

---

## 28. AuthenticationSessionStatus

Estados posibles:

```text
ACTIVE
EXPIRED
REVOKED
TERMINATED
COMPROMISED
SUPERSEDED
```

---

## 29. ACTIVE

Puede restaurarse si todas las demás validaciones son correctas.

---

## 30. EXPIRED

Ya no puede establecer AuthenticationContext.

---

## 31. REVOKED

Fue invalidada explícitamente.

---

## 32. TERMINATED

Representa logout normal completado.

---

## 33. COMPROMISED

La sesión fue identificada como potencialmente robada/comprometida.

---

## 34. SUPERSEDED

Puede utilizarse cuando una política de rotación sustituye una sesión por otra.

---

## 35. SessionRepository

Contrato conceptual:

```php
interface AuthenticationSessionRepositoryInterface
{
    public function find(
        AuthenticationSessionId $id
    ): AuthenticationSessionLookupResult;

    public function save(
        AuthenticationSession $session
    ): void;

    public function revoke(
        AuthenticationSessionId $id
    ): void;
}
```

---

## 36. Repository independence

El Core no deberá depender de:

```text
Redis
SQL
filesystem
PHP native sessions
```

---

## 37. Session Store adapters

Podrán existir:

```text
RedisAuthenticationSessionStore
DatabaseAuthenticationSessionStore
FileAuthenticationSessionStore
InMemoryAuthenticationSessionStore
CustomDistributedSessionStore
```

---

## 38. Production recommendation

Para entornos distribuidos:

```text
Redis
database
distributed KV
```

serán opciones apropiadas.

---

## 39. File sessions

Podrán servir para:

```text
development
single-instance deployment
small applications
```

pero no son ideales para clusters sin filesystem compartido.

---

## 40. PHP native session integration

VoltStack podrá proporcionar adapter sobre:

```text
$_SESSION
```

pero el dominio no dependerá directamente de la variable superglobal.

---

## 41. FrankenPHP safety

El uso de PHP session globals deberá quedar encapsulado por request scope.

Nunca persistir referencias de sesión en servicios singleton.

---

## 42. SessionAuthenticator

Será el Authenticator encargado de detectar estado de sesión existente.

---

## 43. SessionAuthenticator responsibilities

```text
detect session identifier
create SessionReferenceCredential
delegate session lookup
trigger authentication restoration
```

No deberá considerar una cookie automáticamente válida.

---

## 44. Session recovery flow

```text
Request
   ↓
session cookie
   ↓
SessionAuthenticator
   ↓
SessionReferenceCredential
   ↓
AuthenticationSessionRepository
   ↓
AuthenticationSession
   ↓
Validation
   ↓
Context Restoration
```

---

## 45. AuthenticationSessionValidator

Componente central para determinar si una sesión sigue siendo aceptable.

---

## 46. Validator contract

Conceptualmente:

```php
interface AuthenticationSessionValidatorInterface
{
    public function validate(
        AuthenticationSession $session,
        SessionValidationContext $context
    ): AuthenticationSessionValidationResult;
}
```

---

## 47. Validation checks

Podrá incluir:

```text
status
idle timeout
absolute timeout
firewall binding
tenant binding
identity existence
Identity Security State
SecurityVersion
optional CredentialVersion
device policy
session policy version
revocation
```

---

## 48. Validation ordering

Los checks baratos deberán realizarse primero cuando sea razonable:

```text
session exists
status
expiration
firewall
tenant
```

antes de queries adicionales.

---

## 49. Idle Timeout

Una sesión podrá expirar si no tiene actividad durante:

```text
idle_timeout
```

---

## 50. Example

```text
lastActivityAt = 10:00
idle timeout = 30 minutes
request = 10:45
```

Resultado:

```text
EXPIRED
```

---

## 51. Absolute Timeout

Limita la vida máxima independientemente de actividad.

Ejemplo:

```text
createdAt = Monday 08:00
absolute timeout = 12 hours
```

A las 20:00:

```text
session expires
```

aunque haya actividad constante.

---

## 52. Idle vs Absolute

```text
Idle timeout
    protects abandoned sessions

Absolute timeout
    limits maximum continuous trust
```

---

## 53. Session policy

Podrá definir:

```text
idle_timeout
absolute_timeout
renewal_window
freshness_window
```

por Firewall.

---

## 54. Admin sessions

Ejemplo:

```text
short idle timeout
short absolute timeout
higher assurance
more frequent reauthentication
```

---

## 55. Standard web sessions

Podrán tener ventanas más amplias.

---

## 56. Service sessions

Normalmente los service-to-service mechanisms no deberían utilizar browser-style sessions, aunque el sistema no deberá impedir adapters especializados.

---

## 57. Sliding sessions

Una sesión sliding extiende su validez con actividad.

Deberá limitarse mediante:

```text
absolute timeout
```

para impedir duración infinita.

---

## 58. Session renewal

En lugar de extender indefinidamente el mismo registro:

```text
session nearing renewal threshold
        ↓
new AuthenticationSession
        ↓
old session superseded
```

puede proporcionar mejores propiedades.

---

## 59. RenewalPolicy

Podrá definir:

```text
NONE
UPDATE_EXPIRY
ROTATE_SESSION_ID
CREATE_NEW_SESSION_RECORD
REAUTHENTICATE
```

---

## 60. Session ID rotation during lifetime

Además de login, podrá rotarse:

```text
periodically
after step-up
after privilege-sensitive changes
after recovery
after impersonation start/end
```

---

## 61. Rotation should preserve authentication lineage

Nueva sesión podrá registrar:

```text
parent_session_id
rotation reason
```

si resulta útil.

---

## 62. SessionFamily

Podrá introducirse:

```text
AuthenticationSessionFamily
```

para agrupar rotaciones de una misma autenticación continua.

---

## 63. SessionFamily use cases

```text
revoke entire chain
detect replay of previous id
session history
refresh rotation
```

---

## 64. Session fixation vs Session rotation

No son exactamente lo mismo.

```text
Fixation protection
    mandatory rotation at authentication boundary

Rotation
    optional periodic/security lifecycle mechanism
```

---

## 65. Session Cookie

En HTTP, el Session ID normalmente se transportará mediante cookie.

---

## 66. Cookie security defaults

Authentication cookies deberán favorecer:

```text
HttpOnly
Secure
appropriate SameSite
minimal Domain scope
minimal Path scope
```

---

## 67. HttpOnly

Reduce exposición a JavaScript del navegador.

No elimina por sí mismo XSS risk.

---

## 68. Secure

En producción, Authentication cookie deberá enviarse únicamente sobre HTTPS.

---

## 69. SameSite

La configuración dependerá del modelo de aplicación:

```text
Lax
Strict
None
```

pero `None` deberá requerir `Secure` y una necesidad real.

---

## 70. Cookie Domain

Evitar:

```text
.example.com
```

si no es necesario compartir Authentication entre todos los subdominios.

---

## 71. Host-only cookies

Deberán preferirse cuando el sistema no requiere cross-subdomain auth.

---

## 72. Cookie Path

Puede ayudar a limitar exposición a determinadas rutas/aplicaciones, aunque no sustituye controles de seguridad reales.

---

## 73. Cookie name

Podrá ser específico por:

```text
application
firewall
security realm
```

para evitar colisiones.

---

## 74. `__Host-` cookie prefix

Cuando sea compatible con la arquitectura HTTP, VoltStack podrá favorecer cookies con propiedades equivalentes a:

```text
__Host-
```

para endurecer scope.

---

## 75. Cookie value

Deberá contener únicamente un identificador opaco o representación protegida.

No:

```text
user_id=5
role=admin
email=...
```

en plaintext como fundamento de Authentication.

---

## 76. Session ID entropy

La generación utilizará CSPRNG.

Nunca:

```text
md5(time())
uniqid()
incremental IDs
random strings from non-secure PRNG
```

---

## 77. SessionIdGenerator

Contrato:

```php
interface AuthenticationSessionIdGeneratorInterface
{
    public function generate(): AuthenticationSessionId;
}
```

---

## 78. Session ID hashing at rest

Podrá almacenarse un digest del ID en el store en diseños donde resulte útil.

Esto reduce impacto de ciertos leaks del store.

---

## 79. Opaque lookup

Una posible estrategia:

```text
client session token
        ↓
cryptographic digest
        ↓
store lookup
```

---

## 80. Digest requirements

El ID tiene alta entropía, por lo que un hash rápido criptográfico puede ser adecuado para lookup.

No tratarlo como password humano.

---

## 81. Session ID in logs

Nunca deberá registrarse completo.

Puede utilizarse:

```text
session fingerprint
```

de forma segura.

---

## 82. Session ID in URLs

Nunca deberá ser el mecanismo estándar.

Evitar:

```text
?session_id=...
```

por riesgo de filtración.

---

## 83. AuthenticationContext persistence

VoltStack no deberá serializar indiscriminadamente el objeto runtime completo.

---

## 84. Incorrecto

```php
serialize($authenticationContext);
```

si contiene:

```text
service references
runtime metadata
large identity model
unsafe attributes
```

---

## 85. AuthenticationContextSnapshot

Se utilizará una representación segura.

---

## 86. Snapshot model

Podrá contener:

```text
identity reference
authentication method
verified factor summary
assurance
authenticatedAt
security version
tenant
device reference
provenance
authentication chain reference
```

---

## 87. ContextSnapshotSerializer

Deberá utilizar:

```text
allowlisted schema
versioning
typed serialization
```

---

## 88. Context snapshot version

Podrá existir:

```text
AuthenticationContextSnapshotVersion
```

para evolucionar el formato.

---

## 89. No raw evidence persistence

No toda `AuthenticationEvidence` deberá serializarse.

Preferir un resumen seguro suficiente para restauración.

---

## 90. Restored context provenance

La restauración deberá poder distinguir:

```text
fresh authentication
```

de:

```text
restored from session
```

---

## 91. AuthenticationContextRestorer

Contrato conceptual:

```php
interface AuthenticationContextRestorerInterface
{
    public function restore(
        AuthenticationSession $session,
        SessionRestorationContext $context
    ): AuthenticationContextRestorationResult;
}
```

---

## 92. Restoration flow

```text
AuthenticationSession
        ↓
validate identity reference
        ↓
resolve/refresh Identity
        ↓
load security state
        ↓
validate SecurityVersion
        ↓
evaluate eligibility
        ↓
rebuild authentication metadata
        ↓
AuthenticationContext
```

---

## 93. Restored Context is new runtime object

No deberá reutilizarse una referencia de una request anterior.

Cada request obtiene un Context runtime propio.

---

## 94. Identity refresh during restoration

Según strategy:

```text
ALWAYS
VERSION_BASED
INTERVAL_BASED
SNAPSHOT
```

---

## 95. SessionIdentityRefreshPolicy

Será configurable por Firewall/provider.

---

## 96. SecurityVersion validation

La sesión guarda:

```text
securityVersion = N
```

El Identity state actual proporciona:

```text
securityVersion = M
```

Solo continúa si la policy acepta esa relación.

---

## 97. Normal rule

```text
N == M
```

deberá ser el requisito común.

---

## 98. Version mismatch

Produce:

```text
SESSION_STALE
```

y normalmente:

```text
revoke/terminate session
```

---

## 99. CredentialVersion interaction

No siempre será necesario verificar una CredentialVersion en cada sesión.

Podrá utilizarse para invalidación selectiva.

---

## 100. Example

Session autenticada mediante password credential v5.

Password cambia a v6.

La policy puede:

```text
revoke all sessions
```

mediante SecurityVersion, o:

```text
invalidate only sessions derived from password v5
```

en un modelo avanzado.

---

## 101. Authentication provenance

La sesión deberá conservar suficiente provenance para saber cómo fue autenticada.

Ejemplos:

```text
password
password + TOTP
passkey
OIDC
remember-me
```

---

## 102. Provenance use cases

```text
assurance restoration
step-up decisions
selective invalidation
security analytics
session management UI
```

---

## 103. Assurance restoration

Una sesión puede conservar el assurance alcanzado.

Ejemplo:

```text
AAL2
```

Pero deberá considerarse:

```text
freshness decay
method policy
security changes
```

---

## 104. Assurance expiration

Una sesión puede seguir autenticada pero dejar de cumplir cierta exigencia.

Ejemplo:

```text
session valid
AAL2 obtained 8 hours ago
wire transfer requires fresh AAL2 within 5 minutes
```

Resultado:

```text
STEP_UP_REQUIRED
```

---

## 105. Authentication freshness

La sesión deberá conservar:

```text
authenticatedAt
lastPrimaryAuthenticationAt
lastStepUpAt
```

cuando sea necesario.

---

## 106. Session activity

`lastActivityAt` no equivale a:

```text
lastAuthenticationAt
```

Una sesión utilizada continuamente puede seguir necesitando reauthentication.

---

## 107. Session metadata

Podrá incluir:

```text
createdAt
lastActivityAt
lastAuthenticationAt
IP metadata
device reference
user agent classification
origin metadata
```

pero deberá evitar almacenar más PII de la necesaria.

---

## 108. IP binding

No deberá utilizarse binding rígido a IP por defecto.

IPs cambian frecuentemente por:

```text
mobile networks
NAT
VPN
IPv6 privacy addresses
```

---

## 109. IP as risk signal

Mejor:

```text
IP/network changes
    → risk signal
```

que:

```text
IP changed
    → immediate logout
```

salvo aplicaciones especiales.

---

## 110. User-Agent binding

Igualmente, puede ser una señal débil, no una raíz de confianza.

---

## 111. Device binding

VoltStack podrá soportar:

```text
DeviceReference
```

si existe Device System confiable.

---

## 112. Device binding levels

Podrán ser:

```text
NONE
OBSERVATIONAL
SOFT
STRICT
CRYPTOGRAPHIC
```

---

## 113. OBSERVATIONAL

Solo telemetría/risk.

---

## 114. SOFT

Cambio de device metadata puede exigir challenge.

---

## 115. STRICT

Session no restaura fuera del device esperado.

---

## 116. CRYPTOGRAPHIC

La sesión se liga a una prueba criptográfica adicional cuando la tecnología lo permita.

---

## 117. Session theft

El sistema deberá asumir que un session identifier puede ser robado.

Protecciones incluyen:

```text
TLS
Secure cookie
HttpOnly
SameSite
CSRF protection
rotation
short lifetimes
device/risk signals
revocation
security monitoring
```

---

## 118. Session theft detection

Podrán existir señales:

```text
impossible geography
abrupt device change
parallel distant usage
replay of superseded session ID
```

que alimenten Risk Engine.

---

## 119. No infallible theft detection

Estas señales no deberán venderse como prueba definitiva.

---

## 120. Session replay

Si una sesión rotada utiliza family/rotation tracking, reutilizar un ID antiguo podrá indicar:

```text
possible theft/replay
```

---

## 121. Rotated Session Replay Policy

Podrá:

```text
revoke session family
mark compromised
require reauthentication
notify security subsystem
```

---

## 122. Concurrent requests during rotation

Debe evitarse invalidar de forma accidental requests legítimas que comenzaron antes de la rotación.

---

## 123. Rotation grace window

Podrá existir una ventana muy corta/controlada para requests concurrentes.

---

## 124. Grace window risks

No deberá convertirse en:

```text
old session IDs valid for minutes
```

sin necesidad.

---

## 125. Session concurrent update

`lastActivityAt` y metadata no deberán provocar race conditions graves.

Podrán usar:

```text
atomic touch
optimistic updates
coalesced writes
```

---

## 126. Touch optimization

No actualizar DB/Redis en cada request si genera carga excesiva.

Podrá utilizarse:

```text
activity write interval
```

Ejemplo conceptual:

```text
touch at most once per N seconds
```

---

## 127. Idle timeout correctness

Aunque se reduzcan writes, el sistema deberá mantener semántica conservadora.

---

## 128. Session write amplification

En aplicaciones de alto tráfico, escribir cada request puede ser costoso.

El subsistema deberá permitir:

```text
lazy touch
batched touch
TTL-based store
```

---

## 129. Redis TTL

Un store Redis puede representar idle expiration con TTL.

Pero aún puede necesitar metadata para absolute timeout.

---

## 130. Database sessions

Podrán utilizar:

```text
expires_at
last_activity_at
```

con índices adecuados.

---

## 131. Session cleanup

Sesiones expiradas deberán eliminarse mediante:

```text
TTL
garbage collection
scheduled cleanup
database pruning
```

---

## 132. Authentication Session Pruner

Podrá existir:

```text
AuthenticationSessionPruner
```

para stores sin TTL automático.

---

## 133. Pruning is not security validation

Aunque un registro expirado siga físicamente presente, el Validator debe rechazarlo.

---

## 134. Multiple Sessions

VoltStack deberá soportar varias sesiones activas por Identity.

Ejemplo:

```text
Laptop
Phone
Tablet
```

---

## 135. Session inventory

Podrá existir:

```text
AuthenticationSessionRegistry
```

para listar sesiones de una Identity.

---

## 136. Session Registry use cases

```text
logout other devices
account security page
session limit enforcement
incident response
device management
```

---

## 137. Session listing privacy

La UI podrá mostrar:

```text
device label
approximate location
created time
last activity
current session marker
```

sin exponer session IDs.

---

## 138. Concurrent Session Policy

Podrán existir:

```text
UNLIMITED
MAX_N
SINGLE_SESSION
PER_DEVICE
PER_FIREWALL
PER_TENANT
```

---

## 139. Single session

Una nueva Authentication podría revocar la anterior.

---

## 140. MAX_N

Si se supera:

```text
revoke oldest
reject new session
ask user
```

según policy.

---

## 141. Default

VoltStack no deberá imponer `SINGLE_SESSION` por defecto.

---

## 142. Session limit atomicity

Dos logins concurrentes deberán respetar el límite de forma transaccional o aproximadamente consistente según store.

---

## 143. Firewall isolation

Session de:

```text
web
```

no deberá autenticar automáticamente:

```text
admin
```

salvo configuración de shared realm/session.

---

## 144. Session firewall binding

`AuthenticationSession` deberá registrar:

```text
firewall
```

o un realm/session namespace equivalente.

---

## 145. Cross-firewall restoration

Por defecto:

```text
session.firewall != request.firewall
    ↓
reject
```

---

## 146. Shared authentication realms

VoltStack podrá permitir que:

```text
web
admin
```

compartan Identity state si explícitamente configurado.

---

## 147. Shared realm security

Compartir sesión no debe eliminar:

```text
higher assurance requirement
fresh authentication
step-up requirement
```

del Firewall más estricto.

---

## 148. Example

Web session:

```text
AAL1
```

Admin Firewall:

```text
minimum AAL2
```

Puede reutilizar Identity base pero deberá producir:

```text
STEP_UP_REQUIRED
```

---

## 149. Tenant binding

Una sesión tenant-bound deberá almacenar:

```text
TenantIdentifier
```

---

## 150. Cross-tenant request

Si:

```text
session tenant = A
request tenant = B
```

por defecto:

```text
reject session authentication
```

---

## 151. Global Identity with tenant switch

Puede requerir un flow explícito:

```text
switch tenant
        ↓
validate membership / auth conditions
        ↓
create new tenant-bound context/session binding
```

---

## 152. Session tenant mutation prohibited

No simplemente:

```php
$session->tenant = $newTenant;
```

sin validación.

---

## 153. Session scopes

Una Identity global puede tener múltiples sesiones:

```text
Tenant A session
Tenant B session
```

si el modelo lo requiere.

---

## 154. Logout

Logout deberá ser una operación explícita del Authentication System.

---

## 155. Logout flow

```text
Current AuthenticationContext
        ↓
Current AuthenticationSession
        ↓
revoke/terminate
        ↓
clear AuthenticationContextStorage
        ↓
rotate/invalidate application session state
        ↓
expire cookie
        ↓
AuthenticationLoggedOut event
```

---

## 156. Logout current session

Solo invalida la sesión actual.

---

## 157. Logout all sessions

Deberá utilizar:

```text
ForcedLogoutService
```

o Session Registry + SecurityVersion.

---

## 158. Logout other sessions

Permitirá:

```text
keep current
revoke all others
```

normalmente después de fresh authentication.

---

## 159. Logout requires no Authorization to self

El usuario autenticado normalmente puede cerrar su propia sesión.

Administrar sesiones ajenas sí requiere Authorization.

---

## 160. Logout idempotency

Llamar logout sobre una sesión ya revocada deberá producir un resultado seguro/idempotente.

---

## 161. Cookie invalidation

El servidor deberá instruir al cliente para expirar la cookie.

Pero:

> **Eliminar la cookie no sustituye la revocación server-side cuando la sesión es revocable.**

---

## 162. Session revocation

Podrá ocurrir por:

```text
logout
forced logout
password reset
Identity disabled
security compromise
session limit
manual user action
device removal
admin action
rotation replay
```

---

## 163. SessionRevocationReason

Códigos posibles:

```text
LOGOUT
FORCED_LOGOUT
PASSWORD_RESET
SECURITY_VERSION_CHANGED
IDENTITY_DISABLED
COMPROMISED
SESSION_LIMIT
DEVICE_REVOKED
ADMIN_REVOKED
SUPERSEDED
```

---

## 164. Revocation audit

Debe registrar:

```text
session reference/fingerprint
identity reference
reason
actor if applicable
timestamp
```

---

## 165. Session compromise response

Puede producir:

```text
revoke session
revoke family
mark device suspicious
raise Risk signal
force step-up
increment SecurityVersion
```

según gravedad.

---

## 166. Revocation Store

En stores server-side, el propio status puede representar revocación.

En cookies self-contained se requerirá otra estrategia.

---

## 167. Self-contained session revocation

Opciones:

```text
short lifetime
revocation list
security version
session version
reference token
```

---

## 168. Why server-side sessions simplify revocation

Permiten:

```text
lookup
status check
delete/revoke
```

por request.

---

## 169. Session state consistency

Si el store no está disponible:

```text
session cannot be validated
```

Resultado:

```text
ERROR / unauthenticated fail-closed
```

según integración.

---

## 170. Store outage

No deberá tratarse como:

```text
session valid
```

solo porque la cookie exista.

---

## 171. Availability trade-off

Una aplicación podrá diseñar caches o replicated session stores.

Pero cualquier degraded mode deberá ser explícito.

---

## 172. Session cache

Un local request cache es seguro.

Cross-request cache deberá respetar:

```text
revocation
expiry
security version
```

---

## 173. Local in-process cache risk

Con FrankenPHP, cachear AuthenticationSession globalmente puede retrasar revocation.

No recomendado salvo invalidación rigurosa.

---

## 174. Recommended hot path

```text
Session ID
   ↓
distributed/session store lookup
   ↓
minimal validation
   ↓
request memoization
```

---

## 175. Request memoization

Una vez restaurado el Context:

```text
Auth::check()
Auth::identity()
Auth::context()
```

no deberán repetir store lookup.

---

## 176. Negative memoization

Una session ausente/inválida podrá memoizarse durante la request.

---

## 177. Session locking

PHP-style full session locking puede serializar requests concurrentes del mismo usuario.

VoltStack deberá evitar depender obligatoriamente de ese modelo.

---

## 178. Locking strategies

Podrán existir:

```text
NONE
OPTIMISTIC
SHORT_CRITICAL_SECTION
STORE_NATIVE
```

---

## 179. Long session locks undesirable

En SPA:

```text
multiple parallel requests
```

no deberían bloquearse innecesariamente por toda la duración del request.

---

## 180. Authentication Session mutations

Operaciones críticas como:

```text
rotation
revocation
session limit enforcement
```

sí pueden requerir atomicidad específica.

---

## 181. CAS/versioning

`AuthenticationSession` podrá tener:

```text
version
```

para optimistic concurrency.

---

## 182. SessionVersion

Diferente de:

```text
Identity SecurityVersion
CredentialVersion
```

Sirve para concurrencia/mutación del registro de sesión.

---

## 183. CSRF interaction

Session-based browser Authentication implica credenciales ambient authority mediante cookies.

Por ello las operaciones state-changing deberán integrarse con CSRF protection.

---

## 184. Authentication vs CSRF

```text
Authentication
    who is sending the request?

CSRF Protection
    did the authenticated browser intentionally originate this state-changing request?
```

Son sistemas diferentes.

---

## 185. Login CSRF

Incluso `/login` puede necesitar protección contra login CSRF/session swapping según el flow.

---

## 186. SPA session authentication

VoltStack SPA podrá utilizar:

```text
HttpOnly session cookie
+
CSRF mechanism
+
structured Authentication responses
```

---

## 187. SPA should not need to read session cookie

La cookie deberá permanecer:

```text
HttpOnly
```

si se usa como credential.

---

## 188. SPA Auth state

El frontend puede recibir:

```text
authenticated: true
identity summary
assurance status
```

pero no el session secret.

---

## 189. Session expiry in SPA

Backend podrá responder:

```text
session_expired
```

para que el frontend redirija o muestre login.

---

## 190. Step-up in SPA

Una sesión base puede seguir activa mientras una operación devuelve:

```text
step_up_required
```

---

## 191. Reauthentication

Una sesión válida no evita que determinadas operaciones pidan reauthentication.

---

## 192. Reauthentication effect

Después de éxito podrá:

```text
update lastAuthenticationAt
raise assurance
rotate session ID
replace AuthenticationContext
```

---

## 193. Step-up effect

Podrá actualizar:

```text
assurance
verified factor summary
lastStepUpAt
AuthenticationChain
```

---

## 194. Session upgrade

Después de step-up deberá crearse un nuevo snapshot de Authentication state.

---

## 195. Session downgrade

Assurance puede perder freshness, pero el sistema no deberá necesariamente modificar el registro en cada instante.

El Validator/Policy puede calcular effective assurance.

---

## 196. AuthenticationSessionPolicy

Podrá incluir:

```text
idle timeout
absolute timeout
renewal
rotation
device rules
tenant binding
concurrent session limits
identity refresh
security version validation
```

---

## 197. SessionPolicyResolver

Podrá combinar:

```text
framework defaults
firewall policy
tenant profile
identity type profile
```

---

## 198. Security floor

Tenant/application configuration no deberá poder desactivar garantías críticas como:

```text
session ID unpredictability
fixation protection
secure production cookie policy
```

sin override explícito de bajo nivel.

---

## 199. Session lifetime defaults

No deberán codificarse como verdad universal en este documento.

Deben ser configurables y depender del riesgo de la aplicación.

---

## 200. Remember-Me boundary

`RememberMe` no será una AuthenticationSession normal.

Representará una credential persistente capaz de crear una nueva sesión.

---

## 201. Flow

```text
No active AuthenticationSession
        +
RememberMeCredential
        ↓
RememberMeAuthenticator
        ↓
verification
        ↓
reduced/freshness-aware Authentication
        ↓
new AuthenticationSession
```

---

## 202. Why separate remember-me

Permite:

```text
different expiration
different revocation
different assurance
rotation
credential theft detection
```

---

## 203. Remember-me must not resurrect revoked session state

Si SecurityVersion cambió:

```text
remember-me credential
```

deberá validarse contra la nueva seguridad según policy.

---

## 204. Session creation from remember-me

Puede producir una sesión normal, pero con provenance:

```text
source = remember_me
```

y posiblemente menor assurance/freshness.

---

## 205. Session Persistence Coordinator

Podrá existir:

```text
AuthenticationSessionPersistenceCoordinator
```

para coordinar:

```text
ID regeneration
AuthenticationSession creation
repository persistence
cookie issuance
Context activation
transaction completion
```

---

## 206. Commit ordering

Recomendación:

```text
AuthenticationDecision AUTHENTICATED
        ↓
build AuthenticationContext
        ↓
commit-time security validation
        ↓
generate/rotate session ID
        ↓
persist AuthenticationSession
        ↓
activate AuthenticationContext
        ↓
prepare cookie output
        ↓
mark authentication committed
```

---

## 207. Cookie delivery failure

Si la response no puede entregar la cookie:

```text
server-side session may exist unused
```

El sistema deberá tolerarlo y dejar que expire/prune, o compensar cuando sea posible.

---

## 208. Persistence failure

Si guardar AuthenticationSession falla:

```text
do not activate authenticated state
```

para un Firewall que requiere sesión persistente.

---

## 209. Fail before commit

Resultado:

```text
ERROR
no active authenticated session
```

---

## 210. Context activation before store forbidden

Evitar:

```text
Auth::context = authenticated
    ↓
session save fails
```

en flows stateful obligatorios.

---

## 211. Session Store Interface

Podrá exponer operaciones especializadas:

```text
create
get
touch
rotate
revoke
revokeByIdentity
listByIdentity
prune
```

---

## 212. Atomic operations

Stores deberán poder soportar cuando sea necesario:

```text
create-if-absent
compare-and-swap
atomic revoke
atomic rotation
```

---

## 213. Redis adapter

Puede aprovechar:

```text
TTL
atomic commands
transactions/scripts when appropriate
```

---

## 214. Database adapter

Podrá usar:

```text
transactions
row locking
optimistic version
indexes
```

---

## 215. Distributed consistency

No todas las operaciones necesitan linearizability global.

Pero operaciones de seguridad críticas deberán documentar sus garantías.

---

## 216. Revocation consistency profile

Podrá existir:

```text
STRONG
BOUNDED_STALENESS
EVENTUAL
```

según store/arquitectura.

---

## 217. Admin/firewall profile

Podrá exigir:

```text
STRONG
```

para revocación.

---

## 218. Session store sharding

En aplicaciones muy grandes, sesiones podrán particionarse por:

```text
session id hash
tenant
region
```

sin alterar el dominio.

---

## 219. Session affinity

VoltStack no deberá requerir sticky sessions si se utiliza store compartido.

---

## 220. Multi-region sessions

Deberán considerar:

```text
replication latency
revocation propagation
clock consistency
data residency
```

---

## 221. Region binding

Una policy de alta seguridad podría asociar session home region.

---

## 222. Clock consistency

Idle/absolute timeout deben utilizar un clock coherente.

Evitar depender de timestamps proporcionados por cliente.

---

## 223. Client time untrusted

Nunca:

```text
client says last activity = now
```

---

## 224. Session activity update

Debe derivarse del servidor.

---

## 225. Session renewal and absolute time

Una rotación no deberá reiniciar arbitrariamente el absolute lifetime si la policy busca limitar una autenticación continua.

---

## 226. Authentication lineage start

Podrá conservarse:

```text
initialAuthenticatedAt
```

independiente de:

```text
currentSessionCreatedAt
```

---

## 227. Example

```text
initial auth: 08:00
session rotated: 10:00
absolute max: 12 hours
```

Debe expirar a:

```text
20:00
```

no a 22:00.

---

## 228. New fresh reauthentication

Una reauthentication real puede iniciar un nuevo lineage si la policy así lo define.

---

## 229. Session impersonation boundary

Impersonation no deberá sobrescribir silenciosamente la identidad original.

Una futura capa de delegation/impersonation deberá preservar:

```text
actor
subject
base session
```

---

## 230. Session after impersonation

Al entrar/salir de impersonation se recomienda rotación de Session ID.

---

## 231. Privilege transition

Cambios sensibles de Authentication state deberían considerar session rotation para reducir fixation/reuse issues.

---

## 232. Session serializer security

Deberá rechazar campos desconocidos cuando sea apropiado.

---

## 233. Deserialization safety

Nunca utilizar deserialización insegura de objetos arbitrarios provenientes del cliente.

---

## 234. Safe formats

Preferir:

```text
structured arrays
JSON
typed binary formats
```

sobre:

```text
untrusted PHP object serialization
```

---

## 235. Server-side records can still require schema validation

Storage corruption o version mismatch deben fallar cerrados.

---

## 236. Session record signature

En un store server-side confiable puede no ser estrictamente necesario firmar cada record.

En stores menos confiables o client-side sí puede requerirse integridad criptográfica.

---

## 237. Encryption of session store

Puede ser requerida por:

```text
compliance
sensitive metadata
untrusted infrastructure operators
```

pero no sustituye access control y session ID security.

---

## 238. Sensitive session metadata

Minimizar:

```text
IP history
device details
federated claims
authentication attributes
```

al mínimo necesario.

---

## 239. Session data minimization

AuthenticationSession debe ser una referencia compacta, no un dump de toda la Identity.

---

## 240. Identity changes

Cambios de:

```text
display name
email
avatar
```

no deberían requerir reescribir todas las sesiones si se resuelven dinámicamente.

---

## 241. Current identity data

`Auth::identity()` puede resolver/refrescar la Identity actual independientemente del snapshot de sesión.

---

## 242. Session status updates

Logout/revocation deberán ser idempotentes y concurrent-safe.

---

## 243. Session event model

Eventos posibles:

```text
AuthenticationSessionCreated
AuthenticationSessionRestored
AuthenticationSessionRotated
AuthenticationSessionRenewed
AuthenticationSessionExpired
AuthenticationSessionRevoked
AuthenticationSessionTerminated
AuthenticationSessionCompromised
AuthenticationSessionRestorationRejected
```

---

## 244. Session audit

Eventos sensibles:

```text
session created
session revoked
logout all
device session revoked
session compromise
admin revocation
cross-tenant rejection
security version mismatch
```

---

## 245. High-volume restoration event

No necesariamente deberá persistirse un Audit Record en cada request restaurada.

Eso produciría volumen enorme.

Mejor:

```text
metrics/tracing
```

y auditoría solo para eventos relevantes.

---

## 246. Observability spans

```text
auth.session.create
auth.session.restore
auth.session.validate
auth.session.rotate
auth.session.revoke
```

---

## 247. Metrics

Ejemplos:

```text
auth_session_created_total
auth_session_restored_total
auth_session_expired_total
auth_session_revoked_total
auth_session_security_version_mismatch_total
auth_session_rotation_total
auth_session_restoration_latency
auth_session_store_latency
```

---

## 248. Safe labels

```text
firewall
store
result
revocation_reason
state_mode
```

Evitar:

```text
session id
identity id
email
```

como labels de alta cardinalidad.

---

## 249. Security telemetry

Podrá detectar:

```text
many invalid session IDs
replay of superseded IDs
cross-tenant use
repeated revoked session use
unusual concurrent locations
```

---

## 250. Session enumeration resistance

No devolver diferencias externas innecesarias entre:

```text
unknown session
expired session
revoked session
```

a un atacante.

---

## 251. Internal classification

Sí deberá conservarse para métricas/auditoría.

---

## 252. AuthenticationSessionValidationResult

Estados posibles:

```text
VALID
EXPIRED_IDLE
EXPIRED_ABSOLUTE
REVOKED
TERMINATED
STALE_SECURITY_VERSION
TENANT_MISMATCH
FIREWALL_MISMATCH
IDENTITY_INELIGIBLE
DEVICE_MISMATCH
SUPERSEDED
NOT_FOUND
ERROR
```

---

## 253. VALID only after complete validation

No basta con encontrar el registro.

---

## 254. Session restoration result

Podrá ser:

```text
RESTORED
UNAUTHENTICATED
STEP_UP_REQUIRED
REAUTHENTICATION_REQUIRED
REJECTED
ERROR
```

---

## 255. Expired optional session

En una ruta pública web:

```text
expired session
```

puede resultar:

```text
guest
```

después de limpiar la cookie.

---

## 256. Expired session on protected route

Resultará:

```text
UNAUTHENTICATED
```

y la integration layer solicitará login.

---

## 257. SecurityVersion mismatch

Puede producir:

```text
REAUTHENTICATION_REQUIRED
```

o:

```text
RECOVERY_REQUIRED
```

dependiendo de la causa almacenada/policy.

---

## 258. Step-up during restoration

Si la sesión sigue válida pero no cumple requirements actuales:

```text
STEP_UP_REQUIRED
```

en vez de invalidarla automáticamente.

---

## 259. Session cookie cleanup

Si el backend detecta:

```text
expired
revoked
unknown
malformed
```

podrá instruir la expiración de la cookie para reducir requests futuras inútiles.

---

## 260. Malformed session ID

Deberá rechazarse antes de store lookup cuando sea posible.

---

## 261. Session ID length limits

Evitan inputs abusivos.

---

## 262. Session ID parser

Deberá verificar únicamente estructura.

No confianza.

---

## 263. Session fixation via user-supplied IDs

No aceptar IDs arbitrarios creados por el cliente como nuevas sesiones autenticadas.

El servidor genera IDs.

---

## 264. Anonymous session adoption

Puede conservar estado anónimo, pero el ID deberá rotarse al autenticarse.

---

## 265. Logout CSRF

Una aplicación deberá decidir si logout por GET está permitido.

La recomendación de seguridad será tratar logout como acción state-changing y protegerla según el modelo HTTP/CSRF.

---

## 266. Sensitive operation confirmation

Revocar otras sesiones deberá requerir:

```text
current authenticated context
fresh authentication where appropriate
```

---

## 267. User-visible session management

API conceptual:

```php
Auth::sessions()->current();
Auth::sessions()->all();
Auth::sessions()->revoke($id);
Auth::sessions()->revokeOthers();
```

La API final deberá pasar por Authorization/ownership appropriate.

---

## 268. CurrentSessionReference

El Context podrá exponer una referencia segura a la sesión actual.

No necesariamente el raw session ID.

---

## 269. Session public identifier

Para UI de session management podrá existir un:

```text
SessionPublicIdentifier
```

distinto del bearer Session ID.

---

## 270. Razón

Nunca exponer el secret de la sesión en HTML/JSON para permitir revocarla.

---

## 271. Session public ID model

```text
sess_pub_...
```

podrá referenciar server-side al registro.

---

## 272. API for session revocation

Usar public identifier + current Authentication.

No enviar el actual bearer cookie secret como parámetro.

---

## 273. Session limits and device metadata

La policy podrá distinguir sesiones por device si existe device identity suficientemente estable.

---

## 274. Device names are untrusted display metadata

Ejemplo:

```text
"Chrome on Windows"
```

no debe utilizarse como credential.

---

## 275. Session management UI location

Ubicación aproximada derivada de IP es informativa, no prueba exacta de ubicación.

---

## 276. Browser fingerprinting

No deberá convertirse en dependencia obligatoria de Authentication debido a:

```text
privacy
instability
false positives
```

---

## 277. Persistent runtime

El Manager, Repository adapter y Validator podrán vivir como servicios compartidos si son stateless.

---

## 278. Request-specific state

Debe vivir en:

```text
AuthenticationScope
AuthenticationOperationContext
AuthenticationContextStorage
request memoization
```

---

## 279. Prohibido

```php
final class SessionAuthenticator
{
    private ?AuthenticationSession $currentSession;
}
```

si la instancia es singleton.

---

## 280. Scope cleanup

Al terminar cada request:

```text
AuthenticationContextStorage::clear()
SessionResolutionMemoization::clear()
CurrentSessionReference::clear()
```

---

## 281. FrankenPHP leakage test

Request A:

```text
Alice authenticated
```

Request B:

```text
no cookie
```

B nunca deberá recibir:

```text
Alice Context
```

---

## 282. Fiber safety

Concurrent requests dentro del mismo worker deberán tener:

```text
independent AuthenticationScope
```

---

## 283. Testing — creation

Casos:

```text
successful login
ID rotation
session persistence
context activation
cookie output
```

---

## 284. Testing — fixation

```text
anonymous ID A
login
authenticated ID must not equal A
```

---

## 285. Testing — restoration

```text
valid session
unknown session
expired idle
expired absolute
revoked
terminated
security version mismatch
firewall mismatch
tenant mismatch
```

---

## 286. Testing — logout

```text
current logout
double logout
logout other sessions
logout all
cookie expiration
```

---

## 287. Testing — concurrency

Especialmente:

```text
simultaneous requests during rotation
logout concurrent with request
revoke concurrent with restore
session limit concurrent logins
touch races
```

---

## 288. Testing — security version

```text
session v4
identity v4 → restore

session v4
identity v5 → reject
```

---

## 289. Testing — multi-tenant

```text
tenant A session on tenant A
tenant A session on tenant B
global identity explicit tenant switch
```

---

## 290. Testing — firewalls

```text
web session on web
web session on admin
shared realm + insufficient assurance
```

---

## 291. Testing — store failures

```text
read timeout
write failure
revoke failure
touch failure
partial persistence
```

---

## 292. Testing — persistent runtime

Ejecutar múltiples scopes consecutivos y concurrentes.

---

## 293. Property-based testing

Útil para:

```text
session lifecycle transitions
timeout boundaries
version comparisons
rotation family invariants
revocation idempotency
```

---

## 294. Fuzz testing

Adecuado para:

```text
session ID parser
client-side session serializer if supported
cookie parsing
```

---

## 295. Session state machine

```text
NEW
 │
 ▼
ACTIVE
 ├──────► EXPIRED
 ├──────► REVOKED
 ├──────► TERMINATED
 ├──────► COMPROMISED
 └──────► SUPERSEDED
```

Los terminal states no deberán volver a `ACTIVE`.

---

## 296. Rotation state transition

```text
Session A ACTIVE
      ↓
rotation
      ↓
Session A SUPERSEDED
Session B ACTIVE
```

---

## 297. Logout transition

```text
ACTIVE
  ↓
TERMINATED
```

---

## 298. Forced revocation

```text
ACTIVE
  ↓
REVOKED
```

---

## 299. Compromise transition

```text
ACTIVE
  ↓
COMPROMISED
  ↓
REVOKED
```

según implementación/policy.

---

## 300. Lifecycle invariant — Session

### AUTH-SESS-01

Session identifier presence does not imply authenticated state.

#### AUTH-SESS-02

Only validated AuthenticationSession can restore AuthenticationContext.

#### AUTH-SESS-03

AuthenticationSession never stores raw authentication secrets.

#### AUTH-SESS-04

Session IDs are server-generated, opaque and unpredictable.

#### AUTH-SESS-05

Authentication rotates the pre-authentication session ID.

#### AUTH-SESS-06

A revoked or expired session never restores a Context.

#### AUTH-SESS-07

Terminal session states cannot become ACTIVE again.

#### AUTH-SESS-08

Session restoration is firewall-aware.

#### AUTH-SESS-09

Tenant-bound sessions cannot cross tenants implicitly.

#### AUTH-SESS-10

Unknown store state never becomes authenticated state.

---

## 301. Lifecycle invariant — Context restoration

### AUTH-SESS-CTX-01

Restoration creates a new request-scoped AuthenticationContext.

#### AUTH-SESS-CTX-02

Runtime Context objects are never reused across requests.

#### AUTH-SESS-CTX-03

Identity security state is validated according to configured freshness policy.

#### AUTH-SESS-CTX-04

SecurityVersion mismatch invalidates or restricts restoration.

#### AUTH-SESS-CTX-05

Restored assurance does not bypass freshness requirements.

#### AUTH-SESS-CTX-06

A restored Context can require step-up without destroying base Authentication.

#### AUTH-SESS-CTX-07

Serialized session snapshots use an allowlisted versioned schema.

---

## 302. Security invariant — Cookie

### AUTH-SESS-COOKIE-01

Authentication cookies are HttpOnly by default.

#### AUTH-SESS-COOKIE-02

Production authentication cookies are Secure by default.

#### AUTH-SESS-COOKIE-03

SameSite policy is explicit.

#### AUTH-SESS-COOKIE-04

Cookie Domain/Path use minimum required scope.

#### AUTH-SESS-COOKIE-05

Session secrets are never placed in URLs.

#### AUTH-SESS-COOKIE-06

Cookie deletion does not replace server-side revocation.

---

## 303. Security invariant — Concurrency

### AUTH-SESS-CONC-01

Session rotation is atomic or safely coordinated.

#### AUTH-SESS-CONC-02

Replay of superseded identifiers can be detected where rotation tracking is enabled.

#### AUTH-SESS-CONC-03

Concurrent activity updates cannot resurrect expired/revoked sessions.

#### AUTH-SESS-CONC-04

Session limit policies handle simultaneous logins safely.

#### AUTH-SESS-CONC-05

Commit-time identity security checks prevent stale authentication creation.

---

## 304. Security invariant — Persistent runtime

### AUTH-SESS-RT-01

Current session state never resides in mutable process-global storage.

#### AUTH-SESS-RT-02

Authentication context is scope-bound.

#### AUTH-SESS-RT-03

Request memoization is cleared after execution.

#### AUTH-SESS-RT-04

Shared Session services remain stateless.

#### AUTH-SESS-RT-05

Fiber/coroutine executions cannot access each other's AuthenticationContext.

---

## 305. Anti-pattern — `user_id` equals auth

Evitar:

```php
if ($_SESSION['user_id']) {
    // authenticated
}
```

sin un modelo de AuthenticationSession validado.

---

## 306. Anti-pattern — no session rotation

Evitar conservar el mismo ID antes y después del login.

---

## 307. Anti-pattern — full User serialization

Evitar:

```php
$_SESSION['user'] = serialize($user);
```

como fundamento del sistema.

---

## 308. Anti-pattern — cookie contains permissions

Evitar:

```text
role=admin
permissions=[...]
```

como estado de confianza client-side sin architecture específica.

---

## 309. Anti-pattern — session survives SecurityVersion change

Si la policy utiliza version invalidation, la restauración debe detectarla.

---

## 310. Anti-pattern — IP hard binding by default

Produce problemas de UX y falsos positivos.

---

## 311. Anti-pattern — Session ID in logs

Nunca registrar completo.

---

## 312. Anti-pattern — session ID in query string

Debe prohibirse en adapters estándar.

---

## 313. Anti-pattern — logout only clears cookie

Server-side session debe revocarse/terminarse.

---

## 314. Anti-pattern — infinite sliding lifetime

Siempre debe existir posibilidad de absolute timeout o equivalente cuando la policy lo requiera.

---

## 315. Anti-pattern — session restoration queries everything

No cargar profile, roles, permissions y datos de aplicación completos para cada `Auth::check()`.

---

## 316. Anti-pattern — session lock around whole request

No deberá ser requisito del Core.

---

## 317. Anti-pattern — static current session

Especialmente peligroso bajo FrankenPHP.

---

## 318. Componentes principales

```text
AuthenticationSession
AuthenticationSessionId
AuthenticationSessionStatus
AuthenticationSessionFactory
AuthenticationSessionRepository
AuthenticationSessionValidator
AuthenticationContextSnapshot
AuthenticationContextSnapshotSerializer
AuthenticationContextRestorer
SessionAuthenticator
SessionAuthenticationStrategy
AuthenticationSessionPolicy
SessionPolicyResolver
```

---

## 319. Componentes de lifecycle

```text
AuthenticationSessionPersistenceCoordinator
AuthenticationSessionRotator
AuthenticationSessionRevoker
AuthenticationSessionPruner
AuthenticationSessionRegistry
AuthenticationSessionFamily
ConcurrentSessionPolicy
AuthenticationSessionIdGenerator
SessionRevocationReason
```

---

## 320. Componentes de integración HTTP

```text
AuthenticationSessionCookieManager
AuthenticationSessionCookiePolicy
HttpSessionIdentifierExtractor
HttpSessionIdentifierWriter
```

---

## 321. Namespace sugerido

```text
VoltStack\Quantum\Auth\Session
VoltStack\Quantum\Auth\Session\Contracts
VoltStack\Quantum\Auth\Session\Persistence
VoltStack\Quantum\Auth\Session\Restoration
VoltStack\Quantum\Auth\Session\Lifecycle
VoltStack\Quantum\Auth\Session\Cookie
VoltStack\Quantum\Auth\Session\Policy
VoltStack\Quantum\Auth\Session\Store
```

---

## 322. Estructura sugerida

```text
src/Quantum/Auth/
└── Session/
    ├── Contracts/
    │   ├── AuthenticationSessionRepositoryInterface.php
    │   ├── AuthenticationSessionValidatorInterface.php
    │   └── AuthenticationContextRestorerInterface.php
    │
    ├── AuthenticationSession.php
    ├── AuthenticationSessionId.php
    ├── AuthenticationSessionStatus.php
    ├── AuthenticationSessionFactory.php
    ├── AuthenticationSessionPolicy.php
    ├── AuthenticationSessionValidationResult.php
    │
    ├── Persistence/
    │   ├── AuthenticationSessionPersistenceCoordinator.php
    │   ├── AuthenticationContextSnapshot.php
    │   └── AuthenticationContextSnapshotSerializer.php
    │
    ├── Restoration/
    │   ├── AuthenticationContextRestorer.php
    │   ├── SessionRestorationContext.php
    │   └── SessionRestorationResult.php
    │
    ├── Lifecycle/
    │   ├── AuthenticationSessionRotator.php
    │   ├── AuthenticationSessionRevoker.php
    │   ├── AuthenticationSessionPruner.php
    │   ├── AuthenticationSessionRegistry.php
    │   ├── AuthenticationSessionFamily.php
    │   └── ConcurrentSessionPolicy.php
    │
    ├── Cookie/
    │   ├── AuthenticationSessionCookieManager.php
    │   ├── AuthenticationSessionCookiePolicy.php
    │   ├── HttpSessionIdentifierExtractor.php
    │   └── HttpSessionIdentifierWriter.php
    │
    └── Store/
        ├── RedisAuthenticationSessionRepository.php
        ├── DatabaseAuthenticationSessionRepository.php
        ├── FileAuthenticationSessionRepository.php
        └── InMemoryAuthenticationSessionRepository.php
```

---

## 323. Configuración conceptual

```php
return [

    'authentication' => [

        'sessions' => [

            'default_store' => 'redis',

            'idle_timeout' => 1800,

            'absolute_timeout' => 43200,

            'rotate_on_login' => true,

            'rotate_on_step_up' => true,

            'identity_refresh' => 'security_version',

            'cookie' => [
                'name' => '__Host-voltstack_session',
                'http_only' => true,
                'secure' => true,
                'same_site' => 'lax',
                'path' => '/',
            ],

        ],

    ],

];
```

Los valores son ilustrativos y deberán adaptarse al perfil real de seguridad.

---

## 324. Firewall configuration example

```php
'firewalls' => [

    'web' => [
        'state' => 'stateful',

        'session' => [
            'policy' => 'web',
            'idle_timeout' => '2 hours',
            'absolute_timeout' => '24 hours',
        ],
    ],

    'admin' => [
        'state' => 'stateful',

        'session' => [
            'policy' => 'admin',
            'idle_timeout' => '15 minutes',
            'absolute_timeout' => '8 hours',
            'minimum_assurance' => 'AAL2',
        ],
    ],

];
```

---

## 325. Flujo completo de login stateful

```text
Password/Passkey/OIDC Authentication
            ↓
AuthenticationDecision = AUTHENTICATED
            ↓
AuthenticationContextFactory
            ↓
AuthenticationContext
            ↓
Commit-time Identity Security Check
            ↓
SessionAuthenticationStrategy
            ↓
Rotate pre-auth session ID
            ↓
AuthenticationSessionFactory
            ↓
Persist AuthenticationSession
            ↓
Set secure session cookie
            ↓
Activate request AuthenticationContext
            ↓
AuthenticationSucceeded
```

---

## 326. Flujo completo de request posterior

```text
HTTP Request
    ↓
Session Cookie
    ↓
SessionAuthenticator
    ↓
SessionReferenceCredential
    ↓
AuthenticationSessionRepository
    ↓
AuthenticationSession
    ↓
SessionValidator
    ├── active?
    ├── idle timeout?
    ├── absolute timeout?
    ├── firewall?
    ├── tenant?
    ├── security version?
    └── identity eligible?
    ↓
AuthenticationContextRestorer
    ↓
Request-scoped AuthenticationContext
    ↓
Application / Authorization
```

---

## 327. Flujo de sesión expirada

```text
Cookie
  ↓
Session found
  ↓
Idle timeout exceeded
  ↓
mark/recognize EXPIRED
  ↓
clear cookie
  ↓
no AuthenticationContext
```

---

## 328. Flujo SecurityVersion mismatch

```text
Session:
    securityVersion = 8

Identity:
    securityVersion = 9

        ↓

SessionValidator
        ↓
STALE_SECURITY_VERSION
        ↓
revoke session
        ↓
clear cookie
        ↓
reauthentication / recovery
```

---

## 329. Flujo step-up

```text
Session restored
    ↓
Context AAL1
    ↓
Route requires fresh AAL2
    ↓
STEP_UP_REQUIRED
    ↓
Passkey/TOTP
    ↓
new AuthenticationEvidence
    ↓
Context AAL2
    ↓
rotate session ID
    ↓
update AuthenticationSession snapshot
```

---

## 330. Flujo logout

```text
Current AuthenticationContext
        ↓
CurrentSessionReference
        ↓
AuthenticationSessionRevoker
        ↓
TERMINATED
        ↓
ContextStorage clear
        ↓
cookie expired
        ↓
Logout event
```

---

## 331. Flujo logout all

```text
Fresh authenticated request
        ↓
ForcedLogoutService
        ↓
IdentityReference
        ↓
revoke all AuthenticationSessions
        ↓
SecurityVersion++
        ↓
revoke remember-me/token families according to policy
        ↓
optionally establish new current session
```

---

## 332. Flujo session theft/replay

```text
Session A rotated → Session B

Later:
    Session A reused
        ↓
Rotation Replay Detector
        ↓
possible compromise
        ↓
policy:
    revoke SessionFamily
    require reauthentication
    raise security event
```

---

## 333. Flujo multi-tenant

```text
Session:
    Identity = User 15
    Tenant = ACME

Request:
    Tenant = Globex

        ↓

SessionValidator
        ↓
TENANT_MISMATCH
        ↓
no AuthenticationContext
```

---

## 334. Flujo FrankenPHP

```text
Worker
│
├── immutable Session services
│
├── Request A Scope
│      ├── Session A
│      └── AuthenticationContext Alice
│
│   request ends
│      ↓
│   scope reset
│
└── Request B Scope
       ├── no Session
       └── no AuthenticationContext
```

---

## 335. Decisiones arquitectónicas principales

VoltStack adoptará como principios:

```text
1. Session is not Authentication by itself.
2. AuthenticationSession is separate from application session.
3. Session identifiers are opaque server-generated secrets.
4. Login rotates the session identifier.
5. AuthenticationContext is reconstructed per execution.
6. Raw credentials never enter session persistence.
7. SecurityVersion participates in restoration.
8. Tenant and Firewall bindings are explicit.
9. Session state is centrally revocable.
10. Persistent runtime state is always scope-isolated.
11. Assurance and freshness survive only under explicit rules.
12. Cookie-based authentication requires CSRF integration.
```

---

## 336. Comparación conceptual con Laravel y Symfony

VoltStack conservará del enfoque Laravel:

```text
simple session-based web authentication
easy Auth::check()
easy Auth::user()
session regeneration on login
guard ergonomics
multiple session stores
```

y del enfoque Symfony:

```text
firewall-specific state
token/context restoration concepts
user refresh
security-state checks
authentication entry points
structured authenticator lifecycle
```

Pero añadirá una separación más fuerte entre:

```text
Application Session
AuthenticationSession
AuthenticationContext
Identity Security State
Authentication Evidence
Session Persistence
```

junto con:

```text
SecurityVersion
Assurance restoration
Tenant binding
Session families
Distributed invalidation
Persistent-runtime isolation
```

---

## 337. Criterios de aceptación

El subsistema será considerado completo cuando:

1. exista `AuthenticationSession`;
2. la sesión esté separada del Application Session;
3. soporte SessionAuthenticator;
4. soporte server-side stores;
5. soporte Redis;
6. soporte database;
7. permita file/in-memory adapters;
8. use IDs opacos y de alta entropía;
9. rote el ID después de login;
10. prevenga session fixation;
11. persista snapshots seguros;
12. restaure Context por request;
13. soporte idle timeout;
14. soporte absolute timeout;
15. soporte renewal;
16. soporte rotation;
17. soporte revocation;
18. soporte logout;
19. soporte logout all;
20. soporte concurrent sessions;
21. soporte session limits;
22. soporte device metadata;
23. soporte optional device binding;
24. soporte tenant binding;
25. soporte Firewall binding;
26. valide SecurityVersion;
27. integre Identity Security State;
28. soporte restored assurance;
29. soporte step-up;
30. soporte reauthentication;
31. soporte secure cookies;
32. integre CSRF;
33. soporte SPA;
34. maneje concurrent requests;
35. soporte session family/replay detection;
36. soporte pruning;
37. soporte distributed stores;
38. sea observable;
39. sea auditable;
40. sea seguro con FrankenPHP;
41. sea fiber/coroutine safe;
42. no almacene raw credentials;
43. no mezcle Authentication y Authorization.

---

## 338. Regla arquitectónica final

VoltStack deberá preservar:

```text
AUTHENTICATION PROOF
        ↓
AUTHENTICATION CONTEXT
        ↓
AUTHENTICATION SESSION
        ↓
PERSISTED TRUST REFERENCE
        ↓
FUTURE REQUEST
        ↓
SESSION VALIDATION
        ↓
IDENTITY SECURITY REVALIDATION
        ↓
NEW REQUEST-SCOPED AUTHENTICATION CONTEXT
```

La regla fundamental será:

> **Una sesión no conserva eternamente la confianza obtenida durante el login; conserva una referencia revocable y versionada que permite a VoltStack decidir si esa confianza todavía puede reconstruirse.**

Por tanto:

> **La autenticación persistente de VoltStack no se basará en recordar simplemente “qué usuario inició sesión”, sino en poder demostrar en cada restauración que la sesión, la Identity, el tenant, el Firewall y el estado de seguridad siguen siendo compatibles con el contexto autenticado.**

---

## 339. Próximo documento

El siguiente documento recomendado será:

```text
13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md
```

Su responsabilidad será definir completamente el mecanismo de autenticación persistente de larga duración distinto de una sesión normal:

```text
RememberMeCredential
persistent login tokens
selector + validator patterns
opaque token design
token hashing
token rotation
token families
single-use rotation
replay detection
device binding
expiration
SecurityVersion validation
session bootstrap
reduced assurance
freshness semantics
revocation
logout interaction
password reset interaction
credential compromise
multiple devices
tenant binding
cookie policy
token theft mitigation
distributed stores
audit
observability
testing
FrankenPHP safety
```

La separación será fundamental:

```text
AuthenticationSession
    short/medium-lived authenticated state

Remember-Me Credential
    long-lived credential capable of creating
    a new AuthenticationSession
```

Esto impedirá que VoltStack trate un cookie persistente como si fuera simplemente una sesión con duración excesiva.
