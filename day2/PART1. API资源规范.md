# PART1. API资源规范

还是以上节课的demoapp deployment为例,讲解各字段的含义.

```yaml
root@longinus-master-1:~# kubectl get deployment demoapp -o yaml
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
  resourceVersion: "120532"
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
  - lastTransitionTime: "2024-01-16T10:05:07Z"
    lastUpdateTime: "2024-01-16T10:05:49Z"
    message: ReplicaSet "demoapp-7c58cd6bb" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2024-01-18T07:54:58Z"
    lastUpdateTime: "2024-01-18T07:54:58Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

## 1.1 apiVersion

API群组名和版本号.格式为:`Group_Name/Version`

版本分为:

- alpha:内测版.默认是看不到的
- beta:公测版.能看到,格式为`vNbetaM`,例如`v1beta1`
- stable:稳定版.格式为:`vN`,例如`v1`/`v2`

查看当前集群中可用的API群组名和版本号:

```
root@longinus-master-1:~# kubectl api-versions
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1
autoscaling/v1
autoscaling/v2
batch/v1
certificates.k8s.io/v1
coordination.k8s.io/v1
discovery.k8s.io/v1
events.k8s.io/v1
flowcontrol.apiserver.k8s.io/v1beta2
flowcontrol.apiserver.k8s.io/v1beta3
networking.k8s.io/v1
node.k8s.io/v1
policy/v1
rbac.authorization.k8s.io/v1
scheduling.k8s.io/v1
storage.k8s.io/v1
v1
```

其中`v1`这个群组是核心群组,其名称实际上叫`core`,但名称是被省略的

## 1.2 kind

资源类型名称

`apiVersion`和`kind`表示**类型元数据**

查看当前集群中所有的资源类型:

```
root@longinus-master-1:~# kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
controllerrevisions                            apps/v1                                true         ControllerRevision
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
selfsubjectreviews                             authentication.k8s.io/v1               false        SelfSubjectReview
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling/v2                         true         HorizontalPodAutoscaler
cronjobs                          cj           batch/v1                               true         CronJob
jobs                                           batch/v1                               true         Job
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
leases                                         coordination.k8s.io/v1                 true         Lease
endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice
events                            ev           events.k8s.io/v1                       true         Event
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta3   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta3   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
ingresses                         ing          networking.k8s.io/v1                   true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
poddisruptionbudgets              pdb          policy/v1                              true         PodDisruptionBudget
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1           true         Role
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
csistoragecapacities                           storage.k8s.io/v1                      true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```

在配置清单中需要填入的是`KIND`字段的值

- NAME:资源类型名.以复数形式出现
- SHORTNAMES:简短名称
- APIVERSION:资源所属群组
- NAMESPACE:是否命名空间级别.false表示资源为集群级别
- KIND:实际上就是配置清单中,kind字段的值

## 1.3 metadata

对象元数据

### 1.3.1 name

`metadata.name`表示对象的名称.是对象的唯一标识.同一资源类型下的对象间,该字段值唯一.且该字段的作用域受限于资源类型的隶属的命名空间.

K8S中资源类型分为2个级别:

- 集群级别:资源对象在整个集群范围内生效

例如kube-system命名空间中的资源对象:

```
root@longinus-master-1:~# kubectl get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-5dd5756b68-72nbc                    1/1     Running   0          4m8s
coredns-5dd5756b68-g4gfh                    1/1     Running   0          4m23s
etcd-longinus-master-1                      1/1     Running   0          2d5h
kube-apiserver-longinus-master-1            1/1     Running   0          2d5h
kube-controller-manager-longinus-master-1   1/1     Running   0          2d5h
kube-proxy-4s6vr                            1/1     Running   0          2d5h
kube-proxy-j4bkt                            1/1     Running   0          2d5h
kube-proxy-rmszf                            1/1     Running   0          2d5h
kube-proxy-tblv7                            1/1     Running   0          2d5h
kube-scheduler-longinus-master-1            1/1     Running   0          2d5h
```
	
- 命名空间级别:K8S将整个集群划分为多个不同的、用于隔离K8S中对象名称的**逻辑组**.主要用于支持多租户、多项目

### 1.3.2 namespace

若资源对象属于命名空间级别,则资源清单中有该字段,用于指明当前资源对象的所属命名空间.若不指明,则默认为default命名空间

### 1.3.3 labels

标签集,用于支撑标签选择器.由K-V组成.该字段是一个map型数据,大致格式如下:

```yaml
metadata:
  labels:
    key1: value1
    key2: value2
