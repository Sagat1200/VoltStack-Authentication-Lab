# VoltStack Authentication System

## 16 — Passkey, WebAuthn, FIDO2 and Phishing-Resistant Authentication System

- **Archivo:** `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del subsistema de Passkeys, WebAuthn/FIDO2 y autenticación resistente al phishing

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
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`

---

## 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack soportará autenticación moderna basada en criptografía de clave pública mediante:

```text
WebAuthn
FIDO2
Passkeys
Security Keys
Platform Authenticators
Cross-Platform Authenticators
Discoverable Credentials
Non-Discoverable Credentials
Passwordless Authentication
Phishing-Resistant Authentication
```

El objetivo es que Passkeys no sean implementadas como un plugin especial aislado, sino como un mecanismo nativo del dominio Authentication.

---

## 2. Principio fundamental

Una Passkey no es:

```text
password replacement string
API token
shared secret
session credential
```

Es una credencial criptográfica donde:

```text
Authenticator
    owns private key

VoltStack
    stores public key
```

La private key nunca deberá ser conocida por VoltStack.

---

## 3. Modelo criptográfico

Registro:

```text
Authenticator
      │
      ├── generates Private Key
      │
      └── generates Public Key
                     │
                     ▼
                  Server
                     │
                     ▼
             stores Public Key
```

Autenticación:

```text
Server Challenge
      ↓
Authenticator
      ↓
Private Key Signature
      ↓
VoltStack
      ↓
Public Key Verification
      ↓
Authentication Evidence
```

---

## 4. Propiedad principal

A diferencia de Password/TOTP:

```text
no shared authentication secret
```

entre servidor y authenticator.

Esto reduce de forma importante el impacto de una filtración de la base de datos de credentials.

---

## 5. WebAuthn

WebAuthn será el protocolo web mediante el cual:

```text
Browser
Authenticator
Relying Party
```

participan en las ceremonies de:

```text
registration
authentication
```

---

## 6. FIDO2

Conceptualmente:

```text
FIDO2
├── WebAuthn
└── CTAP
```

VoltStack implementará principalmente la perspectiva:

```text
Relying Party / Server
```

de WebAuthn.

---

## 7. CTAP

La comunicación:

```text
Browser ↔ Authenticator
```

mediante CTAP pertenece normalmente al navegador/sistema operativo.

VoltStack no deberá implementar CTAP directamente.

---

## 8. Relying Party

VoltStack actuará como backend de una:

```text
WebAuthn Relying Party
```

---

## 9. RelyingPartyConfiguration

Debe modelar:

```text
RP ID
RP Name
Allowed Origins
User Verification Policy
Resident Key Policy
Attestation Policy
Algorithms
Timeouts
```

---

## 10. RP ID

El `rpId` será uno de los boundaries de seguridad más importantes.

Ejemplo:

```text
example.com
```

---

## 11. RP ID validation

No deberá derivarse arbitrariamente de:

```text
Host header
X-Forwarded-Host
request URL
```

sin configuración y proxy trust apropiados.

---

## 12. RP ID configuration

Debe proceder de:

```text
trusted application configuration
```

o resolución multi-tenant explícitamente controlada.

---

## 13. Origin

VoltStack deberá validar el `origin` reportado por WebAuthn.

Ejemplo:

```text
https://app.example.com
```

---

## 14. Origin validation

No deberá utilizar:

```text
endsWith("example.com")
```

ni comparaciones ambiguas.

---

## 15. AllowedOriginSet

Se recomienda:

```php
final readonly class AllowedOriginSet
{
    // normalized exact trusted origins
}
```

---

## 16. Multi-origin applications

Podrán declararse explícitamente:

```text
https://app.example.com
https://admin.example.com
```

según RP policy.

---

## 17. RP ID vs Origin

No son el mismo concepto.

```text
RP ID
    credential scope

Origin
    web security origin
```

Ambos deben validarse.

---

## 18. Tenant isolation

En aplicaciones multi-tenant puede haber:

```text
shared RP
```

o:

```text
tenant-specific RP
```

---

## 19. Shared RP

Ejemplo:

```text
tenant-a.app.example.com
tenant-b.app.example.com

RP ID = app.example.com
```

Las credentials podrían técnicamente compartir RP scope, por lo que VoltStack deberá añadir binding lógico de Tenant.

---

## 20. Tenant-bound credential

Cada PasskeyCredential deberá asociarse explícitamente con:

```text
Identity
Tenant
RelyingPartyProfile
```

según configuración.

---

## 21. Tenant-specific RP

También podrá utilizar:

```text
tenant-a.example.com
```

como RP ID cuando la arquitectura de dominio lo permita.

---

## 22. Dynamic RP ID danger

Nunca aceptar:

```text
rpId = request host
```

sin resolverlo mediante una registry de tenants/domains confiables.

---

## 23. WebAuthn Ceremonies

Existen dos ceremonies fundamentales:

```text
Registration Ceremony
Authentication Ceremony
```

---

## 24. Registration Ceremony

Permite crear una nueva credential.

```text
Authenticated / Enrollment Context
        ↓
Registration Options
        ↓
Browser navigator.credentials.create()
        ↓
Authenticator
        ↓
Credential Creation Response
        ↓
VoltStack Verification
        ↓
PasskeyCredential stored
```

---

## 25. Authentication Ceremony

Permite demostrar posesión de una credential existente.

```text
Authentication Request
        ↓
Request Options
        ↓
navigator.credentials.get()
        ↓
Authenticator
        ↓
Signed Assertion
        ↓
VoltStack Verification
        ↓
AuthenticationEvidence
```

---

## 26. Challenge

Toda ceremony deberá utilizar un:

```text
cryptographically random challenge
```

---

## 27. Challenge requirements

Debe ser:

```text
CSPRNG generated
unpredictable
sufficient entropy
short-lived
single-purpose
transaction-bound
```

---

## 28. ChallengeRepository

El challenge deberá asociarse a una:

```text
WebAuthnTransaction
```

---

## 29. Challenge storage

Podrá utilizar:

```text
Redis
database
distributed KV
```

según deployment.

---

## 30. Challenge state

Deberá incluir al menos:

```text
challenge digest/value
ceremony type
RP profile
expected origins
identity reference when known
tenant
createdAt
expiresAt
attempt state
```

---

## 31. Challenge client round-trip

El challenge obviamente debe enviarse al browser como parte de las WebAuthn options.

Pero el resto del security state no deberá confiarse al cliente.

---

## 32. Challenge comparison

Debe verificarse exactamente contra el challenge esperado.

---

## 33. Challenge replay

Después de ceremony exitosa:

```text
challenge = consumed
```

---

## 34. Failed challenge

Policy podrá decidir si:

```text
one verification attempt
```

consume inmediatamente el challenge o permite un número reducido de retries.

---

## 35. Recommended behavior

Los responses criptográficos inválidos no deberían permitir intentos ilimitados sobre la misma transaction.

---

## 36. WebAuthnTransaction

Conceptualmente:

```php
final readonly class WebAuthnTransaction
{
    public function __construct(
        public WebAuthnTransactionId $id,
        public WebAuthnCeremony $ceremony,
        public WebAuthnChallenge $challenge,
        public RelyingPartyProfileId $rpProfile,
        public ?IdentityReference $identity,
        public ?TenantReference $tenant,
        public \DateTimeImmutable $expiresAt,
    ) {}
}
```

---

## 37. Ceremony enum

```php
enum WebAuthnCeremony: string
{
    case REGISTRATION = 'registration';
    case AUTHENTICATION = 'authentication';
}
```

---

## 38. PublicKeyCredentialCreationOptions

VoltStack deberá generar opciones equivalentes a WebAuthn:

```text
rp
user
challenge
pubKeyCredParams
timeout
excludeCredentials
authenticatorSelection
attestation
extensions
```

---

## 39. PublicKeyCredentialRequestOptions

Para Authentication:

```text
challenge
timeout
rpId
allowCredentials
userVerification
extensions
```

---

## 40. Options generation boundary

Nunca aceptar opciones críticas propuestas por el frontend.

El servidor decide:

```text
challenge
rpId
allowed algorithms
userVerification
credential allowlist
attestation policy
```

---

## 41. Client preferences

El frontend puede expresar preferencias de UX cuando policy lo permita, pero no debilitar security requirements.

---

## 42. User Entity

Registration requiere:

```text
id
name
displayName
```

según WebAuthn.

---

## 43. User Handle

El campo `user.id` deberá ser un identificador opaco estable.

---

## 44. User Handle must not be raw email

Preferir:

```text
random opaque stable identifier
```

o un identificador interno seguro.

---

