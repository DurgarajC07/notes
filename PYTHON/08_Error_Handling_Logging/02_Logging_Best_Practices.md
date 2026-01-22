# 📝 Logging Best Practices

## Overview

Proper logging is essential for monitoring, debugging, and maintaining production applications. Understanding logging levels, handlers, and structured logging ensures effective observability.

---

## Logging Basics

### Python Logging Module

```python
import logging

# Basic configuration
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)

logger = logging.getLogger(__name__)

# Logging levels (lowest to highest)
logger.debug("Detailed information for debugging")      # DEBUG (10)
logger.info("General information")                      # INFO (20)
logger.warning("Warning message")                       # WARNING (30)
logger.error("Error occurred")                          # ERROR (40)
logger.critical("Critical error, application may stop") # CRITICAL (50)

# Only messages at or above configured level are logged
# With level=INFO, debug messages are ignored

# Log with exception info
try:
    result = 10 / 0
except ZeroDivisionError:
    logger.exception("Division by zero occurred")  # Includes traceback

# Alternative: exc_info parameter
try:
    result = 10 / 0
except ZeroDivisionError as e:
    logger.error("Division error", exc_info=True)

# Format string interpolation
user_id = 123
action = "login"

# ❌ BAD: String concatenation (always evaluated)
logger.info("User " + str(user_id) + " performed " + action)

# ✅ GOOD: Lazy evaluation (only if logged)
logger.info("User %s performed %s", user_id, action)

# ✅ GOOD: f-strings (Python 3.6+, but evaluated immediately)
logger.info(f"User {user_id} performed {action}")
```

---

## Logger Configuration

### Advanced Logger Setup

```python
import logging
import sys
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

def setup_logger(name, log_file, level=logging.INFO):
    """Configure logger with file and console handlers"""

    # Create logger
    logger = logging.getLogger(name)
    logger.setLevel(level)

    # Prevent duplicate handlers
    if logger.handlers:
        return logger

    # Formatter
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - [%(filename)s:%(lineno)d] - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )

    # Console handler (stdout)
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)

    # File handler with rotation
    file_handler = RotatingFileHandler(
        log_file,
        maxBytes=10 * 1024 * 1024,  # 10 MB
        backupCount=5,               # Keep 5 backup files
        encoding='utf-8'
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)

    return logger

# Usage
logger = setup_logger('myapp', 'app.log')
logger.info("Application started")

# Time-based rotation
def setup_daily_logger(name, log_dir):
    """Logger with daily rotation"""
    logger = logging.getLogger(name)

    handler = TimedRotatingFileHandler(
        f"{log_dir}/{name}.log",
        when='midnight',      # Rotate at midnight
        interval=1,           # Every day
        backupCount=30,       # Keep 30 days
        encoding='utf-8'
    )

    formatter = logging.Formatter(
        '%(asctime)s - %(levelname)s - %(message)s'
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger
```

---

## Structured Logging

### JSON Logging with structlog

```python
import structlog

# Configure structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Structured logging with context
logger.info("User login", user_id=123, ip="192.168.1.1")
# Output: {"event": "User login", "user_id": 123, "ip": "192.168.1.1", "timestamp": "..."}

# Bind context to logger
request_logger = logger.bind(
    request_id="abc-123",
    user_id=456
)

request_logger.info("Processing request")
request_logger.info("Request completed")
# Both logs include request_id and user_id

# Context manager for temporary context
def process_order(order_id):
    """Process order with context"""
    log = logger.bind(order_id=order_id)

    log.info("Order processing started")

    try:
        # Process order
        log.info("Payment processed", amount=99.99)
        log.info("Order confirmed")
    except Exception as e:
        log.error("Order failed", error=str(e), exc_info=True)
        raise

# Custom processor
def add_app_context(logger, method_name, event_dict):
    """Add application context to logs"""
    event_dict['app'] = 'myapp'
    event_dict['env'] = 'production'
    return event_dict

# Add to processors
structlog.configure(
    processors=[
        add_app_context,
        # ... other processors
    ]
)
```

---

## Logging in FastAPI

