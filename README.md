# Synchronization problems

Problems commonly found when implementing concurrency and parallelism.

Students:
- Nguyen Thai Khoi 20224868
- Nguyen Thai Hoa 20224850
- Vu Tung Lam 20225140

## Concurrency
In computer science, concurrency is the ability of different parts or units of a program, algorithm, or problem to be executed out-of-order or in partial order, without affecting the outcome. This allows for parallel execution of the concurrent units, which can significantly improve overall speed of the execution in multi-processor and multi-core systems. In more technical terms, concurrency refers to the decomposability of a program, algorithm, or problem into order-independent or partially-ordered components or units of computation.

According to Rob Pike, concurrency is the composition of independently executing computations, and concurrency is not parallelism: concurrency is about dealing with lots of things at once but parallelism is about doing lots of things at once. Concurrency is about structure, parallelism is about execution, concurrency provides a way to structure a solution to solve a problem that may (but not necessarily) be parallelizable.

The following table compares the differences between different forms of execution.

|             | Singlethreading (synchronous) | Singlethreading (asynchronous) | Multithreading | Multiprocessing |
| ----------- | :---------------------------: | :----------------------------: | :------------: | :-------------: |
| Concurrency | | x | x | x |
| Parallelism | | | x | x |

## Synchronization primitives

### Event

Event object is used to notify multiple threads that an event has happened. It usually has 3 public methods: `wait`, `set`, `clear`.

An event object manages an internal flag. If the flag is `true`, any calls to `wait` will return immediately. If the flag is `false`, the wait method will suspend the current thread and wait for the flag to become `true` before returning.

The internal flag can be switched by `set` and `clear` methods.

### Lock

Mutex lock to guarantee exclusive access to a shared state. It usually has 2 public methods: `acquire` and `release`. A Lock object can be in one of two states: "locked" or "unlocked".

If the lock is "locked", all threads that call `acquire` will be put in a waiting FIFO queue and will proceed in order for each release call.

If the lock is "unlocked", calling `acquire` will set the lock to the "locked" state and return immediately.

The `release` method will release the lock held by the current thread. If the lock isn't acquired then this method does nothing.

### Semaphore

A semaphore object that allows a limited number of threads to acquire it. Similar to a lock, the semaphore usually has 2 public methods: `acquire` and `release`.

A semaphore is a synchronization primitive that maintains a counter indicating the number of available resources or permits. In this implementation, the semaphore keeps track of an internal counter. The counter is decremented each time a future acquires the semaphore using the `acquire` method and incremented each time the semaphore is released using the `release` method.

When multiple threads are waiting for the semaphore, they will be put in a FIFO queue and only the first one will proceed when the semaphore becomes available.

If the internal counter of the semaphore has an upper limit of 1, it is identical to a lock.

## ABA Problem

The ABA problem is a subtle challenge that can arise in multithreaded programming when dealing with shared memory. It occurs during synchronization, specifically when relying solely on a variable's current value to determine if data has been modified.

The ABA problem occurs when multiple threads (or processes) accessing shared data interleave. Below is a sequence of events that illustrates the ABA problem:

1. Process $P_1$ reads value A from some shared memory location.

2. $P_1$ is preempted, allowing process $P_2$ to run.

3. $P_2$ writes value B to the shared memory location.

4. $P_2$ writes value A to the shared memory location.

5. $P_2$ is preempted, allowing process $P_1$ to run.

6. $P_1$ reads value A from the shared memory location.

7. $P_1$ determines that the shared memory value has not changed and continues.

### Example

An example illustrating how the ABA problem occurs:

