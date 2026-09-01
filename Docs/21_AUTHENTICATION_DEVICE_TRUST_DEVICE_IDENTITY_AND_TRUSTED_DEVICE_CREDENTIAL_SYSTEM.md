# VoltStack Authentication System

## 21 — Authentication Device Trust, Device Identity and Trusted Device Credential System

- **Archivo:** `21_AUTHENTICATION_DEVICE_TRUST_DEVICE_IDENTITY_AND_TRUSTED_DEVICE_CREDENTIAL_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de identidad de dispositivo, reconocimiento, confianza, credenciales de dispositivo y su integración con Authentication, MFA, Risk y Session.

**Depende especialmente de:**

- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`
- `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md`
- `19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`
- `20_AUTHENTICATION_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL_SYSTEM.md`

---

## 1. Propósito

Este documento define cómo VoltStack representará, reconocerá, evaluará y administrará dispositivos relacionados con procesos de Authentication.
El sistema deberá cubrir:

- Device Identity
- Device Reference
- Device Recognition
- Known Device
- Trusted Device
- Managed Device
- Device Credential
- Trusted Device Credential
- Device Enrollment
- Device Trust
- Device Binding
- Device Attestation
- Device Risk Integration
- Device Lifecycle
- Device Revocation
- Lost Device
- Compromised Device
- Device Metadata
- Trusted Device Cookies
- Cryptographic Device Credentials
- Session Binding
- MFA Integration
- Remember-Me Integration
- Passkey Integration
- Tenant Device Policies
- Device Management
- Privacy
- Audit
- Observability

El objetivo será impedir que conceptos como:

- known device
- trusted device
- remember me
- browser fingerprint
- passkey
- session
- MFA bypass
- se mezclen incorrectamente.

## 2. Principio fundamental

VoltStack distinguirá estrictamente:

```text
DEVICE RECOGNITION
    "¿parece ser un dispositivo observado anteriormente?"

DEVICE IDENTITY
    "¿qué entidad lógica representa este dispositivo?"

DEVICE TRUST
    "¿qué evidencia tenemos para confiar en él?"

DEVICE CREDENTIAL
    "¿qué credential demuestra control del dispositivo?"

DEVICE RISK
    "¿qué riesgo presenta en esta Authentication?"

DEVICE FACTOR
    "¿qué evidencia aporta a Authentication Assurance?"
```

## 3. Known Device no equivale a Trusted Device

Un dispositivo puede ser:

- KNOWN
- porque fue observado anteriormente.

Eso no implica:
TRUSTED

## 4. Ejemplo

Browser visto 40 veces
+
sin credential criptográfica
+
sesión robada potencialmente
puede ser:

- Known Device = true
- Trusted Device = false

## 5. Trusted Device

Un dispositivo deberá ser considerado confiable únicamente cuando exista evidencia suficiente definida por una:
DeviceTrustPolicy

## 6. Arquitectura general

Authentication Request
│
▼
Device Context Extraction
│
▼
Device Recognition
│
├── Device Credential
├── Cookie Credential
├── Cryptographic Proof
├── Device Metadata
├── Passkey Association
└── Managed Device Evidence
│
▼
DeviceIdentityResolver
│
▼
DeviceReference
│
▼
DeviceTrustEvaluator
│
▼
DeviceTrustAssessment
│
├── UNKNOWN
├── KNOWN
├── TRUSTED
├── HIGH_TRUST
└── COMPROMISED
│
▼
Risk / MFA / Session Integration

## 7. Device

Device será una entidad lógica del dominio Authentication.
No deberá equivaler necesariamente a:
one physical computer
porque browsers y ecosystems modernos no permiten identificar hardware con certeza universal.

## 8. DeviceReference

Conceptualmente:

```php
final readonly class DeviceReference
{
    public function __construct(
        public DeviceId $id,
        public DeviceKind $kind,
    ) {}
}
```

## 9. DeviceKind

Ejemplos:

- BROWSER
- NATIVE_APPLICATION
- MOBILE_DEVICE
- MANAGED_DEVICE
- SERVICE_DEVICE
- HARDWARE_AUTHENTICATOR
- UNKNOWN

## 10. Browser Device

Un navegador normalmente no puede demostrar una identidad física fuerte por defecto.
Por ello su Device Identity podrá apoyarse en:

- opaque persistent credential
- session history
- browser metadata
- optional heuristics

## 11. Device fingerprint no es Identity fuerte

VoltStack no deberá convertir automáticamente un fingerprint en:
trusted device identity

## 12. Device fingerprint

Puede aportar una señal heurística para:

- recognition
- risk
- anomaly detection

pero no deberá ser Authentication Credential.

## 13. DeviceIdentity

Conceptualmente:

```php
final readonly class DeviceIdentity
{
    public function__construct(
        public DeviceId $id,
        public DeviceKind $kind,
        public DeviceStatus $status,
        public DeviceTrustState $trust,
        public \DateTimeImmutable $createdAt,
    ) {}
}
```

## 14. DeviceId

Debe ser:

- opaque
- stable within local system
- non-semantic

## 15. DeviceStatus

Estados:

- ACTIVE
- LOST
- REVOKED
- COMPROMISED
- RETIRED

## 16. DeviceTrustState

Separado del status.

- Ejemplo:
- UNKNOWN
- KNOWN
- TRUSTED
- HIGH_TRUST
- RESTRICTED

## 17. Status vs Trust

Ejemplo:

```php
status = ACTIVE
trust = KNOWN
```

o:

```php
status = COMPROMISED
trust = RESTRICTED
```

## 18. DeviceCredential

Representa una credential que demuestra control de un dispositivo o instalación.

## 19. Tipos

Trusted Device Cookie
Cryptographic Device Key
Client Certificate
Managed Device Certificate
Platform-bound Key
Application Installation Credential

## 20. Trusted Device Credential

Para browser applications, una implementación común será una credential persistente de alta entropía.

## 21. Diseño recomendado

selector.secret
igual que otros persistent credentials.

## 22. Server-side storage

Guardar:

- selector
- secret digest
- DeviceReference
- IdentityReference
- TenantReference
- status
- issuedAt
- expiresAt
- lastUsedAt

## 23. Raw secret

Nunca persistido.

## 24. TrustedDeviceCredential model

final readonly class TrustedDeviceCredential
{
public function __construct(
public TrustedDeviceCredentialId $id,
public DeviceReference $device,
public IdentityReference $identity,
public CredentialStatus $status,
public \DateTimeImmutable $issuedAt,
public \DateTimeImmutable $expiresAt,
) {}
}

## 25. CredentialStatus

ACTIVE
EXPIRED
REVOKED
COMPROMISED
ROTATED
SUPERSEDED

## 26. Device Credential != Session

Nunca reutilizar:

- session ID
- como trusted-device credential.

## 27. Device Credential != Remember-Me

Aunque puedan coexistir como cookies persistentes.

## 28. Remember-Me

Responde:
Can a prior authentication help restore Identity?

## 29. Trusted Device

