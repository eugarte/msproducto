# Propuesta de Implementación - msproducto

## Microservicio de Catálogo de Productos Dinámico

---

## 1. Visión General

Esta propuesta detalla la implementación del microservicio **msproducto** - un sistema de catálogo de productos con esquema dinámico que permite:

- ✅ Almacenar productos con atributos variables sin modificar la base de datos
- ✅ Definir atributos personalizados por categoría
- ✅ Realizar búsquedas rápidas por atributos indexados
- ✅ Escalar sin ALTER TABLE ni migraciones complejas

**Enfoque técnico:** Modelo Híbrido (JSON + Virtual Columns + Metadata)

---

## 2. Fases de Implementación

### 📋 FASE 1: Preparación y Configuración (Estimado: 1 día)

#### 2.1.1 Configuración de Entorno Hostinger

**Acciones:**
1. Crear base de datos MySQL en hPanel
2. Configurar variables de entorno en Hostinger
3. Configurar Node.js app en hPanel
4. Generar SSH keys para despliegue

**Configuración hPanel:**
```bash
# MySQL Database
Nombre: u123456789_msproducto
Usuario: u123456789_msproducto
Host: localhost
Charset: utf8mb4_unicode_ci

# Node.js App
Node Version: 18.x
Startup File: src/app.js
Application Port: Usar variable de entorno PORT
```

**Variables de Entorno (hPanel):**
```env
NODE_ENV=production
PORT=3000

# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=u123456789_msproducto
DB_PASSWORD=tu_password_seguro
DB_NAME=u123456789_msproducto
DB_POOL_SIZE=15
DB_CONNECTION_TIMEOUT=10000

# App
APP_NAME=msproducto
APP_VERSION=1.0.0
API_PREFIX=/api/v1

# Limits
RATE_LIMIT_WINDOW=60000
RATE_LIMIT_MAX=100
REQUEST_TIMEOUT=30000
```

#### 2.1.2 Estructura del Proyecto

```
msproducto/
├── .github/
│   └── workflows/
│       └── deploy.yml          # CI/CD para Hostinger
├── src/
│   ├── config/
│   │   ├── database.js         # Config MySQL pool
│   │   ├── app.js              # Config Express
│   │   └── constants.js        # Constantes
│   ├── controllers/
│   │   ├── productController.js
│   │   ├── categoryController.js
│   │   └── attributeController.js
│   ├── services/
│   │   ├── productService.js
│   │   ├── validationService.js
│   │   └── searchService.js
│   ├── repositories/
│   │   ├── productRepository.js
│   │   ├── categoryRepository.js
│   │   └── attributeRepository.js
│   ├── models/
│   │   ├── Product.js
│   │   └── Category.js
│   ├── middleware/
│   │   ├── errorHandler.js
│   │   ├── validateRequest.js
│   │   └── rateLimiter.js
│   ├── utils/
│   │   ├── logger.js
│   │   └── validators.js
│   ├── routes/
│   │   ├── index.js
│   │   ├── products.js
│   │   ├── categories.js
│   │   └── attributes.js
│   └── app.js                  # Entry point
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── sql/
│   ├── 01_schema.sql           # Esquema completo
│   ├── 02_indexes.sql          # Índices optimizados
│   └── 03_seed.sql             # Datos iniciales
├── .env.example
├── .gitignore
├── package.json
└── README.md
```

#### 2.1.3 Instalación de Dependencias

```bash
# Dependencias de producción
npm install express mysql2 cors helmet express-rate-limit joi dotenv morgan slugify

# Dependencias de desarrollo
npm install --save-dev nodemon jest supertest
```

**Entregables Fase 1:**
- [ ] Repositorio Git configurado
- [ ] Estructura de carpetas creada
- [ ] Dependencias instaladas
- [ ] Configuración Hostinger lista

---

### 🗄️ FASE 2: Base de Datos (Estimado: 1 día)

#### 2.2.1 Schema MySQL

**Script: `sql/01_schema.sql`**