```cpp
#include <atomic>
#include <iostream>
#include <stdexcept>
#include <windows.h>

class __StackNode
{
public:
    const int value;
    bool valid = true;  // for demonstration purpose
    __StackNode *next;

    __StackNode(int value, __StackNode *next) : value(value), next(next) {}
    __StackNode(int value) : __StackNode(value, nullptr) {}
};

class Stack
{
private:
    std::atomic<__StackNode *> _head_ptr = nullptr;

public:
    /** Pop the head object and return its value */
    int pop(bool sleep)
    {
        while (true)
        {
            __StackNode *head_ptr = _head_ptr;
            if (head_ptr == nullptr)
            {
                throw std::out_of_range("Empty stack");
            }

            // For simplicity, suppose that we can ensure that this dereference is safe
            // (i.e., that no other thread has popped the stack in the meantime).
            __StackNode *next_ptr = head_ptr->next;

            if (sleep)
            {
                Sleep(200); // interleave
            }

            // If the head node is still head_ptr, then assume no one has changed the stack.
            // (That statement is not always true because of the ABA problem)
            // Atomically replace head node with next.
            if (_head_ptr.compare_exchange_weak(head_ptr, next_ptr))
            {
                int result = head_ptr->value;
                head_ptr->valid = false; // mark this memory as released
                delete head_ptr;
                return result;
            }
            // The stack has changed, start over.
        }
    }

    /** Push a value to the stack */
    void push(int value)
    {
        __StackNode *value_ptr = new __StackNode(value);
        while (true)
        {
            __StackNode *next_ptr = _head_ptr;
            value_ptr->next = next_ptr;
            // If the head node is still next, then assume no one has changed the stack.
            // (That statement is not always true because of the ABA problem)
            // Atomically replace head with value.
            if (_head_ptr.compare_exchange_weak(next_ptr, value_ptr))
            {
                return;
            }
            // The stack has changed, start over.
        }
    }

    void print()
    {
        __StackNode *current = _head_ptr;
        while (current != nullptr)
        {
            std::cout << current->value << " ";
            current = current->next;
        }
        std::cout << std::endl;
    }

    void assert_valid()
    {
        __StackNode *current = _head_ptr;
        while (current != nullptr)
        {
            if (!current->valid)
            {
                throw std::runtime_error("Invalid stack node");
            }
            current = current->next;
        }
    }
};

DWORD WINAPI thread_proc(void *_stack)
{
    Stack *stack = (Stack *)_stack;
    stack->pop(false);
    stack->pop(false);
    stack->push(40);

    return 0;
}

int main()
{
    Stack stack;
    stack.push(10);
    stack.push(20);
    stack.push(30);
    stack.push(40);

    HANDLE thread = CreateThread(
        NULL,         // lpThreadAttributes
        0,            // dwStackSize
        &thread_proc, // lpStartAddress
        &stack,       // lpParameter
        0,            // dwCreationFlags
        NULL          // lpThreadId
    );

    stack.pop(true);

    WaitForSingleObject(thread, INFINITE);
    CloseHandle(thread);

    stack.print();
    stack.assert_valid();  // randomly crash

    return 0;
}
```

In the example above, 2 threads are allowed to run concurrently. To promote the occurrence of the ABA problem, the main thread calls `stack.pop(true)` with `Sleep(200)` to suspend its execution, allowing other threads to run. Before interleaving the main thread, the local pointer variable `head_ptr` and `next_ptr` point to `40` and `30`, respectively. We can assume that while the main thread is sleeping, the second one completes all of its operations (2 `pop`s and 1 `push`), transforms the stack into `[10, 20, 40]` and deallocates the memory of the node `30`. Afterwards, the main thread resumes its execution, where the method call `_head_ptr.compare_exchange_weak(head_ptr, next_ptr)` returns `true` (since the top of the stack is still `40`). It then sets `next_ptr` (which is a pointer to `30`) as the top of the stack. But since this memory is deallocated, it is unsafe to perform the assignment. Thus running the program above will randomly crash at the line `stack.assert_valid()`.

It is worth noticing that since the memory pointed by `next_ptr` may not be touched anyway after deallocation, assigning `next_ptr` in the main thread still appears to work correctly (in this example, we have to clear a flag `valid` to mark the node as deallocated). Accessing freed memory is undefined behavior: this may result in crashes, data corruption or even just silently appear to work correctly.

#### Workarounds

A common workaround is to add extra "tag" or "stamp" bits to the quantity being considered. For example, an algorithm using compare and swap on a pointer might use the low bits of the address to indicate how many times the pointer has been successfully modified. Because of this, the next compare-and-swap will fail, even if the addresses are the same, because the tag bits will not match. This is sometimes called ABAʹ since the second A is made slightly different from the first. Such tagged state references are also used in transactional memory. Although a tagged pointer can be used for implementation, a separate tag field is preferred if double-width CAS is available.

In the example above, it is possible to add a `version` field to the `Stack`, indicating the number of times the stack has been updated. Instead of comparing `_head_ptr.compare_exchange_weak(next_ptr, value_ptr)`, we can compare the stack's version before and after performing assignments.

### Addressing the ABA Problem

Several approaches can help mitigate the ABA problem:

* **Timestamps:** By associating a version number or timestamp with the data, threads can ensure the value hasn't changed between reads. An inconsistency arises if the version retrieved during the second read doesn't match the version associated with the initial value.
* **Locking mechanisms:** Utilizing synchronization primitives like mutexes or spinlocks ensures exclusive access to the shared memory location during critical sections. This prevents other threads from modifying the data while the first thread is performing its read-modify-write operation.
* **Atomic operations:** In specific scenarios, employing atomic operations that combine read and write into a single, indivisible step can eliminate the window of vulnerability between reads. However, atomic operations might not be suitable for all situations.

By understanding the ABA problem and implementing appropriate synchronization techniques, developers can ensure the integrity of data in multithreaded environments.

## Cigarette smokers problem

The cigarette smokers problem is a classic concurrency problem in computer science. This problem highlights the challenges of coordinating multiple processes or threads that share resources and need to synchronize their actions.

Here is the description for this problem
* **Ingredients:** Imagine a scenario where there are three ingredients required to make and smoke a cigarette: tobacco, paper, and matches.
* **Participants:** Around a table, there are three smokers, each of whom has an infinite supply of one of the three ingredients:
  1. One smoker has an infinite supply of tobacco.
  2. Another smoker has an infinite supply of paper.
  3. The third smoker has an infinite supply of matches.
* **Non-smoking Agent:** There is also a non-smoking agent who enables the smokers to make their cigarettes. The agent randomly selects two of the supplies and places them on the table.
* **Smoking Process:**
  1. The smoker who has the third supply should remove the two items from the table.
  2. They use these items (along with their own supply) to make a cigarette, which they smoke for a while.
  3. Once the smoker finishes smoking, the agent places two new random items on the table.
  4. This process continues indefinitely.

### Example

```cpp
#include <chrono>
#include <iostream>
#include <random>
#include <windows.h>

std::mt19937 rng(std::chrono::steady_clock::now().time_since_epoch().count());

template <typename T>
T random_int(const T l, const T r)
{
    std::uniform_int_distribution<T> unif(l, r);
    return unif(rng);
}

HANDLE smoking;
std::vector<HANDLE> semaphores;

void synchronization_primitives()
{
    smoking = CreateSemaphoreW(NULL, 0, 1, NULL);
    for (int i = 0; i < 3; i++)
    {
        semaphores.push_back(CreateSemaphoreW(NULL, 0, 1, NULL));
    }
}

DWORD WINAPI agent(void *)
{
    while (true)
    {
        int ingredient = random_int(0, 2), next_ingredient = (1 + ingredient) % 3;
        std::cout << "Got ingredients " << ingredient << ", " << next_ingredient << std::endl;

        ReleaseSemaphore(semaphores[ingredient], 1, NULL);
        ReleaseSemaphore(semaphores[next_ingredient], 1, NULL);
        WaitForSingleObject(smoking, INFINITE);
    }

    return 0;
}

DWORD WINAPI smoker(void *ptr)
{
    int ingredient = *(int *)ptr;
    while (true)
    {
        WaitForSingleObject(semaphores[(ingredient + 1) % 3], INFINITE);
        std::cout << "Smoker " << ingredient << " got " << (ingredient + 1) % 3 << std::endl;
        WaitForSingleObject(semaphores[(ingredient + 2) % 3], INFINITE);
        std::cout << "Smoker " << ingredient << " got " << (ingredient + 2) % 3 << std::endl;
        std::cout << "Smoker " << ingredient << " is smoking" << std::endl;
        Sleep(500);
        std::cout << "Smoker " << ingredient << " is done" << std::endl;
        ReleaseSemaphore(smoking, 1, NULL);
    }

    return 0;
}

int main()
{
    synchronization_primitives();

    std::vector<HANDLE> threads;
    threads.push_back(
        CreateThread(
            NULL,   // lpThreadAttributes
            0,      // dwStackSize
            &agent, // lpStartAddress
            NULL,   // lpParameter
            0,      // dwCreationFlags
            NULL)   // lpThreadId
    );

    int *ingredient_ptr[3];
    for (int i = 0; i < 3; i++)
    {
        ingredient_ptr[i] = new int(i);
        threads.push_back(
            CreateThread(
                NULL,              // lpThreadAttributes
                0,                 // dwStackSize
                &smoker,           // lpStartAddress
                ingredient_ptr[i], // lpParameter
                0,                 // dwCreationFlags
                NULL)              // lpThreadId
        );
    }

    for (auto &thread : threads)
    {
        WaitForSingleObject(thread, INFINITE);
        CloseHandle(thread);
    }

    for (int i = 0; i < 3; i++)
    {
        delete ingredient_ptr[i];
    }

    return 0;
}
```

