# VoltStack Authentication System

## 10 — Identity Security State, Account Status and Authentication Eligibility System

- **Archivo:** `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`  
- **Sistema:** Authentication  
- **Framework:** VoltStack  
- **Módulo sugerido:** `Quantum/Auth`  
- **Estado:** Especificación arquitectónica del estado de seguridad de identidad y elegibilidad de autenticación  

**Depende de:**

- `00_AUTHENTICATION_PROJECT_CONTEXT.md`
- `01_AUTHENTICATION_ARCHITECTURE.md`
- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de determinar:

> **Si una Identity existente se encuentra actualmente en condiciones válidas para iniciar, continuar, restaurar o mantener una autenticación.**

VoltStack distinguirá estrictamente:

```text
Identity exists
```

de:

```text
Identity may authenticate
```

y también de:

```text
Identity currently has valid AuthenticationContext
```

Estas tres condiciones son diferentes.

---

## 2. Principio fundamental

La regla será:

```text
EXISTENCE
    ≠
AUTHENTICATION ELIGIBILITY
    ≠
CURRENT AUTHENTICATION
```

Una Identity puede:

```text
exist
+
have valid credentials
+
still be ineligible to authenticate
```

por razones como:

```text
disabled
suspended
expired
security hold
tenant restriction
compromise response
administrative lock
identity deletion
credential reset requirement
```

---

## 3. Modelo general

```text
Identity
   │
   ▼
IdentitySecurityState
   │
   ├── Account Status
   ├── Authentication Status
   ├── Security Version
   ├── Credential Version
   ├── Validity Window
   ├── Compromise State
   ├── Tenant Restrictions
   ├── Recovery State
   └── Administrative Restrictions
   │
   ▼
AuthenticationEligibilityEvaluator
   │
   ├── Pre-Authentication Checks
   ├── Post-Credential Checks
   ├── Context Recovery Checks
   └── Step-Up Checks
   │
   ▼
AuthenticationEligibilityDecision
   │
   ├── ELIGIBLE
   ├── CHALLENGE_REQUIRED
   ├── REAUTHENTICATION_REQUIRED
   ├── TEMPORARILY_BLOCKED
   ├── INELIGIBLE
   └── ERROR
```

---

## 4. IdentitySecurityState

`IdentitySecurityState` representará el estado relevante para Authentication de una Identity.

No deberá convertirse en un contenedor de perfil general.

Podrá incluir:

```text
authentication status
security version
credential version
validity window
compromise indicators
administrative restrictions
recovery state
tenant authentication state
```

---

## 5. IdentitySecurityState model

Conceptualmente:

```php
final readonly class IdentitySecurityState
{
    public function __construct(
        public IdentityReference $identity,
        public IdentityAuthenticationStatus $status,
        public SecurityVersion $securityVersion,
        public CredentialVersion $credentialVersion,
        public ?\DateTimeImmutable $validFrom = null,
        public ?\DateTimeImmutable $validUntil = null,
        public IdentityCompromiseState $compromise = IdentityCompromiseState::NONE,
        public array $restrictions = [],
    ) {}
}
```

---

## 6. Separación de Profile y Security State

No deberán cargarse obligatoriamente datos como:

```text
avatar
locale
biography
preferences
marketing settings
```

para decidir Authentication.

Preferir:

```text
Identity Core
+
Identity Security State
```

como carga mínima.

---

## 7. IdentityAuthenticationStatus

Se recomienda un estado explícito.

Valores iniciales:

```text
ACTIVE
PENDING
DISABLED
SUSPENDED
LOCKED
EXPIRED
RECOVERY_REQUIRED
SECURITY_HOLD
COMPROMISED
DELETED
```

---

## 8. ACTIVE

Significa:

> La Identity puede participar normalmente en Authentication, sujeto a credentials, policies, risk y assurance.

No significa:

```text
already authenticated
```

---

## 9. PENDING

Representa una Identity creada pero aún no completamente habilitada.

Ejemplos:

```text
pending email verification
pending administrator approval
pending federation provisioning completion
pending credential enrollment
```

---

## 10. PENDING policy

Una Identity `PENDING` podrá:

```text
authenticate only into onboarding context
```

o:

```text
be completely ineligible
```

según policy.

Esto deberá configurarse explícitamente.

---

## 11. DISABLED

Estado administrativo que impide Authentication.

Ejemplo:

```text
employee terminated
customer account disabled
service account manually disabled
```

---

## 12. SUSPENDED

Representa normalmente una restricción temporal o condicional.

Ejemplos:

```text
compliance investigation
temporary administrative suspension
contract status
security review
```

La semántica exacta deberá definirse por aplicación.

---

## 13. LOCKED

Representará un lock explícito sobre la Identity.

VoltStack deberá evitar usar `LOCKED` automáticamente como única respuesta a múltiples passwords incorrectos.

Esto podría permitir ataques de denial-of-service contra cuentas.

---

## 14. Administrative Lock vs Throttling

Deben distinguirse:

```text
Identity LOCKED
```

de:

```text
Authentication attempt THROTTLED
```

El primero modifica el estado de la Identity.

El segundo controla temporalmente intentos sin cambiar necesariamente la cuenta.

---

## 15. EXPIRED

Aplica a identidades con vida limitada.

Ejemplos:

```text
temporary contractor
guest identity
short-lived service
temporary external operator
```

---

## 16. RECOVERY_REQUIRED

Indica que la Identity no debe continuar con Authentication normal hasta completar una recuperación o actualización de seguridad.

Ejemplos:

```text
forced password reset
MFA reset pending
credential compromise response
administrator-triggered recovery
```

---

## 17. SECURITY_HOLD

Permite bloquear Authentication por una condición de seguridad pendiente.

Ejemplos:

```text
fraud review
suspected takeover
incident response
identity verification review
```

---

## 18. COMPROMISED

Indica que existe evidencia suficiente para considerar comprometida la Identity o su conjunto de credenciales.

Normalmente deberá producir:

```text
authentication rejected
+
existing state invalidation
+
recovery flow
```

---

## 19. DELETED

Una Identity eliminada no deberá restaurarse mediante:

```text
old session
remember-me
refresh token
cached identity snapshot
```

---

## 20. Status extensibility

El conjunto podrá ampliarse, pero deberá evitarse que plugins inventen estados sin semántica conocida por el Eligibility Evaluator.

Preferible:

```text
core status
+
typed restrictions
```

para extensibilidad compleja.

---

## 21. IdentityRestriction

Un estado único puede ser insuficiente.

VoltStack podrá modelar restricciones adicionales:

```text
AuthenticationRestriction
```

Ejemplos:

```text
PASSWORD_AUTH_DISABLED
INTERACTIVE_LOGIN_DISABLED
SERVICE_LOGIN_ONLY
TENANT_LOGIN_DISABLED
MFA_ENROLLMENT_REQUIRED
PASSWORD_CHANGE_REQUIRED
RECOVERY_ONLY
STEP_UP_REQUIRED
TIME_WINDOW_RESTRICTED
```

---

## 22. Por qué Status + Restrictions

Una Identity puede estar:

```text
ACTIVE
```

pero tener:

```text
PASSWORD_AUTH_DISABLED
```

permitiendo Passkey/OIDC.

Otro ejemplo:

```text
ACTIVE
+
MFA_ENROLLMENT_REQUIRED
```

puede permitir autenticación limitada para enrolar MFA.

---

## 23. AuthenticationRestrictionInterface

Conceptualmente:

```php
interface AuthenticationRestrictionInterface
{
    public function type(): AuthenticationRestrictionType;
}
```

