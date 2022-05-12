https://www.1024sou.com/article/351148.html


# [K8s容器存储接口（CSI）介绍](https://www.1024sou.com/article/351148.html)

收录于 2021-10-27 13:02:20

查看 3322 次

Container Storage Interface是由来自Kubernetes、Mesos、Docker等社区member联合制定的一个行业标准接口规范，旨在将任意存储系统暴露给容器化应用程序。

CSI规范定义了存储提供商实现CSI兼容的Volume Plugin的最小操作集和部署建议。CSI规范的主要焦点是声明Volume Plugin必须实现的接口。

 

# 一、CSI插件需实现的接口

CSI Plugin开发者要实现三类gRPC服务接口：

实现了此接口的CSI插件，不但可以在k8s中使用，在所有参与了CSI标准制订的Container Orchestration system中都是通用的。

 

# 二、CSI sidecar容器

为了协助存储提供商开发CSI兼容插件，Kubernetes官方提供了一些CSI sidecar容器（对应了从Kubernetes项目里面剥离出来的存储管理功能）

- ## external-provisioner（csi-provisioner）

在启动时通过--provisioner指定自身provisioner名称，与StorageClass中的provisioner字段对应。

（1）watch PVC对象，判断PVC是否需要动态创建存储卷，标准如下：

①PVC的annotation[volume.beta.kubernetes.io/storage-provisioner]（由PV Controller创建）是否与自己的provisioner名称相等。

②PVC对应StorageClass的VolumeBindingMode若为Immediate，则表示需要立即提供动态存储卷。

此时调用CSI Plugin的CreateVolume接口，同时创建名为${Provisioner指定的PV前缀}-${PVC uuid}的PV

（2）watch PV对象，判断其是否需要删除，标准如下：

①判断其.status.phase是否为Release。

②判断其.spec.PersistentVolumeReclaimPolicy是否为Delete。

③判断其annotation[pv.kubernetes.io/provisioned-by]是否与自己的provisioner名称相等。

若需要，则调用CSI Plugin的DeleteVolume接口，同时删除PV对象

- ## external-attacher（csi-attacher）

（1）watch VolumeAttachment对象，获得PV的所有信息，如volume ID、node ID等。根据VolumeAttachment的DeletionTimestamp和.status.attached来判断是调用CSI Plugin的ControllerPublish做attach，还是调用CntrollerUnpublish接口做detach

（2）在attacher时为相关PV打上Finalizer；当PV处于删除状态（DeletionTimestamp非空）时，删除Finalizer

- ## external-snapshotter

（1）watch VolumeSnapshot对象，根据其状态调用CSI Plugin的CreateSnapshot接口

等存储快照生成后，它会将存储快照生成的相关信息放到VolumeSnapshotContent对象中，并和用户提交的VolumeSnapshot做bound

（2）当VolumeSnapsho处于删除状态（DeletionTimestamp非空）时，调用CSI Plugin的DeleteSnapshot接口

- ## external-resizer

watch PVC对象，判断用户在PVC中是否增加了需求的存储空间。如果PVC状态是Bound且.status.Capacity与.spec.Resources.Requests不等，则进行卷扩展：

①更新PVC的.status.Conditions，表明此时处于Resizing状态

②调用CSI Plugin的ControllerExpandVolume接口，若返回值中NodeExpansionRequired=true（还需要Kubelet中的Volume Manager继续调用CSI Plugin的NodeExpandVolume接口进行扩容），则更新PVC的status.Conditions 为 FileSystemResizePending；否则，更新 PVC的.Status.Conditions为空，且更新PVC的status.Capacity。

③更新PV的spec.Capacity

- ## node-driver-registrar

调用CSI Plugin的接口获取插件信息，通过Kubelet的插件注册机制将CSI Plugin注册到kubelet 

- ## livenessprobe

 调用CSI Plugin的Probe接口，同时在/healthz暴露HTTP健康检查探针 

# 三、CSI插件与CSI sidecar容器的组合

一般会在一个CSI Plugin容器中同时实现所有接口，使其同时作为CSI Controller Server和CSI Node Server

它会和Kubernetes官方提供的CSI sidecar容器组成Pod，每种组合都会完成某种功能：

CSI Controller Server和External CSI SideCar、CSI Node Server和Kubelet通过Unix Socket来通信：

 

因此，CSI分为两部分：

- 由k8s社区驱动实现的通用的部分，如图中的csi-provisioner和csi-attacher
- 由云存储厂商实现的csi-controller-server和csi-node-server（一般集成到一个一个CSI Plugin容器中），对接云存储厂商的API实现真正的create/delete/mount/unmount存储的相关操作

 

# 四、通过CRD管理CSI

对CSI的管理是通过CRD的形式实现的，所以引入了自定义对象类型VolumeAttachment、CSINode、CSIDriver 

## （1）VolumeAttachmen

