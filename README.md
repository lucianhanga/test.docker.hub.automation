# test.docker.hub.automation

Playground for testing for automation of building docker images and upload to git.hub registry.

## To get it started:

### requirements

- create 3 images for nginx with different landing pages
- create one reverse proxy image with nginx
- based on the location header of the request, the reverse proxy should forward the request to the correct nginx image
- create a docker-compose file to start the reverse proxy and the 3 nginx images
- run the docker-compose file on a local swarm cluster

### development & deployment & testing

#### create 3 images for nginx with different landing pages

create the folders for the 3 nginx images

```bash
mkdir -p nginx1 nginx2 nginx3
```
create the Dockerfile for the 3 nginx images

```bash
cat <<EOF > nginx1/Dockerfile
FROM nginx:1.17.10-alpine
COPY nginx1.html /usr/share/nginx/html/index.html
EOF
```
create the nginx1.html file

```bash
cat <<EOF > nginx1/nginx1.html
<html>
<head>
<title>nginx1</title>
</head>
<body>
<h1>nginx1</h1>
</body>
</html>
EOF
```

test the code by creating the image and runing it locally on port 8081 and see if the correct landing page is shown

```bash
docker image build -t nginx1 nginx1

# see if the image is created
docker image ls

# run the image locally on port 8081
docker container run \
    --detach \
    --name nginx1 \
    --publish 8081:80 \
    nginx1

# check if the container is running
docker container ls

# check if the landing page is shown
curl localhost:8081

# cleanup for nginx1
docker container stop nginx1
docker container rm nginx1
docker image rm nginx1
```

start the same container with health check

```bash
docker container run \
    --detach \
    --name nginx1 \
    --publish 8081:80 \
    --health-cmd="curl -f http://localhost:8081 || exit 1" \
    --health-interval=5s \
    --health-retries=3 \
    --health-timeout=2s \
    --health-start-period=5s \
    nginx1
```

picture of the running container:

![health](./README.assets/health.png)

repeate the same steps for nginx2 and nginx3 or just copy and modify the files

buld the other docker images:

```bash
docker image build -t nginx2 nginx2
docker image build -t nginx3 nginx3
```

test the other docker images:

```bash
docker container run \
    --detach \
    --name nginx2 \
    --publish 8082:80 \
    --health-cmd="curl -f http://localhost:8082 || exit 1" \
    --health-interval=5s \
    --health-retries=3 \
    --health-timeout=2s \
    --health-start-period=5s \
    nginx2

docker container run \
    --detach \
    --name nginx3 \
    --publish 8083:80 \
    --health-cmd="curl -f http://localhost:8083 || exit 1" \
    --health-interval=5s \
    --health-retries=3 \
    --health-timeout=2s \
    --health-start-period=5s \
    nginx3
```
cleanup the other docker images:

```bash
docker container stop nginx2
docker container rm nginx2
docker image rm nginx2

docker container stop nginx3
docker container rm nginx3
docker image rm nginx3
```
Create the docker compose file

```bash
cat <<EOF > docker-compose.yml
version: "3.7"
services:
  nginx1:
    image: nginx1
    build: nginx1
    ports:
      - "8081:80"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081"]
      interval: 5s
      retries: 3
      timeout: 2s
      start_period: 5s
  nginx2:
    image: nginx2
    build: nginx2
    ports:
      - "8082:80"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082"]
      interval: 5s
      retries: 3
      timeout: 2s
      start_period: 5s
  nginx3:
    image: nginx3
    build: nginx3
    ports:
      - "8083:80"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8083"]
      interval: 5s
      retries: 3
      timeout: 2s
      start_period: 5s
EOF
```
run the docker compose file

```bash
# start the docker compose file
docker-compose up -d

# check if the containers are running
docker container ls

# check if the landing pages are shown
curl localhost:8081
curl localhost:8082
curl localhost:8083
```
create the reverse proxy image

```bash
mkdir -p reverse-proxy
cat <<EOF > reverse-proxy/Dockerfile
FROM nginx:1.17.10-alpine
COPY nginx.conf /etc/nginx/nginx.conf
EOF
```
create the nginx.conf for the reverse proxy which will forward the requests to the correct nginx image based on the location header

```bash
cat <<EOF > reverse-proxy/nginx.conf
events {
    worker_connections 1024;
}
http {
    server {
        listen 80;
        server_name example.com;

        location /app1 {
            proxy_pass http://nginx1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /app2 {
            proxy_pass http://nginx2;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /app3 {
            proxy_pass http://nginx3;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location / {
            # stay here
            root /usr/share/nginx/html;
        }
    }
}
EOF
```
build the reverse proxy image and run it locally on port 8080

```bash
# build the reverse proxy image
docker image build -t reverse-proxy reverse-proxy
# run with health check the reverse proxy image locally on port 8080
docker container run \
    --detach \
    --name reverse-proxy \
    --publish 8080:80 \
    --health-cmd="curl -f http://localhost:8080 || exit 1" \
    --health-interval=5s \
    --health-retries=3 \
    --health-timeout=2s \
    --health-start-period=5s \
    reverse-proxy
```

get the logs from the reverse proxy container

```bash
docker container logs reverse-proxy
```

add the reverse proxy to the docker compose file

