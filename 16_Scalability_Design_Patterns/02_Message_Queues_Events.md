# 📨 Message Queues and Event-Driven Architecture

## Overview

Building scalable, decoupled systems using message queues and event-driven patterns for asynchronous processing and communication.

---

## Message Queue Basics

### RabbitMQ with Pika

```python
import pika
import json
from typing import Callable, Any
import logging

logger = logging.getLogger(__name__)

class RabbitMQClient:
    """RabbitMQ client for publishing and consuming messages"""

    def __init__(self, host: str = 'localhost', port: int = 5672):
        """Initialize connection to RabbitMQ"""
        self.connection_params = pika.ConnectionParameters(
            host=host,
            port=port,
            heartbeat=600,
            blocked_connection_timeout=300
        )
        self.connection = None
        self.channel = None

    def connect(self):
        """Establish connection"""
        self.connection = pika.BlockingConnection(self.connection_params)
        self.channel = self.connection.channel()

    def close(self):
        """Close connection"""
        if self.connection and not self.connection.is_closed:
            self.connection.close()

    def declare_queue(self, queue_name: str, durable: bool = True):
        """Declare queue"""
        self.channel.queue_declare(
            queue=queue_name,
            durable=durable,  # Survive broker restart
            exclusive=False,
            auto_delete=False
        )

    def publish(
        self,
        queue_name: str,
        message: dict,
        persistent: bool = True
    ):
        """Publish message to queue"""
        if not self.channel:
            self.connect()

        self.channel.basic_publish(
            exchange='',
            routing_key=queue_name,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2 if persistent else 1,  # Persist to disk
                content_type='application/json'
            )
        )

        logger.info(f"Published message to {queue_name}: {message}")

    def consume(
        self,
        queue_name: str,
        callback: Callable,
        auto_ack: bool = False
    ):
        """Consume messages from queue"""
        if not self.channel:
            self.connect()

        self.declare_queue(queue_name)

        def on_message(ch, method, properties, body):
            """Message handler"""
            try:
                message = json.loads(body)
                logger.info(f"Received message: {message}")

                # Process message
                callback(message)

                # Acknowledge message
                if not auto_ack:
                    ch.basic_ack(delivery_tag=method.delivery_tag)

            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # Reject and requeue
                ch.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=True
                )

        self.channel.basic_qos(prefetch_count=1)  # One at a time
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=on_message,
            auto_ack=auto_ack
        )

        logger.info(f"Waiting for messages on {queue_name}")
        self.channel.start_consuming()

# Usage - Publisher
client = RabbitMQClient()
client.connect()

# Publish order processing task
client.publish('order_queue', {
    'order_id': 123,
    'user_id': 456,
    'action': 'process'
})

client.close()

# Usage - Consumer
def process_order(message: dict):
    """Process order from queue"""
    order_id = message['order_id']
    print(f"Processing order {order_id}")
    # Actual processing logic here

consumer = RabbitMQClient()
consumer.connect()
consumer.consume('order_queue', process_order)
```

### Celery for Task Queues

