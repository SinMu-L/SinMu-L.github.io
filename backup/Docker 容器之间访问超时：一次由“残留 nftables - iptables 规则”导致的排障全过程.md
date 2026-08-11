
> **问题类型：** Docker 容器间网络连接超时
> **环境：** Ubuntu + Docker 29.6.1（Snap 安装）+ iptables-nft + nftables + iptables-persistent
> **典型现象：** 容器 A 无法访问容器 B 的端口，但容器 B 本身正常、宿主机通过域名访问也正常
> **最终原因：** Docker 网络重建后，系统中残留了一套针对旧 Docker bridge `br-8b52009aaf0d` 的 `raw/PREROUTING DROP` 和 NAT 规则，而当前网络已经变成 `br-7c31020cfc24`。这些旧规则通过 iptables-nft 映射到了 nftables，并继续参与实际数据包处理。

---

# 一、问题背景

这次故障的应用拓扑比较简单：

```text
Internet
   │
   ▼
Cloudflare
   │
   ▼
Nginx Proxy Manager
   │
   │ Docker network: nginxproxymanager_default
   │ subnet: 172.18.0.0/16
   │
   ├── 172.18.0.2  Nginx Proxy Manager
   │
   └── 172.18.0.3  New API
                     │
                     └── TCP/3000
```

正常情况下：

```text
nginxproxymanager-app-1
        │
        │ http://172.18.0.3:3000
        ▼
      new-api
```

应该直接建立 TCP 连接。

但是最开始实际情况是：

```bash
docker exec nginxproxymanager-app-1 \
  sh -c 'curl -v --connect-timeout 5 http://172.18.0.3:3000/'
```

结果：

```text
* Trying 172.18.0.3:3000...
* Connection timed out after 5002 milliseconds
curl: (28) Connection timed out after 5002 milliseconds
```

也就是说：

**DNS、HTTP、应用层都还没有进入讨论范围，TCP 三次握手就已经失败了。**

---

# 二、第一阶段：确认 New API 本身没有挂

首先需要排除“应用没监听 3000”。

通过容器内部访问：

```bash
docker exec nginxproxymanager-app-1 \
  sh -c 'curl -v --connect-timeout 5 http://172.18.0.3:3000/'
```

最初是：

```text
Trying 172.18.0.3:3000...
Connection timed out
```

所以我们开始怀疑：

* New API 没有监听 3000
* Docker bridge 有问题
* veth 有问题
* Docker firewall 有问题
* nftables / iptables 丢包
* Docker 网络重建后遗留旧规则

---

# 三、检查 Docker Bridge

当前 Docker 网络：

```bash
docker network inspect nginxproxymanager_default \
  --format '{{.Name}} {{.Id}} {{json .IPAM.Config}}'
```

后来确认：

```text
nginxproxymanager_default
7c31020cfc24...
[
  {
    "Gateway":"172.18.0.1",
    "Subnet":"172.18.0.0/16"
  }
]
```

对应 Linux bridge：

```bash
ip link show br-7c31020cfc24
```

结果：

```text
br-7c31020cfc24:
    UP
    LOWER_UP
    state UP
```

说明当前 bridge 本身是正常存在的。

---

# 四、检查 bridge 上的 veth

执行：

```bash
bridge link
```

得到：

```text
veth71b014f@eth0 ... master br-7c31020cfc24 state forwarding
veth3e229d4@eth0 ... master br-7c31020cfc24 state forwarding
```

这非常重要。

说明：

```text
container
   │
   ▼
veth
   │
   ▼
br-7c31020cfc24
```

链路本身是：

```text
UP + LOWER_UP + forwarding
```

并不是典型的 bridge/veth 掉线。

---

# 五、检查 FDB

继续：

```bash
bridge fdb show br br-7c31020cfc24
```

可以看到两个 veth 的 MAC：

```text
ae:79:a1:32:14:48 dev veth71b014f master br-7c31020cfc24
62:0d:4a:dd:44:1d dev veth3e229d4 master br-7c31020cfc24
```

这说明 bridge 的 forwarding database 也正常学习到了端口。

所以基本可以排除：

> “Docker bridge 没有把两个容器接起来”。

---

# 六、最关键的一步：抓包

