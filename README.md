# Microservice App - PRFT Devops Training

This is the application you are going to use through the whole traninig. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although is a TODO application, it is interesting because the microservices that compose it are written in different programming language or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments. 

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.
![microservice-app-example](/arch-img/Microservices.png)

## Solution

In this oportunity, the work was deploying the apps using kubernetes, network policies, config-maps, secretes, deployments strategies and hpa.

The next step were followed.

## Step #1: Creating Dockerfiles

### Frontend Dockerfile:

````Dockerfile
FROM node:8.17.0-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

EXPOSE 8080

CMD ["npm", "start"]
````

En el Dockerfile.

- Se realiza el uso de la imagen de node.
- Se establece ``/app`` como directorio de trabajo.
- Se copian las dependencias de node.
- Se instalan las dependencias de node.
- Se copian los archivos necesarios para la aplicación.
- Se compila la aplicación.
- Se establece como punto de ejecución ``["npm", "start"]``

### Auth-Api Dockerfile:


Antes de crear el dockerfile, fue necesario inicializar un módulo de go. Dicho modulo fue iniciado con ayuda de.

````sh
export GO111MODULE=on
go mod init github.com/bortizf/microservice-app-example/tree/master/auth-api
go mod tidy
go build
````

Los módulos de go fueron necesarios para construir el dockerfile de manera correcta.

````Dockerfile
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

````

En el Dockerfile Se generan 2 etapas, una para construcción y otra para despliegue. 

Esto se hace para reducir el tamaño del contenedor. Al existir 2 imágenes docker se queda unicamente con aquella que realiza el uso de la imagen builder.

En la etapa de construcción:

- Se usa la imagen de go como builder.
- Se establece la dirección en la cual se realizará la copia de los archivos. ``/app``
- Se copian los modulos correspondientes.
- Se descargan las dependencias necesarias.
- Se compila el binario correspondiente.

En la etapa de despliegue.

- Se usa la imagen de alpine.
- Se añaden los certificados necesarios para alpine.
- Se establece la dirección de trabajo. ``/app``
- Se copia el ejecutable de la primera etapa (builder).
- Se establece como punto de ejecución ``["./auth-api"]``

### Users-Api Dockerfile:

````Dockerfile
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

````

En el Dockerfile Se generan 2 etapas, una para construcción y otra para despliegue. 

Esto se hace para reducir el tamaño del contenedor. Al existir 2 imágenes docker se queda unicamente con aquella que realiza el uso de la imagen builder.

En la etapa de construcción:

- Se establece java-8 como image builder.
- Se establece ``/app`` como directorio de trabajo.
- Se copian las dependencias de Maven y se le asignan permisos de ejecución.
- Se copia el código base.
- Se compila la aplicación.

En la etapa de despliegue:

- Se establece java-8 como imagen.
- Se establece ``/app`` como directorio de trabajo.
- Se copia el archivo ejecutable de la primera etapa.
- Se establece como punto de ejecución `` ["java", "-jar", "target/users-api-0.0.1-SNAPSHOT.jar"] ``

### Todos-Api Dockerfile:

````Dockerfile
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
````

En el Dockerfile Se generan 2 etapas, una para construcción y otra para despliegue. 

Esto se hace para reducir el tamaño del contenedor. Al existir 2 imágenes docker se queda unicamente con aquella que realiza el uso de la imagen builder.

En la etapa de construcción.

- Se realiza el uso de la imagen de node.
- Se establece ``/app`` como directorio de trabajo.
- Se copian las dependencias de node.
- Se instalan las dependencias de node.
- Se copian los archivos necesarios para la aplicación.
- Se compila la aplicación.

En la etapa de despliegue.

- Se realiza el uso de la imagen de node.
- Se copia el archivo ejecutable de la primera etapa.
- Se establece como punto de ejecución ``["npm", "start"]``

### Log-Proccessor Dockerfile

````Dockerfile
FROM python:3.6-alpine

WORKDIR /app

RUN apk add --no-cache build-base

COPY . .

RUN pip3 install -r requirements.txt

CMD ["python3", "-u", "main.py"]
````

En el Dockerfile:

- Se realiza uso de la imagen de python 3.6.
- Se establece como directorio de trabajo ``/app``.
- Se añaden herramientas necesarias para la compilación.
- Se copian los archivos necesarios de la app.
- Se instalan dependencias de python.
- Se establece como punto de ejecución ``["python3", "-u", "main.py"]``

## Step #2: Creating Config-maps and Secrets

After the Dockerfiles were created, the next step was create the config-maps and secrets for each microservice.

Not all the microservices require these because some of them do not need env varibles or do not have sensible informatio. 

