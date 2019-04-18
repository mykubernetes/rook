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

2、创建 User 用户  
然后创建一个 CephObjectStoreUser User 账户，来生成 AccessKey 和 SecretKey，为了后边该用户访问 S3 存储使用。  
```
$ vim busy-box-obj-user.yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: busy-box-obj-user
  namespace: rook-ceph
spec:
  store: busy-box-obj
  displayName: "busy-box display name"

$ kubectl create -f busy-box-obj-user.yaml 
cephobjectstoreuser.ceph.rook.io/busy-box-obj-user created
$ kubectl -n rook-ceph get cephobjectstoreuser
NAME                AGE
busy-box-obj-user   31s
```  
默认 Rook 将生成的包含认证Key的用户信息存在 secret 中，那么，查看下该 secret 生成的 Key 是什么。  
```
$ kubectl -n rook-ceph get secret | grep rook-ceph-object-user
rook-ceph-object-user-busy-box-obj-busy-box-obj-user   kubernetes.io/rook         2      2m34s
$ kubectl -n rook-ceph describe secret/rook-ceph-object-user-busy-box-obj-busy-box-obj-user
Name:         rook-ceph-object-user-busy-box-obj-busy-box-obj-user
Namespace:    rook-ceph
Labels:       app=rook-ceph-rgw
              rook_cluster=rook-ceph
              rook_object_store=busy-box-obj
              user=busy-box-obj-user
Annotations:  <none>

Type:  kubernetes.io/rook

Data
====
AccessKey:  20 bytes
SecretKey:  40 bytes
```  
貌似 AccessKey 和 SecretKey 都隐藏了  
```
$ kubectl -n rook-ceph get secret/rook-ceph-object-user-busy-box-obj-busy-box-obj-user -o jsonpath='{.data.AccessKey}' | base64 --decode
RZTGKUPYG4OJ2EASVPCY
$ kubectl -n rook-ceph get secret/rook-ceph-object-user-busy-box-obj-busy-box-obj-user -o jsonpath='{.data.SecretKey}' | base64 --decode
EWRo6aKjO7OcBWaT974UASUX0SWn4PyzvJGzJSKT
```  
要牢记这两个 Key 值，因为以后访问 S3 的时候必须带上，否则认证不通过。  

3、集群内访问  
可以进入到 rook-toolbox 容器内访问该文件存储。首先需要安装一下 s3cmd 客户端工具，该工具提供 CLI 专门用来操作 s3 存储。  
```
$ kubectl -n rook-ceph exec -it rook-ceph-tools-5bd5cdb949-d22ql bash
[root@node2 /]# yum install s3cmd
```  
然后，我们可以将连接 s3 存储的一些配置信息设置为 ENV 环境变量的形式，会大大方便后边访问。  
```
export AWS_HOST=rook-ceph-rgw-busy-box-obj.rook-ceph
export AWS_ENDPOINT=10.68.117.190:80
export AWS_ACCESS_KEY_ID=RZTGKUPYG4OJ2EASVPCY
export AWS_SECRET_ACCESS_KEY=EWRo6aKjO7OcBWaT974UASUX0SWn4PyzvJGzJSKT
```  
简单说明一下：  
- HOST：rgw 服务在集群内 DNS 地址，按照 K8s DNS 服务命名规则，简单一些可以配置为：rook-ceph-rgw-busy-box-obj.rook-ceph  
- ENDPOINT：rgw 服务监听地址，因为是在进群内部访问，所以可以通过获取 rook-ceph-rgw-busy-box-obj Service 的 Cluster_IP 和 Port，上边 5.1、创建 Object Store 已经获取到了，那么地址为： 10.68.117.190:80  
- ACCESS_KEY：上边获取的  
- SECRET_KEY：上边获取的  
一切准备就绪，就可以通过 s3cmd 工具来操作了。注意：以下操作均在 rook-ceph-tools-5bd5cdb949-d22ql 容器内操作。  

