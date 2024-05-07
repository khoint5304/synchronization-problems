## [ABA Problem](https://en.wikipedia.org/wiki/ABA_problem)

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

### Real-world implications

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
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>

// Ingredients
enum Ingredient { TOBACCO, PAPER, MATCHES };

// Shared resources
bool tobacco_on_table = false;
bool paper_on_table = false;
bool matches_on_table = false;

// Semaphores
std::mutex mtx;
std::condition_variable cv;

// Agent places random items on the table
void agent()
{
    while (true)
    {
        std::unique_lock<std::mutex> lock(mtx);
        // Randomly select two ingredients
        int random_ingredient1 = rand() % 3;
        int random_ingredient2 = (random_ingredient1 + 1) % 3;

        if (random_ingredient1 == TOBACCO && random_ingredient2 == PAPER)
        {
            tobacco_on_table = true;
            paper_on_table = true;
        }
        else if (random_ingredient1 == PAPER && random_ingredient2 == MATCHES)
        {
            paper_on_table = true;
            matches_on_table = true;
        }
        else if (random_ingredient1 == MATCHES && random_ingredient2 == TOBACCO)
        {
            matches_on_table = true;
            tobacco_on_table = true;
        }

        cv.notify_all(); // Signal smokers
        cv.wait(lock);   // Wait for smoker to finish
    }
}

// Smoker thread
void smoker(int ingredient)
{
    while (true)
    {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [ingredient]
        {
            switch (ingredient)
            {
                case TOBACCO: return paper_on_table && matches_on_table;
                case PAPER: return tobacco_on_table && matches_on_table;
                case MATCHES: return tobacco_on_table && paper_on_table;
            }
            return false;
        });

        // Make and smoke the cigarette
        std::cout << "Smoker with ingredient " << ingredient << " is smoking." << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));

        // Clear the table
        tobacco_on_table = false;
        paper_on_table = false;
        matches_on_table = false;

        cv.notify_one(); // Signal agent
    }
}

int main()
{
    std::thread agent_thread(agent);
    std::thread smoker_tobacco(smoker, TOBACCO);
    std::thread smoker_paper(smoker, PAPER);
    std::thread smoker_matches(smoker, MATCHES);

    agent_thread.join();
    smoker_tobacco.join();
    smoker_paper.join();
    smoker_matches.join();

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
