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

7、配置ceph dashboard  
在cluster.yaml文件中默认已经启用了ceph dashboard，查看dashboard的service：  
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
rook-ceph-mgr-dashboard监听的端口是8443，创建nodeport类型的service以便集群外部访问。  
```
# kubectl apply -f dashboard-external-https.yaml
```  
查看一下nodeport暴露的端口，这里是30511端口：  
```
# kubectl get service -n rook-ceph | grep dashboard
rook-ceph-mgr-dashboard                  ClusterIP   10.105.88.141    <none>        8443/TCP         29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.104.106.111   <none>        8443:30511/TCP   27m
```  
获取Dashboard的登陆账号和密码  
```
# MGR_POD=`kubectl get pod -n rook-ceph | grep mgr | awk '{print $1}'`
# kubectl -n rook-ceph logs $MGR_POD | grep password
debug 2019-03-21 19:53:32.859 7fd5f7a9b700  0 log_channel(audit) log [DBG] : from='client.14171 10.244.2.3:0/3778027086' entity='client.admin' cmd=[{"username": "admin", "prefix": "dashboard set-login-credentials", "password": "XtfoBbh8T5", "target": ["mgr", ""], "format": "json"}]: dispatch
```  
找到username和password字段，我这里是admin，XtfoBbh8T5  

打开浏览器输入任意一个Node的IP+nodeport端口，这里使用master节点 ip访问：  
https://192.168.101.68:30511  


