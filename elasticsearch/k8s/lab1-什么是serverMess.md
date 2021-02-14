# ServiceMesh和Serverless

# 1 前言

今年，ServiceMesh(服务网格)概念在社区里头非常火，有人提出2018年是ServiceMesh年，还有人提出ServiceMesh是下一代的微服务架构基础。作为架构师，如果你现在还不了解ServiceMesh的话，是否感觉有点落伍了？

那么到底什么是ServiceMesh？它诞生的背景是什么？它解决什么问题？企业是否适合引入ServiceMesh？根据近年在一线互联网企业的实践和思考，从个人视角出发，我为大家一一解答这些问题。

# 2 微服务架构的核心技术问题

在业务规模化和研发效能提升等因素的驱动下，从单块应用向微服务架构的转型(如下图所示)，已经成为很多企业(尤其是互联网企业)数字化转型的趋势。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235351852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
在微服务模式下，企业内部服务少则几个到几十个，多则上百个，每个服务一般都以集群方式部署，这时自然产生两个问题(如下图所示)：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235334280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

- 服务发现：服务的消费方(Consumer)如何发现服务的提供方(Provider)？
- 负载均衡：服务的消费方如何以某种负载均衡策略访问集群中的服务提供方实例？
  作为架构师，如果你理解了这两个问题，可以说就理解了微服务架构在技术上的最核心问题。

# 3 三种服务发现模式

服务发现和负载均衡并不是新问题，业界其实已经探索和总结出一些常用的模式，这些模式的核心其实是代理(Proxy，如下图所以)，以及代理在架构中所处的位置，

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235320713.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
在服务消费方和服务提供方之间增加一层代理，由代理负责服务发现和负载均衡功能，消费方通过代理间接访问目标服务。根据代理在架构上所处的位置不同，当前业界主要有三种不同的服务发现模式：

## 3.1 模式一：传统集中式代理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235311272.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
这是最简单和传统做法，在服务消费者和生产者之间，代理作为独立一层集中部署，由独立团队(一般是运维或框架)负责治理和运维。常用的集中式代理有硬件负载均衡器(如F5)，或者软件负载均衡器(如Nginx)，F5(4层负载)+Nginx(7层负载)这种软硬结合两层代理也是业内常见做法，兼顾配置的灵活性(Nginx比F5易于配置)。

这种方式通常在DNS域名服务器的配合下实现服务发现，服务注册(建立服务域名和IP地址之间的映射关系)一般由运维人员在代理上手工配置，服务消费方仅依赖服务域名，这个域名指向代理，由代理解析目标地址并做负载均衡和调用。

国外知名电商网站eBay，虽然体量巨大，但其内部的服务发现机制仍然是基于这种传统的集中代理模式，国内公司如携程，也是采用这种模式。

## 3.2 模式二：客户端嵌入式代理

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020071123525123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
这是很多互联网公司比较流行的一种做法，代理(包括服务发现和负载均衡逻辑)以客户库的形式嵌入在应用程序中。这种模式一般需要独立的服务注册中心组件配合，服务启动时自动注册到注册中心并定期报心跳，客户端代理则发现服务并做负载均衡。

Netflix开源的Eureka(注册中心)[附录1]和Ribbon(客户端代理)[附录2]是这种模式的典型案例，国内阿里开源的Dubbo也是采用这种模式。

## 3.3 模式三：主机独立进程代理

这种做法是上面两种模式的一个折中，代理既不是独立集中部署，也不嵌入在客户应用程序中，而是作为独立进程部署在每一个主机上，一个主机上的多个消费者应用可以共用这个代理，实现服务发现和负载均衡，如下图所示。这个模式一般也需要独立的服务注册中心组件配合，作用同模式二。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235234253.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
Airbnb的SmartStack[附录3]是这种模式早期实践产品，国内公司唯品会对这种模式也有探索和实践。

# 4 三种服务发现模式的比较

上面介绍的三种服务发现模式各有优劣，没有绝对的好坏，可以认为是三种不同的架构风格，在不同的公司都有成功实践。下表总结三种服务发现模式的优劣比较，业界案例和适用场景建议，供架构师选型参考：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235222814.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

# 5 服务网格ServiceMesh

## 5.1 概述

所谓的ServiceMesh，其实本质上就是上面提到的模式三~主机独立进程模式，这个模式其实并不新鲜，业界(国外的Airbnb和国内的唯品会等)早有实践，那么为什么现在这个概念又流行起来了呢？我认为主要原因如下：

