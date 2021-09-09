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
#if __arm__  ||  __x86_64__  ||  __i386__ // __arm__ 指的是  32-bit ARM

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

#if __arm__   // 32-bit ARM
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

- 存在与宏名称 `CACHE_END_MARKER` 相同的-- 缓存结束标识 `End marker` ，注释中也提到了 `Allocate one extra bucket to mark the end of the list.` ，根据 `endMarker(newBuckets, newCapacity)` 方法的实现逻辑与入参，在这个 `buckets` 创建时，末尾存在一个 `End marker` 的 `bucket` ，其内部数据都进行了设定 `End marker's sel is 1 and imp points BEFORE the first bucket` ([拓展1] 进行了数据验证)

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

由于当前非 `__arm64__` ，因此执行 `CACHE_MASK_STORAGE_OUTLINED` 策略，所以代码选择了下面这一块，其他的就省略不看(该策略宏定义在 《6-1》拓展3中进行粗略分析)

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
- 同时 `_maybeMask.store(newMask, memory_order_release);` ，在 `_maybeMask` 中存储了容量个数(除了 `End Marker` 外的)，同时此处入参 `newCapacity - 1` 进行了减1，也同时验证了 `allocateBuckets` 时确实存在创建 `End Marker` 导致容量减1
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

> arm64架构下(真机走这里)，其逻辑 `i ? i-1 : mask` 
>
> - i 在 0 即首位时，直接返回 mask ，此时 mask 为 `capacity - 1` ，直接放置到容器末尾，注意此处是 `!CACHE_END_MARKER` 判断条件下，即此架构下是不存在末尾的缓存结束标识的，所以此处不存在与 `End marker` 冲突的情况
> - i 非 0时，直接 i - 1前移

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

#### bad_cache

最后的语句 `bad_cache(receiver, (SEL)sel);` 

内存出现紊乱时，执行此句代码

```C++
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

按照正常逻辑在 `do while` 循环中会执行到 `return` 出去了

在 `while` 循环条件下，全都不符合 `do` 中的判断条件，无法跳出的情况，意味着这片内存有问题，可能是开辟少了等等各种异常

查看下 `bad_cache()` 的实现

```C++
void cache_t::bad_cache(id receiver, SEL sel)
{
    // Log in separate steps in case the logging itself causes a crash.
    _objc_inform_now_and_on_crash
        ("Method cache corrupted. This may be a message to an "
         "invalid object, or a memory error somewhere else.");
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED
    bucket_t *b = buckets();
    _objc_inform_now_and_on_crash
        ("%s %p, SEL %p, isa %p, cache %p, buckets %p, "
         "mask 0x%x, occupied 0x%x", 
         receiver ? "receiver" : "unused", receiver, 
         sel, cls(), this, b,
         _maybeMask.load(memory_order_relaxed),
         _occupied);
    _objc_inform_now_and_on_crash
        ("%s %zu bytes, buckets %zu bytes", 
         receiver ? "receiver" : "unused", malloc_size(receiver), 
         malloc_size(b));
#elif (CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16 || \
       CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS || \
       CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4)
    uintptr_t maskAndBuckets = _bucketsAndMaybeMask.load(memory_order_relaxed);
    _objc_inform_now_and_on_crash
        ("%s %p, SEL %p, isa %p, cache %p, buckets and mask 0x%lx, "
         "occupied 0x%x",
         receiver ? "receiver" : "unused", receiver,
         sel, cls(), this, maskAndBuckets, _occupied);
    _objc_inform_now_and_on_crash
        ("%s %zu bytes, buckets %zu bytes",
         receiver ? "receiver" : "unused", malloc_size(receiver),
         malloc_size(buckets()));
