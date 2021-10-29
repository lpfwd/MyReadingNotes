# Cisco Public Key Infra Configuration
## 基本概念
关于public key, private key, CA, 证书，以及数字签名的基础知识可以参考 [阮一峰老师的文章](https://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)

## Cisco Public key 配置
### public key 生成
看配置手册，其中有几个要点

* 一台Router上面可以生成多个RSA key pair.
* key pair需要export/import, 这是为了在HA的情况下，active能同步private key到standby, 当做主备切换的时候，standby不用再去生成key pair.
* import/export的key需要有密码保护

CLI commands to Generate keypair:

```
crypto key generate rsa [general-keys | usage-keys | signature | encryption] [label key-label] [exportable] [modulus modulus-size] [storage devicename:] [on devicename:]
```
### 配置自签名证书
CLI example:

```
Router> enable
Router# configure terminal
Router(config)#crypto pki trustpoint TESTCA
Router(ca-trustpoint)#hash sha256
Router(ca-trustpoint)#rsakeypair testca-rsa-key 2048
Router(ca-trustpoint)#exit
Router(config)#crypto pki enroll TESTCA
% Include the router serial number in the subject name? [yes/no]:no
% Include an IP address in the subject name? [no]: no
Generate Self Signed Router Certificate? [yes/no]: yes

Router Self Signed Certificate successfully created

Router(config)#
Router(config)#exit
Router#
```

TBD (How to execure certificate request on Cisco router)

