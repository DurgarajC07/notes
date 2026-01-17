# Django REST Framework Serializers

## 📖 Concept Explanation

Serializers in Django REST Framework (DRF) convert complex data types (QuerySets, model instances) into Python native datatypes that can be rendered into JSON, XML, or other content types. They also handle deserialization and validation.

### Serialization Flow

```
Model Instance → Serializer → Python Dict → JSON
JSON → Python Dict → Serializer → Validation → Model Instance
```

**Purpose**:

- Convert Django models to JSON/XML
- Validate incoming request data
- Handle nested relationships
- Transform data presentation

### Basic Serializer

```python
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)

    def create(self, validated_data):
        """Create and return new Post instance"""
        return Post.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """Update and return existing Post instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.published = validated_data.get('published', instance.published)
        instance.save()
        return instance
```

## 🧠 ModelSerializer (Recommended)

### 1. Basic ModelSerializer

```python
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):
    """Automatically generates fields from model"""

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'published', 'created_at']
        # Or use all fields
        # fields = '__all__'

        # Exclude specific fields
        # exclude = ['internal_notes']

        # Read-only fields
        read_only_fields = ['id', 'created_at', 'author']

# Usage
post = Post.objects.get(pk=1)
serializer = PostSerializer(post)
print(serializer.data)
# {'id': 1, 'title': 'My Post', 'content': '...', ...}
```

### 2. Custom Fields

```python
class PostSerializer(serializers.ModelSerializer):
    # Custom field (method)
    author_name = serializers.CharField(source='author.username', read_only=True)

    # SerializerMethodField
    comment_count = serializers.SerializerMethodField()

    # URL field
    url = serializers.HyperlinkedIdentityField(view_name='post-detail')

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author_name', 'comment_count', 'url']

    def get_comment_count(self, obj):
        """Custom method for SerializerMethodField"""
        return obj.comments.count()
```

### 3. Write-Only Fields

```python
class UserSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'password', 'password_confirm']

    def validate(self, data):
        """Validate password match"""
        if data['password'] != data['password_confirm']:
            raise serializers.ValidationError("Passwords don't match")
        return data

    def create(self, validated_data):
        """Create user with hashed password"""
        validated_data.pop('password_confirm')
        password = validated_data.pop('password')
        user = User.objects.create(**validated_data)
        user.set_password(password)
        user.save()
        return user
```

## 🎯 Field Types

### 1. Common Fields

```python
class ExampleSerializer(serializers.Serializer):
    # String fields
    char_field = serializers.CharField(max_length=100)
    email_field = serializers.EmailField()
    url_field = serializers.URLField()
    slug_field = serializers.SlugField()

    # Numeric fields
    integer_field = serializers.IntegerField()
    float_field = serializers.FloatField()
    decimal_field = serializers.DecimalField(max_digits=10, decimal_places=2)

    # Boolean
    boolean_field = serializers.BooleanField()

    # Date/Time
    date_field = serializers.DateField()
    time_field = serializers.TimeField()
    datetime_field = serializers.DateTimeField()

    # Choice field
    STATUS_CHOICES = [('draft', 'Draft'), ('published', 'Published')]
    status = serializers.ChoiceField(choices=STATUS_CHOICES)

    # File fields
    file_field = serializers.FileField()
    image_field = serializers.ImageField()

    # JSON field
    json_field = serializers.JSONField()

    # List field
    tags = serializers.ListField(child=serializers.CharField())
```

### 2. Relational Fields

```python
class PostSerializer(serializers.ModelSerializer):
    # PrimaryKeyRelatedField (default for ForeignKey)
    author = serializers.PrimaryKeyRelatedField(queryset=User.objects.all())
    # Returns: {"author": 1}

    # StringRelatedField (uses __str__)
    author = serializers.StringRelatedField()
    # Returns: {"author": "john"}

    # SlugRelatedField
    author = serializers.SlugRelatedField(slug_field='username', queryset=User.objects.all())
    # Returns: {"author": "john"}

    # HyperlinkedRelatedField
    author = serializers.HyperlinkedRelatedField(view_name='user-detail', queryset=User.objects.all())
    # Returns: {"author": "http://api.example.com/users/1/"}

    # Nested serializer (read-only)
    author = UserSerializer(read_only=True)
    # Returns: {"author": {"id": 1, "username": "john", "email": "..."}}

    class Meta:
        model = Post
        fields = ['id', 'title', 'author']
```

## 🔗 Nested Serializers

### 1. Read-Only Nested

```python
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)  # Nested serializer

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author']

# Output:
# {
#     "id": 1,
#     "title": "My Post",
#     "content": "...",
#     "author": {
#         "id": 1,
#         "username": "john",
#         "email": "john@example.com"
#     }
# }
```

### 2. Writable Nested

```python
class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'text', 'author']

class PostSerializer(serializers.ModelSerializer):
    comments = CommentSerializer(many=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'comments']

    def create(self, validated_data):
        """Create post with nested comments"""
        comments_data = validated_data.pop('comments')
        post = Post.objects.create(**validated_data)

        for comment_data in comments_data:
            Comment.objects.create(post=post, **comment_data)

        return post

    def update(self, instance, validated_data):
        """Update post and nested comments"""
        comments_data = validated_data.pop('comments', None)

        # Update post fields
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()

        # Update comments
        if comments_data is not None:
            instance.comments.all().delete()
            for comment_data in comments_data:
                Comment.objects.create(post=instance, **comment_data)

        return instance
```

### 3. Multiple Nesting Levels

```python
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name']

class CommentSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)

    class Meta:
        model = Comment
        fields = ['id', 'text', 'author', 'created_at']

class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = TagSerializer(many=True, read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'tags']
```

