# Django Microservices Architecture

## 📖 Concept Explanation

Microservices architecture breaks a monolithic Django application into small, independent services that communicate via APIs or message queues. Each service owns its domain logic and data.

### Monolith vs Microservices

**Monolith** (Single Django Project):

```
myproject/
├── apps/
│   ├── users/
│   ├── products/
│   ├── orders/
│   ├── payments/
│   └── notifications/
└── Single database
```

**Microservices** (Multiple Django Projects):

```
user-service/          # Django project for users
├── Handles: Authentication, profiles
└── Database: user_db

product-service/       # Django project for products
├── Handles: Product catalog, inventory
└── Database: product_db

order-service/         # Django project for orders
├── Handles: Order management
└── Database: order_db

payment-service/       # Django project for payments
├── Handles: Payment processing
└── Database: payment_db

notification-service/  # Django project for notifications
├── Handles: Email/SMS notifications
└── Database: notification_db
```

**Benefits**:

- **Independent deployment**: Update one service without affecting others
- **Technology flexibility**: Use different databases per service
- **Scalability**: Scale services independently
- **Team autonomy**: Teams own services end-to-end
- **Fault isolation**: Service failure doesn't crash entire system

**Drawbacks**:

- **Complexity**: Network calls, distributed transactions
- **Data consistency**: No ACID transactions across services
- **Testing difficulty**: Integration tests require multiple services
- **Operational overhead**: Multiple deployments, monitoring

## 🧠 Service Boundaries

### 1. Identify Service Boundaries (Domain-Driven Design)

```python
# Monolith models (tightly coupled)
class User(models.Model):
    username = models.CharField(max_length=150)
    email = models.EmailField()

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)

# Microservices (decoupled by domain)
# user-service/models.py
class User(models.Model):
    user_id = models.UUIDField(primary_key=True)  # Use UUID for cross-service references
    username = models.CharField(max_length=150)
    email = models.EmailField()

# product-service/models.py
class Product(models.Model):
    product_id = models.UUIDField(primary_key=True)
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)

# order-service/models.py
class Order(models.Model):
    order_id = models.UUIDField(primary_key=True)
    user_id = models.UUIDField()  # Reference to user-service
    product_id = models.UUIDField()  # Reference to product-service
    total = models.DecimalField(max_digits=10, decimal_places=2)
```

### 2. Service Communication Patterns

#### A. Synchronous (REST APIs)

```python
# order-service/services/order_service.py
import requests
from django.conf import settings

class OrderService:
    """Order service communicating with other services via REST"""

    USER_SERVICE_URL = settings.USER_SERVICE_URL
    PRODUCT_SERVICE_URL = settings.PRODUCT_SERVICE_URL
    PAYMENT_SERVICE_URL = settings.PAYMENT_SERVICE_URL

    @staticmethod
    def create_order(user_id, product_id, quantity):
        """Create order with validation from other services"""

        # 1. Validate user exists
        user_response = requests.get(
            f"{OrderService.USER_SERVICE_URL}/api/users/{user_id}/",
            timeout=5
        )
        if user_response.status_code != 200:
            raise ValidationError("User not found")

        user_data = user_response.json()

        # 2. Validate product and check stock
        product_response = requests.get(
            f"{OrderService.PRODUCT_SERVICE_URL}/api/products/{product_id}/",
            timeout=5
        )
        if product_response.status_code != 200:
            raise ValidationError("Product not found")

        product_data = product_response.json()
        if product_data['stock'] < quantity:
            raise ValidationError("Insufficient stock")

        # 3. Reserve stock
        reserve_response = requests.post(
            f"{OrderService.PRODUCT_SERVICE_URL}/api/products/{product_id}/reserve/",
            json={'quantity': quantity},
            timeout=5
        )
        if reserve_response.status_code != 200:
            raise ValidationError("Failed to reserve stock")

        # 4. Create order
        total = product_data['price'] * quantity
        order = Order.objects.create(
            user_id=user_id,
            product_id=product_id,
            quantity=quantity,
            total=total,
            status='pending'
        )

        # 5. Process payment
        try:
            payment_response = requests.post(
                f"{OrderService.PAYMENT_SERVICE_URL}/api/payments/",
                json={
                    'order_id': str(order.order_id),
                    'amount': str(total),
                    'user_id': str(user_id)
                },
                timeout=10
            )

            if payment_response.status_code == 200:
                order.status = 'paid'
                order.save()
            else:
                # Rollback: Release reserved stock
                requests.post(
                    f"{OrderService.PRODUCT_SERVICE_URL}/api/products/{product_id}/release/",
                    json={'quantity': quantity}
                )
                raise ValidationError("Payment failed")

        except requests.RequestException as e:
            # Handle service unavailable
            order.status = 'failed'
            order.save()
            raise ServiceUnavailableError("Payment service unavailable")

        return order
```

