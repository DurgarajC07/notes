# 🔄 Circuit Breaker Pattern

## Overview

Circuit breaker is a design pattern that prevents cascading failures in distributed systems by detecting failures and temporarily blocking requests to failing services.

---

## Problem: Cascading Failures

### The Problem

```python
"""
Scenario: Microservice calling external API

Without Circuit Breaker:
1. Service A calls Service B (which is slow/down)
2. Requests to B timeout after 30 seconds
3. Service A threads/connections get exhausted
4. Service A becomes unavailable
5. Services calling A also fail
6. Entire system cascades into failure

With Circuit Breaker:
1. Detect Service B is failing
2. Stop sending requests to B immediately
3. Return cached/default response
4. Periodically test if B recovered
5. Restore normal operation when B is healthy
"""
```

---

## Circuit Breaker States

### Three States

```python
from enum import Enum
from datetime import datetime, timedelta
import time

class CircuitState(Enum):
    """Circuit breaker states"""
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Blocking requests
    HALF_OPEN = "half_open"  # Testing recovery

"""
State Transitions:

CLOSED (Normal):
- Requests pass through
- Count failures
- If failures > threshold → OPEN

OPEN (Blocking):
- Reject requests immediately
- No calls to service
- After timeout → HALF_OPEN

HALF_OPEN (Testing):
- Allow limited test requests
- If success → CLOSED
- If failure → OPEN
"""
```

---

## Simple Circuit Breaker Implementation

```python
import time
from typing import Callable, Any
from datetime import datetime, timedelta

class CircuitBreaker:
    """
    Simple circuit breaker implementation

    Args:
        failure_threshold: Number of failures to trip circuit
        timeout: Seconds to wait before retry (OPEN state)
        success_threshold: Successes needed to close circuit (HALF_OPEN)
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        timeout: int = 60,
        success_threshold: int = 2
    ):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold

        # State tracking
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

    def call(self, func: Callable, *args, **kwargs) -> Any:
        """
        Execute function with circuit breaker protection

        Raises:
            CircuitOpenError: When circuit is open
        """
        # Check if circuit should transition
        self._check_state_transition()

        # OPEN state - reject immediately
        if self.state == CircuitState.OPEN:
            raise CircuitOpenError("Circuit breaker is OPEN")

        try:
            # Execute function
            result = func(*args, **kwargs)

            # Record success
            self._on_success()

            return result

        except Exception as e:
            # Record failure
            self._on_failure()

            raise e

    def _on_success(self):
        """Handle successful request"""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1

            # Enough successes - close circuit
            if self.success_count >= self.success_threshold:
                self._close_circuit()

        else:
            # Reset failure count
            self.failure_count = 0

    def _on_failure(self):
        """Handle failed request"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()

        if self.state == CircuitState.HALF_OPEN:
            # Failed during testing - reopen
            self._open_circuit()

        elif self.failure_count >= self.failure_threshold:
            # Too many failures - open circuit
            self._open_circuit()

    def _check_state_transition(self):
        """Check if circuit should transition states"""
        if self.state == CircuitState.OPEN:
            # Check if timeout elapsed
            if self.last_failure_time:
                elapsed = datetime.now() - self.last_failure_time

                if elapsed.total_seconds() >= self.timeout:
                    self._half_open_circuit()

    def _close_circuit(self):
        """Transition to CLOSED state"""
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        print("Circuit CLOSED - normal operation resumed")

    def _open_circuit(self):
        """Transition to OPEN state"""
        self.state = CircuitState.OPEN
        self.success_count = 0
        print(f"Circuit OPEN - blocking requests for {self.timeout}s")

    def _half_open_circuit(self):
        """Transition to HALF_OPEN state"""
        self.state = CircuitState.HALF_OPEN
        self.failure_count = 0
        self.success_count = 0
        print("Circuit HALF_OPEN - testing recovery")

class CircuitOpenError(Exception):
    """Raised when circuit breaker is open"""
    pass

# Example usage
def flaky_api_call():
    """Simulate flaky external API"""
    import random

    if random.random() < 0.7:  # 70% failure rate
        raise Exception("API call failed")

    return "Success"

# Create circuit breaker
breaker = CircuitBreaker(
    failure_threshold=3,
    timeout=10,
    success_threshold=2
)

# Use circuit breaker
for i in range(20):
    try:
        result = breaker.call(flaky_api_call)
        print(f"Request {i+1}: {result}")

    except CircuitOpenError:
        print(f"Request {i+1}: Circuit is OPEN - request blocked")

    except Exception as e:
        print(f"Request {i+1}: Failed - {e}")

    time.sleep(1)
```

