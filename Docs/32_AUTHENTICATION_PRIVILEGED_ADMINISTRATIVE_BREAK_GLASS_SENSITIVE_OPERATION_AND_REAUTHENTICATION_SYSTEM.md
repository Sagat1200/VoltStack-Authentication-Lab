# VoltStack Authentication System

## 32 — Authentication Privileged, Administrative, Break-Glass, Sensitive Operation and Reauthentication System

- **Archivo:** `32_AUTHENTICATION_PRIVILEGED_ADMINISTRATIVE_BREAK_GLASS_SENSITIVE_OPERATION_AND_REAUTHENTICATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Dependencias principales:** documentos 03, 04, 05, 06, 08, 10, 12, 15, 16, 20, 21, 24, 25, 29, 30 y 31.
- **Estado:** Especificación arquitectónica del subsistema encargado de reautenticación, autenticación fresca, operaciones sensibles, sesiones privilegiadas, autenticación administrativa y acceso de emergencia o break-glass.

---

## 1. Propósito

Este documento define cómo VoltStack deberá proteger operaciones cuya sensibilidad sea superior al nivel de confianza proporcionado por una autenticación normal.
El sistema deberá distinguir explícitamente entre:

```text
Authenticated
        │
        ├── Normal Authentication
        │
        ├── Fresh Authentication
        │
        ├── Step-Up Authentication
        │
        ├── Privileged Authentication
        │
        ├── Administrative Authentication
        │
        └── Emergency / Break-Glass Authentication
```

La autenticación deja de ser una condición binaria:
authenticated = true / false
y pasa a representar un contexto de seguridad con:

- Identity
- Authentication Time
- Authentication Method
- Factors
- Assurance Level
- Freshness
- Risk
- Device Trust
- Session
- Tenant
- Realm
- Privilege Context

## 2. Problema fundamental

Un usuario puede estar correctamente autenticado y, aun así, no estar suficientemente autenticado para ejecutar una operación determinada.
Ejemplo:

```text
User authenticated 7 hours ago
        ↓
Valid Session
        ↓
```

Attempts:

```text
"Disable MFA"
        ↓
Session valid?
YES

Authentication sufficient?
NO
```

Por tanto:
Una sesión válida no implica automáticamente autorización criptográfica o autenticación suficiente para ejecutar operaciones sensibles.

## 1. Authentication vs Authorization

Este documento pertenece al sistema de Authentication.
Authorization continúa siendo responsable de responder:
¿Puede esta identidad ejecutar esta operación?
Authentication responde:

```text
¿Quién es?

¿Cómo se autenticó?

¿Cuándo se autenticó?

¿Qué factores presentó?

¿Qué nivel de assurance alcanzó?

¿La autenticación sigue siendo suficientemente fresca?

¿Debe demostrar nuevamente su identidad?
```

## 4. Regla de separación

Ejemplo:
DELETE TENANT
Authorization:

- Does user have:
- tenant.delete
- ?

Authentication:

- Is authentication:
- fresh enough?
- strong enough?
- phishing-resistant if required?

performed from acceptable device?
?
Ambos deberán aprobar.

## 5. Modelo conceptual

REQUEST
│
▼
AUTHENTICATION CONTEXT
│
▼
AUTHORIZATION
│
▼
SENSITIVE OPERATION CLASSIFIER
│
▼
AUTHENTICATION REQUIREMENT
│
├── Existing context sufficient ─────► EXECUTE
│
└── Insufficient
│
▼
REAUTHENTICATION
│
▼
STEP-UP
│
▼
NEW AUTH EVIDENCE
│
▼
RE-EVALUATE
│
▼
EXECUTE

## 6. Principios arquitectónicos

VoltStack adoptará:

## 1. Authentication strength is contextual

## 2. Sensitive operations define explicit authentication requirements

## 3. Session validity and authentication freshness are separate concepts

## 4. Reauthentication does not necessarily create a new session

## 5. Step-up can increase authentication assurance temporarily

## 6. Privileged authentication must be explicitly represented

## 7. Administrative identities require stronger policies

## 8. Break-glass access is exceptional, explicit and auditable

## 9. Emergency access never silently bypasses Authentication

## 10. Sensitive-operation proof must be bounded to scope and time

## 11. Authentication Assurance

VoltStack deberá representar el nivel de confianza de una autenticación.
enum AuthenticationAssuranceLevel: int
{
case Low = 1;
case Standard = 2;
case Strong = 3;
case High = 4;
case Privileged = 5;
}
Los nombres concretos podrán evolucionar.
La semántica será más importante que el número.

## 12. Assurance no es Role

Nunca:

- ADMIN role
- =
- HIGH authentication assurance
- Son conceptos independientes.

Un administrador puede haber iniciado sesión únicamente con contraseña.

## 13. Authentication Methods

Ejemplos:

- PASSWORD
- TOTP
- SMS_OTP
- EMAIL_OTP
- PASSKEY
- WEBAUTHN
- CLIENT_CERTIFICATE
- RECOVERY_CODE
- FEDERATED
- DEVICE_CREDENTIAL

Cada mecanismo puede contribuir de manera distinta al assurance.

## 14. Authentication Method Reference

final readonly class AuthenticationMethodReference
{
public function __construct(
public string $method,
public DateTimeImmutable $verifiedAt,
) {}
}

## 15. Authentication Evidence

El documento 08 proporciona el modelo general de evidencia.
Este subsistema deberá reutilizarlo.
No crear un segundo sistema paralelo de credenciales.

## 16. Authentication Freshness

Freshness responde:

- ¿Cuánto tiempo ha transcurrido desde que el usuario demostró
- su identidad mediante evidencia aceptable?

## 17. Fresh Authentication

Ejemplo:

```text
Session created:
08:00
```

Current time:
15:00

Sensitive operation requires:
authentication <= 10 minutes old

Result:
REAUTHENTICATION_REQUIRED

## 14. AuthenticationFreshness

final readonly class AuthenticationFreshness
{
public function __construct(
public DateTimeImmutable $authenticatedAt,
public DateTimeImmutable $evaluatedAt,
) {}

public function age(): DateInterval
{
return $this->authenticatedAt->diff($this->evaluatedAt);
}
}

## 15. No confiar únicamente en Session Creation Time

Una sesión puede durar:

- 12 hours
- 30 days
- remember-me

mientras una operación sensible puede exigir:
authentication within 5 minutes

## 16. Multiple Freshness Clocks

Podrán existir:

- PrimaryAuthenticationFreshness
- PasswordFreshness
- MfaFreshness
- PasskeyFreshness
- PrivilegedAuthenticationFreshness

## 17. Ejemplo

Usuario:

- Password verified:     08:00
- Passkey verified:      14:55
- Current time:          15:00

Una operación que requiere:

- fresh phishing-resistant authentication <= 10 minutes
- puede aceptarse.

## 18. AuthenticationRequirement

Objeto central:

```php
final readonly class AuthenticationRequirement
{
    public function __construct(
        public AuthenticationAssuranceLevel $minimumAssurance,
        public ?DateInterval $maximumAge,
        public bool $requireMfa,
        public bool $requirePhishingResistance,
        public bool $requireTrustedDevice,
        public bool $requireInteractiveAuthentication,
    ) {}
}
```

## 19. Requirements extensibles

No deberán limitarse a esos campos.

- Podrán incluir:
- RequiredMethods
- ForbiddenMethods
- RequiredFactorClasses
- MinimumIndependentFactors
- RequiredDeviceState
- RequiredRealm
- RequiredTenant
- RequiredSessionType
- RiskThreshold
- UserPresence
- UserVerification

## 20. Sensitive Operation

Una operación sensible deberá tener identidad estable.

```php
final readonly class SensitiveOperation
{
    public function__construct(
        public string $name,
    ) {}
}
```

Ejemplos:

```text
auth.password.change
auth.mfa.disable
auth.passkey.remove
auth.recovery.regenerate
auth.sessions.revoke_all

tenant.delete
tenant.security.change

admin.user.disable
admin.role.assign

api.credentials.create
api.credentials.revoke

billing.payment_method.change
```

## 21. Sensitive Operation Registry

interface SensitiveOperationRegistryInterface
{
public function resolve(
string $operation
): SensitiveOperationDefinition;
}

## 22. SensitiveOperationDefinition

final readonly class SensitiveOperationDefinition
{
public function __construct(
public SensitiveOperation $operation,
public AuthenticationRequirement $authentication,
) {}
}

## 23. Declarative Configuration

Ejemplo conceptual:

```php
'sensitive_operations' => [

    'auth.password.change' => [
        'max_age' => '10 minutes',
        'interactive' => true,
    ],

    'auth.mfa.disable' => [
        'max_age' => '5 minutes',
        'mfa' => true,
    ],

    'tenant.delete' => [
        'max_age' => '5 minutes',
        'assurance' => 'high',
        'phishing_resistant' => true,
    ],

];
```

## 24. Código declarativo

VoltStack podrá soportar:

```php
# [SensitiveOperation(
    name: 'tenant.delete',
    assurance: 'high',
    freshWithin: '5 minutes',
    phishingResistant: true,
)]
public function destroy(Tenant $tenant)
{
}
```

La metadata deberá integrarse con los sistemas de Controller, Route y Compilation de VoltStack.

## 25. Sensitive Operation Classifier

No toda operación deberá declararse manualmente.

```php
Podrán existir clasificadores.
interface SensitiveOperationClassifierInterface
{
    public function classify(
        AuthenticationOperationContext $context
    ): SensitiveOperationClassification;
}
```

## 26. Classification

Ejemplo:

- NORMAL
- ELEVATED
- SENSITIVE
- HIGHLY_SENSITIVE
- PRIVILEGED
- EMERGENCY

## 27. No depender del HTTP Method

No asumir:

```php
DELETE = sensitive
POST = sensitive
GET = safe
```

La semántica de negocio determina sensibilidad.

## 28. Reauthentication

Reauthentication significa:
Solicitar nuevamente evidencia de identidad a una identidad ya autenticada.

## 1. Reauthentication no es Login

No deberá destruir automáticamente la sesión actual.

```text
Flujo:
Authenticated Session
        ↓
