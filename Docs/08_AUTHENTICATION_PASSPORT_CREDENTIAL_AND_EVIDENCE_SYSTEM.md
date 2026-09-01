# VoltStack Authentication System

## 08 — Authentication Passport, Credential and Evidence System

- **Archivo:** `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica de Passport, Credentials y Evidence  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de transformar:

```text
datos de autenticación no confiables
        ↓
AuthenticationPassport
        ↓
Credentials
        ↓
verificación
        ↓
VerifiedCredentials
        ↓
AuthenticationEvidence
```

La finalidad principal es establecer una frontera semántica estricta entre:

```text
lo que el cliente presenta
```

y:

```text
lo que VoltStack ha verificado
```

La arquitectura deberá impedir que una simple credencial, claim o token parseado se convierta accidentalmente en prueba confiable.

---

## 2. Principio fundamental

VoltStack utilizará la siguiente regla:

> **Input presentado no es evidencia. Solo una verificación exitosa produce evidencia.**

Por tanto:

```text
Credential
    ≠
VerifiedCredential

Claim
    ≠
VerifiedIdentity

Token
    ≠
Trusted Token Claims

Passport
    ≠
Authenticated Identity

AuthenticationEvidence
    =
resultado estructurado de pruebas verificadas
```

---

## 3. Relación general

```text
Authenticator
      ↓
AuthenticationPassport
      │
      ├── IdentityClaim
      ├── Credential[]
      ├── FactorInput[]
      ├── ProtocolState
      └── Metadata
      │
      ▼
Passport Processor
      │
      ├── Identity Resolution
      ├── Credential Verification
      └── Factor Verification
      │
      ▼
AuthenticationEvidence
      │
      ├── Identity
      ├── VerifiedCredential[]
      ├── VerifiedFactor[]
      ├── Provenance
      └── Safe Metadata
      │
      ▼
Risk / Assurance / Policy
```

---

## 4. AuthenticationPassport

El `AuthenticationPassport` será la representación estructurada de un intento de autenticación.

Su responsabilidad será describir:

```text
qué mecanismo se está usando
qué identidad se afirma o intenta resolver
qué credenciales fueron presentadas
qué factores adicionales existen
qué transaction/protocol state aplica
qué metadata contextual acompaña el intento
```

---

## 5. Passport no representa confianza

Un Passport podrá contener:

```text
email inexistente
password incorrecto
token expirado
API key revocada
passkey assertion inválida
firma incorrecta
transaction expirada
```

y seguirá siendo un objeto estructuralmente válido.

Por tanto:

```text
Passport valid structurally
    ≠
authentication valid
```

---

## 6. PassportInterface

Contrato conceptual:

```php
interface AuthenticationPassportInterface
{
    public function method(): AuthenticationMethod;

    public function credentials(): CredentialSet;

    public function identityResolution(): IdentityResolutionInput;

    public function metadata(): AuthenticationPassportMetadata;
}
```

La API final podrá utilizar objetos concretos readonly en lugar de una interfaz excesivamente abstracta.

---

## 7. AuthenticationPassport recomendado

Ejemplo conceptual:

```php
final readonly class AuthenticationPassport
{
    public function __construct(
        public AuthenticationMethod $method,
        public IdentityResolutionInput $identity,
        public CredentialSet $credentials,
        public FactorInputSet $factors,
        public AuthenticationPassportMetadata $metadata,
        public ?AuthenticationTransactionReference $transaction = null,
    ) {}
}
```

---

## 8. Passport inmutable

Después de crearse, el Passport deberá ser inmutable.

No deberá evolucionar así:

```text
Passport
    identity = null
    verified = false

later:

Passport
    identity = User
    verified = true
```

Eso mezcla input no confiable y resultado confiable.

---

## 9. Transformaciones explícitas

La secuencia deberá crear objetos nuevos:

```text
AuthenticationPassport
        ↓
ResolvedAuthenticationIdentity
        ↓
CredentialVerificationSet
        ↓
AuthenticationEvidence
```

---

## 10. Passport Metadata

El Passport podrá contener metadata como:

```text
authenticator
firewall
transport
protocol
transaction id
provider hint
tenant hint validado estructuralmente
request correlation id
```

No deberá contener copias redundantes de secrets.

---

## 11. AuthenticationPassportMetadata

Podrá ser un value object:

```php
final readonly class AuthenticationPassportMetadata
{
    public function __construct(
        public string $authenticator,
        public ?string $protocol = null,
        public array $attributes = [],
    ) {}
}
```

Los atributos deberán ser namespaced o tipados cuando sea posible.

---

## 12. IdentityResolutionInput

No todos los mecanismos resuelven identidad en la misma fase.

VoltStack deberá soportar:

```text
CLAIM_FIRST
CREDENTIAL_FIRST
PROVIDER_ASSERTION
DIRECT_REFERENCE
TRANSACTION_BOUND
```

---

## 13. CLAIM_FIRST

Ejemplo password:

```text
email
    ↓
IdentityProvider
    ↓
Identity
    ↓
password verification
```

---

## 14. CREDENTIAL_FIRST

Ejemplo token:

```text
raw token
    ↓
signature verification
    ↓
trusted subject
    ↓
Identity
```

---

## 15. PROVIDER_ASSERTION

Ejemplo OIDC:

```text
verified issuer assertion
    ↓
issuer + subject
    ↓
federated identity mapping
```

---

## 16. DIRECT_REFERENCE

Ejemplo session recovery:

```text
session record
    ↓
identity reference
    ↓
identity reload
```

---

## 17. TRANSACTION_BOUND

Ejemplo Passkey:

```text
authentication transaction
    ↓
registered credential reference
    ↓
assertion verification
    ↓
identity
```

---

## 18. IdentityResolutionInput model

Podrá existir:

```text
IdentityClaimInput
CredentialDerivedIdentityInput
IdentityReferenceInput
FederatedSubjectInput
TransactionIdentityInput
```

como variantes controladas.

---

## 19. IdentityClaimInput

Ejemplo:

```php
new IdentityClaimInput(
    new IdentityClaim(
        type: 'email',
        value: $email,
    )
);
```

---

## 20. Claim normalization

La normalización deberá pertenecer a:

```text
Identity Claim Normalizer
```

o al provider.

No se modificará automáticamente toda claim con reglas globales.

---

## 21. Credential

Una `Credential` representa material presentado para demostrar algo.

Se considera:

```text
UNTRUSTED
```

hasta ser verificada.

---

## 22. CredentialInterface

Contrato mínimo:

```php
interface CredentialInterface
{
    public function type(): CredentialType;
}
```

No deberá incluir:

```php
public function verify(): bool;
```

porque la verificación pertenece al `CredentialVerifier`.

---

## 23. CredentialType

Debe ser extensible.

Ejemplos:

```text
password
bearer_token
api_key
session_reference
remember_me
passkey_assertion
totp
recovery_code
signed_assertion
client_certificate
client_secret
magic_link
service_credential
```

---

## 24. Value object vs enum

Como plugins podrán introducir nuevos tipos, se recomienda:

```php
final readonly class CredentialType
{
    public function __construct(
        public string $value
    ) {}
}
```

con constantes/factories para los tipos del Core.

---

## 25. CredentialSet

Un Passport podrá contener múltiples credentials.

Ejemplo:

```text
service token
+
client certificate reference
```

Para manejarlas se utilizará:

```text
CredentialSet
```

---

## 26. CredentialSet responsibilities

Deberá permitir:

```text
iterate
find by type
detect duplicate credential types
preserve deterministic order
enforce cardinality rules
```

---

## 27. Duplicate Credentials

Por defecto, dos Credentials del mismo tipo deberán considerarse sospechosas si el protocolo no las permite.

Ejemplo:

```text
two bearer tokens
```

no deberá producir:

```text
take the first
```

---

## 28. Credential cardinality

Cada tipo podrá declarar:

```text
SINGLE
MULTIPLE_ALLOWED
ONE_OR_MORE
PROTOCOL_DEFINED
```

---

## 29. SensitiveCredential

Podrá existir un marker interface:

```php
interface SensitiveCredentialInterface
    extends CredentialInterface
{
}
```

para activar:

```text
redaction
serialization restrictions
debug restrictions
short lifetime rules
```

---

## 30. SecretCredential

Para secretos puros:

```text
PasswordCredential
ApiKeySecretCredential
ClientSecretCredential
RecoveryCodeCredential
```

podrá existir:

```php
interface SecretCredentialInterface
    extends SensitiveCredentialInterface
{
}
```

---

## 31. Secret access

Debe minimizarse.

En lugar de exponer libremente:

```php
$credential->value();
```

podrían utilizarse APIs cuidadosamente controladas.

Sin embargo, no deberá introducirse una abstracción tan compleja que impida el verifier real.

---

## 32. PasswordCredential

Conceptualmente:

```php
final readonly class PasswordCredential
    implements SecretCredentialInterface
{
    public function __construct(
        #[\SensitiveParameter]
        private string $password,
    ) {}

    public function type(): CredentialType
    {
        return CredentialType::password();
    }
}
```

---

## 33. Password serialization

`PasswordCredential` no deberá:

```text
serialize to session
serialize to cache
serialize to JSON
appear in exception dumps
appear in event payloads
```

---

## 34. BearerTokenCredential

Representa el raw token recibido.

```php
final readonly class BearerTokenCredential
    implements SecretCredentialInterface
{
    // raw token
}
```

El token todavía no es confiable.

---

## 35. Token parsing vs verification

Podrán distinguirse:

```text
RawTokenCredential
        ↓
