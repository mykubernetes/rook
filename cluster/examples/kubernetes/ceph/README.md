部署Rook Operator
=================
1、克隆rook github仓库到本地    
```
git clone https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph/
```  

2、执行yaml文件部署rook系统组件：  
```
# kubectl apply -f operator.yaml
namespace/rook-ceph-system created
customresourcedefinition.apiextensions.k8s.io/cephclusters.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephfilesystems.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephobjectstores.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephobjectstoreusers.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephblockpools.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/volumes.rook.io created
clusterrole.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
role.rbac.authorization.k8s.io/rook-ceph-system created
clusterrole.rbac.authorization.k8s.io/rook-ceph-global created
clusterrole.rbac.authorization.k8s.io/rook-ceph-mgr-cluster created
serviceaccount/rook-ceph-system created
rolebinding.rbac.authorization.k8s.io/rook-ceph-system created
clusterrolebinding.rbac.authorization.k8s.io/rook-ceph-global created
deployment.apps/rook-ceph-operator created
```  
- namespace：rook-ceph-system，之后的所有rook相关的pod都会创建在该namespace下面  
- CRD：创建五个CRDs，.ceph.rook.io  
- role & clusterrole：用户资源控制  
- serviceaccount：ServiceAccount资源，给Rook创建的Pod使用  
- deployment：rook-ceph-operator，部署rook ceph相关的组件  

3、部署rook-ceph-operator过程中，会触发以DaemonSet的方式在集群部署Agent和Discoverpods。  
   operator会在集群内的每个主机创建两个pod:rook-discover,rook-ceph-agent：  
```
# kubectl get pod -n rook-ceph-system  -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
rook-ceph-agent-ljgsq                1/1     Running   0          28m   192.168.101.68   node03   <none>           <none>
rook-ceph-agent-wvr46                1/1     Running   0          28m   192.168.101.67   node02   <none>           <none>
rook-ceph-operator-b996864dd-562p7   1/1     Running   0          34m   10.244.2.3       node03   <none>           <none>
rook-discover-ktn5s                  1/1     Running   0          28m   10.244.2.4       node03   <none>           <none>
rook-discover-plqtm                  1/1     Running   0          28m   10.244.1.4       node02   <none>           <none>
```  

4、创建rook Cluster  
当检查到Rook operator, agent, and discover pods已经是running状态后，就可以部署roo cluster了。  
执行yaml文件结果：
```
# kubectl apply -f cluster.yaml
namespace/rook-ceph created
serviceaccount/rook-ceph-osd created
serviceaccount/rook-ceph-mgr created
role.rbac.authorization.k8s.io/rook-ceph-osd created
role.rbac.authorization.k8s.io/rook-ceph-mgr-system created
role.rbac.authorization.k8s.io/rook-ceph-mgr created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
rolebinding.rbac.authorization.k8s.io/rook-ceph-osd created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr-system created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr-cluster created
cephcluster.ceph.rook.io/rook-ceph created
```  
- namespace：rook-ceph，之后的所有Ceph集群相关的pod都会创建在该namespace下  
- serviceaccount：ServiceAccount资源，给Ceph集群的Pod使用  
- role & rolebinding：用户资源控制  
- cluster：rook-ceph，创建的Ceph集群  

5、Ceph集群部署成功后，可以查看到的pods如下，其中osd数量取决于你的节点数量：  
```
# kubectl get pod -n rook-ceph -o wide
NAME                                 READY   STATUS      RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
rook-ceph-mgr-a-5f974b8474-ntrpf     1/1     Running     0          22m   10.244.2.7   node03   <none>           <none>
rook-ceph-mon-a-588d99bfb5-fcplj     1/1     Running     0          24m   10.244.1.5   node02   <none>           <none>
rook-ceph-mon-b-5bc68f4596-jg89t     1/1     Running     0          23m   10.244.2.6   node03   <none>           <none>
rook-ceph-mon-c-66c7456c59-mrrzv     1/1     Running     0          22m   10.244.1.6   node02   <none>           <none>
rook-ceph-osd-0-7ddfb84bc-nljxl      1/1     Running     0          21m   10.244.2.9   node03   <none>           <none>
rook-ceph-osd-1-55755bfb4d-28s46     1/1     Running     0          21m   10.244.1.8   node02   <none>           <none>
rook-ceph-osd-prepare-node02-8tdcr   0/2     Completed   0          22m   10.244.1.7   node02   <none>           <none>
rook-ceph-osd-prepare-node03-rrtnf   0/2     Completed   0          22m   10.244.2.8   node03   <none>           <none>
```  
- Ceph Monitors：默认启动三个ceph-mon，可以在cluster.yaml里配置
- Ceph Mgr：默认启动一个，可以在cluster.yaml里配置
- Ceph OSDs：根据cluster.yaml里的配置启动，默认在所有的可用节点上启动

