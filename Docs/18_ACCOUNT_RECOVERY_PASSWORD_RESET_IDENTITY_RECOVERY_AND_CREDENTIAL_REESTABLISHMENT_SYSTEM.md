# VoltStack Authentication System

## 18 — Account Recovery, Password Reset, Identity Recovery and Credential Re-establishment System

- **Archivo:** `18_ACCOUNT_RECOVERY_PASSWORD_RESET_IDENTITY_RECOVERY_AND_CREDENTIAL_REESTABLISHMENT_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de recuperación de cuentas, restablecimiento de credenciales y reestablecimiento seguro de acceso

**Depende especialmente de:**

- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`
- `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md`
- `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de recuperar acceso a una Identity cuando una o varias credenciales normales ya no pueden utilizarse.
El sistema deberá cubrir:

- Forgot Password
- Password Reset
- Lost Password
- Lost MFA Device
- Lost Passkeys
- Lost Security Keys
- Compromised Credentials
- Locked-out Identity
- Recovery Codes
- Email Recovery
- Recovery Links
- Administrative Recovery
- Federated Recovery
- Identity Re-verification
- Credential Re-establishment
- Factor Re-enrollment
- Recovery Security State
- Recovery Session Invalidation
- Persistent Credential Invalidation
- API Token Invalidation
- Recovery Notifications
- Recovery Abuse Protection

El principio central será:
Recovery no será un bypass de Authentication. Será un mecanismo de Authentication especializado cuyo propósito es restablecer un conjunto seguro de credenciales.

## 2. Regla de seguridad fundamental

La seguridad efectiva de Authentication queda limitada por su método de recuperación más débil.
Conceptualmente:

```text
Strong Password

+

Passkey
+
MFA
+
Hardware Security Key
+
Weak Email Recovery
        ↓
Effective Account Security
≈
Weak Email Recovery
```

Por tanto:
VoltStack deberá tratar Account Recovery como un subsistema de seguridad de primer nivel, no como una función auxiliar de “Forgot Password”.

## 1. Recovery vs Authentication normal

Authentication normal pregunta:

- Can this actor prove control
- of an existing Authentication Credential?

Recovery pregunta:

- Can this actor provide sufficient
- recovery evidence to regain control
- of the Identity?

## 2. Recovery vs Credential Reset

También se distinguirá:

```text
ACCOUNT RECOVERY
    process for re-establishing control

PASSWORD RESET
    replacement of PasswordCredential

FACTOR RESET
    removal/replacement of MFA factors

CREDENTIAL RE-ESTABLISHMENT
    creation of new trusted credentials
    after recovery
```

## 5. Recovery no equivale a Password Reset

Una Identity puede ser:

- Passkey-only
- Federated-only

MFA-only after base federation
Service Identity
Passwordless
Por tanto, el dominio deberá ser:
Identity Recovery
y no solamente:
Password Reset

## 6. Arquitectura general

Recovery Request
│
▼
Recovery Manager
│
▼
Identity Claim Resolution
│
▼
Recovery Policy Resolution
│
▼
Recovery Transaction
│
▼
Recovery Evidence Collection
│
├── Email Link
├── Recovery Code
├── Existing Factor
├── Passkey
├── Federated Identity
├── Administrative Approval
└── Identity Verification
│
▼
Recovery Assurance Evaluation
│
▼
Recovery Decision
│
├── MORE_EVIDENCE_REQUIRED
├── DENIED
└── RECOVERY_AUTHORIZED
│
▼
Credential Re-establishment
│
▼
Security Invalidation
│
▼
Recovery Complete

## 7. RecoveryManager

Será el orquestador principal.

```php
Conceptualmente:
interface RecoveryManagerInterface
{
    public function begin(
        RecoveryRequest $request
    ): RecoveryStartResult;

    public function continue(
        RecoveryContinuationRequest $request
    ): RecoveryResult;
}
```

## 8. Responsibilities

RecoveryManager coordinará:

- identity discovery
- recovery policy
- transaction lifecycle
- evidence collection
- challenge generation
- assurance evaluation
- credential re-establishment
- security invalidation
- audit
- notifications

No implementará directamente criptografía ni envío de email.

## 9. RecoveryTransaction

Todo recovery multi-request deberá vivir dentro de una:
RecoveryTransaction

## 10. RecoveryTransaction model

Conceptualmente:

```php
final readonly class RecoveryTransaction
{
    public function __construct(
        public RecoveryTransactionId $id,
        public RecoveryPurpose $purpose,
        public RecoveryTransactionStatus $status,
        public ?IdentityReference $identity,
        public RecoveryRequirementSet $requirements,
        public RecoveryEvidenceSet $evidence,
        public \DateTimeImmutable $createdAt,
        public \DateTimeImmutable $expiresAt,
        public int $version,
    ) {}
}
```

## 11. RecoveryPurpose

Valores posibles:

- PASSWORD_RESET
- ACCOUNT_RECOVERY
- MFA_RECOVERY
- PASSKEY_RECOVERY
- FACTOR_RESET
- CREDENTIAL_REPLACEMENT
- COMPROMISE_RECOVERY
- ADMINISTRATIVE_RECOVERY

## 12. Purpose binding

Una transaction iniciada para:
PASSWORD_RESET
no deberá poder utilizarse para:
MFA_RESET
o:

- ACCOUNT_TAKEOVER_REPLACEMENT
- sin policy explícita.

## 13. RecoveryTransactionStatus

PENDING
CHALLENGED
EVIDENCE_PARTIAL
AUTHORIZED
COMMITTING
COMPLETED
DENIED
EXPIRED
CANCELLED
COMPROMISED

## 14. Terminal states

Estados como:

- COMPLETED
- DENIED
- EXPIRED
- CANCELLED
- COMPROMISED

no deberán regresar a PENDING.

## 15. Recovery Transaction ID

Debe ser:

- opaque
- high entropy
- unguessable
- non-semantic

## 16. Public Transaction Reference

Puede existir:

- RecoveryTransactionPublicId
- distinto de cualquier secret de recuperación.

## 17. Transaction TTL

Debe ser corto y configurado según tipo de recovery.

## 18. Transaction storage

Podrá usar:

- Redis
- database
- distributed KV

## 19. Distributed applications

No deberá depender de memoria de proceso.

- Especialmente bajo:
- FrankenPHP
- multiple workers
- horizontal scaling

## 20. Recovery Request

Un usuario puede comenzar con un claim:

- email
- username
- phone
- organization login
- external identifier

## 21. Recovery Identity Claim

Debe reutilizar:

- IdentityClaim
- cuando sea posible.

## 22. Enumeration resistance

El sistema no deberá revelar fácilmente:

- account exists
- account does not exist
- account has password
- account uses MFA
- account has passkeys

## 23. Generic external response

Ejemplo:

```php
If an eligible account matches the information provided,
recovery instructions will be sent.
```

## 24. Internal behavior

Internamente sí podrá distinguir:

- IDENTITY_FOUND
- IDENTITY_NOT_FOUND
- RECOVERY_NOT_ALLOWED
- DELIVERY_NOT_AVAILABLE

## 25. Timing differences

Deberán minimizarse diferencias obvias entre:

- unknown identity
- known identity

especialmente antes de enviar una response.

## 26. No fake recovery commitment

No afirmar al usuario que se envió un email si el sistema no lo intentó, pero el mensaje externo puede mantenerse genérico.

## 27. Recovery Eligibility

Antes de crear challenges deberá evaluarse:

- Identity Security State
- Recovery Policy
- Identity type
- Tenant policy
- Available recovery methods
- Compromise state

## 28. Deleted Identity

Normalmente:
recovery denied

## 29. Disabled Identity

Dependerá de semántica.
Ejemplo:

- administratively disabled employee
- no debería poder reactivar su cuenta mediante password reset.

## 30. Suspended Identity

Recovery puede permitirse para:

- credential repair
- sin reactivar Authentication normal.

## 31. Compromised Identity

Puede requerir un recovery profile más fuerte.

## 32. RecoverySecurityState

Podrá existir un estado separado:

- NONE
- RECOVERY_REQUIRED
- RECOVERY_IN_PROGRESS
- RECOVERY_VERIFIED
- REESTABLISHMENT_REQUIRED
- RECOVERY_COMPLETED

## 33. Identity Security integration

IdentitySecurityState podrá incluir:

- RecoveryState
- sin almacenar toda la transaction.

## 34. RecoveryRequired

Puede activarse por:

- password compromise
- MFA reset
- admin action
- session hijack
- account takeover suspicion
- lost credentials

## 35. Normal login during RecoveryRequired

Podrá:

- deny normal login
- redirect to recovery
- require additional proof
- según policy.

## 36. RecoveryRequirement

Representará qué evidencia se necesita.

- Ejemplos:
- EmailOwnershipRequirement
- RecoveryCodeRequirement
- ExistingFactorRequirement
- FederatedIdentityRequirement
- PasskeyRequirement
- AdministrativeApprovalRequirement
- IdentityProofingRequirement

## 37. RecoveryRequirementSet

Podrá permitir composición:

- ALL_OF
- ANY_OF
- AT_LEAST_N
- CUSTOM

## 38. Recovery assurance

No basta con:
has recovery token
El sistema deberá evaluar qué nivel de confianza genera la evidencia.

## 39. RecoveryAssurance

Conceptualmente:

```php
final readonly class RecoveryAssurance
{
    public function __construct(
        public RecoveryAssuranceLevel $level,
        public RecoveryAssurancePropertySet $properties,
    ) {}
}
```

## 40. Recovery assurance levels

Ejemplo conceptual:

- LOW
- STANDARD
- STRONG
- VERY_STRONG

## 41. Recovery properties

Ejemplos:

- email_control
- existing_factor
- recovery_code
- federated_control
- hardware_factor
- administrative_approval
- identity_proofed
- multi_evidence

## 42. Recovery policy

Una operación podrá exigir:

- minimum recovery assurance
- specific evidence
- multiple independent methods
- freshness
- administrative approval
- cooldown

## 43. Recovery Evidence

Toda prueba verificada producirá:
RecoveryEvidence

## 44. RecoveryEvidence types

EmailRecoveryEvidence
RecoveryCodeEvidence
ExistingFactorRecoveryEvidence
PasskeyRecoveryEvidence
FederatedRecoveryEvidence
AdministrativeRecoveryEvidence
IdentityProofingEvidence

## 45. Evidence is transaction-bound

Una evidencia obtenida en una recovery transaction no deberá reutilizarse arbitrariamente en otra.

## 46. Evidence freshness

Debe registrar:
verifiedAt

## 47. Evidence expiration

Podrá expirar antes que la transaction completa.

## 48. Email Recovery

El mecanismo más común será:
Email Recovery Link

## 49. Email Recovery no será universal

Una Identity puede:

- have no email
- not trust email for recovery
- require stronger recovery

## 50. EmailRecoveryCredential

Se generará una credential temporal de alta entropía.

## 51. Token design

Preferible:

- selector.secret
- o un opaque token equivalente.

## 52. Raw recovery token

Solo se envía al canal de recuperación.
No deberá almacenarse en plaintext.

## 53. Server-side storage

Guardar:

- selector
- secret digest
- identity reference
- transaction reference
- purpose
- expiration
- status

## 54. RecoveryTokenStatus

ACTIVE
CONSUMED
EXPIRED
REVOKED
SUPERSEDED
COMPROMISED

## 55. Single-use

Un recovery token deberá ser:
single-use

## 56. Token replay

Después de uso:

- CONSUMED
- Una reutilización deberá rechazarse.

## 57. Recovery token lifetime

Debe ser corto y configurable.

## 58. Recovery Token Repository

Conceptualmente:

```php
interface RecoveryTokenRepositoryInterface
{
    public function findBySelector(
        RecoveryTokenSelector $selector
    ): ?RecoveryTokenRecord;