```python
from celery import Celery
from datetime import timedelta
import time

# Initialize Celery
app = Celery(
    'tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

# Configure
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    task_track_started=True,
    task_time_limit=300,  # 5 minutes
    task_soft_time_limit=240,  # 4 minutes warning
)

# Define tasks
@app.task(bind=True, max_retries=3)
def process_payment(self, order_id: int, amount: float):
    """Process payment asynchronously"""
    try:
        logger.info(f"Processing payment for order {order_id}")

        # Simulate payment processing
        time.sleep(2)

        # Payment logic here
        result = {
            'order_id': order_id,
            'amount': amount,
            'status': 'success'
        }

        return result

    except Exception as e:
        logger.error(f"Payment failed: {e}")

        # Retry with exponential backoff
        raise self.retry(exc=e, countdown=2 ** self.request.retries)

@app.task
def send_email(to: str, subject: str, body: str):
    """Send email asynchronously"""
    logger.info(f"Sending email to {to}")
    # Email sending logic
    return {'status': 'sent', 'to': to}

@app.task
def generate_report(user_id: int, report_type: str):
    """Generate report asynchronously"""
    logger.info(f"Generating {report_type} report for user {user_id}")
    # Report generation logic
    return {'status': 'completed', 'url': '/reports/123.pdf'}

# Schedule periodic tasks
from celery.schedules import crontab

app.conf.beat_schedule = {
    'cleanup-every-night': {
        'task': 'tasks.cleanup_old_data',
        'schedule': crontab(hour=2, minute=0),  # 2 AM daily
    },
    'send-daily-summary': {
        'task': 'tasks.send_daily_summary',
        'schedule': crontab(hour=9, minute=0),  # 9 AM daily
    },
    'check-every-5-minutes': {
        'task': 'tasks.health_check',
        'schedule': timedelta(minutes=5),
    },
}

@app.task
def cleanup_old_data():
    """Cleanup old data - runs nightly"""
    logger.info("Running cleanup task")
    # Cleanup logic

# Usage in FastAPI
from fastapi import FastAPI, BackgroundTasks

api = FastAPI()

@api.post("/orders/{order_id}/process")
async def process_order(order_id: int, amount: float):
    """Process order asynchronously"""
    # Queue payment processing
    task = process_payment.delay(order_id, amount)

    return {
        'message': 'Payment processing started',
        'task_id': task.id
    }

@api.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    """Check task status"""
    task = app.AsyncResult(task_id)

    if task.ready():
        return {
            'status': 'completed',
            'result': task.result
        }
    else:
        return {
            'status': task.state,
            'progress': task.info
        }

# Chain tasks
from celery import chain

# Execute tasks in sequence
workflow = chain(
    process_payment.s(123, 100.00),
    send_email.s('user@example.com', 'Payment Confirmed', 'Thank you'),
    generate_report.s(456, 'invoice')
)

result = workflow.apply_async()

# Group tasks (parallel execution)
from celery import group

parallel_tasks = group(
    send_email.s('user1@example.com', 'Subject', 'Body'),
    send_email.s('user2@example.com', 'Subject', 'Body'),
    send_email.s('user3@example.com', 'Subject', 'Body'),
)

result = parallel_tasks.apply_async()
```

---

## Event-Driven Architecture

### Event Bus with Redis Pub/Sub

```python
import redis
import json
from typing import Callable, Dict
import threading

class EventBus:
    """Event bus using Redis Pub/Sub"""

    def __init__(self, redis_url: str = 'redis://localhost:6379'):
        """Initialize event bus"""
        self.redis_client = redis.from_url(redis_url)
        self.pubsub = self.redis_client.pubsub()
        self.handlers: Dict[str, list] = {}
        self._listening = False

    def publish(self, event_type: str, data: dict):
        """Publish event"""
        event = {
            'type': event_type,
            'data': data,
            'timestamp': time.time()
        }

        self.redis_client.publish(
            f'events:{event_type}',
            json.dumps(event)
        )

        logger.info(f"Published event: {event_type}")

    def subscribe(self, event_type: str, handler: Callable):
        """Subscribe to event"""
        if event_type not in self.handlers:
            self.handlers[event_type] = []
            self.pubsub.subscribe(f'events:{event_type}')

        self.handlers[event_type].append(handler)

        # Start listening if not already
        if not self._listening:
            self._start_listening()

    def _start_listening(self):
        """Start listening for events"""
        self._listening = True

        def listen():
            for message in self.pubsub.listen():
                if message['type'] == 'message':
                    self._handle_message(message)

        thread = threading.Thread(target=listen, daemon=True)
        thread.start()

    def _handle_message(self, message):
        """Handle received message"""
        try:
            event = json.loads(message['data'])
            event_type = event['type']

            if event_type in self.handlers:
                for handler in self.handlers[event_type]:
                    try:
                        handler(event['data'])
                    except Exception as e:
                        logger.error(f"Handler error: {e}")

        except Exception as e:
            logger.error(f"Message handling error: {e}")

# Usage
event_bus = EventBus()

# Define event handlers
def on_order_created(data: dict):
    """Handle order created event"""
    order_id = data['order_id']
    print(f"Order created: {order_id}")

    # Send confirmation email
    send_email(data['user_email'], 'Order Confirmed', f'Order {order_id}')

def on_payment_processed(data: dict):
    """Handle payment processed event"""
    order_id = data['order_id']
    print(f"Payment processed for order: {order_id}")

    # Update inventory
    update_inventory(data['items'])

def on_user_registered(data: dict):
    """Handle user registration"""
    user_id = data['user_id']
    print(f"User registered: {user_id}")

    # Send welcome email
    send_email(data['email'], 'Welcome!', 'Thank you for registering')

# Subscribe to events
event_bus.subscribe('order.created', on_order_created)
event_bus.subscribe('payment.processed', on_payment_processed)
event_bus.subscribe('user.registered', on_user_registered)

# Publish events
event_bus.publish('order.created', {
    'order_id': 123,
    'user_email': 'user@example.com',
    'total': 99.99
})

event_bus.publish('payment.processed', {
    'order_id': 123,
    'amount': 99.99,
    'items': [1, 2, 3]
})
```

