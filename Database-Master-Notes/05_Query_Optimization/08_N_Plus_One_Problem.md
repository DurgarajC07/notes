# N+1 Query Problem - The ORM Performance Killer

## 1. Concept Explanation

The **N+1 query problem** occurs when an application executes one query to fetch N records, then executes N additional queries to fetch related data for each record—resulting in N+1 total queries instead of 1 or 2.

Think of it like going to a library:
- **N+1 approach:** Get list of book IDs (1 query), then fetch each book individually (N queries) = N+1 trips
- **Efficient approach:** Get list of book IDs and all books in one go (2 queries or 1 JOIN) = 1-2 trips

### Classic N+1 Example

```python
# Fetch all users (1 query)
users = User.objects.all()  # SELECT * FROM users

# Display users with their order counts
for user in users:  # 1000 users
    print(f"{user.name}: {user.orders.count()} orders")  # 1000 queries!
    # Each iteration: SELECT COUNT(*) FROM orders WHERE user_id = ?

# Total: 1 + 1000 = 1001 queries (N+1 problem)
```

**What happens behind the scenes:**
```sql
-- Query 1: Fetch users
SELECT * FROM users;  -- Returns 1000 users

-- Query 2-1001: For each user, fetch order count
SELECT COUNT(*) FROM orders WHERE user_id = 1;
SELECT COUNT(*) FROM orders WHERE user_id = 2;
SELECT COUNT(*) FROM orders WHERE user_id = 3;
...
SELECT COUNT(*) FROM orders WHERE user_id = 1000;

-- Total: 1001 queries
-- Time: 1001 × 5ms (network latency) = 5 seconds
```

**Efficient version:**
```python
# Fetch users with order counts in one query
users = User.objects.annotate(order_count=Count('orders'))  # 1 query

for user in users:
    print(f"{user.name}: {user.order_count} orders")  # No additional queries!

# Total: 1 query
# Time: 50ms (vs 5 seconds)
# 100× FASTER!
```

---

## 2. Why It Matters

### Production Impact

**Real Scenario: E-commerce Dashboard**

```python
# Admin dashboard: Display 50 recent orders with customer + product info
orders = Order.objects.all()[:50]  # 1 query

for order in orders:
    print(f"Order {order.id}")
    print(f"Customer: {order.customer.name}")  # 50 queries (N+1 #1)
    print(f"Email: {order.customer.email}")    # Uses same query above (cached)
    print("Items:")
    for item in order.items.all():  # 50 queries (N+1 #2)
        print(f"  - {item.product.name}: {item.quantity}")  # 150 queries (N+1 #3)
        # Average 3 items per order

# Total queries:
# - 1 (orders)
# - 50 (customers)
# - 50 (order items)
# - 150 (products)
# = 251 queries

# Time at 5ms/query: 251 × 5ms = 1.25 seconds
# At 200 concurrent users: 250 seconds (4+ minutes per request!)
# Result: Site down, database overload
```

**After optimization:**
```python
# Eager load relationships
orders = Order.objects.select_related('customer').prefetch_related('items__product')[:50]

for order in orders:
    print(f"Order {order.id}")
    print(f"Customer: {order.customer.name}")  # No query, already loaded
    print(f"Email: {order.customer.email}")    # No query
    print("Items:")
    for item in order.items.all():  # No query
        print(f"  - {item.product.name}: {item.quantity}")  # No query

# Total queries:
# - 1 (orders + customer JOIN)
# - 1 (items)
# - 1 (products)
# = 3 queries ✅

# Time: 3 × 5ms + 50ms (query execution) = 65ms
# 19× FASTER (1.25s → 65ms)
```

### Database Load Impact

**N+1 queries:**
- 251 queries × 200 users = 50,200 queries/second
- Each query holds connection (connection pool exhaustion)
- Query cache thrashing (1000s of unique queries)
- Database CPU: 100% (parsing overhead)

**Optimized (3 queries):**
- 3 queries × 200 users = 600 queries/second (84× reduction)
- Predictable connection usage
- Query cache effective (3 query patterns)
- Database CPU: 10%

---

## 3. Internal Working

### How ORMs Cause N+1

**Lazy Loading (Default Behavior):**

```python
# Django ORM:
users = User.objects.all()  # Query executed: SELECT * FROM users

# Accessing relationship triggers new query:
for user in users:
    orders = user.orders.all()  # Each iteration: SELECT * FROM orders WHERE user_id = ?
```

