# [云框架]基于kong的api gateway

![](https://img.shields.io/badge/release-v1.0-green.svg)
[![](https://img.shields.io/badge/Producer-elvis2002-orange.svg)](CONTRIBUTORS.md)
![](https://img.shields.io/badge/License-Apache_2.0-blue.svg)
[![](https://img.shields.io/badge/Chat-Gitter-yellow.svg)](https://gitter.im/cloudframeworks-springcloud?utm_source=share-link&utm_medium=link&utm_campaign=share-link)
<a target="_blank" href="//shang.qq.com/wpa/qunwpa?idkey=08656cfa0add38f98157b017d0f66b6b076fc48d354cd43b8cf2321aa7436f9d"><img border="0" src="https://img.shields.io/badge/Group-QQ-brightgreen.svg" alt="Cloudframeworks-spring cloud" title="Cloudframeworks-spring cloud"></a>

[api gateway]
随着微服务的流行，服务数量急剧增加，连接服务正在激增，部署授权、负载均衡、通信管理、分析和改变的难度也变得越来越大。api gateway可以提供访问限制、安全、流量控制、分析监控、日志、请求转发、合成和协议转换功能，api gateway可以让开发者集中精力去关心具体逻辑的代码，轻松地将应用和其他微服务连接起来，可以看作为准中间件或者服务导向架构媒介，api gateway允许企业打包自己的大型企业应用程序，并连接到网络服务上。

api gateway框架有很多，包括kong(Mashape开源)、microgateway(IBM开源)、Zuul、Amazon API网关、Tyk、APIAxle、apiGrove、janus(HelloFresh开源)等，其中[kong](https://getkong.org/)是基于nginx的网关实现，kong最诱人的一个特性是可以通过插件扩展已有功能，这些插件在API请求响应循环的生命周期中被执行。插件使用lua编写，而且kong还有如下几个基础功能：HTTP基本认证、密钥认证、CORS(Cross-origin Resource Sharing，跨域资源共享)、TCP、UDP、文件日志、API 请求限流、请求转发以及nginx监控。

本篇[api gateway](ABOUT.md)主要介绍kong的实现原理，并结合小案例，为开发者提供基于kong的api gateway。

* 初学者可通过实例代码、文档快速学习kong及api gateway，并在社群中交流讨论；
* 已有一定了解的开发者，不必从零开始开发，仅需在云框架基础上替换部分业务代码，可自行添加kong插件。

# 内容概览

* [快速部署](#快速部署)
   * [镜像部署](#镜像部署)
* [框架说明](#框架说明) 
   * [业务](#业务)
      * [业务背景](#业务背景)
      * [业务架构](#业务架构)
      * [业务模块](#业务模块)
   * [组件](#组件)
      * [组件架构](#组件架构)
      * [Spring Cloud Config](#Spring-Cloud-Config)
      * [Netflix Eureka](#Netflix-Eureka)
      * [Netflix Zuul](#Netflix-Zuul)
      * [Netflix Ribbon](#Netflix-Ribbon)
      * [Netflix Hystrix](#Netflix-Hystrix)
      * [Netflix Feign](#Netflix-Feign)
* [如何变成自己的项目](#如何变成自己的项目)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [参与贡献](#参与贡献)
* [加入社群](#加入社群)

# <a name="快速部署">快速部署</a>

## <a name="镜像部署">镜像部署</a>

1. Docker环境准备

   * ubuntu

   1.更新apt包
    ```
    sudo apt-get update
    ```
   2.安装 Docker
    ```
    sudo apt-get install docker-engine
    ```
        
   3.启动 Docker 服务
   ```
   sudo service docker start
   ```
   4.查看docker状态
    ```
    docker info
    ```

   * mac

   请参考[https://docs.docker.com/docker-for-mac/](https://docs.docker.com/docker-for-mac/)

2. 启动两个web站点用于测试
    ```
    docker pull nginx:alpine
    docker run -d -p 9001:80 nginx:alpine
    docker run -d -p 9002:80 nginx:alpine
    ```
3. 启动kong
   ```
    docker pull kong
    docker pull postgres
    docker run -d --name kong-database \
                  -p 5432:5432 \
                  -e "POSTGRES_USER=kong" \
                  -e "POSTGRES_DB=kong" \
                  postgres
    docker run -d --name kong \
                  --link kong-database:kong-database \
                  -e "KONG_DATABASE=postgres" \
                  -e "KONG_PG_HOST=kong-database" \
                  -p 8000:8000 \
                  -p 8443:8443 \
                  -p 8001:8001 \
                  -p 7946:7946 \
                  -p 7946:7946/udp \
                  kong
   ```
4. 启动kong-dashboard
    ```
    docker pull woodyalen202/docker-kong-dashboard
    docker run -d -p 5000:8080 woodyalen202/docker-kong-dashboard
    ```

5. 基于[docker-compose](https://docs.docker.com/compose/install/)运行如下命令（[docker-compose.yml]()）

   ```
   docker-compose -f docker-compose.yml up -d
   ```


> **Endpoints**
>
> http://127.0.0.1:8000 - kong url
> 
> http://127.0.0.1:8001 - kong admin url
>  
> https://127.0.0.1:8443 - kong https url
> 
> http://127.0.0.1:5000 - kong dashboard ui
> 
> http://127.0.0.1:9001 - nginx demo1 url
>
> http://127.0.0.1:9002 - nginx demo2 url
>

# <a name="Kong">Kong</a>
##<a name="Kong描述">Kong描述</a>
Kong是Mashape开源的高性能高可用API网关和API服务管理层。它基于OpenResty，进行API管理，并提供了插件实现API的AOP。
Kong在Mashape管理了超过15,000个API，为200,000开发者提供了每月数十亿的请求支持。非常稳定、高效。
首先我们先了解下Kong这个系统，如下图所示:

![Kong](image/intro-illustration.png)

Kong在运行过程中，客户端请求将先请求Kong服务器，然后它会被代理到最终的API应用。而插件在api响应循环的生命周期中被执行。
Kong的代理方式有两种: 
一是应用通过携带Host头部路由到对应的API应用
二是通过不同的uri路由到API应用
这两种方式都是都是基于Openresty动态增加upstream以及对upstream的DNS resolver来实现。
插件的各个执行周期则是lua在nginx的各个周期对应。

传统的API结构与Kong API结构对比:

![Kong API](image/supervisord.png)


##<a name="Kong插件">Kong插件</a>
Kong默认提供了7类共31种插件(v0.10.2):
* Authentication
![Authentication](image/authentication.png)
* Security
![Security](image/security.png)
* Traffic Control
![Traffic Control](image/trafficcontrol.png)
* Serverless
![Serverless](image/serverless.png)
* Analytics & Monitoring
![Analytics & Monitoring](image/analyticsmonitoring.png)
* Transformations
![Transformations](image/transformations.png)
* Logging
![Logging](image/logging.png)

这些插件可以满足大多数的需求，对于无法满足的业务需求，Kong提供扩展功能，用户可以自定义Kong插件。
后面以2个插件为例来说明Kong的扩展插件步骤。

# <a name="Kong使用">Kong使用</a>
Kong对外提供rest api进行管理。详见[Kong admin api](https://getkong.org/docs/0.10.x/admin-api/)

## <a name="注册API">注册API</a>
使用Kong代理API，首先需要把API注册到Kong。
我们可以通过命令行进行添加:
```
curl -i -X POST \
      --url http://127.0.0.1:8001/apis/ \
      --data 'name=nginxfirst' \
      --data 'hosts=nginxfirst' \
      --data 'upstream_url=http://xx.xx.xx.xx:9001/'
```
可以从返回的数据判断注册是否成功。
也可以通过kong-dashboard(kong的ui管理界面)进行添加:
![kong add api](image/apiadd.png)

创建成功后可以看到API的列表页查看
![kong list api](image/apilist.png)

上面我们将9001的nginx注册到Kong

## <a name="添加用户">添加用户</a>
对于API来讲，有可能没有用户概念，用户可以随意调用。
对于这种情况，Kong提供了一种consumer对象。
consumer是全局共用的，比如某个API启用了key-auth,那么没有身份的访问者就无法调用这个API了。
需要首先创建一个Consumer，然后在key-auth插件中为这个consumer生成一个key。
然后就可以使用这个key来透过权限验证访问API了。

如果另外一个API也开通了key-auth插件，那么这个consumer也是可以通过key-auth验证访问这个API的，如果要控制这种情况，就需要Kong的ACL插件。
一定要记住: 对于Kong来讲，认证与权限乃是两个不同的东西。

![kong key_auth add](image/keyauth.png)


## <a name="API添加插件">API添加插件</a>
Kong默认提供了31种[插件](#Kong插件)。
Kong的插件独立作用于每一个API，不同的API可以使用完全不同的插件。
有的API完全开放，不需要任何认证;
有的API会涉及敏感数据，权限控制需要非常严格;
有的API完全不在乎调用频次或者日志;
有的API则严格限制调用频次或者日志;

可以通过rest api、kong-dashboard添加api的插件。
```
curl -i -X POST \
  --url http://127.0.0.1:8001/apis/nginxfirst/plugins/ \
  --data 'name=key-auth'
```
![kong plugin add](image/pluginadd.png)

上面注册的9001nginx添加了访问控制，所有通过验证的请求可以访问9001nginx;
验证失败请求则无法访问9001nginx。


我们通过命令行可以访问验证:
```
curl -H 'Host: nginxfirst' -H 'TT: e9da671f5c5d44d5bfdca95585283979' http://127.0.0.1:8000
```
![kong key auth success](image/keyauthsucc.png)
```
curl -H 'Host: nginxfirst' http://127.0.0.1:8000
```
![kong key auth success](image/keyauthfailed.png)



# <a name="如何开发自己的Kong插件">Kong插件开发</a>

**步骤：**

1. git clone Kong到本地
    ```
    git clone git@github.com:Mashape/kong.git
    ```
     
2. 创建自定义插件目录
    ```
    cd ${KONG_DIR}
    cd kong
    mkdir custom_plugins
    ```
     
3. 新增插件
    ```
    cd ${KONG_DIR}
    cd kong
    mkdir custom_plugins
    cd custom_plugins
    mkdir xxx
    ```
     
4. 编辑插件的schema、handler
     
5. 执行luaracks make安装插件到本地进行测试
     
6. 制作kong镜像，之后参照[快速部署](#快速部署)，修改镜像名称，部署kong


## <a name="log2zmq"></a>log2zmq

### 插件描述
这个插件的功能：
1、获取请求的日志
2、将日志数据发送到zeromq服务端

在custom_plugins中创建log2zmq目录，
之后添加schema.lua对应API添加log2zmq功能，添加的参数等信息



## <a name="accesslimiting"></a>accesslimiting



     
# <a name="生产环境"></a>生产环境

* `TODO` CI/CD
* `TODO` 扩容
* `TODO` 服务容错
* `TODO` 业务监控／性能分析
* `TODO` K8s部署

# <a name="常见问题"></a>常见问题

任何相关问题均可通过[GitHub ISSUE](https://github.com/cloudframeworks-springcloud/user-guide/issues)提交或讨论，问题总结请查看[[QA](QA.md)]

# <a name="更新计划"></a>更新计划

### Roadmap

* `文档` 增加在线演示
* `组件` 增加组件内容，如Spring Cloud Sleuth、Spring Cloud Consul等
* `生产环境` 增加生产环境下各项扩展操作，如性能测试及各类部署、特性、技术实现等
* `快速部署` 增加好雨云帮部署
* `常见问题` 补充问题总结[QA](QA.md)

点击查看[历史更新](CHANGELOG.md)

# <a name="参与贡献"></a>参与贡献

[如何成为云框架贡献者](CONTRIBUTING.md)

# <a name="加入社群"></a>加入社群

+ QQ群1: 531980120
+ [订阅邮件](http://goodrain.us15.list-manage.com/subscribe?u=1874f1de4ed82a52890cefb4c&id=b88f73ca56)
+ [联系我们](mailto:info@goodrain.com)

-------

[云框架](ABOUT.md)系列主题，遵循[APACHE LICENSE 2.0](LICENSE.md)协议发布。