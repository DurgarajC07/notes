# System Design: Message Queue System

## 📖 Problem Statement

Design a message queue system like RabbitMQ, Apache Kafka, or AWS SQS that:

- Enables asynchronous communication between services
- Guarantees message delivery (at-least-once or exactly-once)
- Handles high throughput (millions of messages/second)
- Supports multiple producers and consumers
- Provides message ordering guarantees
- Allows message persistence and replay
- Scales horizontally

## 🎯 Requirements

### Functional Requirements

1. PUBLISH(topic, message) - Send message to topic
2. SUBSCRIBE(topic, consumer_group) - Consume messages from topic
3. ACKNOWLEDGE(message_id) - Confirm message processed
4. Support message priorities
5. Support delayed/scheduled messages
6. Dead letter queue for failed messages
7. Message filtering and routing

### Non-Functional Requirements

1. **High Throughput**: 100K+ messages/second
2. **Low Latency**: <10ms for message delivery
3. **Durability**: Messages persisted to disk
4. **Availability**: 99.99% uptime
5. **Scalability**: Horizontal scaling (partition data)
6. **Ordering**: FIFO within partition
7. **Delivery Guarantees**: At-least-once or exactly-once

## 📊 Architecture

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│Producer 1│   │Producer 2│   │Producer N│
└─────┬────┘   └─────┬────┘   └─────┬────┘
      │              │              │
      └──────────────┼──────────────┘
                     │
                     ▼
              ┌─────────────┐
              │   Broker    │ ← Load balancer
              └──────┬──────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │Topic A │  │Topic B │  │Topic C │
    │Part 0  │  │Part 0  │  │Part 0  │
    │Part 1  │  │Part 1  │  │Part 1  │
    └───┬────┘  └───┬────┘  └───┬────┘
        │           │           │
        └───────────┼───────────┘
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Consumer │ │Consumer │ │Consumer │
    │Group A  │ │Group B  │ │Group N  │
    └─────────┘ └─────────┘ └─────────┘
```

## 🔑 Core Components

### 1. Simple In-Memory Queue

```python
import queue
import threading
from typing import Any, Callable, Optional
import time
import logging

logger = logging.getLogger(__name__)

class Message:
    """Message wrapper"""

    def __init__(self, id: str, topic: str, data: Any, priority: int = 0):
        self.id = id
        self.topic = topic
        self.data = data
        self.priority = priority
        self.timestamp = time.time()
        self.retry_count = 0
        self.max_retries = 3

class SimpleQueue:
    """
    Simple in-memory message queue

    - Thread-safe
    - Priority support
    - Basic retry mechanism
    """

    def __init__(self, max_size: int = 1000):
        """
        Args:
            max_size: Maximum queue size
        """
        self.queue = queue.PriorityQueue(maxsize=max_size)
        self.subscribers = []
        self.lock = threading.Lock()
        self.running = False

    def publish(self, message: Message):
        """
        Publish message to queue

        Args:
            message: Message to publish
        """
        try:
            # Priority queue: lower number = higher priority
            priority_tuple = (-message.priority, message.timestamp, message)
            self.queue.put(priority_tuple, block=False)
            logger.info(f"Published message {message.id} to queue")
        except queue.Full:
            logger.error(f"Queue full, cannot publish message {message.id}")
            raise Exception("Queue full")

    def subscribe(self, callback: Callable[[Message], None]):
        """
        Subscribe to queue

        Args:
            callback: Function to call for each message
        """
        with self.lock:
            self.subscribers.append(callback)

    def start(self):
        """Start processing messages"""
        self.running = True

        worker_thread = threading.Thread(target=self._process_messages, daemon=True)
        worker_thread.start()

        logger.info("Queue started")

    def stop(self):
        """Stop processing messages"""
        self.running = False
        logger.info("Queue stopped")

    def _process_messages(self):
        """Process messages from queue"""
        while self.running:
            try:
                # Get message from queue (timeout to check running flag)
                priority_tuple = self.queue.get(timeout=1)
                _, _, message = priority_tuple

                # Deliver to all subscribers
                self._deliver_message(message)

                self.queue.task_done()

            except queue.Empty:
                continue
            except Exception as e:
                logger.error(f"Error processing message: {e}")

    def _deliver_message(self, message: Message):
        """Deliver message to subscribers"""
        with self.lock:
            subscribers = self.subscribers.copy()

        for callback in subscribers:
            try:
                callback(message)
                logger.info(f"Delivered message {message.id}")
            except Exception as e:
                logger.error(f"Error delivering message {message.id}: {e}")
                self._handle_delivery_failure(message)

    def _handle_delivery_failure(self, message: Message):
        """Handle message delivery failure"""
        message.retry_count += 1

        if message.retry_count < message.max_retries:
            # Retry with delay
            logger.info(f"Retrying message {message.id} (attempt {message.retry_count})")
            time.sleep(2 ** message.retry_count)  # Exponential backoff
            self.publish(message)
        else:
            # Move to dead letter queue
            logger.error(f"Message {message.id} failed after {message.retry_count} retries")
            # TODO: Send to dead letter queue

