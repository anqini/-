# 面试面经总结
全栈 / IOS


## 进程和线程 进程通信

进程和线程的主要差别在于它们是不同的操作系统资源管理方式。进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，而线程只是一个进程中的不同执行路径。线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉，所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。但对于一些要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程。

1. 简而言之,一个程序至少有一个进程,一个进程至少有一个线程.
2. 线程的划分尺度小于进程，使得多线程程序的并发性高。
3. 另外，进程在执行过程中拥有独立的内存单元，而多个线程共享内存，从而极大地提高了程序的运行效率。
4. 线程在执行过程中与进程还是有区别的。每个独立的线程有一个程序运行的入口、顺序执行序列和程序的出口。但是线程不能够独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制。
5. 从逻辑角度来看，多线程的意义在于一个应用程序中，有多个执行部分可以同时执行。但操作系统并没有将多个线程看做多个独立的应用，来实现进程的调度和管理以及资源分配。这就是进程和线程的重要区别。

IPC目的

1）数据传输：一个进程需要将它的数据发送给另一个进程，发送的数据量在一个字节到几兆字节之间。
2）共享数据：多个进程想要操作共享数据，一个进程对共享数据的修改，别的进程应该立刻看到。
3）通知事件：一个进程需要向另一个或一组进程发送消息，通知它（它们）发生了某种事件（如进程终止时要通知父进程）。
4）资源共享：多个进程之间共享同样的资源。为了作到这一点，需要内核提供锁和同步机制。
5）进程控制：有些进程希望完全控制另一个进程的执行（如Debug进程），此时控制进程希望能够拦截另一个进程的所有陷入和异常，并能够及时知道它的状态改变。

IPC方式包括：管道、系统IPC（信号量、消息队列、共享内存）和套接字（socket）

### 管道：
3种。管道是面向字节流，自带互斥与同步机制，生命周期随进程。 
1. 普通管道PIPE, 通常有两种限制,一是半双工,数据同时只能单向传输;二是只能在父子或者兄弟进程间使用.，
2. 命令流管道s_pipe: 去除了第一种限制,为全双工，可以同时双向传输，
3. 命名管道FIFO, 去除了第二种限制,可以在许多并不相关的进程之间进行通讯。
4. 无名管道：没有磁盘节点，仅仅作为一个内存对象，用完就销毁了。因此没有显示的打开过程，实际在创建时自动打开，并且生成内存iNode，其内存对象和普通文件的一致，所以读写操作用的同样的接口，但是专用的。因为不能显式打开（没有任何标示），所以只能用在父子进程，兄弟进程， 或者其他继承了祖先进程的管道文件对象的两个进程间使用【具有共同祖先的进程】


### 信号量：

1. 临界资源：同一时刻，只能被一个进程访问的资源
2. 临界区：访问临界资源的代码区
3. 原子操作：任何情况下不能被打断的操作。

它是一个计数器，记录资源能被多少个进程同时访问。用于控制多进程对临界资源的访问（同步)），并且是非负值。主要作为进程间以及同一进程的不同线程间的同步手段。
操作：创建或获取，若是创建必须初始化，否则不用初始化。
```
int semget((key_t)key, int nsems, int flag);//创建或获取信号量

int semop(int semid, stuct sembuf*buf, int length);//加一操作（V操作）：释放资源；减一操作（P操作）：获取资源

int semct(int semid, int pos, int cmd);//初始化和删除
```

注：我们可以封装成库，实现信号量的创建或初始化，p操作，V操作，删除操作。。

### 消息队列：

消息队列是消息的链表，是存放在内核中并由消息队列标识符标识。因此是随内核持续的，只有在内核重起或者显示删除一个消息队列时，该消息队列才会真正被删除。。消息队列克服了信号传递信息少，管道只能承载无格式字节流以及缓冲区受限等特点。允许不同进程将格式化的数据流以消息队列形式发送给任意进程，对消息队列具有操作权限的进程都可以使用msgget完成对消息队列的操作控制，通过使用消息类型，进程可以按顺序读信息，或为消息安排优先级顺序。

与信号量相比，都以内核对象确保多进程访问同一消息队列。但消息队列发送实际数据，信号量进行进程同步控制。

与管道相比，管道发送的数据没有类型，读取数据端无差别从管道中按照前后顺序读取；消息队列有类型，读端可以根据数据类型读取特定的数据。