structural parser
        ↓
ParsedToken
        ↓
cryptographic verifier
        ↓
VerifiedTokenEvidence
```

Pero el objeto `ParsedToken` seguirá siendo no confiable.

---

## 36. ApiKeyCredential

Una API key puede modelarse como:

```text
identifier
+
secret
```

Ejemplo:

```text
ak_live_9F8D.<secret>
```

Esto permite:

```text
lookup by public identifier
verify secret hash
```

sin buscar por secreto completo.

---

## 37. ApiKeyCredential model

Conceptualmente:

```php
final readonly class ApiKeyCredential
    implements SecretCredentialInterface
{
    public function __construct(
        public string $keyId,
        #[\SensitiveParameter]
        private string $secret,
    ) {}
}
```

---

## 38. SessionReferenceCredential

No contiene necesariamente un secret de autenticación de usuario.

Representa:

```text
session reference
```

extraída de infraestructura stateful.

Aun así deberá considerarse sensible.

---

## 39. RememberMeCredential

Representará un token persistente.

Debe tratarse como credential independiente de una sesión activa.

---

## 40. PasskeyAssertionCredential

Podrá contener:

```text
credential id
authenticator data
client data
signature
user handle
transaction/challenge reference
```

La firma todavía no ha sido verificada.

---

## 41. SignedAssertionCredential

Puede representar:

```text
SAML assertion
OIDC ID token
custom signed assertion
service assertion
```

aunque protocolos complejos podrán utilizar tipos especializados.

---

## 42. ClientCertificateCredential

Debe representar información proveniente de una fuente de transporte confiable.

No se creará desde headers arbitrarios sin trusted proxy validation.

---

## 43. Credential origin

Cada Credential podrá tener metadata de origen:

```text
HEADER
COOKIE
BODY
TLS
TRANSACTION
TRUSTED_PROXY
PROTOCOL
```

Esto puede ayudar en políticas y auditoría.

---

## 44. CredentialOrigin no da confianza

Que una credencial venga de TLS no implica automáticamente que sea válida.

Solo describe provenance.

---

## 45. Credential identifiers

Credentials persistidas deberán poseer identificadores diferentes de su secreto.

Ejemplo:

```text
CredentialId
```

para:

```text
API key
passkey
refresh token family
recovery credential
```

---

## 46. CredentialId

Será útil para:

```text
revocation
rotation
audit
lookup
device management
```

sin registrar el secret.

---

## 47. CredentialVerifier

Un `CredentialVerifier` transforma:

```text
Credential
    ↓
verification
    ↓
CredentialVerificationResult
```

---

## 48. CredentialVerifierInterface

Conceptualmente:

```php
interface CredentialVerifierInterface
{
    public function supports(
        CredentialInterface $credential
    ): bool;

    public function verify(
        CredentialVerificationRequest $request
    ): CredentialVerificationResult;
}
```

---

## 49. CredentialVerificationRequest

Podrá contener:

```text
credential
resolved identity if available
authentication environment
firewall
tenant
transaction
provider context
```

---

## 50. Verifier no necesita siempre Identity

En credential-first flows:

```text
token
certificate
signed assertion
```

la identidad puede derivarse de la verificación.

Por ello:

```text
identity
```

deberá ser opcional en ciertos Verifiers.

---

## 51. Verification outputs

Los resultados deberán ser tipados:

```text
VERIFIED
INVALID
EXPIRED
REVOKED
NOT_YET_VALID
MISMATCH
UNSUPPORTED
MALFORMED
ERROR
```

---

## 52. INVALID

La prueba es incorrecta.

Ejemplo:

```text
wrong password
bad API key secret
invalid signature
```

---

## 53. EXPIRED

La prueba pudo haber sido válida, pero ya no lo es.

Ejemplo:

```text
expired token
expired magic link
```

---

## 54. REVOKED

La credencial fue desactivada explícitamente.

---

## 55. NOT_YET_VALID

Aplica a:

```text
nbf claim
future activation
credential not active yet
```

---

## 56. MISMATCH

Útil cuando:

```text
credential belongs to another identity
tenant mismatch
audience mismatch
challenge mismatch
```

---

## 57. ERROR

Representa un fallo de infraestructura o interno.

Nunca deberá convertirse en `VERIFIED`.

---

## 58. VerifiedCredential

Una verificación exitosa debe producir un nuevo objeto:

```text
VerifiedCredential
```

sin secret original.

---

## 59. VerifiedCredential model

Conceptualmente:

```php
final readonly class VerifiedCredential
{
    public function __construct(
        public CredentialType $type,
        public string $verifier,
        public \DateTimeImmutable $verifiedAt,
        public CredentialStrength $strength,
        public array $attributes = [],
    ) {}
}
```

---

## 60. VerifiedCredential does not expose secret

Nunca deberá contener:

```text
password
raw API key
raw bearer token
OTP value
private key
full magic-link token
```

---

## 61. Verification provenance

Debe poder indicar:

```text
verifier
credential type
verified time
credential id if safe
provider/issuer
algorithm family where appropriate
```

---

## 62. CredentialStrength

Podrá describir características verificadas.

Ejemplos:

```text
password
hardware_backed
phishing_resistant
signed
holder_of_key
one_time
device_bound
federated
```

No será igual a Assurance final.

---

## 63. CredentialStrength vs Assurance

```text
CredentialStrength
    describe una prueba individual

AuthenticationAssurance
    describe el conjunto completo de autenticación