创建 Bucket  
```
# 创建一个 rookbucket 
[root@node2 /]# s3cmd mb --no-ssl --host=${AWS_HOST} --host-bucket=  s3://rookbucket
Bucket 's3://rookbucket/' created
```  
获取 Bucket 列表  
```
# 获取所有 bucket 列表
[root@node2 /]# s3cmd ls --no-ssl --host=${AWS_HOST}
2019-01-08 06:46  s3://rookbucket
```  
Put Object  
```
# 创建一个数据文件，并上传到 s3 rookbucket
[root@node2 /]# echo "This is test data from rook-toolbox." > /rookdata   
[root@node2 /]# s3cmd put /rookdata --no-ssl --host=${AWS_HOST} --host-bucket=  s3://rookbucket
upload: '/rookdata' -> 's3://rookbucket/rookdata'  [1 of 1]
 37 of 37   100% in    0s   582.89 B/s  done
```  
Get Object  
```
# 从 s3 rookbucket 获取 rootdata 数据到本地
[root@node2 /]# s3cmd get s3://rookbucket/rookdata /rookdata-s3 --no-ssl --host=${AWS_HOST} --host-bucket=
download: 's3://rookbucket/rookdata' -> '/rookdata-s3'  [1 of 1]
 37 of 37   100% in    0s   763.14 B/s  done
[root@node2 /]# cat /rookdata-s3 
This is test data from rook-toolbox.
```  
数据读取出来了，跟上传的一模一样。s3cmd 其他命令，例如：Remove (rb)、Delete (del)、Copy (cp)、Modify (modify) 等命令，可以通过 s3cmd -h 命令查看，大家可以尝试一下，这里就不在演示了。  


集群外访问  
以上都是在集群内部容器访问的，如果我们需要外部访问的话，就不行了。这里可以通过部署一个新的 Service 使用 NodePort 暴漏方式。  
```
$ vim busy-box-obj-rgw-external.yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-rgw-busy-box-obj-external
  namespace: rook-ceph
  labels:
    app: rook-ceph-rgw
    rook_cluster: rook-ceph
    rook_object_store: busy-box-obj
spec:
  ports:
  - name: rgw
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: rook-ceph-rgw
    rook_cluster: rook-ceph
    rook_object_store: busy-box-obj
  sessionAffinity: None
  type: NodePort
  
$ kubectl create -f busy-box-obj-rgw-external.yaml 
service/rook-ceph-rgw-busy-box-obj-external created
$ kubectl -n rook-ceph get svc | grep rook-ceph-rgw
rook-ceph-rgw-busy-box-obj               ClusterIP   10.68.117.190   <none>        80/TCP           90m
rook-ceph-rgw-busy-box-obj-external      NodePort    10.68.127.192   <none>        80:24313/TCP     66s
```  
这样，外部就可以通过 http://10.222.78.63:24313 来访问了。我们在集群外部机器，写一个 Python 测试脚本，该脚本将会连接 rgw，然后新建一个新的 bucket，再列出所有的 buckets。脚本变量 access_key 和 secret_key 值就是上边的 access_key 和 secret_key。  
```
# 安装一下 python-boto 工具
$ yum install python-boto

$ vim s3.py
import boto
import boto.s3.connection
access_key = 'RZTGKUPYG4OJ2EASVPCY'
secret_key = 'EWRo6aKjO7OcBWaT974UASUX0SWn4PyzvJGzJSKT'
conn = boto.connect_s3(
    aws_access_key_id = access_key,
    aws_secret_access_key = secret_key,
    host = '10.222.78.63', port=24313,
    is_secure=False,
    calling_format = boto.s3.connection.OrdinaryCallingFormat(),
)
bucket = conn.create_bucket('my-s3-bucket')
for bucket in conn.get_all_buckets():
        print "{name}\t{created}".format(
                name = bucket.name,
                created = bucket.creation_date,
)

$ python s3.py 
Traceback (most recent call last):
  File "s3.py", line 13, in <module>
    for bucket in conn.get_all_buckets():
  File "/usr/lib/python2.7/site-packages/boto/s3/connection.py", line 444, in get_all_buckets
    response.status, response.reason, body)
boto.exception.S3ResponseError: S3ResponseError: 403 Forbidden
<?xml version="1.0" encoding="UTF-8"?><Error><Code>RequestTimeTooSkewed</Code><RequestId>tx00000000000000000003b-005c3465ac-363d-busy-box-obj</RequestId><HostId>363d-busy-box-obj-busy-box-obj</HostId></Error>
```  
配置好 host、port、access_key、secret_key，测试一下，不过很遗憾报错了。。。提示 403 Forbidden，但是我们明明在脚本里面设置了正确的 access_key 和 secret_key 为啥报这个错误? 不过后边 <Code>RequestTimeTooSkewed</Code> 这个错误日志指出，是因为请求时间不合理。  

