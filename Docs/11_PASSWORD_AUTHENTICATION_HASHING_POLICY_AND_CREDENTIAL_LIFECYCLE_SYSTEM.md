# VoltStack Authentication System

## 11 — Password Authentication, Hashing Policy and Credential Lifecycle System

- **Archivo:** `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del subsistema de autenticación mediante contraseña, hashing y ciclo de vida de credenciales

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

---

## 1. Propósito

Este documento define el subsistema responsable de la autenticación basada en contraseñas dentro de VoltStack.

Su responsabilidad comprende:

```text
password presentation
password credential representation
password verification
password hashing
hash policy
algorithm agility
hash migration
rehashing
credential persistence
password creation
password change
password reset integration
credential rotation
credential revocation
credential history
compromised-password policies
credential versioning
timing resistance
brute-force interaction
audit
observability
```

La arquitectura deberá ofrecer una API sencilla para el desarrollador sin convertir la contraseña en el fundamento de todo Authentication.

---

## 2. Principio fundamental

VoltStack distinguirá:

```text
Plain Password
      ↓
PasswordCredential
      ↓
PasswordAuthenticator
      ↓
Identity Resolution
      ↓
Stored Password Credential
      ↓
PasswordVerifier
      ↓
Password Hashing Driver
      ↓
Credential Evidence
```

Por tanto:

> **La contraseña es un mecanismo de Authentication, no la Identity ni el sistema completo de Authentication.**

---

## 3. Objetivos

El subsistema deberá:

1. soportar hashing moderno;
2. utilizar algoritmos adaptativos;
3. permitir evolución criptográfica;
4. soportar migración transparente de hashes;
5. separar hashing de Authentication;
6. separar password policy de hashing policy;
7. impedir almacenamiento reversible de passwords;
8. minimizar exposición del plaintext;
9. soportar múltiples stores;
10. integrarse con Identity Security State;
11. integrarse con rate limiting;
12. mitigar timing attacks;
13. soportar password history opcional;
14. soportar detección de passwords comprometidos;
15. soportar pepper opcional;
16. ser compatible con runtimes persistentes como FrankenPHP.

---

## 4. No objetivos

Este subsistema no será responsable directamente de:

```text
MFA
Passkeys
WebAuthn
OAuth
OIDC
SAML
API tokens
session management
Authorization
account recovery completo
```

Aunque deberá integrarse con ellos.

---

## 5. Modelo conceptual

```text
Presented Password
        │
        ▼
PasswordCredential
        │
        ▼
PasswordAuthenticator
        │
        ├── Identity Resolution
        │
        ├── Eligibility Check
        │
        ▼
PasswordCredentialRepository
        │
        ▼
PasswordCredentialRecord
        │
        ▼
PasswordVerifier
        │
        ▼
PasswordHasherManager
        │
        ▼
Configured Hasher
        │
        ├── Argon2id
        ├── bcrypt
        └── Legacy/Migration Drivers
        │
        ▼
PasswordVerificationResult
        │
        ├── VALID
        ├── INVALID
        └── VALID_REHASH_REQUIRED
```

---

## 6. PasswordCredential

`PasswordCredential` representará la contraseña presentada durante una operación de Authentication.

No deberá representar el hash persistido.

Conceptualmente:

```php
final readonly class PasswordCredential
{
    public function __construct(
        private string $value,
    ) {}
}
```

---

## 7. Plaintext password lifetime

El plaintext deberá existir durante el menor tiempo práctico posible.

No deberá copiarse innecesariamente entre:

```text
Request
Passport
Events
Exceptions
Logs
Session
AuthenticationContext
```

---

## 8. PasswordCredential no serializable

La implementación deberá impedir o desalentar:

```php
serialize($passwordCredential);
```

y cualquier persistencia accidental.

---

## 9. PasswordCredential no loggable

Su representación de debugging nunca deberá mostrar el valor.

Ejemplo:

```text
PasswordCredential([REDACTED])
```

---

## 10. PasswordCredentialRecord

El password persistido deberá representarse mediante un objeto diferente:

```text
PasswordCredentialRecord
```

---

## 11. PasswordCredentialRecord model

Conceptualmente:

```php
final readonly class PasswordCredentialRecord
{
    public function __construct(
        public CredentialIdentifier $identifier,
        public IdentityReference $identity,
        public string $hash,
        public PasswordCredentialStatus $status,
        public CredentialVersion $version,
        public ?\DateTimeImmutable $createdAt = null,
        public ?\DateTimeImmutable $changedAt = null,
        public ?\DateTimeImmutable $expiresAt = null,
    ) {}
}
```

---

## 12. PasswordCredentialStatus

Estados posibles:

```text
ACTIVE
RESET_REQUIRED
EXPIRED
REVOKED
COMPROMISED
DISABLED
```

---

## 13. ACTIVE

La credential puede utilizarse normalmente.

---

## 14. RESET_REQUIRED

La contraseña sigue asociada a la Identity, pero no deberá producir Authentication normal.

Puede permitir únicamente:

```text
credential replacement flow
recovery transaction
```

---

## 15. EXPIRED

Solo deberá utilizarse cuando la política de la organización realmente requiera expiración periódica.

VoltStack no deberá imponer expiración arbitraria por defecto.

---

## 16. REVOKED

La credential ya no es válida.

---

## 17. COMPROMISED

La contraseña ha sido identificada como comprometida.

Deberá activar la política correspondiente:

```text
deny
reset
security version increment
session revocation
```

según contexto.

---

## 18. DISABLED

Permite conservar el registro pero impedir su uso.

Ejemplo:

```text
Identity migrated to federation-only authentication
```

---

## 19. CredentialIdentifier

Cada credential podrá tener identidad propia.

```text
Identity
   │
   ├── Password Credential P1
   ├── Passkey P2
   └── Recovery Credential P3
```

Esto facilita lifecycle y revocación selectiva.

---

## 20. PasswordAuthenticator

Será el `Authenticator` especializado en username/password o identity-claim/password.

Conceptualmente:

```php
final class PasswordAuthenticator implements AuthenticatorInterface
{
    // ...
}
```

---

## 21. PasswordAuthenticator responsibilities

Deberá coordinar:

```text
request support detection
claim extraction
password extraction
identity resolution
eligibility evaluation
credential lookup
password verification
evidence creation
rehash scheduling
```

No deberá implementar internamente algoritmos criptográficos.

---

## 22. PasswordAuthenticator no deberá

No deberá:

```text
call password_hash directly
query ORM directly
create sessions directly
assign roles
perform authorization
send reset emails
```

---

## 23. Request extraction

Un adapter HTTP podrá obtener:

```text
identifier
password
```

desde:

```text
HTML form
JSON request
SPA request
```

y producir estructuras internas tipadas.

---

## 24. PasswordAuthenticationRequest

Conceptualmente:

```php
final readonly class PasswordAuthenticationRequest
{
    public function __construct(
        public IdentityClaim $identity,
        public PasswordCredential $password,
    ) {}
}
```

---

## 25. Campo de identidad configurable

VoltStack no deberá asumir exclusivamente:

```text
email
```

Podrá utilizar:

```text
email
username
employee number
tenant username
custom identity claim
```

---

## 26. PasswordCredentialRepository

Responsable de obtener y persistir registros de password.

```php
interface PasswordCredentialRepositoryInterface
{
    public function findForIdentity(
        IdentityReference $identity
    ): PasswordCredentialLookupResult;
}
```

---

## 27. Separación del IdentityProvider

Se mantendrá:

```text
IdentityProvider
    resolves Identity

PasswordCredentialRepository
    resolves PasswordCredentialRecord
