## 第三次计算机网络原理作业

姓名：郑有为  学号：19335286

**1、简述 TCP 的流控制、可靠数据传输和拥塞控制的方法原理和用途。**

* **TCP流控制**

  * **方法原理**：

    发送方维护一个叫**接收窗口**的变量，该变量告诉**发送方接收方还有多少缓存空间可用**。TCP连接两端的发送方各维护一个接收窗口，接收窗口随时间而变化，满足：

    ```c
    RcvWindow = RcvBuffer - [LastByteRcvd - LastByteRead] >= 0
    ```

    对于发送端的两个变量：`LastByteSent`，`LastByteAcked`，前者减后者得到的差为发送端发送但还未被确认的数据量，通过将该数据量控制在`RcvWindow`以内，可以保证发送端不会使接收端缓冲区溢出，即满足：

    ```c
    LastByteSent - LastByteAcked <= RecWindow
    ```

    `RecWindow = 0`时的特殊处理：当接收端接收窗口为0时，发送端继续发送一个只有一个字节数据的报文段，等到该报文段被接收方确认时，返回确认报文里包含非零的`RcvWindow`。

  * **用途**：用于消除（发送方导致的）接收方缓存溢出的可能，是一个**速度匹配服务**，匹配发送方的发送速率和接收方的接收速率。

* **TCP可靠数据传输**

  * **方法原理**：

    TCP差错检测机制：校验和

    * 校验和，将发送的数据包的二进制相加然后取反，接收方检测校验，若出现差错，丢弃这个报文段。

    **TCP差错恢复机制：往往是建立在确认应答与序列号上的GBN协议和选择重传的混合体。**

    * 回退N步（GBN）：设置一个序号跨度为N的窗口，窗口内包含待确认的分组序号和可以被立即发送的分组序号，其中32位的序号按照字节流中的字节计数。

      其中，发送方行为：1、上层想发送时，检查窗口是否已满，未满则创建分组并发送，更新窗口；2、收到ACK时，对序号为n的分组的确认采取累积确认（表明已收到该序号以前的所有分组）；3、发生超时事件时，重发所有已发送但未确认的分组，重启计时器。

      接收方行为：若分组号为n的收到了，且上一个收到的是n-1，则发送ACK，否则丢弃该分组，并为最近已收到的分组重传ACK，不缓存任何失序分组。

    * 选择重传（SR）：通过让发送方仅重传那些怀疑出错的分组，同样使用窗口来限制流水线中未完成、未被确认的分组，在SR下，发送方的窗口中已确认的可以不是连续的。

      其中，发送方行为：1、上层想发送时，检查下一个可用于分组的序号，若序号在窗口内则发送；2、超时：每个分组有自己的逻辑定时器，超时后只发送一个分组；3、收到ACK后，若接收到的序号在窗口内则进行标记，向前移动窗口基序号到最小序号未确认分组处。

      接收方行为：接收分组将被缓存直至所有比它序号小的分组都收到。当窗口内的序号的分组都能接收并缓存，一个选择ACK被发送回发送方，此时，若接受的序号等于接收窗口的基序号，则将分组及以前缓存的分组交付给上层，窗口按交付的数量向前移动。

    **除此之外，TCP还是用加倍超时间隔和快速重传来处理超时事件的发生。**

    * 加倍超时间隔：每次超时事件发生时，下一次的超时间隔会被设为当前超时间隔的两倍，而不是由EstimatedRTT和DevRTT推出，这也提供了一种有限形式的拥塞控制。

    * 快速重传：发送方可以在超时事件发生前通过注意冗余ACK来检测丢包情况。如果发送方接收到相同数据的三个冗余ACK，则就认为已被确认三次的报文之后的报文已经丢失，执行快速重传。

  * **用途**：确保一个进程从其接收缓存中读出**非损坏的、无间隔的、非冗余的、按序的数据流**；满足上层应用对传输过程中数据一致的需求。