Responde:
Can this device contribute trust toward reducing/requiring secondary verification?

## 30. Ejemplo

Remember-Me Credential
↓
Identity restored

Trusted Device Credential
↓
Device trust established

Risk Engine
↓
known network + trusted device

MFA policy
↓
may not require routine secondary prompt

## 31. Still no universal MFA bypass

Trusted device no significa:
skip MFA always

## 32. Sensitive operation

Puede requerir:

- fresh Passkey
- aunque Device sea trusted.

## 33. DeviceTrustAssessment

Conceptualmente:

```php
final readonly class DeviceTrustAssessment
{
    public function __construct(
        public DeviceTrustLevel $level,
        public DeviceTrustPropertySet $properties,
        public DeviceTrustReasonSet $reasons,
        public \DateTimeImmutable $assessedAt,
    ) {}
}
```

## 34. DeviceTrustLevel

Ejemplo:

- UNRECOGNIZED
- KNOWN
- TRUSTED
- STRONG
- MANAGED
- COMPROMISED

## 35. Trust Properties

Ejemplos:

- credential_verified
- cryptographically_bound
- managed
- attested
- previous_strong_auth
- recent_strong_auth
- device_bound
- hardware_backed

## 36. DeviceTrustEvaluator

Contrato:

```php
interface DeviceTrustEvaluatorInterface
{
    public function evaluate(
        DeviceTrustContext $context
    ): DeviceTrustAssessment;
}
```

## 37. DeviceTrustContext

Podrá contener:

- DeviceReference
- verified device credential
- Identity
- Tenant
- Authentication history
- managed-device evidence
- Passkey evidence
- risk signals

## 38. Trust is derived

Nunca aceptar:

```php
trusted_device=true
desde frontend.
```

## 39. Trust policy

Un browser puede convertirse en trusted después de:

- successful strong MFA
- +;
- explicit user choice
- +;
- trusted-device credential issuance

## 40. Enrollment

Flow conceptual:

```text
Strong Authentication
        ↓
```

Policy allows device trust
↓
User selects "Trust this device"
↓
Device Identity created/resolved
↓
TrustedDeviceCredential issued
↓
credential cookie stored
↓
Device status = ACTIVE
↓
Trust = TRUSTED

## 41. Trust issuance prerequisites

Podrán incluir:

- minimum AAL
- fresh authentication
- no high-risk signals
- eligible Identity
- non-compromised session
- tenant policy

## 42. High-risk login

No deberá permitir automáticamente:
Trust this device

## 43. Explicit user choice

Dependiendo de UX/policy, device trust podrá requerir una acción explícita.

## 44. Auto-trust

Solo debería existir para escenarios controlados:

- enterprise managed device
- device attestation
- native app provisioning

## 45. Trusted device cookie

En browser:

- HttpOnly
- Secure
- SameSite appropriate
- host/domain scoped

## 46. Cookie name

Separado de:

- Session Cookie
- Remember-Me Cookie
- Recovery Cookie

## 47. Cookie secret rotation

Credential puede rotar periódicamente.

## 48. Rotation

Credential A
↓
successful validation
↓
Credential B issued
↓
A superseded

## 49. Replay detection

Si A reaparece después de B:

- PERSISTENT_DEVICE_CREDENTIAL_REPLAY
- puede generarse como señal.

## 50. Rotation family

Puede utilizar:
TrustedDeviceCredentialFamily

## 51. Credential theft

Una trusted-device cookie sigue siendo bearer material.
Debe tratarse como sensible.

## 52. Browser Device Credential assurance

No deberá afirmar:

- hardware-backed
- solo por existir una cookie.

## 53. Cryptographic Device Identity

Native applications podrán generar:

- device private key
- +;
- registered public key

## 54. Ventaja

Permite demostrar control mediante:

```text
challenge
    ↓
signature
```

sin bearer secret estático.

## 55. DeviceKeyCredential

Conceptualmente:

```php
final readonly class DeviceKeyCredential
{
    public function __construct(
        public DeviceCredentialId $id,
        public DeviceReference $device,
        public PublicKeyMaterial $publicKey,
        public DeviceCredentialStatus $status,
    ) {}
}
```

## 56. Private key

Nunca llega al servidor.

## 57. Key protection

Puede residir en:

- Secure Enclave
- TPM
- Android Keystore

Windows platform key store
application secure storage
según cliente.

## 58. Hardware-backed property

Solo deberá afirmarse cuando exista evidencia adecuada.

## 59. Device Challenge

Para cryptographic device auth:

```text
server challenge
    ↓
device signs
    ↓
public key verification
```

## 60. Challenge properties

random
short-lived
single-use
device-bound
purpose-bound

## 61. Device attestation

Algunas plataformas pueden demostrar propiedades del dispositivo.

## 62. DeviceAttestation

No será obligatoria para Core.

## 63. Attestation providers

Podrán existir adapters para:

- Apple App Attest
- Android Play Integrity-style evidence
- enterprise MDM
- TPM attestations

custom managed device systems
según disponibilidad y aplicación.

## 64. Attestation != Identity automatically

Una attestation demuestra propiedades de entorno/dispositivo.
No necesariamente quién es el usuario.

## 65. DeviceAttestationEvidence

Podrá producir:

- managed
- app_integrity
- hardware_backed
- device_compliance
- como propiedades.

## 66. AttestationVerifier

Contrato:

```php
interface DeviceAttestationVerifierInterface
{
    public function verify(
        DeviceAttestation $attestation,
        DeviceAttestationContext $context
    ): DeviceAttestationResult;
}
```

## 67. Provider trust

Toda attestation externa deberá validarse contra un provider/profile trusted.

## 68. Managed Device

Enterprise environments podrán marcar devices gestionados.

## 69. Managed does not automatically mean user authenticated

Solo aporta device trust.

## 70. Device Compliance

Podrá incluir:

- managed
- encrypted
- screen_lock_enabled
- OS compliant
- endpoint protection active

si trusted provider lo afirma.

## 71. Compliance is dynamic

Un device puede pasar de:
COMPLIANT
a:
NON_COMPLIANT

## 72. Runtime verification

High-security applications pueden revalidar compliance.

## 73. Cached compliance

Puede tener TTL.

## 74. Expired compliance

No deberá considerarse actual indefinidamente.

## 75. Device Identity Resolution

Será diferente según client type.

## 76. Browser resolution

Puede usar:

- TrustedDeviceCredential
- +;
- existing DeviceReference

## 77. Native app

Puede usar:
cryptographic device key

## 78. Managed endpoint

Puede usar:

- device certificate
- +;
- MDM identity

## 79. Unknown browser

No crear automáticamente durable DeviceIdentity por cada anonymous request.

## 80. Cardinality protection

Ataques podrían generar millones de "devices".

## 81. Device creation policy

Solo crear durable record cuando:

- successful authentication
- explicit enrollment
- trusted credential issuance
- o condición definida.

## 82. Anonymous candidate

Antes puede utilizarse:
DeviceObservation

## 83. DeviceObservation

Es request-scoped/transient.