In the example above, The `agent` thread randomly places two ingredients on the table. Each `smoker` thread waits for the required ingredients, makes a cigarette, smokes it, and then signals the agent to continue.

### Real-world implications

The Cigarette Smokers Problem in computer science, beyond being a theoretical exercise, has real-world implications, particularly in the field of concurrent programming and operating systems. Here are some of the key implications:
* **Deadlock Prevention** The problem demonstrates a scenario where deadlock can occur if resources are not managed properly. In real-world systems, such as databases and operating systems, managing access to shared resources is crucial to prevent deadlock, which can cause systems to halt or become unresponsive.
* **Resource Allocation** It illustrates the challenges in allocating limited resources among competing processes or threads. This is analogous to real-world situations where multiple applications or users require access to a finite set of resources, such as CPU time, memory, or network bandwidth.
* **Synchronization Mechanisms** The problem highlights the importance of proper synchronization mechanisms, like semaphores, locks, and condition variables, to coordinate the actions of concurrent processes. These mechanisms are widely used in developing multi-threaded applications, ensuring that processes operate in the correct sequence without interfering with each other.
* **System Design** It emphasizes the need for careful system design to avoid complex interdependencies that can lead to deadlock. This is relevant for designing systems that are robust, scalable, and maintainable.
* **Understanding Concurrency** The problem serves as an educational tool to help programmers understand the complexities of concurrency, which is essential for developing efficient and reliable software in a multi-core and distributed computing world.
* **Semaphore Limitations** The problem also points out the limitations of traditional semaphores and the need for more powerful synchronization primitives in certain scenarios.

### Addressing the Cigarette Smokers Problem

Several approaches can help mitigate the Cigarette Smokers Problem:
* **Deadlock Avoidance:** Implement a deadlock avoidance strategy to prevent the system from entering a deadlock state.
* **Priority-Based Allocation:** Assign priorities to processes or threads. When allocating resources, give preference to higher-priority processes. This helps prevent low-priority processes from blocking critical resources indefinitely.
* **Resource Pooling:** Create a pool of resources (e.g., semaphores or locks) that processes can request. When a process is done, it releases the resource back to the pool. This avoids resource exhaustion and ensures fair access.
* **Two-Phase Locking:** In database management systems, use two-phase locking to ensure that transactions acquire and release locks in a consistent order. This helps prevent deadlocks during data updates.
* **Timeouts and Rollbacks:** If a process waits too long for a resource, introduce a timeout. If the timeout expires, the process releases its resources and rolls back its work. This prevents indefinite waiting.
* **Resource Hierarchies:** Assign a hierarchy to resources (e.g., locks). Processes must acquire resources in a specific order (from lower to higher levels). This prevents circular waits.
* **Dynamic Resource Allocation:** Dynamically allocate resources based on demand. For example, allocate memory or threads as needed and release them when no longer required.
* **Avoidance of Hold-and-Wait:** Processes should request all required resources upfront (non-preemptive). If a process cannot acquire all resources, it releases any acquired resources and retries later.
* **Preemption:** If a high-priority process needs a resource held by a lower-priority process, preempt the lower-priority process and allocate the resource to the higher-priority one.
* **Transaction Serialization:** In database systems, ensure that transactions are serialized (executed one after the other) to avoid conflicts and deadlocks.

In summary, the Cigarette Smokers Problem is not just a theoretical construct but a representation of the real challenges faced in concurrent programming and system design. It encourages developers to think critically about process synchronization, resource allocation, and system robustness in the context of concurrent operations.

### Barrier Synchronization Problems

Barrier synchronization problems arise in parallel computing when multiple threads or processes must reach a synchronization point (or barrier) at the same time. The primary purpose of a barrier is to ensure that no thread or process proceeds beyond a certain point until all others have reached that point. However, various issues can occur with this synchronization mechanism:

#### Detailed Explanation

1. **Uneven Workload Distribution**:
   - **Problem**: If the workload is unevenly distributed among threads, some threads may finish their tasks much earlier than others and wait idly at the barrier, leading to inefficient use of resources.
   - **Example**: In a parallel algorithm where each thread processes a portion of an array, if some portions take significantly longer to process than others, faster threads will spend time waiting at the barrier.

