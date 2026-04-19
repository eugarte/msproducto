# Requerimientos del Microservicio de Productos (msproducto)

## Información General

| Campo | Valor |
|-------|-------|
| **Nombre del Microservicio** | msproducto |
| **Descripción** | Microservicio de catálogo de productos con esquema dinámico y atributos configurables |
| **Tecnología Base** | Node.js + Express |
| **Base de Datos** | MySQL (Hostinger) |
| **Arquitectura** | Microservicio Independiente |
| **Ambiente de Despliegue** | Hostinger Node.js Hosting |

---

## 1. Requerimientos Funcionales

### 1.1 Gestión de Productos

| ID | Requerimiento | Prioridad | Descripción |
|----|---------------|-----------|-------------|
| RF-001 | CRUD de Productos | Alta | Crear, leer, actualizar y eliminar productos con atributos dinámicos |
| RF-002 | Esquema Dinámico | Alta | Almacenar diferentes tipos de productos con atributos variables sin modificar la estructura de BD |
| RF-003 | Categorización | Alta | Organizar productos en categorías jerárquicas ilimitadas |
| RF-004 | Variantes de Producto | Media | Soporte para variantes (tallas, colores, etc.) con stock independiente |
| RF-005 | Gestión de Stock | Alta | Control de inventario con alertas de stock bajo |
| RF-006 | Precios Múltiples | Media | Precio de venta, precio comparativo, costo |
| RF-007 | Imágenes de Producto | Media | Soportar múltiples imágenes por producto y variante |
| RF-008 | Estados de Producto | Alta | Estados: borrador, activo, inactivo, descontinuado |
| RF-009 | Productos Destacados | Baja | Marcar productos como destacados |
| RF-010 | Relaciones entre Productos | Baja | Productos relacionados, upsell, cross-sell |

### 1.2 Atributos Dinámicos (Core del Sistema)

| ID | Requerimiento | Prioridad | Descripción |
|----|---------------|-----------|-------------|
| RF-011 | Definición de Atributos | Alta | Crear atributos personalizados por categoría de producto |
| RF-012 | Tipos de Datos | Alta | Soportar: string, number, boolean, date, enum, multiselect |
| RF-013 | Validación de Atributos | Alta | Validar tipo de dato, requerido, rangos, patrones regex |
| RF-014 | Atributos Indexables | Media | Definir qué atributos se pueden usar en filtros de búsqueda |
| RF-015 | Valores por Defecto | Media | Asignar valores por defecto a atributos |
| RF-016 | Opciones Predefinidas | Media | Para enums/multiselect: definir lista de opciones válidas |
| RF-017 | Atributos Globales | Baja | Atributos que aplican a todas las categorías |

### 1.3 Búsqueda y Filtrado

| ID | Requerimiento | Prioridad | Descripción |
|----|---------------|-----------|-------------|
| RF-018 | Búsqueda de Texto | Alta | Buscar por nombre, descripción, SKU |
| RF-019 | Filtros por Categoría | Alta | Filtrar productos por categoría y subcategorías |
| RF-020 | Filtros por Atributos | Alta | Filtrar por atributos dinámicos (color, talla, marca, etc.) |
| RF-021 | Filtros por Rango | Media | Filtrar por rangos de precio, stock, atributos numéricos |
| RF-022 | Ordenamiento | Alta | Ordenar por precio, nombre, fecha, popularidad |
| RF-023 | Paginación | Alta | Paginación tradicional y cursor-based para grandes datasets |
| RF-024 | Búsqueda Avanzada | Media | Combinar múltiples filtros con operadores AND/OR |

### 1.4 Integraciones

| ID | Requerimiento | Prioridad | Descripción |
|----|---------------|-----------|-------------|
| RF-025 | API REST | Alta | Exponer endpoints RESTful JSON |
| RF-026 | Health Check | Alta | Endpoint de verificación de estado del servicio |
| RF-027 | Documentación API | Media | OpenAPI/Swagger para documentación automática |
| RF-028 | CORS | Alta | Soportar peticiones cross-origin |

---

## 2. Requerimientos No Funcionales

### 2.1 Rendimiento

| ID | Requerimiento | Objetivo | Descripción |
|----|---------------|----------|-------------|
| RNF-001 | Tiempo de Respuesta | < 200ms | 95% de las peticiones deben responder en menos de 200ms |
| RNF-002 | Throughput | 100 req/s | Soportar 100 peticiones concurrentes |
| RNF-003 | Conexiones DB | < 20 | Mantener pool de conexiones MySQL máximo 20 |
| RNF-004 | Optimización de Queries | Alta | Usar índices y virtual columns para búsquedas frecuentes |
| RNF-005 | Paginación Eficiente | Alta | Cursor-based para datasets grandes (>10k productos) |

### 2.2 Seguridad

