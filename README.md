# Linux 透明代理
## 什么是正向代理？
代理软件通常分为客户端（client）和服务端（server），server 运行在境外服务器（通常为 Linux 服务器），client 运行在本地主机（如 Windows、Linux、Android、iOS），client 与 server 之间通常使用 tcp 或 udp 协议进行数据通信。大多数 client 被实现为一个 http、socks5 代理服务器，一个软件如果想通过 client 进行科学上网，需要使用 http、socks5 协议与 client 进行数据交互，这是绝大多数人的使用方式。这种代理方式，我们称之为 **正向代理**。所谓正向代理就是，一个软件如果想要使用 client 的代理服务，需要经过特定的设置，否则不会经过 client 的代理。

## 什么是透明代理？
在正向代理中，一个软件如果想走 client 的代理服务，我们必须显式配置该软件，对该软件来说，有没有走代理是很明确的，大家都“心知肚明”。而透明代理则与正向代理相反，当我们设置好合适的防火墙规则（仅以 Linux 的 iptables 为例），我们将不再需要显式配置这些软件来让其经过代理或者不经过代理（直连），因为这些软件发出的流量会自动被 iptables 规则所处理，那些我们认为需要代理的流量，会被通过合适的方法发送到 client 进程，而那些我们不需要代理的流量，则直接放行（直连）。这个过程对于我们使用的软件来说是完全透明的，软件自身对其一无所知。这就叫做 **透明代理**。注意，所谓透明是对我们使用的软件透明，而非对 client 或 server 透明，理解这一点非常重要。

## 透明代理如何工作？
在正向代理中，期望使用代理的软件会通过 http、socks5 协议与 client 进程进行交互，以此完成代理操作。而在透明代理中，我们的软件发出的流量是完全正常的流量，并没有像正向代理那样，使用 http、socks5 等专用协议，这些流量经过 iptables 规则的处理后，会被通过“合适的方法”发送给 client 进程（当然是指那些我们认为需要走代理的流量）。注意，此时 client 进程接收到不再是 http、socks5 协议数据，而是经过 iptables 处理的“透明代理数据”，“透明代理数据”从本质上来说与正常数据没有区别，只是多了一些“元数据”在里面，使得 client 进程可以通过 netfilter 或操作系统提供的 API 接口来获取这些元数据（元数据其实就是原始目的地址和原始目的端口）。那么这个“合适的方法”是什么？目前来说有两种：
- REDIRECT：只支持 TCP 协议的透明代理。
- TPROXY：支持 TCP 和 UDP 协议的透明代理。

因此，对于 TCP 透明代理，有两种实现方式，一种是 REDIRECT，一种是 TPROXY；而对于 UDP 透明代理，只能通过 TPROXY 方式来实现。为方便叙述，本文以 **纯 TPROXY 模式** 指代 TCP 和 UDP 都使用 TPROXY 来实现，以 **REDIRECT + TPROXY 模式** 指代 TCP 使用 REDIRECT 实现，而 UDP 使用 TPROXY 来实现，有时候简称 **REDIRECT 模式**，它们都是一个意思。

## 此脚本的作用及由来
通过上面的介绍，其实可以知道，在构建透明代理的过程中，需要的仅仅是 iptables、iproute2 以及 ss/ssr/v2ray 等支持透明代理的软件，那 ss-tproxy 脚本的作用是什么呢？如果你尝试搭建过透明代理，那么你就会体会到，这一过程其实并不容易，你需要设置许多繁琐的 iptables 规则，还要应对国内复杂的 DNS 环境，另外还要考虑 UDP 透明代理的支持，此外你通常还希望这个透明代理能实现分流操作，而不是一股脑的全走代理。于是就有了 ss-tproxy 脚本，该脚本的目的就是辅助大家快速地搭建一个透明代理环境，该透明代理支持 gfwlist、chnroute 等常见分流模式，以及一个无污染的 DNS 解析服务；除此之外，ss-tproxy 脚本不做任何其它事情；因此你仍然需要 iptables、iproute2 以及 ss/ssr/v2ray 等支持透明代理的软件，因为透明代理的底层服务是由它们共同运作的，理解这一点非常重要。

> 为什么叫做 `ss-tproxy`？因为该脚本最早的时候只支持 ss 的透明代理，当然现在它并不局限于特定的代理软件。

另外还有一点需要注意，透明代理使用的 client 与正向代理使用的 client 通常是不同的，因为正向代理的 client 是 http、socks5 服务器，而透明代理的 client 则是透明代理服务器，它们之间有本质上的区别。对于 ss，你需要使用 ss-libev 版本（ss-redir），ssr 则需要使用 ssr-libev 版本（ssr-redir），而对于 v2ray，配置好 `dokodemo-door` 入站协议即可。再次强调，透明代理只是 client 不同，并不关心你的 server 是什么版本，因此你的 vps 上，可以运行所有与之兼容的 server 版本，以 ss/ssr 为例，你可以使用 python 版的 ss、ssr，也可以使用 golang 版的 ss、ssr 等等，只要它们之间可以兼容。

