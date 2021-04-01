# NSProxy AOP


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
>  * [Fishhook](https://houugen.fun/posts/fishhook.html)
>  * [Apsects](https://houugen.fun/posts/apsects.html)
>  * [NSProxy AOP](https://houugen.fun/posts/nsproxy-aop.html)


## 介绍

iOS 有一个原生的 `AOP` 方法，就是利用 `NSProxy` 代理类！，我们先看下[官网](https://developer.apple.com/documentation/foundation/nsproxy)介绍：

{{< admonition quote >}}
Typically, a message to a proxy is forwarded to the real object or causes the proxy to load (or transform itself into) the real object. Subclasses of `NSProxy` can be used to implement transparent distributed messaging (for example, [`NSDistantObject`](https://developer.apple.com/documentation/foundation/nsdistantobject)) or for lazy instantiation of objects that are expensive to create.

`NSProxy` implements the basic methods required of a root class, including those defined in the [`NSObjectProtocol`](https://developer.apple.com/documentation/objectivec/nsobjectprotocol) protocol. However, as an abstract class it doesn’t provide an initialization method, and it raises an exception upon receiving any message it doesn’t respond to. A concrete subclass must therefore provide an initialization or creation method and override the [`forwardInvocation(_:)`](https://developer.apple.com/documentation/foundation/nsproxy/1416417-forwardinvocation) and [`methodSignatureForSelector:`](https://developer.apple.com/documentation/foundation/nsproxy/1589828-methodsignatureforselector) methods to handle messages that it doesn’t implement itself.
{{< /admonition >}}

说明两点：

1. `NSProxy` 本身就是用来做代理转发消息的
2. `NSProxy` 的子类必须通过 `forwardInvocation` 和 `methodSignatureForselector` 来创建方法

{{< admonition note >}}
还记得我们在方法查找和消息转发文章中最后讲的慢速消息转发过程吗？
{{< /admonition >}}

`NSProxy` 的声明：

```objective-c
NS_ROOT_CLASS
@interface NSProxy <NSObject> {
    __ptrauth_objc_isa_pointer Class    isa;
}

+ (id)alloc;
+ (id)allocWithZone:(nullable NSZone *)zone NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
+ (Class)class;

- (void)forwardInvocation:(NSInvocation *)invocation;
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel NS_SWIFT_UNAVAILABLE("NSInvocation and related APIs not available");
- (void)dealloc;
- (void)finalize;
@property (readonly, copy) NSString *description;
@property (readonly, copy) NSString *debugDescription;
+ (BOOL)respondsToSelector:(SEL)aSelector;

- (BOOL)allowsWeakReference API_UNAVAILABLE(macos, ios, watchos, tvos);
- (BOOL)retainWeakReference API_UNAVAILABLE(macos, ios, watchos, tvos);

// - (id)forwardingTargetForSelector:(SEL)aSelector;

@end
```

包含 `isa` 指针，说明其本身就是一个对象。并且遵守 `NSObjcet` 协议，还存在 `forwardInvocation` 和 `methodSignatureForSelector` 方法。

## 🌰 🌰 在目

新增一个 `MyProxy` 类继承 `NSProxy`，并在其中实现一个目标对象的代理引用，同时实现两个关键方法:

```objective-c
// MyProxy.h
@interface MyProxy : NSProxy {
    id _target;
}

+ (id)proxyWithTarget:(id)target;

@end

// MyProxy.m
@implementation MyProxy

+ (id)proxyWithTarget:(id)target {
    MyProxy *mp = [MyProxy alloc];
    mp->_target = target;
    return mp;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [_target methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    if (invocation && [_target respondsToSelector:invocation.selector]) {
        NSString *sn = NSStringFromSelector(invocation.selector);
        NSLog(@"do something before %@", sn);
        [invocation invokeWithTarget:_target];
        NSLog(@"do something after %@", sn);
    }
}

@end

// main.m 中添加代码
//        ==== NSProxy ====
Person *p = [MyProxy proxyWithTarget:[[Person alloc] init]];
[p sleep];
[p walk];
```

{{< figure src="./NSProxyAOP.assets/1617101686961-2f6bfb72-2849-4447-8697-45f570eeaa6a.png" width="80%" >}}

## 原理

`proxyWithTarget:` 实现将一个对象引用到自实现的代理类中，在真正调用对象方法时，代理类没有方法定义，找不到对应的 `IMP`，就会进入消息转发流程，从而执行 `forwardInvocation:` ，而我们在 `forwardInvocation:` 中执行原方法前后加入额外逻辑实现 `AOP`。

这里不仅可以打印方法名和修改方法逻辑，因为 NSInvocation 中封装了方法的全部信息，因此还能够打印或修改**入参**及**返回值**。
