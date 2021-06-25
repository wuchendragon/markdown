[TOC]

# Trantor

第一次写的1k多字的文档被我不小心删了，只能重写了，也是好事，正好重新整理一下（苦笑

## 项目的搭建

Trantor目前是一个半成品，这个最新的版本（0.17.6）可能是最近几天刚弄出来的

### 使用Idea插件快速搭建

* 使用new project-->Trantor
* 选择Trantor版本，确定`project key`和`module key`

**那么这个插件具体做了什么呢？**（以下把项目名简写为`xxx`）

* 首先一开始选择的`Trantor`版本就是整个项目的`dependencies`的版本

* 输入好`project key`和`module key`后，会在主模组下生成两个模组，分别为`xxx-api`和`xxx-impl`

* 在`xxx-api`目录下会生成一个`trantor.yml`里面便是业务域的具体信息

```yaml
#这些信息都是根据一开始填写的信息生成的
product:
  key: gaia
  version: 1.0.0-SNAPSHOT
  dependencies:
module:
  key: demo
  name: api
  packageName: io.terminus.trantor
  version: 1.0.0-SNAPSHPT
  description: 
```

* `xxx-api`的`pom`文件会自动生成，自动添加对`trantor-api`的依赖
* `xxx-impl`的`pom`文件会自动生成，自动添加对`trantor-sdk`,`test-framework`的依赖，以及对第一个模组`api`的依赖（比较早的版本貌似有bug，会出现循环依赖，不过这不是问题）
* `xxx-impl`的`pom`文件会自动配置maven插件

这个插件的内容仅限于此

看最新版的文档，或者看0.16版本的demo，会发现新版本推荐了新的布局，`api`-`impl`-`runtime`

那么就需要手动创建`runtime`模组了

* 新建`maven`模组，命名为`xxx-runtime`

* 修改pom文件，添加对`xxx-impl`的依赖，注意groupid为项目的groupid（我发现`io.terminus.trantor.demo`是他们的半成品project，添加对这个的依赖会无法启动项目）

* 在`src`-`java`下新建名为项目的groupid的包，比如`io.terminus.trantor`

* 之后新建一个java类，这便是启动的主类，需要进行`@SpringBootApplication`的注解

* 最后在`resouces`目录下新建`application.yml`，配置

  ```yaml
  trantor:
    mainModule: demo
  ```

  比较奇怪的是，如果选择让编译器补全，会生成一个`main-module`的配置，而不是`mainModule`，应该还是半成品的原因。

## 项目的发布

项目的发布这方面踩了很多坑，其实还是比较简单的，主要还是版本之间的差异

首先要创建一个制品项目，这个制品项目的`key`要和`project key`一致，版本也要一致

之后在项目根目录（准确来说是main module的根目录下）执行`mvn`代码

``` shell
mvn clean compile -Dtrantor.deploy=true
```

这个代码所做的工作就是生成所有的`built in function`以及`built in view(?)`

生成metadata，配置数据库等等工作

并把代码上传至控制台

* 0.17版本之后，对于是否view是否可以配置为菜单项有了新的标签 `menuView="true"`，这样(应该)就不用配置xml了

之后配置模块，在统一工作台.......

## Test

首先一开始的流程是很正常的，经过`servlet`调配，（中间经历了权限认证），最后由`DispatcherServlet`分配

> 先在此处做标记，之后再看具体的原理

给了`RPCServletMetherdHandler`，执行`invokeForRequest`方法

```java
//RPCServletMetherdHandler.class
@Override
public Object invokeForRequest(@NotNull NativeWebRequest request,
                               ModelAndViewContainer mavContainer,
                               Object @NotNull ... providedArgs) throws Exception {
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }

    // 忽略 health check 之类的请求
    if (shouldNotFilter(Objects.requireNonNull(request
                            .getNativeRequest(HttpServletRequest.class)))) {
        return super.doInvoke(args);
    } else {
        return doInvoke(args);
    }
}
```

此处的args是几个参数，都是从`request`中提取出来的

![image-20210621085643753](/Users/terminus/Library/Containers/com.apple.TextEdit/Data/Documents/Trantor.assets/image-20210621085643753.png)

* 蓝线这一项是具体的json数据，第二项是要调用的方法（但是真正的方法名是下划线后面的部分）

* 第三项是？

接下来的部分是反射调用`ActionController`，中间提到了`GlobalReqManger`暂时不知道其作用

此处的反射调用是由`RPCServletMetherdHandler`写死的