#else
#error Unknown cache mask storage type.
#endif
    _objc_inform_now_and_on_crash
        ("selector '%s'", sel_getName(sel));
    _objc_inform_now_and_on_crash
        ("isa '%s'", cls()->nameForLogging());
    _objc_fatal
        ("Method cache corrupted. This may be a message to an "
         "invalid object, or a memory error somewhere else.");
}
```

代码中都是 `crash` 情况下的报错信息，看最后的语句 `Method cache corrupted. This may be a message to an invalid object, or a memory error somewhere else.`  也提到了可能是无效对象或者是某处出现内存紊乱了

> 拓展问题：为什么系统管理内存还会出现内存紊乱呢？
>
> 上层代码中出现多线程等操作或者系统出现异常都有可能会造成内存混乱，在底层这些源码中看到很多加锁操作也都是为了保证数据的正常
>
> 怎么避免呢？这是有关操作系统稳定性的问题，一般系统进行处理，软件开发者无需处理



## 拓展

### 拓展1 `LLDB` 调试调用方法首次开辟 7 问题

有大佬发现此问题：当使用 `LLDB` 调试调用方法时，首次就直接开辟 7 个空间 [参考链接][https://juejin.cn/post/6976941059163029540/#heading-18]

#### 验证问题

首先来验证下问题，有助于了解此问题

OC 代码

```objc
//LGPerson
@interface LGPerson : NSObject
- (void)saySomething;
@end
@implementation LGPerson
- (void)saySomething{
    NSLog(@"%s",__func__);
}
@end
  
// main函数
 LGPerson *p  = [LGPerson alloc];
//此句代码后添加断点
```

运行捕获断点后，`LLDB` 验证

```shell
(lldb) x/4gx LGPerson.class
0x100008600: 0x0000000100008628 0x000000010036a140
0x100008610: 0x0000000100362380 0x0000804800000000
(lldb) p (cache_t *)0x100008610
(cache_t *) $1 = 0x0000000100008610
(lldb) p *$1
(cache_t) $2 = {
  _bucketsAndMaybeMask = {
    std::__1::atomic<unsigned long> = {
      Value = 4298515328
    }
  }
   = {
     = {
      _maybeMask = {
        std::__1::atomic<unsigned int> = {
          Value = 0
        }
      }
      _flags = 32840
      _occupied = 0
    }
    _originalPreoptCache = {
      std::__1::atomic<preopt_cache_t *> = {
        Value = 0x0000804800000000
      }
    }
  }
}
```

未调用方法前 `_maybeMask` 中存储值为 0， 无方法缓存

使用 `LLDB` 调用 `saySomething` 方法

```shell
(lldb) p [p saySomething]
-[LGPerson saySomething]
(lldb) x/4gx LGPerson.class
0x100008600: 0x0000000100008628 0x000000010036a140
0x100008610: 0x00000001012549a0 0x0001804800000007
(lldb) p (cache_t *)0x100008610
(cache_t *) $4 = 0x0000000100008610
(lldb) p *$4
(cache_t) $5 = {
  _bucketsAndMaybeMask = {
    std::__1::atomic<unsigned long> = {
      Value = 4314188192
    }
  }
   = {
     = {
      _maybeMask = {
        std::__1::atomic<unsigned int> = {
          Value = 7
        }
      }
      _flags = 32840
      _occupied = 1
    }
    _originalPreoptCache = {
      std::__1::atomic<preopt_cache_t *> = {
        Value = 0x0001804800000007
      }
    }
  }
}
```

此时查看到 `_maybeMask` 中值变为 7 了，而且此时的 `_occupied = 1` 

按照分析的代码逻辑，此时进行一个方法缓存应该只开辟 4 空间，`_maybeMask` 为 3 才对啊，这就是问题的现象了

##### 对照组：正常流程

在查看下正常代码调用的情况作为对照组

OC 代码

```objc
//main 函数
    LGPerson *p  = [LGPerson alloc];
    [p saySomething];