上述Ceph组件对应kubernetes的kind是deployment：  
```
# kubectl -n rook-ceph get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
rook-ceph-mgr-a   1/1     1            1           24m
rook-ceph-mon-a   1/1     1            1           25m
rook-ceph-mon-b   1/1     1            1           24m
rook-ceph-mon-c   1/1     1            1           24m
rook-ceph-osd-0   1/1     1            1           23m
rook-ceph-osd-1   1/1     1            1           23m
```


6、删除Ceph集群  
如果要删除已创建的Ceph集群，可执行下面命令：  
``` # kubectl delete -f cluster.yaml ```  

删除Ceph集群后，在之前部署Ceph组件节点的/var/lib/rook/目录，会遗留下Ceph集群的配置信息。  
若之后再部署新的Ceph集群，先把之前Ceph集群的这些信息删除，不然启动monitor会失败；  
```
# cat clean-rook-dir.sh
hosts=( k8s-master
        k8s-node1
        k8s-node2 )
for host in ${hosts[@]} ; do
  ssh $host "rm -rf /var/lib/rook/*"
done
```  

配置ceph dashboard  
-------------------
1、在cluster.yaml文件中默认已经启用了ceph dashboard，查看dashboard的service：  
```
# kubectl get service -n rook-ceph
NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
rook-ceph-mgr                            ClusterIP   10.110.159.79    <none>        9283/TCP         27m
rook-ceph-mgr-dashboard                  ClusterIP   10.105.88.141    <none>        8443/TCP         27m
rook-ceph-mgr-dashboard-external-https   NodePort    10.104.106.111   <none>        8443:30511/TCP   24m
rook-ceph-mon-a                          ClusterIP   10.98.204.137    <none>        6789/TCP         29m
rook-ceph-mon-b                          ClusterIP   10.105.215.160   <none>        6789/TCP         28m
rook-ceph-mon-c                          ClusterIP   10.97.14.19      <none>        6789/TCP         28m
```  
2、rook-ceph-mgr-dashboard监听的端口是8443，创建nodeport类型的service以便集群外部访问。  
```
# kubectl apply -f dashboard-external-https.yaml
```  
3、查看一下nodeport暴露的端口，这里是30511端口：  
```
# kubectl get service -n rook-ceph | grep dashboard
rook-ceph-mgr-dashboard                  ClusterIP   10.105.88.141    <none>        8443/TCP         29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.104.106.111   <none>        8443:30511/TCP   27m
```  
4、获取Dashboard的登陆账号和密码  
```
# MGR_POD=`kubectl get pod -n rook-ceph | grep mgr | awk '{print $1}'`
# kubectl -n rook-ceph logs $MGR_POD | grep password
debug 2019-03-21 19:53:32.859 7fd5f7a9b700  0 log_channel(audit) log [DBG] : from='client.14171 10.244.2.3:0/3778027086' entity='client.admin' cmd=[{"username": "admin", "prefix": "dashboard set-login-credentials", "password": "XtfoBbh8T5", "target": ["mgr", ""], "format": "json"}]: dispatch
```  
找到username和password字段，我这里是admin，XtfoBbh8T5  

5、打开浏览器输入任意一个Node的IP+nodeport端口，这里使用master节点 ip访问：  
https://192.168.101.68:30511  


部署Ceph toolbox  
----------------
1、默认启动的Ceph集群，是开启Ceph认证的，这样你登陆Ceph组件所在的Pod里，是没法去获取集群状态，以及执行CLI命令，这时需要部署Ceph toolbox，命令如下  
```
# kubectl apply -f toolbox.yaml
```  

2、部署成功后，pod如下：  
```
# kubectl -n rook-ceph get pods -o wide | grep ceph-tools
rook-ceph-tools-76c7d559b6-2px47     1/1     Running     0          19s   192.168.101.67   node02   <none>           <none>
```  

3、然后可以登陆该pod后，执行Ceph CLI命令  
```
# kubectl -n rook-ceph exec -it rook-ceph-tools-76c7d559b6-2px47 bash
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
```  

4、查看ceph集群状态  
```
[root@node02 /]# ceph status
  cluster:
    id:     e262d949-d0ce-48e9-97ee-6b19c575b695
    health: HEALTH_WARN
            clock skew detected on mon.b
 
  services:
    mon: 3 daemons, quorum c,a,b
    mgr: a(active)
    osd: 2 osds: 2 up, 2 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   8.7 GiB used, 26 GiB / 35 GiB avail
    pgs:     
 
```  

5、查看ceph配置文件  
```
[root@node02 ceph]# cd /etc/ceph/

[root@node02 ceph]# ll
total 12
-rw-r--r--. 1 root root 120 Mar 30 10:00 ceph.conf
-rw-r--r--. 1 root root  62 Mar 30 10:00 keyring
-rw-r--r--. 1 root root  92 Mar 18 11:15 rbdmap

[root@node02 ceph]# cat ceph.conf
[global]
mon_host = 10.98.204.137:6789,10.105.215.160:6789,10.97.14.19:6789

[client.admin]
keyring = /etc/ceph/keyring

[root@node02 ceph]# cat keyring
[client.admin]
key = AQBL65Ncp+t4ChAAV3sD46EZzJGDzjXld9Eifg==

[root@node02 ceph]# cat rbdmap
# RbdDevice		Parameters
#poolname/imagename	id=client,keyring=/etc/ceph/ceph.client.keyring
[root@node02 ceph]# 
```  