```

---

## 64. Verified Token Evidence

En token-based auth podrá existir:

```text
VerifiedTokenCredential
```

con:

```text
issuer
subject
audience
issued_at
expires_at
token id
safe claims
proof properties
```

---

## 65. Claims filtering

No todos los claims de un token deberán copiarse a Evidence.

Se deberá definir una allowlist o mapping explícito.

---

## 66. Untrusted custom claims

Incluso dentro de un token firmado, un custom claim solo significa:

> el issuer afirmó este valor.

No necesariamente:

> VoltStack debe confiar en él para cualquier propósito.

Policies decidirán su uso.

---

## 67. CredentialVerifierRegistry

VoltStack tendrá:

```text
CredentialVerifierRegistry
```

para mapear:

```text
CredentialType
    ↓
Verifier
```

---

## 68. Registry contract

Conceptualmente:

```php
interface CredentialVerifierRegistryInterface
{
    public function resolve(
        CredentialType $type
    ): CredentialVerifierDescriptor;
}
```

---

## 69. VerifierDescriptor

Podrá contener:

```text
name
service id
credential type
priority
capabilities
provider compatibility
criticality
```

---

## 70. Multiple Verifiers

Podría haber más de un verifier para un tipo.

Ejemplo:

```text
JWT RS256 verifier
opaque token verifier
external introspection verifier
```

En ese caso, deberá existir resolución explícita por Passport metadata/configuración.

No se probarán todos aleatoriamente.

---

## 71. Verifier priority

Al igual que Authenticator priority:

```text
priority
```

solo determina resolución.

No expresa fuerza de seguridad.

---

## 72. Verifier selection lock

Una vez seleccionado un verifier para una Credential, una verificación fallida no deberá provocar:

```text
try next verifier
```

salvo que el formato nunca fuera soportado por el primero y la selección se mantenga estructural.

---

## 73. PasswordVerifier

Responsabilidades:

```text
load password hash/material
invoke PasswordHasher
constant-time-safe verification as appropriate
detect rehash requirement
produce VerifiedCredential
```

---

## 74. Dummy password verification

Cuando Identity no exista, el Password Authentication flow podrá ejecutar una operación dummy para reducir enumeration temporal.

Esto pertenece al Processor/Verifier orchestration, no necesariamente al Passport.

---

## 75. Password rehash signal

Resultado verificado podrá incluir:

```text
needs_rehash = true
```

sin ejecutar directamente persistencia si se desea separar responsabilidades.

---

## 76. TokenVerifier

Podrá verificar:

```text
format
signature
issuer
audience
expiration
not-before
key validity
revocation
token type
binding
```

según policy.

---

## 77. Token type confusion

El Verifier deberá comprobar:

```text
access token
ID token
refresh token
magic-link token
```

y evitar reutilizar un tipo donde corresponde otro.

---

## 78. Audience validation

No deberá omitirse cuando el protocolo la requiera.

---

## 79. Issuer validation

Debe existir allowlist/configuración confiable.

---

## 80. Algorithm validation

Nunca confiar en:

```text
alg
```

sin policy del servidor.

---

## 81. Key identification

Un `kid` solo podrá seleccionar claves dentro de un key set confiable.

---

## 82. API Key Verifier

Flujo:

```text
key id
   ↓
credential repository
   ↓
stored secret hash
   ↓
verify presented secret
   ↓
status / expiration / tenant checks
```

---

## 83. API key scopes

Si una API key contiene scopes, éstos deberán considerarse metadata para Authorization posterior.

No deberán convertir al Verifier en Authorization Engine.

---

## 84. Passkey Verifier

Deberá validar aspectos como:

```text
challenge
origin
RP ID
credential id
signature
user presence
user verification
counter/backup metadata when applicable
transaction binding
```

según el protocolo.

---

## 85. Passkey output

Podrá producir atributos seguros como:

```text
user_verified = true
phishing_resistant = true
credential_id = ...
backup_eligible = ...
```

para Assurance.

---

## 86. TOTP Verifier

Aunque TOTP se tratará más profundamente como factor, su input sigue siendo credential-like.

Deberá considerar:

```text
time window
replay policy
attempt throttling
secret reference
```

---

## 87. Recovery Code Verifier

Deberá realizar:

```text
lookup/hash verification
atomic consumption
status validation
```

---

## 88. One-time credential atomicity

La verificación exitosa y consumo deberán coordinarse para evitar:

```text
two concurrent successes
```

---

## 89. SignedAssertionVerifier

Aplicable a:

```text
SAML
OIDC
service assertions
```

deberá verificar primero firma/integridad antes de confiar en subject/claims.

---

## 90. CertificateVerifier

Deberá validar:

```text
certificate trust
chain
validity period
purpose
revocation where required
identity mapping
```

si esa responsabilidad no fue completamente establecida por la capa TLS confiable.

---

## 91. Verification ordering

No todas las Credentials se verifican en el mismo orden.

VoltStack deberá definir un:

```text
CredentialVerificationPlan
```

---

## 92. CredentialVerificationPlan

Podrá indicar:

```text
credential order
dependencies
identity resolution phase
all required vs one required
short-circuit rules
atomic requirements
```

---

## 93. Example password plan

```text
resolve identity
    ↓
verify password
```

---

## 94. Example bearer plan

```text
verify token
    ↓
derive subject
    ↓
resolve local identity
```

---

## 95. Example service dual proof

```text
verify certificate
verify service token
    ↓
