# ตอนที่ 106: SQL with Node.js/JavaScript

## บทนำ

Node.js มี ecosystem ที่หลากหลายสำหรับการทำงานกับฐานข้อมูล ตั้งแต่ raw drivers ไปจนถึง ORMs และ query builders สมัยใหม่ ในบทนี้เราจะศึกษาการใช้ SQL กับ Node.js อย่างครบถ้วน ตั้งแต่ node-postgres ไปจนถึง Prisma และ Drizzle ORM

---

## 1. node-postgres (pg)

### 1.1 Basic Connection

```javascript
const { Pool, Client } = require('pg');

// Single client connection
const client = new Client({
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    user: 'postgres',
    password: process.env.DB_PASSWORD,
    connectionTimeoutMillis: 5000,
});

await client.connect();
const result = await client.query('SELECT NOW()');
console.log(result.rows[0]);
await client.end();

// Connection Pool (แนะนำสำหรับ production)
const pool = new Pool({
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT) || 5432,
    database: process.env.DB_NAME || 'mydb',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD,
    max: 20,                    // maximum connections
    idleTimeoutMillis: 30000,   // close idle connections after 30s
    connectionTimeoutMillis: 2000, // wait 2s for connection
    ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
});

// Event handlers
pool.on('error', (err, client) => {
    console.error('Unexpected error on idle client', err);
    process.exit(-1);
});

pool.on('connect', (client) => {
    console.log('New connection established');
});

// Query
async function getUsers() {
    const { rows } = await pool.query(
        'SELECT id, username, email FROM users WHERE is_active = $1 ORDER BY username',
        [true]
    );
    return rows;
}

// Named parameters ด้วย object (ต้องใช้ pg-named หรือ format เอง)
// หรือใช้ positional $1, $2, ...
async function searchProducts(keyword, minPrice, maxPrice) {
    const query = {
        text: `
            SELECT id, sku, name, price
            FROM products
            WHERE name ILIKE $1
            AND price BETWEEN $2 AND $3
            ORDER BY price
        `,
        values: [`%${keyword}%`, minPrice, maxPrice]
    };
    
    const { rows } = await pool.query(query);
    return rows;
}
```

### 1.2 Transaction กับ pg

```javascript
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Transaction ด้วย try/catch
async function transferFunds(fromAccountId, toAccountId, amount) {
    const client = await pool.connect();
    
    try {
        await client.query('BEGIN');
        
        // Lock rows for update
        const fromAccount = await client.query(
            'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
            [fromAccountId]
        );
        
        if (!fromAccount.rows[0]) {
            throw new Error('Source account not found');
        }
        
        if (fromAccount.rows[0].balance < amount) {
            throw new Error(`Insufficient funds. Available: ${fromAccount.rows[0].balance}`);
        }
        
        await client.query(
            'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
            [amount, fromAccountId]
        );
        
        await client.query(
            'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
            [amount, toAccountId]
        );
        
        await client.query(
            `INSERT INTO transactions (from_account, to_account, amount, type)
             VALUES ($1, $2, $3, 'transfer')`,
            [fromAccountId, toAccountId, amount]
        );
        
        await client.query('COMMIT');
        
        return { success: true, amount };
        
    } catch (error) {
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release(); // คืน connection กลับ pool
    }
}

// Higher-order function สำหรับ transactions
async function withTransaction(pool, callback) {
    const client = await pool.connect();
    try {
        await client.query('BEGIN');
        const result = await callback(client);
        await client.query('COMMIT');
        return result;
    } catch (error) {
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release();
    }
}

// ใช้งาน
const result = await withTransaction(pool, async (client) => {
    const order = await client.query(
        'INSERT INTO orders (customer_id, total) VALUES ($1, $2) RETURNING *',
        [customerId, totalAmount]
    );
    
    for (const item of orderItems) {
        await client.query(
            'INSERT INTO order_items (order_id, product_id, quantity) VALUES ($1, $2, $3)',
            [order.rows[0].id, item.productId, item.quantity]
        );
    }
    
    return order.rows[0];
});
```

### 1.3 Advanced pg Features

```javascript
// Cursor สำหรับ large result sets
const Cursor = require('pg-cursor');

async function processLargeTable(pool) {
    const client = await pool.connect();
    const cursor = client.query(new Cursor('SELECT * FROM large_table ORDER BY id'));
    
    try {
        let rows;
        while (true) {
            rows = await cursor.read(1000); // อ่านทีละ 1000 rows
            if (rows.length === 0) break;
            
            for (const row of rows) {
                // process row...
                console.log(row.id);
            }
        }
    } finally {
        await cursor.close();
        client.release();
    }
}

// pg-format สำหรับ safe dynamic SQL
const format = require('pg-format');

async function insertMultiple(pool, tableName, data) {
    // %I = identifier (escaped), %L = literal (escaped), %s = string (NOT escaped)
    const query = format(
        'INSERT INTO %I (%I, %I, %I) VALUES %L',
        tableName,
        'name', 'email', 'role',
        data.map(d => [d.name, d.email, d.role])
    );
    
    return pool.query(query);
}

// LISTEN/NOTIFY
async function setupNotificationListener(pool) {
    const client = await pool.connect();
    
    client.on('notification', (msg) => {
        console.log(`Received ${msg.channel}: ${msg.payload}`);
        try {
            const data = JSON.parse(msg.payload);
            // Process notification...
        } catch (e) {
            console.error('Invalid JSON payload', e);
        }
    });
    
    await client.query('LISTEN order_created');
    await client.query('LISTEN inventory_updated');
    
    console.log('Listening for notifications...');
    
    // Cleanup on process exit
    process.on('SIGINT', async () => {
        await client.query('UNLISTEN *');
        client.release();
        await pool.end();
        process.exit(0);
    });
}
```

---

## 2. mysql2

### 2.1 Basic MySQL Connection