---

## 24. Restrictions no son Permissions

No deberán expresar:

```text
cannot edit invoices
cannot delete users
```

Eso pertenece a Authorization.

Las restricciones de este sistema se refieren únicamente a:

```text
whether/how authentication may be established or continued
```

---

## 25. SecurityVersion

`SecurityVersion` será un contador o token de versión para invalidar autenticaciones persistidas.

Ejemplo:

```text
Identity security version = 7
```

Session:

```text
security version = 6
```

Resultado:

```text
STALE AUTHENTICATION STATE
```

---

## 26. SecurityVersion value object

Conceptualmente:

```php
final readonly class SecurityVersion
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

La implementación podrá usar:

```text
integer
ULID
opaque version token
timestamp-derived version
```

---

## 27. Cuándo incrementar SecurityVersion

Eventos típicos:

```text
forced logout all sessions
credential compromise
password reset after compromise
MFA reset
account recovery
administrative security reset
critical identity-state change
```

---

## 28. Qué invalida SecurityVersion

Podrá invalidar:

```text
sessions
remember-me tokens
refresh tokens
persistent authentication contexts
delegation states
```

según strategy.

---

## 29. CredentialVersion

Debe distinguirse de `SecurityVersion`.

`CredentialVersion` representa cambios en las credentials de Authentication.

Ejemplos:

```text
password changed
passkey added
passkey removed
API credential rotated
MFA credential updated
```

---

## 30. SecurityVersion vs CredentialVersion

Ejemplo:

```text
display name changed
    neither changes

password changed
    CredentialVersion changes
    maybe SecurityVersion changes

account compromise
    SecurityVersion changes
    CredentialVersion may or may not
```

---

## 31. Version policy

VoltStack no deberá codificar rígidamente:

```text
every credential change invalidates everything
```

Se utilizará:

```text
AuthenticationStateInvalidationPolicy
```

---

## 32. AuthenticationStateInvalidationPolicy

Decidirá efectos como:

```text
revoke current session?
revoke other sessions?
revoke remember-me?
revoke refresh tokens?
increment security version?
require step-up?
```

---

## 33. Ejemplo — password change normal

Una policy podría definir:

```text
current session:
    keep

other sessions:
    revoke

remember-me:
    revoke

refresh tokens:
    rotate/revoke

SecurityVersion:
    increment
```

---

## 34. Ejemplo — account recovery

Más estricta:

```text
all sessions:
    revoke

all persistent tokens:
    revoke

MFA trusted devices:
    revoke

SecurityVersion:
    increment

RecoveryState:
    clear after completion
```

---

## 35. Validity Window

Una Identity podrá tener:

```text
validFrom
validUntil
```

---

## 36. validFrom

Antes de esa fecha:

```text
INELIGIBLE
```

Ejemplo:

```text
contractor account activated Monday
```

---

## 37. validUntil

Después:

```text
EXPIRED
```

sin necesidad de eliminar Identity.

---

## 38. Clock abstraction

Todas estas evaluaciones deberán usar:

```text
ClockInterface
```

Nunca llamadas dispersas a:

```php
new DateTimeImmutable();
```

dentro de cada checker.

---

## 39. Time-zone semantics

Las fechas de seguridad deberán almacenarse y compararse usando instantes inequívocos.

Preferencia:

```text
UTC internally
```

aunque presentación/localización pertenezca a otra capa.

---

## 40. IdentityCompromiseState

Se deberá modelar por separado del status general cuando sea útil.

Valores posibles:

```text
NONE
SUSPECTED
CONFIRMED
RECOVERING
RESOLVED
```

---

## 41. SUSPECTED compromise

Puede producir:

```text
step-up required
restricted authentication methods
risk escalation
new device challenge
```

sin bloquear totalmente.

---

## 42. CONFIRMED compromise

Normalmente:

```text
deny normal authentication
invalidate persistent state
force recovery
```

---

## 43. RECOVERING

Solo deberá permitir los flows necesarios para completar recuperación.

---

## 44. RESOLVED

La Identity puede volver a estado operativo tras ejecutar las medidas configuradas.

---

## 45. Compromise provenance

No deberá almacenarse únicamente:

```text
compromised = true
```

Puede ser necesario conservar referencias seguras como:

```text
detected_at
source category
incident reference
resolution timestamp
```

sin llenar Authentication con información de incident management.

---

## 46. AuthenticationEligibilityEvaluator

Será el componente central.

Contrato conceptual:

```php
interface AuthenticationEligibilityEvaluatorInterface
{
    public function evaluate(
        AuthenticationEligibilityRequest $request
    ): AuthenticationEligibilityDecision;
}
```

---

## 47. AuthenticationEligibilityRequest

Podrá contener:

```text
Identity
IdentitySecurityState
AuthenticationOperation
AuthenticationMethod
Firewall
Tenant
Current AuthenticationContext
Risk Context
AuthenticationTransaction
```

---

## 48. Eligibility no depende siempre de Credential validity

Puede ejecutarse en varias fases.

Por ello se distinguirán check points.

---

## 49. Check points

VoltStack deberá soportar al menos:

```text
PRE_AUTHENTICATION
POST_PRIMARY_CREDENTIAL
POST_FACTOR
CONTEXT_RECOVERY
STEP_UP
REAUTHENTICATION
```

---

## 50. PRE_AUTHENTICATION

Se ejecuta después de resolver Identity pero antes de operaciones de credencial cuando sea seguro hacerlo.

Podrá detectar:

```text
deleted
expired
hard security hold
method disabled
```

---

## 51. Timing concern

Revelar que una Identity está disabled antes de password verification puede facilitar account enumeration.

Por ello:

```text
internal decision
```

y:

```text
external response
```

deben separarse.

---

## 52. Post-Credential Check

Después de verificar la prueba primaria se podrán aplicar estados que no deseamos revelar a un atacante no autenticado.

Ejemplo:

```text
account suspended
password change required
MFA enrollment required
```

---

## 53. PreAuthenticationIdentityChecker

Contrato conceptual:

```php
interface PreAuthenticationIdentityCheckerInterface
{
    public function check(
        IdentitySecurityCheckRequest $request
    ): IdentitySecurityCheckResult;
}
```

---

## 54. PostAuthenticationIdentityChecker

Conceptualmente:

```php
interface PostAuthenticationIdentityCheckerInterface
{
    public function check(
        VerifiedIdentitySecurityCheckRequest $request
    ): IdentitySecurityCheckResult;
}
```

---

## 55. Por qué separar ambos

Permite equilibrar:

```text
fail early
performance
security
enumeration resistance
UX
```

---

## 56. IdentitySecurityCheckResult

Estados posibles:

```text
PASS
DENY
CHALLENGE
REAUTHENTICATE
RECOVERY_REQUIRED
TEMPORARILY_BLOCKED
ERROR
```

---

## 57. AuthenticationEligibilityDecision

Resultado consolidado:

```text
ELIGIBLE
ELIGIBLE_WITH_RESTRICTIONS
CHALLENGE_REQUIRED
REAUTHENTICATION_REQUIRED
RECOVERY_REQUIRED
TEMPORARILY_BLOCKED
INELIGIBLE
ERROR
```

---

## 58. ELIGIBLE

El pipeline puede continuar normalmente.

No significa que la Credential sea válida.

---

## 59. ELIGIBLE_WITH_RESTRICTIONS

Ejemplo:

```text
Identity may authenticate
but only using Passkey/OIDC
```

o:

```text
may enter security setup flow only
```

---

## 60. CHALLENGE_REQUIRED

Estado de Identity puede exigir evidencia adicional.

Ejemplo:

```text
suspicious security state
requires passkey
```

---

## 61. REAUTHENTICATION_REQUIRED

Puede utilizarse durante recovery/context restore.

Ejemplo:

```text
session valid cryptographically
but identity state now requires fresh authentication
```

---

## 62. RECOVERY_REQUIRED

La autenticación normal debe ceder a un recovery flow.

---

## 63. TEMPORARILY_BLOCKED

Puede representar:

```text
security cooldown
temporary administrative block
time-window restriction
```

No deberá confundirse con rate-limit de intentos, aunque ambos puedan coexistir.

---

## 64. INELIGIBLE

La Identity no puede autenticar en este contexto.

---

## 65. ERROR

No se pudo determinar el estado de manera confiable.

Debe fallar cerrado.

---

## 66. Method-specific eligibility

Una Identity puede permitir:

```text
passkey
OIDC
```

y deshabilitar:

```text
password
```

sin estar globalmente disabled.

---

## 67. AuthenticationMethodPolicy

Podrá existir:

```text
AllowedAuthenticationMethods
DeniedAuthenticationMethods
```

dentro del state/policy result.

---

## 68. Ejemplo password disabled

```text
Identity status = ACTIVE
Restrictions:
    PASSWORD_AUTH_DISABLED
