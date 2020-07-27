---
title: "云原生下的cicd"
date: 2020-07-24
lastmod: 2020-07-24
draft: false
author: "WithLin"
tags: ["cicd", "cicd的设计"]
categories: ["tekton"]
toc: true
comment: true
autoCollapseToc: false
---

>去年9月份，我们打算做自己的容器云，为了方便用户使用，我们需要结合自己公司特性去改造。容器云离不开的ci模块，在ci这块我们选择了用tekton作为我们的ci模块。

### tekton的介绍

  Tekton 是一个强大而灵活的 Kubernetes 原生开源框架，可用于创建持续集成和交付 (CI/CD) 系统。该框架可让您跨多个云服务商或本地系统进行构建、测试和部署，无需操心基础实现细节。


### 为什么选择tekton，作为我们的ci呢？

**云原生**

-  跑在Kubernetes上面
-  使用容器作为他的构建块
-  提供Kubernetes声明式资源的pipeline

**灵活**

- 一条pipeline可以部署到任意的Kubernetes集群中
- 组成pipeline的task可以任意组成
- git资源和image资源的自由组装
- Tekton 赋予您充分的灵活性，您可以使用自己偏好的 CI/CD 工具创建功能强大的流水线。Tekton 让您无需操心基础实现，只需根据团队的要求选择构建、测试和部署工作流即可。
  
**轻量**

- 基于operator的方式编排你的pipeline

**资源限制**

- 多租户场景，资源的控制。

**多环境运行**

- 可让您跨多个环境（例如虚拟机、无服务器、Kubernetes 或 Firebase 环境）进行构建、测试和部署。您还可以使用 Tekton 流水线跨多个云服务商或混合环境进行部署


## tekton的资源介绍

### PipelineResource 资源简介
  
>PipelineResource 是 pipeline 中的一组对象，这些对象可以用于 task 的 input（输入）和 outoput（ 输出）,Pipeline-resoure 目前支持两种类型（git 和 image）, 一个 task(任务)可以有一个或者多个 input(输入)和 output(输出)。

#### 举个栗子：

- 一个 task 的 input 可以是一个 github/gitlab/gitea 地址。
- 一个 task 的 output 可以是一个 image(镜像)。
- 一个 task 的 output 可以输出一个 jar 包,golang 的二进制文件，dotnet 的二进制文件，rust 的二进制执行程序等。


## Task 资源简介

> 一个 task 代表这一次任务，task 里面可以配置多个 step(多个步骤)。step 按照你配置的顺序执行。一个做完才到下一个，如果一个 step 出现了错误，后面的步骤将不会执行。也没必要执行。


## pipeline 资源简介

> 一个 pipeline 可以包含多个 task，定义你的执行顺序，按照你的 task 的顺序执行 task，也可以支持并行 task。

举个栗子：

```
        |            |
        v            v
     test-app    lint-repo
    /        \
   v          v
build-app  build-frontend
   \          /
    v        v
    deploy-all

```

为了更生动形象的表示 用 golang 为代码表示为如下：

```
func task1(inputs,outputs Resource) {}
func task2(inputs,outputs Resource) {}
func task3(inputs,outputs Resource) {}
func pipeline(inputs,outputs Resource){
    task1(inputs,outputs);
    task2(inputs,outputs);
    task3(inputs,outputs);
}

func pipeineRun(){
    var inputs,outputs Resource
    inputs="git"
    outputs="image"
    pipeline(inputs,outputs)
}

func main(){
    pipeineRun()
}

```


### PipelineRun 资源简介

> 一个 pipelierun 允许你指定并执行你定义的 pipeline，并且按照你 pipeline 定义的任务按顺序执行。



以上定义了这么多资源，是为了更好的抽象，更好的复用，更好的职责分离。

### 我们是怎么集成的呢？

> 我们的隔离颗粒度是以部门为单位，每个部门下人只能看见他们部门的ci构建，部门与部门之间相互隔离，部门按照namespaces划分。每个部门的ci资源的限制等。



![image](https://user-images.githubusercontent.com/22409551/88407827-4110cd80-ce05-11ea-923d-01db6ef6aa86.png)

为了方便用户操作，我们给用户一个大的graph，让用户尽情的发挥，动态的增加node(任务)，编排自己的pipeline。

<br/>


![pipeline-run](https://imgkr.cn-bj.ufileos.com/241c79ce-7b02-4baf-b204-a95194924df1.png)

在pipelinerun模块上上可以按到自己的任务构建的状态。


![](https://imgkr.cn-bj.ufileos.com/011cc742-ddd3-4756-8055-887648c6aa15.png)

任务跑成功后的状态。

![](https://imgkr.cn-bj.ufileos.com/016105d4-1238-49d6-8d80-a265606e01d6.png)

日志显示方式。

![](https://imgkr.cn-bj.ufileos.com/14e78d0c-2c1c-4b34-8025-f26ee0164784.png)

整体的pipeline的跑的状态，包括创建时间，耗时等。

上面我省略了git仓储和镜像仓储的配置，还有pipeline的配置，正式这些复杂的配置有了我们后续的优化方式。


说下缺点：

1. 繁琐的配置，对普通开发不能快速上手。
2. 不支持trigger操作。
3. 不支持多语言的模板。
4. 构建成功失败，不能够通知用户。
5. 不支持概览
6. 不支持某个task的暂停功能。

针对以上缺点，说下思路是如何去优化。

1. 每个任务下需要配置繁琐的step。就是类似kaniko构建就需要很多的配置了，针对step做一个step的crd资源，复用step操作。支持用户的step编辑step，上传step模板到仓储里面。

2. devOps工程师提供多语言task的最佳实践模板，用户直接fork一份task对自己的业务任务进行改造，并且编排。


3. 封装一CRD编排以上五种资源，传入一些对应的params(包括 git和image)整个pipeline启动。用户不关新task和resource和pipelin的编排。

4. 支持gitOps和chatOps。

5. pipeline的构建成功和失败，可以通过informer的形式，watch其pipelinerun资源的成功和失败的实践。作出响应的通知。

6. tekton支持了prometheus的收集。根据你需要支持的纬度进行支持即可。
   


总结: CI的道路上任重而道远，目前这块还是比较多毛刺，需要时间的磨合，也需要更多用户的反馈，贴切用户的使用，才能作出更好的产品，共勉。


