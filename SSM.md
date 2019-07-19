# Spring MVC

​	SpringMVC框架围绕着DispatcherServlet这个核心展开。

![1563465443411](D:\data\md笔记\img\springMvc.png)

​	SpringMvc处理请求的整体过程：

1、用户发出Http请求，Web服务器收到请求。如果匹配DispatcherServlet的请求映射路径。则web服务器将该请求交给DispatcherServlet处理

2、DispatcherServlet接收到请求后，根据请求信息(URL，HTTP方法，请求报文头，请求参数等)及HandlerMapping的配置查找处理请求的处理器。

3、当DispatcherServlet根据HandlerMapping找到请求的Handler时，通过HandlerAdapter对Handler进行封装。再以同一的接口进行调用。

4、处理器完成业务逻辑的请求后，返回一个ModelAndView给DispatcherServlet.

5、ModelAndView包含的是逻辑视图名而非真正的视图对象，DispatcherServlet借由ViewResolver完成逻辑视图到真实视图的解析

6、得到真实视图对象View后，DispatcherServlet使用这个View对象对ModelAndView中的模型进行视图渲染

7、返回给客户



## HttpMessageConverter<T>

​	HttpMessageConverter<T>是Spring的一个重要的接口。它负责将请求消息转为为一个对象（类型为T），将对象（类型为T）输出为响应消息

​	DispatherServlet默认已经安装了RequestMappingHanderAdapter作为HandlerAdapter的组件实现类。HttpMessageConverter默认由RequestMappingHanderAdapter使用。将请求消息转换为对象。或者将对象转换为响应消息。

```shell
# HttpMessageConverter 接口的方法
Boolean canRead(Class<?> clazz,MediaType mediaType)
Boolean canWrite(Class<?> clazz,MediaType mediaType)
List<MediaType> getSupportedMediaType()
T read(Class<? extends T> clazz,HttpInputMessage inputMessage)
void write(T t,MediaType contenType,HttpOutputMessage outputMes)
```



