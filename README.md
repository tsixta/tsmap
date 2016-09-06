# Thread-safe wrapper for std::map
Thread-safe C++11 wrapper for std::map with [readers-writer lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock).
To use it, copy `tsmap.h` and `ReadersWriterLock.h` into your project and include `tsmap.h`.
The wrapper is implemented as a single class `tsmap`, which is derived from `std::map` and 
overrides most of its methods (see <http://www.cplusplus.com/reference/map/map/>
for documentation). Non-overriden methods are `begin()`, `end()`, `rbegin()`, `rend()`, 
`cbegin()`, `cend()`, `crbegin()`, `crend()`, `max_size()`, `key_comp()`, `value_comp()` and 
`get_allocator()`. There are also some additional methods that manage the locking mechanism and allow 
to perform thread-safe operations with the map that are not implemented internally:

* `bool set_lock_style(ReadersWriterLock::LockStyle lockStyle)`  
Set internal implementation of the R-W lock. Possible values are
    * `ReadersWriterLock::LockStyle::NONE` No synchronization.
    * `ReadersWriterLock::LockStyle::MSL` One mutex and busy waiting of at most one thread. This is also the default locking mechanism.
    * `ReadersWriterLock::LockStyle::M2CV` Two mutexes, one condition variable.  
See below for more detailed explanation of all implementations. This function is not thread safe and changing 
the locking mechanism during a read or write operation may lead to a dead-lock or race conditions.  
Returns `true` on success and `false` otherwise.
* `void lock_read()`  
Manually lock the map for reading (allow multiple threads to read and block writers). To prevent dead-locks it must be eventually followed by `unlock_read()`.
* `void unlock_read()`  
Manually unlock the map after a reading operation.
* `void lock_write()`  
Manually lock the map for writing (only a single thread has access to the map). To prevent dead-locks it must be eventually followed by `unlock_write()`.
* `void unlock_write()`  
Manually unlock the map after a writing operation.

Locking and unlocking methods are useful for operations, that cannot be made thread-safe internally.
For example if one thread iterates over the content of the tsmap, the map has no way to recognize, 
that the loop has ended and it is time to release the read lock. However without the read lock 
another thread may change the content of the map in the middle of the loop, which may lead to
undefined behavior. A simple solution is to lock the map explicitly:
```C++
tsmap<int, int> map;
for(auto i=0;i<10;i++) map.insert(std::make_pair(i,i));
map.lock_read();
for(auto p: map) std::cout << p.first << ", " << p.second << std::endl;
map.unlock_read();
```

# Recommendations
Try to avoid `operator[]` unless you intend to change the content of the map. It has to lock the map for writing, which excludes 
other threads from accessing the map.

# Implementations of R-W lock
The R-W lock is implemented by class `ReadersWriterLock` in `ReadersWriterLock.h`. All variants
depend purely on features of C++11 and do not require any external libraries.

### One mutex, busy waiting of at most one thread (`ReadersWriterLock::LockStyle::MSL`)
This implementation uses two `std::atomic<int>` variables to keep track of number of readers
and writers. If some thread asks for the write lock, it locks the mutex, increases number of writers and 
then waits in a while loop, until all readers finish. If any other writer asks for the write lock in the same 
time, it is blocked by the mutex and therefore does not waste CPU power by busy waiting. When a writer finishes,
unlocks the mutex and lets the OS to select another (waiting) thread to continue. 
Threads asking for the read lock first increase the number of readers and wait on the mutex only 
if the number of writers is greater than zero. In such a case, before blocking themselves on the mutex
they decrease the number of readers to prevent dead-lock and increment it by 1 only after they are woken up.
This implementation gives the same priority to writers and readers.

**Pros:**  
* Busy waiting is never used by more than one thread.
* With no writers access to the map might be lock free (depending on internal implementation of `std::atomic<int>`).

**Cons:**  
* When a writer releases the write lock, the waiting threads are woken up one by one, which is inefficient.
* With no control which thread is woken up when a writer finishes it is possible (although unlikely), that some threads are starved.



### Two mutexes, one condition variable (`ReadersWriterLock::LockStyle::M2CV`)
This implementation uses two `std::atomic<int>` variables to keep track of number of readers
and writers. If some thread asks for the write lock, it first increases the number of writers,
locks the reading mutex and waits on an associated condition variable, until all readers finish. When this
happens, it locks the writing mutex, which ensures that no other thread writes into the map in the 
same time. When the writer finishes, it decreases the number of writers, notifies all the threads waiting
on the condition variable and unlocks the writing mutex. A reading thread first increases the number
of readers and then checks the current number of writers. If it is greater than zero, it decreases the number 
of readers (in order to prevent dead-lock), locks the reading mutex, notify all the threads waiting on the 
conditional variable and than waits on it until all writers finish. If there are no writers at all (waiting or active), 
signaling the condition variable will wake up the waiting readers, which consequently increase the number
of readers in order to block any new writers. Otherwise it will wake up only writers and 
keep readers waiting. This guarantees that no writer has to wait indefinitely. When releasing the read lock, 
the reader decreases the number of readers and notifies all the threads waiting on the conditional variable.
This implementation gives more priority to writers.

**Pros:**  
* No busy waiting
* With no writers access to the map might be lock free (depending on internal implementation of `std::atomic<int>`).

**Cons:**  
* For some workloads the overhead of an additional mutex and a conditional variable might be actually worse, than let one thread to spin in a loop.
* Too frequent write requests might easily block reading threads forever.

