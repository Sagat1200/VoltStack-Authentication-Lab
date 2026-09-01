# VoltStack Authentication System

## 35 — Authentication Session, Device, Credential Inventory, Security Center and User Security Management System

- **Archivo:** `35_AUTHENTICATION_SESSION_DEVICE_CREDENTIAL_INVENTORY_SECURITY_CENTER_AND_USER_SECURITY_MANAGEMENT_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica
- **Nivel:** Core / Security-Critical / User Security Management
- **Dependencias principales:** 10, 12, 13, 14, 15, 16, 17, 18, 20, 21, 23, 24, 25, 29, 30, 31, 32, 33 y 34.

---

## 1. Propósito

Este documento define el subsistema encargado de presentar, administrar y gobernar de forma centralizada el estado de seguridad relacionado con una identidad autenticada.
VoltStack deberá proporcionar una abstracción equivalente a un:
IDENTITY SECURITY CENTER
capaz de unificar:

- Authentication Methods
- Active Sessions
- Remember-Me Credentials
- Trusted Devices
- Registered Devices
- Passkeys
- MFA Factors
- Federated Accounts
- Recovery Methods
- API Credentials
- Machine Delegations
- Authentication Activity
- Security Events
- Security Alerts
- Credential Compromise State
- Session Revocation
- Device Revocation
- Global Logout
- Security Recommendations

sin acoplar el sistema a:

- Blade
- Vue
- React
- Svelte
- Mobile
- CLI
- Admin UI

El Core deberá exponer un modelo de seguridad independiente de la interfaz.

## 2. Problema arquitectónico

En aplicaciones tradicionales, la seguridad del usuario suele distribuirse entre múltiples páginas y subsistemas:

- /profile/password
- /profile/sessions
- /profile/two-factor
- /profile/passkeys
- /profile/oauth
- /profile/devices
- /profile/tokens

Esto provoca que cada funcionalidad implemente por separado:

- listing
- ownership checks
- revocation
- reauthentication
- audit
- notifications
- risk evaluation
- tenant isolation

VoltStack deberá proporcionar una capa unificada.

## 3. Principio fundamental

El Security Center no será la fuente de autoridad de los objetos de Authentication; será una proyección y capa de gestión coordinada sobre los subsistemas que ya son propietarios de Sessions, Credentials, Devices y Authentication Methods.

## 4. Ejemplo

No crear:
SecurityCenterSession
como duplicado de:
AuthenticationSession
El Security Center deberá consultar:

- Session subsystem
- Device subsystem
- Method subsystem
- Token subsystem
- Recovery subsystem

y producir una representación unificada.

## 5. Arquitectura general

IDENTITY
│
▼
IDENTITY SECURITY CENTER
│
┌────────────────────┼────────────────────┐
▼                    ▼                    ▼
Authentication         Sessions              Devices
Methods                 │                    │
│                    │                    │
├─────────────┬──────┴──────┬─────────────┤
▼             ▼             ▼             ▼
Passkeys      Remember-Me    API Tokens     Recovery
│             │             │             │
└─────────────┴──────┬──────┴─────────────┘
▼
Security Activity
│
▼
Security Actions
│
▼
Audit/Event

## 6. Security Center

Abstracción principal:

```php
interface IdentitySecurityCenterInterface
{
    public function snapshot(
        IdentityReference $identity,
        SecurityCenterContext $context
    ): IdentitySecuritySnapshot;
}
```

## 7. IdentitySecuritySnapshot

Debe ser:

- immutable
- point-in-time
- safe for presentation
- secret-free

## 8. Modelo conceptual

final readonly class IdentitySecuritySnapshot
{
public function __construct(
public IdentitySecuritySummary $summary,
public AuthenticationMethodInventory $methods,
public SessionInventory $sessions,
public DeviceInventory $devices,
public CredentialInventory $credentials,
public SecurityActivityInventory $activity,
public SecurityRecommendationSet $recommendations,
) {}
}

## 9. Snapshot != Live Authority

El snapshot puede quedar desactualizado inmediatamente después de generarse.
Por ello:
Toda acción de seguridad deberá volver a validar el estado autoritativo antes de mutar recursos.

## 10. Ejemplo

UI muestra:

- 3 sesiones activas
- Otro dispositivo revoca una.

Cuando el usuario intenta revocar otra:

```text
Security Center
    ↓
Command
    ↓
Authoritative Session Store
```

y no el snapshot antiguo.

## 11. SecurityCenterContext

Podrá contener:

- Principal
- Tenant
- Realm
- Current Session
- Authorization Context
- Locale
- Presentation Scope

No deberá contener secrets.

## 12. User Security Management

Debe distinguirse entre:

- SELF_SERVICE
- ADMINISTRATIVE
- SECURITY_OPERATIONS

## 13. Self-Service

Usuario gestiona su propia seguridad.

- Ejemplos:
- change password
- add passkey
- remove passkey
- revoke session
- remove trusted device
- unlink provider
- rotate recovery codes

## 14. Administrative Management

Administrador autorizado puede realizar acciones sobre otra Identity.
Ejemplos:

- suspend authentication
- force logout
- revoke compromised credential
- require password reset
- require MFA enrollment

## 15. Security Operations

Operaciones de respuesta a incidentes:

- global credential compromise
- mass revocation
- tenant-wide logout
- force reauthentication
- security epoch increment

## 16. Actor Separation

Toda operación deberá conservar:

- Actor
- Target Identity
- Tenant
- Realm
- Reason

## 17. Self-Service Actor

Actor == Target Identity

## 18. Administrative Actor

Actor != Target Identity

## 19. Authentication Method Inventory

Debe integrar el documento 34.

- Podrá mostrar:
- Password
- Passkeys
- TOTP
- External Accounts
- Enterprise SSO
- Recovery Methods
- Certificates
- Custom Methods

## 20. AuthenticationMethodInventory

final readonly class AuthenticationMethodInventory
{
public function __construct(
public array $methods,
public AuthenticationMethodInventorySummary $summary,
) {}
}

## 21. Method View

Debe contener únicamente datos seguros.

```php
final readonly class SecurityCenterAuthenticationMethod
{
    public function __construct(
        public string $id,
        public string $type,
        public string $status,
        public string $displayName,
        public ?DateTimeImmutable $createdAt,
        public ?DateTimeImmutable $lastUsedAt,
        public bool $currentSessionMethod,
    ) {}
}
```

## 22. No mostrar

password hash
TOTP secret
recovery code
private key
bearer token
refresh token
OIDC token

## 23. Method Security Properties

Podrá mostrar:

- phishing resistant
- hardware backed
- MFA capable
- recovery
- federated
- como metadata simplificada.

## 24. Authentication Method Status

UI podrá representar:

- ACTIVE
- SUSPENDED
- COMPROMISED
- REVOKED
- EXPIRING
- ACTION_REQUIRED

## 25. Method Recommendations

Ejemplos:

- Add another passkey
- Replace compromised password
- Configure recovery codes

Your only MFA factor is expiring

## 26. Session Inventory

El documento 12 define Session Authentication.

- El Security Center deberá proporcionar:
- active sessions
- current session
- recently revoked sessions
- session device metadata
- authentication method
- created time
- last activity

approximate location when available/safe

## 27. SessionInventory

final readonly class SessionInventory
{
public function __construct(
public array $active,
public array $recentlyRevoked = [],
) {}
}

## 28. SecurityCenterSession

final readonly class SecurityCenterSession
{
public function __construct(
public string $id,
public bool $current,
public DateTimeImmutable $createdAt,
public DateTimeImmutable $lastSeenAt,
public ?DateTimeImmutable $expiresAt,
public SessionClientDescriptor $client,
public AuthenticationMethodSummary $authentication,
) {}
}

## 29. Session ID Exposure

No deberá exponerse el verdadero session identifier utilizado como credential.

## 30. Public Session Reference

Usar:

- SessionManagementId
- independiente del session token real.

## 31. SessionManagementId

final readonly class SessionManagementId
{
public function __construct(
public string $value,
) {}
}

## 32. Important Invariant

SessionManagementId
≠
Session Credential

## 33. Current Session

Debe identificarse claramente:
This device / Current session

## 34. Session Client Descriptor

Puede incluir:

- browser family
- OS family
- device class
- first seen
- last seen
- network region

sin intentar fingerprinting invasivo obligatorio.

## 35. User-Agent

Es metadata no confiable.

- Debe considerarse:
- presentation hint
- no Security Identity.

## 36. Approximate Location

Si se utiliza:

- country
- region
- city approximation

deberá considerarse:

- risk/presentation metadata
- no evidencia inequívoca.

## 37. IP Address

Puede utilizarse internamente.
UI podrá mostrar representación parcial según privacy policy.

## 38. Session Revocation

Usuario podrá solicitar:
revoke session X

## 39. SessionRevocationCommand

final readonly class SessionRevocationCommand
{
public function __construct(
public IdentityReference $identity,
public SessionManagementId $session,
public AuthenticationActor $actor,
public SessionRevocationReason $reason,
) {}
}

## 40. Revocation Authority

Security Center delegará a:

- SessionRevocationService
- del subsistema correspondiente.

## 41. Session Revocation Result

REVOKED
ALREADY_REVOKED
NOT_FOUND
NOT_OWNED
DENIED
Externamente algunos estados pueden normalizarse.

## 42. Revoking Current Session

Deberá tener semántica explícita.

```text
Ejemplo:
Revoke Current Session
    ↓