### Event Sourcing Pattern

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Any
import json

@dataclass
class Event:
    """Base event class"""
    event_id: str
    event_type: str
    aggregate_id: str
    data: dict
    timestamp: datetime
    version: int

class EventStore:
    """Store and retrieve events"""

    def __init__(self, db_session):
        self.db = db_session

    def append(self, event: Event):
        """Append event to store"""
        event_record = EventRecord(
            event_id=event.event_id,
            event_type=event.event_type,
            aggregate_id=event.aggregate_id,
            data=json.dumps(event.data),
            timestamp=event.timestamp,
            version=event.version
        )

        self.db.add(event_record)
        self.db.commit()

    def get_events(self, aggregate_id: str) -> List[Event]:
        """Get all events for aggregate"""
        records = self.db.query(EventRecord)\
            .filter(EventRecord.aggregate_id == aggregate_id)\
            .order_by(EventRecord.version)\
            .all()

        return [
            Event(
                event_id=r.event_id,
                event_type=r.event_type,
                aggregate_id=r.aggregate_id,
                data=json.loads(r.data),
                timestamp=r.timestamp,
                version=r.version
            )
            for r in records
        ]

class OrderAggregate:
    """Order aggregate rebuilt from events"""

    def __init__(self, order_id: str):
        self.order_id = order_id
        self.status = 'pending'
        self.items = []
        self.total = 0.0
        self.version = 0

    def apply_event(self, event: Event):
        """Apply event to rebuild state"""
        if event.event_type == 'OrderCreated':
            self.status = 'created'
            self.items = event.data['items']
            self.total = event.data['total']

        elif event.event_type == 'PaymentProcessed':
            self.status = 'paid'

        elif event.event_type == 'OrderShipped':
            self.status = 'shipped'
            self.tracking_number = event.data['tracking_number']

        elif event.event_type == 'OrderCancelled':
            self.status = 'cancelled'

        self.version = event.version

    def load_from_history(self, events: List[Event]):
        """Rebuild state from events"""
        for event in events:
            self.apply_event(event)

    @classmethod
    def load(cls, order_id: str, event_store: EventStore):
        """Load aggregate from event store"""
        aggregate = cls(order_id)
        events = event_store.get_events(order_id)
        aggregate.load_from_history(events)
        return aggregate

# Usage
event_store = EventStore(db_session)

# Create order (append events)
order_created = Event(
    event_id=str(uuid.uuid4()),
    event_type='OrderCreated',
    aggregate_id='order-123',
    data={'items': [1, 2, 3], 'total': 99.99},
    timestamp=datetime.utcnow(),
    version=1
)
event_store.append(order_created)

# Process payment
payment_processed = Event(
    event_id=str(uuid.uuid4()),
    event_type='PaymentProcessed',
    aggregate_id='order-123',
    data={'amount': 99.99},
    timestamp=datetime.utcnow(),
    version=2
)
event_store.append(payment_processed)

# Rebuild order state from events
order = OrderAggregate.load('order-123', event_store)
print(f"Order status: {order.status}")  # 'paid'
print(f"Order total: {order.total}")     # 99.99
```

---

## CQRS Pattern

### Command Query Responsibility Segregation

```python
from abc import ABC, abstractmethod

# Commands (Write side)
class Command(ABC):
    """Base command"""
    pass

@dataclass
class CreateOrderCommand(Command):
    """Create order command"""
    user_id: int
    items: List[dict]
    total: float

@dataclass
class ProcessPaymentCommand(Command):
    """Process payment command"""
    order_id: str
    amount: float

# Command Handlers
class CommandHandler(ABC):
    """Base command handler"""

    @abstractmethod
    def handle(self, command: Command):
        pass

class CreateOrderHandler(CommandHandler):
    """Handle order creation"""

    def __init__(self, event_store: EventStore, event_bus: EventBus):
        self.event_store = event_store
        self.event_bus = event_bus

    def handle(self, command: CreateOrderCommand):
        """Create order and publish event"""
        order_id = str(uuid.uuid4())

        # Create event
        event = Event(
            event_id=str(uuid.uuid4()),
            event_type='OrderCreated',
            aggregate_id=order_id,
            data={
                'user_id': command.user_id,
                'items': command.items,
                'total': command.total
            },
            timestamp=datetime.utcnow(),
            version=1
        )

        # Store event
        self.event_store.append(event)

        # Publish event
        self.event_bus.publish('order.created', event.data)

        return order_id

