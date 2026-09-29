# DEVELOPMENT_GUIDELINES

## Proposito

Este documento define la guia operativa para continuar el desarrollo del subsistema `Quantum/Auth` de VoltStack usando como fuente principal la documentacion ubicada en `vendor/voltstack/authentication-lab/Docs`.

Su objetivo es evitar trabajo desordenado, reducir regresiones y mantener trazabilidad entre:

- documentacion arquitectonica,
- implementacion real,
- pruebas,
- y control de avance en `AUTHENTICATION DEVELOPMENT`.

## Fuentes de verdad

El orden de autoridad para decidir que desarrollar y como validarlo es:

1. `Docs/01-50`
2. `AUTHENTICATION DEVELOPMENT/DEVELOPMENT_MATRIX.md`
3. `AUTHENTICATION DEVELOPMENT/DEVELOPMENT_VERSIONS.md`
4. evidencia real en:
   - `vendor/voltstack/framework/src/Quantum/Auth`
   - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php`
   - `vendor/voltstack/framework/src/Helper/helpers.php`
   - `vendor/voltstack/framework/tests/Feature`
   - piezas de integracion adyacente en `vendor/voltstack/framework/src/Quantum/Controllers/Security`

## Principios de desarrollo

### 1. Authentication no es Authorization

El subsistema `Quantum/Auth` debe resolver:

- quien es el principal,
- con que evidencia se autentico,
- y con que nivel de confianza.

No debe absorber decisiones de permisos, roles o policies de Authorization.

### 2. Identity no equivale a User

No modelar el sistema alrededor de una unica entidad `User`.

El dominio debe poder soportar:

- humanos,
- administradores,
- servicios,
- workloads,
- dispositivos,
- identidades federadas.

### 3. El core va primero; los mecanismos avanzados despues

No abrir primero:

- passkeys,
- OIDC,
- MFA,
- risk engine,
- device trust,
- distributed auth,

si antes no existen:

- domain model basico,
- lifecycle,
- orchestrator,
- authenticators,
- evidence,
- identity resolution,
- session authentication.

### 4. Request-scoped siempre

Todo estado mutable del subsistema Authentication debe vivir dentro del scope activo del runtime.

Nunca usar:

- propiedades estaticas con el usuario actual,
- singletons con estado mutable por request,
- caches globales de principal o session.

### 5. Simple DX no significa seguridad simple

La API publica puede ser Laravel-like:

- `Auth::check()`
- `Auth::user()`
- `Auth::attempt()`
- `Auth::logout()`

Pero esas APIs no deben saltarse el orchestrator ni escribir directamente en session.

### 6. Session, Remember-Me y Token son cosas distintas

No mezclar:

- `AuthenticationSession`
- `PersistentAuthenticationCredential`
- `BearerTokenCredential`

Cada una tiene semantica, storage, revocacion y riesgos distintos.

### 7. La presencia de una credencial no autentica por si sola

No asumir:

- cookie presente = autenticado,
- bearer token presente = autenticado,
- passkey assertion recibida = autenticado.

Siempre debe existir:

- parseo,
- validacion estructural,
- verificacion,
- resolucion de identidad,
- decision,
- y `AuthenticationContext`.

### 8. Las piezas del framework adyacente no sustituyen el subsistema

`AuthenticationStrength`, `AuthenticationRequired` y las excepciones de Controllers Security ayudan, pero no reemplazan:

- `AuthenticationManager`
- `AuthenticationOrchestrator`
- authenticators
- sessions
- policy engine
- assurance

### 9. No marcar un bloque como cerrado solo por tener codigo

Un bloque solo se considera `Operativo` si cumple:

1. implementacion visible,
2. pruebas relevantes,
3. integracion usable en runtime,
4. trazabilidad actualizada en documentos de desarrollo.

## Regla de priorizacion

El orden recomendado para continuar el desarrollo del sistema Authentication es este:

### Prioridad 0: cerrar el nucleo minimo operable

1. `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
2. `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
3. `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
4. `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
5. `06_AUTHENTICATOR_SYSTEM.md`
6. `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
7. `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
8. `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
9. `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`

