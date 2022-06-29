## objc_msgSend 原理汇编分析之 CacheLookup

### 引文

在之前的 `《7-3、objc_msgSend源码查探》`  中，分析了前面一部分的汇编源码，在上次查探中，分析到通过消息接收者获取 `isa` ，通过 `isa` 获取 `class` 之后，最后对 `class` 内部的 `cache` 进行操作，来到了 `CacheLookup` 方法

```assembly
LGetIsaDone:
	// calls imp or objc_msgSend_uncached
	CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached
```

现在就对之后的流程继续分析，当前进行的分析是在 `objc-msg-arm64.s` 文件中对 `arm64` 真机环境下的流程分析

### CacheLookup

#### 汇编源码

在搜索中，找到 `.macro` 宏，首先先查看下整体代码

```assembly
/********************************************************************
 *
 * CacheLookup NORMAL|GETIMP|LOOKUP <function> MissLabelDynamic MissLabelConstant
 *
 * MissLabelConstant is only used for the GETIMP variant.
 *
 * Locate the implementation for a selector in a class method cache.
 *
 * When this is used in a function that doesn't hold the runtime lock,
 * this represents the critical section that may access dead memory.
 * If the kernel causes one of these functions to go down the recovery
 * path, we pretend the lookup failed by jumping the JumpMiss branch.
 *
 * Takes:
 *	 x1 = selector
 *	 x16 = class to be searched
 *
 * Kills:
 * 	 x9,x10,x11,x12,x13,x15,x17
 *
 * Untouched:
 * 	 x14
 *
 * On exit: (found) calls or returns IMP
 *                  with x16 = class, x17 = IMP
 *                  In LOOKUP mode, the two low bits are set to 0x3
 *                  if we hit a constant cache (used in objc_trace)
 *          (not found) jumps to LCacheMiss
 *                  with x15 = class
 *                  For constant caches in LOOKUP mode, the low bit
 *                  of x16 is set to 0x1 to indicate we had to fallback.
 *          In addition, when LCacheMiss is __objc_msgSend_uncached or
 *          __objc_msgLookup_uncached, 0x2 will be set in x16
 *          to remember we took the slowpath.
 *          So the two low bits of x16 on exit mean:
 *            0: dynamic hit
 *            1: fallback to the parent class, when there is a preoptimized cache
 *            2: slowpath
 *            3: preoptimized cache hit
 *
 ********************************************************************/

#define NORMAL 0
#define GETIMP 1
#define LOOKUP 2

// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
.macro CacheHit
.if $0 == NORMAL
	TailCallCachedImp x17, x10, x1, x16	// authenticate and call imp
.elseif $0 == GETIMP
	mov	p0, p17
	cbz	p0, 9f			// don't ptrauth a nil imp
	AuthAndResignAsIMP x0, x10, x1, x16	// authenticate imp and re-sign as IMP
9:	ret				// return IMP
.elseif $0 == LOOKUP
	// No nil check for ptrauth: the caller would crash anyway when they
	// jump to a nil IMP. We don't care if that jump also fails ptrauth.
	AuthAndResignAsIMP x17, x10, x1, x16	// authenticate imp and re-sign as IMP
	cmp	x16, x15
	cinc	x16, x16, ne			// x16 += 1 when x15 != x16 (for instrumentation ; fallback to the parent class)
	ret				// return imp via x17
.else
.abort oops
.endif
.endmacro

.macro CacheLookup Mode, Function, MissLabelDynamic, MissLabelConstant
	//
	// Restart protocol:
	//
	//   As soon as we're past the LLookupStart\Function label we may have
	//   loaded an invalid cache pointer or mask.
	//
	//   When task_restartable_ranges_synchronize() is called,
	//   (or when a signal hits us) before we're past LLookupEnd\Function,
	//   then our PC will be reset to LLookupRecover\Function which forcefully
	//   jumps to the cache-miss codepath which have the following
	//   requirements:
	//
	//   GETIMP:
	//     The cache-miss is just returning NULL (setting x0 to 0)
	//
	//   NORMAL and LOOKUP:
	//   - x0 contains the receiver
	//   - x1 contains the selector
	//   - x16 contains the isa
	//   - other registers are set as per calling conventions
	//

	mov	x15, x16			// stash the original isa
LLookupStart\Function:
	// p1 = SEL, p16 = isa
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
	ldr	p10, [x16, #CACHE]				// p10 = mask|buckets
	lsr	p11, p10, #48			// p11 = mask
	and	p10, p10, #0xffffffffffff	// p10 = buckets
	and	w12, w1, w11			// x12 = _cmd & mask
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	ldr	p11, [x16, #CACHE]			// p11 = mask|buckets
#if CONFIG_USE_PREOPT_CACHES
#if __has_feature(ptrauth_calls)
	tbnz	p11, #0, LLookupPreopt\Function
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
#else
	and	p10, p11, #0x0000fffffffffffe	// p10 = buckets
	tbnz	p11, #0, LLookupPreopt\Function
#endif
	eor	p12, p1, p1, LSR #7
	and	p12, p12, p11, LSR #48		// x12 = (_cmd ^ (_cmd >> 7)) & mask
#else
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
	and	p12, p1, p11, LSR #48		// x12 = _cmd & mask
#endif // CONFIG_USE_PREOPT_CACHES
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	ldr	p11, [x16, #CACHE]				// p11 = mask|buckets
	and	p10, p11, #~0xf			// p10 = buckets
	and	p11, p11, #0xf			// p11 = maskShift
	mov	p12, #0xffff
	lsr	p11, p12, p11			// p11 = mask = 0xffff >> p11
	and	p12, p1, p11			// x12 = _cmd & mask
#else
#error Unsupported cache mask storage for ARM64.
#endif

	add	p13, p10, p12, LSL #(1+PTRSHIFT)
						// p13 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

						// do {
1:	ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--
	cmp	p9, p1				//     if (sel != _cmd) {
	b.ne	3f				//         scan more
						//     } else {
2:	CacheHit \Mode				// hit:    call or return imp
						//     }
3:	cbz	p9, \MissLabelDynamic		//     if (sel == 0) goto Miss;
	cmp	p13, p10			// } while (bucket >= buckets)
	b.hs	1b

	// wrap-around:
	//   p10 = first bucket
	//   p11 = mask (and maybe other bits on LP64)
	//   p12 = _cmd & mask
	//
	// A full cache can happen with CACHE_ALLOW_FULL_UTILIZATION.
	// So stop when we circle back to the first probed bucket
	// rather than when hitting the first bucket again.
	//
	// Note that we might probe the initial bucket twice
	// when the first probed slot is the last entry.


#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
	add	p13, p10, w11, UXTW #(1+PTRSHIFT)
						// p13 = buckets + (mask << 1+PTRSHIFT)
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	add	p13, p10, p11, LSR #(48 - (1+PTRSHIFT))
						// p13 = buckets + (mask << 1+PTRSHIFT)
						// see comment about maskZeroBits
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	add	p13, p10, p11, LSL #(1+PTRSHIFT)
						// p13 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
						// p12 = first probed bucket

						// do {
4:	ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--
	cmp	p9, p1				//     if (sel == _cmd)
	b.eq	2b				//         goto hit
	cmp	p9, #0				// } while (sel != 0 &&
	ccmp	p13, p12, #0, ne		//     bucket > first_probed)
	b.hi	4b

LLookupEnd\Function:
LLookupRecover\Function:
	b	\MissLabelDynamic

#if CONFIG_USE_PREOPT_CACHES
#if CACHE_MASK_STORAGE != CACHE_MASK_STORAGE_HIGH_16
#error config unsupported
#endif
LLookupPreopt\Function:
#if __has_feature(ptrauth_calls)
	and	p10, p11, #0x007ffffffffffffe	// p10 = buckets
	autdb	x10, x16			// auth as early as possible
#endif

	// x12 = (_cmd - first_shared_cache_sel)
	adrp	x9, _MagicSelRef@PAGE
	ldr	p9, [x9, _MagicSelRef@PAGEOFF]
	sub	p12, p1, p9

	// w9  = ((_cmd - first_shared_cache_sel) >> hash_shift & hash_mask)
#if __has_feature(ptrauth_calls)
	// bits 63..60 of x11 are the number of bits in hash_mask
	// bits 59..55 of x11 is hash_shift

	lsr	x17, x11, #55			// w17 = (hash_shift, ...)
	lsr	w9, w12, w17			// >>= shift

	lsr	x17, x11, #60			// w17 = mask_bits
	mov	x11, #0x7fff
	lsr	x11, x11, x17			// p11 = mask (0x7fff >> mask_bits)
	and	x9, x9, x11			// &= mask
#else
	// bits 63..53 of x11 is hash_mask
	// bits 52..48 of x11 is hash_shift
	lsr	x17, x11, #48			// w17 = (hash_shift, hash_mask)
	lsr	w9, w12, w17			// >>= shift
	and	x9, x9, x11, LSR #53		// &=  mask
#endif

	ldr	x17, [x10, x9, LSL #3]		// x17 == sel_offs | (imp_offs << 32)
	cmp	x12, w17, uxtw

.if \Mode == GETIMP
	b.ne	\MissLabelConstant		// cache miss
	sub	x0, x16, x17, LSR #32		// imp = isa - imp_offs
	SignAsImp x0
	ret
.else
	b.ne	5f				// cache miss
	sub	x17, x16, x17, LSR #32		// imp = isa - imp_offs
.if \Mode == NORMAL
	br	x17
.elseif \Mode == LOOKUP
	orr x16, x16, #3 // for instrumentation, note that we hit a constant cache
	SignAsImp x17
	ret
.else
.abort  unhandled mode \Mode
.endif

5:	ldursw	x9, [x10, #-8]			// offset -8 is the fallback offset
	add	x16, x16, x9			// compute the fallback isa
	b	LLookupStart\Function		// lookup again with a new isa
.endif
#endif // CONFIG_USE_PREOPT_CACHES

.endmacro
```

接下来按流程分析下，此时调用 `CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached` 

- `mov	x15, x16 // stash the original isa`  在 x15 寄存器中存储 `isa` 的原始数据。 x16 此时存储的是 `class` (类的地址) ，不过对于类来说，其首地址就是类的 `isa` 的地址，指向 `isa` 的值  

#### LLookupStart\Function

- `LLookupStart\Function` 是一个 `label` 标签，当前传入的第二个参数 `Function` 是 `_objc_msgSend` ，这就是开始查找 `_objc_msgSend` 的方法缓存

之后的宏 `CACHE_MASK_STORAGE` 的判断逻辑，当前 `arm64` 真机环境下，其值为 `CACHE_MASK_STORAGE_HIGH_16` (详细参考 《6-1、cache_t的底层分析》拓展3 else小节)

此部分代码判断条件很多，进行缩进整理下

