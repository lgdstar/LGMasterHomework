## cache_t 源码分析

在 《6-2》结尾提出了一些 `cache_t` 相关的问题，这些问题的探索就要追寻到源码的逻辑中去了

上面提到的问题，都与数据有关，那么再次查看 `cache_t` 的成员变量

```C++
struct cache_t {
private:
    explicit_atomic<uintptr_t> _bucketsAndMaybeMask; 
    union {
        struct {
            explicit_atomic<mask_t>    _maybeMask; 
#if __LP64__
            uint16_t                   _flags; 
#endif
            uint16_t                   _occupied;
        };
        explicit_atomic<preopt_cache_t *> _originalPreoptCache; 
    };
}
```

通过《6-1》的探索，了解到其核心缓存结构为 `bucket_t` ，其对应的成员变量就是 `_bucketsAndMaybeMask` 了，根据其命名这里不仅有 `buckets` 应该还有 `MaybeMask` 

那么其具体的结构形式怎么探索呢？

这个数据成员变量是 `cahce_t` 缓存的，对缓存来说其核心方法就是存与取，那么以缓存的 `insert` 方法为切入点，应该能有助于了解当前的结构形式和实现形式

### insert 源码分析

#### 完整源码

```C++
void cache_t::insert(SEL sel, IMP imp, id receiver)
{
    runtimeLock.assertLocked();

    // Never cache before +initialize is done
    if (slowpath(!cls()->isInitialized())) {
        return;
    }

    if (isConstantOptimizedCache()) {
        _objc_fatal("cache_t::insert() called with a preoptimized cache for %s",
                    cls()->nameForLogging());
    }

#if DEBUG_TASK_THREADS
    return _collecting_in_critical();
#else
#if CONFIG_USE_CACHE_LOCK
    mutex_locker_t lock(cacheUpdateLock);
#endif

    ASSERT(sel != 0 && cls()->isInitialized());

    // Use the cache as-is if until we exceed our expected fill ratio.
    mask_t newOccupied = occupied() + 1;
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) {
        // Cache is read-only. Replace it.
        if (!capacity) capacity = INIT_CACHE_SIZE;
        reallocate(oldCapacity, capacity, /* freeOld */false);
    }
    else if (fastpath(newOccupied + CACHE_END_MARKER <= cache_fill_ratio(capacity))) {
        // Cache is less than 3/4 or 7/8 full. Use it as-is.
    }
#if CACHE_ALLOW_FULL_UTILIZATION
    else if (capacity <= FULL_UTILIZATION_CACHE_SIZE && newOccupied + CACHE_END_MARKER <= capacity) {
        // Allow 100% cache utilization for small buckets. Use it as-is.
    }
#endif
    else {
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);
    }

    bucket_t *b = buckets();
    mask_t m = capacity - 1;
    mask_t begin = cache_hash(sel, m);
    mask_t i = begin;

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot.
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(b, sel, imp, cls());
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));

    bad_cache(receiver, (SEL)sel);
#endif // !DEBUG_TASK_THREADS
}
```

通过入参 `SEL sel, IMP imp, id receiver` ，可以稍微窥探一下整个方法的用意，传入的 `SEL sel，IMP imp` 是缓存的数据，`id receiver` 是消息接受者即是谁调用的消息

#### 核心逻辑

前面的部分判断逻辑、加锁、断言以及 `DEBUG` 宏判断扫一眼后略过，核心的插入逻辑从这里开始

```C++
    // Use the cache as-is if until we exceed our expected fill ratio.
    mask_t newOccupied = occupied() + 1; 
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) {
        // Cache is read-only. Replace it.
        if (!capacity) capacity = INIT_CACHE_SIZE;
        reallocate(oldCapacity, capacity, /* freeOld */false);
    }
    else if (fastpath(newOccupied + CACHE_END_MARKER <= cache_fill_ratio(capacity))) {
        // Cache is less than 3/4 or 7/8 full. Use it as-is.
    }
#if CACHE_ALLOW_FULL_UTILIZATION
    else if (capacity <= FULL_UTILIZATION_CACHE_SIZE && newOccupied + CACHE_END_MARKER <= capacity) {
        // Allow 100% cache utilization for small buckets. Use it as-is.
    }
#endif
    else {
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);
    }

    bucket_t *b = buckets();
    mask_t m = capacity - 1; 
    mask_t begin = cache_hash(sel, m);
    mask_t i = begin;

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot.
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(b, sel, imp, cls());
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));

    bad_cache(receiver, (SEL)sel);
```

