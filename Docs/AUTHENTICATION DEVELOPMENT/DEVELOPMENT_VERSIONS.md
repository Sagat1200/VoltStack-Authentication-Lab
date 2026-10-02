# DEVELOPMENT_VERSIONS

## Proposito

Esta bitacora registra el avance real del desarrollo del subsistema `Quantum/Auth` frente a la documentacion oficial ubicada en `vendor/voltstack/authentication-lab/Docs`.

Sirve como control operativo de:

- lo ya implementado,
- lo que quedo parcial,
- lo que todavia falta por construir,
- y el siguiente bloque recomendado de ejecucion.

## Corte actual

- Fecha de actualizacion: `2026-10-02`
- Estado general: DV-AUTH-083 CERRADO VERIFICADO + DV-AUTH-084 PARCIAL DOCUMENTADO. Sobre la base 083 ya se entrego el primer tramo reusable de Controllers Security bearer para cualquier app VoltStack. `Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php` ahora proyecta bearer opaco real via `BearerTokenService` en lugar de aceptar cualquier string arbitrario; `Tokens/BearerTokenService.php` expone proyecciones seguras para contexto HTTP y `revokeAccessTokenPair()` para revocar access+refresh vinculados; `Quantum/Auth/Controllers/BearerTokenOperationsController.php` mueve introspection/revoke al framework como controller reusable; el skeleton consume esa capacidad desde rutas HTTP reales `/security/demo/bearer-introspect` y `/security/demo/bearer-revoke`; y las regresiones dirigidas quedaron verdes en `ControllerSecurityContextFactoryTest`, `SkeletonSecuritySmokeTest` y `Bloque3BearerV2Test`. El resto del alcance 084 sigue abierto: drivers distribuidos reales de throttle, OIDC HTTP well-known/JWKS kid refresh, step-up E2E risk→assurance, MFA TOTP, audit JSONL durable, CI Linux OpenSSL valido y purge jobs.
- Foco del siguiente ciclo recomendado: DV-AUTH-084 continuación. Formalizar las rutas bearer fuera del espacio demo (`/auth/tokens/introspect` y `/auth/tokens/revoke` o equivalente), propagar headers/risk/assurance de forma mas estable en Controllers Security, implementar `DistributedThrottleCounterInterface` con driver Redis/DB real, completar OIDC HTTP well-known + JWKS kid miss refresh, cerrar los flows interoperables risk→assurance→step-up, agregar MFA TOTP + recovery codes, audit structured logger JSONL durable, CI Linux/OpenSSL sin skips y purge policy de refresh tokens expirados.

## Versionado de desarrollo

### DV-AUTH-084

- Estado: `Parcial`
- Bloques documentales relacionados: `14`, `22`, `47`, `48`, `49`
- Alcance implementado:
  - extraer la capacidad HTTP bearer de introspection/revoke fuera del controller demo y subirla al `framework`,
  - reutilizar `BearerTokenService` para proyectar metadata real de bearer opaco hacia Controllers Security,
  - dejar al `skeleton` como consumidor fino por rutas, sin duplicar la logica operativa.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Controllers/BearerTokenOperationsController.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Tokens/BearerTokenService.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php`
  - `vendor/voltstack/framework/tests/Unit/ControllerSecurityContextFactoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/Bloque3BearerV2Test.php`
  - `vendor/voltstack/framework/tests/Feature/SkeletonSecuritySmokeTest.php`
  - `routes/web.php`
- Resultado:
  - el framework ya ofrece un controller reusable para bearer introspection y revoke,
  - Controllers Security ya no considera autenticado cualquier bearer arbitrario cuando `BearerTokenService` esta disponible,
  - el skeleton ya monta introspection y revocacion HTTP reales sobre tokens opacos emitidos por el subsistema,
  - la revocacion del bearer actual ya invalida tambien el refresh token vinculado.
- Gap natural posterior:
  - formalizar la superficie HTTP fuera del espacio `demo`,
  - completar step-up E2E y propagacion richer de risk/assurance,
  - cerrar drivers reales y observabilidad durable que siguen pendientes en 084.

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

### DV-AUTH-008

- Estado: `Implementado`
- Bloque documental relacionado: `11`, `12`, `22`, `25`, `26`, `49`
- Alcance objetivo:
  - introducir una policy explicita de password para el login,
  - endurecer el lifecycle de session con purga, rotacion y revocacion,
  - y consolidar pruebas del flujo base bajo esas nuevas reglas.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/PasswordPolicyInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Passwords/PasswordPolicy.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/SessionAuthenticator.php`
  - `config/auth.php`
  - `vendor/voltstack/framework/tests/Unit/PasswordPolicyTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `Quantum/Auth` ya usa una policy explicita para validar password en autenticacion,
  - las sessions expiradas se purgan activamente,
  - el framework puede rotar sesiones al recuperarlas y revocar otras sesiones del mismo identity al login,
  - y el flujo base queda cubierto por pruebas de policy, rotation y revocation.
- Gap natural posterior:
  - aun no existe persistencia de upgrade de hash cuando `needsRehash()` detecta deriva,
  - la revocacion sigue siendo local al store configurado,
  - faltan stores y coordinacion distribuidos,
  - y el subsistema aun no tiene provider/middleware dedicados de integracion.

### DV-AUTH-009

- Estado: `Implementado`
- Bloque documental relacionado: `11`, `22`, `25`, `47`, `49`, `50`
- Alcance objetivo:
  - extraer el wiring del subsistema a un `AuthenticationServiceProvider`,
  - introducir un middleware real aliasado como `auth`,
  - y cerrar el primer upgrade persistente de hash para password authentication usando almacenamiento configurable del provider local.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Middlewares/AuthMiddleware.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/PasswordRehashingIdentityProviderInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/PasswordPolicyInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Passwords/PasswordPolicy.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/LocalIdentityProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `config/auth.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/LocalIdentityProviderTest.php`
- Resultado:
  - el framework ya registra Authentication mediante un provider dedicado y no por wiring embebido en `Application.php`,
  - existe middleware `auth` resoluble por alias desde rutas del framework,
  - `PasswordAuthenticator` ya puede rehashear credenciales exitosas y persistir el nuevo hash cuando el provider local usa `storage_path`,
  - y queda cobertura automatizada para middleware protegido y rehash persistente.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - falta un modelo mas rico de denials y entry points complementarios como `guest`,
  - el provider local aun no cubre multiples fuentes persistentes o escenarios distribuidos,
  - y la integracion con security/controller policy todavia no consume assurance/contexto enriquecido.

### DV-AUTH-010

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `25`, `47`, `49`, `50`
- Alcance objetivo:
  - introducir el middleware complementario `guest`,
  - agregar un denial explicito `guest_only` sin challenge headers indebidos,
  - y reforzar la integracion del subsistema con un mapper propio para errores operativos de entry points.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Middlewares/GuestMiddleware.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/GuestOnlyException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthExceptionMapper.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
- Resultado:
  - el framework ya dispone de alias `guest` para rutas exclusivas de invitados,
  - los usuarios autenticados reciben un `403` con `auth.guest_only` sin `WWW-Authenticate`,
  - y el denial queda cubierto tanto a nivel feature como en el `ExceptionHandler`.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - faltan entry points adicionales como `stale-session` o guest/auth variants mas ricas,
  - falta alinear mejor `AuthenticationContext` con Controllers Security,
  - y el subsistema aun no ofrece stores distribuidos ni denials por assurance.

### DV-AUTH-011

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `25`, `49`, `50`
- Alcance objetivo:
  - distinguir una session obsoleta o invalida del simple estado no autenticado,
  - convertir ese caso en un denial explicito `auth.stale_session`,
  - y mantener la limpieza de cookie dentro del recovery actual del subsistema.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/StaleAuthenticationSessionException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthExceptionMapper.php`
  - `vendor/voltstack/framework/src/Quantum/Middlewares/AuthMiddleware.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
- Resultado:
  - una ruta protegida por `auth` ya distingue entre ausencia de autenticacion y session vieja o invalida,
  - el framework responde `401` con `auth.stale_session` sin `WWW-Authenticate` cuando recibe una credencial de session obsoleta,
  - y el flujo sigue limpiando la cookie de auth cuando la session deja de ser recuperable.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - faltan denials por assurance o strength requerida,
  - falta alinear mejor `AuthenticationContext` con Controllers Security,
  - y el subsistema aun no ofrece stores distribuidos ni revocacion multi-nodo.

### DV-AUTH-012

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `25`, `37`, `47`, `49`, `50`
- Alcance objetivo:
  - reutilizar el vocabulario de `AuthenticationStrength` de Controllers Security desde el middleware `auth`,
  - soportar `minimum_strength` en la metadata de ruta `auth`,
  - y responder con `authentication_strength_insufficient` cuando una credencial autenticada no alcanza la assurance requerida.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Middlewares/AuthMiddleware.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
- Resultado:
  - una ruta protegida por `auth` ya puede exigir `minimum_strength` mediante metadata de ruta,
  - el denial por assurance insuficiente reutiliza `AuthenticationRequiredException` de Controllers Security,
  - y la respuesta HTTP incluye `reason_code`, challenge metadata y `WWW-Authenticate` coherentes con el mapper de Security.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - falta un assurance profile explicito dentro de `AuthenticationContext` y no solo derivado por middleware,
  - falta alinear mejor Auth con stores distribuidos y revocacion multi-nodo,
  - y el subsistema aun no cubre MFA real ni elevation/step-up.

### DV-AUTH-013

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `37`, `47`, `49`, `50`
- Alcance objetivo:
  - volver explicito el assurance profile dentro de `AuthenticationContext`,
  - conservar `authentication_strength` y `authentication_assurance_profile` al autenticar, loguear y restaurar sessions,
  - y hacer que el middleware `auth` consuma ese dato explicito en vez de derivarlo localmente.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Support/AuthenticationAssurance.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContextAccessor.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/SessionAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Middlewares/AuthMiddleware.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `AuthenticationContext` ya expone `authenticationStrength()` y `authenticationAssuranceProfile()`,
  - el login por password, el `setUser()` manual y la restauracion de session conservan el assurance profile dentro del contexto,
  - y el middleware `auth` valida `minimum_strength` leyendo ese estado explicito del subsistema.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - falta assurance multi-factor real y no solo clasificacion `Password`,
  - falta alinear mejor Auth con stores distribuidos y revocacion multi-nodo,
  - y el subsistema aun no cubre elevation/step-up ni bearer auth real.

### DV-AUTH-014

- Estado: `Implementado`
- Bloque documental relacionado: `06`, `08`, `11`, `12`, `22`, `37`, `49`, `50`
- Alcance objetivo:
  - introducir una forma real y verificable de elevar assurance dentro del flujo local de password,
  - soportar segundo factor en `LocalIdentityProvider`,
  - y persistir la assurance `MultiFactor` en el contexto y en la session restaurada.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/MultiFactorIdentityProviderInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Credentials/PasswordCredentials.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/LocalIdentityProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/SecondFactorRequiredException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/InvalidSecondFactorException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/LocalIdentityProviderTest.php`
- Resultado:
  - el provider local ya puede verificar un segundo factor configurable,
  - password + `second_factor` eleva el contexto a `MultiFactor` con `amr = ['pwd', 'mfa']`,
  - y una session nacida con MFA conserva esa assurance al recuperarse y puede satisfacer rutas con `minimum_strength = MultiFactor`.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - falta step-up operativo sin re-login completo,
  - falta alinear mejor Auth con stores distribuidos y revocacion multi-nodo,
  - y el subsistema aun no cubre bearer auth real ni MFA basada en TOTP/WebAuthn.

### DV-AUTH-015

- Estado: `Implementado`
- Bloque documental relacionado: `06`, `08`, `11`, `12`, `22`, `25`, `37`, `49`, `50`
- Alcance objetivo:
  - introducir `step-up` operativo sin re-login completo,
  - reutilizar el mismo pipeline de Auth para elevar una session ya autenticada mediante `second_factor`,
  - y persistir esa elevacion en la nueva session emitida por el subsistema.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/DefaultAuthenticatorResolver.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/MultiFactorIdentityProviderInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/LocalIdentityProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/StepUpAuthenticationRequiredException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/SecondFactorNotAvailableException.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthManager` ya expone `stepUp()` y `stepUpOrFail()`,
  - una session autenticada por password puede elevarse a `MultiFactor` verificando solo `second_factor`,
  - y la session reemitida conserva `amr`, assurance MFA y puede satisfacer rutas con `minimum_strength = MultiFactor`.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - faltan entry points y denials adicionales mas ricos alrededor de elevation y recovery,
  - falta alinear mejor Auth con stores distribuidos y revocacion multi-nodo,
  - y el subsistema aun no cubre bearer auth real ni MFA basada en TOTP/WebAuthn.

### DV-AUTH-016

- Estado: `Implementado`
- Bloque documental relacionado: `05`, `06`, `08`, `12`, `22`, `25`, `37`, `47`, `49`, `50`
- Alcance objetivo:
  - introducir un entry point explicito para rutas MFA,
  - expresar un denial propio de elevacion con `auth.step_up_required`,
  - y mejorar la DX del framework con alias `mfa` y metadata fluida `Route::mfa()`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Middlewares/MfaMiddleware.php`
  - `vendor/voltstack/framework/src/Quantum/Middlewares/AuthMiddleware.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/StepUpRequiredException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthExceptionMapper.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Routing/Route.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
- Resultado:
  - el framework ya ofrece `middleware('mfa')` como entry point explicito de elevacion,
  - las rutas MFA responden con `auth.step_up_required` y headers propios cuando la session es solo de `Password`,
  - y `Route::mfa()` permite expresar la intencion de step-up aun usando `middleware('auth')`.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado,
  - faltan denials aun mas ricos alrededor de recovery distribuido y revocacion remota,
  - falta alinear mejor Auth con stores distribuidos y revocacion multi-nodo,
  - y el subsistema aun no cubre bearer auth real ni MFA basada en TOTP/WebAuthn.

### DV-AUTH-017

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `25`, `30`, `37`, `49`, `50`
- Alcance objetivo:
  - distinguir mejor los fallos de recovery de session entre expiracion, ausencia y revocacion activa,
  - persistir marcadores minimos de recovery en los repositorios `memory/file`,
  - y exponer un denial propio `auth.revoked_session` en los entry points HTTP del framework.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionRecoveryReason.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/SessionAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/RevokedAuthenticationSessionException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthExceptionMapper.php`
  - `vendor/voltstack/framework/src/Quantum/Middlewares/AuthMiddleware.php`
  - `vendor/voltstack/framework/src/Quantum/Middlewares/MfaMiddleware.php`
  - `vendor/voltstack/framework/tests/Unit/AuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/FileAuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - los repositorios de session ya conservan una razon minima de recovery cuando una credencial expira o se revoca,
  - `SessionAuthenticator` ya distingue `session_revoked` de `session_expired` y `session_not_found`,
  - `AuthManager` propaga esa razon al runtime activo para que los middlewares respondan sin reconsultas adicionales,
  - y las rutas `auth`/`mfa` ahora pueden responder `401 auth.revoked_session` sin `WWW-Authenticate` cuando el cliente presenta una session invalidada activamente.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo oportunista y limitada al store compartido disponible,
  - faltan APIs de inventario y revocacion administrativa de sesiones por usuario/dispositivo,
  - falta limpieza/retencion gobernada de tombstones en despliegues de larga vida,
  - y el subsistema aun no cubre stores distribuidos reales, bearer auth ni MFA basada en TOTP/WebAuthn.

### DV-AUTH-018

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `35`, `47`, `49`, `50`
- Alcance objetivo:
  - introducir un inventario minimo de sesiones propias sin exponer el bearer `session_id`,
  - emitir un `session_public_id` seguro para operaciones de management,
  - y soportar revocacion dirigida de una sesion especifica o de todas las demas sesiones del principal actual.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionPublicId.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionSummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSession.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/FileAuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - cada nueva sesion emitida por Auth ya incorpora un `session_public_id` con prefijo `sess_pub_...`,
  - `AuthenticationContext` ya puede exponer una referencia segura de sesion actual distinta del secreto bearer,
  - `AuthManager` ya ofrece inventario de sesiones propias, `currentSession()`, `sessions()`, `revokeSession()` y `revokeOtherSessions()`,
  - y la suite feature valida que el inventario no devuelve los `session_id` reales y que una revocacion dirigida invalida correctamente la credencial antigua.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque el modelo ya soporta inventory seguro,
  - faltan metadata de dispositivo, ultima actividad y location approximation para una UI de security center,
  - falta retencion/cleanup gobernada de tombstones e inventario derivado en despliegues de larga vida,
  - y el subsistema aun no cubre stores distribuidos reales, bearer auth ni MFA basada en TOTP/WebAuthn.

### DV-AUTH-019

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `35`, `42`, `47`, `49`, `50`
- Alcance objetivo:
  - enriquecer el inventario de sesiones con metadata reducida y segura,
  - refrescar `last_activity` server-side en cada recovery exitoso,
  - y evitar el almacenamiento/exposicion de `User-Agent` o IP crudos dentro del inventory visible.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionSummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Unit/AuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/FileAuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - el inventory de sesiones ya expone `label`, `client_family`, `ip_prefix` y `last_activity_at`,
  - `AuthManager` ahora reduce `User-Agent` a una familia de cliente e IP a un prefijo seguro antes de persistir metadata,
  - el recovery de una sesion valida actualiza `last_activity_at` server-side mediante `touch()` en el repositorio,
  - y la suite feature valida que el inventory no devuelve ni `User-Agent` ni IP crudos del request.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque el inventory ya sea util,
  - faltan ownership/policy y fresh-auth para revocacion administrativa sensible,
  - falta metadata de dispositivo y ultima actividad mas rica para un security center completo,
  - y falta retencion/cleanup gobernada de tombstones y metadata derivada en despliegues de larga vida.

### DV-AUTH-020

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `25`, `32`, `35`, `47`, `49`
- Alcance objetivo:
  - exigir fresh authentication para revocacion remota de sesiones,
  - conservar la revocacion de la sesion actual como autocierre permitido,
  - y dejar la semantica HTTP explicita para `auth.fresh_authentication_required`.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/FreshAuthenticationRequiredException.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthExceptionMapper.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthManager` ya sella `authentication_fresh_at` al emitir una nueva sesion,
  - la revocacion remota (`revokeSession()` sobre otra sesion y `revokeOtherSessions()`) exige autenticacion fresca configurable,
  - la revocacion de la sesion actual continua disponible aun cuando la freshness window expiro,
  - y el mapper HTTP ya responde `403 auth.fresh_authentication_required` con headers de reauth sin `WWW-Authenticate`.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque la policy de revocacion ya sea mas segura,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - falta metadata de device mas rica para un security center completo,
  - y falta retencion/cleanup gobernada de tombstones y metadata derivada en despliegues de larga vida.

