# EXECUTIVE_IMPLEMENTATION_PLAN

## Proposito

Este documento traduce la arquitectura del sistema Authentication a un plan ejecutivo de implementacion para el paquete:

- `vendor/voltstack/framework/src/Quantum/Auth`

Su objetivo es convertir la documentacion `00-50` en una secuencia de desarrollo operable, incremental y compatible con el estado actual del framework.

## Objetivo del primer cierre real

El primer cierre util del subsistema `Quantum/Auth` no debe intentar cubrir toda la plataforma descrita en `Docs/00-50`.

El objetivo inmediato debe ser entregar un stack minimo, seguro y usable con estas capacidades:

1. principal autenticado request-scoped,
2. identidad canonica,
3. password authentication,
4. session authentication,
5. login/logout reales,
6. `AuthenticationContext` confiable,
7. facade/helper ergonomicos,
8. integracion con runtime y Controllers Security.

## Regla de alcance

No abrir en la primera fase:

- MFA,
- passkeys,
- OIDC,
- risk engine,
- abuse protection avanzado,
- distributed session coordination,
- machine identities,
- policy engine completo.

Esos bloques dependen de un core que todavia no existe.

## Estado actual de partida

Hoy ya existe una base minima ampliada:

- `Quantum\Auth\AuthManager`
- helper `auth()`
- binding scoped en `Application.php`
- `AuthenticationManagerInterface`
- `AuthenticationOrchestratorInterface`
- `AuthenticationContext`
- `AuthenticationDecision`
- `AuthenticationContextAccessor`
- `AuthenticationOrchestrator`
- `IdentityProviderInterface`
- `LocalIdentityProvider`
- `AuthenticatorInterface`
- `PasswordAuthenticator`
- `PasswordCredentials`
- `AuthManager::attempt()`
- `AuthenticationSession`
- `AuthenticationSessionRepositoryInterface`
- `InMemoryAuthenticationSessionRepository`
- `SessionAuthenticator`
- `AuthManager::login()`
- `AuthenticatorResolverInterface`
- `DefaultAuthenticatorResolver`
- facade `Auth`
- `config/auth.php`
- `AuthenticationServiceProvider`
- `AuthenticationSessionRecoveryReason`
- `AuthenticationSessionPublicId`
- `touch()` de session para refresh de metadata
- middleware alias `auth`
- middleware alias `guest`
- middleware alias `mfa`
- `IdentitySecurityState`
- excepciones propias de Authentication
- `attemptOrFail()`
- `FileAuthenticationSessionRepository`
- `PasswordPolicy`
- upgrade persistente inicial de hash usando `storage_path`
- purga, rotacion y revocacion basica de session
- tombstones minimos de recovery para sesiones revocadas/expiradas
- inventory seguro por `session_public_id`
- metadata reducida de inventory y refresh server-side de `last_activity`
- denial explicito `auth.revoked_session`
- pruebas unitarias y feature del lenguaje base y del request scope
- pruebas del flujo minimo de password authentication
- pruebas del flujo minimo de session authentication
- pruebas de facade, resolver y expiracion minima de session
- pruebas de elegibilidad y driver file de session
- pruebas de password policy, rotation y revocation

Por tanto, el plan debe preservar compatibilidad con:

- `auth()->user()`
- `auth()->check()`
- `auth()->guest()`
- `auth()->id()`
- `auth()->logout()`

## Arquitectura minima objetivo para V1

La V1 operativa de `Quantum/Auth` deberia quedar compuesta por estos bloques:

```text
Auth Facade / Helper
        |
        v
AuthenticationManager
        |
        v
AuthenticationOrchestrator
        |
        +--> Firewall Resolver
        +--> Authenticator Resolver
        +--> Authenticator
        +--> Identity Provider
        +--> Credential Verifier
        +--> Decision Engine
        +--> Context Factory
        +--> Session Persistence
```

## Estructura recomendada de namespaces

La estructura inicial recomendada es esta:

