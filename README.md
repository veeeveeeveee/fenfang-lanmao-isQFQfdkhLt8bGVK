
### 一、什么是 SPI


SPI 全名 Service Provider interface，翻译过来就是“服务提供接口”。基本效果是，申明一个接口，然后通过配置获取它的实现，进而实现动态扩展。


Java SPI 是 JDK 内置的一种动态加载扩展点的实现。


一般的业务代码中较少用到，但是在底层框架中却大量使用，包括 JDBC、Dubbo、Spring、Solon、slf4j 等框架都有用到，不同的是有的使用 Java 原生的实现，有的框架则自己实现了一套 SPI 机制.


### 二、Spring SPI


Spring 中的 SPI 相比于 JDK 原生的，它的功能更强大些，它可以替换的类型不仅仅局限于接口/抽象类，它可以是任何一个类，接口，注解；


正因为 Spring SPI 是支持替换注解类型的 SPI，这个特性在 Spring Boot 中的自动装配有体现（EnableAutoConfiguration注解）：


Spring 的 SPI 配置文件，需要放在工程的 META\-INF 下，且文件名为 spring.factories ，而文件的内容本质就是一个 properties；如 spring\-boot\-autoconfigure 包下的 META\-INF/spring.factories 文件，用于自动装配的。



```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration, \
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration, \
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration, \
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration, \
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration, \
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,

```

### 三、Solon SPI


相对于 Java SPI 和 Spring SPI 的“配置式”风格。Solon SPI 则是 “编码式” 风格。就有点儿像 Maven 和 Gradle。Solon SPI，也称为 Solon Plugin SPI。 同样需要一个配置文件，来申明 Plugin 的实现类。


约定插件配置文件，且要求文件名是唯一的：



```
#建议使用包做为文件名，便于识别，且可避免冲突
META-INF/solon/{packname}.properties

```

约定插件配置内容（就固定的两项）：



```
#插件实现类配置
solon.plugin={PluginImpl}  
#插件优化级配置。越大越优先，默认为0
solon.plugin.priority=1

```

插件代码示例（相当于，为整个 “模块” 提供了一个生命周期）。把上面 Spring SPI 的配置翻译过来就是：



```
public class SpringTranslatePlugin implements Plugin{
    @Override
    public void start(AppContext context) {
        //插件启动时...
        context.beanMake(SpringApplicationAdminJmxAutoConfiguration.class);
        context.beanMake(AopAutoConfiguration.class);
        context.beanMake(RabbitAutoConfiguration.class);
        context.beanMake(BatchAutoConfiguration.class);
        context.beanMake(CacheAutoConfiguration.class);
        context.beanMake(CassandraAutoConfiguration.class);
    }
    
    @Override
    public void prestop() throws Throwable {
        //插件预停止时（启用安全停止时：预停止后隔几秒才会进行停止）
    }
    
    @Override
    public void stop(){
        //插件停止时
    }
}

```

因为是 “编码式” 的。所以也可以做更复杂的控制处理。比如：



```
public class SolonDataPlugin implements Plugin {
    @Override
    public void start(AppContext context) {
        //注册缓存工厂
        CacheLib.cacheFactoryAdd("local", new LocalCacheFactoryImpl());

        //添加事务控制支持
        if (context.app().enableTransaction()) {
            context.beanInterceptorAdd(Tran.class, TranInterceptor.instance, 120);
        }

        //添加缓存控制支持
        if (context.app().enableCaching()) {
            CacheLib.cacheServiceAddIfAbsent("", LocalCacheService.instance);

            context.subWrapsOfType(CacheService.class, new CacheServiceWrapConsumer());

            context.lifecycle(() -> {
                if (context.hasWrap(CacheService.class) == false) {
                    context.wrapAndPut(CacheService.class, LocalCacheService.instance);
                }
            });

            context.beanInterceptorAdd(CachePut.class, new CachePutInterceptor(), 110);
            context.beanInterceptorAdd(CacheRemove.class, new CacheRemoveInterceptor(), 110);
            context.beanInterceptorAdd(Cache.class, new CacheInterceptor(), 111);
        }

        //自动构建数据源
        Props props = context.cfg().getProp("solon.dataSources");
        if (props.size() > 0) {
            context.app().onEvent(AppPluginLoadEndEvent.class, e -> {
                //支持 ENC() 加密符
                VaultUtils.guard(props);
                buildDataSource(context, props);
            });
        }
    }
}

```

 本博客参考[蓝猫机场加速器](https://dahelaoshi.com)。转载请注明出处！
