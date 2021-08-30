# Architecture
## Tomcat overall architecture 
Tomcat has 2 core functions, with 2 core components to handle each
- Connector: handle Socket connection, conversion between bytestream and Request/Response
- Servlet Container: load and manage Servlet, handle Request

Tomcat supports multiple I/O model and application protocols, one container needs multiple connectors (similar to a room with many doors)

![architecture](/images/2.1.png)

Each tomcat instance can have multiple services, each service can have multiple connectors and one servlet container. Connector and Servlet Container communicate through `ServletRequest` and `ServletResponse`. 


## Design of Connector

Connector hides the protocol and I/O model detail from Servlet Container. regardless of the I/O model and protocol used, Servlet Container receives the standard ServletRequest instance. \

functions of Connector:
- listen to network ports
- accept network connection request 
- read network request bytestream 
- parse the bytestream according to the application protocol, generate standard Tomcat Request instance
- convert Tomcat Request instance to standard ServletRequest
- call Servlet Container to get ServletResponse
- convert ServletResponse to Tomcat Response instance
- convert Tomcat Response instance to network bytesteam 
- send response bytestream back to browser 

design 3 submodules so the functions are high cohesion but low coupling 
1. `Endpoint`: for network communication 
2. `Processor`: for application layer parsing 
3. `Adapter`: for conversion between Tomcat Request/Response and ServletRequest/ServletResponse

the submodules interact through interfaces.
Endpoint provides bytestream to Processor \
Processor provides Tomcat Request to Adapter \
Adapter provides ServletRequest to ServletContainer 

![architecture](/images/2.2.png)

since there are many variations of i/o model and application protocol, Endpoint and Processor are encapusulated into ProtocolHandler for better usability. 

![protocolhandler](/images/2.3.png)

so tomcat can use Http11NioProtocol directly for example. 



## Design of Servlet Container 
Tomcat has 4 types of containers: Engine, Host, Context, Wrapper. \
relationships are inheritance. \
layered architecture to provide flexibility 

![layered architecture](/images/2.4.png)

one Tomcat service can have only one engine, an Engine can manage multiple hosts (virtal machines). \
a Host represents a virtual machine, multiple webapps (contexts) can be deployed on a virtual machine \
a Context represents a Webapp, a webapp can have multiple Servlets\
a Wrapper represents a Servlet 


server.xml 
![server.xml](/images/2.5.png)

composite design pattern 
all implement Container interface 
```
public interface Container extends Lifecycle {
    public void setName(String name);
    public Container getParent();
    public void setParent(Container container);
    public void addChild(Container child);
    public void removeChild(Container child);
    public Container findChild(String name);
}
```

routing request to Servlet: 
![routing](/images/2.6.png)

 - 由 Mapper 组件完成, Mapper 组件保存了容器组件与访问路径的映射关系, 根据请求的 URL 进行定位
- 端口号定位出 Service 和 Engine
- 域名定位出 Host
- URL 路径定位出 Context Web 应用
- URL 路径定位出 Wrapper ( Servlet )
- 在各个层次定位过程中, 都会对请求做一些处理

pipeline and valve: 

![pipeline and valve](/images/2.7.png)

- 通过 Pipeline-Valve 实现容器间的互相调用 ( 责任链模式 )
- valve 表示一个处理点 ( 如权限认证, 日志等), 处理请求; valve 通过链表串联, 并由 pipeline 维护
- valve 会通过 getNext().invoke() 调用下一个 valve, 最后一个 valve ( Basic ) 调用下一层容器的 pipeline 的第一个 valve
- Adapter 调用 Engine pipeline 的第一个 valve
- Wrapper 最后一个 valve 会创建一个 Filter 链, 并最终调用 Servlet 的 service 方法
- valve 与 filter 对比
- valve 是 Tomcat 的私有机制, Filter 是 Servlet API 公有标准
- valve 工作在容器级别, 拦截所有应用; Servlet Filter 工作在应用级别, 只能拦截某个应用的请求

## Lifecycle management 
diagram showing Request flow through Tomcat:
 ![request flow diagram](/images/2.8.png)

notations used:
- composition: \
A special type of aggregation where parts are destroyed when the whole is destroyed.\
Objects of Class2 live and die with Class1.\
Class2 cannot stand by itself.\

- dependency \
An object of one class might use an object of another class in the code of a method. If the object is not stored in any field, then this is modeled as a dependency relationship. (e.g. Executor uses Processor)


Tomcat needs to dynamically manage the lifecyle of all these components 

第一层关系是组件有大有小，大组件管理小组件，比如 Server 管理 Service，Service 又管理连接器和容器。\
第二层关系是组件有外有内，外层组件控制内层组件，比如连接器是外层组件，负责对外交流，外层组件调用内层组件完成业务功能。也就是说，请求的处理过程是由外层组件来驱动的。\