```text
Quantum/Auth
    Contracts/
    Context/
    Identity/
    Credentials/
    Authenticators/
    Sessions/
    Decisions/
    Support/
    Runtime/
    Exceptions/
    Facades/
```

## Layout inicial sugerido

```text
Quantum/Auth
    AuthManager.php
    Contracts/
        AuthenticationManagerInterface.php
        AuthenticationOrchestratorInterface.php
        AuthenticatorInterface.php
        AuthenticatorResolverInterface.php
        IdentityProviderInterface.php
        CredentialVerifierInterface.php
        AuthenticationSessionRepositoryInterface.php
    Context/
        AuthenticationRequest.php
        AuthenticationContext.php
        AuthenticationContextAccessor.php
    Identity/
        IdentityInterface.php
        IdentityIdentifier.php
        IdentityReference.php
    Credentials/
        PasswordCredentials.php
        VerifiedCredential.php
    Authenticators/
        PasswordAuthenticator.php
        SessionAuthenticator.php
        DefaultAuthenticatorResolver.php
    Sessions/
        AuthenticationSession.php
        AuthenticationSessionId.php
        AuthenticationSessionFactory.php
        AuthenticationSessionRepository.php
    Decisions/
        AuthenticationDecision.php
        AuthenticationDecisionStatus.php
        AuthenticationContextFactory.php
    Runtime/
        AuthenticationOrchestrator.php
        AuthenticationOperationContext.php
    Exceptions/
        AuthenticationException.php
        InvalidCredentialsException.php
        AuthenticationRequiredException.php
    Facades/
        Auth.php
```

## Fases ejecutivas

## Fase 0 - Refactor de compatibilidad

### Objetivo

Preservar el `AuthManager` actual, pero convertirlo en una fachada fina sobre una infraestructura nueva.

### Entregables

1. Mantener `AuthManager` como punto de entrada publico actual.
2. Introducir `AuthenticationContextAccessor`.
3. Mover el acceso a `RuntimeContext` a una pieza dedicada.
4. Conservar compatibilidad del helper `auth()`.

### Criterio de cierre

- el test actual sigue pasando,
- el acceso a `auth.user` deja de estar embebido directamente en toda la logica futura.

## Fase 1 - Dominio base

### Objetivo

Construir el lenguaje minimo del sistema.

### Entregables

1. `IdentityInterface`
2. `IdentityIdentifier`
3. `IdentityReference`
4. `AuthenticationRequest`
5. `AuthenticationDecision`
6. `AuthenticationDecisionStatus`
7. `AuthenticationContext`

### Reglas

- `AuthenticationContext` debe ser inmutable.
- `AuthenticationDecision` no es lo mismo que `AuthenticationContext`.
- `IdentityInterface` no debe asumir email, password ni roles.

### Tests minimos

1. value objects y enums,
2. invariantes de inmutabilidad,
3. `AuthenticationContext` solo nace desde decision autenticada.

## Fase 2 - Orquestacion minima

### Objetivo

Crear el pipeline interno del sistema.

### Entregables

1. `AuthenticationManagerInterface`
2. `AuthenticationOrchestratorInterface`
3. `AuthenticationOperationContext`
4. `DefaultAuthenticationManager`
5. `AuthenticationOrchestrator`

### Operaciones minimas

- `authenticate`
- `recover`
- `logout`

### Regla de diseno

El manager coordina; los servicios especializados deciden y ejecutan.

### Tests minimos

1. flujo feliz de manager -> orchestrator,
2. propagacion de decision,
3. limpieza de contexto al terminar request.

## Fase 3 - Password authentication

### Objetivo

Entregar el primer mecanismo real de autenticacion.

### Entregables

1. `PasswordCredentials`
2. `AuthenticatorInterface`
3. `PasswordAuthenticator`
4. `IdentityProviderInterface`
5. `CredentialVerifierInterface`
6. implementacion default para verificacion de password

### Alcance minimo recomendado

