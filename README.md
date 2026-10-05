# 解决easyconnect劫持所有流量导致无法连接服务器的同时使用claude/codex的情况

### 原理

原生 EasyConnect 一登录，就会接管整台 Mac 的流量，Claude、翻墙代理这些都会受影响。这里的做法是把 EasyConnect 放进 Docker 容器里运行，它接管的就只是容器自己的流量。容器对外开一个 SOCKS5 代理端口（1080），只有需要进学校内网的连接才走这个端口：

ssh ──> nc ──> Mac:1080 ──> 容器里的 SOCKS5 ──> EasyConnect(tun0) ──> 学校 VPN ──> 内网服务器
其他流量（Claude、网页等）──> 直连 / 代理软件，不经过 EasyConnect

### 步骤

##### 1. 关掉原生 EasyConnect

原生版和容器版不能同时登录，会互相把对方顶下线，而且原生版又会接管全部流量。先把原生版关掉：

```
pkill -f EasyConnect; pkill -f ECAgent; pkill -f EasyMonitor
```
##### 2. 安装 OrbStack（Docker）

```
brew install orbstack
open -a OrbStack
```

不用登录 OrbStack 账号，个人使用免费。

如果你开着 v2rayN、Clash 这类代理软件，要关掉 OrbStack 的"跟随系统代理"，否则容器连不上 VPN：
```
orb config set network_proxy none
orbctl stop && orbctl start
```
##### 3. 启动容器
```
docker run -d --name ec --restart unless-stopped \
  --device /dev/net/tun --cap-add NET_ADMIN \
  --dns 114.114.114.114 --dns 223.5.5.5 \
  -p 127.0.0.1:1080:1080 -p 127.0.0.1:5901:5901 \
  -e PASSWORD=自己设一个VNC密码 -e URLWIN=1 \
  hagb/docker-easyconnect:7.6.7
```
- 用的是带图形界面的版本。
- M 系列 Mac 会提示 platform (linux/amd64) does not match，忽略即可，可以正常运行。
- 端口只绑定在 127.0.0.1 上，局域网里的其他人连不进来。

##### 4. 登录 VPN

在 Finder 里按 Cmd+K，输入 vnc://127.0.0.1:5901，然后输入第三步设置的 VNC 密码。在打开的 EasyConnect 窗口里填你的地址 ，像平时一样登录。

登录成功后，关掉 VNC 窗口就行，容器会在后台继续运行。想确认是否登录成功，可以运行：
```
docker exec ec ip addr show tun0   # 能看到 10.x.x.x 的地址就是登上了
```
<img width="1806" height="988" alt="image" src="https://github.com/user-attachments/assets/015124f4-d843-47e2-b62f-73ee20b46f33" />

##### 5. 配置 ssh

在 ~/.ssh/config 里，给要连的内网服务器加一行 ProxyCommand，如：
```
Host 服务器别名
    HostName 服务器内网IP
    User 用户名
    Port 22
    ProxyCommand nc -X 5 -x 127.0.0.1:1080 %h %p
```
然后：
```
ssh 服务器别名
```
VS Code 的 Remote-SSH 用的也是这份配置，同样能直接连。

### 常见的问题和解法

1. Path selection failed, possibly because network connection error occurs（第一次）
原因是容器里的 DNS 坏了，解析不了路径。解决办法是在启动命令里手动指定 DNS，如：--dns 114.114.114.114 --dns 223.5.5.5。

2. Network connection error occurred
OrbStack 默认会把容器流量转给系统代理（v2rayN），流量走不通。运行 orb config set network_proxy none，再重启 OrbStack 就好了。

3. Mac 上直连的网站全都打不开，只有走代理的网站能用
Wi-Fi 通过 DHCP 自动分到的 DNS（192.168.1.1）根本不通，但路由器的实际地址是 192.168.0.1。手动改一下 DNS：
```
networksetup -setdnsservers Wi-Fi 223.5.5.5 119.29.29.29
```
4. The client version and server software version is not matching
服务器下发了强制更新的要求，客户端会把它缓存 12 小时，所以第一次能登上，后面就被拦了。用第 4 步的命令改版本号，然后重启容器。