```sql
-- ============================================
-- MICROSERVICIO: msproducto
-- HOSTINGER MYSQL SCHEMA
-- ============================================

-- Crear database (ejecutar en hPanel phpMyAdmin)
-- CREATE DATABASE IF NOT EXISTS msproducto CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- USE msproducto;

-- ============================================
-- 1. CATEGORÍAS JERÁRQUICAS
-- ============================================
CREATE TABLE IF NOT EXISTS categories (
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    parent_id INT UNSIGNED NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    sort_order INT UNSIGNED DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL,
    INDEX idx_parent (parent_id),
    INDEX idx_slug (slug),
    INDEX idx_active_sort (is_active, sort_order)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- 2. DEFINICIÓN DE ATRIBUTOS (METADATA)
-- ============================================
CREATE TABLE IF NOT EXISTS attribute_definitions (
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    category_id INT UNSIGNED NULL COMMENT 'NULL = atributo global',
    name VARCHAR(100) NOT NULL COMMENT 'Nombre visible',
    code VARCHAR(50) NOT NULL COMMENT 'Key en JSON',
    description TEXT,
    data_type ENUM('string', 'number', 'boolean', 'date', 'enum', 'multiselect') NOT NULL,
    is_required BOOLEAN DEFAULT FALSE,
    is_filterable BOOLEAN DEFAULT FALSE,
    is_searchable BOOLEAN DEFAULT FALSE,
    default_value JSON,
    validation_rules JSON COMMENT '{min, max, pattern}',
    options JSON COMMENT 'Para enum/multiselect',
    sort_order INT UNSIGNED DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE,
    INDEX idx_category (category_id),
    INDEX idx_code (code),
    UNIQUE KEY uk_category_code (category_id, code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- 3. PRODUCTOS (MODELO HÍBRIDO)
-- ============================================
CREATE TABLE IF NOT EXISTS products (
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    
    -- Identificación
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    
    -- Relaciones
    category_id INT UNSIGNED NOT NULL,
    brand VARCHAR(100),
    
    -- Precios (indexados para búsquedas)
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
    
    -- Contenido
    short_description TEXT,
    description LONGTEXT,
    meta_title VARCHAR(255),
    meta_description VARCHAR(500),
    
    -- Estado
    status ENUM('draft', 'active', 'inactive', 'discontinued') DEFAULT 'draft',
    is_featured BOOLEAN DEFAULT FALSE,
    
    -- Métricas
    view_count INT UNSIGNED DEFAULT 0,
    sales_count INT UNSIGNED DEFAULT 0,
    rating DECIMAL(2,1) DEFAULT 5.0,
    review_count INT UNSIGNED DEFAULT 0,
    
    -- VIRTUAL COLUMNS para atributos frecuentes (indexados)
    -- Se agregan dinámicamente según necesidad
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Foreign Keys
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT,
    
    -- Índices base
    INDEX idx_category_status (category_id, status),
    INDEX idx_brand (brand),
    INDEX idx_price (price),
    INDEX idx_status (status),
    INDEX idx_featured (is_featured),
    INDEX idx_created (created_at)
    
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- 4. VARIANTES DE PRODUCTO
-- ============================================
CREATE TABLE IF NOT EXISTS product_variants (
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id INT UNSIGNED NOT NULL,
    sku VARCHAR(100) UNIQUE NOT NULL,
    variant_attributes JSON NOT NULL COMMENT '{"color": "red", "size": "M"}',
    price DECIMAL(12,2),
    compare_at_price DECIMAL(12,2),
    stock INT DEFAULT 0,
    image_url VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT UNSIGNED DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_product (product_id),
    INDEX idx_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- 5. IMÁGENES DE PRODUCTO
-- ============================================
CREATE TABLE IF NOT EXISTS product_images (
    id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id INT UNSIGNED NOT NULL,
    variant_id INT UNSIGNED NULL,
    image_url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(255),
    is_primary BOOLEAN DEFAULT FALSE,
    sort_order INT UNSIGNED DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (variant_id) REFERENCES product_variants(id) ON DELETE SET NULL,
    INDEX idx_product (product_id),
    INDEX idx_primary (is_primary)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### 2.2.2 Índices Optimizados para Virtual Columns

**Script: `sql/02_indexes.sql`**

```sql
-- ============================================
-- ÍNDICES VIRTUALES PARA ATRIBUTOS COMUNES
-- Ejecutar después de crear productos y analizar patrones de búsqueda
-- ============================================

-- Agregar virtual columns para atributos frecuentemente buscados
-- Estos son ejemplos - ajustar según necesidades reales

ALTER TABLE products
ADD COLUMN attr_color VARCHAR(50) 
    AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.color'))) VIRTUAL,
ADD COLUMN attr_size VARCHAR(20) 
    AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.size'))) VIRTUAL,
ADD COLUMN attr_material VARCHAR(50) 
    AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.material'))) VIRTUAL,
