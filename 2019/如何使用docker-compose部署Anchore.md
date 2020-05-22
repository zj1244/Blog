## 1. 如何使用docker-compose部署Anchore

### 1.1. 命令如下：
```
# mkdir ~/aevolume
# mkdir ~/aevolume/config
# mkdir ~/aevolume/db
# cd ~/aevolume
# curl https://raw.githubusercontent.com/anchore/anchore-engine/master/scripts/docker-compose/docker-compose.yaml -o ~/aevolume/docker-compose.yaml
# curl https://raw.githubusercontent.com/anchore/anchore-engine/master/scripts/docker-compose/config.yaml -o ~/aevolume/config/config.yaml
# docker-compose pull
# docker-compose up -d
```

### 1.2. 查看部署是否成功：

```
[root@sec-infoanalysis1-test config]# docker-compose ps
          Name                         Command                  State                           Ports                     
--------------------------------------------------------------------------------------------------------------------------
aevolume_anchore-db_1       docker-entrypoint.sh postgres    Up             5432/tcp                                      
aevolume_anchore-engine_1   /docker-entrypoint.sh anch ...   Up (healthy)   0.0.0.0:8228->8228/tcp, 0.0.0.0:8338->8338/tcp
[root@sec-infoanalysis1-test config]# 
[root@sec-infoanalysis1-test config]# 
[root@sec-infoanalysis1-test config]# 
[root@sec-infoanalysis1-test config]# docker-compose exec anchore-engine anchore-cli --u admin --p foobar system status
Service catalog (dockerhostid-anchore-engine, http://anchore-engine:8082): up
Service analyzer (dockerhostid-anchore-engine, http://anchore-engine:8084): up
Service apiext (dockerhostid-anchore-engine, http://anchore-engine:8228): up
Service kubernetes_webhook (dockerhostid-anchore-engine, http://anchore-engine:8338): up
Service simplequeue (dockerhostid-anchore-engine, http://anchore-engine:8083): up
Service policy_engine (dockerhostid-anchore-engine, http://anchore-engine:8087): up

Engine DB Version: 0.0.9
Engine Code Version: 0.3.4
```

PS：
默认的配置会导致出现如下警告：
![](_v_images/20200522164032162_18389.png)

可以通过修改config.yaml文件内容解决：
把`db_connect: 'postgresql+pg8000://postgres:mysecretpassword@anchore-db:5432/postgres'`改成`db_connect: 'postgresql+psycopg2://postgres:mysecretpassword@anchore-db:5432/postgres'`

![](_v_images/20200522164101762_1383.png)

### 1.3. 参考：
https://anchore.freshdesk.com/support/solutions/articles/36000020729-install-with-docker-compose
https://github.com/anchore/anchore-engine/issues/83