//方法调用后添加断点
```

断点捕获后，进行 `LLDB` 输出

```shell
(lldb) x/4gx LGPerson.class
0x100008608: 0x0000000100008630 0x000000010036a140
0x100008618: 0x0000000101b38210 0x0001804800000003
(lldb) p (cache_t *)0x100008618
(cache_t *) $1 = 0x0000000100008618
(lldb) p *$1
(cache_t) $2 = {
  _bucketsAndMaybeMask = {
    std::__1::atomic<unsigned long> = {
      Value = 4323508752
    }
  }
   = {
     = {
      _maybeMask = {
        std::__1::atomic<unsigned int> = {
          Value = 3
        }
      }
      _flags = 32840
      _occupied = 1
    }
    _originalPreoptCache = {
      std::__1::atomic<preopt_cache_t *> = {
        Value = 0x0001804800000003
      }
    }
  }
}
(lldb) 
```

没错，`_maybeMask` 是为 3 ，是除了 `End marker` 之外的容量

#### 探索问题

那么接下来探索下这个问题

##### 思考1  存在其他方法

首先冒出的念头是，既然开辟了7个，应该是有其他方法进行了缓存，可以输出下全部空间的方法名不就知道是哪些方法造成的了么，那就试下

```shell
(lldb) p [p saySomething]
2021-09-07 20:28:03.518760+0800 KCObjcBuild[39194:6918152] -[LGPerson saySomething]
(lldb) x/4gx LGPerson.class
0x100008618: 0x0000000100008640 0x000000010036a140
0x100008628: 0x0000000101339a10 0x0001804800000007
(lldb) p (cache_t *)0x100008628
(cache_t *) $4 = 0x0000000100008628
(lldb) p *$4
(cache_t) $5 = {
  _bucketsAndMaybeMask = {
    std::__1::atomic<unsigned long> = {
      Value = 4315126288
    }
  }
   = {
     = {
      _maybeMask = {
        std::__1::atomic<unsigned int> = {
          Value = 7
        }
      }
      _flags = 32840
      _occupied = 1
    }
    _originalPreoptCache = {
      std::__1::atomic<preopt_cache_t *> = {
        Value = 0x0001804800000007
      }
    }
  }
}
(lldb) p $5.buckets
(bucket_t *) $6 = 0x0000000101339a10
  Fix-it applied, fixed expression was: 
    $5.buckets()
