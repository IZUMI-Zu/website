---
title: How to configure traffic routing
description: 
weight: 10
---

## 安装示例程序

如果你还没有安装示例程序，请参照 [快速开始](/docs/v1.0/quickstart/) 安装 Aeraki，Istio 及示例程序。

安装完成后，可以看到集群中增加了下面两个 NS，这两个 NS 中分别安装了基于 MetaProtocol 实现的 Dubbo 和 Thrift 协议的示例程序。
你可以选用任何一个程序进行测试。

```bash
➜  ~ kubectl get ns|grep meta
meta-dubbo        Active   16m
meta-thrift       Active   16m
```

## 请求级别的负载均衡

Istio 会使用 TCP proxy 来代理非 HTTP 协议的客户端请求，同一个客户端 TCP 连接上发出的所有请求都会被发送到一个服务器实例。这导致了一个问题：当客户端使用长连接时，多个服务器实例收到的请求不够均衡，当服务端压力过大时，即使及时扩容也不能将已有服务端的压力分担出去。

Aeraki 支持对基于 MetaProtocol 开发的任何协议进行七层（请求级别）负载均衡，因此在不进行任何配置的情况下，客户端的代理会将请求均匀发送到两个不同版本的服务器端。
下面我们用 aerakictl 命令来查看客户端的应用日志，可以看到同一个客户端连接上的多个请求被依次发送到了 v1 和 v2 两个服务器端。

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
```

## 按任意属性将请求路由到某个指定版本的服务

MetaProtocol 支持了非常灵活的路由匹配条件，任何可以从协议数据包中解析出来的属性都能用于路由匹配条件。

> 备注：Aeraki 会按照服务的 VIP 建立 Listener，每个服务独享 Listener，避免了 HTTP 协议的同端口多服务带来的路由表膨胀问题，路由表中只包含本服务相关的路由信息，极大地提高了路由查询效率。

创建一条 MetaRouter 路由规则，将请求路由到 v1：

```bash
kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
    - thrift-sample-server.meta-thrift.svc.cluster.local
  routes:
    - name: v1
      match:
        attributes:
          method:
            exact: sayHello
      route:
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
            subset: v1
EOF
```

使用 aerakictl 命令来查看客户端的应用日志，可以看到客户端的所有请求都被路由到了 v1 版本：

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
```

## 流量拆分

使用 MetaRouter 路由规则将客户端的流量按照指定比例发送到不同版本的服务。

```bash
kubectl apply -f- <<EOF
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: MetaRouter
metadata:
  name: test-metaprotocol-thrift-route
  namespace: meta-thrift
spec:
  hosts:
    - thrift-sample-server.meta-thrift.svc.cluster.local
  routes:
    - name: traffic-split
      match:
        attributes:
          method:
            exact: sayHello
      route:
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
            subset: v1
          weight: 20
        - destination:
            host: thrift-sample-server.meta-thrift.svc.cluster.local
            subset: v2
          weight: 80
EOF
```

使用 aerakictl 命令来查看客户端的应用日志，可以看到客户端的请求按照 MetaRouter 中设置的指定比例发送到了 v1 和 v2：

```bash
➜  ~ aerakictl_app_log client meta-thrift -f --tail 10
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v2-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-6d5bcc885-wglpc/172.17.0.93
Hello Aeraki, response from thrift-sample-server-v1-5c8476684-hr8hh/172.17.0.92
```

## 理解原理

在向 Sidecar Proxy 下发的配置中， Aeraki 在服务对应的 Outbound Listener 的 FilterChain 中设置了 MetaProtocol Proxy，并在 MetaProtocol Proxy 配置中指定 Aeraki 为 RDS 服务器。

Aeraki 会将 MetaRouter 中配置的路由规则翻译为 MetaProtocol Proxy 的路由规则，通过 Aeraki 内置的 RDS 服务器下发给 MetaProtocol Proxy。

可以通过下面的命令查看 sidecar proxy 的配置：

``` bash
aerakictl_sidecar_config client meta-thrift |fx
```

其中 Thrift 服务的 Outbound Listener 中的 MetaProtocol Proxy 配置如下所示：

```yaml
{
 "name": "envoy.filters.network.meta_protocol_proxy",
 "typed_config": {
  "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
  "type_url": "type.googleapis.com/aeraki.meta_protocol_proxy.v1alpha.MetaProtocolProxy",
  "value": {
   "stat_prefix": "outbound|9090||thrift-sample-server.meta-thrift.svc.cluster.local",
   "application_protocol": "thrift",
   "rds": {
    "config_source": {
     "api_config_source": {
      "api_type": "GRPC",
      "grpc_services": [
       {
        "envoy_grpc": {
         "cluster_name": "aeraki-xds"
        }
       }
      ],
      "transport_api_version": "V3"
     },
     "resource_api_version": "V3"
    },
    "route_config_name": "thrift-sample-server.meta-thrift.svc.cluster.local_9090"
   },
   "codec": {
    "name": "aeraki.meta_protocol.codec.thrift"
   },
   "meta_protocol_filters": [
    {
     "name": "aeraki.meta_protocol.filters.router"
    }
   ]
  }
 }
}
```

在导出的文件中还可以查看到目前 Proxy 中生效的 RDS 路由信息，如下所示：

```yaml
{
@type": "type.googleapis.com/aeraki.meta_protocol_proxy.admin.v1alpha.RoutesConfigDump",
dynamic_route_configs": [
{
 "version_info": "1641896797",
 "route_config": {
  "@type": "type.googleapis.com/aeraki.meta_protocol_proxy.config.route.v1alpha.RouteConfiguration",
  "name": "thrift-sample-server.meta-thrift.svc.cluster.local_9090",
  "routes": [
   {
    "name": "traffic-split",
    "match": {
     "metadata": [
      {
       "name": "method",
       "exact_match": "sayHello"
      }
     ]
    },
    "route": {
     "weighted_clusters": {
      "clusters": [
       {
        "name": "outbound|9090|v1|thrift-sample-server.meta-thrift.svc.cluster.local",
        "weight": 20
       },
       {
        "name": "outbound|9090|v2|thrift-sample-server.meta-thrift.svc.cluster.local",
        "weight": 80
       }
      ],
      "total_weight": 100
     }
    }
   }
  ]
 },
 "last_updated": "2022-01-11T10:26:37.357Z"
}
```







