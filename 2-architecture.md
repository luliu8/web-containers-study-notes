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