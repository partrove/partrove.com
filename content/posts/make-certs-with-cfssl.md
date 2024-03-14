+++
title = '使用CFSSL签发TLS证书'
date = 2024-03-14T14:00:19+08:00
draft = true
isCJKLanguage = true
+++

#Security #SSL #CFSSL

# 下载安装  

参考[官方文档](https://github.com/cloudflare/cfssl)，如果主机上装了 [Go](https://go.dev/dl/)，可以直接通过 go 命令安装  

```shell
go install -v github.com/cloudflare/cfssl/cmd/...@latest
```  

同时还可以直接下载已编译好的二进制，[下载地址](https://github.com/cloudflare/cfssl/releases)  

CFSSL 是一整套工具集，分为好几个工具。如果单纯只是想签发证书的话，只需要下面两个  

- [cfssl](https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64)  
- [cfssljson](https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64)  

# 使用方法  

CFSSL 使用 json 作为配置的格式，目前有两种配置  

- config：证书管理的配置文件，可以按场景（profiles）定义证书的属性，例如有效时长、作用等；
- csr：证书签名请求（CSR）配置，用来定义具体证书申请的属性，例如CN、主机列表、加密算法等；

CFSSL 提供了 `print-defaults` 子命令来生成上面两种配置  

```shell
# 打印默认 config 
$ cfssl print-defaults config
# 打印默认 csr
$ cfssl print-defaults csr
```  

因此使用 CFSSL 签发证书的一般流程为：  

- 先要有一份 CA 证书和密钥，可以是已有的，也可以用 CFSSL 生成；
- 准备签发配置 config.json；
- 再构造证书的 CSR 配置 csr.json；  
- 使用上面的三个东西，签发证书。  

# 示例  

通过一个例子来演示为 HTTPs 服务签发证书的全过程。  

## 自签 CA  

先准备 CA 的 CSR 配置，可以用上面提到的命令生成模板，然后再修改，  

```shell
$ cfssl print-defaults csr > ca-csr.json
```  

这里假设我们服务所在的域是 *partrove.com*，对应的服务运行在主机 *192.168.193.128* 上，那么 CA CSR 可以是下面这样，  

```json
{
    "CN": "partrove.com",
    "hosts": [
        "*.partrove.com",
        "192.168.193.128",
        "127.0.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "ca": {
        "expiry": "87600h"
    },
    "names": [
        {
            "C": "CN",
            "ST": "Guangdong",
            "L": "Shenzhen",
            "O": "ParTrove",
            "OU": "Technical Department"
        }
    ]
}
```  

配置中对应的字段稍微解释一下，  

- CN：通用名称，用于标识这整套证书的名称，在实际应用中服务端一般会用这个字段来做校验。例如 K8s 的 **RBAC** 鉴权体系中就拿这个字段当作用户名。而 HTTPs 的服务中这里一般写具体域名；  
- hosts：这个 CA 签发的证书，对应的主体清单，可以是 IP 或域名，也可以写泛域名。证书校验的时候，会对比实际主机是否在这个清单里面。这里如果不写的话，就表示这个 CA 签发的证书可以用于任何主机；  
- key：证书签发的加密算法及长度；
- ca.expiry：设置 CA 证书的有效期，如果没指定默认是 5 年；
- C：证书属性中的国家字段；
- ST：证书属性中的省份字段；
- L：证书属性中的城市字段；
- O：证书属性中的组织/公司字段；
- OU：证书属性中的单位/部门字段；

这里再放上两个用于 CSR 生成和解析的在线工具  

[CSR生成](https://www.sslceshi.com/csr_generate/)  

[CSR解析](https://myssl.com/csr_decode.html)  

下面开始签发 CA 证书，执行下面的命令  

```shell
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2024/03/09 17:30:24 [INFO] generating a new CA key and certificate from CSR
2024/03/09 17:30:24 [INFO] generate received request
2024/03/09 17:30:24 [INFO] received CSR
2024/03/09 17:30:24 [INFO] generating key: rsa-2048
2024/03/09 17:30:24 [INFO] encoded CSR
2024/03/09 17:30:24 [INFO] signed certificate with serial number 214182507133763088203751853032868253568017413678
```  

我们可以用 openssl 再来检查一下生成的 CA 证书  

```shell
$ openssl x509 -in ca.pem -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            25:84:46:99:f3:a4:b5:86:7b:e7:58:2c:8f:76:4a:54:17:44:b6:2e
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=Guangdong, L=Shenzhen, O=ParTrove, OU=Technical Department, CN=partrove.com
        Validity
            Not Before: Mar  9 09:25:00 2024 GMT
            Not After : Mar  7 09:25:00 2034 GMT
        Subject: C=CN, ST=Guangdong, L=Shenzhen, O=ParTrove, OU=Technical Department, CN=partrove.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b2:2e:2a:be:6a:f3:b9:65:bb:c3:a3:df:84:f3:
                    ce:f7:f0:fc:69:c6:8a:a9:94:7b:c4:0e:63:2a:62:
                    d5:71:dc:02:46:dc:b6:03:53:39:30:77:f5:ac:b1:
                    e0:09:3e:af:d5:15:ab:b9:1e:09:64:79:42:67:01:
                    d3:12:54:cd:2e:92:06:ab:0a:27:37:2d:72:fc:62:
                    ed:8d:77:36:57:72:bb:0e:8d:39:6d:a2:c3:dd:12:
                    93:fa:e3:21:6c:9e:cd:48:1b:b7:8b:f6:d3:81:d9:
                    ca:ec:60:56:62:3b:f8:82:6b:57:ef:fc:45:68:89:
                    45:41:16:80:a1:6f:f3:4b:b9:02:40:9a:34:50:f3:
                    46:b3:48:59:e9:26:04:af:8e:b5:ba:4b:50:34:6b:
                    91:a7:e4:d0:a1:c5:59:ab:01:90:14:6e:41:1f:37:
                    ea:3f:bf:f8:20:b2:97:df:b4:d7:80:98:74:87:ca:
                    d8:16:38:8a:f2:3c:1e:37:c9:74:63:e3:02:8e:3e:
                    8f:7b:28:82:20:0f:7f:92:75:66:1b:f7:64:70:81:
                    c0:ee:9c:c6:bd:11:67:60:8f:7b:9d:38:ff:fc:eb:
                    2e:3c:fe:1a:7e:8c:d4:26:76:9f:51:cb:2c:c1:62:
                    0c:d3:62:22:b9:5a:c8:19:c9:4e:a2:73:6b:ab:e6:
                    cd:4b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                26:C3:5C:22:C1:BA:C6:6F:CA:49:E1:8B:6E:23:43:39:53:BE:5F:DA
            X509v3 Subject Alternative Name:
                IP Address:192.168.193.128, IP Address:127.0.0.1
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        29:be:b2:a0:5d:9e:e9:94:10:16:e6:44:b8:04:cb:0d:ba:ba:
        69:0c:71:c2:6c:83:d7:f8:a4:93:0c:aa:9d:8c:5d:6f:d3:f7:
        0f:d5:cd:1c:64:21:13:8e:66:65:57:3c:44:8c:73:09:ee:52:
        1f:75:86:d7:ae:86:dd:73:fc:c1:0f:dd:c6:d7:07:08:7f:0f:
        41:13:a4:ce:08:e0:b7:68:5e:5c:a9:07:42:e4:f6:b6:73:12:
        9a:5f:92:e8:76:35:14:85:1a:5b:96:84:ab:59:e8:3b:49:5d:
        07:ef:83:26:7f:b2:72:40:b4:2c:dd:f8:44:b0:d9:fb:36:2c:
        5e:27:38:d7:d5:5f:1f:b2:96:8b:93:7b:86:0b:da:65:99:48:
        a4:68:b5:28:8e:19:4e:21:80:bb:3c:37:6d:f1:a5:c6:fd:03:
        f3:54:9d:dd:d4:9c:af:5f:06:2a:93:29:d6:65:ec:b6:f4:c3:
        4f:0d:80:5e:d3:ba:02:35:dc:1e:31:97:04:42:2f:6c:90:5c:
        7b:e1:0c:23:5a:2f:ff:2f:bd:0e:ef:9c:d0:1d:6d:ed:79:46:
        ac:8f:91:b8:69:30:ab:fe:d5:e3:b3:20:b5:cd:ae:98:52:9c:
        25:9f:72:01:9e:f1:45:22:21:96:35:f1:e5:93:28:81:77:21:
        a0:ef:f9:83
-----BEGIN CERTIFICATE-----
MIID4TCCAsmgAwIBAgIUJYRGmfOktYZ751gsj3ZKVBdEti4wDQYJKoZIhvcNAQEL
BQAwfTELMAkGA1UEBhMCQ04xEjAQBgNVBAgTCUd1YW5nZG9uZzERMA8GA1UEBxMI
U2hlbnpoZW4xETAPBgNVBAoTCFBhclRyb3ZlMR0wGwYDVQQLExRUZWNobmljYWwg
RGVwYXJ0bWVudDEVMBMGA1UEAxMMcGFydHJvdmUuY29tMB4XDTI0MDMwOTA5MjUw
MFoXDTM0MDMwNzA5MjUwMFowfTELMAkGA1UEBhMCQ04xEjAQBgNVBAgTCUd1YW5n
ZG9uZzERMA8GA1UEBxMIU2hlbnpoZW4xETAPBgNVBAoTCFBhclRyb3ZlMR0wGwYD
VQQLExRUZWNobmljYWwgRGVwYXJ0bWVudDEVMBMGA1UEAxMMcGFydHJvdmUuY29t
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsi4qvmrzuWW7w6PfhPPO
9/D8acaKqZR7xA5jKmLVcdwCRty2A1M5MHf1rLHgCT6v1RWruR4JZHlCZwHTElTN
LpIGqwonNy1y/GLtjXc2V3K7Do05baLD3RKT+uMhbJ7NSBu3i/bTgdnK7GBWYjv4
gmtX7/xFaIlFQRaAoW/zS7kCQJo0UPNGs0hZ6SYEr461uktQNGuRp+TQocVZqwGQ
FG5BHzfqP7/4ILKX37TXgJh0h8rYFjiK8jweN8l0Y+MCjj6PeyiCIA9/knVmG/dk
cIHA7pzGvRFnYI97nTj//OsuPP4afozUJnafUcsswWIM02IiuVrIGclOonNrq+bN
SwIDAQABo1kwVzAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNV
HQ4EFgQUJsNcIsG6xm/KSeGLbiNDOVO+X9owFQYDVR0RBA4wDIcEwKjBgIcEfwAA
ATANBgkqhkiG9w0BAQsFAAOCAQEAKb6yoF2e6ZQQFuZEuATLDbq6aQxxwmyD1/ik
kwyqnYxdb9P3D9XNHGQhE45mZVc8RIxzCe5SH3WG166G3XP8wQ/dxtcHCH8PQROk
zgjgt2heXKkHQuT2tnMSml+S6HY1FIUaW5aEq1noO0ldB++DJn+yckC0LN34RLDZ
+zYsXic419VfH7KWi5N7hgvaZZlIpGi1KI4ZTiGAuzw3bfGlxv0D81Sd3dScr18G
KpMp1mXstvTDTw2AXtO6AjXcHjGXBEIvbJBce+EMI1ov/y+9Du+c0B1t7XlGrI+R
uGkwq/7V47Mgtc2umFKcJZ9yAZ7xRSIhljXx5ZMogXchoO/5gw==
-----END CERTIFICATE-----
```  

## 签发服务证书  

上面我们已经签发了 CA 证书，同时也配置了这个 CA 是用于整个 *partrove.com* 域的，那么我们先给这一套证书编写一份管理配置。同样的，还是可以用命令生成模板，  

```shell
$ cfssl print-defaults config > partrove.com-config.json
```  

对应的配置可以是下面这样，  

```json
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "server": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "intermediate": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "cert sign",
                    "crl sign"
                ],
                "ca_constraint": {
                    "is_ca": true,
                    "max_path_len": 0,
                    "max_path_len_zero": true
                }
            }
        }
    }
}
```  

同样解释下几个字段，  

- default.expiry：默认的证书有效时间，如果下面的 profiles 里面没有单独配置的话，就用这个；
- profiles：配置集；
- usages：证书的作用，比如服务端证书可以进行服务端验证，就设置 `server auth`  

关于证书的 usages，可以参考这里  

[[X.509证书内容解释]]

然后再准备签发证书的 CSR 配置，这里假设我们的服务域名是 *h2.partrove.com*，对应的 CSR 配置如下  

```json
{
    "CN": "h2.partrove.com",
    "hosts": [
        "h2.partrove.com",
        "192.168.193.128",
        "127.0.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Guangdong",
            "L": "Shenzhen",
            "O": "ParTrove",
            "OU": "Technical Department"
        }
    ]
}
```  

跟 [[使用CFSSL签发证书#自签 CA|CA CSR]] 一样，这里就不过多解释了。  

然后执行命令来签发证书，  

```shell
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=partrove.com-config.json -profile=server partrove.com-csr.json | cfssljson -bare h2
2024/03/09 17:56:13 [INFO] generate received request
2024/03/09 17:56:13 [INFO] received CSR
2024/03/09 17:56:13 [INFO] generating key: rsa-2048
2024/03/09 17:56:13 [INFO] encoded CSR
2024/03/09 17:56:13 [INFO] signed certificate with serial number 268191540815040687212966405235110068881085426696
```  

从命令的参数可以看数来，这里用到了前面签发出来的 CA 证书和 key，并且指定了用 `server` 这个配置。至此服务证书签发完成，此时我们的目录下面应该有下面这些文件  

```shell
$ ll
total 30
-rw-r--r-- 1 Feo Lee 197121 1131  3月  9 17:30 ca.csr
-rw-r--r-- 1 Feo Lee 197121 1407  3月  9 17:30 ca.pem
-rw-r--r-- 1 Feo Lee 197121  428  3月  9 17:28 ca-csr.json
-rw-r--r-- 1 Feo Lee 197121 1679  3月  9 17:30 ca-key.pem
-rw-r--r-- 1 Feo Lee 197121 1115  3月  9 17:56 h2.csr
-rw-r--r-- 1 Feo Lee 197121 1505  3月  9 17:56 h2.pem
-rw-r--r-- 1 Feo Lee 197121 1679  3月  9 17:56 h2-key.pem
-rw-r--r-- 1 Feo Lee 197121 1010  3月  9 17:45 partrove.com-config.json
-rw-r--r-- 1 Feo Lee 197121  388  3月  9 17:53 partrove.com-csr.json
 ```  

让我们再来检查一下 *h2.partrove.com* 的服务端证书，  

```shell
$ openssl x509 -in h2.pem -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            2e:fa:1f:e6:ea:7f:d7:2c:22:1c:94:a7:ff:35:4e:43:af:d7:ec:08
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=Guangdong, L=Shenzhen, O=ParTrove, OU=Technical Department, CN=partrove.com
        Validity
            Not Before: Mar  9 09:51:00 2024 GMT
            Not After : Mar  7 09:51:00 2034 GMT
        Subject: C=CN, ST=Guangdong, L=Shenzhen, O=ParTrove, OU=Technical Department, CN=h2.partrove.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e5:7f:ce:b8:e1:1b:f0:ee:d1:5c:78:df:cd:3c:
                    f5:ee:de:eb:85:88:9b:9c:f9:a4:4e:80:8b:60:af:
                    2d:90:3e:b4:c3:28:a1:8f:e7:27:3b:ba:08:9e:85:
                    21:ba:00:18:98:58:5d:bf:bc:09:bb:eb:7f:b4:d1:
                    30:1f:fe:93:bd:bd:81:63:ba:db:f5:c0:5b:be:e0:
                    1c:4f:e7:07:c7:db:0e:ce:9f:00:3b:ec:76:a6:59:
                    c8:34:e9:66:19:68:6d:3e:88:7c:37:72:bb:1b:66:
                    6b:f5:4a:7f:a7:2c:49:bb:90:94:cc:cb:d9:73:30:
                    ad:3e:fc:9a:3b:40:21:a2:28:c8:f3:ca:a0:62:3a:
                    ed:c1:57:86:aa:83:7a:c6:93:9a:d6:05:e6:2a:6d:
                    11:f7:82:76:b1:af:36:07:ef:f7:fc:c0:e7:9d:b9:
                    a6:ab:81:92:c4:ca:f4:e7:84:ea:60:d1:b6:54:99:
                    41:c6:eb:e6:08:f6:27:93:4c:83:16:4d:da:22:32:
                    f2:53:3c:71:aa:4c:4b:c5:f2:e1:15:ec:74:e5:05:
                    38:9f:63:ec:6d:d7:98:a1:71:8e:6d:34:73:2b:0f:
                    be:97:55:84:9b:3b:58:0b:0a:6a:e6:e1:ec:53:7a:
                    e9:72:c4:6b:78:24:32:6e:3a:ae:a8:a9:69:a1:b7:
                    51:7d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                9A:42:A8:87:6B:23:39:9A:3C:67:23:D9:3C:14:8B:62:81:F9:99:2F
            X509v3 Authority Key Identifier:
                26:C3:5C:22:C1:BA:C6:6F:CA:49:E1:8B:6E:23:43:39:53:BE:5F:DA
            X509v3 Subject Alternative Name:
                DNS:h2.partrove.com, IP Address:192.168.193.128, IP Address:127.0.0.1
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        61:04:13:bf:52:c9:f3:97:45:0a:6f:45:6b:29:08:81:77:f7:
        a3:1d:ba:d6:2e:2c:31:39:37:d0:65:55:c4:d7:fd:7c:26:0f:
        ce:ed:b7:43:35:02:5b:88:24:bc:d7:6e:d6:40:90:55:25:f5:
        f4:11:61:30:7e:20:fb:43:22:40:b6:28:cb:16:14:78:62:bd:
        3b:ba:63:f9:03:32:46:ea:94:77:38:eb:c1:4b:aa:eb:81:79:
        b6:e3:c1:56:9d:df:0d:cb:f8:12:4a:11:5f:9e:33:9d:c7:12:
        d3:dc:e7:69:13:66:29:54:db:b8:5b:ca:2f:fd:0d:4b:9e:d6:
        71:5a:b3:40:b2:40:3c:b1:75:16:ec:e9:02:49:46:79:12:27:
        02:08:bc:9d:2d:36:a8:54:5e:01:8d:d5:b1:f3:2a:c9:48:fd:
        3e:54:5b:87:b9:f7:a6:86:07:96:89:3c:3e:52:2a:e6:ce:ad:
        b6:c4:99:11:37:8e:a9:48:48:5b:63:98:ec:bf:67:d6:92:ed:
        4f:80:f2:d4:9f:45:99:cc:b2:8e:a5:93:0c:20:3c:27:99:8a:
        74:35:82:6e:4f:19:41:e3:6b:ca:3d:77:e4:84:3f:13:64:c5:
        bf:fb:76:cb:eb:dd:04:f4:c7:3b:11:a8:19:3c:54:59:74:f6:
        ee:18:86:e4
-----BEGIN CERTIFICATE-----
MIIEKzCCAxOgAwIBAgIULvof5up/1ywiHJSn/zVOQ6/X7AgwDQYJKoZIhvcNAQEL
BQAwfTELMAkGA1UEBhMCQ04xEjAQBgNVBAgTCUd1YW5nZG9uZzERMA8GA1UEBxMI
U2hlbnpoZW4xETAPBgNVBAoTCFBhclRyb3ZlMR0wGwYDVQQLExRUZWNobmljYWwg
RGVwYXJ0bWVudDEVMBMGA1UEAxMMcGFydHJvdmUuY29tMB4XDTI0MDMwOTA5NTEw
MFoXDTM0MDMwNzA5NTEwMFowgYAxCzAJBgNVBAYTAkNOMRIwEAYDVQQIEwlHdWFu
Z2RvbmcxETAPBgNVBAcTCFNoZW56aGVuMREwDwYDVQQKEwhQYXJUcm92ZTEdMBsG
A1UECxMUVGVjaG5pY2FsIERlcGFydG1lbnQxGDAWBgNVBAMTD2gyLnBhcnRyb3Zl
LmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAOV/zrjhG/Du0Vx4
38089e7e64WIm5z5pE6Ai2CvLZA+tMMooY/nJzu6CJ6FIboAGJhYXb+8Cbvrf7TR
MB/+k729gWO62/XAW77gHE/nB8fbDs6fADvsdqZZyDTpZhlobT6IfDdyuxtma/VK
f6csSbuQlMzL2XMwrT78mjtAIaIoyPPKoGI67cFXhqqDesaTmtYF5iptEfeCdrGv
Ngfv9/zA5525pquBksTK9OeE6mDRtlSZQcbr5gj2J5NMgxZN2iIy8lM8capMS8Xy
4RXsdOUFOJ9j7G3XmKFxjm00cysPvpdVhJs7WAsKaubh7FN66XLEa3gkMm46rqip
aaG3UX0CAwEAAaOBnjCBmzAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYB
BQUHAwEwDAYDVR0TAQH/BAIwADAdBgNVHQ4EFgQUmkKoh2sjOZo8ZyPZPBSLYoH5
mS8wHwYDVR0jBBgwFoAUJsNcIsG6xm/KSeGLbiNDOVO+X9owJgYDVR0RBB8wHYIP
aDIucGFydHJvdmUuY29thwTAqMGAhwR/AAABMA0GCSqGSIb3DQEBCwUAA4IBAQBh
BBO/Usnzl0UKb0VrKQiBd/ejHbrWLiwxOTfQZVXE1/18Jg/O7bdDNQJbiCS8127W
QJBVJfX0EWEwfiD7QyJAtijLFhR4Yr07umP5AzJG6pR3OOvBS6rrgXm248FWnd8N
y/gSShFfnjOdxxLT3OdpE2YpVNu4W8ov/Q1LntZxWrNAskA8sXUW7OkCSUZ5EicC
CLydLTaoVF4BjdWx8yrJSP0+VFuHufemhgeWiTw+Uirmzq22xJkRN46pSEhbY5js
v2fWku1PgPLUn0WZzLKOpZMMIDwnmYp0NYJuTxlB42vKPXfkhD8TZMW/+3bL690E
9Mc7EagZPFRZdPbuGIbk
-----END CERTIFICATE-----
```