(lldb) p $5.buckets()
(bucket_t *) $7 = 0x0000000101339a10
(lldb) p $7[0]
(bucket_t) $8 = {
  _sel = {
    std::__1::atomic<objc_selector *> = (null) {
      Value = (null)
    }
  }
  _imp = {
    std::__1::atomic<unsigned long> = {
      Value = 0
    }
  }
}
(lldb) p $7[1]
(bucket_t) $9 = {
  _sel = {
    std::__1::atomic<objc_selector *> = (null) {
      Value = (null)
    }
  }
  _imp = {
    std::__1::atomic<unsigned long> = {
      Value = 0
    }
  }
}
(lldb) p $7[2]
(bucket_t) $10 = {
  _sel = {
    std::__1::atomic<objc_selector *> = "" {
      Value = ""
    }
  }
  _imp = {
    std::__1::atomic<unsigned long> = {
      Value = 49048
    }
  }
}
(lldb) p $10.sel()
(SEL) $11 = "saySomething"
(lldb) p $7[3]
# ... 省略 3-6 都与 7[1] 相同为空的
(lldb) p $7[7]
(bucket_t) $16 = {
  _sel = {
    std::__1::atomic<objc_selector *> = (null) {
      Value = (null)
    }
  }
  _imp = {
    std::__1::atomic<unsigned long> = {
      Value = 4315126288
    }
  }
}
(lldb) p $16.sel()
(SEL) $17 = <no value available>
(lldb) 
```

发现没有多余方法，怎么回事，回顾了一下扩容逻辑，orz，想起来了，扩容时候会清除上次的容器数据的，所以扩容后只有新增的方法。

 此处也正好解释了 `_occupied = 1`  的原因，因为当前就只有一个方法缓存，前面的都在扩容时候清除了

##### 思考2 输出扩容前方法缓存

那么接下来的目标是找到扩容前的方法缓存数据，输出一下，看是多了哪些方法缓存

- 首先考虑 `LLDB` 能否输出？ 找不到扩容前的时机，这个只调用一个方法就扩容了，没啥思路
- 那么就只能考虑在源码中添加 `printf` 来输出了。那么在哪里添加呢？所有方法进行缓存时都要走的函数，那么 `insert` 函数就很合适，赶紧试试

###### 方式1 输出所有 insert 的缓存方法

在 `insert` 函数新增输出语句

```C++
void cache_t::insert(SEL sel, IMP imp, id receiver)
{
    printf("=== %s -- %p -- %p \n", (char *)sel, imp, receiver);
//...省略其他
}
```

此时再重新运行，输出了一大堆方法，说明输出语句有效

```shell
=== dealloc -- 0x100495a9e -- 0x101a11330 
=== dealloc -- 0x100350970 -- 0x101a11330 
# ... 省略一大堆
=== alloc -- 0x1003508c0 -- 0x7fff885e1218 
=== initWithObjects: -- 0x7fff205d478e -- 0x7fff803dc548 
```

不过还没有调用 `saySomething` 方法，所以这些与当前问题无关，都是系统的相关方法，不用看。此时再次使用 `LLDB` 调用 `saySomething` 方法，查看输出

```shell
(lldb) p [p saySomething]
=== respondsToSelector: -- 0x10034fc90 -- 0x100666210 
=== class -- 0x10034f8f0 -- 0x100666210 
=== saySomething -- 0x100003e30 -- 0x100666210 
=== dealloc -- 0x7fff2028f0d3 -- 0x1006646e0 
=== dealloc -- 0x7fff2028f0d3 -- 0x10123e3a0 
=== dealloc -- 0x7fff20292398 -- 0x10073e530 
=== dealloc -- 0x100350970 -- 0x10073e530 
=== dealloc -- 0x1004639f8 -- 0x10123e440 
=== dealloc -- 0x7fff2028f0d3 -- 0x1012114e0 
=== dealloc -- 0x7fff2028f0d3 -- 0x101208fe0 
=== _fastCStringContents: -- 0x7fff20633141 -- 0x100004008 
=== dealloc -- 0x7fff2028f0d3 -- 0x10123e3c0 
 -[LGPerson saySomething]
=== retain -- 0x100350660 -- 0x101436cc0 
=== release -- 0x1003507f0 -- 0x101436cc0 
=== dealloc -- 0x7fff2028f0d3 -- 0x101436d30 
=== dealloc -- 0x100495a9e -- 0x101436cc0 
```

看到输出了这些，` -[LGPerson saySomething]` 方法调用完成后的输出不用看。根据 `saySomething` 的输出顺序，其上新增的方法函数可能是如下这两个

```shell
=== respondsToSelector: -- 0x10034fc90 -- 0x100666210 
=== class -- 0x10034f8f0 -- 0x100666210 
=== saySomething -- 0x100003e30 -- 0x100666210 
```

这只是猜测，怎么确定呢，输出的第三个参数是 `receiver` 消息接收者，这三项输出的 `receiver` 输出相同，应该是 `LGPerson` ，输出验证下

```shell
(lldb) po 0x100666210
=== debugDescription -- 0x100350520 -- 0x100666210 
=== description -- 0x7fff20636d7e -- 0x100666210 
=== isEqual: -- 0x7fff205d379c -- 0x101436db0 
<LGPerson: 0x100666210>

