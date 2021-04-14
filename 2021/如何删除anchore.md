## 1. 如何删除Anchore

```bash
[root@sec-infoanalysis2-test ops]# helm3 uninstall anchore
release "anchore" uninstalled
[root@sec-infoanalysis2-test ops]# kubectl get pods
NAME                                              READY   STATUS        RESTARTS   AGE
anchore-anchore-engine-catalog-575d4c995f-4m7bh   1/1     Terminating   0          19h
anchore-engine-upgrade-75pvh                      0/1     Completed     0          4d11h
jenkins-0                                         1/1     Running       0          163d
nfs-client-provisioner-687c7db4f7-prdz2           1/1     Running       0          161d
nginx-ingress-controller-7pc4n                    1/1     Running       12         182d
nginx-ingress-controller-9brmf                    1/1     Running       3          182d
nginx-ingress-controller-zbj4z                    0/1     Terminating   173        47d
nginx-ingress-default-backend-77947c8df-5zf7m     1/1     Running       0          85d
[root@sec-infoanalysis2-test ops]# kubectl delete pod anchore-anchore-engine-catalog-575d4c995f-4m7bh --force --grace-period=0
warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "anchore-anchore-engine-catalog-575d4c995f-4m7bh" force deleted
[root@sec-infoanalysis2-test ops]# kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
anchore-postgresql   Bound    anchore-pv-volume                          80Gi       RWO                           183d
jenkins-home         Bound    pvc-605b5123-c723-4dba-92f8-6e5ddacf7fd9   50G        RWX            nfs-storage    215d
[root@sec-infoanalysis2-test ops]# kubectl delete pvc anchore-postgresql
persistentvolumeclaim "anchore-postgresql" deleted
[root@sec-infoanalysis2-test ops]# kubectl delete -f pv.yaml 
persistentvolume "anchore-pv-volume" deleted
[root@sec-infoanalysis2-test ops]# cat pv.yaml 
kind: PersistentVolume
apiVersion: v1
metadata:
  name: anchore-pv-volume
  labels:
    type: postgresql
spec:
  capacity:
    storage: 80Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 192.168.47.146 
    path: "/chj/data/data"

```
