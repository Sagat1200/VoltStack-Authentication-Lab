# VoltStack Authentication System

## 33 — Authentication Service, Workload, Machine-to-Machine and Non-Human Identity System

- **Archivo:** `33_AUTHENTICATION_SERVICE_WORKLOAD_MACHINE_TO_MACHINE_AND_NON_HUMAN_IDENTITY_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Dependencias principales:** documentos 04, 06, 07, 08, 09, 10, 14, 20, 24, 25, 27, 28, 29, 30, 31 y 32.

---

## 1. Propósito

Este documento define el subsistema de VoltStack encargado de autenticar identidades que no representan directamente a un usuario humano interactivo.
El sistema deberá tratar como identidades de primera clase:

- Service Accounts
- Machine Identities
- Workload Identities
- Application Identities
- Microservices
- Queue Workers
- Background Workers
- Scheduled Jobs
- CLI Automation
- CI/CD Pipelines
- Deployment Agents
- Infrastructure Agents
- Server-to-Server Clients
- External Integrations

IoT / Edge Devices
Database Agents
Internal Platform Services
La arquitectura deberá evitar el patrón:

```text
Machine
   ↓
Fake User
   ↓
Normal User Authentication
```

y reemplazarlo por:

```text
Non-Human Principal
        ↓
Machine Authentication
        ↓
Workload Identity
        ↓
Authentication Context
        ↓
Authorization
```

## 2. Principio fundamental

Una identidad humana y una identidad de máquina pueden compartir ciertas abstracciones de Authentication, pero no son equivalentes.
Human Identity
│
├── Password
├── Passkey
├── MFA
├── Session
└── Interactive Authentication

Machine Identity
│
├── Certificate
├── Workload Credential
├── Signed Assertion
├── Client Credential
├── mTLS
└── Non-Interactive Authentication
VoltStack deberá modelar ambas bajo un dominio común sin obligarlas a utilizar los mismos mecanismos.

## 3. Objetivo arquitectónico

VoltStack deberá poder responder:

```text
¿Qué servicio está realizando esta solicitud?

¿Qué workload está ejecutándose?

¿Quién emitió su identidad?

¿Qué credencial utilizó?

¿La credencial pertenece realmente a ese workload?

¿En qué entorno está ejecutándose?

¿A qué tenant pertenece?

¿Qué servicio intenta consumir?

¿Qué audience tiene la credencial?

¿Cuándo fue emitida?

¿Cuándo expira?

¿Puede revocarse?
```

¿Es una identidad humana, de servicio, workload o dispositivo?

¿Existe delegación humana detrás de la operación?

## 4. Principal Types

El modelo de identidad deberá distinguir como mínimo:

```php
enum PrincipalType: string
{
    case Human = 'human';

    case Service = 'service';

    case Workload = 'workload';

    case Application = 'application';

    case Device = 'device';

    case Automation = 'automation';

    case ExternalService = 'external_service';
}
```

Esto podrá evolucionar sin romper el modelo principal.

## 5. NonHumanPrincipal

interface NonHumanPrincipalInterface
{
public function id(): string;

public function type(): PrincipalType;

public function identity(): MachineIdentity;
}

## 6. Machine Identity

MachineIdentity representa la identidad lógica de una entidad no humana.
final readonly class MachineIdentity
{
public function __construct(
public string $id,
public string $name,
public PrincipalType $type,
public string $realm,
public ?string $tenantId,
) {}
}

## 7. Machine Identity != Credential

Debe existir separación estricta:

```text
Machine Identity
       ≠
Credential
```

Una identidad puede tener múltiples credenciales.

```text
service.payment
       │
       ├── certificate v12
       ├── certificate v13
       ├── signing key v4
       └── workload federation
```

## 8. Credential Rotation

Gracias a esta separación:

```text
Identity remains stable
        ↓
Credential rotates
        ↓
Identity remains stable
```

No deberá ser necesario crear una identidad nueva por cada rotación.

## 9. Service Account

Un ServiceAccount representa una identidad persistente destinada a software.
final readonly class ServiceAccount
{
public function __construct(
public string $id,
public string $name,
public string $realm,
public ?string $tenantId,
public ServiceAccountStatus $status,
) {}
}

## 10. ServiceAccountStatus

enum ServiceAccountStatus: string
{
case Active = 'active';

case Suspended = 'suspended';

case Disabled = 'disabled';

case Compromised = 'compromised';

case Retired = 'retired';
}

## 11. Service Account no es User

No implementar:

```php
$user = User::create([
    'email' => 'queue-worker@internal',
]);
```

como modelo arquitectónico principal.
Puede existir compatibilidad con sistemas externos que lo requieran, pero internamente deberá conservarse:

- HumanPrincipal
- ≠
- ServicePrincipal

## 12. Workload Identity

Una WorkloadIdentity representa una instancia o clase de workload ejecutándose dentro de una infraestructura.
Ejemplos:

- FrankenPHP Worker
- Queue Worker
- Kubernetes Pod
- Docker Container
- VM
- Serverless Function
- Cron Job
- Deployment Job
- GitHub Actions Runner
- Build Agent

## 13. Workload vs Service

Ejemplo:

```text
Service:
billing-api
```

Workloads:

- billing-api/pod-01
- billing-api/pod-02
- billing-api/pod-03

El servicio es una identidad lógica.
El workload es una ejecución concreta.

## 14. Jerarquía

Service Identity
│
├── Workload A
├── Workload B
└── Workload C

## 15. WorkloadIdentity

final readonly class WorkloadIdentity
{
public function __construct(
public string $id,
public string $serviceId,
public string $environment,
public string $realm,
public ?string $tenantId,
public array $attributes = [],
) {}
}

## 16. Ephemeral Workloads

VoltStack deberá soportar workloads cuya vida sea:

- seconds
- minutes
- hours

No asumir que toda identidad dura años.

## 17. Stable Identity vs Ephemeral Instance

Debe poder distinguir:

```text
ServiceIdentity:
orders-api
```

WorkloadInstance:
orders-api/pod/a7f843

## 18. Authentication Context

Después de autenticarse:

```text
Machine Credential
       ↓
Authenticator
       ↓
Machine Identity Resolution
       ↓
MachineAuthenticationContext
```

## 19. MachineAuthenticationContext

final readonly class MachineAuthenticationContext
{
public function __construct(
public MachineIdentity $identity,
public AuthenticationMethodReference $method,
public DateTimeImmutable $authenticatedAt,
public ?string $credentialId,
public ?string $workloadId,
public ?string $tenantId,
public string $realm,
public array $attributes = [],
) {}
}

## 20. Context deberá ser immutable

Especialmente bajo FrankenPHP.

## 21. Authentication Methods

El subsistema podrá soportar:

- mTLS
- Client Certificate
- OAuth2 Client Credentials
- Private Key JWT
- Signed Request
- Workload Identity Token
- Workload Identity Federation
- SPIFFE-like Identity
- Cloud Provider Identity

Kubernetes Service Account Identity
Unix Socket Peer Identity
Short-Lived Service Token
HMAC Request Signature
Hardware-Backed Machine Credential
No todos deberán implementarse en V1.

## 22. Passwords para Machines

Deberán evitarse como mecanismo principal.

- No:
- service_username
- service_password

cuando pueda utilizarse:

- certificate
- short-lived token
- workload identity
- signed assertion

## 23. Static API Keys

Podrán soportarse por compatibilidad.
Pero deberán clasificarse como mecanismo de menor seguridad.

## 24. API Key

final readonly class ApiKeyCredential
{
public function__construct(
public string $id,
public string $keyId,
public string $secretVerifier,
public DateTimeImmutable $createdAt,
public ?DateTimeImmutable $expiresAt,
) {}
}
Nunca guardar el secret recuperable cuando no sea necesario.

## 25. API Key Structure

Preferible:
vs_live_abc123.xxxxxxxxxxxxxxxxx
donde:
abc123
sea identificador público de la credencial y la segunda parte sea secret.

## 26. Key Lookup

Esto permite:

```text
key id
   ↓
credential record
   ↓
constant-time secret verification
```

sin buscar todos los hashes.

## 27. API Key Prefix

Prefixes pueden identificar:

- environment
- credential class

pero nunca deberán contener secretos.

## 28. API Key Scope

Cada key deberá tener:

- owner identity
- tenant
- realm
- audience
- expiration
- status
- credential version

y Authorization deberá gestionar sus permisos.

## 29. Client Credentials

VoltStack deberá soportar el patrón OAuth2:

```text
client_id
+
client authentication
        ↓
Token Endpoint
        ↓
