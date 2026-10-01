# ตอนที่ 105: SQL with Python - Complete Integration

## บทนำ

Python เป็นภาษาที่นิยมมากในการทำงานกับฐานข้อมูล ด้วย ecosystem ที่หลากหลายตั้งแต่ built-in `sqlite3` ไปจนถึง ORM frameworks อย่าง SQLAlchemy และ Pydantic-based SQLModel ในบทนี้เราจะศึกษาการใช้ SQL กับ Python อย่างครบถ้วน ทั้ง raw SQL, ORMs, และ async patterns

---

## 1. sqlite3 Built-in Module

### 1.1 การใช้งานพื้นฐาน

```python
import sqlite3
from typing import Optional, List, Dict, Any

# เชื่อมต่อกับ database
# in-memory database สำหรับ testing
conn = sqlite3.connect(':memory:')

# หรือไฟล์จริง
conn = sqlite3.connect('/path/to/database.db')

# ตั้งค่า row factory
conn.row_factory = sqlite3.Row  # เข้าถึงด้วย column name

# สร้างตาราง
conn.execute('''
CREATE TABLE IF NOT EXISTS employees (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    department TEXT,
    salary REAL,
    hire_date TEXT DEFAULT (date('now'))
)
''')

# Insert ข้อมูล - parameterized query (ป้องกัน SQL injection)
conn.execute(
    "INSERT INTO employees (name, department, salary) VALUES (?, ?, ?)",
    ('Alice Johnson', 'Engineering', 95000.0)
)

# executemany สำหรับ bulk insert
employees = [
    ('Bob Smith', 'Marketing', 72000.0),
    ('Charlie Brown', 'Engineering', 88000.0),
    ('Diana Prince', 'HR', 65000.0),
    ('Eve Wilson', 'Engineering', 102000.0),
]
conn.executemany(
    "INSERT INTO employees (name, department, salary) VALUES (?, ?, ?)",
    employees
)
conn.commit()

# Query ข้อมูล
cursor = conn.execute("SELECT * FROM employees ORDER BY salary DESC")

# fetchall: ดึงทั้งหมด
all_employees = cursor.fetchall()
for emp in all_employees:
    print(f"{emp['name']}: {emp['department']} - {emp['salary']:,.0f}")

# fetchone: ดึงทีละแถว
cursor = conn.execute("SELECT * FROM employees WHERE id = ?", (1,))
emp = cursor.fetchone()
if emp:
    print(dict(emp))

# fetchmany: ดึงจำนวนที่ระบุ
cursor = conn.execute("SELECT * FROM employees ORDER BY name")
batch = cursor.fetchmany(3)  # ดึง 3 แถว

# Iterate cursor ตรง ๆ (memory efficient)
cursor = conn.execute("SELECT * FROM employees")
for row in cursor:
    print(row['name'])

conn.close()
```

### 1.2 Context Manager และ Transaction Management

```python
import sqlite3
from contextlib import contextmanager

@contextmanager
def get_connection(db_path: str):
    """Context manager ที่จัดการ connection และ transaction อัตโนมัติ"""
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA foreign_keys = ON")
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# ใช้งาน
with get_connection('myapp.db') as conn:
    conn.execute("INSERT INTO users (name) VALUES (?)", ('Test User',))
    # commit อัตโนมัติเมื่อออกจาก with block
    # rollback อัตโนมัติถ้า exception

# Manual transaction
conn = sqlite3.connect('myapp.db')
conn.isolation_level = None  # autocommit mode

# หรือจัดการ transaction เอง
conn = sqlite3.connect('myapp.db')
conn.execute("BEGIN")
try:
    conn.execute("UPDATE accounts SET balance = balance - 1000 WHERE id = 1")
    conn.execute("UPDATE accounts SET balance = balance + 1000 WHERE id = 2")
    conn.execute("COMMIT")
except Exception as e:
    conn.execute("ROLLBACK")
    raise
```

---

## 2. psycopg2 สำหรับ PostgreSQL

### 2.1 Basic Connection และ Operations

```python
import psycopg2
import psycopg2.extras
from psycopg2 import sql, OperationalError
from contextlib import contextmanager
import os

# Connection string
DATABASE_URL = os.getenv('DATABASE_URL', 
    'postgresql://user:password@localhost:5432/mydb')

# เชื่อมต่อ
conn = psycopg2.connect(DATABASE_URL)

# หรือใช้ keyword arguments
conn = psycopg2.connect(
    host='localhost',
    port=5432,
    dbname='mydb',
    user='postgres',
    password='secret',
    connect_timeout=10,
    application_name='MyApp'  # ชื่อแอปในฐานข้อมูล
)

cursor = conn.cursor()

# สร้างตาราง
cursor.execute('''
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2),
    tags TEXT[],
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
)
''')
conn.commit()

# Insert ด้วย %s placeholders (psycopg2 style)
cursor.execute(
    "INSERT INTO products (sku, name, price, tags) VALUES (%s, %s, %s, %s) RETURNING id",
    ('SKU-001', 'iPhone 15', 45000.00, ['mobile', 'apple', 'smartphone'])
)
product_id = cursor.fetchone()[0]
print(f"Created product ID: {product_id}")

# Named parameters ด้วย %(name)s
cursor.execute(
    """
    INSERT INTO products (sku, name, price, metadata)
    VALUES (%(sku)s, %(name)s, %(price)s, %(metadata)s)
    """,
    {
        'sku': 'SKU-002',
        'name': 'MacBook Pro',
        'price': 89000.00,
        'metadata': {'cpu': 'M3 Pro', 'ram': '18GB', 'storage': '512GB'}
    }
)
conn.commit()

# DictCursor: ผลลัพธ์เป็น dict
cursor = conn.cursor(cursor_factory=psycopg2.extras.DictCursor)
cursor.execute("SELECT * FROM products ORDER BY price DESC")
for row in cursor:
    print(dict(row))

# RealDictCursor: ผลลัพธ์เป็น real dict
cursor = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
cursor.execute("SELECT id, name, price FROM products")
products = cursor.fetchall()
for p in products:
    print(p['name'], p['price'])

cursor.close()
conn.close()
```

### 2.2 Connection Pool ด้วย psycopg2

```python
import psycopg2.pool
from contextlib import contextmanager
import threading

# SimpleConnectionPool: single-threaded
simple_pool = psycopg2.pool.SimpleConnectionPool(
    minconn=1,
    maxconn=10,
    dsn=DATABASE_URL
)

# ThreadedConnectionPool: multi-threaded
pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=2,
    maxconn=20,
    dsn=DATABASE_URL,
    keepalives=1,
    keepalives_idle=30,
    keepalives_interval=5,
    keepalives_count=5
)

@contextmanager
def get_db():
    """Context manager สำหรับ connection pool"""
    conn = pool.getconn()
    try:
        conn.autocommit = False
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        pool.putconn(conn)

# ใช้งานใน multi-threaded environment
def worker_function(thread_id: int):
    with get_db() as conn:
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO jobs (worker_id, started_at) VALUES (%s, NOW())",
                (thread_id,)
            )
    print(f"Worker {thread_id} done")

# Cleanup
import atexit
atexit.register(pool.closeall)
```