## 45. UserHandle Value Object

```php
final readonly class UserHandle
{
    public function __construct(
        private string $binaryValue,
    ) {}
}
```

---

## 46. User Handle mapping

Debe poder resolver:

```text
UserHandle
    ↓
IdentityReference
```

---

## 47. User handle privacy

No incluir información sensible o directamente interpretable si puede evitarse.

---

## 48. Credential ID

Cada WebAuthn credential posee un:

```text
credentialId
```

---

## 49. Credential ID uniqueness

Deberá tratarse como identificador binario/opaco.

---

## 50. Credential ID is not secret

Puede almacenarse server-side.

Pero no debe utilizarse como prueba de Authentication.

---

## 51. Public Key

Se almacena:

```text
credential public key
```

asociada al credential ID.

---

## 52. Private Key

Nunca llega a VoltStack.

---

## 53. PasskeyCredential

Conceptualmente:

```php
final readonly class PasskeyCredential
{
    public function __construct(
        public PasskeyCredentialId $id,
        public WebAuthnCredentialId $credentialId,
        public IdentityReference $identity,
        public UserHandle $userHandle,
        public RelyingPartyProfileId $rpProfile,
        public PublicKeyMaterial $publicKey,
        public PasskeyCredentialStatus $status,
        public PasskeyProperties $properties,
        public \DateTimeImmutable $createdAt,
    ) {}
}
```

---

## 54. Additional metadata

Podrá contener:

```text
signCount
transports
AAGUID
backupEligible
backupState
attestation metadata reference
lastUsedAt
name
tenant
```

---

## 55. Credential status

```text
PENDING
ACTIVE
DISABLED
REVOKED
COMPROMISED
REPLACED
```

---

## 56. Registration pending state

Una credential no deberá quedar `ACTIVE` antes de verificar completamente la registration response.

---

## 57. Registration flow

```text
Enrollment Authorization
        ↓
RegistrationTransaction
        ↓
Creation Options
        ↓
Browser/Authenticator
        ↓
Attestation Response
        ↓
clientDataJSON verification
        ↓
authenticatorData verification
        ↓
challenge verification
        ↓
origin verification
        ↓
RP ID hash verification
        ↓
credential public key extraction
        ↓
algorithm validation
        ↓
user presence/verification checks
        ↓
attestation policy
        ↓
credential uniqueness
        ↓
PasskeyCredential ACTIVE
```

---

## 58. Registration authorization

Registrar una Passkey requiere contexto seguro.

---

## 59. Existing account enrollment

Normalmente:

```text
Authenticated Identity
+
Fresh Authentication
```

---

## 60. First-factor bootstrap

Para una cuenta nueva/passwordless, el onboarding system podrá proporcionar un Enrollment Context confiable.

---

## 61. Existing MFA account

Añadir una nueva Passkey puede requerir:

```text
fresh AAL2
```

o un factor ya registrado.

---

## 62. Credential injection defense

Un atacante con una sesión robada no deberá poder añadir silenciosamente su propia Passkey.

---

## 63. Freshness requirement

Enrollment deberá exigir fresh Authentication apropiada.

---

## 64. Registration challenge binding

El challenge debe quedar ligado a:

```text
Identity
Tenant
RP profile
Enrollment purpose
```

---

## 65. `clientDataJSON`

Debe validarse cuidadosamente.

Incluye información como:

```text
type
challenge
origin
crossOrigin
```

según protocolo.

---

## 66. Client data type

Para registration deberá esperarse el tipo correspondiente a:

```text
webauthn.create
```

---

## 67. Authentication type

Para assertion:

```text
webauthn.get
```

---

## 68. Type confusion

Una response de registration no deberá aceptarse en Authentication y viceversa.

---

## 69. Challenge encoding

Debe respetarse la representación Base64URL requerida.

No realizar comparaciones sobre formas ambiguas.

---

## 70. Origin

Debe coincidir con `AllowedOriginSet`.

---

## 71. Cross-origin state

Cuando WebAuthn provea señales cross-origin, deberán validarse según policy.

---

## 72. Authenticator Data

Contiene datos críticos como:

```text
rpIdHash
flags
signCount
attested credential data
extensions
```

---

## 73. RP ID hash

Debe verificarse contra:

```text
SHA-256(expected RP ID)
```

según WebAuthn.

---

## 74. Flags

VoltStack deberá interpretar al menos:

```text
UP
UV
BE
BS
AT
ED
```

cuando correspondan al nivel de WebAuthn soportado.

---

## 75. User Presence

`UP` representa:

```text
User Presence
```

---

## 76. User Verification

`UV` representa:

```text
User Verification
```

---

## 77. UP is not UV

Nunca tratarlos como equivalentes.

---

## 78. UserVerificationPolicy

Valores conceptuales:

```text
REQUIRED
PREFERRED
DISCOURAGED
```

mapeados correctamente al protocolo.

---

## 79. High-security operations

Podrán requerir:

```text
UV = true
```

---

## 80. Passkey Authentication Evidence

Debe conservar si:

```text
userPresence
userVerification
backupEligible
backupState
authenticator properties
```

fueron verificados.

---

## 81. Phishing resistance

WebAuthn obtiene resistencia al phishing principalmente por:

```text
origin binding
RP ID binding
public-key challenge-response
```

---

## 82. Critical consequence

Si VoltStack valida mal:

```text
origin
RP ID
challenge
```

destruye propiedades centrales de seguridad de WebAuthn.

---

## 83. Algorithm policy

Registration options deberán incluir una allowlist explícita de algoritmos aceptados.

---

## 84. Algorithm agility

La policy deberá ser:

```text
configurable
versioned
upgradeable
```

---

## 85. Credential algorithm

Almacenar metadata suficiente para verificar futuras assertions.

---

## 86. Unknown algorithms

Fail closed.

---

## 87. Public key parsing

Debe realizarse mediante componentes criptográficos especializados.

---

## 88. No handwritten crypto

VoltStack deberá abstraer una librería WebAuthn/COSE probada.

No implementar primitives criptográficas manualmente.

---

## 89. Crypto Adapter

Conceptualmente:

```php
interface WebAuthnCryptoAdapterInterface
{
    public function verifyAssertion(
        WebAuthnAssertionVerificationInput $input
    ): WebAuthnCryptoVerificationResult;
}
```

---

## 90. COSE keys

El public key puede representarse en formato COSE.

El dominio no deberá exponer detalles innecesarios a toda la aplicación.

---

## 91. Attestation

Registration puede incluir:

```text
attestation statement
```

que aporta información sobre el authenticator.

---

## 92. Attestation is not always required

Para aplicaciones comunes, exigir attestation fuerte puede:

```text
reduce privacy
reduce compatibility
increase complexity
```

---

## 93. AttestationPolicy

Podrá soportar:

```text
NONE
INDIRECT
DIRECT
ENTERPRISE
```

según capacidades y versión del estándar.

---

## 94. Default recommendation

Para la mayoría de aplicaciones:

```text
minimal / none-style attestation requirements
```

salvo necesidades empresariales específicas.

---

## 95. Enterprise attestation

Podrá utilizarse para:

```text
managed hardware
corporate security keys
device inventory
regulated environments
```

---

## 96. Attestation Trust

No basta con parsear attestation.

Debe validarse contra:

```text
trusted metadata
certificate chains
attestation policy
```

cuando se exija.

---

## 97. Metadata Service

VoltStack podrá integrar:

```text
FIDO Metadata Service
```

mediante adapter/provider.

---

## 98. Metadata cache

Debe ser:

```text
signed/trusted
cached
refreshable
failure-aware
```

---

## 99. Metadata outage

No deberá invalidar necesariamente credentials existentes si policy no depende de metadata online en cada Authentication.

---

## 100. AAGUID

Puede almacenarse para:

```text
metadata lookup
authenticator classification
security policy
diagnostics
```

---

## 101. AAGUID is not identity

Nunca usarlo como usuario/device identifier único.

---

## 102. Authenticator Attachment

Podrá distinguir:

```text
PLATFORM
CROSS_PLATFORM
```

---

## 103. Platform Authenticator

Ejemplos:

```text
Windows Hello
Touch ID
Android device authenticator
```

---

## 104. Cross-platform Authenticator

Ejemplos:

```text
USB/NFC security key
external authenticator
```

---

## 105. Do not over-trust attachment

`platform` no implica automáticamente hardware-backed ni non-exportable.

---

## 106. Discoverable Credentials

Una credential discoverable permite que el authenticator seleccione la cuenta sin que el servidor proporcione `allowCredentials`.

---

## 107. Passwordless / username-less flow