```

aunque una implementación ORM pueda consultar la misma tabla.

---

## 28. Modelo DB simple

Una aplicación sencilla podrá almacenar:

```text
users.password
```

mediante un adapter.

VoltStack no obligará a crear una tabla independiente.

---

## 29. Modelo DB avanzado

Una aplicación podrá utilizar:

```text
authentication_credentials
```

con:

```text
id
identity_id
type
hash
status
version
created_at
changed_at
expires_at
metadata
```

---

## 30. Storage independence

El Core no dependerá de:

```text
Eloquent
Doctrine
VoltStack ORM
LDAP
SQL
```

---

## 31. PasswordVerifier

Será responsable de verificar:

```text
presented plaintext
```

contra:

```text
stored password hash
```

---

## 32. PasswordVerifierInterface

```php
interface PasswordVerifierInterface
{
    public function verify(
        PasswordCredential $presented,
        PasswordCredentialRecord $stored,
        PasswordVerificationContext $context,
    ): PasswordVerificationResult;
}
```

---

## 33. PasswordVerificationResult

Resultados:

```text
VALID
INVALID
VALID_REHASH_REQUIRED
INVALID_CREDENTIAL_STATE
ERROR
```

---

## 34. VALID

La contraseña coincide y el hash cumple la política actual.

---

## 35. VALID_REHASH_REQUIRED

La contraseña coincide, pero:

```text
algorithm
cost
parameters
format
pepper version
```

ya no cumplen la política preferida.

---

## 36. INVALID

La contraseña no coincide.

---

## 37. INVALID_CREDENTIAL_STATE

La contraseña podría coincidir criptográficamente, pero el registro no es utilizable.

Ejemplo:

```text
REVOKED
COMPROMISED
EXPIRED
```

---

## 38. ERROR

Problema interno.

Nunca deberá convertirse en `VALID`.

---

## 39. PasswordHasherInterface

Contrato de hashing:

```php
interface PasswordHasherInterface
{
    public function hash(
        PasswordCredential $password,
        PasswordHashingContext $context,
    ): PasswordHash;

    public function verify(
        PasswordCredential $password,
        PasswordHash $hash,
        PasswordHashingContext $context,
    ): bool;

    public function needsRehash(
        PasswordHash $hash,
        PasswordHashingContext $context,
    ): bool;
}
```

---

## 40. PasswordHash

Value Object para impedir el uso indiscriminado de strings.

```php
final readonly class PasswordHash
{
    public function __construct(
        private string $value,
    ) {}
}
```

---

## 41. PasswordHash puede persistirse

A diferencia del plaintext:

```text
PasswordHash
```

sí es persistible.

Pero deberá seguir tratándose como información sensible.

---

## 42. Hashing algorithm

VoltStack deberá preferir algoritmos diseñados específicamente para passwords.

La configuración inicial deberá favorecer:

```text
Argon2id
```

cuando esté disponible y correctamente soportado por el entorno.

---

## 43. Argon2id

La policy podrá controlar:

```text
memory cost
time cost
parallelism / threads where exposed
```

según las capacidades reales de PHP/runtime.

---

## 44. bcrypt

Deberá soportarse ampliamente por:

```text
compatibility
existing Laravel applications
legacy migration
platform availability
```

---

## 45. No fast general-purpose hashes

Nunca utilizar directamente:

```text
MD5(password)
SHA1(password)
SHA256(password)
SHA512(password)
```

como password hashing moderno.

---

## 46. No manual salt

Con algoritmos modernos:

```text
Argon2id
bcrypt
```

el hasher deberá administrar el salt mediante la implementación segura correspondiente.

No generar salts caseros innecesariamente.

---

## 47. PasswordHasherManager

VoltStack tendrá un administrador de hashers.

```text
PasswordHasherManager
```

similar en ergonomía a otros managers del framework, pero con política criptográfica explícita.

---

## 48. PasswordHasherRegistry

Podrá registrar:

```text
argon2id
bcrypt
legacy
custom
```

---

## 49. Default hasher

Configuración conceptual:

```php
'passwords' => [
    'hasher' => 'argon2id',
];
```

---

## 50. Hashing Policy

La selección del driver no será suficiente.

Existirá:

```text
PasswordHashingPolicy
```

---

## 51. PasswordHashingPolicy responsibilities

Determinará:

```text
preferred algorithm
parameters
minimum accepted algorithms
deprecated algorithms
rehash requirements
pepper policy
migration policy
resource limits
```

---

## 52. Algorithm agility

VoltStack deberá asumir que:

> **El algoritmo recomendado hoy puede no ser el recomendado en el futuro.**

Por ello ningún modelo de dominio deberá depender rígidamente de Argon2id o bcrypt.

---

## 53. Hash self-description

Cuando el formato utilizado sea autocontenido, el sistema deberá aprovechar la información del hash para detectar:

```text
algorithm
parameters
version
```

sin guardar duplicados innecesarios.

---

## 54. Hash metadata

Cuando sea necesario, podrá almacenarse metadata adicional:

```text
hasher profile
pepper version
migration source
created policy version
```

pero nunca plaintext.

---

## 55. PasswordHashProfile

Podrá representar una configuración nombrada:

```text
interactive_default
high_security
legacy_compatibility
low_memory_worker
```

---

## 56. Per-environment tuning

Los parámetros no deberán copiarse ciegamente entre:

```text
development laptop
production server
serverless runtime
FrankenPHP worker
high-density container
```

---

## 57. Hash cost benchmark

VoltStack podrá proporcionar tooling para medir un target de hashing.

Ejemplo conceptual:

```text
volt auth:password-benchmark
```

---

## 58. Benchmark objetivo

La herramienta podría buscar un tiempo aproximado configurable por operación de hash.

No deberá fijarse una cifra universal dentro del Core.

---

## 59. Resource governance

El hashing de passwords consume:

```text
CPU
memory
worker capacity
```

Esto es una propiedad de seguridad intencional.

Pero debe gobernarse para evitar DoS.

---

## 60. FrankenPHP consideration

Con workers persistentes:

```text
N simultaneous password verifications
×
Argon2 memory cost
```

puede producir presión significativa de memoria.

La configuración deberá considerar concurrencia real.

---

## 61. HashingConcurrencyLimiter

VoltStack podrá ofrecer:

```text
HashingConcurrencyLimiter
```

o integrarse con el sistema de Concurrency/Resource Governance.

---

## 62. No weakening under load

Nunca cambiar automáticamente:

```text
Argon2 strong parameters
```

a:

```text
weak parameters
```

porque el servidor tenga carga.

Preferir:

```text
queue/backpressure
rate limiting
capacity control
```

---

## 63. needsRehash

Después de una verificación válida:

```text
PasswordHasher::needsRehash()
```

determinará si el hash debe actualizarse.

---

## 64. Transparent rehash

Flujo:

```text
User submits password
        ↓
old hash verifies
        ↓
needsRehash = true
        ↓
hash plaintext with current policy
        ↓
replace stored hash
```

---

## 65. Rehash only after valid verification

Nunca intentar migrar un hash cuando la contraseña no ha sido validada.

---

## 66. Rehash must not alter Authentication outcome

Si:

```text
password valid
```

pero el almacenamiento del nuevo hash falla, la policy deberá definir el resultado.

Por defecto, un fallo no crítico de rehash podrá:

```text
authenticate
+
record rehash failure
+
retry later
```

si el hash antiguo sigue siendo aceptable.

---

## 67. Mandatory migration

Si el algoritmo antiguo está clasificado como:

```text
CRITICALLY_UNSAFE
```

la policy podrá exigir:

```text
credential replacement
```

en vez de permitir continuar indefinidamente.

---

## 68. Hash migration categories

```text
CURRENT
ACCEPTED_REHASH
DEPRECATED
LEGACY
REJECTED
```

---

## 69. CURRENT

Cumple completamente la política.

---

## 70. ACCEPTED_REHASH

Es seguro para verificación pero debe actualizarse.

---

## 71. DEPRECATED

Se acepta temporalmente bajo migration policy.

---

## 72. LEGACY

Requiere adapter específico.

Ejemplo:

```text
legacy framework hash
old application hash
```

---

## 73. REJECTED

No deberá utilizarse para Authentication.

---

## 74. LegacyPasswordHasher

VoltStack podrá permitir drivers exclusivamente para migración.

```text
LegacyPasswordHasher
```

---

## 75. Legacy driver restrictions

Un driver legacy deberá poder marcarse:

```text
verify_only = true
```

para impedir crear hashes nuevos con él.

---

## 76. Laravel migration

VoltStack deberá poder facilitar migraciones desde aplicaciones Laravel que utilicen formatos soportados por PHP/Laravel, sin acoplar el Core a Laravel.

---

## 77. Symfony migration

Igualmente podrá aceptar/adaptar hashes provenientes de aplicaciones Symfony mediante perfiles de migración.

---

## 78. Migration by successful login

Una estrategia especialmente útil será:

```text
legacy application
      ↓
user logs in
      ↓
legacy hash verifies
      ↓
VoltStack current hash generated
      ↓
credential upgraded
```

---

## 79. Bulk migration limitation

Los hashes existentes no pueden transformarse criptográficamente a un nuevo algoritmo sin conocer el password.

Por tanto:

```text
old hash
    ≠
