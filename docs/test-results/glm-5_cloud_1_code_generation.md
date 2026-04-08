```python
from __future__ import annotations

import heapq
from typing import TypeVar, Iterator, Iterable
from itertools import count

T = TypeVar('T')


def merge_sorted_streams(*streams: Iterable[T]) -> Iterator[T]:
    """
    Merge multiple sorted iterables into a single sorted iterator.
    
    Args:
        *streams: Any number of sorted iterables/iterators to merge.
            Each stream must be sorted in ascending order.
        
    Yields:
        Elements from all streams in sorted order.
        
    Time Complexity:
        O(N log K) where N is total elements and K is number of streams.
        
    Note:
        The input streams must contain comparable elements. If streams
        are exhausted during iteration, they are silently skipped.
    """
    # Convert all iterables to iterators
    iterators = [iter(stream) for stream in streams]
    
    # Use a counter for tie-breaking to ensure stable ordering
    # This avoids TypeError when comparing tuples if values are equal
    counter = count()
    
    # Initialize heap with first element from each iterator
    # Heap entries are: (value, tie_breaker, iterator)
    heap: list[tuple[T, int, Iterator[T]]] = []
    
    for iterator in iterators:
        try:
            first_value = next(iterator)
            heapq.heappush(heap, (first_value, next(counter), iterator))
        except StopIteration:
            # Skip empty streams
            pass
    
    # Process heap until all streams are exhausted
    while heap:
        value, _, current_iter = heapq.heappop(heap)
        yield value
        
        try:
            next_value = next(current_iter)
            heapq.heappush(heap, (next_value, next(counter), current_iter))
        except StopIteration:
            # Stream exhausted, continue with remaining streams
            pass
```