## 84. Observation fields

Podrán incluir:

- browser family
- OS family
- language
- coarse client properties
- device cookie presence

## 85. Observation != DeviceIdentity

Muy importante.

## 86. DeviceRecognitionEngine

Podrá producir:

- UNRECOGNIZED
- PROBABLE_MATCH
- RECOGNIZED

## 87. Recognition sources

trusted device credential
known installation ID
cryptographic device credential
heuristic match

## 88. Strong recognition

Credential-based.

## 89. Weak recognition

Heuristic-based.

## 90. Recognition confidence

Debe expresarse.

## 91. DeviceMatchAssessment

final readonly class DeviceMatchAssessment
{
public function __construct(
public DeviceMatchType $type,
public SignalConfidence $confidence,
public ?DeviceReference $device,
) {}
}

## 92. DeviceMatchType

EXACT_CREDENTIAL_MATCH
CRYPTOGRAPHIC_MATCH
MANAGED_IDENTITY_MATCH
PROBABLE_HEURISTIC_MATCH
NO_MATCH

## 93. Heuristic fingerprint changes

No deberán revocar automáticamente device.

## 94. Example

Browser update:

```text
Chrome 151
→
Chrome 152
```

no implica necesariamente nuevo device.

## 95. Privacy rule

Core no deberá exigir high-entropy invasive fingerprinting.

## 96. Fingerprinting plugin

Si application lo desea, podrá registrar:
DeviceRecognitionSignalProvider

## 97. Device History

Cada Device puede mantener:

- firstSeenAt
- lastSeenAt
- lastAuthenticatedAt
- lastStrongAuthenticationAt

## 98. LastSeen write amplification

No actualizar DB en cada request.

## 99. Touch interval

Ejemplo:
update lastSeenAt at most once every N minutes

## 100. Security-critical updates

No deben retrasarse igual.

- Ejemplo:
- device COMPROMISED
- requiere persistencia inmediata.

## 101. Device Ownership

Un Device puede estar asociado a una o múltiples identities dependiendo del producto.

## 102. Personal application

Puede permitirse:

- same browser
- multiple users

## 103. Therefore

No modelar necesariamente:
Device belongs exactly to one Identity

## 104. DeviceAssociation

Mejor:

- Device
- ↔
- Identity

mediante entidad de relación.

## 105. DeviceIdentityAssociation

Conceptualmente:

```php
final readonly class DeviceIdentityAssociation
{
    public function__construct(
        public DeviceReference $device,
        public IdentityReference $identity,
        public DeviceAssociationStatus $status,
        public DeviceTrustLevel $trust,
    ) {}
}
```

## 106. Trust can be per Identity

Muy importante.
Un browser compartido puede ser:

- trusted for Alice
- not trusted for Bob

## 107. Tenant Trust

También puede ser:
trusted for Identity + Tenant

## 108. Device association scope

Podrá incluir:

- Identity
- Tenant
- Firewall
- Application

## 109. Recommended conceptual key

DeviceReference
+
IdentityReference
+
SecurityRealm

## 110. SecurityRealm

Evita que trust de:
consumer portal
se transfiera automáticamente a:
admin portal

## 111. Device Trust Policy

Contrato:

```php
interface DeviceTrustPolicyInterface
{
    public function evaluate(
        DeviceTrustPolicyContext $context
    ): DeviceTrustPolicyDecision;
}
```

## 112. Inputs

Identity type
Tenant
Firewall
Authentication Assurance
Authentication Risk
Device credential type
Device attestation
Last strong auth
Device age

## 113. Trust requirements

Ejemplo:

- AAL2
- AND
- fresh <= 10m
- AND
- risk <= MODERATE

## 114. Admin devices

Podrán exigir:

- Passkey
- managed device
- device-bound credential

## 115. Consumer devices

Pueden aceptar:

- Password + MFA
- para emitir trusted device cookie.

## 116. Trust lifetime

Debe ser explícita.

## 117. Example

30 days
90 days
session only

## 118. Non-expiring trusted devices

No recomendados como default.

## 119. DeviceTrustExpirationPolicy

Podrá limitar:

- absolute lifetime
- idle lifetime
- revalidation interval

## 120. Idle Trust Expiration

Ejemplo:

```text
not used for 60 days
    ↓
trust expires
```

## 121. Absolute expiration

Aunque se use continuamente, puede requerir revalidation periódica.

## 122. Trust renewal

Podrá requerir strong Authentication.

## 123. Rotation alone does not renew trust indefinitely

Importante.

## 124. Device version

Puede existir:
DeviceSecurityVersion

## 125. Use

Invalidar todas las device credentials asociadas tras compromise.

## 126. Simplicidad V1

Puede usarse:

- credential status
- +;
- Identity SecurityVersion

hasta necesitar más granularidad.

## 127. Device Revocation

Una Identity podrá seleccionar:
Forget this device

## 128. Effect

revoke trusted device credentials
remove/restrict trust association
optionally revoke device sessions

## 129. Device record

No necesita borrarse.
Puede conservarse para audit/history.

## 130. Revoked trust

No deberá restaurarse automáticamente solo porque aparece la misma fingerprint.

## 131. Re-enrollment

Necesita nueva strong Authentication.

## 132. Lost Device

Estado distinto de compromised.

## 133. Lost

Significa:
owner no longer controls device

## 134. Lost device response

Podrá:

- revoke trusted credentials
- revoke sessions

revoke device-bound application tokens
review Passkeys

## 135. Passkey nuance

Perder device no significa necesariamente perder synced Passkey.

## 136. Device vs Passkey

No modelarlos como uno-a-uno.

## 137. Passkey Association

Podrá existir metadata:

- Passkey observed from / associated with Device
- pero sin asumir exclusividad.

## 138. Device-bound Passkey

Solo cuando la credential realmente tenga propiedades que permitan dicha clasificación.

## 139. Synced Passkey

No puede identificarse como perteneciente a un único Device.

## 140. Device Compromise

Más fuerte que Lost.

## 141. COMPROMISED

Puede significar:

- malware
- stolen device credential
- session theft
- managed-device compromise
- cryptographic key compromise

## 142. Compromise response

Podrá incluir:

- revoke all device credentials
- terminate associated sessions
- remove trust
- raise Risk signals

increment SecurityVersion where policy requires
require Step-Up/Recovery

## 143. Compromised device signal

Debe integrarse con documento 20:
COMPROMISED_DEVICE

## 144. Device Trust + Risk

Dos ejes diferentes:

```text
Device Trust
    historical/credential confidence

Device Risk
    contextual current risk
```

## 145. Example

Trusted device
+
malicious network
+
session anomaly
puede tener:

```php
Trust = TRUSTED
Risk = HIGH
```

## 146. Trust does not cancel risk

Nunca.

## 147. New device

Puede tener:

```php
Trust = UNRECOGNIZED
Risk = MODERATE
```

## 148. Managed new device

Puede tener:

```php
Trust = MANAGED
Risk = LOW
```

si attestation/policy lo soporta.

