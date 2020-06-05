## 1. 如何使用docker-compose部署Anchore

1. 命令如下：  

```
# curl https://docs.anchore.com/current/docs/engine/quickstart/docker-compose.yaml > docker-compose.yaml
# docker-compose up -d
```  

2. 查看部署是否成功：  

```bash
# docker-compose ps
        Name                      Command               State           Ports         
--------------------------------------------------------------------------------------
root_analyzer_1        /docker-entrypoint.sh anch ...   Up      8228/tcp              
root_api_1             /docker-entrypoint.sh anch ...   Up      0.0.0.0:8228->8228/tcp
root_catalog_1         /docker-entrypoint.sh anch ...   Up      8228/tcp              
root_db_1              docker-entrypoint.sh postgres    Up      5432/tcp              
root_policy-engine_1   /docker-entrypoint.sh anch ...   Up      8228/tcp              
root_queue_1           /docker-entrypoint.sh anch ...   Up      8228/tcp     
# docker-compose exec api anchore-cli system status
Service analyzer (anchore-quickstart, http://analyzer:8228): up
Service simplequeue (anchore-quickstart, http://queue:8228): up
Service apiext (anchore-quickstart, http://api:8228): up
Service policy_engine (anchore-quickstart, http://policy-engine:8228): up
Service catalog (anchore-quickstart, http://catalog:8228): up

Engine DB Version: 0.0.13
Engine Code Version: 0.7.1
```  

3. 等待漏洞库更新完毕  

```bash
# docker-compose exec api anchore-cli system wait
Starting checks to wait for anchore-engine to be available timeout=-1.0 interval=5.0
API availability: Checking anchore-engine URL (http://localhost:8228)...
API availability: Success.
Service availability: Checking for service set (catalog,apiext,policy_engine,simplequeue,analyzer)...
Service availability: Success.
Feed sync: Checking sync completion for feed set (vulnerabilities)...
Feed sync: Checking sync completion for feed set (vulnerabilities)...
...
...
Feed sync: Success.
```  

参考：  

https://docs.anchore.com/current/docs/engine/quickstart/