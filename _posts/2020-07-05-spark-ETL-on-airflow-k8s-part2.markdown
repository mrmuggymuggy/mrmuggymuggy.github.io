---
title:  "Spark ETL -- Airflow on Kubernetes, Part 2"
date:   2020-07-03 16:36:37 +0100
categories: Spark Kubernetes Airflow
tags:
  - Devops
  - Airflow
  - Spark
  - Kubernetes
---

## Introduction
Let's talk about pitfalls and inconveniences of airflow!
It is the following part of this [article]({{ site.url }}{{ site.baseurl }}/spark/kubernetes/airflow/spark-ETL-on-airflow-k8s-part1/)

## Pitfalls and inconveniences
Keep your airflow dag simple!

It is based on my own experience and applies with our airflow setup: 
* Airflow on kubernetes with local executor mode
* Tasks are run with KubernetesPodOperator.

Following are problems I have encountered, there are many more, Those are what mark me the most:
1. Dag's [`dagrun_timeout`](https://airflow.apache.org/docs/stable/_modules/airflow/models/dag.html#DAG) and operator's `execution_timeout` parameter don't apply, because KubernetesPodOperator does not implement [`on_kill`](https://stackoverflow.com/questions/50054777/airflow-task-is-not-stopped-when-execution-timeout-gets-triggered) method.
2. Backfill disregards Dag's `max_active_runs` parameter on retrigger, it is still not fixed upto the time this article has been written, check [here](https://issues.apache.org/jira/browse/AIRFLOW-137)
3. A Dag can be triggered with parameters this way : `airflow trigger_dag 'example_dag' -r 'run_id' --conf '{"key":"value"}'` it does not work when you retrigger backfill. e.g, backfill on day `x` with `--conf '{"key":"value1"}'` rerun backfill on day `x` with `--conf '{"key":"value2"}'` the `key` keeps `value1` 
4. Can't configure kubernetes ephemeral storage resource with KubernetesPodOperator
5. Dag run history has not been clearly properly. During dag development, I adjust dag or tasks parameters frequently, it messes up with the scheduler database, some dag run can't be marked as failed or cleared, even after deleting and redeploying dag, problems still persist. I need to run sql clean query against airflow instance database to solve problems
6. Kubernetes attached volume name is not a templated field. Spark job use large disk space(when memory is not enough). thus to attach volume to the kubernetes pod. it is problematic for parallel backfilling. e.g we want to reprocess(backfill) in parallel from day `01-01-2020` to `04-01-2020`, I create volumes with name : `{% raw  %}"spark-pv-claim-{{ ds_nodash }}{% endraw  %}"`, it results to 3 volumes :  `"spark-pv-claim-20200101"`, `"spark-pv-claim-20200103"`, `"spark-pv-claim-20200103"` since we can't configure persistentVolumeClaim name as a templated field with KubernetesPodOperator you can not dynamically attach volumes.

Solutions and workaround for upon problems:
1. Use bash timeout command, configure  KubernetesPodOperator with `"cmds": ["timeout", "45m", "/bin/sh", "-c"],` gives you 45 minutes timeout of spark task.
2. Make dag executions independent, no dependencies to the past, as the `max_active_runs` problem only happens for backfilling, clear dag history before retrigger.
3. Use airflow variables instead. e.g we have a `force_reprocess` in the dag, set variable before backfill `airflow variables --set force_reprocess false;` backfill with `airflow backfill "historical_reprocess" --start_date "y" --end_date "y" --reset_dagruns -y;` in the dag I get the variable value with `force_reprocess = Variable.get("force_reprocess", default_var=True)` you need to remember to delete variable after backfill.
4. Airflow version from [1.10.11](https://github.com/apache/airflow/blob/1.10.11rc1/airflow/kubernetes/pod.py) will have ephemeral storage support.
5. run sql clean query against the airflow database, you can find queries in this [post](https://www.astronomer.io/guides/airflow-queries/)
6. This feature is crucial for us, so I extended KubernetesPodOperator to manage volume dynamically. I added a new templated field as `"pvc_name"`

```python
from airflow.contrib.operators.kubernetes_pod_operator import KubernetesPodOperator
from airflow.contrib.kubernetes.volume import Volume
from airflow.contrib.kubernetes.volume_mount import VolumeMount
from airflow.utils.decorators import apply_defaults


class KubernetesPodWithVolumeOperator(KubernetesPodOperator):
    template_fields = (*KubernetesPodOperator.template_fields, "pvc_name")

    @apply_defaults
    def __init__(self, pvc_name=None, *args, **kwargs):
        super(KubernetesPodWithVolumeOperator, self).__init__(*args, **kwargs)
        self.pvc_name = pvc_name
sometime
    def execute(self, context):
        if self.pvc_name:
            volume_mount = VolumeMount(
                "spark-volume", mount_path="/tmp", sub_path=None, read_only=False
            )

            volume_config = {"persistentVolumeClaim": {"claimName": f"{self.pvc_name}"}}
            volume = Volume(name="spark-volume", configs=volume_config)

            self.volumes.append(volume)
            self.volume_mounts.append(volume_mount)
        super(KubernetesPodWithVolumeOperator, self).execute(context)

```
Usage of the custom operator:
```python
{% raw  %}
    historical_process = KubernetesPodWithVolumeOperator(
        namespace=os.environ['AIRFLOW__KUBERNETES__NAMESPACE'],
        name="historical-process",
        image=historical_process_image,
        image_pull_policy="IfNotPresent",
        cmds=["/bin/sh","-c"],
        arguments=[spark_submit_sh],
        env_vars=envs,
        task_id="historical-process-1",
        is_delete_operator_pod=True,
        in_cluster=True,
        hostnetwork=False,
        #important env vars to run spark submit
        pod_runtime_info_envs=pod_runtime_info_envs,
        pvc_name="spark-pv-claim-{{ ds_nodash }}",
    )
{% endraw  %}
```

## Conclusion
Despite of many inconveniences and pitfalls, our pipelines run fine so far with the Airflow scheduler. We achieved huge cost reduction by moving away from AWS Glue. Historical data reprocessing can be done in a single click with backfilling. Realtime data process and the historical data process are in the same code base, Data pipeline steps are now transparent with Dag definition in python.

I will in the future study other alternatives such as [perfect core](https://docs.prefect.io/), [kedro](https://kedro.readthedocs.io/en/stable/index.html) or [dagster](https://github.com/dagster-io/dagster/). Maybe there are better tools out there.