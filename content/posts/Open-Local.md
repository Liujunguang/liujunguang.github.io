+++
title = 'Open Local'
date = 2025-03-23T23:59:45+08:00
draft = false
+++


[github](https://github.com/alibaba/open-local)

## 简介

由多个组件构成的 **本地磁盘管理系统**，解决 Kubernetes 本地存储问题，让本地存储像集中存储一样简单。

### 架构

![](/images/k8s-csi/open_local_arch.png)


### 前置知识

[LVM](LVM.md)

LVM 的创建，大致经过：

1. pvcreate 创建新物理卷
2. vgcreate 创建新卷组
3. lvcreate 创建新逻辑卷

之后就可以像使用正常物理卷那样 mount 到目标目录下。

## 组件

Open-Local 包含四大类组件：

- Scheduler-Extender: 作为 Kubernetes Scheduler 的扩展组件，通过 Extender 方式实现，新增本地存储调度算法  
- CSI: 按照 [CSI(Container Storage Interface)](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/) 标准实现本地磁盘管理能力  
- Agent: 运行在集群中的每个节点，根据配置清单初始化存储设备，并通过上报集群中本地存储设备信息以供 Scheduler-Extender 决策调度  
- Controller: 获取集群存储初始化配置，并向运行在各个节点的 Agent 下发详细的配置清单

### agent

- Informer 机制同步 NodeLocalStorage 资源，用 k8s 的 `RateLimitingInterface` 管理同步循环队列
- 根据 nls（即 `NodeLocalStorage`，下同） 配置的 device 模块，创建 `Device（独占盘）` ->  Physical Volume、vgroup，通过 `K8sMounter.Mount` 挂载卷， k8s PV/PVC 机制的原理，即向用户屏蔽底层存储实现细节
- `Discover()` 周期性更新 nls 资源 status，通过 kubelet 上报
- 与 `spdk server` （如果部署）交互

#### code 分析

```go
// 解析 flag 配置参数
BuildConfigFromFlags

// 创建 k8s client-go 实例，使用的官方 client-go 包，只在包装 EventSinkImpl 对象时用到
kubernetes.NewForConfig(cfg)

// 也是创建 k8s client 实例，但是经过本地包装，实现 Rate limiter、QPS、Burst 等流控配置，后面的同步 nls 资源、更新 nls status 上报，都是通过这个 localClient
clientset.NewForConfig(cfg)

// 如名，实现 snapshot client 实例，定时监测、扩展 snapshot，扫描 vgroup、lvgroup，并根据情况扩展 snapshot lv
snapshot.NewForConfig(cfg)

// 创建 k8s Informer Factory，并开启同步
localinformers.NewSharedInformerFactory(localClient, time.Second*30)
-> localInformerFactory.Start()

// 创建 agent 实例
controller.NewAgent
// 配置 nls 的 Add、Update 事件处理方法 handleNLS，负责处理 agent nls 的主流程，即将 initResourceKey 事件压入 RateLimitingQueue 队列
-> localInformerFactory.Csi().V1alpha1().NodeLocalStorages()
-> AddFunc/UpdateFunc: handleNLS

// 启动 agent 主流程
agent.Run
-> discovery.NewDiscoverer // 负责 agent 资源同步，nls、vg、lv、snap shot 等
discoverer.Discover // 启动定时同步循环
-> c.workqueue.Add(initResourceKey) // agent 首次启动时的资源初始化，压入 k8s 包的 RateLimitingQueue 队列，然后在触发时，根据实时的 nls 配置，初始化创建 pv、vg，完成 Device 独占盘，并通过 k8s mounter 完成挂载到 pod 中容器
d.K8sMounter.Mount

// 启动 agent 循环队列，定时从 RateLimitingQueue 出队执行，这里是整个 agent 真正的主流程，完成 nls 到 device 独占盘的具体实现
go c.runWorker(discoverer)
-> discovery.InitResource()
-> getNodeLocalStorage
-> createVG
-> K8sMounter.Mount

// 利用上面创建的 snaphot client，定时检查、扩容 snap shot lv
go discoverer.ExpandSnapshotLVIfNeeded
```


### csi

是 CSI Driver，负责实现 CSI 接口：
NodePublishVolume（将 Volume 挂载到容器内的目标路径）
NodeUnpublishVolume（从容器卸载 Volume）

#### Kubernets CSI 介绍

根据 k8s CSI 实现相关接口：

- `Identity`: 身份服务，主要负责向 k8s 提供 CSI 插件的名称和可选功能等，必须实现
- `ControllerServer`: 控制器服务，负责大部分 CSI 核心逻辑
- `NodeServer`：节点服务，负责实现 Volume 的挂载/卸载

k8s CSI 的三大阶段：

- **Provisioning and Deleting**: 创建/删除。需要实现 `CreateVolume` 和 `DeleteVolume`
- **Attaching and Detaching**: 挂载/卸载。需要实现 `ControllerPublishVolume` 和 `ControllerUnPublishVolume`
- **Mount 和 Umount**: 当一个 Pod 被调度到某个 Node 上，kubelet 会根据前两个阶段返回到结果创建 Pod，会把已 Attaching 的本地块设备以目录形式挂载到 Pod 中，或从 Pod 中卸载。

Identity Server

```go
// IdentityServer is the server API for Identity service.
type IdentityServer interface{
	GetPluginInfo(context.Context, *GetPluginInfoRequest) (*GetPluginInfoResponse, error)
	GetPluginCapabilities(context.Context, *GetPluginCapabilitiesRequest) (*GetPluginCapabilitiesResponse, error)
	Probe(context.Context, *ProbeRequest) (*ProbeResponse, error)
}
```

Node Server

```go
// NodeServer is the server API for Node service.
type NodeServer interface{
	NodeStageVolume(context.Context, *NodeStageVolumeRequest) (*NodeStageVolumeResponse, error)
	NodeUnstageVolume(context.Context, *NodeUnstageVolumeRequest) (*NodeUnstageVolumeResponse, error)
	NodePublishVolume(context.Context, *NodePublishVolumeRequest) (*NodePublishVolumeResponse, error)
	NodeUnpublishVolume(context.Context, *NodeUnpublishVolumeRequest) (*NodeUnpublishVolumeResponse, error)
	NodeGetVolumeStats(context.Context, *NodeGetVolumeStatsRequest) (*NodeGetVolumeStatsResponse, error)
	NodeExpandVolume(context.Context, *NodeExpandVolumeRequest) (*NodeExpandVolumeResponse, error)
	NodeGetCapabilities(context.Context, *NodeGetCapabilitiesRequest) (*NodeGetCapabilitiesResponse, error)
	NodeGetInfo(context.Context, *NodeGetInfoRequest) (*NodeGetInfoResponse, error)
}
```

> 其中 Volume 的挂载被分成了 NodeStageVolume 和 NodePublishVolume 两个阶段。NodeStageVolume 接口主要是针对块存储类型的 CSI 插件而提供的。块设备在 “Attach” 阶段被附着在 Node 上后，需要挂载至 Pod 对应目录上，但因为块设备在 linux 上只能 mount 一次，而在 kubernetes volume 的使用场景中，一个 volume 可能被挂载进同一个 Node 上的多个 Pod 实例中，所以这里提供了 NodeStageVolume 这个接口，使用这个接口把块设备格式化后先挂载至 Node 上的一个临时全局目录，然后再调用 NodePublishVolume 使用 linux 中的 bind mount 技术把这个全局目录挂载进 Pod 中对应的目录上。
> 
> NodeUnstageVolume 和 NodeUnpublishVolume 正是 volume 卸载阶段所分别对应的上述两个流程。
> 
> 当然，如果是非块存储类型的 CSI 插件，也就不必实现 NodeStageVolume 和 NodeUnstageVolume 这两个接口了。

Controller Server

```go
// ControllerServer is the server API for Controller service.
type ControllerServer interface{
	CreateVolume(context.Context, *CreateVolumeRequest) (*CreateVolumeResponse, error)
	DeleteVolume(context.Context, *DeleteVolumeRequest) (*DeleteVolumeResponse, error)
	ControllerPublishVolume(context.Context, *ControllerPublishVolumeRequest) (*ControllerPublishVolumeResponse, error)
	ControllerUnpublishVolume(context.Context, *ControllerUnpublishVolumeRequest) (*ControllerUnpublishVolumeResponse, error)
	ValidateVolumeCapabilities(context.Context, *ValidateVolumeCapabilitiesRequest) (*ValidateVolumeCapabilitiesResponse, error)
	ListVolumes(context.Context, *ListVolumesRequest) (*ListVolumesResponse, error)
	GetCapacity(context.Context, *GetCapacityRequest) (*GetCapacityResponse, error)
	ControllerGetCapabilities(context.Context, *ControllerGetCapabilitiesRequest) (*ControllerGetCapabilitiesResponse, error)
	CreateSnapshot(context.Context, *CreateSnapshotRequest) (*CreateSnapshotResponse, error)
	DeleteSnapshot(context.Context, *DeleteSnapshotRequest) (*DeleteSnapshotResponse, error)
	ListSnapshots(context.Context, *ListSnapshotsRequest) (*ListSnapshotsResponse, error)
	ControllerExpandVolume(context.Context, *ControllerExpandVolumeRequest) (*ControllerExpandVolumeResponse, error)
	ControllerGetVolume(context.Context, *ControllerGetVolumeRequest) (*ControllerGetVolumeResponse, error)
}
```

#### gRPC server

lvmServer
ControllerServer
NodeServer

是通过 k8s 调度 storage 存储的端点。

内部各接口方法，最终调用诸如 lvcreate、lvremove、vgcreate 等 LVM 工具，并根据配置，从 snapshot 恢复数据。
根据代码，发现甚至支持从 S3 恢复数据，可以支持在集群中部署 MinIO，实现 S3 对象存储，用来存放 snap shot。

#### code 分析

```go
// 也有 kubeClient 和 localClient 的区别，localClient 和 agent 组件一样，带限流。
// localClient 用来同步 nls 资源
// kubeClient 主要用来同步 k8s 其它资源

// 提供一套单独的 rpc 接口，用来操作 node lv 等相关资源
go lvmserver.Start

// 示例化 CSIPlugin
csi.NewDriver
-> // CSIPlugin本身作为 “Identity Server”
-> newNodeServer // 创建 "node server"
-> newControllerServer // 创建 "controller server"
// 创建 CSI 三套接口，并分别实现接口方法

// 启动 csi 组件主流程
driver.Run
-> net.Listen(scheme, addr) // 启动本地 net 监听
-> grpc.NewServer // 启动本地 rpc server
-> csi.RegisterXXXServer // 调 k8s csi client-go 注册 identity、node、controller server
```

### controller

通过 nlsc 更新 nls 资源，即 nls 资源仅被 controller 控制。这样 agent 组件功能就很简单，仅做上报。  
nlsc 控制 nls 实时更新的能力可 disable。  
controller 还可以作为 server 对外暴露 restful api

```text
全局可能有多个 nlsc，controller 需指定 nlsc 名称，之后 controller 根据该 nlsc 对集群进行操作：  
  
- controller 启动后，首先创建 nls 资源。集群有多少个节点，controller 就创建多少个 nls 资源，资源名称同nodename  
- controller 监听 nls 资源，若用户没有 disable nls实时更新能力，则 controller 会使 nls 资源始终保持与 nlsc 一致（只编辑nls的spec字段）  
- nlsc资源中优先级体现  
  - 先按照前后顺序覆盖  
- agent 根据节点上的资源信息，更新 status 字段  
  - extender不再更新nls，由agent来更新status字段中的filteredStorageInfo部分  
  - extender仅watch nls资源，并更新cache  
- 事件监听逻辑  
  - 监听node  
    - add：createNLS  
    - update：不做操作  
    - delete：deleteNLS  
  - 监听nlsc  
    - add：createNLS  
    - update：updateNLS  
    - delete：不做操作  
  - 监听nls  
    - add：是否执行 updateNLS  
    - update：是否执行 updateNLS  
    - delete：createNLS
```


#### code 分析

```go
// controller 主要做资源同步，所以分别创建了三套 Informer Factory
// kubeInformerFactory 负责 pod、node、pv、pvc
// localInformerFactory 负责 nls、nlsc
// snapshotInformerFactory 负责 VolumeSnapshots 等
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)  
localInformerFactory := localinformers.NewSharedInformerFactory(localClient, time.Second*30)  
snapshotInformerFactory := snapshotinformers.NewSharedInformerFactory(snapClient, time.Second*30)

// 和 agent 类似，也是用 RateLimitingQueue 队列
// 通过向各 informer 注册回调，处理各种资源事件
handleNLS
handleNLSC
createNLSByNode/deleteNLSByNode
handlePod
triggerCleanUnusedRepo
```

### scheduler

kubernetes scheduler extender
为集群提供 local storage 的整体调度能力
extener 模块以 deployment 的方式部署在 master
通过“亲和性”和“污点”调度，让 extender 只部署在控制面：

```yaml
...
tolerations:  
- operator: Exists  
  effect: NoSchedule  
  key: node-role.kubernetes.io/control-plane  
affinity:  
  nodeAffinity:  
    requiredDuringSchedulingIgnoredDuringExecution:  
      nodeSelectorTerms:  
      - matchExpressions:  
        - key: node-role.kubernetes.io/control-plane  
          operator: Exists
...
```

#### code 分析

```go
// 和 controller 一样，也有三套 Informer Factory
kubeInformerFactory
localStorageInformerFactory
snapshotInformerFactory

// 通过向各 Informer 注册处理回调
// 提供资源缓存
// 校验
// 调度算法等能力

// 还提供 API 接口
router := httprouter.New()  
AddVersion(router)  
AddMetrics(router, e.Ctx)  
AddGetNodeCache(router, e.Ctx)  
AddPredicate(router, *predicates.NewPredicate(e.Ctx))  
AddPrioritize(router, *priorities.NewPrioritize(e.Ctx))  
AddSchedulingApis(router, e.Ctx)
```

## CRD 资源

### NodeLocalStorage

- 代码 `docs/api/nls_zh_CN.md` 中有说明。
- `helm/crds` 中定义的 CRD 资源。
- 由 Controller 创建，运行在每个 worker node 上，由每个 node 上的 agent 组件负责更新其 Status，用来上报每个节点的存储设备信息。

```yaml
apiVersion: csi.aliyun.com/v1alpha1
kind: NodeLocalStorage
spec:
  nodeName: [node name]       # 与节点名称相同
  listConfig:                 # 可被 Open-Local 分配的存储设备列表，形式为正则表达式。Status 中 .nodeStorageInfo.deviceInfo 和 .nodeStorageInfo.volumeGroups 满足该正则表达式的设备名称将成为 Open-Local 具体可分配的存储设备列表，并在 Status 的 .filteredStorageInfo 中显示。Open-Local 先通过 include 得出全量列表，然后通过 exclude 从全量列表中剔除不需要的项
    devices:                  # Device（独占盘）名单
      include:                # include 正则
      - /dev/vd[a-d]+
      exclude:                # exclude 正则
      - /dev/vda
      - /dev/vdb
    vgs:                      # LVM（共享盘）白黑名单，这里的共享盘名称指的是 VolumeGroup 名称
      include:
      - share
      - paas[0-9]*
      - open-local-pool-[0-9]+
  resourceToBeInited:         # 设备初始化列表
    vgs:                      # LVM（共享盘）初始化
    - devices:                # 将块设备 /dev/vdb3 初始化为名为 open-local-pool-0 的 VolumeGroup。注意：当节点上包含同名 VG，则 Open-Local 不做操作
      - /dev/vdb3
      name: open-local-pool-0
status:
  nodeStorageInfo:            # 具体设备情况，由 Agent 组件更新。包含 分区 和 一整个块设备。设备名称可由 open-local agent --regexp 参数决定（默认为 ^(s|v|xv)d[a-z]+$ ）
    deviceInfo:               # 磁盘情况
    - condition: DiskReady    # 磁盘状态，有三种状态：DiskReady、DiskFull、DiskFault
      mediaType: hdd          # 媒介类型，分为 hdd 和 sdd 两种
      name: /dev/vda1         # 设备名称
      readOnly: false         # 是否只读
      total: 53685353984      # 设备总量
    - condition: DiskReady
      mediaType: hdd
      name: /dev/vda
      readOnly: false
      total: 53687091200
    - condition: DiskReady
      mediaType: hdd
      name: /dev/vdb1
      readOnly: false
      total: 107374164992
    - condition: DiskReady
      mediaType: hdd
      name: /dev/vdb2
      readOnly: false
      total: 106300440576
    - condition: DiskReady
      mediaType: hdd
      name: /dev/vdb3
      readOnly: false
      total: 860066152448
    - condition: DiskReady
      mediaType: hdd
      name: /dev/vdb
      readOnly: false
      total: 1073741824000
    - condition: DiskReady
      mediaType: hdd
      name: /dev/vdc
      readOnly: false
      total: 1073741824000
    volumeGroups:                 # VolumeGroup 情况
    - allocatable: 860063006720   # 可被 Open-Local 分配的VG可用量，会剔除非 Open-Local 的 LV 总量。Open-Local 的 LV 名称由 open-local agent --lvname 参数决定，前缀不匹配的 LV 为非 Open-Local 的 LV。
      available: 800298369024     # VG 可用量
      condition: DiskReady        # VG 状态
      logicalVolumes:                                       # LV 信息
      - condition: DiskReady                                # LV 状态
        name: local-482c664d-764b-461e-be5e-0a60a3abd5ac    # LV 名称
        total: 1073741824                                   # LV 总量
        vgname: open-local-pool-0                           # LV 所在的 VG 名称
      - condition: DiskReady
        name: local-4e0b8d6f-8ea2-431d-8b5d-aaec1b5d4242
        total: 53687091200
        vgname: open-local-pool-0
      - condition: DiskReady
        name: local-cc69d090-15b9-4abd-af1f-04380e1654d9
        total: 5003804672
        vgname: open-local-pool-0
      name: open-local-pool-0     # VG 名称
      physicalVolumes:            # VG 对应的 PVs（Physical Volumes）
      - /dev/vdb3
      total: 860063006720         # VG 总量
  filteredStorageInfo:            # 设备筛选情况，筛选后的设备会参与存储调度&分配。该字段的值由 Status 中的 .nodeStorageInfo.deviceInfo 和 .nodeStorageInfo.volumeGroups 与 Spec 中的 .listConfig 共同决定。本例中 Spec 的 VG 列表中有 open-local-pool-[0-9]+，且该节点有名为 open-local-pool-0 的 VG，故可被纳管。/dev/vdc 同理。
    volumeGroups:
    - open-local-pool-0
    devices:
    - /dev/vdc
```

### NodeLocalStorageInitConfig

- 代码 `docs/api/nlsc_zh_CN.md` 中有说明。
- `helm/crds` 中定义的 CRD 资源。
- Controller 在创建 nls(NodeLocalStorage)时，会根据环境中已存在的 nlsc（NodeLocalStorageInitConfig） 中的配置信息，填充到 nls 中。
- Controller 还会 watch nlsc 的资源更改，并将更改作用到已存在的 nls 上。