Sensitive Operation
        ↓
Reauthentication Challenge
        ↓
Evidence
        ↓
Success
        ↓
Session remains
        +
Fresh Authentication State
```

## 2. ReauthenticationManager

interface ReauthenticationManagerInterface
{
public function challenge(
AuthenticationContext $context,
AuthenticationRequirement $requirement
): ReauthenticationChallenge;

public function verify(
ReauthenticationAttempt $attempt
): ReauthenticationResult;
}

## 3. Reauthentication Challenge

Podrá especificar:

- acceptable methods
- required factors
- challenge expiry
- operation binding
- session binding
- tenant binding

## 4. Challenge ID

final readonly class ReauthenticationChallengeId
{
public function __construct(
public string $value,
) {}
}
Debe ser impredecible.

## 5. Challenge Lifecycle

CREATED
↓
PRESENTED
↓
VERIFIED
↓
CONSUMED
Alternativas:

- EXPIRED
- FAILED
- CANCELLED
- REVOKED

## 6. Single-use

Challenges sensibles deberán ser single-use cuando corresponda.

## 7. Challenge Expiration

Ejemplo:

- 5 minutes
- configurable.

## 8. Challenge Binding

Challenge deberá estar ligado, según profile, a:

- UserId
- SessionId
- TenantId
- RealmId
- Operation
- Request Intent

## 9. Anti-Replay

Un challenge utilizado para:
auth.password.change
no deberá reutilizarse para:
tenant.delete

## 10. Reauthentication Proof

Una reautenticación exitosa podrá generar:
ReauthenticationProof

## 11. ReauthenticationProof

final readonly class ReauthenticationProof
{
public function __construct(
public string $id,
public DateTimeImmutable $authenticatedAt,
public AuthenticationAssuranceLevel $assurance,
public array $methods,
public array $scopes,
public DateTimeImmutable $expiresAt,
) {}
}

## 12. Proof no necesariamente va al cliente

Preferiblemente podrá permanecer server-side y asociarse a:

- Session
- Authentication Context
- Sensitive Operation Context

## 13. Operation-Bound Proof

Para operaciones muy sensibles:

- Proof:
- tenant.delete:tenant-123

no:

- Proof:
- all-sensitive-actions

## 14. Broad Proof

Podrá permitirse para UX cuando policy lo considere aceptable.
Ejemplo:

- security-settings
- por cinco minutos.

## 15. Scope

ReauthenticationScope
podrá representar:

- operation
- resource
- security domain
- tenant
- realm

## 16. ReauthenticationProofStore

interface ReauthenticationProofStoreInterface
{
public function store(ReauthenticationProof $proof): void;

public function find(
AuthenticationContext $context,
SensitiveOperation $operation
): ?ReauthenticationProof;

public function consume(string $id): void;
}

## 17. Proof Lifetime

Debe ser corto.
No convertir proof en otra sesión de larga duración.

## 18. Step-Up Authentication

Step-Up aumenta temporalmente el assurance.

```text
Ejemplo:
Password Login
    ↓
AAL Standard
    ↓
Sensitive Action
    ↓
Passkey
    ↓
AAL High
```

## 19. Step-Up vs Reauthentication

Reauthentication:
demuestra nuevamente identidad
Step-Up:

- incrementa fuerza/assurance
- Pueden ocurrir simultáneamente.

## 20. Ejemplo

Sesión iniciada con:

- Password + TOTP
- hace 8 horas.

Operación requiere:

- MFA fresh <= 5 min
- Esto es principalmente reauthentication.

## 21. Otro ejemplo

Sesión iniciada con:
Password
Operación requiere:

- phishing-resistant MFA
- Esto requiere Step-Up.

## 22. StepUpManager

interface AuthenticationStepUpManagerInterface
{
public function determine(
AuthenticationContext $context,
AuthenticationRequirement $requirement
): StepUpPlan;

public function complete(
StepUpAttempt $attempt
): StepUpResult;
}

## 23. StepUpPlan

Puede indicar:

- Current Assurance
- Required Assurance
- Missing Factors
- Accepted Methods
- Preferred Method
- Fallback Methods

## 24. Strongest Available Method

Para operaciones críticas, el sistema puede preferir:
Passkey/WebAuthn
sobre:

- SMS OTP
- si ambos están disponibles.

## 25. Phishing Resistance

Deberá representarse explícitamente.
No inferirse únicamente de:
factor count >= 2

## 26. Dos factores no equivalen necesariamente a phishing-resistant

Ejemplo:

- Password + SMS OTP
- es MFA, pero no proporciona las mismas propiedades que WebAuthn/passkeys.

## 27. Authentication Factor Classes

Podrán existir:

- KNOWLEDGE
- POSSESSION
- INHERENCE
- CRYPTOGRAPHIC
- DEVICE
- RECOVERY
- FEDERATED

## 28. Factor Independence

Policy podrá exigir factores independientes.

## 29. Privileged Authentication

Una autenticación privilegiada es un contexto temporal en el que se han satisfecho requisitos especiales para ejecutar operaciones privilegiadas.

## 30. PrivilegedAuthenticationContext

final readonly class PrivilegedAuthenticationContext
{
public function __construct(
public string $id,
public DateTimeImmutable $establishedAt,
public DateTimeImmutable $expiresAt,
public AuthenticationAssuranceLevel $assurance,
public array $methods,
public array $scopes,
) {}
}

## 31. No convertir sesión normal permanentemente en privilegiada

Evitar:

```text
login once as admin
        ↓
privileged forever
```

Preferir:

```text
normal session
        ↓
temporary privileged context
```

## 32. Privileged Session

Para ciertos entornos puede existir sesión administrativa separada.
Normal Session
│
▼
Privilege Elevation
│
▼
Privileged Session

## 33. Privileged Session Lifetime

Debe ser considerablemente menor que una sesión normal.
Ejemplo conceptual:

- Normal session:      12 hours
- Privileged session:  15 minutes

No son valores universales.

## 34. Idle Timeout

Privileged sessions deberán soportar idle timeout corto.

## 35. Absolute Timeout

También:

- absolute maximum lifetime
- aunque exista actividad.

## 36. Privilege Elevation

Normal Authentication
↓
Authorization permits elevation
↓
Strong Reauthentication
↓
Risk Evaluation
↓
Device Evaluation
↓
Privileged Context

## 37. Authorization sigue siendo obligatoria

Un usuario normal no puede obtener privilegios administrativos simplemente completando MFA.

## 38. Important

Strong Authentication
≠
Administrative Authorization

## 39. Administrative Authentication

Administradores deberán poder utilizar policies específicas.

- Ejemplo:
- require MFA
- require phishing-resistant factor
- forbid remember-me
- require trusted device
- short session
- short inactivity timeout

fresh auth for destructive actions

## 40. AdministrativeAuthenticationPolicy

interface AdministrativeAuthenticationPolicyInterface
{
public function requirements(
AdministrativeAuthenticationContext $context
): AuthenticationRequirement;
}

## 41. Administrative Realm

VoltStack podrá separar:

- USER_REALM
- ADMIN_REALM
- PLATFORM_ADMIN_REALM
- SECURITY_ADMIN_REALM

## 42. Realm Separation

Idealmente:
/admin
puede utilizar contexto distinto de:
/app

## 43. Separate Authentication Context

En deployments de alta seguridad:
Normal AuthenticationContext
y:

- AdministrativeAuthenticationContext
- deberán estar separados.

## 44. Cookie Separation

Podrá existir:

- normal session cookie
- admin session cookie

con:

- different name
- different path
- different lifetime
- different SameSite policy
- different key purpose

cuando architecture lo requiera.

## 45. Key Separation

Documento 31 deberá permitir:

- SESSION_SIGNING
- ADMIN_SESSION_SIGNING
- como purposes distintos.

## 46. Remember-Me

Por default, privileged/admin sessions no deberán depender exclusivamente de remember-me.

## 47. Remember-Me Restoration

Puede restaurar:
normal authenticated context
pero no automáticamente:
privileged context

## 48. Admin MFA

Administradores deberán tener policies más fuertes que usuarios normales cuando deployment lo requiera.

## 49. Admin Passkeys

VoltStack deberá permitir policy:

```php
admin:
    phishing_resistant_required = true
