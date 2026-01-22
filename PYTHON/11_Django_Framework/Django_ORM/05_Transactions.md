# Django Transactions

## 📖 Concept Explanation

Transactions ensure database operations are atomic - either all succeed or all fail together. Django provides transaction management to maintain data integrity.

### Transaction Properties (ACID)

```
Atomicity:    All or nothing - either all operations succeed or all fail
Consistency:  Database remains in valid state
Isolation:    Concurrent transactions don't interfere
Durability:   Committed changes persist
```

**Django Default Behavior**:

- Each HTTP request runs in a transaction (with ATOMIC_REQUESTS)
- Each ORM operation auto-commits by default
- You can explicitly control transactions

### Basic Transaction

```python
from django.db import transaction

# Atomic block - all or nothing
with transaction.atomic():
    user = User.objects.create(username='john')
    profile = UserProfile.objects.create(user=user)
    # If this fails, user creation is rolled back
    send_welcome_email(user.email)
```

## 🧠 Transaction Basics

### 1. Atomic Decorator

```python
from django.db import transaction

@transaction.atomic
def create_user_with_profile(username, email):
    """Create user and profile atomically"""
    user = User.objects.create(username=username, email=email)
    profile = UserProfile.objects.create(user=user)
    return user

# Usage
try:
    user = create_user_with_profile('john', 'john@example.com')
except Exception as e:
    # Both user and profile creation rolled back
    print(f"Error: {e}")
```

### 2. Atomic Context Manager

```python
from django.db import transaction

def transfer_funds(from_account, to_account, amount):
    """Transfer money between accounts atomically"""
    with transaction.atomic():
        # Deduct from source
        from_account.balance -= amount
        from_account.save()

        # Add to destination
        to_account.balance += amount
        to_account.save()

        # Create transaction record
        Transaction.objects.create(
            from_account=from_account,
            to_account=to_account,
            amount=amount
        )

        # If anything fails, all changes are rolled back
```

### 3. Savepoints

```python
from django.db import transaction

def complex_operation():
    """Use savepoints for partial rollback"""
    with transaction.atomic():
        # Main transaction
        user = User.objects.create(username='john')

        # Create savepoint
        sid = transaction.savepoint()

        try:
            # Risky operation
            profile = UserProfile.objects.create(user=user, bio='...')
            transaction.savepoint_commit(sid)
        except Exception:
            # Rollback to savepoint (user creation persists)
            transaction.savepoint_rollback(sid)

        # Continue with transaction
        user.is_active = True
        user.save()
```

## 🔒 Locking & Concurrency

### 1. select_for_update() - Pessimistic Locking

```python
from django.db import transaction

def decrement_inventory(product_id, quantity):
    """Decrement product inventory with locking"""
    with transaction.atomic():
        # Lock row until transaction completes
        product = Product.objects.select_for_update().get(pk=product_id)

        if product.stock < quantity:
            raise ValueError("Insufficient stock")

        product.stock -= quantity
        product.save()

        return product

# Concurrent requests will wait for lock
```

### 2. select_for_update() Options

```python
from django.db import transaction

# Basic lock (wait for other transactions)
product = Product.objects.select_for_update().get(pk=1)

# nowait - raise error if locked
try:
    product = Product.objects.select_for_update(nowait=True).get(pk=1)
except DatabaseError:
    # Product is locked by another transaction
    pass

# skip_locked - skip locked rows
available_products = Product.objects.select_for_update(skip_locked=True).filter(stock__gt=0)

# of - lock specific related objects (PostgreSQL 9.5+)
orders = Order.objects.select_for_update(of=('customer', 'items'))
```

### 3. Optimistic Locking with Version Field

```python
from django.db import models
from django.db.models import F

class Product(models.Model):
    name = models.CharField(max_length=200)
    stock = models.IntegerField(default=0)
    version = models.IntegerField(default=0)  # Version field

    def decrement_stock(self, quantity):
        """Decrement stock with optimistic locking"""
        # Read current version
        current_version = self.version

        # Update with version check
        updated = Product.objects.filter(
            pk=self.pk,
            version=current_version,
            stock__gte=quantity
        ).update(
            stock=F('stock') - quantity,
            version=F('version') + 1
        )

        if updated == 0:
            raise ValueError("Product was modified by another transaction or insufficient stock")

        # Refresh from database
        self.refresh_from_db()
```

## 🎯 Advanced Transaction Patterns

### 1. on_commit() - Post-Transaction Actions