#### 首次缓存流程

首先进行 `occupied()` 加1操作

  `mask_t newOccupied = occupied() + 1; `  

查看 `occupied()` 方法

```C++
mask_t cache_t::occupied() const
{
    return _occupied;
}
```

只是进行了属性取值，那么其首次数据为 0，在方法调用后 `newOccupied` 就为 1 了

在其后的 if 条件判断中，存在多个判断，由于是首次进入，我们应该选择空缓存 `slowpath(isConstantEmptyCache())`  ，

```c++
bool cache_t::isConstantEmptyCache() const
{
    return
        occupied() == 0  &&
        buckets() == emptyBucketsForCapacity(capacity(), false);
}
```

同时 `slowpath()`也表明缓存初始化执行的比较少

##### 初始化逻辑

```C++
if (slowpath(isConstantEmptyCache())) {
    // Cache is read-only. Replace it.
    if (!capacity) capacity = INIT_CACHE_SIZE;
    reallocate(oldCapacity, capacity, /* freeOld */false);
}
```

###### capacity

首先是 `capacity` 赋值，

```C++
INIT_CACHE_SIZE      = (1 << INIT_CACHE_SIZE_LOG2),

//  ---- INIT_CACHE_SIZE_LOG2 ----
#if CACHE_END_MARKER || (__arm64__ && !__LP64__)
    // When we have a cache end marker it fills a bucket slot, so having a
    // initial cache size of 2 buckets would not be efficient when one of the
    // slots is always filled with the end marker. So start with a cache size
    // 4 buckets.
    INIT_CACHE_SIZE_LOG2 = 2,
#else
    // Allow an initial bucket size of 2 buckets, since a large number of
    // classes, especially metaclasses, have very few imps, and we support
    // the ability to fill 100% of the cache before resizing.
    INIT_CACHE_SIZE_LOG2 = 1,
#endif

// ----- CACHE_END_MARKER -----
#if __arm__  ||  __x86_64__  ||  __i386__

// objc_msgSend has few registers available.
// Cache scan increments and wraps at special end-marking bucket.
#define CACHE_END_MARKER 1

// Historical fill ratio of 75% (since the new objc runtime was introduced).
static inline mask_t cache_fill_ratio(mask_t capacity) {
    return capacity * 3 / 4;
}

#elif __arm64__ && !__LP64__

// objc_msgSend has lots of registers available.
// Cache scan decrements. No end marker needed.
#define CACHE_END_MARKER 0

// Historical fill ratio of 75% (since the new objc runtime was introduced).
static inline mask_t cache_fill_ratio(mask_t capacity) {
    return capacity * 3 / 4;
}

#elif __arm64__ && __LP64__

// objc_msgSend has lots of registers available.
// Cache scan decrements. No end marker needed.
#define CACHE_END_MARKER 0

// Allow 87.5% fill ratio in the fast path for all cache sizes.
// Increasing the cache fill ratio reduces the fragmentation and wasted space
// in imp-caches at the cost of potentially increasing the average lookup of
// a selector in imp-caches by increasing collision chains. Another potential
// change is that cache table resizes / resets happen at different moments.
static inline mask_t cache_fill_ratio(mask_t capacity) {
    return capacity * 7 / 8;
}

// Allow 100% cache utilization for smaller cache sizes. This has the same
// advantages and disadvantages as the fill ratio. A very large percentage
// of caches end up with very few entries and the worst case of collision
// chains in small tables is relatively small.
// NOTE: objc_msgSend properly handles a cache lookup with a full cache.
#define CACHE_ALLOW_FULL_UTILIZATION 1

#else
#error unknown architecture
#endif
```

此处 `INIT_CACHE_SIZE` 等于 `1<<2 = 4` ，`capacity` 赋值即为 4

###### reallocate()

接下来是 `reallocate(oldCapacity, capacity, /* freeOld */false);` 

