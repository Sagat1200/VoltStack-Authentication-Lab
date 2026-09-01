# VoltStack Authentication System

## 31 — Authentication Cryptographic Key, Secret, Certificate, Trust and Key Lifecycle Management System

- **Archivo:** `31_AUTHENTICATION_CRYPTOGRAPHIC_KEY_SECRET_CERTIFICATE_TRUST_AND_KEY_LIFECYCLE_MANAGEMENT_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth` + integración con `Quantum/Config`, `Quantum/Cache`, `Quantum/Concurrency`, `Quantum/Event` y proveedores KMS/HSM
- **Estado:** Especificación arquitectónica del subsistema encargado de claves criptográficas, secretos, certificados, trust stores, key rings, rotación, revocación, distribución y lifecycle criptográfico de Authentication.

---

## 1. Propósito

Este documento define cómo VoltStack deberá administrar todo material criptográfico utilizado directa o indirectamente por Authentication.
Incluye:

- Signing Keys
- Encryption Keys
- MAC Keys
- Password Peppers
- Recovery Secrets
- Session Protection Keys
- Remember-Me Keys
- Token Keys
- OIDC Client Secrets
- OAuth Secrets
- WebAuthn Trust Anchors
- mTLS Certificates
- Device Certificates
- JWT/JWK Keys
- Key Rings
- Trust Stores
- Secret References
- Key Rotation
- Key Activation
- Key Retirement
- Key Revocation
- Key Compromise
- KMS Integration
- HSM Integration
- Certificate Validation
- Cryptographic Algorithm Policy
- Cryptographic Agility

## 2. Principio fundamental

Las credenciales del usuario y las claves criptográficas del sistema son dominios de seguridad distintos y deberán administrarse mediante lifecycles diferentes.

## 3. Segunda regla fundamental

El Authentication Core nunca deberá depender de secretos criptográficos hardcoded, embebidos en caches compiladas o almacenados como texto plano dentro de configuración distribuible.

## 4. Tercera regla fundamental

Toda clave deberá tener identidad, propósito, versión, estado y lifecycle explícitos.

## 5. Arquitectura general

AUTHENTICATION SUBSYSTEMS
│
▼
CRYPTOGRAPHIC SERVICES
│
┌───────────────────┼───────────────────┐
▼                   ▼                   ▼
SIGNATURE            ENCRYPTION             MAC
│                   │                   │
└──────────────┬────┴──────┬────────────┘
▼           ▼
KEY RING    SECRET RESOLVER
│           │
▼           ▼
KMS/HSM   SECRET PROVIDERS
│           │
└─────┬─────┘
▼
TRUST SYSTEM
│
┌────────────┼────────────┐
▼            ▼            ▼
CA STORE       JWKS         CERTS

## 6. Tipos de material criptográfico

VoltStack deberá diferenciar al menos:

- SYMMETRIC_KEY
- ASYMMETRIC_PRIVATE_KEY
- ASYMMETRIC_PUBLIC_KEY
- MAC_KEY
- PASSWORD_PEPPER
- CLIENT_SECRET
- CERTIFICATE
- TRUST_ANCHOR
- SEED_SECRET
- RECOVERY_SECRET

## 7. CryptographicKeyId

Toda clave tendrá identificador estable.

```php
final readonly class CryptographicKeyId
{
    public function __construct(
        public string $value,
    ) {}
}
```

## 8. Key ID no es secret

Podrá aparecer en:

- token headers
- logs
- audit
- diagnostics
- cuando sea apropiado.

## 9. Key Version

Toda clave rotatable deberá tener:
KeyVersion
Ejemplo:
session-signing:v7

## 10. Key Purpose

Una clave nunca deberá reutilizarse indiscriminadamente.

- Propósitos:
- SESSION_SIGNING
- SESSION_ENCRYPTION
- REMEMBER_ME_MAC
- TOKEN_SIGNING
- TOKEN_ENCRYPTION
- RECOVERY_PROTECTION
- DEVICE_CREDENTIAL
- OIDC_CLIENT_AUTH
- INTERNAL_EVENT_SIGNING

## 11. Domain Separation

Regla crítica:
Claves de propósitos distintos deberán ser independientes o derivadas mediante mecanismos criptográficos explícitos con domain separation.

## 12. Anti-pattern

No:

```text
APP_KEY
  used for sessions
  used for tokens
  used for remember-me
  used for recovery
  used for everything
```

## 13. AuthenticationKeyPurpose

enum AuthenticationKeyPurpose: string
{
case SessionSigning = 'session_signing';
case SessionEncryption = 'session_encryption';
case TokenSigning = 'token_signing';
case RememberMeMac = 'remember_me_mac';
case RecoveryProtection = 'recovery_protection';
}

## 14. Key Metadata

Una clave deberá tener metadata como:

- KeyId
- Version
- Purpose
- Algorithm
- CreatedAt
- ActivatesAt
- RetiresAt
- RevokedAt
- Status
- Provider

## 15. Key Status

PENDING
ACTIVE
VERIFY_ONLY
RETIRED
REVOKED
DESTROYED

## 16. PENDING

La clave existe pero todavía no firma/emite artefactos.

## 17. ACTIVE

Puede utilizarse para emitir nuevo material.

## 18. VERIFY_ONLY

No crea artefactos nuevos, pero puede verificar antiguos.

## 19. RETIRED

Normalmente fuera de uso, pero puede mantenerse para recovery histórico controlado.

## 20. REVOKED

No deberá considerarse válida.

## 21. DESTROYED

Material ya no existe.

## 22. Key Lifecycle

GENERATED
↓
PENDING
↓
ACTIVE
↓
VERIFY_ONLY
↓
RETIRED
↓
DESTROYED
Con camino de emergencia:

```text
ANY STATE
    ↓
REVOKED
```

## 23. AuthenticationKeyDescriptor

final readonly class AuthenticationKeyDescriptor
{
public function __construct(
public CryptographicKeyId $id,
public KeyVersion $version,
public AuthenticationKeyPurpose $purpose,
public CryptographicAlgorithm $algorithm,
public AuthenticationKeyStatus $status,
) {}
}

## 24. Key Material

Debe estar separado del descriptor.

## 25. Descriptor puede ser serializable

Material no necesariamente.

## 26. AuthenticationKeyMaterial

No deberá implementar:

```php
__toString()
JsonSerializable
```

debug dump with raw key

## 27. SensitiveKeyMaterial

Podrá utilizar wrapper:

```php
final class SensitiveKeyMaterial
{
    public function __debugInfo(): array
    {
        return ['value' => '[REDACTED]'];
    }
}
```

## 28. SecretReference

Configuración deberá utilizar referencias.

```php
Ejemplo:
'oidc_client_secret' => SecretReference::named(
    'tenant-acme-oidc-secret'
);
```

## 29. Secret Resolver

Contrato:

```php
interface AuthenticationSecretResolverInterface
{
    public function resolve(
        SecretReference $reference
    ): SensitiveSecret;
}
```

## 30. Secret Providers

Podrán existir:

- EnvironmentSecretProvider
- EncryptedFileSecretProvider
- DatabaseSecretProvider
- VaultSecretProvider
- AwsSecretsManagerProvider
- GcpSecretManagerProvider
- AzureKeyVaultProvider
- CustomSecretProvider

## 31. Secret Provider Resolution

Debe estar compilado/configurado durante bootstrap.
El secret real puede resolverse runtime.

## 32. No secret dump during compilation

Documento 27 deberá almacenar:
SecretReference
no:
actual secret

## 33. Environment Secrets

Aceptables para ciertos deployments.

- Pero deberán considerarse:
- process-visible
- deployment-managed

non-rotating unless process reloads

## 34. Dynamic Secret Providers

