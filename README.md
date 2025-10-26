# Microservice App - PRFT DevOps Training

This is the application you are going to use throughout the whole training. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although it is a TODO application, it is interesting because the microservices that compose it are written in different programming languages or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments.

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, it does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.
<p align="center">
  ![microservice-app-example](/arch-img/Microservices.png)
</p>

## Solution

In this opportunity, the work involved deploying the apps using Kubernetes, network policies, config-maps, secrets, deployment strategies and HPA.

The following steps were performed.

## Step #1: Creating Dockerfiles

### Frontend Dockerfile:

```Dockerfile
FROM node:8.17.0-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

EXPOSE 8080

CMD ["npm", "start"]
```

In the Dockerfile:

- The node image is used.
- `/app` is set as the working directory.
- Node dependencies are copied.
- Node dependencies are installed.
- The necessary files for the application are copied.
- The application is built.
- `["npm", "start"]` is set as the entry point.

### Auth-Api Dockerfile:

Before creating the Dockerfile, it was necessary to initialize a Go module. This module was initialized with the help of:

```sh
export GO111MODULE=on
go mod init github.com/bortizf/microservice-app-example/tree/master/auth-api
go mod tidy
go build
```

Go modules were necessary to build the Dockerfile correctly.

```Dockerfile
FROM golang:1.22 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o auth-api .

#Create a small image

FROM alpine:latest

RUN apk --no-cache add ca-certificates libc6-compat

WORKDIR /app

COPY --from=builder /app/auth-api .

EXPOSE 8000

CMD ["./auth-api"]
```

In the Dockerfile, 2 stages are generated, one for building and one for deployment.

This is done to reduce the container size. By having 2 Docker images, Docker only keeps the one that uses the builder image.

In the build stage:

- The Go image is used as builder.
- The directory where files will be copied is set: `/app`
- The corresponding modules are copied.
- Necessary dependencies are downloaded.
- The corresponding binary is compiled.

In the deployment stage:

- The Alpine image is used.
- Necessary certificates for Alpine are added.
- The working directory is set: `/app`
- The executable from the first stage (builder) is copied.
- `["./auth-api"]` is set as the entry point.

### Users-Api Dockerfile:

```Dockerfile
FROM eclipse-temurin:8-jdk-jammy AS builder

WORKDIR /app

COPY .mvn/ .mvn

COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw -Dhttps.protocols=TLSv1.2 dependency:go-offline

COPY src ./src

RUN ./mvnw clean install -DskipTests

# Create a small image for running the app

FROM eclipse-temurin:8-jdk-jammy

WORKDIR /app

COPY --from=builder /app/target/*.jar target/users-api-0.0.1-SNAPSHOT.jar

EXPOSE 8083

ENTRYPOINT ["java", "-jar", "target/users-api-0.0.1-SNAPSHOT.jar"]
```

In the Dockerfile, 2 stages are generated, one for building and one for deployment.

This is done to reduce the container size. By having 2 Docker images, Docker only keeps the one that uses the builder image.

In the build stage:

- Java-8 is set as the builder image.
- `/app` is set as the working directory.
- Maven dependencies are copied and execution permissions are assigned.
- The source code is copied.
- The application is compiled.

In the deployment stage:

- Java-8 is set as the image.
- `/app` is set as the working directory.
- The executable file from the first stage is copied.
- `["java", "-jar", "target/users-api-0.0.1-SNAPSHOT.jar"]` is set as the entry point.

### TODOs-Api Dockerfile:

```Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

#Create a small image

FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app .

EXPOSE 8082

CMD ["npm", "start"]
```

In the Dockerfile, 2 stages are generated, one for building and one for deployment.

This is done to reduce the container size. By having 2 Docker images, Docker only keeps the one that uses the builder image.

In the build stage:

- The node image is used.
- `/app` is set as the working directory.
- Node dependencies are copied.
- Node dependencies are installed.
- The necessary files for the application are copied.
- The application is built.

In the deployment stage:

- The node image is used.
- The executable file from the first stage is copied.
- `["npm", "start"]` is set as the entry point.

### Log-Processor Dockerfile

```Dockerfile
FROM python:3.6-alpine

WORKDIR /app

RUN apk add --no-cache build-base

COPY . .

RUN pip3 install -r requirements.txt

CMD ["python3", "-u", "main.py"]
```

