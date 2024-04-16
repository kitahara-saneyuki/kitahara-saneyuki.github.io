---
layout:     single
title:      "云原生系统架构实践（1）： Terraform 入门"
date:       2024-03-01 03:41:53 +0800
categories: Terraform
published:  true
---

本系列的目标是使用 AWS 非常便宜的 Spot Instance 来支撑一个较高可用（ 99.8% ，大约每周允许宕机 5 - 10 分钟）的 Kubernetes 单节点集群（即 Minikube ），部署我们的服务并实现自动伸缩。

技术栈：
- AWS Spot Instance
- Terraform
- Minikube
- Kubernetes
- Helm
- Docker

## 为什么要使用基础设施即代码？

基础设施即代码（ IaC, Infrastructure as Code ）使用代码取代了云服务上的手动流程和配置，以配置和支持计算基础设施。

手动管理基础设施既耗时又容易出错，使用代码管理基础设施可以
- 实现创建和管理基础设施的自动化，显著减少手动操作带来的错误。
- 通过版本控制系统，追溯基础设施的状态，方便团队成员协作。
- 通过代码化模板抽象出来常用基础设施模块，大幅度降低复用成本。
- 通过自动化测试实现持续集成与持续部署。
- 内建实现安全性与合规性，自动化安全扫描与合规性检查。

总之，好处是多多的。

## 安装 TFSwitch 和 Terraform （ Linux 环境下）

这个事在我国不是非常容易，一般来说只需要按照 TFSwitch 官网 [^1] 给定的操作即可：

```bash
curl -L https://raw.githubusercontent.com/warrensbox/terraform-switcher/release/install.sh | sudo bash
```

但由于我国的特殊国情，此处需要提高上网的科学化程度[^2]。
由于我们下载下来的脚本需要 `sudo` 权限才能执行，因此不仅用户账户需要更新 `~/.bashrc` ， `root` 账户亦然，否则下载执行过程会被茫然的无响应所卡住。

TFSwitch 安装完成，我们输入 `tfswitch` 命令安装所需版本的 `terraform` ，本文使用 terraform 版本 1.7.5。
注意默认环境下 `~/bin` 并不在执行路径中，需要我们在 `~/.bashrc` 中添加一行命令实现：

```bash
export PATH=~/bin:$PATH
```

此时输入 `terraform -v` ，验证 terraform 安装完成

```
Terraform v1.7.5
on linux_amd64
```

## 第一行代码

我们首先使用 `git init` 初始化架构代码库，在 `.gitignore` 文件中插入三行：

```gitignore
.terraform*
*.tfstate*
*.log
```

并同步到 GitHub 中方便管理。

### 获取 AWS 密钥对，并配置到本地

登陆 AWS 控制台，点击最右上角的用户名，点击 `Security credentials` ，创建一个 `Access Key` （密钥对）并下载其 CSV 文件。

安装 AWS 命令行工具：

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

配置 AWS 密钥对：

```bash
aws configure --profile <profile>
```

然后把下列两行代码插入到 `~/.bashrc` 文件尾部，并 `source ~/.bashrc`

```bash
export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
```

### 初始化 Terraform ，并以 S3 仓库保存 Terraform 状态

此时我们的代码库还没有一行代码，我们建立两个文件，分别描述我们的 Terraform 状态（此时还在本地）和 S3 仓库。

resources.tf ：

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.39.0"
    }
  }
  required_version = ">= 1.7.4"
}

provider "aws" {
  region  = "ap-southeast-1"
  profile = "<profile_name>"
}

resource "aws_default_vpc" "default" {
  tags = {
    Name      = "<vpc_name>"
    Terraform = true
  }
}
```

s3.tf ：

```terraform
resource "aws_s3_bucket" "<terraform_bucket>" {
  bucket        = "<terraform_bucket>"
  force_destroy = true

  tags = {
    Name      = "<terraform_bucket>"
    Namespace = "<terraform_namespace>"
    Terraform = true
    Component = "Data"
  }
}

resource "aws_s3_bucket_ownership_controls" "<terraform_bucket>" {
  bucket = aws_s3_bucket.<terraform_bucket>.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "<terraform_bucket>" {
  depends_on = [aws_s3_bucket_ownership_controls.<terraform_bucket>]
  bucket = aws_s3_bucket.<terraform_bucket>.id
  acl    = "private"
}

resource "aws_s3_bucket_public_access_block" "<terraform_bucket>" {
  bucket = aws_s3_bucket.<terraform_bucket>.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

键入 `terraform init` 和 `terraform apply` 以执行。

由于我们此时已经有了保存 Terraform 状态的 S3 文件桶，此时我们调整 `resource.tf` ，使之保存状态到 S3 文件桶中。

```terraform
terraform {
  backend "s3" {
    bucket = "<terraform_bucket>"
    key    = "terraform.tfstate"
    region = "ap-southeast-1"
  }
  ...
}
```

我们可以手动把本地的 `tfstate` 文件上传到 S3 文件桶中，然后删除本地所有 Terraform 状态文件（即 `gitignore` 所忽略掉的文件），重新运行 `terraform init` 以保证云服务架构状态总与远程 S3 文件桶中的状态相同步。

### 请求 Spot Instance

由于 Minikube 不支持 ARM 服务器，此处请务必申请一个 x86-64 的 EC2 机器。

此处需要请求一个弹性 IP （ EIP, Elastic IP ）以维持 Spot Instance 的可访问性。

```terraform
resource "aws_spot_instance_request" "<spot_instance>" {
  ami                            = "<ami_id>"
  instance_type                  = "r6a.medium"

  key_name                       = "<key_name>"
  security_groups                = ["launch-wizard-1"]
  instance_interruption_behavior = "stop"
  tags = {
    Namespace    = "<terraform_namespace>"
    Terraform    = true
    Service      = "Minikube"
    Architecture = "amd64"
  }

  root_block_device {
    volume_size = <size_by_gb>
  }
}

resource "aws_eip" "<spot_instance>" {
  instance = aws_spot_instance_request.<spot_instance>.spot_instance_id
}

resource "aws_eip_association" "<spot_instance>" {
  instance_id   = aws_spot_instance_request.<spot_instance>.spot_instance_id
  allocation_id = aws_eip.<spot_instance>.id
}
```

键入 `terraform apply` 以执行。

[^1]: https://tfswitch.warrensbox.com/Install/
[^2]: https://kitahara-saneyuki.github.io/terraform/guide-of-working-in-china-mainland/#linux-%E7%8E%AF%E5%A2%83%E4%B8%8B-1
