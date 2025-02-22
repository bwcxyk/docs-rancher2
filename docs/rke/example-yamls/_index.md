---
title: cluster.yml 文件示例
description: 您可通过编辑 RKE 的集群配置文件`cluster.yml`，完成多种配置选项。以下是最小文件示例和完整文件示例。
keywords:
  - rancher
  - rancher中文
  - rancher中文文档
  - rancher官网
  - rancher文档
  - Rancher
  - rancher 中文
  - rancher 中文文档
  - rancher cn
  - RKE
  - cluster.yml 文件示例
---

您可通过编辑 RKE 的集群配置文件`cluster.yml`，完成多种配置选项。以下是最小文件示例和完整文件示例。

> **说明：**如果您使用的是 Rancher v2.0.5 或 v2.0.6，使用集群配置文件，配置集群选项时，服务名称不能含有除了英文字母和下划线外的其他字符。

## 最小文件示例

```yaml
nodes:
  - address: 1.2.3.4
    user: ubuntu
    role:
      - controlplane
      - etcd
      - worker
```

## 完整文件示例

```yaml
nodes:
  - address: 1.1.1.1
    user: ubuntu
    role:
      - controlplane
      - etcd
    ssh_key_path: /home/user/.ssh/id_rsa
    port: 2222
  - address: 2.2.2.2
    user: ubuntu
    role:
      - worker
    ssh_key: |-
      -----BEGIN RSA PRIVATE KEY-----

      -----END RSA PRIVATE KEY-----
  - address: example.com
    user: ubuntu
    role:
      - worker
    hostname_override: node3
    internal_address: 192.168.1.6
    labels:
      app: ingress

# 默认值为false，如果设置为true，当发现不支持的Docker版本时，RKE不会报错
ignore_docker_version: false

# 集群级SSH私钥，如果没有为节点设置ssh信息则使用该私钥
ssh_key_path: ~/.ssh/test

# 启用SSH代理，使用带有密码的SSH私钥
# 这需要配置环境`SSH_AUTH_SOCK`，指向已添加私钥的SSH代理
ssh_agent_auth: true

# 镜像仓库凭证列表
# 如果你使用的是Docker Hub注册表，
# 你可以省略`url`
# 或者设置为`docker.io`is_default设置为`true`
# 将覆盖全局设置中设置的系统默认注册表
private_registries:
  - url: registry.com
    user: Username # 请替换为真实的用户名
    password: password # 请替换为真实的密码
    is_default: true

# 堡垒机配置
bastion_host:
  address: x.x.x.x
  user: ubuntu
  port: 22
  ssh_key_path: /home/user/.ssh/bastion_rsa
# or
#   ssh_key: |-
#     -----BEGIN RSA PRIVATE KEY-----
#
#     -----END RSA PRIVATE KEY-----

# Set the name of the Kubernetes cluster
cluster_name: mycluster

# The Kubernetes version used. The default versions of Kubernetes
# are tied to specific versions of the system images.
#
# For RKE v0.2.x and below, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/types/blob/release/v2.2/apis/management.cattle.io/v3/k8s_defaults.go
#
# For RKE v0.3.0 and above, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/kontainer-driver-metadata/blob/master/rke/k8s_rke_system_images.go
#
# In case the kubernetes_version and kubernetes image in
# system_images are defined, the system_images configuration
# will take precedence over kubernetes_version.
kubernetes_version: v1.10.3-rancher2

# System Images are defaulted to a tag that is mapped to a specific
# Kubernetes Version and not required in a cluster.yml.
# Each individual system image can be specified if you want to use a different tag.
#
# For RKE v0.2.x and below, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/types/blob/release/v2.2/apis/management.cattle.io/v3/k8s_defaults.go
#
# For RKE v0.3.0 and above, the map of Kubernetes versions and their system images is
# located here:
# https://github.com/rancher/kontainer-driver-metadata/blob/master/rke/k8s_rke_system_images.go
#
system_images:
  kubernetes: rancher/hyperkube:v1.10.3-rancher2
  etcd: rancher/coreos-etcd:v3.1.12
  alpine: rancher/rke-tools:v0.1.9
  nginx_proxy: rancher/rke-tools:v0.1.9
  cert_downloader: rancher/rke-tools:v0.1.9
  kubernetes_services_sidecar: rancher/rke-tools:v0.1.9
  kubedns: rancher/k8s-dns-kube-dns-amd64:1.14.8
  dnsmasq: rancher/k8s-dns-dnsmasq-nanny-amd64:1.14.8
  kubedns_sidecar: rancher/k8s-dns-sidecar-amd64:1.14.8
  kubedns_autoscaler: rancher/cluster-proportional-autoscaler-amd64:1.0.0
  pod_infra_container: rancher/pause-amd64:3.1

services:
  etcd:
  # if external etcd is used
  # path: /etcdcluster
  # external_urls:
  #   - https://etcd-example.com:2379
  # ca_cert: |-
  #   -----BEGIN CERTIFICATE-----
  #   xxxxxxxxxx
  #   -----END CERTIFICATE-----
  # cert: |-
  #   -----BEGIN CERTIFICATE-----
  #   xxxxxxxxxx
  #   -----END CERTIFICATE-----
  # key: |-
  #   -----BEGIN PRIVATE KEY-----
  #   xxxxxxxxxx
  #   -----END PRIVATE KEY-----
  # Note for Rancher v2.0.5 and v2.0.6 users: If you are configuring
  # Cluster Options using a Config File when creating Rancher Launched
  # Kubernetes, the names of services should contain underscores
  # only: `kube_api`.
  kube-api:
    # IP range for any services created on Kubernetes
    # This must match the service_cluster_ip_range in kube-controller
    service_cluster_ip_range: 10.43.0.0/16
    # Expose a different port range for NodePort services
    service_node_port_range: 30000-32767
    pod_security_policy: false
    # Add additional arguments to the kubernetes API server
    # This WILL OVERRIDE any existing defaults
    extra_args:
      # Enable audit log to stdout
      audit-log-path: "-"
      # Increase number of delete workers
      delete-collection-workers: 3
      # Set the level of log output to debug-level
      v: 4
  # Note for Rancher 2 users: If you are configuring Cluster Options
  # using a Config File when creating Rancher Launched Kubernetes,
  # the names of services should contain underscores only:
  # `kube_controller`. This only applies to Rancher v2.0.5 and v2.0.6.
  kube-controller:
    # CIDR pool used to assign IP addresses to pods in the cluster
    cluster_cidr: 10.42.0.0/16
    # IP range for any services created on Kubernetes
    # This must match the service_cluster_ip_range in kube-api
    service_cluster_ip_range: 10.43.0.0/16
  kubelet:
    # Base domain for the cluster
    cluster_domain: cluster.local
    # IP address for the DNS service endpoint
    cluster_dns_server: 10.43.0.10
    # Fail if swap is on
    fail_swap_on: false
    # Set max pods to 250 instead of default 110
    extra_args:
      max-pods: 250
    # Optionally define additional volume binds to a service
    extra_binds:
      - "/usr/libexec/kubernetes/kubelet-plugins:/usr/libexec/kubernetes/kubelet-plugins"

# Currently, only authentication strategy supported is x509.
# You can optionally create additional SANs (hostnames or IPs) to
# add to the API server PKI certificate.
# This is useful if you want to use a load balancer for the
# control plane servers.
authentication:
  strategy: x509
  sans:
    - "10.18.160.10"
    - "my-loadbalancer-1234567890.us-west-2.elb.amazonaws.com"

# Kubernetes Authorization mode
# Use `mode: rbac` to enable RBAC
# Use `mode: none` to disable authorization
authorization:
  mode: rbac

# If you want to set a Kubernetes cloud provider, you specify
# the name and configuration
cloud_provider:
  name: aws

# Add-ons are deployed using kubernetes jobs. RKE will give
# up on trying to get the job status after this timeout in seconds..
addon_job_timeout: 30

# Specify network plugin-in (canal, calico, flannel, weave, or none)
network:
  plugin: canal

# Specify DNS provider (coredns or kube-dns)
dns:
  provider: coredns

# Currently only nginx ingress provider is supported.
# To disable ingress controller, set `provider: none`
# `node_selector` controls ingress placement and is optional
ingress:
  provider: nginx
  node_selector:
    app: ingress

# All add-on manifests MUST specify a namespace
addons: |-
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-nginx
    namespace: default
  spec:
    containers:
    - name: my-nginx
      image: nginx
      ports:
      - containerPort: 80

addons_include:
  - https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/rook-operator.yaml
  - https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/rook-cluster.yaml
  - /path/to/manifest
```