### Request Logging Middleware

```python
from fastapi import FastAPI, Request
import time
import logging
import uuid

app = FastAPI()

# Configure logger
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    """Log all HTTP requests"""
    # Generate request ID
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id

    # Log request
    logger.info(
        "Request started",
        extra={
            "request_id": request_id,
            "method": request.method,
            "path": request.url.path,
            "client": request.client.host
        }
    )

    # Process request
    start_time = time.time()

    try:
        response = await call_next(request)

        # Calculate duration
        duration = time.time() - start_time

        # Log response
        logger.info(
            "Request completed",
            extra={
                "request_id": request_id,
                "status_code": response.status_code,
                "duration": f"{duration:.3f}s"
            }
        )

        # Add request ID to response headers
        response.headers["X-Request-ID"] = request_id

        return response

    except Exception as e:
        # Log error
        logger.error(
            "Request failed",
            extra={
                "request_id": request_id,
                "error": str(e)
            },
            exc_info=True
        )
        raise

# Dependency for request logger
def get_request_logger(request: Request):
    """Get logger with request context"""
    return logger.getChild(request.state.request_id)

# Use in endpoints
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    request_logger: logging.Logger = Depends(get_request_logger)
):
    """Get user with logging"""
    request_logger.info(f"Fetching user {user_id}")

    try:
        user = db.query(User).filter(User.id == user_id).first()

        if not user:
            request_logger.warning(f"User {user_id} not found")
            raise HTTPException(404, "User not found")

        request_logger.info(f"User {user_id} found")
        return user

    except Exception as e:
        request_logger.error(f"Error fetching user {user_id}", exc_info=True)
        raise
```

---

## Logging Patterns

### Context-Aware Logging

```python
from contextvars import ContextVar
import logging

# Context variable for request ID
request_id_ctx: ContextVar[str] = ContextVar('request_id', default=None)

class ContextFilter(logging.Filter):
    """Add context to log records"""

    def filter(self, record):
        record.request_id = request_id_ctx.get()
        return True

# Configure logger with context filter
logger = logging.getLogger(__name__)
context_filter = ContextFilter()
logger.addFilter(context_filter)

# Formatter with context
formatter = logging.Formatter(
    '%(asctime)s - [%(request_id)s] - %(levelname)s - %(message)s'
)

# Set context in request
async def process_request(request_id: str):
    """Process request with context"""
    # Set context
    request_id_ctx.set(request_id)

    # All logs in this context include request_id
    logger.info("Processing started")
    logger.info("Processing completed")

# Decorator for logging
def log_execution(func):
    """Decorator to log function execution"""
    import functools

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        logger.info(f"Executing {func.__name__}")

        try:
            result = func(*args, **kwargs)
            logger.info(f"{func.__name__} completed successfully")
            return result

        except Exception as e:
            logger.error(
                f"{func.__name__} failed: {e}",
                exc_info=True
            )
            raise

    return wrapper

@log_execution
def process_data(data):
    """Process data with automatic logging"""
    return data.upper()
```

---

## Log Aggregation

### Logging to External Services

