# Django Service Layer Pattern

## 📖 Concept Explanation

The Service Layer pattern separates business logic from views, making code more maintainable, testable, and reusable. Services encapsulate complex operations that span multiple models or require transaction management.

### Without Service Layer (Fat Views)

```python
# ❌ Business logic in views
def create_order(request):
    # Complex logic mixed with HTTP handling
    order = Order.objects.create(user=request.user, total=0)

    for item in request.POST.getlist('items'):
        product = Product.objects.get(id=item['product_id'])
        if product.stock < item['quantity']:
            return HttpResponse("Out of stock", status=400)

        OrderItem.objects.create(
            order=order,
            product=product,
            quantity=item['quantity'],
            price=product.price
        )

        product.stock -= item['quantity']
        product.save()

        order.total += product.price * item['quantity']
        order.save()

    # Send email
    send_mail(...)

    return JsonResponse({'order_id': order.id})
```

### With Service Layer (Thin Views)

```python
# ✅ Business logic in service
# views.py
def create_order(request):
    try:
        order = OrderService.create_order(
            user=request.user,
            items=request.POST.getlist('items')
        )
        return JsonResponse({'order_id': order.id})
    except OutOfStockError as e:
        return HttpResponse(str(e), status=400)

# services.py
class OrderService:
    @staticmethod
    @transaction.atomic
    def create_order(user, items):
        # Reusable business logic
        ...
```

## 🧠 Service Layer Structure

### 1. Basic Service

```python
# apps/blog/services/post_service.py
from django.db import transaction
from django.utils import timezone
from apps.blog.models import Post
from apps.core.exceptions import ValidationError

class PostService:
    """Service for post-related business logic"""

    @staticmethod
    def create_post(title, content, author, tags=None):
        """
        Create a new post with validation

        Args:
            title: Post title
            content: Post content
            author: User instance
            tags: List of tag names

        Returns:
            Post instance

        Raises:
            ValidationError: If validation fails
        """
        # Validation
        if len(title) < 5:
            raise ValidationError("Title must be at least 5 characters")

        if not content.strip():
            raise ValidationError("Content cannot be empty")

        # Create post
        post = Post.objects.create(
            title=title,
            content=content,
            author=author,
            created_at=timezone.now()
        )

        # Add tags
        if tags:
            from apps.blog.models import Tag
            tag_objects = [
                Tag.objects.get_or_create(name=name)[0]
                for name in tags
            ]
            post.tags.set(tag_objects)

        return post

    @staticmethod
    def get_published_posts(category=None, limit=10):
        """Get published posts with optional filtering"""
        queryset = Post.objects.filter(
            published=True,
            deleted_at__isnull=True
        ).select_related('author', 'category').prefetch_related('tags')

        if category:
            queryset = queryset.filter(category=category)

        return queryset[:limit]

    @staticmethod
    @transaction.atomic
    def delete_post(post_id, user):
        """Soft delete post with permission check"""
        try:
            post = Post.objects.get(id=post_id)
        except Post.DoesNotExist:
            raise ValidationError("Post not found")

        # Permission check
        if post.author != user and not user.is_staff:
            raise ValidationError("You don't have permission to delete this post")

        # Soft delete
        post.deleted_at = timezone.now()
        post.save()

        return post
```

### 2. Service with Multiple Models

