---
title: apisix-seed源码解析
date: 2023-12-27 22:00:00 +0800
categories: [原理]
tags: [apisix-seed, apisix]
author: stephan
mermaid: true
---

## 前言

在之前图文分享系统相关文章中，有用到apisix来作为系统网关，apisix集成了服务发现功能，支持主流的服务发现组件如consul、zookeeper、nacos等，在apisix中，接入服务发现功能提供两种方案：
1. 集成在apisix里，在apisix启动前需要先配置服务发现组件，参考：[apisix服务发现](https://apisix.apache.org/zh/docs/apisix/discovery/)
2. 通过apisix-seed发现服务，将服务发现从apisix数据面抽离，通过apisix-seed实现服务发现的控制面，加入apisix-seed后整个apisix网关框架图如下：
![avatar](/assets/img/nov/apisix-discovery-seed.jpg)
图中的数字代表的具体信息如下：
    1. 通过Admin API向APISIX注册上游并指定服务发现类型，APISIX-Seed将监听etcd中的APISIX资源变化，过滤服务发现类型并获取服务名称（如ZooKeeper）；
    2. APISIX-Seed将在服务注册中心（如ZooKeeper）订阅指定的服务名称，以监控和更新对应的服务信息；
    3. 客户端向服务注册中心注册服务后，APISIX-Seed会获取新的服务信息，并将更新后的服务节点写入etcd；
    4. 当APISIX-Seed在etcd中更新相应的服务节点信息时，APISIX会将最新的服务节点信息同步到内存中。

在深入理解apisix-seed实现前，先来了解下apisix中的几个概念：
- Route（路由）：是APISIX中最基础和最核心的资源对象，APISIX可以通过路由定义规则来匹配客户端请求，根据匹配结果加载并执行相应的插件，最后将请求转发给到指定的上游服务。
- Service（也称之为服务）：是某类API的抽象（也可以理解为一组Route的抽象）。它通常与上游服务抽象是一一对应的，但与路由之间，通常是1:N即一对多的关系。不同路由规则同时绑定到一个服务上，这些路由将具有相同的上游和插件配置，减少冗余配置。
- Upstream（也称之为上游）：是对虚拟主机抽象，即应用层服务或节点的抽象。你可以通过Upstream对象对多个服务节点按照配置规则进行负载均衡。

以上是在apisix中可以配置服务发现的3类信息，**由于apisix使用etcd作为持久化，以上路由等配置信息，通过admin api配置完成后，最终在etcd中以k-v形式存储，不同的配置信息在etcd中通过不同的key前缀区分，apisix-seed通过订阅key前缀从而实时捕捉到配置变更。**

## apisix-seed源码解析
在了解了一些基本概念和apisix-seed工作原理后，下面来看看在代码层面apisix-seed是如何实现的，apisix-seed是go写的，代码仓库：[apisix-seed仓库](https://github.com/api7/apisix-seed)

#### 程序入口：
```go
func main() {
	conf.InitConf()

	if err := initLogger(conf.LogConfig); err != nil {
		log.Fatal(err)
	}

	etcdClient, err := storer.NewEtcd(conf.ETCDConfig)
	if err != nil {
		panic(err)
	}

	wg := sync.WaitGroup{}
	wg.Add(2)
	go func() {
		defer wg.Done()
		err := storer.InitStores(etcdClient)
		if err != nil {
			panic(err)
		}
	}()
	go func() {
		defer wg.Done()
		err := discoverer.InitDiscoverers()
		if err != nil {
			panic(err)
		}
	}()
	wg.Wait()

	rewriter := components.Rewriter{
		Prefix: conf.ETCDConfig.Prefix,
	}
	rewriter.Init()
	defer rewriter.Close()

	watcher := components.Watcher{}
	watcher.Watch()
	err = watcher.Init()
	if err != nil {
		log.Error(err.Error())
		return
	}

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	sig := <-quit
	log.Infof("APISIX-Seed receive %s and start shutting down", sig.String())
}
```

程序启动的第一步是读取配置文件，并抽取配置信息，配置文件内容如下：
```yaml
etcd:
  host:                           # it's possible to define multiple etcd hosts addresses of the same etcd cluster.
    - "http://127.0.0.1:2379"     # multiple etcd address, if your etcd cluster enables TLS, please use https scheme,
  prefix: /apisix                 # apisix configurations prefix
  timeout: 30                     # 30 seconds
  #user: root                     # root username for etcd
  #password: 5tHkHhYkjr6cQY       # root password for etcd
  tls:
    #cert: /path/to/cert          # path of certificate used by the etcd client
    #key: /path/to/key            # path of key used by the etcd client

    verify: true                  # whether to verify the etcd endpoint certificate when setup a TLS connection to etcd,
    # the default value is true, e.g. the certificate will be verified strictly.
log:
  level: warn
  path: apisix-seed.log           # path is the file to write logs to.  Backup log files will be retained in the same directory
  maxage: 168h                    # maxage is the maximum number of days to retain old log files based on the timestamp encoded in their filename
  maxsize: 104857600              # maxsize is the maximum size in megabytes of the log file before it gets rotated. It defaults to 100mb
  rotation_time: 1h               # rotation_time is the log rotation time

discovery:                       # service discovery center
  nacos:
    host:                        # it's possible to define multiple nacos hosts addresses of the same nacos cluster.
      - "http://127.0.0.1:8848"
    prefix: /nacos
    user:      "admin"             # username for nacos
    password:  "5tHkHhYkjr6cQY"    # password for nacos
    weight: 100                  # default weight for node
    timeout:
      connect: 2000              # default 2000ms
      send: 2000                 # default 2000ms
      read: 5000                 # default 5000ms
  zookeeper:
    hosts:
      - "127.0.0.1:2181"
    prefix: /zookeeper
    weight: 100                  # default weight for node
    timeout: 10                  # default 10s

```
主要是etcd、服务发现地址等信息，截止到目前apisix-seed只支持zookeeper和nacos服务发现。

获取配置信息后，初始化etcd client，然后通过两个协程并行初始化store和服务发现(`InitDiscoverers`)，`InitStores`主要是初始化三个不同store对应etcd里面key prefix为routes，services，upstreams，store的主要作用是通过key前缀从etcd订阅变更，从而可以捕捉到apisix routes/services/upstreams配置变更；

#### 将服务地址信息回写到etcd
接下来的rewriter是apisix-seed中比较重要的一块，主要作用是从服务发现组件监听地址变化然后将服务ip地址回写到etcd中，rewriter.Init代码如下：
```go
func (r *Rewriter) Init() {
	r.ctx, r.cancel = context.WithCancel(context.TODO())
	// the number of semaphore is referenced to https://github.com/golang/go/blob/go1.17.1/src/cmd/compile/internal/noder/noder.go#L38
	r.sem = make(chan struct{}, runtime.GOMAXPROCS(0)+10)

	// Watch for service updates from Discoverer
	for _, dis := range discoverer.GetDiscoverers() {
		msgCh := dis.Watch()
		go r.watch(msgCh)
	}
}
```
对于每一个服务发现，开启一个协程监听服务地址变化，`dis.Watch`方法返回一个channel，监听的服务ip地址有变更discovery会从这个channel发送消息，而`r.watch(channel)`方法是监听这个channel，对接收到的消息进行处理，逻辑如下：
```go
func (r *Rewriter) watch(ch chan *message.Message) {
	for {
		select {
		case <-r.ctx.Done():
			return
		case msg := <-ch:
			// hand watcher notify message
			_, entity, _ := storer.FromatKey(msg.Key, r.Prefix)
			if entity == "" {
				log.Errorf("key format Invaild: %s", msg.Key)
				return
			}
			if err := storer.GetStore(entity).UpdateNodes(r.ctx, msg); err != nil {
				log.Errorf("update nodes failed: %s", err)
			}
		}
	}
}
```
可以看到watch方法通过for+select方式监听channel，接收到信息会抽取msg信息，根据msg内容找到具体的etcd store，然后进行UpdateNodes操作——即将服务地址信息更新到etcd，这样apisix就有了服务的ip地址信息，从而自动发现服务。

看到这里，我们也许有疑问，那discovery会在什么时候在上述的channel中发消息？答案会在接下来要讲的`watcher.Init()`方法中。

#### 订阅dicovery
一般服务发现组件都支持订阅功能，如nacos和zookeeper，apisix-seed会根据etcd中存储的服务发现信息，从服务发现组件中订阅相关服务配置，从而当服务地址变更时能实时捕获并更新到apisix的etcd中，main方法`watcher.Init()`的作用：**在程序启动时，从etcd获取服务发现配置，根据配置，从服务发现组件获取服务ip地址并从服务发现组件订阅服务变更**，代码如下：
```go
func (w *Watcher) Init() error {
	// the number of semaphore is referenced to https://github.com/golang/go/blob/go1.17.1/src/cmd/compile/internal/noder/noder.go#L38
	w.sem = make(chan struct{}, runtime.GOMAXPROCS(0)+10)

	loadSuccess := true
	// List the initial information
	for _, s := range storer.GetStores() {
		//eg: query from etcd by prefix /apisix/routes/
		msgs, err := s.List(message.ServiceFilter)
		if err != nil {
			log.Errorf("storer list error: %v", err)
			loadSuccess = false
			break
		}

		if len(msgs) == 0 {
			continue
		}
		wg := sync.WaitGroup{}
		wg.Add(len(msgs))
		for _, msg := range msgs {
			w.sem <- struct{}{}
			go w.handleQuery(msg, &wg)
		}
		wg.Wait()
	}

	if !loadSuccess {
		return errors.New("failed to load all etcd resources")
	}
	return nil
}
```
代码非常清晰，s.List会根据store key前缀从etcd中查询获取到有服务发现信息的route/upstream/service并**缓存配置到一个map中**，将配置封装成消息并通过handleQuery方法处理消息，handleQuery方法如下：
```go
// handleQuery: init and query the service from discovery by apisix's conf
func (w *Watcher) handleQuery(msg *message.Message, wg *sync.WaitGroup) {
	defer func() {
		<-w.sem
		wg.Done()
	}()

	_ = discoverer.GetDiscoverer(msg.DiscoveryType()).Query(msg)
}
```
handleQuery会根据配置的服务发现类型查询对应服务发现组件Query方法，根据配置的服务发现唯一地址，查询获取到服务的ip地址，以nacos Query方法为例：
```go
func (d *NacosDiscoverer) Query(msg *message.Message) error {
	serviceId := serviceID(msg.ServiceName(), msg.DiscoveryArgs())

	d.cacheMutex.Lock()
	defer d.cacheMutex.Unlock()

	if discover, ok := d.cache[serviceId]; ok {
		// cache information is already available
		msg.InjectNodes(discover.nodes)
		discover.a6Conf[msg.Key] = msg
	} else {
		// fetch new service information
		dis := &NacosService{
			id:   serviceId,
			name: msg.ServiceName(),
			args: msg.DiscoveryArgs(),
		}
		nodes, err := d.fetch(dis)
		if err != nil {
			return err
		}

		msg.InjectNodes(nodes)

		dis.nodes = nodes
		dis.a6Conf = map[string]*message.Message{
			msg.Key: msg,
		}

		d.cache[serviceId] = dis
	}
	d.msgCh <- msg

	return nil
}
```
可以看到，discovery也会对service信息进行缓存，如果有缓存直接从缓存中获取到服务ip地址，没有则通过discovery接口查询(d.fetch(dis)接口)并设置缓存，nacos fetch接口代码如下：
```go
func (d *NacosDiscoverer) fetch(service *NacosService) ([]*message.Node, error) {
	// if the namespace client has not yet been created
	namespace, _ := service.args["namespace_id"].(string)
	if _, ok := d.namingClients[namespace]; !ok {
		err := d.newClient(namespace)
		if err != nil {
			return nil, err
		}
	}

	client := d.namingClients[namespace][d.hash(service.id, namespace)]

	groupName, _ := service.args["group_name"].(string)
	serviceInfo, err := client.GetService(vo.GetServiceParam{
		ServiceName: service.name,
		GroupName:   groupName,
	})
	if err != nil {
		log.Errorf("Nacos get service[%s] error: %s", service.name, err)
		return nil, err
	}

	// watch the new service
	if err = d.subscribe(service, client); err != nil {
		log.Errorf("Nacos subscribe service[%s] error: %s", service.name, err)
		return nil, err
	}

	// metadata
	metadata := service.args["metadata"]
	nodes := make([]*message.Node, 0)
	for _, host := range serviceInfo.Hosts {
		if metadata != nil {
			discard := 0
			for k, v := range metadata.(map[string]interface{}) {
				if host.Metadata[k] != v {
					discard = 1
				}
			}
			if discard == 1 {
				continue
			}
		}

		weight := int(host.Weight)
		if weight == 0 {
			weight = d.weight
		}

		nodes = append(nodes, &message.Node{
			Host:   host.Ip,
			Port:   int(host.Port),
			Weight: weight,
		})
	}

	return nodes, nil
}
```
可以看到除查询获取服务ip地址外，还通过subscribe方法对服务地址进行订阅，在获取到服务ip地址后，通过channel将信息发送出去，前面的rewriter就能捕捉到并将信息更新到apisix etcd中，整个流程就串起来了。

## 总结
使用apisix-seed相比直接在apisix中集成服务发现有以下**优点：**
1. 网络拓扑变得更简单：APISIX不需要与每个注册中心保持网络连接，只需要关注etcd中的配置信息即可，这将大大简化网络拓扑。

2. 上游服务总数据量变小：由于registry的特性，APISIX可能会在Worker 中存储全量的registry服务数据，例如Consul_KV。通过引入 APISIX-Seed，APISIX的每个进程将不需要额外缓存上游服务相关信息。

3. 更容易管理：服务发现配置需要为每个APISIX实例配置一次。通过引入 APISIX-Seed，APISIX将对服务注册中心的配置变化无感知。

apisix-seed的**缺点**：
1. apisix-seed作额外组件需要单独部署，一定程度上增加了运维成本。
2. 截止目前apisix-seed不支持集群部署，一旦apisix-seed服务宕机，将无法动态捕捉到服务地址变更从而导致系统问题。
3. apisix-seed目前将apisix配置服务发现的路由等信息、服务发现后的nodes等信息通过内存暂存，没有做持久化，一旦宕机，数据无法恢复。
4. 目前apisix-seed还不成熟，支持的服务发现组件有限（目前只支持nacos、 zookeeper两种），另外可能和apsix存在兼容性问题，见之前提过的issue：[apisix-seed和apisix兼容性问题](https://github.com/apache/apisix/issues/10479)

所以缺陷还是挺多的，实际生产还是慎用，阅读源码学习为主。