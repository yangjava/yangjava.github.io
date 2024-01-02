---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Guice
IOC（Inversion of Control）控制反转，是一个实现对象解耦的思想。

依赖注入DI（Dependency Injection），组件之间依赖关系由容器在运行期决定，即由容器动态的将某个依赖关系注入到组件之中。

IoC 和 DI 虽然定义不同，但它们所做的事情都是一样的，都是用来实现对象解耦的，而二者又有所不同：IoC 是一种设计思想，而 DI 是一种具体的实现技术。

IOC的实现之一是Spring，另一个则是Google Guice。一些流行框架/库使用了Guice作为基础DI库，如：Druid、Elastic Searc以及携程开源的Apollo和Netflix开源的Eureka。

## Guice
Google Guice作为一个纯粹的DI框架，主要用于减轻你对工厂的需求以及Java代码中对new的使用。通过它来构建你的代码，能减少依赖，使得更容易更改、更容易单元测试和重用。看下Guice官方给的例子：
```
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(CreditCardProcessor.class).to(PaypalCreditCardProcessor.class);
    bind(BillingService.class).to(RealBillingService.class);
  }
}
```
BillingModule模块，扩展AbstractModule这个抽象类。通过实现configure接口通过bind("interface").to("implement") 来使接口和实现绑定。BillingModule有三个实例：交易日志、支付过程和账单服务。Moudle是实现同一功能接口的集合，并在自定义的Moudle configure方法内实现接口和实现类的绑定。

## Moudle
Module之间的关系：

并列：默认顺序传递就是此关系
```
Guice.createInjector(new MainModule(), .......);
```
嵌套：大的Module可以嵌套任意多个子Module
```
public class ServerModule extends AbstractModule {
    @Override
    protected void configure() {
        install(new MainModule());
    }
}
```
覆盖：如果有冲突的话后者覆盖前者，没有的话就都生效
```
// 用后者覆盖前者
Module finalModule = Modules.override(new MainModule()).with(new ServerModule())
```

## 绑定
类名绑定：把实现类绑定到接口（当然也可以实现类绑到实现类），具体实例交给框架去帮你创建bind(BillingService.class).to(RealBillingService.class);
实例绑定：绑定一个现有实例。bind(Animal.class).toInstance(new Dog())采用这种绑定，依赖注入时永远是单例（即new Dog()这个实例）
注解绑定：在模块的绑定时用annotatedWith方法指定具体的注解来进行绑定
```
BindingAnnotation
@Target({ FIELD, PARAMETER, METHOD })
@Retention(RUNTIME)
public@interface PayPal {}

publicclass RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@PayPal CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
}

// 当注入的方法参数存在@PayPal注解时注入PayPalCreditCardProcessor实现
bind(CreditCardProcessor.class).annotatedWith(PayPal.class).to(PayPalCreditCardProcessor.class);
```
这种方式有一个问题就是我们必须增加自定义的注解来绑定，基于此Guice内置了一个@Named注解满足该场景：
```
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@Named("Checkout") CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
}

// 当注入的方法参数存在@Named注解且值为Checkout时注入CheckoutCreditCardProcessor实现
bind(CreditCardProcessor.class).annotatedWith(Names.named("Checkout")).to(CheckoutCreditCardProcessor.class);
```
Provider绑定:运行时注入，Provider绑定（一个实现了Provider接口的实现类）来实现
```
public interface Provider<T> {
  T get();
}

public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  private final Connection connection;

  @Inject
  public DatabaseTransactionLogProvider(Connection connection) {
    this.connection = connection;
  }

  public TransactionLog get() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setConnection(connection);
    return transactionLog;
  }
}

public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    //Provider绑定
    bind(TransactionLog.class).toProvider(DatabaseTransactionLogProvider.class);
  }
}
```
泛型绑定：支持泛型类型的绑定。借助TypeLiteral来完成
```
bind(new TypeLiteral<RequestValidators<PutMappingRequest>>() {}).toInstance(mappingRequestValidators);
```
集合绑定：可在不同的Module内向同一个集合分别去绑定自己所要支持的内容，当然也可以在同一个Module内
```
Multibinder<Animal> multibinder = Multibinder.newSetBinder(binder(), Animal.class);
multibinder.addBinding().toInstance(new Dog());
multibinder.addBinding().toInstance(new Cat());
```
除此之外应该还有个构造器绑定，ES源码没有使用，这里不讨论。

