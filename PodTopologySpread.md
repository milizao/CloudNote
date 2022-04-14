Pod Topology Spread Constraints介绍

https://cloud.tencent.com/developer/article/1639217

拓扑域：就 Kubernetes 而言，它们是按节点标签定义和分组的一系列 Node，例如属于相同区域/可用区/机架/主机名都可以作为划分拓扑域的依据。

在 k8s 集群调度领域，“亲和性”相关的概念本质上都是控制 Pod 如何被调度——堆叠或是打散。目前 k8s 提供了 podAffinity 以及 podAntiAffinity 两个特性对 Pod 在不同拓扑域的分布进行了一些控制，podAffinity 可以将无数个 Pod 调度到特定的某一个拓扑域，这是堆叠的体现；podAntiAffinity 则可以控制一个拓扑域只存在一个 Pod，这是打散的体现。但这两种情况都太极端了，在不少场景下都无法达到理想的效果，例如为了实现容灾和高可用，将业务 Pod 尽可能均匀得分布在不同可用区就很难实现。

PodTopologySpread 特性的提出正是为了对 Pod 的调度分布提供更精细的控制，以提高服务可用性以及资源利用率。

* labelSelector 用来查找匹配的 Pod。我们能够计算出每个拓扑域中匹配该  label selector 的 Pod 数量；在上图中，假如 label selector 是 "app:foo"，那么 zone1 的匹配个数为 2， zone2 的匹配个数为 0。
* topologyKey 是 Node label 的 key。如果两个 Node 的 label 同时具有该 key 并且 label 值相同，就说它们在同一个拓扑域。在上图中，指定 topologyKey 为 zone， 具有 "zone=zone1"  标签的 Node 被分在一个拓扑域，具有 "zone=zone2"  标签的 Node 被分在另一个拓扑域。
* maxSkew 描述了 Pod 在不同拓扑域中不均匀分布的最大程度，maxSkew 的取值必须大于 0。每个拓扑域都有一个 skew，计算的公式是：skew[i] = 拓扑域[i]中匹配的 Pod 个数 - min{拓扑域[*]中匹配的 Pod 个数}。在上图中，我们新建一个带有 "app=foo" label 的 Pod：
    - 如果该 Pod 被调度到 zone1，那么 zone1 中 Node 的 skew 值变为 3，zone2 中 Node 的 skew 值变为 0 (zone1 有 3 个匹配的 Pod，zone2 有 0 个匹配的 Pod )；
    - 如果该 Pod 被调度到 zone2，那么 zone1 中 Node 的 skew 值变为 1，zone2 中 Node 的 skew 值变为 0 (zone2 有 1 个匹配的 Pod，拥有全局最小匹配 Pod 数的拓扑域正是 zone2 自己 )；
* whenUnsatisfiable 描述了如果 Pod 不满足分布约束条件该采取何种策略：
    - DoNotSchedule (默认) 告诉调度器不要调度该 Pod，因此 DoNotSchedule 也可以叫作硬需求 (hard requirement)；
    - ScheduleAnyway 告诉调度器根据每个 Node 的 skew 值打分排序后仍然调度，因此 ScheduleAnyway 也可以叫作软需求 (soft requirement)；
