这份操作手册为你总结了在使用**商业 VPN（全局虚拟网卡类）**时，如何优雅地为远端服务器提供外网访问权限。

由于商业 VPN 通常不开放本地 HTTP 代理端口，且存在严格的 DNS 污染拦截，传统的端口映射往往会失败（如遇 `(56) Proxy CONNECT aborted` 或 `(35) SSL routines::unexpected eof` 报错）。

通过 **SSH 原生动态反向转发 + 远端 DNS 解析 (`socks5h`)** 是最简便、无需安装任何第三方软件的终极方案。

---

### 📘 操作手册：通过本地商业 VPN 为远端服务器配置代理

**环境假设：**
* **本地电脑**：已开启商业 VPN 并能正常访问外网（如 Windows 系统）。
* **远端服务器**：需要访问外网的 Linux 服务器。
* **代理端口**：假设我们使用 `8888` 作为服务器上的代理端口。

#### 步骤一：建立 SSH 动态反向隧道 (本地电脑操作)
打开本地电脑的终端（如 Windows 的 PowerShell 或 cmd），使用以下命令连接服务器：

```powershell
# 将 2222 替换为实际的 SSH 端口，username@ip 替换为实际的登录信息
ssh -p 2222 -R 8888 username@your_server_ip
```
* **核心原理**：单独的 `-R 8888`（不加 `localhost`）会让本地 SSH 客户端在服务器的 `8888` 端口上直接启动一个 **SOCKS5** 代理服务，流量会直接由本地电脑的 VPN 网卡接管。

#### 步骤二：
使用 Privoxy 进行协议转换（最推荐，一劳永逸）
Privoxy 可以将您的 SOCKS5 代理转换为标准的 HTTP 代理。配置好之后，您可以用标准的环境变量，所有的应用（包括 curl、gemini、Python 等）都能完美运行。

1. 在 Ubuntu 上安装 Privoxy：

Bash
apt-get update
apt-get install privoxy -y
2. 配置 Privoxy 将流量转发给 SSH SOCKS5：
执行下面这行命令，将转发规则直接追加到配置文件末尾（注意最后的那个点 . 不能漏掉）：

Bash
echo "forward-socks5t / 127.0.0.1:8888 ." >> /etc/privoxy/config
3. 启 Privoxy 服务：
#直接在后台运行守护进程
privoxy /etc/privoxy/config

(Privoxy 默认会在本机的 8118 端口提供 HTTP 代理服务)

4. 设置环境变量并启动 Gemini：
现在您可以把代理指向 Privoxy 提供的 8118 端口了：

Bash
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118

# 测试一下，应该能返回正常的 HTTP 状态码了
curl -I https://generativelanguage.googleapis.com

# 启动 Gemini
gemini