### 2.3 Advanced psycopg2 Features

```python
import psycopg2.extras
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT

# execute_values: fast batch insert
def bulk_insert_products(conn, products: list):
    """Batch insert หลาย rows อย่างมีประสิทธิภาพ"""
    with conn.cursor() as cur:
        psycopg2.extras.execute_values(
            cur,
            "INSERT INTO products (sku, name, price) VALUES %s",
            products,
            template="(%s, %s, %s)",
            page_size=1000  # insert ทีละ 1000 rows
        )
    conn.commit()

products_data = [
    (f'SKU-{i:05d}', f'Product {i}', i * 100.0)
    for i in range(10000)
]
bulk_insert_products(conn, products_data)

# execute_batch: batch execute SQL statements
def update_prices(conn, updates: list):
    """Batch update"""
    with conn.cursor() as cur:
        psycopg2.extras.execute_batch(
            cur,
            "UPDATE products SET price = %s WHERE id = %s",
            updates,
            page_size=100
        )
    conn.commit()

# COPY protocol: fastest bulk load
import io
import csv

def bulk_load_csv(conn, table_name: str, data: list, columns: list):
    """ใช้ COPY protocol สำหรับ ultra-fast bulk insert"""
    buffer = io.StringIO()
    writer = csv.writer(buffer)
    writer.writerows(data)
    buffer.seek(0)
    
    with conn.cursor() as cur:
        cur.copy_from(
            buffer,
            table_name,
            columns=columns,
            sep=',',
            null='\\N'
        )
    conn.commit()

# LISTEN/NOTIFY ด้วย psycopg2
def setup_listener(conn):
    conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)
    cur = conn.cursor()
    cur.execute("LISTEN order_notifications")
    
    import select
    while True:
        if select.select([conn], [], [], 5) == ([], [], []):
            print("Waiting for notifications...")
        else:
            conn.poll()
            while conn.notifies:
                notify = conn.notifies.pop(0)
                print(f"Received: channel={notify.channel}, payload={notify.payload}")

# Named Cursor สำหรับ large result sets
def stream_large_table(conn):
    with conn.cursor(name='large_query_cursor') as cur:
        cur.execute("SELECT * FROM large_table ORDER BY id")
        cur.itersize = 10000  # fetch 10000 rows per batch
        
        for row in cur:
            yield row  # generator - memory efficient
```

---

## 3. mysql-connector-python

### 3.1 Basic MySQL Connection

```python
import mysql.connector
from mysql.connector import Error, pooling
from contextlib import contextmanager
import os

# เชื่อมต่อ
config = {
    'host': 'localhost',
    'port': 3306,
    'database': 'mydb',
    'user': 'root',
    'password': os.getenv('DB_PASSWORD'),
    'charset': 'utf8mb4',
    'collation': 'utf8mb4_unicode_ci',
    'use_unicode': True,
    'autocommit': False,
    'connection_timeout': 10,
    'pool_reset_session': True
}

conn = mysql.connector.connect(**config)
cursor = conn.cursor(dictionary=True)  # return dict

# Query
cursor.execute("SELECT * FROM users WHERE active = %s", (True,))
users = cursor.fetchall()

# Prepared statement
stmt = conn.cursor(prepared=True)
stmt.execute("INSERT INTO logs (message, level) VALUES (?, ?)", ("Test", "INFO"))
conn.commit()

cursor.close()
conn.close()
```

### 3.2 MySQL Connection Pool

```python
from mysql.connector import pooling

# สร้าง connection pool
pool = mysql.connector.pooling.MySQLConnectionPool(
    pool_name="myapp_pool",
    pool_size=10,
    pool_reset_session=True,
    **config
)

@contextmanager
def get_mysql_connection():
    conn = pool.get_connection()
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()  # return to pool

# ใช้งาน
with get_mysql_connection() as conn:
    cur = conn.cursor(dictionary=True)
    cur.execute("SELECT COUNT(*) AS total FROM users")
    result = cur.fetchone()
    print(f"Total users: {result['total']}")
```

---

## 4. SQLAlchemy - Core และ ORM

### 4.1 SQLAlchemy Core

SQLAlchemy Core ใช้สำหรับ SQL expression language ที่ type-safe

```python
from sqlalchemy import (
    create_engine, Table, Column, Integer, String, 
    Numeric, DateTime, Boolean, ForeignKey, Text,
    MetaData, select, insert, update, delete,
    and_, or_, not_, func, case, literal_column
)
from sqlalchemy.dialects.postgresql import JSONB, ARRAY, UUID
from datetime import datetime

# สร้าง engine
engine = create_engine(
    'postgresql://user:password@localhost/mydb',
    echo=True,           # log SQL ทุก query
    pool_size=10,
    max_overflow=5,
    pool_timeout=30,
    pool_recycle=3600    # recycle connections ทุก 1 ชั่วโมง
)

# สำหรับ SQLite
engine = create_engine(
    'sqlite:///myapp.db',
    connect_args={'check_same_thread': False},
    echo=False
)

# สำหรับ MySQL
engine = create_engine(
    'mysql+mysqlconnector://user:pass@localhost/mydb?charset=utf8mb4',
    pool_pre_ping=True  # ตรวจสอบ connection ก่อนใช้
)

# สร้าง metadata และ tables
metadata = MetaData()

users_table = Table('users', metadata,
    Column('id', Integer, primary_key=True, autoincrement=True),
    Column('username', String(50), nullable=False, unique=True),
    Column('email', String(255), nullable=False),
    Column('password_hash', String(255)),
    Column('is_active', Boolean, default=True),
    Column('created_at', DateTime, default=datetime.utcnow)
)

products_table = Table('products', metadata,
    Column('id', Integer, primary_key=True),
    Column('sku', String(50), unique=True),
    Column('name', String(200)),
    Column('price', Numeric(10, 2)),
    Column('category_id', Integer, ForeignKey('categories.id'))
)

# สร้างตาราง
metadata.create_all(engine)

# INSERT
with engine.connect() as conn:
    # insert single row
    result = conn.execute(
        insert(users_table).values(
            username='alice',
            email='alice@example.com',
            password_hash='hashed_password'
        )
    )
    print(f"Inserted ID: {result.lastrowid}")
    
    # bulk insert
    conn.execute(
        insert(users_table),
        [
            {'username': 'bob', 'email': 'bob@example.com'},
            {'username': 'charlie', 'email': 'charlie@example.com'},
        ]
    )
    conn.commit()

# SELECT
with engine.connect() as conn:
    # simple select
    stmt = select(users_table).where(users_table.c.is_active == True)
    result = conn.execute(stmt)
    users = result.fetchall()
    
    # select specific columns
    stmt = select(
        users_table.c.id,
        users_table.c.username,
        users_table.c.email
    ).where(
        and_(
            users_table.c.is_active == True,
            users_table.c.created_at > '2024-01-01'
        )
    ).order_by(users_table.c.username)
    
    # join
    stmt = select(
        users_table.c.username,
        products_table.c.name,
        products_table.c.price
    ).join(
        products_table,
        users_table.c.id == products_table.c.created_by
    )
    
    # aggregation
    stmt = select(
        products_table.c.category_id,
        func.count(products_table.c.id).label('count'),
        func.avg(products_table.c.price).label('avg_price'),
        func.max(products_table.c.price).label('max_price')
    ).group_by(products_table.c.category_id)
    
    result = conn.execute(stmt)
    for row in result:
        print(row._mapping)  # dict-like access
```

