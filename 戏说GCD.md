#戏说GCD
    众所周知，iOS开发离不开多线程，在某些场合涉及比较复杂的多线程技巧。现在我想列几个假想的问题：  
1. 等会儿再行动。
2. 我走了，再见！
3. 我走了，你等我回来！
4. 分头行动，在A地集合。
5. 吃独食（你们吃的时候，我等着；我吃的时候，你们等着）。
6. 师兄弟分头找师傅（唐僧）。
7. 车满就出发。

本文的主要目标就是阐述上述几个问题的解决办法和处理技巧，如果觉得只是小case的，那我就不浪费你的时间了啊。
以上几个问题在实际项目中应该都不鲜见，至少1~6我都碰到过，第7个是假想的问题，但肯定会碰上的。

现在先卖个关子，说点闲话，再谈正事可好？
什么是GCD，以下摘自苹果的官方说明。
Grand Central Dispatch（GCD）分发队列(dispatch queue)是执行任务的强大工具。分发队列可根据调用者需要以同步或异步的方式执行任意数量的代码块（blocks of code)，你可以使用分发队列执行几乎全部的过去习惯用多线程来实现的任务。分发队列的优点是，**更加易用**，且比相应的用线程实现的代码 **更加高效**。
所以GCD应该被视为是iOS多任务的核心，以前采用NSThread实现的都应该往GCD迁移（或者用基于GCD实现的面向对向的NSOperationQueue完成）。

准备知识，分发队列(Dispatch Queue)是什么？
分发队列是一种简单地异步和并发地执行任务是方式，在代码层级上是类似于对象的结构体（object-like structure），管理着所有提交给它的任务。所有分发队列都是先进先出（FIFO）的数据结构。所以，提交给它的任务按提交的先后被执行。有如下3种分发队列：  

|       类型      |     描述     |  
| :-------------: | :------------- |  
| 串行(Serial)    | (单向)单车道（同一时刻只执行一个任务，执行完一个才执行下一个），单个线程的管理者 |  
| 并发(Concurrent)| (单向)多车道（可以同时执行多个任务，但仍按提交顺序启动），多个线程的管理者 |  
| 主分发队列(Main) | 特殊的串行队列，主线程的管理者 |  
这里需要说明的是，主分发队列确实值得拿出来作为一个单独的类型，NSThread、分发队列 和 操作队列，这三者仅在主队列上是相互兼容的，即代码可相互嵌套。
```
// 可以认为下述三个方法得到的是同一个线程，或者说是对同一个线程的封装。
[NSThread mainThread];
dispatch_get_main_queue();
[NSOperationQueue mainQueue];
// 下述三个方法仅在主分发队列的执行块中可相互嵌套，
[NSThread currentThread];
[NSOperationQueue currentQueue];
dispatch_get_current_queue(); // 废弃的方法，仅在主分发队列和global队列中有效，其他队列中返回nil
// 例如
dispatch_queue_t q0 = dispatch_get_main_queue();
dispatch_async(q0, ^{ // 若此处q0是自己create出来的队列，则下面打印出来的都是null了
    NSLog(@"current thread: %@", [NSThread currentThread]);
    NSLog(@"current operation queue: %@", [NSOperationQueue currentQueue]);
    NSLog(@"current dispatch queue: %@", dispatch_get_current_queue());
});
```

 ** 0. 创建队列**
