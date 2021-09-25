---
title: iOS | Method Swizzle
date: 2021-09-24
categories: ['iOS']
draft: false
---

[iOS中Method swizzling的使用](https://www.jianshu.com/p/d4322ce6ae4f)

[掘金](https://juejin.cn/post/6844903856497754126#heading-2)

OC的runtime十分的强大，其中一个灵活的黑魔法就是Method Swizzle。

Swizzle的本质是：在运行时交换方法的实现（IMP）。灵活的使用swizzle，可以达到hook的效果，开发者可以hook住系统的方法或者生命周期，来做一些自定义的操作。

## 原理

Objective-C的消息机制：在 Objective-C 中调用一个方法， 实际上是在底层通过 `objc_msgSend()`发送一个消息。 而查找消息的唯一依据是selector的方法名。

```objectivec
[obj doSomething]; /// => objc_msgSend(obj,@selector(doSomething))

```

每一个OC实例对象都保存有isa指针和实例变量，其中isa指针所属类，类维护一个运行时可接收的方法列表(MethodLists)； 方法列表(MethodLists)中保存selector & IMP的映射关系。在运行时，通过selecter找到匹配的IMP，从而找到的具体的实现函数。

开发中可以利用Objective-C的动态特性，在运行时替换selector对应的方法实现（IMP），达到给hook的目的。下图是利用 Method Swizzle 来替换selector对应IMP后的方法列表示意图。

[Method%20Swizzle%2076aa8a50bd07495cae3a80f89563d2f0/16b06aaf3816730ctplv-t2oaga2asx-watermark.awebp](Method%20Swizzle%2076aa8a50bd07495cae3a80f89563d2f0/16b06aaf3816730ctplv-t2oaga2asx-watermark.awebp)

## 使用时注意

- 在使用swizzle的时候，可以在category中实现
- **swizzling 应该只在 dispatch_once 中完成**

    由于 swizzling 改变了全局的状态，所以我们需要确保每个预防措施在运行时都是可用的。原子操作就是这样一个用于确保代码只会被执行一次的预防措施，就算是在不同的线程中也能确保代码只执行一次。Grand Central Dispatch 的 dispatch_once 满足了所需要的需求，并且应该被当做使用 swizzling 的初始化单例方法的标准。

- **swizzling 应该只在 +load 中完成**

    在 Objective-C 的运行时中，每个类有两个方法都会自动调用。+load 是在一个类被初始装载时调用，+initialize 是在应用第一次调用该类的类方法或实例方法前调用的。两个方法都是可选的，并且只有在方法被实现的情况下才会被调用。

- 要记得调用原来的方法

    下面代码在正常情况下会出现循环：

    ```objectivec
    - (void)xxx_viewWillAppear:(BOOL)animated {
        [self xxx_viewWillAppear:animated];
        NSLog(@"viewWillAppear: %@", NSStringFromClass([self class]));
    }

    ```

    然而在交换了方法实现后就不会出现循环了。好的程序员应该对这里出现的方法的递归调用有所警觉，这里我们应该理清在 method swizzling 后方法的实现究竟变成了什么。在交换了方法的实现后，xxx_viewWillAppear:方法的实现已经被替换为了 UIViewController -viewWillAppear：的原生实现，所以这里并不是在递归调用。由于 xxx_viewWillAppear: 这个方法的实现已经被替换为了 viewWillAppear: 的实现，所以，当我们在这个方法中再调用 viewWillAppear: 时便会造成递归循环。

    ## RSSwizzle

    iOS的swizzle代码并未开源，可以看一个已经开源的库RSSwizzle。它的代码比较少，核心的代码我贴在下面。

    核心的思路是：使用`RSSwizzleInfo`存储需要swizzle的信息。其中包含一个selector和寻找IMP的闭包`originalImpProvider`。查找IMP的方法是查询当前类是否实现了方法，如果没有就查询他的父类，一直查到底。

    这里为了保证线程安全，使用了一个自旋锁。这里额外的多说几句，没有抢到自旋锁的进程会不断重试直到取到锁。因此自旋锁适用于消耗比较小，用时短的工作。但自旋锁并不是完全安全的，它会带来优先级反转的问题。iOS10之后可以使用`os_unfair_lock`。

    [不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

    ```objectivec
    static void swizzle(Class classToSwizzle,
                        SEL selector,
                        RSSwizzleImpFactoryBlock factoryBlock)
    {
        Method method = class_getInstanceMethod(classToSwizzle, selector);
        
        NSCAssert(NULL != method,
                  @"Selector %@ not found in %@ methods of class %@.",
                  NSStringFromSelector(selector),
                  class_isMetaClass(classToSwizzle) ? @"class" : @"instance",
                  classToSwizzle);
        
        NSCAssert(blockIsAnImpFactoryBlock(factoryBlock),
                 @"Wrong type of implementation factory block.");
        
        __block OSSpinLock lock = OS_SPINLOCK_INIT;
        // To keep things thread-safe, we fill in the originalIMP later,
        // with the result of the class_replaceMethod call below.
        __block IMP originalIMP = NULL;

        // This block will be called by the client to get original implementation and call it.
        RSSWizzleImpProvider originalImpProvider = ^IMP{
            // It's possible that another thread can call the method between the call to
            // class_replaceMethod and its return value being set.
            // So to be sure originalIMP has the right value, we need a lock.
            OSSpinLockLock(&lock);
            IMP imp = originalIMP;
            OSSpinLockUnlock(&lock);
            
            if (NULL == imp){
                // If the class does not implement the method
                // we need to find an implementation in one of the superclasses.
                Class superclass = class_getSuperclass(classToSwizzle);
                imp = method_getImplementation(class_getInstanceMethod(superclass,selector));
            }
            return imp;
        };
        
        RSSwizzleInfo *swizzleInfo = [RSSwizzleInfo new];
        swizzleInfo.selector = selector;
        swizzleInfo.impProviderBlock = originalImpProvider;
        
        // We ask the client for the new implementation block.
        // We pass swizzleInfo as an argument to factory block, so the client can
        // call original implementation from the new implementation.
        id newIMPBlock = factoryBlock(swizzleInfo);
        
        const char *methodType = method_getTypeEncoding(method);
        
        NSCAssert(blockIsCompatibleWithMethodType(newIMPBlock,methodType),
                 @"Block returned from factory is not compatible with method type.");
        
        IMP newIMP = imp_implementationWithBlock(newIMPBlock);
        
        // Atomically replace the original method with our new implementation.
        // This will ensure that if someone else's code on another thread is messing
        // with the class' method list too, we always have a valid method at all times.
        //
        // If the class does not implement the method itself then
        // class_replaceMethod returns NULL and superclasses's implementation will be used.
        //
        // We need a lock to be sure that originalIMP has the right value in the
        // originalImpProvider block above.
        OSSpinLockLock(&lock);
        originalIMP = class_replaceMethod(classToSwizzle, selector, newIMP, methodType);
        OSSpinLockUnlock(&lock);
    }
    ```
