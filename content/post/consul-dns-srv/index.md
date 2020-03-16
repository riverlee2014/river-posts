---
title: "Consul DNS SRV 服务发现"
date: 2020-01-13T16:24:33+08:00
---

# 前言
根据 Consul 官方文档的说明，它提供了二种类型的 API 接口：HTTP / DNS 。HTTP 是非常容易理解的应用场景，基本上大多数开发语言的 Client SDK 都是基于 HTTP 接口封装的。而本文的重点是实验另一种接口 DNS ，以及它可能的应用场景。

DNS 是一种非常古老的协议，在网络发展史上已经存在比较久，基本上现代大多数操作系统都内置了解析 DNS 的功能。DNS 也是有服务发现、负载均衡、权重控制等能力。最常见的 Linux 发行版下都带有 dig、nslookup 等命令，用来探测解析 DNS 域名。下面，我们来了解一下 `Consul DNS API` 的玩法。

# 准备
假设您已经正确安装完一个 consul cluster，现在先正常注册一个 service 上去。比如，您机器上装了一个服务叫做 hugo，端点提供在 1313 端口。注册信息：
```json
{
  "service": {
    "name": "hugo",
    "tags": ["sit"],
    "port": 1313,
    "check": {
      "tcp": "127.0.0.1:1313",
      "interval": "10s",
      "timeout": "1s"
    }
  }
}
```
注册成功，UI 看起来像如下：


# 查询
consul 的默认 dns 接口暴露在 8600 端口上，而操作系统默认的 DNS 端口为 53 （ 需要 root 权限才能暴露），我们先用 8600 进行 DNS查询：

dig @localhost -p 8600 hugo.service.consul SRV
解析一下：@localhost 表示通过本机查询 DNS，-p 表示 DNS是走 8600 端口，hugo.service.consul 是 consul 注册的服务名称固定格式，hugo 就是上面截图中的 ID，service.consul 是默认的后缀，还有更全的后缀是带 Node、DataCenter 信息的。 SRV 表示是查询 SRV 格式（补充，SRV 是 DNS 发展后来增强一种格式，可以通过名称查询得到  IP 和  Port）。正常查询得到如下结果：



红圈中的部分，就是 SRV 我们关注的查询应答内容段，这里可以看到，查询得到了 hostname 和 port 。你通过查 consul catalog 接口也能得到类似的结果。如果上面的命令你没有带 -p 8600 ，dig 就走向操作系统默认的 dns 解析器，解析器在哪？/etc/resolv.conf 中配置的。

# 过滤
consul 可以启动的时候添加参数 -dnsport = 53 完全接管整个 OS 的 DNS 解析，但是这样要 consul 取得高级权限，不符合常规的安全规范。那么有没有一种办法，让 53 端口即能解析常规的域名（比如 www.baidu.com ），而又能跟 consul 对接解析到它的 service name？因为前面说过了，所有它里面注册的 service 都带有固定后缀：service.consul 。关于这个问题，consul 有一个文章提及：https://learn.hashicorp.com/consul/security-networking/forwarding

这就是 DNS 解析的跳转，其原理就是通过一个第三方组件，让 53 端口操作系统默认监听的解析请求按照一定的规则转发到不同的 DNS Server 上去。有一堆可选现实方案，我们是 CentOS 7，选择第一种 BIND 组件，这个组件在 CentOS 的默认发生版中包含了，如果有光盘 yum 源，直接安装即可：

sudo yum install bind
安装完成，配置文件 /etc/named.conf，修改一些参数，让它能整合 os dns resolver 和 consul dns

// add consul dns zone
include "/etc/named/consul-zone.conf";
 
zone "." IN {
        type hint;
        file "named.ca";
};
// update to no
dnssec-enable no;
// update to no
dnssec-validation no;
注意要把 dnssec-enable yes 和 dnssec-validation yes 二个参数改为 no 。实测如果这 2 个参数不修改，整合不成功，无法通过默认的 53 端口进行解析。

并增加一段 zone 配置，放在 /etc/named/ 目录下，比如叫 consul-zone.conf，注意，上面的 include 放在 zone "." IN 的前面，让它解析的时候优先，保证 consul 注册的域名能优先解析到。

zone "consul" IN {
  type forward;
  forward only;
  forwarders { 127.0.0.1 port 8600; };
};
然后保存，启动 bind

systemctl start named
再测试一下，这个时候，发现 -p 8600 已经可以不要了，也就意味着 53 端口已经承接了标准的域名解析 + Consul 域名解析了！



如果你想将 @localhost 这个参数也去掉，编辑一下 /etc/resolv.conf ，nameserver 127.0.0.1 加在第一行，意味着本机也将作为 dns server 的一项。dig 命令将彻底简化为：

dig hugo.service.consul SRV


到这里， DNS 这个层面就整合完成了。这里再拓展一下思路：只要任何能做 DNS SRV 解析的组件，都能在机器上轻松连接到 Consul 的注册 Service，负载均衡、权重这些是标准的 DNS 定义里面的内容，这里暂不放大讨论。

# 测试
我们用  curl 来测试一下上面的成果，比如这个 consul 集群中有一个服务叫做 nginx，它的全名就是：nginx.service.consul 。



这样就直接解析到 nginx 这个服务后面的具体 IP 去了，curl 如果你不带端口，默认 http 是 80，https 是 443，刚好这个 nginx service 是注册了 port 为 80。

这里就引申出一个小问题：如果任何安装在这上面的软件不支持 DNS SRV，你就只能用 DNS A，也就是解析 name → ip 。

比如，有一个服务叫 prometheus，它提供的端口是 9090，你如果试：

curl http://prometheus.service.consul
是解析不成功的，因为 curl 只支持 DNS A，它会认为 prometheus.service.consul 是一个 hostname，你要请求它的 80 端口。



而带上 9090 则会成功



这也证明，在进行服务注册后，consul 提供了 DNS A 和 SRV 等多种查询格式。所以，尽管提供了  SRV 格式，但是服务发现这个能力，具备不具备，还是取决于具体的 Client 程序，curl 在官方的邮件列表中明确表示，SRV 还是正在开发中的功能：



应用场景
具备 DNS SRV 解析能力的软件，可以和  Consul 做无缝的集成，充分利用 Consul 的服务注册、健康检测功能 。
使用最基本的 DNS A 格式，可以串联起公司 DNS Server 以外的一些服务命名
可以打通传统虚拟机和容器之间的交互，全部走 DNS 解析，根据格式将不同的域名转到不同的解析器上
可以根据定制自己的 SDK，将服务发现、健康检测、权重控制等基本功能下沉交给 DNS 去解决，简化 SDK 复杂度。