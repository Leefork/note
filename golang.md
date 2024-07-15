# 内存分配

## 存储

### 计算机的存储体系

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/8/5/16c6039e92b612da~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

计算机的存储体系如上图，从上至下访问速度略来越慢，依次是

1. CPU寄存器
2. CPU Cache
3. 内存
4. 硬盘等辅存
5. 鼠标等外接设备

CPU速度很快，如果直接从辅存读取数据，CPU的速度将会被拉慢，造成机器整体性能低下，所以在CPU和辅存间增加了内存

随着科技发展，内存与CPU速度差距变大，于是增加了Cache

### 操作系统对内存的管理-虚拟内存

为了向进程屏蔽内存和辅存，防止进程内存空间互相影响，于是OS提供了虚拟内存机制

- 操作系统将进程可访问的内存进行映射，虚拟内存地址映射向物理内存地址和磁盘地址
- 应用访问虚拟内存时，通过页表查看访问的虚拟内存是否已加载到物理内存，如果已加载则返回物理内存数据，否则将硬盘数据加载到物理内存，更新页表并返回
- 进程只能访问自己的虚拟内存空间，防止了多进程并发访问同一段内存。内存的并发访问问题降低为线程级别

### 进程对内存的管理

进程根据功能的不同将内存分为栈和堆

- 栈在高地址，从高地址向低地址增长
- 堆在低地址，从低地址向高地址增长

栈与堆相比有几个好处

- 栈的内存管理简单，分配比堆上快（连续空间分配）
- 栈的内存不需要回收，随着变量离开作用域自动回收（编译器进行管理）。而堆需要回收，花费额外CPU时间
- 栈上内存由于连续，有更好的局部性

## TCMalloc

由于虚拟内存的存在，内存的并发访问问题粒度由进程级别降低为线程级别，同一进程的多线程共享相同的内存空间，为了避免同一块内存被2个线程同时访问，为每个线程预分配一块缓存，线程申请小内存时，从线程缓存分配内存。称之为TCMalloc技术

好处：

- 从线程缓存申请小内存不需要进行系统调用，直接在用户态执行，缩短了内存总体的分配和释放时间
- 多个线程同时申请小内存时，由于都是从各自的缓存分配，访问的是不同空间，无需加锁。将内存的并发访问粒度进一步降低

### TCMalloc重要概念

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/8/5/16c60dc69d212e59~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

- Page
  - 操作系统对内存管理以页为单位，TCMalloc也是这样，不过TCMalloc里的Page大小与OS的不一定相等
- Span 内存块
  - 一组连续的Page成为Span，如2个页大小的Span、16个页大小的Span。
  - Span比页高一层级，主要是为了方便管理一定大小的内存区域，即内存块。
  - TCMalloc中内存管理的基本单位是Span
- ThreadCache
  - 线程缓存是每个线程各自的Cache，一个Cache包含多个空闲Span链表，用来组织空闲Span。
  - 同一链表上Span大小相同，这样申请内存时快速从合适的链表选择空闲Span
  - 每个线程有自己的TC，所以TC的访问不用加锁
- CentralCache
  - 中心缓存是所有线程共享的缓存，也是保存空闲内存块链表
  - 当ThreadCache中Span不足时，可以从CentralCache获取Span，并放置于TC空闲Span链表
  - 当TC的Span过多时，可以放会CC
  - CC是共享的，不同线程访问要加锁
- PageHeap
  - PageHeap页堆是对堆内存的抽象，PageHeap存的也是链表，链表保存的是Span
  - 当CC中内存不足时，会从PageHeap获取空闲的内存Span，将一个Span拆为若干Span，添加到空闲Span链表
  - 由于应用和CentralCache都可以访问，所以访问要加锁

TCMalloc将对象根据大小区分

- 小对象 0~256KB
- 中对象 257KB~1MB
- 大对象 1MB~

#### 小对象分配流程

ThreadCache -> CentralCache -> HeapCache

大部分时候，ThreadCache缓存都是足够的，不需要去访问CentralCache和HeapPage。这种情况下无系统调用和锁，分配效率很高

#### 中对象分配流程

直接在PageHeap选择适当大小，128Page的Span保存的最大内存就是1MB

#### 大对象分配流程

从large span set选择何时数量的页组成span

## Go内存管理

GO的内存管理模型源于TCMalloc，但多了两件东西

- 逃逸分析
- 垃圾回收

### 基本概念

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/8/5/16c61b772b1a8fb4~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

- Page
  - 与TCMalloc中的Page相同
- mspan
  - 与TCMalloc中的Span相同，是内存管理的基本单位
- mcache
  - 与TCMalloc中的ThreadCache类似
  - 小对象直接从mcache分配，无锁访问
  - 不同点是，TCMalloc是每个线程一个ThreadCache，go中是每个P一个mcache