In the Dockerfile:

- The python 3.6 image is used.
- `/app` is set as the working directory.
- Necessary build tools are added.
- The necessary app files are copied.
- Python dependencies are installed.
- `["python3", "-u", "main.py"]` is set as the entry point.

## Step #2: Creating Config-maps and Secrets

After the Dockerfiles were created, the next step was to create the config-maps and secrets for each microservice.

Not all microservices require these because some of them don't need environment variables or don't have sensitive information.

### Frontend config-map

**config-map**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: front-config
  namespace: front
data:
  PORT: "8080"
  AUTH_API_ADDRESS: "http://auth-api-svc.auth-api.svc.cluster.local:8000"
  TODOS_API_ADDRESS: "http://todos-svc.todos.svc.cluster.local:8082"
```

In this config-map:

- The namespace it belongs to is defined (front).
- The necessary environment variables for the frontend are defined (the domain name of the auth-api service and the TODOs-api service).

### Auth-api config-map and Auth-api secret

**config-map**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-api-config
  namespace: auth-api
data:
   AUTH_API_PORT: "8000"
   USERS_API_ADDRESS: "http://users-api-svc.users-api.svc.cluster.local:8083"
```

In this config-map:

- The namespace it belongs to is defined (auth-api).
- The necessary environment variables for the auth-api are defined (the domain name of the users-api service and the corresponding port).

**secret**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-api-secret
  namespace: auth-api
type: Opaque
data:
  JWT_SECRET: UFJGVA==
```

In this secret:

- The namespace it belongs to is defined (auth-api).
- The necessary secrets for auth-api are defined. In this case JWT_SECRET, which is encoded in base64.

### Users-api config-map and Users-api secret

**config-map**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: users-api-config
  namespace: users-api
data:
  SERVER_PORT: "8083"
```

- The namespace it belongs to is defined (users-api).
- The necessary environment variables for the users-api are defined (the users-api server port).

**secret**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: users-api-secret
  namespace: users-api
type: Opaque
data:
  JWT_SECRET: UFJGVA==
```

- The namespace it belongs to is defined (users-api).
- The necessary secrets for users-api are defined. In this case JWT_SECRET, which is encoded in base64.

### TODOs config-map and TODOs secret

**config-map**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: todos-config
  namespace: todos
data:
  TODO_API_PORT: "8082"
  REDIS_HOST: "redis-svc.redis.svc.cluster.local"
  REDIS_PORT: "6379"
  REDIS_CHANNEL: "log_channel"
```

- The namespace it belongs to is defined (todos).
- The necessary environment variables for the TODOs-api are defined:
    - The TODOs server port.
    - The Redis hostname.
    - The Redis port.
    - The Redis channel, which will be used to send messages to this Redis database.

**secret**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: todos-secret
  namespace: todos
type: Opaque
data:
  JWT_SECRET: UFJGVA==
```

- The namespace it belongs to is defined (todos).
- The necessary secrets for TODOs are defined. In this case JWT_SECRET, which is encoded in base64.

### Log-processor config-map

**config-map**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logs-config
  namespace: logs
data:          
   REDIS_HOST: "redis-svc.redis.svc.cluster.local"
   REDIS_PORT: "6379"
   REDIS_CHANNEL: "log_channel"
```

- The namespace it belongs to is defined (logs).
- The necessary environment variables for logs are defined:
    - The Redis hostname.
    - The Redis port.
    - The Redis channel (which will be used to retrieve messages arriving at this Redis database).

## Step #3: Creating deployments, services and HPA

The next step was creating the deployments required to build the application, the services to access each microservice and the HPA to manage the traffic.

### Frontend deployment, service and HPA

We are going to create two frontends. The first one is going to manage the stable version of our application and the second one will manage the latest version of our application.

Two deployment strategies will be used in the frontend: **The Canary** and **Rolling update strategy**.

**Canary strategy definition**

- Canary deployment works by gradually delivering traffic in production between two specific versions, starting with small percentages, such as 10/90%.

**Rolling update strategy definition**

- Rolling updates allow deployment updates to take place with zero downtime by incrementally updating Pods instances with new ones.

*In our case, we are going to create two replicas (with a stable image) in the stable deployment, and one replica (with the newest image) in the canary deployment. The service will handle the traffic and both new and stable versions will coexist.*

