# DEVELOPMENT_VERSIONS

## Proposito

Esta bitacora registra el avance real del desarrollo del subsistema `Quantum/Auth` frente a la documentacion oficial ubicada en `vendor/voltstack/authentication-lab/Docs`.

Sirve como control operativo de:

- lo ya implementado,
- lo que quedo parcial,
- lo que todavia falta por construir,
- y el siguiente bloque recomendado de ejecucion.

## Corte actual

- Fecha de actualizacion: `2026-09-05`
- Estado general: `Existe lenguaje base del subsistema y un primer flujo real de password authentication sobre provider local; la session auth aun esta pendiente`
- Foco del siguiente ciclo recomendado: `session authentication + login persistence + authenticator resolution`

## Versionado de desarrollo

### DV-AUTH-001

- Estado: `Implementado`
- Bloque documental relacionado: `01`, `03`, `04`, `22`, `47`, `49`
- Alcance objetivo:
  - introducir una base minima de autenticacion request-scoped dentro del framework,
  - exponer un helper ergonomico para consultar y mutar el usuario actual,
  - validar que el estado de auth no se fugue fuera del contexto activo.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `vendor/voltstack/framework/src/Helper/helpers.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - el framework ya dispone de un `AuthManager` scoped,
  - existe helper `auth()` para acceder al servicio,
  - el estado `auth.user` se almacena dentro del `RuntimeContext`,
  - la prueba feature confirma aislamiento por request.
- Gap natural posterior:
  - esta base no autentica credenciales,
  - no resuelve identidades,
  - no crea `AuthenticationContext`,
  - no soporta session, remember-me, tokens, MFA ni policy engine.

### DV-AUTH-002

- Estado: `Parcial`
- Bloque documental relacionado: `05`, `25`, `37`, `47`, `49`
- Alcance objetivo:
  - aprovechar la infraestructura de seguridad ya existente del framework como soporte para la futura integracion de Authentication.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Attributes/AuthenticationRequired.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/AuthenticationStrength.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Engine/ControllerSecurityManager.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Exceptions/AuthenticationRequiredException.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Exceptions/ControllerSecurityExceptionMapper.php`
  - `vendor/voltstack/framework/tests/Unit/ControllerSecurityModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
- Resultado:
  - el framework ya conoce la idea de requerir autenticacion para ciertos endpoints,
  - existe una semantica adyacente de fuerza de autenticacion,
  - y ya hay responses `401` con `WWW-Authenticate` para ciertos escenarios de seguridad.
- Gap natural posterior:
  - esas piezas aun viven fuera de `Quantum/Auth`,
  - no existe un pipeline real de autenticacion que alimente esa capa,
  - y la semantica de strength actual no equivale todavia al modelo completo de assurance documentado.

### DV-AUTH-003

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `03`, `04`, `05`, `26`, `47`, `49`, `50`
- Alcance objetivo:
  - introducir el lenguaje base del subsistema Authentication,
  - separar el acceso al contexto autenticado del `AuthManager` legado,
  - crear contratos minimos de manager y orchestrator,
  - dejar una orchestracion inicial compatible con el runtime actual,
  - y validar esa base con pruebas unitarias y feature.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationOrchestratorInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/IdentityInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/IdentityIdentifier.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/IdentityReference.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/GenericIdentity.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationRequest.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContextAccessor.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Decisions/AuthenticationDecision.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Decisions/AuthenticationDecisionStatus.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationOperationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationOrchestrator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `Quantum/Auth` ya dispone de identidad minima tipada, `AuthenticationContext`, `AuthenticationDecision` y contratos base,
  - `AuthManager` ahora actua como capa de compatibilidad sobre `AuthenticationContextAccessor` y `AuthenticationOrchestrator`,
  - el framework resuelve `AuthenticationManagerInterface` y `AuthenticationOrchestratorInterface`,
  - y existe cobertura automatizada para scope por request y para el lenguaje base del subsistema.
- Gap natural posterior:
  - aun no existe autenticacion real con credenciales,
  - no hay `PasswordAuthenticator`,
  - no existe `AuthenticationSession`,
  - y el orchestrator actual solo resuelve recovery minimo desde el contexto presente.

### DV-AUTH-004

- Estado: `Implementado`
- Bloque documental relacionado: `01`, `03`, `04`, `06`, `08`, `09`, `11`, `22`, `26`, `47`, `49`, `50`
- Alcance objetivo:
  - introducir el primer provider real de identidad para autenticacion local,
  - agregar autenticacion real por `identifier/password`,
  - exponer `attempt()` como primer entry point de login por credenciales,
  - y validar el flujo con pruebas unitarias y feature.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/IdentityProviderInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticatorInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Credentials/PasswordCredentials.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/LocalIdentityProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationOrchestrator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `vendor/voltstack/framework/tests/Unit/LocalIdentityProviderTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `Quantum/Auth` ya puede autenticar credenciales reales por password contra un provider local configurado,
  - el contenedor resuelve `IdentityProviderInterface` y `AuthenticatorInterface` con implementaciones por defecto,
  - `AuthManager::attempt()` crea un `AuthenticationContext` autentico cuando las credenciales son validas,
  - y existe cobertura automatizada para login valido, login invalido y resolucion de identidad local.