```text
User opens login
        ↓
Server sends challenge
        ↓
allowCredentials omitted
        ↓
Authenticator discovers credential
        ↓
returns credential + userHandle
        ↓
VoltStack resolves Identity
        ↓
verifies assertion
```

---

## 108. Identity resolution order

En discoverable flow:

```text
credentialId
+
userHandle
+
RP profile
```

deberán correlacionarse.

---

## 109. UserHandle mismatch

Fail closed.

---

## 110. Non-discoverable Credential

Puede requerir que el servidor conozca previamente la Identity y envíe:

```text
allowCredentials
```

---

## 111. Identifier-first login

```text
email/username
    ↓
Identity lookup
    ↓
registered credential IDs
    ↓
allowCredentials
```

---

## 112. Enumeration concern

No deberá revelar fácilmente si una Identity posee Passkeys antes de appropriate anti-enumeration handling.

---

## 113. Discoverable preferred

Para una experiencia Passkey moderna, VoltStack deberá soportar discoverable credentials como first-class.

---

## 114. Resident Key Policy

Conceptualmente:

```text
REQUIRED
PREFERRED
DISCOURAGED
```

---

## 115. Passkeys

Una Passkey moderna suele corresponder a una discoverable WebAuthn credential que puede estar sincronizada entre dispositivos por el ecosystem provider.

---

## 116. Synced Passkeys

No deberán asumirse como:

```text
single physical device
```

---

## 117. Backup Eligibility

WebAuthn puede exponer:

```text
BE
```

---

## 118. Backup State

Puede exponer:

```text
BS
```

---

## 119. PasskeyProperties

Podrá modelar:

```php
final readonly class PasskeyProperties
{
    public function __construct(
        public bool $discoverable,
        public bool $userVerificationCapable,
        public bool $backupEligible,
        public bool $backedUp,
        public ?AuthenticatorAttachment $attachment,
    ) {}
}
```

---

## 120. Backup eligibility and assurance

Una synced passkey sigue siendo phishing-resistant.

Pero ciertas organizaciones pueden diferenciar:

```text
device-bound credential
synced credential
```

para políticas de assurance específicas.

---

## 121. No universal downgrade

VoltStack no deberá declarar automáticamente:

```text
synced passkey = weak
```

La policy decide según threat model.

---

## 122. Device-bound requirement

Una aplicación regulada podría exigir una credential con propiedades compatibles con device-bound/hardware policy.

---

## 123. Backup state changes

El estado puede cambiar entre assertions.

VoltStack podrá actualizar metadata si la response verificada lo indica.

---

## 124. Sign Counter

WebAuthn puede proporcionar:

```text
signCount
```

---

## 125. Historical interpretation

El contador puede ayudar a detectar ciertas credential clones.

---

## 126. Modern limitation

No todos los authenticators incrementan el contador de manera útil.

Synced passkeys complican aún más su interpretación.

---

## 127. Critical rule

Nunca asumir:

```text
signCount == 0
    → invalid
```

---

## 128. Counter verification

Debe seguir semántica WebAuthn compatible y policy configurable.

---

## 129. Counter anomaly

Si:

```text
newCount <= storedCount
```

cuando ambos deberían ser monotónicos, puede producir:

```text
CLONE_SUSPECTED
```

---

## 130. Clone suspicion

No siempre deberá bloquear automáticamente.

Policy podrá:

```text
deny
step-up
mark risk
notify
audit
```

---

## 131. Counter update

Debe ser atómico cuando corresponda.

---

## 132. Concurrent assertions

Dos requests concurrentes pueden producir races.

El persistence layer deberá evitar falsos estados o lost updates.

---

## 133. Authentication flow

```text
Authentication Request
        ↓
WebAuthnAuthenticationTransaction
        ↓
Request Options
        ↓
Authenticator Assertion
        ↓
credential lookup
        ↓
credential status validation
        ↓
clientDataJSON validation
        ↓
challenge validation
        ↓
origin validation
        ↓
authenticatorData validation
        ↓
RP ID hash validation
        ↓
UP/UV validation
        ↓
signature verification
        ↓
counter/risk evaluation
        ↓
Identity resolution
        ↓
Identity Security State
        ↓
AuthenticationEvidence
```

---

## 134. Credential lookup

Por:

```text
credentialId
```

y contexto RP/Tenant.

---

## 135. Unknown credential

No autentica.

---

## 136. Revoked credential

No autentica aunque la signature sea válida.

---

## 137. Compromised credential

No autentica salvo recovery flow explícito, que normalmente tampoco debería usarla.

---

## 138. Credential Identity binding

El credential record define la Identity local asociada.

---

## 139. User Handle cross-check

Si assertion incluye userHandle:

```text
must correspond to credential Identity
```

---

## 140. Signature base

Debe construirse exactamente según WebAuthn.

No manualmente mediante concatenaciones improvisadas.

---

## 141. Signature verification

Usará el public key registrado.

---

## 142. Authentication Evidence

Podrá producir:

```text
PasskeyAuthenticationEvidence
```

---

## 143. Evidence fields

```text
credential public reference
RP profile
user presence
user verification
backup eligibility
backup state
authenticator attachment
AAGUID reference
verifiedAt
ceremony
```

---

## 144. Never include

```text
raw assertion
clientDataJSON
attestation object
```

en el Context normal salvo referencias/diagnostics controlados.

---

## 145. Factor integration

El PasskeyAuthenticator deberá producir también:

```text
VerifiedFactor
```

compatible con documento 15.

---

## 146. Typical properties

```text
category = POSSESSION
phishing_resistant = true
public_key = true
origin_bound = true
```

---

## 147. User verification property

Si `UV=true`:

```text
user_verified = true
```

---

## 148. Hardware-backed property

No deberá afirmarse sin evidencia confiable/attestation/metadata.

---

## 149. Device-bound property

Tampoco deberá inferirse únicamente de `platform`.

---

## 150. Primary Authentication

Passkey podrá autenticar sin Password.

---

## 151. Passwordless Context

```text
Passkey
    ↓
Verified Identity
    ↓
AuthenticationEvidence
    ↓
Assurance
    ↓
AuthenticationContext
```

---

## 152. Step-Up Authentication

Una sesión Password-based puede elevarse mediante Passkey.

---

## 153. Step-up flow

```text
Password Session
    ↓
Sensitive Operation
    ↓
phishing-resistant required
    ↓
WebAuthn Challenge
    ↓
Passkey verified
    ↓
Evidence extended
    ↓
Assurance recalculated
```

---

## 154. Transaction-bound Passkey

Para operaciones críticas, el WebAuthn challenge podrá quedar ligado a:

```text
operation purpose
transaction id
resource
```

mediante server-side transaction state.

---

## 155. Do not invent transaction signing semantics

Si se requiere firma explícita de detalles de transacción, deberá diseñarse un protocolo específico.

No asumir que una Authentication ceremony estándar firma automáticamente toda la operación de negocio.

---

## 156. Conditional UI

VoltStack deberá soportar flows compatibles con:

```text
WebAuthn Conditional Mediation
```

cuando browser lo permita.

---

## 157. Passkey Autofill

Esto permite mostrar Passkeys dentro de experiencias de autofill/login modernas.

---

## 158. Backend implication

El servidor debe poder generar:

```text
discoverable credential request options
```

sin identifier previo.

---

## 159. Frontend Runtime

VoltStack frontend podrá exponer un helper:

```javascript
Volt.auth.passkey.authenticate(...)
```

conceptualmente.

---

## 160. Browser API boundary

El frontend adapter será responsable de interactuar con:

```text
navigator.credentials.create()
navigator.credentials.get()
```

---

## 161. Backend remains authoritative

Frontend no decide:

```text
RP ID
challenge
UV requirement
allowed algorithms
credential policy
```

---

## 162. SPA protocol

El flow deberá funcionar sin full-page reload.

---

## 163. Example registration API

```text
POST /auth/passkeys/registration/options
POST /auth/passkeys/registration/verify
```

---

## 164. Example Authentication API

```text
POST /auth/passkeys/authentication/options
POST /auth/passkeys/authentication/verify
```

---

## 165. Public transaction identifier

Responses pueden incluir:

```text
transaction
```

opaco.

---

## 166. Server transaction state

No enviar toda la transaction serializada al browser.

---

## 167. Credential Management

Una Identity deberá poder tener:

```text
multiple Passkeys
```

---

## 168. Why multiple

Ejemplo:

```text
Phone Passkey
Laptop Passkey
Security Key
Backup Security Key
```

---

## 169. Credential naming

El usuario podrá asignar nombres:

```text
MacBook
YubiKey
Work Laptop
```

---

## 170. Name is metadata

