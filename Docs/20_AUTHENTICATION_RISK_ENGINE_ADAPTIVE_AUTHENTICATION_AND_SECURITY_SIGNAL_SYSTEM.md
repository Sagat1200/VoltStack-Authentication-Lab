# VoltStack Authentication System

## 20 — Authentication Risk Engine, Adaptive Authentication and Security Signal System

- **Archivo:** `20_AUTHENTICATION_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL_SYSTEM.md`
- **Sistema:** Authentication
- **Framework:** VoltStack
- **Módulo sugerido:** `Quantum/Auth`
- **Estado:** Especificación arquitectónica del subsistema de evaluación de riesgo, autenticación adaptativa y señales de seguridad.

**Depende especialmente de:**

- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `13_REMEMBER_ME_PERSISTENT_LOGIN_AND_LONG_LIVED_AUTHENTICATION_CREDENTIAL_SYSTEM.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION_SYSTEM.md`
- `15_MULTI_FACTOR_AUTHENTICATION_FACTOR_ORCHESTRATION_AND_STEP_UP_SYSTEM.md`
- `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION_SYSTEM.md`
- `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION_SYSTEM.md`
- `18_ACCOUNT_RECOVERY_PASSWORD_RESET_IDENTITY_RECOVERY_AND_CREDENTIAL_REESTABLISHMENT_SYSTEM.md`
- `19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`

---

## 1. Propósito

Este documento define el subsistema responsable de evaluar el contexto de una autenticación y determinar si, aun cuando las credenciales sean técnicamente válidas, existen señales suficientes para considerar el intento más riesgoso de lo normal.
El sistema deberá cubrir:

- Authentication Risk Assessment
- Adaptive Authentication
- Security Signals
- Risk Signals
- Signal Providers
- Risk Score
- Risk Levels
- Risk Policies
- Risk Thresholds
- Device Risk
- Network Risk
- Geographic Risk
- Behavioral Risk
- Credential Risk
- Session Risk
- Identity Risk
- Federated Risk
- Impossible Travel
- New Device Detection
- Known Device Recognition
- IP Reputation
- Network Reputation
- Credential Compromise Signals
- Authentication History
- Risk Decay
- Signal Confidence
- Step-Up Recommendations
- MFA Escalation
- Fresh Authentication
- Authentication Denial
- Security Holds
- Explainability
- False Positive Management
- Privacy Controls

La pregunta central será:
¿Debe VoltStack confiar en esta Authentication al nivel habitual, elevar requisitos, limitar el resultado o rechazarla debido al contexto observado?

## 2. Separación respecto a Abuse Protection

El documento 19 responde principalmente:

- ¿Está este actor abusando
- del sistema de Authentication?

Este documento responde:

- ¿Es esta Authentication
- suficientemente confiable
- dado su contexto?

## 3. Ejemplo

Credenciales:
Password correct
pero contexto:

- new device
- new country
- hosting provider IP

previous session 10 minutes ago
from a distant geography
El resultado no debería ser simplemente:

```text
PASSWORD_VALID
→ LOGIN
```

sino:

```text
PASSWORD_VALID
        +
RISK SIGNALS
        ↓
STEP_UP_REQUIRED
```

## 4. Principio fundamental

Risk Engine nunca validará una credential.

Solo consume señales y produce una evaluación contextual.

## 5. Segunda regla fundamental

Una Authentication válida criptográficamente puede seguir siendo insuficiente para el nivel de confianza requerido.

## 1. Arquitectura general

Authentication Attempt
│
▼
Authentication Context Inputs
│
├── Identity
├── Credential
├── Device
├── Network
├── Geography
├── Session History
├── Authentication History
├── Threat Intelligence
└── Abuse Signals
│
▼
Security Signal Providers
│
▼
SecuritySignalSet
│
▼
Risk Engine
│
├── Signal Normalization
├── Confidence Evaluation
├── Weighting
├── Correlation
├── Risk Rules
└── Risk Model
│
▼
AuthenticationRiskAssessment
│
├── LOW
├── MODERATE
├── HIGH
└── CRITICAL
│
▼
Adaptive Authentication Policy
│
├── ALLOW
├── ALLOW_WITH_MONITORING
├── REQUIRE_FRESH_AUTH
├── REQUIRE_STEP_UP
├── RESTRICT
├── HOLD
└── DENY

## 2. AuthenticationRiskEngine

Contrato conceptual:

```php
interface AuthenticationRiskEngineInterface
{
    public function assess(
        AuthenticationRiskContext $context
    ): AuthenticationRiskAssessment;
}
```

## 3. AuthenticationRiskContext

Conceptualmente:

```php
final readonly class AuthenticationRiskContext
{
    public function __construct(
        public AuthenticationAttemptId $attempt,
        public ?IdentityReference $identity,
        public AuthenticationMethod $method,
        public AuthenticationEvidenceSet $evidence,
        public NetworkContext $network,
        public DeviceContext $device,
        public ?SessionContext $session,
        public ?TenantReference $tenant,
        public AuthenticationPurpose $purpose,
        public \DateTimeImmutable $occurredAt,
    ) {}
}
```

## 4. Context no contiene secrets

Nunca:

- password
- OTP
- Bearer Token
- Recovery Token
- Private Key

## 5. Security Signals

Toda observación contextual deberá convertirse preferentemente en:
SecuritySignal

## 6. Signal model

final readonly class SecuritySignal
{
public function __construct(
public SecuritySignalType $type,
public SecuritySignalSeverity $severity,
public SignalConfidence $confidence,
public SignalSource $source,
public \DateTimeImmutable $observedAt,
public array $attributes = [],
) {}
}

## 7. Signal Types

Ejemplos:

- NEW_DEVICE
- KNOWN_DEVICE
- NEW_NETWORK
- KNOWN_NETWORK
- NEW_COUNTRY
- UNUSUAL_COUNTRY
- IMPOSSIBLE_TRAVEL
- ANONYMOUS_PROXY
- HOSTING_PROVIDER_NETWORK
- KNOWN_MALICIOUS_IP
- TOR_EXIT_NODE
- VPN_DETECTED
- DEVICE_RISK
- CREDENTIAL_COMPROMISE
- RECENT_PASSWORD_RESET
- RECENT_ACCOUNT_RECOVERY
- RECENT_MFA_RESET
- RECENT_PASSKEY_CHANGE
- RECENT_EMAIL_CHANGE
- SESSION_ANOMALY
- AUTHENTICATION_FAILURE_BURST
- CREDENTIAL_STUFFING_CONTEXT
- MFA_FATIGUE_CONTEXT
- UNUSUAL_LOGIN_TIME
- BEHAVIORAL_ANOMALY
- FEDERATED_ASSURANCE_LOW

## 8. Signal severity

Conceptualmente:

- INFO
- LOW
- MEDIUM
- HIGH
- CRITICAL

## 9. Confidence

Separado de severity.
Ejemplo:

```text
Signal:
    impossible travel
```

Severity:
HIGH

Confidence:
0.92

## 15. Por qué separar ambos

Una señal puede ser grave pero incierta.

```php
Ejemplo:
Possible compromised device
severity = HIGH
confidence = 0.35
```

No deberá tratarse igual que una señal altamente confirmada.

## 16. Signal Source

Ejemplos:

- LOCAL_HISTORY
- DEVICE_SYSTEM
- NETWORK_SYSTEM
- ABUSE_PROTECTION
- THREAT_INTELLIGENCE
- FEDERATION
- SESSION_SYSTEM
- CREDENTIAL_SYSTEM
- ADMINISTRATIVE
- EXTERNAL_PROVIDER

## 17. Signal Provider

Contrato:

```php
interface SecuritySignalProviderInterface
{
    public function collect(
        AuthenticationRiskContext $context
    ): SecuritySignalSet;
}
```