为了判断数据包到底有没有到达 bridge，我们开始抓包。

## 6.1 在容器对应 veth 上抓包

```bash
sudo tcpdump -ni veth3e229d4 \
  'host 172.18.0.2 and port 3000'
```

然后从 NPM：

```bash
docker exec nginxproxymanager-app-1 \
  sh -c 'curl -v --connect-timeout 5 http://172.18.0.3:3000/'
```

没有看到预期的 TCP 流量。

---

## 6.2 在 Docker bridge 上抓包

进一步：

```bash
sudo tcpdump -ni br-7c31020cfc24 \
  'host 172.18.0.2 and host 172.18.0.3 and port 3000'
```

然后再次：

```bash
curl http://172.18.0.3:3000
```

仍然没有看到正常的 SYN → SYN/ACK。

这说明问题非常可能发生在：

```text
进入 Docker bridge / 容器网络之前
```

或者：

```text
iptables / nftables
```

---

# 七、检查 Docker Firewall

这时候查看：

```bash
sudo nft -a list table ip filter
```

发现 Docker 创建了类似：

```text
chain DOCKER-FORWARD
chain DOCKER-USER
chain DOCKER-BRIDGE
chain DOCKER-CT
chain DOCKER
```

其中当前 bridge：

```text
br-7c31020cfc24
```

是正常的。

例如：

```text
iifname "br-7c31020cfc24" counter ... accept
```

以及：

```text
oifname "br-7c31020cfc24" ...
```

所以一开始看起来：

> Docker 当前生成的 firewall 规则没有明显问题。

---

# 八、真正的突破：检查 raw 表

后来检查：

```bash
sudo nft -a list table ip raw
```

出现了非常关键的规则：

```text
table ip raw {
    chain PREROUTING {
        type filter hook prerouting priority raw;
        policy accept;

        iifname != "br-8b52009aaf0d" \
            ip daddr 172.18.0.2 \
            counter ... drop

        iifname != "br-8b52009aaf0d" \
            ip daddr 172.18.0.3 \
            counter ... drop

        iifname != "br-8b52009aaf0d" \
            ip daddr 172.18.0.4 \
            counter ... drop
    }
}
```

这里出现了一个非常可疑的地方：

**当前 Docker bridge 是：**

```text
br-7c31020cfc24
```

但是规则检查的却是：

```text
br-8b52009aaf0d
```

而且后者已经不是当前 Docker 网络。

---

# 九、为什么这个规则非常危险？

规则：

```text
iifname != "br-8b52009aaf0d"
ip daddr 172.18.0.3
drop
```

含义是：

> 任何进入主机、目标地址为 `172.18.0.3`，并且入口接口不是 `br-8b52009aaf0d` 的数据包，全部 DROP。

问题在于：

```text
br-8b52009aaf0d
```

已经不存在。

后来确认：

```bash
ip link show br-8b52009aaf0d
```

返回：

```text
Device "br-8b52009aaf0d" does not exist.
```

而当前网络是：

```text
br-7c31020cfc24
```

所以这实际上变成了：

```text
目标 172.18.0.3
        │
        ▼
入口接口不是一个已经不存在的 bridge
        │
        ▼
DROP
```

这就是为什么：

```text
NPM → New API
```

会出现：

```text
TCP connection timeout
```

而不是：

```text
Connection refused
```

---

# 十、为什么之前 Docker 网络明明已经换了，旧规则还在？

这才是这次问题最值得记录的地方。

我们进一步搜索：

```bash
sudo grep -R "br-8b52009aaf0d" \
  /etc /usr/local /opt /root 2>/dev/null
```

发现：

```text
/etc/iptables/rules.v4
```

里面存在：

```text
-A PREROUTING -d 172.18.0.2/32 ! -i br-8b52009aaf0d -j DROP
-A PREROUTING -d 172.18.0.3/32 ! -i br-8b52009aaf0d -j DROP
-A PREROUTING -d 172.18.0.4/32 ! -i br-8b52009aaf0d -j DROP
```

同时还有：

```text
-A POSTROUTING -s 172.18.0.0/16 ! -o br-8b52009aaf0d -j MASQUERADE
```

以及：

