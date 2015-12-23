OpenConnet Server（ocserv）是通过实现Cisco的AnyConnect协议，用DTLS作为主要的加密传输协议。我认为它的主要好处在于——

* AnyConnect的VPN协议默认使用UDP DTLS作为数据传输，但如果有什么网络问题导致UDP传输出现问题，它会利用最初建立的TCP TLS通道作为备份通道，降低VPN断开的概率。
* AnyConnect作为Cisco新一代的VPN解决方案，被用于许多大型企业，这些企业依赖它提供正常的商业运作，这些正常运作对应的经济效益（读作GDP），是我们最好的伙伴。
* OpenConnet的架设足够麻烦，我的意思是，如果你不是大型企业，你会用AnyConnect的概率无限趋近于零。再者，如果它足够简单，我就不用写这篇文章了。
至于它的自定义路由表支持，我觉得都是次要了。

介绍到此，让我们按步骤干好事情。

（下文选用最新的Ubuntu 14.04 LTS和OCServ 0.10.10作为标准环境）  

## 1.配置环境
### 1.1安装依赖
```
apt-get install make libgnutls28-dev libwrap0-dev libpam0g-dev liblz4-dev libseccomp-dev libreadline-dev libnl-route-3-dev libkrb5-dev libprotobuf-c0-dev libtalloc-dev libhttp-parser-dev libpcl1-dev libopts25-dev autogen protobuf-c-compiler gperf liblockfile-bin nuttcp 
```

### 1.2编译libtasn1
```
wget http://ftp.gnu.org/gnu/libtasn1/libtasn1-4.7.tar.gz
tar zxf libtasn1*.gz && cd libtasn1*
./configure 
make && make install
```


***

## 2.编译OCserv
查看最新版本	http://www.infradead.org/ocserv/download.html
```
wget ftp://ftp.infradead.org/pub/ocserv/ocserv-0.10.10.tar.xz
tar xvf ocserv-0.10.10.tar.xz
cd ocserv-0.10.10
./configure 
make && make install
```


***

## 3.配置OpenConnectServer

### 3.1配置证书
安装证书工具  `apt-get install gnutls-bin`
```
cd ~
mkdir certificates
cd certificates
```

在此目录下创建一个名为`vi ca.tmpl` 的CA证书模板，写入如下语句：
```
cn = "Your CA name" 
organization = "Your fancy name" 
serial = 1 
expiration_days = 3650
ca 
signing_key 
cert_signing_key 
crl_signing_key
```
生成CA密钥
`certtool --generate-privkey --outfile ca-key.pem`

生成CA证书
`certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem`

然后我们生成服务器证书`vi server.tmpl`，这里注意cn项必须对应你服务器的域名或IP，内容如下：
```
cn = "Your hostname or IP" 
organization = "Your fancy name" 
expiration_days = 3650
signing_key 
encryption_key
tls_www_server
```
生成Server密钥
`certtool --generate-privkey --outfile server-key.pem`

生成Server证书
`certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template server.tmpl --outfile server-cert.pem`

把证书移动到合适的地方：
```
cp ca-cert.pem /etc/ssl/private/my-ca-cert.pem
cp server-cert.pem /etc/ssl/private/my-server-cert.pem
cp server-key.pem /etc/ssl/private/my-server-key.pem
```


### 3.2 准备配置文件
我们把配置文件放到ocserv默认读取的位置：
```
mkdir /etc/ocserv
cd ~/ocserv*
cp doc/sample.config /etc/ocserv/ocserv.conf
```
配置文件可以[官方手册](http://www.infradead.org/ocserv/manual.html)手册来写，不过这里我们重点要确保以下条目正确：
```
# 登陆方式，目前先用密码登录
auth = "plain[/etc/ocserv/ocpasswd]"
 
# 允许同时连接的客户端数量
max-clients = 4
 
# 限制同一客户端的并行登陆数量
max-same-clients = 2
 
# 服务监听的IP（服务器IP，可不设置）
listen-host = 1.2.3.4
 
# 服务监听的TCP/UDP端口（选择你喜欢的数字）
tcp-port = 9000
udp-port = 9001
 
# 自动优化VPN的网络性能
try-mtu-discovery = true
 
# 确保服务器正确读取用户证书（后面会用到用户证书）
cert-user-oid = 2.5.4.3
 
# 服务器证书与密钥
ca-cert = /etc/ssl/private/my-ca-cert.pem
server-cert = /etc/ssl/private/my-server-cert.pem
server-key = /etc/ssl/private/my-server-key.pem
 
# 客户端连上vpn后使用的dns
dns = 8.8.8.8
dns = 8.8.4.4
 
# 注释掉所有的route，让服务器成为gateway
#route = 192.168.1.0/255.255.255.0
 
# 启用cisco客户端兼容性支持
cisco-client-compat = true
```
### 3.3 测试服务器  
创建一个登陆用的用户名与密码。 
`ocpasswd -c /etc/ocserv/ocpasswd your-username`  

修改内核设置，使得支持转发，`vi /etc/sysctl.conf`,将`net.ipv4.ip_forward=0`改为`net.ipv4.ip_forward=1`  
保存生效`sysctl -p`  

查看网卡信息`ifconfig`用于修改防火墙  
开启NAT转发，venet0和IP要对应网卡和配置的IP    
```
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o venet0 -j MASQUERADE
iptables -A FORWARD -s 192.168.1.0/24 -j ACCEPT
```
使用`iptables -t nat -L`来验证转发是否开启成功  
```
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
 
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  192.168.1.0/24       anywhere
```
使用`ocserv -f -d 1`命令来启动下服务啦  
打开你手机上的OpenConnect/Cisco Anyconnect新建一个VPN，添加服务器IP就是你的vps的IP:端口,例如`X.X.X.X:443`  
好了，如果你看到如下信息，那服务器应该已经能够正常运行了：  
```
Parsing plain auth method subconfig using legacy format
Setting 'plain' as primary authentication method
listening (TCP) on 0.0.0.0:110...
listening (UDP) on 0.0.0.0:9000...
ocserv[16104]: main: initialized ocserv 0.10.8
ocserv[16105]: sec-mod: reading supplemental config from files
ocserv[16105]: sec-mod: sec-mod initialized (socket: /var/run/ocserv-socket.16104)
ocserv[16109]: GnuTLS error (at worker-vpn.c:468): A TLS fatal alert has been received.: Unknown certificate
ocserv[16104]: main: 60.0.14.48:9890 user disconnected
ocserv[16105]: sec-mod: using 'plain' authentication to authenticate user (session: FXS0l)
ocserv[16104]: main: 60.0.14.48:36627 user disconnected
ocserv[16105]: sec-mod: initiating session for user 'test' (session: FXS0l)
ocserv[16104]: main[test]: 60.0.14.48:9663 new user session
ocserv[16104]: main[test]: 60.0.14.48:9663 user logged in
ocserv[16104]: main: 60.0.14.48:46429 user disconnected
ocserv[16104]: main[test]: 60.0.14.48:9663 user disconnected
ocserv[16105]: sec-mod: temporarily closing session for test (session: FXS0l)
ocserv[16105]: sec-mod: initiating session for user 'test' (session: FXS0l)ocserv[16104]: main[test]: 60.0.14.48:38135 new user session
ocserv[16104]: main[test]: 60.0.14.48:38135 user logged in
```
好了，目前已经搞定了OpenConnect server，下面讲的是一些优化，创建客户端证书，智能分流  

## 4.优化OpenConnectServer