## 18. Providers built-in

Podrán existir:

- DeviceRiskSignalProvider
- NetworkRiskSignalProvider
- GeoRiskSignalProvider
- AuthenticationHistorySignalProvider
- CredentialRiskSignalProvider
- SessionRiskSignalProvider
- RecoveryRiskSignalProvider
- FederationRiskSignalProvider
- AbuseSignalProvider

## 19. Providers externos

Podrán integrar:

- IP reputation providers
- fraud detection platforms
- SIEM feeds
- threat intelligence
- device intelligence

sin acoplar Core a proveedores específicos.

## 20. Provider cost

Cada provider deberá declarar:

- cost
- latency profile
- cacheability
- required context

## 21. Cheap providers first

Ejemplo:

```text
local device history
    ↓
cached network reputation
    ↓
external fraud API
```

## 22. Short-circuit

Si una regla local produce:

- CRITICAL + confidence 1.0
- puede no ser necesario consultar providers caros.

## 23. Provider timeout

Todo provider externo tendrá:

- timeout
- bounded retry
- circuit breaker

## 24. Provider failure

No deberá interpretarse como:
RISK = LOW

## 25. Unknown risk

Debe existir semántica para:

- UNKNOWN
- o degradation policy.

## 26. RiskLevel

Valores conceptuales:

- LOW
- MODERATE
- HIGH
- CRITICAL
- UNKNOWN

## 27. RiskScore

Opcionalmente:
0..100

## 28. Score no será la única verdad

Evitar:

```php
if ($score > 70) deny();
como toda la arquitectura.
```

## 29. Razón

Diferentes señales tienen semánticas cualitativas.
Ejemplo:
known compromised credential
puede requerir acción fuerte aunque score agregado no supere cierto threshold.

## 30. RiskAssessment

Conceptualmente:

```php
final readonly class AuthenticationRiskAssessment
{
    public function __construct(
        public RiskLevel $level,
        public ?RiskScore $score,
        public SecuritySignalSet $signals,
        public RiskReasonSet $reasons,
        public RiskRecommendationSet $recommendations,
        public \DateTimeImmutable $assessedAt,
    ) {}
}
```

## 31. Recommendations

Podrán ser:

- ALLOW
- MONITOR
- REQUIRE_FRESH_AUTHENTICATION
- REQUIRE_STEP_UP
- REQUIRE_PHISHING_RESISTANT_FACTOR
- LIMIT_PERSISTENCE
- REVOKE_REMEMBER_ME
- REQUIRE_RECOVERY_REVIEW
- CREATE_SECURITY_HOLD
- DENY

## 32. Risk engine does not directly mutate state

No deberá hacer:
$user->disabled = true;

## 33. En su lugar

Produce:
SecurityActionRecommendation

## 34. Adaptive Authentication Policy

Componente:

```php
interface AdaptiveAuthenticationPolicyInterface
{
    public function decide(
        AuthenticationRiskAssessment $risk,
        AuthenticationContext $context
    ): AdaptiveAuthenticationDecision;
}
```

## 35. AdaptiveAuthenticationDecision

ALLOW
ALLOW_WITH_MONITORING
REQUIRE_FRESH_AUTHENTICATION
REQUIRE_STEP_UP
REQUIRE_SPECIFIC_FACTOR
LIMIT_PERSISTENT_LOGIN
RESTRICT_CONTEXT
SECURITY_HOLD
DENY

## 36. Adaptive Authentication

Significa:

```text
same credentials
+
different context
=

different authentication requirements
```

## 37. Ejemplo bajo riesgo

known device
known network
normal location
no recent security changes
Resultado:
ALLOW

## 38. Ejemplo riesgo moderado

new device
known country
correct password
Resultado posible:
REQUIRE_STEP_UP

## 39. Ejemplo riesgo alto

new device
new country
credential stuffing campaign
recent password reset
Resultado:
REQUIRE_PHISHING_RESISTANT_FACTOR

## 40. Ejemplo crítico

known compromised credential
malicious network
session hijack signal
Resultado posible:

- DENY
- +;
- SECURITY_HOLD_RECOMMENDED

## 41. Device Risk

Uno de los principales ejes.

## 42. Device Context

Podrá contener:

- device reference
- known/unknown state
- last seen
- trust state
- credential binding
- platform metadata
- browser family
- OS family

device attestation where available

## 43. Device identity caveat

Device fingerprinting no será root of trust.

## 44. Known device

Debe significar:

- previously observed or explicitly trusted
- según policy.

## 45. Trusted device != known device

Importante.

```text
KNOWN
    seen before

TRUSTED
    has explicit trusted-device credential
```

## 46. New Device Signal

NEW_DEVICE
puede incrementar riesgo.

## 47. New Device no es ataque

Puede ser legítimo.
Por eso normalmente:
step-up
es mejor que:
deny

## 48. Device age

Un device observado por primera vez hace minutos puede ser más riesgoso que uno usado regularmente durante meses.

## 49. Device confidence

Si identificación del device es heurística:
confidence < 1

## 50. Device change sensitivity

Diferencias menores como browser version update no deben crear automáticamente un nuevo device.

## 51. Device Identity Resolver

Podrá producir:
DeviceMatchAssessment

## 52. DeviceMatchAssessment

EXACT
PROBABLE
UNKNOWN
CONFLICTING

## 53. Network Risk

Incluye:

- IP
- network prefix
- ASN
- provider type
- proxy
- VPN
- Tor
- hosting provider
- reputation

## 54. VPN does not equal malicious

Muy importante.
Debe ser una señal contextual, no condena automática.

## 55. Tor

Igualmente.

- Dependiendo de application policy puede:
- increase risk
- require step-up
- sin necesariamente denegar.

## 56. Hosting Provider IP

Puede indicar:

- cloud VM
- bot infrastructure
- corporate egress
- legitimate VPN

Por tanto requiere contexto.

## 57. Malicious IP feed

Puede aportar señal más fuerte.

## 58. Network Reputation Cache

Debe soportar TTL.

## 59. External IP intelligence privacy

Debe ser configurable y documentar qué información se comparte.

## 60. Geographic Risk

Puede considerar:

- country
- region
- approximate coordinates

distance from previous event

## 61. Geo precision

No necesita precise GPS.
Generalmente:
coarse IP-derived location

## 62. Impossible Travel

Ejemplo:

```text
Login A:
Monterrey 10:00
```

Login B:
Tokyo 10:30
puede producir:
IMPOSSIBLE_TRAVEL

## 63. Problema

IP geolocation no es perfecta.

## 64. VPNs

Pueden producir falsos positives.

## 65. Impossible Travel model

Debe incluir:

- distance
- elapsed time
- location confidence
- network type
- known VPN behavior
- device correlation

## 66. Impossible travel does not always mean deny

Puede:
require step-up

## 67. Known same device

Si mismo device pasa de Mexico a Netherlands en minutos vía VPN, risk interpretation cambia.

## 68. Travel history

El sistema podrá mantener:
AuthenticationHistory

## 69. AuthenticationHistoryRecord

Podrá contener:

- timestamp
- device reference
- network summary
- geo summary
- authentication method
- assurance
- result
- risk level

## 70. Minimize PII

Historial deberá cumplir:

- retention
- privacy
- data minimization

## 71. Authentication History Store

Contrato:

```php
interface AuthenticationHistoryRepositoryInterface
{
    public function recentFor(
        IdentityReference $identity,
        AuthenticationHistoryQuery $query
    ): AuthenticationHistory;
}
```

## 72. History query bounds

No cargar años de eventos en cada login.

## 73. Precomputed summaries

Puede mantener:

- last successful auth
- known devices
- known networks
- usual countries
- usual hours

