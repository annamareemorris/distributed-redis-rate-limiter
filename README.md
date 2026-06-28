[README.md](https://github.com/user-attachments/files/29443537/README.md)
# Token Bucket Rate Limiter (Python)

A lightweight token-bucket rate limiter for Python, in two pieces:

- **Free local version** — in-memory, single process. Drop it into a script or a single-instance app.
- **Commercial distributed version** — backed by Redis, **atomic across many servers**. The whole check-and-spend decision runs inside a single Lua script on the Redis server, so concurrent requests can never over-spend the budget.

No background threads and no polling loops: tokens replenish lazily from the exact time elapsed since the bucket was last touched, so a bucket that nobody is hitting costs nothing.

## Which one do I need?

|                      | Free local version        | Commercial (Redis) version            |
| -------------------- | ------------------------- | ------------------------------------- |
| Scope                | one process               | many servers / processes              |
| State                | in memory                 | shared in Redis                       |
| Safe under load      | single instance only      | yes — atomic via a Lua script         |
| Survives a restart   | no                        | yes                                   |
| Good for             | scripts, prototypes, single-instance apps | production APIs behind a load balancer |

If you run more than one copy of your app — multiple servers, or multiple workers/processes behind something like gunicorn — the free version gives **each copy its own separate limit**. For one shared limit across all of them, you need the Redis version.

## Free local quickstart

```python
import time
import threading


class TokenBucketLimiter:
    """In-memory token bucket for a single process. Thread-safe within that process."""

    def __init__(self, capacity: int, refill_rate: float):
        if capacity <= 0:
            raise ValueError("capacity must be > 0")
        if refill_rate < 0:
            raise ValueError("refill_rate must be >= 0")
        self.capacity = capacity          # max tokens the bucket holds
        self.refill_rate = refill_rate    # tokens added per second
        self._tokens = float(capacity)
        self._last = time.monotonic()
        self._lock = threading.Lock()

    def allow_request(self, tokens_requested: int = 1) -> bool:
        with self._lock:
            now = time.monotonic()
            self._tokens = min(
                self.capacity,
                self._tokens + (now - self._last) * self.refill_rate,
            )
            self._last = now
            if self._tokens >= tokens_requested:
                self._tokens -= tokens_requested
                return True
            return False
```

```python
# 100 requests max, refilling 10 per second
limiter = TokenBucketLimiter(capacity=100, refill_rate=10.0)

if limiter.allow_request():
    handle()        # allowed
else:
    reject()        # 429 Too Many Requests
```

## 🚀 Scaling to production? (the distributed version)

The free version above keeps its state in your server's local memory. The moment
you run more than one copy of your app — multiple servers behind a load
balancer, multiple workers, Docker clusters, or serverless like AWS Lambda —
**each copy enforces its own separate limit**, so your real limit is silently
multiplied by the number of instances. Local memory cannot share state.

The commercial version fixes this by moving the state into Redis and running the
entire read → check → spend decision inside one Lua script, which Redis executes
**atomically**. That is what makes it safe when many servers hit the same bucket
at the same instant: there is no window in which two requests both read the same
balance and both succeed. Time is read from Redis itself, so independent app
servers never disagree about the clock.

What you get:

- **One shared limit across every server** — true horizontal scaling, no matter
  how many nodes you add.
- **Atomic, race-free decisions** — proven by the included concurrency test, not
  just claimed.
- **Auto-expiring keys** — idle buckets clean themselves up via Redis TTL.
- **Commercial licence** — use it in your proprietary, closed-source product.

```python
import redis
from limiter import RedisTokenBucketLimiter

r = redis.from_url("redis://localhost:6379/0")

limiter = RedisTokenBucketLimiter(
    redis_client=r,
    bucket_key="user:42",     # one bucket per user / API key / IP address
    capacity=100,             # bucket holds up to 100 tokens
    refill_rate=10.0,         # +10 tokens per second
    fail_open=False,          # deny if Redis is unreachable (True = allow instead)
)

if limiter.allow_request():
    handle()        # allowed
else:
    reject()        # 429 Too Many Requests
```

It ships with a full `pytest` suite, including a concurrency test that fires
hundreds of simultaneous requests at a small bucket and proves it never grants
more than capacity — the guarantee a naive read-modify-write limiter cannot make.

```bash
pip install pytest fakeredis lupa
pytest -v                                   # runs against an in-memory fake Redis, no server needed
REDIS_URL=redis://localhost:6379/0 pytest -v  # or against a real Redis
```

## Getting the commercial version

The Redis version, the test suite, and the commercial licence are delivered as a
package on purchase.

**Buy it here:** https://annamaree.gumroad.com/l/hkjqga

## Licence

- **Free local version (this repo):** _<your choice — e.g. MIT>_
- **Commercial Redis version:** sold under the Enterprise Commercial Software
  Licence (`COMMERCIAL_LICENSE.txt`, included in the package). One commercial
  production application per licence; no reselling or standalone redistribution.