2. **Straggler Effect**:
   - **Problem**: A single slow thread (straggler) can delay the entire batch of threads at the barrier, causing performance degradation.
   - **Example**: In a distributed system, if one node is slower due to network latency or lower processing power, it will cause all other nodes to wait, affecting overall system performance.

3. **Resource Contention**:
   - **Problem**: When multiple threads converge on a barrier, there can be contention for the resources managing the barrier (e.g., locks or semaphores), potentially leading to performance bottlenecks.
   - **Example**: If a barrier implementation uses a shared lock, contention for this lock can increase as the number of threads grows, leading to increased wait times.

4. **Deadlock**:
   - **Problem**: Incorrectly implemented barriers can lead to deadlocks, where threads are permanently blocked waiting at the barrier due to a logical error in the synchronization code.
   - **Example**: If a barrier is supposed to synchronize 10 threads but is mistakenly programmed to wait for 11, all threads will wait indefinitely, causing a deadlock.

5. **Livelock**:
   - **Problem**: Although less common, livelock can occur if threads constantly change state in response to each other without making progress through the barrier.
   - **Example**: Threads repeatedly entering and leaving the barrier due to incorrect signaling can lead to livelock, where no thread progresses past the barrier.

6. **Complexity in Nested Barriers**:
   - **Problem**: In complex programs with nested barriers, ensuring correct synchronization can be challenging and prone to errors, leading to unexpected behavior.
   - **Example**: If an outer barrier depends on the completion of an inner barrier, and there's a misconfiguration, threads may be incorrectly synchronized, causing logic errors.

7. **High Overhead**:
   - **Problem**: The overhead associated with managing barriers can become significant, especially in systems with a large number of threads, leading to performance issues.
   - **Example**: The time taken to manage and coordinate the barrier increases with the number of participating threads, reducing the benefits of parallelism.

#### Solutions and Best Practices

1. **Dynamic Load Balancing**:
   - Implement dynamic load balancing to ensure more even distribution of work among threads, reducing the likelihood of uneven workload distribution.

2. **Hierarchical Barriers**:
   - Use hierarchical barriers that synchronize threads in smaller groups before synchronizing the entire set, reducing contention and overhead.

3. **Timeout Mechanisms**:
   - Implement timeout mechanisms to detect and handle situations where threads are waiting too long at a barrier, potentially indicating a problem like a deadlock.

4. **Profiling and Optimization**:
   - Profile the application to identify bottlenecks and optimize the code to reduce the time threads spend waiting at barriers.

5. **Efficient Barrier Implementations**:
   - Use efficient barrier implementations that minimize contention and overhead, such as tree-based barriers or software combining trees.

6. **Graceful Degradation**:
   - Design the system to handle stragglers gracefully, allowing other threads to perform useful work while waiting.

By understanding and addressing these issues, developers can design more efficient and robust parallel programs that make effective use of barrier synchronization.

### Atomicity Violation

Atomicity violations occur when a sequence of operations that should be executed as a single, indivisible (atomic) operation are interrupted by other threads, leading to inconsistent or incorrect results. This issue is common in concurrent programming, where multiple threads access and modify shared data.

#### Detailed Explanation

1. **Definition of Atomicity**:
   - **Atomic Operation**: An operation or a set of operations that are performed as a single unit without interference from other operations. Either all operations are executed, or none are, ensuring data consistency.

2. **Common Scenarios for Atomicity Violations**:
   - **Check-Then-Act**: A thread checks a condition and then acts based on the result, but another thread changes the condition in between.
   - **Read-Modify-Write**: A thread reads a value, modifies it, and writes it back, but another thread modifies the value in between the read and write steps.

3. **Example Scenarios**:
   - **Bank Account Example**:
     - Suppose two threads are transferring money from a shared bank account. The first thread checks the balance to ensure there are sufficient funds before making a transfer, while the second thread is simultaneously transferring money out. Without synchronization, both threads could read the same initial balance and proceed with the transfer, resulting in an overdrawn account.
   - **Counter Increment Example**:
     - Consider two threads incrementing a shared counter. Both threads read the current value, increment it, and write it back. Without synchronization, both threads might read the same value, increment it, and write back the same result, causing one increment to be lost.

4. **Consequences of Atomicity Violations**:
   - **Data Corruption**: Shared data can become inconsistent or corrupted.
   - **Incorrect Program Behavior**: The program may produce incorrect results or exhibit unexpected behavior.
   - **Security Vulnerabilities**: In some cases, atomicity violations can lead to security issues, such as race conditions exploited by attackers.

