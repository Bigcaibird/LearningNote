# TCP
**Tcp**是面向连接的、可靠的、基于字节流的传输层通信协议。

+ 面向连接的：一对一的才能连接
+ 可靠的：无论网络出现怎么样变化，TCP 都是可以保证一个报文一定能到达对端
+ 字节流：消息是没有边界的，所以无论我们消息有多大都可以进行传输。
## 一. TCP基础
### TCP消息格式
<div align=center><img width='600' height='300' src=./image/Tcp首部.jpg> </div>

+ 首部长度：“选项” 没有使用就是20个字节，使用了就靠首部长度获悉。
+ 序号*Seq*  
    **用于对字节流进行编号**。例如序号为 301，表示第一个字节的编号为 301，如果携带的数据长度为 100 字节，那么下一个报文段的序号应为 401
    
    > 序号用来标识从TCP发端向TCP收端发送的数据字节流，它表示在这个报文段中的的第一个数据字节。如果将字节流看作在两个应用程序间的单向流动，则 **TCP用序号对每个字节进行计数**。序号是32 bit的无符号数，序号到达`2^32-1`后又从0开始。
+ 确认序号*Ack*  
    确认序号， 即**期望收到的下一个报文段的序号**，其值是上次已成功收到数据字节序号 *seq* 加上携带的数据长度：*Ack = seq + length* 。只有 *ACK*  标志为 1 时确认序号字段才有效。
    
    > 例如 B 正确收到 A 发送来的一个报文段，序号为 501，携带的数据长度为 200 字节，因此 B 期望下一个报文段的序号为 701，B 发送给 A 的确认报文段中确认号就为 701。
    > 
    
    
      序号和确认序号是保证数据可靠性的重要方式。
    
       + 序号：用来解决的是 **网络包乱序的问题**
       + 确认序号：用来解决**不丢包**的问题
    
      小写的**Ack**一般表示确认序号，大写的**ACK**一般表示标志位
+ 标志位 : *URG，ACK，PSH，RST，SYN，FIN*
    + ACK：`ACk=1`时，确认序号才有效。TCP 规定除了最初建立连接时的`SYN`包之外该位必须设置为1。 
    + RST：`RST=1`时，表示`TCP`连接出现异常，需要重建连接。
    + SYN：`SYN=1`时，表示发起一个连接。
    + FIN：`FIN=1`时，表示的发送端**不会再有数据发送**，希望断开连接。
+ 窗口大小  
    为流量控制而设计。窗口值作为接收方让发送方设置其发送窗口的依据。之所以要有这个限制，是因为接收方的数据缓存空间是有限的。窗口大小是一个 16 bit字段，因而窗口大小最大为65535字节。
### TCP 建立连接  
<div align=center><img width='500' height='200' src=./image/tcp三次握手.jpg> </div>

**过程**
通常服务端是被动打开，客户端调用 *connect* 函数实现主动打开，开启 TCP 三次握手：

+ 第一次握手
    *client* 调用 *connect* 函数发起主动打开。发送请求报文： **SYN=1, ACK=0** 并随机选择一个初始序号*seq=J*。Client端进入***SYN_SENT*** 状态，等待*Server* 端确认应答。**仅有此处的ACK=0 ** 
+ 第二次握手
    *Server*端接受到请求报文后，如果同意建立连接，则向客户端发送连接确认报文：**SYN=1,ACK=1**，确认序号*Ack=seq+1=J+1*，同时也会选择一个初始的序号*seq=K*。此时服务端进入***SYN_RECV***状态。
+ 第三次握手
    *Client* 端收到 *Server* 的连接确认报文后，还要向 *Server* 发出确认应答**ACK**，确认序号为*seq=K+1*。*Server* 收到 *Client* 的确认后，连接建立。双方都进入 ***ESTABLISHED*** 状态。完成三次握手。**因此，此次应答ACK，客户端是可以携带应用层数据的**。

***socket* 编程**  

+ 客户端：*connect* 函数返回时，是第二次握手返回

+ 服务端：*accept* 在第一次握手时阻塞，*accept* 返回是第三次握手成功。

+ *listen* 函数的 *backlog* 的意义  
  
  *Linux*内核中会维护两个队列，*backlog* 大小就是指定的已完成连接队列的大小
  
  + 未完成连接队列（***SYN*** 队列）：接收到一个 SYN 建立连接请求，处于 ***SYN_RCVD*** 状态  
  + 已完成连接队列（***Accpet*** 队列）：已完成 TCP 三次握手过程，处于 ***ESTABLISHED*** 状态  