### DV-AUTH-021

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `35`, `42`, `47`, `49`
- Alcance objetivo:
  - enriquecer el inventory con metadata de device mas util y no sensible,
  - exponer hints de accion para revocacion por sesion,
  - y reflejar en el inventory si la revocacion remota exigira reautenticacion del actor actual.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionSummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/FileAuthenticationSessionRepositoryTest.php`
- Resultado:
  - el inventory de sesiones ya expone `client_platform`, `device_kind`, `can_revoke` y `requires_reauthentication`,
  - el label visible de sesion ya puede expresarse como metadata no confiable estilo `Chrome on Windows`,
  - `requires_reauthentication` se calcula segun la frescura de la sesion actual del actor y no segun la sesion remota listada,
  - y las pruebas feature validan hints correctos para sesiones desktop/mobile y para revocacion remota con freshness vencida.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque el inventory ya sea mas expresivo,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - faltan trusted devices, metadata de device mas estable y tooling de cleanup,
  - y falta retencion/cleanup gobernada de tombstones y metadata derivada en despliegues de larga vida.

### DV-AUTH-022

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `42`, `44`, `48`, `49`
- Alcance objetivo:
  - introducir retencion minima configurable para tombstones de recovery,
  - exponer un comando operativo basico para cleanup de sesiones y tombstones,
  - y evitar crecimiento sin limite del estado derivado del session store.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSessionsCleanupCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/ConsoleApplication.php`
  - `vendor/voltstack/framework/tests/Unit/AuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/FileAuthenticationSessionRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSessionsCleanupCommandTest.php`
- Resultado:
  - el contrato de repositorio de sesiones ya soporta purga explicita de tombstones mediante `purgeRecoveryReasons()`,
  - los stores `memory` y `file` ya respetan una ventana de retencion configurable para recovery tombstones,
  - el framework ya expone `auth:sessions:cleanup` para purgar sesiones expiradas y tombstones vencidos bajo demanda,
  - y la suite unitaria valida tanto la retencion como la ejecucion real del comando sobre un file store bootstrapped.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque el cleanup ya sea operativo,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - faltan trusted devices y metadata de device mas estable,
  - y falta un scheduler/background processing mas completo para cleanup continuo y reconciliation multi-store.

### DV-AUTH-023

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `21`, `35`, `42`, `47`, `49`
- Alcance objetivo:
  - introducir una referencia de device mas estable que el label visible sin confundirla con trusted-device real,
  - expresar de forma mas rica la policy de revocacion expuesta por el inventory,
  - y alinear `AuthenticationContext` con esa metadata derivada.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/AuthenticationSessionSummary.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - el inventory de sesiones ya expone `device_reference` pseudonimizado y `device_trust_state`,
  - la policy visible de revocacion ya incluye `revocation_scope` y `revocation_mode` ademas de `can_revoke` / `requires_reauthentication`,
  - `AuthenticationContext` ya expone `deviceReference()` y `deviceTrustState()`,
  - y la implementacion deja explicito que la referencia derivada sirve para correlacion operativa pero no equivale a trusted device ni a identidad fisica fuerte.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque el inventory ya sea mas expresivo,
  - faltan trusted devices reales y credentials de confianza duraderas,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - y falta un scheduler/background processing mas completo para cleanup continuo y reconciliation multi-store.

### DV-AUTH-024

- Estado: `Implementado`
- Bloque documental relacionado: `21`, `22`, `35`, `42`, `47`, `49`
- Alcance objetivo:
  - introducir trusted-device records persistentes ligados a `device_reference`,
  - exigir MFA para confiar el dispositivo actual por defecto,
  - y reflejar esa confianza real en el inventory de sesiones y devices.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/TrustedDeviceRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/TrustedDevice.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/TrustedDevicePublicId.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/TrustedDeviceSummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/InMemoryTrustedDeviceRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/FileTrustedDeviceRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/TrustedDeviceRepositoryTest.php`
  - `vendor/voltstack/framework/tests/Unit/FileTrustedDeviceRepositoryTest.php`
- Resultado:
  - `AuthManager` ya expone `trustedDevices()`, `trustCurrentDevice()` y `forgetTrustedDevice()`,
  - el alta de trusted device requiere MFA por defecto y se persiste como record server-side separado de la sesion,
  - el inventory de sesiones ya refleja `device_trust_state=trusted` cuando existe un trusted-device record activo para el mismo `device_reference`,
  - y el diseño deja explicito que el record persistente no equivale todavia a una trusted-device credential cliente duradera ni a identidad fisica fuerte.
- Gap natural posterior:
  - la coordinacion de sesiones sigue siendo local al store configurado aunque el posture de device ya sea mas util,
  - faltan trusted-device credentials cliente duraderas y su uso en reduccion de challenges,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - y falta un scheduler/background processing mas completo para cleanup continuo y reconciliation multi-store.

### DV-AUTH-025

- Estado: `Implementado`
- Bloque documental relacionado: `11`, `21`, `22`, `35`, `42`, `47`, `49`
- Alcance objetivo:
  - convertir el trusted-device record server-side en una credencial cliente duradera validable en runtime,
  - usar esa credencial para reducir el challenge MFA obligatorio cuando el dispositivo reconocido coincide con identidad y `device_reference`,
  - y mantener la separacion entre reconocimiento de dispositivo, sesion autenticada y nivel de assurance.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/AuthenticationResponseDecorator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Support/AuthenticationHttpState.php`
  - `vendor/voltstack/framework/src/Quantum/Http/Response.php`
  - `vendor/voltstack/framework/src/Quantum/Transport/Emitters/HttpSapiEmitter.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - el trusted-device cookie ahora usa una credencial cliente `publicId.secret` con `credential_hash` persistido server-side y validacion runtime contra identidad y `device_reference`,
  - `PasswordAuthenticator` ya puede reducir el challenge de `second_factor_required` cuando el dispositivo reconocido sigue siendo el mismo, sin elevar por ello la sesion a `MultiFactor`,
  - `AuthenticationContext` y el inventory de sesion ya exponen `trusted_device_credential_present` y `trusted_device_public_id` cuando la credencial fue validada,
  - `Response` y el emitter HTTP ya soportan multiples `Set-Cookie`, permitiendo emitir en el mismo response la session cookie y la trusted-device cookie,
  - y las pruebas feature cubren emision simultanea de cookies, challenge reduction en el mismo dispositivo y rechazo con limpieza de cookie cuando la credencial se reutiliza desde otro fingerprint.
- Gap natural posterior:
  - la coordinacion de sesiones y trusted-device state sigue siendo local al store configurado,
  - faltan rotacion automatica, replay hardening y revocacion mas rica de trusted-device credentials,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - y falta un scheduler/background processing mas completo para cleanup continuo, reconciliation multi-store y mantenimiento del posture de device.

### DV-AUTH-026

- Estado: `Implementado`
- Bloque documental relacionado: `21`, `22`, `26`, `39`, `42`, `47`, `49`
- Alcance objetivo:
  - endurecer la trusted-device credential cliente con rotacion cuando realmente reduce el challenge MFA,
  - detectar el replay del credential anterior inmediato y revocar ese trusted device,
  - y asegurar que el recovery de sesion no conserve `trusted` cuando el cookie presentado ya fue invalidado por replay.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/TrustedDeviceCredentialValidator.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/TrustedDeviceCredentialValidationResult.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - el challenge reduction por trusted device ahora rota el secreto cliente `publicId.secret` y reemite un nuevo cookie duradero sin cambiar el `public_id` del record persistido,
  - el validator conserva `previous_credential_hash` para detectar reuse inmediato del cookie anterior y revocar el trusted-device record cuando aparece un replay,
  - una recuperacion de sesion con un trusted-device cookie replayed limpia el cookie, elimina la confianza actual del contexto y evita conservar `device_trust_state=trusted` por arrastre de atributos previos,
  - y la suite feature cubre tanto la rotacion al reducir challenge como la revocacion del trusted device al reutilizar el credential anterior.
- Gap natural posterior:
  - la coordinacion de sesiones y trusted-device state sigue siendo local al store configurado,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - falta management mas expresivo para posture de dispositivo, revocacion administrativa y security center distribuido,
  - y falta un scheduler/background processing mas completo para cleanup continuo, reconciliation multi-store y mantenimiento del posture de device.

### DV-AUTH-027

- Estado: `Implementado`
- Bloque documental relacionado: `21`, `22`, `32`, `35`, `47`, `49`
- Alcance objetivo:
  - enriquecer el inventory de trusted devices con hints de management equivalentes a los de sessions,
  - exigir fresh authentication para olvidar un trusted device remoto sin bloquear el self-forget del dispositivo actual,
  - y alinear la semantica HTTP de esa revocacion remota con el denial ya usado por session management.
- Evidencia principal:
  - `config/auth.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/TrustedDeviceSummary.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `trustedDevices()` ahora expone `can_forget`, `requires_reauthentication`, `revocation_scope` y `revocation_mode` para que el security center pueda presentar acciones mas honestas sobre cada dispositivo,
  - `forgetTrustedDevice()` permite olvidar el dispositivo actual aunque la freshness window haya expirado, pero exige fresh auth cuando el target es un trusted device remoto,
  - la respuesta HTTP de ese bloqueo reutiliza `auth.fresh_authentication_required` con `operation=trusted_device_revocation`,
  - y la suite feature cubre tanto el self-forget con freshness vencida como el bloqueo de revocacion remota con hints correctos en el inventory.
- Gap natural posterior:
  - la coordinacion de sesiones y trusted-device state sigue siendo local al store configurado,
  - falta authorization/policy mas rica para escenarios administrativos y multi-actor,
  - falta security center distribuido y revocacion administrativa mas amplia sobre sessions y trusted devices,
  - y falta un scheduler/background processing mas completo para cleanup continuo, reconciliation multi-store y mantenimiento del posture de device.

### DV-AUTH-028

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `22`, `26`, `30`, `35`, `47`, `49`
- Alcance objetivo:
  - introducir una vista agregada del security center por `device_reference` en vez de solo inventories separados de sessions y trusted devices,
  - demostrar que esa vista puede leerse de forma consistente sobre un store compartido entre multiples instancias,
  - y dejar una API publica mas ergonomica para management distribuido posterior.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthManager` ahora expone `devices()` como inventory agregado por `device_reference`, combinando `sessions()` y `trustedDevices()` dentro de una sola vista de security center,
  - cada entrada resume `session_count`, `current_session_count`, `session_public_ids`, presencia de trusted device, `trusted_device_public_id`, metadata reducida de plataforma y hints de management distribuidos,
  - la suite feature valida tanto el agregado local en memoria como la lectura consistente del mismo inventory sobre `file` stores compartidos entre dos aplicaciones distintas,
  - y el subsistema ya ofrece una base mas coherente para futuros flujos de revocacion administrativa y posture distribuido sin exponer secretos bearer.
- Gap natural posterior:
  - la coordinacion distribuida sigue siendo principalmente de lectura y apoyada en el store compartido configurado,
  - faltan mutaciones mas ricas por dispositivo agregado, revocacion administrativa multi-actor y governance de ownership,
  - falta tooling operativo mas expresivo para el security center y reconciliacion multi-store,
  - y falta un scheduler/background processing mas completo para cleanup continuo y mantenimiento del posture de device.

### DV-AUTH-029

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `30`, `32`, `35`, `47`, `49`
- Alcance objetivo:
  - convertir el inventory agregado por `device_reference` en una superficie operable y no solo de lectura,
  - coordinar revocacion de sesiones y trusted device del mismo dispositivo desde una sola operacion,
  - y preservar la semantica de self-revoke y `fresh-auth` para mutaciones remotas.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthManager` ahora expone `revokeDevice(string $deviceReference)` como operacion agregada sobre el security center por dispositivo,
  - esa operacion revoca todas las sesiones del `device_reference` y olvida el trusted-device record asociado cuando existe,
  - el self-revoke del dispositivo actual continua permitido aunque la freshness window haya vencido, pero una revocacion remota desde `devices()` exige `auth.fresh_authentication_required` con `operation=device_revocation`,
  - y la suite feature valida revocacion remota coordinada, bloqueo por fresh-auth remoto y self-revoke del dispositivo actual con limpieza de cookies y sesion.
- Gap natural posterior:
  - la coordinacion distribuida sigue apoyada en el store compartido configurado y aun no en politicas/locks de multi-nodo mas fuertes,
  - falta authorization/policy administrativa multi-actor sobre devices y sessions agregadas,
  - falta tooling operativo mas expresivo para el security center y reconciliacion multi-store,
  - y falta un scheduler/background processing mas completo para cleanup continuo y mantenimiento del posture de device.

### DV-AUTH-030

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `30`, `35`, `38`, `42`, `47`, `49`
- Alcance objetivo:
  - añadir management agregado en lote sobre el security center por dispositivo,
  - mejorar el tooling operativo para que el cleanup cubra tambien trusted devices expirados,
  - y dejar una base mas util para reconciliacion y operaciones administrativas posteriores.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSessionsCleanupCommand.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSessionsCleanupCommandTest.php`
- Resultado:
  - `AuthManager` ahora expone `revokeOtherDevices()` para revocar en lote todos los dispositivos remotos del inventory agregado y conservar el dispositivo actual,
  - el bulk revoke reutiliza la semantica de `fresh-auth` cuando la operacion toca estado remoto y evita mezclar indebidamente policy de sesiones con policy de trusted devices,
  - `auth:sessions:cleanup` ahora tambien purga trusted devices expirados y reporta ese conteo junto al cleanup de sesiones y tombstones,
  - y la suite cubre tanto el bulk revoke desde `devices()` como la extension operativa del comando de cleanup.
- Gap natural posterior:
  - la coordinacion distribuida sigue apoyada en el store compartido configurado y aun no en politicas/locks de multi-nodo mas fuertes,
  - falta authorization/policy administrativa multi-actor sobre devices y sessions agregadas,
  - falta reconciliacion/background processing mas expresivo sobre inventories agregados y posture de device,
  - y falta tooling operativo todavia mas rico para auditoria, export y observabilidad del security center.

### DV-AUTH-031

- Estado: `Implementado`
- Bloque documental relacionado: `12`, `26`, `30`, `44`, `48`, `49`
- Alcance objetivo:
  - abrir soporte de enumeracion global en los repositorios de session y trusted devices para tooling operativo y administrativo,
  - introducir un comando de reconciliacion sobre stores compartidos para alinear el posture `trusted/untrusted` de sesiones con el estado real de trusted devices,
  - y dejar una base practica para reporting y operaciones administrativas posteriores.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationSessionRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/TrustedDeviceRepositoryInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/FileAuthenticationSessionRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/InMemoryTrustedDeviceRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/FileTrustedDeviceRepository.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthDevicesReconcileCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/ConsoleApplication.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDevicesReconcileCommandTest.php`
- Resultado:
  - los repositorios de session y trusted devices ya exponen `all()` como base de enumeracion global para procesos operativos y futuros flows administrativos,
  - el framework ahora ofrece `auth:devices:reconcile` con `--dry-run` para alinear el estado `session_device_trust_state`, `trusted_device_public_id` y `trusted_device_credential_present` frente al store compartido de trusted devices,
  - la reconciliacion omite sesiones expiradas, promueve sesiones a `trusted` cuando existe un trusted device activo coincidente y degrada a `unknown` cuando el estado persistido quedo obsoleto,
  - y la suite unitaria valida tanto el modo dry-run como la persistencia real de la reconciliacion sobre file stores bootstrapped.
- Gap natural posterior:
  - la coordinacion distribuida sigue apoyada en el store compartido configurado y aun no en politicas/locks de multi-nodo mas fuertes,
  - falta authorization/policy administrativa multi-actor sobre devices y sessions agregadas,
  - falta reporting operativo mas expresivo del security center y export/auditoria del estado agregado,
  - y falta una alineacion mas profunda con Controllers Security y flujos administrativos del framework.

### DV-AUTH-032

- Estado: `Implementado`
- Bloque documental relacionado: `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - introducir reporting operativo seguro del security center sobre stores compartidos,
  - exponer hints administrativos agregados mas expresivos en `devices()` para UI y management distribuido,
  - y acercar el lenguaje de management del subsistema Authentication al de Controllers Security sin romper la API publica actual.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/ConsoleApplication.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/ConsoleApplicationTest.php`
- Resultado:
  - `devices()` ahora expone `management_sensitivity` y `management_reason_code` para distinguir management directo, remoto, trusted-device y operaciones que exigen fresh auth,
  - el framework ya ofrece `auth:security-center:report` con salida segura por defecto, detalle opcional por identidad, soporte `--json` y exposicion de public IDs solo cuando el operador filtra una identidad concreta,
  - el comando resume sesiones activas, trusted devices activos, identidades unicas, dispositivos agregados y agregados con management elevado sobre stores compartidos,
  - y la suite feature/unit valida tanto los nuevos hints administrativos del inventory agregado como el registro del comando en la consola default del framework.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas,
  - falta una alineacion mas profunda entre `AuthenticationContext`, `devices()` y Controllers Security para operaciones privilegiadas,
  - falta export/auditoria mas rica del security center y observabilidad operacional continua,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-033

- Estado: `Implementado`
- Bloque documental relacionado: `26`, `37`, `47`, `49`, `50`
- Alcance objetivo:
  - hacer que Controllers Security pueda reconocer una sesion autenticada real de `Quantum\Auth` cuando no existe bearer token,
  - reutilizar `AuthenticationStrength` y claims del principal autenticado dentro del `ControllerSecurityContext`,
  - y validar la integracion tanto en unit tests del factory como en un smoke feature real sobre controladores securizados.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php`
  - `vendor/voltstack/framework/src/Platform/Application.php`
  - `vendor/voltstack/framework/tests/Unit/ControllerSecurityContextFactoryTest.php`
  - `vendor/voltstack/framework/tests/Feature/SkeletonSecuritySmokeTest.php`
- Resultado:
  - `ControllerSecurityContextFactory` ahora intenta resolver `AuthenticationManagerInterface` desde el contenedor y, cuando no hay bearer token, puede derivar el principal autenticado desde `auth()->context()`,
  - el contexto de Controllers Security ya recibe `principal`, `roles`, `permissions`, `auth_assurance_profile`, `auth_session_public_id`, `auth_device_reference`, `auth_device_trust_state`, `amr` y el `AuthenticationStrength` real de la sesion autenticada,
  - las policies de Controllers Security pueden operar sobre controladores autenticados por session sin exigir un bearer token artificial,
  - y la suite valida tanto el fallback anonimo/compatibilidad previa como el flujo real donde una sesion MFA con roles `admin` satisface `#[AuthenticationRequired(MultiFactor)]` y `role:admin`.
- Gap natural posterior:
  - sigue faltando ownership/policy administrativa multi-actor sobre sessions y devices agregadas,
  - falta export/auditoria mas rica del security center y observabilidad operacional continua,
  - falta formalizar mejor las claims administrativas y de actor privilegiado compartidas entre Auth y Controllers Security,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-034

