# Prometheus 
Prometheus is a free software application used for event monitoring and alerting. It records real-time metrics in a time series database built using a HTTP pull model, with flexible queries and real-time alerting.
Prometheus can collect metrics about your application and infrastructure.  Metrics are small concise descriptions of an event: date, time, and a descriptive value. While prometheus does store or ‘log’ metrics, metrics should not be confused with logs, which can include reams of data.  Rather than gathering a great deal of data about one thing, Prometheus uses the approach of gathering a little bit of data about many things to help you understand the state and trajectory of your system. It has become very popular in the industry because it has many powerful features for monitoring metrics and providing alerts that can, with orchestration systems (e.g. Kubernetes), automate responses to changing conditions.

# Prometheus Architecture

The Prometheus is a  multi-component system. While the following integrate into a Prometheus deployment, there is flexibility in which of these pieces are actually implemented. 
* Prometheus server (scrapes and stores metrics as time series data) 
* Client libraries for instrumenting application code 
* Push gateway (supports metrics collection from short-lived jobs) 
* Special-purpose exporters (Supports tools like HAProxy, StatsD, Graphite, etc.) 
* Alertmanager ( sends alerts based on triggers) 
* Additional support tools.

Prometheus can scrape metrics from jobs directly or, for short-lived jobs by using a push gateway when the job exits. The scraped samples are stored locally and rules are applied to the data to aggregate and generate new time series from existing data or generate alerts based on user-defined triggers. While Prometheus comes with a functional Web dashboard or other API consumers can be used to visualize the collected data, with Grafana being the de facto default.

# How Prometheus works 
Prometheus gets metric from an exposed HTTP endpoint.  A number of client libraries are available to provide this application integration when building software. With an available endpoint, Prometheus can scrape numerical data and store it as a time series in a local time series database. It can also integrate with remote storage options. 

In addition to the stored time series, impermanent times series from the source are produced by queries. These series are recognized by metric name and key value pairs known by labels. Queries are generated using PromQL (Prometheus Query Language) that enables users to choose and aggregate time series data in real time. PromQL is also used to establish alert conditions that can then transmit notifications outside sources such as PagerDuty, Slack or email.  These data can be displayed in graph or tabular form in Prometheus’s Web UI.  Alternatively, and commonly, API integrations with alternative display solutions such as Grafana may be used.

# Prerequsites
First we will have our cluster up and running next we will connect to our cluster and make sure we have an admin privileges to create cluster roles.

# Create a Namespace & ClusterRole

First of all we will create a Kubernetes namespace for all our monitoring components. The name for our namespace will be "monitoring".
If you don’t create a dedicated namespace, all the Prometheus kubernetes deployment objects get deployed on the default namespace.
We will use the following command to create a new namespace named monitoring:  
```
kubectl create namespace monitoring
```
Prometheus uses Kubernetes APIs to read all the available metrics from Nodes, Pods, Deployments, etc. For this reason, we need to create an RBAC policy with read access to required API groups and bind the policy to the monitoring namespace.

Create a file named cluster-role.yaml with following content:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```

# Create a Config Map To Externalize Prometheus Configurations

Create a file called config-map.yaml with following content:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: devopscube demo alert
      rules:
      - alert: High Pod Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_endpoints_name]
          regex: 'node-exporter'
          action: keep
      
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics     
      
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
      
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      
      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
```
Execute the following command to create the config map in Kubernetes:
```
kubectl apply -f config-map.yaml
```
# Create a Prometheus Deployment

Create a file named prometheus-deployment.yaml and copy the following contents onto the file. In this configuration, we are mounting the Prometheus config map as a file inside /etc/prometheus as explained in the previous section:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--storage.tsdb.retention.time=12h"
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          resources:
            requests:
              cpu: 500m
              memory: 500M
            limits:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
  
        - name: prometheus-storage-volume
          emptyDir: {}
```
Create a deployment on monitoring namespace using the above file:

```
kubectl apply -f prometheus-deployment.yaml 
```
You can check the created deployment using the following command:
```
kubectl get deployments --namespace=monitoring
```
# Connecting To Prometheus Dashboard:
First of all in order to connect to our Prometheus Dashboard we need to expose our Prometheus using Ingress.
If you have an existing ingress controller setup, you can create an ingress object to route the Prometheus DNS to the Prometheus backend service.

Also, you can add SSL for Prometheus in the ingress layer.

Here is a sample ingress object. 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ui
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  # Use the host you used in your kubernetes Ingress Configurations
  - host: prometheus.example.com
    http:
      paths:
      - backend:
          serviceName: prometheus-service
          servicePort: 8080
```
# Setting Up Alertmanager

AlertManager is an open-source alerting system that works with the Prometheus Monitoring system. This blog is part of the Prometheus Kubernetes tutorial series.

# Prerequisites 