convert directly
    → Argon2id hash
```

---

## 80. Migration alternatives

Las opciones reales son:

```text
rehash after successful login
forced password reset
dual-system migration
temporary legacy verification
```

---

## 81. Password Policy

Deberá existir independientemente de `PasswordHashingPolicy`.

```text
PasswordPolicy
```

define qué passwords pueden establecerse.

---

## 82. Hashing Policy vs Password Policy

```text
PasswordHashingPolicy
    → how password is protected

PasswordPolicy
    → what password may be chosen
```

---

## 83. PasswordPolicyInterface

```php
interface PasswordPolicyInterface
{
    public function evaluate(
        PasswordCandidate $password,
        PasswordPolicyContext $context,
    ): PasswordPolicyResult;
}
```

---

## 84. PasswordCandidate

Solo deberá existir durante operaciones como:

```text
registration
password change
password reset
credential creation
```

---

## 85. PasswordPolicyResult

```text
ACCEPTED
REJECTED
```

con violations tipadas.

---

## 86. PasswordPolicyViolation

Ejemplos:

```text
TOO_SHORT
TOO_LONG
COMPROMISED
CONTAINS_IDENTITY_DATA
DISALLOWED_PATTERN
HISTORY_REUSE
CUSTOM_POLICY_FAILURE
```

---

## 87. Long passwords

VoltStack deberá soportar contraseñas/frases largas dentro de límites razonables.

---

## 88. Maximum input length

Debe existir un límite técnico para evitar DoS mediante inputs gigantes.

Ejemplo conceptual:

```text
password_max_input_bytes
```

Este límite no deberá confundirse con una recomendación de password corta.

---

## 89. Minimum length

Será configurable.

El framework deberá favorecer políticas modernas basadas principalmente en:

```text
sufficient length
compromised-password rejection
rate limiting
MFA/passkeys
```

antes que reglas arbitrarias de composición.

---

## 90. Composition rules

VoltStack podrá soportar políticas como:

```text
uppercase
lowercase
number
symbol
```

por compatibilidad empresarial.

Pero no deberán constituir necesariamente el default moderno.

---

## 91. Unicode passwords

La arquitectura deberá tratar cuidadosamente Unicode.

---

## 92. Password normalization

VoltStack no deberá modificar silenciosamente el password presentado mediante:

```text
trim
lowercase
case folding
```

---

## 93. Spaces are valid

Un password puede contener:

```text
leading spaces
trailing spaces
internal spaces
```

si la policy lo permite.

Por tanto:

```php
trim($password);
```

es peligroso.

---

## 94. Unicode normalization policy

Si una aplicación decide aplicar una normalización Unicode, deberá hacerlo explícitamente y consistentemente durante creación y verificación.

No deberá cambiarse silenciosamente entre versiones.

---

## 95. Byte length vs character length

La policy deberá distinguir cuando corresponda:

```text
characters
bytes
graphemes
```

especialmente con Unicode.

---

## 96. bcrypt length concern

El driver bcrypt deberá gestionar explícitamente las limitaciones inherentes del algoritmo y evitar truncamientos silenciosos peligrosos.

---

## 97. Driver capability metadata

Cada hasher podrá declarar:

```text
maximum safe input behavior
memory hardness
supported verification formats
rehash support
```

---

## 98. Password strength meters

VoltStack podrá integrarse con evaluadores de fuerza, pero estos serán advisory/policy components.

No deberán guardar el password.

---

## 99. Compromised Password Detection

Se recomienda soportar:

```text
CompromisedPasswordChecker
```

---

## 100. CompromisedPasswordCheckerInterface

```php
interface CompromisedPasswordCheckerInterface
{
    public function check(
        PasswordCandidate $password,
        CompromisedPasswordContext $context,
    ): CompromisedPasswordResult;
}
```

---

## 101. Cuándo verificar compromisos

Especialmente:

```text
registration
password change
password reset
```

Opcionalmente:

```text
successful login
```

bajo estrategias específicas.

---

## 102. Privacy

El password completo no deberá enviarse a servicios externos de breach checking.

Los adapters deberán utilizar mecanismos que preserven privacidad cuando el proveedor lo permita.

---

## 103. Breach service unavailable

La policy podrá decidir:

```text
FAIL_CLOSED
FAIL_OPEN_WITH_AUDIT
DEFER
```

dependiendo del entorno.

---

## 104. Registration policy

En sistemas normales podría preferirse impedir crear una contraseña conocida como comprometida.

---

## 105. Existing compromised password

Si se detecta durante login:

```text
valid password
+
known compromised
```

podrá resultar:

```text
RECOVERY_REQUIRED
```

en lugar de Authentication normal.

---

## 106. Password history

VoltStack podrá soportar:

```text
PasswordHistoryPolicy
```

para organizaciones que lo requieran.

---

## 107. Password history no será obligatorio por defecto

La arquitectura lo soportará sin imponerlo.

---

## 108. PasswordHistoryRecord

Nunca almacenar passwords antiguos en plaintext.

Podrá almacenar:

```text
historical hash
hash profile
created_at
```

---

## 109. History comparison challenge

Los hashes modernos utilizan salts aleatorios.

Por tanto no puede comprobarse reutilización mediante:

```text
newHash === oldHash
```

---

## 110. Correct history check

Debe verificarse el password candidato contra cada hash histórico permitido:

```text
candidate
    ↓
verify against history hash 1
verify against history hash 2
...
```

---

## 111. History cost

Esto puede ser costoso.

Por tanto deberá existir:

```text
bounded history depth
```

---

## 112. History DoS control

No permitir:

```text
hundreds of historical Argon2 hashes
```

en una sola operación.

---

## 113. Password expiration

VoltStack soportará expiración donde exista requerimiento regulatorio/organizacional.

Pero no deberá imponer:

```text
change every 30/60/90 days
```

por defecto.

---

## 114. PasswordExpirationPolicy

Podrá definir:

```text
never
fixed duration
identity type based
tenant policy
external policy
```

---

## 115. Password expired

No necesariamente significa:

```text
Identity disabled
```

Puede significar:

```text
PasswordCredential = EXPIRED
```

y:

```text
credential replacement required
```

---

## 116. Password age

Podrá calcularse desde:

```text
changedAt
```

no necesariamente desde la creación de Identity.

---

## 117. Password creation

Deberá centralizarse mediante:

```text
PasswordCredentialManager
```

---

## 118. PasswordCredentialManager responsibilities

```text
validate candidate
check compromise
check history
hash
persist
increment credential version
apply security invalidation policy
emit events
audit
```

---

## 119. Password creation flow

```text
PasswordCandidate
      ↓
PasswordPolicy
      ↓
CompromisedPasswordChecker
      ↓
HashingPolicy
      ↓
PasswordHasher
      ↓
CredentialRepository
      ↓
CredentialCreated
```

---

## 120. Password change

Debe distinguirse de password reset.

---

## 121. Password change

Normalmente implica:

```text
currently authenticated Identity
+
fresh verification / reauthentication
+
new password
```

---

## 122. Password reset

Normalmente implica:

```text
recovery evidence
+
reset authorization transaction
+
new password
```

sin conocer necesariamente el password actual.

---

## 123. Password change flow

```text
Authenticated Identity
      ↓
Fresh Authentication Check
      ↓
Current Password / Strong Evidence
      ↓
New Password Candidate
      ↓
Policy
      ↓
History
      ↓
Compromise Check
      ↓
Hash
      ↓
Atomic Credential Replacement
      ↓
CredentialVersion++
      ↓
Invalidation Policy
```

---

## 124. Password reset flow

```text
Recovery Transaction
      ↓
Recovery Evidence Verified
      ↓
Password Candidate
      ↓
Policy
      ↓
Compromise Check
      ↓
Credential Replacement
      ↓
CredentialVersion++
      ↓
SecurityVersion++
      ↓
Session/Token Revocation
      ↓
