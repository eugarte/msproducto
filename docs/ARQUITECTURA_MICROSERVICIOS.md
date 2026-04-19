# Arquitectura de Microservicios - Relación entre Sistemas

## Diagrama General de Arquitectura

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              ECOSISTEMA DE MICROSERVICIOS                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                         CAPA DE PRESENTACIÓN                                     │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │   │
│   │  │   Web App   │  │  Mobile App │  │   Admin     │  │  External   │            │   │
│   │  │   (React)   │  │(React Native)│  │   Panel     │  │   APIs      │            │   │
│   │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │   │
│   └─────────┼────────────────┼────────────────┼────────────────┼────────────────────┘   │
│             │                │                │                │                        │
│             └────────────────┴────────────────┴────────────────┘                        │
│                                         │                                               │
│                                         ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              API GATEWAY                                         │   │
│   │  ┌─────────────────────────────────────────────────────────────────────────────┐ │   │
│   │  │  • Rate Limiting  • Authentication  • Routing  • Load Balancing           │ │   │
│   │  │  • SSL/TLS        • CORS            • Caching  • Request Validation       │ │   │
│   │  └─────────────────────────────────────────────────────────────────────────────┘ │   │
│   └────────────────────────────────────────┬────────────────────────────────────────┘   │
│                                            │                                             │
│         ┌──────────────────────────────────┼──────────────────────────────────┐          │
│         │                                  │                                  │          │
│         ▼                                  ▼                                  ▼          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         CAPA DE MICROSERVICIOS                                  │   │
│  │                                                                                  │   │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐             │   │
│  │  │   🔐 msseguridad │◄───│  👤 msclientes   │───►│  📦 msproducto  │             │   │
│  │  │   (Seguridad)    │    │   (Personas)     │    │   (Productos)    │             │   │
│  │  │                  │    │                  │    │                  │             │   │
│  │  │ • Autenticación  │    │ • Registro       │    │ • Catálogo       │             │   │
│  │  │ • Autorización   │    │ • Perfiles       │    │ • Atributos      │             │   │
│  │  │ • JWT Tokens     │    │ • Preferencias   │    │ • Stock          │             │   │
│  │  │ • Permisos       │    │ • Historial      │    │ • Precios        │             │   │
│  │  │ • Auditoría      │    │ • Segmentación   │    │ • Categorías     │             │   │
│  │  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘             │   │
│  │           │                      │                      │                       │   │
│  │           │         ┌────────────┴──────────────────────┘                       │   │
│  │           │         │                                                           │   │
│  │           │         ▼                                                           │   │
│  │           │    ┌─────────────────┐                                              │   │
│  │           │    │  🔔 msnotifica  │                                              │   │
│  │           │    │  (Notificaciones)│◄─────────────────────────────────────────────┘   │
│  │           │    │                  │                                                   │   │
│  │           │    │ • Email          │                                                   │   │
│  │           │    │ • SMS            │                                                   │   │
│  │           │    │ • Push           │                                                   │   │
│  │           │    │ • Webhook        │                                                   │   │
│  │           │    │ • Templates      │                                                   │   │
│  │           │    └─────────────────┘                                                   │   │
│  │           │                      ▲                                                   │   │
│  │           │                      │                                                   │   │
│  │           └──────────────────────┘                                                   │   │
│  │                      Valida Tokens                                                   │   │
│  │                                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────────────────┘   │
│                                            │                                             │
│         ┌──────────────────────────────────┼──────────────────────────────────┐          │
│         │                                  │                                  │          │
│         ▼                                  ▼                                  ▼          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐   │
│  │                         CAPA DE DATOS                                           │   │
│  │                                                                                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │   │
│  │  │   MySQL      │  │   MySQL      │  │   MySQL      │  │   MySQL      │         │   │
│  │  │  (Usuarios)  │  │  (Clientes)  │  │  (Productos) │  │ (Notif Logs) │         │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘         │   │
│  │                                                                                  │   │
│  └──────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Relaciones Detalladas entre Microservicios

### 1️⃣ msseguridad (Seguridad) - 🔐

**Rol:** Centro de autenticación y autorización

#### Interacciones:

| Con | Tipo | Descripción |
|-----|------|-------------|
| **msclientes** | Autenticación | Valida credenciales de login, genera tokens JWT |
| **msproducto** | Autorización | Verifica permisos para gestionar productos |
| **msnotificaciones** | Eventos | Envía alertas de seguridad (login sospechoso, cambio de contraseña) |
| **API Gateway** | Síncrono | Intercepta todas las peticiones para validar tokens |

