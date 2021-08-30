# Connector 
## Websocket
那 Socket 是什么呢？网络上的两个程序通过一个双向链路进行通信，这个双向链路的一端称为一个 Socket。一个 Socket 对应一个 IP 地址和端口号，应用程序通常通过 Socket 向网络发出请求或者应答网络请求。Socket 不是协议，它其实是对 TCP/IP 协议层抽象出来的 API。\
但 WebSocket 不是一套 API，跟 HTTP 协议一样，WebSocket 也是一个应用层协议。为了跟现有的 HTTP 协议保持兼容，它通过 HTTP 协议进行一次握手，握手之后数据就直接从 TCP 层的 Socket 传输，就与 HTTP 协议无关了。浏览器发给服务端的请求会带上跟 WebSocket 有关的请求头，比如Connection: Upgrade和Upgrade: websocket。\
after websocket connection is established, WebSocket 的数据传输会以 frame 形式传输，会将一条消息分为几个 frame，按照先后顺序传输出去。这样做的好处有：大数据的传输可以分片传输，不用考虑数据大小的问题。和 HTTP 的 chunk 一样，可以边生成数据边传输，提高传输效率。\

根据 Java WebSocket 规范的规定，Java WebSocket 应用程序由一系列的 **WebSocket Endpoint** 组成。Endpoint 是一个 Java 对象，代表 WebSocket 连接的一端，就好像处理 HTTP 请求的 Servlet 一样，你可以把它看作是处理 WebSocket 消息的接口。跟 Servlet 不同的地方在于，Tomcat 会给每一个 WebSocket 连接创建一个 Endpoint 实例。你可以通过两种方式定义和实现 Endpoint：
1. 编程式的，就是编写一个 Java 类继承javax.websocket.Endpoint，并实现它的 onOpen、onClose 和 onError 方法。这些方法跟 Endpoint 的生命周期有关，Tomcat 负责管理 Endpoint 的生命周期并调用这些方法。并且当浏览器连接到一个 Endpoint 时，Tomcat 会给这个连接创建一个唯一的 Session（javax.websocket.Session）。Session 在 WebSocket 连接握手成功之后创建，并在连接关闭时销毁。当触发 Endpoint 各个生命周期事件时，Tomcat 会将当前 Session 作为参数传给 Endpoint 的回调方法，因此一个 Endpoint 实例对应一个 Session，我们通过在 Session 中添加 MessageHandler 消息处理器来接收消息，MessageHandler 中定义了 onMessage 方法。在这里 Session 的本质是对 Socket 的封装，Endpoint 通过它与浏览器通信。
2. 注解式的


### EatWhatYouKill
常规的 NIO 编程思路是，将 I/O 事件的侦测和请求的处理分别用不同的线程处理。具体过程是：启动一个线程，在一个死循环里不断地调用 select 方法，检测 Channel 的 I/O 状态，一旦 I/O 事件达到，比如数据就绪，就把该 I/O 事件以及一些数据包装成一个 Runnable，将 Runnable 放到新线程中去处理。在这个过程中按照职责划分，有两个线程在干活，一个是 I/O 事件检测线程，另一个是 I/O 事件处理线程。我们仔细思考一下这两者的关系，其实它们是生产者和消费者的关系。I/O 事件侦测线程作为生产者，负责“生产”I/O 事件，也就是负责接活儿的老板；I/O 处理线程是消费者，它“消费”并处理 I/O 事件，就是干苦力的员工。把这两个工作用不同的线程来处理，好处是它们互不干扰和阻塞对方。\
然而世事无绝对，将 I/O 事件检测和业务处理这两种工作分开的思路也有缺点。当 Selector 检测读就绪事件时，数据已经被拷贝到内核中的缓存了，同时 CPU 的缓存中也有这些数据了，我们知道 CPU 本身的缓存比内存快多了，这时当应用程序去读取这些数据时，如果用另一个线程去读，很有可能这个读线程使用另一个 CPU 核，而不是之前那个检测数据就绪的 CPU 核，这样 CPU 缓存中的数据就用不上了，并且线程切换也需要开销。\
因此 Jetty 的 Connector 做了一个大胆尝试，那就是用把 I/O 事件的生产和消费放到同一个线程来处理，如果这两个任务由同一个线程来执行，如果执行过程中线程不阻塞，操作系统会用同一个 CPU 核来执行这两个任务，这样就能利用 CPU 缓存了。\
**ProduceExecuteConsume and ExecuteProduceConsume**
![thread strategy](/images/3.1.png)

**EatWhatYouKill**：这是 Jetty 对 ExecuteProduceConsume 策略的改良，在线程池线程充足的情况下等同于 ExecuteProduceConsume；当系统比较忙线程不够时，切换成 ProduceExecuteConsume 策略。