Recovery Completion
```

---

## 125. Password reset should be security-sensitive

Por defecto deberá considerarse más sensible que un cambio ordinario autenticado.

---

## 126. Old password after reset

Una vez completado el reset:

```text
old password
```

debe dejar de ser válido inmediatamente.

---

## 127. Atomic replacement

La sustitución deberá evitar ventanas donde:

```text
old password
+
new password
```

sean ambos válidos accidentalmente, salvo una estrategia de rotación explícita.

---

## 128. Credential rotation

Para passwords humanos normalmente será:

```text
replace old credential
```

No:

```text
multiple active passwords
```

---

## 129. Multiple password credentials

VoltStack Core podrá soportar técnicamente múltiples credentials, pero el provider humano estándar deberá asumir normalmente una sola password activa.

---

## 130. Service credentials

Los secretos de servicios pertenecen a un sistema relacionado pero no necesariamente al mismo modelo de `PasswordCredential`.

No deberá forzarse un API secret dentro de la semántica de password humano.

---

## 131. Password change requires fresh authentication

Una sesión antigua no deberá ser suficiente indefinidamente para cambiar password.

Podrá exigirse:

```text
freshness threshold
AAL
specific factor
```

---

## 132. FreshAuthenticationPolicy

Ejemplo:

```text
authenticated within last N minutes
```

o:

```text
fresh primary credential proof
```

---

## 133. Password confirmation

La ergonomía podrá ofrecer una operación equivalente a:

```text
password.confirm
```

pero implementada como reauthentication formal.

---

## 134. Reauthentication evidence

Una confirmación exitosa deberá generar Evidence con:

```text
method = password
verified_at
credential reference
```

---

## 135. Credential Version

Cada cambio de password deberá actualizar:

```text
CredentialVersion
```

---

## 136. SecurityVersion interaction

La policy decidirá si además incrementa:

```text
SecurityVersion
```

---

## 137. Example default

Para cambio normal:

```text
CredentialVersion++
SecurityVersion++
other sessions revoked
current session optionally retained
```

---

## 138. Reset default

Para reset/recovery:

```text
CredentialVersion++
SecurityVersion++
all previous sessions revoked
remember-me revoked
refresh-token families revoked
```

---

## 139. Password hash update due only to rehash

Un rehash transparente no equivale necesariamente a un cambio de credential conocido por el usuario.

Por tanto podría:

```text
keep CredentialVersion
```

o utilizar una versión técnica separada.

---

## 140. HashVersion vs CredentialVersion

Se recomienda distinguir:

```text
CredentialVersion
    semantic credential lifecycle

HashRevision
    storage/cryptographic representation update
```

---

## 141. Transparent rehash should not log user out

Normalmente:

```text
bcrypt → Argon2id
```

tras login válido no deberá invalidar todas las sesiones.

---

## 142. PasswordHasher migration concurrency

Dos requests concurrentes pueden detectar `needsRehash`.

La actualización deberá tolerar:

```text
duplicate rehash attempts
optimistic locking
compare-and-swap
```

---

## 143. Rehash compare-and-swap

Ejemplo:

```text
UPDATE credential
SET hash = new_hash
WHERE id = ?
AND hash_revision = old_revision
```

---

## 144. Rehash race outcome

Si otro request ya actualizó:

```text
no failure necessary
```

si la nueva credential sigue siendo válida.

---

## 145. Pepper

VoltStack podrá soportar un secret adicional:

```text
Password Pepper
```

pero no deberá ser obligatorio.

---

## 146. Salt vs Pepper

```text
Salt
    unique per password
    stored with hash
    public/non-secret

Pepper
    application secret
    not stored with password record
```

---

## 147. Pepper storage

Deberá residir en:

```text
secret manager
environment secret
HSM/KMS-backed configuration
```

según infraestructura.

No en la misma tabla que el hash.

---

## 148. Pepper strategy

Puede aplicarse mediante una construcción segura y claramente definida.

No deberá inventarse criptografía ad hoc.

---

## 149. PepperProvider

Conceptualmente:

```php
interface PasswordPepperProviderInterface
{
    public function current(): PasswordPepper;

    public function resolve(string $version): PasswordPepper;
}
```

---

## 150. Pepper version

Los hashes que dependan de pepper deberán poder identificar:

```text
pepper version
```

sin almacenar el secret.

---

## 151. Pepper rotation

La rotación plantea un problema similar a hash migration:

```text
old pepper
    ↓
verify successful login
    ↓
rehash with current pepper
```

---

## 152. Multiple active pepper versions

Durante migración podrá ser necesario aceptar:

```text
current
previous
```

de forma limitada.

---

## 153. Pepper compromise

Si el pepper se compromete:

```text
rotate secret
evaluate credential reset policy
increase monitoring
```

La respuesta dependerá del incidente.

---

## 154. Secret lifecycle outside Auth

La administración general de secretos pertenece al sistema de configuración/secret management.

Auth solo consume el pepper mediante contrato seguro.

---

## 155. Timing Resistance

El sistema deberá reducir diferencias observables entre:

```text
identity exists
identity does not exist
```

---

## 156. Enumeration timing problem

Flujo inseguro:

```text
unknown email
    → DB lookup
    → immediate reject

known email
    → expensive Argon2 verification
    → reject
```

Esto crea una diferencia temporal significativa.

---

## 157. Dummy Password Verification

Cuando Identity no exista podrá ejecutarse:

```text
DummyPasswordVerifier
```

utilizando un hash válido de costo comparable.

---

## 158. Dummy hash

Debe generarse/configurarse de forma segura.

No usar:

```text
hardcoded weak hash
```

con parámetros distintos a producción.

---

## 159. Dummy verification flow

```text
Identity NOT_FOUND
      ↓
DummyPasswordVerification
      ↓
Generic Authentication Failure
```

---

## 160. Exact timing equality impossible

VoltStack buscará:

```text
timing resistance
```

no prometerá:

```text
perfect constant-time full request execution
```

porque DB, red y runtime introducen variabilidad.

---

## 161. Hash comparison

La implementación criptográfica subyacente deberá realizar comparaciones apropiadas.

No implementar manualmente:

```php
$generated === $stored;
```

para password verification.

---

## 162. User enumeration response

Errores como:

```text
email not found
password incorrect
```

podrán mapearse externamente a:

```text
invalid credentials
```

---

## 163. Internal distinction

Internamente sí deberán distinguirse para:

```text
metrics
security analysis
debugging
```

con protección de privacidad.

---

## 164. Brute-force protection

Password Auth deberá integrarse con:

```text
AuthenticationRateLimiter
```

pero no contener toda la lógica dentro del hasher.

---

## 165. Rate limiting dimensions

Podrán considerarse:

```text
IP/network
identity claim fingerprint
tenant
device/session
firewall
authentication method
```

---

## 166. Avoid only per-account limiter

Un limiter únicamente por cuenta permite ataques de lockout.

---

## 167. Avoid only per-IP limiter

Un atacante distribuido puede evadirlo y redes NAT pueden causar falsos positivos.

---

## 168. Composite throttling

Preferible combinar varias señales mediante políticas.

---

## 169. PasswordAttemptPolicy

Podrá coordinar:

```text
rate limits
progressive delays
risk escalation
CAPTCHA integration
additional challenge
temporary method restrictions
```

---

## 170. Rate limiting occurs before expensive hashing when possible

Para proteger recursos:

```text
request
    ↓
safe rate-limit check
    ↓
allowed
    ↓
expensive password verification
```

---

## 171. Enumeration concern

El rate limiter tampoco deberá revelar fácilmente existencia de cuenta.

---

## 172. Progressive delay

Puede utilizarse con límites razonables.

No mantener workers PHP bloqueados durante tiempos largos mediante:

```php
sleep(30);
```

---

## 173. Better backpressure

Preferir respuestas controladas o sistemas de scheduling/rate limiting.

---

## 174. Credential stuffing

El sistema deberá permitir detección de:

```text
many identities from same source
known compromised credentials
distributed attempts
```

mediante integración con Risk/Observability.

---

## 175. Password spraying

Patrón:

```text
same/common password
against many accounts
```

requiere métricas distintas al brute-force de una sola cuenta.

---

## 176. Plaintext never enters metrics

Nunca usar:

```text
password
password hash
```

como metric label o attribute.

---

## 177. Authentication Evidence

Una verificación válida producirá:

```text
PasswordAuthenticationEvidence
```

---

## 178. PasswordAuthenticationEvidence

Podrá contener:

```text
identity reference
credential identifier
verification timestamp
authentication method = password
credential version
hasher profile
```

Nunca el password ni hash completo.

---

## 179. Evidence assurance

El password podrá contribuir a un nivel de assurance según la policy general.

No deberá codificarse:

```text
password always = AAL2
```

---

## 180. Password + MFA

El Evidence de password podrá combinarse posteriormente con:

```text
TOTP
WebAuthn
Passkey
hardware factor
```

---

## 181. Credential provenance

El AuthenticationContext podrá recordar de forma segura:

```text
credential id
credential version
method
```

para invalidación y auditoría.

---

## 182. Password Credential Lifecycle

Estados conceptuales:

```text
CREATED
   ↓