Logout
```

## 43. Other Sessions

Puede revocarlas sin cerrar la actual.

## 44. Sign Out Everywhere

Debe ser operación first-class.
LOGOUT_EVERYWHERE

## 45. Global Logout Flow

User
↓
Fresh Authentication if required
↓
Global Logout Request
↓
Identity Security Version / Session Epoch
↓
Logical Revocation
↓
Physical Cleanup
↓
Current Session handling

## 46. Current Session Policy

Puede:
revoke all including current
o:

- revoke all except current
- según operación explícita.

## 47. APIs separadas

Preferible:

```php
Auth::logoutEverywhere();
Auth::logoutOtherSessions();
```

No boolean ambiguo:
logoutAll(true, false);

## 48. Reauthentication

logoutOtherSessions() puede requerir fresh Authentication.

## 49. Session Provenance

El inventario deberá mostrar, cuando sea apropiado:

- Signed in with Passkey
- Signed in with Google

Signed in with Password + TOTP

## 50. Sensitive Provenance

No revelar detalles que permitan atacar mecanismos internos.

## 51. Remember-Me Inventory

Documento 13.
Persistent credentials deberán administrarse por separado de Sessions.

## 52. Why

Una sesión puede expirar, pero:

- Remember-Me Credential
- puede crear una nueva.

Por tanto:
no active sessions
no significa necesariamente:
no persistent authentication credentials

## 53. PersistentLoginInventory

Podrá listar:

- device/client label
- created
- last used
- expires
- status

## 54. Remember-Me Credential ID

No exponer secret selector/verifier completo si compromete seguridad.

## 55. Revoke Persistent Login

Debe invalidar la familia correspondiente.

## 56. Revoke All Persistent Logins

Puede formar parte de:

- logout everywhere
- según policy.

## 57. Device Inventory

Documento 21.

- VoltStack deberá distinguir:
- Observed Device
- Registered Device
- Trusted Device
- Managed Device
- Credential-Bearing Device

## 58. Device != Session

Un mismo device puede generar muchas Sessions.

```text
Device A
├── Session 1
├── Session 2
└── Session 3
```

## 59. Device != Passkey

Un device puede contener múltiples credentials.

## 60. DeviceInventory

final readonly class DeviceInventory
{
public function __construct(
public array $devices,
) {}
}

## 61. SecurityCenterDevice

final readonly class SecurityCenterDevice
{
public function __construct(
public string $id,
public string $displayName,
public DeviceTrustState $trust,
public DateTimeImmutable $firstSeenAt,
public DateTimeImmutable $lastSeenAt,
public bool $current,
public array $credentials,
) {}
}

## 62. Device Management ID

Igual que Session:

- public DeviceManagementId
- ≠
- secret device credential

## 63. Trusted Device

Security Center podrá permitir:

- Trust
- Untrust
- Revoke
- Rename
- según policy.

## 64. Trusting Device

Debe ser operación sensible.

- Puede requerir:
- fresh authentication
- MFA
- passkey

## 65. Untrusting Device

Normalmente deberá ser más fácil que confiarlo.

## 66. Device Revocation

Puede invalidar:

- device trust
- device credentials
- remember-me credentials
- sessions
- dependiendo de policy.

## 67. Cascade Policy

interface DeviceRevocationCascadePolicyInterface
{
public function determine(
DeviceReference $device,
DeviceRevocationContext $context
): DeviceRevocationCascadePlan;
}

## 68. Example Cascade

Device stolen
↓
Revoke Device Trust
↓
Revoke Remember-Me
↓
Revoke Device-bound Sessions
↓
Revoke Device Credential
↓
Mark Risk Signal

## 69. Device Loss

Deberá existir una acción de alto nivel:
Report Device Lost

## 70. LostDeviceCommand

Puede coordinar múltiples subsistemas.

## 71. Lost vs Compromised

Estados diferentes:

- LOST
- COMPROMISED
- RETIRED

## 72. Credential Inventory

Security Center también deberá representar credentials que no sean directamente métodos humanos de login.
Ejemplos:

- Personal Access Tokens
- API Tokens
- Application Passwords
- Device Credentials
- Machine Delegations
- Recovery Credentials

## 73. Credential Types

Debe existir una clasificación consistente.

- AUTHENTICATION_METHOD
- PERSISTENT_LOGIN
- API_CREDENTIAL
- RECOVERY_CREDENTIAL
- DEVICE_CREDENTIAL
- DELEGATED_CREDENTIAL

## 74. Unified Credential View

final readonly class SecurityCenterCredential
{
public function __construct(
public string $managementId,
public string $type,
public string $displayName,
public CredentialSecurityStatus $status,
public DateTimeImmutable $createdAt,
public ?DateTimeImmutable $lastUsedAt,
public ?DateTimeImmutable $expiresAt,
) {}
}

## 75. Never Secret

El Security Center jamás devolverá:

- actual API token
- remember-me verifier
- recovery code
- secret key

salvo operaciones explícitas de creación donde se muestre exactamente una vez.

## 76. Show-Once Secrets

Al crear:

- Personal Access Token
- Recovery Codes
- Client Secret

el sistema puede devolver secret una sola vez.

## 77. ShowOnceSecret

Abstracción conceptual:

```php
final class ShowOnceSecret
{
    // deliberately non-serializable to durable storage
}
```

## 78. Re-Viewing Secret

No deberá ser posible recuperar el valor original si se almacena correctamente como verifier/hash.
La acción sería:
rotate
no:
show again

## 79. API Credential Inventory

Podrá mostrar:

- name
- created
- last used
- expiry
- scopes summary
- tenant
- status

Authorization scopes se presentan, pero no pertenecen a Authentication Core.

## 80. API Token Revocation

Deberá delegar al token subsystem.

## 81. Credential Compromise

Usuario podrá marcar:
I don't recognize this token
o:
This device was stolen

## 82. Security Response

Un report de compromise puede disparar:

- credential revoke
- session revoke
- device revoke
- security epoch
- risk signal
- notification
- audit

## 83. Recovery Inventory

Documento 18.

- Security Center podrá mostrar:
- Recovery codes configured
- Recovery email configured
- Recovery phone configured
- Recovery contact configured
- dependiendo de policy.

## 84. No Recovery Secret Disclosure

No mostrar códigos existentes nuevamente.

## 85. Recovery Regeneration

Debe ser operación sensible.

## 86. Authentication Activity

Security Center deberá poder mostrar actividad reciente.

- Ejemplos:
- Successful login
- Failed login
- New passkey added
- Password changed
- Google account linked
- Session revoked
- Device trusted
- Recovery codes regenerated
- API token created
- Global logout

## 87. Activity != Raw Audit Log

El usuario no deberá recibir necesariamente el audit interno completo.

## 88. SecurityActivityProjection

Será una proyección segura y user-facing.

## 89. SecurityActivityEntry

final readonly class SecurityActivityEntry
{
public function __construct(
public string $type,
public DateTimeImmutable $occurredAt,
public SecurityActivitySeverity $severity,
public string $summary,
public ?SecurityActivityClient $client,
) {}
}

## 90. Activity Source

Se construirá a partir de:

- Authentication Events
- Audit
- Session Store
- Security Incident Records
- según arquitectura.

## 91. Security Activity Privacy

No mostrar información de otros usuarios.

## 92. Administrative Activity

Un usuario podrá ser informado:

- Administrator revoked your sessions
- sin revelar información interna innecesaria.

## 93. Failed Login Activity

Debe manejarse con cuidado.
Mostrar demasiado detalle puede permitir:

- attacker network intelligence
- pero puede ser útil para el dueño de la cuenta.

## 94. SecurityActivityVisibilityPolicy

interface SecurityActivityVisibilityPolicyInterface
{
public function visibleTo(
AuthenticationActor $viewer,
SecurityAuditEvent $event
): bool;
}

## 95. Suspicious Activity

El sistema podrá destacar eventos:

- new device
- new country
- unusual browser
- new credential
- password recovery
- MFA removed

## 96. Risk Integration

Documento 20.

- Risk Engine podrá producir:
- Security Recommendation
- Security Alert
- Step-Up Requirement

## 97. Security Alerts

Deben diferenciarse de eventos normales.

## 98. SecurityAlert

final readonly class SecurityAlert
{
public function __construct(
public string $id,
public SecurityAlertSeverity $severity,
public string $type,
public DateTimeImmutable $createdAt,
public SecurityAlertStatus $status,
) {}
}

## 99. Alert States

OPEN
ACKNOWLEDGED
RESOLVED
DISMISSED

## 100. Security Alert Types

Ejemplos:

- CREDENTIAL_COMPROMISE
- NEW_DEVICE
- SUSPICIOUS_LOGIN
- IMPOSSIBLE_TRAVEL
- MFA_REMOVED
- RECOVERY_USED
- API_TOKEN_EXPOSED
- EXTERNAL_PROVIDER_CHANGED

## 101. Alert != Risk Signal

Risk Signal:
input para Risk Engine
Security Alert:
user/admin-visible security concern

## 102. User Action on Alert

Podrá ofrecer:

- This was me
- This wasn't me
- Revoke session
- Revoke device
- Change password
- Review methods

## 103. "This Wasn't Me"

Debe ser operación de seguridad first-class.

## 104. CompromiseResponseCommand

final readonly class IdentityCompromiseResponseCommand
{
public function __construct(
public IdentityReference $identity,
public SecurityActivityReference $activity,
public AuthenticationActor $actor,
) {}
}

## 105. Response Plan

Puede incluir:

- revoke suspicious session
- revoke associated remember-me
- mark device untrusted
- increase security epoch
- require password reset
- require MFA review
- notify Security Operations

## 106. "This Was Me"

No deberá simplemente borrar risk history.

- Puede marcar:
- USER_CONFIRMED
- como señal adicional.

## 107. Security Recommendations

VoltStack podrá generar recomendaciones.

## 108. Recommendation != Requirement

Ejemplo:
You should add another passkey.
no necesariamente significa:
login denied until another passkey exists.

## 109. SecurityRecommendation

final readonly class SecurityRecommendation
{
public function __construct(
public string $code,
public SecurityRecommendationPriority $priority,
public string $action,
) {}
}

## 110. Recommendation Examples

ENABLE_MFA
ADD_PASSKEY
ADD_SECOND_PASSKEY
REMOVE_UNUSED_SESSION
ROTATE_OLD_API_TOKEN
REVIEW_RECOVERY_METHODS
REPLACE_COMPROMISED_PASSWORD
REVOKE_UNUSED_DEVICE

## 111. Security Score

VoltStack deberá tener cuidado con un "security score".
Un número como:

- Security Score: 83/100
- puede simplificar demasiado el modelo.

## 112. Default Recommendation

Preferir:

- typed security findings
- sobre un score único.

## 113. Optional Score Adapter

Una aplicación podrá calcularlo mediante plugin/UI.
No deberá ser authority del Authentication Core.

## 114. IdentitySecuritySummary

Puede mostrar:

- Authentication methods: 4
- Passkeys: 2
- MFA: enabled
- Active sessions: 3
- Trusted devices: 2
- Security alerts: 1
- Recovery configured: yes

## 115. No Binary "Account Secure"

Evitar:

- Your account is secure.
- como garantía absoluta.

## 116. Security Posture

Puede clasificarse:

- NORMAL
- ACTION_RECOMMENDED
- ACTION_REQUIRED
- COMPROMISED
- RESTRICTED

## 117. IdentitySecurityPosture

enum IdentitySecurityPosture: string
{
case Normal = 'normal';

case ActionRecommended = 'action_recommended';

case ActionRequired = 'action_required';

case Restricted = 'restricted';

case Compromised = 'compromised';
}

## 118. Posture Resolver

interface IdentitySecurityPostureResolverInterface
{
public function resolve(
IdentitySecuritySnapshot $snapshot,
IdentitySecurityState $state
): IdentitySecurityPosture;
}

## 119. Posture != Authorization

Una cuenta comprometida puede ser autenticada en modo restringido.
Authorization/policy decide qué acciones siguen disponibles.

## 120. Security Actions

El Security Center deberá exponer acciones tipadas.

## 121. SecurityCenterAction

Ejemplos:

```text
REVOKE_SESSION
REVOKE_OTHER_SESSIONS
LOGOUT_EVERYWHERE