rook提供RBD服务
--------------
rook可以提供以下3类型的存储：  
Block: Create block storage to be consumed by a pod  
Object: Create an object store that is accessible inside or outside the Kubernetes cluster  
Shared File System: Create a file system to be shared across multiple pods  

在提供（Provisioning）块存储之前，需要先创建StorageClass和存储池。K8S需要这两类资源，才能和Rook交互，进而分配持久卷（PV）。  

在kubernetes集群里，要提供rbd块设备服务，需要有如下步骤：  
- 创建rbd-provisioner pod  
- 创建rbd对应的storageclass  
- 创建pvc，使用rbd对应的storageclass  
- 创建pod使用rbd pvc  

通过rook创建Ceph Cluster之后，rook自身提供了rbd-provisioner服务，所以我们不需要再部署其provisioner。  
备注：代码位置pkg/operator/ceph/provisioner/provisioner.go  
创建pool和StorageClass  
查看storageclass.yaml的配置（默认）：  
```
# cat storageclass.yaml
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
  # Specify the namespace of the rook cluster from which to create volumes.
  # If not specified, it will use `rook` as the default namespace of the cluster.
  # This is also the namespace where the cluster will be
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
  # (Optional) Specify an existing Ceph user that will be used for mounting storage with this StorageClass.
  #mountUser: user1
  # (Optional) Specify an existing Kubernetes secret name containing just one key holding the Ceph user secret.
  # The secret must exist in each namespace(s) where the storage will be consumed.
  #mountSecret: ceph-user1-secret
```  
配置文件中包含了一个名为replicapool的存储池，和名为rook-ceph-block的storageClass。  
运行yaml文件  
```
# kubectl apply -f storageclass.yaml
```  

查看创建的storageclass:  
```
# kubectl get storageclass
NAME              PROVISIONER          AGE
rook-ceph-block   ceph.rook.io/block   44s
```  

登录ceph dashboard查看创建的存储池：  


使用存储  
---------
以官方wordpress示例为例，创建一个经典的wordpress和mysql应用程序来使用Rook提供的块存储，这两个应用程序都将使用Rook提供的block volumes。  
查看yaml文件配置，主要看定义的pvc和挂载volume部分，以wordpress.yaml为例：  
```
# cat ../wordpress.yaml 
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
  - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.6.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          value: changeme
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
```  
yaml文件里定义了一个名为wp-pv-claim的pvc，指定storageClassName为rook-ceph-block，申请的存储空间大小为20Gi。最后一部分创建了一个名为wordpress-persistent-storage的volume，并且指定 claimName为pvc的名称，最后将volume挂载到pod的/var/lib/mysql目录下。  
启动mysql和wordpress ：  
```
kubectl apply -f mysql.yaml
kubectl apply -f wordpress.yaml
```  

这2个应用都会创建一个块存储卷，并且挂载到各自的pod中，查看声明的pvc和pv：  
```
# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-pv-claim   Bound    pvc-e28dad25-4c1d-11e9-b6a2-000c299d9e73   20Gi       RWO            rook-ceph-block   76s
wp-pv-claim      Bound    pvc-dbc7f03c-4c1d-11e9-b6a2-000c299d9e73   20Gi       RWO            rook-ceph-block   88s
```  

注意：这里的pv会自动创建，当提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV，这是用到的是Dynamic Provisioning机制来动态创建pv，PV 支持 Static 静态请求，和动态创建两种方式。  

在Ceph集群端检查：  
```
# kubectl -n rook-ceph exec -it rook-ceph-tools-76c7d559b6-2px47 bash
.....
[root@node02 /]# rbd info -p replicapool pvc-dbc7f03c-4c1d-11e9-b6a2-000c299d9e73
rbd image 'pvc-dbc7f03c-4c1d-11e9-b6a2-000c299d9e73':
	size 20 GiB in 5120 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 3b1bb601a041
	block_name_prefix: rbd_data.3b1bb601a041
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Thu Mar 21 20:53:47 2019
```  

登陆pod检查rbd设备：  
```
# kubectl get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
wordpress-mysql-6887bf844f-8nqbp   1/1     Running   0          7m34s   10.244.1.9   node02      <none>           <none>
wordpress-mysql-6887bf844f-9pmg8   1/1     Running   0          135m   10.244.2.14   k8s-node2   <none>           <none>


kubectl exec -it wordpress-7b6c4c79bb-t5pst bash
root@wordpress-7b6c4c79bb-t5pst:/var/www/html#
root@wordpress-7b6c4c79bb-t5pst:/var/www/html#  mount | grep rbd
/dev/rbd0 on /var/www/html type xfs (rw,relatime,attr2,inode64,sunit=8192,swidth=8192,noquota)
root@wordpress-7b6c4c79bb-t5pst:/var/www/html# df -h
Filesystem Size Used Avail Use% Mounted on
......
/dev/rbd0 20G 59M 20G 1% /var/www/html
......
```  