=== dataUsingEncoding:allowLossyConversion: -- 0x7fff213c064c -- 0x101436db0 
=== length -- 0x7fff205d3364 -- 0x101436db0 
=== allocWithZone: -- 0x1003508e0 -- 0x7fff885dde38 
=== initWithLength: -- 0x7fff213c087f -- 0x101436410 
=== setLength: -- 0x7fff213c09f4 -- 0x101436410 
=== mutableBytes -- 0x7fff213c0b3d -- 0x101436410 
=== setLength: -- 0x7fff213c09f4 -- 0x101436410 
=== length -- 0x7fff213c106f -- 0x101436410 
=== bytes -- 0x7fff213a4efe -- 0x101436410 
=== dealloc -- 0x7fff213a4f7b -- 0x101436410 
=== _freeBytes -- 0x7fff213a4fdc -- 0x101436410 
=== dealloc -- 0x100350970 -- 0x101436410 
=== release -- 0x7fff205ec7b3 -- 0x101436db0 
(lldb) 
```

确实是 `<LGPerson: 0x100666210>`  没错，在 `LGPerson` 输出之前还存在三行输出，这些输出不就意味着 `LLDB`  `po` 时产生的方法调用缓存，这就证明了 `LLDB` 在调用函数时必然会插入一些方法

> 稍微分析理解下这些方法 [延伸]
>
> - description  就是要输出的 `LGPerson` 的描述
> - isEqual 就是判断 `LGPerson` 的地址与当前打印地址是否相同

同样的道理在 `(lldb) p [p saySomething]` 使用 `LLDB` 调用方法的时候产生多的方法缓存就可以理解了

###### 方式2 在 saySomething 缓存造成的扩容前输出当前所有的缓存方法

`107/99(ASCII)讲师` 提出的另一种更方便的方式，在 `saySomething` 缓存造成的扩容前输出当前所有的缓存方法，来看下代码

```C++
// void cache_t::insert(SEL sel, IMP imp, id receiver) 方法中
    // Use the cache as-is if until we exceed our expected fill ratio.
    mask_t newOccupied = occupied() + 1;
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    
    if (sel == @selector(saySomething)) {
        bucket_t *lgd_b = buckets();
        for (unsigned i = 0; i < oldCapacity; i++) {
            SEL lgd_sel = lgd_b[i].sel();
            IMP lgd_imp = lgd_b[i].imp(lgd_b, nil);
            printf("%p - %p - %p \n", lgd_sel, lgd_imp, &lgd_b[i]);
        }
    }
```

**分析代码**

- 依然是在 `cache_t::insert` 中添加
- 在指定的 `saySomething` 方法插入时再输出，其他的都不用看
- 这次使用到 `oldCapacity` 所以放在其后，也可以直接使用 `capacity()` ，这时获取的容量是未扩容前的容量
- 目的是输出 `sel` 和  `imp` ，所以要获取到 `bucket_t *` ，然后直接使用内存平移方式取值 `b[i]` ； `sel` 的获取直接使用 `b[i].sel()` ; `imp` 的获取则使用 `b[i].imp(b, nil)`   ( 《cache_t的底层分析》中详解)

> 注意细节：`b[i].imp(b, nil)`  的第二个参数 ` Class cls` 传 nil，是为了实现进行异或编码时保留原地址直接输出，异或nil就相当于不进行编码，这样就不用在输出时再异或一遍 `cls()` 了(《6-2、重写源码探索方法》的拓展中进行了分析)

**运行输出**

运行在 `p` 对象初始化后断点处进行 `LLDB` 方法调用

```shell
(lldb) p [p saySomething]
0x7fff7c4cc00c - 0x347e28 - 0x10131e140 
0x7fff7c4cbf71 - 0x347a48 - 0x10131e150 
0x0 - 0x0 - 0x10131e160 
0x1 - 0x10131e140 - 0x10131e170 
 -[LGPerson saySomething]