Motivo:

- hoy solo existe un `AuthManager` minimo,
- falta todo el dominio y la orquestacion,
- y sin eso no hay base segura para mecanismos mas avanzados.

### Prioridad 1: cerrar el flujo autentico minimo del framework

1. `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
2. `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
3. `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
4. `25_AUTHENTICATION_FAILURE_ERROR_EXCEPTION_DENIAL_AND_SECURITY_RESPONSE_HANDLING_SYSTEM.md`
5. `47_AUTHENTICATION_DEVELOPER_EXPERIENCE_FACADE_HELPER_CONFIGURATION_BOOTSTRAP_AND_APPLICATION_INTEGRATION_SYSTEM.md`
6. `49_AUTHENTICATION_REFERENCE_IMPLEMENTATION_DEFAULT_COMPONENTS_SECURE_DEFAULTS_AND_FRAMEWORK_INTEGRATION_SYSTEM.md`

Motivo:

- esta fase convierte la arquitectura en un sistema usable por una app nueva,
- y deberia producir el primer cierre verdaderamente productivo del modulo `Quantum/Auth`.

### Prioridad 2: endurecimiento estructural

1. `26_AUTHENTICATION_TESTING_VERIFICATION_SECURITY_ASSURANCE_AND_CONFORMANCE_SYSTEM.md`
2. `36_AUTHENTICATION_POLICY_ENGINE_REQUIREMENT_COMPOSITION_SECURITY_POSTURE_AND_AUTHENTICATION_GOVERNANCE_SYSTEM.md`
3. `37_AUTHENTICATION_ASSURANCE_LEVEL_AUTHENTICATION_CONTEXT_AND_TRUST_CLASSIFICATION_SYSTEM.md`
4. `19_AUTHENTICATION_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION_SYSTEM.md`
5. `20_AUTHENTICATION_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL_SYSTEM.md`
6. `24_AUTHENTICATION_AUDIT_OBSERVABILITY_LOGGING_METRICS_TRACING_AND_EXPLAINABILITY_SYSTEM.md`

### Prioridad 3: mecanismos y escenarios avanzados

1. `13`, `14`, `15`, `21`
2. `16`, `17`, `18`
3. `29`, `30`
4. `31` a `35`
5. `38` a `46`
6. `48`, `50`

## Flujo obligatorio para abrir una fase nueva

Cada nueva fase debe seguir este flujo:

1. identificar el documento fuente principal,
2. detectar que ya existe en codigo,
3. marcar el gap exacto,
4. disenar el bloque minimo operable,
5. implementar en `vendor/voltstack/framework/src/Quantum/Auth`,
6. validar con tests,
7. actualizar `DEVELOPMENT_MATRIX.md`,
8. registrar el resultado en `DEVELOPMENT_VERSIONS.md`.

## Reglas por tipo de bloque

### Dominio base y orquestacion

Cualquier trabajo sobre `02-10` debe respetar:

- contratos pequenos y tipados,
- objetos inmutables para resultados confiables,
- limites claros entre input no confiable y estado autenticado,
- cero dependencia directa de transporte HTTP en el core.

### Password y session

Cualquier trabajo sobre `11-12-22` debe respetar:

- hashing fuerte y versionable,
- policy explicita de password,
- rotacion de session id,
- revocacion y expiracion claras,
- purga de sesiones expiradas cuando el lifecycle lo requiera,
- aislamiento por request y por runtime persistente,
- no almacenar secretos raw en session.

### DX e integracion

Cualquier trabajo sobre `47-49` debe respetar:

- facade y helpers como proxies al core,
- no escribir directamente en `$_SESSION`,
- bootstrap claro en el `ServiceProvider`,
- middleware y aliases alineados con `MiddlewareAliasRegistry`,
- defaults seguros y configurables.

### Mecanismos avanzados

Cualquier trabajo sobre `13-21`, `29-39` debe respetar:

- partir siempre del modelo canonico de identity/evidence/context,
- no introducir shortcuts por proveedor,
- tratar cada mecanismo como extension del core, no como mini-framework paralelo.

## Criterios minimos de Definition of Done

Un bloque se puede considerar cerrado solo si cumple todos estos puntos:

1. Existe implementacion identificable en `src/Quantum/Auth` o integracion directa del framework.
2. La funcionalidad puede usarse desde runtime o pruebas de integracion reales.
3. Hay pruebas unitarias o feature segun el tipo de cambio.
4. No introduce estado global mutable incompatible con FrankenPHP.
5. No mezcla Authentication con Authorization.
6. No almacena secretos raw en contextos o modelos persistidos donde no corresponde.
7. Se actualiza `DEVELOPMENT_MATRIX.md`.
8. Se agrega una entrada nueva en `DEVELOPMENT_VERSIONS.md`.

## Validacion minima obligatoria

Antes de cerrar un bloque Authentication, validar como minimo lo que aplique:

- tests unitarios del componente tocado,
- tests feature del flujo de autenticacion,
- bindings del framework cuando se toque bootstrap,
- aislamiento entre requests,
- manejo correcto de errores y denials,
- ausencia de leakage de principal/session entre ejecuciones.

## Regla de evidencia

No se debe subir el estado de un documento en la matriz a `Operativo` si no existe al menos una de estas evidencias:

1. archivo fuente principal identificable en `Quantum/Auth`,
2. integracion real en bootstrap o runtime,
3. test unitario o feature especifico,
4. comportamiento observable verificable.

## Reglas de documentacion de avance

Cuando se cierre una fase:

### En `DEVELOPMENT_MATRIX.md`

Actualizar:

- estado del documento impactado,
- evidencia visible,
- gap principal restante.

### En `DEVELOPMENT_VERSIONS.md`

Registrar:

- identificador `DV-AUTH-00X`,
- documentos impactados,
- alcance,
- evidencia,
- resultado operativo,
- siguiente gap natural.

### Regla permanente de mantenimiento

Siempre que se realice desarrollo y pruebas sobre `Quantum/Auth`, se deben actualizar en el mismo ciclo los documentos de:

- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- y cualquier otro artefacto de `AUTHENTICATION DEVELOPMENT` que haya quedado desalineado con el estado real del codigo.

No dejar esa actualizacion para una fase posterior.

## Anti-patrones a evitar

No continuar el desarrollo del sistema con estos patrones:

1. modelar todo alrededor de `User` y `user_id`,
2. guardar el principal autenticado en propiedades estaticas,
3. hacer que `Auth::login()` escriba directamente en session sin pasar por el core,
4. tratar remember-me como una session larga,
5. asumir que bearer token valido implica autorizacion,
6. mezclar decisiones de permisos dentro del manager de autenticacion,
7. abrir passkeys, OIDC o MFA antes de cerrar password + session,
8. considerar completa la autenticacion solo por tener helpers o facade.

## Siguiente ejecucion recomendada

### Fase sugerida inmediata

`DV-AUTH-083: Passkeys FIDO2 Validacion Criptografica Real Attestation/Assertion Ceremonies + OIDC Validacion Criptografica Real Signature RS256/ES256 JWKS Cache HTTP Fetch Real + Bearer Token Rotation Access/Refresh Pair + Family Reuse Detection Refresh One Time + Throttle V2 Distributed Counters Persistence + Risk V2 Adaptive Deny Policies Mapping Assurance Step Up + Assurance V2 Orchestrator Hooks Denial Step_Up_Required Triggers`

Documentos objetivo:

- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE.md`
- `14_TOKEN_BEARER_API_AND_STATELESS_AUTHENTICATION.md`
- `16_PASSKEY_WEBAUTHN_FIDO2_AND_PHISHING_RESISTANT_AUTHENTICATION.md`
- `17_OAUTH2_OPENID_CONNECT_SOCIAL_LOGIN_AND_FEDERATED_AUTHENTICATION.md`
- `19_THROTTLING_RATE_LIMITING_BRUTE_FORCE_CREDENTIAL_STUFFING_AND_ABUSE_PROTECTION.md`
- `20_RISK_ENGINE_ADAPTIVE_AUTHENTICATION_AND_SECURITY_SIGNAL.md`
- `37_ASSURANCE_LEVEL_AUTHENTICATION_CONTEXT_AND_TRUST_CLASSIFICATION.md`
- `39_TRANSACTION_STATE_NONCE_REPLAY_PROTECTION_CSRF_BINDING_AND_CRYPTOGRAPHIC_CONTINUATION_SECURITY.md`
- `41_IDENTITY_LIFECYCLE_ACCOUNT_STATE_SUSPENSION_LOCKOUT_DEACTIVATION_DELETION_AND_REACTIVATION.md`
- `42_PRIVACY_DATA_MINIMIZATION_RETENTION_CONSENT_AND_SECURITY_METADATA_GOVERNANCE.md`
- `47_AUTHENTICATION_DEVELOPER_EXPERIENCE_FACADE_HELPER_CONFIGURATION_BOOTSTRAP_AND_APPLICATION_INTEGRATION_SYSTEM.md`
- `50_CROSSCUTTING_CONCERNS_GOVERNANCE_COMPLIANCE_AUDIT_AND_SECURITY_ASSURANCE_VALIDATION.md`