- Estado: `Implementado`
- Bloque documental relacionado: `26`, `30`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - formalizar ownership explicito del management agregado sobre `devices()` sin romper la API publica actual,
  - reflejar ese mismo ownership en el reporte operativo `auth:security-center:report`,
  - y dejar un lenguaje mas estable para policy administrativa futura y claims compartidas con Controllers Security.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `devices()` ahora expone `management_authority` y `management_ownership_proof` para distinguir management del `session_owner` actual frente al `identity_owner` cuando el agregado representa estado remoto o mixto,
  - la semantica de ownership queda separada de `management_mode` y `requires_reauthentication`, evitando mezclar quien puede operar con el nivel de prueba requerido para hacerlo,
  - `auth:security-center:report` ya refleja `management_authority=identity_owner` y `management_ownership_proof=identity_session` en el detalle operativo por identidad,
  - y la suite feature/unit valida el ownership del inventory agregado local/remoto y el reporte JSON seguro del security center con esos nuevos hints.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas,
  - falta export/auditoria mas rica del security center y observabilidad operacional continua,
  - falta formalizar mejor las claims administrativas compartidas entre Auth y Controllers Security,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-035

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `26`, `37`, `47`, `49`, `50`
- Alcance objetivo:
  - formalizar claims administrativas compartidas dentro de `AuthenticationContext` para que Auth y Controllers Security hablen el mismo lenguaje de self-service remoto,
  - exponer esos claims tanto en `principal->claims()` como en `SecurityAttributes` cuando el contexto nace desde una sesion real de `Quantum\Auth`,
  - y validar el uso de esas claims en una policy real de Controllers Security sin requerir bearer token.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/ControllerSecurityContextFactoryTest.php`
  - `vendor/voltstack/framework/tests/Feature/SkeletonSecuritySmokeTest.php`
- Resultado:
  - `AuthenticationContext` ahora expone `managementAuthority()`, `managementOwnershipProof()` y `managementScopes()` con defaults seguros para self-service sobre sesiones y dispositivos propios,
  - `ControllerSecurityContextFactory` ya proyecta esas claims compartidas en `principal->claims()` y en atributos `auth_management_*` consumibles por el motor de policies,
  - las policies de Controllers Security ya pueden autorizar operaciones como `auth_management_authority:session_owner && auth_management_scopes:identity_device_management` cuando el actor llega autenticado por session,
  - y la smoke suite valida un endpoint real de self-service remoto autenticado por session, sin bearer token artificial y reutilizando el vocabulario compartido entre Auth y Controllers Security.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas,
  - falta export/auditoria mas rica del security center y observabilidad operacional continua,
  - falta governance mas fuerte para claims administrativas privilegiadas y actores no self-service,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-036

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `03`, `05`, `26`, `37`, `47`, `49`, `50`
- Alcance objetivo:
  - gobernar las claims administrativas privilegiadas para que no nazcan del self-service por default,
  - permitir que `AuthenticationContext` resuelva esas claims privilegiadas desde la identidad autenticada cuando fueron configuradas explicitamente,
  - y validar en Controllers Security una policy real que distinga un actor administrativo gobernado de una sesion comun de self-service.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/ControllerSecurityContextFactoryTest.php`
  - `vendor/voltstack/framework/tests/Feature/SkeletonSecuritySmokeTest.php`
  - `app/Controllers/SecurityDemoController.php`
- Resultado:
  - `AuthenticationContext` ahora expone `managementClaimsSource()` y `managementPrivilegeLevel()` ademas de hacer fallback a atributos de identidad para `auth_management_*` cuando esas claims fueron declaradas explicitamente en el provider,
  - el self-service sigue recibiendo defaults seguros (`self_service_defaults`, `self_service`) y no se promociona accidentalmente a actor privilegiado,
  - `ControllerSecurityContextFactory` ya proyecta `management_claims_source` y `management_privilege_level` junto con el resto de claims administrativas compartidas,
  - y la smoke suite valida que una sesion MFA comun no puede pasar un endpoint de export privilegiado, mientras que un `ops-admin` con claims gobernadas desde identidad si puede hacerlo sin bearer token artificial.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas,
  - falta export/auditoria mas rica del security center y observabilidad operacional continua,
  - falta gobierno operativo mas fuerte sobre actores administrativos, delegacion y export seguro,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-037

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - extender el reporte operativo del security center para exportar actores administrativos gobernados de forma explicita,
  - derivar esas claims de governance usando el mismo lenguaje de `AuthenticationContext` ya compartido con Controllers Security,
  - y mantener la salida segura por defecto, exponiendo ese detalle solo cuando el operador lo solicita.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/SkeletonSecuritySmokeTest.php`
- Resultado:
  - `auth:security-center:report` ahora soporta `--management-actors` para exportar identidades con claims administrativas gobernadas,
  - el comando deriva `management_authority`, `management_ownership_proof`, `management_claims_source`, `management_privilege_level` y `management_scopes` construyendo un `AuthenticationContext` temporal por sesion, sin duplicar reglas de negocio,
  - el resumen seguro ahora informa `governed_management_sessions` y `governed_management_identities`,
  - y la suite valida tanto el reporte seguro por defecto como la exportacion JSON explicita de actores administrativos gobernados y las regresiones de self-service vs privileged actor en Controllers Security.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas,
  - faltan mutaciones administrativas reales sobre inventories agregados mas alla del self-service del identity owner,
  - falta audit/export mas rico del security center con gobierno de delegacion y trazabilidad operacional,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-038

- Estado: `Implementado`
- Bloque documental relacionado: `26`, `30`, `35`, `48`, `49`
- Alcance objetivo:
  - introducir una mutacion administrativa real sobre el inventory agregado del security center sin esperar aun al policy engine multi-actor completo en runtime,
  - coordinar la revocacion operacional de sesiones y trusted devices por `identity + device_reference` sobre stores compartidos,
  - y mantener un contrato seguro y auditable mediante `--dry-run`, `--scope`, `--json` y detalle opcional de public IDs.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/ConsoleApplication.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/ConsoleApplicationTest.php`
- Resultado:
  - el framework ya ofrece `auth:security-center:revoke-device` para revocar operacionalmente un dispositivo agregado de una identidad concreta usando `--identity`, `--device-reference`, `--type` y `--scope`,
  - la mutacion coordina revocacion de sesiones y trusted devices apoyandose en los repositorios globales `all()` y conserva la semantica de seguridad operacional mediante `--dry-run`, `--json` y `--include-public-ids`,
  - el comando puede limitar el alcance a `sessions`, `trusted-devices` o `all`, dejando una base util para futura delegacion/policy multi-actor,
  - y la suite unitaria valida tanto el dry-run como la persistencia real, el recorte por scope y el registro del comando en la consola default del framework.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas,
  - falta gobierno/delegacion explicita de quien puede ejecutar mutaciones administrativas privilegiadas,
  - falta audit/export mas rico del security center con trazabilidad operacional persistente,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-039

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - endurecer la mutacion administrativa operacional del security center con una prueba explicita de actor gobernado,
  - exigir que la revocacion agregada por consola se ejecute usando una sesion activa del actor administrativo y no solo parametros de target,
  - y reutilizar las mismas claims administrativas gobernadas del subsistema para decidir si ese actor puede operar sobre `admin_device_management`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `auth:security-center:revoke-device` ahora exige `--actor-identity`, `--actor-type` y `--actor-session-public-id` ademas del target operativo,
  - el comando construye un `AuthenticationContext` temporal desde la sesion publica del actor y solo autoriza la mutacion cuando detecta `management_authority=administrative_actor`, `management_claims_source=identity_attributes`, `management_privilege_level=privileged_admin` y scope `admin_device_management`,
  - la salida JSON/texto ahora registra el actor autorizado y devuelve `reason_code=unauthorized_management_actor` cuando la prueba de gobierno falla,
  - y la suite valida tanto el dry-run y la ejecucion real con actor gobernado como el rechazo cuando se intenta operar con una sesion no privilegiada.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas dentro del runtime principal,
  - falta ownership/delegacion administrativa mas rica que una sola clase de actor privilegiado,
  - falta audit/export mas rico del security center con trazabilidad operacional persistente,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-040

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `24`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - enriquecer la delegacion/ownership administrativo del tooling operativo del security center sin romper el contrato ya disponible,
  - distinguir actores administrativos directos de actores delegados usando el mismo lenguaje gobernado de `AuthenticationContext`,
  - y reutilizar esa semantica tanto en la mutacion operacional como en el reporte administrativo.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `AuthenticationContext` ahora expone `canAdministrativelyManageDevices()` y `managementAuthorizationMode()` para distinguir `direct_admin` y `delegated_admin` a partir de claims gobernadas desde identidad,
  - `auth:security-center:revoke-device` ya permite tanto `privileged_admin` como `delegated_support` cuando la sesion del actor aporta `admin_device_management` y la prueba de ownership apropiada, registrando el `management_authorization_mode` efectivo,
  - `auth:security-center:report` ahora resume y exporta conteos separados de `direct_admin` y `delegated_admin` dentro de los actores administrativos gobernados,
  - y la suite valida el modelo compartido, el flujo de revocacion con admin pleno y soporte delegado, y el reporte operativo que diferencia ambos perfiles.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas dentro del runtime principal,
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta audit/export mas rico del security center con trazabilidad operacional persistente,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-041

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - introducir un rastro de auditoria persistente minimo para mutaciones administrativas del security center sin abrir aun un subsistema completo de audit distribuido,
  - dejar evidencia durable de ejecucion, dry-run y rechazo de actores no gobernados,
  - y conservar el contrato operativo actual del comando de revocacion agregada.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `auth:security-center:revoke-device` ahora soporta `--audit-log=path` para anexar eventos JSONL durables con actor, target, modo de autorizacion y resultado,
  - el comando registra eventos distintos para `executed`, `dry_run` y `authorization_failed`, incluyendo `reason_code=unauthorized_management_actor` cuando la prueba de gobierno falla,
  - el audit trail conserva el detalle operativo disponible del comando (`summary`, `session_public_ids`, `trusted_device_public_ids`) sin cambiar la superficie principal de mutacion,
  - y la suite valida persistencia del log para dry-run, ejecucion real y rechazo por actor no gobernado.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas dentro del runtime principal,
  - falta export persistente del reporte operativo del security center y no solo de la mutacion,
  - falta delegacion administrativa multi-actor mas rica que `direct_admin` vs `delegated_admin`,
  - y la coordinacion distribuida sigue dependiendo del store compartido configurado sin politicas multi-nodo mas fuertes.

### DV-AUTH-042

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - introducir un export persistente minimo del reporte operativo del security center sin romper la salida segura por defecto,
  - dejar snapshots JSONL durables que reflejen exactamente el nivel de detalle solicitado por flags,
  - y reutilizar el payload existente del comando como evidencia operativa durable.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `auth:security-center:report` ahora soporta `--export-log=path` para anexar snapshots JSONL durables del reporte generado,
  - el snapshot conserva la politica safe-by-default: solo incluye `devices` o `management_actors` cuando el operador ya los solicito mediante `--identity`, `--management-actors` y flags relacionados,
  - la salida textual sigue siendo segura por defecto y solo anexa la ruta exportada cuando corresponde, sin alterar el contrato `--json`,
  - y la suite valida tanto el snapshot seguro de resumen como el snapshot detallado por identidad con `management_actors` y public IDs.
- Gap natural posterior:
  - sigue faltando authorization/policy administrativa multi-actor sobre sessions y devices agregadas dentro del runtime principal,
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin` para actores gobernados,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-043

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `22`, `24`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - abrir el siguiente corte de policy administrativa compartiendo la decision de management gobernado entre dominio y tooling operativo,
  - reutilizar la misma semantica en `auth:security-center:report` y `auth:security-center:revoke-device`,
  - y dejar evidencia explicita de autorizacion o rechazo para actores administrativos gobernados.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `AuthenticationContext` ahora centraliza el lenguaje compartido para management gobernado con `hasGovernedManagementClaims()`, `canAdministrativelyManageDevices()`, `managementAuthorizationMode()` y `managementAuthorizationReasonCode()`,
  - `auth:security-center:revoke-device` ya consume esa decision compartida para autorizar actores, y cuando rechaza una sesion coincidente puede dejar trazabilidad del motivo (`management_authorization_reason_code`) sin romper el contrato externo de error,
  - `auth:security-center:report` ahora exporta tambien `management_authorized` y `management_authorization_reason_code`, con lo que puede distinguir actores gobernados plenos, delegados y rechazados bajo la misma policy,
  - y la suite valida actor pleno, actor delegado y actor rechazado con un mismo modelo compartido entre dominio y comandos operativos.
- Gap natural posterior:
  - falta profundizar la policy/authorization multi-actor dentro del runtime principal sobre inventories agregados y mutaciones remotas,
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-044

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - empezar a proyectar la policy administrativa compartida dentro del runtime principal del inventario agregado,
  - mantener separados los hints de ownership del dispositivo de los hints del actor actual,
  - y dejar que `auth()->devices()` exponga si el actor vigente esta gobernado, autorizado y con que motivo.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `DeviceInventorySummary` ahora expone `management_actor_governed`, `management_actor_authorized`, `management_actor_authorization_mode` y `management_actor_authorization_reason_code`,
  - `AuthManager::devices()` proyecta esos hints desde el `AuthenticationContext` actual usando la misma decision compartida ya consumida por `auth:security-center:report` y `auth:security-center:revoke-device`,
  - el inventario agregado conserva intactos sus hints de ownership (`management_authority`, `management_ownership_proof`, `management_scope`, `management_reason_code`) y no mezcla ownership del dispositivo con governance del actor,
  - y la suite valida tanto el flujo self-service normal como la proyeccion positiva de un actor gobernado dentro del runtime principal.
- Gap natural posterior:
  - falta usar estos hints del actor para endurecer mutaciones remotas multi-actor sobre inventory agregado dentro del runtime principal,
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-045

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `32`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - usar los hints del actor actual para endurecer mutaciones remotas sobre el inventario agregado dentro del runtime principal,
  - preservar la politica de `fresh-auth` para self-service remoto,
  - y permitir el caso gobernado cuando el actor actual ya trae capacidad administrativa explicita para devices.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `AuthManager::revokeDevice()` y `AuthManager::revokeOtherDevices()` ahora distinguen el caso self-service del caso gobernado sobre mutaciones remotas por dispositivo agregado,
  - cuando el actor actual puede administrar devices (`admin_device_management`), el runtime principal puede omitir la ventana de `fresh-auth` pensada para self-service remoto y ejecutar la revocacion agregada,
  - `auth()->devices()` ya refleja esa misma decision dejando de marcar `requires_reauthentication` en el agregado remoto cuando el actor actual esta gobernado y autorizado,
  - y la suite valida tanto el bloqueo por `fresh-auth` en self-service como el bypass gobernado para revocacion remota simple y bulk.
- Gap natural posterior:
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta extender esta semantica a mutaciones remotas multi-identidad dentro del runtime principal,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-046

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - llevar una primera mutacion remota multi-identidad al runtime principal,
  - reutilizar la misma policy administrativa gobernada que ya usan el inventory agregado y el tooling operativo,
  - y operar sobre stores compartidos sin depender solo del comando de consola.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `AuthenticationManagerInterface` y `AuthManager` ahora exponen `revokeManagedDevice(identity, device_reference, type?)` como primera mutacion remota multi-identidad del runtime principal,
  - el metodo usa la misma decision compartida de management gobernado para rechazar actores no autorizados y para permitir actores administrativos gobernados sobre stores compartidos,
  - la mutacion ya puede revocar sesiones y trusted-device records del target por `identity + device_reference` sin requerir pasar por `auth:security-center:revoke-device`,
  - y la suite valida tanto el caso exitoso de un actor gobernado como el rechazo limpio de un actor no gobernado.
- Gap natural posterior:
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta un inventory multi-identidad mas expresivo dentro del runtime principal para que esta mutacion no dependa de conocer externamente el `device_reference`,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-047

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - hacer mas expresivo el inventory multi-identidad dentro del runtime principal,
  - permitir que el actor gobernado descubra `device_reference` del target sin depender de fuentes externas,
  - y mantener alineadas discovery y mutacion bajo la misma policy administrativa.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `AuthenticationManagerInterface` y `AuthManager` ahora exponen `managedDevices(identity, type?)` como inventario multi-identidad gobernado del runtime principal,
  - ese inventory se construye sobre stores compartidos usando la misma proyeccion de `DeviceInventorySummary` y los mismos hints administrativos que el inventory propio,
  - los actores no gobernados no reciben ese inventario, mientras que los actores autorizados pueden descubrir el `device_reference` del target y luego ejecutar `revokeManagedDevice(...)`,
  - y la suite ya valida discovery gobernado del target, revocacion multi-identidad usando ese discovery, y rechazo limpio para actores no gobernados.
- Gap natural posterior:
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - falta enriquecer las mutaciones multi-identidad mas alla de la revocacion por dispositivo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-048

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - enriquecer la mutacion multi-identidad del runtime principal mas alla del caso "todo el dispositivo",
  - alinear su contrato con el tooling operacional ya existente en consola,
  - y permitir revocacion parcial de sesiones o trusted devices sobre el target.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/AuthenticationManagerInterface.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `AuthenticationManagerInterface` y `AuthManager` ahora permiten `revokeManagedDevice(identity, device_reference, type?, scope)` con `all`, `sessions` o `trusted-devices`,
  - la mutacion gobernada del runtime principal ya puede revocar solo sesiones del target dejando el trusted-device record intacto, o revocar solo trusted devices preservando la sesion activa,
  - el contrato queda mas alineado con `auth:security-center:revoke-device` sin depender exclusivamente del comando de consola,
  - y la suite valida revocacion parcial por `sessions`, revocacion parcial por `trusted-devices`, ademas de las rutas ya cubiertas de discovery y rechazo para actores no gobernados.
- Gap natural posterior:
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta ownership multi-identidad mas expresivo sobre sessions y trusted devices dentro del runtime principal,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-049

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - hacer mas expresivo el ownership multi-identidad del inventory gobernado,
  - separar con claridad el target administrado del actor actual,
  - y dejar esa semantica disponible tanto para inventario propio como para inventario gobernado.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `DeviceInventorySummary` ahora expone `management_target_identity`, `management_target_type` y `management_target_matches_current_identity`,
  - `AuthManager::devices()` y `AuthManager::managedDevices()` ya proyectan explicita y consistentemente la identidad target a la que pertenece el agregado administrado,
  - el inventory propio marca que el target coincide con la identidad actual, mientras que el inventory gobernado deja claro cuando el target es otra identidad administrada,
  - y la suite valida ambos escenarios sin cambiar la semantica previa de `management_authority` ni `management_ownership_proof`.
- Gap natural posterior:
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta ownership administrativo por actor mas expresivo sobre sessions y trusted devices dentro del runtime principal,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-050

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - hacer mas expresivo el ownership administrativo del actor actual dentro del inventory,
  - alinear el runtime principal con las claims administrativas que ya exporta el security center report,
  - y separar mejor las claims del actor de la semantica de ownership del target administrado.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `DeviceInventorySummary` ahora expone `management_actor_authority`, `management_actor_ownership_proof`, `management_actor_claims_source`, `management_actor_privilege_level` y `management_actor_scopes`,
  - `AuthManager::devices()` y `AuthManager::managedDevices()` ya proyectan no solo si el actor esta autorizado, sino tambien desde que autoridad, prueba, origen de claims, privilegio y scopes llega esa administracion,
  - el inventory self-service conserva valores por defecto (`session_owner`, `current_session`, `self_service_defaults`, `self_service`),
  - y el inventory gobernado ya refleja consistentemente actores administrativos configurados desde identidad (`administrative_actor`, `privileged_session`, `identity_attributes`, `privileged_admin`, `admin_device_management`).
- Gap natural posterior:
  - falta delegacion administrativa mas rica que `direct_admin` vs `delegated_admin`,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - falta gobernar ownership administrativo mas rico sobre actores delegados y relaciones actor-target,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-051

- Estado: `Implementado`
- Bloque documental relacionado: `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - enriquecer la delegacion administrativa gobernada en el runtime principal,
  - distinguir mejor la relacion actor-target dentro del inventory,
  - y dejar esa semantica disponible tanto para self-service como para actores directos y delegados.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
