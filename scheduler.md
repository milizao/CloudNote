kube-scheduler调度概述
在 Kubernetes 中，调度是指将 Pod 放置到合适的 Node 节点上，然后对应 Node 上的 Kubelet 才能够运行这些 pod。

调度器通过 kubernetes 的 watch 机制来发现集群中新创建且尚未被调度到 Node 上的 Pod。调度器会将发现的每一个未调度的 Pod 调度到一个合适的 Node 上来运行。调度器会依据下文的调度原则来做出调度选择。

调度是容器编排的重要环节，需要经过严格的监控和控制，现实生产通常对调度有各类限制，譬如某些服务必须在业务独享的机器上运行，或者从灾备的角度考虑尽量把服务调度到不同机器，这些需求在Kubernetes集群依靠调度组件kube-scheduler满足。

kube-scheduler是Kubernetes中的关键模块，扮演管家的角色遵从一套机制——为Pod提供调度服务，例如基于资源的公平调度、调度Pod到指定节点、或者通信频繁的Pod调度到同一节点等。容器调度本身是一件比较复杂的事，因为要确保以下几个目标：

公平性：在调度Pod时需要公平的进行决策，每个节点都有被分配资源的机会，调度器需要对不同节点的使用作出平衡决策。
资源高效利用：最大化群集所有资源的利用率，使有限的CPU、内存等资源服务尽可能更多的Pod。
效率问题：能快速的完成对大批量Pod的调度工作，在集群规模扩增的情况下，依然保证调度过程的性能。
灵活性：在实际运作中，用户往往希望Pod的调度策略是可控的，从而处理大量复杂的实际问题。因此平台要允许多个调度器并行工作，同时支持自定义调度器。
为达到上述目标，kube-scheduler通过结合Node资源、负载情况、数据位置等各种因素进行调度判断，确保在满足场景需求的同时将Pod分配到最优节点。显然，kube-scheduler影响着Kubernetes集群的可用性与性能，Pod数量越多集群的调度能力越重要，尤其达到了数千级节点数时，优秀的调度能力将显著提升容器平台性能。