REVOKE_DEVICE
UNTRUST_DEVICE
REPORT_DEVICE_LOST

REMOVE_AUTH_METHOD
REVOKE_AUTH_METHOD
ROTATE_PASSWORD

REGENERATE_RECOVERY_CODES

REVOKE_API_TOKEN
REVOKE_ALL_API_TOKENS

ACKNOWLEDGE_ALERT
REPORT_UNRECOGNIZED_ACTIVITY
```

## 122. Action Availability

No todas las acciones estarán disponibles siempre.

## 123. SecurityActionAvailabilityResolver

interface SecurityActionAvailabilityResolverInterface
{
public function resolve(
SecurityCenterAction $action,
IdentitySecuritySnapshot $snapshot,
SecurityCenterContext $context
): SecurityActionAvailability;
}

## 124. Reasons

Puede responder:

- AVAILABLE
- REAUTHENTICATION_REQUIRED
- AUTHORIZATION_REQUIRED
- POLICY_FORBIDDEN
- NOT_APPLICABLE

## 125. UI Benefits

La UI puede saber:

- mostrar botón
- deshabilitar botón
- pedir reauthentication
- sin implementar policy.

## 126. But Backend Revalidates

La disponibilidad en el snapshot es informativa.
No es autorización para ejecutar.

## 127. Security Command Bus

Mutaciones podrán enviarse mediante:
Security Management Command

## 128. IdentitySecurityCommandBus

interface IdentitySecurityCommandBusInterface
{
public function dispatch(
IdentitySecurityCommandInterface $command
): IdentitySecurityCommandResult;
}

## 129. Why Command Layer

Permite centralizar:

- authorization
- reauthentication
- risk
- tenant scope
- audit
- events
- idempotency

## 130. No Direct Controller Mutations

No:

```php
Session::where('id', $id)->delete();
desde controller.
```

## 131. Command Pipeline

Security Command
↓
Resolve Actor
↓
Authorization
↓
Authentication Requirement
↓
Risk Evaluation
↓
Scope Validation
↓
Authoritative Mutation
↓
Security Version Update
↓
Audit / Event
↓
Result

## 132. Reauthentication Requirements

Documento 32.

- Acciones potencialmente sensibles:
- remove password
- remove final passkey
- regenerate recovery codes
- logout everywhere
- trust device
- revoke all credentials
- unlink enterprise identity

## 133. Dynamic Requirements

Risk puede aumentar requirement.

## 134. Example

Normal:

```text
revoke old session
→ current session sufficient
```

Risk elevado:

```text
revoke current trusted device
→ fresh passkey required
```

## 135. Authorization

Security Center Self-Service deberá utilizar una autorización explícita.
Conceptualmente:
identity.security.manage_self

## 136. Administrative Authorization

Puede utilizar:

- identity.security.inspect
- identity.security.force_logout
- identity.credentials.revoke
- identity.security.suspend
- en Authorization System.

## 137. Authentication != Administrative Authorization

Ser platform admin autenticado fuertemente no implica automáticamente permiso sobre cualquier tenant.

## 138. Multi-Tenant Security Center

Documento 29.

- Debe distinguir:
- Global Security
- Tenant Security
- Realm Security

## 139. Example

Global Identity:
user-123
puede tener:

- Global Passkey
- Global Sessions
- Tenant A SSO
- Tenant B Session

Tenant B Device Trust

## 140. SecurityCenterScope

final readonly class SecurityCenterScope
{
public function __construct(
public SecurityCenterScopeType $type,
public ?TenantId $tenant,
public ?SecurityRealmId $realm,
) {}
}

## 141. Scope Types

GLOBAL
TENANT
REALM

## 142. Tenant View

Un Tenant Administrator no deberá ver necesariamente:

- sessions in other tenants
- global recovery methods

other tenant SSO bindings

## 143. Visibility Policy

interface SecurityInventoryVisibilityPolicyInterface
{
public function filter(
IdentitySecuritySnapshot $snapshot,
SecurityCenterContext $viewer
): IdentitySecuritySnapshot;
}

## 144. Do not Load then Hide if Avoidable

Preferible aplicar scope en repositories/projections para minimizar exposición.

## 145. Cross-Tenant Administration

Debe utilizar contratos explícitos del documento 29.

## 146. Machine Security Center

Documento 33.
La misma arquitectura podrá extenderse a Service Identities.

## 147. Machine Security Inventory

Podrá mostrar:

- Machine Credentials
- Certificates
- Signing Keys
- Workload Federation
- Recent Token Issuance
- Trust Relationships
- Active Delegations
- Credential Expiry

## 148. Human vs Machine UI

El Core no asumirá que todo Security Center es humano.

## 149. PrincipalSecurityCenter

Podría generalizarse posteriormente:

- interface PrincipalSecurityCenterInterface
- pero para V1 se puede mantener IdentitySecurityCenter.

## 150. Administrative Identity Inspection

Administradores autorizados pueden necesitar ver:

- security state
- active sessions
- credential types
- compromise indicators

## 151. Secret Boundaries

Ni siquiera un administrador deberá poder leer:

- password hashes
- TOTP secrets
- private keys
- actual API tokens
- recovery code plaintext

## 152. Credential Reset vs Read

Admin puede:

- revoke
- reset
- replace

pero no necesariamente:
read secret

## 153. Security Principle

Administrability does not imply secret visibility.

## 1. Session Impersonation

Security Center deberá evitar acciones como:

- "Open session as user"
- como simple management operation.

Impersonation pertenece a un subsistema explícito de delegación/authorization.

## 2. Security Freeze

Puede existir una operación:

- SECURITY_FREEZE
- para incident response.

## 3. IdentitySecurityFreeze

Podría:

- block new sessions
- revoke active sessions
- disable credential changes
- require recovery/admin intervention

## 4. Freeze != Account Delete

La identidad continúa existiendo.

## 5. SecurityFreezeStatus

NONE
TEMPORARY
INCIDENT_RESPONSE
ADMINISTRATIVE

## 6. Freeze Authority

No cualquier usuario/administrator puede aplicarlo.
Authorization determina.

## 7. Self-Freeze

Una aplicación puede permitir:

- Lock my account
- en respuesta a compromise.

## 8. Unlock

Deberá requerir recovery/strong Authentication según policy.

## 9. Security Actions after Compromise

Ejemplo integral:

```text
User sees unknown login
        ↓