---

## Production-Ready Circuit Breaker

### With Threading Support

```python
import threading
from typing import Callable, Any, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta
import logging

@dataclass
class CircuitBreakerConfig:
    """Circuit breaker configuration"""
    failure_threshold: int = 5
    timeout: int = 60
    success_threshold: int = 2
    expected_exception: type = Exception

class ThreadSafeCircuitBreaker:
    """
    Thread-safe circuit breaker implementation

    Features:
    - Thread-safe state management
    - Configurable thresholds
    - Metrics tracking
    - Callback hooks
    """

    def __init__(self, config: CircuitBreakerConfig):
        self.config = config

        # State
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time: Optional[datetime] = None

        # Thread safety
        self.lock = threading.Lock()

        # Metrics
        self.total_requests = 0
        self.total_failures = 0
        self.total_successes = 0
        self.circuit_opened_count = 0

        # Logger
        self.logger = logging.getLogger(__name__)

    def call(self, func: Callable, *args, fallback: Optional[Callable] = None, **kwargs) -> Any:
        """
        Execute function with circuit breaker protection

        Args:
            func: Function to execute
            fallback: Fallback function if circuit open
            *args, **kwargs: Arguments for func

        Returns:
            Result from func or fallback

        Raises:
            CircuitOpenError: If circuit open and no fallback
        """
        with self.lock:
            self.total_requests += 1

            # Check state transition
            self._check_transition()

            # Circuit open - use fallback or raise
            if self.state == CircuitState.OPEN:
                if fallback:
                    self.logger.warning("Circuit OPEN - using fallback")
                    return fallback(*args, **kwargs)

                raise CircuitOpenError("Circuit breaker is OPEN")

        # Execute function (outside lock for concurrency)
        try:
            result = func(*args, **kwargs)
            self._record_success()
            return result

        except self.config.expected_exception as e:
            self._record_failure()
            raise e

    def _record_success(self):
        """Record successful call"""
        with self.lock:
            self.total_successes += 1

            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1

                if self.success_count >= self.config.success_threshold:
                    self._close()

            else:
                self.failure_count = 0

    def _record_failure(self):
        """Record failed call"""
        with self.lock:
            self.total_failures += 1
            self.failure_count += 1
            self.last_failure_time = datetime.now()

            if self.state == CircuitState.HALF_OPEN:
                self._open()

            elif self.failure_count >= self.config.failure_threshold:
                self._open()

    def _check_transition(self):
        """Check if state should transition"""
        if self.state == CircuitState.OPEN:
            if self.last_failure_time:
                elapsed = (datetime.now() - self.last_failure_time).total_seconds()

                if elapsed >= self.config.timeout:
                    self._half_open()

    def _close(self):
        """Close circuit"""
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.logger.info("Circuit CLOSED")

    def _open(self):
        """Open circuit"""
        self.state = CircuitState.OPEN
        self.success_count = 0
        self.circuit_opened_count += 1
        self.logger.warning(
            f"Circuit OPEN - failures: {self.failure_count}, "
            f"timeout: {self.config.timeout}s"
        )

    def _half_open(self):
        """Half-open circuit"""
        self.state = CircuitState.HALF_OPEN
        self.failure_count = 0
        self.success_count = 0
        self.logger.info("Circuit HALF_OPEN - testing recovery")

    def get_metrics(self) -> dict:
        """Get circuit breaker metrics"""
        with self.lock:
            return {
                'state': self.state.value,
                'total_requests': self.total_requests,
                'total_successes': self.total_successes,
                'total_failures': self.total_failures,
                'success_rate': (
                    self.total_successes / self.total_requests * 100
                    if self.total_requests > 0 else 0
                ),
                'circuit_opened_count': self.circuit_opened_count,
                'current_failure_count': self.failure_count,
            }
```

---

## Integration with HTTP Clients

### With httpx

