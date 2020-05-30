## 1. 如何在k8s上使用helm部署Anchore

### 1.1. 背景

官方的部署方案不完整，会因为没有为postgres设置pv而导致部署失败，而官方文档也没有提醒需要自己建立pv。  

### 1.2. 安装条件  

- 安装了helm
- 安装了Tiller
- 安装了k8s
- 需要20g空间给数据库使用

### 1.3. 安装  

#### 1.3.1. 更新Helm Charts仓库  

```bash
# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
```  

#### 1.3.2. 使用helm安装  

```bash
# helm install --name anchore stable/anchore-engine --set postgresql.postgresPassword=test,anchoreGlobal.defaultAdminPassword=test,anchoreAnalyzer.layerCacheMaxGigabytes=4,anchoreAnalyzer.replicaCount=5,anchorePolicyEngine.replicaCount=3
NAME:   anchore
LAST DEPLOYED: Wed Oct 23 13:56:29 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                                 DATA  AGE
anchore-anchore-engine           4     0s
anchore-anchore-engine-analyzer  1     0s
anchore-postgresql               0     0s

==> v1/Deployment
NAME                                    READY  UP-TO-DATE  AVAILABLE  AGE
anchore-anchore-engine-analyzer     0/1    1           0          0s
anchore-anchore-engine-api          0/1    1           0          0s
anchore-anchore-engine-catalog      0/1    1           0          0s
anchore-anchore-engine-policy       0/1    1           0          0s
anchore-anchore-engine-simplequeue  0/1    0           0          0s
```  

#### 1.3.3. 检查pod状态

查看pod的状态，发现postgresql的状态pending，原因是没有建立pv，而官方教程并没有提醒。  

```bash
# kubectl get pods
NAME                                                     READY   STATUS    RESTARTS   AGE
anchore-anchore-engine-analyzer-779c48d479-8gtvl     0/1     Running   0          11s
anchore-anchore-engine-api-5c67f58476-lc6rp          0/1     Running   0          11s
anchore-anchore-engine-catalog-99db9fd77-fkfkr       0/1     Running   0          11s
anchore-anchore-engine-policy-bcff8c8fc-lm4k2        0/1     Running   0          11s
anchore-anchore-engine-simplequeue-5cdcb9c74-wq72v   0/1     Running   0          11s
anchore-postgresql-c6657b6c7-4hwh5                   0/1     Pending   0          11s
# kubectl get pvc
NAME                     STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
anchore-postgresql   Pending                                                     4m20s
# kubectl get pv
No resources found.

```  
  
#### 1.3.4. 建立pv  

建立pv，这里使用的是nfs，关于nfs服务的部署这里就不说了  

```bash
# cat pv.yaml 
kind: PersistentVolume
apiVersion: v1
metadata:
  name: anchore-pv-volume
  labels:
    type: postgresql
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.47.146 
    path: "/data"
# showmount -e localhost //查看nfs挂载目录
Export list for localhost:
/data *
# kubectl create -f pv.yaml 
persistentvolume/anchore-pv-volume created
# kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
anchore-pv-volume   20Gi       RWO            Retain           Available                                   4s
# kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
anchore-pv-volume   20Gi       RWO            Retain           Bound    default/anchore-postgresql                           12s
# kubectl get pvc
NAME                 STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
anchore-postgresql   Bound    anchore-pv-volume   20Gi       RWO                           3m24s

```

#### 1.3.5. 查看anchore状态  

```bash
# pip install anchorecli  //安装客户端
# anchore-cli --u admin --p test --url http://10.103.227.236:8228/v1 system status
Service policy_engine (anchore-anchore-engine-policy-bcff8c8fc-lm4k2, http://anchore-anchore-engine-policy:8087): up
Service simplequeue (anchore-anchore-engine-simplequeue-5cdcb9c74-wq72v, http://anchore-anchore-engine-simplequeue:8083): up
Service analyzer (anchore-anchore-engine-analyzer-779c48d479-8gtvl, http://anchore-anchore-engine-analyzer:8084): up
Service apiext (anchore-anchore-engine-api-5c67f58476-lc6rp, http://anchore-anchore-engine-api:8228): up
Service catalog (anchore-anchore-engine-catalog-99db9fd77-fkfkr, http://anchore-anchore-engine-catalog:8082): up

Engine DB Version: 0.0.11
Engine Code Version: 0.5.1

```

#### 1.3.6. 暴露服务  

需要创建一个名为anchore-engine的svc，把pod内部的8228端口暴露出来，用于给Jenkins调用。这里用的是LoadBalancer类型，其中EXTERNAL-IP肯定是pending，因为没有使用Amazon之类的服务，建立后可使用任意节点真实ip+31627端口可以访问暴露出来的服务  

