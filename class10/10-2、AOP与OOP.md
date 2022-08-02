# AOP与OOP

### 引言

上章节我们通过在动态方法决议方法解决方法找不到的问题，这种思维方式引出了 `AOP` 即 `Aspect Oriented Programming` 面向切面编程，此篇进行稍微的了解和扩展

## OOP

`AOP` 作为 `OOP(Object Oriented Programming)`  面向对象编程的一种延伸

对 `OOP` 来说

- 对象分工明确，面向对象开发

- 由于每个对象都要实现各自的功能，所以代码耦合度低，但是对同一个功能每个对象都进行实现因此存在冗余代码

- 为了解决此部分冗余，进行提取公共类，此时所有类都对公共类继承，这个继承关系又造成了强依赖强耦合，这又违背了高内聚低耦合的架构设计

- 此时就又引入了一种动态的无侵入的注入代码的方式来解决这个问题，就像 《10-1》中的在 `NSObject` 的类目中添加 `resolveInstanceMethod` 方式来解决方法找不到的问题，这种解决方式不影响原本类中方法的实现
  - 这个无需导入的 `NSObject` 类目，在运行流程中会执行到，无需侵入改变什么，这就形成了一个切面，切入到程序的运行流程中
  - 切入的方法和切入的类可以称作切点

## AOP

`AOP` 是 `Spring` 框架的一个重要内容

通俗描述：

把 `OOP` 看作二维的平面，`AOP` 是在平面上添加 Z轴变为立体的，对 `OOP` 所在平面切入，由于是另一个维度并不影响 `OOP` 本身原来的运行流程

其他相关

- `hook` 方式体现了 `AOP`
- 协议是 `AOP` 的动态代理体现，本质的思想就是 `AOP` 
- 举例： `resolveInstanceMethod` 动态方法决议
  - 在 `NSObject` 类目中添加 `resolveInstanceMethod` 类方法，在全部方法的查找流程上，切入
  - 这个 `NSObject` 类目写在业务代码外，无需在业务代码的类中间添加
  - 切点就是 `resolveInstanceMethod` 针对某一方法例如 `sayHappy` 增加的判断和处理

### 缺点

上面根据 `AOP` 的特性解决问题，展现了其优点，那么缺点呢

- 在切面切入时，不仅对程序员开发的类进行切入，对系统类也进行了切入，所以也捕捉到了系统类的中的对应条件问题

  - 示例中，在 《10-1》中 `resolveInstanceMethod` 的代码运行时，就存在大量系统方法的日志

    ```shell
     +[NSObject(LG) resolveInstanceMethod:]-__NSCFString-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-__NSCFString-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-__NSCFString-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-NSObject-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-NSTaggedPointerString-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-NSTaggedPointerString-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-LGTeacher-sayHappy
     +[NSObject(LG) resolveInstanceMethod:]-LGTeacher-encodeWithOSLogCoder:options:maxLength:
     LGTeacher - +[LGTeacher sayKC]
     +[NSObject(LG) resolveInstanceMethod:]-NSTaggedPointerString-encodeWithOSLogCoder:options:maxLength:
     +[NSObject(LG) resolveInstanceMethod:]-LGTeacher-say666
     +[NSObject(LG) resolveInstanceMethod:]-LGTeacher-encodeWithOSLogCoder:options:maxLength:
     <LGTeacher: 0x1007c8730> - -[LGTeacher sayNB]
     Hello, World!
    Program ended with exit code: 0
    ```

    这些 `NSCFString/NSObject/NSTaggedPointerString` 相关的方法也不止打印一次，这也必定会造成性能的消耗

- 在切面的方法中，进行判断来筛选指定方法来操作，这些越加越多的筛选代码也造成了一定的性能消耗

- 在切面方法中进行处理后，可能会影响后续流程的作用

  - 示例，在《10-1》中动态决议时进行切面处理时，其苹果系统流程中在方法查找流程中动态决议后的消息转发流程就没有意义了
  - 因此在进行切面时需要寻找合适的切面进行切入，中间流程的切入就不是太合适