- login por identificador + password,
- resolucion de identidad local,
- verificacion de password,
- decision `AUTHENTICATED` o `REJECTED`.

### Fuera de alcance en esta fase

- reset de password,
- password upgrade policy avanzada,
- throttling avanzado,
- MFA,
- step-up.

### Tests minimos

1. credencial valida autentica,
2. credencial invalida rechaza,
3. identidad inexistente rechaza sin leakage innecesario,
4. no se exponen secretos en errores.

### Estado actual del corte

Parcialmente implementada:

- existe `IdentityProviderInterface`,
- existe `LocalIdentityProvider`,
- existe `AuthenticatorInterface`,
- existe `PasswordAuthenticator`,
- existe `PasswordCredentials`,
- y `AuthManager::attempt()` ya autentica contra el provider local configurado.

Falta en esta fase:

- policy formal de password,
- ciclo de vida de credenciales,
- y endurecimiento adicional del flujo.

## Fase 4 - Session authentication

### Objetivo

Persistir y restaurar autenticacion de forma segura.

### Entregables

1. `AuthenticationSession`
2. `AuthenticationSessionId`
3. `AuthenticationSessionFactory`
4. `AuthenticationSessionRepositoryInterface`
5. `SessionAuthenticator`
6. restauracion de contexto desde session validada

### Reglas

- `AuthenticationSession` no es la application session completa.
- no almacenar passwords ni secretos raw.
- validar expiracion y estado antes de restaurar contexto.
- preparar la forma para futura rotacion y revocacion.

### Tests minimos

1. session valida restaura contexto,
2. session expirada no restaura,
3. logout invalida el estado restaurable,
4. aislamiento entre requests consecutivos.

### Estado actual del corte

Parcialmente implementada:

- existe `AuthenticationSession`,
- existe `AuthenticationSessionId`,
- existe `AuthenticationSessionRepositoryInterface`,
- existe `InMemoryAuthenticationSessionRepository`,
- existe `SessionAuthenticator`,
- `AuthManager::login()` y `AuthManager::logout()` ya interactuan con la session,
- y el kernel HTTP ya emite `Set-Cookie` / `X-Auth-Session`.

Falta en esta fase:

- rotacion de session,
- storage persistente,
- y reglas mas fuertes de revocacion.

## Fase 5 - DX e integracion de framework

### Objetivo

Convertir el core en experiencia de desarrollo usable.

### Entregables

1. `Auth` facade
2. metodos en `AuthManager`:
   - `attempt()`
   - `login()`
   - `logout()`
   - `context()`
   - `principal()`
3. configuracion `config/auth.php`
4. `AuthenticationServiceProvider`
5. bindings de contratos default

### Integraciones esperadas

1. helper `auth()` sigue funcionando,
2. `Application.php` delega a provider dedicado,
3. Controllers Security puede consumir `AuthenticationContext` real,
4. ruta o middleware `auth` ya puede apoyarse directamente sobre este core.

### Tests minimos

1. `Auth::attempt()` autentica,
2. `Auth::logout()` limpia session y contexto,
3. facade no filtra estado entre requests,
4. bootstrap registra todos los servicios necesarios.

### Estado actual del corte

Parcialmente implementada:

- existe facade `Auth`,
- existe `config/auth.php`,
- `AuthManager` ya expone `attempt()`, `login()`, `logout()` y `context()`,
- existe `DefaultAuthenticatorResolver`,
- existe `AuthenticationServiceProvider`,
- existe middleware alias `auth`,
- existe middleware alias `guest`,
- y la session minima usa configuracion de cookie y expiracion.

Falta en esta fase:

- entry points adicionales mas alla de `auth/guest`,
- una API publica aun mas pulida para adopcion de framework,
- y alineacion mas profunda con Controllers Security.

## Fase 6 - Endurecimiento minimo para declarar V1

### Objetivo

Cerrar el primer release realmente util.

### Entregables

