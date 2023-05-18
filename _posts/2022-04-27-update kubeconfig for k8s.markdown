---
layout: post
title:  "更新kubeconfig证书和地址"
date:   2022-04-27 23:33:27 +0800
categories: kubernetes
---
1. 删除`/etc/kubernetes/pki/`下的api-server证书

    ```bash
    sudo rm -rf /etc/kubernetes/pki/apiserver.*
    ```
2. 为api-server重新生成证书

    ```bash
    sudo kubeadm init phase certs apiserver --apiserver-advertise-address [api-server监听地址] --apiserver-cert-extra-sans [外网监听地址]
    ```

3. 刷新`admin.conf`

    ```bash
    sudo kubeadm certs renew /etc/kubernetes/admin.conf
    ```

之后就可以使用新的`admin.conf`进行操作了