Permiten rotación sin nuevo build.

## 35. Secret Cache

Puede existir.

- Deberá ser:
- bounded
- TTL-controlled
- never logged
- never serialized

## 36. Secret Cache Scope

Algunas claves pueden cachearse por worker.

## 37. FrankenPHP concern

Worker persistente puede conservar material criptográfico durante mucho tiempo.
Debe existir política de:

- TTL
- refresh
- rotation awareness
- zeroization best effort

## 38. Zeroization

PHP no puede garantizar borrado físico perfecto del secret debido a:

- copy-on-write
- runtime memory management
- garbage collection

VoltStack deberá evitar prometer secure zeroization absoluta.

## 39. Best-effort secret hygiene

Deberá minimizar:

- copies
- string concatenation
- debugging
- serialization
- long-lived references

## 40. KMS Integration

VoltStack deberá permitir claves que nunca salgan del proveedor.

## 41. KMS signing

Payload Hash
↓
KMS Sign API
↓
Signature
Private key no entra al proceso PHP.

## 42. KMS encryption

Igual:

```text
plaintext/data key operation
    ↓
KMS
según perfil.
```

## 43. HSM

Para entornos de alta seguridad podrá integrarse:

- HSM
- PKCS#11
- Cloud HSM
- mediante adapters.

## 44. Crypto Provider

Contrato:

```php
interface AuthenticationCryptographicProviderInterface
{
    public function sign(
        AuthenticationKeyReference $key,
        SensitivePayload $payload
    ): CryptographicSignature;

    public function verify(
        AuthenticationKeyReference $key,
        SensitivePayload $payload,
        CryptographicSignature $signature
    ): bool;
}
```

## 45. Encryption Provider

interface AuthenticationEncryptionProviderInterface
{
public function encrypt(
AuthenticationKeyReference $key,
SensitivePayload $payload
): EncryptedPayload;

public function decrypt(
AuthenticationKeyReference $key,
EncryptedPayload $payload
): SensitivePayload;
}

## 46. MAC Provider

Separado.

## 47. Password Pepper

Pepper no deberá confundirse con password salt.

## 48. Salt

Normalmente:

- unique per password
- stored with password hash

## 49. Pepper

Puede ser:

- application secret
- not stored with password database

## 50. Pepper Policy

Pepper será opcional y profile-driven.

## 51. Pepper Rotation

Es compleja porque puede requerir:

- multi-pepper verification
- rehash on successful login

## 52. Pepper Ring

Podrá existir:

- current pepper
- previous peppers verify-only

## 53. No infinite pepper retention

Debe existir migration/retirement policy.

## 54. Password Hash Migration

Flujo:

```text
verify with previous pepper
        ↓
success
        ↓
```

rehash with active pepper

## 55. Key Ring

VoltStack deberá utilizar AuthenticationKeyRing.

## 56. AuthenticationKeyRing

interface AuthenticationKeyRingInterface
{
public function active(
AuthenticationKeyPurpose $purpose
): AuthenticationKeyReference;

public function verificationKeys(
AuthenticationKeyPurpose $purpose
): iterable;
}

## 57. Key Ring Semantics

Normalmente:

- one ACTIVE signer
- zero or more VERIFY_ONLY keys

## 58. Multiple Active Signing Keys

Solo si el protocolo/deployment lo requiere explícitamente.

## 59. Key Selection

No deberá depender de iteration order.

## 60. ActiveKeySelector

Debe ser determinista.

## 61. Key Ring Version

Podrá existir:

- KeyRingVersion
- para coordinación cluster.

## 62. Distributed Key Ring

Todos los nodes deben disponer de:

- active key metadata
- verification keys
- activation schedule
- consistente.

## 63. Safe Rotation Sequence

Para signing keys:
64. Generate new key
65. Distribute public/verification material
66. Verify all nodes can read it
67. Mark PENDING
68. Activate new signing key
69. Old key → VERIFY_ONLY
70. Wait maximum artifact lifetime
71. Retire old key
72. Destroy when safe
73. Regla importante

La nueva clave de firma no deberá activarse hasta que todos los nodos que necesiten verificarla tengan acceso al material de verificación correspondiente.

## 74. Encryption Key Rotation

Más compleja.

## 75. Strategy A — Re-encrypt

decrypt old
encrypt new
para datos persistidos.

## 76. Strategy B — Envelope Encryption

Preferible para ciertos casos.

## 77. Envelope Encryption

Data
↓
Data Encryption Key
↓
Encrypted Data

+;

Encrypted DEK using Master Key

## 69. Master Key Rotation

Puede rotar wrapping key sin volver a cifrar todo el payload.

## 70. KMS Envelope Encryption

Muy apropiado para:

- tenant federation secrets
- sensitive configuration
- device secrets

## 71. Session Key Rotation

Sessions antiguas pueden verificarse con previous key hasta expiry.

## 72. Session Encryption

Si la Session cookie almacena estado cifrado, debe incluir:

- key identifier
- format version

## 73. Cookie Key ID

No secret.

## 74. Remember-Me Key Rotation

Similar.
Pero persistent credential lifetime puede ser mayor.

## 75. Token Key Rotation

JWT-like token deberá poder incluir:

- kid
- para seleccionar verificador.

## 76. kid handling

Nunca deberá utilizarse directamente como:

- file path
- URL
- database query injection

## 77. Known Key Registry

kid debe resolverse contra registry validado.

## 78. Unknown kid

Puede disparar bounded key refresh si protocolo aplica.

## 79. No unbounded refresh

Attacker puede enviar miles de random kid.

- Debe existir:
- rate limiting
- negative cache
- single-flight
- refresh budget

## 80. Algorithm Policy

VoltStack necesitará:
AuthenticationCryptographicAlgorithmPolicy

## 81. Algorithm Policy

Decide:

- allowed signing algorithms
- allowed encryption algorithms
- minimum key sizes
- deprecated algorithms
- forbidden algorithms

## 82. No algorithm from attacker

El algoritmo indicado por un token no deberá decidir por sí solo qué algoritmo aceptar.

## 83. Algorithm Confusion Resistance

Configured allowed algorithm:
EdDSA
Token says:
HS256
Resultado:
reject

## 84. none

No permitido salvo un protocolo muy específico que explícitamente lo requiriera, lo cual no aplicará a Authentication tokens normales.

## 85. Cryptographic Agility

Algoritmos deberán poder cambiar sin cambiar el dominio Authentication.

## 86. CryptographicAlgorithm enum/VO

No hardcodear verificaciones por todo el código.

## 87. Algorithm profiles

Ejemplo conceptual:

- DEFAULT
- FIPS_COMPATIBLE
- HIGH_SECURITY
- LEGACY_MIGRATION

## 88. Legacy Profile

Solo para verificar artefactos antiguos controladamente.
No emitir nuevos artefactos débiles.

## 89. Minimum Security

LegacyMigration no deberá reducir:

- new credential issuance
- new token signing

## 90. Certificate System

VoltStack deberá gestionar:

- X.509 certificates
- certificate chains
- trust anchors
- revocation state
- expiration
- purpose

cuando Authentication lo requiera.

## 91. Use Cases

mTLS
device certificates
enterprise smart cards
client certificate authentication
OIDC provider HTTPS validation

## 92. CertificateIdentity

Debe distinguirse de Authentication Identity.

## 93. Certificate Authentication

Certificate
↓
Chain Validation
↓
Trust Anchor
↓
Certificate Policy
↓
Certificate Identity Mapping
↓
Authentication Evidence

## 94. Certificate Trust Store

interface AuthenticationTrustStoreInterface
{
public function trustAnchors(
TrustPurpose $purpose,
AuthenticationScope $scope
): iterable;
}

## 95. Trust Purpose

