### Challenge 3 [Deploy tornado app using K8s]:
[![forthebadge](https://forthebadge.com/images/badges/certified-yourboyserge.svg)](https://forthebadge.com)
[![forthebadge](https://forthebadge.com/images/badges/cc-0.svg)](https://forthebadge.com)
### Setting up the environment:
*Although the challenge required a deployment using multi node cluster, i've only experimented with deployment using Minikube till this point (shouldn't differ that much on a multi node cluster).

Vagrant was used with ansible as a provisioner to install the following on the vagrant machine:
- Docker
- Docker-compose
- Portainer
- Kubectl
- Minikube

After starting the machine, the following commands were entered to start using minikube:
```bash
minikube start --driver=none
```

---

### Handling Dockerfile: 
The Dockerfile was pushed to Dockerhub so that it can be used in K8s yml file
#### Dockerfile:
```Dockerfile
FROM python:alpine

WORKDIR /usr/src/app

ENTRYPOINT [ "python3" ]

RUN apk add git && \
    git clone https://github.com/MohamedMSaeed/DevOps-Challenge-Demo-Code.git . && \
    pip install -r requirements.txt
```
* **git clone url .** is used in order to fetch the files directly without the need for a parent directoy, the condition that must be fulfilled is that this command must be executed within an empty directory.

Image creation with the build command
```bash
docker build -t tornado-app .
```

Verify using `docker image ls` command
```bash
docker image ls
REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
tornado-app                               latest              91b74bf84c75        4 hours ago         107MB
```

First login to dockerhub:
```
docker login
```

Tagging and [pushing the image](https://hub.docker.com/repository/docker/thejaxon/tornado-app)
```bash
docker tag tornado-app thejaxon/tornado-app:v1

docker push thejaxon/tornado-app:v1
```

---

### Deploying the app using Kubernetes:
2 Deployments and 2 Services were created.
Deployment 1 is for the tornado-app and it contains the env variables that are needed by the app, it also contains the `port` that will be used to access the deployed app.

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata:
  name: tornado-app

spec:
  replicas: 3
  selector:
    matchLabels:
      service: challenge-3
  template:
    metadata:
      labels:
        service: challenge-3 
    spec: 
      containers:
        - name: tornado
          image: thejaxon/tornado-app:v1
          env: 
          - name: REDIS_HOST 
            value: "redis"
          - name: REDIS_PORT 
            value: "6379"
          - name: REDIS_DB 
            value: "0"
          - name: ENVIRONMENT 
            value: "dev"
          - name: HOST
            value: "localhost"
          - name: PORT
            value: "4222"
          ports:
          - containerPort: 4222
          command: ["python3", "hello.py"]
```

Deployment 2 is for redis.
```yaml
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: redis

spec: 
  replicas: 2
  selector:
    matchLabels:
      service: challenge-3 
  template: 
    metadata:
      labels: 
        service: challenge-3 
    spec:
      
      containers:
      - name: redis 
        image: redis:alpine
```

Service 1 is for tornado-app and is of type `NodePort` so that it allows for app access from outside the cluster as in `http://$IP:$PORT`
so after deploying and running `kubectl get svc`
```bash
vagrant@ServiceDiscovery:/vagrant/k8s$ sudo kubectl get  svc 
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP          167m
redis         ClusterIP   10.101.89.40     <none>        6379/TCP         46m
tornado-app   NodePort    10.106.135.208   <none>        4222:31932/TCP   46m
```
The CLUSTER-IP of tornado-app will be used with the port number to access the app.

```yaml
apiVersion: v1 
kind: Service 
metadata: 
  name: tornado-app 

spec: 
  type: NodePort 
  ports:
  - port: 4222 
  selector: 
    service: challenge-3
```

Service 2 is of type `ClusterIP` and is used locally between the pods (this allows communication between the app and redis).
```yaml
apiVersion: v1 
kind: Service 
metadata: 
  name: redis 

spec: 
  type: ClusterIP 
  ports:
  - port: 6379
  selector: 
    service: challenge-3
```

Finally the command `kubectl create -f challenge-3.yml --save-config ` was used to create the deployments and services defined.

```bash
sudo kubectl create -f challenge-3.yml --save-config 
deployment.apps/tornado-app created
service/tornado-app created
deployment.apps/redis created
service/redis created
```

### Final result:
![Preview](https://github.com/theJaxon/devops-ITI-challenges/blob/master/challenge-03/etc/Preview_Final_Result.jpeg)