```python
import httpx
from typing import Optional

class CircuitBreakerHTTPClient:
    """HTTP client with circuit breaker"""

    def __init__(self, base_url: str, config: Optional[CircuitBreakerConfig] = None):
        self.base_url = base_url
        self.client = httpx.AsyncClient(base_url=base_url, timeout=10.0)

        if config is None:
            config = CircuitBreakerConfig(
                failure_threshold=5,
                timeout=60,
                success_threshold=2,
                expected_exception=httpx.HTTPError
            )

        self.breaker = ThreadSafeCircuitBreaker(config)
        self.logger = logging.getLogger(__name__)

    async def get(self, path: str, **kwargs) -> httpx.Response:
        """GET request with circuit breaker"""
        async def make_request():
            response = await self.client.get(path, **kwargs)
            response.raise_for_status()
            return response

        def fallback():
            self.logger.warning(f"Circuit open for {path} - returning cached response")
            return self._get_cached_response(path)

        return await self.breaker.call(make_request, fallback=fallback)

    async def post(self, path: str, **kwargs) -> httpx.Response:
        """POST request with circuit breaker"""
        async def make_request():
            response = await self.client.post(path, **kwargs)
            response.raise_for_status()
            return response

        return await self.breaker.call(make_request)

    def _get_cached_response(self, path: str):
        """Get cached response (placeholder)"""
        # Implement caching logic
        return None

# Usage
client = CircuitBreakerHTTPClient(
    base_url="https://api.example.com",
    config=CircuitBreakerConfig(failure_threshold=3, timeout=30)
)

try:
    response = await client.get("/users/123")
    print(response.json())

except CircuitOpenError:
    print("Service unavailable - circuit breaker open")
```

### With requests

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

class CircuitBreakerSession:
    """Requests session with circuit breaker"""

    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = self._create_session()
        self.breaker = ThreadSafeCircuitBreaker(
            CircuitBreakerConfig(
                failure_threshold=5,
                timeout=60,
                expected_exception=requests.RequestException
            )
        )

    def _create_session(self):
        """Create session with retry logic"""
        session = requests.Session()

        # Retry configuration
        retry = Retry(
            total=3,
            backoff_factor=0.3,
            status_forcelist=[500, 502, 503, 504]
        )

        adapter = HTTPAdapter(max_retries=retry)
        session.mount('http://', adapter)
        session.mount('https://', adapter)

        return session

    def get(self, path: str, **kwargs):
        """GET with circuit breaker"""
        url = f"{self.base_url}{path}"

        def make_request():
            response = self.session.get(url, **kwargs)
            response.raise_for_status()
            return response

        return self.breaker.call(make_request)

    def post(self, path: str, **kwargs):
        """POST with circuit breaker"""
        url = f"{self.base_url}{path}"

        def make_request():
            response = self.session.post(url, **kwargs)
            response.raise_for_status()
            return response

        return self.breaker.call(make_request)

# Usage
client = CircuitBreakerSession("https://api.example.com")

try:
    response = client.get("/users")
    print(response.json())

except CircuitOpenError:
    print("Circuit breaker open - service unavailable")

except requests.RequestException as e:
    print(f"Request failed: {e}")
```

---

## Multiple Circuit Breakers

### Circuit Breaker Registry

```python
from typing import Dict

class CircuitBreakerRegistry:
    """Manage multiple circuit breakers"""

    def __init__(self):
        self.breakers: Dict[str, ThreadSafeCircuitBreaker] = {}
        self.configs: Dict[str, CircuitBreakerConfig] = {}
        self.lock = threading.Lock()

    def register(self, name: str, config: CircuitBreakerConfig):
        """Register a new circuit breaker"""
        with self.lock:
            self.configs[name] = config
            self.breakers[name] = ThreadSafeCircuitBreaker(config)

    def get(self, name: str) -> ThreadSafeCircuitBreaker:
        """Get circuit breaker by name"""
        if name not in self.breakers:
            # Create default breaker
            with self.lock:
                if name not in self.breakers:
                    config = CircuitBreakerConfig()
                    self.breakers[name] = ThreadSafeCircuitBreaker(config)

        return self.breakers[name]

    def get_all_metrics(self) -> Dict[str, dict]:
        """Get metrics for all circuit breakers"""
        return {
            name: breaker.get_metrics()
            for name, breaker in self.breakers.items()
        }

# Global registry
registry = CircuitBreakerRegistry()

# Register circuit breakers for different services
registry.register('payment_service', CircuitBreakerConfig(
    failure_threshold=3,
    timeout=30,
    success_threshold=2
))

registry.register('user_service', CircuitBreakerConfig(
    failure_threshold=5,
    timeout=60,
    success_threshold=3
))

# Use circuit breakers
payment_breaker = registry.get('payment_service')
user_breaker = registry.get('user_service')