Nunca security evidence.

---

## 171. Automatic labels

Podrán sugerirse a partir de metadata, pero no afirmar modelos exactos sin evidencia.

---

## 172. Credential listing

UI/API podrá mostrar:

```text
public credential id
name
createdAt
lastUsedAt
backup state
status
```

---

## 173. Never show public key unnecessarily

Aunque no sea secret, normalmente no aporta UX.

---

## 174. Credential removal

Debe requerir:

```text
fresh Authentication
appropriate assurance
ownership/Authorization
```

---

## 175. Last credential removal

Si Passkey es el único método disponible:

```text
recovery implications
```

deben evaluarse antes de permitirlo.

---

## 176. Passkey replacement

Normalmente:

```text
enroll new
verify
remove old
```

---

## 177. Passkey compromise

Una credential podrá marcarse:

```text
COMPROMISED
```

---

## 178. Compromise response

Podrá:

```text
revoke credential
invalidate derived sessions
revoke trusted devices
increment SecurityVersion
notify user
audit
```

---

## 179. Credential loss

Perder un device no significa necesariamente perder la Passkey si es synced.

Por ello UI no deberá asumir equivalencia:

```text
device == credential
```

---

## 180. Synced credential UX

Podrá mostrar:

```text
Synced Passkey
```

si esa propiedad se conoce de forma confiable.

---

## 181. Device management boundary

VoltStack gestiona credentials, no necesariamente la lista real de dispositivos donde una synced Passkey existe.

---

## 182. Recovery

Passkey recovery puede involucrar:

```text
another Passkey
security key
recovery code
federated recovery
account recovery
```

---

## 183. No hidden password fallback

Una cuenta Passkey-only no deberá tener un password implícito/inseguro solo para recovery.

---

## 184. Recovery system boundary

Account Recovery será un subsistema separado.

---

## 185. Enrollment notification

Nueva Passkey:

```text
security event
```

---

## 186. Removal notification

También.

---

## 187. Authentication failure

No revelar si:

```text
credential unknown
credential revoked
signature invalid
```

al cliente general.

---

## 188. Internal diagnostics

Sí deberán distinguirse.

---

## 189. Enumeration resistance

Identifier-first flows deberán evitar respuestas significativamente distintas para:

```text
account with passkeys
account without passkeys
unknown account
```

cuando sea viable.

---

## 190. Discoverable flow advantage

Reduce necesidad de exponer existencia de cuenta antes de authenticator selection.

---

## 191. Rate limiting

Aplicar a:

```text
options generation
verification attempts
registration attempts
unknown credential IDs
```

---

## 192. Challenge generation abuse

Un atacante no deberá poder llenar el challenge store ilimitadamente.

---

## 193. Transaction quotas

Podrán aplicarse por:

```text
IP
session
identity
tenant
```

---

## 194. Payload limits

Antes de parsear:

```text
clientDataJSON
authenticatorData
attestationObject
signature
```

aplicar tamaños máximos razonables.

---

## 195. CBOR parsing

Debe utilizar parser seguro con límites.

---

## 196. Parser limits

Controlar:

```text
nesting depth
item count
byte size
duplicate/invalid structures
```

---

## 197. Malformed cryptographic input

Nunca provocar excepciones no controladas que filtren internals.

---

## 198. WebAuthnVerificationResult

Estados conceptuales:

```text
VERIFIED
INVALID_CHALLENGE
INVALID_ORIGIN
INVALID_RP_ID
INVALID_SIGNATURE
INVALID_USER_HANDLE
INVALID_FLAGS
INVALID_ALGORITHM
CREDENTIAL_UNKNOWN
CREDENTIAL_REVOKED
CREDENTIAL_COMPROMISED
COUNTER_ANOMALY
EXPIRED_TRANSACTION
MALFORMED
ERROR
```

---

## 199. Counter anomaly separate from signature invalid

Porque puede ser una señal de riesgo, no necesariamente fallo criptográfico absoluto.

---

## 200. Registration result

```text
REGISTERED
DUPLICATE_CREDENTIAL
INVALID_ATTESTATION
UNTRUSTED_AUTHENTICATOR
POLICY_REJECTED
INVALID_RESPONSE
ERROR
```

---

## 201. Credential uniqueness

El mismo credential ID no deberá registrarse a múltiples Identities dentro del mismo security realm.

---

## 202. Global uniqueness scope

Podrá definirse por:

```text
RP profile
tenant realm
global Auth realm
```

según arquitectura.

---

## 203. Recommended

Evitar que una misma WebAuthn credential quede asociada ambiguamente con múltiples identities en el mismo RP.

---

## 204. Registration exclusion list

`excludeCredentials` podrá incluir credentials existentes para reducir duplicados.

---

## 205. Server-side uniqueness remains mandatory

No confiar únicamente en `excludeCredentials`.

---

## 206. Race condition

Dos registration requests concurrentes pueden intentar insertar la misma credential.

DB constraint deberá proteger.

---

## 207. User Handle uniqueness

Debe existir mapping consistente.

---

## 208. User Handle stability

No cambiar por:

```text
email change
username change
display name change
```

---

## 209. Account merge

Si identities se fusionan, las Passkeys deberán migrarse mediante proceso explícito.

---

## 210. Account split

Igualmente.

---

## 211. Identity deletion

Credentials deberán:

```text
revoke/delete according to retention policy
```

---

## 212. Credential lifecycle

```text
PENDING
   ↓
ACTIVE
   ├──→ DISABLED
   ├──→ REVOKED
   ├──→ COMPROMISED
   └──→ REPLACED
```

---

## 213. Re-enable

Si se permite:

```text
DISABLED → ACTIVE
```

deberá requerir policy explícita.

---

## 214. Revoked is terminal

Normalmente:

```text
REVOKED
```

no vuelve a `ACTIVE`.

---

## 215. Compromised is terminal

Recomendado.

---

## 216. SecurityVersion

Cambios críticos de Passkeys pueden actualizar:

```text
Identity SecurityVersion
```

---

## 217. Registration and SecurityVersion

Añadir una Passkey no necesariamente debe invalidar sesiones.

---

## 218. Removal

Eliminar una credential comprometida sí puede justificar invalidación.

---

## 219. PasskeyVersion

Podría existir si se requiere invalidación selectiva.

---

## 220. V1 recommendation

Utilizar:

```text
SecurityVersion
+
credential status
```

antes de introducir demasiadas versiones.

---

## 221. Session provenance

AuthenticationSession podrá conservar:

```text
passkey credential public reference
factor properties
verifiedAt
```

sin public key/raw assertion.

---

## 222. Session restoration

No debe volver a verificar WebAuthn signature.

Restaura Evidence snapshot sujeto a:

```text
session validity
SecurityVersion
credential policy
```

---

## 223. Credential revocation and active sessions

Policy deberá definir si revocar Passkey:

```text
invalidates sessions created by it
```

---

## 224. Strong recommendation

Para `COMPROMISED`:

```text
yes
```

---

## 225. Credential status check on every request?

No necesariamente.

Puede utilizarse SecurityVersion/session revocation para evitar DB lookup continuo.

---

## 226. Revocation architecture

Debe equilibrar:

```text
immediate invalidation
performance
distributed consistency
```

---

## 227. Passkey lastUsedAt

Podrá actualizarse con touch interval para evitar write amplification.

---

## 228. Sign counter update

No puede tratarse igual que `lastUsedAt` si se usa para seguridad.

Debe persistirse conforme a counter policy.

---

## 229. Audit events

```text
PasskeyRegistrationStarted
PasskeyRegistered
PasskeyAuthenticationSucceeded
PasskeyAuthenticationRejected
PasskeyRemoved
PasskeyRevoked
PasskeyCompromised
PasskeyCounterAnomalyDetected
PasskeyMetadataChanged
```

---

## 230. High-volume success events

Podrán ir a telemetry en vez de audit persistente.

---

## 231. Registration/removal

Sí deberán ser auditables.

---

## 232. Observability spans

```text
auth.webauthn.registration.options
auth.webauthn.registration.verify
auth.webauthn.authentication.options
auth.webauthn.authentication.verify
auth.webauthn.credential.lookup
auth.webauthn.signature.verify
auth.webauthn.attestation.verify
auth.webauthn.metadata.lookup
```

---

## 233. Metrics

```text
auth_passkey_registration_total
auth_passkey_authentication_total
auth_passkey_failure_total
auth_passkey_counter_anomaly_total
auth_passkey_verification_latency
auth_webauthn_challenge_total
auth_webauthn_challenge_expired_total
```

---

## 234. Safe metric labels