ensure same identity
```

---

## 96. Plan compiler

El plan podrá derivarse de:

```text
AuthenticatorDescriptor
Credential types
Firewall policy
Authentication intent
```

durante runtime o compilación parcial.

---

## 97. Verification dependencies

Ejemplo:

```text
RecoveryCodeCredential
```

puede requerir primero una Identity.

Otro:

```text
OIDC ID Token
```

puede producir la identidad externa.

---

## 98. CredentialVerificationManager

Coordinará todos los Verifiers del Passport.

---

## 99. Manager input

```text
AuthenticationPassport
Resolved Identity optional
AuthenticationEnvironment
VerificationPlan
```

---

## 100. Manager output

```text
CredentialVerificationSet
```

---

## 101. CredentialVerificationSet

Podrá contener:

```text
VerifiedCredential[]
FailedCredential[]
DerivedIdentityClaims[]
VerificationMetadata
```

pero un failure que invalide el intento normalmente impedirá Evidence final.

---

## 102. Failure retention

Internamente podrá conservarse información segura para:

```text
diagnostics
audit
metrics
```

sin trasladarla al cliente.

---

## 103. AuthenticationEvidence

Será la representación canónica de todo lo que VoltStack ha aceptado como prueba verificada dentro del intento.

---

## 104. Evidence responsibilities

Debe responder:

```text
qué Identity fue establecida
qué Credentials fueron verificadas
qué Factors fueron verificados
qué método/protocol provenance existe
cuándo se verificaron
qué propiedades seguras se derivaron
```

---

## 105. Evidence is not Decision

Aunque toda Evidence sea válida:

```text
AuthenticationEvidence
```

todavía no significa:

```text
AUTHENTICATED
```

porque pueden faltar:

```text
MFA
minimum assurance
freshness
risk requirements
tenant policy
```

---

## 106. Evidence model

Conceptualmente:

```php
final readonly class AuthenticationEvidence
{
    public function __construct(
        public IdentityInterface $identity,
        public AuthenticationMethod $method,
        public VerifiedCredentialSet $credentials,
        public VerifiedFactorSet $factors,
        public AuthenticationProvenance $provenance,
        public \DateTimeImmutable $verifiedAt,
        public AuthenticationEvidenceAttributes $attributes,
    ) {}
}
```

---

## 107. Evidence requires Identity

La Evidence final destinada a policy/assurance deberá estar ligada a una Identity canónica.

Hasta entonces podrán existir resultados de prueba parciales.

---

## 108. Partial Evidence

Para flows multi-step podrá existir internamente:

```text
AuthenticationEvidenceFragment
```

pero no deberá confundirse con Evidence completa.

---

## 109. EvidenceFragment

Ejemplo:

```text
verified password
identity known
TOTP still pending
```

Este fragmento podrá persistirse dentro de una AuthenticationTransaction de forma segura.

---

## 110. Evidence persistence

Si se persiste Evidence parcial entre requests, deberá utilizarse una representación específica.

Nunca:

```php
serialize($passport);
```

porque podría contener secrets.

---

## 111. PersistedEvidenceSnapshot

Podrá incluir:

```text
identity reference
verified credential types
verified factor types
verification timestamps
safe provenance
security version
transaction binding
```

---

## 112. Evidence expiry

La evidencia parcial no debe ser válida indefinidamente.

Ejemplo:

```text
password verified
```

hace 30 minutos puede no ser aceptable para completar MFA ahora.

---

## 113. Evidence freshness policy

Cada tipo de prueba podrá tener ventanas distintas.

Ejemplo:

```text
password proof: 5 minutes during transaction
passkey challenge: 2 minutes
OIDC state: short-lived
```

---

## 114. VerifiedCredentialSet

Colección inmutable de VerifiedCredentials.

Deberá permitir:

```text
containsType()
all()
count()
properties()
```

---

## 115. Duplicate verified evidence

Dos verificaciones de la misma prueba no deberán inflar assurance.

Ejemplo:

```text
same password checked twice
```

no equivale a dos factores.

---

## 116. Evidence deduplication

Deberá considerar:

```text
credential id
credential type
verification event
factor category
```

según caso.

---

## 117. Factor evidence

Los factores se especificarán en profundidad posteriormente, pero la Evidence deberá reservar un espacio formal para:

```text
VerifiedFactorSet
```

---

## 118. Credential vs factor duplication

Una PasswordCredential verificada puede producir:

```text
VerifiedCredential(password)
```

y contribuir a:

```text
VerifiedFactor(knowledge)
```

Sin embargo, ambos objetos representan dimensiones diferentes.

---

## 119. Evidence provenance

Podrá incluir:

```text
authenticator
provider
issuer
realm
transaction
credential verifier
protocol
```

---

## 120. AuthenticationProvenance

Conceptualmente:

```php
final readonly class AuthenticationProvenance
{
    public function __construct(
        public string $authenticator,
        public ?string $identityProvider,
        public ?string $issuer,
        public ?string $protocol,
    ) {}
}
```

---

## 121. Provenance security

No incluir:

```text
raw URL if user-controlled
credentials
assertion contents
private metadata
```

sin sanitización.

---

## 122. Evidence attributes

Podrán contener propiedades verificadas como:

```text
user_verified
hardware_backed
device_bound
phishing_resistant
token_bound
federated
fresh
```

---

## 123. Typed Evidence Attributes

Es recomendable evitar un `array<string,mixed>` completamente libre.

Podrán existir:

```text
EvidenceAttributeBag
```

con keys namespaced y schemas.

---

## 124. Trust level of Evidence Attributes

Todo atributo dentro de Evidence deberá indicar implícita o explícitamente que ya proviene de una fuente verificada.

Claims sin verificar no podrán insertarse.

---

## 125. Unverified Metadata

Si se necesita conservar metadata no confiable para observabilidad, deberá permanecer separada:

```text
AuthenticationInputMetadata
```

No dentro de Evidence confiable.

---

## 126. EvidenceBuilder

Solo este componente, o factories equivalentes, deberá construir Evidence final.

---

## 127. EvidenceBuilder input

```text
Resolved Identity
VerifiedCredentialSet
VerifiedFactorSet
AuthenticationProvenance
Safe Verified Attributes
```

---

## 128. EvidenceBuilder invariants

Debe garantizar:

```text
identity present
no raw secrets
all credentials verified
all factors verified
no contradictory identity references
no invalid transaction binding
consistent timestamps
```

---

## 129. Identity binding

Todo VerifiedCredential que corresponda a una Identity deberá poder demostrar que fue verificado para:

```text
that identity
```

o produjo esa identity.

---

## 130. CredentialBinding

Podrá existir:

```text
CredentialBinding
```

con tipos:

```text
IDENTITY_BOUND
SUBJECT_DERIVED
TENANT_BOUND
DEVICE_BOUND
SESSION_BOUND
TRANSACTION_BOUND
```

---

## 131. Password binding

```text
password hash
    belongs to
Identity A
```

por tanto su VerifiedCredential debe quedar ligado a A.

---

## 132. Token binding

Un token verificado puede producir:

```text
issuer + subject
```

que posteriormente se mapea a Identity A.

---

## 133. Passkey binding

La credential ID registrada debe estar asociada a la identity correcta o al descubrimiento WebAuthn correspondiente.

---

## 134. Tenant binding

Una credential tenant-scoped no podrá producir Evidence en otro tenant.

---

## 135. Device binding

Credenciales como:

```text
trusted device token
device-bound token
```

deberán registrar el Device binding verificado.

---

## 136. Transaction binding

One-time credentials y protocol assertions deberán estar ligadas a la transaction correspondiente cuando aplique.

---

## 137. Evidence conflicts

El EvidenceBuilder deberá detectar:

```text
identity mismatch
tenant mismatch
transaction mismatch
credential duplication
incompatible subjects
```

---

## 138. IdentityConflict

Ejemplo:

```text
VerifiedCredential A → User 10
VerifiedCredential B → User 12
```

No podrá construirse una Evidence única.

---

## 139. EvidenceConflictResult

Podrá existir un resultado tipado:

```text
IDENTITY_CONFLICT
TENANT_CONFLICT
DEVICE_CONFLICT
TRANSACTION_CONFLICT
PROVENANCE_CONFLICT
```

---

## 140. Evidence immutability

Una vez construida:

```text
AuthenticationEvidence
```

será immutable.

---

## 141. Evidence extension

Tras un challenge exitoso:

```text
Evidence V1
    ↓
EvidenceExtender
    ↓
Evidence V2
```

No mutar V1.

---

## 142. EvidenceExtender

Podrá recibir:

```text
prior safe Evidence snapshot
new VerifiedFactor
transaction
```

y producir una nueva Evidence.

---

## 143. Step-up evidence

Un step-up sobre un AuthenticationContext existente puede producir nueva Evidence basada en:

```text
previous authentication chain
+
new proof
```

pero deberá respetar freshness.

---

## 144. AuthenticationChain interaction

Cada nuevo Context podrá conservar un:

```text
AuthenticationChain
```

derivado de Evidence sucesivas.

---

## 145. Evidence expiration

Algunas VerifiedCredentials pueden perder relevancia para assurance por edad.

El objeto puede conservar el timestamp y dejar que `AssuranceCalculator` evalúe freshness.

---

## 146. Verification Clock

Todas las verificaciones temporales deberán usar:

```text
ClockInterface
```

para consistencia y testing.

---

## 147. Clock skew

Token/TOTP/protocol verifiers podrán tener una tolerance explícita.

No deberá existir un clock skew global oculto para todo tipo de credencial.

---

## 148. Credential redaction

VoltStack deberá definir una política común:

```text
CredentialRedactor
```

---

## 149. Redacted debug representation

Ejemplos:

```text
PasswordCredential([REDACTED])

BearerTokenCredential(
    fingerprint = "sha256:ab12..."
)

ApiKeyCredential(
    keyId = "key_123",
    secret = [REDACTED]
)
```

---

## 150. Token fingerprints

Para correlación segura puede registrarse un fingerprint irreversible del token cuando sea apropiado.

Nunca utilizarlo como reemplazo ingenuo de revocation IDs.

---

## 151. Credential fingerprinting

Podrá utilizarse para:

```text
reuse detection
audit correlation
debugging
```

pero deberá evitar permitir ataques offline sobre secrets de baja entropía.

---

## 152. No password fingerprints

No deberán registrarse hashes rápidos de passwords presentados para correlación.

---

## 153. Secret lifecycle

Modelo:

```text
Transport
   ↓
