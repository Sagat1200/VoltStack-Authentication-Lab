# VoltStack Authentication System

## 13 — Remember-Me, Persistent Login and Long-Lived Authentication Credential System

- **Archivo:** `13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del subsistema de autenticación persistente de larga duración

**Depende especialmente de:**

- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`

---

## 1. Propósito

Este documento define el sistema mediante el cual VoltStack podrá reconocer nuevamente a una Identity después de que su `AuthenticationSession` normal haya desaparecido o expirado, sin exigir inmediatamente una nueva introducción de credenciales primarias.

El sistema deberá proporcionar capacidades equivalentes conceptualmente a:

```text
Laravel Remember Me
Symfony RememberMe
Persistent Login
Trusted Browser Login
Long-Lived Authentication Credential
```

pero utilizando un modelo explícito, revocable, rotatorio, auditable y preparado para entornos distribuidos.

El flujo fundamental será:

```text
AuthenticationSession unavailable
            │
            ▼
Persistent Authentication Credential
            │
            ▼
RememberMeAuthenticator
            │
            ▼
Credential Verification
            │
            ▼
Identity Security Validation
            │
            ▼
Authentication Decision
            │
            ▼
Reduced/Freshness-Aware AuthenticationContext
            │
            ▼
NEW AuthenticationSession
```

---

## 2. Principio fundamental

VoltStack deberá separar completamente:

```text
AuthenticationSession
```

de:

```text
PersistentAuthenticationCredential
```

Una sesión representa:

```text
current authenticated state
```

mientras que una credencial Remember-Me representa:

```text
evidence allowing authentication
to be re-established later
```

Por tanto:

> **Remember-Me no será una sesión extremadamente larga. Será una credencial autenticadora independiente capaz de crear nuevas sesiones.**

---

## 3. Modelo conceptual

```text
Primary Authentication
        │
        ▼
AuthenticationContext
        │
        ├──────────────► AuthenticationSession
        │
        └──────────────► PersistentAuthenticationCredential
                                  │
                                  │ days/weeks later
                                  ▼
                         RememberMeAuthenticator
                                  │
                                  ▼
                         Credential Validation
                                  │
                                  ▼
                         AuthenticationContext
                                  │
                                  ▼
                         AuthenticationSession
```

---

## 4. Objetivos

El subsistema deberá soportar:

```text
persistent login
opaque remember-me tokens
token hashing
token rotation
token families
token replay detection
expiration
revocation
multiple devices
identity binding
tenant binding
firewall binding
SecurityVersion integration
CredentialVersion integration
device metadata
assurance semantics
authentication freshness
session bootstrap
logout integration
logout-all integration
password reset invalidation
account disablement
credential compromise response
distributed persistence
audit
observability
FrankenPHP isolation
```

---

## 5. No objetivos

Este sistema no deberá sustituir:

```text
AuthenticationSession
Password Authentication
MFA
Passkeys
OAuth/OIDC
API tokens
Password Reset
Authorization
```

---

## 6. Terminología

VoltStack utilizará preferentemente:

```text
PersistentAuthenticationCredential
```

como concepto de dominio.

`RememberMe` será una implementación/experiencia específica.

Esto permitirá en el futuro soportar:

```text
Remember Me
Trusted Browser
Long-Lived Device Credential
Persistent Reauthentication Credential
```

sin acoplar el dominio a una única UI.

---

## 7. PersistentAuthenticationCredential

Modelo conceptual:

```php
final readonly class PersistentAuthenticationCredential
{
    public function __construct(
        public PersistentCredentialId $id,
        public PersistentCredentialFamilyId $familyId,
        public IdentityReference $identity,
        public PersistentCredentialType $type,
        public PersistentCredentialStatus $status,
        public SecretDigest $secretDigest,
        public string $firewall,
        public ?TenantIdentifier $tenant,
        public SecurityVersion $securityVersion,
        public AuthenticationMethod $originMethod,
        public AuthenticationAssurance $originAssurance,
        public \DateTimeImmutable $issuedAt,
        public \DateTimeImmutable $expiresAt,
        public ?\DateTimeImmutable $lastUsedAt,
        public ?DeviceReference $device,
    ) {}
}
```

---

## 8. El token cliente no es el registro

Debe distinguirse:

```text
Client Token
```

de:

```text
PersistentAuthenticationCredential Record
```

El navegador posee un secret.

El servidor mantiene el registro necesario para validarlo.

---

## 9. Token model recomendado

VoltStack utilizará preferentemente un modelo:

```text
selector.secret
```

Conceptualmente:

```text
rm_<selector>.<secret>
```

donde:

```text
selector
    identifies the record/family

secret
    proves possession
```

---

## 10. Selector

El selector:

```text
random
opaque
non-semantic
unique
```

podrá almacenarse de forma indexable.

No deberá contener:

```text
user_id
email
tenant_id
roles
```

como información interpretable.

---

## 11. Secret

El secret deberá ser:

```text
cryptographically random
high entropy
unpredictable
```

y será tratado como bearer credential.

---

## 12. Persistencia del secret

El servidor nunca deberá almacenar:

```text
raw remember-me secret
```

sino:

```text
cryptographic digest
```

---

## 13. Token lookup

Flujo:

```text
cookie
  ↓
selector.secret
  ↓
selector lookup
  ↓
credential record
  ↓
hash(secret)
  ↓
constant-time digest comparison
```

---

## 14. Diferencia respecto a password hashing

Un Remember-Me secret generado por CSPRNG posee alta entropía.

Por ello no necesita necesariamente algoritmos lentos como:

```text
Argon2id
bcrypt
```

El sistema podrá utilizar un digest criptográfico apropiado para tokens aleatorios.

---

## 15. Password hashing policy independiente

No reutilizar:

```text
PasswordHasher
```

para Remember-Me tokens simplemente porque ambos son secrets.

Deberá existir:

```text
PersistentCredentialSecretHasher
```

o equivalente.

---

## 16. Token entropy

La entropía deberá ser suficientemente alta para hacer impracticable brute-force.

El framework no deberá permitir tokens:

```text
incrementales
timestamp-based
UUID v1
uniqid()
mt_rand()
```

como secretos.

---

## 17. CSPRNG

La generación utilizará primitivas equivalentes a:

```php
random_bytes()
```

---

## 18. PersistentCredentialTokenGenerator

Contrato conceptual:

```php
interface PersistentCredentialTokenGeneratorInterface
{
    public function generate(): PersistentCredentialToken;
}
```

---

## 19. Token parser

El parser deberá:

```text
validate format
enforce maximum length
extract selector
extract secret
reject malformed encoding
```

sin establecer confianza.

---

## 20. PersistentCredentialStatus

Estados sugeridos:

```text
ACTIVE
ROTATED
REVOKED
EXPIRED
COMPROMISED
SUPERSEDED
```

---

## 21. ACTIVE

Puede utilizarse para Authentication.

---

## 22. ROTATED

Ya fue utilizado y sustituido por otro token.

Su reutilización puede representar replay.

---

## 23. REVOKED

No puede volver a autenticar.

---

## 24. EXPIRED

Superó su lifetime.

---

## 25. COMPROMISED

Existe evidencia suficiente para tratarlo como potencialmente robado.

---

## 26. SUPERSEDED

Puede utilizarse cuando una política reemplaza completamente una credencial/familia.

---

## 27. Token families

Cada Remember-Me login podrá crear una:

```text
PersistentCredentialFamily
```

---

## 28. Family model

```text
Identity
   │
   ├── Laptop Family
   │      ├── Token 1
   │      ├── Token 2
   │      └── Token 3
   │
   └── Phone Family
          ├── Token 1
          └── Token 2
```

---

## 29. Razón de token families

Permiten:

```text
rotation
replay detection
device revocation
credential history
selective logout
compromise containment
```

---

## 30. PersistentCredentialFamily

Conceptualmente:

```php
final readonly class PersistentCredentialFamily
{
    public function __construct(
        public PersistentCredentialFamilyId $id,
        public IdentityReference $identity,
        public string $firewall,
        public ?TenantIdentifier $tenant,
        public PersistentCredentialFamilyStatus $status,
        public \DateTimeImmutable $createdAt,
        public \DateTimeImmutable $expiresAt,
        public ?DeviceReference $device,
    ) {}
}
```

---

## 31. Una familia por dispositivo

Modelo recomendado:

```text
one persistent credential family
per trusted browser/device
```

cuando Device System esté disponible.

---

## 32. Multiple devices

Una Identity podrá tener:

```text
Laptop
Phone
Tablet
```

