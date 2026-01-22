# System Design: Load Balancer

## 📖 Problem Statement

Design a load balancer system like HAProxy, NGINX, or AWS ELB that:

- Distributes incoming traffic across multiple servers
- Performs health checks on backend servers
- Handles millions of requests per second
- Provides session persistence (sticky sessions)
- Supports multiple load balancing algorithms
- Detects and removes unhealthy servers automatically
- Provides SSL termination and HTTPS support

## 🎯 Requirements

### Functional Requirements

1. Distribute requests across backend servers
2. Health check backend servers periodically
3. Remove unhealthy servers from rotation
4. Support multiple load balancing algorithms
5. Session persistence for stateful applications
6. SSL/TLS termination
7. Request routing based on URL/headers
8. Rate limiting per client

### Non-Functional Requirements

1. **High Throughput**: Handle 100K+ requests/second
2. **Low Latency**: <5ms overhead
3. **High Availability**: 99.99% uptime (active-passive setup)
4. **Scalability**: Horizontal scaling
5. **Fault Tolerance**: Automatic failover
6. **Security**: DDoS protection, SSL support

## 📊 Architecture

```
                    ┌─────────────┐
                    │   Clients   │
                    └──────┬──────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Load Balancer 1 │  ← Active
                  │   (Primary)     │
                  └────────┬────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │Server 1 │  │Server 2 │  │Server 3 │
        │(Healthy)│  │(Healthy)│  │(Failed) │
        └─────────┘  └─────────┘  └─────────┘
              │            │
              └────────────┼────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Load Balancer 2 │  ← Passive (Standby)
                  │   (Backup)      │
                  └─────────────────┘
```

## 🔑 Load Balancing Algorithms

### 1. Round Robin

```python
import threading
from typing import List, Optional

class Server:
    """Backend server representation"""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.healthy = True
        self.active_connections = 0
        self.total_requests = 0
        self.failed_requests = 0

    def __str__(self):
        return f"{self.host}:{self.port}"

    @property
    def address(self):
        return f"{self.host}:{self.port}"

class RoundRobinLoadBalancer:
    """
    Round Robin load balancer

    - Distributes requests evenly across servers
    - Simple and fair distribution
    - Doesn't consider server load
    """

    def __init__(self, servers: List[Server]):
        """
        Args:
            servers: List of backend servers
        """
        self.servers = servers
        self.current_index = 0
        self.lock = threading.Lock()

    def get_next_server(self) -> Optional[Server]:
        """
        Get next server using round-robin

        Returns:
            Next healthy server or None if all unhealthy
        """
        with self.lock:
            # Try each server once
            for _ in range(len(self.servers)):
                server = self.servers[self.current_index]
                self.current_index = (self.current_index + 1) % len(self.servers)

                if server.healthy:
                    return server

            return None  # No healthy servers

# Example usage
servers = [
    Server("192.168.1.1", 8080),
    Server("192.168.1.2", 8080),
    Server("192.168.1.3", 8080)
]

lb = RoundRobinLoadBalancer(servers)

for i in range(9):
    server = lb.get_next_server()
    print(f"Request {i} -> {server}")
```

### 2. Least Connections

```python
class LeastConnectionsLoadBalancer:
    """
    Least Connections load balancer

    - Routes to server with fewest active connections
    - Better for long-lived connections
    - Adapts to server load
    """

    def __init__(self, servers: List[Server]):
        """
        Args:
            servers: List of backend servers
        """
        self.servers = servers
        self.lock = threading.Lock()

    def get_next_server(self) -> Optional[Server]:
        """
        Get server with least connections

        Returns:
            Server with fewest connections or None
        """
        with self.lock:
            healthy_servers = [s for s in self.servers if s.healthy]

            if not healthy_servers:
                return None

            # Find server with minimum connections
            return min(healthy_servers, key=lambda s: s.active_connections)

# Example usage
servers = [
    Server("192.168.1.1", 8080),
    Server("192.168.1.2", 8080),
    Server("192.168.1.3", 8080)
]

lb = LeastConnectionsLoadBalancer(servers)

# Simulate connections
servers[0].active_connections = 10
servers[1].active_connections = 5
servers[2].active_connections = 8

server = lb.get_next_server()
print(f"Route to: {server} (connections: {server.active_connections})")
```

