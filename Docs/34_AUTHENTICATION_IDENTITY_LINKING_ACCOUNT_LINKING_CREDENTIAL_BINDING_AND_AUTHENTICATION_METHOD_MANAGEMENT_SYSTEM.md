# VoltStack Authentication System

## 34 — Identity Linking, Account Linking, Credential Binding and Authentication Method Management System

- **Archivo:** `34_AUTHENTICATION_IDENTITY_LINKING_ACCOUNT_LINKING_CREDENTIAL_BINDING_AND_AUTHENTICATION_METHOD_MANAGEMENT_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Nivel:** Core / Security Critical
- **Dependencias principales:** 02, 04, 06, 08, 09, 10, 11, 13, 15, 16, 17, 18, 20, 23, 24, 25, 26, 29, 31, 32 y 33.

---

## 1. Propósito

Este documento define el subsistema responsable de administrar la relación entre una identidad lógica y los distintos mecanismos mediante los cuales esa identidad puede autenticarse.
VoltStack deberá separar explícitamente:

```text
IDENTITY
   │
   ├── Authentication Method
   ├── Authentication Method
   ├── Authentication Method
   └── Authentication Method
```

Una identidad humana podría tener:

```text
Identity: user-123
│
├── Password
├── Passkey: MacBook
├── Passkey: iPhone
├── TOTP
├── Recovery Codes
├── Google OIDC
├── Microsoft OIDC
└── Enterprise SSO
```

La identidad:

- user-123
- no deberá depender conceptualmente de ninguno de esos mecanismos.

La regla central será:
Una identidad representa quién es el principal; un Authentication Method representa una forma mediante la cual puede demostrarlo.

## 2. Problema arquitectónico

Laravel y Symfony proporcionan excelentes mecanismos para autenticar usuarios, providers, passwords, tokens y authenticators.
Sin embargo, un sistema de autenticación moderno necesita resolver también:

```text
¿Cómo agrego una passkey a una cuenta existente?

¿Cómo elimino una contraseña?

¿Cómo vinculo Google?

¿Cómo desvinculo Google?

¿Cómo vinculo una cuenta corporativa?
```

¿Qué ocurre si ese Google Account ya pertenece a otro usuario?

¿Cómo evito account takeover durante linking?

¿Qué ocurre si elimino mi último método de acceso?

¿Cómo cambio una credencial sin cambiar de identidad?

¿Cómo fusiono dos cuentas?

¿Cómo migro usuarios de password hacia passkeys?

¿Cómo verifico que quien agrega una credencial controla realmente esa credencial?

¿Cómo audito todos esos cambios?
Estas operaciones no deberán distribuirse arbitrariamente entre:

- controllers
- authenticators
- OAuth callbacks
- models
- middleware
- password reset handlers

VoltStack tendrá un subsistema específico.

## 3. Separación fundamental

No deberá existir equivalencia:
Identity = Password
ni:
Identity = Email
ni:
Identity = Google Account
ni:
Identity = Passkey
El modelo correcto será:

```text
                   IDENTITY
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
 Authentication   Authentication  Authentication
    Method           Method          Method
       │              │              │
       ▼              ▼              ▼
   Password        Passkey       External OIDC
```

## 4. Identity como raíz

Conceptualmente:

```php
final readonly class Identity
{
    public function __construct(
        public string $id,
        public PrincipalType $type,
        public IdentitySecurityState $securityState,
    ) {}
}
```

La implementación concreta podrá variar según el documento 09.

## 5. AuthenticationMethod

VoltStack introducirá una abstracción explícita:

```php
interface AuthenticationMethodInterface
{
    public function id(): AuthenticationMethodId;

    public function identityId(): string;

    public function type(): AuthenticationMethodType;

    public function status(): AuthenticationMethodStatus;
}
```

## 6. AuthenticationMethodType

enum AuthenticationMethodType: string
{
case Password = 'password';

case Passkey = 'passkey';

case Totp = 'totp';

case RecoveryCode = 'recovery_code';

case ExternalOidc = 'external_oidc';

case ExternalOAuth = 'external_oauth';

case EnterpriseSso = 'enterprise_sso';

case Certificate = 'certificate';

case MachineCredential = 'machine_credential';

case Custom = 'custom';
}
La enumeración podrá extenderse mediante registry.

## 7. Authentication Method != Credential

Debe distinguirse:

```text
Authentication Method
        ≠
Credential
```

Ejemplo:

```text
Method:
Passkey
```

Credential:
specific WebAuthn credential
Otro ejemplo:

```text
Method:
Password
```

Credential:
current password verifier

## 8. Authentication Method Instance

Una identidad puede tener varias instancias del mismo tipo.

```text
Passkey
├── MacBook Pro
├── iPhone
└── YubiKey
```

Por tanto, el modelo no deberá asumir:
one method type = one credential

## 9. AuthenticationMethodId

final readonly class AuthenticationMethodId
{
public function __construct(
public string $value,
) {}
}
Debe ser estable e independiente del secret.

## 10. AuthenticationMethodStatus

enum AuthenticationMethodStatus: string
{
case Pending = 'pending';

case Active = 'active';

case Suspended = 'suspended';

case Compromised = 'compromised';

case Revoked = 'revoked';

case Retired = 'retired';
}

## 11. Method Metadata

Cada método podrá contener metadata no secreta:

- createdAt
- activatedAt
- lastUsedAt
- lastVerifiedAt
- displayName
- provider
- deviceName
- credentialId
- assuranceProperties

## 12. Secret Separation

Material secreto deberá almacenarse mediante los subsistemas apropiados.
No:

```text
AuthenticationMethod
    ├── metadata
    └── plaintext secret
```

## 13. Authentication Method Registry

VoltStack deberá mantener un registro de tipos soportados.

```php
interface AuthenticationMethodRegistryInterface
{
    public function register(
        AuthenticationMethodDefinition $definition
    ): void;

    public function get(
        AuthenticationMethodType|string $type
    ): AuthenticationMethodDefinition;
}
```

## 14. AuthenticationMethodDefinition

Podrá declarar:

- type
- provider
- binding handler
- removal handler
- verification requirements
- assurance characteristics
- supportsMultiple
- supportsRename
- supportsRotation
- supportsSuspension

## 15. Authentication Method Manager

Será el orquestador principal.

```php
interface AuthenticationMethodManagerInterface
{
    public function methodsFor(
        string $identityId
    ): AuthenticationMethodCollection;

    public function bind(
        AuthenticationMethodBindingRequest $request
    ): AuthenticationMethodBindingResult;

    public function remove(
        AuthenticationMethodRemovalRequest $request
    ): AuthenticationMethodRemovalResult;
}
```

## 16. Responsabilidades del Manager

No deberá verificar passwords directamente ni procesar WebAuthn criptográficamente.
Deberá orquestar:

- request
- policy
- proof
- provider
- binding
- security state
- events
- audit
- result

## 17. Credential Binding

Credential Binding es el proceso mediante el cual una nueva credencial queda asociada de forma segura a una identidad existente.
Existing Identity
│
▼
Request New Method
│
▼
Verify Existing Identity
│
▼
Verify New Credential
│
▼
Binding Policy
│
▼
Bind Credential
│
▼
Authentication Method Active

## 18. Binding no es simplemente INSERT

Nunca:

```php
$user->credentials()->create($data);
como único mecanismo de seguridad.
```

El binding es una ceremonia de seguridad.

## 19. Binding Ceremony

Toda vinculación deberá poder modelarse como:

```text
INITIATED
   ↓
CURRENT_IDENTITY_VERIFIED
   ↓
NEW_CREDENTIAL_VERIFIED
   ↓
POLICY_APPROVED
   ↓
BOUND
```

o:

- FAILED
- CANCELLED
- EXPIRED

## 20. BindingTransaction

final readonly class AuthenticationMethodBindingTransaction
{
public function __construct(
public string $id,
public string $identityId,
public AuthenticationMethodType $methodType,
public BindingTransactionStatus $status,
public DateTimeImmutable $createdAt,
public DateTimeImmutable $expiresAt,
) {}
}

## 21. BindingTransactionStatus

enum BindingTransactionStatus: string
{
case Initiated = 'initiated';

case IdentityVerified = 'identity_verified';

case CredentialVerified = 'credential_verified';

case PendingApproval = 'pending_approval';

case Completed = 'completed';

case Failed = 'failed';

case Cancelled = 'cancelled';

case Expired = 'expired';
}

## 22. Binding Transaction Lifetime

Las ceremonias deberán expirar.
Nunca:

- start linking today
- finish six months later

## 23. Binding Nonce

Deberá utilizarse nonce/state/challenge según mecanismo para evitar:

- CSRF
- replay
- session confusion
- provider callback injection

## 24. Current Identity Verification

Agregar un nuevo método de autenticación puede ser una operación sensible.
Por tanto, no siempre bastará:
existing session

## 25. Reauthentication

El documento 32 deberá integrarse aquí.

```text
Ejemplo:
Session created 10 hours ago
        ↓
Add new Passkey
        ↓
Fresh Authentication Required
```

## 26. BindingAuthenticationRequirement

final readonly class BindingAuthenticationRequirement
{
public function __construct(
public AuthenticationAssuranceRequirement $assurance,
public ?DateInterval $maximumAuthenticationAge,
) {}
}

## 27. New Credential Verification

No basta con recibir los datos.
VoltStack deberá demostrar que el usuario controla la nueva credencial.
Ejemplos:

```text
Password
→ valid creation ceremony