```

输出了当前容器内的数据，分析查看下这些数据

- 查看前两个数据的 `SEL` 

```shell
(lldb) p (SEL)0x7fff7c4cc00c
(SEL) $2 = "respondsToSelector:"
(lldb) po (SEL)0x7fff7c4cc00c
"respondsToSelector:"
(lldb) p class_getMethodImplementation([LGPerson class], $2)
(IMP) $4 = 0x000000010034fc90 (libobjc.A.dylib`-[NSObject respondsToSelector:] at NSObject.mm:2299)

(lldb) p (SEL)0x7fff7c4cbf71
(SEL) $5 = "class"
(lldb) p class_getMethodImplementation([LGPerson class], $5)
(IMP) $6 = 0x000000010034f8f0 (libobjc.A.dylib`-[NSObject class] at NSObject.mm:2243)
```

验证了当前 `LLDB` 调用函数时调用了 `respondsToSelector:` 和 `class`  这两个 `NSObject` 的方法，进行了方法缓存

- `0x0 - 0x0 - 0x10131e160` 第三个数据为空，当前未插入数据，所以此时是还未进行扩容的状态
- `0x1 - 0x10131e140 - 0x10131e170 ` 最后一个数据就是 `End marker` 了，回顾一下正文中 `reallocate()` 中的分析，再次查看 `allocateBuckets` 方法，在非 `__arm__` 情况下 `End marker's sel is 1 and imp points to the first bucket` ，`SEL` 为 1，同时 `imp` 指向首个 `bucket` 地址为 `0x10131e140` ，完全符合

```C++
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
```

> `__arm__` 这里的这个 arm 根据之前的判断，是与 `__arm64__` 不同的架构类型，指的是 ` 32-bit ARM` 

##### 总结

问题解决了，总结一下：

由于 `LLDB`  调用方法时，会进行 `respondsToSelector` 和 `class` 两个方法的缓存，造成在 `saySomething` 方法缓存插入时，由于 `End marker` 的存在，此时就超出了首次开辟容量的 `3/4` ，从而引起了扩容逻辑，最终形成了输出的结果，当前开辟容量为 7 ，同时缓存数据个数为 1

#### LLVM 源码验证

有大佬在 `LLVM` 源码中找到了对应的解释，来看下

```C++
// Path：llvm-project/lldb/source/Plugins/LanguageRuntime/ObjC/AppleObjCRuntime/AppleObjCRuntimeV2.cpp
llvm::Expected<std::unique_ptr<UtilityFunction>>
AppleObjCRuntimeV2::CreateObjectChecker(std::string name,
                                        ExecutionContext &exe_ctx) {
  char check_function_code[2048];

  int len = 0;
  if (m_has_object_getClass) {
    len = ::snprintf(check_function_code, sizeof(check_function_code), R"(
                     extern "C" void *gdb_object_getClass(void *);
                     extern "C" int printf(const char *format, ...);
                     extern "C" void
                     %s(void *$__lldb_arg_obj, void *$__lldb_arg_selector) {
                       if ($__lldb_arg_obj == (void *)0)
                         return; // nil is ok
                       if (!gdb_object_getClass($__lldb_arg_obj)) {
                         *((volatile int *)0) = 'ocgc';
                       } else if ($__lldb_arg_selector != (void *)0) {
                         signed char $responds = (signed char)
                             [(id)$__lldb_arg_obj respondsToSelector:
                                 (void *) $__lldb_arg_selector];
                         if ($responds == (signed char) 0)
                           *((volatile int *)0) = 'ocgc';
                       }
                     })",
                     name.c_str());
  } else {
    len = ::snprintf(check_function_code, sizeof(check_function_code), R"(
                     extern "C" void *gdb_class_getClass(void *);
                     extern "C" int printf(const char *format, ...);
                     extern "C" void
                     %s(void *$__lldb_arg_obj, void *$__lldb_arg_selector) {
                       if ($__lldb_arg_obj == (void *)0)
                         return; // nil is ok
                       void **$isa_ptr = (void **)$__lldb_arg_obj;
                       if (*$isa_ptr == (void *)0 ||
                           !gdb_class_getClass(*$isa_ptr))
                         *((volatile int *)0) = 'ocgc';
                       else if ($__lldb_arg_selector != (void *)0) {
                         signed char $responds = (signed char)
                             [(id)$__lldb_arg_obj respondsToSelector:
                                 (void *) $__lldb_arg_selector];
                         if ($responds == (signed char) 0)
                           *((volatile int *)0) = 'ocgc';
                       }
                     })",
                     name.c_str());
  }

  assert(len < (int)sizeof(check_function_code));
  UNUSED_IF_ASSERT_DISABLED(len);

  return GetTargetRef().CreateUtilityFunction(check_function_code, name,
                                              eLanguageTypeC, exe_ctx);
}
```