```

Request:

```text
PasswordAuthenticator
```

Resultado:

```text
INELIGIBLE_FOR_METHOD
```

---

## 69. Passkey remains valid

La misma Identity:

```text
PasskeyAuthenticator
```

puede continuar.

---

## 70. Federation-only account

Una Identity podrá configurarse:

```text
LOCAL_PASSWORD_DISABLED
FEDERATED_LOGIN_REQUIRED
```

---

## 71. Service-only identity

```text
INTERACTIVE_LOGIN_DISABLED
SERVICE_AUTH_ALLOWED
```

---

## 72. Operation-specific eligibility

Una Identity puede ser elegible para:

```text
RECOVER
```

pero no para:

```text
LOGIN
```

---

## 73. Recovery operation

Una Identity `RECOVERY_REQUIRED` podrá participar únicamente en:

```text
ACCOUNT_RECOVERY
CREDENTIAL_RESET
SECURITY_REENROLLMENT
```

---

## 74. Login attempt during recovery

Debe producir:

```text
RECOVERY_REQUIRED
```

sin crear un AuthenticationContext normal.

---

## 75. Restricted AuthenticationContext

En ciertos productos puede ser útil crear un contexto limitado para onboarding/recovery.

Sin embargo, deberá ser explícitamente distinto de un Context normal.

---

## 76. LimitedAuthenticationContext

Podría existir:

```text
RestrictedAuthenticationContext
```

con:

```text
purpose
allowed authentication operations
expiration
provenance
```

pero deberá evitar confundirse con Authorization general.

---

## 77. Recomendación V1

Para mantener fronteras claras:

```text
normal AuthenticationContext
```

solo después de Authentication completa.

Flows de recovery utilizarán:

```text
AuthenticationTransaction
```

hasta completarse.

---

## 78. Tenant-level Authentication Status

Además del estado global de Identity, puede existir estado por tenant.

Ejemplos:

```text
ACTIVE in Tenant A
SUSPENDED in Tenant B
```

---

## 79. TenantAuthenticationState

Podrá representar:

```text
tenant
identity
status
restrictions
security version override
validity
```

---

## 80. Global vs tenant state

La elegibilidad efectiva será combinación de:

```text
Global Identity Security State
+
Tenant Authentication State
+
Firewall Policy
+
Operation
```

---

## 81. EffectiveSecurityState

Podrá existir internamente:

```text
EffectiveIdentitySecurityState
```

calculado por:

```text
IdentitySecurityStateResolver
```

---

## 82. State precedence

Ejemplo:

```text
Global = ACTIVE
Tenant = SUSPENDED

Effective = SUSPENDED
```

---

## 83. Global hard-disable

Si:

```text
Global = DISABLED
```

ningún tenant deberá reactivar implícitamente la Identity.

---

## 84. Restriction composition

Las restricciones podrán combinarse mediante política segura.

Por defecto:

```text
more restrictive wins
```

para restricciones equivalentes.

---

## 85. Tenant switch validation

Una sesión válida en Tenant A no deberá reutilizar el estado de elegibilidad de A para Tenant B.

---

## 86. SecurityVersion per tenant

Aplicaciones avanzadas podrán necesitar:

```text
globalSecurityVersion
tenantSecurityVersion
```

---

## 87. Ejemplo

Un usuario pierde acceso a un tenant:

```text
tenantSecurityVersion++
```

sin cerrar obligatoriamente sesiones en otros tenants.

---

## 88. Session snapshot

AuthenticationSession podrá persistir:

```text
identity reference
global security version
tenant security version if applicable
authentication state profile version
```

---

## 89. Context recovery validation

Al restaurar:

```text
Session
   ↓
IdentitySecurityStateResolver
   ↓
Version validation
   ↓
Eligibility
```

---

## 90. Version mismatch

Resultado:

```text
STALE
```

No reconstruir Context antes de resolver la policy.

---

## 91. Stale state outcomes

Según causa:

```text
reauthentication
session revoke
recovery required
full logout
```

---

## 92. Forced Logout

VoltStack deberá soportar:

```text
logout current session
logout selected sessions
logout device
logout all sessions
```

---

## 93. Forced logout mechanism

Puede combinar:

```text
Session revocation
+
SecurityVersion increment
```

---

## 94. Por qué ambos

Session revocation permite precisión.

SecurityVersion ayuda a invalidar estados distribuidos o sesiones que no fueron enumeradas fácilmente.

---

## 95. ForcedLogoutService

Podrá existir:

```php
interface ForcedLogoutServiceInterface
{
    public function revoke(
        IdentityReference $identity,
        ForcedLogoutScope $scope
    ): ForcedLogoutResult;
}
```

---

## 96. ForcedLogoutScope

Ejemplos:

```text
CURRENT_SESSION
SESSION
DEVICE
TENANT
ALL
```

---

## 97. Logout no elimina Identity

Solo invalida Authentication state.

---

## 98. Credential reset effects

Cambiar PasswordCredential deberá invocar una policy de invalidación.

---

## 99. MFA reset effects

Quitar/resetear factores puede requerir:

```text
SecurityVersion increment
trusted-device revocation
session revocation
step-up reset
```

---

## 100. Passkey removal

La eliminación de una passkey puede:

```text
increment CredentialVersion
revoke sessions authenticated solely with that credential
```

si la policy lo requiere.

---

## 101. Credential compromise

Una credential concreta puede marcarse:

```text
COMPROMISED
```

sin marcar toda Identity.

---

## 102. Credential-specific compromise

Ejemplo:

```text
API key compromised
```

podrá revocar solo:

```text
that key
sessions/tokens derived from it
```

si provenance suficiente existe.

---

## 103. Identity-wide compromise

Cuando no sea posible aislar:

```text
Identity compromise
    ↓
SecurityVersion increment
    ↓
all states invalid
```

---

## 104. Authentication State Derivation

Sessions/tokens deberán registrar suficiente provenance para saber:

```text
which credential/method established them
```

cuando se necesite invalidación selectiva.

---

## 105. AuthenticationDerivationReference

Podrá incluir:

```text
credential id
method
factor ids
session id
token family
authentication chain
```

sin secrets.

---

## 106. Selective invalidation

Ejemplo:

```text
Passkey P1 compromised
```

El sistema puede buscar contextos derivados de P1.

Esto es una capacidad avanzada y dependerá del storage.

---

## 107. Simpler V1 policy

V1 podrá optar por:

```text
critical credential compromise
    → increment global SecurityVersion
