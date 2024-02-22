# PART2. Kubernetes常见的卷类型

## 2.1 emptyDir存储卷

- Pod对象上的一个临时目录
- 在Pod对象启动时即被创建,而在Pod对象被移除时一并被删除.这里的删除,是指卷的定义会被删除,同时卷背后定义的存储介质上的数据也会被清除

### 2.1.1 使用场景

emptyDir存储卷通常只能用于某些特殊场景中:

- 同一Pod内的多个容器间文件共享
- 作为容器数据的临时存储目录用于数据缓存系统

### 2.1.2 配置参数

- `medium`:此目录所在的存储介质的类型,可用值为`default`或`Memory`
	- `default`:kubelet在Pod节点所在宿主机上的磁盘上找一段空间
	- `Memory`:使用Pod节点所在宿主机上的内存作为存储卷
- `sizeLimit`:当前存储卷的空间限额,默认值为nil,表示不限制.通常在`medium`取值为`Memory`时需明确指定

```
root@longinus-master-1:~# kubectl explain pods.spec.volumes.emptyDir
KIND:       Pod
VERSION:    v1

FIELD: emptyDir <EmptyDirVolumeSource>
...
```

### 2.1.3 示例

```
root@longinus-master-1:~/k8s-yaml/day3# vim pods-with-emptydir-vol.yaml
root@longinus-master-1:~/k8s-yaml/day3# cat pods-with-emptydir-vol.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pods-with-emptydir-vol
  namespace: demo
spec:
  volumes:
    # 存储卷名称
  - name: data
    emptyDir:
      # 存储介质 default:磁盘 Memory:内存
      medium: Memory
      sizeLimit: 16Mi
  containers:
  - image: ikubernetes/admin-box:v1.2
    name: admin
    command: ["/bin/sh", "-c"]
    args: ["sleep 99999"]
    # 容器挂载点列表
    volumeMounts:
      # 指定要挂载的存储卷名称
    - name: data
      mountPath: /data
  - image: ikubernetes/demoapp:v1.0
    name: demoapp
    volumeMounts:
      # 此处2个容器挂载了同一个存储卷
    - name: data
      mountPath: /var/www/html
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl get pods pods-with-emptydir-vol -n demo -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
pods-with-emptydir-vol   2/2     Running   0          77s   10.244.2.34   longinus-node-2   <none>           <none>
```

#### step1. 在admin容器中向挂载点上写入数据

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-emptydir-vol -n demo -c admin -- /bin/sh
root@pods-with-emptydir-vol # cd data/
root@pods-with-emptydir-vol data# touch 1.txt
root@pods-with-emptydir-vol data# vi 1.txt 
root@pods-with-emptydir-vol data# cat 1.txt 
123
```

#### step2. 在demoapp容器中读取挂载点上的数据

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-emptydir-vol -n demo -c demoapp -- /bin/sh
[root@pods-with-emptydir-vol /]# cd /var/www/html/
[root@pods-with-emptydir-vol /var/www/html]# cat 1.txt 
123
```

#### step3. 删除并重建Pod,然后查看数据

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl delete -f pods-with-emptydir-vol.yaml 
pod "pods-with-emptydir-vol" deleted
root@longinus-master-1:~/k8s-yaml/day3# kubectl apply -f pods-with-emptydir-vol.yaml 
pod/pods-with-emptydir-vol created
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-emptydir-vol -n demo -c admin -- /bin/sh
root@pods-with-emptydir-vol data# ls -a
.   ..
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-emptydir-vol -n demo -c demoapp -- /bin/sh
[root@pods-with-emptydir-vol /]# cd /var/www/html/
[root@pods-with-emptydir-vol /var/www/html]# ls -a
.   ..
```

## 2.2 hostPath存储卷

- 将Pod所在节点上的文件系统的某目录用作存储卷
- 数据的生命周期与节点相同

想要复用hostPath存储卷上的数据,需要满足2个前提条件:

1. Pod被调度到相同的节点上
2. 节点本身没有宕机

### 2.2.1 配置参数

- `path`:指定工作节点上的目录路径,必须按字段
- `type`:指定节点上的存储类型

|取值|行为|
|:-:|:-:|
|DirectoryOrCreate|宿主机上若不存在给定的路径,则创建一个空目录(级联创建).权限为0755,与kubelet有相同的属组属组信息|
|Directory|给定路径上必须存在的目录|
|FileOrCreate|宿主机上若不存在给定的文件,则创建一个空文件.权限为0644,与kubelet有相同的属组属组信息|
|File|在给定路径上必须存在的文件|
|Socket|在给定路径上必须存在的UNIX套接字|
|CharDevice|在给定路径上必须存在的字符设备|
|BlockDevice|在给定路径上必须存在的块设备|

### 2.2.2 示例

```
root@longinus-master-1:~/k8s-yaml/day3# vim pods-with-hostpath-vol.yaml
root@longinus-master-1:~/k8s-yaml/day3# cat pods-with-hostpath-vol.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pods-with-hostpath-vol
  namespace: demo