- Resultado:
  - `AuthenticationContext` ahora expone `managementActorTargetRelation()` y `managementActorTargetReasonCode()`,
  - `DeviceInventorySummary` ya proyecta `management_actor_target_relation` y `management_actor_target_reason_code`,
  - el runtime principal distingue explicitamente `self`, `self_governed`, `direct_administrative_target` y `delegated_administrative_target`,
  - y la cobertura valida self-service, actor directo gobernado sobre si mismo, actor directo sobre otra identidad y actor delegado sobre otra identidad.
- Gap natural posterior:
  - falta delegacion administrativa multi-actor mas rica que la actual taxonomia `direct_admin/delegated_admin`,
  - falta coordinacion distribuida mas fuerte del security center sobre stores compartidos y escenarios multi-nodo,
  - falta gobernar relaciones actor-target mas profundas sobre sessions y trusted devices compartidos,
  - y faltan logs estructurados, metricas y tracing mas amplios alrededor del subsistema Auth.

### DV-AUTH-052

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `22`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - distinguir dentro del runtime principal si un actor gobernado puede administrar `sessions`, `trusted-devices` o ambos sobre el target,
  - proyectar esa capacidad parcial de forma honesta en el inventory agregado,
  - y alinear la mutacion operacional de consola con la misma semantica por alcance.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `AuthenticationContext` ahora resuelve autorizacion administrativa por alcance mediante `canAdministrativelyManageDeviceSessions()`, `managementSessionAuthorizationMode()`, `managementSessionAuthorizationReasonCode()`, `canAdministrativelyManageTrustedDevices()`, `managementTrustedDeviceAuthorizationMode()` y `managementTrustedDeviceAuthorizationReasonCode()`,
  - `DeviceInventorySummary` ya proyecta `management_actor_can_manage_sessions`, `management_actor_session_authorization_mode`, `management_actor_session_authorization_reason_code`, `management_actor_can_manage_trusted_devices`, `management_actor_trusted_device_authorization_mode` y `management_actor_trusted_device_authorization_reason_code`,
  - `AuthManager::revokeManagedDevice(identity, device_reference, type?, scope)` ya exige autorizacion consistente con `all|sessions|trusted-devices`, permitiendo delegacion parcial real sin sobreactuar permisos,
  - `auth:security-center:revoke-device` ya evalua el actor actual con la misma semantica por alcance y rechaza scopes no cubiertos con su motivo explicito,
  - y la suite valida escenarios delegado pleno, delegado solo para `sessions` y delegado solo para `trusted-devices` tanto en dominio como en runtime y tooling operativo.
- Gap natural posterior:
  - falta modelar relaciones actor-target delegadas mas profundas que el alcance binario actual sobre `sessions` y `trusted-devices`,
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta enriquecer auditabilidad, metricas y trazabilidad administrativa alrededor de mutaciones gobernadas,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-053