```python
import logging
from pythonjsonlogger import jsonlogger

# JSON formatter for log aggregation
class CustomJsonFormatter(jsonlogger.JsonFormatter):
    """Custom JSON formatter"""

    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)

        # Add custom fields
        log_record['app'] = 'myapp'
        log_record['environment'] = 'production'
        log_record['version'] = '1.0.0'

# Configure JSON logging
def setup_json_logging():
    """Setup JSON logging for aggregation"""
    logger = logging.getLogger()

    handler = logging.StreamHandler()
    formatter = CustomJsonFormatter(
        '%(timestamp)s %(level)s %(name)s %(message)s',
        timestamp=True
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger

# Send logs to Elasticsearch
from elasticsearch import Elasticsearch

class ElasticsearchHandler(logging.Handler):
    """Send logs to Elasticsearch"""

    def __init__(self, es_host, index_name):
        super().__init__()
        self.es = Elasticsearch([es_host])
        self.index_name = index_name

    def emit(self, record):
        """Send log record to Elasticsearch"""
        try:
            log_entry = {
                'timestamp': record.created,
                'level': record.levelname,
                'logger': record.name,
                'message': record.getMessage(),
                'module': record.module,
                'function': record.funcName,
                'line': record.lineno
            }

            # Add exception info if present
            if record.exc_info:
                log_entry['exception'] = self.format(record)

            # Index to Elasticsearch
            self.es.index(
                index=self.index_name,
                document=log_entry
            )

        except Exception:
            self.handleError(record)

# Send logs to CloudWatch (AWS)
import boto3
from logging.handlers import BufferingHandler

class CloudWatchHandler(BufferingHandler):
    """Send logs to AWS CloudWatch"""

    def __init__(self, log_group, log_stream, capacity=100):
        super().__init__(capacity)
        self.client = boto3.client('logs')
        self.log_group = log_group
        self.log_stream = log_stream

    def flush(self):
        """Flush logs to CloudWatch"""
        if not self.buffer:
            return

        events = [
            {
                'timestamp': int(record.created * 1000),
                'message': self.format(record)
            }
            for record in self.buffer
        ]

        try:
            self.client.put_log_events(
                logGroupName=self.log_group,
                logStreamName=self.log_stream,
                logEvents=events
            )
        except Exception as e:
            print(f"Failed to send logs to CloudWatch: {e}")
        finally:
            self.buffer = []
```

---

## Performance Considerations

### Efficient Logging

```python
import logging

logger = logging.getLogger(__name__)

# ❌ INEFFICIENT: String formatting always executed
logger.debug("Processing " + str(len(items)) + " items with data: " + str(data))

# ✅ EFFICIENT: Lazy evaluation (only if logged)
logger.debug("Processing %d items with data: %s", len(items), data)

# Check level before expensive operations
if logger.isEnabledFor(logging.DEBUG):
    expensive_data = generate_debug_info()  # Only called if DEBUG enabled
    logger.debug("Debug info: %s", expensive_data)

# Sampling for high-frequency logs
import random

def log_sample(logger, message, sample_rate=0.01):
    """Log only a sample of messages"""
    if random.random() < sample_rate:
        logger.info(message)

# Use in high-frequency code
for item in millions_of_items:
    log_sample(logger, f"Processing item {item}", sample_rate=0.001)  # Log 0.1%

# Async logging for better performance
from logging.handlers import QueueHandler, QueueListener
import queue

def setup_async_logging():
    """Setup async logging with queue"""
    log_queue = queue.Queue(-1)  # Unlimited queue

    # Queue handler (non-blocking)
    queue_handler = QueueHandler(log_queue)

    # Actual handlers (run in separate thread)
    file_handler = RotatingFileHandler('app.log', maxBytes=10*1024*1024)
    console_handler = logging.StreamHandler()

    # Queue listener (processes logs asynchronously)
    listener = QueueListener(
        log_queue,
        file_handler,
        console_handler,
        respect_handler_level=True
    )

    # Start listener
    listener.start()

    # Configure root logger
    root = logging.getLogger()
    root.addHandler(queue_handler)
    root.setLevel(logging.INFO)

    return listener

# Usage
listener = setup_async_logging()
logger.info("Logged asynchronously")  # Non-blocking
```

---

## Security and Compliance

### Sensitive Data Handling