这两层关系决定了系统在创建组件时应该遵循一定的顺序。\
第一个原则是先创建子组件，再创建父组件，子组件需要被“注入”到父组件中。\
第二个原则是先创建内层组件，再创建外层组件，内层组件需要被“注入”到外层组件。\

abstract out LifeCycle interface
 ![lifecycle interface](/images/2.9.png)
在父组件的 init 方法里需要创建子组件并调用子组件的 init 方法。同样，在父组件的 start 方法里也需要调用子组件的 start 方法，因此调用者可以无差别的调用各组件的 init 方法和 start 方法，这就是组合模式的使用，并且只要调用最顶层组件，也就是 Server 组件的 init 和 start 方法，整个 Tomcat 就被启动起来了。
** composite design pattern **
### lifecycle events (for extensibility)
add 2 methods in lifecyle interface: addLifecycleListener() and removeLifecycleListener()\
define Enum to represent the various states 
 ![lifecycle interface 2](/images/2.10.png)


### LifeCycleBase abstract class (for code reusability)
Tomcat 定义一个基类 LifecycleBase 来实现 Lifecycle 接口，把一些公共的逻辑放到基类中去，比如生命状态的转变与维护、生命事件的触发以及监听器的添加和删除等，而子类就负责实现自己的初始化、启动和停止等方法 \
 ![lifecycle interface 3](/images/2.11.png)
 **template design pattern** 

### class diagram of Tomcat components
![class diagram of Tomcat components](/images/2.12.png)

 图中的 StandardServer、StandardService 等是 Server 和 Service 组件的具体实现类，它们都继承了 LifecycleBase。StandardEngine、StandardHost、StandardContext 和 StandardWrapper 是相应容器组件的具体实现类，因为它们都是容器，所以继承了 ContainerBase 抽象基类，而 ContainerBase 实现了 Container 接口，也继承了 LifecycleBase 类，它们的生命周期管理接口和功能接口是分开的，这也符合设计中**接口分离Interface Segregation Principle**。


## Management Components of Tomcat 
![flow diagram of starting tomcat](/images/2.13.png)

## Jetty overall architecture 
Jetty = HTTP server + Servlet Container \
composed of multiple Connectors (HTTP server), multiple Handlers (Servlet Container) and a ThreadPool(global thread resource)
![jetty architecture](/images/2.14.png)

use Server Class to start and coordinate work between components: instantiate and initiate Connector, Handler, ThreadPool, call start \

**Tomcat vs Jetty**\
第一个区别是 Jetty 中没有 Service 的概念，Tomcat 中的 Service 包装了多个连接器和一个容器组件，一个 Tomcat 实例可以配置多个 Service，不同的 Service 通过不同的连接器监听不同的端口；而 Jetty 中 Connector 是被所有 Handler 共享的 \
第二个区别是，在 Tomcat 中每个连接器都有自己的线程池，而在 Jetty 中所有的 Connector 共享一个全局的线程池

## Jetty's Connector 
跟 Tomcat 一样，Connector 的主要功能是对 I/O 模型和应用层协议的封装。I/O 模型方面，最新的 Jetty 9 版本只支持 NIO \
至于应用层协议方面，跟 Tomcat 的 Processor 一样，Jetty 抽象出了 Connection 组件来封装应用层协议的差异 \

### Java [NIO](http://tutorials.jenkov.com/java-nio/index.html)
Java NIO 的核心组件是 Channel、Buffer 和 Selector。Channel 表示一个连接，可以理解为一个 Socket,通过它可以读取和写入数据，但是并不能直接操作数据，需要通过 Buffer 来中转。Selector 可以用来检测 Channel 上的 I/O 事件，比如读就绪、写就绪、连接就绪，一个 Selector 可以同时处理多个 Channel，因此单个线程可以监听多个 Channel，这样会大量减少线程上下文切换的开销\
I/O 通信上主要完成了三件事情：监听连接、I/O 事件查询以及数据读写。因此 Jetty 设计了 Acceptor、SelectorManager 和 Connection 来分别做这三件事情\
![connector workflow](/images/2.15.png)

- Accepter listens to connection request. when connection request comes in, create a Channel for this connection and hand it over to ManagedrSelector 
- ManagedSelector register this Channel to Selection, create an EndPoint and a Connection to bind with this Channel. then starts to detect I/O event. 
- when I/O event is detected, call a method in EndPoint to obtain a Runnable, give to ThreadPool for execution. 
- ThreadPool execute Runnable with a thread
- Runnable call the callback function (registered to EndPoint by Connection) to read data
- Connection parse the read data, generate Request instance for the Handler component to handle. 


## Jetty's Handler 
Jetty is highly customizable through Hanlder\
### Handler is an interface
```
public interface Handler extends LifeCycle, Destroyable
{
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response)
        throws IOException, ServletException;
    
    //Each Handler is associated with a Server, which manages the Handler
    public void setServer(Server server);
    public Server getServer();
    // Handler will load some resource to cache, so need to destroy them 
    public void destroy();
}
```
handle method has 2 parameters ServletRequest and ServletResponse