```

y evolucionar posteriormente a invalidación selectiva.

---

## 108. IdentitySecurityStateProvider

Se recomienda separar carga del estado.

Contrato:

```php
interface IdentitySecurityStateProviderInterface
{
    public function get(
        IdentityReference $identity,
        IdentitySecurityStateContext $context
    ): IdentitySecurityStateResult;
}
```

---

## 109. Provider sources

El estado puede provenir de:

```text
identity table
security table
LDAP attributes
enterprise directory
remote identity service
tenant membership store
```

---

## 110. SecurityStateProvider no es IdentityProvider

Aunque una implementación pueda compartir storage:

```text
IdentityProvider
    loads identity

IdentitySecurityStateProvider
    loads authentication-sensitive state
```

mantener contratos separados permite optimización.

---

## 111. Security state freshness

Este estado suele necesitar más freshness que un profile.

---

## 112. Cache TTL

Debe ser conservador para:

```text
disabled
securityVersion
compromised
```

---

## 113. Event-driven invalidation

En sistemas distribuidos puede usarse:

```text
IdentitySecurityStateChanged
```

para invalidar caches.

---

## 114. SecurityState cache key

Debe incluir:

```text
IdentityReference
tenant scope if applicable
```

---

## 115. No cross-tenant cache contamination

Especialmente si raw IDs coinciden.

---

## 116. State provider outage

Si no puede determinarse el estado:

```text
ERROR
```

No asumir `ACTIVE`.

---

## 117. Fail closed

Esta será una invariante crítica:

> **Unknown security state never becomes eligible by default.**

---

## 118. Graceful availability policies

Algunas aplicaciones pueden querer alta disponibilidad durante outage.

Cualquier comportamiento alternativo deberá ser:

```text
explicit
time bounded
based on trusted cached state
auditable
risk-aware
```

Nunca fallback accidental.

---

## 119. Stale cache policy

Podrá existir:

```text
DENY_ON_STALE
ALLOW_WITHIN_MAX_STALENESS
REQUIRE_STEP_UP
```

para entornos específicos.

---

## 120. Default recomendado

Para estados críticos:

```text
DENY_ON_UNKNOWN_OR_UNACCEPTABLY_STALE
```

---

## 121. Authentication Eligibility Rules

Podrán representarse mediante:

```text
IdentityAuthenticationEligibilityRule
```

---

## 122. Core rules

Ejemplos:

```text
StatusEligibilityRule
ValidityWindowRule
CompromiseRule
AuthenticationMethodRestrictionRule
TenantAuthenticationRule
RecoveryStateRule
SecurityHoldRule
```

---

## 123. Rule composition

El `AuthenticationEligibilityEvaluator` coordinará reglas, no deberá permitir que una rule posterior convierta un hard deny en allow.

---

## 124. Decision severity

Orden conceptual:

```text
ERROR / UNKNOWN
HARD_DENY
RECOVERY_REQUIRED
TEMPORARY_BLOCK
CHALLENGE_REQUIRED
RESTRICTION
ALLOW
```

con semántica determinista.

---

## 125. Hard Deny

Ejemplos:

```text
DELETED
CONFIRMED_COMPROMISE
ADMIN_DISABLED
```

---

## 126. Soft restriction

Ejemplo:

```text
password disabled
but passkey allowed
```

---

## 127. Challenge result

Ejemplo:

```text
suspicious state
    → require fresh Passkey
```

---

## 128. Recovery result

Ejemplo:

```text
administrator reset MFA
    → recovery/enrollment required
```

---

## 129. EligibilityExplanation

Para debugging/auditoría podrá existir:

```text
AuthenticationEligibilityExplanation
```

---

## 130. Explanation content

```text
effective status
rules evaluated
restriction applied
version result
tenant state
final decision
```

---

## 131. Production exposure

No deberá mostrarse todo al cliente.

Ejemplo interno:

```text
Identity status = SUSPENDED
```

externamente:

```text
Unable to authenticate
```

según policy.

---

## 132. UX exception

Aplicaciones pueden decidir mostrar:

```text
Your account has been suspended
```

después de demostrar control de la cuenta.

La arquitectura deberá permitirlo sin exigirlo.

---

## 133. Enumeration-safe failure mapping

Antes de credential verification:

```text
NOT_FOUND
DISABLED
LOCKED
```

pueden mapearse al mismo resultado externo.

---

## 134. Post-verification disclosure

Después de proof exitoso, la aplicación puede ofrecer información más específica si es seguro y apropiado.

---

## 135. Temporary Blocking

No deberá reemplazar al Rate Limiter.

Puede representar restricciones como:

```text
identity may not authenticate before timestamp X
```

---

## 136. BlockedUntil

Podrá existir:

```text
blockedUntil
```

en una restriction.

---

## 137. Cooldown

Después de eventos sensibles:

```text
account recovery
credential rotation
```

la aplicación podría aplicar cooldown para algunas operaciones de Authentication.

---

## 138. Risk-driven block

Risk Engine podrá sugerir bloqueo temporal, pero el estado persistente deberá actualizarse mediante servicio explícito.

No permitir que un evaluator observational mute silenciosamente Identity.

---

## 139. State mutation service

Se recomienda:

```text
IdentitySecurityStateManager
```

para realizar cambios.

---

## 140. Manager responsibilities

```text
disable
enable
suspend
resume
mark compromised
begin recovery
complete recovery
increment security version
set restrictions
```

---

## 141. Command model

Cambios importantes podrán usar comandos:

```text
DisableIdentity
SuspendIdentity
MarkIdentityCompromised
RequireAccountRecovery
ResetIdentitySecurity
```

---

## 142. Mutation authorization

Authentication System define la operación.

Quién puede ejecutar administrativamente:

```text
DisableIdentity
```

pertenece a Authorization/application layer.

---

## 143. State transition validation

No todos los cambios serán válidos.

Ejemplo:

```text
DELETED → ACTIVE
```

probablemente prohibido.

---

## 144. IdentitySecurityStateMachine

Podrá existir:

```text
ACTIVE
  ├──→ SUSPENDED
  ├──→ DISABLED
  ├──→ SECURITY_HOLD
  ├──→ COMPROMISED
  └──→ DELETED
```

---

## 145. Ejemplo de recovery transitions

```text
COMPROMISED
    ↓
RECOVERY_REQUIRED
    ↓
RECOVERING
    ↓
