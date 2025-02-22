---
title: "2、同步镜像到私有镜像仓库"
description: 本节介绍如何配置私有镜像仓库，以便在安装 Rancher 时，Rancher 可以从此私有镜像仓库中拉取所需的镜像。
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
  - 安装指南
  - 其他安装方法
  - 离线安装
  - 同步镜像到私有镜像仓库
---

本节介绍如何配置私有镜像仓库，以便在安装 Rancher 时，Rancher 可以从此私有镜像仓库中拉取所需的镜像。

默认情况下，Rancher 中所有用于[创建 Kubernetes 集群](/docs/rancher2.5/cluster-provisioning/_index)或运行任何[工具](/docs/rancher2.5/cluster-admin/tools/_index)的镜像都从 Docker Hub 中拉取，如： 监控、日志、告警、流水线等镜像。在离线环境下安装 Rancher，您需要一个私有镜像库，这个私有镜像库应该位于 Rancher 中的节点能访问的位置上。然后您需要载入所有镜像到这个私有镜像库里。

对于高可用安装和单节点安装，同步镜像到私有镜像仓库的过程是相同的。

但是同步用来创建 Linux 集群和用来创建 Windows 集群的镜像过程是不同的。所以这取决于您是否要创建 Windows 集群。默认情况下，我们假设只配置 Linux 集群，我们提供了如何推送 Linux 需要的镜像到私有镜像库的步骤。但是如果您计划创建 [Windows 集群](/docs/rancher2.5/cluster-provisioning/rke-clusters/windows-clusters/_index)，我们有单独的文档来介绍如何同步 Linux 和 Windows 集群所需的镜像。

