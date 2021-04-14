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
# helm install anchore anchore/anchore-engine --set postgresql.postgresPassword=test,anchoreGlobal.defaultAdminPassword=test,anchoreAnalyzer.layerCacheMaxGigabytes=4,anchoreAnalyzer.replicaCount=5,anchorePolicyEngine.replicaCount=3
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

##### 1.3.6.1. LoadBalancer  

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

##### 1.3.6.2. ingress  
  
使用lb暴露服务偶尔会出现连接不上接口的问题，使用后来换成了ingress暴露服务。  

1.新建个yml文件，内容如下：
```bash
# cat nginx-ingress-values.yml 
## nginx configuration
## Ref: https://github.com/kubernetes/ingress/blob/master/controllers/nginx/configuration.md
##
controller:
  name: controller
  image:
    repository: quay.io/kubernetes-ingress-controller/nginx-ingress-controller
    tag: "0.25.0"
    pullPolicy: IfNotPresent
    # www-data -> uid 33
    runAsUser: 33
    allowPrivilegeEscalation: true

  # Configures the ports the nginx-controller listens on
  containerPort:
    http: 80
    https: 443
  # Will add custom configuration options to Nginx https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
  config: {}
  # Will add custom header to Nginx https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/customization/custom-headers
  headers: {}

  # Required for use with CNI based kubernetes installations (such as ones set up by kubeadm),
  # since CNI and hostport don't mix yet. Can be deprecated once https://github.com/kubernetes/kubernetes/issues/23920
  # is merged
  hostNetwork: true

  # Optionally change this to ClusterFirstWithHostNet in case you have 'hostNetwork: true'.
  # By default, while using host network, name resolution uses the host's DNS. If you wish nginx-controller
  # to keep resolving names inside the k8s network, use ClusterFirstWithHostNet.
  dnsPolicy: ClusterFirstWithHostNet

  # Bare-metal considerations via the host network https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#via-the-host-network
  # Ingress status was blank because there is no Service exposing the NGINX Ingress controller in a configuration using the host network, the default --publish-service flag used in standard cloud setups does not apply
  reportNodeInternalIp: false

  ## Use host ports 80 and 443
  daemonset:
    useHostPort: false

    hostPorts:
      http: 80
      https: 443

  ## Required only if defaultBackend.enabled = false
  ## Must be <namespace>/<service_name>
  ##
  defaultBackendService: ""

  ## Election ID to use for status update
  ##
  electionID: ingress-controller-leader

  ## Name of the ingress class to route through this controller
  ##
  ingressClass: nginx

  # labels to add to the pod container metadata
  podLabels: {}
  #  key: value

  ## Security Context policies for controller pods
  ## See https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/ for
  ## notes on enabling and using sysctls
  ##
  podSecurityContext: {}

  ## Allows customization of the external service
  ## the ingress will be bound to via DNS
  publishService:
    enabled: false
    ## Allows overriding of the publish service to bind to
    ## Must be <namespace>/<service_name>
    ##
    pathOverride: ""

  ## Limit the scope of the controller
  ##
  scope:
    enabled: false
    namespace: ""   # defaults to .Release.Namespace

  ## Allows customization of the configmap / nginx-configmap namespace
  ##
  configMapNamespace: ""   # defaults to .Release.Namespace

  ## Allows customization of the tcp-services-configmap namespace
  ##
  tcp:
    configMapNamespace: ""   # defaults to .Release.Namespace

  ## Allows customization of the udp-services-configmap namespace
  ##
  udp:
    configMapNamespace: ""   # defaults to .Release.Namespace

  ## Additional command line arguments to pass to nginx-ingress-controller
  ## E.g. to specify the default SSL certificate you can use
  ## extraArgs:
  ##   default-ssl-certificate: "<namespace>/<secret_name>"
  extraArgs: {}

  ## Additional environment variables to set
  extraEnvs: []
  # extraEnvs:
  #   - name: FOO
  #     valueFrom:
  #       secretKeyRef:
  #         key: FOO
  #         name: secret-resource

  ## DaemonSet or Deployment
  ##
  kind: DaemonSet

  # The update strategy to apply to the Deployment or DaemonSet
  ##
  updateStrategy: {}
  #  rollingUpdate:
  #    maxUnavailable: 1
  #  type: RollingUpdate

  # minReadySeconds to avoid killing pods before we are ready
  ##
  minReadySeconds: 0


  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
  #  - key: "key"
  #    operator: "Equal|Exists"
  #    value: "value"
  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

  ## Affinity and anti-affinity
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
  ##
  affinity: {}
    # # An example of preferred pod anti-affinity, weight is in the range 1-100
    # podAntiAffinity:
    #   preferredDuringSchedulingIgnoredDuringExecution:
    #   - weight: 100
    #     podAffinityTerm:
    #       labelSelector:
    #         matchExpressions:
    #         - key: app
    #           operator: In
    #           values:
    #           - nginx-ingress
    #       topologyKey: kubernetes.io/hostname

    # # An example of required pod anti-affinity
    # podAntiAffinity:
    #   requiredDuringSchedulingIgnoredDuringExecution:
    #   - labelSelector:
    #       matchExpressions:
    #       - key: app
    #         operator: In
    #         values:
    #         - nginx-ingress
    #     topologyKey: "kubernetes.io/hostname"

  ## terminationGracePeriodSeconds
  ##
  terminationGracePeriodSeconds: 60

  ## Node labels for controller pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector: {}

  ## Liveness and readiness probe values
  ## Ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  ##
  livenessProbe:
    failureThreshold: 3
    initialDelaySeconds: 10
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 1
    port: 10254
  readinessProbe:
    failureThreshold: 3
    initialDelaySeconds: 10
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 1
    port: 10254

  ## Annotations to be added to controller pods
  ##
  podAnnotations: {}

  replicaCount: 1

  minAvailable: 1

  resources: {}
  #  limits:
  #    cpu: 100m
  #    memory: 64Mi
  #  requests:
  #    cpu: 100m
  #    memory: 64Mi

  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 11
    targetCPUUtilizationPercentage: 50
    targetMemoryUtilizationPercentage: 50

  ## Override NGINX template
  customTemplate:
    configMapName: ""
    configMapKey: ""

  service:
    annotations: {}
    labels: {}
    omitClusterIP: false
    clusterIP: ""

    ## List of IP addresses at which the controller services are available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []

    loadBalancerIP: ""
    loadBalancerSourceRanges: []

    enableHttp: true
    enableHttps: true

    ## Set external traffic policy to: "Local" to preserve source IP on
    ## providers supporting it
    ## Ref: https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer
    externalTrafficPolicy: ""

    healthCheckNodePort: 0

    ports:
      http: 80
      https: 443

    targetPorts:
      http: http
      https: https

    type: ClusterIP

    # type: NodePort
    # nodePorts:
    #   http: 32080
    #   https: 32443
    #   tcp:
    #     8080: 32808
    nodePorts:
      http: ""
      https: ""
      tcp: {}
      udp: {}

  extraContainers: []
  ## Additional containers to be added to the controller pod.
  ## See https://github.com/lemonldap-ng-controller/lemonldap-ng-controller as example.
  #  - name: my-sidecar
  #    image: nginx:latest
  #  - name: lemonldap-ng-controller
  #    image: lemonldapng/lemonldap-ng-controller:0.2.0
  args:
    - --http-port=31050
    - --https-port=31555
  #      - /lemonldap-ng-controller
  #      - --alsologtostderr
  #      - --configmap=$(POD_NAMESPACE)/lemonldap-ng-configuration
  #    env:
  #      - name: POD_NAME
  #        valueFrom:
  #          fieldRef:
  #            fieldPath: metadata.name
  #      - name: POD_NAMESPACE
  #        valueFrom:
  #          fieldRef:
  #            fieldPath: metadata.namespace
  #    volumeMounts:
  #    - name: copy-portal-skins
  #      mountPath: /srv/var/lib/lemonldap-ng/portal/skins

  extraVolumeMounts: []
  ## Additional volumeMounts to the controller main container.
  #  - name: copy-portal-skins
  #   mountPath: /var/lib/lemonldap-ng/portal/skins

  extraVolumes: []
  ## Additional volumes to the controller pod.
  #  - name: copy-portal-skins
  #    emptyDir: {}

  extraInitContainers: []
  ## Containers, which are run before the app containers are started.
  # - name: init-myservice
  #   image: busybox
  #   command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']

  metrics:
    enabled: false

    service:
      annotations: {}
      # prometheus.io/scrape: "true"
      # prometheus.io/port: "10254"

      omitClusterIP: false
      clusterIP: ""

      ## List of IP addresses at which the stats-exporter service is available
      ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
      ##
      externalIPs: []

      loadBalancerIP: ""
      loadBalancerSourceRanges: []
      servicePort: 9913
      type: ClusterIP

    serviceMonitor:
      enabled: false
      additionalLabels: {}
      namespace: ""
      # honorLabels: true

  lifecycle: {}

  priorityClassName: ""

## Rollback limit
##
revisionHistoryLimit: 10

## Default 404 backend
##
defaultBackend:

  ## If false, controller.defaultBackendService must be provided
  ##
  enabled: true

  name: default-backend
  image:
    repository: runcc/defaultbackend-amd64
    tag: "1.5"
    pullPolicy: IfNotPresent
    # nobody user -> uid 65534
    runAsUser: 65534

  extraArgs: {}

  port: 8080

  ## Readiness and liveness probes for default backend
  ## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
  ##
  livenessProbe:
    failureThreshold: 3
    initialDelaySeconds: 30
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 5
  readinessProbe:
    failureThreshold: 6
    initialDelaySeconds: 0
    periodSeconds: 5
    successThreshold: 1
    timeoutSeconds: 5

  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
  #  - key: "key"
  #    operator: "Equal|Exists"
  #    value: "value"
  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

  affinity: {}

  ## Security Context policies for controller pods
  ## See https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/ for
  ## notes on enabling and using sysctls
  ##
  podSecurityContext: {}

  # labels to add to the pod container metadata
  podLabels: {}
  #  key: value

  ## Node labels for default backend pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector: {}

  ## Annotations to be added to default backend pods
  ##
  podAnnotations: {}

  replicaCount: 1

  minAvailable: 1

  resources: {}
  # limits:
  #   cpu: 10m
  #   memory: 20Mi
  # requests:
  #   cpu: 10m
  #   memory: 20Mi

  service:
    annotations: {}
    omitClusterIP: false
    clusterIP: ""

    ## List of IP addresses at which the default backend service is available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []

    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    type: ClusterIP

  priorityClassName: ""

## Enable RBAC as per https://github.com/kubernetes/ingress/tree/master/examples/rbac/nginx and https://github.com/kubernetes/ingress/issues/266
rbac:
  create: true

# If true, create & use Pod Security Policy resources
# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
podSecurityPolicy:
  enabled: false

serviceAccount:
  create: true
  name:

## Optional array of imagePullSecrets containing private registry credentials
## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
imagePullSecrets: []
# - name: secretName

# TCP service key:value pairs
# Ref: https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/examples/tcp
##
tcp: {}
#  8080: "default/example-tcp-svc:9000"

# UDP service key:value pairs
# Ref: https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/examples/udp
##
udp: {}
#  53: "kube-system/kube-dns:53"

```
2. 安装
```bash
# helm install stable/nginx-ingress --name nginx-ingress -f nginx-ingress-values.yml
```
3. 安装完毕，如果需要制定暴露的端口，则需要做如下操作：  

