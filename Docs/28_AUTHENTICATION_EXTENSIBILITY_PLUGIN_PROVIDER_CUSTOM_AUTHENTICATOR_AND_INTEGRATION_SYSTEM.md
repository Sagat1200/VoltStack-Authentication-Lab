# VoltStack Authentication System

## 28 — Authentication Extensibility, Plugin, Provider, Custom Authenticator and Integration System

- **Archivo:** `28_AUTHENTICATION_EXTENSIBILITY_PLUGIN_PROVIDER_CUSTOM_AUTHENTICATOR_AND_INTEGRATION_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth` + Support/Extensions
- **Estado:** Especificación arquitectónica del subsistema de extensibilidad, plugins, providers, authenticators personalizados e integración de componentes externos con el Authentication Core.

---

## 1. Propósito

Este documento define cómo VoltStack permitirá extender el sistema Authentication sin:

- romper contratos internos
- debilitar invariantes de seguridad

acoplar Core a paquetes externos
crear estados globales inseguros
introducir ambigüedad en resolución
romper FrankenPHP
El sistema deberá permitir integrar:

- Custom Authenticators
- Custom Identity Providers
- Custom Credential Types
- Custom Password Hashers
- Custom Token Providers
- Custom MFA Factors
- Custom Passkey Adapters
- Custom Federation Providers
- Custom Recovery Methods
- Custom Risk Providers

Custom Abuse Protection Providers
Custom Device Trust Providers
Custom Session Stores
Custom Flow Stores
Custom Rate Limit Stores
Custom Audit Stores
Custom Observability Exporters
Authentication Plugins
Enterprise Extensions

## 2. Principio fundamental

VoltStack Authentication deberá ser extensible por contratos, no por modificación directa del Core.

## 3. Segunda regla fundamental

Una extensión podrá añadir capacidades, pero no podrá debilitar silenciosamente las garantías mínimas del framework.

## 4. Tercera regla fundamental

Toda extensión de Authentication será tratada como código sensible de seguridad y deberá declarar explícitamente qué capacidades introduce.

## 5. Arquitectura general

AUTHENTICATION CORE
│
▼
EXTENSION CONTRACTS
│
┌──────────────┼──────────────┐
▼              ▼              ▼
AUTHENTICATORS   PROVIDERS       STORES
│              │              │
├──────────────┼──────────────┤
▼              ▼              ▼
RISK            MFA           DEVICE
│              │              │
├──────────────┼──────────────┤
▼              ▼              ▼
FEDERATION       RECOVERY       OBSERVABILITY
│              │              │
└──────────────┼──────────────┘
▼
EXTENSION REGISTRY
│
▼
PLUGINS / PACKAGES

## 6. AuthenticationExtension

Abstracción principal:

```php
interface AuthenticationExtensionInterface
{
    public function register(
        AuthenticationExtensionRegistry $registry
    ): void;
}
```

## 7. Lifecycle de una extensión

DISCOVERED
↓
REGISTERED
↓
VALIDATED
↓
COMPILED
↓
BOOTED
↓
ACTIVE
Estados alternos:

- DISABLED
- INCOMPATIBLE
- FAILED

## 8. AuthenticationExtensionManifest

Cada extensión deberá declarar metadata.

```php
Conceptualmente:
final readonly class AuthenticationExtensionManifest
{
    public function __construct(
        public string $name,
        public string $version,
        public AuthenticationExtensionApiVersion $apiVersion,
        public AuthenticationExtensionCapabilitySet $capabilities,
        public AuthenticationExtensionDependencySet $dependencies,
    ) {}
}
```

## 9. Manifest mínimo

Deberá incluir:

- name
- version
- extension API version
- capabilities
- dependencies
- configuration schema

## 10. Extension API Version

VoltStack deberá separar:
Framework Version
de:
Authentication Extension API Version

## 11. Ejemplo

Framework:
VoltStack 2.4

Auth Extension API:
1.2
Un plugin puede soportar:
Auth API >= 1.0 < 2.0

## 12. Compatibilidad

Durante bootstrap deberá verificarse:

- framework compatibility
- Auth API compatibility
- dependency compatibility
- PHP compatibility
- required subsystem availability

## 13. Incompatibilidad

Debe producir:

- BOOTSTRAP ERROR
- si la extensión es requerida.

## 14. Extensión opcional

Puede quedar:

- DISABLED
- con diagnostic warning.

## 15. AuthenticationExtensionCapabilities

Ejemplos:

- AUTHENTICATOR
- IDENTITY_PROVIDER
- CREDENTIAL_TYPE
- PASSWORD_HASHER
- TOKEN_PROVIDER
- MFA_FACTOR
- FEDERATION_PROVIDER
- RECOVERY_PROVIDER
- RISK_PROVIDER
- ABUSE_PROVIDER
- DEVICE_PROVIDER
- SESSION_STORE
- FLOW_STORE
- RATE_LIMIT_STORE
- AUDIT_STORE
- EVENT_LISTENER
- SECURITY_HOOK
- COMPILER_PASS

## 16. Capability declaration

Una extensión deberá declarar solo lo que realmente necesita.

## 17. Least Privilege

Ejemplo:

- Un analytics plugin necesita:
- EVENT_LISTENER

No:

- AUTHENTICATOR
- SECURITY_HOOK
- FINALIZATION_VETO

## 18. Trusted Computing Base

Toda extensión que implemente:

- Authenticator
- Hasher
- Token Verifier
- MFA Factor
- Recovery Provider
- Security Hook

deberá considerarse parte del:
Trusted Computing Base

## 19. In-process plugins

VoltStack deberá documentar:
Un plugin PHP ejecutado dentro del proceso de Authentication tiene potencialmente acceso al mismo espacio de memoria de la aplicación y debe considerarse código de confianza.

## 1. Extension Registry

Componente:

```php
interface AuthenticationExtensionRegistryInterface
{
    public function register(
        AuthenticationExtensionInterface $extension
    ): void;
}
```

## 2. Registries especializados

Internamente deberá delegar a:

- AuthenticatorRegistry
- IdentityProviderRegistry
- CredentialTypeRegistry
- MfaFactorRegistry
- RiskProviderRegistry
- RecoveryProviderRegistry
- SessionStoreRegistry

## 3. No universal untyped registry

Evitar:

```php
$registry->add('whatever', $object);
Preferir APIs tipadas.
```

## 4. Custom Authenticator

Contrato base:

```php
interface AuthenticatorInterface
{
    public function supports(
        AuthenticationInput $input,
        AuthenticationContext $context
    ): bool;

    public function authenticate(
        AuthenticationInput $input,
        AuthenticationContext $context
    ): AuthenticationOutcome;
}
```

## 5. Responsabilidad de Authenticator

Puede:

```php
recognize supported input
extract credential representation
verify credential
produce Authentication Evidence
return normalized outcome
```

## 6. No deberá

create arbitrary Session
send email
authorize business resources
mutate current Tenant
perform unrelated redirects

## 7. Custom Authenticator Example

final class SmartCardAuthenticator implements AuthenticatorInterface
{
public function supports(
AuthenticationInput $input,
AuthenticationContext $context
): bool {
return $input instanceof SmartCardInput;
}

public function authenticate(
AuthenticationInput $input,
AuthenticationContext $context
): AuthenticationOutcome {
// verify credential
// produce evidence
}
}

