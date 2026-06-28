# High-Performance Token Bucket Rate Limiter (Python)

An ultra-lightweight, atomic rate limiting utility designed for high-concurrency environments. Zero background threads, zero overhead, mathematically calculated on-the-fly.

## Why this architecture?
Standard rate limiters use background sleep loops or heavy intervals to replenish tokens, which wastes CPU cycles. This engine calculates token replenishment dynamically based on exact time elapsed since the client's last request—resulting in sub-millisecond execution times.

## Local Memory Quickstart (Free Component)

Ideal for single-instance applications, local scripts, or small-scale testing.

```python
import time

class TokenBucketLimiter:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_update = time.time()

    def allow_request(self, tokens_requested: int = 1) -> bool:
        now = time.time()
        elapsed = now - self.last_update
        self.last_update = now
        self.tokens = min(self.capacity, self.tokens + (elapsed * self.refill_rate))
        
        if self.tokens >= tokens_requested:
            self.tokens -= tokens_requested
            return True
        return False


## 🚀 Scaling to Production? (Distributed Multi-Server State)

The open-source code above stores data in your server's local RAM. If your application scales across multiple servers, a load balancer, or serverless setups (like AWS Lambda or Docker clusters), **local memory will fail** because individual instances cannot share state.

The **Enterprise Version** solves this instantly by moving the atomic state directly into **Redis**, letting your entire server architecture track global infrastructure traffic locks collectively in real-time.

### Upgrading to Enterprise ($149 USD):
* **Atomic Redis State Synchronization:** Complete horizontal scalability across infinite application nodes.
* **Auto-Expiring Volatile Keys:** Built-in Redis TTL (Time-To-Live) memory optimization prevents database clutter.
* **Commercial Perpetual License:** Fully legally approved for proprietary, closed-source enterprise software ecosystems.

👉 **[Download Enterprise Engine & Commercial License Key Here](https://annamaree.gumroad.com/l/hkjqga)** 

# Usage: Limit to 5 requests maximum, refilling at 1 token per second
limiter = TokenBucketLimiter(capacity=5, refill_rate=1.0)