Short-Lived Access Token
```

## 30. Client Secret

Cuando exista:

- client_secret
- deberá tratarse como credential de alta sensibilidad.

## 31. Private Key JWT

Preferible en muchos escenarios frente a secrets compartidos.
Client
↓
Signed Assertion
↓
Authentication Server
↓
Signature Verification

## 32. Assertion

Debe contener conceptualmente:

- issuer
- subject
- audience
- issued-at
- expiration
- unique identifier

## 33. Replay Protection

jti o equivalente deberá permitir prevenir replay cuando corresponda.

## 34. Assertion Lifetime

Debe ser corto.
No:
signed assertion valid for 30 days

## 35. Audience Binding

Una assertion para:
auth.voltstack.internal
no deberá aceptarse automáticamente en:
billing.voltstack.internal

## 36. Issuer Binding

Debe verificarse issuer esperado.

## 37. Subject Binding

Debe mapear a la Machine Identity correcta.

## 38. Algorithm Policy

Nunca confiar simplemente en:

- algorithm from token header
- El verifier deberá conocer algoritmos permitidos por policy.

## 39. mTLS

Mutual TLS deberá ser un mecanismo de primera clase.

```text
Client
   │
   │ Client Certificate
   ▼
TLS Endpoint
   │
   ▼
Certificate Validation
   │
   ▼
Machine Identity Mapping
```

## 40. mTLS Authentication

Deberá validar:

- certificate chain
- trust anchor
- validity

revocation status where applicable
identity mapping
key usage
extended key usage
expected policy

## 41. TLS termination

Cuando TLS termine en proxy:

```text
Client
  ↓
Trusted Proxy
  ↓
VoltStack
```

VoltStack no deberá confiar ciegamente en:
X-Client-Certificate

## 42. Trusted Proxy Boundary

Solo proxies explícitamente confiables podrán propagar authenticated certificate identity.

## 43. Header Spoofing

Headers de identidad provenientes directamente de internet deberán eliminarse o rechazarse.

## 44. Proxy Authentication Assertion

Idealmente el proxy generará una assertion autenticada o el runtime proporcionará metadata confiable.

## 45. Certificate Identity Mapping

No depender únicamente de:

- Common Name
- cuando existan mecanismos modernos más apropiados.

## 46. Certificate Rotation

Debe soportar:

- Certificate A
- Certificate B

simultáneamente durante ventana de rotación.

## 47. Overlapping Credentials

Esto permite zero-downtime rotation.

## 48. Certificate Revocation

Podrá integrarse con:

- CRL
- OCSP
- internal revocation registry
- security epoch
- dependiendo del PKI.

## 49. Workload Identity Federation

VoltStack deberá poder confiar en identidades emitidas por infraestructura externa sin almacenar secrets permanentes.
Ejemplo:

```text
Cloud/Kubernetes/CI Platform
            ↓
     Signed Workload Token
            ↓
        VoltStack STS
            ↓
     Short-Lived Credential
```

## 50. Security Token Service

Podrá existir:
Authentication Security Token Service
o:
Machine Credential Exchange Service

## 51. Credential Exchange

External Workload Evidence
↓
Evidence Verification
↓
Identity Mapping
↓
Policy Evaluation
↓
Short-Lived VoltStack Credential

## 52. STS Contract

interface SecurityTokenServiceInterface
{
public function exchange(
WorkloadCredentialExchangeRequest $request
): WorkloadCredentialExchangeResult;
}

## 53. Credential Exchange Request

final readonly class WorkloadCredentialExchangeRequest
{
public function __construct(
public AuthenticationEvidence $evidence,
public string $audience,
public ?string $tenantId,
) {}
}

## 54. Short-Lived Credentials

VoltStack deberá preferir:

- long-lived identity
- +;
- short-lived credential

sobre:

- long-lived identity
- +;
- long-lived static secret

## 55. Credential Lifetime

Debe poder ser:

- 5 minutes
- 15 minutes
- 1 hour
- según threat model.

## 56. Automatic Rotation

Short-lived credentials deberán poder renovarse automáticamente cuando la identidad subyacente siga siendo válida.

## 57. Renewal != Infinite Lifetime

Cada renovación deberá volver a comprobar evidencia/policy según corresponda.

## 58. Workload Attestation

Una identidad de workload puede depender de evidencia de ejecución.
Ejemplos conceptuales:

- cloud instance identity
- Kubernetes service account
- container metadata
- signed deployment identity
- hardware attestation
- TPM
- TEE

## 59. WorkloadAttestation

interface WorkloadAttestationInterface
{
public function verify(
WorkloadEvidence $evidence
): WorkloadAttestationResult;
}

## 60. Attestation != Authorization

Que un workload sea auténtico no implica que pueda consumir cualquier servicio.

## 61. Trust Domain

VoltStack deberá poder definir:
TrustDomain
Ejemplos:

- production.voltstack
- staging.voltstack
- tenant-42.voltstack

## 62. TrustDomain

final readonly class TrustDomain
{
public function__construct(
public string $name,
) {}
}

## 63. Cross-Trust-Domain Authentication

Deberá requerir trust explícito.

```text
No:
authenticated somewhere
=

trusted everywhere
```

## 64. Service Identity URI

VoltStack podrá utilizar identificadores estructurados:

- voltstack://production/service/orders
- o modelos compatibles con estándares externos.

## 65. Identity URI

Puede incorporar:

- trust domain
- service
- environment
- tenant
- workload class

sin hacer el identificador innecesariamente mutable.

## 66. SPIFFE Compatibility

La arquitectura deberá permitir integración futura con identidades estilo SPIFFE/SPIRE.
Ejemplo conceptual:

- spiffe://example.org/ns/payments/sa/api
- Sin convertir SPIFFE en dependencia obligatoria del core.

## 67. Kubernetes

Podrá existir:

- KubernetesWorkloadIdentityAuthenticator
- capaz de validar tokens de service accounts según configuración.

## 68. Kubernetes Identity Mapping

Ejemplo:

```text
namespace:
payments
```

service account:

```text
billing-worker
→
```

VoltStack Machine Identity:
service.billing-worker

## 69. Kubernetes Pod Name

No utilizar necesariamente un Pod Name efímero como identidad lógica principal.

## 70. Cloud Identity

Adaptadores podrán soportar:

- AWS workload identity
- Google Cloud workload identity
- Azure managed identity

sin contaminar el dominio central.

## 71. Cloud Provider Adapter

interface ExternalWorkloadIdentityProviderInterface
{
public function verify(
ExternalWorkloadEvidence $evidence
): ExternalWorkloadIdentity;
}

## 72. Provider Independence

Core:
Workload Identity
Adapter:

- AWS
- Azure
- GCP
- Kubernetes
- SPIFFE

## 73. CI/CD Identity

CI pipelines son Machine Principals.

- Ejemplo:
- GitHub Actions
- GitLab CI
- Jenkins
- Buildkite
- Azure DevOps

## 74. CI Long-Lived Secrets

Evitar:

- repository secret:
- PRODUCTION_ROOT_API_KEY

si workload federation está disponible.

## 75. CI Federation

Preferir:

```text
CI OIDC Identity
       ↓
VoltStack Federation
       ↓
Short-Lived Deployment Credential
```

## 76. Repository Binding

Policy podrá comprobar:

- repository
- branch
- environment
- workflow
- organization
- según evidence disponible.

## 77. Deployment Identity

Ejemplo:

- deployment:
- voltstack-framework

environment:
production
workflow:
release

## 78. Branch Protection no es Authentication

Puede contribuir a policy externa, pero no sustituye verificación de identidad del workload.

## 79. Scheduled Jobs

Cron jobs deberán tener Machine Identity explícita.

- No ejecutar automáticamente como:
- system administrator
- sin identidad.

## 80. Scheduler Identity

scheduler:billing-monthly-close
puede ser diferente de:
service:billing-api

## 81. Queue Workers

Cada clase de worker deberá poder tener:

- Machine Identity
- Authentication Context
- Authorization Scope

## 82. Job Actor

Debe distinguirse:
Machine Actor
de:
Original Human Actor

## 83. Delegated Human Context

Ejemplo:

```text
Francisco
   ↓
requests export
   ↓
Queue Job
   ↓
Worker
```

El worker está autenticado como:
service.export-worker
pero puede conservar:
initiated_by = HumanPrincipal

## 84. No Impersonation

El worker no deberá fingir ser el usuario.

## 85. Actor / Subject / Initiator

Audit podrá representar:

```text
Actor:
export-worker
```

Initiator:
user-123

Subject:
report-456

## 86. Delegation Credential

Para algunos workflows podrá emitirse:

- DelegatedOperationCredential
- limitada a una operación.

## 87. DelegatedOperationCredential

final readonly class DelegatedOperationCredential
{
public function __construct(
public string $id,
public string $machineIdentityId,
public ?string $initiatorId,
public string $operation,
public DateTimeImmutable $expiresAt,
) {}
}

## 88. Delegation Scope

Debe ser mínimo.

```text
No:
user requested export
    ↓
```

worker receives full user's permissions

## 89. Confused Deputy

El sistema deberá proteger contra el problema de confused deputy.

## 90. Audience

Toda credencial delegada deberá identificar servicio objetivo cuando sea posible.

## 91. Operation Binding

También podrá identificar:

- operation
- resource
- tenant

## 92. Background Jobs

Un job persistido durante horas no deberá contener una access token que expira en 15 minutos esperando que siga válida.

## 93. Credential Acquisition at Execution

Preferir:

```text
Job starts
   ↓
Worker authenticates
   ↓