**[问题1]：三次握手建立连接时，为什么需要第三次？**  

+ 三次握手才可以阻止历史重复连接的初始化（主要原因）
+ 三次握手才可以同步双方的初始序列号
+ 三次握手才可以避免资源浪费

三次握手就是防止因为网络阻塞导致的不确定性。

<div align=center><img width='500' height='600' src=./image/tcp三次握手的主要原因.webp> </div>

**[原因1]**：如果因为网络拥堵导致客户端发出的旧 **SYN** 报文比新的 **SYN** 报文先到达，服务端会发出应答 **SYN + ACK**。客户端可以判断 **SYN** 是一个历史连接，在第三次握手中返回 **RST** 给服务端，中断这个连接。如果是两次握手，客户端就没有足够的上下文来判断这个过时的 **SYN** 是否过时，那么就不能中断这次连接。

+ 如果是历史连接（序列号过期或超时），则第三次握手发送的报文是**RST** 报文，以此中止历史连接；
+ 如果不是历史连接，则第三次发送的报文是 **ACK** 报文，通信双方就会成功建立连接；

**[原因3]**：如果客户端的 **SYN** 阻塞了，客户端会重复发送多次 **SYN** 报文，那么服务器在收到请求后就会建立多个冗余的无效链接，造成不必要的资源浪费。

#### [SYN 攻击](https://zhuanlan.zhihu.com/p/81144898)
因为**TCP** 连接建立是需要三次握手，假设攻击者短时间伪造不同 IP 地址的 SYN 报文，服务端每接收到一个 SYN 报文，就进入***SYN_RCVD*** 状态，但服务端发送出去的 ***ACK + SYN*** 报文，无法得到未知 IP 主机的 ACK 应答，久而久之就会占满服务端的 SYN 接收队列（**半连接队列**），使得服务器不能为正常用户服务。
### TCP断开连接  

<div align=center><img width='500' height='300' src="./image/tcp断开链接.jpg"> </div>

TCP的终止需要四个分节。
+ 第一次握手：*Client*  端发送一个 *FIN=1, seq=M*  给服务端。*Client*进入 **FIN_WAIT_1** 状态。
+ 第二次握手：*Server*  接收到该报文后，发送确认应答 *ACK=1,  ack=M+1* 给 *Client*。*Server* 进入  **CLOSE_WAIT** 状态，客户端进入半关闭  **FIN_WAIT_2**  状态。
+ 第三次握手：*Server* 也准备关闭时，发送 *FIN=1, seq = N* 给客户端。*Server* 进入 **LAST_ACK** 状态。
+ 第四次握手：*Client* 接受到后，*Client* 端进入  **TIME_WAIT**  状态。接着发送  *ACK=1, seq=N+1*  给 *Server*。*Server* 接受到后进入 **CLOSED** 状态。至此服务端已经完成关闭。客户端在经过  **2MSL**  时间后，自动进入 **CLOSED** 状态。至此，客户端也完成了连接。  
  

**[注意]**：

+ 主动关闭方才有 **TIME_WAIT** 状态，被动关闭方才有**CLOSE_WAIT**状态。
+ 服务端出现  **CLOSE_WAIT**  状态后，接受不到数据了，但是还能发送数据。

**socket编程**：

+ 在*scoket*编程中，调用 *close/shutdown*，对端会接受到*EOF* 标志位，得知对端不再发送数据。
  

**[问题1]：为什么建立连接只需要三次，关闭需要四次握手？**  
建立连接时，四次握手其实也能够可靠的连接，但由于第二步和第三步可以优化成一步，所以就成了三次握手。因此，「三次握手」就已经理论上最少可靠连接建立，所以不需要使用更多的通信次数。

关闭连接时，由于TCP连接时全双工的每个方向都必须要单独进行关闭。每次关闭发送`FIN`+对端`ACK`需要两次握手，且服务端的*ACK* 和 *FIN* 分开发送，从而比三次握手导致多了一次。

[问题2]：为什么需要 **TIME_WAIT** ？   
需要 **TIME_WAIT**  状态，主要是两个原因：

+ 防止具有相同四元组的「旧」数据包被收到；
+ 保证「被动关闭连接」的一方能被正确的关闭，即保证最后的 `ACK` 能让被动关闭方接收，从而帮助其正常关闭；

