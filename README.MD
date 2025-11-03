## Docker, Kubernetes, AWS EKS Deployments

1.  Source Code for the app is available under `src` directory.

2.  Dockerfile for the Flask app is defined [here](./src/Dockerfile).
    Docker compose file with services for Flask and MongoDB is defined [here](./docker-compose.yml)

    Within `src` directory, run the following to build the image:

    ```
    docker build -t todo-flask .
    ```

    Then tag the image & push the image to Docker registry:

    ```docker
    docker tag todo-flask kevzpeter/todo-flask:latest
    docker push kevzpeter/todo-flask:latest
    ```

    ![Docker images](./media/docker%20images.png)

    Link to Docker Hub image: https://hub.docker.com/repository/docker/kevzpeter/todo-flask/general

    To run the app using Docker Compose, run the following from the root directory:

    ```docker
     docker compose up -d
    ```

    ![Docker Compose app running](./media/docker%20compose%20status.png)

    App will be running on `http://localhost:5000/`

    ![Docker images](./media/docker%20compose%20app%20running.png)

3.  To deploy the app locally using Minikube:

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

    ```kubernetes
    kubectl apply -f mongo-pvc.yaml
    kubectl apply -f mongo-deployment.yaml
    kubectl apply -f mongo-service.yaml
    kubectl apply -f web-deployment.yaml
    kubectl apply -f web-service.yaml
    ```

    Confirm running pods using:

    ```
    kubectl get pods
    ```

    ![Minikube pods](./media/minkube%202%20pods.png)

    Access the app using:

    ```
    minikube service web
    ```

    Open the app in your browser via NodePort
    ![Minikube service](./media/minikube%202_2%20service.png)
    ![Minikube deployment](./media/minikube%20deployment.png)
    Minikube Pod Recovery demo:

    ![Pod Recovery](./media/minikube%203%20pod%20repair.png)

4.  Deployment on AWS EKS
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

    ```
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

    ```
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