```

Worker obtains short-lived credential
↓
Operation

## 94. Human Delegation Revalidation

La autorización humana podrá necesitar revalidarse dependiendo del workflow.

## 95. Machine-to-Machine Authentication

Flujo general:

```text
Service A
    ↓
Acquire Credential
    ↓
Request Service B
    ↓
Service B Authenticator
    ↓
Credential Validation
    ↓
Resolve Service A
    ↓
Machine Authentication Context
    ↓
Authorization
    ↓
Response
```

## 96. No Shared Global Service Secret

Evitar:

- INTERNAL_API_SECRET
- compartido por todos los servicios.

Si se compromete:
all services compromised

## 97. Per-Service Identity

Preferir:

- orders-api
- payments-api
- inventory-api
- notifications-api
- con credenciales independientes.

## 98. Per-Environment Identity

No compartir credential:

- development
- staging
- production

## 99. Environment Isolation

payments-api-dev
≠
payments-api-production
desde perspectiva de trust.

## 100. Tenant Isolation

Una identidad tenant-bound deberá incluir binding explícito.
service:importer
tenant:42

## 101. Cross-Tenant Services

Servicios de plataforma podrán operar sobre múltiples tenants.
Pero deberán declararse:

- PlatformServiceIdentity
- y no simular pertenecer a todos.

## 102. Tenant Authentication Context

final readonly class MachineTenantContext
{
public function __construct(
public ?string $tenantId,
public bool $platformWide,
) {}
}

## 103. Tenant Scope != Authorization

De nuevo:

- tenant binding
- establece contexto.
- Authorization establece permisos.

## 104. Credential Tenant Binding

Credencial para:
tenant A
no deberá aceptarse automáticamente en:
tenant B

## 105. Token Claims

Los tokens machine podrán contener:

- iss
- sub
- aud
- iat
- nbf
- exp
- jti
- tenant
- realm
- credential version
- según formato.

## 106. Minimal Claims

No introducir datos innecesarios.

## 107. Sensitive Claims

No almacenar secretos o información confidencial innecesaria dentro de tokens legibles.

## 108. Token Audience

Obligatoria para tokens internos de servicio cuando sea posible.

## 109. Token Expiration

Machine tokens deberán ser short-lived por default.

## 110. Not Before

Podrá utilizarse para controlar validez temporal.

## 111. Clock Skew

Debe existir tolerancia pequeña y configurable.

## 112. Unlimited Clock Skew

Nunca.

## 113. Token ID

jti puede ayudar en:

- replay detection
- revocation
- audit correlation

## 114. Token Revocation

Short-lived tokens reducen dependencia de revocation, pero no la eliminan para incident response.

## 115. Machine Security Epoch

Cada Machine Identity podrá tener:
securityEpoch

## 116. Security Epoch Example

token epoch:
12

identity epoch:
13

Result:
REJECT

## 117. Credential Version

También:

- credentialVersion
- para invalidación selectiva.

## 118. Identity-Wide Revocation

securityEpoch++
puede invalidar todas las credenciales relacionadas según profile.

## 119. Credential-Specific Revocation

credential A compromised
no necesariamente debe revocar credential B.

## 120. MachineCredential

Modelo:

```php
interface MachineCredentialInterface
{
    public function id(): string;

    public function machineIdentityId(): string;

    public function type(): MachineCredentialType;

    public function status(): MachineCredentialStatus;
}
```

## 121. MachineCredentialType

enum MachineCredentialType: string
{
case ApiKey = 'api_key';

case ClientSecret = 'client_secret';

case Certificate = 'certificate';

case SigningKey = 'signing_key';

case WorkloadIdentity = 'workload_identity';

case Federated = 'federated';

case HardwareBacked = 'hardware_backed';
}

## 122. Credential Status

enum MachineCredentialStatus: string
{
case Active = 'active';

case Rotating = 'rotating';

case Expired = 'expired';

case Revoked = 'revoked';

case Compromised = 'compromised';

case Retired = 'retired';
}

## 123. Credential Lifecycle

CREATED
↓
ACTIVE
↓
ROTATING
↓
RETIRED
Alternativas:

- REVOKED
- COMPROMISED
- EXPIRED

## 124. Credential Metadata

Puede incluir:

- createdAt
- activatedAt
- expiresAt
- lastUsedAt
- lastRotatedAt
- issuer
- keyId
- algorithm

sin incluir material secreto en logs.

## 125. Last Used

Útil para detectar credenciales abandonadas.

## 126. Credential Inventory

VoltStack deberá poder responder:

```text
¿Qué credenciales existen?

¿Quién es su owner?

¿Cuándo expiran?

¿Cuándo se utilizaron?

¿Qué servicios dependen de ellas?

¿Cuáles nunca se han rotado?
```

## 127. Credential Ownership

Toda credencial deberá tener owner explícito.
No:
mystery-api-key-17

## 128. Credential Description

Podrá tener metadata:

- purpose
- environment
- service
- owner team
- rotation policy

## 129. Credential Rotation Policy

final readonly class MachineCredentialRotationPolicy
{
public function __construct(
public ?DateInterval $maximumAge,
public ?DateInterval $rotationOverlap,
public bool $automaticRotation,
) {}
}

## 130. Rotation Workflow

Credential A Active
↓
Generate Credential B
↓
A + B valid
↓
Deploy B
↓
Observe B usage
↓
Revoke A

## 131. No Hard Cutover by Default

Evitar outages durante rotation.

## 132. Automatic Rotation

Integración con:

- KMS
- Vault
- Cloud Secret Manager
- PKI
- HSM
- mediante providers.

## 133. Secret Storage

Documento 31 define infraestructura criptográfica.

- Machine credentials deberán utilizar:
- Secret Store
- KMS
- HSM
- Environment Injection
- Mounted Secret
- Runtime Credential Provider
- según tipo.

## 134. Secrets in Source Code

Prohibido.

## 135. Secrets in Git

Prohibido.

## 136. Secrets in Container Image

Deberá evitarse.

## 137. Secrets in Configuration Cache

No introducir plaintext credentials en caches compilados.

## 138. Environment Variables

Podrán soportarse, pero no deberán considerarse almacenamiento ideal para todos los threat models.

## 139. Runtime Credential Provider

Preferir cuando infraestructura lo permita:

```text
Application
    ↓
Credential Provider
    ↓
Short-Lived Credential
```

## 140. CredentialProvider

interface MachineCredentialProviderInterface
{
public function acquire(
MachineCredentialRequest $request
): MachineCredentialResult;
}

## 141. Credential Request

final readonly class MachineCredentialRequest
{
public function__construct(
public string $audience,
public ?string $tenantId,
public array $requestedCapabilities = [],
) {}
}

## 142. Least Privilege

Credential issuance deberá aplicar:

- minimum audience
- minimum lifetime
- minimum tenant scope
- minimum capabilities

## 143. Authentication vs Capability

Authentication identifica al servicio.
Capabilities pueden limitar qué operación específica puede realizar.
Authorization System deberá gobernarlas.

## 144. Authentication Assurance para Machines

El documento 32 introdujo assurance para humanos.
Machines también deberán tener assurance.

## 145. MachineAuthenticationAssurance

Ejemplo:

- STATIC_SECRET
- SIGNED_CREDENTIAL
- MUTUAL_TLS
- WORKLOAD_ATTESTED
- HARDWARE_BACKED

No necesariamente será una escala lineal universal.

## 146. Assurance Properties

Es mejor modelar propiedades:

- shortLived
- hardwareBacked
- phishingIrrelevant
- workloadBound
- channelBound
- nonExportable
- attested
- rotatable
- revocable

## 147. Machine Assurance Profile

final readonly class MachineAuthenticationAssuranceProfile
{
public function __construct(
public bool $shortLived,
public bool $workloadBound,
public bool $hardwareBacked,
public bool $attested,
public bool $revocable,
) {}
}

## 148. Sensitive Machine Operations

Ejemplo:

- deploy production
- rotate platform key
- access backup
- restore database
- issue credentials
- change authentication policy

pueden exigir machine assurance más fuerte.

## 149. No Interactive MFA

No solicitar:

- Enter TOTP:
- a un queue worker.

## 150. Machine Step-Up

Step-Up para máquinas significa adquirir evidencia/credential más fuerte.
Ejemplo:

```text
normal service token
        ↓
critical operation
        ↓