1. mapa de excepciones de Authentication
2. responses `401` coherentes
3. pruebas feature del flujo completo
4. documentacion de configuracion inicial
5. actualizacion de `DEVELOPMENT_MATRIX` y `DEVELOPMENT_VERSIONS`

### Estado actual del corte

Parcialmente implementada:

- existen errores propios de Authentication,
- existe elegibilidad minima de identidad,
- `attemptOrFail()` ya permite flujos con excepcion,
- el storage de session puede ser `memory` o `file`,
- y existe upgrade persistente inicial de hash para provider local con `storage_path`.

Falta en esta fase:

- revocacion distribuida,
- un modelo mas completo de errores por mecanismo,
- entry points complementarios,
- y stores mas robustos para despliegues reales.

### Criterio de cierre de V1

Se puede considerar cerrada la primera version del sistema cuando exista:

1. login por password,
2. restauracion por session,
3. logout,
4. `AuthenticationContext`,
5. facade/helper estables,
6. tests unitarios y feature,
7. integracion usable en una app VoltStack nueva.

## Mapa de clases prioritarias

## Prioridad P0

Estas son las clases que mas valor destraban al inicio:

1. `IdentityInterface`
2. `IdentityIdentifier`
3. `AuthenticationRequest`
4. `AuthenticationDecision`
5. `AuthenticationContext`
6. `AuthenticationManagerInterface`
7. `AuthenticationOrchestratorInterface`
8. `AuthenticationOperationContext`
9. `AuthenticationContextAccessor`

## Prioridad P1

1. `AuthenticatorInterface`
2. `PasswordAuthenticator`
3. `IdentityProviderInterface`
4. `CredentialVerifierInterface`
5. `AuthenticationSession`
6. `AuthenticationSessionRepositoryInterface`
7. `SessionAuthenticator`

## Prioridad P2

1. `Auth` facade
2. `AuthenticationServiceProvider`
3. `config/auth.php`
4. excepciones especificas
5. adaptacion a Controllers Security

## Orden recomendado de implementacion

1. `Context/`
2. `Identity/`
3. `Decisions/`
4. `Contracts/`
5. `Runtime/`
6. `Authenticators/PasswordAuthenticator`
7. `Sessions/`
8. `Facades/ + provider + config`
9. integracion con seguridad existente

## Integraciones del framework que deben tocarse

La implementacion de `Quantum/Auth` probablemente requerira tocar estas piezas:

1. `vendor/voltstack/framework/src/Platform/Application.php`
2. `vendor/voltstack/framework/src/Helper/helpers.php`
3. `vendor/voltstack/framework/src/Quantum/Controllers/Security/Context/ControllerSecurityContextFactory.php`
4. `vendor/voltstack/framework/src/Quantum/Controllers/Security/Engine/ControllerSecurityManager.php`
5. `vendor/voltstack/framework/src/Quantum/Controllers/Security/Exceptions/ControllerSecurityExceptionMapper.php`
6. `config/` para introducir configuracion de auth

## Riesgos a vigilar

### 1. Acoplar todo a arrays

El `AuthManager` actual admite arrays y objetos arbitrarios.
Eso es util como compatibilidad temporal, pero la nueva arquitectura debe migrar progresivamente hacia identidades tipadas.

### 2. Saltarse el core por ergonomia

No permitir que:

- `Auth::login($identity)`

termine siendo solo:

- guardar un objeto en contexto o session

sin pasar por decision, context factory y persistencia.

### 3. Repetir semantica entre Auth y Controllers Security

Hay que alinear:

- `AuthenticationContext`
- `AuthenticationStrength`
- `AuthenticationRequiredException`

para evitar dos modelos paralelos de autenticacion dentro del framework.

### 4. Abrir demasiadas features antes de cerrar password + session

Si se abren temprano:

- tokens,
- MFA,
- federation,
- passkeys,

el riesgo es multiplicar contratos sin cerrar ningun flujo real.

## Definition of Done por fase

Una fase se considera realmente cerrada solo si:

1. existe codigo fuente identificable,
2. existe al menos un test unitario o feature representativo,
3. existe integracion real con runtime o bootstrap,
4. no se introduce estado global mutable,
5. se actualizan los artefactos de `AUTHENTICATION DEVELOPMENT`.

## Siguiente corte recomendado

El siguiente corte de implementacion recomendado es:

### DV-AUTH-063

Alcance sugerido:

- gobernanza operativa mas fuerte del `activity_drift` sobre stores compartidos y escenarios multi-nodo
- explicabilidad mas accionable de consistencia reciente entre stores sobre runtime principal y tooling operativo
- trazabilidad gobernada mas fuerte sobre mutaciones administrativas distribuidas

Entregables minimos:

1. convertir `activity_drift` en una señal operativa mas accionable por store y por ventana reciente.
2. enriquecer activity drift y consistencia reciente entre stores reutilizando la correlacion operativa, metrica, longitudinal, `store_cohorts`, `time_windows`, `store_time_windows` y `multi_store_summary` ya disponible.
3. mantener un policy/runtime compartido mas expresivo para actores administrativos sobre inventory agregado, mutaciones remotas y tooling de consola.
4. ampliar pruebas de integracion y hardening sobre escenarios delegados, distribuidos y multi-nodo.
5. seguir evolucionando el provider local hacia fuentes persistentes mas ricas.
6. sentar base para observabilidad administrativa mas util en produccion.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy administrativa, management de dispositivos, governance multi-actor, coordinacion distribuida, deteccion de drift y observabilidad operativa,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-021

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- metadata de device mas rica y policy/authorization mas expresiva
- limpieza/retencion gobernada de tombstones

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. metadata de device y actividad mas util para security center.
3. policy/authorization mas rica para revocacion administrativa.
4. limpieza/retencion de tombstones y recovery coordinado.
5. alineacion con Controllers Security y `AuthenticationContext`.
6. pruebas de integracion y hardening adicionales.
7. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy y entry points,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-022

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- tooling/cleanup de tombstones y retention minima operativa
- policy/authorization mas expresiva para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. cleanup/retention minima de tombstones y metadata derivada.
3. policy/authorization mas rica para revocacion administrativa.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup y entry points,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-023

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- policy/authorization mas expresiva para revocacion administrativa
- trusted devices y metadata de device mas estable

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. policy/authorization mas rica para revocacion administrativa y escenarios multi-actor.
3. trusted devices o referencias de device mas estables.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup y device posture,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-024

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- trusted devices reales y credentials duraderas
- policy/authorization multi-actor para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. trusted device records o credentials mas formales.
3. policy/authorization mas rica para revocacion administrativa y multi-actor.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup, trusted devices y device posture,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-025

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- trusted-device credentials cliente duraderas
- policy/authorization multi-actor para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. trusted-device credentials cliente mas formales y su validacion.
3. policy/authorization mas rica para revocacion administrativa y multi-actor.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session ya no es solo funcional, sino mejor coordinado entre runtime, recovery, policy, cleanup, trusted-device posture y challenge reduction,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.

### DV-AUTH-026

Alcance sugerido:

- coordinacion de sesiones realmente compartida
- rotacion y replay hardening de trusted-device credentials cliente
- policy/authorization multi-actor para revocacion administrativa

Entregables minimos:

1. stores de session mas robustos o distribuidos.
2. rotacion, invalidacion rica y replay hardening de trusted-device credentials.
3. policy/authorization mas rica para revocacion administrativa y multi-actor.
4. alineacion con Controllers Security y `AuthenticationContext`.
5. pruebas de integracion y hardening adicionales.
6. evolucion del provider local hacia fuentes persistentes mas ricas.

Resultado esperado:

- el flujo password + session conserva challenge reduction para dispositivos reconocidos, pero endurece rotacion, replay protection y revocacion de la credencial cliente,
- Authentication queda mejor posicionado para adopcion real dentro del framework,
- y el subsistema puede crecer hacia MFA, tokens y federation sin rehacer el nucleo.