```

## 50. Recovery Codes

Un recovery code puede recuperar acceso.
No necesariamente deberá otorgar inmediatamente un privileged context.

## 51. Recovery Downgrade

Después de recovery:

- Restricted Authentication State
- puede exigir enrollment de nuevo factor antes de operaciones privilegiadas.

## 52. Sensitive Security Operations

El sistema deberá incluir por default una taxonomía de operaciones de seguridad.

## 53. Password Operations

auth.password.change
auth.password.reset
auth.password.remove

## 54. MFA Operations

auth.mfa.enable
auth.mfa.disable
auth.mfa.factor.add
auth.mfa.factor.remove
auth.mfa.factor.replace

## 55. Passkey Operations

auth.passkey.register
auth.passkey.rename
auth.passkey.remove
auth.passkey.remove_all

## 56. Recovery Operations

auth.recovery.codes.generate
auth.recovery.codes.regenerate
auth.recovery.contact.change

## 57. Session Operations

auth.sessions.list
auth.sessions.revoke
auth.sessions.revoke_all
No todas requieren el mismo nivel.

## 58. Device Operations

auth.device.trust
auth.device.untrust
auth.device.revoke
auth.device.revoke_all

## 59. Federation Operations

auth.federation.link
auth.federation.unlink
auth.federation.provider.change

## 60. API Credential Operations

auth.api_token.create
auth.api_token.rotate
auth.api_token.revoke
auth.api_token.revoke_all

## 61. Tenant Security Operations

tenant.security.policy.change
tenant.authentication.policy.change
tenant.identity_provider.change
tenant.trust_anchor.change
tenant.key.rotate
tenant.delete

## 62. Platform Operations

platform.authentication.policy.change
platform.key.rotate
platform.trust.change
platform.tenant.disable
platform.tenant.delete

## 63. SensitiveOperationPolicyResolver

interface SensitiveOperationPolicyResolverInterface
{
public function resolve(
SensitiveOperation $operation,
AuthenticationContext $context
): AuthenticationRequirement;
}

## 64. Hierarchical Policy Resolution

Policy puede venir de:

```text
Platform
   ↓
Realm
   ↓
Tenant
   ↓
Application
   ↓
Operation
```

## 65. Platform Security Floor

Tenant puede endurecer.
No deberá reducir mínimos globales.

## 66. Ejemplo

Platform:

```text
tenant.delete:
    phishing-resistant = true
```

Tenant:

```text
tenant.delete:
    phishing-resistant = false
```

Resultado:
true

## 67. Policy Merge

Deberá utilizar:

- most restrictive compatible requirement
- según reglas deterministas.

## 68. Risk-Based Reauthentication

Documento 20 deberá integrarse.

- Una operación normalmente aceptable puede requerir reauthentication si:
- new device
- new country
- impossible travel
- IP reputation
- session anomaly
- credential stuffing suspicion
- privilege escalation attempt

## 69. Dynamic Requirements

Static Requirement
+
Risk Requirement
+
Tenant Requirement
+
Realm Requirement
↓
Effective Authentication Requirement

## 70. AuthenticationRequirementResolver

interface AuthenticationRequirementResolverInterface
{
public function resolve(
AuthenticationContext $context,
SensitiveOperation $operation
): EffectiveAuthenticationRequirement;
}

## 71. Effective Requirement

Debe ser explainable.

## 72. Requirement Explainability

Documento 24 deberá poder responder:
Why was reauthentication required?
Ejemplo:

```text
Operation:
tenant.delete
```

Reasons:

```text
- HIGH assurance required
- authentication older than 5 minutes
- phishing-resistant method required
```

## 1. No revelar información peligrosa

Explainability deberá ser adecuada al actor.

## 2. Device Trust Integration

Documento 21.
Una operación podrá exigir:

- trusted device
- pero device trust nunca deberá sustituir automáticamente identidad.

## 3. Device Requirement

Ejemplo:
platform.key.rotate
puede exigir:

```text
trusted managed device

+

passkey
+
fresh authentication
```

## 104. Untrusted Device

Policy puede:
deny
o:
require stronger step-up

## 105. Risk + Device + Assurance

Modelo:

```text
Operation Sensitivity
       +
Current Assurance
       +
Authentication Freshness
       +
Risk
       +
Device Trust
       ↓
Authentication Decision
```

## 106. Authentication Decision

SUFFICIENT
REAUTHENTICATION_REQUIRED
STEP_UP_REQUIRED
PRIVILEGED_CONTEXT_REQUIRED
DENIED
BREAK_GLASS_REQUIRED

## 107. AuthenticationRequirementEvaluator

interface AuthenticationRequirementEvaluatorInterface
{
public function evaluate(
AuthenticationContext $context,
EffectiveAuthenticationRequirement $requirement
): AuthenticationRequirementDecision;
}

## 108. Deterministic Evaluation

Misma entrada y misma policy version deberán producir misma decisión, salvo señales explícitamente temporales.

## 109. Requirement Version

Puede existir:

- AuthenticationRequirementVersion
- para auditoría.

## 110. TOCTOU

Debe evitarse:

```text
authenticate
   ↓
wait long time
   ↓
execute critical operation
```

## 111. Final Security Check

Inmediatamente antes de commit de una operación crítica podrá revalidarse:

- proof validity
- session validity
- privileged context
- security epoch
- authorization

## 112. Sensitive Operation Execution Guard

interface SensitiveOperationGuardInterface
{
public function assert(
AuthenticationContext $context,
SensitiveOperation $operation
): void;
}

## 113. Guard Pipeline

Sensitive Operation
↓
Authorization Check
↓
Authentication Requirement
↓
Proof Validation
↓
Risk Check
↓
Security State Check
↓
Execute

## 114. Reauthentication Race

Si usuario completa reauthentication y después:

- account disabled
- session revoked
- role removed
- tenant suspended

la operación deberá revalidar estado relevante.

## 115. Proof no congela Authorization

Muy importante:
Una prueba de reautenticación demuestra Authentication; no congela permisos ni Authorization.

## 1. Proof no congela Account Status

También debe comprobarse el estado actual de identidad.

## 2. Proof Revocation

Podrá invalidarse por:

- logout
- session revocation
- password change
- MFA change
- security epoch change
- risk escalation
- account suspension

## 3. Security Epoch

Integración con documentos anteriores.

```text
Ejemplo:
proof.securityEpoch = 19
current.securityEpoch = 20
```

Result:
INVALID

## 119. Proof Session Binding

Por default:
proof Session A
no será válido en:
Session B

## 120. Proof Tenant Binding

Igual para tenants.

## 121. Cross-Tenant Elevation

Nunca permitir que reauthentication realizada en:
Tenant A
eleve automáticamente contexto de:
Tenant B

## 122. Multi-Tenant Admin

Platform admins deberán tener scope explícito.

## 123. Platform Administrator

No confundir:
tenant admin
con:
platform admin

## 124. Privileged Scope

TENANT_ADMIN
PLATFORM_ADMIN
SECURITY_ADMIN
BILLING_ADMIN
SUPPORT_OPERATOR
podrán tener políticas de Authentication diferentes.

## 125. Impersonation

El acceso de impersonación pertenece principalmente a Authorization/Delegation, pero Authentication deberá proteger su entrada.

## 126. Impersonation Entry

Puede requerir:

- fresh admin authentication
- MFA
- reason
- ticket/reference
- audit

## 127. Impersonation Context

No deberá confundirse con autenticación del usuario impersonado.
Actor:
Admin A

Effective Subject:
User B

## 128. Authentication Actor Preservation

Audit siempre deberá conservar actor original.

## 129. Sensitive Action during Impersonation

Por default, ciertas operaciones deberán bloquearse:

- change user's password
- disable user's MFA
- create permanent credentials
- change tenant ownership
- salvo policy explícita.

## 130. Break-Glass

Break-glass representa acceso extraordinario utilizado cuando los mecanismos normales no son suficientes o no están disponibles.
Ejemplos:

- Identity Provider outage
- MFA infrastructure outage
- Tenant SSO misconfiguration
- Security incident
- Administrative lockout
- Disaster recovery

## 131. Regla crítica

Break-glass no es un bypass oculto de Authentication. Es un mecanismo de Authentication excepcional con identidad, credenciales, políticas y auditoría propias.

## 1. BreakGlassIdentity

Debe ser explícita.
No utilizar:
if ($username === 'root') bypass();

## 2. BreakGlassAccount

Podrá existir:

```php
final readonly class BreakGlassAccount
{
    public function __construct(
        public string $id,
        public string $realm,
        public BreakGlassPolicy $policy,
    ) {}
}
```

## 3. Break-Glass Credentials

Deberán ser distintas de credenciales normales.

## 4. Preferred Break-Glass Methods

Según deployment:

- hardware security key
- offline passkey
- client certificate
- HSM-backed credential
- high-entropy emergency secret
- multi-party recovery

## 5. Password-only Break-Glass

No deberá ser default para high-security profiles.

## 6. Offline Capability

Break-glass puede necesitar funcionar durante:

- external IdP outage
- network isolation
- federation failure

por lo que su dependencia externa deberá diseñarse cuidadosamente.

## 7. Local Emergency Identity

Puede existir aun cuando Authentication normal sea federada.

## 8. BreakGlassPolicy

final readonly class BreakGlassPolicy
{
public function __construct(
public bool $enabled,
public DateInterval $maximumSessionLifetime,
public bool $requireReason,
public bool $requireIncidentReference,
public bool $requireDualControl,
public bool $notifySecurityTeam,
) {}
}

## 9. Disabled by Default

Deployments que no necesiten break-glass podrán mantenerlo deshabilitado.

## 10. Break-Glass Activation

Puede requerir activation explícita.

```text
Dormant
   ↓
Emergency Activation
   ↓
Usable
   ↓
