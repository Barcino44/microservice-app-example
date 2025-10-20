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

````
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

````
export GO111MODULE=on
go mod init github.com/bortizf/microservice-app-example/tree/master/auth-api
go mod tidy
go build
````

Los módulos de go fueron necesarios para construir el dockerfile de manera correcta.

````
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

````
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

````
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

````
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
