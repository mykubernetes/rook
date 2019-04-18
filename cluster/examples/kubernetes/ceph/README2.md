Block 块存储
==============
1、创建 CephBlockPool、StorageClass
基于 Rook 创建 Ceph 块设备，需要先创建 CephBlockPool 和 StorageClass，CephBlockPool 这是 Rook 的 CRD 自定义资源，顾名思义就是创建一个 Block 池，然后在创建一个 StorageClass，这个是动态卷配置 (Dynamic provisioning) 可以根据需要动态的创建存储卷，Kubernetes 集群存储 PV 支持 Static 静态配置以及 Dynamic 动态配置。  

```
$ vim storageclass.yaml 
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicapool
  clusterNamespace: rook-ceph
  fstype: xfs

$ kubectl create -f storageclass.yaml 
cephblockpool.ceph.rook.io/replicapool created
storageclass.storage.k8s.io/rook-ceph-block created
$ kubectl get sc
NAME              PROVISIONER          AGE
rook-ceph-block   ceph.rook.io/block   9s
$ kubectl -n rook-ceph get cephblockpool 
NAME          AGE
replicapool   1m
```  
