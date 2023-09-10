# S3 compatible Object Storage using MinIO

MinIO is an S3 compatible Object Storage. These notes cover setting up two MinIO servers and an Nginx reverse proxy on a Docker host to provide Object Storage.
For more information: https://github.com/minio/minio/tree/master/docs/orchestration/docker-compose

## Installing Docker

### Step 1: install Docker

To install Docker on a Linux distro, use the appropriete package manager. Fro Arch we will use Pacman

> 
> sudo pacman -S docker


### Start Docker service

Once istalled, the next step is to start the Docker daemon service using using *systemctl start*

> 
> sudo systemctl start docker.service
>
> docker -v
> Docker version 24.0.5, build ced0996600

If you want Docker to run at startup as a system service then use _`systemctl enable`_

> 
> sudo systemctl enable docker.service


### Add user to Docker user group

By default, Docker commands require root permissions. Running a command will reults in the followintg error

> docker search minio
> _Permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock:..._

The reason is the permissions, only root and the docker user-group members can access the Docker socket:

> ls -l /var/run/docker.sock
> srw-rw---- 1 root docker 0 Sep  9 13:53 /var/run/docker.sock

To run Docker as your user, you will need to add yourself to the docker group.

First, use _getent_ to check if the Docker user group exists on the host machine. 

>
> getent group docker
> docker:x:962:

Alternatively check the grpups you are in 

> groups $(whoami)
> wheel audio input lp storage users network power petemak


The above output shows the group exists. If not, we can add it using _`groupadd`_ command:

>
> sudo groupadd docker


>> #### Add yourself to the Docker user group
> 
> sudo usermod -aG docker $(whoami)

If you check the grpups you are in now, docker should be listed 

> groups $(whoami)
> wheel audio input lp storage users network power docker petemak


### Restsart Docker service

The above changes only take effect after next log in. The alternative is to apply the configuration changes by restarting the Docker daemon service using the _`systemctl restart`_ command:

> sudo systemctl restart docker.service


### Install docker-compose

Compose is for defining and running multi-container Docker applications. Application services are configurred in a YAML file.

>
> sudo pacman -S docker-compose
>
> docker-compose -v
> Docker Compose version 2.20.3


## Installing and Running  MinIO

### Service definition

We will use a Compose YAML to define how to start MinIO and Nginx. It specifies:
* Two MinIO containers minio1 and minio2 
* Both expose ports 9000 and 9001. Port 9001 will be the port, for the WebUI
* The secrets for the root admin to login later to the WebUI using the environment variables MINIO_ROOT_USER and  MINIO_ROOT_PASSWORD are in the .env file
* Bind mounts the local directory /docker/minio on the Docker host to the container under the /data directory
* The thrid service is Nginx x which serves as a reverse for Mino.  
* The Nginx config file defines two servers for port Minio and the cosole

```
version: '3'

services:
  minio1:
    container_name: Minio
    hostname: minio1
    image: quay.io/minio/minio:latest
    command: server /data  --console-address ":9001"
    restart: unless-stopped
    env_file:
      - .env
    expose:
      - '9000'
      - '9001'
    volumes:
      - /docker/minio1:/data

  minio2:
    container_name: minio2
    hostname: minio2
    image: quay.io/minio/minio:latest
    command: server /data --console-address ":9001"
    restart: unless-stopped
    env_file:
      - .env
    expose:
      - '9000'
      - '9001'
    volumes:
      - /docker/minio2:/data


  nginx:
    image: nginx:alpine
    hostname: nginx
    ports:
      - "9000:9000"
      - "9001:9001"
    depends_on:
      - minio1
      - minio2
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

### Run containers

Use `docker-cmpose` to start MinIO and Nginx

```
docker-compose up -d
[sudo] password for petemak: 
[+] Running 7/7
 ✔ minio 6 layers [⣿⣿⣿⣿⣿⣿]      0B/0B      Pulled
 ✔ Container Minio        Started
```


Open a browser and browse the port defined for the web UI: _http://localhost:9001_. That should opne the MinIO administraot UI. Sign in using the credentials deined in the YAML

![WebUI](~/Downloads/minio-ui2.png )
