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
2、验证并测试  
```
$ vim busy-box-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: busybox-pvc
  namespace: default
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1024Mi

$ kubectl create -f busy-box-pvc.yaml 
persistentvolumeclaim/busybox-pvc created
$ kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
busybox-pvc   Bound    pvc-3e4adb48-12f0-11e9-9a38-0800272ff396   1Gi        RWO            rook-ceph-block   19s
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS      REASON   AGE
pvc-3e4adb48-12f0-11e9-9a38-0800272ff396   1Gi        RWO            Delete           Bound    default/busybox-pvc   rook-ceph-block            20s     
```  
注意：这里的 PV 会自动创建，当提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV，非常方便了。要注意 storageClassName: rook-ceph-block 要跟上边对应上。  

此时，可以进入到 rook-toolbox 容器内通过 CLI 命令查看验证一下。  
```
$ kubectl -n rook-ceph exec -it rook-ceph-tools-5bd5cdb949-d22ql bash
[root@node2 /]# rbd list -p replicapool
pvc-3e4adb48-12f0-11e9-9a38-0800272ff396
[root@node2 /]# rbd info -p replicapool pvc-3e4adb48-12f0-11e9-9a38-0800272ff396
rbd image 'pvc-3e4adb48-12f0-11e9-9a38-0800272ff396':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	id: 4ede6b8b4567
	block_name_prefix: rbd_data.4ede6b8b4567
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Tue Jan  8 02:51:08 2019
[root@node2 /]# ceph df
GLOBAL:
    SIZE       AVAIL      RAW USED     %RAW USED 
    96 GiB     75 GiB       21 GiB         21.96 
POOLS:
    NAME            ID     USED       %USED     MAX AVAIL     OBJECTS 
    test_pool       1         0 B         0        67 GiB           0 
    replicapool     2      14 MiB      0.02        67 GiB          16 
```  
现在就可以将申请的资源挂载到容器内使用了。创建一个基于busy-box镜像的Pod资源，将申请的资源挂载到 /mnt/busy-box 目录。  
```
$ vim busy-box-pod1.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busy-box-test1
  namespace: default
spec:
  restartPolicy: OnFailure
  containers:
  - name: busy-box-test1
    image: busybox
    volumeMounts:
    - name: busy-box-test-pv1
      mountPath: /mnt/busy-box
    command: ["sleep", "60000"]
  volumes:
  - name: busy-box-test-pv1
    persistentVolumeClaim:
      claimName: busybox-pvc

$ kubectl create -f busy-box-pod1.yaml 
pod/busy-box-test1 created
$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
busy-box-test1                      1/1     Running   0          9s
```  
创建完毕，进入到容器内查看验证一下，是否正确挂载。  
```
$ kubectl exec -it busy-box-test1 /bin/sh
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  32.0G      8.0G     23.9G  25% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     1.1G         0      1.1G   0% /sys/fs/cgroup
/dev/rbd0              1014.0M     32.3M    981.7M   3% /mnt/busy-box
/dev/mapper/cl-root      32.0G      8.0G     23.9G  25% /dev/termination-log
/dev/mapper/cl-root      32.0G      8.0G     23.9G  25% /etc/resolv.conf
......

/ # echo "This message write from busy-box-test1" > /mnt/busy-box/message.txt
/ # ls /mnt/busy-box/
message.txt
```  
可以看到申请的 1024Mi 存储挂载到了 /mnt/busy-box 目录，同时我们写入一个文件到该目录，来验证一下其他容器挂载同一 PVC 后，数据是否可以共享访问。  
```
# 先干掉之前 Pod，保留 PVC
$ kubectl delete pod/busy-box-test1 
pod "busy-box-test1" deleted

# 创建一个新的 Pod，并挂载同一 PVC
$ vim busy-box-pod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busy-box-test2
  namespace: default
spec:
  restartPolicy: OnFailure
  containers:
  - name: busy-box-test2
    image: busybox
    volumeMounts:
    - name: busy-box-test-pv2
      mountPath: /mnt/busy-box
    command: ["sleep", "60000"]
  volumes:
  - name: busy-box-test-pv2
    persistentVolumeClaim:
      claimName: busybox-pvc
      
$ kubectl create -f busy-box-pod2.yaml 
pod/busy-box-test2 created
$ kubectl get pods | grep busy-box
busy-box-test2                      1/1     Running   0          17s

# 进入到容器内，查看验证
$ kubectl exec -it busy-box-test2 /bin/sh
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  32.0G      8.0G     23.9G  25% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     1.1G         0      1.1G   0% /sys/fs/cgroup
/dev/rbd0              1014.0M     32.3M    981.7M   3% /mnt/busy-box
/dev/mapper/cl-root      32.0G      8.0G     23.9G  25% /dev/termination-log
/dev/mapper/cl-root      32.0G      8.0G     23.9G  25% /etc/resolv.conf
...

/ # cat /mnt/busy-box/message.txt 
This message write from busy-box-test1
```  
可以看到数据能够被读取到。  

File System 文件系统
===================
