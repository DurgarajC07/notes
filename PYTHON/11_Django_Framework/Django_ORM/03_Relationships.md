# Django Model Relationships

## 📖 Concept Explanation

Django provides three types of relationships between models: ForeignKey (many-to-one), OneToOneField (one-to-one), and ManyToManyField (many-to-many).

### Relationship Types

```
ForeignKey:        Many Posts → One Author
OneToOneField:     One User → One Profile
ManyToManyField:   Many Posts ↔ Many Tags
```

**Database Implementation**:

- ForeignKey: Foreign key column in table
- OneToOneField: Unique foreign key column
- ManyToManyField: Intermediate join table

## 🧠 ForeignKey (Many-to-One)

### 1. Basic ForeignKey

```python
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,  # Delete posts when user deleted
        related_name='posts',       # Reverse: user.posts.all()
        related_query_name='post'   # Filter: User.objects.filter(post__title='')
    )
    category = models.ForeignKey(
        'Category',
        on_delete=models.SET_NULL,
        null=True,
        blank=True
    )

class Category(models.Model):
    name = models.CharField(max_length=100)
```

### 2. on_delete Options

```python
class Post(models.Model):
    # CASCADE: Delete posts when author deleted
    author = models.ForeignKey(User, on_delete=models.CASCADE)

    # PROTECT: Prevent author deletion if posts exist
    reviewer = models.ForeignKey(User, on_delete=models.PROTECT, related_name='reviewed_posts')

    # SET_NULL: Set to NULL when category deleted
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)

    # SET_DEFAULT: Set to default when deleted
    status = models.ForeignKey('Status', on_delete=models.SET_DEFAULT, default=1)

    # SET: Set to specific value
    def get_default_category():
        return Category.objects.get(name='Uncategorized')

    category_alt = models.ForeignKey(Category, on_delete=models.SET(get_default_category))

    # DO_NOTHING: Do nothing (can break integrity)
    legacy_ref = models.ForeignKey('Legacy', on_delete=models.DO_NOTHING, null=True)
```

### 3. Working with ForeignKeys

```python
# Create with FK
author = User.objects.get(username='john')
post = Post.objects.create(
    title='My Post',
    author=author,
    category_id=1  # Can use ID directly
)

# Access forward relation
post = Post.objects.get(pk=1)
print(post.author.username)  # Access author
print(post.category.name)    # Access category

# Access reverse relation
author = User.objects.get(username='john')
posts = author.posts.all()  # Get all posts by author
count = author.posts.count()

# Filter by FK
posts = Post.objects.filter(author__username='john')
posts = Post.objects.filter(category__name='Technology')

# Filter reverse
users = User.objects.filter(posts__published=True)
categories = Category.objects.filter(post__views__gte=1000)
```

## 🔗 OneToOneField

### 1. Basic OneToOne

```python
class UserProfile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField()
    avatar = models.ImageField(upload_to='avatars/')
    website = models.URLField(blank=True)

    def __str__(self):
        return f"Profile for {self.user.username}"

# Usage
user = User.objects.get(username='john')
profile = user.profile  # Access OneToOne

# Create profile
profile = UserProfile.objects.create(
    user=user,
    bio='Software developer'
)

# Or use get_or_create
profile, created = UserProfile.objects.get_or_create(
    user=user,
    defaults={'bio': 'New user'}
)
```

### 2. Multi-Table Inheritance (OneToOne under the hood)

```python
class Person(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)

    def __str__(self):
        return f"{self.first_name} {self.last_name}"

class Employee(Person):  # Inherits Person fields
    employee_id = models.CharField(max_length=20)
    department = models.CharField(max_length=100)

    # Behind the scenes, Django creates OneToOneField to Person

class Customer(Person):
    customer_id = models.CharField(max_length=20)
    loyalty_points = models.IntegerField(default=0)

# Usage
employee = Employee.objects.create(
    first_name='John',
    last_name='Doe',
    employee_id='E12345',
    department='Engineering'
)

# Access parent fields
print(employee.first_name)

# Access child from parent
person = Person.objects.get(pk=employee.pk)
if hasattr(person, 'employee'):
    print(person.employee.department)
```