## ✅ Validation

### 1. Field-Level Validation

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'published']

    def validate_title(self, value):
        """Validate title field"""
        if len(value) < 10:
            raise serializers.ValidationError("Title must be at least 10 characters")

        if Post.objects.filter(title=value).exists():
            raise serializers.ValidationError("Title already exists")

        return value

    def validate_content(self, value):
        """Validate content field"""
        if 'spam' in value.lower():
            raise serializers.ValidationError("Content contains prohibited words")

        return value
```

### 2. Object-Level Validation

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'published', 'published_date']

    def validate(self, data):
        """Cross-field validation"""
        if data.get('published') and not data.get('published_date'):
            raise serializers.ValidationError(
                "Published posts must have a published_date"
            )

        if data.get('published') and len(data.get('content', '')) < 100:
            raise serializers.ValidationError(
                "Published posts must have at least 100 characters"
            )

        return data
```

### 3. Custom Validators

```python
from rest_framework import serializers

def validate_no_profanity(value):
    """Custom validator function"""
    profanity_list = ['badword1', 'badword2']
    if any(word in value.lower() for word in profanity_list):
        raise serializers.ValidationError("Content contains profanity")

class PostSerializer(serializers.ModelSerializer):
    content = serializers.CharField(validators=[validate_no_profanity])

    class Meta:
        model = Post
        fields = ['id', 'title', 'content']
```

## 🏗️ Advanced Patterns

### 1. Dynamic Fields

```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """Serializer that allows dynamic field selection"""

    def __init__(self, *args, **kwargs):
        # Get fields parameter
        fields = kwargs.pop('fields', None)

        super().__init__(*args, **kwargs)

        if fields is not None:
            # Drop fields not specified
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)

class PostSerializer(DynamicFieldsSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at']

# Usage
serializer = PostSerializer(post, fields=['id', 'title'])
# Only returns: {"id": 1, "title": "My Post"}
```

### 2. Context-Aware Serializers

```python
class PostSerializer(serializers.ModelSerializer):
    # Access request via context
    is_author = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'is_author', 'can_edit']

    def get_is_author(self, obj):
        """Check if current user is author"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user
        return False

    def get_can_edit(self, obj):
        """Check if user can edit"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.author == request.user or request.user.is_staff
        return False

# Usage in view
serializer = PostSerializer(post, context={'request': request})
```

### 3. Different Serializers for Read/Write

```python
class PostListSerializer(serializers.ModelSerializer):
    """Minimal fields for list view"""
    author_name = serializers.CharField(source='author.username', read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'author_name', 'created_at']

class PostDetailSerializer(serializers.ModelSerializer):
    """Full fields for detail view"""
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = TagSerializer(many=True, read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'tags', 'created_at']

class PostCreateUpdateSerializer(serializers.ModelSerializer):
    """Writable serializer"""
    class Meta:
        model = Post
        fields = ['title', 'content', 'published', 'tags']

# In ViewSet
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()

    def get_serializer_class(self):
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'retrieve':
            return PostDetailSerializer
        return PostCreateUpdateSerializer
```

## 🔐 Security Best Practices

### 1. Sensitive Data Protection

```python
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'is_active']
        # Never expose: password, password hash, tokens, internal IDs

        # Use write_only for sensitive input
        extra_kwargs = {
            'password': {'write_only': True},
        }
```

### 2. Input Sanitization

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']

    def validate_content(self, value):
        """Sanitize HTML content"""
        import bleach

        allowed_tags = ['p', 'br', 'strong', 'em', 'a']
        allowed_attrs = {'a': ['href', 'title']}

        cleaned = bleach.clean(
            value,
            tags=allowed_tags,
            attributes=allowed_attrs,
            strip=True
        )

        return cleaned
```

## ⚡ Performance Optimization

### 1. select_related & prefetch_related

```python
class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'comments']

    @classmethod
    def setup_eager_loading(cls, queryset):
        """Optimize queryset with select_related/prefetch_related"""
        queryset = queryset.select_related('author')
        queryset = queryset.prefetch_related('comments__author')
        return queryset

# In ViewSet
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer

    def get_queryset(self):
        queryset = Post.objects.all()
        return PostSerializer.setup_eager_loading(queryset)
```

### 2. Limit Fields in Nested Serializers

```python
class UserMinimalSerializer(serializers.ModelSerializer):
    """Minimal user data for nested serialization"""
    class Meta:
        model = User
        fields = ['id', 'username']  # Only essential fields

class PostSerializer(serializers.ModelSerializer):
    author = UserMinimalSerializer(read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'author']
```

## ❓ Interview Questions

### Q1: What's the difference between Serializer and ModelSerializer?

**Answer**:

- **Serializer**: Manual field definition, custom create/update logic
- **ModelSerializer**: Auto-generates fields from model, handles save automatically

Use ModelSerializer for model-based APIs.

### Q2: How do you handle nested relationships?

**Answer**:
Use nested serializers with `many=True` for multiple objects. Override `create()` and `update()` to handle nested data properly.

### Q3: What is SerializerMethodField used for?

**Answer**:
Compute custom fields at serialization time. Define method `get_<field_name>()` that returns the value.

## 📚 Summary

**Key Takeaways**:

1. ModelSerializer for model-based APIs
2. Nested serializers for relationships
3. SerializerMethodField for computed fields
4. validate\_<field> for field-level validation
5. validate() for object-level validation
6. read_only=True for output-only fields
7. write_only=True for input-only fields
8. Use context for request-aware serialization
9. Optimize with select_related/prefetch_related
10. Never expose sensitive data

DRF serializers are powerful and flexible!
