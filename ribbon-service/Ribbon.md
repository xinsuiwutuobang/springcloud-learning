# Spring Cloud Ribbon：负载均衡的服务调用
主要给服务间调用及API网关转发提供负载均衡的功能。
# Ribbon简介
在微服务架构中，很多服务都会部署多个，其他服务去调用该服务的时候，如何保证负载均衡是个不得不去考虑的问题。负载均衡可以增加系统的可用性和扩展性，当我们使用RestTemplate来调用其他服务时，Ribbon可以很方便的实现负载均衡功能。
# 使用
- 在服务调用者中增加ribbon依赖
- 配置 端口、注册中心地址及user-service的调用路径。
````
server:
  port: 8301
spring:
  application:
    name: ribbon-service
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/
service-url:
  user-service: http://user-service
````
- 使用@LoadBalanced注解赋予RestTemplate负载均衡的能力(可以看出使用Ribbon的负载均衡功能非常简单，和直接使用RestTemplate没什么两样，只需给RestTemplate添加一个@LoadBalanced即可。)
````
@Configuration
public class RibbonConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
````

## 负载均衡演示
- 启动ribbon-service于8003端口；
- 启动eureka-server于8001端口；
- 启动user-service于8201端口；
- 启动另一个user-service于8202端口，可以通过修改IDEA中的SpringBoot的启动配置实现：

# Ribbon的常用配置
## 全局配置
````
ribbon:
  ConnectTimeout: 1000 #服务请求连接超时时间（毫秒）
  ReadTimeout: 3000 #服务请求处理超时时间（毫秒）
  OkToRetryOnAllOperations: true #对超时请求启用重试机制
  MaxAutoRetriesNextServer: 1 #切换重试实例的最大个数
  MaxAutoRetries: 1 # 切换实例后重试最大次数
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #修改负载均衡算法
````
## 指定服务进行配置
与全局配置的区别就是ribbon节点挂在服务名称下面，如下是对ribbon-service调用user-service时的单独配置。
````
user-service:
  ribbon:
    ConnectTimeout: 1000 #服务请求连接超时时间（毫秒）
    ReadTimeout: 3000 #服务请求处理超时时间（毫秒）
    OkToRetryOnAllOperations: true #对超时请求启用重试机制
    MaxAutoRetriesNextServer: 1 #切换重试实例的最大个数
    MaxAutoRetries: 1 # 切换实例后重试最大次数
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #修改负载均衡算法
````
## Ribbon的负载均衡策略
所谓的负载均衡策略，就是当A服务调用B服务时，此时B服务有多个实例，这时A服务以何种方式来选择调用的B实例，ribbon可以选择以下几种负载均衡策略。
````
com.netflix.loadbalancer.RandomRule：从提供服务的实例中以随机的方式；
com.netflix.loadbalancer.RoundRobinRule：以线性轮询的方式，就是维护一个计数器，从提供服务的实例中按顺序选取，第一次选第一个，第二次选第二个，以此类推，到最后一个以后再从头来过；
com.netflix.loadbalancer.RetryRule：在RoundRobinRule的基础上添加重试机制，即在指定的重试时间内，反复使用线性轮询策略来选择可用实例；
com.netflix.loadbalancer.WeightedResponseTimeRule：对RoundRobinRule的扩展，响应速度越快的实例选择权重越大，越容易被选择；
com.netflix.loadbalancer.BestAvailableRule：选择并发较小的实例；
com.netflix.loadbalancer.AvailabilityFilteringRule：先过滤掉故障实例，再选择并发较小的实例；
com.netflix.loadbalancer.ZoneAwareLoadBalancer：采用双重过滤，同时过滤不是同一区域的实例和故障实例，选择并发较小的实例。
````

# Ribbon原理简述

