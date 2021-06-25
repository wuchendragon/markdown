



# **Camel**

An **Exchange** is the message container holding the information during the entire routing of a Message received by a Consumer.



A **consumer** of message exchanges from an Endpoint

```java
//需要重点看
HttpServerMultiplexChannelHandler
  
```

现在看到的部分是



route与processor的交互位于靠后的过程，consumer和exchange以及endpoint在前面的过程

将`Processor`注册为`routes`的过程

* DefaultCamelContext.java

doSrartCamel()——>



* ProcesserDefinition.java

addRoutes()——>



* DefaultRouteContext.java

commit()

```java
Processor target = Pipeline.newInstance(getCamelContext(), eventDrivenProcessors);

// force creating the route id so its known ahead of the route is started
String routeId = route.idOrCreate(getCamelContext().getNodeIdFactory());

// and wrap it in a unit of work so the UoW is on the top, so the entire route will be in the same UoW
CamelInternalProcessor internal = new CamelInternalProcessor(target);
```

在这里新建了所有的 `CamelInternalProcessor`，我们所有的`processor`都在这里注册了



camel有内置的neety4组件，可以直接接受http请求，之后直接把请求交给了需要的`CamelInternalProcessor`

这个过程还需要再仔细看