## 74. Behavioral Risk

Puede incluir:

- login time anomaly
- device usage anomaly
- network pattern anomaly
- authentication method anomaly

## 75. Behavioral signals need caution

Son proclives a false positives.

## 76. Behavior should rarely be sole denial factor

Preferible:

- step-up
- monitor

## 77. Unusual login time

Ejemplo:

- User normally logs in 08:00–18:00
- login at 03:00

Es una señal débil/moderada.

## 78. No opaque AI-only decisions

VoltStack deberá mantener explicabilidad básica.

## 79. Risk Explainability

Toda assessment deberá poder responder:

- What signals contributed?
- What policy triggered?

Why was step-up required?

## 80. RiskExplanation

Objeto conceptual:
RiskExplanation

## 81. Internal explanation

Puede incluir:

- new device
- new country
- recent recovery
- known malicious network

## 82. External explanation

Debe ser más genérica.
Ejemplo:
Additional verification is required because this sign-in looks unusual.

## 83. Do not leak security intelligence

No mostrar:
Your IP is on threat feed X with score 93.

## 84. Credential Risk

Cada credential puede aportar señales.

## 85. Password compromise

Si Password subsystem detecta:
known breached password
puede generar:
CREDENTIAL_COMPROMISE

## 86. Compromised API token

Token subsystem puede marcar:
COMPROMISED

## 87. Passkey counter anomaly

Puede generar señal:
PASSKEY_CLONE_SUSPECTED

## 88. Remember-Me replay

Documento 13 puede producir:
PERSISTENT_CREDENTIAL_REPLAY

## 89. Recovery completion

Documento 18 puede producir:
RECENT_ACCOUNT_RECOVERY

## 90. MFA reset

RECENT_MFA_RESET

## 91. Email/contact change

Puede producir:
RECENT_RECOVERY_CHANNEL_CHANGE

## 92. Risk from recent security events

Ejemplo:

- password reset 5 minutes ago
- +;
- new device
- +;
- new country

mayor que cada señal aislada.

## 93. Correlation Engine

Podrá existir:
RiskSignalCorrelator

## 94. Correlation rule

Ejemplo:

```text
NEW_DEVICE
AND
RECENT_ACCOUNT_RECOVERY
    ↓
HIGH_RISK_RECOVERY_DEVICE_CHANGE
```

## 95. Avoid double counting

Señales derivadas del mismo evento no deben inflar score artificialmente.

## 96. Signal lineage

Cada signal podrá conservar:

- source event reference
- parent signals
- si se necesita.

## 97. Session Risk

Una session existente también puede volverse sospechosa.

## 98. Session signals

IP changes abruptly
device mismatch
security version changed
impossible travel
stolen session suspicion
recent credential compromise

## 99. Continuous Risk Evaluation

VoltStack podrá soportar en el futuro evaluación durante una sesión.

## 100. Initial V1

Puede evaluar riesgo en:

- login
- session restoration
- step-up
- sensitive reauthentication

## 101. Session Revalidation

Puede producir:

- continue
- step-up
- terminate session

## 102. Session risk is not Authorization

Aunque un recurso sea permitido, Auth puede requerir reauthentication por riesgo.

## 103. Remember-Me Risk

Cuando una sesión se restaura desde Remember-Me:

- older credential
- new network
- new device

puede reducir assurance efectivo.

## 104. Persistent login policy

Puede decidir:

- Remember-Me accepted
- but step-up immediately required

## 105. Bearer Token Risk

Para API tokens:

- new network
- unexpected geography
- wrong workload pattern

token used from many ASNs
pueden generar risk.

## 106. Service token behavior

Para machine identities, señales pueden ser distintas.

## 107. Human vs machine risk profiles

No compartir un único modelo.

## 108. Human profile

Puede considerar:

- device
- geography
- time
- recent security events

## 109. Machine profile

Puede considerar:

- expected network
- service identity
- audience
- deployment environment
- certificate/workload identity
- request pattern

## 110. RiskProfile

Conceptualmente:

- HUMAN_INTERACTIVE
- MACHINE
- ADMINISTRATIVE
- RECOVERY
- FEDERATED
- HIGH_SECURITY

## 111. Risk Policy Resolver

Contrato:

```php
interface AuthenticationRiskPolicyResolverInterface
{
    public function resolve(
        AuthenticationRiskContext $context
    ): EffectiveRiskPolicy;
}
```

## 112. Policy hierarchy

Framework Security Floor
↓
Application Policy
↓
Firewall Policy
↓
Tenant Policy
↓
Identity Type Policy
↓
Authentication Purpose
↓
Risk Profile

## 113. Tenant risk policy

Tenant enterprise puede exigir:
new device always requires MFA

## 114. Admin firewall

Puede exigir:

```text
anonymous proxy
    → phishing-resistant step-up
```

## 115. Public consumer app

Puede ser más tolerante.

## 116. Risk Thresholds

Ejemplo:

- 0–29  LOW
- 30–59 MODERATE
- 60–79 HIGH
- 80–100 CRITICAL
- Solo ilustrativo.

## 117. Threshold policies

Deben ser configurables.

## 118. Rule overrides

Una regla crítica puede ignorar score.

```text
Ejemplo:
credential explicitly marked compromised
    → DENY
```

aunque score sea 45.

## 119. Risk Rules

Tipos:

- score rules
- signal rules
- correlation rules
- hard rules
- context rules

## 120. RiskRule interface

interface AuthenticationRiskRuleInterface
{
public function evaluate(
RiskEvaluationContext $context
): RiskRuleResult;
}

## 121. Rule Result

Puede aportar:

- risk contribution
- recommendation
- hard requirement
- explanation

## 122. Risk Model

Puede ser:

- rule-based
- weighted scoring
- hybrid
- external model

## 123. Recommended V1

deterministic rules
+
weighted signals
+
explicit hard overrides

## 124. Por qué

Es:

- auditable
- testable
- explainable
- predictable

## 125. Machine Learning future

Podrá integrarse como signal provider/model.
Pero no deberá ser requisito para Core.

## 126. ML score

Deberá entrar como:

- external/model-derived signal
- con confidence/version.

## 127. Model version

Obligatorio para auditabilidad.

## 128. Feature drift

Futuro observability concern.

## 129. No automatic trust in ML

El framework/application policy decide cómo usar ese score.

## 130. Risk Decay

Algunas señales pierden relevancia con el tiempo.

- Ejemplos:
- RECENT_PASSWORD_RESET
- RECENT_RECOVERY
- NEW_DEVICE

## 131. RiskDecayPolicy

Podrá definir:

- half-life
- fixed expiration
- state transition

## 132. Example

NEW_DEVICE:

- high relevance for 24h
- moderate for 7d
- then known device

## 133. Not all signals decay

Ejemplo:

- credential marked compromised
- no debe desaparecer con tiempo sin lifecycle action.

## 134. Signal lifecycle

Estados posibles:

- ACTIVE
- DECAYING
- EXPIRED
- RESOLVED

## 135. Persistent signals

Algunas señales pueden persistir en Identity Security State.

## 136. Transient signals

Otras viven solo durante request.

## 137. Signal Store

Podrá existir para señales durables.

## 138. RiskSignalRepository

Contrato opcional:

```php
interface RiskSignalRepositoryInterface
{
    public function activeFor(
        IdentityReference $identity
    ): SecuritySignalSet;
}
```

## 139. Do not persist every low-level signal

Puede generar:

- storage explosion
- privacy issues
- high cardinality

## 140. Persist only security-relevant events

Los detalles finos pueden ir a telemetry.

## 141. False Positive Management

Crítico para adaptive auth.

## 142. False positives can lock out legitimate users

Por eso risk actions deberán preferir:

- step-up
- fresh auth
- monitor

antes que deny, salvo señales fuertes.

## 143. Risk Action Severity

Orden conceptual:

- ALLOW
- MONITOR
- STEP_UP
- FRESH_AUTH
- RESTRICT
- HOLD
- DENY

## 144. Safety bias

Para señales inciertas:
step-up
es frecuentemente mejor que:
deny

## 145. User feedback

Si Step-Up se dispara repetidamente por device legítimo, el sistema podrá aprender:

- known device
- si policy lo permite.

## 146. Trusted feedback

Solo successful strong Authentication debe poder fortalecer device trust.

## 147. Failed attempts must not teach trusted state

Obvio pero normativo.

## 148. Known network learning

Puede actualizarse tras successful Authentication.

## 149. Poisoning defense

Un attacker con credential parcial no deberá poder contaminar historial confiable.

## 150. Learning threshold

Solo eventos con:

- sufficient assurance
- low/no critical risk

deben alimentar trusted history.

## 151. Authentication History Trust Level

Cada histórico puede registrar:
confidence / provenance

## 152. Adaptive MFA

El Risk Engine se integra con documento 15.

## 153. Example

Password valid
Risk = LOW
→ no additional factor

Password valid
Risk = MODERATE
→ TOTP or Passkey

Password valid
Risk = HIGH
→ Passkey required

Risk = CRITICAL
→ deny/recovery

## 154. Factor selection influenced by risk

Risk puede exigir property:

- phishing_resistant
- no factor concreto.

## 155. Better abstraction

Risk recommendation:
require phishing-resistant evidence
Step-Up Planner decide:

- Passkey
- Security Key
- trusted federated assurance

## 156. Risk cannot implement MFA

Separación estricta.

## 157. Fresh Authentication

Risk puede exigir:
fresh authentication <= 5 min

## 158. Recent session but new risk

Session activity no es suficiente.

## 159. Adaptive Persistent Login

Risk puede prohibir:

- Remember-Me issuance
- después de high-risk Authentication.

## 160. Example

Authentication succeeded
+
high-risk new device
↓
session allowed
Remember-Me denied

## 161. Adaptive Session Lifetime

Risk puede recomendar lifetime menor.

## 162. Example

LOW risk
8h session

HIGH risk
30m session
si Application Security Policy lo adopta.

## 163. Do not mutate arbitrary session policy silently

Debe existir:
RiskAdjustedSessionPolicy

## 164. Adaptive Token Issuance

Una high-risk session puede no poder crear:

- Personal Access Tokens
- long-lived credentials
- trusted devices
- sin fresh step-up.

## 165. Risk after Recovery

Critical.
Una Identity recién recuperada puede tener:
reduced trust window

## 166. Recent Recovery Signal

RECENT_ACCOUNT_RECOVERY

## 167. Cooldown

Puede influir durante horas/días.

## 168. Authorization may use recovery risk metadata

Auth expone:

- recentRecovery
- pero business policies viven en Authorization.

## 169. Federation Risk

Federated login puede aportar:

- issuer reputation
- federated assurance
- auth_time freshness
- tenant mapping
- provider security state

## 170. External IdP authentication method

Un acr débil puede elevar local risk.

## 171. Provider misconfiguration

Connection marked:

- DEGRADED
- puede elevar risk.

## 172. External provider outage

No necesariamente risk de Identity.
Es system availability.
No mezclar.

## 173. Security Signal Normalization

Providers pueden usar vocabularios distintos.
VoltStack deberá normalizar a un set común.

## 174. Raw Provider Signal

No deberá entrar directamente a policy.

## 175. SignalNormalizer

interface SecuritySignalNormalizerInterface
{
public function normalize(
ProviderSecuritySignal $signal
): SecuritySignal;
}

## 176. Confidence normalization

Debe mapear cuidadosamente escalas externas.

## 177. Provider trust level

Cada external provider puede tener:
trust weight

## 178. Do not multiply blindly

La matemática exacta deberá estar encapsulada en RiskModel.

## 179. Signal Deduplication

Múltiples providers pueden reportar el mismo hecho.

## 180. Example

IP_REPUTATION_BAD
from Provider A
from Provider B
No necesariamente debe contar doble.

## 181. Signal correlation groups

Puede agrupar por:

- network
- device
- credential
- history
- behavior

## 182. Security event correlation

Especialmente útil para:
new device + new country + recovery

## 183. Real-time requirements

Risk assessment debe cumplir latency budget.

## 184. Latency budget

Profiles:

- FAST_LOCAL
- STANDARD
- HIGH_SECURITY

## 185. FAST_LOCAL

Solo local/cache signals.

## 186. STANDARD

Puede incluir one external provider with strict timeout.

## 187. HIGH_SECURITY

Puede permitir más consultas antes de decisión.

## 188. External providers should be parallelizable

Cuando sea seguro.

## 189. Risk Engine Deadline

Debe respetar:
Authentication operation deadline

## 190. Timeout result

No colgar login indefinidamente.

## 191. Unknown due to timeout

Policy decide:

- step-up
- fail closed
- degraded local assessment

## 192. Adaptive policy under unknown

Ejemplo recomendado:

```text
UNKNOWN
    → REQUIRE_STEP_UP
para contexts sensibles.
```

## 193. Low-risk consumer login

Puede tolerar degraded local evaluation.

## 194. High-security admin login

Puede fail closed o exigir Passkey.

## 195. Privacy Boundaries

Risk systems pueden volverse invasivos.

- VoltStack deberá aplicar:
- data minimization
- purpose limitation
- retention
- pseudonymization
- tenant isolation

## 196. Device fingerprinting

Debe ser:

- optional
- transparent
- policy-driven

## 197. Behavioral profiling

También.

## 198. No hidden surveillance requirement

Core debe funcionar sin fingerprinting invasivo.

## 199. Pseudonymization

IP/device identifiers usados para historical correlation podrán pseudonimizarse donde sea compatible con security requirements.

## 200. Retention profiles

Ejemplo:

- raw network security data: short retention
- aggregated risk history: longer

## 201. Tenant isolation

Tenant A nunca debe acceder a risk history de Tenant B.

## 202. Global abuse vs tenant risk

Puede existir global threat intelligence, pero Identity risk debe respetar boundaries.

## 203. Explainability

Cada decision debe producir:
reason codes

## 204. RiskReasonCode

Ejemplos:

- NEW_DEVICE
- NEW_COUNTRY
- IMPOSSIBLE_TRAVEL
- KNOWN_MALICIOUS_NETWORK
- RECENT_RECOVERY
- CREDENTIAL_COMPROMISE
- HIGH_ABUSE_CONTEXT
- LOW_FEDERATED_ASSURANCE

## 205. No free-form reasons as core

Preferir enums/value objects.

## 206. Human-readable explanations

Se generan en adapter/UI layer.

## 207. AdaptiveAuthenticationDecision model

final readonly class AdaptiveAuthenticationDecision
{
public function __construct(
public AdaptiveAuthenticationAction $action,
public AuthenticationRequirementSet $requirements,
public RiskReasonSet $reasons,
public AuthenticationRiskAssessment $assessment,
) {}
}

## 208. Action vs Requirement

Ejemplo:

```php
action = REQUIRE_STEP_UP

requirements =
    phishing_resistant
    fresh <= 5m
```

## 209. Security Hold

Risk puede recomendar:
SECURITY_HOLD

## 210. Hold is durable state transition request

Debe ser procesado por Identity Security system.

## 211. SecurityHoldReason

Ejemplo:

- ACCOUNT_TAKEOVER_SUSPECTED
- CREDENTIAL_COMPROMISE
- SESSION_HIJACK_SUSPECTED

## 212. Hold behavior

Puede:

- block long-lived credential issuance
- require recovery
- terminate sessions

según Security State policy.

## 213. No arbitrary permanent disable

Risk Engine no debe convertirse en account deletion/disable engine.

## 214. Risk after successful Step-Up

Debe recalcularse.

## 215. Example

Antes:

- HIGH
- new device

Después de Passkey:

- MODERATE/LOW enough
- según policy.

## 216. Step-Up evidence reduces uncertainty

Sí, pero no elimina signals.

## 217. Example

Malicious IP + valid Passkey.

- La Passkey aumenta assurance, pero:
- KNOWN_MALICIOUS_NETWORK
- sigue siendo señal.

## 218. Authentication assurance and risk are separate axes

Muy importante.

- High Assurance
- High Risk
- es posible.

## 219. Matriz

RISK
LOW          HIGH
ASSURANCE
LOW        allow?       step-up/deny
HIGH       allow        monitor/restrict/deny

## 220. No single scalar trust

VoltStack deberá conservar:

- AuthenticationAssurance
- +;
- AuthenticationRiskAssessment
- por separado.

## 221. Effective Trust

Podrá derivarse para policy, pero no debe destruir las dos dimensiones.

## 222. Session snapshot

Podrá conservar:

- risk at authentication
- risk reasons summary
- assurance

## 223. Risk snapshot expiration

No debe considerarse válido para siempre.

## 224. Session Risk Re-evaluation

Puede dispararse por:

- new network
- sensitive route
- long session age
- security event

## 225. Event-driven risk invalidation

Ejemplo:

```text
Password compromised
    ↓
emit security event
    ↓
```

active sessions marked for revalidation

## 226. Risk cache

Algunas assessments podrán cachearse.

## 227. Cache key

Debe incluir suficiente contexto:

- Identity
- device
- network
- purpose
- policy version

## 228. Cache danger

No reutilizar low-risk result si:

- network changed
- security version changed
- recent recovery occurred

## 229. RiskAssessment TTL

Corto y profile-specific.

## 230. Risk cache invalidation

Por:

- SecurityVersion
- DeviceVersion
- PolicyVersion
- RiskSignalVersion
- cuando aplique.

## 231. RiskPolicyVersion

Debe poder auditarse qué policy produjo una decision.

## 232. ModelVersion

Igual si existe weighted/ML model.

## 233. Deterministic replay

Para incident analysis debería poder reconstruirse conceptualmente:

- signals
- policy version
- model version
- decision

## 234. Exact replay may be impossible

External provider state can change.
Pero audit metadata debe permitir explicar la decision histórica.

## 235. RiskAuditRecord

Podrá contener:

- attempt public reference
- risk level
- score
- reason codes
- policy version
- model version
- action
- timestamp
- sin secrets.

## 236. Audit events

AuthenticationRiskAssessed
AuthenticationStepUpRequiredByRisk
AuthenticationDeniedByRisk
AuthenticationSecurityHoldRequested
ImpossibleTravelDetected
NewDeviceRiskDetected
CredentialCompromiseRiskDetected
RiskPolicyChanged

## 237. High-volume risk assessments

No todos necesitan durable audit individual.

## 238. Audit selection

Ejemplo:

- HIGH/CRITICAL
- step-up
- deny
- hold
- sí pueden auditarse.

## 239. Metrics

auth_risk_assessment_total
auth_risk_low_total
auth_risk_moderate_total
auth_risk_high_total
auth_risk_critical_total
auth_risk_stepup_total
auth_risk_denied_total
auth_risk_provider_failure_total
auth_risk_evaluation_latency

## 240. Signal metrics

auth_risk_signal_total
auth_new_device_signal_total
auth_impossible_travel_signal_total
auth_malicious_network_signal_total

## 241. Metric cardinality

No usar:

- identity
- device
- IP
- email
- como labels.

## 242. Tracing

Spans:

- auth.risk.assess
- auth.risk.collect_signals
- auth.risk.device
- auth.risk.network
- auth.risk.history
- auth.risk.correlate
- auth.risk.policy
- auth.risk.decision

## 243. Provider spans

External calls:

- auth.risk.provider
- con provider name controlado.

## 244. Do not trace sensitive raw data

Especialmente:

- exact PII
- full IP if policy forbids
- tokens
- credentials

## 245. Risk history lifecycle

Puede actualizarse después de successful Authentication.

## 246. Successful high-assurance auth

Puede registrar:

- known device
- known network
- usual location

## 247. Do not learn from denied attempts

No.

## 248. Failed attempts

Pueden alimentar threat history, pero no trusted history.

## 249. History categories

Separar:

- TrustedAuthenticationHistory
- ThreatAuthenticationHistory
- conceptualmente.

## 250. Risk state contamination

No mezclar:
attacker used IP X
con:
user trusted IP X

## 251. Known network threshold

Solo tras successful trusted auth.

## 252. Device onboarding

Un new device puede pasar a known después de:
successful Passkey/MFA

## 253. Trusted Device issuance

Documento 15 gestiona credential de confianza.
Risk system solo puede recomendarlo/no permitirlo.

## 254. Risk-aware Remember-Me

Ejemplo:

```text
LOW risk + strong auth
    → remember-me allowed

HIGH risk
    → no persistent login
```

## 255. Risk-aware Recovery

Documento 18 puede usar risk assessment específico.

## 256. Recovery risk profile

Debe ser más conservador.

## 257. Example

Recovery email opened
from new device
new country
anonymous proxy
Puede requerir evidence adicional.

## 258. Risk-aware Federation

OIDC login desde high-risk network puede requerir local Passkey.

## 259. Risk-aware API tokens

Un service token utilizado desde expected private network:
LOW
mismo token desde residential proxy:
HIGH

## 260. Binding rules

Service profiles pueden tener hard expected network constraints.

## 261. Hard binding vs risk signal

Distinguir:
token binding requirement
de:
unexpected network signal
Si binding es obligatorio:

- authentication invalid
- no simple risk.

## 262. Important architecture rule

Risk no deberá reemplazar validaciones deterministas.

## 1. Examples of deterministic checks

token expired
wrong audience
wrong tenant
wrong RP ID
invalid signature
revoked credential
No son "risk".
Son Authentication failure.

## 2. Risk begins after/basic alongside deterministic validity

Puede evaluarse antes para resource protection, pero no convierte invalid credential en valid.

## 3. Hard Security Rules

Ejemplos:

- COMPROMISED_CREDENTIAL
- IDENTITY_SECURITY_HOLD
- REVOKED_DEVICE

pueden producir direct deny.

## 4. Risk Policy DSL conceptual

Risk::when(
Signal::newDevice()
->and(Signal::newCountry())
)
->require(
Assurance::phishingResistant()
);

## 5. Another example

Risk::when(
Signal::recentRecovery()
)
->for('24 hours')
->denyPersistentCredentialIssuance();

## 6. Configuration conceptual

'authentication' => [

'risk' => [

'enabled' => true,

'profile' => 'standard',

'providers' => [
'device',
'network',
'history',
'abuse',
],

'levels' => [
'low' => [0, 29],
'moderate' => [30, 59],
'high' => [60, 79],
'critical' => [80, 100],
],

],

];
Los thresholds son ilustrativos.

## 269. Adaptive policy conceptual

'adaptive_authentication' => [

'moderate' => [
'action' => 'step_up',
],

'high' => [
'action' => 'step_up',
'require' => [
'phishing_resistant',
],
],

'critical' => [
'action' => 'deny',
],

];

## 270. Per-firewall policy

'admin' => [
'risk_profile' => 'high_security',
],

'web' => [
'risk_profile' => 'standard',
],

## 271. Tenant policy

'tenant_risk' => [
'new_device_requires_mfa' => true,
'anonymous_proxy' => 'step_up',
],

