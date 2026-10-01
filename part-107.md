# ตอนที่ 107: SQL in Web Applications

## บทนำ

การใช้ SQL ใน Web Applications เป็นเรื่องสำคัญมากที่ developer ทุกคนต้องเชี่ยวชาญ ในบทนี้เราจะศึกษาการใช้ SQL ในกรอบเวลาของ web frameworks ต่าง ๆ รวมถึงปัญหาที่พบบ่อย เช่น N+1 Problem, pagination, search implementation และ API design

---

## 1. Django ORM vs Raw SQL

### 1.1 Django ORM Basic Patterns

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
import uuid

class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)
    description = models.TextField(blank=True)
    
    class Meta:
        verbose_name_plural = 'categories'
        ordering = ['name']
    
    def __str__(self):
        return self.name

class Product(models.Model):
    sku = models.CharField(max_length=50, unique=True, db_index=True)
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock_quantity = models.PositiveIntegerField(default=0)
    category = models.ForeignKey(
        Category, 
        on_delete=models.SET_NULL, 
        null=True, 
        related_name='products'
    )
    is_active = models.BooleanField(default=True, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['category', 'is_active']),
            models.Index(fields=['price', 'is_active']),
        ]
    
    def __str__(self):
        return f"{self.sku}: {self.name}"

class Order(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('processing', 'Processing'),
        ('completed', 'Completed'),
        ('cancelled', 'Cancelled'),
    ]
    
    customer = models.ForeignKey(User, on_delete=models.PROTECT, related_name='orders')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    total_amount = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    notes = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-created_at']

class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product = models.ForeignKey(Product, on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
    
    @property
    def subtotal(self):
        return self.quantity * self.unit_price
```

### 1.2 Django ORM Queries

```python
from django.db.models import (
    Q, F, Count, Sum, Avg, Max, Min, 
    Value, CharField, DecimalField,
    ExpressionWrapper, FloatField,
    Prefetch, Subquery, OuterRef
)
from django.db.models.functions import Coalesce, TruncMonth, TruncYear
from decimal import Decimal

# views.py

# Basic filtering
active_products = Product.objects.filter(
    is_active=True,
    price__gte=1000,
    price__lte=50000,
    category__name__icontains='electronics'
).select_related('category').order_by('price')

# Complex Q objects
from django.utils import timezone
from datetime import timedelta

recent_orders = Order.objects.filter(
    Q(status='pending') | Q(status='processing'),
    created_at__gte=timezone.now() - timedelta(days=7),
    customer__is_active=True
).select_related('customer').prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.select_related('product').order_by('product__name')
    )
)

# Annotation - เพิ่ม computed fields
orders_with_stats = Order.objects.annotate(
    item_count=Count('items'),
    computed_total=Sum(
        ExpressionWrapper(
            F('items__quantity') * F('items__unit_price'),
            output_field=DecimalField()
        )
    )
).filter(
    computed_total__gt=1000
).order_by('-computed_total')

# Complex aggregation
from django.db.models import Window
from django.db.models.functions import RowNumber, Rank

# Monthly revenue by category
monthly_revenue = Order.objects.filter(
    status='completed'
).annotate(
    month=TruncMonth('created_at')
).values(
    'month',
    'items__product__category__name'
).annotate(
    revenue=Sum(
        ExpressionWrapper(
            F('items__quantity') * F('items__unit_price'),
            output_field=DecimalField()
        )
    )
).order_by('month', '-revenue')

# Using F() for atomic updates (ป้องกัน race condition)
Product.objects.filter(id=product_id).update(
    stock_quantity=F('stock_quantity') - quantity,
    updated_at=timezone.now()
)

# Subquery
latest_order_amount = Order.objects.filter(
    customer=OuterRef('pk'),
    status='completed'
).order_by('-created_at').values('total_amount')[:1]

customers_with_latest_order = User.objects.annotate(
    latest_order_amount=Subquery(latest_order_amount)
)
```

### 1.3 Django Raw SQL

```python
from django.db import connection

# Raw SQL ด้วย cursor
def get_top_selling_products(days=30, limit=10):
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT 
                p.id,
                p.sku,
                p.name,
                c.name AS category,
                SUM(oi.quantity) AS units_sold,
                SUM(oi.quantity * oi.unit_price) AS revenue
            FROM products p
            JOIN categories c ON p.category_id = c.id
            JOIN order_items oi ON p.id = oi.product_id
            JOIN orders o ON oi.order_id = o.id
            WHERE o.status = 'completed'
              AND o.created_at >= NOW() - INTERVAL '%s days'
            GROUP BY p.id, p.sku, p.name, c.name
            ORDER BY revenue DESC
            LIMIT %s
        """, [days, limit])
        
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]

# Raw queryset
products = Product.objects.raw("""
    SELECT p.*, 
           COALESCE(AVG(r.rating), 0) AS avg_rating,
           COUNT(r.id) AS review_count
    FROM products p
    LEFT JOIN product_reviews r ON p.id = r.product_id
    WHERE p.is_active = TRUE
    GROUP BY p.id
    HAVING COUNT(r.id) >= 5
    ORDER BY avg_rating DESC