Credential
   ↓
Verifier
   ↓
Verification Result
   ↓
secret reference discarded
```

---

## 154. Secret copies

La arquitectura deberá minimizar:

```text
copies in DTOs
exception context
logging context
event payloads
debug snapshots
```

---

## 155. PHP memory limitations

VoltStack no deberá afirmar que puede borrar físicamente un string secreto de todas las copias en memoria.

El objetivo real será:

```text
minimal lifetime
minimal duplication
redaction
no persistence
```

---

## 156. Serialization policy

Por defecto, SensitiveCredentials deberán rechazar serialization o no ofrecer serializers.

---

## 157. Passport serialization

Un Passport que contiene Credentials sensibles tampoco deberá serializarse.

---

## 158. Transactions must not store Passport directly

Para MFA:

Incorrecto:

```text
AuthenticationTransaction
    passport = serialized PasswordCredential
```

Correcto:

```text
AuthenticationTransaction
    identity reference
    verified evidence snapshot
    pending requirements
```

---

## 159. Event safety

Eventos como:

```text
AuthenticationPassportCreated
CredentialVerificationStarted
CredentialVerified
```

no deberán incluir secrets.

---

## 160. CredentialVerified event

Puede contener:

```text
credential type
credential id safe reference
identity id
verifier
timestamp
outcome
```

---

## 161. Credential failure event

Puede contener:

```text
type
failure class
authenticator
tenant
correlation id
```

sin revelar el valor presentado.

---

## 162. Audit safety

Audit podrá registrar:

```text
API key id
passkey credential id
token jti when safe
session id fingerprint
```

pero no secrets.

---

## 163. Error semantics

Debe distinguirse:

```text
CredentialMalformed
CredentialInvalid
CredentialExpired
CredentialRevoked
CredentialMismatch
CredentialVerificationError
```

---

## 164. External error mapping

Externamente, varios podrán mapearse a:

```text
invalid credentials
```

cuando sea necesario evitar enumeration.

---

## 165. Internal diagnostics

Internamente sí deberá preservarse la clasificación para:

```text
security telemetry
debugging
incident response
```

---

## 166. Verification short-circuit

Cuando una credential obligatoria falla:

```text
stop
```

No continuar verificando pruebas adicionales innecesarias.

---

## 167. Exception: constant-time strategy

En algunos login flows se podrá ejecutar trabajo dummy adicional para reducir side channels.

Eso será una decisión explícita de security policy, no una negación del short-circuit lógico.

---

## 168. Multiple required credentials

Si se requieren todas:

```text
credential A invalid
```

normalmente no será necesario verificar B, salvo políticas de timing/resource específicas.

---

## 169. Verification budget

El sistema podrá limitar:

```text
number of expensive verifications
remote calls
hash computations
signature operations
```

por AuthenticationOperation.

---

## 170. Hashing resource governance

Password verification puede ser costosa en CPU/memoria.

Debe integrarse con:

```text
rate limiting
concurrency control
resource budgets
```

---

## 171. Token introspection resource governance

Remote introspection deberá tener:

```text
timeout
circuit breaker
cache where safe
```

sin convertir un outage en authentication bypass.

---

## 172. Key cache

Public keys para verificar tokens/assertions podrán cachearse.

Credential secret material no.

---

## 173. Verified Evidence caching

Por regla general, Evidence será request/transaction scoped.

No deberá cachearse globalmente sin una arquitectura específica.

---

## 174. Session reconstruction

Una AuthenticationSession puede persistir un snapshot seguro de Evidence/provenance suficiente para reconstruir Context sin repetir password verification.

---

## 175. Re-verification vs reconstruction

Una request posterior no vuelve a comprobar el password.

Comprueba:

```text
session authenticity
session validity
security version
identity validity
tenant binding
```

y reconstruye Context.

---

## 176. CredentialVersion

Puede utilizarse para invalidar state cuando una credential cambia.

Ejemplo:

```text
password_version
MFA_version
```

---

## 177. SecurityVersion

Puede invalidar todas las sesiones/tokens relacionados con cambios críticos.

---

## 178. Evidence security version

Persisted Evidence/Session deberá capturar:

```text
security_version
```

si se usa esta estrategia.

---

## 179. Credential repository separation

El Auth subsystem podrá consumir:

```text
CredentialRepositoryInterface
```

para cargar material persistido.

No deberá mezclar necesariamente el Credential input con Credential record.

---

## 180. CredentialRecord

Representa credential registrada:

```text
id
identity
type
verification material
status
created_at
expires_at
metadata
```

---

## 181. Presented Credential vs CredentialRecord

```text
Presented Credential
    viene del cliente

CredentialRecord
    vive en almacenamiento confiable
```

El verifier compara ambas.

---

## 182. Password record

Contiene:

```text
password hash
algorithm metadata
updated at
version
```

---

## 183. API key record

Contiene:

```text
key id
secret hash
identity
status
expiration
metadata
```

---

## 184. Passkey record

Contiene:

```text
credential id
public key
identity reference
counter/metadata
status
```

---

## 185. Recovery code record

Deberá almacenar una representación resistente adecuada, no plaintext si no es necesario.

---

## 186. Revocation

Credential records deberán poder marcarse:

```text
REVOKED
COMPROMISED
DISABLED
EXPIRED
```

---

## 187. Credential Status

Podrá ser:

```text
PENDING
ACTIVE
EXPIRED
REVOKED
COMPROMISED
DISABLED
CONSUMED
```

---

## 188. Verification must check status

Una comparación criptográfica exitosa no es suficiente si:

```text
credential = REVOKED
```

---

## 189. Key rotation

Signed credential verifiers deberán soportar múltiples claves válidas durante ventanas de rotación.

---

## 190. Password migration

`VerifiedCredential` podrá incluir una acción recomendada:

```text
REHASH_REQUIRED
```

---

## 191. Post-verification actions

No todas las acciones deben formar parte de Evidence.

Podrá existir:

```text
CredentialVerificationSideEffectPlan
```

para:

```text
rehash password
update passkey counter
mark last used
consume recovery code
```

---

## 192. Critical side effects

Algunas son seguridad crítica:

```text
consume recovery code
update replay state
```

y deberán coordinarse con Authentication Commit.

---

## 193. Non-critical side effects

Ejemplo:

```text
update last_used_at
```

puede ser post-commit según policy.

---

## 194. SideEffect classification

```text
CRITICAL
SECURITY_RELEVANT
BEST_EFFORT
```

---

## 195. VerificationResult structure

Conceptualmente:

```php
final readonly class CredentialVerificationResult
{
    public function __construct(
        public CredentialVerificationStatus $status,
        public ?VerifiedCredential $verifiedCredential = null,
        public array $derivedIdentityClaims = [],
        public array $sideEffects = [],
        public ?CredentialFailure $failure = null,
    ) {}
}
```

---

## 196. Derived Identity Claims

Credential-first flows podrán producir claims confiables.

Ejemplo:

```text
verified token
    ↓
