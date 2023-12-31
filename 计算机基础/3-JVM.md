# ==JVM==

## JVM 内存结构

![|400](图集/image-20210220111553294-1690186272223-27.png)

线程**共享**区域：

- **堆**：存放**所有对象的实例和数组**。**GC 的位置**（**字符串常量池在堆里**）
- **方法区**：存储已被虚拟机加载的 **类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据**，以及**运行时常量池**。**1.8 之前为永久代实现**（运行时数据区域），**1.8 之后为元空间**（本地内存），原方法区被分成两部分；1：加载的类信息，2：运行时常量池；加载的类信息被保存在元数据区中，运行时常量池保存在堆中；

线程**私有**区域：

- **虚拟机栈**、**本地方法栈**：方法执行对应着一个**栈帧**的出栈和入栈。调用**native 方法**后，jvm**不会在虚拟机栈中为该线程创建栈帧**，而是**简单的动态链接并直接调用该方法**；
  - 栈帧中存放**局部变量表**（存放各种基本数据类型，对象引用（reference 类型））、**操作数栈** （计算中间结果，计算过程中产生的临时变量）、**动态连接**（将调用其他方法的符号引用转换为直接引用）、**方法返回地址**等信息。调整参数-xss 去调整 jvm 栈的大小。
  - **\*StackOverFlowError：** 若栈的内存大小**不允许动态扩展**，那么当线程请求栈的深度**超过当前 Java 虚拟机栈的最大深度**的时候，就抛出 `StackOverFlowError` 错误。
  - **\*OutOfMemoryError：** 如果栈的内存大小**可以动态扩展**， 如果虚拟机在**动态扩展栈时无法申请到足够的内存空间**，则抛出`OutOfMemoryError`异常。
- **程序计数器**：**随着线程的创建而创建**，随着线程的结束而死亡。（**唯一一个不会 OOM 的区域**）
  - **字节码解释器**通过**改变程序计数器来依次读取指令**，从而实现代码的**流程控制**，如：顺序执行、选择、循环、异常处理。
  - 在**多线程上下文切换**时，**程序计数器用于记录当前线程执行的位置**，从而当线程被切换回来的时候能够知道该线程**上次运行到哪儿**。

## 永久代替换元空间？

1. 整个**永久代**有一个 JVM 本身设置的**固定大小上限**，无法进行调整，而**元空间使用的是本地内存，受本机可用内存的限制**，虽然元空间**仍旧可能溢出**，但是比原来出现的**几率更小**。当**元空间溢出**时会报错：java.lang.**OutOfMemoryError: MetaSpace**

2. 元空间存的是**类的元数据**，这样加载多少类的元数据就**不由 `MaxPermSize` 控制了, 而由系统的实际可用空间来控制**，这样**能加载的类就更多**了。

3. 在 **JDK8**，**合并 HotSpot 和 JRockit 的代码**时, **JRockit 从来没有一个叫永久代的东西**, 合并之后就没有必要额外的设置这么一个永久代的地方了。

## JVM 内存模型

**Java 内存模型**（ **JMM**）就是在底层处理器内存模型的基础上，**定义自己的多线程语义**。它**明确指定了一组排序规则，来保证线程间的可见性**。

这一组规则被称为 **Happens-Before**， JMM 规定，要想保证 B 操作能够看到 A 操作的结果（无论它们是否在同一个线程），那么 A 和 B 之间必须满足 **Happens-Before 关系**，有以下几个规则：

- **单线程规则**：一个线程中的每个动作都 happens-before 该线程中后续的每个动作
- **监视器锁定规则**：监听器的**解锁**动作 happens-before 后续对这个监听器的**锁定**动作
- **volatile 变量规则**：对 volatile 字段的写入动作 happens-before 后续对这个字段的每个读取动作
- **线程 start 规则**：线程 **start()** 方法的执行 happens-before 一个启动线程内的任意动作
- **线程 join 规则**：一个线程内的所有动作 happens-before 任意其他线程在该线程 **join()** 成功返回之前
- **传递性**：如果 A happens-before B, 且 B happens-before C, 那么 A happens-before
  怎么理解 happens-before 呢？如果按字面意思，比如第二个规则，线程（不管是不是同一个）的解锁动作发生在锁定之前？这明显不对。happens-before 也是为了保证可见性，比如那个解锁和加锁的动作，可以这样理解，线程 1 释放锁退出同步块，线程 2 加锁进入同步块，那么线程 2 就能看见线程 1 对共享对象修改的结果。
  ![|398](图集/Pasted%20image%2020230822205541.png)

Java 提供了几种语言结构，包括 volatile，final 和 synchronized，它们旨在帮助程序员向**编译器**描述程序的并发要求，其中：

- **volatile** - 保证**可见性**和**有序性**
- **synchronized** - 保证**可见性**和**有序性**; 通过**管程（Monitor）\保证一组动作的\原子性**
- **final** - 通过禁止**在构造函数初始化**和**给 final 字段赋值**这两个动作的重排序，保证**可见性**（如果 **this 引用逃逸**就不好说可见性了）

编译器在遇到这些关键字时，**会插入相应的内存屏障，保证语义的正确性**。

有一点需要**注意**的是，**synchronized** **不保证**同步块内的代码禁止重排序，因为它通过锁保证同一时刻只有**一个线程**访问同步块（或临界区），也就是说同步块的代码只需满足 **as-if-serial** 语义 - 只要单线程的执行结果不改变，可以进行重排序。

所以说，**Java 内存模型描述的是多线程对共享内存修改后彼此之间的可见性**，另外，**还确保正确同步的 Java 代码可以在不同体系结构的处理器上正确运行**。

## 堆 heap 和栈 stack 的区别

1. **申请方式和效率**
   - **stack**:由**系统自动分配，速度较快**。例如，声明在函数中一个局部变量 int b; 系统自动在栈中为 b 开辟空间
   - **heap**:需要**程序员自己 new 并指明大小**，一般**速度比较慢**，而且**容易产生内存碎片**,不过**用起来最方便**。
2. **申请后系统的响应**
   - **stack**：只要**栈的剩余空间大于所申请空间**，系统将为程序提供内存，**否则将报栈溢出异常**。
   - **heap**：首先应该知道**操作系统有一个记录空闲内存地址的链表**，当系统收到程序的申请时，会遍**历该链表，寻找第一个空间大于所申请空间的堆结点**，然后将该结点**从空闲结点链表中删除**，并将**该结点的空间分配给程序**。另外，由于找到的堆结点的大小不一定正好等于申请的大小，**系统会自动的将多余的那部分重新放入空闲链表中**。
3. **申请大小的限制**
   - **stack**：栈是**向低地址扩展**的数据结构，是一块**连续的**内存的区域。这句话的意思是栈顶的地址和栈的最大容量是系统预先规定好的，在 **WINDOWS** 下，**栈的大小是 2M**（默认值也取决于虚拟内存的大小），**如果申请的空间超过栈的剩余空间时，将提示 overflow**。因此，**能从栈获得的空间较小**。
   - **heap**：堆是**向高地址扩展**的数据结构，是**不连续的**内存区域。这是由于系统是用链表来存储的空闲内存地址的， 自然是不连续的，而链表的遍历方向是由低地址向高地址。**堆的大小受限于计算机系统中有效的虚拟内存**。由此可见， **堆获得的空间比较灵活，也比较大**。
4. **申请效率的比较**
   - **stack**：由系统自动分配，速度较快。但程序员是无法控制的。
   - **heap**：由 new 分配的内存，一般速度比较慢，而且容易产生内存碎片,不过用起来最方便。