## 272. Extensibility

VoltStack deberá permitir:

- custom SignalProvider
- custom RiskRule
- custom RiskModel
- custom Correlator
- custom RiskPolicy
- custom RiskHistoryStore
- custom ExternalRiskProvider

## 273. RiskModel interface

interface AuthenticationRiskModelInterface
{
public function evaluate(
SecuritySignalSet $signals,
EffectiveRiskPolicy $policy
): AuthenticationRiskAssessment;
}

## 274. SignalProviderRegistry

Permitirá registrar providers.

## 275. Provider priorities

priority
cost
required context

## 276. RuleRegistry

Igualmente.

## 277. Compiled Risk Policy

Para performance:

- EffectiveRiskPolicy
- podrá compilar rules estáticas.

## 278. Request hot path

Debe minimizar:

- allocations
- external calls
- database scans

## 279. History cache

Puede cachear summaries.

## 280. New device lookup

Debe ser O(1)/indexado cuando sea posible.

## 281. Geo calculations

Pueden usar summaries, no cargar history completo.

## 282. Impossible travel performance

Solo necesita últimas ubicaciones relevantes.

## 283. Identity with huge history

No debe degradar linealmente.

## 284. History pruning

Debe existir:
AuthenticationHistoryRetentionPolicy

## 285. Security-relevant aggregates

Pueden persistir más que raw history.

## 286. Risk Engine store failure

Debe tener policy.

## 287. History store unavailable

Opciones:

- DEGRADED
- UNKNOWN
- STEP_UP
- FAIL_CLOSED
- según profile.

## 288. External provider unavailable

Igual.

## 289. High-security profile

Puede preferir:

```text
unknown risk
    → strong step-up
```

## 290. Standard profile

Puede utilizar local assessment.

## 291. RiskPolicyFailureMode

ALLOW_WITH_DEGRADED_ASSESSMENT
REQUIRE_STEP_UP
FAIL_CLOSED

## 292. No silent fail-open

Nunca.

## 293. Distributed runtime

Risk history deberá ser shared/durable cuando necesite cross-node context.

## 294. Local caches

Sí podrán existir para:

- provider metadata
- network intelligence
- compiled policy

## 295. Current RiskContext

Siempre request-scoped.

## 296. FrankenPHP

Nunca:

```php
private ?Identity $currentIdentity;
private ?RiskAssessment $lastAssessment;
en singleton mutable.
```

## 297. Fiber safety

Cada concurrent Authentication tendrá su propio:
RiskEvaluationContext

## 298. External provider clients

Pueden compartirse si son stateless/thread-safe según runtime abstraction.

## 299. Cancellation

Si Authentication request se cancela, expensive risk calls deberán cancelarse cuando runtime lo soporte.

## 300. Risk evaluation deadlines

No exceder Authentication lifecycle deadline.

## 301. Testing — Risk Engine

Debe cubrir:

- no signals
- single signal
- multiple signals
- conflicting signals
- critical override
- unknown provider

## 302. Testing — New Device

known
unknown
probable match
first-ever login

## 303. Testing — Impossible Travel

normal travel
impossible distance/time
VPN scenario
low-confidence geo
same trusted device

## 304. Testing — Network Risk

known network
new network
hosting provider
Tor
VPN
malicious IP

## 305. Testing — Recent Recovery

recovery 5m ago
24h ago
expired risk window

## 306. Testing — Credential compromise

Debe producir hard/strong action según policy.

## 307. Testing — Adaptive MFA

low risk -> no step-up
moderate -> TOTP/passkey
high -> phishing-resistant
critical -> deny

## 308. Testing — Assurance independence

Verificar:

- high assurance + high risk
- no se convierta automáticamente en low risk.

## 309. Testing — Provider failure

network provider timeout
history unavailable
external API error

## 310. Testing — Fail modes

degraded local
step-up
fail closed

## 311. Testing — Multi-tenant

Risk history y policies no deben cruzarse.

## 312. Testing — Policy version

Reproduce expected decisions por version.

## 313. Testing — Signal deduplication

Múltiples providers mismos signals.

## 314. Testing — Correlation

new device
+
recent recovery
+
new country
produce expected composite risk.

## 315. Testing — Risk decay

Verificar:

- active
- decaying
- expired

## 316. Testing — History poisoning

Failed attempts no deben registrar trusted device/network.

## 317. Testing — Session risk

IP changes
device changes
credential compromised after session start

## 318. Testing — Remember-Me risk

Restored session desde unknown device.

## 319. Testing — API token risk

Expected service network vs unexpected network.

## 320. Testing — FrankenPHP

Requests concurrentes:

- Alice risk HIGH
- Bob risk LOW
- Machine risk MODERATE
- sin contaminación.

## 321. Testing — Performance

Medir:

- p50
- p95
- p99
- provider calls
- cache hit ratio
- history queries

## 322. Fuzz testing

Especialmente:

- external provider responses
- risk attributes
- signal payloads
- history serialization

## 323. Property-based testing

Útil para:

- risk score bounds
- signal deduplication
- decay
- policy monotonicity
- requirement composition

## 324. Security invariants — General

AUTH-RISK-01
Risk Engine never validates credentials.
AUTH-RISK-02
Deterministic authentication failures cannot be converted into valid Authentication by risk policy.
AUTH-RISK-03
Risk and Assurance remain separate dimensions.
AUTH-RISK-04
Risk decisions are derived from verified or explicitly classified contextual signals.
AUTH-RISK-05
Unknown provider state never silently becomes low risk.
AUTH-RISK-06
Critical state mutations are delegated to the appropriate security subsystem.

## 325. Security invariants — Signals

AUTH-RISK-SIGNAL-01
Signals include provenance.

- AUTH-RISK-SIGNAL-02
- Severity and confidence are distinct.
- AUTH-RISK-SIGNAL-03

Duplicate signals are not blindly double-counted.

- AUTH-RISK-SIGNAL-04
- Low-confidence behavioral signals cannot alone claim deterministic compromise.
- AUTH-RISK-SIGNAL-05

External provider signals are normalized before policy use.

## 326. Security invariants — Device

AUTH-RISK-DEVICE-01
Device fingerprints are not root-of-trust credentials.

- AUTH-RISK-DEVICE-02
- Known device and trusted device are distinct states.
- AUTH-RISK-DEVICE-03

Failed Authentication cannot establish a trusted device.
AUTH-RISK-DEVICE-04
Device learning requires successful sufficiently trusted Authentication.

## 327. Security invariants — Geography/Network

AUTH-RISK-NET-01
VPN/Tor/hosting networks are contextual signals, not universal proof of attack.
AUTH-RISK-NET-02
Geolocation uncertainty is represented in confidence.

- AUTH-RISK-NET-03
- Impossible travel incorporates time and location confidence.
- AUTH-RISK-NET-04

Raw client-controlled forwarding headers do not define network identity.

## 328. Security invariants — Adaptive Authentication

AUTH-RISK-ADAPT-01
Risk may increase Authentication requirements.

- AUTH-RISK-ADAPT-02
- Risk does not itself execute MFA or Step-Up.
- AUTH-RISK-ADAPT-03

Adaptive policies do not silently weaken hard security requirements.
AUTH-RISK-ADAPT-04
High-risk Authentication may restrict persistent credential issuance.
AUTH-RISK-ADAPT-05
Risk-triggered restrictions are explainable through reason codes.

## 329. Security invariants — Privacy

AUTH-RISK-PRIV-01
Risk collection follows data minimization.

- AUTH-RISK-PRIV-02
- Tenant risk history is isolated.
- AUTH-RISK-PRIV-03

Behavioral/device fingerprinting is optional and policy-driven.
AUTH-RISK-PRIV-04
Sensitive context is not exposed through metrics or public error messages.

