# MacBook 搭建美国 VPN 教程

## 1. 创建美国 VPS

### 1.1 注册账号

进入 [Vultr 官方网站](https://www.vultr.com/)，注册账号并登录控制台。VPS（Virtual Private Server，虚拟专用服务器）是一台长期运行在云数据中心的远程 Linux 电脑。我们购买的是美国网络出口和服务器计算资源（数据中心），而不是美国居民家庭宽带。

### 1.2 充值账户

在控制台选择 **Dashboard**，找到**Current Charges**并点击：

![Dashboard页面](/Users/ann/Desktop/截屏2026-11-15 21.39.11.png)

支付方式任选自己适合的，充值金额最低为10美元（够用2个月）：

![image-20261115214355846](/Users/ann/Library/Application Support/typora-user-images/充值页面.png)

充值成功后回到控制台，可以看到账户余额：

![image-20261115214558475](/Users/ann/Library/Application Support/typora-user-images/账户余额.png)

### 1.3 创建实例

选择左侧 **Compute → Instances**，点击 **Create Instance**：

![](/Users/ann/Desktop/创建界面.png)

选择 **Shared GPU**：

![配置界面](/Users/ann/Library/Application Support/typora-user-images/配置界面.png)

推荐配置如下：

| 配置     | 推荐选项                        |
| -------- | ------------------------------- |
| 产品     | Cloud Compute                   |
| 地区     | Los Angeles（美国洛杉矶）       |
| 操作系统 | Ubuntu 24.04 LTS                |
| CPU      | 1 vCPU                          |
| 内存     | 1 GB                            |
| 硬盘     | 约 25 GB SSD                    |
| 网络     | 公网 IPv4                       |
| 额外备份 | 暂不购买（按自身恢复需求决定）  |
| 预算     | 约 5～6 美元/月，按实时价格核对 |

进入以下界面：

![](images/image(20261105-122805).png)

打开 Mac 终端，执行命令：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vultr_wireguard -C "vultr-wireguard"
```

系统会询问是否设置密钥密码，建议设置一个自己能记住的密码。随后执行：

```bash
cat ~/.ssh/vultr_wireguard.pub
```

终端会输出一行以 `ssh-ed25519` 开头的文本。这是公钥，可以安全地添加到 Vultr。

回到网页，点击 SSH Keys 下拉框，寻找添加新密钥的选项。如果下拉框没有添加入口，可以到 Vultr 的 SSH Keys 管理页面添加，再返回部署页面选择。

**注意**：`~/.ssh/vultr_wireguard` 是私钥，绝对不要上传或发给任何人。

完成后，向下检查 Additional Features，确保没有开启收费的 Automatic Backups。然后就可以点击右侧蓝色 Deploy 创建服务器。创建完界面如下：

![image-20261115220629816](/Users/ann/Library/Application Support/typora-user-images/购买完成.png)

点击蓝色的 Cloud Instance，进入服务器详情页，找到 Start（启动） 或 Power On（开机） 按钮。

启动后等待约 1～3 分钟，直到状态显示 Running（运行中）。

随后打开 Mac 终端，输入：

```
ssh -i ~/.ssh/vultr_wireguard root@45.76.175.6 →【你所购买的ip地址】
```

第一次连接会出现：

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入 `yes`

如果之前给私钥设置了密码，终端还会要求输入该密码。回到 Vultr 控制台，服务器详情页的左下方有以下信息：

```
Username: root
Password: •••••••
```

![Vultr 套餐截图七](images/image(20261105-123234).png)

将 Vultr 提供的服务器密码复制下来，回到终端粘贴从 Vultr 复制的 root 密码，然后按回车。

登录成功后，提示符一般类似 `root@...:~#`。从此到第 3 节生成配置之前，命令都在**服务器终端**执行。

本次实例为 `45.76.175.6`。

**注意：Vultr 属于数据中心网络，不是美国住宅 IP，也不能保证某个特定网站的账号资格或访问结果。**

## 2. 安装 WireGuard

> WireGuard 是一种使用公钥密码学建立加密隧道的 VPN 协议。简单来说，Mac 与 VPS 各持有自己的密钥；Mac 将网络数据加密后发送给 VPS，VPS 再负责转发到互联网。

### 2.1 安装程序并生成密钥

在**服务器终端**执行：

```bash
apt update
apt install -y wireguard iptables
wg --version
```

正常情况下，最后会显示 WireGuard 工具版本。接下来生成服务器和 Mac 客户端各自的公私钥：

```bash
install -d -m 700 /etc/wireguard
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee mac_private.key | wg pubkey > mac_public.key
```

命令执行完毕后会创建四个文件。带有 `_private.key` 的是**私钥**，不能截图、发邮件或提交到仓库；带有 `_public.key` 的是对应公钥。这里为了让初学者便于操作，在服务器端生成 Mac 的密钥，稍后通过加密的 SCP 连接下载 Mac 配置。下载、验证后应妥善备份客户端配置。

### 2.2 开启 IPv4 转发

Linux 服务器默认不一定允许把客户端流量转发到互联网，因此需要开启 IPv4 forwarding：

```bash
printf 'net.ipv4.ip_forward = 1\n' > /etc/sysctl.d/99-wireguard.conf
sysctl -p /etc/sysctl.d/99-wireguard.conf
```

如果输出 `net.ipv4.ip_forward = 1`，表示转发设置已生效。

### 2.3 创建服务器配置

首先确认服务器通向互联网的网卡名称：

```bash
ip -4 route show default
```

输出通常包含 `dev enp1s0`、`dev eth0` 等内容。**每台 VPS 的网卡名称可能不同**，所以下面的命令会自动读取默认路由使用的网卡。

接下来在**服务器终端**完整执行这一段，创建 `/etc/wireguard/wg0.conf`：

```bash
cd /etc/wireguard
WAN_IF=$(ip -4 route show default | awk 'NR==1 {print $5}')
echo "当前公网出口网卡：$WAN_IF"

SERVER_PRIVATE=$(cat server_private.key)
MAC_PUBLIC=$(cat mac_public.key)

cat > /etc/wireguard/wg0.conf <<EOF_WG
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = $SERVER_PRIVATE
PostUp = iptables -I FORWARD -i %i -j ACCEPT; iptables -I FORWARD -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.66.66.0/24 -o $WAN_IF -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o $WAN_IF -j MASQUERADE

[Peer]
PublicKey = $MAC_PUBLIC
AllowedIPs = 10.66.66.2/32
EOF_WG

chmod 600 /etc/wireguard/wg0.conf
```

**执行前应确认 `WAN_IF` 显示了正常网卡名称**（例如 `enp1s0`），而不是空值。上述操作适合**首次创建的新服务器**；如果已经安装过 WireGuard，不要直接覆盖原有 `wg0.conf`。

这里 `wg0` 是 VPN 的虚拟网卡；`10.66.66.1` 是服务器在 VPN 内的地址；`10.66.66.2` 是稍后 Mac 使用的内部地址；`51820` 是 WireGuard 的 UDP 端口。`MASQUERADE` 属于 NAT（网络地址转换），使外部网站看到 VPS 的公网 IPv4，而不是 VPN 内部地址。

### 2.4 放行 WireGuard 端口并启动服务

如果 Ubuntu 使用 UFW 防火墙，需要放行 SSH 和 WireGuard 端口。先在**服务器终端**执行：

```bash
ufw allow 22/tcp
ufw allow 51820/udp
ufw status verbose
```

如果显示 `Status: active`，检查规则中是否允许 `51820/udp`。如果显示 `inactive`，说明 UFW 当前没有启用；上面的 `allow` 命令只保存规则，并不会自动启用防火墙。**不建议在不清楚 SSH 放行情况时贸然修改防火墙默认策略**。如果 Vultr 还配置了云防火墙（Firewall Group），也要在那里允许 UDP 51820。

确认后执行：

```bash
systemctl enable --now wg-quick@wg0
systemctl status wg-quick@wg0
wg show
```

按 `q` 可以退出 `systemctl status` 的查看界面。服务显示 `active (exited)` 对 `wg-quick@wg0` 可能是正常的，因为它负责建立网卡后便结束启动命令。此时 `wg show` 应当列出 `wg0` 和 Mac 对应的 peer，但**还不一定有握手记录**，因为 Mac 尚未连接。

## 3. 在 Mac 官方 WireGuard 客户端连接

### 3.1 在服务器上生成 Mac 配置文件

第二节已经生成了 Mac 专用密钥。现在把 Mac 的私钥、服务器公钥、服务器 IP 等信息组合成一个可以导入 WireGuard 应用的 `.conf` 文件。

仍在**服务器终端**执行：

```bash
SERVER_PUBLIC=$(cat /etc/wireguard/server_public.key)
MAC_PRIVATE=$(cat /etc/wireguard/mac_private.key)
```

然后把下面代码中 **`这里填写你的Vultr公网IPv4`** 替换成自己的服务器公网 IPv4，执行整个代码块：

```bash
SERVER_IP="这里填写你的Vultr公网IPv4"

cat > /root/vultr-mac.conf <<EOF_WG
[Interface]
PrivateKey = $MAC_PRIVATE
Address = 10.66.66.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = $SERVER_PUBLIC
Endpoint = $SERVER_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF_WG

chmod 600 /root/vultr-mac.conf
ls -l /root/vultr-mac.conf
```

如果最后出现 `-rw-------` 开头的文件记录，说明配置文件已创建。

这里 `Address = 10.66.66.2/32` 与服务器端 `[Peer]` 中的 `AllowedIPs = 10.66.66.2/32` 必须对应。`AllowedIPs = 0.0.0.0/0` 表示 Mac 的 IPv4 默认流量经过 VPN；`DNS = 1.1.1.1` 指定客户端使用的 DNS 地址；`PersistentKeepalive = 25` 用来帮助维持 NAT 映射。

### 3.2 将配置文件下载到 Mac

**新打开一个 Mac 本地终端窗口**（提示符通常是 `用户名@MacBook ~ %`，不是服务器上的 `root@...#`）。先设置自己的 VPS 公网 IPv4，再通过 SCP 下载：

```bash
VPS_IP="这里填写你的Vultr公网IPv4"
scp -i ~/.ssh/vultr_wireguard root@"$VPS_IP":/root/vultr-mac.conf ~/Downloads/
```

SCP（Secure Copy）会通过 SSH 加密连接传输文件。下载成功后，在 Mac 的「下载」文件夹中应能找到 `vultr-mac.conf`。如果要求输入 SSH 密钥的 passphrase，按之前设置的内容输入即可。

> 注意：`.conf` 包含私钥。**不要上传到 GitHub、聊天工具、在线 YAML 转换网站或不受信任的云盘**。

### 3.3 安装并导入 WireGuard

前往 App Store 安装官方 WireGuard 应用。

安装完成后打开 WireGuard，选择 **Import tunnel(s) from file（从文件导入隧道）**，选中下载的 `vultr-mac.conf`。应用可能要求你批准 macOS 的 VPN 配置权限，按系统提示操作。

导入后会出现名为 `vultr-mac` 的隧道。选中它，点击 **Activate（激活）**；需要断开时点击 **Deactivate（停用）**。不需要重新登录服务器，也不需要让终端保持打开。![image-20261115225443270](/Users/ann/Library/Application Support/typora-user-images/激活.png)

### 3.4 验证连接是否成功

先确认 WireGuard 中显示 **Active**。然后在**Mac 本地终端**执行：

```bash
curl -4 --noproxy '*' --max-time 15 https://api.ipify.org
```

正常情况下，终端应该显示你在 Vultr 购买的服务器公网 IPv4。这里的 `-4` 表示只测试 IPv4，`--noproxy '*'` 表示不使用 curl 的 HTTP 代理设置，但它**不会绕过 WireGuard 这样的系统级 VPN**。

也可以使用 Safari 打开 `https://ipinfo.io/ip`，查看网页显示的公网 IP 是否与 Vultr 一致。

![image-20261115225554867](/Users/ann/Library/Application Support/typora-user-images/验证连接.png)

如果显示 Mac 原来的公网 IP，或者命令超时，请先检查 WireGuard 是否已激活，再按照下一节排查。Mac 本地终端偶尔会在 IP 后面显示 `%`，它通常只是 zsh 的命令提示显示效果，不属于 IP 地址。

### 3.5 在服务器检查握手记录

如果要进一步确认连接状况，可以在 Mac 的终端通过 SSH 登录服务器：

```bash
ssh -i ~/.ssh/vultr_wireguard root@"$VPS_IP"
```

其中 `$VPS_IP` 必须是本地终端中已经设置的服务器地址；若你新开了终端窗口，需要重新设置一次。登录后执行：

```bash
wg show
```

如果 Mac 已成功连接，服务器会显示该客户端的 `latest handshake`（最近握手时间）以及 `transfer`（收发数据量）。如果没有握手，通常要优先检查 UDP 51820 是否已放行、服务器是否运行、客户端密钥与 Endpoint 是否正确。

### 3.6 日常如何开启和关闭

平时只需使用 WireGuard 应用：**Activate 连接、Deactivate 断开**。不需要打开终端，也不需要一直打开 Vultr 网页。关闭 Mac 或 SSH 终端不会关闭 VPS，服务器会独立运行。为了方便以后重新安装 Mac 系统，建议在你控制的安全位置保留一份**加密备份**，不要以明文配置公开分享。

确认 Mac 已经成功导入并安全备份后，可以**选择性地**在服务器上删除临时的客户端配置及 Mac 私钥副本，以减少服务器上存放的敏感文件：

```bash
rm -f /root/vultr-mac.conf /etc/wireguard/mac_private.key
```

这是可选清理操作，**不会撤销已经连接的客户端授权**；但删除后不能再从服务器下载原配置，所以务必先完成备份。服务器的 `server_private.key`、`server_public.key`、`wg0.conf` 以及 Mac 对应的公钥授权都要保留。

## 4. 常见问题与处理办法

### 4.1 WireGuard 显示已激活，但无法上网

先在服务器检查服务是否运行：

```bash
systemctl status wg-quick@wg0
wg show
```

如果没有 `latest handshake`，重点核对 Vultr 公网 IPv4、UDP 51820、客户端和服务器的公私钥配对，以及 UFW / Vultr 云防火墙规则。如果**有握手、却无法访问网页**，检查 IPv4 转发是否开启：

```bash
sysctl net.ipv4.ip_forward
```

结果应当是 `net.ipv4.ip_forward = 1`。同时检查 `wg0.conf` 中的 NAT 规则及公网出口网卡名称是否正确。如果 IP 查询能够成功但网站域名打不开，还需要检查 DNS。

### 4.2 SSH 显示 `Permission denied (publickey)`

先确认自己使用的是创建服务器时对应的 SSH **私钥**文件，并检查 Vultr 创建实例时是否选中了配对的公钥：

```bash
ssh -i ~/.ssh/vultr_wireguard root@"$VPS_IP"
```

如果公钥未安装，可通过 Vultr Web Console 登录服务器修复。不要把私钥上传到任何在线 SSH 排错网站。

### 4.3 终端一直尝试连接旧代理 `127.0.0.1`

如果 Mac 曾经使用过其他代理程序，终端可能残留 `http_proxy`、`https_proxy` 等环境变量，导致 `curl` 报错。可以先在 **Mac 本地终端**执行：

```bash
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
```

这只清除当前终端会话中的相关环境变量，不会删除 WireGuard 配置。如果其他应用仍无法连接，还需要检查 macOS 的网络代理设置是否残留旧地址。

### 4.4 速度慢、延迟高

美国 VPS 并不意味着网络一定快。跨境距离、学校或运营商线路、服务器地域以及不同时段的拥塞都可能影响速度。建议在**同一网络、同一测速网站**下比较 WireGuard 开启和关闭时的结果。不要只根据 IP 查询网站给出的“纯净度评分”判断速度，也不要把数据中心 IP 误认为住宅 IP。

### 4.5 IPv6 和 DNS 会不会泄漏？

这篇教程使用的是 `AllowedIPs = 0.0.0.0/0`，**只明确接管 IPv4**，并未为 IPv6 配置 VPN 出口。即使 IPv4 查询显示美国，设备的 IPv6、DNS 或某些应用仍可能使用本地网络。建议分别在开启与关闭 WireGuard 时访问 [ipleak.net](https://ipleak.net/) 和 [BrowserLeaks](https://browserleaks.com/ip) 对比 IPv4、IPv6、DNS 结果。

如果发现非预期的 IPv6 或 DNS 请求，不应直接声称“全局无泄漏”；需要根据系统网络环境另行处理。本教程不提供已经完整解决 IPv6 泄漏的保证。

## 5. 安全、费用与补充说明

**私钥必须保密。** `server_private.key`、`mac_private.key`、`vultr-mac.conf` 都可能包含敏感凭据，不应该上传到 GitHub，尤其不要使用 `git add .` 时意外把这些文件加入仓库。公钥虽然不必保密，但公开教程也没有必要写出自己正在使用的真实密钥。

**VPS 会独立计费。** Mac 断开 VPN 或关闭终端，都不会自动停止服务器计费。即使在 Vultr 控制台关机，也不一定停止实例费用；不再使用时应了解 Vultr 对实例销毁、存储和其他资源的计费方式。创建前后请以控制台显示的实际价格为准。

**VPS 不等于住宅 IP。** Vultr 提供的是数据中心网络地址。它能够作为美国 IPv4 出口，但不会自动让网站认定使用者是美国居民，也不能保证某个网站的注册资格、访问权限或 IP 信誉。

**参考资料**：[WireGuard 官方安装说明](https://www.wireguard.com/install/) · [Ubuntu：WireGuard 作为默认网关](https://ubuntu.com/server/docs/wireguard-vpn-defaultgw/) · [Ubuntu：WireGuard 故障排查](https://ubuntu.com/server/docs/wireguard-vpn-troubleshooting/)