Passkey
→ WebAuthn registration ceremony

TOTP
→ valid TOTP confirmation

OIDC
→ successful provider authentication

Certificate
→ proof of private key possession
```

## 28. Credential Binding Proof

interface CredentialBindingProofInterface
{
public function methodType(): AuthenticationMethodType;
}

## 29. Binding Proof != Authentication Proof

Podrán compartir primitivas, pero representan ceremonias diferentes.

## 30. Binding Policy

interface AuthenticationMethodBindingPolicyInterface
{
public function evaluate(
AuthenticationMethodBindingContext $context
): AuthenticationMethodBindingDecision;
}

## 31. Policy podrá considerar

identity security state
current authentication age
current assurance
new method type
number of existing methods
tenant
realm
risk
device
provider
organization policy

## 32. Password Binding

Agregar una contraseña a una cuenta passwordless deberá requerir:

- authenticated identity
- +
- fresh authentication
- +
- password policy
- +
- password hashing
- +
- credential binding

## 33. Password Replacement

Cambiar password no deberá implementarse como:

- delete old
- create unrelated identity

Es una operación sobre el método.

## 34. Password Change

Identity
↓
Password Method
↓
Verify Current State
↓
Verify Existing Credential / Step-Up
↓
Validate New Password
↓
Hash New Password
↓
Increment Credential Version
↓
Invalidate Relevant Sessions
según policy.

## 35. Password Rotation

No deberá exigirse periódicamente sin razón únicamente por arquitectura.
La policy podrá imponerla donde exista requisito organizacional.

## 36. Password Compromise

Si se detecta compromiso:

```text
Password Method
    ↓
COMPROMISED
```

sin necesariamente destruir:

- Passkeys
- OIDC
- other methods

## 37. Password Removal

Una identidad podrá eliminar su password si policy lo permite.
Ejemplo:

```text
Password
Passkey
Passkey
→
Passkey
Passkey
```

## 38. Passwordless Migration

VoltStack deberá permitir migraciones progresivas:

```text
PASSWORD ONLY
      ↓
PASSWORD + PASSKEY
      ↓
PASSKEY PREFERRED
      ↓
PASSKEY ONLY
```

## 39. Passkey Binding

El documento 16 define WebAuthn.

```text
Aquí se administra su relación con Identity.
Identity
   ↓
Add Passkey
   ↓
Fresh Authentication
   ↓
WebAuthn Registration
   ↓
Credential Verification
   ↓
Binding
```

## 40. Multiple Passkeys

Debe soportarse como comportamiento normal.
Passkey #1 — Laptop
Passkey #2 — Phone
Passkey #3 — Security Key

## 41. Passkey Naming

Metadata opcional:

- "MacBook personal"
- "iPhone"
- "YubiKey oficina"

No forma parte de la identidad criptográfica.

## 42. Passkey Removal

Eliminar una passkey deberá:

- verify authority
- check last-method policy
- revoke credential

increment relevant security state
audit
notify where required

## 43. Passkey Compromise

Una passkey comprometida podrá revocarse individualmente.

## 44. TOTP Binding

Flujo:

```text
Generate Secret
     ↓
Display Provisioning Data
     ↓
User Configures Authenticator
     ↓
Verify TOTP
     ↓
Bind Method
```

Nunca activar TOTP antes de verificar un código válido.

## 45. Pending TOTP Secret

El secret provisional deberá tener lifetime limitado.

## 46. TOTP Replacement

Debe tratarse como operación sensible.

## 47. Recovery Codes

Recovery codes pertenecen al ecosistema de Authentication Methods, pero tienen semántica especial.
one-time
high sensitivity
recovery-oriented

## 48. Recovery Code Regeneration

Generar nuevos códigos deberá invalidar los anteriores según policy.

## 49. External Identity Linking

Uno de los problemas centrales:

```text
VoltStack Identity
       │
       ▼
```

Google / Microsoft / GitHub / Enterprise IdP

## 50. ExternalIdentityReference

final readonly class ExternalIdentityReference
{
public function __construct(
public string $provider,
public string $issuer,
public string $subject,
) {}
}

## 51. External Identity Key

El binding deberá utilizar preferentemente:
issuer + subject
No:

- email
- como identidad externa primaria.

## 52. Email no es External Identity

Nunca asumir:

```text
same email
=

same external identity
```

## 53. Email Collision

Ejemplo:

```php
Local Account:
<francisco@example.com>
```

Google:

```php
<francisco@example.com>
No deberá producir linking automático por defecto.
```

## 54. Account Takeover clásico

El patrón peligroso:

```php
Attacker controls external provider account
            ↓
provider returns <victim@example.com>
            ↓
application matches email
            ↓
```

attacker owns victim account
deberá impedirse arquitectónicamente.

## 55. Explicit Linking

Por default:

```text
Existing authenticated identity
       +
External provider proof
       =
Possible link
```

No:

```text
Matching email
=

automatic link
```

## 56. External Provider Linking Flow

Authenticated User
↓
"Link Google"
↓
Fresh Authentication
↓
Generate State + PKCE/Nonce
↓
External Authentication
↓
Validate Provider Response
↓
Resolve issuer + subject
↓
Check Existing Bindings
↓
Binding Policy
↓
Link

## 57. External Account Already Linked

Si:
Google issuer+subject
ya pertenece a:
Identity B
y el usuario intenta vincularlo a:
Identity A
default:
REJECT

## 58. Unique External Binding

Invariant:
(provider, issuer, subject)
solo podrá estar vinculado a una Identity dentro del realm correspondiente, salvo que una política explícita defina otra semántica.

## 59. Database Constraint

La unicidad deberá reforzarse también a nivel de persistencia cuando sea posible.

## 60. Race Condition

Dos requests simultáneos no deberán poder vincular la misma external identity a dos cuentas.

## 61. Atomic Linking

La operación deberá ser transaccional.

## 62. Provider Subject Mutation

No asumir que:

- display name
- email
- avatar
- username
- son identificadores permanentes.

## 63. Provider Issuer

El issuer forma parte del trust boundary.

## 64. Provider Aliases

No mezclar automáticamente:

- Google Workspace A
- Google Workspace B

si la política los trata como realms distintos.

## 65. Social Login

Social login podrá funcionar como:
Authentication Method
pero el proveedor externo no se convierte en dueño de la Identity interna.

## 66. Enterprise SSO

Una organización puede imponer:

- Enterprise SSO
- como método obligatorio.

## 67. Managed Identity

Ejemplo:
Tenant: Acme

User Identity:
user-42

Authentication Method:
Acme Enterprise OIDC

## 68. Tenant-Controlled Methods

Un tenant podrá controlar:

- required provider
- allowed providers
- password allowed
- passkeys allowed
- MFA requirements
- external linking allowed

sin reducir el security floor de plataforma.

## 69. SSO Enforcement

Policy:
Enterprise SSO required
podrá deshabilitar password login sin necesariamente borrar inmediatamente la credencial.

## 70. Disabled vs Removed

Debe distinguirse:
DISABLED FOR AUTHENTICATION
de:
CREDENTIAL REMOVED

## 71. Authentication Method Policy

interface AuthenticationMethodPolicyInterface
{
public function methodsAvailableFor(
Identity $identity,
AuthenticationContext $context
): AuthenticationMethodPolicyResult;
}

## 72. Policy Result

Puede indicar:

- allowed
- required
- preferred
- disabled
- deprecated
- forbidden

## 73. Preferred Method

VoltStack podrá determinar:

- Passkey preferred
- Password fallback

## 74. Preferred != Required

Importante:

- preferred
- ≠
- mandatory

## 75. Method Deprecation

Framework deberá permitir:

```text
ACTIVE
   ↓
DEPRECATED
   ↓
DISABLED
   ↓
REMOVED
```

a nivel de policy.

## 76. Migration Campaigns

Esto permitirá migrar organizaciones:

```text
Password
   ↓
Password + Passkey
   ↓
Passkey preferred
   ↓