- mcental
  - 与TCMalloc中的CentralCache类似，所有P共享，需要加锁访问
  - 不同是mcache每个p有两个链表
- mheap
  - 与TCMalloc中的PageHeap类似，访问要加锁
  - 不同点：mheap把Span组织成2颗树结构

# 内存回收GC

GC garbage collect垃圾回收，自动清理程序不再需要的内存，让开发人员只需关注业务，降低开发成本

GC中的概念：

- 垃圾回收
- 内存管理
- 自动释放
  - GC目的是自动释放不再使用的内存空间
- 三色标记法
  - 一种通用的GC方案
- STW 
  - stop the world
  - 为了防止并发操作对象，GC执行时需要暂停全部线程
  - 性能瓶颈之一

## Go1.3标记清除法

记录进程运行期间对象（即变量）及对象间的引用关系，程序与对象形成可达关系（类似图的数据结构）

程序生命周期中要循环执行GC，清理不可达对象的内存

标记清除法GC流程如下

1. 开始GC
2. STW
3. 遍历对象将可达对象做标记
4. 回收不可达对象的内存
5. STW结束
6. 结束GC

标记清除法缺点

- STW会暂停业务流程，让程序出现卡顿（重要问题！！
  - go1.3某个小版本将STW的结束提前到第4步，即标记完之后先STW再执行回收
- 标记需要扫描整个heap
- 记录数据会产生很多heap碎片

## Go1.5三色标记法

程序根节点集合RootSet维护了非引用创建的新对象集合

GC维护三个标记表，新建对象都放入白色标记表

- 白色标记表
- 灰色标记表
- 黑色标记表

三色标记法GC流程如下

1. 开始GC
2. 遍历RootSet，将遍历到的对象由白变灰
3. 遍历灰色标记表，将遍历到对象的可达对象由白变灰，然后将遍历到的对象由灰变黑
4. 重复上一步，直至灰色标记表中无任何对象
5. 收集白色标记表中对象内存
6. 结束GC

三色标记法假如不执行STW，GC过程中对象引用关系发生变化，如果变化满足以下两个条件，则会发生被引用的对象被清理

1. 一个白色对象被黑色对象引用
2. 白色对象不再被引用它的灰色对象所引用
3. 这个白色对象没有灰色对象引用它，会一直留在白色标记表，最后内存被回收

如何在不STW情况下防止上面的事情发生？破坏上面两个条件任意一个就可以

- 强三色不变式
  - 破坏条件1
  - 强制性的不允许黑色对象引用白色对象
- 弱三色不变式
  - 破坏条件2
  - 黑色对象可以引用白色对象，但要保证白色对象的可达链路上存在灰色对象，且上游的引用不要被破坏

## Go1.8三色标记法+混合写屏障

屏障机制来实现强弱三色不变式

屏障在程序执行过程中增加额外判断来实现屏障，类似hook、回调

- 插入屏障
  - 对象新增引用时触发（对象被引用时
  - 在对象被引用时，将被引用的对象标记为灰色
  - 用来满足强三色不变式，不存在黑色对象引用白色对象了，因为白色会被转为灰色
  - 为了不影响性能，不在栈上使用
  - 在回收白色前，**STW**并重新扫描一遍RootSet栈空间，重新执行三色标记
- 删除屏障
  - 对象被删除引用时触发
  - 对象引用被删除时，如果自身为白色，则标记为灰色
  - 用来满足弱三色不变式，引用被删除的对象保护自己的可达性
  - **回收精度比较低**，扫描过程中一个对象即使被删除了所有可达引用也可以活过这一轮
  - 上面描述的对象会在下次GC回收
- 混合写屏障
  1. GC开始时将栈可达对象全部扫描并标记为黑色（之后就不用第二次扫描，无需STW
  2. GC期间任何栈上引用的对象，都标记为黑色
  3. 删除屏障+插入屏障
     1. 被删除引用的对象标记为灰色（删除屏障，满足弱三色不变式
     2. 被添加引用的对象标记为灰色（插入屏障，满足强三色不变式

# 并发调度模型 GMP

## 调度模型由来分析

单进程时代两个问题

- 进程阻塞带来的CPU浪费
- 一个计算机只能处理一个任务

多进程多线程时代问题

- 进程线程切换成本大
- 线程间存在资源竞争
- 高内存、CPU占用

OS的线程模型是内核线程+用户线程（协程），并且是1比1的关系

应用程序可在用户态通过对一条内核线程上的协程进行切换，完成单条线程上的并发执行程序

## GO并发调度模型

Go的并发调度器模型是GMP，包含以下几个概念

- G goroutine go协程
  - 就是协程或者说用户态线程
- P processor 处理器
  - 保存goroutine运行的资源
  - 可通过GOMAXPROCS设置，默认为核心数
  - 每个P维护一个本地G队列，G不超过256个
  - P将队列中的G放在M上运行，程序可并行的goroutine数量就取决于P的数量
  - 运行中的G创建的G优先放在当前G的本地队列，本地队列满了就放全局队列
