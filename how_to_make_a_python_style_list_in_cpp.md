

Today I will discuss about the industry problems I face in C++ array or vector. 

### Stability problems

We face stability problems due to memory corruptions.

Memory corruption can happen because of buffer overflow or pointer invalidation.

Pointer invalidation happens in vector, probably this is why Rust was invented.


##### Solution

There are a lot of cases where we know the maximum-size of the data, and we can just preallocate the memory needed to hold the data. We can easily maintain a fixed size array of pointers in that case.

```
#define MAX_ITEMS 128
std::array<int, MAX_ITEMS> items;
```

In case the content type of the array has bigger size, we can easily use pointer of those content-type.

```
std::array<int*, MAX_ITEMS> items;
```

But then there is a problem , how many of these items are valid , how we can add or delete items. One possible solution is to use optional in case we have value in the array, otherwise we can use nullpointer.

```
std::array<int*, MAX_ITEMS> pointerItems; 
std::array<std::optional<int>, MAX_ITEMS> valueItems; 
```

In case we need it, we can also use shared or unique pointers.

```
std::array<std::unique_ptr<int>, MAX_ITEMS> pointerItems; 
```

Now to have a vector like `idioms`, such as `size` and `capacity`, we can use this code.

```
template<typename T, const size_t CAPACITY=128>
struct Arr final {
    using Iterator = T*;

    Iterator begin()
    {
        return &data[0];
    }

    Iterator end()
    {
        return &data[cnt];
    }

    void push_back(T given) {
        auto&self = *this;
        
        assert(cnt < CAPACITY);
        self.data[self.cnt] = given;
        self.cnt++;
    }


    void emplace(T&&given) {
        auto&self = *this;
        
        assert(cnt < CAPACITY);
        self.data[self.cnt] = std::move(given);
        self.cnt++;
    }

    T& back(size_t ridx=0) {
        auto& self = *this;
        assert(self.cnt > ridx and self.cnt < CAPACITY);
        return self.data[self.cnt-1-ridx];
    }

    T& operator[](size_t idx) {
        auto& self = *this;
        assert(self.cnt > idx); // NOTICE this "assert" here that will protect from buffer overflow.
        return self.data[idx];
    }

    bool empty() const {
        return 0 == cnt;
    }

    std::size_t size() const {
        return cnt;
    }
    T data[CAPACITY];
    std::size_t cnt = 0;
};
```

Notice the assert in the code that will protect from buffer overflow.

Here is an example usage from legacy style DPDK code, how we use modern C++ there,

```
/*
 * Read packet from RX queues
 */
for( auto& portInfo : self.portList ) {
    auto portId = portInfo.portId;
    auto nb_rx = rte_eth_rx_burst(portId, 0, self.pkts_burst.begin(),
                 MAX_PKT_BURST);

    portInfo.rx += nb_rx;
    self.pkts_burst.cnt = nb_rx;

    for(auto*m : self.pkts_burst) {
        rte_prefetch0(rte_pktmbuf_mtod(m, void *));
        self.processRx(m, portId);
    }
}
```

If this array is somehow part of a message payload, then we do not need to allocate the `MAX_ITEMS` memory . We should have a pay as you go strategy. So instead of `T data[CAPACITY];` in `Arr`,
we use `T *data = nullptr;`. And finally we can allocate the memory dynamically in the constructor and destroy in the destructor.


```

template <typename T>
struct Slice final
{
    using Iterator = T*;

    Iterator begin()
    {
        return &data[0];
    }

    Iterator end()
    {
        return &data[cnt];
    }

    bool push_back(T given) {
        auto&self = *this;
        
        if(self.cnt >= self.capacity_) {
            return false;
        }
        self.data[self.cnt] = given;
        self.cnt++;
        return true;
    }


    bool emplace(T&&given) {
        auto&self = *this;
        
        if(self.cnt >= self.capacity_) {
            return false;
        }
        self.data[self.cnt] = std::move(given);
        self.cnt++;
        return true;
    }

    T& back(size_t ridx=0) {
        auto& self = *this;
        assert(self.cnt > ridx and self.cnt < self.capacity_);
        return self.data[self.cnt-1-ridx];
    }

    T& operator[](size_t idx) {
        auto& self = *this;
        assert(self.cnt > idx); // NOTICE this "assert" here that will protect from buffer overflow.
        return self.data[idx];
    }

    bool empty() const {
        return 0 == cnt;
    }

    std::size_t size() const {
        return cnt;
    }

    std::size_t capacity() const {
        return capacity_;
    }

    T *data = nullptr;
    std::size_t cnt = 0;
    std::size_t capacity_ = 0;
};

```