Passkey required
```

## 77. Last Authentication Method Problem

Una de las reglas críticas.

```text
Ejemplo:
Identity
└── Passkey #1
```

usuario solicita eliminar Passkey #1.

```text
Sin controles:
Identity
└── no authentication method
```

## 78. Authentication Survivability

Antes de eliminar un método:

- Will identity remain authentically recoverable?
- deberá evaluarse.

## 79. LastMethodPolicy

interface LastAuthenticationMethodPolicyInterface
{
public function canRemove(
Identity $identity,
AuthenticationMethodInterface $method
): LastMethodDecision;
}

## 80. Política default

Para identidades humanas normales:

- Do not remove last viable authentication method
- salvo operación administrativa/recovery explícita.

## 81. Viable Method

No basta contar filas.

```php
Ejemplo:
Password = disabled
Passkey = revoked
Google = active
```

Solo existe:
1 viable method

## 82. Viability Resolver

interface AuthenticationMethodViabilityResolverInterface
{
public function resolve(
AuthenticationMethodInterface $method,
AuthenticationPolicyContext $context
): AuthenticationMethodViability;
}

## 83. Recovery Method

Recovery codes no necesariamente cuentan como método primario viable.
La policy decide.

## 84. Administrative Identity

Para cuentas administrativas puede exigirse:
minimum 2 independent methods

## 85. High-Value Identity

Puede exigirse:

- 2 passkeys
- +
- recovery method

## 86. Method Diversity

Policy podrá requerir diversidad:
Password + TOTP
no necesariamente ofrece las mismas propiedades que:
2 hardware-backed passkeys

## 87. Authentication Method Assurance

Cada método deberá describir sus propiedades de assurance.

## 88. AuthenticationMethodAssurance

final readonly class AuthenticationMethodAssurance
{
public function __construct(
public bool $phishingResistant,
public bool $hardwareBacked,
public bool $possessionBased,
public bool $knowledgeBased,
public bool $federated,
public bool $recoverable,
) {}
}

## 89. Assurance no es simple score

Evitar:

```php
password = 1
totp = 2
passkey = 3
como único modelo.
Las propiedades importan.
```

## 90. Independent Factors

Documento 15.
Method Manager deberá conocer si dos métodos representan factores independientes.

## 91. Same Provider

Ejemplo:

- Google OIDC
- +
- Google-managed passkey

puede no representar independencia equivalente a dos trust roots diferentes.
Policy podrá analizarlo.

## 92. Method Enrollment

Enrollment será el proceso de incorporación.
Authentication Method Enrollment
puede ser alias conceptual de Binding Ceremony para UX.

## 93. Enrollment State

REQUESTED
CHALLENGE_CREATED
PROOF_RECEIVED
VERIFIED
BOUND

## 94. Pending Enrollment

Nunca deberá autenticar.

## 95. Abandoned Enrollment

Debe expirar y limpiarse.

## 96. Enrollment Cleanup

Worker periódico podrá eliminar:

- expired challenges
- temporary secrets
- abandoned transactions

## 97. Authentication Method Removal

Eliminar un método también es una ceremonia.

```text
Request Removal
      ↓
Fresh Authentication
      ↓
Risk Evaluation
      ↓
Last Method Check
      ↓
Policy
      ↓
Revoke
      ↓
```

Remove / Retain Tombstone

## 98. RemovalRequest

final readonly class AuthenticationMethodRemovalRequest
{
public function__construct(
public string $identityId,
public AuthenticationMethodId $methodId,
public AuthenticationContext $authentication,
) {}
}

## 99. Soft Revocation

Para seguridad/audit puede preferirse:

```text
ACTIVE
 ↓
REVOKED
```

antes de borrar físicamente.

## 100. Credential Tombstone

Puede conservarse metadata mínima:

- credential fingerprint
- revokedAt
- reason
- method type

sin conservar secret innecesario.

## 101. Reuse Prevention

Una credential revocada no deberá poder vincularse accidentalmente de nuevo si la política lo prohíbe.

## 102. Credential Rebinding

Debe ser operación explícita.

## 103. Removal Reason

enum AuthenticationMethodRemovalReason: string
{
case UserRequested = 'user_requested';

case Replaced = 'replaced';

case Compromised = 'compromised';

case AdministratorRevoked = 'administrator_revoked';

case PolicyViolation = 'policy_violation';

case ProviderDisconnected = 'provider_disconnected';

case IdentityMerged = 'identity_merged';
}

## 104. Method Suspension

Una credencial puede suspenderse temporalmente.

- Ejemplo:
- suspicious passkey
- sin eliminarla inmediatamente.

## 105. Method Reactivation

Deberá requerir policy específica.

- No:
- toggle active=true
- sin verificación.

## 106. Authentication Method Replacement

Ejemplos:

```text
old TOTP
→ new TOTP

old password
→ new password

old certificate
→ new certificate
```

## 107. Replacement Transaction

Old Method
↓
Verify Authority
↓
Enroll New
↓
Verify New
↓
Activate New
↓
Revoke Old

## 108. Safe Replacement

Nunca:

```text
revoke old
↓
attempt new
↓
new fails
```

si eso deja al usuario bloqueado.

## 109. Overlap

Algunos métodos podrán tener breve overlap durante reemplazo.
Policy decide.

## 110. Session Consequences

Cambiar métodos puede afectar sesiones existentes.

## 111. Security State Integration

Documento 10.

- Operaciones podrán incrementar:
- authenticationSecurityEpoch
- credentialVersion
- sessionVersion
- según alcance.

## 112. Password Change Consequence

Podrá:

- invalidate other sessions
- pero conservar sesión actual después de reauthentication, según policy.

## 113. Passkey Removal Consequence

Puede invalidar sesiones asociadas al credential comprometido.

## 114. Provider Unlink Consequence

Puede invalidar:

- sessions established through that provider
- si policy lo exige.

## 115. Method Provenance

Authentication Context deberá poder indicar:

- which authentication method
- which credential
- which provider
- originó la sesión.

## 116. Session Provenance

Ejemplo:
Session S1
authenticated_via:
passkey-7

## 117. Credential Revocation Cascade

Passkey 7 compromised
↓
revoke Passkey 7
↓
find dependent sessions
↓
invalidate / step-up
según policy.

## 118. Account Linking

Account Linking deberá diferenciarse de Credential Binding.

- Credential Binding:
- Identity A
- +
- new credential

Account Linking:

- Identity A
- +
- Identity B

## 119. Identity Linking

Puede significar:

- associate identities
- sin necesariamente fusionarlas.

## 120. Identity Merge

Es una operación más fuerte:

```text
Identity A
+
Identity B
       ↓
Canonical Identity C/A/B
```

## 121. Terminología VoltStack

Se recomienda:

```text
Credential Binding
→ vincular método/credencial a una identidad

External Account Linking
→ vincular identidad de proveedor externo

Identity Association
→ relacionar dos identidades manteniéndolas separadas

Identity Merge
→ consolidar identidades
```

## 122. No Ambiguous link()

Core deberá evitar una API demasiado genérica:

```php
$linker->link($a, $b);
sin indicar semántica.
```

## 123. IdentityAssociation

final readonly class IdentityAssociation
{
public function __construct(
public string $sourceIdentityId,
public string $targetIdentityId,
public IdentityAssociationType $type,
) {}
}

## 124. Association Types

enum IdentityAssociationType: string
{
case SamePerson = 'same_person';

case Delegated = 'delegated';

case ParentChild = 'parent_child';

case ManagedIdentity = 'managed_identity';

case Migration = 'migration';
}
Estas asociaciones no otorgan permisos automáticamente.

## 125. Association != Authentication

Saber que:
Identity A related to Identity B
no significa:
A may authenticate as B

## 126. Association != Authorization

Tampoco otorga permisos automáticamente.

## 127. Identity Merge

Necesario para resolver:

- duplicate accounts
- migration duplicates
- provider-created duplicates
- legacy imports

## 128. Merge es Security-Critical

Puede transferir:

- credentials
- sessions
- resources
- roles
- tenant memberships
- recovery methods
- external identities
- audit associations

Por tanto no deberá ser simple database merge.

## 129. IdentityMergeManager

interface IdentityMergeManagerInterface
{
public function prepare(
IdentityMergeRequest $request
): IdentityMergePlan;

public function execute(
IdentityMergePlan $plan
): IdentityMergeResult;
}

## 130. Two-Phase Merge

Preferible:

```text
PREPARE
  ↓
VALIDATE
  ↓
REVIEW PLAN
  ↓
EXECUTE
```

## 131. Merge Plan

Debe identificar:

- source identity
- target identity
- credentials
- external bindings
- sessions
- tenant memberships
- conflicts
- security consequences

## 132. Canonical Identity

Una identidad deberá quedar como canonical.

```text
Identity A → canonical
Identity B → merged tombstone
```

## 133. Merged Identity Tombstone

No reutilizar inmediatamente ID antiguo.

## 134. Redirect

Podrá existir:

```text
old identity ID
→ canonical identity ID
```

para referencias históricas controladas.

## 135. Merge Authentication Requirement

Fusionar cuentas deberá exigir autenticación fuerte.
Idealmente demostrar control de ambas identidades.

## 136. Prove Both Accounts

Flujo:

```text
Authenticate Identity A
        +
Authenticate Identity B
        ↓
Merge Eligibility
```

## 137. Administrative Merge

Administradores podrán iniciar merges bajo policy especial, audit y posiblemente dual control.

## 138. Merge Conflict

Ejemplo:

```text
Identity A:
Google subject X
```

Identity B:

- Google subject Y
- No necesariamente conflicto.

Pero:

- same credential ID
- with inconsistent ownership
- sí.

## 139. Tenant Conflict

Identity A → Tenant A
Identity B → Tenant B
no deberá fusionarse ciegamente.

## 140. Realm Conflict

Mismo principio.

## 141. Security State Conflict

Si una identidad está:
COMPROMISED
y otra:

- ACTIVE
- el merge no deberá simplemente escoger ACTIVE.

## 142. Conservative Merge

Default:

- most restrictive relevant security state wins
- hasta revisión/policy.

## 143. Credential Conflict Resolver

interface CredentialMergeConflictResolverInterface
{
public function resolve(
CredentialMergeConflict $conflict
): CredentialMergeResolution;
}

## 144. Session Merge

Por default, no migrar sesiones existentes como autenticadas a la identidad fusionada.

## 145. Post-Merge Reauthentication

Preferir:

```text
merge
↓
invalidate relevant sessions
↓
fresh authentication
```

## 146. Merge Rollback

Debe analizarse cuidadosamente.
Una vez transferidos recursos y revocadas credenciales puede no ser completamente reversible.

## 147. Merge Audit

Debe registrar plan y resultado.

## 148. Account Split

No deberá asumirse que merge siempre puede revertirse mediante split.
Si se implementa, será operación administrativa independiente.

## 149. Automatic Account Merge

Prohibido por default.

- Especialmente basado únicamente en:
- matching email
- matching name
- matching phone

## 150. Identity Deduplication

Podrá sugerir posibles duplicados.
No fusionarlos automáticamente.

## 151. Identity Linking Risk Engine

Documento 20.

- Binding/linking deberá enviar señales:
- new device
- new country
- new ASN
- recent password reset
- recent account recovery
- new external provider
- high-value identity
- unusual linking sequence

## 152. Post-Recovery Cooldown

Después de account recovery, policy podrá impedir temporalmente:

- add new passkey
- link new external IdP
- remove existing MFA
- change recovery method

## 153. Security Cooldown

final readonly class AuthenticationMethodSecurityCooldown
{
public function __construct(
public DateTimeImmutable $until,
public string $reason,
) {}
}

## 154. Recovery + Linking Attack

Patrón:

```text
Attacker recovers account
      ↓
