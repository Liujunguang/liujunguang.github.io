+++
title = 'Hybridnet'
date = 2025-03-23T23:55:40+08:00
draft = false
+++


## 简介

Alibaba 开源的一款面向混合云场景的 Kubernetes 容器网络解决方案。帮助用户在物理机和虚拟机的异构环境之上，构建一层 Underlay + Overlay 的统一网络平面，并提供丰富的管控运维能力。

![](/images/k8s-cni/datapath.jpeg)

## 原则

1. 确定统一网络模型，降低学习和维护成本，保证稳定的长期演进
2. 屏蔽底层异构基础设施，提升交付落地鲁棒性，降低生产交付、PoC 成本
3. 在统一模型的约束下，既能提供 underlay 高性能直通网络方案，满足网络打通和性能的双重需求，又能支持在某些性能不敏感的场景下提供 overlay 虚拟网络方案
4. 尽量降低对外部环境的依赖，保证数据面的简单
5. 与 Kubernetes 深度集成，提供双栈、IP 保持、IP 固定等高阶 IPAM（IP Address Management）能力，保证用户上云后使用习惯不变。

## 组件

![](/images/k8s-cni/hybridnet_components_arch.png)

### Hybridnet-cni

作为 Hybridnet-daemon 与 kubelet 的中间层适配器，通过 domain socket 与 daemon 作 rpc http 通信。目前只实现了 cni 的 Add、Del 方法。

以 daemonset 的 `initContainer` 部署在每个 Node 上，`实现 kubelet 与 cni 通信的关键`：
1. daemonset pod 的 initContainer 将配置文件目录、Hybridnet-cni 可执行文件目录（/opt/cni/bin）挂载到宿主机
2. 调用 Hybridnet-cni 容器内打包好的 /hybridnet/install-cni.sh 脚本，将 `cni 可执行文件` 和相关配置文件 cp 到挂载好的目录
3. 每当 kubelet 需要调用 cni 时，会去对应目录调用 cni 可执行文件，以启动参数，将 pod 网络配置输入
4. Hybridnet-cni 通过 `skel.PluginMain` -> ioutil.ReadAll(os.stdin) 读取启动参数，调用 Add/Del 相关接口，rpc 调用 deamon，实现网络配置。

#### code 分析

```go
// 来自 ContainerNetworking/cni，将服务注册为 kubernetes cni，方便后续 kubelet 与 cni 的统一调用和配置参数解析
main -> skel.PluginMain(cmdAdd, cmdCheck, cmdDel)

// 加载配置文件，主要来自镜像打包时，注入的配置文件，如: 00-hybridnet.conflist，与 Hybridnet-daemon 的 domain socket 配置，默认：/run/cni/hybridnet.sock
LoadNetConf()

// 添加 Pod 请求
cmdAdd() -> rpc http -> hybridnet-daemon

// 删除 Pod 请求
cmdDell -> 同 cmdAdd
```

### Hybridnet-daemon

控制 Node 上的 iptables、policy routes 等网络配置数据。

以 daemonset 方式部署在每个 Node 上。Hybridnet-daemon `实现修改宿主机网络配置的关键`：
1. 容器提权
```yaml
securityContext:  
  runAsUser: 0  
  privileged: true
```
2. 挂载宿主机网络配置目录
```yaml
volumes:
  - name: host-modules  
  hostPath:  
    path: /lib/modules
  - name: host-run-cni  
  hostPath:  
    path: /run/cni
  - name: host-netns-dir  
  hostPath:  
    path: /var/run/netns
...
```

#### code 分析