*We could decide to increase the version of our stable deployment and see how the rolling update strategy upgrades every pod (We are going to watch that later)*

**Stable deployment, service and HPA**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-stable
  namespace: front
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: front
      version: stable
  template:
    metadata:
      labels:
        app: front
        version: stable
    spec:
      containers:
      - name: front
        image: barcino/frontend:1.0.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        envFrom:
        - configMapRef:
            name: front-config

---
apiVersion: v1
kind: Service
metadata:
  name: front-svc
  namespace: front
spec:
  selector:
    app: front
  ports:
  - port: 8080
    targetPort: 8080
  type: LoadBalancer

--- 
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-front-stable
  namespace: front
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: front-stable
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**Canary deployment, service and HPA**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-canary
  namespace: front
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: front
      version: canary 
  template:
    metadata:
      labels:
        app: front
        version: canary
    spec:
      containers:
      - name: front
        image: barcino/frontend:1.1.0
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        envFrom:
        - configMapRef:
            name: front-config
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-front-canary
  namespace: front
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: front-canary
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

For both canary and stable the structure is very similar.

**In the deployment**

- The namespace was selected (front).
- The rolling update deployment strategy was selected.
- It was set that the deployment will manage the labels with the name front.
- Two/one replica(s) (pods) were created with the image `barcino/frontend:1.0.x` built with the Dockerfile.
- 8080 was defined as the container port.
- CPU resources were assigned to each pod.
- The config maps created previously were added.

**In the service**

- The namespace was selected (front).
- The pods with the front labels were selected.
- The port (Container port) and the target port (Application port) were set.
- The service type was set as `LoadBalancer`.

**In the HPA**

- The namespace was selected (front).
- The minimum (1/2) and maximum (3/5) number of replicas were set.
- The metrics to autoscale were set.

### Auth-api deployment, service and HPA

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: auth-api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-api
  namespace: auth-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-api
  template:
    metadata:
      labels:
        app: auth-api
    spec:
      containers:
      - name: auth-api
        image: barcino/auth-api:1.0.0
        ports:
        - containerPort: 8000
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        envFrom:
        - configMapRef:
            name: auth-api-config
        - secretRef:
            name: auth-api-secret
---
apiVersion: v1
kind: Service
metadata:
  name: auth-api-svc
  namespace: auth-api
spec:
  selector:
    app: auth-api
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-auth-api
  namespace: auth-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-api
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**In the deployment**

- The namespace was selected (auth-api).
- The rolling update deployment strategy was selected.
- It was set that the deployment will manage the labels with the name auth-api.
- Two replicas (pods) were created with the image `barcino/auth-api:1.0.0` built with the Dockerfile.
- 8000 was defined as the container port.
- CPU resources were assigned to each pod.
- The config maps and secrets created previously were added.

**In the service**

- The namespace was selected (auth-api).
- The pods with the auth-api labels were selected.
- The port (Container port) and the target port (Application port) were set.
- The service type was set as `ClusterIP`.

**In the HPA**

- The namespace was selected (auth-api).
- The minimum (2) and maximum (5) number of replicas were set.
- The metrics to autoscale were set.

### Users-api deployment, service and HPA

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: users-api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-api
  namespace: users-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: users-api
  template:
    metadata:
      labels:
        app: users-api
    spec:
      containers:
      - name: users-api
        image: barcino/users-api:1.0.0
        ports:
        - containerPort: 8083
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        envFrom:
        - configMapRef:
            name: users-api-config
        - secretRef:
            name: users-api-secret
---
apiVersion: v1
kind: Service
metadata:
  name: users-api-svc
  namespace: users-api
spec:
  selector:
    app: users-api
  ports:
  - port: 8083
    targetPort: 8083
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-users-api
  namespace: users-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: users-api
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**In the deployment**

- The namespace was selected (users-api).
- The rolling update deployment strategy was selected.
- It was set that the deployment will manage the labels with the name users-api.
- Two replicas (pods) were created with the image `barcino/users-api:1.0.0` built with the Dockerfile.
- 8083 was defined as the container port.
- CPU resources were assigned to each pod.
- The config maps and secrets created previously were added.

**In the service**

- The namespace was selected (users-api).
- The pods with the users-api labels were selected.
- The port (Container port) and the target port (Application port) were set.
- The service type was set as `ClusterIP`.