- 上述模式一和二有一些固有缺陷，模式一相对比较重，有单点问题和性能问题；模式二则有客户端复杂，支持多语言困难，无法集中治理的问题。模式三是模式一和二的折中，弥补了两者的不足，它是纯分布式的，没有单点问题，性能也OK，应用语言栈无关，可以集中治理。
- 微服务化、多语言和容器化发展的趋势，企业迫切需要一种轻量级的服务发现机制，ServiceMesh正是迎合这种趋势诞生，当然这还和一些大厂(如Google/IBM等)的背后推动有关。

模式三(ServiceMesh)也被形象称为边车(Sidecar)模式，如下图，早期有一些摩托车，除了主驾驶位，还带一个边车位，可以额外坐一个人。在模式三中，业务代码进程(相当于主驾驶)共享一个代理(相当于边车)，代理除了负责服务发现和负载均衡，还负责动态路由、容错限流、监控度量和安全日志等功能，这些功能是具体业务无关的，属于跨横切面关注点(Cross-Cutting Concerns)范畴。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235147665.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 5.2 服务模式演进过程

### 5.2.1 时代1：原始通信时代

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713112445636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
通信需要底层能够传输字节码和电子信号的物理层来完成，在TCP协议出现之前，服务需要自己处理网络通信所面临的丢包、乱序、重试等一系列流控问题，因此服务实现中，除了业务逻辑外，还夹杂着对网络传输问题的处理逻辑。

### 5.2.2 TCP时代

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713111145737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
为了避免每个服务都需要自己实现一套相似的网络传输处理逻辑，TCP协议出现了，它解决了网络传输中通用的流量控制问题，将技术栈下移，从服务的实现中抽离出来，成为操作系统网络层的一部分。

### 5.2.3 时代3：第一代微服务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713111239314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
在TCP出现之后，机器之间的网络通信不再是一个难题，以GFS/BigTable/MapReduce为代表的分布式系统得以蓬勃发展。

这时，分布式系统特有的通信语义又出现了，如熔断策略、负载均衡、服务发现、认证和授权、quota限制、trace和监控等等，于是服务根据业务需求来实现一部分所需的通信语义。

### 5.2.4 时代4：第二代微服务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713111325171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
为了避免每个服务都需要自己实现一套分布式系统通信的语义功能，随着技术的发展，一些面向微服务架构的开发框架出现了，如Twitter的Finagle、Facebook的Proxygen以及Spring Cloud等等，这些框架实现了分布式系统通信需要的各种通用语义功能：如负载均衡和服务发现等，因此一定程度上屏蔽了这些通信细节，使得开发人员使用较少的框架代码就能开发出健壮的分布式系统。

### 5.2.5 时代5：第一代Service Mesh

第二代微服务模式看似完美，但开发人员很快又发现，它也存在一些本质问题：

- 其一，虽然框架本身屏蔽了分布式系统通信的一些通用功能实现细节，但开发者却要花更多精力去掌握和管理复杂的框架本身，在实际应用中，去追踪和解决框架出现的问题也绝非易事；
- 其二，开发框架通常只支持一种或几种特定的语言，回过头来看文章最开始对微服务的定义，一个重要的特性就是语言无关，但那些没有框架支持的语言编写的服务，很难融入面向微服务的架构体系，想因地制宜的用多种语言实现架构体系中的不同模块也很难做到；
- 其三，框架以lib库的形式和服务联编，复杂项目依赖时的库版本兼容问题非常棘手，同时，框架库的升级也无法对服务透明，服务会因为和业务无关的lib库升级而被迫升级。

因此以Linkerd，Envoy，Ngixmesh为代表的代理模式（边车模式）应运而生，这就是第一代Service Mesh，它将分布式服务的通信抽象为单独一层，在这一层中实现负载均衡、服务发现、认证授权、监控追踪、流量控制等分布式系统所需要的功能，作为一个和服务对等的代理服务，和服务部署在一起，接管服务的流量，通过代理之间的通信间接完成服务之间的通信请求，这样上边所说的三个问题也迎刃而解。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713112625152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
如果我们从一个全局视角来看，就会得到如下部署图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713112648894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
如果我们暂时略去服务，只看Service Mesh的单机组件组成的网络：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713112702754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
相信现在，大家已经理解何所谓Service Mesh，也就是服务网格了。它看起来确实就像是一个由若干服务代理所组成的错综复杂的网格。

### 5.2.6 时代6：第二代Service Mesh

第一代Service Mesh由一系列独立运行的单机代理服务构成，为了提供统一的上层运维入口，演化出了集中式的控制面板，所有的单机代理组件通过和控制面板交互进行网络拓扑策略的更新和单机数据的汇报。这就是以Istio为代表的第二代Service Mesh。