[原因1]：如果没有 **TIME_WAIT**  或者这个状态时间太短。那么在这个四元组关闭之前发出的消息，有可能在这组四元组被复用重新建立连接后，之前延迟的报文到达这对新的连接，那么会导致新的连接数据错乱。

因此，TCP设计了这么一个状态，经过 **2MSL** 的时间，足以让两个方向上的数据包都在网络中消失，再出现的数据包一定是新建立的连接产生的。

[原因2]：如果在第四次握手中，客户端发送的最后一个**ACK**在网络中丢失了。如果没有这个状态或者这个状态时间过短，直接进入**CLOSED**状态，那么服务端就会一直处于**LAST_ACK**状态。如此，当客户端发起**SYN**建立连接请求，服务端会发送**RST**报文给客户端，无法建立连接。 

因此，设计这个状态：如果服务端正常收到四次挥手的最后一个 `ACK` 报文，则服务端正常关闭连接。否则服务端没有收到四次挥手的最后一个 `ACK` 报文时，则会重发 `FIN` 关闭连接报文并等待新的 `ACK` 报文。

> **MSL** 是 **Maximum Segment Lifetime** ，报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃

[问题3]：**TIME_WAIT**过多有什么危害？  
过多的 **TIME_WAIT** 状态主要的危害有两种：

+ 内存资源占用；
+ 对端口资源的占用，一个 TCP 连接至少消耗一个本地端口。

如果服务端 **TIME_WAIT** 状态过多，占满了所有端口资源，则会导致无法创建新连接。

**[问题4]：如果消除`TIME_WAIT` 过多的影响**

详情参考下面的`SO_linger`选项，可以使用这个选项来跳过`TIME_WAIT`

**TCP 状态图**  
TCP为连接定义了11种状态，如下图是11种状态之间的转换。

<div align=center> <img width = '500' height ='500' src =./image/tcp状态转换.jpg></div>

## 二. TCP的可靠传输
### TCP 的重传机制
TCP实现可靠传输的方的核心就是通过**序号**与**确认序号**。但是针对网络环境较差可能存在数据丢包的情况下，使用重传机制来解决：
+ 超时重传
+ 快速重传
+ SACK
+ D-SACK

**超时重传**   
重传机制的其中一个方式，就是在发送数据时，设定一个定时器，**当超过指定的时间后，没有收到对方的 `ACK` 确认应答报文，就会重发该数据**，也就是我们常说的超时重传。

