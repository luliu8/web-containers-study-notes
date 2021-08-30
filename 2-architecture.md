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

a Context represents a Webapp \
a Wrapper represents a Servlet \
one webapp can have multiple Servlet \
a Host represents a virtual machine \
multiple webapps can be deployed on a virtual machine \
an Engine can manage multiple virtual machines\
one Tomcat service can have only one engine.


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