```assembly
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	ldr	p11, [x16, #CACHE]			// p11 = mask|buckets
	#if CONFIG_USE_PREOPT_CACHES
		#if __has_feature(ptrauth_calls)
	tbnz	p11, #0, LLookupPreopt\Function
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
		#else
	and	p10, p11, #0x0000fffffffffffe	// p10 = buckets
	tbnz	p11, #0, LLookupPreopt\Function
		#endif
	eor	p12, p1, p1, LSR #7
	and	p12, p12, p11, LSR #48		// x12 = (_cmd ^ (_cmd >> 7)) & mask
	#else
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
	and	p12, p1, p11, LSR #48		// x12 = _cmd & mask
	#endif // CONFIG_USE_PREOPT_CACHES
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
```

- `ldr	p11, [x16, #CACHE] // p11 = mask|buckets`  

搜索 `CACHE` 的声明，在文件顶部找到声明

```assembly
/* Selected field offsets in class structure */
#define SUPERCLASS       __SIZEOF_POINTER__
#define CACHE            (2 * __SIZEOF_POINTER__)
```

指针当前的 size 是 8，那么 `CACHE` 是 16，此时当前语句可理解为  p11 = isa地址 + 16，16=0x10，那么 p11 就是 isa 地址偏移 0x10 ，即可得到 `cache_t` 的地址，此时 p11 寄存器就存储 `cache_t` 的指针地址

> 由于 `cache_t` 由 `buckets` 数据以及 `mask` 组成，因此也可由 `mask|buckets` 表示其组成 ，注此处的组成如注释描述 ` _bucketsAndMaybeMask is a buckets_t pointer in the low 48 bits ,  _maybeMask is unused, the mask is stored in the top 16 bits.` (详细参考 《6-1、cache_t的底层分析》拓展3 else小节)

##### CONFIG_USE_PREOPT_CACHES

接下来是判断语句这个宏 `CONFIG_USE_PREOPT_CACHES` 在当前真机 `arm64` 环境下取值为 1 ( `preopt` 大概指 `preoption` 优先选择的 或者 `preoptimized` )

```C++
// ----  CONFIG_USE_PREOPT_CACHES ----
#if defined(__arm64__) && TARGET_OS_IOS && !TARGET_OS_SIMULATOR && !TARGET_OS_MACCATALYST
#define CONFIG_USE_PREOPT_CACHES 1
#else
#define CONFIG_USE_PREOPT_CACHES 0
#endif
```

###### __has_feature(ptrauth_calls)

之后是判断语句  `#if __has_feature(ptrauth_calls)` ，`ptrauth` 是在 A12处理器之后存在，那我们先看旧机型的

```assembly
	and	p10, p11, #0x0000fffffffffffe	// p10 = buckets
	tbnz	p11, #0, LLookupPreopt\Function
```

- `and	p10, p11, #0x0000fffffffffffe	// p10 = buckets`  此处使用与操作， `p10 = p11 & 0x0000fffffffffffe` ，此处的 `0x0000fffffffffffe` 就是下面 [参考1] `cache_t` 对照代码静态变量中的 `preoptBucketsMask`，根据名字也可推断出此掩码是为了获取 `buckets` ，同时对照代码中的结构也可进行验证
  - 首先此时 `_bucketsAndMaybeMask is a buckets_t pointer in the low 48 bits` ，`buckets` 存储在后 48 位
  - 由于 ` 0: always 1` ，第0位总是1( preoptBucketsMarker = 1ul )[参考1]，并未代表数据含义，无需进行掩码获取处理。因此使用掩码 `0x0000fffffffffffe`，11个 f，1个 e，总共 47位1
- `tbnz	p11, #0, LLookupPreopt\Function` 

> TBNZ Rt, bit, label // Test and branch if Rt<bit> is not zero  ( 6.4 Flow control [引用1])