## 8. Authenticator Metadata

Cada Authenticator deberá declarar:

- id
- credential types
- supported transports
- supported purposes
- assurance properties
- stateless/stateful compatibility
- priority

## 9. AuthenticatorDescriptor

final readonly class AuthenticatorDescriptor
{
public function __construct(
public AuthenticatorId $id,
public CredentialTypeSet $credentials,
public AuthenticationCapabilitySet $capabilities,
public int $priority,
) {}
}

## 10. Priority

Será configurada/compilada.
No deberá depender de orden accidental de Composer.

## 11. Authenticator Collision

Dos authenticators pueden soportar el mismo input.

- VoltStack deberá resolver mediante:
- priority
- explicit scope
- firewall binding
- credential type specificity

## 12. Ambiguity

Si no puede resolverse:
CONFIGURATION ERROR

## 13. Fallback Authenticator

Podrá existir, pero deberá ser explícito.

## 14. No silent fallback

Especialmente si cambia assurance.

## 15. Custom Credential Types

VoltStack deberá permitir nuevos:

- Credential
- sin modificar Core.

## 16. Credential Contract

interface AuthenticationCredentialInterface
{
public function type(): CredentialType;
}

## 17. Secret-bearing credentials

Deberán implementar marker:
SensitiveCredentialInterface

## 18. Serialization

Sensitive Credential no deberá serializarse accidentalmente.

## 19. Credential Extractor

Custom transports podrán registrar:
CredentialExtractor

## 20. Example

X-SmartCard-Assertion
↓
SmartCardCredentialExtractor
↓
SmartCardCredential

## 21. Extractor != Verifier

Separación importante.

## 22. Custom Identity Provider

Contrato:

```php
interface IdentityProviderInterface
{
    public function resolve(
        IdentityLookup $lookup,
        IdentityResolutionContext $context
    ): IdentityResolutionResult;
}
```

## 23. Provider capabilities

Puede declarar:

- BY_ID
- BY_EMAIL
- BY_USERNAME
- BY_EXTERNAL_ID
- TENANT_SCOPED
- FEDERATED_MAPPING

## 24. Provider Descriptor

Deberá ser compilable.

## 25. Provider Selection

Nunca seleccionar provider usando input arbitrario del usuario sin registry/policy.

## 26. Multi-tenant providers

Deberán declarar explícitamente:
TENANT_SCOPED

## 27. Cross-tenant lookup

Será una violación grave.

## 28. Identity Provider Error

Deberá producir resultados normalizados.
No vendor exceptions.

## 29. Custom Password Hasher

Contrato conceptual:

```php
interface PasswordHasherInterface
{
    public function hash(
        SensitivePassword $password
    ): PasswordHash;

    public function verify(
        SensitivePassword $password,
        PasswordHash $hash
    ): PasswordVerificationResult;

    public function needsRehash(
        PasswordHash $hash
    ): bool;
}
```

## 30. Hasher Certification

Un custom hasher deberá pasar contract tests.

## 31. Prohibited insecure defaults

VoltStack no deberá aceptar como hasher production-ready:

- MD5
- SHA1
- unsalted custom hashes

plain SHA-256 password hashing

## 32. Legacy Hash Migration

Un plugin sí puede implementar:

- LegacyPasswordHasher
- para migración.

## 33. Legacy boundary

Debe:

- verify legacy hash
- immediately migrate on success

never create new legacy hashes

## 34. Custom Token Provider

Podrá implementar:

- Opaque Tokens
- JWT
- PASETO-style integrations
- Enterprise Tokens
- Custom Service Credentials

## 35. TokenProvider Contract

interface AuthenticationTokenVerifierInterface
{
public function verify(
PresentedToken $token,
TokenVerificationContext $context
): TokenVerificationResult;
}

## 36. Token verifier responsibilities

Debe verificar, según tipo:

- integrity
- expiration
- issuer
- audience
- tenant
- revocation
- binding

## 37. Cannot outsource required validation accidentally

Un plugin no puede decir:

```text
signature valid → authenticated
ignorando audience/issuer requeridos.
```

## 38. Required Verification Properties

Framework podrá declarar:

- TokenVerificationRequirementSet
- que el provider debe satisfacer.

## 39. Custom MFA Factors

Contrato:

```php
interface AuthenticationFactorProviderInterface
{
    public function createChallenge(
        FactorChallengeContext $context
    ): AuthenticationFactorChallenge;

    public function verify(
        FactorEvidence $evidence,
        FactorVerificationContext $context
    ): FactorVerificationResult;
}
```

## 40. Factor Metadata

Debe declarar:

- factor category
- phishing resistance
- user verification
- device binding
- recoverability
- assurance contribution

## 41. Example factor

Hardware Smart Card
podría declarar:

```php
POSSESSION
phishing_resistant = true
hardware_bound = true
```

## 42. Framework no confiará en metadata arbitraria

Un factor custom con propiedades de alta seguridad debe pasar:

- registration policy
- explicit configuration
- conformance tests

## 43. Factor Capability Approval

Puede existir:
AuthenticationFactorCapabilityPolicy

## 44. Custom Passkey Adapters

VoltStack podrá permitir diferentes WebAuthn engines.

## 45. WebAuthn adapter boundary

Core define:

- challenge
- RP config
- expected origin
- credential result

Adapter realiza verificación protocolaria.

## 46. Adapter Output

Debe devolver:
normalized WebAuthnVerificationResult
47. No adapter-specific objects in Core
48. Custom Federation Providers

Podrán integrar:

- OIDC
- OAuth2 social login
- enterprise IdP

SAML via optional package
custom identity broker

## 49. FederationProvider Contract

interface FederatedAuthenticationProviderInterface
{
public function start(
FederatedAuthenticationRequest $request
): FederatedAuthenticationRedirect;

public function complete(
FederatedAuthenticationCallback $callback
): FederatedAuthenticationResult;
}

## 50. Security requirements

Provider deberá respetar:

- state
- nonce where required
- PKCE where required
- issuer validation
- audience validation
- redirect URI binding

## 51. Provider metadata

Deberá declarar:

- protocol
- assurance claims support
- logout support
- PKCE support
- discovery support

## 52. External Identity Mapping

Provider no deberá decidir arbitrariamente:
local user ID
sin pasar por:
FederatedIdentityMapper

## 53. Custom Recovery Methods

Podrán añadir:

- recovery code
- support-assisted recovery
- enterprise recovery
- hardware recovery credential

## 54. RecoveryProvider Contract

interface AuthenticationRecoveryProviderInterface
{
public function begin(
RecoveryRequest $request
): RecoveryChallenge;

public function verify(
RecoveryEvidence $evidence,
RecoveryContext $context
): RecoveryEvidenceResult;
}

## 55. Recovery providers high-trust

Deben considerarse parte crítica del Authentication Security Model.

## 56. No arbitrary weak recovery

Un plugin no deberá reducir:

- minimum recovery assurance
- establecido por framework/application.

## 57. Custom Risk Providers

Contrato:

