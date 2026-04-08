```python
import heapq
from typing import Iterable, Iterator, TypeVar

T = TypeVar('T')

def merge_sorted_streams(*iterators: Iterable[T]) -> Iterator[T]:
    """
    Merges multiple sorted iterators into a single sorted iterator.
    
    Args:
        *iterators: Arbitrary number of sorted iterables or iterators.
        
    Yields:
        Elements from the input streams in ascending order.
        
    Time Complexity: O(N log K) where N is total elements and K is number of streams.
    """
    heap = []
    
    # Initialize heap with the first element from each non-empty iterator
    for idx, it in enumerate(iterators):
        iterator = iter(it)
        try:
            value = next(iterator)
            # Store (value, unique_index, iterator)
            # unique_index ensures iterators are not compared if values are equal
            heap.append((value, idx, iterator))
        except StopIteration:
            continue
    
    heapq.heapify(heap)
    
    while heap:
        value, idx, iterator = heapq.heappop(heap)
        yield value
        
        try:
            next_value = next(iterator)
            heapq.heappush(heap, (next_value, idx, iterator))
        except StopIteration:
            pass
```