Ejemplos:

- CLIENT_CERTIFICATE
- DEVICE_CERTIFICATE
- FEDERATION_SIGNING
- ENTERPRISE_CA

## 96. Trust Anchor Scope

Puede ser:

- platform
- tenant
- realm
- provider

## 97. Tenant Trust Store

Tenant enterprise puede tener CA propia.

## 98. Platform Floor

Tenant no debe poder introducir trust anchors en realms donde platform lo prohíba.

## 99. Certificate Validation

Debe verificar cuando corresponda:

- chain
- validity dates
- key usage
- extended key usage
- hostname/name constraints
- policy constraints
- revocation

## 100. mTLS

Client certificate validation deberá estar separada de TLS termination concern.

## 101. Trusted Proxy

Si TLS termina en proxy, certificado reenviado solo podrá aceptarse desde:

- trusted gateway
- authenticated internal channel
- signed metadata

## 102. Never trust arbitrary headers

No:

- X-Client-Cert
- desde Internet abierta.

## 103. Certificate Revocation

Podrá considerar:

- CRL
- OCSP
- internal revocation repository
- short-lived certificates
- según deployment.

## 104. OCSP Failure

Policy deberá definir:

- hard fail
- soft fail
- stapled response
- según realm.

## 105. Privileged Realm

Puede requerir hard fail.

## 106. Certificate Expiry

Nunca deberá ignorarse por cache.

## 107. Trust Store Versioning

Cada trust store deberá poder tener:
TrustStoreVersion

## 108. Trust Anchor Rotation

Debe ser cluster-safe.

## 109. CA Rotation

Puede requerir overlap:

```text
Old CA trusted
New CA trusted
↓
new certificates issued
↓
```

old CA removed after transition

## 110. JWKS

OIDC/federation utiliza JSON Web Key Sets.

## 111. JWKS Source

Puede ser:

- static
- remote
- cached remote
- tenant-specific

## 112. JWKS Cache

Debe incluir:

- issuer binding
- fetchedAt
- expiresAt
- etag/version where available

## 113. Issuer Binding

Nunca compartir JWKS entre issuers basándose únicamente en kid.

## 114. Correct key

Issuer + kid

## 115. JWKS Refresh

Al unknown kid:
bounded refresh

## 116. Stale Known-Good Keys

Puede permitirse temporalmente según policy.

## 117. New Unknown Key + Provider Down

No aceptar token.

## 118. JWKS Poisoning

Metadata/JWKS URL deberá provenir de trusted provider configuration/discovery.
No de claims del token.

## 119. SSRF Resistance

No usar:

- token-provided URL
- para descargar keys.

## 120. OIDC Discovery

Issuer configurado deberá determinar endpoint esperado.

## 121. Discovery Cache

Debe ser versionado/bounded.

## 122. Federation Trust Profile

Puede incluir:

- issuer
- allowed algorithms
- JWKS source
- key refresh
- clock skew
- TLS requirements

## 123. WebAuthn Trust

Passkeys normalmente verifican public key credentials sin CA tradicional.
Pero attestation puede involucrar trust.

## 124. Attestation

Si la aplicación utiliza attestation, deberá existir:
WebAuthnAttestationTrustPolicy

## 125. Attestation Modes

NONE
INDIRECT
DIRECT
ENTERPRISE
según requisitos/protocolo.

## 126. Trust Metadata

Puede integrarse con trusted metadata services.

## 127. Attestation no debe ser requerida por default

Passkey Authentication puede operar sin usar attestation como device trust universal.

## 128. Device Certificates

Si Device Trust utiliza certificates:

```text
certificate credential
    ↓
validation
    ↓
device evidence
```

Trust evaluator decide semántica.

## 129. Signing vs Encryption Keys

Nunca asumir que una clave puede utilizarse para ambos.

## 130. Key Usage Policy

Descriptor debe declarar:

- SIGN
- VERIFY
- ENCRYPT
- DECRYPT
- WRAP
- UNWRAP
- MAC

## 131. Key Usage Enforcement

Provider deberá rechazar operaciones fuera de uso permitido.

## 132. Private/Public Material

Private keys deberán tener controles más fuertes.

## 133. Public Keys

Pueden cachearse ampliamente.

## 134. Private Key Exportability

Metadata:

- EXPORTABLE
- NON_EXPORTABLE
- HSM_BOUND
- KMS_MANAGED

## 135. High Security

Preferir:

- NON_EXPORTABLE
- para signing keys críticas.

## 136. Tenant Cryptographic Keys

Algunos tenants pueden necesitar claves propias.

## 137. Tenant Key Scope

TenantId
RealmId if applicable
Purpose

## 138. Cross-Tenant Key Reuse

Evitar para secretos tenant-specific críticos cuando isolation lo requiera.

## 139. Platform Master Keys

Pueden envolver tenant keys.

## 140. Key Hierarchy

Conceptualmente:

```text
Platform Master Key
        ↓
```

Tenant Key Encryption Key
↓
Tenant Data/Secret Keys

## 141. Blast Radius

Uno de los objetivos de key hierarchy es limitar blast radius.

## 142. Single Global Key

Tiene blast radius enorme.

## 143. Per-Tenant Keys

Mejor aislamiento, mayor complejidad operacional.

## 144. Configurable Profiles

GLOBAL_KEYS
TENANT_DERIVED_KEYS
TENANT_MANAGED_KEYS
EXTERNAL_KMS_PER_TENANT

## 145. Tenant-managed keys

Enterprise futuro podría permitir:

- BYOK
- Customer Managed Keys

## 146. BYOK

Debe definir:

- ownership
- availability
- rotation
- revocation
- tenant offboarding

## 147. BYOK outage

Puede impedir Authentication del tenant.
Availability policy explícita.

## 148. Key Derivation

Si VoltStack deriva subkeys deberá usar KDF adecuado.

## 149. Domain Separation Example

Conceptualmente:

```php
HKDF(
    master,
    context = "voltstack/auth/session-signing/v1"
)
```

## 150. No custom KDF

Usar primitives estándar.

## 151. Salt/Info

Deberán tener semántica clara.

## 152. Deterministic Derivation

Puede ser útil, pero master key compromise afecta todas derivadas.

## 153. HSM/KMS-backed derivation

Preferible en high-security profiles si aplica.

## 154. Random Key Generation

Siempre mediante CSPRNG.

## 155. PHP

Primitives apropiadas como:

```php
random_bytes()
para material aleatorio cuando no se use KMS/HSM.
```

## 156. No predictable IDs as keys

Nunca.

## 157. Key Length

Definida por algorithm policy.
158. No arbitrary truncation
159. Secret Generation Service

interface AuthenticationSecretGeneratorInterface
{
public function generate(
SecretGenerationProfile $profile
): SensitiveSecret;
}

## 160. Secret Profiles

Ejemplos:

- RECOVERY_TOKEN
- SESSION_SECRET
- DEVICE_SECRET
- REMEMBER_ME_SECRET
- OAUTH_STATE_SECRET

## 161. Entropy Requirements

Cada profile deberá declarar entropy mínima.

## 162. Encoding

Base64url/hex/etc. es representación, no entropía.

## 163. Recovery Token Hashing

Tokens de recuperación almacenables deberán guardarse como:

- hash/derived verifier
- cuando sea posible, no raw token.

## 164. Remember-Me Secret

Mismo principio.

## 165. API Opaque Token

Mismo principio.

## 166. Token Fingerprint

Puede guardar identificador público separado para lookup.

## 167. Secret Comparison

Utilizar constant-time primitives cuando aplique.

## 168. MAC verification

Debe ser constant-time.

## 169. Certificate fingerprint

No secret.

## 170. Key Compromise

Debe existir procedimiento first-class.