cada uno con familia independiente.

---

## 33. Revocación selectiva

El usuario podrá:

```text
Forget this device
```

sin invalidar necesariamente los demás.

---

## 34. Token rotation

Cada uso exitoso deberá poder producir:

```text
Token N
   ↓
validate
   ↓
mark ROTATED
   ↓
issue Token N+1
```

---

## 35. Rotación por uso

Será la estrategia recomendada.

Reduce la ventana de reutilización de un token robado.

---

## 36. Single-use semantics

Idealmente:

```text
each persistent token secret
can successfully authenticate once
```

antes de ser reemplazado.

---

## 37. Rotation flow

```text
Cookie Token A
      ↓
lookup A
      ↓
validate A
      ↓
Identity validation
      ↓
mark A ROTATED
      ↓
create Token B
      ↓
persist B
      ↓
create AuthenticationSession
      ↓
send Cookie B
```

---

## 38. Atomic rotation

La transición:

```text
A ACTIVE
    ↓
A ROTATED
B ACTIVE
```

deberá ser atómica o coordinada con garantías equivalentes.

---

## 39. Replay detection

Supongamos:

```text
Attacker steals Token A
User uses Token A
VoltStack rotates → Token B
Attacker later uses Token A
```

VoltStack detectará:

```text
Token A status = ROTATED
```

Esto constituye una señal fuerte de replay.

---

## 40. Replay response

La policy podrá:

```text
mark family COMPROMISED
revoke Token B
revoke all descendants
terminate sessions created from family
require fresh authentication
emit security event
notify Identity Security State
```

---

## 41. ReplayDetectionPolicy

Posibles niveles:

```text
DISABLED
AUDIT_ONLY
REVOKE_FAMILY
REVOKE_IDENTITY_PERSISTENT_CREDENTIALS
ESCALATE_SECURITY_VERSION
```

---

## 42. Default recomendado

```text
REVOKE_FAMILY
```

con posibilidad de policy más estricta.

---

## 43. Concurrencia legítima

Browsers pueden producir requests paralelos.

Esto genera un problema:

```text
Request A uses Token 1
Request B simultaneously uses Token 1
```

sin que exista necesariamente robo.

---

## 44. Rotation concurrency problem

Si A rota primero:

```text
Token 1 → Token 2
```

B podría parecer replay.

El sistema deberá distinguir razonablemente:

```text
concurrent legitimate use
```

de:

```text
later replay
```

---

## 45. Rotation grace strategy

Podrá existir una ventana extremadamente corta o mecanismo de coordination.

Ejemplo conceptual:

```text
Token 1
    ↓
rotation lock
    ↓
Token 2
```

Requests concurrentes podrán converger hacia Token 2 sin crear múltiples ramas.

---

## 46. No long grace period

No deberá mantenerse Token 1 reutilizable durante minutos.

Eso destruiría buena parte del valor de la rotación.

---

## 47. Rotation sequence

Cada familia podrá tener:

```text
generation
```

Ejemplo:

```text
Family F
generation 1
generation 2
generation 3
```

---

## 48. Generation monotonicity

Dentro de una familia:

```text
generation N+1 > generation N
```

---

## 49. Token lineage

Cada credential podrá registrar:

```text
parentCredentialId
generation
```

para análisis y replay detection.

---

## 50. Expiration

Deberán distinguirse:

```text
token expiration
family expiration
```

---

## 51. Token expiration

Puede ser relativamente corta si rota frecuentemente.

---

## 52. Family expiration

Representa cuánto tiempo puede durar el "remember this device" original.

---

## 53. Rotation does not reset family lifetime

Ejemplo:

```text
Family created:
August 1

Family lifetime:
30 days

Token rotated:
August 20
```

La familia sigue expirando:

```text
August 31
```

no:

```text
September 19
```

salvo una policy explícita diferente.

---

## 54. Persistent login lifetime

Debe configurarse por:

```text
firewall
application security profile
tenant
identity type
```

---

## 55. Absolute persistent lifetime

Se recomienda mantener un límite absoluto.

Evitar credenciales que vivan indefinidamente solo porque se utilizan.

---

## 56. Idle persistent lifetime

Opcionalmente:

```text
family unused for N days
    ↓
expire
```

---

## 57. RememberMePolicy

Podrá definir:

```text
enabled
token_lifetime
family_lifetime
idle_lifetime
rotate_on_use
replay_detection
max_families
device_binding
security_version_validation
assurance
cookie
```

---

## 58. Credential issuance

Remember-Me solo deberá emitirse después de Authentication suficientemente fuerte.

---

## 59. Issuance policy

Ejemplo:

```text
password authentication
    → allowed

password + MFA
    → allowed

remember-me authentication
    → do not issue new family

anonymous
    → impossible
```

---

## 60. No recursive remember-me trust

Una Authentication creada desde Remember-Me no deberá crear silenciosamente una nueva familia independiente.

---

## 61. Family continuation

Podrá rotar la familia existente.

Pero no:

```text
RememberMe A
    ↓
creates unrelated RememberMe B
    ↓
infinite credential proliferation
```

---

## 62. User consent

La creación deberá normalmente depender de una intención explícita:

```text
Remember me
Trust this browser
Keep me signed in
```

según producto.

---

## 63. Authentication request

El login podrá incluir:

```php
RememberMeIntent::REQUESTED
```

---

## 64. Server policy remains authoritative

Aunque el cliente solicite Remember-Me:

```text
client requests persistent login
```

el servidor podrá responder:

```text
DENIED_BY_POLICY
```

---

## 65. Examples

No permitir persistent login para:

```text
high-security admin
shared kiosk
suspended identity
temporary account
risk-elevated authentication
```

---

## 66. Persistent credential issuance service

```php
interface PersistentCredentialIssuerInterface
{
    public function issue(
        AuthenticationContext $context,
        PersistentCredentialIssuanceContext $issuance
    ): PersistentCredentialIssueResult;
}
```

---

## 67. Issuance validation

Antes de emitir:

```text
Identity still eligible?
SecurityVersion unchanged?
Authentication method allowed?
Assurance sufficient?
Tenant valid?
Firewall permits persistent login?
Risk acceptable?
Family limit available?
```

---

## 68. Commit-time validation

Igual que Session Authentication, la Identity deberá validarse nuevamente en el punto de commit si el flujo fue largo.

---

## 69. Cookie transport

Remember-Me normalmente se transportará mediante:

```text
HttpOnly persistent cookie
```

---

## 70. Cookie security

Deberá favorecer:

```text
HttpOnly
Secure
appropriate SameSite
minimum Domain
minimum Path
```

---

## 71. Persistent cookie expiration

La expiración del navegador deberá alinearse con la policy del servidor.

Pero:

> **La fecha del cookie nunca será la fuente autoritativa de expiración.**

---

## 72. Server-side expiration

Aunque el cliente manipule la fecha:

```text
credential.expiresAt
```

será validado server-side.

---

## 73. Cookie name isolation

Podrá utilizarse un nombre diferente de la session cookie:

```text
__Host-voltstack_remember
```

---

## 74. No user information in cookie

Evitar:

```text
remember_user=15
remember_email=...
```

---

## 75. No raw Identity ID as proof

Conocer:

```text
user_id
```

nunca constituye autenticación.

---

## 76. RememberMeAuthenticator

Será un Authenticator estándar del sistema.

---

## 77. Activación

Deberá ejecutarse normalmente cuando:

```text
no valid AuthenticationSession exists
AND
persistent credential is present
```

---

## 78. Priority

Conceptualmente:

```text
valid active AuthenticationSession
        ↓
use session

otherwise
        ↓
try RememberMeAuthenticator
```

---

## 79. No needless persistent token use

Si existe una sesión válida:

```text
do not consume/rotate remember-me token
```

en cada request.

---

## 80. Razón

De otro modo:

```text
every request
    ↓
persistent token rotation
```

sería costoso e innecesario.

---

## 81. Session bootstrap

Remember-Me se usa principalmente para:

```text
bootstrap a new AuthenticationSession
```

cuando la sesión normal ya no existe.

---

## 82. Authentication flow

```text
Request
  ↓
No valid session
  ↓
Remember-Me cookie detected
  ↓
RememberMeAuthenticator
  ↓
PersistentCredentialTokenParser
  ↓
Repository lookup
  ↓
Secret verification
  ↓
Credential lifecycle validation
  ↓
Identity resolution
  ↓
Identity security validation
  ↓
Authentication eligibility
  ↓
AuthenticationContext
  ↓
Rotate persistent credential
  ↓
Create AuthenticationSession
```

