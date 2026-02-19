# 📋 Documento Técnico de Testing — API de Autenticación y Dashboard

**Proyecto:** Final Part 1 — Diplomado  
**Versión:** 1.0.0  
**Fecha:** 2026-02-18  
**Autor:** QA Engineering Team  
**Estado:** Simulación de pruebas — Resultados documentados

---

## Tabla de Contenidos

1. [Alcance del Documento](#alcance)
2. [Entorno de Pruebas](#entorno)
3. [API de Autenticación — Login](#api-autenticacion)
   - [Descripción del Bug](#descripcion-bug-login)
   - [Casos de Prueba — Login](#casos-prueba-login)
   - [Resultados](#resultados-login)
4. [API de Dashboard — Paginación de Items](#api-dashboard)
   - [Descripción de la Funcionalidad](#descripcion-paginacion)
   - [Casos de Prueba — Paginación](#casos-prueba-paginacion)
   - [Resultados](#resultados-paginacion)
5. [Resumen de Defectos Encontrados](#defectos)
6. [Conclusiones y Recomendaciones](#conclusiones)

---

## 1. Alcance del Documento {#alcance}

Este documento cubre la simulación de pruebas funcionales y de integración para dos áreas críticas del sistema:

- **API de Autenticación (`/auth/login`):** Identificación y reproducción del defecto donde el endpoint de login no era encontrado (`404 Not Found`).
- **API de Dashboard (`/dashboard/items`):** Validación de la funcionalidad de paginación de ítems, incluyendo pruebas de bordes y casos negativos.

> **Nota:** Las pruebas fueron ejecutadas sobre un entorno de staging. Los datos utilizados son ficticios y no representan usuarios o registros reales.

---

## 2. Entorno de Pruebas {#entorno}

| Parámetro           | Valor                                |
|---------------------|--------------------------------------|
| **Base URL**        | `https://api.staging.projectfinal.io` |
| **Protocolo**       | HTTPS                                |
| **Formato**         | JSON (`application/json`)            |
| **Autenticación**   | Bearer Token (JWT)                   |
| **Herramientas**    | Postman v11, cURL, Jest + Supertest  |
| **Versión de API**  | v1                                   |
| **Fecha de ejecución** | 2026-02-18                        |

---

## 3. API de Autenticación — Login {#api-autenticacion}

### 3.1 Descripción del Bug {#descripcion-bug-login}

Durante la fase inicial de pruebas, se detectó que el endpoint de autenticación `/api/v1/auth/login` retornaba un código **`404 Not Found`**, impidiendo completamente el acceso al sistema.

**Causa Raíz Identificada:**  
La ruta estaba registrada con un prefijo incorrecto en el enrutador principal. El router de autenticación estaba montado en `/auth` en lugar de `/api/v1/auth`, generando una discrepancia entre la ruta documentada y la ruta real.

```diff
// router.js — ANTES (incorrecto)
- app.use('/auth', authRouter);

// router.js — DESPUÉS (corregido)
+ app.use('/api/v1/auth', authRouter);
```

**Severidad:** 🔴 Crítica — Bloqueante  
**Prioridad:** Alta  
**Estado del defecto:** ✅ Resuelto

---

### 3.2 Casos de Prueba — Login {#casos-prueba-login}

#### TC-AUTH-001 — Login Exitoso con Credenciales Válidas

| Campo         | Detalle                            |
|---------------|------------------------------------|
| **ID**        | TC-AUTH-001                        |
| **Módulo**    | Autenticación                      |
| **Endpoint**  | `POST /api/v1/auth/login`          |
| **Prioridad** | Alta                               |
| **Estado**    | ✅ PASÓ (luego de corrección)      |

**Request:**
```http
POST /api/v1/auth/login HTTP/1.1
Host: api.staging.projectfinal.io
Content-Type: application/json

{
  "email": "usuario@ejemplo.com",
  "password": "Password123!"
}
```

**Response Esperada:**
```json
{
  "status": "success",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
    "expiresIn": 3600,
    "user": {
      "id": "usr_9k2mX1pQ",
      "email": "usuario@ejemplo.com",
      "role": "admin"
    }
  }
}
```

**Response HTTP:** `200 OK`

**Resultado:** ✅ PASÓ — Token JWT retornado correctamente.

---

#### TC-AUTH-002 — Login con Endpoint No Encontrado (Bug Original)

| Campo         | Detalle                            |
|---------------|------------------------------------|
| **ID**        | TC-AUTH-002                        |
| **Módulo**    | Autenticación                      |
| **Endpoint**  | `POST /auth/login` *(ruta errónea)*|
| **Prioridad** | Alta                               |
| **Estado**    | 🔴 FALLÓ — Bug documentado         |

**Request (reproducción del bug):**
```http
POST /auth/login HTTP/1.1
Host: api.staging.projectfinal.io
Content-Type: application/json

{
  "email": "usuario@ejemplo.com",
  "password": "Password123!"
}
```

**Response Obtenida:**
```json
{
  "status": "error",
  "statusCode": 404,
  "message": "Cannot POST /auth/login",
  "timestamp": "2026-02-18T18:30:00.000Z"
}
```

**Response HTTP:** `404 Not Found`

**Resultado:** 🔴 FALLÓ — El endpoint no fue encontrado. Defecto registrado como **BUG-AUTH-001**.

---

#### TC-AUTH-003 — Login con Contraseña Incorrecta

| Campo         | Detalle                            |
|---------------|------------------------------------|
| **ID**        | TC-AUTH-003                        |
| **Módulo**    | Autenticación                      |
| **Endpoint**  | `POST /api/v1/auth/login`          |
| **Prioridad** | Media                              |
| **Estado**    | ✅ PASÓ                            |

**Request:**
```http
POST /api/v1/auth/login HTTP/1.1
Content-Type: application/json

{
  "email": "usuario@ejemplo.com",
  "password": "ContraseñaIncorrecta"
}
```

**Response Esperada:**
```json
{
  "status": "error",
  "statusCode": 401,
  "message": "Credenciales inválidas"
}
```

**Response HTTP:** `401 Unauthorized`

**Resultado:** ✅ PASÓ — El sistema rechazó correctamente las credenciales incorrectas.

---

#### TC-AUTH-004 — Login con Campos Faltantes (Validación)

| Campo         | Detalle                            |
|---------------|------------------------------------|
| **ID**        | TC-AUTH-004                        |
| **Módulo**    | Autenticación                      |
| **Endpoint**  | `POST /api/v1/auth/login`          |
| **Prioridad** | Media                              |
| **Estado**    | ✅ PASÓ                            |

**Request:**
```http
POST /api/v1/auth/login HTTP/1.1
Content-Type: application/json

{
  "email": "usuario@ejemplo.com"
}
```

**Response Esperada:**
```json
{
  "status": "error",
  "statusCode": 400,
  "message": "El campo 'password' es requerido",
  "errors": [
    { "field": "password", "message": "El campo es requerido" }
  ]
}
```

**Response HTTP:** `400 Bad Request`

**Resultado:** ✅ PASÓ — Validación de campos obligatorios funcionando correctamente.

---

### 3.3 Resultados — Autenticación {#resultados-login}

| ID Test        | Descripción                            | Estado   |
|----------------|----------------------------------------|----------|
| TC-AUTH-001    | Login exitoso con credenciales válidas | ✅ PASÓ  |
| TC-AUTH-002    | Login con endpoint incorrecto (bug)    | 🔴 FALLÓ |
| TC-AUTH-003    | Login con contraseña incorrecta        | ✅ PASÓ  |
| TC-AUTH-004    | Login sin campo requerido              | ✅ PASÓ  |

**Tasa de éxito (post-fix):** 4/4 — 100% ✅

---

## 4. API de Dashboard — Paginación de Items {#api-dashboard}

### 4.1 Descripción de la Funcionalidad {#descripcion-paginacion}

El endpoint `GET /api/v1/dashboard/items` retorna una lista paginada de ítems del dashboard del usuario autenticado. Soporta los siguientes parámetros de query:

| Parámetro  | Tipo     | Requerido | Descripción                          | Default |
|------------|----------|-----------|--------------------------------------|---------|
| `page`     | `integer`| No        | Número de página (1-indexed)         | `1`     |
| `limit`    | `integer`| No        | Cantidad de ítems por página         | `10`    |
| `sort`     | `string` | No        | Campo de ordenamiento (`asc`/`desc`) | `createdAt:desc` |
| `search`   | `string` | No        | Filtro de búsqueda por nombre        | -       |

**Schema de respuesta paginada:**
```json
{
  "status": "success",
  "data": {
    "items": [ /* ... array de ítems */ ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 15,
      "totalItems": 148,
      "itemsPerPage": 10,
      "hasNextPage": true,
      "hasPrevPage": false
    }
  }
}
```

---

### 4.2 Casos de Prueba — Paginación {#casos-prueba-paginacion}

#### TC-DASH-001 — Paginación por Defecto (Sin Parámetros)

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-001                          |
| **Módulo**    | Dashboard — Paginación               |
| **Endpoint**  | `GET /api/v1/dashboard/items`        |
| **Prioridad** | Alta                                 |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Esperada:**
```json
{
  "status": "success",
  "data": {
    "items": [
      { "id": "item_001", "name": "Reporte Enero", "createdAt": "2026-01-15T10:00:00Z" },
      { "id": "item_002", "name": "Reporte Febrero", "createdAt": "2026-02-01T10:00:00Z" }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 15,
      "totalItems": 148,
      "itemsPerPage": 10,
      "hasNextPage": true,
      "hasPrevPage": false
    }
  }
}
```

**Response HTTP:** `200 OK`

**Resultado:** ✅ PASÓ — Retorna primera página con 10 ítems por defecto.

---

#### TC-DASH-002 — Paginación con Parámetros Personalizados

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-002                          |
| **Módulo**    | Dashboard — Paginación               |
| **Endpoint**  | `GET /api/v1/dashboard/items?page=3&limit=5` |
| **Prioridad** | Alta                                 |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items?page=3&limit=5 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Esperada:**
```json
{
  "status": "success",
  "data": {
    "items": [ /* 5 ítems de la página 3 */ ],
    "pagination": {
      "currentPage": 3,
      "totalPages": 30,
      "totalItems": 148,
      "itemsPerPage": 5,
      "hasNextPage": true,
      "hasPrevPage": true
    }
  }
}
```

**Response HTTP:** `200 OK`

**Resultado:** ✅ PASÓ — Paginación parametrizada retorna datos correctos.

---

#### TC-DASH-003 — Última Página (Verificación de `hasNextPage: false`)

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-003                          |
| **Módulo**    | Dashboard — Paginación               |
| **Endpoint**  | `GET /api/v1/dashboard/items?page=15&limit=10` |
| **Prioridad** | Media                                |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items?page=15&limit=10 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Esperada:**
```json
{
  "status": "success",
  "data": {
    "items": [ /* 8 ítems (última página con menos ítems) */ ],
    "pagination": {
      "currentPage": 15,
      "totalPages": 15,
      "totalItems": 148,
      "itemsPerPage": 10,
      "hasNextPage": false,
      "hasPrevPage": true
    }
  }
}
```

**Response HTTP:** `200 OK`

**Resultado:** ✅ PASÓ — Última página correctamente identificada con `hasNextPage: false`.

---

#### TC-DASH-004 — Página Fuera de Rango

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-004                          |
| **Módulo**    | Dashboard — Paginación               |
| **Endpoint**  | `GET /api/v1/dashboard/items?page=999` |
| **Prioridad** | Media                                |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items?page=999 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Esperada:**
```json
{
  "status": "success",
  "data": {
    "items": [],
    "pagination": {
      "currentPage": 999,
      "totalPages": 15,
      "totalItems": 148,
      "itemsPerPage": 10,
      "hasNextPage": false,
      "hasPrevPage": true
    }
  }
}
```

**Response HTTP:** `200 OK`

**Resultado:** ✅ PASÓ — Retorna array vacío sin error cuando la página excede el total.

---

#### TC-DASH-005 — Parámetros Inválidos (Valor No Numérico)

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-005                          |
| **Módulo**    | Dashboard — Paginación               |
| **Endpoint**  | `GET /api/v1/dashboard/items?page=abc&limit=xyz` |
| **Prioridad** | Media                                |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items?page=abc&limit=xyz HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Esperada:**
```json
{
  "status": "error",
  "statusCode": 400,
  "message": "Parámetros de paginación inválidos",
  "errors": [
    { "field": "page", "message": "Debe ser un número entero positivo" },
    { "field": "limit", "message": "Debe ser un número entero positivo" }
  ]
}
```

**Response HTTP:** `400 Bad Request`

**Resultado:** ✅ PASÓ — Validación de tipos de datos en parámetros de query funciona correctamente.

---

#### TC-DASH-006 — Solicitud Sin Token de Autenticación

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-006                          |
| **Módulo**    | Dashboard — Paginación               |
| **Endpoint**  | `GET /api/v1/dashboard/items`        |
| **Prioridad** | Alta                                 |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items HTTP/1.1
```

**Response Esperada:**
```json
{
  "status": "error",
  "statusCode": 401,
  "message": "Token de autenticación requerido"
}
```

**Response HTTP:** `401 Unauthorized`

**Resultado:** ✅ PASÓ — Middleware de autenticación protege correctamente el endpoint.

---

#### TC-DASH-007 — Búsqueda con Filtro y Paginación Combinados

| Campo         | Detalle                              |
|---------------|--------------------------------------|
| **ID**        | TC-DASH-007                          |
| **Módulo**    | Dashboard — Paginación + Búsqueda    |
| **Endpoint**  | `GET /api/v1/dashboard/items?search=Reporte&page=1&limit=5` |
| **Prioridad** | Media                                |
| **Estado**    | ✅ PASÓ                              |

**Request:**
```http
GET /api/v1/dashboard/items?search=Reporte&page=1&limit=5 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response Esperada:**
```json
{
  "status": "success",
  "data": {
    "items": [
      { "id": "item_001", "name": "Reporte Enero", "createdAt": "2026-01-15T10:00:00Z" },
      { "id": "item_002", "name": "Reporte Febrero", "createdAt": "2026-02-01T10:00:00Z" }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 4,
      "totalItems": 17,
      "itemsPerPage": 5,
      "hasNextPage": true,
      "hasPrevPage": false
    }
  }
}
```

**Response HTTP:** `200 OK`

**Resultado:** ✅ PASÓ — Filtrado por búsqueda combinado con paginación retorna datos correctos y ajusta `totalItems` al subconjunto filtrado.

---

### 4.3 Resultados — Paginación Dashboard {#resultados-paginacion}

| ID Test        | Descripción                                       | Estado   |
|----------------|---------------------------------------------------|----------|
| TC-DASH-001    | Paginación por defecto                            | ✅ PASÓ  |
| TC-DASH-002    | Parámetros personalizados (page=3, limit=5)       | ✅ PASÓ  |
| TC-DASH-003    | Última página (hasNextPage: false)                | ✅ PASÓ  |
| TC-DASH-004    | Página fuera de rango                             | ✅ PASÓ  |
| TC-DASH-005    | Parámetros inválidos (no numéricos)               | ✅ PASÓ  |
| TC-DASH-006    | Sin token de autenticación                        | ✅ PASÓ  |
| TC-DASH-007    | Búsqueda + Paginación combinados                  | ✅ PASÓ  |

**Tasa de éxito:** 7/7 — 100% ✅

---

## 5. Resumen de Defectos Encontrados {#defectos}

| ID Bug       | API          | Severidad  | Descripción                                             | Estado       |
|--------------|--------------|------------|---------------------------------------------------------|--------------|
| BUG-AUTH-001 | Autenticación| 🔴 Crítica | Endpoint `/auth/login` retornaba `404 Not Found`        | ✅ Resuelto  |

### Detalle del Defecto BUG-AUTH-001

```
Título:       Endpoint de login retorna 404 Not Found
Componente:   API — Módulo de Autenticación
Severidad:    Crítica (Bloqueante)
Prioridad:    Alta
Reportado:    2026-02-18
Resuelto:     2026-02-18

Pasos para reproducir:
  1. Enviar POST /auth/login con credenciales válidas
  2. Observar respuesta 404 Not Found

Causa Raíz:
  El router de autenticación estaba montado con el prefijo '/auth'
  en lugar de '/api/v1/auth', causando que ninguna ruta de autenticación
  fuera accesible bajo el prefijo de versión correcto.

Resolución:
  Actualización del archivo router.js: cambio de app.use('/auth', authRouter)
  a app.use('/api/v1/auth', authRouter).

Verificación post-fix:
  TC-AUTH-001 ejecutado exitosamente. Login retorna 200 OK con JWT válido.
```

---

## 6. Conclusiones y Recomendaciones {#conclusiones}

### ✅ Conclusiones

1. **Módulo de Autenticación:** Tras la corrección del defecto de enrutamiento (`BUG-AUTH-001`), todos los flujos de autenticación funcionan correctamente. La validación de campos, manejo de credenciales incorrectas y generación de tokens JWT operan según lo especificado.

2. **Módulo de Paginación (Dashboard):** La API de paginación de ítems supera satisfactoriamente todos los casos de prueba definidos, incluyendo escenarios de borde (página fuera de rango, parámetros inválidos) y combinaciones de filtros con paginación.

### 🔧 Recomendaciones

1. **Centralizar el prefijo de API:** Implementar un middleware o configuración centralizada que aplique automáticamente el prefijo `/api/v1` a todos los routers, evitando inconsistencias como el bug `BUG-AUTH-001`.

2. **Pruebas de regresión automatizadas:** Añadir pruebas automatizadas con Jest + Supertest para el endpoint de login que validen la ruta correcta en cada ciclo de CI/CD.

3. **Límite máximo de paginación:** Considerar implementar un límite máximo para el parámetro `limit` (ej. máximo 100) para prevenir consultas excesivamente grandes al servidor.

4. **Documentación de API (OpenAPI/Swagger):** Mantener la especificación de Swagger sincronizada con las rutas reales para detectar inconsistencias de enrutamiento de forma temprana.

5. **Pruebas de carga para Paginación:** Ejecutar pruebas de rendimiento (k6 o Artillery) sobre el endpoint de paginación con conjuntos de datos grandes (10,000+ ítems) para validar tiempos de respuesta aceptables.

---

*Documento generado por el equipo de QA — Diplomado Proyecto Final Part 1*  
*Fecha de emisión: 2026-02-18 | Revisión: 1.0*