#### B. Asynchronous (Message Queue)

```python
# Common: Use RabbitMQ or Kafka for async communication

# order-service/events/order_events.py
import pika
import json
from django.conf import settings

class OrderEventPublisher:
    """Publish order events to RabbitMQ"""

    def __init__(self):
        connection = pika.BlockingConnection(
            pika.ConnectionParameters(settings.RABBITMQ_HOST)
        )
        self.channel = connection.channel()
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )

    def publish_order_created(self, order):
        """Publish order.created event"""
        event = {
            'event_type': 'order.created',
            'order_id': str(order.order_id),
            'user_id': str(order.user_id),
            'product_id': str(order.product_id),
            'total': str(order.total),
            'timestamp': order.created_at.isoformat()
        }

        self.channel.basic_publish(
            exchange='orders',
            routing_key='order.created',
            body=json.dumps(event),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent message
            )
        )

    def publish_order_paid(self, order):
        """Publish order.paid event"""
        event = {
            'event_type': 'order.paid',
            'order_id': str(order.order_id),
            'user_id': str(order.user_id),
            'total': str(order.total),
            'timestamp': order.updated_at.isoformat()
        }

        self.channel.basic_publish(
            exchange='orders',
            routing_key='order.paid',
            body=json.dumps(event)
        )

# notification-service/consumers/order_consumer.py
import pika
import json
from django.core.mail import send_mail

class OrderEventConsumer:
    """Consume order events from RabbitMQ"""

    def __init__(self):
        connection = pika.BlockingConnection(
            pika.ConnectionParameters('rabbitmq')
        )
        self.channel = connection.channel()

        # Declare exchange
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )

        # Declare queue
        result = self.channel.queue_declare(
            queue='notification_orders',
            durable=True
        )
        queue_name = result.method.queue

        # Bind queue to exchange
        self.channel.queue_bind(
            exchange='orders',
            queue=queue_name,
            routing_key='order.*'
        )

        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=self.on_message,
            auto_ack=False
        )

    def on_message(self, ch, method, properties, body):
        """Handle incoming order events"""
        try:
            event = json.loads(body)
            event_type = event['event_type']

            if event_type == 'order.created':
                self.handle_order_created(event)
            elif event_type == 'order.paid':
                self.handle_order_paid(event)

            ch.basic_ack(delivery_tag=method.delivery_tag)

        except Exception as e:
            # Log error and reject message
            print(f"Error processing message: {e}")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

    def handle_order_created(self, event):
        """Send order confirmation email"""
        # Get user email from user-service
        user_id = event['user_id']
        user_response = requests.get(f"{USER_SERVICE_URL}/api/users/{user_id}/")
        user_email = user_response.json()['email']

        send_mail(
            subject=f"Order {event['order_id']} Created",
            message=f"Your order has been created. Total: ${event['total']}",
            from_email='noreply@example.com',
            recipient_list=[user_email]
        )

    def handle_order_paid(self, event):
        """Send payment confirmation email"""
        user_id = event['user_id']
        user_response = requests.get(f"{USER_SERVICE_URL}/api/users/{user_id}/")
        user_email = user_response.json()['email']

        send_mail(
            subject=f"Order {event['order_id']} Paid",
            message=f"Payment received. Total: ${event['total']}",
            from_email='noreply@example.com',
            recipient_list=[user_email]
        )

    def start(self):
        """Start consuming messages"""
        print("Waiting for order events...")
        self.channel.start_consuming()

# Run consumer
# python manage.py consume_order_events
```

