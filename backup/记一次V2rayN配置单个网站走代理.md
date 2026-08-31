环境：windows10 + V2ranyN V7.14.12


假设我们的目标网站是youtube.com，只有访问youtube.com的时候才走代理，其他的都不走代理

这个时候我们需要新增一个规则集，然后在规则集中添加两个规则
- 所有请求都走直连（不走代理）
- youtube.com 走代理，且优先级最高。



## 设置-路由设置-添加规则集

别名：OnlyYoutube（随便写）
域名解析策略：IPIfNonMatch / IPOnDemand

### 添加规则1
- 别名：DirectB(随便写)
- outboundTag：direct
- IP或 IP CIDR：0.0.0.0/0,::/0

效果如图
<img width="450" height="300" alt="Image" src="https://github.com/user-attachments/assets/6352dd06-502a-43aa-8db1-b8e7ee3debdc" />

### 添加规则2
- 别名：ProxyA(随便写)
- outboundTag：Proxy
- Domain：domain:youtube.com,domain:github.com


效果如图

<img width="450" height="300" alt="Image" src="https://github.com/user-attachments/assets/86b1b743-a224-4b46-957b-1b99077eea57" />