```text
result
ceremony
user_verification_policy
authenticator_attachment
backup_eligible
```

si cardinalidad permanece controlada.

---

## 235. Avoid labels

```text
credential id
user handle
identity id
origin if unbounded
AAGUID if uncontrolled
```

---

## 236. Logging

Nunca registrar indiscriminadamente:

```text
full attestation object
full clientDataJSON
raw assertion
signature
challenge
```

---

## 237. Challenge logging

Aunque no sea equivalente a password, tratarlo como security-sensitive ephemeral material.

---

## 238. Debug mode

No deberá romper redaction.

---

## 239. HTTPS

WebAuthn requiere secure contexts en navegadores, con excepciones específicas para desarrollo local.

---

## 240. Production policy

Passkeys deberán requerir:

```text
HTTPS
trusted origin
secure cookies/session transport
```

---

## 241. Reverse proxies

VoltStack deberá conocer proxies confiables para construir URLs de frontend, pero RP/Origin trust no deberá depender ciegamente de forwarded headers.

---

## 242. Load balancers

No afectan WebAuthn si challenge state es compartido.

---

## 243. Distributed challenge store

Necesario cuando requests pueden llegar a workers diferentes.

---

## 244. FrankenPHP

Todos los services de WebAuthn deberán ser:

```text
stateless
immutable where possible
request-independent
```

---

## 245. Prohibido

```php
final class WebAuthnAuthenticator
{
    private ?string $currentChallenge;
    private ?Identity $currentIdentity;
}
```

como singleton.

---

## 246. Transaction scope

Debe contener:

```text
challenge
ceremony
expected Identity
RP profile
tenant
requirements
```

---

## 247. Request cleanup

Después de request:

```text
raw client response references cleared
current transaction context cleared
current identity cleared
```

---

## 248. Fiber safety

Dos WebAuthn ceremonies simultáneas deberán permanecer aisladas.

---

## 249. Persistent workers

No cachear accidentalmente:

```text
last credential
last challenge
last RP
```

en mutable shared state.

---

## 250. RP configuration cache

Sí puede compartirse si es:

```text
immutable
compiled
tenant-safe
```

---

## 251. Metadata cache

También.

---

## 252. Public key cache

Podría utilizarse con cuidado, pero credential revocation/status debe respetarse.

---

## 253. Credential verification memoization

Solo dentro de una misma operation/request.

No reutilizar assertion verificada entre requests.

---

## 254. Replay protection

Una assertion está ligada a challenge.

Por tanto un response capturado no deberá funcionar con un nuevo challenge.

---

## 255. Authentication transaction TTL

Corto.

---

## 256. Registration transaction TTL

También corto.

---

## 257. Clock

Usar:

```text
ClockInterface
```

para transaction expiration.

---

## 258. Randomness

Usar:

```text
CryptographicallySecureRandomInterface
```

para challenges/user handles cuando se generen.

---

## 259. Serialization

WebAuthn utiliza datos binarios.

VoltStack deberá centralizar:

```text
Base64URL encoding/decoding
binary value objects
JSON adapters
```

---

## 260. No generic Base64 ambiguity

Usar explícitamente:

```text
Base64URL without padding semantics
```

según protocolo.

---

## 261. Binary Value Objects

Ejemplos:

```text
WebAuthnChallenge
WebAuthnCredentialId
UserHandle
AuthenticatorData
ClientDataHash
```

---

## 262. HTTP boundary

Convertir:

```text
JSON/Base64URL
```

a Value Objects inmediatamente.

---

## 263. Domain internals

No trabajar con arrays arbitrarios si pueden evitarse.

---

## 264. Registration Options DTO

```php
final readonly class PasskeyRegistrationOptions
{
    // safe client-facing representation
}
```

---

## 265. Authentication Options DTO

```php
final readonly class PasskeyAuthenticationOptions
{
    // safe client-facing representation
}
```

---

## 266. Response DTOs

```text
PasskeyRegistrationResponse
PasskeyAuthenticationResponse
```

deberán parsearse estrictamente.

---

## 267. Extension support

WebAuthn permite extensions.

VoltStack deberá usar:

```text
WebAuthnExtensionRegistry
```

si se soportan.

---

## 268. Unknown extensions

No confiar en resultados desconocidos.

---

## 269. Extension policy

Cada extensión deberá declarar:

```text
input generation
output parsing
verification
evidence contribution
```

---

## 270. PRF and future extensions

La arquitectura deberá permitir futuras capacidades sin modificar el core authenticator.

---

## 271. Extension results

No deben entrar directamente a AuthenticationContext sin verification/mapping.

---

## 272. Browser capability detection

Frontend podrá detectar:

```text
PublicKeyCredential availability
conditional mediation support
platform authenticator support
```

---

## 273. Capability is UX information

No security proof.

---

## 274. User agent strings

No deberán usarse para decidir seguridad de Passkeys.

---

## 275. Passkey UX

El framework deberá facilitar:

```text
Create a passkey
Sign in with a passkey
Use another passkey
Manage passkeys
Rename passkey
Remove passkey
```

---

## 276. Framework core vs UI

Core provee domain/services.

UI packages podrán construir componentes.

---

## 277. VoltStack directives future integration

Podría existir:

```text
@passkey
@webauthn
```

en futuras versiones del sistema de directivas, pero no pertenece al core protocol.

---

## 278. Authentication Manager integration

`PasskeyAuthenticator` será un Authenticator normal registrado en el sistema.

---

## 279. supports()

Podrá detectar:

```text
passkey authentication operation
WebAuthn assertion payload
```

No deberá activarse por cualquier JSON con un campo `credential`.

---

## 280. PasskeyAuthenticator responsibilities

```text
parse Authentication operation
resolve transaction
delegate WebAuthn verification
resolve Identity
produce Passport/Evidence
```

---

## 281. WebAuthnVerifier responsibilities

```text
protocol verification
challenge
origin
RP
flags
signature
counter
credential state
```

---

## 282. Separation

```text
PasskeyAuthenticator
        ≠
WebAuthn cryptographic verifier
```

---

## 283. Registration service

Separado:

```text
PasskeyEnrollmentManager
```

---

## 284. Authentication and enrollment must not share operation accidentally

Type/purpose binding obligatorio.

---

## 285. Authentication Passport

Podrá contener:

```text
IdentityBadge
VerifiedWebAuthnBadge
VerifiedFactorBadge
AssuranceEvidenceBadge
```

según el Passport model definido previamente.

---

## 286. Identity Provider

Passkey record puede resolver Identity directamente por reference.

No buscar por email.

---

## 287. Identity eligibility

Después de signature verification:

```text
IdentitySecurityState
```

debe validarse.

---

## 288. Disabled account

Una Passkey criptográficamente válida no debe autenticar una Identity deshabilitada.

---

## 289. Tenant disabled

Tampoco.

---

## 290. SecurityVersion

También deberá comprobarse donde corresponda al Context/session lifecycle.

---

## 291. Credential status before crypto

Puede verificarse temprano después del lookup para evitar trabajo innecesario.

Pero no filtrar externamente la razón.

---

## 292. Timing considerations

No prometer timing indistinguishable perfecto, pero evitar diferencias obvias que permitan enumeration.

---

## 293. Unknown credential handling

Puede usar paths de coste razonablemente comparable cuando sea necesario.

---

## 294. Authentication response

Éxito:

```text
AuthenticationContext
```

No:

```text
"signature valid": true
```

como resultado de dominio final.

---

## 295. Passkey as Factor

Una Authentication exitosa produce Evidence reutilizable por:

```text
MFA
Step-Up
Fresh Authentication
Authorization requirement gates
```

---

## 296. Authorization boundary

Authorization puede exigir:

```text
AuthenticationContext has phishing_resistant assurance
```

pero no debe verificar WebAuthn.

---

## 297. Route metadata example

```text
auth.assurance.property = phishing_resistant
```

---

## 298. Transaction-specific operation

Puede exigir:

```text
fresh passkey <= 2 minutes
```

---

## 299. Passkey vs MFA

Una Passkey no deberá describirse automáticamente como:

```text
2FA
```

en todo contexto.

---

## 300. Passkey vs password + OTP

El modelo de assurance deberá comparar propiedades, no número de pantallas.

---

## 301. Password fallback

Puede existir como alternate Authenticator si policy lo permite.

---

## 302. Invalid Passkey fallback

Si usuario seleccionó explícitamente Passkey y assertion falla:

```text
do not automatically try password
```

---

## 303. User chooses another method

Nueva Authentication attempt explícita.

---

## 304. Passkey-first account

Podrá no tener PasswordCredential.

---

## 305. Database schema conceptual