    public function consume(
        RecoveryTokenId $id
    ): RecoveryTokenConsumeResult;
}
```

## 59. Atomic consume

Es crítico.
Dos requests concurrentes no deben utilizar exitosamente el mismo recovery token.

## 60. Recovery Token hashing

Como secret aleatorio de alta entropía podrá almacenarse mediante:

- cryptographic digest
- HMAC
- según policy.

No requiere necesariamente password hashing adaptativo.

## 61. Recovery URL

Ejemplo conceptual:
<https://example.com/recovery/continue?token=>...

## 62. Token in URL risk

Los recovery links suelen necesitar transportar token en URL, por lo que deberán considerarse fugas por:

- browser history
- proxy logs
- referrer
- analytics
- screenshots

## 63. Recovery landing strategy

Una estrategia más segura será:

```text
GET recovery URL
    ↓
validate/exchange token
    ↓
```

create server-side RecoveryTransaction session
↓
immediately redirect to clean URL

## 64. Referrer policy

Las páginas de recovery deberán utilizar policies adecuadas para evitar leakage.

## 65. No third-party resources

Preferible minimizar:

- analytics
- ads
- third-party scripts
- external images

en páginas que reciben recovery tokens.

## 66. Cache policy

Recovery pages deberán utilizar:

- Cache-Control: no-store
- cuando corresponda.

## 67. Token URL after exchange

Debe desaparecer del navegador mediante redirect limpio.

## 68. Recovery email delivery

Auth generará un:
RecoveryDeliveryRequest

## 69. Notification boundary

El subsistema de Messaging/Notifications será quien entregue:

- email
- SMS
- push

## 70. Recovery link host

Debe generarse desde configuración trusted.
Nunca desde Host no confiable.

## 71. Host header poisoning

Critical.
No construir:

```php
$url = 'https://' . $request->getHost() . '/reset?...';
sin trusted host architecture.
```

## 72. RecoveryUrlGenerator

Debe utilizar:
TrustedApplicationUrlResolver

## 73. Recovery email content

No deberá incluir:

- existing password
- security answers
- sensitive account data

## 74. Email replay

Una copia antigua del email dejará de servir una vez token:

- consumed
- expired
- revoked

## 75. Resend recovery link

Nueva solicitud podrá:

- invalidate previous tokens
- o permitir pocos simultáneos según policy.

## 76. Recommended default

Preferible:

- new token supersedes previous active tokens
- para misma Identity + purpose.

## 77. Email flood protection

Debe existir rate limiting para:

- identity claim
- source IP
- tenant
- delivery destination

## 78. Enumeration-safe rate limit

No revelar si el account existía por diferencias de rate-limit message.

## 79. Recovery spam

El sistema deberá impedir utilizar Forgot Password como plataforma de spam.

## 80. Delivery cooldown

Ejemplo:
one email per bounded interval

## 81. Recovery Code

Los Recovery Codes del sistema MFA podrán utilizarse como recovery evidence.

## 82. Recovery code properties

high entropy
one-time
stored only as hash
identity-bound

## 83. Recovery code use

Debe ser:
atomic consume

## 84. Recovery code is not password

No deberá establecer inmediatamente Authentication normal sin RecoveryPolicy.

## 85. Existing Factor Recovery

Una Identity puede recuperar Password mediante:

- existing Passkey
- TOTP
- Security Key

## 86. Example

Forgot password
↓
authenticate using Passkey
↓
RecoveryAssurance STRONG
↓
allow new password creation

## 87. Existing factor reuse

El factor debe ser verificado de nuevo.
No confiar solo en que:
Identity has Passkey

## 88. Session-based recovery

Una sesión ya autenticada puede permitir cambiar Password, pero eso pertenece más naturalmente a:

- Password Change
- no recovery.

## 89. Fresh authentication

Si una sesión autenticada inicia credential recovery:

- fresh proof
- podrá ser requerida.

## 90. Federated Recovery

Una Identity local vinculada a OIDC puede utilizar el proveedor federado como recovery evidence.

## 91. Example

Password lost
↓
Authenticate with Corporate SSO
↓
issuer + subject mapping confirmed
↓
federated recovery evidence
↓
new local PasswordCredential allowed
solo si policy lo permite.

## 92. Federated-only Identity

No debería recibir una password local solo porque hizo recovery, salvo Enrollment Policy explícita.

## 93. Federation as authoritative recovery

Para enterprise SSO puede ser el mecanismo principal.

## 94. Provider outage

No deberá abrir automáticamente recovery más débil.

## 95. Lost MFA Recovery

Caso:

- password valid
- TOTP device lost

## 96. MFA recovery flow

Primary Authentication
↓
MFA cannot be completed
↓
Recovery Flow
↓
Recovery Code / Passkey / Federation / Admin
↓
Recovery Assurance
↓
MFA Reset Authorized
↓
old MFA credentials revoked
↓
new factor enrollment required

## 97. MFA reset is not normal MFA bypass

No debe producir:

- MFA disabled forever
- si policy requiere MFA.

## 98. Re-enrollment requirement

Después del reset:
MFA_ENROLLMENT_REQUIRED

## 99. Identity state

El estado podrá quedar:

- RECOVERY_REQUIRED
- hasta completar nuevo factor.

## 100. Old MFA Factors

Deberán quedar:

- REVOKED
- no simplemente eliminados sin lifecycle/audit.

## 101. Trusted devices

Después de MFA recovery deberán considerarse para revocación.

## 102. Recommended

revoke trusted-device credentials
porque dependían del conjunto MFA anterior.

## 103. Passkey Recovery

Si la Identity pierde todas sus Passkeys:

- Recovery Code
- Federation
- Email Recovery
- Admin Recovery
- Another factor

puede autorizar enrolar una nueva.

## 104. Lost Passkey does not imply credential compromise

Podrá haber:
LOST
vs:

- COMPROMISED
- como reasons diferentes.

## 105. Lost credential response

Podrá revocarse la credential antigua por precaución.

## 106. Synced Passkey consideration

Perder un dispositivo no significa necesariamente que la credential se haya perdido.

## 107. Passkey re-enrollment

Debe utilizar el flujo seguro de documento 16.

## 108. No server-generated private Passkey

Nunca.

## 109. Password Reset

Este será solo un caso particular de credential re-establishment.

## 110. Password reset flow

Recovery authorized
↓
New PasswordCandidate
↓
PasswordPolicy
↓
Compromised Password Check
↓
History Check
↓
Current Hasher
↓
New PasswordCredential
↓
CredentialVersion++
↓
SecurityVersion policy
↓
Security invalidation

## 111. New password entry

Debe ocurrir únicamente después de que RecoveryTransaction tenga assurance suficiente.

## 112. Reset form token

No debe ser la única fuente de state si ya se intercambió por una server-side transaction.

## 113. Password confirmation

El nuevo password deberá introducirse dos veces solo como UX si se desea.
No es requirement criptográfico.

## 114. Old password

No debe requerirse durante true password recovery.

## 115. Password history

Puede impedir reutilizar passwords anteriores si policy lo requiere.

## 116. Password reset invalidation

Por defecto fuerte:

- CredentialVersion++
- SecurityVersion++
- revoke existing sessions
- revoke Remember-Me

revoke relevant persistent credentials

## 117. API Tokens

Existe una decisión de policy.

- Un Password Reset puede:
- KEEP_API_TOKENS
- REVOKE_USER_TOKENS
- REVOKE_ALL_IDENTITY_TOKENS

## 118. Account Takeover Recovery

Si recovery fue provocado por compromiso:

- revoke everything
- será más apropiado.

## 119. RecoveryInvalidationPolicy

Componente formal:

```php
interface RecoveryInvalidationPolicyInterface
{
    public function decide(
        RecoveryCompletionContext $context
    ): RecoveryInvalidationPlan;
}
```

## 120. Invalidation plan

Podrá especificar:

- increment SecurityVersion
- revoke sessions
- revoke remember-me
- revoke trusted devices
- revoke API tokens
- revoke factors
- revoke provider tokens
- rotate recovery codes

## 121. Recovery Profile

Distintos motivos requieren diferentes invalidaciones.

- Ejemplos:
- PASSWORD_FORGOTTEN
- CREDENTIAL_COMPROMISE
- ACCOUNT_TAKEOVER
- MFA_DEVICE_LOST
- PASSKEY_LOST
- ADMIN_RESET

## 122. Credential Re-establishment

Es la fase posterior a demostrar recovery assurance.

## 123. Recovery != authenticated forever

Después de verificar recovery:
RECOVERY_AUTHORIZED
el actor solo tiene autoridad para ejecutar operaciones de re-establishment permitidas.

## 124. Restricted Recovery Context

Puede existir:
RecoveryAuthorizationContext

## 125. RecoveryAuthorizationContext

Podrá contener:

- IdentityReference
- RecoveryTransactionId
- allowed recovery operations
- RecoveryAssurance
- expiresAt

## 126. No normal Authorization

Este Context no equivale a una sesión completa para navegar la aplicación.

## 127. Allowed operations

Ejemplo:

- set new password
- revoke old factors
- enroll new MFA
- register new Passkey

view minimal recovery state
logout/cancel

## 128. Forbidden

No deberá permitir:

- view invoices
- transfer money
- change billing
- manage users

solo porque recovery fue autorizado.

## 129. Recovery completion

Solo cuando requisitos finales se cumplan.

## 130. Recovery completion conditions

Ejemplo:

- new Password established
- AND
- required MFA re-enrolled
- AND
- security invalidation successful

## 131. Incomplete re-establishment

La Identity podrá permanecer:
RECOVERY_REQUIRED

## 132. Recovery Completion Coordinator

Componente:
RecoveryCompletionCoordinator

## 133. Responsibilities

commit new credentials
invalidate old authentication state
update Identity Security State
consume recovery transaction
emit events
audit
trigger notifications

## 134. Commit ordering

Para un recovery crítico:

1. verify recovery still authorized
2. validate SecurityVersion/Identity state
3. persist new credentials
4. mark old credentials unusable
5. increment SecurityVersion
6. revoke sessions/tokens
7. update RecoveryState
8. consume transaction
9. emit audit/events

El ordering exacto dependerá de storage boundaries.

## 10. Fail-safe ordering

Debe evitarse que una falla parcial deje:

```text
old compromised credentials active

