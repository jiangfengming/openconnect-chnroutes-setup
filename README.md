# Openconnect + chnroutes + ChinaDNS + Dnsmasq 实现家庭所有设备区分国内外IP翻墙

## 1. 环境
国外VPS：Ubuntu 16.04。家里一台双网口的Ubuntu 16.04作为网关，下连交换机和无线AP。


## 2. 国外VPS安装ocserv

### 编译

下载ocserv

地址: http://www.infradead.org/ocserv/download.html
```
wget ftp://ftp.infradead.org/pub/ocserv/ocserv-0.11.4.tar.xz
tar xvf ocserv-0.11.4.tar.xz
cd ocserv-0.11.4
```

安装依赖包
```
sudo apt-get install build-essential libgnutls28-dev libev-dev libwrap0-dev libpam0g-dev liblz4-dev libseccomp-dev libreadline-dev libnl-route-3-dev libkrb5-dev liboath-dev
```

编译 & 安装
```
./configure
make
sudo make install
```

检查
```
ocserv -v
```
能输出版本信息就OK

### 制作CA和服务器证书
安装所需工具
```
sudo apt-get install gnutls-bin
```

先建个文件夹，在这里面生成文件
```
mkdir cert
cd cert
```
生成的文件以后重装也会用到，可以保存一下，就不用重新做了。


生成CA密钥
```
certtool --generate-privkey --outfile ca-key.pem
```

创建ca模板文件`ca.tmpl`，填写如下内容，其中`cn`和`organization`替换为自己喜欢的名字
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

生成CA证书
```
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```

生成服务器证书密钥
```
certtool --generate-privkey --outfile server-key.pem
```

创建服务器证书模板文件`server.tmpl`，填写如下内容，这里的`cn`必须填写之后连接VPN时使用的域名或IP，`organization`替换为自己喜欢的名字
```
cn = "My Company server"
organization = "My Company"
expiration_days = -1
signing_key
encryption_key
tls_www_server
```

生成服务器证书
```
certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template server.tmpl --outfile server-cert.pem
```

### 配置ocserv
创建配置文件夹
```
sudo mkdir /etc/ocserv
```

复制server-key.pem和server-cert.pem到配置文件夹
```
cp server-key.pem /etc/ocserv/
cp server-cert.pem /etc/ocserv/
```

回到解压的文件夹，复制一份配置文件样本
```
sudo cp doc/sample.config /etc/ocserv/ocserv.conf
```

修改配置文件以下参数
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

# sysctl.conf
bash：
```
sysctl -w net.ipv4.ip_forward=1
sudo vi /etc/sysctl.conf
```
sysctl.conf：
```
net.ipv4.ip_forward = 1
```


# dnsmasq
bash：
```
sudo apt-get install dnsmasq
sudo vi /etc/dnsmasq.conf
```
dnsmasq.conf：
```
no-resolv
server=114.114.114.114#5333
address=/home.noindoin.com/192.168.0.1
address=/sync.noindoin.com/192.168.0.1
address=/doc.noindoin.com/192.168.0.1
address=/.17dev.link/192.168.0.1
dhcp-range=192.168.0.50,192.168.0.150,12h
# xbox one
dhcp-host=c0:33:5e:42:db:a7,192.168.0.36
# yoga
dhcp-host=8c:ae:4c:fe:1c:8f,192.168.0.2
```


# interfaces
bash:
```
sudo vi /etc/network/interfaces
```
interfaces:
```
auto enx000ec6c50775
iface enx000ec6c50775 inet static
address 192.168.0.1
netmask 255.255.255.0
```
bash:
```
sudo ifdown enx000ec6c50775
sudo ifup enx000ec6c50775
```

# ChinaDNS
https://github.com/shadowsocks/ChinaDNS
cd ~/Downloads
wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz
tar xvf chinadns-1.3.2.tar.gz
cd chinadns-1.3.2
./configure
make
sudo mkdir /usr/local/chinadns
sudo cp src/chinadns /usr/local/chinadns/
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > chnroute.txt
sudo cp chnroute.txt /usr/local/chinadns/
/usr/local/chinadns/chinadns -m -c /usr/local/chinadns/chnroute.txt -p 5333 -s 114.114.114.114,8.8.8.8


# Openconnect client
#http://www.infradead.org/openconnect/download.html

cd ~/Downloads

#http://www.infradead.org/openconnect/vpnc-script.html

wget http://git.infradead.org/users/dwmw2/vpnc-scripts.git/blob_plain/HEAD:/vpnc-script
sudo mkdir /etc/vpnc
sudo cp vpnc-script /etc/vpnc/
sudo chmod u+x,g+x /etc/vpnc/vpnc-script
sudo apt-get install libgnutls28-dev
sudo apt-get install libxml2-dev
wget ftp://ftp.infradead.org/pub/openconnect/openconnect-7.06.tar.gz
tar xvf openconnect-7.06.tar.gz
cd openconnect-7.06
sudo ./configure
sudo make
sudo make install
sudo ldconfig


sudo cp /usr/local/chinadns/chnroutes /usr/local/bin/chnroutes
sudo chmod u+x,g+x /usr/local/bin/chnroutes
sudo vi /usr/local/bin/chnroutes
:%s/.\+/ip route add \0 via 192.168.1.1
gg
O
#!/bin/sh
:wq


# copy vpnc to /etc/vpnc
# copy iptables.rules to /etc/iptables.rules
# copy openconnect-start to /usr/local/bin/openconnect-start
# copy rc.local to /etc/rc.local
