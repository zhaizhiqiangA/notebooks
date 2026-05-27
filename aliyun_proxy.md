架构 1：针对本地 WSL2 / 浏览器的方案 (使用了阿里云 Gost)
网络路径： 本地 WSL2 -> (本地 Gost 客户端加密) -> 阿里云 Gost 服务端 (1080端口解密) -> 自由互联网。

特点： 阿里云上必须运行 Gost 服务端。

架构 2：针对 mindspore 训练节点的方案 (未使用阿里云 Gost)
网络路径： mindspore 节点 -> (纯 SSH 加密，走 443 端口) -> 阿里云 SSH 服务 (443端口) -> 自由互联网。

特点： 我们使用的是 SSH 自带的动态端口转发功能（-D 1080）。

核心原理： 当你在 mindspore 上执行 ssh -p 443 -D 1080 root@47.252.13.130 时，SSH 协议就已经在本地建立了一个 SOCKS5 代理。所有的流量在到达阿里云后，是由阿里云自带的 SSH 守护进程（sshd） 负责直接解密并转发到外网的，整个流量转发过程完全没有经过阿里云上的 Gost 进程。

上一份操作说明书是专门为了解决 “mindspore 训练节点如何突围” 这个特定问题而汇总的。为了遵循工程上的“最小依赖原则”（能用系统自带工具解决，就绝不引入额外的第三方工具），说明书去掉了对阿里云端 Gost 的依赖。这样可以省去 mindspore 节点到阿里云之间的双重加密损耗，让大模型数据集的拉取速度达到最快。

📚 全栈式操作说明书（包含云端 Gost 服务端搭建）
如果你希望这份记录更加完整，既能解决 mindspore 机器的特殊突围，又能让本地 WSL2 或浏览器通过 Gost 客户端直接连接云端，以下是完整的全功能版操作说明书：

阶段一：阿里云服务器端配置 (基础环境与双重服务)
1. 修改 SSH 配置文件，增加 443 端口监听
用于欺骗严苛的企业/实验室网关，将 SSH 流量伪装成正常的 HTTPS 流量。

Bash
vi /etc/ssh/sshd_config
在文件中添加或修改：

Ini, TOML
Port 22
Port 443
(保存退出：按 Esc，输入 :wq 回车)

2. 破除 Ubuntu 端口接管陷阱并重启 SSH
Bash
# 强制关闭 socket 接管，交回给 service 传统模式
systemctl disable --now ssh.socket
systemctl enable --now ssh.service
systemctl restart ssh
3. 在云服务器上部署 Gost 服务端 (供本地 WSL2/浏览器 使用)
Bash
# 下载、解压并移动二进制文件
wget https://github.com/ginuerzh/gost/releases/download/v2.11.5/gost-linux-amd64-2.11.5.gz
gzip -d gost-linux-amd64-2.11.5.gz
chmod +x gost-linux-amd64-2.11.5
mv gost-linux-amd64-2.11.5 /usr/local/bin/gost

# 编写 systemd 守护进程配置文件
vi /etc/systemd/system/gost.service
粘贴以下内容（注意：替换为你专属的 32 位强密码）：

Ini, TOML
[Unit]
Description=Gost Proxy Service
After=network.target

[Service]
Type=simple
# 监听 1080 端口，使用 Shadowsocks 加密协议
ExecStart=/usr/local/bin/gost -L ss://chacha20-ietf-poly1305:your_strong_password@:1080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
启动云端 Gost：

Bash
systemctl daemon-reload && systemctl enable --now gost
4. 阿里云安全组防火墙放行
在阿里云轻量应用服务器控制台，入方向精准放行两个端口：

TCP 443 (供受限的 mindspore 节点突围使用)

TCP 1080 (供本地 WSL2/浏览器 直接连接加密代理使用)

安全红线：禁止开放 1/65535 全端口。

阶段二：客户端接入（根据环境二选一）
接入途径 A：在受限的 Linux 训练节点（如 mindspore 机器）接入
由于该机器无法直连 1080 端口，必须借道 443 端口：

Bash
# 1. 建立底层 SSH 隧道，在本地生成 1080 SOCKS5 端口
ssh -p 443 -f -N -D 1080 root@47.252.13.130

# 2. 本地单机运行 Gost，将 SOCKS5 转换为 HTTP 兼容端口（避免 8118 端口冲突，使用 18118）
nohup ./gost -L http://:18118 -F socks5://127.0.0.1:1080 > /dev/null 2>&1 &
接入途径 B：在本地个人电脑（WSL2 / 浏览器）直连接入
本地环境没有 22 或 1080 端口的封锁，直接与云端 Gost 握手：

Bash
# 在本地 WSL2 中启动 Gost 客户端，将本地流量加密后送往云端 1080 端口
nohup gost -L auto://:1080 -F ss://chacha20-ietf-poly1305:your_strong_password@47.252.13.130:1080 > /dev/null 2>&1 &
如果是浏览器，可直接在 SwitchyOmega 插件中配置 SOCKS5 代理，填入云端 IP 和 1080 端口，并在锁头图标里输入 Shadowsocks 的密码。

阶段三：应用层流量接管 (通用)
在需要上网的本地终端（无论是 WSL2 还是 mindspore）中，声明环境变量：

Bash
# 若为途径 A (训练节点):
export http_proxy="http://127.0.0.1:18118"
export https_proxy="http://127.0.0.1:18118"

# 若为途径 B (本地 WSL2):
export http_proxy="http://127.0.0.1:1080"
export https_proxy="http://127.0.0.1:1080"

# 保护内网流量不外流
export no_proxy="localhost,127.0.0.1,10.*,192.168.*,172.*"
终极验证： curl -I https://www.google.com 返回 200 OK 即告全线打通。