```php
interface SecuritySignalProviderInterface
{
    public function collect(
        AuthenticationRiskContext $context
    ): SecuritySignalSet;
}
```

## 58. External Risk Providers

Podrán integrar:

- fraud engines
- IP intelligence
- device intelligence
- enterprise security feeds

## 59. Risk provider output

Debe normalizarse a:
SecuritySignal

## 60. No direct denial by generic provider

Preferible:

```text
Provider
    ↓
SecuritySignal
    ↓
Risk Policy
    ↓
Decision
```

## 61. Hard security provider

Si se necesita veto directo, deberá usar capability explícita:
SECURITY_ENFORCEMENT

## 62. Custom Abuse Providers

Podrán aportar:

- rate limit signals
- bot detection
- credential stuffing indicators
- automation reputation

## 63. Abuse Provider Contract

interface AuthenticationAbuseSignalProviderInterface
{
public function evaluate(
AuthenticationAbuseContext $context
): AuthenticationAbuseAssessment;
}

## 64. Local/External Separation

Un external anti-bot provider no deberá sustituir obligatoriamente rate limiting local.

## 65. Defense in Depth

Podrán coexistir:

```text
local rate limit

+

external bot score
+
risk policy
```

## 85. Custom Device Providers

Podrán aportar:

- device recognition
- device certificate
- MDM
- attestation
- cryptographic device identity

## 86. Device provider output

Debe mapearse a:

- DeviceEvidence
- DeviceTrustProperties
- SecuritySignals

## 87. No arbitrary trusted boolean

No:

```php
return ['trusted' => true];
Preferir evidencia tipada.
```

## 88. Custom Session Stores

Contrato:

```php
interface AuthenticationSessionStoreInterface
{
    public function load(
        SessionId $id
    ): ?AuthenticationSessionRecord;

    public function persist(
        AuthenticationSessionRecord $session
    ): void;

    public function revoke(
        SessionId $id
    ): void;
}
```

## 89. Session Store Capabilities

Podrá declarar:

- distributed
- atomic_rotation
- TTL
- revocation
- bulk_revocation
- tenant_partitioning

## 90. Capability Validation

Si policy requiere:
global logout
pero store no soporta enumeración/versioned revocation:

- configuration error
- o adapter strategy explícita.

## 91. Custom Flow Stores

Deberán soportar:

- load
- save
- consume
- expire
- atomic finalization

## 92. Atomic consume

Crítico para replay protection.

## 93. Flow Store Contract Tests

Obligatorios para custom stores.

## 94. Custom Rate Limit Stores

Contrato deberá permitir atomic operations.

## 95. Required properties

atomic increment
TTL/window behavior
bounded key lifecycle
distributed safety when advertised

## 96. In-memory store

Puede ser válido para:

- tests
- single-process development

pero no necesariamente producción multi-node.

## 97. Custom Audit Stores

Documento 24.

- Deberán respetar:
- append orientation
- tenant isolation
- retention semantics
- redaction

## 98. Custom Observability Exporters

Podrán integrar:

- OpenTelemetry
- Prometheus bridge
- Datadog
- SIEM
- custom audit pipeline

## 99. Observability extensions

No deberán recibir raw credentials.

## 100. Extension Configuration

Cada extensión deberá poseer namespace propio.

```php
Ejemplo:
'authentication' => [

    'extensions' => [

        'acme_smartcard' => [
            'enabled' => true,
            'driver' => 'pkcs11',
        ],

    ],

];
```

## 101. Configuration Schema Extension

Plugins podrán registrar:
AuthenticationConfigurationSchemaExtension

## 102. Schema namespace

No podrán registrar keys arbitrarias en raíz y colisionar con Core.

## 103. Example

Correcto:
authentication.extensions.acme_smartcard
No:

- authentication.password
- salvo Core-approved extension point.

## 104. Semantic Validation

Un plugin también deberá registrar validator.

## 105. Example

Smart Card extension requiere:

- certificate authority
- reader provider
- minimum key type

## 106. Compilation

La extensión deberá integrarse con documento 27.

## 107. AuthenticationExtensionCompiler

Podrá convertir manifest + config en:
CompiledAuthenticationExtension

## 108. Extension compile-time

Debe resolver:

- service references
- metadata
- capabilities
- dependencies
- registries
- configuration

## 109. Runtime

No debe volver a descubrir plugin metadata.

## 110. Extension Dependencies

Ejemplo:
EnterpriseRiskExtension
requires:

```text
    Risk subsystem >= 1
    Event API >= 1
```

## 111. Optional dependencies

También:
uses Device subsystem if available

## 112. Dependency Graph

Extensiones participan en:
AuthenticationDependencyGraph

## 113. Cycles

Ejemplo inválido:

- Plugin A requires Plugin B
- Plugin B requires Plugin A

si no pueden resolverse.

## 114. Feature Discovery

Core podrá consultar:

- does a phishing-resistant factor exist?
- is a distributed flow store available?

does federation provider support logout?
mediante:
CapabilityRegistry

## 115. No class-name introspection runtime

Preferir descriptors compilados.

## 116. AuthenticationCapabilityRegistry

interface AuthenticationCapabilityRegistryInterface
{
public function supports(
AuthenticationCapability $capability
): bool;
}

## 117. Capability descriptors

Pueden ser más ricos:

```text
PASSKEY:
    phishing resistant
    UV capable
    available on admin
```

## 118. Integration Adapter Pattern

VoltStack deberá favorecer adapters.

```text
Ejemplo:
ThirdPartyWebAuthnLibrary
        ↓
VoltStackWebAuthnAdapter
        ↓
Passkey subsystem
```

## 119. Anti-Corruption Layer

No exponer vendor model directamente al Core.

## 120. Example

No:

```php
AuthenticationContext::$credential
    = ThirdPartyVendorCredentialObject;
```

## 121. Correct

Vendor Object
↓
Adapter
↓
VoltStack Credential/Evidence

## 122. Vendor Upgrade Isolation

Esto permite actualizar biblioteca externa sin cambiar dominio.

## 123. Integration Failure Translation

Vendor exception:
AcmeTimeoutException
debe convertirse en:

- AuthenticationProviderException
- o normalized Error.

## 124. No vendor exception leakage

Reiteración.

## 125. Integration Timeouts

Adapters externos deberán definir:

- timeout
- retry
- circuit breaker

cuando realizan network I/O.

## 126. No extension controls global timeout arbitrarily

Debe respetar:
AuthenticationExecutionDeadline

## 127. Cancellation

Extensions que realizan I/O deberán respetar cancellation cuando runtime lo soporte.

## 128. Extension Security Floors

VoltStack deberá definir operaciones que third-party extensions no podrán desactivar mediante APIs normales.
Ejemplos:

- credential integrity validation
- security-state validation
- tenant isolation
- flow replay protection
- secret redaction
- session fixation protection

## 129. Security Monotonicity

Custom policies/hooks podrán endurecer.
No debilitar:
FrameworkSecurityFloor

## 130. Example

Framework requiere:
AAL2
Plugin declara:
AAL1 sufficient
Resultado:
AAL2

## 131. Custom Authenticator Assurance

Un Authenticator no decide unilateralmente:
I am AAL3

## 132. Assurance Capability Policy

