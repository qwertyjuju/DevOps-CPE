# TP1 Discover Docker

- [TP1 Discover Docker](#tp1-discover-docker)
  - [Database](#database)
    - [Basics](#basics)
    - [Init database](#init-database)
    - [Persist data](#persist-data)
  - [Backend API](#backend-api)
    - [Basics](#basics-1)
    - [Multistage build](#multistage-build)
  - [http server](#http-server)

## Database
### Basics

- Dockerfile to build a basic postgresql image with some default environment variables: 
```Dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
CMD ["postgres"]
```

- Image creation from the dockerfile with "pgsql" name:
```bash
sudo docker build . -t pgsl
```
- Creation of a new network named "app-network":
```bash
sudo docker network create app-network
```
- Creation of a new container with the name "db" from the "pgsql" image connected to the "app-network" network:
```bash
sudo docker run --name db --network app-network -d pgsql
```
- Creation of a new container  with the name "admindb" from the "adminer" image connected to the "app-network" network:
```bash
sudo docker run --name admindb --network app-network -p 8080:8080 -d adminer
```
with the option "-p 8080:8080", the adminer web interface is now accessible on the host on the port 8080.

- __Q: Why should we run the container with a flag -e to give the environment variables?__

It is better to give the environment variables with the -e option because they are attached to the container and not the image. It is better to define container specific variables like, for example,DB passwords and usernames.

### Init database

One way to initialize the database is to copy the sql scripts in the "/docker-entrypoint-initdb.d" directory of the image:
```Dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
COPY ./CreateScheme.sql /docker-entrypoint-initdb.d
COPY ./InsertData.sql /docker-entrypoint-initdb.d
CMD ["postgres"]
```

We then need to rebuild the image and recreate the containers.

### Persist data

- __Q: Why do we need a volume to be attached to our postgres container?__

volumes allows to attach a part of the containers data to the host. It is important to use volumes to persist data when we destroy a container. For example, if we create a postgres database container with a volume, and that we destroy this container, we can recreate a container with the same volume, so the data from the first database is used in the new database.


- To use a volume for our postgres database:

```bash
sudo docker run --name db --network app-network -v /srv/docker/tp1/vol/pgsql/data:/var/lib/postgresql/data -d pgsql
```
## Backend API

### Basics
- Basic Dockerfile that copies "Main.java" to the "/srv" directory and compiles it in the image.
```Dockerfile
FROM   amazoncorretto:8-alpine3.21
COPY ./backend/Basic/Main.java /srv
WORKDIR /srv
RUN javac Main.java
CMD ["java", "Main"]
```
- Build command:
```bash
sudo docker build . -f backend/Basic/Dockerfile -t java
```
- Docker container creation with the image "java":
```bash
sudo docker run java
```

### Multistage build
- __Q: Why do we need a multistage build? And explain each step of this dockerfile.__

A multistage build is needed in this case so that the building process is managed on the creation of the image. It is not needed to install all the dependencies on the host to do the build.

```Dockerfile
# Build
FROM maven:3.9.9-amazoncorretto-21 AS myapp-build 
# creaion of an environment variable to define the app home directory
ENV MYAPP_HOME=/opt/myapp
# change WORKDIR. All commands passed after this directive will be in the new workdir
WORKDIR $MYAPP_HOME
# Copy of build file and source code
COPY pom.xml .
COPY src ./src
# Run build
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:21
# creaion of an environment variable to define the app home directory
ENV MYAPP_HOME=/opt/myapp 
# change WORKDIR. All commands passed after this directive will be in the new workdir
WORKDIR $MYAPP_HOME
# copy of the build result from the previous image inside the run image
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# ENTRYPOINT sets the default executable for the container
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

- build image:
```bash
sudo docker build . -t simpleapi
```
It is important to note that the "myapp-build" image is only used for the build process. The docker build command will create only one image.

- Create container
```bash
sudo docker run -p 8081:8080 -d simpleapi
```
The application is now accessible on the host on the port 8081. In a web browser goint on localhost:8081 returns a json "Hello, world!".

__Student Simple API:__
- Get the new simpleapi projet:
```bash
git clone https://github.com/takima-training/simple-api-student.git
```
- Modifications in "simple-api/src/main/resources/application.yml":
```yaml
spring:
  datasource:
    url: jdbc:postgresql://db
    username: usr
    password: passwd
    driver-class-name: org.postgresql.Driver
```
- We reuse previous dockerfile to build the image, but under a new name:
```bash
sudo docker build . -t simpleapi-student
```

- Container creation:
```bash
sudo docker run -p 8081:8080 --name simpleapi_student --network app-network -d simpleapi-student
```

## http server

__Q: Why do we need a reverse proxy?__
A reverse proxy has several advantages. One of those is that it allows us to use one port on the host to redirect to several websites. Another advantage is a better 