解析：`cache_t` 的指针地址(p11寄存器) 的第0位(#0, 立即寻址，取值0) 不为0就跳转  `LLookupPreopt\Function` 偏移位置。 `LLookupPreopt\Function` 标签， 此标签对应逻辑稍后专门分析

> 注意：根据 [参考1] 对照位置代码的注释  ` 0: always 1` ，首位大部分情况下为 1，此对应的是 `preoptBucketsMask `,  其实际对应的是 `p10` 寄存器，注意区分别代入当前判断的 `p11` 寄存器

再看 `__has_feature(ptrauth_calls)` 判断下的新机型处理逻辑

```assembly
	#if __has_feature(ptrauth_calls)
	tbnz	p11, #0, LLookupPreopt\Function
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
```

与旧机型区别

- 顺序相反，先进行了 `tbnz` 判断，再进行与操作，
- 与操作掩码为 `#0x0000ffffffffffff`  其首位为 1

###### #endif 

`ptraut` 的判断结束，执行后续逻辑

```assembly
	eor	p12, p1, p1, LSR #7
	and	p12, p12, p11, LSR #48		// x12 = (_cmd ^ (_cmd >> 7)) & mask
```

- `eor	p12, p1, p1, LSR #7`  

  - `eor` 异或操作 ； `LSR` 逻辑右移；`#` 表示立即寻址

  - `LSR #7` 表示逻辑右移 7 位；
  - `p1`  根据注释中 `x1 contains the selector` , 标识 `SEL` 指向 `cmd`
  - 此句总体解释为 ` x12 = _cmd ^ (_cmd >> 7) `

> Logical Shift Right(LSR). The LSR instruction performs division by a power of 2  ( 6.2.3 Shift operations [引用1])

- `and	p12, p12, p11, LSR #48`  

  - 此句解释为 `x12 = x12 & (x11 >> 48)`

  - 追溯可知 `p11`指向 `cache_t` 地址，即  `p11 = mask|buckets`，根据[参考1] 中可知 `mask = x11 >> 48`

    ```objective-c
    // _bucketsAndMaybeMask is a buckets_t pointer in the low 48 bits
    // _maybeMask is unused, the mask is stored in the top 16 bits.
    ```

    > 针对 mask 参考 《6-3、cache_t源码分析》 reallocate() 小节中的 setBucketsAndMask 函数的分析，可知 mask 存储的是 （bukect_t 容器的容量 - 1），即最后一个索引位置
  
    因此使用 `& mask` 计算后的结果这个索引值不会超过容器的容量，可以稍微窥探体会下当前的算法逻辑
  
  - 最终结果可得注释 `x12 = (_cmd ^ (_cmd >> 7)) & mask` 

##### #else

非 `CONFIG_USE_PREOPT_CACHES`  下环境，指的是 非 `arm64` 的真机、模拟器和 `Mac` 环境下

```assembly
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
	and	p12, p1, p11, LSR #48		// x12 = _cmd & mask
```

代码意义很清晰

- `p10 = p11 & #0x0000ffffffffffff`  取  `p11 = mask|buckets`  的后 48 位    `buckets`
- `and	p12, p1, p11, LSR #48		// x12 = _cmd & mask`  根据上面的类似分析理解起来很清晰

##### #endif  + 小结

此处 `#if CACHE_MASK_STORAGE` 判断部分的汇编代码，对照逻辑为 `cache_hash` 方法，其意义是通过 `cache_hash()` 方法来计算初始索引位置

```C++
// Class points to cache. SEL is key. Cache buckets store SEL+IMP.
// Caches are never built in the dyld shared cache.

static inline mask_t cache_hash(SEL sel, mask_t mask) 
{
    uintptr_t value = (uintptr_t)sel;
#if CONFIG_USE_PREOPT_CACHES  //arm64真机环境为 1
    value ^= value >> 7;  //// 此处对照 #if __has_feature(ptrauth_calls) 判断外的汇编代码
#endif
    return (mask_t)(value & mask);
}
```

在 `CACHE_MASK_STORAGE`  的其他枚举值下的判断，虽然根据 `CACHE_MASK_STORAGE` 不同枚举下的 `cache_t` 结构不同造成汇编代码不同，但最终都是达成如下的赋值

```assembly
// p11 = mask // 部分枚举下
// p10 = buckets
// x12 = _cmd & mask // p12是一个 cache_hash 函数产生的 index 索引值
```

##### 其后逻辑

在上述的判断逻辑中，除了两处 `tbnz	p11, #0, LLookupPreopt\Function` 代码大部分跳转 `LLookupPreopt\Function` 标签外，其余都顺序执行到判断结束直接执行后续的方法，这些后续的方法也应对使用 Mac(旧版非M1) 运行工程时执行的代码，先来分析下这些逻辑，与 《6-3、cache_t源码分析》进行对照

###### 查找逻辑1

首先进行的查找逻辑

```assembly
	add	p13, p10, p12, LSL #(1+PTRSHIFT)
						// p13 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

						// do {
1:	ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--
	cmp	p9, p1				//     if (sel != _cmd) {
	b.ne	3f				//         scan more
						//     } else {
2:	CacheHit \Mode				// hit:    call or return imp
						//     }
3:	cbz	p9, \MissLabelDynamic		//     if (sel == 0) goto Miss;
	cmp	p13, p10			// } while (bucket >= buckets)
	b.hs	1b
	
	// wrap-around: (绕回)
	//   p10 = first bucket
	//   p11 = mask (and maybe other bits on LP64)
	//   p12 = _cmd & mask
	//
	// A full cache can happen with CACHE_ALLOW_FULL_UTILIZATION.
	// So stop when we circle back to the first probed bucket
	// rather than when hitting the first bucket again.
	//
	// Note that we might probe the initial bucket twice
	// when the first probed slot is the last entry.


#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
	add	p13, p10, w11, UXTW #(1+PTRSHIFT) 
						// p13 = buckets + (mask << 1+PTRSHIFT)
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	add	p13, p10, p11, LSR #(48 - (1+PTRSHIFT))
						// p13 = buckets + (mask << 1+PTRSHIFT)
						// see comment about maskZeroBits
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	add	p13, p10, p11, LSL #(1+PTRSHIFT)
						// p13 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
						// p12 = first probed bucket

						// do {
4:	ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--
	cmp	p9, p1				//     if (sel == _cmd)
	b.eq	2b				//         goto hit
	cmp	p9, #0				// } while (sel != 0 &&
	ccmp	p13, p12, #0, ne		//     bucket > first_probed)
	b.hi	4b
	
LLookupEnd\Function:
LLookupRecover\Function:
	b	\MissLabelDynamic
```

- `add	p13, p10, p12, LSL #(1+PTRSHIFT)`
  - 由前述代码可知 `p10 = buckets` 、`p12` 是一个索引值
  - ` Logical Shift Left(LSL). The LSL instruction performs multiplication by a power of 2  ( 6.2.3 Shift operations [引用1])`  逻辑左移，相当于乘以 2的幂
  - `PTRSHIFT` 此处为 3。 [参考2] 此处以 `__arm64__ & __LP64__` 环境下的 `CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS` 和 `CACHE_MASK_STORAGE_HIGH_16` 为前提，不考虑 `arm64_32`环境下的值
  - `LSL #(1+PTRSHIFT)` 就等同于乘以 2 的 (1 + 3) = 4 次方，即 `*16` ，转换成十六进制就是 `0x10` 。 再使用索引值 `p12 * 16 `，就很直观的体会到索引值(p12)个单位长度(步长)的偏移  (# 表示立即寻址，使用立即数)
  - 此时整句释义为 `p13 = buckets + (index * 16)`，即是 `p13 = b[i]` 从 `buckets`的首地址偏移 `p12` 个 `bucket_t` 的大小来找到起始查询位置  [参考3]

- 1： 方法块

  - `ldp	p17, p9, [x13], #-BUCKET_SIZE` 
    - `LDP -- load Pair`，此格式的指令描述为：` Loads doubleword at address X13 into p17 and the doubleword at address X13 + 8 into p9 and adds -BUCKET_SIZE to X13.` [引用1-1]
    - `#define BUCKET_SIZE   (2 * __SIZEOF_POINTER__)`  至于 `BUCKET_SIZE` 则为 2 * 8 = 16 为一个 `bucket_t` 的大小
    - `x13` 存储当前偏移位置 `bucket_t` 的首地址，`bucket_t` 由 `sel` 和 `imp` 组成，`__arm64__` 时 `imp` 在前，`sel` 在后 [参考 3]
    - 综上所述， `p17 = imp、p9 = sel`

  - `cmp	p9, p1 `  与 `b.ne 3f` ,  这两句就可解释为 [拓展1]

    ```C++
    if (sel != _cmd) { //对比当前位置 sel 与传入的 sel 是否为同一个
      //向下跳转 3 
    } else {
      //向下执行 2
    }
    ```

- 2：方法块  `CacheHit \Mode`  

  - `CacheHit`   为 `macro`宏 

    ```assembly
    #define NORMAL 0
    #define GETIMP 1
    #define LOOKUP 2
    
    // CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
    .macro CacheHit
    .if $0 == NORMAL
    	TailCallCachedImp x17, x10, x1, x16	// authenticate and call imp
    .elseif $0 == GETIMP
    	mov	p0, p17
    	cbz	p0, 9f			// don't ptrauth a nil imp
    	AuthAndResignAsIMP x0, x10, x1, x16	// authenticate imp and re-sign as IMP
    9:	ret				// return IMP
    .elseif $0 == LOOKUP
    	// No nil check for ptrauth: the caller would crash anyway when they
    	// jump to a nil IMP. We don't care if that jump also fails ptrauth.
    	AuthAndResignAsIMP x17, x10, x1, x16	// authenticate imp and re-sign as IMP
    	cmp	x16, x15
    	cinc	x16, x16, ne			// x16 += 1 when x15 != x16 (for instrumentation ; fallback to the parent class)
    	ret				// return imp via x17
    .else
    .abort oops
    .endif
    .endmacro
    ```

  - `\Mode` 为 `CacheHit` 首个参数，使用 `$0`标识；同时 `Mode` 为 `CacheLookup` 传入的首个参数，其值当前为 `NORMAL`

  - `TailCallCachedImp x17, x10, x1, x16	// authenticate and call imp`

    其注释为 验证并调起 `imp`，查看其实现

    ```assembly
    #if __has_feature(ptrauth_calls)
    // JOP
    
    .macro TailCallCachedImp
    	// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa
    	eor	$1, $1, $2	// mix SEL into ptrauth modifier
    	eor	$1, $1, $3  // mix isa into ptrauth modifier
    	brab	$0, $1
    .endmacro
      
      // JOP
    #else
    // not JOP
    
    .macro TailCallCachedImp
    	// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa
    	eor	$0, $0, $3
    	br	$0
    .endmacro
      
      // not JOP
    #endif
    ```

    此处又区分 `ptrauth` 和 其他，查看下其他的普通样式时的逻辑

    - `eor	$0, $0, $3 `  `eor` 是异或操作，根据其上的注释可得此句可理解为 `imp ^ isa` ，这个好熟悉，在 `《6-2、重写源码探索方法》的 拓展2  imp ^ (uintptr_t)cls 小节` 进行了详细解释，参考一下其本质上是对 `imp` 进行的编码操作
    - `br $0`  `br` 指令参考 [引用 1-3] 中解释 `BR Xn  -  Absolute branch in address Xn`  跳转 Xn 寄存器指定的地址处。 此时就是方法缓存命中，直接执行方法
    - 此时 objc_msgSend 通过 SEL 查找 IMP 的命中流程就完整结束了

- 3：方法块

  - 前置条件是  `sel != _cmd` ，当前索引位置的方法与要查找的方法不匹配

  - `cbz	p9, \MissLabelDynamic` 

    - `cbz` 指令参考[拓展2]，根据其后的注释 `if (sel == 0) goto Miss;`  也可理解其对 `p9` 进行了零值判断
    - 判断 `sel == 0` 时(在遍历查找完成，最终 `sel` 为空)，跳转 `\MissLabelDynamic` ，`\MissLabelDynamic` 为 `CacheLookup` 的第三个入参 `__objc_msgSend_uncached` ，此时则执行  `__objc_msgSend_uncached` 未进行缓存的相关逻辑
    - `sel` 非0则执行后续逻辑

  -  sel  非零时

    ```assembly
    cmp	p13, p10			// } while (bucket >= buckets)
    b.hs	1b
    ```

    根据[拓展1] 以及 [引用 1-4] 中的指令向解析

    ```C++
    // hs 表示 Greater than, Equal to, 即 >= 
    // p13 为 当前偏移位置的 bucket_t
    // p10 为 buckets 首地址
    if (bucket >= buckets) { 
      //b 表示 backward; f 表示 forward 
      1b //此时跳转回到代码块 1 继续执行
    } else {
      //向下执行
    }
    ```

  - `bucket < buckets`  通过 方法块1 进行向前偏移至首地址前时，此时进行了 `wrap-around` 绕回

    ```assembly
    	//   p10 = first bucket
    	//   p11 = mask (and maybe other bits on LP64)
    	//   p12 = _cmd & mask
    
    #elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    	add	p13, p10, p11, LSR #(48 - (1+PTRSHIFT))
    						// p13 = buckets + (mask << 1+PTRSHIFT)
    						// see comment about maskZeroBits
    // ...
    #endif
    	add	p12, p10, p12, LSL #(1+PTRSHIFT)
    						// p12 = first probed bucket
    ```

    不同枚举策略下应对不同环境，想要达到的目的是相同的，这里针对 arm64 真机的 `CACHE_MASK_STORAGE_HIGH_16` 枚举下的代码进行解析

    - `add	p13, p10, p11, LSR #(48 - (1+PTRSHIFT))` 

      `LSR` 逻辑右移；`#` 立即寻址；arm64 真机环境下 `PTRSHIFT` 为 3；

      此处 `LSR #(48 - (1+PTRSHIFT))` 可分解为  `LSR #48` 和  `LSL #(1+PTRSHIFT)` 两步骤

      > 由于 `maskZeroBits` 是 4 位的 0 [参考1] (Additional bits after the mask which must be zero )，因此可把此两步骤合成直接右移 44位，最终结果一致，但是注意按两步骤方式理解

      根据上文可知 `p11`指向 `cache_t` 地址，即  `p11 = mask|buckets`，根据[参考1] 中可知 `mask = x11 >> 48`

      ```objective-c
      // _bucketsAndMaybeMask is a buckets_t pointer in the low 48 bits
      // _maybeMask is unused, the mask is stored in the top 16 bits.
      ```

      `LSL #(1+PTRSHIFT)` 就等同于乘以 2 的 (1 + 3) = 4 次方，即 `*16` ，转换成十六进制就是 `0x10` 。 再使用索引值 `mask * 16 `，就很直观的体会到索引值(mask)个单位长度(步长)的偏移  (# 表示立即寻址，使用立即数)

      > 针对 mask 参考 《6-3、cache_t源码分析》 reallocate() 小节中的 setBucketsAndMask 函数的分析，可知 mask 存储的是 （bukect_t 容器的容量 - 1），即最后一个索引位置
      >
      > 此处的 mask 实现方法根据枚举值为 [参考4] 中对应方法

      此时整句释义为 `p13 = buckets + (mask * 16)`，即是 `p13 = b[i]` 从 `buckets`的首地址偏移 `mask` 个 `bucket_t` 的大小来找到起始查询位置  [参考3]，综合 `mask` 的上述引用，此时 `p13` 偏移到最后一个 `bucket_t` 的地址

    - `add	p12, p10, p12, LSL #(1+PTRSHIFT)`

      > 参考上文，此时  p12是一个 cache_hash 函数产生的 index 索引值

      此句释义为 `p12 = buckets + (p12 * 16)`  ，所以此时 `p12` 赋值为其索引值对应的 `bucket_t` 的地址，即是 `cache_hash` 之后首次查找的索引位置，与注释 `p12 = first probed bucket` 等同

- 4：方法块

  - `ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--`

    此句与方法块1中的首句相同，` Loads doubleword at address X13 into p17 and the doubleword at address X13 + 8 into p9 and adds -BUCKET_SIZE to X13.` [引用1-1]

    理解起来很容易了，当前偏移位置的 `bucket_t` 其  `p17 = imp、p9 = sel`，之后再自减向前偏移

  - `cmp	p9, p1` 与 `b.eq 2b` 参考[拓展1、引用1-4]

    ```C++
     if (sel == _cmd) { //对比当前位置 sel 与传入的 sel 是否为同一个
       //相同，即命中，跳转 
       2 CacheHit
     } else {
       //向下执行
     }
    ```

  - `sel != _cmd` 的后续

    ```assembly
    cmp	p9, #0				// } while (sel != 0 &&
    ccmp	p13, p12, #0, ne		//     bucket > first_probed)
    b.hi	4b
    ```

    `ccmp` 参考[拓展4]，可理解上述代码为

    ```c++
    if (sel != 0) {  // ne 表示 p9 != 0 
      if (bucket > first_probed) { // hi 表示 greater than [引用 1-4]
        b 4b   // offset backward 4. 偏移回到 4
      } else {
        nzcv = 0  //标志位置为 #0
      }
    }
    
    //此代码等同于注释语句的 while 循环
    ```

    > 注意此处 ccmp 对比当前bucket 和 首次hash索引值位置的 bucket，首次hash索引值之前的已经对比过了

  - 上述条件判断之外的情况则直接执行后续 `MissLabelDynamic`，即执行  `__objc_msgSend_uncached` 未进行缓存的相关逻辑
  
    ```
    LLookupEnd\Function:
    LLookupRecover\Function:
    	b	\MissLabelDynamic
    ```
  
    

#### LLookupPreopt\Function

此标签对应的源代码如下

```assembly
#if CONFIG_USE_PREOPT_CACHES
#if CACHE_MASK_STORAGE != CACHE_MASK_STORAGE_HIGH_16
#error config unsupported
#endif
LLookupPreopt\Function:
#if __has_feature(ptrauth_calls)
	and	p10, p11, #0x007ffffffffffffe	// p10 = buckets
	autdb	x10, x16			// auth as early as possible
#endif

	// x12 = (_cmd - first_shared_cache_sel)
	adrp	x9, _MagicSelRef@PAGE
	ldr	p9, [x9, _MagicSelRef@PAGEOFF]
	sub	p12, p1, p9

	// w9  = ((_cmd - first_shared_cache_sel) >> hash_shift & hash_mask)
#if __has_feature(ptrauth_calls)
	// bits 63..60 of x11 are the number of bits in hash_mask
	// bits 59..55 of x11 is hash_shift

	lsr	x17, x11, #55			// w17 = (hash_shift, ...)
	lsr	w9, w12, w17			// >>= shift

	lsr	x17, x11, #60			// w17 = mask_bits
	mov	x11, #0x7fff
	lsr	x11, x11, x17			// p11 = mask (0x7fff >> mask_bits)
	and	x9, x9, x11			// &= mask
#else
	// bits 63..53 of x11 is hash_mask
	// bits 52..48 of x11 is hash_shift
	lsr	x17, x11, #48			// w17 = (hash_shift, hash_mask)
	lsr	w9, w12, w17			// >>= shift
	and	x9, x9, x11, LSR #53		// &=  mask
#endif

	ldr	x17, [x10, x9, LSL #3]		// x17 == sel_offs | (imp_offs << 32)
	cmp	x12, w17, uxtw

.if \Mode == GETIMP
	b.ne	\MissLabelConstant		// cache miss
	sub	x0, x16, x17, LSR #32		// imp = isa - imp_offs
	SignAsImp x0
	ret
.else
	b.ne	5f				// cache miss
	sub	x17, x16, x17, LSR #32		// imp = isa - imp_offs
.if \Mode == NORMAL
	br	x17                          
.elseif \Mode == LOOKUP
	orr x16, x16, #3 // for instrumentation, note that we hit a constant cache  
	SignAsImp x17
	ret
.else
.abort  unhandled mode \Mode
.endif

5:	ldursw	x9, [x10, #-8]			// offset -8 is the fallback offset
	add	x16, x16, x9			// compute the fallback isa
	b	LLookupStart\Function		// lookup again with a new isa
.endif
#endif // CONFIG_USE_PREOPT_CACHES
```

其跳转前的代码条件判断，参考 `CONFIG_USE_PREOPT_CACHES` 小节的解析，当前环境为 `arm64` 真机环境下

##### 代码分析：

###### ptrauth_calls

首先进行 `ptrauth( point authentication 指针认证 )` 参考 《4-2、类的结构初探》中的 `ptrauth_calls`

```assembly
#if __has_feature(ptrauth_calls)
	and	p10, p11, #0x007ffffffffffffe	// p10 = preoptBuckets
	// p10 = p11 & 0x007ffffffffffffe，由[参考1]中，preoptBucketsMask = 0x007ffffffffffffe，可知 p10 = preoptBuckets [参考1 此处不能写成 preopt_cache_t]
	autdb	x10, x16			// auth as early as possible // 参考[引用2-1]中描述，用来认证数据地址
#endif
```

###### adrp 指令代码 

```assembly
	// x12 = (_cmd - first_shared_cache_sel)
	adrp	x9, _MagicSelRef@PAGE
	ldr	p9, [x9, _MagicSelRef@PAGEOFF]
	sub	p12, p1, p9
```

- `adrp	x9, _MagicSelRef@PAGE` 计算  `_MagicSelRef@PAGE` 所在分页的基地址赋值给 x9 寄存器 [拓展5]

- `ldr	p9, [x9, _MagicSelRef@PAGEOFF]`   `[]` 存储器寻址，`Load from address x9 + _MagicSelRef@PAGEOFF`  可简译为  `p9 =  x9 + _MagicSelRef@PAGEOFF`

- `sub` 就很直接  `x12 = (_cmd - first_shared_cache_sel)`  此时 x12 存储的是查找方法所在地址到首个共享缓存方法地址的偏移量

  > 根据注释可推断 `_MagicSelRef` 是有关 `first_shared_cache_sel` 共享缓存方法的

###### ptrauth_calls 判断语句

```assembly
	// w9  = ((_cmd - first_shared_cache_sel) >> hash_shift & hash_mask)
#if __has_feature(ptrauth_calls)
	// bits 63..60 of x11 are the number of bits in hash_mask
	// bits 59..55 of x11 is hash_shift

	lsr	x17, x11, #55			// w17 = (hash_shift, ...)   // >>= 55 
	lsr	w9, w12, w17			// >>= shift

	lsr	x17, x11, #60			// w17 = mask_bits           // >>= 60  hash_mask_shift
	
	//以下代码，对照[参考1]中对 mask_shift 的组装方式，反向推导 mask_shift 转换成 mask 的逻辑
	mov	x11, #0x7fff                                   // Copy #0x7fff  to x11
	lsr	x11, x11, x17			// p11 = mask (0x7fff >> mask_bits) 
	
	and	x9, x9, x11			// &= mask
#else
	// bits 63..53 of x11 is hash_mask
	// bits 52..48 of x11 is hash_shift
	lsr	x17, x11, #48			// w17 = (hash_shift, hash_mask)
	lsr	w9, w12, w17			// >>= shift
	and	x9, x9, x11, LSR #53		// &=  mask
#endif
```

此部分代码参考 [参考1] 中的结构，最终是在不同架构下得到 `w9  = ((_cmd - first_shared_cache_sel) >> hash_shift & hash_mask)`

```C++
#if CONFIG_USE_PREOPT_CACHES
    static constexpr uintptr_t preoptBucketsMarker = 1ul;
#if __has_feature(ptrauth_calls)
    // 63..60: hash_mask_shift
    // 59..55: hash_shift
    // 54.. 1: buckets ptr + auth
    //      0: always 1
    static constexpr uintptr_t preoptBucketsMask = 0x007ffffffffffffe;
#else
    // 63..53: hash_mask
    // 52..48: hash_shift
    // 47.. 1: buckets ptr
    //      0: always 1
    static constexpr uintptr_t preoptBucketsMask = 0x0000fffffffffffe;
#endif
#endif // CONFIG_USE_PREOPT_CACHES
```

此处 `w9` 应该怎么理解呢？在后面的对照解析部分进行了对照理解

- `lsr	w9, w12, w17			// >>= shift`  

  此处 `w17 =  (hash_shift, hash_mask)` 如何理解此指令实现  `>>= shift` ？

  - 根据上面架构的位数标志，`hash_shift` 为 4位数据，最大的数字表示为 `31` ，同时 `hash_shift` 在 `w17` 的低位
  - `lsr	w9, w12, w17` 由于指令使用 32位寄存器，所以在使用时偏移位数参数 `w17`，其大小不能超过32位，则取低4位即 `hash_shift` 的数据进行偏移(可等同于 `w17 & 0x0f` )，此时就等同于 `>>= shift`
  - 拓展 对于 `lsr x9, x12, x17` 使用64位寄存器，此时的使用偏移位数参数 `x17` ，其大小不能超过 64 位，即取低5位，可等同于 `x17 & 0x1f`

  

###### cmp 判断条件语句

```assembly
	ldr	x17, [x10, x9, LSL #3]		// x17 == sel_offs | (imp_offs << 32)  //Load from address x10(buckets) + (X9 << 3)
	// x9 * 8 8为 preopt_cache_entry_t 的大小，preopt_cache_entry_t entries[] 的步长 x17 = preopt_cache_entry_t = sel_offs | (imp_offs << 32)
	
	cmp	x12, w17, uxtw                // uxtw 补零至单字

.if \Mode == GETIMP                 // Mode 为 CacheLookup 传入的首个参数，其值当前为 NORMAL
	b.ne	\MissLabelConstant		// cache miss   // x12 != x17
	
	sub	x0, x16, x17, LSR #32		// imp = isa - imp_offs
	SignAsImp x0                      // imp 签名
	// 对照 (uintptr_t)cls() - entry.imp_offs ==
                (uintptr_t)ptrauth_strip(imp, ptrauth_key_function_pointer) 可看出 imp是这样进行PAC解码的
	
	ret                               // return 
.else
	b.ne	5f				// cache miss   // x12 != x17
	
	sub	x17, x16, x17, LSR #32		// imp = isa - imp_offs
  .if \Mode == NORMAL
	br	x17                        //Absolute branch to address in X17 跳转imp
  .elseif \Mode == LOOKUP
	orr x16, x16, #3 // for instrumentation, note that we hit a constant cache  // x16 = x16 | 3
	SignAsImp x17
	ret
  .else
  .abort  unhandled mode \Mode
  .endif
     
5:	ldursw	x9, [x10, #-8]			// offset -8 is the fallback offset   
	add	x16, x16, x9			// compute the fallback isa                   //x16 = isa + x9
	b	LLookupStart\Function		// lookup again with a new isa
.endif
```

- `cmp x12, w17, uxtw `  结合后续的 `b.ne	\MissLabelConstant`

  x12 存储的是查找方法所在地址到首个共享缓存方法地址的偏移量

  > 那么此时可以等同理解为 `x17` 是在指定算法得出的索引值对应的 imp 到首个共享缓存方法地址的偏移量

  这样才能对比不等后直接缓存未命中

- 此段代码整体理解为对比 `当前查找 imp 到首个共享缓存方法地址的偏移量` 与 `PREOPT_CACHES (x17)` 是否相同，相同则命中跳转，否则计算  `fallback isa ` 继续回到查找方法整体流程

- `SignAsImp x0` 在命中后 `return` 前，进行了此指令，查找 `SignAsImp` 指令

  ```assembly
  //arm64-asm.h中
  #if __has_feature(ptrauth_calls)
  // JOP [拓展6]
  //...省略其他
  
  .macro SignAsImp
  	paciza	$0
  .endmacro
  
  // JOP
  #else
  // not JOP
  //...省略其他
  
  .macro SignAsImp
  .endmacro
  
  // not JOP
  #endif
  ```

  在 `ptrauth_calls` 指针验证(即 A12 以上处理器机型)时，进行 `paciza	$0` 指令操作，其他情况什么都不进行

  - `paciza	$0`  参考[引用2-4]

    ```tex
    Pointer Authentication Code for Instruction address, using key A.This instruction computes and inserts a pointer authentication code for an instruction address, using a modifier and key A.
    ```

    对地址进行计算和插入 `PAC` 

  - 最终在 `ptrauth_calls` 指针验证(即 A12 以上处理器机型)时，对 `$0` 进行 `PAC` 计算和插入，其他环境无需进行

- `orr x16 x16 #3` 

  ```assembly
  //objc-msg-arm64.s CacheLookup 的方法注释中
  /********************************************************************
   *
   * CacheLookup NORMAL|GETIMP|LOOKUP <function> MissLabelDynamic MissLabelConstant
   // 省略部分
  On exit: (found) calls or returns IMP
   *                  with x16 = class, x17 = IMP
   *                  In LOOKUP mode, the two low bits are set to 0x3
   *                  if we hit a constant cache (used in objc_trace)
   //省略其他
   *
  ********************************************************************/
  ```

  此时处于 `LOOKUP` 分支条件判断下，同时匹配到缓存方法，因此按照注释中的备注  `the two low bits are set to 0x3` ，即 `x16 = x16 | 0x11`



##### 对照解析

为了理解 `LLookupPreopt` 标签方法的代码，查找 `PREOPT_CACHES` 对应的相关方法，查找到部分方法，有助于理解和反推当前方法的实现

###### preopt_cache_t

首先展示 `preopt_cache_t` 结构体的成员组成，其相应的成员名称，对照代码中的注释，以及后续对照函数中的使用

```C++
/* dyld_shared_cache_builder and obj-C agree on these definitions */
struct preopt_cache_entry_t {
    uint32_t sel_offs;
    uint32_t imp_offs;
};

/* dyld_shared_cache_builder and obj-C agree on these definitions */
struct preopt_cache_t {
    int32_t  fallback_class_offset;
    union {
        struct {
            uint16_t shift       :  5;
            uint16_t mask        : 11;
        };
        uint16_t hash_params;
    };
    uint16_t occupied    : 14;
    uint16_t has_inlines :  1;
    uint16_t bit_one     :  1;
    preopt_cache_entry_t entries[];

    inline int capacity() const {
        return mask + 1;
    }
};
```

对于 `union` 联合体的组成，可推断出结论：`shift` 和 `mask` 形成的整体结构等同于 `hash_params`

```C++
    union {
        struct {
            uint16_t shift       :  5;
            uint16_t mask        : 11;
        };
        uint16_t hash_params;
    };
```



###### cache_t::preopt_cache()

对照  `objc-cache.mm`  中的   `cache_t::preopt_cache()` 函数

```C++
const preopt_cache_t *cache_t::preopt_cache() const
{
    auto addr = _bucketsAndMaybeMask.load(memory_order_relaxed);
    addr &= preoptBucketsMask;
#if __has_feature(ptrauth_calls)
#if __BUILDING_OBJCDT__
    addr = (uintptr_t)ptrauth_strip((preopt_cache_entry_t *)addr,
            ptrauth_key_process_dependent_data);
#else
    addr = (uintptr_t)ptrauth_auth_data((preopt_cache_entry_t *)addr,
            ptrauth_key_process_dependent_data, (uintptr_t)cls());
#endif
#endif
    return (preopt_cache_t *)(addr - sizeof(preopt_cache_t));
}
```

- ` addr &= preoptBucketsMask` 

  等同于 `and	p10, p11, #0x007ffffffffffffe`  

  即 `mask|buckets & preoptBucketsMask`

- `autdb	x10, x16` 则对应其后的

  ```C++
  addr = (uintptr_t)ptrauth_auth_data((preopt_cache_entry_t *)addr,ptrauth_key_process_dependent_data, (uintptr_t)cls());
  ```

  搜索 `ptrauth_auth_data` 函数，在 `ptrauth.h` 中查到宏

  ```C++
  #define ptrauth_auth_data(__value, __old_key, __old_data) __value
  ```

  

###### shouldFlush

对照 `objc-cache.mm` 中的 `shouldFlush` 函数

```C++
#if CONFIG_USE_PREOPT_CACHES
//省略...
bool cache_t::shouldFlush(SEL sel, IMP imp) const
{
    // This test isn't backwards: disguised caches aren't "strict"
    // constant optimized caches
    if (!isConstantOptimizedCache(/*strict*/true)) {
        const preopt_cache_t *cache = disguised_preopt_cache();
        if (cache) {
            uintptr_t offs = (uintptr_t)sel - (uintptr_t)@selector(🤯);
            uintptr_t slot = ((offs >> cache->shift) & cache->mask);
            auto &entry = cache->entries[slot];

            return entry.sel_offs == offs &&
                (uintptr_t)cls() - entry.imp_offs ==
                (uintptr_t)ptrauth_strip(imp, ptrauth_key_function_pointer);
        }
    }

    return cache_getImp(cls(), sel) == imp;
}
//省略...
#endif
```

- `const preopt_cache_t *cache = disguised_preopt_cache()` 

  此时的 `x10 = buchets` 也应该对照 `const preopt_cache_t *cache = disguised_preopt_cache()` ，此时 `x10` 应该为 `preoptBuckets ` 

  > 注意 `x10`不是 `preopt_cache_t` 
  >
  >  [参考1 `maybeConvertToPreoptimized`] 

- `uintptr_t offs = (uintptr_t)sel - (uintptr_t)@selector(🤯);` 

  对照 `x12 = (_cmd - first_shared_cache_sel)` ，理解 `x12` 存储的是当前查找方法所在地址到首个共享缓存方法`@selector(🤯)` 地址的偏移量

- `uintptr_t slot = ((offs >> cache->shift) & cache->mask);` 

  对照 `w9 = ((_cmd - first_shared_cache_sel) >> hash_shift & hash_mask)`

  可理解 `w9 = slot` ，相当于 `CacheLookup` 中使用 `hash` 函数获取的索引值，其实现方法就等同于指定的 `hash` 函数

- `auto &entry = cache->entries[slot];`

  对照  `ldr	x17, [x10, x9, LSL #3]` 理解  ` x17 = entries + slot * 8`

  - 查看 `preopt_cache_t` 的结构，对应 `cache->entries` 数组成员为 `preopt_cache_entry_t` 其由 `sel_offs ` 与 `imp_offs` 两个 `unint32_t` 类型数据组成，其大小为步长 `4 * 2 = 8 ` 
  - 此时 `x10` 应该为 `preoptBuckets ` ，根据 [参考1 `maybeConvertToPreoptimized`] 的解析，可知 `preoptBuckets` 的核心数据 `buckets` 即是 `preopt_cache_t` 的 `entries` 数组指针/首地址，此时类比 `isa` 与 `class` [参考 《3-4、Isa探索》]，`x10` 即指向 `entries` 的首地址
  - 此时 `x17 ` 作为指定索引值 `slot ` 位置的 `preopt_cache_entry_t` 数据的地址，可理解为 `x17 = entry[slot]`  ，此处注释还存在 `x17 == sel_offs | (imp_offs << 32)` ，此方式表明了 `preopt_cache_entry_t` 结构体中成员的组成方式，`imp_offs` 左移 32 位形成高32位与低32位的 `sel_offs` 进行组装

- 其后的判断语句  `entry.sel_offs == offs` 

  对应 `cmp  x12, w17, uxtw  ` 与其后续的 `b.ne` 相关语句

  此处 `x17 => w17` 从64位寄存器变更为32位寄存器，即获取 `x17` 低32位的 `sel_offs` 

- 同时的判断条件  `&& (uintptr_t)cls() - entry.imp_offs == (uintptr_t)ptrauth_strip(imp, ptrauth_key_function_pointer);`

  则与 `sub	x0, x16, x17, LSR #32		// imp = isa - imp_offs`  代码对照

  判断条件的结果与 `entry.sel_offs == offs` 语句一起 ，对应表示是否命中 `PREOPT_CACHES` 

  `sub` 语句则是在 `cmp`  语句判断的非 `ne` 分支下，即 `x12 = x17 ` 时进行，与判断条件语句等同
  
  - `(uintptr_t)ptrauth_strip(imp, ptrauth_key_function_pointer)` 进行 `PAC` 脱去，即进行了解码，对应 `SignAsImp x0` 时的  `paciza	$0` [引用 2-4] 的 `PAC` 插入操作即编码操作



###### preoptFallbackClass()

```C++
Class cache_t::preoptFallbackClass() const
{
    return (Class)((uintptr_t)cls() + preopt_cache()->fallback_class_offset);
}
```

对照 `fallback` 相关汇编代码

```assembly
5:	ldursw	x9, [x10, #-8]			// offset -8 is the fallback offset   
	add	x16, x16, x9			// compute the fallback isa                   //x16 = isa + x9
	b	LLookupStart\Function		// lookup again with a new isa
```

- `ldursw	x9, [x10, #-8]`  对照 `preopt_cache()->fallback_class_offset)` 

  此时 `x10 = entries` ，使用地址进行 `-8`，此时则对照 `preopt_cache_t` 的结构体结构，向上偏移 8 字节/ 64 位，即可找到 `int32_t  fallback_class_offset` 的地址

  ```C++
  struct preopt_cache_t {
      int32_t  fallback_class_offset; // int32_t : 32位/4字节
      union {   //结构体 16位/2字节
          struct {
              uint16_t shift       :  5;
              uint16_t mask        : 11;
          };
          uint16_t hash_params;
      };
     // 三个参数共 16位/2字节
      uint16_t occupied    : 14;
      uint16_t has_inlines :  1;
      uint16_t bit_one     :  1;
    
      preopt_cache_entry_t entries[];
  
      inline int capacity() const {
          return mask + 1;
      }
  };
  ```

- `add	x16, x16, x9` 则对照了 `(Class)((uintptr_t)cls() + preopt_cache()->fallback_class_offset)` ，即 `cls + fallback_class_offset`



### CacheHit

调用方式 `CacheHit \Mode` ，`\Mode` 参数源自 `CacheLookup Mode, Function, MissLabelDynamic, MissLabelConstant` 的首个参数，由于只存在一个入参，因此在源码中使用 `$0 = \Mode` 

#### 汇编源码

`.macro CacheHit` 对应宏代码块

```assembly
#define NORMAL 0
#define GETIMP 1
#define LOOKUP 2

// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
.macro CacheHit
.if $0 == NORMAL
	TailCallCachedImp x17, x10, x1, x16	// authenticate and call imp
.elseif $0 == GETIMP
	mov	p0, p17
	cbz	p0, 9f			// don't ptrauth a nil imp
	AuthAndResignAsIMP x0, x10, x1, x16	// authenticate imp and re-sign as IMP
9:	ret				// return IMP
.elseif $0 == LOOKUP
	// No nil check for ptrauth: the caller would crash anyway when they
	// jump to a nil IMP. We don't care if that jump also fails ptrauth.
	AuthAndResignAsIMP x17, x10, x1, x16	// authenticate imp and re-sign as IMP
	cmp	x16, x15
	cinc	x16, x16, ne			// x16 += 1 when x15 != x16 (for instrumentation ; fallback to the parent class)
	ret				// return imp via x17
.else
.abort oops
.endif
.endmacro
```

#### NORMAL

`NORMAL`  参数类型下执行 `TailCallCachedImp` 命令

##### TailCallCachedImp指令

当前指令代码

```assembly
	// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
	TailCallCachedImp x17, x10, x1, x16	// authenticate and call imp
```

###### 宏源码

源码全局搜索 `TailCallCachedImp` ，在 `arm64-asm.h ` 中找到对应实现

```assembly
//arm64-asm.h中
#if __has_feature(ptrauth_calls)
// JOP [拓展6]
//...省略其他

.macro TailCallCachedImp
	// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa
	eor	$1, $1, $2	// mix SEL into ptrauth modifier
	eor	$1, $1, $3  // mix isa into ptrauth modifier
	brab	$0, $1
.endmacro

// JOP
#else
// not JOP
//...省略其他

.macro TailCallCachedImp
	// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa
	eor	$0, $0, $3
	br	$0
.endmacro

// not JOP
#endif
```

对应参数

```C++
// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa

$0 = x17, $1 = x10, $2 = x1, $3 = x16
```

###### ptrauth_calls 判断条件下

- `eor`  异或指令 [拓展7]

  ```assembly
  	eor	$1, $1, $2	// buckets ^ SEL
  	eor	$1, $1, $3  // buckets ^ isa
  	
  	// x10 = buckets ^ SEL ^ isa
  ```

- `brab` 指令 [引用 2-6] ,  `brab	$0, $1`

  ```tex
  //释义
  Branch to Register, with pointer authentication. This instruction authenticates the address in the general-purpose register that is specified by <Xn>, using a modifier and the specified key, and branches to the authenticated address.
  The modifier is:
  • In the general-purpose register or stack pointer that is specified by <Xm|SP> for BRAA and BRAB.
  • The value zero, for BRAAZ and BRABZ.
  Key A is used for BRAA and BRAAZ, and key B is used for BRAB and BRABZ.
  If the authentication passes, the PE continues execution at the target of the branch. If the authentication fails, a Translation fault is generated.
  The authenticated address is not written back to the general-purpose register.
  ```

  - 与 `br` 相同，都进行 `Branch to Register` ，同时最终 `branches to the authenticated address` 分支跳转认证过的寄存器地址
  - 跳转后 `The authenticated address is not written back to the general-purpose register`  并不把认证过的地址写入目标寄存器
  - 使用 `x10` 作为 `modifier` 修饰参数进行指针认证

###### 非ptrauth_calls 判断条件下

```assembly
	eor	$0, $0, $3  // imp ^ isa
	br	$0          // branch to imp
```

###### 对照方法

 可对照 `objc-runtime-new.h` 中的代码理解

```C++
    // Compute the ptrauth signing modifier from &_imp, newSel, and cls.
    uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
        return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;
    }

    inline IMP imp(UNUSED_WITHOUT_PTRAUTH bucket_t *base, Class cls) const {
        uintptr_t imp = _imp.load(memory_order_relaxed);
        if (!imp) return nil;
#if CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_PTRAUTH
        SEL sel = _sel.load(memory_order_relaxed);
        return (IMP)
            ptrauth_auth_and_resign((const void *)imp,
                                    ptrauth_key_process_dependent_code,
                                    modifierForSEL(base, sel, cls),
                                    ptrauth_key_function_pointer, 0);
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_ISA_XOR
        return (IMP)(imp ^ (uintptr_t)cls);
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_NONE
        return (IMP)imp;
#else
#error Unknown method cache IMP encoding.
#endif
    }
```

- `CACHE_IMP_ENCODING_PTRAUTH` 判断条件下

  - 进行的 `ptrauth_auth_and_resign()`  函数 (参考《8-1》中的当前函数分析) 对应的宏定义 

    ```C++
    #define ptrauth_auth_and_resign(__value, __old_key, __old_data, __new_key, __new_data) __value
    ```

  - 其参数 `__old_data` 取值 `modifierForSEL(base, newSel, cls)` 对应的实现方法进行了 `(uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;`  与 当前的 `x10 = buckets ^ SEL ^ isa` 逻辑相同，可理解为先对 `imp` 进行解码，之后再经过 `pointer authentication` 之后，只是最后指令进行跳转调用而不是如当前方法一样进行了返回

- `CACHE_IMP_ENCODING_ISA_XOR` 判断条件下，直接 `(uintptr_t)newImp ^ (uintptr_t)cls` 进行解码后调用

###### 总结

对应 `authenticate and call imp` 

#### GETIMP

```assembly
	mov	p0, p17     
	cbz	p0, 9f			// don't ptrauth a nil imp
	AuthAndResignAsIMP x0, x10, x1, x16	// authenticate imp and re-sign as IMP
9:	ret				// return IMP
```

- `mov	p0, p17`  `p0` 存入 `p17` 的数据
- `cbz	p0, 9f` 参考[拓展2]，如果 `p0 == 0` 则向下跳转方法块 9，直接 `return IMP`，对照注释 `don't ptrauth a nil imp`

##### AuthAndResignAsIMP 指令

这指令名字就对照了之前 `对照方法` 中 `ptrauth_auth_and_resign()` 函数，来详细查看下

由 `cbz p0, 9f` 延伸，当前 `p0 != 0` ，且根据注释可知，当前指令目的是进行 `ptrauth imp`

###### 宏源码

源码全局搜索 `AuthAndResignAsIMP ` ，在 `arm64-asm.h ` 中找到对应实现

```assembly
//arm64-asm.h中
#if __has_feature(ptrauth_calls)
// JOP [拓展6]
//...省略其他

.macro AuthAndResignAsIMP
	// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa
	// note: assumes the imp is not nil
	eor	$1, $1, $2	// mix SEL into ptrauth modifier
	eor	$1, $1, $3  // mix isa into ptrauth modifier
	autib	$0, $1	// authenticate cached imp
	ldr	xzr, [$0]	// crash if authentication failed
	paciza	$0		// resign cached imp as IMP
.endmacro

// JOP
#else
// not JOP
//...省略其他

.macro AuthAndResignAsIMP
	// $0 = cached imp, $1 = address of cached imp, $2 = SEL
	eor	$0, $0, $3
.endmacro

// not JOP
#endif
```

对应参数

```C++
// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
// $0 = cached imp, $1 = address of cached imp, $2 = SEL, $3 = isa
// note: assumes the imp is not nil  // 非空值的判断提示多次

$0 = x17, $1 = x10, $2 = x1, $3 = x16
```

###### ptrauth_calls 判断条件下

- `eor`  异或指令 [拓展7]  与 `TailCallCachedImp`指令相同

  ```assembly
  	eor	$1, $1, $2	// buckets ^ SEL
  	eor	$1, $1, $3  // buckets ^ isa
  	
  	// $1 = buckets ^ SEL ^ isa
  ```

- `autib` 指令 [引用 2-7] 与 `autdb`指令[引用2-1] 对 `Data address` 的认证不同，当前针对的是 `Instruction address` 

  ```tex
  Authenticate Instruction address, using key B. This instruction authenticates an instruction address, using a modifier and key B.
  The address is:
  • In the general-purpose register that is specified by <Xd> for AUTIB and AUTIZB.
  ...
  The modifier is:
  • In the general-purpose register or stack pointer that is specified by <Xn|SP> for AUTIB.
  ...
  If the authentication passes, the upper bits of the address are restored to enable subsequent use of the address. If the authentication fails, the upper bits are corrupted and any subsequent use of the address results in a Translation fault.
  ```

  简单来说还是注释的语句 `autib	$0, $1	// authenticate cached imp` 

- `ldr	xzr, [$0]	// crash if authentication failed`

  - `xzr` 参考[引用2-8] 是 `64位 zero register`
  -  `[]` 存储器寻址， `xzr` 赋值为 `authenticate cached imp` ，由于 `autib` 指令存在 `If the authentication fails, the upper bits are corrupted and any subsequent use of the address results in a Translation fault` 特性，因此如果 `authentication failed` 时，此时 `ldr` 指令出错，根据注释是进行了 `crash` 

- `paciza	$0		// resign cached imp as IMP`  参考[引用2-4]

  ```tex
  Pointer Authentication Code for Instruction address, using key A.This instruction computes and inserts a pointer authentication code for an instruction address, using a modifier and key A.
  ```

  对指令地址进行计算和插入 `PAC` 

###### 非ptrauth_calls 判断条件下

```assembly
	eor	$0, $0, $3  // imp ^ isa
```

###### 对照方法

 可对照 `objc-runtime-new.h` 中的代码理解，当前指令比 `TailCallCachedImp` 指令更贴合此对照方法

```C++
    // Compute the ptrauth signing modifier from &_imp, newSel, and cls.
    uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
        return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;
    }

    inline IMP imp(UNUSED_WITHOUT_PTRAUTH bucket_t *base, Class cls) const {
        uintptr_t imp = _imp.load(memory_order_relaxed);
        if (!imp) return nil;
#if CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_PTRAUTH
        SEL sel = _sel.load(memory_order_relaxed);
        return (IMP)
            ptrauth_auth_and_resign((const void *)imp,
                                    ptrauth_key_process_dependent_code,
                                    modifierForSEL(base, sel, cls),
                                    ptrauth_key_function_pointer, 0);
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_ISA_XOR
        return (IMP)(imp ^ (uintptr_t)cls);
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_NONE
        return (IMP)imp;
#else
#error Unknown method cache IMP encoding.
#endif
    }
```

- `CACHE_IMP_ENCODING_PTRAUTH` 判断条件下

  - 进行的 `ptrauth_auth_and_resign()`  函数 (参考《8-1》中的当前函数分析) 对应的宏定义 

    ```C++
    #define ptrauth_auth_and_resign(__value, __old_key, __old_data, __new_key, __new_data) __value
    ```

  - 其参数 `__old_data` 取值 `modifierForSEL(base, newSel, cls)` 对应的实现方法进行了 `(uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;`  与 当前的 `$1 = x10 = buckets ^ SEL ^ isa` 逻辑相同，可理解为先对 `imp` 进行解码，之后再经过 `pointer authentication` 

  - 最后 `return` 结果即 `x0` 

- `CACHE_IMP_ENCODING_ISA_XOR` 判断条件下，直接 `(uintptr_t)newImp ^ (uintptr_t)cls` 进行解码后返回

###### 总结

对应 `authenticate and return imp` 

#### LOOKUP

```assembly
	// No nil check for ptrauth: the caller would crash anyway when they
	// jump to a nil IMP. We don't care if that jump also fails ptrauth.
	AuthAndResignAsIMP x17, x10, x1, x16	// authenticate imp and re-sign as IMP
	cmp	x16, x15
	cinc	x16, x16, ne			// x16 += 1 when x15 != x16 (for instrumentation ; fallback to the parent class)
	ret				// return imp via x17
```

对应参数

```C++
// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
```

`AuthAndResignAsIMP` 指令同 `GETIMP` 分支中所述，区别为当前最终赋值结果给了  `x17` 寄存器

##### cinc	x16, x16, ne

`cmp x16 x15`  比较 `x16` 与 `x15` ，判断条件在之后 `cinc` 指令中

###### cinc 指令

参考[引用2-8]

```tex
示例：CINC <Xd>, <Xn>, <cond>

Conditional Increment returns, in the destination register, the value of the source register incremented by 1 if the condition is TRUE, and otherwise returns the value of the source register.
This instruction is an alias of the CSINC instruction. 
```

###### 解析

`cinc	x16, x16, ne`  

- 判断条件为 `ne = not equal` 
- 指令行为等同于注释代码 `x16 += 1 when x15 != x16`

为何要执行此操作呢？

- `x15` 当前存储的是 `mov	x15, x16 // stash the original isa` 原始的 `isa`

- 那么 `x16` 是在何时进行的改变呢？ 

  在 `CacheLookup` 相关代码内搜索，在 `LLookupPreopt\Function` 中的逻辑找到

  ```assembly
  5:	ldursw	x9, [x10, #-8]			// offset -8 is the fallback offset
  	add	x16, x16, x9			// compute the fallback isa
  	b	LLookupStart\Function		// lookup again with a new isa
  ```

  即此时执行了 `cache_t::preoptFallbackClass()` 方法， `cls + fallback_class_offset` 对 `cls` 进行了偏移

###### warning  此处不太清晰

不太确认此指令是否是针对上述这个变动，存疑！  `x16 += 1` 能否把 `preoptFallbackClass` 转变为原始 `cls` ? 

注释中的 `fallback to the parent class` 指的是当前逻辑么，还是只是单纯地表示退回到 `parent class` ?

猜测是把 `x16` 寄存器退回到原 `isa` ，有可能后续某些操作要使用

##### return imp via x17

最后的 `ret` 指令，根据注释理解的话应该是此时经由 `x17` 寄存器获取 `AuthAndResignAsIMP` 后的 `imp`



## 参考

### 参考1 cache_t中对照代码 和 PREOPT_CACHES 的相关参数

汇编语句和 `cache_t` 结构体中的 `CACHE_MASK_STORAGE_HIGH_16` 代码形成对照 

```C++
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    // _bucketsAndMaybeMask is a buckets_t pointer in the low 48 bits
    // _maybeMask is unused, the mask is stored in the top 16 bits.

    // How much the mask is shifted by.
    static constexpr uintptr_t maskShift = 48;

    // Additional bits after the mask which must be zero. msgSend
    // takes advantage of these additional bits to construct the value
    // `mask << 4` from `_maskAndBuckets` in a single instruction.
    static constexpr uintptr_t maskZeroBits = 4;

    // The largest mask value we can store.
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;
    
    // The mask applied to `_maskAndBuckets` to retrieve the buckets pointer.
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;
    
    // Ensure we have enough bits for the buckets pointer.
    static_assert(bucketsMask >= MACH_VM_MAX_ADDRESS,
            "Bucket field doesn't have enough bits for arbitrary pointers.");

#if CONFIG_USE_PREOPT_CACHES
    static constexpr uintptr_t preoptBucketsMarker = 1ul;
#if __has_feature(ptrauth_calls)
    // 63..60: hash_mask_shift
    // 59..55: hash_shift
    // 54.. 1: buckets ptr + auth
    //      0: always 1
    static constexpr uintptr_t preoptBucketsMask = 0x007ffffffffffffe;
    static inline uintptr_t preoptBucketsHashParams(const preopt_cache_t *cache) {
        uintptr_t value = (uintptr_t)cache->shift << 55;
        // masks have 11 bits but can be 0, so we compute
        // the right shift for 0x7fff rather than 0xffff
        return value | ((objc::mask16ShiftBits(cache->mask) - 1) << 60);
    }
#else
    // 63..53: hash_mask
    // 52..48: hash_shift
    // 47.. 1: buckets ptr
    //      0: always 1
    static constexpr uintptr_t preoptBucketsMask = 0x0000fffffffffffe;
    static inline uintptr_t preoptBucketsHashParams(const preopt_cache_t *cache) {
        return (uintptr_t)cache->hash_params << 48;
    }
#endif
#endif // CONFIG_USE_PREOPT_CACHES
```

针对 `PREOPT_CACHES` 的相关参数

#### preoptBucketsMask

`preoptBucketsMask`  用来获取 `preopt_cache_t * ` 

```C++
const preopt_cache_t *cache_t::preopt_cache() const
{
    auto addr = _bucketsAndMaybeMask.load(memory_order_relaxed);
    addr &= preoptBucketsMask;
#if __has_feature(ptrauth_calls)
#if __BUILDING_OBJCDT__
    addr = (uintptr_t)ptrauth_strip((preopt_cache_entry_t *)addr,
            ptrauth_key_process_dependent_data);
#else
    addr = (uintptr_t)ptrauth_auth_data((preopt_cache_entry_t *)addr,
            ptrauth_key_process_dependent_data, (uintptr_t)cls());
#endif
#endif
    return (preopt_cache_t *)(addr - sizeof(preopt_cache_t));
}
```

此处的 `addr == preoptBuckets 等同于 entries `  

#### preoptBucketsHashParams

`preoptBucketsHashParams`  函数用来组装 `hash_params` 

参考 `preopt_cache_t` 结构体的成员 

```C++
struct preopt_cache_t {
    int32_t  fallback_class_offset;
    union {
        struct {
            uint16_t shift       :  5;
            uint16_t mask        : 11;
        };
        uint16_t hash_params;
    };
    uint16_t occupied    : 14;
    uint16_t has_inlines :  1;
    uint16_t bit_one     :  1;
    preopt_cache_entry_t entries[];

    inline int capacity() const {
        return mask + 1;
    }
};
```

根据 `union` 联合体可知 `shift` 与 `mask` 组合起来等同于 `hash_params`

`preoptBucketsHashParams` 函数就针对不同架构下的 `buckets` 中的 `hash_params` 的数据进行组装然后进行位移到对应的位置

其中调用的函数 `mask16ShiftBits` 其实现为

```C++
namespace objc {
static inline uintptr_t mask16ShiftBits(uint16_t mask)
{
    // returns by how much 0xffff must be shifted "right" to return mask
    uintptr_t maskShift = __builtin_clz(mask) - 16; //__builtin_clz 统计前导零个数[参考5]
    ASSERT((0xffff >> maskShift) == mask);
    return maskShift;
}
}
```

其对 `mask` 的偏移量 `maskShift` 进行存储

在使用 `0xffff` 进行 `mask16ShiftBits` 时，存在如下规则

- 由于 `__builtin_clz(mask)` 在 `mask = 1` 时，最大值为 `31`   (`mask = 0` 时 `undefined`)[参考5] ，则 `maskShift <= 15` 
- 由位域 `uint16_t mask : 11;` 可知 `mask` 最多 `11` 位，则 `__builtin_clz(mask) >= 21` ，可得 `maskShift >= 5`

对应理解其注释语句

```C++
    // masks have 11 bits but can be 0, so we compute
    // the right shift for 0x7fff rather than 0xffff
```

使用 `0x7fff` 时

- 在 `maskShift = 15` 时，得到 `mask = 0x7fff >> 15 = 0` ，同时在 `maskShift = 14` 时，得到 `mask = 0x7fff >> 14 = 1` ，因此修改成 `0x7fff` 不影响从最小值开始获取，同时还可以附加 0 这个值；
- 针对最大边界 `mask = 0x7ff` 时， ` 0x7fff >> 4 = 0x7ff` ，也可满足实现，因此有了注释的语句，此时使用 `0x7fff` 来进行编码和解码更合适

#### maybeConvertToPreoptimized

`maybeConvertToPreoptimized` 对照 `preoptBuckets` 的成员组成，使用 `preopt_cache_t` 的数据来组装 `preoptBuckets` 最终进行存储 `_bucketsAndMaybeMask.store`

```C++
void cache_t::maybeConvertToPreoptimized()
{
    const preopt_cache_t *cache = disguised_preopt_cache();

    if (cache == nil) {
        return;
    }

    if (!cls()->allowsPreoptCaches() ||
            (cache->has_inlines && !cls()->allowsPreoptInlinedSels())) {
        if (PrintCaches) {
            _objc_inform("CACHES: %sclass %s: dropping cache (from %s)",
                         cls()->isMetaClass() ? "meta" : "",
                         cls()->nameForLogging(), "setInitialized");
        }
        return setBucketsAndMask(emptyBuckets(), 0);
    }

    uintptr_t value = (uintptr_t)&cache->entries;
#if __has_feature(ptrauth_calls)
    value = (uintptr_t)ptrauth_sign_unauthenticated((void *)value,
            ptrauth_key_process_dependent_data, (uintptr_t)cls());
#endif
    value |= preoptBucketsHashParams(cache) | preoptBucketsMarker;
    _bucketsAndMaybeMask.store(value, memory_order_relaxed);
    _occupied = cache->occupied;
}
```

- `preoptBuckets` 的组装逻辑核心代码为

  ```C++
  uintptr_t value = (uintptr_t)&cache->entries;
  value |= preoptBucketsHashParams(cache) | preoptBucketsMarker
  ```

  组装逻辑解析得 `hash_param | entries | preoptBucketsMarker` 与 `preoptBuckets` 的结构对照一致

  `static constexpr uintptr_t preoptBucketsMarker = 1ul;  // ul 无符号长整型`

  ```C++
      // 63..53: hash_mask
      // 52..48: hash_shift
      // 47.. 1: buckets ptr
      //      0: always 1
  ```

- 同时此逻辑也验证了 `preoptBuckets !==  preopt_cache_t`  

  不同于其他架构下 `cahce_t & mask` 得到 `bucket_t` ，在使用 `preoptBucketsMask` 掩码获取的是 `preoptBuckets` 不是 `preopt_cache_t` ，其使用 `preopt_cache_t` 的数据进行了拼装



### 参考2 PTRSHIFT

`arm64-asm.h` 文件中，在 `__arm64__ & __LP64__` 架构下，`PTRSHIFT` 为 3

```assembly
#if __arm64__

#include "objc-config.h"

#if __LP64__
// true arm64

#define SUPPORT_TAGGED_POINTERS 1
#define PTR .quad
#define PTRSIZE 8
#define PTRSHIFT 3  // 1<<PTRSHIFT == PTRSIZE

//省略后续 和 arm64_32架构下的判断
```



### 参考3 buchet_t

查看 `buchet_t` 的结构

```c++
struct bucket_t {
private:
    // IMP-first is better for arm64e ptrauth and no worse for arm64.
    // SEL-first is better for armv7* and i386 and x86_64.
#if __arm64__
    explicit_atomic<uintptr_t> _imp;
    explicit_atomic<SEL> _sel;
#else
    explicit_atomic<SEL> _sel;
    explicit_atomic<uintptr_t> _imp;
#endif

    // Compute the ptrauth signing modifier from &_imp, newSel, and cls.
    uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
        return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;
    }
```

其只由 `uintptr_t _imp` 和 `SEL _sel`  ，分析其大小

- 查看声明 `typedef unsigned long uintptr_t;` ，其类型 `unsigned long` 在 64位系统大小为 8
- 根据下面的方法中 `(uintptr_t)newSel` 进行的强转类型，可知 `SEL` 的大小一定与 `uintptr_t` 一致
- 汇总可知 `buchet_t` 的大小为 16

同时参考之前输出结果

> 参考 《6-3、cache_t源码分析》- reallocate() 小节

```shell
(lldb) p sizeof(bucket_t)
(unsigned long) $31 = 16
```

### 参考4  ARM64真机环境 setBucketsAndMask 函数实现

```C++
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16 || CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS

void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask)
{
    uintptr_t buckets = (uintptr_t)newBuckets;
    uintptr_t mask = (uintptr_t)newMask;

    ASSERT(buckets <= bucketsMask);
    ASSERT(mask <= maxMask);

    _bucketsAndMaybeMask.store(((uintptr_t)newMask << maskShift) | (uintptr_t)newBuckets, memory_order_relaxed);
    _occupied = 0;
}

mask_t cache_t::mask() const
{
    uintptr_t maskAndBuckets = _bucketsAndMaybeMask.load(memory_order_relaxed);
    return maskAndBuckets >> maskShift;
}
```



### 参考5 __builtin_clz()

`__builtin_clz()` 为 GCC 内建函数，clz表示 `Count Leading Zeros`，统计前导零个数

官方文档释义：

```tex
Built-in Function: int __builtin_clz (unsigned int x)

    Returns the number of leading 0-bits in x, starting at the most significant bit position. If x is 0, the result is undefined. 
```

官方文档：[6.59 Other Built-in Functions Provided by GCC](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)

协助理解文章参考：

[理解__builtin_clz特性](https://www.cnblogs.com/liyou-blog/p/4191146.html)

[如何得到一个二进制数的最高有效位](https://www.zhihu.com/question/35361094) 中的回答



### 参考6 preopt_cache_t 获取方法

两种获取 `preot_cahce_t *` 数据的方法

```C++
//方式1
const preopt_cache_t *cache_t::preopt_cache() const
{
    auto addr = _bucketsAndMaybeMask.load(memory_order_relaxed);
    addr &= preoptBucketsMask;
#if __has_feature(ptrauth_calls)
#if __BUILDING_OBJCDT__
    addr = (uintptr_t)ptrauth_strip((preopt_cache_entry_t *)addr,
            ptrauth_key_process_dependent_data);
#else
    addr = (uintptr_t)ptrauth_auth_data((preopt_cache_entry_t *)addr,
            ptrauth_key_process_dependent_data, (uintptr_t)cls());
#endif
#endif
    return (preopt_cache_t *)(addr - sizeof(preopt_cache_t));
}

//方式2
const preopt_cache_t *cache_t::disguised_preopt_cache() const
{
    bucket_t *b = buckets();
    if ((intptr_t)b->sel() >= 0) return nil;

    uintptr_t value = (uintptr_t)b + bucket_t::offsetOfSel() + sizeof(SEL);
    return (preopt_cache_t *)(value - sizeof(preopt_cache_t));
}
```





## 拓展

### 拓展1 CMP 指令

CMP 指令是 Comparison 类型的指令，其描述为

` CMP W3, W4     // Set flags based on W3 - W4` [引用 1-2]

在结合其后的 `b` 指令[引用 1-3] 进行跳转时，根据其后的判断代码(Condition codes)[引用 1-4] 决定判断条件

### 拓展2 CBZ 指令

CBZ 指令归属于 `Conditional branch instructions` ，其描述为

` CBZ Rt, label        // Compare and branch if zero. If Rt is not zero, branch forward or back up to 1MB.` [引用 1-5]

### 拓展3 UXTW 指令

`UXTW/UXTH/UXTB : Zero-extend single-word/half-word/byte`

```tex
Zero-extend
Int is a 32-bit value, char is a 16-bit value. Zero-extend just means that the higher-order "unused" bits in the int are zeroes
即 补零
```

参考： [What does "Zero-extend" mean?](https://stackoverflow.com/questions/42171285/what-does-zero-extend-mean/42171319)

### 拓展4 CCMP 指令

`Conditional Compare (register) sets the value of the condition flags to the result of the comparison of two registers if the condition is TRUE, and an immediate value otherwise.` [引用 2-2]

```assembly
CCMP Wn, #uimm5, #uimm4, cond
Conditional Compare (immediate):
NZCV = if cond then CMP(Wn,uimm5) else uimm4.
```

参考：[逻辑表达式的短路是如何实现的？](https://www.zhihu.com/question/53273670?from=profile_question_card) 中的 `RednaxelaFX` 的回答

### 拓展5 ADRP指令

[引用 2-3]中的指令描述

`Form PC-relative address to 4KB page adds an immediate value that is shifted left by 12 bits, to the PC value to form a PC-relative address, with the bottom 12 bits masked out, and writes the result to the destination register.`

`ADRP <Xd>, <label>`

计算当前 pc 寄存器指向地址所处的内存页，由于内存页大小 4KB(0x1000) 因此将 低12位置为0，可得到当前内存页的起始地址；再加上立即数( imm ) 进行左移 12 位后的值 (即 偏移到 `<label>` 所在页)，得到的偏移后所在内存页的首地址写入 `<xd>` 寄存器

- imm 解释：立即数 `imm = SignExtend(immhi:immlo:Zeros(12), 64);`

  immhi:immlo 释义： immhi(immediate value high 立即数高位) 和 immlo(immediate value  low立即数低位) 一起是21位，1位是符号位（往前/后跳），剩下20位表示1MB（2的10次方=1KB，2的20次方=1MB...），immhi:immlo 按照高低顺序形成立即数

adrp将12个较低位归零并相对于当前PC页面偏移，所以我们可以寻找+/-4GB的相对地址，代价是adrp指令后面，要跟着add指令去设置较低的12位；adrp指令将21位立即数左移12位，将其和PC(程序计数器)相加，最后较低12位清零，然后将结果写入通用寄存器。这允许计算4KB对齐的存储区域的地址。 结合add指令，设置较低12位，可以计算或访问当前PC的±4GB范围内的任何地址

释义参考 [iOS程序员的自我修养-MachO文件动态链接](https://juejin.cn/post/6844903922654511112)

- 示例：

  ```tex
  以如下指令为例：
  ffff000011c30044:       f0003260        adrp    x0, ffff00001227f000 <init_pg_dir>
  
  查看符号表
  $ cat System.map | grep init_pg_dir
  ffff00001227f000 B init_pg_dir
  
  解析：
  ffff000011c30044 低12位置0可得 PC寄存器内存页基地址 ffff000011c30000
  
  偏移量计算
  f0003260 机器码 0b'1111 0000 0000 0000 0011 0010 0110 0000
  immlo 为 bit30:bit29 对应值为 1:1
  immhi 为 bit23~bit5  对应值为 0000 0000 0011 0010 011
  所以 immhi:immlow 为21位 0b'000000000011001001111
  imm = SignExtend(immhi:immlo:Zeros(12), 64)，左移12位为 0b'000000000011001001111000000000000 == 0x'64f000
  
  ffff000011c30000 + 0x'64f000 = ffff00001227f000 
  此地址即 <init_pg_dir> 指向地址所在分页的基地址
  ```

  参考链接：[ARM指令解析之ADRP](https://blog.csdn.net/shipinsky/article/details/123321451)

- 参考知识

  ```tex
  内存分页机制：将虚拟内存空间和物理内存空间划分为大小相同的页，并以页作为内存空间划分的最小单位。空间增长也容易实现：只需要分配额外的虚拟页面，并找到一个闲置的物理页面存放即可。一个内存页大小为PAGE_SIZE字节，在不同的平台PAGE_SIZE不同。例如：Mac OS中，在终端通过PAGESIZE命令可获取内存页大小为：4096(4K)。而在Aarch64 (Arm64) Linux 系统上的内存页配置经常是64KB
  ```

  参考链接：[04 - ADPR指令&cmp指令&switch汇编](https://www.jianshu.com/p/592da10cd7af)

  

### 拓展6 JOP

jop，全称 `Jump-Oriented Programming`，中文译为面向跳转编程，是代码重用攻击方式的一种

```tex
In a JOP-based
attack, the attacker abandons all reliance on the stack for
control flow and ret for gadget discovery and chaining, in-
stead using nothing more than a sequence of indirect jump
instructions. 
```

参考文档：

[JOP代码复用攻击](https://zhuanlan.zhihu.com/p/39695776)

[Jump-Oriented Programming: A New Class of Code-Reuse Attack](https://www.comp.nus.edu.sg/~liangzk/papers/asiaccs11.pdf)

### 拓展7 EOR 指令

详细指令文档信息参考 [引用2-5]

逻辑异或 `Exclusive OR` ，按位异或，相同为0，不同为1



## 引用

### 引用1 指令

文档 `《ARM Cortex-A Series Programmer's Guide for ARMv8-A》`

#### 引用 1-1 LDP

`6.3.5 Accessing multiple memory locations` 小节 以及 `Table 6-11 Register Load/Store pair`

![8-2-1](8-2、objc_msgSend原理汇编分析.assets/8-2-1.png)

#### 引用 1-2 CMP

`6.2.1 Arithmetic and logical operations` 小节 以及 `Table 6-1 Arithmetic and logical operations` 与 `Example 6-1 Arithmetic instructions` 

![8-2-2](8-2、objc_msgSend原理汇编分析.assets/8-2-2.png)



#### 引用 1-3 B

`6.4 Flow control` 小节 以及 `Table 6-12 Branch instructions`

![8-2-3](8-2、objc_msgSend原理汇编分析.assets/8-2-3.png)

#### 引用 1-4 Condition codes

`6.2.5 Conditonal instructions` 小节 `Table 6-5 Condition codes` 

![8-2-4](8-2、objc_msgSend原理汇编分析.assets/8-2-4.png)

#### 引用 1-5 CBZ

`6.4 Flow control` 小节以及 `Table 6-12 Branch instructions` 中下半部分 `Conditional branch instructions`

![8-2-5](8-2、objc_msgSend原理汇编分析.assets/8-2-5.png)



### 引用2 指令

文档 `《Arm® Architecture Reference Manual for A-profile architecture》`

#### 引用2-1  AUTDB

![8-2-6](8-2、objc_msgSend原理汇编分析.assets/8-2-6.png)

#### 引用2-2 CCMP

![8-2-7](8-2、objc_msgSend原理汇编分析.assets/8-2-7.png)



#### 引用2-3 ADRP指令

 ![8-2-8](8-2、objc_msgSend原理汇编分析.assets/8-2-8.png)



#### 引用2-4 PACIZA 指令

![8-2-9](8-2、objc_msgSend原理汇编分析.assets/8-2-9.png)



#### 引用2-5 EOR指令

![8-2-10](8-2、objc_msgSend原理汇编分析.assets/8-2-10.png)



#### 引用2-6 brab指令

`C6.2.38`

![8-2-11](8-2、objc_msgSend原理汇编分析.assets/8-2-11.png)

#### 引用2-7 AUTIB

![8-2-12](8-2、objc_msgSend原理汇编分析.assets/8-2-12.png)

#### 引用2-8 CINC

![8-2-14](8-2、objc_msgSend原理汇编分析.assets/8-2-14.png)



### 引用3 Register Name

 #### 引用3-1 XZR

![8-2-13](8-2、objc_msgSend原理汇编分析.assets/8-2-13.png)





### End