```javascript
const mysql = require('mysql2/promise');

// Single connection
const connection = await mysql.createConnection({
    host: 'localhost',
    port: 3306,
    user: 'root',
    password: process.env.MYSQL_PASSWORD,
    database: 'mydb',
    charset: 'utf8mb4',
    timezone: '+07:00',
    connectTimeout: 10000
});

// Execute query
const [rows] = await connection.execute(
    'SELECT * FROM users WHERE id = ?',
    [userId]
);
console.log(rows);

// Connection Pool
const pool = mysql.createPool({
    host: process.env.MYSQL_HOST,
    user: process.env.MYSQL_USER,
    password: process.env.MYSQL_PASSWORD,
    database: process.env.MYSQL_DATABASE,
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0,
    enableKeepAlive: true,
    keepAliveInitialDelay: 0
});

// Promise API
const promisePool = pool.promise();

async function getProducts(category) {
    const [rows] = await promisePool.execute(
        'SELECT * FROM products WHERE category = ? AND is_active = 1',
        [category]
    );
    return rows;
}

// Prepared statements (แนะนำ)
async function createUser(username, email, passwordHash) {
    const [result] = await promisePool.execute(
        'INSERT INTO users (username, email, password_hash) VALUES (?, ?, ?)',
        [username, email, passwordHash]
    );
    
    return {
        id: result.insertId,
        affectedRows: result.affectedRows
    };
}
```

### 2.2 MySQL Transaction

```javascript
async function placeOrder(customerId, items) {
    const conn = await pool.promise().getConnection();
    
    try {
        await conn.beginTransaction();
        
        // สร้าง order
        const [orderResult] = await conn.execute(
            'INSERT INTO orders (customer_id, status) VALUES (?, "pending")',
            [customerId]
        );
        const orderId = orderResult.insertId;
        
        let totalAmount = 0;
        
        for (const item of items) {
            // ตรวจสอบ stock พร้อม lock
            const [stock] = await conn.execute(
                'SELECT quantity FROM inventory WHERE product_id = ? FOR UPDATE',
                [item.productId]
            );
            
            if (!stock[0] || stock[0].quantity < item.quantity) {
                throw new Error(`Insufficient stock for product ${item.productId}`);
            }
            
            // Get product price
            const [product] = await conn.execute(
                'SELECT price FROM products WHERE id = ?',
                [item.productId]
            );
            
            const itemTotal = product[0].price * item.quantity;
            totalAmount += itemTotal;
            
            // Insert order item
            await conn.execute(
                'INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (?, ?, ?, ?)',
                [orderId, item.productId, item.quantity, product[0].price]
            );
            
            // Update stock
            await conn.execute(
                'UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?',
                [item.quantity, item.productId]
            );
        }
        
        // Update order total
        await conn.execute(
            'UPDATE orders SET total_amount = ? WHERE id = ?',
            [totalAmount, orderId]
        );
        
        await conn.commit();
        return { orderId, totalAmount };
        
    } catch (error) {
        await conn.rollback();
        throw error;
    } finally {
        conn.release();
    }
}
```

---

## 3. better-sqlite3

### 3.1 Synchronous SQLite

```javascript
const Database = require('better-sqlite3');
const path = require('path');

// เปิด database (synchronous API)
const db = new Database(path.join(__dirname, 'app.db'), {
    verbose: process.env.NODE_ENV === 'development' ? console.log : undefined,
    fileMustExist: false
});

// Optimize settings
db.pragma('journal_mode = WAL');
db.pragma('synchronous = NORMAL');
db.pragma('cache_size = -64000'); // 64MB
db.pragma('foreign_keys = ON');

// สร้างตาราง
db.exec(`
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT NOT NULL UNIQUE,
        email TEXT NOT NULL,
        created_at TEXT DEFAULT (datetime('now'))
    );
    
    CREATE TABLE IF NOT EXISTS sessions (
        id TEXT PRIMARY KEY,
        user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        created_at TEXT DEFAULT (datetime('now')),
        expires_at TEXT NOT NULL
    );
    
    CREATE INDEX IF NOT EXISTS idx_sessions_user ON sessions(user_id);
    CREATE INDEX IF NOT EXISTS idx_sessions_expires ON sessions(expires_at);
`);

// Prepared statements (เร็วกว่า run() สำหรับ queries ที่รัน บ่อย ๆ)
const insertUser = db.prepare(
    'INSERT INTO users (username, email) VALUES (@username, @email)'
);

const getUserById = db.prepare(
    'SELECT * FROM users WHERE id = ?'
);

const searchUsers = db.prepare(
    'SELECT * FROM users WHERE username LIKE ? OR email LIKE ? ORDER BY username LIMIT ?'
);

// ใช้งาน
const result = insertUser.run({ username: 'alice', email: 'alice@example.com' });
console.log(result.lastInsertRowid);

const user = getUserById.get(1); // get() = fetchone()
console.log(user);

const users = searchUsers.all('%alice%', '%alice%', 10); // all() = fetchall()
console.log(users);

// Transaction (synchronous)
const placeOrderSync = db.transaction((customerId, items) => {
    const createOrder = db.prepare(
        'INSERT INTO orders (customer_id) VALUES (?) RETURNING id'
    );
    const { id: orderId } = createOrder.get(customerId);
    
    const insertItem = db.prepare(
        'INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (?, ?, ?, ?)'
    );
    
    for (const item of items) {
        insertItem.run(orderId, item.productId, item.quantity, item.price);
    }
    
    return orderId;
});

// Transactions สามารถ nest ได้ (inner จะเป็น savepoint)
const outerTx = db.transaction(() => {
    const innerTx = db.transaction(() => {
        insertUser.run({ username: 'nested', email: 'nested@example.com' });
    });
    
    innerTx(); // inner transaction
    // ถ้า inner throw จะ rollback แค่ inner savepoint
});
```

### 3.2 Advanced better-sqlite3

```javascript
// Custom functions
db.function('regexp', { deterministic: true }, (pattern, str) => {
    return new RegExp(pattern).test(str) ? 1 : 0;
});

// ใช้ custom function ใน query
const results = db.prepare(
    "SELECT * FROM products WHERE regexp(?, sku)"
).all('^PHONE-\\d{3}$');

// Aggregate function
db.aggregate('sum_of_squares', {
    start: 0,
    step: (accumulator, value) => accumulator + value * value,
    result: (accumulator) => Math.sqrt(accumulator)
});

const rms = db.prepare('SELECT sum_of_squares(price) AS rms FROM products').pluck().get();

// Virtual table (JSON)
db.exec(`
    CREATE VIRTUAL TABLE IF NOT EXISTS json_each_products 
    USING json_each('[1,2,3]')
`);

// Backup
db.backup('backup.db')
    .then(() => console.log('Backup complete'))
    .catch(console.error);

// Serialize/deserialize
const data = db.prepare("SELECT * FROM users").all();
const serialized = JSON.stringify(data);

// Memory database
const memDb = new Database(':memory:');
memDb.pragma('journal_mode = MEMORY');
```

