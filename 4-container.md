# Servlet Container

Jetty achieves chained request handling through HandlerWrapper. Each HandlerWrapper calls the next HandlerWrapper \

for example: WebAppContext -> SessionHandler -> SecurityHandler -> ServletHandler \
Also, Jetty has backtrack chaining: 从头到尾依次链式调用 Handler 的方法 A，完成后再回到头节点，再进行一次链式调用，只不过这一次调用另一个方法 B。\
for example, call init for all handlers , then returns to head and call other methods one by one. \
achieved through ScopedHandler. \
![handler inheritance map](/images/2.16.png)
Handlers related to Servlet Specs all inherited ScopedHandler. 

通过设置_outScope和_nextScope的值，并且在代码中判断这些值并采取相应的动作，目的就是让 ScopedHandler 链上的 doScope 方法在 doHandle、handle 方法之前执行。并且不同 ScopedHandler 的 doScope 都是按照它在链上的先后顺序执行的，doHandle 和 handle 方法也是如此。这样 ScopedHandler 帮我们把调用框架搭好了，它的子类只需要实现 doScope 和 doHandle 方法。比如在 doScope 方法里做一些初始化工作，在 doHanlde 方法处理请求。