El framework deberá aprobar el mapeo:

```text
Authenticator Evidence Properties
    ↓
Authentication Assurance
```

## 133. Example

SmartCard Authenticator produce:

- possession
- hardware-backed
- user verification

El Assurance subsystem evalúa.
134. Authenticator does not self-certify final assurance
135. Custom Risk Signal Severity

Provider puede proponer severity.
Risk Normalizer/Policy puede ajustar.

## 136. Custom Device Trust

Provider puede aportar evidence.
DeviceTrustEvaluator decide trust.

## 137. Custom Recovery

Provider verifica evidence.
Recovery Orchestrator decide si assurance suficiente.

## 138. Fundamental pattern

EXTENSION
provides evidence/capability

CORE POLICY
decides security meaning

## 139. Extension Conflict Resolution

Puede haber conflictos:

- same Authenticator ID
- same Credential Type
- same Provider Alias
- same Hook ID
- same Configuration namespace

## 140. Duplicate IDs

Deberán rechazarse por default.

## 141. Explicit Override

Solo mediante mecanismo explícito:

- replace
- decorate
- extend

## 142. No accidental service overwrite

## 143. Decoration

VoltStack podrá soportar:

- Authenticator Decorator
- Provider Decorator
- Risk Provider Decorator

## 144. Example

PasswordAuthenticator
↓
CorporatePasswordPolicyDecorator

## 145. Decorator rules

Debe declarar:

- target
- priority
- capabilities

## 146. Security Decorator

Puede endurecer checks.

## 147. Weakening decorator

No podrá desactivar security floor.

## 148. Extension Ordering

Orden deberá resolverse durante compile time.

## 149. Deterministic ordering

Misma configuración produce mismo orden.

## 150. Composer discovery order

No será security semantic.

## 151. Plugin Discovery

VoltStack podrá soportar:

- explicit registration
- Composer package metadata
- service provider registration

## 152. Production recommendation

Preferir discovery durante bootstrap/build.

## 153. Auto-discovery

Debe ser disableable.

## 154. Enterprise environments

Pueden exigir allowlist:
allowed authentication extensions

## 155. AuthenticationExtensionAllowlist

Puede validar:

- package
- vendor
- version

signature metadata if available

## 156. Package Trust

VoltStack no podrá garantizar que un Composer package sea seguro solo por instalarse.

## 157. Supply Chain

Authentication extensions deberán tratarse con revisión elevada.

## 158. Package provenance

Tooling podrá mostrar:

- package name
- version
- source
- capabilities
- Auth API version

## 159. auth:extensions

CLI futuro:
php volt auth:extensions

## 160. Output conceptual

acme/smartcard-auth
Version: 1.4
Capabilities:

```text
  AUTHENTICATOR
  MFA_FACTOR
```

Security Authority:
HIGH
Status:
ACTIVE

## 161. Security Authority Classification

Podrá clasificarse:

- LOW
- MEDIUM
- HIGH
- CRITICAL
- según capabilities.

## 162. Example

Analytics listener:
LOW
Custom Recovery Provider:
CRITICAL

## 163. Tooling warning

Instalar extensión critical deberá producir warning/documentation visible.

## 164. Extension Testing

Documento 26 será obligatorio.

## 165. Authenticator Contract Suite

Todo custom Authenticator deberá probar:

- unsupported input
- invalid credential
- valid credential
- error handling
- secret leakage
- outcome normalization

## 166. Identity Provider Contract Suite

Debe probar:

- unknown identity
- tenant isolation
- provider error
- canonical lookup

## 167. Session Store Contract Suite

Debe probar:

- persist/load
- expiration
- revocation
- atomicity
- tenant isolation

## 168. Flow Store Contract Suite

Debe probar:

- consume once
- replay
- TTL
- concurrent completion

## 169. Rate Limit Store Contract Suite

Debe probar atomicidad.

## 170. Risk Provider Contract Suite

Debe verificar:

- safe signal normalization
- timeout
- no mutation
- no secret leakage

## 171. MFA Provider Contract Suite

Debe verificar challenge binding y replay.

## 172. Federation Provider Contract Suite

Debe verificar:

- state
- nonce
- issuer
- audience
- callback replay
- según protocolo.

## 173. Recovery Provider Contract Suite

Debe probar single-use/replay/expiration.

## 174. Device Provider Contract Suite

Debe impedir convertir metadata no verificada en trust fuerte.

## 175. Extension Conformance Profile

Podrá existir:

- AUTH_EXTENSION_STANDARD_V1
- AUTH_EXTENSION_SECURITY_CRITICAL_V1

## 176. Certification

Un paquete puede declarar:

- VoltStack Auth Conformance:
- PASS

solo si ejecuta runner correspondiente.

## 177. Framework no debe confiar solo en self-declaration

Certificación es evidencia, no garantía absoluta.

## 178. Static Analysis

Plugins podrán analizarse para:

- forbidden dependencies
- unsafe globals
- raw credential logging
- mutable singleton state

## 179. Architectural Contract

Custom Authenticator no deberá depender directamente de:

- HTTP response
- Controller
- global session helper
- salvo adapters.

## 180. Long-running Runtime Safety

Toda extensión deberá ser compatible con FrankenPHP.

## 181. Shared Services

Podrán ser singleton si:

- stateless
- immutable
- thread/fiber safe

## 182. Forbidden shared mutable state

Nunca:

```php
class CustomAuthenticator
{
    private ?Identity $lastUser = null;
}
```

si el service es shared.

## 183. Request Context

Debe recibirse explícitamente.

## 184. Fiber Context

No usar static current request.

## 185. Extension Reset Interface

Si una extensión necesita estado request-local reutilizable:

```php
interface ResettableAuthenticationExtensionInterface
{
    public function reset(): void;
}
```

## 186. Preferencia

Mejor evitar necesidad de reset mediante diseño stateless.

## 187. Worker reset

Framework deberá invocar resetters registrados.

## 188. Exception-safe reset

Incluso tras failure.

## 189. Extension Memory Bounds

Caches locales de plugins deberán ser:

- bounded
- TTL-controlled
- observable

## 190. Unbounded cache

No aceptable bajo workers persistentes.

## 191. Tenant-aware Extensions

Una extensión puede depender de tenant.

## 192. Pattern correcto

public function authenticate(
AuthenticationInput $input,
AuthenticationContext $context
)
y usar:
$context->tenant

## 193. Pattern incorrecto

$this->currentTenant = Tenant::current();
en singleton mutable.

## 194. Tenant Extension Configuration

Puede resolverse mediante:
TenantAuthenticationExtensionConfigurationResolver

## 195. Base config + overlay

Igual que policies.

## 196. Tenant cannot enable forbidden extension

Platform security floor puede restringir.

## 197. Example

Tenant intenta habilitar:

- legacy weak authenticator
- pero platform lo prohíbe.

Resultado:
DENIED CONFIGURATION

## 198. Extension Feature Flags

Podrán controlarse.

## 199. Runtime feature toggle

Debe ser seguro.

## 200. Example

Deshabilitar temporalmente:

- social login provider
- durante incidente.

## 201. Emergency Disable

Security operations deberá poder:

- disable authenticator/provider
- sin despliegue completo cuando arquitectura lo permita.