---

## 4. Knex.js Query Builder

### 4.1 Setup และ Basic Usage

```javascript
const knex = require('knex')({
    client: 'pg',  // 'mysql', 'mysql2', 'sqlite3', 'mssql'
    connection: {
        host: process.env.DB_HOST,
        port: parseInt(process.env.DB_PORT),
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME,
    },
    pool: {
        min: 2,
        max: 10,
        acquireTimeoutMillis: 30000,
        idleTimeoutMillis: 600000
    },
    searchPath: ['public'], // PostgreSQL schema
    debug: process.env.NODE_ENV === 'development',
    asyncStackTraces: process.env.NODE_ENV === 'development'
});

// Basic CRUD
// SELECT
const users = await knex('users')
    .select('id', 'username', 'email')
    .where('is_active', true)
    .orderBy('username')
    .limit(20)
    .offset(0);

// SELECT with conditions
const products = await knex('products')
    .where(function() {
        this.where('price', '>=', 1000).andWhere('price', '<=', 50000)
    })
    .orWhere('is_featured', true)
    .whereNotNull('image_url')
    .orderBy([
        { column: 'is_featured', order: 'desc' },
        { column: 'price', order: 'asc' }
    ])
    .limit(20);

// INSERT
const [id] = await knex('users').insert({
    username: 'alice',
    email: 'alice@example.com',
    created_at: new Date()
}).returning('id');  // PostgreSQL specific

// ใน MySQL/SQLite
const result = await knex('users').insert({ username: 'alice', email: 'alice@example.com' });
const insertedId = result[0]; // insertId

// UPDATE
const updateCount = await knex('users')
    .where('id', userId)
    .update({ 
        last_login: knex.fn.now(), 
        login_count: knex.raw('login_count + 1') 
    });

// DELETE
await knex('sessions')
    .where('expires_at', '<', new Date())
    .delete();

// UPSERT (INSERT ... ON CONFLICT DO UPDATE)
await knex('products')
    .insert({ sku: 'SKU-001', name: 'Product', price: 1000 })
    .onConflict('sku')
    .merge(['name', 'price', 'updated_at']); // update these columns

// PostgreSQL: DO NOTHING
await knex('event_log')
    .insert({ event_id: eventId, user_id: userId })
    .onConflict(['event_id', 'user_id'])
    .ignore();
```

### 4.2 Knex Joins และ Aggregations

```javascript
// JOIN
const ordersWithCustomers = await knex('orders as o')
    .join('customers as c', 'o.customer_id', 'c.id')
    .leftJoin('promotions as p', 'o.promo_code', 'p.code')
    .select(
        'o.id as order_id',
        'c.name as customer_name',
        'o.total_amount',
        knex.raw("COALESCE(p.discount_percent, 0) as discount")
    )
    .where('o.status', 'completed')
    .orderBy('o.created_at', 'desc');

// Aggregation
const salesStats = await knex('orders')
    .join('order_items as oi', 'orders.id', 'oi.order_id')
    .join('products as p', 'oi.product_id', 'p.id')
    .select('p.category')
    .count('orders.id as order_count')
    .sum('oi.quantity as total_qty')
    .sum(knex.raw('oi.quantity * oi.unit_price as total_revenue'))
    .avg('orders.total_amount as avg_order_value')
    .where('orders.status', 'completed')
    .whereRaw('orders.created_at >= NOW() - INTERVAL \'30 days\'')
    .groupBy('p.category')
    .orderBy('total_revenue', 'desc');

// Subquery
const topCustomers = await knex('customers')
    .whereIn('id', function() {
        this.select('customer_id')
            .from('orders')
            .where('status', 'completed')
            .groupBy('customer_id')
            .havingRaw('SUM(total_amount) > ?', [100000]);
    })
    .select('id', 'name', 'email');

// CTE (Common Table Expression)
const monthlyRevenue = await knex
    .with('monthly', knex.raw(`
        SELECT 
            DATE_TRUNC('month', created_at) as month,
            SUM(total_amount) as revenue
        FROM orders
        WHERE status = 'completed'
        GROUP BY 1
    `))
    .select('month', 'revenue')
    .from('monthly')
    .orderBy('month');

// Raw query
const result = await knex.raw(
    'SELECT * FROM search_products(?, ?) WHERE price BETWEEN ? AND ?',
    [keyword, category, minPrice, maxPrice]
);
```

### 4.3 Knex Migrations

```javascript
// knexfile.js
module.exports = {
    development: {
        client: 'sqlite3',
        connection: { filename: './dev.sqlite3' },
        migrations: { directory: './migrations' },
        seeds: { directory: './seeds' }
    },
    production: {
        client: 'pg',
        connection: process.env.DATABASE_URL,
        migrations: { directory: './migrations' },
        pool: { min: 2, max: 10 }
    }
};

// migrations/001_create_users.js
exports.up = async function(knex) {
    await knex.schema.createTable('users', (table) => {
        table.increments('id').primary();
        table.string('username', 50).notNullable().unique();
        table.string('email', 255).notNullable().unique();
        table.string('password_hash', 255);
        table.boolean('is_active').defaultTo(true);
        table.string('role', 20).defaultTo('user');
        table.timestamps(true, true); // created_at, updated_at
        
        table.index(['email']);
        table.index(['role', 'is_active']);
    });
    
    await knex.schema.createTable('profiles', (table) => {
        table.increments('id').primary();
        table.integer('user_id').notNullable().references('id').inTable('users').onDelete('CASCADE');
        table.string('first_name', 50);
        table.string('last_name', 50);
        table.string('avatar_url', 500);
        table.jsonb('preferences').defaultTo('{}');
        table.unique(['user_id']);
    });
};

exports.down = async function(knex) {
    await knex.schema.dropTableIfExists('profiles');
    await knex.schema.dropTableIfExists('users');
};

// migrations/002_add_phone_to_users.js
exports.up = async function(knex) {
    await knex.schema.alterTable('users', (table) => {
        table.string('phone', 20).nullable().after('email');
        table.index(['phone']);
    });
};

exports.down = async function(knex) {
    await knex.schema.alterTable('users', (table) => {
        table.dropIndex(['phone']);
        table.dropColumn('phone');
    });
};

// seeds/01_users.js
exports.seed = async function(knex) {
    await knex('users').del();
    await knex('users').insert([
        { username: 'admin', email: 'admin@example.com', role: 'admin' },
        { username: 'user1', email: 'user1@example.com', role: 'user' },
    ]);
};
```