5. **Detection and Debugging**:
   - **Testing and Debugging**: Atomicity violations can be difficult to detect through testing because they may only occur under specific timing conditions. Tools like thread analyzers and race condition detectors can help identify these issues.
   - **Code Review**: Careful code review and understanding of concurrent programming principles can help spot potential atomicity violations.

6. **Preventive Measures and Solutions**:
   - **Locks (Mutexes)**:
     - Use locks to ensure that a sequence of operations is executed atomically. For example, acquire a lock before checking and modifying a shared variable and release it afterward.
   - **Atomic Operations**:
     - Use atomic operations provided by the programming language or library (e.g., `AtomicInteger` in Java, `std::atomic` in C++) to perform atomic read-modify-write operations.
   - **Transaction Memory**:
     - Utilize transactional memory systems where a series of read and write operations are grouped into a transaction, ensuring atomicity.
   - **Higher-Level Concurrency Constructs**:
     - Use higher-level constructs like semaphores, barriers, or synchronized collections that manage synchronization internally to prevent atomicity violations.
   - **Volatile Keyword**:
     - In languages like Java, use the `volatile` keyword to ensure visibility and ordering of changes to a variable across threads, though this alone does not ensure atomicity.

7. **Programming Practices**:
   - **Minimize Shared Data**: Reduce the amount of shared data that needs to be accessed concurrently.
   - **Immutable Objects**: Use immutable objects to avoid the need for synchronization on shared data.
   - **Design for Concurrency**: Design the application with concurrency in mind from the start, considering how threads will interact with shared resources.

### Lost Wakeup

**Lost wakeup** occurs in concurrent programming when a thread misses a signal (or wakeup call) indicating that a condition it is waiting for has been met, leading to potential indefinite blocking. This typically happens when using condition variables, semaphores, or other synchronization mechanisms.

#### Detailed Explanation

1. **Basic Concept**:
   - In concurrent programming, threads often need to wait for certain conditions to be true before proceeding. They typically wait on condition variables or similar constructs and are signaled (or "woken up") when the condition is met.
   - A lost wakeup occurs when the signal intended to wake up a waiting thread is sent before the thread starts waiting, causing the thread to miss the signal and remain blocked indefinitely.

2. **Common Scenarios**:
   - **Condition Variables**:
     - Threads wait on a condition variable and are expected to be notified when a particular condition is true. If the notify call happens before the thread starts waiting, the thread will miss the signal and might wait forever.
   - **Semaphores**:
     - A thread waits for a semaphore to be released. If the release signal is sent before the thread starts waiting, the semaphore count is incremented, but the thread may not notice it and continue to wait.

3. **Example Scenario**:
   - **Producer-Consumer Problem**:
     - Consider a producer-consumer scenario where the consumer waits for an item to be produced. If the producer signals the condition variable before the consumer starts waiting, the consumer may miss the signal and wait indefinitely even though an item is available.

4. **Consequences of Lost Wakeup**:
   - **Indefinite Blocking**: The most severe consequence is that the affected thread remains blocked indefinitely, causing parts of the application to become unresponsive.
   - **Performance Degradation**: Even if the application does not deadlock, lost wakeups can lead to performance issues as threads spend unnecessary time waiting.

5. **Detection and Debugging**:
   - **Difficult to Reproduce**: Lost wakeups are often hard to detect and reproduce because they depend on specific timing conditions.
   - **Thorough Testing**: Extensive and thorough testing, including stress testing and using concurrency testing tools, can help identify lost wakeup issues.
   - **Code Review**: Careful review of synchronization logic can help spot potential causes of lost wakeup.

6. **Preventive Measures and Solutions**:
   - **Always Use Locks with Condition Variables**:
     - Ensure that the condition check and the wait call are both made while holding a lock. This prevents the condition from being signaled before the thread is ready to wait.
   - **Double-Checked Locking**:
     - Before waiting, check the condition without holding the lock. Then, acquire the lock and check the condition again. Only wait if the condition is still false.
   - **Spurious Wakeups**:
     - Be aware that some systems can cause spurious wakeups, where a thread is woken up without a corresponding signal. Always use a loop to recheck the condition after being woken up.
   - **Semaphore Alternatives**:
     - Use other synchronization primitives that may be less prone to lost wakeups, such as `CountDownLatch` or `CyclicBarrier` in Java.