# Monitor all breakers
metrics = registry.get_all_metrics()
print(metrics)
```

---

## FastAPI Integration

### Circuit Breaker Middleware

```python
from fastapi import FastAPI, HTTPException, Request
from starlette.middleware.base import BaseHTTPMiddleware

class CircuitBreakerMiddleware(BaseHTTPMiddleware):
    """Circuit breaker middleware for external services"""

    def __init__(self, app, registry: CircuitBreakerRegistry):
        super().__init__(app)
        self.registry = registry

    async def dispatch(self, request: Request, call_next):
        # Process request normally
        response = await call_next(request)

        # Add circuit breaker status header
        service_name = request.url.path.split('/')[1]  # Extract service name
        breaker = self.registry.get(service_name)

        response.headers['X-Circuit-Breaker-State'] = breaker.state.value

        return response

# Application
app = FastAPI()
registry = CircuitBreakerRegistry()

# Register services
registry.register('payment', CircuitBreakerConfig(failure_threshold=3))
registry.register('inventory', CircuitBreakerConfig(failure_threshold=5))

app.add_middleware(CircuitBreakerMiddleware, registry=registry)

@app.get("/payment/process")
async def process_payment():
    """Process payment with circuit breaker"""
    breaker = registry.get('payment')

    try:
        result = breaker.call(external_payment_api)
        return {"status": "success", "result": result}

    except CircuitOpenError:
        raise HTTPException(
            status_code=503,
            detail="Payment service temporarily unavailable"
        )

@app.get("/metrics/circuit-breakers")
async def get_circuit_breaker_metrics():
    """Get all circuit breaker metrics"""
    return registry.get_all_metrics()
```

---

## Best Practices

### ✅ Do's:

1. **Use for external services** - Protect from cascading failures
2. **Set reasonable thresholds** - Based on SLA and patterns
3. **Provide fallbacks** - Cached data or default responses
4. **Monitor metrics** - Track opens, failures, success rates
5. **Log state transitions** - Debug and alerting
6. **Different configs** per service - One size doesn't fit all
7. **Test failover** - Chaos engineering
8. **Combine with retries** - Retry before opening circuit

### ❌ Don'ts:

1. **Don't use** for internal services (same datacenter)
2. **Don't set** threshold too low (normal failures)
3. **Don't forget** timeout configuration
4. **Don't block** in HALF_OPEN for too long
5. **Don't ignore** metrics and alerts
6. **Don't use** same breaker for all services

---

## Interview Questions

### Q1: What is circuit breaker pattern?

**Answer**: Prevents cascading failures in distributed systems:

- **Problem**: Failing service causes callers to fail
- **Solution**: Detect failures, stop calling, allow recovery
- **States**: CLOSED (normal), OPEN (blocking), HALF_OPEN (testing)
- **Benefits**: Prevents cascades, faster failures, automatic recovery
  Essential for microservices resilience.

### Q2: When to open the circuit?

**Answer**:

- **Threshold**: Number of consecutive failures (e.g., 5)
- **Rate**: Failure rate over time window (e.g., 50% in 1min)
- **Combination**: Both threshold and rate
- **Considerations**: Service SLA, normal error rate, timeout duration
  Tune based on monitoring data.

### Q3: Circuit breaker vs retry?

**Answer**:

- **Retry**: Try again immediately or with backoff
- **Circuit breaker**: Stop trying, fail fast
- **Use both**: Retry few times, then open circuit
- **Retry for**: Transient failures, network blips
- **Circuit for**: Service degradation, cascading failures
  Complementary patterns, not alternatives.

### Q4: What happens in HALF_OPEN state?

**Answer**:

- **Purpose**: Test if service recovered
- **Behavior**: Allow limited test requests
- **Success**: Close circuit, resume normal operation
- **Failure**: Reopen circuit, wait again
- **Duration**: Short, few requests only
  Balance between testing and protecting.

### Q5: How to handle circuit open?

**Answer**:

- **Fallback**: Return cached/default data
- **Fail fast**: Return error immediately (503)
- **Queue**: Store request for later
- **Alternative**: Call backup service
- **User feedback**: Clear error message
  Choose based on criticality and user experience.

---

## Summary

Circuit breaker essentials:

- **Three states**: CLOSED, OPEN, HALF_OPEN
- **Fail fast**: Stop calling failing services
- **Auto recovery**: Test periodically in HALF_OPEN
- **Fallbacks**: Provide degraded functionality
- **Metrics**: Monitor state and transitions
- **Per service**: Different configs for each
- **With retries**: Complementary patterns

Build resilient systems! 🔄