## 202. Dynamic Extension State

Debe estar separado de immutable registration.

## 203. Example

Compiled registry:
GoogleOidcProvider exists
Runtime state:
ENABLED / DISABLED

## 204. ExtensionStateProvider

Contrato:

```php
interface AuthenticationExtensionStateProviderInterface
{
    public function state(
        AuthenticationExtensionId $extension
    ): AuthenticationExtensionState;
}
```

## 205. State cache

Versioned/short TTL.

## 206. Disabled Authenticator

No podrá seleccionarse.

## 207. Active Flow Impact

Si extension se deshabilita durante flow:

- restart
- alternative factor
- deny
- según policy.

## 208. No silent substitute

Reiteración.

## 209. Extension Health

Podrá declarar health check.

## 210. Examples

OIDC provider connectivity
HSM available
remote fraud API
device attestation endpoint

## 211. Health != Authentication decision

Health ayuda a operaciones.

## 212. AuthenticationExtensionHealth

HEALTHY
DEGRADED
UNAVAILABLE

## 213. Health provider

No deberá ejecutar network calls por cada request.
214. Cache/monitor separately
215. Extension Observability

Toda extensión crítica deberá poder exponer:

- metrics
- traces
- diagnostics
- health
- sin secrets.

## 216. Common telemetry contract

Podrá integrarse a documento 24.

## 217. Extension Event Namespace

Custom events deberán usar namespacing.
Ejemplo:
acme.smartcard.credential_verified

## 218. Event Collision

No permitido.

## 219. Public Extension Events

Si forman parte de plugin API, deberán versionarse.

## 220. Extension Hooks

Plugins podrán registrar hooks solo en puntos públicos.

## 221. No arbitrary middleware injection in Core pipeline

Preferir:
defined HookPoint

## 222. Why

Evita modificar ordering de seguridad accidentalmente.

## 223. Compiler Pass Extensions

Más poderosas.
Solo para plugins trusted.

## 224. Compiler Pass Capability

Debe declararse:
COMPILER_PASS

## 225. Extension Compiler Phases

Podrán existir:

- BEFORE_VALIDATION
- AFTER_VALIDATION
- BEFORE_FREEZE

pero con muy pocos puntos públicos.

## 226. Prefer structured registry APIs

En vez de arbitrary compiler mutations.

## 227. Extension Decoration Flow

Core Authenticator
│
▼
Registered Decorator A
│
▼
Registered Decorator B
│
▼
Compiled Authenticator Chain

## 228. Decorator priority

Deterministic.

## 229. Decorator introspection

CLI deberá mostrar chain.

## 230. Example

PasswordAuthenticator
decorated by:

```text
    CompromisedPasswordCheck
    CorporatePasswordRestriction
```

## 231. Authentication Integration Profiles

VoltStack podrá ofrecer perfiles simplificados.

- Ejemplo:
- social
- enterprise
- hardware
- legacy

## 232. Enterprise integration

Podría agrupar:

- OIDC
- managed device
- risk provider
- SIEM

## 233. Integration Bundle

No deberá ocultar qué capabilities instala.

## 234. Installation Diagnostics

Debe mostrar:

- new authenticators
- new providers
- new hooks
- new stores
- security authority

## 235. Migration Compatibility

Extensions reemplazadas deben poder migrar estado.

## 236. Example

Cambiar:

```text
SessionStore A
→ SessionStore B
```

puede requerir:

- session migration
- dual-read
- forced reauthentication

## 237. Framework no deberá asumir migración transparente

## 238. Credential Provider Migration

Ejemplo:

```text
legacy password hash plugin
→ native Argon2id
Debe permitir migration-on-login.
```

## 239. Extension Removal

Eliminar plugin puede afectar:

- active sessions
- registered credentials
- flows
- user factors

## 240. Removal Guard

Tooling deberá poder detectar:
extension still owns active credentials

## 241. Example

No eliminar Passkey provider adapter si existen credentials incompatibles con nuevo engine sin migration.

## 242. AuthenticationExtensionRemovalAnalyzer

Podrá generar:

- SAFE
- MIGRATION_REQUIRED
- REAUTHENTICATION_REQUIRED
- BLOCKED

## 243. Extension Ownership Metadata

Credenciales/records extensibles deberían registrar:

- provider type
- schema version

## 244. Avoid hard dependency on package class names

Prefer stable identifiers.

## 245. Extension Data Versioning

Plugin-owned records deberán tener:
schema version

## 246. Upgrade Migration

Debe ejecutarse fuera del hot path cuando sea posible.

## 247. No arbitrary migrations during login

Excepto controlled lazy credential migration.

## 248. Security Update

Si una extensión presenta vulnerabilidad crítica:

- Emergency Disable
- deberá ser posible para flows nuevos.

## 249. Existing Sessions

Policy podrá decidir:

```text
sessions created using vulnerable authenticator
    → revalidate/revoke
```

## 250. Authentication Provenance

Sessions deberán conservar sufficient provenance para esto.

## 251. Example

Authentication methods:

```text
    password
    plugin:acme-smartcard-v1
```

## 252. Provider version provenance

También puede ser útil.

## 253. Security Incident Compatibility

Documento 24 puede buscar sesiones afectadas por provider/version.

## 254. Testing — Extension Registration

Debe probar:

- valid manifest
- missing manifest
- duplicate ID
- incompatible API

## 255. Testing — Capability Enforcement

Plugin sin capability:

- FINALIZATION_VETO
- no puede registrar ese hook.

## 256. Testing — Authenticator Contract

Todos los resultados normalizados.

## 257. Testing — Secret Leakage

Custom Authenticator no debe filtrar credential.

## 258. Testing — Tenant Isolation

Custom provider Tenant A no devuelve Identity Tenant B.

## 259. Testing — Runtime State

Plugin singleton no conserva previous Identity.

## 260. Testing — FrankenPHP

Requests secuenciales/concurrentes.

## 261. Testing — Extension Disable

Disabled provider no se selecciona.

## 262. Testing — Active Flow Disable

Debe seguir policy definida.

## 263. Testing — Fallback

No puede reducir assurance.

## 264. Testing — Conflict Detection

Duplicate authenticator ID debe fallar.

## 265. Testing — Decoration

Orden correcto.

## 266. Testing — Compiler Integration

Registry compiled incluye plugin correcto.

## 267. Testing — Extension Removal

Analyzer detecta active dependencies.

## 268. Testing — Health

Health failure no produce low-risk success automáticamente.

## 269. Testing — Provider Timeout

Respetar Authentication deadline.

## 270. Testing — Contract Violation

Custom provider devuelve objeto inválido:
programming error

## 271. Testing — Conformance

Security-critical plugin deberá pasar profile requerido.

## 272. Testing — Version Compatibility

Plugin old API:

- rejected
- cuando ya no sea compatible.

## 273. Testing — Configuration Schema

Unknown/invalid keys.

## 274. Testing — Extension Configuration Secrets

Secrets no aparecen en compiled config dumps.

## 275. Testing — Dependency Cycle

Debe detectarse.

## 276. Testing — Memory Bounds

Custom provider cache no crece ilimitadamente.

## 277. Testing — Fault Injection