+

new credentials active
de forma indefinida.
```

## 136. Atomicity

Cuando sea posible:
database transaction

## 137. Distributed systems

Puede requerir:

- idempotent commands
- outbox
- compensating actions
- version checks

## 138. Recovery commit idempotency

Doble submit del form no debe crear múltiples credentials ni ejecutar invalidación inconsistente.

## 139. RecoveryVersion

La transaction tendrá version/optimistic concurrency.

## 140. Commit-time revalidation

Antes de completar:

- Identity still exists?
- still recoverable?
- SecurityVersion expected?
- transaction active?
- requirements unchanged?

## 141. TOCTOU

Caso:

```text
Recovery begins
    ↓
Admin disables Identity
    ↓
Recovery completes
```

No deberá reactivar la cuenta.

## 142. Another recovery completes first

La segunda transaction deberá quedar obsoleta.

## 143. Recovery Epoch

Una estrategia útil será:

- RecoveryVersion / RecoveryEpoch
- por Identity.

## 144. Example

Transaction A:
recoveryEpoch = 4
Recovery B completes:

```php
recoveryEpoch = 5
A ya no puede completar.
```

## 145. Simplicidad V1

También puede lograrse mediante:

- SecurityVersion
- +;
- transaction version

si cubre correctamente los casos.

## 146. Multiple active recovery transactions

Puede limitarse a:
one active per Identity + purpose

## 147. New request

Podrá:
supersede previous

## 148. Recovery Token replacement

También.

## 149. Parallel channel recovery

En high-security puede requerirse:

- email
- +;
- existing factor

en la misma transaction.

## 150. Evidence independence

Igual que MFA, múltiples recovery proofs no siempre son independientes.

## 151. Example

email link
+
email OTP
pueden depender del mismo mailbox.

## 152. RecoveryIndependencePolicy

Podrá evaluar:

- channel
- provider
- credential source
- device
- administrative authority

## 153. Recovery Assurance Calculator

Contrato:

```php
interface RecoveryAssuranceCalculatorInterface
{
    public function calculate(
        RecoveryEvidenceSet $evidence,
        RecoveryAssuranceContext $context
    ): RecoveryAssurance;
}
```

## 154. Email-only recovery

Puede ser suficiente para aplicaciones estándar, pero no para todos los perfiles.

## 155. High-security recovery

Podrá exigir:

- Passkey OR Recovery Code
- +;
- Administrative Approval
- o equivalente.

## 156. Recovery Escalation

Si la evidencia normal no está disponible:

- escalate
- a un flow más riguroso.

## 157. Recovery levels

Ejemplo:

- SELF_SERVICE
- STRONG_SELF_SERVICE
- ADMIN_ASSISTED
- IDENTITY_PROOFING
- MANUAL_REVIEW

## 158. Escalation must not weaken automatically

No:

```text
Passkey unavailable
    ↓
```

just use easy security question

## 159. Security Questions

VoltStack no deberá recomendarlas como mecanismo base.

## 160. Razón

Información de este tipo suele ser:

- guessable
- discoverable
- reused
- low entropy

## 161. Legacy support

Podrá existir adapter legacy, pero deberá tener assurance bajo y policy explícita.

## 162. SMS Recovery

Podrá soportarse, pero con las mismas limitaciones de seguridad discutidas en MFA.

## 163. Phone number ownership

No equivale universalmente a Identity ownership fuerte.

## 164. SIM Swap Risk

Deberá ser considerado por RecoveryPolicy.

## 165. Phone number change

Un cambio reciente de teléfono puede activar:
recovery cooldown

## 166. Recovery Cooldowns

Después de cambios sensibles puede impedirse temporalmente recovery basado en ese canal.

## 167. Example

email changed 2 minutes ago
No permitir inmediatamente:

- password reset using new email
- para determinadas policies.

## 168. RecoveryChannelAgePolicy

Podrá considerar:

- email age
- phone age
- federated link age
- factor enrollment age
- device trust age

## 169. New recovery channel

Un atacante que logró una sesión temporal no debería poder:

- add email
- then immediately recover account

## 170. Recovery Lock

Después de múltiples intentos sospechosos puede aplicarse:
RecoverySecurityHold

## 171. Avoid attacker-driven permanent denial

Los intentos externos no deben permitir bloquear permanentemente una cuenta con facilidad.

## 172. Rate Limiting

Dimensiones:

- source IP
- identity claim
- Identity
- tenant
- channel
- transaction
- delivery destination

## 173. Progressive defense

Puede usar:

- throttling
- cooldown
- CAPTCHA integration
- additional evidence
- manual review

## 174. CAPTCHA boundary

CAPTCHA puede reducir automatización.
No es Authentication Evidence.

## 175. Bot protection

Será una señal complementaria.

## 176. Recovery Fraud Detection

Risk subsystem podrá evaluar:

- new device
- unusual geography
- known compromised IP
- recent security changes
- multiple failed recoveries

## 177. Risk can raise recovery requirements

Ejemplo:

```text
email recovery normally sufficient
+
high risk
    ↓