5. **存储内容**
   - **stack**：在**函数调用时**，**第一个进栈**的是**主函数中后的下一条指令（函数调用语句的下一条可执行语句）的地址**， 然后是函数的**各个参数**，在大多数的 C 编译器中，**参数是由右往左入栈的**，**然后是**函数中的**局部变量**。注意**静态变量是不入栈**的。
   - 当本次函数**调用结束**后，**局部变量先出栈**，然后是**参数**，**最后栈顶指针指向最开始存的地址，也就是主函数中的下一条指令，程序由该点继续运行**。
   - **heap**：一般是在**堆的头部用一个字节存放堆的大小**。**堆中的具体内容有程序员安排**。

## OOM 是什么，如何排查

**除了程序计数器**，其他内存区域都有 OOM 的风险。

- **栈**一般经常会发生 **StackOverflowError**，比如 32 位的 windows 系统单进程限制 2G 内存，无限创建线程就会发生栈的 OOM
- Java 8 常量池移到堆中，溢出会出 java.lang.OutOfMemoryError: Java heap space，设置最大元空间大小参数无效；
- **堆内存 OOM**，报错同上，这种比较好理解，GC 之后无法在堆中申请内存创建对象就会报错；
- **方法区 OOM**，经常会遇到的是动态生成大量的类、jsp 等；
- **直接内存 OOM**，涉及到 -XX:MaxDirectMemorySize 参数和 Unsafe 对象对内存的申请。

**排查 OOM** 的方法：

- 增加两个参数 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof，**当 OOM 时自动 dump 堆内存信息到指定目录**；
- 同时 **jstat 查看监控 JVM 的内存和 GC 情况**，先观察问题大概**出在什么区域**；
- **使用 MAT 工具载入到 dump 文件，分析大对象的占用情况**，比如 **HashMap 做缓存未清理**，时间长了就会内存溢出，可以把**改为弱引用** 。

## JVM 中常量池

JVM 常量池主要分为**Class 文件常量池、运行时常量池，全局字符串常量池，以及基本类型包装类对象常量池**。

- **Class 文件常量池**。class 文件是一组以字节为单位的二进制数据流，在 java 代码的编译期间，我们编写的 java 文件就被编译为.class 文件格式的二进制数据存放在磁盘中，其中就包括 class 文件常量池。
- **运行时常量池**：运行时常量池相对于 class 常量池一大特征就是具有动态性，java 规范并不要求常量只能在运行时才产生，也就是说运行时常量池的内容并不全部来自 class 常量池，在运行时可以通过代码生成常量并将其放入运行时常量池中，这种特性被用的最多的就是 String.intern()。
- **全局字符串常量池**：字符串常量池是 JVM 所维护的一个字符串实例的引用表，在 HotSpot VM 中，它是一个叫做 StringTable 的全局表。在字符串常量池中维护的是字符串实例的引用，底层 C++实现就是一个 Hashtable。这些被维护的引用所指的字符串实例，被称作”被驻留的字符串”或”interned string”或通常所说的”进入了字符串常量池的字符串”。
- **基本类型包装类对象常量池**：java 中基本类型的包装类的大部分都实现了常量池技术，这些类是 Byte,Short,Integer,Long,Character,Boolean,另外两种浮点数类型的包装类则没有实现。另外上面这 5 种整型的包装类也只是在对应值小于等于 127 时才可使用对象池，也即对象不负责创建和管理大于 127 的这些类的对象。

## 判断对象是否存活

**引用计数法**：
给每一个对象设置一个引用计数器，当有一个地方**引用该对象时就+1，引用失效时就-1**；当引用计数器**为 0 时**，就说明这个对象没有被引用，也就是**垃圾对象，等待回收**；
缺点：**无法解决循环引用**的问题，当 A 引用 B，B 也引用 A 的时候，此时 AB 对象的引用都不为 0，此时也就无法垃圾回收，所以**一般主流虚拟机都不采用**这个方法；

**可达性分析法**：
从一个被称为**GC Roots 的对象向下搜索**，**如果一个对象到 GC Roots 没有任何引用链相连接时，说明此对象不可用**，在 java 中可以作为**GC Roots 的对象有以下几种**：

- 虚拟机栈中引用的对象
- 方法区类静态属性引用的变量
- 方法区常量池引用的对象
- 本地方法栈 JNI 引用的对象

但一个对象满足上述条件的时候，**不会马上被回收，还需要进行两次标记**；

第一次标记：**判断当前对象是否有 finalize()方法并且该方法没有被执行过**，**若不存在**则标记为垃圾对象，**等待回收**；若有的话，则进行第二次标记；

第二次标记：将当前对象**放入 F-Queue 队列，并生成一个 finalize 线程去执行该方法**，虚拟机不保证该方法一定会被执行，这是因为如果线程执行缓慢或进入了死锁，会导致回收系统的崩溃；**如果执行了 finalize 方法之后仍然没有与 GC Roots 有直接或者间接的引用，则该对象会被回收**；

## 强引用/软引用/弱引用/虚引用

- **强引用**：就是**普通的对象引用**关系，如 String s = new String("ConstXiong")
- **软引用**：用于**维护一些可有可无的对象**。只有在**内存不足时，系统则会回收软引用对象**，如果回收了软引用对象之后仍然没有足够的内存，才会抛出内存溢出异常。**SoftReference 实现**
- **弱引用**：相比软引用来说，要更加无用一些，它**拥有更短的生命周期**，当 JVM 进行垃圾回收时，**无论内存是否充足，都会回收被弱引用关联的对象**。**WeakReference 实现**
- **虚引用**是一种形同虚设的引用，在现实场景中用的不是很多，它主要用来**跟踪对象被垃圾回收的活动**。**PhantomReference 实现**

**被引用的对象不一定能存活吗？**

看 引用类型，**弱引用在 GC 时会被回收**，**软引用在内存不足的时候**，即 OOM 前会被回收，但如果没有在 Reference Chain 中的对象就一定会被回收。

## 垃圾回收算法 GC

java 中有四种垃圾回收算法，分别是标记清除法、标记整理法、复制算法、分代收集算法；
**1.标记清除法**：
第一步：**利用可达性去遍历内存，把存活对象和垃圾对象进行标记**；
第二步：在遍历一遍，**将所有标记的对象回收掉**；
特点：**效率不行，标记和清除的效率都不高**；标记和清除后会**产生大量的不连续的空间分片**，可能会导致之后程序运行的时候**需分配大对象而找不到连续分片而不得不触发一次 GC**；

![](图集/image-20210220111918592.png)

**2.标记整理法**：
第一步：利用可达性去遍历内存，把存活对象和垃圾对象进行标记；
第二步：将所有的**存活的对象向一段移动**，将**端边界以外的对象都回收掉**；
特点：适用于**存活对象多**，**垃圾少**的情况；需要整理的过程，**无空间碎片产生**；

![](图集/image-20210220111933505.png)

**3.复制算法**：
将内存按照容量大小**分为大小相等的两块**，每次只使用一块，当一块使用完了，**就将还存活的对象移到另一块上，然后在把使用过的内存空间移除**；
特点：**不会产生空间碎片**；内存使用率**极低**；

**4.分代收集算法**：
根据内存对象的存活周期不同，将内存划分成几块：

- **新生代中，有大量对象死去和少量对象存活，所以采用复制算法**，只需要付出**少量存活对象的复制成本**就可以完成收集；

- **老年代中因为对象的存活率极高，没有额外的空间对他进行分配担保**，所以采用**标记清理**或者**标记整理算法**进行回收；

![image-20210329224002527](./图集/image-20210329224002527.png)

## 垃圾回收器