## 149. Device Assurance Contribution

Un verified device credential puede contribuir a Authentication Assurance.

## 150. Pero no siempre como factor fuerte

Bearer trusted-device cookie podría aportar:
device_trust
sin convertirse en:
phishing_resistant possession

## 151. Factor mapping

DeviceTrustFactorAdapter podrá convertir ciertos proofs en:

- VerifiedFactor
- cuando policy lo permita.

## 152. Example

Cryptographic device key:

```php
category = POSSESSION
device_bound = true
```

## 153. Browser cookie

Podría ser:

- device_trust_evidence
- sin contar como independent strong factor.

## 154. MFA reduction

Trusted Device puede permitir:
do not challenge with TOTP every login

## 155. But base authentication still required

Ejemplo:

- Password
- +;
- Trusted Device

## 156. Sensitive route

Puede exigir:

- fresh Passkey
- sin importar trusted device.

## 157. Step-Up integration

Risk/route requirement puede ignorar device trust para ciertas operaciones.

## 158. Remember-Me + Trusted Device

Posible:

```text
Remember-Me
    restores Identity

Trusted Device
    contributes device trust

Risk
    evaluates context
```

## 159. Result

Puede permitir low-friction session restoration.

## 160. But no hidden full assurance

La nueva session deberá registrar provenance:

- remember_me
- trusted_device

## 161. Session Binding

Una session puede registrar:
DeviceReference

## 162. Binding policy

Puede ser:

- NONE
- SOFT
- STRICT

## 163. NONE

Session no depende del Device.

## 164. SOFT

Cambio de device/network produce:

- risk signal
- step-up

## 165. STRICT

Session solo es válida si se presenta device credential compatible.

## 166. Strict binding

Adecuado para ciertos:

- native apps
- admin consoles
- machine sessions

## 167. Browser strict binding caution

Puede degradar UX y generar lockouts por cambios legítimos.

## 168. SessionDeviceBinding

Conceptualmente:

```php
final readonly class SessionDeviceBinding
{
    public function __construct(
        public DeviceReference $device,
        public DeviceBindingMode $mode,
        public \DateTimeImmutable $boundAt,
    ) {}
}
```

## 169. Session restoration

Podrá comprobar:

- device credential still valid?
- Device not compromised?
- binding policy satisfied?

## 170. Session migration

No deberá ocurrir silenciosamente a otro Device en STRICT mode.

## 171. Multiple tabs

Mismo browser/device no debe crear problemas.

## 172. Browser profile reset

Puede perder trusted-device credential.
Entonces:

- Device becomes unrecognized from browser perspective
- aunque server tenga record histórico.

## 173. Device deletion by client

No equivale automáticamente a server revocation.

## 174. Device logout

Logout puede elegir:

- logout session only
- logout all sessions on device
- forget trusted device

## 175. Distinct operations

Muy importante.

## 176. Device Session Index

Para:
logout all sessions on this device
podría existir:
DeviceSessionIndex

## 177. Performance

No deberá requerir scan global de sessions.

## 178. DeviceCredentialRepository

Contrato:

```php
interface DeviceCredentialRepositoryInterface
{
    public function findBySelector(
        DeviceCredentialSelector $selector
    ): ?DeviceCredentialRecord;

    public function revokeForDevice(
        DeviceReference $device
    ): void;
}
```

## 179. DeviceRepository

interface DeviceRepositoryInterface
{
public function find(
DeviceId $id
): ?DeviceIdentity;

public function save(
DeviceIdentity $device
): void;
}

## 180. DeviceAssociationRepository

Separado para multi-user/shared device.

## 181. Device Enrollment Manager

Componente:
DeviceEnrollmentManager

## 182. Responsibilities

resolve/create Device
validate trust eligibility
issue credential
create Identity association
set trust metadata
audit

## 183. Device Trust Credential Issuer

Separado:
TrustedDeviceCredentialIssuer

## 184. Credential issuance flow

Authenticated Context
↓
Trust Policy
↓
Generate selector.secret
↓
Store digest
↓
Associate Device
↓
Set client cookie

## 185. Server-generated random

CSPRNG.

## 186. Token size

Debe tener suficiente entropía.

## 187. Credential storage

Nunca raw.

## 188. Credential comparison

Secure cryptographic primitive.

## 189. Device Public ID

Para management UI:
dev_public_xxx

## 190. No credential selector in UI

Preferir IDs separados.

## 191. Device Metadata

Puede incluir:

- display name
- browser family
- OS family
- device kind
- first seen
- last seen
- last authenticated
- trust state

## 192. Device Name

Ejemplo:

- Chrome on Windows
- Francisco's Laptop
- Work iPhone

## 193. Name is metadata

No security proof.

## 194. User-provided names

Podrán editarse.

## 195. Auto-generated names

Basados solo en coarse client metadata.

## 196. Never claim exact hardware without evidence

No mostrar:

- Dell XPS 15 serial XYZ
- si no se conoce.

## 197. Device Management API

Conceptualmente:

```php
Auth::devices()->current();
Auth::devices()->all();
Auth::devices()->trust(...);
Auth::devices()->forget(...);
Auth::devices()->revoke(...);
```

## 198. Authorization

Device management deberá pasar por Authorization.

## 199. Current Device

Debe resolverse desde verified credential/context, no solo fingerprint.

## 200. Device list

Mostrar:

- name
- kind
- trust
- last used
- current
- status

## 201. Sensitive metadata

Evitar mostrar:

- raw IP histories
- credential identifiers
- attestation blobs
- por default.

## 202. Device removal

Puede exigir:

- fresh Authentication
- especialmente para otros trusted devices.

## 203. Remove current trusted device

Puede revocar credential sin cerrar session actual, según policy.

## 204. Security settings

Una aplicación puede elegir:

```text
forget device
    → terminate its sessions too
```

## 205. Device Revocation Service

interface DeviceRevocationServiceInterface
{
public function revoke(
DeviceReference $device,
DeviceRevocationContext $context
): DeviceRevocationResult;
}

## 206. Revocation reason

USER_REVOKED
LOST
COMPROMISED
ADMIN_REVOKED
EXPIRED
TENANT_POLICY
DEVICE_NON_COMPLIANT

## 207. Compromise vs normal revoke

Debe conservarse reason.

## 208. Trust re-establishment

Después de normal revoke, device puede volver a confiarse con strong auth.

## 209. Compromised device

Puede requerir un nuevo DeviceIdentity/credential en vez de reactivar el mismo.

## 210. Tenant Device Policies

Multi-tenant applications podrán imponer:

- MFA every new device
- managed devices only
- trusted devices disabled
- maximum trusted devices
- device lifetime

## 211. Maximum device count

Puede ser una policy UX/security.

## 212. Attacker DoS caution

No permitir que un atacante agregue fake devices y bloquee al usuario sin haber autenticado fuertemente.

## 213. New device creation requires trusted context

Como regla.

## 214. Enterprise managed-only

Policy:

```text
admin firewall
    requires managed device
```

## 215. Device Requirement