additional factor required
```

## 178. Risk cannot prove Identity

Solo altera policy.

## 179. Administrative Recovery

Un administrador autorizado puede asistir en recovery.

## 180. Authentication boundary

Auth define:

- AdministrativeRecoveryCommand
- Authorization decide quién puede ejecutarlo.

## 181. Admin recovery should not set arbitrary password casually

Preferir:

- force recovery transaction
- send secure enrollment link

require user credential re-establishment

## 182. Temporary Password

Si se soporta:

- RESET_REQUIRED
- short-lived

single-use semantics where possible

## 183. Better alternative

temporary recovery credential
de propósito específico.

## 184. Admin cannot see old Password

Nunca.

## 185. Dual Control

En entornos regulados podría exigirse:

- two administrators
- para recovery crítico.

## 186. AdministrativeApprovalRequirement

Podrá integrarse con un futuro approval/workflow subsystem.

## 187. Identity Proofing

Para recovery de alto riesgo puede requerirse:

- KYC
- government ID
- human verification
- enterprise HR process

## 188. Identity Proofing is separate domain

Recovery consume:

- VerifiedIdentityProofingEvidence
- pero no implementa todo KYC.

## 189. Manual Review

Puede producir:

- PENDING_REVIEW
- sin otorgar acceso inmediato.

## 190. Asynchronous review state

La transaction puede persistir hasta resolución dentro de una ventana definida.

## 191. Admin decision

Debe ser:

- auditable
- reasoned
- identity-bound
- transaction-bound

## 192. Recovery notification

Eventos sensibles deberían notificar al usuario.

- Ejemplos:
- recovery requested
- password reset completed
- MFA reset
- Passkey changed

recovery failed due to security concern

## 193. Notification on request

Debe evitar confirmar account existence al atacante en HTTP, pero puede enviar una notificación al propietario real.

## 194. Recovery cancellation

Un usuario autenticado en otro device puede tener opción:
cancel recovery

## 195. Cancel effect

revoke recovery transactions
revoke active recovery tokens

## 196. Security alert

Puede ofrecer:

- This wasn't me
- mediante un flow seguro.

## 197. “Not me” link

También es una security credential/action y deberá tener token/purpose binding propios.

## 198. No mutation from unsigned email URL

Nunca permitir acciones críticas basadas solo en predictable parameters.

## 199. Email address change recovery

Cambiar email primario durante recovery es especialmente sensible.

## 200. Separate capability

No asumir que:
password recovery
permite también:
change recovery email

## 201. Recovery channel changes

Deben requerir assurance adicional.

## 202. Recovery Destination Snapshot

Al iniciar transaction se podrá registrar:
recovery channel reference/version

## 203. Prevent mid-flow channel switch

Si email cambia durante recovery:

- existing transaction
- debe reevaluarse/revocarse según policy.

## 204. Email version

Podrá existir:

- IdentityContactVersion
- o event/version en Identity state.

## 205. Simplicidad

También puede invalidarse mediante:

- SecurityVersion
- cuando el cambio sea security-sensitive.

## 206. Password reset from authenticated session

Debe distinguir:
forgot password recovery
de:
authenticated password change

## 207. Authenticated Password Change

Debe usar documento 11.

## 208. Forgot Password

Debe usar este sistema.

## 209. Password Reset token should not log user in directly

Recommended:

```text
verify recovery token
    ↓
```

grant restricted recovery authorization
↓
set new credential
No:

```text
click email
    ↓
full AuthenticationContext
```

## 210. Recovery Context after password reset

La policy decidirá si:

- AUTO_LOGIN
- REQUIRE_FRESH_LOGIN
- CREATE_RESTRICTED_SESSION

## 211. Recommended security default

Para recovery sensible:

- REQUIRE_FRESH_LOGIN
- o una nueva session explícitamente marcada con recovery provenance.

## 212. UX profile

Aplicaciones de menor riesgo pueden optar por auto-login después de successful reset si todos los controles lo justifican.

## 213. Recovery provenance

Si se crea sesión:

```php
authenticationSource = recovery
deberá conservarse.
```

## 214. Reduced assurance

Una sesión creada inmediatamente tras email-only recovery puede tener assurance menor hasta completar step-up.

## 215. Sensitive operations after recovery

Podrán exigir:

- cooldown
- fresh MFA
- passkey

## 216. Recovery Cooldown after completion

Ejemplo:

```text
password reset complete
    ↓
```

bank payout changes blocked for 24h
Esto pertenece parcialmente a Authorization/business risk, pero Auth puede exponer:

- recent_recovery_at
- como provenance.

## 217. Recovery timestamps

AuthenticationContext podrá conocer:

- recoveredAt
- cuando relevante.

## 218. Authorization boundary

Authorization puede utilizar:

- recent recovery
- reduced assurance
- para policy.

Auth no decide business operations.

## 219. Credential Re-establishment Plan

Podrá existir:
CredentialReestablishmentPlan

## 220. Example plan

1. revoke password
2. set new password
3. revoke TOTP
4. enroll Passkey
5. regenerate recovery codes
6. revoke sessions
7. Plan Resolver

CredentialReestablishmentPlanner
decidirá según:

- RecoveryPurpose
- Identity type
- Current credentials
- Security policy
- Recovery assurance

## 8. Account Takeover Recovery

Un perfil fuerte podría hacer:

- revoke all passwords
- revoke all Passkeys
- revoke MFA
- revoke sessions
- revoke remember-me
- revoke PATs
- revoke trusted devices
- require new password
- require new Passkey/MFA
- regenerate recovery codes

## 9. Do not always revoke everything

Para un simple forgot-password estándar, revocar unrelated Service Tokens puede ser innecesario.

## 10. Policy-driven invalidation

Clave del diseño.

## 11. CredentialDependencies

El sistema podrá conocer relaciones:

- TrustedDevice issued from MFA
- RememberMe issued from Password

Session derived from Passkey
para invalidación selectiva avanzada.

## 12. V1 simpler model

critical recovery
→ SecurityVersion++
es una alternativa más simple y segura.

## 13. Password-only Identity

Simple flow:

```text
email link
    ↓
new password
    ↓
SecurityVersion++
    ↓
