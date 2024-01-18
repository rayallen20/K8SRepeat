# README

马永亮K8S课程笔记

## 备注

虚拟机挂起再恢复后,core-dns可能就会不断重启,因为无法连接到 Kubernetes API.

解决办法:

`kubectl delete pod coredns-xxx -n kube-system`

再让K8S重建core-dns即可