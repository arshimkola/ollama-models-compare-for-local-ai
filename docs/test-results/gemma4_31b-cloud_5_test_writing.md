Since the `LRUCache` is a synchronous data structure, "concurrent access" in a single-threaded JavaScript environment refers to interleaved operations (async/await cycles) or rapid sequential mutations. 

Below is the comprehensive test suite using **Jest**.

```typescript
import { LRUCache } from './LRUCache'; // Adjust path accordingly

describe('LRUCache', () => {
  describe('Initialization', () => {
    it('should initialize with a valid capacity', () => {
      const cache = new LRUCache<string, number>(10);
      expect(cache.size).toBe(0);
    });

    it('should throw an error for zero capacity', () => {
      expect(() => new LRUCache<string, number>(0)).toThrow("Capacity must be positive");
    });

    it('should throw an error for negative capacity', () => {
      expect(() => new LRUCache<string, number>(-1)).toThrow("Capacity must be positive");
    });
  });

  describe('Basic Operations', () => {
    it('should store and retrieve values', () => {
      const cache = new LRUCache<string, string>(2);
      cache.put('a', '1');
      expect(cache.get('a')).toBe('1');
    });

    it('should return undefined for non-existent keys', () => {
      const cache = new LRUCache<string, string>(2);
      expect(cache.get('non-existent')).toBeUndefined();
    });

    it('should correctly report size', () => {
      const cache = new LRUCache<string, string>(3);
      cache.put('a', '1');
      cache.put('b', '2');
      expect(cache.size).toBe(2);
    });

    it('should verify existence using has()', () => {
      const cache = new LRUCache<string, string>(2);
      cache.put('a', '1');
      expect(cache.has('a')).toBe(true);
      expect(cache.has('b')).toBe(false);
    });

    it('should peek at values without updating LRU position', () => {
      const cache = new LRUCache<string, number>(2);
      cache.put('a', 1); // Order: [a]
      cache.put('b', 2); // Order: [a, b]
      
      expect(cache.peek('a')).toBe(1); 
      
      // Since peek didn't update 'a', 'a' is still the oldest.
      // Putting 'c' should evict 'a'.
      cache.put('c', 3);
      expect(cache.has('a')).toBe(false);
      expect(cache.has('b')).toBe(true);
    });

    it('should clear the cache', () => {
      const cache = new LRUCache<string, number>(2);
      cache.put('a', 1);
      cache.clear();
      expect(cache.size).toBe(0);
      expect(cache.get('a')).toBeUndefined();
    });
  });

  describe('LRU Eviction Logic', () => {
    it('should evict the least recently used item when capacity is exceeded', () => {
      const cache = new LRUCache<string, number>(2);
      cache.put('a', 1); // [a]
      cache.put('b', 2); // [a, b]
      cache.put('c', 3); // [b, c] (a evicted)
      
      expect(cache.has('a')).toBe(false);
      expect(cache.has('b')).toBe(true);
      expect(cache.has('c')).toBe(true);
    });

    it('should update priority on get()', () => {
      const cache = new LRUCache<string, number>(2);
      cache.put('a', 1); // [a]
      cache.put('b', 2); // [a, b]
      
      // Access 'a', making 'b' the oldest
      cache.get('a');     // [b, a]
      
      cache.put('c', 3); // [a, c] (b evicted)
      
      expect(cache.has('b')).toBe(false);
      expect(cache.has('a')).toBe(true);
    });

    it('should update priority when putting an existing key', () => {
      const cache = new LRUCache<string, number>(2);
      cache.put('a', 1); // [a]
      cache.put('b', 2); // [a, b]
      
      // Update 'a', making 'b' the oldest
      cache.put('a', 10); // [b, a]
      
      cache.put('c', 3); // [a, c] (b evicted)
      
      expect(cache.has('b')).toBe(false);
      expect(cache.get('a')).toBe(10);
    });
  });

  describe('Edge Cases & Stress Tests', () => {
    it('should handle capacity of 1', () => {
      const cache = new LRUCache<string, number>(1);
      cache.put('a', 1);
      expect(cache.get('a')).toBe(1);
      cache.put('b', 2);
      expect(cache.has('a')).toBe(false);
      expect(cache.get('b')).toBe(2);
    });

    it('should handle very large amounts of data (stability)', () => {
      const size = 1000;
      const cache = new LRUCache<number, number>(size);
      for (let i = 0; i < size * 2; i++) {
        cache.put(i, i);
      }
      expect(cache.size).toBe(size);
      expect(cache.has(0)).toBe(false); // First items should be gone
      expect(cache.has(size * 2 - 1)).toBe(true); // Last item should exist
    });

    it('should handle concurrent-like interleaved async patterns', async () => {
      const cache = new LRUCache<string, number>(2);
      
      // Simulate interleaved asynchronous "requests" updating the cache
      const task = async (key: string, val: number) => {
        await new Promise(res => setTimeout(res, Math.random() * 10));
        cache.put(key, val);
      };

      await Promise.all([
        task('a', 1),
        task('b', 2),
        task('c', 3),
        task('a', 4),
      ]);

      // The cache should still maintain internal consistency (size <= capacity)
      expect(cache.size).toBeLessThanOrEqual(2);
    });

    it('should work with different types (Generics)', () => {
      interface User { name: string }
      const cache = new LRUCache<number, User>(2);
      const user = { name: 'Alice' };
      
      cache.put(1, user);
      expect(cache.get(1)).toBe(user);
      expect(cache.get(1)?.name).toBe('Alice');
    });
  });
});
```

### Key Testing Strategies Used:

1.  **Boundary Analysis**: Tested `capacity` at 1, and tested the constructor with `0` and `-1` to ensure the error handling works.
2.  **LRU Property Verification**: 
    *   **`get` update**: Verified that calling `get()` moves an item to the "most recent" position, preventing its eviction.
    *   **`put` update**: Verified that updating an existing key also refreshes its priority.
    *   **`peek` non-update**: Specifically tested that `peek` does **not** trigger a priority change, which is a common bug in LRU implementations.
3.  **State Consistency**: Checked that the `size` getter always matches the actual number of elements and that the cache never exceeds its `capacity`.
4.  **Interleaved Execution**: While JS is single-threaded, `Promise.all` with random timeouts simulates a real-world scenario where multiple async functions might be interacting with a shared cache instance.
5.  **Generics**: Ensured that the class correctly handles complex types (objects) and different primitive types.
6.  **Complexity**: The stress test with 2,000 operations ensures that the `Map.keys().next()` approach performs reliably under load.