**1.Serial**:**单线程**的收集器，收集垃圾时，必须 stop the world，使用引用。它的最大特点是在**进行垃圾回收时，需要对所有正在执行的线程暂停**（stop the world），对于有些应用是难以接受的，但是如果应用的**实时性要求不高**，只要停顿的时间控制在 N 毫秒之内，大多数应用还是可以接受的，是**client 级别的默认 GC 方式**。

**2.ParNew**：**Serial 收集器的多线程版本**，也需要 stop the world，**复制算法**

**3.Parallel Scavenge**:新生代收集器，**复制算法**的收集器，并发的多线程收集器，目标是达到一个可控的吞吐量，**和 ParNew 的最大区别是 GC 自动调节策略**；虚拟机会**根据系统的运行状态收集性能监控信息，动态设置这些参数**，以提供最优停顿时间和最高的吞吐量；

**4.Serial Old**:**Serial 收集器的老年代版本**，**单线程**收集器，使用**标记整理算法**。

**5.Parallel Old**：是**Parallel Scavenge 收集器的老年代版本**，使用**多线程**，**标记整理算法**。

**6.CMS**:是一种**以获得最短回收停顿时间为目标**的收集器，**标记清除算法**，运作过程：初始标记，并发标记，重新标记，并发清除，**收集结束会产生大量空间碎片**；

**7.G1**:**标记整理算法**实现，运作流程主要包括以下：初始标记，并发标记，最终标记，筛选回收。**不会产生空间碎片，可以精确地控制停顿**；G1 将整个堆分为大小相等的多个 Region（区域），G1 跟踪每个区域的垃圾大小，在后台维护一个优先级列表，每次根据允许的收集时间，优先回收价值最大的区域，已达到在有限时间内获取尽可能高的回收效率；

**垃圾回收器间的配合使用图：**

![|450](图集/image-20210329224424220.png)

**各个垃圾回收器对比**：

![|675](./图集/image-20210329230618579.png)

## 垃圾收集器

Serial: **单线程**，工作在新生代 采用**标记-复制**算法

Serial Old：**单线程**，工作在老年代 采用**标记-整理**算法

ParNew：**并行收集**，新生代采用标记-复制，老年代采用标记-整理。

Parallel Scavenge：于 ParNew 类似。注重于**吞吐量**。新生代标记-复制。老年代标记-整理。

$$
吞吐量 = 运行用户代码时间/（运行用户代码时间+垃圾收集时间）
$$

有自适应调节参数功能（比如各个分区的比例，晋升老年代对象阈值等参数）。

CMS：注重系统停顿时间短。（**只作用于老年代，采用标记-清除**）

过程：
​ 1.初始标记（停顿用户线程）
​ 2.并发标记
​ 3.重新标记（停顿用户线程）
​ 4.并发清除

缺点：
​ 1.**对 CPU 资源敏感**：垃圾收集会占用一部分线程（CPU）导致用户程序变慢（虽然不会停顿）。
​ 2.**无法处理浮动垃圾**：并发清除阶段由于是于用户线程并发进行，会产生新的垃圾，无法在当次清除中清理掉，只能留在下一次清理中清除。而且 CMS 不能等到老年代快被占满了才触发。
​ 3.**会产生大量空间碎片**：先标记-清理 实在不行了再标记-整理。

G1：基于**Region**内存布局。（Humongous Region 存放大对象）

​ 每次根据用户设定的**收集停顿时间**，优先回收**价值收益最大**的区域（后台维护的优先级列表）。
过程：
​ 1.初始标记（停顿用户线程）
​ 2.并发标记
​ 3.最终标记（停顿用户线程）
​ 4.筛选回收（停顿用户线程）

CMS 于 G1 区别：

​ 1.不会产生内存空间碎片（整体上是标记-整理，局部上是标记-复制）。--G1 好处
​ 2.占用内存上，G1 对于记忆集的维护开销更大（扫描 GCRoots 时，记录跨 Region 引用）。
​ 3.执行负载上，G1 消耗更大，在并发标记阶段，跟踪引用的变化。

## CMS

CMS(Concurrent Mark Sweep，**并发标记清除**) 收集器是**以获取最短回收停顿时间为目标**的收集器（**追求低停顿**），它在垃圾收集时**使得用户线程和 GC 线程并发执行**，因此在垃圾收集过程中用户也不会感到明显的卡顿。

从名字就可以知道，CMS 是基于“标记-清除”算法实现的。CMS 回收过程分为以下四步：

1. **初始标记** （CMS initial mark)：主要是标记 GC Root 开始的下级（注：仅下一级）对象，这个过程会 STW，但是跟 GC Root 直接关联的下级对象不会很多，因此这个过程其实很快。
2. **并发标记** (CMS concurrent mark)：根据上一步的结果，继续向下标识所有关联的对象，直到这条链上的最尽头。这个过程是多线程的，虽然耗时理论上会比较长，但是其它工作线程并不会阻塞，没有 STW。
3. **重新标记**（CMS remark）：顾名思义，就是要再标记一次。为啥还要再标记一次？因为第 2 步并没有阻塞其它工作线程，其它线程在标识过程中，很有可能会产生新的垃圾。
4. **并发清除**（CMS concurrent sweep）：清除阶段是清理删除掉标记阶段判断的已经死亡的对象，由于不需要移动存活对象，所以这个阶段也是可以与用户线程同时并发进行的。

**CMS 的问题：**

**1. 并发回收导致 CPU 资源紧张：**

在并发阶段，它虽然不会导致用户线程停顿，但却会因为占用了一部分线程而导致应用程序变慢，降低程序总吞吐量。CMS 默认启动的回收线程数是：（CPU 核数 + 3）/ 4，当 CPU 核数不足四个时，CMS 对用户程序的影响就可能变得很大。

**2. 无法清理浮动垃圾：**

在 CMS 的并发标记和并发清理阶段，用户线程还在继续运行，就还会伴随有新的垃圾对象不断产生，但这一部分垃圾对象是出现在标记过程结束以后，CMS 无法在当次收集中处理掉它们，只好留到下一次垃圾收集时再清理掉。这一部分垃圾称为“浮动垃圾”。

**3. 并发失败（Concurrent Mode Failure）：**

由于在垃圾回收阶段用户线程还在并发运行，那就还需要预留足够的内存空间提供给用户线程使用，因此 CMS 不能像其他回收器那样等到老年代几乎完全被填满了再进行回收，必须预留一部分空间供并发回收时的程序运行使用。默认情况下，当老年代使用了 92% 的空间后就会触发 CMS 垃圾回收，这个值可以通过 -XX**:** CMSInitiatingOccupancyFraction 参数来设置。

这里会有一个风险：要是 CMS 运行期间预留的内存无法满足程序分配新对象的需要，就会出现一次“并发失败”（Concurrent Mode Failure），这时候虚拟机将不得不启动后备预案：Stop The World，临时启用 Serial Old 来重新进行老年代的垃圾回收，这样一来停顿时间就很长了。

**4.内存碎片问题：**

CMS 是一款基于“标记-清除”算法实现的回收器，这意味着回收结束时会有内存碎片产生。内存碎片过多时，将会给大对象分配带来麻烦，往往会出现老年代还有很多剩余空间，但就是无法找到足够大的连续空间来分配当前对象，而不得不提前触发一次 Full GC 的情况。

为了解决这个问题，CMS 收集器提供了一个 -XX**:**+UseCMSCompactAtFullCollection 开关参数（默认开启），用于在 Full GC 时开启内存碎片的合并整理过程，由于这个内存整理必须移动存活对象，是无法并发的，这样停顿时间就会变长。还有另外一个参数 -XX**:**CMSFullGCsBeforeCompaction，这个参数的作用是要求 CMS 在执行过若干次不整理空间的 Full GC 之后，下一次进入 Full GC 前会先进行碎片整理（默认值为 0，表示每次进入 Full GC 时都进行碎片整理）。