```go
// 系统初始化，配置允许 ip 转发、ARP 等
initSysctl() ->

// k8s client-go
NewManager() -> // 用于后面作 CRDs 资源同步（Subnet、IPInstance、nodeInfo 等）

// daemon 核心逻辑，类似 kubelet 实现，初始化各种资源 Manager，负责后面的资源同步循环
NewCtrlHub()
-> RouterManager // ip route
-> NeighManager // ip neigh
-> IPtablesManager // iptables ipv4 & ipv6
-> AddrManager // 缓存目标网卡名 -> subnet 的 ip 映射关系。代码注释里说，因为有的物理路由、物理交换机，会检测 arp 发送方的请求，如果 sender 和 target 的 ip 不在同一个 subnet，arp 请求会被自动丢弃
-> bgp.NewManager // 管理 bgp，link bgp 网卡，启动 bgp server
-> NewNodeIPCache // 程序内缓存：node ip -> MAC，负责 VxLan 隧道的 VTEP（端点）

// 启动所有同步循环
go Run()
-> // 启动服务健康检查端口；
-> // 注册 IPInstance 资源索引；
-> // 启动各资源同步主循环；
Subnet
IPInstance
NodeInfo
-> // 监听网卡 UP 事件，并重新初始化 router 和 neigh 配置；
-> // 监听 VxLan 网卡事件；
-> // 启动 iptables 主同步循环

// rpc server，与 cni 的 domain socket server，只实现了 /add 和 /del
RunServer()
```

### Hybridnet-manager

监听 Pod 的创建/删除，并为之分配/删除 IP。
以 deployments 的方式部署

#### code 分析

```go
// k8s client-go
NewManager() -> 用于后面作资源同步

// 注册 Network k8s 资源索引
InitINdexers

// 多实例 leader 选举机制
<- mgr.Elected

// 启动主流程
networking.RegisterToManager
-> IPAMManager // 负责管理 IPAM（IP Address Manager)
-> PodIPCache // 频繁调用 api-server 的模块，一般都需要有相应的本地缓存
-> IPAMStore // 封装 IPInstance 的 CRD 数据操作
-> 启动 k8s 各资源同步主循环
-> 
IPAM
IPInstance
Node
Pod
NetworkStatus
SubnetStatus
Quota
```

### Hybridnet-webhook

负责校验与调度。
以 deployments 的方式部署

#### code 分析

```go
// 向 k8s webhook 机制，注册处理 handler
mgr.GetWebhookServer.Register
-> /validate // 校验 Pod、Network 等资源配置

-> /mutate // 为 pod 匹配 Toleration 调度，防止 host-network 的 pod 被 Hybridnet 影响
// parse network config，处理调度优先级
// 为 pod 添加 annotation
```

### 其它

除了 Hybridnet 自己的组件，还用到了 `felix` 作 NetworkPolicy，当 Node 数量超过 50 个时，还用到了 `calico-typha`， 通过减少每个节点对数据存储的影响来扩大规模。在数据存储和 Felix 实例之间作为守护进程运行。

## CRD 资源

1. 每个 Network 至少需要一个 Subnet 才能被使用
2. StatefulSet 类型 Pod 默认开启 IP 保持
3. 通过在 Pod 上添加 annotation 配置，可指定 Pod 被分配到哪个 Subnet
4. Subnet 对象配置： `spec.config.private` 为 true，实现只有 Subnet 中 Pod 可以使用该网段 IP

### Network

通过 `spec.nodeSelector` 与相关 Node 关联，构成一个 `网络域`，代表一组具有“相同网络性质”的节点（如：同 VLAN、同网关、同二层网络）。

### Subnet

表示网络域中 node 的可用 `网段`。
可设置 gateway、cidr、start、end、reservedIps（保留 ip 列表）、excludeIps（排除 ip 列表）

### IPInstance

`docs/crd.md` 中简单介绍了下：一个 IPInstance 代表，由 Hybridnet 分配给 pod 的实际 ip，IPInstance 虽然是 CRD，但只用作监控，并不能手动配置。
和 Network、Subnet 不同，它们是集群层面的，IPInstance 与 namespace 绑定，是 namespace 级别，与绑定的 Pod 处在同一个 ns 下。

文档中没有详细介绍，根据大致过了一遍代码，主要在自己的资源同步循环中，更新 IP、MAC、router、neigh 等信息，并与 Manger 通信。
本地也维护了各个 pod 和自身节点的网络资源。

## 特性

