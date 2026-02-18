# 📄 Requerimiento: Paginación para Listado de Items — Datasets > 500,000 Registros

> **Rama:** `feature/pagination-large-datasets`  
> **Fecha:** 2026-02-17  
> **Severidad:** 🟠 Media-Alta  
> **Tipo:** Mejora de rendimiento / Requerimiento funcional  
> **Estado:** En desarrollo

---

## 📋 Descripción del Problema

La API actual retorna **todos los items en una sola respuesta** sin ningún mecanismo de paginación. Con colecciones que superan los **500,000 registros**, esto genera:

- Tiempos de respuesta superiores a 30 segundos.
- Consumo excesivo de memoria en el servidor (heap overflow).
- Timeouts en el cliente antes de recibir la respuesta completa.
- Degradación del rendimiento general del sistema para otros usuarios concurrentes.
- Riesgo de caída total del servicio bajo carga alta.

**Ejemplo del problema actual:**

```http
GET /api/items
→ Respuesta: 500,000+ objetos JSON en un solo payload (~2.3 GB)
→ Tiempo de respuesta: 45s - 120s
→ Resultado: Timeout / Out of Memory
```

---

## 🎯 Objetivo

Implementar un sistema de **paginación eficiente** que:

1. Limite la cantidad de registros por respuesta (máximo recomendado: **100 items/página**).
2. Permita al cliente navegar entre páginas de forma controlada.
3. Reduzca el tiempo de respuesta a menos de **500ms por página**.
4. No sobrecargue la base de datos ni la API con consultas masivas.

---

## 🧩 Estrategias de Paginación Disponibles

### Comparativa

| Estrategia | Ventajas | Desventajas | Recomendada para |
|------------|----------|-------------|------------------|
| **Offset / Limit** | Simple de implementar | Lento en páginas altas (OFFSET grande) | Datasets < 100k |
| **Cursor-based** | Rápido, consistente | Más complejo de implementar | Datasets > 100k ✅ |
| **Keyset Pagination** | Muy eficiente en BD | Requiere campo ordenable único | Datasets muy grandes ✅ |
| **Page Token** | Opaco, seguro | Estado en servidor | APIs públicas |

> 💡 **Para este caso (+500k items) se recomienda Cursor-based o Keyset Pagination.**

---

## 🏗️ Diseño de la Solución

### Opción 1: Paginación por Offset/Limit (Simple)

```http
GET /api/items?page=1&limit=100
GET /api/items?page=2&limit=100
GET /api/items?offset=0&limit=100
```

**Respuesta:**

```json
{
  "data": [ /* 100 items */ ],
  "pagination": {
    "total": 523847,
    "page": 1,
    "limit": 100,
    "totalPages": 5239,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

**Problema con datasets grandes:**

```sql
-- Esta consulta es LENTA en páginas altas
SELECT * FROM items ORDER BY id LIMIT 100 OFFSET 499900;
-- La BD debe recorrer 499,900 filas antes de retornar las 100 deseadas
```

---

### Opción 2: Paginación por Cursor ✅ (Recomendada)

```http
# Primera página
GET /api/items?limit=100