adds own passkey
      ↓
removes victim methods
      ↓
permanent takeover
```

VoltStack deberá contemplarlo explícitamente.

## 155. Defensive Sequence

Policy puede imponer:

```text
Recovery
   ↓
Restricted Security State
   ↓
Cooldown
   ↓
Existing owner notification
   ↓
```

Only then credential changes

## 156. Method Removal after Recovery

Puede requerir assurance adicional o delay.

## 157. Notifications

Cambios críticos deberían poder generar:

- New authentication method added
- Passkey removed
- Password changed
- External account linked
- External account unlinked
- Recovery method regenerated
- Accounts merged

## 158. Notification != Audit

Son sistemas diferentes.
Audit es obligatorio según policy.
Notification es comunicación al principal.

## 159. Security Notification Event

Eventos podrán alimentar un Notification subsystem futuro.

## 160. Authentication Method Discovery

UI podrá consultar:

- ¿Qué métodos tiene esta identidad?
- sin recibir secretos.

## 161. AuthenticationMethodView

final readonly class AuthenticationMethodView
{
public function __construct(
public string $id,
public string $type,
public string $displayName,
public string $status,
public ?DateTimeImmutable $createdAt,
public ?DateTimeImmutable $lastUsedAt,
) {}
}

## 162. Sensitive Metadata

No exponer:

- password hash
- TOTP secret
- private keys
- recovery code hashes
- full credential internals

## 163. Partial Identifiers

Podrá mostrarse:

```php
Google — f***@example.com
Security Key — ****93A7
cuando sea apropiado.
```

## 164. Method Management UI

El core deberá ser UI-agnostic.

- Podrá alimentar:
- Blade
- VoltStack Components
- Vue
- React
- Svelte
- Mobile API
- CLI
- Admin Console

## 165. Authentication Method Ordering

La UI podrá ordenar:

- recommended
- active
- fallback
- recovery
- deprecated

## 166. Preferred Login Method

Usuario podrá elegir una preferencia si policy lo permite.

## 167. Preference != Credential

Guardar:

```php
preferred_method = passkey
no crea ni valida passkeys.
```

## 168. Authentication Method Availability

Un método registrado puede no estar disponible para una identidad.
Ejemplo:

- SMS disabled for tenant
- Password forbidden for administrators
- Enterprise SSO mandatory

## 169. Method Availability Resolver

interface AuthenticationMethodAvailabilityResolverInterface
{
public function resolve(
Identity $identity,
AuthenticationMethodType $type,
AuthenticationPolicyContext $context
): AuthenticationMethodAvailability;
}

## 170. Login Method Enumeration

Debe evitarse revelar innecesariamente métodos asociados a una cuenta antes de autenticar.

## 171. Account Enumeration

Endpoint público:

```php
"Which login methods does <francisco@example.com> have?"
puede filtrar información.
```

## 172. Privacy-Preserving Discovery

Responder genéricamente o usar flujos iniciados desde identificadores ya validados.

## 173. Provider Enumeration

También evitar:

- "This account uses Microsoft SSO"
- cuando esa información no debe ser pública.

## 174. Authentication Method Identifier

Los IDs internos deberán ser no predecibles cuando se expongan.

## 175. CSRF

Todas las operaciones stateful de method management deberán protegerse contra CSRF cuando utilicen cookies.

## 176. OAuth/OIDC State

External linking deberá utilizar state.

## 177. PKCE

Cuando corresponda, utilizar PKCE.

## 178. OIDC Nonce

Cuando corresponda, validar nonce.

## 179. Callback Confusion

Callback de:
Login with Google
no deberá confundirse con:
Link Google Account

## 180. Flow Purpose Binding

Toda transacción deberá indicar:

```php
enum ExternalAuthenticationPurpose: string
{
    case Login = 'login';

    case Link = 'link';

    case Reauthenticate = 'reauthenticate';

    case Recovery = 'recovery';
}
```

## 181. Purpose Confusion Protection

Una respuesta iniciada para:
LOGIN
no podrá consumirse como:
LINK

## 182. Callback Transaction ID

El callback deberá resolver una transacción previamente iniciada.

## 183. One-Time Transaction

Consumida una vez:
cannot replay

## 184. Concurrent Linking

Dos linking ceremonies para el mismo provider deberán manejarse de forma segura.

## 185. Stale Browser Tab

Una pestaña antigua no deberá sobrescribir silenciosamente cambios recientes.

## 186. Optimistic Concurrency

Method collections podrán tener:
version

## 187. AuthenticationMethodSetVersion

final readonly class AuthenticationMethodSetVersion
{
public function __construct(
public int $value,
) {}
}

## 188. Compare-and-Swap

Operaciones críticas podrán exigir:
expected version = current version

## 189. Security Epoch

Cambios relevantes incrementarán un epoch/version central.

## 190. Authentication Method Set

final readonly class AuthenticationMethodSet
{
public function __construct(
public string $identityId,
public AuthenticationMethodSetVersion $version,
public AuthenticationMethodCollection $methods,
) {}
}

## 191. Atomic Mutation

Binding/removal/replacement deberá mutar el set atómicamente.

## 192. Database Transaction

Cuando persistence lo permita:

- BEGIN
- validate current state
- reserve credential
- mutate methods
- increment version
- write audit outbox
- COMMIT

## 193. Event Outbox

Para eventos críticos podrá utilizarse outbox transaccional.

## 194. Eventual Events

No permitir:

- credential bound
- database commit
- event silently lost

si evento es necesario para seguridad distribuida.

## 195. Events

Documento 23.

```text
Eventos mínimos:
AuthenticationMethodEnrollmentStarted
AuthenticationMethodEnrollmentCompleted
AuthenticationMethodEnrollmentFailed

AuthenticationMethodBound
AuthenticationMethodRemoved
AuthenticationMethodRevoked
AuthenticationMethodSuspended
AuthenticationMethodReactivated
AuthenticationMethodReplaced

ExternalIdentityLinkStarted
ExternalIdentityLinked
ExternalIdentityLinkFailed
ExternalIdentityUnlinked

IdentityAssociationCreated
IdentityAssociationRemoved

IdentityMergePrepared
IdentityMergeCompleted
IdentityMergeFailed

AuthenticationMethodSetChanged
```

## 196. Password Events

Podrán especializarse:

- PasswordBound
- PasswordChanged
- PasswordRemoved

## 197. Passkey Events

PasskeyBound
PasskeyRenamed
PasskeyRevoked
PasskeyRemoved

## 198. External Provider Events

ExternalAccountLinked
ExternalAccountUnlinked
ExternalProviderBindingRevoked

## 199. Event Payload Security

Nunca incluir:

- password
- password hash
- TOTP secret
- recovery code
- private key
- OAuth token

OIDC ID token completo

## 200. Audit

Documento 24.

- Toda mutación deberá registrar:
- identity
- actor
- initiator
- method
- credential reference
- provider
- tenant
- realm
- operation
- authentication assurance
- risk result
- timestamp
- source context
- outcome
- reason

## 201. Audit Actor

Debe distinguir:

- User self-service
- Administrator
- Recovery system
- Automated security system
- Migration tool

## 202. Administrative Credential Binding

Un administrador no debería poder agregar silenciosamente su propia passkey a la cuenta de otro usuario.

## 203. Admin Binding Policy

Default:
administrator may initiate recovery/reset
pero no:
administrator becomes authentication method owner

## 204. Support Staff

Nunca necesitarán conocer password del usuario.

## 205. Temporary Credentials

Soporte podrá emitir:

- temporary recovery credential
- si policy lo permite.

Deberá:

- expire
- be one-time
- require replacement
- be audited

## 206. Break-Glass Integration

Documento 32.
Identidades privilegiadas pueden requerir reglas especiales para method management.

## 207. Privileged Method Removal

Eliminar última passkey hardware-backed de administrador puede requerir:

- second administrator
- approval
- break-glass process

## 208. Separation of Duties

Puede integrarse con Authorization 24_APPROVAL_WORKFLOW....
Authentication solicita aprobación; Authorization/workflow determina quién puede concederla.

## 209. Machine Identity Integration

Documento 33.

```text
Machine identities también pueden tener múltiples credentials:
Service Identity
├── Certificate A
├── Certificate B
├── Signing Key
└── Workload Federation
```

## 210. Machine Credential Binding

El mismo dominio general podrá administrar:

- bind certificate
- rotate signing key

attach workload identity provider
remove API key

## 211. Human vs Machine Policy

No utilizar exactamente las mismas ceremonias.

```text
Human:
fresh reauthentication
```

Machine:

- administrative authority
- workload attestation
- key proof-of-possession

## 212. Generic Method Ownership

Principal
↓
Authentication Method
permite compartir abstracciones.

## 213. Principal-Specific Binding Policy

interface PrincipalAuthenticationMethodPolicyInterface
{
public function forPrincipal(
Principal $principal
): AuthenticationMethodPolicy;
}

## 214. Multi-Tenant

Documento 29.
Authentication Method deberá pertenecer al contexto correcto.

## 215. Global Identity / Tenant Method

Si VoltStack permite identidad global:

```text
Global Identity
   │
   ├── Global Passkey
   └── Tenant A Enterprise SSO
