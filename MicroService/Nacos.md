# Nacos注册中心

Nacos是Dynamic Naming and Configuration Service的首字母简称，定位于一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

## 1、注册中心概念

**价值**：解决分布式场景下，服务之间互相发现的问题

**流程**：服务提供方注册实例信息到注册中心；服务消费者订阅服务提供者的变更；当服务提供方实例信息变更时，注册中心推送最新的实例信息给服务消费者；服务消费者通过服务提供者的实例信息对服务提供者发起调用

**误区**：服务消费者调用服务提供者的时候请求需要经过注册中心

![注册中心图](..\Resource\Nacos\注册中心.png)

## 2、持久化实例VS临时实例

**持久化实例**：通常适用于mysql、redis等无法引用SDK的服务，一旦注册**实例会一直存在**，直到被**手动注销**

**临时实例**：注册后通过**心跳**或者**长连接**方式与服务端保持健康检查，一旦心跳丢失一定时间或者长连接断开实例会**自动被注销**

| 对比项   | 临时实例     | 持久化实例 |
| ------------------------------ | -------- | -------- |
| 是否能主动与服务端维持健康检查    | 是   | 否       |
| 优先级     | 默认   | 需要参数指定      |
| grpc注册    | 支持 | 不支持       |
| 是否自动注销   | 是     | 否       |

## 3、服务注册-Client发起注册

**补偿机制**：SDK发起注册后会定时补偿，保证最终注册成功（临时实例）；网络异常或者服务端宕机导致连接断开，SDK无限重连，重新发起注册，保证数据不丢失（临时实例）；**持久化实例注册无补偿机制**

![临时实例补偿图](..\Resource\Nacos\临时实例补偿.png)

## 4、服务注册-Server处理注册

![server处理注册图](..\Resource\Nacos\server处理注册.png)

## 5、服务注册-一致性DistroVSJRaft

| 对比项   | Distro     | 持久化实例 |
| ------------------------------ | -------- | -------- |
| 协议类型    | AP（保证可用性）   | CP（保证一致性）       |
| Nacos应用场景     | 临时实例数据   | 持久化实例数据；持久化服务数据；实例元数据；配置数据（不使用mysql）      |
| 实现原理    | 1.每个节点都可接受写请求 2.定时对账 3.异步同步 | 基于Raft协议实现一致性，只有Leader才能接受写请求       |
| 优缺点   | 性能高，数据最终一致     | 强一致性，性能相比Distro协议低，对磁盘敏感       |

## 6、健康检查-临时实例

![健康检查图](..\Resource\Nacos\健康检查.png)

* 1.x版本通过客户端主动定时上报心跳来维持健康状态，实例不健康的时间可以通过**preserved.heart.beat.timeout**参数来设置，实例被删除的时间可以通过**preserved.ip.delete.timeout**参数来设置，服务端通过多线程和定时任务来进行检查
* 2.x版本依赖grpc长连接来维持健康状态，当grpc连接断开实例被删除

## 7、健康检查-持久化实例

* 健康检查协议：TCP（默认）、MySQL协议、HTTP协议、不检查

![健康检查持久化实例图](..\Resource\Nacos\健康检查持久化实例.png)

## 8、服务发现-订阅与推送

**订阅**：订阅是为了收到推送，订阅也有补偿机制

当发生推送时，如果连接处于断开状态，则不会重试，仅仅网络异常，但是连接还在的时候推送失败会重试

![推送过程图](..\Resource\Nacos\推送过程.png)

## 9、服务发现-查询

![服务发现查询图](..\Resource\Nacos\服务发现查询.png)

## Nacos的QA

* Q：假如Nacos有三个节点，是三个节点都在检查，还是只有一个节点?
* A：Nacos会把健康检查任务分布到集群的每个节点中，假如Nacos服务有三个节点，总共有9个服务实例，那每个节点是负责三个实例的健康检查。

* Q：Nacos节点挂了，节点检查xx？
* A：如果Nacos某个节点挂了，这个节点负责的客户端会感知到并且重新注册到其他节点，少数节点宕机不影响整体可用性，所有节点宕机不影响运行中的服务。

* Q：客户端，服务端之间沟通有安全问题吗？
* A：目前不管是Nacos的1.x的http请求需要经过鉴权，2.x的版本客户端与服务端创建grpc连接的时候也需要经过鉴权。

* Q：nacos的distro协议是最终一致，如果有强一致的需求咋办，也就是有没有办法一定能拿到最新的数据？nacos的订阅消息可能会丢失么？
* A：持久化实例的jraft协议就是强一致的。订阅操作客户端同样具有补偿机制，客户端与服务端连接断线重连会重新发起订阅，客户端还有一个1分钟执行一次的兜底的定时任务来保证数据一致，另外客户端每次重启都会全量拉取服务端所有数据，所以数据不会丢失。

