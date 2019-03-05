### 阿里开源 iOS 协程开发框架 coobjc 学习

## coobjc 概览
coobjc 为 OC 和 Swift 提供了协程的功能。支持 await、generator 和 actor model，并且在 cokit 库中为 Foundation 和 UIKit 的部分 API 提供了协程化支持，包括 NSFileManager , JSON , NSData , UIImage 等。coobjc 同时还提供了元组的支持。


### 什么是协程？
- 通俗的讲，协程是一种比线程更轻量级的存在。如一个进程可以拥有多个线程一样，一个线程可以拥有多个协程。一个协程 A 在执行过程中，如果碰到 yield 关键字，就会中断执行，直到主线程调用 send/next 方法发送了数据，协程 A 才会接收到数据继续执行。

- 在协程中通过 yield 来暂停协程执行和在线程的阻塞是有本质区别的。线程的阻塞状态是由操作系统的内核进行切换的，而协程不是被操作系统内核管理，而是完全由程序控制，也就是在用户态执行，不需要像线程那样在用户态和内核态之间来回切换。所以，协程的开销会远远小于线程的开销。和多线程相比，线程的数量越多，协程的性能优势也就越明显。

- 协程相对使用线程的另一个优点就是：协程不需要多线程的锁机制，因为只有一个线程，也不存在同时写变量冲突，在协程中控制共享资源不需要加锁，只需要判断状态就好了，所以执行效率也会比多线程高很多。

- ! 使用协程的优点：

1. 协程不是被操作系统内核管理，是在用户态执行，所以节省了线程切换的性能开销
2. 不需要多线程的锁机制

- ！使用协程的缺点：

1. 我们需要自己承担协程之间调度的责任。
2. 由于协程本质上是在单线程上跑的，也就失去了线程使用多 CPU 的能力，无法利用多核资源。只有将协程和进程配合才可以使用多 CPU。 
3. 在协程中如果使用了阻塞操作，会阻塞掉整个程序。。

### 使用 coobjc 能解决什么问题？

coobjc 使用协程的方式优化了 iOS 中的异步操作，解决了 iOS 基于 block 的异步编程回调中容易碰到的以下问题：

1. 容易进入"嵌套地狱"
2. 错误处理复杂和冗长
3. 容易忘记调用 completion handler
4. 条件执行变得很困难
5. 从互相独立的调用中组合返回结果变得极其困难
6. 在错误的线程中继续执行
7. 难以定位原因的多线程崩溃
8. 锁和信号量滥用带来的卡顿、卡死

一个明显的优点就是，coobjc 使用协程把异步变同步，简化代码，方便使用和维护。
下面是官方文档中给出的一个使用传统异步回调方法通过网络请求加载一张图片和使用 coobjc 的方式加载图片的代码对比示例：


```
//Asynchronous loading of data from the network
[NSURLSession sharedSession].configuration.requestCachePolicy = NSURLRequestReloadIgnoringCacheData;
    NSURLSessionDownloadTask *task = [[NSURLSession sharedSession] downloadTaskWithURL:url completionHandler:
                                      ^(NSURL *location, NSURLResponse *response, NSError *error) {
                                          if (error) {
                                              return;
                                          }

                                          //Parsing data in child threads and generating images                                         
                                          dispatch_async(dispatch_get_global_queue(0, 0), ^{
                                              NSData *data = [[NSData alloc] initWithContentsOfURL:location];
                                              UIImage *image = [[UIImage alloc] initWithData:data];
                                              dispatch_async(dispatch_get_main_queue(), ^{
                                                  //Dispatch to the main thread to display the image 
                                                  imageView.image = image;
                                              });
                                          });

                                      }];
```

coobjc:

```
co_launch(^{
    NSData *data = await(downloadDataFromUrl(url));
    UIImage *image = await(imageFromData(data));
    imageView.image = image;
});
```


## 使用 coobjc

### Simple Launch
你可以在任何地方添加一个协程：

```
// Create a coroutine with block, and just resume it.
co_launch(^{
    // do something. The coroutine just asynchronous run in the current `dispatch_queue`
});
```

```
co_launch_onqueue(q, ^{
    // ...
});

```

### Await
```
/**
 Wait a  `COPromise` or `COChan` object, until Promise is fulfilled/rejected, or Channel has value send.

 @param awaitable `COPromise` or `COChan` object
 @return If await Promise, return fulfilled value; elseif Channel, return sent value.
 */
id _Nullable co_await(id  awaitable);

```
用 COPromise 或者 COChan 实现一个函数，然后在协程 coroutine 中使用 await 方法去等待函数执行的结果。直到函数返回的 promise 执行 resolve 或者 reject，或者函数返回的 channel 中有值被 send 进来，await 才会继续往下执行。