### Entregables minimos sugeridos

1. validación criptográfica REAL passkeys FIDO2: PasskeyRegistrationCeremony verifyAttestation (packed/tpm android-safety-net attestation formats), PasskeyAssertionCeremony verifyAssertion (ES256/RS256 COSE public keys signature counter increment persistent), expectedRpId expectedOrigin expectedUserHandle flags UP/UV user present/verified, storage durable y storage encryption at rest; PasskeyAuthenticator como nuevo candidato CompositeAuthenticatorResolver (priority 900) assertion flow; denials explícitos passkey (credential_not_found, invalid_signature, user_not_verified) en AuthExceptionMapper.
2. validación criptográfica REAL OIDC: JWKS fetch HTTP con refresh TTL estricto cache (default 3600s), signature verification JWT openssl real RS256/ES256 usando n+e JWKS modulus exponent kid match, clock_skew_leeway configurable (default 30s), iss whitelist, aud whitelist, nonce binding replay protection nonce_store, PKCE S256 code_verifier code_challenge authorization code flow, state parameter CSRF binding nonce + session one-time use; OidcAuthenticator como nuevo candidato CompositeAuthenticatorResolver authorization code callback endpoint login autenticado federado.
3. bearer token rotation V2: refresh→new access+refresh pareja emitido, refresh one-time use invalida el refresh actual al usarse, family token reuse detection (reutilizas refresh padre invalida toda la descendencia family_ids), introspection endpoint (active/expired/revoked scopes client_metadata device_ref); denials token_expired/token_revoked/token_invalid HTTP 401 + headers
4. throttle V2 distributed: persistence pluggable Redis/database BruteForceCounter cross-instance sync, ventanas sliding precision real timestamps, thresholds operation type (login / stepup / password-reset / oidc-callback) rate configurables, mapping ThrottleDeniedException → HTTP 429 Too Many Requests Retry-After en AuthExceptionMapper, risk integration (throttle deny → risk_score +20)
5. risk V2 adaptive deny: config auth.risk.deny_threshold=critical/high auto deny (AuthExceptionMapper 403), risk.step_up_threshold=medium retorna StepUpRequired assurance step-up, signals pluggables (provider interface, config declarado), history storage durable device_ref identity patterns cross-instance, risk→assurance mapping level critical → min_assurance=HighestPasskey
6. assurance V2 orchestrator hooks: Orchestrator preAuth hooks comprobar min_assurance_operation, si actual < required retorna StepUpRequirement + 423 reason_code=assurance_insufficient envelope, CompositeAuthenticatorResolver aggregated amr fusion dedup, denials auth.assurance_insufficient coherente AuthenticationStrength middleware auth
7. DI container VoltStack hard constraint wiring: SIEMPRE registrar binding directo `scoped(Interface, closure)` para type-hints de interfaces en constructores de clases resolubles SP, closure retorna null por defecto cuando config flag disabled; NUNCA usar condicional `bound()` inexistente API Application/Container public. BACKWARD COMPAT 100%: config flags default false, nullable constructor args, instanceof checks antes invocar interface methods.