```python
# apps/ecommerce/services/order_service.py
from django.db import transaction
from django.core.mail import send_mail
from apps.ecommerce.models import Order, OrderItem, Product
from apps.core.exceptions import OutOfStockError, ValidationError

class OrderService:
    """Service for order-related business logic"""

    @staticmethod
    @transaction.atomic
    def create_order(user, items):
        """
        Create order with inventory management

        Args:
            user: User instance
            items: List of dicts [{'product_id': 1, 'quantity': 2}, ...]

        Returns:
            Order instance

        Raises:
            OutOfStockError: If product out of stock
            ValidationError: If validation fails
        """
        # Validate items
        if not items:
            raise ValidationError("Order must contain at least one item")

        # Create order
        order = Order.objects.create(
            user=user,
            status='pending',
            total=0
        )

        total = 0

        for item_data in items:
            product_id = item_data['product_id']
            quantity = item_data['quantity']

            # Get product with row-level lock
            try:
                product = Product.objects.select_for_update().get(id=product_id)
            except Product.DoesNotExist:
                raise ValidationError(f"Product {product_id} not found")

            # Check stock
            if product.stock < quantity:
                raise OutOfStockError(
                    f"Product '{product.name}' out of stock. "
                    f"Available: {product.stock}, Requested: {quantity}"
                )

            # Create order item
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=quantity,
                price=product.price
            )

            # Update stock
            product.stock -= quantity
            product.save()

            # Update total
            total += product.price * quantity

        # Update order total
        order.total = total
        order.save()

        # Send confirmation email
        OrderService._send_order_confirmation(order)

        return order

    @staticmethod
    def _send_order_confirmation(order):
        """Send order confirmation email (private method)"""
        send_mail(
            subject=f'Order #{order.id} Confirmation',
            message=f'Your order has been placed. Total: ${order.total}',
            from_email='noreply@example.com',
            recipient_list=[order.user.email],
            fail_silently=True
        )

    @staticmethod
    @transaction.atomic
    def cancel_order(order_id, user):
        """Cancel order and restore inventory"""
        try:
            order = Order.objects.select_for_update().get(id=order_id)
        except Order.DoesNotExist:
            raise ValidationError("Order not found")

        # Permission check
        if order.user != user and not user.is_staff:
            raise ValidationError("You don't have permission to cancel this order")

        # Check if cancellable
        if order.status in ['shipped', 'delivered', 'cancelled']:
            raise ValidationError(f"Cannot cancel order with status '{order.status}'")

        # Restore inventory
        for item in order.items.all():
            product = Product.objects.select_for_update().get(id=item.product_id)
            product.stock += item.quantity
            product.save()

        # Update order status
        order.status = 'cancelled'
        order.save()

        return order

    @staticmethod
    def get_order_statistics(user, date_from=None, date_to=None):
        """Get order statistics for user"""
        from django.db.models import Sum, Count, Avg

        queryset = Order.objects.filter(user=user)

        if date_from:
            queryset = queryset.filter(created_at__gte=date_from)
        if date_to:
            queryset = queryset.filter(created_at__lte=date_to)

        stats = queryset.aggregate(
            total_orders=Count('id'),
            total_spent=Sum('total'),
            average_order=Avg('total')
        )

        return stats
```

### 3. Service with External APIs

```python
# apps/payments/services/payment_service.py
import stripe
from django.conf import settings
from django.db import transaction
from apps.payments.models import Payment

stripe.api_key = settings.STRIPE_SECRET_KEY

class PaymentService:
    """Service for payment processing"""

    @staticmethod
    @transaction.atomic
    def create_payment(order, payment_method):
        """
        Process payment for order

        Args:
            order: Order instance
            payment_method: Stripe payment method ID

        Returns:
            Payment instance

        Raises:
            PaymentError: If payment fails
        """
        # Create payment record
        payment = Payment.objects.create(
            order=order,
            amount=order.total,
            status='pending'
        )

        try:
            # Process with Stripe
            intent = stripe.PaymentIntent.create(
                amount=int(order.total * 100),  # Convert to cents
                currency='usd',
                payment_method=payment_method,
                confirmation_method='automatic',
                confirm=True,
                metadata={
                    'order_id': order.id,
                    'payment_id': payment.id
                }
            )

            # Update payment
            payment.stripe_payment_intent_id = intent.id
            payment.status = 'succeeded'
            payment.save()

            # Update order
            order.status = 'paid'
            order.save()

            return payment

        except stripe.error.CardError as e:
            # Card declined
            payment.status = 'failed'
            payment.error_message = str(e)
            payment.save()

            raise PaymentError(f"Card declined: {e.user_message}")

        except stripe.error.StripeError as e:
            # Stripe error
            payment.status = 'failed'
            payment.error_message = str(e)
            payment.save()

            raise PaymentError("Payment processing failed")

    @staticmethod
    def refund_payment(payment_id):
        """Refund a payment"""
        try:
            payment = Payment.objects.get(id=payment_id)
        except Payment.DoesNotExist:
            raise ValidationError("Payment not found")

        if payment.status != 'succeeded':
            raise ValidationError("Can only refund succeeded payments")

        try:
            # Create Stripe refund
            refund = stripe.Refund.create(
                payment_intent=payment.stripe_payment_intent_id
            )

            # Update payment
            payment.status = 'refunded'
            payment.stripe_refund_id = refund.id
            payment.save()

            # Update order
            payment.order.status = 'refunded'
            payment.order.save()

            return payment

        except stripe.error.StripeError as e:
            raise PaymentError("Refund failed")
```

## ✅ Service Layer Best Practices

### 1. Dependency Injection

```python
# services/notification_service.py
from abc import ABC, abstractmethod

class NotificationBackend(ABC):
    """Abstract notification backend"""

    @abstractmethod
    def send(self, recipient, message):
        pass

class EmailBackend(NotificationBackend):
    """Email notification backend"""

    def send(self, recipient, message):
        from django.core.mail import send_mail
        send_mail(
            subject=message['subject'],
            message=message['body'],
            from_email='noreply@example.com',
            recipient_list=[recipient]
        )

class SMSBackend(NotificationBackend):
    """SMS notification backend"""

    def send(self, recipient, message):
        from twilio.rest import Client
        client = Client(account_sid, auth_token)
        client.messages.create(
            to=recipient,
            from_='+1234567890',
            body=message['body']
        )

class NotificationService:
    """Service with dependency injection"""

    def __init__(self, backend: NotificationBackend):
        self.backend = backend

    def notify_order_placed(self, order):
        message = {
            'subject': f'Order #{order.id} Placed',
            'body': f'Your order has been placed. Total: ${order.total}'
        }
        self.backend.send(order.user.email, message)

# Usage
email_backend = EmailBackend()
notification_service = NotificationService(email_backend)
notification_service.notify_order_placed(order)
```