sessions revoked
```

## 14. Password + MFA Identity

email recovery
↓
new password
↓
MFA still active
Puede o no ser suficiente para login dependiendo de qué se perdió.

## 15. Lost MFA case

Debe distinguirse para no resetear Password innecesariamente.

## 16. Lost Everything case

Requiere:
stronger recovery path

## 17. Passkey-only account

Puede utilizar:

- recovery code
- federated identity
- admin assisted
- secondary Passkey

## 18. Federation-only enterprise account

Normalmente el recovery ocurre en:

- external IdP
- no dentro de VoltStack.

## 19. Local response

VoltStack puede indicar:
Use your organization's identity provider to recover access.

## 20. No local fallback unless configured

Muy importante.

## 21. Service Identity Recovery

Las service identities normalmente no usan email reset.

## 22. Service Credential Recovery

Podrá significar:

- rotate API secret
- issue replacement token
- revoke certificate
- provision workload credential

## 23. Separate recovery profile

MACHINE_CREDENTIAL_RECOVERY

## 24. Human vs Machine recovery

Comparten primitives:

- transaction
- evidence
- reestablishment
- invalidation

pero no necesariamente delivery mechanisms.

## 25. Recovery API

Conceptualmente:

```php
Auth::recovery()->begin(...);
Auth::recovery()->verify(...);
Auth::recovery()->complete(...);
```

## 26. Password reset ergonomics

Puede ofrecerse:

```php
Password::sendResetLink($claim);
como façade de alto nivel.
```

Pero internamente deberá delegar al Recovery subsystem.

## 27. Legacy Laravel-style API compatibility

Podrá ofrecer ergonomía similar a:

- Password Broker
- Reset Password

sin limitar la arquitectura general a passwords.

## 28. Recovery Broker

Podría existir:

- RecoveryBroker
- como API application-facing.

## 29. Broker responsibilities

start common recovery
send recovery challenge
complete password reset

## 30. Recovery Manager remains deeper core

RecoveryBroker
UX/application convenience

RecoveryManager
domain orchestration

## 245. RecoveryChannel

Tipos:

- EMAIL
- SMS
- PUSH
- FEDERATED
- ADMINISTRATIVE
- RECOVERY_CODE
- EXISTING_FACTOR

## 246. RecoveryChannelProvider

SPI:

```php
interface RecoveryChannelProviderInterface
{
    public function initiate(
        RecoveryChallengeRequest $request
    ): RecoveryChallengeResult;
}
```

## 247. Email provider

No deberá conocer Password reset semantics.
Solo delivery.

## 248. RecoveryChallenge

Conceptualmente:

- RecoveryChallenge
- ligado a transaction y method.

## 249. Challenge types

TOKEN_LINK
OTP
FACTOR_VERIFICATION
FEDERATED_LOGIN
ADMIN_APPROVAL

## 250. Challenge expiry

Siempre explícita.

## 251. Challenge retries

Bounded.

## 252. Recovery OTP

Si se utiliza:

- short random code
- digest stored
- attempt limit
- expiration

## 253. Short numeric OTP risk

Debe tener:

- strict throttling
- short lifetime
- limited attempts

## 254. Recovery token entropy

Los links pueden usar tokens de mayor entropía y por tanto tienen mejores brute-force properties.

## 255. Recovery delivery binding

El challenge debe quedar ligado al destination que existía al emitirlo.

## 256. No destination disclosure

La UI puede enmascarar:

```php
f***@example.com
solo después de enough context, y evitando enumeration.
```

## 257. Recovery method selection

Puede ocurrir después de una primera proof.

## 258. Multiple available methods

Passkey
Recovery Code
Email
Corporate SSO

## 259. Preferred ordering

Security policy podrá favorecer:

- Passkey
- Federation
- Recovery Code
- Email
- SMS
- según threat model.

## 260. User choice cannot bypass requirement

Si policy exige:

- strong recovery
- no podrá seleccionar email-only si no alcanza assurance.

## 261. Recovery Method Descriptor

Podrá exponer:

- public label
- method type
- assurance properties
- availability
- sin sensitive metadata.

## 262. Recovery Availability

AVAILABLE
UNAVAILABLE
REQUIRES_ADMIN
TEMPORARILY_BLOCKED

## 263. Recovery attempt monitoring

Podrá detectar:

- many accounts from one source
- many attempts against one account
- token guessing
- OTP brute force

## 264. Unknown token brute force

Selector-based lookup ayuda a rechazar rápidamente, pero debe existir rate limiting.

## 265. Recovery token database leak

Guardar solo digest reduce riesgo.

## 266. Email compromise

Un atacante con mailbox puede usar recovery email.
La policy debe reconocer esa realidad.

## 267. Recovery channel compromise signal

Si email fue cambiado/comprometido:

- disable email recovery
- temporalmente.

## 268. IdentityCompromise integration

COMPROMISED puede exigir:
non-email recovery

## 269. Recovery reason

Internamente se deberá registrar:

- FORGOTTEN_CREDENTIAL
- LOST_FACTOR
- COMPROMISED_FACTOR
- ACCOUNT_TAKEOVER
- ADMIN_FORCED
- USER_REQUESTED

## 270. Reason is not client authority

Un usuario puede declarar:

- I lost my device
- pero eso no cambia security policy por sí mismo.

## 271. Recovery Audit

Deberá registrar eventos como:

- RecoveryRequested
- RecoveryTransactionCreated
- RecoveryChallengeIssued
- RecoveryEvidenceVerified
- RecoveryEvidenceRejected
- RecoveryAuthorized
- RecoveryDenied
- PasswordResetCompleted
- MfaRecoveryCompleted
- PasskeyRecoveryCompleted
- AdministrativeRecoveryApproved
- RecoveryCompleted
- RecoveryCancelled
- RecoveryReplayDetected

## 272. Audit privacy

No almacenar:

- raw recovery tokens
- OTP
- password
- recovery codes

## 273. Audit actor

Podrá identificar:

- self-service
- administrator
- system
- security automation

## 274. Observability spans

auth.recovery.begin
auth.recovery.resolve_identity
auth.recovery.issue_challenge
auth.recovery.verify_evidence
auth.recovery.assurance
auth.recovery.reestablish
auth.recovery.invalidate
auth.recovery.complete

## 275. Metrics

Ejemplos:

- auth_recovery_requested_total
- auth_recovery_completed_total
- auth_recovery_failed_total
- auth_recovery_token_invalid_total
- auth_recovery_token_replay_total
- auth_recovery_delivery_total
- auth_recovery_evidence_total
- auth_recovery_admin_escalation_total
- auth_recovery_latency

## 276. Safe labels

purpose
recovery_method
result
firewall
failure_category

## 277. Avoid labels

email
phone
identity id
token selector
transaction id

## 278. Alerting

Eventos relevantes:

- many recovery requests for one account
- recovery from unusual source
- multiple token replays
- admin recovery spike

recovery immediately after contact change

## 279. Recovery Notifications

El sistema deberá permitir notificaciones de:

- recovery started
- credential changed
- MFA reset

Passkey added after recovery
all sessions revoked

## 280. Completion notification

Debe enviarse por canales independientes cuando sea apropiado.

## 281. Example

Si Password se reseteó usando email:

- notify existing trusted devices
- si existen.

## 282. Alternate channel notification

Muy útil cuando un atacante controla el recovery channel principal.

## 283. Contact change notification

Cambios de email/phone de recuperación deberían notificar al canal anterior cuando sea posible.

## 284. Cancellation link

Podrá incluirse bajo token/purpose propios.

## 285. Recovery token parser

Debe aplicar:

- maximum length
- strict encoding
- format validation
- version validation

## 286. Token version

Ejemplo:
vst_rec1_selector.secret

## 287. Cryptographic agility

Permitirá cambiar digest/keying strategy.

## 288. Keyed digest

Puede utilizar:

```php
HMAC(server secret, token secret)
como defense in depth.
```

## 289. Key rotation

Necesitará:

- key id
- previous verification keys

si se adopta esa estrategia.

## 290. Recovery token and CSRF

Una vez que token se intercambió por una recovery transaction protegida por cookie/session, las operaciones state-changing deberán integrarse con CSRF cuando corresponda.

## 291. Token itself as anti-CSRF?

No deberá asumirse automáticamente.

## 292. Browser session after recovery token exchange

Puede crearse una:

- RecoverySession
- independiente de AuthenticationSession normal.

## 293. RecoverySession

Contiene:

- transaction reference
- anti-CSRF state
- expiration

No Identity full authenticated context.

## 294. Recovery cookie

Debe ser:

- HttpOnly
- Secure
- SameSite appropriate
- short-lived

## 295. Separate cookie name

No reutilizar necesariamente la Authentication Session cookie antes de recovery completion.

## 296. Session fixation protection

Si finalmente se crea AuthenticationSession:
rotate/regenerate session ID

## 297. Authentication after recovery

Si policy permite auto-login:

- new AuthenticationSession
- debe ser completamente nueva.

## 298. No reuse of recovery token as session token

Nunca.

## 299. Recovery provenance in new session

Debe registrar:

- authentication source = recovery
- recovery assurance
- recoveredAt
- cuando sea relevante.

## 300. Recovery Session expiration

No debe sobrevivir horas/días más allá de lo necesario.

## 301. Persistent credentials after recovery

Remember-Me anterior normalmente se revoca en recovery sensible.

## 302. New Remember-Me

No deberá emitirse automáticamente desde un recovery de assurance bajo.

## 303. Trusted device issuance

Igualmente.

## 304. New trusted state

Debe requerir Authentication normal/strong según policy.

## 305. Recovery and MFA bypass

Un recovery email no debería generar automáticamente:

- trusted device
- MFA remembered

## 306. Factor re-enrollment

Después de recovery crítico:

- old factor set revoked
- new enrollment required

## 307. Recovery codes regeneration

Los anteriores deben invalidarse.

## 308. Existing Passkeys

Depende del incident.
Forgot password:
may keep Passkeys
Account takeover:
may revoke all

## 309. Federation mappings

Igual:

```text
normal reset
    may keep

suspected account takeover
    review/revoke suspicious mappings