```text
passkey_credentials

id
public_id
identity_type
identity_id
tenant_id
rp_profile_id
credential_id
user_handle
public_key
algorithm
sign_count
aaguid
backup_eligible
backup_state
attachment
status
name
created_at
last_used_at
revoked_at
compromised_at
metadata
```

---

## 306. Sensitive columns

Aunque public key no sea secret:

```text
credential_id
user_handle
metadata
```

deben tratarse como authentication data y limitar exposición.

---

## 307. Indexes

Probablemente:

```text
unique(rp_profile_id, credential_id)
index(identity_reference)
index(user_handle)
index(status)
```

ajustados al DB subsystem.

---

## 308. Binary storage

Credential IDs/public keys podrán almacenarse binariamente o codificados canónicamente.

---

## 309. Canonical representation

Debe evitar que dos codificaciones representen ambiguamente el mismo credential ID.

---

## 310. Repository interface

```php
interface PasskeyCredentialRepositoryInterface
{
    public function findByCredentialId(
        RelyingPartyProfileId $rp,
        WebAuthnCredentialId $credential
    ): ?PasskeyCredential;

    public function findForIdentity(
        IdentityReference $identity
    ): PasskeyCredentialSet;

    public function save(
        PasskeyCredential $credential
    ): void;
}
```

---

## 311. Atomic counter update

Podrá requerir método especializado:

```text
compareAndUpdateSignCount()
```

---

## 312. Credential uniqueness constraint

Repository deberá traducir race a:

```text
DUPLICATE_CREDENTIAL
```

---

## 313. PasskeyManager API conceptual

```php
Auth::passkeys()->register(...);
Auth::passkeys()->all();
Auth::passkeys()->rename(...);
Auth::passkeys()->revoke(...);
```

---

## 314. Public API separation

Ceremony endpoints no son iguales que credential management APIs.

---

## 315. Passkey registration endpoint authorization

Debe validar actor/subject.

---

## 316. Admin registering for another user

Normalmente prohibido.

Si existe enterprise provisioning, será un flow explícito.

---

## 317. Passkey export

VoltStack no puede exportar private key.

---

## 318. Passkey import

No existe import genérico de private Passkey al servidor.

---

## 319. Backup responsibility

Synced Passkeys son gestionadas por authenticator ecosystem, no por VoltStack.

---

## 320. Credential migration between RP IDs

No es una simple DB update.

RP binding criptográfico impide tratarlo como rename trivial.

---

## 321. Domain migration

Cambiar:

```text
old.example.com
→
new.example.com
```

requiere estrategia WebAuthn específica.

---

## 322. RP ID is long-term architecture

Debe decidirse cuidadosamente antes de producción.

---

## 323. Multi-domain SaaS

Especial atención si tenants usan custom domains.

---

## 324. Custom domains

No asumir que una credential registrada para:

```text
tenant.voltstack.app
```

funcionará en:

```text
customer.com
```

---

## 325. RelyingPartyResolver

VoltStack necesitará:

```php
interface RelyingPartyResolverInterface
{
    public function resolve(
        AuthenticationOperationContext $context
    ): RelyingPartyProfile;
}
```

---

## 326. Resolver input

Puede considerar:

```text
trusted application host mapping
tenant
firewall
deployment profile
```

---

## 327. Resolver never trusts raw Host alone

Debe mapear contra configuración confiable.

---

## 328. RelyingPartyProfile

```php
final readonly class RelyingPartyProfile
{
    public function __construct(
        public RelyingPartyProfileId $id,
        public RelyingPartyId $rpId,
        public string $name,
        public AllowedOriginSet $origins,
        public UserVerificationPolicy $userVerification,
        public AttestationPolicy $attestation,
    ) {}
}
```

---

## 329. Profile compilation

Podrá compilarse/cacharse.

---

## 330. Dynamic tenant profiles

Deben invalidarse al cambiar custom domain/security configuration.

---

## 331. Testing — registration

Casos:

```text
valid registration
wrong challenge
expired challenge
wrong origin
wrong RP ID
wrong type
invalid CBOR
unsupported algorithm
duplicate credential
UV required but missing
invalid attestation
tenant mismatch
```

---

## 332. Testing — Authentication

```text
valid assertion
wrong challenge
replayed challenge
wrong origin
wrong RP
unknown credential
revoked credential
compromised credential
bad signature
userHandle mismatch
UV missing
UP missing
counter anomaly
expired transaction
```

---

## 333. Testing — discoverable

```text
no identifier
credential returned
userHandle resolves Identity
credential/handle mismatch
unknown handle
tenant isolation
```

---

## 334. Testing — multi-tenant

```text
Tenant A credential
cannot authenticate Tenant B

wrong RP profile
wrong origin
custom domain mapping
```

---

## 335. Testing — step-up

```text
AAL1 session
passkey challenge
successful elevation
freshness timestamp
base session revoked mid-flow
Identity disabled mid-flow
```

---

## 336. Testing — concurrent counter

```text
two valid concurrent assertions
atomic update behavior
risk classification
```

---

## 337. Testing — persistent runtime

Request A:

```text
Tenant A / credential A
```

Request B:

```text
Tenant B / credential B
```

Request C:

```text
invalid challenge
```

No state leakage.

---

## 338. Fuzz testing

Especialmente:

```text
CBOR
COSE keys
clientDataJSON
authenticatorData
Base64URL
attestation objects
extension outputs
```

---

## 339. Property-based tests

Útiles para:

```text
binary round-trips
challenge matching
credential ID canonicalization
counter transitions
origin normalization
```

---

## 340. Browser integration tests

Deberán existir con navegadores reales cuando sea posible.

---

## 341. Virtual authenticators

Browser automation podrá usar virtual WebAuthn authenticators para pruebas.

---

## 342. Unit tests are insufficient

El subsystem requiere:

```text
unit
integration
protocol vectors
browser E2E
security regression
```

---

## 343. Specification test vectors

La implementación deberá aprovechar vectors de WebAuthn/COSE libraries cuando estén disponibles.

---

## 344. Security invariant — Challenge

### AUTH-WEBAUTHN-CHALLENGE-01

Every ceremony uses a fresh cryptographically secure challenge.

#### AUTH-WEBAUTHN-CHALLENGE-02

Challenge is transaction-bound.

#### AUTH-WEBAUTHN-CHALLENGE-03

Challenge has explicit expiration.

#### AUTH-WEBAUTHN-CHALLENGE-04

Successful challenge cannot be replayed.

#### AUTH-WEBAUTHN-CHALLENGE-05

Registration and Authentication challenges are not interchangeable.

---

## 345. Security invariant — RP/Origin

### AUTH-WEBAUTHN-RP-01

RP ID comes from trusted configuration.

#### AUTH-WEBAUTHN-RP-02

Origin is explicitly validated.

#### AUTH-WEBAUTHN-RP-03

Host headers do not define trust automatically.

#### AUTH-WEBAUTHN-RP-04

RP ID hash is verified.

#### AUTH-WEBAUTHN-RP-05

Tenant/RP boundaries are explicit.

---

## 346. Security invariant — Credential

### AUTH-WEBAUTHN-CRED-01

Private keys never enter VoltStack.

#### AUTH-WEBAUTHN-CRED-02

Credential IDs are not authentication secrets.

#### AUTH-WEBAUTHN-CRED-03

Credential ID uniqueness is enforced server-side.

#### AUTH-WEBAUTHN-CRED-04

Credential status is checked before AuthenticationContext creation.

#### AUTH-WEBAUTHN-CRED-05

Revoked/compromised credentials cannot authenticate.

#### AUTH-WEBAUTHN-CRED-06

UserHandle and credential Identity mappings cannot conflict.

---

## 347. Security invariant — Verification

### AUTH-WEBAUTHN-VERIFY-01

ClientData type is verified.

#### AUTH-WEBAUTHN-VERIFY-02

Challenge is verified.

#### AUTH-WEBAUTHN-VERIFY-03

Origin is verified.

#### AUTH-WEBAUTHN-VERIFY-04

RP ID hash is verified.

#### AUTH-WEBAUTHN-VERIFY-05

UP/UV flags follow server policy.

#### AUTH-WEBAUTHN-VERIFY-06

Signature is cryptographically verified.

#### AUTH-WEBAUTHN-VERIFY-07

Algorithms follow server allowlists.

#### AUTH-WEBAUTHN-VERIFY-08

Malformed cryptographic input fails closed.

---

## 348. Security invariant — Enrollment

### AUTH-WEBAUTHN-ENROLL-01

Enrollment requires an authorized trusted context.

#### AUTH-WEBAUTHN-ENROLL-02

Pending credentials do not authenticate.

