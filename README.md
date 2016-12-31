# Openconnect + chnroutes + ChinaDNS + Dnsmasq 实现家庭所有设备区分国内外IP翻墙

## 环境
国外VPS：Ubuntu 16.04。家里一台双网口的Ubuntu 16.04作为网关，下连交换机和无线AP。


## 配置国外VPS

### 安装ocserv (OpenConnect VPN Server)

#### 下载ocserv

地址: http://www.infradead.org/ocserv/download.html

ocserv-0.11.6有[bug](https://gitlab.com/ocserv/ocserv/issues/74)编译不了.

```
wget ftp://ftp.infradead.org/pub/ocserv/ocserv-0.11.5.tar.xz
tar xvf ocserv-0.11.5.tar.xz
cd ocserv-0.11.5
```


#### 安装依赖包

```
sudo apt-get install build-essential libgnutls28-dev libev-dev libwrap0-dev libpam0g-dev liblz4-dev libseccomp-dev libreadline-dev libnl-route-3-dev libkrb5-dev liboath-dev
```


#### 编译 & 安装

```
./configure
make
sudo make install
```


#### 检查

```
ocserv -v
```

能输出版本信息就OK


### 制作证书

#### 安装所需工具

```
sudo apt-get install gnutls-bin
```


#### 建个文件夹用来存放文件

```
mkdir cert
cd cert
```

生成的文件以后重装也会用到，可以保存一下，就不用重新做了。


#### 创建CA证书

生成CA密钥:

```
certtool --generate-privkey --outfile ca-key.pem
```


新建CA证书模板文件`ca.tmpl`，填写如下内容，其中`cn`和`organization`替换为自己喜欢的名字:

```
cn = "My Company CA"
organization = "My Company"
serial = 1
expiration_days = -1
ca
signing_key
cert_signing_key
crl_signing_key
```

生成CA证书:

```
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```


#### 创建服务器证书

生成服务器证书密钥:

```
certtool --generate-privkey --outfile server-key.pem
```

创建服务器证书模板文件`server.tmpl`，填写如下内容，这里的`cn`必须填写之后连接VPN时使用的域名或IP，`organization`替换为自己喜欢的名字:

```
cn = "123.123.123.123"
organization = "My Company"
expiration_days = -1
signing_key
encryption_key
tls_www_server
```

生成服务器证书:

```
certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template server.tmpl --outfile server-cert.pem
```


### 配置ocserv

创建配置文件夹:

```
sudo mkdir /etc/ocserv
```

复制server-key.pem和server-cert.pem到配置文件夹:

```
cp server-key.pem /etc/ocserv/
cp server-cert.pem /etc/ocserv/
```

回到解压的文件夹，复制一份配置文件样本:

```
sudo cp doc/sample.config /etc/ocserv/ocserv.conf
```

创建VPN账号:

```
sudo ocpasswd USER_NAME
```

`USER_NAME`替换为自己要的名字. 命令行提示输入密码并确认密码.


然后修改配置文件`/etc/ocserv/ocserv.conf`以下参数:

```
auth = "plain[passwd=/etc/ocserv/ocpasswd]"

# 服务器监听的TCP端口
tcp-port = 4433

# 关闭UDP端口
#udp-port = 443

try-mtu-discovery = true

server-cert = /etc/ocserv/server-cert.pem
server-key = /etc/ocserv/server-key.pem

dns = 8.8.8.8
dns = 8.8.4.4

# 注释掉所有的route
#route = 10.10.10.0/255.255.255.0
#route = 192.168.0.0/255.255.0.0

cisco-client-compat = true
```

### 配置IP转发

修改`/etc/sysctl.conf`:

```
net.ipv4.ip_forward = 1
```

执行此命令使它立即生效:
```
sysctl -w net.ipv4.ip_forward=1
```

修改iptables:

```
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

编辑`/etc/rc.local`, 在`exit 0`上面插入:

```
iptables-restore < /etc/iptables.rules
```


## 配置家里作为网关的Ubuntu

### 配置dnsmasq

安装dnsmasq:

```
sudo apt-get install dnsmasq
```

修改 `/etc/dnsmasq.conf`以下参数:

```
no-resolv
server=114.114.114.114
dhcp-range=192.168.0.50,192.168.0.150,12h
```


### 配置网卡参数

主机上需要用两张网卡, 一张连接外网的网卡, 和一张作为内网网关的网卡. 执行:

```
ifconfig
```

从IP地址可以判断哪张是外网网卡, 哪张是内网网卡. 外网网卡的IP地址是由上级路由器(运营商网关等)分配的, 剩下那张就是内网网卡.

假设内网网卡名字为`enp2s1`, 我们在`/etc/network/interfaces`加入它的配置:

```
auto enp2s1
iface enp2s1 inet static
address 192.168.0.1
netmask 255.255.255.0
```

重启网卡以生效:

```
sudo ifdown enp2s1
sudo ifup enp2s1
```


### 配置IP转发

修改`/etc/sysctl.conf`:

```
net.ipv4.ip_forward = 1
```

执行此命令使它立即生效:
```
sysctl -w net.ipv4.ip_forward=1
```

修改iptables:

```
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

编辑`/etc/rc.local`, 在`exit 0`上面插入:

```
iptables-restore < /etc/iptables.rules
```

### 配置ChinaDNS

在 https://github.com/shadowsocks/ChinaDNS/releases 下载最新版

```
wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz
```

解压 & 编译 & 安装:

```
tar xvf chinadns-1.3.2.tar.gz
cd chinadns-1.3.2
./configure
make
```

拷贝默认配置文件:

```
sudo mkdir /usr/local/chinadns
sudo cp src/chinadns /usr/local/chinadns/
```

获取最新的中国IP列表:

```
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > chnroutes.txt
sudo cp chnroutes.txt /usr/local/chinadns/
```

根据IP生成路由规则:

```
sudo sed -r -e 's;.+;ip route add & via 192.168.1.1;' -e '1s;^;#!/bin/sh\n;' chnroutes.txt | sudo tee /usr/local/bin/chnroutes > /dev/null
sudo chmod u+x,g+x /usr/local/bin/chnroutes
```

启动chinadns:

```
/usr/local/chinadns/chinadns -m -c /usr/local/chinadns/chnroutes.txt -p 5333 -s 114.114.114.114,8.8.8.8
```


### 配置Openconnect客户端

#### 安装vpnc-script
地址: http://www.infradead.org/openconnect/vpnc-script.html

```
wget http://git.infradead.org/users/dwmw2/vpnc-scripts.git/blob_plain/HEAD:/vpnc-script
sudo mkdir /etc/vpnc
sudo cp vpnc-script /etc/vpnc/
sudo chmod u+x,g+x /etc/vpnc/vpnc-script
```


#### 安装openconnect
地址: http://www.infradead.org/openconnect/download.html

下载 & 解压:
```
wget ftp://ftp.infradead.org/pub/openconnect/openconnect-7.08.tar.gz
tar xvf openconnect-7.08.tar.gz
cd openconnect-7.08
```

安装依赖:

```
sudo apt-get install libgnutls28-dev libxml2-dev
```

编译 & 安装:

```
sudo ./configure
sudo make
sudo make install
sudo ldconfig
```

创建文件`/usr/local/bin/openconnect-start`:

```
```




```
# copy openconnect-start to /usr/local/bin/openconnect-start
# copy rc.local to /etc/rc.local
```
