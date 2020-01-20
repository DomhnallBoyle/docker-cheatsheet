# docker-cheatsheet

- Contains commands for building images and running containers on one or more host machines
- Also contains some pitfalls/findings when running the various docker applications
- Contains information on networking with Docker containers

## Nvidia Docker
- Nvidia Docker is a plugin to Docker to allow containers to access the GPU on the host machine. 
- To setup, just follow the commands located [here](https://github.com/NVIDIA/nvidia-docker).
- Before attempting to build images/containers, you should edit the docker daemon to include nvidia as the 
default runtime. This means that the docker daemon will get access to the GPU when building images. 
    - ```sudo vi /etc/docker/daemon.json```
    - Add ```"default-runtime": "nvidia"```
- You should also ensure that CUDA based images have versions compatible with the host machines nvidia driver. 
Run ```nvidia-smi``` to check the driver on the host machine and you can check CUDA compatibility [here](https://tech.amikelive.com/node-930/cuda-compatibility-of-nvidia-display-gpu-drivers/). 
- Some pre-built images with CUDA installed are available [here](https://hub.docker.com/r/nvidia/cuda/). 
- If you plan on using Tensorflow as a backend in the container, make sure to check the compatibility versions
between Tensorflow and CUDA [here](https://www.tensorflow.org/install/source#tested_build_configurations). 

## Docker

- Some useful commands
```
# bash into a running container
docker exec -it <container_id> /bin/bash

# stop a running container
docker stop <container_id>

# removes all stopped containers
docker container prune

# removes all images that aren't associated with a container
docker image prune -a

# builds an image from a Dockerfile located in the same directory
docker build --build-arg ARG=?? . 

# saves an image as a tar file 
docker save -o <path for generated tar file> <image name>

# loads an image into docker from the saved tar file
docker load -i <path to image tar file>

# stop and remove container in the one command
docker rm -f <container id>

# create a new overlay network - making it attachable means any
# container can join the network
docker network create -d overlay --attachable <network name> 

# create a container and start it in the background from a built image
# mount a shared filesystem between host and docker container
# set up open ports between host and container
# attach the container to an pre-created network
# uses nvidia-runtime if not already defined in the daemon
docker run -d --runtime=nvidia -v /shared:/shared -p 5000:5000 --network <network name> -it <image id> 

# remove all containers (includes running)
docker rm -f $(docker ps -a -q)

# remove all docker images
docker rmi $(docker images -q)

# remove dangling images
docker rmi $(docker images --filter "dangling=true" -q --no-trunc)

# bash into stopped/exited container
docker commit <container name> <image name> # commit container changes to new image
docker run -v ... -it <image name> bash
```

## Docker Compose
- Allows you create multi-container applications on a single host
- A ```docker-compose.yml``` must be created. An example is given below: 
```yaml
version: "3"
services: 
  service_1:
    build: 
      context: /path/to/docker/file/for/service/1
      args: 
        - 'CUDA_VERSION=9.0'
    ports: 
    - '5000:5000'
    runtime: nvidia # not required if changed default runtime in docker daemon config
    volumes: 
    - /shared:/shared
  service_2:
    build: 
      context: /path/to/docker/file/for/service/2
      args: 
        - 'CUDA_VERSION=9.0'
    ports: 
    - '5000:5000'
    runtime: nvidia # not required if changed default runtime in docker daemon config
    volumes: 
    - /shared:/shared
```
- A range of docker-compose reference file options can be viewed [here](https://docs.docker.com/compose/compose-file/).
- Some useful commands: 
```
# just build the images from the yaml
docker-compose build

# build the images and run containers with them
docker-compose up -d

# remove the created containers from the yaml
docker-compose down

# remove the containers as well as their associated images
docker-compose down --rmi 'all'

# if you have a local/remote registry, you can push the built images using
docker-compose commit
docker-compose push 

# pulling images from the local registry by hosts connected to it
docker-compose pull
```

## Docker Swarm
- Allows you to create multi-container applications across multiple hosts
- Uses the same configuration as the docker-compose.yml with some additions
- The pipeline to creating a multi-container app across multiple hosts is as follows: 
    - Assumes 1 master and 1 worker host
    - Ensure the nodes have static IPs on the same network and can ping eachother
    - Ensure SSH enabled on each host machine
    - Ensure the following ports are opened: 
        - TCP port 2377
        - TCP and UDP port 7946
        - UDP port 4789
```
# enable SSH on port 22
sudo apt-get install openssh-server

# enable firewall and add ports
sudo ufw enable
sudo ufw allow 2377/tcp && ufw allow 7946/tcp && ufw allow 7946/udp && ufw allow 4789/udp
sudo ufw status verbose # you should see the ports with ALLOW IN from ANYWHERE
```
- See [here](https://docs.docker.com/engine/swarm/swarm-tutorial/) for other pre-requisites
- Initial setup example: 
```
# IP address of the manager/master
export MANAGER_IP=169.254.100.6

# initialise the swarm - create a manager node with an IP address
docker swarm init --advertise-addr $MANAGER_IP

# on the other machines (workers), run the following to add them to the swarm
# the TOKEN is given to you when you create the manager
docker swarm join --token TOKEN $MANAGER_IP:2377

# on the master node, check machines are added to the swarm
docker node ls 
```

Docker swarm does not allow you to build images, images must be pre-built and pushed to the registry (either local or docker hub registries)
- The docker-compose.yml can be used both by docker-compose and docker stack but they use/ignore different options in the file
- An example docker-compose.yml for deploying containers to different hosts is as follows: 
```yaml
version: '3.7'
services:
  service_1:
    image: 169.254.100.6:5000/openpose # url of local image registry
    build:
      args:
        - 'CUDA_BASE_VERSION=${CUDA_BASE_VERSION}' # argument for Dockerfile from shell environment
      context: /path/to/dockerfile
    deploy:
      placement:
        constraints: [node.role == manager] # type of node to run container on
    ports:
      - '5000:5000'
    volumes:
      - /shared:/shared
    networks:
      - core # attach to this network by name
  service_2:
    image: 169.254.100.6:5000/pgn # url of local image registry
    build:
      args:
        - 'CUDA_BASE_VERSION=${CUDA_BASE_VERSION}' # argument for Dockerfile from shell environment
      context: /path/to/dockerfile
    deploy:
      placement:
        constraints: [node.role == worker] # type of node to run container on
    ports:
      - '5000:5000'
    volumes:
      - /shared:/shared
    networks:
      - core # attach to this network by name
networks:
  core:
    external: true # attachable, external network - a container not in the swarm can attach
```

- Docker swarm is only in beta and requires you to add config to the daemon 
- ```sudo vi /etc/docker/daemon.json``` and add to every host machine: 
```
"insecure-registries": "169.254.100.6:100:5000", # accepts http access
"experimental": true
``` 
- Run ```sudo service docker restart``` on every host machine
- Below are some useful commands. See [here](https://docs.docker.com/engine/swarm/stack-deploy/) for reference:
```
# ensure machines in swarm have the directories for mounting e.g. /shared
# run the following command on each host
sudo mkdir /shared

# create a local registry for docker images between host machines (rather than docker hub) 
# creates registry at $MANAGER_IP:5000
docker service create --name registry --publish published=5000,target=5000 --constraint 'node.role == manager' registry:2

# export required environment variables
export CUDA_VERSION=9.0

# build the images from the docker-compose.yml
docker-compose build

# push built images to registry
docker-compose push 

# create an attachable overlay network to allow containers outside of the swarm to attach
docker network create -d overlay --attachable core

# deploy the built images using the host options in the docker-compose.yml
docker stack deploy -c docker-compose.yml <stack name>

# show services running on stack 
docker stack services <stack name>

# remove a deployed stack
docker stack rm <stack name>

# show state of containers running on nodes of the swarm (useful for debugging)
docker node ps $(docker node ls -q)
```
