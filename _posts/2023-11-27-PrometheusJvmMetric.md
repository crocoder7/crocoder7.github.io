---
title: "Prometheus JVM Spring metric"

categories: 
  - Prometheus
  - JVM
  - Spring
  - DevOps
  - Helm
  - k8s
  - metric
last_modified_at: now
---
# How to get centralize JVM & Spring metric using Prometheus
Amazon managed prometheus and grafana might be a good option for monitoring when you are AWS EKS. <br>
Pros when using Amazon managed prometheus : Since it is not in out cluster, we don't have to worry about it affecting nodes of the cluster. <br>
Prometheus operator is needed to be install in k8s cluster in order to send metric to our central Prometheus. <br>

- First, create name space for prometheus. <br>

```
kubectl create namespace prometheus
```

- Second, create OIDC provider and service account (from the code below : iamserviceaccount). Attach prometheus write access policy to service account. <br>

```
eksctl utils associate-iam-oidc-provider --cluster ${EKS_CLUSTER_NAME} --approve

eksctl create iamserviceaccount --name my-service-account --namespace prometheus --cluster my-cluster \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess --approve
```

- Third, install prometheus operator using helm with values.yaml below. <br>
This is values.yaml <br>

```
alertmanager:
  enabled: false

grafana:
  enabled: false

prometheus:
  serviceAccount:
    create: false
    name: iamserviceaccount

  ingress:
    enabled: false

  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 1d
    retentionSize: "5GiB"
    scrapeInterval: 15s
    evaluationInterval: 15s
    remoteWrite:
    - url: Your Amazon Managed Grafana URL
      sigv4:
        region: ap-northeast-2
      queueConfig:
        maxSamplesPerSend: 1000
        maxShards: 200
        capacity: 2500
```

Please make sure that the value 'serviceMonitorSelectorNilUsesHelmValues' should set to false since we are using additional service monitor to collect JVM, Spring metric below. <br>
Commands for installing prometheus with helm below. <br>

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
--namespace prometheus \
-f values.yaml
```

- Now, metrics of your k8s node is collected in your managed prometheus.
From now on, it's time to collect JVM Spring metric! <br>
Use the following configuration for service monitor yaml file. (Let's say the name of this yaml file service-monitor.yaml) <br>

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    release: kube-prometheus-stack
  name: your-choice-for-name
  namespace: prometheus
spec:
  endpoints:
  - path: /actuator/prometheus
    port: your-service-port
  namespaceSelector:
    matchNames:
    - your-app-namespace
  selector:
    matchLabels:
      release: service-label
```

```
kubectl apply -f service-monitor.yaml
```

Please make sure your service monitor is registered properly. <br>

```
kubectl get servicemonitors.monitoring.coreos.com -n prometheus
```

The labels of the metadata might be different for the future so that please confirm. (It is said that service monitor is selected based on the label in some other articles.) <br>
Please use the command below to find out the proper label. (I used 'kube-prometheus-stack-operator' to find out the label. You can use other objects on the list from the result of the command above.) <br>

```
kubectl get servicemonitors.monitoring.coreos.com -n prometheus kube-prometheus-stack-operator -o yaml
```

Also, you can find out on the web UI of your prometheus operator. <br>
You have change type of the prometheus since it is set to cluster which cannot be accessed from outside. <br>
In my case, changed it to LoadBalancer type to access its web UI. <br>

```
kubectl patch svc kube-prometheus-stack-prometheus -n prometheus --type='json' -p '[{"op": "replace", "path": "/spec/type", "value": "LoadBalancer"}]'
```

Please make sure that the name of your service monitor is on the Status -> Targets. <br>
You can change its type back to cluster after you confirmed. <br>
<br>
- This diagram shows well how the service monitor process works.
![flow diagram](/assets/images/prometheus-custom-metrics-elements.png)

**This is important**
- You CANNOT use your service port NUMBER for the service monitor configuration of your yaml file.
  - If you haven't defined the name of your port of your service object, you should put name on it and use it in servcie monitor configuration!
- Additional annotations for your service
- Prometheus and actutator dependency should be properly configured and opend in your application.
<br>

```
apiVersion: v1
kind: Service
metadata:
  name: your-service-name
  namespace: your-app-namespace
  labels:
    release: service-label
  annotations:
    prometheus.io/path: /actuator/prometheus
    prometheus.io/port: "80"
    prometheus.io/scrape: "true"
spec:
  selector:
    app: your-deployment-label
  type: NodePort
  ports:
    -name: your-service-port
     port: 80
     protocol: TCP
     targetPort: 8080
```
This is what the service object configuration should be like. <br>

* References
  * [Medium](https://medium.com/@damindubandara/configuring-jvm-monitoring-dashboard-in-grafana-using-spring-boot-actuator-with-prometheus-and-e7142b5e4c81)
  * [Blog](https://fabianlee.org/2022/07/07/prometheus-monitoring-a-custom-service-using-servicemonitor-and-prometheusrule/)
  * [Blog](https://malwareanalysis.tistory.com/602)