deberá diferenciar scopes.
```

## 216. MethodScope

enum AuthenticationMethodScope: string
{
case Global = 'global';

case Tenant = 'tenant';

case Realm = 'realm';
}

## 217. Tenant-Bound SSO

Un SSO de Tenant A no deberá autenticar automáticamente al usuario dentro de Tenant B.

## 218. Cross-Tenant Linking

Debe prohibirse por default salvo arquitectura explícita.

## 219. Tenant Deletion

Debe definir qué ocurre con métodos tenant-bound.

## 220. Tenant SSO Provider Removal

Puede provocar:

- method disabled
- para todos los usuarios vinculados.

## 221. Safe Provider Removal

Antes de eliminar provider:

- identify affected identities
- identify identities with no fallback
- migration plan
- notification
- policy validation

## 222. Provider Outage

No equivale a eliminar method.
provider unavailable
es estado operativo.

## 223. Provider Disabled

Puede ser estado administrativo.

## 224. Provider Compromised

Puede requerir revocar todos los bindings relacionados.

## 225. Provider Security Epoch

Podrá existir:

- providerSecurityEpoch
- para invalidación masiva.

## 226. Federated Subject Reassignment

Si proveedor externo reasigna identificadores incorrectamente, VoltStack deberá depender de contracts de issuer/subject y políticas de trust, no de emails.

## 227. Provider Trust Change

Cambios de issuer/trust no deberán modificar bindings silenciosamente.

## 228. Provider Migration

Ejemplo:

```text
Old Corporate IdP
      ↓
New Corporate IdP
requiere migration plan.
```

## 229. Bulk Method Migration

VoltStack deberá soportar operaciones administrativas masivas de forma segura.

## 230. Migration != Login

Herramientas de migración no deberán fingir autenticaciones de usuarios.

## 231. Imported Credentials

Credenciales importadas deberán indicar provenance.

## 232. CredentialProvenance

enum CredentialProvenance: string
{
case UserEnrolled = 'user_enrolled';

case AdministratorIssued = 'administrator_issued';

case Migrated = 'migrated';

case Federated = 'federated';

case SystemGenerated = 'system_generated';
}

## 233. Imported Password Hashes

Podrán soportarse mediante hashing migration del documento 11.

## 234. Imported Credential Trust

Una credential migrada puede tener assurance diferente hasta ser verificada.

## 235. First-Use Upgrade

Ejemplo:

```text
Legacy Password Hash
       ↓
Successful Login
       ↓
Rehash
       ↓
Modern Password Method
```

## 236. Credential Binding Timestamp

Registrar cuándo fue vinculada.

## 237. Last Verification

Separar:
lastUsedAt
de:
lastVerifiedAt

## 238. Last Used

Útil para mostrar:
"Esta passkey no se utiliza desde hace 14 meses."

## 239. Dormant Method

Policy podrá marcar credenciales abandonadas.

## 240. Automatic Removal

No eliminar automáticamente métodos únicamente por no uso sin policy explícita.

## 241. Credential Expiration

Algunos métodos expiran:

- certificates
- temporary credentials
- some enterprise credentials

otros normalmente no:

- passkey public credential
- aunque pueden revocarse.

## 242. Method Expiration

El modelo deberá soportar:

```php
public ?DateTimeImmutable $expiresAt;
sin exigirlo.
```

## 243. Authentication Method Lifecycle

Modelo general:

```text
                    ┌───────────┐
                    │  PENDING  │
                    └─────┬─────┘
                          │
                          ▼
                    ┌───────────┐
                    │  ACTIVE   │
                    └─────┬─────┘
                          │
          ┌───────────────┼──────────────┐
          ▼               ▼              ▼
      SUSPENDED      COMPROMISED      REVOKED
          │                              │
          ▼                              ▼
       ACTIVE                         RETIRED
```

## 244. State Machine

Transiciones deberán validarse.

```text
No:
REVOKED → ACTIVE
arbitrariamente.
```

## 245. MethodStateMachine

interface AuthenticationMethodStateMachineInterface
{
public function transition(
AuthenticationMethodInterface $method,
AuthenticationMethodStatus $target
): AuthenticationMethodTransitionResult;
}

## 246. Compromised → Active

Generalmente requerirá reemplazo, no simple reactivación.

## 247. Revoked Credential

Deberá considerarse terminal salvo policy excepcional.

## 248. Authentication Method Inventory

Usuario y administradores autorizados podrán consultar inventario.

## 249. Inventory deberá responder

¿Qué métodos existen?

¿Cuáles están activos?

¿Cuándo fueron agregados?

¿Cuándo se usaron?

¿Cuál autenticó esta sesión?

¿Cuáles son recovery methods?

¿Cuáles son tenant-bound?

¿Cuáles están comprometidos?

¿Cuáles están próximos a expirar?

## 250. Credential Exposure

Inventory nunca devolverá secrets.

## 251. Method Fingerprint

Podrá existir fingerprint no secreto.

## 252. Duplicate Credential Detection

Podrá prevenir:

- same passkey credential
- vinculada dos veces cuando no sea válido.

## 253. Duplicate Passkey

Credential ID deberá ser único según WebAuthn domain model.

## 254. Duplicate Certificate

Certificate/key fingerprint puede utilizarse para detectar duplicados según policy.

## 255. Duplicate External Identity

Issuer + subject unique.

## 256. Password Duplicate

No tiene sentido buscar passwords iguales globalmente.
Nunca realizar comparaciones entre hashes de usuarios para linking.

## 257. Authentication Method Limits

Policy podrá establecer:

- max passkeys = 20
- max external providers = 10
- para prevenir abuso.

## 258. Limit no debe impedir recuperación

Si usuario alcanza límite, debe existir workflow seguro para reemplazo.

## 259. Naming Collision

Dos passkeys pueden tener mismo display name.
No utilizar nombre como identificador.

## 260. Authentication Method Manager API

Ejemplo conceptual:

```php
$methods = Auth::identity($identity)
    ->methods();
```

## 261. Binding API

Auth::identity($identity)
->methods()
->bind($credential);
La API pública real deberá utilizar ceremonias seguras, no aceptar secretos arbitrariamente.

## 262. Safer API

Preferible:

```php
$binding = Auth::methods()
    ->for($identity)
    ->begin(Passkey::class);
```

## 263. Completion

$result = $binding->complete($proof);

## 264. External Linking API

$link = Auth::identity($identity)
->external()
->beginLink('google');

## 265. Removal API

Auth::identity($identity)
->methods()
->remove($methodId);
internamente ejecutará requirements y policies.

## 266. No Direct Repository Mutation

Aplicaciones no deberían:

```php
$credentialRepository->delete($id);
saltándose el Manager.
```

## 267. Repository Visibility

Repositorios podrán ser internal/service-level.

## 268. Application Service Boundary

Mutaciones de seguridad deberán atravesar servicios de dominio/aplicación.

## 269. AuthenticationMethodRepository

interface AuthenticationMethodRepositoryInterface
{
public function find(
AuthenticationMethodId $id
): ?AuthenticationMethodInterface;

public function forIdentity(
string $identityId
): AuthenticationMethodCollection;
}

## 270. Repository no decide Policy

Solo persistence.

## 271. Binding Handler

Cada tipo podrá proporcionar:

```php
interface AuthenticationMethodBindingHandlerInterface
{
    public function begin(
        AuthenticationMethodBindingContext $context
    ): AuthenticationMethodBindingChallenge;

    public function verify(
        AuthenticationMethodBindingProofInterface $proof
    ): VerifiedAuthenticationMethod;
}
```

## 272. Removal Handler

interface AuthenticationMethodRemovalHandlerInterface
{
public function revoke(
AuthenticationMethodInterface $method,
AuthenticationMethodRemovalContext $context
): void;
}

## 273. Provider Plugin

Documento 28.
Plugins podrán registrar nuevos method types.

## 274. Plugin Security Declaration

Plugin deberá declarar:

- credential storage requirements
- binding requirements
- assurance properties
- revocation behavior
- recovery behavior

## 275. Custom Method

Ejemplo:

- Smart Card
- Corporate Hardware Token
- Biometric External Provider
- Custom Enterprise Identity

## 276. Core Invariants no pueden omitirse

Un plugin no podrá evitar:

- audit
- binding ownership
- state lifecycle
- policy
- tenant isolation
- cuando sean requeridos.

## 277. Observability

Documento 24.

```text
Metrics:
auth_method_binding_started_total
auth_method_binding_completed_total
auth_method_binding_failed_total

