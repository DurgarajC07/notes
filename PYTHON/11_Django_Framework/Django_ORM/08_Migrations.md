# Django Migrations

## 📖 Concept Explanation

Migrations are Django's way of propagating changes to your models (adding fields, deleting models, etc.) into your database schema. They're version control for your database.

### Migration Workflow

```
Model Change → makemigrations → Migration File → migrate → Database Schema Update
```

**Process**:

1. Modify models (add/change/delete fields)
2. Run `python manage.py makemigrations` - creates migration files
3. Review migration files
4. Run `python manage.py migrate` - applies to database
5. Commit migration files to version control

### Basic Example

```python
# models.py
class Post(models.Model):
    title = models.CharField(max_length=200)
    # Add new field
    featured = models.BooleanField(default=False)
```

```bash
# Create migration
python manage.py makemigrations

# Apply migration
python manage.py migrate
```

## 🧠 Migration Commands

### 1. Creating Migrations

```bash
# Create migrations for all apps
python manage.py makemigrations

# Create migration for specific app
python manage.py makemigrations blog

# Empty migration (for data migration)
python manage.py makemigrations --empty blog

# Name migration
python manage.py makemigrations blog --name add_featured_field

# Dry run (show what would be created)
python manage.py makemigrations --dry-run

# Merge conflicting migrations
python manage.py makemigrations --merge
```

### 2. Applying Migrations

```bash
# Apply all migrations
python manage.py migrate

# Apply migrations for specific app
python manage.py migrate blog

# Apply specific migration
python manage.py migrate blog 0001_initial

# Rollback to migration
python manage.py migrate blog 0001

# Rollback all migrations for app
python manage.py migrate blog zero

# Show migration plan without applying
python manage.py migrate --plan

# Fake migration (mark as applied without running)
python manage.py migrate --fake
python manage.py migrate blog 0002 --fake
```

### 3. Inspecting Migrations

```bash
# Show all migrations and their status
python manage.py showmigrations

# Show migrations for specific app
python manage.py showmigrations blog

# Show SQL for migration
python manage.py sqlmigrate blog 0001

# Check for unapplied migrations
python manage.py migrate --check
```

## 🏗️ Migration File Structure

### 1. Basic Migration

```python
# blog/migrations/0001_initial.py
from django.db import migrations, models

class Migration(migrations.Migration):
    initial = True

    dependencies = []

    operations = [
        migrations.CreateModel(
            name='Post',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True)),
                ('title', models.CharField(max_length=200)),
                ('content', models.TextField()),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
        ),
    ]
```

### 2. Adding Fields

```python
# 0002_add_featured.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0001_initial'),
    ]

    operations = [
        migrations.AddField(
            model_name='post',
            name='featured',
            field=models.BooleanField(default=False),
        ),
    ]
```

### 3. Renaming Fields

```python
# 0003_rename_field.py
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0002_add_featured'),
    ]

    operations = [
        migrations.RenameField(
            model_name='post',
            old_name='created_at',
            new_name='published_date',
        ),
    ]
```

### 4. Altering Fields

```python
# 0004_alter_field.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0003_rename_field'),
    ]

    operations = [
        migrations.AlterField(
            model_name='post',
            name='title',
            field=models.CharField(max_length=300),  # Increased from 200
        ),
    ]
```

### 5. Removing Fields

```python
# 0005_remove_field.py
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0004_alter_field'),
    ]

    operations = [
        migrations.RemoveField(
            model_name='post',
            name='featured',
        ),
    ]
```

## 🎯 Data Migrations

### 1. Basic Data Migration

```python
# 0006_populate_slug.py
from django.db import migrations
from django.utils.text import slugify

def populate_slug(apps, schema_editor):
    """Populate slug field for existing posts"""
    Post = apps.get_model('blog', 'Post')
    for post in Post.objects.all():
        post.slug = slugify(post.title)
        post.save()

def reverse_populate_slug(apps, schema_editor):
    """Reverse migration - clear slugs"""
    Post = apps.get_model('blog', 'Post')
    Post.objects.all().update(slug='')

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0005_add_slug_field'),
    ]

    operations = [
        migrations.RunPython(populate_slug, reverse_populate_slug),
    ]
```