在新一代的ServiceMesh架构中(下图上方)，服务的消费方和提供方主机(或者容器)两边都会部署代理SideCar。ServiceMesh比较正式的术语也叫数据面板(DataPlane)，与数据面板对应的还有一个独立部署的控制面板(ControlPlane)，用来集中配置和管理数据面板，也可以对接各种服务发现机制(如K8S服务发现)。术语数据面板和控制面板，估计是偏网络SDN背景的人提出来的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713105609546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
在Service Mesh架构中，给每一个微服务实例部署一个Sidecar Proxy。该Sidecar Proxy负责接管对应服务的入流量和出流量，并将微服务架构中的服务订阅、服务发现、熔断、限流、降级、分布式跟踪等功能从服务中抽离到该Proxy中。

Sidecar以一个独立的进程启动，可以每台宿主机共用同一个Sidecar进程，也可以每个应用独占一个Sidecar进程。所有的服务治理功能，都由Sidecar接管，应用的对外访问仅需要访问Sidecar即可。当该Sidecar在微服务中大量部署时，这些Sidecar节点自然就形成了一个服务网格。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713102411543.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235140202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
上图左下角，每个主机上同时居住了业务逻辑代码(绿色表示)和代理(蓝色表示)，服务之间通过代理发现和调用目标服务，形成服务之间的一种网络状依赖关系，控制面板则可以配置这种依赖调用关系，也可以调拨路由流量。如果我们把主机和业务逻辑剥离，就出现一种网格状架构(上图右下角)，服务网格由此得名。

只看单机代理组件（数据面板）和控制面板的Service Mesh全局部署视图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713113106416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 5.3 两代Service Mesh介绍

第一代Service Mesh的代表为Linkerd和Envoy。Linkerd基于Twitter的Fingle，使用Scala编写，是业界第一个开源的Service Mesh方案，在长期的实际生产环境中获得验证。Envoy底层基于C++，性能上优于使用Scala的Linkerd。同时，Envoy社区成熟度较高，商用稳定版本面世时间也较长。这两个开源实现都是以Sidecar为核心，绝大部分关注点都是如何做好Proxy，并完成一些通用控制面的功能。但是当你在容器中大量部署Sidecar以后，如何管理和控制这些Sidecar本身就是一个不小的挑战。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713105636356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

第二代Service Mesh主要改进集中在更加强大的控制面功能（与之对应的Sidecar Proxy被称之为数据面），典型代表有Istio和Conduit。Istio是Google、IBM和Lyft合作的开源项目，是目前最主流的Service Mesh方案，也是事实上的第二代Service Mesh标准。在Istio中，直接把Envoy作为Sidecar。除了Sidecar，Istio中的控制面组件都是使用Go语言编写。

## 5.4 Istio

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200711235133338.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
Istio[附录4]是Google/IBM等大厂支持和推进的一个ServiceMesh标准化工作组，上图是Istio给出的ServiceMesh参考架构。

Istio专注在控制面板的架构、功能、以及控制面板和数据面板之间API的标准化，它的控制面板功能主要包括：

- Istio-Manager
  负责服务发现，路由分流，熔断限流等配置数据的管理和下发

- Mixer
  负责收集代理上采集的度量数据，进行集中监控

- Istio-Auth
  负责安全控制数据的管理和下发

  提供强大的服务间认证和终端用户认证，使用交互TLS，内置身份和证书管理。可以升级服务网格中的未加密流量，并为运维人员提供基于服务身份而不是网络控制来执行策略的能力。Istio的未来版本将增加细粒度的访问控制和审计，以使用各种访问控制机制（包括基于属性和角色的访问控制以及授权钩子）来控制和监视访问服务、API或资源的访问者。

Istio的设计理念先进，功能也比较强大，加之Google、IBM的影响力让Istio迅速传播，让广大开发者认识到了Istio项目在Service Mesh领域的重要性。但是Istio目前版本也存在了一些不足：

目前的Istio大部分能力与Kubernetes是强关联的。而我们在构建微服务的时候往往是希望服务层与容器层是解耦的，服务层在设计上需要能够对接多种容器层平台。
Istio至今未有稳定版本，截至本文编写时为止，Istio的最新版本为0.8版本，预计在2018年内会发布1.0版本。

其他主要组件：

- Envoy
  扮演Sidecar的功能，协调服务网格中所有服务的出入站流量，并提供服务发现、负载均衡、限流熔断等能力，还可以收集与流量相关的性能指标。

  Envoy[附录5]是目前Istio主力支持的数据面板代理，其它主流代理如nginx/kong等也正在陆续加入这个阵营。kubernetes是目前Isito主力支持的容器云环境。

- Pilot
  负责部署在Service Mesh中的Envoy实例的生命周期管理。本质上是负责流量管理和控制，将流量和基础设施扩展解耦，这是Istio的核心。可以把Pilot看做是管理Sidecar的Sidecar, 但是这个特殊的Sidacar并不承载任何业务流量。Pilot让运维人员通过Pilot指定它们希望流量遵循什么规则，而不是哪些特定的pod/VM应该接收流量。有了Pilot这个组件，我们可以非常容易的实现 A/B 测试和金丝雀Canary测试。

