# Docker Workshop walkthrough

## Hands On 0 - UDOO Neo SSH Connection
#### UDOO Neo
How to connect the UDOO NEO to the laptop.<br>
Follow the [tutorial](https://github.com/alegeno92/docker-workshop/blob/master/SSH_connection.md)


## Hands On 1 - Check the environment 
#### UDOO Neo & Laptop
Check that the docker and docker-compose are installed both locally and on the board. <br>
``` bash 
docker --version
docker-compose --version
```


## Hands On 2 - Dockerizing the first application (python)
#### Laptop
1. [Develop] 
   - Clone the cloud connector app from Github. 
    ``` bash  
        git clone https://github.com/alegeno92/cloud-connector.git 
        cd cloud-connector
    ```
    - Register to Adafruit IO cloud
    - Generate new Auth Token
    - Fill the AIO key in the config.json file (in order to data online)
    - Set ``"mock_client": true`` to disable the MQTT Local Client
2. [Dockerfile] 
    - Go to the project directory
    - Create a backup of the existing Dockerfile
    - Create a new Dockerfile 
    -  fill it with:
        ``` Dockerfile 
        FROM python:3-alpine
        RUN pip install --no-cache-dir paho-mqtt adafruit-io 
        RUN mkdir /app 
        WORKDIR /app 
        COPY *.py /app/ 
        CMD ["python", "-u", "/app/main.py" , "/config/config.json"]
        ```
3. [Build]
   - Launch  
   ```bash
        docker build . -t cloud-connector
    ```
   - At the end check if everithing works

   ```bash 
        docker image ls
    ```

4. [Run]
   - ```bash 
        docker run -v `pwd`:/config --network=host cloud-connector 
     ```

## Hands On 3 - Dockerizing the more complex app (C++) 
#### Laptop
1. [Develop] 
   - Clone the cloud connector app from Github.
```bash 
    git clone https://github.com/alegeno92/agent.git 
    cd agent
```
1. [Dockerfile] 
    - Go to the project directory
    - Create a backup of the existing Dockerfile
    - Create a new Dockerfile 
    -  fill it with:
        ``` Dockerfile 
        FROM alpine:latest AS paho-c
        RUN apk add --no-cache cmake g++ make
        RUN apk add --no-cache git libressl-dev
        RUN git clone https://github.com/eclipse/paho.mqtt.c.git
        WORKDIR paho.mqtt.c
        RUN git checkout v1.3.1
        RUN cmake -Bbuild -H. -DPAHO_WITH_SSL=ON -DPAHO_ENABLE_TESTING=OFF
        RUN cmake --build build/ --target install
        
        FROM paho-c AS paho-cpp
        WORKDIR /
        RUN apk add --no-cache git libressl-dev
        RUN apk add --no-cache --upgrade bash
        RUN git clone https://github.com/eclipse/paho.mqtt.cpp.git
        WORKDIR paho.mqtt.cpp
        RUN cmake -Bbuild -H. -DPAHO_BUILD_DOCUMENTATION=FALSE -DPAHO_BUILD_SAMPLES=FALSE
        RUN cmake --build build/ --target install
        
        FROM paho-cpp AS build-container
        RUN apk add --no-cache zlib-dev
        COPY . /home/agent
        WORKDIR /home/agent
        RUN if [ \-d "./build" ] ; then rm -rvf build  && mkdir build ; else mkdir build ; fi
        WORKDIR build
        RUN cmake ..
        RUN cmake --build . --target agent --


        
        FROM alpine:latest
        RUN apk add --no-cache zlib libressl libstdc++
        COPY --from=build-container /paho.mqtt.c/build/src/* /usr/lib/
        COPY --from=build-container /paho.mqtt.cpp/build/src/* /usr/lib/
        COPY --from=build-container /home/agent/build/bin/agent /
        CMD ["/agent", "/config/config.json"]
        ```
2. [Build]
   - Launch 
   ```bash 
   docker build . -t agent
   ```
   - At the end check if everithing works
   ```bash 
   docker image ls
   ```

3. [Run]
  ```bash 
  docker run -v `pwd`:/config --network=host agent
  ```


## Hands On 4 - Docker BuildX 
#### Laptop
1. create a new builder
  ```bash
  docker buildx create --name arm_builder
  ```
2. bootstrap the builder 
```bash
docker buildx inspect --bootstrap
```
3. Use the builder 
```bash 
docker buildx use arm_builder
```

4. Build the telemetry agent using  Docker BuildX
```bash
docker buildx build -t alegeno92/server --platform linux/arm64,linux/arm/v7 --load .
```

## Hands On 5 - DockerHub
#### Laptop 
1. Create an account on DockerHub
2. Login to DockerHub via CLI 
```bash
docker login
```

## Hands On 6 - DockerHub & BuildX
#### Laptop 
1. Build the agent for different platforms and push the results on DockerHub 
```bash
docker buildx build -t alegeno92/agent --platform linux/amd64,linux/arm/v7 --push .
```


## Hands On 7 - Docker Compose 
#### Laptop
1. Clone the cloud connector app from Github.
```bash 
git clone https://github.com/alegeno92/compose.git 
cd compose
```
2. Create a backup copy 
```bash
      mv docker-compose.yaml docker-compose.yaml.bk
```
3. create a new Docker Compose file with:
   ```yaml
   version: '3'
    services:
        server:
            image: alegeno92/server:latest
            ports:
               - "8000:8000"
               - - "8086:8086"
            depends_on:
               - "mosquitto"
               - "people-counter"
            volumes:
               - ./configurations/server:/config/

        agent:
            image: alegeno92/agent:latest
            hostname: agent
            depends_on:
            - "mosquitto"
            volumes:
            - ./configurations/agent:/config/

        mosquitto:
            image: eclipse-mosquitto:latest
            hostname: mosquitto
            container_name: mosquitto
            expose:
            - "1883"
            - "9001"
            ports:
            - "1883:1883"
            volumes:
            - ./configurations/mosquitto:/mosquitto/config/

        cloud-connector:
            image: alegeno92/cloud-connector:latest
            hostname: cloudconnector
            depends_on:
            - "mosquitto"
            volumes:
            - ./configurations/cloud-connector/:/config
            environment:
            - PYTHONUNBUFFERED=1

        people-counter:
            image: alegeno92/people-counter:latest
            depends_on:
            - "mosquitto"
            expose:
            - "8084"
            ports:
            - "8084:8084"
            volumes:
            - ./configurations/people-counter/:/config
            environment:
            - PYTHONUNBUFFERED=1
   ```
4. compress the directory to copy it on the board
```bash 
cd ..
tar cvf compose.tgz compose/
```
#### UDOO Neo & Laptop
5. copy the directory on the board
```bash
    scp compose.tgz udooer@192.168.7.2:~
```
password: udooer
#### UDOO Neo
6. connect to the board via ssh
```bash 
    ssh udooer@192.168.7.2
```
password: udooer

7. extract the archive and enter
```bash 
tar xvf compose.tgz
cd compose
```
8. launch the application with docker compose
```bash
docker-compose -f "docker-compose.yml" up
```
#### Laptop
9. navigate to http://192.168.7.2:8000 to see the realtime dashboard

10. navigate to https://io.adafruit.com/ to see the data sent on the cloud.