```

## 310. Recovery system must not over-delete identity data

Credential invalidation es distinto de borrar profile/application data.

## 311. Recovery Cancellation

Una transaction puede cancelarse por:

- user action
- security signal
- new recovery request
- admin action
- identity state change
- expiry

## 312. Cancellation invalidates challenges

Obligatorio.

## 313. Cancellation is idempotent

Sí.

## 314. Recovery resume

Puede continuar mientras:

- transaction valid
- base assumptions unchanged

## 315. Long-running manual recovery

Puede requerir una transaction con TTL mayor y approval states.

## 316. Manual recovery token separation

El approval de un administrador no deberá ser el mismo secret que usa el usuario final.

## 317. Dual identities

Administrative Recovery deberá preservar:

- actor administrator
- subject recovered identity

## 318. Actor/Subject audit

Importantísimo.

## 319. Impersonation prohibited

Recovery admin no deberá convertirse automáticamente en la Identity recuperada.

## 320. Administrative setup links

Puede generar un recovery credential destinado al usuario.

## 321. Recovery policy hierarchy

Puede componerse:

- Framework Security Floor
- Application Recovery Policy
- Firewall Policy
- Tenant Policy
- Identity Type Policy
- Risk Policy
- Incident Policy

## 322. More restrictive wins

Para requirements críticos:

- Tenant requires strong recovery
- Application allows email recovery

Effective:
strong recovery

## 323. Incident override

Si Identity está:

- COMPROMISED
- una incident policy podrá elevar requirements.

## 324. No tenant weakening below security floor

Nunca.

## 325. RecoveryPolicyResolver

Componente:

```php
interface RecoveryPolicyResolverInterface
{
    public function resolve(
        RecoveryPolicyContext $context
    ): EffectiveRecoveryPolicy;
}
```

## 326. EffectiveRecoveryPolicy

Podrá incluir:

- allowed methods
- requirements
- minimum assurance
- transaction lifetime
- delivery limits
- recovery cooldown
- completion actions
- invalidation plan
- auto-login policy

## 327. Recovery policy version

La transaction podrá registrar:
policyVersion

## 328. Policy change mid-recovery

Antes de commit deberá reevaluarse.

## 329. Stronger new policy

Puede exigir evidence adicional.

## 330. Weaker new policy

No necesariamente debe debilitar una transaction ya iniciada.

## 331. Security monotonicity

Preferible que una recovery transaction no se vuelva más permisiva durante su lifetime por cambios dinámicos.

## 332. Recovery state compilation

Policies estáticas podrán compilarse para performance.

## 333. Recovery secret memory safety

Raw token/OTP/password nuevos deberán tener lifetime mínimo en memoria.

## 334. PHP limitation

No prometer zeroization física garantizada.

## 335. SensitiveParameter

Usar mecanismos del lenguaje donde sean útiles para redacción de stack traces.

## 336. Exceptions

Ejemplos internos:

- RecoveryTransactionExpiredException
- RecoveryTokenInvalidException
- RecoveryEvidenceRejectedException
- RecoveryPolicyUnsatisfiedException
- RecoveryCommitException
- RecoveryChannelUnavailableException

## 337. Public errors

Deben ser más genéricos.

## 338. Recovery channel unavailable

No deberá convertirse en:
Recovery Authorized

## 339. Fail closed

Regla general.

## 340. Delivery failure

Si email no pudo enviarse:

- transaction may exist
- pero no debe considerarse evidence.

## 341. Retry delivery

Puede reutilizar transaction con nuevo challenge/token según policy.

## 342. Provider outage

Email/Federation/SMS provider outage podrá ofrecer otro método solo si ese método ya es permitido.

## 343. No emergency downgrade

Nunca bajar requirements automáticamente por disponibilidad.

## 344. Recovery testing — identity enumeration

Casos:

- known email
- unknown email
- disabled account
- federated-only account

Las responses externas deberán mantener semántica segura.

## 345. Testing — tokens

valid
wrong secret
unknown selector
expired
consumed
revoked
superseded
malformed
oversized

## 346. Testing — concurrency

Especialmente:

- same token used twice
- two recoveries complete concurrently

new recovery supersedes old
admin disables identity during recovery
SecurityVersion changes before commit

## 347. Testing — password reset

valid recovery
new password accepted
password policy rejection
compromised password rejection
history reuse
storage failure
invalidation failure

## 348. Testing — MFA recovery

lost TOTP
recovery code
factor reset
trusted device revocation
new MFA enrollment

## 349. Testing — Passkey recovery

lost Passkey
existing second Passkey
federated recovery
new Passkey registration
old credential revoked

## 350. Testing — Federated recovery

trusted issuer
subject mapping
provider unavailable
wrong subject
tenant mismatch

## 351. Testing — Administrative recovery

authorized admin
unauthorized admin
dual approval
reason required
subject binding
audit

## 352. Testing — Recovery assurance

email only
recovery code
passkey
federated identity
email + same-email OTP
independent multiple evidence

## 353. Testing — Cooldowns

email recently changed
phone recently changed
new federated link
new recovery method

## 354. Testing — Session invalidation

existing session
remember-me
trusted device
API token
según profile.

## 355. Testing — No auto-login

Verify profile requiring fresh login.

## 356. Testing — Auto-login profile

Verify:

- new session ID
- recovery provenance

reduced assurance if configured

## 357. Testing — Persistent Runtime

Request A:
Alice recovery
Request B:
Bob recovery
Request C:

- anonymous
- No state leakage.

## 358. Fuzz testing

Especialmente:

- recovery token parser
- transaction identifiers
- callback parameters
- OTP inputs
- recovery state serialization

## 359. Property-based testing

Útil para:

- transaction state machine
- token single-use
- evidence composition
- recovery assurance monotonicity
- invalidation plan composition

## 360. Recovery state machine

PENDING
│
▼
CHALLENGED
│
▼
EVIDENCE_PARTIAL
│
├────────────► DENIED
│
├────────────► EXPIRED
│
▼
AUTHORIZED
│
▼
COMMITTING
│
├────────────► ERROR/RECOVERABLE FAILURE
│
▼
COMPLETED

## 361. Credential re-establishment state

Recovery Authorized
↓
Credentials Pending
↓
Required Credentials Established
↓
Invalidation Complete
↓
Identity Security State Restored

## 362. Security invariant — Recovery Transaction

AUTH-REC-01
Every recovery flow has an explicit server-side transaction.

- AUTH-REC-02
- Recovery transactions are purpose-bound.
- AUTH-REC-03

Recovery transactions are short-lived unless an explicit manual-review profile applies.
AUTH-REC-04
Completed/expired/cancelled transactions cannot be reused.

- AUTH-REC-05
- Recovery state is never trusted from client serialization.
- AUTH-REC-06

Concurrent recovery completion requires consistency control.
AUTH-REC-07
Commit-time Identity Security State is revalidated.

## 363. Security invariant — Recovery Tokens

AUTH-REC-TOKEN-01
Recovery tokens are cryptographically random.

- AUTH-REC-TOKEN-02
- Raw recovery tokens are not stored server-side.
- AUTH-REC-TOKEN-03

Recovery tokens are single-use.

- AUTH-REC-TOKEN-04
- Recovery tokens expire.
- AUTH-REC-TOKEN-05

Recovery tokens are purpose/transaction bound.

- AUTH-REC-TOKEN-06
- Consumed tokens cannot authenticate again.
- AUTH-REC-TOKEN-07

Token parsing is strictly bounded.
AUTH-REC-TOKEN-08
Raw tokens never enter logs/events/metrics.

## 364. Security invariant — Identity

AUTH-REC-ID-01
Recovery does not reactivate a disabled Identity automatically.
AUTH-REC-ID-02
Deleted Identities cannot be recovered unless a separate restoration system explicitly permits it.
AUTH-REC-ID-03
Recovery eligibility is controlled by Identity Security State.
AUTH-REC-ID-04
Federated recovery still maps to a local Identity.
AUTH-REC-ID-05
Email equality does not prove Identity ownership universally.

## 365. Security invariant — Re-establishment

AUTH-REC-REEST-01
Recovery authorization grants only bounded credential re-establishment authority.
AUTH-REC-REEST-02
Recovery authorization is not equivalent to unrestricted AuthenticationContext.
AUTH-REC-REEST-03
New credentials pass their normal creation policies.
AUTH-REC-REEST-04
Old credentials are invalidated according to explicit RecoveryInvalidationPolicy.
AUTH-REC-REEST-05
Critical recovery changes SecurityVersion according to policy.
AUTH-REC-REEST-06
MFA recovery can require re-enrollment before normal Authentication resumes.

## 366. Security invariant — Assurance

AUTH-REC-ASSURANCE-01
Recovery assurance is derived from verified evidence.

- AUTH-REC-ASSURANCE-02
- Multiple proofs are not automatically independent.
- AUTH-REC-ASSURANCE-03

Risk can raise recovery requirements.

- AUTH-REC-ASSURANCE-04
- Provider outages cannot silently weaken recovery requirements.
- AUTH-REC-ASSURANCE-05

Legacy weak methods cannot claim high assurance by configuration alone.

## 367. Security invariant — Enumeration

AUTH-REC-ENUM-01
Recovery initiation does not reveal account existence unnecessarily.
AUTH-REC-ENUM-02
Available factor/channel disclosure is appropriately gated.

- AUTH-REC-ENUM-03
- Rate-limit behavior should not create trivial account-enumeration side channels.
- AUTH-REC-ENUM-04

Internal diagnostics remain richer than external responses.

## 368. Security invariant — Runtime

AUTH-REC-RT-01
Current recovery state is transaction/request scoped.

- AUTH-REC-RT-02
- Shared Recovery services remain stateless.
- AUTH-REC-RT-03

No current Identity/recovery token survives FrankenPHP request boundaries.
AUTH-REC-RT-04
Concurrent fibers have isolated Recovery contexts.
AUTH-REC-RT-05
Distributed workers share authoritative transaction state.

## 369. Anti-pattern — Reset Password as isolated controller

Evitar:

```php
if ($tokenIsValid) {
    $user->password = Hash::make($request->password);
}
```

sin transaction, lifecycle, invalidation y Security State.

## 370. Anti-pattern — Password reset token stored plaintext

Nunca.

## 371. Anti-pattern — reusable recovery link

Nunca como default.

## 372. Anti-pattern — account exists response

Evitar:

- No user with that email.
- en public recovery initiation.

## 373. Anti-pattern — email is account ownership proof

No universalmente.

## 374. Anti-pattern — admin types temporary password into database

No.

## 375. Anti-pattern — reset password leaves compromised sessions active

No para compromise recovery.

## 376. Anti-pattern — recovery automatically disables MFA permanently

No.

## 377. Anti-pattern — Recovery Code becomes normal password

No.

## 378. Anti-pattern — provider outage downgrades to email

No automáticamente.

## 379. Anti-pattern — recovery token directly equals authenticated session

No.

## 380. Anti-pattern — recovery link generates unrestricted session immediately

Debe pasar por RecoveryPolicy/Re-establishment.

## 381. Anti-pattern — security questions default

No recomendado.

## 382. Anti-pattern — new email immediately becomes recovery authority

Debe considerar cooldown/policy.

## 383. Anti-pattern — reset only password after account takeover

Puede ser insuficiente si:

- sessions
- Passkeys
- MFA
- tokens
- trusted devices
- también fueron comprometidos.

## 384. Anti-pattern — process-global RecoveryTransaction

Crítico bajo FrankenPHP.

## 385. Componentes principales

RecoveryManager
RecoveryBroker

RecoveryTransaction
RecoveryTransactionId
RecoveryTransactionStatus
RecoveryPurpose
RecoveryTransactionRepository

RecoveryPolicy
RecoveryPolicyResolver
EffectiveRecoveryPolicy

RecoveryRequirement
RecoveryRequirementSet
RecoveryEvidence
RecoveryEvidenceSet
RecoveryAssurance
RecoveryAssuranceCalculator

## 386. Componentes de challenges

RecoveryChallenge
RecoveryChallengeId
RecoveryChallengeManager
RecoveryChannel
RecoveryChannelProvider
RecoveryAttemptTracker

## 387. Componentes token/email

RecoveryToken
RecoveryTokenRecord
RecoveryTokenGenerator
RecoveryTokenRepository
RecoveryTokenVerifier
RecoveryUrlGenerator
EmailRecoveryChannel

## 388. Componentes re-establishment

CredentialReestablishmentPlanner
CredentialReestablishmentPlan
RecoveryCompletionCoordinator
RecoveryInvalidationPolicy
RecoveryInvalidationPlan

## 389. Componentes especializados

PasswordRecoveryHandler
MfaRecoveryHandler
PasskeyRecoveryHandler
FederatedRecoveryHandler
AdministrativeRecoveryHandler
RecoveryCodeHandler

## 390. Namespace sugerido

VoltStack\Quantum\Auth\Recovery
VoltStack\Quantum\Auth\Recovery\Contracts
VoltStack\Quantum\Auth\Recovery\Transaction
VoltStack\Quantum\Auth\Recovery\Policy
VoltStack\Quantum\Auth\Recovery\Evidence
VoltStack\Quantum\Auth\Recovery\Challenge
VoltStack\Quantum\Auth\Recovery\Token
VoltStack\Quantum\Auth\Recovery\Channel
VoltStack\Quantum\Auth\Recovery\Reestablishment
VoltStack\Quantum\Auth\Recovery\Administrative

## 391. Estructura sugerida

src/Quantum/Auth/Recovery/
├── Contracts/
│   ├── RecoveryManagerInterface.php
│   ├── RecoveryTransactionRepositoryInterface.php
│   ├── RecoveryPolicyResolverInterface.php
│   ├── RecoveryChannelProviderInterface.php
│   └── RecoveryTokenRepositoryInterface.php
│
├── Transaction/
│   ├── RecoveryTransaction.php
│   ├── RecoveryTransactionId.php
│   ├── RecoveryTransactionStatus.php
│   ├── RecoveryPurpose.php
│   └── RecoveryTransactionRepository.php
│
├── Policy/
│   ├── RecoveryPolicy.php
│   ├── EffectiveRecoveryPolicy.php
│   ├── RecoveryPolicyResolver.php
│   ├── RecoveryRequirement.php
│   └── RecoveryRequirementSet.php
│
├── Evidence/
│   ├── RecoveryEvidence.php
│   ├── RecoveryEvidenceSet.php
│   ├── RecoveryAssurance.php
│   └── RecoveryAssuranceCalculator.php
│
├── Challenge/
│   ├── RecoveryChallenge.php
│   ├── RecoveryChallengeId.php
│   ├── RecoveryChallengeManager.php
│   └── RecoveryAttemptTracker.php
│
├── Token/
│   ├── RecoveryToken.php
│   ├── RecoveryTokenRecord.php
│   ├── RecoveryTokenGenerator.php
│   ├── RecoveryTokenVerifier.php
│   └── RecoveryTokenRepository.php
│
├── Channel/
│   ├── EmailRecoveryChannel.php
│   ├── RecoveryCodeChannel.php
│   ├── FederatedRecoveryChannel.php
│   └── ExistingFactorRecoveryChannel.php
│
├── Reestablishment/
│   ├── CredentialReestablishmentPlanner.php
│   ├── CredentialReestablishmentPlan.php
│   ├── RecoveryCompletionCoordinator.php
│   ├── RecoveryInvalidationPolicy.php
│   └── RecoveryInvalidationPlan.php
│
├── Administrative/
│   ├── AdministrativeRecoveryService.php
│   ├── AdministrativeRecoveryCommand.php
│   └── AdministrativeApprovalRequirement.php
│
├── RecoveryManager.php
└── RecoveryBroker.php

## 392. Configuración conceptual

return [

'authentication' => [

'recovery' => [

'default' => 'standard',

'profiles' => [

'standard' => [
'methods' => [
'email',
'recovery_code',
'passkey',
],

'minimum_assurance' => 'standard',

'transaction_lifetime' => '30 minutes',

'after_completion' => [
'revoke_sessions' => true,
'revoke_remember_me' => true,
'require_fresh_login' => true,
],
],

'high_security' => [
'methods' => [
'passkey',
'recovery_code',
'federated',
'administrative',
],

'minimum_assurance' => 'strong',

'email_only' => false,

'after_completion' => [
'increment_security_version' => true,
'revoke_sessions' => true,
'revoke_persistent_credentials' => true,
'revoke_tokens' => true,
],
],

],

],

],

];
Los valores son ilustrativos.

## 393. Configuración por Identity Type

'recovery_profiles' => [

'human' => 'standard',

'administrator' => 'high_security',

'service' => 'machine_recovery',

];

## 394. Configuración tenant

'tenant_recovery' => [

'allow_email' => true,

'require_mfa_recovery_for_admins' => true,

'administrative_approval' => false,

];

## 395. Flujo completo Forgot Password

User
↓
POST /forgot-password
↓
IdentityClaim
↓
RecoveryManager
↓
Identity resolution
↓
EffectiveRecoveryPolicy
↓
Create RecoveryTransaction
↓
Generate RecoveryToken
↓
Store token digest
↓
Send email
↓
Generic HTTP response

User opens link
↓
RecoveryTokenVerifier
↓
Token consumed/exchanged
↓
Recovery Evidence
↓
Recovery Assurance sufficient
↓
RecoveryAuthorizationContext
↓
New PasswordCandidate
↓
Password Policy
↓
New PasswordCredential
↓
CredentialVersion++
↓
SecurityVersion++
↓
sessions revoked
↓
remember-me revoked
↓
Recovery completed
↓
fresh login required

## 396. Flujo MFA perdido

User proves Password
↓
MFA unavailable
↓
Begin MFA Recovery
↓
Recovery Code / Passkey / Federation
↓
Recovery Evidence
↓
Recovery Assurance
↓
MFA Reset Authorized
↓
old MFA factors REVOKED
↓
trusted devices revoked
↓
MFA_ENROLLMENT_REQUIRED
↓
new factor enrollment
↓
Recovery completed

## 397. Flujo Account Takeover

Account Takeover Suspected
↓
Identity = COMPROMISED
↓
Strong Recovery Required
↓
multiple evidence / admin review
↓
Recovery Authorized
↓
revoke Password
revoke Passkeys
revoke MFA
revoke sessions
revoke Remember-Me
revoke trusted devices
revoke selected API tokens
↓
SecurityVersion++
↓
new credentials established
↓
notifications
↓
Identity returns ACTIVE

## 398. Flujo Federation-only

Identity uses Corporate SSO only
↓
User selects recovery
↓
VoltStack detects federation-only profile
↓
Redirect to Corporate IdP recovery/login
↓
OIDC Authentication
↓
issuer + subject verified
↓
local Identity matched
↓
no local Password created
↓
Authentication restored

## 399. Flujo Passkey-only

Passkey-only Identity
↓
Passkey unavailable
↓
Recovery Code
OR
Federated Recovery
OR
Admin-assisted Recovery
↓
Recovery authorized
↓
Register new Passkey
↓
old lost Passkey revoked
↓
Security state updated

## 400. Flujo concurrent recovery

Transaction A
expected SecurityVersion = 10

Transaction B
completes
SecurityVersion = 11

Transaction A
attempts completion
↓
version mismatch
↓
STALE_RECOVERY_TRANSACTION
↓
reject

## 401. Flujo recovery token replay

Token A
↓
successful exchange
↓
CONSUMED

Later:

```text
Token A reused
   ↓