Plugin failure aislado según criticality.

## 278. Testing — Supply Chain Diagnostics

CLI muestra package/version/capabilities.

## 279. Security invariants — Extensions

AUTH-EXT-CORE-01
Authentication Core does not require direct modification to support supported extension types.
AUTH-EXT-CORE-02
Extensions interact with Core through explicit contracts.

- AUTH-EXT-CORE-03
- Extension registration is validated before runtime.
- AUTH-EXT-CORE-04

Extension identifiers are unique within their registry.
AUTH-EXT-CORE-05
Extension ordering is deterministic.

## 280. Security invariants — Capabilities

AUTH-EXT-CAP-01
Extensions declare security-relevant capabilities.

- AUTH-EXT-CAP-02
- Extensions cannot register privileged hooks without the required capability.
- AUTH-EXT-CAP-03

Generic extensions cannot weaken framework security floors.

- AUTH-EXT-CAP-04
- Security-critical extensions are identifiable in diagnostics.
- AUTH-EXT-CAP-05

In-process security extensions are treated as trusted code.

## 281. Security invariants — Authenticators

AUTH-EXT-AUTH-01
Custom Authenticators return normalized Authentication Outcomes.
AUTH-EXT-AUTH-02
Custom Authenticators do not establish sessions directly.

- AUTH-EXT-AUTH-03
- Custom Authenticators do not self-assign final Authentication Assurance.
- AUTH-EXT-AUTH-04

Unsupported inputs are rejected safely.
AUTH-EXT-AUTH-05
Raw credentials do not leak through extension outputs.

## 282. Security invariants — Providers

AUTH-EXT-PROV-01
Provider errors are normalized before crossing Core boundaries.
AUTH-EXT-PROV-02
Tenant-aware providers enforce Tenant boundaries.

- AUTH-EXT-PROV-03
- External provider metadata does not bypass local policy.
- AUTH-EXT-PROV-04

Provider timeouts respect Authentication execution deadlines.
AUTH-EXT-PROV-05
Provider failure does not silently become successful Authentication.

## 283. Security invariants — MFA/Recovery/Risk

AUTH-EXT-SEC-01
Custom MFA factors expose evidence properties rather than declaring arbitrary final assurance.
AUTH-EXT-SEC-02
Custom Recovery methods cannot weaken minimum Recovery Assurance.
AUTH-EXT-SEC-03
Risk Providers contribute signals; Risk Policy determines the Authentication decision.
AUTH-EXT-SEC-04
Device Providers contribute evidence; Device Trust Policy determines trust.
AUTH-EXT-SEC-05
Fallback never lowers mandatory security requirements.

## 284. Security invariants — Stores

AUTH-EXT-STORE-01
Custom Flow Stores support replay-safe consume semantics.

- AUTH-EXT-STORE-02
- Custom distributed Rate Limit Stores provide atomic operations where advertised.
- AUTH-EXT-STORE-03

Custom Session Stores preserve configured revocation semantics.
AUTH-EXT-STORE-04
Custom stores preserve Tenant isolation.
AUTH-EXT-STORE-05
Store capability deficiencies are detected during configuration validation.

## 285. Security invariants — Runtime

AUTH-EXT-RT-01
Extensions do not retain mutable current Identity state in shared services.
AUTH-EXT-RT-02
Extension request context is execution scoped.

- AUTH-EXT-RT-03
- FrankenPHP worker reuse does not leak extension state between requests.
- AUTH-EXT-RT-04

Extension-local caches are bounded.
AUTH-EXT-RT-05
Resettable extension state is reset even after exceptions.

## 286. Security invariants — Compilation

AUTH-EXT-COMP-01
Extension metadata is validated and compiled before production request handling.
AUTH-EXT-COMP-02
Production runtime registries are immutable after freeze.

- AUTH-EXT-COMP-03
- Extension configuration namespaces do not collide silently.
- AUTH-EXT-COMP-04

Extension dependency cycles are detected.
AUTH-EXT-COMP-05
Compiled extension metadata does not expose raw secrets.

## 287. Anti-pattern — Editing Authentication Core for every provider

No.

## 288. Anti-pattern — Plugin returns trusted=true

No como security authority.

## 289. Anti-pattern — Custom Authenticator creates Session directly

No.

## 290. Anti-pattern — Authenticator decides final AAL

No.

## 291. Anti-pattern — Provider exceptions leak to Core/HTTP

No.

## 292. Anti-pattern — Auto-discovered order decides priority

No.

## 293. Anti-pattern — Plugin can override security floor

Nunca.

## 294. Anti-pattern — Unknown plugin loaded automatically in production

No en entornos estrictos.

## 295. Anti-pattern — Arbitrary root configuration injection

No.

## 296. Anti-pattern — Plugin class name persisted as permanent domain ID

Evitar.

## 297. Anti-pattern — Unbounded extension cache

Especialmente bajo FrankenPHP.

## 298. Anti-pattern — Current Tenant stored in plugin singleton

Nunca.

## 299. Anti-pattern — Extension disabled but still selected from stale cache

No.

## 300. Anti-pattern — Removing extension without dependency analysis

No.

## 301. Anti-pattern — Security plugin considered safe because Composer installed successfully

No.

## 302. Componentes principales

AuthenticationExtension
AuthenticationExtensionManifest
AuthenticationExtensionRegistry
AuthenticationExtensionManager
AuthenticationExtensionId
AuthenticationExtensionApiVersion

AuthenticationExtensionCapability
AuthenticationExtensionCapabilitySet
AuthenticationCapabilityRegistry

AuthenticationExtensionDependency
AuthenticationExtensionDependencyGraph
AuthenticationExtensionState
AuthenticationExtensionHealth

## 303. Custom Authenticator components

AuthenticatorInterface
AuthenticatorDescriptor
AuthenticatorRegistry
AuthenticatorDecorator
AuthenticatorExtension

## 304. Provider components

IdentityProviderInterface
IdentityProviderDescriptor
AuthenticationTokenVerifier
AuthenticationFactorProvider
FederatedAuthenticationProvider
AuthenticationRecoveryProvider
SecuritySignalProvider
AuthenticationAbuseSignalProvider
DeviceEvidenceProvider

## 305. Store components

AuthenticationSessionStore
AuthenticationFlowStore
AuthenticationRateLimitStore
AuthenticationAuditStore

## 306. Integration components

AuthenticationIntegrationAdapter
AuthenticationProviderExceptionTranslator
AuthenticationExtensionCompiler
AuthenticationExtensionStateProvider
AuthenticationExtensionRemovalAnalyzer

## 307. Testing components

AuthenticatorContractTestSuite
IdentityProviderContractTestSuite
SessionStoreContractTestSuite
FlowStoreContractTestSuite
RateLimitStoreContractTestSuite
AuthenticationExtensionConformanceRunner

## 308. Namespace sugerido

VoltStack\Quantum\Auth\Extension
VoltStack\Quantum\Auth\Extension\Contracts
VoltStack\Quantum\Auth\Extension\Manifest
VoltStack\Quantum\Auth\Extension\Capability
VoltStack\Quantum\Auth\Extension\Registry
VoltStack\Quantum\Auth\Extension\Compiler
VoltStack\Quantum\Auth\Extension\Runtime
VoltStack\Quantum\Auth\Extension\Integration
VoltStack\Quantum\Auth\Extension\Diagnostics

