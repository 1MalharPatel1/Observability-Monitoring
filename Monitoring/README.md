# Monitoring

### Steps

1. minikube start
2. helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
3. kubectl create ns monitoring
4. helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
-f ./custom_kube_prometheus_stack.yml
5. kubectl --namespace monitoring get pods -l "release=monitoring"
6.  kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
    kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
    kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093