### Simplified Handler inheritance map
![handler inheritance map](/images/2.16.png)

Handler interface has AbstractHandler (interface and abstract class often goes hand in hand)

### HandlerWrapper vs HandlerCollection
HandlerWrapper 和 HandlerCollection 都是 Handler，但是这些 Handler 里还包括其他 Handler 的引用。不同的是，HandlerWrapper 只包含一个其他 Handler 的引用，而 HandlerCollection 中有一个 Handler 数组的引用。
![handler containers](/images/2.17.png)

**Server**\
Server 比较好理解，它本身是 Handler 模块的入口，必然要将请求传递给其他 Handler 来处理，为了触发其他 Handler 的调用，所以它是一个 HandlerWrapper。\

**ScopedHandler**\
它也是一个比较重要的 Handler，实现了“具有上下文信息”的责任链调用。为什么我要强调“具有上下文信息”呢？那是因为 Servlet 规范规定 Servlet 在执行过程中是有上下文的。那么这些 Handler 在执行过程中如何访问这个上下文呢？这个上下文又存在什么地方呢？答案就是通过 ScopedHandler 来实现的。\
而 ScopedHandler 有一堆的子类，这些子类就是用来实现 Servlet 规范的，比如 ServletHandler、ContextHandler、SessionHandler、ServletContextHandler 和 WebAppContext

**why do we need HandlerCollection?** \
因为 Jetty 可能需要同时支持多个 Web 应用，如果每个 Web 应用有一个 Handler 入口，那么多个 Web 应用的 Handler 就成了一个数组，比如 Server 中就有一个 HandlerCollection，Server 会根据用户请求的 URL 从数组中选取相应的 Handler 来处理，就是选择特定的 Web 应用来处理请求。

### 3 types of Handlers
1. Coordination Handler: route request to a handler. e.g. HandlerCollection has an array of handlers, route request to one of the Handlers in the array. 

2. Filter Handler: handle reqeust, then forward to next handler. all handlers that inherited HandlerWrapp has filtering function. e.g. ContextHandler, SessionHandler, WebAppContext

3. Content Handler: call Servlet to handle request, e.g. ServletHandler, ResourceHandler

### how does Jetty starts a web app
```
//新建一个WebAppContext，WebAppContext是一个Handler
WebAppContext webapp = new WebAppContext();
webapp.setContextPath("/mywebapp");
webapp.setWar("mywebapp.war");

//将Handler添加到Server中去
server.setHandler(webapp);

//启动Server
server.start();
server.join();
```
WebAppContext 对应一个 Web 应用
![handler chain in Jetty](/images/2.18.png)

Jetty 的 Handler 组件和 Tomcat 中的容器组件是大致是对等的概念，Jetty 中的 WebAppContext 相当于 Tomcat 的 Context 组件，都是对应一个 Web 应用；而 Jetty 中的 ServletHandler 对应 Tomcat 中的 Wrapper 组件，它负责初始化和调用 Servlet，并实现了 Filter 的功能。

## key ideas from the design of Tomcat and Jetty 
### how to achieve component-based design  
- 第一个是面向接口编程。我们需要对系统的功能按照“高内聚、低耦合”的原则进行拆分，每个组件都有相应的接口，组件之间通过接口通信，这样就可以方便地替换组件了。比如我们可以选择不同连接器类型，只要这些连接器组件实现同一个接口就行。
- 第二个是 Web 容器提供一个载体把组件组装在一起工作。组件的工作无非就是处理请求，因此容器通过责任链模式把请求依次交给组件去处理。对于用户来说，我只需要告诉 Web 容器由哪些组件来处理请求。把组件组织起来需要一个“管理者”，这就是为什么 Tomcat 和 Jetty 都有一个 Server 的概念，Server 就是组件的载体，Server 里包含了连接器组件和容器组件；容器还需要把请求交给各个子容器组件去处理，Tomcat 和 Jetty 都是责任链模式来实现的

### instantiation of components
通过反射机制来动态地创建。具体来说，Web 容器不是通过 new 方法来实例化组件对象的, 而是通过 Class.forName 来创建组件. Web 容器设计了自己类加载器

### lifecycle management of components
不同类型的组件具有父子层次关系，父组件处理请求后再把请求传递给某个子组件\
Jetty 中的 Handler 也是分层次的\
而 Tomcat 通过容器的概念，把小容器放到大容器来实现父子关系\

### use skeletal abstract class and tempate design pattern 
比如说 Tomcat 中 ProtocolHandler 接口，ProtocolHandler 有抽象基类 AbstractProtocol，它实现了协议处理层的骨架和通用逻辑，而具体协议也有抽象基类，比如 HttpProtocol 和 AjpProtocol。对于 Jetty 来说，Handler 接口之下有 AbstractHandler，Connector 接口之下有 AbstractConnector，这些抽象骨架类实现了一些通用逻辑，并且会定义一些抽象方法，这些抽象方法由子类实现，抽象骨架类调用抽象方法来实现骨架逻辑。


