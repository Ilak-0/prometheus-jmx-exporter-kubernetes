# Monitoring JAVA app with jmx_exporter 

## Requirement
* `prometheus` server to collect metrics
* `jmx_exporter` run as a Java Agent exposing a HTTP server and serving metrics of the local JVM
* `grafana` dashboard for visualization runnung on port `3000`
* `Java` application example

## Dockerfile
```hcl
FROM java:8-jdk-alpine
COPY config.yaml nasapicture-0.0.1-SNAPSHOT.war jmx_prometheus_javaagent-0.13.0.jar /usr/app/
EXPOSE 8080 8082
WORKDIR /usr/app
ENTRYPOINT ["java", "-javaagent:./jmx_prometheus_javaagent-0.13.0.jar=8082:config.yaml", "-jar", "nasapicture-0.0.1-SNAPSHOT.war"]

```
`Metrics will now be accessible at http://localhost:8082/metrics`


## Running

`docker build -t jmx-java:v1 .`

`docker run -dit -p 8082:8082 jmx-java:v1`

## Prometheus configuration

```hcl
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus_jmx'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'jmx'
    static_configs:
      - targets:
        - 127.0.0.1:8082
```



### JMX Grafana dashboard
Using `jmx_exporter.json` for this dashboard

![](./pic/grafana.PNG)