```C++
ALWAYS_INLINE
void cache_t::reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld)
{
    bucket_t *oldBuckets = buckets();
    bucket_t *newBuckets = allocateBuckets(newCapacity);

    // Cache's old contents are not propagated. 
    // This is thought to save cache memory at the cost of extra cache fills.
    // fixme re-measure this

    ASSERT(newCapacity > 0);
    ASSERT((uintptr_t)(mask_t)(newCapacity-1) == newCapacity-1);

    setBucketsAndMask(newBuckets, newCapacity - 1);
    
    if (freeOld) {
        collect_free(oldBuckets, oldCapacity);
    }
}

```

此处进行了 `buckets` 的开辟，开辟一个 `bucket_t` 的容器，用来存放 `bucket_t` 这些缓存数据

根据传入的 `capacity` 进行了开辟 `allocateBuckets(newCapacity)` ，查看下 `allocateBuckets` 方法

```C++
#if CACHE_END_MARKER

bucket_t *cache_t::endMarker(struct bucket_t *b, uint32_t cap)
{
    return (bucket_t *)((uintptr_t)b + bytesForCapacity(cap)) - 1;
}

bucket_t *cache_t::allocateBuckets(mask_t newCapacity)
{
    // Allocate one extra bucket to mark the end of the list.
    // This can't overflow mask_t because newCapacity is a power of 2.
    bucket_t *newBuckets = (bucket_t *)calloc(bytesForCapacity(newCapacity), 1);

    bucket_t *end = endMarker(newBuckets, newCapacity);

#if __arm__
    // End marker's sel is 1 and imp points BEFORE the first bucket.
    // This saves an instruction in objc_msgSend.
    end->set<NotAtomic, Raw>(newBuckets, (SEL)(uintptr_t)1, (IMP)(newBuckets - 1), nil);
#else
    // End marker's sel is 1 and imp points to the first bucket.
    end->set<NotAtomic, Raw>(newBuckets, (SEL)(uintptr_t)1, (IMP)newBuckets, nil);
#endif
    
    if (PrintCaches) recordNewCache(newCapacity);

    return newBuckets;
}

#else

bucket_t *cache_t::allocateBuckets(mask_t newCapacity)
{
    if (PrintCaches) recordNewCache(newCapacity);

    return (bucket_t *)calloc(bytesForCapacity(newCapacity), 1);
}

#endif
```

此处的 `CACHE_END_MARKER` 与上面相同都为1，因此执行 `if` 代码块中代码

分析此处代码实现

- `bucket_t *newBuckets = (bucket_t *)calloc(bytesForCapacity(newCapacity), 1);`  开辟了内存空间

```C++
size_t cache_t::bytesForCapacity(uint32_t cap)
{
    return sizeof(bucket_t) * cap;
}
```

其中使用了 `bucket_t` 的大小，根据其成员 `SEL` 和 `IMP` ，大小应该为 16，`lldb`输出验证下

```shell
(lldb) p sizeof(bucket_t)
(unsigned long) $31 = 16
```

那就应该是开辟 `16 * capacity` 大小的内存空间

- 存在与宏名称 `CACHE_END_MARKER` 相同的-- 缓存结束标识 `End marker` ，注释中也提到了 `Allocate one extra bucket to mark the end of the list.` ，根据 `endMarker(newBuckets, newCapacity)` 方法的实现逻辑与入参，在这个 `buckets` 创建时，末尾存在一个 `End marker` 的 `bucket` ，其内部数据都进行了设定 `End marker's sel is 1 and imp points BEFORE the first bucket` 

**根据上述逻辑，当前 `capacity` 为 4 的新开辟的 `buckets` ，其能使用的空间其实只有3个** 

接下来跳过断言，来到 `setBucketsAndMask(newBuckets, newCapacity - 1)` ，点击跳转实现，发现存在宏判断以及其下的多个函数实现

```C++
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED

void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask) {
  //...
}
//... 省略其他宏 else 判断和内部实现

// ----  CACHE_MASK_STORAGE ---
#if defined(__arm64__) && __LP64__
#if TARGET_OS_OSX || TARGET_OS_SIMULATOR
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
#else
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_HIGH_16
#endif
#elif defined(__arm64__) && !__LP64__
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_LOW_4
#else
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_OUTLINED
#endif
```

