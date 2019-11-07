---
title: consul-and-nginx
date: 2019-10-29 19:25:29
tags:
   - consul
   - nginx
---

# 使用consul实现nginx动态负载均衡

## consul简介

consul 是一个 service mesh解决方案，主要提供服务发现、健康检查、KV存储、数据中心等功能。

## 安装consul

consul 使用go语言实现，只需要到官网[https://www.consul.io/downloads.html](https://www.consul.io/downloads.html)下载对应平台的二进制文件即可。 linux和windows平台均可以放在环境变量已有路径或者另外配置环境变量。

## 运行 agent

consul 完成安装后，需要运行agent 。 agent可以分为 server 和 client 模式，每个数据中心至少必须有一台 server ，建议一个集群里有3到5个 server ，部署单一的 server，在出现失败时会不可避免的造成数据丢失。

## consul 架构

consul官方架构图如图所示:
![consul-arch](https://raw.githubusercontent.com/paulzjw/paulzjw.github.io/src/source/_posts/2019-10-29-consul-and-nginx/consul-arch.png)

图中包含了consul多数据中心的设计，本次只讲上面单数据中心的情况。consul推荐3-5台server做集群，其他节点和server通信通过client代理。

consul client设计为sidecar模式。使用client代理的好处，所有操作都只用对本机client进程操作，读操作(如服务发现、kv读)减轻consul集群压力，写操作(如服务注册、kv写等)由client代理通过rpc转发到consul集群，解耦一些业务不相关的操作，如建康检查等。

## consul配合nginx使用实现动态负载均衡

### 只使用nginx做负载均衡

一般只使用nginx做负载均衡的设计如下图所示：
![consul-arch](https://raw.githubusercontent.com/paulzjw/paulzjw.github.io/src/source/_posts/2019-10-29-consul-and-nginx/Snipaste_2019-11-01_15-50-58.jpg)

nginx单独部署，然后将请求负载均衡到对应服务器上的具体业务服务。
这样做的好处是使用了nginx带来的好处，缺点是nginx负载均衡只支持静态配置文件，业务服务下线或者上线，需要手动修改配置文件重启nginx生效。

### consul配合nginx使用

使用conusl做为注册中心，可以将设计改为如下图所示：
![consul-arch](https://raw.githubusercontent.com/paulzjw/paulzjw.github.io/src/source/_posts/2019-10-29-consul-and-nginx/Snipaste_2019-11-01_16-12-17.jpg)

所有微服务的服务注册和服务发现由consul集群管理，微服务只需要实现/调用服务注册，服务发现和建康检查3个http接口，在服务启动时将服务注册请求到本机consul client即可。

我们将nginx的upstream配置使用consul-template写出模板文件，然后启动consul-template进程监听。当有微服务下线或者上线新的微服务，consul-template能根据模板文件更新nginx配置文件，并执行自定义命令(重启nginx)，这样就完成了微服务上下线/维护/扩容时，nginx负载均衡做动态更新。

consul集群的高可用需要部署3-5台consul server，多了影响网络性能，小于3台则consul server不能出现单机故障。

nginx所在机子的高可用可以使用keepalived，主从配置，从机可同主机配置和启动方式一样，这样微服务在线状态更新时，从机的nginx配置也是最新的，如果主机故障了，从机可以顺利切换。

微服务的高可用理论上(单机能够满足业务qps的情况）只需相同的微服务部署大于1台机子即可，这样一台机子故障了，请求还能负载均衡到另一台上继续业务。

讲了使用consul, consul-template,nginx配合进行动态负载均衡的情况，也总结一下缺点：

1.如果网络情况不好的情况，可能会出现具体业务微服务频繁上下线导致频繁更新nginx配置，nginx频繁重启。这点可以考虑适当加大心跳时间来避免偶尔网络状况带来的心跳检测问题，或者将consul-template配置成定时同步(配置会更新不实时，但是如果大量上下线微服务可以只用更新一次配置，重启一次)。

2.nginx重启多少会对性能带来一些影响，nginx reload机制保证已经接收请求的进程在处理完请求再退出，但是不满足http keepalive。
consul支持dns查询，nginx也支持配置dns，可以考虑使用nginx dns查询代替consul-template，这样不需要更新nginx配置和重启nginx。但是原生nginx不支持dns srv，所以如果出现一台机子上部署相同微服务使用不同端口，nginx dns查询方式就不能正确支持并负载均衡了。

## consul 集群部署

### consul server单节点启动命令示例

```sh
consul agent -bind=ip -server=true -bootstrap -client=0.0.0.0 -ui -data-dir=/path/to/data -config-dir=/path/to/config -datacenter=datacenter_name -node=node_name
```

`-bind`在多网卡时需要使用，多网卡时，consul不清楚绑定哪个网卡，需要指定ip，单网卡时可以不填。

`-sever`模式表示consul agent以client还是server模式启动，默认为server模式，以client模式启动需修改为-server=false,

`-bootstrap`表示以bootstrap模式启动，单节点时必填，不然会因为没有leader无法正常启动，同命令`-bootstrap-expect`互斥，后续部署集群再继续讲。

`-client` 配置ip表示可以访问consul http api这个server监听的ip，如果为127.0.0.1只能本机访问。

`-ui`表示启动consul默认web界面。-data-dir为consul保存节点信息的路径，必填。

`-config-dir`为consul配置文件信息路径，可填可不填，在配置文件路径里的json文件。

`-datacenter`指定数据中心名称，不填默认dc1,同一数据中心下的名称要求一定相同。

`-node`表示节点名称，不填默认pc的名称，建议按照一定格式规范节点名

更多命令参数和解释可以参考官网:[https://www.consul.io/docs/agent/options.html#command-line-options](https://www.consul.io/docs/agent/options.html#command-line-options)

### consul 集群部署命令示例

consul可以以两种方式启动集群，一种是指定leader启动，先启动一个leader，再依次启动剩下节点，然后执行命令 `consul join leader ip`。leader启动命令同上，带上`-bootstrap`参数。

另一种启动方式为不指定leader启动三个节点，启动命令同上，`-bootstrap`替换为`-bootstrap-expecp=3`，三个节点都启动完后因为没有leader和不知道彼此，集群还不能使用，在三个节点依次执行命令`consul join ip1 ip2 ip3` 三台机子都执行完后，会进行选举，然后集群可以使用。

判断集群是否可以使用可以登录consul自带的web页面，若报错则表示未正常运行，web页面默认是8500端口，如图所示为一个三节点集群，leader为五角星标志的机子：![cluster.jpg](https://raw.githubusercontent.com/paulzjw/paulzjw.github.io/src/source/_posts/2019-10-29-consul-and-nginx/Snipaste_2019-11-04_19-19-28.jpg)

## consul client启动

consul client启动命令同consul server大致相同，区别 -server=false必须为false，不能使用-bootstrap或者-bootstrap-expect参数，其他同server启动命令一致。
consul client加入器群节点也很简单，可以在启动时加入-join=consul server中其中一台的ip，也可先启动，再执行命令`consul join server ip`

```sh
consul agent -bind=ip -server=false -client=0.0.0.0 -ui -data-dir=/path/to/data -config-dir=/path/to/config -datacenter=datacenter_name -node=node_name -join=ip
```

或者不带上`-join`，启动完后执行`consul join ip`

consul client不可脱离consul server运行，至少要有一个consul server节点，且client要join server节点才可使用

## consul-tempalte使用

### 安装conusl-template

到官网[https://releases.hashicorp.com/consul-template](https://releases.hashicorp.com/consul-template)下载对应平台的二进制文件即可，部署方式同consul。

### 编写consul-template模板

{%raw%}
```tpl
upstream web {
   {{ range service "web" }}
   server {{ .Address }}:{{ .Port }} weight=1;
   {{ end }}
}
```
{%endraw%}

示例为一个upstream.tpl文件，`range service "web"`和`end`类似for循环， `web`表示服务发现名称为web的服务，ip和端口不做解释。

nginx.conf里include这部分模板生成的对应文件,如图所示：![config](https://raw.githubusercontent.com/paulzjw/paulzjw.github.io/src/source/_posts/2019-10-29-consul-and-nginx/Snipaste_2019-11-04_19-54-23.jpg)

### 启动consul-template

```sh
consul-template -consul-addr 127.0.0.1:8500 -template "./upstream.tpl:./conf/upstream.conf:nginx -s reload"
```

如上所示为一个consul-template监控集群微服务上下线状况后并更新相应nginx配置的启动命令。

`-consul-addr`指定 consul client/server的ip和http端口，此处我们还是使用consul推荐的连本机client的方式，所以ip是127.0.0.1，端口默认是8500。consul不限制server可以被使用，也可以将ip配置成consul集群的里其中一个consul server的ip，推荐还是使用当前方式，即consul推荐的方式。

`-template`表示执行的模板命令,upstream.tpl为之前编写的consul-template模板文件的具体路径，upstream.conf为生成的配置文件，同上面截图`include upstream.conf`相对应，`nginx -s reload`生成完配置文件后重启nignx，也可以自定义别的命令。
这样当有服务在线状态变化，upstream.conf文件会自动生成，并重启nginx动态更新nginx的负载均衡。

更多命令和用法参考官网:[https://github.com/hashicorp/consul-template#command-line-flags](https://github.com/hashicorp/consul-template#command-line-flags)