---

## 5. Sequelize ORM

### 5.1 Model Definition

```javascript
const { Sequelize, DataTypes, Model, Op } = require('sequelize');

const sequelize = new Sequelize({
    dialect: 'postgres',
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT),
    username: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    pool: {
        max: 10,
        min: 0,
        acquire: 30000,
        idle: 10000
    },
    logging: process.env.NODE_ENV === 'development' ? console.log : false,
    define: {
        underscored: true,         // snake_case column names
        timestamps: true,          // createdAt, updatedAt
        paranoid: false,           // soft delete
        freezeTableName: false     // pluralize table names
    }
});

// User Model
class User extends Model {}
User.init({
    id: {
        type: DataTypes.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    username: {
        type: DataTypes.STRING(50),
        allowNull: false,
        unique: true,
        validate: {
            len: [3, 50],
            isAlphanumeric: true
        }
    },
    email: {
        type: DataTypes.STRING(255),
        allowNull: false,
        unique: true,
        validate: {
            isEmail: true
        }
    },
    isActive: {
        type: DataTypes.BOOLEAN,
        defaultValue: true
    }
}, {
    sequelize,
    modelName: 'User',
    tableName: 'users',
    indexes: [
        { fields: ['email'] },
        { fields: ['username'] }
    ]
});

// Product Model
class Product extends Model {
    getDisplayPrice() {
        return `${this.price.toLocaleString()} THB`;
    }
}
Product.init({
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    sku: { type: DataTypes.STRING(50), unique: true, allowNull: false },
    name: { type: DataTypes.STRING(200), allowNull: false },
    price: {
        type: DataTypes.DECIMAL(10, 2),
        allowNull: false,
        validate: { min: 0 }
    },
    stockQuantity: { type: DataTypes.INTEGER, defaultValue: 0 },
    isActive: { type: DataTypes.BOOLEAN, defaultValue: true }
}, { sequelize, modelName: 'Product' });

// Associations
User.hasMany(Order, { foreignKey: 'customerId', as: 'orders' });
Order.belongsTo(User, { foreignKey: 'customerId', as: 'customer' });
Order.hasMany(OrderItem, { foreignKey: 'orderId', as: 'items' });
OrderItem.belongsTo(Order, { foreignKey: 'orderId' });
OrderItem.belongsTo(Product, { foreignKey: 'productId', as: 'product' });
Product.hasMany(OrderItem, { foreignKey: 'productId' });

// Sync (development only)
await sequelize.sync({ alter: true });
```

### 5.2 Sequelize Queries

```javascript
// findAll
const users = await User.findAll({
    where: {
        isActive: true,
        createdAt: {
            [Op.gte]: new Date('2024-01-01')
        }
    },
    order: [['username', 'ASC']],
    limit: 20,
    offset: 0,
    attributes: ['id', 'username', 'email'] // select specific columns
});

// findOne
const user = await User.findOne({
    where: { email: 'alice@example.com' },
    include: [{
        model: Order,
        as: 'orders',
        required: false,
        include: [{
            model: OrderItem,
            as: 'items',
            include: [{ model: Product, as: 'product' }]
        }],
        limit: 5,
        order: [['createdAt', 'DESC']]
    }]
});

// findByPk (find by primary key)
const product = await Product.findByPk(1);

// findOrCreate
const [user, created] = await User.findOrCreate({
    where: { email: 'new@example.com' },
    defaults: { username: 'newuser', isActive: true }
});

// Aggregation
const stats = await Order.findAll({
    attributes: [
        [sequelize.fn('DATE_TRUNC', 'month', sequelize.col('created_at')), 'month'],
        [sequelize.fn('COUNT', sequelize.col('id')), 'orderCount'],
        [sequelize.fn('SUM', sequelize.col('total_amount')), 'totalRevenue']
    ],
    where: {
        status: 'completed',
        createdAt: {
            [Op.gte]: sequelize.literal("NOW() - INTERVAL '12 months'")
        }
    },
    group: ['month'],
    order: [[sequelize.literal('month'), 'ASC']]
});

// Raw query
const [results] = await sequelize.query(
    'SELECT * FROM products WHERE price BETWEEN :min AND :max',
    {
        replacements: { min: 1000, max: 50000 },
        type: sequelize.QueryTypes.SELECT
    }
);

// Transaction
const transaction = await sequelize.transaction();
try {
    const order = await Order.create({ customerId: 1, status: 'pending' }, { transaction });
    await OrderItem.bulkCreate([
        { orderId: order.id, productId: 1, quantity: 2, unitPrice: 500 }
    ], { transaction });
    await transaction.commit();
} catch (error) {
    await transaction.rollback();
    throw error;
}
```

---

## 6. Prisma ORM

### 6.1 Schema Definition

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  username  String   @unique @db.VarChar(50)
  email     String   @unique @db.VarChar(255)
  isActive  Boolean  @default(true)
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  orders    Order[]
  profile   Profile?
  
  @@map("users")
  @@index([email])
}

enum Role {
  USER
  ADMIN
  MANAGER
}

model Profile {
  id        Int     @id @default(autoincrement())
  userId    Int     @unique
  firstName String? @db.VarChar(50)
  lastName  String? @db.VarChar(50)
  bio       String?
  
  user      User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("profiles")
}

model Product {
  id            Int      @id @default(autoincrement())
  sku           String   @unique @db.VarChar(50)
  name          String   @db.VarChar(200)
  price         Decimal  @db.Decimal(10, 2)
  stockQuantity Int      @default(0)
  isActive      Boolean  @default(true)
  category      Category? @relation(fields: [categoryId], references: [id])
  categoryId    Int?
  
  orderItems    OrderItem[]
  
  @@map("products")
  @@index([categoryId, isActive])
}

model Order {
  id          Int         @id @default(autoincrement())
  customerId  Int
  status      OrderStatus @default(PENDING)
  totalAmount Decimal     @db.Decimal(12, 2) @default(0)
  createdAt   DateTime    @default(now())
  
  customer    User        @relation(fields: [customerId], references: [id])
  items       OrderItem[]
  
  @@map("orders")
}

enum OrderStatus {
  PENDING
  PROCESSING
  COMPLETED
  CANCELLED
}