Podrá integrarse con Authentication Requirements:

- ManagedDeviceRequirement
- TrustedDeviceRequirement
- DeviceBoundCredentialRequirement

## 216. Authentication vs Authorization boundary

Un route puede exigir:

- authenticated on managed device
- como Authentication requirement.

Pero permiso de negocio sigue en Authorization.

## 217. Risk integration

Device subsystem producirá señales:

- NEW_DEVICE
- KNOWN_DEVICE
- TRUSTED_DEVICE
- DEVICE_COMPROMISED
- DEVICE_NON_COMPLIANT
- DEVICE_CREDENTIAL_REPLAY

## 218. Risk engine interprets

No DeviceManager directamente.

## 219. Abuse Protection integration

Challenge/credential guessing deberá participar en documento 19.

## 220. Device credential failures

Podrán alimentar:

- TOKEN_GUESSING
- DEVICE_CREDENTIAL_ABUSE

## 221. Credential lookup

Selector permite lookup barato.

## 222. Unknown selectors

Rate limited.

## 223. Device credential rotation race

Dos requests concurrentes pueden presentar credential A.

## 224. Rotation strategy

Debe definir:

- grace window
- family version
- one successor
- replay detection

## 225. Concurrent legitimate requests

No deberán marcar compromise por cualquier race normal.

## 226. Rotation grace

Puede permitirse una ventana muy corta.

## 227. Strong replay signal

Credential superseded utilizada desde:

- different network/device context
- puede elevar suspicion.

## 228. Device Credential Family

FamilyId
current generation
previous generation

## 229. Generation

Cada rotation aumenta.

## 230. Duplicate successor prevention

Atomicity requerida.

## 231. Device Store

Deberá soportar concurrency control.

## 232. Device credential lastUsedAt

Touch interval.

## 233. Credential verification

Security status siempre authoritative.

## 234. Cache

Puede cachearse metadata.

## 235. Revocation-safe cache

Necesita:

- TTL
- version
- invalidation

## 236. High-security profile

Puede verificar credential record directamente.

## 237. Device Recognition cache

Heuristic summaries pueden cachearse.

## 238. Privacy retention

Device history no debe crecer indefinidamente.

## 239. DeviceHistoryRetentionPolicy

Podrá definir:

- last N events
- retention days
- aggregate-only after period

## 240. PII

Device metadata puede ser personal data.

## 241. Data minimization

No almacenar información innecesaria del browser/hardware.

## 242. Fingerprint raw components

No deberían persistirse por default.

## 243. Hash/pseudonymize where appropriate

Especialmente para recognition hints.

## 244. Device tracking transparency

Aplicaciones que utilicen fingerprinting avanzado deberían comunicarlo según sus obligaciones de privacidad.

## 245. Shared devices

Importantísimo para:

- family PC
- kiosk
- enterprise shared workstation

## 246. Shared Device policy

No confiar en Device a nivel global si múltiples identities lo usan.

## 247. Trust association per Identity

Soluciona este problema.

## 248. Public/Kiosk Device

Podrá marcarse:

- SHARED
- PUBLIC

## 249. Public devices

No deberían recibir:

- long-lived trusted device credentials
- por policy.

## 250. User choice

UI puede ofrecer:

- This is a shared/public device
- como hint.

## 251. Client hint not authoritative

Pero puede endurecer behavior.

## 252. "Trust this device"

Debería explicar duración.
Ejemplo:
Don't ask for secondary verification on routine sign-ins for 30 days.
No prometer bypass universal.

## 253. Device Trust UI

Debe distinguir:

- Current
- Trusted
- Last used
- Revoked
- Lost

## 254. Current session device

Puede resaltarse.

## 255. Device notifications

Eventos:

- new device trusted
- device revoked
- device marked lost
- device compromise detected

new managed device enrolled

## 256. Notification channel

Otro subsystem entrega.

## 257. Security notification

Puede permitir:

- This wasn't me
- que active revocation/recovery.

## 258. Notification action token

Debe usar un purpose-bound secure token.

## 259. Audit events

DeviceObserved
DeviceEnrolled
DeviceTrusted
DeviceTrustRenewed
DeviceCredentialRotated
DeviceRevoked
DeviceMarkedLost
DeviceMarkedCompromised
DeviceAttestationVerified
DeviceComplianceChanged
DeviceAuthenticationRejected

## 260. DeviceObserved audit

No cada request.
Puede ser telemetry.

## 261. Durable audit

Especialmente:

- trust
- revoke
- lost
- compromise
- admin changes

## 262. Observability spans

auth.device.resolve
auth.device.recognize
auth.device.credential.verify
auth.device.trust.evaluate
auth.device.enroll
auth.device.rotate
auth.device.revoke
auth.device.attestation.verify

## 263. Metrics

auth_device_recognition_total
auth_device_trust_total
auth_device_credential_failure_total
auth_device_rotation_total
auth_device_revocation_total
auth_device_replay_total
auth_device_attestation_total
auth_device_resolution_latency

## 264. Safe labels

device_kind
trust_level
credential_type
result
firewall

## 265. Avoid labels

DeviceId
IdentityId
credential selector
raw user-agent
IP

## 266. Device Recognition Telemetry

Puede medir:

- known-device rate
- new-device rate
- trusted-device rate

## 267. Device trust anomalies

Ejemplo:

- same trusted credential
- used from highly divergent contexts

## 268. DeviceCredentialReplayDetector

Podrá generar risk signal.

## 269. Security Event

DEVICE_CREDENTIAL_REPLAY_SUSPECTED

## 270. Device trust and IP binding

No bindear rígidamente a IP por default.

## 271. Razón

IPs cambian frecuentemente:

- mobile networks
- home DHCP
- VPN
- corporate egress

## 272. IP changes are Risk signals

Mejor.

## 273. Strong enterprise policy

Puede bindear a:

- network range
- managed network
- si necesario.

## 274. Geographic binding

Tampoco como default.

## 275. Device credential and User-Agent

No bindear estrictamente a exact UA string.

## 276. Browser updates

Romperían trust.

## 277. Soft metadata comparison

Puede alimentar DeviceRecognition/Risk.

## 278. Native Device binding

Puede ser más fuerte usando cryptographic installation key.

## 279. Device reinstall

Puede generar nuevo DeviceIdentity.

## 280. Restore from backup

Debe definirse por client ecosystem.
No asumir mismo private key.

## 281. Device transfer

No debería ocurrir para device-bound private key salvo client/platform flow explícito.

## 282. Device Lifecycle

OBSERVED
↓
ENROLLED
↓
ACTIVE
├── KNOWN
├── TRUSTED
└── MANAGED
│
├──→ LOST
├──→ REVOKED
├──→ COMPROMISED
└──→ RETIRED

## 283. OBSERVED

No necesariamente durable entity.

## 284. ENROLLED

Tiene una asociación/credential.

## 285. RETIRED

Dispositivo ya no utilizado.

## 286. Automatic retirement

Puede ocurrir después de larga inactividad.

## 287. Retired vs Revoked