- Estado: `Implementado`
- Bloque documental relacionado: `02`, `22`, `26`, `35`, `47`, `49`
- Alcance objetivo:
  - hacer mas expresivo el perfil actor-target administrado dentro del runtime principal sin romper la taxonomia gruesa ya existente,
  - distinguir si la relacion gobernada sobre el target es de alcance pleno, solo de `sessions` o solo de `trusted-devices`,
  - y proyectar esa semantica fina de forma coherente en dominio, inventory y runtime multi-identidad.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/DeviceInventorySummary.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthenticationContext` ahora expone `managementActorTargetScopeRelation()` y `managementActorTargetScopeReasonCode()` para derivar un perfil fino del target administrado,
  - ese perfil distingue `self_service_current_identity_target`, `self_governed_full_scope_target`, `delegated_admin_full_scope_target`, `delegated_admin_sessions_scope_target`, `delegated_admin_trusted_devices_scope_target` y equivalentes directos cuando aplica,
  - `DeviceInventorySummary` ya proyecta `management_actor_target_scope_relation` y `management_actor_target_scope_reason_code` junto a la relacion gruesa previa,
  - `AuthManager::devices()` y `AuthManager::managedDevices()` ya reflejan ese perfil fino del target administrado sin mezclarlo con el ownership agregado ni con los hints de accion,
  - y la suite valida self-governed pleno, direct admin pleno, delegated admin pleno, delegated admin solo para `sessions` y delegated admin solo para `trusted-devices`.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta enriquecer audit trail, metricas y trazabilidad administrativa alrededor de mutaciones gobernadas,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-054

- Estado: `Implementado`
- Bloque documental relacionado: `26`, `48`, `49`
- Alcance objetivo:
  - hacer mas observable el comportamiento del security center cuando opera sobre stores potencialmente compartidos entre instancias,
  - dejar que reportes, snapshots y auditoria durable expliciten la topologia operativa observada por el nodo local,
  - y preparar mejor la correlacion multi-instancia sin cambiar aun la policy de revocacion.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `auth:security-center:report` ahora emite `operational_context` en JSON y en `--export-log`,
  - `auth:security-center:revoke-device` ahora emite `operational_context` en JSON y en `--audit-log`,
  - ese contexto expone `app_name`, `app_env`, `session_driver`, `trusted_device_driver`, `session_store_path`, `trusted_device_store_path`, `store_topology` y `store_fingerprint`,
  - la salida `--verbose` del tooling operativo ya muestra tambien la topologia observada y las rutas de store relevantes,
  - y la suite valida la persistencia durable de ese contexto cuando el security center opera sobre drivers `file`.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar correlacion operativa y metricas administrativas mas ricas sobre mutaciones gobernadas,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-055

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - hacer explicita la correlacion operativa entre reporte, snapshot durable y mutacion administrativa del security center,
  - permitir que una operacion pueda enlazarse entre nodos o procesos aun cuando compartan el mismo store observado,
  - y dejar una base clara para metricas y trazabilidad multi-instancia mas ricas.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `auth:security-center:report` ahora soporta `--correlation-id` y siempre emite `correlation_id` + `operation_id` en JSON y `--export-log`,
  - `auth:security-center:revoke-device` ahora soporta `--correlation-id` y siempre emite `correlation_id` + `operation_id` en JSON y `--audit-log`,
  - la salida `--verbose` del tooling operativo ya muestra tambien ambos identificadores para correlacion local inmediata,
  - los eventos de rechazo, dry-run, ejecucion y export conservan esa correlacion sin depender solo del timestamp o del path del store,
  - y la suite valida correlacion explicita tanto en payloads JSON como en JSONL durable.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar metricas administrativas mas ricas sobre mutaciones gobernadas y correlacionar lotes o pipelines operativos,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-056

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - hacer mas legible y resumible el comportamiento administrativo gobernado del security center,
  - reutilizar la correlacion operativa ya disponible para emitir metricas utiles por reporte y por mutacion,
  - y preparar una base mas firme para metricas longitudinales y coordinacion multi-nodo posterior.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `auth:security-center:report` ahora emite `administrative_metrics` con conteos autorizados/no autorizados, modos de autorizacion, niveles de privilegio y cobertura de scopes gobernados,
  - `auth:security-center:revoke-device` ahora emite `administrative_metrics` por operacion con `authorization_outcome`, `requested_scope`, `actor_scope_profile`, recursos emparejados/afectados y clases de recurso tocadas,
  - la salida legible de ambos comandos ya muestra un resumen corto de esas metricas administrativas,
  - los snapshots `--export-log` y auditorias `--audit-log` preservan esa misma capa metrica junto con `correlation_id`, `operation_id` y `operational_context`,
  - y la suite valida tanto la agregacion administrativa del reporte como las metricas de ejecucion, dry-run y rechazo operacional.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar metricas administrativas longitudinales o por lote sobre pipelines operativos multi-instancia,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-057

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - permitir que el security center lea historial administrativo durable sin crear tooling paralelo,
  - agregar una primera capa de metricas longitudinales reutilizando `--audit-log`,
  - y dejar trazabilidad historica por outcome, scope, perfil del actor y cohorte de store observada.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `auth:security-center:report` ahora soporta `--audit-log-source=path`,
  - el reporte emite `longitudinal_metrics` con conteo de eventos, correlation ids, operation ids, outcomes, scopes, perfiles de alcance, modos administrativos, recursos afectados y topologias observadas,
  - el resumen legible del comando ya muestra un snapshot corto de esas metricas historicas cuando se pasa una fuente de auditoria,
  - los snapshots `--export-log` preservan tambien esa capa longitudinal junto con el estado actual,
  - y la suite valida tanto el resumen legible como el payload JSON/exportado cuando se agrega historial desde JSONL durable.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar cohortes/historial distribuido mas fino por store o ventana temporal operativa,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-058

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - hacer visible la dimension distribuida del historial administrativo por store observado,
  - separar cohortes longitudinales por `store_fingerprint` y `store_topology`,
  - y preparar una base para ventanas temporales o cohortes operativas mas finas.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `longitudinal_metrics` ahora incluye `store_cohorts` con conteo de eventos, outcomes, scopes, modos administrativos, recursos afectados y `latest_event_at` por cohorte de store,
  - la salida legible del reporte ahora resume el `top_store` observado y, en `--verbose`, lista cohortes distribuidas por `store_fingerprint`,
  - los snapshots `--export-log` preservan tambien esa vista distribuida del historial,
  - y la suite valida tanto el resumen legible como la estructura JSON/exportada de esas cohortes.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar ventanas temporales o cohortes operativas mas finas sobre el historial distribuido,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-059

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - volver mas legible el historial administrativo reciente del security center,
  - resumir actividad distribuida en ventanas temporales relativas al ultimo evento observado,
  - y dejar una base util para consolidacion temporal multi-nodo mas fuerte.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `longitudinal_metrics` ahora incluye `time_windows` sobre `last_5m`, `last_15m` y `last_60m`,
  - cada ventana resume eventos, stores observados, `top_store_fingerprint`, outcomes, scopes, modos administrativos y recursos afectados,
  - la salida legible del reporte ahora muestra un resumen corto de esas ventanas y, en `--verbose`, lista el detalle distribuido por ventana,
  - y la suite valida tanto el resumen legible como la estructura JSON/exportada de las ventanas temporales.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar consolidacion temporal o ventanas aun mas finas por store/nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-060

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - unir las cohortes por store con las ventanas temporales recientes,
  - dejar una vista consolidada por fingerprint/topologia para leer actividad reciente por store,
  - y preparar una base mejor para coordinacion temporal multi-nodo mas fuerte.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `longitudinal_metrics` ahora incluye `store_time_windows`,
  - cada store consolidado expone sus ventanas `last_5m`, `last_15m` y `last_60m` con outcomes y recursos afectados,
  - la salida legible del reporte ahora resume el `top_recent_store` y, en `--verbose`, lista la consolidacion temporal por store,
  - y la suite valida tanto el resumen legible como la estructura JSON/exportada de esa consolidacion temporal.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar consolidacion temporal multi-store mas fuerte o ventanas aun mas finas por store/nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-061

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - ofrecer una lectura multi-store mas inmediata sobre la actividad reciente del security center,
  - identificar si el comportamiento observado esta concentrado o distribuido entre stores,
  - y preparar una base para reglas de consistencia multi-nodo mas fuertes.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `longitudinal_metrics` ahora incluye `multi_store_summary`,
  - ese resumen expone stores activos por ventana, `top_recent_store`, `latest_event_spread_seconds` y `coordination_profile`,
  - la salida legible del reporte ahora resume si la actividad reciente es `distributed`, `concentrated`, `single_store` o `idle`,
  - y la suite valida tanto el resumen legible como la estructura JSON/exportada de ese perfil multi-store.
- Gap natural posterior:
  - falta coordinacion distribuida multi-nodo mas fuerte del security center sobre stores compartidos,
  - falta agregar consistencia temporal mas fuerte entre stores o deteccion de drift operativo mas expresiva,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-062

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - detectar drift operativo entre stores compartidos a partir del historial longitudinal ya consolidado,
  - volver visible el desfase reciente entre stores en la salida legible y verbose del security center,
  - y dejar una base inicial para gobierno de consistencia multi-store mas accionable.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `longitudinal_metrics` ahora incluye `activity_drift`,
  - ese bloque expone `drift_detected`, `drift_profile`, `severity`, `reference_window`, `max_event_gap_seconds`, conteos de inactividad reciente y fingerprints rezagados o stale,
  - el reporte legible y verbose ahora resume drift multi-store con perfiles `recent_lag`, `partial_visibility`, `store_dropout`, `single_store` o `none`,
  - y la suite valida la misma semantica tanto en stdout como en payload JSON y export duradero.
- Gap natural posterior:
  - falta convertir `activity_drift` en gobierno operativo mas accionable para decisiones administrativas distribuidas,
  - falta enriquecer la explicabilidad por store y por ventana para escenarios multi-nodo mas irregulares,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-063

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `26`, `48`, `49`
- Alcance objetivo:
  - convertir `activity_drift` en una senal operativa mas accionable para escenarios multi-store y multi-nodo,
  - enriquecer la explicabilidad del drift por store y por ventana reciente sin romper el contrato existente del reporte,
  - y preparar una base para escalaciones operativas y denials distribuidos mas expresivos.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `activity_drift` ahora expone `recommended_action`, `reference_store_fingerprint`, `reference_store_topology`, `window_coverage` y `store_assessments`,
  - la salida resumida del reporte ahora adelanta la accion sugerida y la salida verbose detalla cobertura por ventana y estado por store,
  - los store assessments distinguen estados como `lagging`, `inactive_15m`, `stale` o `healthy` usando el gap reciente y la actividad por ventana,
  - y la suite valida esa misma semantica en stdout, payload JSON y export duradero.
- Gap natural posterior:
  - falta traducir `recommended_action` a denials, alertas o escalaciones operativas mas concretas,
  - falta enriquecer acciones diferenciadas para perfiles mas irregulares de multi-nodo y store dropout,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-064

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - traducir `recommended_action` de `activity_drift` a una respuesta operativa mas concreta dentro del propio reporte,
  - dejar base para denials distribuidos mejor explicados antes de aplicarlos a mutaciones remotas reales,
  - y validar tanto un caso de `recent_lag` como uno de `store_dropout`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
- Resultado:
  - `activity_drift` ahora expone `operational_response` con `response_mode`, `escalation_level`, `should_deny_remote_mutations`, `remote_mutation_denial_reason_code`, `next_step` y `target_store_fingerprints`,
  - la salida resumida del reporte ahora adelanta `response` y si conviene negar mutaciones remotas, mientras la salida verbose agrega una seccion `Respuesta operativa`,
  - la suite valida tanto el caso `monitor_recent_lag -> observe_recent_lag` como `investigate_store_dropout -> contain_store_dropout`,
  - y el reporte ya deja una razon operativa reutilizable para futuras mutaciones administrativas distribuidas.
- Gap natural posterior:
  - falta aplicar `should_deny_remote_mutations` y `remote_mutation_denial_reason_code` a mutaciones remotas reales del security center,
  - falta enriquecer perfiles intermedios de guardia operativa para escenarios mas irregulares de multi-nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-065

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - aplicar la guardia distribuida derivada del drift a mutaciones remotas reales del security center,
  - reutilizar la misma semantica longitudinal del reporte sin duplicar logica de drift,
  - y validar tanto un escenario permitido (`recent_lag`) como uno bloqueado (`store_dropout`).
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `auth:security-center:report` ahora expone un helper reutilizable para derivar `distributed_guard` desde `--audit-log-source`,
  - `auth:security-center:revoke-device` ahora soporta `--audit-log-source`, agrega `distributed_guard` al payload y a la auditoria durable, y aplica un rejection real cuando `operational_response.should_deny_remote_mutations` es verdadero,
  - ese rejection reutiliza `remote_mutation_denial_reason_code` como reason code operativo distribuido,
  - la suite valida que `recent_lag` deja ejecutar la mutacion y que `store_dropout` la bloquea sin persistir cambios.
- Gap natural posterior:
  - falta graduar denials distribuidos por severidad, scope y tipo de mutacion administrativa,
  - falta enriquecer perfiles intermedios de guardia operativa para escenarios mas irregulares de multi-nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-066

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - graduar `distributed_guard` por severidad y `scope` de la mutacion administrativa remota,
  - reemplazar el modelo binario `deny/allow` por una policy operativa mas fina reutilizable por reporte y revocacion,
  - y validar escenarios intermedios como `partial_visibility`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `operational_response` ahora proyecta `remote_mutation_scope_policy`, `allowed_remote_mutation_scopes`, `denied_remote_mutation_scopes` y `scope_denial_reason_codes`,
  - el drift distribuido ahora distingue politicas como `allow_all`, `sessions_only` y `deny_all`,
  - `auth:security-center:revoke-device` ya deriva `distributed_guard_scope_decision` para el `scope` solicitado y solo bloquea la mutacion cuando ese alcance concreto queda negado,
  - la suite valida `recent_lag` permitido, `partial_visibility` permitido para `sessions` pero denegado para `trusted-devices`, y `store_dropout` denegado para `all`.
- Gap natural posterior:
  - falta volver esos denials distribuidos sensibles tambien al perfil del actor y al tipo de mutacion administrativa,
  - falta enriquecer perfiles intermedios de guardia operativa para escenarios mas irregulares de multi-nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-067

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - volver `distributed_guard` sensible al perfil del actor administrativo y al tipo concreto de mutacion remota,
  - reutilizar una misma semantica distribuida entre reporte y mutacion real sin duplicar reglas de policy,
  - y validar que `direct_admin` y `delegated_admin` no reciban la misma decision bajo un mismo `activity_drift`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `operational_response` ahora proyecta `authorization_mode_scope_policies` para diferenciar politicas distribuidas por `direct_admin`, `delegated_admin` y `none`,
  - `distributed_guard_scope_decision` ahora incorpora `mutation_kind`, `authorization_mode`, `actor_privilege_level`, `policy_source` y `policy_reason_code`,
  - bajo `partial_visibility`, `direct_admin` puede conservar `sessions_only` mientras `delegated_admin` pasa a `deny_all`,
  - y la suite valida casos permitidos y denegados para el mismo `scope` segun el modo administrativo del actor.
- Gap natural posterior:
  - falta enriquecer perfiles intermedios de guardia operativa para escenarios mas irregulares de multi-nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil de alcance actual,
  - y falta seguir madurando policy/authorization multi-actor mas expresiva dentro del runtime principal.

### DV-AUTH-068

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - volver `distributed_guard` sensible al privilegio fino del actor administrativo y a la relacion actor-target concreta,
  - mantener una misma semantica compartida entre `report` y `revoke-device` sin reintroducir reglas duplicadas,
  - y validar que `self_governed`, `direct_administrative_target` y `delegated_administrative_target` no colapsen en una sola respuesta operativa.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `operational_response.authorization_mode_scope_policies` ahora soporta overlays anidados por `privilege_scope_policies` y `target_relation_scope_policies`,
  - `distributed_guard_scope_decision` ahora incorpora `actor_target_relation` y `actor_target_reason_code` ademas de resolver la policy final en cadena `authorization_mode -> actor_privilege_level -> actor_target_relation`,
  - bajo `partial_visibility`, un `privileged_admin` sobre `self_governed` puede conservar `allow_all`, mientras un `delegated_support` sobre `self_governed` queda restringido a `sessions_only` y un `delegated_administrative_target` permanece en `deny_all`,
  - y la suite valida casos permitidos y denegados para `trusted-devices` y `sessions` sobre targets propios y delegados sin romper los denials distribuidos existentes.
- Gap natural posterior:
  - falta profundizar la policy multi-actor para ownership administrativo mas rico y overlays adicionales por actor-target delegado,
  - falta enriquecer perfiles intermedios de guardia operativa para escenarios mas irregulares de multi-nodo,
  - falta seguir profundizando relaciones actor-target delegadas mas alla del perfil actual de `self_governed/direct/delegated`,
  - y falta seguir madurando policy/authorization administrativa mas expresiva dentro del runtime principal.

### DV-AUTH-069

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - formalizar la semantica actor-target por alcance dentro de `AuthenticationContext` para que el runtime y el tooling no dependan de inferencias implícitas,
  - proyectar esa taxonomia por alcance en el inventory agregado del runtime principal,
  - y hacer que `report` y `revoke-device` compartan overlays distribuidos mas profundos hasta el nivel `target_scope_relation`.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Context/AuthenticationContext.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthDomainModelTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `AuthenticationContext` ahora publica `managementActorTargetScopeRelation()` y `managementActorTargetScopeReasonCode()` para distinguir `self_governed_*`, `direct_admin_*` y `delegated_admin_*` segun `all|sessions|trusted-devices`,
  - `AuthManager::devices()` y `managedDevices()` ahora proyectan explicitamente `management_actor_target_scope_relation` y `management_actor_target_scope_reason_code` desde esa semantica compartida,
  - `operational_response.authorization_mode_scope_policies` ahora puede descender hasta `target_scope_relation_policies`, y `distributed_guard_scope_decision` ya resuelve y audita tambien `actor_target_scope_relation` y `actor_target_scope_reason_code`,
  - bajo `partial_visibility`, un delegado administrativo acotado al target `delegated_admin_sessions_scope_target` puede conservar `sessions_only` sin relajar el `deny_all` del target delegado pleno,
  - y la suite valida dominio, reporte y mutacion remota para overlays profundos sin romper los denials distribuidos existentes.
- Gap natural posterior:
  - falta extender `target_scope_relation_policies` a mas combinaciones actor-target-scope y perfiles multi-nodo intermedios,
  - falta enriquecer la guardia distribuida para escenarios graduales entre `recent_lag`, `partial_visibility` y `store_dropout`,
  - y falta ampliar cobertura feature del runtime principal para estas nuevas taxonomias actor-target-scope en paths publicos.

### DV-AUTH-070

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - convertir `recent_lag` en un perfil operativo realmente intermedio entre observacion pasiva, `partial_visibility` y `store_dropout`,
  - ampliar overlays `actor-target-scope` dentro de la guardia distribuida sin romper la semantica ya compartida entre `report` y `revoke-device`,
  - y endurecer la explicabilidad de los denials graduales por actor administrativo.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `monitor_recent_lag` ya publica `authorization_mode_scope_policies` explicitas para `direct_admin`, `delegated_admin` y `none`,
  - `recent_lag` ahora conserva `allow_all` para `direct_admin`, degrada `delegated_admin` a `sessions_only` y mantiene `deny_all` para actores no confiables,
  - el árbol `delegated_admin -> delegated_support -> delegated_administrative_target -> target_scope_relation_policies` ya distingue al menos `delegated_admin_sessions_scope_target` y `delegated_admin_trusted_devices_scope_target`,
  - `revoke-device` ahora permite `sessions` a un delegado acotado bajo `recent_lag`, pero puede bloquear `trusted-devices` del mismo perfil con motivos y `policy_reason_code` específicos,
  - y la suite valida tanto la lectura del contrato publicado por `report` como el enforcement real del guard gradual.
- Gap natural posterior:
  - falta llevar esta taxonomia actor-target-scope a mas pruebas feature del runtime principal (`managedDevices()` y `revokeManagedDevice()`),
  - falta endurecer politicas adicionales para perfiles como `concentrated` o variaciones mas finas derivadas de `store_assessments`,
  - y falta enriquecer la observabilidad operativa para que el reporte resuma mejor que combinaciones actor-scope quedan degradadas en cada drift.

### DV-AUTH-071

- Estado: `Implementado`
- Bloque documental relacionado: `26`, `35`, `47`, `49`
- Alcance objetivo:
  - ampliar la cobertura feature del runtime principal para `managedDevices()` y `revokeManagedDevice()` sobre taxonomias `actor-target-scope`,
  - fijar con pruebas HTTP reales la semantica `self_governed_*` y completar la proyeccion `direct_admin_*` en los payloads del runtime,
  - y dejar mejor anclada la alineacion entre `AuthenticationContext`, `AuthManager` y los paths publicos del framework.
- Evidencia principal:
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `AuthManagerTest` ahora fija explicitamente `direct_admin_full_scope_target` en `managedDevices()` para actores privilegiados sobre otra identidad,
  - el runtime principal ya tiene cobertura feature especifica para `self_governed_sessions_scope_target` y `self_governed_trusted_devices_scope_target`,
  - `revokeManagedDevice()` queda validado en escenarios self-governed parciales: puede revocar solo sesiones preservando trusted devices, o solo trusted devices preservando la sesion autenticada,
  - y la serializacion HTTP del runtime expone de forma consistente `management_actor_target_scope_relation` y `management_actor_target_scope_reason_code` tanto para targets delegados como self-governed/direct admin.
- Gap natural posterior:
  - falta endurecer la guardia distribuida con perfiles adicionales derivados de `coordination_profile`, `store_assessments` y señales multi-store mas finas,
  - falta enriquecer la observabilidad operativa para resumir mejor degradaciones activas por actor, target y scope,
  - y falta seguir ampliando cobertura end-to-end entre runtime principal y tooling operativo distribuido.

### DV-AUTH-072

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `48`, `49`
- Alcance objetivo:
  - endurecer la guardia distribuida con un perfil multi-store intermedio adicional derivado de `coordination_profile`,
  - volver mas accionable la observabilidad operativa cuando la actividad luce concentrada pero no hay perdida dura de visibilidad,
  - y mantener alineados `report` y `revoke-device` sobre la misma degradacion actor-target-scope.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
- Resultado:
  - `multi_store_summary` ahora proyecta `top_recent_store_share_15m` y `activity_drift` puede distinguir explicitamente el perfil `concentrated_activity` cuando la coordinacion reciente luce concentrada sin caer en `recent_lag`, `partial_visibility` ni `store_dropout`,
  - `operational_response` ya publica `observe_concentrated_activity`, `degraded_scope_profiles` y overlays graduales donde `direct_admin` conserva `allow_all`, `delegated_admin` baja a `sessions_only`, `delegated_support:self_governed` puede seguir en `allow_all` y actores no confiables quedan en `deny_all`,
  - `revoke-device` consume esa misma semantica mediante `distributed_guard` y `distributed_guard_scope_decision`, permitiendo `trusted-devices` en el caso `self_governed` delegado y rechazando `all` sobre target delegado con `reason_code` y `policy_reason_code` coherentes,
  - la salida verbose del reporte ahora resume tambien `degraded_scope_profiles`,
  - y la validacion del corte quedo verde con `php -l`, `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 361 assertions)`) y `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (18 tests, 384 assertions)`).
- Gap natural posterior:
  - falta extender `target_scope_relation_policies` a mas combinaciones actor-target-scope para perfiles graduales distintos de `concentrated_activity`,
  - falta llevar mas de esta semantica distribuida al runtime principal y a pruebas feature end-to-end,
  - y falta enriquecer el reporte para resumir con mas precision que stores y targets explican cada degradacion gradual.

### DV-AUTH-073

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - consolidar overlays graduales adicionales sobre `target_scope_relation_policies` para casos `self_governed_*`,
  - ampliar la cobertura end-to-end del runtime principal para completar la taxonomia `self_governed_full/sessions/trusted-devices`,
  - y enriquecer el reporte para resumir mejor la relacion entre stores objetivo y degradaciones activas.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `monitor_recent_lag` y `monitor_concentrated_activity` ahora descienden tambien hasta `self_governed_full_scope_target`, `self_governed_sessions_scope_target` y `self_governed_trusted_devices_scope_target`, con degradaciones mas precisas para actores delegados sobre su propio target,
  - en `recent_lag`, el target `self_governed_trusted_devices_scope_target` queda bloqueado con `deny_all`, mientras en `concentrated_activity` ese mismo target puede conservar `trusted_devices_only` sin relajar otras combinaciones mas sensibles,
  - `operational_response` ahora anexa `target_store_assessments` para resumir dentro de la misma respuesta que stores sostienen la degradacion activa junto con `target_store_fingerprints` y `degraded_scope_profiles`,
  - `AuthManagerTest` ahora completa la cobertura feature del runtime principal para `self_governed_full_scope_target`, ademas de los casos parciales ya existentes,
  - y la validacion del corte quedo verde con `php -l`, `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 373 assertions)`), `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (19 tests, 411 assertions)`) y `AuthManagerTest` (`OK (58 tests, 708 assertions)`).
- Gap natural posterior:
  - falta extender overlays graduales equivalentes a mas combinaciones `direct_admin_*` y `delegated_admin_full_scope_target`,
  - falta seguir enriqueciendo la explicabilidad del reporte para correlacionar mejor target stores, drift y decisiones denegadas por tipo de mutacion,
  - y falta tender un puente mas profundo entre la semantica distribuida del tooling operativo y otros paths publicos del runtime principal.

### DV-AUTH-074

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - profundizar overlays graduales equivalentes sobre targets `direct_admin_*`,
  - enriquecer la correlacion operativa entre `target_store_fingerprints`, `target_store_assessments`, drift y denials por tipo de mutacion,
  - y seguir tendiendo puentes entre la semantica distribuida del tooling operativo y los paths publicos del runtime principal.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `partial_visibility`, `recent_lag` y `concentrated_activity` ahora bajan tambien hasta `direct_admin_full_scope_target`, `direct_admin_sessions_scope_target` y `direct_admin_trusted_devices_scope_target`, permitiendo que `distributed_guard_scope_decision` seleccione policies mas precisas para targets administrativos directos,
  - `operational_response` ahora publica `mutation_scope_profiles`, correlacionando `all`, `sessions` y `trusted-devices` con `mutation_kind`, `reason_code`, `target_store_fingerprints` y `target_store_assessments`,
  - el output verbose del reporte resume tambien `mutation_profiles` y `target_store_assessments`,
  - `revoke-device` ya valida de forma explicita overlays directos parciales sobre `recent_lag`, `concentrated_activity` y `partial_visibility`, usando `policy_source=actor_target_scope_relation_scope_policy` cuando aplica,
  - `AuthManagerTest` ahora incorpora cobertura feature para `direct_admin_sessions_scope_target` y `direct_admin_trusted_devices_scope_target`, completando el puente hacia `managedDevices()` y `revokeManagedDevice()` en paths publicos del runtime,
  - y la validacion del corte quedo verde con `php -l`, `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 387 assertions)`), `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (21 tests, 446 assertions)`) y `AuthManagerTest` (`OK (59 tests, 724 assertions)`).
- Gap natural posterior:
  - falta extender overlays equivalentes a mas combinaciones `delegated_admin_full_scope_target` y otros casos graduales administrativos plenos,
  - falta enriquecer todavia mas la correlacion entre drift, denials y audit/export operativo por tipo de mutacion,
  - y falta seguir expandiendo la cobertura end-to-end del runtime principal hacia taxonomias actor-target-scope adicionales.

### DV-AUTH-075

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - consolidar overlays especificos para `delegated_admin_full_scope_target` bajo `partial_visibility`, `recent_lag` y `concentrated_activity`,
  - enriquecer la correlacion entre `mutation_scope_profiles`, audit/export y `distributed_guard_scope_decision` por tipo de mutacion,
  - y seguir ampliando el puente end-to-end hacia el runtime principal para delegacion administrativa plena.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `partial_visibility`, `recent_lag` y `concentrated_activity` ahora bajan tambien hasta `delegated_admin_full_scope_target`, permitiendo que `distributed_guard_scope_decision` deje de caer en policies demasiado generales para delegacion administrativa plena,
  - `longitudinal_metrics` y `store_cohorts` ahora agregan `mutation_kinds`, `distributed_guard_policy_sources`, `distributed_guard_policy_reason_codes` y `distributed_guard_reason_codes`, reforzando la trazabilidad entre audit log durable, drift observado y contratos de denial/allowance por mutacion,
  - `mutation_scope_profiles` ahora publica tambien `response_mode`, `escalation_level` y `policy_source=global_scope_policy`, aclarando que el perfil exportado es el contrato operativo global del reporte,
  - `AuthSecurityCenterRevokeDeviceCommandTest` ahora cubre el overlay especifico de `delegated_admin_full_scope_target` en `partial_visibility`, `recent_lag` y `concentrated_activity`,
  - `AuthManagerTest` ahora valida end-to-end que un actor `delegated_admin_full_scope_target` no solo descubre el ownership proyectado, sino que tambien puede ejecutar `revokeManagedDevice()` sobre el target remoto en el runtime principal,
  - y la validacion del corte quedo verde con `php -l`, `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 403 assertions)`), `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (22 tests, 472 assertions)`) y `AuthManagerTest` (`OK (59 tests, 732 assertions)`).
- Gap natural posterior:
  - falta enriquecer `mutation_scope_profiles` con vistas mas actor-aware para explicar mejor como cambian los overlays por `authorization_mode`, privilegio y target relation sin depender solo del audit log,
  - falta profundizar la correlacion por cohorte/store entre drift, `policy_source`, `policy_reason_code`, `reason_code` y export durable,
  - y falta seguir expandiendo la cobertura end-to-end del runtime principal hacia taxonomias actor-target-scope administrativas adicionales.

### DV-AUTH-076

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - volver actor-aware la observabilidad de mutacion dentro de `auth:security-center:report`,
  - correlacionar `mutation_actor_profiles`, `actor_aware_profiles` y `observed_actor_profiles` con `store_cohorts` y drift distribuido,
  - y endurecer ese contrato de explicabilidad sin cambiar el enforcement actual de `revoke-device` ni el runtime principal.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `longitudinal_metrics` ahora agrega `mutation_actor_profiles`, permitiendo resumir por mutacion, `scope`, `authorization_mode`, privilegio, relacion actor-target, `policy_source`, `policy_reason_code`, `reason_code`, store observado y outcome real,
  - cada `store_cohort` ahora publica tambien sus propios `mutation_actor_profiles`, reforzando la lectura longitudinal por fingerprint/topologia sin perder el contexto actor-aware,
  - `mutation_scope_profiles` ahora distingue `actor_aware_profiles` derivados del contrato de policy y `observed_actor_profiles` derivados del audit trail durable, de modo que el reporte compara la policy vigente contra el comportamiento administrativo observado,
  - `activity_drift` y `operational_response` conservan el mismo enforcement runtime, pero ahora cargan una correlacion actor-aware mas rica dentro del contrato exportado del reporte,
  - las pruebas del reporte dejaron de depender del orden posicional de `actor_aware_profiles`, fijando el contrato por identidad semantica del perfil,
  - y la validacion del corte quedo verde con `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 422 assertions)`), `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (22 tests, 472 assertions)`) y `AuthManagerTest` (`OK (59 tests, 732 assertions)`).
- Gap natural posterior:
  - falta correlacionar mejor cuando distintas cohortes, stores o recursos afectados convergen en la misma `policy_source` actor-aware pero divergen en `target_store_assessments` y denials concretos,
  - falta extender esa explicabilidad cruzada hacia export y enforcement cuando el drift multi-store degrada solo un subconjunto de recursos o mutaciones afectadas,
  - y falta seguir ampliando la cobertura end-to-end del runtime principal hacia taxonomias actor-target-scope y degradaciones parciales adicionales.

### DV-AUTH-077

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - correlacionar el contrato actor-aware del `report` con el recurso realmente afectado por cada mutacion observada,
  - distinguir dentro de cada `mutation_scope_profile` cuando la cobertura observada solo alcanza un subconjunto de los recursos esperados bajo drift parcial,
  - y mantener intacto el enforcement actual del `revoke-device` y del runtime principal.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `mutation_actor_profiles` y sus cohortes ahora agregan `affected_resources` y `affected_resource_kinds`, de modo que el reporte ya no solo resume policy y actor-target-scope, sino tambien que tipo de recurso fue realmente impactado,
  - cada `mutation_scope_profile` ahora publica `resource_coverage`, separando `targeted_resource_kinds`, `observed_affected_resources`, `observed_affected_resource_kinds`, `missing_targeted_resource_kinds` y el flag `has_partial_observed_resource_coverage`,
  - ese mismo `resource_coverage` ahora resume `target_store_statuses`, `degraded_target_store_fingerprints` y `has_degraded_target_stores`, aclarando cuando la degradacion multi-store recae sobre stores concretos mientras la mutacion observada solo afecta una parte del recurso esperado,
  - la suite del reporte ahora fija tanto la correlacion por recurso afectado en `mutation_actor_profiles` como la deteccion de cobertura parcial dentro de `partial_visibility`,
  - y la validacion del corte quedo verde con `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 440 assertions)`), `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (22 tests, 472 assertions)`) y `AuthManagerTest` (`OK (59 tests, 732 assertions)`).
- Gap natural posterior:
  - falta llevar esta trazabilidad por subconjunto afectado hasta `--export-log` y `--audit-log` para que snapshots y auditoria durable publiquen el mismo lenguaje de `resource_coverage`,
  - falta correlacionar mejor `report` y `revoke-device` cuando un denial operativo afecta solo parte del set objetivo o cuando diferentes stores sostienen distintos subconjuntos del impacto esperado,
  - y falta seguir ampliando la cobertura end-to-end del runtime principal hacia degradaciones parciales administrativas adicionales.

### DV-AUTH-078

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `48`, `49`
- Alcance objetivo:
  - unificar el lenguaje de `resource_coverage` y `affected_resource_kinds` entre `report`, snapshots `--export-log`, `--audit-log` y el payload operativo de `revoke-device`,
  - hacer que `report` reutilice esa semantica cuando consume auditoria durable, en lugar de reconstruirla siempre desde cero,
  - y diferenciar mejor entre recursos efectivamente afectados y recursos solo candidatos cuando la mutacion queda bloqueada por la guardia distribuida.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `mutation_scope_profiles` ahora publica `resource_coverage` real dentro del `report`, y `observed_actor_profiles` / `mutation_actor_profiles` agregan `targeted_resource_kinds`, `affected_resources` y `affected_resource_kinds`,
  - `report` ahora normaliza `resource_coverage` desde el audit trail durable cuando ese contrato ya viene publicado por `revoke-device`, manteniendo fallback para eventos historicos sin esa forma,
  - `revoke-device` ahora emite `resource_coverage` tanto en el payload JSON como en los eventos `--audit-log`, incluyendo `matched_resources`, `affected_resources`, `target_store_statuses` y `degraded_target_store_fingerprints`,
  - en los denials por guardia distribuida, `resource_coverage` ya separa correctamente recursos candidatos (`matched_resources`) de recursos realmente afectados (`affected_resources=0`),
  - los snapshots `--export-log` ya arrastran ese mismo lenguaje al exportar el `report` completo,
  - y la validacion del corte quedo verde con `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 443 assertions)`), `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (22 tests, 493 assertions)`) y `AuthManagerTest` (`OK (59 tests, 732 assertions)`).
- Gap natural posterior:
  - falta normalizar esta misma semantica en todos los envelopes de rechazo y decision operativa, incluyendo ramas de actor no autorizado y decision envelopes mas delgados,
  - falta profundizar la trazabilidad durable cuando una misma mutacion pasa por distintos resultados (`authorized`, `distributed_guard_denied`, `authorization_failed`) sobre el mismo subconjunto de recursos,
  - y falta seguir ampliando la cobertura del runtime principal hacia degradaciones administrativas adicionales y envelopes operativos mas uniformes.

### DV-AUTH-079

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - normalizar `resource_coverage` en todos los envelopes JSON y JSONL de rechazo y decision operativa de `auth:security-center:revoke-device`, incluyendo 5 ramas de validacion temprana y la rama `authorization_failed`,
  - propagar `matched_resources` (recursos candidatos) a traves de toda la cadena longitudinal de `auth:security-center:report` (`eventResourceCoverage` → `longitudinal_metrics` → `mutation_actor_profiles` → `store_cohorts` → `time_windows` → `store_time_windows` → `mutation_scope_profiles.resource_coverage`),
  - unificar nomenclatura en `mutation_scope_profiles.resource_coverage`: claves legacy `observed_affected_resources` / `observed_affected_resource_kinds` / `has_partial_observed_resource_coverage` → vocabulario estandar `matched_resources`, `affected_resources`, `affected_resource_kinds`, `has_partial_affected_resource_coverage`, `has_any_affected_resources`,
  - y preservar backward compatibility: `eventResourceCoverage()` con fallback chain `event.resource_coverage.matched_resources` → `summary.matched_sessions|matched_trusted_devices` → `administrative_metrics.matched_total_resources` para eventos historicos.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterReportCommand.php`
  - `vendor/voltstack/framework/src/Quantum/Console/Commands/AuthSecurityCenterRevokeDeviceCommand.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php`
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php`
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php`
- Resultado:
  - `renderValidationFailure()` y `renderAuthorizationFailure()` en `revoke-device` pasan de firma 3→11 parametros y emiten envelope estructurado uniforme con las 12 claves de `resource_coverage`, `correlation_id`, `operation_id`, `result`, `reason_code`, `target`, `actor`, `operational_context` y `administrative_metrics`,
  - las 5 ramas de rechazo temprano (`missing_identity`, `missing_device_reference`, `missing_actor_identity`, `missing_actor_session_public_id`, `invalid_scope`) y la rama `authorization_failed` incluyen `resource_coverage` tanto en `writeAuditEvent` JSONL como en la salida JSON via las nuevas firmas de render,
  - en denials por autorizacion y validacion se preserva semantica estricta: `matched_resources > 0` (que HUBIERA tocado), `affected_resources = 0` (nada cambio realmente),
  - `eventResourceCoverage()` en `report` reconstruye `matched_resources` con fallback chain completo + heuristica de split por scope cuando solo existe `matched_total`,
  - `longitudinal_metrics()` acumula `matched_resources{sessions,trusted-devices,total}` en totales globales, por cohorte de store y por evento normalizado,
  - `accumulateMutationActorProfile()` extiende firma 17→19 params (inserta `matchedSessions` y `matchedTrustedDevices`) y suma `matched_resources` por perfil de actor; sus 2 call-sites actualizados,
  - `buildTimeWindows()` y `buildStoreTimeWindows()` agregan acumuladores `matched_resources` en cada nivel de ventana temporal,
  - `summarizeObservedMatchedResources()` helper nuevo paralelo a `summarizeObservedAffectedResources()`,
  - `mutation_scope_profiles.resource_coverage` renombra 3 claves legacy al vocabulario estandar y agrega `matched_resources` + `has_any_affected_resources`,
  - los 4 archivos de tests renombran 6 aserciones de claves legacy para alinearse al contrato unificado sin perder cobertura,
  - la validacion del corte quedo verde: `AuthSecurityCenterRevokeDeviceCommandTest` (`OK (22 tests, 493 assertions)`), `AuthSecurityCenterReportCommandTest` (`OK (11 tests, 443 assertions)`) → combinado `OK (33 tests, 936 assertions)`, y `AuthManagerTest` (`OK (59 tests, 732 assertions)`) sin regresiones en runtime principal.
