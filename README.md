# Use-Multus-CNI-to-enable-DPDK-on-EKS

## 免责声明

建议测试过程中使用此方案，生产环境使用请自行考虑评估。

当您对方案需要进一步的沟通和反馈后，可以联系 nwcd_labs@nwcdcloud.cn 获得更进一步的支持。

欢迎联系参与方案共建和提交方案需求, 也欢迎在 github 项目 issue 中留言反馈 bugs。

Amazon Elastic Kubernetes Service (Amazon EKS) 在2021年8月宣布支持 Multus 容器网络接口 (CNI) 插件，使客户能够附加多个网络接口并将高级网络配置应用于基于 Kubernetes 的应用程序。 凭借 Multus 的可用性，通信服务提供商和其他具有独特网络要求的客户可以配置他们的 EKS 集群来运行连接到多个接口的多宿主 Kubernetes pod。

在这篇文章中，我们将介绍 Multus CNI，涵盖适用的用例，并演练如何部署和设置 Multus CNI。 

## 什么是Multus CNI？

CNI 是容器网络接口，它提供应用程序编程接口来配置容器中的网络接口。 [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) 是 Kubernetes 的容器网络接口插件，可以将多个网络接口附加到 pod。 在 Kubernetes 中，除了本地环回之外，每个 pod 默认只有一个网络接口。 使用 Multus，您可以创建具有多个接口的多宿主 Pod。 Multus 充当“元”插件，可以调用其他 CNI 插件来配置其他接口。 

