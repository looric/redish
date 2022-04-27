# redish v10 install on Ubuntu 20.04.4 LTS

1、update os and docker
```
sudo apt update
sudo apt -yy install apt-transport-https ca-certificates curl software-properties-common wget pwgen
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update && sudo apt -y install docker-ce docker-ce-cli containerd.io

curl -s https://api.github.com/repos/docker/compose/releases/latest \
  | grep browser_download_url \
  | grep docker-compose-linux-x86_64 \
  | cut -d '"' -f 4 \
  | wget -qi -

chmod +x docker-compose-linux-x86_64
sudo mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
```

2、prepare redash env and docker-compose.yml
```
REDASH_BASE_PATH=/opt/redash
sudo mkdir -p $REDASH_BASE_PATH
sudo chown $USER:$USER $REDASH_BASE_PATH

sudo mkdir $REDASH_BASE_PATH/postgres-data
rm $REDASH_BASE_PATH/env 2>/dev/null
touch $REDASH_BASE_PATH/env

COOKIE_SECRET=$(pwgen -1s 64)
POSTGRES_PASSWORD=$(pwgen -1s 64)
REDASH_DATABASE_URL="postgresql://postgres:${POSTGRES_PASSWORD}@postgres/postgres"
echo "PYTHONUNBUFFERED=0" >> $REDASH_BASE_PATH/env
echo "REDASH_DISABLE_PUBLIC_URLS=true" >> $REDASH_BASE_PATH/env
echo "REDASH_LOG_LEVEL=INFO" >> $REDASH_BASE_PATH/env
echo "REDASH_REDIS_URL=redis://redis:6379/0" >> $REDASH_BASE_PATH/env
echo "POSTGRES_PASSWORD=$POSTGRES_PASSWORD" >> $REDASH_BASE_PATH/env
echo "REDASH_COOKIE_SECRET=$COOKIE_SECRET" >> $REDASH_BASE_PATH/env
echo "REDASH_SECRET_KEY=$COOKIE_SECRET" >> $REDASH_BASE_PATH/env
echo "REDASH_DATABASE_URL=$REDASH_DATABASE_URL" >> $REDASH_BASE_PATH/env

echo "export COMPOSE_PROJECT_NAME=redash" >> ~/.profile
echo "export COMPOSE_FILE=/opt/redash/docker-compose.yml" >> ~/.profile
source ~/.profile

chmod 640 /opt/redash/docker-compose.yml
```

3、create db and run
```
docker-compose run --rm server create_db
docker-compose up -d
```

4、create service
```
vi /etc/systemd/system/docker-compose-redash.service

[Unit]
Description=Docker Compose Application Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/redash
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable docker-compose-redash
systemctl stop docker-compose-redash
systemctl start docker-compose-redash
```

5、some useful command
```
docker container ls
docker container ls -a
docker container ls -f "status=exited"
docker container ls -q

docker version   – Echoes Client’s and Server’s Version of Docker
docker images   – List all Docker images
docker build <image>   – Builds an image form a Docker file
docker save <path> <image>   – Saves Docker image to .tar file specified by path
docker run   – Runs a command in a new container.
docker start   – Starts one or more stopped containers
docker stop <container_id>   – Stops container
docker rmi <image>   – Removes Docker image
docker rm <container_id>   – Removes Container
docker pull   – Pulls an image or a repository from a registry
docker push   – Pushes an image or a repository to a registry
docker export   – Exports a container’s filesystem as a tar archive
docker exec   – Runs a command in a run-time container
docker ps   – Show running containers
docker ps -a   – Show all containers
docker ps -l   – Show latest created container
docker search   – Searches the Docker Hub for images
docker attach   – Attaches to a running container
docker commit   – Creates a new image from a container’s changes

docker inspect redash-server-1 | grep -A50 "Env"
docker inspect -f "{{ .Config.Env }}" redash-server-1
```

6、content of docker-compose.yml
```
version: "2"
x-redash-service: &redash-service
 image: redash/redash:10.1.0.b50633
 depends_on:
    - postgres
    - redis
 env_file: /opt/redash/env
 restart: always
services:
 server:
    <<: *redash-service
    command: server
    ports:
     - "5000:5000"
    environment:
     REDASH_WEB_WORKERS: 4
 scheduler:
    <<: *redash-service
    command: scheduler
 scheduled_worker:
    <<: *redash-service
    command: worker
    environment:
     QUEUES: "periodic,emails,default"
     WORKERS_COUNT: 1
 adhoc_worker:
    <<: *redash-service
    command: worker
    environment:
     QUEUES: "queries,scheduled_queries,schemas"
     WORKERS_COUNT: 2
 redis:
    image: redis:6.2-alpine
    restart: always
 postgres:
    image: postgres:14.2-alpine
    env_file: /opt/redash/env
    volumes:
     - /opt/redash/postgres-data:/var/lib/postgresql/data
    restart: always
 ```
 
  7、thanks
```
 https://redash.io/
 https://computingforgeeks.com/install-redash-data-visualization-dashboard-on-ubuntu/
 https://hub.docker.com/_/postgres/
 ```