1 服务列表获取：Ribbon负载的核心接口是LoadBalancerClient， LoadBalancerClient在初始化时会通过Eureka Client向Eureka服务端获取所有服务实例的注册信息，并且每10秒向EurekaClient 发送“ ping ”，来判断服务的可用性。如果服务的可用性发生了改变或者服务数量和之前的不一致，则更新或者重新拉取。
相关源码如下：
````
public class BaseLoadBalancer extends AbstractLoadBalancer implements PrimeConnections.PrimeConnectionListener, IClientConfigAware {
    //...
    private final static IRule DEFAULT_RULE = new RoundRobinRule();
    protected IRule rule = DEFAULT_RULE;
    protected int pingIntervalSeconds = 10;
    public BaseLoadBalancer(String name, IRule rule, LoadBalancerStats stats,IPing ping, IPingStrategy pingStrategy) {	
       //...
        setupPingTask();
       //...
    }
    void setupPingTask() {
        //../
        lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
                true);
        lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
        forceQuickPing();
    }
}
 
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,ServerListUpdater serverListUpdater) {
        //...        
        restOfInit(clientConfig);
    } 
    void restOfInit(IClientConfig clientConfig) {
        //...
        updateListOfServers();
        //...
    }
    public void updateListOfServers() {
        List<T> servers = new ArrayList<T>();
        if (serverListImpl != null) {
            servers = serverListImpl.getUpdatedListOfServers();
            //...
        }     
    }
}
 
public class DiscoveryEnabledNIWSServerList extends bstractServerList<DiscoveryEnabledServer>{
    @Override
    public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
        return obtainServersViaDiscovery();
    }
 
    private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
       //...
       EurekaClient eurekaClient = eurekaClientProvider.get();
       //...
}
````
2 负载算法：LoadBalancerClient的实现类为RibbonLoadBalancerClient，RibbonLoadBalancerClient中的choose(..)会根据实例名，提取出对应的服务实例列表，然后交给ILoadBalancer去选择服务实例，集群环境下ILoadBalancer的实现类为ZoneAwareLoadBalancer，该类采用ZoneAvoidanceRule负载算法，即片区域加轮询的负载算法选择实例。  Ribbon默认提供了以下负载算法（即IRule的实现类）： 
````
BestAvailableRule ： 选择最小请求数。
ClientConfigEnabledRoundRobinRule：轮询。
RandornRule ： 随机选择一个server 。
RoundRobinRule ： 轮询选择server ，又分为换响应时间权重、和
RetryRule ： 根据轮询的方式重试。
WeightedResponseTimeRule： 根据响应时间去分配一个weight , weight 越低，被选择的可能性就越低。
ZoneAvoidanceRule ：根据server的zone区域和可用性来轮询选择。集群环境下默认为这种轮询方式，也就是按片区来轮询。     
      负载相关源码如下：

public class RibbonLoadBalancerClient implements LoadBalancerClient {
    //...
    public ServiceInstance choose(String serviceId, Object hint) {
		Server server = getServer(getLoadBalancer(serviceId), hint);
		//...
	}
    //...
    protected ILoadBalancer getLoadBalancer(String serviceId) {
        return this.clientFactory.getLoadBalancer(serviceId);
    }
｝
 
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {
    public ILoadBalancer getLoadBalancer(String name) {
        return getInstance(name, ILoadBalancer.class);
    }
 
    @Override
	public <C> C getInstance(String name, Class<C> type) {
		C instance = super.getInstance(name, type);
		if (instance != null) {
			return instance;
		}
		IClientConfig config = getInstance(name, IClientConfig.class);
		return instantiateWithConfig(getContext(name), type, config);
	}
｝
````
整体来说，Ribbon底层是通过Eureka Client获取服务实例列表并本地缓存，然后通过心跳检测服务可用性，当访问服务时，根据一定的负载算法获取服务实例。

为什么加上了@LoadBalance之就能使调用有负载均衡效果？

原因在@LoadBalance注解的处理器LoadBalancerAutoConfiguration 当中，LoadBalancerAutoConfiguration 中维护了一个被＠LoadBalanced 修饰的RestTemplate 对象的List。在初始化的过程中，通过调用customizer.customize(restTemplate）方
法来给RestTemplate增加拦截器LoadBalancerlnterceptor ，这样调用restTemplate的方法时就会先进入到LoadBalancerinterceptor中，在LoadBalancerlnterceptor 中实现了负载均衡的方法。源码如下：
````
Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
 
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
 
	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();
 
	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);  //配置拦截器
				}
			}
		});
	}
}
````