## 330. Security invariants — Runtime

AUTH-RISK-RT-01
Current risk evaluation state is request-scoped.

- AUTH-RISK-RT-02
- Shared Risk services are stateless or immutable.
- AUTH-RISK-RT-03

No Identity/RiskAssessment state survives FrankenPHP request boundaries.
AUTH-RISK-RT-04
Concurrent fibers have isolated RiskEvaluationContexts.
AUTH-RISK-RT-05
Distributed history required for security decisions is authoritative across nodes.

## 331. Anti-pattern — correct password means low risk

Incorrecto.

## 332. Anti-pattern — VPN means deny

Demasiado simplista.

## 333. Anti-pattern — new device means compromise

No necesariamente.

## 334. Anti-pattern — score only

No reducir todo a un número.

## 335. Anti-pattern — assurance equals risk

No.

## 336. Anti-pattern — ML black box as sole decision

No como Core.

## 337. Anti-pattern — unavailable provider = low risk

Nunca.

## 338. Anti-pattern — every signal persisted forever

Privacy/storage problem.

## 339. Anti-pattern — failed login teaches known device

Nunca.

## 340. Anti-pattern — external signal directly mutates Identity

No.

## 341. Anti-pattern — arbitrary PII in metrics

No.

## 342. Anti-pattern — process-global last risk score

Especialmente crítico con FrankenPHP.

## 343. Anti-pattern — risk replaces deterministic validation

Nunca.

## 344. Componentes principales

AuthenticationRiskEngine
AuthenticationRiskContext
AuthenticationRiskAssessment
RiskLevel
RiskScore
RiskReason
RiskRecommendation

SecuritySignal
SecuritySignalSet
SecuritySignalType
SecuritySignalSeverity
SignalConfidence
SignalSource

SecuritySignalProvider
SecuritySignalNormalizer
RiskSignalCorrelator

## 345. Policy components

AuthenticationRiskPolicy
EffectiveRiskPolicy
AuthenticationRiskPolicyResolver
AdaptiveAuthenticationPolicy
AdaptiveAuthenticationDecision
AuthenticationRiskRule
RiskRuleRegistry

## 346. Model components

AuthenticationRiskModel
WeightedRiskModel
RuleBasedRiskModel
HybridRiskModel
RiskModelVersion

## 347. History components

AuthenticationHistoryRepository
AuthenticationHistoryRecord
AuthenticationHistorySummary
KnownDeviceHistory
KnownNetworkHistory
RiskSignalRepository
RiskDecayPolicy

## 348. Providers

DeviceRiskSignalProvider
NetworkRiskSignalProvider
GeoRiskSignalProvider
BehaviorRiskSignalProvider
CredentialRiskSignalProvider
SessionRiskSignalProvider
RecoveryRiskSignalProvider
FederationRiskSignalProvider
AbuseRiskSignalProvider
ExternalThreatIntelligenceProvider

## 349. Namespace sugerido

VoltStack\Quantum\Auth\Risk
VoltStack\Quantum\Auth\Risk\Contracts
VoltStack\Quantum\Auth\Risk\Signal
VoltStack\Quantum\Auth\Risk\Provider
VoltStack\Quantum\Auth\Risk\Model
VoltStack\Quantum\Auth\Risk\Policy
VoltStack\Quantum\Auth\Risk\History
VoltStack\Quantum\Auth\Risk\Adaptive
VoltStack\Quantum\Auth\Risk\Telemetry

## 350. Estructura sugerida

src/Quantum/Auth/Risk/
├── Contracts/
│   ├── AuthenticationRiskEngineInterface.php
│   ├── SecuritySignalProviderInterface.php
│   ├── AuthenticationRiskModelInterface.php
│   ├── AuthenticationRiskPolicyResolverInterface.php
│   └── AuthenticationHistoryRepositoryInterface.php
│
├── Signal/
│   ├── SecuritySignal.php
│   ├── SecuritySignalSet.php
│   ├── SecuritySignalType.php
│   ├── SecuritySignalSeverity.php
│   ├── SignalConfidence.php
│   ├── SignalSource.php
│   ├── SecuritySignalNormalizer.php
│   └── RiskSignalCorrelator.php
│
├── Provider/
│   ├── DeviceRiskSignalProvider.php
│   ├── NetworkRiskSignalProvider.php
│   ├── GeoRiskSignalProvider.php
│   ├── BehaviorRiskSignalProvider.php
│   ├── CredentialRiskSignalProvider.php
│   ├── SessionRiskSignalProvider.php
│   ├── RecoveryRiskSignalProvider.php
│   └── FederationRiskSignalProvider.php
│
├── Model/
│   ├── AuthenticationRiskAssessment.php
│   ├── RiskLevel.php
│   ├── RiskScore.php
│   ├── RuleBasedRiskModel.php
│   ├── WeightedRiskModel.php
│   └── HybridRiskModel.php
│
├── Policy/
│   ├── AuthenticationRiskPolicy.php
│   ├── EffectiveRiskPolicy.php
│   ├── AuthenticationRiskPolicyResolver.php
│   ├── AuthenticationRiskRule.php
│   └── RiskRuleRegistry.php
│
├── Adaptive/
│   ├── AdaptiveAuthenticationPolicy.php
│   ├── AdaptiveAuthenticationDecision.php
│   └── AdaptiveAuthenticationAction.php
│
├── History/
│   ├── AuthenticationHistoryRecord.php
│   ├── AuthenticationHistorySummary.php
│   ├── AuthenticationHistoryRepository.php
│   ├── KnownDeviceHistory.php
│   ├── KnownNetworkHistory.php
│   └── RiskDecayPolicy.php
│
├── AuthenticationRiskContext.php
└── AuthenticationRiskEngine.php

## 351. Flujo completo de login adaptativo

Login Request
↓
Abuse Protection
↓
Password Authenticator
↓
Password VALID
↓
AuthenticationEvidence
↓
Risk Context
↓
Signal Providers
├─ Device: NEW_DEVICE
├─ Network: NEW_NETWORK
├─ Geo: NORMAL
└─ History: NO_RECENT_RECOVERY
↓
Risk Engine
↓
Risk = MODERATE
↓
Adaptive Policy
↓
STEP_UP_REQUIRED
↓
Factor Orchestrator
↓
Passkey
↓
Passkey VALID
↓
Assurance increased
↓
Risk re-evaluated
↓
AuthenticationContext
↓
Session created

## 352. Flujo impossible travel

Previous:

```text
    Mexico
    22:00
```

Current:

```text
    Singapore
    22:30

        ↓

GeoRiskProvider
        ↓
distance + time
        ↓
IMPOSSIBLE_TRAVEL
confidence 0.87
        ↓
Risk HIGH
        ↓
require phishing-resistant factor
```

## 353. Flujo recent recovery

Account recovered 15m ago
↓
new device
↓
new network
↓
correct password
↓
Risk HIGH
↓
Passkey required
↓
Remember-Me issuance denied
↓
session lifetime restricted

## 354. Flujo compromised password

Password VALID
↓
Credential subsystem:

```text
    credential compromise signal
        ↓
Risk Engine
        ↓
CRITICAL
        ↓
DENY
        ↓
```

Recovery / Password Rotation Required

## 355. Flujo service token

Service Token valid
↓
expected network = 10.0.0.0/8

current network = public residential ASN
↓
MACHINE risk profile
↓
HIGH/CRITICAL
↓
deny or require stronger workload proof

## 356. Flujo known device

Password valid
known device
known network
usual time
same country
no recent security events
↓
LOW RISK
↓
ALLOW

## 357. Flujo high-assurance + high-risk