"This wasn't me"
        ↓
Revoke Session
        ↓
Revoke Device Trust
        ↓
```

Increment Identity Security Epoch
↓
Require Credential Review
↓
Security Center → RESTRICTED

## 10. Credential Review Mode

Puede exigir al usuario revisar:

- password
- passkeys
- MFA
- federated accounts
- recovery methods
- API tokens

## 11. Review Completion

Solo después:

```text
Security Posture
RESTRICTED → NORMAL
```

si policy lo permite.

## 12. Security Checklist

Security Center podrá generar:

- Review Password
- Review Passkeys
- Review Devices
- Review Sessions
- Review Recovery

## 13. Workflow State

Podrá existir:
IdentitySecurityReview

## 14. IdentitySecurityReview

final readonly class IdentitySecurityReview
{
public function __construct(
public string $id,
public IdentityReference $identity,
public array $requiredSteps,
public DateTimeImmutable $createdAt,
) {}
}

## 15. Security Review != Authentication Flow

Puede utilizar Authentication primitives, pero es workflow de gestión.

## 16. Recovery Integration

Recovery puede terminar en Security Review obligatoria.

## 17. Example

Account Recovered
↓
Restricted Authentication
↓
Security Center Review
↓
Remove Unknown Passkey
↓
Regenerate Recovery Codes
↓
Confirm Trusted Devices
↓
Restore Normal State

## 18. Credential Age

Security Center podrá mostrar edad de:

- password
- API token
- certificate
- recovery material
- cuando sea útil.

## 19. Password Age

No deberá interpretarse automáticamente como riesgo.
Policy decide.

## 20. Token Age

Puede generar recommendation si credential permanente es antigua.

## 21. Certificate Expiry

Puede generar:

- ACTION_REQUIRED
- para machine identities.

## 22. Last Used Data

Puede ayudar a detectar:

- unused token
- stale passkey
- old device

## 23. Privacy

Last-used/device metadata puede ser sensible.
Debe someterse a visibility policy.

## 24. Data Retention

Security Center no deberá conservar indefinidamente activity por sí mismo.
Audit/retention subsystem define lifecycle.

## 25. Activity Window

Puede mostrar:

- last 30 days
- last 90 days
- según policy.

## 26. Historical Sessions

No deben confundirse con activas.

## 27. Session States

ACTIVE
EXPIRED
REVOKED
TERMINATED
COMPROMISED

## 28. Session History

Security Center podrá mostrar un subconjunto de sesiones históricas si policy lo permite.

## 29. Session Device Correlation

Puede correlacionar:

```text
Session → Device
si el Device system tiene confianza suficiente.
```

No inferir relaciones fuertes únicamente por User-Agent/IP.

## 30. Unknown Device

Debe poder representarse:

- Unknown Device
- sin inventar identidad.

## 31. Device Naming

User labels no deben modificar identity del dispositivo.

## 32. Credential Naming

Igual para:

- API token "Production"
- Passkey "Laptop"

El nombre es presentation metadata.

## 33. Inventory Pagination

Para usuarios normales puede haber pocos registros.
Enterprise/machine identities podrían tener muchos.

- Debe soportar:
- pagination
- cursoring
- filtering

## 34. Streaming

No necesario para Self-Service normal, pero APIs administrativas pueden usarlo.

## 35. Inventory Query API

interface SecurityInventoryQueryInterface
{
public function sessions(...): SessionInventoryPage;

public function devices(...): DeviceInventoryPage;

public function credentials(...): CredentialInventoryPage;

public function activity(...): SecurityActivityPage;
}

## 189. Query vs Command

Separación recomendada:

```text
Query Side
→ Inventory / Snapshot

Command Side
→ Revocation / Mutation
```

## 190. CQRS

No requiere CQRS completo, pero el patrón resulta útil.

## 191. Read Models

Security Center puede usar proyecciones optimizadas.

## 192. Authoritative Mutation

Commands siempre vuelven a stores propietarios.

## 193. Event-Driven Projection

Opcional:

```text
Authentication Events
      ↓
Security Center Projection
```

## 194. Eventual Consistency

Inventory puede ser ligeramente eventual.
Pero las acciones de seguridad no confiarán en él como authority.

## 195. Example

Snapshot muestra token activo.
Token ya fue revocado.
Usuario presiona Revoke.

- Resultado:
- ALREADY_REVOKED
- no error peligroso.

## 196. Idempotency

Muchas acciones deben ser idempotentes.

## 197. Examples

revoke session
revoke token
untrust device
acknowledge alert

## 198. Revocation Idempotency

Repetir una revocación deberá producir resultado seguro.

## 199. Distributed Runtime

Documento 30.

- Security Center debe funcionar con:
- Node A snapshot
- Node B mutation

Node C next request

## 200. Cluster Session View

Session inventory deberá consultar una proyección/store distribuido adecuado.

## 201. L1 Caches

Podrán utilizarse para lectura.
No deberán impedir revocation inmediata lógica.

## 202. Security Version

Cambios deberán actualizar:

- SessionVersion
- IdentitySecurityVersion
- DeviceSecurityVersion
- CredentialVersion
- según operación.

## 203. Global Logout

Debe utilizar mecanismos del documento 30.

## 204. Multi-Region

Security Center puede mostrar:

- revocation pending propagation
- solo si realmente la arquitectura tiene ese concepto.

Preferible que logical revocation ya sea efectiva según SLA.

## 205. Security Consistency

Para comandos críticos:

- STRONG / MONOTONIC
- según estado.

Inventories:

- EVENTUAL_ACCEPTABLE
- en muchos casos.

## 206. FrankenPHP

SecurityCenterSnapshot no deberá almacenarse en singleton mutable.

## 207. Never

static $currentSecurityCenter;

## 208. Request-Scoped View

Cada request genera el snapshot correspondiente.

## 209. Cache Keys

Si se cachea:

- identity
- tenant
- realm
- security version
- viewer scope

deberán formar parte de la key.

## 210. Viewer-Aware Cache

Muy importante.
No reutilizar snapshot administrativo para user self-service.

## 211. Cache Contamination

Ejemplo peligroso:

```text
Admin View
    includes global sessions
```

cached by IdentityId only

User View
receives global admin data
Debe ser imposible.

## 212. SecurityCenterCacheKey

Conceptualmente:

- Identity
- Viewer Authority Scope
- Tenant
- Realm
- Projection Version

## 213. Cache Bounds

FrankenPHP local cache:

- bounded
- TTL
- version aware

## 214. Runtime Reset

Limpiar:

- current snapshot
- current security command context
- current actor
- temporary proof

## 215. Fiber Safety

Dos Security Center requests concurrentes no deben compartir actor/context.

## 216. Events

Documento 23.

```text
Eventos del subsistema:
SecurityCenterViewed

SessionRevocationRequested
SessionRevokedFromSecurityCenter

