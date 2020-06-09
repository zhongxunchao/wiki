##1. NSProxy和NSObject
基本所有的iOS中的类都是NSObject的字类，但是NSProxy不是。
NSProxy是一个虚类，你可以通过继承它，并重载下面两个方法以实现将消息转发到另一个实体。
```
- (void)forwardInvocation:(NSInvocation *)invocation;
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel NS_SWIFT_UNAVAILABLE(“NSInvocation and related APIs not available”);
```
这里最好描述一下虚类的概念：虚类又叫做抽象类，这个类主要定义一些方法然后让子类去实现。正如有人描述的，动物是一个大概念，但是通常情况下你不会去定义动物的对象；而是先产生继承的字类猫啊，狗啊的，再去实例化。这个动物就可以用虚类来表示了，毕竟动物是有共性的。以上纯属个人理解，非官方语言描述！
NSObject既是对象的基类，又是一种协议。它的头文件是这样的：
```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
而NSObject协议的定义是这样的:
```
#include <objc/objc.h>
#include <objc/NSObjCRuntime.h>

@class NSString, NSMethodSignature, NSInvocation;

@protocol NSObject

- (BOOL)isEqual:(id)object;
@property (readonly) NSUInteger hash;

@property (readonly) Class superclass;
- (Class)class OBJC_SWIFT_UNAVAILABLE("use 'anObject.dynamicType' instead");
- (instancetype)self;

- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;

- (BOOL)isProxy;

- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;

- (BOOL)respondsToSelector:(SEL)aSelector;

- (instancetype)retain OBJC_ARC_UNAVAILABLE;
- (oneway void)release OBJC_ARC_UNAVAILABLE;
- (instancetype)autorelease OBJC_ARC_UNAVAILABLE;
- (NSUInteger)retainCount OBJC_ARC_UNAVAILABLE;

- (struct _NSZone *)zone OBJC_ARC_UNAVAILABLE;

@property (readonly, copy) NSString *description;
@optional
@property (readonly, copy) NSString *debugDescription;

@end
```
很全很熟悉不是吗？！

NSProxy一个虚类，但是同时它也实现了NSObject协议，它的定义是这样的:
```
@interface NSProxy <NSObject> {
    Class   isa;
}
```

##2. 方法重定向
比较优秀的iOS工程师应该都知道对象在调用方法时的机制，会寻找当前类的方法缓存，方法链表；然后依次向父类去寻找，一直到NSObject,到了NSobject还是找不到的话，我们有机会使用上面的方法来转移方法的注意力了。
到这里我本来想引用另一个博客中定义了NSProxy父类的方法，但是博主不允许随便转载，大家有空自己去看吧。
[链接在此](http://blog.csdn.net/devday/article/details/7418022)
我们先定义一个TestTool类。
```
//TestTool.h
@interface TestTool : NSObject
{
    id star;
}

- (instancetype)initWithId:(id)obj;

//- (void)test;
@end
```
```
//TestTool.m
- (instancetype)initWithId:(id)obj
{
    star = [obj copy];
    return self;
}
```
请注意这里我们没有定义test方法。如果我们对TestTool对象调用该方法，毫无疑问会崩溃的。（请注意这里编译器如何不报错，使用id对象）
这时重定向的方法可以起作用了,在TestTool.m中加入以下方法：
```
//如果可以在m文件内部捕捉到这些方法则这些都不会调用
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    NSLog(@"调用了forwardInvocation方法");
    [anInvocation invokeWithTarget:star];
}