## G1

G1（Garbage First）回收器采用面向局部收集的设计思路和基于 Region 的内存布局形式，是一款主要面向服务端应用的垃圾回收器。G1 设计初衷就是替换 CMS，成为一种全功能收集器。G1 在 JDK9 之后成为服务端模式下的默认垃圾回收器，取代了 Parallel Scavenge 加 Parallel Old 的默认组合，而 CMS 被声明为不推荐使用的垃圾回收器。G1 从整体来看是基于 标记-整理 算法实现的回收器，但从局部（两个 Region 之间）上看又是基于 标记-复制 算法实现的。

**G1 回收过程**，G1 回收器的运作过程大致可分为四个步骤：

1. **初始标记**（**会 STW**）：仅仅只是标记一下 GC Roots 能直接关联到的对象，并且修改 TAMS 指针的值，让下一阶段用户线程并发运行时，能正确地在可用的 Region 中分配新对象。这个阶段需要停顿线程，但耗时很短，而且是借用进行 Minor GC 的时候同步完成的，所以 G1 收集器在这个阶段实际并没有额外的停顿。
2. **并发标记**：从 GC Roots 开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户程序并发执行。当对象图扫描完成以后，还要重新处理在并发时有引用变动的对象。
3. **最终标记**（**会 STW**）：对用户线程做短暂的暂停，处理并发阶段结束后仍有引用变动的对象。
4. **清理阶段**（**会 STW**）：更新 Region 的统计数据，对各个 Region 的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个 Region 构成回收集，然后把决定回收的那一部分 Region 的存活对象复制到空的 Region 中，再清理掉整个旧 Region 的全部空间。这里的操作涉及存活对象的移动，必须暂停用户线程，由多条回收器线程并行完成的。

## 一次完整 GC

在 Java 中，堆（**大小-XMX/-XMS 指定**）被划分成两个不同的区域：**新生代** ( Young )、**老年代** ( Old )，新生代：老年代**默认 1:2**。
**新生代有 3 个分区：Eden、From、To**，它们的默认占比是 **8:1:1**。

**新生代**的垃圾回收（又称 Minor GC）后**只有少量对象存活**，所以选用**复制算法**，只需要少量的复制成本就可以完成回收。

**老年代**的垃圾回收（又称 Major GC）通常使用“**标记-清理**”或“**标记-整理**”算法。

![](./图集/image-20210329225348086.png)

再描述它们之间转化流程：

- 对象**优先在 Eden 分配**。当 eden 区**没有足够空间进行分配**时，虚拟机将发起一次 **Minor GC**。
  - 在 Eden 区执行了第一次 GC 之后，**存活的对象会被移动**到其中一个 **Survivor 分区**；
  * Eden 区**再次 GC**，这时会采用**复制算法**，将 Eden 和 from 区一起清理，**存活的对象会被复制到 to 区**；
  * **移动一次，对象年龄加 1**，对象年龄大于**一定阀值**会**直接移动到老年代**。GC 年龄的阀值可以通过参数 **-XX:MaxTenuringThreshold 设置，默认为 15**；
  * **动态对象年龄判定**：Survivor 区**相同年龄所有对象大小的总和** > **Survivor 区内存大小这个目标使用率**时，**大于或等于该年龄**的对象直接进入老年代。其中这个使用率通过 **-XX:TargetSurvivorRatio 指定，默认为 50%**；
  * Survivor 区内存不足会发生担保分配，**超过指定大小的对象可以直接进入老年代**。
- **大对象直接进入老年代**，大对象就是**需要大量连续内存空间**的对象（比如：**字符串、数组**），为了**避免为大对象分配内存时由于分配担保机制带来的复制而降低效率**。
- 老年代满了而**无法容纳更多的对象**，Minor GC 之后通常就会进行 Full GC，Full GC 清理整个内存堆 – **包括年轻代和老年代**。

## Minor GC/Full GC

**Minor GC**：只收集**新生代**的 GC。当**Eden 区满时触发**

**Full GC**: 收集**整个堆**，包括 **新生代，老年代，永久代**(在 JDK1.8 后，永久代被移除，换为**metaspace 元空间**)等所有部分的模式。

**Full GC 触发条件**：

- **通过 Minor GC 后进入老年代的平均大小大于老年代的可用内存**。如果发现统计数据说之前 Minor GC 的平均晋升大小比目前 old gen 剩余的空间大，则不会触发 Minor GC 而是转为触发 full GC。
- **老年代空间不够分配新的内存**（或永久代空间不足，但只是 JDK1.7 有的，这也是用元空间来取代永久代的原因，可以减少 Full GC 的频率，减少 GC 负担，提升其效率）。
- 由 Eden 区、From Space 区向 To Space 区复制时，对象大小大于 To Space 可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小。
- 调用 System.gc 时，系统建议执行 Full GC，但是不必然执行。

## 空间分配担保原则

**如果 YougGC 时新生代有大量对象存活下来，而 survivor 区放不下了，这时必须转移到老年代中，但这时发现老年代也放不下这些对象了**，那怎么处理呢？其实 JVM 有一个**老年代空间分配担保机制来保证对象能够进入老年代**。

在执行每次 YoungGC 之前，JVM 会先检查老年代最大可用连续空间是否大于新生代所有对象的总大小。因为在极端情况下，可能新生代 YoungGC 后，所有对象都存活下来了，而 survivor 区又放不下，那可能所有对象都要进入老年代了。这个时候如果老年代的可用连续空间是大于新生代所有对象的总大小的，那就可以放心进行 YoungGC。但如果老年代的内存大小是小于新生代对象总大小的，那就有可能老年代空间不够放入新生代所有存活对象，这个时候 JVM 就会先检查 -XX:HandlePromotionFailure 参数是否允许担保失败，如果允许，就会判断老年代最大可用连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试进行一次 YoungGC，尽快这次 YoungGC 是有风险的。如果小于，或者 -XX:HandlePromotionFailure 参数不允许担保失败，这时就会进行一次 Full GC。

在允许担保失败并尝试进行 YoungGC 后，可能会出现三种情况：

- ① YoungGC 后，存活对象小于 survivor 大小，此时存活对象进入 survivor 区中
- ② YoungGC 后，存活对象大于 survivor 大小，但是小于老年大可用空间大小，此时直接进入老年代。
- ③ YoungGC 后，存活对象大于 survivor 大小，也大于老年大可用空间大小，老年代也放不下这些对象了，此时就会发生“**Handle Promotion Failure**”，就**触发了 Full GC**。**如果 Full GC 后，老年代还是没有足够的空间**，此时就会发生**OOM**内存溢出了。

通过下图来了解**空间分配担保原则**：

![|525](./图集/image-20210329230240201.png)

## ==类加载==

### 类加载过程

![|475](图集/Pasted%20image%2020230822210748.png)

1.**加载**： 1. 通过**全类名**获取定义此类的**二进制字节流**。 2. 将**字节流**所代表的**静态存储结构**转换为**方法区**的**运行时数据结构**。 3. 在**内存中生成**一个代表该类的**Class 对象**，作为方法区这些数据的**访问入口**。 2.**验证**：确保**Class**文件的**字节流中**包含的信息**符合规范**，**不会危害虚拟机**。 3.**准备**：**静态变量分配内存**赋默认值，**final 常量**赋初值 4.**解析**：将常量池里的**符号引用替换为直接引用/地址引用**。**符号引用**就是一组符号来描述目标，可以是任何字面量。**直接引用**就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。
**在某些情况下可以在初始化阶段之后**，这是**为了支持 Java 语言的运行时绑定**（也称为**动态绑定或晚期绑定**） 5.**初始化**：执行**静态代码块**，**赋值语句**，为**静态变量赋值**，**调用类构造器**。