DeviceRevocationRequested
DeviceRevokedFromSecurityCenter

IdentityGlobalLogoutRequested
IdentityGlobalLogoutCompleted

SecurityAlertAcknowledged
SecurityAlertResolved

IdentityCompromiseReported
IdentitySecurityReviewStarted
IdentitySecurityReviewCompleted

IdentitySecurityFreezeApplied
IdentitySecurityFreezeReleased
```

## 217. Avoid View Audit Noise

SecurityCenterViewed puede ser:

- optional/audit profile dependent
- para evitar exceso de eventos.

## 218. Mutation Events

Sí deberán ser auditables.

## 219. Audit

Cada acción deberá registrar:

- Actor
- Target Identity
- Tenant
- Realm
- Action
- Artifact Type
- Artifact Management ID
- Reason
- Reauthentication
- Risk
- Outcome
- Timestamp

## 220. Management IDs

Audit puede registrar IDs internos seguros.
Nunca secretos.

## 221. Security Center Reads

Administradores leyendo credenciales sensibles pueden requerir audit.

## 222. Example

Administrator inspected security state of executive account
puede ser relevante.

## 223. Audit Visibility

El propio usuario no necesariamente verá todo audit administrativo.
SecurityActivityProjection decide.

## 224. Notifications

Las siguientes acciones pueden notificar:

- new security method
- session revoked
- device revoked
- password changed
- global logout
- security freeze
- recovery method changed

## 225. User Notification Preferences

Algunas security notifications no deberían ser disableable.

## 226. Critical Notifications

Ejemplos:

- password changed
- MFA disabled
- new passkey added
- recovery used

podrán ignorar preferences normales.

## 227. Notification Channel Failure

No deberá revertir la operación de seguridad ya completada.

## 228. Audit Before Notification

Security mutation deberá persistirse aunque notification falle.

## 229. Observability

Metrics:

```text
auth_security_center_snapshot_total

auth_session_revoke_self_total
auth_session_revoke_other_total
auth_global_logout_total

auth_device_revoke_total
auth_device_untrust_total

auth_security_alert_ack_total
auth_security_compromise_report_total
auth_security_review_total
```

## 230. Metric Labels

action
artifact_type
realm
outcome
con cardinalidad controlada.

## 231. No Identity IDs

No usar como metric label.

## 232. Traces

Spans:

- auth.security_center.snapshot
- auth.security_center.revoke_session
- auth.security_center.revoke_device
- auth.security_center.logout_everywhere
- auth.security_center.compromise_response

## 233. Sensitive Trace Data

Nunca incluir tokens, secrets o session credentials.

## 234. Failure Taxonomy

Documento 25.

```text
SECURITY_CENTER_ACCESS_DENIED

SECURITY_ARTIFACT_NOT_FOUND
SECURITY_ARTIFACT_NOT_OWNED
SECURITY_ARTIFACT_SCOPE_MISMATCH

SESSION_MANAGEMENT_REFERENCE_INVALID
SESSION_ALREADY_REVOKED

DEVICE_MANAGEMENT_REFERENCE_INVALID
DEVICE_ALREADY_REVOKED

CREDENTIAL_ALREADY_REVOKED

SECURITY_ACTION_REAUTHENTICATION_REQUIRED
SECURITY_ACTION_ASSURANCE_INSUFFICIENT

SECURITY_ALERT_NOT_FOUND
SECURITY_ALERT_ALREADY_RESOLVED

SECURITY_REVIEW_REQUIRED
SECURITY_FREEZE_ACTIVE

SECURITY_CENTER_SNAPSHOT_UNAVAILABLE
```

## 235. Public Error Normalization

No revelar:
session exists but belongs to another user

## 236. Internal Reason

Audit puede registrar:
ARTIFACT_OWNERSHIP_MISMATCH

## 237. Security Center Controller

Controller deberá ser delgado.

## 238. Incorrecto

public function destroySession($id)
{
DB::table('sessions')->where('id', $id)->delete();
}

## 239. Correcto conceptualmente

$this->securityCommands->dispatch(
new RevokeSessionCommand(...)
);

## 240. SPA Protocol

VoltStack deberá poder exponer JSON seguro.

## 241. Example Snapshot

{
"summary": {
"mfa": true,
"passkeys": 2,
"active_sessions": 3,
"trusted_devices": 2,
"alerts": 1
},
"sessions": [],
"devices": [],
"methods": []
}

## 242. No Internal Policy Dumps

No enviar todas las policies internas al frontend.

## 243. Action Capabilities

Cada item podrá incluir:

- can_revoke
- can_rename
- can_untrust
- requires_reauthentication
- como hints.

## 244. Backend Rechecks

Siempre.

## 245. Sensitive Action Protocol

Ejemplo:

```php
DELETE /security/sessions/{management-id}
    ↓
Reauth Required?
    ↓
challenge
    ↓
proof
    ↓