* Q：从健康检查初次出现问题，到确认实例不健康，实例仍然会被当作可用的节点吗？如果觉得这个时间阈值太久，能否自己调节？
* A：1.x版本的临时实例健康检查到实例删除的时间默认是30秒，在实例被删除期间都会被当做可用节点，可用通过在控制台实例列表修改元数据设置preserved.ip.delete.timeout 来修改这个时间，但是建议直接升级到2.1.0的nacos-client的版本，因为2.x版本健康检查依赖的是grpc的连接状态，可以秒级感知到实例下线。

# Nacos的源码分析（结合spring-cloud-alibaba + dubbo + nacos的整合）：

**服务注册的流程**：在基于Dubbo服务发布的过程中，自动装配是走的事件监听机制，在 DubboServiceRegistrationNonWebApplicationAutoConfiguration 这个类中，这个类会监听 ApplicationStartedEvent 事件，这个事件是spring boot在2.0新增的，就是当spring boot应用启动完成之后会发布这个事件。而此时监听到这个事件之后，会触发注册的动作。

```java
@EventListener(ApplicationStartedEvent.class)
public void onApplicationStarted() {
    setServerPort();
    register();
}

private void register() {
    if (registered) {
        return;
    }
    serviceRegistry.register(registration);
    registered = true;
}
```

serviceRegistry 是 spring-cloud 提供的接口实现(org.springframework.cloud.client.serviceregistry.ServiceRegistry)，很显然注入的实例是：NacosServiceRegistry。

```java
@Override
public void register(Registration registration) {
    if (StringUtils.isEmpty(registration.getServiceId())) {
        log.warn("No service to register for nacos client...");
        return;
    }
    //对应当前应用的application.name
    String serviceId = registration.getServiceId();
    //表示nacos上的分组配置
    String group = nacosDiscoveryProperties.getGroup();
    //表示服务实例信息
    Instance instance = getNacosInstanceFromRegistration(registration);

    try {
        //通过命名服务进行注册
        namingService.registerInstance(serviceId, group, instance);
        log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
                instance.getIp(), instance.getPort());
    }
    catch (Exception e) {
        log.error("nacos registry, {} register failed...{},", serviceId,
                registration.toString(), e);
        // rethrow a RuntimeException if the registration is failed.
        // issue : https://github.com/alibaba/spring-cloud-alibaba/issues/1132
        rethrowRuntimeException(e);
    }
}    
```

然后进入到实现类的注册方法开始注册实例，主要做两个动作：1. 如果当前注册的是临时节点，则构建心跳信息，通过beat反应堆来构建心跳任务；2. 调用registerService发起服务注册

```java
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    ////是否是临时节点，如果是临时节点，则构建心跳信息
    if (instance.isEphemeral()) {
        BeatInfo beatInfo = new BeatInfo();
        beatInfo.setServiceName(NamingUtils.getGroupedName(serviceName, groupName));
        beatInfo.setIp(instance.getIp());
        beatInfo.setPort(instance.getPort());
        beatInfo.setCluster(instance.getClusterName());
        beatInfo.setWeight(instance.getWeight());
        beatInfo.setMetadata(instance.getMetadata());
        beatInfo.setScheduled(false);

        //beatReactor， 添加心跳信息进行处理
    beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
    }
    //调用服务代理类进行注册   
    serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
}
```

然后调用 NamingProxy 的注册方法进行注册，构建请求参数，发起请求。往下走我们就会发现服务在进行注册的时候会轮询配置好的注册中心的地址，最后通过 callServer(api, params, server, method) 发起调用，这里通过JSK自带的 HttpURLConnection 进行发起调用

```java
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

    NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}",
        namespaceId, serviceName, instance);

    final Map<String, String> params = new HashMap<String, String>(8);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put(CommonParams.GROUP_NAME, groupName);
    params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
    params.put("ip", instance.getIp());
    params.put("port", String.valueOf(instance.getPort()));
    params.put("weight", String.valueOf(instance.getWeight()));
    params.put("enable", String.valueOf(instance.isEnabled()));
    params.put("healthy", String.valueOf(instance.isHealthy()));
    params.put("ephemeral", String.valueOf(instance.isEphemeral()));
    params.put("metadata", JSON.toJSONString(instance.getMetadata()));

    reqAPI(UtilAndComs.NACOS_URL_INSTANCE, params, HttpMethod.POST);

}
```