""")

for product in products:
    print(f"{product.name}: {product.avg_rating:.1f} ({product.review_count} reviews)")

# Transaction ใน Django
from django.db import transaction

@transaction.atomic
def place_order(user, cart_items):
    """Place order ด้วย atomic transaction"""
    order = Order.objects.create(
        customer=user,
        status='pending'
    )
    
    total = Decimal('0')
    
    for cart_item in cart_items:
        # Lock product row
        product = Product.objects.select_for_update().get(id=cart_item.product_id)
        
        if product.stock_quantity < cart_item.quantity:
            raise ValueError(f"Insufficient stock for {product.name}")
        
        unit_price = product.price
        OrderItem.objects.create(
            order=order,
            product=product,
            quantity=cart_item.quantity,
            unit_price=unit_price
        )
        
        product.stock_quantity -= cart_item.quantity
        product.save(update_fields=['stock_quantity', 'updated_at'])
        
        total += unit_price * cart_item.quantity
    
    order.total_amount = total
    order.save(update_fields=['total_amount'])
    
    return order

# Savepoint
def complex_operation(data):
    with transaction.atomic():
        # Main operation
        create_main_record(data)
        
        # Optional operation (ไม่ต้อง rollback main ถ้า fail)
        try:
            with transaction.atomic():  # savepoint
                create_optional_record(data)
        except Exception as e:
            logger.warning(f"Optional operation failed: {e}")
            # Main transaction continues
```

---

## 2. Laravel Eloquent + Query Builder

### 2.1 Eloquent ORM

```php
<?php
// app/Models/Product.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends Model
{
    use SoftDeletes;
    
    protected $fillable = [
        'sku', 'name', 'description', 'price', 
        'stock_quantity', 'category_id', 'is_active'
    ];
    
    protected $casts = [
        'price' => 'decimal:2',
        'is_active' => 'boolean',
        'metadata' => 'array'
    ];
    
    // Relationships
    public function category()
    {
        return $this->belongsTo(Category::class);
    }
    
    public function orderItems()
    {
        return $this->hasMany(OrderItem::class);
    }
    
    // Local scopes
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('is_active', true);
    }
    
    public function scopeInPriceRange(Builder $query, float $min, float $max): Builder
    {
        return $query->whereBetween('price', [$min, $max]);
    }
    
    public function scopeByCategory(Builder $query, string $category): Builder
    {
        return $query->whereHas('category', function ($q) use ($category) {
            $q->where('slug', $category);
        });
    }
    
    // Accessors
    public function getFormattedPriceAttribute(): string
    {
        return number_format($this->price, 2) . ' THB';
    }
}

// ProductController.php
class ProductController extends Controller
{
    public function index(Request $request)
    {
        $products = Product::active()
            ->when($request->keyword, fn($q, $keyword) => 
                $q->where('name', 'LIKE', "%{$keyword}%")
                  ->orWhere('sku', 'LIKE', "%{$keyword}%")
            )
            ->when($request->category, fn($q, $cat) => 
                $q->byCategory($cat)
            )
            ->when($request->min_price, fn($q, $min) => 
                $q->where('price', '>=', $min)
            )
            ->when($request->max_price, fn($q, $max) => 
                $q->where('price', '<=', $max)
            )
            ->with(['category:id,name,slug'])
            ->orderBy($request->sort_by ?? 'name', $request->sort_dir ?? 'asc')
            ->paginate($request->per_page ?? 20);
        
        return response()->json($products);
    }
    
    public function salesReport(Request $request)
    {
        // Complex query with Query Builder
        $report = DB::table('order_items as oi')
            ->join('orders as o', 'oi.order_id', '=', 'o.id')
            ->join('products as p', 'oi.product_id', '=', 'p.id')
            ->join('categories as c', 'p.category_id', '=', 'c.id')
            ->select([
                'c.name as category',
                DB::raw('DATE_FORMAT(o.created_at, "%Y-%m") as month'),
                DB::raw('SUM(oi.quantity) as units_sold'),
                DB::raw('SUM(oi.quantity * oi.unit_price) as revenue'),
                DB::raw('COUNT(DISTINCT o.id) as order_count')
            ])
            ->where('o.status', 'completed')
            ->whereYear('o.created_at', $request->year ?? now()->year)
            ->groupBy('c.name', 'month')
            ->orderBy('month')
            ->orderByDesc('revenue')
            ->get();
        
        return response()->json($report);
    }
}
```

### 2.2 Laravel Raw SQL

```php
<?php
// Raw SQL ด้วย DB facade

// Select
$products = DB::select(
    'SELECT * FROM products WHERE price BETWEEN ? AND ? AND is_active = 1',
    [$minPrice, $maxPrice]
);

// Statement (INSERT/UPDATE/DELETE)
$affected = DB::statement(
    'UPDATE products SET stock_quantity = stock_quantity - ? WHERE id = ?',
    [$quantity, $productId]
);

// Raw with bindings
$topProducts = DB::select(DB::raw('
    SELECT p.id, p.name, 
           SUM(oi.quantity) as total_sold,
           SUM(oi.quantity * oi.unit_price) as revenue
    FROM products p
    JOIN order_items oi ON p.id = oi.product_id
    JOIN orders o ON oi.order_id = o.id
    WHERE o.status = :status
      AND o.created_at >= :since
    GROUP BY p.id, p.name
    ORDER BY revenue DESC
    LIMIT :limit
'), [
    'status' => 'completed',
    'since' => now()->subDays(30)->toDateString(),
    'limit' => 10
]);

// Transaction
DB::transaction(function () use ($userId, $items) {
    $order = Order::create([
        'customer_id' => $userId,
        'status' => 'pending'
    ]);
    
    $total = 0;
    
    foreach ($items as $item) {
        $product = Product::lockForUpdate()->findOrFail($item['product_id']);
        
        if ($product->stock_quantity < $item['quantity']) {
            throw new \Exception("Insufficient stock");
        }
        
        OrderItem::create([
            'order_id' => $order->id,
            'product_id' => $item['product_id'],
            'quantity' => $item['quantity'],
            'unit_price' => $product->price
        ]);
        
        $product->decrement('stock_quantity', $item['quantity']);
        $total += $product->price * $item['quantity'];
    }
    
    $order->update(['total_amount' => $total]);
    
    return $order;
});
```

---

## 3. SQL N+1 Problem และ Solutions

### 3.1 N+1 Problem คืออะไร

```python
# ❌ N+1 Problem ใน Django
# Query 1: ดึง orders ทั้งหมด
orders = Order.objects.filter(status='completed')[:20]

for order in orders:
    # Query 2, 3, 4... N+1: ดึง customer ของแต่ละ order
    print(f"Order {order.id} by {order.customer.username}")  # N queries เพิ่ม!

# สิ่งที่เกิดขึ้น:
# 1 query สำหรับ orders
# N queries สำหรับ customer (1 query per order)
# = N+1 queries total!

# ✅ วิธีแก้: select_related (JOIN)
orders = Order.objects.filter(
    status='completed'
).select_related(
    'customer'  # JOIN กับ users table
).order_by('-created_at')[:20]

for order in orders:
    print(f"Order {order.id} by {order.customer.username}")
# ผลลัพธ์: 1 query เท่านั้น!

# ❌ N+1 กับ many-to-many
products = Product.objects.filter(is_active=True)[:10]
for product in products:
    print(product.tags.all())  # N queries!

# ✅ prefetch_related (separate query + Python JOIN)
products = Product.objects.filter(
    is_active=True
).prefetch_related('tags')[:10]

for product in products:
    print(product.tags.all())  # ไม่มี extra queries!

# ✅ Prefetch กับ queryset ที่กำหนดเอง
from django.db.models import Prefetch

orders = Order.objects.prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.select_related('product').filter(
            product__is_active=True
        ).order_by('product__name')
    )
)[:20]

# ❌ N+1 ใน Node.js
const orders = await Order.findAll({ limit: 20 });
for (const order of orders) {
    const customer = await order.getCustomer();  // N queries!
    console.log(customer.name);
}

// ✅ Eager loading
const orders = await Order.findAll({
    include: [{
        model: Customer,
        attributes: ['id', 'name', 'email']
    }],
    limit: 20
});
```

### 3.2 DataLoader Pattern (GraphQL N+1 Solution)

```javascript
const DataLoader = require('dataloader');

// สร้าง DataLoader สำหรับ batch loading
const userLoader = new DataLoader(async (userIds) => {
    // รับ array ของ IDs และ return array ของ users ในลำดับเดียวกัน
    const users = await db.query(
        'SELECT * FROM users WHERE id = ANY($1)',
        [userIds]
    );
    
    // Map results กลับให้ตรงกับ input order
    const userMap = Object.fromEntries(users.rows.map(u => [u.id, u]));
    return userIds.map(id => userMap[id] || null);
});

const productLoader = new DataLoader(async (productIds) => {
    const products = await db.query(
        'SELECT * FROM products WHERE id = ANY($1)',
        [productIds]
    );
    
    const productMap = Object.fromEntries(products.rows.map(p => [p.id, p]));
    return productIds.map(id => productMap[id] || null);
});

// ใช้ใน GraphQL resolvers
const resolvers = {
    Order: {
        customer: (order, _, { loaders }) => {
            return loaders.user.load(order.customerId);  // Batched automatically!
        },
        items: async (order, _, { db }) => {
            return db.query('SELECT * FROM order_items WHERE order_id = $1', [order.id]);
        }
    },
    OrderItem: {
        product: (item, _, { loaders }) => {
            return loaders.product.load(item.productId);  // Batched!
        }
    }
};
```

---

## 4. Pagination ใน Web Apps

### 4.1 Offset-based Pagination

```python
# Django: offset pagination
from django.core.paginator import Paginator

def product_list(request):
    page = int(request.GET.get('page', 1))
    per_page = int(request.GET.get('per_page', 20))
    
    queryset = Product.objects.filter(is_active=True).select_related('category')
    paginator = Paginator(queryset, per_page)
    
    page_obj = paginator.get_page(page)
    
    return {
        'items': list(page_obj.object_list.values()),
        'pagination': {
            'page': page,
            'per_page': per_page,
            'total': paginator.count,
            'pages': paginator.num_pages,
            'has_next': page_obj.has_next(),
            'has_prev': page_obj.has_previous()
        }
    }

# SQL equivalent
def get_products_page(page: int, per_page: int, keyword: str = None):
    offset = (page - 1) * per_page
    
    base_query = "FROM products WHERE is_active = TRUE"
    params = []
    
    if keyword:
        base_query += " AND name ILIKE %s"
        params.append(f'%{keyword}%')
    
    count = db.execute(f"SELECT COUNT(*) {base_query}", params).fetchone()[0]
    
    items = db.execute(
        f"SELECT id, sku, name, price {base_query} ORDER BY name LIMIT %s OFFSET %s",
        params + [per_page, offset]
    ).fetchall()
    
    return {
        'items': items,
        'total': count,
        'page': page,
        'pages': (count + per_page - 1) // per_page
    }
```

### 4.2 Cursor-based Pagination (ดีกว่าสำหรับ large datasets)

```python
import base64
import json
from datetime import datetime

def encode_cursor(last_id: int, last_created_at: datetime) -> str:
    """Encode cursor เป็น base64"""
    data = {'id': last_id, 'created_at': last_created_at.isoformat()}
    return base64.b64encode(json.dumps(data).encode()).decode()

def decode_cursor(cursor: str) -> dict:
    """Decode cursor จาก base64"""
    try:
        data = json.loads(base64.b64decode(cursor).decode())
        data['created_at'] = datetime.fromisoformat(data['created_at'])
        return data
    except Exception:
        return None

def get_orders_cursor(
    cursor: str = None,
    limit: int = 20,
    direction: str = 'next'
) -> dict:
    params = [limit + 1]  # ดึงเพิ่ม 1 เพื่อตรวจสอบว่ามีหน้าต่อไป
    
    if cursor:
        decoded = decode_cursor(cursor)
        if decoded:
            where_clause = """
                WHERE (created_at < %s OR (created_at = %s AND id < %s))
            """
            params = [decoded['created_at'], decoded['created_at'], decoded['id']] + params
        else:
            where_clause = ""
    else:
        where_clause = ""
    
    query = f"""
        SELECT id, customer_id, status, total_amount, created_at
        FROM orders
        {where_clause}
        ORDER BY created_at DESC, id DESC
        LIMIT %s
    """
    
    items = db.execute(query, params).fetchall()
    
    has_more = len(items) > limit
    if has_more:
        items = items[:limit]
    
    next_cursor = None
    if has_more and items:
        last = items[-1]
        next_cursor = encode_cursor(last['id'], last['created_at'])
    
    return {
        'items': items,
        'has_more': has_more,
        'next_cursor': next_cursor
    }

# Keyset Pagination สำหรับ sorted results
def get_products_keyset(
    last_price: float = None,
    last_id: int = None,
    limit: int = 20
) -> list:
    if last_price is not None and last_id is not None:
        # ดึง records ที่มาหลัง cursor
        query = """
            SELECT id, sku, name, price
            FROM products
            WHERE is_active = TRUE
            AND (price > %s OR (price = %s AND id > %s))
            ORDER BY price ASC, id ASC
            LIMIT %s
        """
        params = [last_price, last_price, last_id, limit]
    else:
        query = """
            SELECT id, sku, name, price
            FROM products
            WHERE is_active = TRUE
            ORDER BY price ASC, id ASC
            LIMIT %s
        """
        params = [limit]
    
    return db.execute(query, params).fetchall()
```

---

## 5. Search Implementation

### 5.1 Full-Text Search ใน Django (PostgreSQL)

```python
from django.contrib.postgres.search import (
    SearchVector, SearchQuery, SearchRank, SearchHeadline
)
from django.contrib.postgres.indexes import GinIndex

# Model กับ full-text search
class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    search_vector = SearchVectorField(null=True, editable=False)
    
    class Meta:
        indexes = [
            GinIndex(fields=['search_vector'])
        ]

# อัพเดท search_vector ด้วย trigger
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.postgres.search import SearchVectorField
from django.db.models import F

@receiver(post_save, sender=Article)
def update_search_vector(sender, instance, **kwargs):
    Article.objects.filter(pk=instance.pk).update(
        search_vector=(
            SearchVector('title', weight='A', config='english') +
            SearchVector('content', weight='B', config='english')
        )
    )

# Search views
def search_articles(request):
    keyword = request.GET.get('q', '')
    
    if not keyword:
        return []
    
    search_query = SearchQuery(keyword, config='english')
    
    results = Article.objects.annotate(
        rank=SearchRank(F('search_vector'), search_query),
        headline=SearchHeadline(
            'content',
            search_query,
            start_sel='<mark>',
            stop_sel='</mark>',
            max_words=50,
            min_words=25,
            config='english'
        )
    ).filter(
        search_vector=search_query
    ).order_by('-rank').select_related('author')
    
    return results

# Similarity Search (pg_trgm)
from django.contrib.postgres.search import TrigramSimilarity

def fuzzy_search(keyword: str):
    return Product.objects.annotate(
        similarity=TrigramSimilarity('name', keyword)
    ).filter(
        similarity__gt=0.3
    ).order_by('-similarity')
```

### 5.2 Elasticsearch Integration

```python
from elasticsearch import Elasticsearch
from elasticsearch_dsl import (
    Document, Text, Keyword, Date, Float, 
    Integer, Boolean, connections, Q
)

# Connect
connections.create_connection(hosts=['localhost:9200'], timeout=20)

class ProductIndex(Document):
    sku = Keyword()
    name = Text(analyzer='standard', fields={'keyword': Keyword()})
    description = Text()
    price = Float()
    category = Keyword()
    tags = Keyword(multi=True)
    is_active = Boolean()
    
    class Index:
        name = 'products'
        settings = {
            'number_of_shards': 1,
            'number_of_replicas': 0
        }

# Index document
def index_product(product):
    doc = ProductIndex(
        meta={'id': product.id},
        sku=product.sku,
        name=product.name,
        description=product.description,
        price=float(product.price),
        category=product.category.name if product.category else '',
        is_active=product.is_active
    )
    doc.save()

# Search
def search_products_es(keyword, category=None, min_price=None, max_price=None):
    s = ProductIndex.search()
    
    # Full-text search
    if keyword:
        s = s.query('multi_match', 
                    query=keyword, 
                    fields=['name^3', 'description'],
                    fuzziness='AUTO')
    
    # Filters
    filters = [Q('term', is_active=True)]
    
    if category:
        filters.append(Q('term', category=category))
    
    if min_price or max_price:
        range_filter = {}
        if min_price: range_filter['gte'] = min_price
        if max_price: range_filter['lte'] = max_price
        filters.append(Q('range', price=range_filter))
    
    s = s.filter('bool', must=filters)
    
    # Sort
    s = s.sort('_score', {'price': 'asc'})
    
    # Pagination
    s = s[0:20]
    
    response = s.execute()
    
    return [hit.to_dict() for hit in response]
```

---

## 6. API Design สำหรับ SQL Backends

### 6.1 RESTful API Pattern

```python
# api/views.py - Django REST Framework
from rest_framework import viewsets, filters, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related('category').filter(is_active=True)
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'is_active']
    search_fields = ['name', 'sku', 'description']
    ordering_fields = ['name', 'price', 'created_at']
    ordering = ['name']
    
    def get_queryset(self):
        queryset = super().get_queryset()
        
        # Price range filter
        min_price = self.request.query_params.get('min_price')
        max_price = self.request.query_params.get('max_price')
        
        if min_price:
            queryset = queryset.filter(price__gte=float(min_price))
        if max_price:
            queryset = queryset.filter(price__lte=float(max_price))
        
        return queryset
    
    @action(detail=True, methods=['post'])
    def adjust_stock(self, request, pk=None):
        """Adjust product stock"""
        product = self.get_object()
        adjustment = request.data.get('adjustment', 0)
        
        if not isinstance(adjustment, int):
            return Response({'error': 'Adjustment must be integer'}, 
                          status=status.HTTP_400_BAD_REQUEST)
        
        with transaction.atomic():
            # Atomic update
            rows_updated = Product.objects.filter(
                id=product.id
            ).filter(
                stock_quantity__gte=-adjustment if adjustment < 0 else 0
            ).update(
                stock_quantity=F('stock_quantity') + adjustment
            )
            
            if not rows_updated:
                return Response(
                    {'error': 'Insufficient stock'},
                    status=status.HTTP_400_BAD_REQUEST
                )
            
            product.refresh_from_db()
            return Response({'stock_quantity': product.stock_quantity})
    
    @action(detail=False, methods=['get'])
    def statistics(self, request):
        """Product statistics"""
        stats = Product.objects.aggregate(
            total_products=Count('id'),
            avg_price=Avg('price'),
            total_value=Sum(F('price') * F('stock_quantity')),
            out_of_stock=Count('id', filter=Q(stock_quantity=0))
        )
        return Response(stats)
```

---

## 7. GraphQL กับ SQL Backends

### 7.1 Strawberry GraphQL + SQLAlchemy

```python
import strawberry
from strawberry.types import Info
from typing import List, Optional
from sqlalchemy.orm import Session

@strawberry.type
class ProductType:
    id: int
    sku: str
    name: str
    price: float
    stock_quantity: int
    category_name: Optional[str]

@strawberry.type
class OrderType:
    id: int
    status: str
    total_amount: float
    customer_name: str
    items: List['OrderItemType']

@strawberry.type
class OrderItemType:
    product_name: str
    quantity: int
    unit_price: float
    
    @strawberry.field
    def subtotal(self) -> float:
        return self.quantity * self.unit_price

@strawberry.type
class Query:
    @strawberry.field
    def products(
        self,
        info: Info,
        keyword: Optional[str] = None,
        category_id: Optional[int] = None,
        page: int = 1,
        per_page: int = 20
    ) -> List[ProductType]:
        db: Session = info.context['db']
        
        query = db.query(Product)
        
        if keyword:
            query = query.filter(Product.name.ilike(f'%{keyword}%'))
        if category_id:
            query = query.filter(Product.category_id == category_id)
        
        products = query.offset((page - 1) * per_page).limit(per_page).all()
        
        return [
            ProductType(
                id=p.id,
                sku=p.sku,
                name=p.name,
                price=float(p.price),
                stock_quantity=p.stock_quantity,
                category_name=p.category.name if p.category else None
            )
            for p in products
        ]
    
    @strawberry.field
    def order(self, info: Info, id: int) -> Optional[OrderType]:
        db: Session = info.context['db']
        
        order = db.query(Order).options(
            joinedload(Order.customer),
            joinedload(Order.items).joinedload(OrderItem.product)
        ).get(id)
        
        if not order:
            return None
        
        return OrderType(
            id=order.id,
            status=order.status,
            total_amount=float(order.total_amount),
            customer_name=order.customer.name,
            items=[
                OrderItemType(
                    product_name=item.product.name,
                    quantity=item.quantity,
                    unit_price=float(item.unit_price)
                )
                for item in order.items
            ]
        )

schema = strawberry.Schema(query=Query)
```

---

## 8. Reporting Queries

### 8.1 Sales Dashboard Queries

```sql
-- Dashboard metrics

-- 1. KPI summary
SELECT 
    COUNT(DISTINCT o.id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    COALESCE(SUM(o.total_amount), 0) AS total_revenue,
    COALESCE(AVG(o.total_amount), 0) AS avg_order_value,
    COUNT(DISTINCT CASE WHEN o.created_at >= NOW() - INTERVAL '7 days' THEN o.id END) AS orders_this_week
FROM orders o
WHERE o.status = 'completed'
AND o.created_at >= DATE_TRUNC('month', NOW());

-- 2. Revenue by day (last 30 days)
SELECT 
    DATE(o.created_at) AS date,
    COUNT(*) AS orders,
    SUM(o.total_amount) AS revenue
FROM orders o
WHERE o.status = 'completed'
AND o.created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE(o.created_at)
ORDER BY date;

-- 3. Top products this month
SELECT 
    p.name,
    p.sku,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    RANK() OVER (ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS revenue_rank
FROM products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o ON oi.order_id = o.id
WHERE o.status = 'completed'
AND DATE_TRUNC('month', o.created_at) = DATE_TRUNC('month', NOW())
GROUP BY p.id, p.name, p.sku
ORDER BY revenue DESC
LIMIT 10;

-- 4. Customer segments
WITH customer_stats AS (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS lifetime_value,
        MAX(created_at) AS last_order_date,
        CURRENT_DATE - MAX(created_at)::DATE AS days_since_last_order
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT 
    CASE 
        WHEN days_since_last_order <= 30 AND order_count >= 5 THEN 'VIP Active'
        WHEN days_since_last_order <= 30 THEN 'Active'
        WHEN days_since_last_order <= 90 THEN 'At Risk'
        WHEN days_since_last_order <= 180 THEN 'Lapsed'
        ELSE 'Lost'
    END AS segment,
    COUNT(*) AS customer_count,
    AVG(lifetime_value) AS avg_ltv,
    SUM(lifetime_value) AS total_ltv
FROM customer_stats
GROUP BY segment
ORDER BY total_ltv DESC;
```

---

## แบบฝึกหัด

### ข้อที่ 1: N+1 Fix ใน Django

**เฉลย:**
```python
# ❌ N+1 version
def get_order_report_bad():
    orders = Order.objects.filter(status='completed').order_by('-created_at')[:50]
    report = []
    for order in orders:
        # N+1 problems here!
        customer = order.customer  # N queries
        items = order.items.all()  # N queries
        for item in items:
            product = item.product  # N*M queries!
        report.append({'order': order, 'customer': customer})
    return report

# ✅ Optimized version
def get_order_report_good():
    orders = Order.objects.filter(
        status='completed'
    ).select_related(
        'customer'  # JOIN users
    ).prefetch_related(
        Prefetch(
            'items',
            queryset=OrderItem.objects.select_related('product', 'product__category')
        )
    ).order_by('-created_at')[:50]
    
    return [{
        'order_id': o.id,
        'customer': o.customer.username,
        'total': float(o.total_amount),
        'items': [{
            'product': item.product.name,
            'qty': item.quantity,
            'price': float(item.unit_price)
        } for item in o.items.all()]
    } for o in orders]
    # ผลลัพธ์: 3 queries แทน N*M+1 queries!
```

### ข้อที่ 2: Cursor Pagination

**เฉลย:**
```python
class CursorPaginator:
    def __init__(self, queryset, order_by='id', per_page=20):
        self.queryset = queryset
        self.order_by = order_by
        self.per_page = per_page
    
    def get_page(self, cursor=None):
        qs = self.queryset
        
        if cursor:
            last_value = self._decode_cursor(cursor)
            qs = qs.filter(**{f'{self.order_by}__gt': last_value})
        
        items = list(qs.order_by(self.order_by)[:self.per_page + 1])
        
        has_more = len(items) > self.per_page
        if has_more:
            items = items[:self.per_page]
        
        next_cursor = None
        if has_more and items:
            last_item = items[-1]
            next_cursor = self._encode_cursor(getattr(last_item, self.order_by))
        
        return {
            'items': items,
            'has_more': has_more,
            'next_cursor': next_cursor
        }
    
    def _encode_cursor(self, value):
        return base64.b64encode(str(value).encode()).decode()
    
    def _decode_cursor(self, cursor):
        return base64.b64decode(cursor).decode()
```

### ข้อที่ 3: Search Engine ด้วย PostgreSQL FTS

**เฉลย:**
```python
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank
from django.db.models import F

def advanced_product_search(request):
    keyword = request.GET.get('q', '').strip()
    
    if not keyword:
        return Product.objects.filter(is_active=True)[:20]
    
    # Full-text search
    search_query = SearchQuery(keyword, config='english')
    search_vector = SearchVector('name', weight='A') + SearchVector('description', weight='B')
    
    results = Product.objects.annotate(
        rank=SearchRank(search_vector, search_query)
    ).filter(
        rank__gte=0.1,
        is_active=True
    ).select_related('category').order_by('-rank', 'price')
    
    # Fallback: fuzzy search ถ้าไม่พบ
    if not results.exists():
        from django.contrib.postgres.search import TrigramSimilarity
        results = Product.objects.annotate(
            similarity=TrigramSimilarity('name', keyword)
        ).filter(similarity__gt=0.2, is_active=True).order_by('-similarity')
    
    return results
```

### ข้อที่ 4: Sales Dashboard API

**เฉลย:**
```python
from django.db.models import Count, Sum, Avg, F, ExpressionWrapper, DecimalField
from django.db.models.functions import TruncDay, TruncMonth
from django.utils import timezone
from datetime import timedelta

def sales_dashboard_api(request):
    today = timezone.now().date()
    month_start = today.replace(day=1)
    
    # This month stats
    monthly_stats = Order.objects.filter(
        status='completed',
        created_at__date__gte=month_start
    ).aggregate(
        orders=Count('id'),
        revenue=Sum('total_amount'),
        customers=Count('customer_id', distinct=True),
        avg_order=Avg('total_amount')
    )
    
    # Daily trend (last 30 days)
    daily_trend = Order.objects.filter(
        status='completed',
        created_at__date__gte=today - timedelta(days=30)
    ).annotate(
        day=TruncDay('created_at')
    ).values('day').annotate(
        orders=Count('id'),
        revenue=Sum('total_amount')
    ).order_by('day')
    
    # Top 5 products
    top_products = OrderItem.objects.filter(
        order__status='completed',
        order__created_at__date__gte=month_start
    ).values(
        name=F('product__name')
    ).annotate(
        units=Sum('quantity'),
        revenue=Sum(ExpressionWrapper(
            F('quantity') * F('unit_price'),
            output_field=DecimalField()
        ))
    ).order_by('-revenue')[:5]
    
    return {
        'monthly_stats': monthly_stats,
        'daily_trend': list(daily_trend),
        'top_products': list(top_products)
    }
```

### ข้อที่ 5-10 (เฉลย ย่อ)

```sql
-- ข้อที่ 5: Cohort Retention Report
WITH cohorts AS (
    SELECT customer_id,
           DATE_TRUNC('month', MIN(created_at)) AS cohort_month
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
activity AS (
    SELECT o.customer_id,
           DATE_TRUNC('month', o.created_at) AS activity_month
    FROM orders o WHERE o.status = 'completed'
    GROUP BY o.customer_id, DATE_TRUNC('month', o.created_at)
)
SELECT 
    c.cohort_month,
    EXTRACT(MONTH FROM AGE(a.activity_month, c.cohort_month)) AS month_number,
    COUNT(DISTINCT a.customer_id) AS active_users,
    COUNT(DISTINCT c.customer_id) AS cohort_size
FROM cohorts c
JOIN activity a ON c.customer_id = a.customer_id
GROUP BY c.cohort_month, month_number
ORDER BY c.cohort_month, month_number;

-- ข้อที่ 6: Customer Lifetime Value
SELECT 
    c.id,
    c.name,
    COUNT(o.id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value,
    AVG(o.total_amount) AS avg_order,
    MAX(o.created_at) AS last_order,
    CURRENT_DATE - MAX(o.created_at)::DATE AS days_inactive,
    NTILE(4) OVER (ORDER BY SUM(o.total_amount)) AS value_quartile
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.id, c.name
ORDER BY lifetime_value DESC;

-- ข้อที่ 7: Inventory Alerts
SELECT 
    p.sku, p.name, p.stock_quantity,
    COALESCE(recent.units_sold_7d, 0) AS units_sold_7d,
    CASE 
        WHEN p.stock_quantity = 0 THEN 'OUT_OF_STOCK'
        WHEN p.stock_quantity < COALESCE(recent.units_sold_7d, 0) THEN 'LOW_STOCK'
        ELSE 'OK'
    END AS status,
    CASE 
        WHEN recent.units_sold_7d > 0 
        THEN ROUND(p.stock_quantity::NUMERIC / (recent.units_sold_7d / 7.0), 1)
        ELSE NULL
    END AS days_of_stock
FROM products p
LEFT JOIN (
    SELECT product_id, SUM(quantity) AS units_sold_7d
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    WHERE o.status = 'completed'
    AND o.created_at >= NOW() - INTERVAL '7 days'
    GROUP BY product_id
) recent ON p.id = recent.product_id
WHERE p.is_active = TRUE
ORDER BY days_of_stock NULLS FIRST, p.stock_quantity;
```

```python
# ข้อที่ 8: Rate Limiting กับ database
from django.core.cache import cache
from django.http import HttpResponse
import time

def rate_limit(key_func, limit=100, window=60):
    def decorator(view_func):
        def wrapper(request, *args, **kwargs):
            key = f"ratelimit:{key_func(request)}"
            current = cache.get(key, 0)
            
            if current >= limit:
                return HttpResponse(status=429, content='Rate limit exceeded')
            
            cache.set(key, current + 1, window)
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

# ข้อที่ 9: API versioning
from django.urls import path, include

urlpatterns = [
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),
]

# ข้อที่ 10: Async Django view
import asyncio
from asgiref.sync import sync_to_async
from django.http import JsonResponse

async def async_dashboard(request):
    # Run multiple DB queries concurrently
    orders_count, revenue, top_products = await asyncio.gather(
        sync_to_async(lambda: Order.objects.filter(status='completed').count())(),
        sync_to_async(lambda: Order.objects.filter(status='completed').aggregate(Sum('total_amount'))['total_amount__sum'])(),
        sync_to_async(lambda: list(
            Product.objects.annotate(
                sales=Sum('orderitem__quantity')
            ).filter(sales__gt=0).order_by('-sales')[:5].values('name', 'sales')
        ))()
    )
    
    return JsonResponse({
        'total_orders': orders_count,
        'total_revenue': float(revenue or 0),
        'top_products': top_products
    })
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การใช้ SQL ใน Web Applications:

1. **Django ORM** - queryset API, select_related, prefetch_related
2. **Django Raw SQL** - cursor queries, atomic transactions
3. **Laravel Eloquent** - Eloquent API, Query Builder, DB facade
4. **N+1 Problem** - วิธีตรวจสอบและแก้ไข
5. **DataLoader Pattern** - batch loading สำหรับ GraphQL
6. **Pagination** - offset vs cursor-based
7. **Search** - Full-text search, trigram, Elasticsearch
8. **API Design** - RESTful patterns, GraphQL
9. **Reporting Queries** - dashboard metrics, analytics
10. **Performance** - eager loading, query optimization

Web applications ที่ดีต้องจัดการกับ database อย่างมีประสิทธิภาพ เข้าใจ N+1, ใช้ pagination ที่เหมาะสม, และออกแบบ API ให้ตอบสนองความต้องการของ clients ได้อย่างครบถ้วน