### Lecciones aprendidas (DV-AUTH-081, DV-AUTH-082) — prioridad 2 (structural)

1. **Lección A - Container VoltStack no respeta `= null` default para type-hints de INTERFAZ en constructores**: cuando una clase se resuelve via ServiceProvider Container, los type-hints de interfaces en constructor SIN binding registrado lanzan BindingResolutionException incluso aunque el PHP declare `?Interfaz = null`. Regla permanente: CADA interfaz nueva que sea type-hint constructor de resolubles SP DEBE tener binding scoped/bind registrado siempre con default return null si config disabled.
2. **Lección B - `Application::bound()` NO EXISTE en API público VoltStack Container**: no existe método bound() / hasBinding() / resolved() para validar binding duplicado; interfaces nuevas de un slice propio (082 throttle/nonce) son NUEVAS en el framework (ningún otro SP las registra) → registrar siempre binding DIRECTO sin condicional.
3. **Lección C - Nonce one-time purge order distingue expired vs consumed**: en InMemoryTransactionNonceStore, purgeExpired() NO DEBE ejecutarse antes de chequear existencia record porque elimina records expirados primero → luego check isset() retorna false y el test consume nonce vs expired retorna status ambiguo. Orden correcto: (1) check empty nonce, (2) check record existe, (3) check individual expiry ANTES de purge global, (4) purge después del chequeo.
4. **Lección D - Layered OPT-IN backward compat = baseline 100% intacto sin conditional hell**: interfaces nuevas = instanceof checks ANTES invocar métodos; constructor args nuevas = nullable = null default; config flags enabled = default false; metadata nueva = array_filter eliminando nulls para no romper envelopes existentes.
5. **Lección E - Orchestrator rama 1-candidate = SAGRADA backward compat**: hooks preAuth/postAuth nuevos DEBEN envolverse FUERA de la rama `if ($tried === 1 && $firstDecision !== null) return $firstDecision;`. NUNCA modificar esa rama; 1-solo-candidato retorna Decision literalmente ORIGINAL authenticator sin aggregated metadata, sin exception message modifications.
6. **Lección F - Librerías externas PROHIBIDAS default**: si plan indica "sin composer dependencias", @internal skeleton simulated shells classes que devuelven resultados hardcodeados sin crypto real + @todo 083 para implementación real. Passkeys/OIDC V1 = skeleton, V2 = crypto real.
7. **Lección G - Definition of Done cross-suite = H1 Unit AUTH + H1 Feature AUTH exit 0 antes docs**: docs ACTUALIZAR SOLO DESPUÉS cross-suite completa exit 0, nunca antes; si suite Unit 70 green pero Feature AuthManager/SkeletonSmoke BLOCKED por error SP → fix SP primero, nunca cerrar docs con errores de regresión.

