Here is a comprehensive Jest test suite for the `LRUCache` class.

### Prerequisites

Ensure you have Jest and TypeScript support installed:
```bash
npm install --save-dev jest ts-jest @types/jest typescript
```

### Test File (`LRUCache.test.ts`)

```typescript
import { LRUCache } from './LRUCache'; // Adjust path based on your project structure

describe('LRUCache', () => {
  describe('Initialization & Validation', () => {
    test('should create a cache with valid capacity', () => {
      const cache = new LRUCache<string, number>(5);
      expect(cache.size).toBe(0);
    });

    test('should throw error if capacity is zero', () => {
      expect(() => new LRUCache(0)).toThrow('Capacity must be positive');
    });

    test('should throw error if capacity is negative', () => {
      expect(() => new LRUCache(-10)).toThrow('Capacity must be positive');
    });
  });

  describe('Basic Operations (Put, Get, Has)', () => {
    test('should store and retrieve a value', () => {
      const cache = new LRUCache<number, string>(2);
      cache.put(1, 'one');
      expect(cache.get(1)).toBe('one');
    });

    test('should return undefined for non-existent key', () => {
      const cache = new LRUCache<number, string>(2);
      expect(cache.get(1)).toBeUndefined();
    });

    test('should report existence correctly via has()', () => {
      const cache = new LRUCache<number, string>(2);
      expect(cache.has(1)).toBeFalsy();
      cache.put(1, 'one');
      expect(cache.has(1)).toBeTruthy();
    });

    test('should distinguish between undefined value and missing key', () => {
      const cache = new LRUCache<number, undefined | number>(2);
      cache.put(1, undefined);
      
      // get returns undefined for both missing keys and stored undefined values
      expect(cache.get(1)).toBeUndefined(); 
      
      // has distinguishes them
      expect(cache.has(1)).toBeTruthy();
      expect(cache.has(2)).toBeFalsy();
    });
  });

  describe('LRU Eviction Logic', () => {
    test('should evict the least recently used item when capacity is exceeded', () => {
      const cache = new LRUCache<number, string>(2);
      cache.put(1, 'one');
      cache.put(2, 'two');
      
      // Capacity reached, adding 3 should evict 1 (LRU)
      cache.put(3, 'three');

      expect(cache.get(1)).toBeUndefined(); // Evicted
      expect(cache.get(2)).toBe('two');
      expect(cache.get(3)).toBe('three');
    });

    test('should evict correct item after access reordering', () => {
      const cache = new LRUCache<number, string>(2);
      cache.put(1, 'one');
      cache.put(2, 'two');
      
      // Access 1, making it Most Recently Used (MRU)
      cache.get(1); 
      
      // Add 3. 2 is now the LRU, not 1.
      cache.put(3, 'three');

      expect(cache.get(1)).toBe('one'); // Still exists
      expect(cache.get(2)).toBeUndefined(); // Evicted
      expect(cache.get(3)).toBe('three');
    });
  });

  describe('Update Behavior', () => {
    test('should update value and refresh access order on put existing key', () => {
      const cache = new LRUCache<number, string>(2);
      cache.put(1, 'one');
      cache.put(2, 'two');
      
      // Update key 1
      cache.put(1, 'one-updated');

      // Size should not increase
      expect(cache.size).toBe(2);
      expect(cache.get(1)).toBe('one-updated');

      // Add new key. Key 2 should be evicted because 1 was refreshed.
      cache.put(3, 'three');
      expect(cache.get(2)).toBeUndefined();
      expect(cache.get(1)).toBe('one-updated');
    });
  });

  describe('Peek vs Get', () => {
    test('peek should return value without updating access order', () => {
      const cache = new LRUCache<number, string>(2);
      cache.put(1, 'one');
      cache.put(2, 'two');

      // Peek at 1. It should remain LRU.
      expect(cache.peek(1)).toBe('one');

      // Add 3. 1 should be evicted because peek did not refresh it.
      cache.put(3, 'three');

      expect(cache.get(1)).toBeUndefined(); // Evicted
      expect(cache.get(2)).toBe('two');
    });

    test('peek should return undefined for missing keys', () => {
      const cache = new LRUCache<number, string>(1);
      expect(cache.peek(999)).toBeUndefined();
    });
  });

  describe('Utility Methods', () => {
    test('clear should empty the cache', () => {
      const cache = new LRUCache<number, string>(3);
      cache.put(1, 'one');
      cache.put(2, 'two');
      expect(cache.size).toBe(2);

      cache.clear();
      expect(cache.size).toBe(0);
      expect(cache.get(1)).toBeUndefined();
      expect(cache.get(2)).toBeUndefined();
    });

    test('size should track count accurately', () => {
      const cache = new LRUCache<number, string>(3);
      expect(cache.size).toBe(0);
      
      cache.put(1, 'a');
      expect(cache.size).toBe(1);
      
      cache.put(1, 'b'); // Update
      expect(cache.size).toBe(1);
      
      cache.put(2, 'c');
      expect(cache.size).toBe(2);
      
      cache.clear();
      expect(cache.size).toBe(0);
    });
  });

  describe('Edge Cases', () => {
    test('should handle capacity of 1', () => {
      const cache = new LRUCache<number, string>(1);
      cache.put(1, 'one');
      expect(cache.get(1)).toBe('one');
      
      cache.put(2, 'two');
      expect(cache.get(1)).toBeUndefined();
      expect(cache.get(2)).toBe('two');
    });

    test('should handle complex object keys and values', () => {
      type KeyObj = { id: number };
      type ValObj = { data: string };
      
      // Note: Map uses reference equality for objects. 
      // In a real scenario, you might serialize keys, but we test the class as-is.
      const cache = new LRUCache<KeyObj, ValObj>(2);
      
      const k1 = { id: 1 };
      const k2 = { id: 2 };
      const v1 = { data: 'A' };
      const v2 = { data: 'B' };

      cache.put(k1, v1);
      cache.put(k2, v2);

      expect(cache.get(k1)?.data).toBe('A');
      expect(cache.get(k2)?.data).toBe('B');
    });
  });

  describe('Concurrent Access Patterns (Async Simulation)', () => {
    // Note: JS is single-threaded. True race conditions don't exist on sync methods.
    // However, we test that the cache remains consistent when used within async flows.
    
    test('should maintain consistency under rapid async operations', async () => {
      const cache = new LRUCache<number, number>(100);
      const operations: Promise<void>[] = [];

      // Simulate 50 concurrent put operations
      for (let i = 0; i < 50; i++) {
        operations.push(Promise.resolve().then(() => {
          cache.put(i, i * 10);
        }));
      }

      await Promise.all(operations);

      expect(cache.size).toBe(50);
      for (let i = 0; i < 50; i++) {
        expect(cache.get(i)).toBe(i * 10);
      }
    });

    test('should handle interleaved read/write async flows', async () => {
      const cache = new LRUCache<string, number>(5);
      
      const writer = async () => {
        for (let i = 0; i < 10; i++) {
          await Promise.resolve(); // Yield to event loop
          cache.put(`key-${i}`, i);
        }
      };

      const reader = async () => {
        for (let i = 0; i < 10; i++) {
          await Promise.resolve(); // Yield to event loop
          cache.get(`key-${i}`);
        }
      };

      await Promise.all([writer(), reader()]);

      // After 10 writes with capacity 5, keys 0-4 should be evicted.
      // Keys 5-9 should exist.
      expect(cache.has('key-0')).toBeFalsy();
      expect(cache.has('key-4')).toBeFalsy();
      expect(cache.has('key-5')).toBeTruthy();
      expect(cache.has('key-9')).toBeTruthy();
      expect(cache.size).toBe(5);
    });
  });
});
```

### Key Testing Strategies Explained

1.  **Eviction Verification**: The core logic of an LRU cache is eviction. Tests verify that when capacity is exceeded, the *oldest accessed* item is removed, not necessarily the *oldest inserted* item (if access order changed).
2.  **Access Order Updates**: We specifically test that both `get` and `put` (on existing keys) move the item to the "Most Recently Used" position.
3.  **`peek` vs `get`**: A common bug in LRU implementations is accidentally updating access order on read-only operations. The `peek` tests ensure the internal order is untouched.
4.  **`undefined` Values**: Since `get` returns `undefined` for missing keys, storing `undefined` as a valid value is an edge case. We verify `has()` is used to distinguish the two states.
5.  **Concurrency Simulation**: JavaScript is single-threaded, so synchronous code cannot have race conditions. However, in real applications, caches are often called within `async` functions. The "Concurrent Access" tests simulate high-frequency event-loop interleaving to ensure the internal `Map` state doesn't corrupt when wrapped in Promises.
6.  **Capacity = 1**: This is a specific boundary condition where every new `put` (of a new key) forces an eviction.

### Running the Tests

Add this to your `package.json` scripts:
```json
"scripts": {
  "test": "jest"
}
```

Then run:
```bash
npm test
```