### 3. Weighted Round Robin

```python
class WeightedRoundRobinLoadBalancer:
    """
    Weighted Round Robin load balancer

    - Servers assigned weights based on capacity
    - Higher weight = more requests
    - Good for heterogeneous server pools
    """

    def __init__(self, servers: List[Server], weights: List[int]):
        """
        Args:
            servers: List of backend servers
            weights: Weight for each server
        """
        self.servers = servers
        self.weights = weights
        self.current_index = 0
        self.current_weight = 0
        self.max_weight = max(weights)
        self.gcd_weight = self._gcd_list(weights)
        self.lock = threading.Lock()

    def _gcd(self, a: int, b: int) -> int:
        """Calculate greatest common divisor"""
        while b:
            a, b = b, a % b
        return a

    def _gcd_list(self, numbers: List[int]) -> int:
        """Calculate GCD of list"""
        result = numbers[0]
        for num in numbers[1:]:
            result = self._gcd(result, num)
        return result

    def get_next_server(self) -> Optional[Server]:
        """
        Get next server using weighted round-robin

        Returns:
            Next healthy server based on weights
        """
        with self.lock:
            attempts = 0
            max_attempts = len(self.servers) * self.max_weight

            while attempts < max_attempts:
                self.current_index = (self.current_index + 1) % len(self.servers)

                if self.current_index == 0:
                    self.current_weight = self.current_weight - self.gcd_weight
                    if self.current_weight <= 0:
                        self.current_weight = self.max_weight

                server = self.servers[self.current_index]

                if server.healthy and self.weights[self.current_index] >= self.current_weight:
                    return server

                attempts += 1

            return None

# Example usage
servers = [
    Server("192.168.1.1", 8080),  # High capacity
    Server("192.168.1.2", 8080),  # Medium capacity
    Server("192.168.1.3", 8080)   # Low capacity
]

weights = [5, 3, 1]  # Server 1 gets 5x traffic of Server 3

lb = WeightedRoundRobinLoadBalancer(servers, weights)

for i in range(9):
    server = lb.get_next_server()
    print(f"Request {i} -> {server}")
```

### 4. IP Hash (Session Persistence)

```python
import hashlib

class IPHashLoadBalancer:
    """
    IP Hash load balancer

    - Routes same client IP to same server
    - Provides session persistence (sticky sessions)
    - Good for stateful applications
    """

    def __init__(self, servers: List[Server]):
        """
        Args:
            servers: List of backend servers
        """
        self.servers = servers
        self.lock = threading.Lock()

    def _hash_ip(self, client_ip: str) -> int:
        """Hash client IP to determine server"""
        hash_value = hashlib.md5(client_ip.encode()).hexdigest()
        return int(hash_value, 16)

    def get_server_for_ip(self, client_ip: str) -> Optional[Server]:
        """
        Get server for client IP

        Args:
            client_ip: Client IP address

        Returns:
            Server for this IP or None
        """
        with self.lock:
            healthy_servers = [s for s in self.servers if s.healthy]

            if not healthy_servers:
                return None

            # Hash IP to server index
            hash_value = self._hash_ip(client_ip)
            index = hash_value % len(healthy_servers)

            return healthy_servers[index]

# Example usage
servers = [
    Server("192.168.1.1", 8080),
    Server("192.168.1.2", 8080),
    Server("192.168.1.3", 8080)
]

lb = IPHashLoadBalancer(servers)

# Same client IP always goes to same server
client_ip = "203.0.113.45"
for i in range(5):
    server = lb.get_server_for_ip(client_ip)
    print(f"Client {client_ip} -> {server}")
```