## 171. Compromise States

SUSPECTED
CONFIRMED
CONTAINED
RECOVERED

## 172. KeyCompromiseEvent

Podrá disparar:

- stop issuance
- revoke key
- activate emergency key
- invalidate sessions/tokens
- force reauthentication
- rotate secrets
- audit incident

## 173. Compromise Scope

Dependerá del purpose.

## 174. Session Signing Key Compromise

Puede requerir:
invalidate all sessions signed by key

## 175. Token Signing Key Compromise

Puede requerir:
revoke all tokens with kid/version

## 176. OIDC Client Secret Compromise

Requiere:

- rotate provider secret
- possibly coordinate external IdP

## 177. Pepper Compromise

Puede requerir:

- pepper rotation
- password rehash on future login
- risk response

pero no implica automáticamente plaintext passwords exposed.

## 178. Master Key Compromise

Blast radius potencialmente crítico.
Debe existir emergency plan.

## 179. KeyCompromiseResponsePolicy

interface AuthenticationKeyCompromiseResponsePolicyInterface
{
public function decide(
AuthenticationKeyDescriptor $key,
KeyCompromiseContext $context
): KeyCompromiseResponsePlan;
}

## 180. Key-to-Artifact Provenance

Para responder a compromisos, artefactos deberán conocer:

- kid
- key version
- cuando aplique.

## 181. Session Provenance

Puede registrar:

- signing key version
- encryption key version

## 182. Token Provenance

Naturalmente via kid/metadata.

## 183. Remember-Me Provenance

Credential family puede registrar key version.

## 184. Key Impact Analyzer

Podrá responder:
What artifacts were issued by key K7?

## 185. AuthenticationKeyImpactAnalyzer

Útil para incident response.

## 186. Rotation Scheduling

Cada key profile podrá definir:

- rotation interval
- maximum lifetime
- overlap period
- emergency rotation

## 187. No universal rotation interval

Depende de:

- purpose
- provider
- regulation
- artifact lifetime
- risk

## 188. Rotation Planner

interface AuthenticationKeyRotationPlannerInterface
{
public function plan(
AuthenticationKeyDescriptor $key
): AuthenticationKeyRotationPlan;
}

## 189. Rotation State Machine

GENERATE
↓
DISTRIBUTE
↓
VERIFY_AVAILABILITY
↓
ACTIVATE
↓
DEMOTE_OLD
↓
WAIT_OVERLAP
↓
RETIRE
↓
DESTROY

## 190. Rotation Failure

Debe quedar en estado consistente.

## 191. Example

Nueva key no pudo distribuirse a Node C.
No activar todavía.

## 192. Cluster Key Coordination

Documento 30 deberá integrarse.

## 193. Key Propagation Barrier

Opcional:

- required nodes/regions confirm readiness
- antes de activation.

## 194. Availability Tradeoff

No depender de todos los nodos muertos para siempre.

- Puede usar:
- deployment membership
- quorum policy
- expected active nodes

## 195. Node Join

Nuevo node deberá cargar key ring compatible antes de aceptar tráfico.

## 196. Node Health

Health check:

- KeyRingVersion expected?
- Active key present?
- Verification keys complete?
- TrustStoreVersion correct?

## 197. Node with missing keys

Debe marcarse:
NOT READY

## 198. Rolling Deployment

Key rotation y app deploy deben ser independientes siempre que sea posible.

## 199. Key schema compatibility

Old version debe reconocer new key metadata dentro de supported window.

## 200. Multi-region Key Distribution

Puede usar global KMS.

## 201. Region-local KMS

Si existen claves region-specific, artefactos deberán indicar scope.

## 202. Cross-region verification

Debe estar soportada o el artefacto deberá ser region-bound.

## 203. Key Residency

Enterprise/regulatory deployment puede exigir:
keys remain in region

## 204. Authentication Region Policy

Debe contemplar.

## 205. Key Backup

Solo claves que deban ser recoverable.

## 206. Backup vs Non-exportable

Algunas HSM keys no se exportan y se respaldan mediante provider replication.

## 207. Backup Security

Debe tener protección igual o superior al original.

## 208. Recovery Drill

High-security deployments deberían poder probar:

- key recovery
- disaster recovery
- sin exponer material.

## 209. No ad-hoc backup

No:
copy private-key.pem to desktop

## 210. Key Destruction

Debe existir lifecycle explícito.

## 211. Destroy Preconditions

Antes de destruir verification key:

- all dependent artifacts expired/revoked
- audit requirements satisfied
- recovery not needed

## 212. Destroying too early

Puede invalidar legítimamente:

- sessions
- tokens
- encrypted records

## 213. Cryptographic Erasure

Para ciertos datos:

- delete wrapping key
- puede hacer datos irrecuperables.

## 214. Audit

Key operations deberán auditarse.

## 215. Audit Events

KeyGenerated
KeyActivated
KeyRotated
KeyRetired
KeyRevoked
KeyDestroyed
SecretRotated
TrustAnchorAdded
TrustAnchorRemoved
CertificateRevoked

## 216. Actor

Especialmente:

- administrator
- automation
- KMS
- deployment system

## 217. Audit payload

Nunca raw key.

## 218. Example

KeyActivated
Purpose: TOKEN_SIGNING
KeyId: auth-token-k8
Version: 8
Actor: SECURITY_AUTOMATION

## 219. Metrics

Podrán incluir:

- auth_key_rotation_total
- auth_key_resolution_failure_total
- auth_key_version_mismatch_total
- auth_secret_provider_failure_total
- auth_jwks_refresh_total
- auth_certificate_validation_failure_total

## 220. No key material in labels

## 221. Tracing

Puede registrar:

- key provider
- key id
- operation type
- cuando seguro.

## 222. No private key bytes

## 223. Secret Redaction

Documento 24 debe tratar tipos:

- SensitiveSecret
- SensitiveKeyMaterial
- PrivateKeyHandle
- ClientSecret
- Pepper
- como siempre redacted.

## 224. Exception Safety

No:
throw new Exception("Failed key: {$rawPrivateKey}");

## 225. Secret Value Objects

Deberán evitar exportación accidental.

## 226. Serialization Guard

SensitiveSecret podrá lanzar exception ante serialización no permitida.

## 227. Secret Access API

Idealmente:

```php
$secret->use(
    fn (string $value) => ...
);
```

para limitar lifecycle conceptual.

## 228. PHP limitation

No garantiza que closures eliminen copias, pero mejora diseño.

## 229. Config Debugging

auth:inspect deberá mostrar:

- secret reference configured
- key provider configured
- key version
- status
- no contenido.

## 230. CLI Key Management

Podrán existir comandos:

- php volt auth:key:list
- php volt auth:key:generate
- php volt auth:key:rotate
- php volt auth:key:revoke
- php volt auth:key:inspect
- php volt auth:trust:list
- php volt auth:trust:validate

## 231. Dangerous Operations

Como:

- key:revoke
- key:destroy

deberán requerir explicit safeguards.

## 232. No secret output

key:generate no debería imprimir private key por default si provider puede almacenarla directamente.

## 233. Generated Local Keys

Si developer mode necesita exportarlas:

- explicit development command
- secure file permissions

## 234. Production Key Management

Preferir provider-backed lifecycle.

## 235. Environment Separation

Nunca compartir:

- production keys
- development keys
- test keys

## 236. KeyEnvironment

Metadata:

- development
- testing
- staging
- production

## 237. Production Validation

Framework deberá detectar ciertos errores obvios:

- test key configured in production
- known default secret
- short development key
- cuando pueda.

## 238. Default Secrets

VoltStack no deberá incluir secretos universales.

## 239. Bootstrap

Si key obligatoria falta:

- fail bootstrap
- para subsystem correspondiente.