ACTIVE
```

---

## 146. State transition guard

```php
interface IdentitySecurityStateTransitionGuardInterface
{
    public function canTransition(
        IdentitySecurityState $from,
        IdentitySecurityStateMutation $mutation
    ): bool;
}
```

---

## 147. Transition atomicity

Cambios como:

```text
mark compromised
+
increment security version
+
revoke sessions
```

deben coordinarse coherentemente.

---

## 148. IdentitySecurityMutationCoordinator

Podrá encargarse de:

```text
state persistence
version increment
revocation
events
audit
cache invalidation
```

---

## 149. Security transaction boundary

Cuando estado y sessions comparten DB puede utilizarse transaction.

En sistemas distribuidos:

```text
idempotency
ordered operations
eventual revocation
compensation
```

---

## 150. Compromise mutation ordering

Una estrategia segura:

```text
1. persist identity as compromised
2. increment security version
3. revoke persistent auth state
4. invalidate caches
5. emit security events
```

Así un fallo posterior no deja la Identity activa.

---

## 151. Enable operation

Reactivar una Identity deberá revisar:

```text
credentials
recovery state
MFA state
security version
administrative requirements
```

No simplemente cambiar:

```text
disabled = false
```

en todos los casos.

---

## 152. Reactivation policy

Puede exigir:

```text
password reset
MFA enrollment
fresh recovery
administrator review
```

---

## 153. Pending Activation

El paso:

```text
PENDING → ACTIVE
```

puede requerir:

```text
email verification
credential enrollment
identity proofing
admin approval
```

según aplicación.

---

## 154. Email verification

Email verification no deberá confundirse con Authentication completa.

Puede ser una condición de eligibility.

---

## 155. EmailVerifiedRestriction

Por ejemplo:

```text
PENDING_EMAIL_VERIFICATION
```

puede impedir login normal o limitarlo a onboarding.

---

## 156. Identity proofing

En sistemas regulados puede existir un nivel separado de:

```text
identity verification / KYC
```

No deberá confundirse con Authentication.

Sin embargo, su estado puede afectar eligibility mediante adapter/policy.

---

## 157. Identity proofing boundary

Authentication pregunta:

```text
is this actor controlling the credential for Identity X?
```

Identity proofing pregunta:

```text
is Identity X really the real-world person claimed?
```

Son sistemas diferentes.

---

## 158. Authentication Eligibility can consume proofing state

Ejemplo:

```text
KYC incomplete
    → login permitted
    → some application actions denied
```

Eso normalmente pertenece a Authorization/business policy.

Solo bloquear Authentication si existe una razón arquitectónica explícita.

---

## 159. Avoid domain pollution

No poner estados como:

```text
SUBSCRIPTION_EXPIRED
INVOICE_UNPAID
```

dentro de IdentitySecurityState salvo que realmente determinen Authentication.

---

## 160. Business suspension vs Authentication suspension

Una cuenta comercial puede estar suspendida para ciertas operaciones y seguir autenticándose.

Eso debería resolverse en Authorization/business layer.

---

## 161. Principle

> **No usar Authentication Eligibility para reemplazar Authorization.**

---

## 162. Context Recovery Checks

La elegibilidad deberá evaluarse al restaurar Authentication.

```text
Session
   ↓
Identity Reference
   ↓
Security State
   ↓
Eligibility
   ↓
Context
```

---

## 163. Session valid but Identity disabled

Resultado:

```text
Session invalidated
No AuthenticationContext
```

---

## 164. Token valid but Identity disabled

Si la arquitectura requiere live state checking:

```text
Token signature valid
    ↓
Identity state disabled
    ↓
Authentication rejected
```

---

## 165. Self-contained stateless tokens

Un sistema totalmente stateless puede no consultar Identity state cada request.

Eso introduce trade-offs de revocación.

VoltStack deberá soportar policies claras.

---

## 166. Token identity-state modes

Podrían existir:

```text
SNAPSHOT_ONLY
SECURITY_VERSION_CHECK
LIVE_IDENTITY_CHECK
INTROSPECTION
```

---

## 167. SNAPSHOT_ONLY

Mayor rendimiento, menor revocación inmediata.

Adecuado solo para tokens muy cortos y ciertos contextos.

---

## 168. SECURITY_VERSION_CHECK

Comprueba una versión central ligera.

---

## 169. LIVE_IDENTITY_CHECK

Consulta estado actual.

---

## 170. INTROSPECTION

El estado del token/identity se obtiene desde un authority remoto.

---

## 171. Policy per Firewall

Ejemplo:

```text
internal API:
    SECURITY_VERSION_CHECK

external short-lived token:
    SNAPSHOT_ONLY

admin API:
    LIVE_IDENTITY_CHECK
```

---

## 172. Context freshness

Una AuthenticationContext existente puede seguir criptográficamente válida pero quedar obsoleta por cambios de IdentitySecurityState.

---

## 173. ContextValidator integration

`AuthenticationContextValidator` deberá consultar:

```text
session/token validity
identity security version
eligibility
tenant state
```

según estrategia.

---

## 174. Step-Up after state change

Ejemplo:

```text
Identity ACTIVE
session AAL1
security restriction added:
    require AAL2
```

La sesión no necesita necesariamente destruirse.

Puede resultar:

```text
STEP_UP_REQUIRED
```

---

## 175. Hard state change

En cambio:

```text
Identity DISABLED
```

debe invalidar el contexto.

---

## 176. Restriction severity

Las restricciones deberán declarar efectos como:

```text
INVALIDATE
STEP_UP
REAUTHENTICATE
RECOVERY
METHOD_DENY
```

---

## 177. Context reaction matrix

Ejemplo:

| Cambio de estado | Contexto actual |
| --- | --- |
| `DISABLED` | Invalidar |
| `DELETED` | Invalidar |
| `COMPROMISED` | Invalidar + recovery |
| `PASSWORD_AUTH_DISABLED` | Puede conservar contexto existente según policy |
| `REQUIRE_AAL2` | Step-up |
| `PASSWORD_CHANGE_REQUIRED` | Reauthentication/recovery flow |
| `TENANT_SUSPENDED` | Invalidar contexto de ese tenant |

---

## 178. AuthenticationEligibilityPolicy

Podrá personalizar estas reacciones.

---

## 179. Identity State Snapshot

Una Session podrá conservar:

```text
status snapshot
security version
restriction version
```

pero no confiar indefinidamente en ellos.

---

## 180. RestrictionVersion

En sistemas avanzados podría existir:

```text
AuthenticationPolicyVersion
```

o:

```text
RestrictionVersion
```

para invalidar contexts cuando cambian requisitos.

---

## 181. Simplicidad V1

V1 puede usar:

```text
SecurityVersion
```

como versión general de cualquier cambio crítico.

---

## 182. Distributed invalidation

Cambios de estado deberán propagarse a:

```text
application workers
session stores
token stores
caches
websocket connections
```

según arquitectura.

---

## 183. SecurityStateChanged event

Podrá emitir:

```text
IdentitySecurityStateChanged
```

con:

```text
identity reference
old status
new status
new security version
timestamp
reason category
```

sin secrets.

---

## 184. Cache invalidation subscriber

Podrá escuchar el evento y eliminar:

```text
identity security cache
authentication context caches
```

---

## 185. WebSocket invalidation

Conexiones persistentes deberán volver a validar estado o recibir señal de cierre cuando:

```text
identity disabled
security version changes
```

---

## 186. Queue workers

Un queued job con delegated identity deberá validar si su execution policy exige que la Identity siga elegible.

---

## 187. Long-running processes

CLI daemons o workers deberán evitar conservar eligibility result indefinidamente.

---

## 188. Eligibility TTL

Para contextos de larga duración podrá existir un máximo tiempo antes de reevaluar.

---

## 189. AuthenticationStateLease

Conceptualmente:

```text
Eligibility verified until:
    timestamp
```

para conexiones largas.

---

## 190. No indefinite trust

Especialmente:

```text
WebSocket
stream
long-lived RPC
```

---

## 191. Concurrency — disable during login

Caso:

```text
Request A:
    verifies password

Request B:
    administrator disables identity

Request A:
    about to commit session