## 🔄 ManyToManyField

### 1. Basic ManyToMany

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(
        'Tag',
        related_name='posts',
        blank=True
    )

class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name

# Usage - Add tags
post = Post.objects.get(pk=1)
tag1 = Tag.objects.get(name='django')
tag2 = Tag.objects.get(name='python')

post.tags.add(tag1, tag2)

# Or by ID
post.tags.add(1, 2, 3)

# Remove tags
post.tags.remove(tag1)

# Clear all tags
post.tags.clear()

# Set tags (replace all)
post.tags.set([tag1, tag2])

# Access from other side
tag = Tag.objects.get(name='django')
posts = tag.posts.all()
```

### 2. ManyToMany with Through Model

```python
class Author(models.Model):
    name = models.CharField(max_length=100)

class Book(models.Model):
    title = models.CharField(max_length=200)
    authors = models.ManyToManyField(
        Author,
        through='BookAuthor',  # Custom intermediate model
        related_name='books'
    )

class BookAuthor(models.Model):
    """Intermediate model with extra fields"""
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    role = models.CharField(max_length=50)  # e.g., 'Lead Author', 'Co-Author'
    order = models.PositiveIntegerField(default=0)

    class Meta:
        unique_together = ['book', 'author']
        ordering = ['order']

# Usage - Create with through model
book = Book.objects.create(title='Django Book')
author = Author.objects.create(name='John Doe')

BookAuthor.objects.create(
    book=book,
    author=author,
    role='Lead Author',
    order=1
)

# Query with through data
book_authors = BookAuthor.objects.filter(book=book).select_related('author')
for ba in book_authors:
    print(f"{ba.author.name} - {ba.role}")

# Access through data
book = Book.objects.prefetch_related('bookauthor_set').get(pk=1)
for book_author in book.bookauthor_set.all():
    print(f"{book_author.author.name}: {book_author.role}")
```

### 3. Real-World M2M Examples

```python
# E-commerce: Products and Orders
class Order(models.Model):
    customer = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    products = models.ManyToManyField('Product', through='OrderItem')

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity = models.PositiveIntegerField(default=1)
    price = models.DecimalField(max_digits=10, decimal_places=2)  # Price at time of order

    class Meta:
        unique_together = ['order', 'product']

# Social Network: Users and Followers
class User(AbstractUser):
    following = models.ManyToManyField(
        'self',
        through='Follow',
        symmetrical=False,  # If A follows B, B doesn't follow A
        related_name='followers'
    )

class Follow(models.Model):
    follower = models.ForeignKey(User, on_delete=models.CASCADE, related_name='following_set')
    following = models.ForeignKey(User, on_delete=models.CASCADE, related_name='follower_set')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ['follower', 'following']
        constraints = [
            models.CheckConstraint(
                check=~models.Q(follower=models.F('following')),
                name='prevent_self_follow'
            )
        ]
```

## 🎯 Advanced Relationship Patterns

### 1. Self-Referencing ForeignKey

```python
class Category(models.Model):
    name = models.CharField(max_length=100)
    parent = models.ForeignKey(
        'self',
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name='children'
    )

    def get_ancestors(self):
        """Get all parent categories"""
        ancestors = []
        current = self.parent
        while current:
            ancestors.append(current)
            current = current.parent
        return ancestors

    def get_descendants(self):
        """Get all child categories recursively"""
        descendants = list(self.children.all())
        for child in self.children.all():
            descendants.extend(child.get_descendants())
        return descendants

# Usage
electronics = Category.objects.create(name='Electronics')
computers = Category.objects.create(name='Computers', parent=electronics)
laptops = Category.objects.create(name='Laptops', parent=computers)

