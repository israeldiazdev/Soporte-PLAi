# Integración de Autenticación Keycloak en Index

## Resumen de Cambios

Se ha implementado la integración de Keycloak SSO en el botón "Ver mis tickets" del formulario de soporte (index.html), replicando exactamente el mismo flujo de autenticación que usa la página de login.

## Qué se Cambió

### 1. **index.html - Modificación del Botón**

**Antes:**
```html
<a href="login.html" class="btn-tickets">🎫 Ver mis tickets</a>
```

**Después:**
```html
<button type="button" class="btn-tickets" onclick="startKeycloakLogin()">🎫 Ver mis tickets</button>
```

### 2. **index.html - Adición de Función de Autenticación**

Se agregaron funciones de autenticación Keycloak al inicio del script:

```javascript
// KEYCLOAK AUTHENTICATION - Shared Logic
const KC_CONFIG = {
  url:      '***',
  realm:    '***',
  clientId: '****',
};

const KC_REDIRECT = {
  admin: 'dashboardtickets.html',
  user:  'mis-tickets.html',
};

// PKCE helpers
const kc_b64url = buf => btoa(...).replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,'');
const kc_sha256 = plain => crypto.subtle.digest('SHA-256', ...);
const kc_rnd = (n=64) => { ... };

// Check if user is admin
function kc_checkIsAdmin(payload) { ... }

// Start Keycloak login flow
async function startKeycloakLogin() { ... }
```

## Arquitectura de Autenticación

### Flujo PKCE (Proof Key for Code Exchange)

1. **Usuario clickea "Ver mis tickets"** en index.html
2. **Función `startKeycloakLogin()`** genera:
   - `verifier`: String aleatorio para PKCE
   - `state`: Token para prevenir CSRF
   - `challenge`: Hash SHA-256 del verifier
3. **Guarda en sessionStorage**: `kc_verifier`, `kc_state`
4. **Redirige a Keycloak** con parámetros:
   - `client_id`: Identificador de la app
   - `response_type`: 'code' (autorización)
   - `scope`: 'openid profile email roles'
   - `code_challenge`: Hash PKCE
   - `code_challenge_method`: 'S256'
5. **Usuario se autentica** en Keycloak
6. **Keycloak redirige a index.html** con el `code` de autorización
7. **Login.html intercepta el callback** (mismo flujo en ambas páginas)
8. **Exchange code por tokens** JWT del usuario
9. **Extrae roles y datos** del token
10. **Redirige al dashboard** según rol (admin o user)

## Configuración

### Parámetros Keycloak (KC_CONFIG)

```javascript
const KC_CONFIG = {
  url:      '***',      // URL del servidor Keycloak
  realm:    '***',      // Nombre del realm
  clientId: '****',     // Client ID configurado en Keycloak
};
```

**Nota:** Los valores están marcados con `***` por seguridad. Deben configurarse con los valores reales de tu instancia de Keycloak.

### Rutas de Redirección

```javascript
const KC_REDIRECT = {
  admin: 'dashboardtickets.html',  // Para usuarios con rol 'admin'
  user:  'mis-tickets.html',        // Para usuarios sin rol 'admin'
};
```

## Funciones Principales

### `startKeycloakLogin()`
- **Propósito:** Iniciar el flujo de autenticación con Keycloak
- **Parámetros:** Ninguno (se ejecuta al hacer click)
- **Acciones:**
  1. Genera verifier y state aleatorios
  2. Calcula challenge PKCE
  3. Guarda verifier y state en sessionStorage
  4. Redirige a la URL de autorización de Keycloak

### `kc_checkIsAdmin(payload)`
- **Propósito:** Verificar si el usuario tiene rol de administrador
- **Parámetros:** `payload` - JWT decodificado del usuario
- **Retorna:** `boolean` - true si es admin, false si no

### `kc_b64url(buf)`, `kc_sha256(plain)`, `kc_rnd(n)`
- **Propósito:** Funciones utilitarias para PKCE
- **Notas:** Idénticas a las usadas en login.html

## Ventajas de esta Implementación

1. **Sin Duplicación:** La lógica de autenticación es reutilizable y consistente
2. **Seguridad PKCE:** Implementa el estándar PKCE para OAuth2 en navegadores
3. **Manejo de Roles:** Redirige automáticamente según el rol del usuario
4. **Session Management:** Usa sessionStorage para mantener estado temporal
5. **Consistencia:** El flujo es idéntico en login.html e index.html

## Flujo de Referencia en Login.html

La función `handleCallback()` en login.html:
1. Intercepta el código de autorización en la URL
2. Realiza POST al endpoint de token de Keycloak
3. Obtiene access_token, refresh_token e id_token
4. Decodifica el JWT para extraer:
   - Email
   - Nombre
   - Roles (client + realm)
5. Guarda datos en sessionStorage:
   - `kc_access_token`
   - `kc_refresh_token`
   - `kc_id_token`
   - `kc_email`
   - `kc_name`
   - `kc_roles`
6. Redirige al dashboard apropiado

## Consideraciones de Seguridad

- ✅ **PKCE:** Previene ataques de intercepción de código
- ✅ **State Token:** Previene ataques CSRF
- ✅ **Session Storage:** Los tokens se guardan en memoria (no en cookies)
- ✅ **HTTPS:** Debe usarse siempre en producción
- ⚠️ **Client Secret:** No se usa en el cliente (solo en servidor)

## Testing

Para probar la integración:

1. **Desde Index:**
   - Click en "Ver mis tickets"
   - Debe redirigir a Keycloak para autenticación
   - Luego a login.html para callback
   - Finalmente al dashboard (admin o user)

2. **Desde Login:**
   - Click en "Iniciar sesión"
   - Mismo flujo que antes
   - El comportamiento debe ser idéntico

## Archivos Modificados

- `/Users/macairsisrael/Soporte-PLAi/index.html` - Agregada función y cambio de botón
- `/Users/macairsisrael/Soporte-PLAi/login.html` - Sin cambios (referencia)
- `/Users/macairsisrael/Soporte-PLAi/dashboardtickets.html` - Sin cambios (destino)
- `/Users/macairsisrael/Soporte-PLAi/mis-tickets.html` - Sin cambios (destino)

## Próximos Pasos (Opcional)

1. **Extraer a archivo externo:** Si hay más funciones de autenticación compartidas, crear `auth.js`
2. **Service Worker:** Implementar refresh de tokens automático
3. **Error Handling:** Agregar manejo de errores más robusto
4. **Testing:** Agregar tests E2E con Playwright o Cypress