由于当前非 `__arm64__` ，因此执行 `CACHE_MASK_STORAGE_OUTLINED` 策略，所以代码选择了下面这一块，其他的就省略不看

```C++
void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask)
{
    // objc_msgSend uses mask and buckets with no locks.
    // It is safe for objc_msgSend to see new buckets but old mask.
    // (It will get a cache miss but not overrun the buckets' bounds).
    // It is unsafe for objc_msgSend to see old buckets and new mask.
    // Therefore we write new buckets, wait a lot, then write new mask.
    // objc_msgSend reads mask first, then buckets.

#ifdef __arm__
    // ensure other threads see buckets contents before buckets pointer
    mega_barrier();

    _bucketsAndMaybeMask.store((uintptr_t)newBuckets, memory_order_relaxed);

    // ensure other threads see new buckets before new mask
    mega_barrier();

    _maybeMask.store(newMask, memory_order_relaxed);
    _occupied = 0;
#elif __x86_64__ || i386
    // ensure other threads see buckets contents before buckets pointer
    _bucketsAndMaybeMask.store((uintptr_t)newBuckets, memory_order_release);

    // ensure other threads see new buckets before new mask
    _maybeMask.store(newMask, memory_order_release);
    _occupied = 0;
#else
#error Don't know how to do setBucketsAndMask on this architecture.
#endif
}

mask_t cache_t::mask() const
{
    return _maybeMask.load(memory_order_relaxed);
}
```

分析其代码实现

- 在 `_bucketsAndMaybeMask.store((uintptr_t)newBuckets, memory_order_release);` 中把 `newBuckets` 强转为 `uintptr_t` 类型进行了存储 `store` -- 这里就是存储了首地址的指针的值，以便访问数据时偏移使用，使用指针偏移方式存储数据，减少了运行内存的使用 (同时读取时速度就不会太慢不需要加载很多数据)
- 同时 `_maybeMask.store(newMask, memory_order_release);` ，在 `_maybeMask` 中存储了剩余容量个数，同时此处入参 `newCapacity - 1` 进行了减1，也同时验证了 `allocateBuckets` 时确实存在创建 `End Marker` 导致容量减1
- 此处的两个注释也挺有意思 `ensure other threads see buckets contents before buckets pointer` 和 `ensure other threads see new buckets before new mask`   `buckets` 数据要在 `buckets` 指针前完成；`buckets` 要在 `mask` 之前完成。品味一下，有点意思，代码健壮么
- 最后进行了 `_occupied = 0` 的赋值，当前首次缓存，且缓存方法未存入，此时占用设置为0

最后就是释放 `oldBuckets` 主要是 GC 相关，首次缓存时传入 `freeOld` 参数为 `false` ，因此当前不执行此处的释放逻辑

 `reallocate()` 的主要逻辑大致分析完成后，回到主流程

##### 存储逻辑

之后的代码

```C++
    bucket_t *b = buckets();
    mask_t m = capacity - 1;
    mask_t begin = cache_hash(sel, m);
    mask_t i = begin;

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot.
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(b, sel, imp, cls());
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));

```

接下来就存储数据了

```C++
struct bucket_t *cache_t::buckets() const
{
    uintptr_t addr = _bucketsAndMaybeMask.load(memory_order_relaxed);
    return (bucket_t *)(addr & bucketsMask);
}
```

从 `_bucketsAndMaybeMask` 加载 `bucket_t` 指针数据

此处 `mask_t m = capacity - 1;` 由于存在 `End Marker` 因此需要减 1 才是剩余的可用容量

###### cache_hash()

之后 `mask_t begin = cache_hash(sel, m);`  通过 `cache_hash()` 方法来计算初始索引位置

```C++
// Class points to cache. SEL is key. Cache buckets store SEL+IMP.
// Caches are never built in the dyld shared cache.

static inline mask_t cache_hash(SEL sel, mask_t mask) 
{
    uintptr_t value = (uintptr_t)sel;
#if CONFIG_USE_PREOPT_CACHES  //当前环境为 0
    value ^= value >> 7;
#endif
    return (mask_t)(value & mask);
}

// ----  CONFIG_USE_PREOPT_CACHES ----
#if defined(__arm64__) && TARGET_OS_IOS && !TARGET_OS_SIMULATOR && !TARGET_OS_MACCATALYST
#define CONFIG_USE_PREOPT_CACHES 1
#else
#define CONFIG_USE_PREOPT_CACHES 0
#endif
```