```

mTLS + workload attestation
↓
high-assurance machine context

## 151. Machine Reauthentication

Podrá consistir en:

- credential refresh
- new signed assertion
- new certificate proof
- workload re-attestation

## 152. Sensitive Operation Integration

Documento 32 deberá aceptar:

- HumanAuthenticationRequirement
- MachineAuthenticationRequirement

o un requirement genérico capaz de discriminar principal.

## 153. MachineAuthenticationRequirement

final readonly class MachineAuthenticationRequirement
{
public function __construct(
public bool $requireShortLivedCredential,
public bool $requireWorkloadBinding,
public bool $requireAttestation,
public bool $requireHardwareBackedKey,
public ?DateInterval $maximumCredentialAge,
) {}
}

## 154. Service-to-Service Authorization

Después de Authentication:
payments-api authenticated
Authorization decide:
may payments-api invoke refunds?

## 155. Service Identity Permissions

No pertenecen a este documento, pero deberán poder expresarse mediante el Authorization System.

## 156. Roles para Services

Podrán existir si el Authorization System lo soporta.
Pero evitar asumir que todo service necesita un human-style role.

## 157. Capabilities

Para M2M pueden resultar más naturales:

- invoice.read
- invoice.create
- payment.capture

## 158. Relationship-Based Authorization

También puede aplicar:

- service belongs to tenant
- service owns resource

service delegated by workflow

## 159. Delegation

Machine A puede solicitar que Machine B ejecute una operación.
Esto deberá ser explícito.

## 160. Delegation Chain

Human
↓
Frontend Service
↓
API Service
↓
Worker
↓
Storage Service
Audit deberá poder conservar cadena cuando sea necesario.

## 161. Authentication Chain

No significa que todos impersonen al primer actor.

## 162. Actor Chain

final readonly class AuthenticationActorChain
{
public function __construct(
public array $actors,
) {}
}

## 163. Chain Depth

Deberá limitarse para evitar:
unbounded delegation

## 164. Delegation Policy

Puede controlar:

- who can delegate
- what can be delegated
- to whom
- for how long
- for which tenant

## 165. Original Actor

Nunca perderlo cuando sea necesario para audit.

## 166. Token Exchange

VoltStack podrá soportar patrones similares a OAuth token exchange.
Token A
↓
Exchange
↓
Token B
con audience y scope reducidos.

## 167. No Privilege Amplification

Token exchange jamás deberá producir permisos superiores a los permitidos por policy.

## 168. Downscoping

Preferir:

```text
broad internal credential
        ↓
downscope
        ↓
single-service credential
```

cuando arquitectura lo permita.

## 169. Service Mesh

VoltStack deberá poder integrarse con service meshes.

- Ejemplos conceptuales:
- Envoy
- Istio
- Linkerd
- sin requerirlos.

## 170. Mesh Authentication

Si mesh realiza mTLS:

```text
Mesh
  ↓
Authenticated Peer
  ↓
VoltStack
```

VoltStack deberá disponer de una frontera confiable para importar esa identidad.

## 171. Application-Layer Authentication

Mesh mTLS no necesariamente elimina necesidad de application-layer credentials.
Dependerá del threat model.

## 172. Defense in Depth

Puede existir:

- mTLS
- +;
- signed application token

## 173. Channel Binding

Credenciales avanzadas podrán vincularse al canal cuando protocolo lo soporte.

## 174. Request Signing

VoltStack podrá soportar:

```text
HTTP Request
     ↓
Canonicalization
     ↓
Signature
     ↓
Verification
```

## 175. SignedRequest

final readonly class SignedRequestEvidence
{
public function__construct(
public string $keyId,
public string $algorithm,
public string $signature,
public DateTimeImmutable $createdAt,
public string $nonce,
) {}
}

## 176. Canonical Request

Debe incluir según protocolo:

- method
- path
- selected headers
- body digest
- timestamp
- nonce
- audience

## 177. Request Tampering

Modificar payload invalidará firma.

## 178. Replay Protection

Signed requests deberán utilizar:

- timestamp
- nonce
- request id

cuando policy lo requiera.

## 179. Nonce Store

interface MachineNonceStoreInterface
{
public function consume(
string $identityId,
string $nonce,
DateTimeImmutable $expiresAt
): bool;
}

## 180. Distributed Nonce Store

En cluster deberá ser compartido cuando replay protection sea global.

## 181. HMAC

Podrá soportarse para integraciones simples.

## 182. HMAC Shared Secret

Tiene limitación:

- both sides know same secret
- y complica attribution si se comparte entre múltiples clients.

## 183. Asymmetric Signing

Preferible cuando se necesite:

- separate signing/verifying roles
- better attribution
- easier verifier distribution

## 184. Key IDs

Toda clave deberá tener:

- kid
- o identificador equivalente.

## 185. Key Rotation

Verifier deberá aceptar múltiples claves activas durante transición.

## 186. Algorithm Migration

También deberá soportarse:

```text
algorithm A
   ↓
A + B
   ↓
algorithm B
```

## 187. Trust Store

Documento 31.

- Machine Authentication necesitará:
- trusted issuers
- trusted certificates
- trusted public keys
- trusted workload providers

## 188. TrustStore

interface MachineTrustStoreInterface
{
public function resolve(
MachineTrustQuery $query
): MachineTrustMaterial;
}

## 189. Trust Versioning

Cambios de trust deberán versionarse.

## 190. Trust Revocation

Eliminar issuer comprometido deberá propagarse rápidamente.

## 191. Unknown Issuer

Fail closed.

## 192. Unknown Key

Fail closed.

## 193. Expired Certificate

Fail closed.

## 194. Invalid Signature

Fail closed.

## 195. Audience Mismatch

Fail closed.

## 196. Tenant Mismatch

Fail closed.

## 197. Realm Mismatch

Fail closed.

## 198. Environment Mismatch

Fail closed.

## 199. Workload Mismatch

Fail closed.

## 200. Authentication Result

final readonly class MachineAuthenticationResult
{
public function __construct(
public bool $authenticated,
public ?MachineAuthenticationContext $context,
public ?MachineAuthenticationFailure $failure,
) {}
}

## 201. MachineAuthenticator

interface MachineAuthenticatorInterface
{
public function supports(
MachineAuthenticationRequest $request
): bool;

public function authenticate(
MachineAuthenticationRequest $request
): MachineAuthenticationResult;
}

## 202. Authenticators

Implementaciones:

- ApiKeyAuthenticator
- ClientSecretAuthenticator
- PrivateKeyJwtAuthenticator
- MutualTlsAuthenticator
- SignedRequestAuthenticator
- WorkloadIdentityAuthenticator
- FederatedWorkloadAuthenticator

## 203. Resolver

Documento 07.

```text
Request
  ↓
Machine Authenticator Resolver
  ↓
Applicable Authenticators
  ↓
Priority / Policy
```

## 204. Multiple Credentials

Un request puede contener:

- mTLS
- +;
- Bearer Token

## 205. Credential Combination Policy

VoltStack deberá definir si:

- both must agree
- one is sufficient
- one strengthens another

## 206. Identity Conflict

Si:

- mTLS says service A
- token says service B

Resultado default:
REJECT

## 207. IdentityAgreementPolicy

interface MachineIdentityAgreementPolicyInterface
{
public function evaluate(
array $authenticatedEvidence
): MachineIdentityAgreementDecision;
}

## 208. Credential Confusion

Nunca elegir silenciosamente una identidad cuando dos credenciales válidas contradicen.

## 209. Credential Precedence

Debe ser explícita.

## 210. Bearer Tokens

Bearer significa:

```php
possession = authority to present
por lo que deben protegerse contra robo.
```

## 211. Sender-Constrained Credentials

Cuando sea posible, soportar credenciales vinculadas al cliente:

- mTLS-bound token
- proof-of-possession
- signed request

## 212. Proof of Possession

Reduce riesgo de token robado.

## 213. DPoP-Like Integration

Arquitectura podrá soportar mecanismos proof-of-possession sin acoplar core a uno específico.

## 214. Machine Session

Por default, M2M deberá ser stateless o short-lived.
No necesita session cookie humana.

## 215. Stateful Machine Sessions

Podrán existir para protocolos específicos, pero no serán modelo default.

## 216. Remember-Me

No aplica.

## 217. Login Form

No aplica.

## 218. Password Reset

No aplica.

- Machine credentials utilizan:
- rotation
- replacement
- revocation
- reissuance

## 219. MFA

No aplica en sentido humano.
Puede existir multi-evidence machine authentication.

## 220. Machine Multi-Factor Evidence

Ejemplo:

- mTLS certificate
- +;
- workload attestation

## 221. Independent Evidence

Policy puede requerir evidencia proveniente de trust roots distintos.

## 222. Device Identity

Los dispositivos no humanos podrán usar parte de esta arquitectura.

## 223. Device vs Workload

Device:
physical/logical equipment identity

Workload:

- software execution identity
- No siempre son equivalentes.

## 224. Hardware Identity

Puede utilizar:

- TPM
- Secure Enclave
- HSM
- hardware certificate

## 225. Non-Exportable Keys

Deberán poder marcarse como propiedad de assurance.

## 226. Attestation

Podrá probar:

- key generated in trusted hardware
- workload running in approved environment
- dependiendo del provider.

## 227. Trust in Attestation

No será universal.
Cada attestation provider tendrá trust policy.

## 228. Authentication Eligibility

Documento 10 deberá extenderse a machine identities.

## 229. Machine Security State

Ejemplo:

- ACTIVE
- DISABLED
- SUSPENDED
- COMPROMISED
- RETIRED

## 230. Compromised Identity

Debe producir rechazo incluso si credential criptográficamente válida.

## 231. Credential Compromise

Puede afectar solo una credential.

## 232. Service Retirement

Retirar servicio deberá revocar:

- credentials
- tokens
- trust mappings
- active machine sessions
- según policy.

## 233. Tenant Suspension

Machine identities tenant-bound deberán dejar de autenticarse o recibir contexto restringido según policy.

## 234. Environment Shutdown

Credenciales de un entorno retirado deberán invalidarse.

## 235. MachineIdentityProvider

interface MachineIdentityProviderInterface
{
public function findById(
string $id
): ?MachineIdentity;

public function findByCredential(
MachineCredentialReference $credential
): ?MachineIdentity;

}

## 236. Provider Sources

Podrán ser:

- Database
- Configuration
- Directory
- Cloud IAM
- Service Registry
- External Identity Provider

## 237. Federated Mapping

Documento 09.

```text
External identity:
GitHub workflow
→
VoltStack deployment identity
```

debe utilizar mapping explícito.

## 238. No Auto-Provisioning Wildcard

Evitar:

```text
any authenticated GitHub workflow
=

