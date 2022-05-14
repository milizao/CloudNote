## 卷

支持将cinder卷、glusterfs卷等多种类型的卷，直接绑定到Pod中使用。一些类型的卷（比如glusterfs）在使用前需要先进行卷的配置。

临时卷类型的生命周期与 Pod 相同，但持久卷可以比 Pod 的存活期长。 当 Pod 不再存在时，Kubernetes 也会销毁临时卷；不过 Kubernetes 不会销毁持久卷。

临时卷：`emptyDir`、`configMap`、`downwardAPI`、`secret`

持久卷：持久卷是用插件的形式来实现的。Kubernetes 目前支持HostPath 、local、CephFS 、Glusterfs 、Cinder、AWS 、Azure 等多种插件。

Cinder卷示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-cinder
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-cinder-container
    volumeMounts:
    - mountPath: /test-cinder
      name: test-volume
  volumes:
  - name: test-volume
    # 此 OpenStack 卷必须已经存在
    cinder:
      volumeID: "<volume-id>"
      fsType: ext4
```

hostPath卷示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # 宿主上目录位置
      path: /data
      # 此字段为可选
      type: Directory
```

local卷示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

## 持久卷

### 供应 

PV 卷的供应有两种方式：静态供应或动态供应。

#### 静态供应 

集群管理员创建若干 PV 卷。这些卷对象带有真实存储的细节信息，并且对集群 用户可用（可见）。PV 卷对象存在于 Kubernetes API 中，可供用户消费（使用）。

#### 动态供应 

如果管理员所创建的所有静态 PV 卷都无法与用户的 PersistentVolumeClaim 匹配， 集群可以尝试为该 PVC 申领动态供应一个存储卷。 这一供应操作是基于 StorageClass 来实现的：PVC 申领必须请求某个`StorageClass`，同时集群管理员必须 已经创建并配置了该`StorageClass`，这样动态供应卷的动作才会发生。 如果 PVC 申领指定存储类为 `""`，则相当于为自身**禁止**使用动态供应的卷。

### 绑定 

当 PVC 绑定 PV 时，需考虑以下参数来筛选当前集群内是否存在满足条件的 PV。

| 参数         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| VolumeMode   | 主要定义 volume 是文件系统（FileSystem）类型还是块（Block）类型，PV 与 PVC 的 VolumeMode 标签必须相匹配。 |
| Storageclass | PV 与 PVC 的 storageclass 类名必须相同（或同时为空）。       |
| AccessMode   | 主要定义 volume 的访问模式，PV 与 PVC 的 AccessMode 必须相同。 |
| Size         | 主要定义 volume 的存储容量，PVC 中声明的容量必须小于等于 PV，如果存在多个满足条件的 PV，则选择最小的 PV 与 PVC 绑定。 |

为PVC绑定的PV可能会超出PVC请求的容量大小。 一旦绑定关系建立，则 PersistentVolumeClaim 绑定就是排他性的，无论该 PVC 申领是 如何与 PV 卷建立的绑定关系。 PVC 申领与 PV 卷之间的绑定是一种一对一的映射，实现上使用 ClaimRef 来记述 PV 卷 与 PVC 申领间的双向绑定关系。

如果找不到匹配的 PV 卷，PVC 申领会无限期地处于未绑定状态。 当与之匹配的 PV 卷可用时，PVC 申领会被绑定。 例如，即使某集群上供应了很多 50 Gi 大小的 PV 卷，也无法与请求 100 Gi 大小的存储的 PVC 匹配。当新的 100 Gi PV 卷被加入到集群时，该 PVC 才有可能被绑定。

如果要将PV预留给特定的PVC，可以通过PV 的 `claimRef` 字段进行设置：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc   # 只有foo-pvc才可以绑定至该PV
    namespace: foo
  ...
```

PVC绑定至特定PV：通过PVC的`volumeName`字段指定 PV：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: pv1   # 指定改PV绑定到pv1这个PVC
```

另外，也可以通过给PV打label，然后让PVC去匹配这个label的方式进行绑定：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv2
  namespace: chenqiang-pv-test
  labels:
    pv: nfs-pv2  # PV增加标签
spec:
  capacity:
    storage: 100MiB
  accessModes:
    - ReadWriteMany
  nfs:
    server: 1.2.3.4
    path: /test/mysql-nfs 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc2
  namespace: chenqiang-pv-test
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 90MiB
  selector:
    matchLabels:  # PVC选择标签
      pv: nfs-pv2
