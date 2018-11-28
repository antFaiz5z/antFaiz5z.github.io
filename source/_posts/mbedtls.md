---
title: mbedTLS (PolarSSL) 的几点问题
comments: true
toc: false
date: 2018-07-20 15:54:55
categories: C/C++
tags: mbedTLS
---

1. 

```c
mbedtls_ssl_conf_authmode( &conf, MBEDTLS_SSL_VERIFY_REQUIRED );
```

第二个参数不建议用MBEDTLS_SSL_VERIFY_OPTIONAL，不然验证通不过的时候也能用，意义不大

2. 

```c
mbedtls_ssl_set_hostname( &ssl, "MQTT" )
```

第二个参数一定要和生成证书时所指定的CN字段一样

3. 

```c
mbedtls_ssl_conf_max_version(&conf, MBEDTLS_SSL_MAJOR_VERSION_3, MBEDTLS_SSL_MINOR_VERSION_1);
mbedtls_ssl_conf_min_version(&conf, MBEDTLS_SSL_MAJOR_VERSION_3, MBEDTLS_SSL_MINOR_VERSION_1);
```

这两条语句有可能会用到

4. 
    
实现收发数据的接口时一定要注意采用阻塞方式还是超时方式，不然可能会死锁