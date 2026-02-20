Ethan Chang
CSCI 251 - Project 1: Race Condition Detective

## Bug 1: BuggyBank

### Race Condition Location
Lines 34-36 in `Deposit()` and lines 51-53 in `Withdraw()` methods have read-modify-write on `_balance`.

### Shared State Involved
`_balance` is being accessed by multiple threads at the same time.

### Why It's a Bug
The read-modify-write sequence (`current = _balance` then `_balance = current + amount`) is not atomic. This means two threads can both read the same value from `_balance`, then compute the updated value independently, then both write back causing one updated value to be lost and the total balance to be inaccurate.

### Your Fix
Wrapping `Deposit()` and `Withdraw()` functions with `lock(_lock)`.

### Why Your Fix Works
This fix works since the lock during the read-modify-write sequence can allow only one thread to modify `_balance` at a time, and ensures the total balance is preserved.

## Bug 2: BuggyCounter

### Race Condition Location
Line 23 in `Increment()` where it contained the `_count++` operation.

### Shared State Involved
`_count` can be accessed by multiple threads simultaneously and is not atomic.

### Why It's a Bug
`_count++` looks like a single operation but it first reads `_count`, then adds 1 as modification, then writes back the value. This means two threads can both read the same value, both increment by 1, and write back at the same time.

### Your Fix
Replaced `_count++` with `Interlocked.Increment(ref _count)`.

### Why Your Fix Works
`Interlocked.Increment` can perform the read-modify-write as a single atomic instruction. This means no two threads can run simultaneously during the operation.

## Bug 3: BuggyCache

### Race Condition Location
Lines 33-40 in `GetOrCompute()`, contains check-then-act on `_cache.ContainsKey(key)` outside of the lock.

### Shared State Involved
`_cache` is being accessed by multiple threads at the same time.

### Why It's a Bug
Multiple threads can pass the `ContainsKey` statement simultaneously since it's outside of the lock. They can view the key as missing and proceed to compute the value independently, causing the expensive computation to run multiple times for the same key.

### Your Fix
Wrapping the expensive computation with `lock(_lock)` and using a second `ContainsKey` check inside as a double-checked locking pattern.

### Why Your Fix Works
The `ContainsKey` check inside the lock ensures only the first thread can compute and store the value. All subsequent threads will find the key already present and skip the computation.

## Bug 4: BuggyLogger

### Race Condition Location
Line 32 in `Log()` and lines 46-55 in `Flush()` have unsynchronized access to `_buffer`.

### Shared State Involved
`_buffer` is accessed by multiple threads simultaneously.

### Why It's a Bug
First issue is that multiple threads call `Log()` and would append to StringBuilder `_buffer` at the same time. Second, `Flush()` reads and clears the buffer allowing another thread to append a message between `ToString()` and `Clear()`, causing the message to be lost.

### Your Fix
Wrapping `Log()` and `Flush()` functions with a `lock(_lock)` statement.

### Why Your Fix Works
Using the same lock for both methods ensures that no threads can append to the buffer while another thread is flushing it.

## Bug 5: BuggyQueue

### Race Condition Location
Lines 37-41 in `Enqueue()`, lines 53-65 in `Dequeue()`, and lines 79-82 in `Complete()` have unsynchronized access to `_queue` and `_count`.

### Shared State Involved
`_queue` and `_count` are accessed by multiple threads simultaneously.

### Why It's a Bug
First problem is that `_queue.Enqueue()` and `_count++` were outside of the lock, allowing the queue and count to be out of sync when multiple threads enqueue at the same time. The second issue is that `_queue.Dequeue()` and `_count--` were outside of the lock, allowing a thread to take an item between `Monitor.Wait` and the actual dequeue, possibly causing a consumer to dequeue from an empty queue.

### Your Fix
Wrapping `_queue.Enqueue()`, `_count++`, and `Monitor.Pulse` all together in `lock(_lock)` in `Enqueue()`. Wrapping `Monitor.Wait`, `_queue.Dequeue()`, and `_count--` together in `lock(_lock)` in `Dequeue()`.

### Why Your Fix Works
Keeping `_queue` and `_count` updates inside the same lock ensures they are always consistent.