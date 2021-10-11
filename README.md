# jmx-exporter
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

## Build exporter images if needed

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

Build steps:
```
docker build . -t zhu733756/jmx-tomcat-exporter:latest -f Dockerfile.quick
```
If you don't want to build it, you can use `zhu733756/jmx-tomcat-exporter:latest` directly.

## Deploy exporter and servicemonitor

The follows based on you have installed prometheus-operator:

```
kubectl  apply -f tomcat.yaml
kubectl  apply -f servicemonitor.yaml
```

## Deploy jvm dashboard
```
kubectl apply -f dashboard.yaml
```