ACTIVE
   ├──→ REHASHED
   ├──→ RESET_REQUIRED
   ├──→ EXPIRED
   ├──→ COMPROMISED
   ├──→ DISABLED
   └──→ REVOKED
```

---

## 183. CREATED → ACTIVE

Solo después de:

```text
policy validation
hashing
successful persistence
```

---

## 184. ACTIVE → REHASHED

No es necesariamente un cambio semántico de credential.

---

## 185. ACTIVE → RESET_REQUIRED

Puede ocurrir por:

```text
admin action
security policy
compromise detection
legacy migration
```

---

## 186. ACTIVE → EXPIRED

Solo si existe expiration policy.

---

## 187. ACTIVE → COMPROMISED

Por:

```text
breach detection
incident response
user report
security automation
```

---

## 188. ACTIVE → DISABLED

Ejemplo:

```text
password authentication turned off
```

---

## 189. ACTIVE → REVOKED

Cuando se reemplaza permanentemente.

---

## 190. Credential resurrection

Una credential `REVOKED` no deberá volver a `ACTIVE`.

Crear una nueva credential/version.

---

## 191. CredentialStateMachine

Podrá validar transiciones.

---

## 192. Credential mutation

Cambios deberán realizarse mediante:

```text
PasswordCredentialManager
```

no modificando directamente:

```php
$user->password = ...
```

en el Core.

---

## 193. Ergonomía para aplicaciones simples

VoltStack podrá ofrecer una fachada:

```php
Password::hash($password);
```

para hashing genérico.

Pero Authentication deberá utilizar el pipeline completo.

---

## 194. Hash facade

Ejemplo:

```php
Hash::make($value);
Hash::check($value, $hash);
Hash::needsRehash($hash);
```

podrá existir por ergonomía/compatibilidad conceptual.

---

## 195. Hash facade no equivale a PasswordAuthenticator

```text
Hash
    cryptographic utility

PasswordAuthenticator
    Authentication mechanism
```

---

## 196. Password facade

Podría ofrecer:

```php
Password::change(...);
Password::reset(...);
```

si encaja con las convenciones finales del framework.

---

## 197. Configuration

Ejemplo conceptual:

```php
'authentication' => [

    'password' => [

        'hasher' => 'argon2id',

        'policy' => [
            'minimum_length' => 12,
            'maximum_input_bytes' => 4096,
            'compromised_check' => true,
            'history' => 0,
            'expiration' => null,
        ],

        'rehash' => true,

        'pepper' => [
            'enabled' => false,
        ],
    ],
];
```

Los valores exactos deberán definirse en configuración final y benchmarks, no considerarse universales por este ejemplo.

---

## 198. Hasher profiles

Ejemplo:

```php
'hashers' => [

    'argon2id' => [
        'driver' => 'argon2id',
        'memory' => env('AUTH_ARGON_MEMORY'),
        'time' => env('AUTH_ARGON_TIME'),
        'threads' => env('AUTH_ARGON_THREADS'),
    ],

    'bcrypt' => [
        'driver' => 'bcrypt',
        'cost' => env('AUTH_BCRYPT_COST'),
    ],
];
```

---

## 199. Configuration validation

Durante bootstrap deberá validarse:

```text
supported algorithm
valid parameters
reasonable bounds
pepper availability
migration drivers
```

---

## 200. Fail startup on dangerous configuration

Ejemplos:

```text
unknown hasher
missing required pepper
invalid cost
unsupported algorithm
```

deberán producir errores tempranos.

---

## 201. Security floor

VoltStack podrá establecer límites mínimos de seguridad para evitar configuraciones accidentalmente triviales.

La posibilidad de bajar de ese floor deberá ser explícita y claramente advertida si se soporta.

---

## 202. Development configuration

No debería reducirse tanto el costo que se oculten problemas de rendimiento o se generen hashes que terminen accidentalmente en producción.

Podrán existir perfiles separados.

---

## 203. Testing hasher

Para tests podrá existir:

```text
TestingPasswordHasher
```

con costo reducido.

Pero deberá activarse únicamente en testing.

---

## 204. Production guard

VoltStack podrá impedir que:

```text
TestingPasswordHasher
```

arranque bajo `production`.

---

## 205. Password storage schema

Modelo avanzado sugerido:

```text
authentication_password_credentials

id
identity_provider
identity_id
tenant_id nullable
hash
status
credential_version
hash_revision
pepper_version nullable
created_at
changed_at
expires_at nullable
revoked_at nullable
```

---

## 206. No algorithm column necessarily required

Si el hash es self-describing, duplicar:

```text
algorithm
```

puede crear inconsistencias.

Metadata adicional solo cuando sea útil.

---

## 207. Hash column size

La DB deberá reservar suficiente longitud para formatos presentes y futuros.

No asumir:

```text
VARCHAR(60)
```

solo porque bcrypt tenga un formato conocido.

---

## 208. Hash indexing

No existe necesidad normal de indexar el hash completo.

---

## 209. Credential lookup index

Indexar apropiadamente:

```text
identity reference
credential type
status
tenant
```

según modelo.

---

## 210. Sensitive backups

Hashes también deben protegerse en:

```text
database backups
replicas
logs
debug dumps
```

---

## 211. Encryption at rest

Puede complementar la seguridad, pero:

```text
encryption at rest
```

no sustituye:

```text
password hashing
```

---

## 212. Never encrypt passwords for later recovery

VoltStack no deberá proporcionar:

```text
decrypt user password
```

---

## 213. Password retrieval impossible by design

El sistema deberá permitir:

```text
reset password
```

no:

```text
send existing password
```

---

## 214. Password events

Podrán existir:

```text
PasswordCredentialCreated
PasswordCredentialChanged
PasswordCredentialReset
PasswordCredentialRehashed
PasswordCredentialExpired
PasswordCredentialCompromised
PasswordCredentialRevoked
PasswordAuthenticationSucceeded
PasswordAuthenticationFailed
```

---

## 215. Event payload safety

Nunca incluir:

```text
plaintext
hash
pepper
reset secret
```

---

## 216. Rehash event verbosity

`PasswordCredentialRehashed` podrá incluir:

```text
old profile
new profile
```

pero no hashes.

---

## 217. Audit events

Especialmente:

```text
password changed
password reset
admin forced reset
credential compromised
credential disabled
credential revoked
```

---

## 218. Failed password attempts

No necesariamente deberán producir un audit record persistente individual para cada intento.

Podrán utilizar:

```text
metrics
security telemetry
aggregated detection
```

para evitar audit flooding.

---

## 219. Observability spans

```text
auth.password.authenticate
auth.password.verify
auth.password.rehash
auth.password.policy
auth.password.change
auth.password.reset
```

---

## 220. Metrics

Ejemplos:

```text
auth_password_attempt_total
auth_password_success_total
auth_password_failure_total
auth_password_rehash_total
auth_password_rehash_failure_total
auth_password_policy_rejection_total
auth_password_compromised_rejection_total
auth_password_hash_duration
```

---

## 221. Safe metric labels

```text
hasher
profile
result
firewall
credential_status
policy_violation_type
```

Evitar:

```text
email
username
password
hash
identity id
```

como labels de alta cardinalidad.

---

## 222. Hash duration histogram

Será especialmente útil para:

```text
capacity planning
parameter tuning
DoS analysis
```

---

## 223. Memory metrics

Para Argon2id, el framework/runtime podrá correlacionar:

```text
concurrent hash operations
worker memory
authentication latency
```

---

## 224. Slow authentication alerts

Una degradación anormal puede indicar:

```text
resource exhaustion
misconfigured cost
credential stuffing
infrastructure pressure
```

---

## 225. Error handling

Las exceptions deberán distinguir internamente:

```text
PasswordHashingException
PasswordVerificationException
PasswordCredentialStorageException
PasswordPolicyException
PasswordRehashException
```

---

## 226. Public error mapping

No exponer:

```text
Argon2 allocation failed
database credential row missing
pepper version unavailable
```

al cliente.

---

## 227. Pepper unavailable

Si la credential depende de pepper y el secret no puede recuperarse:

```text
ERROR
```

y Authentication falla cerrada.

---

## 228. Unknown hash format

No asumir que significa password incorrecto.

Internamente:

```text
UNSUPPORTED_HASH_FORMAT
```

y fail closed.

---

## 229. Hash downgrade attack

Si un atacante logra modificar metadata para forzar un hasher más débil, el sistema deberá detectar incompatibilidades entre:

```text
stored format
allowed migration policy
hasher capabilities
```

---

## 230. Hasher selection from stored hash

Nunca instanciar clases arbitrarias basándose en strings controlables desde storage/request.

Resolver únicamente drivers registrados.

---

## 231. User-controlled hasher prohibited

Nunca:

```php
$hasher = app($request->input('hasher'));
```

---

## 232. Algorithm allowlist

Los algoritmos aceptados para verificación deberán provenir de configuración compilada.

---

## 233. Password pre-hashing

VoltStack no deberá introducir pre-hashing casero universal antes de Argon2id/bcrypt sin una razón y construcción criptográfica formal.

---

## 234. Client-side hashing

Un hash producido en JavaScript y enviado como sustituto permanente del password puede convertirse en:

```text
password-equivalent bearer secret
```

Por tanto no reemplaza TLS ni el hashing seguro del servidor.

---

## 235. Transport requirement

Las passwords deberán transmitirse únicamente sobre canales protegidos según el Security/HTTP layer.

Auth podrá exigir:

```text
secure transport
```

en producción.

---

## 236. Password in URL prohibited

Nunca aceptar password desde:

```text
query string
URL path
GET parameter intended for navigation
```

en authenticators estándar.

---

## 237. Password in redirect prohibited

Nunca colocar credentials en redirect URLs.

---

## 238. Password autocomplete

Las vistas/UI podrán utilizar atributos web adecuados, pero esta responsabilidad pertenece principalmente al frontend/view system.

---

## 239. CSRF

Un form login basado en cookie/session deberá integrarse con CSRF protection cuando corresponda.

Esto pertenece al Security/HTTP layer, aunque el PasswordAuthenticator deberá exigir el resultado necesario.

---

## 240. Login CSRF

El login también puede sufrir CSRF/login confusion.

La integración deberá contemplarse explícitamente.

---

## 241. Password reset token separation

Un:

```text
PasswordResetToken
```

no es:

```text
PasswordCredential
```

y será definido por el sistema de Account Recovery.

---

## 242. Password reset token storage

No deberá reutilizar el campo `password`.

---

## 243. Reset transaction binding

El reset deberá estar vinculado a:

```text
Identity
purpose
expiry
nonce/token state
```

según el futuro Recovery System.

---

## 244. Admin-set passwords

Si se permite que un administrador establezca una password temporal, la policy podrá exigir:

```text
RESET_REQUIRED
```

para el primer uso.

---

## 245. Generated temporary password

No deberá enviarse por canales inseguros si puede evitarse.

Preferible:

```text
secure enrollment/reset link
```

---

## 246. Passwordless identities

Una Identity podrá no tener `PasswordCredentialRecord`.

Esto deberá producir:

```text
password method not available
```

no un error de modelo.

---

## 247. Federation-only Identity

Ejemplo:

```text
Identity ACTIVE
PasswordCredential = none
Restriction = FEDERATED_LOGIN_REQUIRED
```

---

## 248. Passkey-only Identity

Igualmente válida.

---

## 249. Optional password subsystem

Una aplicación VoltStack podrá desactivar completamente password authentication.

---

## 250. PasswordAuthenticator registration

Solo deberá registrarse en Firewalls donde esté permitido.

---

## 251. Multi-firewall policy

Ejemplo:

```text
web:
    password allowed