retry
```

integrado con documento 32.

## 246. Bulk Actions

Podrán existir:

- Revoke Selected Sessions
- Revoke All API Tokens

Remove All Trusted Devices

## 247. Bulk Mutation Atomicity

No siempre posible hacer todo en una única DB transaction distribuida.
Debe producir:

- per-item outcome
- o utilizar logical epoch donde convenga.

## 248. Global Operations

Preferir epoch/version para:

- logout everywhere
- revoke all persistent login
- cuando sea apropiado.

## 249. Bulk Operation ID

SecurityOperationId
permitirá tracing/idempotency.

## 250. Security Operations Journal

Podrá conservar:

- operation
- status
- logical completion
- cleanup completion

## 251. Example

Global Logout:
LOGICALLY_COMPLETE

Cleanup:
IN_PROGRESS

## 252. User-facing Semantics

Puede considerarse completado cuando acceso ya es inválido lógicamente.

## 253. Security Center Extensibility

Documento 28.

- Plugins podrán agregar:
- Inventory Section
- Credential Type
- Security Recommendation
- Security Alert Renderer
- Security Action
- Projection Provider

## 254. Extension Boundary

Plugin no deberá saltarse:

- Authorization
- Reauthentication
- Tenant Isolation
- Audit
- Secret Redaction

## 255. SecurityCenterSectionProvider

interface SecurityCenterSectionProviderInterface
{
public function section(
IdentityReference $identity,
SecurityCenterContext $context
): SecurityCenterSection;
}

## 256. Example Plugin

Enterprise package añade:

- Corporate Smart Cards
- Managed Workstations
- Hardware Certificates

## 257. No Arbitrary HTML in Core

El provider deberá exponer datos/modelos.
Presentation adapters deciden UI.

## 258. Testing — Snapshot

Debe ensamblar correctamente:

- methods
- sessions
- devices
- credentials
- alerts
- activity

## 259. Testing — Secret Redaction

Canary secrets jamás aparecen en snapshot.

## 260. Testing — Session Management ID

No equivale al real Session ID/token.

## 261. Testing — Session Ownership

Identity A no revoca Session B.

## 262. Testing — Current Session

Se identifica correctamente.

## 263. Testing — Revoke Other Session

La otra Session deja de autenticar.

## 264. Testing — Revoke Current Session

Produce logout correcto.

## 265. Testing — Global Logout

Todas las targeted Sessions quedan lógicamente inválidas.

## 266. Testing — Remember-Me

Logout-all revoca persistent credentials si policy lo exige.

## 267. Testing — Device Revocation

Cascade correcto.

## 268. Testing — Lost Device

Sessions/credentials/trust afectados correctamente.

## 269. Testing — Device Ownership

No se modifica device de otra identidad.

## 270. Testing — API Token Inventory

No devuelve token secreto.

## 271. Testing — Show Once Secret

Solo creación devuelve secret.

## 272. Testing — Re-view

No puede recuperarse plaintext posteriormente.

## 273. Testing — Recovery Inventory

No revela recovery codes.

## 274. Testing — Security Activity

Solo muestra eventos visibles al principal.

## 275. Testing — Admin Activity

Visibility policy correcta.

## 276. Testing — Security Alert

Acknowledgment no borra evidence.

## 277. Testing — "Wasn't Me"

Dispara compromise response configurada.

## 278. Testing — Security Review

Recovery puede requerir review.

## 279. Testing — Security Freeze

Impide acciones configuradas.

## 280. Testing — Unlock

Exige requirements correctos.

## 281. Testing — Tenant Scope

Tenant Admin A no ve inventario de Tenant B.

## 282. Testing — Global Identity

Global user ve solo secciones permitidas por scope.

## 283. Testing — Viewer Cache Separation

Snapshot administrativo jamás se reutiliza para self-service.

## 284. Testing — Stale Snapshot

Mutation vuelve a validar authority.

## 285. Testing — Idempotent Revocation

Repeated revoke remains safe.

## 286. Testing — Distributed Global Logout

Node A revoca y Node B rechaza Session.

## 287. Testing — Multi-Node Snapshot

Inventario consistente con projection policy.

## 288. Testing — Event Loss

Revocation correctness no depende del Security Center projection.

## 289. Testing — Reauthentication

Sensitive action rechaza proof insuficiente.

## 290. Testing — Risk

Risk dinámico puede endurecer una acción.

## 291. Testing — Authorization

Strong Authentication no permite gestionar Identity ajena sin Authorization.

## 292. Testing — Admin Secret Visibility

Admin no obtiene secrets.

## 293. Testing — FrankenPHP

Snapshot de User A no aparece en User B.

## 294. Testing — Fibers

Dos Security Centers concurrentes mantienen actor/scope aislados.

## 295. Testing — Cache Bounds

No crece indefinidamente.

## 296. Testing — Extension

Custom section no puede filtrar secrets.

## 297. Testing — Machine Identity

Security inventory de service principal funciona con machine credentials.

## 298. Testing — Bulk Revocation

Per-item/epoch semantics correctas.

## 299. Security Invariants — Snapshot

AUTH-SEC-CENTER-SNAP-01
El Security Center es una proyección, no la autoridad de Authentication State.
AUTH-SEC-CENTER-SNAP-02
Snapshots son secret-free.

- AUTH-SEC-CENTER-SNAP-03
- Una acción de seguridad vuelve a consultar estado autoritativo.
- AUTH-SEC-CENTER-SNAP-04

Snapshots están scoped por Identity, Viewer, Tenant y Realm.
AUTH-SEC-CENTER-SNAP-05
Un snapshot administrativo no puede reutilizarse para un contexto con menor autoridad.

## 300. Security Invariants — Sessions

AUTH-SEC-CENTER-SESSION-01
Management IDs nunca son Session credentials.

- AUTH-SEC-CENTER-SESSION-02
- Una Identity no puede revocar Sessions ajenas mediante self-service.
- AUTH-SEC-CENTER-SESSION-03

Global logout utiliza revocación distribuida autoritativa.

- AUTH-SEC-CENTER-SESSION-04
- Revocación repetida es segura e idempotente.
- AUTH-SEC-CENTER-SESSION-05

Remember-Me credentials se consideran independientemente de Sessions activas.

## 301. Security Invariants — Devices

AUTH-SEC-CENTER-DEVICE-01
Device y Session son conceptos independientes.

- AUTH-SEC-CENTER-DEVICE-02
- Device y Passkey son conceptos independientes.
- AUTH-SEC-CENTER-DEVICE-03

Confiar un dispositivo puede requerir fresh Authentication.

- AUTH-SEC-CENTER-DEVICE-04
- Revocación de Device sigue una cascade policy explícita.
- AUTH-SEC-CENTER-DEVICE-05

Device Management IDs no contienen material autenticador.

## 302. Security Invariants — Credentials

AUTH-SEC-CENTER-CRED-01
Credentials secrets jamás aparecen en inventario.
AUTH-SEC-CENTER-CRED-02
Show-once credentials no pueden recuperarse posteriormente desde Security Center.
AUTH-SEC-CENTER-CRED-03
Revocación de Credential delega al subsistema propietario.
AUTH-SEC-CENTER-CRED-04
Administrative visibility nunca implica secret visibility.

## 303. Security Invariants — Activity

AUTH-SEC-CENTER-ACT-01
User Security Activity es una proyección, no el Audit Log interno completo.
AUTH-SEC-CENTER-ACT-02
Activity Visibility es viewer-aware.

- AUTH-SEC-CENTER-ACT-03
- Security Alerts no reemplazan Risk Signals.
- AUTH-SEC-CENTER-ACT-04

Acknowledge/resolve no elimina audit evidence.

## 304. Security Invariants — Commands

AUTH-SEC-CENTER-CMD-01
Toda mutación pasa por un Security Management Command o servicio equivalente.
AUTH-SEC-CENTER-CMD-02
Commands revalidan Authorization y Authentication Requirements.
AUTH-SEC-CENTER-CMD-03
Commands preservan Tenant/Realm scope.

- AUTH-SEC-CENTER-CMD-04
- Commands críticos generan Audit/Event apropiados.
- AUTH-SEC-CENTER-CMD-05

La UI nunca es enforcement authority.

## 305. Security Invariants — Compromise

AUTH-SEC-CENTER-COMP-01
Reportar actividad desconocida puede iniciar respuesta a compromiso.
AUTH-SEC-CENTER-COMP-02
Compromise response utiliza políticas explícitas.

- AUTH-SEC-CENTER-COMP-03
- Security Freeze no elimina Identity.
- AUTH-SEC-CENTER-COMP-04

Recovery puede exigir Security Review antes de volver a estado normal.

## 306. Security Invariants — Multi-Tenant

AUTH-SEC-CENTER-TENANT-01
Tenant administrators solo ven artefactos dentro de su authority scope.
AUTH-SEC-CENTER-TENANT-02
Global Identity state y Tenant-specific state permanecen distinguibles.
AUTH-SEC-CENTER-TENANT-03
Cross-Tenant security administration utiliza contratos privilegiados explícitos.
AUTH-SEC-CENTER-TENANT-04
Cache keys incluyen scope suficiente para impedir cross-tenant contamination.

## 307. Security Invariants — Distributed Runtime

AUTH-SEC-CENTER-DIST-01
Snapshot eventual nunca impide una revocación autoritativa.

- AUTH-SEC-CENTER-DIST-02
- Logical global logout no depende de cleanup físico inmediato.
- AUTH-SEC-CENTER-DIST-03

Security version changes se propagan según documento 30.
AUTH-SEC-CENTER-DIST-04
Node-local caches nunca se consideran autoridad de revocación.

## 308. Security Invariants — FrankenPHP

AUTH-SEC-CENTER-RT-01
Security Center context nunca vive en mutable global state.

- AUTH-SEC-CENTER-RT-02
- Snapshots request-local se limpian entre requests.
- AUTH-SEC-CENTER-RT-03

Fiber concurrency no comparte viewer, target identity ni security proof.
AUTH-SEC-CENTER-RT-04
Caches persistentes son bounded, scoped y version-aware.

## 309. Anti-Pattern

Nunca:

```text
Security Center DB table
=
```

source of truth for every Authentication subsystem

## 310. Anti-Pattern

list raw sessions
including real session token

## 311. Anti-Pattern

admin can view user's TOTP secret

## 312. Anti-Pattern

admin can display user's API token again

## 313. Anti-Pattern

User-Agent
=

trusted device identity

## 314. Anti-Pattern

same IP
=

same device

## 315. Anti-Pattern

delete session database row
=

complete global logout

## 316. Anti-Pattern

zero active sessions
=

no remaining persistent login credentials

## 317. Anti-Pattern

UI hides button
=

security enforcement

## 318. Anti-Pattern

snapshot says session active
=

session definitely still active

## 319. Anti-Pattern

snapshot says revoke allowed
=

backend skips Authorization

## 320. Anti-Pattern

security score = 100
=

account cannot be compromised

## 321. Anti-Pattern

administrator sees identity
=

administrator can see secrets

## 322. Anti-Pattern

security activity
=

raw audit log

## 323. Anti-Pattern

"This wasn't me"
=

delete activity record

## 324. Anti-Pattern

recovered account
=

all security restrictions removed immediately

## 325. Anti-Pattern

Security Center snapshot cached only by identity ID
sin considerar viewer/tenant/realm.

## 326. Anti-Pattern

static $currentSecuritySnapshot;
bajo FrankenPHP.

## 327. Componentes principales

IdentitySecurityCenter
IdentitySecuritySnapshot
IdentitySecuritySummary
IdentitySecurityPosture

AuthenticationMethodInventory
SessionInventory
DeviceInventory
CredentialInventory
SecurityActivityInventory

SecurityCenterSession
SecurityCenterDevice
SecurityCenterCredential
SecurityActivityEntry

SecurityAlert
SecurityRecommendation

IdentitySecurityCommandBus
SecurityActionAvailabilityResolver

IdentityCompromiseResponse
IdentitySecurityReview
IdentitySecurityFreeze

## 328. Session Components

SessionManagementId
SecurityCenterSession
SessionRevocationCommand
SessionRevocationReason
GlobalLogoutCommand
LogoutOtherSessionsCommand
PersistentLoginInventory

## 329. Device Components

DeviceManagementId
SecurityCenterDevice
DeviceRevocationCommand
LostDeviceCommand
DeviceRevocationCascadePolicy
DeviceRevocationCascadePlan

## 330. Credential Components

CredentialManagementId
SecurityCenterCredential
CredentialInventory
CredentialRevocationCommand
ShowOnceSecret
CredentialSecurityStatus

## 331. Activity Components

SecurityActivityEntry
SecurityActivitySeverity
SecurityActivityProjection
SecurityActivityVisibilityPolicy
SecurityAlert
SecurityAlertStatus
SecurityRecommendation

## 332. Incident Components

IdentityCompromiseResponseCommand
IdentitySecurityReview
IdentitySecurityFreeze
IdentitySecurityPostureResolver

## 333. Namespace sugerido

VoltStack\Quantum\Auth\SecurityCenter
VoltStack\Quantum\Auth\SecurityCenter\Contracts
VoltStack\Quantum\Auth\SecurityCenter\Snapshot
VoltStack\Quantum\Auth\SecurityCenter\Session
VoltStack\Quantum\Auth\SecurityCenter\Device
VoltStack\Quantum\Auth\SecurityCenter\Credential
VoltStack\Quantum\Auth\SecurityCenter\Activity
VoltStack\Quantum\Auth\SecurityCenter\Alert
VoltStack\Quantum\Auth\SecurityCenter\Recommendation
VoltStack\Quantum\Auth\SecurityCenter\Command
VoltStack\Quantum\Auth\SecurityCenter\Incident
VoltStack\Quantum\Auth\SecurityCenter\Runtime

## 334. Estructura sugerida

src/Quantum/Auth/SecurityCenter/
├── Contracts/
│   ├── IdentitySecurityCenterInterface.php
│   ├── SecurityInventoryQueryInterface.php
│   ├── IdentitySecurityCommandBusInterface.php
│   ├── SecurityActionAvailabilityResolverInterface.php
│   ├── SecurityInventoryVisibilityPolicyInterface.php
│   ├── SecurityActivityVisibilityPolicyInterface.php
│   ├── DeviceRevocationCascadePolicyInterface.php
│   └── IdentitySecurityPostureResolverInterface.php
│
├── Snapshot/
│   ├── IdentitySecuritySnapshot.php
│   ├── IdentitySecuritySummary.php
│   ├── IdentitySecurityPosture.php
│   ├── SecurityCenterContext.php
│   └── SecurityCenterScope.php
│
├── Session/
│   ├── SessionInventory.php
│   ├── SecurityCenterSession.php
│   ├── SessionManagementId.php
│   ├── SessionClientDescriptor.php
│   ├── SessionRevocationCommand.php
│   ├── GlobalLogoutCommand.php
│   └── LogoutOtherSessionsCommand.php
│
├── Device/
│   ├── DeviceInventory.php
│   ├── SecurityCenterDevice.php
│   ├── DeviceManagementId.php
│   ├── DeviceRevocationCommand.php
│   ├── LostDeviceCommand.php
│   └── DeviceRevocationCascadePlan.php
│
├── Credential/
│   ├── CredentialInventory.php
│   ├── SecurityCenterCredential.php
│   ├── CredentialManagementId.php
│   ├── CredentialSecurityStatus.php
│   ├── CredentialRevocationCommand.php
│   └── ShowOnceSecret.php
│
├── Activity/
│   ├── SecurityActivityInventory.php
│   ├── SecurityActivityEntry.php
│   ├── SecurityActivitySeverity.php
│   ├── SecurityActivityProjection.php
│   └── SecurityActivityClient.php
│
├── Alert/
│   ├── SecurityAlert.php
│   ├── SecurityAlertStatus.php
│   ├── SecurityAlertSeverity.php
│   └── SecurityAlertType.php
│
├── Recommendation/
│   ├── SecurityRecommendation.php
│   ├── SecurityRecommendationSet.php
│   └── SecurityRecommendationPriority.php
│
├── Command/
│   ├── IdentitySecurityCommand.php
│   ├── IdentitySecurityCommandBus.php
│   ├── IdentitySecurityCommandResult.php
│   └── SecurityOperationId.php
│
├── Incident/
│   ├── IdentityCompromiseResponseCommand.php
│   ├── IdentitySecurityReview.php
│   ├── IdentitySecurityFreeze.php
│   └── IdentitySecurityPostureResolver.php
│
└── Runtime/
├── SecurityCenterRuntimeContext.php
├── SecurityCenterSnapshotCache.php
└── SecurityCenterRuntimeResetter.php

## 335. Configuración conceptual

return [

'authentication' => [

'security_center' => [

'enabled' => true,

'activity' => [
'enabled' => true,
'retention_view' => '90 days',
],

'sessions' => [
'allow_revoke_current' => true,
'allow_logout_other_sessions' => true,
'allow_logout_everywhere' => true,
],

'devices' => [
'allow_trust_management' => true,
],

'recommendations' => [
'enabled' => true,
],

],

],

];

## 336. Snapshot conceptual

$snapshot = Auth::security()
->for($identity)
->snapshot();
Podría exponer:

- summary
- methods
- sessions
- devices
- credentials
- activity
- alerts
- recommendations

## 337. Developer Experience

Consultar:
$security = Auth::security()->current();
Revocar una sesión:

```php
Auth::security()
    ->sessions()
    ->revoke($sessionManagementId);
