# Prometheus Monitoring Setup on Kubernetes

The folders `web` and `api` describe how to install and run each app.

The folder Prometheus contains .yaml files for Prometheus deployment as well as PVC, PV, Storage Class. In addition to that it has .yaml files for NodePort Service, ConfigMap and ClusterRole.

The folder Grafana contains .yaml files for Grafana deployment as well as ConfigMap and Grafana NodePort Service for external access.

The folder Alertmanager contains .yaml files for Alertmanager deployment as well as ConfigMaps and NodePort Service.

First of all we will create a Kubernetes namespace for all our monitoring components. The name for our namespace will be "monitoring".

The next step will be to create an RBAC policy with read access to required API groups and bind the policy to the monitoring namespace.