```python
import re
import logging

class SensitiveDataFilter(logging.Filter):
    """Filter sensitive data from logs"""

    PATTERNS = {
        'email': re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'),
        'ssn': re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
        'credit_card': re.compile(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b'),
        'api_key': re.compile(r'\b[A-Za-z0-9]{32,}\b'),
        'password': re.compile(r'password["\']?\s*[:=]\s*["\']?([^"\'\s]+)', re.IGNORECASE)
    }

    def filter(self, record):
        """Redact sensitive data"""
        message = record.getMessage()

        # Redact each pattern
        for name, pattern in self.PATTERNS.items():
            message = pattern.sub(f'[REDACTED-{name.upper()}]', message)

        # Update record
        record.msg = message
        record.args = ()

        return True

# Apply filter
logger = logging.getLogger(__name__)
logger.addFilter(SensitiveDataFilter())

# Now sensitive data is redacted
logger.info("User email: user@example.com")  # User email: [REDACTED-EMAIL]
logger.info("Password: secret123")           # Password: [REDACTED-PASSWORD]

# GDPR-compliant logging
class GDPRLogger:
    """Logger with GDPR compliance"""

    def __init__(self, logger):
        self.logger = logger

    def log_personal_data(self, level, message, user_id=None, **kwargs):
        """Log with user consent tracking"""
        extra = {
            'user_id': user_id,
            'data_type': 'personal',
            'consent': True  # Track consent
        }
        extra.update(kwargs)

        self.logger.log(level, message, extra=extra)

    def log_anonymized(self, level, message, **kwargs):
        """Log anonymized data"""
        extra = {'data_type': 'anonymized'}
        extra.update(kwargs)

        self.logger.log(level, message, extra=extra)
```

---

## Best Practices

### ✅ Do's:

1. **Use appropriate levels** (DEBUG for dev, INFO+ for prod)
2. **Include context** (request ID, user ID, timestamps)
3. **Use structured logging** (JSON for aggregation)
4. **Log exceptions** with traceback
5. **Rotate log files** to prevent disk full
6. **Sanitize sensitive** data (passwords, tokens)
7. **Use lazy evaluation** for expensive formatting
8. **Monitor log volume** in production
9. **Centralize logs** for distributed systems
10. **Set retention policies**

### ❌ Don'ts:

1. **Don't log sensitive** data (passwords, tokens, PII)
2. **Don't log in loops** excessively (use sampling)
3. **Don't use print()** (use logger)
4. **Don't ignore log levels**
5. **Don't log everything** (noise vs signal)
6. **Don't forget to rotate** logs
7. **Don't block** on logging (use async)

---

## Interview Questions

### Q1: Explain logging levels and when to use each.

**Answer**:

- **DEBUG**: Detailed diagnostic info (development only)
- **INFO**: General events (user actions, system status)
- **WARNING**: Unexpected but handled (deprecated API, fallback)
- **ERROR**: Error occurred but app continues
- **CRITICAL**: Severe error, app may stop
  Production typically uses INFO+.

### Q2: What is structured logging?

**Answer**: Logs as structured data (JSON) instead of plain text:

- **Machine-readable**: Easy parsing and querying
- **Searchable**: Filter by fields (user_id, status_code)
- **Aggregatable**: Analytics and dashboards
- **Contextual**: Attach metadata to logs
  Tools: structlog, python-json-logger.

### Q3: How do you handle sensitive data in logs?

**Answer**:

- **Never log**: Passwords, tokens, full credit cards
- **Redact/mask**: Use filters to remove sensitive patterns
- **Hash**: One-way hash for debugging (user_id)
- **Comply**: GDPR, HIPAA requirements
- **Audit**: Track what's logged about users

### Q4: Explain log rotation.

**Answer**: Prevent logs from filling disk:

- **Size-based**: Rotate when file reaches size (RotatingFileHandler)
- **Time-based**: Rotate daily/weekly (TimedRotatingFileHandler)
- **Keep backups**: Compress old logs, delete oldest
- **Consider**: Log aggregation instead of local files

### Q5: What's the difference between logging and print?

**Answer**:

- **Logging**: Levels, handlers, formatters, production-ready
- **Print**: stdout only, no levels, hard to control
- **Production**: Logging allows filtering, routing, aggregation
- **Development**: Print for quick debug, logging for permanent
  Never use print in production code.

---

## Summary

Logging essentials:

- **Levels**: DEBUG, INFO, WARNING, ERROR, CRITICAL
- **Handlers**: Console, file, rotating, remote
- **Formatters**: Text, JSON, custom
- **Structured logging**: JSON for aggregation
- **Context**: Request ID, user ID, metadata
- **Security**: Redact sensitive data
- **Performance**: Lazy evaluation, sampling, async
- **Aggregation**: Elasticsearch, CloudWatch, etc.

Log thoughtfully for better observability! 📝