Valid Passkey
UV = true
phishing resistant
+
known malicious network
recent takeover recovery
↓
Assurance = HIGH
Risk = HIGH
↓
Policy:
monitor / restrict / deny persistent issuance
Este caso demuestra por qué:
Assurance != Risk

## 358. Arquitectura global

AUTHENTICATION CONTEXT INPUTS
│
┌─────────────────┼──────────────────┐
▼                 ▼                  ▼
DEVICE            NETWORK            HISTORY
│                 │                  │
├───────────┬─────┴─────┬────────────┤
▼           ▼           ▼            ▼
CREDENTIAL   SESSION      RECOVERY      ABUSE
│           │           │            │
└───────────┴──────┬────┴────────────┘
▼
SECURITY SIGNAL SET
│
▼
SIGNAL NORMALIZATION
│
▼
CORRELATION
│
▼
RISK MODEL
│
▼
AuthenticationRiskAssessment
│
┌──────┴──────┐
▼             ▼
RISK          ASSURANCE
│             │
└──────┬──────┘
▼
ADAPTIVE AUTHENTICATION POLICY
│
┌─────────────────┼──────────────────┐
▼                 ▼                  ▼
ALLOW            STEP-UP             DENY
│
▼
FACTOR ORCHESTRATOR
│
▼
NEW AUTHENTICATION EVIDENCE
│
▼
RE-EVALUATION

## 359. Decisiones arquitectónicas principales

VoltStack adoptará:

1. Risk and credential validity are separate concerns.
2. Risk and Authentication Assurance are separate dimensions.
3. Security Signals carry severity, confidence and provenance.
4. Adaptive Authentication consumes risk but does not implement factors itself.
5. Device/network/geographic data are contextual signals, not credentials.
6. Risk can raise requirements but cannot weaken deterministic security rules.
7. External providers are optional and normalized.
8. Risk decisions remain explainable.
9. Risk history is privacy- and retention-aware.
10. Successful trusted Authentication may build known-device/network history.
11. Failed attempts never build trusted history.
12. Risk-aware policy can restrict sessions and persistent credential issuance.
13. Risk state is request-safe under FrankenPHP and distributed runtimes.
14. Relación con Laravel y Symfony

Laravel y Symfony permiten construir componentes de seguridad, rate limiting, authentication events y custom logic alrededor del login, pero normalmente no ofrecen como núcleo un modelo completo de:

- Security Signals
- Risk Context
- Risk Assessment
- Adaptive Authentication
- Risk-triggered Step-Up

VoltStack añadirá esta capa de forma explícita.

```text
La composición quedará:
Authenticator
    ↓
Verified Authentication Evidence
    ↓
Authentication Assurance
    ↓
Risk Engine
    ↓
Adaptive Authentication Policy
    ↓
Optional Step-Up
    ↓
Authentication Context
```

Esto permitirá evolucionar desde un sistema tradicional:

```text
credentials correct?
    yes → login
```

hacia:

```text
credentials valid?
    ↓
```

how strong is the evidence?
↓
what risk exists right now?
↓
is more evidence needed?
↓
what type of evidence?
↓
final Authentication decision

## 15. Criterios de aceptación

El subsistema será considerado completo cuando:
16. soporte AuthenticationRiskContext;
17. soporte SecuritySignals;
18. soporte severity;
19. soporte confidence;
20. soporte provenance;
21. soporte Signal Providers;
22. soporte RiskAssessment;
23. soporte RiskLevel;
24. soporte optional RiskScore;
25. soporte Risk Rules;
26. soporte correlation;
27. soporte deduplication;
28. soporte Device Risk;
29. soporte Known/New Device;
30. soporte Network Risk;
31. soporte Geo Risk;
32. soporte Impossible Travel;
33. soporte Authentication History;
34. soporte Behavioral Signals;
35. soporte Credential Risk;
36. soporte Session Risk;
37. soporte Recovery Risk;
38. soporte Federation Risk;
39. integre Abuse Protection signals;
40. soporte Adaptive Authentication;
41. soporte risk-triggered Step-Up;
42. soporte risk-triggered Fresh Authentication;
43. soporte phishing-resistant requirement;
44. soporte restriction de Remember-Me;
45. soporte adaptive session policy;
46. soporte Security Hold recommendations;
47. mantenga Risk separado de Assurance;
48. soporte Risk Decay;
49. soporte history retention;
50. soporte false-positive management;
51. soporte explainability;
52. soporte privacy controls;
53. soporte external risk providers;
54. soporte caching seguro;
55. soporte provider failure policies;
56. soporte multi-tenancy;
57. soporte audit;
58. soporte metrics;
59. soporte tracing;
60. soporte extensibility;
61. sea seguro con FrankenPHP;
62. sea fiber-safe;
63. no sustituya deterministic credential validation.
64. Regla arquitectónica final

VoltStack deberá preservar:

```text
VALID AUTHENTICATION EVIDENCE
            │
            ▼
AUTHENTICATION ASSURANCE
            │
            ├──────────────────────┐
            │                      │
            ▼                      ▼
     SECURITY SIGNALS       CONTEXTUAL HISTORY
            │                      │
            └──────────┬───────────┘
                       ▼
                  RISK ENGINE
                       │
                       ▼
          AUTHENTICATION RISK ASSESSMENT
                       │
                       ▼
           ADAPTIVE AUTHENTICATION POLICY
                       │
        ┌──────────────┼───────────────┐
        ▼              ▼               ▼
      ALLOW          STEP-UP          DENY
                        │
                        ▼
                  MORE EVIDENCE
                        │
                        ▼
             FINAL AUTHENTICATION CONTEXT
```

La primera regla central será:
VoltStack no confundirá una credential válida con una Authentication confiable en cualquier contexto.

La segunda será:
Authentication Assurance responderá qué tan fuerte es la evidencia presentada; Authentication Risk responderá qué tan sospechoso es el contexto en el que esa evidencia está siendo utilizada.

La tercera:
Adaptive Authentication utilizará Risk para exigir evidencia adicional, reauthentication o restricciones, pero nunca para convertir una credential inválida en válida ni para debilitar requisitos de seguridad deterministas.

Siguiente documento recomendado
La secuencia natural continúa con:
`21_AUTHENTICATION_DEVICE_TRUST_DEVICE_IDENTITY_AND_TRUSTED_DEVICE_CREDENTIAL_SYSTEM.md`
Ese documento deberá formalizar la capa de dispositivo que hasta ahora hemos utilizado desde Remember-Me, MFA y Risk:

- Device Identity
- Device Reference
- Known Device
- Trusted Device
- Device Enrollment
- Device Trust
- Device Credential
- Device Binding
- Device Recognition
- Browser Device
- Native Device
- Managed Device
- Device Attestation
- Device Trust Levels
- Device Lifecycle
- Device Revocation
- Lost Device
- Compromised Device
- Trusted Device Cookies
- Cryptographic Device Credentials
- Device-bound Passkeys
- Device Metadata
- Device History
- Tenant Device Policies
- Risk Integration
- MFA Reduction Boundaries
- Remember-Me Integration
- Session Binding
- Device Management UI
- Privacy
- Audit
- Observability
- Distributed Runtime
- FrankenPHP Safety

Con esto quedará formalizada una separación importante:

```text
DEVICE RECOGNITION
    "¿hemos visto este dispositivo?"

DEVICE TRUST
    "¿tenemos evidencia para confiar en él?"

DEVICE RISK
    "¿qué tan sospechoso es en este contexto?"

AUTHENTICATION FACTOR
    "¿qué evidencia aporta para autenticar?"

SESSION
    "¿qué estado de Authentication mantiene?"
```

Ese documento evitará que conceptos como known device, trusted device, Remember-Me, Passkey, device fingerprint y MFA bypass terminen mezclándose en una sola abstracción.