### COPromise && COChan

#### 什么是 COChan？

coobjc 的 COChan，也就是 Channel,是 CSP(Communicating Sequential Processes)的一种并发模型。它的实现是参考 [libtask](https://swtch.com/libtask/) 的。

> CSP并发模型

> CSP模型是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型。 CSP中channel是一类对象，它不关注发送消息的实体，而关注与发送消息时使用的channel。

> channel 是被单独创建并且可以在进程之间传递，它的通信模式类似于 boss-worker 模式的，一个实体通过将消息发送到channel 中，然后又监听这个 channel 的实体处理，两个实体之间是匿名的，这个就实现实体中间的解耦，其中 channel 是同步的一个消息被发送到 channel 中，最终是一定要被另外的实体消费掉的，在实现原理上其实是一个阻塞的消息队列。

画重点：channel 在实现原理上其实是一个阻塞的消息队列

那么在 coobjc 中也是使用 COCoroutine 做为并发实体，coroutine 非常轻量级可以创建几十万个实体。实体间通过 COChan 继续匿名消息传递使之解耦。

![channel](https://github.com/alibaba/coobjc/raw/master/docs/images/channel1.png)

![channel](https://github.com/alibaba/coobjc/raw/master/docs/images/channel2.png)

COChan 类的结构也比较简单，整个 .h 文件中，只有 COChan 的初始化方法、sendValue、receiveValue 和 cancel 方法。-> COChan.h

![channel](http://www.moye.me/wp-content/uploads/2017/05/CSP-commuication-semantics-1024x564.png)

在 coobjc 中，COChan 的 send 和 receive 分别有两种方法。一种是有缓冲区，另一种是没有缓冲区的方法。

-> Code testForChannelWithNoCache 

PS: 

1. receive 和 send 必须要在协程中进行调用
2. receive 和 send 会使当前协程挂起。如果使用send，并且没有人 receive 这个消息或者 buffer  已经满了的情况下，会导致当前协程被挂起。直到有人 receive 处理了这个消息。如果使用 receive，但是 Channel 中没有任何消息，那么当前协程会被挂起，直到 Channel 中被 send 了消息。
3. send\_nonblock 和 receive\_nonblock 没有必须在协程中使用的限制。调用这两个方法不会造成阻塞。
4. send_nonblock 在调用时，如果有人正在 receiving，那么就把消息发送给他。如果没有人在 receive 消息，并且 channel 的 buffer 没有满的情况下，就将消息保存到 buffer 中。如果没人 receive 并且 buffer 也满了的情况下，就丢弃掉这条消息。
5. receive_nonblock 在调用时，如果 channel 的 buffer 中有值，那么就取这个值。如果 buffer 中没有值，但是这时候有人正在调用 sending，那么就接收 sending 的值。如果 buffer 中没有值，也没有人在 sending 消息，那么就 return nil。

#### 什么是 COPromise？
COPromise 和前端中的 Promise 用法大致相同。
> Promise 是异步编程的一种解决方案：从语法上讲，promise是一个对象，从它可以获取异步操作的消息；从本意上讲，它是承诺，承诺它过一段时间会给你一个结果。promise有三种状态：pending(等待态)，fulfiled(成功态)，rejected(失败态)；状态一旦改变，就不会再变。创造promise实例后，它会立即执行。
> 

用起来就类似这样：

```
- (COPromise<id> *)co_fetchSomethingAsynchronous {

    return [COPromise promise:^(COPromiseResolve  _Nonnull resolve, COPromiseReject  _Nonnull reject) {
        dispatch_async(_someQueue, ^{
            id ret = nil;
            NSError *error = nil;
            // fetch result operations
            ...

            if (error) {
                reject(error);
            } else {
                resolve(ret);
            }
        });
    }];
}

```

### 回到 Await
-> Code co_fetchSomethingAsynchronous

-> Code co_fetchSomething

??? 

1. 在 co\_fetchSomething 中，如果把 send_nonblock 方法改成 send 方法会怎么样？
2. 如果想要用 send 方法去实现，需要怎么做？
-> Code co\_fetchSomething1

3. 在 co\_fetchSomething1 中，如果把 co_launch 注释掉会怎么样？

- Await 的内部实现

await 内部对参数进行了类型判断，如果是 Channel 就调用 channel 的 receive 方法，阻塞当前的协程并且等待 receive 返回值，这也就是 await 会使当前 coroutine 挂起的原因。那么如果参数是 Promise 类型，那么内部会生成一个 Channel，将这个 Channel 与 Promise 绑定在一起，然后调用 channel 的 receive 方法，阻塞当前的协程并且等待返回值。当 Promise 返回处理结果时，channel 会通过 send_nonblock 的方法将值 send 过来，然后由于这时候 channel 在 receive 等待中，所以 receive 会马上接收到这个值然后返回结果。那么，如果 Promise 返回的是 error， send_nonblock 会塞一个 nil 进来。所以外部可以通过值是否为 nil 来判断是否发生了错误。

-
### Generator

生成器：生成器在迭代中以某种方式生成下一个值并且返回和next()调用一样的东西。挂起返回出中间值并多次继续的协同程序被称作生成器。

生成器可以在很多场景中进行使用，比如消息队列、批量下载文件、批量加载缓存等：

![Generator](https://github.com/alibaba/coobjc/raw/master/docs/images/generator_execute.png)

```
- (void)testForRandomGenerator {
    COCoroutine *generator = co_sequence(^{
        NSArray *array = @[@"🍎",@"🐶",@"🤖️",@"✈️"];
        while(co_isActive()){
            int index = arc4random() % array.count;
            NSString *result = [array objectAtIndex:index];
            NSLog(@"this is a %@",result);
            yield_val(result);
        }
    });
    
    co_launch(^{
        for(int i = 0; i < 10; i++){
            NSString *whatIsThis = [generator next];
            NSLog(@"look, what I get in the box! %@",whatIsThis);
        }
        [generator cancel];
    });
}
```

生成器中的代码只有在外部需要的时候才会执行，也就是外部向生成器发送 next 消息的时候才会触发生成器并开始产生数据。在生成数据之后，它就会挂起，等待下次收到 next 消息才会继续执行生成数据。

和传统的 NSArray、NSSet、NSDictionary 等数据容器相比，生成器不需要提前将所有数据准备好并存储到容器中。并且，生成器的实现是线程安全的，因为它们都是在单线程上运行，数据在生成器中生成，然后在另一条协程上使用，期间不需要加任何锁。而使用传统容器需要注意线程安全问题并且容易引发 crash。

使用生成器去实现生产者消费者模型的时候，我们可以把传统的生产者生产出东西，然后去通知消费者消费的方式转变为 消费者需要消费的时候去告诉生产者马上生产出东西来给我。与传统的模式相比，使用生成器实现的方式，避免了去使用一些多线程共享的变量计算，也避免了锁的使用。

？？？： 如果注释掉 [generator cancel] 会有什么问题？

### Actor

actor模式是一种最古老的也是最简单的并行和分布式计算解决方案
Actor模型=数据+行为+消息。
Actor模型内部的状态由自己的行为维护，外部线程不能直接调用对象的行为，必须通过消息才能激发行为，这样就保证Actor内部数据只有被自己修改。


实现一个计数器的代码可能是这样：

```
@interface Counter: NSObject 

@property (nonatomic, assign) int count;
- (void)incCount;
- (int)getCount;
@end

@implementation Counter

- (void)incCount {
	@synchronized(self) {
		_count ++;
	}
}

- (int)getCount {
	@synchronized(self) {
		return _count;
	}
}
@end
```

使用 coobjc 实现 Actor 的计数器代码：

```
- (void)testForActor {
    COActor *countActor = co_actor(^(COActorChan * _Nonnull channel) {
        //定义actor的状态变量
        int count = 0;
        for (COActorMessage *message in channel) {
            //处理消息
            if ([[message stringType] isEqualToString:@"inc"]) {
                count++;
                NSLog(@"the count is %d now.", count);
            }
            else if ([[message stringType] isEqualToString:@"get"]) {
                message.complete(@(count));
                NSLog(@"get the count %d", count);
            }
        }
    });
    
    // 给 actor 发送消息
    [countActor sendMessage:@"inc"];
    [countActor sendMessage:@"inc"];
}
```

-> Code COActor.h

COActor: COActor 是继承于 COCoroutine 的一个子类，一个 Actor 就是一条协程。COActor 的方法列表很简单，除了初始化方法，就只有一个 sendMessage 方法用于在 Actors 之间发送消息。所以整个 Actor 的机制就是用 sendMessage 来维持消息传递的。

COActorChan: COChan 的子类，实现了快速枚举协议
COActorMessage：发送给 COActor 的消息对象类型
COActorCompletable:COPromise 的子类，啥🐔儿没写，就是一个 COPromise。

message 的 complete 方法内部其实就是返回 promise 的 fulfill 值。

```
- (void (^)(id))complete {
    COActorCompletable *completable = _completableObj;
    return ^(id val){
        if (completable) {
            [completable fulfill:val];
        }
    };
}
```

-> Code testForActor

？？？ 
1. 为什么发送 get 获取到值为0？


- sendMessage的内部实现：

```
- (COActorCompletable *)sendMessage:(id)message {
    
    COActorCompletable *completable = [COActorCompletable promise];
    dispatch_async(self.queue, ^{
        COActorMessage *actorMessage = [[COActorMessage alloc] initWithType:message completable:completable];
        [self.messageChan send_nonblock:actorMessage];
    });
    return completable;
}
```
sendMessage 内部其实是初始化了一个 Promise，然后再异步根据 promise 和 消息内容生成一个 COActorMessage，然后对这个 Actor 中的 channel 通过 send_nonblock 发送这个 message。

划重点：send_nonblock ！也就是说 sendMessage 并不会阻塞当前的代码/协程，所以如果直接通过获取 promise 的 value 值的时候，根本还没有值返回，所以得到为 0。

我们可以用上面提到过的 await 方法去获取这个 promise 的返回值。

？？？
1. 那么我们在初始化 actor 的时候去设置了这个 excute block，在 sendMessage 的时候往 channel 中 send 了消息。那，有关 receive 的代码呢？

在 ActorChannel 里面有一个 next 方法，其中调用了 receive 方法去处理这些消息。而 next 的调用就在 excute block 中 for 循环遍历时就调用了，如果此时没有消息 send 进来，就会中断等待消息。

#### actor并发模型的应用场景？
适合有状态或者称可变状态的业务场景,具体案例如订单，订单有状态，比如未付款未发货，已经付款未发货，已付款已发货，导致订单状态的变化是事件行为，比如付款行为导致顶大状态切换到"已经付款未发货"。

> 行为导致状态变化，行为执行是依靠线程，比如用户发出一个付款的请求，服务器后端派出一个线程来执行付款请求，携带付款的金额和银行卡等等信息，当付款请求被成功完成后，线程还要做的事情就是改变订单状态，这时线程访问订单的一个方法比如changeState。

> 如果后台有管理员同时修改这个订单状态，那么实际有两个线程共同访问同一个数据，这时就必须锁，比如我们在changeState方法前加上sychronized这样同步语法。

> 使用同步语法坏处是每次只能一个线程进行处理，如同上厕所，只有一个蹲坑，人多就必须排队，这种情况性能很低。

如何避免锁？

> 避免changeState方法被外部两个线程同时占用访问，那么我们自己设计专门的线程守护订单状态，而不是普通方法代码，普通方法代码比较弱势，容易被外部线程hold住，而我们设计的这个对象没有普通方法，只有线程，这样就变成Order的守护线程和外部访问请求线程的通讯问题了。

> Actor采取的这种类似消息机制的方式，实际在守护线程和外部线程之间有一个队列，俗称信箱，外部线程只要把请求放入，守护线程就读取进行处理。
 
### More

1. COTuple
2. cokit
3. coobjc 更底层的实现
4. 更多的使用场景
...

### Demo 下载地址
[coobjc learn Demo](https://github.com/shaqima123)
PS: 文中有关的问题大家可以通过运行 Demo 或者自己看源码跑一下寻找答案。有疑问的地方可以在评论区提问～

### 参考资料 && 扩展阅读

[刚刚，阿里开源 iOS 协程开发框架 coobjc！](https://mp.weixin.qq.com/s/hXmkd0TqTrCD-4kYlZcqvQ)

[什么是协程](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432090171191d05dae6e129940518d1d6cf6eeaaa969000)

[协程 生成器](https://blog.csdn.net/andybegin/article/details/77884645)

[actor并发模型&基于共享内存线程模型](https://www.jdon.com/45516)

[Actor模型和CSP模型的区别](https://www.jdon.com/concurrent/actor-csp.html)

[为什么Actor模型是高并发事务的终极解决方案？](https://blog.csdn.net/yongche_shi/article/details/51523661)

[goroutine, channel 和 CSP](http://www.moye.me/2017/05/05/go-concurrency-patterns/)