You should have a working Prometheus setup up and running.
Prometheus should have the correct alert manager service endpoint as shown below to send the alert to Alert Manager:
```
alerting:
   alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"
```
# Config Map for Alert Manager Configuration
Alert Manager reads its configuration from a config.yaml file. It contains the configuration of alert template path, email, and other alert receiving configurations.
Create a file named alertmanager-conf.yaml and copy the following contents:
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  config.yml: |-
    global:
    templates:
    - '/etc/alertmanager/*.tmpl'
    route:
      receiver: alert-emailer
      group_by: ['alertname', 'priority']
      group_wait: 10s
      repeat_interval: 30m
      routes:
        - receiver: slack_demo
        # Send severity=slack alerts to slack.
          match:
            severity: slack
          group_wait: 10s
          repeat_interval: 1m
 
    receivers:
    - name: alert-emailer
      email_configs:
      - to: demo@devopscube.com
        send_resolved: false
        from: from-email@email.com
        smarthost: smtp.eample.com:25
        require_tls: false
    - name: slack_demo
      slack_configs:
      - api_url: https://hooks.slack.com/services/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sz4Qk
        channel: '#devopscube-demo'
```

Let’s create the config map using kubectl:
```
kubectl apply -f alertmanager-conf.yaml
```
# Config Map for Alert Template
We need alert templates for all the receivers we use (email, Slack, etc). Alert manager will dynamically substitute the values and deliver alerts to the receivers based on the template. You can customize these templates based on your needs.

Create the configmap using kubectl:
```
kubectl create -f AlertTemplateConfigMap.yaml
```

# Create a Deployment
In this deployment, we will mount the two config maps we created.

Create a file called alertmanager-deployment.yaml with the following content:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      name: alertmanager
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:latest
        args:
          - "--config.file=/etc/alertmanager/config.yml"
          - "--storage.path=/alertmanager"
        ports:
        - name: alertmanager
          containerPort: 9093
        resources:
            requests:
              cpu: 500m
              memory: 500M
            limits:
              cpu: 1
              memory: 1Gi
        volumeMounts:
        - name: config-volume
          mountPath: /etc/alertmanager
        - name: templates-volume
          mountPath: /etc/alertmanager-templates
        - name: alertmanager
          mountPath: /alertmanager
      volumes:
      - name: config-volume
        configMap:
          name: alertmanager-config
      - name: templates-volume
        configMap:
          name: alertmanager-templates
      - name: alertmanager
        emptyDir: {}
```

Create the alert manager deployment using kubectl:
```
kubectl apply -f Deployment.yaml
```
# Create the Alert Manager Service Endpoint
We need to expose the alert manager using NodePort or Load Balancer just to access the Web UI. Prometheus will talk to the alert manager using the internal service endpoint.
Create a alertmanager-svc.yaml file with the following contents:
```
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9093'
spec:
  selector: 
    app: alertmanager
  type: NodePort  
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 31000
```
Create the service using kubectl:
```
kubectl apply -f Service.yaml
```
Now, you will be able to access Alert Manager on Node Port 31000. Note that in our case we didn't expose our service through Node Port but rather used Cluster IP.
Let's move forward to setting up our Grafana dashboard to visualize our data from Prometheus.

# What is Grafana
Grafana is an open-source observability platform for visualizing metrics, logs, and traces collected from your applications. It’s a cloud-native solution for quickly assembling data dashboards that let you inspect and analyze your stack.

# Deploy Grafana On Kubernetes
Create a file named grafana-datasource-config.yaml:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
               "access":"proxy",
                "editable": true,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus-service.monitoring.svc:8080",
                "version": 1
            }
        ]
    }
```
Create the configmap using the following command:
```
kubectl apply -f grafana-datasource-config.yaml
```
Create a file named grafana-deployment.yaml with following content:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - name: grafana
          containerPort: 3000
        resources:
          limits:
            memory: "1Gi"
            cpu: "1000m"
          requests: 
            memory: 500M
            cpu: "500m"
        volumeMounts:
          - mountPath: /var/lib/grafana
            name: grafana-storage
          - mountPath: /etc/grafana/provisioning/datasources
            name: grafana-datasources
            readOnly: false
      volumes:
        - name: grafana-storage
          emptyDir: {}
        - name: grafana-datasources
          configMap:
              defaultMode: 420
              name: grafana-datasources
```
Create the deployment:

```
kubectl apply -f deployment.yaml
```

Create a service file named service.yaml with following content: 
```
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '3000'
spec:
  selector: 
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
```
In the same file we will add an ingress resource for our grafana so it will be exposed to internet: 
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ui
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  # Use the host you used in your kubernetes Ingress Configurations
  - host: grafana.example.com
    http:
      paths:
      - backend:
          serviceName: prometheus-service
          servicePort: 3000
```
Create the service:
```
kubectl apply -f service.yaml
```
First of all in order to connect to our GRafana Dashboard we need to expose our Grafana using Ingress.
If you have an existing ingress controller setup, you can create an ingress object to route the Grafana DNS to the Prometheus backend service.

Also, you can add SSL for Grafana in the ingress layer.

# Conclusion

Grafana is a very powerful tool when it comes to Kubernetes monitoring dashboards.

It is used by many organizations to monitor their Kubernetes workloads. With the wide range of pre-built templates, you can get started with the templates pretty quickly.
