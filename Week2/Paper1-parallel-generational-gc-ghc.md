## Title : Parallel generational-copying garbage collection with a block-structured heap

## [Link](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.146.1688&rep=rep1&type=pdf)

Summary: This paper talks about a parallel version of the Cheney's algorithm. An additional novelty of the algorithm is exploiting the block strucutred nature of the heap, where the heap is essentially divided into blocks of fixed size. The algorithm, unlike concurrent mark and sweep still is stopping collector, but ensures the activities run in parallel.

### Generational and copying collector

Whenever generation n is collected so are all the other generations. A remembered set for generation n keeps track of all pointers of generation n into younger ones. Copying proceeds in two steps:

1. Evacuate each root pointer and remembered set pointer. Evacuation means " copy the object it points to into to-space, overwrite the original object (in from-space) with a forwarding
pointer to the new copy of the object, and return the forwarding pointer.  If the object has already been evacuated, and hence has been overwritten with a forwarding pointer, just return that pointer." Evacuation is basically the first step of a standard Cheney's algorithm. 

2. Scavenge each object in the to-space. "evacuate each pointer in the object, replacing the pointer with the address of the evacuated object. When all objects in to-space have been scavenged, garbage collection is complete."


Promotion to higher generations is based on some tenuring policy which usually promotes an object after it has survived `kn` GCs.

### Block strucutred heap

Heap is divided into fixed size B byte blocks. Each block has an associated block descriptor which describes the generation and step of the block. Blocks are linked to form an area. Heap contains heap objects (first word of every heap object -> its info pointer, points
to its statically-allocated info table, which in turn contains layout information that guides the garbage collector). Heap pointer addresses the first word of the heap object. Large objects are allocated in a block group.


### Parallel GC

Each thread has a single pending set, and runs the following loop

```C
while (pending set non-empty) {
  remove an object p from the pending set
  scavenge(p)
  add any newly-evacuated objects to the pending set
}
```
The to-set is much more trickier in GHC than how it was described by Cheney and this is where the block structured heap comes into play. The heap is already divided into blocks and each thread is allocated a particular multiple of a block area (the granularity of parallelism is decided based on empirical evidences)

"We call the set of blocks remaining to be scavenged the PendingBlock Set. In our implementation, it is represented by a single list of blocks linked together via their block descriptors, protected by a mutex. One could use per-thread work-stealing queues to hold the
set of pending blocks, but so far (using only a handful of processors) we have found that there is negligible contention whenusing a conventional lock to protect a single data structure;"


### Parallel generational GC

1. Each GC thread maintains one Allocation Block for each step of each generation (there is still just a single Scan block).
2. When evacuating an object, the GC must decide into which generation and step to copy the object
3. The GC must implement a write-barrier to track old-to-new pointers, and a remembered set for each generation. Currently a single, shared remembered set for each generation protected by a mutex is used. It would be quite possible instead to have a thread-local remembered set for each generation if contention for these mutexes became a problem.
4. When looking for work, a GC thread always seeks to scan the oldest-possible block.

Eager promotion: Suppose that an object W in generation 0 is pointed to by an immutable object in generation 2, then it is eagerly promoted to generation 2 rather than waiting for a few more cycles