| ID | Requerimiento | Descripción |
|----|---------------|-------------|
| RNF-006 | Validación de Inputs | Sanitizar y validar todos los inputs de usuario |
| RNF-007 | SQL Injection Prevention | Usar prepared statements obligatoriamente |
| RNF-008 | XSS Prevention | Escapar outputs en respuestas JSON |
| RNF-009 | Rate Limiting | Limitar peticiones: 100/min por IP |
| RNF-010 | HTTPS Only | Forzar conexiones HTTPS en producción |
| RNF-011 | Variables de Entorno | Nunca exponer credenciales en código |

### 2.3 Disponibilidad y Escalabilidad

| ID | Requerimiento | Descripción |
|----|---------------|-------------|
| RNF-012 | Uptime | 99.9% de disponibilidad |
| RNF-013 | Graceful Shutdown | Manejar señales SIGTERM para cierre limpio |
| RNF-014 | Reconexión DB | Reconectar automáticamente si falla conexión a BD |
| RNF-015 | Stateless | El servicio debe ser stateless para escalabilidad horizontal |

### 2.4 Mantenibilidad

| ID | Requerimiento | Descripción |
|----|---------------|-------------|
| RNF-016 | Logging | Logs estructurados con niveles (info, warn, error) |
| RNF-017 | Estructura Modular | Arquitectura en capas: controllers, services, repositories |
| RNF-018 | Manejo de Errores | Respuestas de error consistentes y descriptivas |
| RNF-019 | Versionado API | /api/v1/ como prefijo de rutas |
| RNF-020 | Documentación | Código documentado con JSDoc |

### 2.5 Limitaciones de Hostinger

| ID | Requerimiento | Descripción |
|----|---------------|-------------|
| RNF-021 | MySQL Only | Usar MySQL (Hostinger no soporta MariaDB, PostgreSQL, MongoDB) |
| RNF-022 | Límite de Procesos | Máximo 20 entry processes concurrentes |
| RNF-023 | Límite de Memoria | Máximo 768MB - 1GB RAM según plan |
| RNF-024 | Conexiones DB | Máximo 500 conexiones MySQL globales, ~75 por usuario |
| RNF-025 | Variables Entorno | Configurar desde hPanel, no archivos .env en producción |
| RNF-026 | Sin Redis Local | No usar Redis, caché en memoria con Map/weak límite |
| RNF-027 | Sin Servicios Externos | No depender de RabbitMQ, Kafka, Elasticsearch locales |
| RNF-028 | Puerto Asignado | Usar puerto proporcionado por Hostinger (process.env.PORT) |

---

## 3. Modelo de Datos

### 3.1 Entidades Principales

```
┌─────────────────────────────────────────────────────────────────┐
│                     DIAGRAMA DE ENTIDADES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐         ┌──────────────┐                      │
│  │  categories  │         │   products   │                      │
│  ├──────────────┤         ├──────────────┤                      │
│  │ id (PK)      │◄────────│ id (PK)      │                      │
│  │ parent_id    │   1:N   │ category_id  │                      │
│  │ name         │         │ sku (UQ)     │                      │
│  │ slug         │         │ name         │                      │
│  └──────────────┘         │ price        │                      │
│           ▲               │ stock        │                      │
│           │               │ attributes   │◄── JSON dinámico    │
│           │               │ status       │                      │
│           │               └──────────────┘                      │
│           │                      │                              │
│           │               ┌──────┴──────┐                       │
│           │               │             │                       │
│           │        ┌──────▼──────┐ ┌────▼─────┐                │
│           │        │   variants  │ │  images  │                │
│           │        └─────────────┘ └──────────┘                │
│           │                                                      │
│           │         ┌──────────────────────┐                    │
│           └─────────│ attribute_definitions│                    │
│                     └──────────────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Estructura de JSON de Atributos

```json
// Ejemplo Laptop
{
  "cpu": "Intel Core i9-13900H",
  "ram_gb": 32,
  "storage": "1TB NVMe SSD",
  "screen_size": 15.6,
  "gpu": "NVIDIA RTX 4060",
  "color": "Silver",
  "weight_kg": 1.86,
  "ports": ["USB-C", "Thunderbolt 4", "HDMI"],
  "warranty_years": 2
}

