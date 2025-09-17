# Resource Efficiency Dashboard

## Project Description

Resource Efficiency Dashboard is a Grafana dashboard designed to monitor resource utilization efficiency in Kubernetes
clusters. The dashboard visualizes the relationship between requested and actually used resources (CPU and RAM) for
containers, which allows optimizing resource allocation and reducing infrastructure costs.

## What the Dashboard Shows

The dashboard provides the following information:

- **Resource Requests Usage**: Shows how efficiently the requested CPU and RAM resources are being used (in percentages).
- **Resource Limits Usage**: Displays how close containers are to their established limits.
- **Average Resource Idle Percentage**: Visualizes unused resources in the selected namespace.
- **Optimal Resource Ratio**: Calculates the ideal RAM to CPU ratio based on data from the last 7 days.
- **Resource Usage Graphs**: Show resource usage dynamics over time.
- **Recommendations Table**: Compares current requests and limits with recommended values based on historical data.

## Why This Information is Important for Kubernetes Users

This dashboard helps DevOps engineers and Kubernetes cluster administrators to:

1. **Optimize Costs**: Identify containers with excessive resource requests, which helps reduce infrastructure expenses.
2. **Improve Stability**: Detect containers with insufficient resources that may cause performance issues.
3. **Enhance Placement Density**: Optimize pod placement on cluster nodes.
4. **Plan Scaling**: Obtain data for making decisions about cluster scaling needs.
5. **Identify Anomalies**: Detect unusual resource usage patterns that may indicate problems in applications.

## Importance of Setting Correct Resources for Containers

Setting correct resources (requests and limits) for containers in Kubernetes is critically important for several reasons:

- **Effective Scheduling**: The Kubernetes Scheduler uses request values to make decisions about pod placement on nodes.
  Incorrect values can lead to suboptimal load distribution.
- **Preventing Resource Contention**: Proper limits prevent situations where one container consumes all available node
  resources, affecting the operation of other containers.
- **Resource Conservation**: Excessive requests lead to inefficient cluster utilization and increased costs.
- **Operational Stability**: Insufficient requests and limits can lead to unstable application performance, OOMKills,
  and performance degradation.
- **Predictability**: Correct resource values make system behavior more predictable and simplify autoscaling.

This dashboard provides the necessary metrics for making informed decisions when configuring container resources,
contributing to more efficient use of the Kubernetes cluster and reducing operational costs.

## How to Use the Dashboard

1. Select the desired namespace from the dropdown list.
2. If necessary, refine the selection by specifying specific pods and/or containers.
3. Analyze resource usage metrics for the selected period.
4. Pay attention to the recommended resource values in the table at the bottom of the dashboard.
5. Use the obtained information to adjust requests and limits in Kubernetes manifests.

## How to prepare the local development environment

To prepare the development environment, you can use the following commands on your kubernetes cluster. I recommend using
minikube.

Install Prometheus:

```bash
helm upgrade --install --create-namespace prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set alertmanager.enabled=false \
  --set pushgateway.enabled=false \
  --set nodeExporter.enabled=true \
  --set kubeStateMetrics.enabled=true \               
  --set server.service.type=NodePort \                                  
  --set server.service.nodePort=30090 \                                 
  --set server.global.scrape_interval="60s" \                                                                     
  --set 'server.metricRelabelings[0].action=replace' \               
  --set 'server.metricRelabelings[0].target_label=cluster' \         
  --set 'server.metricRelabelings[0].replacement=minikube'
```

Install Grafana:

```bash
helm upgrade --install grafana grafana/grafana --create-namespace \                   
  --namespace monitoring \
  --set service.type=NodePort \     
  --set service.nodePort=30300 \   
  --set adminPassword='admin' \    
  --set="datasources.datasources\.yaml.apiVersion=1" \
  --set="datasources.datasources\.yaml.datasources[0].name=Prometheus" \
  --set="datasources.datasources\.yaml.datasources[0].type=prometheus" \
  --set="datasources.datasources\.yaml.datasources[0].url=http://prometheus-server.monitoring.svc.cluster.local" \
  --set="datasources.datasources\.yaml.datasources[0].access=proxy" \
  --set="datasources.datasources\.yaml.datasources[0].isDefault=true"
```

After that, you can import the dashboard from the `dashboard.json` file. To do this, go to the Grafana dashboard on
http://<you-minikube-ip>:30300 and log in with the password `admin`. Then go to the `Configuration` tab in the left menu and
click the `+` button in the upper right corner. Select `Import` and upload the `dashboard.json` file.`

Now you can add a test workload to your cluster to get some metrics. For example, you can use the following command:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alpine
spec:
  replicas: 3
  selector:
    matchLabels:
      app: alpine
  template:
    metadata:
      labels:
        app: alpine
    spec:
      containers:
        - name: alpine
          image: alpine
          command: ["/bin/sh", "-c", "sleep infinity"]
          resources:
            requests:
              cpu: "100m"
              memory: "100Mi"
            limits:
              cpu: "400m"
              memory: "200Mi"
EOF
```

Your development environment is ready! New metrics will be available in the dashboard after a few minutes.