```

Debe impedirse que A cree una sesión después del disable.

---

## 192. Commit-time revalidation

Para estados críticos, Authentication Commit podrá verificar nuevamente:

```text
SecurityVersion
Eligibility
```

antes de activar Context.

---

## 193. Optimistic version check

Request A cargó:

```text
SecurityVersion = 10
```

Antes del commit:

```text
expected version = 10
```

Si actual:

```text
11
```

el commit falla.

---

## 194. Why commit-time check

Evita TOCTOU:

```text
check eligibility
    ↓
state changes
    ↓
commit authentication
```

---

## 195. EligibilitySnapshot

El evaluator podrá producir:

```text
IdentityEligibilitySnapshot
```

con:

```text
security version
evaluated at
effective status
restrictions
```

que el Commit Coordinator valida.

---

## 196. Concurrency — account recovery

Recovery completion y login concurrente deberán coordinar SecurityVersion.

---

## 197. Concurrency — session revocation

Revocation deberá ser visible antes de permitir context reconstruction.

---

## 198. Atomic state change

Stores deberán proporcionar CAS/version checks cuando sea apropiado.

---

## 199. AuthenticationEligibilityCache

Puede memoizar dentro de request:

```text
IdentityReference + tenant → eligibility
```

---

## 200. Request memoization safe

Sí, siempre que:

```text
same operation/request
```

y commit revalidation se aplique donde sea necesario.

---

## 201. Cross-request cache

Necesita TTL + invalidation.

---

## 202. High-security mode

Podrá deshabilitar cross-request security-state caching.

---

## 203. State source hierarchy

En modelos complejos:

```text
Global Identity State
Tenant State
Provider State
Administrative Security State
Credential State
```

podrán contribuir al Effective State.

---

## 204. StateResolver

Se recomienda:

```text
IdentitySecurityStateResolver
```

que componga fuentes.

---

## 205. StateSourceInterface

```php
interface IdentitySecurityStateSourceInterface
{
    public function resolve(
        IdentitySecurityStateRequest $request
    ): IdentitySecurityStateFragment;
}
```

---

## 206. Fragment examples

```text
GlobalIdentityStateSource
TenantIdentityStateSource
DirectoryStateSource
SecurityIncidentStateSource
```

---

## 207. Avoid unnecessary complexity

V1 debería comenzar con:

```text
primary state provider
+
optional tenant state
```

y mantener SPI extensible.

---

## 208. State merge

El merge deberá ser determinista y monotónico respecto a hard restrictions.

---

## 209. Restriction cannot be accidentally cancelled

Ejemplo:

```text
Global DISABLED
Tenant ACTIVE
```

debe seguir:

```text
DISABLED
```

---

## 210. Directory-disabled identity

Si LDAP/AD indica disabled, un local cache diciendo active no deberá ganarle fuera de una policy de staleness explícita.

---

## 211. Federated provider account disabled

OIDC normalmente autentica solo cuentas que el IdP permite, pero una Identity local puede estar disabled aunque el IdP autentique correctamente.

---

## 212. Local state remains authoritative where configured

```text
OIDC success
    ↓
local Identity DISABLED
    ↓
reject
```

---

## 213. Federated deprovisioning

SCIM u otros sistemas pueden actualizar IdentitySecurityState fuera de login.

Authentication deberá reaccionar mediante SecurityVersion/invalidation.

---

## 214. SCIM boundary

SCIM provisioning no forma parte de Auth Core, pero podrá integrarse con:

```text
IdentitySecurityStateManager
```

---

## 215. Service account lifecycle

Service Identity también deberá soportar:

```text
ACTIVE
DISABLED
EXPIRED
COMPROMISED
```

---

## 216. Machine account expiry

Puede ser automático por:

```text
validUntil
certificate lifecycle
workload registration
```

---

## 217. Workload identity eligibility

Puede depender de:

```text
trusted workload authority
active deployment
service registration
```

mediante provider-specific checker.

---

## 218. Device identity eligibility

IoT Device Identity puede tener:

```text
ACTIVE
REVOKED
COMPROMISED
```

independientemente de User Device Context.

---

## 219. Temporary identities

Deberán requerir:

```text
validUntil
purpose
```

y preferiblemente no podrán renovarse implícitamente.

---

## 220. Account Status Reason

Internamente puede existir:

```text
IdentityStatusReason
```

para auditoría.

Ejemplos:

```text
ADMIN_DISABLED
EMPLOYMENT_ENDED
SECURITY_INCIDENT
RECOVERY_STARTED
TEMPORARY_HOLD
```

---

## 221. Reason security

No todas las razones deberán exponerse al usuario.

---

## 222. Reason codes

Preferir códigos tipados, no strings libres para decisiones críticas.

---

## 223. Mutation reason required

Operaciones sensibles como:

```text
disable
mark compromised
force recovery
```

podrán requerir un reason/audit context.

---

## 224. Actor information

La mutación puede registrar:

```text
initiating actor
system
administrator
security automation
```

en Audit, pero no dentro del IdentitySecurityState mínimo.

---

## 225. Audit events

Deberán contemplarse:

```text
IdentityDisabled
IdentityEnabled
IdentitySuspended
IdentityResumed
IdentityLocked
IdentityUnlocked
IdentityMarkedCompromised
IdentityRecoveryRequired
IdentityRecoveryCompleted
SecurityVersionChanged
AuthenticationMethodDisabled
AuthenticationRestrictionAdded
AuthenticationRestrictionRemoved
ForcedLogoutExecuted
```

---

## 226. Authentication failure audit

Un intento bloqueado por state podrá producir:

```text
AuthenticationRejectedByIdentityState
```

sin necesariamente revelar detalles externamente.

---

## 227. Observability spans

```text
auth.identity.security_state.resolve
auth.identity.eligibility.evaluate
auth.identity.security_state.mutate
auth.identity.version.validate
```

---

## 228. Metrics

Ejemplos:

```text
identity_authentication_ineligible_total
identity_security_state_resolution_total
identity_security_state_error_total
security_version_mismatch_total
forced_logout_total
identity_recovery_required_total
```

---

## 229. Metric labels

Seguros:

```text
status
restriction_type
firewall
operation
outcome
```

Evitar:

```text
identity id
email
tenant id de alta cardinalidad
```

cuando no sea apropiado.

---

## 230. Security alerts

Eventos como:

```text
confirmed compromise
repeated stale-session use
revoked identity token attempts
```

pueden alimentar security monitoring.

---

## 231. Eligibility logging

Podrá registrar:

```text
decision
status category
restriction category
security version mismatch
```

sin atributos personales innecesarios.

---

## 232. External response mapping

Ejemplos:

```text
Identity DISABLED
    → generic authentication failure

RECOVERY_REQUIRED after verified password
    → structured recovery_required response

STEP_UP restriction
    → challenge_required