---

## 83. Authentication Evidence

Un Remember-Me token válido deberá producir:

```text
PersistentCredentialEvidence
```

---

## 84. Evidence model

Podrá contener:

```text
credential family reference
credential generation
original authentication method
original assurance
issuedAt
lastUsedAt
device reference
```

sin incluir el raw secret.

---

## 85. Authentication method

El Context deberá poder indicar:

```text
AuthenticationMethod::REMEMBER_ME
```

o provenance equivalente.

---

## 86. Assurance

Remember-Me no deberá heredar ciegamente el assurance original.

---

## 87. Example

Login original:

```text
password + passkey
AAL2
```

Tres semanas después:

```text
Remember-Me token
```

no necesariamente debe producir automáticamente:

```text
fresh AAL2
```

---

## 88. Persistent authentication assurance

La policy podrá producir:

```text
AAL1
```

o:

```text
AAL2_STALE
```

conceptualmente, dependiendo del assurance model.

---

## 89. Freshness

El Context deberá conservar:

```text
primaryAuthenticatedAt
persistentCredentialAuthenticatedAt
```

como conceptos diferentes.

---

## 90. Sensitive operations

Ejemplo:

```text
Remember-Me restored session
        ↓
view dashboard
        → allowed

change password
        → reauthentication required

wire transfer
        → step-up required
```

---

## 91. Reauthentication boundary

El framework deberá permitir:

```text
requiresFreshAuthentication()
```

sin invalidar toda la sesión Remember-Me.

---

## 92. Identity resolution

El token deberá apuntar a:

```text
IdentityReference
```

no necesariamente a un ORM User model.

---

## 93. Provider independence

El Persistent Credential Repository no deberá depender de Eloquent/Doctrine.

---

## 94. IdentityProvider resolution

Flujo:

```text
PersistentCredential
       ↓
IdentityReference
       ↓
IdentityProviderResolver
       ↓
Identity
```

---

## 95. Deleted Identity

Si la Identity ya no existe:

```text
revoke credential family
authentication rejected
```

---

## 96. Disabled Identity

Si:

```text
IdentitySecurityState = DISABLED
```

Remember-Me deberá fallar.

---

## 97. Locked Identity

La policy podrá decidir si el lock bloquea también Persistent Login.

Para account-security locks, normalmente sí.

---

## 98. Expired account

No deberá ser reactivado por Remember-Me.

---

## 99. SecurityVersion

Cada familia/token deberá conservar la SecurityVersion relevante.

---

## 100. Validation

```text
credential.securityVersion
        ==
identity.securityVersion
```

será el comportamiento base recomendado.

---

## 101. Version mismatch

Produce:

```text
STALE_SECURITY_STATE
```

y deberá impedir Authentication.

---

## 102. Password change

La aplicación podrá incrementar:

```text
SecurityVersion
```

para invalidar:

```text
sessions
remember-me credentials
other persistent credentials
```

de forma central.

---

## 103. Password reset

Normalmente deberá ser aún más estricto:

```text
revoke persistent credential families
```

---

## 104. CredentialVersion

Podrá utilizarse para invalidación selectiva.

Ejemplo:

```text
remember-me issued from PasswordCredential v7
password changes to v8
```

Policy:

```text
invalidate family
```

---

## 105. OriginCredentialReference

Una familia podrá conservar opcionalmente:

```text
originCredentialReference
```

sin almacenar secretos.

---

## 106. Origin authentication method

Ejemplos:

```text
PASSWORD
PASSKEY
OIDC
PASSWORD_PLUS_TOTP
```

---

## 107. Origin method use

Puede afectar:

```text
persistent credential eligibility
assurance
reauthentication policy
audit
risk
```

---

## 108. Federated login

VoltStack podrá emitir Remember-Me después de OIDC/SAML si la policy lo permite.

---

## 109. Federation does not imply external token persistence

No es necesario guardar:

```text
OIDC access token
refresh token
```

dentro del Remember-Me credential.

Son dominios separados.

---

## 110. External provider account disabled

Si la aplicación depende de verificar periódicamente el proveedor externo, deberá definir una Identity refresh policy.

No deberá asumirse universalmente.

---

## 111. Firewall binding

Cada persistent credential deberá estar ligado a:

```text
firewall
```

o Authentication Realm.

---

## 112. Cross-firewall use

Por defecto:

```text
remember-me from web
        ≠
remember-me for admin
```

---

## 113. Shared realms

Podrán configurarse explícitamente.

Pero el Firewall más sensible podrá exigir fresh authentication aunque exista persistent credential compartido.

---

## 114. Tenant binding

En sistemas multi-tenant:

```text
PersistentCredential
    tenant = ACME
```

no deberá autenticar automáticamente:

```text
tenant = Globex
```

---

## 115. Global account

Si una Identity puede pertenecer a varios tenants:

```text
persistent global identity recognition
        ↓
tenant selection
        ↓
tenant-specific eligibility validation
        ↓
AuthenticationContext
```

deberá ser un flow explícito.

---

## 116. Tenant switch

No mutar simplemente:

```text
credential.tenant
```

---

## 117. Tenant-scoped families

Podrán existir familias independientes:

```text
Laptop / ACME
Laptop / Globex
```

si la arquitectura lo requiere.

---

## 118. Device binding

Remember-Me es un candidato natural para Device integration.

---

## 119. DeviceReference

Una familia podrá asociarse a:

```text
DeviceReference
```

---

## 120. Device binding levels

```text
NONE
OBSERVATIONAL
SOFT
STRICT
CRYPTOGRAPHIC
```

---

## 121. Observational

Registra metadata para:

```text
UI
risk
audit
```

sin bloquear Authentication.

---

## 122. Soft

Un cambio importante puede producir:

```text
STEP_UP_REQUIRED
```

---

## 123. Strict

Cambio de Device Reference:

```text
persistent authentication rejected
```

---

## 124. Cryptographic

Una futura implementación podría combinar:

```text
remember-me secret
+
device-held cryptographic proof
```

---

## 125. Browser fingerprint warning

No utilizar fingerprinting heurístico como root of trust.

---

## 126. Device deletion

Cuando un usuario selecciona:

```text
Forget this device
```

deberá revocarse:

```text
PersistentCredentialFamily
```

y opcionalmente sus sesiones asociadas.

---

## 127. Session association

Una AuthenticationSession creada por Remember-Me podrá conservar:

```text
sourcePersistentCredentialFamilyId
```

como provenance.

---

## 128. Beneficio

Si una familia se compromete:

```text
revoke sessions created from that family
```

puede ser posible.

---

## 129. Session creation

Una vez validado Remember-Me:

```text
new AuthenticationSession
```

deberá generarse con un ID completamente nuevo.

---

## 130. No reuse

Nunca:

```text
remember-me token
    =
session ID
```

---

## 131. Separate secrets

La sesión y Remember-Me deberán usar secretos independientes.

---

## 132. Session lifetime

La sesión creada desde Remember-Me podrá tener el lifetime normal del Firewall.

---

## 133. Session provenance

Debe indicar:

```text
restored_from = persistent_credential
family = ...
assurance = ...
```

---

## 134. Session fixation protection

Aunque Remember-Me sea automático:

```text
anonymous application session
        ↓
Remember-Me Authentication
        ↓
rotate session ID
```

---

## 135. Logout current session

Aquí existe una decisión importante.

`Logout` puede significar:

```text
terminate current AuthenticationSession only
```

o:

```text
terminate session + forget persistent login
```

---

## 136. Recommended browser semantics

Un logout explícito del usuario normalmente deberá:

```text
terminate current session
revoke current remember-me family
expire remember-me cookie
```

para evitar:

```text
logout
    ↓
next request
    ↓
Remember-Me logs user back in automatically
```

---

## 137. LogoutMode

Podrán existir:

```text
SESSION_ONLY
SESSION_AND_CURRENT_PERSISTENT_CREDENTIAL
ALL_SESSIONS
ALL_AUTHENTICATION_STATE
```

---

## 138. SESSION_ONLY

Útil para flows especiales, pero no debería ser necesariamente la UI estándar de "Sign out".

---

## 139. SESSION_AND_CURRENT_PERSISTENT_CREDENTIAL

Comportamiento recomendado para logout normal de navegador.

---

## 140. ALL_SESSIONS

Revoca AuthenticationSessions pero puede conservar persistent credentials si la policy lo permite.

---

## 141. ALL_AUTHENTICATION_STATE

Revoca:

```text
sessions
remember-me families
trusted device credentials
```

