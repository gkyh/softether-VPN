# 在 Ubuntu 20.04 安装与配置 SoftEther VPN Server

> 来源：<https://cloudinfrastructureservices.co.uk/how-to-install-softether-vpn-server-on-ubuntu-20-04/>  
> 本文按原网页步骤整理。页面使用的 SoftEther v4.38（2021）下载链接已较旧；实际部署时应从官方页面选择当前稳定版，并相应替换文件名与下载地址。

## 前提条件

- 两台 Ubuntu 20.04 服务器：一台 VPN 服务端、一台 VPN 客户端。
- 两台机器均可使用 root 权限（或将下列命令改为 `sudo`）。

## 一、安装服务端

### 1. 安装依赖

```bash
apt-get update -y
apt-get install build-essential gnupg2 gcc make -y
```

### 2. 下载、解压与编译

原文使用的版本如下：

```bash
wget http://www.softether-download.com/files/softether/v4.38-9760-rtm-2021.08.17-tree/Linux/SoftEther_VPN_Server/64bit_-_Intel_x64_or_AMD64/softether-vpnserver-v4.38-9760-rtm-2021.08.17-linux-x64-64bit.tar.gz
tar -xvzf softether-vpnserver-v4.38-9760-rtm-2021.08.17-linux-x64-64bit.tar.gz
cd vpnserver
make
```

编译过程中按提示接受许可协议。完成后，网页管理界面可通过 `https://服务器IP:5555/` 访问。

### 3. 安装到 `/usr/local` 并设置权限

```bash
cd ..
mv vpnserver /usr/local/
cd /usr/local/vpnserver/
chmod 600 *
chmod 700 vpnserver
chmod 700 vpncmd
```

### 4. 创建启动脚本并启用开机启动

创建 `/etc/init.d/vpnserver`：

```sh
#!/bin/sh
# chkconfig: 2345 99 01
# description: SoftEther VPN Server
DAEMON=/usr/local/vpnserver/vpnserver
LOCK=/var/lock/subsys/vpnserver
test -x $DAEMON || exit 0
case "$1" in
start)
  $DAEMON start
  touch $LOCK
  ;;
stop)
  $DAEMON stop
  rm $LOCK
  ;;
restart)
  $DAEMON stop
  sleep 3
  $DAEMON start
  ;;
*)
  echo "Usage: $0 {start|stop|restart}"
  exit 1
esac
exit 0
```

设置权限、启动并注册：

```bash
mkdir /var/lock/subsys
chmod 755 /etc/init.d/vpnserver
/etc/init.d/vpnserver start
update-rc.d vpnserver defaults
```

## 二、配置服务端

启动命令行管理工具：

```bash
cd /usr/local/vpnserver
./vpncmd
```

在交互界面中：

1. 输入 `1`，选择 **Management of VPN Server or VPN Bridge**。
2. 主机地址直接按 Enter（连接本机）；Virtual Hub 名称也按 Enter（服务端管理员模式）。
3. 执行以下命令，并按提示输入密码或参数：

```text
ServerPasswordSet
HubCreate myhub
Hub myhub
SecureNatEnable
UserCreate vpnuser
UserPasswordSet vpnuser
IPsecEnable
exit
```

`IPsecEnable` 的原文示例参数：

```text
Enable L2TP over IPsec Server Function (yes / no): yes
Enable Raw L2TP Server Function (yes / no): yes
Enable EtherIP / L2TPv3 over IPsec Server Function (yes / no): yes
Pre Shared Key for IPsec (Recommended: 9 letters at maximum): vpnserver
Default Virtual HUB in a case of omitting the HUB on the Username: myhub
```

请把示例的预共享密钥 `vpnserver` 换成自己的强密钥。

## 三、放行 UFW 防火墙端口

若服务器启用了 UFW：

```bash
ufw allow 443/tcp
ufw allow 5555/tcp
ufw allow 992/tcp
ufw allow 1194/udp
ufw reload
```

云服务器还需在云厂商安全组/网络防火墙中放行相同端口。

## 四、安装并连接 Linux 客户端

### 1. 安装依赖并下载客户端

```bash
apt-get install build-essential gnupg2 gcc make -y
wget http://www.softether-download.com/files/softether/v4.38-9760-rtm-2021.08.17-tree/Linux/SoftEther_VPN_Client/64bit_-_Intel_x64_or_AMD64/softether-vpnclient-v4.38-9760-rtm-2021.08.17-linux-x64-64bit.tar.gz
tar -xvzf softether-vpnclient-v4.38-9760-rtm-2021.08.17-linux-x64-64bit.tar.gz
```

### 2. 下载原文所用辅助脚本

```bash
mkdir /root/vpnscript
cd /root/vpnscript
wget https://raw.githubusercontent.com/mfaizanse/intellexlab-files/main/softether-vpn-client/remove-client.sh
wget https://raw.githubusercontent.com/mfaizanse/intellexlab-files/main/softether-vpn-client/setup-client.sh
wget https://raw.githubusercontent.com/mfaizanse/intellexlab-files/main/softether-vpn-client/vpn-connect.sh
wget https://raw.githubusercontent.com/mfaizanse/intellexlab-files/main/softether-vpn-client/vpn-disconnect.sh
wget https://raw.githubusercontent.com/mfaizanse/intellexlab-files/main/softether-vpn-client/vpn_config
chmod 755 *
```

> 执行网上下载的脚本前，建议先检查其内容，并确认来源可信。

### 3. 配置客户端

编辑 `vpn_config`：

```bash
nano vpn_config
```

按实际环境填写：

```bash
CLIENT_DIR="/root/vpnclient"
NIC_NAME="nic1"
ACCOUNT_NAME="vpnuser"
VPN_HOST_IPv4="vpn-server-ip"
LOCAL_GATEWAY="gateway-ip-of-client-machine"
```

运行初始化脚本：

```bash
./setup-client.sh
```

根据提示输入服务端 `IP:端口`（原文示例为 `服务器IP:443`）、Virtual Hub 名称 `myhub`、用户 `vpnuser`、虚拟网卡名 `nic1` 和用户密码。

### 4. 连接与验证

```bash
./vpn-connect.sh
ifconfig
```

连接成功后应能看到类似 `vpn_nic1` 的网络接口；可用 `./vpn-disconnect.sh` 断开连接。

## 注意事项

- 建议使用官方当前版本，不要直接用于生产环境的旧版 v4.38 下载链接。
- 服务端管理密码、Hub 密码、VPN 用户密码及 IPsec 预共享密钥均应设为不同的强密码。
- 管理端口 5555 应尽可能限制来源 IP，避免暴露给整个互联网。