spec:
  containers:
  - name: redis
    image: redis:7-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    hostPath:
      # 存储类型
      type: DirectoryOrCreate
      # 关联的宿主机目录路径
      path: /data/redis
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl apply -f pods-with-hostpath-vol.yaml
pod/pods-with-hostpath-vol created
root@longinus-master-1:~/k8s-yaml/day3# kubectl get pods pods-with-hostpath-vol -n demo -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
pods-with-hostpath-vol   1/1     Running   0          18s   10.244.2.35   longinus-node-2   <none>           <none>
```

#### step1. 进入容器创建一些K-V并持久化

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-hostpath-vol -n demo -- /bin/sh
/data # redis-cli
127.0.0.1:6379> set site www.baidu.com
OK
127.0.0.1:6379> BGSAVE
Background saving started
127.0.0.1:6379> exit
/data # ls
dump.rdb
```

#### step2. 在Pod所在的工作节点上查看该文件

```
root@longinus-node-2:~# ll /data/redis
total 12
drwxr-xr-x 2 lxd  root  4096 Feb 22 15:38 ./
drwxr-xr-x 3 root root  4096 Feb 22 15:29 ../
-rw------- 1 lxd  roach  113 Feb 22 15:38 dump.rdb
```

#### step3. 删除并重建Pod

##### case1. 重建后调度到相同的节点上

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl delete -f pods-with-hostpath-vol.yaml 
pod "pods-with-hostpath-vol" deleted
```

```
root@longinus-master-1:~/k8s-yaml/day3# vim pods-with-hostpath-vol.yaml
root@longinus-master-1:~/k8s-yaml/day3# cat pods-with-hostpath-vol.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pods-with-hostpath-vol
  namespace: demo
spec:
  # 指定要调度的目标节点名称
  nodeName: longinus-node-2
  containers:
  - name: redis
    image: redis:7-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    hostPath:
      # 存储类型
      type: DirectoryOrCreate
      # 关联的宿主机目录路径
      path: /data/redis
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl apply -f pods-with-hostpath-vol.yaml
pod/pods-with-hostpath-vol created
root@longinus-master-1:~/k8s-yaml/day3# kubectl get pods pods-with-hostpath-vol -n demo -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
pods-with-hostpath-vol   1/1     Running   0          17s   10.244.2.37   longinus-node-2   <none>           <none>
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-hostpath-vol -n demo -- /bin/sh
/data # redis-cli
127.0.0.1:6379> GET site
"www.baidu.com"
127.0.0.1:6379> exit
/data # exit
```

可以看到,之前保存的K-V还在

##### case2. 重建后调度到不同的节点上

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl delete -f pods-with-hostpath-vol.yaml
pod "pods-with-hostpath-vol" deleted
```

```
root@longinus-master-1:~/k8s-yaml/day3# vim pods-with-hostpath-vol.yaml
root@longinus-master-1:~/k8s-yaml/day3# cat pods-with-hostpath-vol.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pods-with-hostpath-vol
  namespace: demo
spec:
  # 指定要调度的目标节点名称
  nodeName: longinus-node-3
  containers:
  - name: redis
    image: redis:7-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    hostPath:
      # 存储类型
      type: DirectoryOrCreate
      # 关联的宿主机目录路径
      path: /data/redis
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl apply -f pods-with-hostpath-vol.yaml
pod/pods-with-hostpath-vol created
root@longinus-master-1:~/k8s-yaml/day3# kubectl get pods pods-with-hostpath-vol -n demo -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
pods-with-hostpath-vol   1/1     Running   0          63s   10.244.3.31   longinus-node-3   <none>           <none>
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pods-with-hostpath-vol -n demo -- /bin/sh
/data # redis-cli
127.0.0.1:6379> GET site
(nil)
127.0.0.1:6379> exit
/data # exit
```

可以看到,调度到不同节点上,就无法访问之前的数据了

### 2.3 NFS存储卷

[NFS搭建文档]()