admin:
    passkey + MFA only

api:
    password unsupported
```

---

## 252. Multi-tenant password policy

Distintos tenants podrán tener políticas diferentes dentro de límites de seguridad definidos por la plataforma.

---

## 253. Platform security floor

Un tenant no deberá poder configurar:

```text
minimum_length = 1
weak hashing
plaintext storage
```

si viola el security floor de VoltStack.

---

## 254. Policy composition

```text
Framework Security Floor
        +
Application Policy
        +
Tenant Policy
        ↓
Effective Password Policy
```

---

## 255. More restrictive composition

Cuando policies entren en conflicto, deberá definirse composición segura.

Ejemplo:

```text
Framework min = 10
Tenant min = 14

Effective = 14
```

---

## 256. Hashing parameters and tenant control

No es recomendable permitir que cada tenant controle arbitrariamente parámetros criptográficos que afecten capacidad global del servidor.

La plataforma deberá conservar control.

---

## 257. Authentication Policy Version

Un cambio de PasswordPolicy no deberá invalidar automáticamente todas las credentials existentes salvo que una migration policy lo indique.

---

## 258. Existing credentials grandfathering

Podrán existir políticas:

```text
APPLY_ON_NEXT_CHANGE
REQUIRE_RESET
RECHECK_ON_LOGIN
```

---

## 259. Compromised password recheck

Podrá ejecutarse periódicamente fuera del request login mediante procesos especializados.

No será requisito del hot path.

---

## 260. Offline security scans

Podrán trabajar sobre metadata o integraciones específicas, pero nunca requerir plaintext almacenado.

---

## 261. Testing — hashing

Casos:

```text
hash then verify
wrong password
needs rehash
parameter change
algorithm migration
unknown format
pepper version
```

---

## 262. Testing — policy

```text
too short
long passphrase
Unicode
spaces
maximum input size
compromised password
history reuse
```

---

## 263. Testing — lifecycle

```text
create
change
reset
expire
disable
revoke
compromise
rehash
```

---

## 264. Testing — identity states

```text
ACTIVE + valid password
DISABLED + valid password
RECOVERY_REQUIRED + valid password
PASSWORD_AUTH_DISABLED + valid password
```

---

## 265. Testing — enumeration

Comparar comportamientos para:

```text
unknown identity
known identity + wrong password
```

y verificar dummy hashing/failure mapping.

---

## 266. Testing — rate limiting

```text
per-source limits
per-claim limits
distributed attempts
legitimate NAT traffic
```

---

## 267. Testing — concurrency

Especialmente:

```text
simultaneous password changes
rehash race
password reset during login
credential revoke during verification
security version change before commit
```

---

## 268. Testing — persistent runtime

Verificar que un worker no conserve:

```text
previous password
previous credential
previous Identity
previous verification result
pepper material in request state
```

entre requests.

---

## 269. Memory inspection considerations

PHP no permite garantizar borrado físico inmediato de todos los copies de strings.

VoltStack deberá minimizar copias y lifetime sin prometer secure memory guarantees que el runtime no ofrece.

---

## 270. No false zeroization guarantee

No documentar:

```text
password is guaranteed erased from RAM
```

si PHP/runtime no puede garantizarlo.

---

## 271. Stateless PasswordVerifier

Deberá ser preferentemente:

```text
stateless
immutable
```

---

## 272. Stateless PasswordAuthenticator

No deberá almacenar en propiedades compartidas:

```text
current password
current identity
current credential
current result
```

---

## 273. Request-scoped sensitive state

Vivirá en:

```text
AuthenticationTransaction
Passport
PasswordCredential
```

y desaparecerá al finalizar la operación.

---

## 274. Core invariants — Password

### AUTH-PWD-01

Plaintext passwords are never persisted.

#### AUTH-PWD-02

Plaintext passwords are never logged.

#### AUTH-PWD-03

PasswordCredential and PasswordCredentialRecord are different concepts.

#### AUTH-PWD-04

Password verification uses dedicated password-hashing algorithms.

#### AUTH-PWD-05

PasswordAuthenticator does not implement hashing algorithms.

#### AUTH-PWD-06

IdentityProvider does not own password verification.

#### AUTH-PWD-07

PasswordPolicy and PasswordHashingPolicy remain separate.

#### AUTH-PWD-08

Passwordless identities are valid identities.

#### AUTH-PWD-09

Password authentication can be disabled independently of Identity status.

#### AUTH-PWD-10

A valid password does not override Identity ineligibility.

---

## 275. Core invariants — Hashing

### AUTH-HASH-01

New hashes use only approved registered hashers.

#### AUTH-HASH-02

Legacy hashers may be verify-only.

#### AUTH-HASH-03

Rehash occurs only after successful verification.

#### AUTH-HASH-04

Algorithm migration must be supported.

#### AUTH-HASH-05

Hash parameters are centrally governed.

#### AUTH-HASH-06

Load does not trigger automatic cryptographic weakening.

#### AUTH-HASH-07

Unknown hash formats fail closed.

#### AUTH-HASH-08

Pepper secrets are never stored beside hashes.

#### AUTH-HASH-09

Hasher selection cannot be controlled by request input.

#### AUTH-HASH-10

Transparent rehash does not inherently mean credential rotation.

---

## 276. Core invariants — Lifecycle

### AUTH-PWD-LIFE-01

Password changes increment credential lifecycle state.

#### AUTH-PWD-LIFE-02

Password reset is distinct from password change.

#### AUTH-PWD-LIFE-03

Revoked passwords cannot be resurrected.

#### AUTH-PWD-LIFE-04

Critical resets can invalidate existing Authentication state.

#### AUTH-PWD-LIFE-05

Credential mutation is explicit and auditable.

#### AUTH-PWD-LIFE-06

Concurrent mutation requires consistency control.

#### AUTH-PWD-LIFE-07

History never stores plaintext.

#### AUTH-PWD-LIFE-08

Expiration is policy-driven, not universally mandatory.

#### AUTH-PWD-LIFE-09

Credential compromise can trigger Identity Security State changes.

#### AUTH-PWD-LIFE-10

Password reset never reveals the previous password.

---

## 277. Core invariants — Enumeration and Abuse

### AUTH-PWD-ABUSE-01

Unknown Identity and wrong Password should not have trivially distinguishable execution paths.

#### AUTH-PWD-ABUSE-02

Dummy password verification uses representative hashing cost.

#### AUTH-PWD-ABUSE-03

Rate limiting occurs independently from account disabling.

#### AUTH-PWD-ABUSE-04

Brute-force protection must not rely solely on permanent account lockout.

#### AUTH-PWD-ABUSE-05

Password values never enter telemetry.

#### AUTH-PWD-ABUSE-06

Oversized password inputs are bounded before expensive processing.

#### AUTH-PWD-ABUSE-07

Hash resource usage is governed under concurrency.

---

## 278. Anti-pattern — Password in User contract

Evitar:

```php
interface UserInterface
{
    public function getPassword(): string;
}
```

como requisito universal de Identity.

---

## 279. Anti-pattern — hashing inside controller

Evitar:

```php
$user->password = password_hash(
    $request->password,
    PASSWORD_DEFAULT
);
```

disperso por la aplicación.

---

## 280. Anti-pattern — direct hash comparison

Evitar:

```php
hash($password) === $storedHash;
```

---

## 281. Anti-pattern — SHA-256 password storage

Evitar:

```php
hash('sha256', $password);
```

---

## 282. Anti-pattern — plaintext backup

Nunca guardar:

```text
current_password
original_password
temporary_plain_password
```

---

## 283. Anti-pattern — password trim

Evitar:

```php
$password = trim($request->password);
```

---

## 284. Anti-pattern — same error internally and externally

Mantener distinción interna sin revelar información innecesaria externamente.

---

## 285. Anti-pattern — user not found returns immediately

Esto facilita timing-based enumeration.

---

## 286. Anti-pattern — weak test hasher in production

Debe existir protección de entorno.

---

## 287. Anti-pattern — arbitrary tenant hash cost

Puede provocar:

```text
resource exhaustion
```

---

## 288. Anti-pattern — automatic password expiration

No imponer cambios periódicos sin una policy/requirement real.

---

## 289. Anti-pattern — password history by hash equality

Los salts hacen incorrecto este enfoque.

---

## 290. Anti-pattern — password reset keeps all sessions

Para recovery sensible deberá aplicarse invalidation policy adecuada.

---

## 291. Anti-pattern — pepper hardcoded in source

Nunca:

```php
$pepper = 'voltstack-secret';
```

---

## 292. Anti-pattern — logging hashes

Aunque sean hashes, siguen siendo material sensible para ataques offline.

---

## 293. Anti-pattern — client-side hash as replacement for TLS

No proporciona el modelo de seguridad correcto.

---

## 294. Componentes principales

```text
PasswordAuthenticator
PasswordCredential
PasswordCredentialRecord
PasswordCredentialStatus
PasswordCredentialRepository
PasswordCredentialManager
PasswordVerifier
PasswordVerificationResult