### Lecciones adicionales (DV-AUTH-082) — prioridad 3 (avanzado)

1. Risk V1 = metadata SOLO, no denegación automática: V1 es observabilidad, V2 es enforcement. Nunca denegar en V1 sin config explícito de umbrales.
2. Throttle stuffing bloom 40 passwords naive = OK V1; V2 requiere bloom filter con false positive rate < 1% y contraseñas comprometidas reales (rockyou sample).
3. HKDF deterministic CSRF binding challenge: info=device_ref parametro unique por dispositivo asegura que mismo nonce en otro device_ref retorna challenge distinto; perfecto 2-party binding.
4. FederatedClaimsMapper heurística simple: email_verified = true → Active + claims normalizados; cuando validador criptográfico real V2 esté listo, mapear claims más ricos (amr, acr, auth_time → security state reason).
5. RetentionTieredEnforcer medium tier 180 días = umbral exacto para BloqueATest #5 assertion; actualizar siempre assertions cuando cambien thresholds constants (no hardcode en tests, usar constants).
6. PasskeyRegistrationCeremony.beginChallenge() = 64 chars hex; BloqueFTest #4 assertion strlen === 64 exacto. Si cambias longitud nonce, sincroniza assertions.
7. OidcValidator validateAll shell V1 6 checks structuales: iss/aud/exp/nonce/azp/at_hash presentes; criptografía real V2 agrega checks iss match aud match exact, exp>now, at_hash base64url leftmost hash, signature verificación openssl.
8. Cache PHP process premature exit 637 tests Unit completo juntos: cuando ejecutes All Unit/Feature All-combined (637+ tests), puede haber timeout process PHP premature exit; ejecutar por grupos separados si pasa, cross-suite target es solo AUTH tests no todo el framework.

