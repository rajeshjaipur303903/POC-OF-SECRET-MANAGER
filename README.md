
<img src="https://raw.githubusercontent.com/kubernetes/kubernetes/master/logo/logo.png" width="150" height="150" />

# AWS EKS & Secrets Manager (File & Env | Kubernetes | Secrets Store CSI Driver | K8s)

## 1. Create Secret in AWS Secrets Manager
- Select `Other type of secrets`
- Create key: `MY_API_TOKEN` and random value: `7623fd72g3d`
- Give it a name `prod/service/token`
- Open created secret to check ARN

## 2. Create EKS Cluster(I used eksctl)
- Create `eks.yaml` config file
- Create EKS cluster
```bash
eksctl create cluster -f eks.yaml
```
- Check connection to EKS cluster
```bash
kubectl get svc
```

## 3. Create IAM OIDC Provider for EKS
USE CLI
```bash
eksctl utils associate-iam-oidc-provider --region="$REGION" --cluster="$CLUSTERNAME" --approve # Only run this once
```
#(Optionally)
- Copy `OpenID Connect provider URL`
- Create Identety Provider - select `OpenID Connect`
- Enter `sts.amazonaws.com` for Audience

## 4. Create IAM Policy to Read Secrets
USE CLI
```bash
POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name <Name> --policy-document '{
    "Version": "2012-10-17",
    "Statement": [ {
        "Effect": "Allow",
        "Action": ["ssm:GetParameter", "ssm:GetParameters"], ##there is some changes
        "Resource": ["parameter-arn"]
    } ]
}')
```
#(optionally)
USE CONSOLE
- Create `NAME` IAM policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "<secret-arn>"
        }
    ]
}
```
## 5. Create Service Account for a Kubernetes 
IN service account we have directed attach policy ARN
policy_arn=arn:aws:iam::"${account_id}":policy/"${policy_name}"
```bash
eksctl create iamserviceaccount --name nginx-deployment-sa --region="$REGION" --cluster "$CLUSTERNAME" --attach-policy-arn "$POLICY_ARN" --approve --override-existing-serviceaccounts
```
# ADDITONAL 
## 1. Create IAM Role for a Kubernetes Service Account
- Click `Web identity` and select Identity provider that we created
- Select `APITokenReadAccess` IAM Policy
- Give it a name `api-token-access`
- Update trust relationships on the role 
- Update `aud` -> `sub`
- Update `sts.amazonaws.com` -> `system:serviceaccount:production:nginx`

## 2. Associate an IAM Role with Kubernetes Service Account
- Create `nginx/namespace.yaml`
- Create `nginx/service-account.yaml`
- Apply kubernetes objects
```bash
kubectl apply -f nginx
```
- Get Kubernetes namespaces
```bash
kubectl get ns
```
- Describe service account
```bash
kubectl get sa -n production
```

## 6. Install the Kubernetes Secrets Store CSI Driver
- Create `secrets-store-csi-driver/0-secretproviderclasses-crd.yaml`
- Create `secrets-store-csi-driver/1-secretproviderclasspodstatuses-crd.yaml`
- Apply CRDs
```bash
kubectl apply -f secrets-store-csi-driver
```
- Create `secrets-store-csi-driver/2-service-account.yaml`
- Create `secrets-store-csi-driver/3-cluster-role.yaml`
- Create `secrets-store-csi-driver/4-cluster-role-binding.yaml`
- Create `secrets-store-csi-driver/5-daemonset.yaml`
- Create `secrets-store-csi-driver/6-csi-driver.yaml`
- Apply Kubernetes objects
```bash
kubectl apply -f secrets-store-csi-driver
```
- Check the logs
```bash
kubectl logs -n kube-system -f -l app=secrets-store-csi-driver
```

- (Optionally) use helm chart
```bash
helm repo add secrets-store-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/master/charts
```
- (Optionally) install helm chart
```bash
helm -n kube-system install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver
```

## 7. Install AWS Secrets & Configuration Provider (ASCP)
- Create `aws-provider-installer/0-service-account.yaml`
- Create `aws-provider-installer/1-cluster-role.yaml`
- Create `aws-provider-installer/2-cluster-role-binding.yaml`
- Create `aws-provider-installer/3-daemonset.yaml`
- Apply aws-provider-installer
```bash
kubectl apply -f aws-provider-installer
```
- (Optionally) use helm
```bash
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```
- Check logs
```bash
kubectl logs -n kube-system -f -l app=csi-secrets-store-provider-aws
```

## 8. Create Secret Provider Class
- Create `nginx/2-secret-provider-class.yaml`
```yaml
---
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  
spec:
  provider: aws
  secretObjects:
  - secretName: api-token
    type: Opaque
    data: 
    - objectName: secret-token
      key: SECRET_TOKEN

  parameters:
    objects: |
      - objectName: prod/service/token5
        objectType: secretsmanager
        objectAlias: secret-token
```

```bash
kubectl apply -f <filename>
```

## 11. Create Deployment file
- Create nginx `3-deployment.yaml`
```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: nginx-deployment-sa
      containers:
      - name: nginx
        image: nginx:1.14.2

        ports:
        - containerPort: 80
        volumeMounts:
        - name: my-api-token
          mountPath: /mnt/api-token
          readOnly: true
        env:
        - name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: api-token
              key: SECRET_TOKEN
      
      volumes:
      - name: my-api-token
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: aws-secrets
 ---
 ```
- Open 2 tabs
```bash
kubectl logs -n kube-system -f -l app=secrets-store-csi-driver
```
```bash
kubectl apply -f <filename>
```
```bash
kubectl -n production exec -it nginx-<id> -- bash
```
- Print mounted file
```bash
cat /mnt/api-token/secret-token
```
- Print environment variables with a secret
```bash
echo $API_TOKEN
```


## Clean Up
- Delete EKS Cluster
```bash
eksctl delete cluster -f eks.yaml
```
- Delete IAM Policy `APITokenReadAccess`
- Delete IAM Role `api-token-access`

##Links
for key_value
https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html
https://www.youtube.com/watch?v=Rmgo6vCytsg&t=582s


