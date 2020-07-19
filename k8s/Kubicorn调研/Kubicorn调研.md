# Kubicorn调研

[TOC]

## 1 kubicorn是什么

一个在不同云上部署kubernetes集群的工具，以kubeadm为基础。最大的特点是提供snapshot的能力，包括基础设施层与应用层。

最新issue是19年10月提出的，最新的答复是18年10月，仿佛已经不会再更新了。

## 2 kubicorn如何运行

术语：

1. **Cluster API**：Kubernetes集群的集群API通用表示(与云服务商无关)。它定义了集群的每个部分，包括基础设施部分，如虚拟机、网络单元和防火墙。集群的通用表示与特定云服务商的表示的匹配是协调过程（reconciling process）的一部分。

2. **state**：特定Kubernetes集群的表示。 在调节过程（reconciling process）中用于创建集群。

3. **state store**：存储状态的地方。 当前，我们仅支持磁盘上的状态存储以及YAML或JSON格式。 我们期待实现Git和S3状态存储。

4. **reconciler**：项目和供应逻辑所在的核心。 它将对云服务商透明的通用集群定义与特定的云服务商定义进行匹配，该定义用于配置集群。 它负责配置新集群，销毁旧集群并保持actual和expected状态之间的一致性。

5. **bootstrap scripts**：启动脚本作为创建集群时的用户数据提供，以安装依赖项并创建集群。 它们以Bash脚本的形式提供，因此您无需Go语言即可轻松创建它们。 您还可以根据需要在协调过程（reconciling process）中注入值。

6. **profile**：配置文件是用Go编写的集群的唯一表示。 配置文件包含创建集群所需的所有信息，例如：集群名称，云服务商，VM大小，SSH密钥，网络和防火墙配置。

   ***

reconciler：

1. Reconciler的首要任务是将集群的通用表示转换为特定云服务商的表示。 此过程称为渲染（rendering），在model.go文件以及特定于云的Reconciler的`immutableRender()`函数中定义。 
2. 集群渲染成功后，我们可以使用`Expected()`函数轻松获得集群的期望状态；同时使用`Actual()`函数获取实际状态。
3. Reconciler最重要的任务是使用`Apply()`函数按照集群定义申请资源。 首先，Reconciler通过比较实际状态和预期状态来检查集群是否已存在，如果是，则在此处停止，返回已存在的集群。 如果不存在，我们将继续进行创建，该部分包括：
   1. 构建启动脚本并在运行时注入值，
   2. 为主节点创建适当的资源，
   3. 获取创建工作节点所需的信息，
   4. 创建工作节点。
   5. 从主节点下载集群的.kubeconfig文件。
4. 至此，我们已经拥有功能齐全的Kubernetes集群。除了创建集群之外，Reconciler还负责销毁集群，这由`Delete()`函数完成。

## 3 添加新的云服务商步骤

1. 确定云服务商的名字，例如“aliyun”。

2. 在apis/cluster/cluster.go中创建一个字符串常量`const Cloud<name> = name`，例如`const CloudAliyun = "aliyun"`。

3. 在cloud/下创建一个文件夹放置基础设施的供应代码，文件夹名为步骤1中云服务商的名字。

4. 在profiles/下创建一个文件夹放置配置文件，文件夹名同步骤3。配置文件包括在特定基础设施上构建kubernetes集群所需的关键信息。例如instance size, ssh key。

5. 将云服务商的profile添加到pkg/cli/profiles_map.go的`profileMapIndexed map[string]ProfileMap`中作为配置选项。例如:

   ```go
   "aliyun": {
   		ProfileFunc: aliyun.NewUbuntuCluster,
   		Description: "Ubuntu on Aliyun",
   }
   ```

6. 在pkg/reconciler.go添加云服务商相关信息作为`known.Cloud`。

7. 添加启动脚本到bootstrap/。脚本名为`<provider>_k8s_<profile>_master.sh`和者`<provider>_k8s_<profile>_node.sh`。

***

## 4 代码阅读笔记

### 4.1 Kubicorn架构图

![Kubicorn架构图](/Users/xianghun/data-migrate-from-magic-to-mac/kubicorn调研/images/Kubicorn架构图.png)

### 4.2 Kubicorn组合结构图

![Kubicorn组合结构图](/Users/xianghun/data-migrate-from-magic-to-mac/kubicorn调研/images/Kubicorn组合结构图.png)



***

## 5 可借鉴设计

Reconclier模式，分为集群级与机器级两级，解析通用Cluster API对象期望状态，通过Infrastructure Provider SDK获取云服务商基础设施实际状态，并协调两者差异。