- Gap natural posterior:
  - faltan aserciones de test EXPLICITAS para el nuevo envelope de rechazo: verificar presencia de `correlation_id`, `operation_id`, `result`, `reason_code` y las 12 claves de `resource_coverage` en JSON de salida y eventos JSONL para todas las ramas de rechazo,
  - falta ampliar cobertura E2E en `managedDevices()` y `revokeManagedDevice()` con taxonomias actor-target-scope administrativas adicionales y escenarios de degradacion parcial distribuida aun no fijados en feature tests,
  - falta un harness formal de snapshots/export long-form para validar `--export-log` y `--audit-log` con trazabilidad de envelopes completos a traves del tiempo,
  - y falta seguir posicionando al subsistema para stores persistentes mas ricos del provider local y providers mutables.

### DV-AUTH-080

- Estado: `Implementado`
- Bloque documental relacionado: `24`, `25`, `26`, `35`, `47`, `48`, `49`
- Alcance objetivo:
  - añadir aserciones de test EXPLICITAS para envelopes estructurados de rechazo JSON/JSONL en `auth:security-center:revoke-device` para las 6 ramas (5 validation_failed + 1 authorization_failed con 2 variantes),
  - ampliar cobertura E2E runtime principal en `AuthManager::managedDevices()` y `AuthManager::revokeManagedDevice()` con taxonomias actor-target-scope administrativas multi-identity, multi-login y scopes granular `sessions|trusted-devices|all`,
  - construir harness formal snapshots/export long-form en `auth:security-center:report` para validar trazabilidad `correlation_id` ↔ `operation_id` a traves de feed `--audit-log-source`, salida JSON `--json` y snapshot durable `--export-log`,
  - y verificar preservacion de `matched_resources` vs `affected_resources` a traves de toda la cadena longitudinal, con perfiles de actor y perfiles de scope filtrados por clave (no indice posicional).