#### Flujo de Autenticación:
```
Usuario ──► API Gateway ──► msseguridad (valida)
                              │
                              ▼
                        Genera JWT Token
                              │
                              ▼
Usuario ◄── Token ── API Gateway
```

---

### 2️⃣ msclientes (Personas/Clientes) - 👤

**Rol:** Gestión de usuarios y perfiles

#### Interacciones:

| Con | Tipo | Descripción |
|-----|------|-------------|
| **msseguridad** | Síncrono | Solicita validación de credenciales |
| **msproducto** | Síncrono | Consulta productos favoritos, historial de vistas |
| **msnotificaciones** | Eventos | Dispara notificaciones de bienvenida, recuperación de contraseña |

#### Eventos Publicados:

```javascript
// Cliente Registrado
{
  event: "cliente.registrado",
  data: {
    clienteId: "12345",
    email: "cliente@ejemplo.com",
    nombre: "Juan Pérez",
    timestamp: "2024-01-15T10:30:00Z"
  }
}

// Perfil Actualizado
{
  event: "cliente.perfil_actualizado",
  data: {
    clienteId: "12345",
    cambios: ["email", "telefono"],
    timestamp: "2024-01-15T11:00:00Z"
  }
}
```

---

### 3️⃣ msproducto (Productos) - 📦

**Rol:** Catálogo dinámico de productos

#### Interacciones:

| Con | Tipo | Descripción |
|-----|------|-------------|
| **msseguridad** | Síncrono | Valida permisos de administrador para modificaciones |
| **msclientes** | Síncrono | Recibe consultas de productos con filtros personalizados |
| **msnotificaciones** | Eventos | Notifica cambios de precio, stock bajo, nuevos productos |

#### Eventos Publicados:

```javascript
// Precio Actualizado
{
  event: "producto.precio_cambiado",
  data: {
    productoId: "PROD-001",
    sku: "LAP-DEL-001",
    precioAnterior: 999.99,
    precioNuevo: 899.99,
    clientesInteresados: ["12345", "67890"]
  }
}

// Stock Bajo
{
  event: "producto.stock_bajo",
  data: {
    productoId: "PROD-001",
    stockActual: 5,
    umbralAlerta: 10,
    adminNotificar: ["admin@tienda.com"]
  }
}
```

---

### 4️⃣ msnotificaciones (Notificaciones) - 🔔

**Rol:** Central de comunicaciones

#### Interacciones:

| Con | Tipo | Descripción |
|-----|------|-------------|
| **msclientes** | Subscribe | Escucha eventos de registro, recuperación de contraseña |
| **msproducto** | Subscribe | Escucha eventos de precio, stock, nuevos productos |
| **msseguridad** | Subscribe | Escucha eventos de seguridad (login fallido, cambio de contraseña) |

#### Tipos de Notificaciones:

| Evento Origen | Canal | Template |
|---------------|-------|----------|
| `cliente.registrado` | Email | Bienvenida + confirmación email |
| `cliente.password_reset` | Email | Link de recuperación |
| `producto.precio_cambiado` | Email/Push | Alerta de oferta |
| `producto.stock_bajo` | Email | Notificación a administradores |
| `seguridad.login_sospechoso` | Email/SMS | Alerta de seguridad |

---

## Flujos de Negocio Completos

### 📝 Flujo 1: Registro de Nuevo Cliente

```
┌─────────┐     ┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Cliente │────►│ API Gateway │────►│  msclientes  │────►│  msseguridad    │
└─────────┘     └─────────────┘     └──────────────┘     └─────────────────┘
   │                                         │                      │
   │                                         │  1. Crear usuario    │
   │                                         │─────────────────────►│
   │                                         │                      │
   │                                         │  2. Hash password    │
   │                                         │◄─────────────────────│
   │                                         │                      │
   │                                         │  3. Guardar cliente  │
   │                                         │────┐                 │
   │                                         │    │                 │
   │                                         │◄───┘                 │
   │                                         │                      │
   │  4. Publicar evento                     │                      │
   │  "cliente.registrado"                   │                      │
   │────────────────────────────────────────►│                      │
   │                                         │                      │
   │         ┌───────────────────────────────┘                      │
   │         │                                                      │
   │         ▼                                                      │
   │  ┌─────────────────┐     ┌──────────────┐                     │
   │  │ msnotificaciones│────►│   SendGrid   │                     │
   │  │                 │     │   (Email)    │                     │
   │  │ • Lee evento    │     │              │                     │
   │  │ • Aplica        │     │              │                     │
   │  │   template      │     │              │                     │
   │  │ • Envía email   │     │              │                     │
   │  └─────────────────┘     └──────────────┘                     │
   │                                                               │
   │  5. Enviar token JWT                                          │
   │◄──────────────────────────────────────────────────────────────┤
   │                                                               │
```