- M thread 内核态线程
  - 是真正运行在CPU上的程序
  - M的运行通过CPU调度器调度
  - 数量取决于当前操作系统分配到当前Go程序的内核线程数
  - M数量是动态的，当有一个M阻塞，则会创建一个新的M。如果M空闲，则会回收或唤醒
- G 全局队列
  - 全局goroutine队列
  - 访问加锁

## 设计策略

- 复用线程
  - work stealing机制
    - 当P中没有了G，线程空闲了，则会从全局队列或其他P的本地队列拿goroutine来执行，复用空闲线程
  - hand off机制
    - 当P中正在执行的G阻塞，P会唤醒或创建一个M并带领本地队列的其他G使用新的M继续执行程序
    - G不阻塞后，放入到全局队列
    - 旧M执行完后找空闲P绑定，如无则进入睡眠或回收
- 利用并行
  - 配置P的个数，利用多核CPU并行执行任务 
- 抢占
  - 其他程序的coroutine需要主动释放M，其他的coroutine才能执行
  - goroutine实行抢占策略，每个G设置了最高运行时间10ms，时间到后其他G抢占CPU（时间片）
  - 防止G被饿死
- 全局G队列
  - work stealing的补充，先从全局拿

## 场景分析

#### 创建G

1. 尝试唤醒正在休眠的M，将其与空闲P绑定
   1. 如果空闲P本地队列没有G，形成自旋线程，不断寻找G
   1. 自选线程从全局队列批量获取n个G（全局队列访问要加锁，批量性能好）获取后变为非自旋线程
   1. n = min(len(GQ)/P + 1, len(LQ/2)) 为了负载均衡
   1. 如果全局队列已满，从其他P队列获取G（work stealing），获取队列中后一半的G
2. 新创建的G优先加入当前P的本地队列中（局部性原理）
3. 加入前如果P本地队列满了
   1. 打乱队列头部G顺序，将新建的G加入其中，将它们一起并放入全局队列
   2. 队列尾部G会前移

#### G执行完毕

1. G执行goexit
2. M加载G0，G0执行schedule执行调度
3. M关联的P上本地队列的G被调度执行
4. 如P已空，M运行G0，与P组合成自旋线程。如果自旋线程 + 执行线程 > GOMAXPROCS，M休眠

#### G系统调用

1. G执行系统调用，进入阻塞
2. 则将G脱离P并于M绑定
3. P寻找休眠M并唤醒，P绑定新M继续执行，若没M则加入空闲P队列
4. G系统调用结束，进入非阻塞状态
5. G优先获取原来的P，将其绑定到M
6. 如果原来的P已绑定其他M，则获取空闲P
7. 如果无空闲P，则进入全局队列，M进入休眠

## go func生命周期

1. G被创建，初始化goroutine资源
2. G优先放入当前G所属P的本地队列中（局部性原理），如本地队列已满则放入全局队列中
3. M通过P获取G并运行
4. G如果时间片到且没执行完，则将G重新入P本地队列
5. G执行过程中，如果发生systemcall阻塞，则创建或唤醒一个M，并让新M接管P（hand off机制
6. 重复步骤3、4，直至G执行完

## 调度器生命周期

- M0
  - 启动程序后编号为0的主线程
  - 在全局变量runtime.m0中，不需要在heap上分配
  - 负责执行初始化操作和启动第一个G
  - 启动第一个G后，M0和其他M一样
- G0
  - 每启动一个M，都会创建第一个goroutine，成为该M的G0
  - 每个M都以一个G0与自己绑定，当调度
  - G0仅用于负责调度G，不执行函数
  - M执行调度或系统调用时，先切换到G0来调度

进程生命周期

1. 启动进程
2. 进程创建线程M0
3. M0创建G0，执行P和全局队列的初始化，并创建main的G

## GMP可视化

### trace

代码中开启trace

```go
//创建文件用于写入trace数据
f, err := os.Create("trace.out")
if err != nil {
    panic(err)
}
defer f.Close()

//启动trace
err = trace.Start(f)
if err != nil {
    panic(err)
}

//停止trace
trace.Stop()
```

用工具打开trace文件 go tool trace trace.out

### debug trace

debug流程

1. go build
2. set GODEBUG=schedtrace=1000
3. 运行程序

```
SCHED 0ms: gomaxprocs=12 idleprocs=11 threads=6 spinningthreads=0 needspinning=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
```

- sched 表示是调试信息，值为程序运行时间
- gomaxprocs P数量
- idleprocs 空闲P数量
- threads 开辟线程数量
- spinningthreads 自旋线程数量
- needspinning
- idlethreads 空闲线程数量
- runqueue 全局队列G数量
- [] P的本地队列G数量