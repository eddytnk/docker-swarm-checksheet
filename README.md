# Docker swarm commands

# Introduction 
In this Tutorial we are going to provide quick started docker swarm command and tutorials by working through a three nodes docker swarm cluster

## Setup Docker cluster
* Make sure you have three Servers (Linux VPS) running over your network or the Internet with each node having a dedicated IP address. This machines will be the nodes on the docker clusters
* I recommend you create an Amazon EC2 instances or a Digital Ocean Droplet
* Install Docker in each of the node (easy installing command `curl -sSL https://get.docker.com | sh`). This command will pull a docker installation script and run it on the node.
* In this tutorials, we will refer to the nodes as (Node-01, Node-02 and Node-03)
* On Node-01, run the command 
```shell script
docker swarm init --advertise-addr <Node-01 IPAddress>
```
This command will set up the docker swarm and enable Node-01 as the manager node on the swarm.
* The next step will be to add Node-02 and Node-03 as nodes on the swarm. 
    * To do this run this command on Manager Node, Node-01
```shell script
docker swarm join-token manager # For a manager node token
docker swarm join-token worker # For a worker node token
```
This command will display the docker swarm join token which is needed to add other nodes to the cluster or swarm as a manager node. It will look like this
```shell script
docker swarm join --token <swarm token> <Node-01 IP>:2377
```
Copy the join cluster command and run on the two other nodes to add them to the cluster.
This will create a three node cluster with Node-01 as the lead manager node. 
* Another way to upgrade a worker node (e.g Node-02) to a manage node, will be to join as a worker and then run this command on Node-01
```shell script
docker node update --role manager Node-02
```
Only manager nodes have access to the swarm commands, the worker nodes are just docker environments to execute tasks.

* To see that cluster created, run on the manager node
```shell script
docker node ls
```

## Creating a Service
* Service is the tasks to execute on the nodes. 
* When you create a service, you specify which container image to use and which commands to execute inside running containers.
```shell script
docker service create --name <ServiceName> -p <exposePort>:<serviceTaskPort> <image_name>:<image_tag>
```
* To create 3 replicas service use 
```shell script
docker service create --name <ServiceName> --replicas 3 -p <exposePort>:<serviceTaskPort> <image_name>:<image_tag>
```
* To see all the services created use
```shell script
docker service ls
```

## Network

**Overlay network**
 
Creates an internal private network that spans across all the nodes participating in the swarm cluster. So, Overlay networks facilitate communication between a docker swarm service and a standalone container, or between two standalone containers on different Docker Daemons.

* Use this command to create an overlay network on your nodes

```shell script
docker network create --driver overlay <network_name>
```
* To see all network created do

```shell script
docker network ls
```

* To create a service in a network use

```shell script
docker service create --name <ServiceName> --network <network_name> -p <exposePort>:<serviceTaskPort> <image_name>:<image_tag>
```

## Docker Stacks
Stacks accepts compose files as their declarative definition for services, network and volumes. `docker stack deploy`
* To deploy your application as a docker stack, define your application in a yml compose file. the run the command
```shell script
docker stack deploy -c <compose-file.yml> <stack_name>
```
* To see all running stacks use

```shell script
docker stack ls
```
* To list all task in the stack use
```shell script
docker stack ps <stack_name>
```
* To remove stack use
```shell script
docker stack rm <stack_name>
```
* To list all services in a stack you can use

```shell script
docker stack services <stack_name>
```

* Example of a stack compose file

```yaml

version: "3.8"
services:
  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]

```
    -   The Example will create a service call db using the image postgres:9.4 and run on a network call `backend`. 
    -   The constraint property on the deploy tell the swarn that This service can only be deployed on a node with a manager role

## Docker Secret

* Only stored on Disk on Manager nodes
* Secret are assign to a service
* They looks like files in container but are actually in-memory
    * `/run/secrets/<secret_name>` or `/run/secrets/<secret_alias>`
To create a secret
```shell script
docker secret create <secret_name> <secret.txt> # using a file
```    
OR

```shell script
docker secret create <secret_name> - # Then type the screat in std input
```
* Available commands

```
  create      Create a secret from a file or STDIN as content
  inspect     Display detailed information on one or more secrets
  ls          List secrets
  rm          Remove one or more secrets
```

* When a secret is create, you will never be able to see it value. you can only assign it to a service
* Ex: let's create psql service and pass it password
```shell script
docker service create --name psql --secret psql_user --secret psql_pwd -e POSTGRES_PASSWORD_FILE=/run/secrets/psql_pwd -e POSTGRES_USER_FILE=/run/secrets/psql_user postgres:9.4
```

* Using a stack compose file: We can pre-create the secret or we can add the secrete in our compose file

1. Pre-create: This means that `psql_user` and `psql_pwd` would have already been created in the swarm using `docker secret create` 

`file: docker-compose.yml`

```yaml

version: "3.8"
services:
  psql:
    image: postgres:9.4
    secrets:
        - psql_user
        - psql_pwd
    environment:
        POSTGRES_USER_FILE=/run/secrets/psql_user
        POSTGRES_PASSWORD_FILE=/run/secrets/psql_pwd
    volumes:
      - psql-data:/var/lib/postgresql/data
  
secrets:
    psql_user:
        external: true
    psql_pwd:
        external: true

volumes:
    psql-data:
    
```

2. Creating the the secrete in the stack compose file on the fly
 
`file: docker-compose.yml`
 
```yaml

version: "3.8"
services:
  psql:
    image: postgres:9.4
    secrets:
        - psql_user
        - psql_pwd
    environment:
        POSTGRES_USER_FILE: /run/secrets/psql_user
        POSTGRES_PASSWORD_FILE: /run/secrets/psql_pwd
    volumes:
      - psql-data:/var/lib/postgresql/data

secret:
    psql_user:
        file: ./psql_user.txt
    psql_pwd:
        file: ./psql_pwd.txt
volumes:
    psql-data:

```

* Make sure the compose file and the `psql_user.txt` and `psql_pwd.txt` are in same directory when creating the stack
* with this format, if we remove our stack, it will also remove all the secrets created

## Docker healthchecks  
* Support in Dockerfile, Compose yml, docker run, Swarm service
* Docker engine will exec the command in the container 
    - e.g curl localhost
* It expect exit 0 (OK) or exit 1 (Error) we could also have false for error
* Three container state: starting, healthy, unhealthy    
* We can see the status of our container when we do 
    - `docker container ls` or   
    - `docker container inspect` : check last 5 healthcheck
* Docker run do not take action when the healthcheck is fails. Just needed for status check
* In swarm, Healthcheck is important!!!
    - Services will replace tasks if they fail healthcheck
    
* Ex: Healthcheck in Docker run
 ```shell script
  docker run \
    -- health-cmd "curl -f localhost/_cluster/health || false" \
    -- health-interval=5s \
    -- health-retries=3s
    -- health-timeout=2s
    -- health-start-period=15s \
  elasticsearch:2
```        

* Health check in a compose stack file

```yaml

version "3.8"
services:
    web:
      image: eddytnk/myapp:latest
      healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost"]
          interval: 1m30s
          timeout: 10s
          retries: 3
          start_period: 40s

```