```bash
# kubectl expose deployment anchore-anchore-engine-api --type=LoadBalancer --name=anchore-engine --port=8228 
service/anchore-engine exposed
# kubectl get svc
NAME                                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
anchore-anchore-engine-api           ClusterIP      10.103.227.236   <none>        8228/TCP         131m
anchore-anchore-engine-catalog       ClusterIP      10.104.243.28    <none>        8082/TCP         131m
anchore-anchore-engine-policy        ClusterIP      10.105.2.122     <none>        8087/TCP         131m
anchore-anchore-engine-simplequeue   ClusterIP      10.100.39.207    <none>        8083/TCP         131m
anchore-postgresql                   ClusterIP      10.111.41.100    <none>        5432/TCP         131m
anchore-engine                           LoadBalancer   10.101.52.243    <pending>     8228:31627/TCP   2s

# anchore-cli --u admin --p test --url http://192.168.47.144:31627/v1 system status
Service catalog (anchore-anchore-engine-catalog-99db9fd77-fkfkr, http://anchore-anchore-engine-catalog:8082): up
Service policy_engine (anchore-anchore-engine-policy-bcff8c8fc-lm4k2, http://anchore-anchore-engine-policy:8087): up
Service simplequeue (anchore-anchore-engine-simplequeue-5cdcb9c74-wq72v, http://anchore-anchore-engine-simplequeue:8083): up
Service analyzer (anchore-anchore-engine-analyzer-779c48d479-8gtvl, http://anchore-anchore-engine-analyzer:8084): up
Service apiext (anchore-anchore-engine-api-5c67f58476-lc6rp, http://anchore-anchore-engine-api:8228): up

Engine DB Version: 0.0.11
Engine Code Version: 0.5.1

```

### 1.4. 报错处理：

如果报错MountVolume.SetUp failed for volume "anchore-pv-volume" : mount failed: exit status 32，是因为k8s节点没有安装nfs相关，需执行命令：`yum install nfs-common  nfs-utils -y`  

```bash
# kubectl describe pod anchore-postgresql-c6657b6c7-4hwh5
Name:           anchore-postgresql-c6657b6c7-4hwh5
Namespace:      default
Priority:       0
Node:           sec-k8s-node1/192.168.47.144
Start Time:     Wed, 23 Oct 2019 14:33:47 +0800
Labels:         app=postgresql
                pod-template-hash=c6657b6c7
                release=anchore-chj
Annotations:    <none>
Status:         Pending
IP:             
Controlled By:  ReplicaSet/anchore-postgresql-c6657b6c7
Containers:
  anchore-postgresql:
    Container ID:   
    Image:          postgres:9.6.2
    Image ID:       
    Port:           5432/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Requests:
      cpu:      100m
      memory:   256Mi
    Liveness:   exec [sh -c exec pg_isready --host $POD_IP] delay=60s timeout=5s period=10s #success=1 #failure=6
    Readiness:  exec [sh -c exec pg_isready --host $POD_IP] delay=5s timeout=3s period=5s #success=1 #failure=3
    Environment:
      POSTGRES_USER:         anchoreengine
      PGUSER:                anchoreengine
      POSTGRES_DB:           anchore
      POSTGRES_INITDB_ARGS:  
      PGDATA:                /var/lib/postgresql/data/pgdata
      POSTGRES_PASSWORD:     <set to the key 'postgres-password' in secret 'anchore-postgresql'>  Optional: false
      POD_IP:                 (v1:status.podIP)
    Mounts:
      /var/lib/postgresql/data/pgdata from data (rw,path="postgresql-db")
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-m4s9k (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  anchore-postgresql
    ReadOnly:   false
  default-token-m4s9k:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-m4s9k
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason            Age                   From                    Message
  ----     ------            ----                  ----                    -------
  Warning  FailedScheduling  5m40s (x30 over 42m)  default-scheduler       pod has unbound immediate PersistentVolumeClaims (repeated 2 times)
  Normal   Scheduled         5m36s                 default-scheduler       Successfully assigned default/anchore-postgresql-c6657b6c7-4hwh5 to sec-k8s-node1
  Warning  FailedMount       5m35s                 kubelet, sec-k8s-node1  MountVolume.SetUp failed for volume "anchore-pv-volume" : mount failed: exit status 32

```

### 1.5. 建议使用外部数据库：  

```bash
# helm install --name anchore stable/anchore-engine --set postgresql.postgresPassword=xxx,anchoreGlobal.defaultAdminPassword=xxx,postgresql.postgresUser=postgres,postgresql.postgresDatabase=anchore,postgresql.externalEndpoint=192.168.47.146:5432,anchoreGlobal.logLevel=DEBUG,anchore-feeds-db.postgresPassword=xxx,anchore-feeds-db.externalEndpoint=192.168.47.146:5432,anchoreAnalyzer.layerCacheMaxGigabytes=4
```  

### 1.6. 参考链接：  

https://docs.anchore.com/current/docs/engine/engine_installation/helm/