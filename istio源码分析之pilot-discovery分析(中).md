# Service Mesh深度学习系列（三）| istio源码分析之pilot-discovery模块分析（中）
本文分析的istio代码版本为0.8.0，commit为0cd8d67，commit时间为2018年6月18日。 


## pilot总体架构 
![istio architecture](https://camo.githubusercontent.com/919e2e3cd8e4267a00035b813df53902864a3388/68747470733a2f2f63646e2e7261776769742e636f6d2f697374696f2f70696c6f742f6d61737465722f646f632f70696c6f742e737667)
首先我们回顾一下pilot总体架构，上面是[官方关于pilot的架构图](https://github.com/istio/old_pilot_repo/blob/master/doc/design.md)，因为是old_pilot_repo目录下，可能与最新架构有出入，仅供参考。所谓的pilot包含两个组件：pilot-agent和pilot-discovery。图里的agent对应pilot-agent二进制，proxy对应Envoy二进制，它们两个在同一个容器中，discovery service对应pilot-discovery二进制，在另外一个跟应用分开部署的单独的deployment中。   

1. **discovery service**：从Kubernetes apiserver list/watch `service`、`endpoint`、`pod`、`node`等资源信息，监听istio控制平面配置信息（如VirtualService、DestinationRule等）， 翻译为Envoy可以直接理解的配置格式。
2. **proxy**：也就是Envoy，直接连接discovery service，间接地从Kubernetes等服务注册中心获取集群中微服务的注册情况
3. **agent**：生成Envoy配置文件，管理Envoy生命周期
4. **service A/B**：使用了istio的应用，如Service A/B，的进出网络流量会被proxy接管

> 对于模块的命名方法，本文采用模块对应源码main.go所在包名称命名法。其他istio分析文章有其他命名方法。比如pilot-agent也被称为istio pilot，因为它在Kubernetes上的部署形式为一个叫istio-pilot的deployment。

## pilot-discovery的统一存储模型（Abstract Model）
![](https://mmbiz.qpic.cn/mmbiz_png/KpsG9zYT9aJRzkRoPPc7kU1juU20kaWo8PKaMaSHy8bpyG85NdRUMSdUJEeicArsiaxHic06ecLKQ9UOFIHFUgc2A/?wx_fmt=png)

根据上面官方的pilot-discovery架构图，pilot-discovery有两个输入信息（黄色部分）  

1. 来自istioctl的控制面信息，也就是图中的Rules API，如route rule、virtual service等，这些信息以Kubernetes CRD资源形式保存
2. 来自服务注册中心的服务注册信息，也就是图上的Kubernetes、Mesos、Cloud Foundry等。在Kubernetes环境下包括`pod` 、`service`、`node`、`endpoint`

为了实现istio对不同服务注册中心的支持，如Kubernetes，consul、Cloud Foundry等，pilot-discovery需要对以上两个输入来源的数据有一个统一的存储格式，也就是图中的Abstract Model，这种格式就定义在pilot/pkg/model包下。

举例，下面列表罗列了istio Abstract Model中service的一些成员如何跟根据Kubernetes服务注册信息中的service对象转化得到：


1. `HostName`：`<name>.<namespace>.svc.cluster.local`  
其中`name`和`namespace`分别为Kubernetes service对象的name和所属的namespace。cluster.local为默认domain suffix，可以通过proxy-discovery `discovery`命令的`domain` flag提供自定义值 
2. `Ports`： 对应Kubernetes service的spec.ports。   
3. `Address`: 对应Kubernetes service的spec.ClusterIP。  
4. `ExternalName`: 对应Kubernetes service的spec.ExternalName。  
5. `ServiceAccounts`: 对应Kubernetes service的annotation中key值为alpha.istio.io/kubernetes-serviceaccounts和alpha.istio.io/canonical-serviceaccounts的annotation信息。  
6. `Resolution`: 根据情况可以设置为client side LB、DNS Lb和Passthrough。比如对于ClusterIP类型的Kubernetes service，Resolution设置为client side LB，表示应用发出的请求由sidecar（也就是Envoy）负责做负载均衡，而对于Kubernetes中的headless service则设置为Passthrough。

上面pilot-discovery架构图中的Platform Adaptor负责实现服务注册中心数据到Abstract Model之间的数据转换，在代码里，Platform Adaptor包含两部分：

1. pilot/pkg/serviceregistry/kube/conversion.go里包括一系列将Kubernetes服务注册中心中的`label`、`pod`、`service`、`service port`等Kubernetes资源对象转换为Abstract Model中的对应资源对象的函数
2. pilot/pkg/config/kube/crd/conversion.go里包括将`DestinationRule`等CRD转换为Abstract Model中的`Config`对象的函数

在`pilot/pkg/bootstrap`包下的`Server`结构体代表pilot-discovery，其中包含3个重要成员负责这两类信息的获取、格式转换、以及构建数据变更事件的处理框架：

1. `ConfigStoreCache`
`ConfigStoreCache`对象中embed了`ConfigStore`对象。`ConfigStore`对象利用client-go库从Kubernetes获取route rule、virtual service等CRD形式存在控制面信息，转换为model包下的`Config`对象，对外提供`Get`、`List`、`Create`、`Update、Delete`等CRUD服务。而`ConfigStoreCache`在此基础之上还允许注册控制面信息变更处理函数。
2. `IstioConfigStore`  
IstioConfigStore封装了embed在ConfigStoreCache中的同一个ConfigStore对象。其主要目的是为访问route rule、virtual service等数据提供更加方便的接口。相对于ConfigStore提供的`Get`、`List`、`Create`、`Update、Delete`接口，IstioConfigStore直接提供更为方便的RouteRules、VirtualServices接口。
3. `ServiceController`  
利用client-go库从Kubernetes获取`pod` 、`service`、`node`、`endpoint`，并将这些CRD转换为model包下的Service、ServiceInstance对象。

> 在istio中，使用istioctl配置的VirtualService、DestinationRule等被称为configuration，而从Kubernetes等服务注册中心获取的信息被称为service信息。所以从名称看`ConfigStoreCache`、`IstioConfigStore`负责处理第一类信息，`ServiceController`负责第二类。

## pilot-discovery为Envoy提供的`xds`服务
### 所谓`xds`
基于上面介绍的统一数据存储格式Abstract Model，pilot-discovery为数据面（运行在sidecar中的Envoy等proxy组件）提供控制信息服务，也就是所谓的discovery service或者`xds`服务。这里的x是一个代词，类似云计算里的XaaS可以指代IaaS、PaaS、SaaS等。在istio中，`xds`包括`cds`(cluster discovery service)、`lds`(listener discovery service)、`rds`(route discovery service)、`eds`(endpoint discovery service)，而`ads`(aggregated discovery service)是对这些服务的一个统一封装。  

以上cluster、endpoint、route等概念的详细介绍和实现细节可以参考Envoy在社区推广的data plane api（github.com/envoyproxy/data-plane-api），这里只做简单介绍：

1. endpoint：一个具体的“应用实例”，对应ip和端口号，类似Kubernetes中的一个pod。
2. cluster：一个cluster


## 本文作者

丁轶群博士

谐云科技CTO

2004年作为高级技术顾问加入美国道富银行(浙江)技术中心，负责分布式大型金融系统的设计与研发。2011年开始领导浙江大学开源云计算平台的研发工作，是浙江大学SEL实验室负责人，2013年获得浙江省第一批青年科学家称号，CNCF会员，多次受邀在Cloud Foundry, Docker大会上发表演讲，《Docker：容器与容器云》主要作者之一。

**原创文章，未经允许，不得转载！**