REPLAY_DETECTED
   ↓
reject
   ↓
security telemetry
```

## 402. Flujo email change cooldown

Attacker gains temporary session
↓
changes recovery email
↓
immediately requests password reset
↓
RecoveryChannelAgePolicy
↓
new email too recent
↓
stronger recovery required / blocked

## 403. Arquitectura global

ACCOUNT RECOVERY
│
▼
RecoveryManager
│
▼
Identity Claim
│
▼
Identity / Security State
│
▼
RecoveryPolicyResolver
│
▼
RecoveryTransaction
│
┌─────────────────────┼─────────────────────┐
▼                     ▼                     ▼
EMAIL                 FACTOR               FEDERATION
│                     │                     │
▼                     ▼                     ▼
Recovery Token        Passkey / Code / TOTP      OIDC Proof
│                     │                     │
└─────────────────────┼─────────────────────┘
▼
RecoveryEvidence
│
▼
RecoveryAssuranceCalculator
│
┌───────────┴────────────┐
▼                        ▼
INSUFFICIENT              AUTHORIZED
│                        │
▼                        ▼
MORE CHALLENGES      RecoveryAuthorizationContext
│
▼
CredentialReestablishmentPlan
│
┌───────────────────────────────┼────────────────────┐
▼                               ▼                    ▼
Password                         MFA                 Passkeys
│                               │                    │
└───────────────────────────────┼────────────────────┘
▼
Security Invalidation
│
▼
Recovery Completion
│
▼
Fresh Authentication

## 404. Decisiones arquitectónicas principales

VoltStack adoptará:

1. Recovery is an Authentication subsystem, not an exception path.
2. Recovery is broader than Password Reset.
3. Every recovery flow is transaction-based.
4. Recovery evidence has explicit assurance semantics.
5. Recovery authorization is bounded and not equivalent to a full session.
6. Recovery tokens are high-entropy, hashed, expiring and single-use.
7. Account existence is protected against enumeration.
8. Credential re-establishment and security invalidation are coordinated.
9. Identity Security State remains authoritative during recovery.
10. Strong Authentication cannot be undermined by an implicitly weak recovery path.
11. Recovery policy is profile-, tenant-, risk- and incident-aware.
12. Recovery is safe under distributed and persistent runtimes.
13. Comparación conceptual con Laravel y Symfony

Laravel proporciona una experiencia excelente para casos estándar mediante conceptos como:

- Password Broker
- Password Reset Tokens
- ResetPassword notification
- password reset callbacks

Symfony permite construir flujos equivalentes mediante:

- Security
- User providers
- reset-password bundles/ecosystem
- custom authenticators

VoltStack conservará la ergonomía de:

- send reset link
- verify reset request
- set new password

pero ampliará el dominio a:

- RecoveryTransaction
- RecoveryEvidence
- RecoveryAssurance
- RecoveryPolicy
- CredentialReestablishmentPlan
- RecoveryInvalidationPolicy
- MFA Recovery
- Passkey Recovery
- Federated Recovery
- Administrative Recovery

La diferencia fundamental será:

```text
Traditional approach:
    Forgot Password
        ↓
    Reset Password