### 2. Complex Data Migration

```python
# 0007_migrate_tags.py
from django.db import migrations

def migrate_tags_to_m2m(apps, schema_editor):
    """Migrate tags from CharField to ManyToMany"""
    Post = apps.get_model('blog', 'Post')
    Tag = apps.get_model('blog', 'Tag')

    for post in Post.objects.all():
        if post.tags_old:
            # Split comma-separated tags
            tag_names = [t.strip() for t in post.tags_old.split(',')]

            for tag_name in tag_names:
                # Get or create tag
                tag, created = Tag.objects.get_or_create(name=tag_name)
                post.tags.add(tag)

def reverse_migration(apps, schema_editor):
    """Reverse: ManyToMany back to CharField"""
    Post = apps.get_model('blog', 'Post')

    for post in Post.objects.all():
        tag_names = [tag.name for tag in post.tags.all()]
        post.tags_old = ', '.join(tag_names)
        post.save()

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0006_add_tag_model'),
    ]

    operations = [
        migrations.RunPython(migrate_tags_to_m2m, reverse_migration),
    ]
```

### 3. Bulk Data Migration

```python
# 0008_bulk_update.py
from django.db import migrations

def bulk_update_status(apps, schema_editor):
    """Bulk update post status"""
    Post = apps.get_model('blog', 'Post')

    # Use bulk_update for efficiency
    posts = list(Post.objects.all())
    for post in posts:
        if post.published and not post.status:
            post.status = 'published'
        elif not post.published:
            post.status = 'draft'

    Post.objects.bulk_update(posts, ['status'], batch_size=1000)

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0007_add_status_field'),
    ]

    operations = [
        migrations.RunPython(bulk_update_status),
    ]
```

## 🔄 Advanced Migration Operations

### 1. Custom SQL Migration

```python
# 0009_custom_sql.py
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0008_previous'),
    ]

    operations = [
        migrations.RunSQL(
            # Forward SQL
            """
            CREATE INDEX idx_post_title_trgm ON blog_post
            USING gin (title gin_trgm_ops);
            """,
            # Reverse SQL
            """
            DROP INDEX IF EXISTS idx_post_title_trgm;
            """
        ),
    ]
```

### 2. Conditional Migration

```python
# 0010_conditional.py
from django.db import migrations, connection

def add_field_if_not_exists(apps, schema_editor):
    """Add field only if it doesn't exist"""
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT column_name
            FROM information_schema.columns
            WHERE table_name='blog_post' AND column_name='featured'
        """)
        if not cursor.fetchone():
            cursor.execute("""
                ALTER TABLE blog_post
                ADD COLUMN featured BOOLEAN DEFAULT FALSE
            """)

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0009_previous'),
    ]

    operations = [
        migrations.RunPython(add_field_if_not_exists),
    ]
```

### 3. Separating Schema and Data

```python
# 0011_add_field.py - Schema only
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0010_previous'),
    ]

    operations = [
        migrations.AddField(
            model_name='post',
            name='view_count',
            field=models.IntegerField(default=0),
        ),
    ]

# 0012_populate_view_count.py - Data only
from django.db import migrations

def populate_view_count(apps, schema_editor):
    Post = apps.get_model('blog', 'Post')
    # Populate from analytics table
    # ...

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0011_add_field'),
    ]

    operations = [
        migrations.RunPython(populate_view_count),
    ]
```

## ✅ Best Practices

### 1. Reversible Migrations

```python
# GOOD: Reversible
class Migration(migrations.Migration):
    operations = [
        migrations.RunPython(forward_func, reverse_func),
    ]

# BAD: Not reversible
class Migration(migrations.Migration):
    operations = [
        migrations.RunPython(forward_func),  # No reverse!
    ]
```

### 2. Atomic Migrations