验证的代码是红色代码块中的这部分

```C++
if (!gdb_object_getClass($__lldb_arg_obj)) {
   *((volatile int *)0) = 'ocgc';
 } else if ($__lldb_arg_selector != (void *)0) {
   signed char $responds = (signed char)
       [(id)$__lldb_arg_obj respondsToSelector:
           (void *) $__lldb_arg_selector];
   if ($responds == (signed char) 0)
     *((volatile int *)0) = 'ocgc';
 }
```

##### 分析

- `gdb_object_getClass()` 对应调用 `class` ，获取当前对象 `$__lldb_arg_obj` 的 Class
- 之后`else if` 代码中的 `[(id)$__lldb_arg_obj respondsToSelector:(void *) $__lldb_arg_selector];`  调用了 ` respondsToSelector:` 方法来验证当前对象是否能响应当前方法(和 OC 写代理方法调用时判断一样)来查看方法实现

#### 延伸 

##### 描述

方式1 中 `po 类地址` 时的 方法

```shell
(lldb) po 0x100666210
=== debugDescription -- 0x100350520 -- 0x100666210 
=== description -- 0x7fff20636d7e -- 0x100666210 
=== isEqual: -- 0x7fff205d379c -- 0x101436db0 
<LGPerson: 0x100666210>
```

在方式2 使用 `class_getMethodImplementation` 的输出方法的详细信息后，这里也可以进行输出下

##### 输出

`printf` 语句中新增输出 `SEL` 指针地址

```C++
    printf("=== %s -- %p -- %p -- %p \n", (char *)sel, sel, imp, receiver);
```

按方式1流程运行，至 `po` 类地址时

```shell
(lldb) po 0x101836040
=== debugDescription -- 0x7fff7c4cc041 -- 0x100350500 -- 0x101836040 
=== description -- 0x7fff7c4cc035 -- 0x7fff20636d7e -- 0x101836040 
=== isEqual: -- 0x7fff7c4cbf68 -- 0x7fff205d379c -- 0x101d17dc0 
# ... 省略其他 
<LGPerson: 0x101836040>
```

使用 `class_getMethodImplementation` 进行输出

```shell
(lldb) p (SEL)0x7fff7c4cc041
(SEL) $3 = "debugDescription"
(lldb) p class_getMethodImplementation([LGPerson class], $3)
=== respondsToSelector: -- 0x7fff7c4cc00c -- 0x10034fc20 -- 0x1000082b8 
=== class -- 0x7fff7c4cbf71 -- 0x10034f8b0 -- 0x1000082b8 
(IMP) $4 = 0x0000000100350500 (libobjc.A.dylib`-[NSObject debugDescription] at NSObject.mm:2469)

(lldb) p class_getMethodImplementation([LGPerson class], (SEL)0x7fff7c4cc035)
(IMP) $5 = 0x00007fff20636d7e (CoreFoundation`-[NSObject(NSObject) description])
```

至于 `isEqual:` 由于其类型没输出出来，所以考虑使用 `[NSObject class]` 作为首个参数进行输出

```shell
(lldb) po 0x101d17dc0
=== respondsToSelector: -- 0x7fff7c4cc00c -- 0x10034fc70 -- 0x101d17dc0 
4325473728

=== class -- 0x7fff7c4cbf71 -- 0x10034f8d0 -- 0x101d17dc0 
=== debugDescription -- 0x7fff7c4cc041 -- 0x100350500 -- 0x101d17dc0 

(lldb) p class_getMethodImplementation([NSObject class], (SEL)0x7fff7c4cbf68)
=== respondsToSelector: -- 0x7fff7c4cc00c -- 0x10034fc20 -- 0x10036a140 
=== class -- 0x7fff7c4cbf71 -- 0x10034f8b0 -- 0x10036a140 
=== isEqual: -- 0x7fff7c4cbf68 -- 0x10034fe70 -- 0x0 
(IMP) $17 = 0x000000010034fe70 (libobjc.A.dylib`-[NSObject isEqual:] at NSObject.mm:2331)
```

查看到都是 `NSObject` 的方法，`debugDescription` 和 `isEqual:`  在 `libobjc.A.dylib` 中，可以在 Objc 源码查看， `description` 在 `CoreFoundation` 库

##### 源码查看

在 `Objc` 源码的  `NSObject.mm` 中找到对应方法

```C++
+ (BOOL)isEqual:(id)obj {
    return obj == (id)self;
}