## 240. Optional subsystem

Puede desactivarse si no requerido.

## 241. Key Availability Policy

Si KMS unavailable:

- SIGN operation
- VERIFY operation
- DECRYPT operation

pueden tener policies distintas.

## 242. Example

Session signing KMS unavailable:
cannot issue new session

## 243. Verification using cached public key

Puede seguir funcionando.

## 244. Asymmetric advantage

Verifier no necesita private key.

## 245. Symmetric key outage

Puede impedir tanto emisión como verification si key no está local.

## 246. Key Cache Availability

High availability architecture deberá contemplarlo.

## 247. KMS Timeout

No unlimited wait.
Debe respetar:
AuthenticationExecutionDeadline

## 248. Circuit Breaker

Puede aplicarse a secret/KMS providers.

## 249. Fail Open

Nunca validar signature como correcta porque KMS no respondió.

## 250. Secret Provider Outage

Debe producir:

- AUTH_CRYPTO_PROVIDER_UNAVAILABLE
- o equivalente normalizado.

## 251. Trust Provider Outage

OIDC JWKS service caído puede usar:

- still-valid known-good cached keys
- según policy.

## 252. Unknown key

No.

## 253. Certificate Trust Provider Outage

Policy explícita.

## 254. Secret Rotation

Secret no siempre es key.

- Ejemplo:
- OIDC client secret
- API client secret

## 255. Secret Versioning

Puede tener:
SecretVersion

## 256. Dual-secret transition

External provider puede requerir overlap.

## 257. Client Secret Rotation

create new external secret
↓
store new SecretReference version
↓
switch application
↓
verify
↓
remove old external secret

## 258. Coordination

Debe permitir rollback si activación falla.

## 259. Secret Lease

Algunos secret managers entregan secrets dinámicos con lease.

## 260. Dynamic Credentials

Ejemplo:

- short-lived database credentials
- aunque no sean Authentication user credentials.

## 261. Lease Renewal

Debe estar fuera del hot path cuando posible.

## 262. Secret Expiry

Runtime debe detectar.
263. No stale secret indefinitely
264. Certificate Lifecycle

REQUEST
ISSUE
ACTIVE
RENEW
EXPIRE
REVOKE
DESTROY

## 265. Certificate Renewal

Debe ocurrir antes de expiry.

## 266. Expiring Certificate Alert

Metrics/health.

## 267. Certificate Health

Podrá reportar:

- expires in X
- chain valid
- revocation status

## 268. Key Health

Igual:

- active key age
- rotation due
- provider availability

## 269. AuthenticationCryptographicHealthService

interface AuthenticationCryptographicHealthServiceInterface
{
public function report(): AuthenticationCryptographicHealthReport;
}

## 270. Health Report

Puede incluir:

- ACTIVE
- DEGRADED
- ROTATION_DUE
- KEY_MISSING
- TRUST_STORE_STALE
- CERTIFICATE_EXPIRING

## 271. No secret values

## 272. Trust Store Freshness

Remote trust metadata puede tener freshness requirements.

## 273. Stale Trust

Privileged realm puede fail closed.

## 274. Certificate Pinning

No debe ser universal default.

## 275. Pinning Policy

Puede ser apropiada para specific machine/service integrations.

## 276. Pinning Rotation

Debe soportar overlap.

## 277. Public Key Pin

No hardcode sin lifecycle.

## 278. Key Revocation vs Retirement

Distinción:

```text
RETIREMENT
    planned lifecycle

REVOCATION
    security invalidation
```

## 279. Revoked key

No deberá volver a ACTIVE.

## 280. Monotonic Key State

REVOKED
es terminal salvo administrative correction model altamente controlado; preferiblemente crear key nueva.

## 281. Distributed Key State

Debe seguir principios del documento 30.

## 282. Out-of-order events

Evento viejo:
ACTIVE v5
no debe revertir:
REVOKED v6

## 283. Key State Version

Puede existir:
KeyMetadataVersion

## 284. Key Cache

Debe ser monotonic/version-aware.

## 285. Private Key Cache

Si se permite, cuidado especial.

## 286. Prefer handles

KMS:

- KeyHandle
- en vez de bytes.

## 287. Local Private Keys

Podrán cargarse en memory.

## 288. File Permissions

Private key file:

- not web-accessible
- restricted permissions
- outside public directory

## 289. Container Images

No bakear production secrets en image.

## 290. Git

Nunca versionar private keys/secrets.

## 291. Secret Scanning

CI debería integrar detección de secrets.

## 292. Testing

Documento 26 deberá incluir crypto-specific suites.

## 293. Testing — Key Lifecycle

Verificar:

- PENDING cannot sign
- ACTIVE can sign
- VERIFY_ONLY cannot sign

REVOKED cannot verify where policy requires revoke

## 294. Testing — Rotation

Artefacto firmado con old key sigue verificando durante overlap.

## 295. Testing — Retirement

Después de artifact max lifetime, old key puede retirarse.

## 296. Testing — Unknown kid

Debe rechazar.

## 297. Testing — Algorithm Confusion

Debe rechazar algorithm no permitido.

## 298. Testing — alg=none

Debe rechazar.

## 299. Testing — Key Usage

Encryption key no puede firmar.

## 300. Testing — Cross-Purpose

Session key no verifica token signature.

## 301. Testing — Cross-Tenant

Tenant A key no descifra/verifica Tenant B artifact cuando scope lo impide.

## 302. Testing — Key Compromise

Revocar key debe invalidar artifacts según policy.

## 303. Testing — Cluster Rotation

Nodes antiguos reciben verifier antes de activation.

## 304. Testing — Missing Node Key

Node queda unhealthy/not ready.

## 305. Testing — KMS Outage

No signature bypass.

## 306. Testing — Secret Resolver Failure

Produce normalized Authentication Error.

## 307. Testing — Secret Redaction

Canary secret nunca aparece en:

- logs
- audit
- traces
- exceptions
- cache
- CLI

## 308. Testing — Compiled Cache

Secret value no aparece en generated Authentication cache.

## 309. Testing — Environment Separation

Test key rechazada en production profile cuando detectable.

## 310. Testing — Certificate Expiry

Expired cert rejected.

## 311. Testing — Wrong Key Usage

Certificate con EKU incorrecto rejected.

## 312. Testing — Wrong Trust Anchor

Rejected.

## 313. Testing — Tenant CA

CA Tenant A no autentica Tenant B.

## 314. Testing — JWKS Issuer Binding

Same kid, different issuer.
Debe seleccionar por issuer.

## 315. Testing — JWKS Refresh Flood

Random kid no provoca unlimited network calls.

## 316. Testing — JWKS Stale

Known-good key behavior según policy.

## 317. Testing — OIDC Provider Down

Unknown key not accepted.

## 318. Testing — Key Cache Monotonicity

v8 REVOKED
then stale v7 ACTIVE
permanece revoked.

## 319. Testing — Secret Cache Bounds

No crecimiento ilimitado en FrankenPHP.

## 320. Testing — Worker Rotation Refresh

Worker viejo descubre nueva key ring version.

## 321. Testing — Key Destroy Preconditions

No destruir key todavía requerida.

## 322. Testing — Envelope Encryption

Rotar KEK no cambia plaintext.

## 323. Testing — Recovery after restart

Persisted encrypted secret sigue disponible con correct key ring.

## 324. Testing — Wrong Key Version

Debe fallar deterministicamente.

## 325. Testing — Crypto Fault Injection

Simular:

- KMS timeout
- KMS permission denied
- corrupted ciphertext
- invalid signature
- missing key

## 326. Testing — Concurrency

Dos procesos intentan rotar misma key.
Solo un rotation plan debe activarse.

## 327. Rotation Lock

