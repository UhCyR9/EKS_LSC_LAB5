
# Kubernetes Deployment with Amazon EKS and NFS

Follow these steps to set up an EKS cluster, configure Kubernetes, and deploy an application using NFS provisioning.

---

## 1) Creating an EKS Cluster

1. **Create the EKS Cluster**  
   - Go to **EKS** in AWS Console.
   - Select **Add cluster** > **Create**.
   - Name the cluster (e.g., `DemoEKSCluster`) and select the role from step 2.
   - Keep default settings for other options.
   - Click **Create**.

2. **Add Node Group**  
   - Go to **Compute** tab in EKS dashboard and click **Add Node Group**.
   - Name it (e.g., `DemoNodeGroup`), choose an IAM role with **AmazonEKSWorkerNodePolicy** and **AmazonEC2ContainerRegistryReadOnly** policies.
   - Select instance type and scaling options.
   - Click **Create**.

4. **Install kubectl**
   - Download the latest `kubectl` binary and install it:
     ```
      curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
      sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
     ```


3. **Configure kubectl**  
   - Ensure AWS CLI and kubectl are installed.
   - Configure AWS CLI: `aws configure`.
   - Update kubeconfig for the cluster:
     ```bash
     aws eks --region us-east-1 update-kubeconfig --name DemoEKSCluster
     ```

---

## 2) Installing Helm and NFS Server Provisioner

1. **Install Helm**  
   ```bash
   curl -fsSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

2. **Add Helm Chart Repository**  
   ```bash
   helm repo add stable https://charts.helm.sh/stable
   helm repo update
   ```

3. **Install NFS Server Provisioner**  
   ```bash
   helm install nfs-provisioner stable/nfs-server-provisioner      --set storageClass.name=custom-nfs-storage      --set storageClass.defaultClass=true
   ```

---

## 3) Creating Kubernetes Resources

### Persistent Volume Claim (PVC)

1. Create a file `persistent-volume-claim.yaml` with the following content:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: shared-pvc
   spec:
     storageClassName: custom-nfs-storage
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 2Gi
   ```

2. Apply the PVC configuration:

   ```bash
   kubectl apply -f persistent-volume-claim.yaml
   ```

### Deployment

1. Create a file `apache-deployment.yaml`:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: apache-server
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: apache
     template:
       metadata:
         labels:
           app: apache
       spec:
         containers:
         - name: apache
           image: httpd:latest
           ports:
           - containerPort: 80
           volumeMounts:
           - name: web-content
             mountPath: /usr/local/apache2/htdocs/
         volumes:
         - name: web-content
           persistentVolumeClaim:
             claimName: shared-pvc
   ```

2. Deploy using `kubectl`:

   ```bash
   kubectl apply -f apache-deployment.yaml
   ```

### Service

1. Create a file `apache-service.yaml`:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: apache-service
   spec:
     type: LoadBalancer
     selector:
       app: apache
     ports:
     - port: 80
       targetPort: 80
   ```

2. Apply the Service configuration:

   ```bash
   kubectl apply -f apache-service.yaml
   ```

### Job

1. Create a file `content-copy-job.yaml`:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: content-copy-job
   spec:
     template:
       spec:
         containers:
         - name: content-copier
           image: alpine
           command: ['sh', '-c', 'echo "<h1>Hello from Apache Server!</h1>" > /data/index.html']
           volumeMounts:
           - name: content-volume
             mountPath: /data
         restartPolicy: OnFailure
         volumes:
         - name: content-volume
           persistentVolumeClaim:
             claimName: shared-pvc
   ```

2. Apply the Job configuration:

   ```bash
   kubectl apply -f content-copy-job.yaml
   ```

---

## 4) Accessing the Application

Retrieve the IP address of the `apache-service`:

```bash
kubectl get service apache-service
```

Open the IP address in a browser to confirm the Apache server displays the message.