Automatic Expiration
```

## 11. Dormant Credential

Un break-glass account no necesita estar permanentemente habilitado.

## 12. Activation Authority

Podrá requerir:

- Security Officer
- Platform Owner
- Incident Commander
- según organización.

## 13. Dual Control

Para sistemas críticos:

```text
Operator A
     +
Operator B
     ↓
Break-Glass Activation
```

## 14. Separation of Duties

Integración con Authorization document 24 de aprobación/dual control cuando corresponda.

## 15. BreakGlassActivationRequest

final readonly class BreakGlassActivationRequest
{
public function __construct(
public string $accountId,
public string $reason,
public ?string $incidentReference,
public DateTimeImmutable $requestedAt,
) {}
}

## 16. BreakGlassActivation

Debe tener:

- ActivationId
- AccountId
- ActivatedAt
- ExpiresAt
- Reason
- Approvers
- Scope

## 17. Break-Glass Session

Siempre deberá ser identificable:
session.type = BREAK_GLASS

## 18. No disguise

No deberá aparecer internamente como una sesión administrativa ordinaria.

## 19. Break-Glass Scope

Preferir:
minimum required scope
sobre:
unrestricted root

## 20. Emergency Scope

Ejemplo:

- restore.identity_provider
- unlock.security_admin
- rotate.compromised_key

## 21. Time-Bounded

Siempre temporal.

## 22. Short Lifetime

Debe utilizar un TTL corto.

## 23. No Remember-Me

Break-glass jamás deberá generar:

- remember-me
- persistent login
- long-lived browser credential

## 24. No Silent Renewal

No renovar automáticamente.

## 25. Explicit Expiration

Al terminar:

- session revoked
- activation expired
- proofs revoked

## 26. Post-Emergency Rotation

Si se utilizaron emergency secrets, policy puede exigir rotarlos después del incidente.

## 27. Break-Glass Audit

Debe ser exhaustivo.

## 28. Audit Events

BreakGlassRequested
BreakGlassApproved
BreakGlassActivated
BreakGlassAuthenticationSucceeded
BreakGlassAuthenticationFailed
BreakGlassSessionCreated
BreakGlassOperationExecuted
BreakGlassSessionExpired
BreakGlassSessionRevoked
BreakGlassCredentialRotated

## 29. Immediate Alerting

Activación de break-glass debería poder generar:

- Security Alert
- SOC Notification
- Administrator Notification
- SIEM Event

## 30. No Alert Dependency

El acceso de emergencia no deberá depender necesariamente de que el sistema de notificaciones esté operativo.
Audit debe persistirse localmente o por mecanismo resiliente.

## 31. Break-Glass Reason

Debe ser obligatorio en profiles empresariales.

## 32. Reason no es Authentication Evidence

Es audit metadata.

## 33. Incident Reference

Puede vincular:
INC-2026-000123

## 34. Break-Glass Restrictions

Aunque autenticado mediante break-glass, ciertas acciones pueden permanecer prohibidas.
Ejemplo:

- delete audit logs
- disable all security auditing

remove evidence of break-glass usage

## 35. Immutable Audit

Break-glass identity no deberá poder borrar su propio audit trail.

## 36. Break-Glass Credentials Storage

Documento 31.

- Preferir:
- KMS
- HSM
- offline secure storage
- hardware key
- según modelo.

## 37. Emergency Secret

Si se utiliza:

- high entropy
- single-purpose
- rotatable
- versioned
- audited

## 38. Secret Hashing

Si verification puede realizarse mediante hash/verifier, no guardar plaintext.

## 39. Break-Glass Test

Un mecanismo nunca probado puede no servir durante una emergencia.
Debe soportarse:
controlled emergency access drill

## 40. Drill Mode

Debe diferenciarse de incidente real.
BREAK_GLASS_DRILL

## 41. Drill Audit

También deberá auditarse.

## 42. Production Safety

Drill no deberá otorgar más privilegios que los configurados.

## 43. Emergency Credential Health

Sistema podrá verificar periódicamente:

- credential exists
- not expired
- certificate valid
- hardware key registered
- required key accessible

sin utilizar el credential para autenticarse.

## 44. BreakGlassHealthCheck

interface BreakGlassHealthCheckInterface
{
public function check(): BreakGlassHealthReport;
}
45. No credential disclosure in health report
46. Administrative Login Entry Point

Documento 22 deberá soportar entry points distintos:

- /login
- /admin/login
- /security/login
- /emergency/login

si deployment lo requiere.

## 47. Entry Point Policy

Cada uno podrá tener:

- different authenticators
- different MFA requirements
- different session profile
- different risk policy
- different failure response

## 48. Privileged Entry Point Discovery

No debe revelar información sensible innecesaria.

## 49. Admin Username Enumeration

Mismas protecciones contra enumeration aplican.

## 50. Throttling

Documento 19.
Privileged endpoints deberán tener throttling específico.

## 51. Break-Glass Throttling

Debe protegerse contra brute force.
Pero evitar lockout permanente que destruya su utilidad durante una emergencia.

## 52. Emergency Lockout Policy

Puede usar:

- progressive delay
- security alert
- manual unlock
- hardware-factor requirement

en vez de un lockout irreversible.

## 53. Credential Stuffing

Break-glass accounts deberán excluirse de patrones inseguros como passwords reutilizadas.

## 54. IP Restrictions

Podrán ser defense-in-depth:

- corporate network
- VPN
- management network
- bastion

pero no sustituir Authentication.

## 55. Network Restriction Failure

Break-glass policy deberá considerar si emergencia incluye pérdida de red administrativa.

## 56. Risk Engine

Break-glass access siempre debería producir señal de riesgo alta.
Pero no necesariamente bloquearlo automáticamente si precisamente existe para incident response.

## 57. Special Risk Policy

break-glass risk
debe tratarse distinto de login normal.

## 58. Security Signal

Documento 20 recibirá:

- BREAK_GLASS_AUTHENTICATION
- PRIVILEGE_ELEVATION
- SENSITIVE_REAUTHENTICATION
- ADMIN_AUTHENTICATION

## 59. Sensitive Operation Token

Para SPA/API puede ser útil emitir proof opaco.
Ejemplo:

- X-VoltStack-Reauth-Proof
- pero deberá ser cuidadosamente protegido.

## 60. Preferred SPA Design

POST /tenant/delete
↓
428 Reauthentication Required
↓
Frontend opens challenge
↓
Passkey verification
↓
Proof created
↓
Original intent retried

## 61. HTTP Response

VoltStack podrá definir código/protocolo interno.
No asumir que HTTP 401 cubre toda la semántica.

## 62. Authentication Challenge Response

Ejemplo conceptual:

```php
{
    "error": "reauthentication_required",
    "challenge": "...",
    "requirements": {
        "fresh": true,
        "phishing_resistant": true
    }
}
```

## 63. No secret policy disclosure

Frontend puede conocer métodos aceptables.
No necesita recibir detalles internos del Risk Engine.

## 64. SPA Intent Preservation

El frontend puede preservar:

- route
- form state
- operation intent

sin ejecutar operación hasta completar challenge.

## 65. No automatic replay of arbitrary requests

Especialmente:

- financial transaction
- destructive operation
- external side effect

Debe existir idempotency/intent mechanism.

## 66. SensitiveOperationIntent

final readonly class SensitiveOperationIntent
{
public function __construct(
public string $id,
public SensitiveOperation $operation,
public string $subjectId,
public DateTimeImmutable $expiresAt,
) {}
}

## 67. Intent Binding

Proof podrá vincularse a:
IntentId

## 68. Payload Binding

Para operaciones extremadamente sensibles puede vincularse hash del payload.
operation:
wire.transfer

amount:
100000

destination:

- XYZ
- La prueba no deberá servir para otro payload.

## 200. Transaction Authentication

Arquitectura extensible para:

- What You See Is What You Sign
- en perfiles avanzados.

## 201. OperationDigest

final readonly class SensitiveOperationDigest
{
public function __construct(
public string $algorithm,
public string $digest,
) {}
}

## 202. Passkey Confirmation

WebAuthn puede utilizarse como parte de una confirmación fuerte, según protocolo y UX.

## 203. User Presence

Sensitive policy podrá exigir:
user presence

## 204. User Verification

También:

- user verification
- cuando authenticator lo soporte.

## 205. Reauthentication Method Selection

Debe reutilizar el sistema de Authenticator Resolution de documentos 06 y 07.

## 206. No duplicar Authenticator System

Flujo:

```text
Reauthentication Requirement
        ↓
Authenticator Resolver
        ↓
Eligible Authenticators
        ↓
Challenge
```

## 207. Authenticator Eligibility

Algunos authenticators podrán estar prohibidos para reauthentication crítica.
Ejemplo:

- email magic link
- puede no satisfacer HIGH.

## 208. Recovery Authentication

Recovery factor podrá autenticar, pero policy puede marcar:
assurance = restricted

## 209. Restricted Authentication Context

Después de recovery:

```text
can:
    enroll new MFA
```

cannot:

```text
    delete tenant
    rotate platform key
    create admin token
