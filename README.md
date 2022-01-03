# Data-and-model-isolation-on-EKS
## 介绍
在机器学习中，有一种常见的需求：甲方提供数据，乙方提供机器学习模型。甲方要确保数据不被乙方保存副本，而乙方也要保护自己的机器学习模型不被甲方获取。这种场景借助AWS的三个服务可以轻松实现，这三个服务分别是EKS，AWS Resource Access Manager，AWS Organizations。

如下图，先使用AWS Resource Access Manage，将甲方的一个私有子网共享给乙方。并且对取消私有子网的互联网访问权限，在路由表中添加S3 endpoint，并配置权限，这个私有子网中的EC2实例仅可以访问指定指定S3存储桶。并且也在S3存储桶配置，[必须特定的s3 endpoint才能访问这个存储桶](https://aws.amazon.com/cn/premiumsupport/knowledge-center/block-s3-traffic-vpc-ip/)。然后乙方在自己的VPC内创建EKS集群，选择子网时，则选择甲方共享的子网。这样乙方可以通过在甲方共享的子网内创建机器学习推理集群，访问甲方的数据，但数据不能外传。甲方也无法获得乙方的机器学习模型。最后，再使用AWS Organizations SCP，由甲方控制乙方的权限，禁止乙方快照和挂载EBS，防止利用EKS集群复制数据。

![image](arch.png)

下面来实验下具体操作步骤。

## 第一步，使用Resource Access Manage共享子网

默认VPC的子网是不能被Resource Access Manage共享的，所以需要新创建一个VPC。

进入Resource Access Manage服务页面，选择创建新的共享资源。

![image](ram1.png)

如上图，我们选择指定的子网。

![image](ram2.png)

如上图，我们可以看到，共享出的子网，对方仅能在上面运行EC2实例，并没有其他权限。

![image](ram3.png)

如上图，输入一个要分享的AWS account ID。

![image](rtb.png)

我们现在检查下共享出子网的路由表，确定只有本地路由和S3 endpoint路由。如果没有S3 endpoint路由，请创建S3 endpoint。

## 第二步，配置S3 endpoint和存储桶权限

我们需要确保从这个s3 endpoint只能访问到指定的子网。并且也要确定存储桶只能从这个s3 endpoint来访问。

我们先来配置存储桶策略，下面存储桶策略限定了如果要访问DOC-EXAMPLE-BUCKET存储桶，则必须从名为vpce-1111111的S3 endpoint访问。我们把存储桶和s3 endpoint名字都替换为实际名称。
```
{
  "Id": "VPCe",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VPCe",
      "Action": "s3:*",
      "Effect": "Deny",
      "Resource": [
        "arn:aws:s3:::DOC-EXAMPLE-BUCKET",
        "arn:aws:s3:::DOC-EXAMPLE-BUCKET/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": [
            "vpce-1111111"
          ]
        }
      },
      "Principal": "*"
    }
  ]
}
```
继续来配置S3 endpoint权限，下面的权限表示通过这个s3 endpoint，仅可以访问example-bucket存储桶，将example-bucket替换为实际名称。
```
{
  "Effect": "Allow",
  "Action": [
     "s3:ListBucket",
     "s3:GetObject",
     "s3:PutObject"
  ],
  "Resource": [
     "arn:aws:s3:::example-bucket",
     "arn:aws:s3:::example-bucket/*"
  ]
}	
```