print(laptops.get_ancestors())  # [Computers, Electronics]
print(electronics.children.all())  # [Computers]
```

### 2. Generic Relations

```python
from django.contrib.contenttypes.fields import GenericForeignKey, GenericRelation
from django.contrib.contenttypes.models import ContentType

class Comment(models.Model):
    """Generic comment that can be attached to any model"""
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')

    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

class Post(models.Model):
    title = models.CharField(max_length=200)
    comments = GenericRelation(Comment)

class Photo(models.Model):
    caption = models.CharField(max_length=200)
    comments = GenericRelation(Comment)

# Usage
post = Post.objects.create(title='My Post')
Comment.objects.create(content_object=post, text='Great post!')

photo = Photo.objects.create(caption='Sunset')
Comment.objects.create(content_object=photo, text='Beautiful!')

# Query comments
post_comments = post.comments.all()
```

## ✅ Best Practices

### 1. Efficient Relationship Queries

```python
# BAD: N+1 queries
posts = Post.objects.all()
for post in posts:
    print(post.author.username)  # Extra query per post
    for tag in post.tags.all():  # Extra query per post
        print(tag.name)

# GOOD: Optimized with select_related and prefetch_related
posts = Post.objects.select_related('author', 'category') \
                    .prefetch_related('tags')

for post in posts:
    print(post.author.username)  # No extra query
    for tag in post.tags.all():  # No extra query
        print(tag.name)
```

### 2. Proper related_name

```python
class Post(models.Model):
    # GOOD: Descriptive related_name
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')

    # BAD: Default related_name (post_set)
    # author = models.ForeignKey(User, on_delete=models.CASCADE)

# Usage
user.posts.all()  # Clear and readable
```

### 3. Database Indexes

```python
class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE, db_index=True)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)

    class Meta:
        indexes = [
            models.Index(fields=['author', 'created_at']),
            models.Index(fields=['category', 'published']),
        ]
```

## 🔐 Security Best Practices

### 1. Authorization on Relationships

```python
class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)

    def can_edit(self, user):
        """Check if user can edit post"""
        return self.author == user or user.is_staff

    def can_delete(self, user):
        """Check if user can delete post"""
        return self.author == user or user.is_superuser

# In view
def edit_post(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if not post.can_edit(request.user):
        raise PermissionDenied
    # ...
```

## ⚡ Performance Optimization

### 1. Use select_related for FKs

```python
# Single query with JOIN
posts = Post.objects.select_related('author', 'category').all()
```

### 2. Use prefetch_related for M2M

```python
# Two queries (better than N+1)
posts = Post.objects.prefetch_related('tags').all()
```

### 3. Custom Prefetch

```python
from django.db.models import Prefetch

posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(approved=True).select_related('author'),
        to_attr='approved_comments'
    )
)

for post in posts:
    for comment in post.approved_comments:  # Use custom attribute
        print(comment.text)
```

## ❓ Interview Questions

### Q1: What's the difference between ForeignKey and OneToOneField?

**Answer**:

- **ForeignKey**: Many-to-one (many posts → one author)
- **OneToOneField**: One-to-one with unique constraint (one user → one profile)

### Q2: When do you need a through model in ManyToManyField?

**Answer**:
When you need extra fields on the relationship (e.g., order, role, timestamp).

### Q3: What does symmetrical=False mean in self-referencing M2M?

**Answer**:
If User A follows User B, it doesn't automatically mean B follows A. Used for followers/following relationships.

## 📚 Summary

**Key Takeaways**:

1. ForeignKey: many-to-one relationships
2. OneToOneField: one-to-one relationships
3. ManyToManyField: many-to-many relationships
4. Use on_delete to control cascade behavior
5. related_name for reverse relations
6. Through models for M2M with extra fields
7. select_related for FK/OneToOne optimization
8. prefetch_related for M2M/reverse FK optimization
9. Generic relations for flexible relationships
10. Always add indexes on ForeignKey fields

Django relationships are powerful and flexible!