## Scope机制
默认情况下，Guice每次都会返回一个新的实例，这个可以通过范围（Scope）来配置。常见的范围有单例（@Singleton）、会话（@SessionScoped）和请求（@RequestScoped），另外还可以通过自定义的范围来扩展。

范围的注解可以应该在实现类、@Provides方法中，或在绑定的时候指定（优先级最高）：
```
@Singleton
public class InMemoryTransactionLog implements TransactionLog {
  /* everything here should be threadsafe! */
}

// scopes apply to the binding source, not the binding target
bind(TransactionLog.class).to(InMemoryTransactionLog.class).in(Singleton.class);

@Provides @Singleton
TransactionLog provideTransactionLog() {
    ...
}
```
另外，Guice还有一种特殊的单例模式叫饥饿单例（相对于懒加载单例来说）：
```
// Eager singletons reveal initialization problems sooner,
// and ensure end-users get a consistent, snappy experience.
bind(TransactionLog.class).to(InMemoryTransactionLog.class).asEagerSingleton();
```

## 注入
依赖注入的要求就是将行为和依赖分离，它建议将依赖注入而非通过工厂类的方法去查找。注入的方式通常有构造器注入、方法注入、属性注入等。

构造器注入
```
// 构造器注入
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processorProvider;
  private final TransactionLog transactionLogProvider;

  @Inject
  public RealBillingService(CreditCardProcessor processorProvider,
      TransactionLog transactionLogProvider) {
    this.processorProvider = processorProvider;
    this.transactionLogProvider = transactionLogProvider;
  }
}
```
方法注入
```
// 方法注入
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  private static final String DEFAULT_API_KEY = "development-use-only";
  private String apiKey = DEFAULT_API_KEY;

  @Inject
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
}
```
属性注入
```
// 属性注入
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  @Inject Connection connection;

  public TransactionLog get() {
    returnnew DatabaseTransactionLog(connection);
  }
}
```

## Guice示例
先通过模仿ES guice的使用方式简单写了一个基本Demo 方便理解, 之后再来理一下ES的Guice使用. 编写的测试类原理图如下:

总共有两个Module，一个是ToolModule，用于绑定IAnimal接口、ITool接口以及Map对象. 另一个是HumanModule 用于绑定Person对象。

## ToolMoudle
iTool接口与实现类
```
public interface ITool {
    public void doWork();
}

public class IToolImpl implements ITool {
    @Override
    public void doWork() {
        System.out.println("use tool to work");
    }
}
```
IAnimal 接口与实现类
```
public interface IAnimal {
    void work();
}

public class IAnimalImpl implements IAnimal {
    @Override
    public void work() {
        System.out.println("animals can also do work");
    }
}
```
ToolModule的实现, 它绑了三个实例
```
public class ToolModule extends AbstractModule {

    @Override
    protected void configure() {
        //此处注入的实例可以注入到其他类的构造函数中, 只要那个类使用@Inject进行注入即可
        bind(IAnimal.class).to(IAnimalImpl.class);
        bind(ITool.class).to(IToolImpl.class);

        // 注入Map实例
        MapBinder<String, String> mapBinder = MapBinder.newMapBinder(binder(), String.class, String.class);
        mapBinder.addBinding("test1").toInstance("test1");
        mapBinder.addBinding("test2").toInstance("test2");
    }
}
```
将接口与其具体实现绑定起来
```
bind(IAnimal.class).to(IAnimalImpl.class);
bind(ITool.class).to(IToolImpl.class);  
```
完成Map的绑定.
```
MapBinder<String,String> mapBinder =MapBinder.newMapBinder(binder(), String.class, String.class);
mapBinder.addBinding("test1").toInstance("test1");
mapBinder.addBinding("test2").toInstance("test2"); 
```
HumanModule
```
public class Person {

    private IAnimal iAnimal;
    private ITool iTool;
    private Map<String, String> map;

    @Inject
    public Person(IAnimal iAnimal, ITool iTool, Map<String, String> map) {
        this.iAnimal = iAnimal;
        this.iTool = iTool;
        this.map = map;
    }

    public void startwork() {
        iTool.doWork();
        iAnimal.work();
        for (Map.Entry entry : map.entrySet()) {
            System.out.println("注入的map 是 " + entry.getKey() + " value " + entry.getValue());
        }
    }
}
```
Person 类中由 IAnimal、ITool 和 Map<String, String> 这三个接口定义的变量，对象将通过 @Inject 从构造方法中注入进来
```
public class HumanModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(Person.class).asEagerSingleton();
    }
}
```
Person类的构造函数是通过注入的方式，注入对象实例的。