### 4.2 SQLAlchemy ORM

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey, Boolean, DateTime, Numeric, Text
from sqlalchemy.orm import DeclarativeBase, relationship, Session, sessionmaker
from sqlalchemy.orm import validates, column_property
from datetime import datetime
from typing import Optional, List

# Base class
class Base(DeclarativeBase):
    pass

# Model definitions
class Category(Base):
    __tablename__ = 'categories'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False, unique=True)
    description = Column(Text)
    
    # Relationship
    products = relationship('Product', back_populates='category', lazy='select')
    
    def __repr__(self):
        return f"<Category(id={self.id}, name={self.name!r})>"

class Product(Base):
    __tablename__ = 'products'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    sku = Column(String(50), unique=True, nullable=False)
    name = Column(String(200), nullable=False)
    price = Column(Numeric(10, 2), nullable=False)
    description = Column(Text)
    is_active = Column(Boolean, default=True)
    stock_quantity = Column(Integer, default=0)
    category_id = Column(Integer, ForeignKey('categories.id'))
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    category = relationship('Category', back_populates='products')
    order_items = relationship('OrderItem', back_populates='product')
    
    # Validation
    @validates('price')
    def validate_price(self, key, value):
        if value < 0:
            raise ValueError("Price cannot be negative")
        return value
    
    @validates('sku')
    def validate_sku(self, key, value):
        return value.upper() if value else value
    
    def __repr__(self):
        return f"<Product(id={self.id}, sku={self.sku!r}, name={self.name!r})>"

class Customer(Base):
    __tablename__ = 'customers'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    phone = Column(String(20))
    created_at = Column(DateTime, default=datetime.utcnow)
    
    orders = relationship('Order', back_populates='customer', 
                          order_by='Order.created_at.desc()')

class Order(Base):
    __tablename__ = 'orders'
    
    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey('customers.id'), nullable=False)
    status = Column(String(20), default='pending')
    total_amount = Column(Numeric(12, 2), default=0)
    notes = Column(Text)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    customer = relationship('Customer', back_populates='orders')
    items = relationship('OrderItem', back_populates='order', cascade='all, delete-orphan')

class OrderItem(Base):
    __tablename__ = 'order_items'
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey('orders.id', ondelete='CASCADE'))
    product_id = Column(Integer, ForeignKey('products.id'))
    quantity = Column(Integer, nullable=False)
    unit_price = Column(Numeric(10, 2), nullable=False)
    
    order = relationship('Order', back_populates='items')
    product = relationship('Product', back_populates='order_items')
    
    @property
    def subtotal(self):
        return self.quantity * self.unit_price

# Engine และ Session
engine = create_engine('sqlite:///shop.db', echo=False)
Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

# Repository pattern กับ ORM
class ProductRepository:
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, **kwargs) -> Product:
        product = Product(**kwargs)
        self.session.add(product)
        self.session.flush()  # get ID without commit
        return product
    
    def get_by_id(self, product_id: int) -> Optional[Product]:
        return self.session.get(Product, product_id)
    
    def get_by_sku(self, sku: str) -> Optional[Product]:
        return self.session.query(Product).filter_by(sku=sku.upper()).first()
    
    def search(
        self, 
        keyword: str = None, 
        category_id: int = None,
        min_price: float = None,
        max_price: float = None,
        page: int = 1,
        per_page: int = 20
    ) -> tuple:
        query = self.session.query(Product)
        
        if keyword:
            query = query.filter(Product.name.ilike(f'%{keyword}%'))
        if category_id:
            query = query.filter(Product.category_id == category_id)
        if min_price is not None:
            query = query.filter(Product.price >= min_price)
        if max_price is not None:
            query = query.filter(Product.price <= max_price)
        
        total = query.count()
        products = query.offset((page - 1) * per_page).limit(per_page).all()
        
        return products, total
    
    def update(self, product_id: int, **kwargs) -> Optional[Product]:
        product = self.get_by_id(product_id)
        if product:
            for key, value in kwargs.items():
                setattr(product, key, value)
            self.session.flush()
        return product
    
    def delete(self, product_id: int) -> bool:
        product = self.get_by_id(product_id)
        if product:
            self.session.delete(product)
            return True
        return False

# ใช้งาน
with SessionLocal() as session:
    repo = ProductRepository(session)
    
    # Create
    product = repo.create(
        sku='PHONE-001',
        name='iPhone 15 Pro',
        price=45000,
        category_id=1
    )
    session.commit()
    
    # Read with relationships (eager loading)
    from sqlalchemy.orm import joinedload
    products = session.query(Product).options(
        joinedload(Product.category)
    ).filter(Product.is_active == True).all()
    
    for p in products:
        print(f"{p.name} - {p.category.name if p.category else 'No category'}")
    
    # Complex queries
    from sqlalchemy import func, desc
    
    # Top selling products
    top_products = session.query(
        Product.name,
        func.sum(OrderItem.quantity).label('total_sold'),
        func.sum(OrderItem.quantity * OrderItem.unit_price).label('total_revenue')
    ).join(
        OrderItem, Product.id == OrderItem.product_id
    ).join(
        Order, OrderItem.order_id == Order.id
    ).filter(
        Order.status == 'completed'
    ).group_by(
        Product.id, Product.name
    ).order_by(
        desc('total_revenue')
    ).limit(10).all()
    
    for name, sold, revenue in top_products:
        print(f"{name}: {sold} units, {revenue:,.0f} THB")
