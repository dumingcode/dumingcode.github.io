---
title: 微服务实践04-熔断监控Hystrix Dashboard和Turbine
tags: [微服务]
---
# 微服务实践04-熔断监控Hystrix Dashboard和Turbine
Hystrix-dashboard是一款针对Hystrix进行实时监控的工具，通过Hystrix Dashboard我们可以在直观地看到各Hystrix Command的请求响应时间, 请求成功率等数据。但是只使用Hystrix Dashboard的话, 你只能看到单个应用内的服务信息, 这明显不够. 我们需要一个工具能让我们汇总系统内多个服务的数据并显示到Hystrix Dashboard上, 这个工具就是Turbine。  
## Hystrix Dashboard
在熔断示例项目eureka-consumer的基础上更改，重新命名为：eureka-consumer-hystrix-dashboard。
### 添加依赖
```
        <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
```

### 启动类
```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@EnableHystrixDashboard
@EnableCircuitBreaker
public class EurekaConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumerApplication.class, args);
	}
}

```
### 解决404异常
`actuator/hystrix.stream` 打开后404错误，需要新建bootstrap.yml内容如下，具体解决方法见参考文章。
```
management:
    endpoints:
        web:
          exposure:
            include: '*'
```

### 测试
访问`http://localhost:9101/actuator/hystrix.stream`,页面会出现如下所示ping：
```
ping: 

data: {"type":"HystrixCommand","name":"HelloRemote#hello(String)","group":"spring-cl

```

然后进入页面`http://localhost:9101/hystrix/`，在页面中的input输入框中，输入`http://localhost:9101/actuator/hystrix.stream`，相当于将此链接的返回json数据图形化，见下图：
![微服务实践05-hystrix-turbine](/images/wfwsj04_turbine.png)<br/>  
Hystrix Dashboard Wiki上详细说明了图上每个指标的含义，如下图：
![微服务实践05-wiki](/images/wfwsj04_wiki.png)<br/>  
到此单个应用的熔断监控已经完成。

## Turbine
复杂的分布式系统中，相同服务的节点经常需要部署上百甚至上千个，很多时候，运维人员希望能够把相同服务的节点状态以一个整体集群的形式展现出来，这样可以更好的把握整个系统的状态。 为此，Netflix提供了一个开源项目（Turbine）来提供把多个hystrix.stream的内容聚合为一个数据源供Dashboard展示。  

### pom文件
新增依赖如下：
```
	<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-netflix-turbine</artifactId>
		</dependency>
```
### 配置文件
```
spring.application.name=hystrix-dashboard-turbine
server.port=9101
eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
feign.hystrix.enabled=true
turbine.appConfig=node01,node02
turbine.aggregator.clusterConfig= default
turbine.clusterNameExpression= new String("default")
```
### 启动类
添加`EnableTurbine`
```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@EnableHystrixDashboard
@EnableCircuitBreaker
@EnableTurbine
public class EurekaConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumerApplication.class, args);
	}
}

```
### 新建两个node消费者
node1配置文件
```
spring.application.name=node01
server.port=9001
feign.hystrix.enabled=true
eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
```
node2配置文件
```
spring.application.name=node02
server.port=9002
feign.hystrix.enabled=true
eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
```
### 测试
依次启动eureka-server ,turbine,node1-consumer,node2-consumer。访问页面`http://localhost:9100/turbine.stream`，可以看到不停的ping
```
: ping
: ping
data: {"reportingHostsLast10Seconds":0,"name":"meta","type":"meta","timestamp":1534662688727}

```
且会不断刷新以获取实时的监控数据，说明和单个的监控类似，返回监控项目的信息。进行图形化监控查看，输入：`http://localhost:9100/hystrix`，输入： `http://localhost:9100/turbine.stream`，然后点击 Monitor Stream ,可以看到出现了俩个监控列表。



## 参考
[springcloud(五)：熔断监控Hystrix Dashboard和Turbine](http://www.ityouknow.com/springcloud/2017/05/18/hystrix-dashboard-turbine.html)   
[spring-cloud-netflix](http://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html)    
[hystrix-dashboard 报错 /actuator/hystrix.stream 404 Not Found](https://blog.csdn.net/yp090416/article/details/81437390)   
[本文github地址](https://github.com/dumingcode/mySpringCloud) 