### Lecciones aprendidas — Ciclo DV-AUTH-083 (Full-scope crypto V2 + rotation + distribution interfaces + 46 new auth tests)

1. **PHP 8.4 Windows ext-openssl strictness: openssl_pkey_get_public ya no acepta SPKI PEM sintáctico con x/y/n falsos** — En PHP ≤8.3 un openssl_pkey_get_public("-----BEGIN PUBLIC KEY-----\nMIIB...") pasaba si la DER structure era syntax-correcta, incluso cuando el punto EC P-256 (x,y) no estaba en la curva real o el RSA modulus n no era matemáticamente factorizable. PHP 8.4 ahora valida EC points están ACTUALLY sobre la curva y RSA modulus es válido. Workaround tests: (a) openssl_pkey_new() real cuando esté disponible el algoritmo en el entorno, (b) split test paths "A (keygen disponible) vs B (structural assertions count fallback addToAssertionCount)", nunca fabricar PEMs sintéticos con n/x/y fake.
2. **ES256 signature = raw 64-byte r‖s de WebAuthn/JOSE pero openssl_verify requiere DER ECDSA-Sig-Value (0x30‖len‖0x02‖rLen‖r‖0x02‖sLen‖s)** — Conversión bi-direccional fixed required; openssl NO soporta raw ECDSA format, hay que encode/decode DER manualmente en ambos sentidos. Regla permanente para COSE/JOSE ES256.
3. **Bearer familyId reuse detection = BFS traversal rotatedTo linked-list depth-first** — cuando se detecta reuse del padre consumido 2ª vez, revokeFamilyByReuse NO solo revoca el padre; revoca TODOS los hijos, nietos, bisnietos por rotatedTo chain (árbol rotación completo) porque cualquier token descendiente ya se emitió con compromiso del leak del padre. Implementar BFS queue-iterative no recursion para evitar stack overflow en familias profundas.
4. **AES-256-GCM encryption-at-rest FilePasskeyCredentialStore + HKDF sha256 salt+info key derivation NO vale la pena inventar custom envelope format** — Envelope shape correcto: 12 bytes random nonce GCM || 32 bytes salt HKDF || 1 byte version=0x02 || ciphertext AES-GCM || 16 bytes auth tag GCM concatenado. Autentica el header completo para detectar tampers; HKDF info = "voltstack-passkey-v2" context separation; backward compat legacy plaintext detecta si el byte[0] no es version=0x01/0x02 y lee como JSON legacy para migracion automática read-only.
5. **CompositeAuthenticatorResolver priorities globales Passkey=900 / OIDC=850 / Bearer=500 / Password=500 / Session=500** — Passkey > OIDC sobre Session/Password/Bearer porque phishing-resistant > federated > credential. NUNCA colocar password/session > 800; phishing resistant siempre arriba.
6. **AuthExceptionMapper headers strategy = array_filter eliminando nulls para que headers opcionales no aparezcan** — cuando ThrottleDeniedException->identifier === null, X-Auth-Throttle-Identifier NO se envía. Patrón general: `array_filter(['Header' => $nullableValue], static fn($v): bool => $v !== null);` evita headers vacíos o "".
7. **Application::extend() NO existe en VoltStack Container 083; para composite wiring usa binding único closure inline + instanceof check Composite o crea composite si no lo era** — wiring Passkey priority 900 + OIDC 850 se hace DENTRO del único binding AuthenticatorResolverInterface usando 2 bloques `$this->app->extend(AuthenticatorResolverInterface::class, static fn($prev, $app) => ...)` secuencialmente; el primer extend convierte DefaultResolver a Composite si no lo era, el segundo addResolver al composite existente.
8. **JUnit cross-suite 1038 tests puede generar warnings output grande a stdout; el único fallo permitido FUERA ALCANCE hay que listarlo explícitamente en SUMMARY** — si el único fallo es BloqueCTest RiskV1 aggregation mismatch (Risk V1 legacy no tocado en 083), documentarlo en corte actual VERSIONS, MATRIX y EXEC PLAN para no perder trazabilidad baseline preexistente.
9. **Atomic file writes tmp+rename pattern = rule for ALL JSON file stores** — FilePasskeyCredentialStore, FileOidcJwksCache, FileOpaqueTokenRepository, FileSessionRepository: NUNCA `file_put_contents` directo, siempre `$tmp = $dir . '/.tmp-' . bin2hex(random_bytes(8)) . '.json'`; `file_put_contents($tmp, $json, LOCK_EX); rename($tmp, $target)`; si falla cleanup `@unlink($tmp)`. Evita JSON corruptos por proceso half-write crash mid-write.
10. **Clock skew leeway OIDC exp/nbf = default 30s configurable por validator; nunca validar exp >= now exact** — distribuidos servers con clock drift 5s puede haber falsos negativos. Leeway configurable y usar siempre exp+leeway > now y nbf-leeway <= now. Audience validar overlap por lista (aud puede ser string O array de strings); issuer trailing-slash tolerant (rtrim ambos '/').

