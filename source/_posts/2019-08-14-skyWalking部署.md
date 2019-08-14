---
title: SkyWalking部署
tags: [devops,docker]
---
# SkyWalking部署
## 部署整体方案  
1. docker安装ElasticSearch6.8.2，由于ES部署的阿里云主机无法直接连接外网，所以需要设置docker的网络代理配置，以及添加dockerhub的国内源。  

2. skyWalking 的backend及ui则直接安装在阿里云虚机中，docker-compose不熟悉，后续学习了再通过docker安装。  


## docker网络代理设置
1. 创建目录 `mkdir -p /etc/systemd/system/docker.service.d` 
2. 创建文件`/etc/systemd/system/docker.service.d/http-proxy.conf`
3. 文件新增内容如下(HTTP_PROXY需要改成自己的代理)：
```
[Service]
Environment="HTTP_PROXY=http://pill:pill@node2:3128/"
```
4. 重启docker
`systemctl daemon-reload && systemctl restart docker`
5. 执行指令`docker info | grep Proxy`验证proxy是否成功，
显示`HTTP Proxy: 172.17.xxx.xx:3128`表示成功
## dockerhub源更新
```
vim /etc/docker/daemon.json
添加以下选项：
{
"registry-mirrors":["http://18817714.m.daocloud.io"]
}
保存文件之后，重启docker
systemctl  restart docker
```

## 使用docker下载ES
1. 执行命令`docker pull elasticsearch:6.8.2 `
2. 运行es `docker run -it -p 9200:9200 -p 9300:9300 --name myes 对应的imageid`

## skywalking后台配置
1. 修改`apache-skywalking-apm-bin/config/application.yml`
2. 将storage改为elasticsearch（默认为h2），注意下面的配置中es的ip地址要修改，其余可以不改。
```
storage:
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:172.173.xx.xx:9200}
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:2}
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    # Those data TTL settings will override the same settings in core module.
    recordDataTTL: ${SW_STORAGE_ES_RECORD_DATA_TTL:7} # Unit is day
    otherMetricsDataTTL: ${SW_STORAGE_ES_OTHER_METRIC_DATA_TTL:45} # Unit is day
    monthMetricsDataTTL: ${SW_STORAGE_ES_MONTH_METRIC_DATA_TTL:18} # Unit is month
    # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:1000} # Execute the bulk every 1000 requests
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}

```

## 修改skywalking UI配置
1. 修改文件`apache-skywalking-apm-bin/webapp.webapp.yml`
2. 下面文件中8082是ui 应用app运行的端口，使用nginx的话，需要重定向到这个端口。
3. `127.0.0.1:12800`是运行backend的ip和端口，视情况自己修改。
```
server:
  port: 8082

collector:
  path: /graphql
  ribbon:
    ReadTimeout: 10000
    # Point to all backend's restHost:restPort, split by ,
    listOfServers: 127.0.0.1:12800

```

## 运行skywalking
1. 运行`apache-skywalking-apm-bin/bin/startup.sh`统一启动ui和backend。
2. 配置nginx 重定向到 skywalking ui服务，此处详细配置省略。
3. 通过浏览器访问nginx反向代理的地址，查看效果。





## 参考
[Docker代理配置](https://www.cnblogs.com/lixiaolun/p/7449017.html)   
[阿里云squid网络代理设置](https://blog.gunxueqiu.site/2019/05/16/2019-05-16-%E9%98%BF%E9%87%8C%E4%BA%91%E5%86%85%E7%BD%91%E6%9C%BA%E9%80%9A%E8%BF%87%E4%BB%A3%E7%90%86%E8%AE%BF%E9%97%AE%E5%A4%96%E7%BD%91/)   
[skywalking 官方文档](https://github.com/apache/skywalking/tree/v6.3.0/docs)