ADD COLUMN attr_weight_kg DECIMAL(8,3) 
    AS (JSON_EXTRACT(attributes, '$.weight_kg')) VIRTUAL,
ADD COLUMN attr_cpu VARCHAR(100) 
    AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.cpu'))) VIRTUAL,
ADD COLUMN attr_ram_gb INT 
    AS (JSON_EXTRACT(attributes, '$.ram_gb')) VIRTUAL;

-- Crear índices en virtual columns
CREATE INDEX idx_attr_color ON products(attr_color);
CREATE INDEX idx_attr_size ON products(attr_size);
CREATE INDEX idx_attr_material ON products(attr_material);
CREATE INDEX idx_attr_weight ON products(attr_weight_kg);
CREATE INDEX idx_attr_cpu ON products(attr_cpu);
CREATE INDEX idx_attr_ram ON products(attr_ram_gb);

-- Índices compuestos para filtros comunes
CREATE INDEX idx_category_price ON products(category_id, price);
CREATE INDEX idx_brand_status ON products(brand, status);
CREATE INDEX idx_category_color ON products(category_id, attr_color);
```

#### 2.2.3 Datos de Prueba

**Script: `sql/03_seed.sql`**

```sql
-- ============================================
-- DATOS DE PRUEBA
-- ============================================

-- Categorías
INSERT INTO categories (id, name, slug, description) VALUES
(1, 'Electrónica', 'electronica', 'Productos electrónicos y tecnología'),
(2, 'Laptops', 'laptops', 'Computadoras portátiles', 1),
(3, 'Smartphones', 'smartphones', 'Teléfonos inteligentes', 1),
(4, 'Ropa', 'ropa', 'Vestimenta y accesorios'),
(5, 'Camisetas', 'camisetas', 'Camisetas de todos los estilos', 4)
ON DUPLICATE KEY UPDATE name=name;

-- Definición de atributos
INSERT INTO attribute_definitions (category_id, name, code, data_type, is_required, is_filterable, validation_rules) VALUES
-- Atributos para Laptops
(2, 'Procesador', 'cpu', 'string', TRUE, TRUE, '{"maxLength": 100}'),
(2, 'Memoria RAM', 'ram_gb', 'number', TRUE, TRUE, '{"min": 1, "max": 128}'),
(2, 'Almacenamiento', 'storage', 'string', TRUE, FALSE, NULL),
(2, 'Pantalla (pulgadas)', 'screen_size', 'number', TRUE, TRUE, '{"min": 10, "max": 18}'),
(2, 'Tarjeta Gráfica', 'gpu', 'string', FALSE, TRUE, NULL),
(2, 'Color', 'color', 'string', FALSE, TRUE, NULL),
(2, 'Peso (kg)', 'weight_kg', 'number', FALSE, FALSE, '{"min": 0.1, "max": 5}'),

-- Atributos para Camisetas
(5, 'Color', 'color', 'string', TRUE, TRUE, NULL),
(5, 'Talla', 'size', 'enum', TRUE, TRUE, '{"options": ["XS", "S", "M", "L", "XL", "XXL"]}'),
(5, 'Material', 'material', 'string', FALSE, TRUE, NULL),
(5, 'Peso (kg)', 'weight_kg', 'number', FALSE, FALSE, '{"min": 0.05, "max": 1}')

ON DUPLICATE KEY UPDATE name=name;

-- Productos de prueba
INSERT INTO products (sku, name, slug, category_id, brand, price, stock, attributes, status) VALUES
('LAP-DEL-XPS15-001', 'Dell XPS 15 9530', 'dell-xps-15-9530', 2, 'Dell', 1899.99, 25,
 '{"cpu": "Intel Core i9-13900H", "ram_gb": 32, "storage": "1TB NVMe SSD", "screen_size": 15.6, "gpu": "NVIDIA RTX 4060", "color": "Silver", "weight_kg": 1.86}',
 'active'),

('LAP-MAC-PRO16-001', 'MacBook Pro 16', 'macbook-pro-16', 2, 'Apple', 2499.00, 15,
 '{"cpu": "M3 Pro", "ram_gb": 18, "storage": "512GB SSD", "screen_size": 16, "color": "Space Black", "weight_kg": 2.14}',
 'active'),

('SHI-NIK-DRY-001', 'Nike Dri-FIT Tee', 'nike-dri-fit-tee', 5, 'Nike', 35.00, 500,
 '{"color": "Black", "size": "M", "material": "100% Polyester", "weight_kg": 0.15}',
 'active'),

