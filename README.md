# SPSCQueuePow2.h

[![C/C++ CI](https://github.com/dhruvan2006/SPSCQueuePow2/workflows/C/C++%20CI/badge.svg)](https://github.com/dhruvan2006/SPSCQueuePow2/actions)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/dhruvan2006/SPSCQueuePow2/master/LICENSE)

A single producer single consumer wait-free and lock-free fixed size queue
written in C++11. This queue **requires the capacity to be a power of 2** for enhanced performance.

This implementation is faster than [*rigtorp::SPSCQueue*](https://github.com/rigtorp/SPSCQueue).

## Example

```cpp
SPSCQueuePow2<int> q(2);
auto t = std::thread([&] {
  while (!q.front());
  std::cout << *q.front() << std::endl;
  q.pop();
});
q.push(1);
t.join();
```

See `src/SPSCQueueExample.cpp` for the full example.

## Usage

- `SPSCQueuePow2<T>(size_t capacity);`

  Create a `SPSCqueuePow2` holding items of type `T` with capacity
  `capacity`. Capacity needs to be a power of 2.

- `void emplace(Args &&... args);`

  Enqueue an item using inplace construction. Blocks if queue is full.

- `bool try_emplace(Args &&... args);`

  Try to enqueue an item using inplace construction. Returns `true` on
  success and `false` if queue is full.

- `void push(const T &v);`

  Enqueue an item using copy construction. Blocks if queue is full.

- `template <typename P> void push(P &&v);`

  Enqueue an item using move construction. Participates in overload
  resolution only if `std::is_constructible<T, P&&>::value == true`.
  Blocks if queue is full.

- `bool try_push(const T &v);`

  Try to enqueue an item using copy construction. Returns `true` on
  success and `false` if queue is full.

- `template <typename P> bool try_push(P &&v);`

  Try to enqueue an item using move construction. Returns `true` on
  success and `false` if queue is full. Participates in overload
  resolution only if `std::is_constructible<T, P&&>::value == true`.

- `T *front();`

  Return pointer to front of queue. Returns `nullptr` if queue is
  empty.

- `void pop();`

  Dequeue first item of queue. You must ensure that the queue is non-empty
  before calling pop. This means that `front()` must have returned a
  non-`nullptr` before each call to `pop()`. Requires
  `std::is_nothrow_destructible<T>::value == true`.

- `size_t size();`

  Return the number of items available in the queue.

- `bool empty();`

  Return true if queue is currently empty.

Only a single writer thread can perform enqueue operations and only a
single reader thread can perform dequeue operations. Any other usage
is invalid.

## Huge page support

In addition to supporting custom allocation through the [standard custom
allocator interface](https://en.cppreference.com/w/cpp/named_req/Allocator) this
library also supports standard proposal [P0401R3 Providing size feedback in the
Allocator
interface](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p0401r3.html).
This allows convenient use of [huge
pages](https://www.kernel.org/doc/html/latest/admin-guide/mm/hugetlbpage.html)
without wasting any allocated space. Using size feedback is only supported when
C++17 is enabled.

The library currently doesn't include a huge page allocator since the APIs for
allocating huge pages are platform dependent and handling of huge page size and
NUMA awareness is application specific.

Below is an example huge page allocator for Linux:

```cpp
#include <sys/mman.h>

template <typename T> struct Allocator {
  using value_type = T;

  struct AllocationResult {
    T *ptr;
    size_t count;
  };

  size_t roundup(size_t n) { return (((n - 1) >> 21) + 1) << 21; }

  AllocationResult allocate_at_least(size_t n) {
    size_t count = roundup(sizeof(T) * n);
    auto p = static_cast<T *>(mmap(nullptr, count, PROT_READ | PROT_WRITE,
                                   MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                                   -1, 0));
    if (p == MAP_FAILED) {
      throw std::bad_alloc();
    }
    return {p, count / sizeof(T)};
  }

  void deallocate(T *p, size_t n) { munmap(p, roundup(sizeof(T) * n)); }
};
```

See `src/SPSCQueueExampleHugepages.cpp` for the full example on how to use huge
pages on Linux.

## Implementation

![Memory layout](https://github.com/dhruvan2006/SPSCQueuePow2/blob/master/spsc.svg)

The underlying implementation is based on a [ring
buffer](https://en.wikipedia.org/wiki/Circular_buffer).

By enforcing the queue's capacity to be a power of 2 (e.g., 2, 4, 8, 16, ...), the expensive branch operation `if (index==capacity_)` required to wrap around the ring buffer is replaced with a much faster bitwise AND operation `(index&(capacity−1))`.

This fork also removes the extra "padding" element used in the original implementation to distinguish between full and empty states, simplifying the buffer layout to be exactly capacity elements.

Care has been taken to make sure to avoid any issues with [false
sharing](https://en.wikipedia.org/wiki/False_sharing). The head and tail indices
are aligned and padded to the false sharing range (cache line size).
Additionally the slots buffer is padded with the false sharing range at the
beginning and end, this prevents false sharing with any adjacent allocations.

This implementation has higher throughput than a typical concurrent ring buffer
by locally caching the head and tail indices in the writer and reader
respectively. The caching increases throughput by reducing the amount of cache
coherency traffic.

To understand how that works first consider a read operation in absence of
caching: the head index (read index) needs to be updated and thus that cache
line is loaded into the L1 cache in exclusive state. The tail (write index)
needs to be read in order to check that the queue is not empty and is thus
loaded into the L1 cache in shared state. Since a queue write operation needs to
read the head index it's likely that a write operation requires some cache
coherency traffic to bring the head index cache line back into exclusive state.
In the worst case there will be one cache line transition from shared to
exclusive for every read and write operation.

Next consider a queue reader that caches the tail index: if the cached tail
index indicates that the queue is empty, then load the tail index into the
cached tail index. If the queue was non-empty multiple read operations up until
the cached tail index can complete without stealing the writer's tail index
cache line's exclusive state. Cache coherency traffic is therefore reduced. An
analogous argument can be made for the queue write operation.

References:

- *Intel*. [Avoiding and Identifying False Sharing Among Threads](https://software.intel.com/en-us/articles/avoiding-and-identifying-false-sharing-among-threads).
- *Wikipedia*. [Ring buffer](https://en.wikipedia.org/wiki/Circular_buffer).
- *Wikipedia*. [False sharing](https://en.wikipedia.org/wiki/False_sharing).

<details>
<summary>Click to view the full diff (SPSCQueuePow2.h vs. SPSCQueue.h)</summary>

```diff
44c44
< template <typename T, typename Allocator = std::allocator<T>> class SPSCQueuePow2 {
---
> template <typename T, typename Allocator = std::allocator<T>> class SPSCQueue {
58c58
<   explicit SPSCQueuePow2(const size_t capacity,
---
>   explicit SPSCQueue(const size_t capacity,
61,65c61,65
<     // Capacity must be power of 2
<     assert(capacity_ > 0 && ((capacity_ & (capacity_ - 1)) == 0) && "Capacity must be a power of 2");
< 
<     mask_ = capacity_ - 1;
< 
---
>     // The queue needs at least one element
>     if (capacity_ < 1) {
>       capacity_ = 1;
>     }
>     capacity_++; // Needs one slack element
68c68
<       throw std::length_error("Capacity too large");
---
>       capacity_ = SIZE_MAX - 2 * kPadding;
75c75
<       // capacity_ = res.count - 2 * kPadding; TODO: Is this safe
---
>       capacity_ = res.count - 2 * kPadding;
85,86c85,86
<     static_assert(alignof(SPSCQueuePow2<T>) == kCacheLineSize, "");
<     static_assert(sizeof(SPSCQueuePow2<T>) >= 3 * kCacheLineSize, "");
---
>     static_assert(alignof(SPSCQueue<T>) == kCacheLineSize, "");
>     static_assert(sizeof(SPSCQueue<T>) >= 3 * kCacheLineSize, "");
92c92
<   ~SPSCQueuePow2() {
---
>   ~SPSCQueue() {
101,102c101,102
<   SPSCQueuePow2(const SPSCQueuePow2 &) = delete;
<   SPSCQueuePow2 &operator=(const SPSCQueuePow2 &) = delete;
---
>   SPSCQueue(const SPSCQueue &) = delete;
>   SPSCQueue &operator=(const SPSCQueue &) = delete;
110c110,114
<     while (writeIdx - readIdxCache_ == capacity_) {
---
>     auto nextWriteIdx = writeIdx + 1;
>     if (nextWriteIdx == capacity_) {
>       nextWriteIdx = 0;
>     }
>     while (nextWriteIdx == readIdxCache_) {
113,114c117,118
<     new (&slots_[(writeIdx & mask_) + kPadding]) T(std::forward<Args>(args)...);
<     writeIdx_.store(writeIdx + 1, std::memory_order_release);
---
>     new (&slots_[writeIdx + kPadding]) T(std::forward<Args>(args)...);
>     writeIdx_.store(nextWriteIdx, std::memory_order_release);
123c127,131
<     if (writeIdx - readIdxCache_ == capacity_) {
---
>     auto nextWriteIdx = writeIdx + 1;
>     if (nextWriteIdx == capacity_) {
>       nextWriteIdx = 0;
>     }
>     if (nextWriteIdx == readIdxCache_) {
125c133
<       if (writeIdx - readIdxCache_ == capacity_) {
---
>       if (nextWriteIdx == readIdxCache_) {
129,130c137,138
<     new (&slots_[(writeIdx & mask_) + kPadding]) T(std::forward<Args>(args)...);
<     writeIdx_.store(writeIdx + 1, std::memory_order_release);
---
>     new (&slots_[writeIdx + kPadding]) T(std::forward<Args>(args)...);
>     writeIdx_.store(nextWriteIdx, std::memory_order_release);
168c176
<     return &slots_[(readIdx & mask_) + kPadding];
---
>     return &slots_[readIdx + kPadding];
177,178c185,190
<     slots_[(readIdx & mask_) + kPadding].~T();
<     readIdx_.store(readIdx + 1, std::memory_order_release);
---
>     slots_[readIdx + kPadding].~T();
>     auto nextReadIdx = readIdx + 1;
>     if (nextReadIdx == capacity_) {
>       nextReadIdx = 0;
>     }
>     readIdx_.store(nextReadIdx, std::memory_order_release);
182,183c194,199
<     return writeIdx_.load(std::memory_order_acquire) -
<            readIdx_.load(std::memory_order_acquire);
---
>     std::ptrdiff_t diff = writeIdx_.load(std::memory_order_acquire) -
>                           readIdx_.load(std::memory_order_acquire);
>     if (diff < 0) {
>       diff += capacity_;
>     }
>     return static_cast<size_t>(diff);
191c207
<   RIGTORP_NODISCARD size_t capacity() const noexcept { return capacity_; }
---
>   RIGTORP_NODISCARD size_t capacity() const noexcept { return capacity_ - 1; }
206d221
<   size_t mask_;
```
</details>

## Testing

Testing lock-free algorithms is hard. I'm using two approaches to test
the implementation:

- A single threaded test that the functionality works as intended,
  including that the item constructor and destructor is invoked
  correctly.
- A multi-threaded fuzz test verifies that all items are enqueued and dequeued
  correctly under heavy contention.

## Benchmarks

Throughput benchmark measures throughput between 2 threads for a queue of `int`
items.

Latency benchmark measures round trip time between 2 threads communicating using
2 queues of `int` items.

Benchmark results for a AMD Ryzen 9 6900HS 8-Core Processor, the 2 threads are
running on different cores on the same chiplet:

| Queue                 | Throughput (ops/ms) | Latency RTT (ns) | Multiplier |
|-----------------------|--------------------:|-----------------:|-----------:|
| SPSCQueuePow2         |              235962 |               64 |  **1.13x** |
| rigtorp::SPSCQueue    |              208813 |               64 |       1.0x |
| boost::lockfree::spsc |              118993 |               78 |      0.57x |

## Cited by

SPSCQueue have been cited by the following papers:

- Peizhao Ou and Brian Demsky. 2018. Towards understanding the costs of avoiding
  out-of-thin-air results. Proc. ACM Program. Lang. 2, OOPSLA, Article 136
  (October 2018), 29 pages. DOI: <https://doi.org/10.1145/3276506>

## About

This project is a fork of the original SPSCQueue created by [Erik Rigtorp](http://rigtorp.se)
<[erik@rigtorp.se](mailto:erik@rigtorp.se)>.

This SPSCQueuePow2 fork was created by [Dhruvan Gnanadhandayuthapani](https://dhruvan.dev) <[dhruvan2006@gmail.com](mailto:dhruvan2006@gmail.com)>.