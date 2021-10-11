# jmx-tomcat-exporter
Quickly deploy jmx exporter in Kubesphere v3.0.0+

## Install Kubesphere dashboard CRDs if lack of these CRDs

```
kubectl  apply -f https://raw.githubusercontent.com/zhu733756/jmx-exporter/master/CRD/monitoring.kubesphere.io_clusterdashboards.yaml
kubectl  apply -f https://raw.githubusercontent.com/zhu733756/jmx-exporter/master/CRD/monitoring.kubesphere.io_dashboards.yaml
```

## Modify prometheus-jmx-config.yaml

Refer to https://grafana.com/grafana/dashboards/8563
```
---   
lowercaseOutputLabelNames: true
lowercaseOutputName: true
whitelistObjectNames: ["java.lang:type=OperatingSystem"]
blacklistObjectNames: []
rules:
  - pattern: 'java.lang<type=OperatingSystem><>(committed_virtual_memory|free_physical_memory|free_swap_space|total_physical_memory|total_swap_space)_size:'
    name: os_$1_bytes
    type: GAUGE
    attrNameSnakeCase: true
  - pattern: 'java.lang<type=OperatingSystem><>((?!process_cpu_time)\w+):'
    name: os_$1
    type: GAUGE
    attrNameSnakeCase: true
```

Combine with [tomcat.yaml](https://github.com/prometheus/jmx_exporter/blob/master/example_configs/tomcat.yml)
```
---   
lowercaseOutputLabelNames: true
lowercaseOutputName: true
rules:
- pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+-\w+)-(\d+)\"><>(\w+):'
  name: tomcat_$3_total
  labels:
    port: "$2"
    protocol: "$1"
  help: Tomcat global $3
  type: COUNTER
- pattern: 'Catalina<j2eeType=Servlet, WebModule=//([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), name=([-a-zA-Z0-9+/$%~_-|!.]*), J2EEApplication=none, J2EEServer=none><>(requestCount|maxTime|processingTime|errorCount):'
  name: tomcat_servlet_$3_total
  labels:
    module: "$1"
    servlet: "$2"
  help: Tomcat servlet $3 total
  type: COUNTER
- pattern: 'Catalina<type=ThreadPool, name="(\w+-\w+)-(\d+)"><>(currentThreadCount|currentThreadsBusy|keepAliveCount|pollerThreadCount|connectionCount):'
  name: tomcat_threadpool_$3
  labels:
    port: "$2"
    protocol: "$1"
  help: Tomcat threadpool $3
  type: GAUGE
- pattern: 'Catalina<type=Manager, host=([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), context=([-a-zA-Z0-9+/$%~_-|!.]*)><>(processingTime|sessionCounter|rejectedSessions|expiredSessions):'
  name: tomcat_session_$3_total
  labels:
    context: "$2"
    host: "$1"
  help: Tomcat session $3 total
  type: COUNTER
```

## Other example configs
Refer to [example configs](https://github.com/prometheus/jmx_exporter/tree/master/example_configs)

## Build exporter image with above configuration if needed

Refer to https://www.kubernetes.org.cn/8515.html

```Dockerfile
FROM ubuntu:16.04 as jar
WORKDIR /
RUN apt-get update -y
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y wget
RUN wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.13.0/jmx_prometheus_javaagent-0.13.0.jar

FROM tomcat:jdk8-openjdk-slim
ADD prometheus-jmx-config.yaml /prometheus-jmx-config.yaml
COPY --from=jar /jmx_prometheus_javaagent-0.13.0.jar /jmx_prometheus_javaagent-0.13.0.jar
```

Since the author pushed the image, we can use `Dockerfile.quick` as follows:
```
FROM ccr.ccs.tencentyun.com/imroc/tomcat:jdk8
ADD prometheus-jmx-config.yaml /prometheus-jmx-config.yaml
```

Build cmdline:
```
docker build . -t zhu733756/jmx-tomcat-exporter:latest -f Dockerfile.quick
```
If you don't want to build it, you can use `zhu733756/jmx-tomcat-exporter:latest` directly.

## Deploy tomcat exporter and servicemonitor

The follows based on you have installed prometheus-operator:

```
kubectl  apply -f tomcat.yaml
kubectl  apply -f servicemonitor.yaml
```

## Deploy jvm dashboard
```
kubectl apply -f dashboard.yaml
```
Will see as follows:

![img](https://github.com/zhu733756/jmx-exporter/blob/master/png/kubesphere-jvm-dashboard.jpg "kubesphere-jvm-dashboard.jpg")