issuer + subject
```

---

## 197. Derived claims trust

Estas claims ya derivan de una credencial verificada.

Aun así deberán mapearse a Identity mediante un provider/mapping confiable.

---

## 198. Federated claim mapping

Ejemplo:

```text
issuer = corporate-idp
subject = 78293
```

no deberá tratarse directamente como:

```text
local user id = 78293
```

sin mapping.

---

## 199. Local identity mapping

Podrá realizarse mediante:

```text
FederatedIdentityMapper
```

---

## 200. Evidence and Authorization

Evidence podrá aportar metadata útil a Authorization posteriormente:

```text
assurance inputs
authentication method
provider
device binding
freshness
```

pero no:

```text
permissions
roles
resource decisions
```

---

## 201. API scopes from credentials

Un token puede contener scopes.

Authentication deberá:

```text
verify that issuer/token asserts them
```

y podrá exponerlos como token claims o authorization input.

La decisión de acceso pertenece a Authorization.

---

## 202. Authentication Evidence Attributes policy

Debe diferenciarse:

```text
authentication-relevant verified attributes
authorization-relevant verified attributes
protocol metadata
```

para evitar una bolsa de atributos sin control.

---

## 203. Namespacing example

```text
auth.method
auth.provider
auth.passkey.user_verified
auth.token.issuer
auth.token.scopes
auth.device.bound
```

---

## 204. Evidence schema evolution

Los atributos persistidos deberán ser versionables si forman parte de sessions/transactions.

---

## 205. EvidenceVersion

Podrá existir:

```text
AuthenticationEvidenceVersion
```

para migrar snapshots.

---

## 206. Runtime-only attributes

Algunos atributos no deberán persistirse.

Ejemplo:

```text
internal verifier object
temporary parser state
raw protocol response
```

---

## 207. Persistence whitelist

Los serializers deberán tener una allowlist explícita de campos.

---

## 208. Evidence serializer

Podrá existir:

```text
AuthenticationEvidenceSnapshotSerializer
```

solo para representaciones seguras.

---

## 209. Serialization encryption

Si ciertos snapshots contienen metadata sensible, podrá requerirse:

```text
encryption at rest
integrity protection
```

según store.

---

## 210. Transaction storage integrity

Un atacante no deberá poder modificar:

```text
verified factor list
identity reference
pending requirement
```

en una transaction client-side sin detección.

---

## 211. Client-side transaction state

Si se usa state firmado:

```text
signed
possibly encrypted
short-lived
purpose-bound
nonce-bound
```

---

## 212. Server-side transaction state

Preferible para flows muy sensibles cuando se requiera control de revocación y one-time use.

---

## 213. Passport Builder

Para extensibilidad podría existir:

```text
AuthenticationPassportBuilder
```

que facilite custom Authenticators.

---

## 214. Builder restrictions

El builder podrá validar:

```text
duplicate credentials
missing method
invalid metadata
credential cardinality
```

---

## 215. Example custom Passport

```php
$passport = AuthenticationPassport::builder()
    ->method(AuthenticationMethod::custom('signed_webhook'))
    ->credential(
        new SignedWebhookCredential(
            signature: $signature,
            timestamp: $timestamp,
        )
    )
    ->identity(
        IdentityResolutionInput::credentialDerived()
    )
    ->build();
```

---

## 216. Credential factories

Parsers podrán crear Credentials mediante factories cuando haya normalización protocol-specific.

---

## 217. No public constructor for some credentials

Para tipos complejos, puede ser preferible:

```text
PasskeyAssertionCredential::fromParsedResponse(...)
```

para garantizar invariantes estructurales.

---

## 218. Structural validity vs cryptographic validity

Una factory podrá garantizar:

```text
fields exist
format valid
length acceptable
```

pero no:

```text
signature valid
credential trusted
```

---

## 219. Verifier result explainability

Internamente podrá describirse:

```text
credential type: bearer_token
verifier: jwt_rs256
status: INVALID
reason: audience mismatch
```

---

## 220. External redaction

El cliente podría recibir simplemente:

```text
invalid authentication credentials
```

---

## 221. Timing security

Password/API key secret comparisons deberán utilizar primitivas apropiadas.

No implementar comparaciones manuales inseguras.

---

## 222. API key hash strategy

API key secrets podrán almacenarse con:

```text
fast keyed hash/HMAC strategy
or password-style hashing
```

según entropía y modelo de threat.

La elección concreta se definirá en documentación especializada.

---

## 223. Recovery codes

Por su entropía menor y uso humano podrán requerir hashing tipo password o estrategia equivalente segura.

---

## 224. Token hashes

Opaque tokens de alta entropía pueden almacenarse mediante digest seguro para lookup/comparison.

---

## 225. Secret entropy awareness

El Verifier no deberá aplicar ciegamente la misma estrategia a:

```text
human password
128-bit API key
opaque refresh token
6-digit OTP
```

porque sus modelos de amenaza son distintos.

---

## 226. Verification policies

Podrá existir:

```text
CredentialVerificationPolicy
```

por tipo.

---

## 227. Policy examples

```text
maximum attempts
clock skew
allowed algorithms
issuer allowlist
minimum key size
password hash migration
revocation requirement
```

---

## 228. Credential-specific rate limiting

Ejemplos:

```text
password → identity/IP
OTP → transaction/identity
API key → key id/IP
recovery → identity/transaction
```

---

## 229. Throttling before verification

Debe evitarse ejecutar hashing costoso antes de rate limit cuando la policy lo permita.

---

## 230. User enumeration defense

Password flow deberá tratar:

```text
identity not found
wrong password
```

de forma externamente similar cuando corresponda.

---

## 231. Credential error boundaries

Un `CredentialVerifier` no deberá generar HTTP responses.

Retorna domain results.

---

## 232. No session creation in verifier

Prohibido:

```text
password valid
    ↓
verifier creates session
```

La sesión pertenece al commit/persistence layer.

---

## 233. No context creation in verifier

También prohibido.

---

## 234. No Authorization inside verifier

Ejemplo incorrecto:

```text
token valid, but scope insufficient
    ↓