model OrderItem {
  id        Int     @id @default(autoincrement())
  orderId   Int
  productId Int
  quantity  Int
  unitPrice Decimal @db.Decimal(10, 2)
  
  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product   Product @relation(fields: [productId], references: [id])
  
  @@map("order_items")
  @@unique([orderId, productId])
}
```

### 6.2 Prisma Client Usage

```javascript
const { PrismaClient } = require('@prisma/client');

const prisma = new PrismaClient({
    log: process.env.NODE_ENV === 'development'
        ? ['query', 'info', 'warn', 'error']
        : ['error'],
    errorFormat: 'minimal'
});

// CRUD Operations
// CREATE
const user = await prisma.user.create({
    data: {
        username: 'alice',
        email: 'alice@example.com',
        profile: {
            create: {
                firstName: 'Alice',
                lastName: 'Johnson'
            }
        }
    },
    include: { profile: true }
});

// READ
const products = await prisma.product.findMany({
    where: {
        isActive: true,
        price: {
            gte: 1000,
            lte: 50000
        },
        category: {
            name: { in: ['Electronics', 'Mobile'] }
        }
    },
    select: {
        id: true,
        sku: true,
        name: true,
        price: true,
        category: { select: { name: true } }
    },
    orderBy: [
        { price: 'asc' }
    ],
    take: 20,
    skip: 0
});

// UPDATE
const updatedProduct = await prisma.product.update({
    where: { id: 1 },
    data: {
        price: 45000,
        updatedAt: new Date()
    }
});

// UPSERT
const product = await prisma.product.upsert({
    where: { sku: 'IPHONE-15' },
    update: { price: 45000, stockQuantity: { increment: 10 } },
    create: { sku: 'IPHONE-15', name: 'iPhone 15', price: 45000, stockQuantity: 10 }
});

// DELETE
await prisma.user.delete({ where: { id: userId } });
await prisma.user.deleteMany({ where: { isActive: false } });

// Aggregation
const stats = await prisma.order.aggregate({
    where: { status: 'COMPLETED' },
    _count: { id: true },
    _sum: { totalAmount: true },
    _avg: { totalAmount: true },
    _max: { totalAmount: true }
});

// GroupBy
const salesByCategory = await prisma.orderItem.groupBy({
    by: ['productId'],
    where: {
        order: { status: 'COMPLETED' }
    },
    _sum: { quantity: true },
    _count: { id: true },
    orderBy: { _sum: { quantity: 'desc' } },
    take: 10
});

// Transaction
const [order, _] = await prisma.$transaction([
    prisma.order.create({
        data: {
            customerId: 1,
            items: {
                create: [
                    { productId: 1, quantity: 2, unitPrice: 45000 }
                ]
            }
        }
    }),
    prisma.product.update({
        where: { id: 1 },
        data: { stockQuantity: { decrement: 2 } }
    })
]);

// Interactive transaction
const result = await prisma.$transaction(async (tx) => {
    const product = await tx.product.findUnique({
        where: { id: productId }
    });
    
    if (product.stockQuantity < quantity) {
        throw new Error('Insufficient stock');
    }
    
    const order = await tx.order.create({
        data: { customerId, status: 'PENDING' }
    });
    
    await tx.orderItem.create({
        data: {
            orderId: order.id,
            productId,
            quantity,
            unitPrice: product.price
        }
    });
    
    await tx.product.update({
        where: { id: productId },
        data: { stockQuantity: { decrement: quantity } }
    });
    
    return order;
});

// Raw SQL ใน Prisma
const users = await prisma.$queryRaw`
    SELECT u.*, COUNT(o.id) as order_count
    FROM users u
    LEFT JOIN orders o ON u.id = o.customer_id
    WHERE u.is_active = true
    GROUP BY u.id
    HAVING COUNT(o.id) > ${minOrders}
    ORDER BY order_count DESC
`;
```

---

## 7. Drizzle ORM (Modern)

### 7.1 Schema Definition

```typescript
// drizzle/schema.ts
import { pgTable, serial, varchar, boolean, decimal, integer, timestamp, text } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
    id: serial('id').primaryKey(),
    username: varchar('username', { length: 50 }).notNull().unique(),
    email: varchar('email', { length: 255 }).notNull().unique(),
    isActive: boolean('is_active').default(true),
    createdAt: timestamp('created_at').defaultNow()
});

export const products = pgTable('products', {
    id: serial('id').primaryKey(),
    sku: varchar('sku', { length: 50 }).notNull().unique(),
    name: varchar('name', { length: 200 }).notNull(),
    price: decimal('price', { precision: 10, scale: 2 }).notNull(),
    stockQuantity: integer('stock_quantity').default(0),
    categoryId: integer('category_id').references(() => categories.id)
});

export const categories = pgTable('categories', {
    id: serial('id').primaryKey(),
    name: varchar('name', { length: 100 }).notNull().unique(),
    description: text('description')
});

export const orders = pgTable('orders', {
    id: serial('id').primaryKey(),
    customerId: integer('customer_id').notNull().references(() => users.id),
    status: varchar('status', { length: 20 }).default('pending'),
    totalAmount: decimal('total_amount', { precision: 12, scale: 2 }).default('0'),
    createdAt: timestamp('created_at').defaultNow()
});

export const orderItems = pgTable('order_items', {
    id: serial('id').primaryKey(),
    orderId: integer('order_id').notNull().references(() => orders.id, { onDelete: 'cascade' }),
    productId: integer('product_id').notNull().references(() => products.id),
    quantity: integer('quantity').notNull(),
    unitPrice: decimal('unit_price', { precision: 10, scale: 2 }).notNull()
});

// Relations
export const usersRelations = relations(users, ({ many }) => ({
    orders: many(orders)
}));

