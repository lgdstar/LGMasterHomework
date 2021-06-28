## init 与 new

此节主题是探究下对象在  ` alloc init` 与 `new` 创建时有什么区别

```objc
LGPerson *p1 = [[LGPerson alloc] init];

LGPerson *p2 = [LGPerson new];
```

### 源码分析

#### init

> `alloc` 的源码我们在前面分析过了，在此就不再重复

查看实现

```C++
// NSObject.mm

- (id)init {
    return _objc_rootInit(self);
}

id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

好像只返回了 `self` 对象，没有其他代码。那这个方法添加上有什么作用么？

他是一个初始化方法，构造函数用来给子类进行重写 ，根据各自的功能拓展自己的功能(工厂设计模式 [拓展1])。

` init` 只是提供接口，便于扩展

#### new

查看实现

```c++
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```

- `callAlloc` 是 `alloc` 流程中执行的方法
- 也同时调用了 `init`
- 等同于 alloc init

#### 汇编验证

```assembly
->  0x100003bdf <+31>: movq   0x4722(%rip), %rdi        ; (void *)0x00000001000083a8: LGPerson
    0x100003be6 <+38>: callq  0x100003dd2               ; symbol stub for: objc_alloc_init
    0x100003beb <+43>: movq   %rax, -0x20(%rbp)
    0x100003bef <+47>: movq   0x4712(%rip), %rdi        ; (void *)0x00000001000083a8: LGPerson
    0x100003bf6 <+54>: callq  0x100003df0               ; symbol stub for: objc_opt_new
    0x100003bfb <+59>: movq   %rax, -0x18(%rbp)
```

可看到 `alloc init` 对应的符号是 `objc_alloc_init` ; `new` 对应的是 `objc_opt_new` 

##### objc_alloc_init

```C++
// Calls [[cls alloc] init].
id
objc_alloc_init(Class cls)
{
    return [callAlloc(cls, true/*checkNil*/, false/*allocWithZone*/) init];
}

```

此时是编译阶段，方法首次执行起来到 `callAlloc` 方法中最终进行 `objc_msgSend` 

##### objc_opt_new

```C++
// Calls [cls new]
id
objc_opt_new(Class cls)
{
#if __OBJC2__
    if (fastpath(cls && !cls->ISA()->hasCustomCore())) {
        return [callAlloc(cls, false/*checkNil*/) init];
    }
#endif
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(new));
}
```

首次进入，解析判断条件

```C++
// ---- hasCustomCore
bool hasCustomCore() const {
    return !cache.getBit(FAST_CACHE_HAS_DEFAULT_CORE);
}

// ------   FAST_CACHE_HAS_DEFAULT_CORE
// class or superclass has default new/self/class/respondsToSelector/isKindOfClass
#define FAST_CACHE_HAS_DEFAULT_CORE   (1<<15)

//  cache.getBit   又是这个家伙
  bool getBit(uint16_t flags) const {
      return _flags & flags;
  }
```

此时还是llvm编译阶段， `_flags = nil`  ，所以 `if`  判断条件进不去，因此也直接走 `objc_msgSend` 进行消息转发

#### 汇总

汇编的符号绑定最后都是进行了 `objc_msgSend` ，又找到各自的方法实现里了，通过上面的 `init` 和 `new` 的源码已经验证了方法执行一致。 因此可以认为 `alloc init` 等同于 `new` 

## 拓展

### 拓展1 工厂设计模式