('SHI-ADI-COT-001', 'Adidas Cotton Tee', 'adidas-cotton-tee', 5, 'Adidas', 28.00, 350,
 '{"color": "White", "size": "L", "material": "100% Cotton", "weight_kg": 0.18}',
 'active')

ON DUPLICATE KEY UPDATE name=name;
```

**Entregables Fase 2:**
- [ ] Scripts SQL ejecutados en Hostinger
- [ ] Tablas creadas con índices
- [ ] Datos de prueba insertados
- [ ] Conexión desde Node.js verificada

---

### ⚙️ FASE 3: Core del Backend (Estimado: 2 días)

#### 2.3.1 Configuración de Base de Datos

**Archivo: `src/config/database.js`**

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 3306,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    
    // Optimizaciones para Hostinger (limitado recursos)
    connectionLimit: parseInt(process.env.DB_POOL_SIZE) || 15,
    queueLimit: 0,
    waitForConnections: true,
    
    // Timeouts
    connectTimeout: parseInt(process.env.DB_CONNECTION_TIMEOUT) || 10000,
    acquireTimeout: 10000,
    timeout: 60000,
    
    // Reconexión automática
    enableKeepAlive: true,
    keepAliveInitialDelay: 10000,
    
    // SSL para conexiones externas (si aplica)
    ssl: process.env.DB_SSL === 'true' ? {
        rejectUnauthorized: false
    } : false
});

// Eventos de pool
pool.on('connection', (connection) => {
    console.log('New DB connection established');
});

pool.on('error', (err) => {
    console.error('Unexpected DB pool error:', err);
});

// Helper para queries
const query = async (sql, params) => {
    const [rows] = await pool.execute(sql, params);
    return rows;
};

// Transaction helper
const transaction = async (callback) => {
    const connection = await pool.getConnection();
    try {
        await connection.beginTransaction();
        const result = await callback(connection);
        await connection.commit();
        return result;
    } catch (error) {
        await connection.rollback();
        throw error;
    } finally {
        connection.release();
    }
};

// Health check
const healthCheck = async () => {
    try {
        await pool.execute('SELECT 1');
        return { status: 'healthy', database: 'connected' };
    } catch (error) {
        return { status: 'unhealthy', database: 'disconnected', error: error.message };
    }
};

module.exports = { pool, query, transaction, healthCheck };
```

#### 2.3.2 Modelo de Producto

**Archivo: `src/models/Product.js`**

```javascript
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
        this.status = data.status;
        this.isFeatured = data.is_featured === 1;
        
        // Parsear JSON de atributos
        this.attributes = this.parseJson(data.attributes);
        
        // Virtual columns
        this.color = data.attr_color || this.getAttribute('color');
        this.size = data.attr_size || this.getAttribute('size');
        
        // Contenido
        this.description = data.description;
        this.shortDescription = data.short_description;
        
        // Timestamps
        this.createdAt = data.created_at;
        this.updatedAt = data.updated_at;
        
        // Relaciones (opcional)
        this.category = null;
        this.images = [];
    }

    parseJson(json) {
        if (!json) return {};
        if (typeof json === 'string') {
            try {
                return JSON.parse(json);
            } catch {
                return {};
            }
        }
        return json;
    }

    getAttribute(key, defaultValue = null) {
        return this.attributes[key] ?? defaultValue;
    }

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
            shortDescription: this.shortDescription,
            status: this.status,
            isFeatured: this.isFeatured,
            images: this.images,
            createdAt: this.createdAt,
            updatedAt: this.updatedAt
        };
    }
}

module.exports = Product;
```

#### 2.3.3 Repositorio de Productos

**Archivo: `src/repositories/productRepository.js`**

