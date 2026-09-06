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

`Session Coordination Distribuida + Step-Up Operativo + Alineacion Con Controllers Security`

Documentos objetivo:

- `02_AUTHENTICATION_DOMAIN_MODEL_AND_CORE_CONCEPTS.md`
- `03_AUTHENTICATION_LIFECYCLE_AND_REQUEST_PIPELINE.md`
- `04_AUTHENTICATION_MANAGER_AND_ORCHESTRATION_SYSTEM.md`
- `05_AUTHENTICATION_FIREWALL_GUARD_AND_CONTEXT_RESOLUTION_SYSTEM.md`
- `06_AUTHENTICATOR_SYSTEM.md`
- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `09_IDENTITY_MODEL_PROVIDER_RESOLUTION_AND_FEDERATED_MAPPING_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
- `47_AUTHENTICATION_DEVELOPER_EXPERIENCE_FACADE_HELPER_CONFIGURATION_BOOTSTRAP_AND_APPLICATION_INTEGRATION_SYSTEM.md`
- `49_AUTHENTICATION_REFERENCE_IMPLEMENTATION_DEFAULT_COMPONENTS_SECURE_DEFAULTS_AND_FRAMEWORK_INTEGRATION_SYSTEM.md`

### Entregables minimos sugeridos

1. revocacion avanzada o distribuida de session.
2. step-up o elevation sobre `AuthenticationContext` sin re-login completo.
3. denial model mas rico para distinguir `required`, `guest-only`, `stale-session` y assurance insufficiente.
4. alineacion de `AuthenticationContext` con Controllers Security.
5. suite de pruebas de integracion del flujo endurecido.
6. base para stores persistentes adicionales del provider local o providers mutables mas ricos.