### 2. Service Factory Pattern

```python
# services/service_factory.py
from apps.blog.services import PostService
from apps.ecommerce.services import OrderService
from apps.payments.services import PaymentService

class ServiceFactory:
    """Centralized service access"""

    _post_service = None
    _order_service = None
    _payment_service = None

    @classmethod
    def get_post_service(cls):
        if cls._post_service is None:
            cls._post_service = PostService()
        return cls._post_service

    @classmethod
    def get_order_service(cls):
        if cls._order_service is None:
            cls._order_service = OrderService()
        return cls._order_service

    @classmethod
    def get_payment_service(cls):
        if cls._payment_service is None:
            cls._payment_service = PaymentService()
        return cls._payment_service

# Usage
post_service = ServiceFactory.get_post_service()
post = post_service.create_post(...)
```

### 3. Testing Services

```python
# tests/test_order_service.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from apps.ecommerce.services import OrderService
from apps.ecommerce.models import Product, Order
from apps.core.exceptions import OutOfStockError

User = get_user_model()

class OrderServiceTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com'
        )

        self.product = Product.objects.create(
            name='Test Product',
            price=10.00,
            stock=5
        )

    def test_create_order_success(self):
        """Test successful order creation"""
        items = [{'product_id': self.product.id, 'quantity': 2}]

        order = OrderService.create_order(self.user, items)

        self.assertEqual(order.user, self.user)
        self.assertEqual(order.total, 20.00)
        self.assertEqual(order.items.count(), 1)

        # Check stock updated
        self.product.refresh_from_db()
        self.assertEqual(self.product.stock, 3)

    def test_create_order_out_of_stock(self):
        """Test order fails when out of stock"""
        items = [{'product_id': self.product.id, 'quantity': 10}]

        with self.assertRaises(OutOfStockError):
            OrderService.create_order(self.user, items)

        # Check stock not changed
        self.product.refresh_from_db()
        self.assertEqual(self.product.stock, 5)

        # Check no order created
        self.assertEqual(Order.objects.count(), 0)

    def test_cancel_order(self):
        """Test order cancellation restores inventory"""
        items = [{'product_id': self.product.id, 'quantity': 2}]
        order = OrderService.create_order(self.user, items)

        # Cancel order
        OrderService.cancel_order(order.id, self.user)

        # Check stock restored
        self.product.refresh_from_db()
        self.assertEqual(self.product.stock, 5)

        # Check order cancelled
        order.refresh_from_db()
        self.assertEqual(order.status, 'cancelled')
```

## ❌ Common Mistakes

### 1. Putting Everything in Services

```python
# ❌ Bad: Simple CRUD in service (unnecessary)
class PostService:
    @staticmethod
    def get_post_by_id(post_id):
        return Post.objects.get(id=post_id)

# ✅ Good: Use ORM directly for simple queries
post = Post.objects.get(id=post_id)

# Services are for COMPLEX business logic
```

### 2. Services Depending on Views

```python
# ❌ Bad: Service depends on request
class PostService:
    @staticmethod
    def create_post(request):  # Don't pass request!
        title = request.POST['title']
        ...

# ✅ Good: Service is independent of HTTP
class PostService:
    @staticmethod
    def create_post(title, content, author):
        ...
```

## ❓ Interview Questions

### Q1: What is the Service Layer pattern and why use it?

**Answer**:
The Service Layer separates business logic from views, making code:

- **Reusable**: Same logic in API, admin, CLI
- **Testable**: Test without HTTP requests
- **Maintainable**: Business logic in one place
- **Transaction-safe**: Use @transaction.atomic

### Q2: When should you use a Service Layer?

**Answer**:
Use services for:

- Complex business logic spanning multiple models
- Transaction management
- External API integration
- Logic reused across views/tasks/commands

Don't use for simple CRUD (use ORM directly).

## 📚 Summary

**Key Takeaways**:

1. Services encapsulate business logic
2. Keep views thin (HTTP handling only)
3. Use `@transaction.atomic` in services
4. Services are independent of HTTP (no request objects)
5. Test services without views
6. Use dependency injection for flexibility
7. Don't put simple CRUD in services
8. Services can call other services
9. Raise custom exceptions for error handling
10. Services make code reusable across API/admin/CLI

The Service Layer is essential for maintainable Django applications!
