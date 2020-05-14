# <center>SkyWalking原理分析
> 以下说明基于SkyWalking v6.6.0  
> [在线演示地址](http://122.112.182.72:8080/)

## 1 类增强插件

### 1.1**关键抽象类`AbstractClassEnhancePluginDefine`**
> 该抽象类将插件增强的方法进行了统一管理，结合接口`ConstructorInterceptPoint`、`InstanceMethodsInterceptPoint`、`StaticMethodsInterceptPoint`使得自定义开发组件增强类变得简单。

- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#define` 核心方法（插件增强入口）

- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#enhance`该抽象方法只有唯一实现`org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhance`，该实现会调用两个方法
  - **增强静态方法**`org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhanceClass`
    - 找对应插件实现类静态方法的拦截器数组`StaticMethodsInterceptPoint`，并通过`byte-buddy`工具包实现字节码增强
  - **增强实例方法和构造器方法**`org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhanceInstance`
    - 找到对应插件的构造器`ConstructorInterceptPoint`,并通过`byte-buddy`工具包实现字节码增强
    - 找到对应插件的实例方法`instanceMethodsInterceptPoints`,并通过`byte-buddy`工具包实现字节码增强
- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#enhanceClass` 进行类匹配，参考SkyWalking提供的默认处理`ClassMatch`
- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#witnessClasses` 为了处理一个插件有多个版本的问题
- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#isBootstrapInstrumentation`
- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#getConstructorsInterceptPoints` 获取构造器增强插件数组
- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#getInstanceMethodsInterceptPoints` 获取实例方法增强插件数组
- `org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#getStaticMethodsInterceptPoints` 获取静态方法增强插件数组

### 1.2 [自定义增强插件](https://github.com/SkyAPM/document-cn-translation-of-skywalking/blob/master/docs/zh/master/setup/service-agent/java-agent/Customize-enhance-trace.md)

#### 1.2.1 实现对类的自定义增强需要2步。

> 1. 激活插件，将插件从`optional-plugins/apm-customize-enhance-plugin.jar`移动到`plugin/apm-customize-enhance-plugin.jar`。
> 2. 在agent.config中配置`plugin.customize.enhance_file`，指明增强规则文件，比如`/absolute/path/to/customize_enhance.xml`。
> 3. 在`customize_enhance.xml`中配置增强规则。
> 	```xml
> 	<?xml version="1.0" encoding="UTF-8"?>
> 	<enhanced>
> 	    <class class_name="test.apache.skywalking.testcase.customize.service.TestService1">
> 	        <method method="staticMethod()" operation_name="/is_static_method" static="true"/>
> 	        <method method="staticMethod(java.lang.String,int.class,java.util.Map,java.util.List,[Ljava.lang.Object;)" operation_name="/is_static_method_args" static="true">
> 	            <operation_name_suffix>arg[0]</operation_name_suffix>
> 	            <operation_name_suffix>arg[1]</operation_name_suffix>
> 	            <operation_name_suffix>arg[3].[0]</operation_name_suffix>
> 	            <tag key="tag_1">arg[2].['k1']</tag>
> 	            <tag key="tag_2">arg[4].[1]</tag>
> 	            <log key="log_1">arg[4].[2]</log>
> 	        </method>
> 	        <method method="method()" static="false"/>
> 	        <method method="method(java.lang.String,int.class)" operation_name="/method_2" static="false">
> 	            <operation_name_suffix>arg[0]</operation_name_suffix>
> 	            <tag key="tag_1">arg[0]</tag>
> 	            <log key="log_1">arg[1]</log>
> 	        </method>
> 	        <method method="method(test.apache.skywalking.testcase.customize.model.Model0,java.lang.String,int.class)" operation_name="/method_3" static="false">
> 	            <operation_name_suffix>arg[0].id</operation_name_suffix>
> 	            <operation_name_suffix>arg[0].model1.name</operation_name_suffix>
> 	            <operation_name_suffix>arg[0].model1.getId()</operation_name_suffix>
> 	            <tag key="tag_os">arg[0].os.[1]</tag>
> 	            <log key="log_map">arg[0].getM().['k1']</log>
> 	        </method>
> 	    </class>
> 	    <class class_name="test.apache.skywalking.testcase.customize.service.TestService2">
> 	        <method method="staticMethod(java.lang.String,int.class)" operation_name="/is_2_static_method" static="true">
> 	            <tag key="tag_2_1">arg[0]</tag>
> 	            <log key="log_1_1">arg[1]</log>
> 	        </method>
> 	        <method method="method([Ljava.lang.Object;)" operation_name="/method_4" static="false">
> 	            <tag key="tag_4_1">arg[0].[0]</tag>
> 	        </method>
> 	        <method method="method(java.util.List,int.class)" operation_name="/method_5" static="false">
> 	            <tag key="tag_5_1">arg[0].[0]</tag>
> 	            <log key="log_5_1">arg[1]</log>
> 	        </method>
> 	    </class>
> 	</enhanced>
> 	```
> 
> - 文件中的配置说明。
> 
> 	| 配置  | 说明 |
> 	|:----------------- |:---------------|
> 	| class_name | 要被增强的类 |
> 	| method | 类的拦截器方法 |
> 	| operation_name | 如果进行了配置，将用它替代默认的operation_name |
> 	| operation_name\_suffix | 表示在operation_name后添加动态数据 |
> 	| static | 方法是否为静态方法 |
> 	| tag | 将在local span中添加一个tag。key的值需要在XML节点上表示。|
> 	| log | 将在local span中添加一个log。key的值需要在XML节点上表示。|
> 	| arg[x]   | 表示输入的参数值。比如args[0]表示第一个参数。 |
> 	| .[x]     | 当正在被解析的对象是Array或List，你可以用这个表达式得到对应index上的对象。|
> 	| .['key'] | 当正在被解析的对象是Map, 你可以用这个表达式得到map的key。|

#### 1.2.2 源码实现
##### 1.2.2.1自定义插件的加载方式
- 入口`org.apache.skywalking.apm.agent.core.plugin.PluginBootstrap#loadPlugins`
- 动态加载插件`org.apache.skywalking.apm.agent.core.plugin.DynamicPluginLoader#load`
    - 通过`ServiceLoader`的方式加载`org.apache.skywalking.apm.agent.core.plugin.loader.InstrumentationLoader`的实现类即`org.apache.skywalking.apm.plugin.customize.loader.CustomizeInstrumentationLoader`将通过`customize_enhance.xml`配置的自定义增强类、方法通过Class.forNama实例化成插件对象
    ```java
    for (String enhanceClass : enhanceClasses) {
        String[] classDesc = CustomizeUtil.getClassDesc(enhanceClass);
        AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(
            Boolean.valueOf(classDesc[1]) ? CustomizeStaticInstrumentation.class.getName() : CustomizeInstanceInstrumentation.class.getName(),
            true, classLoader).getConstructor(String.class).newInstance(classDesc[0]);
        instrumentations.add(plugin);
    }
    ```
     
##### 1.2.2.2自定义插件类增强方式
- 自定义实例方法增强`org.apache.skywalking.apm.plugin.customize.define.CustomizeInstanceInstrumentation`
    - 拦截器类`org.apache.skywalking.apm.plugin.customize.interceptor.CustomizeStaticInterceptor` 
- 自定义静态方法增强`org.apache.skywalking.apm.plugin.customize.interceptor.CustomizeInstanceInterceptor`
    - 拦截器类`org.apache.skywalking.apm.plugin.customize.define.CustomizeStaticInstrumentation` 
- 自定义拦截器基类`org.apache.skywalking.apm.plugin.customize.interceptor.BaseInterceptorMethods`提供默认创建**LocalSpan**、**Tag**、**Log**


## 2 Collector 观测分析平台(Observability Analysis Platform, OAP)

### 2.1 OAP的启动及初始化流程
> 启动入口：`org.apache.skywalking.oap.server.starter.OAPServerBootstrap#start`
```java
ApplicationConfigLoader configLoader = new ApplicationConfigLoader();
ModuleManager manager = new ModuleManager();
try {
    // 1.读取并加载application.yml中的配置信息
    // 2.获取到的当前系统环境进行配置覆盖（主要是本地JDK信息，电脑信息等）
    ApplicationConfiguration applicationConfiguration = configLoader.load();
    // 1.通过JDK的ServiceLoader机制（SPI）获取`META-INF.services`下的内容，见2.1.1 补充信息
    // 2.通过名称equals后进行newInstance实例化对象，见2.1.2详细说明
    manager.init(applicationConfiguration);
} catch (Throwable t) {
    logger.error(t.getMessage(), t);
    System.exit(1);
}
```

#### 2.1.1 补充信息(包含但不限于以下基本实现，还有自定义拓展组件)
>- `org.apache.skywalking.oap.server.library.module.ModuleDefine`  
>> org.apache.skywalking.oap.server.core.storage.StorageModule
>> org.apache.skywalking.oap.server.core.cluster.ClusterModule  
>> org.apache.skywalking.oap.server.core.CoreModule
>> org.apache.skywalking.oap.server.core.query.QueryModule
>> org.apache.skywalking.oap.server.core.alarm.AlarmModule
>> org.apache.skywalking.oap.server.core.exporter.ExporterModule  
>
>- `org.apache.skywalking.oap.server.library.module.ModuleProvider`  
>> org.apache.skywalking.oap.server.core.CoreModuleProvider


#### 2.1.2 组件管理`ModuleManager`

`org.apache.skywalking.oap.server.library.module.ModuleManager#init`初始化指定组件
```java
String[] moduleNames = applicationConfiguration.moduleList();
// 通过ServiceLoader在META-INF/services/org.apache.skywalking.oap.server.library.module.ModuleDefine文件中加载外置的ModuleDefine组件
// 大多数的实现类均继承抽象类org.apache.skywalking.oap.server.library.module.ModuleDefine
ServiceLoader<ModuleDefine> moduleServiceLoader = ServiceLoader.load(ModuleDefine.class);
// 通过ServiceLoader在META-INF/services/org.apache.skywalking.oap.server.library.module.ModuleProvider文件中加载外置的ModuleProvider组件
// 大多数的实现类均继承抽象类org.apache.skywalking.oap.server.library.module.ModuleProvider
ServiceLoader<ModuleProvider> moduleProviderLoader = ServiceLoader.load(ModuleProvider.class);

LinkedList<String> moduleList = new LinkedList<>(Arrays.asList(moduleNames));
for (ModuleDefine module : moduleServiceLoader) {
    for (String moduleName : moduleNames) {
        if (moduleName.equals(module.name())) {
            ModuleDefine newInstance;
            try {
                newInstance = module.getClass().newInstance();
            } catch (InstantiationException | IllegalAccessException e) {
                throw new ModuleNotFoundException(e);
            }
            // 准备阶段,将ModuleDefine与ModuleProvider关联并初始化，其中包含着ModuleProvider的prepare阶段
            newInstance.prepare(this, applicationConfiguration.getModuleConfiguration(moduleName), moduleProviderLoader);
            loadedModules.put(moduleName, newInstance);
            moduleList.remove(moduleName);
        }
    }
}
// Finish prepare stage
isInPrepareStage = false;

if (moduleList.size() > 0) {
    throw new ModuleNotFoundException(moduleList.toString() + " missing.");
}

// 实例化即完成数据的初始化以及排序
BootstrapFlow bootstrapFlow = new BootstrapFlow(loadedModules);

// 开始正式启动各个组件
bootstrapFlow.start(this);
// 初始化完成后的回调操作
bootstrapFlow.notifyAfterCompleted();
```        


 
## 3 本地代码调试踩坑

#### 3.1.1 OAP初始化启动无问题，但停止ES后导致异常中断再次连接的时候UI显示异常，控制台报错
> - [SearchPhaseExecutionException[Failed to execute phase [query], all shards >failed]](http://elasticsearch-users.115913.n3.nabble.com/Oops-SearchPhaseExecutionExc>eption-Failed-to-execute-phase-query-all-shards-failed-td4068795.html)
> - [elasticsearch出现SearchPhaseExecutionException的解决方案](http://hanc.cc/index.php/archives/149/)

#### 3.1.2 自定义开发插件由于Linsence问题无法打Jar包
```shell
mvn clean install -DskipTests -Drat.skip=true
```
    


### 友情链接
- [SkyWalking 文档中文版（社区提供）](https://github.com/SkyAPM/document-cn-translation-of-skywalking)
- [opentracing文档中文版](https://wu-sheng.gitbooks.io/opentracing-io/content/)
- [Skywalking 6.X源码构建及运行](https://www.bilibili.com/video/av35990012/)
- [插件开发指南中文版](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/master/guides/Java-Plugin-Development-Guide.html)
- [SkyWalking 源码分析 —— Agent 收集 Trace 数据](http://www.iocoder.cn/SkyWalking/agent-collect-trace/)
- [SkyWalking 源码分析 —— Collector 初始化](http://www.iocoder.cn/SkyWalking/collector-init/)
- [SkyWalking 源码分析 —— traceId集成到日志组件](http://www.iocoder.cn/SkyWalking/trace-id-integrate-into-logs/)
- [由于license问题其他的插件（Oracle）拓展仓库](https://github.com/SkyAPM/java-plugin-extensions)