// Ejemplo Camiseta
{
  "color": "Black",
  "size": "M",
  "material": "100% Cotton",
  "weight_kg": 0.15,
  "care_instructions": ["Machine wash cold", "Tumble dry low"]
}
```

---

## 4. API Endpoints Requeridos

### 4.1 Productos

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | /api/v1/products | Listar productos (con filtros) |
| GET | /api/v1/products/:id | Obtener producto por ID |
| POST | /api/v1/products | Crear producto |
| PUT | /api/v1/products/:id | Actualizar producto |
| PATCH | /api/v1/products/:id/attributes | Actualizar atributos dinámicos |
| DELETE | /api/v1/products/:id | Eliminar producto |
| POST | /api/v1/products/search | Búsqueda avanzada |

### 4.2 Categorías

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | /api/v1/categories | Listar categorías (árbol) |
| GET | /api/v1/categories/:id | Obtener categoría |
| GET | /api/v1/categories/:id/products | Productos de categoría |
| POST | /api/v1/categories | Crear categoría |
| PUT | /api/v1/categories/:id | Actualizar categoría |
| DELETE | /api/v1/categories/:id | Eliminar categoría |

### 4.3 Atributos (Metadata)

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | /api/v1/attributes | Listar definiciones de atributos |
| GET | /api/v1/attributes/:id | Obtener definición |
| GET | /api/v1/categories/:id/attributes | Atributos de una categoría |
| POST | /api/v1/attributes | Crear definición de atributo |
| PUT | /api/v1/attributes/:id | Actualizar definición |
| DELETE | /api/v1/attributes/:id | Eliminar definición |

### 4.4 Sistema

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | /health | Health check |
| GET | /api/v1/docs | Documentación OpenAPI |

---

## 5. Flujos de Negocio

### 5.1 Crear Producto con Atributos Dinámicos

```
1. Cliente solicita atributos requeridos para categoría X
   → GET /api/v1/categories/X/attributes
   
2. Cliente envía datos del producto con atributos
   → POST /api/v1/products
   
3. Servicio valida atributos según metadata
   → Valida tipos de datos, requeridos, reglas
   
4. Servicio guarda producto
   → INSERT con JSON de atributos
   
5. Respuesta con producto creado
```

### 5.2 Buscar Productos con Filtros

```
1. Cliente envía filtros
   → GET /api/v1/products?category=1&color=red&min_price=100
   
2. Servicio construye query dinámica
   → Usa columnas indexadas + virtual columns
   
3. Ejecuta query optimizada
   → SELECT con índices
   
4. Retorna resultados paginados
```

---

## 6. Dependencias Técnicas

### 6.1 Paquetes NPM Requeridos

| Paquete | Versión | Propósito |
|---------|---------|-----------|
| express | ^4.18.x | Framework web |
| mysql2 | ^3.9.x | Driver MySQL |
| cors | ^2.8.x | CORS middleware |
| helmet | ^7.1.x | Seguridad HTTP headers |
| express-rate-limit | ^7.1.x | Rate limiting |
| joi | ^17.12.x | Validación de schemas |
| dotenv | ^16.4.x | Variables de entorno (dev) |
| morgan | ^1.10.x | Logging de requests |
| slugify | ^1.6.x | Generar slugs |

### 6.2 Dependencias de Desarrollo

| Paquete | Propósito |
|---------|-----------|
| nodemon | Hot reload en desarrollo |
| jest | Testing |
| supertest | Testing de API |

---

## 7. Criterios de Aceptación

### 7.1 Funcionales

- [ ] Se pueden crear productos de diferentes categorías con atributos distintos
- [ ] Los atributos se validan según su tipo de dato y reglas definidas
- [ ] Se pueden buscar productos por atributos dinámicos
- [ ] El sistema soporta 1000+ productos sin degradación de performance
- [ ] La API responde en < 200ms para consultas comunes

### 7.2 No Funcionales

- [ ] Código cobertura de tests > 80%
- [ ] Documentación OpenAPI generada automáticamente
- [ ] Logs estructurados en todos los endpoints
- [ ] Manejo de errores consistente
- [ ] Despliegue exitoso en Hostinger

---

## 8. Consideraciones Especiales Hostinger

### 8.1 Restricciones

| Aspecto | Restricción | Solución |
|---------|-------------|----------|
| Base de Datos | MySQL únicamente | Usar MySQL 8.0 con soporte JSON |
| Memoria | 768MB - 1GB | Limitar pool de conexiones, sin caché externa |
| Procesos | 20 entry processes | Código async/await eficiente |
| Almacenamiento | NVMe SSD | Optimizar queries, índices correctos |
| Conexiones DB | ~75 por usuario | Pool de máximo 15 conexiones |

### 8.2 Configuración Requerida en Hostinger

```
hPanel → Node.js → Configuración:
- Node.js version: 18.x o 20.x
- Application mode: production
- Startup file: src/app.js
- Environment Variables:
  * NODE_ENV=production
  * PORT=3000 (o asignado por Hostinger)
  * DB_HOST=localhost
  * DB_PORT=3306
  * DB_USER=u123456789_msproducto
  * DB_PASSWORD=********
  * DB_NAME=u123456789_msproducto
  * DB_POOL_SIZE=15
```

---

**Versión:** 1.0  
**Fecha:** 2024  
**Autor:** ZagaloAI