TCP 会在以下两种情况发生超时重传：
+  数据包丢失：接收端没有接收到数据
+ 确认应答丢失：接收到数据，但是返回的`ACk 端没有接收到
  

在超时重传在，关键在于如何计算超时重复时间(`Retransmission Timeout`, `RTO`)：
+ 当超时时间 RTO 较大时，重发就慢，丢了老半天才重发，没有效率，性能差；
+ 当超时时间 RTO 较小时，会导致可能并没有丢就重发，于是重发的就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发

一般，`RTO`计算方式和包的往返时间(`Round-Trip Time, RTT`)相关（具体的计算方式，见相关书籍）。`RTO`是一个动态的计算方式，根据`RTT`变动。

> `RTT`计算的时间是：从发出数据包到接收到对端的`ACK`应答的时间

**快速重传**    
快速重传的最大特点是**不以时间为驱动，而是以数据驱动重传**。

+ 工作方式：快速重传的工作方式是**当发送端收到三个相同的 `ACK` 报文时，会在定时器过期之前，重传丢失的报文段**。
+ 解决的问题：快速重传机制只解决了一个问题，就是超时时间的问题。但是它依然面临着另外一个问题：就是重传的时候，是重传之前的一个，还是重传所有的问题

在下图中，接收端没有收到`seq=2`的数据包，无论后面接收到什么数据，总是向发送端发送缺乏的那个数据包的应答`ACK=2`，发送端收到了三个 `ACK = 2` 的确认，知道了 `Seq2` 还没有收到，就会在定时器过期之前，重传丢失的 `Seq2`。

<div align=center> <img width = '300' height ='300' src =./image/快速重传.png></div>

但是，比如对于上面的例子，是重传 `Seq2` 呢？还是重传 `Seq2、Seq3、Seq4、Seq5` 呢？因为发送端并不清楚这连续的三个 `ACK 2` 是谁传回来的。

**SACK**   
这个重传机制，就是为了解上面快速重传机制中存在问题。

这种方式需要在 TCP 头部 **「选项」** 字段里加一个 *SACK* 的东西，它可以将缓存的地图发送给发送方，这样发送方就可以知道哪些数据收到了，哪些数据没收到，知道了这些信息，就可以只重传丢失的数据。

如下图，发送方收到了三次同样的 ACK 确认报文，于是就会触发快速重发机制，通过 SACK 信息发现只有 200~299 这段数据丢失，则重发时，就只选择了这个 TCP 段进行重发。
<div align=center> <img width = '600' height ='450' src =./image/SACK.png></div>

**D-SACK**  
`D-SACK`(`Duplicate SACK`)其主要使用了`SACK`来告诉「发送方」有哪些数据被重复接收了。

针对网络延迟和数据丢包，很有作用：
+ 可以让「发送方」知道，是发出去的包丢了，还是接收方回应的 ACK 包丢了;
+ 可以知道是不是「发送方」的数据包被网络延迟了;

### 滑动窗口   
滑动窗口的设计是为了应对：`TCP`每发一个数据包，都要进行一次应答而导致的低效率问题。有了窗口，就可以指定窗口大小，**窗口大小就是指无需等待确认应答，而可以继续发送数据的最大值**。

因此，发送方主机在等到确认应答返回之前，必须在缓冲区中保留已发送的数据。如果按期收到确认应答，此时数据就可以从缓存区清除。

**注意：** 滑动窗口的大小是由 **接受方**的窗口决定的。TCP 头里有一个16位的字段叫 `Window`，也就是**窗口大小**。这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来

**累计确认/累计应答**

下图中的 接受方的`ACK 600` 确认应答报文在发送过程中丢失，也没关系，因为可以通过**下一个确认应答**进行确认，只要发送端收到了` ACK 700` 确认应答，就意味着 700 之前的所有数据「接收方」都收到了。这个模式就叫累计确认或者累计应答。

<div align=center> <img width = '400' height ='300' src =./image/滑动窗口.png></div>

**发送端窗口与接收端窗口**

+ 发送端数据可以分为四个部分，如图
  + #1 已发送并收到`ACK`确认的数据：1~31 字节
  + #2 已发送但未收到`ACK`确认的数据：32~45 字节
  + #3 未发送但总大小在接收方处理范围内（接收方还有空间）：46~51字节
  + #4 未发送但总大小超过接收方处理范围（接收方没有空间）：52字节以后

<div align=center> <img width = '600' height ='180' src =./image/发送端滑动窗口.png></div>

+ 接收端数据分为三个部分，如图
  + #1 + #2 是已成功接收并确认的数据（等待应用进程读取）；
  + #3 是未收到数据但可以接收的数据；
  + #4 未收到数据并不可以接收的数据；

<div align=center> <img width = '600' height ='180' src =./image/接受端滑动窗口.png></div>

整个滑动窗口是变化的，是根据接受端的滑动窗口变化的。
+ 发送端，之前发送出去未接收到ACK的数据（比如上图的32~45字节）收到对端的`ACK`应答后，在滑动窗口大小没有改变的情况下，窗口右边就会右移动被确认的字节数。
+ 接收端：接收端需要等应用层取走**已经接受成功并确认的数据**，才会右移滑动滑动窗口。

由于滑动窗口是动态变化的，因此二者并不是完全相等，接收窗口的大小是约等于发送窗口的大小的。注意，上面图中**深蓝色的才是发送/接受端窗口**。

### 流量控制

为了防止发送端发送过多数据量导致接受端处理不过来，TCP 提供一种机制可以让「发送方」根据「接收方」的实际接收能力控制发送的数据量，这就是所谓的流量控制。

发送方窗口大小会根据接受方的窗口大小进行动态的调整。

**窗口关闭**  
TCP 通过让接收方指明希望从发送方接收的数据大小（窗口大小）来进行流量控制。如果窗口大小为 0 时，就会阻止发送方给接收方传递数据，直到窗口变为非 0 为止，这就是窗口关闭。

> 窗口关闭存在的危险 
>
接收方向发送方通告窗口大小时，是通过 ACK 报文来通告的。当发生窗口关闭时，接收方处理完数据后，接受端会向发送端通告一个窗口非 0 的 `ACK` 报文，如果这个通告窗口的 `ACK` 报文在网络中丢失了，这就会导致发送方一直等待非0的通知，而接受方在发出`ACK`之后就开始等待发送方的数据，如此就会产生死锁。
>避免死锁

为了解决这个问题，TCP 为每个连接设有一个持续定时器，只要 TCP 连接一方收到对方的零窗口通知，就启动持续计时器。如果持续计时器超时，就会发送窗口探测 ( *Window probe* ) 报文，而对方在确认这个探测报文时，给出自己现在的接收窗口大小。

**Nagle算法** 
发送方的数据只有几个字节，为了发送这几个字节却需要添加上**TCP+IP**共有40个字节的头部，开销很大。为此，需要禁止*Nagle*算法，不让小数据量单独发送，而是等几个小数据量合并为大的数据包再发送。

在 *socket* 设置 **TCP_NODELAY** 选项来关闭这个算法（关闭 *Nagle* 算法没有全局参数，需要根据每个应用自己的特点来关闭）
```c
int value=1;
setsockopt(sock_fd, IPPROTO_TCP, TCP_NODELAY, &value, sizeof(int));
```

### 拥塞控制

流量控制通过滑动窗口来实现，但是滑动窗口只考虑了发送方窗口(`send window, swnd`)和接收方窗口(`recv window, rwnd`)的问题，没考虑网络的问题。有可能接收端很快，但是网络拥塞了，所以加了一个拥塞窗口(`crowded window, cwnd`)。**拥塞窗口的意思就是一次性可以连续提交多少个包到网络中**。最终发送窗口大小 $swnd = min(cwnd, rwnd)$ 由两个窗口共同控制发送速度。

TCP的拥塞控制主要避免两种现象：包丢失和包重传。网络的带宽是固定的，当发送端发送速度超过带宽后，中间设备处理不完多出来的包就会被丢弃，这就是包丢失。如果我们在中间设备上加上缓存，处理不过来的包就会被加到缓存队列中，不会丢失，但是会增加时延。如果时延到达一定的程度，就会超时重传，这就是包重传。

*cwnd*的变化规则：

+ 只要网络中没有出现拥塞，*cwnd* 就会增大
+ 网络中出现了拥塞，*cwnd* 就会减少
  

拥塞控制主要是四个算法：
+ 慢启动
+ 拥塞避免
+ 拥塞发生
+ 快速恢复。

**[1-慢启动]**   
TCP 在刚建立连接完成后，首先是有个慢启动的过程，这个慢启动的意思就是一点一点的提高发送数据包的数量。慢启动的算法记住一个规则就行：当发送方每收到一个 `ACK`，就拥塞窗口 `cwnd` 的大小就会加 1。初始化为1个`MSS`的大小。

慢启动是指数型增长

<div align=center> <img width = '450' height ='300' src =./image/慢启动.png></div>

慢启动有个阈值*ssthresh(slow start thresh)*：
+ `cwnd < ssthresh` : 使用**慢启动**算法
+ `cwnd >= ssthresh` : 使用**拥塞避免**算法

**[2-拥塞避免]**  
`ssthresh`一般是 65535个字节。当 `cwnd` 超过 `ssthresh` 就会进入拥塞避免算法。此时规则：每当接受到一个`ACK`。

接着慢启动的例子，且假设`ssthresh=8`。当 8 个 `ACK` 应答确认到来时，每个确认增加 $1/8$，8 个 ACK 确认 `cwnd` 一共增加 1，于是这一次能够发送 `9 个 MSS` 大小的数据，变成了线性增长。

<div align=center> <img width = '450' height ='300' src =./image/拥塞避免.png></div>

拥塞避免算法是将原本慢启动算法的指数增长变成了线性增长。一直增长着，网络就会慢慢进入了拥塞的状况了，于是就会出现**丢包现象**，这时就需要对丢失的数据包进行重传。

> 当触发了重传机制，也就发生了拥塞，也就进入了拥塞发生算法

**[3-拥塞发生]**
当网络出现拥塞，也就是会发生数据包重传，重传机制主要有两种：

+ 超时重传   
  超时重传发生如下改变：
  
  + `ssthresh = cwnd/2`，
+ `cwnd = 1`    
  
  然后重新从头慢启动。但是慢启动是会突然减少数据流的，会造成网络卡顿
+ 快速重传   
  前面我们讲过「快速重传算法」。当接收方发现丢了一个中间包的时候，发送三次前一个包的 ACK，于是发送端就会快速地重传，不必等待超时再重传。

  TCP 认为这种情况不严重，因为大部分没丢，只丢了一小部分，则 ssthresh 和 cwnd 变化如下：
  + *cwnd = cwnd/2*
  + *ssthresh = cwnd*
  + 进入快速恢复算法

**[4-快速恢复]**

接着上一段拥塞发生的第二种情况，快速恢复算法的逻辑如下: 
+ `cwnd = sshthresh + 3 * MSS` （3的意思是确认有3个数据包被收到了）。
+ 重传丢失的数据包
+ 如果再收到重复的`ACK`，那么 `cwnd = cwnd +1`
+ 如果收到了新的Ack，那么 `cwnd = sshthresh` ，然后就进入了拥塞避免的算法了。
  

如此，就是避免了再次进入慢启动了。

<div align=center> <img width = '480' height ='350' src =./image/快速恢复.png></div>

## 面试问题 
#### TCP 短连接和长连接的区别  
+ 短连接    
  Client 向 Server 发送消息，Server 回应 Client，然后一次读写就完成了，这时候双方任何一个都可以发起 close 操作，不过一般都是 Client 先发起 close 操作。短连接一般只会在 Client/Server 间传递一次读写操作。

  短连接的优点：管理起来比较简单，建立存在的连接都是有用的连接，不需要额外的控制手段。
+ 长连接  
  *Client* 与 *Server* 完成一次读写之后，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。

  在长连接的应用场景下，*Client* 端一般不会主动关闭它们之间的连接，*Client* 与 *Server* 之间的连接如果一直不关闭的话，随着客户端连接越来越多，*Server* 压力也越来越大，这时候 *Server* 端需要采取一些策略，如关闭一些长时间没有读写事件发生的连接，这样可以避免一些恶意连接导致 *Server* 端服务受损；如果条件再允许可以以客户端为颗粒度，限制每个客户端的最大长连接数，从而避免某个客户端连累后端的服务。

  长连接和短连接的产生在于 *Client* 和 *Server* 采取的关闭策略，具体的应用场景采用具体的策略。

#### TCP 粘包、拆包及其解决办法  
UDP 是基于报文发送的，UDP首部采用了 16bit 来指示 UDP 数据报文的长度，因此在应用层能很好的将不同的数据报文区分开，从而避免粘包和拆包的问题。

而 TCP 是基于字节流的，虽然应用层和 TCP 传输层之间的数据交互是大小不等的数据块，但是 TCP 并没有把这些数据块区分边界，仅仅是一连串没有结构的字节流。另外从 TCP 的帧结构也可以看出，在 TCP 的首部没有表示数据长度的字段，基于上面两点，在使用 TCP 传输数据时，才有粘包或者拆包现象发生的可能。

**发生原因**  

+ 要发送的数据大于 TCP 发送缓冲区剩余空间大小，将会发生拆包。
+ 待发送数据大于 MSS（最大报文长度），TCP 在传输前将进行拆包。
+ 要发送的数据小于 TCP 发送缓冲区的大小，TCP 将多次写入缓冲区的数据一次发送出去，将会发生粘包。
+ 接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包
  

**解决办法**   
由于 TCP 本身是面向字节流的，无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的，这个问题 **只能通过上层的应用协议栈设计来解决**，根据业界的主流协议的解决方案，归纳如下：

+ 使得消息定长：发送端将每个数据包封装为固定长度（不够的可以通过补 0 填充），这样接收端每次接收缓冲区中读取固定长度的数据就自然而然的把每个数据包拆分开来。
+ 设置消息边界：服务端从网络流中按消息边界分离出消息内容。在包尾增加回车换行符进行分割，例如 FTP 协议。
+ 将消息分为消息头和消息体：消息头中包含表示消息总长度（或者消息体长度）的字段。更复杂的应用层协议比如 Netty 中实现的一些协议都对粘包、拆包做了很好的处理

#### 为什么Tcp实现不了实时传输
流量控制和拥塞控制

#### TCP协议和UDP协议的区别是什么
+ 连接：`TCP` 协议是面向连接的协议，传输数据前先建立连接。而 `UDP` 是不需要连接，即可传输数据
+ 可靠性：`TCP` 是可靠交付数据的，数据可以无差错、不丢失、不重复到达。`UDP` 是尽最大努力交付，不保证可靠交付数据。
+ 服务对象：`TCP` 是点对点的服务。`UDP`支持一对一、一对多
+ 拥塞控制和流量控制：`TCP` 有拥塞控制和流量控制机制，保证数据传输的安全性。`UDP` 则没有，即使网络非常拥堵了也不会影响 `UDP` 的发送速率。
+ 首部开销：`TCP` 首部长度较长，会有一定的开销，首部在没有使用「选项」字段时是 20 个字节，如果使用了「选项」字段则会变长的。`UDP` 首部只有 8 个字节，并且是固定不变的，开销较小。
#### 以下应用一般或必须用udp实现？
+ 多播的信息一定要用udp实现，因为tcp只支持一对一通信。
+ 如果一个应用场景中大多是简短的信息，适合用udp实现，因为udp是基于报文段的，它直接对上层应用的数据封装成报文段，然后丢在网络中，如果信息量太大，会在链路层中被分片，影响传输效率。
+ 如果一个应用场景重性能甚于重完整性和安全性，那么适合于udp，比如多媒体应用，缺一两帧不影响用户体验，但是需要流媒体到达的速度快，因此比较适合用udp
+ 如果要求快速响应，那么udp听起来比较合适如果又要利用udp的快速响应优点，又想可靠传输，那么只能考上层应用自己制定规则了。
+ 常见的使用udp的例子：QQ的聊天模块。

#### UDP 怎么做到可靠传输
在udp头部有个校验和，使得`UDP`传输更加可靠

#### 为什么UDP可以用于数据实时传输的场景？
不需要建立连接即可传输数据，没有流量控制和拥塞控制，不会影响网络数据的传输速率。

### 常见的 socket 选项  

严格意义上说套接字选项是有不同层级的（level），如socket级别、TCP级别、IP级别，这里我们不区分具体的级别。

**1. SO_SNDTIMEO 与 SO_RCVTIMEO**  
这两个选项用于设置阻塞模式下套接字，

+ `SO_SNDTIMEO`用于在send数据由于对端tcp窗口太小，发不出去而最大的阻塞时长；
+ `SO_RCVTIMEO`用于recv函数因接受缓冲区无数据而阻塞的最大阻塞时长。  

如果你需要获取它们的默认值，请使用getsockopt函数。

**2. TCP_NODELAY**  
禁止 `nagle` 算法  

操作系统底层协议栈默认有这样一个机制，为了减少网络通信次数，会将send等函数提交给tcp协议栈的多个小的数据包合并成一个大的数据包，最后再一次性发出去，也就是说，如果你调用send函数往内核协议栈缓冲区拷贝了一个数据，这个数据也许不会马上发到网络上去，而是要等到协议栈缓冲区积累到一定量的数据后才会一次性发出去，我们把这种机制叫做`nagle`算法。默认打开了这个机制，有时候我们希望关闭这种机制，让`send`的数据能够立刻发出去，我们可以选择关闭这个算法，这就可以通过设置套接字选项`TCP_NODELAY`，即关闭`nagle`算法。

**3. SO_LINGER**

这个选项的用处是 **用于解决，当需要关闭套接字时，协议栈发送缓冲区中尚有未发送出去的数据，等待这些数据发完的最长等待时间**。

```cpp
  struct linger { 
    int l_onoff;  // 是否开启
    int l_linger; // 多久
  };