ss-tproxy 可以运行在 Linux 软路由/网关、Linux 物理机、Linux 虚拟机等环境中，可以透明代理 ss-tproxy 主机本身以及所有网关指向 ss-tproxy 主机的其它主机的 TCP、UDP 流量。也就是说，你可以在任意一台 Linux 主机上部署 ss-tproxy 脚本，然后同一局域网内的其它主机可以随时将其网关及 DNS 指向 ss-tproxy 主机，这样它们的 TCP 和 UDP 流量就会自动走代理了。

**ss-tproxy v4.0 简介**
- 去除不常用的 `global` 分流模式
- 支持 IPv4、IPv6 双协议栈的透明代理（可配置）
- 使用 [chinadns-ng](https://github.com/zfl9/chinadns-ng) 替代原版 chinadns，修复若干问题
- 完美兼容"端口映射"，只代理"主动出站"的流量，规则更加细致化
- 支持配置要代理的黑名单端口，这样可以比较好的处理 BT/PT 流量
- 支持自定义 dnsmasq/chinadns 端口，支持加载外部 dnsmasq 配置
- ss-tproxy stop 后，支持重定向内网主机发出的 DNS 到本地直连 DNS
- 支持网络可用性检查，无需利用其它的 hook 来避免脚本自启失败问题
- 脚本逻辑优化及结构调整，尽量提高脚本的可移植性，去除非核心依赖

v4.0 只剩下 `gfwlist`、`chnroute`、`chnlist` 3 种分流模式，相关介绍：
- `gfwlist` 分流模式：`gfwlist.txt` 中的域名走代理，其余走直连，即黑名单模式。
- `chnroute` 分流模式：除了国内地址、保留地址之外，其余均走代理，即白名单模式。
- `chnlist` 分流模式：本质还是 `gfwlist` 模式，只是域名列表为国内域名，即回国模式。

## 相关依赖
核心依赖：
- `iptables`：核心部件，用于配置 IPv4 的透明代理规则。
- `ip6tables`：核心部件，用于配置 IPv6 的透明代理规则。
- `ip`：通常位于 iproute2 软件包，用于配置策略路由（TPROXY）。
- `ipset`：ipset 用于存储 gfwlist 的黑名单 IP，以及 chnroute 的白名单 IP。
- `dnsmasq`：构建 DNS 服务，对于 gfwlist 模式，该 dnsmasq 需要支持 `--ipset` 选项。
- `chinadns-ng`：chnroute 模式的 DNS 服务，注意是 [chinadns-ng](https://github.com/zfl9/chinadns-ng)，而不是原版 chinadns。

> 如果某些模式你基本不用，那么对应的依赖就不用管。比如，你不打算使用 IPv6 透明代理，则无需关心 ip6tables，又比如你不打算使用 chnroute 模式，也无需关心 chinadns-ng，安装依赖之前先检查当前系统是否已有对应依赖。

可选依赖：
- `curl`：用于更新 chnlist、gfwlist、chnroute 分流模式的相关列表。
- `base64`：用于更新 gfwlist 的域名列表，gfwlist.txt 是 `base64` 格式编码的。
- `perl`：用于更新 gfwlist 的域名列表，gfwlist.txt 是 `Adblock Plus` 规则，要进行转换。

[ss-tproxy 脚本相关依赖的安装方式参考](https://www.zfl9.com/ss-redir.html#%E5%AE%89%E8%A3%85%E4%BE%9D%E8%B5%96)

## 安装脚本
```bash
git clone https://github.com/zfl9/ss-tproxy
cd ss-tproxy
chmod +x ss-tproxy
cp -af ss-tproxy /usr/local/bin
mkdir -p /etc/ss-tproxy
cp -af ss-tproxy.conf gfwlist* chnroute* /etc/ss-tproxy
cp -af ss-tproxy.service /etc/systemd/system # 可选，安装 service 文件
```

## 卸载脚本
```bash
ss-tproxy stop
ss-tproxy flush-dnsredir
ss-tproxy delete-gfwlist
rm -fr /usr/local/bin/ss-tproxy /etc/ss-tproxy # 删除脚本及配置文件
```
> 升级脚本前请先卸载脚本，如果有残留规则无法清除，请务必重启系统。

## 文件列表
- `ss-tproxy`：shell 脚本，欢迎各位大佬一起来改进此脚本。
- `ss-tproxy.conf`：配置文件，本质是 shell 脚本，修改需重启生效。
- `ss-tproxy.service`：systemd 服务文件，用于 ss-tproxy 的开机自启。
- `chnroute.set`：IPv4 国内地址及保留地址的 ipset 文件，不要手动修改。
- `chnroute6.set`：IPv6 国内地址及保留地址的 ipset 文件，不要手动修改。
- `gfwlist.txt`：存储 gfwlist、chnlist 分流模式的黑名单域名，不要手动修改。
- `gfwlist.ext`：存储 gfwlist、chnlist 分流模式的扩展黑名单，可配置，重启生效。

> ss-tproxy 只是一个 shell 脚本，并不是常驻后台的服务，因此所有的修改都需要 restart 来生效。

## 配置说明
- 注释：井号开头的行为注释行，配置文件本质上是一个 shell 脚本，对于同名变量或函数，后定义的会覆盖先定义的。
- `mode`：分流模式，默认为 chnroute，可根据需要改为 gfwlist，需要说明的是，`chnlist` 模式的 mode 为 `gfwlist`。
- `ipv4/ipv6`：启用 IPv4/IPv6 透明代理，你需要确保本机代理进程能正确处理 IPv4/IPv6 相关数据包，脚本不检查它。
- `tproxy`：true 为纯 TPROXY，false 为 REDIRECT/TPROXY 混合，ss/ssr 只能使用 false，v2ray 经配置后可使用 true。
- `proxy_svraddr4/6`、`proxy_svrport`、`proxy_tcpport`、`proxy_udpport`、`proxy_startcmd`、`proxy_stopcmd` 见后。
- `dnsmasq_bind_port`：dnsmasq 监听端口，默认 53，如果端口已被占用则修改为其它未占用的端口，如 `60053`。
- `dnsmasq_conf_dir/dnsmasq_conf_file`：dnsmasq 外部配置文件/目录，被作为 `conf-dir`、`conf-file` 选项值。
- `ipts_set_snat`：是否设置 IPv4 的 MASQUERADE 规则，通常保持为 false 即可。有两种情况需要将其设置为 true：第一种，ss-tproxy 部署在出口路由位置且确实需要 MASQUERADE 规则（即该主机至少两张网卡，一张连接内网，一张连接公网，要进行源地址转换）；第二种，在设置为 false 的情况下，代理不正常，那么也需要将其改为 true，曾经遇到过这种情况，可能与路由器的配置有关。注意，MASQUERADE 规则在 ss-tproxy stop 仍然是有效的，如果你想清空这些残留规则，可以执行 `ss-tproxy flush-postrule` 命令。
- `ipts_set_snat6`：是否设置 IPv6 的 MASQUERADE 规则，默认为 true，通常你必须将其设置为 true，因为 IPv6 的透明代理需要利用 ULA 内网地址，如果设置为 false，那么其它内网主机将无法访问 IPv6 网络，因为 ULA 地址是不可以在公网上被路由的，因此你必须进行 SNAT 源地址转换。除非你使用 GUA 地址进行透明代理，但这样会让配置变得复杂且不易使用，因为 GUA 地址是动态变化的，运营商不会给你分配一个固定的 GUA 地址段。
- `ipts_intranet/ipts_intranet6`：填写需要代理的内网网段，对于 IPv6，你从 ISP 分配到的应该是 GUA 公网地址，但是 GUA 地址是随时会变化的（DHCP），这对于透明代理来说可是个糟糕的事情，因为你不可能实时的去修改 ipts_intranet6 选项，因此通常的做法是，给内网设备分配一个固定 ULA 私有网段，这样每台内网设备就有了两个 IPv6 地址，一个是 GUA 公网地址，一个是 ULA 私有地址，ss-tproxy 主机的 IPv6 网关应保持不变，即还是原来的 GUA 自动分配的网关，以保证 ss-tproxy 主机能够正常进行 IPv6 访问，其它内网主机的 IPv6 网关则需要修改为 ss-tproxy 主机的 ULA 地址，这样它们发出的流量才会经过 ss-tproxy，然后在 ipts_intranet6 中填写此私有地址段即可。其实这就相当于 IPv4 的经典配置，ss-tproxy 主机有一个公网地址（GUA）且还有一个私有地址（ULA），ss-tproxy 主机的默认网关是 GUA 地址，因此可直接访问 IPv6 公网，而其它内网主机则只有一个私有地址（ULA，虽然它还有一个 GUA 地址，但它似乎并没有什么作用了，因为上网是通过 ULA 地址进行的），它们的默认网关为 ss-tproxy 主机（ULA 地址），然后我们会在 ss-tproxy 主机上配置 SNAT/MASQUERADE 规则（`ipts_set_snat6`），这样这些 ULA 内网主机就可以上网了。
- `ipts_reddns_onstop`：当 ss-tproxy stop 之后，是否使用 iptables 规则将内网主机发往 ss-tproxy 主机的 DNS 请求重定向至本地直连 DNS（即 `dns_direct/dns_direct6`），为什么要这么做呢？因为其它内网主机的 DNS 是指向 ss-tproxy 主机的，但是现在我们已经关闭了 ss-tproxy（dnsmasq 进程关闭了），所以这些内网主机会因为无法解析 DNS 而无法正常上网，而设置此选项后，这些 DNS 请求会被重定向给 114.114.114.114 等国内直连 DNS，这样它们就又可以正常上网了，在 ss-tproxy start 前，这些规则会自动删除，如果你需要手动删除这些规则，可以执行 `ss-tproxy flush-postrule` 命令。该选项的默认值为 true，注意如果 ss-tproxy 主机上有正常运行的 DNS 服务，那么这个选项其实就可以设置为 false 了。