### Frontend config-map.

**config-map**

````yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: front-config
  namespace: front
data:
  PORT: "8080"
  AUTH_API_ADDRESS: "http://auth-api-svc.auth-api.svc.cluster.local:8000"
  TODOS_API_ADDRESS: "http://todos-svc.todos.svc.cluster.local:8082"
````

En este config-map:

- Se define el namespace al cual hace parte (front).
- Se definen las variables de entorno necesarias para el front (El nombre de dominio del servicio de auth-api y de todos-api).

### Auth-api config-map y Auth-api secret

**config-map**

````yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-api-config
  namespace: auth-api
data:
   AUTH_API_PORT: "8000"
   USERS_API_ADDRESS: "http://users-api-svc.users-api.svc.cluster.local:8083"
````
En este config-map:

- Se define el namespace al cual hace parte (auth-api).
- Se definen las variables de entorno necesarias para el auth-api (El nombre de dominio del servicio de users-api y el puerto correspondiente).

**secret**

````yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-api-secret
  namespace: auth-api
type: Opaque
data:
  JWT_SECRET: UFJGVA==
````
En este secret:

- Se define el namespace al cual hace parte (auth-api).
- Se definen los secretos necesarios para auth-api. En este caso JWT_SECRET, que se encuentra codificado en base64.

### Users-api config-map y Users-api secret

**config-map**

````yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: users-api-config
  namespace: users-api
data:
  SERVER_PORT: "8083"
````
- Se define el namespace al cual hace parte (users-api).
- Se definen las variables de entorno necesarias para el users-api (El puerto del servidor de users-api).

**secret**

````yaml
apiVersion: v1
kind: Secret
metadata:
  name: users-api-secret
  namespace: users-api
type: Opaque
data:
  JWT_SECRET: UFJGVA==
````
- Se define el namespace al cual hace parte (users-api).
- Se definen los secretos necesarios para users-api. En este caso JWT_SECRET, que se encuentra codificado en base64.

### Todos config-map y Todos secret

**config-map**

````yaml
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
````
- Se define el namespace al cual hace parte (todos).
- Se definen las variable de entorno necesarias para todos-api:
    - El puerto del servidor de Todos.
    - El nombre de host de redis.
    - El puerto de redis.
    - El canal de redis, el cual será usado para enviar mensajes a esta base de datos redis.

**secret**

````yaml
apiVersion: v1
kind: Secret
metadata:
  name: todos-secret
  namespace: todos
type: Opaque
data:
  JWT_SECRET: UFJGVA==
````
- Se define el namespace al cual hace parte (todos).
- Se definen los secretos necesarios para todos. En este caso JWT_SECRET, que se encuentra codificado en base64.

### Log-procesor config-map

**config-map**

````yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logs-config
  namespace: logs
data:          
   REDIS_HOST: "redis-svc.redis.svc.cluster.local"
   REDIS_PORT: "6379"
   REDIS_CHANNEL: "log_channel"
````
- Se define el namespace al cual hace parte (logs).
- Se definen las variable de entorno necesarias para logs:
    - El nombre de host de redis.
    - El puerto de redis.
    - El canal de redis (el cual será usado para traer los mensajes que llegan a esta base de datos).

## Step #3: Creating deployments, services and hpa

The next step was creating the deployments required to build the application, the services to access to each microservice and the HPA to manage to the traffic.

### Frontend deployment, service and hpa

We are going to create two frontends. The first is going to manage the stable version of our aplication and the second one will manage the latest version of our application.

Two deployments strategies will be used in the frontend. **The Canary** and **Rolling update strategy**.

**Definition Canary deployment strategy**

- Canary deployment works by gradually delivering traffic in production between two specific versions, starting with small percentages, such as 10/90%.

**Definition Rolling update strategy**

- Rolling updates allow deployments update to take place with zero downtime by incrementally updating Pods instances with new ones.

*In our case, we are going to create two replicas (with a stable image) in the stable deployment, and one replica (with the newest image) in the canary deployment. The service will handle the traffic and both new and stable versión will coexist.*

*We could decide to increase the version of our stable deployment and see how the rolling update strategy upgrades every pod (We are going to watch that later)*

**Stable deployment, service and hpa**

````yaml
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

````
**Canary deployment, service and hpa**

````yaml
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
````
For both canary and stable the structure is very similar.

**In the deployment**

- It was selected the namespace (front)
- It was selected the deployment strategy rolling update.
- It was setted that the deplpyment will manage the labels with the name of front.
- It was created two/one replica(s) (pods) with the image ``barcino/frontend:1.0.x`` builded with the Dockerfile.
- It was defined 8080 as the port of the container.
- It was asigned CPU resources to each pod.
- It was added the config maps created previously.