> 省略中间的CGLIB源码
>
> 暂时没懂为什么会调用到executeFlow()方法，所以在以下几个地方做了标记
>
> * `HandlerMethod.class`--->`getBridgedMethod()` ,`public HandlerMethod(<构造方法>)`
> * `Method.class`--->`Method copy() `,`invoke()496行(看看是否为空)`

`ActionController`是个普通的`@RestController`，关键点在于

```java
private final FunctionExecutor functionExecutor;
private final ModelDataService modelDataService;
private final LoadDataExecutor loadDataExecutor;
```

之后再深入看

关键是看`executeFlow()`方法

```java
@PostMapping("/flow/{flowKey}")
public Response<Object> executeFlow(
        @RequestBody List<Map<String, Object>> records,
        @PathVariable String flowKey,
        @RequestParam(required = false) String implKey,
        @RequestParam(required = false) String label
) {
    // TODO context
    Object result = functionExecutor.execute(FunctionType.LogicFlow, flowKey, implKey, label, records);
    return Response.ok(result);
}
```

可以看到，真正的调用位置是`/flow/{flowKey}`，`json`部分被解释为了`@RequestBody`，`flowKey`是`@PathVariable`，`implKey`可能已经不再需要了，最后一项是`label`。

之后是对flow的调用

>  对`FunctionInvokerComposite.class` --->`invoke()`打上断点
>
> 对`InJvmFunctionImplInvoker.class`---->`invoke()`打上断点
>
> 对`FunctionImplBean.class`---->`invoke()`打上断点



### 业务上下文



业务上下文由 `BusinessController`调度，由`BusinessContextExecutor`执行

### 调度任务

`TrantorAutoConfigration.class`中有关于其的配置

```java
@Bean
@ConditionalOnMissingBean(JobProvider.class)
public JobProvider jobProvider(JobClient jobClient, TrantorKeyProvider keyProvider) {
    return new DefaultJobProvider(jobClient, keyProvider);
}
```

在bean创建完毕后，会注入`Job.class`中（虽然不知道为什么要采用这种注入方式）

所以说`job.class`才是真正的实现类（并不是）

里面提供了三种调度方式

但是这也只是进行了定义，真正的调用应该还是隐藏在了其他地方

#### JobLoader.class

### CacheManager

由于这个玩意的底层是kotlin写的，所以暂时没搞懂

首先缓存都由`TResourceCacheManager`管理，它有三个实现类

`trantor`选择了`CaffeineCacheManager`

（首先声明我看不懂kotlin）

`init`方法对`caches`进行了声明，它是一个` MutableMap`类型

可能和go语言类似，把map解释为类似于基本类型

  `MutableMap<ResourceType, TResourceCache<*>>`

这个map的第二个参数为接口，后面的实现类为`xxxCaffeineCache`(xxx为各种type)

我不懂kotlin，idea也没法追踪kotlin内部（或者只是我不会）

我猜测所谓的缓存是动态获取的，因为这个类里面压根就没有创建缓存的选项

继续说这个map，它的第二个参数是一个类，里面有两个方法，分别是`getType()`和`fandResource()`

`fundresouce()`指向的目的地是`MetaClient.class`，这又是一个`Conntroller`

随便摘取里面的一个方法

```java

@RequestMapping("api/internal/meta")
public interface MetaClient {
  //.......
@GetMapping("/action")
Action findAction(@RequestParam String key) throws ResourceNotFoundException;}
```

那么基本可以认定，所谓的缓存是运行时获取的了，同时也抓到了它的包

申请：![image-20210621143346861](/Users/terminus/Library/Containers/com.apple.TextEdit/Data/Documents/Trantor.assets/image-20210621143346861.png)

返回：

```json
//足足几百行
JavaScript Object Notation: application/json
    Object
        Member Key: key
            String value: demo_Staff
            Key: key
        Member Key: originalKey
            String value: Staff
            Key: originalKey
        Member Key: name
            String value: 员工模型
            Key: name
          Key: typeMeta
                        Member Key: validations
                            Array
                            Key: validations
                        Member Key: systemField
                            False value
                            Key: systemField
                        Member Key: order
                            Number value: 8
                            Key: order
                   
```

但是只看到这些的话是完全不够的，因为项目本身根本就没有提供meta-data

没错，如果看端口指向的话，这里访问的是8082端口(Trantor-metastore)而不是8080端口(项目端口)

`MetaClient.java`的实现类`HttpMetaClient.java`是个空的类

···

Feign

Camel

Consul