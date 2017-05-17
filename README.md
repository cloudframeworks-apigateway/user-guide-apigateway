# [云框架]KONG API Gateway

![](https://img.shields.io/badge/release-v0.1-green.svg)
[![](https://img.shields.io/badge/Producer-woodyalen202-orange.svg)](CONTRIBUTORS.md)
![](https://img.shields.io/badge/License-Apache_2.0-blue.svg)

随着微服务的流行，服务数量急剧增加，连接服务正在激增，部署授权、负载均衡、通信管理、分析和改变的难度也变得越来越大。api gateway可以提供访问限制、安全、流量控制、分析监控、日志、请求转发、合成和协议转换功能，api gateway可以让开发者集中精力去关心具体逻辑的代码，轻松地将应用和其他微服务连接起来，可以看作为准中间件或者服务导向架构媒介，api gateway允许企业打包自己的大型企业应用程序，并连接到网络服务上。

Api Gateway框架有很多，包括kong(Mashape开源)、microgateway(IBM开源)、Zuul、Amazon API网关、Tyk、APIAxle、apiGrove、janus(HelloFresh开源)等，其中[kong](https://getkong.org/)是基于nginx的网关实现，kong最诱人的一个特性是可以通过插件扩展已有功能，这些插件在API请求响应循环的生命周期中被执行。插件使用lua编写，而且kong还有如下几个基础功能：HTTP基本认证、密钥认证、CORS(Cross-origin Resource Sharing，跨域资源共享)、TCP、UDP、文件日志、API 请求限流、请求转发以及nginx监控。

本篇[云框架](ABOUT.md)主要介绍kong的实现原理，并结合小案例，为开发者提供基于kong的api gateway。

* 初学者可通过实例代码、文档快速学习kong及api gateway，并在社群中交流讨论；
* 已有一定了解的开发者，不必从零开始开发，仅需在云框架基础上替换部分业务代码，可自行添加kong插件。

# 内容概览

* [快速部署](#快速部署)
* [框架说明-业务](#框架说明-业务)
* [框架说明-组件](#框架说明-组件)
   * [组件架构](#组件架构)
   * [KONG基本使用](#KONG基本使用)
      * [注册API](#注册API)
      * [添加用户](#添加用户)
      * [API添加插件](#API添加插件)
   * [ROUTING实现](#ROUTING)
   * [AUTHENTICATION实现](#AUTHENTICATION)
   * [SECURITY实现](#SECURITY)
   * [TRAFFIC CONTROL实现](#TRAFFICCONTROL)
   * [LOGGING实现](#LOGGING)
* [KONG插件开发](#KONG插件开发)
   * [开发流程](#开发流程)
   * [开发示例1:log2zmq](#log2zmq)
   * [开发实例2:accesslimiting](#accesslimiting)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [社群贡献](#社群贡献)

# <a name="快速部署"></a>快速部署

@丽川

# <a name="框架说明-业务"></a>框架说明-业务

# <a name="框架说明-组件"></a>框架说明-组件

## <a name="组件架构"></a>组件架构

## <a name="KONG基本使用"></a>KONG基本使用

### <a name="注册API"></a>注册API

### <a name="添加用户"></a>添加用户

### <a name="API添加插件"></a>API添加插件

## <a name="ROUTING"></a>ROUTING实现

## <a name="AUTHENTICATION"></a>AUTHENTICATION实现

## <a name="SECURITY"></a>SECURITY实现

## <a name="TRAFFICCONTROL"></a>TRAFFIC CONTROL实现

## <a name="LOGGING"></a>LOGGING实现

# <a name="KONG插件开发"></a>KONG插件开发

## <a name="KONG开发流程"></a>KONG开发流程

## <a name="#log2zmq"></a>开发示例1:log2zmq

## <a name="#accesslimiting"></a>开发示例1:accesslimiting

# <a name="#常见问题"></a>常见问题

`TODO`

# <a name="#更新计划"></a>更新计划

* `组件` 增加SERVERLESS实现
* `组件` 增加ANALYTICS&MONITORING实现
* `组件` 增加TRANSFORMATIONS实现

# <a name="#社群贡献"></a>社群贡献

+ QQ群: 
+ [参与贡献](CONTRIBUTING.md)
+ [联系我们](mailto:info@goodrain.com)

-------

[云框架](ABOUT.md)系列主题，遵循[APACHE LICENSE 2.0](LICENSE.md)协议发布。