auth_method_removed_total
auth_method_replaced_total
auth_method_revoked_total

auth_external_link_total
auth_external_unlink_total

auth_identity_merge_total
auth_identity_merge_failure_total
```

## 278. Metric Labels

method_type
provider
realm
outcome
failure_category
con cardinalidad controlada.

## 279. No Identity IDs in Metric Labels

Evitar cardinalidad y exposición.

## 280. Tracing

Spans conceptuales:

- auth.method.bind
- auth.method.verify
- auth.method.remove
- auth.external.link
- auth.identity.merge

## 281. Trace Secrets

Nunca.

## 282. Failure Taxonomy

Documento 25.

```text
AUTH_METHOD_NOT_FOUND

AUTH_METHOD_ALREADY_BOUND
AUTH_METHOD_NOT_ALLOWED
AUTH_METHOD_LIMIT_REACHED
AUTH_METHOD_DISABLED
AUTH_METHOD_REVOKED
AUTH_METHOD_COMPROMISED

AUTH_METHOD_BINDING_EXPIRED
AUTH_METHOD_BINDING_INVALID
AUTH_METHOD_BINDING_REPLAYED
AUTH_METHOD_BINDING_PROOF_INVALID

AUTH_METHOD_REAUTHENTICATION_REQUIRED
AUTH_METHOD_ASSURANCE_INSUFFICIENT

AUTH_METHOD_LAST_VIABLE_METHOD
AUTH_METHOD_MINIMUM_DIVERSITY_VIOLATION

EXTERNAL_IDENTITY_ALREADY_LINKED
EXTERNAL_IDENTITY_PROVIDER_INVALID
EXTERNAL_IDENTITY_SUBJECT_INVALID
EXTERNAL_IDENTITY_LINK_CONFLICT

IDENTITY_MERGE_NOT_ALLOWED
IDENTITY_MERGE_CONFLICT
IDENTITY_MERGE_PROOF_REQUIRED

AUTH_METHOD_SECURITY_COOLDOWN
```

## 283. Public Errors

No revelar:
"Google account belongs to user 92817"

## 284. Internal Explainability

Audit sí podrá registrar conflicto exacto según permisos.

## 285. Idempotency

Operaciones administrativas podrán aceptar:

- Idempotency-Key
- cuando corresponda.

## 286. Duplicate Completion

Completar dos veces un binding:

```text
first → success
second → reject/idempotent result
```

Nunca crear dos credenciales.

## 287. Distributed Runtime

Documento 30.

- El sistema deberá funcionar con:
- multiple HTTP nodes
- multiple regions
- queue workers
- FrankenPHP

## 288. Distributed Binding State

Binding transactions podrán requerir almacenamiento compartido.

## 289. One-Time Consumption

Challenge/state deberá consumirse atómicamente.

## 290. Cross-Node Callback

Inicio en Node A:
OAuth link
callback en Node B:

- must work
- si arquitectura distribuida lo requiere.

## 291. Cache

Method inventory podrá cachearse cuidadosamente.

## 292. Mutation Invalidation

Después de:

- bind
- remove
- revoke
- replace
- merge
- cache deberá invalidarse.

## 293. Security-Critical Reads

Para operaciones sensibles podrá requerirse lectura authoritative.

## 294. FrankenPHP

No almacenar:

```php
static $currentBinding;
static $currentIdentity;
static $methods;
```

como estado de request.

## 295. Request Isolation

Toda ceremonia deberá vivir en:

- request context
- transaction storage
- explicit state

## 296. Fiber Safety

Concurrent requests deberán mantener binding contexts independientes.

## 297. Cleanup

Después de request:

- binding context
- external provider state
- temporary proof
- method management context
- deberán liberarse.

## 298. Testing Strategy

Documento 26 deberá incorporar una suite completa.

## 299. Test — Bind Password

Identidad autenticada puede agregar password cuando policy lo permite.

## 300. Test — Weak Reauthentication

No puede agregar passkey si requirement exige assurance superior.

## 301. Test — TOTP Pending

TOTP no verificado no autentica.

## 302. Test — TOTP Confirmation

Código correcto activa method.

## 303. Test — Passkey Multiple

Una identidad puede tener varias passkeys.

## 304. Test — Duplicate Passkey

Misma credential no se vincula dos veces.

## 305. Test — External Linking

Issuer + subject se vinculan correctamente.

## 306. Test — Email Collision

Mismo email no produce linking automático.

## 307. Test — External Already Linked

External identity vinculada a B no puede vincularse a A.

## 308. Test — Linking Race

Dos identidades intentan vincular mismo external subject:
exactly one succeeds

## 309. Test — Wrong Purpose

OAuth login callback no puede completar link transaction.

## 310. Test — Replayed Callback

Rejected.

## 311. Test — Expired Binding

Rejected.

## 312. Test — CSRF State

State incorrecto:
REJECT

## 313. Test — Last Method

No permite eliminar último método viable.

## 314. Test — Disabled Method Count

Método disabled no cuenta como viable.

## 315. Test — Recovery Method Policy

Recovery codes cuentan o no según policy configurada.

## 316. Test — Minimum Diversity

Administrador no puede eliminar método requerido.

## 317. Test — Safe Replacement

Old credential permanece hasta que new credential esté validada.

## 318. Test — Compromised Credential

No puede reactivarse arbitrariamente.

## 319. Test — Session Cascade

Revocación invalida sesiones dependientes según policy.

## 320. Test — Provider Unlink

Unlink aplica consecuencias correctas.

## 321. Test — Tenant Isolation

Tenant A no modifica method tenant-bound de B.

## 322. Test — SSO Enforcement

Password no autentica cuando tenant policy lo deshabilita.

## 323. Test — Provider Outage

No elimina bindings.

## 324. Test — Provider Compromise

Security epoch invalida bindings/sessions según policy.

## 325. Test — Identity Merge

Dos identities controladas se fusionan correctamente.

## 326. Test — Merge Proof

Sin prueba requerida:
REJECT

## 327. Test — Merge Tenant Conflict

No se fusiona silenciosamente.

## 328. Test — Merge Sessions

Sesiones no se transfieren automáticamente.

## 329. Test — Merge Compromised State

No se pierde estado de seguridad restrictivo.

## 330. Test — Concurrent Mutation

Method set version previene lost updates.

## 331. Test — Distributed Binding

Inicio Node A / completion Node B funciona.

## 332. Test — One-Time Challenge

Solo un completion tiene éxito.

## 333. Test — Audit

Cada mutación genera audit correcto sin secrets.

## 334. Test — Event Outbox

Evento crítico no se pierde después de commit.

## 335. Test — Recovery Cooldown

Account recién recuperada no puede ejecutar cambios prohibidos.

## 336. Test — FrankenPHP Leakage

Identity A no hereda binding context de Identity B.

## 337. Test — Fiber Isolation

Dos ceremonies concurrentes permanecen aisladas.

## 338. Test — Machine Credential Binding

Service identity puede rotar credencial sin cambiar de identidad.

## 339. Security Invariants — Identity

AUTH-LINK-ID-01
Identity y Authentication Method son conceptos independientes.
AUTH-LINK-ID-02
Eliminar una credencial no elimina automáticamente la Identity.
AUTH-LINK-ID-03
Agregar una credencial no crea automáticamente una nueva Identity.
AUTH-LINK-ID-04
External provider identity no sustituye a la Identity interna.

## 340. Security Invariants — Binding

AUTH-LINK-BIND-01
Todo binding requiere prueba de control de la nueva credencial.
AUTH-LINK-BIND-02
Binding ceremonies tienen lifetime limitado.

- AUTH-LINK-BIND-03
- Challenges de binding son one-time.
- AUTH-LINK-BIND-04

Operaciones sensibles pueden requerir fresh authentication.
AUTH-LINK-BIND-05
Pending methods nunca autentican.

## 341. Security Invariants — External Linking

AUTH-LINK-EXT-01
Email matching nunca produce account linking automático por default.
AUTH-LINK-EXT-02
External identities se identifican mediante issuer + subject o equivalente confiable.
AUTH-LINK-EXT-03
Una external identity no pertenece simultáneamente a múltiples identities cuando policy exige unicidad.
AUTH-LINK-EXT-04
Login y linking son ceremonies distintas.
AUTH-LINK-EXT-05
Una respuesta de login no puede completar una transacción de linking.
AUTH-LINK-EXT-06
External linking es transaccional.

## 342. Security Invariants — Removal

AUTH-LINK-REM-01
Eliminar un método requiere authority suficiente.

- AUTH-LINK-REM-02
- No se elimina el último método viable salvo workflow explícito.
- AUTH-LINK-REM-03

Métodos comprometidos no se reactivan arbitrariamente.
AUTH-LINK-REM-04
Revocación puede invalidar sesiones dependientes.

## 343. Security Invariants — Merge

AUTH-LINK-MERGE-01
Identity merge nunca ocurre por coincidencia de email.

- AUTH-LINK-MERGE-02
- Merge requiere policy y pruebas apropiadas.
- AUTH-LINK-MERGE-03

Merge no transfiere sesiones ciegamente.

- AUTH-LINK-MERGE-04
- Estados de seguridad restrictivos no se pierden.
- AUTH-LINK-MERGE-05

Conflictos tenant/realm se resuelven explícitamente.

## 344. Security Invariants — Recovery

AUTH-LINK-REC-01
Recovery reciente puede restringir credential binding.

- AUTH-LINK-REC-02
- Recovery no permite automáticamente eliminar todos los métodos anteriores.
- AUTH-LINK-REC-03

Credential changes posteriores a recovery son auditables.

## 345. Security Invariants — Multi-Tenant

AUTH-LINK-TENANT-01
Tenant-bound method no se reutiliza automáticamente entre tenants.
AUTH-LINK-TENANT-02
Tenant policy puede endurecer pero no reducir platform security floor.
AUTH-LINK-TENANT-03
Cross-tenant linking requiere policy explícita.

## 346. Security Invariants — Runtime

AUTH-LINK-RT-01
Binding state nunca vive en global mutable.

- AUTH-LINK-RT-02
- FrankenPHP limpia context entre requests.
- AUTH-LINK-RT-03

Binding completion es atómico.
AUTH-LINK-RT-04
Method set mutations son concurrency-safe.

## 347. Anti-Pattern

Nunca:

```text
same email
=