```text
-A DOCKER ! -i br-8b52009aaf0d \
  -p tcp \
  -j DNAT \
  --to-destination 172.18.0.4:3000
```

这基本把整个问题串起来了。

---

# 十一、系统中还安装了 iptables-persistent

继续检查：

```bash
sudo systemctl status netfilter-persistent --no-pager
```

结果：

```text
Active: active (exited)
```

并且：

```bash
sudo systemctl is-enabled netfilter-persistent
```

得到：

```text
enabled
```

同时：

```bash
dpkg -l | grep -E 'iptables-persistent|netfilter-persistent'
```

得到：

```text
iptables-persistent
netfilter-persistent
```

所以这里存在一个非常重要的机制：

```text
/etc/iptables/rules.v4
        │
        ▼
netfilter-persistent
        │
        ▼
系统启动 / 服务加载
        │
        ▼
iptables rules
```

---

# 十二、为什么 nftables 也能看到这些规则？

这台机器使用的是：

```bash
iptables -V
```

结果：

```text
iptables v1.8.7 (nf_tables)
```

进一步：

```bash
update-alternatives --display iptables
```

确认：

```text
iptables -> /usr/sbin/iptables-nft
```

也就是说：

```text
iptables
   │
   ▼
iptables-nft
   │
   ▼
nftables
```

所以：

> `/etc/iptables/rules.v4` 虽然看起来是 iptables 配置，但加载后实际进入的是 nftables backend。

因此你看到：

```bash
iptables-save
```

里面有旧规则，

同时：

```bash
nft list ruleset
```

里面也有旧规则，

**并不矛盾。**

实际上它们是同一个 netfilter backend 的不同观察入口。

---

# 十三、为什么 nftables.service 没有运行？

这里还有一个容易误判的地方。

检查：

```bash
sudo systemctl status nftables --no-pager
```

结果：

```text
nftables.service
Active: inactive (dead)
```

很容易产生一个错误判断：

> “nftables 服务没运行，所以 nftables 规则不可能是它造成的。”

实际上不是这样。

因为：

```text
iptables-nft
```

本身就可以直接向 kernel 的 nftables subsystem 写入规则。

所以：

```text
nftables.service inactive
```

并不能证明：

```text
kernel nftables ruleset 是空的
```

这是本次排障非常重要的经验。

---

# 十四、另一个关键点：Docker 是 Snap 安装

后来检查：

```bash
docker info | grep -E 'Server Version|Docker Root Dir'
```

发现：

```text
Server Version: 29.6.1
Docker Root Dir: /var/snap/docker/common/var-lib-docker
```

再：

```bash
which docker
```

得到：

```text
/snap/bin/docker
```

查看 dockerd：

```bash
ps -ef | grep '[d]ockerd'
```

发现：

```text
dockerd \
  --group docker \
  --exec-root=/run/snap.docker \
  --data-root=/var/snap/docker/common/var-lib-docker \
  --pidfile=/run/snap.docker/docker.pid \
  --config-file=/var/snap/docker/3579/config/daemon.json
```

所以这不是传统 apt 安装的：

```bash
systemctl restart docker
```

而是：

```text
Snap Docker
```

正确重启方式：

```bash
sudo snap restart docker.dockerd
```

之前执行：

```bash
sudo systemctl restart docker
```

会得到：

```text
Failed to restart docker.service: Unit docker.service not found.
```

这不是 Docker 出问题，而是：

> Docker daemon 根本不是由 `docker.service` 管理。

---

# 十五、重启 Docker 后，网络重新生成

执行：

```bash
sudo snap restart docker.dockerd
```

之后重新检查：

```bash
docker network inspect nginxproxymanager_default \
  --format '{{.Name}} {{.Id}} {{json .IPAM.Config}}'
```

得到：

```text
nginxproxymanager_default
7c31020cfc24...
172.18.0.0/16
```

容器：

```bash
docker inspect nginxproxymanager-app-1 new-api \
  --format '{{.Name}} {{range $n,$v := .NetworkSettings.Networks}}{{$n}}={{$v.IPAddress}} {{end}}'
```

得到：

```text
/nginxproxymanager-app-1 nginxproxymanager_default=172.18.0.3
/new-api nginxproxymanager_default=172.18.0.2
```