**In the HPA**

- The namespace was selected (users-api).
- The minimum (2) and maximum (5) number of replicas were set.
- The metrics to autoscale were set.

### TODOs-api deployment and service

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: todos
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todos
  namespace: todos
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todos
  template:
    metadata:
      labels:
        app: todos
    spec:
      containers:
      - name: todos
        image: barcino/todos:1.0.0
        ports:
        - containerPort: 8082
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        envFrom:
        - configMapRef:
            name: todos-config
        - secretRef:
            name: todos-secret
---
apiVersion: v1
kind: Service
metadata:
  name: todos-svc
  namespace: todos
spec:
  selector:
    app: todos
  ports:
  - port: 8082
    targetPort: 8082
  type: ClusterIP
```

**In the deployment**

- The namespace was selected (todos).
- The rolling update deployment strategy was selected.
- It was set that the deployment will manage the labels with the name todos.
- One replica (pod) were created with the image `barcino/todos-api:1.0.0` built with the Dockerfile.
- 8082 was defined as the container port.
- CPU resources were assigned to each pod.
- The config maps and secrets created previously were added.

**In the service**

- The namespace was selected (todos).
- The pods with the todos labels were selected.
- The port (Container port) and the target port (Application port) were set.
- The service type was set as `ClusterIP`.

In this case there is just on replica and there are not hpa. This is justified because the data is storage locally (not in redis).
If we have more replicas every time we call the TODO's microservices a different pod could answer.  

### Redis deployment, service

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.4
          ports:
           - containerPort: 6379
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
---
```

**In the deployment**

- The namespace was selected (redis).
- The rolling update deployment strategy was selected.
- It was set that the deployment will manage the labels with the name redis.
- One replica (pod) were created with the image `redis:7.4` built with the Dockerfile.
- 6379 was defined as the container port.
- CPU resources were assigned to each pod.
- The config maps created previously were added.

**In the service**

- The namespace was selected (redis).
- The pods with the redis labels were selected.
- The port (Container port) and the target port (Application port) were set.
- The service type was set as `ClusterIP`.

Something similar happen with redis. To avoid inconsistences, we are going to only have one replica and no hpa.

### Log-processor deployment, service and HPA

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logs
  namespace: logs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logs
  template:
    metadata:
      labels:
        app: logs
    spec:
      containers:
      - name: logs
        image: barcino/logs:1.0.0
        ports:
        - containerPort: 4000
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        envFrom:
        - configMapRef:
            name: logs-config
---
apiVersion: v1
kind: Service
metadata:
  name: logs-svc
  namespace: logs
spec:
  selector:
    app: logs
  ports:
  - port: 4000
    targetPort: 4000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-logs
  namespace: logs
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logs
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**In the deployment**

- The namespace was selected (logs).
- The rolling update deployment strategy was selected.
- It was set that the deployment will manage the labels with the name logs.
- Two replicas (pods) were created with the image `barcino/logs:1.0.0` built with the Dockerfile.
- 4000 was defined as the container port.
- CPU resources were assigned to each pod.
- The config maps created previously were added.

**In the service**

- The namespace was selected (logs).
- The pods with the logs labels were selected.
- The port (Container port) and the target port (Application port) were set.
- The service type was set as `ClusterIP`.

**In the HPA**

- The namespace was selected (logs).
- The minimum (1) and maximum (3) number of replicas were set.
- The metrics to autoscale were set.

## Step 4: Creating network policies

After creating the deployments for each microservice, the next step was creating policies in order to allow communication between only the necessary microservices.

### Default deny-policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: front
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: auth-api
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: users-api
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: todos
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: redis
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: logs
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
```

**The configured rules include:**
- Ingress: Complete denial of incoming traffic in all microservices.
- Egress: Complete denial of outgoing traffic in all microservices.

### Frontend policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: front-allow-ingress
  namespace: front
spec:
  podSelector:
    matchLabels:
      app: front
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: front-allow-egress
  namespace: front
spec:
  podSelector:
    matchLabels:
      app: front
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  
  # auth-api
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: auth-api
      podSelector:
        matchLabels:
          app: auth-api
    ports:
    - protocol: TCP
      port: 8000
        
  # todos
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: todos
      podSelector:
        matchLabels:
          app: todos
    ports:
    - protocol: TCP
      port: 8082
```

