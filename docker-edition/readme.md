
# Docker Edition Training

Discuss concepts of how Docker "works" with containerization of environnments and communication with Docker or Kubernetes as an orchestrator:

    Docker, as well as Kubernetes and any container orchestration platform,
    uses cgroups to isolate process capabilities (cpu, ram, etc) and also namespaces
    to isolate these processes from each other, all within one external host. In effect,
    you can have multiple distinct groups of processes running on the same machine that
    are incapable of interacting with other groups. For the simple demonstration we are
    going to make a namespace using Docker as the orchestrator. Containers are "immutable"
    to an extent. The entrypoint scripts adjust them every time they launch as it pulls
    configuration from a central repository each time.

- Discuss size considerations for rapid deployments.

    - Further discussion about Alpine Linux

- Discuss startup and what purpose entrypoints serve as well as wait-for-it scripts

- Dicuss networking and potential conflicts related to proxies and other hurdles for communication

- Recycling of IP Adresses

- Discuss all the components in a Docker installation.

    - Consul (ConfigMap in Kubernetes)
    
    - oxAuth

    - oxTrust

    - Registrator (Docker specific)

    - config-init

    - OpenDJ

    - oxPassport

    - oxShibboleth

    - Nginx

    - key-rotation

- Cheat sheet of Docker commands.

```
# View all containers, including exited ones

docker container ls -a

# View all images currently downloaded on the system

docker images

# Remove a container

docker rm $container_name

# Access the container

docker exec -it $container_name sh

# Add a package

apk update && apk add $package

# Inspect a container

docker inspect $container_name

```

Give a demonstration on spinning up containers in Kubernetes. Time permitting I can go through how Kubernetes deployments work in regards to yaml files, etc.