#### AUTH-WEBAUTHN-ENROLL-03

Enrollment is Identity/Tenant/RP bound.

#### AUTH-WEBAUTHN-ENROLL-04

Fresh Authentication is required according to policy.

#### AUTH-WEBAUTHN-ENROLL-05

Credential insertion races cannot create duplicate ownership.

#### AUTH-WEBAUTHN-ENROLL-06

New credential enrollment is auditable.

---

## 349. Security invariant — Assurance

### AUTH-WEBAUTHN-ASSURANCE-01

Phishing resistance derives from verified protocol properties.

#### AUTH-WEBAUTHN-ASSURANCE-02

User presence does not imply user verification.

#### AUTH-WEBAUTHN-ASSURANCE-03

Hardware backing is not inferred without evidence.

#### AUTH-WEBAUTHN-ASSURANCE-04

Platform attachment does not automatically imply device binding.

#### AUTH-WEBAUTHN-ASSURANCE-05

Synced Passkeys are not automatically downgraded.

#### AUTH-WEBAUTHN-ASSURANCE-06

Passkey evidence integrates with the common Assurance system.

---

## 350. Security invariant — Runtime

### AUTH-WEBAUTHN-RT-01

Current challenge state is never process-global.

#### AUTH-WEBAUTHN-RT-02

Shared WebAuthn services remain stateless.

#### AUTH-WEBAUTHN-RT-03

Concurrent ceremonies are isolated.

#### AUTH-WEBAUTHN-RT-04

Distributed workers share authoritative transaction state.

#### AUTH-WEBAUTHN-RT-05

No challenge/Identity state survives FrankenPHP request boundaries.

---

## 351. Anti-pattern — WebAuthn as “encrypted password”

Incorrecto.

---

## 352. Anti-pattern — RP ID from Host

Nunca sin trusted resolution.

---

## 353. Anti-pattern — loose origin comparison

No usar substring/suffix matching improvisado.

---

## 354. Anti-pattern — trust frontend options

El backend define security-critical options.

---

## 355. Anti-pattern — private key storage

VoltStack jamás debe solicitarla.

---

## 356. Anti-pattern — custom crypto

No implementar ECDSA/COSE/attestation manualmente.

---

## 357. Anti-pattern — signCount mandatory increment

No todos los authenticators funcionan así.

---

## 358. Anti-pattern — platform means hardware

No.

---

## 359. Anti-pattern — synced means insecure

No.

---

## 360. Anti-pattern — Passkey means 2FA

No necesariamente.

---

## 361. Anti-pattern — Passkey means single device

Especialmente incorrecto para synced Passkeys.

---

## 362. Anti-pattern — credential ID as secret

No.

---

## 363. Anti-pattern — user email as UserHandle

Evitar.

---

## 364. Anti-pattern — challenge stored only in singleton memory

Incompatible con distributed/persistent runtime.

---

## 365. Anti-pattern — registration from stale session

Riesgo de credential injection.

---

## 366. Anti-pattern — credential migration by changing RP ID column

No funciona criptográficamente.

---

## 367. Anti-pattern — raw protocol objects in Session

Persistir Evidence normalizada, no responses WebAuthn completas.

---

## 368. Componentes principales

```text
PasskeyAuthenticator
WebAuthnVerifier
WebAuthnRegistrationVerifier
WebAuthnAuthenticationVerifier

RelyingPartyProfile
RelyingPartyResolver
AllowedOriginSet

WebAuthnTransaction
WebAuthnChallenge
WebAuthnTransactionRepository

PasskeyCredential
PasskeyCredentialRepository
PasskeyCredentialStatus
PasskeyProperties

PasskeyEnrollmentManager
PasskeyCredentialManager
```

---

## 369. Componentes criptográficos

```text
WebAuthnCryptoAdapter
CoseKeyParser
SignatureVerifier
AttestationVerifier
AttestationTrustResolver
AuthenticatorMetadataProvider
```

Estos serán adapters sobre implementaciones criptográficas maduras.

---

## 370. Componentes frontend

```text
PasskeyRegistrationOptions
PasskeyAuthenticationOptions
PasskeyRegistrationResponse
PasskeyAuthenticationResponse
WebAuthnJsonCodec
```

---

## 371. Namespace sugerido

```text
VoltStack\Quantum\Auth\Passkey
VoltStack\Quantum\Auth\Passkey\Contracts
VoltStack\Quantum\Auth\Passkey\Credential
VoltStack\Quantum\Auth\Passkey\Ceremony
VoltStack\Quantum\Auth\Passkey\Challenge
VoltStack\Quantum\Auth\Passkey\Verification
VoltStack\Quantum\Auth\Passkey\Enrollment
VoltStack\Quantum\Auth\Passkey\RelyingParty
VoltStack\Quantum\Auth\Passkey\Attestation
VoltStack\Quantum\Auth\Passkey\Metadata
VoltStack\Quantum\Auth\Passkey\Transport
```

---

## 372. Estructura sugerida

```text
src/Quantum/Auth/Passkey/
├── Contracts/
│   ├── PasskeyCredentialRepositoryInterface.php
│   ├── WebAuthnTransactionRepositoryInterface.php
│   ├── RelyingPartyResolverInterface.php
│   ├── WebAuthnCryptoAdapterInterface.php
│   └── AuthenticatorMetadataProviderInterface.php
│
├── Credential/
│   ├── PasskeyCredential.php
│   ├── PasskeyCredentialId.php
│   ├── WebAuthnCredentialId.php
│   ├── UserHandle.php
│   ├── PasskeyCredentialStatus.php
│   └── PasskeyProperties.php
│
├── Ceremony/
│   ├── WebAuthnCeremony.php
│   ├── WebAuthnTransaction.php
│   ├── WebAuthnTransactionId.php
│   ├── WebAuthnRegistrationService.php
│   └── WebAuthnAuthenticationService.php
│
├── Challenge/
│   ├── WebAuthnChallenge.php
│   ├── WebAuthnChallengeGenerator.php
│   └── WebAuthnTransactionRepository.php
│
├── Verification/
│   ├── WebAuthnVerifier.php
│   ├── RegistrationVerifier.php
│   ├── AssertionVerifier.php
│   ├── WebAuthnVerificationResult.php
│   └── SignCounterEvaluator.php
│
├── Enrollment/
│   ├── PasskeyEnrollmentManager.php
│   ├── PasskeyRegistrationOptions.php
│   └── PasskeyRegistrationResult.php
│
├── RelyingParty/
│   ├── RelyingPartyProfile.php
│   ├── RelyingPartyProfileId.php
│   ├── RelyingPartyId.php
│   ├── AllowedOriginSet.php
│   └── RelyingPartyResolver.php
│
├── Attestation/
│   ├── AttestationPolicy.php
│   ├── AttestationVerifier.php
│   └── AttestationTrustResolver.php
│
├── Metadata/
│   ├── AuthenticatorMetadata.php
│   ├── AuthenticatorMetadataProvider.php
│   └── MetadataCache.php
│
└── Transport/
    ├── WebAuthnJsonCodec.php
    ├── Base64UrlCodec.php
    ├── PasskeyRegistrationResponse.php
    └── PasskeyAuthenticationResponse.php
```

---

## 373. Configuración conceptual

```php
return [

    'passkeys' => [

        'default_rp' => 'application',

        'relying_parties' => [

            'application' => [
                'id' => 'example.com',
                'name' => 'VoltStack Application',

                'origins' => [
                    'https://app.example.com',
                ],

                'user_verification' => 'preferred',
                'resident_key' => 'preferred',
                'attestation' => 'none',
            ],

        ],

    ],

];
```

---

## 374. Configuración high-security

```php
'admin' => [

    'id' => 'admin.example.com',

    'origins' => [
        'https://admin.example.com',
    ],

    'user_verification' => 'required',
    'resident_key' => 'preferred',

    'assurance' => [
        'phishing_resistant' => true,
    ],

];
```

---

## 375. Flujo completo de Registration

```text
User
  ↓
Security Settings
  ↓
Fresh Authentication Requirement
  ↓
PasskeyEnrollmentManager
  ↓
RelyingPartyResolver
  ↓
WebAuthnChallengeGenerator
  ↓
RegistrationTransaction
  ↓
PublicKeyCredentialCreationOptions
  ↓
Browser
  ↓
Authenticator
  ↓
Credential Creation Response
  ↓
RegistrationVerifier
  ├─ challenge
  ├─ type
  ├─ origin
  ├─ RP ID hash
  ├─ flags
  ├─ public key
  ├─ algorithm
  └─ attestation policy
  ↓
Credential uniqueness
  ↓
PasskeyCredential
  ↓
Repository
  ↓
Audit
```