production deployer
```

## 239. Mapping Constraints

Deberá poder comprobar:

- issuer
- organization
- repository
- environment
- subject
- audience

## 240. Mapping Version

Debe ser auditable.

## 241. Authentication Events

Documento 23.

```text
Eventos:
MachineAuthenticationStarted
MachineAuthenticationSucceeded
MachineAuthenticationFailed

MachineCredentialIssued
MachineCredentialRotated
MachineCredentialRevoked
MachineCredentialExpired

WorkloadIdentityVerified
WorkloadAttestationFailed

MachineTokenIssued
MachineTokenExchanged
MachineTokenRevoked

MachineIdentityDisabled
MachineIdentityCompromised
```

## 242. Credential Material in Events

Nunca.

## 243. Audit

Documento 24.

- Registrar:
- machine identity
- service
- workload
- tenant
- realm
- environment
- credential id
- credential type
- issuer
- audience
- authentication method
- source network metadata
- result
- failure category
- timestamp
- sin material secreto.

## 244. Machine Audit Identity

No registrar simplemente:
SYSTEM
cuando se conoce:
service.invoice-worker

## 245. SYSTEM Actor

Debe reservarse para operaciones realmente internas sin principal independiente.

## 246. Trace Propagation

Distributed tracing podrá incluir:

- authenticated.service
- authenticated.workload
- authentication.method

con controles de cardinalidad y privacidad.

## 247. Authentication Trace

service A
↓
service B
↓
service C
deberá permitir correlacionar actores sin confiar en headers arbitrarios.

## 248. Trace Context != Authentication

traceparent no autentica identidad.

## 249. Forwarded Identity

Headers como:

- X-Service-Name
- X-User-ID

no deberán aceptarse como Authentication Evidence salvo frontera confiable explícita.

## 250. Audit Delegation Chain

Cuando corresponda:

```text
Human user-123
  ↓
api-gateway
  ↓
orders-api
  ↓
orders-worker
```

## 251. Logging

No registrar:

- API keys
- client secrets
- private keys
- bearer tokens
- signed assertions completas
- certificate private material

## 252. Token Fingerprint

Puede registrarse:

- credential id
- token jti
- non-secret fingerprint

## 253. Metrics

Ejemplos:

```text
auth_machine_attempt_total
auth_machine_success_total
auth_machine_failure_total

auth_machine_credential_issued_total
auth_machine_credential_rotated_total
auth_machine_credential_revoked_total

auth_workload_attestation_total
auth_workload_attestation_failure_total

auth_machine_token_exchange_total
auth_machine_replay_detected_total
```

## 254. Labels

authenticator
credential_type
realm
environment
outcome
failure_category
con cardinalidad controlada.

## 255. No Credential ID como Metric Label

Puede causar cardinalidad excesiva.

## 256. Risk Engine

Documento 20.

- Machine Authentication deberá generar señales:
- new source network
- unexpected environment

credential used from multiple regions
impossible workload location
old credential
unexpected audience
replay
abnormal request volume
certificate near expiry

## 257. Machine Risk

No asumir los mismos modelos que usuarios humanos.

## 258. Impossible Travel

Puede no tener sentido para algunos workloads.
Pero:

- same machine credential used simultaneously in two cloud regions
- puede ser una señal crítica.

## 259. Credential Usage Baseline

Risk Engine podrá aprender:

- usual audiences
- usual source networks
- usual environments
- usual request volume

## 260. Adaptive Machine Authentication

Risk alto podrá exigir:

- stronger credential
- re-attestation
- credential rotation
- manual approval
- deny

## 261. Throttling

Documento 19.

- Machine endpoints también requieren:
- rate limits
- credential failure limits
- replay detection
- abuse protection

## 262. Machine Lockout

Debe diseñarse cuidadosamente para evitar DoS.

## 263. Invalid Credential Flood

No permitir que atacante deshabilite servicio legítimo simplemente enviando credenciales incorrectas con su client ID.

## 264. Rate Limit Dimensions

Podrán incluir:

- source
- credential identifier
- service identity
- tenant
- endpoint
- audience
- según riesgo.

## 265. Authentication Failure Taxonomy

Documento 25.

```text
MACHINE_IDENTITY_NOT_FOUND
MACHINE_IDENTITY_DISABLED
MACHINE_IDENTITY_COMPROMISED

MACHINE_CREDENTIAL_INVALID
MACHINE_CREDENTIAL_EXPIRED
MACHINE_CREDENTIAL_REVOKED
MACHINE_CREDENTIAL_COMPROMISED

MACHINE_SIGNATURE_INVALID
MACHINE_CERTIFICATE_INVALID
MACHINE_CERTIFICATE_EXPIRED

MACHINE_ISSUER_UNTRUSTED
MACHINE_AUDIENCE_MISMATCH
MACHINE_TENANT_MISMATCH
MACHINE_REALM_MISMATCH
MACHINE_ENVIRONMENT_MISMATCH

WORKLOAD_ATTESTATION_FAILED
WORKLOAD_IDENTITY_MISMATCH

MACHINE_TOKEN_REPLAYED
MACHINE_ASSERTION_EXPIRED

MACHINE_AUTHENTICATION_ASSURANCE_INSUFFICIENT
```

## 266. External Failure Responses

No revelar detalles internos innecesarios.

## 267. Internal Explainability

Audit podrá conocer:

- certificate chain invalid
- issuer not trusted
- tenant mismatch
- security epoch mismatch

## 268. Error Response

Cliente puede recibir:

- invalid_client
- invalid_token
- authentication_failed
- según protocolo.

## 269. Distributed Runtime

Documento 30.

- Machine Authentication deberá funcionar en:
- multiple application nodes
- multiple regions
- queue clusters
- FrankenPHP workers

## 270. Shared State

Cuando sea necesario:

- revocation
- nonce consumption
- credential status
- security epochs
- trust versions
- deberán propagarse.

## 271. Replay Store

Debe tener atomicidad.
consume nonce
debe ser una operación atómica.

## 272. Race Condition

Dos requests con mismo nonce concurrentemente:

- one accepted
- one rejected

## 273. Regional Consistency

Para credenciales críticas puede requerirse consistencia más fuerte.

## 274. Revocation Latency

Debe ser configurable y observable.

## 275. Short-Lived Token Tradeoff

Permite reducir necesidad de consultas de revocation por request.

## 276. Online Validation

Para operaciones críticas podrá exigirse:

- current identity state
- current credential state
- current security epoch

## 277. Offline Validation

Puede utilizarse para requests normales cuando threat model lo permita.

## 278. Hybrid Validation

Ejemplo:

- signature validation locally
- +;
- cached security epoch
- +;
- short token lifetime

## 279. Fail Closed

En operaciones sensibles, si trust material requerido no está disponible:
deny

## 280. Controlled Degradation

Para workloads no críticos puede existir policy de degradación explícita.
Nunca accidental.

## 281. FrankenPHP

Long-lived workers hacen obligatorio aislar:

- MachineAuthenticationContext
- MachineCredentialContext
- DelegationContext
- WorkloadContext
- por request/fiber.

## 282. No Static Principal

Nunca:

```php
static $machineIdentity;
como estado autenticado actual.
```

## 283. Worker Leakage

Request de:
service A
jamás deberá contaminar request posterior de:
service B

## 284. Context Reset

Al finalizar request:

```text
machine identity
credential metadata
delegation chain
trust evaluation
risk state

