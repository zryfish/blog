---
title: "Kubernetes多集群"
date: 2020-06-28T20:22:01+08:00
draft: true
abstract: ddddd
---

# 多集群使用场景

随着容器的普及和Kubernetes的日渐成熟，企业内部运行多个Kubernetes集群已变得颇为常见。概括起来，多个集群的使用场景主要有以下几种:

- **高可用**  
可以将业务负载分布在多个集群上，使用一个全局的VIP或者DNS域名将请求发送到对应的后端集群，当一个集群发生故障无法处理请求时，将VIP或者DNS记录切换健康的集群。

- **低延迟**  
在多个区域部署集群，将用户请求转向距离最近的集群处理，以此来最大限度减少网络带来的延迟。举个例子，假如在北京，上海，广州三地部署了三个K8s集群，对于广东的用户就将请求转发到所在广州的集群处理，这样可以减少地理距离带来的网络延迟，最大限度地实现各地一致的用户体验。

- **故障隔离**  
通常来说，多个小规模的集群比一个大规模的集群更容易隔离故障。当集群发生诸如断电、网络故障、资源不足引起的连锁反应等问题时，使用多个集群可以将故障隔离在特定的集群，不会向其它集群传播。

- **业务隔离**  
虽然 Kubernetes 里提供了 namespace 来做应用的隔离，但是这只是逻辑上的隔离，不同 namespace 之间网络互通，而且还是存在资源抢占的问题。要想实现更进一步的隔离需要额外设置诸如网络隔离策略，资源限额等。多集群可以在物理上实现彻底隔离，安全性和可靠性相比使用 namespace 隔离更高。例如企业内部不同部门部署各自独立的集群、使用多个集群来分别部署开发/测试/生成环境等。

- **避免厂商锁定**  
Kubernetes 已经成容器编排领域的事实标准，在不同云服务商上部署集群避免将鸡蛋都放在一个篮子里，可以随时迁移业务，在不同集群间伸缩。缺点是成本增加，考虑到不同厂商提供的K8s服务对应的存储、网络接口有差异，业务迁移也不是轻而易举。

# 多集群部署

上面说到了多集群的主要使用场景，多集群可以解决很多问题，但是它本身带来的一个问题就是运维复杂性。对于单个集群来说，部署、更新应用都非常简单直白，直接更新集群上的 yaml 即可。多个集群下，虽然可以挨个集群更新，但如何保证不同集群的应用负载状态一致？如何在不同集群间做服务发现？如何做集群间的负载均衡？社区给出的答案是联邦集群(Federation)。

## Federation v1

![Kubernetes Federation v1](../images/federation-v1.png)

Federation 方案经历了两个版本，最早的 v1 版本已经废弃。在 v1 版本中，整体架构与 Kubernetes 自身的架构非常相似，在管理的各个集群之前引入了 Federated APIServer 用来接收创建多集群部署的请求，在控制平面的 Federation Controller Manager 负责将对应的负载创建到各个集群上。


```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: nginx-us
  annotations:
    federation.kubernetes.io/replica-set-preferences: |
        {
            "rebalance": true,
            "clusters": {
                "us-east1-b": {
                    "minReplicas": 2,
                    "maxReplicas": 4,
                    "weight": 1
                },
                "us-central1-b": {
                    "minReplicas": 2,
                    "maxReplicas": 4,
                    "weight": 1
                }
            }
        }
spec:
  replicas: 4
  template:
    metadata:
      labels:
        region: nginx-us
    spec:
      containers:
        - name: nginx
          image: nginx:1.10
```

在API层面上，联邦资源的调度通过 annotation 实现，最大程度地保持与原有Kubernetes API的兼容，这样的好处是可以复用现有的代码，用户已有的部署文件也不需要做太大改动即可迁移过来。但这也是制约 Federation 进一步的发展，无法很好地对 API 进行演进，同时对于每一种联邦资源，需要有对应的 Controller 来实现多集群的调度，早期的 Federation 只支持有限的几种资源类型。


## Federation v2

![Kubefed](../images/federation-v2.png)

