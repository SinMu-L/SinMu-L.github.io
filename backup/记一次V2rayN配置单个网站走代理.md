
# V2rayN 路由配置：只让 YouTube 走代理，其他全部直连

> 环境：Windows 10 + V2rayN V7.14.12  
> 目标：访问 YouTube 时走代理，访问其他网站时全部直连。

## 一、核心原则：规则顺序就是优先级

V2ray / Xray 的路由规则是**从上到下逐条匹配，命中第一条就停止**。

所以在这个场景里：

- 想让 YouTube 走代理，就必须把 YouTube 规则放在最上面；
- 想让其他请求直连，就把 `0.0.0.0/0,::/0` 兜底规则放在最下面；
- 不是看规则名字叫不叫“优先级最高”，而是看它在列表里的位置。

一句话：**谁在上面，谁优先；命中即停。**

---

## 二、容易写错的顺序

很多人会这样加：

1. `DirectB` → `direct` → IP：`0.0.0.0/0,::/0`
2. `ProxyA` → `proxy` → Domain：`youtube.com`

这样写是错的。

因为所有请求都会先命中 `0.0.0.0/0,::/0`，直接走直连，后面的 YouTube 规则永远不会生效。

正确顺序应该是：

1. `ProxyA` → `proxy` → YouTube 域名
2. `DirectB` → `direct` → `0.0.0.0/0,::/0`

---

## 三、正确配置步骤

### 1. 添加规则集

进入：

```text
设置 -> 路由设置 -> 添加规则集
```

填写：

- 别名：`OnlyYoutube`（随便写）
- 域名解析策略：`IPIfNonMatch`（推荐）或 `IPOnDemand`

### 2. 添加规则 1：YouTube 走代理

- 别名：`ProxyA`
- outboundTag：`proxy`
- Domain：

```text
geosite:youtube
```

如果 geosite 不可用，也可以手动写：

```text
domain:youtube.com,domain:youtu.be,domain:ytimg.com,domain:googlevideo.com,domain:youtube-nocookie.com,domain:ggpht.com
```

作用：命中 YouTube 相关域名后，走代理。

![ProxyA 规则内容示意](https://github.com/user-attachments/assets/86b1b743-a224-4b46-957b-1b99077eea57)

### 3. 添加规则 2：其他全部直连

- 别名：`DirectB`
- outboundTag：`direct`
- IP：

```text
0.0.0.0/0,::/0
```

作用：前面没有命中的请求，全部走直连兜底。

![DirectB 规则内容示意](https://github.com/user-attachments/assets/6352dd06-502a-43aa-8db1-b8e7ee3debdc)

> 注意：上面两张图只是字段内容示意。实际规则顺序必须是 `ProxyA` 在上，`DirectB` 在下。

最终规则顺序：

```text
1. ProxyA  -> proxy  -> geosite:youtube
2. DirectB -> direct -> 0.0.0.0/0,::/0
```

---

## 四、如果还想让 GitHub 也走代理

把 `ProxyA` 的 Domain 改成：

```text
geosite:youtube,geosite:github
```

或者手动添加：

```text
domain:github.com,domain:githubusercontent.com,domain:githubassets.com
```

这样 YouTube 和 GitHub 都会走代理，其他仍然直连。

---

## 五、为什么推荐用 `geosite:youtube`

YouTube 不只有 `youtube.com`，还会用到很多相关域名，例如：

- `googlevideo.com`
- `ytimg.com`
- `youtu.be`
- `youtube-nocookie.com`
- `ggpht.com`

如果只写 `domain:youtube.com`，很容易出现视频加载慢、图片不显示、播放中断等问题。

所以只要你的 V2rayN 带有 geosite 数据，优先用：

```text
geosite:youtube
```

---

## 六、域名解析策略怎么选

规则集里的“域名解析策略”建议选：

```text
IPIfNonMatch
```

含义是：

- 先拿域名去匹配规则；
- 如果命中 `geosite:youtube`，直接走代理；
- 如果没命中，再解析 IP，然后被 `0.0.0.0/0,::/0` 兜底直连。

`IPOnDemand` 也能用，但 `IPIfNonMatch` 更直观，适合这种“白名单代理”场景。

---

## 七、检查清单

配置完后，建议逐项检查：

1. 规则顺序：`ProxyA` 在上，`DirectB` 在下。
2. `outboundTag` 是否和实际出站标签一致，通常是小写 `proxy`、`direct`。
3. 规则集是否已经启用。
4. V2rayN 主界面的路由模式是否选对。
5. 如果有多个规则集，代理规则集也要排在直连规则集前面。
6. 改完不生效时，重启 V2rayN，或清理系统 DNS 缓存后再试。

---

## 八、总结

这个需求本质上是一个“白名单代理”：

- 只有指定域名走代理；
- 其他所有请求直连；
- 代理规则必须放在最上面；
- 直连兜底必须放在最下面。

记住这句话就够了：

> **YouTube 走代理的规则放最上面，`0.0.0.0/0,::/0` 直连兜底放最下面。V2rayN 路由规则从上到下匹配，命中即停。**


