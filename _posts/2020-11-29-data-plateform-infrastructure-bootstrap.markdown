## Introduction
The motivation of this article is to show how easily one can setup a resiliant, scalable and budget data processing plateform infrastucture in cloud thanks to Terraform/Terragrunt. In this article, I tackle only AWS cloud provider, I will make the same thing with GCP in an another article.

In my past articles about Spark on kubernetes, it takes a resiliant, scalable infrastructure as granted, it is indeed usualy managed by the company's plateform team. if your organisation does not use Kubernetes company wide, and request such feature would probably take few months. Son't worry, just do it yourself! A Data processing plateform has no intention to manage any incoming traffic, service mesh or dns/ssl certificates configurations, thus it is easy to setup.

## Setup Kubernetes cluster on AWS
If you are not familiar with [Terraform](https://blog.gruntwork.io/an-introduction-to-terraform-f17df9c6d180) or [Terragrunt](https://blog.gruntwork.io/terragrunt-how-to-keep-your-terraform-code-dry-and-maintainable-f61ae06959d8), have a glance on their blogs, it's a powerfull code as infrastructure tool. 

TL;TR, I just want code! check out the entire code base at my [git repository](https://github.com/mrmuggymuggy/terragrunt-dataplaform-bootstrap), it contains a deployment guide.

```
{% raw  %}
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "13.2.1"
  cluster_name    = var.cluster_name
  cluster_version = "1.17"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id
  enable_irsa     = true

  worker_groups_launch_template = [
    {
      name                    = "on-demand-1"
      override_instance_types = var.ondemand_instance_types
      asg_max_size            = var.ondemand_asg_max_size
      kubelet_extra_args      = "--node-labels=node.kubernetes.io/lifecycle=ondemand"
      suspended_processes     = ["AZRebalance"]
      tags = [
        {
          "key"                 = "k8s.io/cluster-autoscaler/enabled"
          "propagate_at_launch" = "false"
          "value"               = "true"
        },
        {
          "key"                 = "k8s.io/cluster-autoscaler/${var.cluster_name}"
          "propagate_at_launch" = "false"
          "value"               = "true"
        }
      ]
    },
    {
      name                    = "spot-1"
      override_instance_types = var.spot_instance_types
      spot_instance_pools     = 4
      asg_max_size            = var.spot_asg_max_size
      asg_desired_capacity    = 1
      kubelet_extra_args      = "--node-labels=node.kubernetes.io/lifecycle=spot --register-with-taints=node-role.kubernetes.io/spot=true:PreferNoSchedule"
      tags = [
        {
          "key"                 = "k8s.io/cluster-autoscaler/enabled"
          "propagate_at_launch" = "false"
          "value"               = "true"
        },
        {
          "key"                 = "k8s.io/cluster-autoscaler/${var.cluster_name}"
          "propagate_at_launch" = "false"
          "value"               = "true"
        }
      ]
    },
  ]

  worker_additional_security_group_ids = [aws_security_group.all_worker_mgmt.id]
}
{% endraw  %}
```

It uses existing [Terraform EKS module](https://github.com/terraform-aws-modules/terraform-aws-eks/), it defines two worker node pools, one with on demande instance to host vital components, one with spot instances to schedule compute units. you can later in your kubernetes deployments to specify which work node pool to target by setting kubernetes [taint and toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

## Deploy the cluster autoscaler and the spot instance handler
Use terraform Helm provider to deploy the cluster autoscaler and the spot instance termination handler.
```
{% raw  %}
resource "helm_release" "cluster-autoscaler" {
  name       = "cluster-autoscaler"
  version    = "8.0.0"
  repository = "https://charts.helm.sh/stable"
  chart      = "cluster-autoscaler"
  namespace  = "kube-system"
}

resource "helm_release" "spot-handler" {
  name       = "spot-handler"
  version    = "1.4.9"
  repository = "https://charts.helm.sh/stable"
  chart      = "k8s-spot-termination-handler"
  namespace  = "kube-system"
}
{% endraw  %}
```
now you have a managed, fully scalable, resiliant Kubernetes cluster on AWS!

### Deploy Airflow
We now deploy an Airflow instance in our cluster to perform Spark streaming jobs or Spark batch ETL as we discussed in my other articles.
```
{% raw  %}
resource "helm_release" "aiflow" {
  name       = "airflow"
  repository = "https://mrmuggymuggy.github.io/helm-charts/"
  chart      = "airflow"
  namespace  = "default"
}
{% endraw  %}
```
I packed my Airflow helm chart into my own [helm repo](https://mrmuggymuggy.github.io/helm-charts/).
Here were are,  a resiliant, scalable and budget data processing plateform infrastucture in AWS!