y posiblemente incrementa SecurityVersion.

---

## 142. Logout all devices

Normalmente:

```text
revoke all session families
revoke all persistent credential families
```

---

## 143. Current device preservation

Una acción:

```text
Sign out all other devices
```

podrá conservar:

```text
current session
current persistent family
```

si el usuario lo solicita.

---

## 144. Password change policy

Posibles políticas:

```text
KEEP_PERSISTENT_CREDENTIALS
REVOKE_OTHER_FAMILIES
REVOKE_ALL_FAMILIES
SECURITY_VERSION_INVALIDATION
```

---

## 145. Default recomendado

Para cambio de password sensible:

```text
REVOKE_OTHER_FAMILIES
```

o más estricto según perfil.

---

## 146. Password reset

Recomendación:

```text
REVOKE_ALL_FAMILIES
```

---

## 147. Compromised password

Podrá además:

```text
increment SecurityVersion
revoke all sessions
```

---

## 148. Account recovery

Después de recovery, persistent credentials anteriores deberían considerarse sospechosos salvo policy explícita.

---

## 149. Repository

Contrato conceptual:

```php
interface PersistentCredentialRepositoryInterface
{
    public function findBySelector(
        PersistentCredentialSelector $selector
    ): PersistentCredentialLookupResult;

    public function save(
        PersistentAuthenticationCredential $credential
    ): void;

    public function rotate(
        PersistentAuthenticationCredential $current,
        PersistentAuthenticationCredential $replacement
    ): PersistentCredentialRotationResult;

    public function revokeFamily(
        PersistentCredentialFamilyId $familyId,
        PersistentCredentialRevocationReason $reason
    ): void;
}
```

---

## 150. Repository independence

El Core no dependerá de:

```text
MySQL
PostgreSQL
Redis
Eloquent
Doctrine
```

---

## 151. Stores

Podrán existir:

```text
DatabasePersistentCredentialRepository
RedisPersistentCredentialRepository
HybridPersistentCredentialRepository
InMemoryPersistentCredentialRepository
```

---

## 152. Recommended durability

Remember-Me credentials son de larga duración.

Por ello un store durable como:

```text
SQL database
durable distributed KV
```

será normalmente preferible.

---

## 153. Redis-only persistence

Será posible si la infraestructura garantiza la durabilidad requerida.

---

## 154. Database schema conceptual

```text
persistent_auth_credentials

id
family_id
selector
secret_digest
identity_provider
identity_id
firewall
tenant_id
status
generation
security_version
origin_method
origin_assurance
device_id
issued_at
last_used_at
expires_at
rotated_at
revoked_at
revocation_reason
version
```

---

## 155. Family table

```text
persistent_auth_credential_families

id
identity_provider
identity_id
firewall
tenant_id
device_id
status
created_at
last_used_at
expires_at
compromised_at
revoked_at
version
```

---

## 156. Indexes

Especialmente:

```text
selector UNIQUE
family_id
identity reference
expires_at
status
device_id
```

---

## 157. Raw secret forbidden

La tabla nunca contendrá:

```text
secret
raw_token
cookie_value
```

---

## 158. Selector exposure

Aunque el selector no sea suficiente para autenticarse, no deberá exponerse innecesariamente.

---

## 159. Token logs

Nunca registrar:

```text
selector.secret
```

---

## 160. Safe logging

Podrá registrarse:

```text
credential public reference
family public reference
token fingerprint
```

---

## 161. Public credential identifier

Para UI/API podrá existir:

```text
PersistentCredentialPublicId
```

distinto del selector/token.

---

## 162. User session/device UI

Podrá mostrar:

```text
Chrome on Windows
Last used: today
Remembered: August 1
Location: approximate
Current device
```

sin mostrar secretos.

---

## 163. Max persistent devices

Policy:

```text
max_families_per_identity
```

---

## 164. Family limit actions

Al superar el límite:

```text
reject new family
revoke oldest
ask user
```

---

## 165. Default

No imponer arbitrariamente single-device persistent login.

---

## 166. Family limit concurrency

Dos logins simultáneos deberán respetar la policy mediante operaciones suficientemente atómicas.

---

## 167. Revocation

Razones:

```text
LOGOUT
USER_FORGOT_DEVICE
PASSWORD_CHANGED
PASSWORD_RESET
ACCOUNT_RECOVERY
IDENTITY_DISABLED
ADMIN_REVOKED
SECURITY_VERSION_CHANGED
TOKEN_REPLAY
DEVICE_COMPROMISED
FAMILY_LIMIT
EXPIRED
```

---

## 168. RevocationReason

Debe ser un Value Object/enum estructurado.

---

## 169. Family revocation

Revocar una familia implica:

```text
all credentials in family invalid
```

---

## 170. Identity-wide revocation

```text
all persistent families
for IdentityReference
```

---

## 171. Tenant-wide revocation

Puede requerirse:

```text
revoke credentials
for identity + tenant
```

---

## 172. Firewall-wide revocation

Puede limitarse a:

```text
web persistent credentials
```

sin afectar otro realm.

---

## 173. SecurityVersion vs explicit revocation

Ambos son útiles.

```text
explicit revocation
    → precise lifecycle/audit

SecurityVersion
    → broad invalidation primitive
```

---

## 174. Defense in depth

Una operación crítica puede hacer ambas:

```text
revoke families
SecurityVersion++
```

---

## 175. Revocation consistency

En arquitectura distribuida deberá documentarse cuánto tarda una revocación en ser efectiva.

---

## 176. High-security profile

Debe favorecer:

```text
strong consistency
```

---

## 177. Persistent credential cache

Puede existir cache de metadata.

Pero nunca deberá permitir que una credencial revocada siga autenticando durante largos períodos.

---

## 178. Request memoization

Una vez validada/rechazada durante una operación:

```text
memoize within request
```

es seguro.

---

## 179. No process-global current credential

Especialmente con FrankenPHP.

---

## 180. Persistent runtime rule

El repository puede ser compartido.

El credential actual no.

---

## 181. Authentication operation scope

Deberá contener temporalmente:

```text
parsed token
lookup result
credential evidence
rotation state
```

y destruirlo al terminar.

---

## 182. Secret lifetime in memory

El raw secret deberá mantenerse únicamente durante el tiempo mínimo necesario.

---

## 183. Secret redaction

Exceptions, dumps y traces deberán ocultarlo.

---

## 184. Cookie parser exception

Nunca:

```text
Invalid token: rm_selector.raw_secret
```

---

## 185. Safe exception

```text
Persistent authentication credential is malformed.
```

---

## 186. Constant-time verification

El digest del secret deberá compararse mediante operación constant-time apropiada.

---

## 187. Unknown selector timing

No debe introducirse una diferencia extremadamente obvia entre:

```text
selector exists
selector absent
```

si puede evitarse razonablemente.

---

## 188. Enumeration resistance

Respuesta externa:

```text
persistent authentication failed
```

sin revelar:

```text
token exists
user exists
family revoked
```

---

## 189. Internal failure classification

Sí deberá distinguir:

```text
NOT_FOUND
SECRET_MISMATCH
EXPIRED
REVOKED
ROTATED_REPLAY
SECURITY_VERSION_MISMATCH
TENANT_MISMATCH
FIREWALL_MISMATCH
IDENTITY_INELIGIBLE
DEVICE_MISMATCH
```

---

## 190. Validation pipeline

```text
Parse Token
   ↓
Lookup Selector
   ↓
Validate Record Status
   ↓
Validate Expiration
   ↓
Verify Secret
   ↓
Validate Firewall
   ↓
Validate Tenant
   ↓
Resolve Identity
   ↓
Validate SecurityVersion
   ↓
Validate Eligibility
   ↓
Evaluate Device/Risk
   ↓
Create Evidence
```

---

## 191. Ordering

Checks que no requieren Identity query podrán realizarse primero cuando no generen side channels problemáticos.

---

## 192. Secret verification before expensive Identity resolution

Reduce trabajo innecesario sobre tokens inválidos.

---

## 193. Risk integration

Persistent login puede generar señales:

```text
new network
new geography
device mismatch
long inactivity
unusual hour
replayed token
```

---

## 194. Risk does not equal Authentication

El Risk Engine no deberá validar el secret.

Solo influir en la decisión posterior.

---

## 195. Risk outcomes

```text
ALLOW
STEP_UP
REAUTHENTICATE
DENY
REVOKE
```

---

## 196. High-risk persistent login

Puede producir:

```text
valid token
+
high risk
=
fresh primary authentication required
```

---

## 197. Do not destroy valid family unnecessarily

