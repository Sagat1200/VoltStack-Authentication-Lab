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
- Estado general: `Existe lenguaje base del subsistema, password authentication real, session auth minima, resolver formal, facade Auth, elegibilidad minima y storage configurable memory/file`
- Foco del siguiente ciclo recomendado: `password policy + session hardening + failure model mas rico`

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

### DV-AUTH-005

- Estado: `Implementado`
- Bloque documental relacionado: `01`, `03`, `04`, `12`, `22`, `26`, `47`, `49`, `50`
- Alcance objetivo:
  - introducir session auth minima dentro de `Quantum/Auth`,
  - persistir el login autenticado entre requests,
  - restaurar el `AuthenticationContext` desde cookie/header de session,
  - e invalidar esa session mediante `logout()`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionId.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSession.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/SessionAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationResponseDecorator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Support/AuthenticationHttpState.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationOrchestrator.php`
  - `vendor/voltstack/framework/src/Quantum/Http/Request.php`
  - `vendor/voltstack/framework/src/Quantum/HttpKernel/HttpKernel.php`
  - `vendor/voltstack/framework/tests/Unit/AuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `Quantum/Auth` ya puede persistir una session autenticada y recuperarla en un request posterior,
  - `AuthManager::login()` y `AuthManager::logout()` ya participan del lifecycle real de session,
  - el kernel HTTP emite `Set-Cookie` y `X-Auth-Session` cuando cambia el estado autenticado,
  - y existe cobertura automatizada para login, recovery entre requests y logout que invalida la session.
- Gap natural posterior:
  - la session aun usa repositorio en memoria,
  - no hay expiracion configurable ni rotacion,
  - no existe resolver formal de multiples authenticators,
  - y falta facade/configuracion dedicada para DX de framework.

### DV-AUTH-006

- Estado: `Implementado`
- Bloque documental relacionado: `01`, `03`, `07`, `12`, `22`, `26`, `27`, `47`, `49`
- Alcance objetivo:
  - extraer la resolucion de authenticators fuera del orchestrator,
  - introducir facade `Auth`,
  - agregar configuracion dedicada en `config/auth.php`,
  - y endurecer la session minima con expiracion configurable y limpieza de cookie al expirar.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticatorResolverInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/DefaultAuthenticatorResolver.php`
  - `vendor/voltstack/framework/src/Quantum/Facades/Auth.php`
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationOrchestrator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Support/AuthenticationHttpState.php`
  - `vendor/voltstack/framework/tests/Unit/DefaultAuthenticatorResolverTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthenticationOrchestrator` ya no decide directamente entre authenticators concretos,
  - existe `DefaultAuthenticatorResolver` como primer punto formal de composicion,
  - el framework ya dispone de facade `Auth` y `config/auth.php`,
  - y la session minima soporta expiracion configurable con limpieza de cookie cuando el recovery falla por expiracion.
- Gap natural posterior:
  - la session sigue usando repositorio en memoria,
  - no hay policy de elegibilidad de identidad,
  - falta failure handling mas coherente para Authentication,
  - y no existe storage configurable ni provider dedicado del subsistema.

### DV-AUTH-007

- Estado: `Implementado`
- Bloque documental relacionado: `10`, `12`, `25`, `26`, `49`
- Alcance objetivo:
  - introducir elegibilidad minima de identidad,
  - definir errores propios del subsistema Authentication,
  - exponer un flujo `attemptOrFail()` con semantica de excepcion,
  - y agregar storage configurable de session con driver de archivo.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/IdentitySecurityState.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthenticationException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/InvalidCredentialsException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/IdentityNotEligibleException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/LocalIdentityProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Exceptions/ExceptionHandler.php`
  - `config/auth.php`
  - `vendor/voltstack/framework/tests/Unit/FileAuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/LocalIdentityProviderTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `Quantum/Auth` ya puede rechazar identidades no elegibles antes de autenticar,
  - existe un set inicial de excepciones propias de Authentication con integracion en el `ExceptionHandler`,
  - `attemptOrFail()` habilita flujos con semantica de error autentico,
  - y las sessions pueden almacenarse en `memory` o `file` segun configuracion.
- Gap natural posterior:
  - aun falta policy formal de password,
  - faltan rotacion y revocacion avanzadas de session,
  - el storage file no cubre escenarios distribuidos,
  - y el failure model todavia no tiene taxonomy completa por mecanismo.

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
7. Session auth minima con persistencia y recovery entre requests.
8. Facade `Auth` y configuracion inicial en `config/auth.php`.
9. Resolver formal de authenticators para password y session.
10. Elegibilidad minima de identidad y errores propios de Authentication.
11. Driver de session configurable `memory` o `file`.

### Ya preparado de forma adyacente

1. Metadata y atributos de seguridad relacionados con autenticacion.
2. Excepcion `AuthenticationRequiredException`.
3. Mapeo de respuestas `401` en el handler de seguridad.
4. Semantica preliminar de `AuthenticationStrength` en Controllers Security.

### Todavia parcial o incompleto

1. Dominio de identidad y evidencias.
2. Manager y orchestrator completos para multiples mecanismos de autenticacion.
3. Password authentication con policy y lifecycle formal.
4. Session authentication con rotacion, revocacion y stores mas robustos.
5. Identity eligibility y estado de seguridad mas ricos.
6. Failure handling coherente y mas completo del subsistema.
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

Consolidar el flujo ya operativo y endurecer su nucleo:

- `10_IDENTITY_SECURITY_STATE_ACCOUNT_STATUS_AND_AUTHENTICATION_ELIGIBILITY_SYSTEM.md`
- `25_AUTHENTICATION_FAILURE_ERROR_EXCEPTION_DENIAL_AND_SECURITY_RESPONSE_HANDLING_SYSTEM.md`
- `11_PASSWORD_AUTHENTICATION_HASHING_POLICY_AND_CREDENTIAL_LIFECYCLE_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
- `49_AUTHENTICATION_REFERENCE_IMPLEMENTATION_DEFAULT_COMPONENTS_SECURE_DEFAULTS_AND_FRAMEWORK_INTEGRATION_SYSTEM.md`

### Motivo

- ya existe autenticacion real minima por password y session con resolver, facade y errores propios,
- el valor inmediato ahora esta en endurecer password, session y revocacion,
- y abrir MFA, federation o passkeys antes de cerrar eso produciria sobrearquitectura sin cierre operativo.

## Entregables minimos sugeridos para ese siguiente ciclo

1. policy de password mas explicita.
2. rotacion de session.
3. revocacion mas fuerte y cleanup.
4. posible provider dedicado del subsistema.
5. API publica minima:
   - `Auth::check()`
   - `Auth::guest()`
   - `Auth::user()`
   - `Auth::id()`
   - `Auth::attempt()`
   - `Auth::login()`
   - `Auth::logout()`
6. pruebas unitarias y feature del flujo:
   - identidad inelegible,
   - rotacion/revocacion de session,
   - recovery correcto,
   - fallo autenticado con respuesta coherente,
   - aislamiento entre requests.

## Regla de actualizacion de esta bitacora

Cada nuevo cierre de fase Authentication debe registrar:

1. un nuevo identificador `DV-AUTH-00X`,
2. documentos impactados,
3. alcance implementado,
4. evidencia principal,
5. resultado operativo,
6. siguiente gap natural.
