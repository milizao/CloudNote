Label是Kubernetes系统中的一个核心概念。Label以key/value键值对的形式附加到任何对象上，如Pod，Service，Node，RC（ReplicationController）/RS（ReplicaSet）等。Label可以在创建对象时就附加到对象上，也可以在对象创建后通过API进行额外添加或修改。

在为对象定义好Label后，其他对象就可以通过Label来对对象进行引用。Label的最常见的用法便是通过spec.selector来引用对象。

作者：沉沦2014
链接：https://www.jianshu.com/p/18b94993adbe
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



pod 与 node 之间的部署关系

https://www.cnblogs.com/zhanglianghhh/p/14077078.html


nodeName调度
nodeName是节点选择约束的最简单形式，但是由于其限制，通常很少使用它。nodeName是PodSpec的领域。

pod.spec.nodeName将Pod直接调度到指定的Node节点上，会【跳过Scheduler的调度策略】，该匹配规则是【强制】匹配。可以越过Taints污点进行调度。

nodeName用于选择节点的一些限制是：

如果指定的节点不存在，则容器将不会运行，并且在某些情况下可能会自动删除。
如果指定的节点没有足够的资源来容纳该Pod，则该Pod将会失败，并且其原因将被指出，例如OutOfmemory或OutOfcpu。
云环境中的节点名称并非总是可预测或稳定的。

nodeSelector调度
nodeSelector是节点选择约束的最简单推荐形式。nodeSelector是PodSpec的领域。它指定键值对的映射。

Pod.spec.nodeSelector是通过Kubernetes的label-selector机制选择节点，由调度器调度策略匹配label，而后调度Pod到目标节点，该匹配规则属于【强制】约束。由于是调度器调度，因此不能越过Taints污点进行调度。


Label使用场景

常用的，多维度标签分类：

版本标签（release）： stable（稳定版），canary（金丝雀版本），beta（测试版）

环境类（environment）： dev（开发），qa（测试），production（生产），op（运维）

应用类（applaction）： ui（设计），as（应用软件），pc（电脑端），sc（网络方面）

架构层（tier）： frontend（前端），backend（后端），cache（缓存）

分区标签（partition）： customerA（客户），customerB

品控级别（track）： daily（每天），weekly（每周）

作者：IT小分享
链接：https://www.jianshu.com/p/5990b68ae755
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
