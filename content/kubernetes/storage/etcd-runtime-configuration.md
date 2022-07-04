> **Have a nice day! :heart:**

# 参考
[Runtime reconfiguration](https://etcd.io/docs/v3.5/op-guide/runtime-configuration/)  
[Design of runtime reconfiguration](https://etcd.io/docs/v3.5/op-guide/runtime-reconf-design/)  
[etcd learner design](https://etcd.io/docs/v3.5/learning/design-learner/)

# 前言
> 如何替换一个由`statefulset`启动的`etcd cluster`中的某个节点

# 当前环境

- 测试部署文件
> 在单独的 namespace 中通过 statefulset 的方式部署了 3 个实例的 etcd 集群
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: etcd-test
---
apiVersion: v1
kind: Service
metadata:
  name: etcd-headless
  namespace: etcd-test
spec:
  clusterIP: None
  ports:
  - name: service-port
    port: 2379
    protocol: TCP
    targetPort: 2379
  - name: peer-port
    port: 2380
    protocol: TCP
    targetPort: 2380
  selector:
    component: etcd
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    component: etcd
  name: etcd
  namespace: etcd-test
spec:
  podManagementPolicy: OrderedReady
  replicas: 3
  selector:
    matchLabels:
      component: etcd
  serviceName: etcd-headless
  template:
    metadata:
      labels:
        component: etcd
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: component
                operator: In
                values:
                - etcd
            topologyKey: kubernetes.io/hostname
      containers:
      - command:
        - sh
        - -c
        - etcd -name $NODE_NAME -data-dir /data/data.etcd -advertise-client-urls http://$NODE_NAME.etcd-headless:2379
          -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380
          -initial-advertise-peer-urls http://$NODE_NAME.etcd-headless:2380 -initial-cluster-token
          etcd-cluster -initial-cluster $ETCD_CLUSTER -initial-cluster-state new
        env:
        - name: ETCD_SNAPSHOT_COUNT
          value: "10000"
        - name: ETCD_AUTO_COMPACTION_RETENTION
          value: "8"
        - name: ETCD_MAX_WALS
          value: "5"
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: NETWORK_HOST
          value: 0.0.0.0
        - name: ETCD_CLUSTER
          value: etcd-0=http://etcd-0.etcd-headless:2380,etcd-1=http://etcd-1.etcd-headless:2380,etcd-2=http://etcd-2.etcd-headless:2380
        image: quay.io/coreos/etcd:v3.5.0
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 20
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: service-port
          timeoutSeconds: 1
        name: etcd
        ports:
        - containerPort: 2379
          name: service-port
          protocol: TCP
        - containerPort: 2380
          name: peer-port
          protocol: TCP
        securityContext:
          capabilities:
            add:
            - IPC_LOCK
          privileged: true
        volumeMounts:
        - mountPath: /data/data.etcd
          name: data
      dnsPolicy: ClusterFirst
      restartPolicy: Always
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      creationTimestamp: null
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 16Gi
      storageClassName: rook-ceph-block
      volumeMode: Filesystem
```

- 状态
```shell
# kubectl -n etcd-test get pods -w
NAME     READY   STATUS    RESTARTS   AGE
etcd-0   1/1     Running   0          2m11s
etcd-1   1/1     Running   0          2m3s
etcd-2   1/1     Running   1          110s
### 3 个 member 
# kubectl -n etcd-test exec -it etcd-0 -- etcdctl member list -w table
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
|        ID        | STATUS  |  NAME  |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
| 3f54ac025181a433 | started | etcd-1 | http://etcd-1.etcd-headless:2380 | http://etcd-1.etcd-headless:2379 |      false |
| 7c5422fe4922f16f | started | etcd-2 | http://etcd-2.etcd-headless:2380 | http://etcd-2.etcd-headless:2379 |      false |
| e4ad7553eba80d25 | started | etcd-0 | http://etcd-0.etcd-headless:2380 | http://etcd-0.etcd-headless:2379 |      false |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
### 都很健康
# kubectl -n etcd-test exec -it etcd-0 -- etcdctl endpoint --cluster health -w table
+----------------------------------+--------+------------+-------+
|             ENDPOINT             | HEALTH |    TOOK    | ERROR |
+----------------------------------+--------+------------+-------+
| http://etcd-0.etcd-headless:2379 |   true | 2.749755ms |       |
| http://etcd-1.etcd-headless:2379 |   true | 2.582209ms |       |
| http://etcd-2.etcd-headless:2379 |   true | 2.792835ms |       |
+----------------------------------+--------+------------+-------+

# kubectl -n etcd-test exec -it etcd-0 --  etcdctl endpoint --cluster status -w table
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|             ENDPOINT             |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://etcd-1.etcd-headless:2379 | 3f54ac025181a433 |   3.5.0 |   20 kB |     false |      false |         2 |         24 |                 24 |        |
| http://etcd-2.etcd-headless:2379 | 7c5422fe4922f16f |   3.5.0 |   20 kB |     false |      false |         2 |         24 |                 24 |        |
| http://etcd-0.etcd-headless:2379 | e4ad7553eba80d25 |   3.5.0 |   20 kB |      true |      false |         2 |         24 |                 24 |        |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

```

- 节点，volume 信息

pod<br>node | podIP<br>nodeIP | pvc | size | sc
----|-----|---|------|-----
etcd-0<br>node4 | 10.233.74.101<br>192.168.201.104 | data-etcd-0 | 16G | rook-ceph-block
etcd-1<br>node6 | 10.233.75.85<br>192.168.201.106  | data-etcd-1 | 16G | rook-ceph-block
etcd-2<br>node5 | 10.233.97.146<br>192.168.201.105 | data-etcd-2 | 16G | rook-ceph-block

# 实操
> 假设 etcd-0 里面的数据坏了，问题： 怎么才能把 etcd-0 弄坏呢？
## 步骤
- 破坏 etcd-0
- 重新加入 etcd-0
- 完成

### 破坏 etcd-0
- 给 etcd-0 所在的 node 添加个污点
- 删除 etcd-0 的 data(其实也可以把 pvc 给删了)
- delete etcd-0，使 etcd-0 Pending
```shell
### etcd-0 允许在 node4， 给 node4 添加个污点
# kubectl taint node node4 dedicated=nonetcd0:NoSchedule
node/node4 tainted
### 删除 pvc 里面的数据（方式千万种，这种顺手）
# kubectl -n etcd-test  exec -it etcd-0 -- sh -c "rm -rf /data/*"
### 删除 pod
# kubectl -n etcd-test delete pods etcd-0 
pod "etcd-0" deleted
### etcd-0 已处于 Pending 状态
# kubectl -n etcd-test get pods etcd-0
NAME     READY   STATUS    RESTARTS   AGE
etcd-0   0/1     Pending   0          110s
```

### 重新加入 etcd-0
- 删除失效的成员
- 新建成员，加入集群

删除失效的成员
```shell
### etcd-0 已经失联了
# kubectl -n etcd-test exec -it etcd-1  -- etcdctl endpoint --cluster health -w table
{"level":"warn","ts":1656731129.2060163,"logger":"client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc000518380/#initially=[http://etcd-0.etcd-headless:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp: lookup etcd-0.etcd-headless on 10.233.0.10:53: no such host\""}
+----------------------------------+--------+--------------+---------------------------+
|             ENDPOINT             | HEALTH |     TOOK     |           ERROR           |
+----------------------------------+--------+--------------+---------------------------+
| http://etcd-2.etcd-headless:2379 |   true |    4.25798ms |                           |
| http://etcd-1.etcd-headless:2379 |   true |   4.226687ms |                           |
| http://etcd-0.etcd-headless:2379 |  false | 5.000539191s | context deadline exceeded |
+----------------------------------+--------+--------------+---------------------------+
Error: unhealthy cluster
### etcd-0 的 ID：e4ad7553eba80d25
# kubectl -n etcd-test exec -it etcd-1  -- etcdctl member list -w table
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
|        ID        | STATUS  |  NAME  |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
| 3f54ac025181a433 | started | etcd-1 | http://etcd-1.etcd-headless:2380 | http://etcd-1.etcd-headless:2379 |      false |
| 7c5422fe4922f16f | started | etcd-2 | http://etcd-2.etcd-headless:2380 | http://etcd-2.etcd-headless:2379 |      false |
| e4ad7553eba80d25 | started | etcd-0 | http://etcd-0.etcd-headless:2380 | http://etcd-0.etcd-headless:2379 |      false |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
### 通过 ID 删除 etcd-0
# kubectl -n etcd-test exec -it etcd-1  -- etcdctl member remove e4ad7553eba80d25
Member e4ad7553eba80d25 removed from cluster b08eae3b3a94d794
### etcd-0 已经从 member list 删除了
# kubectl -n etcd-test exec -it etcd-1  -- etcdctl member list -w table
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
|        ID        | STATUS  |  NAME  |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
| 3f54ac025181a433 | started | etcd-1 | http://etcd-1.etcd-headless:2380 | http://etcd-1.etcd-headless:2379 |      false |
| 7c5422fe4922f16f | started | etcd-2 | http://etcd-2.etcd-headless:2380 | http://etcd-2.etcd-headless:2379 |      false |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
```

新建成员，加入集群

```shell
# kubectl -n etcd-test exec -it etcd-1 -- etcdctl member add etcd-0 --peer-urls=http://etcd-0.etcd-headless:2380
Member 788e82417cb654eb added to cluster b08eae3b3a94d794

ETCD_NAME="etcd-0"
ETCD_INITIAL_CLUSTER="etcd-1=http://etcd-1.etcd-headless:2380,etcd-0=http://etcd-0.etcd-headless:2380,etcd-2=http://etcd-2.etcd-headless:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd-0.etcd-headless:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"

# kubectl -n etcd-test exec -it etcd-1 -- etcdctl member list -w table
+------------------+-----------+--------+----------------------------------+----------------------------------+------------+
|        ID        |  STATUS   |  NAME  |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+-----------+--------+----------------------------------+----------------------------------+------------+
| 3f54ac025181a433 |   started | etcd-1 | http://etcd-1.etcd-headless:2380 | http://etcd-1.etcd-headless:2379 |      false |
| 788e82417cb654eb | unstarted |        | http://etcd-0.etcd-headless:2380 |                                  |      false |
| 7c5422fe4922f16f |   started | etcd-2 | http://etcd-2.etcd-headless:2380 | http://etcd-2.etcd-headless:2379 |      false |
+------------------+-----------+--------+----------------------------------+----------------------------------+------------+
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: new-member
  namespace: etcd-test
  labels:
    component: etcd
spec:
  hostname: etcd-0
  subdomain: etcd-headless
  containers:
  - name: newmember
    image: quay.io/coreos/etcd:v3.5.0
    command: ['sh', '-c', 'etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://etcd-0.etcd-service:2379 --listen-peer-urls http://0.0.0.0:2380 --data-dir /var/lib/etcd']
    volumeMounts:
    - mountPath: "/var/lib/etcd"
      name: etcd-0
    env:
    ### 这里的环境变量其实就是上面 etcdctl member add 的输出
    - name: ETCD_NAME
      value: "etcd-0"
    - name: ETCD_INITIAL_CLUSTER
      value: "etcd-0=http://etcd-0.etcd-headless:2380,etcd-1=http://etcd-1.etcd-headless:2380,etcd-2=http://etcd-2.etcd-headless:2380"
    - name: ETCD_INITIAL_ADVERTISE_PEER_URLS
      value: "http://etcd-0.etcd-headless:2380"
    - name: ETCD_INITIAL_CLUSTER_STATE
      value: "existing"
  tolerations:
  - effect: NoSchedule
    operator: Exists
    key: dedicated
  volumes:
    - name: etcd-0
      persistentVolumeClaim:
        claimName: data-etcd-0
```
```shell
# kubectl apply -f new_member.yaml
pod/new-member created
# kubectl -n etcd-test get pods new-member 
NAME         READY   STATUS    RESTARTS   AGE
new-member   1/1     Running   0          57s
# kubectl -n etcd-test exec -it new-member -- etcdctl member list -w table
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
|        ID        | STATUS  |  NAME  |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
| 3f54ac025181a433 | started | etcd-1 | http://etcd-1.etcd-headless:2380 | http://etcd-1.etcd-headless:2379 |      false |
| 788e82417cb654eb | started | etcd-0 | http://etcd-0.etcd-headless:2380 |  http://etcd-0.etcd-service:2379 |      false |
| 7c5422fe4922f16f | started | etcd-2 | http://etcd-2.etcd-headless:2380 | http://etcd-2.etcd-headless:2379 |      false |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
### 删除这个 pod
# kubectl delete -f new_member.yaml 
pod "new-member" deleted
```

### 完成
- 删除节点的污点
- etcd-0 正常调度
- 验证状态

```shell
# kubectl taint node node4 dedicated-
node/node4 untainted
# Pod 已经正常运行
# kubectl -n etcd-test get pods etcd-0 
NAME     READY   STATUS    RESTARTS   AGE
etcd-0   1/1     Running   1          34m
# member 正常
# kubectl -n etcd-test exec -it etcd-0 -- etcdctl member list -w table
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
|        ID        | STATUS  |  NAME  |            PEER ADDRS            |           CLIENT ADDRS           | IS LEARNER |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
| 3f54ac025181a433 | started | etcd-1 | http://etcd-1.etcd-headless:2380 | http://etcd-1.etcd-headless:2379 |      false |
| 788e82417cb654eb | started | etcd-0 | http://etcd-0.etcd-headless:2380 | http://etcd-0.etcd-headless:2379 |      false |
| 7c5422fe4922f16f | started | etcd-2 | http://etcd-2.etcd-headless:2380 | http://etcd-2.etcd-headless:2379 |      false |
+------------------+---------+--------+----------------------------------+----------------------------------+------------+
# health 正常
# kubectl -n etcd-test exec -it etcd-0 -- etcdctl endpoint health --cluster -w table
+----------------------------------+--------+------------+-------+
|             ENDPOINT             | HEALTH |    TOOK    | ERROR |
+----------------------------------+--------+------------+-------+
| http://etcd-2.etcd-headless:2379 |   true | 2.536916ms |       |
| http://etcd-0.etcd-headless:2379 |   true | 2.552926ms |       |
| http://etcd-1.etcd-headless:2379 |   true | 2.771849ms |       |
+----------------------------------+--------+------------+-------+
# status 正常
# kubectl -n etcd-test exec -it etcd-0 -- etcdctl endpoint status --cluster -w table
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|             ENDPOINT             |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://etcd-1.etcd-headless:2379 | 3f54ac025181a433 |   3.5.0 |   20 kB |     false |      false |         3 |         42 |                 42 |        |
| http://etcd-0.etcd-headless:2379 | 788e82417cb654eb |   3.5.0 |   20 kB |     false |      false |         3 |         42 |                 42 |        |
| http://etcd-2.etcd-headless:2379 | 7c5422fe4922f16f |   3.5.0 |   20 kB |      true |      false |         3 |         42 |                 42 |        |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+


```

# 总结
> 理论上来说坏掉任意的一个节点都可以按照这样的方式来处理，但这个复杂的过程是否还有优化的空间呢？