Una simple señal de riesgo incierta puede exigir challenge sin marcar automáticamente compromise.

---

## 198. Replay is stronger signal

Un token ya rotado reutilizado sí representa evidencia considerablemente más fuerte.

---

## 199. Authentication Context

Después de éxito:

```text
Identity
AuthenticationMethod = REMEMBER_ME
Evidence = PersistentCredentialEvidence
Assurance = policy-derived
Freshness = persistent
Provenance = family/generation
```

---

## 200. Passport integration

El Authenticator podrá crear un:

```text
PersistentCredentialPassport
```

---

## 201. Passport

Conceptualmente:

```php
final readonly class PersistentCredentialPassport
{
    public function __construct(
        public IdentityReference $identity,
        public PersistentCredentialEvidence $evidence,
        public AuthenticationAssurance $assurance,
        public AuthenticationProvenance $provenance,
    ) {}
}
```

---

## 202. Passport contains no raw secret

Una vez verificado, el secret debe desaparecer del dominio posterior.

---

## 203. Credential verification result

```text
VALID
INVALID
EXPIRED
REVOKED
REPLAYED
COMPROMISED
STALE
ERROR
```

---

## 204. INVALID vs ERROR

Mantener separación:

```text
INVALID
    bad credential

ERROR
    infrastructure/system failure
```

---

## 205. Store unavailable

Nunca:

```text
store unavailable
    ↓
trust cookie anyway
```

---

## 206. Fail closed

Sin poder validar:

```text
no persistent authentication
```

---

## 207. Session creation failure

Si Remember-Me valida pero no puede crearse AuthenticationSession:

```text
do not leave request authenticated
```

si el Firewall requiere sesión.

---

## 208. Rotation failure

Debe definirse coordinación entre:

```text
token rotation
session creation
cookie issuance
```

---

## 209. Persistence coordinator

Podrá existir:

```text
PersistentAuthenticationCoordinator
```

---

## 210. Commit sequence

Recomendación conceptual:

```text
credential verified
        ↓
Identity commit-time validation
        ↓
prepare replacement credential
        ↓
atomic credential rotation
        ↓
create AuthenticationSession
        ↓
activate AuthenticationContext
        ↓
issue new session cookie
        ↓
issue replacement remember-me cookie
```

---

## 211. Partial failure complexity

Si rotation se completa pero session creation falla:

```text
old token invalid
new token may need delivery
```

El sistema deberá tener estrategia de compensación/retry.

---

## 212. Transaction boundaries

Cuando credential y session stores sean distintos no siempre existirá una transacción ACID global.

---

## 213. Saga-style coordination

Podrá utilizar:

```text
prepared rotation
commit marker
idempotency key
short-lived transition state
```

en implementaciones avanzadas.

---

## 214. V1 simplification

Para V1 puede favorecerse:

```text
durable persistent credential store
+
well-defined idempotent rotation
+
session creation retry-safe semantics
```

sin construir distributed transactions completas.

---

## 215. Cookie delivery failure

Si Token B se genera pero el navegador no recibe la response:

```text
client still has Token A
```

que ya podría estar ROTATED.

---

## 216. Critical rotation problem

El sistema deberá contemplar esta situación para no bloquear usuarios legítimos por fallos de red.

---

## 217. Rotation recovery window

Podrá existir una estrategia controlada donde el token anterior conserve una referencia al replacement durante una ventana breve.

---

## 218. Recovery semantics

```text
Token A reused shortly after rotation
+
same expected context
+
replacement still valid
        ↓
reissue Token B
```

sin crear una nueva rama.

---

## 219. Replay after recovery window

Debe tratarse como replay real.

---

## 220. Recovery window configurable

Debe ser:

```text
short
bounded
security-profile dependent
```

---

## 221. RotationRecoveryPolicy

```text
NONE
SHORT_GRACE
CONTEXT_BOUND_GRACE
```

---

## 222. High-security systems

Podrán elegir:

```text
NONE
```

y aceptar mayor fricción.

---

## 223. Persistent token theft

Amenazas:

```text
malware
XSS where cookie protections are bypassed
browser profile theft
database leak
logs
proxy leak
backup leak
physical device compromise
```

---

## 224. Defensas

```text
HttpOnly
Secure
SameSite
TLS
token hashing
rotation
replay detection
device/risk signals
expiration
revocation
SecurityVersion
minimal cookie scope
```

---

## 225. XSS

`HttpOnly` dificulta lectura directa del token mediante JavaScript.

No elimina el impacto general de XSS.

---

## 226. CSRF

Remember-Me termina creando session cookie authentication.

Las requests state-changing siguen necesitando CSRF protection según el modelo HTTP.

---

## 227. Persistent credential cookie itself

No deberá utilizarse como autorización directa de cada request state-changing si puede evitarse.

Preferir:

```text
remember-me
    ↓
bootstrap AuthenticationSession
    ↓
normal session authentication
```

---

## 228. SPA integration

Una SPA no necesita leer Remember-Me cookie.

---

## 229. SPA bootstrap

```text
SPA loads
   ↓
request backend
   ↓
no session
   ↓
Remember-Me automatically validated
   ↓
new session
   ↓
frontend receives authenticated state
```

---

## 230. Frontend response

Puede indicar:

```text
authenticated = true
authentication_source = persistent
fresh_authentication_required = false/true
```

sin exponer secretos.

---

## 231. API clients

Remember-Me está principalmente orientado a browser/session authentication.

API authentication deberá utilizar mecanismos apropiados:

```text
API tokens
OAuth access tokens
mTLS
service credentials
```

---

## 232. Mobile native apps

Podrán reutilizar conceptos de persistent credentials, pero probablemente mediante un perfil distinto.

---

## 233. Browser profile

```text
RememberMeBrowserPolicy
```

---

## 234. Native device profile

Futuro:

```text
PersistentDeviceCredentialPolicy
```

---

## 235. No semantic overloading

No forzar que todos los long-lived credentials se llamen `RememberMeToken`.

---

## 236. Authentication events

Eventos:

```text
PersistentCredentialIssued
PersistentCredentialUsed
PersistentCredentialRotated
PersistentCredentialExpired
PersistentCredentialRevoked
PersistentCredentialReplayDetected
PersistentCredentialFamilyCompromised
PersistentCredentialAuthenticationSucceeded
PersistentCredentialAuthenticationFailed
```

---

## 237. High-volume event caution

`PersistentCredentialUsed` ocurre mucho menos que Session restoration porque solo se utiliza al reconstruir sesión.

Por ello puede ser razonablemente auditable.

---

## 238. Audit record

Podrá contener:

```text
identity reference
family public reference
device reference
firewall
tenant
result
reason
timestamp
actor
```

---

## 239. Never audit raw token

Ni selector+secret completo.

---

## 240. Observability spans

```text
auth.remember.detect
auth.remember.lookup
auth.remember.verify
auth.remember.rotate
auth.remember.restore
auth.remember.revoke
auth.remember.replay
```

---

## 241. Metrics

```text
auth_persistent_credential_issued_total
auth_persistent_credential_used_total
auth_persistent_credential_rotated_total
auth_persistent_credential_revoked_total
auth_persistent_credential_replay_total
auth_persistent_credential_failed_total
auth_persistent_credential_store_latency
```

---

## 242. Safe metric labels

```text
firewall
result
reason
store
credential_type
```

No:

```text
identity_id
family_id
selector
```

---

## 243. Security alerting

Alertas posibles:

```text
replay detected
multiple compromised families
mass invalid token attempts
cross-tenant credential use
repeated revoked-token usage
```

---

## 244. Garbage collection

Expired credentials deberán eliminarse mediante:

```text
TTL
scheduled pruning
database cleanup
partition expiration
```

---

## 245. Expiration validation independent of pruning

Un registro físicamente presente pero expirado nunca será válido.

---

## 246. PersistentCredentialPruner

Podrá existir:

```text
PersistentCredentialPruner
```

---

## 247. Retention after revocation

Para security/audit podría conservarse metadata durante cierto periodo.

Pero deberá eliminarse:

```text
secret digest
```

cuando ya no sea necesario según policy.

---

## 248. Tombstones

Para replay detection puede ser útil conservar temporalmente un:

```text
rotated/revoked token tombstone
```

---

## 249. Tombstone data

Mínimo:

```text
selector/fingerprint
family reference
status
rotation timestamp
```

sin información innecesaria.

---

## 250. Tombstone retention

Debe ser limitada.

---

## 251. Privacy

Remember-Me device tracking deberá respetar:

