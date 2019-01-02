---
title: k8s-schedule
update: 2018-05-24 11:16:23
tags: kubernetes
categories: cloud
---

#### Labels and Selectors
1. label
  label是k8s中所有资源都有的一个域, 他是一个<key>=<value>对, 表示这个资源具有某种特定属性, key必须不能重复. 这样user和其他资源可以通过label来选择具有这种属性的资源集合.
```yaml
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```
  label的key有两个段,分别是前缀和名称, 通过'/'分开. 名称字段必须少于63个字符, 必须以'[a-z0-9A-Z]', '-', '.', '_'和数字 开头和结尾.前缀是可选的, 一旦指定, 不能超过253个字符. label的value指和key中的名称要求一样.

2. Selectors的使用
通过api中使用 -l 来选择特定集合的objects: `$ kubectl get pods -l 'environment in (production, qa)'`

一些k8s资源, 比如Service and ReplicationController 会有selector字段, 用来选择pod.
```
selector:
    component: redis
```

基于相等的匹配和基于集合的匹配:
  ```yaml
  selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
  ```
   注意区分概念, label永远都是key=value, 多个label在一起能组成一个集合, 类似map. 只有selector有集合的操作, label本身就是一个<k,v> 对, 没有<k in v>这样的label.
   集合方式有四种operator: In, NotIn, Exist, DoesNotExist. `{key: tier, operator: In, values: [cache]}`这条表达式等价于matchLabels中的: `tier: cache`.
   必须满足所有的selector表达式, 才算匹配.
   Job, Deployment, Replica Set, and Daemon Set都是支持基于集合的匹配.

3. nodeSelector
  nodeSelector用于选择某个node, 再次之前, 需要在node上添加label: `kubectl label nodes <node-name> <label-key>=<label-value>`. 然后在pod中配置相应的nodeSelector, 就能保证pod调度到符合语义的node:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```
k8s node不同版本会有一些内置labels:
```
kubernetes.io/hostname
failure-domain.beta.kubernetes.io/zone
failure-domain.beta.kubernetes.io/region
beta.kubernetes.io/instance-type
beta.kubernetes.io/os
beta.kubernetes.io/arch
```

#### Affinity
nodeSelector是比较简单的pod选择节点的方式, k8s提供了affinity/anti-affinity用来更复杂的提供节点选择方案. 他有两种: node affinity 和 inter-pod affinity/anti-affinity:
* node affinity
nodeAffinity与nodeSelector相似, 也是基于label, 包括两种类型, 可以分别理解为'hard'和'soft', 一个强制要求, 一个尽可能要求. 
  - requiredDuringSchedulingIgnoredDuringExecution: hard类型
  - preferredDuringSchedulingIgnoredDuringExecution: soft类型
内容和label selector基本一致:
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```
`weight` 是1-100.

* pod affinity
    podAffinity/podAnti-Affinity是比较已经调度到节点上的pod而不是看node本身. 他也有两种类型hard和soft:
    - requiredDuringSchedulingIgnoredDuringExecution: hard
    - preferredDuringSchedulingIgnoredDuringExecution: soft
    
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

#### Taints and Tolerations

affinity是站在pod的角度, 而taints是站在node的角度, 
```
kubectl taint nodes node1 key=value:NoSchedule
```
这个命令将node1加了一个taint, 表示无法调度, 除非你有相应的toleration:

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```
```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```
这两个tolerations都能匹配那个taint. operator默认是`Equal`
一个空的tolerations匹配所有的taint:
```yaml
tolerations:
- operator: "Exists"
```
以下这个tolerations匹配所有key为`key`的taints:
```yaml
tolerations:
- key: "key"
  operator: "Exists"
```

effect有两种： NoExecute和ＮoSchedule， 如果加了NoExecute， 那么所有不匹配的pod将会立即被evicted掉， 如果加了Noschedule, 那么只是Pod无法被调度， 已经调度的Ｐod不受影响。NoExecute还可以指定一个可选的域`tolerationSeconds`， 表示尽管匹配了taint可以不被立即evicted掉, 但在一定时间之后就会被evicted掉。

通过`kubectl taint nodes node1 key:NoSchedule-` 取消taint

##### 更有意思的是多个taint和多个tolerations的情况.
例如创建三个taint:
```bash
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```
一个pod拥有两个tolerations：
```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```
该pod没有匹配所有的taint, 因此他不能被调度到这个节点， 但如果调度了，他不会被移除， 因为他只有第三个`NoSchedule`的taint不匹配。

##### 注意
在1.6之后， 官方加了一些内置的taint， 他会将pod evctied掉：
```
node.kubernetes.io/not-ready: notready.
node.alpha.kubernetes.io/unreachable: Unknown
node.kubernetes.io/out-of-disk: Node becomes out of disk.
node.kubernetes.io/memory-pressure: Node has memory pressure.
node.kubernetes.io/disk-pressure: Node has disk pressure.
node.kubernetes.io/network-unavailable: Node’s network is unavailable.
node.cloudprovider.kubernetes.io/uninitialized: When kubelet is started with “external” cloud provider, it sets this taint on a node to mark it as unusable. When a controller from the cloud-controller-manager initializes this node, kubelet removes this taint.
```