```javascript
const { query, transaction } = require('../config/database');
const Product = require('../models/Product');

class ProductRepository {
    
    async create(data) {
        const sql = `
            INSERT INTO products 
            (sku, name, slug, category_id, brand, price, compare_at_price, 
             stock, attributes, short_description, description, status, is_featured)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        `;
        
        const params = [
            data.sku,
            data.name,
            data.slug,
            data.categoryId,
            data.brand,
            data.price,
            data.compareAtPrice || null,
            data.stock || 0,
            JSON.stringify(data.attributes || {}),
            data.shortDescription || null,
            data.description || null,
            data.status || 'draft',
            data.isFeatured || false
        ];
        
        const result = await query(sql, params);
        return this.findById(result.insertId);
    }

    async findById(id) {
        const sql = `
            SELECT p.*, c.name as category_name, c.slug as category_slug
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE p.id = ?
        `;
        
        const rows = await query(sql, [id]);
        if (rows.length === 0) return null;
        
        const product = new Product(rows[0]);
        product.category = {
            id: product.categoryId,
            name: rows[0].category_name,
            slug: rows[0].category_slug
        };
        
        return product;
    }

    async search(filters = {}, options = {}) {
        const {
            categoryId,
            brand,
            minPrice,
            maxPrice,
            status = 'active',
            color,
            size,
            searchQuery,
            isFeatured
        } = filters;
        
        const {
            page = 1,
            limit = 20,
            sortBy = 'created_at',
            sortOrder = 'DESC'
        } = options;
        
        const whereClause = ['p.status = ?'];
        const params = [status];
        
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
        
        // Filtros en virtual columns (indexados)
        if (color) {
            whereClause.push('p.attr_color = ?');
            params.push(color);
        }
        
        if (size) {
            whereClause.push('p.attr_size = ?');
            params.push(size);
        }
        
        // Búsqueda de texto
        if (searchQuery) {
            whereClause.push('(p.name LIKE ? OR p.sku LIKE ?)');
            const like = `%${searchQuery}%`;
            params.push(like, like);
        }
        
        const whereString = whereClause.join(' AND ');
        const offset = (page - 1) * limit;
        
        // Validar sortBy para prevenir SQL injection
        const allowedSortFields = ['id', 'name', 'price', 'created_at', 'stock'];
        const safeSortBy = allowedSortFields.includes(sortBy) ? sortBy : 'created_at';
        const safeSortOrder = sortOrder.toUpperCase() === 'ASC' ? 'ASC' : 'DESC';
        
        const sql = `
            SELECT p.*, c.name as category_name
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE ${whereString}
            ORDER BY p.${safeSortBy} ${safeSortOrder}
            LIMIT ? OFFSET ?
        `;
        
        params.push(parseInt(limit), parseInt(offset));
        
        const rows = await query(sql, params);
        const products = rows.map(row => new Product(row));
        
        // Count total
        const countSql = `SELECT COUNT(*) as total FROM products p WHERE ${whereString}`;
        const countParams = params.slice(0, -2);
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

    async update(id, updateData) {
        const allowedFields = {
            name: 'name',
            slug: 'slug',
            categoryId: 'category_id',
            brand: 'brand',
            price: 'price',
            compareAtPrice: 'compare_at_price',
            stock: 'stock',
            attributes: 'attributes',
            shortDescription: 'short_description',
            description: 'description',
            status: 'status',
            isFeatured: 'is_featured'
        };
        
        const updates = [];
        const params = [];
        
        for (const [key, value] of Object.entries(updateData)) {
            if (allowedFields[key] && value !== undefined) {
                updates.push(`${allowedFields[key]} = ?`);
                params.push(key === 'attributes' ? JSON.stringify(value) : value);
            }
        }
        
        if (updates.length === 0) return null;
        
        params.push(id);
        
        const sql = `UPDATE products SET ${updates.join(', ')} WHERE id = ?`;
        await query(sql, params);
        
        return this.findById(id);
    }

    async delete(id) {
        return await transaction(async (conn) => {
            await conn.execute('DELETE FROM product_images WHERE product_id = ?', [id]);
            await conn.execute('DELETE FROM product_variants WHERE product_id = ?', [id]);
            const [result] = await conn.execute('DELETE FROM products WHERE id = ?', [id]);
            return result.affectedRows > 0;
        });
    }
}

module.exports = new ProductRepository();
```

#### 2.3.4 Servicio de Validación de Atributos

**Archivo: `src/services/validationService.js`**

