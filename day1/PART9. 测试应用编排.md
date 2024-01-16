# PART9. 测试应用编排

## 9.1 集群管理相关的其它常用操作

### 9.1.1 证书管理

kubeadm部署集群时设定的证书通常在一年后到期

- 检查证书是否过期:`kubeadm certs check-expiration`

```
root@longinus-master-1:~# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jan 15, 2025 03:16 UTC   364d            ca                      no      
apiserver                  Jan 15, 2025 03:16 UTC   364d            ca                      no      
apiserver-etcd-client      Jan 15, 2025 03:16 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jan 15, 2025 03:16 UTC   364d            ca                      no      
controller-manager.conf    Jan 15, 2025 03:16 UTC   364d            ca                      no      
etcd-healthcheck-client    Jan 15, 2025 03:16 UTC   364d            etcd-ca                 no      
etcd-peer                  Jan 15, 2025 03:16 UTC   364d            etcd-ca                 no      
etcd-server                Jan 15, 2025 03:16 UTC   364d            etcd-ca                 no      
front-proxy-client         Jan 15, 2025 03:16 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jan 15, 2025 03:16 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jan 13, 2034 03:16 UTC   9y              no      
etcd-ca                 Jan 13, 2034 03:16 UTC   9y              no      
front-proxy-ca          Jan 13, 2034 03:16 UTC   9y              no
```

- 手动更新证书:`kubeadm certs renew`
	- 提示:`kubeadm`会在控制平面升级时自动更新所有的证书

### 9.1.2 重置集群

 提示:**危险操作**,请务必再三确认是否必须要执行该操作,尤其是在管理生产环境时要更加注意
 
 - 命令:`kubeadm reset`
	- 负责尽最大努力还原通过`kubeadm init`或者`kubeadm join`命令对主机所作的更改
	- 一般需要配置`--cri-socket`选项使用

- 如果需要重置整个集群,一般要先reset各工作节点,而后再reset控制平面各节点,这与集群初始化的次序相反
- 最后还需一些清理操作,包括清理iptables规则或ipvs规则、删除相关的各文件等

### 9.1.3 集群升级

#### a. 注意事项

- 升级前,务必要备份所有的重要组件,例如存储在数据库中应用层面的状态等.但`kubeadm upgrade`并不会影响工作负载,它只会涉及Kubernetes集群的内部组件
- 必须禁用Swap

#### b. 整体步骤概览

- 先升级控制平面节点
- 而后再升级工作节点

#### c. 各节点升级的步骤简介

- 升级`kubeadm`,但升级控制平面的第一个节点和升级控制平面的其它节点以及各工作节点的命令有所不同
- 排空节点,而后升级kubelet和kubectl

#### d. 具体的升级步骤

[官方文档](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

### 9.1.4 令牌过期后向集群中添加新节点

#### 9.1.4.1 方式一

##### a. 生成新token

生成新token命令:`kubeadm token create`

##### b. 获取取CA证书的hash编码(SHA256)

获取取CA证书的hash编码命令:`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`

##### c. 将节点加入集群

`kubeadm join kubeapi.magedu.com:6443 --token TOKEN --discovery-token-ca-cert-hash sha256:HASH`

#### 9.1.4.2 方式二

##### a. 直接生成将节点加入集群的命令

`kubeadm token create --print-join-command`

##### b. 先生成token再生成命令

- step1. 生成token:`TOKEN=$(kubeadm token generate)`
- step2. 生成命令:`kubeadm token create ${TOKEN} --print-join-command`

#### 9.1.4.3 添加控制平面节点

##### a. 先上传CA证书，并生成hash

`kubeadm init phase upload-certs --upload-certs`

##### b. 生成添加控制平面节点的命令

`kubeadm token create --print-join-command --certificate-key $CERT_HASH`

## 9.2 API资源规范

### 9.2.1 核心资源类型

Kubernetes API Primitive:

- 用于描述在Kubernetes上运行应用程序的基本组件,即俗称的Kubernetes对象(Object)
- 它们持久存储于API Server上,用于描述集群的状态

依据资源的主要功能作为分类标准,Kubernetes的API对象大体可分为如下几个类别:

- 工作负载(Workload)
- 服务发现和负载均衡(Discovery & LB)
- 配置和存储(Config & Storage)
- 集群(Cluster)
- 元数据(Metadata)

"以应用为中心":

- Kubernetes API Primitive基本都是围绕一个核心目的而设计:**如何更好地运行和丰富Pod资源,从而为容器化应用提供更灵活和更完善的操作与管理组件**

这里我们举一个例子:

```
root@longinus-master-1:~# kubectl create deployment demoapp --image=ikubernetes/demoapp:v1.0 --replicas=3 --dry-run=client -o yaml
```

其中:

- `demoapp`: 资源名称
- `--image`: 指定要使用的镜像
- `--replicas`:指定副本数
- `--dry-run`:不运行Pod,仅测试Pod能否被顺利部署
- `-o yaml`:以yaml格式输出结果

其输出内容如下:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: demoapp
  name: demoapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoapp
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: demoapp
    spec:
      containers:
      - image: ikubernetes/demoapp:v1.0
        name: demoapp
        resources: {}
status: {}
```

这就是所谓的资源规范.kubectl就是根据这个资源规范来创建资源对象,并提交给API Server的.换言之,这就是一份**资源对象的配置定义示例**.K8S中大部分资源类型都是遵循同样的语法规范的,基本上都有5个1级字段:

### 9.2.2 资源类型的一级字段

#### 9.2.2.1 apiVersion字段与kind字段

`apiVersion`字段与`kind`字段称为类型元数据

其中:

- `apiVersion`:用于指定API所属的群组和版本号
- `kind`:表示资源类型

#### 9.2.2.2 spec字段与status字段

- `spec`:用于指明期望状态
- `status`:表示实际状态

`spec`字段是我们用户定义的;`status`字段是Controller字段生成资源后回填的,不需要我们来定义

这里我们将资源创建出来,再来看它的status字段:

```
root@longinus-master-1:~# kubectl create deployment demoapp --image=ikubernetes/demoapp:v1.0 --replicas=3
deployment.apps/demoapp created
```

```
root@longinus-master-1:~# kubectl get deployment demoapp -o yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2024-01-16T10:05:07Z"
  generation: 1
  labels:
    app: demoapp
  name: demoapp
  namespace: default
  resourceVersion: "39644"
  uid: 37548fdf-849b-4d5c-a821-f407278fdaec
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: demoapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: demoapp
    spec:
      containers:
      - image: ikubernetes/demoapp:v1.0
        imagePullPolicy: IfNotPresent
        name: demoapp
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2024-01-16T10:05:49Z"
    lastUpdateTime: "2024-01-16T10:05:49Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2024-01-16T10:05:07Z"
    lastUpdateTime: "2024-01-16T10:05:49Z"
    message: ReplicaSet "demoapp-7c58cd6bb" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

可以看到status字段是有内容的

#### 9.2.2.3 metadata字段

- `metadata`:对象元数据.表示当前对象的属性.例如当前对象的名称、所属的namespace等

注意:API Server不支持yaml格式的资源定义;仅支持JSON格式的资源定义.但是为了简化用户的操作,API Server设计让用户可以提交一个yaml,由API Server转换成JSON