**The configured rules include:**
- Ingress: Ingress from the internet through port 8080.
- Egress: Egress to the auth-api microservice (8000) and the TODOs microservice (8082), and DNS resolution is used.

### Auth-api policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auth-api-allow-ingress
  namespace: auth-api
spec:
  podSelector:
    matchLabels:
      app: auth-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: front
      podSelector:
        matchLabels:
          app: front
    ports:
    - protocol: TCP
      port: 8000
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auth-api-allow-egress
  namespace: auth-api
spec:
  podSelector:
    matchLabels:
      app: auth-api
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  #users-api
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: users-api
      podSelector:
        matchLabels:
          app: users-api
    ports:
    - protocol: TCP
      port: 8083
```

**The configured rules include:**
- Ingress: Ingress from the frontend microservice.
- Egress: Egress to users-api and DNS resolution is used.

### Users-api policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: users-api-allow-ingress-from-auth-api
  namespace: users-api
spec:
  podSelector:
    matchLabels:
      app: users-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: auth-api
      podSelector:
        matchLabels:
          app: auth-api
    ports:
    - protocol: TCP
      port: 8083
  egress: []   
```

**The configured rules include:**
- Ingress: Ingress from the auth-api microservice.
- Egress: No egress allowed.

### TODOs policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: todos-allow-ingress-from-frontend
  namespace: todos
spec:
  podSelector:
    matchLabels:
      app: todos
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: front
      podSelector:
        matchLabels:
          app: front
    ports:
    - protocol: TCP
      port: 8082
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: todos-allow-egress
  namespace: todos
spec:
  podSelector:
    matchLabels:
      app: todos
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # redis
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: redis
      podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

**The configured rules include:**
- Ingress: Ingress from the frontend microservice.
- Egress: Egress to the redis microservice (6379) and DNS resolution is used.

### Redis policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow-ingress-from-logs-and-todos
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: logs
      podSelector:
        matchLabels:
          app: logs
    ports:
    - protocol: TCP
      port: 6379
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: todos
      podSelector:
        matchLabels:
          app: todos
    ports:
    - protocol: TCP
      port: 6379
  egress: []
