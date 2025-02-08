# Tracing Jaegar

### Steps


1. Instrumentation

- already done inside demo folder's application services a and b

2. Create IAM Role for Service Account

- Reason for creating IAM Role
    - Elasticsearch is with the EKS cluster
    - EBS volume is outside the cluster
    - So, to communicate between two different serivces of AWS, we need IAM role
    - AWS services can't use IAM role
    - So, we integrate service account with IAM role
    - To connect service account with IAM role we use OIDC connector

eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster observability \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve

3. Retrieve IAM role ARN

ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)

4. Deploy EBS CSI Driver

eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force

5. Create Namespace for Logging

kubectl create namespace logging

6. Install Elasticsearch on K8s

helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging

7. Retrieve Elasticsearch Username & Password

kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d

kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d

8. Export Elasticsearch CA Certificate

kubectl get secret elasticsearch-master-certs -n logging -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem

- This command retrieves the CA certificate from the Elasticsearch master certificate secret and decodes it, saving it to a ca-cert.pem file.

9. Create Tracing Namespace

kubectl create ns tracing

- Creates a new Kubernetes namespace called tracing if it doesn't already exist, where Jaeger components will be installed.

10. Create ConfigMap for Jaeger's TLS Certificate

kubectl create configmap jaeger-tls --from-file=ca-cert.pem -n tracing

11. Create Secret for Elasticsearch TLS

kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n tracing

12. Add Jaeger Helm Repository

helm repo add jaegertracing https://jaegertracing.github.io/helm-charts

helm repo update

13. Install Jaeger with Custom Values

- Note: Please update the password field and other related field in the jaeger-values.yaml file with the password retrieved earlier in the logging folder at step 6: (i.e NJyO47UqeYBsoaEU)"

helm install jaeger jaegertracing/jaeger -n tracing --values jaeger-values.yaml

14. Port Forward Jaeger Query Service

kubectl port-forward svc/jaeger-query 8080:80 -n tracing

- Command forwards port 8080 on your local machine to the Jaeger Query service, allowing you to access the Jaeger UI locally.

### Clean Up

helm uninstall jaeger -n tracing

helm uninstall elasticsearch -n logging

- Also delete PVC created for elasticsearch

helm uninstall monitoring -n monitoring

kubectl delete -k kubernetes-manifest/

kubectl delete -k alerts-alertmanager-servicemonitor-manifest/

eksctl delete cluster --name observability