```

### 1.3.4 annotations

注解集,表示注释信息.也是K-V组成.与labels字段不同之处在于,labels字段用于支撑标签选择器,用于被挑选和过滤;annotations字段就是一些说明信息

## 1.4 spec

由用户声明的,对该资源对象的**期望状态**.是英文单词specification的缩写

如果该资源对象的类型为数据支撑类型的组件(有数据即可,不需要控制器再做任何操作),**通常**(注意是通常不是绝对)没有spec字段

spec字段属于不同字段时,其内容是不同的

查看指定资源类型的资源规范:`kubectl explain 类型名称`

```
root@longinus-master-1:~# kubectl explain Deployment
GROUP:      apps
KIND:       Deployment
VERSION:    v1

DESCRIPTION:
    Deployment enables declarative updates for Pods and ReplicaSets.
    
FIELDS:
  apiVersion	<string>
    APIVersion defines the versioned schema of this representation of an object.
    Servers should convert recognized schemas to the latest internal value, and
    may reject unrecognized values. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

  kind	<string>
    Kind is a string value representing the REST resource this object
    represents. Servers may infer this from the endpoint the client submits
    requests to. Cannot be updated. In CamelCase. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

  metadata	<ObjectMeta>
    Standard object's metadata. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

  spec	<DeploymentSpec>
    Specification of the desired behavior of the Deployment.

  status	<DeploymentStatus>
    Most recently observed status of the Deployment.
```

注意类型名称是可以嵌套的:

```
root@longinus-master-1:~# kubectl explain Deployment.spec
GROUP:      apps
KIND:       Deployment
VERSION:    v1

FIELD: spec <DeploymentSpec>

DESCRIPTION:
    Specification of the desired behavior of the Deployment.
    DeploymentSpec is the specification of the desired behavior of the
    Deployment.
    
FIELDS:
  minReadySeconds	<integer>
    Minimum number of seconds for which a newly created pod should be ready
    without any of its container crashing, for it to be considered available.
    Defaults to 0 (pod will be considered available as soon as it is ready)

  paused	<boolean>
    Indicates that the deployment is paused.

  progressDeadlineSeconds	<integer>
    The maximum time in seconds for a deployment to make progress before it is
    considered to be failed. The deployment controller will continue to process
    failed deployments and a condition with a ProgressDeadlineExceeded reason
    will be surfaced in the deployment status. Note that progress will not be
    estimated during the time a deployment is paused. Defaults to 600s.

  replicas	<integer>
    Number of desired pods. This is a pointer to distinguish between explicit
    zero and not specified. Defaults to 1.

  revisionHistoryLimit	<integer>
    The number of old ReplicaSets to retain to allow rollback. This is a pointer
    to distinguish between explicit zero and not specified. Defaults to 10.

  selector	<LabelSelector> -required-
    Label selector for pods. Existing ReplicaSets whose pods are selected by
    this will be the ones affected by this deployment. It must match the pod
    template's labels.

  strategy	<DeploymentStrategy>
    The deployment strategy to use to replace existing pods with new ones.

  template	<PodTemplateSpec> -required-
    Template describes the pods that will be created. The only allowed
    template.spec.restartPolicy value is "Always".
