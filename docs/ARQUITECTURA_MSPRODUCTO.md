# Arquitectura de Microservicio de Productos Dinámico

## Índice
1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [El Problema](#el-problema)
3. [Opciones de Arquitectura Analizadas](#opciones-de-arquitectura-analizadas)
4. [Arquitectura Recomendada: Modelo Híbrido](#arquitectura-recomendada-modelo-híbrido)
5. [Diseño de Base de Datos](#diseño-de-base-de-datos)
6. [Arquitectura del Microservicio](#arquitectura-del-microservicio)
7. [Implementación Node.js](#implementación-nodejs)
8. [API REST](#api-rest)
9. [Consideraciones de Performance](#consideraciones-de-performance)
10. [Mejores Prácticas](#mejores-prácticas)
11. [Conclusión](#conclusión)

---

## Resumen Ejecutivo

Este documento presenta una arquitectura de microservicio de productos **dinámica, configurable y mantenible** utilizando **Node.js y MariaDB**. La solución permite almacenar diferentes tipos de productos con atributos variables sin requerir cambios en la estructura de la base de datos.

### Características Clave
- ✅ Esquema flexible sin `ALTER TABLE`
- ✅ Atributos dinámicos por tipo de producto
- ✅ Validación de datos mediante metadatos
- ✅ Búsquedas indexadas en atributos frecuentes
- ✅ API RESTful con operaciones CRUD completas
- ✅ Soporte para variantes de productos
- ✅ Categorización jerárquica

---

## El Problema

### Situación Actual
Los sistemas de catálogo de productos tradicionales utilizan esquemas rígidos donde cada atributo es una columna:

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2),
    color VARCHAR(50),      -- Solo aplica a algunos productos
    size VARCHAR(20),       -- Solo ropa
    ram_gb INT,             -- Solo electrónica
    screen_size DECIMAL,    -- Solo TVs/monitores
    engine_cc INT,          -- Solo vehículos
    -- Más columnas NULL...
);
```

### Problemas Identificados
1. **Tablas escasas (Sparse Tables)**: Muchas columnas NULL
2. **Migraciones constantes**: Nuevo atributo = ALTER TABLE
3. **API frágil**: Cambios en BD rompen contratos de API
4. **Difícil mantenimiento**: Lógica de negocio dispersa
5. **Sin validación flexible**: No se pueden definir reglas por tipo

### Requerimientos del Negocio
- Productos de diferentes categorías con atributos únicos
- Campos configurables sin intervención de desarrolladores
- Búsquedas rápidas por atributos comunes (precio, marca, categoría)
- Validación de datos según el tipo de producto
- Escalabilidad para miles de atributos diferentes

---

## Opciones de Arquitectura Analizadas

### Opción 1: EAV (Entity-Attribute-Value)

```sql
-- Entidades
CREATE TABLE entities (
    id INT PRIMARY KEY AUTO_INCREMENT,
    entity_type VARCHAR(50),
    name VARCHAR(255)
);

-- Atributos
CREATE TABLE attributes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    data_type ENUM('text', 'number', 'date', 'boolean')
);

-- Valores
CREATE TABLE entity_attribute_values (
    entity_id INT,
    attribute_id INT,
    value_text VARCHAR(1000),
    value_number DECIMAL(18,4),
    value_date DATE,
    value_bool TINYINT(1),
    PRIMARY KEY (entity_id, attribute_id)
);
```

#### Pros
- Máxima flexibilidad
- Sin ALTER TABLE
- Atributos tipados

#### Contras
- Queries complejas con múltiples JOINs
- Pérdida de integridad referencial
- Difícil de indexar eficientemente
- Queries lentos con muchos atributos
- Complejidad en el código

#### Veredicto
❌ **No recomendado** para catálogos de productos con búsquedas frecuentes.

---

### Opción 2: JSON Completo (Document Store)

```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    sku VARCHAR(100),
    data JSON
);

-- Ejemplo de inserción
INSERT INTO products (name, sku, data) VALUES (
    'Laptop Dell XPS 15',
    'DELL-XPS15-001',
    '{
        "brand": "Dell",
        "category": "laptops",
        "price": 1299.99,
        "specs": {
            "cpu": "Intel i7",
            "ram": "16GB",
            "storage": "512GB SSD"
        }
    }'
);
```

#### Pros
- Máxima flexibilidad
- Sin schema rígido
- Fácil de implementar

#### Contras
- Búsquedas en JSON son lentas sin índices
- No se pueden indexar fácilmente campos anidados
- Validación compleja en aplicación
- Difícil hacer agregaciones

#### Veredicto
❌ **No recomendado** como única solución. Útil como complemento.

---

### Opción 3: Modelo Híbrido (Recomendado) ⭐

Combina lo mejor de ambos mundos:
- **Columnas tradicionales** para atributos comunes e indexables
- **JSON** para atributos dinámicos específicos
- **Virtual Columns** para indexar atributos JSON frecuentemente buscados
- **Metadata** para validación y configuración

```sql
CREATE TABLE products (
    -- Campos fijos e indexables
    id INT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    category_id INT NOT NULL,
    brand VARCHAR(100),
    price DECIMAL(12,2) NOT NULL,
    stock INT DEFAULT 0,
    status ENUM('active', 'inactive', 'discontinued') DEFAULT 'active',
    
    -- Atributos dinámicos en JSON
    attributes JSON,
    
    -- Virtual columns para indexación (ejemplos)
    color VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.color'))) VIRTUAL,
    weight_kg DECIMAL(8,3) AS (JSON_EXTRACT(attributes, '$.weight_kg')) VIRTUAL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_category (category_id),
    INDEX idx_brand (brand),
    INDEX idx_price (price),
    INDEX idx_color (color),  -- Index en virtual column
    INDEX idx_weight (weight_kg)
);
```

#### Pros
- ✅ Rendimiento de SQL tradicional para campos comunes
- ✅ Flexibilidad de JSON para atributos dinámicos
- ✅ Indexación posible con Virtual Columns
- ✅ Validación mediante metadatos
- ✅ Sin ALTER TABLE para nuevos atributos
- ✅ Queries simples y rápidas

#### Contras
- Planificación inicial requerida para identificar campos indexables
- Virtual Columns consumen recursos en queries

#### Veredicto
✅ **RECOMENDADO** - Balance óptimo entre flexibilidad y performance.

---

## Arquitectura Recomendada: Modelo Híbrido

### Principios de Diseño

1. **Separación de Responsabilidades**: Atributos comunes vs. específicos
2. **Metadata-Driven**: Configuración de atributos en tablas
3. **Indexación Inteligente**: Virtual columns para campos de búsqueda frecuente
4. **Validación Declarativa**: Reglas de negocio en metadatos
5. **API Unificada**: Interfaz consistente independiente del esquema

### Diagrama de Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENTE (Web/Mobile)                      │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTP/REST
┌─────────────────────────▼───────────────────────────────────┐
│              API GATEWAY / LOAD BALANCER                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│           MICROSERVICIO: ms-producto                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Controllers (REST API)                               │  │
│  │  - ProductController                                  │  │
│  │  - CategoryController                                 │  │
│  │  - AttributeController                                │  │
│  └───────────────────┬───────────────────────────────────┘  │
│                      │                                       │
│  ┌───────────────────▼───────────────────────────────────┐  │
│  │  Services (Business Logic)                            │  │
│  │  - ProductService                                     │  │
│  │  - ValidationService                                  │  │
│  │  - SearchService                                      │  │
│  └───────────────────┬───────────────────────────────────┘  │
│                      │                                       │
│  ┌───────────────────▼───────────────────────────────────┐  │
│  │  Repositories (Data Access)                           │  │
│  │  - ProductRepository                                  │  │
│  │  - CategoryRepository                                 │  │
│  └───────────────────┬───────────────────────────────────┘  │
│                      │                                       │
└──────────────────────┼───────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────┐
│              MARIADB (Base de Datos)                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐          │
│  │   products   │ │  categories  │ │  attributes  │          │
│  │  (Híbrido)   │ │  (Jerarquía) │ │  (Metadata)  │          │
│  └──────────────┘ └──────────────┘ └──────────────┘          │
│  ┌──────────────┐ ┌──────────────┐                           │
│  │   variants   │ │product_images│                           │
│  └──────────────┘ └──────────────┘                           │
└──────────────────────────────────────────────────────────────┘
```

---

## Diseño de Base de Datos

### Schema Completo MariaDB

```sql
-- ============================================
-- BASE DE DATOS: msproducto
-- MOTOR: MariaDB 10.6+
-- ============================================

CREATE DATABASE IF NOT EXISTS msproducto 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;

USE msproducto;

-- ============================================
-- 1. TABLA: CATEGORÍAS (Jerárquica)
-- ============================================
CREATE TABLE categories (
    id INT PRIMARY KEY AUTO_INCREMENT,
    parent_id INT NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    sort_order INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL,
    INDEX idx_parent (parent_id),
    INDEX idx_slug (slug),
    INDEX idx_active (is_active)
) ENGINE=InnoDB;

-- ============================================
-- 2. TABLA: DEFINICIÓN DE ATRIBUTOS (Metadata)
-- ============================================
CREATE TABLE attribute_definitions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    category_id INT NULL,  -- NULL = atributo global
    name VARCHAR(100) NOT NULL,
    code VARCHAR(50) NOT NULL,  -- Identificador en JSON
    description TEXT,
    data_type ENUM('string', 'number', 'boolean', 'date', 'enum', 'multiselect') NOT NULL,
    is_required BOOLEAN DEFAULT FALSE,
    is_filterable BOOLEAN DEFAULT FALSE,  -- ¿Se puede usar en filtros?
    is_searchable BOOLEAN DEFAULT FALSE,  -- ¿Se puede buscar?
    default_value JSON,
    validation_rules JSON,  -- {min, max, pattern, etc}
    options JSON,  -- Para enum/multiselect: ['option1', 'option2']
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE,
    INDEX idx_category (category_id),
    INDEX idx_code (code),
    UNIQUE KEY uk_category_code (category_id, code)
) ENGINE=InnoDB;

-- ============================================
-- 3. TABLA: PRODUCTOS (Modelo Híbrido)
-- ============================================
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    category_id INT NOT NULL,
    brand VARCHAR(100),
    
    -- Precios (fijos para indexación)
    price DECIMAL(12,2) NOT NULL,
    compare_at_price DECIMAL(12,2),
    cost_price DECIMAL(12,2),
    currency VARCHAR(3) DEFAULT 'USD',
    
    -- Inventario
    stock INT DEFAULT 0,
    stock_alert_level INT DEFAULT 10,
    track_inventory BOOLEAN DEFAULT TRUE,
    
    -- Atributos dinámicos (JSON)
    attributes JSON CHECK (JSON_VALID(attributes)),
    
    -- SEO y contenido
    short_description TEXT,
    description LONGTEXT,
    meta_title VARCHAR(255),
    meta_description VARCHAR(500),
    
    -- Estado
    status ENUM('draft', 'active', 'inactive', 'discontinued') DEFAULT 'draft',
    is_featured BOOLEAN DEFAULT FALSE,
    
    -- Métricas
    view_count INT DEFAULT 0,
    sales_count INT DEFAULT 0,
    rating DECIMAL(2,1) DEFAULT 5.0,
    review_count INT DEFAULT 0,
    
    -- VIRTUAL COLUMNS para indexación (ejemplos comunes)
    -- Estos se crean dinámicamente según attribute_definitions.is_filterable
    color VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.color'))) VIRTUAL,
    size VARCHAR(20) AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.size'))) VIRTUAL,
    material VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.material'))) VIRTUAL,
    weight_kg DECIMAL(8,3) AS (JSON_EXTRACT(attributes, '$.weight_kg')) VIRTUAL,
    dimensions JSON AS (JSON_EXTRACT(attributes, '$.dimensions')) VIRTUAL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Foreign Keys
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT,
    
    -- Índices
    INDEX idx_category (category_id),
    INDEX idx_brand (brand),
    INDEX idx_price (price),
    INDEX idx_status (status),
    INDEX idx_featured (is_featured),
    INDEX idx_created (created_at),
    
    -- Índices en virtual columns (para búsquedas frecuentes)
    INDEX idx_color (color),
    INDEX idx_size (size),
    INDEX idx_material (material)
    
) ENGINE=InnoDB;

-- ============================================
-- 4. TABLA: VARIANTES DE PRODUCTO
-- ============================================
CREATE TABLE product_variants (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    sku VARCHAR(100) UNIQUE NOT NULL,
    
    -- Atributos que definen la variante (ej: {"color": "red", "size": "M"})
    variant_attributes JSON NOT NULL,
    
    -- Precio específico de la variante
    price DECIMAL(12,2),
    compare_at_price DECIMAL(12,2),
    
    -- Inventario específico
    stock INT DEFAULT 0,
    
    -- Imagen específica de la variante
    image_url VARCHAR(500),
    
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_product (product_id),
    INDEX idx_active (is_active)
) ENGINE=InnoDB;

-- ============================================
-- 5. TABLA: IMÁGENES DE PRODUCTO
-- ============================================
CREATE TABLE product_images (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    variant_id INT NULL,
    image_url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(255),
    is_primary BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (variant_id) REFERENCES product_variants(id) ON DELETE SET NULL,
    INDEX idx_product (product_id)
) ENGINE=InnoDB;

-- ============================================
-- 6. TABLA: RELACIONES DE PRODUCTOS
-- ============================================
CREATE TABLE product_relations (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    related_product_id INT NOT NULL,
    relation_type ENUM('related', 'upsell', 'cross_sell', 'bundle') DEFAULT 'related',
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (related_product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_relation (product_id, related_product_id, relation_type)
) ENGINE=InnoDB;

-- ============================================
-- VISTAS ÚTILES
-- ============================================

-- Vista de productos con información de categoría
CREATE VIEW v_products_full AS
SELECT 
    p.*,
    c.name AS category_name,
    c.slug AS category_slug,
    COALESCE(
        (SELECT image_url FROM product_images 
         WHERE product_id = p.id AND is_primary = TRUE LIMIT 1),
        (SELECT image_url FROM product_images 
         WHERE product_id = p.id ORDER BY sort_order LIMIT 1)
    ) AS primary_image
FROM products p
LEFT JOIN categories c ON p.category_id = c.id;

-- ============================================
-- TRIGGERS
-- ============================================

-- Trigger para validar JSON en productos
DELIMITER //

CREATE TRIGGER trg_validate_product_attributes
BEFORE INSERT ON products
FOR EACH ROW
BEGIN
    IF NEW.attributes IS NOT NULL AND NOT JSON_VALID(NEW.attributes) THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Invalid JSON in attributes field';
    END IF;
END//

CREATE TRIGGER trg_validate_product_attributes_update
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    IF NEW.attributes IS NOT NULL AND NOT JSON_VALID(NEW.attributes) THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Invalid JSON in attributes field';
    END IF;
END//

DELIMITER ;
```

### Ejemplos de Inserción

```sql
-- 1. Crear categorías
INSERT INTO categories (name, slug, description) VALUES
('Electrónica', 'electronica', 'Productos electrónicos y tecnología'),
('Laptops', 'laptops', 'Computadoras portátiles', 1),
('Smartphones', 'smartphones', 'Teléfonos inteligentes', 1),
('Ropa', 'ropa', 'Vestimenta y accesorios'),
('Camisetas', 'camisetas', 'Camisetas de todos los estilos', 4);

-- 2. Definir atributos por categoría
INSERT INTO attribute_definitions (category_id, name, code, data_type, is_required, is_filterable, validation_rules) VALUES
-- Atributos para Laptops
(2, 'Procesador', 'cpu', 'string', TRUE, TRUE, '{"maxLength": 100}'),
(2, 'Memoria RAM', 'ram_gb', 'number', TRUE, TRUE, '{"min": 1, "max": 128}'),
(2, 'Almacenamiento', 'storage', 'string', TRUE, FALSE, NULL),
(2, 'Pantalla', 'screen_size', 'number', TRUE, TRUE, '{"min": 10, "max": 18}'),
(2, 'Tarjeta Gráfica', 'gpu', 'string', FALSE, TRUE, NULL),

-- Atributos para Camisetas  
(5, 'Color', 'color', 'string', TRUE, TRUE, NULL),
(5, 'Talla', 'size', 'enum', TRUE, TRUE, '{"options": ["XS", "S", "M", "L", "XL", "XXL"]}'),
(5, 'Material', 'material', 'string', FALSE, TRUE, NULL),
(5, 'Peso', 'weight_kg', 'number', FALSE, FALSE, '{"min": 0.1, "max": 2}');

-- 3. Insertar productos con atributos dinámicos
INSERT INTO products (sku, name, slug, category_id, brand, price, stock, attributes, status) VALUES
-- Laptop
('LAP-DEL-XPS15-001', 
 'Dell XPS 15 9530', 
 'dell-xps-15-9530',
 2, 
 'Dell', 
 1899.99, 
 25,
 '{
    "cpu": "Intel Core i9-13900H",
    "ram_gb": 32,
    "storage": "1TB NVMe SSD",
    "screen_size": 15.6,
    "gpu": "NVIDIA RTX 4060",
    "color": "Silver",
    "weight_kg": 1.86,
    "ports": ["USB-C", "Thunderbolt 4", "SD Card"],
    "warranty_years": 2
 }',
 'active'
),

-- Smartphone
('PHN-IPH-15PRO-001',
 'iPhone 15 Pro',
 'iphone-15-pro',
 3,
 'Apple',
 999.00,
 150,
 '{
    "cpu": "A17 Pro",
    "ram_gb": 8,
    "storage": "256GB",
    "screen_size": 6.1,
    "color": "Natural Titanium",
    "weight_kg": 0.187,
    "camera_mp": 48,
    "5g": true,
    "waterproof": "IP68"
 }',
 'active'
),

-- Camiseta
('SHI-NIK-DRY-001',
 'Nike Dri-FIT Tee',
 'nike-dri-fit-tee',
 5,
 'Nike',
 35.00,
 500,
 '{
    "color": "Black",
    "size": "M",
    "material": "100% Polyester",
    "weight_kg": 0.15,
    "care_instructions": ["Machine wash cold", "Tumble dry low"]
 }',
 'active'
);
```

---

## Arquitectura del Microservicio

### Estructura de Carpetas

```
msproducto/
├── src/
│   ├── config/
│   │   ├── database.js          # Configuración de MariaDB
│   │   ├── app.js               # Configuración de Express
│   │   └── constants.js         # Constantes de la aplicación
│   │
│   ├── controllers/
│   │   ├── productController.js    # CRUD de productos
│   │   ├── categoryController.js   # Gestión de categorías
│   │   ├── attributeController.js  # Definición de atributos
│   │   └── searchController.js     # Búsquedas avanzadas
│   │
│   ├── services/
│   │   ├── productService.js       # Lógica de negocio de productos
│   │   ├── validationService.js    # Validación de atributos
│   │   ├── searchService.js        # Lógica de búsqueda
│   │   └── attributeService.js     # Gestión de atributos dinámicos
│   │
│   ├── repositories/
│   │   ├── productRepository.js    # Acceso a datos de productos
│   │   ├── categoryRepository.js   # Acceso a datos de categorías
│   │   └── attributeRepository.js  # Acceso a metadatos
│   │
│   ├── models/
│   │   ├── Product.js              # Modelo de Producto
│   │   ├── Category.js             # Modelo de Categoría
│   │   └── AttributeDefinition.js  # Modelo de definición de atributo
│   │
│   ├── middleware/
│   │   ├── errorHandler.js         # Manejo de errores global
│   │   ├── validateRequest.js      # Validación de requests
│   │   ├── sanitizeInput.js        # Sanitización de inputs
│   │   └── rateLimiter.js          # Rate limiting
│   │
│   ├── utils/
│   │   ├── jsonHelper.js           # Helpers para manejo de JSON
│   │   ├── queryBuilder.js         # Constructor de queries dinámicas
│   │   ├── logger.js               # Logger configurado
│   │   └── validators.js           # Validaciones genéricas
│   │
│   ├── routes/
│   │   ├── index.js                # Router principal
│   │   ├── products.js             # Rutas de productos
│   │   ├── categories.js           # Rutas de categorías
│   │   └── attributes.js           # Rutas de atributos
│   │
│   └── app.js                      # Punto de entrada
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│
├── docs/
│   └── ARQUITECTURA.md             # Este documento
│
├── migrations/                     # Scripts SQL de migración
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── package.json
└── README.md
```

---

## Implementación Node.js

### 1. Configuración de Base de Datos

```javascript
// src/config/database.js
const mariadb = require('mariadb');

const pool = mariadb.createPool({
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 3306,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    connectionLimit: 20,
    acquireTimeout: 10000,
    idleTimeout: 600000,
    
    // Configuraciones importantes para JSON
    checkDuplicate: false,
    metaAsArray: false,
    
    // Conversión automática de JSON
    typeCast: function(field, next) {
        if (field.type === 'JSON') {
            const value = field.string();
            return value ? JSON.parse(value) : null;
        }
        return next();
    }
});

// Helper para queries
const query = async (sql, params) => {
    let conn;
    try {
        conn = await pool.getConnection();
        const result = await conn.query(sql, params);
        return result;
    } catch (err) {
        console.error('Database error:', err);
        throw err;
    } finally {
        if (conn) conn.release();
    }
};

// Transaction helper
const transaction = async (callback) => {
    let conn;
    try {
        conn = await pool.getConnection();
        await conn.beginTransaction();
        const result = await callback(conn);
        await conn.commit();
        return result;
    } catch (err) {
        if (conn) await conn.rollback();
        throw err;
    } finally {
        if (conn) conn.release();
    }
};

module.exports = { pool, query, transaction };
```

### 2. Modelo de Producto

```javascript
// src/models/Product.js
class Product {
    constructor(data) {
        this.id = data.id;
        this.sku = data.sku;
        this.name = data.name;
        this.slug = data.slug;
        this.categoryId = data.category_id;
        this.brand = data.brand;
        this.price = parseFloat(data.price);
        this.compareAtPrice = data.compare_at_price ? parseFloat(data.compare_at_price) : null;
        this.stock = data.stock;
        
        // Atributos dinámicos (JSON)
        this.attributes = typeof data.attributes === 'string' 
            ? JSON.parse(data.attributes) 
            : data.attributes || {};
        
        // Campos calculados de virtual columns
        this.color = data.color;
        this.size = data.size;
        this.material = data.material;
        this.weightKg = data.weight_kg;
        
        // SEO y contenido
        this.shortDescription = data.short_description;
        this.description = data.description;
        this.metaTitle = data.meta_title;
        this.metaDescription = data.meta_description;
        
        // Estado
        this.status = data.status;
        this.isFeatured = data.is_featured;
        
        // Timestamps
        this.createdAt = data.created_at;
        this.updatedAt = data.updated_at;
        
        // Relaciones
        this.category = null;
        this.images = [];
        this.variants = [];
    }

    // Getter para acceder a atributos dinámicos fácilmente
    getAttribute(key, defaultValue = null) {
        return this.attributes[key] ?? defaultValue;
    }

    // Setter para atributos dinámicos
    setAttribute(key, value) {
        this.attributes[key] = value;
    }

    // Serializar para respuesta API
    toJSON() {
        return {
            id: this.id,
            sku: this.sku,
            name: this.name,
            slug: this.slug,
            category: this.category,
            brand: this.brand,
            pricing: {
                price: this.price,
                compareAtPrice: this.compareAtPrice,
                currency: 'USD'
            },
            stock: this.stock,
            attributes: this.attributes,
            description: this.description,
            status: this.status,
            isFeatured: this.isFeatured,
            images: this.images,
            variants: this.variants,
            createdAt: this.createdAt,
            updatedAt: this.updatedAt
        };
    }
}

module.exports = Product;
```

### 3. Repository de Productos

```javascript
// src/repositories/productRepository.js
const { query, transaction } = require('../config/database');
const Product = require('../models/Product');

class ProductRepository {
    
    // Crear producto con atributos dinámicos
    async create(productData) {
        const sql = `
            INSERT INTO products 
            (sku, name, slug, category_id, brand, price, compare_at_price, 
             stock, attributes, short_description, description, 
             meta_title, meta_description, status, is_featured)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        `;
        
        const params = [
            productData.sku,
            productData.name,
            productData.slug,
            productData.categoryId,
            productData.brand,
            productData.price,
            productData.compareAtPrice,
            productData.stock,
            JSON.stringify(productData.attributes),
            productData.shortDescription,
            productData.description,
            productData.metaTitle,
            productData.metaDescription,
            productData.status || 'draft',
            productData.isFeatured || false
        ];
        
        const result = await query(sql, params);
        return this.findById(result.insertId);
    }

    // Buscar por ID con todas las relaciones
    async findById(id, includeRelations = true) {
        const sql = `
            SELECT p.*, c.name as category_name, c.slug as category_slug
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE p.id = ?
        `;
        
        const rows = await query(sql, [id]);
        if (rows.length === 0) return null;
        
        const product = new Product(rows[0]);
        
        if (includeRelations) {
            product.category = {
                id: product.categoryId,
                name: rows[0].category_name,
                slug: rows[0].category_slug
            };
            product.images = await this.getProductImages(id);
            product.variants = await this.getProductVariants(id);
        }
        
        return product;
    }

    // Búsqueda con filtros dinámicos
    async search(filters = {}, options = {}) {
        const {
            categoryId,
            brand,
            minPrice,
            maxPrice,
            status = 'active',
            attributes = {},  // Filtros de atributos dinámicos: { color: 'red', size: 'M' }
            searchQuery,
            isFeatured
        } = filters;
        
        const {
            page = 1,
            limit = 20,
            sortBy = 'created_at',
            sortOrder = 'DESC'
        } = options;
        
        let whereClause = ['p.status = ?'];
        let params = [status];
        
        // Filtros en columnas fijas (índices)
        if (categoryId) {
            whereClause.push('p.category_id = ?');
            params.push(categoryId);
        }
        
        if (brand) {
            whereClause.push('p.brand = ?');
            params.push(brand);
        }
        
        if (minPrice !== undefined) {
            whereClause.push('p.price >= ?');
            params.push(minPrice);
        }
        
        if (maxPrice !== undefined) {
            whereClause.push('p.price <= ?');
            params.push(maxPrice);
        }
        
        if (isFeatured !== undefined) {
            whereClause.push('p.is_featured = ?');
            params.push(isFeatured);
        }
        
        // Filtros en virtual columns (índices)
        if (attributes.color) {
            whereClause.push('p.color = ?');
            params.push(attributes.color);
        }
        
        if (attributes.size) {
            whereClause.push('p.size = ?');
            params.push(attributes.size);
        }
        
        // Búsqueda de texto
        if (searchQuery) {
            whereClause.push('(p.name LIKE ? OR p.description LIKE ?)');
            const likeQuery = `%${searchQuery}%`;
            params.push(likeQuery, likeQuery);
        }
        
        // Construir query
        const whereString = whereClause.join(' AND ');
        const offset = (page - 1) * limit;
        
        // Query principal
        const sql = `
            SELECT p.*, c.name as category_name, c.slug as category_slug
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE ${whereString}
            ORDER BY p.${sortBy} ${sortOrder}
            LIMIT ? OFFSET ?
        `;
        
        params.push(limit, offset);
        
        const rows = await query(sql, params);
        const products = rows.map(row => new Product(row));
        
        // Contar total
        const countSql = `
            SELECT COUNT(*) as total 
            FROM products p
            WHERE ${whereString}
        `;
        const countParams = params.slice(0, -2); // Remover limit y offset
        const countResult = await query(countSql, countParams);
        
        return {
            data: products,
            pagination: {
                page: parseInt(page),
                limit: parseInt(limit),
                total: countResult[0].total,
                totalPages: Math.ceil(countResult[0].total / limit)
            }
        };
    }

    // Búsqueda avanzada por atributos JSON (para filtros no indexados)
    async searchByJsonAttribute(attributePath, value) {
        const sql = `
            SELECT p.*, c.name as category_name
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE JSON_EXTRACT(p.attributes, ?) = ?
            AND p.status = 'active'
        `;
        
        const rows = await query(sql, [`$.${attributePath}`, value]);
        return rows.map(row => new Product(row));
    }

    // Actualizar producto
    async update(id, updateData) {
        const allowedFields = [
            'name', 'slug', 'category_id', 'brand', 'price', 
            'compare_at_price', 'stock', 'attributes', 'short_description',
            'description', 'meta_title', 'meta_description', 
            'status', 'is_featured'
        ];
        
        const updates = [];
        const params = [];
        
        for (const [key, value] of Object.entries(updateData)) {
            if (allowedFields.includes(key)) {
                updates.push(`${key} = ?`);
                params.push(key === 'attributes' ? JSON.stringify(value) : value);
            }
        }
        
        if (updates.length === 0) return null;
        
        params.push(id);
        
        const sql = `UPDATE products SET ${updates.join(', ')} WHERE id = ?`;
        await query(sql, params);
        
        return this.findById(id);
    }

    // Eliminar producto
    async delete(id) {
        return await transaction(async (conn) => {
            // Eliminar relaciones primero
            await conn.query('DELETE FROM product_images WHERE product_id = ?', [id]);
            await conn.query('DELETE FROM product_variants WHERE product_id = ?', [id]);
            await conn.query('DELETE FROM product_relations WHERE product_id = ? OR related_product_id = ?', [id, id]);
            
            // Eliminar producto
            const result = await conn.query('DELETE FROM products WHERE id = ?', [id]);
            return result.affectedRows > 0;
        });
    }

    // Obtener imágenes del producto
    async getProductImages(productId) {
        const sql = `
            SELECT * FROM product_images 
            WHERE product_id = ? 
            ORDER BY is_primary DESC, sort_order ASC
        `;
        return await query(sql, [productId]);
    }

    // Obtener variantes del producto
    async getProductVariants(productId) {
        const sql = `
            SELECT * FROM product_variants 
            WHERE product_id = ? AND is_active = TRUE
            ORDER BY sort_order ASC
        `;
        const rows = await query(sql, [productId]);
        return rows.map(row => ({
            ...row,
            variantAttributes: typeof row.variant_attributes === 'string' 
                ? JSON.parse(row.variant_attributes) 
                : row.variant_attributes
        }));
    }

    // Actualizar atributos dinámicos específicos
    async updateAttributes(id, attributes) {
        const sql = `
            UPDATE products 
            SET attributes = JSON_MERGE_PATCH(
                COALESCE(attributes, '{}'),
                ?
            )
            WHERE id = ?
        `;
        
        await query(sql, [JSON.stringify(attributes), id]);
        return this.findById(id);
    }
}

module.exports = new ProductRepository();
```

### 4. Service de Validación

```javascript
// src/services/validationService.js
const attributeRepository = require('../repositories/attributeRepository');

class ValidationService {
    
    // Validar atributos de producto según definiciones
    async validateProductAttributes(categoryId, attributes) {
        const errors = [];
        
        // Obtener definiciones de atributos para esta categoría
        const definitions = await attributeRepository.findByCategory(categoryId);
        
        for (const def of definitions) {
            const value = attributes[def.code];
            
            // Validar requerido
            if (def.is_required && (value === undefined || value === null || value === '')) {
                errors.push({
                    field: def.code,
                    message: `El atributo "${def.name}" es requerido`
                });
                continue;
            }
            
            // Si no es requerido y no hay valor, continuar
            if (!value && !def.is_required) {
                continue;
            }
            
            // Validar tipo de dato
            const typeError = this.validateDataType(value, def.data_type, def.code, def.name);
            if (typeError) {
                errors.push(typeError);
                continue;
            }
            
            // Validar reglas específicas
            if (def.validation_rules) {
                const rules = typeof def.validation_rules === 'string' 
                    ? JSON.parse(def.validation_rules) 
                    : def.validation_rules;
                
                const ruleError = this.validateRules(value, rules, def.code, def.name);
                if (ruleError) {
                    errors.push(ruleError);
                }
            }
            
            // Validar opciones (enum/multiselect)
            if (def.options && def.data_type === 'enum') {
                const options = typeof def.options === 'string' 
                    ? JSON.parse(def.options) 
                    : def.options;
                
                if (!options.includes(value)) {
                    errors.push({
                        field: def.code,
                        message: `El valor "${value}" no es válido para "${def.name}". Opciones: ${options.join(', ')}`
                    });
                }
            }
        }
        
        return {
            isValid: errors.length === 0,
            errors
        };
    }

    validateDataType(value, dataType, code, name) {
        switch (dataType) {
            case 'string':
                if (typeof value !== 'string') {
                    return {
                        field: code,
                        message: `"${name}" debe ser un texto`
                    };
                }
                break;
                
            case 'number':
                if (typeof value !== 'number' || isNaN(value)) {
                    return {
                        field: code,
                        message: `"${name}" debe ser un número`
                    };
                }
                break;
                
            case 'boolean':
                if (typeof value !== 'boolean') {
                    return {
                        field: code,
                        message: `"${name}" debe ser verdadero o falso`
                    };
                }
                break;
                
            case 'date':
                const date = new Date(value);
                if (isNaN(date.getTime())) {
                    return {
                        field: code,
                        message: `"${name}" debe ser una fecha válida`
                    };
                }
                break;
                
            case 'array':
                if (!Array.isArray(value)) {
                    return {
                        field: code,
                        message: `"${name}" debe ser una lista`
                    };
                }
                break;
        }
        
        return null;
    }

    validateRules(value, rules, code, name) {
        // Validar mínimo
        if (rules.min !== undefined) {
            if (typeof value === 'number' && value < rules.min) {
                return {
                    field: code,
                    message: `"${name}" debe ser mayor o igual a ${rules.min}`
                };
            }
            if (typeof value === 'string' && value.length < rules.min) {
                return {
                    field: code,
                    message: `"${name}" debe tener al menos ${rules.min} caracteres`
                };
            }
        }
        
        // Validar máximo
        if (rules.max !== undefined) {
            if (typeof value === 'number' && value > rules.max) {
                return {
                    field: code,
                    message: `"${name}" debe ser menor o igual a ${rules.max}`
                };
            }
            if (typeof value === 'string' && value.length > rules.max) {
                return {
                    field: code,
                    message: `"${name}" debe tener máximo ${rules.max} caracteres`
                };
            }
        }
        
        // Validar patrón regex
        if (rules.pattern) {
            const regex = new RegExp(rules.pattern);
            if (!regex.test(value)) {
                return {
                    field: code,
                    message: `"${name}" no tiene el formato válido`
                };
            }
        }
        
        return null;
    }

    // Sanitizar atributos (remover atributos no definidos)
    sanitizeAttributes(categoryId, attributes, definitions) {
        const validCodes = definitions.map(d => d.code);
        const sanitized = {};
        
        for (const code of validCodes) {
            if (attributes[code] !== undefined) {
                sanitized[code] = attributes[code];
            }
        }
        
        return sanitized;
    }
}

module.exports = new ValidationService();
```

### 5. Controller de Productos

```javascript
// src/controllers/productController.js
const productService = require('../services/productService');
const validationService = require('../services/validationService');

class ProductController {
    
    // GET /api/products
    async list(req, res, next) {
        try {
            const filters = {
                categoryId: req.query.category_id,
                brand: req.query.brand,
                minPrice: req.query.min_price,
                maxPrice: req.query.max_price,
                status: req.query.status || 'active',
                searchQuery: req.query.q,
                isFeatured: req.query.is_featured === 'true' ? true : 
                           req.query.is_featured === 'false' ? false : undefined,
                // Atributos dinámicos de query params
                attributes: {}
            };
            
            // Extraer atributos dinámicos (prefijo attr_)
            for (const [key, value] of Object.entries(req.query)) {
                if (key.startsWith('attr_')) {
                    filters.attributes[key.replace('attr_', '')] = value;
                }
            }
            
            const options = {
                page: parseInt(req.query.page) || 1,
                limit: parseInt(req.query.limit) || 20,
                sortBy: req.query.sort_by || 'created_at',
                sortOrder: req.query.sort_order?.toUpperCase() === 'ASC' ? 'ASC' : 'DESC'
            };
            
            const result = await productService.search(filters, options);
            
            res.json({
                success: true,
                data: result.data.map(p => p.toJSON()),
                pagination: result.pagination
            });
        } catch (error) {
            next(error);
        }
    }

    // GET /api/products/:id
    async getById(req, res, next) {
        try {
            const product = await productService.findById(req.params.id);
            
            if (!product) {
                return res.status(404).json({
                    success: false,
                    message: 'Producto no encontrado'
                });
            }
            
            res.json({
                success: true,
                data: product.toJSON()
            });
        } catch (error) {
            next(error);
        }
    }

    // POST /api/products
    async create(req, res, next) {
        try {
            const productData = req.body;
            
            // Validar atributos dinámicos
            if (productData.attributes && productData.categoryId) {
                const validation = await validationService.validateProductAttributes(
                    productData.categoryId,
                    productData.attributes
                );
                
                if (!validation.isValid) {
                    return res.status(400).json({
                        success: false,
                        message: 'Validación fallida',
                        errors: validation.errors
                    });
                }
            }
            
            const product = await productService.create(productData);
            
            res.status(201).json({
                success: true,
                data: product.toJSON()
            });
        } catch (error) {
            if (error.code === 'ER_DUP_ENTRY') {
                return res.status(409).json({
                    success: false,
                    message: 'SKU o slug ya existe'
                });
            }
            next(error);
        }
    }

    // PUT /api/products/:id
    async update(req, res, next) {
        try {
            const { id } = req.params;
            const updateData = req.body;
            
            // Validar atributos dinámicos si se actualizan
            if (updateData.attributes) {
                const product = await productService.findById(id, false);
                if (!product) {
                    return res.status(404).json({
                        success: false,
                        message: 'Producto no encontrado'
                    });
                }
                
                const categoryId = updateData.categoryId || product.categoryId;
                const validation = await validationService.validateProductAttributes(
                    categoryId,
                    updateData.attributes
                );
                
                if (!validation.isValid) {
                    return res.status(400).json({
                        success: false,
                        message: 'Validación fallida',
                        errors: validation.errors
                    });
                }
            }
            
            const product = await productService.update(id, updateData);
            
            res.json({
                success: true,
                data: product.toJSON()
            });
        } catch (error) {
            next(error);
        }
    }

    // DELETE /api/products/:id
    async delete(req, res, next) {
        try {
            const deleted = await productService.delete(req.params.id);
            
            if (!deleted) {
                return res.status(404).json({
                    success: false,
                    message: 'Producto no encontrado'
                });
            }
            
            res.json({
                success: true,
                message: 'Producto eliminado'
            });
        } catch (error) {
            next(error);
        }
    }

    // PATCH /api/products/:id/attributes
    async updateAttributes(req, res, next) {
        try {
            const { id } = req.params;
            const { attributes } = req.body;
            
            const product = await productService.findById(id, false);
            if (!product) {
                return res.status(404).json({
                    success: false,
                    message: 'Producto no encontrado'
                });
            }
            
            // Validar nuevos atributos
            const validation = await validationService.validateProductAttributes(
                product.categoryId,
                { ...product.attributes, ...attributes }
            );
            
            if (!validation.isValid) {
                return res.status(400).json({
                    success: false,
                    message: 'Validación fallida',
                    errors: validation.errors
                });
            }
            
            const updated = await productService.updateAttributes(id, attributes);
            
            res.json({
                success: true,
                data: updated.toJSON()
            });
        } catch (error) {
            next(error);
        }
    }
}

module.exports = new ProductController();
```

---

## API REST

### Endpoints

```yaml
# API Specification

# PRODUCTOS

## Listar productos
GET /api/products
Query Params:
  - page: número de página (default: 1)
  - limit: items por página (default: 20)
  - category_id: filtrar por categoría
  - brand: filtrar por marca
  - min_price: precio mínimo
  - max_price: precio máximo
  - status: draft|active|inactive|discontinued
  - is_featured: true|false
  - q: búsqueda de texto
  - sort_by: campo de ordenamiento
  - sort_order: ASC|DESC
  - attr_*: filtros por atributos dinámicos (ej: attr_color=red)

Response:
  {
    "success": true,
    "data": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "totalPages": 8
    }
  }

## Obtener producto por ID
GET /api/products/:id

## Crear producto
POST /api/products
Body:
  {
    "sku": "PROD-001",
    "name": "Nombre del Producto",
    "slug": "nombre-del-producto",
    "categoryId": 1,
    "brand": "Marca",
    "price": 99.99,
    "stock": 100,
    "attributes": {
      "color": "red",
      "size": "M",
      "material": "cotton"
    },
    "description": "Descripción larga...",
    "status": "active"
  }

## Actualizar producto
PUT /api/products/:id

## Actualizar atributos específicos
PATCH /api/products/:id/attributes
Body:
  {
    "attributes": {
      "color": "blue",
      "new_field": "value"
    }
  }

## Eliminar producto
DELETE /api/products/:id

# CATEGORÍAS

GET    /api/categories
GET    /api/categories/:id
POST   /api/categories
PUT    /api/categories/:id
DELETE /api/categories/:id

# ATRIBUTOS (Metadata)

GET    /api/attributes                    # Listar definiciones
GET    /api/attributes?category_id=1      # Por categoría
GET    /api/attributes/:id
POST   /api/attributes
PUT    /api/attributes/:id
DELETE /api/attributes/:id

# BÚSQUEDA AVANZADA

POST /api/products/search
Body:
  {
    "filters": [
      { "field": "price", "operator": ">=", "value": 100 },
      { "field": "brand", "operator": "=", "value": "Nike" },
      { "field": "attributes.color", "operator": "=", "value": "red" }
    ],
    "sort": { "field": "price", "order": "asc" },
    "pagination": { "page": 1, "limit": 20 }
  }
```

---

## Consideraciones de Performance

### 1. Indexación Estratégica

```sql
-- Virtual Columns para atributos frecuentemente buscados
ALTER TABLE products 
ADD COLUMN color VARCHAR(50) AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.color'))) VIRTUAL,
ADD INDEX idx_color (color);

-- Índices compuestos para filtros comunes
CREATE INDEX idx_category_price ON products(category_id, price);
CREATE INDEX idx_brand_status ON products(brand, status);
```

### 2. Query Optimization

```javascript
// ❌ MAL: JSON_EXTRACT en cada fila
const sql = `SELECT * FROM products WHERE JSON_EXTRACT(attributes, '$.color') = 'red'`;

// ✅ BIEN: Usar virtual column indexada
const sql = `SELECT * FROM products WHERE color = 'red'`;

// ✅ BIEN: Combinar filtros fijos primero
const sql = `
    SELECT * FROM products 
    WHERE category_id = ?    -- Índice primero
    AND color = ?            -- Virtual column indexada
    AND price <= ?           -- Columna indexada
`;
```

### 3. Caché de Metadatos

```javascript
// Cachear definiciones de atributos en memoria
const attributeCache = new Map();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutos

async function getCachedAttributes(categoryId) {
    const cacheKey = `attrs_${categoryId}`;
    const cached = attributeCache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
        return cached.data;
    }
    
    const definitions = await attributeRepository.findByCategory(categoryId);
    attributeCache.set(cacheKey, {
        data: definitions,
        timestamp: Date.now()
    });
    
    return definitions;
}
```

### 4. Paginación Cursor-Based (para grandes datasets)

```javascript
// Para catálogos muy grandes, usar cursor en lugar de offset
async function getProductsCursor(cursor, limit = 20) {
    const decodedCursor = cursor ? decodeCursor(cursor) : null;
    
    const sql = `
        SELECT * FROM products 
        WHERE status = 'active'
        ${decodedCursor ? 'AND (created_at, id) < (?, ?)' : ''}
        ORDER BY created_at DESC, id DESC
        LIMIT ?
    `;
    
    const params = decodedCursor 
        ? [decodedCursor.createdAt, decodedCursor.id, limit + 1]
        : [limit + 1];
    
    const rows = await query(sql, params);
    const hasMore = rows.length > limit;
    const products = rows.slice(0, limit);
    
    return {
        data: products,
        nextCursor: hasMore ? encodeCursor(products[products.length - 1]) : null
    };
}
```

---

## Mejores Prácticas

### 1. Estructura de Atributos

```javascript
// ✅ JSON plano para atributos simples
{
    "color": "red",
    "size": "M",
    "weight_kg": 1.5,
    "material": "cotton"
}

// ✅ JSON anidado para grupos lógicos
{
    "dimensions": {
        "width_cm": 100,
        "height_cm": 200,
        "depth_cm": 50
    },
    "display": {
        "size_inches": 15.6,
        "resolution": "4K",
        "panel_type": "IPS"
    }
}

// ❌ Evitar anidamiento excesivo
```

### 2. Convenciones de Nombres

```javascript
// ✅ Usar snake_case para códigos de atributo
{
    "cpu_model": "Intel i7",
    "ram_gb": 16,
    "storage_type": "SSD",
    "screen_size_inches": 15.6
}

// ✅ Prefijos por dominio
{
    "tech_cpu": "...",
    "tech_ram": "...",
    "ship_weight_kg": "...",
    "ship_dimensions": "..."
}
```

### 3. Migración de Esquema

```javascript
// Script de migración para nuevos atributos
async function migrateAddAttribute(attributeCode, defaultValue) {
    // 1. Agregar definición
    await attributeRepository.create({
        code: attributeCode,
        name: 'Nuevo Atributo',
        dataType: 'string'
    });
    
    // 2. Actualizar productos existentes con valor por defecto
    await query(`
        UPDATE products 
        SET attributes = JSON_SET(
            COALESCE(attributes, '{}'),
            '$.${attributeCode}',
            ?
        )
        WHERE JSON_CONTAINS_PATH(attributes, 'one', '$.${attributeCode}') = 0
    `, [defaultValue]);
    
    // 3. Opcional: Crear virtual column si se va a buscar frecuentemente
    await query(`
        ALTER TABLE products 
        ADD COLUMN ${attributeCode}_vc VARCHAR(100) 
        AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.${attributeCode}'))) VIRTUAL,
        ADD INDEX idx_${attributeCode} (${attributeCode}_vc)
    `);
}
```

### 4. Backup y Recovery

```bash
# Backup de esquema
mysqldump --no-data msproducto > schema_backup.sql

# Backup de datos (excluyendo JSON pesado si es necesario)
mysqldump --complete-insert --single-transaction msproducto > full_backup.sql

# Backup incremental por timestamp
mysqldump --where "updated_at > '2024-01-01'" msproducto products > incremental.sql
```

---

## Conclusión

### Arquitectura Recomendada: Modelo Híbrido

La arquitectura híbrida **JSON + Virtual Columns + Metadata** es la solución óptima para un microservicio de productos dinámico con MariaDB:

| Aspecto | Solución | Beneficio |
|---------|----------|-----------|
| **Flexibilidad** | JSON para atributos dinámicos | Sin ALTER TABLE |
| **Performance** | Virtual Columns indexadas | Búsquedas rápidas |
| **Validación** | Tablas de metadata | Integridad de datos |
| **Mantenibilidad** | Configuración declarativa | Cambios sin código |
| **Escalabilidad** | Columnas fijas + dinámicas | Balance óptimo |

### Decisiones Clave

1. **Columnas fijas** para: SKU, nombre, precio, stock, estado (siempre se buscan)
2. **Virtual Columns** para: Color, tamaño, marca (filtros frecuentes)
3. **JSON puro** para: Especificaciones técnicas, metadata adicional
4. **Tablas de metadata** para: Definición de atributos por categoría

### Próximos Pasos

1. ✅ Implementar schema de base de datos
2. ✅ Desarrollar API REST con validación
3. ⏳ Agregar sistema de caché (Redis)
4. ⏳ Implementar búsqueda full-text
5. ⏳ Agregar sistema de imágenes (S3/MinIO)
6. ⏳ Implementar eventos (Kafka/RabbitMQ)

---

## Referencias

- [MariaDB JSON Functions](https://mariadb.com/kb/en/json-functions/)
- [MariaDB Generated Columns](https://mariadb.com/kb/en/generated-columns/)
- [EAV Model Pattern](https://mysql.rjweb.org/doc.php/eav)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)

---

**Documento generado para:** msproducto  
**Tecnologías:** Node.js + MariaDB  
**Enfoque:** Arquitectura híbrida dinámica
