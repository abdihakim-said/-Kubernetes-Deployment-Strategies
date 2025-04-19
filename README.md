
# **Enterprise Project: Kubernetes Deployment Strategies on Amazon EKS**

---
![argo-rollout](https://github.com/user-attachments/assets/ce623999-118d-4230-8944-9e5ddd3be07d)



## **Scenario Overview**

A large e-commerce enterprise is modernizing its infrastructure by adopting **Amazon EKS (Elastic Kubernetes Service)** for container orchestration. The company’s web application consists of a **React-based front-end** and a **MongoDB backend**. The development team wants to implement **robust deployment strategies** to ensure seamless updates, zero downtime, and enhanced security for their application.

The project focuses on implementing **Canary Deployments**, **Blue-Green Deployments**, and **Secret Management** to achieve the following goals:
1. **Minimize downtime** during deployments.
2. **Enable quick rollbacks** in case of issues.
3. **Secure sensitive data** like database credentials.
4. **Automate deployment processes** for scalability and efficiency.

---

## **Challenges Faced by the Enterprise**

1. **Risk of Downtime**:
   - Deploying new versions of the front-end or backend often caused service interruptions, leading to poor user experience.

2. **Rollback Complexity**:
   - Rolling back to a previous version after a failed deployment was manual and error-prone.

3. **Security Concerns**:
   - Database credentials were managed manually, increasing the risk of leaks or mismanagement.

4. **Lack of Deployment Automation**:
   - The team relied on basic Kubernetes rolling updates, which lacked advanced deployment controls like traffic splitting and monitoring.

---

## **Proposed Solution**

To address these challenges, the following strategies were implemented:

### **1. Front-End Deployment: Canary Strategy**
- Gradually roll out new versions of the front-end service to a subset of users.
- Start by deploying **20% of new pods** running the updated version.
- Incrementally increase the percentage to **50%**, then **100%** after monitoring for errors or performance issues.
- Use **Argo Rollouts** to manage the Canary deployment and enable quick rollbacks if issues are detected.

### **2. Backend Deployment: Blue-Green Strategy**
- Create a **parallel environment** with the new version of the MongoDB backend.
- Test the new backend infrastructure in isolation to ensure it works as expected.
- Switch traffic to the new environment only after full verification.
- Use Kubernetes services to route traffic between the old (blue) and new (green) environments, ensuring **zero-downtime deployments**.

### **3. Secret Management with HashiCorp Vault**
- Use **HashiCorp Vault** to dynamically manage and inject database credentials into Kubernetes pods.
- Automatically rotate secrets to enhance security.
- Remove manual secret management, reducing the risk of human error and credential leaks.

---

## **Tools and Technologies Used**

1. **Amazon EKS**:
   - Fully managed Kubernetes service for deploying and managing containerized applications.

2. **Argo Rollouts**:
   - Advanced deployment controller for Kubernetes.
   - Supports Canary, Blue-Green, and other deployment strategies.

3. **HashiCorp Vault**:
   - Securely manages and injects secrets into Kubernetes pods.

4. **AWS Load Balancer Controller**:
   - Manages Application Load Balancers (ALBs) and Network Load Balancers (NLBs) for Kubernetes services.

5. **AWS IAM Roles for Service Accounts (IRSA)**:
   - Provides fine-grained permissions for Kubernetes pods to access AWS resources securely.

---

## **Implementation Steps**

### **Step 1: Set Up the Amazon EKS Cluster**
1. **Create an EKS Cluster**:
   - Use the AWS Management Console, AWS CLI, or Terraform to create an EKS cluster.
   - Ensure the cluster has worker nodes (EC2 instances or Fargate profiles) for running workloads.

2. **Install Kubernetes Tools**:
   - Install `kubectl` and configure it to connect to the EKS cluster.
   - Install the **AWS CLI** and configure it with appropriate IAM credentials.

3. **Install AWS Load Balancer Controller**:
   - Deploy the AWS Load Balancer Controller to manage ALBs for Kubernetes services.

---

### **Step 2: Implement Canary Deployment for Front-End**
1. **Install Argo Rollouts**:
   - Deploy Argo Rollouts in the EKS cluster:
     ```bash
     kubectl create namespace argo-rollouts
     kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
     ```

2. **Define a Canary Deployment**:
   - Create a `Rollout` resource for the front-end service:
     ```yaml
     apiVersion: argoproj.io/v1alpha1
     kind: Rollout
     metadata:
       name: frontend-rollout
       namespace: default
     spec:
       replicas: 5
       strategy:
         canary:
           steps:
             - setWeight: 20
             - pause: { duration: 60s }
             - setWeight: 50
             - pause: { duration: 60s }
             - setWeight: 100
       selector:
         matchLabels:
           app: frontend
       template:
         metadata:
           labels:
             app: frontend
         spec:
           containers:
             - name: frontend
               image: frontend-app:v2
     ```

3. **Monitor the Deployment**:
   - Use the Argo Rollouts dashboard to monitor the progress of the Canary deployment:
     ```bash
     kubectl argo rollouts dashboard
     ```

---

### **Step 3: Implement Blue-Green Deployment for Backend**
1. **Deploy the New Backend Version**:
   - Deploy the new version of the MongoDB backend in a parallel environment (green).
   - Example `Deployment` YAML:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: backend-green
       namespace: default
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: backend-green
       template:
         metadata:
           labels:
             app: backend-green
         spec:
           containers:
             - name: backend
               image: backend-app:v2
     ```

2. **Switch Traffic Using Kubernetes Services**:
   - Update the Kubernetes service to route traffic to the green environment:
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: backend-service
       namespace: default
     spec:
       selector:
         app: backend-green
       ports:
         - protocol: TCP
           port: 27017
           targetPort: 27017
     ```

3. **Test and Verify**:
   - Test the new backend version before switching traffic.
   - Roll back to the blue environment if issues are detected.

---

### **Step 4: Secure Secrets with HashiCorp Vault**
1. **Install HashiCorp Vault**:
   - Deploy HashiCorp Vault in the EKS cluster or use a managed Vault instance.

2. **Enable Kubernetes Authentication**:
   - Configure Vault to authenticate Kubernetes pods using service accounts.

3. **Inject Secrets into Pods**:
   - Annotate the backend pod to inject database credentials from Vault:
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: backend-pod
       annotations:
         vault.hashicorp.com/agent-inject: "true"
         vault.hashicorp.com/role: "backend-role"
         vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/mongodb"
     spec:
       containers:
         - name: backend
           image: backend-app:v2
           env:
             - name: DB_USER
               valueFrom:
                 secretKeyRef:
                   name: vault-secret
                   key: username
             - name: DB_PASSWORD
               valueFrom:
                 secretKeyRef:
                   name: vault-secret
                   key: password
     ```

---

## **Benefits of the Solution**

1. **Minimized Downtime**:
   - Canary and Blue-Green strategies ensure seamless updates with no service interruptions.

2. **Quick Rollbacks**:
   - Argo Rollouts enables instant rollback to the previous version if issues are detected.

3. **Enhanced Security**:
   - HashiCorp Vault automates secret management, reducing the risk of credential leaks.

4. **Improved Deployment Confidence**:
   - Testing in parallel environments (Blue-Green) and gradual rollouts (Canary) allow teams to deploy with confidence.

5. **Scalability**:
   - The solution is scalable and can be extended to other microservices in the application.

---

## **Conclusion**

By leveraging **Amazon EKS**, **Argo Rollouts**, and **HashiCorp Vault**, this project demonstrates how enterprises can implement advanced deployment strategies like **Canary** and **Blue-Green Deployments** to achieve seamless updates, quick rollbacks, and enhanced security. This approach ensures high availability and reliability for modern cloud-native applications.