# Páginas siguientes (usando el cursor de la respuesta anterior)
GET /api/items?limit=100&cursor=eyJpZCI6MTAwfQ==
```

**Respuesta:**

```json
{
  "data": [ /* 100 items */ ],
  "pagination": {
    "limit": 100,
    "nextCursor": "eyJpZCI6MjAwfQ==",
    "prevCursor": null,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

**Consulta SQL eficiente:**

```sql
-- Usando el ID del último item de la página anterior como cursor
SELECT * FROM items
WHERE id > :last_id          -- Keyset condition (usa índice)
ORDER BY id ASC
LIMIT 100;
-- Tiempo constante independientemente de la página
```

---

## 💻 Implementación

### Backend — Endpoint con Cursor Pagination

```javascript
// routes/items.routes.js
router.get('/api/items', async (req, res) => {
  const limit = Math.min(parseInt(req.query.limit) || 100, 500); // máx 500
  const cursor = req.query.cursor ? decodeCursor(req.query.cursor) : null;

  try {
    const items = await ItemService.getPaginated({ limit, cursor });
    res.json(items);
  } catch (error) {
    res.status(500).json({ error: 'Error al obtener items' });
  }
});
```

```javascript
// services/item.service.js
class ItemService {
  static async getPaginated({ limit, cursor }) {
    const whereClause = cursor ? { id: { $gt: cursor.lastId } } : {};

    // Obtener limit+1 para saber si hay página siguiente
    const items = await Item.findAll({
      where: whereClause,
      order: [['id', 'ASC']],
      limit: limit + 1,
    });

    const hasNextPage = items.length > limit;
    const data = hasNextPage ? items.slice(0, limit) : items;

    const nextCursor = hasNextPage
      ? encodeCursor({ lastId: data[data.length - 1].id })
      : null;

    return {
      data,
      pagination: {
        limit,
        hasNextPage,
        nextCursor,
        count: data.length,
      },
    };
  }
}

// Helpers para cursor
const encodeCursor = (payload) =>
  Buffer.from(JSON.stringify(payload)).toString('base64');

const decodeCursor = (cursor) =>
  JSON.parse(Buffer.from(cursor, 'base64').toString('utf8'));
```

---

### Frontend — Consumo de la API Paginada

```javascript
// services/api.service.js

class ItemsApiService {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.defaultLimit = 100;
  }

  /**
   * Obtiene una página de items
   * @param {string|null} cursor - Cursor de la página anterior (null = primera página)
   * @param {number} limit - Items por página
   */
  async getPage(cursor = null, limit = this.defaultLimit) {
    const params = new URLSearchParams({ limit });
    if (cursor) params.append('cursor', cursor);

    const response = await fetch(`${this.baseUrl}/api/items?${params}`);

    if (!response.ok) {
      throw new Error(`Error ${response.status}: ${response.statusText}`);
    }

    return response.json();
  }

  /**
   * Itera sobre TODOS los items de forma eficiente (sin cargar todo en memoria)
   * @param {Function} callback - Función a ejecutar por cada lote de items
   */
  async iterateAll(callback) {
    let cursor = null;
    let pageNumber = 0;

    do {
      const result = await this.getPage(cursor);
      pageNumber++;

      console.log(`Procesando página ${pageNumber} (${result.data.length} items)`);
      await callback(result.data, pageNumber);

      cursor = result.pagination.nextCursor;
    } while (cursor !== null);

    console.log(`✅ Iteración completa: ${pageNumber} páginas procesadas`);
  }
}
```

**Uso en componente UI:**

```javascript
// components/ItemsList.js
class ItemsList {
  constructor() {
    this.api = new ItemsApiService('https://api.example.com');
    this.currentCursor = null;
    this.items = [];
  }

  async loadFirstPage() {
    const result = await this.api.getPage(null, 100);
    this.items = result.data;
    this.nextCursor = result.pagination.nextCursor;
    this.render();
  }

  async loadNextPage() {
    if (!this.nextCursor) return;
    const result = await this.api.getPage(this.nextCursor, 100);
    this.items = [...this.items, ...result.data]; // append o reemplazar según UX
    this.nextCursor = result.pagination.nextCursor;
    this.render();
  }
}
```

---

## ⚙️ Configuración Recomendada

### Límites por Entorno

```bash
# .env
PAGINATION_DEFAULT_LIMIT=100
PAGINATION_MAX_LIMIT=500
PAGINATION_MIN_LIMIT=10
```

### Índices de Base de Datos Requeridos

```sql
-- CRÍTICO: Sin estos índices la paginación será lenta
CREATE INDEX idx_items_id ON items(id);
CREATE INDEX idx_items_created_at ON items(created_at);

-- Índice compuesto si se filtra por estado + fecha
CREATE INDEX idx_items_status_created ON items(status, created_at);

-- Verificar que los índices están siendo usados
EXPLAIN ANALYZE
SELECT * FROM items WHERE id > 10000 ORDER BY id ASC LIMIT 100;
```

---

## 📊 Métricas Esperadas

| Métrica | Sin Paginación | Con Paginación (Cursor) |
|---------|---------------|------------------------|
| Tiempo de respuesta | 45s - 120s | < 200ms |
| Payload por respuesta | ~2.3 GB | ~50 KB |
| Uso de memoria (servidor) | 8+ GB | < 100 MB |
| Usuarios concurrentes soportados | 1-2 | 100+ |
| Riesgo de timeout | Muy alto | Mínimo |

---

## 🧪 Plan de Pruebas

### Prueba de Carga

```bash
# Instalar k6 para pruebas de carga
# https://k6.io/docs/getting-started/installation/

# Script de prueba: load-test-pagination.js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  vus: 50,           // 50 usuarios virtuales concurrentes
  duration: '60s',
};

export default function () {
  const res = http.get('http://api.example.com/api/items?limit=100');
  check(res, {
    'status es 200': (r) => r.status === 200,
    'tiempo < 500ms': (r) => r.timings.duration < 500,
    'tiene nextCursor': (r) => JSON.parse(r.body).pagination.nextCursor !== undefined,
  });
}
```

```bash
# Ejecutar prueba
k6 run load-test-pagination.js
```

### Pruebas Unitarias

```javascript
// tests/pagination.test.js
describe('Paginación de Items', () => {
  test('Primera página retorna 100 items y nextCursor', async () => {
    const result = await ItemService.getPaginated({ limit: 100, cursor: null });
    expect(result.data).toHaveLength(100);
    expect(result.pagination.nextCursor).toBeTruthy();
    expect(result.pagination.hasNextPage).toBe(true);
  });

  test('Última página no tiene nextCursor', async () => {
    // Simular última página
    const result = await ItemService.getPaginated({
      limit: 100,
      cursor: { lastId: 523800 }, // cerca del final
    });
    expect(result.pagination.hasNextPage).toBe(false);
    expect(result.pagination.nextCursor).toBeNull();
  });

  test('Cursor inválido retorna error 400', async () => {
    const response = await request(app)
      .get('/api/items?cursor=cursor_invalido')
      .expect(400);
    expect(response.body.error).toBe('cursor_invalido');
  });

  test('Limit mayor al máximo es rechazado', async () => {
    await request(app)
      .get('/api/items?limit=10000')
      .expect(400);
  });
});
```

---

## ✅ Checklist de Implementación

- [ ] Diseño del endpoint con parámetros `cursor` y `limit`
- [ ] Implementación del servicio con cursor-based pagination
- [ ] Helpers `encodeCursor` / `decodeCursor`
- [ ] Validación de parámetros (limit máximo, cursor válido)
- [ ] Creación de índices en base de datos
- [ ] Actualización del cliente/frontend para consumir paginación
- [ ] Pruebas unitarias del servicio de paginación
- [ ] Prueba de carga con dataset real (+500k registros)
- [ ] Documentación de la API (OpenAPI/Swagger)
- [ ] Monitoreo de métricas de rendimiento post-deploy

---

## 📁 Archivos a Crear / Modificar

```
projectFinalPart1/
├── src/
│   ├── routes/
│   │   └── items.routes.js          # [MODIFICAR] Agregar parámetros de paginación
│   ├── services/
│   │   └── item.service.js          # [MODIFICAR] Implementar getPaginated()
│   ├── utils/
│   │   └── pagination.utils.js      # [NUEVO] Helpers encodeCursor/decodeCursor
│   └── middleware/
│       └── pagination.middleware.js # [NUEVO] Validación de parámetros
├── tests/
│   └── pagination.test.js           # [NUEVO] Pruebas unitarias
├── scripts/
│   └── db/
│       └── add-pagination-indexes.sql # [NUEVO] Índices de BD
└── docs/
    └── troubleshooting/
        └── pagination-large-datasets.md  # Este documento
```

---

## 📝 Historial de Cambios

| Fecha | Autor | Descripción |
|-------|-------|-------------|
| 2026-02-17 | Equipo Backend | Creación inicial del documento |

---

*Documento generado para el proyecto `projectFinalPart1` — Rama: `feature/pagination-large-datasets`*