//方法签名需要和NSInvocation联合使用
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    //对于NSObject的字类，如果头文件中没有声明该方法，那么直接编译错误
    //而如果头文件声明，但是在m文件中找不到实现，则会调用该方法，从而有机会使用别的对象来实现
    NSLog(@"检验本类中该函数的签名");
    NSMethodSignature *sig;
    if ([star methodSignatureForSelector:aSelector])
    {
        //如果这里不把方法签名传到star上也会报错的，test方法找不到实现的目标
        sig = [star methodSignatureForSelector:aSelector];
    }
    return sig;
}
```
很明显，在methodSignatureForSelector：中，我们将试图把star的方法签名赋予Target;而forwardInvocation:则指定了方法实现的目标。两者缺一不可。
接着我们试着去测试它,先定义重定向的目标类，当然它最好有test方法了。
```
//Another.h
@interface Another : NSObject<NSCopying>

- (void)test;
@end
```
```
- (void)test
{
    NSLog(@"这里实现了test方法");
}

- (id)copyWithZone:(NSZone *)zone
{
    return [[[self class] allocWithZone:zone] init];
}
```
OK,现在我们可以正式去看看重定向的结果了：
```
Another *another = [Another new];
id tool = [[TestTool alloc] initWithId:another];
[tool test];
```
可以看到打印的结果：这里实现了test方法.
##3.注意
上面值得注意的一点是记得TargetProxy或者其他对象初始化时返回的是id，而不直接指定为该类的对象，否则编译器将去该类的父类簇的方法链表中寻找该方法，导致直接编译出错，这样连消息重定向的机会都没有了。
##4.再看看
本来研究到这里告一段落了，和我之前留下的印象差不多。但是看到NSObject中的一些方法，忍不住拿来玩了一下。
```
+ (BOOL)instancesRespondToSelector:(SEL)aSelector
{
    NSLog(@"调用instancesRespondToSelector");
    return YES;
}
+ (BOOL)conformsToProtocol:(Protocol *)protocol
{
    NSLog(@"调用conformsToProtocol");
    return YES;
}
- (IMP)methodForSelector:(SEL)aSelector
{
    NSLog(@"调用methodForSelector");
    IMP imp = [self methodForSelector:aSelector];
    return imp;
}
+ (IMP)instanceMethodForSelector:(SEL)aSelector
{
    NSLog(@"调用instanceMethodForSelector");
    IMP imp = [self instanceMethodForSelector:aSelector];
    return imp;
}
- (void)doesNotRecognizeSelector:(SEL)aSelector
{
    NSLog(@"调用doesNotRecognizeSelector");
}

- (id)forwardingTargetForSelector:(SEL)aSelector
{
    NSLog(@"调用forwardingTargetForSelector");
    return self;
}

+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector
{
    NSMethodSignature *sig = [self instanceMethodSignatureForSelector:aSelector];
    NSLog(@"调用instanceMethodSignatureForSelector");
    return sig;
}
```
大多数方法含义是明确的，但是确实发现forwardingTargetForSelector：这个方法会在消息转移之前被调用。
查阅了一些资料后，它的作用是可以直接将要转发的对象返回。你应该有办法去尝试这个方法的作用。
A如果想要把一封信交给B，可以直接把B叫到家里来；也可以给B送过去，这个机制可谓相当厉害了。
在以上3个方法都没有找到消息接收者的情况下，系统大概也没办法了，只能给你最后一次补救的机会：
```
- (void)doesNotRecognizeSelector:(SEL)aSelector;
```
我没能找到这个方法，你自己看着办吧，我要抛出异常了。好吧，到这里应用基本上就会崩溃了。
##5.小结
最后总结一下呢，就是调用方法的消息发出后，先去该类及类别的方法链表中去找，找不到就去找父类了；直到最后找到基类：NSObject.
然后会依次调用下面几个方法（在没有消息响应者的情况下）：
```
- (id)forwardingTargetForSelector:(SEL)aSelector；
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel；
- (void)forwardInvocation:(NSInvocation *)invocation；
- (void)doesNotRecognizeSelector:(SEL)aSelector；
```
另外补一句，使用try…catch方法也可以不崩溃，不过个人不是特别感冒这个。问题隐藏了不代表不存在，不是吗？