**Descripción paso a paso:**
1. Cliente envía datos de registro
2. `msclientes` solicita a `msseguridad` hashear la contraseña
3. `msclientes` guarda el cliente en su BD
4. `msclientes` publica evento `cliente.registrado`
5. `msnotificaciones` recibe el evento y envía email de bienvenida
6. Cliente recibe JWT token para autenticación futura

---

### 🛒 Flujo 2: Cliente Consulta Productos

```
┌─────────┐     ┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Cliente │────►│ API Gateway │────►│  msseguridad │────►│  msproducto     │
└─────────┘     └─────────────┘     └──────────────┘     └─────────────────┘
   │                                         │                      │
   │  GET /api/products                      │                      │
   │  Headers: Authorization: Bearer <JWT>   │                      │
   │────────────────────────────────────────►│                      │
   │                                         │  1. Validar token    │
   │                                         │─────────────────────►│
   │                                         │                      │
   │                                         │  2. Token válido     │
   │                                         │◄─────────────────────│
   │                                         │                      │
   │                                         │  3. Extraer userId   │
   │                                         │─────────────────────►│
   │                                         │                      │
   │                                         │                      │
   │  4. Consultar productos                 │                      │
   │     con filtros y                     │                      │
   │     preferencias                      │                      │
   │     de usuario                        │                      │
   │                                         │◄─────────────────────│
   │                                         │                      │
   │                                         │  5. Registrar        │
   │                                         │     visualización    │
   │                                         │     (histórico)      │
   │                                         │────┐                 │
   │                                         │    │                 │
   │                                         │◄───┘                 │
   │                                         │                      │
   │  6. Retornar productos                  │                      │
   │◄──────────────────────────────────────────────────────────────┤
   │                                         │                      │
```

---

### 💰 Flujo 3: Alerta de Precio (Producto + Notificación)

```
┌──────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Admin      │────►│   msproducto    │────►│ msnotificaciones│
│  (Actualiza  │     │                 │     │                 │
│   precio)    │     │  1. Validar     │     │  4. Buscar      │
└──────────────┘     │     permisos    │     │     suscriptores│
                     │        │        │     │        │        │
                     │◄───────┘        │     │◄───────┘        │
                     │                 │     │                 │
                     │  2. Actualizar  │     │  5. Enviar      │
                     │     precio      │     │     notificaciones
                     │        │        │     │        │        │
                     │◄───────┘        │     │◄───────┘        │
                     │                 │     │                 │
                     │  3. Publicar    │     │  ┌───────────┐  │
                     │     evento      │────►│  │  Email    │  │
                     │  "precio_cambiado"     │  │  Push     │  │
                     │                 │     │  │  SMS      │  │
                     │                 │     │  └───────────┘  │
                     │                 │     │                 │
                     │                 │     └─────────────────┘
                     │                 │              │
                     │                 │              ▼
                     │                 │       ┌──────────────┐
                     │                 │       │  Clientes    │
                     │                 │       │  (Alertados) │
                     │                 │       └──────────────┘
```

---

## Patrones de Comunicación

### Síncrono (HTTP/REST)

Usado cuando se necesita respuesta inmediata:

```javascript
// msclientes → msseguridad
const response = await axios.post('https://msseguridad/auth/validate', {
  token: jwtToken
});

// msproducto → msseguridad
const hasPermission = await axios.get(
  `https://msseguridad/permissions/check?user=${userId}&resource=product&action=update`
);
```

**Cuándo usar:**
- ✅ Validación de tokens
- ✅ Verificación de permisos
- ✅ Consultas que requieren respuesta inmediata

**Riesgos:**
- ⚠️ Acoplamiento temporal
- ⚠️ Cascada de fallos (circuit breaker necesario)

---

### Asíncrono (Event-Driven)

Usado para operaciones que no requieren respuesta inmediata:

```javascript
// msclientes publica evento
const eventBus = require('./eventBus');