PasswordHasher
PasswordHasherManager
PasswordHasherRegistry
PasswordHash
PasswordHashingPolicy
PasswordHashProfile
PasswordHashRevision

PasswordPolicy
PasswordCandidate
PasswordPolicyResult
PasswordPolicyViolation

CompromisedPasswordChecker
PasswordHistoryPolicy
PasswordExpirationPolicy
PasswordPepperProvider
DummyPasswordVerifier
```

---

## 295. Componentes de integración

```text
AuthenticationRateLimiter
PasswordAttemptPolicy
AuthenticationStateInvalidationPolicy
FreshAuthenticationPolicy
IdentitySecurityStateManager
AuthenticationAudit
AuthenticationTelemetry
```

---

## 296. Namespace sugerido

```text
VoltStack\Quantum\Auth\Password
VoltStack\Quantum\Auth\Password\Authenticator
VoltStack\Quantum\Auth\Password\Credential
VoltStack\Quantum\Auth\Password\Hashing
VoltStack\Quantum\Auth\Password\Policy
VoltStack\Quantum\Auth\Password\Lifecycle
VoltStack\Quantum\Auth\Password\Security
```

---

## 297. Estructura sugerida

```text
src/Quantum/Auth/
└── Password/
    ├── Authenticator/
    │   ├── PasswordAuthenticator.php
    │   └── PasswordAuthenticationRequest.php
    │
    ├── Credential/
    │   ├── PasswordCredential.php
    │   ├── PasswordCredentialRecord.php
    │   ├── PasswordCredentialStatus.php
    │   ├── PasswordCredentialRepository.php
    │   └── PasswordVerificationResult.php
    │
    ├── Hashing/
    │   ├── PasswordHasher.php
    │   ├── PasswordHasherManager.php
    │   ├── PasswordHasherRegistry.php
    │   ├── PasswordHash.php
    │   ├── PasswordHashingPolicy.php
    │   ├── PasswordHashProfile.php
    │   ├── PasswordHashRevision.php
    │   ├── Argon2idPasswordHasher.php
    │   ├── BcryptPasswordHasher.php
    │   └── LegacyPasswordHasher.php
    │
    ├── Policy/
    │   ├── PasswordPolicy.php
    │   ├── PasswordCandidate.php
    │   ├── PasswordPolicyResult.php
    │   ├── PasswordPolicyViolation.php
    │   ├── PasswordHistoryPolicy.php
    │   └── PasswordExpirationPolicy.php
    │
    ├── Lifecycle/
    │   ├── PasswordCredentialManager.php
    │   ├── PasswordCredentialStateMachine.php
    │   ├── PasswordChangeService.php
    │   └── PasswordResetIntegration.php
    │
    └── Security/
        ├── PasswordVerifier.php
        ├── DummyPasswordVerifier.php
        ├── CompromisedPasswordChecker.php
        ├── PasswordPepperProvider.php
        └── PasswordAttemptPolicy.php
```

---

## 298. Flujo completo de login

```text
HTTP Request
    │
    ▼
PasswordAuthenticator
    │
    ├── extract IdentityClaim
    └── extract PasswordCredential
    │
    ▼
IdentityResolver
    │
    ├── Identity found
    │       │
    │       ▼
    │   Eligibility Check
    │       │
    │       ▼
    │   PasswordCredentialRepository
    │       │
    │       ▼
    │   PasswordVerifier
    │
    └── Identity not found
            │
            ▼
      DummyPasswordVerifier
            │
            ▼
       Generic Failure

Valid password path:

PasswordVerifier
    │
    ▼
PasswordHasher
    │
    ├── invalid
    │      ↓
    │   failure
    │
    └── valid
           │
           ▼
      needsRehash?
        │      │
       no     yes
        │      │
        │      └──→ Rehash Coordinator
        │
        ▼
PasswordAuthenticationEvidence
        │
        ▼
Authentication Policy
        │
        ▼
Commit-time Security Check
        │
        ▼
AuthenticationContext
```

---

## 299. Flujo de migración bcrypt → Argon2id

```text
Stored Credential:
    bcrypt hash
        ↓
User submits correct password
        ↓
BcryptPasswordHasher verifies
        ↓
