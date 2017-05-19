# [云框架]KONG API Gateway

![](https://img.shields.io/badge/release-v0.1-green.svg)
[![](https://img.shields.io/badge/Producer-lucienu2-orange.svg)](CONTRIBUTORS.md)
![](https://img.shields.io/badge/License-Apache_2.0-blue.svg)

当我们决定对应用进行微服务改造时，应用客户端如何与微服务交互的问题也随之而来，毕竟服务数量的增加会直接导致部署授权、负载均衡、通信管理、分析和改变的难度增加。

面对以上问题，API GATEWAY是一个不错的解决方案，其所提供的访问限制、安全、流量控制、分析监控、日志、请求转发、合成和协议转换功能，可以解放开发者去把精力集中在具体逻辑的代码，而不是把时间花费在考虑如何解决应用和其他微服务链接的问题上。

在众多API GATEWAY框架中，Mashape开源的高性能高可用API网关和API服务管理层——[KONG](https://getkong.org/)（基于NGINX）特点尤为突出，它可以通过插件扩展已有功能，这些插件（使用lua编写）在API请求响应循环的生命周期中被执行。于此同时，KONG本身提供包括HTTP基本认证、密钥认证、CORS、TCP、UDP、文件日志、API请求限流、请求转发及NGINX监控等基本功能。目前，Kong在Mashape管理了超过15,000个API，为200,000开发者提供了每月数十亿的请求支持。

本篇[云框架](ABOUT.md)将结合一个**数据查询应用**介绍KONG API GATEWAY及其最佳实践方法。

# 内容概览

* [快速部署](#快速部署)
* [框架说明-业务](#框架说明-业务)
* [框架说明-组件](#框架说明-组件)
   * [组件架构](#组件架构)
   * [KONG基本使用](#KONG基本使用)
   * [ROUTING实现](#ROUTING)
   * [AUTHENTICATION实现](#AUTHENTICATION)
   * [SECURITY实现](#SECURITY)
   * [TRAFFIC CONTROL实现](#TRAFFICCONTROL)
   * [LOGGING实现](#LOGGING)
* [KONG插件开发](#KONG插件开发)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [社群贡献](#社群贡献)

# <a name="快速部署"></a>快速部署

1. [准备Docker环境](https://github.com/cloudframeworks-apigateway/user-guide-apigateway/blob/master/READMORE/install%20docker.md)

2. 启动两个web站点用于测试

   ```
   docker pull goodraincloudframeworks:springdata
   docker run -d -p 8080:8080 goodraincloudframeworks:springdata
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
   docker pull goodraincloudframeworks/docker-kong-dashboard
   docker run -d -p 5000:5000 goodraincloudframeworks/docker-kong-dashboard
   ```

5. 基于[docker-compose](https://docs.docker.com/compose/install/)运行如下命令

   ```
   docker-compose -f docker-compose.yml up -d
   ```

6. 访问路径

   http://127.0.0.1:8000 - kong url
   
   http://127.0.0.1:8001 - kong admin url
   
   https://127.0.0.1:8443 - kong https url
   
   http://127.0.0.1:5000 - kong dashboard ui
   
   https://172.16.0.133:8080/api/persons - user api url
   
   https://172.16.0.133:8080/api/newinfos - newinfo api url

# <a name="框架说明-业务"></a>框架说明-业务

数据查询应用，顾名思义提供简单的数据查询服务，应用对外提供两个端口：

* user：处理敏感数据，如限制访问

* newinfo：普通数据，对所有人开发

对比以上两个端口，我们可以相对清楚的理解KONG及其插件的效果和配置方法。

本例数据查询应用业务架构比较简明，如下图所示：

<div align=center><img width="900" height="" src="./image/kong业务架构.png"/></div>

# <a name="框架说明-组件"></a>框架说明-组件

本例使用KONG本身实现ROUTING，并添加了[OAuth 2.0](https://getkong.org/plugins/oauth2-authentication/)（AUTHENTICATION实现）、[IP Restriction](https://getkong.org/plugins/ip-restriction/)（SECURITY实现）、[Rate Limiting](https://getkong.org/plugins/rate-limiting/)（TRAFFIC CONTROL实现）、[File](https://getkong.org/plugins/file-log/)（LOGGING实现)等4个插件。（[查看更多官方插件](https://getkong.org/plugins/)）

以下管理配置通过命令行实现。KONG也可以通过UI管理界面进行管理和配置，方法请参考[KONG DASHBOARD](./READMORE/kongdashboard方法.md)。

<a name="组件架构"></a>组件架构如下图所示：

<div align=center><img width="900" height="" src="./image/kong组件架构.png"/></div>

* newinfo端口仅通过KONG实现与user端口的路由，其他插件未使用

   + KONG代理方式包括：1）应用通过携带HOST头部路由到对应的API应用；2）通过不同的uri路由到API应用
    
   + 以上两种方式均为基于Openresty动态增加upstream以及对upstream的DNS resolver来实现
    
   + 客户端将先请求KONG服务器，并被代理到最终的API应用，而插件在API响应循环的生命周期中被执行

* user端口信息敏感，限制访问用户；newinfo端口信息不敏感，无需限制访问用户（AUTHENTICATION实现）

* user端口控制访问地址，仅规定IP可访问；newinfo端口无此限制（SECURITY实现）

* user端口控制访问频率，newinfo端口可无限制访问（TRAFFIC CONTROL实现）

* user端口可获取每次访问日志（LOGGING实现）

## <a name="KONG基本使用"></a>KONG基本使用 （需匹配实例）

### <a name="注册API"></a>注册API

使用Kong代理API，首先需要把API注册到Kong，并通过返回数据查看注册是否成功：

```
curl -i -X POST \
      --url http://127.0.0.1:8001/apis/ \
      --data 'name=personapi' \
      --data 'hosts=personapi' \
      --data 'upstream_url=https://172.16.0.133:8080/api/persons'
```

### <a name="添加用户"></a>添加用户

API可能没有用户概念，会出现随意调用的情况。为此Kong提供了一种consumer对象（全局共用），如某API启用了key-auth，没有身份的访问者将无法调用该API。

首先创建一个consumer，然后在key-auth插件中为这个consumer生成一个key，然后就可以使用这个key来透过权限验证访问API了。

```
curl -X POST \
    --data "username=oauthadmin" \
    --data "custom_id=personapi"
    http://127.0.0.1:8001/consumers/ \
```
```
curl -X POST \
    http://127.0.0.1:8001/consumers/personapi/key-auth \
```

<div align=center><img width="600" height="" src="./image/keyauth.png"/></div>

需要注意的是：

* 若另一API也开通了key-auth插件，那么这个consumer也是可以通过key-auth验证访问这个API的，想要控制这种情况，需借助Kong的[ACL插件](https://getkong.org/plugins/acl/)

* 对于Kong来讲，认证与权限是两个不同的东西

### <a name="API添加插件"></a>API添加插件

目前，Kong默认提供了31种插件，插件独立作用于每一个API，不同的API可以使用完全不同的插件。

这是一种非常科学的设计，因为在实际情况中很可能会出现有的API完全开放，不需要任何认证，有的API会涉及敏感数据，权限控制需要非常严格；有的API完全不在乎调用频次或者日志，有的API则严格限制调用频次或者日志等类似情况。

我们可以通过如下命令添加插件：

```
curl -i -X POST \
  --url http://127.0.0.1:8001/apis/personapi/plugins/ \
  --data 'name=key-auth'
```

并通过命令行进行访问验证：

```
curl -H 'Host: personapi' -H 'TT: 78182b121a074fe6961555d802e40b3b' http://127.0.0.1:8000
```

<div align=center><img width="600" height="" src="./image/keyauthsucc.png"/></div>

```
curl -H 'Host: personapi' http://127.0.0.1:8000/
```

<div align=center><img width="600" height="" src="./image/keyauthfailed.png"/></div>

## <a name="ROUTING"></a>ROUTING实现

user端口和newinfo端口之间实现路由，需先将服务注册到Kong，外部访问将统一走api gateway代理。

* 注册user api

```
curl -i -X POST \
      --url http://127.0.0.1:8001/apis/ \
      --data 'name=personapi' \
      --data 'hosts=personapi' \
      --data 'upstream_url=https://172.16.0.133:8080/api/persons'
```

* 注册newinfo api

```
curl -i -X POST \
      --url http://127.0.0.1:8001/apis/ \
      --data 'name=newinfoapi' \
      --data 'hosts=newinfoapi' \
      --data 'upstream_url=https://172.16.0.133:8080/api/newinfos'
```

注册成功后即可通过Kong代理访问用户信息（user端口）、新闻信息（newinfo端口）

```
curl -H 'Host: personapi' http://127.0.0.1:8000

[
    {"pid":1,"name":"lucien","age":30},
    {"pid":2,"name":"Joe","age":28},
    {"pid":3,"name":"smith","age":32},
    {"pid":4,"name":"Tod","age":56},
    {"pid":5,"name":"linken","age":34},
    {"pid":6,"name":"truple","age":23},
    {"pid":7,"name":"tdt","age":20}
]
```

```
curl -H 'Host: newinfoapi' http://127.0.0.1:8000

[
    {"nid":1,"title":"一路一代代","content":"what happending...."},
    {"nid":2,"title":"雪中悍刀行","content":"人生三不朽,立功立德立言"}
]
```
<div align=center><img width="600" height="" src="./image/kong-proxyperson.png"/></div>

<div align=center><img width="600" height="" src="./image/kong-proxynewinfo.png"/></div>

此时，可以将用户信息、新闻通知对外访问控制限制为只有Kong可以访问，外部请求全部通过Kong进行代理。

## <a name="AUTHENTICATION"></a>AUTHENTICATION实现

通过OAuth 2.0 Authentication插件实现user端口的用户访问限制，

* 注册Oauth2插件，参见[配置说明](https://getkong.org/plugins/oauth2-authentication/#configuration)。

```
curl -X POST \
     --data 'name=oauth2' \
     --data 'enable_password_grant=true' \
     --data 'provision_key=qwe1238amsdh23' \
     --data 'config.scopes=read,write' \
     http://127.0.0.1:8001/apis/personapi/plugins 
```

* 添加Consumer及Consumer对应的credentials

```
curl -X POST \
    --data "username=oauthadmin" \
    --data "custom_id=personapi"
    http://127.0.0.1:8001/consumers/ \
```
```
curl -X POST \
    --data "name=oauthadmin" \
    --data "redirect_uri=https://172.16.0.133:8080/api/persons"
    http://127.0.0.1:8001/consumers/personapi/oauth2
```

* 申请accesstoken并访问
```
curl -k -H 'Host: personapi' \
    --data "client_id=5bee1b6679e5463599d7ce64b14c2795" \
    --data "client_secret=54f2a058f30f46e8b5ccc8d6788eb081" \
    --data "provision_key=qwe1238amsdh23" \
    --data "authenticated_userid=b48bf407-c2b7-41a9-8e0f-43eead2fc60f" 
    --data "grant_type=password" 
    https://127.0.0.1:8443/oauth2/token

{
    "refresh_token":"e87d871957eb4717bb0002054ae8c9a3",
    "token_type":"bearer",
    "access_token":"bad2a7ee579e4389880ae29b3610c639",
    "expires_in":7200
}
```

使用token访问user api

```
curl -H 'Host: personapi' \
     -H 'Authorization: bearer bad2a7ee579e4389880ae29b3610c639' \
     http://127.0.0.1:8000

[
    {"pid":1,"name":"lucien","age":30},
    {"pid":2,"name":"Joe","age":28},
    {"pid":3,"name":"smith","age":32},
    {"pid":4,"name":"Tod","age":56},
    {"pid":5,"name":"linken","age":34},
    {"pid":6,"name":"truple","age":23},
    {"pid":7,"name":"tdt","age":20}
]
```

不使用token访问user api
```
curl -H 'Host: personapi' http://127.0.0.1:8000

{
    "error_description":"The access token is missing",
    "error":"invalid_request"
}
```


newinfo端口由于数据不敏感，无需特殊配置。

## <a name="SECURITY"></a>SECURITY实现

通过添加IP Restriction插件，实现对user端口的访问限制，即仅规定IP可访问。

* 为user端口添加IP Restriction插件扩展，并设置白名单（只有名单内的IP可以访问API）。

```
curl -X POST \
     --data 'name=ip-restriction' \
     --data 'config.whitelist=172.17.0.1' \
     http://127.0.0.1:8001/apis/personapi/plugins 
```

白名单内IP访问：

```
curl -H 'Host: personapi' http://127.0.0.1:8000

[
    {"pid":1,"name":"lucien","age":30},
    {"pid":2,"name":"Joe","age":28},
    {"pid":3,"name":"smith","age":32},
    {"pid":4,"name":"Tod","age":56},
    {"pid":5,"name":"linken","age":34},
    {"pid":6,"name":"truple","age":23},
    {"pid":7,"name":"tdt","age":20}
]
```

<div align=center><img width="600" height="" src="./image/kong-proxyperson.png"/></div>

其他IP访问：

```
curl -H 'Host: personapi' http://172.17.0.1:8000

{
    "message":"Your IP address is not allowed"
}
```

<div align=center><img width="600" height="" src="./image/kong-proxyperson-ipfail.png"/></div>

newinfo端口无需配置此插件。

## <a name="TRAFFICCONTROL"></a>TRAFFIC CONTROL实现

user端口通过Rate Limiting插件控制用户访问频率，避免无限制访问。

* 为user端口添加Rate Limiting插件扩展，设置为1分钟内只能访问1次

```
curl -X POST \
     --data 'name=rate-limiting' \
     --data 'config.minute=1' \
     http://127.0.0.1:8001/apis/personapi/plugins 
```

正常访问展示:

```
curl -H 'Host: personapi' http://127.0.0.1:8000

[
    {"pid":1,"name":"lucien","age":30},
    {"pid":2,"name":"Joe","age":28},
    {"pid":3,"name":"smith","age":32},
    {"pid":4,"name":"Tod","age":56},
    {"pid":5,"name":"linken","age":34},
    {"pid":6,"name":"truple","age":23},
    {"pid":7,"name":"tdt","age":20}
]
```

<div align=center><img width="600" height="" src="./image/kong-proxyperson.png"/></div>

超出次数的访问展示:

```
curl -H 'Host: personapi' http://127.0.0.1:8000

{
    "message":"API rate limit exceeded"
}
```

<div align=center><img width="600" height="" src="./image/kong-proxyperson-ratefail.png"/></div>

newinfo端口无需配置此插件。

## <a name="LOGGING"></a>LOGGING实现

user端口通过File-log插件实现对于每次访问日志的获取，需要注意为日志文件写权限，日志格式参考[Log Format](https://getkong.org/plugins/file-log/#log-format)。

* 为user端口添加File-log插件，并设置为日志文件路径设为:/tmp/file.log

```
curl -X POST \
     --data 'name=file-log' \
     --data 'config.path=/tmp/file.log' \
     http://127.0.0.1:8001/apis/personapi/plugins 
```

* 添加日志插件后，每次访问都会被记录

<div align=center><img width="600" height="" src="./image/kong-proxyperson-filelog.png"/></div>

newinfo端口无需配置此插件。

# <a name="KONG插件开发"></a>KONG插件开发

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
     
4. 编辑插件的schema.lua、handler.lua, 根据实际情况完成插件逻辑（[lua教程](http://www.runoob.com/lua/lua-tutorial.html)）

5. 修改`${KONG_DIR}/templates/kong_defaults.lua`，配置custom_plugins=xxx

6. 执行luaracks make安装插件到本地进行测试
     
7. 制作kong镜像，并[快速部署](#快速部署)

[KONG插件开发示例：log2zmq](https://github.com/cloudframeworks-apigateway/user-guide-apigateway/blob/master/READMORE/KONG%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91%E7%A4%BA%E4%BE%8B%EF%BC%9Alog2zmq.md)

[KONG插件开发示例：accesslimiting](https://github.com/cloudframeworks-apigateway/user-guide-apigateway/blob/master/READMORE/KONG%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91%E7%A4%BA%E4%BE%8B%EF%BC%9Aaccesslimiting.md)

# <a name="#生产环境"></a>生产环境

`TODO`

# <a name="#常见问题"></a>常见问题

`TODO`

# <a name="#更新计划"></a>更新计划

* `组件` 增加SERVERLESS实现
* `组件` 增加ANALYTICS&MONITORING实现
* `组件` 增加TRANSFORMATIONS实现

# <a name="#社群贡献"></a>社群贡献

+ QQ群: 117317266
+ [参与贡献](CONTRIBUTING.md)
+ [联系我们](mailto:info@goodrain.com)

-------

[云框架](ABOUT.md)系列主题，遵循[APACHE LICENSE 2.0](LICENSE.md)协议发布。