社区在 v1 的基础上发展出了 Federation v2，即 Kubefed。Kubefed 使用 Kubernetes 已经相对成熟的 CRD 定义一套自己的 API 规范，抛弃了之前使用的 annotation 方式。架构上也较之前有很大改变，放弃了之前需要独立部署的 Federated APIServer/ETCD，Kubefed 的控制平面采用了流行的 CRD + Controller 的实现方式，可以直接安装在现有的 Kubernetes 集群上，无需额外的部署。

v2 定义的资源类型主要包含4种：
- **Cluster Configuration**: 定义了 Member Cluster 加入到控制平面时所需要用到的注册信息，包含集群的名称， APIServer 地址以及创建部署时需要用到的凭证。

- **Type Configuration**: 定义了 Federation 可以处理的资源对象。每个 Type Configuration 即是一个 CRD 对象，包含下面三个配置项
    * **Template** Template 包含了要处理的资源对象，如果处理的对象在即将部署的集群上没有相应的定义，则会创建失败。下面例子中的FederatedDeployment，template 包含了创建 Deployment 对象所需要的全部信息。
    * **Placement** 定义了该资源对象将要创建到的集群名称，可以使用 `clusters` 和 `clusterSelector` 两种方式。
    * **Override** 顾名思义，Override 可以根据不同集群覆盖 Template 中的内容，做个性化配置。下面例子中，模板定义的副本数为1，Override 里覆盖了 gondor 集群的副本数，当部署到gondor集群时不再时模板中的1副本，而是4副本。Override 实现了 [jsonpatch](http://jsonpatch.com)(http://jsonpatch.com) 的一个子集，理论上 template 里的所有内容都可以被覆盖。


    ```yaml
    apiVersion: types.kubefed.io/v1beta1
    kind: FederatedDeployment
    metadata:
      labels:
        app: nginx
    name: nginx
    namespace: jeff
    spec:
      overrides:
      - clusterName: gondor
        clusterOverrides:
        - path: /spec/replicas
          value: 4
      placement:
        clusters:
        - name: gondor
        - name: rohan
        - name: shire
      template:
        metadata:
          labels:
            app: nginx
        namespace: jeff
        spec:
          replicas: 1
          template:
            metadata:
              labels:
                app: nginx
            spec:
              containers:
              - image: nginx:latest
                name: nginx
    ```
- **Schedule**: 主要定义应用在集群间的分布，目前主要涉及到 ReplicaSet 和 Deployment。可以通过 Schedule 定义负载在集群中的最大副本和最小副本数，这部分是将之前v1中通过annotation调度的方式独立出来。

- **MultiClusterDNS**:  MultiClusterDNS 实现了多个集群间的服务发现。多集群下的服务发现相比单个集群下要复杂许多，kubefed 里使用了 ServiceDNSRecord、IngressDNSRecord、DNSEndpoint 对象来处理多集群的服务发现，需要配合 DNS 使用。

总体来说，Kubefed 解决了 v1 存在的许多问题，使用 CRD 的方式很大程度地保证了联邦资源的可扩展性，基本上 Kubernetes 所有的资源都可以实现多集群部署，包含用户自定义的 CRD 资源。

Kubefed目前也存在一些值得关注的问题,
- **单点问题** Kubefed的控制平面即是 CRD + Controller 的实现方式，虽然 controller 本身可以实现高可用，但如果其运行所在的Kubernetes无法使用，则整个控制平面即会失效。这点上在社区中也有[讨论](https://github.com/kubernetes-sigs/kubefed/issues/636)(https://github.com/kubernetes-sigs/kubefed/issues/636)，Kubefed 目前使用的是一种 Push reconcile 的方式，当联邦资源创建时，由控制平面上对应的 Controller 将资源对象的发送对应的集群上，至于资源之后怎么处理那是 Member Cluster 自己的事情了，和控制平面无关。所以当 Kubefed 控制平面不可用时，不影响已经存在的应用负载。
- **成熟度** Kubefed 社区不如 Kubernetes 社区活跃，版本迭代的周期较长，目前很多功能仍处在beta阶段。
- **过于抽象** Kubefed 使用 Type Configuration 来定义需要管理的资源，不同的 Type Configuration 仅是 Template 不同，好处是对应处理逻辑可以统一，便于快速实现，Kubefed里Type Configuration资源对应的Controller都是[模板化的](https://github.com/kubernetes-sigs/kubefed/blob/master/pkg/controller/federatedtypeconfig/controller.go)(https://github.com/kubernetes-sigs/kubefed/blob/master/pkg/controller/federatedtypeconfig/controller.go)。但是缺点也很明显，不能针对特殊的 Type 实现个性化的功能。例如 FederatedDeployment 对象，对于 Kubefed 来说仅需要根据 Template 和 Override 生成对应的 Deployment 对象，然后创建到 Placement 指定的集群，至于 Deployment 在集群上是否生成了对应的 Pod，状态是否正常则不能反映到 FederatedDeployment 上，只能去对应的集群上查看。社区目前也意识到这点，正在积极的解决，已经有对应的[提案](https://github.com/kubernetes-sigs/kubefed/pull/1237)(https://github.com/kubernetes-sigs/kubefed/pull/1237)


## KubeSphere 多集群

上面提到的 Federation 是社区提出的解决多集群部署问题的方案，可以通过将资源联邦化来实现多集群的部署。对于很多用户来说，多集群的联合部署其实并不是刚需，更需要的在一处能够同时管理多个集群的资源即可。KubeSphere v3.0 支持了多集群管理的功能，实现了资源独立管理和联邦部署的功能。KubeSphere 多集群参考了社区的 Federation 方案，在 Federation 的基础上实现了管理多个集群的功能。

<!-- ks 多集群架构图 -->

这里有几点值得注意：
- KubeSphere 的管理集群 Host Cluster 独立于其所管理的成员集群，Member Cluster并不知道 Host Cluster 存在，这点和 Federation 的设计思想一致。这样做的好处是当控制平面发生故障时不会影响到成员集群，已经部署的负载仍然可以正常运行，不会受到影响。
- KubeSphere Host Cluster 实现了对成员集群请求的转发。当用户请求具体成员集群的API时，请求会发往 Host Cluster，由 Host Cluster 进行认证，认证通过后发往具体的成员集群。 Host Cluster同时需要实现各个成员集群用户和权限信息的同步(使用Kubefed同步)，但实际的鉴权工作仍然下放到各个集群中去做。这样的一个思路也是为了实现上面提到的，当 Host Cluster 发生故障时不影响各个集群的正常运行。Host Cluster 是一个资源的协调者，不是一个独裁者。
- API 层面上，尽可能兼容之前版本的API。在多集群环境下，API请求路径中以 `/apis/clusters/{cluster}` 开头的请求会被转发到 `{cluster}` 集群，并且去除 `/clusters/{cluster}`，这样的好处对应的集群收到请求和其它的请求并无任何区别，无需做额外的工作。举个例子：
  ```
  GET /api/clusters/gondor/v1/namespaces/default
  GET /kapis/clusters/gondor/servicemesh.kubesphere.io/v1alpha1/namespaces/default/strategies/canary
  ```
  会被转发到名称为 gondor 的集群，并且请求会被处理为 
  ```
  GET /api/v1/namespaces/default
  GET /kapis/servicemesh.kubesphere.io/v1alpha1/namespaces/default/strategies/canary
  ```

<!-- 多集群demo视频 -->

## 总结
多集群问题远比想象的要复杂，从社区 Federation 方案就能看出，经历了前后两个版本但是时至今日还未发布正式版。软件领域有句经典的话，No Silver Bullet，Kubefed、KubeSphere等多集群工具并不能也不可能解决多集群的所有问题，还是需要根据具体业务场景选择合适自己的。相信随着时间的推移，这些工具也会变得更加成熟，届时可以覆盖到更多的使用场景。

----
参考资料:  
1. Kubefed https://github.com/kubernetes-sigs/kubefed
2. KubeSphere https://kubesphere.com.cn/
3. Kubernetes Federation Evolution https://kubernetes.io/blog/2018/12/12/kubernetes-federation-evolution/