也就是说当前拓扑已经稳定：

```text
br-7c31020cfc24
       │
       ├── 172.18.0.2
       │
       └── 172.18.0.3
```

---

# 十六、先做备份，再修改持久化规则

这一步非常重要。

在修改 `/etc/iptables/rules.v4` 前，先：

```bash
sudo cp /etc/iptables/rules.v4 \
  /etc/iptables/rules.v4.backup-before-cleanup-$(date +%Y%m%d-%H%M%S)
```

这样即使清理错误，也可以恢复。

之后检查：

```bash
sudo grep -nE 'br-8b52009aaf0d|172.18.0.4' \
  /etc/iptables/rules.v4
```

确认旧 Docker 网络规则存在。

随后从：

```text
/etc/iptables/rules.v4
```

删除旧网络相关配置。

最终文件中不再存在：

```text
br-8b52009aaf0d
172.18.0.4
```

而保留正常 Docker 规则。

---

# 十七、但是仅仅修改 rules.v4 还不够

这是本次排障第二个非常重要的经验。

我们已经修改：

```text
/etc/iptables/rules.v4
```

但内核当前已经加载的 nftables 规则不会因为你修改文件而自动消失。

所以还需要检查：

```bash
sudo nft -a list ruleset | grep 'br-8b52009aaf0d'
```

当时仍然能看到：

```text
iifname != "br-8b52009aaf0d" ...
```

说明：

```text
磁盘上的配置
```

和：

```text
kernel 当前运行中的规则
```

是两个不同层面的东西。

---

# 十八、删除运行中的旧规则

先删除旧 NAT：

```bash
sudo nft delete rule ip nat POSTROUTING handle 8
```

然后删除 raw PREROUTING：

```bash
sudo nft delete rule ip raw PREROUTING handle 5
sudo nft delete rule ip raw PREROUTING handle 3
```

注意：

> `handle` 是根据当时 `nft -a list ruleset` 输出得到的，不能照抄到另一台机器。

正确流程永远是：

```bash
sudo nft -a list ruleset
```

找到目标规则的：

```text
# handle N
```

然后：

```bash
sudo nft delete rule <family> <table> <chain> handle N
```

例如：

```bash
sudo nft delete rule ip raw PREROUTING handle 3
```

---

# 十九、清理后验证

再次：

```bash
sudo nft -a list ruleset | grep 'br-8b52009aaf0d'
```

最终：

```text
OK: 旧 bridge 规则已全部清除
```

同时：

```bash
ip link show br-8b52009aaf0d
```

结果：

```text
Device "br-8b52009aaf0d" does not exist.
```

这时候：

```text
旧 bridge：不存在
旧 nft 规则：不存在
旧持久化规则：不存在
当前 Docker bridge：br-7c31020cfc24
```

状态终于一致了。

---

# 二十、最终验证：NPM → New API

执行：

```bash
docker exec nginxproxymanager-app-1 \
  sh -c 'curl -fsS --connect-timeout 5 \
  http://172.18.0.3:3000/ >/dev/null && echo "NPM -> NewAPI OK"'
```

得到：

```text
NPM -> NewAPI OK
```

之前：

```text
Connection timed out
```

现在：

```text
HTTP 200
```

问题彻底解决。

---

# 二十一、为什么公网访问一直正常？

这是这次故障最容易让人误判的地方。

公网：

```text
https://newapi.sinmu.xyz
```

一直可以：

```text
HTTP/2 200
x-new-api-version: v1.0.0-rc.24
```

甚至 Cloudflare：

```text
server: cloudflare
```

也正常。

这并不能证明：

```text
Docker 内部网络正常
```

因为公网请求走的是另一条路径：

```text
Internet
   ↓
Cloudflare
   ↓
宿主机 443
   ↓
Docker NAT
   ↓
NPM
   ↓
New API
```

而容器内部请求：

```text
NPM
 ↓
172.18.0.3:3000
 ↓
Docker bridge
 ↓
New API
```

两条路径经过的 netfilter hook、接口、DNAT/SNAT 都可能不同。

因此：

> **公网正常 ≠ Docker 容器间网络正常。**

---

# 二十二、为什么 curl 是 timeout，而不是 connection refused？