```python
from django.db import transaction

def create_order(user, items):
    """Create order and send email after commit"""
    with transaction.atomic():
        order = Order.objects.create(user=user)

        for item_data in items:
            OrderItem.objects.create(order=order, **item_data)

        # Send email only if transaction succeeds
        transaction.on_commit(lambda: send_order_confirmation(order.id))

        # Send webhook only if transaction succeeds
        transaction.on_commit(lambda: trigger_webhook('order_created', order.id))

        return order

def send_order_confirmation(order_id):
    """Send confirmation email"""
    order = Order.objects.get(pk=order_id)
    send_mail(
        subject='Order Confirmation',
        message=f'Your order #{order.id} has been confirmed.',
        recipient_list=[order.user.email]
    )
```

### 2. Nested Transactions (Savepoints)

```python
from django.db import transaction

@transaction.atomic
def create_blog_post(title, content, tags):
    """Create post with nested transactions"""
    # Outer transaction
    post = Post.objects.create(title=title, content=content)

    # Nested atomic block (creates savepoint)
    try:
        with transaction.atomic():
            # Try to create tags
            for tag_name in tags:
                tag, created = Tag.objects.get_or_create(name=tag_name)
                post.tags.add(tag)
    except Exception as e:
        # Tag creation failed, but post still created
        print(f"Failed to add tags: {e}")

    return post
```

### 3. Manual Transaction Control

```python
from django.db import transaction

def manual_transaction_example():
    """Manual transaction control"""
    # Disable auto-commit
    transaction.set_autocommit(False)

    try:
        # Perform operations
        user = User.objects.create(username='john')
        profile = UserProfile.objects.create(user=user)

        # Manually commit
        transaction.commit()
    except Exception as e:
        # Manually rollback
        transaction.rollback()
        raise
    finally:
        # Re-enable auto-commit
        transaction.set_autocommit(True)
```

## 🏗️ Real-World Transaction Examples

### 1. E-commerce Order Processing

```python
from django.db import transaction
from decimal import Decimal

@transaction.atomic
def process_order(cart, user, payment_method):
    """Process order with inventory and payment"""
    # Lock inventory
    order_items = []

    for item in cart.items.select_for_update():
        product = item.product

        # Check stock
        if product.stock < item.quantity:
            raise ValueError(f"Insufficient stock for {product.name}")

        # Deduct inventory
        product.stock -= item.quantity
        product.save()

        order_items.append({
            'product': product,
            'quantity': item.quantity,
            'price': product.price
        })

    # Calculate total
    total = sum(item['quantity'] * item['price'] for item in order_items)

    # Create order
    order = Order.objects.create(
        user=user,
        total_amount=total,
        status='pending'
    )

    # Create order items
    for item in order_items:
        OrderItem.objects.create(
            order=order,
            product=item['product'],
            quantity=item['quantity'],
            price=item['price']
        )

    # Process payment
    payment = Payment.objects.create(
        order=order,
        amount=total,
        method=payment_method,
        status='processing'
    )

    # Charge payment (external API)
    try:
        charge_payment(payment, total)
        payment.status = 'completed'
        payment.save()

        order.status = 'confirmed'
        order.save()
    except PaymentError as e:
        # Payment failed - transaction will rollback
        raise ValueError(f"Payment failed: {e}")

    # Clear cart
    cart.items.all().delete()

    # Send confirmation email after commit
    transaction.on_commit(lambda: send_order_email(order.id))

    return order
```

### 2. Banking Transaction

```python
from django.db import transaction
from decimal import Decimal

@transaction.atomic
def transfer_money(from_account_id, to_account_id, amount):
    """Transfer money between accounts"""
    # Lock both accounts
    accounts = Account.objects.select_for_update().filter(
        id__in=[from_account_id, to_account_id]
    ).order_by('id')  # Consistent locking order prevents deadlocks

    from_account = accounts.get(id=from_account_id)
    to_account = accounts.get(id=to_account_id)

    # Validate
    if from_account.balance < amount:
        raise ValueError("Insufficient funds")

    if amount <= 0:
        raise ValueError("Amount must be positive")

    # Deduct from source
    from_account.balance -= amount
    from_account.save()

    # Add to destination
    to_account.balance += amount
    to_account.save()

    # Create transaction records
    Transaction.objects.create(
        account=from_account,
        type='debit',
        amount=amount,
        reference=f"Transfer to {to_account.account_number}"
    )

    Transaction.objects.create(
        account=to_account,
        type='credit',
        amount=amount,
        reference=f"Transfer from {from_account.account_number}"
    )

    # Send notifications after commit
    transaction.on_commit(lambda: notify_transfer(from_account.id, to_account.id, amount))

    return True
```