export const ordersRelations = relations(orders, ({ one, many }) => ({
    customer: one(users, { fields: [orders.customerId], references: [users.id] }),
    items: many(orderItems)
}));
```

### 7.2 Drizzle Queries

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { eq, and, gte, lte, like, desc, asc, sql, count, sum, avg } from 'drizzle-orm';
import * as schema from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool, { schema });

// Type-safe SELECT
const users = await db.select({
    id: schema.users.id,
    username: schema.users.username,
    email: schema.users.email
})
.from(schema.users)
.where(eq(schema.users.isActive, true))
.orderBy(asc(schema.users.username))
.limit(20);

// Type inference: users is typed as { id: number, username: string, email: string }[]

// JOIN
const productsWithCategory = await db.select({
    productId: schema.products.id,
    sku: schema.products.sku,
    name: schema.products.name,
    price: schema.products.price,
    categoryName: schema.categories.name
})
.from(schema.products)
.leftJoin(schema.categories, eq(schema.products.categoryId, schema.categories.id))
.where(
    and(
        eq(schema.products.isActive, true),
        gte(schema.products.price, '1000'),
        lte(schema.products.price, '50000')
    )
)
.orderBy(asc(schema.products.price));

// Aggregation
const salesStats = await db.select({
    category: schema.categories.name,
    orderCount: count(schema.orders.id),
    totalRevenue: sum(schema.orders.totalAmount),
    avgOrder: avg(schema.orders.totalAmount)
})
.from(schema.orders)
.innerJoin(schema.orderItems, eq(schema.orders.id, schema.orderItems.orderId))
.innerJoin(schema.products, eq(schema.orderItems.productId, schema.products.id))
.innerJoin(schema.categories, eq(schema.products.categoryId, schema.categories.id))
.where(eq(schema.orders.status, 'completed'))
.groupBy(schema.categories.name)
.orderBy(desc(sql`total_revenue`));

// WITH (Relation queries)
const usersWithOrders = await db.query.users.findMany({
    where: eq(schema.users.isActive, true),
    with: {
        orders: {
            where: eq(schema.orders.status, 'completed'),
            limit: 5,
            orderBy: [desc(schema.orders.createdAt)],
            with: {
                items: {
                    with: { product: true }
                }
            }
        }
    }
});

// INSERT
const [newProduct] = await db.insert(schema.products)
    .values({
        sku: 'NEW-001',
        name: 'New Product',
        price: '9999.00',
        stockQuantity: 100
    })
    .returning();

// UPDATE
const updated = await db.update(schema.products)
    .set({ stockQuantity: sql`${schema.products.stockQuantity} - 1` })
    .where(eq(schema.products.id, productId))
    .returning({ id: schema.products.id, newStock: schema.products.stockQuantity });

// Transaction
const result = await db.transaction(async (tx) => {
    const [order] = await tx.insert(schema.orders)
        .values({ customerId, status: 'pending' })
        .returning();
    
    await tx.insert(schema.orderItems)
        .values(items.map(item => ({
            orderId: order.id,
            productId: item.productId,
            quantity: item.quantity,
            unitPrice: item.price
        })));
    
    for (const item of items) {
        await tx.update(schema.products)
            .set({ stockQuantity: sql`${schema.products.stockQuantity} - ${item.quantity}` })
            .where(eq(schema.products.id, item.productId));
    }
    
    return order;
});
```

---

## 8. Express.js + PostgreSQL REST API

### 8.1 Complete REST API

```javascript
const express = require('express');
const { Pool } = require('pg');
const { body, query, param, validationResult } = require('express-validator');
const rateLimit = require('express-rate-limit');

const app = express();
app.use(express.json());

// Connection pool
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Rate limiting
const apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100
});

// Middleware
const validateRequest = (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
        return res.status(400).json({ errors: errors.array() });
    }
    next();
};

// Helper: execute query with connection from pool
const query = (sql, params) => pool.query(sql, params);

// Transaction helper
async function withTransaction(callback) {
    const client = await pool.connect();
    try {
        await client.query('BEGIN');
        const result = await callback(client);
        await client.query('COMMIT');
        return result;
    } catch (error) {
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release();
    }
}

// ======== Products API ========

// GET /api/products
app.get('/api/products',
    apiLimiter,
    [
        query('keyword').optional().isString().trim().escape().isLength({ max: 100 }),
        query('category').optional().isInt({ min: 1 }),
        query('min_price').optional().isFloat({ min: 0 }),
        query('max_price').optional().isFloat({ min: 0 }),
        query('page').optional().isInt({ min: 1 }).default(1),
        query('per_page').optional().isInt({ min: 1, max: 100 }).default(20)
    ],
    validateRequest,
    async (req, res) => {
        try {
            const { keyword, category, min_price, max_price, page, per_page } = req.query;
            const offset = (parseInt(page) - 1) * parseInt(per_page);
            
            let whereClause = 'WHERE p.is_active = true';
            const params = [];
            let paramIdx = 1;
            
            if (keyword) {
                whereClause += ` AND p.name ILIKE $${paramIdx}`;
                params.push(`%${keyword}%`);
                paramIdx++;
            }
            
            if (category) {
                whereClause += ` AND p.category_id = $${paramIdx}`;
                params.push(parseInt(category));
                paramIdx++;
            }
            
            if (min_price) {
                whereClause += ` AND p.price >= $${paramIdx}`;
                params.push(parseFloat(min_price));
                paramIdx++;
            }
            
            if (max_price) {
                whereClause += ` AND p.price <= $${paramIdx}`;
                params.push(parseFloat(max_price));
                paramIdx++;
            }
            
            // Count
            const countResult = await query(
                `SELECT COUNT(*) as total FROM products p ${whereClause}`,
                params
            );
            const total = parseInt(countResult.rows[0].total);
            
            // Data
            const dataResult = await query(
                `SELECT p.id, p.sku, p.name, p.price, p.stock_quantity,
                        c.name as category_name
                 FROM products p
                 LEFT JOIN categories c ON p.category_id = c.id
                 ${whereClause}
                 ORDER BY p.name
                 LIMIT $${paramIdx} OFFSET $${paramIdx + 1}`,
                [...params, parseInt(per_page), offset]
            );
            
            res.json({
                items: dataResult.rows,
                pagination: {
                    page: parseInt(page),
                    per_page: parseInt(per_page),
                    total,
                    pages: Math.ceil(total / parseInt(per_page))
                }
            });
            
        } catch (error) {
            console.error('Error fetching products:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }
);

// GET /api/products/:id
app.get('/api/products/:id',
    [param('id').isInt({ min: 1 })],
    validateRequest,
    async (req, res) => {
        try {
            const { rows } = await query(
                `SELECT p.*, c.name as category_name,
                        json_agg(DISTINCT pi.image_url) FILTER (WHERE pi.image_url IS NOT NULL) as images
                 FROM products p
                 LEFT JOIN categories c ON p.category_id = c.id
                 LEFT JOIN product_images pi ON p.id = pi.product_id
                 WHERE p.id = $1
                 GROUP BY p.id, c.name`,
                [parseInt(req.params.id)]
            );
            
            if (rows.length === 0) {
                return res.status(404).json({ error: 'Product not found' });
            }
            
            res.json(rows[0]);
            
        } catch (error) {
            console.error('Error fetching product:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }
);

// POST /api/products
app.post('/api/products',
    [
        body('sku').isString().trim().notEmpty().isLength({ max: 50 }),
        body('name').isString().trim().notEmpty().isLength({ max: 200 }),
        body('price').isFloat({ min: 0 }),
        body('category_id').optional().isInt({ min: 1 }),
        body('stock_quantity').optional().isInt({ min: 0 })
    ],
    validateRequest,
    async (req, res) => {
        try {
            const { sku, name, price, category_id, stock_quantity = 0 } = req.body;
            
            const { rows } = await query(
                `INSERT INTO products (sku, name, price, category_id, stock_quantity)
                 VALUES ($1, $2, $3, $4, $5)
                 RETURNING *`,
                [sku.toUpperCase(), name, price, category_id, stock_quantity]
            );
            
            res.status(201).json(rows[0]);
            
        } catch (error) {
            if (error.code === '23505') { // Unique violation
                return res.status(409).json({ error: 'SKU already exists' });
            }
            console.error('Error creating product:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }
);

// POST /api/orders
app.post('/api/orders',
    [
        body('customer_id').isInt({ min: 1 }),
        body('items').isArray({ min: 1 }),
        body('items.*.product_id').isInt({ min: 1 }),
        body('items.*.quantity').isInt({ min: 1 })
    ],
    validateRequest,
    async (req, res) => {
        try {
            const { customer_id, items } = req.body;
            
            const order = await withTransaction(async (client) => {
                // Create order
                const { rows: [newOrder] } = await client.query(
                    'INSERT INTO orders (customer_id, status) VALUES ($1, $2) RETURNING *',
                    [customer_id, 'pending']
                );
                
                let totalAmount = 0;
                const orderItems = [];
                
                for (const item of items) {
                    // Lock and check stock
                    const { rows: [product] } = await client.query(
                        'SELECT id, price, stock_quantity FROM products WHERE id = $1 FOR UPDATE',
                        [item.product_id]
                    );
                    
                    if (!product) {
                        throw Object.assign(new Error(`Product ${item.product_id} not found`), { status: 404 });
                    }
                    
                    if (product.stock_quantity < item.quantity) {
                        throw Object.assign(
                            new Error(`Insufficient stock for product ${item.product_id}`),
                            { status: 400 }
                        );
                    }
                    
                    const itemTotal = parseFloat(product.price) * item.quantity;
                    totalAmount += itemTotal;
                    
                    // Create order item
                    const { rows: [orderItem] } = await client.query(
                        `INSERT INTO order_items (order_id, product_id, quantity, unit_price)
                         VALUES ($1, $2, $3, $4) RETURNING *`,
                        [newOrder.id, item.product_id, item.quantity, product.price]
                    );
                    orderItems.push(orderItem);
                    
                    // Reduce stock
                    await client.query(
                        'UPDATE products SET stock_quantity = stock_quantity - $1 WHERE id = $2',
                        [item.quantity, item.product_id]
                    );
                }
                
                // Update total
                await client.query(
                    'UPDATE orders SET total_amount = $1 WHERE id = $2',
                    [totalAmount, newOrder.id]
                );
                
                return { ...newOrder, total_amount: totalAmount, items: orderItems };
            });
            
            res.status(201).json(order);
            
        } catch (error) {
            if (error.status) {
                return res.status(error.status).json({ error: error.message });
            }
            console.error('Error creating order:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }
);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

---

## 9. TypeScript กับ SQL

### 9.1 Type-safe Database Access

```typescript
import { Pool, PoolClient, QueryResult } from 'pg';