**Behind the scenes:**
```
1. Application: Fetch users
   → ORM: SELECT * FROM users
   → DB returns: 1000 rows
   ↓
2. Application: Loop over users
   ↓
3. Application: Access user.orders (lazy load)
   → ORM: SELECT * FROM orders WHERE user_id = 1
   → DB returns: 5 rows
   ↓
4. Repeat step 3 for each user (999 more times)

Total queries: 1 + 1000 = 1001
```

### Why ORMs Default to Lazy Loading

**Trade-off:**
- **Eager loading:** Fetches all related data upfront (may load unnecessary data)
- **Lazy loading:** Fetches related data on-demand (N+1 risk)

**Example:**
```python
# If you only need user names:
users = User.objects.all()
for user in users:
    print(user.name)  # No relationships accessed

# Eager loading wastes resources:
users = User.objects.select_related('profile', 'preferences', 'orders')  # Unnecessary!
```

**ORMs prioritize convenience over performance by default:**
- Lazy = convenient (developer doesn't think about loading strategy)
- Lazy = N+1 prone (performance suffers if not careful)

---

## 4. Best Practices

### Practice 1: Use Eager Loading (select_related, prefetch_related)

**❌ Bad: Lazy Loading (N+1)**
```python
# Django:
posts = Post.objects.all()  # 1 query

for post in posts:  # 100 posts
    print(post.author.username)  # 100 queries (N+1)
    print(post.category.name)    # 100 queries (N+1)

# Total: 1 + 100 + 100 = 201 queries
```

**✅ Good: Eager Loading**
```python
# Django: Use select_related for ForeignKey/OneToOne (JOIN)
posts = Post.objects.select_related('author', 'category').all()  # 1 query with JOINs

for post in posts:
    print(post.author.username)  # No query, already loaded ✅
    print(post.category.name)    # No query, already loaded ✅

# Total: 1 query
# Query executed:
# SELECT posts.*, authors.*, categories.*
# FROM posts
# JOIN authors ON posts.author_id = authors.id
# JOIN categories ON posts.category_id = categories.id
```

**When to use select_related vs prefetch_related:**

| Relationship | Use | Reason |
|--------------|-----|--------|
| **ForeignKey** (1-to-1, many-to-1) | `select_related` | Single JOIN query |
| **Reverse ForeignKey** (1-to-many) | `prefetch_related` | Separate query (can't JOIN efficiently) |
| **ManyToMany** | `prefetch_related` | Separate query (through table) |

**Example:**
```python
# select_related (ForeignKey): 1 query with JOIN
posts = Post.objects.select_related('author')

# prefetch_related (reverse ForeignKey): 2 queries
authors = Author.objects.prefetch_related('posts')
# Query 1: SELECT * FROM authors
# Query 2: SELECT * FROM posts WHERE author_id IN (1, 2, 3, ...)

# prefetch_related (ManyToMany): 2 queries
posts = Post.objects.prefetch_related('tags')
# Query 1: SELECT * FROM posts
# Query 2: SELECT tags.*, post_tags.post_id FROM tags JOIN post_tags ON tags.id = post_tags.tag_id WHERE post_tags.post_id IN (...)
```

### Practice 2: Use Aggregation Instead of Iteration

**❌ Bad: Count in Loop (N+1)**
```python
# Django:
users = User.objects.all()  # 1 query

for user in users:  # 1000 users
    order_count = user.orders.count()  # 1000 queries!
    print(f"{user.name}: {order_count} orders")

# Total: 1001 queries
```

**✅ Good: Annotate with COUNT**
```python
# Django: Aggregate in database
users = User.objects.annotate(order_count=Count('orders'))  # 1 query

for user in users:
    print(f"{user.name}: {user.order_count} orders")  # No additional queries

# Total: 1 query
# SQL:
# SELECT users.*, COUNT(orders.id) as order_count
# FROM users
# LEFT JOIN orders ON users.id = orders.user_id
# GROUP BY users.id
```

**Other aggregations:**
```python
# Sum, average, min, max
products = Product.objects.annotate(
    total_sales=Sum('orders__quantity'),
    avg_rating=Avg('reviews__rating'),
    min_price=Min('order_items__price'),
    max_stock=Max('inventory__quantity')
)  # 1 query (complex JOIN + aggregations)

# vs N+1:
# products.count() × 4 queries = 400 queries for 100 products
```

### Practice 3: Batch Queries with IN Clause

**❌ Bad: Loop with Individual Queries**
```python
# Get order details for list of order IDs
order_ids = [1, 2, 3, ..., 100]

order_details = []
for order_id in order_ids:  # 100 iterations
    order = Order.objects.get(id=order_id)  # 100 queries
    order_details.append(order)

# Total: 100 queries
```

**✅ Good: Fetch in Bulk with IN**
```python
# Django:
order_ids = [1, 2, 3, ..., 100]
orders = Order.objects.filter(id__in=order_ids)  # 1 query

# SQL: SELECT * FROM orders WHERE id IN (1, 2, 3, ..., 100)

# Total: 1 query
```

**Real-world pattern: Cache warm-up**
```python
# Warm cache with user data
def warm_user_cache(user_ids):
    # Bad: 1000 queries
    for user_id in user_ids:
        user = User.objects.get(id=user_id)
        cache.set(f"user:{user_id}", user)
    
    # Good: 1 query
    users = User.objects.filter(id__in=user_ids)
    for user in users:
        cache.set(f"user:{user.id}", user)
```

### Practice 4: Monitor Query Counts in Development

**❌ Bad: No Query Monitoring**
```python
# Developer writes code, never sees query count
# Ships to production with 1000-query N+1 problem
```

**✅ Good: Assert Query Count in Tests**
```python
# Django test:
from django.test.utils import override_settings
from django.db import connection
from django.test import TestCase

class PerformanceTestCase(TestCase):
    def test_dashboard_query_count(self):
        # Create test data
        users = [User.objects.create(name=f"User {i}") for i in range(10)]
        
        # Assert maximum query count
        with self.assertNumQueries(3):  # Expect exactly 3 queries
            orders = Order.objects.select_related('customer').prefetch_related('items__product')[:10]
            for order in orders:
                _ = order.customer.name
                _ = [item.product.name for item in order.items.all()]
        
        # Test fails if query count exceeds 3  ✅
```

**Development Middleware:**
```python
# Django: Show query count in debug toolbar
INSTALLED_APPS = [
    'debug_toolbar',  # Shows all queries, highlights duplicates (N+1 detection)
]

# Query count in response headers:
class QueryCountMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Reset query counter
        from django.db import reset_queries
        reset_queries()
        
        response = self.get_response(request)
        
        # Add header with query count
        from django.db import connection
        response['X-Query-Count'] = len(connection.queries)
        response['X-Query-Time'] = sum(float(q['time']) for q in connection.queries)
        
        return response

# Every response shows:
# X-Query-Count: 251  ← RED FLAG for simple page!
# X-Query-Time: 1.25   ← 1.25 seconds in queries
```

---

## 5. Common Mistakes

### Mistake 1: Nested N+1 (Exponential Queries)

**Problem:**
```python
# Triple-nested N+1
users = User.objects.all()  # 1 query, 100 users

for user in users:  # 100 iterations
    orders = user.orders.all()  # 100 queries (N+1 #1)
    
    for order in orders:  # Average 10 orders/user = 1000 iterations
        items = order.items.all()  # 1000 queries (N+1 #2)
        
        for item in items:  # Average 3 items/order = 3000 iterations
            product = item.product  # 3000 queries (N+1 #3)

# Total: 1 + 100 + 1000 + 3000 = 4101 queries!
# Time: 20+ seconds
```

**Solution: Nested Eager Loading**
```python
# Django: Chain prefetch_related
users = User.objects.prefetch_related(
    'orders__items__product'  # Loads through relationships
)

for user in users:
    for order in user.orders.all():  # No query
        for item in order.items.all():  # No query
            product = item.product  # No query

# Total: 4 queries (users, orders, items, products)
# Time: 200ms (100× faster)
```

### Mistake 2: Conditional N+1 (Hard to Spot)

**Problem:**
```python
# Looks innocent, hides N+1
posts = Post.objects.all()

for post in posts:
    # Only some posts have featured images
    if post.featured_image_id:  # Check for NULL
        print(post.featured_image.url)  # N+1 for posts with images!

# 50 out of 100 posts have featured images
# Total: 1 + 50 = 51 queries
# (Hidden because not every iteration triggers query)
```

**Solution:**
```python
# Eager load even if optional
posts = Post.objects.select_related('featured_image')

for post in posts:
    if post.featured_image:  # No query, uses LEFT JOIN
        print(post.featured_image.url)

# Total: 1 query (LEFT JOIN featured_images)
```

### Mistake 3: Aggregation in Loop Instead of Database

**Problem:**
```python
# Calculate total sales per product
products = Product.objects.all()  # 1 query, 1000 products

for product in products:
    # SUM in Python (fetches all order items!)
    total_sales = sum(item.quantity * item.price for item in product.order_items.all())  # 1000 queries
    print(f"{product.name}: ${total_sales}")

# Total: 1 + 1000 = 1001 queries
# Fetches millions of order_items into memory!
```

**Solution:**
```python
# Aggregate in database
from django.db.models import Sum, F

products = Product.objects.annotate(
    total_sales=Sum(F('order_items__quantity') * F('order_items__price'))
)

for product in products:
    print(f"{product.name}: ${product.total_sales}")  # No additional queries

# Total: 1 query with GROUP BY
# Database does the aggregation (efficient)
```

### Mistake 4: Ignoring N+1 in Background Jobs

**Problem:**
```python
# Celery task: Send email to all users with pending orders
@celery_task
def send_pending_order_emails():
    users = User.objects.all()  # 10K users
    
    for user in users:
        pending_orders = user.orders.filter(status='pending')  # 10K queries!
        if pending_orders.exists():
            send_email(user.email, pending_orders)

# Background job takes 5 minutes (should be 10 seconds)
# Database load spike every hour
```

**Solution:**
```python
@celery_task
def send_pending_order_emails():
    # Fetch users with pending orders in one query
    users_with_pending = User.objects.filter(
        orders__status='pending'
    ).prefetch_related(
        Prefetch('orders', queryset=Order.objects.filter(status='pending'))
    ).distinct()
    
    for user in users_with_pending:
        pending_orders = user.orders.all()  # Already loaded, no query
        send_email(user.email, pending_orders)

# Total: 2 queries (users + orders)
# Time: 10 seconds (vs 5 minutes)
```

---

## 6. Security Considerations

### 1. N+1 as DoS Vector

**Risk:**
```python
# Public API: Fetch posts with comments
# No rate limiting, N+1 vulnerability

GET /api/posts?include=comments,author

# Backend code (vulnerable):
posts = Post.objects.all()  # Fetches all posts (attacker controls count)

for post in posts:
    include_comments = request.GET.get('include') == 'comments'
    if include_comments:
        comments = post.comments.all()  # N queries

# Attacker creates 10,000 posts, requests with include=comments
# 1 + 10,000 = 10,001 queries → Database overload
```

**Mitigation:**
```python
# Limit results, eager load
POSTS_PER_PAGE = 100

posts = Post.objects.all()[:POSTS_PER_PAGE]  # Limit to 100

if 'comments' in request.GET.get('include', ''):
    posts = posts.prefetch_related('comments')  # Eager load

# Max queries: 1 + 1 = 2 (regardless of attacker's posts)
```

### 2. Data Exposure via Over-Eager Loading

**Risk:**
```python
# Over-eager loading exposes sensitive data

# API endpoint: /users
def get_users(request):
    users = User.objects.select_related('profile', 'payment_info', 'orders')  # Over-eager!
    return jsonify([u.to_dict() for u in users])

# to_dict() serializes all loaded data:
# - profile: OK (name, bio)
# - payment_info: LEAK! (credit card, SSN)
# - orders: OK (public purchases)
```

**Solution:**
```python
# Only load what's exposed
def get_users(request):
    users = User.objects.select_related('profile')  # Only public data
    # Don't load payment_info unless needed
    
    return jsonify([{
        'id': u.id,
        'name': u.name,
        'bio': u.profile.bio  # Safe: only loaded what's needed
    } for u in users])
```

---

## 7. Performance Optimization

### Optimization 1: Prefetch with Filtering

**Problem: Prefetch Loads Too Much**
```python
# Load posts with comments (but only approved comments)
posts = Post.objects.prefetch_related('comments')  # Loads ALL comments (including spam)

for post in posts:
    approved_comments = [c for c in post.comments.all() if c.status == 'approved']  # Filter in Python (slow)
```

**Solution: Prefetch with Filter**
```python
from django.db.models import Prefetch

# Prefetch only approved comments
posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(status='approved').order_by('-created_at')
    )
)

for post in posts:
    approved_comments = post.comments.all()  # Already filtered in database ✅

# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM comments WHERE post_id IN (...) AND status = 'approved' ORDER BY created_at DESC

# Benefits:
# - Less data transferred (no spam comments)
# - No Python-side filtering
# - Maintains eager loading (no N+1)
```

### Optimization 2: Defer/Only for Large Columns

**Problem: Loading Unnecessary Large Columns**
```python
# List view: Only need id, title (not body, which is 50KB per post)
posts = Post.objects.all()  # Loads id, title, body, metadata, ...

for post in posts[:100]:
    print(post.title)  # Only uses title, but body was loaded (5MB wasted)
```

**Solution: defer/only**
```python
# Django: Only load needed columns
posts = Post.objects.only('id', 'title').all()

for post in posts[:100]:
    print(post.title)  # No query
    # Accessing post.body would trigger additional query (lazy load)

# Or: Defer large columns
posts = Post.objects.defer('body', 'metadata').all()

# Query: SELECT id, title, created_at, author_id FROM posts (excludes body, metadata)
# Result: 100× less data transferred
```

**Combined with eager loading:**
```python
# List + author, but defer large columns
posts = Post.objects.select_related('author').only('id', 'title', 'author__username').all()

# Query: SELECT posts.id, posts.title, authors.username
#        FROM posts JOIN authors ON posts.author_id = authors.id
```

### Optimization 3: Batch Processing with Chunking

**Problem: Loading All Records into Memory**
```python
# Process 1M users (loads all into memory: 10GB!)
users = User.objects.all()

for user in users:
    process_user(user)

# Memory: 10GB (OOM risk)
# Database: 1 large query (holds connection for minutes)
```

**Solution: Iterator with Chunking**
```python
# Django: Iterate in chunks
CHUNK_SIZE = 1000

for user in User.objects.iterator(chunk_size=CHUNK_SIZE):
    process_user(user)

# Queries: 1000 chunks (1K users each)
# Memory: Constant (1K users at a time: 10MB)
# Database: Connection released between chunks
```

**Chunking with eager loading:**
```python
# Process users with orders in chunks
from django.db.models import Prefetch

user_queryset = User.objects.prefetch_related(
    Prefetch('orders', queryset=Order.objects.filter(status='pending'))
)

for user in user_queryset.iterator(chunk_size=1000):
    for order in user.orders.all():  # No N+1 within chunk
        process_order(order)

# Each chunk: 2 queries (users, orders)
# Total: 2 × 1000 chunks = 2000 queries (vs 1M N+1 queries)
```

---

## 8. Examples

### Example 1: GraphQL N+1 (Classic Problem)

**Scenario:** GraphQL API with nested resolvers

**Problem:**
```python
# GraphQL schema:
type User {
    id: ID!
    name: String!
    posts: [Post!]!
}

type Post {
    id: ID!
    title: String!
    author: User!
    comments: [Comment!]!
}

# Resolver (naive implementation):
class UserType:
    def resolve_posts(self, info):
        return Post.objects.filter(author=self)  # N+1!

class PostType:
    def resolve_comments(self, info):
        return Comment.objects.filter(post=self)  # N+1!

# Query:
query {
  users {           # 1 query
    name
    posts {         # N queries (100 users)
      title
      comments {    # N×M queries (100 users × 10 posts = 1000 queries)
        text
      }
    }
  }
}

# Total: 1 + 100 + 1000 = 1101 queries ❌
```

**Solution: DataLoader Pattern**
```python
from promise import Promise
from promise.dataloader import DataLoader

# Batch loader
class PostLoader(DataLoader):
    def batch_load_fn(self, user_ids):
        # Fetch all posts for given users in ONE query
        posts = Post.objects.filter(author_id__in=user_ids).select_related('author')
        
        # Group by user_id
        posts_by_user = defaultdict(list)
        for post in posts:
            posts_by_user[post.author_id].append(post)
        
        # Return in same order as user_ids
        return Promise.resolve([posts_by_user.get(uid, []) for uid in user_ids])

class CommentLoader(DataLoader):
    def batch_load_fn(self, post_ids):
        comments = Comment.objects.filter(post_id__in=post_ids)
        comments_by_post = defaultdict(list)
        for comment in comments:
            comments_by_post[comment.post_id].append(comment)
        return Promise.resolve([comments_by_post.get(pid, []) for pid in post_ids])

# Resolver (using dataloaders):
class UserType:
    def resolve_posts(self, info):
        return info.context['post_loader'].load(self.id)  # Batched!

class PostType:
    def resolve_comments(self, info):
        return info.context['comment_loader'].load(self.id)  # Batched!

# Same query:
# Total: 3 queries (users, posts, comments) ✅
# 367× faster (1101 → 3 queries)
```

---

### Example 2: ORM Relationship Chain N+1

**Scenario:** E-commerce order fulfillment report

**Problem:**
```python
# Report: Orders → OrderItems → Products → Suppliers
# For each order, show product supplier

orders = Order.objects.filter(status='processing')[:100]  # 1 query

for order in orders:
    print(f"Order {order.id}:")
    
    for item in order.items.all():  # 100 queries (N+1 #1)
        product = item.product  # 500 queries (N+1 #2, avg 5 items/order)
        supplier = product.supplier  # 500 queries (N+1 #3)
        print(f"  - {product.name} from {supplier.name}")

# Total: 1 + 100 + 500 + 500 = 1101 queries
# Time: 5.5 seconds
```

**Solution: Chained Eager Loading**
```python
# Eager load entire chain
orders = Order.objects.filter(status='processing').prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.select_related('product__supplier')
        # select_related for FK: product and product.supplier (single JOIN)
    )
)[:100]

for order in orders:
    print(f"Order {order.id}:")
    
    for item in order.items.all():  # No query
        product = item.product  # No query
        supplier = product.supplier  # No query
        print(f"  - {product.name} from {supplier.name}")

# Total: 2 queries
# Query 1: SELECT * FROM orders WHERE status='processing' LIMIT 100
# Query 2: SELECT order_items.*, products.*, suppliers.*
#          FROM order_items
#          JOIN products ON order_items.product_id = products.id
#          JOIN suppliers ON products.supplier_id = suppliers.id
#          WHERE order_items.order_id IN (...)

# Time: 0.15 seconds (37× faster)
```

---

## 9. Real-World Use Cases

### Use Case 1: REST API Pagination with Relationships

**Scenario:** API endpoint `/api/posts?page=1&limit=20`

**Implementation:**
```python
# Django REST Framework serializer (vulnerable to N+1):
class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer()  # Nested serializer
    comments_count = serializers.SerializerMethodField()
    
    def get_comments_count(self, obj):
        return obj.comments.count()  # N+1 if not prefetched
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'comments_count']

# View:
class PostListView(ListAPIView):
    queryset = Post.objects.all()  # N+1 problem!
    serializer_class = PostSerializer

# Request: GET /api/posts?page=1&limit=20
# Queries:
# - 1: Get 20 posts
# - 20: Get author for each post
# - 20: Count comments for each post
# Total: 41 queries
```

**Optimized:**
```python
# View with eager loading:
class PostListView(ListAPIView):
    queryset = Post.objects.select_related('author').annotate(comments_count=Count('comments'))
    serializer_class = PostSerializer

# Serializer (use annotated field):
class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer()
    comments_count = serializers.IntegerField(read_only=True)  # Uses annotated value
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'comments_count']

# Request: GET /api/posts?page=1&limit=20
# Queries: 1 (JOIN author, COUNT comments in single query)
# SQL:
# SELECT posts.*, authors.*, COUNT(comments.id) as comments_count
# FROM posts
# JOIN authors ON posts.author_id = authors.id
# LEFT JOIN comments ON posts.id = comments.post_id
# GROUP BY posts.id
# LIMIT 20 OFFSET 0
```

---

## 10. Interview Questions

### Q1: How would you detect N+1 query problems in a production application?

**Staff Answer:**

```
**Detection strategies:**

**1. Application Performance Monitoring (APM)**
```python
# NewRelic, Datadog, Scout APM automatically detect N+1

# NewRelic example alert:
# "Query called 1000 times in one transaction:
#  SELECT * FROM orders WHERE user_id = ?"

# Datadog APM shows:
# - Query count per request (spike indicates N+1)
# - Duplicate queries (same query with different params)
# - Query time as % of request time (80% = query-heavy)
```

**2. Database Slow Query Log**
```sql
-- PostgreSQL: Enable query logging
ALTER SYSTEM SET log_min_duration_statement = 100;  -- Log queries >100ms
SELECT pg_reload_conf();

-- Analyze logs:
-- Look for patterns like:
-- SELECT * FROM orders WHERE user_id = 1;  (100ms)
-- SELECT * FROM orders WHERE user_id = 2;  (100ms)
-- SELECT * FROM orders WHERE user_id = 3;  (100ms)
-- ... (repeated 1000 times)

-- Indicator: Same query structure, different parameters, high frequency
```

**3. ORM Query Counter Middleware**
```python
# Django middleware to log high query counts:
class QueryCounterMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        from django.db import reset_queries, connection
        reset_queries()
        
        response = self.get_response(request)
        
        query_count = len(connection.queries)
        query_time = sum(float(q['time']) for q in connection.queries)
        
        # Alert if high query count
        if query_count > 50:
            logger.warning(
                f"High query count: {query_count} queries in {request.path}",
                extra={'queries': connection.queries}
            )
        
        # Add response headers for monitoring
        response['X-Query-Count'] = query_count
        response['X-Query-Time'] = f'{query_time:.3f}'
        
        return response

# Monitoring: Alert if any endpoint has >50 queries
```

**4. Database Query Analysis Tools**
```sql
-- PostgreSQL: pg_stat_statements
SELECT
    query,
    calls,
    mean_exec_time,
    stddev_exec_time
FROM pg_stat_statements
WHERE calls > 1000  -- Query called frequently
ORDER BY calls DESC;

-- Look for:
-- - High call count with low execution time (repeated simple queries)
-- - Similar queries differing only by parameter (N+1 pattern)

-- Example N+1 detection:
-- Query: SELECT * FROM orders WHERE user_id = $1
-- Calls: 10,000
-- Mean time: 2ms
-- Pattern: Same query, different user_id each time → N+1!
```

**5. Load Testing with Query Monitoring**
```python
# Load test that monitors query count:
import locust
from django.db import connection, reset_queries

class UserBehavior(locust.HttpUser):
    @locust.task
    def view_dashboard(self):
        reset_queries()
        
        response = self.client.get('/dashboard')
        
        query_count = int(response.headers.get('X-Query-Count', 0))
        
        # Assert reasonable query count
        if query_count > 20:
            print(f"WARNING: Dashboard generated {query_count} queries (N+1?)")

# Run load test, check for query count spikes
```

**Real-world example:**

"At my previous company, we noticed API response time increased 5× after adding a 'related posts' feature. APM showed 200 queries per request (was 5 before). Investigation revealed N+1: for each post, we fetched related posts one by one. Fixed with .prefetch_related(), query count dropped to 3, response time back to normal. Now we have CI test that fails if query count exceeds threshold."
```

### Q2: Explain the difference between select_related and prefetch_related in Django.

**Senior Answer:**

```
**Fundamental difference: JOIN vs separate IN query**

**select_related (SQL JOIN):**

```python
# Use for: ForeignKey, OneToOneField (single-valued relationships)
posts = Post.objects.select_related('author', 'category')

# SQL generated:
SELECT posts.*, authors.*, categories.*
FROM posts
INNER JOIN authors ON posts.author_id = authors.id
INNER JOIN categories ON posts.category_id = categories.id;

# Result: 1 query, all data in single result set

# Access (no additional queries):
for post in posts:
    print(post.author.name)     # Already loaded
    print(post.category.name)   # Already loaded
```

**prefetch_related (Separate IN query):**

```python
# Use for: Reverse ForeignKey, ManyToManyField (multi-valued relationships)
authors = Author.objects.prefetch_related('posts')

# SQL generated (2 queries):
# Query 1:
SELECT * FROM authors;

# Query 2:
SELECT * FROM posts WHERE author_id IN (1, 2, 3, ...);  -- IN clause with all author IDs

# Python: Stitches results together (O(N) lookups)

# Access (no additional queries):
for author in authors:
    for post in author.posts.all():  # Uses prefetched data
        print(post.title)
```

**Why the difference?**

**Architecture reason:**

```
ForeignKey (many-to-one):
- Each post has 1 author
- JOIN yields same # of rows as posts table
- Efficient: SELECT posts.*, authors.* FROM posts JOIN authors

Reverse ForeignKey (one-to-many):
- Each author has N posts
- JOIN yields posts.count() rows (large!)
- Better: 2 queries (authors, then posts WHERE author_id IN (...))
```

**Example showing why JOIN doesn't work for reverse FK:**

```python
# 3 authors, 1000 posts total (avg 333 posts/author)

# If we used JOIN (inefficient):
SELECT authors.*, posts.*
FROM authors
LEFT JOIN posts ON authors.id = posts.author_id;

# Result:
# - 1000 rows (one per post)
# - Each author's data repeated 333 times
# - Data transfer: 1000 × (author_size + post_size) ≈ 500KB

# prefetch_related (efficient):
# Query 1: SELECT * FROM authors;  -- 3 rows
# Query 2: SELECT * FROM posts WHERE author_id IN (1,2,3);  -- 1000 rows
# Result:
# - Data transfer: 3 × author_size + 1000 × post_size ≈ 300KB
# - Python stitches together (minimal overhead)
```

**ManyToMany case:**

```python
# Posts with tags (many-to-many through post_tags table)
posts = Post.objects.prefetch_related('tags')

# SQL generated (2 queries):
# Query 1:
SELECT * FROM posts;

# Query 2:
SELECT tags.*, post_tags.post_id
FROM tags
INNER JOIN post_tags ON tags.id = post_tags.tag_id
WHERE post_tags.post_id IN (1, 2, 3, ...);

# Why not JOIN?
# - Would create cartesian product
# - If 100 posts × 5 tags avg = 500 rows (duplication)
# - prefetch_related: 100 + 500 = 600 rows (no duplication)
```

**Performance characteristics:**

| | select_related | prefetch_related |
|---|---|---|
| **Queries** | 1 (JOIN) | 2+ (IN query) |
| **Network** | 1 round trip | 2+ round trips |
| **Data transfer** | Duplicated for reverse FK | Minimal duplication |
| **Database work** | JOIN (can be expensive) | Multiple simple queries |
| **Python work** | Minimal | O(N) stitching |

**When to use which:**

```python
# select_related (JOIN): ForeignKey, OneToOne
Post.objects.select_related('author', 'category', 'featured_image')

# prefetch_related (IN): Reverse FK, ManyToMany
Author.objects.prefetch_related('posts', 'awards')

# Combining both:
Post.objects.select_related('author').prefetch_related('tags', 'comments')
# Query 1: SELECT posts.*, authors.* FROM posts JOIN authors
# Query 2: SELECT * FROM tags JOIN post_tags WHERE post_id IN (...)
# Query 3: SELECT * FROM comments WHERE post_id IN (...)
# Total: 3 queries
```

**Edge case: select_related depth limit**

```python
# Nested ForeignKeys:
Post.objects.select_related('author__profile__country')

# SQL:
SELECT posts.*, authors.*, profiles.*, countries.*
FROM posts
JOIN authors ON posts.author_id = authors.id
JOIN profiles ON authors.profile_id = profiles.id
JOIN countries ON profiles.country_id = countries.id;

# Deep nesting (4+ levels) can make query slow
# Consider splitting: select_related for near relationships, prefetch for deep ones
```
```

---

## 11. Summary

### N+1 Detection Checklist

- [ ] **Loops accessing relationships:** `for x in items: x.related.all()` → N+1
- [ ] **Aggregations in loops:** `for x in items: x.related.count()` → Use `.annotate()`
- [ ] **Nested loops with ORM:** Triple-nested = exponential queries
- [ ] **GraphQL resolvers:** Each field = potential N+1
- [ ] **Background jobs:** Often ignored, but same rules apply

### Quick Reference: ORM Optimization

| Pattern | Bad (N+1) | Good (Optimized) |
|---------|-----------|------------------|
| **ForeignKey access** | `for p in posts: p.author` | `posts.select_related('author')` |
| **Reverse FK** | `for a in authors: a.posts.all()` | `authors.prefetch_related('posts')` |
| **ManyToMany** | `for p in posts: p.tags.all()` | `posts.prefetch_related('tags')` |
| **Counting** | `for u in users: u.orders.count()` | `users.annotate(order_count=Count('orders'))` |
| **Aggregation** | `for p in products: sum(...)` | `products.annotate(total=Sum('...'))` |
| **Conditional load** | `if x.rel_id: x.rel` | `select_related('rel')` (LEFT JOIN) |
| **Nested chain** | `for o in orders: o.item.product` | `prefetch_related('items__product')` |
| **Bulk fetch** | `for id in ids: Model.get(id)` | `Model.filter(id__in=ids)` |

### Key Principles

1. **Lazy loading = N+1 risk**
   - ORM default: convenience over performance
   - Explicit eager loading required

2. **1 query vs N queries: 100× difference**
   - Network latency dominates (5ms × 1000 = 5s)
   - Database connection pool exhaustion
   - Query parsing overhead

3. **Test query counts in development**
   - CI test: assert max query count
   - Debug toolbar: visualize N+1
   - Response headers: monitor in production

4. **Database aggregation > Python loops**
   - SUM, COUNT, AVG in database (1 query)
   - vs fetching all data and computing in Python (N queries + memory)

5. **GraphQL = N+1 magnet**
   - Nested resolvers = nested N+1
   - DataLoader pattern required

### Optimization Strategy

```
1. Identify relationship access in loops
   → Instrument with query counter
   ↓
2. Eager load with select_related/prefetch_related
   → Test query count drops
   ↓
3. Use aggregations instead of iteration
   → annotate(), Count(), Sum()
   ↓
4. Batch queries with IN clauses
   → filter(id__in=...)
   ↓
5. Monitor in production
   → APM, slow query logs, query count alerts
```

**Remember:** N+1 is the most common ORM performance problem. Always profile query counts before shipping. One `select_related()` can make your app 100× faster.

---

**Related Reading:**
- `03_Cost_Estimation.md` - Database cost model (why 1000 queries > 1 query)
- `05_Index_Selection.md` - Indexes speed up each query in N+1, but eliminating N-1 queries is better
- `07_Statistics_Management.md` - Statistics accuracy for JOINs in eager loading