VoltStack\Testing\Auth\Extension

## 309. Estructura sugerida

src/Quantum/Auth/Extension/
├── Contracts/
│   ├── AuthenticationExtensionInterface.php
│   ├── AuthenticationExtensionRegistryInterface.php
│   ├── AuthenticationCapabilityRegistryInterface.php
│   ├── AuthenticationExtensionStateProviderInterface.php
│   └── ResettableAuthenticationExtensionInterface.php
│
├── Manifest/
│   ├── AuthenticationExtensionManifest.php
│   ├── AuthenticationExtensionId.php
│   ├── AuthenticationExtensionApiVersion.php
│   └── AuthenticationExtensionDependency.php
│
├── Capability/
│   ├── AuthenticationExtensionCapability.php
│   ├── AuthenticationExtensionCapabilitySet.php
│   └── AuthenticationSecurityAuthority.php
│
├── Registry/
│   ├── AuthenticationExtensionRegistry.php
│   ├── AuthenticatorExtensionRegistry.php
│   ├── IdentityProviderExtensionRegistry.php
│   ├── AuthenticationFactorExtensionRegistry.php
│   └── AuthenticationStoreExtensionRegistry.php
│
├── Compiler/
│   ├── AuthenticationExtensionCompiler.php
│   ├── AuthenticationExtensionCompilerPass.php
│   └── CompiledAuthenticationExtension.php
│
├── Runtime/
│   ├── AuthenticationExtensionManager.php
│   ├── AuthenticationExtensionState.php
│   ├── AuthenticationExtensionStateProvider.php
│   └── AuthenticationExtensionHealth.php
│
├── Integration/
│   ├── AuthenticationIntegrationAdapter.php
│   ├── AuthenticationProviderExceptionTranslator.php
│   ├── AuthenticatorDecorator.php
│   └── AuthenticationExtensionRemovalAnalyzer.php
│
└── Diagnostics/
├── AuthenticationExtensionInspector.php
├── AuthenticationExtensionCompatibilityReport.php
└── AuthenticationExtensionDependencyReport.php
Y en Testing:

```text
src/Testing/Auth/Extension/
├── AuthenticatorContractTestSuite.php
├── IdentityProviderContractTestSuite.php
├── SessionStoreContractTestSuite.php
├── FlowStoreContractTestSuite.php
├── RateLimitStoreContractTestSuite.php
├── FactorProviderContractTestSuite.php
├── FederationProviderContractTestSuite.php
└── AuthenticationExtensionConformanceRunner.php
```

## 310. Registro conceptual de extensión

final class SmartCardAuthenticationExtension
implements AuthenticationExtensionInterface
{
public function register(
AuthenticationExtensionRegistry $registry
): void {
$registry->authenticators()->register(
SmartCardAuthenticator::class
);

$registry->factors()->register(
SmartCardFactorProvider::class
);
}
}

## 311. Manifest conceptual

return [

'name' => 'acme/smartcard-auth',

'version' => '1.4.0',

'auth_api' => '^1.0',

'capabilities' => [
'authenticator',
'mfa_factor',
],

];

## 312. Custom Authenticator registration

Auth::extend()
->authenticator(
'smartcard',
SmartCardAuthenticator::class
);
API de alto nivel conceptual.

## 313. Custom Provider registration

Auth::extend()
->identityProvider(
'enterprise-directory',
EnterpriseDirectoryProvider::class
);

## 314. Risk Provider

Auth::extend()
->riskProvider(
'acme-fraud',
AcmeFraudSignalProvider::class
);

## 315. Session Store

Auth::extend()
->sessionStore(
'custom-distributed',
CustomDistributedSessionStore::class
);

## 316. Production compilation

Estas APIs deberán ejecutarse durante:

- bootstrap
- y alimentar el compiler.
- No cada request.

## 317. Flujo Custom Authenticator

Request
↓
Compiled Authenticator Registry
↓
SmartCardAuthenticator selected
↓
SmartCard Credential verified
↓
Verified Authentication Evidence
↓
Assurance Engine
↓
Risk Engine
↓
Authentication Policy
↓
Session Finalization
El plugin no controla las últimas cuatro etapas unilateralmente.

## 318. Flujo External Risk Provider

Authentication Context
↓
Risk Provider Registry
↓
Acme Fraud Adapter
↓
External API
↓
Normalized Security Signals
↓
VoltStack Risk Model
↓
Adaptive Authentication Decision

## 319. Flujo Custom MFA Factor

MFA Planner
↓
Required property:

```text
phishing-resistant possession
    ↓
Capability Registry
    ↓
SmartCard Factor available
    ↓
Challenge
    ↓
Provider verifies evidence
    ↓
Verified Factor Evidence
    ↓
Assurance Engine
```

## 320. Flujo Extension Disable

Security Operations
↓
Disable Extension
↓
ExtensionStateVersion++
↓
Runtime registry sees DISABLED
↓
New Authentication Flow
↓
Extension excluded
Flows activos siguen policy explícita.

## 321. Flujo Extension Upgrade

Plugin v1
↓
compatibility check
↓
data migration
↓
Plugin v2 compiled
↓
registry freeze
↓
runtime activation

## 322. Flujo Extension Removal

Request remove plugin
↓
Removal Analyzer
↓
Check:

```text
  credentials
  sessions
  flows
  stored records
        ↓
SAFE?
 ┌──────┴──────┐
 ▼             ▼
YES            NO
 │             │
remove      migration /
            reauth required
```

## 323. Arquitectura global

VOLTSTACK AUTH CORE
│
▼
AUTH EXTENSION API
│
┌────────────────────────┼────────────────────────┐
▼                        ▼                        ▼
AUTHENTICATORS              PROVIDERS                 STORES
│                        │                        │
├─────────────┬──────────┼──────────┬────────────┤
▼             ▼          ▼          ▼            ▼
MFA           RISK     FEDERATION  DEVICE       SESSION
│             │          │          │            │
└─────────────┴──────────┼──────────┴────────────┘
▼
CAPABILITY REGISTRY
│
▼
AUTH COMPILER
│
▼
COMPILED EXTENSION GRAPH
│
▼
IMMUTABLE AUTH RUNTIME

## 324. Relación con Laravel

Laravel ofrece una experiencia muy flexible mediante:

```php
custom guards
custom user providers
Auth::extend()
Auth::provider()
service providers
drivers
contracts
```

Su principal ventaja es la ergonomía.
VoltStack deberá conservar una experiencia comparable.

## 325. Relación con Symfony

Symfony aporta una arquitectura particularmente potente mediante:

- custom authenticators
- user providers
- service tags
- security factories
- compiler passes
- event subscribers
- decorators
- dependency injection

VoltStack adoptará especialmente la idea de extensiones compiladas y contratos explícitos.

## 326. Diferenciador VoltStack

VoltStack combinará:

- Laravel-like extension ergonomics
- +;
- Symfony-like service compilation
- +;
- capability-based Authentication extensions
- +;
- formal security conformance
- +;
- FrankenPHP-safe plugin lifecycle