```



### 使用 

Pod 将 PVC 申领当做存储卷来使用。集群会检视 PVC 申领，找到所绑定的卷，并 为 Pod 挂载该卷。对于支持多种访问模式的卷，用户要在 Pod 中以卷的形式使用申领 时指定期望的访问模式。



### 访问模式

PersistentVolume 卷可以用资源提供者所支持的任何方式挂载到**宿主系统**上。

* ReadWriteOnce

卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的**多个 Pod 访问**卷。

* ReadOnlyMany

卷可以被多个节点以只读方式挂载。

* ReadWriteMany

卷可以被多个节点以读写方式挂载。

* ReadWriteOncePod

卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。



卷是挂载到了node上还是Pod上？如果是node上，Pod是怎么访问的？



Pod、StatefulSet、Deployment如何使用PVC、StorageClass？



https://akomljen.com/kubernetes-persistent-volumes-with-deployment-and-statefulset/

### Deployment使用PV样例

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: zookeeper
spec:
  selector:
    matchLabels:
      app: zookeeper
  replicas: 1
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - env:
        - name: ZOOKEEPER_SERVERS
          value: "1"
        image: "komljen/zookeeper:3.4.10"
        imagePullPolicy: IfNotPresent
        name: zookeeper
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        readinessProbe:
          exec:
            command:
            - /opt/zookeeper/bin/zkOK.sh
          initialDelaySeconds: 10
          timeoutSeconds: 2
          periodSeconds: 5
        livenessProbe:
          exec:
            command:
            - /opt/zookeeper/bin/zkOK.sh
          initialDelaySeconds: 120
          timeoutSeconds: 2
          periodSeconds: 5
        volumeMounts:
        - mountPath: /data
          name: zookeeper-data
      restartPolicy: Always
      volumes:
      - name: zookeeper-data
        persistentVolumeClaim:    # PVC配置
          claimName: zookeeper-vol    # zookeeper-vol需要提前建好
```

### StatefulSet使用PVC样例

```yaml
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: zookeeper
spec:
  selector:
    matchLabels:
      app: zookeeper
  replicas: 1
  serviceName: zookeeper-server
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - env:
        - name: ZOOKEEPER_SERVERS
          value: "1"
        image: "komljen/zookeeper:3.4.10"
        imagePullPolicy: IfNotPresent
        name: zookeeper
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        readinessProbe:
          exec:
            command:
            - /opt/zookeeper/bin/zkOK.sh
          initialDelaySeconds: 10
          timeoutSeconds: 2
          periodSeconds: 5
        livenessProbe:
          exec:
            command:
            - /opt/zookeeper/bin/zkOK.sh
          initialDelaySeconds: 120
          timeoutSeconds: 2
          periodSeconds: 5
        volumeMounts:
        - mountPath: /data
          name: zookeeper-vol
      restartPolicy: Always
  volumeClaimTemplates:   # PVC模板配置，根据该模板自动创建PVC，每个replica一个，PVC以-0这样的数字结尾，确保每个replica不重复，并且同一个replica总是会绑定到同一个PVC
  - metadata:
      name: zookeeper-vol
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 8Gi
      storageClassName: rbd   # storageClass需要提前建好
```

而如果直接给StatefulSet绑定PV，像这样：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: "redis"
  selector:
    matchLabels:
      app: redis
  updateStrategy:
    type: RollingUpdate
  replicas: 3
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis
        resources:
          limits:
            memory: 2Gi
        ports:
          - containerPort: 6379
        volumeMounts:
          - name: redis-data
            mountPath: /usr/share/redis
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: redis-data-pvc
```

应当也可以（待验证），前提是这个StatefulSet里面所有replica需要共享同一个pvc，并且该PVC必须是ReadOnlyMany或ReadWriteMany的。

参考：https://stackoverflow.com/questions/65266223/how-to-set-pvc-with-statefulset-in-kubernetes



StatefulSet和Deployment的区别

https://developer.aliyun.com/article/603755

“Deployment用于部署无状态服务，StatefulSet用来部署有状态服务”。

具体的，什么场景需要使用StatefulSet呢？官方给出的建议是，如果你部署的应用满足以下一个或多个部署需求，则建议使用StatefulSet。

- 稳定的、唯一的网络标识。
- 稳定的、持久的存储。
- 有序的、优雅的部署和伸缩。
- 有序的、优雅的删除和停止。
- 有序的、自动的滚动更新。

具体到存储，StatefulSet中的多个Pod虽然是同一套模板建立出来的，但每个Pod都有着自己特定的存储，多个Pod之间不能错乱（所以PVC必须通过模板创建，不能提前指定）。因此，如果希望每次Pod都能与同一个PVC绑定，就需要用到StatefulSet。而Deployment的多个Pod之间是共享同一个存储的，因此在Pod模板中就要指定好用哪个PVC，并且必须是ReadWriteMany类型的PVC。
