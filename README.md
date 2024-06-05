# README

## Overview

This project involves creating a 3-tier web API using Docker containers. The application consists of three main components: an HTTP server, a Backend API, and a Database. Each component will be containerized and we will orchestrate these containers to work together seamlessly. The final goal is to have a fully functional web API running with Docker Compose.

## Goals

- Follow good practices for Docker containerization.
- Document each step of the process.
- Create an appropriate file structure with 1 folder per image.
- Successfully run a 3-tier web API.

## PART 1

### 1. Database

#### Base Image

We will use the image `postgres:14.1-alpine` to create our database and we will configure it by changing our Dockerfile for logins and creating our dataset (with COPY):

```Dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=Navid \
    POSTGRES_PASSWORD=1234

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d 
```

#### Commands

    1. Build the image
    ```docker build -t my_postres_image ```

    2. Create a network (to allow other images to interact):
    ```docker network create app-network ```

    3. Run the container:
    ```docker run --name my_postgres_container -v ./data:/var/lib/postgresql/data--network app-network -p 5432:5432 -d my_postgres_image```

in this line we specified the port, where we can access locally, the network and the volume to store the data of the image in case the container is killed



### 2. Backend API

#### Why do we need a multistage build ?

A multistage build allows us to separate the build environment from the runtime environment. This has several benefits:
    1.	Reduced Image Size: By only including the necessary runtime dependencies in the final image, we significantly reduce the size of the Docker image.
    2.	Security: The final image does not include the build tools and dependencies, reducing the attack surface.
    3.	Efficiency: We avoid including unnecessary files and dependencies in the final image, making it lean and optimized for running the application.


This is our backend config :

```FROM maven:3.8.6-amazoncorretto-17 AS build
WORKDIR /app
COPY pom.xml /app
RUN mvn dependency:go-offline
COPY src /app/src
RUN mvn package -DskipTests
FROM amazoncorretto:17
WORKDIR /app
COPY --from=build /app/target/*.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

##### First Stage: Build Stage

1.	```FROM maven:3.8.6-amazoncorretto-17 AS build```:
o	This line specifies the base image for the build stage. The image includes Maven 3.8.6 and the Amazon Corretto 17 JDK. The AS build part names this stage "build" so it can be referenced later.
2.	```WORKDIR /app```:
o	This line sets the working directory inside the container to /app. All subsequent commands will be run in this directory.
3.	```COPY pom.xml /app```:
o	This line copies the pom.xml file from the host machine to the /app directory inside the container. This is done separately from the source code to take advantage of Docker's layer caching. If the pom.xml file doesn't change, this layer will be cached, making the build faster.
4.	```RUN mvn dependency:go-offline```:
o	This command tells Maven to download all dependencies and make them available offline. This ensures that the dependencies are cached and won't need to be downloaded again during subsequent builds, provided the pom.xml file hasn't changed.
5.	```COPY src /app/src```:
o	This line copies the source code from the src directory on the host machine to the /app/src directory inside the container.
6.	```RUN mvn package -DskipTests```:
o	This command runs Maven to build the project and package it into a JAR file, skipping the tests to save time. The output JAR file will be placed in the target directory inside the container.


##### Second Stage: Run Stage

7.	```FROM amazoncorretto:17```:
o	This line specifies the base image for the runtime stage. It includes the Amazon Corretto 17 JRE, which is sufficient to run the application but does not include the build tools (JDK and Maven).
8.	```WORKDIR /app```:
o	This line sets the working directory inside the container to /app. All subsequent commands will be run in this directory.
9.	```COPY --from=build /app/target/*.jar /app/app.jar```:
o	This line copies the JAR file generated in the build stage from the target directory of the build container to the app directory of the runtime container. The --from=build syntax specifies that the file should be copied from the build stage.
10.	```ENTRYPOINT ["java", "-jar", "app.jar"]```:
o	This line sets the entry point for the container. When the container starts, it will run the command java -jar app.jar, which starts the Spring Boot application.


We build a springboot image and run the container within the same network.
Now if we got to localhost:8080/departments/IRC/students we get :


```
[
  {
    "id": 1,
    "firstname": "Eli",
    "lastname": "Copter",
    "department": {
      "id": 1,
      "name": "IRC"
    }
  }
]
```

### 3. HTTP Server

#### Reverse Proxy

A reverse proxy server is an intermediate server that sits between client devices and backend servers, forwarding client requests to the appropriate backend server and returning the server's response to the client.
In the conf file, we add : 

```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://my_java:8080/
ProxyPassReverse / http://my_java:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

<VirtualHost *:80> : indicates that it is linked to all IP address on the port 80


### Docker Compose

The docker compose file allows us to create all the images with one file. We just have to configure the good port, dependency (the database for instance needs to be created first for the application to be functional) and the network to link them all. 
```
version: '3.7'
services:
    backend: 
        container_name: my_java
        build: ./backend_API/simple-api-student
        
        networks:
        - app-network

        depends_on:
        - database

    database: 
        container_name: my_postgres_container
        build: ./database
        
        networks:
        - app-network

    httpd:
        container_name: my-running-app
        build: ./Http  
        ports:
        - 80:80
        networks:
         - app-network
        depends_on:
        - backend
        - database

networks:
    app-network:
```

### Publish on Docker Hub

To publish our images, we can use docker hub, associated with our github account.
To do so just use the command :
```docker tag name-of-image USERNAME/image-name
   docker push name-of-image USERNAME/image```