#### 2.3.1 基本概念

nfs存储卷:将nfs服务器上导出(export)的文件系统用作存储卷.nfs是**文件系统级**共享服务,它支持多路挂载请求,可由多个Pod对象同时用作存储卷后端

#### 2.3.2 配置参数

- `server <string>`:NFS服务器的IP地址或主机名,必选字段
- `path <string>`:NFS服务器导出(共享)的文件系统路径,必选字段
- `readOnly <boolean>`:是否以只读方式挂载,默认为`false`

#### 2.3.3 网络卷的关联方式

1. **网络卷首先要由主机节点挂载,然后才能暴露在pod中**
2. **有些网络卷未必支持多主机节点同时挂载**

#### 2.3.4 示例

##### a. 部署Pod

```
root@longinus-master-1:~/k8s-yaml/day3# vim pod-with-nfs-vol.yaml
root@longinus-master-1:~/k8s-yaml/day3# cat pod-with-nfs-vol.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-nfs-vol
  namespace: demo
spec:
  containers:
  - name: redis
    image: redis:7-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    nfs:
      # pod定义中的volumes.nfs.server
      # 可以写nfs服务器的IP地址 也可以写域名
      server: 192.168.1.60
      path: /data/redis
      # 是否只读 默认为false
      readOnly: false
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl apply -f pod-with-nfs-vol.yaml
pod/pod-with-nfs-vol created
root@longinus-master-1:~/k8s-yaml/day3# kubectl get pods pod-with-nfs-vol -n demo -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
pod-with-nfs-vol   1/1     Running   0          41s   10.244.2.39   longinus-node-2   <none>           <none>
```

##### b. 在Pod内持久化数据

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pod-with-nfs-vol -n demo -- /bin/sh
/data # redis-cli
127.0.0.1:6379> SET host 192.168.1.100
OK
127.0.0.1:6379> BGSAVE
Background saving started
127.0.0.1:6379> exit
/data # ls
dump.rdb
```

##### c. 在nfs服务器上查看持久化的文件

```
root@nfs-server-1:~# ll /data/redis/
total 12
drwxr-xr-x 2 lxd  root  4096 Feb 22 13:53 ./
drwxr-xr-x 3 root root  4096 Feb 22 13:15 ../
-rw------- 1 lxd  roach  113 Feb 22 13:53 dump.rdb
```

##### d. 强制使Pod调度到其他节点上

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl delete -f pod-with-nfs-vol.yaml
pod "pod-with-nfs-vol" deleted
root@longinus-master-1:~/k8s-yaml/day3# vim pod-with-nfs-vol.yaml 
root@longinus-master-1:~/k8s-yaml/day3# cat pod-with-nfs-vol.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-nfs-vol
  namespace: demo
spec:
  nodeName: longinus-node-1
  containers:
  - name: redis
    image: redis:7-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: redisdata
      mountPath: /data
  volumes:
  - name: redisdata
    nfs:
      # pod定义中的volumes.nfs.server
      # 可以写nfs服务器的IP地址 也可以写域名
      server: 192.168.1.60
      path: /data/redis
      # 是否只读 默认为false
      readOnly: false
```

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl apply -f pod-with-nfs-vol.yaml
pod/pod-with-nfs-vol created
root@longinus-master-1:~/k8s-yaml/day3# kubectl get pods pod-with-nfs-vol -n demo -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
pod-with-nfs-vol   1/1     Running   0          19s   10.244.1.30   longinus-node-1   <none>           <none>
```

进入容器,确认K-V依然存在:

```
root@longinus-master-1:~/k8s-yaml/day3# kubectl exec -it pod-with-nfs-vol -n demo -- /bin/sh
/data # ls
dump.rdb
/data # redis-cli
127.0.0.1:6379> GET host
"192.168.1.100"
```

### 2.4 网络存储卷的问题

从这个例子中可以看出,网络存储卷有着极大的问题:

为了能够使用网络存储卷,你必须知道以下信息:

1. 网络存储卷背后具体的文件系统/块设备
2. 后端存储服务器的地址或域名
3. 该服务器上导出的具体目录
4. 存储服务的认证信息(例如账号密码,本例中为了简单起见没有涉及到这部分)

那么问题来了:如果用户只是希望在K8S上运行一个能够持久化保存数据的Pod,那么他必须了解后端的存储拓扑,还得理解存储的功能与特性.

换言之,**Pod与后端存储细节的耦合度过高了**.因此不建议Pod直接使用网络卷插件.