```

---

## 233. SPA response

Ejemplo:

```json
{
    "authentication": {
        "status": "recovery_required",
        "transaction": "txn_..."
    }
}
```

sin exponer razón administrativa interna.

---

## 234. API semantics

Una Identity ineligible al intentar login puede devolver:

```text
authentication failure
```

mientras una session previamente válida que fue revocada:

```text
401 unauthenticated
```

---

## 235. Authorization distinction

Una Identity autenticada que no puede acceder a un recurso:

```text
403
```

Eso no pertenece a este sistema.

---

## 236. Testing — state matrix

Deberán probarse:

```text
ACTIVE
PENDING
DISABLED
SUSPENDED
LOCKED
EXPIRED
RECOVERY_REQUIRED
SECURITY_HOLD
COMPROMISED
DELETED
```

contra distintas operaciones.

---

## 237. Testing — method restrictions

```text
password disabled + password
password disabled + passkey
federated only + password
interactive disabled + service token
```

---

## 238. Testing — version mismatch

```text
session version old
token version old
remember-me version old
tenant version old
```

---

## 239. Testing — tenant state

```text
global active + tenant active
global active + tenant suspended
global disabled + tenant active
tenant switch
```

---

## 240. Testing — concurrency

Especialmente:

```text
disable during login
security version increment during context commit
recovery during session restoration
simultaneous enable/disable
forced logout concurrent with refresh token use
```

---

## 241. Testing — outage

```text
security state store unavailable
cache stale
event invalidation delayed
```

debe fallar según policy configurada.

---

## 242. Property-based testing

Útil para comprobar:

```text
hard-deny monotonicity
state merge rules
version comparison
restriction composition
state transitions
```

---

## 243. Persistent-runtime testing

Request A:

```text
Identity A = DISABLED
```

Request B:

```text
Identity B = ACTIVE
```

El checker compartido no deberá conservar resultado de A.

---

## 244. Stateless service requirement

`AuthenticationEligibilityEvaluator` y checkers deberán ser preferentemente:

```text
stateless
immutable
scope-independent
```

---

## 245. Mutable evaluation state

Debe vivir en:

```text
EligibilityEvaluationContext
AuthenticationOperationContext
request memoization
```

---

## 246. Security invariant — State

### AUTH-STATE-01

Identity existence never implies authentication eligibility.

#### AUTH-STATE-02

Unknown security state never defaults to ACTIVE.

#### AUTH-STATE-03

`DISABLED`, `DELETED` y hard security states fail closed.

#### AUTH-STATE-04

Authentication throttling and Identity lock are different concepts.

#### AUTH-STATE-05

Identity status must not be used as replacement for Authorization.

#### AUTH-STATE-06

SecurityVersion mismatch invalidates state according to configured policy.

#### AUTH-STATE-07

Tenant-bound eligibility must be evaluated in the correct tenant.

#### AUTH-STATE-08

A global hard restriction cannot be weakened by tenant state.

#### AUTH-STATE-09

Security state changes must invalidate relevant caches.

#### AUTH-STATE-10

Existing AuthenticationContext must be revalidated when security policy requires it.

---

## 247. Security invariant — Eligibility

### AUTH-ELIG-01

Eligibility evaluation is operation-aware.

#### AUTH-ELIG-02

Eligibility evaluation is Authentication-method-aware.

#### AUTH-ELIG-03

Pre-auth checks must account for enumeration risk.

#### AUTH-ELIG-04

Post-credential checks may reveal more precise state only under safe policy.

#### AUTH-ELIG-05

Eligibility ERROR never produces authentication success.

#### AUTH-ELIG-06

A recovery-only Identity cannot create a normal AuthenticationContext.

#### AUTH-ELIG-07

Method restrictions do not necessarily invalidate other authentication methods.

#### AUTH-ELIG-08

A hard deny cannot be overridden by a later allow rule.

#### AUTH-ELIG-09

Eligibility is revalidated at commit when TOCTOU risk matters.

#### AUTH-ELIG-10

Eligibility decisions contain no Authorization permissions.

---

## 248. Security invariant — Mutation

### AUTH-STATE-MUT-01

Security-sensitive state mutations are explicit.

#### AUTH-STATE-MUT-02

Critical mutations must be auditable.

#### AUTH-STATE-MUT-03

Compromise handling must prioritize disabling trust before cleanup side effects.

#### AUTH-STATE-MUT-04

Session/token invalidation policy must be deterministic.

#### AUTH-STATE-MUT-05

Invalid transitions must be rejected.

#### AUTH-STATE-MUT-06

Concurrent mutations require version/atomicity control.

#### AUTH-STATE-MUT-07

Re-enabling a compromised Identity may require recovery conditions.

---

## 249. Anti-pattern — boolean enabled only

Evitar limitar el modelo a:

```php
$user->enabled;
```

porque no distingue:

```text
suspended
recovery required
expired
compromised
method-restricted
```

---

## 250. Anti-pattern — failed passwords disable account permanently

Evitar como política base:

```text
5 failures
    ↓
account disabled
```

porque permite lockout attacks.

Preferir:

```text
rate limit
progressive throttling
risk escalation
temporary challenge
```

---

## 251. Anti-pattern — account state in Authorization role

Evitar:

```text
role = suspended
```

para controlar Authentication.

---

## 252. Anti-pattern — authorization in account status

Evitar:

```text
status = cannot_edit_invoices
```

---

## 253. Anti-pattern — provider outage returns ACTIVE

Nunca.

---

## 254. Anti-pattern — stale session survives disable

Si la policy exige live invalidation:

```text
Identity disabled
```

debe impedir reconstruir Context.

---

## 255. Anti-pattern — full session enumeration required for every invalidation

SecurityVersion deberá permitir alternativas más escalables.

---

## 256. Anti-pattern — SecurityVersion as timestamp without semantics

Una fecha puede usarse, pero debe existir comparación y atomicidad claramente definidas.

---

## 257. Anti-pattern — state mutation from event listener

Un listener observational no debería cambiar silenciosamente la seguridad de Identity.

Usar `IdentitySecurityStateManager`.

---

## 258. Anti-pattern — client chooses eligibility state

Nunca confiar en:

```text
status=active
tenant_active=true
```

desde request/token no verificado.

---

## 259. Anti-pattern — token claim overrides local disabled state

Un token puede afirmar:

```text
active=true
```

pero la policy local puede seguir considerando la Identity disabled.

---

## 260. Core components

```text
IdentitySecurityState
IdentityAuthenticationStatus
AuthenticationRestriction
SecurityVersion
CredentialVersion
IdentityCompromiseState
IdentitySecurityStateProvider
IdentitySecurityStateResolver
AuthenticationEligibilityEvaluator
AuthenticationEligibilityDecision
PreAuthenticationIdentityChecker
PostAuthenticationIdentityChecker
AuthenticationStateInvalidationPolicy
IdentitySecurityStateManager
IdentitySecurityStateTransitionGuard
```

---

## 261. Supporting components

```text
ForcedLogoutService
IdentityEligibilitySnapshot
TenantAuthenticationState
IdentitySecurityStateMutationCoordinator
AuthenticationContextSecurityValidator
SecurityStateCache
SecurityStateInvalidationPublisher
```

---

## 262. Namespace sugerido

```text
VoltStack\Quantum\Auth\Identity\Security
VoltStack\Quantum\Auth\Identity\Security\State
VoltStack\Quantum\Auth\Identity\Security\Eligibility
VoltStack\Quantum\Auth\Identity\Security\Mutation
VoltStack\Quantum\Auth\Identity\Security\Invalidation
VoltStack\Quantum\Auth\Identity\Security\Tenant
```

---

## 263. Estructura sugerida

```text
src/Quantum/Auth/
└── Identity/
    └── Security/
        ├── State/
        │   ├── IdentitySecurityState.php
        │   ├── IdentityAuthenticationStatus.php
        │   ├── IdentityCompromiseState.php
        │   ├── SecurityVersion.php
        │   ├── CredentialVersion.php
        │   ├── AuthenticationRestriction.php
        │   ├── IdentitySecurityStateProvider.php
        │   └── IdentitySecurityStateResolver.php
        │
        ├── Eligibility/
        │   ├── AuthenticationEligibilityEvaluator.php
        │   ├── AuthenticationEligibilityRequest.php
        │   ├── AuthenticationEligibilityDecision.php
        │   ├── PreAuthenticationIdentityChecker.php
        │   ├── PostAuthenticationIdentityChecker.php
        │   ├── IdentityEligibilitySnapshot.php
        │   └── Rule/
        │       ├── StatusEligibilityRule.php
        │       ├── ValidityWindowRule.php
        │       ├── CompromiseRule.php
        │       ├── AuthenticationMethodRule.php
        │       ├── RecoveryStateRule.php
        │       └── TenantAuthenticationRule.php
        │
        ├── Mutation/
        │   ├── IdentitySecurityStateManager.php
        │   ├── IdentitySecurityStateTransitionGuard.php
        │   ├── IdentitySecurityMutationCoordinator.php
        │   └── IdentitySecurityMutation.php
        │
        ├── Invalidation/
        │   ├── AuthenticationStateInvalidationPolicy.php
        │   ├── ForcedLogoutService.php
        │   ├── ForcedLogoutScope.php
        │   └── SecurityVersionValidator.php
        │
        └── Tenant/
            ├── TenantAuthenticationState.php
            └── TenantSecurityVersion.php