```javascript
const attributeRepository = require('../repositories/attributeRepository');

class ValidationService {
    
    async validateProductAttributes(categoryId, attributes) {
        const definitions = await attributeRepository.findByCategory(categoryId);
        const errors = [];
        
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
            
            if (!value && !def.is_required) continue;
            
            // Validar tipo
            const typeError = this.validateType(value, def.data_type, def);
            if (typeError) {
                errors.push(typeError);
                continue;
            }
            
            // Validar reglas
            if (def.validation_rules) {
                const rules = typeof def.validation_rules === 'string' 
                    ? JSON.parse(def.validation_rules) 
                    : def.validation_rules;
                
                const ruleError = this.validateRules(value, rules, def);
                if (ruleError) errors.push(ruleError);
            }
            
            // Validar opciones enum
            if (def.data_type === 'enum' && def.options) {
                const options = typeof def.options === 'string' 
                    ? JSON.parse(def.options) 
                    : def.options;
                
                if (!options.includes(value)) {
                    errors.push({
                        field: def.code,
                        message: `"${def.name}" debe ser uno de: ${options.join(', ')}`
                    });
                }
            }
        }
        
        return {
            isValid: errors.length === 0,
            errors
        };
    }

    validateType(value, dataType, def) {
        switch (dataType) {
            case 'number':
                if (typeof value !== 'number' || isNaN(value)) {
                    return {
                        field: def.code,
                        message: `"${def.name}" debe ser un número`
                    };
                }
                break;
                
            case 'boolean':
                if (typeof value !== 'boolean') {
                    return {
                        field: def.code,
                        message: `"${def.name}" debe ser verdadero o falso`
                    };
                }
                break;
                
            case 'date':
                if (isNaN(Date.parse(value))) {
                    return {
                        field: def.code,
                        message: `"${def.name}" debe ser una fecha válida`
                    };
                }
                break;
        }
        return null;
    }

    validateRules(value, rules, def) {
        if (rules.min !== undefined && value < rules.min) {
            return {
                field: def.code,
                message: `"${def.name}" debe ser mayor o igual a ${rules.min}`
            };
        }
        
        if (rules.max !== undefined && value > rules.max) {
            return {
                field: def.code,
                message: `"${def.name}" debe ser menor o igual a ${rules.max}`
            };
        }
        
        if (rules.maxLength && String(value).length > rules.maxLength) {
            return {
                field: def.code,
                message: `"${def.name}" no puede exceder ${rules.maxLength} caracteres`
            };
        }
        
        return null;
    }
}

module.exports = new ValidationService();
```

#### 2.3.5 Controllers y Routes

**Archivo: `src/controllers/productController.js`**

```javascript
const productService = require('../services/productService');
const validationService = require('../services/validationService');

class ProductController {
    
    async list(req, res, next) {
        try {
            const filters = {
                categoryId: req.query.category_id,
                brand: req.query.brand,
                minPrice: req.query.min_price,
                maxPrice: req.query.max_price,
                status: req.query.status || 'active',
                color: req.query.color,
                size: req.query.size,
                searchQuery: req.query.q,
                isFeatured: req.query.is_featured === 'true'
            };
            
            const options = {
                page: parseInt(req.query.page) || 1,
                limit: Math.min(parseInt(req.query.limit) || 20, 100),
                sortBy: req.query.sort_by,
                sortOrder: req.query.sort_order
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

    async create(req, res, next) {
        try {
            const productData = req.body;
            
            // Validar atributos
            if (productData.attributes && productData.categoryId) {
                const validation = await validationService.validateProductAttributes(
                    productData.categoryId,
                    productData.attributes
                );
                
                if (!validation.isValid) {
                    return res.status(400).json({
                        success: false,
                        message: 'Validación de atributos fallida',
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

    async update(req, res, next) {
        try {
            const { id } = req.params;
            const updateData = req.body;
            
            // Validar atributos si se actualizan
            if (updateData.attributes) {
                const product = await productService.findById(id);
                if (!product) {
                    return res.status(404).json({
                        success: false,
                        message: 'Producto no encontrado'
                    });
                }
                
                const categoryId = updateData.categoryId || product.categoryId;
                const mergedAttrs = { ...product.attributes, ...updateData.attributes };
                
                const validation = await validationService.validateProductAttributes(
                    categoryId, mergedAttrs
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
                message: 'Producto eliminado correctamente'
            });
        } catch (error) {
            next(error);
        }
    }
}

module.exports = new ProductController();
```

**Archivo: `src/routes/products.js`**

```javascript
const express = require('express');
const router = express.Router();
const productController = require('../controllers/productController');

router.get('/', productController.list);
router.get('/:id', productController.getById);
router.post('/', productController.create);
router.put('/:id', productController.update);
router.delete('/:id', productController.delete);

module.exports = router;
```

**Archivo: `src/routes/index.js`**

```javascript
const express = require('express');
const router = express.Router();

const productsRouter = require('./products');
const categoriesRouter = require('./categories');
const attributesRouter = require('./attributes');

router.use('/products', productsRouter);
router.use('/categories', categoriesRouter);
router.use('/attributes', attributesRouter);

module.exports = router;
```