```

Cerrar otras:

```php
Auth::security()
    ->sessions()
    ->logoutOthers();
```

Cerrar todo:

```php
Auth::security()
    ->sessions()
    ->logoutEverywhere();
```

Reportar un dispositivo:

```php
Auth::security()
    ->devices()
    ->reportLost($deviceManagementId);
```

La API exacta podrá evolucionar.

## 338. Flujo Security Center

USER REQUEST
│
▼
Resolve Identity / Viewer
│
▼
Resolve Tenant / Realm
│
▼
Authorization
│
▼
Security Center Query
│
┌───┼──────────┬───────────┬───────────┐
▼   ▼          ▼           ▼           ▼
Methods Sessions Devices Credentials Activity
│   │          │           │           │
└───┴──────────┴─────┬─────┴───────────┘
▼
Security Snapshot
│
▼
Visibility Filtering
│
▼
UI / API

## 339. Flujo de una acción

User selects:

```text
"Revoke Session"
        │
        ▼
Security Command
        │
        ▼
Resolve Target Session
        │
        ▼
```

Verify Ownership / Scope
│
▼
Authorization
│
▼
Reauthentication Requirement
│
▼
Risk
│
▼
Authoritative Revocation
│
▼
Security Version Update
│
▼
Audit / Event
│
▼
Updated Snapshot

## 340. Flujo "Sign Out Everywhere"

Current Identity
│
▼
Global Logout Request
│
▼
Fresh Authentication
│
▼
Identity Security Epoch++
│
▼
All old Sessions invalid
│
├── Remember-Me revoke
├── Flow invalidation
└── Cache purge
│
▼
Async Cleanup

## 341. Flujo Lost Device

Unknown / Lost Device
│
▼
Report Lost
│
▼
Device Security State = LOST
│
├── Trust revoked
├── Device credential revoked
├── Remember-Me revoked
└── Sessions revoked
│
▼
Risk Signal
│
▼
Audit / Notification

## 342. Flujo de compromiso

Security Activity:

```text
"Login from unknown device"
        │
        ▼
```

User:

```php
"This wasn't me"
        │
        ▼
Compromise Response Policy
        │
        ├── Revoke Session
        ├── Revoke Device
        ├── Increase Security Epoch
        ├── Restrict Identity
        ├── Require Credential Review
        └── Notify Security
        │
        ▼
Security Review
        │
        ▼