- Gap natural posterior:
  - aun no existe `AuthenticationSession`,
  - el login no persiste entre requests,
  - no hay resolver de multiples authenticators,
  - y falta `login()` explicito ademas de facade/configuracion dedicada.

## Estado consolidado del sistema Authentication

### Ya utilizable hoy

1. Estado minimo request-scoped de auth dentro del runtime.
2. Helper `auth()` con operaciones basicas:
   - `user()`
   - `setUser()`
   - `check()`
   - `guest()`
   - `id()`
   - `attempt()`
   - `logout()`
3. Integracion minima del servicio en bootstrap del framework.
4. `AuthenticationContext` y `AuthenticationDecision` como lenguaje base del subsistema.
5. `AuthenticationManagerInterface` y `AuthenticationOrchestratorInterface` resueltos por el contenedor.
6. Autenticacion minima por password contra un provider local configurado.

### Ya preparado de forma adyacente

1. Metadata y atributos de seguridad relacionados con autenticacion.
2. Excepcion `AuthenticationRequiredException`.
3. Mapeo de respuestas `401` en el handler de seguridad.
4. Semantica preliminar de `AuthenticationStrength` en Controllers Security.

### Todavia parcial o incompleto

1. Dominio de identidad y evidencias.
2. Manager y orchestrator completos para multiples mecanismos de autenticacion.
3. Password authentication con policy y lifecycle formal.
4. Session authentication.
5. Login/logout flow persistido.
6. Facade y DX completas.
7. Testing system formal del subsistema.

### Aun no desarrollado con evidencia suficiente

1. Remember-me.
2. Bearer/API tokens.
3. MFA y step-up.
4. Passkeys / WebAuthn.
5. OIDC / federacion.
6. Risk engine.
7. Policy engine.
8. Multi-tenancy auth.
9. Distributed auth runtime.
10. Audit, observability y tooling operacional.

## Siguiente bloque recomendado

### Opcion recomendada inmediata

Cerrar la persistencia y restauracion del primer flujo ya autenticable:

- `07_AUTHENTICATOR_RESOLUTION_SELECTION_AND_PRIORITY_SYSTEM.md`
- `08_AUTHENTICATION_PASSPORT_CREDENTIAL_AND_EVIDENCE_SYSTEM.md`
- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
- `47_AUTHENTICATION_DEVELOPER_EXPERIENCE_FACADE_HELPER_CONFIGURATION_BOOTSTRAP_AND_APPLICATION_INTEGRATION_SYSTEM.md`
- `49_AUTHENTICATION_REFERENCE_IMPLEMENTATION_DEFAULT_COMPONENTS_SECURE_DEFAULTS_AND_FRAMEWORK_INTEGRATION_SYSTEM.md`

### Motivo

- ya existe autenticacion real minima por password,
- el valor inmediato ahora esta en persistir y restaurar ese login entre requests,
- y abrir MFA, federation o passkeys antes de cerrar session + context produciria sobrearquitectura sin cierre operativo.

## Entregables minimos sugeridos para ese siguiente ciclo

1. `AuthenticationSession` y restauracion segura.
2. `AuthenticationSessionRepositoryInterface`.
3. `SessionAuthenticator`.
4. `login()` explicito y persistencia del flujo autenticado.
5. API publica minima:
   - `Auth::check()`
   - `Auth::guest()`
   - `Auth::user()`
   - `Auth::id()`
   - `Auth::attempt()`
   - `Auth::login()`
   - `Auth::logout()`
6. pruebas unitarias y feature del flujo:
   - login correcto,
   - password invalido,
   - restauracion de session,
   - logout,
   - aislamiento entre requests.

## Regla de actualizacion de esta bitacora

Cada nuevo cierre de fase Authentication debe registrar:

1. un nuevo identificador `DV-AUTH-00X`,
2. documentos impactados,
3. alcance implementado,
4. evidencia principal,
5. resultado operativo,
6. siguiente gap natural.
