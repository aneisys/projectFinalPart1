# 🔐 Troubleshooting: API de Autenticación — Error `401 Not Authorized`

> **Rama:** `feature/auth-api-connection`  
> **Fecha de reporte:** 2026-02-17  
> **Severidad:** 🔴 Alta  
> **Estado:** En investigación

---

## 📋 Descripción del Problema

El sistema no puede establecer conexión con la API de autenticación. Todas las solicitudes realizadas al endpoint de autenticación retornan consistentemente el error:

```
HTTP 401 Unauthorized
{
  "error": "not_authorized",
  "message": "Not Authorized",
  "status": 401
}
```

Este error ocurre **independientemente** del usuario, credenciales o entorno desde el que se realice la solicitud.

---

## 🔍 Síntomas Observados

- Todas las llamadas a `/api/auth/login`, `/api/auth/token` y `/api/auth/refresh` retornan `401`.
- El error se presenta tanto en entorno de **desarrollo** como en **staging**.
- Los logs del servidor de autenticación no registran intentos de conexión entrantes.
- El cliente recibe la respuesta de error en menos de 50ms (indicativo de rechazo antes de llegar al servicio real).
- No hay diferencia de comportamiento entre usuarios administradores y usuarios regulares.

---

## 🧩 Posibles Causas

### 1. API Key o Token de Servicio Inválido / Expirado

El header `Authorization` enviado en la solicitud contiene un token que:
- Ha expirado.
- Fue revocado manualmente desde el panel de administración.
- Fue generado para un entorno diferente (ej. producción vs. desarrollo).

### 2. Variables de Entorno Mal Configuradas

Las variables de entorno que almacenan las credenciales del servicio no están correctamente definidas o apuntan a valores incorrectos.

```bash
# Variables críticas a verificar
AUTH_API_URL=
AUTH_API_KEY=
AUTH_CLIENT_ID=
AUTH_CLIENT_SECRET=
```

### 3. Cabeceras HTTP Incorrectas o Faltantes

La solicitud no incluye las cabeceras requeridas por la API:

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Client-ID: <client_id>
```

### 4. Certificado SSL / TLS No Válido

El certificado del servidor de autenticación ha expirado o no es reconocido por el cliente, lo que provoca un rechazo a nivel de transporte que se manifiesta como `401`.

### 5. IP o Dominio No Incluido en la Lista Blanca (Whitelist)

La API de autenticación tiene restricciones de acceso por IP. El servidor que realiza las solicitudes no está autorizado en la configuración del proveedor de autenticación.

### 6. Configuración de CORS Incorrecta

En entornos de frontend, las solicitudes pueden ser bloqueadas por políticas de CORS antes de llegar al servidor, resultando en un error que se interpreta como `401`.

---

## 🛠️ Pasos de Diagnóstico

### Paso 1: Verificar Variables de Entorno

```bash
# En el servidor de aplicación
printenv | grep AUTH

# Resultado esperado (ejemplo)
AUTH_API_URL=https://auth.example.com
AUTH_API_KEY=sk-live-xxxxxxxxxxxx
AUTH_CLIENT_ID=client_abc123
AUTH_CLIENT_SECRET=secret_xyz789
```

> ⚠️ **Asegúrate de que ninguna variable esté vacía o con valor `undefined`.**

---

### Paso 2: Probar la API Directamente con `curl`

```bash
curl -v -X POST https://auth.example.com/api/auth/token \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $AUTH_API_KEY" \
  -d '{
    "client_id": "'"$AUTH_CLIENT_ID"'",
    "client_secret": "'"$AUTH_CLIENT_SECRET"'",
    "grant_type": "client_credentials"
  }'
```

**Interpretar la respuesta:**

| Código | Significado |
|--------|-------------|
| `200 OK` | La API funciona; el problema está en la aplicación |
| `401 Unauthorized` | Credenciales inválidas o expiradas |
| `403 Forbidden` | IP no autorizada o permisos insuficientes |
| `000` / Sin respuesta | Problema de red o URL incorrecta |

---

### Paso 3: Validar el Token / API Key

```bash
# Verificar fecha de expiración de un JWT
echo "<token_aqui>" | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# Buscar el campo "exp" en el payload
# Convertir timestamp Unix a fecha legible
date -d @<timestamp_exp>
```

---

### Paso 4: Revisar Logs del Servidor de Autenticación

```bash
# Si tienes acceso al servidor de autenticación
tail -f /var/log/auth-service/access.log
tail -f /var/log/auth-service/error.log

# Con Docker
docker logs <auth_container_name> --tail=100 -f

