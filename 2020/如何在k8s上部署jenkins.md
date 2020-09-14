## 1. 如何在k8s上部署jenkins

### 1.1. 安装nfs服务
```
yum install -y nfs-utils
```

### 1.2. 设置nfs目录
```
# cat /etc/exports
/data/     *(rw,sync,no_root_squash,no_all_squash)
```

### 1.3. 新建如下文件  

#### 1.3.1. nfs-client-deployment.yaml
```
#ServiceAccount
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
#rbac
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
#deployment
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
     app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          # image: quay.io/external_storage/nfs-client-provisioner:latest
          image: registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.47.146
            - name: NFS_PATH
              value: /data/data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.47.146
            path: /data/data

```

#### 1.3.2. nfs-client-class.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
```

#### 1.3.3. jenkins-rbac.yaml
```
# In GKE need to get RBAC permissions first with
# kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin [--user=<user-name>|--group=<group-name>]

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
```

#### 1.3.4. jenkins-pvc.yaml
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-home
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 50G
```

#### 1.3.5. jenkins-statefulset.yaml
```
# StatefulSet
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: jenkins
  labels:
    name: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  selector:
   matchLabels:
    app: jenkins
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      name: jenkins
      labels:
       app: jenkins
       name: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: web
              protocol: TCP
            - containerPort: 50000
              name: agent
              protocol: TCP
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 0.5
              memory: 500Mi
          env:
            - name: LIMITS_MEMORY
              valueFrom:
                resourceFieldRef:
                  resource: limits.memory
                  divisor: 1Mi
            - name: JAVA_OPTS
              # value: -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=1 -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
              value: -Duser.timezone=Asia/Shanghai -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12 # ~2 minutes
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12 # ~2 minutes
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-home
      securityContext:
        fsGroup: 1000
```

#### 1.3.6. jenkins-svc.yaml
```
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  selector:
    name: jenkins
  ports:
  - name: web
    port: 8080
    targetPort: web
  - name: agent
    port: 50000
    targetPort: agent
```

### 1.4. 运行所有yaml  

```
kubectl create -f nfs-client-deployment.yaml
kubectl create -f nfs-client-class.yaml
kubectl create -f jenkins-rbac.yaml
kubectl create -f jenkins-pvc.yaml
kubectl create -f jenkins-statefulset.yaml
kubectl create -f jenkins-svc.yaml
```  

### 1.5. 访问jenkins  

使用浏览器访问jenkinshttp://192.168.1.68:31669/，通过查看日志获取密码登录  
```
# kubectl get svc|grep jenkins
jenkins      NodePort    10.100.221.153   <none>        8080:31669/TCP,50000:31011/TCP  
```

### 1.6. 在jenkins上运行docker

使用了Docker-in-Docker，原理就是把宿主机的docker.sock映射到pod里，需要注意的是镜像里要添加docker客户端。
  
#### 1.6.1. 把docker加到jenkins镜像里

这里用的是`zj1244/jenkins_with_docker:v1`，该镜像的Dockerfile文件内容如下：  

```
FROM jenkins/jenkins:latest
MAINTAINER yanmingfan1006@gmail.com
USER root

# Install the latest Docker CE binaries
RUN apt-get update && \
    apt-get -y install apt-transport-https \
      ca-certificates \
      curl \
      gnupg2 \
      software-properties-common && \
    curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
    add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
      $(lsb_release -cs) \
      stable" && \
   apt-get update && \
   apt-get -y install docker-ce

```
  
#### 1.6.2. 把docker.sock映射到pod  

我们在jenkins-statefulset.yaml里添加文件映射：

```
# StatefulSet
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  labels:
    name: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  selector:
   matchLabels:
    app: jenkins
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      name: jenkins
      labels:
       app: jenkins
       name: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: zj1244/jenkins_with_docker:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: web
              protocol: TCP
            - containerPort: 50000
              name: agent
              protocol: TCP
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 0.5
              memory: 500Mi
          env:
            - name: LIMITS_MEMORY
              valueFrom:
                resourceFieldRef:
                  resource: limits.memory
                  divisor: 1Mi
            - name: JAVA_OPTS
              # value: -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=1 -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
              value: -Duser.timezone=Asia/Shanghai -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
            - name: dockersock
              mountPath: /var/run/docker.sock
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12 # ~2 minutes
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12 # ~2 minutes
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-home
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      securityContext:
        fsGroup: 1000

```

#### 1.6.3. 更新pod
```
# kubectl apply -f jenkins-statefulset.yaml
```

### 1.7. 参考：  
https://hyrepo.com/tech/kubernetes-jenkins/#Docker-in-Docker