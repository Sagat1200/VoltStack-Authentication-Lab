# VoltStack Authentication System

## 46 — Authentication Migration, Legacy Credential Import, Backward Compatibility and Progressive Security Upgrade System

- **Archivo:** `46_AUTHENTICATION_MIGRATION_LEGACY_CREDENTIAL_IMPORT_BACKWARD_COMPATIBILITY_AND_PROGRESSIVE_SECURITY_UPGRADE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Migration / Compatibility / Credential Modernization
- **Dependencias principales:** documentos 08, 09, 10, 11, 12, 14, 15, 16, 17, 18, 23, 24, 25, 27, 29, 30, 31, 33, 34, 35, 36, 37, 39, 40, 41, 42, 44, 45.

---

## 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack podrá incorporar sistemas de autenticación existentes sin obligar a:
resetear todas las contraseñas
cerrar todas las sesiones inmediatamente
eliminar proveedores existentes
migrar todos los usuarios en una ventana de mantenimiento
mantener algoritmos inseguros indefinidamente
El sistema deberá permitir una transición controlada desde:
Legacy Authentication
        ↓
Compatibility Layer
        ↓
Progressive Security Upgrade
        ↓
Native VoltStack Authentication

## 2. Problema

La adopción de un framework de Authentication raramente ocurre sobre una base vacía.
Una aplicación puede tener millones de identidades con:
MD5
SHA-1
bcrypt
PBKDF2
Argon2 variants

Laravel password hashes
Symfony password hashes
custom hashes

legacy sessions
remember-me cookies
API keys
OAuth identities
SAML mappings
TOTP secrets
recovery codes
passkeys
client certificates
Una arquitectura empresarial no puede asumir:
"todos los usuarios empiezan desde cero"

## 3. Objetivo fundamental

VoltStack deberá poder responder:
¿Cómo trasladamos identidades y credenciales existentes hacia el nuevo modelo de seguridad sin perder acceso legítimo, sin crear ventanas de bypass y sin conservar indefinidamente la deuda de seguridad heredada?

1. Principio central
Backward compatibility es una estrategia de transición, no una propiedad permanente del Security Model.

2. Segunda regla fundamental
Legacy Support
     ≠
Legacy Trust
Que VoltStack pueda verificar una credencial heredada no significa que deba asignarle el mismo nivel de assurance que a una credencial moderna.
3. Tercera regla fundamental
Successful Legacy Authentication
             ↓
may establish identity
             ↓
but may require
             ↓
Progressive Security Upgrade

## 4. Migration System Scope

Este sistema abarcará:
Identity Import
Credential Import
Password Hash Migration
Session Migration
Remember-Me Migration
MFA Migration
Passkey Migration
Federated Identity Migration
Recovery Migration
API Credential Migration
Machine Credential Migration
Provider Migration
Identifier Migration
Tenant Migration
Realm Migration
Authentication Metadata Migration
Security State Migration
Progressive Reauthentication
Progressive Enrollment
Legacy Compatibility
Migration Observability
Rollback
Cutover
Legacy Retirement

## 5. Fuera de alcance

No sustituye:
general database migration
ETL framework
application data migration
authorization migration
profile migration
billing migration
aunque puede integrarse con ellos.

## 6. Arquitectura general

Legacy Authentication System
          │
          ▼
Legacy Source Adapter
          │
          ▼
Migration Discovery
          │
          ▼
Normalization
          │
          ▼
Security Classification
          │
          ▼
Migration Planner
          │
     ┌────┴─────┐
     ▼          ▼
Bulk Import   Lazy Migration
     │          │
     └────┬─────┘
          ▼
Compatibility Authentication
          │
          ▼
Progressive Security Upgrade
          │
          ▼
Native VoltStack Authentication
          │
          ▼
Legacy Retirement

## 7. Migration != Import

Deben separarse.
Import
→ copies/transforms data

Migration
→ changes security authority

Upgrade
→ improves security properties

Retirement
→ removes legacy dependency

## 11. Migration Domains

enum AuthenticationMigrationDomain: string
{
    case Identity = 'identity';
    case PasswordCredential = 'password_credential';
    case AuthenticationMethod = 'authentication_method';
    case Session = 'session';
    case RememberMe = 'remember_me';
    case Mfa = 'mfa';
    case Passkey = 'passkey';
    case Recovery = 'recovery';
    case Federation = 'federation';
    case ApiCredential = 'api_credential';
    case MachineCredential = 'machine_credential';
    case SecurityMetadata = 'security_metadata';
}

## 12. Migration Modes

VoltStack soportará varios modelos.
enum AuthenticationMigrationMode: string
{
    case OfflineBulk = 'offline_bulk';
    case OnlineBulk = 'online_bulk';
    case Lazy = 'lazy';
    case DualRead = 'dual_read';
    case DualWrite = 'dual_write';
    case Progressive = 'progressive';
    case Cutover = 'cutover';
}

## 13. Offline Bulk

Stop legacy writes
       ↓
Export
       ↓
Transform
       ↓
Import
       ↓
Validate
       ↓
Cutover
Adecuado para sistemas pequeños o ventanas controladas.

## 14. Online Bulk

Permite importar mientras legacy continúa operando.
Requiere resolver:
concurrent writes
versioning
change capture
conflicts
cutover point

## 15. Lazy Migration

Una estrategia especialmente útil para passwords.
User attempts login
       ↓
VoltStack credential absent
       ↓
Legacy credential lookup
       ↓
Legacy verification
       ↓
Successful
       ↓
Native credential generated
       ↓
Legacy credential retired

## 16. Ventaja de Lazy Migration

No requiere conocer el plaintext password durante migración inicial.

## 17. Dual Read

Durante transición:
Native Store
    ↓ miss
Legacy Store

## 18. Dual Read Security Rule

El fallback a legacy deberá ser explícitamente temporal.
Nunca:
try {
    return $modern->authenticate();
} catch (\Throwable) {
    return $legacy->authenticate();
}

## 19. Why

Un error moderno:
database unavailable
policy failure
security rejection
no debe convertirse accidentalmente en:
try weaker legacy authentication

## 20. Explicit Fallback Reason

Solo determinados resultados podrán activar legacy lookup.
Ejemplo:
NATIVE_IDENTITY_NOT_MIGRATED
NATIVE_CREDENTIAL_NOT_MIGRATED
No:
INVALID_PASSWORD
ACCOUNT_SUSPENDED
POLICY_DENIED

## 21. Dual Write

Más peligroso.
New credential
   ├── Native
   └── Legacy
debe minimizarse.

## 22. Security Rule

VoltStack no deberá degradar una nueva credencial moderna para poder almacenarla en legacy.

## 23. Example

Prohibido:
VoltStack Argon2id password
        ↓
also generate SHA1 legacy representation
si ello perpetúa una vulnerabilidad innecesaria.

## 24. Prefer One-Way Cutover

Después de una actualización exitosa:
Legacy → Native
y no:
Native → Legacy

## 25. Progressive Migration

Modelo recomendado para grandes sistemas.
Import identities
      ↓
Accept legacy credentials temporarily
      ↓
Upgrade on successful authentication
      ↓
Require stronger factors progressively
      ↓
Retire legacy credentials
      ↓
Disable compatibility

## 26. Migration Source

interface AuthenticationMigrationSourceInterface
{
    public function describe(): AuthenticationMigrationSourceDescriptor;

    public function read(
        AuthenticationMigrationCursor $cursor,
        AuthenticationMigrationBatchPolicy $batch
    ): AuthenticationMigrationBatch;
}

## 27. Sources

Podrán existir adapters para:
Laravel
Symfony
custom PHP applications
SQL databases
LDAP
Active Directory
CSV
JSON
SCIM exports
identity providers
legacy APIs
custom authentication services

## 28. Source Trust

Source authentication deberá clasificarse.
enum AuthenticationMigrationSourceTrust: string
{
    case Untrusted = 'untrusted';
    case Imported = 'imported';
    case Verified = 'verified';
    case Authoritative = 'authoritative';
}

## 29. Source Authentication

Remote migration source puede requerir:
mTLS
signed requests
OAuth
VPN/private network
service identity
Documento 33.

## 30. Source Integrity

Imports de archivos deberían poder validar:
checksum
signature
schema
origin
generation timestamp

## 31. Never Trust Filename

users-final-secure.csv
no es prueba de integridad.

## 32. Migration Manifest

final readonly class AuthenticationMigrationManifest
{
    public function __construct(
        public AuthenticationMigrationId $migrationId,
        public AuthenticationMigrationSourceDescriptor $source,
        public AuthenticationMigrationSchemaVersion $schemaVersion,
        public AuthenticationMigrationDatasetDigest $digest,
        public DateTimeImmutable $createdAt,
    ) {}
}

## 33. Migration Identity

Toda migración deberá poseer un ID estable.
final readonly class AuthenticationMigrationId
{
    public function __construct(
        public string $value,
    ) {}
}

## 34. Migration Lifecycle

DRAFT
  ↓
DISCOVERING
  ↓
ANALYZING
  ↓
PLANNED
  ↓
VALIDATED
  ↓
READY
  ↓
RUNNING
  ↓
VERIFYING
  ↓
CUTOVER_READY
  ↓
CUTOVER
  ↓
MONITORING
  ↓
COMPLETED
  ↓
LEGACY_RETIRED
Alternativas:
PAUSED
FAILED
ROLLED_BACK
CANCELLED

## 35. Migration State

enum AuthenticationMigrationState: string
{
    case Draft = 'draft';
    case Discovering = 'discovering';
    case Analyzing = 'analyzing';
    case Planned = 'planned';
    case Validated = 'validated';
    case Ready = 'ready';
    case Running = 'running';
    case Verifying = 'verifying';
    case CutoverReady = 'cutover_ready';
    case Cutover = 'cutover';
    case Monitoring = 'monitoring';
    case Completed = 'completed';
    case LegacyRetired = 'legacy_retired';
    case Paused = 'paused';
    case Failed = 'failed';
    case RolledBack = 'rolled_back';
    case Cancelled = 'cancelled';
}

## 36. Migration Plan

final readonly class AuthenticationMigrationPlan
{
    public function__construct(
        public AuthenticationMigrationId $migrationId,
        public AuthenticationMigrationStageSet $stages,
        public AuthenticationMigrationPolicyVersion $policyVersion,
        public AuthenticationMigrationPlanDigest $digest,
    ) {}
}

## 37. Plan Before Apply

Para migraciones críticas:
DISCOVER
   ↓
PLAN
   ↓
SIMULATE
   ↓
REVIEW
   ↓
APPROVE
   ↓
APPLY

## 38. Dry Run

Obligatorio/recomendado para grandes migraciones.

## 39. Dry Run Result

Ejemplo:
Identities discovered:           4,920,441
Duplicate identifiers:              18,242
Unsupported password hashes:         1,103
Weak password hashes:            1,844,022
Federated mappings:                720,183
Ambiguous mappings:                    912
TOTP credentials:                  340,811
Legacy sessions:                 2,010,448
Immediate reset required:            7,221

## 40. Security Classification

Cada artefacto heredado deberá clasificarse antes de importarlo.

## 41. Legacy Security Classification

enum LegacyCredentialSecurityClass: string
{
    case Acceptable = 'acceptable';
    case UpgradeRecommended = 'upgrade_recommended';
    case UpgradeRequired = 'upgrade_required';
    case Restricted = 'restricted';
    case Unsupported = 'unsupported';
    case Compromised = 'compromised';
}

## 42. Credential Import Decision

enum LegacyCredentialImportDecision: string
{
    case ImportNative = 'import_native';
    case ImportRestricted = 'import_restricted';
    case VerifyAndUpgrade = 'verify_and_upgrade';
    case RequireReenrollment = 'require_reenrollment';
    case RequireReset = 'require_reset';
    case Reject = 'reject';
}

## 43. Credential Classification Inputs

algorithm
parameters
salt quality
credential age
source integrity
known compromise
provider status
factor type
exportability
tenant policy
realm policy
platform security floor

## 44. Security Floor

Tenant puede endurecer import policy.
No puede debilitar el minimum platform floor.

## 45. Password Migration

Es uno de los problemas principales.

## 46. Password Credential Categories

Native Modern Hash
Compatible Modern Hash
Legacy Strong Hash
Legacy Weak Hash
Unsalted Hash
Unknown Hash
Plaintext
Encrypted Password
External Password Authority

## 47. Plaintext Password Import

Por defecto:
REJECT
o inmediatamente:
hash securely
destroy plaintext
audit migration anomaly
bajo un procedimiento explícitamente aprobado.

## 48. Never Persist Imported Plaintext

No:
$legacyPassword = $row['password'];
saveMigrationLog($legacyPassword);

## 49. Logs

Nunca contienen:
plaintext password
hash input
OTP
recovery code
private key
API secret

## 50. Password Hash Descriptor

final readonly class LegacyPasswordHashDescriptor
{
    public function __construct(
        public string $algorithm,
        public array $parameters,
        public bool $salted,
        public ?string $format,
    ) {}
}

## 51. Hash Identification

No depender únicamente de longitud.

## 52. Example

32 hexadecimal chars
puede parecer MD5, pero formato deberá clasificarse con evidencia suficiente.

## 53. Password Hash Adapter

interface LegacyPasswordVerifierInterface
{
    public function supports(
        LegacyPasswordHashDescriptor $descriptor
    ): bool;

    public function verify(
        SensitivePassword $password,
        LegacyPasswordCredential $credential
    ): LegacyPasswordVerificationResult;
}

## 54. Legacy Verifier Isolation

Algoritmos obsoletos deberán vivir en un componente separado.
VoltStack Native Password Hasher
            ≠
Legacy Password Verifier

## 55. Why

Evita que:
MD5
SHA1
obsolete PBKDF parameters
se conviertan accidentalmente en opciones válidas para nuevas credenciales.

## 56. Critical Invariant

Legacy verification capability nunca implica legacy credential creation capability.

1. Legacy Hash Registry
interface LegacyPasswordVerifierRegistryInterface
{
    public function resolve(
        LegacyPasswordHashDescriptor $descriptor
    ): LegacyPasswordVerifierInterface;
}
2. Native Password Policy
Documento 11 sigue siendo autoridad sobre nuevos hashes.
3. Rehash-on-Login
Flujo recomendado:
Password submitted
      ↓
Legacy hash detected
      ↓
Legacy verifier
      ↓
SUCCESS
      ↓
Native password policy
      ↓
Generate Argon2id/native hash
      ↓
Atomic credential replacement
      ↓
Legacy credential retired
4. Atomic Upgrade
El login no debería quedar en estado ambiguo:
legacy accepted
native write failed
legacy removed
5. Safe Sequence
verify legacy
     ↓
create native credential
     ↓
commit
     ↓
retire legacy credential
preferiblemente en una operación transaccional cuando storage lo permita.
6. Same Transaction
Si ambos registros comparten datastore:
BEGIN
create native
mark legacy retired
increment credential version
COMMIT
7. Different Stores
Usar state machine/outbox/reconciliation.
8. Failed Upgrade
Successful legacy authentication no necesariamente debe fallar si rehash persistence falla.
Esto dependerá de policy.
9. High-Security Realm
Puede decidir:
native upgrade persistence required
antes de emitir sesión.
10. Standard Realm
Puede:
authenticate with restricted context
schedule upgrade retry
solo cuando hacerlo sea seguro y no requiera conservar plaintext.
11. Important Limitation
Password plaintext existe únicamente durante el request.
No debe enviarse a background queue para rehash posterior.
12. Therefore
Si rehash necesita password:
do it synchronously
durante successful verification.
13. Background Password Upgrade
Solo puede manejar tareas que no necesiten plaintext:
classification
reporting
marking
notification
policy migration status
14. Progressive Security Upgrade
No se limita a passwords.
15. Upgrade Dimensions
Password Hash Strength
MFA Enrollment
Phishing Resistance
Passkey Adoption
Recovery Security
Device Trust
Federation Security
Credential Lifetime
Session Security
Machine Credential Strength
16. Upgrade Requirement
final readonly class AuthenticationSecurityUpgradeRequirement
{
    public function __construct(
        public AuthenticationSecurityUpgradeType $type,
        public AuthenticationSecurityUpgradeUrgency $urgency,
        public ?DateTimeImmutable $deadline,
        public AuthenticationSecurityUpgradeReasonSet $reasons,
    ) {}
}
17. Upgrade Urgency
enum AuthenticationSecurityUpgradeUrgency: string
{
    case Advisory = 'advisory';
    case Recommended = 'recommended';
    case RequiredSoon = 'required_soon';
    case RequiredNow = 'required_now';
    case Blocking = 'blocking';
}
18. Upgrade State
NOT_REQUIRED
RECOMMENDED
REQUIRED
IN_PROGRESS
COMPLETED
FAILED
EXEMPTED
EXPIRED
19. Security Upgrade != Incident
Una credencial antigua no significa necesariamente compromiso.
20. Security Upgrade != Authentication Failure
Un método heredado todavía permitido puede autenticar.
Pero puede producir menor assurance.
21. Integration with Assurance
Documento 37.
Ejemplo:
legacy password verified
        ↓
Identity proven at STANDARD
        ↓
cannot satisfy HIGH
        ↓
step-up required
22. Legacy Assurance Mapping
interface LegacyAuthenticationAssuranceMapperInterface
{
    public function map(
        LegacyAuthenticationEvidence $evidence,
        AuthenticationPolicyContext $context
    ): AuthenticationAssuranceAssessment;
}
23. Never Inflate Assurance
Un imported flag:
mfa_verified = true
no deberá convertirse automáticamente en permanent HIGH assurance.
24. Historical MFA
Significa que MFA ocurrió históricamente.
No prueba MFA fresca para la sesión actual.
25. Imported Security Metadata
Debe conservar provenance.
26. Provenance
final readonly class ImportedAuthenticationMetadata
{
    public function __construct(
        public AuthenticationMigrationId $migrationId,
        public AuthenticationMigrationSourceId $source,
        public DateTimeImmutable $importedAt,
        public AuthenticationMigrationSourceTrust $sourceTrust,
    ) {}
}
27. Provenance Is First-Class
Permite responder:
Where did this credential come from?
When was it imported?
Was its source authoritative?
Was it subsequently verified natively?
28. Imported != Native Verified
Esta distinción debe poder sobrevivir después de migration.
29. Credential Provenance States
IMPORTED_UNVERIFIED
IMPORTED_VERIFIED
NATIVE_CREATED
NATIVE_UPGRADED
FEDERATED_VERIFIED
ADMIN_PROVISIONED
30. Identity Migration
Identity es estable.
Documento 41.
31. Legacy User ID
No deberá convertirse necesariamente en VoltStack identity ID.
32. External Mapping
final readonly class LegacyIdentityReference
{
    public function __construct(
        public AuthenticationMigrationSourceId $source,
        public string $legacyId,
    ) {}
}
33. Identity Mapping Store
interface LegacyIdentityMappingStoreInterface
{
    public function resolve(
        LegacyIdentityReference $legacy
    ): ?IdentityId;

    public function bind(
        LegacyIdentityReference $legacy,
        IdentityId $identity
    ): void;
}
34. Mapping Uniqueness
(source, legacy_id)
deberá mapear de manera no ambigua.
35. Duplicate Identity Problem
Legacy puede contener:
User A: <john@example.com>
User B: <JOHN@example.com>
o múltiples accounts con mismo email.
36. Never Auto-Merge by Email
Documento 34.
37. Identity Conflict
Debe producir:
MIGRATION_IDENTITY_CONFLICT
para revisión/resolución.
38. Identifier Normalization
Email/usernames pueden haber usado reglas diferentes.
39. Canonicalization Version
final readonly class AuthenticationIdentifierCanonicalizationPolicy
{
    public function __construct(
        public string $version,
        public AuthenticationIdentifierNormalizationRules $rules,
    ) {}
}
40. Migration Must Preserve Original
Puede almacenar:
original identifier
canonical identifier
legacy normalization version
cuando governance lo permita.
41. Identifier Reuse
Debe considerar documentos 41 y 42.
42. Tenant Mapping
Legacy tenant IDs no se asumirán iguales a VoltStack Tenant IDs.
43. Tenant Mapping Store
interface AuthenticationMigrationTenantMapperInterface
{
    public function map(
        LegacyTenantReference $legacy
    ): TenantId;
}
44. Cross-Tenant Safety
Un error de mapping puede ser catastrófico.
Por ello:
tenant mapping
deberá validarse antes de importar credenciales.
45. Realm Mapping
Legacy:
users
admins
partners
pueden mapear a:
USER_REALM
ADMIN_REALM
PARTNER_REALM
46. Role != Realm
No inferir automáticamente realm exclusivamente por role.
47. Administrative Identities
Migración de admins requiere políticas más estrictas.
48. Example
Legacy admin with password only
puede importarse como identity, pero:
ADMIN_REALM login
→ requires MFA/passkey enrollment
49. Privileged Migration
Puede requerir:
forced reauthentication
fresh MFA enrollment
passkey enrollment
credential reset
manual verification
50. Session Migration
Sessions heredadas son especialmente delicadas.
51. Default Strategy
Preferir:
do not migrate legacy sessions
y exigir nuevo login.
52. But
Grandes sistemas pueden requerir seamless migration.
53. Legacy Session Compatibility
interface LegacySessionAuthenticatorInterface
{
    public function authenticate(
        LegacySessionCredential $credential,
        AuthenticationRequestContext $context
    ): LegacySessionAuthenticationResult;
}
54. Session Migration Risks
unknown fixation protections
weak session IDs
unknown rotation history
weak cookie settings
long lifetimes
missing security epochs
unknown MFA context
55. Therefore
Una legacy session aceptada puede producir:
restricted Authentication Context
56. Session Upgrade
Legacy Session
      ↓
Validate legacy
      ↓
Resolve identity
      ↓
Evaluate current security state
      ↓
Issue new VoltStack session
      ↓
Retire legacy session

## 57. Session Fixation

Nueva VoltStack session deberá tener nuevo session identifier.
Nunca reutilizar legacy ID.

## 58. Session Assurance

No inferir strong assurance si legacy session no preserva evidencia suficiente.

## 59. Fresh Authentication

Sensitive operations pueden requerir reauth incluso si legacy session fue aceptada.

## 60. Remember-Me Migration

Más restrictiva todavía.

## 61. Default

Legacy remember-me credential debería normalmente:
restore limited identity context
o requerir full authentication.

## 62. Never Elevate

Legacy remember-me no debe producir privileged authentication.

## 63. Token Format Compatibility

VoltStack puede implementar parsers temporales para legacy tokens.

## 64. Parser != Issuer

Mismo principio:
VoltStack puede aceptar temporalmente un formato heredado sin seguir emitiéndolo.

## 65. Read Legacy, Write Native

Regla preferida:
READ:
Legacy + Native

WRITE:
Native only

## 122. Token Compatibility Window

Debe tener:
start
deadline
retirement policy
telemetry

## 123. No Permanent Compatibility

Toda compatibility feature deberá poder responder:
When does this disappear?

## 124. Compatibility Policy

final readonly class AuthenticationCompatibilityPolicy
{
    public function __construct(
        public AuthenticationCompatibilityFeature $feature,
        public DateTimeImmutable $enabledFrom,
        public ?DateTimeImmutable $deprecatedAt,
        public ?DateTimeImmutable $disabledAt,
        public AuthenticationCompatibilitySecurityClass $securityClass,
    ) {}
}

## 125. Compatibility States

DISABLED
SHADOW
ENABLED
DEPRECATED
RESTRICTED
RETIRING
RETIRED

## 126. Shadow Mode

Muy útil antes del cutover.
Legacy login
     ↓
VoltStack verifies in shadow
     ↓
does not affect result
     ↓
compare outcomes

## 127. Shadow Security

No almacenar plaintext password.
La comparación ocurre dentro del request mientras secret está disponible.

## 128. Shadow Output

legacy accepted / VoltStack accepted
legacy accepted / VoltStack rejected
legacy rejected / VoltStack accepted
both rejected

## 129. Privacy

No registrar password ni sensitive evidence.

## 130. Shadow Disagreement

Debe analizarse antes del cutover.

## 131. Example Causes

different identifier normalization
account state differences
hash parsing
tenant mapping
lockout differences
clock skew
provider configuration

## 132. MFA Migration

Puede incluir:
TOTP
HOTP
SMS enrollment
email OTP enrollment
security keys
passkeys
recovery methods

## 133. TOTP Import

Si legacy TOTP secret puede exportarse legítimamente:
import encrypted
preserve provenance
validate parameters
classify security

## 134. TOTP Parameters

No asumir:
SHA1
6 digits
30 seconds
sin metadata.

## 135. TOTP Descriptor

final readonly class LegacyTotpDescriptor
{
    public function __construct(
        public string $algorithm,
        public int $digits,
        public int $period,
        public int $allowedSkew,
    ) {}
}

## 136. TOTP Secret Protection

Durante migration:
encrypted transport
restricted memory lifetime
encrypted storage
no logs
no analytics

## 137. Non-Exportable MFA

Si source no permite exportar:
require reenrollment

## 138. SMS/Email Enrollment

Un teléfono/email importado no necesariamente debe considerarse verified bajo nueva policy.

## 139. Imported Contact Verification

Debe conservar:
verification provenance
verified_at
verification method
source trust

## 140. Passkey Migration

Passkeys/WebAuthn credentials pueden ser migrables si existe la información pública necesaria.

## 141. WebAuthn Credential Data

Puede incluir:
credential ID
public key
sign count
transports
AAGUID
user handle mapping
RP information
backup eligibility/state

## 142. RP ID Constraint

Un WebAuthn credential está ligado al RP.

## 143. Critical Consequence

Cambiar:
login.old.example.com
a:
auth.new.example.com
puede impedir reutilizar credentials dependiendo del RP ID original.

## 144. Passkey Migration Planner

Debe evaluar RP compatibility antes de prometer seamless migration.

## 145. Sign Counter

No reiniciar arbitrariamente.

## 146. Counter Anomaly

Migration debe conservar counter semantics o establecer explícitamente baseline seguro.

## 147. Passkey Provenance

Imported credential deberá marcarse como tal hasta primera successful native verification si policy lo requiere.

## 148. Recovery Code Migration

Generalmente difícil si legacy solo almacena hashes.

## 149. Compatible Hash

Puede conservarse temporalmente si verifier compatible.

## 150. Prefer Rotation

Después de successful use o security upgrade:
invalidate legacy recovery set
generate new native set

## 151. Never Reveal Existing Recovery Codes

Migration no convierte hashes en plaintext.

## 152. Federated Identity Migration

Documento 17 y 34.

## 153. External Identity Key

Usar:
provider
issuer
subject
No solo:
email

## 154. Legacy Provider Mapping

final readonly class LegacyFederatedIdentityDescriptor
{
    public function __construct(
        public string $provider,
        public ?string $issuer,
        public string $subject,
        public AuthenticationMigrationSourceTrust $trust,
    ) {}
}

## 155. Missing Issuer

Legacy systems pueden haber almacenado:
provider = google
provider_user_id = 123
sin issuer.
Migration deberá aplicar adapter-specific canonicalization.

## 156. Never Guess Provider Identity

Ambigüedad → manual/policy resolution.

## 157. OAuth Tokens

Por defecto no deberían migrarse como Authentication authority salvo necesidad explícita.

## 158. Refresh Tokens

Son secretos de alto valor.
Preferir:
re-consent
re-authenticate provider
cuando sea posible.

## 159. Provider Client Change

Cambiar OAuth client puede invalidar assumptions sobre refresh token portability.

## 160. Federation Upgrade

Puede requerir:
relink provider
verify new issuer/client
new consent

## 161. Enterprise SSO Migration

Debe preservar:
tenant
organization
issuer
subject
domain policy
IdP metadata
certificate trust

## 162. SAML/OIDC Migration

No auto-link por email cuando cambia provider.

## 163. API Key Migration

API keys pueden ser:
plaintext stored
hashed
encrypted
prefix + hash
unknown

## 164. Plaintext Legacy API Key

Si debe conservarse temporalmente:
hash/import verifier
destroy plaintext migration copy

## 165. Preferred

Emitir nueva VoltStack API credential y establecer deprecation window.

## 166. Dual-Key Rotation

Legacy Key
    +
New VoltStack Key
pueden coexistir durante transición.

## 167. Cutover

Después:
legacy key revoked

## 168. API Key Security Class

Legacy static key puede recibir menor machine assurance.

## 169. Machine Credential Migration

Documento 33.

## 170. Machine Migration Strategy

Preferir:
create new machine identity
issue new short-lived/native credential
update workload
verify
revoke legacy secret

## 171. Shared Legacy Secrets

Deben eliminarse.
Ejemplo:
one API secret shared by 200 servers

## 172. Split Migration

Shared Secret
     ↓
Inventory Consumers
     ↓
Create Per-Service Identities
     ↓
Issue Individual Credentials
     ↓
Progressive Cutover
     ↓
Revoke Shared Secret

## 173. Machine Identity Mapping

No mapear máquinas a fake users.

## 174. Certificate Migration

Debe validar:
issuer
chain
EKU
SAN
expiration
revocation
key usage
trust domain

## 175. Private Keys

Preferiblemente no se migran fuera de HSM/KMS.

## 176. Re-Key

Mejor que exportar private key cuando sea posible.

## 177. Security State Migration

Legacy puede tener:
active
disabled
banned
locked
deleted

## 178. Never Map Blindly

Estos estados no necesariamente significan lo mismo que VoltStack lifecycle.

## 179. State Mapping

interface LegacyIdentityStateMapperInterface
{
    public function map(
        LegacyIdentityState $state,
        AuthenticationMigrationContext $context
    ): AuthenticationIdentityLifecycleMigrationDecision;
}

## 180. Example

legacy.locked = true
podría significar:
temporary brute-force lockout
administrative suspension
security freeze
manual ban
No son equivalentes.

## 181. Protection State

Documento 40.
No mezclar:
COMPROMISED
FROZEN
RECOVERY_REQUIRED
con lifecycle.

## 182. Identity Lifecycle

Documento 41.

## 183. Lockout Migration

Temporary lockouts deberían conservar:
reason
expiresAt
scope
cuando sea posible.

## 184. Expired Legacy Lock

No importar como permanent suspension.

## 185. Security Incident Migration

Historial legacy puede importarse como historical evidence.
No necesariamente como active incident.

## 186. Session Security Epoch

Legacy sessions sin epoch pueden necesitar:
compatibility epoch
o restricted migration.

## 187. Authentication Method Inventory

Documento 34/35.
Imported methods deberán aparecer en Security Center con provenance.

## 188. Example

Password
Imported from Legacy System
Upgraded: pending
Last verified: unknown
sin revelar información sensible.

## 189. User Security Communication

Documentos 40 y 43.
Migration puede requerir comunicar:
security upgrade required
password reset required
MFA reenrollment
passkey migration limitation
legacy API key retirement

## 190. Avoid Alert Fatigue

No enviar 10 notificaciones por cada sub-step de migración.

## 191. Communication Aggregation

Ejemplo:
"We've upgraded account security.
Please verify your MFA method before September 30."

## 192. No Security Secrets in Migration Emails

Aplican reglas de doc43.

## 193. Migration Authentication Policy

Documento 36.
Policy puede establecer:
legacy passwords allowed until X
legacy session accepted only for STANDARD assurance
admin legacy password requires immediate step-up
SMS MFA must migrate before Y
legacy API keys expire by Z

## 194. Progressive Enforcement

Phase 1: Observe
Phase 2: Warn
Phase 3: Step-Up
Phase 4: Require Upgrade
Phase 5: Block Legacy
Phase 6: Remove Legacy Code

## 195. Security Upgrade Phases

enum AuthenticationSecurityUpgradePhase: string
{
    case Observe = 'observe';
    case Recommend = 'recommend';
    case EnforceForSensitiveOperations = 'enforce_sensitive';
    case RequireOnAuthentication = 'require_on_authentication';
    case RejectLegacy = 'reject_legacy';
    case Retired = 'retired';
}

## 196. Canary Migration

Aplicar primero a:
internal users
test tenants
small tenant cohort
low-risk accounts
según estrategia.

## 197. High-Privilege Users

Pueden migrarse antes, no después, debido al riesgo.

## 198. But

La migración de administradores deberá ser especialmente controlada.

## 199. Tenant Rollout

Tenant A → Native
Tenant B → Compatibility
Tenant C → Shadow
puede coexistir.

## 200. Tenant Isolation

Migration state deberá ser scoped.

## 201. Migration Scope

final readonly class AuthenticationMigrationScope
{
    public function __construct(
        public AuthenticationMigrationScopeType $type,
        public ?TenantId $tenantId,
        public ?RealmId $realmId,
        public ?ApplicationId $applicationId,
        public EnvironmentId $environmentId,
    ) {}
}

## 202. Scope Types

PLATFORM
TENANT
REALM
APPLICATION
COHORT
IDENTITY_SET

## 203. No Cross-Tenant Import

Un source record nunca puede caer en tenant equivocado por default fallback.

## 204. Unknown Tenant

QUARANTINE
no:
default tenant

## 205. Migration Quarantine

Artefactos ambiguos deberán poder aislarse.

## 206. Quarantine Reasons

UNKNOWN_TENANT
UNKNOWN_REALM
DUPLICATE_IDENTITY
UNSUPPORTED_HASH
INVALID_CREDENTIAL
MISSING_PROVIDER_ISSUER
CORRUPT_RECORD
CONFLICTING_MAPPING
SECURITY_POLICY_VIOLATION

## 207. Quarantine Store

interface AuthenticationMigrationQuarantineStoreInterface
{
    public function quarantine(
        AuthenticationMigrationRecord $record,
        AuthenticationMigrationQuarantineReason $reason
    ): void;
}

## 208. Quarantine Security

Puede contener credential metadata.
Debe aplicar:
encryption
access control
retention
audit
redaction

## 209. Quarantine != Error Log

No volcar raw record a logs.

## 210. Migration Validation

Tres niveles:
Structural
Semantic
Security

## 211. Structural Validation

schema
types
required fields
encoding
length
format

## 212. Semantic Validation

identity references exist
tenant mapping valid
provider mapping valid
credential belongs to identity
state transition makes sense

## 213. Security Validation

algorithm allowed
credential not compromised
source trusted enough
tenant/realm policy satisfied
secret handling safe

## 214. Validation Pipeline

Raw Record
    ↓
Schema Validation
    ↓
Normalization
    ↓
Semantic Validation
    ↓
Security Classification
    ↓
Mapping
    ↓
Migration Decision

## 215. Validation Result

final readonly class AuthenticationMigrationValidationResult
{
    public function__construct(
        public bool $valid,
        public AuthenticationMigrationFindingSet $findings,
        public AuthenticationMigrationDecision $decision,
    ) {}
}

## 216. Findings

Severity:
INFO
WARNING
SECURITY_WARNING
ERROR
CRITICAL

## 217. Example

Password hash uses SHA-1
→ SECURITY_WARNING
→ VERIFY_AND_UPGRADE

## 218. Unsupported

Unknown proprietary hash
→ ERROR
→ REQUIRE_RESET

## 219. Compromised

Known credential breach:
→ REJECT
→ REQUIRE_RECOVERY/RESET

## 220. Credential Compromise Data

Puede integrarse con doc40.

## 221. Migration Versioning

Todo transformation logic deberá versionarse.

## 222. Mapper Version

interface AuthenticationMigrationMapperInterface
{
    public function version(): AuthenticationMigrationMapperVersion;
}

## 223. Why

Necesitamos saber:
why did legacy state X become VoltStack state Y?

## 224. Reproducibility

Migration plan deberá registrar:
source schema version
mapping version
normalization version
security policy version
framework version

## 225. Migration Snapshot

final readonly class AuthenticationMigrationExecutionSnapshot
{
    public function __construct(
        public AuthenticationMigrationId $migrationId,
        public AuthenticationMigrationPlanDigest $planDigest,
        public AuthenticationMigrationMapperVersion $mapperVersion,
        public AuthenticationMigrationPolicyVersion $policyVersion,
    ) {}
}

## 226. Runtime Config Changes

No deberán alterar silenciosamente un batch ya iniciado.

## 227. Batch Snapshot

Cada batch usa configuration snapshot.

## 228. New Batch

Puede adoptar nueva version según orchestration policy.

## 229. Batch Processing

Documento 44.
Migration Plan
     ↓
Partition
     ↓
Background Tasks
     ↓
Batches

## 230. Batch Policy

final readonly class AuthenticationMigrationBatchPolicy
{
    public function __construct(
        public int $size,
        public int $maxConcurrency,
        public DateInterval $maxExecutionTime,
    ) {}
}

## 231. Batch Idempotency

Importar mismo batch dos veces no debe duplicar identities/credentials.

## 232. Stable Import Key

Ejemplo:
migration_id + source + source_record_id

## 233. Import Record

final readonly class AuthenticationMigrationRecordId
{
    public function __construct(
        public AuthenticationMigrationId $migration,
        public AuthenticationMigrationSourceId $source,
        public string $sourceRecordId,
    ) {}
}

## 234. Exactly-Once

No asumirlo.
Documento 44.

## 235. Migration Ledger

VoltStack debería mantener un ledger operativo.

## 236. Ledger

interface AuthenticationMigrationLedgerInterface
{
    public function record(
        AuthenticationMigrationRecordOutcome $outcome
    ): void;

    public function find(
        AuthenticationMigrationRecordId $record
    ): ?AuthenticationMigrationRecordOutcome;
}

## 237. Ledger Purpose

Permite:
idempotency
resume
reporting
reconciliation
rollback planning
audit correlation

## 238. Ledger != Credential Store

No contiene secretos innecesarios.

## 239. Checkpoint

final readonly class AuthenticationMigrationCheckpoint
{
    public function __construct(
        public AuthenticationMigrationPartitionId $partition,
        public AuthenticationMigrationCursor $cursor,
        public DateTimeImmutable $updatedAt,
    ) {}
}

## 240. Resume

Después de failure:
resume from checkpoint
no comenzar todo desde cero.

## 241. Cursor Security

Cursor no podrá modificar scope.

## 242. Parallelization

Partition by:
tenant
identity range
source partition
credential type

## 243. Ordering

Algunos records requieren orden.
Ejemplo:
Identity
  ↓
Authentication Methods
  ↓
Credentials
  ↓
Session mappings

## 244. Dependency Graph

Migration Planner deberá poder definirlo.

## 245. Migration Stage

final readonly class AuthenticationMigrationStage
{
    public function __construct(
        public AuthenticationMigrationStageId $id,
        public AuthenticationMigrationDomain $domain,
        public AuthenticationMigrationStageDependencySet $dependencies,
    ) {}
}

## 246. Cycles

Plan compiler deberá detectar dependencies cíclicas.

## 247. Background Resource Governance

Documento 45.
Migration no podrá consumir toda capacidad de Authentication.

## 248. Critical Rule

Login production tiene prioridad sobre:
bulk migration

## 249. Migration Resource Pool

Preferir separado:
auth-migration

## 250. Admission Control

Puede reducir migration concurrency durante:
traffic spike
incident
database pressure
KMS saturation

## 251. Adaptive Migration Throughput

System Healthy
→ 100 workers

System Degraded
→ 20 workers

Authentication Incident
→ 2 workers / pause

## 252. Noisy Tenant

Migración Tenant A no degrada Authentication Tenant B.

## 253. Rate Limits

Aplicar a source APIs y providers.

## 254. Source Backpressure

Migration reader debe soportar:
pause
resume
rate reduction

## 255. Legacy Dependency Outage

No deberá provocar fallback inseguro.

## 256. Example

Legacy DB unavailable:
Native user
→ authenticate normally

Unmigrated legacy user
→ temporary migration unavailable
No:
bypass credential verification

## 257. Availability Policy

Puede existir:
FAIL_CLOSED
NATIVE_ONLY
CONTROLLED_DEGRADED_COMPATIBILITY
según stage.

## 258. Fail Open

No permitido para credential verification.

## 259. Legacy Source Isolation

Idealmente read-only durante lazy migration.

## 260. SQL Injection Boundary

Legacy source adapters deberán parametrizar queries.

## 261. Source Record Validation

Legacy DB es input no confiable desde la perspectiva del nuevo security boundary.

## 262. Legacy PHP Serialization

Especialmente peligroso.

## 263. Never

unserialize($legacyValue);
sobre contenido no estrictamente confiable.

## 264. Prefer Safe Parser

Migrar formatos heredados mediante parser limitado.

## 265. Legacy Cookie Deserialization

Misma regla.

## 266. Algorithm Confusion

Legacy token parser no deberá permitir que input elija algoritmo arbitrariamente.

## 267. Cryptographic Compatibility

Documento 31.

## 268. Legacy Signing Keys

Pueden necesitar coexistir durante migration.

## 269. Key Separation

LEGACY_TOKEN_VERIFY
NATIVE_TOKEN_SIGN
serán propósitos diferentes.

## 270. Verification-Only Keys

Una key heredada puede estar disponible únicamente para:
VERIFY
no:
SIGN

## 271. Excellent Migration Pattern

Old Key
→ verify only

New Key
→ sign + verify

## 272. Legacy Key Retirement

Después de:
maximum token lifetime
+
clock skew
+
migration safety margin
puede retirarse.

## 273. Unknown kid

No descargar keys desde URL arbitraria.
Aplica doc31/39.

## 274. Token Issuer Migration

Debe validar:
old issuer
new issuer
audiences
token type
purpose

## 275. Token Exchange

Puede utilizarse:
valid legacy token
      ↓
controlled exchange
      ↓
native VoltStack token

## 276. Exchange Requirements

legacy token valid
issuer trusted
audience valid
identity mapping unique
security state valid
migration policy permits

## 277. Downscoping

Native token no deberá obtener más authority automáticamente.

## 278. Token Exchange != Authorization Migration

Authorization se reevalúa.

## 279. Session Exchange

Misma filosofía.

## 280. Legacy Authentication Evidence

final readonly class LegacyAuthenticationEvidence
{
    public function __construct(
        public AuthenticationMigrationSourceId $source,
        public AuthenticationMethodType $method,
        public AuthenticationEvidenceProperties $properties,
        public AuthenticationMigrationProvenance $provenance,
        public DateTimeImmutable $verifiedAt,
    ) {}
}

## 281. Evidence Normalization

Legacy authenticator produce evidence.
Assurance system decide cuánto vale.

## 282. Plugin Cannot Assert Assurance

Legacy adapter no podrá decir:
"this is HIGH assurance"
por sí solo.

## 283. Current Security Policy Wins

Documento 36.

## 284. Reauthentication During Migration

Legacy authenticated identity puede necesitar:
new factor enrollment
password re-entry
passkey
admin verification
antes de sensitive operation.

## 285. Upgrade Challenge

Documento 38.

## 286. Migration Flow Type

Podría incorporarse:
case SecurityUpgrade = 'security_upgrade';
al AuthenticationFlowType.

## 287. Upgrade Flow

Legacy Authentication succeeds
          ↓
Policy detects upgrade requirement
          ↓
Security Upgrade Flow
          ↓
Enroll/Verify Modern Method
          ↓
Update Authentication Context
          ↓
Retire Legacy Method
          ↓
Continue

## 288. Do Not Trap User Unnecessarily

Si upgrade no es obligatorio inmediatamente:
login succeeds
→ security center recommendation

## 289. Blocking Upgrade

Si security floor lo exige:
login
→ limited upgrade context
→ upgrade required
→ normal application unavailable

## 290. Restricted Upgrade Session

Puede permitir únicamente:
security upgrade
logout
support
recovery
security center

## 291. Upgrade Session != Normal Session

Scope explícito.

## 292. Recovery

Si legacy credential no puede migrarse:
Account Recovery
puede ser vía de transición.

## 293. Recovery Cannot Bypass Suspension

Documento 41.

## 294. Recovery Cannot Bypass Security Freeze

Documento 40.

## 295. Legacy Reset Flow

Debe usar native VoltStack recovery after cutover cuando sea posible.

## 296. Compatibility Routing

Authentication request puede ser resuelta:
Native Authenticator
       ↓
Migration Compatibility Resolver
       ↓
Legacy Authenticator
pero solo bajo reglas explícitas.

## 297. Compatibility Resolver

interface AuthenticationCompatibilityResolverInterface
{
    public function resolve(
        AuthenticationRequestContext $request,
        AuthenticationCompatibilityContext $context
    ): AuthenticationCompatibilityDecision;
}

## 298. Decision

enum AuthenticationCompatibilityDecisionType: string
{
    case NativeOnly = 'native_only';
    case LegacyEligible = 'legacy_eligible';
    case LegacyRestricted = 'legacy_restricted';
    case UpgradeRequired = 'upgrade_required';
    case CompatibilityExpired = 'compatibility_expired';
    case Denied = 'denied';
}

## 299. Compatibility Context

Incluye:
migration stage
identity migration status
credential migration status
tenant
realm
application
risk
security posture
deadline
policy version

## 300. Per-Identity Migration Status

enum IdentityAuthenticationMigrationStatus: string
{
    case NotDiscovered = 'not_discovered';
    case Pending = 'pending';
    case Imported = 'imported';
    case Compatibility = 'compatibility';
    case UpgradeRequired = 'upgrade_required';
    case Native = 'native';
    case Failed = 'failed';
    case Quarantined = 'quarantined';
}

## 301. Per-Credential Status

LEGACY_ONLY
IMPORTED
COMPATIBILITY
VERIFIED
UPGRADED
RETIRED
REJECTED

## 302. Migration Completeness

No usar solo:
98% users migrated

## 303. Security Completeness

Debe medir:
identities migrated
active identities migrated
successful-login population migrated
admin identities migrated
legacy credentials remaining
weak credentials remaining
legacy sessions remaining
legacy tokens remaining
machine credentials remaining
legacy verification traffic

## 304. Important Metric

legacy authentication rate
deberá tender a cero.

## 305. Legacy Usage Decay

Ejemplo:
Week 1   34.8%
Week 2   18.2%
Week 4    6.1%
Week 8    0.7%
Week 12   0.03%

## 306. Long Tail

Accounts inactivas pueden mantener legacy credentials durante mucho tiempo.

## 307. Deadline Strategy

Después de cierta fecha:
legacy login
→ recovery/reset required

## 308. Dormant Accounts

No necesitan conservar weaker verifier indefinidamente.

## 309. Legacy Credential Purge

Después de compatibility deadline:
retire/destroy legacy verifier data
según retention policy.

## 310. Privacy Integration

Documento 42.
Migration copies no deben convertirse en una segunda base de datos permanente.

## 311. Temporary Migration Data

Debe tener:
purpose
owner
retention
encryption
deletion schedule

## 312. Export Files

Especialmente:
users.csv
credentials.json
database dump
son security-sensitive.

## 313. Temporary File Policy

encrypted
restricted permissions
short retention
audit access
verified destruction workflow

## 314. Local Developer Machines

No deberán ser destino por defecto de production credential exports.

## 315. Backup Copies

Migration team deberá conocer dónde existen.

## 316. Migration Data Inventory

interface AuthenticationMigrationDataInventoryInterface
{
    public function register(
        AuthenticationMigrationDataAsset $asset
    ): void;
}

## 317. Data Asset

source dump
staging table
quarantine store
temporary mapping
validation report
migration ledger

## 318. Sensitive Report

No incluir raw hashes salvo necesidad operacional estricta.

## 319. Password Hashes Are Sensitive

Aunque no sean plaintext.

## 320. Hash Export

Debe minimizarse.

## 321. Consent

Documento 42.
Migration por sí misma no inventa consentimiento.

## 322. Legal Basis

Debe provenir de governance/application policy.

## 323. Audit

Toda migración deberá generar audit de alto nivel.

## 324. Migration Audit Events

AuthenticationMigrationCreated
AuthenticationMigrationValidated
AuthenticationMigrationApproved
AuthenticationMigrationStarted
AuthenticationMigrationPaused
AuthenticationMigrationResumed
AuthenticationMigrationFailed
AuthenticationMigrationRolledBack
AuthenticationMigrationCutoverStarted
AuthenticationMigrationCutoverCompleted
AuthenticationMigrationCompleted
AuthenticationLegacyCompatibilityEnabled
AuthenticationLegacyCompatibilityRestricted
AuthenticationLegacyCompatibilityDisabled
AuthenticationLegacySystemRetired

## 325. Credential Events

LegacyCredentialImported
LegacyCredentialVerified
LegacyCredentialUpgraded
LegacyCredentialRejected
LegacyCredentialQuarantined
LegacyCredentialRetired
LegacyCredentialDestroyed

## 326. Identity Events

LegacyIdentityMapped
LegacyIdentityConflictDetected
LegacyIdentityImported
LegacyIdentityQuarantined

## 327. Event Volume

Bulk imports no necesariamente necesitan un permanent audit event por cada harmless row.

## 328. But

Security-sensitive anomalies sí.

## 329. Aggregate Audit

Batch 182 completed
identities: 50,000
credentials: 48,722
quarantined: 112
failed: 9
weak hashes: 17,904

## 330. Detailed Ledger

Puede conservar operational record separado.

## 331. Actor

Migration puede ser iniciada por:
ADMINISTRATOR
SECURITY_OPERATOR
SYSTEM
AUTOMATION
MACHINE

## 332. Privileged Approval

Migración production deberá requerir authorization.

## 333. Fresh Authentication

Para:
cutover
rollback
legacy verifier enable
bulk credential import
key migration
legacy system retirement
puede requerirse fresh privileged authentication.

## 334. Dual Control

Recomendado para migraciones de alta criticidad.

## 335. Cutover

Debe ser un objeto formal.

## 336. Cutover Plan

final readonly class AuthenticationMigrationCutoverPlan
{
    public function__construct(
        public AuthenticationMigrationId $migration,
        public AuthenticationCutoverCheckpointSet $checks,
        public AuthenticationCutoverRollbackPlan $rollback,
        public DateTimeImmutable $plannedAt,
    ) {}
}

## 337. Pre-Cutover Checks

mapping validated
critical identities migrated
admin access tested
recovery tested
MFA tested
legacy verification metrics understood
session strategy ready
rollback tested
security keys available
queues healthy
database healthy
audit healthy
incident team ready

## 338. Cutover State Machine

PLANNED
  ↓
PRECHECK
  ↓
FREEZE/COORDINATE
  ↓
SWITCH_AUTHORITY
  ↓
VERIFY
  ↓
MONITOR
  ↓
COMMIT
Alternativas:
ABORT
ROLLBACK
DEGRADED

## 339. Authority Switch

Debe ser explícito.

## 340. Example

Before:
Legacy = authoritative
VoltStack = shadow

After:
VoltStack = authoritative
Legacy = compatibility-only

## 341. Later

VoltStack = authoritative
Legacy = disabled

## 342. Rollback

Debe definirse antes del cutover.

## 343. But Rollback Has Limits

Después de emitir nuevas native credentials:
full rollback
puede no ser seguro.

## 344. Forward Recovery

A menudo es preferible.

## 345. Example

Después de que usuarios creen passkeys en VoltStack:
restore old DB
perdería esos métodos.

## 346. Therefore

Rollback strategy puede ser:
restore compatibility routing
preserve native writes
fix forward
en vez de revertir todos los datos.

## 347. No Security Downgrade During Rollback

Rollback operacional no debe reactivar:
revoked credentials
weak legacy methods
expired sessions
deleted identities

## 348. Monotonic Security Upgrade

Principio clave:
Una migración puede retroceder operacionalmente, pero no debe retroceder silenciosamente las garantías de seguridad ya establecidas.

  1. Security Monotonicity
Ejemplos:
Revoked → never Active due rollback
Upgraded Hash → never replaced with weak hash
New MFA → never discarded silently
Security Epoch 8 → never return to 7
Deleted Session → never resurrected
  2. Rollback Ledger
Debe registrar qué cambios son:
REVERSIBLE
COMPENSATABLE
IRREVERSIBLE
SECURITY_MONOTONIC
  3. Migration Reconciliation
Después de cada stage.
  4. Reconciler
interface AuthenticationMigrationReconcilerInterface
{
    public function reconcile(
        AuthenticationMigrationId $migration,
        AuthenticationMigrationReconciliationScope $scope
    ): AuthenticationMigrationReconciliationResult;
}
  5. Reconciliation Checks
source identity count
target identity count
mapping completeness
credential counts
credential status
tenant mapping
realm mapping
security state
migration ledger
quarantine
duplicate records
  6. Count Equality Is Not Enough
1,000,000 source
1,000,000 target
no prueba correctness.
  7. Semantic Reconciliation
Debe verificar relationships.
  8. Example
Credential C
must belong to
mapped Identity I
in correct Tenant T
and Realm R
  9. Cryptographic Reconciliation
No intentar verificar password hashes sin password.
 10. Instead
Validar:
format
algorithm
parameters
parseability
policy classification
 11. Live Verification
La prueba real ocurre cuando usuario autentica.
 12. Migration Confidence
enum AuthenticationMigrationConfidence: string
{
    case Structural = 'structural';
    case SemanticallyValidated = 'semantically_validated';
    case CryptographicallyVerified = 'cryptographically_verified';
    case NativelyUpgraded = 'natively_upgraded';
}
 13. Excellent Distinction
Imported
≠
Verified
≠
Upgraded
 14. Legacy Retirement
Es parte obligatoria de la arquitectura.
 15. Retirement Checklist
legacy auth traffic near zero
compatibility deadline passed
legacy sessions expired
legacy tokens expired
legacy signing keys no longer required
legacy API keys revoked
unmigrated identities handled
quarantine resolved
recovery path tested
rollback window closed
security exports destroyed
legacy credentials destroyed/retained according policy
 16. Legacy Retirement State
COMPATIBILITY
DEPRECATED
READ_ONLY
VERIFY_ONLY
DISABLED
DATA_RETENTION
DESTROYED
 17. Verify-Only
Excelente estado transitorio para cryptographic keys/verifiers.
 18. Legacy Code Removal
No basta con configuration:
legacy.enabled = false
 19. Final Goal
Eliminar:
legacy verifiers
legacy parsers
legacy DB credentials
legacy network routes
legacy secrets
legacy signing capability
compatibility flags
cuando ya no sean necesarios.
 20. Attack Surface Reduction
Cada compatibility adapter retirado reduce superficie de ataque.
 21. Legacy Feature Budget
VoltStack debería poder listar deuda restante.
 22. Compatibility Inventory
interface AuthenticationCompatibilityInventoryInterface
{
    public function active(): AuthenticationCompatibilityFeatureSet;
}
 23. Example
Legacy SHA1 verifier       ACTIVE      0.04% logins
Legacy session parser      RETIRING    0.00%
Legacy OAuth provider      ACTIVE      2 tenants
Legacy API keys            RESTRICTED  17 credentials
 24. Security Debt Dashboard
Security Center/Admin tooling puede mostrar:
legacy authentication usage
weak credentials
migration deadlines
unresolved quarantine
compatibility adapters
retirement blockers
 25. Observability
Metrics:
auth_migration_records_total
auth_migration_records_failed_total
auth_migration_records_quarantined_total
auth_migration_credentials_imported_total
auth_migration_credentials_upgraded_total
auth_migration_legacy_authentication_total
auth_migration_legacy_authentication_failed_total
auth_migration_identity_conflict_total
auth_migration_batches_total
auth_migration_batch_duration_seconds
auth_migration_reconciliation_failure_total
auth_migration_compatibility_usage_total
auth_migration_security_upgrade_required_total
auth_migration_security_upgrade_completed_total
 26. Controlled Labels
migration_type
credential_type
outcome
security_class
stage
realm_class
 27. Avoid
identity_id
email
username
credential_id
migration_record_id
como metric labels.
 28. Tracing
Spans:
auth.migration.read
auth.migration.normalize
auth.migration.validate
auth.migration.map
auth.migration.import
auth.migration.verify
auth.migration.upgrade
auth.migration.reconcile
auth.migration.cutover
 29. Trace Redaction
Nunca incluir raw credentials.
 30. Failure Taxonomy
AUTH_MIGRATION_NOT_FOUND
AUTH_MIGRATION_INVALID_STATE
AUTH_MIGRATION_INVALID_TRANSITION
AUTH_MIGRATION_NOT_APPROVED

AUTH_MIGRATION_SOURCE_UNAVAILABLE
AUTH_MIGRATION_SOURCE_UNTRUSTED
AUTH_MIGRATION_SOURCE_INTEGRITY_FAILED
AUTH_MIGRATION_SOURCE_SCHEMA_UNSUPPORTED

AUTH_MIGRATION_RECORD_INVALID
AUTH_MIGRATION_RECORD_DUPLICATE
AUTH_MIGRATION_RECORD_QUARANTINED

AUTH_MIGRATION_IDENTITY_MAPPING_FAILED
AUTH_MIGRATION_IDENTITY_CONFLICT
AUTH_MIGRATION_IDENTIFIER_CONFLICT
AUTH_MIGRATION_TENANT_MAPPING_FAILED
AUTH_MIGRATION_REALM_MAPPING_FAILED

AUTH_MIGRATION_CREDENTIAL_UNSUPPORTED
AUTH_MIGRATION_CREDENTIAL_WEAK
AUTH_MIGRATION_CREDENTIAL_COMPROMISED
AUTH_MIGRATION_CREDENTIAL_IMPORT_FAILED
AUTH_MIGRATION_CREDENTIAL_UPGRADE_FAILED

AUTH_MIGRATION_PASSWORD_HASH_UNSUPPORTED
AUTH_MIGRATION_PASSWORD_RESET_REQUIRED

AUTH_MIGRATION_SESSION_UNSUPPORTED
AUTH_MIGRATION_SESSION_REAUTHENTICATION_REQUIRED

AUTH_MIGRATION_MFA_REENROLLMENT_REQUIRED
AUTH_MIGRATION_PASSKEY_RP_INCOMPATIBLE
AUTH_MIGRATION_FEDERATED_MAPPING_AMBIGUOUS

AUTH_MIGRATION_COMPATIBILITY_DISABLED
AUTH_MIGRATION_COMPATIBILITY_EXPIRED
AUTH_MIGRATION_LEGACY_FALLBACK_FORBIDDEN

AUTH_MIGRATION_BATCH_FAILED
AUTH_MIGRATION_CHECKPOINT_INVALID
AUTH_MIGRATION_VERSION_CONFLICT

AUTH_MIGRATION_RECONCILIATION_FAILED
AUTH_MIGRATION_CUTOVER_PRECHECK_FAILED
AUTH_MIGRATION_CUTOVER_FAILED
AUTH_MIGRATION_ROLLBACK_UNSAFE

AUTH_MIGRATION_SECURITY_DOWNGRADE_FORBIDDEN
AUTH_MIGRATION_POLICY_VIOLATION
AUTH_MIGRATION_RESOURCE_LIMITED

## 379. Security Invariants — General

AUTH-MIG-001
Migration nunca implicará trust automático.
AUTH-MIG-002
Imported != Verified.
AUTH-MIG-003
Verified != Upgraded.
AUTH-MIG-004
Backward compatibility será temporal y gobernada.
AUTH-MIG-005
Current security floor tendrá prioridad sobre legacy behavior.
AUTH-MIG-006
Migration no podrá ampliar Authorization.
AUTH-MIG-007
Toda imported credential conservará provenance suficiente.

## 380. Security Invariants — Passwords

AUTH-MIG-PWD-001
Legacy verifier no podrá crear nuevas legacy credentials.
AUTH-MIG-PWD-002
New passwords usarán native password policy.
AUTH-MIG-PWD-003
Plaintext passwords no se persistirán en migration logs/queues.
AUTH-MIG-PWD-004
Rehash que requiere plaintext ocurrirá durante el request que lo posee.
AUTH-MIG-PWD-005
Successful rehash retirará el credential heredado según policy.
AUTH-MIG-PWD-006
Weak legacy hashes no recibirán assurance artificialmente alta.

## 381. Security Invariants — Identity

AUTH-MIG-ID-001
No se hará auto-merge por email.
AUTH-MIG-ID-002
Legacy ID no se asumirá igual a Identity ID.
AUTH-MIG-ID-003
Tenant mapping será explícito.
AUTH-MIG-ID-004
Unknown tenant no caerá en default tenant.
AUTH-MIG-ID-005
Realm mapping será explícito.
AUTH-MIG-ID-006
Lifecycle state y Protection state no se mezclarán.

## 382. Security Invariants — Sessions

AUTH-MIG-SES-001
Legacy session no reutilizará su identifier como native session ID.
AUTH-MIG-SES-002
Legacy session no obtendrá assurance no demostrada.
AUTH-MIG-SES-003
Privileged context no se restaurará desde una session incompatible sin evidencia suficiente.
AUTH-MIG-SES-004
Revoked legacy session nunca será reactivada durante migration.

## 383. Security Invariants — Federation

AUTH-MIG-FED-001
Federated identity se mapeará por issuer/subject/provider apropiado.
AUTH-MIG-FED-002
Email no será external identity key.
AUTH-MIG-FED-003
Provider ambiguity deberá fallar de forma segura.
AUTH-MIG-FED-004
Refresh tokens no se migrarán automáticamente sin policy explícita.

## 384. Security Invariants — Distributed Migration

AUTH-MIG-DIST-001
Migration batches serán idempotentes.
AUTH-MIG-DIST-002
Exactly-once no será asumido.
AUTH-MIG-DIST-003
Checkpoints estarán scoped.
AUTH-MIG-DIST-004
Retries no crearán identities duplicadas.
AUTH-MIG-DIST-005
Stale batch no sobrescribirá state más nuevo.
AUTH-MIG-DIST-006
Migration tendrá resource governance separado del login path.

## 385. Security Invariants — Rollback

AUTH-MIG-RB-001
Rollback no reactivará revoked credentials.
AUTH-MIG-RB-002
Rollback no reducirá security epoch.
AUTH-MIG-RB-003
Rollback no reemplazará modern password hash por weak hash.
AUTH-MIG-RB-004
Rollback no resucitará sessions.
AUTH-MIG-RB-005
Irreversible security changes estarán marcados explícitamente.

## 386. Anti-Pattern

if (!$voltStackLogin) {
    return $legacyLogin();
}

## 387. Anti-Pattern

native authentication error
→ fallback to weaker legacy system

## 388. Anti-Pattern

import users
→ match by email
→ merge duplicates automatically

## 389. Anti-Pattern

import legacy admin
→ automatically grant privileged assurance

## 390. Anti-Pattern

legacy SHA1 supported
→ allow creation of new SHA1 passwords

## 391. Anti-Pattern

rehash later in queue
→ queue contains plaintext password

## 392. Anti-Pattern

migration successful
because row counts match

## 393. Anti-Pattern

legacy session cookie
→ copy directly as VoltStack session ID

## 394. Anti-Pattern

Google account
→ link by matching email

## 395. Anti-Pattern

unknown tenant
→ assign default tenant

## 396. Anti-Pattern

migration rollback
→ restore all old credentials

## 397. Anti-Pattern

compatibility mode
→ permanent

## 398. Anti-Pattern

migration dump remains on developer laptop

## 399. Anti-Pattern

legacy verifier package remains installed forever

## 400. Anti-Pattern

migration state stored in static property
bajo FrankenPHP.

## 401. FrankenPHP Safety

Migration workers pueden ser long-lived.
Por tanto:
No static current migration
No static current tenant
No static current identity
No static current credential
No mutable cross-job security context

## 402. Worker Lifecycle

RESET
  ↓
Load Migration Task
  ↓
Establish Explicit Scope
  ↓
Load Immutable Migration Snapshot
  ↓
Process
  ↓
Flush
  ↓
RESET

## 403. Fiber Safety

Fiber A → Tenant A migration
Fiber B → Tenant B migration
no compartirán contexto mutable.

## 404. Migration Context

final readonly class AuthenticationMigrationContext
{
    public function __construct(
        public AuthenticationMigrationId $migrationId,
        public AuthenticationMigrationScope $scope,
        public AuthenticationMigrationExecutionSnapshot $snapshot,
        public MachineIdentityReference $executor,
    ) {}
}

## 405. No Ambient Tenant

Todo adapter recibe scope explícito.

## 406. Secret Lifetime

Migration worker deberá minimizar el tiempo que secretos importables permanecen en memoria.

## 407. Worker Recycling

Especialmente importante después de batches que procesen:
TOTP secrets
API secrets
private credential material

## 408. Extensibility

Plugins podrán añadir:
Legacy Sources
Legacy Password Verifiers
Identity Mappers
Credential Mappers
Session Parsers
Token Parsers
MFA Importers
Federation Mappers
Security Classifiers
Migration Validators
Reconciliation Strategies

## 409. Plugin Contract

interface AuthenticationMigrationPluginInterface
{
    public function register(
        AuthenticationMigrationRegistry $registry
    ): void;
}

## 410. Plugin Trust Boundary

Plugin no podrá afirmar por sí solo:
HIGH assurance
platform authorization
privileged identity
trusted tenant mapping
sin pasar por sistemas centrales.

## 411. Legacy Adapter Capability

final readonly class AuthenticationLegacyAdapterCapabilities
{
    public function __construct(
        public bool $canRead,
        public bool $canVerify,
        public bool $canImport,
        public bool $canWriteLegacy,
        public bool $containsSecrets,
    ) {}
}

## 412. Prefer

canWriteLegacy = false

## 413. Compilation

Migration registries estáticos pueden compilarse.

## 414. Compiled Migration Registry

final readonly class CompiledAuthenticationMigrationRegistry
{
    public function__construct(
        public array $sources,
        public array $verifiers,
        public array $mappers,
        public array $validators,
    ) {}
}

## 415. Compile-Time Validation

Detectar:
duplicate adapters
unsupported source
missing mapper
conflicting verifier
unsafe fallback
write-capable legacy adapter
missing tenant mapper
missing retirement deadline

## 416. Configuration

Ejemplo conceptual:
return [

    'migration' => [

        'enabled' => true,

        'mode' => 'progressive',

        'source' => 'legacy_app',

        'compatibility' => [
            'read_legacy' => true,
            'write_legacy' => false,
            'deadline' => '2027-03-01',
        ],

        'passwords' => [
            'rehash_on_login' => true,

            'allowed_legacy_verifiers' => [
                'bcrypt',
                'pbkdf2',
                'sha1_legacy',
            ],

            'reject' => [
                'plaintext',
                'unknown',
            ],
        ],

        'sessions' => [
            'accept_legacy' => false,
        ],

        'security_upgrade' => [
            'admins' => 'blocking',
            'users' => 'progressive',
        ],

        'background' => [
            'queue' => 'auth-migration',
            'batch_size' => 1000,
            'max_concurrency' => 10,
        ],
    ],
];

## 417. Configuration Security

Production config no deberá contener:
legacy database passwords
private keys
migration export encryption keys
directamente.
Usar secret references.

## 418. Developer Experience

Ejemplo:
AuthMigration::source('legacy')
    ->identityMapper(LegacyIdentityMapper::class)
    ->passwordVerifier(LegacySha1Verifier::class)
    ->readLegacy()
    ->writeNativeOnly()
    ->upgradeOnLogin()
    ->retireLegacyAfter('2027-03-01');

## 419. Security Guardrail

Si developer intenta:
->writeLegacyPasswords()
VoltStack podrá:
reject configuration
require explicit insecure compatibility capability
emit security finding
dependiendo del use case.

## 420. CLI Conceptual

voltstack auth:migration:discover
voltstack auth:migration:plan
voltstack auth:migration:validate
voltstack auth:migration:dry-run
voltstack auth:migration:start
voltstack auth:migration:status
voltstack auth:migration:pause
voltstack auth:migration:resume
voltstack auth:migration:reconcile
voltstack auth:migration:cutover
voltstack auth:migration:rollback
voltstack auth:migration:retire

## 421. CLI Security

Sensitive commands requieren privileged administration.

## 422. Example Status

Migration: legacy-production-v1
State: MONITORING

Identities
  Imported:          4,912,288
  Native upgraded:   4,102,992
  Compatibility:       801,144
  Quarantined:            8,152

Passwords
  Native:            83.5%
  Legacy strong:     12.7%
  Legacy weak:        3.8%

Legacy login traffic:
  0.42%

Cutover:
  COMPLETE

Legacy retirement:
  BLOCKED
  Reason: 17 privileged identities require upgrade

## 423. Testing Strategy

Debe cubrir:
unit
integration
migration fixtures
shadow comparison
failure injection
distributed batches
rollback
cutover
FrankenPHP
multi-tenant
security invariants

## 424. Test — Weak Hash

Legacy SHA1 verifies.
Native Argon2 credential generated.
Legacy credential retired.

## 425. Test — Wrong Password

Native missing + legacy hash exists.
Wrong password:
DENIED
No upgrade.

## 426. Test — Native Invalid Password

Native credential exists.
Invalid password no activa legacy fallback.

## 427. Test — Suspended Identity

Valid legacy password + suspended identity:
DENIED

## 428. Test — Tenant Conflict

Legacy identity maps ambiguously.
QUARANTINED

## 429. Test — Duplicate Email

No auto-merge.

## 430. Test — Admin Migration

Legacy admin password verifies.
Privileged realm requires modern step-up.

## 431. Test — Session Migration

Legacy session cannot obtain undocumented MFA assurance.

## 432. Test — Password Rehash Failure

Policy-specific safe outcome.
No plaintext queued.

## 433. Test — Duplicate Batch

No duplicate identities.

## 434. Test — Worker Crash

Checkpoint + idempotency recover correctly.

## 435. Test — Stale Batch

Cannot overwrite newer credential state.

## 436. Test — Rollback

Does not reactivate revoked credential.

## 437. Test — Security Epoch

Never decreases.

## 438. Test — Legacy Key

Can verify but cannot sign.

## 439. Test — Compatibility Deadline

Expired compatibility:
legacy login rejected / upgrade path offered

## 440. Test — Legacy DB Outage

No fail-open.

## 441. Test — Shadow Mode

No Authentication decision changed.

## 442. Test — Sensitive Data

No password/hash/OTP/private key leaked to logs.

## 443. Test — FrankenPHP

Migration context reset between jobs.

## 444. Suggested Namespace

VoltStack\Quantum\Auth\Migration

## 445. Suggested Directory Structure

src/Quantum/Auth/Migration/
├── Contracts/
│   ├── AuthenticationMigrationSourceInterface.php
│   ├── AuthenticationMigrationMapperInterface.php
│   ├── AuthenticationMigrationReconcilerInterface.php
│   ├── AuthenticationCompatibilityResolverInterface.php
│   ├── LegacyPasswordVerifierInterface.php
│   ├── LegacyPasswordVerifierRegistryInterface.php
│   ├── LegacyIdentityMappingStoreInterface.php
│   └── AuthenticationMigrationLedgerInterface.php
│
├── Domain/
│   ├── AuthenticationMigrationId.php
│   ├── AuthenticationMigrationState.php
│   ├── AuthenticationMigrationMode.php
│   ├── AuthenticationMigrationScope.php
│   ├── AuthenticationMigrationManifest.php
│   ├── AuthenticationMigrationPlan.php
│   ├── AuthenticationMigrationStage.php
│   └── AuthenticationMigrationContext.php
│
├── Source/
│   ├── AuthenticationMigrationSourceRegistry.php
│   ├── SqlMigrationSource.php
│   ├── FileMigrationSource.php
│   ├── ApiMigrationSource.php
│   └── LegacyApplicationSource.php
│
├── Identity/
│   ├── LegacyIdentityReference.php
│   ├── LegacyIdentityMapper.php
│   ├── LegacyIdentityStateMapper.php
│   └── LegacyIdentityMappingStore.php
│
├── Credential/
│   ├── LegacyCredentialClassifier.php
│   ├── LegacyCredentialImportDecision.php
│   ├── LegacyCredentialSecurityClass.php
│   └── CredentialMigrationManager.php
│
├── Password/
│   ├── LegacyPasswordHashDescriptor.php
│   ├── LegacyPasswordVerifierRegistry.php
│   ├── LegacyBcryptVerifier.php
│   ├── LegacyPbkdf2Verifier.php
│   ├── LegacySha1Verifier.php
│   └── PasswordUpgradeManager.php
│
├── Session/
│   ├── LegacySessionAuthenticator.php
│   ├── LegacySessionParser.php
│   └── SessionMigrationManager.php
│
├── Mfa/
│   ├── LegacyTotpDescriptor.php
│   ├── TotpMigrationManager.php
│   └── MfaReenrollmentPlanner.php
│
├── Passkey/
│   ├── PasskeyMigrationManager.php
│   └── PasskeyRpCompatibilityChecker.php
│
├── Federation/
│   ├── LegacyFederatedIdentityDescriptor.php
│   ├── FederatedIdentityMigrationManager.php
│   └── LegacyProviderMapper.php
│
├── Machine/
│   ├── MachineCredentialMigrationManager.php
│   └── SharedSecretMigrationPlanner.php
│
├── Compatibility/
│   ├── AuthenticationCompatibilityPolicy.php
│   ├── AuthenticationCompatibilityResolver.php
│   ├── AuthenticationCompatibilityInventory.php
│   └── LegacyAuthenticationRouter.php
│
├── Upgrade/
│   ├── AuthenticationSecurityUpgradeRequirement.php
│   ├── AuthenticationSecurityUpgradePhase.php
│   ├── AuthenticationSecurityUpgradeManager.php
│   └── AuthenticationSecurityUpgradePolicy.php
│
├── Validation/
│   ├── AuthenticationMigrationValidator.php
│   ├── AuthenticationMigrationFinding.php
│   └── AuthenticationMigrationValidationResult.php
│
├── Quarantine/
│   ├── AuthenticationMigrationQuarantineStore.php
│   └── AuthenticationMigrationQuarantineReason.php
│
├── Batch/
│   ├── AuthenticationMigrationBatch.php
│   ├── AuthenticationMigrationBatchPolicy.php
│   ├── AuthenticationMigrationCheckpoint.php
│   └── AuthenticationMigrationLedger.php
│
├── Cutover/
│   ├── AuthenticationMigrationCutoverPlan.php
│   ├── AuthenticationMigrationCutoverManager.php
│   └── AuthenticationCutoverRollbackPlan.php
│
├── Reconciliation/
│   ├── AuthenticationMigrationReconciler.php
│   └── AuthenticationMigrationReconciliationResult.php
│
├── Compilation/
│   ├── AuthenticationMigrationCompiler.php
│   └── CompiledAuthenticationMigrationRegistry.php
│
├── Events/
├── Exceptions/
└── Testing/

## 446. Comparación con Laravel

Laravel ofrece una base excelente para:
Hash
Auth Providers
Guards
Sessions
Password Brokers
Events
Queues
Migrations
y facilita verificar si un hash necesita rehash.
Sin embargo, una migración empresarial entre sistemas normalmente queda en manos de la aplicación:
legacy identity mapping
dual-read authentication
safe fallback semantics
legacy verifier isolation
credential provenance
progressive MFA/passkey upgrades
session compatibility
migration quarantine
cutover state machine
rollback security monotonicity
legacy retirement governance
VoltStack convierte esas necesidades en un subsistema formal.

## 447. Comparación con Symfony

Symfony proporciona primitivas especialmente útiles mediante:
PasswordHasher
PasswordUpgraderInterface
User Providers
Authenticators
Security
Messenger
Console
LDAP integrations
y posee una arquitectura sólida para password upgrading.
VoltStack toma esa disciplina y amplía el concepto desde:
Password Upgrade
hacia:
Complete Authentication Security Migration
incluyendo identities, sessions, MFA, passkeys, federation, machine credentials, provenance, cutover y retirement.

## 448. Diferenciador VoltStack

El modelo final será:
Legacy System
     │
     ▼
Migration Source
     │
     ▼
Validation + Classification
     │
     ▼
Identity/Credential Mapping
     │
     ├──────────────┐
     ▼              ▼
Bulk Import     Lazy Migration
     │              │
     └──────┬───────┘
            ▼
   Compatibility Layer
            │
            ▼
 Legacy Authentication Evidence
            │
            ▼
 Current Policy + Assurance
            │
      ┌─────┴─────┐
      ▼           ▼
   Accept      Upgrade
                  │
                  ▼
          Native Credential
                  │
                  ▼
          Legacy Retirement

## 449. Matriz de estrategia recomendada

Artefacto Estrategia preferida
Identity Import + explicit mapping
Modern password hash Import compatible / upgrade when needed
Weak password hash Verify once + native rehash
Unknown password hash Reset/recovery
Plaintext password Reject or immediate secure hashing under exceptional controlled migration
Legacy session Prefer re-login; controlled exchange only if necessary
Remember-me Prefer reauthentication
TOTP Import only with secure secret handling and compatible parameters
Passkey Import public credential data only when RP semantics remain valid
Recovery codes Prefer regenerate
OAuth/OIDC link Map by issuer + subject
OAuth refresh token Prefer re-consent
API key Rotate to native key
Shared machine secret Split into per-service identities
Client certificate Validate/reissue where possible
Legacy signing key Verification-only during transition
Security state Semantic mapping, never blind field copy

  1. Architectural Relationship
Los documentos recientes quedan organizados así:
41 Identity Lifecycle
        │
42 Privacy / Retention
        │
43 Security Communication
        │
44 Background Processing
        │
45 Capacity / Abuse / DoS Governance
        │
46 Migration / Compatibility / Upgrade
Pero 46 conecta prácticamente con todo Authentication:
                    ┌─────────────────────┐
                    │   Legacy Systems    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Migration System 46 │
                    └──────────┬──────────┘
                               │
             ┌─────────────────┼──────────────────┐
             ▼                 ▼                  ▼
         Identity          Credentials          Sessions
           41                11/34               12
             │                 │                  │
             ├─────────────────┼──────────────────┤
                               ▼
                    Authentication Evidence
                               08
                               │
                               ▼
                        Assurance System
                               37
                               │
                               ▼
                         Policy Engine
                               36
                               │
                               ▼
                     Progressive Upgrade
                               │
                               ▼
                    Native VoltStack State
  2. Acceptance Criteria
El sistema 46 se considerará arquitectónicamente completo cuando VoltStack pueda:

- importar identidades sin auto-merge inseguro;
- mantener mappings estables de legacy IDs;
- clasificar credenciales heredadas;
- verificar hashes legacy sin permitir crearlos nuevamente;
- realizar rehash-on-login;
- importar credenciales compatibles;
- exigir reset/reenrollment cuando la migración segura sea imposible;
- migrar MFA, federation, passkeys y machine identities de forma controlada;
- ejecutar dual-read sin fallback inseguro;
- trabajar en shadow mode;
- soportar progressive migration;
- manejar millones de registros mediante background batches;
- garantizar idempotencia;
- aplicar tenant isolation;
- preservar provenance;
- ejecutar reconciliation;
- realizar cutover formal;
- soportar rollback sin security downgrade;
- medir el uso de legacy Authentication;
- establecer deadlines de compatibility;
- retirar verificadores, keys y adapters legacy;
- destruir datos temporales según governance;
- operar correctamente bajo FrankenPHP y workers persistentes.
  1. Regla arquitectónica final
El objetivo de este sistema no es que VoltStack sea compatible para siempre con cualquier Authentication heredada.
El objetivo es permitir:
Legacy Security
      ↓
Safely Understood
      ↓
Explicitly Classified
      ↓
Temporarily Supported
      ↓
Progressively Upgraded
      ↓
Cryptographically Replaced
      ↓
Operationally Cut Over
      ↓
Measured
      ↓
Retired
      ↓
Removed
Por tanto:
La compatibilidad heredada en VoltStack siempre deberá tener dirección, estado, política y una estrategia de salida.

Y la propiedad más importante de todo el proceso será:
Operational Migration
        +
Security Monotonicity
es decir:
VoltStack podrá retroceder operacionalmente cuando una migración falle, pero nunca deberá restaurar silenciosamente una garantía de seguridad más débil después de que una identidad, credencial, sesión o método haya sido actualizado de forma segura.

  1. Siguiente documento recomendado
El siguiente documento es:
47_AUTHENTICATION_DEVELOPER_EXPERIENCE_FACADE_HELPER_CONFIGURATION_BOOTSTRAP_AND_APPLICATION_INTEGRATION_SYSTEM.md
Aquí cambiaremos de la infraestructura interna hacia la experiencia real que tendrá un desarrollador usando Authentication en VoltStack.
El objetivo será convertir los sistemas 01–46 en una API coherente:
Auth Architecture
      ↓
Auth Manager
      ↓
Facade / Contracts
      ↓
Configuration
      ↓
Bootstrap
      ↓
Middleware
      ↓
Routes / Controllers
      ↓
Application Code
incluyendo conceptos como:
Auth::user();

Auth::check();

Auth::id();

Auth::attempt([
    'email' => $email,
    'password' => $password,
]);

Auth::logout();

Auth::sessions();

Auth::methods();

Auth::security();

Auth::requireFreshAuthentication();

Auth::requireAssurance(...);
pero sin permitir que la comodidad tipo Laravel destruya las garantías arquitectónicas que hemos definido en los 46 sistemas anteriores.