# Con Kubernetes
kubectl logs -n <namespace> deployment/auth-service --tail=100 -f
```

---

### Paso 5: Verificar Conectividad de Red

```bash
# Verificar resolución DNS
nslookup auth.example.com
dig auth.example.com

# Verificar conectividad TCP al puerto
telnet auth.example.com 443
nc -zv auth.example.com 443

# Verificar certificado SSL
openssl s_client -connect auth.example.com:443 -showcerts
```

---

### Paso 6: Revisar Configuración en el Código

```javascript
// ❌ Configuración incorrecta (ejemplo)
const authClient = new AuthClient({
  apiUrl: process.env.AUTH_URL,        // Variable incorrecta
  apiKey: "hardcoded-key-here",        // Nunca hardcodear credenciales
  timeout: 0,                          // Timeout indefinido
});

// ✅ Configuración correcta
const authClient = new AuthClient({
  apiUrl: process.env.AUTH_API_URL,    // Variable correcta
  apiKey: process.env.AUTH_API_KEY,    // Desde variables de entorno
  clientId: process.env.AUTH_CLIENT_ID,
  timeout: 5000,                       // 5 segundos de timeout
  headers: {
    'Content-Type': 'application/json',
    'X-Client-Version': '1.0.0',
  },
});
```

---

## ✅ Soluciones Aplicadas

### Solución A: Regenerar API Key

1. Acceder al panel de administración del proveedor de autenticación.
2. Navegar a **Settings → API Keys → Manage Keys**.
3. Revocar la clave actual y generar una nueva.
4. Actualizar la variable de entorno `AUTH_API_KEY` en todos los entornos.
5. Reiniciar los servicios afectados.

```bash
# Actualizar variable en el servidor (ejemplo con systemd)
sudo systemctl edit auth-dependent-service
# Agregar: Environment="AUTH_API_KEY=nueva_clave_aqui"
sudo systemctl daemon-reload
sudo systemctl restart auth-dependent-service
```

### Solución B: Corregir Variables de Entorno

```bash
# Archivo .env (desarrollo local)
AUTH_API_URL=https://auth.example.com
AUTH_API_KEY=sk-dev-xxxxxxxxxxxxxxxxxxxx
AUTH_CLIENT_ID=dev_client_001
AUTH_CLIENT_SECRET=dev_secret_abc123

# Verificar que el archivo .env está siendo cargado
node -e "require('dotenv').config(); console.log(process.env.AUTH_API_URL)"
```

### Solución C: Agregar IP a la Lista Blanca

1. Obtener la IP pública del servidor de aplicación:
   ```bash
   curl -s https://api.ipify.org
   ```
2. Contactar al administrador del servicio de autenticación para agregar la IP.
3. Verificar en el panel del proveedor: **Security → IP Whitelist → Add IP**.

---

## 📊 Checklist de Verificación

Antes de escalar el problema, confirmar que se revisaron todos los puntos:

- [ ] Variables de entorno definidas y con valores correctos
- [ ] API Key/Token no expirado
- [ ] Prueba directa con `curl` realizada
- [ ] Logs del servidor de autenticación revisados
- [ ] Conectividad de red verificada (DNS, TCP, SSL)
- [ ] IP del servidor en la lista blanca del proveedor
- [ ] Cabeceras HTTP correctamente configuradas en el código
- [ ] Entorno correcto (dev/staging/prod) configurado

---

## 📁 Archivos Relacionados

```
projectFinalPart1/
├── src/
│   ├── services/
│   │   └── authService.js        # Cliente de la API de autenticación
│   ├── middleware/
│   │   └── authMiddleware.js     # Middleware de validación de tokens
│   └── config/
│       └── auth.config.js        # Configuración del módulo de autenticación
├── .env.example                  # Plantilla de variables de entorno
└── docs/
    └── troubleshooting/
        └── auth-api-not-authorized.md  # Este documento
```

---

## 📞 Escalamiento

Si después de seguir todos los pasos anteriores el problema persiste:

| Nivel | Contacto | Cuándo escalar |
|-------|----------|----------------|
| **L1** | Equipo de desarrollo | Primeras 2 horas |
| **L2** | Arquitecto de soluciones | Si el problema es de infraestructura |
| **L3** | Proveedor de autenticación (soporte) | Si el problema es del lado del proveedor |

**Información a incluir al escalar:**
- Resultado del `curl` de diagnóstico (sin exponer credenciales reales)
- Logs relevantes del servidor
- Timestamp exacto del primer error detectado
- Entornos afectados

---

## 📝 Historial de Cambios

| Fecha | Autor | Descripción |
|-------|-------|-------------|
| 2026-02-17 | Equipo Backend | Creación inicial del documento |

---

*Documento generado para el proyecto `projectFinalPart1` — Rama: `feature/auth-api-connection`*