Return to Normal State
```

## 343. Flujo Recovery → Security Review

Account Recovery
│
▼
Restricted Authentication
│
▼
Security Center
│
├── Review Password
├── Review Passkeys
├── Review MFA
├── Review Sessions
├── Review Devices
└── Review Recovery
│
▼
Requirements Completed
│
▼
Normal Authentication State

## 344. Relación con Laravel

Laravel ofrece piezas muy útiles para este dominio mediante:

- Auth
- Sessions
- Fortify
- Sanctum
- Personal Access Tokens
- Password Confirmation
- Authentication Events

Jetstream, por ejemplo, muestra cómo una experiencia de usuario puede incluir:

- browser sessions
- password management
- two-factor authentication

VoltStack deberá conservar esa simplicidad, pero llevarla al Core como arquitectura reusable y no como una UI concreta.

## 345. Relación con Symfony

Symfony aporta primitives de:

- Security
- Sessions
- Authenticators
- User Providers
- Events
- Cache
- Messenger

sobre las cuales puede construirse administración avanzada.

- VoltStack deberá aportar además una capa de dominio explícita para:
- Session Inventory
- Device Inventory
- Credential Inventory
- Security Activity
- Security Actions
- Compromise Response

## 346. Diferenciador VoltStack

VoltStack combinará:

- Laravel-like Security Management DX
- +
- Symfony-like Contracts
- +
- Unified Security Inventory
- +
- Session Management
- +
- Device Management
- +
- Credential Management
- +
- Authentication Method Management
- +
- Security Activity
- +
- Security Alerts
- +
- Compromise Response
- +
- Security Review
- +
- Multi-Tenant Visibility
- +
- Distributed Revocation
- +
- FrankenPHP-safe Runtime

## 347. Decisiones arquitectónicas principales

VoltStack adoptará:

## 01. Security Center será una proyección, no una fuente de verdad.

## 1. Cada subsistema seguirá siendo owner de su estado.

## 2. Snapshots nunca contendrán secrets.

## 3. Session Management IDs serán distintos de Session Credentials.

## 4. Device Management IDs serán distintos de Device Credentials.

## 5. Security Center distinguirá Sessions, Devices, Authentication Methods y Credentials.

## 6. Remember-Me Credentials serán visibles conceptualmente aunque no existan Sessions activas.

## 7. Revocaciones siempre volverán al store autoritativo.

## 8. La UI nunca será enforcement authority.

## 9. Security Center Commands volverán a validar Authorization.

## 10. Reauthentication será aplicada según sensibilidad de cada acción.

## 11. Risk podrá endurecer dinámicamente requisitos.

## 12. Global Logout será una operación first-class.

## 13. Logical revocation precederá physical cleanup cuando corresponda.

## 14. Lost Device podrá coordinar una cascade de revocación.

## 15. Show-once credentials nunca podrán recuperarse posteriormente.

## 16. Administrative visibility no implicará secret visibility.

## 17. Security Activity será una proyección segura del Audit/Event system.

## 18. Security Alerts y Risk Signals serán conceptos distintos.

## 19. "This wasn't me" será una operación formal de compromise response.

## 20. Recovery podrá terminar en Security Review obligatorio.

## 21. Security Freeze será distinto de Identity deletion.

## 22. Security Posture se representará sin prometer seguridad absoluta.

## 23. Typed recommendations serán preferibles a un único security score.

## 24. Multi-Tenant visibility será explícita.

## 25. Admin Security Center y Self-Service Security Center tendrán authority scopes diferentes.

## 26. Machine identities podrán usar la misma arquitectura de inventario cuando corresponda.

## 27. Security Center read models podrán ser eventually consistent.

## 28. Security mutations requerirán consistency apropiada al estado.

## 29. Snapshots cacheados incluirán viewer/scope/version.

## 30. Administrative snapshots nunca contaminarán self-service caches.

## 31. Commands importantes serán idempotentes.

## 32. Bulk operations tendrán Operation IDs.

## 33. Audit será obligatorio para security mutations.

## 34. Notifications no serán requisito para que una revocación ya efectiva siga siendo válida.

## 35. Plugins podrán añadir secciones mediante contratos.

## 36. Plugins no podrán saltarse redaction, Authorization o Tenant isolation.

## 37. Security Center state será request/fiber scoped.

## 38. FrankenPHP nunca conservará snapshot del usuario anterior.

39. El diseño será UI-agnostic.
40. Criterios de aceptación

El subsistema será considerado completo cuando soporte:
41. IdentitySecurityCenter;
42. IdentitySecuritySnapshot;
43. Security Summary;
44. Security Posture;
45. Authentication Method Inventory;
46. Session Inventory;
47. Current Session identification;
48. Session Management IDs;
49. Session revocation;
50. logout current session;
51. logout other sessions;
52. logout everywhere;
53. Remember-Me Inventory;
54. persistent login revocation;
55. Device Inventory;
56. Trusted Device management;
57. Device Management IDs;
58. Lost Device workflow;
59. Device revocation cascade;
60. Credential Inventory;
61. API Credential inventory;
62. credential revocation;
63. show-once secret semantics;
64. Recovery Inventory;
65. Security Activity projection;
66. Security Activity visibility policy;
67. Security Alerts;
68. Alert acknowledgment;
69. compromise reporting;
70. "This wasn't me";
71. compromise response policies;
72. Security Recommendations;
73. Security Review workflows;
74. Recovery → Review workflow;
75. Security Freeze;
76. self-freeze if enabled;
77. administrative freeze;
78. Action Availability resolver;
79. Security Management Command Bus;
80. Authorization integration;
81. Reauthentication integration;
82. Risk integration;
83. Audit integration;
84. Events;
85. Notifications integration;
86. Multi-Tenant scopes;
87. tenant admin views;
88. platform admin views;
89. machine identity inventories;
90. read models/projections;
91. pagination;
92. idempotent mutations;
93. bulk operations;
94. distributed revocation;
95. cache versioning;
96. viewer-aware cache isolation;
97. secret redaction;
98. observability;
99. FrankenPHP reset;
100. fiber-safe execution.
101. Regla arquitectónica final

VoltStack deberá entender el estado de seguridad de una identidad así:

```text
                              IDENTITY
                                 │
                                 ▼
                       IDENTITY SECURITY
                                 │
        ┌───────────────┬────────┼────────┬───────────────┐
        ▼               ▼        ▼        ▼               ▼
   AUTH METHODS      SESSIONS   DEVICES CREDENTIALS   RECOVERY
        │               │        │        │               │
        └───────────────┴────────┼────────┴───────────────┘
                                 ▼
                       SECURITY ACTIVITY
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
                ALERTS                RECOMMENDATIONS
                    │                         │
                    └────────────┬────────────┘
                                 ▼
                         SECURITY ACTIONS
                                 │
                                 ▼
               AUTHORIZATION + REAUTH + RISK
                                 │
                                 ▼
                       AUTHORITATIVE STATE
                                 │
                                 ▼
                         AUDIT / EVENTS
```

La primera regla será:
El Security Center no poseerá duplicados de Sessions, Devices ni Credentials; compondrá una vista coherente sobre los subsistemas propietarios.

La segunda:
Ningún inventario o snapshot será suficiente para autorizar una mutación; toda acción volverá a validar el estado real, la identidad, el scope y la política actual.

La tercera:
Un identificador utilizado para administrar una sesión, dispositivo o credential nunca deberá ser el mismo secreto que permite autenticarse con ese artefacto.

La cuarta:
El hecho de que un administrador pueda inspeccionar o revocar una credencial nunca significará que pueda leer su material secreto.

La quinta:
Cerrar sesiones, revocar Remember-Me, eliminar confianza de dispositivos y revocar credentials serán operaciones distintas aunque puedan combinarse dentro de workflows como logout everywhere, lost device o compromise response.

La sexta:
El Security Center deberá convertir actividad sospechosa en acciones concretas y seguras, no limitarse a mostrar información.

La séptima:
Una cuenta recuperada o comprometida podrá entrar temporalmente en un estado restringido y requerir revisión explícita de sus mecanismos de Authentication antes de recuperar su postura normal.

La octava:
Toda visión del estado de seguridad será scoped por Identity, Viewer, Tenant y Realm; nunca existirá un snapshot universal reutilizable indiscriminadamente.

La novena:
VoltStack podrá ofrecer la misma arquitectura de administración a usuarios humanos, administradores y Machine Identities sin confundir sus políticas ni tipos de credenciales.

La décima:
Bajo FrankenPHP, clusters y múltiples regiones, el Security Center podrá ser eventualmente consistente como interfaz de lectura, pero la revocación y las decisiones de seguridad continuarán dependiendo del estado autoritativo y de las garantías distribuidas definidas por Authentication.

Siguiente documento recomendado
La siguiente pieza natural del sistema sería:
`36_AUTHENTICATION_POLICY_ENGINE_REQUIREMENT_COMPOSITION_SECURITY_POSTURE_AND_AUTHENTICATION_GOVERNANCE_SYSTEM.md`
Hasta este punto ya tenemos políticas distribuidas en prácticamente todos los subsistemas:

- Password Policy
- Session Policy
- Remember-Me Policy
- MFA Policy
- Passkey Policy
- Federation Policy
- Recovery Policy
- Risk Policy
- Device Policy
- Tenant Policy
- Privileged Policy
- Machine Authentication Policy
- Authentication Method Policy
- Security Center Policy

El 36 podría consolidar formalmente el motor de gobierno de Authentication para evitar que estas reglas terminen implementadas como configuraciones independientes sin una semántica común.
La arquitectura resultante podría ser:

```text
Framework Security Floor
        │
        ▼
Platform Authentication Policy
        │
        ▼
Realm Policy
        │
        ▼
Tenant Policy
        │
        ▼
```

Identity / Principal Policy
│
▼
Operation Policy
│
▼
Risk-Adaptive Requirements
│
▼
EFFECTIVE AUTHENTICATION REQUIREMENT
Ese documento sería importante para que VoltStack tenga un verdadero Authentication Policy Engine central en vez de una colección de reglas aisladas.
