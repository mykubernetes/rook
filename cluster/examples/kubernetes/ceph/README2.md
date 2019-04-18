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
Ceph文件系统，我们一般称为cephfs  
1、创建 CephFileSystem  
基于Rook创建Ceph块设备，需要先创建CephFilesystem。  
```
$ vim busy-box-fs.yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: busy-box-fs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - failureDomain: osd
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
    placement:
    resources:

# 干掉之前 busy-box Pod容器
$ kubectl delete pod/busy-box-test2
pod "busy-box-test2" deleted

$ kubectl create -f busy-box-fs.yaml 
cephfilesystem.ceph.rook.io/busy-box-fs created
$ kubectl -n rook-ceph get cephfilesystem 
NAME          MDSCOUNT   AGE
busy-box-fs   1          30s
$ kubectl -n rook-ceph get pod -l app=rook-ceph-mds
NAME                                          READY   STATUS    RESTARTS   AGE
rook-ceph-mds-busy-box-fs-a-d8955cd94-wthsc   1/1     Running   0          98s
rook-ceph-mds-busy-box-fs-b-8f5b6cf8-b27q9    1/1     Running   0          97s
```  
注意：这里 spec.metadataPool.replicated.size 字段和 spec.dataPools.replicated.size 配置为集群 OSD 个数。可以看到，默认会创建两个相关 MDS：rook-ceph-mds-busy-box-fs，同时他还会在底层创建两个 Pool：busy-box-fs-metadata 元数据和 busy-box-fs-data0 数据，这点跟之前 初试 Ceph 存储之块设备、文件系统、对象存储 #3、Ceph 文件系统 文章中通过 CLI 命令操作是一致的，这里自动帮我们创建了。  

2、验证并测试  
进入到 rook-toolbox 容器内通过 CLI 命令查看验证一下。  
```
$ kubectl -n rook-ceph exec -it rook-ceph-tools-5bd5cdb949-d22ql bash
[root@node2 /]# ceph status
  cluster:
    id:     6c117372-a462-447c-bfd4-a0378393f69e
    health: HEALTH_WARN
            clock skew detected on mon.a, mon.c
 
  services:
    mon: 3 daemons, quorum b,a,c
    mgr: a(active)
    mds: busy-box-fs-1/1/1 up  {0=busy-box-fs-b=up:active}, 1 up:standby-replay
    osd: 3 osds: 3 up, 3 in
 
  data:
    pools:   4 pools, 364 pgs
    objects: 38  objects, 14 MiB
    usage:   21 GiB used, 75 GiB / 96 GiB avail
    pgs:     364 active+clean
 
  io:
    client:   1.2 KiB/s rd, 2 op/s rd, 0 op/s wr
 
[root@node2 /]# ceph fs ls
name: busy-box-fs, metadata pool: busy-box-fs-metadata, data pools: [busy-box-fs-data0 ]
[root@node2 /]# ceph mds stat
busy-box-fs-1/1/1 up  {0=busy-box-fs-b=up:active}, 1 up:standby-replay
```  
确认自动创建了，接下来我们创建一个实例来挂载 fs 到指定目录 /mnt/busy-box/fs。  
```
$ vim busy-box-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: busy-box-test3
  namespace: default
  labels:
    k8s-app: busy-box-test3
spec:
  replicas: 3
  selector:
    k8s-app: busy-box-test3
  template:
    metadata:
      labels:
        k8s-app: busy-box-test3
        kubernetes.io/cluster-service: "true"
    spec:
      restartPolicy: Always
      containers:
      - name: busy-box-test3
        image: busybox
        volumeMounts:
        - name: busy-box-test-fs
          mountPath: /mnt/busy-box/fs
        command: ["sleep", "60000"]
      volumes:
      - name: busy-box-test-fs
        flexVolume:
          driver: ceph.rook.io/rook
          fsType: ceph
          options:
              fsName: busy-box-fs
              clusterNamespace: rook-ceph 

$ vim busy-box-rc.yaml
$ kubectl create -f busy-box-rc.yaml 
replicationcontroller/busy-box-test3 created
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE
busy-box-test3-4lm9n                1/1     Running   0          31s     172.20.1.9    10.222.78.64   <none>
busy-box-test3-5gc4x                1/1     Running   0          31s     172.20.2.14   10.222.78.86   <none>
busy-box-test3-kbm8c                1/1     Running   0          31s     172.20.0.17   10.222.78.63   <none>             
```  
注意：这里 fsName：busy-box-fs 要跟上边的 CephFilesystem Name 一致。replicas: 3 这个我设置成跟集群节点个数一致，这样可以分配到每个节点上，方便下边验证。  