### 3. Batch Processing with Savepoints

```python
from django.db import transaction

def import_users_batch(user_data_list):
    """Import users with partial failure handling"""
    results = {'success': 0, 'failed': 0, 'errors': []}

    with transaction.atomic():
        for user_data in user_data_list:
            # Create savepoint for each user
            sid = transaction.savepoint()

            try:
                # Try to create user
                user = User.objects.create(**user_data)
                UserProfile.objects.create(user=user)

                # Commit savepoint
                transaction.savepoint_commit(sid)
                results['success'] += 1

            except Exception as e:
                # Rollback this user, continue with others
                transaction.savepoint_rollback(sid)
                results['failed'] += 1
                results['errors'].append({
                    'user': user_data.get('username'),
                    'error': str(e)
                })

    return results
```

## ✅ Best Practices

### 1. Keep Transactions Short

```python
# BAD: Long-running transaction
@transaction.atomic
def bad_example():
    user = User.objects.create(username='john')
    time.sleep(10)  # Locks database for 10 seconds!
    profile = UserProfile.objects.create(user=user)

# GOOD: Short transaction
@transaction.atomic
def good_example():
    user = User.objects.create(username='john')
    profile = UserProfile.objects.create(user=user)
# Do long-running tasks after transaction
```

### 2. Lock Order to Prevent Deadlocks

```python
# BAD: Inconsistent lock order (can deadlock)
def transfer_bad(from_id, to_id, amount):
    with transaction.atomic():
        from_acc = Account.objects.select_for_update().get(id=from_id)
        to_acc = Account.objects.select_for_update().get(id=to_id)
        # ...

# GOOD: Consistent lock order
def transfer_good(from_id, to_id, amount):
    with transaction.atomic():
        # Always lock in same order (by ID)
        accounts = Account.objects.select_for_update().filter(
            id__in=[from_id, to_id]
        ).order_by('id')
        # ...
```

### 3. Use on_commit() for External Actions

```python
@transaction.atomic
def create_user(username, email):
    user = User.objects.create(username=username, email=email)

    # BAD: Send email immediately (might rollback)
    # send_welcome_email(user.email)

    # GOOD: Send email after commit
    transaction.on_commit(lambda: send_welcome_email(user.email))

    return user
```

## 🔐 Security Best Practices

### 1. Prevent Race Conditions

```python
from django.db import transaction

@transaction.atomic
def reserve_seat(event_id, user_id):
    """Reserve seat with race condition protection"""
    event = Event.objects.select_for_update().get(pk=event_id)

    if event.available_seats <= 0:
        raise ValueError("No seats available")

    # Create reservation
    reservation = Reservation.objects.create(event=event, user_id=user_id)

    # Decrement available seats
    event.available_seats -= 1
    event.save()

    return reservation
```

## ⚡ Performance Optimization

### 1. Use Bulk Operations

```python
@transaction.atomic
def create_posts_bulk(post_data_list):
    """Create posts efficiently"""
    posts = [Post(**data) for data in post_data_list]
    Post.objects.bulk_create(posts, batch_size=500)
```

### 2. Avoid Unnecessary Transactions

```python
# BAD: Transaction for read-only operation
@transaction.atomic
def get_user_posts(user_id):
    return Post.objects.filter(author_id=user_id)

# GOOD: No transaction needed
def get_user_posts(user_id):
    return Post.objects.filter(author_id=user_id)
```

## ❓ Interview Questions

### Q1: What is a database transaction?

**Answer**:
A transaction is a sequence of operations that execute as a single unit - either all succeed or all fail (ACID properties).

### Q2: What is select_for_update()?

**Answer**:
Pessimistic locking - locks database rows until transaction completes. Prevents concurrent modifications.

### Q3: What is on_commit() used for?

**Answer**:
Executes actions only after transaction successfully commits (e.g., sending emails, triggering webhooks).

## 📚 Summary

**Key Takeaways**:

1. Use @transaction.atomic or context manager
2. select_for_update() for pessimistic locking
3. Savepoints for partial rollback
4. on_commit() for post-transaction actions
5. Keep transactions short
6. Lock in consistent order (prevent deadlocks)
7. Use nowait/skip_locked for non-blocking
8. Optimistic locking with version fields
9. Avoid long-running operations in transactions
10. Always handle exceptions

Django transactions ensure data integrity!