此方法实现在 Mac 环境下只是进行了 `value & mask` 与操作 

###### do while

之后是 `do while` 循环，其注释表明了整体涵义

```C++
    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot.
```

要找到空的位置来插入

来看下具体实现逻辑

- `do` 语句中存在判断，可以直接执行

```C++
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(b, sel, imp, cls());
            return;
        }
```

第一个判断，判断当前位置的 `sel()` 为 0， 就表明未使用，则就直接 `set` 插入数据即可，同时进行了 `incrementOccupied()` ，这方法名就表明了`Occupied` 的自增，查看下实现

```C++
void cache_t::incrementOccupied() 
{
    _occupied++;
}
```

没错，这里如果执行的话就直接 `return` 掉了，不执行 `while` 循环语句了

```C++
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
```

第二个判断表明当前 `sel` 的缓存已经存在，插入过了，那么也直接 `return` 掉

- 接下来是循环判断条件 `while` 语句了

首先判断下什么时候能进入 `while` 判断，根据上面两个 `return` 的判断条件，在 `sel() != 0 && sel() != sel` 条件下，可理解为当前 `bucket_t` 存在缓存但是不是当前方法的缓存，这时候就执行 `while` 语句进行查找下一个位置，最终目的是找的空位置插入

查看下语句 `(i = cache_next(i, m)) != begin` 

`cache_next()` 方法实现

```C++
#if CACHE_END_MARKER
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return (i+1) & mask;
}
#elif __arm64__
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}
#else
#error unexpected configuration
#endif
```

这个也执行首个 if 语句中代码，就是 i (即是 begin ) 加 1 在与上 `mask` 不等于 `begin` 时就继续循环，直至寻找到空位置插入或 `return`

此时首次缓存流程就完成了，插入了首个缓存方法，接下来该插入更多个，当前的3个空位插满时怎么处理呢，接下来查看下非首次的缓存流程

#### 非首次缓存流程

后面的插入缓存数据的 `do while` 循环逻辑都与首次一致就不赘述了，主要区分是前面的 if 判断语句

```C++
    mask_t newOccupied = occupied() + 1; 
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
//... 省略首次
    else if (fastpath(newOccupied + CACHE_END_MARKER <= cache_fill_ratio(capacity))) {
        // Cache is less than 3/4 or 7/8 full. Use it as-is.
    }
#if CACHE_ALLOW_FULL_UTILIZATION
    else if (capacity <= FULL_UTILIZATION_CACHE_SIZE && newOccupied + CACHE_END_MARKER <= capacity) {
        // Allow 100% cache utilization for small buckets. Use it as-is.
    }
#endif
    else {
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);
    }
```

##### 不超出容器指定长度

###### else if 1

首个 `else if`  判断语句中核心代码 `newOccupied + CACHE_END_MARKER <= cache_fill_ratio(capacity)`

- `newOccupied ` 为 `occupied() + 1` ，即是当前已缓存的方法个数加1
- `CACHE_END_MARKER ` 当前环境下为 1 ，配置 `EndMarker` 末尾标识 `bucket`
- 查看 `cache_fill_ratio(capacity)` 方法实现

```C++
// 当前配置宏判断条件下
// Historical fill ratio of 75% (since the new objc runtime was introduced).
static inline mask_t cache_fill_ratio(mask_t capacity) {
    return capacity * 3 / 4;
}

// 完整代码查看 capacity 小节内 CACHE_END_MARKER 相关的宏判断代码
```

综合上述条目的分析，此时判断条件是 

`当前缓存的方法总数(包括当前要缓存的方法) + 1 <=  capacity * 3 / 4`

也符合注释语句 `// Cache is less than 3/4 or 7/8 full. Use it as-is.` 的涵义 (`7/8` 是 `__arm64__ && __LP64__` 环境下的配置，参考下面代码)

此情况下直接执行后续的插入逻辑即可

###### else if 2 (非当前环境，了解即可)

第二个 `else if ` 在 `CACHE_ALLOW_FULL_UTILIZATION` 宏判断下 

