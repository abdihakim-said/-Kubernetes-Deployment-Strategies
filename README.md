
# **Enterprise Deployment Strategies with Argo Rollouts, Vault, and Kubernetes on EKS**

---
![argo-rollout](https://github.com/user-attachments/assets/ce623999-118d-4230-8944-9e5ddd3be07d)




This project demonstrates how to implement **secure and automated deployment strategies** in an **EKS (Amazon Elastic Kubernetes Service)** cluster using **Argo Rollouts**, **HashiCorp Vault**, and **Kubernetes Secrets**. The project includes:

- **Frontend Application**: Deployed using a **Canary Deployment** strategy.
- **MongoDB Database**: Deployed using a **Blue-Green Deployment** strategy.
- **Secure Secrets Management**: Secrets are securely managed using **HashiCorp Vault** and Kubernetes Secrets.

---

## Challenges Faced by the Enterprise

### 1. Risk of Downtime
- Deploying new versions of the frontend or backend often caused service interruptions, leading to poor user experience.
- MongoDB updates required downtime, which impacted critical services.

### 2. Rollback Complexity
- Rolling back to a previous version after a failed deployment was manual and error-prone.
- Lack of automated rollback mechanisms increased recovery time during failures.

### 3. Security Concerns
- Database credentials were managed manually, increasing the risk of leaks or mismanagement.
- Static secrets were hardcoded in configuration files, violating security best practices.

### 4. Lack of Deployment Automation
- The team relied on basic Kubernetes rolling updates, which lacked advanced deployment controls like traffic splitting, monitoring, and rollback capabilities.
- Deployment progress and health were difficult to observe.

---

## Proposed Solution

To address these challenges, the following strategies were implemented:

### 1. Frontend Deployment: Canary Strategy
- Gradually roll out new versions of the frontend service to a subset of users.
- Start by deploying 20% of new pods running the updated version.
- Incrementally increase the percentage to 50%, then 100% after monitoring for errors or performance issues.
- Use **Argo Rollouts** to manage the Canary deployment and enable quick rollbacks if issues are detected.

### 2. Backend Deployment: Blue-Green Strategy
- Create a parallel environment with the new version of the MongoDB backend.
- Test the new backend infrastructure in isolation to ensure it works as expected.
- Switch traffic to the new environment only after full verification.
- Use Kubernetes services to route traffic between the old (blue) and new (green) environments, ensuring zero-downtime deployments.

### 3. Secret Management with HashiCorp Vault
- Use **HashiCorp Vault** to dynamically manage and inject database credentials into Kubernetes pods.
- Automatically rotate secrets to enhance security.
- Remove manual secret management, reducing the risk of human error and credential leaks.

---

## Features

- **Argo Rollouts**:
  - Automates Canary and Blue-Green deployment strategies.
  - Provides observability and rollback capabilities.
- **Vault Integration**:
  - Securely injects secrets into pods using Vault Agent.
- **Kubernetes Secrets**:
  - Stores MongoDB credentials securely as a fallback.
- **Node Affinity**:
  - Ensures MongoDB pods are scheduled on nodes labeled with `servertype=large`.

---

## Setting Up HashiCorp Vault on EKS

### 1. Deploy Vault on EKS

If Vault is not already installed, you can deploy it using Helm:
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

Verify that Vault is running:
```bash
kubectl get pods
```

### 2. Enable Kubernetes Authentication

Enable the Kubernetes authentication method in Vault:
```bash
vault auth enable kubernetes
```

Configure the Kubernetes auth method:
```bash
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[0].cluster.server}')" \
    kubernetes_ca_cert=@<(kubectl get configmap -n kube-system extension-apiserver-authentication -o=jsonpath='{.data.client-ca-file}')
```

### 3. Create a Vault Policy

Create a policy to allow access to the secret path:
```bash
vault policy write wezvatech-policy - <<EOF
path "internal/data/database/config" {
  capabilities = ["read"]
}
EOF
```

### 4. Create a Vault Role

Bind the service account to the policy:
```bash
vault write auth/kubernetes/role/wezvatech-admin \
    bound_service_account_names="wezvatech-admin" \
    bound_service_account_namespaces="default" \
    policies="wezvatech-policy" \
    ttl="24h"
```

### 5. Store Secrets in Vault

Store MongoDB credentials in Vault:
```bash
vault kv put internal/data/database/config username="adminuser" password="password123"
```

---

## Configuring IAM Roles for Service Accounts (IRSA) on EKS

To securely allow Vault to access AWS resources (e.g., S3 for storage), configure **IAM Roles for Service Accounts (IRSA)**:

1. **Create an IAM Role**:
   ```bash
   aws iam create-role --role-name eks-vault-role --assume-role-policy-document file://trust-policy.json
   ```

   Example `trust-policy.json`:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "<OIDC_PROVIDER>:sub": "system:serviceaccount:default:wezvatech-admin"


           }


         }
       }
     ]
   }
   ```

2. **Attach Policies to the Role**:
   Attach the necessary policies to the role (e.g., `AmazonS3FullAccess` for Vault storage).

3. **Associate the Role with the Service Account**:
   Annotate the service account with the IAM role:
   ```bash
   kubectl annotate serviceaccount wezvatech-admin eks.amazonaws.com/role-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/eks-vault-role
   ```

---

## Deployment Strategies

### 1. Canary Deployment (Frontend)

#### a. Deploy the Frontend
Apply the rollout configuration:
```bash
kubectl apply -f rollout.yml
```

Monitor the rollout:
```bash
kubectl argo rollouts get rollout canary-demo
```

---

### 2. Blue-Green Deployment (MongoDB)

#### a. Deploy MongoDB (Blue Environment)
Apply the MongoDB deployment:
```bash
kubectl apply -f deploymongodb.yml
```

#### b. Deploy the New MongoDB Service (Green Environment)
Apply the new service:
```bash
kubectl apply -f mongodb-newsvc.yml
```

#### c. Switch Between Blue and Green
Update the active service (`mongo-svc`) to point to the new pods:
```bash
kubectl patch service mongo-svc -p '{"spec":{"selector":{"app":"mongo-new"}}}'
```

---

### 3. Secure Secrets Management with Kubernetes Secrets

As a fallback, store MongoDB credentials in Kubernetes Secrets:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-creds
data:
  username: YWRtaW51c2Vy
  password: cGFzc3dvcmQxMjM=
```