```java
public String reqAPI(String api, Map<String, String> params, List<String> servers, String method) {

    params.put(CommonParams.NAMESPACE_ID, getNamespaceId());

    if (CollectionUtils.isEmpty(servers) && StringUtils.isEmpty(nacosDomain)) {
        throw new IllegalArgumentException("no server available");
    }

    Exception exception = new Exception();
    //如果服务地址不为空
    if (servers != null && !servers.isEmpty()) {
        //随机获取一台服务器节点
        Random random = new Random(System.currentTimeMillis());
        int index = random.nextInt(servers.size());
        // 遍历服务列表
        for (int i = 0; i < servers.size(); i++) {
            String server = servers.get(index);//获得索引位置的服务节点
            try {//调用指定服务
                return callServer(api, params, server, method);
            } catch (NacosException e) {
                exception = e;
                NAMING_LOGGER.error("request {} failed.", server, e);
            } catch (Exception e) {
                exception = e;
                NAMING_LOGGER.error("request {} failed.", server, e);
            }
            //轮询
            index = (index + 1) % servers.size();
        }
    // ..........
    }
}
```

**Nacos服务端接受服务提供者注册请求的处理流程**：Nacos服务端提供了一个InstanceController类，在这个类中提供了服务注册相关的API，而服务提供者发起注册时，调用的接口是：[post]: /nacos/v1/ns/instance
，serviceName: 代表客户端的项目名称 ，namespace: nacos 的namespace。

```java
@CanDistro
@PostMapping
@Secured(parser = NamingResourceParser.class, action = ActionTypes.WRITE)
public String register(HttpServletRequest request) throws Exception {
        
    final String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    final String namespaceId = WebUtils
            .optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
    // 从请求中解析出instance实例
    final Instance instance = parseInstance(request);
    
    serviceManager.registerInstance(namespaceId, serviceName, instance);
    return "ok";
}
```

然后调用ServiceManager进行服务的注册

```java
public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    //创建一个空服务，在Nacos控制台服务列表展示的服务信息，实际上是初始化一个serviceMap，它是一个ConcurrentHashMap集合
    createEmptyService(namespaceId, serviceName, instance.isEphemeral());
    //从serviceMap中，根据namespaceId和serviceName得到一个服务对象
    Service service = getService(namespaceId, serviceName);
    
    if (service == null) {
        throw new NacosException(NacosException.INVALID_PARAM,
                "service not found, namespace: " + namespaceId + ", service: " + serviceName);
    }
    //调用addInstance创建一个服务实例
    addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
}
```

在创建空的服务实例的时候我们发现了存储实例的map，在getService方法中我们发现了Map：

```java
/*
* Map(namespace, Map(group::serviceName, Service)).
*/
private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
```

```java
public void createServiceIfAbsent(String namespaceId, String serviceName, boolean local, Cluster cluster) throws NacosException {
    //从serviceMap中获取服务对象
    Service service = getService(namespaceId, serviceName);
    if (service == null) {//如果为空。则初始化
      Loggers.SRV\_LOG.info("creating empty service {}:{}", namespaceId, serviceName);
      service = new Service();
      service.setName(serviceName);
      service.setNamespaceId(namespaceId);
      service.setGroupName(NamingUtils.getGroupName(serviceName));
      // now validate the service. if failed, exception will be thrown
      service.setLastModifiedMillis(System.currentTimeMillis());
      service.recalculateChecksum();
      if (cluster != null) {
          cluster.setService(service);
          service.getClusterMap().put(cluster.getName(), cluster);
      }
      service.validate();
      putServiceAndInit(service);
      if (!local) {
          addOrReplaceService(service);
      }
    }
}
```

Nacos是通过不同的namespace来维护服务的，而每个namespace下有不同的group，不同的group下才有对应的Service，再通过这个serviceName来确定服务实例。第一次进来则会进入初始化，初始化完会调用 putServiceAndInit，获取到服务以后把服务实例添加到集合中，然后基于一致性协议进行数据的同步。然后调用addInstance，最后给服务注册方发送注册成功的响应，结束服务注册流程。


```java
private void putServiceAndInit(Service service) throws NacosException {
    putService(service);//把服务信息保存到serviceMap集合
    service.init();//建立心跳检测机制
    //实现数据一致性监听，ephemeral(标识服务是否为临时服务，默认是持久化的，也就是true)=true表示采用raft协议，false表示采用Distro
    consistencyService
            .listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), true), service);
    consistencyService
            .listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), false), service);
    Loggers.SRV_LOG.info("[NEW-SERVICE] {}", service.toJson());
}
```


```java
public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips) throws NacosException {
    // 组装key
    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);
    // 获取刚刚组装的服务
    Service service = getService(namespaceId, serviceName);
    
    synchronized (service) {
        List<Instance> instanceList = addIpAddresses(service, ephemeral, ips);
        
        Instances instances = new Instances();
        instances.setInstanceList(instanceList);
        // 也就是上一步实现监听的类里添加注册服务
        consistencyService.put(key, instances);
    }
}
```