这个现象也非常有诊断价值。

如果：

```text
172.18.0.3:3000
```

没有程序监听，通常更容易看到：

```text
Connection refused
```

而这次：

```text
Connection timed out
```

更符合：

```text
SYN
 ↓
某个 firewall rule DROP
 ↓
没有 SYN/ACK
 ↓
curl 等待重传
 ↓
timeout
```

因此遇到：

```text
Docker container A
   ↓
container B:PORT
   ↓
Connection timed out
```

应该优先考虑：

* nftables
* iptables
* Docker FORWARD
* raw PREROUTING
* bridge filtering
* policy routing
* 安全软件
* conntrack

而不是第一时间重启应用。

---

# 二十三、最终根因链条

整个事故可以浓缩成下面这条链：

```text
旧 Docker 网络
    │
    └── br-8b52009aaf0d
             │
             ├── 172.18.0.2
             ├── 172.18.0.3
             └── 172.18.0.4
             │
             ▼
    被删除 / 重建 Docker 网络
             │
             ▼
新 Docker 网络
    │
    └── br-7c31020cfc24
             │
             ├── 172.18.0.2
             └── 172.18.0.3

但是旧的 firewall 配置没有同步清理
             │
             ▼
/etc/iptables/rules.v4
             │
             ▼
netfilter-persistent
             │
             ▼
iptables-nft
             │
             ▼
nftables kernel ruleset
             │
             ▼
raw PREROUTING
             │
             │ 目标 172.18.0.3
             │
             │ iif != br-8b52009aaf0d
             ▼
            DROP
             │
             ▼
NPM → New API
             │
             ▼
TCP SYN 无响应
             │
             ▼
Connection timed out
```

---

# 二十四、这次故障真正的根因

可以明确写成：

> **Docker 网络重建后，旧 Docker bridge `br-8b52009aaf0d` 对应的 iptables/nftables 规则没有被清理。由于系统使用 `iptables-nft` backend，这些持久化 iptables 规则实际存在于 nftables ruleset 中。旧规则位于 `raw/PREROUTING`，针对 `172.18.0.2/3/4` 做了基于旧 bridge 名称的 DROP；当前 Docker 网络已经切换到 `br-7c31020cfc24`，导致容器间访问 `172.18.0.3:3000` 的 TCP SYN 被提前丢弃，最终表现为连接超时。**

---

# 二十五、以后遇到同类问题，推荐按照这个顺序排查

这是最值得保存的一部分。

---

## Step 1：确认目标容器 IP

```bash
docker inspect <container> \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

例如：

```text
172.18.0.3
```

---

## Step 2：确认应用是否监听

进入目标容器：

```bash
docker exec -it <container> sh
```

然后：

```bash
ss -lntp
```

或者：

```bash
netstat -lntp
```

确认：

```text
0.0.0.0:3000
```

或者：

```text
172.18.0.3:3000
```

---

## Step 3：从另一个容器直接测试

```bash
docker exec <source-container> \
  curl -v --connect-timeout 5 http://172.18.0.3:3000/
```

判断：

```text
Connection refused
```

还是：

```text
Connection timed out
```

---

## Step 4：确认 Docker network

```bash
docker network inspect <network>
```

重点看：

```text
Subnet
Gateway
Containers
```

---

## Step 5：确认 Linux bridge

```bash
ip link
```

找：

```text
br-xxxxxxxxxxxx
```

然后：

```bash
ip -d link show br-xxxxxxxxxxxx
```

---

## Step 6：检查 veth

```bash
bridge link
```

以及：

```bash
bridge fdb show br br-xxxxxxxxxxxx
```

确认：

```text
veth
   ↓
bridge
   ↓
forwarding
```

---

# 二十六、Step 7：抓包

这是最重要的定位手段。

先：

```bash
tcpdump -ni <bridge> \
  'host <source-ip> and host <destination-ip> and port <port>'
```

再：

```bash
docker exec <source> \
  curl -v --connect-timeout 5 http://<destination-ip>:<port>/
```

如果完全看不到 SYN：

```text
应用
 ↓
？？？
 ↓
