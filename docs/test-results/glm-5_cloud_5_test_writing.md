Here is a comprehensive test suite for the `LRUCache` class using Jest.

### Test Strategy

1.  **Construction & Configuration:** Verify valid and invalid initialization.
2.  **Core Functionality (`get`, `put`):** Ensure basic CRUD operations work and that `get` updates the recency.
3.  **Eviction Logic:** Verify that the cache evicts the correct item (Least Recently Used) when capacity is reached.
4.  **Edge Cases:** Test capacity of 1, handling `undefined` values, and different key types.
5.  **Concurrency Patterns:** Since the provided class is synchronous and JavaScript is single-threaded, "concurrency" here refers to rapid interleaving of operations (simulating high-load scenarios) and ensuring state consistency.

```typescript
import { LRUCache } from './LRUCache'; // Adjust import path as necessary

describe('LRUCache', () => {
  
  // ==========================================
  // 1. Construction and Configuration
  // ==========================================
  describe('Constructor', () => {
    it('should initialize successfully with a positive capacity', () => {
      expect(() => new LRUCache(10)).not.toThrow();
      const cache = new LRUCache(5);
      expect(cache.size).toBe(0);
    });

    it('should throw an error for zero capacity', () => {
      expect(() => new LRUCache(0)).toThrow('Capacity must be positive');
    });

    it('should throw an error for negative capacity', () => {
      expect(() => new LRUCache(-5)).toThrow('Capacity must be positive');
    });
  });

  // ==========================================
  // 2. Core Functionality
  // ==========================================
  describe('put and get basics', () => {
    let cache: LRUCache<string, number>;

    beforeEach(() => {
      cache = new LRUCache<string, number>(3);
    });

    it('should return undefined for non-existent keys', () => {
      expect(cache.get('missing')).toBeUndefined();
    });

    it('should set and get a value correctly', () => {
      cache.put('a', 1);
      expect(cache.get('a')).toBe(1);
      expect(cache.size).toBe(1);
    });

    it('should update an existing key\'s value without increasing size', () => {
      cache.put('a', 1);
      cache.put('a', 99);
      
      expect(cache.size).toBe(1);
      expect(cache.get('a')).toBe(99);
    });

    it('should update recency when updating an existing key', () => {
      // Fill cache: a (LRU), b, c (MRU)
      cache.put('a', 1);
      cache.put('b', 2);
      cache.put('c', 3);

      // Update 'a'. It should become MRU.
      cache.put('a', 100);

      // Add new item 'd'. Should evict 'b' (now LRU).
      cache.put('d', 4);

      expect(cache.has('a')).toBe(true);
      expect(cache.has('b')).toBe(false); // Evicted
      expect(cache.has('c')).toBe(true);
      expect(cache.has('d')).toBe(true);
    });

    it('should update recency when getting a key', () => {
      cache.put('a', 1);
      cache.put('b', 2);
      cache.put('c', 3);

      // Access 'a'. Order now: b (LRU), c, a (MRU)
      cache.get('a');

      // Add 'd'. Should evict 'b'.
      cache.put('d', 4);

      expect(cache.has('a')).toBe(true);
      expect(cache.has('b')).toBe(false);
    });
  });

  // ==========================================
  // 3. Eviction Logic
  // ==========================================
  describe('Eviction', () => {
    let cache: LRUCache<string, number>;

    beforeEach(() => {
      cache = new LRUCache<string, number>(3);
    });

    it('should evict the least recently used item when capacity is exceeded', () => {
      cache.put('a', 1);
      cache.put('b', 2);
      cache.put('c', 3);
      
      // Cache is full: a (LRU), b, c (MRU)
      cache.put('d', 4); // Should evict 'a'

      expect(cache.size).toBe(3);
      expect(cache.has('a')).toBe(false);
      expect(cache.get('b')).toBe(2);
      expect(cache.get('c')).toBe(3);
      expect(cache.get('d')).toBe(4);
    });

    it('should handle a cascade of gets and evict correctly', () => {
      cache.put('a', 1);
      cache.put('b', 2);
      cache.put('c', 3);

      // Order: a, b, c
      cache.get('a'); // Order: b, c, a
      cache.put('d', 4); // Evicts b. Order: c, a, d
      
      expect(cache.has('b')).toBe(false);
      expect(cache.has('a')).toBe(true);
      
      cache.put('e', 5); // Evicts c. Order: a, d, e
      expect(cache.has('c')).toBe(false);
    });

    it('should evict correctly with capacity of 1', () => {
      const singleCache = new LRUCache<string, number>(1);
      singleCache.put('a', 1);
      expect(singleCache.size).toBe(1);
      
      singleCache.put('b', 2); // Evicts a
      expect(singleCache.has('a')).toBe(false);
      expect(singleCache.get('b')).toBe(2);

      singleCache.get('b'); // Access 'b'
      singleCache.put('c', 3); // Evicts b
      expect(singleCache.has('b')).toBe(false);
    });
  });

  // ==========================================
  // 4. Helper Methods (has, peek, clear, size)
  // ==========================================
  describe('Helper Methods', () => {
    let cache: LRUCache<string, number>;

    beforeEach(() => {
      cache = new LRUCache<string, number>(3);
      cache.put('a', 1);
      cache.put('b', 2);
    });

    it('has() should return boolean without affecting recency', () => {
      cache.put('c', 3); // Full: a, b, c
      
      // Check 'a' exists but do not promote
      expect(cache.has('a')).toBe(true);
      
      // Add 'd'. Should evict 'a' because 'has' should not update order
      cache.put('d', 4);
      
      expect(cache.has('a')).toBe(false); 
    });

    it('peek() should return value without affecting recency', () => {
      cache.put('c', 3); // Full: a, b, c
      
      expect(cache.peek('a')).toBe(1);
      
      // Add 'd'. Should evict 'a' because 'peek' should not update order
      cache.put('d', 4);
      
      expect(cache.has('a')).toBe(false);
      expect(cache.peek('d')).toBe(4);
      expect(cache.peek('missing')).toBeUndefined();
    });

    it('clear() should empty the cache', () => {
      expect(cache.size).toBe(2);
      cache.clear();
      expect(cache.size).toBe(0);
      expect(cache.has('a')).toBe(false);
    });

    it('size property should reflect current number of items', () => {
      expect(cache.size).toBe(2);
      cache.put('c', 3);
      expect(cache.size).toBe(3);
      cache.put('d', 4); // Evicts one, size stays 3
      expect(cache.size).toBe(3);
    });
  });

  // ==========================================
  // 5. Edge Cases
  // ==========================================
  describe('Edge Cases', () => {
    it('should allow undefined as a value', () => {
      const cache = new LRUCache<string, undefined | number>(2);
      cache.put('a', undefined);
      
      expect(cache.has('a')).toBe(true);
      expect(cache.get('a')).toBeUndefined();
      // Ensure we distinguish between "missing" and "stored undefined"
      // In Map, has() returns true for undefined values.
    });

    it('should support various key types (number, object)', () => {
      const cache = new LRUCache<string | number | object, string>(2);
      
      cache.put(1, 'numberKey');
      expect(cache.get(1)).toBe('numberKey');

      const objKey = { id: 123 };
      cache.put(objKey, 'objectKey');
      expect(cache.get(objKey)).toBe('objectKey');
    });
  });

  // ==========================================
  // 6. Concurrent Access Patterns
  // ==========================================
  describe('Concurrency & Interleaving Patterns', () => {
    /**
     * Note: JavaScript is single-threaded. These tests verify logical consistency
     * during complex rapid sequences, simulating "concurrent-like" behavior
     * where operations interleave in unexpected ways.
     */
    
    it('should maintain consistency during rapid alternating read/writes', () => {
      const cache = new LRUCache<number, number>(3);
      
      // Sequence: Put 1, Put 2, Put 3, Get 1 (promote 1), Put 4 (evict 2)
      cache.put(1, 10);
      cache.put(2, 20);
      cache.put(3, 30);
      
      const val = cache.get(1); 
      expect(val).toBe(10); 
      
      // State should now be: 2 (LRU), 3, 1 (MRU)
      cache.put(4, 40);
      
      // 2 should be evicted
      expect(cache.has(1)).toBe(true);
      expect(cache.has(2)).toBe(false);
      expect(cache.has(3)).toBe(true);
      expect(cache.has(4)).toBe(true);
    });

    it('should handle repeated updates to the same key correctly (no duplicate entries)', () => {
      const cache = new LRUCache<string, number>(2);
      
      cache.put('a', 1);
      cache.put('a', 1); // Duplicate update
      cache.put('a', 1); // Duplicate update
      
      expect(cache.size).toBe(1);
      
      // Now fill cache
      cache.put('b', 2);
      cache.put('c', 3); // Should evict 'a'
      
      expect(cache.size).toBe(2);
      expect(cache.has('a')).toBe(false);
    });

    it('should handle interleaved has/peek/gets correctly under capacity pressure', () => {
      const cache = new LRUCache<string, number>(2);
      
      // 1. Add A, B
      cache.put('A', 1);
      cache.put('B', 2); // State: A, B
      
      // 2. Peek A (no change)
      cache.peek('A');
      
      // 3. Add C (Evict A)
      cache.put('C', 3); // State: B, C
      expect(cache.has('A')).toBe(false);
      
      // 4. Get B (promote B)
      cache.get('B');    // State: C, B
      
      // 5. Add D (Evict C)
      cache.put('D', 4); // State: B, D
      
      expect(cache.has('C')).toBe(false);
      expect(cache.get('B')).toBe(2);
      expect(cache.get('D')).toBe(4);
    });
  });
});
```