// Type definitions
interface User {
    id: number;
    username: string;
    email: string;
    isActive: boolean;
    createdAt: Date;
}

interface CreateUserInput {
    username: string;
    email: string;
    password?: string;
}

interface Product {
    id: number;
    sku: string;
    name: string;
    price: number;
    stockQuantity: number;
    categoryName?: string;
}

// Type-safe query helper
async function typedQuery<T>(
    pool: Pool,
    sql: string,
    params?: unknown[]
): Promise<T[]> {
    const result: QueryResult<T> = await pool.query(sql, params);
    return result.rows;
}

// Repository with types
class UserRepository {
    constructor(private pool: Pool) {}
    
    async findAll(): Promise<User[]> {
        return typedQuery<User>(
            this.pool,
            'SELECT * FROM users ORDER BY username'
        );
    }
    
    async findById(id: number): Promise<User | null> {
        const users = await typedQuery<User>(
            this.pool,
            'SELECT * FROM users WHERE id = $1',
            [id]
        );
        return users[0] ?? null;
    }
    
    async create(data: CreateUserInput): Promise<User> {
        const users = await typedQuery<User>(
            this.pool,
            `INSERT INTO users (username, email) 
             VALUES ($1, $2) 
             RETURNING *`,
            [data.username, data.email]
        );
        return users[0];
    }
    
    async update(id: number, data: Partial<CreateUserInput>): Promise<User | null> {
        const fields = Object.keys(data);
        if (fields.length === 0) return this.findById(id);
        
        const setClause = fields.map((f, i) => `${f} = $${i + 2}`).join(', ');
        const users = await typedQuery<User>(
            this.pool,
            `UPDATE users SET ${setClause} WHERE id = $1 RETURNING *`,
            [id, ...Object.values(data)]
        );
        return users[0] ?? null;
    }
}
```

---

## แบบฝึกหัด

### ข้อที่ 1: Connection Pool Manager

**เฉลย:**
```javascript
class DatabasePool {
    constructor(config) {
        this.pool = new Pool({
            ...config,
            max: config.maxConnections || 10,
            idleTimeoutMillis: 30000,
            connectionTimeoutMillis: 2000
        });
        
        this._setupEventHandlers();
    }
    
    _setupEventHandlers() {
        this.pool.on('error', (err) => {
            console.error('Pool error:', err.message);
        });
    }
    
    async query(sql, params) {
        return this.pool.query(sql, params);
    }
    
    async transaction(callback) {
        const client = await this.pool.connect();
        try {
            await client.query('BEGIN');
            const result = await callback(client);
            await client.query('COMMIT');
            return result;
        } catch (e) {
            await client.query('ROLLBACK');
            throw e;
        } finally {
            client.release();
        }
    }
    