bridge
```

说明包可能在更早阶段被过滤。

---

# 二十七、Step 8：检查 nftables

不要只看：

```bash
nft list table ip filter
```

一定要看：

```bash
sudo nft -a list ruleset
```

特别关注：

```text
raw
filter
nat
```

重点搜索：

```bash
sudo nft -a list ruleset | grep -E \
'172\.18\.|br-[0-9a-f]+'
```

---

# 二十八、Step 9：检查旧 bridge

这是本次事故最关键的检查。

```bash
docker network ls
```

找到当前网络：

```text
7c31020cfc24
```

然后：

```bash
ip link show br-7c31020cfc24
```

再搜索所有规则：

```bash
sudo nft -a list ruleset | grep 'br-'
```

如果发现：

```text
当前：
br-7c31020cfc24

规则：
br-8b52009aaf0d
```

立即重点调查。

---

# 二十九、Step 10：检查 iptables-persistent

```bash
systemctl status netfilter-persistent
```

以及：

```bash
systemctl is-enabled netfilter-persistent
```

检查：

```bash
sudo grep -nE '172\.18\.|br-' \
  /etc/iptables/rules.v4
```

如果这里存在已经不存在的 Docker bridge：

```text
br-xxxxxxxxxxxx
```

就非常可疑。

---

# 三十、Step 11：确认 iptables backend

```bash
iptables -V
```

如果：

```text
(nf_tables)
```

再：

```bash
update-alternatives --display iptables
```

确认：

```text
iptables-nft
```

那么：

```text
iptables-save
```

和：

```text
nft list ruleset
```

需要结合起来看。

---

# 三十一、Step 12：修改前一定备份

```bash
sudo cp /etc/iptables/rules.v4 \
  /etc/iptables/rules.v4.backup-$(date +%Y%m%d-%H%M%S)
```

然后才修改：

```bash
sudo vim /etc/iptables/rules.v4
```

---

# 三十二、Step 13：清理运行时规则

修改配置文件之后：

```bash
sudo nft -a list ruleset
```

找到旧规则的 handle：

```text
# handle 123
```

然后：

```bash
sudo nft delete rule ...
```

最后：

```bash
sudo nft list ruleset | grep '旧bridge'
```

必须确认没有残留。

---

# 三十三、Step 14：重新验证 Docker

```bash
docker network inspect <network>
```

```bash
docker inspect <container>
```

```bash
docker exec <source> \
  curl -fsS --connect-timeout 5 \
  http://<destination>:<port>/
```

最后再验证公网：

```bash
curl -I https://your-domain.example
```

这样可以确认：

```text
容器内部网络
+
Docker NAT
+
反向代理
+
公网入口
```

全部正常。

---

# 三十四、几个非常值得记住的经验

## 经验 1：Docker 网络问题不要只看 Docker

很多人看到：

```text
Docker container → container timeout
```

第一反应是：

```bash
docker restart
```

但实际上：

```text
Docker
 ↓
Linux bridge
 ↓
veth
 ↓
netfilter
 ↓
nftables
 ↓
iptables
```

任何一层都可能出问题。

---

## 经验 2：`nftables.service inactive` 不代表 nftables 没规则

尤其是：

```text
iptables v1.8.x (nf_tables)
```

情况下。

一定直接：

```bash
sudo nft list ruleset
```

---

## 经验 3：Docker bridge ID 变化非常重要

例如：

```text
旧：
br-8b52009aaf0d

新：
br-7c31020cfc24
```

如果 firewall 里还存在：

```text
br-8b52009aaf0d
```

这就是非常强的异常信号。

---

## 经验 4：`Connection timed out` 和 `Connection refused` 意义完全不同

```text
refused
```

通常优先检查：

```text
服务是否监听
```

而：

```text
timeout
```

优先检查：

```text
DROP
firewall
routing
network path
```

---

## 经验 5：配置文件和运行时规则必须同时检查

不能只修改：

```text
/etc/iptables/rules.v4
```

然后认为已经解决。

必须同时确认：

```text
/etc/iptables/rules.v4
        +
kernel nftables ruleset
```

都干净。

---

# 三十五、最终状态

最终服务器状态已经恢复到：

```text
Docker
  │
  ├── nginxproxymanager_default
  │       │
  │       ├── br-7c31020cfc24
  │       │
  │       ├── 172.18.0.2
  │       │     └── Nginx Proxy Manager
  │       │
  │       └── 172.18.0.3
  │             └── New API :3000
  │
  └── new-api_new-api-network