> **先决条件：** 已有一个 Rancher 可访问的[私有镜像仓库](https://docs.docker.com/registry/deploying/)。

## 集群中仅有 Linux 节点

对于创建仅有 Linux 节点的集群的 Rancher Server，请按以下步骤推送镜像到私有镜像库。

1. 查找您用的 Rancher 版本所需要的资源
2. 搜集 cert-manager 镜像
3. 将镜像保存到您的工作站中
4. 推送镜像到私有镜像库

#### 先决条件

这些步骤要求您使用一个 Linux 工作站，它可以访问互联网和您的私有镜像库。请确保至少有 20GB 的磁盘空间可用。

如果要使用 ARM64 主机，则镜像仓库必须支持 Docker Manifest。截至 2020 年 4 月，Amazon Elastic Container Registry 不支持 Docker Manifest 功能。

#### 1、查找您用的 Rancher 版本所需要的资源

1. 浏览我们的[版本发布页面](https://github.com/rancher/rancher/releases)，查找您想安装的 Rancher v2.x.x 版本。不要下载标记为 `rc` 或 `Pre-release` 的版本，因为它们在生产环境下是不稳定的。

1. 从发行版 **Assets** 部分下载以下文件，这些文件是离线环境下安装 Rancher 所必需的：

   | Release 文件             | 描述                                                                                                                 |
   | :----------------------- | :------------------------------------------------------------------------------------------------------------------- |
   | `rancher-images.txt`     | 此文件包含安装 Rancher、创建集群和运行 Rancher 工具所需的镜像列表。                                                  |
   | `rancher-save-images.sh` | 这个脚本会从 DockerHub 中拉取在文件`rancher-images.txt`中描述的所有镜像，并将它们保存为文件`rancher-images.tar.gz`。 |
   | `rancher-load-images.sh` | 这个脚本会载入文件`rancher-images.tar.gz`中的镜像，并将它们推送到您自己的私有镜像库。                                |

#### 2、收集 cert-manager 镜像

如果您使用自己的证书，或者要在外部负载均衡器上终止 TLS，请跳过此步骤。

在安装高可用过程中，如果选择使用 Rancher 默认的自签名 TLS 证书，则还必须将 [`cert-manager`](https://hub.helm.sh/charts/jetstack/cert-manager) 镜像添加到 `rancher-images.txt` 文件中。如果使用自己的证书，则跳过此步骤。

1.  获取最新的`cert-manager` Helm chart，解析模板,获取镜像详细信息：

    > **注意：** 由于`cert-manager`最近的改动，您需要升级`cert-manager`版本。如果您要升级 Rancher 并且使用`cert-manager`的版本低于 v0.12.0，请看我们的[升级文档](/docs/rancher2.5/installation/resources/upgrading-cert-manager/_index)。

    ```plain
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    helm fetch jetstack/cert-manager --version v0.12.0
    helm template ./cert-manager-<version>.tgz | grep -oP '(?<=image: ").*(?=")' >> ./rancher-images.txt
    ```

2.  对镜像列表进行排序和唯一化，去除重复的镜像源：

    ```plain
    sort -u rancher-images.txt -o rancher-images.txt
    ```

#### 3、将镜像保存到您的工作站中

1. 为`rancher-save-images.sh` 文件添加可执行权限：

   ```
   chmod +x rancher-save-images.sh
   ```

1. 执行脚本`rancher-save-images.sh`并以`--image-list ./rancher-images.txt` 作为参数，创建所有需要镜像的压缩包：

   ```plain
   ./rancher-save-images.sh --image-list ./rancher-images.txt
   ```

   :::tip 提示
   国内用户，可以从 http://mirror.rancher.cn --> rancher --> [rancher 版本] 下载 rancher-save-images.sh，该脚本支持通过参数 `--from-aliyun true` 来指定从阿里云镜像仓库拉去rancher镜像（从rancher/rancher release下载的rancher-save-images.sh不支持该参数），例如：

   ```
   ./rancher-save-images.sh --image-list ./rancher-images.txt --from-aliyun true
   Image pull success: registry.cn-hangzhou.aliyuncs.com/rancher/busybox
   Image pull success: registry.cn-hangzhou.aliyuncs.com/rancher/backup-restore-operator:v1.0.4-rc4
   ...
   ```

   :::

   **结果：** Docker 会开始拉取用于离线安装所需的镜像。这个过程会花费几分钟时间。完成时，您的当前目录会输出名为`rancher-images.tar.gz`的压缩包。请确认输出文件是否存在。

#### 4、推送镜像到私有镜像库

下一步，您将使用脚本将文件 `rancher-images.tar.gz` 中的镜像上传到您自己的私有镜像库。

文件 `rancher-images.txt` 和 `rancher-images.tar.gz` 应该位于工作站中运行 `rancher-load-images.sh` 脚本的同一目录下。

1.  登录私有镜像库：
    ```plain
    docker login <REGISTRY.YOURDOMAIN.COM:PORT>
    ```
1.  为 `rancher-load-images.sh` 添加可执行权限：

    ```
    chmod +x rancher-load-images.sh
    ```

1.  使用脚本 `rancher-load-images.sh`提取`rancher-images.tar.gz`文件中的镜像，根据文件`rancher-images.txt`中的镜像列表对提取的镜像文件重新打 tag 并推送到您的私有镜像库：

    ```
    ./rancher-load-images.sh --image-list ./rancher-images.txt --registry <REGISTRY.YOURDOMAIN.COM:PORT>`
    ```

## 集群中同时含有 Linux 节点 和 Windows 节点

_v2.3.0 及以后版本可用_

对于用来创建包含 Linux 节点和 Windows 节点的集群的 Rancher Server，因为一个集群会包含 Linux 和 Windows 的混合节点，所以会有不同的步骤来为您的私有镜像库同步 Windows 镜像和 Linux 镜像。

### 同步 Windows 镜像

从 Windows 工作站中收集和推送 Windows 镜像。

- 1. 查找您用的 rancher 版本所需要的资源
- 2. 将镜像保存到您的 Windows 服务工作站中
- 3. 准备 Docker 守护进程
- 4. 推送镜像到私有镜像库

#### 先决条件

这些步骤要求您使用 Windows Server 1809 工作站，该工作站能访问网络、您的私有镜像库以及至少拥有 50 GB 的磁盘空间。

工作站必须安装 Docker 18.02+版本，因为这个版本的 Docker 支持 manifests，这个特性是设置 Windows 集群所必须的。

如果要使用 ARM64 主机，则镜像仓库必须支持 Docker Manifest。截至 2020 年 5 月，Amazon Elastic Container Registry 不支持 Docker Manifest 功能。

#### 1、查找您用的 Rancher 版本所需要的资源

1. 浏览我们的[版本发布页面](https://github.com/rancher/rancher/releases)并查找您想要安装的 Rancher v2.x.x 版本。不要下载标记为 `rc` or `Pre-release` 的版本，因为它们在生产环境下不稳定。

2. 从发行版本的 "Assets" 部分，下载下列文件:

   | Release 文件                 | 描述                                                                                                                                   |
   | :--------------------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
   | `rancher-windows-images.txt` | 这个文件包含设置 Windows 集群所需的 Windows 镜像列表。                                                                                 |
   | `rancher-save-images.ps1`    | 这个脚本会从 DockerHub 上拉取所有在文件`rancher-windows-images.txt`中列举的镜像，并将它们保存到`rancher-windows-images.tar.gz`文件中。 |
   | `rancher-load-images.ps1`    | 这个脚本会从压缩包`rancher-windows-images.tar.gz`中加载镜像，并把它们推送到您的私有镜像库 中。                                         |

#### 2、将镜像保存到您的 Windows 服务工作站中

1. 在 `powershell`中，进入上一步有下载文件的目录里。

1. 运行脚本 `rancher-save-images.ps1`， 去创建所有必需的压缩包：

   ```plain
   ./rancher-save-images.ps1
   ```

   **结果：** Docker 会开始拉取离线安装所需要的镜像。这个过程需要几分钟时间，请耐心等待。拉取镜像结束后，您的当前目录会输出名为`rancher-windows-images.tar.gz`的压缩包。请确认输出文件是否存在。

#### 3、准备 Docker 守护进程

将您的私有镜像库地址追加到 Docker 守护进程配置文件(`C:\ProgramData\Docker\config\daemon.json`)的`allow-nondistributable-artifacts`配置字段中。由于 Windows 镜像的基础镜像是由`mcr.microsoft.com` Registry 维护的，这一步是必须的，因为 Docker Hub 中缺少 Microsoft 注册表层，需要将其拉入私有镜像库。

```
{
  ...
  "allow-nondistributable-artifacts": [
    ...
    "<REGISTRY.YOURDOMAIN.COM:PORT>"
  ]
  ...
}
```

#### 4、推送镜像到私有镜像库

将通过脚本，将文件`rancher-windows-images.tar.gz`中的镜像移入到您私有镜像库中。

在工作站中，文件`rancher-windows-images.txt`和`rancher-windows-images.tar.gz`要放在与运行脚本`rancher-load-images.ps1`的同一目录下。

1. 如果需要，请使用 `powershell`，登录到您的私有镜像库：

   ```plain
   docker login <REGISTRY.YOURDOMAIN.COM:PORT>
   ```

1. 在 `powershell`中，使用 `rancher-load-images.ps1`脚本，提取文件`rancher-images.tar.gz`中的镜像，重新打 tag 并将它们推送到您的私有镜像库中：

   ```plain
   ./rancher-load-images.ps1 --registry <REGISTRY.YOURDOMAIN.COM:PORT>
   ```

### 同步 Linux 镜像

Linux 镜像需要从 Linux 主机上收集和推送，但是必须先将 Windows 镜像推送到您的私有镜像库，然后再推送 Linux 镜像。这些步骤不同于只支持 Linux 集群设置步骤，因为被推送的 Linux 镜像实际是支持 Windows 和 Linux 镜像的 manifest 镜像。

- 查找您用的 rancher 版本所需要的资源
- 收集所有需要的镜像
- 将镜像保存到您的 Linux 服务工作站中
- 推送镜像到私有镜像库

#### 先决条件

在把 Linux 镜像推送到私有镜像库中之前，必须先把 Windows 镜像推送到私有镜像库中。如果您已经把 Linux 镜像推送到私有镜像库中，则需要再次按照这些说明重新推送，因为它们将发布支持 Windows 和 Linux 镜像的 manifest 镜像。

这些步骤要求您使用一个 Linux 工作站，它能访问互联网，而且可以访问您的私有镜像库。并且至少有 20GB 的磁盘空间。

因为需要支持 manifest 特性，工作站必须装有 Docker 18.02+ 版本，这是配置 Windows 集群的先决条件。

#### 1、查找您用的 Rancher 版本所需要的资源

1. 浏览我们的[版本发布页面](https://github.com/rancher/rancher/releases)并查找您想要安装的 Rancher v2.x.x 版本。不要下载标记为 rc or Pre-release 的版本，因为它们在生产环境下不稳定。

2. 从发行版 **Assets** 部分下载以下文件：

   | Release 文件                 | 描述                                                                                                                  |
   | :--------------------------- | :-------------------------------------------------------------------------------------------------------------------- |
   | `rancher-images.txt`         | 此文件包含安装 Rancher、设置集群和用户 Rancher 工具所需的镜像列表。                                                   |
   | `rancher-windows-images.txt` | 此文件包含需要设置 Windows 集群所需的镜像列表。                                                                       |
   | `rancher-save-images.sh`     | 这个脚本会从 Docker Hub 中拉取所有在文件`rancher-images.txt`中列举的镜像并将它们保存到`rancher-images.tar.gz`文件中。 |
   | `rancher-load-images.sh`     | 这个脚本会从压缩包`rancher-images.tar.gz`中加载镜像，并把它们推送到您的私有镜像库 中。                                |

#### 2、收集所有必需的镜像

**针对使用 Rancher 生成的自签名证书安装的高可用：** 在安装 Kubernetes 过程中，如果选择使用 Rancher 默认的自签名 TLS 证书，则还必须将[`cert-manager`](https://hub.helm.sh/charts/jetstack/cert-manager)镜像添加到`rancher-images.txt`文件中。如果使用自己的证书，则跳过此步骤。

1. 获取最新的`cert-manager` Helm chart，并解析模板以获取镜像详细信息：

   > **注意：** 由于`cert-manager`最近的改动，您需要升级`cert-manager`版本。如果您要升级 Rancher 并且使用`cert-manager`的版本低于 v0.12.0，请看我们的[升级文档](/docs/rancher2.5/installation/resources/upgrading-cert-manager/_index)。

   ```plain
   helm repo add jetstack https://charts.jetstack.io
   helm repo update
   helm fetch jetstack/cert-manager --version v0.12.0
   helm template ./cert-manager-<version>.tgz | grep -oP '(?<=image: ").*(?=")' >> ./rancher-images.txt
   ```

2. 对镜像列表进行排序和唯一化，以去除重复的镜像源：

   ```plain
   sort -u rancher-images.txt -o rancher-images.txt
   ```

#### 3、将镜像保存到您的工作站中

1. 给脚本添加 `rancher-save-images.sh` 可执行权限：

   ```
   chmod +x rancher-save-images.sh
   ```

1. 以镜像列表文件`rancher-images.txt`为参数执行脚本 `rancher-save-images.sh`来创建所有必须镜像的压缩包：

   ```plain
   ./rancher-save-images.sh --image-list ./rancher-images.txt
   ```

   :::tip 提示
   国内用户，可以从 http://mirror.rancher.cn --> rancher --> [rancher 版本] 下载 rancher-save-images.sh，该脚本支持通过参数 `--from-aliyun true` 来指定从阿里云镜像仓库拉去rancher镜像（从rancher/rancher release下载的rancher-save-images.sh不支持该参数），例如：

   ```
   ./rancher-save-images.sh --image-list ./rancher-images.txt --from-aliyun true
   Image pull success: registry.cn-hangzhou.aliyuncs.com/rancher/busybox
   Image pull success: registry.cn-hangzhou.aliyuncs.com/rancher/backup-restore-operator:v1.0.4-rc4
   ...
   ```

   :::

   **结果：** Docker 会开始拉取用于离线安装所需的镜像。这个过程会花费几分钟时间。完成时，您的当前目录会输出名为 rancher-images.tar.gz 的压缩包。请确认文件是否存在。

#### 4、推送镜像到私有镜像库

将用脚本`rancher-load-images.sh`将文件`rancher-images.tar.gz`中的镜像移入到您私有镜像库中。在工作站中，镜像列表文件 `rancher-images.txt` / `rancher-windows-images.txt`要放在与运行脚本`rancher-load-images.sh`同一目录下。

1. 请先登录到自己的私有镜像库中：

   ```plain
   docker login <REGISTRY.YOURDOMAIN.COM:PORT>
   ```

1. 给文件 `rancher-load-images.sh` 添加可执行权限：

   ```
   chmod +x rancher-load-images.sh
   ```

1. 使用 `rancher-load-images.sh`脚本，提取文件`rancher-images.tar.gz`中的镜像，重新打 tag 并将它们推送到您的私有镜像库 中：
   ```plain
   ./rancher-load-images.sh --image-list ./rancher-images.txt \
     --windows-image-list ./rancher-windows-images.txt \
     --registry <REGISTRY.YOURDOMAIN.COM:PORT>
   ```

## 后续操作

[安装 Rancher 高可用的下一步](/docs/rancher2.5/installation/other-installation-methods/air-gap/launch-kubernetes/_index)

[安装 Rancher 单节点的下一步](/docs/rancher2.5/installation/other-installation-methods/air-gap/install-rancher/_index)
