+++
title = 'k8s节点上证书缺失问题'
date = 2024-05-15
draft = false
tags = ['kubernetes', 'security']
categories = ['Infrastructure']
+++

最近遇到一堆node not ready都和证书有关，不知道为什么工作节点的私钥和证书就没有了，kubelet启动失败，现在研究下如何手动生成这些证书。

<!--more-->

首先是worker node和master node的通信是需要证书的，证书的目录在 `/etc/kubernetes/ssl/` 这个下面（这个目录是可以自己指定的，如果使用kubeadm的话不是这个目录），不知道为什么在这个目录里有时没有生成私钥和证书导致kubelet启动失败，以前一直通过重新provision node来解决，现在还是想研究下具体原因和对策。

首先是有这3个 `ca.pem, worker-key.pem, worker.pem` 的话，kubelet是能启动的，不会报找不到证书错误

尝试了将别的node上的这些给拷贝到有问题的node上，结果报错：
```
Mar 14 10:26:57 server2277z.prod.jp.local kubelet-wrapper[18698]: I0314 01:26:57.425346   18698 kubelet_node_status.go:82] Attempting to register node server2277z.prod.jp.local
Mar 14 10:26:57 server2277z.prod.jp.local kubelet-wrapper[18698]: E0314 01:26:57.428143   18698 kubelet_node_status.go:106] Unable to register node "server2277z.prod.jp.local" with API server: nodes "server2277z.prod.jp.local" is forbidden: node "server2267z.prod.jp.local" cannot modify node "server2277z.prod.jp.local"
Mar 14 10:26:57 server2277z.prod.jp.local kubelet-wrapper[18698]: E0314 01:26:57.691576   18698 mirror_client.go:88] Failed deleting a mirror pod "kube-proxy-server2277z.prod.jp.local_kube-system": pods "kube-proxy-server2277z.prod.jp.local" is forbidden: node "server2267z.prod.jp.local" can only delete pods with spec.nodeName set to itself
```
缺失证书的 node 被识别为其他node 😅，不过这也没办法，毕竟是从别的node上拷贝来的。

这3个的作用：
  1. ca.pem: 集群的根证书（CA，Certificate Authority），用于签发和验证 Kubernetes 集群中各种其他证书的有效性。 这个直接拷贝给其他 worker node 节点上的应该没啥问题
  2. worker-key.pem: kubelet的私钥
  3. worker.pem: kubelet的证书


对于有问题的node现在看来需要重新生成 worker-key.pem 和 worker.pem，具体的生成方法看这些：  
[Kubernetes集群安全：Api Server认证 | 青蛙小白 (frognew.com)](https://blog.frognew.com/2017/01/kubernetes-api-server-authc.html)，  [手动生成证书 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/#openssl)

基本步骤：
1. 拿到集群的ca的私钥和证书（根证书），ca.key 和 ca.pem
2. 用 openssl 工具生成一个私钥，然后生成这个私钥的csr
3. 利用这个 csr 和 ca.key 和 ca.pem 来生成（签发）私钥的证书，最后将私钥和证书拷贝给工作节点并确保kubelet的启动参数指定了私钥和证书的文件路径即可  


总之如果是自己手动搭建k8s集群，证书的生成全手动来做的话，那大致过程就是：
1. 生成一个私钥和他的证书，作为整个集群的ca的私钥和证书，整个集群中其他的所有证书都要由这个ca的证书来签发
2. 生成n个私钥，并让ca签发其证书
3. 将一对私钥和证书设置为 api server，kubelet 等组件时的启动参数
4. k8s里很多组件之间的通信都需要这种基于ca的双向验证，所以基本就是一个组件（可以认为他是一个通信客户端）就给它配一组私钥和证书


最后就是关于 公钥，私钥，证书，ca 这套通信机制的简单概括：
1. 一个客户端持有一对公钥和私钥，和自己的证书（这里应该是服务器，因为只有服务器需要验证自己是真的服务器，但是在互相通信的模型中，每个通信单位即是客户端也是服务器）  
2. 私钥在通信中不交换，只交换公钥（公钥字符串信息在证书里）  
3. A和B双向通信时，先向ca确认彼此的证书，因为所有的证书由ca签发，所以ca知道B是不是真正的B  
4. ca相当于一个第三方的绝对可信机构，浏览器中一般的内置了这些ca机构的证书，当然在一个私有集群里也可以使用私有的ca，比如在k8s集群里用 openssl 生成的 ca的私钥和其证书