```

### 4.3 SQLAlchemy 2.0 Style

```python
from sqlalchemy import create_engine, select, update, delete, insert
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column
from sqlalchemy import Integer, String, Numeric, Boolean, DateTime, ForeignKey
from typing import Optional
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'
    
    # New-style annotations (SQLAlchemy 2.0)
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    email: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    
    # Optional field
    bio: Mapped[Optional[str]] = mapped_column(String(500), nullable=True)

engine = create_engine('sqlite:///app.db')
Base.metadata.create_all(engine)

# SQLAlchemy 2.0 session usage
with Session(engine) as session:
    # INSERT
    new_user = User(username='alice', email='alice@example.com')
    session.add(new_user)
    session.commit()
    session.refresh(new_user)  # reload from DB
    
    # SELECT
    stmt = select(User).where(User.is_active == True).order_by(User.username)
    users = session.execute(stmt).scalars().all()
    
    # SELECT with filter
    user = session.execute(
        select(User).where(User.username == 'alice')
    ).scalar_one_or_none()
    
    # UPDATE
    session.execute(
        update(User)
        .where(User.username == 'alice')
        .values(bio='Software Engineer')
    )
    session.commit()
    
    # DELETE
    session.execute(
        delete(User).where(User.is_active == False)
    )
    session.commit()
    
    # Bulk operations
    session.execute(
        insert(User),
        [
            {'username': 'bob', 'email': 'bob@example.com'},
            {'username': 'charlie', 'email': 'charlie@example.com'},
        ]
    )
    session.commit()
```

---

## 5. Pandas กับ SQL

### 5.1 read_sql และ to_sql

```python
import pandas as pd
from sqlalchemy import create_engine
import sqlite3

engine = create_engine('sqlite:///myapp.db')

# read_sql_query: รัน query และ return DataFrame
df = pd.read_sql_query(
    '''
    SELECT 
        strftime('%Y-%m', created_at) AS month,
        COUNT(*) AS order_count,
        SUM(total_amount) AS revenue,
        AVG(total_amount) AS avg_order
    FROM orders
    WHERE status = 'completed'
    GROUP BY month
    ORDER BY month
    ''',
    engine
)

print(df.head())
print(df.dtypes)

# read_sql_table: read entire table
df_products = pd.read_sql_table('products', engine)
print(df_products.describe())

# ใช้ parameters ป้องกัน SQL injection
df_filtered = pd.read_sql_query(
    "SELECT * FROM orders WHERE customer_id = %(cid)s AND status = %(status)s",
    engine,
    params={'cid': 101, 'status': 'completed'}
)

# chunksize: process large tables ทีละส่วน
total_revenue = 0
for chunk in pd.read_sql_query("SELECT * FROM orders", engine, chunksize=10000):
    total_revenue += chunk['total_amount'].sum()

print(f"Total revenue: {total_revenue:,.2f}")

# to_sql: เขียน DataFrame ไปยัง database
new_products = pd.DataFrame({
    'sku': ['PROD-100', 'PROD-101', 'PROD-102'],
    'name': ['Product A', 'Product B', 'Product C'],
    'price': [999.0, 1499.0, 2999.0],
    'category': ['Electronics', 'Electronics', 'Computers']
})

new_products.to_sql(
    'products',
    engine,
    if_exists='append',  # 'replace', 'append', 'fail'
    index=False,          # ไม่รวม DataFrame index
    method='multi',       # INSERT หลาย rows ทีเดียว
    chunksize=1000        # ทีละ 1000 rows
)
print("Data written to database")
```

### 5.2 Pandas กับ Complex SQL Analysis

```python
import pandas as pd
import numpy as np
from sqlalchemy import create_engine
from datetime import datetime, timedelta

engine = create_engine('postgresql://user:pass@localhost/analytics')

# ETL pipeline
def etl_daily_report(date_str: str) -> pd.DataFrame:
    """ETL สำหรับ daily report"""
    
    # Extract
    raw_data = pd.read_sql_query(
        '''
        SELECT 
            o.id AS order_id,
            o.created_at,
            c.name AS customer_name,
            c.segment AS customer_segment,
            p.category,
            oi.quantity,
            oi.unit_price,
            oi.quantity * oi.unit_price AS line_total
        FROM orders o
        JOIN customers c ON o.customer_id = c.id
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        WHERE DATE(o.created_at) = %(date)s
        AND o.status = 'completed'
        ''',
        engine,
        params={'date': date_str}
    )
    
    if raw_data.empty:
        return pd.DataFrame()
    
    # Transform
    raw_data['created_at'] = pd.to_datetime(raw_data['created_at'])
    raw_data['hour'] = raw_data['created_at'].dt.hour
    
    # Summary by category and segment
    summary = raw_data.groupby(['category', 'customer_segment']).agg(
        order_count=('order_id', 'nunique'),
        total_revenue=('line_total', 'sum'),
        avg_order_value=('line_total', 'mean'),
        items_sold=('quantity', 'sum')
    ).reset_index()
    
    # Load
    summary['report_date'] = date_str
    summary['created_at'] = datetime.now()
    
    summary.to_sql(
        'daily_category_report',
        engine,
        if_exists='append',
        index=False
    )
    
    return summary

# การวิเคราะห์ขั้นสูง
def analyze_customer_segments(engine):
    # Read data
    df = pd.read_sql_query('''
        SELECT 
            c.id,
            c.segment,
            DATE_TRUNC('month', o.created_at) AS month,
            COUNT(DISTINCT o.id) AS order_count,
            SUM(o.total_amount) AS total_spent
        FROM customers c
        JOIN orders o ON c.id = o.customer_id
        WHERE o.created_at >= NOW() - INTERVAL '12 months'
        GROUP BY c.id, c.segment, DATE_TRUNC('month', o.created_at)
    ''', engine)
    
    # Pivot table
    pivot = df.pivot_table(
        values='total_spent',
        index='segment',
        columns=pd.Grouper(key='month', freq='MS'),
        aggfunc='sum',
        fill_value=0
    )
    
    # YoY growth
    current_year = df[df['month'].dt.year == datetime.now().year]['total_spent'].sum()
    last_year = df[df['month'].dt.year == datetime.now().year - 1]['total_spent'].sum()
    
    yoy_growth = (current_year - last_year) / last_year * 100 if last_year else 0
    print(f"YoY Growth: {yoy_growth:.1f}%")
    
    return pivot
```

---

## 6. Connection Pooling

### 6.1 SQLAlchemy Connection Pool

```python
from sqlalchemy import create_engine, event
from sqlalchemy.pool import QueuePool, NullPool, StaticPool