Pod 的多个网络接口在各种用例中都很有用； 例子包括： 
* 流量拆分：运行需要分离控制/管理和数据/用户平面网络流量以满足低延迟服务质量 (QoS) 要求的网络功能 (NF)。
* 性能：附加接口通常利用专门的硬件规范，例如单根 I/O 虚拟化 [(SR-IOV)](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin) 和数据平面开发套件 [(DPDK)](https://www.dpdk.org/)，它们绕过操作系统内核以提高带宽和网络性能。
* 安全性：支持对流量隔离要求严格的多租户网络。 将多个子网连接到 Pod 以满足合规性要求。 

## Multi-homed pod
Multus CNI 插件允许 Pod 在 Kubernetes 中拥有多个接口。对 Multus 的 EKS 支持的当前版本捆绑了 Amazon VPC CNI 作为默认委托插件（这是唯一受支持和验证的默认委托插件）。默认委托插件为 Pod 配置主网络接口 (eth0) 以启用 Kubernetes 控制平面流量，包括用于 Pod 主网络接口的 IP 地址管理 (IPAM)。

以下是多宿主 Pod 如何在 AWS 上工作的示例。下图显示了具有两个网络接口 eth0 和 net1 的两个 Pod。在这两种情况下，Amazon VPC CNI 管理 pod eth0（默认 Multus 委托）。接口 net1 由 Multus 通过 pod1 的 ipvlan CNI 插件管理，该插件处理与 k8 控制平面流量分离的用户平面（例如：语音、视频）流量。其中 pod2 net1 通过主机-设备 CNI 插件连接到主机弹性网络接口，并启用 DPDK 加速数据包处理。 

![image](image-2021-07-29T013301.821.png)

现在让我们看看如何在 Amazon EKS 集群上设置 Multus CNI。 我们将通过两个示例应用程序之间的简单 ping 测试来演示流量拆分场景。 我们将设置 ipvlan CNI 来管理 Pod 的辅助接口。 ping 测试将针对 ipvlan 插件提供并由 Multus 管理的网络执行。

在此设置中，我们使用 CloudFormation 模板来创建基础架构、EKS 集群和自我管理的节点组。 基础设施模板预配 Amazon Virtual Private Cloud (VPC)、集群和 Multus 的公有和私有子网，以及执行 EKS 操作的堡垒主机。 节点模板创建附加 ENI 的工作节点，以运行多宿主 Pod。 这两个 AWS CloudFormation 模板共同创建了以下资源： 

* 基础设施创建模板 
    - VpcCidr：将用于部署的 VPC CIDR。
    - AvailabilityZones：根据 EKS 要求至少有两个可用区。
    - PublicSubnet1/2：这些子网将托管堡垒主机以运行 kubectl 命令。 此外，这将托管 NAT-GW，为私有子网提供互联网访问。
    - PrivateSubnetAz1/2：AZ1 和 AZ2 中 EKS 控制平面的子网。
    - MultusSubnet1Az1/2：Multus 将用于在示例 Pod 中创建辅助接口的第一个子网。
    - MultusSubnet2Az1/2：Multus 将用于在示例 Pod 中创建辅助接口的第二个子网。
    - BastionInstance：我们可以从 (kubectl) 运行 EKS 集群操作的堡垒主机。
    - EksCluster：将运行示例工作负载的 EKS 集群。 
* EKS 工作节点组创建模板 
    - NodeGroup：运行示例 Pod 的工作节点组。
    - LambdaAttachCNI：Lambda 函数，用于将额外的 Multus 子网连接到工作节点。
    - EventbridgeEventRule：CloudWatch 事件规则，用于监控实例向上和向下扩展以触发 Lambda 钩子将额外的弹性网络接口 (ENI) 从 Multus 子网附加到工作程序节点组。 

![image](arch2.png)

## 先决条件
* 具有管理员权限的 AWS 账户：在本博客中，我们假设您已经拥有一个具有管理员权限的 AWS 账户。
* 使用 AWS 管理控制台创建 EC2 密钥对（EC2 用户指南中提到了这些步骤）。
* 命令行工具：Mac/Linux 用户需要在他们的工作站上安装最新版本的 AWS CLI、aws-iam-authenticator 和 git。而 Windows 用户可能希望在 AWS 中使用 Cloud9 环境，然后在他们的 Cloud9 环境中安装这些 CLI。
* 要开始安装 Multus，请在本地工作站或 Cloud9 实例上克隆 eks-install-guide-for-multus github 存储库。
* 在您最近克隆存储库的本地文件夹中找到 eks-install-guide-for-multus/cfn/templates/cfn/nodegroup/lambda_function.zip 文件。在 AWS 管理控制台中导航到 S3 并创建一个名为 eks-multus-cluster 的存储桶。选择与 EKS 集群相同的 AWS 区域，其余保持默认。在新形成的存储桶上单​​击“上传”。点击“添加文件”。上传 lambda_function.zip。在运行“供应工作节点组”部分中描述的工作节点组的 CloudFormation 时，S3 存储桶名称将用作输入参数之一。 

## 第 1 步：创建 VPC 和 EKS 集群 

运行 CloudFormation 以创建基础架构
1. 使用您的管理员权限登录 AWS 控制台，然后转到 CloudFormation。
2. 单击创建堆栈 → 使用新资源（标准）。
3. 上传模板文件并选择“eks-install-guide-for-multus/cfn/templates/infra/eks-infra.yaml”。
4. 输入堆栈名称“eks-multus-cluster”，堆栈名称也将是 EKS 集群名称。
5. 从下拉列表中选择两个可用区。
6. 使用默认 VPC CIDR (10.0.0.0/16) 块和子网范围。
7. 选择堡垒实例类型（默认，t3-medium 也可以）。
8. 从下拉列表中选择 EC2 密钥对名称。
9. 在本练习中，您可以使用默认的 AMI Id。
10. 单击下一步 → 我确认 → 创建堆栈。 

等待 CloudFormation 堆栈完成（CREATE COMPLETE）。 创建了一个具有两个公有和私有 EKS 子网、四个 Multus 子网（每个可用区两个）、一个 EKS 集群、IGW 和 NAT-GW 的 Amazon VPC。 该堆栈还为 Multus 子网和控制平面安全组构建安全组。 

从名为“eks-multus-cluster”的堆栈的 CloudFormation 控制台输出记录 *EksCluster*、*EksControlSecurityGroup*、*PrivateSubnetAz1/2*、*MultusSubnet1/2Az1/2*、*MultusSecurityGroup* 和 *VpcId*。 此外，从基础架构输出记录 *BastionPublicIp*。 您将在下一节中需要此 IP。 

## 堡垒主机配置 
**要使用 SSH 连接到您的堡垒：**

在终端窗口中，使用 ssh 命令连接到实例。 指定私钥（.pem）的路径和文件名，即上一步记录的公网ip。 要连接到您的实例，请输入以下命令。 
```
ssh -i /path/my-key-pair.pem ec2-user@BastionPublicIp
```

您将看到如下响应： 
```
The authenticity of host 'ec2-198-51-100-1.compute-1.amazonaws.com (198-51-100-1)' can't be established.
ECDSA key fingerprint is l4UB/neBad9tvkgJf1QZWxheQmR59WgrgzEimCG6kZY.
Are you sure you want to continue connecting (yes/no) 
```

输入"Yes"
**配置AWS CLI**

您可以使用临时凭证来运行与管理员配置文件链接的 AWS CLI 命令。 您将使用自己的密钥（以下密钥不适用于您的环境）。 
```
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEX...
export AWS_DEFAULT_REGION=us-west-2
export AWS_SESSION_TOKEN=FJ9BrbSXgCN4XYxFin+QYXsG8L6+lMMkMIv9xwZC48UqpwII9P.....
```

使用以下命令确认凭据。 
```
aws sts get-caller-identity
```

输出：
```
{
"Account": "my-account",
"UserId": "CROAY6YKLVK242KL6D5W3:my-user-id",
"Arn": "arn:aws:sts::my-account:assumed-role/Admin/my-user-id"
}
```
**安装配置 kubectl**
```
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
curl -o kubectl.sha256 https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl.sha256
openssl sha1 -sha256 kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version —short —client
```
**使用AWS CLI创建kubeconfig :**
```
aws eks update-kubeconfig --name eks-multus-cluster
kubectl get svc
```
输出
```
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 172.20.0.1 <none> 443/TCP 31m
```
## 第 2 步： 创建节点组

作为此步骤的一部分，我们将创建自我管理的节点组。 此步骤需要在先决条件下创建的 S3 存储桶以及基础设施堆栈是完整的。 或者，在新选项卡中打开基础架构堆栈 CloudFormation 输出控制台。 节点组堆栈将使用这些输出。 

**运行 CloudFormation 模板以创建节点组 **
1. 使用您的管理员权限登录 AWS 管理控制台并转到 CloudFormation。
2. 单击创建堆栈 → 使用新资源（标准）。
3. 指定模板，并上传模板文件。
4. 从本地存储库位置选择 [eks-install-guide-for-multus/cfn/templates/node-group/eks-nodegroup-multus.yaml](https://github.com/aws-samples/eks-install-guide-for-multus/blob/main/cfn/templates/nodegroup/eks-nodegroup-multus.yaml) 文件。
5. 从之前的堆栈创建中指定集群名称和集群控制平面安全组
6. 输入堆栈名称 multus-cluster-ng01。
7. 输入 multus-cluster-ng01 作为节点组名称。
8. 为节点自动缩放组所需容量、最大和最小大小指定 1。
9. 选择 c5.large 作为实例类型，选择 20 作为卷大小。
10. 使用默认节点 ImageId SSM 参数并留空以使用默认 EKS AMI。
11. 选择在先决条件部分下创建的 EC2 密钥对。
12. 您可以使用 bootstrap 的默认参数。
13. 选择 vpc-eks-multus-cluster VPC Id。
14. 选择将在其中创建工作程序的 EKS 子网 (privateAz1-eks-multus-cluster)。
15. 指定 Multus 子网（multus1Az1-eks-multus-cluster、multus2Az1-eks-multus-cluster）。
16. 输入 Multus 安全组，eks-multus-cluster-MultusSecurityGroup*
17. 将 lambda S3 存储桶名称指定为 eks-multus-cluster，将 S3 密钥指定为 lambda_function.zip。
18. 单击下一步 → 我确认(I Acknowledge) → 创建堆栈。 

等待 CloudFormation 堆栈完成（CREATE COMPLETE）。 为了从定义的 Multus 子网附加 ENI，节点组堆栈部署 AWS Lambda 函数和 Amazon CloudWatch 事件规则。 该堆栈启动带有从 Multus 子网附加的 ENI 以及标签 *no_manage: true* 的 EC2 实例。 AWS VPC CNI 不会管理 ENI 的标记 *no_manage: true*。 这是 Multus 为 Pod 管理额外网络的必要步骤。 

从 CloudFormation 控制台输出记录 **NodeInstanceRole**。

**应用 K8s ConfigMap 更新  **

登录 bastion 主机，您可以在其中运行 kubectl 命令。

下载、编辑和应用 AWS authenticator配置映射： 

```
curl -o aws-auth-cm.yaml https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml
```

使用您喜欢的文本编辑器打开文件。 将“rolearn”替换为 NodeInstanceRole（工作节点组 CloudFormation 堆栈的输出）并保存文件。 不要修改此文件中的任何其他行。 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::my-account:role/worker-nodegroup-01-NodeInstanceRole-1M3F6VK25IKB0
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```


应用配置。 此命令可能需要几分钟才能完成。 
```
kubectl apply -f aws-auth-cm.yaml
```

观察节点的状态并等待它们达到就绪状态。 
```
kubectl get nodes --watch
```

## 第 3 步：安装和配置 Multus 

**使用daemonset安装 Multus CNI **

登录 bastion 主机，您可以在其中运行 kubectl 命令。

运行以下命令下载并安装 Multus daemonset。 此命令将 AWS VPC CNI 配置为 Multus CNI 的默认委托插件。 

```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/multus/v3.7.2-eksbuild.1/aws-k8s-multus.yaml
```

通过运行以下命令验证部署。 每个节点都应该有一个名为 kube-multus-ds 的 pod。 

```
kubectl get pods -n kube-system
```

**创建额外的接口**

接下来，我们将为添加到 pod 的每个附加接口创建配置。 Multus 提供名为 NetworkAttachmentDefinition 的自定义资源定义 (CRD)。 我们将使用这个 CRD 来构建额外的界面设置。 

**创建host-device-1**

使用 host-device CNI 为 pod 配置一个额外的接口（来自 Multus 子网 10.0.4.0/24）。 将配置应用到集群。 

```
cat <<EOF | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: host-device-1
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "host-device",
      "master": "eth1",
      "mode": "l3",
      "ipam": {
        "type": "host-local",
        "subnet": "10.0.4.0/24",
        "rangeStart": "10.0.4.70",
        "rangeEnd": "10.0.4.80",
        "gateway": "10.0.4.1"
      }
    }'
EOF
```

通过运行以下命令验证配置。
```
kubectl describe network-attachment-definitions
```

要了解有关网络连接的配置选择和示例的更多信息，请参阅 [Multus 使用指南](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md)。 

由于官方的案例并为提供DPDK应用范例，所以DPDK网络接口测试需要客户自行提供应用测试。另外客户需要自行修改Node AMI去打开DPDK功能，或直接使用marketplace中的EKS Optimized with DPDK AMI.

## 结论

在这篇文章中，我们介绍了 Multus CNI 及其可能的用例。我们还使用 Multus CNI 创建了一个 Amazon EKS 集群，并为示例 Pod 配置了额外的网络定义以显示流量拆分。

请注意，当前对 Multus CNI 的支持不包括对 pods 辅助网络接口或其他更高阶接口的本机配置支持。这包括工作节点上 Multus 管理接口的 IPAM（ENI 标记为 no_manage: true）以及与更高阶 pod 接口相关联的 CNI 驱动程序和配置。在我们继续为 EKS 客户改进 Multus 体验的同时，请在 GitHub 上提供的 AWS 容器路线图上提供反馈并建议新功能。


