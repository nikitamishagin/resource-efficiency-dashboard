# Contributing

Thanks for your interest in contributing to **Resource Efficiency Dashboard**!

Contributions of all sizes are welcome — from fixing typos and improving wording, to refining PromQL, enhancing panels,
adjusting recommendation logic, or adding clearer examples for real-world workloads. If you’re not sure where to start,
open an issue with what you observed (a screenshot + short description is perfect), or propose a small improvement via
pull request.

To make contributing easy, start by setting up a reproducible local environment and validating that the dashboard works
end‑to‑end. Once you can run Prometheus + Grafana and see metrics flowing, you’ll be able to safely iterate on panels,
queries, and dashboard structure and verify your changes before submitting a PR.

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
  --set server.global.scrape_interval="30s" \
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

After that, you can import the dashboard from the `dashboard.json` file. To do this, go to the Grafana dashboard at
`http://<your-minikube-ip>:30300` and log in with the password `admin`. Then open the `Configuration` tab in the left
menu and click the `+` button in the upper right corner. Select `Import` and upload the `dashboard.json` file.

Now you can add a test workload to your cluster to get some metrics. For example, you can use the following command:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fake-load
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fake-load
  template:
    metadata:
      labels:
        app: fake-load
    spec:
      containers:
        - name: fake-load
          image: alpine:3.20
          imagePullPolicy: IfNotPresent
          command: ["/bin/sh", "-c"]
          args:
            - |
              set -eu
              apk add --no-cache stress-ng >/dev/null
              echo "Starting ~40m CPU and 40Mi memory load"
              while true; do
                stress-ng --cpu 1 --cpu-load 4 --vm 1 --vm-bytes 40M --vm-keep --timeout 60s
                echo "Pause 30s"
                sleep 30
              done
          resources:
            requests:
              cpu: "30m"
              memory: "30Mi"
            limits:
              cpu: "100m"
              memory: "100Mi"
EOF
```

Your development environment is ready! New metrics will be available in the dashboard after a few minutes.