eventBus.publish('cliente.registrado', {
  clienteId: newClient.id,
  email: newClient.email,
  nombre: newClient.nombre
});

// msnotificaciones suscribe y procesa
eventBus.subscribe('cliente.registrado', async (event) => {
  await emailService.sendWelcome(event.data.email, event.data.nombre);
});
```

**Cuándo usar:**
- ✅ Notificaciones
- ✅ Logs y auditoría
- ✅ Procesos batch
- ✅ Desacoplamiento de servicios

**Beneficios:**
- ✅ Resiliencia (servicio caído no afecta al emisor)
- ✅ Escalabilidad independiente
- ✅ Desacoplamiento

---

## API Gateway - Punto de Entrada Único

### Responsabilidades:

```
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. AUTHENTICATION                                           │
│     └── Validar JWT con msseguridad                         │
│                                                              │
│  2. ROUTING                                                  │
│     ├── /api/auth/*    → msseguridad                        │
│     ├── /api/users/*   → msclientes                         │
│     ├── /api/products/* → msproducto                        │
│     └── /api/notify/*  → msnotificaciones                   │
│                                                              │
│  3. RATE LIMITING                                            │
│     └── 100 req/min por IP                                  │
│                                                              │
│  4. REQUEST/RESPONSE TRANSFORMATION                          │
│     └── Agregar headers de correlación                      │
│                                                              │
│  5. CACHING                                                  │
│     └── Cachear respuestas de productos (5 min)             │
│                                                              │
│  6. SSL/TLS TERMINATION                                      │
│     └── Certificados HTTPS                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Base de Datos por Microservicio

| Microservicio | Base de Datos | Tablas Principales |
|---------------|---------------|-------------------|
| **msseguridad** | MySQL | users, roles, permissions, sessions, audit_logs |
| **msclientes** | MySQL | clients, profiles, preferences, addresses, history |
| **msproducto** | MySQL | categories, products, variants, attributes, images |
| **msnotificaciones** | MySQL | templates, logs, subscriptions, queues |

**Principio:** Cada microservicio tiene su propia base de datos (Database per Service)

---

## Seguridad entre Microservicios

### 1. Service-to-Service Authentication

```javascript
// Cada microservicio tiene un "service account"
const serviceToken = generateServiceToken({
  service: 'msproducto',
  permissions: ['read:products', 'write:products']
});

// Llamada entre servicios
const response = await axios.get('https://msclientes/internal/clients', {
  headers: {
    'X-Service-Token': serviceToken
  }
});
```

### 2. Zero Trust Network

- Ningún servicio confía ciegamente en otro
- Todos los endpoints internos requieren autenticación
- Logs de auditoría de todas las llamadas

---

## Monitoreo y Observabilidad

### Métricas a Rastrear:

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILIDAD                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  📊 MÉTRICAS (Prometheus/Grafana)                           │
│     ├── Latencia de requests por endpoint                   │
│     ├── Tasa de errores (4xx, 5xx)                          │
│     ├── Uso de recursos (CPU, memoria)                      │
│     └── Conexiones de base de datos                         │
│                                                              │
│  📝 LOGS (ELK Stack / CloudWatch)                           │
│     ├── Request/Response tracing                            │
│     ├── Eventos de negocio                                  │
│     └── Errores y excepciones                               │
│                                                              │
│  🔗 DISTRIBUTED TRACING (Jaeger/Zipkin)                     │
│     └── Seguimiento de requests entre microservicios        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Resumen de Relaciones

```
                    ┌─────────────────┐
                    │   API GATEWAY   │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │ msseguridad │◄──►│ msclientes  │◄──►│ msproducto  │
   │   (Auth)    │    │  (Users)    │    │  (Catalog)  │
   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ msnotificaciones│
                    │ (Notifications) │
                    └─────────────────┘

Leyenda:
───► Llamada síncrona (HTTP)
───► Comunicación asíncrona (Eventos)
◄──► Bidireccional
```

---

**Documento generado:** Arquitectura de Microservicios
**Sistemas documentados:** msseguridad, msclientes, msproducto, msnotificaciones
**Patrón:** API Gateway + Event-Driven + Database per Service
