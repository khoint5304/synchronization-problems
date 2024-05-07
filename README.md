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

In the example above, 2 threads are scheduled to run concurrently. To promote the occurrence of the ABA problem, the main thread calls `stack.pop(true)` with `Sleep(200)` to suspend its execution, allowing other threads to run. Before interleaving the main thread, the local pointer variable `head_ptr` and `next_ptr` points to `40` and `30`, respectively. We can assume that while the main thread is sleeping, the second one completes its operations (2 `pop`s and 1 `push`), transforms the stack into `[10, 20, 40]` and deallocates the memory of the node `30`. Afterwards, the main thread resumes its execution, where the method call `_head_ptr.compare_exchange_weak(head_ptr, next_ptr)` returns `true` (since the top of the stack is still `40`). It then sets `next_ptr` (which is a pointer to `30`) as the top of the stack. But since this memory is deallocated, it is unsafe to perform the assignment. Thus running the program above will randomly crash at the line `stack.assert_valid();`

It is worth noticing that since the memory pointed by `next_ptr` may not be touched anyway after deallocation, assigning `next_ptr` in the main thread still appears to work correctly (in this example, we have to clear a flag `valid` to mark the node as deallocated). Accessing freed memory is undefined behavior: this may result in crashes, data corruption or even just silently appear to work correctly.

### Real-world implications

### Addressing the ABA Problem

Several approaches can help mitigate the ABA problem:

* **Timestamps:** By associating a version number or timestamp with the data, threads can ensure the value hasn't changed between reads. An inconsistency arises if the version retrieved during the second read doesn't match the version associated with the initial value.
* **Locking mechanisms:** Utilizing synchronization primitives like mutexes or spinlocks ensures exclusive access to the shared memory location during critical sections. This prevents other threads from modifying the data while the first thread is performing its read-modify-write operation.
* **Atomic operations:** In specific scenarios, employing atomic operations that combine read and write into a single, indivisible step can eliminate the window of vulnerability between reads. However, atomic operations might not be suitable for all situations.

By understanding the ABA problem and implementing appropriate synchronization techniques, developers can ensure the integrity of data in multithreaded environments.

##[Race condition](https://en.wikipedia.org/wiki/Race_condition)

Bài toán "Race Condition" là một vấn đề phổ biến trong lập trình đa luồng, xảy ra khi hai hoặc nhiều tiến trình hoặc luồng cố gắng thực hiện cùng một tài nguyên hoặc hoạt động đồng thời mà không có sự đồng bộ hóa đúng đắn. Kết quả của việc này là không thể dự đoán được, và phụ thuộc vào thứ tự thực thi của các tiến trình hoặc luồng.

Trong một race condition, kết quả của một chương trình có thể thay đổi dựa trên cách thực thi của hệ thống, khiến cho kết quả không ổn định hoặc không đoán trước được. Điều này có thể gây ra các vấn đề nghiêm trọng như lỗi dữ liệu, sự cố không xác định được, hoặc thậm chí là lỗi bảo mật.

Các race condition thường xảy ra trong các trường hợp sau:
1. **Đọc/Ghi dữ liệu:** Khi một tiến trình đọc dữ liệu cùng lúc với một tiến trình khác đang ghi dữ liệu vào cùng một vị trí bộ nhớ.
2. **Thực thi đồng thời của các luồng:** Khi hai hoặc nhiều luồng cố gắng thực hiện một phần của mã mà có thể tác động đến dữ liệu chia sẻ hoặc trạng thái của hệ thống.
3. **Sử dụng tài nguyên hệ thống:** Khi các tiến trình hoặc luồng cố gắng truy cập và sử dụng cùng một tài nguyên hệ thống, chẳng hạn như một tệp hoặc một cổng I/O.

Để giải quyết vấn đề race condition, cần sử dụng các kỹ thuật đồng bộ hóa như mutex, semaphore, hoặc các cơ chế khác để đảm bảo rằng chỉ một tiến trình hoặc luồng có thể truy cập vào một tài nguyên hoặc hoạt động cùng một lúc.

### Example

```cpp
#include <iostream>
#include <windows.h>

int shared = 0;

void update()
{
    for (int i = 0; i < 200000; i++)
    {
        shared++;
    }
}

DWORD WINAPI thread_proc(void *)
{
    update();
    return 0;
}

int main()
{
    HANDLE thread = CreateThread(
        NULL,         // lpThreadAttributes
        0,            // dwStackSize
        &thread_proc, // lpStartAddress
        NULL,         // lpParameter
        0,            // dwCreationFlags
        NULL          // lpThreadId
    );

    update();

    WaitForSingleObject(thread, INFINITE);
    CloseHandle(thread);

    std::cout << shared << std::endl;

    return 0;
}
```

Bài toán "Race Condition" có thể ảnh hưởng đến nhiều lĩnh vực trong thực tế, từ phần mềm máy tính đến hệ thống điều khiển các thiết bị cơ học và điện tử. Dưới đây là một số ví dụ về ứng dụng của bài toán này trong thực tế:

1. **Hệ thống ghi log:**
   Trong một ứng dụng hoặc dịch vụ web, nhiều luồng hoặc tiến trình có thể cố gắng ghi log đồng thời vào một tệp log. Nếu không có sự đồng bộ hóa, các lời ghi log có thể bị ghi đè lên nhau, dẫn đến mất dữ liệu hoặc dữ liệu không hợp lệ.

2. **Quản lý tài nguyên mạng:**
   Trong hệ thống mạng, nhiều tiến trình hoặc luồng có thể cố gắng thay đổi cấu hình mạng hoặc quản lý kết nối cùng một lúc. Nếu không có sự đồng bộ hóa, có thể xảy ra tình trạng truy cập song song vào các tài nguyên mạng, dẫn đến lỗi kết nối hoặc mất mát dữ liệu.

3. **Ứng dụng đa luồng trên thiết bị di động:**
   Trong các ứng dụng di động, các luồng có thể cố gắng truy cập và cập nhật cùng một dữ liệu, chẳng hạn như dữ liệu người dùng hoặc cài đặt ứng dụng. Nếu không có sự đồng bộ hóa, có thể xảy ra tình trạng đọc/ghi không đồng bộ, dẫn đến mất mát dữ liệu hoặc trạng thái ứng dụng không đoán trước được.

4. **Hệ thống điều khiển tự động:**
   Trong hệ thống điều khiển tự động như hệ thống đo lường và điều khiển công nghiệp, nhiều tiến trình hoặc luồng có thể cố gắng điều khiển các thiết bị cùng một lúc. Nếu không có sự đồng bộ hóa, có thể xảy ra tình trạng không đoán trước được, dẫn đến việc hoạt động không ổn định hoặc nguy hiểm.

Trong tất cả các trường hợp trên, việc áp dụng các kỹ thuật đồng bộ hóa như mutex, semaphore, hoặc thậm chí là tái thiết kế cấu trúc dữ liệu để tránh race condition là rất quan trọng để đảm bảo tính ổn định và an toàn của hệ thống.
