# VoltStack Authentication System

## 42 — Authentication Privacy, Data Minimization, Retention, Consent and Security Metadata Governance System

- **Archivo:** `42_AUTHENTICATION_PRIVACY_DATA_MINIMIZATION_RETENTION_CONSENT_AND_SECURITY_METADATA_GOVERNANCE_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Clasificación:** Security-Critical / Privacy / Data Governance / Authentication Metadata
- **Dependencias principales:** 09, 12, 13, 17, 18, 20, 21, 23, 24, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41.

---

## 1. Propósito

Este documento define el sistema responsable de gobernar qué información puede recolectar, procesar, almacenar, exponer, conservar, anonimizar y eliminar el subsistema de Authentication de VoltStack.
Authentication inevitablemente procesa información de alto valor de seguridad:
Identity identifiers
Email addresses
Phone numbers
IP addresses
Device information
User-Agent
Authentication methods
Authentication timestamps
Failed login attempts
Session metadata
Passkey metadata
MFA metadata
Federated identities
Recovery information
Risk signals
Approximate locations
Security incidents
Credential metadata
Machine identity metadata
Audit evidence
Pero:
Que un dato sea útil para Authentication no significa que deba almacenarse indefinidamente.

VoltStack deberá aplicar explícitamente:
Purpose Limitation
Data Minimization
Storage Limitation
Access Control
Retention Governance
Redaction
Pseudonymization
Anonymization
Erasure
Security Evidence Preservation
Tenant Isolation
Auditability

## 2. Problema fundamental

Una implementación ingenua puede terminar acumulando:
every login IP forever
every User-Agent forever
every device fingerprint forever
every failed password attempt forever
every approximate location forever
every risk signal forever
every OAuth claim forever
every authentication trace forever
porque:
"Puede ser útil para seguridad."

Ese criterio no es suficiente.
El resultado puede convertirse en:
Authentication System
        ↓
Massive Security Metadata Store
        ↓
Permanent Behavioral History
        ↓
Privacy / Security Liability

## 3. Principio fundamental

Authentication deberá recolectar la mínima información necesaria para cumplir una finalidad de Authentication o seguridad explícitamente definida.

1. Segundo principio
Los datos de Authentication deberán tener propietario, propósito, clasificación, alcance, política de acceso, política de retención y estrategia de eliminación.

2. Tercer principio
Security telemetry is not exempt from data governance merely because it improves security.

3. Cuarto principio
Consentimiento tampoco será tratado como autorización universal.
Consent
   ≠
Legal Basis
   ≠
Authentication Requirement
   ≠
Authorization
   ≠
Security Necessity
VoltStack proporcionará primitives técnicas.
La aplicación determinará requisitos jurídicos concretos según jurisdicción y contexto.

## 4. Alcance

El sistema gobernará principalmente:
Authentication Identity Metadata
Session Metadata
Credential Metadata
Authentication Method Metadata
Device Metadata
Federation Metadata
Recovery Metadata
Authentication Activity
Authentication Audit Data
Risk Metadata
Security Alerts
Security Incident Metadata
Authentication Transaction Metadata
Machine Authentication Metadata
Authentication Telemetry
Authentication Security Center Projections

## 5. Fuera de alcance directo

No será el sistema general de privacidad de:
orders
payments
medical records
CRM
documents
messages
business analytics
marketing profiles
Estos dominios tendrán sus propias políticas.

## 6. Arquitectura conceptual

Authentication Data
        │
        ▼
Data Classification
        │
        ▼
Purpose Resolution
        │
        ▼
Collection Policy
        │
        ▼
Data Minimization
        │
        ▼
Processing
        │
        ├───────────────┐
        ▼               ▼
Operational Data   Security Evidence
        │               │
        ▼               ▼
Retention Policy   Evidence Policy
        │               │
        └───────┬───────┘
                ▼
        Lifecycle Engine
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
     Erase   Anonymize  Retain

## 7. Authentication Data Governance

Se introduce un dominio formal:
Authentication Data Governance
responsable de determinar cómo debe manejarse cada categoría de datos.

## 8. Authentication Data Asset

final readonly class AuthenticationDataAsset
{
    public function __construct(
        public AuthenticationDataType $type,
        public AuthenticationDataClassification $classification,
        public AuthenticationDataPurposeSet $purposes,
        public AuthenticationDataScope $scope,
        public AuthenticationRetentionPolicyId $retentionPolicy,
    ) {}
}

## 9. Data Asset != Data Value

AuthenticationDataAsset describe la clase de información.
Ejemplo:
LOGIN_SOURCE_IP
No contiene necesariamente:
203.0.113.10

## 10. Data Type Registry

VoltStack deberá mantener un catálogo de tipos de datos.
enum AuthenticationDataType: string
{
    case IdentityIdentifier = 'identity_identifier';
    case AuthenticationTimestamp = 'authentication_timestamp';
    case SourceIp = 'source_ip';
    case UserAgent = 'user_agent';
    case DeviceMetadata = 'device_metadata';
    case SessionMetadata = 'session_metadata';
    case AuthenticationMethodMetadata = 'authentication_method_metadata';
    case CredentialMetadata = 'credential_metadata';
    case FederationMetadata = 'federation_metadata';
    case RecoveryMetadata = 'recovery_metadata';
    case RiskMetadata = 'risk_metadata';
    case SecurityIncidentMetadata = 'security_incident_metadata';
    case AuditMetadata = 'audit_metadata';
}
El registry deberá ser extensible.

## 11. Data Classification

Cada dato deberá tener clasificación.
Modelo inicial:
enum AuthenticationDataClassification: string
{
    case Public = 'public';
    case Internal = 'internal';
    case Confidential = 'confidential';
    case Sensitive = 'sensitive';
    case Secret = 'secret';
    case CryptographicSecret = 'cryptographic_secret';
}

## 12. Public

Información que puede ser mostrada públicamente según aplicación.
Authentication deberá contener muy poca información de esta categoría.

## 13. Internal

Ejemplo:
Authentication method display name
internal policy identifier
non-sensitive provider name

## 14. Confidential

Ejemplo:
authentication history
session activity
device inventory
federated account relationships

## 15. Sensitive

Ejemplo:
IP addresses
approximate location
risk findings
security incident associations
device identifiers
recovery metadata

## 16. Secret

Ejemplo:
bearer tokens
recovery codes
API token plaintext
TOTP enrollment secret
OAuth refresh token

## 17. Cryptographic Secret

Ejemplo:
private keys
signing keys
client secrets
HMAC keys
Su governance principal se integra con documento 31.

## 18. Classification Is Not Enough

Dos datos Sensitive pueden tener políticas distintas.
Ejemplo:
Source IP
y:
Security Incident Evidence
pueden requerir retenciones diferentes.

## 19. Data Purpose

Cada dato deberá asociarse a una finalidad.
enum AuthenticationDataPurpose: string
{
    case Authentication = 'authentication';
    case SessionManagement = 'session_management';
    case CredentialManagement = 'credential_management';
    case FraudPrevention = 'fraud_prevention';
    case RiskAssessment = 'risk_assessment';
    case IncidentResponse = 'incident_response';
    case SecurityAudit = 'security_audit';
    case AccountRecovery = 'account_recovery';
    case UserSecurityManagement = 'user_security_management';
    case Compliance = 'compliance';
    case OperationalDiagnostics = 'operational_diagnostics';
}

## 20. Purpose Set

Un dato puede tener más de una finalidad legítima.
final readonly class AuthenticationDataPurposeSet
{
    /** @param list<AuthenticationDataPurpose> $purposes */
    public function __construct(
        public array $purposes
    ) {}
}

## 21. Purpose Limitation

Si IP fue recolectada para:
Fraud Prevention
no deberá convertirse automáticamente en:
Marketing Analytics
porque ya existe en la base de datos.

## 22. Cross-Purpose Access

Debe requerir una política explícita.

## 23. Purpose Binding

final readonly class AuthenticationDataPurposeBinding
{
    public function __construct(
        public AuthenticationDataType $dataType,
        public AuthenticationDataPurpose $purpose,
        public AuthenticationDataPolicyId $policy,
    ) {}
}

## 24. Collection Policy

Antes de almacenar un dato, deberá existir una decisión.
interface AuthenticationDataCollectionPolicyInterface
{
    public function evaluate(
        AuthenticationDataCollectionRequest $request
    ): AuthenticationDataCollectionDecision;
}

## 25. Collection Decision

enum AuthenticationDataCollectionDecisionType: string
{
    case Collect = 'collect';
    case CollectMinimized = 'collect_minimized';
    case CollectEphemeral = 'collect_ephemeral';
    case CollectPseudonymized = 'collect_pseudonymized';
    case DoNotCollect = 'do_not_collect';
}

## 26. Ephemeral Collection

Un dato puede utilizarse durante Authentication sin persistirse.
Ejemplo:
Raw network information
        ↓
Risk evaluation
        ↓
Risk category
        ↓
Raw information discarded

## 27. Important Distinction

Process
   ≠
Persist
VoltStack deberá poder procesar información sin convertirla automáticamente en historial permanente.

## 28. Data Minimization Engine

interface AuthenticationDataMinimizerInterface
{
    public function minimize(
        AuthenticationDataValue $value,
        AuthenticationDataMinimizationContext $context
    ): MinimizedAuthenticationData;
}

## 29. Minimization Strategies

enum AuthenticationDataMinimizationStrategy: string
{
    case None = 'none';
    case Drop = 'drop';
    case Truncate = 'truncate';
    case Generalize = 'generalize';
    case Mask = 'mask';
    case Hash = 'hash';
    case Hmac = 'hmac';
    case Tokenize = 'tokenize';
    case Pseudonymize = 'pseudonymize';
    case Aggregate = 'aggregate';
}

## 30. IP Address Minimization

No siempre es necesario conservar IP completa.
Podrá existir:
Full IP
Truncated IP
Network prefix
Country
Region
Risk category
HMAC(IP)
según finalidad.

## 31. Example

En lugar de conservar:
2001:db8:abcd:1234:5678:90ab:cdef:1234
indefinidamente, una política puede conservar posteriormente únicamente:
2001:db8:abcd::/48
o una clasificación geográfica suficientemente general.

## 32. Progressive Minimization

VoltStack deberá soportar reducción de precisión con el tiempo.
Ejemplo:
0–7 days
Full IP

8–30 days
Truncated IP

31–180 days
Country + Risk Classification

>180 days
Aggregated Statistics Only
 1. Progressive Retention
Esto evita el modelo binario:
keep everything
vs
delete everything
 2. User-Agent
No almacenar necesariamente User-Agent completo permanentemente.
Puede normalizarse:
Browser = Firefox
Browser Major = 151
OS = Windows
Device Class = Desktop
 3. Raw User-Agent
Puede mantenerse temporalmente para diagnóstico/security analysis.
 4. Device Fingerprinting
VoltStack no deberá asumir que fingerprinting invasivo es requisito de Authentication.
 5. Device Identity
Preferir identificadores explícitos y privacy-aware cuando sea posible.
 6. Device Metadata
Separar:
Device Management Identifier
Device Credential
Device Trust State
Presentation Metadata
Fingerprint Signals
 7. Device Name
Un nombre como:
Chrome on Windows
es presentation metadata.
No debe convertirse en trusted device identity.
 8. Approximate Location
Debe considerarse metadata sensible.
 9. Location Precision
Preferir precisión mínima necesaria:
Country
Region
City
antes que coordenadas exactas cuando Authentication no las requiera.
10. Precise Geolocation
No deberá formar parte del core Authentication por defecto.
11. Location Source
Debe conocerse:
IP-derived
device-provided
trusted network
external risk provider
porque su significado cambia.
12. Authentication Activity
Documento 35.
Security Center puede mostrar:
Login from Monterrey, Mexico
sin conservar necesariamente coordenadas.
13. Authentication Event Data
Documento 23.
Cada evento deberá declarar un esquema de datos permitido.
14. Event Data Contract
interface AuthenticationEventDataPolicyInterface
{
    public function allowedFields(
        AuthenticationEventType $event,
        AuthenticationEventContext $context
    ): AuthenticationEventFieldPolicySet;
}
15. Event Payload Minimization
No:
event(new LoginSucceeded(
    password: $password,
    token: $token,
    sessionCookie: $cookie,
));
Nunca.
16. Secret-Free Events
Regla:
Authentication domain events deberán ser secret-free salvo un contrato excepcional explícitamente diseñado para secret transport.

La implementación default no transportará secretos.

## 52. Logging

Documento 24 y Telemetry System.
Logs nunca deberán contener:
password
TOTP secret
recovery code
raw bearer token
refresh token
private key
client secret
session cookie
WebAuthn private material

## 53. Redaction Engine

interface AuthenticationDataRedactorInterface
{
    public function redact(
        mixed $value,
        AuthenticationDataRedactionContext $context
    ): mixed;
}

## 54. Central Redaction

La redacción no deberá depender de que cada developer recuerde:
unset($payload['password']);

## 55. Secret Detector

Puede existir defensa adicional para detectar campos conocidos:
password
secret
token
authorization
cookie
private_key
recovery_code
pero no sustituye esquemas seguros.

## 56. Allowlist > Denylist

Para eventos/audit/tracing críticos:
Preferir campos explícitamente permitidos sobre intentar detectar todos los secretos posibles.

 1. Structured Audit
Ejemplo seguro:
{
  "event": "authentication.login.succeeded",
  "method": "passkey",
  "assurance": "high",
  "tenant": "tenant-reference",
  "timestamp": "..."
}
No:
{
  "credential": "...",
  "cookie": "...",
  "token": "..."
}
 2. Authentication Audit
Audit tiene requisitos distintos de operational logs.
 3. Audit Purpose
Principalmente:
security accountability
incident reconstruction
administrative accountability
compliance
 4. Audit Data Policy
interface AuthenticationAuditDataPolicyInterface
{
    public function policyFor(
        AuthenticationAuditEventType $event
    ): AuthenticationAuditDataPolicy;
}
 5. Audit Minimization
Audit no significa "guardar todo".
Debe conservar:
who
what
when
scope
result
reason
security context
con la mínima información necesaria.
 6. Actor Reference
Preferir stable opaque identity reference sobre copiar:
full name
email
phone
address
en cada audit record.
 7. Snapshot Duplication
Evitar duplicar perfiles completos dentro del audit.
 8. Historical Display
La UI puede resolver display information cuando sea apropiado.
 9. Deleted Identity
Audit histórico puede conservar:
Identity ID 01J...
sin conservar necesariamente email original.
10. Audit Integrity
Data minimization no debe destruir la integridad necesaria para investigar eventos.
11. Security Evidence
Se introduce categoría conceptual:
Security Evidence
para datos retenidos específicamente por investigación de incidentes.
12. Evidence != Normal Telemetry
Cuando un incidente se confirma, cierta información puede pasar de:
Operational Security Metadata
a:
Incident Evidence
mediante proceso explícito.
13. Evidence Preservation
interface AuthenticationSecurityEvidencePolicyInterface
{
    public function preserve(
        AuthenticationSecurityEvidenceCandidate $candidate,
        SecurityIncidentReference $incident
    ): AuthenticationSecurityEvidenceDecision;
}
14. Evidence Hold
Un incidente puede aplicar hold a datos que normalmente expirarían.
15. Evidence Hold Scope
Debe ser limitado a:
specific incident
specific identities
specific artifacts
specific time window
specific data types
16. No Global Forever Hold
No:
security incident exists
→ retain all authentication data forever
17. Retention Policy
Cada data type deberá tener retention policy.
interface AuthenticationRetentionPolicyInterface
{
    public function retentionFor(
        AuthenticationDataAsset $asset,
        AuthenticationRetentionContext $context
    ): AuthenticationRetentionDecision;
}
18. Retention Decision
final readonly class AuthenticationRetentionDecision
{
    public function __construct(
        public DateInterval|null $retention,
        public AuthenticationRetentionAction $expirationAction,
        public bool $allowSecurityHold,
    ) {}
}
19. Expiration Actions
enum AuthenticationRetentionAction: string
{
    case Delete = 'delete';
    case Anonymize = 'anonymize';
    case Pseudonymize = 'pseudonymize';
    case Aggregate = 'aggregate';
    case Archive = 'archive';
    case Review = 'review';
}
20. No Universal Retention Duration
VoltStack no impondrá:
"All auth logs = 90 days"
como regla universal.
Depende de:
data type
purpose
tenant
environment
jurisdiction
incident state
contract
security requirements
21. Framework Floor
Sí podrá imponer máximos/mínimos técnicos para ciertas categorías cuando sean necesarios para seguridad.
22. Example Retention Configuration
return [

    'privacy' => [

        'retention' => [

            'authentication_activity' => '90 days',

            'failed_login_detail' => '30 days',

            'raw_user_agent' => '30 days',

            'full_ip_address' => '7 days',

            'normalized_device_metadata' => '180 days',

            'security_audit' => '1 year',

        ],

    ],

];
Solo ejemplo conceptual.

## 79. Retention Policy Hierarchy

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
Data Type
          ↓
Purpose

## 80. Tenant Retention

Tenant podrá definir políticas dentro de límites permitidos.

## 81. Tenant Cannot Disable Mandatory Security Evidence

Si platform policy exige determinada retención mínima para security-critical audit, tenant no podrá reducirla por debajo del floor.

## 82. Tenant Cannot Retain Indefinitely by Accident

También pueden existir máximos.
Platform maximum raw IP retention = X
Tenant asks FOREVER
→ rejected
si governance policy lo determina.

## 83. Retention Composition

Necesita semántica formal.
No:
last configuration wins

## 84. Minimum Retention Constraint

Puede provenir de:
security
compliance
legal hold

## 85. Maximum Retention Constraint

Puede provenir de:
privacy policy
tenant agreement
jurisdiction
data minimization

## 86. Conflict

Ejemplo:
minimum retention = 1 year
maximum retention = 90 days
Debe producir:
AUTH_PRIVACY_RETENTION_POLICY_CONFLICT
No escoger silenciosamente uno.

## 87. Retention Compiler

interface AuthenticationRetentionPolicyCompilerInterface
{
    public function compile(
        AuthenticationRetentionPolicySet $policies
    ): CompiledAuthenticationRetentionPolicySet;
}

## 88. Compilation

Validará:
unknown data types
invalid duration
minimum > maximum
unsupported action
tenant boundary violations
missing policy
illegal indefinite retention
según configuración.

## 89. Retention Version

final readonly class AuthenticationRetentionPolicyVersion
{
    public function __construct(
        public string $value
    ) {}
}

## 90. Retention Policy Changes

No necesariamente deben aplicarse retroactivamente sin análisis.

## 91. Example

Cambiar:
IP retention
90 days → 7 days
puede requerir cleanup inmediato de registros mayores a siete días.

## 92. Policy Migration

El sistema deberá generar un retention migration plan.

## 93. Retention Planner

interface AuthenticationRetentionPlannerInterface
{
    public function plan(
        AuthenticationRetentionPolicyChange $change
    ): AuthenticationRetentionMigrationPlan;
}

## 94. Retention Execution

Será preferentemente asynchronous.

## 95. Retention Worker

Retention Scheduler
       ↓
Find Expired Data
       ↓
Verify Holds
       ↓
Apply Action
       ↓
Delete / Minimize / Aggregate
       ↓
Audit Maintenance Operation

## 96. Retention Worker Idempotency

Debe soportar reintentos.

## 97. Retention Race

Antes de eliminar:
check current hold/version
para evitar borrar evidence que acaba de entrar en incident hold.

## 98. Atomic Hold Check

Cuando sea crítico deberá existir garantía transaccional/distribuida adecuada.

## 99. Data Lifecycle State

Opcionalmente:
enum AuthenticationDataLifecycleState: string
{
    case Active = 'active';
    case Expiring = 'expiring';
    case Held = 'held';
    case Archived = 'archived';
    case Anonymized = 'anonymized';
    case Deleted = 'deleted';
}

## 100. Avoid State Explosion

No todos los records necesitan columna lifecycle.
Puede calcularse mediante policy/metadata.

## 101. Consent

Consentimiento requiere modelado cuidadoso.

## 102. Authentication Does Not Depend Universally on Consent

Ciertas operaciones de seguridad pueden ser necesarias para prestar/proteger el servicio.
VoltStack no deberá implementar:
no consent → no brute-force protection
como regla general.

## 103. Consent Is Contextual

Consent puede ser relevante para capacidades opcionales como:
remember this device
persistent login
optional device recognition
optional security notifications
optional external identity linking
certain optional telemetry
según aplicación y regulación.

## 104. Consent Purpose

Debe ser específico.
No:
I consent to all authentication processing forever.

## 105. Consent Record

final readonly class AuthenticationConsentRecord
{
    public function __construct(
        public AuthenticationConsentId $id,
        public IdentityReference $identity,
        public AuthenticationConsentPurpose $purpose,
        public AuthenticationConsentPolicyVersion $policyVersion,
        public AuthenticationConsentStatus $status,
        public DateTimeImmutable $recordedAt,
        public DateTimeImmutable|null $withdrawnAt,
    ) {}
}

## 106. Consent Status

enum AuthenticationConsentStatus: string
{
    case Granted = 'granted';
    case Withdrawn = 'withdrawn';
    case Expired = 'expired';
    case Superseded = 'superseded';
}

## 107. Consent Versioning

Debe saberse a qué versión de policy correspondió.

## 108. Consent Text

El core no deberá depender del texto legal como identity del consentimiento.
Utilizar:
policyId
policyVersion
purpose

## 109. Withdrawal

Retirar consentimiento deberá producir efectos explícitos.

## 110. Example — Remember Device

Consent/Preference:
Remember this device

Withdraw
   ↓
Revoke Device Trust
   ↓
Remove optional recognition metadata
   ↓
Retain only mandatory security audit if required

## 111. Consent Withdrawal != Erase Everything

Si existen datos que deban conservarse por security/legal reasons:
withdraw consent
no implica necesariamente borrarlos.
Pero deberán dejar de utilizarse para el propósito basado en ese consentimiento.

## 112. Consent vs Preference

No toda elección de usuario es consentimiento jurídico.
VoltStack deberá distinguir conceptualmente:
Consent
Preference
Security Configuration
Authentication Enrollment

## 113. Example

Enable TOTP MFA
es principalmente security configuration/enrollment.
No debería llamarse genéricamente "consent".

## 114. Authentication Enrollment

Registrar passkey significa autorizar técnicamente su uso como Authentication Method.
Eso no debe confundirse automáticamente con consentimiento de privacy.

## 115. Consent Resolver

interface AuthenticationConsentResolverInterface
{
    public function resolve(
        IdentityReference $identity,
        AuthenticationConsentPurpose $purpose
    ): AuthenticationConsentDecision;
}

## 116. Consent Storage

No almacenar datos innecesarios como:
full IP
full browser fingerprint
solo para demostrar consentimiento si no son necesarios.

## 117. Consent Audit

Registrar:
identity
purpose
policy version
timestamp
action
normalmente será suficiente.

## 118. Consent in Multi-Tenant

Puede existir:
Global Consent
Tenant Consent
Application Consent
según finalidad.

## 119. Tenant Isolation

Tenant A no podrá consultar consent records de Tenant B salvo authority explícita.

## 120. Consent Revocation Propagation

En sistemas distribuidos, retirada de consentimiento deberá invalidar caches/projections relevantes.

## 121. Privacy Scope

final readonly class AuthenticationPrivacyScope
{
    public function __construct(
        public TenantId|null $tenant,
        public RealmId|null $realm,
        public ApplicationId|null $application,
        public EnvironmentId $environment,
    ) {}
}

## 122. Global vs Tenant Metadata

No mezclar accidentalmente:
Global identity authentication history
con:
Tenant-specific authentication activity

## 123. Tenant Visibility

Tenant administrators deberán ver únicamente metadata necesaria para administrar su tenant.

## 124. Example

Tenant admin puede necesitar:
last authentication time
tenant session status
tenant MFA compliance
pero no necesariamente:
other tenant sessions
global recovery methods
other organizations
personal security incidents unrelated to tenant

## 125. Viewer-Aware Projection

Documento 35.
interface AuthenticationPrivacyProjectionPolicyInterface
{
    public function project(
        AuthenticationSecurityData $data,
        AuthenticationPrivacyViewerContext $viewer
    ): AuthenticationPrivacyProjection;
}

## 126. Viewer Context

Considera:
self
tenant admin
security operator
platform admin
support
machine administrator

## 127. Administrability != Full Visibility

Regla:
Poder administrar una identidad no implica poder visualizar toda su metadata de seguridad.

  1. Support Personnel
Support no debería poder ver:
raw risk signals
full IP history
recovery secrets
token values
private credentials
solo porque atiende al usuario.
  2. Field-Level Visibility
Debe poder resolverse por campo.
  3. Example
User:
IP → 192.0.2.xxx

Security Operator:
IP → full value if authorized

Tenant Admin:
IP → country only
según policy.

## 131. Authentication Data Access Policy

interface AuthenticationDataAccessPolicyInterface
{
    public function authorize(
        AuthenticationDataAccessRequest $request
    ): AuthenticationDataAccessDecision;
}

## 132. Access Decision

Puede devolver:
ALLOW_FULL
ALLOW_REDACTED
ALLOW_AGGREGATED
DENY

## 133. Authentication Data Export

El sistema deberá poder producir exportaciones de metadata cuando una aplicación lo requiera.

## 134. Export != Database Dump

Nunca exponer directamente:
auth tables
session tables
credential tables

## 135. Safe Export

interface AuthenticationDataExporterInterface
{
    public function export(
        IdentityReference $identity,
        AuthenticationDataExportContext $context
    ): AuthenticationDataExport;
}

## 136. Export Redaction

Nunca incluir:
password hashes
TOTP secrets
recovery codes
session secrets
bearer tokens
private keys
secret verifiers where unsafe

## 137. Export Categories

Puede incluir:
authentication methods
active sessions metadata
devices
linked providers
security activity
consent records
security settings
según policy.

## 138. Machine Identity Export

Debe utilizar reglas distintas.

## 139. Data Subject Request

VoltStack puede ofrecer primitives para:
access
export
erasure
correction
restriction
sin declarar automáticamente requisitos legales universales.

## 140. Privacy Request

final readonly class AuthenticationPrivacyRequest
{
    public function __construct(
        public AuthenticationPrivacyRequestId $id,
        public AuthenticationPrivacyRequestType $type,
        public IdentityReference $subject,
        public ActorReference $actor,
        public DateTimeImmutable $createdAt,
    ) {}
}

## 141. Privacy Request Types

enum AuthenticationPrivacyRequestType: string
{
    case Access = 'access';
    case Export = 'export';
    case Erasure = 'erasure';
    case Restriction = 'restriction';
    case Correction = 'correction';
}

## 142. Identity Verification

Privacy requests themselves pueden ser sensitive operations.

## 143. Export Authentication Requirement

Una exportación puede requerir:
fresh authentication
MFA
passkey
según sensibilidad.

## 144. Erasure Request

Se integra con documento 41.

## 145. Privacy Erasure != Immediate Identity Deletion

Puede requerir lifecycle workflow.

## 146. Correction

Authentication metadata derivada no siempre puede "corregirse".
Ejemplo:
security event occurred at T
no debería editarse arbitrariamente.

## 147. Append Correction

Puede añadirse:
metadata correction
annotation
dispute
preservando audit integrity.

## 148. Security History Dispute

Usuario puede indicar:
This wasn't me
pero eso no elimina el evento.
Cambia su clasificación/incident response.

## 149. Security Metadata

Se define como información derivada o recolectada para proteger Authentication.

## 150. Examples

risk score
risk level
new device signal
impossible travel signal
credential compromise indicator
IP reputation
failed login velocity
authentication anomaly

## 151. Raw Signal vs Derived Finding

Separar:
Raw Security Signal
        ↓
Risk Processing
        ↓
Derived Security Finding

## 152. Derived Data Minimization

Cuando sea suficiente conservar:
Risk = HIGH
no siempre es necesario conservar todos los raw inputs.

## 153. Risk Score Retention

Un score sin contexto puede volverse inútil.
Debe conservar:
model/rule version
timestamp
purpose
relevant explanation references
cuando sea necesario para explainability.

## 154. Risk Model Version

final readonly class AuthenticationRiskEvaluationReference
{
    public function __construct(
        public string $engine,
        public string $version,
        public DateTimeImmutable $evaluatedAt,
    ) {}
}

## 155. Automated Security Decisions

Si risk metadata produce:
STEP_UP_REQUIRED
LOGIN_DENIED
ACCOUNT_PROTECTED
debe existir suficiente explainability para investigación.

## 156. Explainability vs Privacy

No conservar raw data ilimitadamente solo porque quizá se necesite explicar una decisión.
Preferir:
decision
reason codes
policy version
risk category
limited evidence references

## 157. Security Metadata Governance

interface AuthenticationSecurityMetadataGovernanceInterface
{
    public function govern(
        AuthenticationSecurityMetadata $metadata,
        AuthenticationSecurityMetadataContext $context
    ): AuthenticationSecurityMetadataDecision;
}

## 158. Decision

Puede definir:
store?
precision?
retention?
visibility?
purpose?
security hold eligibility?

## 159. Authentication Transaction Data

Documento 39.
Nonces, CSRF state, PKCE verifier, continuation tokens y transaction secrets tienen políticas especiales.

## 160. Transaction Secrets

Generalmente:
short-lived
single-use
purpose-bound
y deberán eliminarse rápidamente.

## 161. Nonce Retention

Después de consumo puede conservarse:
hash/reference + consumed timestamp
durante ventana mínima de replay protection si es necesario.
No el secret original.

## 162. CSRF Tokens

No deben almacenarse en logs/audit.

## 163. PKCE Verifier

Nunca debe persistirse más allá de la transacción cuando no sea necesario.

## 164. OAuth/OIDC Metadata

Documento 17.
Separar:
issuer
subject
provider
link metadata
de:
access token
refresh token
ID token raw value

## 165. Raw ID Token

No debería convertirse automáticamente en permanent identity record.
Extraer únicamente claims necesarios.

## 166. Claims Minimization

Si solo se requiere:
issuer
subject
email_verified
no persistir todos los claims recibidos.

## 167. Federation Claim Allowlist

interface FederatedClaimRetentionPolicyInterface
{
    public function retainedClaims(
        FederatedProviderReference $provider,
        AuthenticationPurpose $purpose
    ): FederatedClaimSet;
}

## 168. Provider Tokens

Access/refresh tokens deberán seguir secret storage policies.

## 169. Provider Profile Snapshots

No copiar perfil completo indefinidamente.

## 170. Passkey Metadata

Puede conservar:
credential management ID
public key
sign count where applicable
createdAt
lastUsedAt
transport hints
backup eligibility/state where protocol provides it
display name
según WebAuthn requirements.

## 171. Passkey Privacy

No exponer detalles innecesarios del authenticator a otros viewers.

## 172. Credential ID

WebAuthn credential identifiers son security-sensitive metadata.
No deberán aparecer indiscriminadamente en logs.

## 173. Password Metadata

Puede conservar:
algorithm
parameters
createdAt
lastChangedAt
needsRehash
pero no mostrar hash al Security Center.

## 174. Password Hash

Será secret authentication material.

## 175. Password History

Si se soporta, almacenar verificadores adecuados, nunca plaintext.

## 176. Password History Retention

No conservar indefinidamente sin policy.

## 177. MFA Metadata

Puede mostrar:
TOTP configured
created date
last used
pero nunca:
TOTP seed

## 178. Recovery Codes

Security Center:
8 codes remaining
No:
actual codes
después de ceremonia show-once.

## 179. Recovery Contact Data

Email/phone utilizados para recovery tienen finalidad y retention específicas.

## 180. Recovery Metadata Sensitivity

Alta.
Puede facilitar account takeover.

## 181. Session Metadata

Puede conservar:
session management ID
createdAt
lastActivity
device display
approximate location
authentication method
sin exponer session credential.

## 182. Revoked Session Retention

Una sesión revocada no necesita permanecer indefinidamente.

## 183. Revocation Evidence

Puede conservarse metadata mínima:
session management ID hash/reference
revokedAt
reason
actor
durante retention period.

## 184. Remember-Me Metadata

Misma separación:
management metadata
≠
persistent login secret

## 185. Device Trust Metadata

Debe conservar:
trust state
created
last used
revoked
según necesidad.

## 186. Device Recognition Secret

No debe mostrarse.

## 187. Machine Authentication Metadata

Documento 33.
Puede contener:
service identity
certificate serial
key ID
issuer
audience
workload identity
attestation metadata

## 188. Machine Metadata != Non-Sensitive

Aunque no represente directamente una persona, puede revelar infraestructura crítica.
Por ello también necesita clasificación.

## 189. Infrastructure Confidentiality

Ejemplos:
cluster names
service topology
certificate identities
internal hostnames
trust domains
pueden ser Confidential/Sensitive.

## 190. Machine Secrets

Nunca entran en general telemetry.

## 191. Security Center

Documento 35.
Debe consumir privacy-aware projections.

## 192. Security Center Snapshot

Authoritative Security Data
        ↓
Privacy Projection
        ↓
Viewer Scope
        ↓
Redaction
        ↓
Security Center Snapshot

## 193. Never Build Then Hide

Para información altamente sensible:
Preferir consultar únicamente los datos permitidos en vez de cargar todo y ocultarlo después.

Especialmente en multi-tenancy.

## 194. Query Scoping

interface AuthenticationPrivacyAwareQueryInterface
{
    public function query(
        AuthenticationPrivacyQueryContext $context
    ): AuthenticationPrivacySafeResult;
}

## 195. Caching

Privacy projection cache debe incluir:
identity
viewer authority
tenant
realm
purpose
privacy policy version
security version

## 196. Critical Rule

Nunca:
security_center:{identityId}
como única cache key.

## 197. Why

Un platform security admin podría obtener:
full metadata
y contaminar cache usada posteriormente por usuario normal.

## 198. Privacy Cache Key

Conceptualmente:
auth_privacy:
  identity:
  viewer_scope:
  tenant:
  realm:
  purpose:
  policy_version:
  projection_version:

## 199. Cache Retention

Cache TTL no deberá superar de forma peligrosa la retention/consent/revocation semantics.

## 200. Erasure and Cache

Erasure deberá invalidar caches.

## 201. Search Indexes

También:
Elasticsearch
OpenSearch
analytics indexes
materialized views
deberán formar parte del data lifecycle cuando contengan Authentication metadata.

## 202. Backup Problem

Eliminar una fila de producción no necesariamente elimina inmediatamente:
backups
snapshots
replicas
archives

## 203. Backup Governance

Authentication privacy deberá integrarse con Database Backup/Retention systems.

## 204. Backup Erasure

No siempre será práctico modificar backups históricos individualmente.
La arquitectura deberá soportar políticas como:
backup expires naturally
+
deleted identity remains inaccessible after restore
+
post-restore erasure replay
según requisitos.

## 205. Restore Safety

Después de restaurar backup antiguo:
deleted credentials
revoked sessions
deleted identities
no deberán resucitar.

## 206. Erasure Ledger

Puede existir:
interface AuthenticationErasureLedgerInterface
{
    public function record(
        AuthenticationErasureRecord $record
    ): void;
}

## 207. Purpose

Después de restore:
Replay Erasure Ledger
        ↓
Reapply deletions/revocations

## 208. Ledger Privacy

Debe contener la mínima información necesaria para identificar records afectados.

## 209. Tombstones

Documento 41.
Tombstones deberán estar sujetos a privacy policy.

## 210. Tombstone Minimalism

Ideal:
opaque identity reference
deletedAt
deletion operation reference
No:
full name
email
phone
address
all old login history

## 211. Pseudonymization

Se define como transformación reversible o relacionable mediante información adicional controlada.

## 212. Anonymization

Debe eliminar razonablemente la posibilidad de reidentificación dentro del threat model definido.

## 213. Hashing Is Not Automatically Anonymization

Ejemplo:
SHA256(email)
puede ser fácilmente reversible por dictionary attack.

## 214. HMAC

Para ciertos lookup/pseudonymization use cases puede ser preferible:
HMAC(secret, normalized_identifier)
pero sigue siendo pseudonymous, no necesariamente anonymous.

## 215. Key Rotation

Pseudonymization keys se integrarán con documento 31.

## 216. Crypto-Shredding

Para determinados datasets cifrados puede soportarse eliminación de clave como parte de erasure.

## 217. Crypto-Shredding Caveat

Solo es válido si:
all copies are encrypted under relevant key
key truly destroyed
no plaintext replicas exist

## 218. Tokenization

Puede separar:
operational reference
de PII almacenada en un vault especializado.

## 219. Privacy Vault

No será obligatorio, pero arquitectura extensible.

## 220. Data Residency

Multi-region Authentication puede necesitar controlar dónde se almacena metadata.

## 221. Residency Policy

interface AuthenticationDataResidencyPolicyInterface
{
    public function placement(
        AuthenticationDataAsset $asset,
        AuthenticationPrivacyScope $scope
    ): AuthenticationDataPlacementDecision;
}

## 222. Region

Puede decidir:
EU
US
MX
Tenant-selected region
Global replicated
según deployment.

## 223. Replication

No toda Authentication metadata debe replicarse globalmente.

## 224. Global Security vs Residency

Necesita balance:
fast global compromise detection
vs
data residency
mediante derived/minimized signals cuando sea posible.

## 225. Example

Región local conserva:
full IP
mientras sistema global recibe:
risk category = HIGH
country = MX
signal type = credential_stuffing

## 226. Cross-Region Transfer

Debe ser explícito en governance policy.

## 227. Third-Party Risk Providers

Si se envía información a:
fraud provider
IP reputation service
device intelligence service
SMS provider
email provider
identity provider
debe existir outbound data policy.

## 228. Outbound Data Governance

interface AuthenticationOutboundDataPolicyInterface
{
    public function evaluate(
        AuthenticationOutboundDataRequest $request
    ): AuthenticationOutboundDataDecision;
}

## 229. Provider Minimization

Enviar únicamente campos necesarios.

## 230. Example

IP reputation provider puede necesitar:
IP
pero no:
user full name
password hash
tenant billing information

## 231. Provider Purpose

Cada external transfer tendrá:
provider
purpose
data categories
scope
retention expectations

## 232. Plugin Governance

Authentication plugins tampoco podrán recolectar arbitrariamente metadata.

## 233. Plugin Data Declaration

Cada plugin deberá declarar:
data types consumed
data types produced
purpose
classification
retention
external transfers

## 234. Privacy Capability Manifest

interface AuthenticationPrivacyCapabilityManifestInterface
{
    public function dataCapabilities(): AuthenticationDataCapabilitySet;
}

## 235. Plugin Installation Validation

Un plugin que declare:
retain full authentication payload forever
podrá ser rechazado por governance policy.

## 236. Custom Authentication Methods

También deberán registrar metadata schema.

## 237. Unknown Fields

No deben entrar automáticamente a audit/logs.

## 238. Schema Registry

interface AuthenticationDataSchemaRegistryInterface
{
    public function register(
        AuthenticationDataSchema $schema
    ): void;
}

## 239. Schema Metadata

Cada field puede declarar:
classification
purpose
persistence
retention
redaction
exportability
viewer visibility

## 240. Example Schema

AuthenticationDataSchema::for('login_attempt')
    ->field('identity_id')
        ->classification('confidential')
        ->purpose('authentication')
    ->field('source_ip')
        ->classification('sensitive')
        ->purpose('fraud_prevention')
        ->retention('7 days')
    ->field('password')
        ->classification('secret')
        ->persist(false)
        ->log(false)
        ->export(false);
Conceptual DSL.

## 241. Secure Defaults

Unknown Authentication data fields deberían default a:
do not log
do not export
do not expose
hasta ser clasificados.

## 242. Privacy Policy Engine

interface AuthenticationPrivacyPolicyEngineInterface
{
    public function evaluate(
        AuthenticationPrivacyOperation $operation,
        AuthenticationPrivacyContext $context
    ): AuthenticationPrivacyDecision;
}

## 243. Operations

COLLECT
STORE
READ
PROJECT
EXPORT
TRANSFER
ARCHIVE
PSEUDONYMIZE
ANONYMIZE
DELETE
HOLD

## 244. Policy Inputs

Data Type
Classification
Purpose
Identity Type
Tenant
Realm
Environment
Viewer
Security Incident State
Retention Hold
Consent/Preference
Policy Version

## 245. Static vs Dynamic

Static:
classification
default retention
field schema
Dynamic:
incident hold
viewer
tenant
current purpose
consent state

## 246. Policy Compilation

Static privacy policy deberá compilarse.

## 247. Compiled Privacy Policy

final readonly class CompiledAuthenticationPrivacyPolicy
{
    public function __construct(
        public AuthenticationPrivacyPolicyVersion $version,
        public array $dataSchemas,
        public array $retentionRules,
        public array $accessRules,
    ) {}
}

## 248. Secret-Free Compilation

Compiled policy nunca contendrá:
PII
tokens
keys
credentials

## 249. Policy Version

Todas las decisiones relevantes deberán poder asociarse con versión.

## 250. Governance Lifecycle

Policies podrán tener:
DRAFT
VALIDATED
ACTIVE
DEPRECATED
RETIRED

## 251. Privacy Policy Changes

Serán auditables.

## 252. Privileged Operation

Cambiar:
raw IP retention from 7 days to forever
debe tratarse como governance-sensitive operation.

## 253. Policy Change Authentication

Puede requerir documento 32:
fresh admin authentication
phishing-resistant MFA

## 254. Authorization

Y autorización independiente.

## 255. Dual Control

Cambios de privacy/security retention críticos podrán requerir aprobación dual.

## 256. Dry Run

Antes de aplicar policy:
How much data becomes eligible for deletion?
How much retention increases?
Which tenants are affected?
Which holds conflict?

## 257. Privacy Policy Simulation

interface AuthenticationPrivacyPolicySimulatorInterface
{
    public function simulate(
        AuthenticationPrivacyPolicyCandidate $policy
    ): AuthenticationPrivacySimulationResult;
}

## 258. Shadow Evaluation

Puede comparar nueva policy sin aplicarla.

## 259. Privacy Posture

Security Center/admin tooling puede mostrar findings:
raw IP retained beyond policy
unclassified auth metadata
expired data pending deletion
plugin missing privacy manifest
retention conflict
stale incident hold

## 260. Privacy Finding

final readonly class AuthenticationPrivacyFinding
{
    public function __construct(
        public AuthenticationPrivacyFindingType $type,
        public AuthenticationPrivacyFindingSeverity $severity,
        public AuthenticationDataType|null $dataType,
        public string $reasonCode,
    ) {}
}

## 261. Governance Scanner

interface AuthenticationPrivacyGovernanceScannerInterface
{
    public function scan(
        AuthenticationPrivacyGovernanceContext $context
    ): AuthenticationPrivacyFindingSet;
}

## 262. Data Inventory

VoltStack debería poder responder:
What Authentication data do we store?
Why?
Where?
For how long?
Who can see it?
Which providers receive it?

## 263. Authentication Data Inventory

interface AuthenticationDataInventoryInterface
{
    public function inventory(
        AuthenticationDataInventoryContext $context
    ): AuthenticationDataInventorySnapshot;
}

## 264. Inventory Example

SOURCE_IP
Classification: Sensitive
Purpose: Fraud Prevention
Retention: 7 days full / 90 days generalized
Storage: Auth Security Store
Exportable: Limited
Tenant Visible: No
Security Operator: Yes
External Transfer: Reputation Provider

## 265. Data Lineage

Enterprise deployments pueden necesitar saber:
Source IP
   ↓
Risk Engine
   ↓
Risk Signal
   ↓
Authentication Decision
   ↓
Security Event

## 266. Lineage Metadata

Debe describir flujo, no duplicar data values innecesariamente.

## 267. Telemetry

Authentication metrics deberán evitar high-cardinality PII.
No:
auth_login_total{email="user@example.com"}

## 268. Safe Metrics

Preferir:
auth_login_total{
  method="passkey",
  outcome="success"
}

## 269. Tenant IDs in Metrics

Solo cuando cardinalidad/policy lo permitan.

## 270. Tracing

No incluir:
email
raw IP
tokens
credential IDs
session cookies
por default.

## 271. Trace Attributes

Preferir:
auth.method
auth.outcome
auth.assurance
auth.realm
auth.policy_decision

## 272. Trace Sampling

Security failure traces pueden necesitar sampling diferente, pero no relajación de redaction.

## 273. Error Reporting

Exception monitoring tampoco debe recibir secrets.

## 274. Exception Context Sanitization

interface AuthenticationExceptionSanitizerInterface
{
    public function sanitize(
        Throwable $exception,
        AuthenticationExceptionContext $context
    ): SanitizedAuthenticationException;
}

## 275. Database Queries

Observability de queries puede revelar:
email
token
identifier
mediante bindings.
Auth deberá poder marcar bindings como sensitive.

## 276. Sensitive SQL Bindings

Telemetry system deberá redacted them.

## 277. Development Environment

Debug mode no autoriza mostrar secrets.

## 278. Critical Rule

APP_DEBUG=true
nunca deberá significar:
show authentication credentials

## 279. Test Data

Testing system deberá favorecer synthetic identities.

## 280. Production Data in Tests

No copiar production Authentication data a development sin governance.

## 281. Fixtures

Deben utilizar:
fake emails
fake IPs
fake device metadata
fake credentials

## 282. Data Masking

Si se usan snapshots reales autorizados, deberán poder ser masked.

## 283. Backups

Backups de Authentication stores deberán cifrarse conforme documento 31 y Database Security.

## 284. Backup Access

Más restringido que acceso normal de aplicación.

## 285. Retention of Backups

Debe estar alineado con data governance.

## 286. Authentication Data Store Separation

VoltStack podrá separar:
Operational Auth Store
Security Event Store
Audit Store
Incident Evidence Store
Secrets Store

## 287. Why Separation

Permite políticas diferentes de:
access
retention
encryption
replication
backup

## 288. Secret Store

Nunca debe tratarse como analytics source.

## 289. Incident Evidence Store

Puede tener controles más fuertes.

## 290. Immutable Audit

"Inmutable" no significa necesariamente "eterno".
Puede ser:
immutable during retention window
y luego expirar conforme policy.

## 291. Cryptographic Audit Integrity

Opcionalmente:
hash chaining
signatures
append-only storage
WORM
según threat model.

## 292. Privacy vs Audit Integrity

Erasure puede requerir:
remove identifying field
sin modificar evidencia estructural.
Ejemplo:
Actor Identity ID → pseudonymous tombstone reference

## 293. Identity Deletion Integration

Documento 41.
Deletion deberá solicitar:
Authentication Data Erasure Plan

## 294. Erasure Plan

final readonly class AuthenticationDataErasurePlan
{
    public function __construct(
        public array $delete,
        public array $anonymize,
        public array $retain,
        public array $held,
    ) {}
}

## 295. Categories

Ejemplo:
Password Hash           → DELETE
Active Sessions         → DELETE
Remember-Me Credentials → DELETE
Passkey Credentials     → DELETE
TOTP Secret             → DELETE
Recovery Codes          → DELETE
Security Audit          → PSEUDONYMIZE / RETAIN
Incident Evidence       → HOLD
Aggregated Metrics      → RETAIN if anonymous

## 296. Erasure Orchestrator

interface AuthenticationDataErasureOrchestratorInterface
{
    public function execute(
        AuthenticationDataErasureRequest $request
    ): AuthenticationDataErasureResult;
}

## 297. Erasure Idempotency

Debe soportar múltiples ejecuciones.

## 298. Partial Erasure

Debe poder reportar:
credentials erased
sessions erased
audit pseudonymized
incident evidence retained under hold
backup expiration pending

## 299. Never Claim Full Erasure Incorrectly

Resultado deberá distinguir:
AUTHENTICATION_ERASURE_COMPLETED
de:
GLOBAL_APPLICATION_ERASURE_COMPLETED

## 300. Erasure Audit Paradox

Necesitamos registrar que ocurrió erasure sin volver a almacenar todos los datos borrados.

## 301. Minimal Erasure Record

operation ID
opaque identity tombstone
completedAt
policy version
categories processed
holds remaining

## 302. Retention Hold

Tipos:
enum AuthenticationRetentionHoldType: string
{
    case SecurityIncident = 'security_incident';
    case Legal = 'legal';
    case Compliance = 'compliance';
    case Investigation = 'investigation';
    case AdministrativeReview = 'administrative_review';
}

## 303. Hold Authority

No cualquier plugin/user puede crear holds.

## 304. Hold Creation

Security-sensitive privileged operation.

## 305. Hold Expiration

Debe tener:
reviewAt
expiresAt
cuando sea posible.

## 306. Indefinite Holds

Si se permiten deberán requerir governance excepcional.

## 307. Hold Review

Sistema deberá detectar holds sin revisión.

## 308. Hold Release

Auditable.

## 309. Hold Scope Expansion

No debe ocurrir implícitamente.

## 310. Data Breach / Compromise

Documento 40.
Si Authentication metadata es comprometida, governance inventory ayuda a determinar:
what data existed
classification
affected scope
retention
external providers

## 311. Credential Secret Exposure

Se maneja como security incident, no solo privacy finding.

## 312. Privacy Incident vs Authentication Incident

Pueden superponerse pero son conceptos distintos.

## 313. Security Notification

No incluir información excesiva.
Ejemplo email:
A new sign-in was detected.
puede mostrar metadata limitada.

## 314. Notification Privacy

No enviar:
full security incident evidence
full IP history
raw risk engine output
por email/SMS salvo necesidad explícita.

## 315. Out-of-Band Channels

Documento 43 profundizará esta materia.

## 316. Data Governance Events

AuthenticationPrivacyPolicyActivated
AuthenticationRetentionPolicyChanged
AuthenticationDataRetentionExpired
AuthenticationDataDeleted
AuthenticationDataAnonymized
AuthenticationDataPseudonymized
AuthenticationRetentionHoldApplied
AuthenticationRetentionHoldReleased
AuthenticationConsentGranted
AuthenticationConsentWithdrawn
AuthenticationPrivacyRequestCreated
AuthenticationPrivacyRequestCompleted
AuthenticationDataExportGenerated
AuthenticationErasureStarted
AuthenticationErasureCompleted

## 317. Event Privacy

Irónicamente, eventos de privacy también deberán estar minimizados.

## 318. Export File Security

Authentication data export deberá:
expire
be access-controlled
be encrypted where appropriate
be auditable
avoid public URLs

## 319. Download Token

Debe ser:
short-lived
single-purpose
revocable

## 320. Export Storage Cleanup

Eliminar automáticamente después del retention window.

## 321. Export Generation

Puede ejecutarse async.

## 322. Export Notification

Usuario recibe notificación cuando esté listo.
Documento 43.

## 323. Export Reauthentication

Descargar puede requerir fresh auth incluso si request inicial ya fue autenticado, dependiendo de tiempo/policy.

## 324. Privacy Admin Tooling

CLI conceptual:
volt auth:privacy:inventory
volt auth:privacy:scan
volt auth:privacy:retention
volt auth:privacy:retention:simulate
volt auth:privacy:cleanup
volt auth:privacy:holds
volt auth:privacy:export {identity}
volt auth:privacy:erase {identity}

## 325. CLI Does Not Bypass Policy

Mismos principios:
Actor
Authorization
Privileged Authentication
Reason
Audit

## 326. Governance Dashboard

Podrá mostrar:
Unclassified data types
Retention conflicts
Expired data pending cleanup
Active holds
Stale holds
Plugins without privacy manifests
Outbound providers
Data residency violations
Erasure backlog

## 327. Performance

Privacy governance no deberá convertir cada Authentication request en decenas de DB queries.

## 328. Static Policies

Compilar:
classification
collection
retention defaults
redaction rules
schemas

## 329. Dynamic Decisions

Evaluar únicamente cuando sea necesario:
viewer
tenant
incident hold
consent
current purpose

## 330. Hot Path

Login debería poder utilizar una estructura precompilada:
Data Type
→ Collection Strategy
→ Minimization Strategy
→ Retention Class

## 331. Privacy Decision Cache

Solo para policy metadata, no para secret values.

## 332. Distributed Policy Version

Nodos deberán conocer:
privacyPolicyVersion
retentionPolicyVersion

## 333. Stale Privacy Policy

Un nodo no debería continuar recolectando datos prohibidos indefinidamente.

## 334. Critical Policy Update

Ejemplo:
STOP COLLECTING RAW DEVICE FINGERPRINT
debe propagarse rápidamente.

## 335. Fail Behavior

Si privacy policy crítica no puede resolverse:
default = do not collect optional metadata
será una estrategia segura.

## 336. Security-Critical Collection

Pero no deberá impedir controles indispensables sin policy diseñada.
Por tanto debe distinguirse:
mandatory security processing
optional enrichment

## 337. Fail-Closed for Optional Collection

Si no sabemos si está permitido almacenar metadata opcional:
do not persist it

## 338. Fail-Safe Authentication

No necesariamente negar login porque analytics/privacy enrichment falló.

## 339. Data Governance Availability

Debe separarse de Authentication availability cuando sea seguro hacerlo.

## 340. FrankenPHP

No almacenar:
current privacy viewer
current consent
current tenant retention
current data projection
en mutable singleton.

## 341. Request-Scoped Privacy Context

final readonly class AuthenticationPrivacyContext
{
    public function__construct(
        public AuthenticationPrivacyScope $scope,
        public AuthenticationDataPurposeSet $purposes,
        public AuthenticationPrivacyPolicyVersion $policyVersion,
        public ActorReference|null $actor,
    ) {}
}

## 342. Fiber Safety

Cada concurrent request deberá mantener su propio privacy context.

## 343. Critical Example

Fiber A:
Platform Security Admin
→ full IP visibility

Fiber B:
Normal User
→ redacted IP visibility
Nunca deberán compartir projection.

## 344. Worker Reset

Limpiar:
PrivacyContext
ViewerContext
ConsentSnapshot
ProjectionPolicy
Temporary Data Buffers
Export Context
entre requests/jobs.

## 345. Temporary Buffers

Especial atención a:
raw tokens
raw headers
raw provider payloads
en long-running workers.

## 346. Memory Retention

Los secrets no deberán quedar referenciados innecesariamente en worker singletons.

## 347. Exception Objects

Evitar que capturen raw request payloads con credentials.

## 348. Queue Payloads

No serializar Authentication secrets dentro de jobs.

## 349. Async Privacy Jobs

Utilizar:
opaque references
operation IDs
y recuperar datos autorizados en ejecución.

## 350. Dead-Letter Queues

También deben cumplir retention/redaction.

## 351. Retry Logs

No deben duplicar sensitive payloads.

## 352. Failure Taxonomy

AUTH_PRIVACY_DATA_TYPE_UNCLASSIFIED
AUTH_PRIVACY_PURPOSE_UNDEFINED
AUTH_PRIVACY_COLLECTION_DENIED
AUTH_PRIVACY_COLLECTION_POLICY_UNAVAILABLE
AUTH_PRIVACY_RETENTION_POLICY_INVALID
AUTH_PRIVACY_RETENTION_POLICY_CONFLICT
AUTH_PRIVACY_RETENTION_HOLD_ACTIVE
AUTH_PRIVACY_RETENTION_HOLD_INVALID
AUTH_PRIVACY_DATA_ACCESS_DENIED
AUTH_PRIVACY_EXPORT_DENIED
AUTH_PRIVACY_EXPORT_REAUTHENTICATION_REQUIRED
AUTH_PRIVACY_ERASURE_DENIED
AUTH_PRIVACY_ERASURE_PARTIAL
AUTH_PRIVACY_ERASURE_HOLD_ACTIVE
AUTH_PRIVACY_CONSENT_REQUIRED
AUTH_PRIVACY_CONSENT_WITHDRAWN
AUTH_PRIVACY_POLICY_VERSION_STALE
AUTH_PRIVACY_DATA_RESIDENCY_VIOLATION
AUTH_PRIVACY_OUTBOUND_TRANSFER_DENIED
AUTH_PRIVACY_PLUGIN_MANIFEST_INVALID
AUTH_PRIVACY_UNSAFE_DATA_FIELD

## 353. Public Errors

No revelar:
incident evidence retained
legal hold details
security operator notes
risk model internals
sin authority.

## 354. Security Invariants — Collection

AUTH-PRIV-COL-01
Todo dato persistido por Authentication tendrá una finalidad definida.
AUTH-PRIV-COL-02
Procesar un dato no implica persistirlo.
AUTH-PRIV-COL-03
Optional metadata no se almacenará si policy no puede determinar que está permitido.
AUTH-PRIV-COL-04
Unknown fields no se loguearán automáticamente.
AUTH-PRIV-COL-05
Authentication plugins declararán sus necesidades de datos.
AUTH-PRIV-COL-06
Precise geolocation no será requisito default del core.
AUTH-PRIV-COL-07
Device fingerprinting invasivo no será default.

## 355. Security Invariants — Secrets

AUTH-PRIV-SEC-01
Passwords nunca aparecerán en logs.
AUTH-PRIV-SEC-02
Bearer tokens nunca aparecerán en audit.
AUTH-PRIV-SEC-03
Session cookies nunca aparecerán en traces.
AUTH-PRIV-SEC-04
TOTP seeds nunca aparecerán en Security Center.
AUTH-PRIV-SEC-05
Recovery codes serán show-once.
AUTH-PRIV-SEC-06
Private keys nunca serán exportadas por privacy export.
AUTH-PRIV-SEC-07
Debug mode no desactivará secret redaction.

## 356. Security Invariants — Retention

AUTH-PRIV-RET-01
Todo persisted Authentication data tendrá retention semantics.
AUTH-PRIV-RET-02
Retention indefinida requerirá policy explícita.
AUTH-PRIV-RET-03
Minimum/maximum conflicts no se resolverán silenciosamente.
AUTH-PRIV-RET-04
Expired data será procesable de forma idempotente.
AUTH-PRIV-RET-05
Security holds serán scoped.
AUTH-PRIV-RET-06
Hold release será auditable.
AUTH-PRIV-RET-07
Backup restore no resucitará datos/credentials eliminados.

## 357. Security Invariants — Consent

AUTH-PRIV-CON-01
Consent no será tratado como universal legal basis.
AUTH-PRIV-CON-02
Consent tendrá purpose.
AUTH-PRIV-CON-03
Consent tendrá policy version.
AUTH-PRIV-CON-04
Withdrawal será propagable.
AUTH-PRIV-CON-05
Withdrawal no eliminará mandatory security evidence automáticamente.
AUTH-PRIV-CON-06
Security configuration != consent.
AUTH-PRIV-CON-07
Authentication enrollment != privacy consent automáticamente.

## 358. Security Invariants — Visibility

AUTH-PRIV-VIS-01
Administrability != unrestricted visibility.
AUTH-PRIV-VIS-02
Tenant A no verá metadata de Tenant B.
AUTH-PRIV-VIS-03
Viewer-aware projection será obligatoria para sensitive metadata.
AUTH-PRIV-VIS-04
Caches incluirán viewer authority.
AUTH-PRIV-VIS-05
Support no tendrá acceso automático a raw security evidence.
AUTH-PRIV-VIS-06
Exports nunca incluirán Authentication secrets.

## 359. Security Invariants — Erasure

AUTH-PRIV-ERA-01
Erasure será idempotente.
AUTH-PRIV-ERA-02
Erasure respetará active holds.
AUTH-PRIV-ERA-03
Authentication erasure no afirmará global application erasure.
AUTH-PRIV-ERA-04
Deleted credentials no podrán resucitar desde backups.
AUTH-PRIV-ERA-05
Tombstones serán minimizados.
AUTH-PRIV-ERA-06
Erasure audit no reconstruirá los datos borrados.

## 360. Security Invariants — Runtime

AUTH-PRIV-RT-01
Privacy Context será request/fiber scoped.
AUTH-PRIV-RT-02
No existirán mutable global viewer contexts.
AUTH-PRIV-RT-03
Temporary secret buffers no sobrevivirán intencionalmente entre requests.
AUTH-PRIV-RT-04
Queue jobs no transportarán raw credentials.
AUTH-PRIV-RT-05
Dead-letter storage seguirá privacy policy.
AUTH-PRIV-RT-06
Policy caches contendrán metadata, no secret values.

## 361. Anti-Pattern

Store every login IP forever just in case.

## 362. Anti-Pattern

Security data doesn't need privacy rules.

## 363. Anti-Pattern

logger()->debug($request->all());
dentro de login.

## 364. Anti-Pattern

APP_DEBUG=true
→ dump OAuth tokens

## 365. Anti-Pattern

Store complete OIDC ID token forever.

## 366. Anti-Pattern

Hash email with SHA-256
→ therefore anonymous

## 367. Anti-Pattern

Tenant admin
→ can see global authentication history

## 368. Anti-Pattern

Support role
→ can see recovery metadata and raw risk signals

## 369. Anti-Pattern

Consent withdrawn
→ delete security incident evidence
automáticamente.

## 370. Anti-Pattern

Legal hold
→ retain all data forever

## 371. Anti-Pattern

Delete production row
→ assume backups are erased

## 372. Anti-Pattern

Privacy export
→ SELECT * FROM auth_tables

## 373. Anti-Pattern

Plugin installed
→ plugin may log anything

## 374. Anti-Pattern

Cache security center only by identity ID.

## 375. Anti-Pattern

Security score
→ preserve every raw signal forever for explainability.

## 376. Anti-Pattern

Remember device checkbox
→ consent to all tracking.

## 377. Componentes principales

AuthenticationDataAsset
AuthenticationDataType
AuthenticationDataClassification
AuthenticationDataPurpose
AuthenticationDataPurposeSet

AuthenticationDataSchema
AuthenticationDataSchemaRegistry

AuthenticationDataCollectionPolicy
AuthenticationDataMinimizer
AuthenticationDataRedactor

AuthenticationPrivacyPolicyEngine
AuthenticationPrivacyPolicyVersion
CompiledAuthenticationPrivacyPolicy

AuthenticationRetentionPolicy
AuthenticationRetentionDecision
AuthenticationRetentionPlanner
AuthenticationRetentionPolicyCompiler

AuthenticationConsentRecord
AuthenticationConsentResolver

AuthenticationDataAccessPolicy
AuthenticationPrivacyProjectionPolicy

AuthenticationSecurityMetadataGovernance
AuthenticationSecurityEvidencePolicy

AuthenticationRetentionHold

AuthenticationDataExporter
AuthenticationDataErasurePlan
AuthenticationDataErasureOrchestrator
AuthenticationErasureLedger

AuthenticationDataResidencyPolicy
AuthenticationOutboundDataPolicy

AuthenticationPrivacyGovernanceScanner
AuthenticationDataInventory

## 378. Namespace sugerido

VoltStack\Quantum\Auth\Privacy

## 379. Estructura sugerida

src/Quantum/Auth/Privacy/
├── Contracts/
│   ├── AuthenticationPrivacyPolicyEngineInterface.php
│   ├── AuthenticationDataCollectionPolicyInterface.php
│   ├── AuthenticationDataMinimizerInterface.php
│   ├── AuthenticationDataRedactorInterface.php
│   ├── AuthenticationRetentionPolicyInterface.php
│   ├── AuthenticationRetentionPolicyCompilerInterface.php
│   ├── AuthenticationRetentionPlannerInterface.php
│   ├── AuthenticationConsentResolverInterface.php
│   ├── AuthenticationDataAccessPolicyInterface.php
│   ├── AuthenticationPrivacyProjectionPolicyInterface.php
│   ├── AuthenticationSecurityMetadataGovernanceInterface.php
│   ├── AuthenticationSecurityEvidencePolicyInterface.php
│   ├── AuthenticationDataExporterInterface.php
│   ├── AuthenticationDataErasureOrchestratorInterface.php
│   ├── AuthenticationErasureLedgerInterface.php
│   ├── AuthenticationDataResidencyPolicyInterface.php
│   ├── AuthenticationOutboundDataPolicyInterface.php
│   ├── AuthenticationPrivacyGovernanceScannerInterface.php
│   └── AuthenticationDataInventoryInterface.php
│
├── Data/
│   ├── AuthenticationDataAsset.php
│   ├── AuthenticationDataType.php
│   ├── AuthenticationDataClassification.php
│   ├── AuthenticationDataValue.php
│   └── AuthenticationDataScope.php
│
├── Purpose/
│   ├── AuthenticationDataPurpose.php
│   ├── AuthenticationDataPurposeSet.php
│   └── AuthenticationDataPurposeBinding.php
│
├── Schema/
│   ├── AuthenticationDataSchema.php
│   ├── AuthenticationDataField.php
│   └── AuthenticationDataSchemaRegistry.php
│
├── Collection/
│   ├── AuthenticationDataCollectionRequest.php
│   ├── AuthenticationDataCollectionDecision.php
│   └── AuthenticationDataCollectionPolicy.php
│
├── Minimization/
│   ├── AuthenticationDataMinimizationStrategy.php
│   ├── AuthenticationDataMinimizationContext.php
│   └── AuthenticationDataMinimizer.php
│
├── Redaction/
│   ├── AuthenticationDataRedactionContext.php
│   └── AuthenticationDataRedactor.php
│
├── Retention/
│   ├── AuthenticationRetentionPolicy.php
│   ├── AuthenticationRetentionDecision.php
│   ├── AuthenticationRetentionAction.php
│   ├── AuthenticationRetentionPolicyVersion.php
│   ├── AuthenticationRetentionPolicyCompiler.php
│   ├── CompiledAuthenticationRetentionPolicySet.php
│   ├── AuthenticationRetentionPlanner.php
│   └── AuthenticationRetentionMigrationPlan.php
│
├── Hold/
│   ├── AuthenticationRetentionHold.php
│   ├── AuthenticationRetentionHoldType.php
│   └── AuthenticationRetentionHoldRepository.php
│
├── Consent/
│   ├── AuthenticationConsentRecord.php
│   ├── AuthenticationConsentStatus.php
│   ├── AuthenticationConsentPurpose.php
│   └── AuthenticationConsentResolver.php
│
├── Access/
│   ├── AuthenticationDataAccessRequest.php
│   ├── AuthenticationDataAccessDecision.php
│   └── AuthenticationDataAccessPolicy.php
│
├── Projection/
│   ├── AuthenticationPrivacyViewerContext.php
│   ├── AuthenticationPrivacyProjection.php
│   └── AuthenticationPrivacyProjectionPolicy.php
│
├── Evidence/
│   ├── AuthenticationSecurityEvidenceCandidate.php
│   ├── AuthenticationSecurityEvidenceDecision.php
│   └── AuthenticationSecurityEvidencePolicy.php
│
├── Export/
│   ├── AuthenticationDataExport.php
│   ├── AuthenticationDataExportContext.php
│   └── AuthenticationDataExporter.php
│
├── Erasure/
│   ├── AuthenticationDataErasurePlan.php
│   ├── AuthenticationDataErasureRequest.php
│   ├── AuthenticationDataErasureResult.php
│   ├── AuthenticationDataErasureOrchestrator.php
│   └── AuthenticationErasureLedger.php
│
├── Residency/
│   ├── AuthenticationDataResidencyPolicy.php
│   └── AuthenticationDataPlacementDecision.php
│
├── Outbound/
│   ├── AuthenticationOutboundDataRequest.php
│   ├── AuthenticationOutboundDataDecision.php
│   └── AuthenticationOutboundDataPolicy.php
│
├── Governance/
│   ├── AuthenticationPrivacyFinding.php
│   ├── AuthenticationPrivacyGovernanceScanner.php
│   ├── AuthenticationDataInventory.php
│   └── AuthenticationPrivacyPolicySimulator.php
│
├── Compiler/
│   ├── AuthenticationPrivacyPolicyCompiler.php
│   └── CompiledAuthenticationPrivacyPolicy.php
│
├── Events/
│   └── ...
│
├── Runtime/
│   ├── AuthenticationPrivacyContext.php
│   ├── AuthenticationPrivacyScope.php
│   ├── AuthenticationPrivacyContextStorage.php
│   └── AuthenticationPrivacyRuntimeResetter.php
│
└── Exceptions/
    └── ...

## 380. Ejemplo de configuración

return [

    'privacy' => [

        'collection' => [

            'source_ip' => [
                'purpose' => 'fraud_prevention',
                'strategy' => 'collect_minimized',
            ],

            'raw_user_agent' => [
                'purpose' => 'operational_diagnostics',
                'strategy' => 'collect_ephemeral',
            ],

        ],

        'retention' => [

            'source_ip' => [
                'full' => '7 days',
                'generalized' => '90 days',
            ],

            'authentication_activity' => '90 days',

            'security_audit' => '1 year',

        ],

        'exports' => [
            'expires_after' => '24 hours',
            'require_fresh_authentication' => true,
        ],

        'unknown_fields' => [
            'log' => false,
            'export' => false,
        ],

    ],

];

## 381. Ejemplo — Login Privacy Pipeline

Login Request
     │
     ├── Email
     ├── IP
     ├── User-Agent
     └── Credential Evidence
           │
           ▼
Authentication Data Classification
           │
           ▼
Collection Policy
           │
    ┌──────┼──────────────┐
    ▼      ▼              ▼
Process  Persist       Discard
    │      │
    │      ▼
    │   Minimization
    │      │
    └──────┼──────────────┐
           ▼              ▼
       Risk Engine    Security Audit
           │              │
           ▼              ▼
      Risk Finding    Safe Metadata
Credential secret nunca entra al audit.

## 382. Ejemplo — Progressive IP Minimization

Successful Login
      ↓
Full IP
      │
      │ 7 days
      ▼
Network Prefix
      │
      │ 30 days
      ▼
Country + Risk Category
      │
      │ 90 days
      ▼
Aggregated Statistics
Esto permite seguridad operacional sin conservar indefinidamente máxima precisión.

## 383. Ejemplo — Account Deletion

Identity Deletion Approved
        ↓
Authentication Erasure Planner
        │
        ├── Password Hash → DELETE
        ├── TOTP Secret → DELETE
        ├── Recovery Codes → DELETE
        ├── Sessions → DELETE
        ├── Remember-Me → DELETE
        ├── Passkeys → DELETE
        ├── Device Credentials → DELETE
        ├── Activity → ANONYMIZE
        ├── Audit → PSEUDONYMIZE
        └── Incident Evidence → HOLD
        ↓
Execute
        ↓
Erasure Ledger
        ↓
Authentication Erasure Completed

## 384. Ejemplo — Security Incident Hold

Normal Login Metadata
Retention = 30 days
        │
        ▼
Account Takeover Confirmed
        │
        ▼
Evidence Preservation Policy
        │
        ▼
Relevant 48-hour Window
        │
        ▼
SECURITY_INCIDENT_HOLD
        │
        ▼
Normal Retention Temporarily Suspended
Solo para los datos relevantes.

## 385. Ejemplo — Multi-Tenant Visibility

Global Authentication Data
           │
           ├─────────────┐
           ▼             ▼
      Tenant A        Tenant B
      Viewer          Viewer
           │             │
           ▼             ▼
Scoped Projection   Scoped Projection
Tenant A nunca recibe metadata exclusiva de Tenant B.

## 386. Ejemplo — Privacy-Aware Security Center

Security Data
      ↓
Viewer = User
      ↓
Privacy Policy
      ↓
Session:
Chrome on Windows
Monterrey area
Last active 2h ago
Security operator autorizado podría recibir mayor precisión, pero mediante otra projection.

## 387. Ejemplo — Federated Login

OIDC Provider Response
       │
       ├── iss
       ├── sub
       ├── email
       ├── email_verified
       ├── picture
       ├── locale
       ├── profile
       └── many custom claims
              │
              ▼
        Claim Allowlist
              │
              ▼
        iss + sub
        email_verified
        email if needed
              │
              ▼
        Persist Minimum
No almacenar automáticamente todo el ID Token.

## 388. Ejemplo — Risk Processing

Raw IP
Device Signal
Velocity Signal
Provider Reputation
       │
       ▼
Risk Engine
       │
       ▼
Risk = HIGH
Reasons:
NEW_DEVICE
CREDENTIAL_STUFFING_PATTERN
       │
       ▼
Persist Decision Metadata
       │
       ▼
Expire unnecessary raw inputs

## 389. Comparación con Laravel

Laravel proporciona primitives como:
Authentication
Sessions
Fortify
Sanctum
Notifications
Events
Logging
Cache
Database
Encryption
y permite construir políticas de privacidad alrededor de ellas.
Sin embargo, normalmente la aplicación debe diseñar por separado:
Authentication data classification
field-level redaction
retention
security metadata minimization
consent records
incident holds
safe exports
erasure orchestration
privacy-aware projections
plugin data manifests
VoltStack convertirá estos elementos en arquitectura formal del Authentication Core.

## 390. Comparación con Symfony

Symfony ofrece primitives robustas mediante:
Security
Serializer
EventDispatcher
Messenger
Cache
RateLimiter
Secrets
HttpFoundation
Monolog integrations
y permite implementar privacy/data governance sobre estos componentes.
VoltStack añadirá una capa específicamente orientada a Authentication:
Authentication Data Registry
Purpose Binding
Collection Policies
Progressive Minimization
Retention Compiler
Security Evidence Holds
Privacy-Aware Security Center
Authentication Erasure Planning
Multi-Tenant Privacy Scoping

## 391. Diferenciador VoltStack

Laravel-like Developer Experience
+
Symfony-like Contracts
+
Privacy by Architecture
+
Authentication Data Classification
+
Purpose Limitation
+
Collection Governance
+
Data Minimization
+
Progressive Minimization
+
Retention Governance
+
Consent / Preference Separation
+
Security Evidence Holds
+
Safe Authentication Exports
+
Erasure Orchestration
+
Privacy-Aware Security Center
+
Multi-Tenant Data Isolation
+
Data Residency
+
Outbound Provider Governance
+
Plugin Privacy Manifests
+
Distributed Policy Versions
+
FrankenPHP / Fiber Safety

## 392. Decisiones arquitectónicas definitivas

VoltStack adoptará:

1. Authentication Data Governance será first-class.
2. Todo dato persistido tendrá clasificación.
3. Todo dato persistido tendrá propósito.
4. Todo dato persistido tendrá retention semantics.
5. Processing no implicará persistence.
6. Collection será policy-driven.
7. Optional collection fallará hacia no persistencia.
8. Unknown fields no se loguearán automáticamente.
9. Allowlist será preferida para security event payloads.
10. Secrets nunca entrarán en general telemetry.
11. Debug mode no desactivará redaction.
12. Raw OAuth/OIDC payloads no serán permanent identity records.
13. Federated claims serán allowlisted.
14. Device fingerprinting invasivo no será default.
15. Precise geolocation no será default.
16. Approximate location será sensitive metadata.
17. IP podrá someterse a progressive minimization.
18. User-Agent podrá normalizarse.
19. Security metadata seguirá privacy governance.
20. Risk inputs podrán expirar antes que derived findings.
21. Audit también será minimizado.
22. Audit integrity no significará infinite retention.
23. Security Evidence será distinto de normal telemetry.
24. Incident Holds serán explícitos.
25. Holds serán scoped.
26. Holds serán auditables.
27. Holds deberán revisarse.
28. Retention podrá tener mínimos y máximos.
29. Conflictos de retention serán errores explícitos.
30. Retention policies serán versionadas.
31. Static retention policies podrán compilarse.
32. Policy changes podrán generar migration plans.
33. Cleanup será idempotente.
34. Retention race deberá considerar holds.
35. Consent no será universal legal basis.
36. Consent será purpose-specific.
37. Consent será versionado.
38. Consent withdrawal será propagable.
39. Consent != preference.
40. Consent != authentication enrollment.
41. Mandatory security processing no dependerá ciegamente de consentimiento.
42. Tenant privacy scope será explícito.
43. Tenant A no verá Tenant B.
44. Administrability != unrestricted data visibility.
45. Security Center será viewer-aware.
46. Field-level redaction será soportada.
47. Cache incluirá viewer authority.
48. Privacy exports serán safe projections.
49. Exports nunca contendrán Authentication secrets.
50. Export files serán temporales.
51. Export download podrá requerir reauthentication.
52. Identity deletion generará Authentication Erasure Plan.
53. Erasure será idempotente.
54. Erasure respetará holds.
55. Authentication erasure != global application erasure.
56. Tombstones serán minimalistas.
57. Backups serán parte del lifecycle.
58. Backup restore no resucitará deleted credentials.
59. Erasure Ledger podrá soportar restore safety.
60. Hashing no se considerará automáticamente anonymization.
61. HMAC normalmente será pseudonymization.
62. Crypto-shredding requerirá garantías reales.
63. Data residency será policy-driven.
64. Cross-region transfer será explícito.
65. Third-party transfers serán minimizados.
66. Plugins declararán data capabilities.
67. Custom methods declararán schemas.
68. Privacy policy será compilable.
69. Governance changes serán privileged operations.
70. Policy simulation será soportada.
71. Privacy findings serán first-class.
72. Data inventory será consultable.
73. Metrics evitarán PII/high-cardinality identifiers.
74. Traces serán secret-free.
75. Exceptions serán sanitized.
76. SQL telemetry podrá marcar sensitive bindings.
77. Production Authentication data no deberá copiarse libremente a test.
78. Queue jobs no transportarán raw credentials.
79. Dead-letter queues seguirán retention policy.
80. Privacy Context será request/fiber scoped.
81. FrankenPHP no mantendrá viewer state entre requests.
82. Temporary secret references deberán liberarse cuanto antes.
83. Policy versioning será distribuible.
84. Optional metadata collection deberá detenerse ante policy crítica desconocida.
85. Privacy governance no deberá degradar innecesariamente Authentication availability.
86. Criterios de aceptación
El sistema estará arquitectónicamente completo cuando soporte al menos:
87. Authentication Data Registry.
88. Data Types.
89. Data Classification.
90. Data Purposes.
91. Purpose Binding.
92. Data Schemas.
93. Field Classification.
94. Collection Policy.
95. Collection Decisions.
96. Ephemeral processing.
97. Data Minimization.
98. Progressive Minimization.
99. Redaction.
100. Secret-free events.
101. Secret-free audit.
102. Secret-free telemetry.
103. Sensitive SQL bindings.
104. Authentication Retention Policy.
105. Minimum retention.
106. Maximum retention.
107. Retention conflicts.
108. Retention compilation.
109. Retention policy versioning.
110. Retention migration planning.
111. Async cleanup.
112. Idempotent cleanup.
113. Security Evidence.
114. Incident Holds.
115. Legal/Compliance Holds.
116. Hold expiration/review.
117. Consent Records.
118. Consent purpose.
119. Consent versioning.
120. Consent withdrawal.
121. Consent vs preference separation.
122. Tenant privacy scope.
123. Viewer-aware access.
124. Field-level visibility.
125. Privacy-aware Security Center.
126. Safe Authentication Data Export.
127. Export expiry.
128. Export reauthentication.
129. Privacy Requests.
130. Erasure planning.
131. Erasure orchestration.
132. Partial erasure reporting.
133. Erasure Ledger.
134. Tombstone minimization.
135. Backup lifecycle integration.
136. Restore safety.
137. Pseudonymization.
138. Anonymization.
139. Tokenization extensibility.
140. Crypto-shredding extensibility.
141. Data Residency.
142. Cross-region governance.
143. Outbound Data Policy.
144. Provider minimization.
145. Plugin privacy manifests.
146. Custom method data schemas.
147. Privacy Policy Engine.
148. Compiled Privacy Policy.
149. Governance lifecycle.
150. Privileged policy changes.
151. Policy simulation.
152. Governance scanner.
153. Privacy findings.
154. Authentication Data Inventory.
155. Data lineage metadata.
156. Metrics privacy.
157. Tracing privacy.
158. Exception sanitization.
159. Development/test safeguards.
160. Queue privacy.
161. DLQ privacy.
162. Distributed policy versions.
163. Cache isolation.
164. FrankenPHP isolation.
165. Fiber isolation.
166. Multi-tenant privacy tests.
167. Secret leakage tests.
168. Retention tests.
169. Hold race tests.
170. Erasure tests.
171. Backup restoration tests.
172. Export tests.
173. Plugin governance tests.
174. Policy conflict tests.
175. Data residency tests.
176. Security Center projection tests.
177. Arquitectura final
┌─────────────────────────────────────────────────────────────────────┐
│          AUTHENTICATION PRIVACY & DATA GOVERNANCE                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    Authentication Data                              │
│                            │                                        │
│                            ▼                                        │
│                     Data Registry                                   │
│                            │                                        │
│          ┌─────────────────┼─────────────────┐                      │
│          ▼                 ▼                 ▼                      │
│    Classification       Purpose            Schema                   │
│          │                 │                 │                      │
│          └─────────────────┼─────────────────┘                      │
│                            ▼                                        │
│                     Collection Policy                               │
│                            │                                        │
│        ┌───────────────────┼───────────────────┐                    │
│        ▼                   ▼                   ▼                    │
│      Drop               Ephemeral           Persist                 │
│                                                │                    │
│                                                ▼                    │
│                                         Data Minimization           │
│                                                │                    │
│                    ┌───────────────────────────┼────────────┐       │
│                    ▼                           ▼            ▼       │
│             Operational Data            Security Audit   Evidence   │
│                    │                           │            │       │
│                    └──────────────┬────────────┴────────────┘       │
│                                   ▼                                 │
│                            Retention Engine                          │
│                                   │                                 │
│                ┌──────────────────┼──────────────────┐              │
│                ▼                  ▼                  ▼              │
│             Delete           Pseudonymize          Hold             │
│                │                  │                  │              │
│                └──────────────────┼──────────────────┘              │
│                                   ▼                                 │
│                           Data Lifecycle                             │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                      ACCESS & PROJECTION                            │
│                                                                     │
│ Authentication Data                                                 │
│        ↓                                                            │
│ Viewer + Purpose + Tenant + Realm                                   │
│        ↓                                                            │
│ Privacy Policy                                                      │
│        ↓                                                            │
│ Redaction / Aggregation / Denial                                    │
│        ↓                                                            │
│ Security Center / Admin / Export / API                              │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                         ERASURE                                     │
│                                                                     │
│ Identity Deleted                                                    │
│       ↓                                                             │
│ Authentication Erasure Plan                                         │
│       ├── Secrets → DELETE                                          │
│       ├── Sessions → DELETE                                         │
│       ├── Activity → ANONYMIZE                                      │
│       ├── Audit → PSEUDONYMIZE                                      │
│       └── Evidence → HOLD                                           │
│       ↓                                                             │
│ Erasure Ledger                                                       │
│       ↓                                                             │
│ Backup Restore Protection                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
178. Regla arquitectónica final
El principio central será:
VoltStack Authentication no almacenará información simplemente porque pueda ser útil algún día.

Cada dato deberá responder:
¿Qué es?
   ↓
¿Por qué lo necesitamos?
   ↓
¿Realmente necesitamos almacenarlo?
   ↓
¿Con qué precisión?
   ↓
¿Quién puede verlo?
   ↓
¿En qué tenant/realm aplica?
   ↓
¿Puede salir a un tercero?
   ↓
¿Cuánto tiempo debe existir?
   ↓
¿Qué ocurre cuando expira?
   ↓
¿Qué ocurre cuando la identidad se elimina?
Por tanto:
Authentication Data
        ≠
Permanent Data
y:
Security Metadata
        ≠
Unlimited Surveillance Data
La arquitectura final deberá buscar simultáneamente:
Strong Authentication Security
            +
Incident Investigability
            +
Auditability
            +
Data Minimization
            +
Privacy
            +
Operational Performance
sin sacrificar arbitrariamente uno por otro.

## 396. Integración 36–42

La arquitectura transversal queda:
36 Authentication Policy Engine
          │
          ▼
37 Assurance / Context / Trust
          │
          ▼
38 Challenge Negotiation
          │
          ▼
39 Transaction / Nonce / Replay / CSRF Security
          │
          ▼
40 Notification / Compromise / Incident Response
          │
          ▼
41 Identity Lifecycle
          │
          ▼
42 Privacy / Data Governance
El 42 establece ahora una frontera transversal alrededor de todos los anteriores:
                  PRIVACY GOVERNANCE
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Policy → Assurance → Challenge → Transaction       │
│                ↓                                    │
│  Incident → Identity Lifecycle → Security Center    │
│                                                     │
│  Every Authentication subsystem                    │
│  collects/processes data under governance.          │
│                                                     │
└─────────────────────────────────────────────────────┘

## 397. Siguiente documento

El siguiente documento de la serie debería ser:
43_AUTHENTICATION_NOTIFICATION_SECURITY_COMMUNICATION_OUT_OF_BAND_CHANNEL_AND_DELIVERY_GOVERNANCE_SYSTEM.md
El 40 ya definió qué eventos de seguridad deben generar alertas, mientras que el 43 deberá resolver otra responsabilidad:
cómo transportar comunicaciones de Authentication de manera segura, confiable, privada y verificable.

Deberá cubrir, entre otros:
Email
SMS
Push
In-App Notifications
Security Center
Webhooks
Out-of-Band Challenges
OTP Delivery
Magic Links
Recovery Messages
Security Alerts
Administrative Notifications
Delivery Providers
Fallback Channels
Channel Verification
Rate Limiting
Message Integrity
Anti-Phishing
Sensitive Data Redaction
Delivery Receipts
Retry
Deduplication
Provider Failover
Multi-Tenant Branding
Localization
FrankenPHP / Async Delivery
estableciendo claramente la separación:
40
Security Event / Incident
        │
        │ decides WHAT must be communicated
        ▼
43
Security Communication System
        │
        │ decides HOW / WHERE / WHEN
        ▼
Email / SMS / Push / In-App / Webhook