```

---

## 264. Flujo de login normal

```text
IdentityClaim
    ↓
IdentityResolver
    ↓
Identity
    ↓
SecurityStateResolver
    ↓
Status ACTIVE
    ↓
PRE_AUTH eligibility PASS
    ↓
Password Verification
    ↓
POST_AUTH eligibility PASS
    ↓
Evidence
    ↓
Policy / Assurance
    ↓
AuthenticationContext
```

---

## 265. Flujo de Identity disabled

```text
Identity resolved
    ↓
SecurityState:
    DISABLED
    ↓
Eligibility:
    INELIGIBLE
    ↓
safe external failure
    ↓
no AuthenticationContext
```

---

## 266. Flujo con password deshabilitado

```text
Identity:
    ACTIVE

Restriction:
    PASSWORD_AUTH_DISABLED
        ↓
Password login
        ↓
INELIGIBLE_FOR_METHOD
```

Pero:

```text
Passkey login
    ↓
ELIGIBLE
```

---

## 267. Flujo recovery required

```text
Identity
    ↓
SecurityState:
    RECOVERY_REQUIRED
    ↓
Normal LOGIN
    ↓
RECOVERY_REQUIRED result
    ↓
AuthenticationTransaction
    ↓
Recovery flow
```

---

## 268. Flujo session stale

```text
Session
    securityVersion = 8
        ↓
Identity SecurityState
    securityVersion = 9
        ↓
VersionValidator
        ↓
STALE
        ↓
session invalidated
        ↓
reauthentication / recovery according to reason
```

---

## 269. Flujo admin disable concurrente

```text
Request A:
    password verified
    state version = 5

Request B:
    admin disables identity
    state version = 6

Request A:
    commit-time version check
        ↓
    expected 5
    actual 6
        ↓
    COMMIT REJECTED
```

---

## 270. Flujo tenant-specific

```text
Global Identity:
    ACTIVE

Tenant A:
    ACTIVE

Tenant B:
    SUSPENDED

        ↓

Login Tenant A:
    ELIGIBLE

Login Tenant B:
    INELIGIBLE
```

---

## 271. Flujo compromise response

```text
Security signal
    ↓
IdentitySecurityStateManager
    ↓
mark COMPROMISED
    ↓
SecurityVersion increment
    ↓
session/token invalidation
    ↓
cache invalidation
    ↓
audit
    ↓
future login → RECOVERY_REQUIRED / DENY
```

---

## 272. Flujo forced logout

```text
Administrator / Security Process
        ↓
ForcedLogoutService
        ↓
IdentityReference
        ↓
scope = ALL
        ↓
SecurityVersion increment
        ↓
session revocation
        ↓
remember-me revocation
        ↓
token family revocation
        ↓
active contexts become stale
```

---

## 273. Arquitectura global

```text
                      CANONICAL IDENTITY
                             │
                             ▼
                    Security State Resolver
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
         Global State    Tenant State    Provider State
             │               │                │
             └───────────────┼────────────────┘
                             ▼
                  EffectiveSecurityState
                             │
                             ▼
               Authentication Eligibility
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
        ELIGIBLE       RESTRICTED          DENY
            │                │                │
            ▼                ▼                ▼
      Credential       Challenge /         Failure
      Verification     Recovery
            │
            ▼
     Verified Evidence
            │
            ▼
      Commit-time State
         Revalidation
            │
            ├── version changed → reject
            │
            ▼
    AuthenticationContext
```

---

## 274. Criterios de aceptación

El subsistema será considerado correcto cuando:

1. distinga existencia de Identity y eligibility;
2. soporte estados explícitos;
3. soporte restricciones independientes del status;
4. soporte `ACTIVE`;
5. soporte `PENDING`;
6. soporte `DISABLED`;
7. soporte `SUSPENDED`;
8. soporte `LOCKED`;
9. soporte `EXPIRED`;
10. soporte `RECOVERY_REQUIRED`;
11. soporte `SECURITY_HOLD`;
12. soporte `COMPROMISED`;
13. soporte `DELETED`;
14. distinga account lock de throttling;
15. soporte SecurityVersion;
16. soporte CredentialVersion;
17. soporte validity windows;
18. soporte method-specific restrictions;
19. soporte operation-specific eligibility;
20. soporte pre-auth checks;
21. soporte post-credential checks;
22. reduzca account enumeration;
23. soporte tenant-specific state;
24. soporte global state;
25. soporte effective-state composition;
26. soporte forced logout;
27. soporte account recovery effects;
28. soporte compromise handling;
29. invalide sesiones/tokens según policy;
30. soporte context recovery checks;
31. soporte commit-time revalidation;
32. prevenga TOCTOU;
33. soporte distributed invalidation;
34. sea observable;
35. sea auditable;
36. soporte concurrency;
37. sea seguro bajo FrankenPHP;
38. permita plugins;
39. no reemplace Authorization;
40. falle cerrado ante estado desconocido.

---

## 275. Regla arquitectónica final

VoltStack deberá preservar:

```text
IDENTITY
    answers:
        who exists?

IDENTITY SECURITY STATE
    answers:
        what is the current security condition?

AUTHENTICATION ELIGIBILITY
    answers:
        may this identity authenticate
        using this method,
        for this operation,
        in this context?

AUTHENTICATION
    answers:
        has control of sufficient proof been demonstrated?

AUTHORIZATION
    answers:
        what may the authenticated identity do?
```

La regla crítica será:

> **Una Identity válida y una Credential válida no son suficientes si el estado actual de seguridad de la Identity prohíbe o restringe la autenticación.**

Y, al mismo tiempo:

> **El estado de Authentication no deberá convertirse en un sustituto para reglas de negocio o permisos de Authorization.**

---

## 276. Próximo documento

El siguiente documento recomendado será:

```text
11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md
```

Su responsabilidad será definir en profundidad todo el subsistema de autenticación mediante contraseña:

```text
PasswordCredential
PasswordCredentialRecord
PasswordAuthenticator
PasswordVerifier
PasswordHasher
Argon2id
bcrypt compatibility
algorithm agility
cost parameters
rehash detection
transparent hash migration
password policy
password creation/change
password history
password expiration where required
compromised password detection
pepper strategy
password reset interaction
credential versioning
security version interaction
timing resistance
dummy verification
rate limiting
brute-force protection
credential lifecycle
storage
audit
observability
testing
performance
FrankenPHP safety
```

Este será el primer documento especializado en un mecanismo concreto de Authentication y establecerá cómo VoltStack ofrecerá la ergonomía de Laravel para passwords sin acoplar el sistema completo a las contraseñas.
