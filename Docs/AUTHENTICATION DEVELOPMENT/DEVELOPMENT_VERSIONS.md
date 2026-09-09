# DEVELOPMENT_VERSIONS

## Proposito

Esta bitacora registra el avance real del desarrollo del subsistema `Quantum/Auth` frente a la documentacion oficial ubicada en `vendor/voltstack/authentication-lab/Docs`.

Sirve como control operativo de:

- lo ya implementado,
- lo que quedo parcial,
- lo que todavia falta por construir,
- y el siguiente bloque recomendado de ejecucion.

## Corte actual

- Fecha de actualizacion: `2026-09-09`
- Estado general: `Existe lenguaje base del subsistema, password authentication real con rehash persistente opcional, session auth endurecida con tombstones minimos de recovery, inventory seguro por session_public_id, metadata de sesion y device reducida, refresh server-side de last_activity, hints de accion para revocacion, policy de revocacion mas expresiva, fresh-auth configurable para revocacion remota de sesiones y trusted devices, device_reference derivado pseudonimizado, trusted-device records server-side gestionables, trusted-device credential cliente duradera validada, challenge reduction para MFA obligatorio en dispositivos reconocidos, rotacion del trusted-device credential al reducir challenge, revocacion por replay del credential anterior, inventory de trusted devices con hints `can_forget/requires_reauthentication/revocation_scope/revocation_mode`, soporte HTTP para multiples Set-Cookie, retencion minima de tombstones y comando auth:sessions:cleanup, resolver formal, facade Auth, provider dedicado, middleware aliases auth/guest/mfa, MFA local, step-up operativo y denials explicitos auth.revoked_session/auth.stale_session/auth.fresh_authentication_required`
- Foco del siguiente ciclo recomendado: `session coordination distribuida real + policy/authorization multi-actor mas amplia + security center distribuido para sessions y trusted devices`

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
3. coordinacion distribuida real de session, policy/authorization multi-actor y background cleanup mas completo.
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