```python
# Atomic by default
class Migration(migrations.Migration):
    atomic = True  # Default

    operations = [
        # All operations in transaction
    ]

# Non-atomic (for operations that can't run in transaction)
class Migration(migrations.Migration):
    atomic = False  # e.g., for CREATE INDEX CONCURRENTLY

    operations = [
        migrations.RunSQL(
            "CREATE INDEX CONCURRENTLY idx_title ON blog_post (title);"
        ),
    ]
```

### 3. Safe Migrations in Production

```python
# BAD: Removing field (data loss!)
migrations.RemoveField('post', 'old_field')

# GOOD: Two-step process
# Step 1: Deploy code that doesn't use field
# Step 2: After deployment, remove field in new migration

# BAD: Renaming with data
migrations.RenameField('post', 'old_name', 'new_name')

# GOOD: Add new, migrate data, remove old
# Migration 1: Add new field
# Migration 2: Copy data
# Migration 3: Remove old field
```

## 🔐 Security & Safety

### 1. Review Before Applying

```bash
# Always review generated migrations
cat blog/migrations/0001_initial.py

# Check SQL that will be executed
python manage.py sqlmigrate blog 0001

# Dry run
python manage.py migrate --plan
```

### 2. Backup Before Migration

```bash
# PostgreSQL backup
pg_dump mydb > backup_before_migration.sql

# MySQL backup
mysqldump mydb > backup_before_migration.sql

# Then apply migration
python manage.py migrate
```

### 3. Test Migrations

```python
# tests/test_migrations.py
from django.test import TransactionTestCase
from django_test_migrations.migrator import Migrator

class TestMigration(TransactionTestCase):
    def test_migration_0002(self):
        migrator = Migrator(database='default')

        # Start from previous state
        old_state = migrator.apply_initial_migration(('blog', '0001_initial'))

        # Create test data
        Post = old_state.apps.get_model('blog', 'Post')
        Post.objects.create(title='Test')

        # Apply migration
        new_state = migrator.apply_tested_migration(('blog', '0002_add_featured'))

        # Verify migration worked
        Post = new_state.apps.get_model('blog', 'Post')
        post = Post.objects.first()
        self.assertFalse(post.featured)
```

## ⚡ Performance Optimization

### 1. Batch Operations

```python
def bulk_update(apps, schema_editor):
    Post = apps.get_model('blog', 'Post')

    # Update in batches
    batch_size = 1000
    posts = Post.objects.all()

    for i in range(0, posts.count(), batch_size):
        batch = list(posts[i:i+batch_size])
        for post in batch:
            post.slug = slugify(post.title)
        Post.objects.bulk_update(batch, ['slug'], batch_size=batch_size)
```

### 2. Index Creation

```python
# Create index concurrently (PostgreSQL)
class Migration(migrations.Migration):
    atomic = False

    operations = [
        migrations.RunSQL(
            "CREATE INDEX CONCURRENTLY idx_post_title ON blog_post (title);",
            "DROP INDEX IF EXISTS idx_post_title;"
        ),
    ]
```

## ❓ Interview Questions

### Q1: What are Django migrations?

**Answer**:
Version control for database schema. Track and apply model changes to database using migration files.

### Q2: What's the difference between makemigrations and migrate?

**Answer**:

- **makemigrations**: Creates migration files from model changes
- **migrate**: Applies migration files to database

### Q3: How do you handle conflicting migrations?

**Answer**:
Use `python manage.py makemigrations --merge` to create merge migration.

### Q4: What is RunPython used for?

**Answer**:
Data migrations - modify existing data during migration (e.g., populate new fields, transform data).

## 📚 Summary

**Key Takeaways**:

1. makemigrations creates migration files
2. migrate applies migrations to database
3. Always review migrations before applying
4. Use RunPython for data migrations
5. Keep migrations reversible
6. Test migrations before production
7. Backup database before migration
8. Use atomic=False for non-transactional operations
9. Batch bulk operations for performance
10. Commit migration files to version control

Django migrations make database evolution safe and trackable!
