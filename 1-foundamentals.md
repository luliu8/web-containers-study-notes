# Foundamentals

## What is Web Container
Servlets are mini java programs running on the server. They dont have Main method, so they cant run independency. Servlets need to be deployed inside servlet container, so the servlet container can instantiate and run them. 

Tomcat and Jetty are both implementations of Servlet container. For convenience they also function as HTTP servers, therefore they are **HTTP server + Servlet Container**, we call them **Web Containers**. 

## Servlet Specification
When HTTP server receives a HTTP request from browser, it will call a Java class on the server to handle the request. HTTP server needs to know which Java class to call for each request. \
We can use Interface-based programming to decouple HTTP server and service logic. Define Servlet Interface, all service class needs to implement Servlet Interface. Sometimes we call the service class that implements Servlet Interface as Servlet. 

![servlet interface](/images/1.1.png)

whenever we need to add new business function, we implement a Servlet, register to Servlet container. the rest will be handled by Tomcat/Jetty 

```
public interface Servlet {
    void init(ServletConfig config) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest req, ServletResponse res）throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```

ServletConfig encapsulate initialization parameters. configure them in web.xml. \
We can use Abstract class that implement Servlet interface to common logics. Servlet spec provided **GenericServlet** abstract class. and since most Servlets are for HTTP requests, Servlet spec also provided **HttpServlet** that inherits GenericServlet. when we inherit HttpServlet, we only need to override doGet and doPost. \

## Servlet Container
### Servlet Container workflow 
- HTTP server encapsulates client request as a `ServletRequest` and call Servlet Container's `service` method \
- Servlet container after receiving the request, find the corresponding Servlet from the mapping. If the Servlet hasn't been loaded, use reflection mechanism to instantiate this Servlet, call `init`, then call Servlet's `service` method to handle request. return ServletResponse to HTTP server. HTTP server send the response back to client. 

![servlet container workflow](/images/1.2.png)

### Web apps
Servlets are deployed through webapps. webapp follow directory structure similar to this: 
```
| -  MyWebApp 
	| -  WEB-INF/web.xml        -- configure servlets 
	| -  WEB-INF/lib/           -- Jars needed for the webapp 
	| -  WEB-INF/classes/      
	| -  META-INF/              
```
Servlet spec defined `ServletContext` interface. \
When Servlet container starts, it will load the webapp, and create a unique ServletContext instance for each webapp. A webapp can have multiple Servlet, they share data through the global ServletContext instance, including: initiation configurations, resources. Since ServletContext hold all Servlet instances,it can redirect servlet request from one servlet to another. 

## Extension Mechanism
### Filter 
Servlet Container instantiate the Filters and chain them together into FilterChain. when request comes in, the doFilter method of each filter are called one by one. 

### Listener 
has default listeners \
customized listeners are configured through web.xml 



