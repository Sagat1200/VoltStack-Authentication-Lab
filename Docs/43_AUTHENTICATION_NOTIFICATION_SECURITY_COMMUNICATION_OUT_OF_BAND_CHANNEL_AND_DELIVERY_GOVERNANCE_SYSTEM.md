# VoltStack Authentication System

## 43 — Authentication Notification, Security Communication, Out-of-Band Channel and Delivery Governance System

- **Archivo:** `43_AUTHENTICATION_NOTIFICATION_SECURITY_COMMUNICATION_OUT_OF_BAND_CHANNEL_AND_DELIVERY_GOVERNANCE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Communication / OOB / Delivery Governance
- **Dependencias principales:** 15, 18, 19, 20, 21, 22, 23, 24, 29, 30, 31, 32, 35, 36, 37, 38, 39, 40, 41, 42.

---

## 1. Propósito

Este documento define la arquitectura responsable de todas las comunicaciones de seguridad originadas por Authentication.
El sistema deberá manejar de forma coherente:
Security Notifications
Authentication Alerts
Out-of-Band Challenges
One-Time Codes
Magic Links
Recovery Communications
Verification Messages
New Login Notifications
Password Change Notifications
MFA Change Notifications
Passkey Notifications
Device Notifications
Session Notifications
Account Protection Notifications
Account Lifecycle Notifications
Administrative Security Messages
Machine Identity Security Notifications
y transportarlas mediante canales como:
Email
SMS
Push
In-App
Security Center
Webhooks
Enterprise Messaging
Custom Channels

## 2. Problema arquitectónico

Authentication puede detectar:
New Login
Suspicious Login
Credential Compromise
Password Changed
MFA Disabled
Passkey Added
Device Lost
Recovery Started
Account Locked
Session Revoked
Security Freeze
pero detectar un evento y comunicarlo son responsabilidades diferentes.
No:
Authentication Logic
      ↓
sendEmail()
La arquitectura correcta será:
Authentication Event
        ↓
Communication Policy
        ↓
Communication Intent
        ↓
Channel Selection
        ↓
Message Composition
        ↓
Delivery Governance
        ↓
Provider

## 3. Principio fundamental

Authentication decide qué evento de seguridad ocurrió; el Security Communication System decide si debe comunicarse, a quién, por qué canal, con qué contenido y bajo qué garantías.

## 4. Separación con documento 40

Documento 40 define:
Compromise Detection
Security Alerts
Incident Response
Account Protection
Documento 43 define:
Communication Intent
Recipient Resolution
Channel Selection
Message Construction
Out-of-Band Delivery
Provider Routing
Retries
Fallback
Delivery Evidence
Communication Governance
Por tanto:
40 = WHAT security condition occurred

43 = HOW that condition is communicated

## 5. Notification != Authentication Event

Ejemplo:
PasswordChanged
es un Authentication Event.
SendPasswordChangedSecurityNotification
es una Communication Intent.

## 6. Communication Intent

Se introduce:
final readonly class AuthenticationCommunicationIntent
{
    public function __construct(
        public AuthenticationCommunicationType $type,
        public AuthenticationCommunicationPurpose $purpose,
        public AuthenticationCommunicationRecipient $recipient,
        public AuthenticationCommunicationContext $context,
        public AuthenticationCommunicationPriority $priority,
    ) {}
}

## 7. Communication Type

enum AuthenticationCommunicationType: string
{
    case SecurityNotification = 'security_notification';
    case SecurityAlert = 'security_alert';
    case Verification = 'verification';
    case Challenge = 'challenge';
    case Recovery = 'recovery';
    case AccountProtection = 'account_protection';
    case Administrative = 'administrative';
    case MachineSecurity = 'machine_security';
}

## 8. Communication Purpose

enum AuthenticationCommunicationPurpose: string
{
    case Inform = 'inform';
    case Warn = 'warn';
    case Verify = 'verify';
    case Authenticate = 'authenticate';
    case Recover = 'recover';
    case Confirm = 'confirm';
    case Approve = 'approve';
    case Escalate = 'escalate';
    case IncidentResponse = 'incident_response';
}

## 9. Purpose Matters

Dos mensajes enviados por email pueden tener semánticas radicalmente distintas:
"Your password changed."
vs:
"Use this link to sign in."
El primero es informativo.
El segundo transporta Authentication authority.

## 10. Security Classification

Cada comunicación tendrá clasificación.
enum AuthenticationCommunicationSecurityClass: string
{
    case Informational = 'informational';
    case Sensitive = 'sensitive';
    case AuthenticationBearing = 'authentication_bearing';
    case RecoveryBearing = 'recovery_bearing';
    case Privileged = 'privileged';
}

## 11. Authentication-Bearing Communication

Contiene material capaz de participar en Authentication.
Ejemplos:
OTP
Magic Link
Recovery Challenge
Verification Token
Approval Challenge
Debe recibir controles superiores a una simple notificación.

## 12. Communication != Credential

Una comunicación puede transportar una credential temporal.
Pero:
Email Message
   ≠
Credential
La credential tendrá lifecycle independiente.

## 13. OOB

Out-of-Band significa que parte del Authentication flow utiliza un canal distinto al canal principal.
Ejemplo:
Browser Login
      ↓
SMS OTP

## 14. OOB Channel

enum AuthenticationOutOfBandChannel: string
{
    case Email = 'email';
    case Sms = 'sms';
    case Push = 'push';
    case AuthenticatorPush = 'authenticator_push';
    case Voice = 'voice';
    case InApp = 'in_app';
    case Custom = 'custom';
}

## 15. OOB Does Not Automatically Mean Strong MFA

Enviar OTP por SMS:
Password
+
SMS OTP
no debe clasificarse automáticamente como phishing-resistant authentication.

## 16. Channel Assurance

Cada canal tendrá propiedades de seguridad.
final readonly class AuthenticationChannelAssurance
{
    public function __construct(
        public bool $outOfBand,
        public bool $phishingResistant,
        public bool $deviceBound,
        public bool $possessionEvidence,
        public bool $confidentialityExpected,
        public bool $deliveryIntegrityExpected,
    ) {}
}

## 17. Channel Assurance != Delivery Success

SMS delivered
no significa:
correct human authenticated

## 18. Channel Registry

interface AuthenticationCommunicationChannelRegistryInterface
{
    public function resolve(
        AuthenticationCommunicationChannelId $channel
    ): AuthenticationCommunicationChannelInterface;
}

## 19. Channel Contract

interface AuthenticationCommunicationChannelInterface
{
    public function supports(
        AuthenticationCommunicationEnvelope $envelope
    ): bool;

    public function deliver(
        AuthenticationCommunicationEnvelope $envelope
    ): AuthenticationDeliveryResult;
}

## 20. Core Channels

VoltStack podrá proporcionar adapters para:
Email
SMS
Push
In-App
Webhook
sin acoplar Authentication a proveedores concretos.

## 21. Provider != Channel

Ejemplo:
Channel = SMS
Provider = Twilio
o:
Channel = Email
Provider = SES
Conceptualmente:
Communication Intent
       ↓
Channel
       ↓
Provider

## 22. Provider Contract

interface AuthenticationDeliveryProviderInterface
{
    public function deliver(
        AuthenticationProviderDeliveryRequest $request
    ): AuthenticationProviderDeliveryResult;
}

## 23. Provider Capabilities

Cada provider podrá declarar:
Email
SMS
Push
Webhook
Delivery Receipts
Priority
Region
Tenant Support
Template Support

## 24. Provider Capability Registry

interface AuthenticationDeliveryProviderRegistryInterface
{
    public function providersFor(
        AuthenticationCommunicationChannelId $channel
    ): AuthenticationDeliveryProviderSet;
}

## 25. Provider Selection

No deberá estar hard-coded:
if ($channel === 'sms') {
    useTwilio();
}

## 26. Provider Resolver

interface AuthenticationDeliveryProviderResolverInterface
{
    public function resolve(
        AuthenticationDeliveryProviderContext $context
    ): AuthenticationDeliveryProviderSelection;
}

## 27. Resolver Inputs

Puede considerar:
Channel
Tenant
Realm
Environment
Region
Provider Health
Priority
Data Residency
Cost Policy
Security Policy
Message Type

## 28. Provider Failover

Ejemplo:
Primary Email Provider
        ↓ failure
Secondary Email Provider
pero solo si policy permite transferir datos al segundo proveedor.
Documento 42.

## 29. Failover Is Governed

No:
Provider A failed
→ send identity metadata to any available provider

## 30. Communication Policy Engine

interface AuthenticationCommunicationPolicyEngineInterface
{
    public function evaluate(
        AuthenticationCommunicationIntent $intent,
        AuthenticationCommunicationPolicyContext $context
    ): AuthenticationCommunicationDecision;
}

## 31. Communication Decision

Puede determinar:
SEND
SUPPRESS
DEFER
ESCALATE
REQUIRE_STRONGER_CHANNEL
REQUIRE_VERIFIED_CHANNEL
REQUIRE_ADMIN_REVIEW

## 32. Policy Inputs

Communication Type
Purpose
Security Class
Identity
Tenant
Realm
Risk
Incident State
Account State
Channel Verification
Recipient Preferences
Security Requirements
Privacy Policy
Provider Availability

## 33. Mandatory Security Communications

Ciertas comunicaciones podrán ser no deshabilitables.
Ejemplos conceptuales:
Password Changed
MFA Removed
Recovery Method Changed
Account Recovery Completed
Security Freeze Applied
Critical Credential Compromise

## 34. Preference != Mandatory Security Policy

Usuario puede decir:
Do not send product notifications.
Eso no necesariamente implica:
Do not warn me if my password changes.

## 35. Communication Preference

final readonly class AuthenticationCommunicationPreference
{
    public function __construct(
        public AuthenticationCommunicationPurpose $purpose,
        public AuthenticationCommunicationChannelId $channel,
        public bool $enabled,
    ) {}
}

## 36. Preference vs Consent

Documento 42.
Preference
≠
Consent
≠
Security Requirement

## 37. Recipient Resolution

No todas las comunicaciones van al authenticated identity.
Podrían ir a:
Identity
Security Contact
Tenant Security Administrator
Platform Security Team
Incident Response Team
Machine Identity Owner
Application Owner
Webhook Endpoint

## 38. Recipient Contract

interface AuthenticationCommunicationRecipientResolverInterface
{
    public function resolve(
        AuthenticationCommunicationIntent $intent
    ): AuthenticationCommunicationRecipientSet;
}

## 39. Recipient Types

enum AuthenticationCommunicationRecipientType: string
{
    case Identity = 'identity';
    case SecurityContact = 'security_contact';
    case TenantAdministrator = 'tenant_administrator';
    case PlatformSecurity = 'platform_security';
    case MachineOwner = 'machine_owner';
    case ApplicationOwner = 'application_owner';
    case Webhook = 'webhook';
}

## 40. Recipient Snapshot

Delivery debe utilizar un recipient snapshot adecuado.

## 41. Contact Point

final readonly class AuthenticationContactPoint
{
    public function__construct(
        public AuthenticationContactPointId $id,
        public AuthenticationContactPointType $type,
        public string $value,
        public AuthenticationContactPointStatus $status,
    ) {}
}

## 42. Contact Point Types

EMAIL
PHONE
PUSH_ENDPOINT
WEBHOOK
IN_APP

## 43. Contact Verification

enum AuthenticationContactPointStatus: string
{
    case Pending = 'pending';
    case Verified = 'verified';
    case Suspended = 'suspended';
    case Compromised = 'compromised';
    case Revoked = 'revoked';
}

## 44. Verified Contact

No significa automáticamente que sea apropiado para cualquier security purpose.

## 45. Contact Purpose Binding

Un email puede estar permitido para:
Security Alerts
pero no necesariamente:
Passwordless Login

## 46. Contact Policy

interface AuthenticationContactPointPolicyInterface
{
    public function evaluate(
        AuthenticationContactPoint $contact,
        AuthenticationCommunicationPurpose $purpose
    ): AuthenticationContactPointDecision;
}

## 47. Verification Ceremony

Agregar un nuevo email/phone security contact deberá requerir verificación.

## 48. Security Contact Change

Cambiar recovery/security contact puede ser una operación altamente sensible.
Debe integrarse con 32.

## 49. Contact Change Attack

Atacante con sesión robada podría intentar:
Add attacker email
        ↓
Make it recovery channel
        ↓
Remove legitimate MFA
        ↓
Take over account

## 50. Protection

Cambios críticos pueden requerir:
Fresh Authentication
MFA
Passkey
Existing Contact Confirmation
Cooldown
Security Notification
Risk Evaluation

## 51. Cooldown

Un nuevo contact point puede estar verificado pero temporalmente restringido para recovery.

## 52. Contact Activation Time

final readonly class AuthenticationContactActivationPolicy
{
    public function __construct(
        public DateInterval|null $recoveryCooldown,
        public bool $notifyExistingContacts,
    ) {}
}

## 53. Old Contact Notification

Cuando security contact cambia:
New Contact
    ↓
Verify
    ↓
Notify Old Contact
cuando sea apropiado.

## 54. Avoid Confirmation Attack

No incluir información que revele innecesariamente el nuevo contacto.

## 55. Contact Redaction

Ejemplo:
j***@example.com
+52 ******1234

## 56. Channel Selection

interface AuthenticationCommunicationChannelSelectorInterface
{
    public function select(
        AuthenticationCommunicationIntent $intent,
        AuthenticationCommunicationChannelContext $context
    ): AuthenticationCommunicationChannelSelection;
}

## 57. Selection

Puede devolver:
PRIMARY
FALLBACKS
PARALLEL
ESCALATION

## 58. Example

Critical Account Compromise

Primary:
Push

Parallel:
Email

Escalation:
Security Center

Fallback:
SMS
según policy.

## 59. Channel Selection != User Preference Only

Debe combinar:
Security Policy
User Preference
Channel Availability
Verification
Privacy
Tenant Policy
Incident Severity

## 60. Channel Restrictions

Ejemplo:
Admin Realm
puede prohibir:
SMS as sole recovery channel

## 61. Assurance Integration

Documento 37.
Cada channel/method deberá mapearse a propiedades de assurance.

## 62. Authentication Policy Integration

Documento 36.
Ejemplo:
Operation requires phishing-resistant authentication
No puede satisfacerse simplemente enviando email OTP.

## 63. Challenge Integration

Documento 38.
Challenge Negotiation puede preguntar:
Which OOB challenges are currently available?

## 64. Transaction Integration

Documento 39.
OOB challenges deberán estar ligados a Authentication Transaction.

## 65. OOB Challenge

final readonly class AuthenticationOutOfBandChallenge
{
    public function __construct(
        public AuthenticationChallengeId $id,
        public AuthenticationTransactionId $transaction,
        public AuthenticationCommunicationChannelId $channel,
        public DateTimeImmutable $expiresAt,
        public AuthenticationChallengePurpose $purpose,
    ) {}
}

## 66. Challenge Binding

Debe incluir:
Transaction
Identity
Tenant
Realm
Purpose
Operation
Channel
Expiration

## 67. OTP

One-Time Password deberá ser:
short-lived
single-use
purpose-bound
transaction-bound where possible
rate-limited

## 68. OTP Secret Storage

No almacenar OTP plaintext si puede evitarse.

## 69. OTP Verifier

Preferir:
OTP
   ↓
secure verifier
con protección contra brute force.

## 70. OTP Entropy

Longitud y espacio de búsqueda deberán alinearse con:
expiration
attempt limits
rate limiting
channel threat model

## 71. OTP Attempt Counter

Debe ser server-side authoritative.

## 72. OTP Consumption

Atómico:
VALID
+
UNCONSUMED
+
NOT EXPIRED
+
ATTEMPTS AVAILABLE
        ↓
atomic consume

## 73. OTP Replay

Segundo uso:
REJECT

## 74. OTP Resend

No deberá crear ilimitados OTPs válidos simultáneamente sin policy.

## 75. OTP Generation Strategy

Puede:
invalidate previous
o mantener una bounded generation family.
Default preferido:
new OTP
→ supersede previous OTP

## 76. OTP Generation Version

challenge_generation = 4
Solo generación actual aceptada.

## 77. OTP Delivery Delay

No extender automáticamente lifetime por retrasos del provider.

## 78. Delivery vs Challenge Expiration

Challenge lifetime es authority del Authentication system.
Provider no decide expiración.

## 79. SMS OTP

Amenazas:
SIM Swap
SS7 attacks
Number recycling
Malware
Message forwarding
Social engineering
Phishing

## 80. SMS Classification

SMS podrá ser soportado, pero no se clasificará como phishing-resistant.

## 81. Email OTP

Amenazas:
Compromised mailbox
Forwarding
Phishing
Shared mailbox
Long-lived sessions

## 82. Email OTP != Email Verification

Aunque utilicen el mismo canal.

## 83. Verification Code

Confirma control de un contact point.

## 84. Authentication Code

Participa en Authentication.
No deben compartir necesariamente:
purpose
TTL
rate limit
storage
templates

## 85. Magic Links

Un magic link es una bearer credential temporal.

## 86. Magic Link Security

Debe incluir:
High entropy token
Short expiration
Single use
Purpose binding
Transaction binding where appropriate
Tenant binding
Realm binding
Secure transport

## 87. Magic Link Token Storage

Preferir almacenar verifier/hash.

## 88. URL

Nunca incluir:
password
session cookie
long-lived API token

## 89. Query String Leakage

Magic tokens pueden filtrarse mediante:
Browser history
Referrer
Proxy logs
Analytics
Screenshots

## 90. Landing Page

Después de consumir token, redirigir a URL limpia.

## 91. Referrer Policy

Páginas de magic/recovery links deberán utilizar políticas restrictivas.

## 92. Third-Party Resources

Evitar cargar analytics/third-party resources en páginas que contienen Authentication tokens.

## 93. Link Scanner Problem

Email security scanners pueden abrir links automáticamente.

## 94. Critical Rule

Un GET prefetch no debería necesariamente consumir una credential crítica.

## 95. Two-Step Consumption

Puede utilizar:
GET token landing page
       ↓
validate candidate
       ↓
explicit user confirmation / POST
       ↓
consume
según flow.

## 96. Link Preview

Diseñar flows considerando bots, crawlers y preview systems.

## 97. Magic Link Device Binding

Opcionalmente puede ligarse a:
browser transaction
device
nonce
pero debe balancearse con UX.

## 98. Recovery Links

Más sensibles que simple email verification.

## 99. Recovery Communication

Documento 18.
Debe considerar:
Recovery State
Cooldown
Risk
Existing Methods
Security Review
Account Protection

## 100. Recovery Channel

No todo contact point verificado deberá convertirse automáticamente en recovery channel.

## 101. Recovery Channel Registry

interface AuthenticationRecoveryChannelRegistryInterface
{
    public function channelsFor(
        IdentityReference $identity
    ): AuthenticationRecoveryChannelSet;
}

## 102. Recovery Communication Privacy

No revelar si cuenta existe.

## 103. Account Enumeration

Solicitud:
Forgot password for <user@example.com>
respuesta pública:
If an eligible account exists, instructions will be sent.

## 104. Internal Outcome

Internamente:
ACCOUNT_NOT_FOUND
RECOVERY_NOT_ALLOWED
MESSAGE_QUEUED
RATE_LIMITED
pueden diferenciarse.

## 105. Timing Leakage

Evitar diferencias extremas de timing que faciliten enumeration cuando sea práctico.

## 106. Delivery Status Leakage

No devolver:
SMS delivered to +52...
antes de establecer autoridad adecuada.

## 107. Security Notifications

Ejemplos:
New Login
New Device
Password Changed
Password Reset
MFA Enabled
MFA Disabled
Passkey Added
Passkey Removed
Recovery Codes Regenerated
Federated Account Linked
Federated Account Unlinked
Session Revoked
Global Logout
Device Trusted
Device Revoked
Account Locked
Account Suspended
Account Reactivated
Security Freeze
Compromise Detected

## 108. Notification Catalog

interface AuthenticationSecurityNotificationCatalogInterface
{
    public function definition(
        AuthenticationSecurityNotificationType $type
    ): AuthenticationSecurityNotificationDefinition;
}

## 109. Definition

final readonly class AuthenticationSecurityNotificationDefinition
{
    public function __construct(
        public AuthenticationSecurityNotificationType $type,
        public AuthenticationCommunicationPriority $priority,
        public bool $mandatory,
        public AuthenticationCommunicationSecurityClass $securityClass,
        public AuthenticationChannelPolicyId $channelPolicy,
    ) {}
}

## 110. Priority

enum AuthenticationCommunicationPriority: string
{
    case Low = 'low';
    case Normal = 'normal';
    case High = 'high';
    case Critical = 'critical';
}

## 111. Priority != Queue Priority Only

También puede afectar:
fallback
retry
parallel channels
provider selection
escalation

## 112. Critical Notification

Ejemplo:
Account takeover suspected
puede requerir:
Push + Email + Security Center

## 113. Notification Content

Debe ser action-oriented.
Ejemplo:
A new sign-in was detected.

Device:
Chrome on Windows

Approximate location:
Monterrey, Mexico

If this was you:
No action is required.

If this wasn't you:
Review your account security.

## 114. Avoid Excessive Detail

No enviar:
full IP
risk model score
internal security rule IDs
credential identifiers
salvo policy.

## 115. "This Wasn't Me"

Debe dirigir a un safe incident response entry point.
Documento 40.

## 116. Link Security

Nunca:
/compromise?user=123&session=456
con identifiers explotables.

## 117. Action Token

Usar transaction-bound security action token si la acción necesita authority.

## 118. Notification Does Not Grant Authority

Una security notification puede contener enlace a Security Center sin autenticar automáticamente al usuario.

## 119. Message Composer

interface AuthenticationCommunicationComposerInterface
{
    public function compose(
        AuthenticationCommunicationIntent $intent,
        AuthenticationCommunicationCompositionContext $context
    ): AuthenticationCommunicationMessage;
}

## 120. Structured Message

final readonly class AuthenticationCommunicationMessage
{
    public function __construct(
        public string $template,
        public AuthenticationCommunicationSubject|null $subject,
        public AuthenticationCommunicationBody $body,
        public AuthenticationCommunicationActionSet $actions,
        public AuthenticationCommunicationMetadata $metadata,
    ) {}
}

## 121. Domain Data != Presentation

El Authentication Event no deberá contener HTML email.

## 122. Template System

Communication Intent
       ↓
Template Resolver
       ↓
Locale
       ↓
Tenant Branding
       ↓
Channel Renderer

## 123. Template ID

Usar identificadores estables:
auth.security.password_changed
auth.security.new_login
auth.recovery.started
auth.challenge.email_otp

## 124. Template Version

Security-sensitive templates deberían ser versionables.

## 125. Why Versioning

Permite:
audit
rollback
translation synchronization
anti-phishing review

## 126. Localization

Mensajes deberán soportar locale.

## 127. Locale Resolution

Puede considerar:
identity preference
tenant default
application locale
safe fallback

## 128. Locale Is Not Trusted Input for Template Path

Nunca:
include $locale . '/message.php';
sin resolver contra registry seguro.

## 129. Tenant Branding

Puede incluir:
Tenant Name
Logo
Support URL
Brand Name

## 130. Branding Security

Tenant no podrá inyectar:
arbitrary HTML
JavaScript
unsafe URLs
tracking pixels
credential collection forms
en templates críticos.

## 131. Trusted Branding Model

final readonly class AuthenticationCommunicationBranding
{
    public function__construct(
        public string $displayName,
        public SafeAssetReference|null $logo,
        public SafeUrl|null $supportUrl,
    ) {}
}

## 132. Anti-Phishing Design

Mensajes de Authentication deberán reducir patrones que entrenen al usuario a entregar secretos.

## 133. Password Requests

VoltStack jamás enviará:
Reply with your password.

## 134. TOTP Requests

Nunca solicitar TOTP por email de respuesta.

## 135. Security Message Rule

VoltStack security communications nunca solicitarán passwords, recovery codes o private keys mediante respuesta al mensaje.

  1. Anti-Phishing Phrase
Puede incluir:
VoltStack will never ask you to send your password by email.
según producto.
  2. Domain Governance
Links de Authentication deberán usar dominios permitidos.
  3. Allowed Link Host Registry
interface AuthenticationCommunicationLinkPolicyInterface
{
    public function validate(
        AuthenticationCommunicationLink $link,
        AuthenticationCommunicationContext $context
    ): AuthenticationCommunicationLinkDecision;
}
  4. Open Redirect
Prohibido en Authentication links.
  5. Return URL
Debe validarse contra allowlist/routing policy.
  6. URL Signing
Links pueden utilizar signatures/tokens según flow.
  7. Short URLs
Evitar third-party public URL shorteners para Authentication links.
  8. Tracking Links
Marketing click tracking deberá estar deshabilitado para security-critical links por default.
  9. Email Provider Link Rewriting
Debe considerarse porque algunos providers reescriben URLs.
 10. Security Templates
Deberán poder prohibir provider-side link tracking.
 11. Email Headers
Evitar leaking unnecessary identifiers.
 12. Message Metadata
Nunca poner secrets en:
Subject
Message-ID
Custom headers
Provider tags
Analytics labels
 13. SMS Content
Debe minimizar información.
No:
Your admin account for ACME Corp was compromised because...
si no es necesario.
 14. OTP SMS Example
Your VoltStack verification code is 482913.
It expires in 5 minutes.
Do not share this code.
 15. OTP Purpose Context
Cuando sea útil:
Code to approve password change
es mejor que código completamente ambiguo.
 16. Transaction Context
Para operaciones críticas puede mostrarse:
Approve sign-in to Admin Console
 17. Number Matching
Push authentication puede utilizar:
browser displays 42
mobile asks select 42
para reducir MFA fatigue.
 18. Push Challenge
final readonly class AuthenticationPushChallenge
{
    public function __construct(
        public AuthenticationChallengeId $challenge,
        public AuthenticationTransactionId $transaction,
        public AuthenticationPushChallengeType $type,
        public DateTimeImmutable $expiresAt,
    ) {}
}
 19. Push Responses
APPROVE
DENY
EXPIRED
CANCELLED
 20. Push Approval != Generic Button
Debe ligarse a challenge específico.
 21. MFA Fatigue
Atacante puede bombardear push requests.
 22. Push Fatigue Protection
Rate Limits
Number Matching
Transaction Context
User Denial Signal
Cooldown
Risk Escalation
 23. Repeated Denials
Pueden generar security signal.
Documento 40.
 24. In-App Notifications
Ventajas:
Authenticated context
Rich security details
Direct Security Center integration
 25. Limitation
No sirven si atacante bloqueó acceso o usuario no tiene sesión.
 26. Security Center Channel
Security Center puede ser un durable communication surface.
 27. Communication Inbox
Opcional:
Security Communication Inbox
para mensajes importantes.
 28. Inbox Message
final readonly class AuthenticationSecurityInboxMessage
{
    public function __construct(
        public AuthenticationCommunicationId $id,
        public AuthenticationSecurityNotificationType $type,
        public DateTimeImmutable $createdAt,
        public AuthenticationInboxMessageStatus $status,
    ) {}
}
 29. Inbox Status
UNREAD
READ
ACKNOWLEDGED
RESOLVED
ARCHIVED
 30. Inbox != Audit
Usuario puede borrar/archivar mensaje.
Audit continúa independiente.
 31. Webhooks
Enterprise tenants pueden recibir Authentication security events.
 32. Webhook Security
Debe utilizar:
HTTPS
Signature
Timestamp
Replay Protection
Secret Rotation
Event ID
Delivery ID
 33. Webhook Payload
Minimizado según documento 42.
 34. Webhook Signature
Ejemplo conceptual:
timestamp + "." + canonical_payload
        ↓
HMAC / asymmetric signature
 35. Webhook Replay
Receiver puede verificar timestamp + event ID.
 36. Webhook Secrets
Gestionados mediante documento 31.
 37. Webhook Retry
Debe mantener mismo:
event ID
pero puede usar nuevo:
delivery attempt ID
 38. Delivery Identity
final readonly class AuthenticationDeliveryId
{
    public function __construct(
        public string $value
    ) {}
}
 39. Communication Identity
final readonly class AuthenticationCommunicationId
{
    public function __construct(
        public string $value
    ) {}
}
 40. Distinction
Communication ID
      │
      ├── Email Delivery ID
      ├── Push Delivery ID
      └── SMS Delivery ID
Una comunicación puede producir múltiples deliveries.
 41. Delivery Attempt
final readonly class AuthenticationDeliveryAttempt
{
    public function __construct(
        public AuthenticationDeliveryId $delivery,
        public AuthenticationDeliveryAttemptId $attempt,
        public int $sequence,
        public DateTimeImmutable $startedAt,
    ) {}
}
 42. Delivery Lifecycle
CREATED
   ↓
QUEUED
   ↓
DISPATCHED
   ↓
ACCEPTED_BY_PROVIDER
   ↓
DELIVERED
Alternativas:
FAILED
REJECTED
BOUNCED
EXPIRED
CANCELLED
SUPPRESSED
 43. Delivered Semantics
DELIVERED significa lo que el provider puede demostrar.
No:
Human read it.
 44. Read Receipts
No deberán considerarse security proof.
 45. Delivery Receipt
final readonly class AuthenticationDeliveryReceipt
{
    public function __construct(
        public AuthenticationDeliveryId $delivery,
        public AuthenticationDeliveryReceiptStatus $status,
        public DateTimeImmutable $receivedAt,
        public string|null $providerReference,
    ) {}
}
 46. Provider Reference
No deberá usarse como authentication authority.
 47. Async Delivery
La mayoría de notificaciones serán asynchronous.
 48. Exception
Ciertos challenge flows pueden requerir que el sistema confirme:
provider accepted request
antes de mostrar:
Code sent.
pero no necesariamente esperar entrega real.
 49. Outbox Pattern
Recomendado:
Authentication Transaction
        ↓
Commit
        ↓
Transactional Outbox
        ↓
Communication Worker
        ↓
Provider
 50. Why Outbox
Evita:
Password changed
DB committed
Email enqueue failed
Notification lost
 51. Eventual Delivery
Security mutation puede ser correcta aunque comunicación llegue segundos después.
 52. Critical Notifications
Pueden requerir stronger delivery guarantees.
 53. Communication Outbox
interface AuthenticationCommunicationOutboxInterface
{
    public function append(
        AuthenticationCommunicationEnvelope $envelope
    ): void;
}
 54. Envelope
final readonly class AuthenticationCommunicationEnvelope
{
    public function __construct(
        public AuthenticationCommunicationId $id,
        public AuthenticationCommunicationIntent $intent,
        public AuthenticationCommunicationPolicyVersion $policyVersion,
        public DateTimeImmutable $createdAt,
    ) {}
}
 55. Envelope Must Be Secret-Minimized
No serializar:
raw password
session cookie
private key
full OAuth response
 56. Challenge Secret Delivery
Si worker necesita OTP, utilizar secure temporary reference/material handling.
 57. Secret Reference
Envelope
   ↓
Challenge ID
   ↓
Secure Challenge Store
preferible a propagar secret por múltiples queues.
 58. Queue Encryption
Puede utilizarse como defense-in-depth.
No sustituye minimización.
 59. Queue Retention
Documento 42.
Authentication communication jobs no permanecerán indefinidamente.
 60. Expired Challenge Job
Si worker recibe:
OTP challenge expired
no deberá enviarlo.
 61. Pre-Dispatch Validation
Antes de enviar Authentication-bearing message:
challenge active?
recipient still valid?
account state allows?
transaction active?
communication cancelled?
 62. Stale Message Suppression
Ejemplo:
Password reset requested
        ↓
Password changed elsewhere
        ↓
Queued recovery email executes
Debe poder ser cancelado/suprimido.
 63. Communication Cancellation
interface AuthenticationCommunicationCancellationPolicyInterface
{
    public function shouldCancel(
        AuthenticationCommunicationEnvelope $envelope,
        AuthenticationCommunicationRuntimeContext $context
    ): bool;
}
 64. Communication Version
Security state changes pueden invalidar queued messages.
 65. Security Epoch
Integrar security epochs/versioning de Authentication.
Ejemplo:
communication.securityEpoch = 7

current identity securityEpoch = 8
Puede implicar invalidación según message type.

## 201. Retry

Retries deberán depender del tipo.

## 202. Informational Retry

Puede tolerar varios minutos.

## 203. OTP Retry

No debe llegar después de expiración.

## 204. Recovery Retry

Debe considerar transaction lifetime.

## 205. Retry Policy

interface AuthenticationDeliveryRetryPolicyInterface
{
    public function nextAttempt(
        AuthenticationDeliveryFailure $failure,
        AuthenticationDeliveryContext $context
    ): AuthenticationDeliveryRetryDecision;
}

## 206. Retry Decision

RETRY
RETRY_WITH_BACKOFF
FAILOVER
SUPPRESS
EXPIRE
ESCALATE

## 207. Exponential Backoff

Adecuado para provider outages.

## 208. Jitter

Evita retry storms.

## 209. Retry Budget

No reintentar infinitamente.

## 210. Expiration-Aware Retry

nextRetry >= challenge.expiresAt
→ DO NOT RETRY

## 211. Provider Failover

Debe evitar duplicate OTP messages confusos cuando sea posible.

## 212. Idempotency

Provider adapter debería utilizar idempotency key cuando provider la soporte.

## 213. Idempotency Key

Derivada de:
Delivery ID
no del secret.

## 214. Duplicate Delivery

Debe ser observable.

## 215. Deduplication

interface AuthenticationCommunicationDeduplicatorInterface
{
    public function acquire(
        AuthenticationCommunicationDeduplicationKey $key
    ): AuthenticationCommunicationDeduplicationDecision;
}

## 216. Deduplication Example

PasswordChanged
same identity
same operation ID
same channel
no debería generar 12 emails por retries internos.

## 217. Deduplication != Suppression of Independent Events

Dos cambios reales de password son dos eventos.

## 218. Correlation

Usar:
AuthenticationOperationId
AuthenticationTransactionId
SecurityIncidentId
CommunicationId
DeliveryId
según contexto.

## 219. Rate Limiting

Documento 19 se aplica también aquí.

## 220. Dimensions

Identity
Destination
IP
Tenant
Channel
Purpose
Challenge Type
Provider

## 221. OTP Flood

Atacante podría enviar miles de SMS al teléfono de víctima.

## 222. Abuse Controls

Per identity limit
Per phone/email limit
Per source limit
Per tenant limit
Cooldown
Global abuse protection

## 223. Cost Abuse

SMS/voice tienen costo.
Resource governance deberá considerar:
security
+
financial abuse

## 224. Enumeration via Rate Limit

Mensajes públicos de throttling no deben revelar si destination pertenece a cuenta.

## 225. Resend UI

Puede mostrar:
Try again in 30 seconds.
sin revelar estado interno.

## 226. Global Capacity

Provider outage no debe permitir que queue crezca sin límite.

## 227. Backpressure

Queue Capacity
Provider Capacity
Tenant Quotas
Priority
deberán gobernarse.

## 228. Priority Scheduling

Critical security alerts pueden tener prioridad sobre informational notifications.

## 229. But No Starvation

Low-priority messages tampoco deben quedar eternamente pendientes.

## 230. Circuit Breaker

Provider adapters pueden usar:
CLOSED
OPEN
HALF_OPEN

## 231. Provider Health

interface AuthenticationDeliveryProviderHealthInterface
{
    public function status(
        AuthenticationDeliveryProviderId $provider
    ): AuthenticationDeliveryProviderHealthStatus;
}

## 232. Health Status

HEALTHY
DEGRADED
UNAVAILABLE
DISABLED

## 233. Automatic Failover

Solo cuando:
policy allows
+
privacy allows
+
provider capability matches

## 234. Provider Credentials

Nunca en application config plaintext cuando secure secret provider esté disponible.
Documento 31.

## 235. Provider Credential Rotation

Sin detener Authentication delivery cuando sea posible.

## 236. Tenant Providers

Tenant podría usar su propio:
SMTP
SMS Provider
Webhook
Push Infrastructure
si policy lo permite.

## 237. Platform Provider

Puede existir fallback central.

## 238. Tenant Isolation

Tenant A nunca utilizará accidentalmente:
Tenant B SMTP credentials
Tenant B branding
Tenant B recipient lists
Tenant B webhook secret

## 239. Provider Context

Debe incluir tenant explícitamente.

## 240. No Ambient Tenant

No depender de:
Tenant::current()
mutable global dentro de async worker.

## 241. Explicit Scope

final readonly class AuthenticationDeliveryScope
{
    public function __construct(
        public TenantId|null $tenant,
        public RealmId|null $realm,
        public ApplicationId|null $application,
        public EnvironmentId $environment,
    ) {}
}

## 242. Async Tenant Safety

Job serializa:
Tenant ID
Realm ID
Application ID
no mutable tenant object global.

## 243. Realm Separation

Admin realm puede utilizar:
different sender
different templates
different provider
different escalation

## 244. Admin Notifications

Ejemplo:
New privileged login
Admin MFA changed
Break-glass account used
Security policy changed

## 245. Break-Glass Communications

Documento 32.
Uso de break-glass deberá generar comunicaciones altamente prioritarias.

## 246. Break-Glass Rule

Break-Glass Authentication
        ↓
Immediate Security Communication
        ↓
Security Team
        +
Audit
        +
Incident/Review Workflow

## 247. Break-Glass Notification Suppression

El actor break-glass no deberá poder silenciar por sí solo la alerta.

## 248. Independent Delivery

Idealmente usar canal suficientemente independiente del sistema que se está recuperando.

## 249. IdP Outage Example

Si break-glass existe por caída del IdP, no depender únicamente del mismo IdP para alertar.

## 250. Machine Identities

Documento 33.
Las máquinas no necesariamente reciben emails.

## 251. Machine Security Recipients

Alertas pueden ir a:
Service Owner
Application Owner
Security Team
Tenant Admin
Webhook
Incident Platform

## 252. Machine Credential Expiry

Puede generar:
certificate expires in 30 days
signing key expires
API credential stale
workload trust misconfigured

## 253. Machine Notification != Human Authentication Challenge

No enviar SMS MFA a service account.

## 254. Webhook / Automation

Más apropiado para machine security.

## 255. Expiration Notifications

Credential lifecycle puede generar:
30 days
7 days
1 day
alerts.

## 256. Deduplicate Scheduled Alerts

No duplicar al reiniciar workers.

## 257. Scheduled Communication Key

Puede incluir:
credential
notification type
scheduled threshold

## 258. Communication Governance

Toda comunicación deberá responder:
Why?
Who?
What?
Which channel?
Which provider?
What data?
How long valid?
Can it retry?
Can it fail over?
Can user disable it?
What is audited?

## 259. Governance Definition

final readonly class AuthenticationCommunicationGovernanceDefinition
{
    public function__construct(
        public AuthenticationCommunicationType $type,
        public AuthenticationCommunicationPurpose $purpose,
        public bool $mandatory,
        public AuthenticationCommunicationSecurityClass $securityClass,
        public AuthenticationChannelPolicyId $channelPolicy,
        public AuthenticationRetryPolicyId $retryPolicy,
        public AuthenticationRetentionPolicyId $retentionPolicy,
    ) {}
}

## 260. Governance Registry

interface AuthenticationCommunicationGovernanceRegistryInterface
{
    public function definition(
        AuthenticationCommunicationDefinitionId $id
    ): AuthenticationCommunicationGovernanceDefinition;
}

## 261. Policy Hierarchy

Framework Security Floor
          ↓
Platform
          ↓
Environment
          ↓
Realm
          ↓
Tenant
          ↓
Application
          ↓
Communication Type

## 262. Tenant Hardening

Tenant puede requerir:
all admin security events → email + webhook

## 263. Tenant Weakening

Tenant no podrá deshabilitar mandatory platform alert si platform floor lo prohíbe.

## 264. Communication Policy Composition

Debe seguir semántica explícita, no last-write-wins.

## 265. Example

Platform:
PasswordChanged → mandatory email

Tenant:
PasswordChanged → webhook

Effective:
Email + Webhook
No:
Webhook replaces mandatory Email
si policy no lo permite.

## 266. Required Channels

Composición:
union
normalmente.

## 267. Forbidden Channels

Composición:
union

## 268. Allowed Channels

Composición:
intersection

## 269. Minimum Priority

Composición:
max

## 270. Maximum Delivery Delay

Composición:
min

## 271. Contradiction

Required = SMS
Forbidden = SMS
→ policy conflict.

## 272. Conflict

Nunca fallback silencioso a un canal inseguro.

## 273. Policy Compiler

interface AuthenticationCommunicationPolicyCompilerInterface
{
    public function compile(
        AuthenticationCommunicationPolicySet $policies
    ): CompiledAuthenticationCommunicationPolicySet;
}

## 274. Compiler Validations

Required/Forbidden conflict
No eligible provider
Unsupported channel
Invalid fallback
Invalid tenant override
Unsafe template
Missing mandatory notification
Unknown recipient
Invalid retry lifetime

## 275. Policy Version

final readonly class AuthenticationCommunicationPolicyVersion
{
    public function __construct(
        public string $value
    ) {}
}

## 276. Template Compilation

Templates pueden compilarse/cacharse.

## 277. No Secrets in Compiled Templates

Obvio pero obligatorio.

## 278. Template Variable Schema

Cada template declarará variables permitidas.

## 279. Example

auth.security.new_login

Allowed:
device_display
approximate_location
occurred_at
security_center_url
No:
raw_session_token
password_hash
risk_engine_dump

## 280. Template Variable Validation

interface AuthenticationCommunicationTemplateSchemaInterface
{
    public function validate(
        AuthenticationTemplateVariables $variables
    ): AuthenticationTemplateValidationResult;
}

## 281. Escaping

Cada renderer debe escapar según contexto:
HTML
Plain Text
SMS
JSON
Push

## 282. HTML Injection

Tenant name, device name, location, browser, etc. son untrusted presentation data.

## 283. Email HTML

Siempre escapar.

## 284. Webhook JSON

Serialización estructurada; no concatenación manual.

## 285. Header Injection

Email subject/sender/display names deberán sanitizar CR/LF.

## 286. SMS Injection

Evitar control characters/problematic payloads.

## 287. Unicode Security

Considerar:
homoglyphs
bidi controls
invisible characters
en tenant branding/security display.

## 288. Safe Display Normalization

Especialmente en:
device names
application names
tenant names

## 289. Sender Identity

Email security notifications deberán utilizar sender identity estable.

## 290. Email Authentication

Deployment deberá soportar/configurar adecuadamente:
SPF
DKIM
DMARC
fuera del core protocol, pero tooling puede verificar posture.

## 291. Domain Verification

Custom tenant sender domain deberá verificarse antes de usarlo.

## 292. Tenant Sender Attack

Tenant no puede establecer:
security@google.com
arbitrariamente.

## 293. Sender Domain Registry

interface AuthenticationSenderDomainRegistryInterface
{
    public function status(
        AuthenticationSenderDomain $domain
    ): AuthenticationSenderDomainStatus;
}

## 294. Sender Domain Status

PENDING
VERIFIED
ACTIVE
SUSPENDED
REVOKED

## 295. Reply-To

Security messages preferiblemente no dependen de respuestas para Authentication.

## 296. Bounce Handling

Email bounce debe actualizar delivery health, no identity proof.

## 297. Phone Recycling

Un teléfono previamente verificado puede cambiar de propietario.

## 298. Contact Freshness

Policy puede exigir re-verificación después de determinado tiempo o risk signal.

## 299. Compromised Channel

Channel puede marcarse:
COMPROMISED

## 300. Channel Compromise

Debe excluirse de:
recovery
step-up
sensitive confirmation
según policy.

## 301. Contact Security State

enum AuthenticationContactSecurityState: string
{
    case Normal = 'normal';
    case Suspected = 'suspected';
    case Compromised = 'compromised';
    case Restricted = 'restricted';
}

## 302. Account Compromise

Documento 40 puede invalidar communication channels.

## 303. Example

Email account suspected compromised
        ↓
Do not send password recovery there
        ↓
Use passkey / recovery code / security review

## 304. Delivery Privacy

Documento 42.
Provider debe recibir mínimo data.

## 305. Provider Payload

No enviar entire Identity object.

## 306. Safe Provider Request

final readonly class AuthenticationProviderDeliveryRequest
{
    public function __construct(
        public AuthenticationDeliveryId $deliveryId,
        public AuthenticationDeliveryDestination $destination,
        public AuthenticationRenderedMessage $message,
        public AuthenticationProviderDeliveryMetadata $metadata,
    ) {}
}

## 307. Provider Metadata

Solo operational metadata.

## 308. Provider Tags

No incluir:
email=<user@example.com>
risk=credential_stolen
como tags si no es necesario.

## 309. Delivery Logs

Nunca registrar full OTP/magic token.

## 310. Safe Log

communication_id
delivery_id
channel
provider
template
outcome
latency

## 311. OTP Logging

No:
OTP 482913 sent successfully

## 312. Message Body Logging

Deshabilitado por default para Authentication-bearing communications.

## 313. Provider Debug Logs

Adapters deberán sanitizar SDK debug output.

## 314. Tracing

Spans:
auth.communication.resolve
auth.communication.compose
auth.communication.dispatch
auth.communication.provider
auth.communication.receipt

## 315. Trace Attributes

channel
provider
message_type
priority
outcome
attempt

## 316. No Recipient PII in Traces

No:
phone
email
por default.

## 317. Metrics

auth_communication_created_total
auth_communication_delivered_total
auth_communication_failed_total
auth_communication_suppressed_total
auth_communication_retry_total
auth_communication_failover_total
auth_communication_latency_seconds
auth_communication_queue_depth
auth_communication_expired_total
auth_oob_challenge_created_total
auth_oob_challenge_failed_total

## 318. Controlled Cardinality

Labels:
channel
provider
type
outcome
realm_class
No identity IDs.

## 319. Delivery SLO

Enterprise deployments podrán definir:
Critical security email accepted < 5 sec
Push alert < 2 sec
OTP provider acceptance < 3 sec
como operational targets.

## 320. Delivery SLO != Authentication Guarantee

Provider latency no debe confundirse con security assurance.

## 321. Audit

Security communication creation y delivery crítico serán auditables.

## 322. Audit Fields

Communication ID
Type
Purpose
Recipient Reference
Channel
Provider
Tenant
Realm
Policy Version
Template Version
Outcome
Timestamp

## 323. Audit Does Not Store Body by Default

Especialmente OTP/magic/recovery.

## 324. Communication Content Hash

Opcionalmente puede conservarse hash/canonical template reference para evidencia sin almacenar secret body.

## 325. Message Reconstruction

Para informational messages puede reconstruirse:
template version + safe variables
si policy lo requiere.

## 326. Authentication-Bearing Message

No debería ser reconstruible si implica recuperar secret.

## 327. Security Notification Acknowledgement

Algunas comunicaciones pueden permitir:
ACKNOWLEDGED

## 328. Acknowledgement != Resolution

I saw the alert
no significa:
incident resolved

## 329. Communication Escalation

Si critical notification no puede entregarse:
Primary channel failed
       ↓
Fallback
       ↓
Security Center
       ↓
Security Team
según policy.

## 330. Escalation Policy

interface AuthenticationCommunicationEscalationPolicyInterface
{
    public function escalate(
        AuthenticationCommunicationEscalationContext $context
    ): AuthenticationCommunicationEscalationDecision;
}

## 331. Escalation Loop

Debe impedir:
email fails
→ webhook
→ webhook generates email
→ email fails
→ ...

## 332. Escalation Depth

Bounded.

## 333. Communication Origin

enum AuthenticationCommunicationOrigin: string
{
    case AuthenticationEvent = 'authentication_event';
    case SecurityIncident = 'security_incident';
    case Challenge = 'challenge';
    case Recovery = 'recovery';
    case Administrative = 'administrative';
    case ScheduledMaintenance = 'scheduled_maintenance';
}

## 334. Origin Tracking

Permite evitar loops y mejorar audit.

## 335. User-Triggered Send

Ejemplo:
Resend OTP
también crea Communication Intent gobernada.

## 336. Admin-Triggered Send

No deberá permitir enviar arbitrary content desde Authentication Security Center.

## 337. Predefined Security Communication

Admins pueden activar tipos definidos.
No convertir sistema en generic mailer.

## 338. Generic Notification System

VoltStack puede tener otro módulo general para comunicaciones.
Authentication Communication podrá usarlo como transport abstraction.

## 339. Critical Boundary

El módulo general:
Notification
no debe decidir Authentication security semantics.

## 340. Correct Integration

Quantum\Auth\Communication
       ↓
Security semantics
       ↓
Platform Notification Transport
       ↓
Email/SMS/etc.

## 341. Framework Independence

Authentication no dependerá obligatoriamente de un mailer específico.

## 342. Laravel-Style Adapter

Puede existir:
LaravelMailAuthenticationChannel
en bridge/package de compatibilidad.

## 343. Symfony-Style Adapter

Puede existir integración con:
Mailer
Notifier
Messenger
sin contaminar core contracts.

## 344. Persistent Workers

FrankenPHP requiere aislamiento estricto.

## 345. No Mutable Current Recipient

Nunca:
static::$currentRecipient = $user;

## 346. No Mutable Current Tenant Branding

Nunca compartir entre requests.

## 347. No Mutable Template Context

Debe ser request/job scoped.

## 348. Worker Reset

Limpiar:
Recipient Context
Tenant Scope
Realm Scope
Branding
Locale
Template Variables
Provider Context
Temporary Challenge References

## 349. Fiber Safety

Ejemplo:
Fiber A
Tenant A
Spanish
Provider A

Fiber B
Tenant B
English
Provider B
No contaminación.

## 350. Async Worker Safety

Cada job reconstruirá scope explícitamente.

## 351. Provider Client Reuse

Sí puede reutilizarse cliente stateless/thread-safe.

## 352. Provider Credential Context

Si credentials son tenant-specific, no almacenarlas en mutable client singleton.

## 353. Provider Pool

Puede indexarse por immutable configuration reference/version.

## 354. Credential Rotation

Invalidará/reconstruirá clients según versión.

## 355. Distributed Delivery

Múltiples nodes pueden consumir queue.

## 356. Exactly-Once

No asumir exactamente una vez.
Diseñar:
at-least-once delivery infrastructure
+
idempotent communication semantics

## 357. Delivery Claim

Workers deberán adquirir claim/lease sobre delivery.

## 358. Lease

interface AuthenticationDeliveryLeaseStoreInterface
{
    public function acquire(
        AuthenticationDeliveryId $delivery,
        DateInterval $ttl
    ): AuthenticationDeliveryLeaseResult;
}

## 359. Worker Crash

Lease expira y otro worker reintenta.

## 360. Provider Ambiguity

Problema:
Provider accepted message
Worker crashed before persisting success
Retry puede duplicar.

## 361. Mitigation

Provider idempotency
Delivery IDs
Reconciliation
Bounded duplicate tolerance

## 362. Delivery Reconciliation

interface AuthenticationDeliveryReconcilerInterface
{
    public function reconcile(
        AuthenticationDeliveryId $delivery
    ): AuthenticationDeliveryReconciliationResult;
}

## 363. Multi-Region

Communication intent puede originarse en region A y provider operar en B.

## 364. Data Residency

Documento 42 gobierna transferencia.

## 365. Region-Aware Provider Resolution

Preferir provider permitido en región.

## 366. Regional Outage

Failover cross-region solo si policy permite.

## 367. Clock

TTL de challenges requiere clock consistente.

## 368. Clock Interface

interface AuthenticationClockInterface
{
    public function now(): DateTimeImmutable;
}

## 369. Provider Timestamp

Nunca usar provider timestamp como autoridad para challenge expiration.

## 370. Message Expiry

Communication envelope puede tener:
public DateTimeImmutable $notAfter;

## 371. Dispatch Rule

now >= notAfter
→ EXPIRE

## 372. Notification Expiration

Incluso informational alerts pueden perder relevancia.
Ejemplo:
Your OTP was requested 4 days ago.
no debe enviarse tarde.

## 373. Queue Poison Message

Malformed communication debe ir a controlled DLQ.

## 374. DLQ

Debe ser:
encrypted where appropriate
access-controlled
retention-bound
secret-minimized

## 375. Manual Replay

Security operator no puede simplemente replayar cualquier old OTP job.

## 376. Replay Validation

Re-evaluar:
message validity
challenge state
recipient
security epoch
policy

## 377. Testing

El sistema deberá cubrir unit, integration, distributed y adversarial tests.

## 378. Test — OTP Single Use

Generate
Send
Verify
Consume
Verify again
→ reject

## 379. Test — OTP Resend

OTP1
Resend
OTP2
OTP1
→ rejected
si policy supersede.

## 380. Test — Expired Queue

OTP expires
Worker executes
→ do not deliver

## 381. Test — Enumeration

Known y unknown account deberán tener safe public behavior.

## 382. Test — Provider Failover

Provider A falla.
Provider B solo se usa si privacy/policy lo permiten.

## 383. Test — Tenant Isolation

Tenant A SMTP nunca utilizado para Tenant B.

## 384. Test — Branding Injection

Tenant display name:
<script>alert(1)</script>
debe renderizarse seguro.

## 385. Test — Header Injection

Malicious display name con CRLF deberá rechazarse/sanitizarse.

## 386. Test — Magic Link Scanner

Automated GET no consume credential cuando flow usa explicit consumption.

## 387. Test — Replay

Magic link consumido no funciona nuevamente.

## 388. Test — Push Fatigue

Exceso de push genera throttle/risk signal.

## 389. Test — Contact Change

Nuevo recovery email no puede utilizarse inmediatamente si cooldown aplica.

## 390. Test — Old Contact Alert

Cambio crítico notifica canal anterior cuando policy lo requiere.

## 391. Test — Compromised Contact

Canal marcado compromised no participa en recovery.

## 392. Test — Stale Security Epoch

Queued sensitive communication se invalida.

## 393. Test — Duplicate Worker

Dos workers no producen dos logical deliveries cuando puede evitarse.

## 394. Test — Provider Ambiguity

Crash después de provider acceptance se reconcilia.

## 395. Test — Privacy

OTP nunca aparece en:
logs
traces
metrics
audit
DLQ metadata

## 396. Test — FrankenPHP

Request A no contamina:
tenant
recipient
locale
provider
branding
de Request B.

## 397. Test — Fiber

Dos concurrent communications mantienen contextos aislados.

## 398. Security Invariants — Communication

AUTH-COMM-01
Authentication Events y Communications serán dominios separados.
AUTH-COMM-02
Toda comunicación tendrá purpose.
AUTH-COMM-03
Toda comunicación tendrá security classification.
AUTH-COMM-04
Mandatory security notifications no serán deshabilitables mediante preferencias normales.
AUTH-COMM-05
Notification delivery no constituirá Authentication proof.
AUTH-COMM-06
Communication policy se evaluará antes de dispatch.

## 399. Security Invariants — OOB

AUTH-COMM-OOB-01
OOB no implicará automáticamente strong MFA.
AUTH-COMM-OOB-02
OTP será short-lived.
AUTH-COMM-OOB-03
OTP será single-use.
AUTH-COMM-OOB-04
OTP tendrá attempt limits.
AUTH-COMM-OOB-05
OTP estará purpose-bound.
AUTH-COMM-OOB-06
OTP no aparecerá en telemetry.
AUTH-COMM-OOB-07
Resend seguirá generation policy.
AUTH-COMM-OOB-08
Challenge expiration será controlada por Authentication, no provider.

## 400. Security Invariants — Links

AUTH-COMM-LINK-01
Magic links serán high entropy.
AUTH-COMM-LINK-02
Magic links serán short-lived.
AUTH-COMM-LINK-03
Magic links serán single-use.
AUTH-COMM-LINK-04
Hosts serán allowlisted.
AUTH-COMM-LINK-05
Open redirects estarán prohibidos.
AUTH-COMM-LINK-06
Third-party analytics estarán prohibidos por default en token landing pages.
AUTH-COMM-LINK-07
Automated GET/prefetch será considerado en consumption design.

## 401. Security Invariants — Recipients

AUTH-COMM-REC-01
Contact points tendrán verification state.
AUTH-COMM-REC-02
Verified contact != universal recovery channel.
AUTH-COMM-REC-03
Security contact changes podrán requerir fresh authentication.
AUTH-COMM-REC-04
Compromised contacts podrán excluirse de recovery.
AUTH-COMM-REC-05
Contact points se mostrarán redactados donde corresponda.

## 402. Security Invariants — Providers

AUTH-COMM-PROV-01
Provider != Channel.
AUTH-COMM-PROV-02
Provider selection será policy-driven.
AUTH-COMM-PROV-03
Failover respetará privacy/residency.
AUTH-COMM-PROV-04
Provider credentials seguirán Secret Management.
AUTH-COMM-PROV-05
Tenant provider credentials estarán aisladas.
AUTH-COMM-PROV-06
Provider debug logs serán sanitizados.

## 403. Security Invariants — Delivery

AUTH-COMM-DEL-01
Delivery será idempotent-aware.
AUTH-COMM-DEL-02
Retries serán bounded.
AUTH-COMM-DEL-03
Retries respetarán message/challenge expiration.
AUTH-COMM-DEL-04
Queued Authentication-bearing messages serán revalidados antes de dispatch.
AUTH-COMM-DEL-05
Provider delivery receipt no probará human receipt.
AUTH-COMM-DEL-06
Duplicate delivery será observable.

## 404. Security Invariants — Privacy

AUTH-COMM-PRIV-01
Provider payload será minimizado.
AUTH-COMM-PRIV-02
Message bodies sensibles no se loguearán.
AUTH-COMM-PRIV-03
Recipient PII no aparecerá en metrics.
AUTH-COMM-PRIV-04
Recipient PII no aparecerá en traces por default.
AUTH-COMM-PRIV-05
Provider tags no contendrán security-sensitive metadata innecesaria.
AUTH-COMM-PRIV-06
Communication retention seguirá documento 42.

## 405. Security Invariants — Runtime

AUTH-COMM-RT-01
Recipient context será request/job scoped.
AUTH-COMM-RT-02
Tenant context será explícito.
AUTH-COMM-RT-03
Branding será scope-safe.
AUTH-COMM-RT-04
Locale será scope-safe.
AUTH-COMM-RT-05
Provider credentials no estarán en mutable global context.
AUTH-COMM-RT-06
FrankenPHP worker reset será obligatorio.
AUTH-COMM-RT-07
Fiber contexts estarán aislados.

## 406. Anti-Patterns

No:
Mail::to($user->email)->send(
    new PasswordChangedMail($user)
);
directamente desde security-critical domain mutation.

## 407. Anti-Pattern

Authentication Event
→ Provider SDK
sin policy/outbox/governance.

## 408. Anti-Pattern

SMS delivered
→ MFA satisfied

## 409. Anti-Pattern

Verified email
→ automatically recovery-capable forever

## 410. Anti-Pattern

OTP resend
→ unlimited active OTPs

## 411. Anti-Pattern

Magic link
→ long-lived reusable bearer credential

## 412. Anti-Pattern

GET request from email scanner
→ permanently consume recovery credential
sin considerar scanner behavior.

## 413. Anti-Pattern

Provider failure
→ send through arbitrary foreign provider

## 414. Anti-Pattern

User disabled notifications
→ suppress password-change alert
cuando mandatory policy lo exige.

## 415. Anti-Pattern

Tenant branding
→ arbitrary HTML

## 416. Anti-Pattern

Email subject:
OTP: 482913
cuando provider/logging infrastructure puede exponer subjects.

## 417. Anti-Pattern

logger($mailBody);

## 418. Anti-Pattern

Queue serialized with raw magic token
sin necesidad.

## 419. Anti-Pattern

static $currentTenantMailer
en FrankenPHP.

## 420. Anti-Pattern

Retry forever

## 421. Anti-Pattern

Webhook failed
→ webhook event creates another webhook event
sin loop protection.

## 422. Failure Taxonomy

AUTH_COMMUNICATION_POLICY_DENIED
AUTH_COMMUNICATION_POLICY_CONFLICT
AUTH_COMMUNICATION_RECIPIENT_UNAVAILABLE
AUTH_COMMUNICATION_RECIPIENT_UNVERIFIED
AUTH_COMMUNICATION_RECIPIENT_COMPROMISED
AUTH_COMMUNICATION_CHANNEL_UNAVAILABLE
AUTH_COMMUNICATION_CHANNEL_FORBIDDEN
AUTH_COMMUNICATION_PROVIDER_UNAVAILABLE
AUTH_COMMUNICATION_PROVIDER_REJECTED
AUTH_COMMUNICATION_PROVIDER_TIMEOUT
AUTH_COMMUNICATION_DELIVERY_FAILED
AUTH_COMMUNICATION_DELIVERY_EXPIRED
AUTH_COMMUNICATION_DELIVERY_DUPLICATE
AUTH_COMMUNICATION_DELIVERY_CANCELLED
AUTH_COMMUNICATION_RATE_LIMITED
AUTH_COMMUNICATION_SUPPRESSED
AUTH_COMMUNICATION_TEMPLATE_INVALID
AUTH_COMMUNICATION_TEMPLATE_VARIABLE_INVALID
AUTH_COMMUNICATION_UNSAFE_LINK
AUTH_COMMUNICATION_PRIVACY_DENIED
AUTH_COMMUNICATION_RESIDENCY_DENIED
AUTH_COMMUNICATION_RETRY_EXHAUSTED
AUTH_COMMUNICATION_FAILOVER_UNAVAILABLE

AUTH_OOB_CHALLENGE_EXPIRED
AUTH_OOB_CHALLENGE_CONSUMED
AUTH_OOB_CHALLENGE_ATTEMPTS_EXCEEDED
AUTH_OOB_CHALLENGE_SUPERSEDED
AUTH_OOB_CHALLENGE_TRANSACTION_MISMATCH
AUTH_OOB_CHALLENGE_PURPOSE_MISMATCH

AUTH_MAGIC_LINK_EXPIRED
AUTH_MAGIC_LINK_CONSUMED
AUTH_MAGIC_LINK_INVALID
AUTH_MAGIC_LINK_TRANSACTION_MISMATCH

AUTH_PUSH_CHALLENGE_DENIED
AUTH_PUSH_CHALLENGE_EXPIRED
AUTH_PUSH_CHALLENGE_RATE_LIMITED

## 423. Public Failure Normalization

No revelar:
phone belongs to account
email exists
provider rejected specific destination
recovery channel unavailable because compromised
a callers no autorizados.

## 424. Extensibility

Plugins podrán agregar:
Communication Channels
Delivery Providers
Recipient Resolvers
Channel Selection Policies
Templates
Renderers
Retry Policies
Escalation Policies
Delivery Receipt Parsers
Webhook Signature Strategies

## 425. Plugin Boundary

Plugins no podrán saltarse:
Communication Policy
Privacy
Tenant Isolation
Secret Redaction
Rate Limits
Audit

## 426. Channel Plugin

interface AuthenticationCommunicationChannelProviderInterface
{
    public function register(
        AuthenticationCommunicationChannelRegistryInterface $registry
    ): void;
}

## 427. Provider Plugin

interface AuthenticationDeliveryProviderPluginInterface
{
    public function capabilities(): AuthenticationDeliveryProviderCapabilities;
}

## 428. Template Plugin

Debe declarar:
template ID
variables
channels
security classification

## 429. Namespace

Namespace recomendado:
VoltStack\Quantum\Auth\Communication

## 430. Estructura sugerida

src/Quantum/Auth/Communication/
├── Contracts/
│   ├── AuthenticationCommunicationPolicyEngineInterface.php
│   ├── AuthenticationCommunicationChannelInterface.php
│   ├── AuthenticationCommunicationChannelRegistryInterface.php
│   ├── AuthenticationCommunicationChannelSelectorInterface.php
│   ├── AuthenticationDeliveryProviderInterface.php
│   ├── AuthenticationDeliveryProviderRegistryInterface.php
│   ├── AuthenticationDeliveryProviderResolverInterface.php
│   ├── AuthenticationCommunicationRecipientResolverInterface.php
│   ├── AuthenticationContactPointPolicyInterface.php
│   ├── AuthenticationCommunicationComposerInterface.php
│   ├── AuthenticationCommunicationOutboxInterface.php
│   ├── AuthenticationDeliveryRetryPolicyInterface.php
│   ├── AuthenticationCommunicationEscalationPolicyInterface.php
│   ├── AuthenticationCommunicationDeduplicatorInterface.php
│   ├── AuthenticationDeliveryLeaseStoreInterface.php
│   ├── AuthenticationDeliveryReconcilerInterface.php
│   └── AuthenticationCommunicationLinkPolicyInterface.php
│
├── Intent/
│   ├── AuthenticationCommunicationIntent.php
│   ├── AuthenticationCommunicationType.php
│   ├── AuthenticationCommunicationPurpose.php
│   ├── AuthenticationCommunicationPriority.php
│   └── AuthenticationCommunicationOrigin.php
│
├── Recipient/
│   ├── AuthenticationCommunicationRecipient.php
│   ├── AuthenticationCommunicationRecipientSet.php
│   ├── AuthenticationCommunicationRecipientType.php
│   └── AuthenticationCommunicationRecipientResolver.php
│
├── Contact/
│   ├── AuthenticationContactPoint.php
│   ├── AuthenticationContactPointId.php
│   ├── AuthenticationContactPointStatus.php
│   ├── AuthenticationContactSecurityState.php
│   ├── AuthenticationContactPointPolicy.php
│   └── AuthenticationContactActivationPolicy.php
│
├── Channel/
│   ├── AuthenticationCommunicationChannelId.php
│   ├── AuthenticationChannelAssurance.php
│   ├── AuthenticationCommunicationChannelRegistry.php
│   └── AuthenticationCommunicationChannelSelector.php
│
├── Oob/
│   ├── AuthenticationOutOfBandChannel.php
│   ├── AuthenticationOutOfBandChallenge.php
│   ├── Otp/
│   ├── MagicLink/
│   └── Push/
│
├── Message/
│   ├── AuthenticationCommunicationMessage.php
│   ├── AuthenticationCommunicationBody.php
│   ├── AuthenticationCommunicationSubject.php
│   └── AuthenticationCommunicationActionSet.php
│
├── Template/
│   ├── AuthenticationCommunicationTemplate.php
│   ├── AuthenticationCommunicationTemplateId.php
│   ├── AuthenticationCommunicationTemplateVersion.php
│   ├── AuthenticationCommunicationTemplateSchema.php
│   └── AuthenticationCommunicationTemplateRegistry.php
│
├── Rendering/
│   ├── HtmlRenderer.php
│   ├── TextRenderer.php
│   ├── SmsRenderer.php
│   ├── PushRenderer.php
│   └── WebhookRenderer.php
│
├── Branding/
│   ├── AuthenticationCommunicationBranding.php
│   ├── AuthenticationSenderDomain.php
│   └── AuthenticationSenderDomainRegistry.php
│
├── Provider/
│   ├── AuthenticationDeliveryProvider.php
│   ├── AuthenticationDeliveryProviderRegistry.php
│   ├── AuthenticationDeliveryProviderResolver.php
│   ├── AuthenticationDeliveryProviderCapabilities.php
│   └── AuthenticationDeliveryProviderHealth.php
│
├── Delivery/
│   ├── AuthenticationDeliveryId.php
│   ├── AuthenticationDeliveryAttempt.php
│   ├── AuthenticationDeliveryReceipt.php
│   ├── AuthenticationDeliveryResult.php
│   ├── AuthenticationDeliveryRetryPolicy.php
│   ├── AuthenticationDeliveryLeaseStore.php
│   └── AuthenticationDeliveryReconciler.php
│
├── Outbox/
│   ├── AuthenticationCommunicationEnvelope.php
│   ├── AuthenticationCommunicationOutbox.php
│   └── AuthenticationCommunicationOutboxWorker.php
│
├── Policy/
│   ├── AuthenticationCommunicationPolicy.php
│   ├── AuthenticationCommunicationPolicyEngine.php
│   ├── AuthenticationCommunicationPolicyCompiler.php
│   ├── CompiledAuthenticationCommunicationPolicySet.php
│   └── AuthenticationCommunicationPolicyVersion.php
│
├── Governance/
│   ├── AuthenticationCommunicationGovernanceDefinition.php
│   ├── AuthenticationCommunicationGovernanceRegistry.php
│   └── AuthenticationCommunicationGovernanceScanner.php
│
├── Notification/
│   ├── AuthenticationSecurityNotificationCatalog.php
│   ├── AuthenticationSecurityNotificationDefinition.php
│   └── AuthenticationSecurityInboxMessage.php
│
├── Webhook/
│   ├── AuthenticationWebhookChannel.php
│   ├── AuthenticationWebhookSigner.php
│   └── AuthenticationWebhookReplayProtection.php
│
├── Runtime/
│   ├── AuthenticationCommunicationRuntimeContext.php
│   ├── AuthenticationDeliveryScope.php
│   └── AuthenticationCommunicationRuntimeResetter.php
│
├── Events/
├── Exceptions/
└── Testing/

## 431. Configuración conceptual

return [

    'communication' => [

        'security_notifications' => [
            'password_changed' => [
                'mandatory' => true,
                'channels' => ['email', 'in_app'],
            ],

            'mfa_disabled' => [
                'mandatory' => true,
                'channels' => ['email', 'push', 'in_app'],
            ],

            'new_login' => [
                'channels' => ['push', 'email'],
            ],
        ],

        'otp' => [
            'ttl' => '5 minutes',
            'max_attempts' => 5,
            'resend_cooldown' => '30 seconds',
            'supersede_previous' => true,
        ],

        'magic_links' => [
            'ttl' => '10 minutes',
            'single_use' => true,
            'clean_redirect_after_consumption' => true,
        ],

        'delivery' => [
            'max_attempts' => 5,
            'backoff' => 'exponential',
            'jitter' => true,
        ],

        'privacy' => [
            'log_message_body' => false,
            'trace_recipient' => false,
        ],

    ],

];

## 432. Developer Experience

Una aplicación no debería necesitar:
Mail::send(...);
Sms::send(...);
Push::send(...);
para security events.
Podrá utilizar:
Auth::communications()
    ->security()
    ->notify(
        SecurityNotification::passwordChanged($identity)
    );

## 433. OOB Challenge DX

Conceptualmente:
$challenge = Auth::challenges()->create(
    purpose: ChallengePurpose::Login,
    identity: $identity,
);

Auth::communications()
    ->oob()
    ->deliver($challenge);

## 434. Communication Intent DX

Auth::communications()->dispatch(
    AuthenticationCommunicationIntent::securityAlert(
        identity: $identity,
        type: SecurityAlertType::NewLogin,
    )
);

## 435. Domain Event Integration

Preferido:
PasswordChanged
       ↓
Security Communication Policy
       ↓
Communication Intent
sin que password service conozca mail/SMS.

## 436. Laravel Comparison

Laravel dispone de primitives potentes mediante:
Notifications
Mail
Queues
Events
Rate Limiting
Cache
Broadcasting
y ecosistema para SMS/push.
Sin embargo, normalmente la aplicación debe decidir por sí misma:
which security events are mandatory
OOB assurance
OTP lifecycle
magic-link scanner behavior
channel compromise
recipient cooldown
security-aware failover
communication policy composition
tenant security communication governance
VoltStack formaliza esas responsabilidades dentro del Authentication architecture.

## 437. Symfony Comparison

Symfony proporciona:
Notifier
Mailer
Messenger
EventDispatcher
RateLimiter
Translation
Secrets
con transports robustos.
VoltStack puede aprovechar ideas similares de transport abstraction, pero añade un dominio específicamente diseñado para Authentication:
Authentication Communication Intent
Security Classification
OOB Challenge Governance
Channel Assurance
Contact Security State
Mandatory Security Notifications
Provider Governance
Delivery Lifecycle
Security-Aware Retry
Security-Aware Failover

## 438. Diferenciador VoltStack

Laravel-like Notification DX
+
Symfony-like Transport Abstraction
+
Authentication-Specific Communication Semantics
+
OOB Challenge Governance
+
OTP Lifecycle
+
Magic Link Security
+
Push Fatigue Protection
+
Contact Security State
+
Security Notification Catalog
+
Mandatory Alert Policies
+
Provider Failover Governance
+
Privacy-Aware Delivery
+
Multi-Tenant Provider Isolation
+
Distributed Outbox
+
Idempotent Delivery
+
FrankenPHP / Fiber Safety

## 439. Decisiones arquitectónicas definitivas

VoltStack adoptará:

1. Authentication Event != Communication.
2. Communication Intent será first-class.
3. Toda comunicación tendrá purpose.
4. Toda comunicación tendrá security classification.
5. Informational y Authentication-bearing messages serán distintos.
6. OOB no significará automáticamente strong MFA.
7. Channel Assurance será explícito.
8. Provider != Channel.
9. Provider resolution será policy-driven.
10. Provider failover respetará privacy/residency.
11. Security communications podrán ser mandatory.
12. User preference no podrá deshabilitar automáticamente mandatory alerts.
13. Recipient resolution será first-class.
14. Contact points tendrán lifecycle.
15. Verified contact != recovery-capable contact.
16. Security contact changes serán sensitive operations.
17. Recovery cooldown será soportado.
18. Compromised contacts podrán ser bloqueados.
19. Channel selection combinará security/policy/preference/availability.
20. Challenge Negotiation consumirá channel capabilities.
21. OOB challenge estará ligado a Authentication Transaction.
22. OTP será short-lived.
23. OTP será single-use.
24. OTP tendrá attempt limits.
25. OTP tendrá resend governance.
26. Nueva generación podrá invalidar anterior.
27. SMS no será phishing-resistant.
28. Email OTP y email verification serán propósitos distintos.
29. Magic links serán bearer credentials temporales.
30. Magic links serán high-entropy.
31. Magic links serán single-use.
32. Magic links tendrán safe host policy.
33. Open redirects estarán prohibidos.
34. Link scanners serán parte del threat model.
35. Recovery no revelará account existence.
36. Security Notification Catalog será first-class.
37. Critical notifications podrán usar múltiples channels.
38. Message composition estará separada del domain event.
39. Templates tendrán IDs estables.
40. Templates podrán versionarse.
41. Template variables estarán allowlisted.
42. Tenant branding será sanitized.
43. Authentication messages seguirán anti-phishing rules.
44. Security links usarán trusted hosts.
45. Tracking estará deshabilitado por default en critical links.
46. Push fatigue tendrá mitigaciones.
47. In-App/Security Center serán first-class channels.
48. Webhooks tendrán signing/replay protection.
49. Communication ID != Delivery ID.
50. Una comunicación podrá tener múltiples deliveries.
51. Delivery receipts no demostrarán human receipt.
52. Delivery será principalmente async.
53. Transactional Outbox será recomendado/default.
54. Queue envelopes serán secret-minimized.
55. Sensitive queued messages serán revalidados antes de dispatch.
56. Security epochs podrán invalidar queued communications.
57. Retry será bounded.
58. Retry será expiration-aware.
59. Failover será governed.
60. Idempotency será first-class.
61. Deduplication será first-class.
62. Rate limiting protegerá identidad/destination/tenant/channel.
63. Cost abuse será considerado.
64. Provider health será observable.
65. Circuit breakers serán soportados.
66. Provider secrets se integrarán con doc31.
67. Tenant providers estarán aislados.
68. Async tenant context será explícito.
69. Realm-specific communication será soportada.
70. Break-glass generará immediate communication.
71. Break-glass actor no podrá silenciar alerta crítica.
72. Machine identity notifications tendrán recipients apropiados.
73. Communication governance será declarativa.
74. Policy composition no será last-write-wins.
75. Required channels normalmente se unirán.
76. Forbidden channels se unirán.
77. Allowed channels se intersectarán.
78. Contradicciones serán errores.
79. Policies serán compilables/versionadas.
80. Message body sensible no se logueará.
81. Recipient PII no estará en metrics/traces por default.
82. Delivery audit no guardará secrets.
83. Escalation será bounded.
84. Generic Notification module no decidirá Authentication semantics.
85. Runtime contexts serán request/job/fiber scoped.
86. FrankenPHP no mantendrá recipient/tenant/branding entre requests.
87. Distributed delivery asumirá at-least-once.
88. Provider ambiguity tendrá reconciliation.
89. Multi-region delivery respetará residency.
90. Expired messages no serán enviados.
91. Criterios de aceptación
El sistema se considerará completo cuando implemente al menos:
92. Communication Intent.
93. Communication Type.
94. Communication Purpose.
95. Security Classification.
96. Communication Priority.
97. Communication Origin.
98. Communication Policy Engine.
99. Communication Policy Compiler.
100. Policy Version.
101. Policy composition.
102. Conflict detection.
103. Recipient Resolver.
104. Recipient Types.
105. Contact Points.
106. Contact verification.
107. Contact security state.
108. Contact purpose binding.
109. Contact cooldown.
110. Channel Registry.
111. Channel Assurance.
112. Channel Selector.
113. OOB Channels.
114. OTP generation.
115. OTP verification.
116. OTP single-use.
117. OTP attempts.
118. OTP resend.
119. OTP supersession.
120. SMS support.
121. Email OTP support.
122. Magic Links.
123. Magic Link verifier storage.
124. Magic Link scanner protection.
125. Safe redirect.
126. Push challenges.
127. Number matching extensibility.
128. Push fatigue protection.
129. Recovery communication.
130. Enumeration resistance.
131. Security Notification Catalog.
132. Mandatory notifications.
133. Message Composer.
134. Structured Messages.
135. Template Registry.
136. Template Versions.
137. Variable schemas.
138. Localization.
139. Tenant Branding.
140. Anti-phishing rules.
141. Link Policy.
142. Trusted hosts.
143. In-App channel.
144. Security Center inbox.
145. Webhooks.
146. Webhook signatures.
147. Webhook replay protection.
148. Communication IDs.
149. Delivery IDs.
150. Delivery Attempts.
151. Delivery Receipts.
152. Delivery lifecycle.
153. Transactional Outbox.
154. Secret-minimized envelopes.
155. Pre-dispatch validation.
156. Communication cancellation.
157. Security epoch validation.
158. Retry Policy.
159. Exponential backoff.
160. Jitter.
161. Retry budgets.
162. Expiration-aware retries.
163. Provider failover.
164. Idempotency.
165. Deduplication.
166. Rate limiting.
167. Cost abuse protection.
168. Backpressure.
169. Priority queues.
170. Circuit breakers.
171. Provider health.
172. Provider Registry.
173. Provider Resolver.
174. Provider credential management.
175. Tenant provider isolation.
176. Realm provider isolation.
177. Break-glass notifications.
178. Machine security notifications.
179. Credential expiration alerts.
180. Governance Registry.
181. Governance Scanner.
182. Privacy integration.
183. Data residency integration.
184. Safe delivery logging.
185. Metrics.
186. Tracing.
187. Audit.
188. Escalation.
189. Loop prevention.
190. DLQ governance.
191. Manual replay validation.
192. Delivery leases.
193. Distributed workers.
194. Reconciliation.
195. Multi-region routing.
196. Clock abstraction.
197. Message expiry.
198. FrankenPHP reset.
199. Fiber isolation.
200. Tenant isolation tests.
201. OOB adversarial tests.
202. Arquitectura final
┌──────────────────────────────────────────────────────────────────────┐
│              AUTHENTICATION SECURITY COMMUNICATION                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Authentication / Security Event                                      │
│               │                                                      │
│               ▼                                                      │
│      Communication Intent                                            │
│               │                                                      │
│               ▼                                                      │
│      Communication Policy                                            │
│               │                                                      │
│       ┌───────┴────────┐                                             │
│       ▼                ▼                                             │
│   SUPPRESS           SEND                                            │
│                        │                                             │
│                        ▼                                             │
│               Recipient Resolver                                     │
│                        │                                             │
│                        ▼                                             │
│                 Channel Selector                                     │
│                        │                                             │
│          ┌─────────────┼─────────────┐                               │
│          ▼             ▼             ▼                               │
│        Email          SMS           Push                              │
│          │             │             │                               │
│          └─────────────┼─────────────┘                               │
│                        ▼                                             │
│                 Message Composer                                     │
│                        │                                             │
│                        ▼                                             │
│                 Security Template                                    │
│                        │                                             │
│                        ▼                                             │
│                Communication Outbox                                  │
│                        │                                             │
│                        ▼                                             │
│                  Delivery Worker                                     │
│                        │                                             │
│                        ▼                                             │
│                 Provider Resolver                                    │
│                        │                                             │
│             ┌──────────┴──────────┐                                  │
│             ▼                     ▼                                  │
│       Primary Provider      Fallback Provider                         │
│             │                     │                                  │
│             └──────────┬──────────┘                                  │
│                        ▼                                             │
│                  Delivery Result                                     │
│                        │                                             │
│             ┌──────────┼──────────┐                                  │
│             ▼          ▼          ▼                                  │
│          Receipt     Retry      Escalation                            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
203. Arquitectura OOB
Authentication Transaction
          │
          ▼
Challenge Negotiation
          │
          ▼
OOB Challenge
          │
          ▼
Channel Policy
          │
     ┌────┼─────┐
     ▼    ▼     ▼
   Email SMS   Push
     │    │     │
     └────┼─────┘
          ▼
Temporary Authentication Evidence
          │
          ▼
Verify
          │
          ▼
Atomic Consume
          │
          ▼
Authentication Context Updated
La comunicación únicamente transporta el challenge.
La autoridad sigue perteneciendo al Authentication system.
204. Integración 36–43
Con este documento, la capa transversal queda:
36 Policy Engine
      │
      ▼
37 Assurance / Context / Trust
      │
      ▼
38 Challenge Negotiation
      │
      ▼
39 Transaction Integrity
      │
      ▼
40 Compromise / Incident Response
      │
      ▼
41 Identity Lifecycle
      │
      ▼
42 Privacy / Data Governance
      │
      ▼
43 Security Communication / OOB Delivery
Y las relaciones más importantes son:
                 36 POLICY
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
     37 Assurance  38 Flow   43 Communication
                       │             │
                       ▼             ▼
                 39 Transaction ← OOB Challenge
                       │
                       ▼
                 Authentication
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     40 Incident   41 Lifecycle   35 Security Center
          │            │            │
          └────────────┼────────────┘
                       ▼
                 43 Notifications
                       │
                       ▼
                 42 Privacy
205. Regla arquitectónica final
La regla central será:
Una comunicación de Authentication nunca será considerada un simple email, SMS o push. Será una operación de seguridad con propósito, clasificación, destinatario, canal, política, lifecycle y evidencia de entrega explícitos.

Especialmente:
OTP
Magic Link
Recovery Link
Push Approval
deberán considerarse:
Temporary Security Artifacts
y no simples mensajes.
La arquitectura final buscará:
Authentication Security
        +
Reliable Delivery
        +
Anti-Phishing
        +
Replay Protection
        +
Privacy
        +
Multi-Tenant Isolation
        +
Provider Resilience
        +
Operational Observability
        +
Developer Experience
sin permitir que la capa de transporte se convierta en una vía para debilitar las garantías del Authentication Core.

## 445. Siguiente documento

El siguiente documento de la serie será:
44_AUTHENTICATION_BACKGROUND_PROCESSING_ASYNC_SECURITY_TASK_MAINTENANCE_CLEANUP_AND_SCHEDULED_OPERATION_SYSTEM.md
Su responsabilidad será formalizar toda la infraestructura asíncrona que hasta ahora hemos ido necesitando transversalmente:
Authentication Async Jobs
Security Maintenance
Expired Session Cleanup
Expired Challenge Cleanup
Credential Maintenance
Key/Credential Rotation Jobs
Retention Cleanup
Communication Delivery
Security Notification Processing
Risk Re-Evaluation
Identity Lifecycle Maintenance
Recovery Cleanup
Nonce Cleanup
Revocation Propagation
Projection Rebuilding
Scheduled Security Scans
Distributed Locks
Job Idempotency
Retries
Dead-Letter Queues
Job Security Context
Multi-Tenant Queue Isolation
FrankenPHP Workers
La separación quedará:
Authentication Domain
        │
        │ decides WHAT must happen
        ▼
44 Background Processing
        │
        │ decides HOW deferred work executes
        ▼
Queue / Scheduler / Worker / Distributed Runtime
y será especialmente importante para que VoltStack no termine acoplando Authentication directamente a un sistema de colas específico.