# QueuePool (default): reuse connections
engine = create_engine(
    'postgresql://user:pass@localhost/mydb',
    poolclass=QueuePool,
    pool_size=10,        # จำนวน connections ที่เก็บไว้
    max_overflow=5,      # เพิ่มได้อีกกี่เมื่อ pool เต็ม
    pool_timeout=30,     # รอกี่วินาทีก่อน raise error
    pool_recycle=3600,   # recycle connection ทุก 1 ชั่วโมง
    pool_pre_ping=True   # ตรวจสอบ connection ก่อนใช้
)

# NullPool: ไม่ reuse connections (สำหรับ Lambda, serverless)
engine = create_engine(
    'postgresql://user:pass@localhost/mydb',
    poolclass=NullPool
)

# StaticPool: single connection (สำหรับ SQLite testing)
engine = create_engine(
    'sqlite:///:memory:',
    poolclass=StaticPool,
    connect_args={'check_same_thread': False}
)

# Monitor pool events
@event.listens_for(engine, 'checkout')
def receive_checkout(dbapi_connection, connection_record, connection_proxy):
    print(f"Connection checked out: {id(dbapi_connection)}")

@event.listens_for(engine, 'checkin')
def receive_checkin(dbapi_connection, connection_record):
    print(f"Connection returned to pool: {id(dbapi_connection)}")

# ดูสถานะ pool
pool_status = engine.pool.status()
print(pool_status)
# "Pool size: 5  Connections in pool: 3  Current Overflow: 0"
```

---

## 7. Parameterized Queries และ SQL Injection Prevention

### 7.1 SQL Injection Examples

```python
import sqlite3

conn = sqlite3.connect(':memory:')
conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, password TEXT)")
conn.execute("INSERT INTO users VALUES (1, 'admin', 'secret')")
conn.commit()

# ไม่ปลอดภัย: string concatenation
def unsafe_login(username: str, password: str) -> bool:
    """ห้ามใช้นี้ - vulnerable to SQL injection"""
    query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
    print(f"Executing: {query}")
    result = conn.execute(query).fetchone()
    return result is not None

# SQL Injection attack!
print(unsafe_login("admin' --", "anything"))  # True! bypasses password
print(unsafe_login("' OR 1=1 --", ""))        # True! returns all users

# ปลอดภัย: parameterized queries
def safe_login(username: str, password: str) -> bool:
    """ปลอดภัย - ใช้ parameterized query"""
    result = conn.execute(
        "SELECT * FROM users WHERE username = ? AND password = ?",
        (username, password)  # parameters ถูก escape อัตโนมัติ
    ).fetchone()
    return result is not None

print(safe_login("admin", "secret"))      # True
print(safe_login("admin' --", "anything"))  # False - attack ไม่ได้ผล
print(safe_login("' OR 1=1 --", ""))        # False

# ตัวอย่างกับ psycopg2
import psycopg2
from psycopg2 import sql

conn = psycopg2.connect(DATABASE_URL)
cur = conn.cursor()

# ปลอดภัย: %s placeholders
cur.execute("SELECT * FROM users WHERE username = %s", (username,))

# ปลอดภัย: named parameters
cur.execute(
    "SELECT * FROM users WHERE username = %(user)s AND status = %(status)s",
    {'user': username, 'status': 'active'}
)

# Dynamic table/column names (ต้องใช้ sql.Identifier)
table_name = 'users'
column_name = 'email'
cur.execute(
    sql.SQL("SELECT {} FROM {} WHERE id = %s").format(
        sql.Identifier(column_name),
        sql.Identifier(table_name)
    ),
    (user_id,)
)
```

### 7.2 Input Validation Layer

```python
from pydantic import BaseModel, validator, Field
from typing import Optional
import re

class UserSearchParams(BaseModel):
    keyword: Optional[str] = Field(None, max_length=100)
    min_age: Optional[int] = Field(None, ge=0, le=150)
    max_age: Optional[int] = Field(None, ge=0, le=150)
    status: Optional[str] = None
    
    @validator('keyword')
    def validate_keyword(cls, v):
        if v:
            # อนุญาตเฉพาะตัวอักษร, ตัวเลข, ช่องว่าง
            if not re.match(r'^[a-zA-Z0-9\s\-_]+$', v):
                raise ValueError('Invalid keyword characters')
        return v
    
    @validator('status')
    def validate_status(cls, v):
        allowed_statuses = {'active', 'inactive', 'pending', 'blocked'}
        if v and v not in allowed_statuses:
            raise ValueError(f'Status must be one of {allowed_statuses}')
        return v

def search_users(params: UserSearchParams, conn):
    """ค้นหา users ด้วย validated parameters"""
    conditions = ["1=1"]
    query_params = []
    
    if params.keyword:
        conditions.append("name LIKE ?")
        query_params.append(f'%{params.keyword}%')
    
    if params.min_age is not None:
        conditions.append("age >= ?")
        query_params.append(params.min_age)
    
    if params.max_age is not None:
        conditions.append("age <= ?")
        query_params.append(params.max_age)
    
    if params.status:
        conditions.append("status = ?")
        query_params.append(params.status)
    
    query = f"SELECT * FROM users WHERE {' AND '.join(conditions)}"
    return conn.execute(query, query_params).fetchall()
```

---

## 8. Transaction Management

### 8.1 ACID Transactions

```python
from sqlalchemy.orm import Session
from sqlalchemy import create_engine
import logging

engine = create_engine('sqlite:///bank.db')

def transfer_funds(
    session: Session,
    from_account_id: int,
    to_account_id: int,
    amount: float
) -> bool:
    """โอนเงินระหว่างบัญชี - ACID transaction"""
    try:
        # Lock rows for update (ป้องกัน race condition)
        from_account = session.query(Account).filter_by(
            id=from_account_id
        ).with_for_update().first()
        
        to_account = session.query(Account).filter_by(
            id=to_account_id
        ).with_for_update().first()
        
        if not from_account or not to_account:
            raise ValueError("Account not found")
        
        if from_account.balance < amount:
            raise ValueError(f"Insufficient funds. Available: {from_account.balance}")
        
        # Update balances
        from_account.balance -= amount
        to_account.balance += amount
        
        # Log transaction
        session.add(Transaction(
            from_account_id=from_account_id,
            to_account_id=to_account_id,
            amount=amount,
            type='transfer'
        ))
        
        session.commit()
        logging.info(f"Transfer successful: {amount} from {from_account_id} to {to_account_id}")
        return True
        
    except Exception as e:
        session.rollback()
        logging.error(f"Transfer failed: {e}")
        raise