# Queries (Read side)
class OrderQueryService:
    """Read-optimized query service"""

    def __init__(self, read_db):
        self.db = read_db

    def get_order(self, order_id: str) -> dict:
        """Get order details"""
        order = self.db.query(OrderView)\
            .filter(OrderView.order_id == order_id)\
            .first()

        return order.to_dict() if order else None

    def get_user_orders(self, user_id: int) -> List[dict]:
        """Get all orders for user"""
        orders = self.db.query(OrderView)\
            .filter(OrderView.user_id == user_id)\
            .all()

        return [order.to_dict() for order in orders]

# Projection (Update read model from events)
class OrderProjection:
    """Project events to read model"""

    def __init__(self, read_db):
        self.db = read_db

    def handle_order_created(self, event: Event):
        """Handle OrderCreated event"""
        order_view = OrderView(
            order_id=event.aggregate_id,
            user_id=event.data['user_id'],
            items=json.dumps(event.data['items']),
            total=event.data['total'],
            status='created',
            created_at=event.timestamp
        )

        self.db.add(order_view)
        self.db.commit()

    def handle_payment_processed(self, event: Event):
        """Handle PaymentProcessed event"""
        order_view = self.db.query(OrderView)\
            .filter(OrderView.order_id == event.aggregate_id)\
            .first()

        if order_view:
            order_view.status = 'paid'
            order_view.updated_at = event.timestamp
            self.db.commit()

# Wire it all together
command_bus = CommandBus()
command_bus.register(CreateOrderCommand, CreateOrderHandler(event_store, event_bus))

query_service = OrderQueryService(read_db)
projection = OrderProjection(read_db)

# Subscribe projection to events
event_bus.subscribe('order.created', projection.handle_order_created)
event_bus.subscribe('payment.processed', projection.handle_payment_processed)

# Usage
# Write: Execute command
command = CreateOrderCommand(
    user_id=123,
    items=[{'id': 1, 'qty': 2}],
    total=99.99
)
order_id = command_bus.dispatch(command)

# Read: Query data
order = query_service.get_order(order_id)
user_orders = query_service.get_user_orders(123)
```

---

## Saga Pattern

### Distributed Transaction Management

```python
from enum import Enum
from typing import List, Callable

class SagaStep:
    """Single step in saga"""

    def __init__(
        self,
        name: str,
        action: Callable,
        compensation: Callable
    ):
        self.name = name
        self.action = action
        self.compensation = compensation

class SagaStatus(Enum):
    """Saga execution status"""
    PENDING = 'pending'
    IN_PROGRESS = 'in_progress'
    COMPLETED = 'completed'
    FAILED = 'failed'
    COMPENSATING = 'compensating'
    COMPENSATED = 'compensated'

class Saga:
    """Saga orchestrator"""

    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.steps: List[SagaStep] = []
        self.completed_steps: List[str] = []
        self.status = SagaStatus.PENDING

    def add_step(self, step: SagaStep):
        """Add step to saga"""
        self.steps.append(step)

    def execute(self) -> bool:
        """Execute saga"""
        self.status = SagaStatus.IN_PROGRESS

        try:
            # Execute each step
            for step in self.steps:
                logger.info(f"Executing step: {step.name}")
                step.action()
                self.completed_steps.append(step.name)

            self.status = SagaStatus.COMPLETED
            return True

        except Exception as e:
            logger.error(f"Saga failed at step {step.name}: {e}")
            self.status = SagaStatus.FAILED
            self._compensate()
            return False

    def _compensate(self):
        """Compensate completed steps"""
        self.status = SagaStatus.COMPENSATING

        # Compensate in reverse order
        for step_name in reversed(self.completed_steps):
            step = next(s for s in self.steps if s.name == step_name)

            try:
                logger.info(f"Compensating step: {step.name}")
                step.compensation()
            except Exception as e:
                logger.error(f"Compensation failed for {step.name}: {e}")

        self.status = SagaStatus.COMPENSATED