## CustomModuleBuilder
最后 CustomModuleBuilder 进行统一管理所有的Module，实例化所有Module中的对象. 完成依赖注入。

这里的CustomModuleBuilder是修改自Elasticsearch中的ModulesBuilder，其原理是一样的。

就是一个迭代器，内部封装的是Module集合, 统一管理所有的Module
```
public class CustomModuleBuilder implements Iterable<Module> {

    private final List<Module> modules = new ArrayList<>();

    public CustomModuleBuilder add(Module... newModules) {
        for (Module module : newModules) {
            modules.add(module);
        }
        return this;
    }

    @Override
    public Iterator<Module> iterator() {
        return modules.iterator();
    }

    public Injector createInjector() {
        Injector injector = Guice.createInjector(modules);
        return injector;
    }
}
```
Main 方法
```
public class Main {
    public static void main(String[] args) {
        CustomModuleBuilder moduleBuilder = new CustomModuleBuilder();
        moduleBuilder.add(new ToolModule());
        moduleBuilder.add(new HumanModule());
        Injector injector = moduleBuilder.createInjector();
        Person person = injector.getInstance(Person.class);
        person.startwork();
    }
}
```

## Guice在Node初始化中的应用
Node初始化
在以下代码之前，Node初始化的代码已经完成了NetworkService、ClusterService、IngestService等服务的初始化，这里进行Moudle创建并添加到ModulesBuilder。通过查看Moudel初始化代码，能够看到该Moudle依赖的Service。
```
//ModulesBuilder类加入各种模块 ScriptModule、AnalysisModule、SettingsModule、pluginModule、ClusterModule、
// IndicesModule、SearchModule、GatewayModule、RepositoriesModule、ActionModule、NetworkModule、DiscoveryModule
ModulesBuilder modules = new ModulesBuilder();
// plugin modules must be added here, before others or we can get crazy injection errors...
// 这里面把包括上面所说的ActionModule全部添加进来
for (Module pluginModule : pluginsService.createGuiceModules()) {
    modules.add(pluginModule);
}
final MonitorService monitorService = new MonitorService(settings, nodeEnvironment, threadPool, clusterInfoService);
ClusterModule clusterModule = new ClusterModule(settings, clusterService, clusterPlugins, clusterInfoService);
modules.add(clusterModule);
IndicesModule indicesModule = new IndicesModule(pluginsService.filterPlugins(MapperPlugin.class));
modules.add(indicesModule);

SearchModule searchModule = new SearchModule(settings, false, pluginsService.filterPlugins(SearchPlugin.class));
CircuitBreakerService circuitBreakerService = createCircuitBreakerService(settingsModule.getSettings(),
    settingsModule.getClusterSettings());
modules.add(new GatewayModule());
```
Moudle初始化
Moudle初始化后加入到ModulesBuilder，对应的是以下几个：

Guice提供依赖配置类，具体的Moudle需要继承AbstractModule或者实现Moudle接口，实现cofigure方法，完成Binder配置绑定。

## PluginMoudle
插件模块 在PluginService服务初始化时，已经把所有的插件模块进行加载。Guice绑定依赖Plugin接口的createGuiceModules方法。
```
/**
 * Node level guice modules.
 */
public Collection<Module> createGuiceModules() {
    return Collections.emptyList();
}
```
以AnalyticsPlugin为例，查看createGuiceModules方法
```
public Collection<Module> createGuiceModules() {
    List<Module> modules = new ArrayList<>();

    if (transportClientMode) {
        return modules;
    }
    //b -> xxx是对接口configure方法的实现，入参是Binder
    modules.add(b -> XPackPlugin.bindFeatureSet(b, AnalyticsFeatureSet.class));
    return modules;
}
```

ActionMoudle