La extensión developer-friendly:

```php
Auth::extend()->authenticator(
    'smartcard',
    SmartCardAuthenticator::class
);
```

deberá convertirse durante bootstrap en:

```text
Validated Extension
        ↓
Capabilities Checked
        ↓
Contract Validated
        ↓
Compiled Authenticator Descriptor
        ↓
Immutable Registry Entry
```

## 327. Decisiones arquitectónicas principales

VoltStack adoptará:

## 1. Authentication is extended through explicit contracts

## 2. Every extension has a stable identifier and manifest

## 3. Authentication Extension API is versioned separately from the framework

## 4. Extensions declare capabilities

## 5. Security-critical capabilities are explicit

## 6. In-process security extensions are trusted code

## 7. Custom Authenticators produce Evidence/Outcomes but do not directly create Sessions

## 8. Custom Authenticators do not self-declare final Assurance

## 9. Providers produce normalized domain results

## 10. Risk Providers contribute signals; Core Risk Policy decides

## 11. Device Providers contribute evidence; Device Policy decides trust

## 12. Recovery Providers cannot weaken minimum Recovery Assurance

## 13. Extension dependencies and conflicts are validated at compile time

## 14. Extension configuration is namespaced and schema-validated

## 15. Vendor objects/exceptions do not cross Core boundaries

## 16. Extension registries are compiled and immutable in production

## 17. Runtime extension state may disable capabilities without mutating registration

## 18. Fallback cannot reduce required security

## 19. Security-critical extensions must pass conformance suites

## 20. Extensions must be safe for FrankenPHP, fibers and multi-tenancy

## 21. Criterios de aceptación

El subsistema será considerado completo cuando:
22. soporte AuthenticationExtensionInterface;
23. soporte Extension Manifest;
24. soporte stable Extension IDs;
25. soporte Auth Extension API versioning;
26. soporte dependency declarations;
27. soporte capability declarations;
28. soporte security authority classification;
29. soporte compatibility checks;
30. soporte custom Authenticators;
31. soporte Authenticator descriptors;
32. soporte custom Credential Types;
33. soporte Credential Extractors;
34. soporte custom Identity Providers;
35. soporte custom Password Hashers;
36. soporte custom Token Providers;
37. soporte custom MFA Factors;
38. soporte custom Passkey adapters;
39. soporte custom Federation Providers;
40. soporte custom Recovery Providers;
41. soporte custom Risk Providers;
42. soporte custom Abuse Providers;
43. soporte custom Device Providers;
44. soporte custom Session Stores;
45. soporte custom Flow Stores;
46. soporte custom Rate Limit Stores;
47. soporte custom Audit Stores;
48. soporte custom observability exporters;
49. soporte configuration schema extensions;
50. soporte semantic extension validation;
51. soporte extension compilation;
52. soporte extension dependency graph;
53. detecte dependency cycles;
54. detecte identifier conflicts;
55. soporte explicit decorators;
56. soporte deterministic priority;
57. soporte capability registry;
58. soporte feature discovery;
59. soporte plugin auto-discovery opcional;
60. soporte strict allowlisting;
61. soporte extension state enable/disable;
62. soporte emergency disable;
63. soporte extension health;
64. soporte removal analysis;
65. soporte extension-owned data versioning;
66. soporte contract tests;
67. soporte conformance profiles;
68. soporte security regression;
69. evite vendor object leakage;
70. evite vendor exception leakage;
71. preserve security floors;
72. preserve assurance requirements;
73. preserve tenant isolation;
74. preserve flow replay protection;
75. sea fiber-safe;
76. sea seguro bajo FrankenPHP;
77. mantenga caches bounded;
78. limpie request-local extension state;
79. sea compilable/immutable en producción.
80. Regla arquitectónica final

VoltStack deberá preservar:

```text
                THIRD-PARTY / APPLICATION EXTENSION
                              │
                              ▼
                    EXTENSION CONTRACT
                              │
                              ▼
                   MANIFEST + CAPABILITIES
                              │
                              ▼
                CONFIGURATION VALIDATION
                              │
                              ▼
                  SECURITY VALIDATION
                              │
                              ▼
                      COMPILATION
                              │
                              ▼
                  IMMUTABLE REGISTRY
                              │
                              ▼
                     RUNTIME USAGE
                              │
                              ▼
               EVIDENCE / SIGNAL / RESULT
                              │
                              ▼
                     VOLTSTACK POLICY
                              │
                              ▼
              AUTHENTICATION DECISION
```

La primera regla será:
Las extensiones aportarán mecanismos, evidencia, señales, stores e integraciones; el Authentication Core conservará la autoridad sobre la semántica final de seguridad.

La segunda:
Un Authenticator personalizado podrá verificar una credential, pero no podrá saltarse Identity Eligibility, Risk, Assurance, MFA, Session Finalization ni las políticas mínimas de VoltStack.

La tercera:
La extensibilidad no utilizará “trust me” como contrato: capabilities, configuración, compilación, tests y conformance deberán hacer explícito qué puede hacer cada integración.

La cuarta:
Los paquetes externos nunca deberán contaminar el modelo interno con excepciones, objetos o convenciones específicas de proveedores; adapters y resultados normalizados mantendrán una frontera estable.

La quinta:
Una extensión Authentication deberá ser segura no solo durante una petición PHP tradicional, sino también bajo FrankenPHP, concurrencia, fibers, multi-tenancy, rolling deployments y workers persistentes.

Siguiente documento recomendado
La secuencia natural continúa con:
`29_AUTHENTICATION_MULTI_TENANCY_SECURITY_REALMS_CROSS_TENANT_ISOLATION_AND_TENANT_AUTHENTICATION_POLICY_SYSTEM.md`
Aunque ya hemos incorporado multi-tenancy transversalmente, vale la pena formalizarlo como subsistema propio porque Authentication debe resolver con precisión:

- Tenant Resolution
- Authentication Tenant Context
- Tenant-specific Identity Providers
- Tenant-specific Login Methods
- Tenant Security Policies
- Tenant MFA Policies
- Tenant Federation
- Tenant OIDC Configuration
- Tenant Sessions
- Tenant Remember-Me
- Tenant Device Trust
- Tenant Risk History
- Tenant Recovery
- Tenant Rate Limits
- Tenant Audit
- Tenant Authentication Domains
- Security Realms
- Cross-Tenant Isolation
- Cross-Tenant Identity Collisions
- Tenant Switching
- Tenant-aware Authentication Flows

Shared Identity vs Tenant Identity
Global Identity Models
Tenant Membership
Tenant Suspension
Tenant-specific Credential Policy
Tenant-specific Extension Policy
Platform Security Floors
Tenant Policy Overrides
Distributed Tenant Authentication
FrankenPHP Tenant Context Isolation
Ese documento permitiría establecer de manera definitiva una regla crítica para VoltStack:

```text
GLOBAL IDENTITY
      ≠
TENANT MEMBERSHIP
      ≠
TENANT AUTHENTICATION CONTEXT
      ≠
SECURITY REALM
```

y evitaría uno de los tipos de vulnerabilidad más graves en sistemas SaaS: que una Authentication válida en un tenant, firewall o realm termine siendo aceptada accidentalmente en otro.