## 🎯 API Gateway Pattern

```python
# api-gateway/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
import requests

class APIGateway:
    """Single entry point for all services"""

    USER_SERVICE = 'http://user-service:8001'
    PRODUCT_SERVICE = 'http://product-service:8002'
    ORDER_SERVICE = 'http://order-service:8003'

    @classmethod
    def forward_request(cls, service_url, path, method='GET', **kwargs):
        """Forward request to service"""
        url = f"{service_url}{path}"

        try:
            if method == 'GET':
                response = requests.get(url, **kwargs)
            elif method == 'POST':
                response = requests.post(url, **kwargs)
            elif method == 'PUT':
                response = requests.put(url, **kwargs)
            elif method == 'DELETE':
                response = requests.delete(url, **kwargs)

            return response

        except requests.RequestException as e:
            return None

class UserAPIView(APIView):
    """Proxy to user service"""

    def get(self, request, user_id=None):
        if user_id:
            path = f'/api/users/{user_id}/'
        else:
            path = '/api/users/'

        response = APIGateway.forward_request(
            APIGateway.USER_SERVICE,
            path,
            method='GET',
            params=request.query_params
        )

        if response and response.status_code == 200:
            return Response(response.json())

        return Response({'error': 'Service unavailable'}, status=503)

class OrderAPIView(APIView):
    """Proxy to order service with data aggregation"""

    def get(self, request, order_id):
        """Get order with user and product details"""

        # 1. Get order from order-service
        order_response = APIGateway.forward_request(
            APIGateway.ORDER_SERVICE,
            f'/api/orders/{order_id}/',
            method='GET'
        )

        if not order_response or order_response.status_code != 200:
            return Response({'error': 'Order not found'}, status=404)

        order_data = order_response.json()

        # 2. Get user details from user-service
        user_response = APIGateway.forward_request(
            APIGateway.USER_SERVICE,
            f'/api/users/{order_data["user_id"]}/',
            method='GET'
        )

        # 3. Get product details from product-service
        product_response = APIGateway.forward_request(
            APIGateway.PRODUCT_SERVICE,
            f'/api/products/{order_data["product_id"]}/',
            method='GET'
        )

        # 4. Aggregate data
        result = {
            'order': order_data,
            'user': user_response.json() if user_response else None,
            'product': product_response.json() if product_response else None
        }

        return Response(result)
```

## ✅ Best Practices

### 1. Database Per Service

```python
# user-service/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'user_db',
        'USER': 'user_service',
        'PASSWORD': 'password',
        'HOST': 'user-db',
        'PORT': '5432',
    }
}

# order-service/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'order_db',
        'USER': 'order_service',
        'PASSWORD': 'password',
        'HOST': 'order-db',
        'PORT': '5432',
    }
}

# Each service owns its data
# No direct database access across services
```

### 2. Circuit Breaker Pattern

```python
# circuit_breaker.py
from datetime import datetime, timedelta
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"  # Normal operation
    OPEN = "open"  # Requests blocked
    HALF_OPEN = "half_open"  # Testing recovery

class CircuitBreaker:
    """Prevent cascading failures"""

    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout  # seconds
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker"""

        if self.state == CircuitState.OPEN:
            # Check if timeout expired
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpenError("Circuit breaker is open")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result

        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        """Reset on successful call"""
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        """Increment failure count"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_attempt_reset(self):
        """Check if enough time passed since last failure"""
        return (
            self.last_failure_time and
            datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout)
        )

# Usage
user_service_breaker = CircuitBreaker(failure_threshold=5, timeout=60)

def get_user(user_id):
    def _fetch():
        response = requests.get(f"{USER_SERVICE_URL}/api/users/{user_id}/", timeout=5)
        response.raise_for_status()
        return response.json()

    try:
        return user_service_breaker.call(_fetch)
    except CircuitBreakerOpenError:
        # Return cached data or default
        return get_cached_user(user_id)
```

