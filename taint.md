Taints污点和Tolerations容忍概述
节点和Pod亲和力，是将Pod吸引到一组节点【根据拓扑域】（作为优选或硬性要求）。污点（Taints）则相反，它们允许一个节点排斥一组Pod。

容忍（Tolerations）应用于pod，允许（但不强制要求）pod调度到具有匹配污点的节点上。

污点（Taints）和容忍（Tolerations）共同作用，确保pods不会被调度到不适当的节点。一个或多个污点应用于节点；这标志着该节点不应该接受任何不容忍污点的Pod。

说明：我们在平常使用中发现pod不会调度到k8s的master节点，就是因为master节点存在污点。

 

Taints污点
Taints污点的组成
使用kubectl taint命令可以给某个Node节点设置污点，Node被设置污点之后就和Pod之间存在一种相斥的关系，可以让Node拒绝Pod的调度执行，甚至将Node上已经存在的Pod驱逐出去。

每个污点的组成如下：

key=value:effect
每个污点有一个key和value作为污点的标签，effect描述污点的作用。当前taint effect支持如下选项：

NoSchedule：表示K8S将不会把Pod调度到具有该污点的Node节点上
PreferNoSchedule：表示K8S将尽量避免把Pod调度到具有该污点的Node节点上
NoExecute：表示K8S将不会把Pod调度到具有该污点的Node节点上，同时会将Node上已经存在的Pod驱逐出去

https://www.cnblogs.com/zhanglianghhh/p/14022018.html