---

# AuthenticationServiceProvider DI Wiring Standards (Standing Rules permanentes desde DV-AUTH-083)

Reglas OBLIGATORIAS para cualquier ServiceProvider del framework VoltStack (especialmente `AuthenticationServiceProvider`) al inyectar interfaces nuevas en constructores resolubles por Container. Estas reglas son hard-constraints NO negables; cualquier violación causa crash de arranque `BindingResolutionException` o `Call to undefined method Application::extend()`.

## Regla 1: Método `Application::extend()` NO EXISTE — PROHIBIDO SU USO EN CUALQUIER SP

VoltStack `Application` / Container no implementa `extend()`, `afterResolving()` ni `resolved()` como API pública. Cualquier intento de `$this->app->extend(Interface::class, static fn($prev, $app) => ...)` causa Fatal error `Call to undefined method VoltStack\Framework\Application::extend()` en ComponentHydrationTest y runtime bootstrap.

**Patrón correcto**: composición de resolvers (ej: `CompositeAuthenticatorResolver`) DENTRO de un ÚNICO binding `scoped()` closure inline, donde construyes el base resolver, luego aplicas `addResolver()` secuencialmente ANTES del `return`:

```php
$this->app->scoped(AuthenticatorResolverInterface::class, static function ($app): AuthenticatorResolverInterface {
    $base = $app->make(DefaultAuthenticatorResolver::class);

    if (config('auth.passkeys.enabled', false)) {
        $passkey = $app->make(PasskeyAuthenticator::class);
        if ($base instanceof CompositeAuthenticatorResolver || ! $base instanceof CompositeAuthenticatorResolver) {
            $composite = $base instanceof CompositeAuthenticatorResolver ? $base : new CompositeAuthenticatorResolver([$base]);
            $composite->addResolver($passkey, 900);
            $base = $composite;
        }
    }

    if (config('auth.oidc.enabled', false)) {
        $oidcResolver = $app->make(OidcAuthenticatorResolver::class);
        if ($base instanceof CompositeAuthenticatorResolver || ! $base instanceof CompositeAuthenticatorResolver) {
            $composite = $base instanceof CompositeAuthenticatorResolver ? $base : new CompositeAuthenticatorResolver([$base]);
            $composite->addResolver($oidcResolver, 850);
            $base = $composite;
        }
    }

    return $base;
});
```