Retired es lifecycle normal.
Revoked es security/control action.

## 288. Device reactivation

Retired puede volver a observarse, pero trust deberá reevaluarse.

## 289. Compromised terminal behavior

Preferentemente no reactivar directamente.

## 290. Device Trust State Machine

UNRECOGNIZED
↓
KNOWN
↓
TRUSTED
↓
STRONG/MANAGED

Any state
↓
RESTRICTED

Any trusted state
↓
REVOKED / COMPROMISED

## 291. Device Credential state machine

ISSUED
↓
ACTIVE
├──→ EXPIRED
├──→ ROTATED
├──→ REVOKED
├──→ COMPROMISED
└──→ SUPERSEDED

## 292. Testing — Recognition

Casos:

- no credential
- valid trusted credential
- unknown credential
- heuristic probable match

different Identity same browser

## 293. Testing — Trust

AAL1 cannot issue trust when AAL2 required
AAL2 low risk can issue
high risk denied
managed device auto-trust

## 294. Testing — Cookies

Secure
HttpOnly
SameSite
expiration
domain/path
rotation

## 295. Testing — Credential Theft

same credential
different contexts
replay after rotation
compromised status

## 296. Testing — Shared Browser

Alice trusts browser
Bob logs in
Bob does not inherit Alice trust

## 297. Testing — Multi-Tenant

Device trusted for Tenant A
not automatically trusted for Tenant B

## 298. Testing — Session Binding

NONE
SOFT
STRICT

## 299. Testing — Lost Device

Verify:

- credential revoked
- sessions according to policy
- risk signal generated

## 300. Testing — Compromised Device

Verify stronger invalidation.

## 301. Testing — Passkey

synced passkey not treated as single-device
device-bound passkey properties preserved

## 302. Testing — Remember-Me

remember-me + trusted device
remember-me without device credential
trusted device without remember-me

## 303. Testing — MFA

trusted device reduces routine challenge
sensitive route still requires fresh factor

## 304. Testing — Risk

trusted device + high-risk network
new device + strong passkey
compromised device + valid password

## 305. Testing — Attestation

valid
expired
untrusted provider
wrong app/device
malformed

## 306. Testing — Concurrency

credential rotation race
parallel verification
parallel trust issuance
duplicate device creation

## 307. Testing — Cardinality

Mass anonymous requests must not create millions of durable Device records.

## 308. Testing — FrankenPHP

Concurrent requests:

- Alice / Device A
- Bob / Device B

Anonymous / no device
No leakage.

## 309. Testing — Fiber Safety

Device context isolated.

## 310. Testing — Store Failure

Explicit failure policy.

## 311. Fuzz testing

Especialmente:

- device credential parser
- attestation payloads
- device metadata adapters
- client device IDs

## 312. Property-based testing

Útil para:

- credential rotation state machine
- trust expiration
- association isolation
- device state transitions

## 313. Security invariants — Device Identity

AUTH-DEVICE-ID-01
Device Identity is not inferred solely from client-controlled metadata.
AUTH-DEVICE-ID-02
Device fingerprinting does not establish cryptographic identity.
AUTH-DEVICE-ID-03
Unknown anonymous requests do not automatically create durable Device identities.
AUTH-DEVICE-ID-04
Device identity and Identity ownership are separate concepts.
AUTH-DEVICE-ID-05
Shared devices may have distinct trust relationships per Identity.

## 314. Security invariants — Device Trust

AUTH-DEVICE-TRUST-01
Known Device does not imply Trusted Device.

- AUTH-DEVICE-TRUST-02
- Trust is derived from explicit policy and verified evidence.
- AUTH-DEVICE-TRUST-03

High-risk Authentication cannot silently establish device trust.
AUTH-DEVICE-TRUST-04
Device trust expires/requires revalidation according to policy.
AUTH-DEVICE-TRUST-05
Trusted Device does not universally bypass MFA.
AUTH-DEVICE-TRUST-06
Trust for one security realm does not automatically transfer to another.

## 315. Security invariants — Device Credentials

AUTH-DEVICE-CRED-01
Raw trusted-device secrets are never persisted.

- AUTH-DEVICE-CRED-02
- Device credentials are distinct from Session and Remember-Me credentials.
- AUTH-DEVICE-CRED-03

Device credential verification is constant-time/cryptographically safe where applicable.
AUTH-DEVICE-CRED-04
Revoked/expired/compromised credentials do not contribute trust.
AUTH-DEVICE-CRED-05
Credential rotation is concurrency-safe.
AUTH-DEVICE-CRED-06
Replayed superseded credentials can produce security signals.

## 316. Security invariants — Cryptographic Devices

AUTH-DEVICE-CRYPTO-01
Device private keys never enter VoltStack.

- AUTH-DEVICE-CRYPTO-02
- Device challenge-response uses fresh purpose-bound challenges.
- AUTH-DEVICE-CRYPTO-03

Hardware-backed trust is not claimed without supporting evidence.
AUTH-DEVICE-CRYPTO-04
Attestation is validated through trusted providers/profiles.

## 317. Security invariants — Risk

AUTH-DEVICE-RISK-01
Device Trust and Device Risk remain separate dimensions.

- AUTH-DEVICE-RISK-02
- Trusted Device does not erase current high-risk signals.
- AUTH-DEVICE-RISK-03

New Device is contextual risk, not proof of attack.
AUTH-DEVICE-RISK-04
Compromised Device produces explicit security signals.

## 318. Security invariants — Runtime

AUTH-DEVICE-RT-01
Current DeviceContext is request-scoped.

- AUTH-DEVICE-RT-02
- Shared Device services remain stateless or immutable.
- AUTH-DEVICE-RT-03

No current Device/Identity state survives FrankenPHP request boundaries.
AUTH-DEVICE-RT-04
Concurrent fibers maintain isolated device contexts.
AUTH-DEVICE-RT-05
Distributed credential state is authoritative across application nodes.

## 319. Privacy invariants

AUTH-DEVICE-PRIV-01
Device metadata collection follows data minimization.

- AUTH-DEVICE-PRIV-02
- Fingerprinting is optional and cannot be a mandatory root-of-trust.
- AUTH-DEVICE-PRIV-03

Raw fingerprint attributes are not stored indefinitely by default.
AUTH-DEVICE-PRIV-04
Tenant device data remains isolated.
AUTH-DEVICE-PRIV-05
Device management APIs expose only necessary metadata.

## 320. Anti-pattern — Fingerprint equals trusted device

Incorrecto.

## 321. Anti-pattern — Cookie equals physical machine

Incorrecto.

## 322. Anti-pattern — Known = Trusted

Nunca.

## 323. Anti-pattern — Trusted Device = MFA disabled

No.

## 324. Anti-pattern — Session ID reused as trusted-device ID

No.

## 325. Anti-pattern — Remember-Me reused as device credential

No.

## 326. Anti-pattern — Trust stored as boolean on User

Evitar:
$user->trusted_device = true;

## 327. Anti-pattern — one Device belongs to one User universally

