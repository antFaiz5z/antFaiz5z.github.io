---
title: Deepin下安装vmware无法联网的问题
comments: true
toc: true
date: 2017-08-28 11:47:32
categories: vmware
tags: 
    - vmware
    - deepin
---

### 问题描述

```bash
Could not connect 'Ethernet0' to virtual network '/dev/vmnet0'.
```

### 解决办法

1. 使用sudo vmware-networks --start查看是否能够启动网络

    ```bash
    $ sudo vmware-networks --start
    Started Bridge networking on vmnet0
    Failed to enable hostonly virtual adapter on vmnet1
    Failed to start DHCP service on vmnet1
    Failed to start NAT service on vmnet8
    Failed to enable hostonly virtual adapter on vmnet8
    Failed to start DHCP service on vmnet8
    Failed to start some/all services
    ```

2. 使用命令sudo modprobe vmnet

    ```bash
    $ sudo modprobe vmnet
    ```

3. 重新启动网络成功

    ```bash
    $ sudo vmware-networks --start
    Started Bridge networking on vmnet0
    Enabled hostonly virtual adapter on vmnet1
    Started DHCP service on vmnet1
    Started NAT service on vmnet8
    Enabled hostonly virtual adapter on vmnet8
    Started DHCP service on vmnet8
    Started all configured services on all networks
    ```

4. VMware成功启动