# Logging ELK Stack

### Steps

- Use the k8s cluster created inside demo folder

1. Create IAM Role for Service Account

eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster observability \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve

- This command creates an IAM role for the EBS CSI controller.
- IAM role allows EBS CSI controller to interact with AWS resources, specifically for managing EBS volumes in the Kubernetes cluster.
- We will attach the Role with service account

2. Retrieve IAM role ARN

ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)

- Command retrieves the ARN of the IAM role created for the EBS CSI controller service account.

3. Deploy ESB CSI Driver

eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force

- Above command deploys the AWS EBS CSI driver as an addon to your Kubernetes cluster.
- It uses the previously created IAM service account role to allow the driver to manage EBS volumes securely.

4. Create Namespace for Logging

kubectl create namespace logging

5. Install Elasticsearch on K8s

helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging

- Installs Elasticsearch in the logging namespace.
- It sets the number of replicas, specifies the storage class, and enables persistence labels to ensure data is stored on persistent volumes.

6. Retrieve Elasticsearch Username & Password

- for username
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
- for password
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d

- Retrieves the password for the Elasticsearch cluster's master credentials from the Kubernetes secret.
- The password is base64 encoded, so it needs to be decoded before use.
- Note: Please write down the password for future reference

7. Install Kibana

helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging

- Kibana provides a user-friendly interface for exploring and visualizing data stored in Elasticsearch.
- It is exposed as a LoadBalancer service, making it accessible from outside the cluster.

8. Install Fluentbit with Custom Values/Configurations

- Note: Please update the HTTP_Passwd field in the fluentbit-values.yml file with the password retrieved earlier in step 6: (i.e NJyO47UqeYBsoaEU)"

helm repo add fluent https://fluent.github.io/helm-charts

helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging

### Clean Up

helm uninstall monitoring -n monitoring

helm uninstall fluent-bit -n logging

helm uninstall elasticsearch -n logging

helm uninstall kibana -n logging

kubectl delete -k kubernetes-manifest/

kubectl delete -k alerts-alertmanager-servicemonitor-manifest/

eksctl delete cluster --name observability