#### 2.3.6 Aplicación Express

**Archivo: `src/app.js`**

```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');

const routes = require('./routes');
const errorHandler = require('./middleware/errorHandler');
const { healthCheck } = require('./config/database');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors());

// Rate limiting (Hostinger: limitado recursos)
const limiter = rateLimit({
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW) || 60000,
    max: parseInt(process.env.RATE_LIMIT_MAX) || 100,
    message: {
        success: false,
        message: 'Demasiadas peticiones, intente más tarde'
    }
});
app.use(limiter);

// Logging
app.use(morgan('combined'));

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Health check
app.get('/health', async (req, res) => {
    const dbHealth = await healthCheck();
    res.json({
        success: true,
        service: process.env.APP_NAME || 'msproducto',
        version: process.env.APP_VERSION || '1.0.0',
        timestamp: new Date().toISOString(),
        database: dbHealth
    });
});

// API routes
const apiPrefix = process.env.API_PREFIX || '/api/v1';
app.use(apiPrefix, routes);

// 404 handler
app.use((req, res) => {
    res.status(404).json({
        success: false,
        message: 'Endpoint no encontrado'
    });
});

// Error handler
app.use(errorHandler);

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received, closing gracefully');
    process.exit(0);
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`msproducto running on port ${PORT}`);
});

module.exports = app;
```

**Archivo: `src/middleware/errorHandler.js`**

```javascript
module.exports = (err, req, res, next) => {
    console.error('Error:', err);
    
    // MySQL errors
    if (err.code === 'ER_DUP_ENTRY') {
        return res.status(409).json({
            success: false,
            message: 'Registro duplicado'
        });
    }
    
    if (err.code === 'ER_NO_REFERENCED_ROW') {
        return res.status(400).json({
            success: false,
            message: 'Referencia no válida'
        });
    }
    
    // Validation errors
    if (err.name === 'ValidationError') {
        return res.status(400).json({
            success: false,
            message: 'Error de validación',
            errors: err.errors
        });
    }
    
    // Default error
    res.status(err.status || 500).json({
        success: false,
        message: err.message || 'Error interno del servidor',
        ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
    });
};
```

**Entregables Fase 3:**
- [ ] Código fuente completo implementado
- [ ] Repositorios, servicios y controllers funcionando
- [ ] Validación de atributos operativa
- [ ] Manejo de errores implementado

---

### 🧪 FASE 4: Testing y Despliegue (Estimado: 1 día)

#### 2.4.1 Tests Básicos

**Archivo: `tests/integration/products.test.js`**

```javascript
const request = require('supertest');
const app = require('../../src/app');

describe('Products API', () => {
    
    describe('GET /api/v1/products', () => {
        test('debe listar productos', async () => {
            const res = await request(app)
                .get('/api/v1/products')
                .expect(200);
            
            expect(res.body.success).toBe(true);
            expect(Array.isArray(res.body.data)).toBe(true);
            expect(res.body.pagination).toBeDefined();
        });
        
        test('debe filtrar por categoría', async () => {
            const res = await request(app)
                .get('/api/v1/products?category_id=1')
                .expect(200);
            
            expect(res.body.success).toBe(true);
        });
    });
    
    describe('POST /api/v1/products', () => {
        test('debe crear producto con atributos', async () => {
            const productData = {
                sku: 'TEST-001',
                name: 'Producto Test',
                slug: 'producto-test',
                categoryId: 2,
                brand: 'TestBrand',
                price: 99.99,
                stock: 10,
                attributes: {
                    cpu: 'Intel i5',
                    ram_gb: 16,
                    color: 'Black'
                },
                status: 'active'
            };
            
            const res = await request(app)
                .post('/api/v1/products')
                .send(productData)
                .expect(201);
            
            expect(res.body.success).toBe(true);
            expect(res.body.data.sku).toBe(productData.sku);
            expect(res.body.data.attributes.cpu).toBe('Intel i5');
        });
        
        test('debe rechazar SKU duplicado', async () => {
            const res = await request(app)
                .post('/api/v1/products')
                .send({ sku: 'TEST-001', name: 'Otro' })
                .expect(409);
            
            expect(res.body.success).toBe(false);
        });
    });
});
```

#### 2.4.2 Scripts de Build

**Archivo: `package.json`**