```

+ 默认情况下，`l_onoff=0` 即关闭本选项，`l_linger` 的值被忽略，此时 `close(fd)`函数立即返回，如果发送缓冲区中有数据会尝试发送。
+ `l_onoff=1, l_linger=0`：此时 `close(fd)` 行为是 TCP将立即中止这个连接，TCP会丢弃发送缓冲区中的所有数据并且发送一个 `RST` 给对端，而且**结束过程没有四次握手过程。这样就避免了 `TIMEWAIT` 状态**。
+ `l_onoff=1, 1_linger=val`：此时 `l_linger` 非0。 那么 `close(fd)` 内核将会拖延一段时间。如果发送缓冲区中仍有数据，那么进程将会投入睡眠，直到 
  + 所有数据都已经发送完+被对方确认
  + 时间到

如果 sockfd 被设置为非阻塞，将不会等待 close 完成，即使设置了 `l_onoff=1, 1_linger=val` 也是如此。因此，在非阻塞模式下检测错误返回值很重要。如果在非阻塞模式下，延迟时间到数据没有发送完且被对端确认，那么发送缓冲区中的剩余数据将会被丢弃，并返回错误：**EWOULDBLOCK** 错误。

`close`返回只是说明：先前发送的数据和 `FIN` 已由对端确认，而不能告诉我们对端应用进程是否已经读取了数据。如果不设置这个选择，连对端确认都不知道。

`shutdown(fd, SHUT_WR)` + `read` 可以知道对端的是否已经读取了数据，并且等待对端 `close` 服务端。

<div align=center> <img width =600, height=250 src=./image/shutdown获取对端信息.jpg></div>

**4. SO_REUSEADDR/SO_REUSEPORT**

一个端口，尤其是作为服务器端端口在四次挥手的最后一步，有一个为`TIME_WAIT`的状态，这个状态一般持续2MSL，占据着ip地址和端口，使其不能被立即复用。为了立即回收复用ip地址端口号，我们可以通过开启套接字 `SO_REUSEADDR/SO_REUSEPORT`。

**6. SO_KEEPALIVE**

默认情况下，当一个连接长时间没有数据来往，会被系统防火墙之类的服务关闭。为了避免这种现象，尤其是一些需要长连接的应用场景下，我们需要使用心跳包机制，即定时从两端定时发一点数据，这种行为叫做“**保活**”。

而 `tcp` 协议栈本身也提供了这种机制，那就是设置套接字 `SO_KEEPALIVE` 选项，开启这个选项后，`tcp` 协议栈会定时发送心跳包探针，但是这个默认时间比较长（2个小时），我们可以继续通过相关选项改变这个默认值


### 错误类型

**1. accept 返回前连接中止**

在三次握建立连接之后，客户端 `Tcp` 发送了一个 `RST` 分节。在服务端上，即在准备`Accept`的时候`RST`到达，那么此时服务端调用 `accept` 会 `accept` 返回0-1，并且 `erron=ECONNABORTED`。解决办法是：重新调用一次 `accept` 即可

<div align=center> <img width =500, height=300 src=./image/accept返回前出错.jpg></div>

**2. 服务器进程终止**

服务端终止，会给客户端发送一个**FIN**报文，客户端回应**ACK**。

因为客户端接受到 `FIN` 只是表示服务端不再发送数据，并不知道服务端已经终止。因此当客户端向已终止的服务端写数据时，之前响应打开的`socket` 已经关闭，因为服务会回应一个`RST` 给客户端。如果在收到 `RST` 之前调用`read`，会接收到未预料的`0`，表示`EOF`。如果在`RST`之后调用`read`会返回错误：`ECONNREST`，表示”对方服务连接错误。

**3. SIGPIPE**  
针对上面的情况，如果客户端不理会服务器终止产生的错误，**继续**向已经关闭的socket里写入数据，就会触发`SIGPIPE`信号，这个信号的默认动作是终止当前进程。对于服务端需要捕获这个信号，以防止向一个已经关闭的socket写入数据，导致整个服务端进程终止。

向已经关闭的socket通道写数据：
+ 第一次写，会返回 `RST`
+ 第二次写，触发`SIGPIPE`信号

  不论该进程是捕获了该信号并从其信号处理函数处返回，还是选择简单的忽略该信号，写操作都是会返回错误：`EPIPE`。

**4. 服务器主机崩溃**   
如果客户端向已经崩溃的服务器写数据，但此时的服务器已经崩溃。客户端的TCP试图接受到服务端的 `ACK`。但是没有接收到，因为不知道是因为服务器没有接收到还是因为网络拥塞导致没有达到，都会触发重传机制。经过多次重传，`Berkeley`实现是在尝试12次共9分钟后才放弃重传，会给客户端返回一个错误：`ETIMEOUT` 。

如果是中间某个路由器判定服务器主机已经是不可达，从而响应以`destination unreachable`的ICMP消息。错误就是 **EHOSTUNREACH / ENETUNREACH**。

**5. 服务器主机崩溃后重启**    
如果服务端主机重启后接收到来自客户端的数据，因为之前的TCP信息已经全部丢失，服务端会响应一个 `RST` 报文信息。客户端会接调用 `read` 收到 `ECONNREST` 错误。

如果对于客户而言，检测服务器主机是否崩溃很重要，除了可以主动发送数据检测外，还可以采用`SO_KEEPLIVE`选项。

**6. 服务器主动关机**  
服务器主动关机时，`*unix`系统的`init`进程

+ 先给所有的进程发送 `SIGTERM` 信号（可被捕获），等待一段固定的时间
+ 然后给所有的子进程发送 `SIGKILL` 信号（不可捕获）

如果不捕获 `SIGTERM` 信号并终止，那么服务器将会由 `SIGKILL` 信号终止，子进程的所有fd都会被关闭。

### 网络编程相关的信号 
+ `SIGPIPE`：见上面分析
+ `SIGHUP`：当挂起进程的控制终端时，`SIGHUP` 信号触发。对于没有控制终端的网络后台程序而言，他们通常是利用 `SIGHUP` 信号来强制服务器重读配置文件。
+ SIGURG：LINUX环境下，内核通知应程序带外数据到达，主要有两种方式
  + IO复用计数
  + SIGURG信号