```

## 210. Restricted Context

final readonly class RestrictedAuthenticationContext
{
public function __construct(
public array $allowedRecoveryOperations,
public DateTimeImmutable $expiresAt,
) {}
}

## 211. Credential Change Reauthentication

Cambiar password deberá normalmente requerir conocimiento o autenticación fresca apropiada.

## 212. Federated User

Puede no tener password local.

- Entonces no exigir:
- current password
- si no existe.

Exigir evidencia compatible con identity origin.

## 213. Passwordless User

Igual.

## 214. Authentication-Method-Aware Reauthentication

Policy debe solicitar métodos disponibles y válidos para esa identidad.

## 215. MFA Disable

Debe ser especialmente protegido.

## 216. MFA Disable Flow

Authenticated User
↓
Fresh Authentication
↓
Strong Factor
↓
Risk Evaluation
↓
Disable MFA
↓
Security Epoch Increment
↓
Revoke Sensitive Proofs
↓
Audit
↓
Notify User

## 217. Passkey Removal

Si se elimina el último factor fuerte:

- higher confirmation requirements
- pueden aplicarse.

## 218. Last Authentication Factor

No permitir dejar cuenta inaccesible accidentalmente salvo recovery plan.

## 219. Last Admin

Operaciones que afectan al último administrador pertenecen también a Authorization/business invariants.
Authentication deberá soportar stronger proof.

## 220. Change Primary Email

Aunque no sea exclusivamente Authentication, puede afectar recovery.
Debe clasificarse como sensitive.

## 221. Change Recovery Contact

Muy sensible.

## 222. Create API Token

Puede requerir:
fresh authentication

## 223. Create High-Privilege API Token

Puede exigir:

- fresh
- +;
- phishing-resistant
- +;
- privileged context

## 224. Reveal Secret

Mostrar nuevamente una credencial sensible podrá requerir reauthentication.

## 225. Secret Export

Igual.

## 226. Cryptographic Key Operations

Documento 31.

- Operaciones:
- key.generate
- key.activate
- key.rotate
- key.revoke
- key.destroy

deberán tener policies privilegiadas.

## 227. Key Destroy

Probablemente:

- Privileged
- Fresh
- Phishing-resistant
- Dual-control
- según deployment.

## 228. Authentication Policy Changes

Modificar:

- MFA requirements
- password policy
- OIDC providers
- trusted CAs
- break-glass configuration
- es altamente sensible.

## 229. Self-Weakening Security Change

Sistema debe detectar acciones que reducen protección del propio actor.
Ejemplo:
admin disables own MFA

## 230. Security Downgrade Guard

interface AuthenticationSecurityDowngradeGuardInterface
{
public function evaluate(
AuthenticationSecurityChange $change,
AuthenticationContext $context
): SecurityDowngradeDecision;
}
231. Downgrade may require stronger authentication
232. Administrative Separation

Security admin y application admin pueden tener realms/policies diferentes.

## 233. Service Accounts

No deben usar reauthentication interactiva como humanos.

## 234. Machine Identity Sensitive Operations

Deberán usar:

- short-lived credentials
- mTLS
- workload identity
- signed requests
- capabilities
- approval workflow
- según architecture.

## 235. No fake MFA for service accounts

## 236. Human vs Machine Authentication

Requirement resolver deberá conocer principal type.

## 237. CLI Administrative Operations

VoltStack CLI también deberá respetar privileged Authentication.

## 238. CLI Reauthentication

Podrá utilizar:

- passkey/security key
- device flow
- certificate
- short-lived admin credential

## 239. No assumption that localhost = trusted admin

## 240. Console Authentication Context

Debe ser explícito.

## 241. Queue Workers

Background jobs no pueden realizar reauthentication humana en hot path.

## 242. Sensitive Job Authorization

La intención deberá haber sido aprobada/autenticada antes de enqueue cuando corresponda.

## 243. Proof Propagation to Jobs

No enviar raw reusable proof indiscriminadamente.
Preferir:
signed operation authorization record
con:

- operation
- actor
- scope
- expiresAt
- policyVersion

## 244. Job Execution Revalidation

Antes de side effect crítico:

- account state
- authorization
- security epoch
- operation approval
- según semantics.

## 245. Long-Running Workflow

Si dura horas/días, reauthentication proof inicial puede expirar.
Workflow deberá tener su propia approval/security semantics.
246. Reauthentication no debe mantenerse viva artificialmente
247. Event System

Documento 23.

```text
Eventos:
ReauthenticationRequired
ReauthenticationStarted
ReauthenticationSucceeded
ReauthenticationFailed
StepUpRequired
StepUpSucceeded
StepUpFailed

PrivilegedAuthenticationStarted
PrivilegedAuthenticationEstablished
PrivilegedAuthenticationExpired

SensitiveOperationRequested
SensitiveOperationAuthenticated
SensitiveOperationRejected

BreakGlassRequested
BreakGlassActivated
BreakGlassExpired
```

## 248. Events no deben contener credenciales

## 249. Audit

Documento 24.

- Registrar:
- Actor
- Subject
- Tenant
- Realm
- Operation
- Authentication Requirement
- Methods Used
- Assurance
- Freshness
- Risk Decision
- Device State
- Proof ID
- Outcome
- Timestamp
- sin secrets.

## 250. Privileged Audit

Debe poder reconstruir:

- Who elevated?
- When?

Using what authentication class?
For what scope?
What operations were performed?
When did elevation expire?

## 251. Break-Glass Audit

Debe ser aún más detallado.

## 252. Explainability

Ejemplo:
Sensitive operation denied.

```text
Operation:
platform.key.destroy
```

Required:

- PRIVILEGED assurance
- Phishing-resistant authentication

Fresh authentication <= 5 minutes

Current:

- HIGH assurance
- Passkey verified 18 minutes ago

## 253. User-Facing Error

No necesita mostrar todos los detalles internos.
Puede decir:
Additional authentication is required.

## 254. Failure Taxonomy

Documento 25.

```text
Errores:
REAUTHENTICATION_REQUIRED
REAUTHENTICATION_FAILED
REAUTHENTICATION_EXPIRED

STEP_UP_REQUIRED
STEP_UP_FAILED

AUTHENTICATION_ASSURANCE_INSUFFICIENT
AUTHENTICATION_NOT_FRESH

TRUSTED_DEVICE_REQUIRED
PHISHING_RESISTANT_AUTHENTICATION_REQUIRED

PRIVILEGED_AUTHENTICATION_REQUIRED
PRIVILEGED_SESSION_EXPIRED

BREAK_GLASS_DISABLED
BREAK_GLASS_APPROVAL_REQUIRED
BREAK_GLASS_AUTHENTICATION_FAILED
BREAK_GLASS_SESSION_EXPIRED

SENSITIVE_OPERATION_PROOF_INVALID
SENSITIVE_OPERATION_PROOF_EXPIRED
SENSITIVE_OPERATION_PROOF_REPLAYED
```

## 255. No Enumeration

Errores de privileged login no deberán revelar:

- admin exists
- break-glass account exists

a actores no confiables.

## 256. Metrics

Ejemplos:

```text
auth_reauthentication_required_total
auth_reauthentication_success_total
auth_reauthentication_failure_total

auth_step_up_required_total
auth_step_up_success_total

auth_privileged_context_created_total
auth_privileged_context_expired_total

auth_sensitive_operation_denied_total

auth_break_glass_activation_total
auth_break_glass_failure_total
```

## 257. Metric Labels

Permitidos:

- realm
- operation_class
- assurance_level
- outcome
- con cardinalidad controlada.

## 258. No User ID in high-cardinality metrics

## 259. Tracing

Span:

- auth.sensitive_operation.evaluate
- auth.reauthentication.challenge
- auth.step_up
- auth.privilege_elevation
- auth.break_glass.authenticate

## 260. Sensitive Trace Data

Nunca:

- password
- OTP
- recovery code
- private key
- raw proof token

## 261. Runtime Performance

La evaluación normal deberá ser rápida.

- Idealmente:
- AuthenticationContext
- +;
- Compiled Requirement
- sin consultas innecesarias.

## 262. Compiled Sensitive Policies

Documento 27.

```text
Podrá compilar:
operation → effective static requirement
```

## 263. Dynamic Signals

No compilar:

- current risk
- current device state
- current session freshness

## 264. Policy Compilation

Configuration
↓
Validation
↓
Sensitive Operation Registry
↓
Compiled Requirement Graph

## 265. Configuration Validation

Detectar:

- unknown assurance
- unknown authenticator
- impossible factor combination
- negative freshness
- missing break-glass provider

privileged policy weaker than platform floor

## 266. Impossible Policy

Ejemplo:

```text
require:
    passkey
```

but:

```text
    passkey authenticator disabled