### **类初始化时机**

对于初始化阶段，虚拟机严格规范了有且只有 6 种情况下，必须对类进行初始化(只有主动去使用类才会初始化类)：

1. 当遇到 `new`、 `getstatic`、`putstatic` 或 `invokestatic` 这 4 条字节码指令时，比如 `new` 一个类，读取一个静态字段(未被 final 修饰)、或调用一个类的静态方法时。
   - 当 jvm 执行 `new` 指令时会初始化类。即当程序创建一个类的实例对象。
   - 当 jvm 执行 `getstatic` 指令时会初始化类。即程序访问类的静态变量(不是静态常量，常量会被加载到运行时常量池)。
   - 当 jvm 执行 `putstatic` 指令时会初始化类。即程序给类的静态变量赋值。
   - 当 jvm 执行 `invokestatic` 指令时会初始化类。即程序调用类的静态方法。
2. 使用 `java.lang.reflect` 包的方法对类进行反射调用时如 `Class.forname("...")`, `newInstance()` 等等。如果类没初始化，需要触发其初始化。
3. 初始化一个类，如果其父类还未初始化，则先触发该父类的初始化。
4. 当虚拟机启动时，用户需要定义一个要执行的主类 (包含 `main` 方法的那个类)，虚拟机会先初始化这个类。
5. `MethodHandle` 和 `VarHandle` 可以看作是轻量级的反射调用机制，而要想使用这 2 个调用， 就必须先使用 `findStaticVarHandle` 来初始化要调用的类。
6. **「补充，来自[issue745open in new window](https://github.com/Snailclimb/JavaGuide/issues/745 "issue745")」** 当一个接口中定义了 JDK8 新加入的默认方法（被 default 关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

构造子类时，**父类 static**{} > **子类 static**{} > **父类方法块**{} > **父类构造方法** > **子类方法块** > **子类构造器**

### 类加载器

类加载器是指：**通过一个类的全限定性类名获取该类的二进制字节流**；以下四种：

- **启动类加载器**（BootStrapClassLoader）：用来加载**java 核心类库**，无法被 java 程序直接引用；
- **扩展类加载器**（Extension ClassLoader）：用来加载**java 的扩展库**，java 的虚拟机实现会提供一个扩展库目录，该类加载器在扩展库目录里面查找并加载 java 类；
- **系统/应用类加载器**（AppClassLoader）：它**根据 java 的类路径来加载类**，一般来说，**java 应用的类**都是通过它来加载的；
- **自定义类加载器**：由 java 语言实现，**继承自 ClassLoader**；

![|425](图集/image-20210329231439914.png)

### 自定义类加载器方法及应用场景

**场景**：比如一个项目中，**多个模块依赖同一个 jar 包的不同版本**，比如**fastJSON**这种，有的依赖 1.0 有的依赖 2.0；可能 maven 机制下最终只依赖 1.0，那么就会报方法错误，此时就需要自定义类加载器，将不同版本的类加载进来。

**方法**：**继承 ClassLoader**，如果我们**不想打破**双亲委派模型，就**重写 ClassLoader 类中的 findClass**()方法即可，无法被父类加载器加载的类最终会通过这个方法被加载。如果**想打破**双亲委派模型则需要**重写 loadClass**()方法。

**Tomcat 打破了双亲委派机制**，**重写 loadClass**（）方法。

### 双亲委派模型/如何打破

当一个类加载器收到一个类加载的请求，他首先不会尝试自己去加载，而是将这个请求**委派给父类加载器去加载**，只有**父类加载器在自己的搜索范围类查找不到给类时，子加载器才会尝试自己去加载该类**；

**为什么**：为了防止内存中出现多个相同的字节码；因为如果没有双亲委派的话，用户就可以自己定义一个 java.lang.String 类，那么就无法保证类的唯一性。

- **保证核心 API 不会被篡改**。
- **不会重复加载同一个类**。

![](图集/W2BBU55`X}[LL6{@T{$QCGH-1685100063021.png)

补充：**那怎么打破双亲委派模型**？
**自定义类加载器**，**继承 ClassLoader 类**，**重写 loadClass 方法和 findClass 方法**。

### 打破双亲委派机制的例子

- JNDI 通过引入线程上下文类加载器，可以在 Thread.setContextClassLoader 方法设置，默认是应用程序类加载器，来加载 SPI 的代码。有了线程上下文类加载器，就可以完成父类加载器请求子类加载器完成类加载的行为。打破的原因，是为了 JNDI 服务的类加载器是启动器类加载，为了完成高级类加载器请求子类加载器（即上文中的线程上下文加载器）加载类。

- Tomcat，应用的类加载器优先自行加载应用目录下的 class，并不是先委派给父加载器，加载不了才委派给父加载器。

  tomcat 之所以造了一堆自己的 classloader，大致是出于下面三类目的：

  - 对于各个 `webapp`中的 `class`和 `lib`，需要相互隔离，不能出现一个应用中加载的类库会影响另一个应用的情况，而对于许多应用，需要有共享的 lib 以便不浪费资源。
  - 与 `jvm`一样的安全性问题。使用单独的 `classloader`去装载 `tomcat`自身的类库，以免其他恶意或无意的破坏；
  - 热部署。

  tomcat 类加载器如下图：

  ![|475](图集/image-20210329231930719.png)

- OSGi，实现模块化热部署，为每个模块都自定义了类加载器，需要更换模块时，模块与类加载器一起更换。其类加载的过程中，有平级的类加载器加载行为。打破的原因是为了实现模块热替换。

- JDK 9，Extension ClassLoader 被 Platform ClassLoader 取代，当平台及应用程序类加载器收到类加载请求，在委派给父加载器加载前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要优先委派给负责那个模块的加载器完成加载。打破的原因，是为了添加模块化的特性。

### 双亲委派机制

![](图集/YLWH8ULNU`9QYZ{9UO3T0XX.png)

1.**BootstrapClassLoader(启动类加载器)** ：最顶层的加载类，由 C++实现，负责加载 `%JAVA_HOME%/lib`目录下的 jar 包和类或者被 `-Xbootclasspath`参数指定的路径中的所有类。

2.**ExtensionClassLoader(扩展类加载器)** ：主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类，或被 `java.ext.dirs` 系统变量所指定的路径下的 jar 包。

3.**AppClassLoader(应用程序类加载器)** ：面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。

4.**User Class Loader(自定义类加载器)**。

```java
private final ClassLoader parent;
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查请求的类是否已经被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//父加载器不为空，调用父加载器loadClass()方法处理
                        c = parent.loadClass(name, false);
                    } else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //抛出异常说明父类加载器无法完成加载请求
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                    //自己尝试加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

## ==对象==

### 对象生命周期

创建阶段: 分配空间，构造对象，从父类到子类静态成员初始化，父类成员变量初始化，子类成员变量初始化
对象引用阶段: 创建后被分派给某些变量赋值
不可见阶段: 程序不再持有该对象任何强引用 超出该对象作用域（但 GC Root 还持有引用）
不可达阶段: 对象本身不再被任何强引用（GC ROOT 也不持有）
对象收集阶段:垃圾回收器发现对象不可达，为内存重新分配做好准备进入收集阶段，执行对象 finalize（）方法  
对象终结阶段:执行完 finalize（）仍然不可达，等待垃圾回收器回收。  
对象空间重新分配

### 创建对象过程

1.检查该对象所属的类知否被加载，若没有则先加载这个类。 2.为对象分配内存：
​ 方式一：指针碰撞（内存空间规整）
​ 方式二：空闲列表（内存空间不规整）
​ 多线程创建对象同步方式：
​ 1）CAS+失败重试
​ 2）TLAB。每个线程中的缓冲区中分配 3.对象实例字段初始化零值。保证对象的实例字段在不赋值就能直接使用。 4.设置对象头。 5.通过构造函数，为对象字段赋值。

### 对象内存布局

1.**对象头**

- **Mark Word**（64bit）：存储自身的运行时数据（哈希码、GC 分代年龄、锁状态标志、偏向线程 ID、偏向时间戳等）。
- **类型指针**：用于确定是哪个类的实例。 2.**实例数据**：各个字段的内容。 3.**对齐填充**：因为虚拟机要求对象的**起始地址**必须是**8 字节的整数倍**。

### 判断对象是否存活

方法一：**引用计数法**。
方法二：**可达性分析算法**。

**GC Roots 对象**：

- **虚拟机栈、本地方法栈**中引用的对象。
- **方法区**中的**静态变量**、**常量引用**的对象。
- **被同步锁持有**的对象。
- **基本数据类型**对应的**Class**对象、**异常类**对象（Exception）。

# ==JVM 调优==

1. -Xms:java 堆内存的大小；-Xmx：java 堆内存的最大大小
2. -Xmn：新生代大小；-Xmn515M

### 命令行工具

**top** 查看进程的资源占用。（查看哪个进程占用 CPU 过高）

```
top
top -p 进程号 -H
Printf “%x” 占用最高的进程号 12791
得到16进制
jstack 12863 |grep -A 100 3242
```

**jps：虚拟机进程状况工具**。可以列出正在运行的虚拟机进程，并显示其执行的主类的名称和进程 ID。
**jstat：虚拟机统计信息监视工具。** 可以显示虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。
**jinfo：java 配置信息工具**。实时查看和调整虚拟机各项参数。
**jmap：java 内存映像工具。** 生成堆转储快照，java 堆和方法区的详细信息，如空间使用率、当前使用的哪种收集器。
**jstack**：**java 堆栈跟踪工具。** 生成线程快照。查看线程的状态。

### JVM 调优命令

- jps：JVM Process Status Tool,显示指定系统内所有的 HotSpot 虚拟机进程。
- jstat：jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT 编译等运行数据。
- jmap：jmap(JVM Memory Map)命令用于生成 heap dump 文件，如果不使用这个命令，还阔以使用-XX:+HeapDumpOnOutOfMemoryError 参数来让虚拟机出现 OOM 的时候·自动生成 dump 文件。
  jmap 不仅能生成 dump 文件，还阔以查询 finalize 执行队列、Java 堆和永久代的详细信息，如当前使用率、当前使用的是哪种收集器等。
- jhat：jhat(JVM Heap Analysis Tool)命令是与 jmap 搭配使用，用来分析 jmap 生成的 dump，jhat 内置了一个微型的 HTTP/HTML 服务器，生成 dump 的分析结果后，可以在浏览器中查看。在此要注意，一般不会直接在服务器上进行分析，因为 jhat 是一个耗时并且耗费硬件资源的过程，一般把服务器生成的 dump 文件复制到本地或其他机器上进行分析。
- jstack：jstack 用于生成 java 虚拟机当前时刻的线程快照。jstack 来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果 java 程序崩溃生成 core 文件，jstack 工具可以用来获得 core 文件的 java stack 和 native stack 的信息，从而可以轻松地知道 java 程序是如何崩溃和在程序何处发生问题。

### 可视化工具

**JConsole：java 监视与管理控制台。** 内存监控、线程监控。

# ==线上故障排查==

## 硬件故障排查

如果一个实例发生了问题，根据情况选择，要不要着急去重启。如果出现的 CPU、内存飙高或者日志里出现了 OOM 异常

第一步是**隔离**，第二步是**保留现场**，第三步才是**问题排查**。

**隔离**
就是把你的这台机器从请求列表里摘除，比如把 nginx 相关的权重设成零。

**现场保留**

**瞬时态和历史态**
![](图集/Pasted%20image%2020230822211929.png)

查看比如 CPU、系统内存等，通过历史状态可以体现一个趋势性问题，而这些信息的获取一般依靠监控系统的协作。

**保留信息**

（1）**系统当前网络连接**

```
ss -antp > $DUMP_DIR/ss.dump 2>&1
```

使用 ss 命令而不是 netstat 的原因，是因为 netstat 在网络连接非常多的情况下，执行非常缓慢。

后续的处理，可通过查看各种网络连接状态的梳理，来排查 TIME_WAIT 或者 CLOSE_WAIT，或者其他连接过高的问题，非常有用。

（2）**网络状态统计**

```java
netstat -s > $DUMP_DIR/netstat-s.dump 2>&1
```

它能够按照各个协议进行统计输出，对把握当时整个网络状态，有非常大的作用。

```java
sar -n DEV 1 2 > $DUMP_DIR/sar-traffic.dump 2>&1
```

在一些速度非常高的模块上，比如 Redis、Kafka，就经常发生跑满网卡的情况。表现形式就是网络通信非常缓慢。

（3）**进程资源**

```java
lsof -p $PID > $DUMP_DIR/lsof-$PID.dump
```

通过查看进程，能看到打开了哪些文件，可以以进程的维度来查看整个资源的使用情况，包括每条网络连接、每个打开的文件句柄。同时，也可以很容易的看到连接到了哪些服务器、使用了哪些资源。这个命令在资源非常多的情况下，输出稍慢，请耐心等待。

（4）**CPU 资源**

```
mpstat > $DUMP_DIR/mpstat.dump 2>&1
vmstat 1 3 > $DUMP_DIR/vmstat.dump 2>&1
sar -p ALL  > $DUMP_DIR/sar-cpu.dump  2>&1
uptime > $DUMP_DIR/uptime.dump 2>&1
```

主要用于输出当前系统的 CPU 和负载，便于事后排查。

（5）**I/O 资源**

```java
iostat -x > $DUMP_DIR/iostat.dump 2>&1
```

一般，以计算为主的服务节点，I/O 资源会比较正常，但有时也会发生问题，比如**日志输出过多，或者磁盘问题**等。此命令可以输出每块磁盘的基本性能信息，用来排查 I/O 问题。在第 8 课时介绍的 GC 日志分磁盘问题，就可以使用这个命令去发现。

（6）**内存问题**

```java
free -h > $DUMP_DIR/free.dump 2>&1
```

free 命令能够大体展现操作系统的内存概况，这是故障排查中一个非常重要的点，比如 SWAP 影响了 GC，SLAB 区挤占了 JVM 的内存。

（7）**其他全局**

```java
ps -ef > $DUMP_DIR/ps.dump 2>&1
dmesg > $DUMP_DIR/dmesg.dump 2>&1
sysctl -a > $DUMP_DIR/sysctl.dump 2>&1
```

dmesg 是许多静悄悄死掉的服务留下的最后一点线索。当然，ps 作为执行频率最高的一个命令，由于内核的配置参数，会对系统和 JVM 产生影响，所以我们也输出了一份。

（8）**进程快照**，最后的遗言（jinfo）

```java
${JDK_BIN}jinfo $PID > $DUMP_DIR/jinfo.dump 2>&1
```

此命令将输出 Java 的基本进程信息，包括**环境变量和参数配置**，可以查看是否因为一些错误的配置造成了 JVM 问题。

**（9）dump 堆信息**

```java
${JDK_BIN}jstat -gcutil $PID > $DUMP_DIR/jstat-gcutil.dump 2>&1
${JDK_BIN}jstat -gccapacity $PID > $DUMP_DIR/jstat-gccapacity.dump 2>&1
```

jstat 将输出当前的 gc 信息。一般，基本能大体看出一个端倪，如果不能，可将借助 jmap 来进行分析。

**（10）堆信息**

```java
${JDK_BIN}jmap $PID > $DUMP_DIR/jmap.dump 2>&1
${JDK_BIN}jmap -heap $PID > $DUMP_DIR/jmap-heap.dump 2>&1
${JDK_BIN}jmap -histo $PID > $DUMP_DIR/jmap-histo.dump 2>&1
${JDK_BIN}jmap -dump:format=b,file=$DUMP_DIR/heap.bin $PID > /dev/null  2>&1
```

jmap 将会得到当前 Java 进程的 dump 信息。如上所示，其实最有用的就是第 4 个命令，但是前面三个能够让你初步对系统概况进行大体判断。因为，第 4 个命令产生的文件，一般都非常的大。而且，需要下载下来，导入 MAT 这样的工具进行深入分析，才能获取结果。这是分析内存泄漏一个必经的过程。

**（11）JVM 执行栈**

```java
${JDK_BIN}jstack $PID > $DUMP_DIR/jstack.dump 2>&1
```

jstack 将会获取当时的执行栈。一般会多次取值，我们这里取一次即可。这些信息非常有用，能够还原 Java 进程中的线程情况。

```java
top -Hp $PID -b -n 1 -c >  $DUMP_DIR/top-$PID.dump 2>&1
```

为了能够得到更加精细的信息，我们使用 top 命令，来获取进程中所有线程的 CPU 信息，这样，就可以看到资源到底耗费在什么地方了。

**（12）高级替补**

```java
kill -3 $PID
```

有时候，jstack 并不能够运行，有很多原因，比如 Java 进程几乎不响应了等之类的情况。我们会尝试向进程发送 kill -3 信号，这个信号将会打印 jstack 的 trace 信息到日志文件中，是 jstack 的一个替补方案。

```java
gcore -o $DUMP_DIR/core $PID
```

对于 jmap 无法执行的问题，也有替补，那就是 GDB 组件中的 gcore，将会生成一个 core 文件。我们可以使用如下的命令去生成 dump：

```java
${JDK_BIN}jhsdb jmap --exe ${JDK}java  --core $DUMP_DIR/core --binaryheap
```

3. **内存泄漏的现象**

稍微提一下 jmap 命令，它在 9 版本里被干掉了，取而代之的是 jhsdb，你可以像下面的命令一样使用。

```java
jhsdb jmap  --heap --pid  37340
jhsdb jmap  --pid  37288
jhsdb jmap  --histo --pid  37340
jhsdb jmap  --binaryheap --pid  37340
```

一般内存溢出，表现形式就是 Old 区的占用持续上升，即使经过了多轮 GC 也没有明显改善。比如 ThreadLocal 里面的 GC Roots，内存泄漏的根本就是，这些对象并没有切断和 GC Roots 的关系，可通过一些工具，能够看到它们的联系。

## 报表异常 | JVM 调优

有一个报表系统，频繁发生内存溢出，在高峰期间使用时，还会频繁的发生拒绝服务，由于大多数使用者是管理员角色，所以很快就反馈到研发这里。

业务场景是由于有些结果集的字段不是太全，因此需要对结果集合进行循环，并通过 HttpClient 调用其他服务的接口进行数据填充。使用 Guava 做了 JVM 内缓存，但是响应时间依然很长。

初步排查，JVM 的资源太少。接口 A 每次进行报表计算时，都要涉及几百兆的内存，而且在内存里驻留很长时间，有些计算又非常耗 CPU，特别的“吃”资源。而我们分配给 JVM 的内存只有 3 GB，在多人访问这些接口的时候，内存就不够用了，进而发生了 OOM。在这种情况下，没办法，只有升级机器。把机器配置升级到 4C8G，给 JVM 分配 6GB 的内存，这样 OOM 问题就消失了。但随之而来的是频繁的 GC 问题和超长的 GC 时间，平均 GC 时间竟然有 5 秒多。

进一步，由于报表系统和高并发系统不太一样，它的对象，存活时长大得多，并不能仅仅通过增加年轻代来解决；而且，如果增加了年轻代，那么必然减少了老年代的大小，由于 CMS 的碎片和浮动垃圾问题，我们可用的空间就更少了。虽然服务能够满足目前的需求，但还有一些不太确定的风险。

第一，了解到程序中有很多缓存数据和静态统计数据，为了减少 MinorGC 的次数，通过分析 GC 日志打印的对象年龄分布，把 MaxTenuringThreshold 参数调整到了 3（特殊场景特殊的配置）。这个参数是让年轻代的这些对象，赶紧回到老年代去，不要老呆在年轻代里。

第二，我们的 GC 时间比较长，就一块开了参数 CMSScavengeBeforeRemark，使得在 CMS remark 前，先执行一次 Minor GC 将新生代清掉。同时配合上个参数，其效果还是比较好的，一方面，对象很快晋升到了老年代，另一方面，年轻代的对象在这种情况下是有限的，在整个 MajorGC 中占的时间也有限。

第三，由于缓存的使用，有大量的弱引用，拿一次长达 10 秒的 GC 来说。我们发现在 GC 日志里，处理 weak refs 的时间较长，达到了 4.5 秒。这里可以加入参数 ParallelRefProcEnabled 来并行处理 Reference，以加快处理速度，缩短耗时。

优化之后，效果不错，但并不是特别明显。经过评估，针对高峰时期的情况进行调研，我们决定再次提升机器性能，改用 8core16g 的机器。但是，这带来另外一个问题。

**高性能的机器带来了非常大的服务吞吐量**，通过 jstat 进行监控，能够看到年轻代的分配速率明显提高，但随之而来的 MinorGC 时长却变的不可控，有时候会超过 1 秒。累积的请求造成了更加严重的后果。

这是由于堆空间明显加大造成的回收时间加长。为了获取较小的停顿时间，我们在堆上**改用了 G1 垃圾回收器**，把它的目标设定在 200ms。G1 是一款非常优秀的垃圾收集器，不仅适合堆内存大的应用，同时也简化了调优的工作。通过主要的参数初始和最大堆空间、以及最大容忍的 GC 暂停目标，就能得到不错的性能。修改之后，虽然 GC 更加频繁了一些，但是停顿时间都比较小，应用的运行较为平滑。

到目前为止，也只是勉强顶住了已有的业务，但是，这时候领导层面又发力，**要求报表系统可以支持未来两年业务 10 到 100 倍的增长**，并保持其可用性，但是这个“千疮百孔”的报表系统，稍微一压测，就宕机，那如何应对十倍百倍的压力呢 ? 硬件即使可以做到动态扩容，但是毕竟也有极限。

使用 **MAT 分析堆快照**，发现很多地方可以通过代码优化，那些占用内存特别多的对象：

1、select \* 全量排查，只允许获取必须的数据

2、报表系统中**cache 实际的命中率并不高**，将**Guava 的 Cache 引用级别**改成**弱引用**（WeakKeys）

3、**限制报表导入文件大小**，同时**拆分用户超大范围查询导出请求**。

每一步操作都使得 JVM 使用变得更加可用，一系列优化以后，机器相同压测数据性能提升了数倍。

## 服务雪崩/大屏异常 | JUC 调优

有些数据需要使用 HttpClient 来获取进行补全。提供数据的服务提供商有的响应时间可能会很长，也有可能会造成服务整体的阻塞。

![](图集/Pasted%20image%2020230822212052.png)

接口 A 通过 HttpClient 访问服务 2，响应 100ms 后返回；接口 B 访问服务 3，耗时 2 秒。HttpClient 本身是有一个最大连接数限制的，如果服务 3 迟迟不返回，就会造成 HttpClient 的连接数达到上限，**概括来讲，就是同一服务，由于一个耗时非常长的接口，进而引起了整体的服务不可用**

这个时候，通过 jstack 打印栈信息，会发现大多数竟然阻塞在了接口 A 上，而不是耗时更长的接口 B，这个现象起初十分具有迷惑性，不过经过分析后，我们猜想其实是因为接口 A 的速度比较快，在问题发生点进入了更多的请求，它们全部都阻塞住的同时被打印出来了。

为了验证这个问题，我搭建了一个 demo 工程，模拟了两个使用同一个 HttpClient 的接口。fast 接口用来访问百度，很快就能返回；slow 接口访问谷歌，由于众所周知的原因，会阻塞直到超时，大约 10 s。 利用 ab 对两个接口进行压测，同时使用 jstack 工具 dump 堆栈。首先使用 jps 命令找到进程号，然后把结果重定向到文件（可以参考 10271.jstack 文件）。

过滤一下 nio 关键字，可以查看 tomcat 相关的线程，足足有 200 个，这和 Spring Boot 默认的 maxThreads 个数不谋而合。更要命的是，有大多数线程，都处于 BLOCKED 状态，说明线程等待资源超时。通过 grep fast | wc -l 分析，确实 200 个中有 150 个都是 blocked 的 fast 的进程。

问题找到了，解决方式就顺利成章了。

1、fast 和 slow 争抢连接资源，通过**线程池限流**或者**熔断处理**

2、有时候 slow 的线程也不是一直 slow，所以就得加入监控

3、使用带**countdownLaunch**对线程的**执行顺序逻辑**进行控制

## 接口延迟 | SWAP 调优

**开启了 SWAP 分区--->GC 和硬盘交互-->导致 GC 时间很长-->导致服务卡顿-->关闭 SWAP 分区**

有一个关于服务的某个实例，**经常发生服务卡顿**。由于服务的并发量是比较高的，每多停顿 1 秒钟，几万用户的请求就会感到延迟。

我们统计、类比了此服务其他实例的 CPU、内存、网络、I/O 资源，区别并不是很大，所以一度怀疑是机器硬件的问题。

接下来我们对比了节点的 GC 日志，发现无论是 Minor GC，还是 Major GC，这个节点所花费的时间，都比其他实例长得多。

通过仔细观察，我们发现在 GC 发生的时候，vmstat 的 si、so 飙升的非常严重，这和其他实例有着明显的不同。

使用 free 命令再次确认，发现 SWAP 分区，使用的比例非常高，引起的具体原因是什么呢？

更详细的操作系统内存分布，从 /proc/meminfo 文件中可以看到具体的逻辑内存块大小，有多达 40 项的内存信息，这些信息都可以通过遍历 /proc 目录的一些文件获取。我们注意到 slabtop 命令显示的有一些异常，dentry（目录高速缓冲）占用非常高。

问题最终定位到是由于某个运维工程师删除日志时，**定时执行了一句命令**：

**find / | grep "xxx.log"**

他是想找一个叫做 要被删除 的日志文件，看看在哪台服务器上，结果，这些老服务器由于文件太多，扫描后这些文件信息都缓存到了 slab 区上。而**服务器开了 swap**，操作系统发现物理内存占满后，并没有立即释放 cache，导致**每次 GC 都要和硬盘打一次交道**。

**解决方式就是关闭 SWAP 分区。**

**swap** 是很多性能场景的**万恶之源**，建议禁用。在高并发 SWAP 绝对能让你体验到它魔鬼性的一面：**进程倒是死不了了，但 GC 时间长**的却让人无法忍受。

## 内存溢出 | Cache 调优

> 有一次线上遇到故障，重新启动后，使用 jstat 命令，发现 Old 区一直在增长。我使用 jmap 命令，导出了一份线上堆栈，然后使用 MAT 进行分析，通过对 GC Roots 的分析，发现了一个非常大的 HashMap 对象，这个原本是其他同事做缓存用的，但是做了一个无界缓存，没有设置超时时间或者 LRU 策略，在使用上又没有重写 key 类对象的 hashcode 和 equals 方法，对象无法取出也直接造成了堆内存占用一直上升，后来，将这个缓存**改成 guava 的 Cache，并设置了弱引用**，故障就消失了。
>
> 关于文件处理器的应用，在读取或者写入一些文件之后，由于发生了一些异常，**close 方法又没有放在 finally** 块里面，造成了文件句柄的泄漏。由于文件处理十分频繁，产生了严重的内存泄漏问题。

**内存溢出**是一个结果，而**内存泄漏**是一个原因。内存溢出的原因有**内存空间不足、配置错误**等因素。

**内存泄漏**：一些**错误的编程**方式，**不再被使用的对象、没有被回收、没有及时切断与 GC Roots 的联系**。

例子 1：使用了 **HashMap 做缓存**，但是并**没有设置超时时间**或者 **LRU 策略**，造成了放入 Map 对象的数据越来越多，而产生了**内存泄漏**。如果**没有重写 Key 类**的 **hashCode** 和 **equals 方法**，造成了放入 HashMap 的**所有对象都无法被取出来**，与外界失联了，也会产生**内存泄漏**。所以下面的代码结果是 null。

```java
//leak example
import java.util.HashMap;
import java.util.Map;
public class HashMapLeakDemo {
    public static class Key {
        String title;
    public Key(String title) {
        this.title = title;
    }
}
public static void main(String[] args) {
    Map<Key, Integer> map = new HashMap<>();
    map.put(new Key("1"), 1);
    map.put(new Key("2"), 2);
    map.put(new Key("3"), 2);
    Integer integer = map.get(new Key("2"));
    System.out.println(integer);
    }
}
```

**即使提供了 equals 方法和 hashCode 方法，也要非常小心，尽量避免使用自定义的对象作为 Key。**

例子 2：**关于文件处理器的应用**，在读取或者写入一些文件之后，由于发生了一些异常，**close 方法又没有放在 finally** 块里面，造成了**文件句柄的泄漏**。由于**文件处理十分频繁**，产生了严重的**内存泄漏**问题。

## CPU 飙高 | 死循环

我们有个线上应用，单节点在运行一段时间后，CPU 的使用会飙升，一旦飙升，一般怀疑某个业务逻辑的计算量太大，或者是触发了死循环（比如著名的 HashMap 高并发引起的死循环），但排查到最后其实是 GC 的问题。

（1）**使用 top 命令**，查找到**使用 CPU 最多的某个进程**，记录它的 pid。使用 **Shift + P 快捷键**可以按 CPU **使用率进行排序**。

```java
top
```

（2）再次使用 **top -H**命令，查看**某个进程中使用 CPU 最多的某个线程**，记录线程 ID。

```java
top -Hp $pid
```

（3）使用 **printf** 函数，将**十进制的 线程 ID 转化成十六进制**。

```java
printf %x $tid
```

（4）使用 **jstack** 命令，**查看 Java 进程的线程栈**。

```java
jstack $pid >$pid.log
```

（5）使用 **less 命令查看生成的文件**，并**查找刚才转化的十六进制 tid**，找到**发生问题的线程上下文**。

```java
less $pid.log
```

我们**在 jstack 日志搜关键字 DEAD**，找到了 CPU 使用最多的几个线程 id。

可以看到问题发生的根源，是我们的**堆已经满了**，但是**又没有发生 OOM**，于是 **GC 进程就一直在那里回收**，**回收效果又非常一般**，造成 **CPU 升高应用假死**。接下来的具体问题排查，就需要**把内存 dump 一份下来**，使用 **MAT 等工具分析具体原因**了。