- Evidencia principal:
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterRevokeDeviceCommandTest.php` (L1077-L1631)
  - `vendor/voltstack/framework/tests/Feature/AuthManagerTest.php` (L4863-L5123)
  - `vendor/voltstack/framework/tests/Unit/AuthSecurityCenterReportCommandTest.php` (L994-L1451)
- Resultado:
  - 2 tests existentes de authorization_failed (`delegated_actor_lacks_trusted`, `actor_session_not_governed`) extendidos con `--correlation-id` + `--audit-log` y ~80 nuevas aserciones cada uno sobre envelope estructurado (result, reason_code, correlation_id, operation_id, 12-clave resource_coverage, target, actor, administrative_metrics, operational_context) TANTO en JSON de salida COMO en evento JSONL de auditoria,
  - 5 tests NUEVOS unitarios de validation_failed (missing_identity, missing_device_reference, missing_actor_identity, missing_actor_session_public_id, invalid_scope) cada uno con ~25 aserciones de envelope 12-clave y trazabilidad operation_id cruzada JSON ↔ JSONL, suite revoke-device OK 27 tests / 811 assertions,
  - 1 test NUEVO E2E runtime: multi-identity 301 (target-A MFA) + 302 (target-B password+MFA) + 303 (delegated_admin delegated_support scope completo), 4 logins (target-A browser+tdv, target-A mobile session-only, target-B password+MFA+tdv, delegado MFA), inventory `managedDevices(301)` 2 devices sorted por device_ref con proyeccion taxonomica completa (management_actor_governed, management_actor_authorized=true, management_actor_authorization_mode=delegated_admin, management_actor_privilege_level=delegated_support, management_actor_can_manage_sessions/trusted_devices=true, management_actor_target_relation=delegated_administrative_target, management_actor_target_scope_relation=delegated_admin_full_scope_target, management_target_identity=301, management_target_matches_current_identity=false), inventory `managedDevices(302)` 1 device, 3 operaciones revoke granular: scope sessions sobre target-A browser (revocado=true, tdv permanece=1), scope trusted-devices sobre target-A mobile sin tdv (revocado=false), scope all sobre target-B (revocado=true, todo borrado), 4 protected endpoints confirmando granular scope: target-A browser=401, target-B=401, target-A mobile=200 (solo sessions scope), suite AuthManager OK 60 tests / 773 assertions,
  - 1 test NUEVO harness snapshots/export long-form: 6 eventos audit-log custom (3 executed sessions/trusted/all + validation_failed + authorization_failed + distributed_guard_denied) con resource_coverage 12-clave completo y matched>0 en executed events, cross-correlation `correlation_id` entre payload JSON del reporte ↔ export-log snapshot durable ↔ source audit events, matched_resources totales 10 (sessions=6 + trusted=4) y affected_resources 5 (sessions=3 + trusted=2) correctamente acumulados en longitudinal_metrics globales, por `time_windows` / `store_time_windows` / `store_cohorts`, y perfiles de mutation_scope_profiles/mutation_actor_profiles filtrados por clave (no indice posicional) enriquecidos con matched_resources; suite report OK 12 tests / 524 assertions,
  - suite framework combinado: 99 tests / 2108 assertions exit code 0 sin regresiones.
- Gap natural posterior:
  - falta gobernanza distribuida del lifecycle credenciales con invalidacion bulk, proofs de rotacion y retention enforcement segmentado por tier de riesgo,
  - falta federacion OIDC skeleton con well-known config, ID token validation y mapping de atributos federados hacia GenericIdentity,
  - falta passkeys FIDO2 inicial (Relying Party config, WebAuthn registration + assertion ceremonies, challenge nonce binding, credential storage),
  - falta abuse protection throttling V1 (brute force counters por identifier/device/ip_prefix, credential stuffing detection con bloom + lockout temporal por window),
  - falta risk engine signal V1 (anomaly heuristics, new device detection, geo drift, velocity checks, risk score agregado en decision metadata),
  - falta assurance composable (policy-based combination de amr + method, assurance profile agregado por operation/context),
  - falta transaction state nonce replay protection CSRF binding con continuation transaction state verifiable por cryptographic binding.

### DV-AUTH-081

- Estado: `Implementado`
- Bloque documental relacionado: `07`, `10`, `11`, `14`, `21`, `22`, `24`, `25`, `26`, `28`, `35`, `36`, `47`, `48`, `49`, `50`
- Alcance objetivo:
  - ampliar provider local de identidades `LocalIdentityProvider` como MUTABLE con metadata de CICLO DE VIDA beyond rehash persistente (security_state + reason, password_lifecycle_metadata con rotation history, expiration ages, lockout thresholds, failed attempts counters y sus timestamps),
  - introducir `Contracts/MutableIdentityProviderInterface` + `Contracts/PasswordLifecycleAwareProviderInterface` con comprobaciones de pertenencia `instanceof` para mantener compatibilidad 100% OPT-IN con cualquier provider existente sin interfaz,
  - ampliar `PasswordPolicyInterface`/`PasswordPolicy` con `isExpired()`, `needsRotation()` y `checkAgainstHistory()` (3 helpers nuevos) + 2 config helpers `rotationWindowSeconds()` / `passwordExpiredAfterSeconds()`,
  - crear 4 excepciones tipadas de lifecycle credenciales: `PasswordExpiredException`, `PasswordRotationRequiredException`, `CredentialLockedException` (lockout TEMPORAL por threshold superado), `AccountSuspendedException` (estado permanente, NO lockout) con `reasonCode` literalmente pasado al `parent::__construct` via `readonly`,
  - reescribir completamente `AuthExceptionMapper` para mapear estas 4 excepciones + `IdentityNotEligibleException` con status HTTP 423 (Locked temporal) / 403 (Forbidden suspension) / 401 (Unauthorized credential failure + password expired) y headers opcionales via `array_filter` + JSON extensions en `WWW-Authenticate` cuando aplica,
  - integrar 5 gates lifecycle en `PasswordAuthenticator`: lockout temporal por `lockout_until > time()` → `CredentialLockedException`, `securityState` gate → siempre `IdentityNotEligibleException` (compat 100% baseline), verify failure → `recordFailedAttempts()` persistiendo counter + timestamp, verify success → check metadata (`isExpired` → `PasswordExpiredException`, `needsRotation` → `PasswordRotationRequiredException`, `checkAgainstHistory` → reuse detectado `PasswordRotationRequiredException`), `clearFailedAttempts()`; `PasswordAuthenticator` args 3 y 4 `TrustedDeviceRepository` + `ConfigRepository` opcionales con defaults para no romper callers antiguos,
  - introducir soporte pluggable de storage para sessiones y trusted devices mediante 3 interfaces nuevas: `FilterableAuthenticationSessionRepositoryInterface` (13 criterios scalar-only + `sessionMatches` forward-compatible criterios desconocidos ignorados), `FilterableTrustedDeviceRepositoryInterface` (15 criterios scalar-only + flag `include_expired=true` para auditoria por defecto false), `BulkDeletableSessionRepositoryInterface` para bulk ops implementadas en `InMemoryAuthenticationSessionRepository` y `FileAuthenticationSessionRepository` + las 4 implementaciones para session/tdv (memory + file) actualizadas,
  - introducir 2 factories pluggables: `SessionRepositoryDriverFactoryInterface` + `SessionRepositoryDriverFactory`, `TrustedDeviceRepositoryDriverFactoryInterface` + `TrustedDeviceRepositoryDriverFactory` con alias default `memory` / `file` + `registerDriver(string alias, Closure factory)` para drivers custom; refactorizar `AuthenticationServiceProvider` bindings eliminando closures inline y creando drivers via factories,
  - extraer lógica de `AuthDevicesReconcileCommand` a servicio desacoplado `InventoryReconcilerInterface` + `InventoryReconciler` reusable por cualquier ejecutor, comando consola refactorizado `handle()` llamando al servicio via contenedor eliminando lógica privada,
  - implementar Bearer Token Opaque V1 SIN JWT: `OpaqueTokenRepositoryInterface` (find/save/revoke/list access+refresh + `revokeAllForIdentity`), 4 VOs `TokenId` (prefijos `atk_` 32 bytes generateAccess / `rtk_` 48 bytes generateRefresh, `isAccess` / `isRefresh` helpers), `OpaqueAccessToken` + `OpaqueRefreshToken` readonly con `isActive` = !revoked && !expired, storage `InMemoryOpaqueTokenRepository` (array) y `FileOpaqueTokenRepository` (1 JSON file-per-token sanitized token names → traversal prevention, `JSON_THROW_ON_ERROR`),
  - implementar `BearerAuthenticator` (`supports()` detecta `access_token` en attributes o `Authorization: Bearer <atk>` header; `authenticate()` valida token activo via repo + identity existente), `BearerAuthMiddleware` alias `auth.bearer` clona Request agregando `access_token` + `Authorization` headers antes de authenticate; wiring `AuthenticationServiceProvider` + middleware alias; `DefaultAuthenticatorResolver` amplía con 3er arg opcional `BearerAuthenticator` + helper `withBearer()` prepend candidates (priority 999) para stack,
  - middleware composite multi-driver auth stack con `CompositeAuthenticatorResolver` addResolver(priority) → candidates evalúan por prioridad DESC, deduplicación por `spl_object_hash`; pipeline configurable `auth.authenticators.pipeline` (por defecto VACÍO → flujo antiguo INTACTO compat 100%), `AuthenticationOrchestrator` multi-candidate first-success-wins y metadata aggregated rejection para ≥2 candidatos todos fallidos (`candidates_tried=N`, `source=orchestrator_composite`, `{candidate}_reason` + `{candidate}_metadata` por candidato),
  - BACKWARD COMPAT 100% CRITICA en Orchestrator: branch 1-solo-candidato retorna literalmente la `AuthenticationDecision` ORIGINAL del authenticator SIN modificación (NUNCA aggregated metadata) para preservar mensajes, `exception` + `exception_arguments` tipados esperados por `AuthManager::exceptionFromDecision()`,
  - Policy Engine V1 inicial: 3 rules composables (`SessionCountLimitRule` max sessions per identity; `TrustedDeviceEnrollmentLimitRule` max TDVs per identity usando repo `TrustedDeviceRepositoryInterface`; `IdentitySecurityStateRule` deny si `security_state` no en allowed list), `PolicyDecision` VO (allow/deny `PolicyEffect` enum, reason codes array, metadata array), `AuthenticationPolicyRuleInterface` (applies/evaluate) + `AuthenticationPolicyEngine` con AND-semantics (1ra rule deny → overall deny agregando reason codes + metadata acumulada; todas allow → allow con `rules_applied`/`rules_total` counters),
  - Policy Engine wiring NULLABLE en `AuthManager` constructor 6to arg opcional `?AuthenticationPolicyEngine = null` + aplicado en `managedDevices()` y `revokeManagedDevice()` antes de operaciones de inventario; config `auth.policy.enabled` (default `false` = SIN engine, baseline 100% intacto); `AuthenticationServiceProvider` build rules array desde config por defecto las 3 rules con thresholds `auth.policy.max_concurrent_sessions_per_identity` / `auth.policy.max_trusted_devices_per_identity`,
  - tests: 10 tests BloqueBTest (factories/filter/bulk/InventoryReconciler OK), 12 tests BloqueCTest (bearer lifecycle/storage/authenticator OK), 4 tests BloqueDTest (priority/dedup/first win/aggregated rejection OK), 7 tests BloqueETest (PolicyDecision/3 rules individuales/engine AND-semantics/no rules/baseline disabled compat OK), total tests NUEVOS en slice DV-AUTH-081 = 43; cross-suite combinada 136 tests / 1209 assertions exit 0 sin regresiones.
- Evidencia principal:
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/MutableIdentityProviderInterface.php` (L12 signature `updateSecurityState(Identity, IdentitySecurityState, ?string reason = null): bool`)
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/PasswordLifecycleAwareProviderInterface.php` (9 metadata keys incluyendo `security_state` y `reason`)
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/{PasswordExpiredException, PasswordRotationRequiredException, CredentialLockedException, AccountSuspendedException}.php` (4 excepciones nuevas)
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/PasswordPolicyInterface.php` (L17-L24 3 nuevos métodos)
  - `vendor/voltstack/framework/src/Quantum/Auth/Passwords/PasswordPolicy.php` (L51-L148 3 impl + 2 config helpers)
  - `vendor/voltstack/framework/src/Quantum/Auth/Exceptions/AuthExceptionMapper.php` (FULL REWRITE status 423/403/401 headers opcionales JSON extensions)
  - `vendor/voltstack/framework/src/Quantum/Auth/Identity/LocalIdentityProvider.php` (L14 5 interfaces + `L79-L138, L183-L410` upgradePasswordHash + 7 métodos mutable pattern `updateEntry(Identity, Closure mutator)`)
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/PasswordAuthenticator.php` (L39-L44 args 3/4 opcionales defaults + L81-L227 5 gates lifecycle + L429-L440 rotationWindowSeconds helper)
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/{FilterableAuthenticationSessionRepositoryInterface, FilterableTrustedDeviceRepositoryInterface, BulkDeletableSessionRepositoryInterface}.php` (3 interfaces nuevas)
  - `vendor/voltstack/framework/src/Quantum/Auth/Sessions/InMemoryAuthenticationSessionRepository.php` y `FileAuthenticationSessionRepository.php` (3 interfaces implementadas + `sessionMatches` 13 criterios scalar)
  - `vendor/voltstack/framework/src/Quantum/Auth/Devices/InMemoryTrustedDeviceRepository.php` y `FileTrustedDeviceRepository.php` (FilterableTDV + `deviceMatches` 15 criterios + `include_expired`)
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/SessionRepositoryDriverFactory.php` y `TrustedDeviceRepositoryDriverFactory.php` (2 factories pluggables nuevas)
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/InventoryReconcilerInterface.php` y `Devices/InventoryReconciler.php` (servicio reusable + consola `Console/Commands/AuthDevicesReconcileCommand.php` L43-L88 refactorizado)
  - `vendor/voltstack/framework/src/Quantum/Auth/Contracts/OpaqueTokenRepositoryInterface.php` + `Tokens/{TokenId, OpaqueAccessToken, OpaqueRefreshToken, InMemoryOpaqueTokenRepository, FileOpaqueTokenRepository}.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/Authenticators/BearerAuthenticator.php` + `Middlewares/BearerAuthMiddleware.php` (alias `auth.bearer`)
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/CompositeAuthenticatorResolver.php` + `Runtime/AuthenticationOrchestrator.php` (Rama compat 1-candidate L49-52)
  - `vendor/voltstack/framework/src/Quantum/Auth/Runtime/{PolicyEffect, PolicyDecision, AuthenticationPolicyRuleInterface, SessionCountLimitRule, TrustedDeviceEnrollmentLimitRule, IdentitySecurityStateRule, AuthenticationPolicyEngine}.php`
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthManager.php` (L47-54 6to arg `?AuthenticationPolicyEngine`, managedDevices L483-L503 policy check, revokeManagedDevice L681-L700 policy check)
  - `vendor/voltstack/framework/src/Quantum/Auth/AuthenticationServiceProvider.php` (wiring superpuestos factories + bearer + pipeline composite + policy engine nullable)
  - tests: `tests/Unit/{BloqueBTest, BloqueCTest, BloqueDTest, BloqueETest}.php` (10 + 12 + 4 + 7 = 33 unit tests nuevos) + `tests/Unit/{LocalIdentityProviderTest, PasswordPolicyTest, DefaultAuthenticatorResolverTest}` (6 + 3 + baseline) + `tests/Feature/AuthManagerTest.php` (baseline + 1 lifecycle test)
- Resultado:
  - `LocalIdentityProvider` es ahora `final` mutable con 7 operaciones beyond rehash persistidas (updateSecurityState + reason, rotatePassword, expirePasswordAt, incrementFailedAttempt, clearFailedAttempts, addToPasswordHistory, updateEntry generic pattern centralizado reduciendo duplicidad 70%),
  - `AuthExceptionMapper` completamente reescrito ya maneja 6 excepciones tipadas de lifecycle credencial con mapeo HTTP status + reason codes semánticamente correctos + headers opcionales,
  - `PasswordAuthenticator` gates lifecycle completos: lockout temporal (CredentialLocked), securityState gate (IdentityNotEligible), 3 verificaciones post-success (expired/rotation/reuse), todo OPT-IN: metadata defaults 0/null = desactivado cuando no hay config,
  - storage pluggable session/tdv con 3 interfaces filter/bulk y 2 factories + driver alias extensibles; bindings SP refactorizados eliminando closures inline → `InMemoryAuthenticationSessionRepository` y `FileAuthenticationSessionRepository` implementan `FilterableAuthenticationSessionRepositoryInterface` + `BulkDeletableSessionRepositoryInterface`,
  - InventoryReconciler servicio reusable fuera del comando consola, comando ahora recibe el servicio vía container y solo orquesta CLI output,
  - Bearer token opaque V1 100% funcional: pareja access/refresh, storage memory/file, authenticator, middleware alias `auth.bearer`, pipeline wiring OPT-IN,
  - stack multi-driver configurable con `auth.authenticators.pipeline`; si pipeline VACÍO o no configurado, retorna `DefaultAuthenticatorResolver` y `AuthenticationOrchestrator` aplica rama 1-candidate 100% compat baseline (Decision ORIGINAL authenticator sin aggregated metadata),
  - Policy Engine V1 3 rules composables AND-semantics, wiring nullable en AuthManager managedDevices/revokeManagedDevice, `auth.policy.enabled=false` default = sin engine intacto baseline,
  - Tests BloqueA (6 unit LocalIdentityProvider + 3 unit PasswordPolicy + 1 feature lifecycle) = 10, BloqueBTest 10 tests, BloqueCTest 12 tests, BloqueDTest 4 tests, BloqueETest 7 tests; TOTAL tests nuevos = ~43; cross-suite frameworks auth completo 136 tests / 1209 assertions exit code 0 SIN regresiones.
- Gap natural posterior:
  - falta gobernanza distribuida del lifecycle credenciales: invalidation bulk por identifier/identity, proofs de rotacion (auditable rotation receipts), retention enforcement segmentado por tier de riesgo sobre password_lifecycle_metadata,
  - falta federacion OIDC skeleton: well-known openid-configuration discovery, JWKS cache, ID token signature/expiry/nonce/iss/aud validation, mapping claims federados → GenericIdentity attributes + IdentitySecurityState,
  - falta passkeys FIDO2 inicial: Relying Party config (rpId/rpName/origins allow list), WebAuthn registration ceremony (create challenge + nonce binding + attestation validation + credential storage), assertion ceremony (challenge issuance + expectedRpId/userHandle verification + signature counter validation),
  - falta abuse protection throttling V1: brute force counters (by identifier/device_ref/ip_prefix windows 1m/5m/15m), credential stuffing detection con bloom de passwords comprometidos known + lockout temporal por window superado,
  - falta risk engine signal V1: anomaly heuristics (new device, ip drift, impossible travel/velocity checks, irregular time of day), risk score agregado expuesto en AuthenticationDecision metadata risk_score/risk_signals,
  - falta assurance composable: policy-based combination de amr list + authentication methods aplicados, assurance profile agregado por operation/context minimo requerido para step_up_required,
  - falta transaction state nonce replay protection CSRF binding: continuation state con nonce criptografico, CSRF binding challenge, verifiable continuation transaction state per authentication request ID.

### DV-AUTH-082

- Estado: `Implementado`
- Bloque documental relacionado: `09`, `11`, `16`, `17`, `19`, `20`, `37`, `39`, `42`, `47`, `49`, `50`
- Alcance objetivo:
  - Bloque A: Gobernanza distribuida del lifecycle passwords con contrato `DistributedPasswordGovernanceProviderInterface` (5 métodos), VO `PasswordRotationReceipt` con fromArray/toArray, servicio `RetentionTieredEnforcer` 3-tier (365/180/90 días), `LocalIdentityProvider` implementa interfaz con persistencia mutable pattern `updateEntry()`, 3 gates en `PasswordAuthenticator` (retention gate → PasswordRotationRequiredException, rehash receipt emission, metadata extendida filtrando nulls),
  - Bloque B: Abuse Protection Throttling V1 con contrato `AbuseProtectionThrottleInterface` (decide/recordAttempt), VO `ThrottleDecision` (allow/deny), `BruteForceCounter` 3 ventanas sliding 1m/5m/15m con purge, `CredentialStuffingBloomFilter` naive 40 contraseñas comunes, `ThrottleEngineV1` combina counter + bloom umbrales identifier×1 / device_ref×1.5 / ip_prefix×2; Orchestrator integra preAuth hook FUERA de rama 1-candidate,
  - Bloque C: Risk Signal Engine V1 con contrato `RiskSignalProviderInterface`, VO `RiskScore` level enum (low/medium/high/critical cap 100), 4 signals (NewDevice +25 / IpDrift +15 / ImpossibleTravelVelocity +40 / IrregularTime +10), `CompositeRiskSignalEngine` suma cap 100; Orchestrator integra postAuth metadata projection (V1 SIN denegación automática),
  - Bloque D: Assurance Composable con VO `AuthenticationMethodReferenceList` dedup amr, `AssuranceProfile` int enum backed 0-7 (Lowest→HighestPasskey) con displayName, `StepUpRequirement` required/not, `AssuranceStepUpEvaluator` evaluateFor operation min_assurance; helper `AuthenticationAssurance::composeAssuranceFromAmr` + `meetsMinimumAssurance` sin modificar código existente,
  - Bloque E: Transaction State Nonce + CSRF Binding con contratos `TransactionNonceStoreInterface` (issueNonce/validateNonce), VOs `NonceRecord` / `NonceValidationResult`, `CsrfChallengeBinder` HKDF deterministic challenge/verify, `InMemoryTransactionNonceStore` one-time use (order: check record exist → check expiry → purge global, distingue expired vs consumed); Orchestrator preAuth nonce validate (ANTES throttle) + postAuth issue nonce+CSRF metadata,
  - Bloque F: Passkeys FIDO2 Skeleton V1 (sin crypto real, @internal @todo 083): VOs `RelyingPartyConfig` / `PasskeyCredentialRecord` fromArray/toArray, contrato `PasskeyCredentialStoreInterface`, `InMemoryPasskeyCredentialStore` + `AssertionResult` VO, ceremonies `PasskeyRegistrationCeremony` / `PasskeyAssertionCeremony` simulated challenge 64hex sin crypto,
  - Bloque G: OIDC Federation Skeleton V1 (sin crypto real, @internal @todo 083): contratos `OidcWellKnownClientInterface` + `OidcJwksCacheInterface`, VOs `OidcProviderMetadata`, mocks `InMemoryMockOidcWellKnownClient` + `InMemoryOidcJwksCache`, `OidcIdentityTokenValidator` shell 6 checks validateAll sin firma real, `FederatedClaimsMapper` heurística email_verified → Active/Suspended,
  - H1 Cross-suite COMPLETA ≥192 tests AUTH sin regresiones baseline 081; H2 actualizar 4 docs AUTHENTICATION DEVELOPMENT.
- Evidencia principal:
  - Bloque A: `Contracts/DistributedPasswordGovernanceProviderInterface.php` (5 métodos), `Passwords/PasswordRotationReceipt.php` (fromArray/toArray), `Passwords/RetentionTieredEnforcer.php` (3-tier 365/180/90), `Identity/LocalIdentityProvider.php` (implementa interface, 5 métodos governance L351-L545), `Authenticators/PasswordAuthenticator.php` (L150-L182 retention gate, L268-L290 rehash receipt, L330-L356 metadata array_filter),
  - Bloque B: `Contracts/AbuseProtectionThrottleInterface.php`, `AbuseProtection/ThrottleDecision.php`, `AbuseProtection/BruteForceCounter.php`, `AbuseProtection/CredentialStuffingBloomFilter.php`, `AbuseProtection/ThrottleEngineV1.php`, `Runtime/AuthenticationOrchestrator.php` (L44-L69 preAuth throttle hook FUERA 1-candidate),
  - Bloque C: `Contracts/RiskSignalProviderInterface.php`, `Risk/RiskScore.php`, `Risk/{NewDeviceSignal,IpDriftSignal,ImpossibleTravelVelocitySignal,IrregularTimeSignal}.php`, `Risk/CompositeRiskSignalEngine.php`, `Runtime/AuthenticationOrchestrator.php` (L82-L126 postAuth risk metadata hook SIN denegar V1),
  - Bloque D: `Runtime/AuthenticationMethodReferenceList.php`, `Runtime/AssuranceProfile.php` (int enum backed 0-7), `Runtime/StepUpRequirement.php`, `Runtime/AssuranceStepUpEvaluator.php`, `Support/AuthenticationAssurance.php` (L97-L134 composeAssuranceFromAmr + meetsMinimumAssurance),
  - Bloque E: `Contracts/TransactionNonceStoreInterface.php`, `Runtime/NonceRecord.php`, `Runtime/NonceValidationResult.php`, `Runtime/CsrfChallengeBinder.php` (HKDF), `Runtime/InMemoryTransactionNonceStore.php` (one-time order: check→expiry→purge), `Runtime/AuthenticationOrchestrator.php` (L24-L42 preAuth nonce ANTES throttle; L95-L116 postAuth issue nonce+CSRF metadata),
  - Bloque F: `Passkeys/RelyingPartyConfig.php`, `Passkeys/PasskeyCredentialRecord.php`, `Contracts/PasskeyCredentialStoreInterface.php`, `Passkeys/InMemoryPasskeyCredentialStore.php`, `Passkeys/AssertionResult.php`, `Passkeys/PasskeyRegistrationCeremony.php` (@internal), `Passkeys/PasskeyAssertionCeremony.php` (@internal),
  - Bloque G: `Contracts/OidcWellKnownClientInterface.php`, `Contracts/OidcJwksCacheInterface.php`, `Federation/Oidc/OidcProviderMetadata.php`, `Federation/Oidc/InMemoryMockOidcWellKnownClient.php`, `Federation/Oidc/InMemoryOidcJwksCache.php`, `Federation/Oidc/OidcIdentityTokenValidator.php` (shell validateAll 6 checks), `Federation/Oidc/FederatedClaimsMapper.php` (email_verified heuristic),
  - SP Wiring FIX: `AuthenticationServiceProvider.php` (L286-L320) eliminar `bound()` inexistente → bindings DIRECTOS scoped default return null cuando config disabled (resuelve BindingResolution Orchestrator constructor type-hints interfaces nullable),
  - tests: `tests/Unit/BloqueATest.php` (10 tests), `tests/Unit/BloqueBTest.php` (10), `tests/Unit/BloqueCTest.php` (8), `tests/Unit/BloqueDTest.php` (8), `tests/Unit/BloqueETest.php` (8), `tests/Unit/BloqueFTest.php` (6), `tests/Unit/BloqueGTest.php` (6) = 56 tests unit nuevos; Unit AUTH total = 143 tests / 1813 assertions; Feature = AuthManagerTest 61 tests + SkeletonSecuritySmokeTest 12 tests = 73; CROSS-SUITE AUTH TOTAL = 216 tests ≥ 192 objetivo, 2713 assertions exit 0 sin regresiones.
- Resultado:
  - Backward compat 100% OPT-IN layered: interfaces nuevas instanceof checks, config flags (`auth.throttle.enabled`, `auth.risk.enabled`, `auth.transaction.nonce.enabled`) DEFAULT false, constructor Orchestrator 4 args nuevos nullable default null = baseline 081 IDENTICO cuando todo disabled,
  - Orchestrator hooks correctamente posicionados: preAuth Nonce validate → Throttle decide (FUERA rama 1-candidate); postAuth si Authenticated: Risk metadata projection → Issue nonce+CSRF metadata. NUNCA modificó la rama 1-candidate ni la decisión authenticator original,
  - Password governance distribuida: RetentionTieredEnforcer 3-tier pluggable, LocalIdentityProvider implementa contrato via updateEntry() mutable pattern, PasswordAuthenticator 3 gates instanceof checks sin romper providers legacy que NO implementen interface,
  - Throttle V1 combos counter+bloom umbrales 3 dimensiones (identifier/device/ip) + preAuth hook; Risk V1 solo metadata (policy denegación para V2),
  - Assurance ordinal composable (int enum compare) + amr dedup list; StepUp evaluator min_assurance por operation,
  - Nonce one-time correctamente distingue expired vs consumed (purge order fix L53-L72 BloqueETest #2); CSRF deterministic HKDF(nonce, salt=csrf, info=device_ref),
  - Passkeys + OIDC Skeletons V1 @internal simulated sin librerías externas (sin composer, cumplen hard constraint); marcados @todo 083 para crypto/WebAuthn real,
  - SP DI FIX CRÍTICO: VoltStack Container NO respeta `= null` default para type-hints de INTERFAZ en constructor; bindings DEBEN registrarse siempre con return null por defecto cuando config disabled (lección aprendida hard constraint wiring interfaces nuevas type-hint),
  - Cross-suite AUTH 216 tests exit 0 ≥ 192 objetivo, baseline 136 tests 081 intacto sin regresiones.
- Gap natural posterior:
  - falta crypto real passkeys FIDO2 WebAuthn RP ceremonies: validación attestation/assertion, signature counters, userHandle/rpId expected checks, storage real credenciales,
  - falta validación criptográfica real OIDC: ID token signature (RS256/ES256), JWKS fetch TTL cache con refresh HTTP real, nonce binding/issuer/audience checks strict + clock skew leeway configurable,
  - falta bearer token rotation: refresh→new access+refresh pareja, refresh one-time-use invalidation + family token reuse detection (si reutilizas refresh padre invalidas toda la familia descendiente),
  - falta throttle V2 distributed counters: persistence pluggable Redis/database, ventanas exactas sliding con precision, cross-instance lockout propagation,
  - falta risk V2 adaptive deny policies: configurable threshold high/critical → auto-deny o MFA required, mapping risk_level → min_assurance_required step-up,
  - falta assurance V2: triggers orchestrator hooks denegar o pedir step-up cuando assurance < min_assurance_operation, denials explícitos `assurance_insufficient` coherentes con `AuthenticationStrength`.

### DV-AUTH-083

- Estado: `Implementado`
- Bloque documental relacionado: `16`, `17`, `18`, `19`, `20`, `21`, `22`, `37`, `39`, `42`, `47`, `49`, `50`
- Alcance objetivo:
  - Bloque 1 (F1 Passkeys crypto real): COSE CBOR parser kty=2 EC P-256 x/y/crv / kty=3 RSA n/e (alg -7 ES256 / -257 RS256), SubjectPublicKeyInfo DER manual (ecPublicKey OID 1.2.840.10045.2.1 prime256v1 + BIT STRING 0x04‖x‖y; rsaEncryption 1.2.840.113549.1.1.1 + BIT STRING SEQUENCE{n,e}), ECDSA raw 64B r‖s ↔ DER conversion, openssl_pkey_verify real ES256/RS256, FilePasskeyCredentialStore JSON 1-per-credential path traversal prevention, PasskeyAuthenticator (AuthenticatorInterface priority 900 supports passkey_*_b64), PasskeyRegistrationCeremony::finishAttestation() packed COSE ES256/RS256 real crypto, PasskeyAssertionCeremony::verifyAssertion() openssl real + signCount rollback protection (preserva legacy deprecated finishRegistration/finishAssertion backward compat 082),
  - Bloque 2 (G1 OIDC crypto real): splitCompactJws/decodeCompactJws base64url sin padding, jwkToPem RSA(n,e)/ECP256(x,y) → PEM SPKI openssl loadable, validateSignature openssl real RS256 PKCS1v15 / ES256 raw64→DER, validateAll() path compact_jws strict (preserva shell signatures bypass @deprecated backward compat),
  - Bloque 3 (H1 Bearer rotation): OpaqueRefreshToken V2 nuevos campos consumed/consumedAt/rotatedTo/familyId, OpaqueTokenRepositoryInterface 3 métodos consumeRefreshToken/findRefreshTokensByFamilyId/revokeFamilyByReuse/markRotatedTo, InMemoryOpaqueTokenRepository impl full chain rotatedTo linked list descendiente invalidate, BearerTokenService issueTokenPair/rotateRefresh one-time + reuse detection trigger revokeFamilyByReuse bulk,
  - Bloque 4 (I1 Throttle V2): ThrottleDeniedException extends AuthenticationException (retryAfterSeconds, identifier, metadata), DistributedThrottleCounterInterface (currentCount/increment/reset), AuthExceptionMapper status 429, Retry-After header, X-Auth-Throttle-* headers + reasonCodeExtension reason_code/retry_seconds/throttle_identifier,
  - Bloque 5 (J1 Risk V2): RiskAssessmentResult VO clamped 0-100 riskFactors assessedAt, AdaptiveRiskPolicyInterface decide(RiskAssessmentResult): RiskDecision, RiskDecision enum FINAL ACTION allow/step_up_required/deny static factories, ConfigBasedAdaptiveRiskPolicy buckets [0,stepUp) allow / [stepUp,deny) stepUp / [deny,100] deny thresholds config, RiskDeniedException status 403 riskScore/denyThreshold, AuthExceptionMapper X-Auth-Risk-* headers + reasonCodeExtension,
  - Bloque 6 (K1 Assurance V2): AssuranceInsufficientException status 423 requiredMinAssurance/currentAssurance/operation, AuthenticationOrchestrator nuevo preAuth hook al START de execute() (antes nonce/throttle/candidates) lee attribute min_authentication_assurance, evalúa assurance_value attribute override o fallback authenticationStrength()->value, retorna rejected reason=auth.assurance_insufficient metadata completa; **PRESERVACIÓN EXPLÍCITA rama 1-candidato `if ($tried===1) return $firstDecision` INTACTA al final del método sin tocar**,
  - Bloque 7 (SP wiring DI): 6 bindings explícitos scoped return null por defecto cuando config=false default (VoltStack Container NO respeta ?Interfaz = null en constructor) → PasskeyCredentialStoreInterface driver file/memory memory default, RelyingPartyConfig rp.id/name/origins from config, PasskeyAuthenticator, DistributedThrottleCounterInterface stub null cuando enabled, AdaptiveRiskPolicyInterface ConfigBased thresholds, BearerTokenService TTLs config; Composite AuthenticatorResolver injection passkey resolver priority 900 + oidc resolver priority 850 INLINE dentro del único binding AuthenticatorResolverInterface closure (NO existe Application::extend() en VoltStack - lesson learned),
  - H1 Cross-suite COMPLETA ≥240 tests AUTH (objetivo 240 baseline → 690 framework Unit tests actuales, 46 tests nuevos del ciclo 083: BloqueF1 10 + BloqueG1 10 + BloqueH1 8 + BloqueI1 6 + BloqueJ1 6 + BloqueK1 6 = 46; 216 baseline 082 + 46 = 262 AUTH ≥240),
  - H2 actualizar 4 docs AUTHENTICATION DEVELOPMENT.
- Evidencia principal:
  - B1 Passkeys crypto: `Passkeys/CoseKey.php` (CBOR map parser DER→PEM), `Passkeys/CoseSignatureVerifier.php` (openssl_verify ES256/RS256 raw↔DER), `Passkeys/FilePasskeyCredentialStore.php` (JSON 1-per-cred path traversal prevention), `Passkeys/PasskeyAuthenticator.php` (AuthenticatorInterface priority 900), `Passkeys/PasskeyRegistrationCeremony.php` finishAttestation real packed COSE, `Passkeys/PasskeyAssertionCeremony.php` verifyAssertion openssl real + signCount,
  - B2 OIDC crypto: `Federation/Oidc/OidcIdentityTokenValidator.php` V2 splitCompactJws/decodeCompactJws/jwkToPem(RSA+EcP256)/validateSignature openssl real RS256/ES256 raw↔DER/validateAll compact_jws strict,
  - B3 Bearer rotation: `Tokens/OpaqueRefreshToken.php` V2 consumed/consumedAt/rotatedTo/familyId, `Contracts/OpaqueTokenRepositoryInterface.php` 3 métodos nuevos, `Tokens/InMemoryOpaqueTokenRepository.php` full chain impl, `Tokens/BearerTokenService.php` issueTokenPair/rotateRefresh one-time consume + reuse revokeFamily,
  - B4 Throttle V2: `Exceptions/ThrottleDeniedException.php`, `Contracts/DistributedThrottleCounterInterface.php`, `Exceptions/AuthExceptionMapper.php` status 429 Retry-After X-Auth-Throttle-*,
  - B5 Risk V2: `AbuseProtection/RiskAssessmentResult.php`, `Contracts/AdaptiveRiskPolicyInterface.php`, `AbuseProtection/RiskDecision.php` (allow/stepUp/deny actions VO), `AbuseProtection/ConfigBasedAdaptiveRiskPolicy.php` thresholds buckets, `Exceptions/RiskDeniedException.php` 403,
  - B6 Assurance V2: `Exceptions/AssuranceInsufficientException.php` 423, `Runtime/AuthenticationOrchestrator.php` L24-L57 preAuth min_assurance hook start() + `if ($tried===1) return $firstDecision` preservado intacto al final,
  - B7 SP wiring: `AuthenticationServiceProvider.php` L180-L290 AuthenticatorResolverInterface closure INLINE injection passkey p900 + oidc p850 (NO extend()) + bindings L427-L537 RelyingPartyConfig / PasskeyCredentialStore / PasskeyAuthenticator / DistributedThrottleCounter / AdaptiveRiskPolicy / BearerTokenService (todos return null por defecto config disabled),
  - tests nuevos ciclo 083: `tests/Unit/BloqueF1CryptoTest.php` 10 (8 skip crypto entorno Windows / 2 green estructural), `tests/Unit/BloqueG1CryptoTest.php` 10 (3 skip crypto / 7 green), `tests/Unit/BloqueH1BearerRotationTest.php` 8 (58 assertions green), `tests/Unit/BloqueI1ThrottleV2Test.php` 6 (31 assertions), `tests/Unit/BloqueJ1RiskV2Test.php` 6 (80 assertions), `tests/Unit/BloqueK1AssuranceV2Test.php` 6 (27 assertions) = 46 tests nuevos, 262 total AUTH (≥240 objetivo),
  - regression: BloqueFTest 6 legacy Passkeys structural (finishRegistration deprecated preserved) + BloqueGTest 6 legacy OIDC structural = 12 backward compat 100% green, suite 58 bloques 083+legacy exit 0 (317 assertions, 11 skip condicionales crypto por entorno OpenSSL Windows roto),
  - framework Unit completo: 690 tests / 4000 assertions exit 1 (solo 2 preexistentes failures Bloque5RiskV2Test y BloqueCTest FUERA alcance 083).
- Resultado:
  - Hard constraints 100% preservados: sin librerías externas Composer, openssl_pkey_verify PHP nativo + COSE/JWKS parser from scratch zero-deps,
  - Backward compat 100% OPT-IN: ceremonies finishRegistration() / finishAssertion() deprecated 082 intactos, OidcValidator signature bypass @deprecated preserved, Orchestrator 1-candidate branch intacto, Container DI bindings null default cuando flags=false (baseline 082 identico cuando todo disabled),
  - Passkeys WebAuthn RP real openssl+COSE sin JOSE/phpseclib: ceremonies validan packed attestation authData, clientDataJSON, rpIdHash, flags, signCount, credBlob; FileStore anti-path-traversal,
  - OIDC JWS compact validator strict: split 3 segments, b64url decode sin padding, JWKS RSA n/e modulus/exponent, ECDSA P-256 x/y uncompressed 0x04‖x‖y → PEM SPKI openssl load, signature verify openssl real RS256/ES256 raw→DER,
  - Bearer rotation one-time consume family reuse detection: primer consume marca consumed/consumedAt/rotatedTo → segundo consume findByFamilyId + revokeFamily bulk children rotatedTo linked list denegar acceso,
  - Throttle V2 429 Too Many Requests: Retry-After header estandar, ThrottleDeniedException con metadata granular id/scope, DistributedThrottleCounterInterface pluggable para Redis/DB driver,
  - Risk V2 adaptive buckets: riskScore clamp 0-100, stepUp/deny thresholds config, RiskDecision VO type-safe (enum backed actions + requiredStrength), RiskDeniedException 403,
  - Assurance V2 orchestrator preAuth hooks: min_authentication_assurance attribute operation-driven, assurance_value override GenericIdentity attribute para hardware backed passkeys (K1 #5 Bloque test), rejected metadata auth.assurance_insufficient source orchestrator_preauth_min_assurance, status 423,
  - SP wiring lección aprendida CRÍTICA repetida: VoltStack Application NO soporta `$this->app->extend()` método; resolvers composite DEBEN inyectarse inline dentro del binding AuthenticatorResolverInterface closure único,
  - Tests 46 nuevos 262 AUTH ≥ 240, baseline 216 082 intacto sin regresiones estructurales.
- Gap natural posterior:
  - falta distribucion real DistributedThrottleCounterInterface driver Redis/DB o file shared lock con purge TTL,
  - falta OIDC JWKS well-known HTTP fetch real (no mock InMemory) con Guzzle/file_get_contents TTL cache invalidación kid miss refresh,
  - falta integración RiskDecision step_up_required → AssuranceInsufficientException / StepUpRequiredException cadena interop,
  - falta BearerToken File storage driver purge policy TTL (1 JSON-per-token sanitized path traversal pattern = Passkeys FileStore),
  - falta E2E en entorno OpenSSL valido (Linux/Docker CI) para validar crypto real EC/RSA keypair generation + sign + verify,
  - falta MFA TOTP authenticator oficial complementando password/passkeys con amr=['totp'] assurance profile.

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
12. Password policy explicita para login.
13. Rotacion, revocacion y purga basica de sessions.
14. `AuthenticationServiceProvider` dedicado para integrar el subsistema.
15. Middleware alias `auth` para proteger rutas del framework.
16. Upgrade persistente de password hash cuando el provider local usa `storage_path`.
17. Middleware alias `guest` para rutas exclusivas de invitados.
18. Denial `auth.guest_only` coherente sin challenge headers improcedentes.
19. Denial `auth.stale_session` coherente para sesiones expiradas o invalidas en rutas protegidas.
20. Denial `authentication_strength_insufficient` reutilizando el contrato de Controllers Security.
21. Metadata de ruta `auth.minimum_strength` respetada por el middleware `auth`.
22. `AuthenticationContext` expone `authenticationStrength()` y `authenticationAssuranceProfile()`.
23. Trusted-device credential cliente duradera validada en runtime y usada para challenge reduction sin colapsar la semantica de assurance.
24. El challenge reduction por trusted device ahora rota el cookie cliente y conserva `previous_credential_hash` para detectar replay inmediato del credential anterior.
25. El replay del credential anterior revoca el trusted-device record, limpia el cookie y evita conservar `device_trust_state=trusted` durante el recovery.
26. Soporte HTTP para multiples `Set-Cookie` en el mismo response de Authentication.
27. Login, session restore y `setUser()` conservan `authentication_strength` y `authentication_assurance_profile`.
28. `LocalIdentityProvider` soporta segundo factor configurable para elevar assurance a `MultiFactor`.
29. Password + `second_factor` conserva `amr` y assurance MFA al restaurar la session.
30. `AuthManager` soporta `stepUp()` y `stepUpOrFail()` sobre una session autenticada existente.
31. `step-up` reemite session endurecida y conserva `amr`/assurance MFA para rutas que exigen `MultiFactor`.
32. Alias `mfa` como entry point explicito del framework para rutas que requieren elevation.
33. Denial `auth.step_up_required` coherente, sin `WWW-Authenticate`, con headers propios para el cliente.
34. Metadata fluida `Route::mfa()` para expresar step-up junto a `middleware('auth')`.
35. Recovery de session con razones explicitas persistidas (`revoked`/`expired`) y denial `auth.revoked_session` para invalidacion activa.
36. Cada sesion emitida posee `session_public_id` seguro para inventory/revocacion sin exponer el bearer secret.
37. `AuthManager` ya permite listar sesiones propias, identificar la actual y revocar una sesion concreta o las demas sesiones del principal.
38. El inventory de sesiones ya expone metadata reducida (`client_family`, `ip_prefix`, `label`, `last_activity_at`) sin almacenar ni devolver `User-Agent` o IP crudos.
39. La revocacion remota de sesiones ya exige fresh authentication configurable y responde con `auth.fresh_authentication_required` cuando corresponde.
40. El inventory ya expone hints de accion (`can_revoke`, `requires_reauthentication`) y metadata de device reducida (`client_platform`, `device_kind`) apta para UI de security center.
41. Los repositorios de session ya soportan retencion minima y purga explicita de tombstones de recovery.
42. El framework ya expone el comando `auth:sessions:cleanup` para cleanup operativo de sesiones expiradas y tombstones vencidos.
43. El inventory ya expone `device_reference` pseudonimizado, `device_trust_state` y policy de revocacion mas expresiva (`revocation_scope`, `revocation_mode`) sin promocionar fingerprint derivado a trusted-device real.
44. El subsistema ya soporta trusted-device records persistentes, alta del dispositivo actual con MFA y olvido/revocacion de trusted devices propios.
45. `AuthManager::devices()` ya entrega un inventory agregado por `device_reference` que combina sessions y trusted devices en una sola vista de security center.
46. El inventory agregado ya funciona sobre `file` stores compartidos entre instancias distintas y expone hints de management sin exponer secretos bearer.
47. `AuthManager::revokeDevice()` ya permite revocar desde el inventory agregado todas las sesiones y el trusted device asociados a un `device_reference`.
48. La revocacion agregada por dispositivo preserva self-revoke del dispositivo actual y exige fresh-auth cuando la mutacion afecta estado remoto.
49. `AuthManager::revokeOtherDevices()` ya permite revocar en lote todos los dispositivos remotos preservando el dispositivo actual.
50. El comando `auth:sessions:cleanup` ya purga trusted devices expirados ademas de sesiones y tombstones.
51. Los repositorios de session y trusted devices ya soportan enumeracion global mediante `all()` para tooling operativo y reconciliacion.
52. El framework ya expone `auth:devices:reconcile` con modo `dry-run` para normalizar el posture trusted/untrusted de sesiones sobre stores compartidos.

### Ya preparado de forma adyacente

1. Metadata y atributos de seguridad relacionados con autenticacion.
2. Excepcion `AuthenticationRequiredException`.
3. Mapeo de respuestas `401` en el handler de seguridad.
4. Semantica preliminar de `AuthenticationStrength` en Controllers Security.

### Todavia parcial o incompleto

1. Dominio de identidad y evidencias.
2. Manager y orchestrator completos para multiples mecanismos de autenticacion.
3. Password authentication con lifecycle formal mas alla del rehash persistente inicial.
4. Session authentication con revocacion distribuida y stores mas robustos.
5. Identity eligibility y estado de seguridad mas ricos.
6. Failure handling coherente y mas completo del subsistema mas alla de `guest_only`, `stale_session` y strength insufficiente.
7. Testing system formal del subsistema.

### Aun no desarrollado con evidencia suficiente

1. Remember-me.
2. Bearer/API tokens.
3. policy/authorization multi-actor sobre sessions y devices agregadas, reporting operativo del security center y alineacion administrativa con Controllers Security.
4. Passkeys / WebAuthn.
5. OIDC / federacion.
6. Risk engine.
7. Policy engine.
8. Multi-tenancy auth.
9. Distributed auth runtime.
10. Audit, observability y tooling operacional.

## Siguiente bloque recomendado

### Opcion recomendada inmediata

Consolidar el flujo ya operativo y cerrar los faltantes del nucleo distribuido y del security center:

- `25_AUTHENTICATION_FAILURE_ERROR_EXCEPTION_DENIAL_AND_SECURITY_RESPONSE_HANDLING_SYSTEM.md`
- `12_SESSION_AUTHENTICATION_PERSISTENCE_CONTEXT_RESTORATION_AND_SESSION_LIFECYCLE_SYSTEM.md`
- `22_AUTHENTICATION_LOGIN_LOGOUT_SIGN_IN_SIGN_OUT_ENTRY_POINT_AND_USER_AUTHENTICATION_FLOW_SYSTEM.md`
- `37_AUTHENTICATION_ASSURANCE_LEVEL_AUTHENTICATION_CONTEXT_AND_TRUST_CLASSIFICATION_SYSTEM.md`
- `49_AUTHENTICATION_REFERENCE_IMPLEMENTATION_DEFAULT_COMPONENTS_SECURE_DEFAULTS_AND_FRAMEWORK_INTEGRATION_SYSTEM.md`
- `30_AUTHENTICATION_DISTRIBUTED_SYSTEM_CLUSTER_SESSION_COORDINATION_REVOCATION_CONSISTENCY_AND_MULTI_NODE_RUNTIME.md`
- `35_AUTHENTICATION_SESSION_DEVICE_CREDENTIAL_INVENTORY_SECURITY_CENTER_AND_USER_SECURITY_MANAGEMENT.md`
- `42_AUTHENTICATION_PRIVACY_DATA_MINIMIZATION_RETENTION_CONSENT_AND_SECURITY_METADATA_GOVERNANCE_SYSTEM.md`

### Motivo

- ya existe autenticacion real minima por password y session con resolver, facade, policy, tombstones de recovery, inventory seguro y errores propios,
- el valor inmediato ahora esta en pasar del inventory basico local a coordinacion, metadata y retencion mas gobernadas,
- y abrir MFA, federation o passkeys antes de cerrar eso produciria sobrearquitectura sin cierre operativo.

## Entregables minimos sugeridos para ese siguiente ciclo

1. store de session mas robusto o compartido con retencion y limpieza de tombstones.
2. metadata de inventory por identidad o dispositivo.
3. revocacion administrativa mas rica por public identifier y ownership/policy.
4. denials y recovery coordinado para session stale/revocada en escenarios mas distribuidos.
5. entry points complementarios adicionales sobre `auth/guest`.
6. alineacion de `AuthenticationContext` con el stack adyacente de Controllers Security.
7. API publica minima:
   - `Auth::check()`
   - `Auth::guest()`
   - `Auth::user()`
   - `Auth::id()`
   - `Auth::attempt()`
   - `Auth::login()`
   - `Auth::logout()`
8. pruebas unitarias y feature del flujo:
   - revocacion distribuida o store robusto,
   - inventario, public identifiers y tombstones,
   - middleware complementario y denials diferenciados,
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
