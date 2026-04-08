# Bug Analysis and Fixes

## Bugs Found

### 1. **`get_remaining` - Missing Lock (Race Condition)**
**Problem:** Accesses `self.requests` without acquiring the lock, causing potential race conditions when multiple threads read/modify data concurrently.

### 2. **`reset` - Missing Lock (Race Condition)**
**Problem:** Modifies `self.requests` without the lock, which can corrupt data if called while other methods are accessing it.

### 3. **`get_wait_time` - Doesn't Filter Old Requests**
**Problem:** Calculates wait time using all timestamps including expired ones, giving incorrect results. Should only consider requests within the current window.

### 4. **`import time` Inside Methods (Code Style)**
**Problem:** Import statements should be at module level, not inside methods. While functional, this is poor practice.

## Corrected Code

```python
import threading
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests = defaultdict(list)
        self.lock = threading.Lock()

    def is_allowed(self, client_id):
        now = time.time()
        with self.lock:
            # Clean old requests
            self.requests[client_id] = [
                t for t in self.requests[client_id]
                if t > now - self.window
            ]
            if len(self.requests[client_id]) < self.max_requests:
                self.requests[client_id].append(now)
                return True
            return False

    def get_remaining(self, client_id):
        now = time.time()
        with self.lock:  # Fix 1: Added lock
            active = [t for t in self.requests[client_id] if t > now - self.window]
            return self.max_requests - len(active)

    def reset(self, client_id=None):
        with self.lock:  # Fix 2: Added lock
            if client_id:
                if client_id in self.requests:  # Avoid KeyError
                    del self.requests[client_id]
            else:
                self.requests.clear()

    def get_wait_time(self, client_id):
        now = time.time()
        with self.lock:
            # Fix 3: Filter old requests before calculating wait time
            active = [t for t in self.requests[client_id] if t > now - self.window]
            if not active:
                return 0
            oldest = min(active)
            wait = (oldest + self.window) - now
            return max(0, wait)
```

## Summary of Fixes

| Method | Bug | Fix |
|--------|-----|-----|
| `get_remaining` | Missing lock | Added `with self.lock:` |
| `reset` | Missing lock + potential KeyError | Added lock + safe deletion check |
| `get_wait_time` | Uses stale timestamps | Filter to active requests only |
| Module level | `import time` inside methods | Moved to top of file |

All fixes ensure thread safety and correct rate-limiting behavior.