credential invalid
```

Si el scope fue autenticado correctamente, la credential puede ser válida aunque la posterior Authorization niegue acceso.

---

## 235. Exception

Si el token está presentado para un audience/purpose incorrecto, eso sí es problema de autenticidad/validez del token para ese authentication context.

---

## 236. Purpose binding

Credentials one-time deberán contener o asociarse a:

```text
purpose
```

Ejemplos:

```text
login
password_reset
email_verification
step_up
```

---

## 237. Purpose mismatch

Un token de:

```text
email_verification
```

no deberá aceptarse como:

```text
login credential
```

aunque la firma sea válida.

---

## 238. Credential audience

Concepto similar para tokens/service assertions.

---

## 239. Context binding

Una Credential puede estar vinculada a:

```text
tenant
client
device
session
transaction
route purpose
```

---

## 240. Verification must validate bindings

No basta con verificar el secret/firma.

---

## 241. Replay resistance

One-time credentials deberán mantener:

```text
nonce
jti
consumed state
counter
challenge
```

según mecanismo.

---

## 242. ReplayDetected

Debe ser un resultado de seguridad explícito.

---

## 243. CredentialVerificationStatus extension

Podría incluir:

```text
REPLAYED
```

separado de `INVALID`.

---

## 244. Replay telemetry

Debe generar señal de alta relevancia para risk/audit.

---

## 245. Evidence from replayed proof

Nunca deberá construirse.

---

## 246. Credential repository caching

Metadata no sensible puede cachearse.

Ejemplo:

```text
public passkey key
credential status
token public key
```

pero revocation freshness deberá respetarse.

---

## 247. Negative caching

Cachear `credential not found` puede ayudar performance, pero deberá evaluarse frente a:

```text
timing leakage
rapid credential creation
revocation semantics
```

---

## 248. Provider outages

Si el CredentialRepository falla:

```text
ERROR
```

No `INVALID`, porque la clasificación interna importa.

---

## 249. Fail-closed

Tanto `INVALID` como `ERROR` no producen Evidence válida.

---

## 250. Observability spans

Posibles:

```text
auth.passport.process
auth.credential.resolve_verifier
auth.credential.verify
auth.evidence.build
```

---

## 251. Metrics

```text
credential_verification_total
credential_verification_failure_total
credential_verification_duration
credential_revoked_total
credential_expired_total
credential_replay_total
evidence_build_total
evidence_conflict_total
```

---

## 252. Metric labels

Seguros:

```text
credential_type
verifier
status
firewall
```

Controlando cardinalidad.

---

## 253. No identity labels in global metrics

Evitar:

```text
email
user id
token id
```

como labels.

---

## 254. Audit records

Podrán incluir:

```text
credential type
credential reference
identity reference
outcome
verifier
authenticator
tenant
correlation id
```

---

## 255. Testing

Cada Credential y Verifier deberá tener pruebas unitarias independientes.

---

## 256. Password verifier tests

```text
correct password
incorrect password
rehash required
malformed stored hash
disabled credential
identity mismatch
```

---

## 257. Bearer verifier tests

```text
valid token
bad signature
wrong issuer
wrong audience
expired
not yet valid
revoked
wrong token type
unknown kid
algorithm mismatch
```

---

## 258. API key verifier tests

```text
valid id/secret
unknown id
wrong secret
revoked
expired
tenant mismatch
```

---

## 259. Passkey verifier tests

```text
challenge mismatch
origin mismatch
RP ID mismatch
signature invalid
credential unknown
user verification absent
replay/counter anomaly
```

---

## 260. Recovery verifier tests

```text
valid code
invalid code
already consumed
concurrent consumption
expired recovery transaction
```

---

## 261. EvidenceBuilder tests

```text
single valid credential
multiple valid credentials
identity conflict
tenant conflict
duplicate evidence
raw secret rejection
invalid provenance
```

---

## 262. Secret leakage tests

Automated tests deberán asegurar que:

```text
var_dump-like debug
JSON serialization
exception rendering
logs
events
```

no revelen secrets.

---

## 263. Fuzz testing

Especialmente para:

```text
token parser
API key parser
SAML assertion parser
WebAuthn parser
JWT parser
```

---

## 264. Property-based testing

Útil para:

```text
CredentialSet cardinality
evidence deduplication
binding invariants
serialization whitelist
```

---

## 265. FrankenPHP tests

Reutilizar el mismo Verifier service entre requests y verificar ausencia de:

```text
previous password
previous token
previous identity
previous Evidence
```

en properties compartidas.

---

## 266. Verifiers should be stateless

Preferencia:

```text
Verifier service
    immutable/stateless

VerificationRequest
    operation-scoped
```

---

## 267. Request-specific repository data

Debe permanecer en variables locales/resultados.

---

## 268. Credential verifier concurrency

Dos verificaciones concurrentes deberán ser independientes salvo uso de stores atómicos para:

```text
replay prevention
one-time consumption
counters
```

---

## 269. Atomic credential side effects

Ejemplo recovery code:

```text
verify hash
    +
mark consumed
```

debe usar operación transaccional/atómica.

---

## 270. Passkey counter update

Debe manejar concurrencia y comportamiento del authenticator según WebAuthn.

---

## 271. Session token rotation

No pertenece directamente al Passport, pero los outputs de Credential verification pueden alimentar el sistema de token/session rotation.

---

## 272. Extensibility model

Un plugin deberá poder definir:

```text
CustomCredential
CustomCredentialVerifier
CustomVerifiedCredential attributes
Passport integration
```

sin modificar Core.

---

## 273. Registration example

```php
Auth::credentialVerifier(
    CredentialType::custom('signed_webhook'),
    SignedWebhookCredentialVerifier::class,
);
```

---

## 274. Custom Credential rules

Debe respetar:

```text
redaction
no secret serialization
typed verification result
binding
Evidence creation
```

---

## 275. Core Credential types V1

Recomendados:

```text
PasswordCredential
SessionReferenceCredential
RememberMeCredential
BearerTokenCredential
ApiKeyCredential
```

---

## 276. Extended types

Posteriormente:

```text
PasskeyAssertionCredential
TotpCredential
RecoveryCodeCredential
MagicLinkCredential
SignedAssertionCredential
ClientCertificateCredential
ServiceCredential
```

---

## 277. Core interfaces sugeridas

```text
AuthenticationPassportInterface
CredentialInterface
SensitiveCredentialInterface
CredentialVerifierInterface
CredentialVerifierRegistryInterface
AuthenticationEvidenceInterface
AuthenticationEvidenceBuilderInterface
```

---

## 278. Namespace sugerido

```text
VoltStack\Quantum\Auth\Passport
VoltStack\Quantum\Auth\Credential
VoltStack\Quantum\Auth\Credential\Contracts
VoltStack\Quantum\Auth\Credential\Verification
VoltStack\Quantum\Auth\Credential\Registry
VoltStack\Quantum\Auth\Evidence
VoltStack\Quantum\Auth\Evidence\Persistence
```

---

## 279. Estructura sugerida

```text
src/Quantum/Auth/
│
├── Passport/
│   ├── AuthenticationPassport.php
│   ├── AuthenticationPassportMetadata.php
│   ├── AuthenticationPassportBuilder.php
│   └── IdentityResolutionInput.php
│
├── Credential/
│   ├── Contracts/
│   │   ├── CredentialInterface.php
│   │   ├── SensitiveCredentialInterface.php
│   │   └── CredentialVerifierInterface.php
│   │
│   ├── CredentialType.php
│   ├── CredentialSet.php
│   ├── CredentialId.php
│   │
│   ├── Builtin/
│   │   ├── PasswordCredential.php
│   │   ├── BearerTokenCredential.php
│   │   ├── ApiKeyCredential.php
│   │   ├── SessionReferenceCredential.php
│   │   └── RememberMeCredential.php
│   │
│   ├── Verification/
│   │   ├── CredentialVerificationRequest.php
│   │   ├── CredentialVerificationResult.php
│   │   ├── CredentialVerificationStatus.php
│   │   ├── CredentialVerificationManager.php
│   │   ├── VerifiedCredential.php
│   │   ├── VerifiedCredentialSet.php
│   │   └── CredentialVerificationPlan.php
│   │
│   └── Registry/
│       ├── CredentialVerifierRegistry.php
│       └── CredentialVerifierDescriptor.php
│
└── Evidence/
    ├── AuthenticationEvidence.php
    ├── AuthenticationEvidenceBuilder.php
    ├── AuthenticationEvidenceFragment.php
    ├── AuthenticationProvenance.php
    ├── EvidenceAttributeBag.php
    ├── EvidenceConflict.php
    ├── PersistedEvidenceSnapshot.php
    └── AuthenticationEvidenceSnapshotSerializer.php
```

---

## 280. Anti-pattern — Passport mutates into success

Prohibido:

```php
$passport->setUser($user);
$passport->setVerified(true);
$passport->setAssurance('AAL2');
```

---

## 281. Anti-pattern — Credential with self-validation

Evitar:

```php
$password->verify($user);
```

si acopla hasher/provider/policy.

---

## 282. Anti-pattern — raw token becomes identity

Prohibido:

```text
decode token
    ↓
sub claim
    ↓
User
```

sin verification.

---

## 283. Anti-pattern — Evidence contains secrets

Nunca:

```text
AuthenticationEvidence.password
AuthenticationEvidence.token
AuthenticationEvidence.otp
```

---

## 284. Anti-pattern — all claims trusted

Un token firmado no convierte cada claim en una regla interna de seguridad válida.

---

## 285. Anti-pattern — serializing Passport into MFA transaction

Especialmente peligroso por secrets.

---

## 286. Anti-pattern — verifier creates session

Credential verification termina en `VerifiedCredential`, no en Session.

---

## 287. Anti-pattern — verification fallback

```text
Verifier A says INVALID
    ↓