deben liberarse.
```

## 285. Fiber Local Context

El runtime deberá soportar aislamiento en concurrencia.

## 286. Immutable Snapshot

Authentication Context deberá representar snapshot de la decisión.

## 287. Live State

Para operaciones críticas puede volver a consultar security state.

## 288. Configuration

Ejemplo:

```php
return [

    'machine_authentication' => [

        'default_realm' => 'services',

        'credentials' => [
            'api_keys' => true,
            'client_secrets' => true,
            'private_key_jwt' => true,
            'mtls' => true,
            'workload_identity' => true,
        ],

        'tokens' => [
            'default_lifetime' => '15 minutes',
        ],

        'security' => [
            'require_audience' => true,
            'allow_static_secrets' => false,
        ],

    ],

];
```

## 289. Environment Policy

'environments' => [

'production' => [
'allow_api_keys' => false,
'allow_client_secrets' => false,
'require_short_lived' => true,
],

'development' => [
'allow_api_keys' => true,
],

];

## 290. Security Floor

Production policy podrá imponer:

- no static credentials
- short-lived tokens
- audience binding
- tenant binding
- trusted issuer

## 291. Tenant Policy

Tenant podrá endurecer.
No reducir platform floor.

## 292. Compilation

Documento 27.

- Podrá compilarse:
- machine identity mappings
- issuer policies
- audience rules
- authenticator routing
- trust metadata references
- credential policies

## 293. No Compile Secrets

Nunca insertar:

- private key
- client secret
- API secret

dentro de configuration cache generada.

## 294. Trust Metadata Cache

Puede cachearse cuidadosamente:

- public keys
- certificates
- issuer metadata
- policy
- con versioning.

## 295. JWKS-like Rotation

Cuando provider publique claves:

- kid unknown
- podrá provocar refresh controlado.

## 296. Refresh Storm

Evitar que miles de requests con kid falso provoquen refresh masivo.

## 297. Negative Caching

Puede utilizarse con TTL pequeño para unknown key IDs.

## 298. Key Fetch Security

Trust material remoto deberá descargarse únicamente desde endpoints configurados/confiables.

## 299. No URL from Token

Nunca:

- token says fetch keys from this URL
- sin trust policy.

## 300. SSRF Protection

Discovery de providers externos deberá proteger contra SSRF.

## 301. Extensibility

Documento 28.

- Providers podrán añadir:
- MachineAuthenticator
- MachineCredentialProvider
- WorkloadIdentityProvider
- WorkloadAttestationProvider
- MachineTrustProvider
- MachineIdentityMapper
- MachineRiskContributor

## 302. Provider Contract

Plugins deberán utilizar contratos del core.

## 303. No Privileged Plugin Shortcut

Plugin no podrá:

```php
return authenticated = true
sin producir Authentication Evidence/Context válido según contrato.
```

## 304. Provider Capabilities

Cada provider deberá declarar:

- credential types
- assurance properties
- rotation support
- revocation support
- attestation support

## 305. Testing

Documento 26 deberá incorporar suite M2M.

## 306. Test — Service Account

Service Account válido autentica correctamente.

## 307. Test — Disabled Service

Credential válida + identity disabled:
REJECT

## 308. Test — Compromised Service

Rejected.

## 309. Test — Expired Credential

Rejected.

## 310. Test — Revoked Credential

Rejected.

## 311. Test — API Key Hash

Secret plaintext nunca se persiste.

## 312. Test — API Key Timing

Verification utiliza comparación segura.

## 313. Test — Wrong Audience

Rejected.

## 314. Test — Wrong Tenant

Rejected.

## 315. Test — Wrong Realm

Rejected.

## 316. Test — Wrong Environment

Rejected.

## 317. Test — Invalid Signature

Rejected.

## 318. Test — Wrong Issuer

Rejected.

## 319. Test — Unknown Key

Rejected.

## 320. Test — Expired Certificate

Rejected.

## 321. Test — mTLS Mapping

Certificate correcto resuelve identidad correcta.

## 322. Test — Proxy Spoofing

Internet client no puede falsificar certificate headers.

## 323. Test — Private Key JWT Replay

Assertion replay se rechaza según policy.

## 324. Test — Assertion Lifetime

Long-lived assertion no aceptada si excede máximo.

## 325. Test — Workload Federation

Evidence externa produce identidad correcta.

## 326. Test — Mapping Wildcard

Issuer válido pero repository no autorizado:
REJECT

## 327. Test — CI Production

Workflow no aprobado no obtiene production credential.

## 328. Test — Short-Lived Credential

Expira correctamente.

## 329. Test — Credential Rotation

A + B funcionan durante overlap.

## 330. Test — Old Credential Retirement

A deja de funcionar después de rotation.

## 331. Test — Security Epoch

Token antiguo se invalida.

## 332. Test — Credential Version

Revocación selectiva funciona.

## 333. Test — Identity Conflict

mTLS = A y token = B:
REJECT

## 334. Test — Delegation

Worker conserva initiator sin impersonarlo.

## 335. Test — Delegation Scope

Worker no obtiene permisos adicionales.

## 336. Test — Token Exchange

Nunca amplifica privilege.

## 337. Test — Nonce Replay

Primera solicitud:
ACCEPT
segunda:
REJECT

## 338. Test — Concurrent Replay

Dos nodos consumen mismo nonce:
exactly one succeeds

## 339. Test — Revocation Cluster

Revocación se propaga.

## 340. Test — Trust Rotation

Old/new trust material funciona durante ventana definida.

## 341. Test — Trust Revocation

Issuer comprometido deja de funcionar.

## 342. Test — Unknown KID Flood

No provoca fetch storm.

## 343. Test — SSRF

Token no puede controlar URL de key retrieval.

## 344. Test — FrankenPHP

Machine identity no se filtra entre requests.

## 345. Test — Fiber Concurrency

Service A y B concurrentes mantienen contexts separados.

## 346. Test — Queue Worker

Job A no hereda initiator/context de Job B.

## 347. Test — Machine Sensitive Operation

Weak credential no satisface strong machine requirement.

## 348. Test — Machine Step-Up

Re-attestation permite operación crítica cuando policy lo permite.

## 349. Test — Static Credential Production Policy

API key rechazada si production la prohíbe.

## 350. Test — Tenant Security Floor

Tenant no debilita policy global.

## 351. Security Invariants — Identity

AUTH-MACHINE-ID-01
Toda entidad no humana autenticada tiene PrincipalType explícito.
AUTH-MACHINE-ID-02
Una Machine Identity no se modela internamente como usuario humano ficticio.
AUTH-MACHINE-ID-03
Identity y Credential son conceptos independientes.

- AUTH-MACHINE-ID-04
- La rotación de credencial no cambia automáticamente la identidad lógica.
- AUTH-MACHINE-ID-05

Toda credencial tiene owner identificable.

## 352. Security Invariants — Credentials

AUTH-MACHINE-CRED-01
Credenciales secretas nunca se registran en logs.

- AUTH-MACHINE-CRED-02
- Credenciales revocadas no autentican.
- AUTH-MACHINE-CRED-03

Credenciales expiradas no autentican.

- AUTH-MACHINE-CRED-04
- Credenciales comprometidas no autentican.
- AUTH-MACHINE-CRED-05

Secrets no forman parte de caches compilados.

- AUTH-MACHINE-CRED-06
- Credential rotation soporta transición segura.
- AUTH-MACHINE-CRED-07

Credenciales estáticas no son el mecanismo preferido.

## 353. Security Invariants — Tokens

AUTH-MACHINE-TOKEN-01
Todo token tiene lifetime limitado.

- AUTH-MACHINE-TOKEN-02
- Audience se valida cuando policy lo requiere.
- AUTH-MACHINE-TOKEN-03

Issuer se valida explícitamente.

- AUTH-MACHINE-TOKEN-04
- Algoritmo aceptado viene de policy, no del atacante.
- AUTH-MACHINE-TOKEN-05

Tenant/realm/environment binding se valida cuando exista.
AUTH-MACHINE-TOKEN-06
Token exchange nunca amplifica privilegios.

## 354. Security Invariants — Workload

AUTH-MACHINE-WORKLOAD-01
Workload identity y service identity son distinguibles.

- AUTH-MACHINE-WORKLOAD-02
- Workload attestation no sustituye Authorization.
- AUTH-MACHINE-WORKLOAD-03

External workload identities requieren mapping explícito.

- AUTH-MACHINE-WORKLOAD-04
- Trust entre dominios nunca es implícito.
- AUTH-MACHINE-WORKLOAD-05

Un workload efímero no requiere una credencial estática de larga duración.

## 355. Security Invariants — M2M

AUTH-MACHINE-M2M-01
Cada servicio deberá poder tener identidad independiente.
AUTH-MACHINE-M2M-02
No existe un secret global compartido como identidad de todos los servicios.
AUTH-MACHINE-M2M-03
Dos credenciales contradictorias producen rechazo por default.
AUTH-MACHINE-M2M-04
Forwarded headers no son evidencia salvo frontera confiable.
AUTH-MACHINE-M2M-05
Trace context nunca se considera Authentication Evidence.

## 356. Security Invariants — Delegation

AUTH-MACHINE-DEL-01
Machine Actor y Human Initiator permanecen distinguibles.

- AUTH-MACHINE-DEL-02
- Background workers no impersonan automáticamente al usuario iniciador.
- AUTH-MACHINE-DEL-03

Delegation tiene scope y lifetime limitados.

- AUTH-MACHINE-DEL-04
- Delegation nunca amplifica privilegios.
- AUTH-MACHINE-DEL-05

La cadena de actores tiene profundidad controlada.

## 357. Security Invariants — Multi-Tenant

AUTH-MACHINE-TENANT-01
Credential de Tenant A no autentica automáticamente en Tenant B.
AUTH-MACHINE-TENANT-02
Platform Service y Tenant Service son contextos distinguibles.
AUTH-MACHINE-TENANT-03
Tenant puede endurecer pero no debilitar platform security floor.
AUTH-MACHINE-TENANT-04
Cross-tenant machine access debe ser explícito.

## 358. Security Invariants — Runtime

AUTH-MACHINE-RT-01
Machine Authentication Context nunca vive en estado global mutable.
AUTH-MACHINE-RT-02
FrankenPHP workers limpian machine context entre requests.

- AUTH-MACHINE-RT-03
- Concurrent requests mantienen identities aisladas.
- AUTH-MACHINE-RT-04

Queue jobs no heredan accidentalmente Authentication Context anterior.

## 359. Security Invariants — Distributed

AUTH-MACHINE-DIST-01
Replay stores usan operaciones atómicas.

- AUTH-MACHINE-DIST-02
- Revocación se propaga según security policy.
- AUTH-MACHINE-DIST-03

Trust changes se versionan.
AUTH-MACHINE-DIST-04
Una falla de trust validation crítica produce fail-closed.

## 360. Anti-Patterns

Nunca:
Machine = User

## 361. Anti-Pattern

all microservices
↓
same API key

## 362. Anti-Pattern

production secret
stored in Git

## 363. Anti-Pattern

API key never expires
never rotates

## 364. Anti-Pattern

Bearer token
valid forever

## 365. Anti-Pattern

token says alg=...
server accepts it
sin policy.

## 366. Anti-Pattern

token says key_url=https://...
server downloads it

## 367. Anti-Pattern

X-Service-Name: admin-service
→ authenticated.

## 368. Anti-Pattern

localhost
=

trusted service

## 369. Anti-Pattern

internal network
=

authenticated

## 370. Anti-Pattern

Kubernetes namespace exists
=

authorized

## 1. Anti-Pattern

GitHub OIDC valid
=
production deployment allowed
sin mapping.

## 2. Anti-Pattern

worker
=
original human user

## 3. Anti-Pattern

background job contains user's full bearer token

## 4. Anti-Pattern

trace ID
=
identity

## 5. Anti-Pattern

credential valid
=
service active
sin comprobar security state cuando corresponda.

## 6. Anti-Pattern

machine MFA:
please enter SMS code

## 7. Anti-Pattern

static $currentService = $service;
en FrankenPHP.

## 8. Componentes principales

NonHumanPrincipal
MachineIdentity
ServiceAccount
WorkloadIdentity
MachineAuthenticationContext

MachineCredential
ApiKeyCredential
MachineCredentialProvider
MachineCredentialRotationPolicy

MachineAuthenticator
MachineIdentityProvider
MachineAuthenticatorResolver

MutualTlsAuthenticator
PrivateKeyJwtAuthenticator
ApiKeyAuthenticator
SignedRequestAuthenticator
WorkloadIdentityAuthenticator

WorkloadAttestation
ExternalWorkloadIdentityProvider
MachineIdentityMapper

SecurityTokenService
WorkloadCredentialExchange
MachineTokenExchange

MachineTrustStore
TrustDomain

AuthenticationActorChain
DelegatedOperationCredential

MachineAuthenticationRequirement
MachineAuthenticationAssuranceProfile

MachineNonceStore

## 379. Namespace sugerido

VoltStack\Quantum\Auth\Machine
VoltStack\Quantum\Auth\Machine\Contracts
VoltStack\Quantum\Auth\Machine\Identity
VoltStack\Quantum\Auth\Machine\Credential
VoltStack\Quantum\Auth\Machine\Authenticator
VoltStack\Quantum\Auth\Machine\Workload
VoltStack\Quantum\Auth\Machine\Federation
VoltStack\Quantum\Auth\Machine\Trust
VoltStack\Quantum\Auth\Machine\Token
VoltStack\Quantum\Auth\Machine\Delegation
VoltStack\Quantum\Auth\Machine\Attestation
VoltStack\Quantum\Auth\Machine\Runtime
VoltStack\Quantum\Auth\Machine\Events

## 380. Estructura sugerida

src/Quantum/Auth/Machine/
├── Contracts/
│   ├── NonHumanPrincipalInterface.php
│   ├── MachineAuthenticatorInterface.php
│   ├── MachineIdentityProviderInterface.php
│   ├── MachineCredentialProviderInterface.php
│   ├── WorkloadAttestationInterface.php
│   ├── ExternalWorkloadIdentityProviderInterface.php
│   ├── SecurityTokenServiceInterface.php
│   ├── MachineTrustStoreInterface.php
│   ├── MachineNonceStoreInterface.php
│   └── MachineIdentityAgreementPolicyInterface.php
│
├── Identity/
│   ├── MachineIdentity.php
│   ├── ServiceAccount.php
│   ├── WorkloadIdentity.php
│   ├── PrincipalType.php
│   └── ServiceAccountStatus.php
│
├── Credential/
│   ├── MachineCredential.php
│   ├── MachineCredentialType.php
│   ├── MachineCredentialStatus.php
│   ├── ApiKeyCredential.php
│   ├── MachineCredentialRequest.php
│   ├── MachineCredentialResult.php
│   └── MachineCredentialRotationPolicy.php
│
├── Authenticator/
│   ├── MachineAuthenticatorResolver.php
│   ├── ApiKeyAuthenticator.php
│   ├── ClientSecretAuthenticator.php
│   ├── PrivateKeyJwtAuthenticator.php
│   ├── MutualTlsAuthenticator.php
│   ├── SignedRequestAuthenticator.php
│   └── WorkloadIdentityAuthenticator.php
│
├── Workload/
│   ├── WorkloadEvidence.php
│   ├── WorkloadAttestationResult.php
│   ├── MachineTenantContext.php
│   └── TrustDomain.php
│
├── Federation/
│   ├── ExternalWorkloadIdentity.php
│   ├── WorkloadCredentialExchangeRequest.php
│   ├── WorkloadCredentialExchangeResult.php
│   ├── MachineIdentityMapper.php
│   └── SecurityTokenService.php
│
├── Token/
│   ├── MachineToken.php
│   ├── MachineTokenIssuer.php
│   ├── MachineTokenValidator.php
│   └── MachineTokenExchange.php
│
├── Trust/
│   ├── MachineTrustStore.php
│   ├── MachineTrustQuery.php
│   └── MachineTrustMaterial.php
│
├── Delegation/
│   ├── AuthenticationActorChain.php
│   ├── DelegatedOperationCredential.php
│   └── MachineDelegationPolicy.php
│
├── Attestation/
│   ├── WorkloadAttestation.php
│   └── WorkloadAttestationPolicy.php
│
├── RequestSigning/
│   ├── SignedRequestEvidence.php
│   ├── CanonicalRequest.php
│   ├── RequestSignatureVerifier.php
│   └── MachineNonceStore.php
│
├── Assurance/
│   ├── MachineAuthenticationRequirement.php
│   └── MachineAuthenticationAssuranceProfile.php
│
├── Runtime/
│   ├── MachineAuthenticationContext.php
│   ├── MachineRuntimeContext.php
│   └── MachineRuntimeResetter.php
│
└── Events/
├── MachineAuthenticationStarted.php
├── MachineAuthenticationSucceeded.php
├── MachineAuthenticationFailed.php
├── MachineCredentialIssued.php
├── MachineCredentialRotated.php
└── MachineCredentialRevoked.php

## 381. Flujo Service-to-Service

┌────────────────────┐
│     SERVICE A      │
│   orders-service   │
└─────────┬──────────┘
│
▼
┌────────────────────┐
│ Credential Provider│
└─────────┬──────────┘
│
▼
Short-Lived Credential
│
▼
┌────────────────────┐
│     SERVICE B      │
│  payments-service  │
└─────────┬──────────┘
│
▼
Machine Authenticator
│
▼
Credential Validation
│
▼
Identity Resolution
│
▼
Machine Security State
│
▼
Authentication Context
│
▼
Authorization
│
▼
Execute

## 382. Flujo Workload Federation

┌───────────────────────┐
│ Kubernetes / Cloud /  │
│ CI Workload Platform  │
└──────────┬────────────┘
│
▼
Workload Evidence
│
▼
┌───────────────────────┐
│ Evidence Verifier     │
└──────────┬────────────┘
│
▼
Identity Mapping
│
▼
Trust Policy
│
▼
┌───────────────────────┐
│ VoltStack STS         │
└──────────┬────────────┘
│
▼
Short-Lived Credential
│
▼
VoltStack Service

## 383. Flujo Background Worker

USER
│
│ requests operation
▼
APPLICATION
│
│ creates Job
▼
QUEUE
│
▼
WORKER
│
├── authenticates as:

```text
 │   service.export-worker
 │