Puede requerir distributed lock/transaction.
328. But Authentication verification should not depend on rotation lock.
329. Testing — Multi-region Rotation

Region B puede verificar nueva key antes de activation.

## 330. Testing — Trust Store Rotation

Old/new CA overlap correcto.

## 331. Testing — Certificate Revocation

Revoked cert rejected según policy.

## 332. Testing — Fail Closed

Admin realm no autentica si required trust validation unavailable.

## 333. Security Invariants — Keys

AUTH-CRYPTO-KEY-01
Every Authentication cryptographic key has an explicit purpose.
AUTH-CRYPTO-KEY-02
Keys for unrelated purposes are not reused indiscriminately.

- AUTH-CRYPTO-KEY-03
- Key identifiers are distinct from secret key material.
- AUTH-CRYPTO-KEY-04

Private key material is never logged or included in public diagnostics.
AUTH-CRYPTO-KEY-05
Key lifecycle state is explicit and versioned.

## 334. Security Invariants — Secrets

AUTH-CRYPTO-SECRET-01
Authentication configuration prefers Secret References over raw secret values.
AUTH-CRYPTO-SECRET-02
Compiled Authentication configuration does not expose raw secrets.
AUTH-CRYPTO-SECRET-03
Secrets are never serialized into events, metrics, traces or audit payloads.
AUTH-CRYPTO-SECRET-04
Secret caches are bounded.
AUTH-CRYPTO-SECRET-05
Secret provider failure never causes cryptographic verification to succeed implicitly.

## 335. Security Invariants — Algorithms

AUTH-CRYPTO-ALG-01
Accepted cryptographic algorithms come from trusted policy, not attacker-controlled metadata.
AUTH-CRYPTO-ALG-02
Deprecated algorithms cannot be used for new artifact issuance unless explicitly required by a migration profile.
AUTH-CRYPTO-ALG-03
Legacy verification does not imply legacy issuance.

- AUTH-CRYPTO-ALG-04
- Key size and algorithm requirements are validated before use.
- AUTH-CRYPTO-ALG-05

Algorithm confusion attempts are rejected.

## 336. Security Invariants — Rotation

AUTH-CRYPTO-ROT-01
A new signing key is not activated before required verifiers can access it.
AUTH-CRYPTO-ROT-02
Old verification keys remain available only for the required overlap window.
AUTH-CRYPTO-ROT-03
Key rotation state transitions are deterministic and auditable.
AUTH-CRYPTO-ROT-04
Concurrent rotation cannot activate conflicting keys accidentally.
AUTH-CRYPTO-ROT-05
Revoked keys cannot return to active service through stale distributed state.

## 337. Security Invariants — Trust

AUTH-CRYPTO-TRUST-01
Trust anchors are scoped by explicit trust purpose.

- AUTH-CRYPTO-TRUST-02
- Tenant trust anchors cannot cross Tenant boundaries unintentionally.
- AUTH-CRYPTO-TRUST-03

OIDC verification keys are bound to their configured issuer.
AUTH-CRYPTO-TRUST-04
Remote key URLs are derived from trusted configuration, not untrusted token content.
AUTH-CRYPTO-TRUST-05
Certificate validation checks the properties required by the active trust policy.

## 338. Security Invariants — Certificates

AUTH-CRYPTO-CERT-01
Expired certificates cannot authenticate.

- AUTH-CRYPTO-CERT-02
- Revoked certificates follow configured revocation policy.
- AUTH-CRYPTO-CERT-03

Certificates with incompatible key usage are rejected.
AUTH-CRYPTO-CERT-04
Forwarded client certificate metadata is trusted only from explicitly trusted infrastructure.
AUTH-CRYPTO-CERT-05
Certificate trust stores are versioned and observable.

## 339. Security Invariants — Distributed Runtime

AUTH-CRYPTO-DIST-01
All compatible nodes know verification material before new signing keys become active.
AUTH-CRYPTO-DIST-02
Key and trust-store version drift is detectable.

- AUTH-CRYPTO-DIST-03
- Stale key metadata cannot override newer revoked state.
- AUTH-CRYPTO-DIST-04

New nodes do not accept Authentication traffic until required key material is available.
AUTH-CRYPTO-DIST-05
Multi-region key semantics are explicit.

## 340. Security Invariants — FrankenPHP

AUTH-CRYPTO-RT-01
Long-lived workers never expose cached secrets between requests.
AUTH-CRYPTO-RT-02
Worker-local key/secret caches are bounded and rotation-aware.
AUTH-CRYPTO-RT-03
Request-local secret references do not become mutable global state.
AUTH-CRYPTO-RT-04
Key refresh can occur without restarting the entire application when the configured provider supports dynamic rotation.
AUTH-CRYPTO-RT-05
Debugging long-lived workers never dumps key material.

## 341. Anti-pattern — One APP_KEY for everything

No.

## 342. Anti-pattern — Private key inside Git repository

Nunca.

## 343. Anti-pattern — Raw secret inside compiled PHP cache

No.

## 344. Anti-pattern — JWT algorithm accepted from token header alone

No.

## 345. Anti-pattern — Unknown kid triggers arbitrary URL request

Nunca.

## 346. Anti-pattern — Same kid trusted across issuers

No.

## 347. Anti-pattern — Activate key before cluster propagation

No.

## 348. Anti-pattern — Delete previous key immediately after rotation

No si existen artefactos aún válidos.

## 349. Anti-pattern — Retired key still issuing new tokens

No.

## 350. Anti-pattern — Revoked key reactivated from stale cache

Nunca.

## 351. Anti-pattern — KMS outage means skip signature verification

Nunca.

## 352. Anti-pattern — OIDC client secret printed in auth:inspect

No.

## 353. Anti-pattern — Tenant secrets stored unencrypted with ordinary config

No para production profile.

## 354. Anti-pattern — Certificate forwarded header accepted from any proxy

No.

## 355. Anti-pattern — Certificate validity dates ignored

No.

## 356. Anti-pattern — WebAuthn attestation treated as mandatory device trust universally

No.

## 357. Anti-pattern — Password pepper stored beside password hashes without purpose

No.

## 358. Anti-pattern — Custom encryption/signature primitives

No.

## 359. Anti-pattern — Unlimited private key cache under FrankenPHP

No.

## 360. Componentes principales

AuthenticationKeyDescriptor
CryptographicKeyId
KeyVersion
AuthenticationKeyPurpose
AuthenticationKeyStatus
AuthenticationKeyReference
AuthenticationKeyRing
KeyRingVersion

SensitiveKeyMaterial
SensitiveSecret
SecretReference
SecretVersion

## 361. Provider Components

AuthenticationSecretResolver
AuthenticationSecretProvider
AuthenticationCryptographicProvider
AuthenticationEncryptionProvider
AuthenticationMacProvider

KmsAuthenticationCryptographicProvider
HsmAuthenticationCryptographicProvider
LocalAuthenticationCryptographicProvider

## 362. Policy Components

AuthenticationCryptographicAlgorithmPolicy
AuthenticationKeyUsagePolicy
AuthenticationKeyRotationPolicy
AuthenticationKeyCompromiseResponsePolicy
AuthenticationTrustPolicy
AuthenticationCertificateValidationPolicy

## 363. Trust Components

AuthenticationTrustStore
TrustStoreVersion
TrustAnchor
TrustPurpose

FederationTrustProfile
JwksProvider
JwksCache
WebAuthnAttestationTrustPolicy
CertificateTrustProvider

## 364. Lifecycle Components

AuthenticationKeyGenerator
AuthenticationKeyRotationPlanner
AuthenticationKeyRotationCoordinator
AuthenticationKeyRevocationService
AuthenticationKeyDestructionService
AuthenticationKeyImpactAnalyzer

