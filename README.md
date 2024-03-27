# Bitnami package for JMX Exporter

## What is JMX Exporter?

> A process for exposing JMX Beans via HTTP for Prometheus consumption.

[Overview of JMX Exporter](https://github.com/prometheus/jmx_exporter)
Trademarks: This software listing is packaged by Bitnami. The respective trademarks mentioned in the offering are owned by the respective companies, and use of them does not imply any affiliation or endorsement.

## Purpose
Run JMX Exporter in standalone mode instead of running it inside the application. 

## Implementation
To enable JMX beans inside your applicatoin, put the following command in the entrypoint of Dockerfile or deployment manifest command:

```
java -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.rmi.port=1099 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=localhost -Xms512M -Xmx4G -jar /opt/app/app.jar
```
This will enable the JMX beans to be exposed at port 1099. Now we can use this port inside the config.yaml of jmx-exporter to run as side car container. 

We use the kubernetes manifest to deploy jmx-exporter as side car standanlone agent. The manifest consists of 
- deployment of application with jmx-exporter as side-car container and ports required which are 1099 for JMX beans and 5556 for jmx-exporter
- Service manifest to expose these ports as ClusterIP
-  Configmaps to configure jmx-exporter


```console
---
apiVersion: v1
kind: Service
metadata:
  name: app-svc
  namespace: ns-app
spec:
  ports:
    - name: app-svc-1099
      port: 1099
      targetPort: 1099
    - name: app-svc-5556
      port: 5556
      targetPort: 5556
  selector:
    name: app

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    "prometheus.io/scrape": "true"
    "prometheus.io/port": "5556"
  name: app
  namespace: ns-app
  labels:
    name: app
spec:
  replicas: 1
  serviceName: app-svc
  selector:
    matchLabels:
      name: app
  template:
    metadata:
      labels:
        name: app
    spec:
      imagePullSecrets:
        - name: image-secret
      securityContext:
        runAsUser: 100
        fsGroup: 101
        runAsNonRoot: true
      containers:
        - name: jmx-exporter
          image: ahsanraza/jmx-exporter:develop
          command:
            - /bin/sh
            - -c
            - java -jar jmx_prometheus_httpserver.jar 5556 /config/config.yaml
          volumeMounts:
            - name: config-volume
              mountPath: /config
          ports:
            - containerPort: 5556
        - name: app
          image: app:v1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 1099
          volumeMounts:
            - name: app-data
              mountPath: /opt/app/data
      restartPolicy: Always
      volumes:
        - name: config-volume
          configMap:
            name: jmx-exporter-config
            defaultMode: 420
  volumeClaimTemplates:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: app-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 8Gi
        volumeMode: Filesystem
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: jmx-exporter-config
  namespace: ns-app
data:
  config.yaml: |
    hostPort: app-svc.ns-app.svc.cluster.local:1099
    username:
    password:
    rules:
      - pattern: ".*"
    startDelaySeconds: 0
    ssl: false
    lowercaseOutputName: true
    lowercaseOutputLabelNames: true
    whitelistObjectNames:
      - "org.apache.activemq:destinationType=Queue,*"
      - "org.apache.activemq:destinationType=Topic,*"
      - "org.apache.activemq:type=Broker,brokerName=*"
      - "org.apache.activemq:type=Topic,brokerName=*"
```
save this in a file and run the following command:



```console
kubectl apply -f deployment.yaml
```