No para browsers compartidos.

## 328. Anti-pattern — exact User-Agent binding

Demasiado frágil.

## 329. Anti-pattern — IP binding by default

Demasiado frágil.

## 330. Anti-pattern — synced Passkey means device-bound

Incorrecto.

## 331. Anti-pattern — managed means authenticated user

No.

## 332. Anti-pattern — all anonymous visits create Device rows

Storage/cardinality problem.

## 333. Anti-pattern — revocation by deleting record only

Pierde lifecycle/audit y puede complicar replay detection.

## 334. Anti-pattern — process-global current device

Crítico bajo FrankenPHP.

## 335. Componentes principales

DeviceIdentity
DeviceId
DeviceReference
DeviceKind
DeviceStatus

DeviceObservation
DeviceRecognitionEngine
DeviceMatchAssessment
DeviceMatchType

DeviceIdentityAssociation
DeviceAssociationStatus

DeviceTrustAssessment
DeviceTrustLevel
DeviceTrustPropertySet
DeviceTrustEvaluator
DeviceTrustPolicy

## 336. Credential components

DeviceCredential
DeviceCredentialId
DeviceCredentialStatus
TrustedDeviceCredential
TrustedDeviceCredentialFamily
DeviceKeyCredential

TrustedDeviceCredentialIssuer
DeviceCredentialVerifier
DeviceCredentialRotator
DeviceCredentialRepository

## 337. Management components

DeviceEnrollmentManager
DeviceRevocationService
DeviceTrustManager
DeviceRepository
DeviceAssociationRepository
DeviceSessionIndex
DeviceManagementService

## 338. Attestation components

DeviceAttestation
DeviceAttestationEvidence
DeviceAttestationVerifier
DeviceAttestationProvider
DeviceComplianceAssessment
ManagedDeviceProfile

## 339. Namespace sugerido

VoltStack\Quantum\Auth\Device
VoltStack\Quantum\Auth\Device\Contracts
VoltStack\Quantum\Auth\Device\Identity
VoltStack\Quantum\Auth\Device\Recognition
VoltStack\Quantum\Auth\Device\Trust
VoltStack\Quantum\Auth\Device\Credential
VoltStack\Quantum\Auth\Device\Attestation
VoltStack\Quantum\Auth\Device\Management
VoltStack\Quantum\Auth\Device\Session
VoltStack\Quantum\Auth\Device\Policy

## 340. Estructura sugerida

src/Quantum/Auth/Device/
├── Contracts/
│   ├── DeviceRepositoryInterface.php
│   ├── DeviceAssociationRepositoryInterface.php
│   ├── DeviceCredentialRepositoryInterface.php
│   ├── DeviceTrustEvaluatorInterface.php
│   ├── DeviceTrustPolicyInterface.php
│   └── DeviceAttestationVerifierInterface.php
│
├── Identity/
│   ├── DeviceIdentity.php
│   ├── DeviceId.php
│   ├── DeviceReference.php
│   ├── DeviceKind.php
│   └── DeviceStatus.php
│
├── Recognition/
│   ├── DeviceObservation.php
│   ├── DeviceRecognitionEngine.php
│   ├── DeviceMatchAssessment.php
│   └── DeviceMatchType.php
│
├── Trust/
│   ├── DeviceTrustAssessment.php
│   ├── DeviceTrustLevel.php
│   ├── DeviceTrustPropertySet.php
│   ├── DeviceTrustEvaluator.php
│   └── DeviceTrustPolicy.php
│
├── Credential/
│   ├── DeviceCredential.php
│   ├── DeviceCredentialId.php
│   ├── DeviceCredentialStatus.php
│   ├── TrustedDeviceCredential.php
│   ├── TrustedDeviceCredentialFamily.php
│   ├── DeviceKeyCredential.php
│   ├── TrustedDeviceCredentialIssuer.php
│   ├── DeviceCredentialVerifier.php
│   └── DeviceCredentialRotator.php
│
├── Attestation/
│   ├── DeviceAttestation.php
│   ├── DeviceAttestationEvidence.php
│   ├── DeviceAttestationVerifier.php
│   ├── DeviceComplianceAssessment.php
│   └── ManagedDeviceProfile.php
│
├── Management/
│   ├── DeviceEnrollmentManager.php
│   ├── DeviceTrustManager.php
│   ├── DeviceRevocationService.php
│   └── DeviceManagementService.php
│
├── Session/
│   ├── SessionDeviceBinding.php
│   ├── DeviceBindingMode.php
│   └── DeviceSessionIndex.php
│
└── Policy/
├── DeviceEnrollmentPolicy.php
├── DeviceTrustExpirationPolicy.php
└── DeviceRequirement.php

## 341. Configuración conceptual

return [

'authentication' => [

'devices' => [

'enabled' => true,

'recognition' => [
'trusted_credential' => true,
'heuristics' => false,
],

'trust' => [
'enabled' => true,
'default_lifetime' => '30 days',
'rotation' => true,

'minimum_assurance' => 'aal2',

'max_risk' => 'moderate',
],

],

],

];

## 342. Configuración Admin

'admin' => [

'device' => [
'require_managed' => true,
'require_cryptographic_binding' => true,
'trusted_device_cookie' => false,
],

];

## 343. Configuración Tenant

'tenant_device_policy' => [

'trust_enabled' => true,

'maximum_trusted_devices' => 10,

'new_device_requires_step_up' => true,

'trust_lifetime' => '30 days',

];

## 344. Flujo Trusted Browser

Password Authentication
↓
TOTP verified
↓
AAL2
↓
Risk LOW
↓
User selects:

```php
    Trust this device
        ↓
DeviceEnrollmentManager
        ↓
DeviceIdentity created
        ↓
Identity association created
        ↓
TrustedDeviceCredential issued
        ↓
cookie stored
        ↓
future login
        ↓
credential verified
        ↓
DeviceTrust = TRUSTED
```

## 345. Flujo future login

Password valid
+
TrustedDeviceCredential valid
↓
Device recognized
↓
Risk Engine
↓
LOW
↓
MFA Policy
↓
routine secondary challenge may be skipped
↓
AuthenticationContext

## 346. Flujo high-risk trusted device

TrustedDeviceCredential valid
↓
same Device known
↓
Network = malicious
↓
Risk HIGH
↓
Trusted Device DOES NOT override Risk
↓
Passkey Step-Up required

## 347. Flujo device credential replay

Credential generation 12
↓
rotated to generation 13
↓
old generation 12 later reused
↓
Replay Detector
↓
DEVICE_CREDENTIAL_REPLAY_SUSPECTED
↓
Risk HIGH
↓
credential family revoked
↓
step-up / recovery

## 348. Flujo lost device

User opens Security Settings
↓
selects Laptop
↓
Mark Lost
↓
DeviceRevocationService
↓
trusted credentials revoked
↓
sessions terminated according to policy
↓
Device status = LOST
↓
risk signals updated

## 349. Flujo cryptographic native device