## 5.5 Conduit

我们再来看一下Conduit的实现，下图是Conduit的架构设计图，其中重点由Conduit Data Plane和Conduit Control Plane两部分组成：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713103421604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)
Conduit各方面的设计理念与Istio非常类似，作者使用Rust语言重新编写了Sidecar, 叫做Conduit Data Plane, 控制面则由Go语言编写的Conduit Control Plane接管。从Conduit的架构看，作者号称Conduit吸取了很多Linkerd的教训，比Linkerd更快、更轻、更简单，控制面功能更强。与Istio比较，Conduit的架构一方面比较简单，另一方面对于要解决的问题足够聚焦。

# 6 关于ServiceMesh的建议

目前我本人并不特别看好ServiceMesh，也不是特别建议企业在生产上试水ServiceMesh，主要原因如下：

- ServiceMesh其实并不是什么新东西，本质就是上面提到的服务发现模式三~主机独立进程模式，这个模式很早就有公司在探索和实践，但是一直没有普遍流行起来，说明这个模式也是存在落地挑战的。从表面上看，模式三是模式一和模式二的折中，同时解决了模式一和模式二存在的问题，但是在每个主机上独立部署一个代理进程，是有很大运维管理开销的，一方面是规模化部署的问题(考虑服务很多，机器也很多的场景)；另一方面是如何监控治理的问题，代理挂了怎么办？你的团队是否具备自动化运维和监控的能力？另外开发人员在服务调试的时候，会依赖于这个独立的代理，调试排错比较麻烦，这个问题怎么解决？
- Istio的确做了一些标准化工作，但是没有什么特别的创新，可是说换汤不换药，就是把模式三规范化和包装了一下。透过现象看本质，Google/IBM等行业大厂在背后推Isito/ServiceMesh，背后有一些市场利益诉求考虑，例如Google要推进它的kubernates和公有云生态。
- ServiceMesh在年初声音比较大，最近渐渐安静下来，我听到国内只有一些大厂(华为，新浪微博，蚂蚁金服等)在试水，实际生产级落地的案例聊聊无几。大多数企业对ServiceMesh只是观望，很多架构师对ServiceMesh实际落地都存在疑虑。

所以我的个人建议，对于大部分企业(一般运维和研发能力不是特别强)，采用模式一~集中代理模式就足够了。这个模式比较传统不新鲜，但是在很多一线企业已经切实落地，我甚至认为，除了一些大厂，大部分中小企业的服务发现架构采用的就是集中代理。我本人经历过三家互联网公司，大的有eBay，中等有携程，小的有拍拍贷，都是采用集中式代理模式，而且玩得都很好。我的架构理念很简单，对于生产级应用，不追新，老实采用大部分企业落地过的方案。

模式一的最大好处是集中治理，应用不侵入，语言栈无关，另外因为模式一是集中部署的，不像模式三是分布式部署，所以模式一的运维开销也远小于模式三。对于模式一，大家最大的顾虑是性能和单点问题，其实性能还是OK的，如果架构和容量规划合理的话，实际生产中经过集中代理的性能开销一般可以控制在小于10个ms，eBay和携程等大流量企业的成功实践已经验证了这点。单点问题一般建议采用两层负载结构，例如硬件F5+软件nginx两层负载，F5以主从HA部署，nginx则以集群多实例部署，这种架构兼顾了高可用和配置的灵活性。

另外，模式一还可以和服务注册中心结合，从而降低手工配置的复杂性，实现DevOps研发自助部署，一种方案如下图所示：

