---
title: "Method Swizzling"
date: 2021-03-20 21:00:00
tags: ["iOS","AOP"]
categories: ["iOS"]
---

>  iOS AOP文章系列
>
>  前导知识：
>  * [Mach-O文件结构分析](https://houugen.fun/posts/mach-o%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84%E5%88%86%E6%9E%90.html)
>  * [静态链接&动态链接](https://houugen.fun/posts/%E9%9D%99%E6%80%81%E9%93%BE%E6%8E%A5%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5.html)
>  * [OC方法&OC类&OC对象](https://houugen.fun/posts/oc%E6%96%B9%E6%B3%95oc%E7%B1%BBoc%E5%AF%B9%E8%B1%A1.html)
>  * [方法查找和消息转发](https://houugen.fun/posts/%E6%96%B9%E6%B3%95%E6%9F%A5%E6%89%BE%E5%92%8C%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91.html)
>
>  AOP框架：
>  * [Method Swizzling](https://houugen.fun/posts/method-swizzling.html)
>  * Fishhook
>  * Apsects
>  * NSProxy AOP

## iOS AOP

接下来的文章我们正式进入 iOS AOP 的世界。

AOP 是一种面向切面编程，核心思想是在不修改源码情况下给程序添加功能的技术。

根据 OC 的 runtime 特性，我们先来介绍比较基础的一种实现方式 -- **Method Swizzling**。

## 🌰 演示

我们依旧使用 **前导知识文章** 中的 `Demo`，先为 `Person` 添加一个 `Category` 类来扩展一个 `eat` 方法，毕竟干饭人除了睡还要吃，要不和咸鱼有什么区别，我们想在 **人 睡**前先 **吃饱饭**，该如何利用 `Swizzling` 实现呢？

```objective-c
// Person.m
@implementation Person

- (void)sleep {
    NSLog(@"i am sleeping");
}

@end

// Person+swizzle.m
@implementation Person (swizzle)

- (void)eat {
    NSLog(@"i am eating");
    // 这里调用自身并不会导致递归调用
    // 因为在 main 中交换了 eat 和 sleep 的 IMP，所以这里实际上调用的是 sleep
    [self eat];
}

@end

// main.m
int main(int argc, const char * argv[]) {
    
    @autoreleasepool {
        Person *p = [[Person alloc] init];
        
        // 交换 sleep 和 eat 的 IMP
        Method ori_Method = class_getInstanceMethod([Person class], @selector(sleep));
        Method new_Method = class_getInstanceMethod([Person class], @selector(eat));
        method_exchangeImplementations(ori_Method, new_Method);
        
        [p sleep];
    }
    return 0;
}
```

这里调用 `Person` 的 `sleep` 方法，交换后实际调用 `Person+swizzle` 的 `eat` 方法，先打印 “i am eating”，然后执行 `sleep` 方法打印 “i am sleeping”。

{{< figure src="./MethodSwizzling.assets/1616729180721-8374fa05-7ed6-43c1-971d-07dc30cb3861.png" width="50%" >}}

当然还有一些在 `runtime` 中的 `API` 也能实现类似效果：

- `BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)` // 类中添加方法
- `IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)` // 修改类方法
- `Method class_getInstanceMethod(Class aClass, SEL aSelector)` // 获取类实力方法
- `IMP method_setImplementation(Class cls, method_t *m, IMP imp)` // 设置方法 imp

## 原理

原理非常简单，我们看下 `method_exchangeImplementations` 的核心实现一目了然

```c
void method_exchangeImplementations(Method m1, Method m2)
{
    ...
    IMP imp1 = m1->imp(false);
    IMP imp2 = m2->imp(false);
    SEL sel1 = m1->name();
    SEL sel2 = m2->name();

    m1->setImp(imp2);
    m2->setImp(imp1);
    ...
}
```

就是在运行时，改变方法结构体 `method_t` 的底层实现

```c
struct method_t {
    SEL name; // 运行时方法名
    const char *types; // 方法类型编码
    MethodListIMP imp; // 方法实现的指针
}
```

执行的操作图形化为

{{< figure src="./MethodSwizzling.assets/1616741225024-eccea5bf-d3dc-428a-a3a0-cc6708e1cd19.png" width="80%" >}}

## 使用建议

- +load 

`Swizzling` 在类的 `+load` 方法中完成。

因为 `+load`  方法会在类被添加到 OC 运行时执行，且只会被调用一次，保证了 `Swizzling` 方法的及时处理。



- dispath_once

`Swizzling` 在 `dispatch_once` 中完成。保证只执行一次。



- prefix

`Swizzling` 方法添加前缀，避免方法名称冲突。



- invoke the original imp

不调用原始实现很可能会导致程序状态或逻辑异常。