## 🔍 Health Checks

```python
import requests
import time
import threading
from typing import Callable

class HealthChecker:
    """
    Health checker for backend servers

    - Periodically checks server health
    - Marks servers as healthy/unhealthy
    - Supports HTTP, TCP, and custom checks
    """

    def __init__(
        self,
        servers: List[Server],
        check_interval: int = 10,
        timeout: int = 5,
        unhealthy_threshold: int = 3
    ):
        """
        Args:
            servers: List of servers to monitor
            check_interval: Seconds between checks
            timeout: Request timeout in seconds
            unhealthy_threshold: Failures before marking unhealthy
        """
        self.servers = servers
        self.check_interval = check_interval
        self.timeout = timeout
        self.unhealthy_threshold = unhealthy_threshold
        self.failure_counts = {s: 0 for s in servers}
        self.running = False

    def start(self):
        """Start health checking"""
        self.running = True

        thread = threading.Thread(target=self._check_loop, daemon=True)
        thread.start()

        print("Health checker started")

    def stop(self):
        """Stop health checking"""
        self.running = False
        print("Health checker stopped")

    def _check_loop(self):
        """Main health check loop"""
        while self.running:
            for server in self.servers:
                self._check_server(server)

            time.sleep(self.check_interval)

    def _check_server(self, server: Server):
        """
        Check if server is healthy

        Args:
            server: Server to check
        """
        try:
            # HTTP health check
            response = requests.get(
                f"http://{server.host}:{server.port}/health",
                timeout=self.timeout
            )

            if response.status_code == 200:
                # Server healthy
                self.failure_counts[server] = 0

                if not server.healthy:
                    server.healthy = True
                    print(f"Server {server} marked as HEALTHY")
            else:
                self._mark_failure(server)

        except Exception as e:
            self._mark_failure(server)

    def _mark_failure(self, server: Server):
        """Mark server check failure"""
        self.failure_counts[server] += 1

        if self.failure_counts[server] >= self.unhealthy_threshold:
            if server.healthy:
                server.healthy = False
                print(f"Server {server} marked as UNHEALTHY")

# Example usage
servers = [
    Server("192.168.1.1", 8080),
    Server("192.168.1.2", 8080),
    Server("192.168.1.3", 8080)
]

health_checker = HealthChecker(
    servers=servers,
    check_interval=10,
    unhealthy_threshold=3
)

health_checker.start()
```

## 🎯 Complete Load Balancer Implementation

