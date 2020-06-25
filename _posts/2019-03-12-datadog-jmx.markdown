---
title:  "Collect jmx metrics of a Kubernetes deployment with datadog"
date:   2019-03-12 16:36:37 +0100
categories: Datadog Monitoring Jmx Kubernetes
tags:
  - Devops
  - Datadog
---
Since 2018 My team decided to swich to Datadog to collect the application related metrics. 

We use Kafka as our message queuing service, and we have Kafka connect/Kafka stream applications(JVM based)to do the realtime Data enrichment/transformation/unload. Our applications are deployed on AWS managed Kubernetes clusters. In Kafka connect/stream framework JMX metrics are available by default. so, we need to collect those metrics constantly and continually with Kubernetes pods restarts or with the application redeployment.

Datadog has Kubernetes integration which allows to collect application JMX metrics with the service auto-discovery. You need to run [Kubernetes daemon-set in your cluster](https://docs.datadoghq.com/agent/kubernetes) to act as a metric collection agent from pods. And you need to follow their [JMX guide](https://docs.datadoghq.com/integrations/java) to configure your pod annotations  Of course it was not as easy as it has been advertised. There were many bugs on their jmxfetcher back then, after quite some back and forth with Datadog Devs, we managed to fix many bugs and finally have a relatively satifactory integration.

## Java app side configurations
For a kafka connector app we have following datadog jmx integration configuration :
```yaml
{% raw  %}
instances:
  - jmx_url: service:jmx:rmi:///jndi/rmi://%%host%%:9999/jmxrmi
    name: {{ include "kafka-connect-helm.fullname" . }}
    tags:
    - kafka:connect
    - env:{{ .Values.ENV }}
    - app:{{ include "kafka-connect-helm.fullname" . }}
logs:
  - type: file
    path: /var/log/kafka/server.log
...
{% endraw %}
```

the whole config file is [here](https://github.com/mrmuggymuggy/kafka-connect-s3offload/blob/master/kafka-connect-helm/monitoring/custom_metrics.yaml)

Datadog autodiscovery relies on the datadog daemon-set to fetch upon jmx configuration from the kubernetes deployment annotations, so you need to transform upon yaml file to json format in the annotations.

```yaml
{% raw  %}
   {{- if and (.Values.datadog.enabled) }}
      annotations:
        iam.amazonaws.com/role: '{{ .Values.kafka_connect.CONNECT_IAM_ROLE }}'
        #Logs
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.logs: '{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).logs }}'
        # label for existing template on file
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.check_names: '["{{ include "kafka-connect-helm.fullname" . }}-{{- uuidv4 | trunc 5 -}}"]'  # becomes instance tag in datadog
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.init_configs: '[{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).init_config }}]'
        ad.datadoghq.com/{{ include "kafka-connect-helm.fullname" . }}.instances: '{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).instances }}'
    {{- end }}
    spec:
      volumes:
{% endraw %}
```
the whole config file is [here](https://github.com//mrmuggymuggy/kafka-connect-s3offload/blob/master/kafka-connect-helm/templates/deployment.yaml)

The code snippet use Helm(Go template) language
```jinja
{% raw  %}
'[{{ toJson (fromYaml (tpl (.Files.Get "monitoring/custom_metrics.yaml") .)).init_config }}]'
{% endraw %}
``` 
to transform the yaml configuration to json and put as part of kubernetes deployment annotation at runtime.

{% capture notice-2 %}
Note very important points:
1. The `name` in `ad.datadoghq.com/$name` in the annotation has to be the same as your container name
2. The `ad.datadoghq.com/$name.check_names` better to contain a UUID, the datadog daemon-set stores the checkname and host ip pairs, during a service redeployment, the host ip will change, if the checkname stays the same, the datadog daemon-set will keep check on an outdated host ip and you will lost metrics
{% endcapture %}
<div class="notice">{{ notice-2 | markdownify }}</div>

## Datadog daemonset configurations

Everything works fine with upon setup, but after few weeks, we suddenly experienced missing application metrics, after investigations, we found out that the jmxfetcher process in the datadog daemonset has been constantly killed by our host OOM killer. The Datadog daemonset spawns up the jmxfetcher(Java) process as a sub process with`-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap` jvm options, as the datadog main agent process took more memory over time, at some point Java process won’t be able to allocate memory anymore, thus get killed by the host OOM killer. Datadog has implemented auto-restart for the jmxfetcher, by the time the other processes took up to 80% of memory, it will try to restart the jmxfetcher again and again, thus, at one point we lost jmx metrics. I have implemented a solution based on upon finding to fail the liveness probe of datadog daemonset by patching the existing pod liveness probe tests:

Memory check based on the container cgroup:
[container_memory_check.py](https://github.com/mrmuggymuggy/data-team-bootstrap/blob/master/data-team-bootstrap-helm/monitoring/container_memory_check.py)
```python
#!/usr/bin/env python
import psutil
import sys

CGROUP_MEM_LIMIT_FILE = '/sys/fs/cgroup/memory/memory.limit_in_bytes'

def main():
  mem_total = 0
  f = open(CGROUP_MEM_LIMIT_FILE, "r")
  container_cgroup_limit = float(f.read())
  f.close()

  for p in psutil.process_iter(): 
    p_mem = p.memory_full_info()
    mem_total = mem_total + (p_mem.rss - p_mem.shared)

  mem_used_percent = (mem_total/container_cgroup_limit) * 100
  print("container cgroup limit : {0}, total container process memory : {1}".format(container_cgroup_limit, mem_total))
  print("total memory use percentage {0}".format(mem_used_percent))
  if mem_used_percent > 90:
    sys.exit(1)
  else:
    sys.exit(0)

if __name__== "__main__":
  main()
```
when the total memory of all processes is over 90% of the allocated resource, we will fail the memory check.

Add checks to the end of the datadog liveness probe:
[probe.sh](https://github.com/mrmuggymuggy/data-team-bootstrap/blob/master/data-team-bootstrap-helm/monitoring/probe.sh)
```bash
#!/bin/sh
set -e

/opt/datadog-agent/bin/agent/agent health
python container_memory_check.py
```
you can have a look at my [data-team-bootstrap](https://github.com/mrmuggymuggy/data-team-bootstrap) project, it contains the datadog daemon-set deployment setup.