ActionMoudle继承AbstractModule初始化后，进行绑定。绑定类型包含类名绑定、泛型绑定、集合类型绑定等，代码如下：
```
@Override
protected void configure() {
    //1. 类名绑定。绑定一个现存的实例, 将手动实例化的实例注册到对应的类上
    bind(ActionFilters.class).toInstance(actionFilters);
    bind(DestructiveOperations.class).toInstance(destructiveOperations);
    //2. 泛型绑定
    bind(new TypeLiteral<RequestValidators<PutMappingRequest>>() {}).toInstance(mappingRequestValidators);
    bind(new TypeLiteral<RequestValidators<IndicesAliasesRequest>>() {}).toInstance(indicesAliasesRequestRequestValidators);

    if (false == transportClient) {
        // Supporting classes only used when not a transport client
        bind(AutoCreateIndex.class).toInstance(autoCreateIndex);
        // 单例绑定
        bind(TransportLivenessAction.class).asEagerSingleton();

        // register ActionType -> transportAction Map used by NodeClient
        @SuppressWarnings("rawtypes")
        //3. 集合类型绑定
        MapBinder<ActionType, TransportAction> transportActionsBinder
                = MapBinder.newMapBinder(binder(), ActionType.class, TransportAction.class);
        for (ActionHandler<?, ?> action : actions.values()) {
            // bind the action as eager singleton, so the map binder one will reuse it
            // 这里是将循环将所有TransportAction相关实例注册起来，其中构造函数依赖的实例会自动注入
            bind(action.getTransportAction()).asEagerSingleton();
            transportActionsBinder.addBinding(action.getAction()).to(action.getTransportAction()).asEagerSingleton();
            for (Class<?> supportAction : action.getSupportTransportActions()) {
                bind(supportAction).asEagerSingleton();
            }
        }
    }
}
```
guice 绑定依赖以及依赖注入，这里b -> xxx依旧是对 configure方法的具体实现。
```
//elasticsearch里面的组件基本都进行进行了模块化管理，elasticsearch对guice进行了封装，通过ModulesBuilder类构建es的模块
modules.add(b -> {
        b.bind(Node.class).toInstance(this);
        b.bind(NodeService.class).toInstance(nodeService);
        b.bind(NamedXContentRegistry.class).toInstance(xContentRegistry);
        b.bind(PluginsService.class).toInstance(pluginsService);
        b.bind(Client.class).toInstance(client);
..........................................
```
到此Moudle的初始化完成并添加到ModulesBuilder，接下来该依赖的注入。

## 依赖注入
createInjector
调用createInjector方法把Service注入的ES环境中，这里的实例都是单例。
```
// 生成注入器, 利用Guice将各种模块以及服务(xxxService)注入到Elasticsearch环境中
injector = modules.createInjector();
-----------------------------------------------------
//实例都是单例singletons
public Injector createInjector() {
    Injector injector = Guice.createInjector(modules);
    ((InjectorImpl) injector).clearCache();
    // in ES, we always create all instances as if they are eager singletons
    // this allows for considerable memory savings (no need to store construction info) as well as cycles
    ((InjectorImpl) injector).readOnlyAllSingletons();
    return injector;
}
```
初始化
injector.getInstance可以根据类或者Type获取实例。injector.getInstance(newKey<Map<ActionType,TransportAction>>(){})获取<ActionType,TransportAction>实例，即Rest**Action与Transport**Action的对应Map。在ES中，客户端Client和服务端Broker之间的请求称为REST请求，集群内不同Node之间的请求称为RPC请求，REST请求和RPC请求都称为Action。ActionType是一个通用Action，是Client与Broker之间的请求，以创建索引为例对应的是CreateIndexAction；TransportAction是Node节点之间的请求，以创建索引为例对应的是TransportCreateIndexAction，

Node类中，client作为节点触发TransportAction的入口，会将所有TransportAction注入进来。

client.initialize执行了初始化。
```
//Key:Binder中对应一个Provider
//Node类中，client作为节点触发TransportAction的入口，会将所有TransportAction注入进来
client.initialize(injector.getInstance(new Key<Map<ActionType, TransportAction>>() {}),
                    () -> clusterService.localNode().getId(), transportService.getRemoteClusterService());
```
使用
TransportAction映射提到的transportAction(action)方法，actions属性的初始化在这里完成。
```
/**
 * Get the {@link TransportAction} for an {@link ActionType}, throwing exceptions if the action isn't available.
 */
@SuppressWarnings("unchecked")
private <    Request extends ActionRequest,
            Response extends ActionResponse
        > TransportAction<Request, Response> transportAction(ActionType<Response> action) {
    if (actions == null) {
        throw new IllegalStateException("NodeClient has not been initialized");
    }
    //根据action获取TransportAction的实现类
    TransportAction<Request, Response> transportAction = actions.get(action);
    if (transportAction == null) {
        throw new IllegalStateException("failed to find action [" + action + "] to execute");
    }
    return transportAction;
}
```


