## Docker, Kubernetes, AWS EKS Deployments

### Part 1: Creating the Application

Source Code for the app is available under [src](./src/) directory.

### Part 2: Containerizing the Application on Docker

Dockerfile for the Flask app is defined [here](./src/Dockerfile).
Docker compose file with services for Flask and MongoDB is defined [here](./docker-compose.yml)

Within `src` directory, run the following to build the image:

```bash
docker build -t todo-flask .
```

Then tag the image & push the image to Docker registry:

```bash
docker tag todo-flask kevzpeter/todo-flask:latest
docker push kevzpeter/todo-flask:latest
```

![Docker images](./media/docker%20images.png)

Link to Docker Hub image: https://hub.docker.com/repository/docker/kevzpeter/todo-flask/general

To run the app using Docker Compose, run the following from the root directory:

```bash
docker compose up -d
```

![Docker Compose app running](./media/docker%20compose%20status.png)

App will be running on `http://localhost:5000/`

![Docker images](./media/docker%20compose%20app%20running.png)

### Part 3: Deploying the Application on Minikube

After installing Minikube, start local node:

```bash
minikube start
```

Make sure node is up and running:

```bash
kubectl get nodes
```

![Minikube node](./media/minikube%201.png)

| File                    | Description                            |
| ----------------------- | -------------------------------------- |
| `mongo-deployment.yaml` | MongoDB Deployment definition          |
| `mongo-service.yaml`    | Service to expose MongoDB internally   |
| `mongo-pvc.yaml`        | PersistentVolumeClaim for MongoDB data |
| `web-deployment.yaml`   | Flask web app Deployment definition    |
| `web-service.yaml`      | NodePort Service to expose the web app |

Run the following commands:

```bash
kubectl apply -f mongo-pvc.yaml
kubectl apply -f mongo-deployment.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
```

Confirm running pods using:

```bash
kubectl get pods
```

![Minikube pods](./media/minkube%202%20pods.png)

Access the app using:

```bash
minikube service web
```

Open the app in your browser via NodePort
![Minikube service](./media/minikube%202_2%20service.png)
![Minikube deployment](./media/minikube%20deployment.png)

Minikube Pod Recovery demo:

![Pod Recovery](./media/minikube%203%20pod%20repair.png)

### Part 4: Deploying the Application on AWS EKS

Set up a managed Kubernetes cluster on AWS Elastic Kubernetes Service (EKS), configure IAM permissions, create a worker node group, and deploy the MongoDB + Flask web application.

Create EKS cluster using `eksctl` CLI tool:

```bash
eksctl create cluster \
--name todo-cluster-v2 \
--region us-east-1 \
--nodegroup-name todo-nodes \
--node-type t3.small \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--managed
```

![EKS Cluster](./media/eks%20cluster.png)
![EKS Node Group](./media/eks%20node%20group.png)

After creation, update your local kubeconfig to point to the new cluster:

```bash
aws eks update-kubeconfig --region us-east-1 --name todo-cluster-v2
```

Verify connection:

```bash
kubectl get nodes
kubectl get pods
kubectl get svc
```

![EKS Pods, SVC](./media/eks%20nodes.png)
![EKS Pods, SVC](./media/eks%20pods_svc.png)

IAM stuff:

When connecting as an IAM user, EKS blocks access to Kubernetes objects unless explicitly granted.

Retrieve the node instance role from the AWS console (e.g. eksctl-todo-cluster-v2-nodegroup-t-NodeInstanceRole-hYGskgKzuMc2).

Add IAM identity mapping so worker nodes can authenticate to the cluster:

```bash
    eksctl create iamidentitymapping \
--region us-east-1 \
--cluster todo-cluster-v2 \
--arn arn:aws:iam::700638340879:role/eksctl-todo-cluster-v2-nodegroup-t-NodeInstanceRole-hYGskgKzuMc2 \
--group system:bootstrappers \
--group system:nodes \
--username system:node:{{EC2PrivateDNSName}}
```

I also had to install EBS CSI Driver

```bash
eksctl create addon \
--name aws-ebs-csi-driver \
--cluster todo-cluster-v2 \
--region us-east-1 \
--service-account-role-arn arn:aws:iam::700638340879:role/AmazonEKS_EBS_CSI_DriverRole \
--force
```

Once EKS Cluster and Node group are ready, we deploy the MongoDB and Flask app manifests:

```bash
kubectl apply -f mongo-pvc.yaml
kubectl apply -f mongo-deployment.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
```

![EKS](./media/eks%20pvc.png)
![EKS](./media/eks%20storage%20class.png)

Deployed app on EKS:

![EKS](./media/eks%20app.png)

### Part 5: Deployments and ReplicaSets