```bash
# kubectl edit daemonset.apps/nginx-ingress-controller
      - args:
        - /nginx-ingress-controller
        - --default-backend-service=default/nginx-ingress-default-backend
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=default/nginx-ingress-controller
        - --http-port=31050 #添加这两行
        - --https-port=31055 #添加这两行

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
# helm install anchore anchore/anchore-engine --set postgresql.postgresPassword=xxx,anchoreGlobal.defaultAdminPassword=xxx,postgresql.postgresUser=postgres,postgresql.postgresDatabase=anchore,postgresql.externalEndpoint=192.168.47.146:5432,anchoreGlobal.logLevel=DEBUG,anchore-feeds-db.postgresPassword=xxx,anchore-feeds-db.externalEndpoint=192.168.47.146:5432
```  
使用外部数据库一定要新建个名为anchore的数据库，不然会一直报错
```bash
# kubectl logs anchore-anchore-engine-api-5479b98897-njcc8
[MainThread] [anchore_engine.configuration.localconfig/validate_config()] [WARN] no webhooks defined in configuration file - notifications will be disabled
[MainThread] [anchore_manager.cli.service/start()] [INFO] Loading DB routines from module (anchore_engine)
[MainThread] [anchore_manager.util.db/connect_database()] [INFO] DB params: {"db_connect_args": {"connect_timeout": 86400}, "db_pool_size": 30, "db_pool_max_overflow": 100, "db_echo": false, "db_engine_args": null}
[MainThread] [anchore_manager.util.db/connect_database()] [INFO] DB connection configured: True
[MainThread] [anchore_manager.util.db/connect_database()] [INFO] DB attempting to connect...
[MainThread] [anchore_manager.util.db/connect_database()] [WARN] DB connection failed, retrying - exception: test connection failed - exception: (psycopg2.OperationalError) FATAL:  database "anchore" does not exist

(Background on this error at: http://sqlalche.me/e/e3q8)
[MainThread] [anchore_manager.util.db/connect_database()] [INFO] DB attempting to connect...
[MainThread] [anchore_manager.util.db/connect_database()] [WARN] DB connection failed, retrying - exception: test connection failed - exception: (psycopg2.OperationalError) FATAL:  database "anchore" does not exist


```  

### 1.6. 参考链接：  

https://docs.anchore.com/current/docs/engine/engine_installation/helm/