VolumeAttachment描述一个Volume的attach/detach信息。可以通过VolumeAttachment对一个Volume在某个节点上的attach状态进行跟踪。

 

VolumeAttachmentSpec数据结构：

- Attacher

根据Driver name指定由哪个CSI Plugin来进行attach/detach

- NodeName

指定了attach/detach是发生在哪个节点上的

- Source

指定了对哪一个PV进行attach/detach。

 

VolumeAttachmentStatus数据结构：

- Attached

指示了是否已经attach

  如果VolumeAttachment的DeletionTimestamp为空且Attached=false，则external-attacher需要调用CSI Plugin的ControllerPublish做attach

  如果VolumeAttachment的DeletionTimestamp不为空且Attached=true，则external-attacher需要调用CSI Plugin的ControllerUnPublish做detach

 

## （2）CSIDriver

CSIDriver描述了部署的CSI Plugin、定义了Kubernetes调用CSI Plugin的行为，需要管理员根据插件类型进行创建

CSIDriverSpec数据结构：

- AttachRequired

定义一个Plugin是否支持Attach功能。

不需要Attach操作时，该标签就是False，说明无需部署external-attacher，也不需要VolumeAttachment对象。

例子：调用CreateVolume接口时，CreateVolumeRequest中VolumeCapability决定了是文件系统直接挂载还是块设备挂载。块存储需要Attach操作，但文件存储不需要Attach操作，例如NFS没有挂载、格式化的概念，只需要mount远端文件系统到本地即可

 

 

- PodInfoOnMount

定义了调用CSI Plugin的NodePublishVolume接口时是否带上Pod信息（PodName、PodUID、PodNamespace）。

默认为false，即CSI Plugin无法获知Pod信息

- VolumeLifecycleModes

该CSI Plugin支持哪些VolumeLifecycleMode

包括Persistent和Ephemeral两种

- StorageCapacity

若为true，表示希望调度器调度时考虑节点的存储容量

- FSGroupPolicy

Volume挂载前是否允许更改属主、权限的定义

- TokenRequests

CSI Plugin可能需要Pod的Service Account Token

- RequiresRepublish

若为true，表示CSI Plugin希望不断被调用NodepublishVolume接口，以更新挂载到Pod中的Volume的相关变化。

不过只能更新Volume的contents，running状态Pod的挂载点是不能改变的

## （3）CSINode

CSINode是集群中的节点信息，由Node上的node-driver-registrar在启动时创建。

因此，可以只定义集群中一部分节点拥有CSINode。

 

CSINodeSpec由一组CSIDriver信息组成。每一个新的CSI Plugin注册后，都会在其中添加自身信息

每个CSI Plugin可添加的CSIDriver信息包括：

- Name
- NodeID
- Allocatable

节点最多可以attach多少个Volume

- TopologyKeys

该CSINode支持哪些TopologyKeys。进行支持拓扑感知的provisioning时，将会根据这些TopologyKeys从Node获取values，会传递给CSI Plugin

如果CSI Plugin不支持拓扑，可以置为null。

 

# 五、总结

## 插件注册

（1）kubelet在启动的时候有一个约定，例如在/var/lib/kubelet/plugins_registry目录每新加一个文件，就相当于新加了一个Plugin。

（2）node-driver-registrar首先会调用CSI-Plugin的GetPluginInfo接口，获取CSI Plugin监听的地址以及其Driver name。

（3）node-driver-registrar会监听自身的GetInfo和NotifyRegistrationStatus接口。

（4）在/var/lib/kuberlet/plugins_registry这个目录下启动一个Socket，生成一个Socket文件 ，例如："com.nfs.csi.tds-reg.sock”

（5）kubelet Watche到Socket后，通过该Socket调用node-driver-registrar的GetInfo接口，获取其刚刚获得的CSI-Plugin的信息

（6）获取成功之后，Kubelet会：

①根据CSI Plugin监听地址调用其NodeGetInfo接口，获取[nodeID]、[Drive name]、[AccessibleTopology]、maxAttachLimit（节点最大可挂载卷数量）

   为Node新建（或更新已有）annotation[csi.volume.kubernetes.io/nodeid]=xxx

  基于[AccessibleTopology]给Node添加label

   根据maxAttachLimit更改Node的.status.Allocatable：attachable-volumes-csi-[Drive name]=[maxAttachLimit]。

②根据CSI Plugin监听地址调用其NodeGetVolumeStats接口，并反映在其metrics：

   kubelet_volume_stats_capacity_bytes：存储卷容量

   kubelet_volume_stats_used_bytes：存储卷已使用容量

   kubelet_volume_stats_available_bytes：存储卷可使用容量

   kubelet_volume_stats_inodes：存储卷inode总量

   kubelet_volume_stats_inodes_used：存储卷inode使用量

   kubelet_volume_stats_inodes_free：存储卷inode剩余量