HashingPolicy:
    bcrypt = ACCEPTED_REHASH
        ↓
Argon2idPasswordHasher
        ↓
new Argon2id hash
        ↓
atomic credential hash update
        ↓
Authentication continues
```

---

## 300. Flujo de password incorrecto

```text
Identity resolved
    ↓
PasswordCredential loaded
    ↓
PasswordVerifier
    ↓
INVALID
    ↓
attempt telemetry
    ↓
rate/risk update
    ↓
generic authentication failure
```

---

## 301. Flujo Identity inexistente

```text
IdentityClaim
    ↓
IdentityResolver
    ↓
NOT_FOUND
    ↓
DummyPasswordVerifier
    ↓
attempt telemetry
    ↓
same external failure class
```

---

## 302. Flujo de password change

```text
Authenticated Identity
    ↓
Fresh Authentication Policy
    ↓
fresh proof satisfied
    ↓
New Password Candidate
    ↓
Password Policy
    ↓
Compromised Password Check
    ↓
Password History Check
    ↓
Current Hashing Policy
    ↓
New Password Hash
    ↓
Atomic Credential Replacement
    ↓
CredentialVersion++
    ↓
AuthenticationStateInvalidationPolicy
    ↓
Audit + Events
```

---

## 303. Flujo de password comprometido

```text
Password Candidate / Existing Credential
        ↓
CompromisedPasswordChecker
        ↓
COMPROMISED
        ↓
PasswordCredentialManager
        ↓
credential state update
        ↓
IdentitySecurityStateManager
        ↓
RECOVERY_REQUIRED / COMPROMISED
        ↓
SecurityVersion++
        ↓
Authentication state invalidation
```

---

## 304. Flujo de rehash concurrente

```text
Request A                  Request B
    │                          │
verify old hash            verify old hash
    │                          │
needs rehash               needs rehash
    │                          │
new hash A                 new hash B
    │                          │
CAS update succeeds        CAS update fails
    │                          │
    └──────────────┬───────────┘
                   ▼
           credential already
             upgraded safely
```

---

## 305. Flujo con pepper

```text
PasswordCredential
       │
       ▼
PepperProvider
       │
       ▼
Versioned Pepper
       │
       ▼
Approved Pepper Construction
       │
       ▼
PasswordHasher
       │
       ▼
PasswordHash
       │
       ▼
Stored Hash
+
Pepper Version Metadata
```

El pepper secret nunca se almacena en el registro.

---

## 306. Arquitectura global

```text
                     PASSWORD AUTHENTICATION
                              │
             ┌────────────────┴────────────────┐
             ▼                                 ▼
      Identity Claim                  Password Credential
             │                                 │
             ▼                                 │
      Identity Resolver                        │
             │                                 │
             ▼                                 │
     Identity Security State                   │
             │                                 │
             ▼                                 │
     Eligibility Evaluator                     │
             │                                 │
             └──────────────┬──────────────────┘
                            ▼
                Password Credential Store
                            │
                            ▼
                    Password Verifier
                            │
                            ▼
                  Password Hasher Manager
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       Argon2id           bcrypt            Legacy
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                 Verification Result
                            │
                ┌───────────┼────────────┐
                ▼           ▼            ▼
             INVALID      VALID      VALID + REHASH
                            │            │
                            │            ▼
                            │      Rehash Coordinator
                            │            │
                            └──────┬─────┘
                                   ▼
                     Authentication Evidence
                                   │
                                   ▼
                      Authentication Policy
                                   │
                                   ▼
                     Authentication Context
```

---

## 307. Decisiones arquitectónicas principales

VoltStack adoptará como principios:

```text
1. Password is optional.
2. Identity is password-independent.
3. Hashing is driver-based.
4. Password policy is independent from hashing policy.
5. Algorithm agility is mandatory.
6. Legacy hashes are migration concerns.
7. Rehash is transparent after successful verification.
8. Password reset is a security lifecycle operation.
9. Credential state is independently versioned.
10. Identity Security State always wins over valid password.
```

---

## 308. Comparación conceptual con Laravel y Symfony

VoltStack deberá conservar la ergonomía que hace atractivo el enfoque Laravel:

```text
simple hashing API
manager/driver model
easy password verification
automatic rehash capabilities
clear configuration
```

y las fortalezas arquitectónicas del enfoque Symfony:

```text
password hasher abstraction
identity/user separation
password upgrading
credential lifecycle hooks
security pipeline integration
```

pero deberá llevar ambos conceptos a un modelo más general:

```text
Identity
+
Credential
+
Credential Repository
+
Hasher
+
Hashing Policy
+
Password Policy
+
Credential Lifecycle
+
Identity Security State
+
Evidence
```

De esta forma Password Authentication será solamente uno de los mecanismos enchufables del sistema completo.

---

## 309. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte PasswordCredential;
2. separe plaintext y stored credential;
3. soporte PasswordCredentialRecord;
4. soporte CredentialIdentifier;
5. soporte PasswordAuthenticator;
6. soporte PasswordCredentialRepository;
7. soporte PasswordVerifier;
8. soporte PasswordHasher abstraction;
9. soporte Argon2id;
10. soporte bcrypt;
11. soporte drivers legacy;
12. soporte algorithm agility;
13. soporte `needsRehash`;
14. soporte transparent rehash;
15. soporte migration de hashes;
16. soporte PasswordHashingPolicy;
17. soporte PasswordPolicy;
18. soporte long passwords;
19. preserve espacios;
20. soporte Unicode de forma explícita;
21. limite inputs excesivos;
22. soporte compromised-password checking;
23. soporte password history opcional;
24. soporte expiration opcional;
25. soporte password change;
26. soporte password reset integration;
27. soporte CredentialVersion;
28. distinga HashRevision;
29. soporte SecurityVersion integration;
30. soporte pepper opcional;
31. soporte pepper rotation;
32. soporte dummy verification;
33. mitigue enumeration timing;
34. integre rate limiting;
35. soporte resource governance;
36. soporte concurrency;
37. soporte audit;
38. soporte observability;
39. sea seguro con FrankenPHP;
40. permita desactivar completamente passwords.

---

## 310. Regla arquitectónica final

VoltStack deberá preservar la siguiente separación:

```text
PASSWORD
    secret presented by the actor

PASSWORD CREDENTIAL
    authentication input

PASSWORD CREDENTIAL RECORD
    persisted credential state

PASSWORD HASHER
    protects and verifies password representation

HASHING POLICY
    determines cryptographic protection

PASSWORD POLICY
    determines acceptable password selection

CREDENTIAL LIFECYCLE
    controls creation/change/reset/revocation

IDENTITY SECURITY STATE
    determines whether credential may be used

AUTHENTICATION EVIDENCE
    records successful proof

AUTHENTICATION CONTEXT
    represents established authentication
```

La regla central será:

> **Una contraseña correcta únicamente demuestra conocimiento de una credential. No puede reactivar una Identity deshabilitada, ignorar una restricción de seguridad ni sustituir los controles posteriores del pipeline de Authentication.**

Además:

> **VoltStack deberá asumir que algoritmos, costos y políticas de password evolucionarán durante toda la vida de una aplicación. Por ello, la migración criptográfica será una capacidad fundamental del sistema y no una adaptación posterior.**

---

## 311. Próximo documento

El siguiente documento recomendado será:

```text
12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md
```

Este documento deberá definir el subsistema completo de autenticación persistente mediante sesiones:

```text
AuthenticationSession
SessionAuthenticator
session identity binding
AuthenticationContext persistence
context serialization
context restoration
session identifiers
session rotation
session fixation protection
login session migration
logout
forced logout
idle timeout
absolute timeout
session renewal
SecurityVersion validation
CredentialVersion interaction
remember-me integration boundaries
concurrent sessions
session limits
device/session metadata
tenant binding
session revocation
session store abstraction
distributed session stores
Redis/database/file adapters
session theft detection
cookie binding
secure cookie policy
SameSite
HttpOnly
Secure
CSRF interaction
SPA session authentication
FrankenPHP persistent runtime safety
audit
observability
testing
```

Con este documento se pasará de:

```text
Identity + valid proof
```

a:

```text
Identity + valid proof
        ↓
persistent Authentication state
        ↓
future request
        ↓
safe AuthenticationContext restoration
```

cerrando así la primera gran transición entre **autenticar una Identity** y **mantener esa autenticación de forma segura entre múltiples requests**.