* **TCP拥塞控制**

  * **方法原理**：如果一个TCP发送方感知从它到目的地之间的路径上无拥堵，则发送放会增加速率，否则降低速率。TCP连接两端记录变量：拥塞窗口`CongWin`，用于限制一个发送方向网络发送时流量的速率，以保证发送方中未确认的数据量不超过拥塞窗口和接收窗口。

    **三个TCP拥塞控制算法：AIMD、SS、超时反应**

    * AIMD 加性增，乘性减 ：丢包事件发生后，让`CongWin`减半，以MSS为单位，单次发送不得少于一个MSS。不丢包则缓慢增加拥塞窗口的长度（每次增加一个MSS），探测端到端路径上额外的可用宽带，增长阶段称为拥塞避免CA。

    * SS 慢启动：当一个TCP连接开始时，`CongWin`的初始值设置为1个MSS，每过一个RTT将`CongWin`翻倍，直到发生丢包事件，降到一半并从此线性增长。

    * 对超时时间作出反应：发生超时时，TCP发送方进入一个慢启动阶段，CongWin被置为1，开始成倍增加。

      TCP通过维持阈值`Threshold`来管理这些过程。

  * **用途**：**充分利用网络信道资源，提高网络传输性能**，并且在发生网络拥塞时，抑制发送端发送数据的速率，避免网络更加拥塞。

**2、从应用需求和传输层协议性能的角度，论述互联网传输层提供 TCP 和 UDP 两种不同协议的必要性。** 

 * 应用需求角度：不同应用对网络传输服务的需求不同，对应用而言，两种协议各有各的好处：

   UDP能让应用层更好地控制要发送的数据和发送时间，且无需建立连接、因而无需管理连接状态，并且分组首部开销小，仅有8字节开销。对于强调传输速度、允许丢包发生、不希望进行拥塞控制的应用，可以使用基于UDP的不可靠无连接的服务；

   TCP传输的无差错和按序性，对于强调传输准确的应用，可以使用基于TCP的可靠连接的传输服务。

* 传输层协议性能角度：UDP只提供**进程间数据交付和差错检测**两种最低限度的运输层服务。而TCP通过**流量控制、序号、确认和定时器**等技术确保正确、按序地将数据从发送进程交付给接收进程，还提供拥塞机制，防止任何一条连接过多使用流量来淹没通信主机之间的链路和交换设备。**TCP可靠性更优、UDP灵活性、传输速度更优，性能上互补**。

**3、TCP 拥塞控制中调整发送速率的方法有慢启动、加法提速和乘法降速，总结这些方法的设计原理，并举例说明其运作过程。** 

**慢启动、加法提速和乘法降速的原理：**

* 慢启动：当一个TCP连接开始时，`CongWin`（用于限制一个发送方向网络发送是流量的速率）的初始值设置为1个MSS（最大报文长度），每过一个RTT（往返时间）将`CongWin`翻倍，直到发生丢包事件，降到一半并从此线性增长。

* 加法提速和乘法降速 ：不发生丢包则缓慢增加拥塞窗口的长度（每次增加一个MSS），探测端到端路径上额外的可用宽带，丢包事件发生后，让`CongWin`减半，以MSS为单位，但单次发送不得少于一个MSS。

**举例说明运作过程：**

假设主机A与主机B建立TCP连接后，主机A准备向主机B发送数据报文。假设网络的阈值最开始为20个MSS，并在第14个RTT时变成了12MSS。主机A每一个RTT向主机B发送的数据报数量（即拥塞窗口的大小）如下图所示：

![mssrtt](C:\Users\17727\Desktop\CN\理论作业\第3次作业\mssrtt3.png)

图片中橙色为网络环境的变化，一旦超过橙线即发生丢包，蓝色为用色过程中每一个RTT的拥塞窗口大小。

数据传输的初始阶段，基于慢启动，从一个MSS开始，每次窗口大小翻倍（1，2，4，8...)，直到超过阈值，慢启动结束，转入加法提速过程，每次增加一个数据包，直到超出阈值，进入减法降速阶段，若不发生丢包则转为加法提速，否则继续进行减法降速，最后成锯齿状变化。

**4、在 TCP 通信中，如果连续收到三个对同一个段的 ACK 则推断发生了丢包事件，分析此方法的合理性。** 