```

旧网络：

```text
br-8b52009aaf0d
```

已经不存在。

旧 nftables 规则：

```text
br-8b52009aaf0d
```

已经全部删除。

持久化：

```text
/etc/iptables/rules.v4
```

也已经清理。

最终：

```bash
docker exec nginxproxymanager-app-1 \
  sh -c 'curl -fsS --connect-timeout 5 \
  http://172.18.0.3:3000/ >/dev/null && echo OK'
```

得到：

```text
OK
```

公网：

```bash
curl -I https://newapi.sinmu.xyz/
```

得到：

```text
HTTP/2 200
x-new-api-version: v1.0.0-rc.24
x-served-by: newapi.sinmu.xyz
```

因此可以确认：

> **Docker 内部网络、New API、Nginx Proxy Manager、Docker NAT、Cloudflare 公网入口均已经恢复正常。**

---

# 三十六、以后可以直接复制的“快速诊断脚本”

如果下次再次遇到：

> **Docker 容器 A 无法访问容器 B，表现为 Connection timeout**

可以先执行下面这一组：

```bash
echo '=== Docker Networks ==='
docker network ls

echo
echo '=== Containers ==='
docker ps --format 'table {{.Names}}\t{{.Networks}}\t{{.Ports}}'

echo
echo '=== Bridges ==='
ip -br link | grep -E 'br-|docker'

echo
echo '=== Docker iptables backend ==='
iptables -V

echo
echo '=== iptables rules ==='
sudo iptables-save | grep -E 'DOCKER|172\.1[789]\.|br-' || true

echo
echo '=== nftables rules ==='
sudo nft -a list ruleset | grep -E 'DOCKER|172\.1[789]\.|br-' || true

echo
echo '=== Persistent iptables ==='
sudo grep -nE '172\.1[789]\.|br-' \
  /etc/iptables/rules.v4 2>/dev/null || true

echo
echo '=== netfilter-persistent ==='
systemctl is-enabled netfilter-persistent 2>/dev/null || true
systemctl is-active netfilter-persistent 2>/dev/null || true

echo
echo '=== Docker service ==='
systemctl status docker --no-pager 2>/dev/null || true
snap services docker 2>/dev/null || true
```

然后重点观察一个原则：

```text
当前 Docker bridge
        VS
iptables/nftables 中的 bridge
        VS
/etc/iptables/rules.v4 中的 bridge
```

**三者必须一致。**

如果：

```text
Docker：
br-7c31020cfc24

nft：
br-8b52009aaf0d

rules.v4：
br-8b52009aaf0d
```

那么基本就可以沿着这次事故的方向排查了。

---

# 最后总结

这次问题表面上看是：

> **Nginx Proxy Manager 无法访问 New API 的 3000 端口。**

但真正的故障链实际上是：

```text
Docker 网络发生变化
        ↓
旧 bridge 被删除
        ↓
旧 iptables-persistent 配置没有清理
        ↓
iptables 使用 nf_tables backend
        ↓
旧规则进入 nftables
        ↓
raw/PREROUTING 对 172.18.0.3 执行 DROP
        ↓
Docker 当前 bridge 已经变成 br-7c31020cfc24
        ↓
容器 TCP SYN 被丢弃
        ↓
curl Connection timed out
```

最终解决方案也不是简单的：

```bash
docker restart
```

而是完成了三个层面的清理：

```text
① 清理持久化配置
/etc/iptables/rules.v4

② 清理当前 kernel 中的 nftables 运行时规则
nft delete rule ...

③ 确认 Docker 当前网络重新建立
br-7c31020cfc24
```

这次最值得记住的一句话就是：

> **Docker 网络重建以后，如果出现容器间 TCP timeout，除了检查 Docker 自己的规则，一定要检查 nftables/iptables 里有没有残留的旧 `br-*`、旧容器 IP，以及 `raw/PREROUTING` 中的 DROP 规则。尤其是使用 `iptables-nft` + `iptables-persistent` 的机器。**

这套排障思路基本可以原样复用于下一次类似事故。