经过一番排查后，的确是这个问题，因为集群外部机器时间跟集群内 Pod 时间是不一致的，相差了 10 个小时左右，初步判断应该是时区设置不正确。当初启动 Pod 的时候没有将系统当前时间挂载到容器内，导致时间不一致。那么怎么办呢？我们可以去集群内任意一个容器内执行该脚本，因为他们的时间是一致的。虽然还是容器内执行，但是我们连接 s3 服务地址和方式都一样了呦！那么继续去 rook-toolbox 容器内执行吧！  
```
$ kubectl -n rook-ceph exec -it rook-ceph-tools-5bd5cdb949-d22ql bash
[root@node2 /] yum install python-boto
[root@node2 /] vim s3.py
[root@node2 /] python s3.py 
my-s3-bucket	2019-01-08T07:49:31.517Z
rookbucket	2019-01-08T06:46:23.843Z
```  
测试通过，只要时间一致，没问题的  

配置 Ceph Object Gateway Management Frontend  

最后我们访问一下 Rook Dashboard Object Gateway，发现还是什么都没有。但是页面提示我们去参考 Ceph Documents 文档，开启 Object Gateway Management Frontend 功能，这样在 Rook Dashboard 就可以看到了。那么赶紧去实验一下吧！  

进入 rook-toolbox 容器内操作。
```
$ kubectl -n rook-ceph exec -it rook-ceph-tools-5bd5cdb949-d22ql bash

# 获取 user 列表
[root@node2 /]# radosgw-admin user list
[
    "busy-box-obj-user"
]

# 查看 user 信息
[root@node2 /]# radosgw-admin user info --uid=busy-box-obj-user
{
    "user_id": "busy-box-obj-user",
    "display_name": "busy-box display name",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "busy-box-obj-user",
            "access_key": "RZTGKUPYG4OJ2EASVPCY",
            "secret_key": "EWRo6aKjO7OcBWaT974UASUX0SWn4PyzvJGzJSKT"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

# ceph dashboard 设置 host、port、access-key、secret-key 等连接 s3 的配置信息
[root@node2 /]# ceph dashboard set-rgw-api-access-key RZTGKUPYG4OJ2EASVPCY
Option RGW_API_ACCESS_KEY updated
[root@node2 /]# ceph dashboard set-rgw-api-secret-key EWRo6aKjO7OcBWaT974UASUX0SWn4PyzvJGzJSKT
Option RGW_API_SECRET_KEY updated
[root@node2 /]# ceph dashboard set-rgw-api-host 10.222.78.63
Option RGW_API_HOST updated
[root@node2 /]# ceph dashboard set-rgw-api-port 24313
Option RGW_API_PORT updated
[root@node2 /]# ceph dashboard set-rgw-api-user-id busy-box-obj-user
Option RGW_API_USER_ID updated
[root@node2 /]# ceph dashboard set-rgw-api-scheme http
Option RGW_API_SCHEME updated
```  