---

## 376. Flujo completo de Passwordless Authentication

```text
Login Page
   ↓
Passkey Request
   ↓
RelyingPartyResolver
   ↓
Challenge
   ↓
Discoverable Request Options
   ↓
Browser / Conditional UI
   ↓
Authenticator selects Passkey
   ↓
Signed Assertion
   ↓
AssertionVerifier
   ├─ transaction
   ├─ challenge
   ├─ origin
   ├─ RP ID hash
   ├─ credential
   ├─ userHandle
   ├─ UP/UV
   ├─ signature
   └─ counter
   ↓
Identity Resolution
   ↓
Identity Security State
   ↓
PasskeyAuthenticationEvidence
   ↓
VerifiedFactor
   ↓
Assurance Calculator
   ↓
AuthenticationContext
   ↓
AuthenticationSession
```

---

## 377. Flujo completo de Step-Up

```text
Existing Session
   ↓
Sensitive Route
   ↓
Requirement:
   phishing-resistant + fresh
   ↓
StepUpPlanner
   ↓
PasskeyFactor
   ↓
WebAuthn Authentication Transaction
   ↓
Challenge
   ↓
Assertion
   ↓
Verification
   ↓
Passkey Evidence
   ↓
Authentication Evidence extended
   ↓
Assurance recalculated
   ↓
New Context Snapshot
   ↓
Session rotation
   ↓
Operation resumes
```

---

## 378. Arquitectura global

```text
                      PASSKEY / WEBAUTHN
                              │
                    RelyingPartyResolver
                              │
                              ▼
                      RP SECURITY PROFILE
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
          REGISTRATION                  AUTHENTICATION
               │                             │
               ▼                             ▼
       Registration Options          Request Options
               │                             │
               ▼                             ▼
             Browser                       Browser
               │                             │
               ▼                             ▼
          Authenticator                 Authenticator
               │                             │
        Public Key Created              Assertion Signed
               │                             │
               ▼                             ▼
        Registration Response         Assertion Response
               │                             │
               └──────────────┬──────────────┘
                              ▼
                      WEBAUTHN VERIFIER
                              │
               ┌──────────────┼───────────────┐
               ▼              ▼               ▼
           Challenge        Origin          RP ID
               │              │               │
               ├──────────────┼───────────────┤
               ▼              ▼               ▼
             Flags        Signature       Credential
               │              │               │
               └──────────────┼───────────────┘
                              ▼
                     VERIFIED CEREMONY
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
       Store Credential                Resolve Identity
                                             │
                                             ▼
                                   Authentication Evidence
                                             │
                                             ▼
                                       Factor Evidence
                                             │
                                             ▼
                                         Assurance
                                             │
                                             ▼
                                  AuthenticationContext
```

---

## 379. Decisiones arquitectónicas principales

VoltStack adoptará:

```text
1. WebAuthn is a first-class Authentication mechanism.
2. Passkeys are public-key credentials, never shared secrets.
3. Private keys never enter the framework.
4. RP ID and Origin are explicit trust boundaries.
5. Challenges are short-lived, transaction-bound and replay-resistant.
6. Registration and Authentication are distinct ceremonies.
7. Passkeys may be primary credentials or step-up factors.
8. Discoverable credentials are first-class.
9. UserHandle is stable and opaque.
10. Synced Passkeys are modeled explicitly.
11. Sign counters are risk signals, not universal truth.
12. Attestation is policy-driven.
13. Cryptographic primitives are delegated to mature adapters/libraries.
14. Passkey Evidence integrates with the common Factor/Assurance model.
15. All runtime state is safe for FrankenPHP and concurrent workers.
```

---

## 380. Comparación conceptual con Laravel y Symfony

VoltStack no deberá limitar Passkeys a un paquete que simplemente añada endpoints alrededor de WebAuthn.

La integración será más profunda:

```text
Passkey
   ↓
Authenticator
   ↓
Passport / Evidence
   ↓
VerifiedFactor
   ↓
AuthenticationAssurance
   ↓
Session / Stateless Context
   ↓
Step-Up
```

De Laravel se conservará la meta de una API de desarrollador sencilla.

De Symfony se tomará la separación rigurosa entre:

```text
Authenticator
Credential
Identity
Authentication result
Security context
```

VoltStack añadirá de forma nativa:

```text
RelyingParty profiles
tenant isolation
challenge transactions
Passkey lifecycle
Assurance properties
phishing-resistant requirements
discoverable credentials
synced Passkey metadata
step-up integration
persistent-runtime isolation
```

---

## 381. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte WebAuthn Registration;
2. soporte WebAuthn Authentication;
3. soporte Passkeys;
4. soporte Security Keys;
5. soporte platform authenticators;
6. soporte cross-platform authenticators;
7. soporte discoverable credentials;
8. soporte non-discoverable credentials;
9. soporte passwordless Authentication;
10. soporte Conditional UI;
11. soporte RP profiles;
12. valide RP ID;
13. valide Origin;
14. valide challenge;
15. prevenga challenge replay;
16. valide ceremony type;
17. valide authenticator data;
18. valide UP;
19. valide UV;
20. valide signatures;
21. soporte algorithm allowlists;
22. soporte credential IDs;
23. soporte UserHandle;
24. soporte credential public keys;
25. soporte credential status;
26. soporte multiple Passkeys;
27. soporte credential naming;
28. soporte credential revocation;
29. soporte credential compromise;
30. soporte sign counters correctamente;
31. soporte backup eligibility/state;
32. modele synced Passkeys;
33. soporte attestation policies;
34. soporte authenticator metadata extensible;
35. soporte tenant isolation;
36. soporte multi-RP deployments;
37. soporte custom domain resolution seguro;
38. soporte Factor integration;
39. soporte Assurance integration;
40. soporte Step-Up;
41. soporte Fresh Authentication;
42. soporte audit;
43. soporte observability;
44. soporte distributed challenge storage;
45. sea seguro con FrankenPHP;
46. sea fiber-safe;
47. use adapters criptográficos maduros;
48. mantenga Authentication separada de Authorization.

---

## 382. Regla arquitectónica final

VoltStack deberá preservar:

```text
SERVER SECURITY POLICY
        ↓
RP ID + ORIGIN + CHALLENGE
        ↓
WEBAUTHN CEREMONY
        ↓
AUTHENTICATOR PRIVATE KEY
        ↓
SIGNED ASSERTION
        ↓
SERVER PUBLIC KEY VERIFICATION
        ↓
PROTOCOL VALIDATION
        ↓
CREDENTIAL STATE VALIDATION
        ↓
IDENTITY RESOLUTION
        ↓
IDENTITY SECURITY STATE
        ↓
PASSKEY AUTHENTICATION EVIDENCE
        ↓
VERIFIED FACTOR
        ↓
AUTHENTICATION ASSURANCE
        ↓
AUTHENTICATION CONTEXT
```

La primera regla central será:

> **VoltStack jamás tratará una Passkey como una contraseña sofisticada; la tratará como una credencial criptográfica de clave pública ligada al Relying Party y al Origin.**

La segunda:

> **La resistencia al phishing de WebAuthn depende de verificar correctamente challenge, RP ID, Origin, flags y firma; omitir cualquiera de estos controles puede destruir las propiedades de seguridad que justifican utilizar Passkeys.**

La tercera:

> **Passkeys deberán poder funcionar como autenticación primaria, passwordless, MFA, fresh authentication o step-up sin crear implementaciones separadas del dominio Authentication.**

---

## 383. Próximo documento recomendado

El siguiente documento será:

```text
17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md
```

Este documento deberá definir la federación completa de Identity y Authentication:

```text
OAuth 2.x integration
OpenID Connect
Authorization Code Flow
PKCE
state
nonce
OIDC ID Tokens
Access Tokens
Refresh Tokens
UserInfo
Issuer Discovery
JWKS
Provider Registry
Google/Microsoft/GitHub-style login
enterprise Identity Providers
Identity federation
external subject mapping
account linking
account unlinking
email trust
verified email mapping
provider claims
federated assurance
acr
amr
prompt
max_age
login_hint
tenant federation
organization IdPs
SSO
Single Logout boundaries
token lifecycle
provider outages
account takeover prevention
confused deputy prevention
mix-up attack prevention
redirect URI security
audit
observability
testing
FrankenPHP safety
```

Con este documento, VoltStack pasará de manejar credenciales locales —Password, Token, MFA y Passkeys— a poder **delegar Authentication a proveedores externos y sistemas empresariales de identidad sin perder el control sobre su Identity Model, Security State, Assurance y Authorization internos**.