same identity
```

## 348. Anti-Pattern

Google email matches
↓
automatically merge accounts

## 349. Anti-Pattern

user has session
↓
can add arbitrary passkey
sin evaluar fresh authentication.

## 350. Anti-Pattern

OAuth callback
↓
link to currently logged-in account
sin una linking transaction previa.

## 351. Anti-Pattern

Login state
=

Link state

## 352. Anti-Pattern

delete credential row
=

safe credential removal

## 353. Anti-Pattern

count(methods) > 1
=

safe to remove
sin comprobar viability.

## 354. Anti-Pattern

recovery completed
↓
attacker immediately adds passkey
↓
removes victim methods

## 355. Anti-Pattern

administrator knows user email
↓
administrator adds own credential

## 356. Anti-Pattern

Identity A + Identity B
↓
merge database rows

## 357. Anti-Pattern

merged identity
↓
all old sessions remain authenticated

## 358. Anti-Pattern

external provider unavailable
=

unlink all users

## 359. Anti-Pattern

passkey display name
=

credential identifier

## 360. Anti-Pattern

static $bindingContext
bajo FrankenPHP.

## 361. Componentes principales

AuthenticationMethod
AuthenticationMethodId
AuthenticationMethodType
AuthenticationMethodStatus
AuthenticationMethodSet

AuthenticationMethodManager
AuthenticationMethodRegistry
AuthenticationMethodRepository

AuthenticationMethodBindingTransaction
AuthenticationMethodBindingHandler
AuthenticationMethodBindingPolicy
CredentialBindingProof

AuthenticationMethodRemovalHandler
LastAuthenticationMethodPolicy
AuthenticationMethodViabilityResolver

AuthenticationMethodAssurance
AuthenticationMethodAvailabilityResolver

ExternalIdentityReference
ExternalIdentityLinkManager

IdentityAssociation
IdentityMergeManager
IdentityMergePlan
CredentialMergeConflictResolver

AuthenticationMethodStateMachine
AuthenticationMethodSecurityCooldown

## 362. Namespace sugerido

VoltStack\Quantum\Auth\Method
VoltStack\Quantum\Auth\Method\Contracts
VoltStack\Quantum\Auth\Method\Binding
VoltStack\Quantum\Auth\Method\Removal
VoltStack\Quantum\Auth\Method\Policy
VoltStack\Quantum\Auth\Method\External
VoltStack\Quantum\Auth\Method\Merge
VoltStack\Quantum\Auth\Method\Runtime
VoltStack\Quantum\Auth\Method\Events

## 363. Estructura sugerida

src/Quantum/Auth/Method/
├── Contracts/
│   ├── AuthenticationMethodInterface.php
│   ├── AuthenticationMethodManagerInterface.php
│   ├── AuthenticationMethodRegistryInterface.php
│   ├── AuthenticationMethodRepositoryInterface.php
│   ├── AuthenticationMethodBindingHandlerInterface.php
│   ├── AuthenticationMethodBindingPolicyInterface.php
│   ├── AuthenticationMethodRemovalHandlerInterface.php
│   ├── AuthenticationMethodStateMachineInterface.php
│   ├── AuthenticationMethodViabilityResolverInterface.php
│   ├── AuthenticationMethodAvailabilityResolverInterface.php
│   ├── LastAuthenticationMethodPolicyInterface.php
│   ├── IdentityMergeManagerInterface.php
│   └── CredentialMergeConflictResolverInterface.php
│
├── Model/
│   ├── AuthenticationMethod.php
│   ├── AuthenticationMethodId.php
│   ├── AuthenticationMethodType.php
│   ├── AuthenticationMethodStatus.php
│   ├── AuthenticationMethodSet.php
│   ├── AuthenticationMethodSetVersion.php
│   ├── AuthenticationMethodAssurance.php
│   └── CredentialProvenance.php
│
├── Binding/
│   ├── AuthenticationMethodBindingTransaction.php
│   ├── BindingTransactionStatus.php
│   ├── AuthenticationMethodBindingRequest.php
│   ├── AuthenticationMethodBindingResult.php
│   ├── AuthenticationMethodBindingContext.php
│   ├── AuthenticationMethodBindingChallenge.php
│   ├── CredentialBindingProof.php
│   └── BindingAuthenticationRequirement.php
│
├── Removal/
│   ├── AuthenticationMethodRemovalRequest.php
│   ├── AuthenticationMethodRemovalResult.php
│   ├── AuthenticationMethodRemovalContext.php
│   └── AuthenticationMethodRemovalReason.php
│
├── External/
│   ├── ExternalIdentityReference.php
│   ├── ExternalIdentityLinkManager.php
│   ├── ExternalIdentityLinkTransaction.php
│   ├── ExternalAuthenticationPurpose.php
│   └── ExternalIdentityBinding.php
│
├── Merge/
│   ├── IdentityAssociation.php
│   ├── IdentityAssociationType.php
│   ├── IdentityMergeRequest.php
│   ├── IdentityMergePlan.php
│   ├── IdentityMergeResult.php
│   ├── CredentialMergeConflict.php
│   └── CredentialMergeResolution.php
│
├── Policy/
│   ├── AuthenticationMethodPolicy.php
│   ├── AuthenticationMethodAvailability.php
│   ├── AuthenticationMethodViability.php
│   ├── LastAuthenticationMethodPolicy.php
│   └── AuthenticationMethodSecurityCooldown.php
│
├── Runtime/
│   ├── AuthenticationMethodManagementContext.php
│   └── AuthenticationMethodRuntimeResetter.php
│
└── Events/
├── AuthenticationMethodBound.php
├── AuthenticationMethodRemoved.php
├── AuthenticationMethodRevoked.php
├── AuthenticationMethodReplaced.php
├── ExternalIdentityLinked.php
├── ExternalIdentityUnlinked.php
├── IdentityMergePrepared.php
└── IdentityMergeCompleted.php

## 364. Flujo general de Binding

┌──────────────────────┐
│      IDENTITY        │
└──────────┬───────────┘
│
▼
Request New Auth Method
│
▼
┌──────────────────────┐
│ Binding Transaction  │
└──────────┬───────────┘
│
▼
Fresh Authentication
│
▼
Verify New Credential
│
▼
Risk Engine
│
▼
Binding Policy
│
▼
Atomic Credential Bind
│
▼
Increment Method Version
│
▼
Audit/Event
│
▼
METHOD ACTIVE

## 365. Flujo External Account Linking

IDENTITY A
│
▼
Link External Provider
│
▼
Fresh Authentication
│
▼
Create Linking Transaction
│
▼
External Provider
│
▼
Validate State / Nonce / PKCE
│
▼
Resolve issuer + subject
│
▼
Existing Binding?
│
┌──┴─────────────┐
│                │
YES              NO
│                │
REJECT        Risk/Policy
│
▼
Atomic Link
│
▼
Audit/Event

## 366. Flujo Safe Removal

Authentication Method
│
▼
Removal Request
│
▼
Fresh Authentication
│
▼
Risk Check
│
▼
Method Viability Resolver
│
▼
Last Method / Diversity Policy
│
┌──┴─────┐
│        │
DENY      ALLOW
│
▼
REVOKE
│
▼
Session Consequences
│
▼
Audit/Event

## 367. Flujo Identity Merge

Identity A               Identity B
│                         │
└────────────┬────────────┘
▼
Prove Ownership
│
▼
Merge Request
│
▼
Build Merge Plan
│
▼
Detect Security Conflicts
│
▼
Merge Policy
│
▼
Atomic Migration
│
▼
Revoke Old Sessions
│
▼
Canonical Identity
│
▼
Audit/Event

## 368. Relación con Laravel

Laravel facilita considerablemente:

- users
- password authentication
- guards
- providers
- Fortify
- Sanctum
- Socialite
- password reset

VoltStack deberá conservar esa facilidad de uso, pero formalizará un dominio que normalmente queda distribuido entre varios componentes de aplicación.
Conceptualmente Laravel suele comenzar desde:

```text
User
  ↓
credentials/providers
```

VoltStack formalizará:

```text
Principal
   ↓
Identity
   ↓
Authentication Method Set
   ├── Password
   ├── Passkeys
   ├── MFA
   ├── Recovery
   └── Federated Identities
