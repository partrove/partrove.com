+++
title = '在Kubernetes集群中部署安全的metrics-server服务'
date = 2024-03-14T13:40:57+08:00
draft = true
isCJKLanguage = true
+++

# Metrics Server介绍  

Kubernetes Metrics Server 是一个监控 metrics 采集组件，用于从集群中各 kubelet 中采集 metrics 数据，然后以扩展 APIServer 的形式对外暴露 [Metrics API](https://github.com/kubernetes/metrics)。  

Metrics Server 作用包括但不限于下面这些：

- 给集群的 [HPA](https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/) 和 [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler/) 提供数据支撑；
- `kubectl top` 命令依赖 Metrics Server；
- 给 Prometheus 之类的监控方案提供数据接口；

# 部署安全的metrics-server  

## 启用 Aggregation Layer  

由于 Metrics Server 使用的是 Kubernetes 扩展 APIServer 模式，因此要部署 Metrics Server，首先要开启 kube-apiserver 的聚合层。  

> [Kubernetes API 聚合层](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)  
> >[配置聚合层](https://kubernetes.io/zh-cn/docs/tasks/extend-kubernetes/configure-aggregation-layer/)  

这里我们使用 kubeadm 来搭建集群，因此先定制 kubeadm [[kubeadm - ClusterConfiguration|ClusterConfiguration]]，在 *apiServer.extraArgs* 字段中添加聚合层的配置项。 

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  timeoutForControlPlane: 4m0s
  extraArgs:
    # Security
    anonymous-auth: "true"
    # Enable Aggregation Layer
    requestheader-client-ca-file: /etc/kubernetes/pki/front-proxy-ca.crt
    requestheader-allowed-names: front-proxy-client
    requestheader-extra-headers-prefix: X-Remote-Extra-
    requestheader-group-headers: X-Remote-Group
    requestheader-username-headers: X-Remote-User
    proxy-client-cert-file: /etc/kubernetes/pki/front-proxy-client.crt
    proxy-client-key-file: /etc/kubernetes/pki/front-proxy-client.key
    # Logging
    event-ttl: "2h"
    audit-log-format: legacy
    audit-log-maxage: "7"
    audit-log-maxbackup: "5"
    audit-log-maxsize: "100"
    audit-log-path: /var/log/kubernetes/apiserver-audit.log
    audit-policy-file: /var/lib/kubernetes/audit-policy.yaml
  extraVolumes:
  - name: audit-log
    hostPath: /var/log/kubernetes
    mountPath: /var/log/kubernetes
    pathType: DirectoryOrCreate
  - name: audit-policy
    hostPath: /var/lib/kubernetes
    mountPath: /var/lib/kubernetes
    pathType: DirectoryOrCreate
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kubernetesVersion: 1.29.2
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```  

## 配置 kubelet 使用集群 CA 签发的服务证书  

默认情况下，kubelet 使用自签证书对外提供 HTTPS 服务（默认的10250端口），而由于 metrics-server 需要连接 kubelet 去获取 metrics 数据，自签证书使得无法通过 HTTPS 进行连接。  

要使 metrics-server 可以从 kubelet 获取数据，一种方案是，在 metrics-server 启动参数中加上 `--kubelet-insecure-tls` 来跳过 TLS 验证，但这种方案不推荐用于生产环境。  

另一种可行的方案，就是用集群 CA 来签发 kubelet 所需的服务端证书。  

对于将要新建的集群，可以先定制 kubeadm KubeletConfiguration  

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
  webhook:
    enabled: true
    cacheTTL: 0s
  anonymous:
    enabled: false
authorization:
  mode: Webhook
serverTLSBootstrap: true
imageGCHighThresholdPercent: 80
imageGCLowThresholdPercent: 70
serializeImagePulls: false
cgroupDriver: systemd
```  

对于运行中的集群，可以参考官方文档进行修改。  

> [启用已签名的 kubelet 服务证书](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs)  

启用这个功能后，需要注意的是节点加入集群后，要手动前往集群里批复 CSR，之后 kubelet 的 10250 才能正常提供服务。  
等 kubelet 服务运行正常，可以通过下面的命令验证其服务端证书。  

```shell
[root@master ~]# openssl s_client -connect 127.0.0.1:10250 -showcerts
CONNECTED(00000003)
Can't use SSL_get_servername
depth=0 O = system:nodes, CN = system:node:192.168.193.10
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 O = system:nodes, CN = system:node:192.168.193.10
verify error:num=21:unable to verify the first certificate
verify return:1
depth=0 O = system:nodes, CN = system:node:192.168.193.10
verify return:1
---
Certificate chain
 0 s:O = system:nodes, CN = system:node:192.168.193.10
   i:CN = kubernetes
```  

可以看到证书的 Issuer 为 *CN = kubernetes*， 也就是集群的 CA。  

## 部署metrics-server  

前面的步骤准备完成后，部署就比较简单了，我们这里使用 helm 进行部署。  

```shell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server
```  

这里不需要修改默认参数就可以正常使用。部署完成后，metrics-server 默认会使用 10250 作为 containerPort，443作为 servicePort。  
# 问题记录  

## FailedDiscoveryCheck  

1. 先检查 metrics-server 自己的 pod 是否正常，pod 日志有没有报错；
2. 检查 metrics-server 的 service 是否正常；
3. 检查集群的 CNI 是否正常；