- 基于 Kubernetes CRD
- 支持 IPv4 / IPv6 双栈
- 复合网络架构，Overlay 仅支持 `VXLAN`，Underlay 支持 `VLAN` 和 `BGP`
- 高级 IPAM
- 与其他网络组件等高兼容性（如：kube-proxy, cilium）

## NetworkPolicy 实现

Kubernetes 的 NetworkPolicy 实现，Hybridnet 是通过内置开源的 calico/felix(v3.20.2) 实现的。
通过 Hybridnet `Dockerfile` 也能看到：

```text
ENV FELIX_GIT_BRANCH=v3.20.2  
ENV FELIX_GIT_COMMIT=ab06c3940caa8ac201f85c1313b2d72d724409d2
...
RUN cd /go/src/github.com/projectcalico/ && \  
    git clone -b ${FELIX_GIT_BRANCH} --depth 1 https://github.com/projectcalico/felix.git && \  
    cd felix && [ "`git rev-parse HEAD`" = "${FELIX_GIT_COMMIT}" ]  
COPY policy/felix /hybridnet_patch  
RUN cd /go/src/github.com/projectcalico/felix && git apply /hybridnet_patch/*.patch  
RUN cd /go/src/github.com/projectcalico/felix && \  
    export GOCACHE=/tmp && \  
    export GO111MODULE=on && \  
    export GOARCH=${ARCH} && \  
    export CGO_ENABLED=0 && \  
    export GOOS=${OS} && \  
    go build -v -o bin/calico-felix -v -ldflags \  
    "-X github.com/projectcalico/felix/buildinfo.GitVersion=${FELIX_GIT_BRANCH} \  
    -X github.com/projectcalico/felix/buildinfo.BuildDate=$(date -u +'%FT%T%z') \  
    -X github.com/projectcalico/felix/buildinfo.GitRevision=${FELIX_GIT_COMMIT} \  
    -B 0x${FELIX_GIT_COMMIT}" "github.com/projectcalico/felix/cmd/calico-felix" && \  
    chmod +x /go/src/github.com/projectcalico/felix/bin/calico-felix
...
...
COPY --from=calico-builder /go/src/github.com/projectcalico/felix/bin/calico-felix /hybridnet/calico-felix  
COPY --from=calico-builder /go/src/github.com/projectcalico/typha/bin/calico-typha /hybridnet/calico-typha
```

然后在 `daemonsets.yaml` 中，容器 `felix` 通过运行 `policyinit.sh` 脚本，以环境变量的形式，控制 felix 运行。

## Hybridnet 对比 Calico

- Calico 只支持 bgp 网络架构
- Hybridnet 除了 bgp 实现 Underlay 外，还提供 Underlay 网络下的 vLan（二层），Overlay 网络下的 VXLAN（三层封包）

[Calico](https://www.projectcalico.org/) is an open-source networking and security solution for Kubernetes with good  
performance and security policy.  

Both of Hybridnet and Calico provide an end-to-end IP network that interconnects the pod in a scale-out or cloud environment,  
by building an *interconnect fabric* to provide the physical networking layer on which Calico or Hybridnet operates.  

The main difference between Hybridnet and Calico is the implementation of interconnect fabric in underlay network. Calico  
uses [bgp-only interconnect fabrics](https://docs.projectcalico.org/reference/architecture/design/l3-interconnect-fabric#bgp-only-interconnect-fabrics)  
while Hybridnet offers a vlan interconnect fabric. This matters when we need the ability of static ip. Once a pod needs to  
retain its ip after being recreated, as we never want a pod to be fixed on only one node, a seperate single-ip route  
(/32 or /128) will appear in the bgp fabric unavoidably, which requires a very large route table size and might bring  
a stability risk into the whole network environment. This will never happen for a vlan fabric, because we always need  
just one static route configured manually on the psychical switch for every subnet, which also brings extra operating  
costs.

## 参考

[github](https://github.com/alibaba/hybridnet)
[wiki--一键安装](https://github.com/alibaba/hybridnet/wiki/%E4%B8%80%E9%94%AE%E5%AE%89%E8%A3%85)
[一段内部模拟采访](https://www.cnblogs.com/alisystemsoftware/p/15950917.html)