```

├── carries initiator:

```text
 │   user-123
 │
 ▼
AUTHORIZATION
 │
 ▼
RESOURCE
```

Por tanto:

```php
Actor      = Worker
Initiator  = User
```

No:
Actor = User

## 384. Flujo de rotación

Credential V1
│
▼
Generate V2
│
▼
┌───────────────┐
│ V1 + V2 valid │
└───────┬───────┘
│
▼
Deploy V2
│
▼
Observe Usage
│
▼
Revoke V1
│
▼
Credential V2

## 385. Flujo Machine Step-Up

Machine Authentication
│
▼
Normal Assurance
│
▼
Sensitive Operation
│
▼
Machine Requirement
│
▼
Current Evidence Insufficient
│
▼
Re-Attestation /
Stronger Credential
│
▼
High-Assurance Machine Context
│
▼
Authorization
│
▼
Execute

## 386. Relación con Laravel

Laravel proporciona herramientas excelentes para autenticación de usuarios y APIs mediante conceptos como:

- Guards
- Providers
- Sanctum
- Passport
- API Tokens
- Middleware

Sin embargo, una arquitectura general de framework deberá distinguir claramente:
User Authentication
de:

- Workload Identity
- Service Identity
- Machine Credential Lifecycle
- Machine Attestation
- M2M Trust

VoltStack podrá conservar una experiencia sencilla similar a Laravel:
MachineAuth::service('payments');
o:

```php
MachineAuth::credential()->for('payments-api');
mientras internamente utiliza el modelo más completo descrito aquí.
```

## 387. Relación con Symfony

Symfony proporciona una arquitectura extensible de:

- Authenticators
- Passports
- Credentials
- User Providers
- Security

que sirve como referencia importante para la separación contractual.
VoltStack deberá extender esa filosofía hacia principals no humanos sin obligarlos a implementar semánticas de UserInterface.
El concepto fundamental será:
Principal

y no exclusivamente:
User

## 1. Diferenciador VoltStack

VoltStack combinará:

```text
Laravel-like DX