try Verifier B
```

sin resolución explícita.

---

## 288. Anti-pattern — weak binding

Una Credential válida pero asociada a:

```text
wrong tenant
wrong transaction
wrong audience
wrong purpose
```

no es Evidence válida para esa operación.

---

## 289. Security invariants

### AUTH-PCE-01

Todo Passport se considera no confiable.

#### AUTH-PCE-02

Toda Credential presentada se considera no verificada.

#### AUTH-PCE-03

Solo un Verifier exitoso produce `VerifiedCredential`.

#### AUTH-PCE-04

`VerifiedCredential` no contiene el secret original.

#### AUTH-PCE-05

Solo pruebas verificadas forman `AuthenticationEvidence`.

#### AUTH-PCE-06

El Passport es inmutable.

#### AUTH-PCE-07

AuthenticationEvidence es inmutable.

#### AUTH-PCE-08

Claims parseadas no son confiables antes de verification.

#### AUTH-PCE-09

Credenciales tenant-bound deben validar tenant binding.

#### AUTH-PCE-10

Credenciales transaction-bound deben validar transaction binding.

#### AUTH-PCE-11

One-time credentials deben prevenir replay.

#### AUTH-PCE-12

Credential failure no produce fallback implícito.

#### AUTH-PCE-13

Evidence no ejecuta Authorization.

#### AUTH-PCE-14

Raw secrets no se persisten en Session/Transaction/Event/Audit.

#### AUTH-PCE-15

Los Verifiers compartidos deben ser stateless.

#### AUTH-PCE-16

Una Verification infraestructura en ERROR nunca se trata como VERIFIED.

#### AUTH-PCE-17

Evidence conflict impide crear Evidence válida.

#### AUTH-PCE-18

AuthenticationContext solo podrá derivarse de Evidence válida y una decisión posterior `AUTHENTICATED`.

---

## 290. Flujo password

```text
PasswordAuthenticator
        ↓
AuthenticationPassport
        ├── email claim
        └── PasswordCredential
        ↓
IdentityResolver
        ↓
UserIdentity
        ↓
PasswordCredentialVerifier
        ↓
VerifiedCredential(password)
        ↓
AuthenticationEvidenceBuilder
        ↓
AuthenticationEvidence
```

---

## 291. Flujo bearer

```text
BearerTokenAuthenticator
        ↓
AuthenticationPassport
        └── BearerTokenCredential
        ↓
TokenVerifier
        ↓
Verified token evidence
        ↓
Trusted issuer + subject
        ↓
IdentityProvider/Mapper
        ↓
Identity
        ↓
AuthenticationEvidence
```

---

## 292. Flujo API key

```text
ApiKeyAuthenticator
        ↓
ApiKeyCredential
        ├── key id
        └── secret
        ↓
CredentialRepository
        ↓
ApiKeyCredentialRecord
        ↓
secret verification
status
tenant binding
        ↓
VerifiedCredential
        ↓
Identity
        ↓
AuthenticationEvidence
```

---

## 293. Flujo Passkey

```text
PasskeyAuthenticator
        ↓
PasskeyAssertionCredential
        ↓
WebAuthnVerifier
        ├── challenge
        ├── origin
        ├── RP ID
        ├── signature
        └── user verification
        ↓
VerifiedCredential(passkey)
        ↓
Identity
        ↓
AuthenticationEvidence
```

---

## 294. Flujo service dual proof

```text
Service Authenticator Plan
        │
        ├── ClientCertificateCredential
        └── ServiceTokenCredential
        │
        ▼
CertificateVerifier + TokenVerifier
        │
        ├── Service A
        └── Service A
        │
        ▼
Identity compatibility
        │
        ▼
AuthenticationEvidence
```

---

## 295. Flujo de conflicto

```text
Certificate → Service A
Token       → Service B
        ↓
EvidenceBuilder
        ↓
IDENTITY_CONFLICT
        ↓
no AuthenticationEvidence
```

---

## 296. Lifecycle resumido

```text
UNTRUSTED PASSPORT
        ↓
STRUCTURAL VALIDATION
        ↓
CREDENTIAL RESOLUTION
        ↓
VERIFIER SELECTION
        ↓
CRYPTOGRAPHIC / SECRET VERIFICATION
        ↓
BINDING VALIDATION
        ↓
REPLAY / STATUS / EXPIRATION CHECKS
        ↓
VERIFIED CREDENTIALS
        ↓
IDENTITY CONSISTENCY
        ↓
AUTHENTICATION EVIDENCE
```

---

## 297. Criterios de aceptación

El subsistema será considerado correcto cuando:

1. Passport y Evidence sean conceptos independientes;
2. Passport sea immutable;
3. Credentials sean typed;
4. secrets estén marcados y tratados como sensibles;
5. exista CredentialSet;
6. exista CredentialVerifierRegistry;
7. existan verification results explícitos;
8. `INVALID`, `EXPIRED`, `REVOKED` y `ERROR` sean diferentes;
9. soporte identity-first flows;
10. soporte credential-first flows;
11. soporte password credentials;
12. soporte bearer credentials;
13. soporte API keys;
14. soporte session references;
15. permita Passkeys;
16. permita signed assertions;
17. soporte credential binding;
18. soporte transaction binding;
19. soporte tenant binding;
20. soporte replay detection;
21. produzca VerifiedCredential sin secrets;
22. construya Evidence solo desde resultados verificados;
23. detecte identity conflicts;
24. detecte tenant conflicts;
25. soporte evidence fragments para MFA;
26. permita persistence segura de snapshots;
27. tenga serializers allowlist-based;
28. sea observable sin secret leakage;
29. soporte rate/resource governance;
30. sea seguro con FrankenPHP;
31. permita plugins;
32. mantenga Authentication separado de Authorization.

---

## 298. Regla arquitectónica final

VoltStack deberá preservar estrictamente:

```text
PASSPORT
    contains claims and presented proof
        ↓
CREDENTIAL
    represents untrusted proof material
        ↓
VERIFIER
    validates that proof
        ↓
VERIFIED CREDENTIAL
    records a successful verification without secret
        ↓
AUTHENTICATION EVIDENCE
    aggregates trusted proof bound to one identity
        ↓
POLICY / ASSURANCE / RISK
        ↓
AUTHENTICATION DECISION
```

La frontera crítica será:

> **Nada que venga directamente del cliente podrá incorporarse a `AuthenticationEvidence` sin una verificación explícita o una transformación desde una fuente previamente confiable.**

Esto convertirá `AuthenticationEvidence` en la frontera formal entre **datos de autenticación presentados** y **hechos de autenticación que VoltStack está dispuesto a considerar confiables**.

---

## 299. Próximo documento

El siguiente documento recomendado será:

```text
09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md
```

Su responsabilidad será profundizar en:

```text
IdentityInterface
IdentityIdentifier
IdentityType
IdentityReference
IdentityClaim
IdentityClaimNormalizer
IdentityProviderInterface
IdentityProviderRegistry
IdentityProviderResolver
identity-first resolution
credential-derived resolution
tenant-aware identity resolution
global vs tenant identities
identity refresh
identity security version
account/authentication status
federated subject mapping
issuer + subject mapping
multiple providers
provider precedence
provider fallback rules
LDAP/DB/remote providers
service/machine identities
identity conflicts
caching
observability
testing
persistent-runtime safety
```

Con ese documento quedará definido formalmente **quién puede ser autenticado por VoltStack y cómo se resuelve una identidad canónica a partir de claims, credenciales o proveedores externos**.
