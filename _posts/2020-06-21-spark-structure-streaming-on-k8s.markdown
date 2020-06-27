---
title:  "Spark structure streaming on Kubernetes"
date:   2020-06-21 16:36:37 +0100
categories: Spark Kubernetes
tags:
  - Devops
  - Spark
  - Kubernetes
---

## Introduction
There are many ways to process real time data, In the company I work for, we use Kafka as the message service. Naturally it comes to use Kafka streaming framework(kafka connect/kafka stream), After one year experience, we realize that the limitation is quite big , it can only deal with Kafka input source and we have hard time to efficiently reprocess historical data. In Kafka service point of view, it is a bad practice to persist a big amount of data(more than 30 days), so we persist our data to cloud storage(AWS S3), for the historical data reprocess we need to read from S3, we used AWS Glue to accomplish the task. Hence we have one code base to process the real time data(kafka streaming), one code base for the historical process(AWS Glue). it introduces problems of code base sync. Meanwhile, we were also surprised by the AWS Glue bill.

Idea of Spark makes surface then, it can read from various data sources(Kafka, S3), it does real time streaming(structure streaming) and batch processing. we can have one code base to do everything.

Following chart is the pipeline that we want to implement:

| ![data pipeline with spark on Kubernetes]({{ site.url }}{{ site.baseurl }}/assets/images/spark-k8s/spark_on_k8s_archi.png)
|:--:|
| *Data pipelines with spark on Kubernetes* |

In this Article, I will focus on how to run spark streaming on Kubernetes, it concerns only the red box part of the chart. I will write another article to cover the historical processing