Debe detectarse cuando sea estáticamente posible.
```

## 267. FrankenPHP

El sistema deberá ser seguro con workers persistentes.

## 268. Nunca almacenar Privileged Context globalmente

No:
static $isAdminAuthenticated = true;

## 269. Request Context

Privileged state deberá ser:

- request-local
- session-bound
- immutable snapshot

## 270. Worker Reset

Al terminar request:

- ReauthenticationChallengeContext
- SensitiveOperationContext
- PrivilegedRuntimeContext
- deben liberarse/resetearse.

## 271. Fiber Safety

Dos requests concurrentes dentro del mismo worker no deberán compartir:

- proof
- privileged state
- break-glass state
- operation intent

## 272. Authentication Context Immutability

Preferir value objects readonly.

## 273. Caching

Puede cachearse:

- compiled requirements
- operation metadata
- policy graph

No cachear globalmente:

- current user's privileged state
- reauth proof
- current risk decision

## 274. Distributed Runtime

Documento 30.
Reauthentication proofs server-side deberán funcionar en cluster.

## 275. Proof Store

Puede utilizar:

- Redis
- distributed cache
- database
- signed stateless proof
- según profile.

## 276. Stateless Proof

Si se permite:

- signed
- short-lived
- audience-bound
- session-bound
- operation-bound

nonce/replay protected where necessary

## 277. Stateful Proof

Facilita:

- revocation
- single-use
- replay detection

## 278. Default Recommendation

Para operaciones altamente sensibles:

- stateful short-lived proof
- es preferible.

## 279. Proof Replication

Debe respetar cluster consistency.

## 280. Proof Revocation Propagation

Logout o security epoch change deberá invalidarlo en todos los nodos.

## 281. Privileged Session Revocation

Debe propagarse rápidamente.

## 282. Break-Glass Revocation

Aún más importante.

## 283. Fail Closed

Si no puede verificarse proof de operación crítica:
deny

## 284. Distributed Store Outage

No asumir proof válido.

## 285. Availability Profiles

Algunas operaciones normales pueden degradar.
Privileged/break-glass deberá tener semantics explícitas.

## 286. Break-Glass Paradox

Break-glass existe precisamente cuando componentes fallan.

- Por tanto, no deberá depender innecesariamente de:
- primary IdP
- normal MFA SaaS
- normal session store

si el threat model exige independencia.

## 287. Emergency Runtime

Puede existir:

- EmergencyAuthenticationProvider
- aislado.

## 288. Emergency Provider

Debe seguir contratos normales de Authentication tanto como sea posible.

## 289. No secret backdoor

Nunca:

```php
if ($_GET['emergency'] === SECRET) {
    authenticateRoot();
}
```

## 290. Extensibility

Documento 28.

- Extensiones podrán añadir:
- SensitiveOperationClassifier
- AuthenticationRequirementContributor
- ReauthenticationMethod
- StepUpStrategy
- PrivilegedPolicy
- BreakGlassProvider
- BreakGlassApprovalStrategy

## 291. Extension Security Floor

Plugin no podrá reducir platform security floor sin permiso explícito.

## 292. Custom Sensitive Operation

Paquete:

```php
SensitiveOperations::register(
    'payments.refund.large',
    ...
);
```

## 293. Package Metadata

Podrá declarar requisitos.

## 294. Application Override

Puede endurecer.

## 295. Weakening Override

Debe requerir configuración explícita y validación.

## 296. Testing Strategy

Documento 26 deberá incluir pruebas específicas de este subsistema.

## 297. Test — Session Valid but Stale

session valid
auth age > requirement

Expected:
REAUTHENTICATION_REQUIRED

## 298. Test — Fresh Authentication

auth age < requirement

Expected:
SUFFICIENT

## 299. Test — MFA Required

Password-only context:

- Expected:
- STEP_UP_REQUIRED

## 300. Test — Phishing Resistance

Password + SMS
no satisface:
phishing_resistant = true

## 301. Test — Passkey

Passkey fresh satisface policy compatible.

## 302. Test — Wrong Operation Proof

Proof para:
auth.password.change
usado en:
tenant.delete
Expected:
INVALID

## 303. Test — Cross-Session Proof

Session A proof usado en Session B.
Rejected.

## 304. Test — Cross-Tenant Proof

Tenant A → Tenant B.
Rejected.

## 305. Test — Expired Proof

Rejected.

## 306. Test — Consumed Proof Replay

Rejected cuando single-use.

## 307. Test — Security Epoch Changed

Proof invalidado.

## 308. Test — Logout

Proof invalidado.

## 309. Test — Account Disabled after Reauth

Operation denied.

## 310. Test — Authorization Removed after Reauth

Operation denied.

## 311. Test — Privileged Session Timeout

Expired context no ejecuta operación.

## 312. Test — Privileged Idle Timeout

Igual.

## 313. Test — Remember-Me

Remember-me restored session no restaura privileged context.

## 314. Test — Admin Policy

Admin password-only login debe step-up si MFA obligatorio.

## 315. Test — Recovery Context

Recovery code no obtiene privileged context automáticamente.

## 316. Test — Break-Glass Disabled

Emergency login rechazado.

## 317. Test — Break-Glass Activation

No puede autenticarse antes de activation si policy lo exige.

## 318. Test — Break-Glass Expiration

Session deja de funcionar.

## 319. Test — Break-Glass No Remember-Me

Persistent credential no se emite.

## 320. Test — Break-Glass Audit

Todos los eventos relevantes aparecen.

## 321. Test — Break-Glass Audit Deletion

Break-glass principal no puede eliminar evidence protegido.

## 322. Test — Break-Glass Dual Control

Un solo approver no basta.

## 323. Test — Emergency Secret Rotation

Credential anterior deja de funcionar según rotation policy.

## 324. Test — IdP Outage

Local emergency identity funciona si deployment lo diseñó para ello.

## 325. Test — Brute Force

Privileged/break-glass endpoints aplican throttling.

## 326. Test — Lockout Resilience

Attack no puede inutilizar permanentemente break-glass mediante lockout trivial.

## 327. Test — SPA Challenge

Original operation no se ejecuta antes de completar reauthentication.

## 328. Test — Request Replay

Proof no duplica side effect accidentalmente.

## 329. Test — Payload Binding

Proof para:
amount=100
no acepta:

```php
amount=100000
si operation digest está habilitado.
```

## 330. Test — FrankenPHP Leakage

Request B nunca ve privileged context de Request A.

## 331. Test — Fiber Concurrency

Dos concurrent requests no comparten proof.

## 332. Test — Cluster Proof

Proof creado en Node A puede validarse en Node B cuando profile lo permite.

## 333. Test — Revocation Propagation

Proof revocado en Node A no sigue válido indefinidamente en Node B.

## 334. Test — Store Failure

Critical operation fails closed.

## 335. Test — Policy Merge

Tenant no puede debilitar platform floor.

## 336. Test — Dynamic Risk

Elevated risk aumenta requirement.

## 337. Test — Device Trust

Untrusted device produce step-up/deny según policy.

## 338. Test — Algorithm/Key Operation

Key destruction exige privileged authentication configurada.

## 339. Test — Service Account

No entra accidentalmente en flujo MFA interactivo humano.

## 340. Test — CLI

CLI administrative command aplica Authentication policy.

## 341. Test — Background Job

No reutiliza raw reauthentication proof expirado.

## 342. Security Invariants — Reauthentication

AUTH-PRIV-REAUTH-01
Una sesión válida no implica que Authentication sea suficientemente fresca.
AUTH-PRIV-REAUTH-02
Reauthentication proof tiene lifetime limitado.

- AUTH-PRIV-REAUTH-03
- Proof está ligado al Authentication Context correspondiente.
- AUTH-PRIV-REAUTH-04

Proof expirado nunca autoriza una operación.
AUTH-PRIV-REAUTH-05
Reauthentication no congela Authorization ni account security state.

## 343. Security Invariants — Step-Up

AUTH-PRIV-STEP-01
Step-Up incrementa assurance únicamente mediante evidencia válida.
AUTH-PRIV-STEP-02
Número de factores y phishing resistance son propiedades diferentes.
AUTH-PRIV-STEP-03
Recovery authentication no obtiene automáticamente privileged assurance.
AUTH-PRIV-STEP-04
Un factor no permitido por policy no satisface el requirement aunque sea válido.
AUTH-PRIV-STEP-05
Step-Up permanece limitado por tiempo y scope.

## 344. Security Invariants — Privileged Authentication

AUTH-PRIV-CTX-01
Privileged Authentication y administrative Authorization son independientes.
AUTH-PRIV-CTX-02
Una sesión normal no se convierte permanentemente en privilegiada.
AUTH-PRIV-CTX-03
Privileged contexts tienen lifetime explícito.

- AUTH-PRIV-CTX-04
- Remember-Me no restaura automáticamente privileged context.
- AUTH-PRIV-CTX-05

Privileged context pertenece a identidad, sesión, tenant/realm y scope correctos.

## 345. Security Invariants — Sensitive Operations

AUTH-PRIV-SENS-01
Toda operación clasificada como sensible tiene Authentication Requirement explícito.
AUTH-PRIV-SENS-02
La sensibilidad no se determina únicamente mediante HTTP method.
AUTH-PRIV-SENS-03
Sensitive proof para una operación no puede utilizarse para otra fuera de scope.
AUTH-PRIV-SENS-04
Una operación crítica revalida estado relevante inmediatamente antes de ejecución cuando sea necesario.
AUTH-PRIV-SENS-05
Authentication proof nunca sustituye Authorization.

## 346. Security Invariants — Break-Glass

AUTH-PRIV-BG-01
Break-glass nunca es un bypass oculto.

- AUTH-PRIV-BG-02
- Toda utilización de break-glass tiene identidad explícita.
- AUTH-PRIV-BG-03

Break-glass sessions son temporales y no persistentes.

- AUTH-PRIV-BG-04
- Break-glass nunca emite Remember-Me credentials.
- AUTH-PRIV-BG-05

Break-glass activity siempre es auditable.

- AUTH-PRIV-BG-06
- Break-glass actor no puede eliminar su propio audit trail protegido.
- AUTH-PRIV-BG-07

Break-glass credentials tienen lifecycle y rotation explícitos.
AUTH-PRIV-BG-08
Break-glass no depende innecesariamente del mismo sistema cuya falla debe recuperar.

## 347. Security Invariants — Multi-Tenant

AUTH-PRIV-TENANT-01
Proof Tenant A no eleva Tenant B.

- AUTH-PRIV-TENANT-02
- Tenant policy puede endurecer pero no reducir platform floor.
- AUTH-PRIV-TENANT-03

Platform-admin y tenant-admin contexts son distinguibles.
AUTH-PRIV-TENANT-04
Break-glass scope identifica explícitamente tenant/platform boundary.

## 348. Security Invariants — Distributed Runtime

AUTH-PRIV-DIST-01
Revocación de privileged context se propaga entre nodos.
AUTH-PRIV-DIST-02
Proof validity nunca depende exclusivamente de memoria local cuando debe funcionar en cluster.
AUTH-PRIV-DIST-03
Store outage no convierte proof desconocido en válido.
AUTH-PRIV-DIST-04
Security epoch changes invalidan proofs según policy en todos los nodos.

## 349. Security Invariants — Runtime

AUTH-PRIV-RT-01
Privileged context nunca vive en estado global mutable.

- AUTH-PRIV-RT-02
- Long-lived workers limpian todo request-local elevation state.
- AUTH-PRIV-RT-03

Concurrent requests no comparten reauthentication state.
AUTH-PRIV-RT-04
Sensitive operation metadata puede cachearse; identidad/elevation runtime no.

## 350. Anti-Patterns

Nunca implementar:

```php
$isAuthenticated === true
therefore
```

all sensitive actions allowed

## 351. Anti-Pattern

$isAdmin === true
therefore
MFA unnecessary

## 352. Anti-Pattern

remember-me
↓
restore admin privileged session

## 353. Anti-Pattern

reauthenticated once
↓
all sensitive operations forever

## 354. Anti-Pattern

MFA = phishing resistant

## 355. Anti-Pattern

admin role = high authentication assurance

## 356. Anti-Pattern

recovery code
↓
full privileged access

## 357. Anti-Pattern

break-glass password hardcoded in source

## 358. Anti-Pattern

secret query parameter activates root

## 359. Anti-Pattern

break-glass session lasts 30 days

## 360. Anti-Pattern

break-glass creates remember-me cookie

## 361. Anti-Pattern

tenant can disable mandatory platform admin MFA

## 362. Anti-Pattern

proof from Tenant A accepted in Tenant B

## 363. Anti-Pattern

reauthentication proof bypasses authorization

## 364. Anti-Pattern

proof remains valid after logout
cuando policy exige session binding.

## 365. Anti-Pattern

KMS/store unavailable
↓
assume privileged proof valid

## 366. Anti-Pattern

static $privileged = true
en FrankenPHP.

## 367. Componentes principales

AuthenticationRequirement
EffectiveAuthenticationRequirement
AuthenticationAssuranceLevel
AuthenticationFreshness
AuthenticationRequirementResolver
AuthenticationRequirementEvaluator

SensitiveOperation
SensitiveOperationDefinition
SensitiveOperationRegistry
SensitiveOperationClassifier
SensitiveOperationGuard
SensitiveOperationIntent
SensitiveOperationDigest

ReauthenticationManager
ReauthenticationChallenge
ReauthenticationProof
ReauthenticationProofStore

AuthenticationStepUpManager
StepUpPlan
StepUpAttempt
StepUpResult

PrivilegedAuthenticationContext
PrivilegedAuthenticationManager
AdministrativeAuthenticationPolicy

BreakGlassAccount
BreakGlassPolicy
BreakGlassActivation
BreakGlassAuthenticationProvider
BreakGlassHealthCheck

## 368. Namespace sugerido

VoltStack\Quantum\Auth\Privileged
VoltStack\Quantum\Auth\Privileged\Contracts
VoltStack\Quantum\Auth\Privileged\Requirement
VoltStack\Quantum\Auth\Privileged\Sensitive
VoltStack\Quantum\Auth\Privileged\Reauthentication
VoltStack\Quantum\Auth\Privileged\StepUp
VoltStack\Quantum\Auth\Privileged\Administrative
VoltStack\Quantum\Auth\Privileged\BreakGlass
VoltStack\Quantum\Auth\Privileged\Proof
VoltStack\Quantum\Auth\Privileged\Runtime
VoltStack\Quantum\Auth\Privileged\Events
VoltStack\Quantum\Auth\Privileged\Audit

## 369. Estructura sugerida

src/Quantum/Auth/Privileged/
├── Contracts/
│   ├── AuthenticationRequirementResolverInterface.php
│   ├── AuthenticationRequirementEvaluatorInterface.php
│   ├── ReauthenticationManagerInterface.php
│   ├── ReauthenticationProofStoreInterface.php
│   ├── AuthenticationStepUpManagerInterface.php
│   ├── SensitiveOperationRegistryInterface.php
│   ├── SensitiveOperationClassifierInterface.php
│   ├── SensitiveOperationGuardInterface.php
│   ├── AdministrativeAuthenticationPolicyInterface.php
│   └── BreakGlassHealthCheckInterface.php
│
├── Requirement/
│   ├── AuthenticationRequirement.php
│   ├── EffectiveAuthenticationRequirement.php
│   ├── AuthenticationAssuranceLevel.php
│   ├── AuthenticationFreshness.php
│   ├── AuthenticationRequirementResolver.php
│   └── AuthenticationRequirementEvaluator.php
│
├── Sensitive/
│   ├── SensitiveOperation.php
│   ├── SensitiveOperationDefinition.php
│   ├── SensitiveOperationRegistry.php
│   ├── SensitiveOperationClassifier.php
│   ├── SensitiveOperationGuard.php
│   ├── SensitiveOperationIntent.php
│   └── SensitiveOperationDigest.php
│
├── Reauthentication/
│   ├── ReauthenticationChallenge.php
│   ├── ReauthenticationChallengeId.php
│   ├── ReauthenticationAttempt.php
│   ├── ReauthenticationResult.php
│   ├── ReauthenticationManager.php
│   └── ReauthenticationProof.php
│
├── Proof/
│   ├── ReauthenticationProofStore.php
│   ├── ReauthenticationScope.php
│   ├── StatefulProofStore.php
│   └── StatelessProofCodec.php
│
├── StepUp/
│   ├── AuthenticationStepUpManager.php
│   ├── StepUpPlan.php
│   ├── StepUpAttempt.php
│   └── StepUpResult.php
│
├── Administrative/
│   ├── PrivilegedAuthenticationContext.php
│   ├── PrivilegedAuthenticationManager.php
│   ├── AdministrativeAuthenticationContext.php
│   ├── AdministrativeAuthenticationPolicy.php
│   └── AuthenticationSecurityDowngradeGuard.php
│
├── BreakGlass/
│   ├── BreakGlassAccount.php
│   ├── BreakGlassPolicy.php
│   ├── BreakGlassActivationRequest.php
│   ├── BreakGlassActivation.php
│   ├── BreakGlassAuthenticationProvider.php
│   ├── BreakGlassHealthCheck.php
│   └── BreakGlassHealthReport.php
│
├── Runtime/
│   ├── PrivilegedRuntimeContext.php
│   ├── SensitiveOperationRuntimeContext.php
│   └── PrivilegedRuntimeResetter.php
│
└── Events/
├── ReauthenticationRequired.php
├── ReauthenticationSucceeded.php
├── StepUpRequired.php
├── PrivilegedAuthenticationEstablished.php
├── SensitiveOperationRequested.php
└── BreakGlassActivated.php

## 370. Configuración conceptual

return [

'authentication' => [

'privileged' => [

'default_freshness' => '10 minutes',

'admin' => [
'require_mfa' => true,
'phishing_resistant' => true,
'session_lifetime' => '30 minutes',
'idle_timeout' => '10 minutes',
],

'break_glass' => [
'enabled' => false,
'session_lifetime' => '15 minutes',
'remember_me' => false,
],

],

],

];

## 371. Sensitive Operations Configuration

'sensitive_operations' => [

'auth.password.change' => [
'fresh_within' => '10 minutes',
],

'auth.mfa.disable' => [
'fresh_within' => '5 minutes',
'mfa' => true,
],

'tenant.delete' => [
'fresh_within' => '5 minutes',
'assurance' => 'high',
'phishing_resistant' => true,
],

'platform.key.destroy' => [
'fresh_within' => '3 minutes',
'assurance' => 'privileged',
'phishing_resistant' => true,
'trusted_device' => true,
],

];

## 372. Flujo normal

REQUEST
↓
Authenticated?
↓
Authorized?
↓
Sensitive?
↓
Resolve Authentication Requirement
↓
Evaluate Current Authentication Context
↓
Sufficient?
┌─────┴─────┐
▼           ▼
YES          NO
│            │
Execute    Challenge
↓
Reauthenticate
↓
Step-Up
↓
Proof Created
↓
Re-Evaluate
↓
Execute

## 373. Flujo privilegiado

Normal Session
↓
Privileged Operation
↓
Privilege Elevation Required
↓
Fresh Strong Authentication
↓
Risk + Device Evaluation
↓
Privileged Context Created
↓
Short Lifetime
↓
Privileged Operations
↓
Expiration / Revocation
↓
Normal Session Remains

## 374. Flujo Break-Glass

Emergency
↓
Break-Glass Enabled?
↓
Activation Required?
↓
Approval / Dual Control
↓
Emergency Authentication
↓
Strong Credential Verification
↓
Break-Glass Session
↓
Restricted Emergency Scope
↓
Full Audit + Alert
↓
Automatic Expiration
↓
Credential Rotation / Incident Review

## 375. Relación con Laravel

Laravel ofrece mecanismos útiles como:

- Auth
- Guards
- Password Confirmation
- password.confirm middleware
- Authentication events
- Fortify
- Sanctum
- session authentication

Especialmente:
password.confirm
introduce la idea de confirmar nuevamente una credencial antes de ciertas operaciones.
VoltStack deberá conservar esa simplicidad de developer experience, pero generalizar el concepto.
En lugar de limitarse a:
password recently confirmed?
VoltStack deberá poder responder:

- Is authentication fresh?
- Is MFA fresh?
- Is assurance sufficient?

Is phishing-resistant authentication required?
Is device trusted?
Is privileged context required?

## 376. Relación con Symfony

Symfony aporta conceptos útiles mediante:

- Security
- Authenticators
- Badges
- Passport
- Access Control
- Login throttling
- User Checkers
- Authentication events

VoltStack deberá mantener la separación contractual y el modelo de evidence/authenticator desarrollado en documentos anteriores, agregando un sistema explícito de requirements y proofs para operaciones sensibles.

## 377. Diferenciador VoltStack

El modelo será:

- Laravel-like developer simplicity
- +;
- Symfony-like authentication composition
- +;
- Authentication Assurance
- +;
- Authentication Freshness
- +;
- Generic Reauthentication
- +;
- Step-Up Authentication
- +;
- Sensitive Operation Policies
- +;
- Privileged Contexts
- +;
- Administrative Realms
- +;
- Break-Glass Authentication
- +;
- Risk-aware Requirements
- +;
- Device-aware Requirements
- +;
- Tenant-aware Security Floors
- +;
- Distributed Proof Revocation
- +;
- FrankenPHP-safe Runtime

## 378. Developer Experience

Una operación simple podrá declararse:

```php
# [SensitiveOperation(
    freshWithin: '10 minutes'
)]
public function changePassword()
{
}
```

Una crítica:

```php
# [SensitiveOperation(
    assurance: 'privileged',
    freshWithin: '5 minutes',
    phishingResistant: true,
    trustedDevice: true,
)]
public function rotatePlatformKey()
{
}
```

VoltStack resolverá internamente:

```text
Authentication Context
        ↓