# Savepoints
def complex_operation(session: Session):
    """ใช้ savepoints สำหรับ nested transactions"""
    try:
        # Main transaction
        session.execute("INSERT INTO logs VALUES (?)", ('start',))
        
        try:
            # Nested operation ที่อาจ fail
            with session.begin_nested():  # savepoint
                session.execute("UPDATE risky_table SET value = 1")
                session.execute("INSERT INTO another_table VALUES (1)")
                # ถ้า fail ที่นี่ จะ rollback แค่ savepoint
        except Exception as inner_e:
            logging.warning(f"Inner operation failed, continuing: {inner_e}")
            # Outer transaction ยังดำเนินต่อ
        
        session.execute("INSERT INTO logs VALUES (?)", ('end',))
        session.commit()
        
    except Exception as e:
        session.rollback()
        raise
```

---

## 9. Async Database Access

### 9.1 asyncpg สำหรับ PostgreSQL

```python
import asyncpg
import asyncio
from contextlib import asynccontextmanager
from typing import Optional, List

DATABASE_URL = 'postgresql://user:pass@localhost/mydb'

# สร้าง connection pool
async def create_pool():
    pool = await asyncpg.create_pool(
        DATABASE_URL,
        min_size=5,
        max_size=20,
        max_queries=50000,
        max_inactive_connection_lifetime=300.0,
        command_timeout=60
    )
    return pool

@asynccontextmanager
async def get_db_connection(pool: asyncpg.Pool):
    async with pool.acquire() as conn:
        async with conn.transaction():
            yield conn