    async close() {
        await this.pool.end();
    }
    
    get status() {
        return {
            total: this.pool.totalCount,
            idle: this.pool.idleCount,
            waiting: this.pool.waitingCount
        };
    }
}
```

### ข้อที่ 2: Pagination Helper

**เฉลย:**
```javascript
class PaginatedQuery {
    constructor(pool) {
        this.pool = pool;
    }
    
    async paginate({ sql, countSql, params = [], page = 1, perPage = 20 }) {
        const offset = (page - 1) * perPage;
        
        const [dataResult, countResult] = await Promise.all([
            this.pool.query(`${sql} LIMIT $${params.length + 1} OFFSET $${params.length + 2}`,
                [...params, perPage, offset]),
            this.pool.query(countSql, params)
        ]);
        
        const total = parseInt(countResult.rows[0].count);
        
        return {
            data: dataResult.rows,
            meta: {
                page,
                perPage,
                total,
                pages: Math.ceil(total / perPage),
                hasNext: page * perPage < total,
                hasPrev: page > 1
            }
        };
    }
}

// ใช้งาน
const paginator = new PaginatedQuery(pool);
const result = await paginator.paginate({
    sql: 'SELECT id, name, price FROM products WHERE is_active = true ORDER BY name',
    countSql: 'SELECT COUNT(*) FROM products WHERE is_active = true',
    page: 2,
    perPage: 10
});
```

### ข้อที่ 3-10 (เฉลย ย่อ)

```javascript
// ข้อที่ 3: Knex migration สร้าง blog system
exports.up = async (knex) => {
    await knex.schema.createTable('posts', t => {
        t.increments('id');
        t.integer('author_id').references('users.id');
        t.string('title').notNullable();
        t.text('content');
        t.string('status').defaultTo('draft');
        t.timestamps(true, true);
        t.index(['status', 'created_at']);
    });
    await knex.schema.createTable('tags', t => {
        t.increments('id');
        t.string('name').unique().notNullable();
    });
    await knex.schema.createTable('post_tags', t => {
        t.integer('post_id').references('posts.id').onDelete('CASCADE');
        t.integer('tag_id').references('tags.id');
        t.primary(['post_id', 'tag_id']);
    });
};

// ข้อที่ 4: Prisma with soft delete
// ใน schema.prisma เพิ่ม deletedAt DateTime? @map("deleted_at")
// จากนั้น middleware:
prisma.$use(async (params, next) => {
    if (params.action === 'delete') {
        params.action = 'update';
        params.args.data = { deletedAt: new Date() };
    }
    if (['findMany', 'findFirst'].includes(params.action)) {
        params.args.where = { ...params.args.where, deletedAt: null };
    }
    return next(params);
});

// ข้อที่ 5: Drizzle ORM search
const results = await db.select()
    .from(products)
    .where(and(
        like(products.name, `%${keyword}%`),
        eq(products.isActive, true)
    ))
    .leftJoin(categories, eq(products.categoryId, categories.id))
    .orderBy(asc(products.price))
    .limit(20);

// ข้อที่ 6: Better-sqlite3 cache
const cache = new Map();
const getCachedData = db.transaction((key, fn) => {
    if (cache.has(key)) return cache.get(key);
    const data = fn();
    cache.set(key, data);
    return data;
});

// ข้อที่ 7: Express.js error handler
app.use((err, req, res, next) => {
    console.error(err.stack);
    if (err.code === '23505') return res.status(409).json({ error: 'Duplicate entry' });
    if (err.code === '23503') return res.status(400).json({ error: 'Foreign key violation' });
    res.status(err.status || 500).json({ error: err.message || 'Internal server error' });
});

// ข้อที่ 8: Sequelize hooks
User.addHook('beforeCreate', async (user) => {
    user.email = user.email.toLowerCase();
    if (user.password) {
        user.passwordHash = await bcrypt.hash(user.password, 10);
        delete user.password;
    }
});

// ข้อที่ 9: TypeScript generic repository
class Repository<T extends { id: number }> {
    constructor(private pool: Pool, private tableName: string) {}
    
    async findById(id: number): Promise<T | null> {
        const { rows } = await this.pool.query(
            `SELECT * FROM ${this.tableName} WHERE id = $1`, [id]
        );
        return rows[0] as T ?? null;
    }
    
    async findAll(where: Partial<T> = {}): Promise<T[]> {
        const keys = Object.keys(where);
        if (!keys.length) {
            const { rows } = await this.pool.query(`SELECT * FROM ${this.tableName}`);
            return rows as T[];
        }
        const conditions = keys.map((k, i) => `${k} = $${i+1}`).join(' AND ');
        const { rows } = await this.pool.query(
            `SELECT * FROM ${this.tableName} WHERE ${conditions}`,
            Object.values(where)
        );
        return rows as T[];
    }
}

// ข้อที่ 10: Real-time notifications ด้วย LISTEN/NOTIFY + WebSocket
const WebSocket = require('ws');
const { Pool } = require('pg');

const wss = new WebSocket.Server({ port: 8080 });
const notifyPool = new Pool({ connectionString: process.env.DATABASE_URL });

async function startNotificationBridge() {
    const client = await notifyPool.connect();
    client.on('notification', (msg) => {
        const data = JSON.parse(msg.payload);
        wss.clients.forEach((ws) => {
            if (ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({ channel: msg.channel, data }));
            }
        });
    });
    await client.query('LISTEN order_created');
    await client.query('LISTEN inventory_low');
}
startNotificationBridge();
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การใช้ SQL กับ Node.js/JavaScript ครบถ้วน:

1. **node-postgres (pg)** - PostgreSQL driver พื้นฐาน + connection pool
2. **mysql2** - MySQL driver ที่รองรับ promises
3. **better-sqlite3** - SQLite synchronous API
4. **Knex.js** - query builder ที่ยืดหยุ่น + migrations
5. **Sequelize** - ORM ดั้งเดิมที่ feature-rich
6. **Prisma** - Modern ORM with type safety
7. **Drizzle ORM** - Lightweight type-safe ORM
8. **Express.js API** - Complete REST API
9. **TypeScript** - Type-safe database access
10. **Real-time** - LISTEN/NOTIFY + WebSocket

การเลือก ORM/Driver ขึ้นอยู่กับ: Prisma สำหรับ DX ที่ดี, Drizzle สำหรับ performance, Sequelize สำหรับ feature-richness, raw SQL สำหรับ control สูงสุด