```json
{
  "name": "msproducto",
  "version": "1.0.0",
  "description": "Microservicio de productos dinámico",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest --coverage",
    "test:watch": "jest --watch",
    "lint": "eslint src/"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.18.3",
    "express-rate-limit": "^7.2.0",
    "helmet": "^7.1.0",
    "joi": "^17.12.2",
    "morgan": "^1.10.0",
    "mysql2": "^3.9.2",
    "slugify": "^1.6.6"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "nodemon": "^3.1.0",
    "supertest": "^6.3.4"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

#### 2.4.3 Despliegue en Hostinger

**Instrucciones:**

```bash
# 1. Subir código a GitHub
git add .
git commit -m "feat: Implementación completa msproducto"
git push origin main

# 2. En Hostinger hPanel:
# - Ir a "Node.js" 
# - Configurar:
#   * Repository: https://github.com/eugarte/msproducto
#   * Branch: main
#   * Startup file: src/app.js
#   * Node version: 18.x

# 3. Configurar Variables de Entorno en hPanel
# (Ver sección FASE 1)

# 4. Ejecutar migraciones SQL
# - Ir a phpMyAdmin
# - Importar: sql/01_schema.sql, sql/02_indexes.sql, sql/03_seed.sql

# 5. Deploy
# - Click "Deploy" en hPanel Node.js
# - Verificar logs
```

**Entregables Fase 4:**
- [ ] Tests ejecutándose correctamente
- [ ] Código en GitHub
- [ ] Despliegue exitoso en Hostinger
- [ ] Health check respondiendo
- [ ] Datos de prueba visibles

---

## 3. Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ARQUITECTURA MSPRODUCTO                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐      ┌──────────────────────────────────────────┐     │
│  │   Cliente   │─────▶│          Hostinger Node.js                │     │
│  │  (Web/App)  │◀─────│  ┌────────────────────────────────────┐  │     │
│  └─────────────┘      │  │        Express App                  │  │     │
│                       │  │  ┌─────────┬─────────┬───────────┐  │  │     │
│                       │  │  │Routes   │Services │Repositories│  │  │     │
│                       │  │  └────┬────┴────┬────┴─────┬─────┘  │  │     │
│                       │  └───────┼─────────┼──────────┼────────┘  │     │
│                       └──────────┼─────────┼──────────┼───────────┘     │
│                                  │         │          │                 │
│                                  ▼         ▼          ▼                 │
│                       ┌──────────────────────────────────────┐          │
│                       │        MySQL (Hostinger)              │          │
│                       │  ┌──────────┬──────────┬──────────┐   │          │
│                       │  │categories│ products │ variants │   │          │
│                       │  │attributes│  images  │relations │   │          │
│                       │  └──────────┴──────────┴──────────┘   │          │
│                       └──────────────────────────────────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Endpoints API

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | /health | Health check |
| GET | /api/v1/products | Listar productos |
| GET | /api/v1/products/:id | Obtener producto |
| POST | /api/v1/products | Crear producto |
| PUT | /api/v1/products/:id | Actualizar producto |
| DELETE | /api/v1/products/:id | Eliminar producto |
| GET | /api/v1/categories | Listar categorías |
| GET | /api/v1/categories/:id | Obtener categoría |
| GET | /api/v1/attributes | Listar atributos |
| GET | /api/v1/categories/:id/attributes | Atributos por categoría |

---

## 5. Métricas de Éxito

### 5.1 Rendimiento
- [ ] Tiempo de respuesta promedio < 200ms
- [ ] Soporte para 100+ productos iniciales
- [ ] Conexiones DB estable < 15

### 5.2 Funcionalidad
- [ ] CRUD de productos funcionando
- [ ] Validación de atributos operativa
- [ ] Búsquedas por filtros funcionando
- [ ] Esquema dinámico probado

### 5.3 Calidad
- [ ] Tests pasando > 80% coverage
- [ ] Sin errores en logs
- [ ] Health check OK

---

## 6. Consideraciones de Hostinger

| Aspecto | Configuración |
|---------|---------------|
| **Base de Datos** | MySQL 8.0 (utf8mb4) |
| **Pool de Conexiones** | Máximo 15 |
| **Rate Limiting** | 100 req/min por IP |
| **Timeout** | 30s requests, 10s DB |
| **Logs** | Console.log → hPanel logs |
| **SSL** | Automático (Let's Encrypt) |

---

**Versión:** 1.0  
**Fecha:** 2024  
**Autor:** ZagaloAI  
**Estimación Total:** 5 días hábiles