## 365. Health Components

AuthenticationCryptographicHealthService
AuthenticationCryptographicHealthReport
AuthenticationKeyRingHealth
AuthenticationCertificateHealth
AuthenticationTrustStoreHealth

## 366. Namespace sugerido

VoltStack\Quantum\Auth\Crypto
VoltStack\Quantum\Auth\Crypto\Contracts
VoltStack\Quantum\Auth\Crypto\Key
VoltStack\Quantum\Auth\Crypto\Secret
VoltStack\Quantum\Auth\Crypto\Provider
VoltStack\Quantum\Auth\Crypto\Algorithm
VoltStack\Quantum\Auth\Crypto\Rotation
VoltStack\Quantum\Auth\Crypto\Trust
VoltStack\Quantum\Auth\Crypto\Certificate
VoltStack\Quantum\Auth\Crypto\Federation
VoltStack\Quantum\Auth\Crypto\Health
VoltStack\Quantum\Auth\Crypto\Runtime

## 367. Estructura sugerida

src/Quantum/Auth/Crypto/
├── Contracts/
│   ├── AuthenticationKeyRingInterface.php
│   ├── AuthenticationSecretResolverInterface.php
│   ├── AuthenticationCryptographicProviderInterface.php
│   ├── AuthenticationEncryptionProviderInterface.php
│   ├── AuthenticationTrustStoreInterface.php
│   ├── AuthenticationKeyRotationPlannerInterface.php
│   └── AuthenticationKeyCompromiseResponsePolicyInterface.php
│
├── Key/
│   ├── AuthenticationKeyDescriptor.php
│   ├── CryptographicKeyId.php
│   ├── KeyVersion.php
│   ├── AuthenticationKeyPurpose.php
│   ├── AuthenticationKeyStatus.php
│   ├── AuthenticationKeyReference.php
│   ├── AuthenticationKeyRing.php
│   └── KeyRingVersion.php
│
├── Secret/
│   ├── SensitiveSecret.php
│   ├── SensitiveKeyMaterial.php
│   ├── SecretReference.php
│   ├── SecretVersion.php
│   ├── AuthenticationSecretResolver.php
│   └── AuthenticationSecretCache.php
│
├── Provider/
│   ├── LocalAuthenticationCryptographicProvider.php
│   ├── KmsAuthenticationCryptographicProvider.php
│   ├── HsmAuthenticationCryptographicProvider.php
│   └── AuthenticationSecretProviderRegistry.php
│
├── Algorithm/
│   ├── CryptographicAlgorithm.php
│   ├── AuthenticationCryptographicAlgorithmPolicy.php
│   ├── AuthenticationKeyUsagePolicy.php
│   └── CryptographicSecurityProfile.php
│
├── Rotation/
│   ├── AuthenticationKeyRotationPlan.php
│   ├── AuthenticationKeyRotationPlanner.php
│   ├── AuthenticationKeyRotationCoordinator.php
│   ├── AuthenticationKeyRevocationService.php
│   ├── AuthenticationKeyDestructionService.php
│   └── AuthenticationKeyImpactAnalyzer.php
│
├── Trust/
│   ├── AuthenticationTrustStore.php
│   ├── TrustStoreVersion.php
│   ├── TrustAnchor.php
│   ├── TrustPurpose.php
│   └── AuthenticationTrustPolicy.php
│
├── Certificate/
│   ├── AuthenticationCertificateValidator.php
│   ├── AuthenticationCertificateValidationPolicy.php
│   ├── CertificateIdentity.php
│   ├── CertificateRevocationPolicy.php
│   └── AuthenticationCertificateHealth.php
│
├── Federation/
│   ├── FederationTrustProfile.php
│   ├── JwksProvider.php
│   ├── JwksCache.php
│   ├── JwksKeyResolver.php
│   └── WebAuthnAttestationTrustPolicy.php
│
├── Health/
│   ├── AuthenticationCryptographicHealthService.php
│   ├── AuthenticationCryptographicHealthReport.php
│   ├── AuthenticationKeyRingHealth.php
│   └── AuthenticationTrustStoreHealth.php
│
└── Runtime/
├── AuthenticationCryptoExecutionContext.php
├── AuthenticationSecretCache.php
└── AuthenticationCryptoRuntimeResetter.php

## 368. Configuración conceptual

return [

'authentication' => [

'crypto' => [

'profile' => 'default',

'provider' => 'kms',

'keys' => [

'session_signing' => [
'ring' => 'auth-session',
],

'token_signing' => [
'ring' => 'auth-token',
],

],

'secrets' => [
'provider' => 'vault',
],

],

],

];

## 369. Key Ring conceptual

'key_rings' => [

'auth-token' => [

'purpose' => 'token_signing',

'active' => 'token-k8',

'verify' => [
'token-k8',
'token-k7',
],

],

];

## 370. Secret Reference conceptual

'oidc' => [

'client_id' => 'voltstack',

'client_secret' => [
'secret' => 'tenant/acme/oidc/client-secret',
],

];

## 371. Tenant KMS conceptual

'tenant_keys' => [

'strategy' => 'kms',

'key_resolver' => TenantKeyResolver::class,

];

## 372. Signing Flow

Authentication subsystem
↓
Key Purpose
↓
Key Ring
↓
ACTIVE Key Reference
↓
Crypto Provider
↓
Signature
↓
Artifact with Key ID

## 373. Verification Flow

Artifact
↓
Key ID
↓
Validated Key Registry
↓
Verification Key
↓
Algorithm Policy
↓
Signature Verification
↓
Valid / Invalid

## 374. Key Rotation Flow

Rotation Due
↓
Generate New Key
↓
PENDING
↓
Distribute Verification Material
↓
Cluster Readiness Check
↓
Activate New Key
↓
Old Key → VERIFY_ONLY
↓
Wait Artifact Lifetime
↓
Retire Old Key
↓
Destroy when Safe

## 375. Emergency Compromise Flow

Key Compromise Detected
↓
Mark Key REVOKED
↓
Stop New Issuance
↓
Activate Emergency Key
↓
Determine Affected Artifacts
↓
Security Epoch / Revocation
↓
Force Reauthentication if required
↓
Audit + Incident Response

## 376. OIDC JWKS Flow

OIDC Token
↓
Issuer
↓
Federation Trust Profile
↓
kid
↓
Issuer-bound JWKS Cache
↓
Known Key?
┌──┴───┐
▼      ▼
YES     NO
│       │
verify  bounded refresh
│
found?
┌─┴─┐
▼   ▼
YES  NO
│   │
verify reject

## 377. Certificate Authentication Flow

Client Certificate
↓
Certificate Parser
↓
Expected Realm/Tenant Trust Policy
↓
Trust Store
↓
Chain Validation
↓
Validity / Usage / Revocation
↓
Certificate Identity Mapper
↓
Authentication Evidence

## 378. Envelope Encryption Flow

Sensitive Tenant Secret
↓
Generate Data Key
↓
Encrypt Secret
↓
Wrap Data Key with KMS Key
↓
Store:

```text
  Ciphertext
  Wrapped DEK
  Key ID
```

## 379. FrankenPHP Model

WORKER BOOT
│
▼
Load immutable key metadata
│
├── Key handles
├── Key Ring versions
└── Trust metadata
│
▼
REQUEST
│
▼
Resolve secret/key when needed
│
▼
Use bounded runtime cache
│
▼
REQUEST END
│
▼
Clear request-local crypto context

## 380. Relación con Laravel

Laravel ofrece una gran simplicidad mediante:

- APP_KEY
- encrypted configuration
- cookie encryption/signing
- hashing facilities
- password hashing
- encrypter

