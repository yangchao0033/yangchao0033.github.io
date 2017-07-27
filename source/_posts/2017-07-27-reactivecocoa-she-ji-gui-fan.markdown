---
layout: post
title: "ReactiveCocoa 设计规范"
date: 2017-07-27 00:36:48 +0800
comments: true
categories: ios
---

[Design Guidelines](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/Documentation/DesignGuidelines.md#use-descriptive-declarations-for-methods-and-properties-that-return-a-signal)【译】
#### RACSequence 的一些约定
1. 默认情况延迟发生
2. 默认会阻塞调用者
3. 副作用只发生一次

#### RACSignal 的约定
1. 信号事件有序串行执行，保证不会同时到达两个或多个信号，但是可以运行在不同的scheduler之上。
2. 订阅事件总会在一个 scheduler 上
3. 错误会立即传播（优先级最高）
4. 订阅事件会产生副作用
5. 订阅关系总会在complete或error时自动销毁
6. 销毁会取消正在执行的任务并且清理相关的资源

<!--more-->

#### 常用练习
* 为返回信号的方法或属性使用描述性声明
声明语义有三个维度
  1. 冷信号（是否在此时被激活）还是热信号（需要订阅后激活）
  2. 信号返回值的个数是0个一个或更多？
  3. 信号是否有副作用

* 举个栗子
  * __没有副作用的热信号：__通常声明为属性来代替方法。暗示订阅之前不需要任何初始化，并且之后附加的任何订阅不会影响当前的订阅语义（没有副作用且每次订阅都是独立的语义）。信号属性通常命名在事件之后（eg: `textChangedSignal`）
  * __没有副作用的冷信号：__通常使用一个方法返回信号，并且通常使用名词命名（eg:`-currentText`）。方法声明暗示信号不会被保存（持有），暗示只有在订阅时才会执行任务。如果信号发送多个值，该名词必须是复数的。(-`currentModels`）
  * __有副作用的冷信号：__通常使用方法返回，并且使用动词命名（eg:`-logIn`）.动词命名暗示该方法的执行不是幂等得并且使用该方法的调用者需要小心清楚副作用的影响，除非副作用是被需要的或允许范围内的。如果该信号包含一个或多个值，需要用一个名词命名，并且使用对应的单复数形式。

#### 缩进流操作一致
 如果不适当的格式化，纯流式的代码将会变得很密集并且混乱。使用一致的缩进来高亮链式流的开始和结束。

当在单个流上调用方法时，不需要额外的缩进。
```objc
RACStream *result = [stream startWith:@0];

RACStream *result2 = [stream map:^(NSNumber *value) {
    return @(value.integerValue + 1);
}];
```
当多次转换相同的流时，请确保所有步骤都对齐。 复杂运算符`+ zip：reduce：`或`+ combineLatest：reduce：`可以拆分为多行以提高可读性：

```objc
RACStream *result = [[[RACStream
    zip:@[ firstStream, secondStream ]
    reduce:^(NSNumber *first, NSNumber *second) {
        return @(first.integerValue + second.integerValue);
    }]
    filter:^ BOOL (NSNumber *value) {
        return value.integerValue >= 0;
    }]
    map:^(NSNumber *value) {
        return @(value.integerValue + 1);
    }];
```
当然，使用block参数嵌套的流需要从block的自然缩进开始。
```objc
[[signal
    then:^{
        @strongify(self);

        return [[self
            doSomethingElse]
            catch:^(NSError *error) {
                @strongify(self);
                [self presentError:error];

                return [RACSignal empty];
            }];
    }]
    subscribeCompleted:^{
        NSLog(@"All done.");
    }];
```

#### 单个流中的所有值类型必须保持一致。
使用不同类型需要使用复杂的操作流程，且会给使用流的消费者增加额外的负担。

#### 避免过久的持有流
如果持有流的时间超过他本身需要存活的时间时，将会导致流本身持有的或依赖的资源无法释放，潜在的造成内存使用过高。

#### 只处理需要的信号
除了会增加内存使用，不必要的持有流或者 `RACSignal` 订阅关系会导致 CPU 使用增加, 因为不必要的任务执行所得出的结果永远不会被使用到。一般可以使用`-take`或`-takeUntil`可以自动在不必要的时候取消流执行

#### 在已知的 Scheduler 上传送信号事件
流信号返回的值可能会来自很多复杂的场景，很可能是在底层线程中造作得出，所以在给UI元素这是返回的信号值时，一定要保证其赋值造作发成在`mainScheduler` (主线程)只上。

#### 尽可能少的切换调度者（`scheduler`）

非必须的情况下，务必让流信号的执行发生在同一 `Scheduler` 之上。因为切换 `Scheduler` 会引入不必要的延迟并且会增加CPU资源的消耗。

通常 `-deliverOn:`严格限制只会在信号链的尾部使用，比如在订阅之前，或为属性绑定值时

#### 明确信号的副作用
通常一定要避免副作用的发生，但是有一些副作用是我们所期望的,一般应用在一些hook方法进行调试，例如 `-doNext:`、`-doError:`、`-doCompleted:`等

```objc
NSMutableArray *nexts = [NSMutableArray array];
__block NSError *receivedError = nil;
__block BOOL success = NO;
RACSignal *bookkeepingSignal = [[[valueSignal
    doNext:^(id x) {
        [nexts addObject:x];
    }]
    doError:^(NSError *error) {
        receivedError = error;
    }]
    doCompleted:^{
        success = YES;
    }];

RAC(self, value) = bookkeepingSignal;

```

#### 使用组播分享带有副作用的信号
使用组播可以允许单个信号拥有任意数量的订阅关系。

```objc
// This signal starts a new request on each subscription.
RACSignal *networkRequest = [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
    AFHTTPRequestOperation *operation = [client
        HTTPRequestOperationWithRequest:request
        success:^(AFHTTPRequestOperation *operation, id response) {
            [subscriber sendNext:response];
            [subscriber sendCompleted];
        }
        failure:^(AFHTTPRequestOperation *operation, NSError *error) {
            [subscriber sendError:error];
        }];

    [client enqueueHTTPRequestOperation:operation];
    return [RACDisposable disposableWithBlock:^{
        [operation cancel];
    }];
}];

// Starts a single request, no matter how many subscriptions `connection.signal`
// gets. This is equivalent to the -replay operator, or similar to
// +startEagerlyWithScheduler:block:.
RACMulticastConnection *connection = [networkRequest multicast:[RACReplaySubject subject]];
[connection connect];

[connection.signal subscribeNext:^(id response) {
    NSLog(@"subscriber one: %@", response);
}];

[connection.signal subscribeNext:^(id response) {
    NSLog(@"subscriber two: %@", response);
}];
```

#### 通过给信号命名进行调试

```objc
RACSignal *signal = [[[RACObserve(self, username)
    distinctUntilChanged]
    take:3]
    filter:^(NSString *newUsername) {
        return [newUsername isEqualToString:@"joshaber"];
    }];

NSLog(@"%@", signal);
// 打印结果 [[[RACObserve(self, username)] -distinctUntilChanged] -take: 3] -filter:
```
使用 [-setNameWithFormat:](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/RACStream.h) 对信号自定义信号名。
此外，`RACSignal` 还提供了 `-logNext`, `-logError`,` -logCompleted`, `-logAll`方法，当信号事件执行时自动打印信号的信息（包括 `name`），更方便检测到信号的实时状况。

#### 避免显式的订阅和销毁
虽然-subscribeNext：error：completed：和它的变体是处理信号的最基本的方式，但是它们较少的声明性，鼓励使用副作用和潜在地复制内置功能而使代码复杂化。

同时，显式的调用 `RACDisposable` 类会快速导致一堆乱七八糟的资源管理并且清理代码。

* 解决方案 ：使用较高级别的模式来替换手动订阅和销毁
  * 使用 `RAC()` 或者 `RACChannelTo` 宏定义可以用来绑定信号或者属性，代替在状态变化时手动更新。
  * 例如 `-takeUntil:` 等操作符可以用来当参数中的事件发生时自动销毁一段订阅关系（比如 `取消按钮` 被点击时）。

通常 ，相比于使用订阅的回调，使用内置的 `stream` 和 `signal` 操作符将会让代码变得更简洁并且能产生更少的易错代码。

#### 尽可能避免使用 subjects
`Subject` 是一个用来桥接代码到 signal 世界的强大工具，但是就像 RAC 中的可变变量一样，当发生滥用时，他们会更快的导致程序变得复杂。

Subjects 通常会被 ReactiveCocoa 的其他模式所代替：
  * 相比于给 `Subject` 提供初始值，不如考虑使用  [+createSignal:](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/RACSignal.h) block 产生值代替。
  * 相比于将一些中间值发送给发送给 subject，不如使用 `+combineLatest:` 或者 `+zip:`操作符组合多个信号的输出值。
  * 基于一个 Signal 使用 [multicast](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/Documentation/DesignGuidelines.md#share-the-side-effects-of-a-signal-by-multicasting) (组播)，而不是使用多个 订阅者共享 `subjects` 的结果。
  * 使用 [command](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/RACCommand.h) 或者 [-rac_signalForSelector:](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/NSObject+RACSelectorSignal.h) ，而不是为了简单的控制一个 subject 实现一个事件方法。

  当不得不使用 subject 时，他们必须是信号链中最基本的输入，而不是其中的一个。

###  支持自定义操作符

#### 推荐使用 RACStream 的方法
 

#### 尽可能使用现有的操作符进行构思
 

### 尽量避免引入并发
尽量使用 RACScheduler 代替

#### 在 disposable 中取消任务和清理所有关联资源。
当使用 [+createSignal:](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/RACSignal.h) 创建信号是，通常会返回一个 [RACDisposable](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/RACDisposable.h) 对象。而这个 disposable 对象应该：
* 尽可能方便优雅的取消正在进行的由 `signal` 开启的任务。
* 立即销毁任何信号的订阅关系，由此来触发他们的取消和清除代码。
* 释放任何信号开辟的内存和占用的资源。

#### 不要在操作符中分块
流操作符应该立即返回一个新的流。操作符需要执行的任何操作都应该是执行新流的一部分，而不是运算符调用本身的一部分。

```objc
// WRONG!
- (RACSequence *)map:(id (^)(id))block {
    RACSequence *result = [RACSequence empty];
    for (id obj in self) {
        id mappedObj = block(obj);
        result = [result concat:[RACSequence return:mappedObj]];
    }

    return result;
}

// Right!
- (RACSequence *)map:(id (^)(id))block {
    return [self flattenMap:^(id obj) {
        id mappedObj = block(obj);
        return [RACSequence return:mappedObj];
    }];
}
```
#### 避免深度递归导致的堆栈溢出
任何可能无限递归的操作符都应该使用 `-scheduleRecursiveBlock:`  [RACScheduler](https://github.com/ReactiveCocoa/ReactiveObjC/blob/master/ReactiveObjC/RACScheduler.h) 方法。这个方法将会把递归转换为迭代来防止堆栈溢出。
例如：下面的例子将会错误地实现 `-repeat` ,应为这将潜在的引起堆栈溢出并且崩溃。

```objc 
- (RACSignal *)repeat {
    return [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

        __block void (^resubscribe)(void) = ^{
            RACDisposable *disposable = [self subscribeNext:^(id x) {
                [subscriber sendNext:x];
            } error:^(NSError *error) {
                [subscriber sendError:error];
            } completed:^{
                resubscribe();
            }];

            [compoundDisposable addDisposable:disposable];
        };

        return compoundDisposable;
    }];
}
```

相比之下，下面的版本将会避免。

```objc
- (RACSignal *)repeat {
    return [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

        RACScheduler *scheduler = RACScheduler.currentScheduler ?: [RACScheduler scheduler];
        RACDisposable *disposable = [scheduler scheduleRecursiveBlock:^(void (^reschedule)(void)) {
            RACDisposable *disposable = [self subscribeNext:^(id x) {
                [subscriber sendNext:x];
            } error:^(NSError *error) {
                [subscriber sendError:error];
            } completed:^{
                reschedule();
            }];

            [compoundDisposable addDisposable:disposable];
        }];

        [compoundDisposable addDisposable:disposable];
        return compoundDisposable;
    }];
}
```