# Repository ที่ใช้ asyncpg
class AsyncProductRepository:
    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool
    
    async def get_by_id(self, product_id: int) -> Optional[dict]:
        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT * FROM products WHERE id = $1",
                product_id
            )
            return dict(row) if row else None
    
    async def search(self, keyword: str, limit: int = 20) -> List[dict]:
        async with self.pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT id, sku, name, price, category
                FROM products
                WHERE name ILIKE $1
                ORDER BY name
                LIMIT $2
                """,
                f'%{keyword}%',
                limit
            )
            return [dict(row) for row in rows]
    
    async def create(self, **kwargs) -> int:
        async with self.pool.acquire() as conn:
            async with conn.transaction():
                row = await conn.fetchrow(
                    """
                    INSERT INTO products (sku, name, price, category_id)
                    VALUES ($1, $2, $3, $4)
                    RETURNING id
                    """,
                    kwargs['sku'],
                    kwargs['name'],
                    kwargs['price'],
                    kwargs.get('category_id')
                )
                return row['id']
    
    async def bulk_create(self, products: List[dict]) -> int:
        """Bulk insert ด้วย COPY protocol"""
        async with self.pool.acquire() as conn:
            result = await conn.copy_records_to_table(
                'products',
                records=[
                    (p['sku'], p['name'], p['price'], p.get('category_id'))
                    for p in products
                ],
                columns=['sku', 'name', 'price', 'category_id']
            )
            return int(result.split()[-1])  # "COPY 100"

# ใช้งาน async
async def main():
    pool = await create_pool()
    
    repo = AsyncProductRepository(pool)
    
    # สร้าง product
    product_id = await repo.create(
        sku='ASYNC-001',
        name='Async Product',
        price=9999.0,
        category_id=1
    )
    print(f"Created: {product_id}")
    
    # ค้นหา
    products = await repo.search('async')
    for p in products:
        print(p)
    
    # Concurrent queries
    results = await asyncio.gather(
        repo.get_by_id(1),
        repo.get_by_id(2),
        repo.search('phone'),
        return_exceptions=True
    )
    
    await pool.close()

asyncio.run(main())
```

### 9.2 SQLAlchemy Async

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import selectinload
from sqlalchemy import select

# Async engine
engine = create_async_engine(
    'postgresql+asyncpg://user:pass@localhost/mydb',
    echo=False,
    pool_size=10,
    max_overflow=5
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_user_with_orders(user_id: int):
    async with AsyncSessionLocal() as session:
        stmt = select(User).options(
            selectinload(User.orders).selectinload(Order.items)
        ).where(User.id == user_id)
        
        result = await session.execute(stmt)
        user = result.scalar_one_or_none()
        return user

async def batch_update_prices(price_increases: dict):
    """Update หลาย products พร้อมกัน"""
    async with AsyncSessionLocal() as session:
        async with session.begin():
            tasks = [
                session.execute(
                    update(Product)
                    .where(Product.id == product_id)
                    .values(price=new_price)
                )
                for product_id, new_price in price_increases.items()
            ]
            await asyncio.gather(*tasks)
```

---

## 10. Testing กับ SQL ใน Python

### 10.1 Unit Testing กับ Database

```python
import pytest
import pytest_asyncio
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# Fixtures
@pytest.fixture(scope='session')
def engine():
    """Create test engine"""
    engine = create_engine(
        'sqlite:///:memory:',
        echo=False,
        connect_args={'check_same_thread': False}
    )
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture(autouse=True)
def session(engine):
    """Create fresh session for each test with rollback"""
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()

@pytest.fixture
def sample_products(session):
    """Sample product data"""
    products = [
        Product(sku='TEST-001', name='Test Product 1', price=100.0),
        Product(sku='TEST-002', name='Test Product 2', price=200.0),
        Product(sku='TEST-003', name='Test Product 3', price=300.0),
    ]
    session.add_all(products)
    session.flush()
    return products

# Test cases
class TestProductRepository:
    def test_create_product(self, session):
        repo = ProductRepository(session)
        product = repo.create(sku='NEW-001', name='New Product', price=150.0)
        
        assert product.id is not None
        assert product.sku == 'NEW-001'
        assert product.price == 150.0
    
    def test_get_by_id(self, session, sample_products):
        repo = ProductRepository(session)
        product = repo.get_by_id(sample_products[0].id)
        
        assert product is not None
        assert product.sku == 'TEST-001'
    
    def test_search_by_keyword(self, session, sample_products):
        repo = ProductRepository(session)
        results, total = repo.search(keyword='Test Product')
        
        assert total == 3
        assert len(results) == 3
    
    def test_search_by_price_range(self, session, sample_products):
        repo = ProductRepository(session)
        results, total = repo.search(min_price=150.0, max_price=250.0)
        
        assert total == 1
        assert results[0].sku == 'TEST-002'
    
    def test_update_product(self, session, sample_products):
        repo = ProductRepository(session)
        product = repo.update(sample_products[0].id, price=999.0)
        
        assert product.price == 999.0
    
    def test_delete_product(self, session, sample_products):
        repo = ProductRepository(session)
        result = repo.delete(sample_products[0].id)
        
        assert result == True
        assert repo.get_by_id(sample_products[0].id) is None
    
    def test_negative_price_raises_error(self, session):
        with pytest.raises(ValueError, match="Price cannot be negative"):
            Product(sku='BAD-001', name='Bad Product', price=-10.0)
```

### 10.2 SQLModel (Pydantic + SQLAlchemy)

```python
from sqlmodel import SQLModel, Field, Session, create_engine, select, Relationship
from typing import Optional, List
from datetime import datetime

class ProductBase(SQLModel):
    sku: str = Field(unique=True, index=True, max_length=50)
    name: str = Field(max_length=200)
    price: float = Field(ge=0)
    is_active: bool = True

class Product(ProductBase, table=True):
    """Database model"""
    id: Optional[int] = Field(default=None, primary_key=True)
    created_at: Optional[datetime] = Field(default_factory=datetime.utcnow)
    
    # Relationship
    order_items: List["OrderItem"] = Relationship(back_populates="product")

class ProductCreate(ProductBase):
    """Input model"""
    pass

class ProductRead(ProductBase):
    """Output model"""
    id: int
    created_at: datetime

class ProductUpdate(SQLModel):
    """Update model - all optional"""
    name: Optional[str] = None
    price: Optional[float] = None
    is_active: Optional[bool] = None

# Engine
engine = create_engine('sqlite:///sqlmodel.db')
SQLModel.metadata.create_all(engine)

# CRUD operations
def create_product(product_data: ProductCreate) -> Product:
    with Session(engine) as session:
        product = Product.model_validate(product_data)
        session.add(product)
        session.commit()
        session.refresh(product)
        return product

def get_products(
    skip: int = 0,
    limit: int = 20,
    keyword: Optional[str] = None
) -> List[ProductRead]:
    with Session(engine) as session:
        statement = select(Product)
        if keyword:
            statement = statement.where(Product.name.contains(keyword))
        statement = statement.offset(skip).limit(limit)
        
        products = session.exec(statement).all()
        return products

# ใช้งาน
new_product = create_product(ProductCreate(
    sku='SM-001',
    name='SQLModel Product',
    price=5000.0
))
print(new_product)

products = get_products(keyword='SQLModel')
for p in products:
    print(ProductRead.model_validate(p))
```

---

## แบบฝึกหัด

### ข้อที่ 1: Connection Pool Manager

**เฉลย:**
```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker
from contextlib import contextmanager
import threading
import time
import logging

class DatabaseManager:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls, db_url: str, **kwargs):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._init(db_url, **kwargs)
        return cls._instance
    
    def _init(self, db_url: str, **kwargs):
        self.engine = create_engine(
            db_url,
            pool_size=kwargs.get('pool_size', 10),
            max_overflow=kwargs.get('max_overflow', 5),
            pool_timeout=kwargs.get('pool_timeout', 30),
            pool_recycle=kwargs.get('pool_recycle', 3600),
            pool_pre_ping=True,
            echo=kwargs.get('echo', False)
        )
        self.SessionFactory = sessionmaker(
            bind=self.engine,
            autoflush=False,
            autocommit=False
        )
    
    @contextmanager
    def session(self):
        session = self.SessionFactory()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
    
    def check_health(self) -> dict:
        try:
            with self.engine.connect() as conn:
                conn.execute(text("SELECT 1"))
            return {'status': 'healthy', 'pool': str(self.engine.pool.status())}
        except Exception as e:
            return {'status': 'unhealthy', 'error': str(e)}

# ใช้งาน
db = DatabaseManager('sqlite:///test.db', pool_size=5)
with db.session() as session:
    # use session
    pass

print(db.check_health())
```

### ข้อที่ 2: SQL Builder Pattern

**เฉลย:**
```python
class SQLBuilder:
    """Type-safe SQL query builder"""
    
    def __init__(self, table: str):
        self._table = table
        self._conditions = []
        self._params = []
        self._order_by = []
        self._limit = None
        self._offset = None
        self._columns = ['*']
        self._joins = []
    
    def select(self, *columns):
        self._columns = list(columns)
        return self
    
    def where(self, condition: str, *params):
        self._conditions.append(condition)
        self._params.extend(params)
        return self
    
    def join(self, table: str, on: str, join_type: str = 'INNER'):
        self._joins.append(f"{join_type} JOIN {table} ON {on}")
        return self
    
    def order_by(self, *columns):
        self._order_by.extend(columns)
        return self
    
    def limit(self, n: int):
        self._limit = n
        return self
    
    def offset(self, n: int):
        self._offset = n
        return self
    
    def build(self) -> tuple:
        query = f"SELECT {', '.join(self._columns)} FROM {self._table}"
        
        if self._joins:
            query += ' ' + ' '.join(self._joins)
        
        if self._conditions:
            query += ' WHERE ' + ' AND '.join(self._conditions)
        
        if self._order_by:
            query += ' ORDER BY ' + ', '.join(self._order_by)
        
        if self._limit:
            query += f' LIMIT {self._limit}'
        
        if self._offset:
            query += f' OFFSET {self._offset}'
        
        return query, self._params

# ใช้งาน
query, params = (
    SQLBuilder('products')
    .select('id', 'name', 'price', 'c.name AS category')
    .join('categories c', 'products.category_id = c.id')
    .where('price >= ?', 1000)
    .where('price <= ?', 50000)
    .where('products.is_active = ?', True)
    .order_by('price DESC')
    .limit(20)
    .offset(0)
    .build()
)

print(query)
print(params)

import sqlite3
conn = sqlite3.connect(':memory:')
# conn.execute(query, params)
```

### ข้อที่ 3: Async API Endpoints กับ asyncpg

**เฉลย:**
```python
import asyncpg
import asyncio
from typing import List, Optional
from dataclasses import dataclass, asdict

@dataclass
class ProductDTO:
    id: int
    sku: str
    name: str
    price: float
    category: Optional[str] = None

class AsyncProductService:
    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool
    
    async def list_products(
        self,
        category: Optional[str] = None,
        page: int = 1,
        per_page: int = 20
    ) -> dict:
        offset = (page - 1) * per_page
        
        async with self.pool.acquire() as conn:
            # Count query
            count_query = "SELECT COUNT(*) FROM products p"
            params = []
            
            if category:
                count_query += " JOIN categories c ON p.category_id = c.id WHERE c.name = $1"
                params.append(category)
            
            total = await conn.fetchval(count_query, *params)
            
            # Data query
            data_query = """
                SELECT p.id, p.sku, p.name, p.price, c.name AS category
                FROM products p
                LEFT JOIN categories c ON p.category_id = c.id
            """
            
            if category:
                data_query += " WHERE c.name = $1"
                data_query += f" ORDER BY p.name LIMIT ${len(params)+1} OFFSET ${len(params)+2}"
                params.extend([per_page, offset])
            else:
                data_query += f" ORDER BY p.name LIMIT $1 OFFSET $2"
                params = [per_page, offset]
            
            rows = await conn.fetch(data_query, *params)
            
            return {
                'items': [ProductDTO(**dict(row)) for row in rows],
                'total': total,
                'page': page,
                'per_page': per_page,
                'pages': (total + per_page - 1) // per_page
            }
    
    async def get_product(self, product_id: int) -> Optional[ProductDTO]:
        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(
                """
                SELECT p.id, p.sku, p.name, p.price, c.name AS category
                FROM products p
                LEFT JOIN categories c ON p.category_id = c.id
                WHERE p.id = $1
                """,
                product_id
            )
            return ProductDTO(**dict(row)) if row else None
    
    async def create_product(self, data: dict) -> ProductDTO:
        async with self.pool.acquire() as conn:
            async with conn.transaction():
                row = await conn.fetchrow(
                    """
                    INSERT INTO products (sku, name, price, category_id)
                    VALUES ($1, $2, $3, $4)
                    RETURNING id, sku, name, price
                    """,
                    data['sku'], data['name'], data['price'], data.get('category_id')
                )
                return ProductDTO(**dict(row))
```

### ข้อที่ 4: Data Migration Script

**เฉลย:**
```python
from sqlalchemy import create_engine, text, MetaData, Table, Column, Integer, String
from sqlalchemy.orm import Session
import logging
from tqdm import tqdm

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def migrate_data(
    source_url: str,
    target_url: str,
    batch_size: int = 1000
):
    """Migrate data จาก source database ไปยัง target"""
    source_engine = create_engine(source_url)
    target_engine = create_engine(target_url)
    
    with source_engine.connect() as source_conn:
        # Count total records
        total = source_conn.execute(text("SELECT COUNT(*) FROM old_products")).scalar()
        logger.info(f"Total records to migrate: {total}")
        
        # Process in batches
        migrated = 0
        failed = 0
        
        for offset in tqdm(range(0, total, batch_size)):
            batch = source_conn.execute(text(
                f"SELECT * FROM old_products LIMIT {batch_size} OFFSET {offset}"
            )).fetchall()
            
            if not batch:
                break
            
            # Transform data
            transformed = []
            for row in batch:
                try:
                    transformed.append({
                        'sku': row.product_code.upper(),
                        'name': row.product_name.strip(),
                        'price': float(row.price or 0),
                        'description': row.notes,
                        'is_active': bool(row.active)
                    })
                except Exception as e:
                    logger.warning(f"Failed to transform row {row.id}: {e}")
                    failed += 1
                    continue
            
            # Load batch
            with target_engine.connect() as target_conn:
                try:
                    target_conn.execute(
                        text("""
                            INSERT INTO products (sku, name, price, description, is_active)
                            VALUES (:sku, :name, :price, :description, :is_active)
                            ON CONFLICT (sku) DO UPDATE SET
                                name = EXCLUDED.name,
                                price = EXCLUDED.price
                        """),
                        transformed
                    )
                    target_conn.commit()
                    migrated += len(transformed)
                except Exception as e:
                    logger.error(f"Batch insert failed: {e}")
                    failed += len(batch)
    
    logger.info(f"Migration complete: {migrated} migrated, {failed} failed")
    return {'migrated': migrated, 'failed': failed}
```

### ข้อที่ 5-10 (เฉลย ย่อ)

```python
# ข้อที่ 5: Pandas ETL Pipeline
def etl_sales_report(engine, report_date: str):
    df = pd.read_sql_query("""
        SELECT p.category, SUM(oi.quantity * oi.unit_price) as revenue
        FROM order_items oi
        JOIN products p ON oi.product_id = p.id
        JOIN orders o ON oi.order_id = o.id
        WHERE DATE(o.created_at) = %(date)s
        GROUP BY p.category
    """, engine, params={'date': report_date})
    
    df['pct_total'] = df['revenue'] / df['revenue'].sum() * 100
    df.to_sql('daily_revenue_by_category', engine, if_exists='append', index=False)
    return df

# ข้อที่ 6: Test Factory
def create_test_db():
    engine = create_engine('sqlite:///:memory:')
    Base.metadata.create_all(engine)
    return engine

# ข้อที่ 7: Query Performance Logger
from functools import wraps
import time

def log_query_performance(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = (time.perf_counter() - start) * 1000
        if elapsed > 100:
            logger.warning(f"Slow query in {func.__name__}: {elapsed:.2f}ms")
        return result
    return wrapper

# ข้อที่ 8: Retry Pattern
import time
from functools import wraps

def retry_on_db_error(max_retries=3, delay=1.0):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt < max_retries - 1:
                        time.sleep(delay * (2 ** attempt))
                    else:
                        raise
        return wrapper
    return decorator

# ข้อที่ 9: Database Health Monitor
async def check_db_health(pool: asyncpg.Pool) -> dict:
    try:
        async with pool.acquire(timeout=5) as conn:
            await conn.fetchval("SELECT 1")
            stats = pool.get_size(), pool.get_idle_size()
        return {'healthy': True, 'pool_size': stats[0], 'idle': stats[1]}
    except Exception as e:
        return {'healthy': False, 'error': str(e)}

# ข้อที่ 10: Generic Repository
class GenericRepository:
    def __init__(self, model_class, session: Session):
        self.model = model_class
        self.session = session
    
    def get_all(self, page=1, per_page=20):
        return self.session.query(self.model).offset(
            (page-1)*per_page
        ).limit(per_page).all()
    
    def get_by_id(self, id):
        return self.session.get(self.model, id)
    
    def create(self, **kwargs):
        obj = self.model(**kwargs)
        self.session.add(obj)
        self.session.flush()
        return obj
    
    def update(self, id, **kwargs):
        obj = self.get_by_id(id)
        for k, v in kwargs.items():
            setattr(obj, k, v)
        return obj
    
    def delete(self, id):
        obj = self.get_by_id(id)
        if obj:
            self.session.delete(obj)
            return True
        return False
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การใช้ SQL กับ Python อย่างครบถ้วน:

1. **sqlite3** - built-in module สำหรับ SQLite
2. **psycopg2** - PostgreSQL driver ที่ทรงพลัง
3. **mysql-connector-python** - MySQL driver
4. **SQLAlchemy Core** - SQL expression language
5. **SQLAlchemy ORM** - object-relational mapping
6. **SQLAlchemy 2.0** - new-style annotations
7. **Pandas + SQL** - data analysis
8. **Connection Pooling** - การจัดการ connections
9. **SQL Injection Prevention** - parameterized queries
10. **Transaction Management** - ACID operations
11. **Async Access** - asyncpg และ SQLAlchemy async
12. **Testing** - pytest patterns
13. **SQLModel** - Pydantic + SQLAlchemy

การเลือกใช้เครื่องมือที่เหมาะสมกับ use case คือกุญแจสำคัญ: ใช้ raw SQL เมื่อต้องการ performance สูง, ใช้ ORM เมื่อต้องการ productivity และ maintainability, และใช้ async เมื่อต้องการ scalability สูง
