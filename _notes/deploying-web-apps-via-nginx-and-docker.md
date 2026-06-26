---
layout: post  
name: Deploying Web Apps via Nginx and Docker with a Minimal Blueprint
birth: 2026-06-24
---

## Introduction

**nginx** is a **web server** that serves content directly to clients and also functions as a **reverse proxy**, accepting HTTP requests from clients and forwarding them to backend services before returning the responses. 

**Docker** packages applications in isolated **containers**. Multiple containers can run simultaneously on a single **host** within a private network, enabling separate containers—such as for the nginx server, frontend UI, and backend Python—to construct our web service.

This article demonstrates a minimal web service that stays alive and responds to incoming network requests. It covers Docker container and nginx server configuration, service deployment, and the traffic lifecycle to illustrate how these components work together.

---

## 0. Prerequisites

This guide assumes a fresh Ubuntu 22.04 LTS (or later) environment with sudo privileges.

**Install Docker:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```

**Verify the installation:**
```bash
docker --version
docker-compose --version
docker run hello-world
```

The `hello-world` container confirms Docker is properly installed. nginx and the Python application will be containerized and automatically pulled when services are deployed, so there is no need to install or configure them on the host.

---

## 1. Configuration Setup

Two Docker containers will be built: one for the nginx proxy and the other for a minimal Python application. By placing these services under the same Docker Compose file (`docker-compose.yml`), they automatically join an isolated virtual network where container names serve as internal domain names.

```yaml
version: '3.8'

services:
  # A super simple web service running Python's built-in server
  web-app-service:
    image: python:3.12-slim
    container_name: simple-python-app
    restart: unless-stopped
    # Run a simple server that stays alive and listens on port 8080
    command: python -m http.server 8080
    # Port is hidden from the host for maximum security

  # Our nginx gateway container
  nginx-proxy:
    image: nginx:alpine
    container_name: nginx-gateway
    restart: unless-stopped
    ports:
      - "127.0.0.1:80:80" # Only nginx is exposed to the Host Loopback
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web-app-service

```

An nginx configuration (`nginx.conf`) file should be created to define the routing rules and map public URLs to our containerized services.


```nginx
server {
    listen 80;
    server_name localhost;

    client_max_body_size 50M;

    # Route for our Python web application
    location /my-app {
        # Forward traffic to the Python service container name and port
        proxy_pass http://web-app-service:8080/my-app;
        
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        return 404;
    }
}
```

This configuration defines how nginx manages web traffic. The `listen 80` directive monitors standard HTTP requests, while `location /my-app` targets our specific URL path prefix. Inside this block, `proxy_pass` links this path directly to our internal container's domain and port, and the proxy_set_header lines preserve the user's original connection details. Finally, the `return 404` block at the bottom serves as a catch-all security rule to reject any undefined paths.

---

## 2. Deployment


To spin up this entire multi-container service, we don't need to manually run `docker run` for each container. We just need to open the terminal in the directory containing `docker-compose.yml` and execute a single command:

```bash
docker compose up -d
```

* **`up`**: Tells Docker to build the images (if not built yet), create the private network, and start all services in the correct order based on `depends_on`.
* **`-d` (Detached mode)**: Runs the containers in the background, so they don’t block your terminal.

If we ever need to stop and remove all these containers and the private network, we can simply run:

```bash
docker compose down
```

---

## 3. Deep Dive into the Traffic Lifecycle

Now that we know how to successfully keep our minimal web service alive through deployment, let’s observe how the network traffic flows through a distinct lifecycle to complete the request-response loop when a user accesses the application:

```
[ Browser ] 
   │
   │ (External Request: http://<HOST_IP>/my-app)
   ▼
[ nginx Reverse Proxy Container ] (Listens on Host Port 80 via Forwarding)
   │
   │ (Forwards Traffic via Private Network: http://web-app-service:8080/my-app)
   ▼
[ Python Web Service Container ] (Listens on internal 0.0.0.0:8080)
```

The nginx container sits at the very front of our server, acting as a traffic controller that accepts requests and routes them to our backend application:


The Request Phase:

1. The user navigates to `http://<HOST_IP>/my-app`.
2. The host machine intercepts the traffic on port `80` and maps it directly into our `nginx-proxy` container.
3. nginx reads the URL, matches it against `location /my-app`, and passes the payload over to `http://web-app-service:8080/my-app` inside the virtual network.
4. The Python container receives the request because it is listening on all local interfaces (`0.0.0.0:8080`).

The Response Phase:

1. The Python server processes the request and automatically generates a standard "Directory Listing" HTML page showing the container's internal folder structure.
2. It is important to note that our configuration files only explicitly declare the "ingress" route (the path heading inward). This is because `proxy_pass` establishes a two-way communication channel; the configuration only needs to declare the incoming route, and the underlying protocol automatically handles the return response back to nginx, which then serves it to the browser.

---

## Summary

This architecture establishes a secure network perimeter by isolation. Pairing nginx with Docker Compose forces all public ingress traffic through port 80 of the nginx gateway container, while the application container remains hidden inside the internal virtual network. 

For future maintenance and expansion, this architecture allows seamless tool integration. To add new components later, you only need to declare the new service inside `docker-compose.yml` and append a corresponding `location` block within `nginx.conf`. 

This multi-container blueprint ensures the foundational infrastructure is structurally ready for horizontal expansion.