## Regla 2: Interfaces nuevas = binding explícito SIEMPRE (retornar null cuando config disabled) — NO confiar en `?Interfaz = null` constructor

**VoltStack Container NO respeta el valor default `= null` en type-hints de interfaz de constructores resolubles.** Si una clase tiene `public function __construct(private ?DistributedThrottleCounterInterface $throttle = null)` y ese Interface NO tiene binding registrado en SP, el Container lanza `BindingResolutionException` sin importar el `= null` del PHP.

**Patrón correcto (OBLIGATORIO)**: cada Interface nueva declarada en constructores de resolubles SP DEBE tener un binding `scoped()` o `bind()` EXPLÍCITO en el ServiceProvider, retornando `null` explícitamente cuando el config flag está desactivado (feature-default OFF):

```php
$this->app->scoped(DistributedThrottleCounterInterface::class, static function ($app): ?DistributedThrottleCounterInterface {
    if (! config('auth.throttle.distributed.enabled', false)) {
        return null;
    }
    return $app->make(RedisDistributedThrottleCounter::class);
});
```

Luego el consumidor usa `instanceof` defensive check ANTES de invocar métodos (nunca trust type-hint solo):

```php
if ($this->throttle instanceof DistributedThrottleCounterInterface) {
    $this->throttle->increment($identifier);
}
```

Flags de activación features DEFAULT false registrados oficialmente en 083:
- `auth.passkeys.enabled` (PasskeyCredentialStore + PasskeyAuthenticator)
- `auth.oidc.enabled` (OidcAuthenticatorResolver + WellKnownClient HTTP)
- `auth.throttle.distributed.enabled` (DistributedThrottleCounterInterface)
- `auth.risk.adaptive.enabled` (AdaptiveRiskPolicyInterface ConfigBased)
- `auth.bearer.tokens.enabled` (BearerTokenService scoped, DEFAULT TRUE — siempre on)

## Regla 3: Prohibido usar `bound()` / `hasBinding()` / `resolved()` en SP — no forman parte de la API pública VoltStack Container

Nunca condicionar un binding a la existencia previa via `if ($this->app->bound(Interface::class))`. Método `bound()` no existe en la API pública Application/Container; solo existe como método privado interno. Cualquier intento de usarlo causa crash arranque.

**Regla**: Interfaces nuevas de tu propio slice (ej: 083 throttle/risk) son NUEVAS en el framework (ningún otro ServiceProvider las registra). Registrar binding DIRECTO sin condicional de existencia previa.

## Regla 4: Prioridades globales CompositeAuthenticatorResolver (NO cambiar sin corte semántico)

Orden de priorities oficial registrado en 083. Nunca colocar password/session/bearer por encima de 800 (phishing-resistant siempre arriba):

| Priority | Authenticator            | Razón técnica                                                               |
|----------|--------------------------|-----------------------------------------------------------------------------|
| **900**  | PasskeyAuthenticator     | FIDO2 phishing-resistant hardware-backed = mayor assurance                 |
| **850**  | OidcAuthenticator        | Federated OIDC IDP-managed = más seguro que credential, menos que passkey   |
| **500**  | BearerTokenAuthenticator | API stateless opaque tokens = máquina-a-máquina                              |
| **500**  | PasswordAuthenticator    | Credencial shared-secret = menos seguro, fallback                           |
| **500**  | SessionAuthenticator     | Cookie remember-me = default flow de sesión ya autenticada                  |

Las inyecciones addResolver dentro del binding (Regla 1) DEBEN respetar estas prioridades numéricas para garantizar que `CompositeAuthenticatorResolver::supports()` itere en el orden correcto de trust.