Apply the secret:
```bash
kubectl apply -f mongodb-secrets.yaml
```

---

## Verify the Deployment

1. **Check the Pods**:
   ```bash
   kubectl get pods
   ```

2. **Monitor Argo Rollouts**:
   ```bash
   kubectl argo rollouts get rollout canary-demo
   ```

3. **Verify MongoDB**:
   ```bash
   kubectl get service mongo-svc -o yaml
   ```

4. **Check Injected Secrets**:
   ```bash
   kubectl exec -it <pod-name> -- cat /vault/secrets/database-config.txt
   ```

---

## Cleanup

To delete the deployment and associated resources:
```bash
kubectl delete -f rollout.yml
kubectl delete -f deploymongodb.yml
kubectl delete -f mongodb-newsvc.yml
kubectl delete -f mongodb-secrets.yaml
kubectl delete serviceaccount wezvatech-admin
```

---

## Outcome

- **Secure Secrets Management**: All sensitive information is securely stored in Vault and injected into pods at runtime.
- **Zero Downtime Updates**: MongoDB updates are now seamless with the Blue-Green Deployment strategy.
- **Gradual Rollouts**: Frontend updates are rolled out gradually using the Canary Deployment strategy, reducing the risk of failures.
- **Improved Observability**: Deployment progress and health are now easily monitored using Argo Rollouts.

---

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.
```