Requirement
        ↓
Risk
        ↓
Device
        ↓
Authenticator Selection
        ↓
Reauthentication / Step-Up
        ↓
Proof
        ↓
Execution Guard
```

## 379. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Authentication is not binary for sensitive operations

## 2. Session validity and Authentication freshness are independent

## 3. Sensitive operations declare Authentication Requirements

## 4. Reauthentication is generic and not limited to passwords

## 5. Step-Up increases Authentication Assurance

## 6. MFA and phishing resistance are modeled separately

## 7. Privileged Authentication is temporary

## 8. Administrative Authorization never derives from stronger Authentication

## 9. Remember-Me never restores privileged state automatically

## 10. Reauthentication proofs are short-lived and scoped

## 11. Proofs are session/tenant/realm bound by default

## 12. Proofs never freeze Authorization

## 13. Risk can dynamically strengthen Authentication Requirements

## 14. Device trust can strengthen requirements but never replace identity

## 15. Recovery authentication can produce restricted Authentication state

## 16. Security-weakening operations receive stronger protection

## 17. Break-glass is explicit Authentication, never a hidden bypass

## 18. Break-glass sessions are temporary and fully auditable

## 19. Break-glass never emits persistent-login credentials

## 20. Break-glass dependencies are designed independently enough to survive the failures it exists to recover

## 21. Tenant policies may strengthen but not weaken platform security floors

## 22. Privileged state never resides in global mutable runtime state

## 23. Distributed proof revocation is first-class

## 24. Critical operations fail closed when proof cannot be verified

## 25. The entire system remains safe under FrankenPHP persistent workers

## 26. Criterios de aceptación

Este subsistema será considerado completo cuando VoltStack soporte:
27. Authentication Assurance;
28. Authentication Freshness;
29. method-specific freshness;
30. generic Authentication Requirements;
31. Sensitive Operation Registry;
32. Sensitive Operation Classification;
33. declarative sensitive-operation metadata;
34. Reauthentication Challenges;
35. challenge expiration;
36. challenge binding;
37. challenge anti-replay;
38. Reauthentication Proofs;
39. proof scope;
40. proof expiration;
41. proof session binding;
42. proof tenant binding;
43. proof realm binding;
44. proof revocation;
45. Step-Up Authentication;
46. factor-class requirements;
47. phishing-resistant requirements;
48. trusted-device requirements;
49. risk-aware requirements;
50. Privileged Authentication Context;
51. short-lived privileged sessions;
52. privileged idle timeout;
53. privileged absolute timeout;
54. Administrative Authentication Policies;
55. separate administrative realms;
56. optional separate admin sessions;
57. no privileged Remember-Me restoration;
58. recovery-restricted Authentication;
59. Security Downgrade Guard;
60. Sensitive Operation Intent;
61. operation-bound proofs;
62. optional payload-bound proofs;
63. SPA reauthentication protocol;
64. CLI privileged authentication;
65. machine identity handling;
66. distributed proof stores;
67. proof revocation propagation;
68. security epoch integration;
69. multi-tenant security floors;
70. Break-Glass Accounts;
71. Break-Glass Activation;
72. optional Dual Control;
73. Break-Glass Session;
74. Break-Glass restricted scope;
75. Break-Glass expiration;
76. Break-Glass audit;
77. Break-Glass alerting;
78. Break-Glass health checks;
79. emergency credential rotation;
80. emergency drills;
81. throttling integration;
82. Risk Engine integration;
83. Device Trust integration;
84. Authentication Events integration;
85. Observability integration;
86. FrankenPHP worker isolation.
87. Regla arquitectónica final

El sistema deberá entender la autenticación como:

```text
                    AUTHENTICATED IDENTITY
                             │
                             ▼
                   AUTHENTICATION CONTEXT
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
       Assurance         Freshness            Risk
           │                 │                 │
           └─────────────────┼─────────────────┘
                             ▼
                        Device Trust
                             │
                             ▼
                  SENSITIVE OPERATION
                             │
                             ▼
                AUTHENTICATION REQUIREMENT
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
             SUFFICIENT              INSUFFICIENT
                 │                       │
                 │                 ┌─────┴─────┐
                 │                 ▼           ▼
                 │             REAUTH       STEP-UP
                 │                 │           │
                 │                 └─────┬─────┘
                 │                       ▼
                 │                     PROOF
                 │                       │
                 └─────────────┬─────────┘
                               ▼
                         AUTHORIZATION
                               │
                               ▼
                            EXECUTE