```

VoltStack:

```text
    Lost Authentication Control
        ↓
    Verify Recovery Evidence
        ↓
    Establish Recovery Assurance
        ↓
    Re-establish Credentials
        ↓
    Invalidate Compromised Trust
        ↓
    Restore Authentication Eligibility
```

## 406. Criterios de aceptación

El subsistema será considerado completo cuando:

1. soporte Forgot Password;
2. soporte Password Reset;
3. soporte general Account Recovery;
4. soporte RecoveryTransaction;
5. soporte transaction TTL;
6. soporte transaction single-use;
7. soporte RecoveryPurpose;
8. soporte RecoveryPolicy;
9. soporte Recovery Requirements;
10. soporte Recovery Evidence;
11. soporte Recovery Assurance;
12. soporte Email Recovery;
13. soporte secure recovery tokens;
14. almacene solo digest de tokens;
15. soporte token expiration;
16. soporte token single-use;
17. soporte atomic consume;
18. soporte recovery email rate limiting;
19. reduzca account enumeration;
20. soporte Recovery Codes;
21. soporte existing-factor recovery;
22. soporte Passkey recovery;
23. soporte MFA recovery;
24. soporte Federated recovery;
25. soporte Administrative recovery;
26. permita identity proofing integration;
27. soporte recovery escalation;
28. soporte recovery cooldown;
29. soporte channel-age policies;
30. soporte credential re-establishment;
31. soporte PasswordPolicy integration;
32. soporte MFA re-enrollment;
33. soporte Passkey re-enrollment;
34. soporte SecurityVersion;
35. soporte CredentialVersion;
36. soporte session invalidation;
37. soporte Remember-Me invalidation;
38. soporte Trusted Device invalidation;
39. soporte API-token invalidation policy;
40. soporte commit-time revalidation;
41. soporte concurrency;
42. soporte transaction supersession;
43. soporte audit;
44. soporte notifications;
45. soporte observability;
46. sea seguro con FrankenPHP;
47. sea fiber-safe;
48. no utilice Recovery como Authorization bypass.
49. Regla arquitectónica final

VoltStack deberá preservar:

```text
LOSS OF NORMAL AUTHENTICATION CONTROL
             ↓
       RECOVERY REQUEST
             ↓
     RECOVERY TRANSACTION
             ↓
    VERIFIED RECOVERY EVIDENCE
             ↓
      RECOVERY ASSURANCE
             ↓
      RECOVERY AUTHORIZATION
             ↓
 CREDENTIAL RE-ESTABLISHMENT
             ↓
 OLD TRUST INVALIDATION
             ↓
```

IDENTITY SECURITY STATE UPDATE
↓
RECOVERY COMPLETION
↓
FRESH AUTHENTICATION
La primera regla central será:
Recovery no debe demostrar menos control sobre una Identity que el nivel mínimo de confianza que la aplicación considera aceptable para volver a emitir credenciales.

La segunda:
Un recovery token no será una sesión ni una prueba ilimitada de identidad; será una credential temporal, de propósito específico y de un solo uso que contribuye a una RecoveryTransaction.

La tercera:
Completar Recovery no consiste únicamente en crear una nueva contraseña. También implica decidir qué sesiones, factores, Passkeys, Remember-Me credentials, trusted devices y tokens existentes continúan siendo confiables.

Y la regla de seguridad que gobernará todo el subsistema será:
VoltStack deberá asumir que cualquier credential que permitió o precedió a una recuperación por compromiso puede haber dejado de ser confiable.

## 1. Próximo documento recomendado

El siguiente documento será:
`19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`
Este documento deberá definir de forma transversal la defensa contra abuso del sistema Authentication:

- Authentication Rate Limiting
- Login Throttling
- Brute Force Protection
- Credential Stuffing
- Password Spraying
- Token Guessing
- OTP Guessing
- Recovery Abuse
- MFA Challenge Abuse
- Passkey Challenge Flooding
- Federation Abuse
- Distributed Attack Detection
- IP Limits
- Identity Limits
- Tenant Limits
- Device Limits

Network / ASN Signals
Progressive Throttling
Exponential Backoff boundaries
Cooldowns
CAPTCHA integration
Proof-of-work boundaries
Risk integration
Lockout avoidance
Account Enumeration resistance
Denial-of-service resistance
Hashing resource protection
Distributed counters
Redis-backed rate limiting
Atomic counters
Sliding windows
Token buckets
leaky buckets
fixed windows
adaptive limits
trusted network profiles
security telemetry
FrankenPHP safety
La separación será importante porque controles como:

- Password attempt limits
- OTP attempt limits
- Recovery limits
- Token verification limits
- Federation initiation limits

no deberán implementarse de manera aislada dentro de cada Authenticator.
VoltStack tendrá un Authentication Abuse Protection System transversal, capaz de proteger todos los mecanismos de autenticación sin convertir un atacante remoto en alguien capaz de bloquear permanentemente cuentas legítimas.