```text
data minimization
retention
tenant boundaries
user visibility
```

---

## 252. Approximate location

Si se muestra ubicación de dispositivo:

```text
approximate
```

deberá comunicarse como tal.

---

## 253. User controls

VoltStack deberá facilitar:

```text
view remembered devices
forget one device
forget all devices
logout all sessions
```

---

## 254. Administrative controls

Administradores podrán revocar persistent credentials si Authorization lo permite.

---

## 255. Authentication vs Authorization

El Auth subsystem proporciona la operación.

El Authorization subsystem decide:

```text
who may revoke whose credentials
```

---

## 256. Account deletion

Debe revocar/eliminar:

```text
persistent credential families
```

---

## 257. Identity merge

Si se soporta Identity merge, las credenciales persistentes no deberán migrarse automáticamente sin una política explícita.

---

## 258. Identity provider migration

Puede requerir reissue o revocation.

---

## 259. Credential schema versioning

El token/record deberá poder evolucionar.

---

## 260. Token format version

Ejemplo:

```text
rm1_selector.secret
```

---

## 261. Parser version awareness

Permite futuras migraciones.

---

## 262. Unknown token version

Debe rechazarse.

---

## 263. Cryptographic agility

El digest algorithm deberá poder evolucionar.

---

## 264. DigestAlgorithmIdentifier

El registro podrá almacenar:

```text
digest_algorithm
```

si existen múltiples versiones.

---

## 265. Secret digest migration

Al rotar un token antiguo:

```text
old algorithm
    ↓
new token
    ↓
current algorithm
```

permite migración progresiva.

---

## 266. Keyed token digest

VoltStack podrá soportar:

```text
HMAC(server_pepper, token_secret)
```

como estrategia de defensa adicional.

---

## 267. Pepper management

Si se utiliza:

```text
external secret management
rotation
key identifiers
```

deberán formar parte del diseño.

---

## 268. Pepper loss

No deberá dejar al sistema incapaz de revocar credenciales.

Puede invalidar capacidad de verificarlas, lo cual debe fallar cerrado.

---

## 269. Key rotation

Si hay HMAC key rotation:

```text
key_id
```

podrá asociarse al digest.

---

## 270. Security profile abstraction

Podrán existir:

```text
STANDARD
ELEVATED
HIGH_SECURITY
```

---

## 271. STANDARD

```text
persistent login enabled
rotation
family replay detection
moderate lifetime
```

---

## 272. ELEVATED

```text
shorter lifetime
device soft binding
fresh auth for sensitive operations
stronger replay response
```

---

## 273. HIGH_SECURITY

Puede:

```text
disable remember-me
```

completamente.

---

## 274. Firewall example

```php
'web' => [
    'remember_me' => [
        'enabled' => true,
        'family_lifetime' => '30 days',
        'rotate_on_use' => true,
        'replay_detection' => 'revoke_family',
    ],
],

'admin' => [
    'remember_me' => [
        'enabled' => false,
    ],
],
```

---

## 275. Tenant override

Un tenant empresarial podrá definir:

```text
remember_me = disabled
```

si la política organizacional lo exige.

---

## 276. Security floor

Un tenant no deberá poder habilitar persistent login en un Firewall donde el framework/application lo haya marcado:

```text
FORBIDDEN
```

---

## 277. Configuration resolution

```text
Framework Security Floor
        ↓
Application Policy
        ↓
Firewall Policy
        ↓
Tenant Policy
        ↓
Identity Policy
        ↓
Effective RememberMePolicy
```

---

## 278. Most restrictive wins

Para restricciones críticas, la composición deberá tender a:

```text
most restrictive applicable rule
```

---

## 279. Example

```text
Framework: allowed
Application: allowed
Firewall: allowed
Tenant: disabled
```

Resultado:

```text
disabled
```

---

## 280. Persistent authentication manager

Podrá existir:

```text
PersistentAuthenticationManager
```

para orquestar:

```text
issuance
validation
rotation
revocation
family management
```

---

## 281. No God Object

El Manager no deberá implementar directamente:

```text
crypto
repository
cookie serialization
identity loading
risk evaluation
```

---

## 282. Delegation

```text
PersistentAuthenticationManager
 ├── TokenParser
 ├── CredentialRepository
 ├── SecretVerifier
 ├── IdentityResolver
 ├── CredentialValidator
 ├── RotationManager
 ├── FamilyManager
 ├── CookieManager
 └── EventDispatcher
```

---

## 283. Suggested namespaces

```text
VoltStack\Quantum\Auth\Persistent
VoltStack\Quantum\Auth\Persistent\Contracts
VoltStack\Quantum\Auth\Persistent\Credential
VoltStack\Quantum\Auth\Persistent\Family
VoltStack\Quantum\Auth\Persistent\Rotation
VoltStack\Quantum\Auth\Persistent\Cookie
VoltStack\Quantum\Auth\Persistent\Policy
VoltStack\Quantum\Auth\Persistent\Store
VoltStack\Quantum\Auth\Persistent\Events
```

---

## 284. Suggested structure

```text
src/Quantum/Auth/Persistent/
├── Contracts/
│   ├── PersistentCredentialRepositoryInterface.php
│   ├── PersistentCredentialIssuerInterface.php
│   ├── PersistentCredentialTokenGeneratorInterface.php
│   └── PersistentCredentialSecretHasherInterface.php
│
├── Credential/
│   ├── PersistentAuthenticationCredential.php
│   ├── PersistentCredentialId.php
│   ├── PersistentCredentialPublicId.php
│   ├── PersistentCredentialSelector.php
│   ├── PersistentCredentialStatus.php
│   ├── PersistentCredentialEvidence.php
│   └── PersistentCredentialToken.php
│
├── Family/
│   ├── PersistentCredentialFamily.php
│   ├── PersistentCredentialFamilyId.php
│   ├── PersistentCredentialFamilyStatus.php
│   └── PersistentCredentialFamilyManager.php
│
├── Rotation/
│   ├── PersistentCredentialRotator.php
│   ├── PersistentCredentialRotationResult.php
│   ├── PersistentCredentialReplayDetector.php
│   └── RotationRecoveryPolicy.php
│
├── Cookie/
│   ├── RememberMeCookieManager.php
│   ├── RememberMeCookiePolicy.php
│   └── PersistentCredentialTokenParser.php
│
├── Policy/
│   ├── RememberMePolicy.php
│   ├── RememberMePolicyResolver.php
│   ├── ReplayDetectionPolicy.php
│   └── PersistentCredentialIssuancePolicy.php
│
├── Store/
│   ├── DatabasePersistentCredentialRepository.php
│   ├── RedisPersistentCredentialRepository.php
│   └── InMemoryPersistentCredentialRepository.php
│
├── PersistentAuthenticationManager.php
├── PersistentAuthenticationCoordinator.php
├── RememberMeAuthenticator.php
└── PersistentCredentialPruner.php
```

---

## 285. Facade/API conceptual

```php
Auth::remember()->enabled();

Auth::remember()->devices();

Auth::remember()->forget($device);

Auth::remember()->forgetCurrent();

Auth::remember()->forgetAll();
```

---

## 286. Login API conceptual

```php
$result = Auth::attempt(
    credentials: [
        'email' => $email,
        'password' => $password,
    ],
    remember: true,
);
```

---

## 287. More explicit API

Internamente será preferible:

```php
AuthenticationOptions::create()
    ->withPersistentLogin();
```

para evitar booleans ambiguos.

---

## 288. Issuance flow

```text
User submits login
        ↓
Primary Authenticator
        ↓
Authentication succeeds
        ↓
RememberMeIntent = REQUESTED
        ↓
RememberMePolicyResolver
        ↓
Issuance allowed
        ↓
Create Family
        ↓
Create Token Generation 1
        ↓
Store digest
        ↓
Send persistent cookie
```

---

## 289. Restoration flow

```text
Request
  ↓
Session unavailable
  ↓
Remember-Me cookie
  ↓
Parse
  ↓
Lookup
  ↓
Verify
  ↓
Identity Security Validation
  ↓
Persistent Authentication
  ↓
Rotate Token
  ↓
Create AuthenticationSession
  ↓
Set new Session cookie
  ↓
Set new Remember-Me cookie
```

---

## 290. Replay flow

```text
Token 8 already rotated
        ↓
Token 8 appears again
        ↓
ReplayDetector
        ↓
Family COMPROMISED
        ↓
Token 9 revoked
        ↓
Sessions from Family revoked
        ↓
Fresh Authentication required
        ↓
Security event
```

---