# Example usage
def process_order(message: Message):
    """Process order message"""
    print(f"Processing order: {message.data}")
    # Simulate processing
    time.sleep(0.1)

# Create queue
mq = SimpleQueue(max_size=100)
mq.subscribe(process_order)
mq.start()

# Publish messages
msg1 = Message(id="msg1", topic="orders", data={"order_id": 123}, priority=1)
msg2 = Message(id="msg2", topic="orders", data={"order_id": 456}, priority=2)

mq.publish(msg1)
mq.publish(msg2)

time.sleep(2)
mq.stop()
```

### 2. Topic-Based Pub/Sub

```python
from collections import defaultdict
from typing import Callable, List
import threading
import uuid

class Topic:
    """Message topic with multiple consumer groups"""

    def __init__(self, name: str, num_partitions: int = 3):
        """
        Args:
            name: Topic name
            num_partitions: Number of partitions
        """
        self.name = name
        self.num_partitions = num_partitions
        self.partitions = [[] for _ in range(num_partitions)]
        self.consumer_groups = {}
        self.lock = threading.Lock()

    def publish(self, message: Message, partition_key: Optional[str] = None):
        """
        Publish message to topic

        Args:
            message: Message to publish
            partition_key: Key to determine partition (optional)
        """
        # Determine partition
        if partition_key:
            partition = hash(partition_key) % self.num_partitions
        else:
            partition = hash(message.id) % self.num_partitions

        with self.lock:
            self.partitions[partition].append(message)

        logger.info(f"Published message {message.id} to partition {partition}")

    def subscribe(self, consumer_group: str, callback: Callable[[Message], None]):
        """
        Subscribe consumer group to topic

        Args:
            consumer_group: Consumer group name
            callback: Function to call for each message
        """
        with self.lock:
            if consumer_group not in self.consumer_groups:
                self.consumer_groups[consumer_group] = {
                    'callback': callback,
                    'offsets': [0] * self.num_partitions,  # Track position per partition
                    'thread': None
                }

        # Start consumer thread
        consumer = self.consumer_groups[consumer_group]
        consumer['thread'] = threading.Thread(
            target=self._consume_messages,
            args=(consumer_group,),
            daemon=True
        )
        consumer['thread'].start()

    def _consume_messages(self, consumer_group: str):
        """Consume messages for consumer group"""
        while True:
            with self.lock:
                consumer = self.consumer_groups.get(consumer_group)
                if not consumer:
                    break

                # Round-robin across partitions
                for partition_idx in range(self.num_partitions):
                    offset = consumer['offsets'][partition_idx]
                    partition = self.partitions[partition_idx]

                    if offset < len(partition):
                        message = partition[offset]

                        try:
                            # Deliver message
                            consumer['callback'](message)

                            # Update offset
                            consumer['offsets'][partition_idx] = offset + 1

                            logger.info(
                                f"Consumed message {message.id} from partition {partition_idx} "
                                f"by group {consumer_group}"
                            )
                        except Exception as e:
                            logger.error(f"Error consuming message {message.id}: {e}")

            time.sleep(0.1)  # Polling interval

