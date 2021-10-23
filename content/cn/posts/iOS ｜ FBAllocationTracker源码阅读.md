---
title: iOS ｜ FBAllocationTracker源码阅读
date: 2021-10-18
categories: ['iOS']
draft: false
---

## FBAllocationTracker
FBAllocationTracker 是一个监控组件，作用有两个：
- 追踪对象。使用 FBAllocationTracker 可以得到当前正在活跃的对象类的实例的个数、类名、初始化次数、销毁次数和实例所占的内存大小。
- 统计 alloc 和 dealloc 的次数。

## 文件结构

![](https://raw.githubusercontent.com/LumenAnnex/picHome/master/img/90D1D697-57C1-4167-8F0C-57A5095E3800.png)

FBAllocationTracker代码不多，整体对外暴露的是两个文件：
- FBAllocationTrackerManager。这是统一的 manager，内部使用 FBAllocationTracker 来进行真正的操作。这部分的实现在 FBAllocationTrackerImpl 内部。
- FBAllocationTrackerSummary。这是用于存储结果的对象，通过该类来传递监控的结果。

另外一大块文件是Generations，这也算是FBAllocationTracker的一个特性。可以对所有创建的类进行分代。举个具体的例子，复现了一套用户的操作路径之后，可以手动分一代，这样所有该操作路径里的记录就被划分为同一代，避免和其他的操作冲突。

其余的文件为工具类：
- FBAllocationTrackerFunctors。包含一些运算符的重载，获取到每个类的实例个数一定需要对每个类或者是实例进行比较，这里包含了一些方便的运算符重载。
```c
struct ClassEqualFunctor {
    bool operator()(const Class left, const Class right) const {
      return left == right;
    }
  };
```
- FBAllocationTrackerHelpers。内部定义了两个方法，`incrementAllocations` 和 `incrementDeallocations`。用来对 alloc 和 dealloc 进行计数。
- NSObject+FBAllocationTracker。这里对 NSObject 的 alloc 和 dealloc 进行 swizzle。


## 使用
接口基本定义在 FBAllocationTrackerManager 中，使用很简单
```objectivec
[[FBAllocationTrackerManager sharedManager] startTrackingAllocations];
```
如果要使用分代的功能，那么就在调用一个 `enableGenerations` 的方法即可。

`currentAllocationSummary` 是获取到所有追踪内容的接口。
```objectivec
/**
 Grab summary snapshot for current moment. Every object in this array will have a quick
 snapshot containing how many instances were allocated/deallocated and few more.
 Check FBAllocationTrackerSummary.h.
 */
- (**nullable** NSArray<FBAllocationTrackerSummary *> *)currentAllocationSummary;
```


## 追踪对象的具体实现
接下来看一下追踪对象是如何实现的。
当调用到 `startTrackingAllocations` 时，会一路调用到 `turnOnTracking`。这里面干了两件事：
- 存储原本的 alloc 和 dealloc 方法。存储到 fb_originalAllocWithZone 和 fb_originalDealloc 的 selector 中。这里一定要存储，因为在 tracking 结束的时候还要再换回来。这里用一个标志位 _didCopyOriginalMethods 来避免多次的copy。
```objectivec
  void prepareOriginalMethods(void) {
    if (_didCopyOriginalMethods) {
      return;
    }

    // prepareOriginalMethods called from turnOn/Off which is synced by
    // _lock, this is thread-safe
    _didCopyOriginalMethods = true;

    replaceSelectorWithSelector([NSObject class],
                                @selector(fb_originalAllocWithZone:),
                                @selector(allocWithZone:),
                                FBClassMethod);

    replaceSelectorWithSelector([NSObject class],
                                @selector(fb_originalDealloc),
                                sel_registerName("dealloc"),
                                FBInstanceMethod);
  }
```

- swizzle 系统的 alloc 和 dealloc。
```objectivec
  void turnOnTracking(void) {
    prepareOriginalMethods();

    replaceSelectorWithSelector([NSObject class],
                                @selector(allocWithZone:),
                                @selector(fb_newAllocWithZone:),
                                FBClassMethod);

    replaceSelectorWithSelector([NSObject class],
                                sel_registerName("dealloc"),
                                @selector(fb_newDealloc),
                                FBInstanceMethod);
  }
```

Swizzle 之后先调用了原本的方法，接下来做两件事，这里以 alloc 为例：增加计数 & 添加到 generation 管理器里面。
```objectivec
  void incrementAllocations(__unsafe_unretained id obj) {
    Class aCls = [obj class];

    if (!_shouldTrackClass(aCls)) {
      return;
    }

    std::lock_guard<std::mutex> l(*_lock);

    if (_trackingInProgress) {
      (*_allocations)[aCls]++;
    }

    if (_generationManager) {
      _generationManager->addObject(obj);
    }
  }
```


## Generation的具体实现
分代的功能基本上是用一个大的 Vector 来实现的。在上面的 incrementAllocations 方法里面，调用了 	` _generationManager->addObject(obj);` 方法，这里实际上把 obj 添加到了一个临时的 Generation 里面。 Generation 里面持有一个 GenerationMap：
```c
  typedef std::unordered_set<__unsafe_unretained id, ObjectHashFunctor, ObjectEqualFunctor> GenerationList;

  typedef std::unordered_map<Class, GenerationList, ClassHashFunctor, ClassEqualFunctor> GenerationMap;
```
当调用到 `markGeneration`时，做的操作是把这个临时的 Generation 添加到总的 Generations 的 vector 的尾部。
```c
  void GenerationManager::markGeneration() {
    generations.emplace_back(Generation {});
  }
```
注意这里没有用 push_back 而用的是 emplace_back, 后者会直接传入这个对象的构造函数，而不会产生 copy 的开销。

## 总结
### 优势
总的来说，FBAllocationTracker 实现简单，并且对性能影响较小。接入方便有效。基本可以实现内存监控的基本诉求。
### 劣势
1. 监控粒度不够细，像大量分配小内存引起的质变无法监控;
2. 打log间隔不好控制，间隔过长可能丢失中间峰值情况，间隔过短会引起耗电、io频繁等性能问题;
3. 上报的原始log靠人工分析，缺少好的页面工具展现和归类问题。