操作：创建或获取消息队列， int msgget((key_tkey, int flag);//若存在获取，否则创建它

发送消息：int msgsnd(int msgid, void *ptr, size_t size, int flag); ptr指向一个结构体存放类型和数据 size 数据的大小

接受消息：int msgrcv(int msgid, void *ptr, size_t size, long type, int flag);

删除消息队列： int msgctl(int msgid, int cmd, struct msgid_ds*buff);
       
### 共享内存：

共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。    
共享内存是最快的一种IPC，因为不需要在客户进程和服务器进程之间赋值。使用共享内存的唯一注意的是是多个进程对一给定的存储区的同步访问。【若服务器进程正在向共享存储区写入数据，则写完数据之前客户进程不应读取数据，或者客户进程正在从共享内存中读取数据，服务进程不应写入数据。。所以我们要对共享内存进行同步控制，通常是信号量。】
```
int shmget((key_t)key, size_t size, int flag); //size 开辟内存空间的大小，flag：若存在则获取，否则创建共享内存存储段，返回一个标识符。

void *shmat(int shmid, void *addr, int flag); //将共享内存段连接到进程的地址空间中，返回一个共享内存首地址
```
函数shmat将标识号为shmid共享内存映射到调用进程的地址空间中，映射的地址由参数shmaddr和shmflg共同确定，其准则为：

1. 如果参数shmaddr取值为NULL，系统将自动确定共享内存链接到进程空间的首地址。

2. 如果参数shmaddr取值不为NULL且参数shmflg没有指定SHM_RND标志，系统将运用地址shmaddr链接共享内存。

3. 如果参数shmaddr取值不为NULL且参数shmflg指定了SHM_RND标志位，系统将地址shmaddr对齐后链接共享内存。其中选项SHM_RND的意思是取整对齐，常数SHMLBA代表了低边界地址的倍数，公式“shmaddr – (shmaddr % SHMLBA)”的意思是将地址shmaddr移动到低边界地址的整数倍上。
```
int shmdt(void *ptr); //断开进程与共享内存的链接
```
进程脱离共享内存区后，数据结构 shmid_ds 中的 shm_nattch 就会减 1 。但是共享段内存依然存在，只有 shm_attch 为 0 后，即没有任何进程再使用该共享内存区，共享内存区才在内核中被删除。一般来说，当一个进程终止时，它所附加的共享内存区都会自动脱离。
```
int shmctl(int shmid, int cmd, struct shmid_ds *buff); //删除共享内存（内核对象）
```
shmid是shmget返回的标识符；

cmd是执行的操作：有三种值，一般为 IPC_RMID  删除共享内存段；

buff默认为0.

如果共享内存已经与所有访问它的进程断开了连接，则调用IPC_RMID子命令后，系统将立即删除共享内存的标识符，并删除该共享内存区，以及所有相关的数据结构；

如果仍有别的进程与该共享内存保持连接，则调用IPC_RMID子命令后，该共享内存并不会被立即从系统中删除，而是被设置为IPC_PRIVATE状态，并被标记为”已被删除”（使用ipcs命令可以看到dest字段）；直到已有连接全部断开，该共享内存才会最终从系统中消失。

需要说明的是：一旦通过shmctl对共享内存进行了删除操作，则该共享内存将不能再接受任何新的连接，即使它依然存在于系统中！所以，可以确知， 在对共享内存删除之后不可能再有新的连接，则执行删除操作是安全的；否则，在删除操作之后如仍有新的连接发生，则这些连接都将可能失败！

### socket通信

适合同一主机的不同进程间和不同主机的进程间进行全双工网络通信。但并不只是Linux有，在所有提供了TCP/IP协议栈的操作系统中几乎都提供了socket，而所有这样操作系统，对套接字的编程方法几乎是完全一样的,即“网络编程”。

## 死锁产生 死锁避免

### 死锁概念及产生原理
概念：多个并发进程因争夺系统资源而产生相互等待的现象。
原理：当一组进程中的每个进程都在等待某个事件发生，而只有这组进程中的其他进程才能触发该事件，这就称这组进程发生了死锁。
本质原因：
- 系统资源有限。
- 进程推进顺序不合理。

### 死锁产生的4个必要条件
1. 互斥：某种资源一次只允许一个进程访问，即该资源一旦分配给某个进程，其他进程就不能再访问，直到该进程访问结束。
2. 占有且等待：一个进程本身占有资源（一种或多种），同时还有资源未得到满足，正在等待其他进程释放该资源。
3. 不可抢占：别人已经占有了某项资源，你不能因为自己也需要该资源，就去把别人的资源抢过来。
4. 循环等待：存在一个进程链，使得每个进程都占有下一个进程所需的至少一种资源。
       
当以上四个条件均满足，必然会造成死锁，发生死锁的进程无法进行下去，它们所持有的资源也无法释放。这样会导致CPU的吞吐量下降。所以死锁情况是会浪费系统资源和影响计算机的使用性能的。那么，解决死锁问题就是相当有必要的了。

### 避免死锁的方法

死锁预防 ----- 确保系统永远不会进入死锁状态
     
     
产生死锁需要四个条件，那么，只要这四个条件中至少有一个条件得不到满足，就不可能发生死锁了。由于互斥条件是非共享资源所必须的，不仅不能改变，还应加以保证，所以，主要是破坏产生死锁的其他三个条件。
a. 破坏“占有且等待”条件

方法1：所有的进程在开始运行之前，必须一次性地申请其在整个运行过程中所需要的全部资源。
  优点：简单易实施且安全。
  缺点：因为某项资源不满足，进程无法启动，而其他已经满足了的资源也不会得到利用，严重降低了资源的利用率，造成资源浪费。
           使进程经常发生饥饿现象。

方法2：该方法是对第一种方法的改进，允许进程只获得运行初期需要的资源，便开始运行，在运行过程中逐步释放掉分配到的已经使用完毕的资源，然后再去请求新的资源。这样的话，资源的利用率会得到提高，也会减少进程的饥饿问题。
     
b. 破坏“不可抢占”条件
当一个已经持有了一些资源的进程在提出新的资源请求没有得到满足时，它必须释放已经保持的所有资源，待以后需要使用的时候再重新申请。这就意味着进程已占有的资源会被短暂地释放或者说是被抢占了。
该种方法实现起来比较复杂，且代价也比较大。释放已经保持的资源很有可能会导致进程之前的工作实效等，反复的申请和释放资源会导致进程的执行被无限的推迟，这不仅会延长进程的周转周期，还会影响系统的吞吐量。
      
c. 破坏“循环等待”条件
     可以通过定义资源类型的线性顺序来预防，可将每个资源编号，当一个进程占有编号为i的资源时，那么它下一次申请资源只能申请编号大于i的资源。如图所示：
这样虽然避免了循环等待，但是这种方法是比较低效的，资源的执行速度回变慢，并且可能在没有必要的情况下拒绝资源的访问，比如说，进程c想要申请资源1，如果资源1并没有被其他进程占有，此时将它分配个进程c是没有问题的，但是为了避免产生循环等待，该申请会被拒绝，这样就降低了资源的利用率

避免死锁 ----- 在使用前进行判断，只允许不会产生死锁的进程申请资源的死锁避免是利用额外的检验信息，在分配资源时判断是否会出现死锁，只在不会出现死锁的情况下才分配资源。

两种避免办法：
1. 如果一个进程的请求会导致死锁，则不启动该进程
2. 如果一个进程的增加资源请求会导致死锁 ，则拒绝该申请。

避免死锁的具体实现通常利用银行家算法

银行家算法

银行家算法的相关数据结构
- 可利用资源向量Available：用于表示系统里边各种资源剩余的数目。由于系统里边拥有的资源通常都是有很多种（假设有m种），所以，我们用一个有m个元素的数组来表示各种资源。数组元素的初始值为系统里边所配置的该类全部可用资源的数目，其数值随着该类资源的分配与回收动态地改变。
- 最大需求矩阵Max：用于表示各个进程对各种资源的额最大需求量。进程可能会有很多个（假设为n个），那么，我们就可以用一个nxm的矩阵来表示各个进程多各种资源的最大需求量
- 分配矩阵Allocation：顾名思义，就是用于表示已经分配给各个进程的各种资源的数目。也是一个nxm的矩阵。
- 需求矩阵Need：用于表示进程仍然需要的资源数目，用一个nxm的矩阵表示。系统可能没法一下就满足了某个进程的最大需求（通常进程对资源的最大需求也是只它在整个运行周期中需要的资源数目，并不是每一个时刻都需要这么多），于是，为了进程的执行能够向前推进，通常，系统会先分配个进程一部分资源保证进程能够执行起来。那么，进程的最大需求减去已经分配给进程的数目，就得到了进程仍然需要的资源数目了。

银行家算法通过对进程需求、占有和系统拥有资源的实时统计，确保系统在分配给进程资源不会造成死锁才会给与分配。
死锁避免的优点：不需要死锁预防中的抢占和重新运行进程，并且比死锁预防的限制要少。
死锁避免的限制：
- 必须事先声明每个进程请求的最大资源量
- 考虑的进程必须无关的，也就是说，它们执行的顺序必须没有任何同步要求的限制
- 分配的资源数目必须是固定的。
- 在占有资源时，进程不能退出

## 网络七层协议

应用层，表示层，会话层，传输层，网络层，链路层，物理层

## tcp udp属于哪层 区别
传输层
UDP：

无连接：意思就是在通讯之前不需要建立连接，直接传输数据。

不可靠：是将数据报的分组从一台主机发送到另一台主机，但并不保证数据报能够到达另一端，任何必须的可靠性都由应用程序提供。

TCP协议：

TCP协议是面向连接的、可靠传输、有流量控制，拥塞控制，面向字节流传输等很多优点的协议。

## 两次握手行不行

因为TCP传输是双向的，每个方向的通道的建立都需要一次SYN+ACK，所以建立两个方向的通道最多需要两次(SYN+ACK)，即四次握手。
另外，建立一个方向的通道，最关键就是确定初始的seq号，因此，在SYN中确定seq=n，并收到ACK=n+1时，才算确定好seq号。
因此，TCP的建立最多需要四次握手，但是由于第二次和第三次的握手SYN和ACK可以合并，因此最少可以三次握手。
另外，若只进行两次握手，那么显然只能建立一个单方向的通道，即数据只能从客户端发送到服务器。


## tcp怎样保证可靠交付

TCP通过下列方式来提供可靠性：
1. 将数据截断为合理的长度。
应用数据被分割成TCP认为最适合发送的数据块。这和UDP完全不同，应用程序产生的数据报长度将保持不变。

2. 超时重发
当TCP发出一个段后，它启动一个定时器，等待目的端确认收到这个报文段。如果不能及时收到一个确认，将重发这个报文段。

3. 对于收到的请求，给出确认响应
当TCP收到发自TCP连接另一端的数据，它将发送一个确认。这个确认不是立即发送，通常将推迟几分之一秒。(之所以推迟，可能是要对包做完整校验)

4. 校验出包有错，丢弃报文段，不给出响应，TCP发送数据端，超时时会重发数据
TCP将保持它首部和数据的检验和。这是一个端到端的检验和，目的是检测数据在传输过程中的任何变化。 如果收到段的检验和有差错，TCP将丢弃这个报文段和不确认收到此报文段。 （希望发端超时并重发）

5. 对失序数据进行重新排序，然后才交给应用层
既然TCP报文段作为IP数据报来传输，而IP数据报的到达可能会失序，因此TCP报文段的到达也可能会失序。 如果必要，TCP将对收到的数据进行重新排序，将收到的数据以正确的顺序交给应用层。

6. 对于重复数据，能够丢弃重复数据
既然IP数据报会发生重复，TCP的接收端必须丢弃重复的数据。

7. TCP还能提供流量控制。
TCP连接的每一方都有固定大小的缓冲空间。
TCP的接收端只允许另一端发送接收端缓冲区所能接纳的数据。这将防止较快主机致使较慢主机的缓冲区溢出。TCP使用的流量控制协议是可变大小的滑动窗口协议。

TCP提供一种面向连接的、可靠的字节流服务。

面向连接：意味着两个使用TCP的应用（通常是一个客户和一个服务器）在彼此交换数据之前必须先建立一个TCP连接。在一个TCP连接中，仅有两方进行彼此通信。广播和多播不能用于TCP。

字节流服务：两个应用程序通过TCP连接交换8bit字节构成的字节流。TCP不在字节流中插入记录标识符。我们将这称为字节流服务（bytestreamservice）。 TCP对字节流的内容不作任何解释:TCP对字节流的内容不作任何解释。TCP不知道传输的数据字节流是二进制数据，还是ASCII字符、EBCDIC字符或者其他类型数据。对字节流的解释由TCP连接双方的应用层解释。


## 输入url后的事件流程

1、首先，在浏览器地址栏中输入url

2、浏览器先查看浏览器缓存-系统缓存-路由器缓存，如果缓存中有，会直接在屏幕中显示页面内容。若没有，则跳到第三步操作。

3、在发送http请求前，需要域名解析(DNS解析)，解析获取相应的IP地址。

4、浏览器向服务器发起tcp连接，与浏览器建立tcp三次握手。

5、握手成功后，浏览器向服务器发送http请求，请求数据包。

6、服务器处理收到的请求，将数据返回至浏览器

7、浏览器收到HTTP响应

8、读取页面内容，浏览器渲染，解析html源码

9、生成Dom树、解析css样式、js交互

10、客户端和服务器交互

11、ajax查询

 

其中，步骤2的具体过程是：

    浏览器缓存：浏览器会记录DNS一段时间，因此，只是第一个地方解析DNS请求；
    操作系统缓存：如果在浏览器缓存中不包含这个记录，则会使系统调用操作系统，获取操作系统的记录(保存最近的DNS查询缓存)；
    路由器缓存：如果上述两个步骤均不能成功获取DNS记录，继续搜索路由器缓存；
    ISP缓存：若上述均失败，继续向ISP搜索。


## 路由表查找

1.基本概念
路由的概念：路由是一种指向标，因为网络是一跳一跳往前推进的，因此在每一跳都要有一系列的指向标。实际上不仅仅是分组交换网需要路由，电路交换网在创建虚电路的时候也需要路由，更实际的例子，我们日常生活中，路由无处不在。简单的说，路由由三元素组成：目标地址，掩码，下一跳。注意，路由项中其实没有输出端口-它是链路层概念，Linux操作系统将路由表和转发表混为一谈，而实际上它们应该是分开的(分开的好处之一使得MPLS更容易实现)。
     路由项通过两种途径加入内核，一种是通过用户态路由协议进程或者用户静态配置配置加入，另一种是主机自动发现的路由。所谓自动发现的路由实际上是“发现了一个路由项和一个转发表”，其含义在主机某一个网卡启动的时候生效，比如eth0启动，那么系统生成下列路由表项/转发项：往eth0同一IP网段的包通过eth0发出。
路由表：路由表包含了一系列的表项，包括上述的三元素。
路由框架的层次：路由大致分为两个要素，也可以看成两个层次。第一个层次是路由表项的生成；第二个层次是主机对路由表项的查找。
路由表项生成算法：生成路由表项的方式有两种，第一种是管理员手工配置，第二种为通过路由协议动态生成。
路由查找算法：本文着重于主机层面上对路由表项的查询算法。毕竟这是一个纯技术活儿…相反的，路由协议的实现和配置更讲究人为的策略，如果你人为配置RIP或者OSPF只需要配几条命令就OK了，那么配一个BGP试试，它讲究大量的策略，不是纯技术能解决的。如果有时间，我会单独写一篇文章谈路由协议的，但是今天，只谈路由器/主机对路由表项的查找过程。
     这个过程很重要，如果路由器的查找算法效率提高了，那么很显然，端到端的延迟就降低了，这是一定的。

## 获取mac地址的过程


## Hashmap底层实现

简单说下HashMap的实现原理：

 首先有一个每个元素都是链表（可能表述不准确）的数组，当添加一个元素（key-value）时，就首先计算元素key的hash值，以此确定插入数组中的位置，但是可能存在同一hash值的元素已经被放在数组同一位置了，这时就添加到同一hash值的元素的后面，他们在数组的同一位置，但是形成了链表，同一各链表上的Hash值是相同的，所以说数组存放的是链表。而当链表长度太长时，链表就转换为红黑树，这样大大提高了查找的效率。

 当链表数组的容量超过初始容量的0.75时，再散列将链表数组扩大2倍，把原链表数组的搬移到新的数组中链表加数组的形式存放的 


## 面向对象和面向过程

面向对象相比面向过程的好处：

封装：我们可以根据不同功能和操作的数据来封装成不同对象，由对象实现具体的操作，我们只需要调用对象的方法即可，代码简洁、而且方便测试。

可能你会说面向过程也可以分离出一个一个的函数出来啊，也可以分成各个模块来调用啊，为什么要用面向对象？

接下来看下面的特性：

继承：假如有同一类操作，然后这些操作都有一部分共同的操作，通过面向对象，我们可以让子类对象继承父类对象，父类对象实现了这部分操作，所有子类对象都可以很容易复用这些代码，不用自己实现一遍。

那你可能有会说，面向过程也可以分离出公共的函数来调用吧，为什么要面向对象？

好，接下来看下面向对象的这个特性：

多态：其实前面的继承不单单是复用了父类的代码，还表示所有继承了父类的子类都是同一类对象。

假如我们有这么一个操作，要判断传进来的动物类型，然后执行这个动物的eat操作，面向过程是怎么做的呢？

通过多态，我们只需要声明我需要某一类对象的父类，然后传进来一个具体子类，在运行的时候我们调用的是子类的方法。而且不管后面有加多了多少个子类，都不用改我写好的代码，耦合度大大降低，而且代码易维护。
这下没话说了吧，哈哈

那么面向对象有什么缺点嘛？
面向过程的性能比面向对象高,因为类调用时需要实例化,开销比较大,比较消耗资源，所以单片机、嵌入式开发、Linux/Unix等一般采用面向过程开发，性能是最重要的因素。
 
总结：
- 面向对象：代码易复用、易测试、易扩展、耦合度低、易维护。但性能没面向过程高，因为有对象的实例化，开销较大。
- 面向过程：没有面向对象的易复用、易测试、易扩展、耦合度低、易维护。但性能高。

## 10亿数据查找目标，bitmap

### Bitmap

BitMap也就是位图
引出Bitmap

举一个小例子，有一个无序整形数组{8,4,9}，也就占用内存3*4=12字节，这很正常，但如果有1亿个这样的数呢？1亿*4/(1024*1024*1024)=0.372G左右。如果对该数据做查找，那内存压力很大，我们想要高性能地解决这个问题，就得引出Bitmap。
Bitmap概念

一个byte占8个bit，如果每一个bit的值就是有或没有，也就是二进制的0或1。那就可用bit的0或1代表某数组值有或没有。0代表该数值没有出现过，1代表该数组值出现过。具体如下图：

那1亿的数据现在所需的空间0.372G/32，一个原占32bit的数据现在只占用1bit，节省了不少内存。
应用和代码

疑问：一个数怎么快速定位它的索引，也就是说搞清楚byte[index]的索引号index是多少，位置号position是哪一位。

例子：例如add(14)。14已经超出byte[0]的映射范围，在byte[1]的范围内。那怎么快速定位它的索引号？如果找到它的索引号，又怎么找到它的位置号？Index(N)代表N的索引号，Position(N)代表N的位置号。

### 公式：
Index(N) = N/8 = N >> 3;
Position(N) = N%8 = N & 7;（对于2的幂的数才能这样干！）

### 例子：
add(int num)向Bitmap里add数据该怎么办?
上面已分析了，add的目的是为了将所在的位置从0变成1，其他位置不变。
（图中间，不是“作与操作”，是“作或操作”。“a|=b”表示a和b作或操作，结果赋给a。） 

## 浏览器渲染页面过程

1. 解析HTML文件，创建DOM树

自上而下，遇到任何样式（link、style）与脚本（script）都会阻塞（外部样式不阻塞后续外部脚本的加载）。

2. 解析CSS

优先级：浏览器默认设置<用户设置<外部样式<内联样式<HTML中的style样式；

特定级：id数*100+类或伪类数*10+tag名称*1

3. 将CSS与DOM合并，构建渲染树（renderingtree）

DOM树与HTML一一对应，渲染树会忽略诸如head、display:none的元素

4. 布局和绘制，重绘（repaint）和重排（reflow）

重排：若渲染树的一部分更新，且尺寸变化，就会发生重排；

重绘：部分节点需要更新，但不改变其他集合形状。如改变某个元素的颜色，就会发生重绘。


## UDP怎么才能实现可靠

自定义通讯协议，在应用层定义一些可靠的协议，比如检测包的顺序，重复包等问题，如果没有收到对方的ACK，重新发包

UDP没有Delievery Garuantee，也没有顺序保证，所以如果你要求你的数据发送与接受既要高效，又要保证有序，收包确认等，你就需要在UDP协议上构建自己的协议。比如RTCP，RTP协议就是在UPD协议之上专门为H.323协议簇上的IP电话设计的一种介于传输层和应用层之间的协议。

下面分别介绍三种使用UDP进行可靠数据传输的协议
RUDP
RTP
UDT

RUDP（Reliable User Datagram Protocol）
可靠用户数据报协议（RUDP）是一种基于可靠数据协议（RDP： RFC908 和 1151 （第二版））的简单分组传输协议。作为一个可靠传输协议，RUDP 用于传输 IP 网络间的电话信号。它允许独立配置每个连接属性，这样在不同的平台可以同时实施不同传输需求下的协议。
RUDP 提供一组数据服务质量增强机制，如拥塞控制的改进、重发机制及淡化服务器算法等，从而在包丢失和网络拥塞的情况下， RTP 客户机（实时位置）面前呈现的就是一个高质量的 RTP 流。在不干扰协议的实时特性的同时，可靠 UDP 的拥塞控制机制允许 TCP 方式下的流控制行为。
为了与网络 TCP 通信量同时工作，
RUDP 使用类似于 TCP 的重发机制和拥塞控制算法。
在最大化利用可用带宽上，这些算法都得到了很好的证明。
RUDP特性
客户机确认响应服务器发送给客户机的包；
视窗和拥塞控制，服务器不能超出当前允许带宽；
一旦发生包丢失，服务器重发给客户机；
比实时流更快速，称为“缓存溢出”。
用户数据报协议（UDP）

RTP（Real Time Protocol）
RTP，实时协议被用来为应用程序如音频，视频等的实时数据的传输提供端到端（end to end）的网络传输功能。传输的模型可以是单点传送或是多点传送。数据传输被一个姐妹协议——实时控制协议（RTCP）来监控，后者允许在一个大的多点传送网络上监视数据传送，并且提供最小限度的控制和识别功能。
RTP是被IETF在RFC1889中提出来的。顺带提及，RTP已经被接受为实时多媒体传送的通用标准。ITU-T跟IETF都在各自的系统中将这一协议标准化。

UDT（UDP-based Data Transfer Protocol）
基于UDP的数据传输协议（UDP-based Data Transfer Protocol，简称UDT）是一种互联网数据传输协议。UDT的主要目的是支持高速广域网上的海量数据传输，而互联网上的标准数据传输协议TCP在高带宽长距离网络上性能很差。 顾名思义，UDT建于UDP之上，并引入新的拥塞控制和数据可靠性控制机制。UDT是面向连接的双向的应用层协议。它同时支持可靠的数据流传输和部分可靠的数据报传输。 由于UDT完全在UDP上实现，它也可以应用在除了高速数据传输之外的其它应用领域，例如点到点技术（P2P），防火墙穿透，多媒体数据传输等等。

## 设计多线程下载一个1G文件
## 算法:股票交易 

## Cookie and Session

由于HTTP是一种无状态协议,服务器没有办法单单从网络连接上面知道访问者的身份,为了解决这个问题,就诞生了Cookie

Cookie实际上是一小段的文本信息。客户端请求服务器，如果服务器需要记录该用户状态，就使用response向客户端浏览器颁发一个Cookie

客户端浏览器会把Cookie保存起来。当浏览器再请求该网站时，浏览器把请求的网址连同该Cookie一同提交给服务器。服务器检查该Cookie，

以此来辨认用户状态。服务器还可以根据需要修改Cookie的内容。

实际就是颁发一个通行证，每人一个，无论谁访问都必须携带自己通行证。这样服务器就能从通行证上确认客户身份了。这就是Cookie的工作原理

cookie 可以让服务端程序跟踪每个客户端的访问，但是每次客户端的访问都必须传回这些Cookie，如果 Cookie 很多，这无形地增加了客户端与服务端的数据传输量，

而 Session 的出现正是为了解决这个问题。同一个客户端每次和服务端交互时，不需要每次都传回所有的 Cookie 值，而是只要传回一个 ID，这个 ID 是客户端第一次访问服务器的时候生成的， 而且每个客户端是唯一的。这样每个客户端就有了一个唯一的 ID，客户端只要传回这个 ID 就行了，这个 ID 通常是 NANE 为JSESIONID 的一个 Cookie。

### 使用cookie的缺点

如果浏览器使用的是cookie，那么所有的数据都保存在浏览器端，

cookie可以被用户禁止

cookie不安全(对于敏感数据，需要加密)

cookie只能保存少量的数据(大约是4k)，cookie的数量也有限制(大约是几百个)，不同浏览器设置不一样，反正都不多

cookie只能保存字符串

对服务器压力小

### 使用session的缺点

一般是寄生在Cookie下的，当Cookie被禁止，Session也被禁止

当然可以通过url重写来摆脱cookie

当用户访问量很大时，对服务器压力大

我们现在知道session是将用户信息储存在服务器上面,如果访问服务器的用户越来越多,那么服务器上面的session也越来越多, session会对服务器造成压力，影响服务器的负载.如果Session内容过于复杂，当大量客户访问服务器时还可能会导致内存溢出。

用户信息丢失, 或者说用户访问的不是这台服务器的情况下,就会出现数据库丢失.

### cookie和session的区别

具体来说cookie机制采用的是在客户端保持状态的方案，而session机制采用的是在服务器端保持状态的方案。同时我们也看到， 由于采用服务器端保持状态的方案在客户端也需要保存一个标识，所以session机制可能需要借助于cookie机制来达到保存标识的目的

cookie不是很安全，别人可以分析存放在本地的cookie并进行cookie欺骗，考虑到安全应当使用session

session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能，考虑到减轻服务器性能方面，应当使用cookie

单个cookie保存的数据不能超过4k,很多浏览器都限制一个站点最多保存20个cookie。

可以将登陆信息等重要信息存放为session。

## https中间人攻击

针对SSL的中间人攻击方式主要有两类，分别是SSL劫持攻击和SSL剥离攻击

1. SSL劫持攻击
SSL劫持攻击即SSL证书欺骗攻击，攻击者为了获得HTTPS传输的明文数据，需要先将自己接入到客户端和目标网站之间；在传输过程中伪造服务器的证书，将服务器的公钥替换成自己的公钥，这样，中间人就可以得到明文传输带Key1、Key2和Pre-Master-Key，从而窃取客户端和服务端的通信数据；

但是对于客户端来说，如果中间人伪造了证书，在校验证书过程中会提示证书错误，由用户选择继续操作还是返回，由于大多数用户的安全意识不强，会选择继续操作，此时，中间人就可以获取浏览器和服务器之间的通信数据

2. SSL剥离攻击
这种攻击方式也需要将攻击者设置为中间人，之后见HTTPS范文替换为HTTP返回给浏览器，而中间人和服务器之间仍然保持HTTPS连接。由于HTTP是明文传输的，所以中间人可以获取客户端和服务器传输数据

## UIKit view 生命周期

在运行时展示View
UIKit极大的简化了加载和展示View的过程，它大概会按照以下顺序执行一些任务：
- 通过storyboard文件中的信息实例化视图
- 连接outlet和action
- 把根视图赋值给UIViewController的view属性（其实就是调用loadView 方法）
- 调用UIViewController的awakeFromNib方法。要注意，在调用方法前，的trait collecion为空且子视图的位置可能不正确
- 调用UIViewController的viewDidLoad方法。

此时已经完成了视图的加载工作，在展示到屏幕之前，还有以下几个步骤：
- 调用UIViewController的viewWillAppear方法。
- 更新视图的布局
- 把视图展示到屏幕上
- 调用UIViewController的viewDidAppear方法。

### awakeFromNib方法

至此，第一个问题已经几乎解释完了，还剩一个awakeFromNib方法。

我们已经知道，awakeFromNib方法被调用时，所有视图的outlet和action已经连接，但还没有被确定。这个方法可以算作是和视图控制器的实例化配合在一起使用的，因为有些需要根据用户喜好来进行设置的内容，无法存在storyboard中，所以可以在awakeFromNib方法中被加载进来。

awakeFromNib方法在视图控制器的生命周期内只会被调用一次。因为它和视图控制器从nib文件中的解档密切相关，和view的关系却不大。

### loadView方法
当执行到loadView方法时，视图控制器已经从nib文件中被解档并创建好了，接下来的任务主要是对view进行初始化。

loadView方法在UIViewController对象的view属性被访问到且为空的时候调用。
这是它与awakeFromNib方法的一个区别。假设我们在处理内存警告时释放view属性（其实并不应该这么做，这里举个例子）：self.view = nil。因此loadView方法在视图控制器的生命周期内可能会被多次调用。

这个方法不应该被直接调用，而是由系统自动调用。它会加载或创建一个view并把它赋值给UIViewController的view属性。

在创建view的过程中，首先会根据nibName去找对应的Nib文件然后加载。如果nibName为空，或找不到对应的Nib文件，则会创建一个空视图(这种情况一般是纯代码，也就是为什么说代码构建View的时候，要重写loadView 方法)。

注意在重写loadView方法的时候，不要调用父类的方法。

### viewDidLoad方法
loadView方法执行完之后，就会执行viewDidLoad方法。此时整个视图层次(view hierarchy)已经被放到内存中。

无论是从nib文件加载，还是通过纯代码编写界面，viewDidLoad方法都会执行。我们可以重写这个方法，对通过nib文件加载的view做一些其他的初始化工作。比如可以移除一些视图，修改约束，加载数据等。

### viewWillAppear和viewDidAppear方法
在视图加载完成，并即将显示在屏幕上时，会调用viewWillAppear方法，在这个方法里，可以改变当前屏幕方向或状态栏的风格等。

当viewWillAppear方法执行完后，系统会执行viewDidAppear方法。在这个方法中，还可以对视图做一些关于展示效果方面的修改。

### 视图的生命历程
到目前为止，我们已经了解了每个方法的作用，接下来就把整个流程梳理一遍。

1. `-[ViewController initWithCoder:]`或`-[ViewController initWithNibName:Bundle]:`首先从归档文件中加载UIViewController对象。即使是纯代码，也会把nil作为参数传给后者。
2. `-[ViewController awakeFromNib]:`作为第一个方法的助手，方便处理一些额外的设置。
3. `-[ViewController loadView]:`创建或加载一个view并把它赋值给UIViewController的view属性
4. `-[ViewController viewDidLoad]:`此时整个视图层次(view hierarchy)已经被放到内存中，可以移除一些视图，修改约束，加载数据等
5. `-[ViewController viewWillAppear:]:`视图加载完成，并即将显示在屏幕上,还没有设置动画，可以改变当前屏幕方向或状态栏的风格等。
6. `-[ViewController viewWillLayoutSubviews]：`即将开始子视图位置布局
7. `-[ViewController viewDidLayoutSubviews]：`用于通知视图的位置布局已经完成
8. `-[ViewController viewDidAppear:]：`视图已经展示在屏幕上，可以对视图做一些关于展示效果方面的修改。
9. `-[ViewController viewWillDisappear:]：`视图即将消失
10. `-[ViewController viewDidDisappear:]：`视图已经消失
如果考虑UIViewController可能在某个时刻释放整个view。那么再次加载视图时显然会从步骤3开始。因为此时的UIViewController对象依然存在。

