Algorithm for LRU Replacement Policy in Buffer:-

1. During the initialization of server all the buffers are placed initially in freelist.
2. When the process requests the page it first looks in the buffer hash table.If the buffer is found holding that page then that buffer is pinned and the buffer id is returned to the process.
3. If the page requested is not found in buffer, then the page is loaded from storage to buffer.
4. To load page into buffer, it first finds the buffer to which the page needs to be loaded(StrategyGetBuffer handles this in Postgresql Database).
5. To find the suitable buffer, we first check in the freelist if any buffer is available and return it.
6. If no buffer is available in the freelist, then it means all the buffers are occupied with the page. Then to replace the buffer, Least-Recently-Used Algorithm is used to replace the buffer.
7. In Least-Recently-Used Algorithm we evict the buffer which is least recently used and the algorithm works in below manner.
8. To implement the LRU algorithm, I have created the LRUQueue which holds the buffers which have refcount 0 and have valid page. LRUQueue is implemented as the doubly linked list and the head and tail of list is stored in the BufferStrategyControl structure.
9. When the ref-count of buffer reaches to 0, then it was added to the tail of LRUQueue and to fetch the buffer for the replacement, the buffer at the head of Queue is selected for the replacement and it will be removed from the LRU Queue. This always removes the buffer which is least recently used.
10. When buffer holds the valid page and have refcount of 0, it will be in LRUQueue and when the process requests that page, then it needs to be removed from LRUQueue.
11. If there is no buffer in the freelist and LRUQueue, then it means no unpinned buffers are available.
12. End.


Data-Structures Manipulated:-

In BufferStrategyControl added two new fields lruQueueHead, lruQueueTail which holds the buffer id of head and tail of the LRUQueue.
int lruQueueHead;
int lruQueueTail;


#define LRUNEXT_END_IN_LRU    (-3)
#define LRUNEXT_NOT_IN_LRU    (-4)

Added these constants


In BufferDesc Structure, added the two new fields nextInLRU, prevInLRU of buffer which holds the buffer id of previous and next buffer in the LRUQueue.

Implemented three functions
1) AddToTail(bufId) - Add the buffer to the end of LRUQueue. This function takes the bufferId of the buffer as the Input.
2) RemoveFromLRU_Locked(buf) - Removes the bufferDescriptor from the LRUQueue(Linked List). This function takes the buffer as the input and removes it from the LRUQueue. It expects the hold the lock of buffer header and the releases the lock upon returning.
3) RemoveFromLRU(bufId) - Removes the buffer from the LRUQueue(Linked List). This function takes the id of the buffer to be removed form the Linked List and it doesn't require to hold the buffer lock header when the function is called.


Implementation:-
When initializing the BufferStrategyControl the lruQueueHead, lruQueueTail are initially set to LRUNEXT_END_IN_LRU.
In UnpinBuffer when the ref-count of buffer reaches to 0 then AddToTail function is called to add that buffer in end of the queue.
In strategyGetBuffer when there is no buffer in freelist, the buffer from the head of LRUQueue which is  BufferStrategyControl.lruQueueHead is returned. If it's value is LRUNEXT_END_IN_LRU then it means no unpinnned buffers are available.
If the page in buffer is pinned which has refcount 0, then it is removed from LRU Queue.

