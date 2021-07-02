# 使用场景
自建数据中心，需要采集各个客户的业务数据和IOT数据。每个客户都有自己完全独立的机房，机房之间无法直接通信，需要有办法能够安全的把数据传输到数据中心里。
# 目标
- 保证数据安全，不会被截获
- 能够实现远程控制任务流，实现远程调度
- 能够实现对remote input port的权限管理
- 尽可能少的暴露机房的真实ip和端口
- 能够识别NIFI用户身份
# 解决方案
- 通过nginx反向代理实现隐藏真实ip和端口号
- 用户认证通过tls证书识别用户身份，及权限管理
- 通过nifi的s2s功能实现远程数据传输
![nifi-s2s-security](/images/nifi-s2s-security.png)
# 什么是s2s
将数据从一个NiFi实例发送到另一个实例时，可以使用许多不同的协议。不过，首选协议是NiFi站点到站点协议。通过站点到站点，可以轻松安全地和高效地将数据与一个NiFi实例中的节点或数据生成应用程序之间的数据传输到另一个NiFi实例。
使用站点到站点具有以下好处：
易于配置: 输入远程NiFi实例的URL后，将自动发现可用的端口（端点）并将其提供在下拉列表中
安全: 站点到站点可以选择使用证书来加密数据并提供身份验证和授权。可以将每个端口配置为仅允许特定用户，并且只有那些用户才能看到该端口甚至存在。
可扩展: 当远程集群中的节点发生更改时，将自动检测到这些更改，并在集群中的所有节点之间扩展数据。
高效: 站点到站点允许立即发送批量的FlowFile，以避免建立连接以及在对等方之间进行多个往返请求的开销。
可靠: 校验和由发送方和接收方自动生成，并在数据传输后进行比较，以确保不会发生损坏。如果校验和不匹配，事务将被简单地取消并重试。
自动负载均衡: 当节点联机或从远程群集中退出时，或者节点的负载变大或变轻时，定向到该节点的数据量将自动进行调整。
FlowFiles维护属性: 通过此协议传输FlowFile时，所有FlowFile的属性都会随之自动传输。在许多情况下，这是非常有利的，因为由NiFi的一个实例确定的所有上下文和丰富度都会随数据一起传播，从而使数据易于路由并允许用户轻松检查数据。
适应性强: 随着新技术和思想的涌现，用于处理站点间通信的协议也随之变化。与远程NiFi实例建立连接时，将执行一次握手以协商将使用哪个协议和哪个版本的协议。这允许添加新功能，同时仍保持与所有旧实例的向后兼容性。此外，如果在协议中发现了漏洞或不足，则它将允许较新版本的NiFi禁止通过协议的受感染版本进行通信。
主要操作方式：
- 推送：客户端将数据发送到远程进程组，服务器通过输入端口接收数据
- 接收：客户端从远程进程组接收数据，服务器通过输出端口发送数据
更多详情[点击这里查看](https://nifi.apache.org/docs/nifi-docs/html/user-guide.html#site-to-site)
# TLS安全配置
## NIFI是如何验证DN的
什么是tls[点击这里查看](https://juejin.im/post/6844903667577929742)
参考nifi-tookit[点击这里查看](https://nifi.apache.org/docs.html)
# 用户管理与授权

# 开始部署
## 运行环境
系统：centos 7.8
机器配置： 2G内存+4核心CPU+120G硬盘 * 3
nifi版本：1.11.4
nginx版本: 1.16.x
机器1： app：nginx ip:192.168.2.60 host:ca.nifiserver.com 127.0.0.1 s61.nifiserver.com 192.168.2.61 s62.nifiserver.com 192.168.2.62 -> nifi代理服务器，生成证书的服务器
机器2： app: nifi  ip:192.168.2.61 host:s61.nifiserver.com 0.0.0.0 p62.nifiserver.com 192.168.2.60 -> nifi实例1(注意nifi本地ip必须是0.0.0.0,nifi就是这么设计的，所以需要在本地添加host)
机器2： app: nifi  ip:192.168.2.62 host:s62.nifiserver.com 0.0.0.0 p61.nifiserver.com 192.168.2.60 -> nifi实例2(注意nifi本地ip必须是0.0.0.0,nifi就是这么设计的，所以需要在本地添加host)
## 配置运行环境
### 安装java
执行以下脚本后重启机器 java即安装完成(地址可能会失效，自己重新下载java安装包就行了)
```shell script
#!/usr/bin/env bash
JAVA_RES="http://home.37work.cn:1288/public/linux/jdk-8u261-linux-x64.tar.gz";
JAVA_HOME="/apps/java8"
if ! test -d $JAVA_HOME; then
    echo "start install java..."
    yum install -y wget
    wget -q $JAVA_RES && tar -zxf jdk-8u261-linux-x64.tar.gz && rm -rf jdk-8u261-linux-x64.tar.gz
    mkdir -p $JAVA_HOME && rm -rf $JAVA_HOME && mv jdk* $JAVA_HOME
    echo "
    export JAVA_HOME=$JAVA_HOME
    export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
    export PATH=\$PATH:\$JAVA_HOME/bin
    export JAVA_HOME CLASSPATH PATH
    " >> /etc/profile
    source /etc/profile
    java -version
    echo "java install is done!"
fi
source /etc/profile
java -version
```
### 安装nginx
```shell script
yum install -y epel-release
yum install -y nginx
```
### 下载nifi
nifi:[点击这里下载](https://mirrors.tuna.tsinghua.edu.cn/apache/nifi/1.11.4/nifi-1.11.4-bin.zip)
nifi-toolkit[点击这里下载](https://mirrors.tuna.tsinghua.edu.cn/apache/nifi/1.11.4/nifi-toolkit-1.11.4-bin.zip)

## 生成TLS证书
按照下面的例子，分别生成这4个JKS证书,用于proxy和s2s通信（CN必须是对应机器的域名，否则会TLS握手失败） CN=p61.nifiserver.com,CN=NIFI CN=p62.nifiserver.com,CN=NIFI CN=s61.nifiserver.com,CN=NIFI CN=s62.nifiserver.com,CN=NIFI
再额外生成一个pkcs12证书 CN=browser,CN=NIFI， 用于浏览器访问nifi进行安全认证.

```
# 登录192.168.2.60机器配置好host后进行以下操作
#启动nifi-toolkit server
bin/tls-toolkit.sh server -c ca.nifiserver.com -t differentKeyAndKeystorePasswords &

mkdir s61 s62 p61 p62
# 生成nifi服务证书
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=s61.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks s61
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=s62.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks s62
#生成nginx代理证书
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=p61.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks p61
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=p62.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks p62
#生成浏览器用的证书
mkdir browser
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=browseruser,OU=NIFI -T PKCS12
mv config.json keystore.pkcs12 nifi-cert.pem truststore.jks browser/

#转换nginx代理证书为pem/key格式,密钥在config.json 中的keyStorePassword就是转换时要输入的密码
cd p61
keytool -importkeystore -srckeystore keystore.jks -destkeystore nifiserver.p12 -srcstoretype jks -deststoretype pkcs12
openssl pkcs12 -nodes -in nifiserver.p12 -out nifiserver-fullchain.pem
touch nifiserver.key nifiserver.pem

cd p62
keytool -importkeystore -srckeystore keystore.jks -destkeystore nifiserver.p12 -srcstoretype jks -deststoretype pkcs12
openssl pkcs12 -nodes -in nifiserver.p12 -out nifiserver-fullchain.pem
touch nifiserver.key nifiserver.pem
cat nifiserver-fullchain.pem
```
# nifiserver-fullchain.pem 
以下为nifiserver-fullchain.pem的内容
```text
Bag Attributes
    friendlyName: nifi-key
    localKeyID: 54 69 6D 65 20 31 35 39 36 38 30 38 36 34 38 34 32 35 
Key Attributes: <No Attributes>
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCMyg2FDBUVpWrG
Fi6F6bGjOwE+OBWVjYINHEM0plGjxcqljxFwgYgNl3qg+Zdcr7HIEBorB4kqxBhV
SqR2puFPFyBvSmDJxKAbrnwrztPN9UxYltC4Xz1TFVRx3fk3xZb113RvZrxA2u66
/oObFgG1xzntjFyavQfxV0Vk1jQP6YAq+w2FfxD7xgt1WWJEWJcg3qUYYuLYhrRQ
JHxVCys3q2ve0A7mIs3rAje63KlT+3w4EaXTRTH8FLRruDVxjeLQvDuyPD/ZXDVk
QdfXsixWm689tCo1EKhhZw859C8gznWe55rw64x5/q4bx8Rwc7z6DXMPOXJY+5wi
JMNZGvPvAgMBAAECggEAIoJkTfhoMqYZRfSp8qkVoa0U4OteXwoQlqYW0xDxcfNJ
eMtYuvsFHk/C/zIup8lpCmDoCSQPuyvVyxJAvdSp5XkFukHA97P6is56IULRJ+q4
i/5rqsWtgm/4AvEl5UXJevkU0Tmda0g+vBcmqxz5zlTHHjMJf+RVzhJWMCFRIZT9
b4hfvtFv+nobmaH210ExAXiiCgCq2hn7ADSJjytaAXQ0V12FXc/EXKNOQ357/PUe
2uogHxtNHD6MYhA8gcYyIhQv1jxlr4MLCgMGXRs2bmdFh198FZkboza7vGCGeqD9
f2z3hEqrF1pZIxyvOBamk28VpzX9O4DMPScSHcSqAQKBgQDF0bUt7FuVynEZWVQ8
I6TvC7rHFl8rysbpK5HRyU/6eVqnuzfplIXJua6fymMc7qJ4yfigSIF6cD660zU9
DEO0yTy8z7fy8xWclLBA0FsUVPCARBogwfXKJyafYh2bSud5YNDBD0W9eb9HBYE0
YL/bpBSV5o4lQVMhOlog5TsLqwKBgQC2MmxjDaVJBtgp4/abhtIfXu5D9Tjts+tQ
VcqbMB13bH7TQOMpeQ2u1ObMW1z/uDLVRzKZSIrqrU6ys5fmxR6EFKgBx0s/9fIH
xipRKXt/4pkEajkW4fBERzfIyBbJdWaZ0EfznUc4h891aAgOM05rX9N983Kf7+mh
sfAg2HvUzQKBgGlYrYjgR0G1Bpf+R2qjfNFEyNn/Iv26RkWkS0qST8JO4CVVAYil
7L2p4cH80N12hBWZUYtiMXnXzsBFfCOfpWrghDT01bxPEeJKGLbbfrWMKmvUWKm7
QT6/rMTSRnwN3sl38pPtozEtZdzXpKAVKfc5ITFXD7ntWOzoG1lLWi9zAoGAKxUn
ThDm+aq1qMowATzTKPngq48sBAFcbmWrACFThm7QWpHoZWErnCDZ5o7gIdPjqU0p
qNdfifirOFSBYd9QxPjBdZIzuA8nSTFRxllhy67AcivQDholH3Abv82YndC2Dz8S
FIgnVDXBF8kexoTZUUiakRjlDO7FNygFWS73sS0CgYEAkWO996oYoRzjW0NOavRM
z+S8WWCaoN2L/jb7S53jp40uU/LsRBmB//hF+1kxgMgGG3QR3rHNhTp2OdvgfTsl
iZkScu6h2m7jYMhb817DQCA5WhKJZzM4wBTp3kc+QP1rKfKcOP6Ysyd00Qr7/u11
ScEUx9lZXT+JT/80ahpa7Yw=
-----END PRIVATE KEY-----
Bag Attributes
    friendlyName: nifi-key
    localKeyID: 54 69 6D 65 20 31 35 39 36 38 30 38 36 34 38 34 32 35 
subject=/OU=NIFI/CN=p61.nifiserver.com
issuer=/OU=NIFI/CN=ca.nifiserver.com
-----BEGIN CERTIFICATE-----
MIIDdjCCAl6gAwIBAgIKAXPJIRZ9AAAAADANBgkqhkiG9w0BAQsFADArMQ0wCwYD
VQQLDAROSUZJMRowGAYDVQQDDBFjYS5uaWZpY2VudGVyLmNvbTAeFw0yMDA4MDcx
MzM0MTVaFw0yMjExMTAxMzM0MTVaMCsxDTALBgNVBAsMBE5JRkkxGjAYBgNVBAMM
EXA2MS5uaWZpcHJveHkuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAjMoNhQwVFaVqxhYuhemxozsBPjgVlY2CDRxDNKZRo8XKpY8RcIGIDZd6oPmX
XK+xyBAaKweJKsQYVUqkdqbhTxcgb0pgycSgG658K87TzfVMWJbQuF89UxVUcd35
N8WW9dd0b2a8QNruuv6DmxYBtcc57Yxcmr0H8VdFZNY0D+mAKvsNhX8Q+8YLdVli
RFiXIN6lGGLi2Ia0UCR8VQsrN6tr3tAO5iLN6wI3utypU/t8OBGl00Ux/BS0a7g1
cY3i0Lw7sjw/2Vw1ZEHX17IsVpuvPbQqNRCoYWcPOfQvIM51nuea8OuMef6uG8fE
cHO8+g1zDzlyWPucIiTDWRrz7wIDAQABo4GbMIGYMB0GA1UdDgQWBBQf+sja3LF5
BExpI/snTWprbGEr9DAfBgNVHSMEGDAWgBThEUaMSwkUEtog4vP0xSXWitG1JDAO
BgNVHQ8BAf8EBAMCA/gwCQYDVR0TBAIwADAdBgNVHSUEFjAUBggrBgEFBQcDAgYI
KwYBBQUHAwEwHAYDVR0RBBUwE4IRcDYxLm5pZmlwcm94eS5jb20wDQYJKoZIhvcN
AQELBQADggEBAGyUvJo0/nVtDOs0ZU6rtqqJLhhS1Xxp2ETYkERYBtFWdIeHU9ax
4arOiM246K37hBI9Ij1yrNIn4jZKmFQgOgm1KiZPWcWcRpmxTO6UTh7m6f2Xr7hf
BeLkKnt8b/K99IGgNlpeUlUOmT+xP06/JrliR5Fpg4iF58YYy+I0XHjXFJoYhMta
8uB784pivWqJrMzYr9Tvq0Pb+iw/0mGbHV5z7XY1lX81bkd3SY5op52Qg66oeZ06
FXnPzrRd9mSTfZJUFQB9flnhJvHSMs1OD/FSBPORcHuF2H9IDGObieQc2/+0nL9T
CwULqMO9p/smTWUYkCGMEj928qnakibmBSY=
-----END CERTIFICATE-----
Bag Attributes
    friendlyName: CN=ca.nifiserver.com,OU=NIFI
subject=/OU=NIFI/CN=ca.nifiserver.com
issuer=/OU=NIFI/CN=ca.nifiserver.com
-----BEGIN CERTIFICATE-----
MIIDWTCCAkGgAwIBAgIKAXPJGfCUAAAAADANBgkqhkiG9w0BAQsFADArMQ0wCwYD
VQQLDAROSUZJMRowGAYDVQQDDBFjYS5uaWZpY2VudGVyLmNvbTAeFw0yMDA4MDcx
MzI2MjdaFw0yMjExMTAxMzI2MjdaMCsxDTALBgNVBAsMBE5JRkkxGjAYBgNVBAMM
EWNhLm5pZmljZW50ZXIuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEA3xl3Tm98Eac+BIJvSroVE34T3/vFctAH6V52K1irvmo2ZfaeVUqIqVnwEnqu
i8Ka/KDh+bE5dE06ZNDwhbydiH/Cwwtxutqld4eFV7jNYUVYzQB3cqPJXx5wjzbK
wU1hYcyw3oh32ct3GN5e9joFzz1yreziFfSjXm8U6qZXg0bjLrUc7Xjlb9C6CUxZ
2l0lZCuuh4ATRuML85zDP6BU5kcBHGRjFuZb+8IVSe8/XeMrDOwNykDiJxaa55dI
h14yEjytwmMbLE6l8JdOZK2Yk7DcKwXrAlQ3qL+TyGqZmDo3sO+GyrX8QQBJyjnL
E2M9v5rW8vpwAtmub311JjDQTQIDAQABo38wfTAOBgNVHQ8BAf8EBAMCAf4wDAYD
VR0TBAUwAwEB/zAdBgNVHQ4EFgQU4RFGjEsJFBLaIOLz9MUl1orRtSQwHwYDVR0j
BBgwFoAU4RFGjEsJFBLaIOLz9MUl1orRtSQwHQYDVR0lBBYwFAYIKwYBBQUHAwIG
CCsGAQUFBwMBMA0GCSqGSIb3DQEBCwUAA4IBAQCd1RUOyhm3CUK07w/xXFo4qauX
Ogn3N2alz8ATX8GKLp1AJ2K0XjsrhPlnUsJqDd1RtpA/DuvR7X2LAiuZ4OVshY4E
b1uNYSsAKogxLJcJgZXVSrGzpoCj+HE1c77TtQL6BtWZf4NkHsRfSveHTEpeiLZO
vgMZtLeZBUAXcLLAXFBBS5NgffPiZpLQSsjnQucTLPTGJpW+HmRpvlPn7/3UlddO
U/i+hQDA2ENP0YP339MtZVnYqlZZKuyav2yJiqWw3xK4YrHKd1t3v6dNYIpgNc8w
lwGcsRazfF5MGtNjd4TqUfINRdBiunKmTeyljZKBUztuv6QGhLnpdveo2w1c
-----END CERTIFICATE-----
```

### nifiserver.key 
文件内容例子(从nifiserver-fullchain.pem复制出来的)
```text
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCMyg2FDBUVpWrG
Fi6F6bGjOwE+OBWVjYINHEM0plGjxcqljxFwgYgNl3qg+Zdcr7HIEBorB4kqxBhV
SqR2puFPFyBvSmDJxKAbrnwrztPN9UxYltC4Xz1TFVRx3fk3xZb113RvZrxA2u66
/oObFgG1xzntjFyavQfxV0Vk1jQP6YAq+w2FfxD7xgt1WWJEWJcg3qUYYuLYhrRQ
JHxVCys3q2ve0A7mIs3rAje63KlT+3w4EaXTRTH8FLRruDVxjeLQvDuyPD/ZXDVk
QdfXsixWm689tCo1EKhhZw859C8gznWe55rw64x5/q4bx8Rwc7z6DXMPOXJY+5wi
JMNZGvPvAgMBAAECggEAIoJkTfhoMqYZRfSp8qkVoa0U4OteXwoQlqYW0xDxcfNJ
eMtYuvsFHk/C/zIup8lpCmDoCSQPuyvVyxJAvdSp5XkFukHA97P6is56IULRJ+q4
i/5rqsWtgm/4AvEl5UXJevkU0Tmda0g+vBcmqxz5zlTHHjMJf+RVzhJWMCFRIZT9
b4hfvtFv+nobmaH210ExAXiiCgCq2hn7ADSJjytaAXQ0V12FXc/EXKNOQ357/PUe
2uogHxtNHD6MYhA8gcYyIhQv1jxlr4MLCgMGXRs2bmdFh198FZkboza7vGCGeqD9
f2z3hEqrF1pZIxyvOBamk28VpzX9O4DMPScSHcSqAQKBgQDF0bUt7FuVynEZWVQ8
I6TvC7rHFl8rysbpK5HRyU/6eVqnuzfplIXJua6fymMc7qJ4yfigSIF6cD660zU9
DEO0yTy8z7fy8xWclLBA0FsUVPCARBogwfXKJyafYh2bSud5YNDBD0W9eb9HBYE0
YL/bpBSV5o4lQVMhOlog5TsLqwKBgQC2MmxjDaVJBtgp4/abhtIfXu5D9Tjts+tQ
VcqbMB13bH7TQOMpeQ2u1ObMW1z/uDLVRzKZSIrqrU6ys5fmxR6EFKgBx0s/9fIH
xipRKXt/4pkEajkW4fBERzfIyBbJdWaZ0EfznUc4h891aAgOM05rX9N983Kf7+mh
sfAg2HvUzQKBgGlYrYjgR0G1Bpf+R2qjfNFEyNn/Iv26RkWkS0qST8JO4CVVAYil
7L2p4cH80N12hBWZUYtiMXnXzsBFfCOfpWrghDT01bxPEeJKGLbbfrWMKmvUWKm7
QT6/rMTSRnwN3sl38pPtozEtZdzXpKAVKfc5ITFXD7ntWOzoG1lLWi9zAoGAKxUn
ThDm+aq1qMowATzTKPngq48sBAFcbmWrACFThm7QWpHoZWErnCDZ5o7gIdPjqU0p
qNdfifirOFSBYd9QxPjBdZIzuA8nSTFRxllhy67AcivQDholH3Abv82YndC2Dz8S
FIgnVDXBF8kexoTZUUiakRjlDO7FNygFWS73sS0CgYEAkWO996oYoRzjW0NOavRM
z+S8WWCaoN2L/jb7S53jp40uU/LsRBmB//hF+1kxgMgGG3QR3rHNhTp2OdvgfTsl
iZkScu6h2m7jYMhb817DQCA5WhKJZzM4wBTp3kc+QP1rKfKcOP6Ysyd00Qr7/u11
ScEUx9lZXT+JT/80ahpa7Yw=
-----END PRIVATE KEY-----
```
### nifiserver.pem 文件内容例子(从nifiserver-fullchain.pem复制出来的)
```text
-----BEGIN CERTIFICATE-----
MIIDdjCCAl6gAwIBAgIKAXPJIRZ9AAAAADANBgkqhkiG9w0BAQsFADArMQ0wCwYD
VQQLDAROSUZJMRowGAYDVQQDDBFjYS5uaWZpY2VudGVyLmNvbTAeFw0yMDA4MDcx
MzM0MTVaFw0yMjExMTAxMzM0MTVaMCsxDTALBgNVBAsMBE5JRkkxGjAYBgNVBAMM
EXA2MS5uaWZpcHJveHkuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAjMoNhQwVFaVqxhYuhemxozsBPjgVlY2CDRxDNKZRo8XKpY8RcIGIDZd6oPmX
XK+xyBAaKweJKsQYVUqkdqbhTxcgb0pgycSgG658K87TzfVMWJbQuF89UxVUcd35
N8WW9dd0b2a8QNruuv6DmxYBtcc57Yxcmr0H8VdFZNY0D+mAKvsNhX8Q+8YLdVli
RFiXIN6lGGLi2Ia0UCR8VQsrN6tr3tAO5iLN6wI3utypU/t8OBGl00Ux/BS0a7g1
cY3i0Lw7sjw/2Vw1ZEHX17IsVpuvPbQqNRCoYWcPOfQvIM51nuea8OuMef6uG8fE
cHO8+g1zDzlyWPucIiTDWRrz7wIDAQABo4GbMIGYMB0GA1UdDgQWBBQf+sja3LF5
BExpI/snTWprbGEr9DAfBgNVHSMEGDAWgBThEUaMSwkUEtog4vP0xSXWitG1JDAO
BgNVHQ8BAf8EBAMCA/gwCQYDVR0TBAIwADAdBgNVHSUEFjAUBggrBgEFBQcDAgYI
KwYBBQUHAwEwHAYDVR0RBBUwE4IRcDYxLm5pZmlwcm94eS5jb20wDQYJKoZIhvcN
AQELBQADggEBAGyUvJo0/nVtDOs0ZU6rtqqJLhhS1Xxp2ETYkERYBtFWdIeHU9ax
4arOiM246K37hBI9Ij1yrNIn4jZKmFQgOgm1KiZPWcWcRpmxTO6UTh7m6f2Xr7hf
BeLkKnt8b/K99IGgNlpeUlUOmT+xP06/JrliR5Fpg4iF58YYy+I0XHjXFJoYhMta
8uB784pivWqJrMzYr9Tvq0Pb+iw/0mGbHV5z7XY1lX81bkd3SY5op52Qg66oeZ06
FXnPzrRd9mSTfZJUFQB9flnhJvHSMs1OD/FSBPORcHuF2H9IDGObieQc2/+0nL9T
CwULqMO9p/smTWUYkCGMEj928qnakibmBSY=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDWTCCAkGgAwIBAgIKAXPJGfCUAAAAADANBgkqhkiG9w0BAQsFADArMQ0wCwYD
VQQLDAROSUZJMRowGAYDVQQDDBFjYS5uaWZpY2VudGVyLmNvbTAeFw0yMDA4MDcx
MzI2MjdaFw0yMjExMTAxMzI2MjdaMCsxDTALBgNVBAsMBE5JRkkxGjAYBgNVBAMM
EWNhLm5pZmljZW50ZXIuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEA3xl3Tm98Eac+BIJvSroVE34T3/vFctAH6V52K1irvmo2ZfaeVUqIqVnwEnqu
i8Ka/KDh+bE5dE06ZNDwhbydiH/Cwwtxutqld4eFV7jNYUVYzQB3cqPJXx5wjzbK
wU1hYcyw3oh32ct3GN5e9joFzz1yreziFfSjXm8U6qZXg0bjLrUc7Xjlb9C6CUxZ
2l0lZCuuh4ATRuML85zDP6BU5kcBHGRjFuZb+8IVSe8/XeMrDOwNykDiJxaa55dI
h14yEjytwmMbLE6l8JdOZK2Yk7DcKwXrAlQ3qL+TyGqZmDo3sO+GyrX8QQBJyjnL
E2M9v5rW8vpwAtmub311JjDQTQIDAQABo38wfTAOBgNVHQ8BAf8EBAMCAf4wDAYD
VR0TBAUwAwEB/zAdBgNVHQ4EFgQU4RFGjEsJFBLaIOLz9MUl1orRtSQwHwYDVR0j
BBgwFoAU4RFGjEsJFBLaIOLz9MUl1orRtSQwHQYDVR0lBBYwFAYIKwYBBQUHAwIG
CCsGAQUFBwMBMA0GCSqGSIb3DQEBCwUAA4IBAQCd1RUOyhm3CUK07w/xXFo4qauX
Ogn3N2alz8ATX8GKLp1AJ2K0XjsrhPlnUsJqDd1RtpA/DuvR7X2LAiuZ4OVshY4E
b1uNYSsAKogxLJcJgZXVSrGzpoCj+HE1c77TtQL6BtWZf4NkHsRfSveHTEpeiLZO
vgMZtLeZBUAXcLLAXFBBS5NgffPiZpLQSsjnQucTLPTGJpW+HmRpvlPn7/3UlddO
U/i+hQDA2ENP0YP339MtZVnYqlZZKuyav2yJiqWw3xK4YrHKd1t3v6dNYIpgNc8w
lwGcsRazfF5MGtNjd4TqUfINRdBiunKmTeyljZKBUztuv6QGhLnpdveo2w1c
-----END CERTIFICATE-----
```
## 配置nifi
先将对应机器的 keystore.jks   truststore.jks 复制到nifi/config文件夹中
```shell script
cd ~/certs/s61
cp *.jks ~/nifi-1.11.4/conf/
cd ~/nifi-1.11.4/conf/
```
然后进入到nifi主目录conf文件夹，修改nifi.properties文件,以下为完整配置文件例子，这里只需要更改下面3个配置就可以了，对应的密码在s61，s62文件夹的config.json 中
- nifi.security.keystorePasswd=GfjOfKDsw81PuiyWiVwblmtSlcYOxlIpmWCgCw/8bks
- nifi.security.keyPasswd=GfjOfKDsw81PuiyWiVwblmtSlcYOxlIpmWCgCw/8bks
- nifi.security.truststorePasswd=bUswQMZLbsDTz5BLklwiQJWzlIt+bYv/TD0eUvybMfY
修改完成后开始配置用户角色权限

```properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Core Properties #
nifi.flow.configuration.file=./conf/flow.xml.gz
nifi.flow.configuration.archive.enabled=true
nifi.flow.configuration.archive.dir=./conf/archive/
nifi.flow.configuration.archive.max.time=30 days
nifi.flow.configuration.archive.max.storage=500 MB
nifi.flow.configuration.archive.max.count=
nifi.flowcontroller.autoResumeState=true
nifi.flowcontroller.graceful.shutdown.period=10 sec
nifi.flowservice.writedelay.interval=500 ms
nifi.administrative.yield.duration=30 sec
# If a component has no work to do (is "bored"), how long should we wait before checking again for work?
nifi.bored.yield.duration=10 millis
nifi.queue.backpressure.count=10000
nifi.queue.backpressure.size=1 GB

nifi.authorizer.configuration.file=./conf/authorizers.xml
nifi.login.identity.provider.configuration.file=./conf/login-identity-providers.xml
nifi.templates.directory=./conf/templates
nifi.ui.banner.text=
nifi.ui.autorefresh.interval=30 sec
nifi.nar.library.directory=./lib
nifi.nar.library.autoload.directory=./extensions
nifi.nar.working.directory=./work/nar/
nifi.documentation.working.directory=./work/docs/components

####################
# State Management #
####################
nifi.state.management.configuration.file=./conf/state-management.xml
# The ID of the local state provider
nifi.state.management.provider.local=local-provider
# The ID of the cluster-wide state provider. This will be ignored if NiFi is not clustered but must be populated if running in a cluster.
nifi.state.management.provider.cluster=zk-provider
# Specifies whether or not this instance of NiFi should run an embedded ZooKeeper server
nifi.state.management.embedded.zookeeper.start=false
# Properties file that provides the ZooKeeper properties to use if <nifi.state.management.embedded.zookeeper.start> is set to true
nifi.state.management.embedded.zookeeper.properties=./conf/zookeeper.properties


# H2 Settings
nifi.database.directory=./database_repository
nifi.h2.url.append=;LOCK_TIMEOUT=25000;WRITE_DELAY=0;AUTO_SERVER=FALSE

# FlowFile Repository
nifi.flowfile.repository.implementation=org.apache.nifi.controller.repository.WriteAheadFlowFileRepository
nifi.flowfile.repository.wal.implementation=org.apache.nifi.wali.SequentialAccessWriteAheadLog
nifi.flowfile.repository.directory=./flowfile_repository
nifi.flowfile.repository.partitions=256
nifi.flowfile.repository.checkpoint.interval=2 mins
nifi.flowfile.repository.always.sync=false
nifi.flowfile.repository.encryption.key.provider.implementation=
nifi.flowfile.repository.encryption.key.provider.location=
nifi.flowfile.repository.encryption.key.id=
nifi.flowfile.repository.encryption.key=

nifi.swap.manager.implementation=org.apache.nifi.controller.FileSystemSwapManager
nifi.queue.swap.threshold=20000
nifi.swap.in.period=5 sec
nifi.swap.in.threads=1
nifi.swap.out.period=5 sec
nifi.swap.out.threads=4

# Content Repository
nifi.content.repository.implementation=org.apache.nifi.controller.repository.FileSystemRepository
nifi.content.claim.max.appendable.size=1 MB
nifi.content.claim.max.flow.files=100
nifi.content.repository.directory.default=./content_repository
nifi.content.repository.archive.max.retention.period=12 hours
nifi.content.repository.archive.max.usage.percentage=50%
nifi.content.repository.archive.enabled=true
nifi.content.repository.always.sync=false
nifi.content.viewer.url=../nifi-content-viewer/
nifi.content.repository.encryption.key.provider.implementation=
nifi.content.repository.encryption.key.provider.location=
nifi.content.repository.encryption.key.id=
nifi.content.repository.encryption.key=

# Provenance Repository Properties
nifi.provenance.repository.implementation=org.apache.nifi.provenance.WriteAheadProvenanceRepository
nifi.provenance.repository.debug.frequency=1_000_000
nifi.provenance.repository.encryption.key.provider.implementation=
nifi.provenance.repository.encryption.key.provider.location=
nifi.provenance.repository.encryption.key.id=
nifi.provenance.repository.encryption.key=

# Persistent Provenance Repository Properties
nifi.provenance.repository.directory.default=./provenance_repository
nifi.provenance.repository.max.storage.time=24 hours
nifi.provenance.repository.max.storage.size=1 GB
nifi.provenance.repository.rollover.time=30 secs
nifi.provenance.repository.rollover.size=100 MB
nifi.provenance.repository.query.threads=2
nifi.provenance.repository.index.threads=2
nifi.provenance.repository.compress.on.rollover=true
nifi.provenance.repository.always.sync=false
# Comma-separated list of fields. Fields that are not indexed will not be searchable. Valid fields are:
# EventType, FlowFileUUID, Filename, TransitURI, ProcessorID, AlternateIdentifierURI, Relationship, Details
nifi.provenance.repository.indexed.fields=EventType, FlowFileUUID, Filename, ProcessorID, Relationship
# FlowFile Attributes that should be indexed and made searchable.  Some examples to consider are filename, uuid, mime.type
nifi.provenance.repository.indexed.attributes=
# Large values for the shard size will result in more Java heap usage when searching the Provenance Repository
# but should provide better performance
nifi.provenance.repository.index.shard.size=500 MB
# Indicates the maximum length that a FlowFile attribute can be when retrieving a Provenance Event from
# the repository. If the length of any attribute exceeds this value, it will be truncated when the event is retrieved.
nifi.provenance.repository.max.attribute.length=65536
nifi.provenance.repository.concurrent.merge.threads=2


# Volatile Provenance Respository Properties
nifi.provenance.repository.buffer.size=100000

# Component Status Repository
nifi.components.status.repository.implementation=org.apache.nifi.controller.status.history.VolatileComponentStatusRepository
nifi.components.status.repository.buffer.size=1440
nifi.components.status.snapshot.frequency=1 min

# Site to Site properties
nifi.remote.input.host=此处修改成本机域名,例如s61.nifiserver.com（本机host必须为0.0.0.0）
nifi.remote.input.secure=true
nifi.remote.input.socket.port=10443
nifi.remote.input.http.enabled=true
nifi.remote.input.http.transaction.ttl=30 sec
nifi.remote.contents.cache.expiration=30 secs

# web properties #
nifi.web.war.directory=./lib
nifi.web.http.host=
nifi.web.http.port=
nifi.web.http.network.interface.default=
nifi.web.https.host=此处修改成本机域名,例如s61.nifiserver.com（本机host必须为0.0.0.0）
nifi.web.https.port=9443
nifi.web.https.network.interface.default=
nifi.web.jetty.working.directory=./work/jetty
nifi.web.jetty.threads=200
nifi.web.max.header.size=16 KB
nifi.web.proxy.context.path=
nifi.web.proxy.host=

# security properties #
nifi.sensitive.props.key=
nifi.sensitive.props.key.protected=
nifi.sensitive.props.algorithm=PBEWITHMD5AND256BITAES-CBC-OPENSSL
nifi.sensitive.props.provider=BC
nifi.sensitive.props.additional.keys=

nifi.security.keystore=./conf/keystore.jks
nifi.security.keystoreType=jks
nifi.security.keystorePasswd=GfjOfKDsw81PuiyWiVwblmtSlcYOxlIpmWCgCw/8bks
nifi.security.keyPasswd=GfjOfKDsw81PuiyWiVwblmtSlcYOxlIpmWCgCw/8bks
nifi.security.truststore=./conf/truststore.jks
nifi.security.truststoreType=jks
nifi.security.truststorePasswd=bUswQMZLbsDTz5BLklwiQJWzlIt+bYv/TD0eUvybMfY
nifi.security.user.authorizer=managed-authorizer
nifi.security.user.login.identity.provider=
nifi.security.ocsp.responder.url=
nifi.security.ocsp.responder.certificate=

# OpenId Connect SSO Properties #
nifi.security.user.oidc.discovery.url=
nifi.security.user.oidc.connect.timeout=5 secs
nifi.security.user.oidc.read.timeout=5 secs
nifi.security.user.oidc.client.id=
nifi.security.user.oidc.client.secret=
nifi.security.user.oidc.preferred.jwsalgorithm=
nifi.security.user.oidc.additional.scopes=
nifi.security.user.oidc.claim.identifying.user=

# Apache Knox SSO Properties #
nifi.security.user.knox.url=
nifi.security.user.knox.publicKey=
nifi.security.user.knox.cookieName=hadoop-jwt
nifi.security.user.knox.audiences=

# Identity Mapping Properties #
# These properties allow normalizing user identities such that identities coming from different identity providers
# (certificates, LDAP, Kerberos) can be treated the same internally in NiFi. The following example demonstrates normalizing
# DNs from certificates and principals from Kerberos into a common identity string:
#
# nifi.security.identity.mapping.pattern.dn=^CN=(.*?), OU=(.*?), O=(.*?), L=(.*?), ST=(.*?), C=(.*?)$
# nifi.security.identity.mapping.value.dn=$1@$2
# nifi.security.identity.mapping.transform.dn=NONE
# nifi.security.identity.mapping.pattern.kerb=^(.*?)/instance@(.*?)$
# nifi.security.identity.mapping.value.kerb=$1@$2
# nifi.security.identity.mapping.transform.kerb=UPPER

# Group Mapping Properties #
# These properties allow normalizing group names coming from external sources like LDAP. The following example
# lowercases any group name.
#
# nifi.security.group.mapping.pattern.anygroup=^(.*)$
# nifi.security.group.mapping.value.anygroup=$1
# nifi.security.group.mapping.transform.anygroup=LOWER

# cluster common properties (all nodes must have same values) #
nifi.cluster.protocol.heartbeat.interval=5 sec
nifi.cluster.protocol.is.secure=true

# cluster node properties (only configure for cluster nodes) #
nifi.cluster.is.node=false
nifi.cluster.node.address=此处修改成本机域名,例如s61.nifiserver.com（本机host必须为0.0.0.0）
nifi.cluster.node.protocol.port=11443
nifi.cluster.node.protocol.threads=10
nifi.cluster.node.protocol.max.threads=50
nifi.cluster.node.event.history.size=25
nifi.cluster.node.connection.timeout=5 sec
nifi.cluster.node.read.timeout=5 sec
nifi.cluster.node.max.concurrent.requests=100
nifi.cluster.firewall.file=
nifi.cluster.flow.election.max.wait.time=5 mins
nifi.cluster.flow.election.max.candidates=

# cluster load balancing properties #
nifi.cluster.load.balance.host=
nifi.cluster.load.balance.port=6342
nifi.cluster.load.balance.connections.per.node=4
nifi.cluster.load.balance.max.thread.count=8
nifi.cluster.load.balance.comms.timeout=30 sec

# zookeeper properties, used for cluster management #
nifi.zookeeper.connect.string=
nifi.zookeeper.connect.timeout=3 secs
nifi.zookeeper.session.timeout=3 secs
nifi.zookeeper.root.node=/nifi

# Zookeeper properties for the authentication scheme used when creating acls on znodes used for cluster management
# Values supported for nifi.zookeeper.auth.type are "default", which will apply world/anyone rights on znodes
# and "sasl" which will give rights to the sasl/kerberos identity used to authenticate the nifi node
# The identity is determined using the value in nifi.kerberos.service.principal and the removeHostFromPrincipal
# and removeRealmFromPrincipal values (which should align with the kerberos.removeHostFromPrincipal and kerberos.removeRealmFromPrincipal
# values configured on the zookeeper server).
nifi.zookeeper.auth.type=
nifi.zookeeper.kerberos.removeHostFromPrincipal=
nifi.zookeeper.kerberos.removeRealmFromPrincipal=

# kerberos #
nifi.kerberos.krb5.file=

# kerberos service principal #
nifi.kerberos.service.principal=
nifi.kerberos.service.keytab.location=

# kerberos spnego principal #
nifi.kerberos.spnego.principal=
nifi.kerberos.spnego.keytab.location=
nifi.kerberos.spnego.authentication.expiration=12 hours

# external properties files for variable registry
# supports a comma delimited list of file locations
nifi.variable.registry.properties=

# analytics properties #
nifi.analytics.predict.enabled=false
nifi.analytics.predict.interval=3 mins
nifi.analytics.query.interval=5 mins
nifi.analytics.connection.model.implementation=org.apache.nifi.controller.status.analytics.models.OrdinaryLeastSquares
nifi.analytics.connection.model.score.name=rSquared
nifi.analytics.connection.model.score.threshold=.90
```
## 配置nifi用户权限
修改 ~/nifi-1.11.4/conf/authorizers.xml 文件配置用户权限
```
vi authorizers.xml 
```
搜索并修改如下配置
```xml
 <userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">./conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>
        <!--指定初始管理员用户-->
        <property name="Initial User Identity 1">CN=browseruser, OU=NIFI</property>
    </userGroupProvider>

    <accessPolicyProvider>
        <identifier>file-access-policy-provider</identifier>
        <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
        <property name="User Group Provider">file-user-group-provider</property>
        <property name="Authorizations File">./conf/authorizations.xml</property>
        <!--指定初始管理员用户-->
        <property name="Initial Admin Identity">CN=browseruser, OU=NIFI</property>
        <property name="Legacy Authorized Users File"></property>
        <property name="Node Identity 1"></property>
        <property name="Node Group"></property>
    </accessPolicyProvider>
```
修改完成后执行退到nifi根目录执行 ~/nifi-1.11.4/bin/nifi.sh start 启动nifi

将前面生成的browser文件夹中的p12证书导入到自己电脑的浏览器中，密码填browser目录中的config.json中的keyStorePassword就是p12证书的密码.
导入完成后在页面中进行权限配置。至此，简单的s2s配置就完成了。

## 配置nginx
在http块中配置如下内容
```text
server {
        listen 443 ssl;
        server_name p61.nifiserver.com;
        ssl_certificate /root/certs/p61/nifiserver.pem;
        ssl_certificate_key /root/certs/p61/nifiserver.key;
        ssl_client_certificate /root/certs/p61/nifi-cert.pem;
        ssl_verify_client on;

        proxy_ssl_certificate /root/certs/p61/nifiserver.pem;
        proxy_ssl_certificate_key /root/certs/p61/nifiserver.key;
        proxy_ssl_trusted_certificate /root/certs/p61/nifi-cert.pem;

        location / {
            proxy_pass https://s61.nifiserver.com:9443;
            proxy_set_header X-ProxyScheme https;
            proxy_set_header Origin https://s61.nifiserver.com:9443;
            proxy_set_header X-ProxyHost p61.nifiserver.com;
            proxy_set_header X-ProxyPort 443;
            proxy_set_header X-ProxyContextPath /;
            proxy_set_header X-ProxiedEntitiesChain $ssl_client_s_dn;
        }
}
server {
        listen 443 ssl;
        server_name p62.nifiserver.com;
        ssl_certificate /root/certs/p62/nifiserver.pem;
        ssl_certificate_key /root/certs/p62/nifiserver.key;
        ssl_client_certificate /root/certs/p62/nifi-cert.pem;
        ssl_verify_client on;

        proxy_ssl_certificate /root/certs/p62/nifiserver.pem;
        proxy_ssl_certificate_key /root/certs/p62/nifiserver.key;
        proxy_ssl_trusted_certificate /root/certs/p62/nifi-cert.pem;

        location / {
            proxy_pass https://s62.nifiserver.com:9443;
            proxy_set_header X-ProxyScheme https;
            # 设置origin主要是为了解决调用nifi-api的跨域问题
            proxy_set_header Origin https://s62.nifiserver.com:9443;
            proxy_set_header X-ProxyHost p62.nifiserver.com;
            proxy_set_header X-ProxyPort 443;
            proxy_set_header X-ProxyContextPath /;
            proxy_set_header X-ProxiedEntitiesChain $ssl_client_s_dn;
        }
}
```
在stream块中配置如下内容,用于转发s2s raw模式的请求
```text
server {
   listen 7443;
   proxy_pass s61.nifiserver.com:10443;
}

server {
   listen 8443;
   proxy_pass s62.nifiserver.com:10443;
}
```
配置nifiproperties配置路由信息,注意这里的proxyhost地址是nginx对应服务的域名与ip，填错了就路由不到这个地址了。这个路由的功能是把原始地址改为nginx的代理地址（就是前面配置的nginxstream），最终由nginx转发到真是的nifi服务端口上。
```properties
# S2S Routing for HTTP
nifi.remote.route.http.nginx1.when=${X-ProxyHost:contains('.nifiserver.com')}
nifi.remote.route.http.nginx1.hostname=p61.nifiserver.com
nifi.remote.route.http.nginx1.port=443
nifi.remote.route.http.nginx1.secure=true

nifi.remote.route.raw.nginx.when=\
${X-ProxyHost:equals('p61.nifiserver.com'):or(\
${s2s.source.hostname:equals('p61.nifiserver.com'):or(\
${s2s.source.hostname:equals('192.168.2.60')})})}
nifi.remote.route.raw.nginx.hostname=p61.nifiserver.com
nifi.remote.route.raw.nginx.port=7443
nifi.remote.route.raw.nginx.secure=true
```

## 创建测试任务


## 部署总结