首先，并不是连续收到三个对同一个段的 ACK 则推断发生了丢包事件，而是在连续收到三个冗余ACK时则推断发生了丢包。冗余ACK是指再次确认某个报文段的ACK，指明期待的下一个字节的序号。根据TCP序列号和确认号的机制，若接收端接收到的序列号不等于上一次接收的序列号与上一次数据报的长度之和，则向发送端发送包含上一次的序列号的确认报文，也即冗余ACK。		

若仅收到一个冗余ACK，有可能由以下事件导致：（1）相邻乱序：相邻两个数据报文中，后发送的先到了；（2）重复报文：由于网络原因，一个数据报文被连续接收了两次；（3）丢包。实际上，接收到一个冗余ACK时我们更愿意相信这是乱序而不是丢包，若视一次冗余ACK为丢包，很有可能导致网络复杂加重，发送效率降低。

若收到两个冗余ACK，相邻乱序和一个重复报文的事件被排除了，但依然可能有乱序和重复接收有关，如三个相邻报文的乱序或两个重复报文被接收。而随着冗余ACK次数的增加，乱序和重复接收的可能性变小了（因为我们可以假设乱序和重复接收的规模成正态分布），我们推测本次丢包的把握增加。

当收到三个冗余ACK时，我们有极大的把握认为发生了丢包，首先是乱序和重复接收的可能性更小了，其次是当我们收到三个冗余ACK时，已经经过了较长的时间，为了保证发送的效率，不应该再继续等待的那个未接收到的报文，即便它的确没有丢包，也应视其已经丢失，进而开启快速重传。

至于为什么不选择接收到更多用于ACK，考虑到我们丢包事件的判定还可以通过时钟超时来检测，当接收到很多冗余ACK时可能已经出发超时事件了，导致冗余ACK发挥不了作用。

综上考虑，将连续收到三个对同一个段的 ACK 视作发生了丢包事件是合理的，成语“事不过三”也可以反应它的合理性。

**5、对于可靠数据传输，滑动窗口协议相比停等协议有更好的效率，从两种协议的传输方法原理来对此进行分析。** 

停等协议原理：接收、发送双方各自设置一个缓冲存储空间和帧序号，以实现数据重发和接收方接收不重复。属于非流水线式的结构，一个包传输成功后，下一个包才能开始传输，引入的协议要点包括：检验和、序号、定时器、肯定确认。

滑动窗口协议：该协议使用序列号、确认号、窗口的概念，允许发送方在停止并等待确认前发送多个数据分组，并保证按序传输，发送过程属于流水线型的结构。传输的每个部分被分配唯一的连续序列号，接收方使用数字并以正确的顺序放置接收到的数据包，丢弃重复的数据包并识别丢失的数据。考虑缓冲区大小、网络条件等制约，使用窗口来控制发送方的发送速度，对发送方已发出但尚未经确认的帧的数目加以限制。同时，使用发送窗口保存尚未收到确认的数据包序号以实现丢包检测和重传。发送窗口满了，发送发需必须停止从上层接收新的数据包，直到有空闲缓存区。当序列号等于接收窗口下限的数据包到达时，把它提交给应用程序并向发送端发送确认，接收窗口向前移动一位。

可以看到：滑动窗口协议能够充分利用网络带宽，在一个RTT内，停等协议只能发送一个分组，而滑动窗口可以发送多个分组。但后者的管理上更加复杂，需增加序号范围、缓存多个分组和差错恢复机制（回退N步或选择重传），在处理时会产生时延，但总体而言，滑动窗口协议的效率更高。

**6、GBN 和 SR 两种 RDT 方法所采用的一些技术在 TCP 中均有运用，根据教学内容进行总结。**

累计确认：TCP确认是累计式的，正确接收单失序的报文段是不会被接收方逐个确认的。例如接收端返回ACK=80，则证明报文80以及之前的报文段都被正确接收。并且，TCP发送方仅需维持已发送单位被确认的最小序号和下一个要发送的字节序号。这一点采用的是GBN的实际思路。

缓存失序分组：许多TCP实现会将正确接收但失序的报文暂时储存起来。TCP允许接收方有选择地确认失序报文段，而不是累计地确认最后一个正确接收的有序的报文段。这一点采用的是SR的实际思路。

因此TCP差错机制是GBN与选择重传协议的混合体。