### 3. Service Discovery

```python
# service_registry.py
from typing import Dict, List
import consul

class ServiceRegistry:
    """Service discovery with Consul"""

    def __init__(self, consul_host='localhost', consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)

    def register_service(self, name: str, host: str, port: int):
        """Register service with Consul"""
        self.consul.agent.service.register(
            name=name,
            service_id=f"{name}-{host}-{port}",
            address=host,
            port=port,
            check=consul.Check.http(
                f"http://{host}:{port}/health/",
                interval="10s",
                timeout="5s"
            )
        )

    def get_service(self, name: str) -> List[Dict]:
        """Get available instances of service"""
        _, services = self.consul.health.service(name, passing=True)

        return [
            {
                'host': service['Service']['Address'],
                'port': service['Service']['Port']
            }
            for service in services
        ]

    def deregister_service(self, service_id: str):
        """Deregister service"""
        self.consul.agent.service.deregister(service_id)

# Usage in settings
from service_registry import ServiceRegistry

registry = ServiceRegistry()

# Get user service instances
user_services = registry.get_service('user-service')
if user_services:
    USER_SERVICE_URL = f"http://{user_services[0]['host']}:{user_services[0]['port']}"
```

## ❌ Common Mistakes

### 1. Distributed Monolith

```python
# ❌ Bad: Services tightly coupled
# order-service depends on user-service schema
class OrderService:
    def create_order(self, user_data):
        # Directly using user-service schema
        if user_data['role'] == 'premium':  # Coupled to user-service
            discount = 0.1
        ...

# ✅ Good: Services loosely coupled
class OrderService:
    def create_order(self, user_id):
        # Get discount from user-service API
        response = requests.get(f"{USER_SERVICE}/api/users/{user_id}/discount/")
        discount = response.json()['discount']
        ...
```

### 2. Ignoring Network Failures

```python
# ❌ Bad: No error handling
response = requests.get(f"{USER_SERVICE}/api/users/{user_id}/")
user = response.json()  # Crashes if service down

# ✅ Good: Handle failures gracefully
try:
    response = requests.get(
        f"{USER_SERVICE}/api/users/{user_id}/",
        timeout=5
    )
    response.raise_for_status()
    user = response.json()
except requests.RequestException:
    # Use cached data or return error
    user = get_cached_user(user_id)
```

## ❓ Interview Questions

### Q1: When should you use microservices?

**Answer**:
Use microservices when:

- Large team (20+ developers)
- Need independent scaling
- Different parts use different technologies
- Need independent deployments

Don't use for small teams or simple apps (monolith is simpler).

### Q2: How do you handle transactions across microservices?

**Answer**:
Use **Saga pattern**:

1. Break transaction into steps
2. Each service completes its step
3. If step fails, execute compensating transactions (rollback)
4. Use event-driven architecture (publish events)

### Q3: What's the difference between orchestration and choreography?

**Answer**:

- **Orchestration**: Central coordinator tells services what to do (order-service tells payment-service to charge)
- **Choreography**: Services listen to events and react (payment-service listens for order.created event)

Choreography is more decoupled but harder to trace.

## 📚 Summary

**Key Takeaways**:

1. Microservices split monolith into independent services
2. Each service owns its domain and database
3. Use REST APIs for synchronous communication
4. Use message queues for asynchronous events
5. Implement API Gateway for single entry point
6. Use Circuit Breaker to prevent cascading failures
7. Each service should be independently deployable
8. Handle network failures gracefully
9. Use Saga pattern for distributed transactions
10. Microservices add complexity—use when needed

Microservices are powerful but complex—only use when monolith limitations are clear!
