---
layout: post
title: 对 SSL 密钥和证书的通俗理解
---

干计算机行业，SSL 变得有点绕不开。（其实是因为架 Tunnel 和 VPN 啊）
HTTPS 是最开始接触 SSL 的地方，后来架设 VPN 要 SSL（L2TP、IPSec、IKEv2、SSTP、OpenVPN 都是以 SSL 为基础的），然后发现各种服务都会有 SSL 版。
跟 SSL 有一定功能重合的 SSH 也跟 SSL 非常相似，OpenSSH 基于 OpenSSL，导致我一度以为 SSH 是基于 SSL 的，直到后来写个 SSH 客户端，才知道这俩只是。。。“比较重合”。

之前架设涉及 SSL 的服务，经常发现不知道自己在干嘛。在网上找教程一步一步照着操作完成的，不知道原理。
最近几天。。。理顺了一下，可能跟实际有误差。

加密原理
----

### 非对称加密
这个当然是整套机制的基础，私钥公钥，用一个加密，用另一个就能解密。不过有个东西容易引起歧义：公钥并不是一定是公开的，很多情况下，公钥只是说可以明码传输的。非对称加密的性质，导致别人就算获取公钥，也无法解密公钥加密的内容。
### 非对称加密通讯
`A <-> B` 通讯
1、假设 B 有了 A 的公钥，那么 B 就可以向 A 发送加密内容，无法解密。
（但是别人如果截获了公钥，依然可以解密 A 发送给 B 的内容，所以实际上，A 基本不再发送什么重要内容了。）
2、B 很开心的给 A 发送了 B 的公钥。
（使用 A 的公钥加密传输，只有 A 能解析出 B 的公钥。）
这样一来，使用 B 的密钥对，就可以实现加密传输了，其他人谁都无法解密。
### 对称加密通讯和 KEY 交换
假如我想使用对称加密算法咋办，比如 AES，两者需要一个公共密钥，那就需要 KEY 交换。
费马小定理能得出一个奇怪的公式：已知两个数 C 和 P，P 是大素数（或者大素数的倍数），而 C 通常是 2。
`(C ^ X MOD P) ^ Y MOD P == (C ^ Y MOD P) ^ X MOD P ==> KEY`
那对 A 来说，拿着 X，把 `(C ^ X MOD P)` 给 B，B 也把 `(C ^ Y MOD P)` 给 A，两人就都拿到 KEY 了。
然后两个就可以愉快地使用对称加密算法如 AES 加密了。
因为 P 是素数，就算别人知道 C 和 P，也非常难从 `(C ^ X MOD P)` 里弄出 X 来，更别提 KEY 了。但费马小定理又能实现出一个线性复杂度的 PowMod 算法，所以加密成本很低，破解成本很高。
关于 C 为什么通常是 2，我觉得也是计算成本考虑，2^E，只需要位移操作。


证书原理
----

### 证书内容
一个证书一般至少有公钥和身份信息。证书是别人授权的，则还有授权人信息和签名。签名是由授权人私钥加密，只有用授权人的公钥才能解密。
### 证书授权人（CA）
证书授权人的证书通常预装在操作系统中，认为是可信的。
### 服务器如何证明自己
通常需要同时出示两份证书服务器的证书和授权人的证书。（有时候，系统会自动去寻找授权人证书，另当别论）
客户端验证过程大概如下：
1、授权人的证书跟自己预装的授权人证书比较，如果找到一样的，则证明“授权人证书”是真的。
2、用 “授权人证书” 的公钥去解密被签名的部分，解密成功了，说明签名是真的。
3、签名通常包含了“服务器证书”的 HASH 值，如果一致，说明 “服务器证书” 是被授权人认真过的。
4、服务器证书里有一些身份信息，比如域名，如果跟请求的一致，说明服务器是真的。
5、利用服务器证书里的公钥，跟服务器进一步通讯。
整个逻辑只有一个潜在问题：证书不是唯一的，比如一个域名，可以有多个证书，别人申请一个假的咋办。
目前 “根证书授权人” 会通过一些方式审核申请人，比如域名证书，会要求邮件验证，邮箱必须跟域名注册信息一致。
### 如何成为“证书授权人”
“证书授权人” 可以分为 “根证书授权人” 和 “中间证书授权人”。
“根证书授权人” 只有公认的那么几家，有的 “根证书授权人” 只出现在部分操作系统里。
企业来说，如果服务对象是自己的员工，可以要求员工在电脑上预装证书，也可以成为 “中间证书授权人”，“中间证书授权人” 理论上可以无数个。
证书链就是这么运作的。
### 自签名证书
自签名证书可以认为就是没人证明的证书。要跟服务器通信，要么客户端预装自签名证书，要么客户端无视证书错误（容易遭到中间人攻击）。
### 证书信息
证书里一些东西，比如国家、省、组织名称、通用名称，在不同证书里不同要求，很多是不能留空的，有些是必须写得跟授权人一致。
通用名称是最常用的，比如域名证书，域名就写在通用名称里面。

常见操作
----
以 OPENSSL 为例
windows 下建议先执行
	set RANDFILE=.rnd
### 生成私钥公钥对（自签名证书）
	openssl req -new -x509 -nodes -out crt.pem -keyout key.pem
生成一个 X509 格式的私钥，证书（公钥）为 crt.pem，私钥为 key.pem。
注意 crt.pem 如果后缀改为 .crt，是可以在 Windows 上直接识别安装的。
crt.pem 和 key.pem 可以配置在 NGINX、SQUID 等服务上，crt.pem 会被直接传给客户端。
注意，虽然 SSL 支持各种算法和格式，但是最好按照常理出牌。。。。
### 生成私钥
	openssl genrsa -out key.pem 2048
生成 2048 位 RSA 私钥。目前比较常见的是 1204 位和 2048 位，2048 位用的越来越多。Google、OpenSSH 默认都 2048 了。
### 利用私钥生成证书（公钥）
	openssl req -new -x509 -key key.pem -out ca.crt
### 用 CA 密钥给别人的证书签名
生成证书请求
	openssl req -new -key server.key -out server.csr
签名
	openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key
server.crt 就是签名后的证书，包含了签名和证书。
证书链
	cat ca.crt >> server.crt
真的只要 cat 就可以。

常见应用场景
----
在 SSL 连接建立后，理论上任何 SOCKET 信息都可以通过 SSL 传输，跟 TCP 更是基本没差异，所以老协议移植性很高。

### 已有服务的 SSL 化
比如 HTTPS，SMTP on SSL，FTP on SSL。
注意，有些是跟 SSH 的相关服务容易搞混的，比如 FTP on SSL 和 SFTP。
### 新的基于 SSL 的服务
加密现在基本处于硬需了，貌似现在是个协议都支持 SSL。
SPDY、各种 VPN。
### 各种软件的加密通讯
。。。
