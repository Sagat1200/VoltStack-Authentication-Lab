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

Hoy existe solamente:

- `Quantum\Auth\AuthManager`
- helper `auth()`
- binding scoped en `Application.php`
- test feature basico de request scope

Por tanto, el plan parte de una base minima y debe preservar compatibilidad con:

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
4. ruta o middleware `auth` puede evolucionar sobre este core.

### Tests minimos

1. `Auth::attempt()` autentica,
2. `Auth::logout()` limpia session y contexto,
3. facade no filtra estado entre requests,
4. bootstrap registra todos los servicios necesarios.

## Fase 6 - Endurecimiento minimo para declarar V1

### Objetivo

Cerrar el primer release realmente util.

### Entregables

1. mapa de excepciones de Authentication
2. responses `401` coherentes
3. pruebas feature del flujo completo
4. documentacion de configuracion inicial
5. actualizacion de `DEVELOPMENT_MATRIX` y `DEVELOPMENT_VERSIONS`

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

### DV-AUTH-003

Alcance sugerido:

- Fase 0
- Fase 1
- inicio de Fase 2

Entregables minimos:

1. `AuthenticationRequest`
2. `AuthenticationDecision`
3. `AuthenticationContext`
4. `IdentityInterface`
5. `AuthenticationManagerInterface`
6. `AuthenticationOrchestratorInterface`
7. `AuthenticationOperationContext`
8. refactor de `AuthManager` a fachada de compatibilidad

Resultado esperado:

- el framework deja de tener solo un contenedor `auth.user`,
- y pasa a tener el lenguaje base sobre el que se construira el resto del sistema.