- (BOOL)isEqual:(id)obj {
    return obj == self;
}

// Replaced by CF (returns an NSString)
+ (NSString *)description {
    return nil;
}

// Replaced by CF (returns an NSString)
- (NSString *)description {
    return nil;
}

+ (NSString *)debugDescription {
    return [self description];
}

- (NSString *)debugDescription {
    return [self description];
}
```

`description` 由 `debugDescription` 调起，所以他们的顺序是 `debugDescription` 在前

在 `Core Foudation` 源码的 [1153.18版本-CFError.c][https://opensource.apple.com/source/CF/CF-1153.18/CFError.c.auto.html] 中找到相关信息

> gitHub 下载旧版本源码，使用 VSCode 打开搜索  description (太卷了orz)

```C++
/* The "debug" description, used by CFCopyDescription and -[NSObject description].
*/
static void userInfoKeyValueShow(const void *key, const void *value, void *context) {
    CFStringRef desc;
    if (CFEqual(key, kCFErrorUnderlyingErrorKey) && (desc = CFErrorCopyDescription((CFErrorRef)value))) {	// We check desc, see <rdar://problem/8415727>
	CFStringAppendFormat((CFMutableStringRef)context, NULL, CFSTR("%@=%p \"%@\", "), key, value, desc); 
	CFRelease(desc);
    } else {
	CFStringAppendFormat((CFMutableStringRef)context, NULL, CFSTR("%@=%@, "), key, value); 
    }
}


/* This is the full debug description. Shows the description (possibly localized), plus the domain, code, and userInfo explicitly. If there is a debug description, shows that as well. 
*/
static CFStringRef __CFErrorCopyDescription(CFTypeRef cf) {
    return _CFErrorCreateDebugDescription((CFErrorRef)cf);
}


CFStringRef _CFErrorCreateDebugDescription(CFErrorRef err) {
    CFStringRef desc = CFErrorCopyDescription(err);
    CFStringRef debugDesc = _CFErrorCopyUserInfoKey(err, kCFErrorDebugDescriptionKey);
    CFDictionaryRef userInfo = _CFErrorGetUserInfo(err);
    CFMutableStringRef result = CFStringCreateMutable(kCFAllocatorSystemDefault, 0);
    CFStringAppendFormat(result, NULL, CFSTR("Error Domain=%@ Code=%ld"), CFErrorGetDomain(err), (long)CFErrorGetCode(err));
    CFStringAppendFormat(result, NULL, CFSTR(" \"%@\""), desc);
    if (debugDesc && CFStringGetLength(debugDesc) > 0) CFStringAppendFormat(result, NULL, CFSTR(" (%@)"), debugDesc);
    if (userInfo && CFDictionaryGetCount(userInfo)) {
        CFStringAppendFormat(result, NULL, CFSTR(" UserInfo=%p {"), userInfo);
	CFDictionaryApplyFunction(userInfo, userInfoKeyValueShow, (void *)result);
	CFIndex commaLength = (CFStringHasSuffix(result, CFSTR(", "))) ? 2 : 0;
	CFStringReplace(result, CFRangeMake(CFStringGetLength(result)-commaLength, commaLength), CFSTR("}"));
    }
    if (debugDesc) CFRelease(debugDesc);
    if (desc) CFRelease(desc);
    return result;
}

```

就探索到这里了



### 拓展2 C中输出格式

`%p` 输出 `Pointer` 指针