分别进入到各个容器内，查看验证一下。  
```
# 进入 busy-box-test3-4lm9n 容器内查看
$ kubectl exec -it busy-box-test3-4lm9n /bin/sh
/ # cat /etc/hostname 
busy-box-test3-4lm9n
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  32.0G      7.9G     24.0G  25% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     1.1G         0      1.1G   0% /sys/fs/cgroup
10.68.144.184:6790,10.68.107.193:6790,10.68.253.125:6790:/
                         22.3G         0     22.3G   0% /mnt/busy-box/fs
/dev/mapper/cl-root      32.0G      7.9G     24.0G  25% /dev/termination-log
/dev/mapper/cl-root      32.0G      7.9G     24.0G  25% /etc/resolv.conf
/dev/mapper/cl-root      32.0G      7.9G     24.0G  25% /etc/hostname
......

# 写入测试文件
/ # echo "This message write from busy-box-test3-4lm9n." > /mnt/busy-box/fs/message.txt
```  
然后，我们进入到 busy-box-test3-5gc4x 容器内查看下。  
```
# 进入到 busy-box-test3-5gc4x 查看下
$ kubectl exec -it busy-box-test3-5gc4x /bin/sh
/ # cat /etc/hostname 
busy-box-test3-5gc4x
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  32.0G      5.1G     26.9G  16% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     1.1G         0      1.1G   0% /sys/fs/cgroup
10.68.144.184:6790,10.68.107.193:6790,10.68.253.125:6790:/
                         22.3G         0     22.3G   0% /mnt/busy-box/fs
/dev/mapper/cl-root      32.0G      5.1G     26.9G  16% /dev/termination-log
/dev/mapper/cl-root      32.0G      5.1G     26.9G  16% /etc/resolv.conf
/dev/mapper/cl-root      32.0G      5.1G     26.9G  16% /etc/hostname
......

/ # cat /mnt/busy-box/fs/message.txt 
This message write from busy-box-test3-4lm9n.
/ # echo "This message write from busy-box-test3-5gc4x." >> /mnt/busy-box/fs/message.txt 
/ # cat /mnt/busy-box/fs/message.txt 
This message write from busy-box-test3-4lm9n.
This message write from busy-box-test3-5gc4x.
```  

Object 对象存储
=============
1、创建一个 CephObjectStore，名称为：busy-box-obj。  
```
$ vim busy-box-obj.yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: busy-box-obj
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: osd
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  gateway:
    type: s3
    sslCertificateRef:
    port: 80
    securePort:
    instances: 1
    allNodes: false
    placement:
    resources:

$ kubectl create -f busy-box-obj.yaml 
cephobjectstore.ceph.rook.io/busy-box-obj created
$ kubectl -n rook-ceph get cephobjectstore
NAME           AGE
busy-box-obj   25s
$ kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
NAME                                          READY   STATUS    RESTARTS   AGE
rook-ceph-rgw-busy-box-obj-545b8b8b6b-j84wq   1/1     Running   0          21s
$ kubectl -n rook-ceph get svc -l app=rook-ceph-rgw
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
rook-ceph-rgw-busy-box-obj   ClusterIP   10.68.117.190   <none>        80/TCP    2m7s
```  
注意：它会自动创建一个 rook-ceph-rgw，并且创建一个 Service。默认使用 s3 存储类型，端口 80。这里 metadataPool.replicated.size 字段配置为集群 OSD 个数。如果 Ceph 集群 OSD 数量 >=3，dataPool 字段配置如上，否则配置如下：  
```
dataPool:
    failureDomain: osd
    replicated:
      size: 1
```