```

Para administración:

```text
NORMAL AUTHENTICATION
        ↓
STRONG REAUTHENTICATION
        ↓
PRIVILEGED CONTEXT
        ↓
SHORT-LIVED PRIVILEGED ACCESS
        ↓
EXPIRATION
        ↓
NORMAL CONTEXT
```

Y para emergencias:

```text
NORMAL AUTHENTICATION FAILURE / EMERGENCY
                    ↓
             BREAK-GLASS POLICY
                    ↓
              EXPLICIT ACTIVATION
                    ↓
            EMERGENCY AUTHENTICATION
                    ↓
            TEMPORARY LIMITED ACCESS
                    ↓
             COMPLETE AUDIT TRAIL
                    ↓
               AUTO-EXPIRATION
                    ↓
        INCIDENT REVIEW / ROTATION
```

La primera regla será:
Authenticated nunca significará automáticamente sufficiently authenticated for every operation.

La segunda:
Authorization determina si una identidad puede ejecutar una operación; Authentication determina si la evidencia con la que esa identidad se presentó es suficientemente fuerte, reciente y apropiada para ejecutarla en ese momento.

La tercera:
VoltStack no limitará la reautenticación a solicitar nuevamente una contraseña; podrá utilizar cualquier Authenticator capaz de satisfacer el AuthenticationRequirement efectivo.

La cuarta:
Los privilegios administrativos no surgirán de Authentication Assurance: demostrar fuertemente quién eres no concede permisos que Authorization no haya otorgado.

La quinta:
Break-glass será un mecanismo de Authentication excepcional, explícito, temporal, limitado y auditable; jamás una contraseña maestra, ruta oculta, parámetro secreto o bypass hardcoded.

La sexta:
Toda elevación, reautenticación y acceso de emergencia deberá permanecer correctamente aislado por identidad, sesión, tenant, realm, operación y request incluso bajo workers persistentes de FrankenPHP.

Siguiente documento recomendado
La siguiente pieza natural del sistema sería:
`33_AUTHENTICATION_SERVICE_WORKLOAD_MACHINE_TO_MACHINE_AND_NON_HUMAN_IDENTITY_SYSTEM.md`
porque hasta este punto hemos diseñado profundamente la autenticación de usuarios humanos, mientras que VoltStack también necesitará tratar como dominio de primera clase:

- Service Accounts
- Machine Identities
- Workload Identities
- Application Identities
- Microservices
- Background Workers
- Queue Workers
- Scheduled Jobs
- CLI Automation
- CI/CD Pipelines
- Server-to-Server Authentication
- mTLS
- Client Credentials
- Signed Requests
- Workload Identity Federation
- Short-Lived Machine Credentials
- Credential Rotation
- Service Identity Impersonation
- Machine Authentication Assurance
- Tenant-Bound Service Identities

Esto evitaría cometer el error común de intentar autenticar servicios, workers y microservicios como si fueran usuarios humanos.