**In the service**

- It was selected the namespace (front).
- It was selected the pods of the labels of front.
- It was setted the port (Container port) and the target port (Application port).
- It was setted the service type as ``loadBalancer``.

**In the HPA**

- It was selected the namespace (front).
- It was setted the minimum (1/2) and maximum (3/5) number of replicas.
- It was setted the metrics to autoscal.

### Auth-api deployment, service and hpa

As in the frontend, the structure is very similar.

````yaml
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
````

**In the deployment**

- It was selected the namespace (auth-api).
- It was selected the deployment strategy rolling update.
- It was setted that the deployment will manage the labels with the name of auth-api.
- It was created two replicas (pods) with the image ``barcino/auth-api:1.0.0`` builded with the Dockerfile.
- It was defined 8000 as the port of the container.
- It was asigned CPU resources to each pod.
- It was added the config maps and secrets created previously.

**In the service**

- It was selected the namespace (auth-api).
- It was selected the pods of the labels of auth-api.
- It was setted the port (Container port) and the target port (Application port).
- It was setted the service type as ``ClusterIP``.

**In the HPA**

- It was selected the namespace (auth-api).
- It was setted the minimum (2) and maximum (5) number of replicas.
- It was setted the metrics to autoscal.

### Users-api deployment, service and hpa

Again, the structure is very similar.

````yaml
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
````
**In the deployment**

- It was selected the namespace (users-api).
- It was selected the deployment strategy rolling update.
- It was setted that the deployment will manage the labels with the name of users-api.
- It was created two replicas (pods) with the image ``barcino/users-api:1.0.0`` builded with the Dockerfile.
- It was defined 8083 as the port of the container.
- It was asigned CPU resources to each pod.
- It was added the config maps and secrets created previously.

**In the service**

- It was selected the namespace (users-api).
- It was selected the pods of the labels of users-api.
- It was setted the port (Container port) and the target port (Application port).
- It was setted the service type as ``ClusterIP``.

**In the HPA**

- It was selected the namespace (users-api).
- It was setted the minimum (2) and maximum (5) number of replicas.
- It was setted the metrics to autoscal.

### Users-api deployment, service and hpa

````yaml
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
  replicas: 2
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
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-todos
  namespace: todos
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todos
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
````

**In the deployment**

- It was selected the namespace (todos).
- It was selected the deployment strategy rolling update.
- It was setted that the deployment will manage the labels with the name of todos.
- It was created two replicas (pods) with the image ``barcino/todos-api:1.0.0`` builded with the Dockerfile.
- It was defined 8082 as the port of the container.
- It was asigned CPU resources to each pod.
- It was added the config maps and secrets created previously.

**In the service**

- It was selected the namespace (todos).
- It was selected the pods of the labels of todos.
- It was setted the port (Container port) and the target port (Application port).
- It was setted the service type as ``ClusterIP``.

**In the HPA**

- It was selected the namespace (todos).
- It was setted the minimum (2) and maximum (5) number of replicas.
- It was setted the metrics to autoscal.

### Redis deployment, service and hpa

````
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
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-redis
  namespace: redis
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: redis
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
````

**In the deployment**

- It was selected the namespace (redis).
- It was selected the deployment strategy rolling update.
- It was setted that the deployment will manage the labels with the name of redis.
- It was created two replicas (pods) with the image ``redis:7.4`` builded with the Dockerfile.
- It was defined 6379 as the port of the container.
- It was asigned CPU resources to each pod.
- It was added the config maps created previously.

**In the service**

- It was selected the namespace (redis).
- It was selected the pods of the labels of redis.
- It was setted the port (Container port) and the target port (Application port).
- It was setted the service type as ``ClusterIP``.

**In the HPA**

- It was selected the namespace (redis).
- It was setted the minimum (1) and maximum (3) number of replicas.
- It was setted the metrics to autoscal.

````
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
````

**In the deployment**

- It was selected the namespace (logs).
- It was selected the deployment strategy rolling update.
- It was setted that the deployment will manage the labels with the name of logs.
- It was created two replicas (pods) with the image ``barcino/logs:1.0.0`` builded with the Dockerfile.
- It was defined 4000 as the port of the container.
- It was asigned CPU resources to each pod.
- It was added the config maps created previously.

**In the service**

- It was selected the namespace (logs).
- It was selected the pods of the labels of logs.
- It was setted the port (Container port) and the target port (Application port).
- It was setted the service type as ``ClusterIP``.

**In the HPA**

- It was selected the namespace (logs).
- It was setted the minimum (1) and maximum (3) number of replicas.
- It was setted the metrics to autoscal.
