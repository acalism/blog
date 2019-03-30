# 大话西游之一种名叫GCD的法术
    话说唐僧自得观音菩萨指点，又得大唐皇帝赐酒祝福，骑着马，拿着禅杖，就踏上了漫漫取经路！路行十万八千里，历经九九八十一难，终于取得真经。
    八零后和九零后的小朋友听完这个故事，也都心潮澎湃。好，咱们今天就用iOS的GCD大法，来模拟一下唐僧师傅一路上可能遇到的困难。
    小可拍脑门一想，觉得很可能会碰到如下场景或困难（排名不分先后，仅依难易程度排列）：  
1. 歇一宿再走。
2. （师傅太啰嗦）俺老孙去也！
3. 师傅在此稍候，俺老孙去去便回！
4. 咱们分头行动（师傅坐船，我腾云），在河对岸集合。
5. 吃独食（徒儿们长相太寒碜，徒弟吃的时候，我不上桌，还没上桌的也等着；我吃的时候，还没吃的徒弟先候着）。
6. 师兄弟分头找师傅（唐僧）。
7. 我就招三个徒弟，先完成比赛的先录用，招满就踏上取经路。

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
这里需要说明的是，主分发队列确实值得拿出来作为一个单独的类型，NSThread、分发队列 和 操作队列，这三者仅在主队列上是相互兼容的（可以认为是同一实体），下面的代码反映这种兼容性。
```objc
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
```objc
// 这儿的队列名，强烈建议给有意义的名字，调试和分析崩溃日志时方便查看。
dispatch_queue_t serialQueue = dispatch_queue_create("com.tencent.byod.module1.serialQueue.1", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.tencent.byod.module1.concurrentQueue.0", DISPATCH_QUEUE_CONCURRENT);
dispatch_queue_t concurrentQueue1 = dispatch_queue_create("com.tencent.byod.module1.concurrentQueue.1", DISPATCH_QUEUE_CONCURRENT); // 预定义的并发队列
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_queue_t mainQueue = dispatch_get_main_queue();

```

1. 歇一宿再走  
唐僧一人一马，日近黄昏，人困马乏，“日暮苍山远，天寒白屋贫”（胡诌的），为师决定歇一宿再走，
```objc
// 延迟0.5秒再执行
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), serialQueue, ^{
    // code to be executed in serialQueue after a specified delay
});
```

2. （师傅太啰嗦）俺老孙去也！  
***顺便说下，下面api中带sync表示当前线程被阻塞，而async表示当前线程申通无阻，而block与queue的关系，则取决于queue本身是串行还是并发***
```objc
// 单个异步任务
dispatch_async(serialQueue, ^{
    // 要在queue上执行的代码
});
// 老孙能离开师傅，独我八戒沙僧不行？
// n个异步任务，此处的queue若为串行，结果仍为顺序，否则为随机序，  
dispatch_apply(10, queue, ^(size_t i) { // 阻塞当前线程
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

3. 师傅在此稍候，俺老孙去去便回！
```objc
// 注意***死锁问题***：如果queue是串行队列，在queue里执行这段代码会导致死锁（互相等待）。
// 解决办法也很简单：改用并发队列，或者在另一个队列里执行下面的代码，或者改用dispatch_async  
dispatch_sync(queue, ^{
    // 要在queue上执行的代码
});
```

4. 咱们分头行动（师傅坐船，我腾云），在河对岸集合！
```objc
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
    NSLog(@"报告师父：子任务1和2完成");
});
```
如果要执行的任务本身就是一个涉及多线程的复杂任务呢？有更灵活的办法：
```objc
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
    NSLog(@"报告师父：任务全部完成");
});
```
需要注意的是，dispatch_group_notify会阻塞其指定的serial queue，可以指定一个专门的waitQueue来解决这个问题，
```objc
dispatch_group_notify(group1, waitQueue, ^{ // 阻塞waitQueue
    dispatch_sync(mainQueue, ^{ // 主线程就不会阻塞了
        NSLog(@"报告师父：任务全部完成");
    });
});
```

5. 吃独食（徒儿们长相太寒碜，徒弟吃的时候，我不上桌，还没上桌的也等着；我吃的时候，还没吃的徒弟先候着）  
几个徒弟的看相，实在不敢恭维，想想师傅不情愿在饭桌上见到徒弟们，也是可以理解的啊（不要喷我啊，我只是这么假想了一下下而已）。  
***注：本为读写锁问题(读为并发，而写为独占)***  
```
@interface MyDemoClass : NSObject
@property (strong, nonatomic) NSMutableArray *books;
@property (strong, nonatomic) dispatch_queue_t queue;
 - (NSString *)bookAtIndex:(NSUInteger)index; //!< 读
 - (void)insertBook:(NSString *)object atIndex:(NSUInteger)index; //!< 写（插入）
 - (void)addBook:(NSString *)object; //!< 写（追加）
@end
//
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
    dispatch_sync(_queue, ^{ // 并发读，阻塞当前线程（要返回读出的数据啊）
        book = _books[index];
    });
    return book;
}
 - (void)insertBook:(NSString *)object atIndex:(NSUInteger)index {
    // 不必阻塞当前线程，故用async。
    // 这儿的barrier可以理解为，平时多车道并行不悖，但外国政要经过时要进行交通管制（barrier很贴切吧），暂变成单车道；政要们经过后，又变回多车道。
    dispatch_barrier_async(_queue, ^{
        _books[index] = object;
    });
}
 - (void)addBook:(NSString *)object {
  // 追加同insert，亦是独占式操作。
    dispatch_barrier_async(_queue, ^{
        [_books addObject:object];
    });
}
@end
```
6. 师兄弟分头找师傅（唐僧）  
师傅又被妖怪抓走了（你懂的），云深不知处，只在此3山。于是3个徒弟各找一片山区，誓要把师傅找到，且看徒弟们怎么找师傅，
```objc
dispatch_async(serialQueue, ^{ // 为免阻塞主线程，另开一线程
    NSUInteger const count = 30000;
    NSMutableArray *arr = [NSMutableArray arrayWithCapacity:count];
    for (int i = 0; i < count; i++) {
        arr[i] = @(arc4random());
    }
    int tangseng = 1000; // 唐僧在此
    //arr[1001] = @(tangseng);
    // 数据准备妥当。开始分派任务，
    __block BOOL found = false;
    __block NSInteger indexOfTangseng = -1;
    dispatch_semaphore_t sema = dispatch_semaphore_create(0);
    for (int i = 0; i < 3; i++) { // 三个徒弟，兵分3路，各查找一个山头
        dispatch_async(globalQueue, ^{
            NSUInteger countOfPart = count / 3;
            for (NSUInteger j = i * countOfPart; j < (i+1) * countOfPart; j++) {
                if (found || tangseng == [arr[j] integerValue]) {
                    dispatch_semaphore_signal(sema); // 找到师傅，则发出信号，且标记found为yes，告知师兄弟们不要再找了。
                    if (!found) { // 不曾有其他师兄弟找到
                        indexOfTangseng = j;  // 记录唐僧的位置
                        found = true;
                    }
                    break;
                }
            }
        });
    }
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
    NSLog(@"found: %@, %zd", found ? @"true" : @"false", indexOfTangseng);
});
```
还有点小问题，如果师傅不在这3片山头，如何是好？那上面的代码中的dispatch_semaphore_wait语句就是永久阻塞。这显然不是我们想要的结果。得结合上述第4个问题的答案，在查找完后给个信儿。
```objc
dispatch_async(serialQueue, ^{
    NSUInteger const count = 3000;
    NSMutableArray *arr = [NSMutableArray arrayWithCapacity:count];
    for (int i = 0; i < count; i++) {
        arr[i] = @(arc4random());
    }
    int tangseng = 1000; // 唐僧在此
    // arr[1001] = @(tangseng);
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
                    dispatch_semaphore_signal(sema); // 找到师傅：撤退方案A，
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
        dispatch_semaphore_signal(sema); // 未找到，撤退方案B
    });
    //dispatch_semaphore_wait(sema, dispatch_time(DISPATCH_TIME_NOW, 5)); // 如果限定搜山时长（从现在开始的5秒后），不然师傅要被煮熟吃掉啦
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
    NSLog(@"found: %@, %zd", found ? @"true" : @"false", indexOfTangseng);
});
```
妖怪下令，5秒之内没找到就吃了你师傅。
可将上述DISPATCH_TIME_FOREVER改为dispatch_time(DISPATCH_TIME_NOW, 5)。

7. 我就招三个徒弟，先完成比赛的先录用，招满就踏上取经路。  
唐僧觉得取经路太坎坷，妖怪众多，需招3个徒弟方可。由于唐僧是大唐皇帝的弟弟（临行时加封的也算好不，要不然女儿国国王怎么会称为“御弟哥哥”呢），所以应聘者甚众，得想个选拔办法。于是唐僧宣布，凡参赛者，同时开跑10km，先到终点者录用，录满（3人）即结束比赛。模拟如下：
```objc
// 记录全部结果的方案：
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
```objc
// 只记录前3名的方案：
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
                NSLog(@"第%zu个选手跑第%d千米", i, j+1);
            } else {
                stop = @(true); // 徒弟收满了，洗洗睡吧
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