![640?wx_fmt=other](https://img-blog.csdnimg.cn/20200711235056263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

服务启动时自动注册到服务注册中心并定期报心跳，Proxy则定期到服务注册中心同步实例。这种方式下，不需要为每个服务申请一个域名，只需一个泛域名即可，消费者访问服务时采用服务名+泛域名即可，整个服务上线流程可以做到DevOps研发自助。目前社区流行的一些开源代理如traefik[附录7]和kong[附录8]等都支持和多种服务注册中心(Consul/Eureka/Etcd/Zookeeper等)进行集成。目前这种方案在拍拍贷有初步成功实践，采用kong[附录7]和自研服务注册中心Radar[附录8]，同时和容器云调度平台配合，实现了研发全自助式发布上线。

# 7 ServiceMesh结论

至此，见证了6个时代的变迁，大家一定清楚了Service Mesh技术到底是什么，以及是如何一步步演化到今天这样一个形态。

现在，我们再回过头来看Buoyant的CEO William Morgan，也就是Service Mesh这个词的发明人，对Service Mesh的定义：
`服务网格是一个基础设施层，用于处理服务间通信。云原生应用有着复杂的服务拓扑，服务网格保证请求在这些拓扑中可靠地穿梭。在实际应用当中，服务网格通常是由一系列轻量级的网络代理组成的，它们与应用程序部署在一起，但对应用程序透明。`

这个定义中，有四个关键词：

- 基础设施层+请求在这些拓扑中可靠穿梭
  这两个词加起来描述了Service Mesh的定位和功能，是不是似曾相识？没错，你一定想到了TCP；
- 网络代理
  这描述了Service Mesh的实现形态；
- 对应用透明
  这描述了Service Mesh的关键特点，正是由于这个特点，Service Mesh能够解决以Spring Cloud为代表的第二代微服务框架所面临的三个本质问题。

1. 服务注册发现和负载均衡是微服务架构在技术上的根本问题，解决的办法是采用代理Proxy。根据代理在架构上的位置不同，服务发现代理一般有三种模式：
   - 模式一：集中式代理
   - 模式二：客户端嵌入式代理
   - 模式三：主机独立进程代理 这三种模式没有绝对的好还之分，只是三种不同的架构风格，各有优劣和适用场景，在不同企业都有成功落地案例。
2. ServiceMesh本质上就是模式三~主机独立进程代理，它结合了模式一和模式二的优势，但是分布式部署运维管理开销大。Istio对ServiceMesh的架构、功能和API进行了标准化。
3. ServiceMesh还在演进中，生产落地仍有挑战，一般企业不建议生产级使用。集中式代理最成熟，对于一般中小企业，建议从集中式代理开始，等达到一定规模和具备一定的研发运维能力，再根据需要考虑其它服务发现模式。
4. 架构师不要盲目追新，在理解微服务架构原理的基础上，可以学习和试点新技术，但是对于生产级应用，应该以成熟稳定，有大规模落地案例作为选型第一准则。

总结一下，Service Mesh具有如下优点：

- 屏蔽分布式系统通信的复杂性（负载均衡、服务发现、认证授权、监控追踪、流量控制等等），服务只用关注业务逻辑；
- 真正的语言无关，服务可以用任何语言编写，只需和Service Mesh通信即可；
- 对应用透明，Service Mesh组件可以单独升级。

当然，Service Mesh目前也面临一些挑战：

- Service Mesh组件以代理模式计算并转发请求，一定程度上会降低通信系统性能，并增加系统资源开销；
- Service Mesh组件接管了网络流量，因此服务的整体稳定性依赖于Service Mesh，同时额外引入的大量Service Mesh服务实例的运维和管理也是一个挑战；

# 8 Serverless

## 8.1 概述

Serverless被翻译为“无服务器架构”，这个概念在2012年时便已经存在，比微服务和Service Mesh的概念出现都要早，但是直到微服务概念大红大紫之后，Serverless才重新又被人们所关注。

Serverless（无服务器架构）并不意味着没有任何服务器去运行代码，Serverless是无需管理服务器，只需要关注代码，而提供者将处理其余部分工作。

“无服务器架构”也可以指部分服务器端逻辑依然由应用程序开发者来编写的应用程序，但与传统架构的不同之处在于，这些逻辑运行在完全由第三方管理，由事件触发的无状态（Stateless）暂存于计算容器内。

对于开发者来说，Serverless架构可以将其服务器端应用程序分解成多个执行不同任务的函数，整个应用分为几个独立、松散耦合的组件，这些组件可以在任何规模上运行。

下图为一种常见的Serverless架构图，所有的服务都以FaaS（函数即服务）的方式对外进行提供。在语言和环境方面，FaaS 函数就是常规的应用程序，例如使用JavaScript、Python以及 Java等语言实现。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713104656773.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

## 8.2 Serverless计算是什么

云计算涌现出很多改变传统IT架构和运维方式的新技术，比如虚拟机、容器、微服务，无论这些技术应用在哪些场景，降低成本、提升效率是云服务永恒的主题。

过去十年来，我们已经把应用和环境中很多通用的部分变成了服务。Serverless的出现，带来了跨越式变革。Serverless把主机管理、操作系统管理、资源分配、扩容，甚至是应用逻辑的全部组件都外包出去，把它们看作某种形式的商品——厂商提供服务，我们掏钱购买。

过去是“构建一个框架运行在一台服务器上，对多个事件进行响应”，Serverless则变为“构建或使用一个微服务或微功能来响应一个事件”，做到当访问时，调入相关资源开始运行，运行完成后，卸载所有开销，真正做到按需按次计费。这是云计算向纵深发展的一种自然而然的过程。

Serverless是一种构建和管理基于微服务架构的完整流程，允许你在服务部署级别而不是服务器部署级别来管理你的应用部署。

它与传统架构的不同之处在于，完全由第三方管理，由事件触发，存在于无状态（Stateless）、暂存（可能只存在于一次调用的过程中）计算容器内。构建无服务器应用程序意味着开发者可以专注在产品代码上，而无须管理和操作云端或本地的服务器或运行时。Serverless真正做到了部署应用无需涉及基础设施的建设，自动构建、部署和启动服务。

国内外的各大云厂商 Amazon、微软、Google、IBM、阿里云、腾讯云、华为云相继推出Serverless产品，Serverless也从概念、愿景逐步走向落地，在各企业、公司应用开来。

## 8.3 理解Serverless技术—FaaS和BaaS

### 8.3.1 概述

Serverless由开发者实现的服务端逻辑运行在无状态的计算容器中，它由事件触发， 完全被第三方管理，其业务层面的状态则被开发者使用的数据库和存储资源所记录。Serverless涵盖了很多技术，分为两类：FaaS和BaaS。

## 8.3.2 FaaS（Function as a Service，函数即服务）

FaaS意在无须自行管理服务器系统或自己的服务器应用程序，即可直接运行后端代码。其中所指的服务器应用程序，是该技术与容器和PaaS（平台即服务）等其他现代化架构最大的差异。

FaaS可以取代一些服务处理服务器（可能是物理计算机，但绝对需要运行某种应用程序），这样不仅不需要自行供应服务器，也不需要全时运行应用程序。

FaaS产品不要求必须使用特定框架或库进行开发。在语言和环境方面，FaaS函数就是常规的应用程序。例如AWS Lambda的函数可以通过Javascript、Python以及任何JVM语言（Java、Clojure、Scala）等实现。然而Lambda函数也可以执行任何捆绑有所需部署构件的进程，因此可以使用任何语言，只要能编译为Unix进程即可。FaaS函数在架构方面确实存在一定的局限，尤其是在状态和执行时间方面。

在迁往FaaS的过程中，唯一需要修改的代码是“主方法/启动”代码，其中可能需要删除顶级消息处理程序的相关代码（“消息监听器接口”的实现），但这可能只需要更改方法签名即可。在FaaS的世界中，代码的其余所有部分（例如向数据库执行写入的代码）无须任何变化。

相比传统系统，部署方法会有较大变化 – 将代码上传至FaaS供应商，其他事情均可由供应商完成。目前这种方式通常意味着需要上传代码的全新定义（例如上传zip或JAR文件），随后调用一个专有API发起更新过程。

FaaS中的函数可以通过供应商定义的事件类型触发。对于亚马逊AWS，此类触发事件可以包括S3（文件）更新、时间（计划任务），以及加入消息总线的消息（例如Kinesis）。通常你的函数需要通过参数指定自己需要绑定到的事件源。

大部分供应商还允许函数作为对传入Http请求的响应来触发，通常这类请求来自某种该类型的API网关（例如AWS API网关、Webtask）。

### 8.3.3 BaaS（Backend as a Service，后端即服务）

BaaS（Backend as a Service，后端即服务）是指我们不再编写或管理所有服务端组件，可以使用领域通用的远程组件（而不是进程内的库）来提供服务。理解BaaS，需要搞清楚它与PaaS的区别。

首先BaaS并非PaaS，它们的区别在于：PaaS需要参与应用的生命周期管理，BaaS则仅仅提供应用依赖的第三方服务。典型的PaaS平台需要提供手段让开发者部署和配置应用，例如自动将应用部署到Tomcat容器中，并管理应用的生命周期。BaaS不包含这些内容，BaaS只以API的方式提供应用依赖的后端服务，例如数据库和对象存储。BaaS可以是公共云服务商提供的，也可以是第三方厂商提供的。其次从功能上讲，BaaS可以看作PaaS的一个子集，即提供第三方依赖组件的部分。

BaaS服务还允许我们依赖其他人已经实现的应用逻辑。对于这点，认证就是一个很好的例子。很多应用都要自己编写实现注册、登录、密码管理等逻辑的代码，而对于不同的应用这些代码往往大同小异。完全可以把这些重复性的工作提取出来，再做成外部服务，而这正是Auth0和Amazon Cognito等产品的目标。它们能实现全面的认证和用户管理，开发团队再也不用自己编写或者管理实现这些功能的代码。

### 8.3.4 无服务器（Serverless）计算如何工作？

与使用虚拟机或一些底层的技术来部署和管理应用程序相比，无服务器计算提供了一种更高级别的抽象。因为它们有不同的抽象和“触发器”的集合。

拿计算来讲，这种抽象有一个特定函数和抽象的触发器，它通常是一个事件。以数据库为例，这种抽象也许是一个表，而触发器相当于表的查询或搜索，或者通过在表中做一些事情而生成的事件。

比如一款手机游戏，允许用户在不同的平台上为全球顶级玩家使用高分数表。当请求此信息时，请求从应用程序到API接口。API接口或许会触发AWS的Lambda函数，或者无服务器函数，这些函数再从数据库表中获取到数据流，返回包含前五名分数的一定格式的数据。

一旦构建完成，应用程序的功能就可以在基于移动和基于 Web 的游戏版本中重用。

这跟设置服务器不同，不是必须要有Amazon EC2实例或服务器，然后等待请求。环境由事件触发，而响应事件所需的逻辑只在响应时执行。这意味着，运行函数的资源只有在函数运行时被创建，产生一种非常高效的方法来构建应用程序。

## 8.4 无服务器（Serverless）适用于哪些场景？

### 8.4.1 概述

在现阶段，Serverless主要应用在以下几个场景

- 首先在Web及移动端服务中，可以整合API网关和Serverles服务构建Web及移动后端，帮助开发者构建可弹性扩展、高可用的移动或 Web后端应用服务。
- 在IoT场景下可高效的处理实时流数据，由设备产生海量的实时信息流数据，通过Serverles服务分类处理并写入后端处理。
- 另外在实时媒体资讯内容处理场景里，用户上传的音视频到对象存储OBS，通过上传事件触发多个函数，分别完成高清转码、音频转码等功能，满足用户对实时性和并发能力的高要求。
- 无服务器计算还适合于任何事件驱动的各种不同的用例，这包括物联网，移动应用，基于网络的应用程序和聊天机器人等。这里简单说两个场景，方便大家思考。

### 8.4.2 场景一：应用负载有显著的波峰波谷

Serverless 应用成功与否的评判标准并不是公司规模的大小，而是其业务背后的具体技术问题，比如业务波峰波谷明显，如何实现削峰填谷。一个公司的业务负载具有波峰波谷时，机器资源要按照峰值需求预估；而在波谷时期机器利用率则明显下降，因为不能进行资源复用而导致浪费。

业界普遍共识是，当自有机器的利用率小于 30%，使用 Serverless 后会有显著的效率提升。对于云服务厂商，在具备了足够多的用户之后，各种波峰波谷叠加后平稳化，聚合之后资源复用性更高。比如，外卖企业负载高峰是在用餐时期，安防行业的负载高峰则是夜间，这是受各个企业业务定位所限的；而对于一个成熟的云服务厂商，如果其平台足够大，用户足够多，是不应该有明显的波峰波谷现象的。

### 8.4.3 场景二：典型用例 - 基于事件的数据处理

视频处理的后端系统，常见功能需求如下：视频转码、抽取数据、人脸识别等，这些均为通用计算任务，可由函数计算执行。

开发者需要自己写出实现逻辑，再将任务按照控制流连接起来，每个任务的具体执行由云厂商来负责。如此，开发变得更便捷，并且构建的系统天然高可用、实时弹性伸缩，用户不需要关心机器层面问题。

## 8.5 Serverless架构优势

- 缩短交付时间
  Serverless架构允许开发人员在极短时间内（几小时、几天）交付新的应用程序，而不是像以前一样需要几个星期或几个月。在新的应用程序中，依赖于第三方API提供服务的例子很多，如认证(OAuth)、社交、地图、人工智能等。
- 增强可伸缩性
  所有人都希望自己开发的应用能够快速获取大量的新增用户，但是当活跃用户快速增长的时候，服务器的压力也会激增。使用Serverless架构的体系不再有上述担忧，可以及时、灵活进行扩展来应对快速增长的活跃用户带来的访问压力。
- 降低成本
  Serverless架构模式可以降低计算能力和人力资源方面的成本，如果不需要服务器，就不用花费时间重新造轮子、风险监测、图像处理，以及基础设施管理，操作成本会直线下降。
- 改善用户体验
  用户通常不太关心基础设施，而更注重于功能和用户体验。Serverless架构允许团队将资源集中在用户体验上。
- 减少延迟及优化地理位置信息
  应用规模能力取决于三个方面：用户数量、所在位置及网络延迟。当应用要面向全国甚至全球用户的时候，通常会产生较高的延迟，从而降低用户体验。在Serverless架构下，供应商在每个用户附近都有节点，大幅度降低了访问延迟，因此所有用户的体验都可以得到提升。

## 8.6 Serverless 的问题

### 8.6.1 概述

对于企业来说，支持Serverless计算的平台可以节省大量时间和成本，同时可以释放员工，让开发者得以开展更有价值的工作，而不是管理基础设施。另一方面可以提高敏捷度，更快速地推出新应用和新服务，进而提高客户满意度。但是Serverless不是完美的，它也存在一些问题，需要慎重应用在生产环境。

### 8.6.2 不适合长时间运行应用

Serverless 在请求到来时才运行。这意味着，当应用不运行的时候就会进入 “休眠状态”，下次当请求来临时，应用将会需要一个启动时间，即冷启动时间。如果你的应用需要一直长期不间断的运行、处理大量的请求，那么你可能就不适合采用 Serverless 架构。如果你通过 CRON 的方式或者 CloudWatch 来定期唤醒应用，又会比较消耗资源。这就需要我们对它做优化，如果频繁调用，这个资源将会常驻内存，第一次冷启之后，就可以一直服务，直到一段时间内没有新的调用请求进来，则会转入“休眠”状态，甚至被回收，从而不消耗任何资源。

### 8.6.3 完全依赖于第三方服务

当你所在的企业云环境已经有大量的基础设施的时候，Serverless 对于你来说，并不是一个好东西。当我们采用某云服务厂商的 Serverless 架构时，我们就和该服务供应商绑定了，那么我们再将服务迁到别的云服务商上就没有那么容易了。

我们需要修改一下系列的底层代码，能采取的应对方案，便是建立隔离层。这意味着，在设计应用的时候，就需要隔离 API 网关、隔离数据库层，考虑到市面上还没有成熟的 ORM 工具，让你既支持Firebase，又支持 DynamoDB等等。这些也将带给我们一些额外的成本，可能带来的问题会比解决的问题多。

### 8.6.4 缺乏调试和开发工具

当我使用 Serverless Framework 的时候，遇到了这样的问题：缺乏调试和开发工具。后来，我发现了 serverless-offline、dynamodb-local 等一系列插件之后，问题有一些改善。然而，对于日志系统来说，这仍然是一个艰巨的挑战。

每次你调试的时候，你需要一遍又一遍地上传代码。而每次上传的时候，你就好像是在部署服务器，并不能总是快速地定位出问题在哪。后来，找了一个类似于 log4j 这样的可以分级别记录日志的 Node.js 库 winston。它可以支持 error、warn、info、verbose、debug、silly 六个不同级别的日志，再结合大数据进行日志分析过滤，才能快速定位问题。

### 8.6.5 构建复杂

Serverless 很便宜，但是这并不意味着它很简单。AWS Lambda的 CloudFormation配置是如此的复杂，并且难以阅读及编写（JSON 格式），虽然CloudFomation提供了Template模板，但想要使用它的话，需要创建一个Stack，在Stack中指定你要使用的Template,然后aws才会按照Template中的定义来创建及初始化资源。

而Serverless Framework的配置更加简单，采用的是 YAML 格式。在部署的时候，Serverless Framework 会根据我们的配置生成 CloudFormation 配置。然而这也并非是一个真正用于生产的配置,真实的应用场景远远比这复杂。

# 9 总结

- Service Mesh
  对于大规模部署微服务，内部服务异构程度高的场景，使用Service Mesh方案是一个不错的选择。Service Mesh实现了业务逻辑和控制的解耦，但是也带来了额外的开销，由于网络中多了一跳，增加了性能的损耗和访问的延迟。同时，由于每个服务都需要部署Sidecar, 这也会使本来就具有一定复杂度的分布式系统变得更加复杂。尤其是在实施初期，对Service Mesh的管理和运维会是一个棘手的问题。因此，当我们选择使用Service Mesh架构的时候，需要对具体的Service Mesh实现方案（例如：Istio）做好充分的技术准备和经验积累工作，方能确保方案的顺利实施。

- Serverless
  在微服务与容器技术火热之后，Serverless（无服务器架构）成为新的热点，无服务器云函数可以让用户无需关心服务器的部署运营，只需开发最核心的业务逻辑，即可实现上线运营，具备分布容灾能力，可以依据负载自动扩缩容，并按照实际调用次数与时长计费。

  使用Serverless架构可以免除所有运维性操作，开发人员可以更加专注于核心业务的开发，实现快速上线和迭代，把握业务发展的节奏。Serverless架构可以认为是对微服务和容器化的一种补充，为用户提供了一种新的选择，来应对复杂多变的需求，尤其适合快速发展的初创型公司。

# 10 附录

Netflix Eureka https://github.com/netflix/eureka

Netflix Ribbon https://github.com/netflix/ribbon

Airbnb SmartStack https://medium.com/airbnb-engineering/smartstack-service-discovery-in-the-cloud-4b8a080de619

Istio https://istio.io/

Envoy https://www.envoyproxy.io/

Traefik https://github.com/containous/traefik

Kong https://github.com/kong/kong

Radar https://github.com/ppdai-incubator/radar

# 11 好文推荐

- [浅谈服务治理、微服务与Service Mesh](https://blog.csdn.net/wabiaozia/article/details/84197308)
  包括SOA、服务治理、Dubbo、Spring Cloud等