## 291. Logout flow

```text
Current session
     +
Current persistent family
        ↓
Logout
        ↓
Session TERMINATED
        ↓
Persistent family REVOKED
        ↓
session cookie expired
        ↓
remember-me cookie expired
```

---

## 292. Password reset flow

```text
Password Reset succeeds
        ↓
CredentialVersion++
        ↓
SecurityVersion++
        ↓
revoke PersistentCredentialFamilies
        ↓
revoke AuthenticationSessions
        ↓
fresh login required
```

La política exacta podrá configurarse, pero este será un perfil seguro recomendado.

---

## 293. Account disablement flow

```text
Identity ACTIVE
    ↓
DISABLED
    ↓
SecurityVersion++
    ↓
existing sessions fail restoration
persistent credentials fail authentication
```

---

## 294. Re-enable account

No deberá reactivar automáticamente familias previamente revocadas.

---

## 295. Testing — token generation

Verificar:

```text
entropy
uniqueness
format
length
parser behavior
```

---

## 296. Testing — hashing

```text
raw token never stored
correct secret validates
wrong secret fails
comparison is safe
```

---

## 297. Testing — issuance

```text
remember requested
remember not requested
policy disabled
insufficient assurance
tenant disabled
admin firewall disabled
```

---

## 298. Testing — restoration

```text
valid token
unknown selector
bad secret
expired token
expired family
revoked token
revoked family
SecurityVersion mismatch
Identity disabled
Identity deleted
tenant mismatch
firewall mismatch
```

---

## 299. Testing — rotation

```text
Token 1 → Token 2
Token 1 becomes ROTATED
Token 2 becomes ACTIVE
cookie receives Token 2
```

---

## 300. Testing — replay

```text
use Token 1
rotate to Token 2
reuse Token 1
detect replay
revoke family
reject Token 2
```

---

## 301. Testing — concurrent requests

```text
two requests use Token 1 simultaneously
```

Debe verificarse que:

```text
no uncontrolled branching
no accidental multiple families
no false compromise beyond configured policy
```

---

## 302. Testing — lost response

```text
Token 1
    ↓
server rotates to Token 2
    ↓
response lost
    ↓
client retries Token 1
```

Probar `RotationRecoveryPolicy`.

---

## 303. Testing — logout

```text
logout
session invalid
remember-me family invalid
cookies expired
next request does not auto-login
```

---

## 304. Testing — logout all

```text
multiple sessions
multiple persistent families
        ↓
logout all
        ↓
all invalid
```

---

## 305. Testing — password reset

Todas las familias anteriores deberán quedar inválidas según policy.

---

## 306. Testing — persistent runtime

Requests consecutivas:

```text
Alice remember-me
Bob remember-me
Anonymous
```

nunca deberán compartir state.

---

## 307. Testing — fuzzing

Especialmente:

```text
token parser
cookie parser
selector parsing
version parsing
oversized tokens
malformed encoding
```

---

## 308. Testing — property-based

Útil para:

```text
generation monotonicity
family state transitions
rotation invariants
revocation idempotency
expiration boundaries
```

---

## 309. Credential state machine

```text
          ┌─────────────┐
          │   ACTIVE    │
          └──────┬──────┘
                 │
       ┌─────────┼─────────┬─────────────┐
       ▼         ▼         ▼             ▼
    ROTATED   EXPIRED   REVOKED     COMPROMISED
       │
       ▼
 replacement ACTIVE
```

---

## 310. Family state machine

```text
ACTIVE
 ├──► EXPIRED
 ├──► REVOKED
 └──► COMPROMISED
```

No existe transición de vuelta a `ACTIVE`.

---

## 311. Invariant — Credential

### AUTH-PERSIST-01

Persistent credential is not an AuthenticationSession.

#### AUTH-PERSIST-02

Persistent credential secrets are cryptographically random.

#### AUTH-PERSIST-03

Raw persistent secrets are never stored server-side.

#### AUTH-PERSIST-04

Raw persistent secrets never enter logs, events or metrics.

#### AUTH-PERSIST-05

Persistent credentials are server-verifiable and revocable.

#### AUTH-PERSIST-06

Expired or revoked credentials never authenticate.

#### AUTH-PERSIST-07

Credential possession does not bypass Identity Security State.

#### AUTH-PERSIST-08

SecurityVersion mismatch prevents persistent Authentication.

---

## 312. Invariant — Rotation

### AUTH-PERSIST-ROT-01

Successful token use can rotate the credential.

#### AUTH-PERSIST-ROT-02

Rotation produces a new independent secret.

#### AUTH-PERSIST-ROT-03

Rotated tokens cannot normally authenticate again.

#### AUTH-PERSIST-ROT-04

Replay detection operates at family/lineage level.

#### AUTH-PERSIST-ROT-05

Family lifetime does not silently reset on rotation.

#### AUTH-PERSIST-ROT-06

Concurrent rotation cannot create uncontrolled token branches.

#### AUTH-PERSIST-ROT-07

Rotation recovery windows are short and bounded.

---

## 313. Invariant — Session bootstrap

### AUTH-PERSIST-SESS-01

Remember-Me creates a new AuthenticationSession.

#### AUTH-PERSIST-SESS-02

Remember-Me token and Session ID are independent secrets.

#### AUTH-PERSIST-SESS-03

Session fixation protection applies to persistent Authentication.

#### AUTH-PERSIST-SESS-04

Session provenance records persistent Authentication origin.

#### AUTH-PERSIST-SESS-05

Remember-Me assurance is derived by policy and not blindly inherited.

---

## 314. Invariant — Logout

### AUTH-PERSIST-LOGOUT-01

Explicit standard logout prevents immediate automatic re-login.

#### AUTH-PERSIST-LOGOUT-02

Current persistent family can be revoked independently.

#### AUTH-PERSIST-LOGOUT-03

Identity-wide revocation can invalidate all persistent families.

#### AUTH-PERSIST-LOGOUT-04

Deleting a browser cookie alone is not sufficient server-side revocation.

---

## 315. Invariant — Tenant/Firewall

### AUTH-PERSIST-SCOPE-01

Persistent credentials are Firewall/Realm aware.

#### AUTH-PERSIST-SCOPE-02

Tenant-bound credentials cannot cross tenants implicitly.

#### AUTH-PERSIST-SCOPE-03

Tenant switching requires explicit re-evaluation.

#### AUTH-PERSIST-SCOPE-04

Shared realms cannot bypass stronger assurance requirements.

---

## 316. Invariant — Runtime

### AUTH-PERSIST-RT-01

Raw credential state is operation-scoped.

#### AUTH-PERSIST-RT-02

Persistent runtime workers never retain the current credential between requests.

#### AUTH-PERSIST-RT-03

Shared services remain stateless.

#### AUTH-PERSIST-RT-04

Fiber/coroutine requests have isolated authentication state.

---

## 317. Anti-pattern — permanent session

Evitar implementar Remember-Me como:

```text
session lifetime = 1 year
```

---

## 318. Anti-pattern — plaintext token DB

Nunca:

```text
remember_token = raw cookie secret
```

---

## 319. Anti-pattern — static remember token

Evitar un único token permanente que nunca rota.

---

## 320. Anti-pattern — token equals user ID

Nunca.

---

## 321. Anti-pattern — email + signature without lifecycle

Aunque una cookie firmada pueda ser criptográficamente válida, sin:

```text
revocation
versioning
rotation
expiry
```

puede tener lifecycle insuficiente para los objetivos de VoltStack.

---

## 322. Anti-pattern — unlimited rotation lifetime

No permitir que cada rotación reinicie automáticamente meses de confianza.

---

## 323. Anti-pattern — preserve AAL2 forever

Remember-Me de semanas de antigüedad no deberá equivaler automáticamente a MFA fresco.

---

## 324. Anti-pattern — use token every request

Una vez creada AuthenticationSession, usar la sesión.

---

## 325. Anti-pattern — logout leaves token active

Provocaría auto-login inmediato.

---

## 326. Anti-pattern — process-global token

Crítico bajo FrankenPHP.

---

## 327. Anti-pattern — revoke cookie only

La credencial server-side debe quedar revocada.

---

## 328. Anti-pattern — hard browser fingerprint dependency

No deberá ser requisito base.

---

## 329. Anti-pattern — client-controlled expiration

Nunca confiar en la expiración del navegador como fuente autoritativa.

---

## 330. Anti-pattern — recursive issuance

Una Authentication Remember-Me no crea familias nuevas indefinidamente.

---

## 331. Anti-pattern — no replay handling