Application installation
↓
Device generates key pair
↓
private key stays client-side
↓
public key registered
↓
DeviceIdentity created
↓
later authentication
↓
server challenge
↓
device signs
↓
signature verified
↓
Device Trust Evidence

## 350. Flujo managed enterprise device

Authentication Request
↓
Managed Device Credential
↓
MDM / attestation verification
↓
Device Compliance = VALID
↓
Device Trust = MANAGED
↓
Risk LOW
↓
Admin Authentication Requirement satisfied
si los demás requisitos de Authentication también son válidos.

## 351. Arquitectura global

CLIENT DEVICE
│
┌─────────┴─────────┐
▼                   ▼
Device Metadata      Device Credential
│                   │
▼                   ▼
DeviceObservation    CredentialVerifier
│                   │
└─────────┬─────────┘
▼
DeviceRecognitionEngine
│
▼
DeviceReference
│
┌───────────────┼────────────────┐
▼               ▼                ▼
History          Attestation      Association
│               │                │
└───────────────┼────────────────┘
▼
DeviceTrustEvaluator
│
▼
DeviceTrustAssessment
│
┌────────────┴────────────┐
▼                         ▼
Risk Engine              MFA / Step-Up
│                         │
└────────────┬────────────┘
▼
AuthenticationContext
│
▼
AuthenticationSession

## 352. Decisiones arquitectónicas principales

VoltStack adoptará:

1. Device Identity, Device Recognition, Device Trust and Device Risk are separate concepts.
2. Known Device never automatically means Trusted Device.
3. Browser fingerprints are heuristic signals, not credentials.
4. Trusted-device credentials are distinct from Session and Remember-Me credentials.
5. Device trust is scoped to Identity/security realm where appropriate.
6. Shared devices are first-class scenarios.
7. Trusted devices never universally bypass MFA.
8. Strong device trust may use cryptographic credentials or managed-device attestation.
9. Synced Passkeys are not treated as one-device credentials.
10. Device compromise can invalidate sessions and persistent credentials.
11. Anonymous traffic does not create unbounded durable Device records.
12. Device state is safe under FrankenPHP, fibers and distributed deployments.
13. Relación conceptual con Laravel y Symfony

Laravel y Symfony permiten implementar dispositivos confiables mediante:

- cookies
- remember-me
- custom middleware
- security events
- custom authenticators

pero estos conceptos suelen quedar definidos por la aplicación o paquetes externos.
VoltStack formalizará:

- DeviceIdentity
- DeviceAssociation
- DeviceRecognition
- DeviceCredential
- DeviceTrust
- DeviceAttestation
- DeviceSessionBinding
- DeviceLifecycle

como primitives del Authentication Core.
La diferencia principal será evitar:
remember_me == trusted_device
y construir:

```text
Remember-Me
    identity persistence

Trusted Device
    device trust

Session
    authenticated continuity

Passkey
    authentication credential

Risk
    contextual suspicion

MFA
    evidence composition
```

como subsistemas relacionados pero separados.

## 354. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte DeviceIdentity;
2. soporte DeviceReference;
3. soporte DeviceKind;
4. soporte DeviceStatus;
5. soporte transient DeviceObservation;
6. soporte DeviceRecognition;
7. soporte recognition confidence;
8. distinga Known y Trusted;
9. soporte DeviceIdentityAssociation;
10. soporte shared devices;
11. soporte per-Identity trust;
12. soporte tenant/security realm scoping;
13. soporte DeviceTrustAssessment;
14. soporte DeviceTrustPolicy;
15. soporte trust lifetime;
16. soporte idle/absolute expiry;
17. soporte trusted-device cookies;
18. almacene solo secret digests;
19. soporte credential rotation;
20. soporte replay detection;
21. soporte cryptographic Device credentials;
22. soporte device public keys;
23. soporte Device Attestation extensible;
24. soporte Managed Devices;
25. soporte Device Compliance;
26. soporte Device Enrollment;
27. soporte Device Revocation;
28. soporte Lost Device;
29. soporte Compromised Device;
30. soporte Device Management API;
31. soporte session-device binding;
32. soporte logout por device;
33. integre Remember-Me;
34. integre MFA;
35. integre Step-Up;
36. integre Passkeys sin confundir synced/device-bound;
37. integre Risk Engine;
38. integre Abuse Protection;
39. soporte tenant policies;
40. proteja privacidad;
41. evite durable cardinality abuse;
42. soporte audit;
43. soporte metrics;
44. soporte tracing;
45. soporte extensibilidad;
46. sea concurrency-safe;
47. sea fiber-safe;
48. sea seguro bajo FrankenPHP.
49. Regla arquitectónica final

VoltStack deberá preservar:

```text
DEVICE OBSERVATION
        ↓
DEVICE RECOGNITION
        ↓
DEVICE REFERENCE
        ↓
VERIFIED DEVICE EVIDENCE
        ↓
DEVICE TRUST EVALUATION
        ↓
DEVICE TRUST ASSESSMENT
        │
        ├──────────────► RISK ENGINE
        │
        ├──────────────► MFA / STEP-UP
        │
        ├──────────────► SESSION BINDING
        │
        └──────────────► PERSISTENT LOGIN POLICY
                               │
                               ▼
                    AUTHENTICATION CONTEXT
```

La primera regla central será:
VoltStack jamás utilizará “este navegador parece conocido” como equivalente a “este dispositivo está autenticado y es confiable”.

La segunda será:
Device Trust será el resultado de evidencia verificable y políticas explícitas, mientras que Device Recognition podrá utilizar señales heurísticas con confidence limitada.

La tercera:

```php
Trusted Device no será un alias de Remember-Me ni un bypass permanente de MFA; será una fuente de evidencia contextual que puede reducir fricción únicamente cuando la política de Authentication y Risk lo permita.

Siguiente documento recomendado
```

La secuencia natural continúa con:
`22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
Este documento deberá unificar cómo todos los subsistemas construidos hasta ahora se exponen finalmente al desarrollador y al usuario mediante flujos de entrada y salida:

- Login / Sign-In
- Logout / Sign-Out
- Authentication Entry Points
- Login Controllers
- Authentication Flow Coordinator
- Form Login
- JSON/API Login
- Passkey Login
- Federated Login
- Remember-Me Restoration
- MFA Continuation
- Step-Up Entry Points
- Recovery Entry Points
- Authentication Success Handling
- Authentication Failure Handling
- Intended Destination
- Redirect Security
- SPA Authentication Protocol
- Logout Semantics
- Current Session Logout
- All Sessions Logout
- Device Logout
- Global Logout
- Federated Logout Boundaries
- Session Rotation
- Credential Cleanup
- CSRF
- Events
- Hooks
- Customization
- Audit
- Testing
- FrankenPHP Safety

Con el documento 22 podremos tomar todas las primitives construidas entre 00 y 21 y definir finalmente el flujo completo de Sign-In/Sign-Out que utilizará una aplicación VoltStack, sin volver a mezclar la lógica de Password, MFA, Risk, Session, Device, Passkeys o Federation dentro de un único LoginController.
