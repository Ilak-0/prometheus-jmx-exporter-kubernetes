# Monitoring JAVA app with jmx_exporter in k8s(using initContainer) 

## Requirement
* `prometheus` server to collect metrics
* `jmx_exporter` run as a Java Agent exposing a HTTP server and serving metrics of the local JVM
* `grafana` dashboard for visualization runnung on port `3000`
* `Java` application example eg tomcat
* `k8s,docker` run container 

## Dockerfile to create initContainer
to create a initContainer image
```hcl
FROM alpine

#COPY config.yaml jmx_prometheus_javaagent-0.13.0.jar /copyfile/agent/
COPY jmx_prometheus_javaagent-0.13.0.jar /copyfile/agent/
```
docker build -f Dockerfile -t tagNameXXX  .

## Get images
get tomcat, prometheus,grafana images from public docker hug

## Create k8s config map
jmx-config
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jmx-config
  namespace: default
data:
  jmx-exporter.yml: |
    lowercaseOutputName: true
    lowercaseOutputLabelNames: true
    
    rules:
      - pattern: 'java.lang<type=OperatingSystem><>(FreeSwapSpaceSize|SystemCpuLoad|ProcessCpuLoad|OpenFileDescriptorCount)'
        name: java_lang_OperatingSystem_$1
        type: GAUGE
    
      - pattern: 'java.lang<type=Threading><>(TotalStartedThreadCount|ThreadCount)'
        name: java_lang_threading_$1
        type: GAUGE
    
      - pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+-\w+)-(\d+)\"><>(\w+)'
        name: catalina_globalrequestprocessor_$3_total
        labels:
          port: "$2"
          protocol: "$1"
        help: Catalina global $3
        type: COUNTER
    
      - pattern: 'Catalina<j2eeType=Servlet, WebModule=//([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), name=([-a-zA-Z0-9+/$%~_-|!.]*), J2EEApplication=none, J2EEServer=none><>(requestCount|processingTime)'
        name: catalina_servlet_$3_total
        labels:
          module: "$1"
          servlet: "$2"
        help: Catalina servlet $3 total
        type: COUNTER
    
      - pattern: 'Catalina<type=ThreadPool, name="(\w+-\w+)-(\d+)"><>(currentThreadsBusy|keepAliveCount|pollerThreadCount|connectionCount)'
        name: catalina_threadpool_$3
        labels:
          port: "$2"
          protocol: "$1"
        help: Catalina threadpool $3
        type: GAUGE
    
      - pattern: 'Catalina<type=Manager, host=([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), context=([-a-zA-Z0-9+/$%~_-|!.]*)><>(processingTime|sessionCounter|rejectedSessions|expiredSessions)'
        name: catalina_session_$3_total
        labels:
          context: "$2"
          host: "$1"
        help: Catalina session $3 total
        type: COUNTER

```
prometheus-config
```yaml
# cat pro_cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: default
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'jmx-exporter'
      static_configs:
      ## your java app/ tomcat endpoints address
      - targets: ['xx.x.x.xxxx:1234']

```
## Create k8s yaml
grafana
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      bb: web
  template:
    metadata:
      labels:
        bb: web
    spec:
      containers:
        - name: grafana-site
          image: grafana/grafana
          imagePullPolicy: Never

---
apiVersion: v1
kind: Service
metadata:
  name: grafana-entrypoint
  namespace: default
spec:
  type: NodePort
  selector:
    bb: web
  ports:
    - name: grafana
      port: 3000
      targetPort: 3000
      nodePort: 30300

```
prometheus
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      bb: web
  template:
    metadata:
      labels:
        bb: web
    spec:
      containers:
        - name: prometheus-site
          image: ubuntu/prometheus
          imagePullPolicy: Never
          args:
            - "--web.enable-admin-api"                       #控制对admin HTTP API的访问，其中包括删除时间序列等功能
            - "--web.enable-lifecycle"                       #支持热更新，直接执行localhost:9090/-/reload立即生效
            - "--config.file=/etc/prometheus/prometheus.yml" #通过volume挂载prometheus.yml
          volumeMounts:
            - mountPath: "/etc/prometheus"
              name: config-volume
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config     #定义的prometeus.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-entrypoint
  namespace: default
spec:
  type: NodePort
  selector:
    bb: web
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 30090
```
tomcat
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      bb: web
  template:
    metadata:
      labels:
        bb: web
    spec:
      containers:
        - name: tomcat-site
          image: tomcat
          imagePullPolicy: Never
          env:
            - name: JAVA_TOOL_OPTIONS
              value: -javaagent:/jmx/agent/jmx_prometheus_javaagent-0.13.0.jar=1234:/jmx/agent/config/prometheus-jmx-config.yaml
          volumeMounts:
            - mountPath: /jmx/agent
              name: shared-volume
            - mountPath: /jmx/agent/config
              name: prometheus-jmx-config
      volumes:
        - name: shared-volume
          emptyDir: {}
        - configMap:
            name: prometheus-jmx-config
          name: prometheus-jmx-config
      initContainers:
        - args:
            - -c
            - mkdir -p /jmx/agent && cp -r /copyfile/agent/* /jmx/agent
          command:
            - sh
          image: prometheus-jmx-exporter-kubernetes
          name: prometheus-jmx-exporter
          imagePullPolicy: Never
          volumeMounts:
            - mountPath: /jmx/agent
              name: shared-volume

---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-entrypoint
  namespace: default
  annotations:
    "prometheus.io/scrape": "true"
    "prometheus.io/port": "1234"
spec:
  type: NodePort
  selector:
    bb: web
  ports:
    - name: tomcat
      port: 8080
      targetPort: 8080
      nodePort: 30080
    - name: jmx-exporter
      port: 1234
      targetPort: 1234
      nodePort: 31234

```
## Running

kubectl apply -f xxx.yaml


### JMX Grafana dashboard
Using `jmx_exporter.json` for this dashboard

![](./pic/grafana.png)