③创建一个CSINode对象（或更新已有CSINode对象）

④调用node-driver-registrar的NotifyRegistrationStatus接口，告诉它CSI-Plugin已经注册成功了。

## Provisioning Volumes

（1）PV Controller观察到集群中新创建的PVC没有与之匹配的PV，且其对应SC指定的Provisioner不是Internal Provisioner，于是为PVC打annotation[volume.beta.kubernetes.io/storage-provisioner]=xxx

（2）对应external-provisioner判断需要创建存储卷，于是调用CSI Plugin的CreateVolume接口

（3）external-provisioner在请求参数设置AccessibilityRequirements

   如果SC的VolumeBindingMode为Immediate模式：如果SC指定了AllowedTopologies则，只考虑AllowedTopologies，否则需要将所有包含TopologyKeys键的节点Value添进来。

   如果SC的VolumeBindingMode为WaitForFirstConsumer模式：获取Pod调度到的节点的CSINode对象的TopologyKeys，根据这些TopologyKeys从Node的Label获取values值，与SC的AllowedTopologies比对，不符合则报错

（4）CSI Plugin返回成功后，表示存储卷已经在Cloud Storage Vendor真正创建完成

存储卷既有可能是新的，也有可能是根据VolumeSnapshotContent中的Snapshot的ID创建的：

（5）external-provisioner会创建一个PV，PV Controller会将PV与PVC进行绑定。

 

## Attaching Volumes

（1）AD Controller观察到使用VolumeSource为CSI的PV的Pod被调度到某一节点，会调用AD Controller内部in-tree CSI插件（csiAttacher）的Attach函数，由其创建一个VolumeAttachment

（2）同时，external-attacher也会同步一些PV的信息在VolumeAttachment里。

（3）external-attacher观察到该VolumeAttachment对象，如果其.spec.Attacher中的Driver name指定的是自己同一Pod内的CSI Plugin，则调用CSI Plugin的ControllerPublish接口

（4）CSI Plugin会调用云存储厂商的API，把远端的Volume attach到目标节点中上（如/dev/vdb）

（5）external-attacher更新该VolumeAttachment的.status.Attached为true。

（4）AD Controller内部in-tree CSI插件（csiAttacher）观察到该VolumeAttachment的.status.Attached为 true，于是更新AD Controller内部状态（ActualStateOfWorld），该状态会显示在Node资源的.status.VolumesAttached上。

## Mounting Volumes

（1）kubelet组件Volume Manager观察到有新的使用PersistentVolumeSource为CSI的PV的Pod调度到本节点上，于是调用内部in-tree CSI插件（csiAttacher）的WaitForAttach函数。

（2）内部in-tree CSI插件（csiAttacher）等待attach完成后，调用MountDevice函数，由其调用CSI plugin的NodeStageVolume接口；

（3）插件csiAttacher调用内部 in-tree CSI 插件（csiMountMgr）的SetUp函数，由其调用CSI plugin的NodePublishVolume接口。

## Unmounting Volumes

用户删除相关Pod后

（1）kubelet组件Volume Manager观察到有新的使用PersistentVolumeSource为CSI的PV的Pod被删除，于是调用内部 in-tree CSI 插件（csiMountMgr）的TearDown函数，由其调用CSI plugin的NodeUnpublishVolume接口。

（2）kubelet组件Volume Manager调用内部in-tree CSI 插件（csiAttacher）的 UnmountDevice函数，由其调用CSI plugin的NodeUnpublishVolume接口。

## Detaching Volumes

（1）AD Controller观察到有PersistentVolumeSource为CSI的PV被删除，就会调用内部 in-tree CSI 插件（csiAttacher）的Detach函数。

（2）csiAttacher会删除相关VolumeAttachment对象（但由于存在finalizer，不会立即删除成功）。

（3）external-attacher观察到该VolumeAttachment对象的DeletionTimestamp非空，于是调用CSI Plugin的 ControllerUnpublish接口以从对应节点上detach Volume。

（4）detach成功后，external-attacher会移除该VolumeAttachment对象的finalizer字段，使其被彻底删除。

（4）AD Controller内部in-tree CSI插件（csiAttacher）观察到该VolumeAttachment对象已删除，于是更新AD Controller中的内部状态；同时从Node的.status.VolumesAttached中清除此attach信息。

## Deleting Volumes

external-provisioner watch到用户删除了PVC，根据 PVC的回收策略（Reclaim）执行不同操作：

   Delete：调用CSI Plugin的DeleteVolume接口，由其去远端存储删除存储卷；一旦成功删除，external-provisioner会删除对应PV对象。

   Retain：external-provisioner不执行存储卷删除操作。

 

参考资料：

[1] https://kubernetes.io/docs/home/

[2] https://edu.aliyun.com/roadmap/cloudnative

[3] 郑东旭《Kubernetes源码剖析》