class MessageBroker:
    """
    Message broker with multiple topics

    - Supports pub/sub pattern
    - Multiple consumer groups per topic
    - Partition-based parallelism
    """

    def __init__(self):
        self.topics = {}
        self.lock = threading.Lock()

    def create_topic(self, topic_name: str, num_partitions: int = 3):
        """Create new topic"""
        with self.lock:
            if topic_name not in self.topics:
                self.topics[topic_name] = Topic(topic_name, num_partitions)
                logger.info(f"Created topic: {topic_name}")

    def publish(self, topic_name: str, data: Any, partition_key: Optional[str] = None):
        """
        Publish message to topic

        Args:
            topic_name: Topic name
            data: Message data
            partition_key: Key to determine partition
        """
        with self.lock:
            if topic_name not in self.topics:
                self.create_topic(topic_name)

            topic = self.topics[topic_name]

        message = Message(
            id=str(uuid.uuid4()),
            topic=topic_name,
            data=data
        )

        topic.publish(message, partition_key)

    def subscribe(self, topic_name: str, consumer_group: str, callback: Callable[[Message], None]):
        """
        Subscribe to topic

        Args:
            topic_name: Topic name
            consumer_group: Consumer group name
            callback: Function to call for each message
        """
        with self.lock:
            if topic_name not in self.topics:
                raise ValueError(f"Topic {topic_name} does not exist")

            topic = self.topics[topic_name]

        topic.subscribe(consumer_group, callback)
        logger.info(f"Subscribed group {consumer_group} to topic {topic_name}")

# Example usage
broker = MessageBroker()

# Create topic
broker.create_topic("orders", num_partitions=3)

# Subscribe consumers
def process_order_group1(message: Message):
    print(f"[Group 1] Processing: {message.data}")

def process_order_group2(message: Message):
    print(f"[Group 2] Processing: {message.data}")

broker.subscribe("orders", "analytics", process_order_group1)
broker.subscribe("orders", "notifications", process_order_group2)

# Publish messages
for i in range(10):
    broker.publish(
        "orders",
        {"order_id": i, "amount": 100 * i},
        partition_key=f"user_{i % 3}"  # Partition by user
    )

time.sleep(5)
```

### 3. RabbitMQ Integration

```python
import pika
import json
from typing import Callable
import threading

class RabbitMQProducer:
    """RabbitMQ message producer"""

    def __init__(self, host: str = 'localhost', exchange: str = 'orders'):
        """
        Args:
            host: RabbitMQ host
            exchange: Exchange name
        """
        self.host = host
        self.exchange = exchange

        # Connect to RabbitMQ
        connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=self.host)
        )
        self.channel = connection.channel()

        # Declare exchange
        self.channel.exchange_declare(
            exchange=self.exchange,
            exchange_type='topic',
            durable=True
        )

    def publish(self, routing_key: str, message: dict):
        """
        Publish message

        Args:
            routing_key: Routing key (e.g., 'order.created')
            message: Message data
        """
        self.channel.basic_publish(
            exchange=self.exchange,
            routing_key=routing_key,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                content_type='application/json'
            )
        )
        logger.info(f"Published message to {routing_key}")

    def close(self):
        """Close connection"""
        self.channel.close()