+

Symfony-like Authentication Contracts
+
First-Class Non-Human Principals
+
Service Accounts
+
Workload Identities
+
Short-Lived Credentials
+
Workload Federation
+
mTLS
+
Request Signing
+
Machine Credential Rotation
+
Machine Security Epochs
+
Service-to-Service Authentication
+
Delegated Operation Credentials
+
Machine Assurance
+
Multi-Tenant Machine Isolation
+
Distributed Revocation
+
FrankenPHP-safe Runtime
```

## 389. Developer Experience

Registro:

```php
MachineAuth::service('orders-api')
    ->realm('services')
    ->environment('production')
    ->credential(
        WorkloadIdentity::class
    );
```

Servicio:

```php
$credential = MachineAuth::credentials()
    ->for('orders-api')
    ->audience('payments-api')
    ->acquire();
```

Servidor:

```php
# [RequireMachineIdentity('orders-api')]
public function process(): Response
{
}
```

Operación crítica:

```php
# [MachineAuthentication(
    shortLived: true,
    workloadBound: true,
    attested: true,
)]
public function rotateKey(): Response
{
}
```

La API exacta se definirá posteriormente.

## 390. Decisiones arquitectónicas

VoltStack adoptará:

## 1. Principal es la abstracción superior; User es un tipo de Principal.

## 2. Human y Non-Human identities permanecen explícitamente diferenciadas.

## 3. Machine Identity y Machine Credential son objetos independientes.

## 4. Service Identity y Workload Identity son conceptos distintos.

## 5. Los workloads efímeros no requieren secrets permanentes.

## 6. Short-lived credentials son preferidos.

## 7. Static API keys existen principalmente por compatibilidad.

## 8. Production podrá prohibir static credentials.

## 9. mTLS será un Authenticator de primera clase.

## 10. Workload Identity Federation será una capacidad arquitectónica central.

## 11. External identities requieren mapping explícito.

## 12. Trust domains son explícitos.

## 13. Cross-domain trust nunca es automático.

## 14. Audience binding será first-class.

## 15. Tenant, realm y environment binding serán first-class.

## 16. Credential rotation será zero-downtime cuando sea posible.

## 17. Credential revocation podrá ser identity-wide o credential-specific.

## 18. Machine Security Epoch permitirá invalidación masiva.

## 19. Service-to-service Authentication y Authorization permanecerán separados.

## 20. Queue workers tendrán identidad propia.

## 21. Human initiator y Machine actor nunca se confundirán.

## 22. Delegation nunca amplificará privilegios.

## 23. Signed requests deberán soportar replay protection.

## 24. Headers arbitrarios nunca serán Authentication Evidence.

## 25. Service mesh identity solo se confiará mediante una frontera explícita.

## 26. Machine assurance no se modelará como MFA humano.

## 27. Machine Step-Up utilizará stronger machine evidence.

## 28. Trust material podrá cachearse; secret material no.

## 29. Unknown issuer/key/audience/tenant fallará cerrado.

30. Machine contexts serán seguros bajo workers persistentes de FrankenPHP.
31. Criterios de aceptación

El subsistema será considerado completo cuando soporte:
32. PrincipalType;
33. Non-Human Principals;
34. Machine Identities;
35. Service Accounts;
36. Workload Identities;
37. ephemeral workload instances;
38. Machine Authentication Context;
39. Machine Identity Providers;
40. Machine Credentials;
41. credential lifecycle;
42. credential inventory;
43. credential ownership;
44. API Keys;
45. Client Secrets;
46. Private Key JWT;
47. mTLS;
48. certificate identity mapping;
49. certificate rotation;
50. request signing;
51. replay protection;
52. nonce stores;
53. asymmetric signing;
54. short-lived machine tokens;
55. audience binding;
56. issuer binding;
57. tenant binding;
58. realm binding;
59. environment binding;
60. Machine Security Epoch;
61. credential-specific revocation;
62. Workload Identity Federation;
63. Security Token Service;
64. credential exchange;
65. workload attestation;
66. Trust Domains;
67. external workload providers;
68. Kubernetes identity integration;
69. cloud identity adapters;
70. CI/CD identity federation;
71. queue worker identities;
72. scheduled-job identities;
73. human initiator preservation;
74. actor chains;
75. delegated operation credentials;
76. token exchange;
77. privilege downscoping;
78. service mesh integration;
79. Machine Authentication Assurance;
80. Machine Authentication Requirements;
81. machine step-up;
82. machine reauthentication;
83. risk integration;
84. throttling integration;
85. audit integration;
86. metrics;
87. tracing;
88. distributed revocation;
89. distributed replay protection;
90. trust rotation;
91. FrankenPHP request/fiber isolation.
92. Regla arquitectónica final

VoltStack deberá entender una identidad de software de la siguiente forma:

```text
                   NON-HUMAN PRINCIPAL
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           Service      Workload      Device
              │            │            │
              └────────────┼────────────┘
                           ▼
                    MACHINE IDENTITY
                           │
                           ▼
                    AUTHENTICATION
                           │
              ┌────────────┼─────────────┐
              ▼            ▼             ▼
             mTLS       Federation     Signing
              │            │             │
              └────────────┼─────────────┘
                           ▼
               AUTHENTICATION CONTEXT
                           │
                           ▼
                     AUTHORIZATION
                           │
                           ▼
                        RESOURCE
```

La primera regla será:
Una máquina no será modelada como un usuario humano ficticio.

La segunda:
Una identidad deberá sobrevivir a la rotación de sus credenciales; por ello, identidad y credential serán dominios independientes.

La tercera:
VoltStack preferirá identidades verificables y credenciales de corta duración sobre secretos estáticos distribuidos entre servidores.

La cuarta:
Un workload deberá demostrar quién es mediante evidencia vinculada a su entorno cuando la infraestructura permita hacerlo, en lugar de depender permanentemente de secretos almacenados.

La quinta:
Todo servicio tendrá identidad propia; la red interna, localhost, un nombre de header o el hecho de ejecutar dentro del mismo cluster jamás constituirán Authentication por sí mismos.

La sexta:
Cuando una operación iniciada por un humano continúe en un worker, el worker seguirá siendo el actor autenticado y el usuario permanecerá como iniciador; VoltStack nunca confundirá delegación con impersonación.

La séptima:
Machine-to-Machine Authentication establecerá quién es el servicio; el Authorization System determinará qué puede hacer ese servicio.

La octava:
Toda credencial de máquina deberá poder identificarse, inventariarse, expirar, rotarse, revocarse y auditarse.

La novena:
Los workers persistentes de FrankenPHP, procesos de colas y ejecuciones concurrentes jamás compartirán accidentalmente Machine Authentication Context entre solicitudes o trabajos.

La décima:
La arquitectura deberá permitir evolucionar desde API keys sencillas hasta mTLS, workload federation, hardware-backed identities y zero-trust service authentication sin sustituir el dominio central de Authentication.

Siguiente documento recomendado
La siguiente pieza natural del sistema es:
`34_AUTHENTICATION_IDENTITY_LINKING_ACCOUNT_LINKING_CREDENTIAL_BINDING_AND_AUTHENTICATION_METHOD_MANAGEMENT_SYSTEM.md`
Este documento cubriría un problema diferente que todavía necesitamos formalizar: cómo una misma identidad administra y relaciona sus distintos métodos de autenticación.
Identity
│
├── Password
├── Passkey #1
├── Passkey #2
├── TOTP
├── Recovery Codes
├── Google OIDC
├── Microsoft OIDC
└── Enterprise SSO
Ahí definiríamos de forma centralizada link/unlink de proveedores, binding de nuevas credenciales, prevención de account takeover durante linking, eliminación del último método de acceso, credential replacement, identidad duplicada, merge seguro de cuentas y lifecycle de métodos de autenticación.

Instantánea