# Example: Order processing saga
class OrderSaga:
    """Order processing with saga pattern"""

    def __init__(self, order_id: str):
        self.order_id = order_id
        self.saga = Saga(f"order-saga-{order_id}")

        # Define steps
        self.saga.add_step(SagaStep(
            name='reserve_inventory',
            action=self.reserve_inventory,
            compensation=self.release_inventory
        ))

        self.saga.add_step(SagaStep(
            name='process_payment',
            action=self.process_payment,
            compensation=self.refund_payment
        ))

        self.saga.add_step(SagaStep(
            name='create_shipment',
            action=self.create_shipment,
            compensation=self.cancel_shipment
        ))

        self.saga.add_step(SagaStep(
            name='send_confirmation',
            action=self.send_confirmation,
            compensation=self.send_cancellation
        ))

    def reserve_inventory(self):
        """Reserve inventory"""
        logger.info(f"Reserving inventory for order {self.order_id}")
        # Call inventory service

    def release_inventory(self):
        """Release inventory (compensation)"""
        logger.info(f"Releasing inventory for order {self.order_id}")
        # Call inventory service

    def process_payment(self):
        """Process payment"""
        logger.info(f"Processing payment for order {self.order_id}")
        # Call payment service
        # if payment_fails:
        #     raise Exception("Payment failed")

    def refund_payment(self):
        """Refund payment (compensation)"""
        logger.info(f"Refunding payment for order {self.order_id}")
        # Call payment service

    def create_shipment(self):
        """Create shipment"""
        logger.info(f"Creating shipment for order {self.order_id}")
        # Call shipping service

    def cancel_shipment(self):
        """Cancel shipment (compensation)"""
        logger.info(f"Cancelling shipment for order {self.order_id}")
        # Call shipping service

    def send_confirmation(self):
        """Send confirmation email"""
        logger.info(f"Sending confirmation for order {self.order_id}")
        # Send email

    def send_cancellation(self):
        """Send cancellation email (compensation)"""
        logger.info(f"Sending cancellation for order {self.order_id}")
        # Send email

    def execute(self) -> bool:
        """Execute order saga"""
        return self.saga.execute()

# Usage
order_saga = OrderSaga('order-123')
success = order_saga.execute()

if success:
    print("Order processed successfully")
else:
    print("Order processing failed and compensated")
```

---

## Best Practices

### ✅ Do's:

1. **Use message queues** for async processing
2. **Handle failures** with retries and DLQ
3. **Make operations** idempotent
4. **Use event sourcing** for audit trail
5. **Separate read/write** with CQRS
6. **Implement sagas** for distributed transactions
7. **Monitor queue** depth and lag
8. **Set message TTL** to prevent buildup
9. **Use dead letter queues** for failed messages
10. **Version events** for backward compatibility

### ❌ Don'ts:

1. **Don't use queues** for synchronous operations
2. **Don't forget** message acknowledgment
3. **Don't process** same message twice
4. **Don't lose messages** - use persistent queues
5. **Don't ignore** failed messages
6. **Don't create** circular dependencies
7. **Don't put large** payloads in messages

---

## Interview Questions

### Q1: Message queue vs Event bus?

**Answer**:

- **Message Queue**: Point-to-point, guaranteed delivery, order preserved
- **Event Bus**: Pub/sub, multiple subscribers, eventual consistency
- **Queue**: Task distribution, work queues
- **Bus**: Event notification, decoupled services
  Use both for different needs.

### Q2: What is event sourcing?

**Answer**: Store all changes as events:

- **Event log**: Immutable sequence of events
- **Rebuild state**: Replay events
- **Audit trail**: Complete history
- **Time travel**: State at any point
  Good for audit, debugging, analytics.

### Q3: Explain CQRS pattern.

**Answer**: Separate read/write models:

- **Commands**: Modify state (write)
- **Queries**: Read data (read)
- **Benefits**: Optimize each independently, scale separately
- **Trade-off**: Eventual consistency
  Use for complex domains.

### Q4: What is saga pattern?

**Answer**: Distributed transactions:

- **Sequence of steps**: Each with compensation
- **Rollback**: Compensate on failure
- **No 2PC**: Eventual consistency
- **Choreography vs Orchestration**: Events vs coordinator
  Essential for microservices.

### Q5: How to ensure message idempotency?

**Answer**:

- **Message ID**: Check if already processed
- **Database constraint**: Unique constraint on message ID
- **State check**: Only process if in expected state
- **Idempotent operations**: Same result if repeated
  Always assume duplicate messages possible.

---

## Summary

Event-driven essentials:

- **Message Queues**: RabbitMQ, Celery for async tasks
- **Event Bus**: Redis Pub/Sub for events
- **Event Sourcing**: Store events, rebuild state
- **CQRS**: Separate read/write models
- **Saga**: Distributed transactions with compensation
- **Idempotency**: Handle duplicates gracefully

Decouple and scale! 📨