class RabbitMQConsumer:
    """RabbitMQ message consumer"""

    def __init__(
        self,
        host: str = 'localhost',
        exchange: str = 'orders',
        queue: str = 'order_processing',
        routing_keys: List[str] = None
    ):
        """
        Args:
            host: RabbitMQ host
            exchange: Exchange name
            queue: Queue name
            routing_keys: Routing keys to bind (e.g., ['order.*'])
        """
        self.host = host
        self.exchange = exchange
        self.queue = queue
        self.routing_keys = routing_keys or ['#']

        # Connect to RabbitMQ
        connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=self.host)
        )
        self.channel = connection.channel()

        # Declare exchange
        self.channel.exchange_declare(
            exchange=self.exchange,
            exchange_type='topic',
            durable=True
        )

        # Declare queue
        self.channel.queue_declare(queue=self.queue, durable=True)

        # Bind queue to exchange
        for routing_key in self.routing_keys:
            self.channel.queue_bind(
                exchange=self.exchange,
                queue=self.queue,
                routing_key=routing_key
            )

        # Set prefetch count (QoS)
        self.channel.basic_qos(prefetch_count=10)

    def consume(self, callback: Callable[[dict], None]):
        """
        Start consuming messages

        Args:
            callback: Function to call for each message
        """
        def on_message(ch, method, properties, body):
            """Handle incoming message"""
            try:
                message = json.loads(body)
                callback(message)

                # Acknowledge message
                ch.basic_ack(delivery_tag=method.delivery_tag)

            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # Reject and requeue
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        self.channel.basic_consume(
            queue=self.queue,
            on_message_callback=on_message
        )

        logger.info(f"Started consuming from queue: {self.queue}")
        self.channel.start_consuming()

    def close(self):
        """Close connection"""
        self.channel.close()

# Example usage
# Producer
producer = RabbitMQProducer(host='localhost', exchange='orders')
producer.publish('order.created', {'order_id': 123, 'amount': 99.99})
producer.publish('order.shipped', {'order_id': 123, 'tracking': 'ABC123'})
producer.close()

# Consumer
def handle_order(message: dict):
    print(f"Received order: {message}")

consumer = RabbitMQConsumer(
    host='localhost',
    exchange='orders',
    queue='order_processing',
    routing_keys=['order.*']  # Subscribe to all order events
)

# Start consuming in separate thread
consumer_thread = threading.Thread(target=consumer.consume, args=(handle_order,), daemon=True)
consumer_thread.start()
```

## ❓ Interview Questions

### Q1: What's the difference between queue and pub/sub?

**Answer**:

- **Queue**: Point-to-point, one consumer receives each message
- **Pub/Sub**: Broadcast, multiple subscribers receive each message
- **Use queue for**: Task distribution (load balancing)
- **Use pub/sub for**: Event broadcasting (notifications)

### Q2: How do you guarantee exactly-once delivery?

**Answer**:

1. **Idempotent processing**: Design consumers to handle duplicates
2. **Deduplication**: Track processed message IDs
3. **Transactional outbox**: Write to database and queue atomically
4. **Two-phase commit**: Coordinate distributed transactions

Most systems use **at-least-once** + **idempotency**.

### Q3: How do you handle message ordering?

**Answer**:

- **Single partition**: All messages in order (low throughput)
- **Partition by key**: Order within partition (e.g., user_id)
- **Sequence numbers**: Add sequence to messages, consumer reorders
- **Trade-off**: Ordering vs parallelism

## 📚 Summary

**Key Takeaways**:

1. **Queue vs Pub/Sub**: Choose based on use case
2. **Partitioning**: Enable parallel processing
3. **Consumer Groups**: Scale consumers independently
4. **Acknowledgments**: Ensure message processed before removing
5. **Dead Letter Queue**: Handle failures gracefully
6. **Idempotency**: Design for at-least-once delivery
7. **Ordering**: Partition by key for ordered processing
8. **Persistence**: Write to disk for durability
9. **Monitoring**: Track lag, throughput, errors
10. **Backpressure**: Handle slow consumers

Message queues enable scalable, decoupled architectures!