## Spark modes
Spark has native kubernetes support since 2.3, detailed docs can be found [here](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html). we have few options to deploy spark on kubernetes clusters
1. [Cluster mode](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html#cluster-mode)
2. [Client mode](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html#client-mode)
3. [Kubernetes spark operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)
4. [Local mode](https://spark.apache.org/docs/latest/submitting-applications.html#master-urls)

We have experienced all 4, we retained the **client mode** and **local mode** for our usage.

### Spark cluster mode
In the [Spark cluster mode](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html#cluster-mode), `spark-submit` creates a spark driver pod, the driver pod will then create executor pods
* The driver pod need to run on a [servieaccount](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html#rbac) to allow it to create spark executor pods
* The CI/CD runner(or instance) need to contain spark binary to do `spark-submit`
* If the driver pod dies, the app is gone(no restart policy)

As you can see, there are some major drawbacks with this approach. especially the streaming app is supposed for long run, but kubernetes cluster can kill pod sometime on rebalance, so it is an absolute no go

Nevertheless, you can check out this [repo](https://github.com/flix-tech/k8s-spark-cluster-mode-example) if you want to run it.

### Spark client mode
In the [Spark client mode](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html#client-mode), you can pack the spark driver in a kubernetes deployment(or replicatset), it can restart the driver pod on any disruption and CI/CD runner don't need to contain spark binary(simple helm chart deployment). It brought up some other complications though
* Client mode [networking](https://spark.apache.org/docs/2.4.5/running-on-kubernetes.html#client-mode-networking) has to be correct
* Lengthy kubernetes related configurations for spark

To configure the client mode network, you should configure your kubernetes deployments to have environment variables of driver pod IP/name:
```yaml
{% raw  %}
    containers:
        - name: {{ .Release.Name }}
        ....
          env:
            #for client mode network configuration on spark-submit
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            ...
{% endraw  %}
```
Start the spark driver with upon env variables:
```bash
    #spark-submit executed to spin up spark driver
    #MY_POD_NAMESPACE/MY_POD_IP/MY_POD_NAME get from pod inspection in deployment/pod
    #KUBERNETES_SERVICE_HOST/KUBERNETES_SERVICE_PORT are native k8s pod env variables
    #SERVICE_NAME is given by deployment, set as the service name
    #CONTAINER_IMAGE is used for executor pod
    #PROPERTIES_FILE is the spark properties
    #PYTHON_FILE is the spark application file
    /opt/spark/bin/spark-submit \
    --master k8s://https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT \
    --deploy-mode client \
    --name $SERVICE_NAME \
    --conf spark.kubernetes.namespace=$MY_POD_NAMESPACE \
    --conf spark.kubernetes.driver.pod.name=$MY_POD_NAME \
    --conf spark.driver.host=$MY_POD_IP \
    --conf spark.driver.port=$SPARK_DRIVER_PORT \
    --conf spark.kubernetes.container.image=$CONTAINER_IMAGE \
    --conf spark.pyspark.driver.python=/usr/bin/python3 \
    --properties-file $PROPERTIES_FILE \
    --py-files $PY_FILES \
    $PYTHON_FILE
    ;;
```
{% capture notice-2 %}
You absolutely need to configure consistent
spark.kubernetes.namespace, spark.kubernetes.driver.pod.name, spark.driver.host, spark.driver.port values
{% endcapture %}
<div class="notice">{{ notice-2 | markdownify }}</div>
Kubernetes related spark properties:

```
spark.kubernetes.pyspark.pythonVersion=3
spark.kubernetes.allocation.batch.size=10
spark.kubernetes.allocation.batch.delay=1s

spark.kubernetes.authenticate.driver.serviceAccountName=spark
spark.kubernetes.executor.lostCheck.maxAttempts=5
spark.kubernetes.submission.waitAppCompletion=false
spark.kubernetes.report.interval=1s
spark.kubernetes.local.dirs.tmpfs=true
spark.kubernetes.container.image.pullPolicy=Always

spark.executor.instances=10
spark.kubernetes.driver.request.cores=100m
spark.kubernetes.executor.request.cores=1
spark.kubernetes.executor.limit.cores=2
spark.driver.cores=3
spark.driver.memory=16g
spark.executor.cores=2
spark.executor.memory=8g
spark.kubernetes.memoryOverheadFactor=0.1
spark.task.cpus=1
```

As you can see, configurations are lengthy, you can checkout our [repo](https://github.com/flix-tech/k8s-spark-example) to run it on the local minikube or deploy to the cloud.

### Kubernetes spark operator
[It](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator) implements a set of yaml configurations to deploy your spark application. it is simple and clean, unfortunately it asks too many [permissions](https://github.com/helm/charts/blob/master/incubator/sparkoperator/templates/spark-operator-rbac.yaml) to our kubernetes cluster, for the security reasons, we discard it. I highly recommend you to have a look at the project and examples.

### Spark local mode
The simplest and probably fits 90% of your usages. We have considered this option after a few months running with spark client mode. At some point I realized that we can take advantage of the kubernetes pod cpu/memory adjustment feature to scale spark applications, the uplimit of the Spark in local mode is the underline single kubernetes node capacity, and in most cases we don't need that much. By simply packing a spark standalone into Kubernetes deployment, we gain following advantages:

* No extra serviceaccount to setup
* Driver and executors in the same pod -- no data exchange between driver and executors over network
* Driver and executors share same heap space for better RAM usage
* No lengthy spark configurations
* No complicated client network configurations

Spark startup script:
```bash
    /opt/spark/bin/spark-submit \
    --name $SERVICE_NAME \
    --conf spark.pyspark.python=/usr/bin/python3 \
    --properties-file $PROPERTIES_FILE \
    --py-files $PY_FILES \
    $PYTHON_FILE
    ;;
```
Spark properties:
```yaml
spark_properties: |-
  #set spark master local when spark_driver.mode is local, set number of executors in `[]`
  spark.master=local[6]
  spark.driver.memory=6g
```
for the spark app resource configuration, simply set kubernetes resources:
```yaml
  resources:
    requests:
      cpu: 1
      ephemeral-storage: 1Gi
      memory: 6Gi
    limits:
      memory: 12Gi
```
As you can see, it is much simpler than with spark client mode

We only keep spark on client mode for the historical data reprocess, as it can dispatch executors to many Kubernetes nodes for better parallelism.

you can checkout our [repo](https://github.com/flix-tech/k8s-spark-example) to run it on the local minikube or deploy to cloud

## Conclusions and follow up
Running spark on Kubernetes is very convenient, you can forget about cluster capacity provision as run on Yarn or Mesos.
You can choose from different operation modes. Here is a table to help you to make decisions

| Spark modes       | Configurations |Resillient |Permissions| Parallelism  |
| :-------------:   |:-------------: | :-----:   | :----: |:-----:       |
| Cluster mode      | Lengthy        | No   |Admin on namespace   |Uplimit to cluster capacity|
| Client mode       | Lengthy        | Yes  |Admin on namespace| Uplimit to cluster capacity|
| k8S Spark operator| Clean and neat | Yes  |Admin on cluster(for setup) |Uplimit to cluster capacity|
| Local mode        | Clean          | Yes  |No |Uplimit to single k8s node |

Maybe I will make an article for a deeper explanation of my Spark deployment Helm chart in the future. 