Rotar sin detectar reutilización desaprovecha una de las principales ventajas de token families.

---

## 332. Anti-pattern — cross-tenant Remember-Me

Nunca restaurar tenant distinto implícitamente.

---

## 333. Comparación conceptual con Laravel

Laravel proporciona una experiencia muy simple mediante:

```php
Auth::attempt($credentials, true);
```

y tradicionalmente utiliza un `remember_token` asociado al usuario.

VoltStack conservará la ergonomía:

```text
simple persistent login opt-in
automatic restoration
guard/firewall integration
```

pero separará el mecanismo en entidades explícitas:

```text
PersistentCredential
PersistentCredentialFamily
PersistentCredentialRepository
Rotation
ReplayDetection
SecurityVersion
DeviceBinding
AuthenticationSession bootstrap
```

---

## 334. Mejora frente al modelo simple de `remember_token`

En vez de:

```text
User
 └── remember_token
```

VoltStack utilizará:

```text
Identity
 ├── Persistent Family Laptop
 │      └── rotating credentials
 │
 ├── Persistent Family Phone
 │      └── rotating credentials
 │
 └── Persistent Family Tablet
        └── rotating credentials
```

Esto permite:

```text
multiple devices
selective revocation
rotation
replay detection
auditing
```

---

## 335. Comparación conceptual con Symfony

Symfony ofrece una arquitectura más estructurada alrededor de:

```text
firewalls
remember-me configuration
remember-me handlers
persistent token providers
signatures
```

VoltStack conservará:

```text
Firewall awareness
Authenticator integration
configurable handlers/providers
persistent-token architecture
```

pero llevará el concepto hacia:

```text
credential families
SecurityVersion
assurance
freshness
tenant isolation
device management
rotation lineage
replay detection
persistent-runtime safety
```

---

## 336. Filosofía VoltStack

La meta no será copiar:

```text
Laravel remember_token
```

ni:

```text
Symfony RememberMe
```

sino utilizar las mejores ideas de ambos como punto de partida.

La abstracción final será:

```text
Long-Lived Authentication Credential System
```

del cual Remember-Me será únicamente una capacidad.

---

## 337. Configuración conceptual

```php
return [

    'authentication' => [

        'persistent_credentials' => [

            'remember_me' => [

                'enabled' => true,

                'family_lifetime' => '30 days',

                'idle_lifetime' => '14 days',

                'rotate_on_use' => true,

                'replay_detection' => 'revoke_family',

                'max_families' => 10,

                'security_version_validation' => true,

                'cookie' => [
                    'name' => '__Host-voltstack_remember',
                    'http_only' => true,
                    'secure' => true,
                    'same_site' => 'lax',
                    'path' => '/',
                ],

            ],

        ],

    ],

];
```

Los valores son únicamente ilustrativos.

---

## 338. Arquitectura final

```text
┌──────────────────────────────────────────────────────────────┐
│                     PRIMARY AUTHENTICATION                   │
│                                                              │
│ Password / Passkey / MFA / Federation                        │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
                    AuthenticationContext
                               │
             ┌─────────────────┴──────────────────┐
             │                                    │
             ▼                                    ▼
 AuthenticationSession              PersistentCredentialFamily
                                               │
                                               ▼
                                     PersistentCredential
                                               │
                                      long-lived cookie
                                               │
                                               │ later
                                               ▼
                                      RememberMeAuthenticator
                                               │
                                               ▼
                                     Credential Verification
                                               │
                                               ▼
                                      Identity Resolution
                                               │
                                               ▼
                                  Identity Security Validation
                                               │
                                               ▼
                                    Assurance/Freshness Policy
                                               │
                                               ▼
                                      Token Rotation
                                               │
                                               ▼
                                    AuthenticationContext
                                               │
                                               ▼
                                  NEW AuthenticationSession
```

---

## 339. Regla arquitectónica final

VoltStack deberá preservar:

```text
PRIMARY AUTHENTICATION
        ↓
EXPLICIT USER/POLICY CONSENT
        ↓
PERSISTENT CREDENTIAL FAMILY
        ↓
HIGH-ENTROPY OPAQUE TOKEN
        ↓
HASHED SERVER-SIDE REPRESENTATION
        ↓
LONG-LIVED CLIENT COOKIE
        ↓
SESSION DISAPPEARS
        ↓
PERSISTENT CREDENTIAL PRESENTED
        ↓
TOKEN VERIFICATION
        ↓
IDENTITY SECURITY REVALIDATION
        ↓
ASSURANCE / FRESHNESS EVALUATION
        ↓
TOKEN ROTATION
        ↓
NEW AUTHENTICATION SESSION
```

La regla central será:

> **Remember-Me no prolonga una sesión existente; conserva una credencial limitada que permite demostrar nuevamente una relación autenticable con una Identity.**

Y una segunda regla será igualmente importante:

> **Una credencial persistente nunca deberá poseer más confianza que la que pueda justificarse en el momento exacto en que intenta reconstruir Authentication.**

Por ello:

```text
valid persistent token
```

no significa automáticamente:

```text
fully trusted fresh authentication
```

sino:

```text
valid persistent evidence
        +
current Identity Security State
        +
current Firewall Policy
        +
current Tenant
        +
current Device/Risk context
        ↓
effective Authentication
```

---

## 340. Criterios de aceptación

El subsistema será considerado completo cuando:

1. `PersistentAuthenticationCredential` sea independiente de Session;
2. soporte tokens opacos de alta entropía;
3. almacene únicamente digest de secrets;
4. soporte selector + secret;
5. soporte token families;
6. soporte generaciones;
7. soporte rotation;
8. soporte single-use semantics;
9. detecte replay;
10. gestione concurrencia legítima;
11. soporte rotation recovery;
12. tenga absolute family lifetime;
13. soporte idle expiration;
14. permita multiple devices;
15. permita revocación por device;
16. permita revocación por Identity;
17. permita revocación por tenant;
18. soporte Firewall binding;
19. soporte Tenant binding;
20. valide SecurityVersion;
21. pueda integrar CredentialVersion;
22. valide Identity Security State;
23. soporte account disablement;
24. soporte password change invalidation;
25. soporte password reset invalidation;
26. soporte account recovery invalidation;
27. cree nuevas AuthenticationSessions;
28. rote Session ID durante bootstrap;
29. preserve Authentication provenance;
30. modele assurance correctamente;
31. modele freshness correctamente;
32. soporte reauthentication;
33. soporte step-up;
34. soporte secure cookies;
35. no exponga secrets a frontend;
36. integre logout;
37. integre logout all;
38. soporte "forget device";
39. sea auditable;
40. sea observable;
41. soporte durable stores;
42. soporte distributed deployments;
43. sea seguro bajo FrankenPHP;
44. sea fiber/coroutine safe;
45. nunca mezcle Remember-Me con Authorization.

---

## 341. Relación con los subsistemas anteriores

Con este documento, la cadena de Authentication empieza a quedar definida de forma continua:

```text
Identity
   ↓
Credential
   ↓
Authenticator
   ↓
Passport
   ↓
Evidence
   ↓
AuthenticationContext
   ↓
AuthenticationSession
   ↓
Persistent Authentication Credential
```

Pero todavía falta una pieza fundamental para aplicaciones modernas:

```text
STATELESS AUTHENTICATION
```

Es decir:

```text
Authorization: Bearer ...
API Tokens
Personal Access Tokens
Machine Credentials
Service Tokens
Token Extraction
Token Validation
Token Revocation
Token Scopes
```

sin depender de browser sessions.

---

## 342. Próximo documento recomendado

El siguiente documento será:

```text
14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md
```

Su responsabilidad será definir la arquitectura completa de Authentication stateless de VoltStack:

```text
Bearer Authentication
API Authentication
Opaque API Tokens
Personal Access Tokens
Token Credentials
Token hashing
Token issuance
Token lookup
Token rotation
Token revocation
Token expiration
Token scopes
Token abilities
Identity binding
Tenant binding
Firewall binding
Machine identities
Service accounts
Authorization header extraction
Credential transport
Authentication provenance
SecurityVersion integration
token compromise
audit
observability
distributed validation
SPA/API boundaries
FrankenPHP safety
```

La separación arquitectónica será:

```text
Browser stateful authentication
        │
        ├── AuthenticationSession
        └── Remember-Me Credential

API/stateless authentication
        │
        └── Bearer / API Credential
```

Esto permitirá que VoltStack tenga una capa de Authentication coherente tanto para aplicaciones tradicionales y SPA como para APIs, servicios internos y arquitecturas distribuidas.