```

## 1.5 status

由控制器回填的,该资源对象的**实际状态**

如果该资源对象的类型为数据支撑类型的组件(有数据即可,不需要控制器再做任何操作),**通常**(注意是通常不是绝对)没有status字段

因为这种资源对象不需要控制器去分别该对象的实际状态与期望状态

## 1.6 API资源对象管理的方式

### 1.6.1 指令式命令

- 直接作用于集群上的活动对象
- 适合在开发环境中完成一次性的操作任务

说白了就是通过命令行选项来定义要管理的资源对象,一般就是新手用,因为功能太有限了

### 1.6.2 指令式对象配置

- 基于资源配置文件执行对象管理操作,但只能独立引用每个配置清单文件
- 可用于生产环境的管理任务

说白了就是`kubectl COMMAND -f FILE/DIR`,就是以配置文件(资源清单)的形式完成资源对象的声明

这里的COMMAND实际上就是增删改查操作,明确能够表达自己要干啥的操作:

- create
- delete
- get
- edit
- replace

### 1.6.3 声明式对象配置

- 基于配置文件执行对象管理操作
- 可直接引用目录下的所有配置清单文件，也可直接作用于单个配置文件

这种方式也是隐含了增删改的意义

`kubectl apply -f FILE/DIR`

### 1.6.4 资源对象的CRUD

#### 1.6.4.1 增

建议采用配置文件的方式,因为可以将配置文件纳入版本库,利于管理

`kubectl create`

#### 1.6.4.2 查

- `kubectl get TYPE NAME1 NAME2...`
- `kubectl get TYPE1 NAME1 TYPE2 NAME2...`
- `kubectl get -f FILE`

其中,TYPE表示资源对象类型;NAME表示资源对象的名称

##### a. 指定输出格式

- `-o`选项:
	- `wide`:显示更多字段
	- `yaml`:以yaml格式显示
	- `json`:以json格式显示
	- `jsonpath`:过滤某些字段

`kubectl describe`:描述对象的详细信息.实际上就是将`kubectl get`中的status字段格式化成更易读的方式显示

- `kubectl describe TYPE NAME1 NAME2...`
- `kubectl describe TYPE1 NAME1 TYPE2 NAME2...`
- `kubectl describe -f FILE`

##### b. 监视输出

- `-w`:监视输出

##### c. 显示对象上的标签

- `showlabels`:显示对象上的标签

#### 1.6.4.3 删

- `kubectl delete TYPE NAME1 NAME2...`
- `kubectl delete TYPE1 NAME1 TYPE2 NAME2...`
- `kubectl delete -f FILE`

#### 1.6.4.4 改

注意,这里的改是指在当前的活动对象上修改,不太建议使用这种方式.最好将配置文件纳入版本库

- `kubectl edit TYPE NAME`

## 1.7 以创建Deployment为例演示

### 1.7.1 以指令式命令形式仅打印资源清单

```
root@longinus-master-1:~# kubectl create deployment nginx --image=nginx:1.22 --replicas=3 --dry-run=client -o yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.22
        name: nginx
        resources: {}
status: {}
```

保存输出为yaml文件:

```
root@longinus-master-1:~# mkdir k8s-yaml
root@longinus-master-1:~# cd k8s-yaml/
root@longinus-master-1:~/k8s-yaml# kubectl create deployment nginx --image=nginx:1.22 --replicas=3 --dry-run=client -o yaml > deploy-nginx.yaml
```

### 1.7.2 以声明式对象配置的方式创建Pod

#### 1.7.2.1 编辑配置文件

其实配置文件中所有值为空对象(`{}`)和`null`的字段都是可以删除的,删除后配置文件如下:

```
root@longinus-master-1:~/k8s-yaml# vim deploy-nginx.yaml 
root@longinus-master-1:~/k8s-yaml# cat deploy-nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.22
        name: nginx
```

#### 1.7.2.2 创建Pod

```
root@longinus-master-1:~/k8s-yaml# kubectl apply -f deploy-nginx.yaml 
deployment.apps/nginx created
```

#### 1.7.2.3 查看Pod

```
root@longinus-master-1:~/k8s-yaml# kubectl get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE    IP           NODE              NOMINATED NODE   READINESS GATES
demoapp-7c58cd6bb-22pjs   1/1     Running   0          2d7h   10.244.1.3   longinus-node-1   <none>           <none>
demoapp-7c58cd6bb-blnzl   1/1     Running   0          2d7h   10.244.3.3   longinus-node-3   <none>           <none>
demoapp-7c58cd6bb-srf5g   1/1     Running   0          2d7h   10.244.2.3   longinus-node-2   <none>           <none>
nginx-6bbc57b98d-n9l46    1/1     Running   0          98s    10.244.2.5   longinus-node-2   <none>           <none>
nginx-6bbc57b98d-nknpg    1/1     Running   0          98s    10.244.1.6   longinus-node-1   <none>           <none>
nginx-6bbc57b98d-smfhz    1/1     Running   0          98s    10.244.3.6   longinus-node-3   <none>           <none>
```