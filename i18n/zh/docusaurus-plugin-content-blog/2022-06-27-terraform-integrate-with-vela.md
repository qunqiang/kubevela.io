---
title: 将 Terraform 生态系统集成到 Kubernetes 世界中
author: Jianbo Sun
author_title: KubeVela Team
author_url: https://github.com/kubevela/KubeVela
author_image_url: https://KubeVela.io/img/logo.svg
tags: [ KubeVela, Terraform, Kubernetes, DevOps, CNCF, CI/CD, Application delivery]
description: "这篇文章主要讨论如何使用 KubeVela 集成 Terraform 生态"
image: https://raw.githubusercontent.com/oam-dev/KubeVela.io/main/docs/resources/KubeVela-03.png
hide_table_of_contents: false
---

如果你正在寻找一些能够将 Terraform 生态集成到 Kubernetes 世界中的方法，那么恭喜你。 你将在这篇博客中找到你想要的方法。

我们会通过一个真实的案例来介绍如何集成 Terraform 模块到 KubeVela 中，灵感来自 Alex Ellis 的文章 “[改善开发人员在 Kubernetes 端口转发中的体验](https://inlets.dev/blog/2022/06/24/fixing-kubectl-port-forward.html)”。

总的来说，这篇文章会分为两个部分：

* 第一部分主要介绍如何把 Terraform 导入到 KubeVela中，这个工作需要一些 Terraform 和 KubeVela 的基本知识。如果你不想要作为一个开发人员来扩展 KubeVela，你可以跳过这个部分。
* 第二部分主要介绍 KubeVela 能做的一些事情 1）通过 KubeVela 使用公共 IP 配置 Cloud ECS 实例； 2）使用 ECS 实例作为隧道服务器，为内网环境中的任何容器服务提供公共访问。

好的，让我们开始吧！

## 第一部分 集成 Terraform 模块作为 KubeVela的能力

众所周知，[KubeVela](https://kubevela.net/docs/) 是一个现代化软件分发控制平面。你可能会问，这么做能有什么收益？

1. 将 Terraform 与 Kubernetes 生态系统（包括 Helm Charts）结合在一个统一的解决方案中，它可帮助你进行 GitOps、CI/CD 集成和应用程序生命周期管理。
    - 考虑部署一个包含云数据库、容器服务和几个 helm 图表的产品，现在你可以一起管理和部署它们，而无需切换到不同的工具。
2. 所有资源的声明模型，KubeVela 将运行协调循环直到成功。
    - 你不会被 terraform CLI 的网络问题所阻止。
3. 一个强大的基于 CUE 的工作流程，你可以在应用程序交付过程中定义任何首选步骤。
    - 你可以按照自己喜欢的方式编写，例如金丝雀发布、多集群/多环境支持。

如果你已经非常熟悉 Terraform，这次的集成过程会变得非常简单。

### 构建你的 Terraform 模块

> 如果你已经有一个测试完善的 Terraform 模块的话，可以跳过这部分。

在开始之前，确认你已经具备了下面的条件：

- 已经安装了 [terraform CLI](https://www.terraform.io/downloads)。
- 拥有云服务凭证，本文以阿里云为例。
- 学习过[如何使用 terraform](https://www.terraform.io/language) 官网中的基本知识。

这里是我准备这个案例的 Terraform 模块（ https://github.com/wonderflow/terraform-alicloud-ecs-instance ）

* 克隆这个模块：

```
git clone https://github.com/wonderflow/terraform-alicloud-ecs-instance.git
cd terraform-alicloud-ecs-instance
```

* 初始化并安装阿里云最新的稳定版本：
* Initialize and download the latest stable version of the Alibaba Cloud provider:

```shell
terraform init
```

* 配置阿里云提供商凭据：

```shell
export ALICLOUD_ACCESS_KEY="your-accesskey-id"
export ALICLOUD_SECRET_KEY="your-accesskey-secret"
export ALICLOUD_REGION="your-region-id"
```

You can also create an `provider.tf` including the credentials instead:

```hcl
provider "alicloud" {
    access_key  = "your-accesskey-id"
    secret_key   = "your-accesskey-secret"
    region           = "cn-hangzhou"
}
```

* 测试创建资源：

```shell
terraform apply -var-file=test/test.tfvars
```

* 销毁测试资源：

```shell
terraform destroy  -var-file=test/test.tfvars
```

你可以根据自己的需要定制这个模块并推送到你自己的 git 仓库。

### 集成 Terraform 模块作为 KubeVela 的能力

在开始之前，确认你已经 [安装了 KubeVela 控制平面](https://kubevela.net/docs/install#1-install-velad),不用担心你没有 Kubernetes 集群，valad 运行案例已经足够了。

接下来，我们将会使用我们之前准备好的 Terraform 模块。

* 生成组件定义

```
vela def init ecs --type component --provider alibaba --desc "Terraform configuration for Alibaba Cloud Elastic Compute Service" --git https://github.com/wonderflow/terraform-alicloud-ecs-instance.git > alibaa-ecs.yaml
```

> 如果你有自己的 git 地址，你可以替换成你自己的

* 安装到 vela 控制平面中

```
vela kube apply -f alibaa-ecs-def.yaml
```

> `vela kube apply` 和 `kubectl apply` 完全一样


我们成功添加了 ECS 模块的扩展，你可以从 [这里](https://kubevela.net/docs/platform-engineers/components/component-terraform) 了解更多详细信息。

我们已经完成了集成，最终用户可以在应用后立即发现能力。

最后，用户可以使用以下命令检查参数：

```
vela show alibaba-ecs
```

他们还可以通过启动网站上来查看它：

```
vela show alibaba-ecs --web
```

这就是集成 Terraform 全部所需要的。

## 第二部分 改善开发人员使用 Kubernetes 端口转发的体验

在这部分，我们会介绍一个解决方案，它可以让你将任意一个 Kubernetes 服务通过一个指定的端口暴露到公网。这个方案由以下几部分组成：

1. KubeVela 环境，你已经在第一部分的练习中搭建好了。
2. 阿里云 ECS，KubeVela 会使用访问秘钥自动创建一个`1u1g`规格的 ecs。
3. [frp](https://github.com/fatedier/frp)，KubeVela 会同时在客户端和服务端加载这个代理服务。


### 准备 KubeVela 环境

* 安装 KubeVela

```
curl -fsSl https://static.kubevela.net/script/install-velad.sh | bash
velad install
```

你可以查看[文档](https://kubevela.net/docs/install#1-install-velad)来获得更多安装细节。


* 启用 Terraform 插件和 Terraform 阿里云插件

```
vela addon enable terraform
vela addon enable terraform-alibaba
```

* 添加阿里云凭证

```
vela provider add terraform-alibaba --ALICLOUD_ACCESS_KEY <"your-accesskey-id"> --ALICLOUD_SECRET_KEY "your-accesskey-secret" --ALICLOUD_REGION <your-region> --name terraform-alibaba-default
```

查看[文档](https://kubevela.net/docs/reference/addons/terraform)以获得更多关于其他云的资料。

### 启动一个拥有公网 IP 的 ECS，并部署`frp`服务

环境准备完成之后，你可以通过下面的方式创建一个应用：


```shell
cat <<EOF | vela up -f -
# YAML begins
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: ecs-demo
spec:
  components:
    - name: ecs-demo
      type: alibaba-ecs
      properties:
        providerRef:
          name: terraform-alibaba-default
        writeConnectionSecretToRef:
          name: outputs-ecs          
        name: "test-terraform-vela-123"
        instance_type: "ecs.n1.tiny"
        host_name: "test-terraform-vela"
        password: "Test-123456!"
        internet_max_bandwidth_out: "10"
        associate_public_ip_address: "true"
        instance_charge_type: "PostPaid"
        user_data_url: "https://raw.githubusercontent.com/wonderflow/terraform-alicloud-ecs-instance/master/frp.sh"
        ports:
        - 8080
        - 8081
        - 8082
        - 8083
        - 9090
        - 9091
        - 9092
        tags:
          created_by: "Terraform-of-KubeVela"
          created_from: "module-tf-alicloud-ecs-instance"
# YAML ends
EOF
```

这个应用将会被部署为一个带公网 IP 的 ECS 实例，下面是一些有用的字段：


| 字段 | 用途  |
|:----:|:---:|
| providerRef | 添加的云服务凭证的引用 |
| writeConnectionSecretToRef | terraform 模块的输出将被写入 secret |
| name | ecs 实例名称 |
| instance_type | ecs 实例类型 |
| host_name | ecs 主机名称 |
| password | ecs 主机登录密码，你可以通过`ssh`进行连接时使用|
| internet_max_bandwidth_out | ecs 实例的最大带宽 |
| associate_public_ip_address | 是否创建公网 IP |
| instance_charge_type | 资源的收费方式 |
| user_data_url | 创建 ecs 实例后的安装脚本，我们已经在脚本中安装了 frp 服务器 |
| ports | VPC 和安全组中允许的端口，9090/9091 是 frp 服务器必须的，而其他端口则保留供客户端使用 |
| tags | ECS实例的标签 |

你可以通过以下方式了解更多字段信息：

```
vela show alibaba-ecs
```

部署完成之后，你可以检查应用的状态和日志：

```
vela status ecs-demo
vela logs ecs-demo
```

你可以从包含输出值的 terraform 资源中获取 secret。

你可能已经在 `vela logs` 中看到了结果，也可以通过以下方式检查 Terraform 的输出信息：

```shell
$ kubectl get secret outputs-ecs --template={{.data.this_public_ip}} | base64 --decode
["121.196.106.174"]
```

> KubeVela 将很快支持这样的查询资源 https://github.com/kubevela/kubevela/issues/4268 。

这样就可以在`:9091`端口访问 frp 服务器管理页面，脚本中用户名是`admin`密码为`vela123`。

至此，我们已经完成了服务器部分。

### 在 KubeVela 中使用 frp 客户端

frp 客户端的使用非常简单，我们可以为集群内的任何服务提供公共 IP。

![](../docs/resources/terraform-ecs.png)

1. 独立部署以代理任何 [Kubernetes 服务](https://kubernetes.io/docs/concepts/services-networking/service/)。

```yaml
cat <<EOF | vela up -f -
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: frp-proxy
spec:
  components:
    - name: frp-proxy
      type: worker
      properties:
        image: oamdev/frpc:0.43.0
        env:
          - name: server_addr
            value: "121.196.106.174"
          - name: server_port
            value: "9090"
          - name: local_port
            value: "80"
          - name: connect_name
            value: "velaux-service"
          - name: local_ip
            value: "velaux.vela-system"
          - name: remote_port
            value: "8083"
EOF
```

在这种情况下，我们通过 `velaux.vela-system` 指定 `local_ip`，这意味着我们正在访问命名空间 `vela-system` 中名称为 `velaux` 的 Kubernetes 服务。

你可以从公共 IP `121.196.106.174:8083` 访问 velaux 服务。

2. 将两个组件组合在一起以实现相同的生命周期。

```yaml
cat <<EOF | vela up -f -
# YAML begins
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: composed-app
spec:
  components:
    - name: web-new
      type: webservice
      properties:
        image: oamdev/hello-world:v2
        ports:
          - port: 8000
            expose: true
    - name: frp-web
      type: worker
      properties:
        image: oamdev/frpc:0.43.0
        env:
          - name: server_addr
            value: "121.196.106.174"
          - name: server_port
            value: "9090"
          - name: local_port
            value: "8000"
          - name: connect_name
            value: "composed-app"
          - name: local_ip
            value: "web-new.default"
          - name: remote_port
            value: "8082"
EOF
```

哇！ 然后你可以通过以下方式访问 `hello-world`：

```
curl 121.196.106.174:8082
```

`webservice` 类型的组件会自动生成一个带有组件名称的服务。 `frp-web` 组件会将流量代理到 `default` 命名空间中的服务 `web-new`，这正是生成的服务。

当应用程序被删除时，同一个应用程序中定义的所有资源都会被一起删除。

你还可以与它们一起组成数据库，然后你可以一次性交付所需的所有组件。

### 清理

你可以通过 `vela delete` 清理演示中的所有应用程序：

```
vela delete composed-app -y
vela delete frp-proxy -y
vela delete ecs-demo -y
```

我想你现在已经学会了如何在这个场景中使用 KubeVela，只需在你的环境中尝试一下！

## 更多

在这篇博客中，我们介绍了将 Terraform 模块与 KubeVela 集成的方式。 它提供了有趣的用例，允许你向公众公开任何内部服务。

KubeVela 可以做更多的事情，请前往 [kubevela.io](https://kubevela.io/) 去探索吧！