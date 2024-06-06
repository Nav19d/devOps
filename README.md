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
   docker push name-of-image USERNAME/image
```



## Part 2 : Github Actions

We will use GitHub Actions to push my code to SonarCloud, ensuring continuous integration and code quality checks. For enhanced security, we will use GitHub secrets to manage sensitive information.

### Test Containers 
GitHub Test Containers are a set of libraries that help you run integration tests using Docker containers. They ensure your tests run in an environment similar to production, leading to more reliable results. Key benefits include reusable containers, consistent environments, and isolated tests. 

### Github Workflow

```
name: CI devops 2024
on:
  push:
    branches:
      - master
      - develop
  pull_request:
jobs:
  build-and-test-backend: 
    runs-on: ubuntu-22.04
    steps:
      # Checkout your GitHub code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0
      # Set up JDK 17 using actions/setup-java@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      # Build and test your app with Maven
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=Nav19d_devOps -Dsonar.organization=nav19d -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./backend_API/simple-api-student/pom.xml
# define job to build and publish docker image

  build-and-push-docker-image:
    needs: build-and-test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PASSWORD }}
        

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./backend_API/simple-api-student
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/backend:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{secrets.DOCKERHUB_USERNAME}}/database:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./Http
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/httpd:latest
          push: ${{ github.ref == 'refs/heads/master' }}

```

### Workflow Triggers

- **Push Events**: Triggers on push to `master` or `develop` branches.
- **Pull Request Events**: Triggers on pull requests.

### Jobs

#### 1. Build and Test Backend

**Job Name**: `build-and-test-backend`  
**Runs on**: `ubuntu-22.04`

**Steps**:
1. **Checkout Code**: Uses `actions/checkout@v2.5.0` to check out the repository code.
2. **Set up JDK 17**: Uses `actions/setup-java@v3` to set up Java Development Kit version 17 with the Temurin distribution.
3. **Build and Test with Maven**: Runs Maven to build the project, run tests, and perform a SonarCloud analysis.

    ```yaml
    - name: Build and test with Maven
      run: mvn -B verify sonar:sonar -Dsonar.projectKey=Nav19d_devOps -Dsonar.organization=nav19d -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./backend_API/simple-api-student/pom.xml
    ```

#### 2. Build and Push Docker Image

**Job Name**: `build-and-push-docker-image`  
**Needs**: `build-and-test-backend` (Runs only if the backend build and tests pass)  
**Runs on**: `ubuntu-22.04`

**Steps**:
1. **Checkout Code**: Uses `actions/checkout@v2.5.0` to check out the repository code.
2. **Login to DockerHub**: Logs in to DockerHub using credentials stored in GitHub secrets.

    ```yaml
    - name: Login to DockerHub
      run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PASSWORD }}
    ```

3. **Build and Push Docker Images**: Uses `docker/build-push-action@v3` to build and push Docker images.

    - **Backend Image**:
      - **Context**: `./backend_API/simple-api-student`
      - **Tag**: `${{secrets.DOCKERHUB_USERNAME}}/backend:latest`
      - **Push Condition**: Only pushes if the branch is `master`.

        ```yaml
        - name: Build image and push backend
          uses: docker/build-push-action@v3
          with:
            context: ./backend_API/simple-api-student
            tags: ${{secrets.DOCKERHUB_USERNAME}}/backend:latest
            push: ${{ github.ref == 'refs/heads/master' }}
        ```

    - **Database Image**:
      - **Context**: `./database`
      - **Tag**: `${{secrets.DOCKERHUB_USERNAME}}/database:latest`
      - **Push Condition**: Only pushes if the branch is `master`.

        ```yaml
        - name: Build image and push database
          uses: docker/build-push-action@v3
          with:
            context: ./database
            tags: ${{secrets.DOCKERHUB_USERNAME}}/database:latest
            push: ${{ github.ref == 'refs/heads/master' }}
        ```

    - **HTTPD Image**:
      - **Context**: `./Http`
      - **Tag**: `${{secrets.DOCKERHUB_USERNAME}}/httpd:latest`
      - **Push Condition**: Only pushes if the branch is `master`.

        ```yaml
        - name: Build image and push httpd
          uses: docker/build-push-action@v3
          with:
            context: ./Http
            tags: ${{secrets.DOCKERHUB_USERNAME}}/httpd:latest
            push: ${{ github.ref == 'refs/heads/master' }}
        ```

### Summary

This GitHub Actions pipeline automates the process of building, testing, and deploying the application. It ensures consistency and reliability by leveraging Docker for deployment and SonarCloud for code quality analysis.


## TP3 : Ansible

For our project, we will use Ansible to install and deploy our application automatically. We will be able to access our web app not only locally

Ansible inventories define the hosts and groups of hosts upon which commands, modules, and playbooks can be executed. By default, the inventory is stored in /etc/ansible/hosts. For our project, we create a custom inventory file to define our production environment.

```
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ~/ssh_keys/id_rsa
 children:
   prod:
     hosts: navid.bar-hen.takima.cloud
```

To test the connectivity to our host using the inventory, we can use the Ansible ping module:
```
ansible all -i inventories/setup.yml -m ping
```

Now we create a playbook to configurates our host and decide the order of the tasks
```
- hosts: all
  gather_facts: false
  become: true
  roles:
  - docker
  - network
  - database
  - app
  - proxy
```

The playbook runs on all hosts specified in the Ansible inventory.
Fact collection is disabled (gather_facts: false) to optimize performance, as facts are not needed in this case.
Privilege (become: true) are granted to allow execution of commands with superuser privileges.
The Docker role is included (- docker) in the list of roles to apply. This triggers the execution of the docker/tasks/main.yml job file in the Docker role on each target host.
In the Docker role, environment variables are loaded from the . env, the required packages are installed, Docker is configured and the Docker service is started.

We create our roles :

```
ansible-galaxy init roles/docker
```


our tasks : 

```
- name: Run backend
  docker_container:
    name: my_java
    image: nav19d/backend:latest
    state: started
    networks: 
     - name: app-network 
```

```
- name: Run database
  docker_container:
    name: my_postgres_container
    image: nav19d/database:latest
    state: started
    networks: 
     - name: app-network
```

```
- name: Create a network
  docker_network:
    name: app-network
    state: present
```
```

# Install Docker

  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```
```
- name: Run HTTPD
  docker_container:
    name: my-running-app
    image: nav19d/httpd:latest
    ports:
    - "80:80" 
    networks: 
    - name: app-network
```

and now we can deploy our app accessible at navid.bar-hen.takima.cloud" with
```
ansible-playbook -i ansible/inventory/setup.yml ansible/playbook.yml
```