```C++
#elif __arm64__ && __LP64__

// objc_msgSend has lots of registers available.
// Cache scan decrements. No end marker needed.
#define CACHE_END_MARKER 0

// Allow 87.5% fill ratio in the fast path for all cache sizes.
// Increasing the cache fill ratio reduces the fragmentation and wasted space
// in imp-caches at the cost of potentially increasing the average lookup of
// a selector in imp-caches by increasing collision chains. Another potential
// change is that cache table resizes / resets happen at different moments.
static inline mask_t cache_fill_ratio(mask_t capacity) {
    return capacity * 7 / 8;
}

// Allow 100% cache utilization for smaller cache sizes. This has the same
// advantages and disadvantages as the fill ratio. A very large percentage
// of caches end up with very few entries and the worst case of collision
// chains in small tables is relatively small.
// NOTE: objc_msgSend properly handles a cache lookup with a full cache.
#define CACHE_ALLOW_FULL_UTILIZATION 1

#else
#error unknown architecture
#endif
```

此时 `CACHE_END_MARKER 0` 

分析判断语句 `capacity <= FULL_UTILIZATION_CACHE_SIZE && newOccupied + CACHE_END_MARKER <= capacity` 

查看 `FULL_UTILIZATION_CACHE_SIZE ` 

```C++
// 枚举值
FULL_UTILIZATION_CACHE_SIZE_LOG2 = 3,
FULL_UTILIZATION_CACHE_SIZE = (1 << FULL_UTILIZATION_CACHE_SIZE_LOG2),
```

根据此枚举值，`FULL_UTILIZATION_CACHE_SIZE` 值为 8 

此处判断语句就可以翻译为  `capacity <= 8 && newOccupied <= capacity`

此情况再加上 `else if 1` 中的逻辑在此判断之前执行，就对应了其注释 `// Allow 100% cache utilization for small buckets. Use it as-is.`  允许当前缓存方法添加后缓存容器充满

##### 最后 else

根据前述的判断条件，最后的 `else` 就是超出缓存容器指定的值后，此时按照正常逻辑就是要进行扩容来应对更多的缓存数据

###### 扩容时机

总结下扩容时机，根据当前的环境和逻辑，由于存在 `EndMarker` ，此时的扩容逻辑应该是

`当前缓存的方法总数(包括当前要缓存的方法) + 1 >  capacity * 3 / 4`

> 针对首次的 大小4的容器，当插入第3个方法缓存时，就进行扩容

###### 扩容源码

那么来看下代码

```C++
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);
```

###### 分析逻辑

- `capacity` 存在则进行 2倍扩容；不存在，则进行初始化，`INIT_CACHE_SIZE` 在首次缓存时我们分析了，当前取值为 4
- 还存在最大值 `MAX_CACHE_SIZE` = `1 << 16` (2 ^ 16)

```C++
    MAX_CACHE_SIZE_LOG2  = 16,
    MAX_CACHE_SIZE       = (1 << MAX_CACHE_SIZE_LOG2),
```

- 之后进行了 `reallocate(oldCapacity, capacity, true);`  这个函数在上面分析过了，方法中进行了新 `buckets` 的开辟 ，此时区别为最后一个参数 `freeOld` 传为 `true`

```C++
    if (freeOld) {
        collect_free(oldBuckets, oldCapacity);
    }
```

此时进行了垃圾回收，释放了全部 `oldBuckets` ，展示在结果上就是之前的缓存方法全部清空，此时新建的缓存容器中只有本次缓存的方法

> 这也解释了《6-2》中提出的问题2，扩容后大多数为空数据，且之前缓存数据没有了

总结此重点逻辑：**缓存扩容后，新创建缓存容器，旧缓存清空，当前只存在造成扩容的缓存方法**

###### 问题

那么问题就来了，明明是扩容，为什么要清空原数据，明明扩容后不清空的话也能放下这些数据

- 首先原来已开辟的内存不能更改，当前是伪逻辑更改，只是进行新增容器进行替换。这样如果保留原数据需要进行 数据平移，比较耗费性能，当缓存数据较多是就更加严重，这就严重降低效率
- 考虑 LRU(最近最少使用) 缓存机制，扩容时旧数据使用机会较低所以进行舍弃