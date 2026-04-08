```python
import heapq
from typing import Iterable, Iterator, TypeVar, Any

T = TypeVar('T')

def merge_sorted_streams(*streams: Iterable[T]) -> Iterator[T]:
    """
    Merges multiple sorted iterators into a single sorted iterator.
    
    Time Complexity: O(N log K) where N is total elements and K is number of streams.
    Space Complexity: O(K) to store the min-heap.
    """
    # Create iterators from all provided iterables
    iterators = [iter(s) for s in streams]
    min_heap = []

    # Initialize the heap with the first element from each iterator
    for i, it in enumerate(iterators):
        try:
            # Push tuple: (value, stream_index)
            # We include the index to handle non-comparable elements in the stream
            # (e.g., if two values are equal, it won't try to compare the iterators)
            first_val = next(it)
            heapq.heappush(min_heap, (first_val, i))
        except StopIteration:
            continue

    while min_heap:
        # Extract the smallest current element
        val, idx = heapq.heappop(min_heap)
        yield val

        # Get the next element from the iterator that produced the popped value
        try:
            next_val = next(iterators[idx])
            heapq.heappush(min_heap, (next_val, idx))
        except StopIteration:
            # This stream is exhausted, continue with others
            continue
```