```

**The configured rules include:**
- Ingress: Ingress from the logs and TODOs microservices.
- Egress: No egress allowed.

### Logs policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: logs-egress-redis
  namespace: logs
spec:
  podSelector:
    matchLabels:
      app: logs
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress:
  # redis
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: redis
      podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

**The configured rules include:**
- Ingress: No ingress allowed.
- Egress: Egress to the redis microservice (6379) and DNS resolution is used.

## Step #5: Proving the aplication

In this case we are going to prove the network policies, hpa and the deployment strategies of every microservice.

## Proving network policies

### Front policies

For each microservice we are going to prove one positive and negative policy.

**Allowed access from internet**

We use minikube tunnel to access via localhost and the port expose in the frontend service.

<p align="center">
  <img width="1239" height="181" alt="image" src="https://github.com/user-attachments/assets/493de269-63ec-4fb4-b3d5-eb3121535f56" />
</p>

<p align="center">
  <img width="1915" height="529" alt="image" src="https://github.com/user-attachments/assets/f780151f-0e89-42df-9bad-578d682883da" />
</p>

We notice we can access to the frontend service from internet.

**Disallowed access to logs**

We are going to prove the access to logs-svc using the DNS. 

<p align="center">
  <img width="1375" height="91" alt="image" src="https://github.com/user-attachments/assets/c63ae819-2afa-4958-9826-3900a6400835" />
</p>

In this case we can notice that the access from frontend to logs is not possible.

### Auth-api policies

**Allowed access to users-api**

To prove this communication it is required to generate a JWT. The command used to get the JWT is the following.

<p align="center">
  <img width="1443" height="177" alt="image" src="https://github.com/user-attachments/assets/7b101d1e-d1d2-46b8-8a6c-655f6225f6d3" />
</p>

With this JWT, we prove the connectivity.

<p align="center">
<img width="1444" height="179" alt="image" src="https://github.com/user-attachments/assets/057f2577-f56c-400a-81ed-61418e9be3b4" />
</p>

We notice the answer from the users-api microservice suggesting there is connectivity between it and auth-api.

**Disallowed access to redis**

We are going to use the ``wget`` command to prove the access to redis.

<p align="center">
  <img width="1356" height="91" alt="image" src="https://github.com/user-attachments/assets/a6a3b55b-62cc-4a37-b134-6d2acdab40a5" />
</p>

We can see that the access from auth-api to frontend is not possible.

### Users-api policies

We prove the positive policy before (auth-api -> users-api).

**Disallowed access to frontend**

We are going to use the ``wget`` command to prove the access to frontend.

<p align="center">
  <img width="1429" height="109" alt="image" src="https://github.com/user-attachments/assets/1bfcf927-aabf-4e36-b08f-2b8491a5d9d3" />
</p>

We can see that the access to frontend from users-api is not possible.

### TODO's policies

**Allowed access to redis**

Redis does not how to response ``wget`` connections, we are going to prove the connectivity using the typical command to prove reddis connections.

````
printf '*1\r\n$4\r\nPING\r\n' | nc redis-svc.redis.svc.cluster.local 6379
````

<p align="center">
  <img width="1243" height="154" alt="image" src="https://github.com/user-attachments/assets/021591e0-a8d2-4fd8-8b97-246fe2dd2e23" />
</p>

In this case we can notice the typical redis answer ``PING - PONG``, that suggest there is connectivity. 

**Disallowed access to users-api**

We are going to use the wget command to prove the access to users-api.

<p align="center">
  <img width="1280" height="88" alt="image" src="https://github.com/user-attachments/assets/977a38ca-44b9-489b-a74e-eb046d144775" />
</p>

We can see that the access to users-api from TODO's is not possible.

### Redis policies

*We proved the positive rule with the communication with TODO's.*

*Because the actual version of redis does not have tools to prove connectivity with other service, we can affirm that the deny-all policy is correct.*

<p align="center">
  <img width="1257" height="158" alt="image" src="https://github.com/user-attachments/assets/2a7bf364-730c-435c-8713-f82ae9e6d1ac" />
</p>

### Logs policies

**Allowed access to redis**

As we mentioned before, redis does not how to handle http connections. We try the connectivity using the ``PING - PONG`` command.

<p align="center">
  <img width="1246" height="151" alt="image" src="https://github.com/user-attachments/assets/7e66a22b-e63c-4635-83fa-be0c31feecd8" />
</p>

Again we can notice the communication between logs and redis thanks to ``PING - PONG``.

**Disallowed access to user-api**

We are going to use the ``wget`` command to prove the access to auth-api.

<p align="center">
  <img width="1226" height="83" alt="image" src="https://github.com/user-attachments/assets/3f443e90-6416-42f6-8510-ebaea5b9e3ca" />
</p>

We can notice that the access from logs to user-api is not possible.

## Proving deployment strategies

### Canary strategy

As we mentioned before, we implement Canary strategy. Both stable and new version of the frontend will coexist.

<p align="center">
  <img width="888" height="134" alt="image" src="https://github.com/user-attachments/assets/67b9ac84-c20d-4220-86b2-913c19d9ff66" />
</p>

If we access to the frontend-svc, we could recieve answer from our **latest version** ``v1.1.0``  or our **stable version** ``v1.0.0``.

<p align="center">
  <img width="1919" height="526" alt="image" src="https://github.com/user-attachments/assets/84b39f7b-3f12-4d99-b915-356650f273dd" />
</p>

*latest v1.1.0*

<p align="center">
  <img width="1911" height="537" alt="image" src="https://github.com/user-attachments/assets/4ac873cc-4e6f-41f4-94a4-c131e8eb5e0e" />
</p>

*stable v1.0.0*

### Rolling update strategy

In the case of the rolling update strategy, we can prove it upgrading the version of our stable deployment to watch the inmediate update of the pods.

<p align="center">
  <img width="1368" height="407" alt="image" src="https://github.com/user-attachments/assets/8b76affa-8fc6-40f1-8f70-d90f59400b82" />
</p>

In this case were updated the ``front-stable-5699d7846b-clwds`` and ``front-stable-5699d7846b-wcxm8 `` pods.

And now every time we access to our frontend, we get.

<p align="center">
<img width="1919" height="520" alt="image" src="https://github.com/user-attachments/assets/0f4b9567-86a4-4725-be15-e8672a436639" />
</p>

## Proving HPA

To prove the Horizontal Pod Autoscaling we have to stress a deployment who has enable hpa.

<img width="989" height="149" alt="image" src="https://github.com/user-attachments/assets/37528015-98b4-4b97-9134-d144624bf315" />

In this case we are going to stress a pod in the logs deployment using the comand.

````
while true; do echo "scale test"; done
````
Which stress CPU cores.

After some minutes we can notice the increment of CPU usage and the increment of number of pods in the logs deployment.

<p align="center">
<img width="1047" height="216" alt="image" src="https://github.com/user-attachments/assets/49613bd6-4b7e-45d8-bbe2-1dcaf9a92455" />
</p>

## Step #6: Monitoring using grafana and prometheus

In this case, we are going to watch some metrics about the cluster using grafana and prometheues. The first step is installing them using helm.

````
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
````
If we want to see helm version.

````
helm version
````

We add grafana and prometheus repositories.

````
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
````

We also update them.

````
helm repo update
````

We are going to create a namespace to put grafana and prometheus services.

````
kubectl create namespace monitoring
````

We are going to install grafana and prometheus using helm.

````
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30090 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30091 \
  --set grafana.adminPassword=admin
````
In this case:

* We setted the namespace.
* We setted prometheus type of service as NodePort and we expose 30090 port.
* We setted grafana type of service as NodePort and we expose 30091 port.
* We setted grafana admin password as admin.

We can see the prometheus stack is correctly installed using.

````
kubectl get svc -n monitoring
````

<p align="center">
<img width="1119" height="242" alt="image" src="https://github.com/user-attachments/assets/18ab03f0-4e68-4315-9b1c-409293e28d56" />
</p>

### Accessing grafana

We can get the ip to access grafana using.

````
minikube service prometheus-grafana -n monitoring --url
````
<p align="center">
  <img width="1299" height="93" alt="image" src="https://github.com/user-attachments/assets/2e02ef9b-f39c-436e-a4dc-e15735b3fdee" />
</p>

<p align="center">
  <img width="1919" height="590" alt="image" src="https://github.com/user-attachments/assets/e8e7973f-377c-4999-b036-7c0ae8ee619a" />
</p>

We access to the interface using ``admin`` ``admin`` as we mentioned before.

<p align="center">
  <img width="1900" height="724" alt="image" src="https://github.com/user-attachments/assets/9114bf47-bcfd-451b-a60e-833e9514da8b" />
</p>  

### Accesing prometheus

In this case we can port-forward the service port to access via localhost.

````
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
````
<p align="center">
  <img width="1915" height="493" alt="image" src="https://github.com/user-attachments/assets/07d75622-3d45-43f2-bc6a-fd940113d82f" />
</p>

In prometheus we can create queries to see information about our cluster, for example cpu and memory utilization of each pod.

<p align="center">
  <img width="1879" height="745" alt="image" src="https://github.com/user-attachments/assets/bd46cb8b-cfe2-4da4-a10d-e8019ad7ffff" />
</p>

*Memory utilization*

<p align="center">
  <img width="1893" height="754" alt="image" src="https://github.com/user-attachments/assets/01b06e13-f5d0-457f-8144-9c33b7c73d48" />
</p>

*CPU utilization*

### Grafana Dashboards

We can see some dashboards about our cluster. For example, we can import some sophisticated graphs from the community.

<p align="center">
<img width="1000" height="524" alt="image" src="https://github.com/user-attachments/assets/8ff37e88-58d2-48b6-933d-be24f75d34e3" />
</p>

*Importation process*

<p align="center">
  <img width="1700" height="700" alt="image" src="https://github.com/user-attachments/assets/ed31a5ef-21ff-4134-97e8-78b109fc21fc" />
</p>

*Visualization*

We can also we can build our own dashboard using the prometheus queries we mentioned before.

<p align="center">
<img width="15002" height="602" alt="image" src="https://github.com/user-attachments/assets/954d39e3-b120-4528-a2db-4db7560ab2b9" />
</p>

*Dashboard configuration*

<p align="center">
  <img width="1438" height="671" alt="image" src="https://github.com/user-attachments/assets/b43dfb39-af45-4d95-9050-7072a165bc6e" />
</p>

*Visualization*

# References

* https://keepcoding.io/blog/que-es-el-despliegue-canary/
* https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/





