# PART5. 安全上下文

## 5.1 概要

Pod及容器的安全上下文

- Pod及容器的安全上下文是一组用来决定容器是如何创建和运行的**约束条件**,这些条件代表创建和运行容器时使用的运行时参数
- Pod及容器的安全上下文给了用户为Pod或容器定义特权和访问控制的机制

在K8S上创建一个Pod时:

- 能否将Pod内的容器运行为特权模式,以便于让该容器可以操作Root Namespace?
- 或者能否在容器中运行`iptables`命令,以便操作内核中的iptables规则?

以上操作都是内核的特权级操作,这些特权级操作中,有些操作会影响到同一Pod内的其他容器,有些操作则会影响到其他Pod.为了安全起见,默认情况下关闭了这些操作.但仍然允许我们人为手动打开这些操作.

因此通常会在整个集群级别施加一些规则,要求用户创建Pod时不能使用某些特权级操作(例如不能创建特权级容器/不能直接共享宿主机的Network Namespace等).

假定我们自己就是集群管理员,且我们不会做一些高风险操作,也不打算防范其他用户,那么我们就不需要定义这些全局的规则.但是在个别的Pod上,可能出现的情况是:

这个Pod中运行的进程来自某个未经充分测试的开源程序.我们假定这个程序中存在一些危及整个集群的bug.这时我们就需要遵循最小化权限法则,降低该Pod中的进程可以获得的权限.而安全上下文(Security Context)就是用于给Pod和容器做削权和提权的.

因此这个场景下,给这个Pod定义Security Context即可.

当然权限仅仅是Security Context的一种管控逻辑,Security Context还有其他管控逻辑.

Pod和容器的安全上下文设置主要包括以下几个方面

- 自主访问控制DAC
- **容器进程运行身份及资源访问权限设定**
- **Linux Capabilities**:操作Linux内核的能力设定
- seccomp:安全盔甲设定
- AppArmor:安全方面的设定
- SELinux:安全方面的设定
- Privileged Mode:特权模型(升级为特权模型)
- Privilege Escalation:权限升级

再次提醒,以上功能都是有风险的操作

安全上下文的配置分为2个级别:

- Pod:在Pod级别上设定,表示安全上下文的配置对Pod内的所有容器都生效
- 容器:表示安全上下文的配置对单个容器生效

注意:**Pod级别和容器级别所支持的配置是不同的**

```
root@longinus-master-1:~/k8s-yaml# kubectl explain pods.spec.securityContext
KIND:       Pod
VERSION:    v1

FIELD: securityContext <PodSecurityContext>
...
```

```
root@longinus-master-1:~/k8s-yaml# kubectl explain pods.spec.containers.securityContext
KIND:       Pod
VERSION:    v1

FIELD: securityContext <SecurityContext>
```

## 5.2 示例

默认在容器内是不具备修改内核的iptables的能力的,这里我们随便找一个之前创建的容器看一下:

```
[root@pod-demo /]# iptables -vnL
iptables v1.8.3 (legacy): can't initialize iptables table `filter': Permission denied (you must be root)
Perhaps iptables or your kernel needs to be upgraded.
```

但是我们如果在Security Context中赋予用户`NET_ADMIN`的能力,就可以在容器中修改iptables规则:

```
root@longinus-master-1:~/k8s-yaml# vim securitycontext-capabilities-demo.yaml
root@longinus-master-1:~/k8s-yaml# cat securitycontext-capabilities-demo.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: securitycontext-capabilities-demo
  namespace: demo
spec:
  containers:
    - name: demo
      image: ikubernetes/demoapp:v1.0
      imagePullPolicy: IfNotPresent
      # 执行一个修改iptables的命令
      command:
      - "/bin/sh"
      - "-c"
      args: 
      - "/sbin/iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80 && /usr/bin/python3 /usr/local/bin/demo.py"
      securityContext:
        capabilities:
          add:
            # 修改iptables需要赋予用户网络管理员的权限
            # 默认没有赋予容器内用户的这个能力 也就是说默认是无法修改iptables的
          - "NET_ADMIN"
          drop:
          - "CHOWN"
```

```
root@longinus-master-1:~/k8s-yaml# kubectl apply -f securitycontext-capabilities-demo.yaml
pod/securitycontext-capabilities-demo created
```

```
root@longinus-master-1:~/k8s-yaml# kubectl get pods -n demo -o wide
NAME                                READY   STATUS    RESTARTS       AGE    IP            NODE              NOMINATED NODE   READINESS GATES
securitycontext-capabilities-demo   1/1     Running   0              25s    10.244.1.23   longinus-node-1   <none>           <none>
```

```
root@longinus-master-1:~/k8s-yaml# kubectl exec -it securitycontext-capabilities-demo -n demo -- /bin/sh
[root@securitycontext-capabilities-demo /]# iptables -vnL
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

可以看到,此时就可以查看和修改iptables规则了.

```
[root@securitycontext-capabilities-demo /]# touch a.txt
[root@securitycontext-capabilities-demo /]# ls -lh a.txt 
-rw-r--r--    1 root     root           0 Feb 14 06:51 a.txt
[root@securitycontext-capabilities-demo /]# chown bin a.txt 
chown: a.txt: Operation not permitted
```

但是,由于我们在Security Context中取消了用户`CHOWN`的能力,因此用户没有权限修改文件的属主属组了.