```
// 这儿的队列名，强烈建议给有意义的名字，调试和分析崩溃日志时方便查看。
dispatch_queue_t serialQueue = dispatch_queue_create("com.tencent.byod.module1.serialQueue.1", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.tencent.byod.module1.concurrentQueue.0", DISPATCH_QUEUE_CONCURRENT);
dispatch_queue_t concurrentQueue1 = dispatch_queue_create("com.tencent.byod.module1.concurrentQueue.1", DISPATCH_QUEUE_CONCURRENT); // 预定义的并发队列
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_queue_t mainQueue = dispatch_get_main_queue(); 

```
1. 等会儿再行动
```
// 延迟0.5秒再执行
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), serialQueue, ^{
    // code to be executed in serialQueue after a specified delay
});
```
2. 我走了，再见！  
```
// 单个异步任务
dispatch_async(serialQueue, ^{
    // 要在queue上执行的代码
});
// 一次分发十个类似的异步任务，此处的queue若为串行，结果仍为顺序，  
dispatch_apply(10, queue, ^(size_t i) {
    NSLog(@"第%zd次执行", i);
});
// 如果queue为并发队列，可能的结果为：
// 第0次执行
// 第1次执行
// 第2次执行
// 第5次执行
// 第6次执行
// 第4次执行
// 第3次执行
// 第7次执行
// 第9次执行
// 第8次执行
```
3. 我走了，你等我回来！
```
// 注意死锁问题：如果queue是串行队列，在queue里执行这段代码会导致死锁（互相等待）。
// 解决办法也很简单：改用并发队列，或者在另一个队列里执行下面的代码，或者改用dispatch_async  
dispatch_sync(queue, ^{
    // 要在queue上执行的代码
});
```
4. 分头行动，在A地集合！
```
// 用任务组来完成
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, concurrentQueue, ^{
    [NSThread sleepForTimeInterval:2];
    NSLog(@"子任务1完成");
});
dispatch_group_async(group, concurrentQueue, ^{
    [NSThread sleepForTimeInterval:1];
    NSLog(@"子任务2完成");
});
// 好了，在这儿集合吧
dispatch_group_notify(group, mainQueue, ^{
    NSLog(@"报告首长：子任务1和2完成");
});
```
如果要执行的任务本身就是一个涉及多线程的复杂任务呢？有更灵活的办法：
```
// 更灵活的办法，
dispatch_group_t group1 = dispatch_group_create();
dispatch_group_enter(group1);
dispatch_async(globalQueue, ^{
    // do something
    dispatch_group_leave(group1);
});
dispatch_group_enter(group1);
dispatch_async(globalQueue, ^{
    // do something
    dispatch_group_leave(group1);
});
dispatch_group_notify(group1, mainQueue, ^{
    NSLog(@"报告老板：任务全部完成");
});
```
需要注意的是，dispatch_group_notify会阻塞其指定的serial queue，可以指定一个专门的waitQueue来解决这个问题，
```
dispatch_group_notify(group1, waitQueue, ^{
    dispatch_sync(mainQueue, ^{ // 主线程就不会阻塞了
        NSLog(@"报告老板：任务全部完成");
    });
});
```
5. 吃独食
```
@interface MyDemoClass : NSObject
@property (strong, nonatomic) NSMutableArray *books;
@property (strong, nonatomic) dispatch_queue_t queue;
- (NSString *)bookAtIndex:(NSUInteger)index;
- (void)insertBook:(NSString *)object atIndex:(NSUInteger)index;
- (void)addBook:(NSString *)object;
@end

@implementation MyDemoClass
- (instancetype)init {
    self = [super init];
    if (self) {
        _queue = dispatch_queue_create("com.tencent.byod.concurrentQueue.MyDemoClass", DISPATCH_QUEUE_CONCURRENT);
    }
    return self;
}
- (NSString *)bookAtIndex:(NSUInteger)index {
    __block NSString * book;
    dispatch_sync(_queue, ^{
        book = _books[index];
    });
    return book;
}
- (void)insertBook:(NSString *)object atIndex:(NSUInteger)index {
    dispatch_barrier_async(_queue, ^{
        _books[index] = object;
    });
}
- (void)addBook:(NSString *)object {
    dispatch_barrier_async(_queue, ^{
        [_books addObject:object];
    });
}
@end
```
6. 师兄弟分头找师傅（唐僧）
```
dispatch_async(serialQueue, ^{ // 为免阻塞主线程，另开一线程
    NSUInteger const count = 30000;
    NSMutableArray *arr = [NSMutableArray arrayWithCapacity:count];
    for (int i = 0; i < count; i++) {
        arr[i] = @(arc4random());
    }
    int tangseng = 1000; // 唐僧在此
    //arr[1000] = @(tangseng);
    // 数据准备妥当。开始分派任务，
    __block BOOL found = false;
    __block NSInteger indexOfTangseng = -1;
    dispatch_semaphore_t sema = dispatch_semaphore_create(0);
    for (int i = 0; i < 3; i++) { // 三个徒弟，兵分3路，各查找一个山头
        dispatch_async(globalQueue, ^{
            NSUInteger countOfPart = count / 3;
            for (NSUInteger j = i * countOfPart; j < (i+1) * countOfPart; j++) {
                if (found || tangseng == [arr[j] integerValue]) {
                    dispatch_semaphore_signal(sema);
                    if (!found) {
                        indexOfTangseng = j;
                        found = true;
                    }
                    break;
                }
            }
            dispatch_group_leave(group2);
        });
    }
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
    NSLog(@"found: %@, %zd", found ? @"true" : @"false", indexOfTangseng);
});
```
还有点小问题，如果师傅不在这3片山头，如何是好？那上面的代码中的dispatch_semaphore_wait语句就是永久阻塞。这显然不是我们想要的结果。得结合上述第4个问题的答案，在查找完后给个信儿。
```
dispatch_async(serialQueue, ^{
    NSUInteger const count = 3000;
    NSMutableArray *arr = [NSMutableArray arrayWithCapacity:count];
    for (int i = 0; i < count; i++) {
        arr[i] = @(arc4random());
    }
    int tangseng = 1000; // 唐僧在此
    // arr[1000] = @(tangseng);
    // 数据准备妥当。开始分派任务，
    __block BOOL found = false;
    __block NSInteger indexOfTangseng = -1;
    dispatch_group_t group2 = dispatch_group_create();
    dispatch_semaphore_t sema = dispatch_semaphore_create(0);
    for (int i = 0; i < 3; i++) { // 三个徒弟，兵分3路，各查找一个山头
        dispatch_group_enter(group2);
        dispatch_async(globalQueue, ^{
            NSUInteger countOfPart = count / 3;
            for (NSUInteger j = i * countOfPart; j < (i+1) * countOfPart; j++) {
                if (found || tangseng == [arr[j] integerValue]) {
                    dispatch_semaphore_signal(sema);
                    if (!found) {
                        indexOfTangseng = j;
                        found = true;
                    }
                    break;
                }
            }
            dispatch_group_leave(group2);
        });
    }
    dispatch_group_notify(group2, concurrentQueue, ^{
        dispatch_semaphore_signal(sema);
    });
    //dispatch_semaphore_wait(sema, dispatch_time(DISPATCH_TIME_NOW, 5));
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
    NSLog(@"found: %@, %zd", found ? @"true" : @"false", indexOfTangseng);
});
```
妖怪下令，5秒之内没找到就吃了你师傅。
可将上述DISPATCH_TIME_FOREVER改为dispatch_time(DISPATCH_TIME_NOW, 5)。
7. 车满就出发。
```
// 8人来应聘，但只要3个徒弟
size_t const countOfCandidate = 8;
size_t const countOfStudents = 3;
NSMutableArray *resultsOfAllCandidates = [NSMutableArray arrayWithCapacity:countOfCandidate];
dispatch_apply(countOfCandidate, globalQueue, ^(size_t i) { // 并发队列，阻塞当前线程
    dispatch_barrier_sync(concurrentQueue1, ^{
        [resultsOfAllCandidates addObject:@(i)];
    });
});
for (int i = 0; i < countOfStudents; i++) { // 录取的徒弟
    NSLog(@"%d: %@", i, resultsOfAllCandidates[i]);
}
```
上面的过程仍不够简洁，记录了全部参选人员的名次，事实上我只需要录用3个人。
```
// 8人来应聘，但只要3个徒弟
size_t const countOfCandidate = 8;
size_t const countOfStudents = 3;
NSMutableArray *resultsOfRace = [NSMutableArray arrayWithCapacity:countOfStudents];
dispatch_apply(countOfCandidate, concurrentQueue, ^(size_t i) { // 每人一条跑道
    __block BOOL stop = false;
    // runing，各跑10步
    for (int j = 0; j < 10; j++) {
        dispatch_sync(concurrentQueue1, ^{ // 阻塞当前线程
            if (resultsOfRace.count < countOfStudents) {
                NSLog(@"第%zu个选手跑第%d步", i, j+1);
            } else {
                stop = @(true); // 徒弟收完了，洗洗睡吧
            }
        });
        if (stop) {
            return; // 结束循环，并结束比赛（不继续记录比赛过程和结果）
        }
    }
    // 如果不是被完赛（即，不是因为选够了徒弟强制结束比赛）
    dispatch_barrier_sync(concurrentQueue1, ^{ // 阻塞当前线程
        [resultsOfRace addObject:@(i)];
        NSLog(@"第%zu个选手到达终点", i);
    });
});
for (int j = 0; j < resultsOfRace.count; j++) {
    NSLog(@"第%d个徒弟是第%@个候选人", j, resultsOfRace[j]);
}
```