Ese modelo funciona muy bien para una gran cantidad de aplicaciones.
VoltStack, sin embargo, busca un Authentication System más amplio que debe soportar:

- multiple key purposes
- multiple tenants
- key rings
- rotation
- distributed nodes
- OIDC/JWKS
- certificates
- KMS/HSM

por lo que no conviene basar todo Authentication en una única clave de aplicación.

## 381. Relación con Symfony

Symfony aporta un diseño fuerte mediante componentes y contracts para:

- Secrets
- PasswordHasher
- Security
- HttpClient
- Certificate/TLS integrations

además de una filosofía compatible con proveedores externos y servicios especializados.
VoltStack deberá conservar esa separación contractual y combinarla con key lifecycle explícito.

## 382. Diferenciador VoltStack

VoltStack combinará:

- Laravel-like developer simplicity
- +;
- Symfony-like service abstraction
- +;
- purpose-bound key management
- +;
- KMS/HSM integration
- +;
- tenant-aware trust
- +;
- distributed rotation
- (+)
- FrankenPHP-safe secret lifecycle

El developer podrá configurar:

```php
'crypto' => [
    'provider' => 'kms',
]
```

mientras internamente VoltStack mantiene:

- Key Purpose
- Key Ring
- Key Version
- Key Status
- Provider
- Algorithm Policy
- Rotation Policy
- Trust Scope
- Cluster Version

## 383. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Keys and user credentials are separate security domains

## 2. Cryptographic keys have explicit purpose, identity, version and lifecycle

## 3. Key reuse across unrelated Authentication purposes is discouraged/prohibited

## 4. Raw secrets are replaced by Secret References in configuration

## 5. Compiled Authentication caches do not contain raw secrets by default

## 6. Key Rings support active and verification-only generations

## 7. Key rotation is a first-class lifecycle

## 8. Signing keys are distributed before activation

## 9. Revoked keys cannot return to active state from stale distributed metadata

## 10. Algorithm policy is authoritative over attacker-provided algorithm metadata

## 11. Vendor/KMS/HSM integrations remain behind Authentication crypto contracts

## 12. Tenant trust/key scope is explicit

## 13. OIDC JWKS keys are issuer-bound

## 14. Remote key retrieval is based on trusted configuration, never arbitrary token URLs

## 15. Certificate trust and certificate identity mapping are separate

## 16. Key compromise response is first-class

## 17. Key provenance is preserved on issued Authentication artifacts where applicable

## 18. Crypto providers respect Authentication execution deadlines

## 19. Worker caches are bounded and rotation-aware

## 20. Authentication never succeeds because cryptographic infrastructure failed

## 21. Criterios de aceptación

El subsistema será considerado completo cuando:
22. soporte typed Key IDs;
23. soporte Key Versions;
24. soporte explicit Key Purpose;
25. soporte Key Status;
26. soporte Key Usage;
27. soporte Key Rings;
28. soporte Key Ring Versions;
29. soporte Secret References;
30. soporte Secret Providers;
31. soporte bounded Secret Cache;
32. soporte local cryptographic provider;
33. soporte KMS adapters;
34. soporte HSM adapters;
35. soporte signing;
36. soporte signature verification;
37. soporte encryption;
38. soporte MAC;
39. soporte Password Pepper opcional;
40. soporte Pepper rotation;
41. soporte Algorithm Policy;
42. soporte cryptographic agility;
43. soporte legacy verification profile;
44. impida legacy weak issuance;
45. soporte key generation;
46. soporte planned rotation;
47. soporte emergency rotation;
48. soporte key retirement;
49. soporte key revocation;
50. soporte key destruction;
51. soporte Key Impact Analysis;
52. soporte cluster-aware activation;
53. soporte multi-region key metadata;
54. soporte key-set drift detection;
55. soporte certificate trust;
56. soporte CA stores;
57. soporte certificate expiry validation;
58. soporte certificate revocation policies;
59. soporte mTLS integration;
60. soporte tenant-specific trust anchors;
61. soporte JWKS;
62. soporte issuer-bound JWKS cache;
63. soporte bounded unknown-kid refresh;
64. soporte OIDC trust profiles;
65. soporte WebAuthn attestation trust policy;
66. soporte Device Certificate trust;
67. soporte envelope encryption;
68. soporte tenant key strategies;
69. soporte BYOK/extensible tenant keys;
70. soporte Key Health;
71. soporte Certificate Health;
72. soporte Trust Store Health;
73. soporte cryptographic audit;
74. soporte crypto metrics;
75. soporte secret-safe tracing;
76. soporte fault injection;
77. soporte secret leakage testing;
78. soporte cluster rotation testing;
79. sea seguro bajo FrankenPHP;
80. sea fiber-safe;
81. nunca haga fail-open ante errores criptográficos.
82. Regla arquitectónica final

VoltStack deberá mantener:

```text
                   AUTHENTICATION ARTIFACT
                            │
                            ▼
                     CRYPTO PURPOSE
                            │
                            ▼
                        KEY RING
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
           ACTIVE KEY             VERIFY-ONLY KEYS
                │                       │
                └───────────┬───────────┘
                            ▼
                    CRYPTO PROVIDER
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
             LOCAL         KMS          HSM
                            │
                            ▼
                       TRUST SYSTEM
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
             JWKS        CERTIFICATES   TENANT CA
```

La primera regla será:
VoltStack no deberá utilizar una única clave global como solución universal para todos los mecanismos de Authentication; las claves deberán estar ligadas a propósitos criptográficos explícitos.

La segunda:
Una key rotation segura requiere overlap controlado: primero distribuir capacidad de verificación, luego activar la nueva key y solo después retirar la anterior cuando ya no existan artefactos legítimos que dependan de ella.

La tercera:
Los secretos reales deberán permanecer fuera de configuración compilada, logs, eventos, trazas, auditoría y herramientas de diagnóstico; el sistema operará preferentemente mediante SecretReference, key handles y proveedores especializados.

La cuarta:
Una falla de KMS, HSM, JWKS, Trust Store o Secret Provider nunca convertirá una verificación criptográfica pendiente en una verificación exitosa.

La quinta:
Toda confianza criptográfica deberá estar explícitamente delimitada por purpose, issuer, tenant, realm o provider según corresponda; una clave o certificado válido en un contexto no será automáticamente válido en otro.

La sexta:
VoltStack deberá ser criptográficamente ágil: algorithms, key providers, trust anchors y formatos podrán evolucionar sin modificar la semántica central del sistema Authentication.

Siguiente documento recomendado
La secuencia natural continúa con:
`32_AUTHENTICATION_PRIVILEGED_ADMINISTRATIVE_BREAK_GLASS_SENSITIVE_OPERATION_AND_REAUTHENTICATION_SYSTEM.md`
Este documento permitiría formalizar un área crítica distinta de la autenticación normal:

- Privileged Authentication
- Administrative Authentication
- Sensitive Operations
- Fresh Authentication
- Reauthentication

Step-Up for Sensitive Actions
Break-Glass Accounts
Emergency Access
Root / Platform Administrator Authentication
Tenant Administrator Authentication
Privileged Sessions
Privileged Session Lifetime
Privileged Device Requirements
Phishing-Resistant Requirements
Dual Control
Administrative Recovery
Sensitive Credential Changes
Change Password Verification
Change MFA Verification
Disable MFA Verification
Add Passkey Verification
API Credential Creation
Security Policy Changes
Global Logout Confirmation
Impersonation Entry
Privilege Escalation Boundaries
Break-Glass Audit
Emergency Credential Rotation
La regla central sería que estar autenticado no significa estar suficientemente autenticado para cualquier operación. Una sesión válida puede seguir necesitando fresh authentication, mayor assurance o un factor phishing-resistant antes de ejecutar una acción de alto impacto.