```python
import socket
import threading
import logging
from typing import Tuple

logger = logging.getLogger(__name__)

class LoadBalancer:
    """
    Complete HTTP load balancer

    - Accepts client connections
    - Proxies requests to backend servers
    - Performs health checks
    - Supports multiple algorithms
    """

    def __init__(
        self,
        listen_host: str = "0.0.0.0",
        listen_port: int = 8000,
        servers: List[Server] = None,
        algorithm: str = "round_robin"
    ):
        """
        Args:
            listen_host: Host to listen on
            listen_port: Port to listen on
            servers: Backend servers
            algorithm: Load balancing algorithm
        """
        self.listen_host = listen_host
        self.listen_port = listen_port
        self.servers = servers or []

        # Choose algorithm
        if algorithm == "round_robin":
            self.balancer = RoundRobinLoadBalancer(self.servers)
        elif algorithm == "least_connections":
            self.balancer = LeastConnectionsLoadBalancer(self.servers)
        elif algorithm == "ip_hash":
            self.balancer = IPHashLoadBalancer(self.servers)
        else:
            raise ValueError(f"Unknown algorithm: {algorithm}")

        # Health checker
        self.health_checker = HealthChecker(self.servers)

        self.running = False
        self.server_socket = None

    def start(self):
        """Start load balancer"""
        self.running = True

        # Start health checker
        self.health_checker.start()

        # Create server socket
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((self.listen_host, self.listen_port))
        self.server_socket.listen(100)

        logger.info(f"Load balancer listening on {self.listen_host}:{self.listen_port}")

        # Accept connections
        while self.running:
            try:
                client_socket, client_address = self.server_socket.accept()

                # Handle in separate thread
                thread = threading.Thread(
                    target=self._handle_client,
                    args=(client_socket, client_address),
                    daemon=True
                )
                thread.start()

            except Exception as e:
                if self.running:
                    logger.error(f"Error accepting connection: {e}")

    def stop(self):
        """Stop load balancer"""
        self.running = False
        self.health_checker.stop()

        if self.server_socket:
            self.server_socket.close()

        logger.info("Load balancer stopped")

    def _handle_client(self, client_socket: socket.socket, client_address: Tuple):
        """
        Handle client connection

        Args:
            client_socket: Client socket
            client_address: Client address tuple
        """
        try:
            # Get backend server
            if isinstance(self.balancer, IPHashLoadBalancer):
                server = self.balancer.get_server_for_ip(client_address[0])
            else:
                server = self.balancer.get_next_server()

            if not server:
                # No healthy servers
                error_response = (
                    "HTTP/1.1 503 Service Unavailable\r\n"
                    "Content-Type: text/plain\r\n"
                    "Content-Length: 21\r\n"
                    "\r\n"
                    "No servers available"
                )
                client_socket.sendall(error_response.encode())
                return

            server.active_connections += 1
            server.total_requests += 1

            # Connect to backend
            backend_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            backend_socket.connect((server.host, server.port))

            # Proxy request
            request = client_socket.recv(4096)
            backend_socket.sendall(request)

            # Proxy response
            response = backend_socket.recv(4096)
            client_socket.sendall(response)

            backend_socket.close()
            server.active_connections -= 1

            logger.info(f"Proxied request to {server}")

        except Exception as e:
            logger.error(f"Error handling client: {e}")
            if server:
                server.failed_requests += 1
                server.active_connections -= 1

        finally:
            client_socket.close()

# Example usage
servers = [
    Server("192.168.1.1", 8080),
    Server("192.168.1.2", 8080),
    Server("192.168.1.3", 8080)
]

lb = LoadBalancer(
    listen_host="0.0.0.0",
    listen_port=8000,
    servers=servers,
    algorithm="round_robin"
)

# Start load balancer
lb.start()
```

## ❓ Interview Questions

### Q1: How do you ensure high availability of load balancer itself?

**Answer**:

1. **Active-Passive**: Two load balancers, one active, one standby
2. **Keepalived + VRRP**: Virtual IP fails over to backup
3. **DNS Round Robin**: Multiple load balancer IPs in DNS
4. **Cloud Load Balancers**: AWS ELB, Google Cloud Load Balancer (managed)

### Q2: When to use Layer 4 vs Layer 7 load balancing?

**Answer**:

- **Layer 4** (TCP/UDP): Faster, no content inspection, simpler
- **Layer 7** (HTTP): Slower, content-based routing, more features
- **Use L4 for**: Database connections, generic TCP traffic
- **Use L7 for**: Web applications, API gateways, microservices

### Q3: How do you handle sticky sessions?

**Answer**:

1. **IP Hash**: Route based on client IP
2. **Cookie-based**: Load balancer inserts session cookie
3. **Session Store**: Use Redis for shared sessions (best)

**Trade-off**: Sticky sessions limit load balancing effectiveness.

## 📚 Summary

**Key Takeaways**:

1. **Round Robin**: Simple, fair distribution
2. **Least Connections**: Better for long-lived connections
3. **Weighted**: Handle heterogeneous server pools
4. **IP Hash**: Session persistence (sticky sessions)
5. **Health Checks**: Automatic failover
6. **Active-Passive**: High availability
7. **Layer 4 vs 7**: Choose based on requirements
8. **SSL Termination**: Offload from backends
9. **Connection Pooling**: Reuse backend connections
10. **Monitoring**: Track server health, latency, errors

Load balancers are essential for scalable, reliable systems!
