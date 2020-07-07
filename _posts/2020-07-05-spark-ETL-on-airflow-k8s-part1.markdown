---
title:  "Spark ETL -- Airflow on Kubernetes, Part 1"
date:   2020-06-21 16:36:37 +0100
categories: Spark Kubernetes Airflow
tags:
  - Devops
  - Airflow
  - Spark
  - Kubernetes
---

## Introduction
[Airflow](https://airflow.apache.org/docs/stable/) is nowadays a widely used ETL scheduler. I really like that one can write ETL pipelines in python code. My colleague Laurent said : Airflow is great for simple tasks, don't try to implement complex logic.

The reason we introduce Airflow is to replace AWS Glue for historical data reprocess, to lower AWS cost and achieve code unity of real time / historical data process.

This article is the part two of my last [article](https://mrmuggymuggy.github.io/spark/kubernetes/spark-structure-streaming-on-k8s/) -- Spark data process pipeline on kubernetes, I will not introduce airflow basic notions such as `dag`, `operator`. We are interested in airflow's setup on Kubernetes and the usage of the airflow's [kubernetes pod operator](https://airflow.apache.org/docs/stable/kubernetes.html) for Spark batch ETL data reprocess(The blue rectangle part of the flowchart).

| ![data pipeline with spark on Kubernetes]({{ site.url }}{{ site.baseurl }}/assets/images/spark-k8s/spark_on_k8s_archi.png)
|:--:|
| *Data pipelines with spark on Kubernetes* |

I will also list a few pitfalls and inconveniences of its usage based on my experiences in Part two of this article.

### Airflow Kubernetes executor
Airflow supports various executors, we must mention the kubernetes executor. This [article](https://marclamberti.com/blog/airflow-kubernetes-executor/) gives a detailed explanation of kubernetes executor mode. The official [documentation](https://airflow.readthedocs.io/en/latest/executor/kubernetes.html) gives insides on how it works behind the scene. Unfortunately we could not use this executor mode due to the limitation of AWS EBS. Airflow on kubernetes claims persistent volumes to mount `logs` and `dags`, those volumes are shared between Airflow webUI, schedulers and workers(in this case kubernetes pods), Our kubernetes cluster run on AWS EKS, it uses aws EBS for persistent volumes, AWS EBS can only be mounted to one EC2 instance. Thus it can't be shared between multiple pods of different nodes. we could define the pod affinity of the airflow worker pod to always run on one node, then it lost the scale advantage.

The [tutorial](https://marclamberti.com/blog/airflow-kubernetes-executor/) I pointed earlier, doesn't contain any airflow deployment. You can try out the official [airflow helm chart](https://github.com/helm/charts/tree/master/stable/airflow) with Kubernetes executor, it works for you if your Kubernetes cluster does not rely on AWS EBS.  

### Airflow Local executor
Our Airflow setup is deployed with a helm chart, we use [puckel/docker-airflow](https://hub.docker.com/r/puckel/docker-airflow/dockerfile) as the base docker image, Kubernetes support is then installed in the [Dockerfile](https://github.com/flix-tech/k8s-airflow/blob/master/Dockerfile), we use airflow `1.10.x` version. We choose the local executor for its simplicity, to ensure scalability, all spark tasks are spawned up with kubernetes pod operator(explained in the next section). Only light weight tasks are run in the airflow instance. Thus we take advantage of kubernetes to scale airflow tasks, we don't need to care about airflow workers capacity as many tasks run in parallel, because we don't have any workers!

Checkout our airflow setup [repository](https://github.com/flix-tech/k8s-airflow), it contains a helm chart. it can be run out of box on your local minikube. Compared to the official airflow helm chart, ours is much simpler, the webUI and scheduler are run on the same pod. Use it only with Local executor.

### Airflow Kubernetes operator
**Not to confuse with the Airflow Kubernetes executor,** An airflow operator describes a single task in a workflow. see detailed definition [here](https://airflow.readthedocs.io/en/latest/concepts.html#operators). The Kubernetes pod operator allows you to create your tasks as Pod on Kubernetes. as the Spark ETL tasks run with KubernetesPodOperator, thus the airflow instance acts purely as a scheduler(use very limited resources). example reprocess ETL dag is here :
```python
# -*- coding: utf-8 -*-
"""
This is an example dag for using the Kubernetes Executor.
"""
import os

import airflow
from airflow.models import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.contrib.operators.kubernetes_pod_operator import KubernetesPodOperator
from airflow.operators.bash_operator import BashOperator
from airflow.contrib.kubernetes.pod_runtime_info_env import PodRuntimeInfoEnv

DAG_NAME = "historical_process"
ENV = os.environ.get("ENV")

properties = """
spark.executor.instances=6
spark.kubernetes.allocation.batch.size=5
spark.kubernetes.allocation.batch.delay=1s
spark.kubernetes.authenticate.driver.serviceAccountName=spark
spark.kubernetes.executor.lostCheck.maxAttempts=10
spark.kubernetes.submission.waitAppCompletion=false
spark.kubernetes.report.interval=1s
spark.kubernetes.pyspark.pythonVersion=3
spark.pyspark.python=/usr/bin/python3

#kubernetes resource managements
spark.kubernetes.driver.request.cores=10m
spark.kubernetes.executor.request.cores=50m
spark.executor.memory=500m
spark.kubernetes.memoryOverheadFactor=0.1

spark.kubernetes.executor.annotation.cluster-autoscaler.kubernetes.io/safe-to-evict=true
spark.sql.streaming.metricsEnabled=true
"""

historical_process_image = "dcr.flix.tech/data/flux/k8s-spark-example:latest"

envs = {
"SERVICE_NAME": f"historical_process_{ENV}",
"CONTAINER_IMAGE": historical_process_image,
"SPARK_DRIVER_PORT": "35000",
"APP_FILE": "/workspace/python/pi.py",
}

pod_runtime_info_envs = [
    PodRuntimeInfoEnv('MY_POD_NAMESPACE','metadata.namespace'),
    PodRuntimeInfoEnv('MY_POD_NAME','metadata.name'),
    PodRuntimeInfoEnv('MY_POD_IP','status.podIP')
]

spark_submit_sh = f"""
echo '{properties}' > /tmp/properties;
/opt/spark/bin/spark-submit \
--master k8s://https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT \
--deploy-mode client \
--name $SERVICE_NAME \
--conf spark.kubernetes.namespace=$MY_POD_NAMESPACE \
--conf spark.kubernetes.driver.pod.name=$MY_POD_NAME \
--conf spark.driver.host=$MY_POD_IP \
--conf spark.driver.port=$SPARK_DRIVER_PORT \
--conf spark.kubernetes.container.image=$CONTAINER_IMAGE \
--properties-file /tmp/properties \
$APP_FILE
"""

args = {
    'owner': 'Airflow',
    'start_date': airflow.utils.dates.days_ago(2)
}

with DAG(
    dag_id=DAG_NAME,
    default_args=args,
    schedule_interval='30 0 * * *'
) as dag:

    # Limit resources on this operator/task with node affinity & tolerations
    historical_process = KubernetesPodOperator(
        namespace=os.environ['AIRFLOW__KUBERNETES__NAMESPACE'],
        name="historical-process",
        image=historical_process_image,
        image_pull_policy="IfNotPresent",
        cmds=["/bin/sh","-c"],
        arguments=[spark_submit_sh],
        env_vars=envs,
        service_account_name="airflow",
        resources={
            'request_memory': "1024Mi",
            'request_cpu': "100m"
            },
        task_id="historical-process-1",
        is_delete_operator_pod=True,
        in_cluster=True,
        hostnetwork=False,
        #important env vars to run spark submit
        pod_runtime_info_envs=pod_runtime_info_envs
    )
    historical_process
```
{% capture notice-2 %}
1. When your airflow run on kubernetes, You absolutely need to configure `in_cluster=True` in the KubernetesPodOperator, it is not mentioned in the official documentation
2. You must specify the `namespace` parameter in KubernetesPodOperator, otherwise it will spawn up the task in the default namespace(you may have no permission!!)
{% endcapture %}
<div class="notice">{{ notice-2 | markdownify }}</div>

The historical reprocess dag is in the same code base as the spark streaming job(discussed in my last [article](https://mrmuggymuggy.github.io/spark/kubernetes/spark-structure-streaming-on-k8s/#spark-client-mode)), we achieved the spark realtime and batch process code unity, The spark submit script is in the airflow dag, to have a total transparency on the pipeline execution. you can pack spark submit script and properties into a docker image, to have even simpler airflow dag.

Airflow task launch spark reprocess ETL with the [spark client mode](https://mrmuggymuggy.github.io/spark/kubernetes/spark-structure-streaming-on-k8s/#spark-client-mode). the driver(airflow task instance) then spawn up 6 spark executors to work on data reprocess ETL. it is blading fast.

## Conclusion

I will draw conclusions and list few pitfalls and inconveniences of airflow usage based on my experiences in the [part two]({{ site.url }}{{ site.baseurl }}//spark/kubernetes/airflow/spark-ETL-on-airflow-k8s-part2/) of this article.