Replicas are defined in [web-deployment.yaml](./k8s/web-deployment.yaml) file.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
```

Apply updated manifest:

```bash
kubectl apply -f web-deployment.yaml
```

Verify ReplicaSet creation:

```bash
kubectl get rs
```

Demo:
![Part 5](./media/part%205%20-%201.png)

Simulate Pod recovery by deleting one of the pods:

```bash
kubectl delete pod <pod-name>
```

ReplicaSet automatically created a replacement pod to maintain the desired count.

Demo:

![Part 5.2](./media/part%205%20-%202.png)

Scale the Deployment

Eg: Scaling down deployment from 2 pods to 1

```bash
kubectl scale deployment web --replicas=1
kubectl get pods
```

Demo:
![Part 5.3](./media/part%205%20-%203.png)

### Part 6: Rolling Update Strategy

Update [web-deployment.yaml](./k8s/web-deployment.yaml) with rolling update strategy:

```yaml
spec:
replicas: 2
strategy:
  type: RollingUpdate
  rollingUpdate:
  maxUnavailable: 1
  maxSurge: 1
```

Update the docker image in the deployment (eg: kevzpeter/todo-flask:v2)

Apply deployment:

```bash
kubectl apply -f web-deployment.yaml
```

Check rollout status

```bash
kubectl rollout status deployment/web
```

Deployment is updated and new pods are brought up.

Demo:

![Part 6](./media/part%206%20-%201.png)
![Part 6](./media/part%206%20-%202.png)
![Part 6](./media/part%206%20-%201_2.png)
![Part 6](./media/part%206%20-%203.png)
![Part 6](./media/part%206%20-%204.png)

### Part 7: Health Monitoring

I added liveness and readiness probes in the [web-deployment.yaml](./k8s/web-deployment.yaml) manifest to allow Kubernetes to periodically test pod health.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /
    port: 5000
  initialDelaySeconds: 20
  periodSeconds: 10
```

| Probe               | Purpose                                           | Action                                            |
| ------------------- | ------------------------------------------------- | ------------------------------------------------- |
| **Readiness Probe** | Checks if the pod is ready to serve traffic       | If it fails -> pod removed from Service endpoints |
| **Liveness Probe**  | Detects if the container is stuck or unresponsive | If it fails -> pod is restarted automatically     |

Apply the new manifest:

```bash
kubectl apply -f web-deployment.yaml
kubectl get pods
```

To simulate an application crash, we intentionally stopped the gunicorn process (production server) inside one of the running pods.

- Open shell in the web pod:

  ```bash
  kubectl exec -it $(kubectl get pod -l app=web -o name | head -n1) -- /bin/sh
  ```

- Kill the running process (simulate crash):

  ```bash
  ps aux
  kill 1
  ```

The pod immediately transitions to a CrashLoopBackOff state. Kubernetes restarts it automatically.

Demo:

![Part 7](./media/part%207%20%20-1.png)
![Part 7](./media/part%207%20-3.png)

<!-- ![Part 7](./media/part%207%20-3_3.png) -->

![Part 7](./media/part%207%20-3_4.png)

Testing intentional deletion demo:

![Part 7.4](./media/part%207%20-%204.png)
![Part 7.4](./media/part%207%20-%204.2.png)

### Part 8: Alerting | Prometheus + Slack integration

Prometheus setup using helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

![Prometheus setup](./media/part%208%20-%201.png)
![Prometheus setup](./media/part%208%20-%202.png)

Port forwarding for local access:

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

![Prometheus dashboard](./media/part%208%20-%203%20.png)

I added a PrometheusRule ([web-pod-alert.yaml](./k8s/web-pod-alert.yaml)) to alert when a pod restarts frequently.

Apply this manifest

```bash
kubectl apply -f web-pod-alert.yaml -n monitoring

```

![web-pod-alert](./media/part%208%20-%204.png)

Slack bot integration

More information can be found on Slack's documentation page: https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/

- Create a Slack Incoming Webhook
  ![slack-incoming-webhook](./media/part%208%20-%20slack%20incoming%20web%20hooks.png)
- Add bot to channel (#all-aws-eks) in custom workspace I created (aws-eks)
  ![prometheus-bot](./media/part%208%20-%20slack%20bot%20integration.png)
- Copy the webhook URL (e.g. https://hooks.slack.com/services/<>/<>/<>)

Update alert manager manifest by creating a new secret from [alertmanager.yaml](./k8s/alertmanager.yaml):

```bash
kubectl delete secret alertmanager-prometheus-kube-prometheus-alertmanager -n monitoring
kubectl create secret generic alertmanager-prometheus-kube-prometheus-alertmanager -n monitoring --from-file=alertmanager.yaml
kubectl delete pod -l app.kubernetes.io/name=alertmanager -n monitoring
```

Wait until the new Alertmanager pod is Running.

Example: KubeMemoryOverCommit Error and Notification message on Slack:

![KML](./media/part%208%20-%20kube%20memory%20overcommit.png)
![KML 2](./media/part%208%20-%20slack%20message.png)

Then, I simulated a crash loop by breaking the web container using command:

```bash
kubectl patch deployment web -p '{"spec":{"template":{"spec":{"containers":[{"name":"web","command":["false"]}]}}}}'
```

![simulating high pod restarts](./media/part%208%20-%20simulating%20high%20pod%20restarts.png)

HighPodRestarts Firing in Prometheus Dashboard

![HighPodRestarts](./media/part%208%20-%20highpodrestarts.png)

Notification on Slack channel

![HighPodRestarts Message](./media/part%208%20-%20highpodrestarts%20slack.png)
