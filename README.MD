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