```

La ventaja será poder evolucionar la autenticación sin convertir el modelo User en un contenedor de decenas de responsabilidades de seguridad.

## 369. Relación con Symfony

La separación de Symfony entre:

- Authenticator
- Passport
- Badge
- User Provider
- Credentials
- Security

es una referencia importante.
VoltStack añadirá un dominio explícito de gestión del ciclo de vida de métodos de autenticación, separado del proceso de autenticar un request.
Por tanto:
Authenticator
responde:
¿Cómo demuestro quién eres?
mientras:
AuthenticationMethodManager
responde:
¿Qué mecanismos están vinculados a tu identidad y cómo pueden agregarse, reemplazarse, revocarse o eliminarse?

## 370. Relación con otros documentos

08 Passport / Credential / Evidence
│
▼
11 Password Lifecycle
│
▼
15 MFA
│
▼
16 Passkeys / WebAuthn
│
▼
17 OAuth / OIDC / Social
│
▼
18 Recovery
│
▼
32 Reauthentication
│
▼
33 Machine Identity
│
▼
┌──────────────────────────────────────┐
│ 34 AUTHENTICATION METHOD MANAGEMENT  │
└──────────────────────────────────────┘
│
▼
Unified Credential / Method Lifecycle
El documento 34 no reemplaza a esos subsistemas.
Los coordina desde la perspectiva de Identity.

## 371. Diferenciador VoltStack

VoltStack combinará:

- Laravel-like Developer Experience
- +
- Symfony-like Authentication Contracts
- +
- Explicit Identity Domain
- +
- Authentication Method Registry
- +
- Credential Binding Ceremonies
- +
- Passkey Enrollment
- +
- Passwordless Migration
- +

Safe Social Account Linking
+
Enterprise SSO Binding
+
Credential Viability Policies
+
Last-Method Protection
+
Credential Replacement
+
Credential Provenance
+
Identity Association
+
Secure Identity Merge
+
Recovery Cooldowns
+
Session Provenance
+
Multi-Tenant Method Policies
+
Machine Credential Management
+
Distributed Atomic Linking
+
FrankenPHP-safe Runtime

## 372. Decisiones arquitectónicas

VoltStack adoptará:

## 01. Identity y Authentication Method serán dominios separados.

## 1. Email no será Identity.

## 2. External Provider Account no será Identity interna.

## 3. Authentication Method y Credential permanecerán conceptualmente separados.

## 4. Una Identity podrá tener múltiples métodos del mismo tipo.

## 5. Todo nuevo método requerirá una ceremonia de binding.

## 6. Binding requerirá prueba de control de la nueva credential.

## 7. Operaciones sensibles podrán exigir fresh authentication.

## 8. Pending methods nunca podrán autenticar.

## 9. Passkeys múltiples serán first-class.

## 10. Password podrá eliminarse para soportar passwordless.

## 11. External linking será siempre explícito por default.

## 12. Email matching no provocará linking automático.

## 13. External identity utilizará issuer + subject.

## 14. Login y Account Linking serán flows independientes.

## 15. Flow purpose estará criptográficamente/contextualmente vinculado.

## 16. External identity ownership tendrá unicidad transaccional.

## 17. El último método viable estará protegido.

## 18. Viability se evaluará por policy, no por conteo de registros.

## 19. Métodos comprometidos serán revocados/reemplazados, no simplemente reactivados.

## 20. Credential replacement activará primero el nuevo método cuando sea posible.

## 21. Method mutations podrán afectar sesiones existentes.

## 22. Authentication Context conservará provenance del método utilizado.

## 23. Recovery reciente podrá activar security cooldown.

## 24. Recovery no permitirá takeover permanente mediante linking inmediato.

## 25. Identity Association no implicará Authentication.

## 26. Identity Association no implicará Authorization.

## 27. Identity Merge será una operación security-critical.

## 28. Merge nunca se realizará automáticamente por email.

## 29. Merge requerirá resolución explícita de conflictos.

## 30. Sessions no se transferirán automáticamente durante merge.

## 31. Multi-tenant method bindings conservarán scope explícito.

## 32. Machine identities utilizarán el mismo dominio general cuando sea apropiado.

## 33. Plugins podrán agregar nuevos Authentication Methods mediante contratos.

## 34. Secrets jamás formarán parte del Method Inventory.

## 35. Method mutations serán concurrency-safe.

## 36. Binding challenges serán one-time y expiring.

## 37. Distributed callbacks podrán completarse en nodos distintos.

## 38. Audit será obligatorio para mutaciones security-critical.

39. FrankenPHP nunca conservará Method Management Context entre requests.
40. Criterios de aceptación

El subsistema será considerado arquitectónicamente completo cuando soporte:
41. AuthenticationMethod;
42. AuthenticationMethodType;
43. AuthenticationMethodStatus;
44. AuthenticationMethodSet;
45. Method Registry;
46. Method Manager;
47. Method Repository;
48. Binding transactions;
49. binding challenges;
50. binding proof;
51. fresh-authentication requirements;
52. binding policy;
53. method assurance properties;
54. method availability;
55. method viability;
56. password binding;
57. password replacement;
58. password removal;
59. passwordless migration;
60. multiple passkeys;
61. passkey removal;
62. TOTP enrollment;
63. recovery method management;
64. external identity references;
65. explicit external account linking;
66. issuer + subject identity;
67. linking transaction purpose;
68. state/nonce/PKCE integration;
69. external binding uniqueness;
70. account-takeover protection;
71. method removal ceremony;
72. last viable method protection;
73. method diversity policy;
74. method suspension;
75. credential replacement;
76. session provenance;
77. session invalidation cascade;
78. method security epochs/versioning;
79. recovery cooldown;
80. security notifications/events;
81. identity associations;
82. identity merge;
83. merge planning;
84. merge conflict resolution;
85. canonical identities;
86. merged identity tombstones;
87. multi-tenant method scope;
88. enterprise SSO enforcement;
89. provider compromise handling;
90. credential provenance;
91. imported credential migration;
92. method inventory;
93. method lifecycle state machine;
94. plugin extensibility;
95. audit;
96. metrics;
97. tracing;
98. failure taxonomy;
99. atomic distributed mutations;
100. FrankenPHP/fiber isolation.
101. Regla arquitectónica final

La arquitectura deberá entender una cuenta moderna así:

```text
                         PRINCIPAL
                             │
                             ▼
                          IDENTITY
                             │
                             ▼
                 AUTHENTICATION METHOD SET
                             │
       ┌────────────┬────────┼─────────┬──────────────┐
       ▼            ▼        ▼         ▼              ▼
   Password      Passkey   TOTP      OIDC       Enterprise SSO
                    │
             ┌──────┼───────┐
             ▼      ▼       ▼
          Laptop  Phone  Security Key
```

y jamás así:

- User
- =
- Email
- =
- Password
- =
- Google Account

La primera regla será:
La identidad deberá sobrevivir al cambio, rotación, eliminación o incorporación de cualquier Authentication Method.

La segunda:
Vincular una nueva credencial es una ceremonia de seguridad y nunca una simple operación de persistencia.

La tercera:
Una cuenta externa nunca se vinculará automáticamente únicamente porque comparte email con una identidad existente.

La cuarta:
Login, reauthentication, recovery y account linking serán propósitos distintos y sus pruebas no podrán intercambiarse silenciosamente.

La quinta:
VoltStack impedirá que una identidad elimine accidentalmente su último método viable de autenticación cuando la política requiera conservar acceso.

La sexta:
Una recuperación reciente no otorgará automáticamente libertad inmediata para reemplazar todos los mecanismos de seguridad de la cuenta.

La séptima:
Identity Association, Authentication y Authorization permanecerán separados.

La octava:
Identity Merge será una operación explícita, transaccional, auditable y conservadora respecto al estado de seguridad.

La novena:
El mismo modelo deberá poder administrar progresivamente passwords, passkeys, MFA, proveedores federados, SSO empresarial y credenciales de máquinas sin convertir Identity en una abstracción dependiente de un mecanismo específico.

La décima:
VoltStack tratará los Authentication Methods como recursos de seguridad con lifecycle propio: enrollment, verification, activation, usage, suspension, compromise, revocation, replacement y retirement.

Siguiente documento recomendado
La siguiente pieza que conviene formalizar es:
`35_AUTHENTICATION_SESSION_DEVICE_CREDENTIAL_INVENTORY_SECURITY_CENTER_AND_USER_SECURITY_MANAGEMENT_SYSTEM.md`
El documento 34 define cómo se administran los métodos de autenticación. El 35 puede construir sobre él el Security Center completo de la identidad, unificando:

```text
Identity Security Center
│
├── Authentication Methods
│   ├── Password
│   ├── Passkeys
│   ├── MFA
│   └── Federated Accounts
│
├── Active Sessions
│   ├── Browser
│   ├── Mobile
│   └── Other Devices
│
├── Trusted Devices
│
├── API / Personal Credentials
│
├── Recent Authentication Activity
│
├── Security Alerts
│
├── Session Revocation
│
├── Device Revocation
│
├── Credential Revocation
│
└── "Sign out everywhere"
```

Eso nos permitiría convertir todos los subsistemas anteriores en un modelo centralizado de administración de seguridad de la identidad, sin acoplarlo a una interfaz gráfica concreta.
