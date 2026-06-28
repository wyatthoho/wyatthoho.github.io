---
layout: post  
name: Deploying Web Apps via Nginx and Docker with a Minimal Blueprint
birth: 2026-06-24
---

## Introduction

**Nginx** is a **web server** that serves content directly to clients and also functions as a **reverse proxy**, accepting HTTP requests from clients and forwarding them to backend services before returning the responses. 

**Docker** packages applications in isolated **containers**. Multiple containers can run simultaneously on a single **host** within a private network, enabling separate containers—such as for the Nginx server, frontend UI, and backend Python—to construct our web service.

This article demonstrates a minimal web service that stays alive and responds to incoming network requests. It covers Docker container and Nginx server configuration, service deployment, and the traffic lifecycle to illustrate how these components work together.

---

## Prerequisites

This guide assumes a fresh Ubuntu environment with sudo privileges.

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

The `hello-world` container confirms Docker is properly installed. Nginx and the Python application will be containerized and automatically pulled when services are deployed, so there is no need to install or configure them on the host.

---

## Docker Setup

Two Docker containers will be built: one for the Nginx server and the other for a minimal Python application. By placing these services under the same Docker Compose file (`docker-compose.yml`), they automatically join an isolated virtual network where container names serve as internal domain names.

```yaml
version: '3.8'

services:
  web-app-service:
    image: python:3.12-slim
    restart: unless-stopped
    command: python -m http.server 8080

  nginx-proxy:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "127.0.0.1:80:80"  # <HOST_IP>:<HOST_PORT>:<CONTAINER_PORT>
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web-app-service

```

In this setup, `version: '3.8'` specifies the Docker Compose file format. The `services` section defines all containers to be deployed. Each service requires an `image` pulled from Docker Hub—a specific version can be assigned without manual installation or environment configuration. The `restart: unless-stopped` policy automatically restarts containers if they crash, unless manually stopped.

The `web-app-service` runs a Python image (`python:3.12-slim`). By default, this image would start an interactive Python shell and do nothing useful. The `command` directive overrides this default behavior and instead starts Python's built-in HTTP server on port `8080` with `python -m http.server 8080`. This server automatically listens on all network interfaces (`0.0.0.0:8080`), which allows Nginx to reach it through the private Docker network. Since this port is internal to the container, it remains hidden from the host and external users.

The `nginx-proxy` service uses the `nginx:alpine` image. The `ports` directive follows the format `<HOST_IP>:<HOST_PORT>:<CONTAINER_PORT>` (`127.0.0.1` is same as `localhost`). This means requests to `http://127.0.0.1:80` on the host are forwarded directly to the container's port `80`. The `volumes` directive mounts the local `nginx.conf` file into the container as read-only (`:ro`). The `depends_on` ensures the Python service starts before Nginx, since Nginx needs the backend service to forward traffic to.

---

## Nginx Setup

An Nginx configuration (`nginx.conf`) file should be created to define the routing rules and map public URLs to our containerized services.

```nginx
server {
    listen 80;  # Listens on the <CONTAINER_PORT>
    server_name localhost;

    location /my-app {
        proxy_pass http://web-app-service:8080/my-app;
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

This configuration defines how Nginx manages web traffic. The `listen 80` directive specifies the container port that Nginx listens on. The `server_name localhost;` directive specifies that this server block handles requests to `localhost` (or `127.0.0.1`). 

The `location /my-app` block handles requests to any URL starting with `/my-app` (such as `http://localhost/my-app`), and the `proxy_pass` directive forwards these requests to `web-app-service:8080/my-app` inside the container network. 

The `proxy_set_header` lines preserve the user's original connection details. Finally, the `return 404` block at the bottom serves as a catch-all security rule to reject any requests to undefined paths.

---

## Deployment

To spin up this entire multi-container service, open the terminal in the directory containing `docker-compose.yml` and execute:

```bash
docker compose up -d
```

- **`up`**: Tells Docker to build the images (if not built yet), create the private network, and start all services in the correct order based on `depends_on`.
- **`-d`**: Runs the containers in detached mode, executing in the background.

To stop and remove all containers and the private network:

```bash
docker compose down
```

---

## Traffic Lifecycle

With the web service deployed, the following section examines how network traffic flows through the request-response lifecycle when a user accesses the application:

```text
[ Browser ]
  |
  | User requests: http://xxx.x.x.x/my-app
  |
  |     ┌> IP recognized from outside: xxx.x.x.x
┌─|───Host───────────────────────────────────────────────────────────┐
| |     └> IP recognized in inside: 127.0.0.1                        |
| |                                                                  |
| |  ┌──────────────────────────────────┐                            |
| |  | Host Port 80                     |                            |
| └> | - Default HTTP port              |                            |
|    └──────────────────────────────────┘                            |
|         │                                                          |
|         │  Docker Daemon                                           |
|         │  - Map Host Port 80 to Container Port 80                 |
|         │                                                          |
|    ┌────|────-Private Docker Network────────────────────────────┐  |
|    │    ▼                                                       |  │
|    │ ┌─────────────────────────────────────────────────────┐    │  |
|    │ | Container Port 80                                   |    │  |
|    │ └─────────────────────────────────────────────────────┘    │  |
|    │    |                                                       │  |
|    │    │  nginx-prox (Nginx Process)                           │  │
|    │    │  - Listens on Container Port 80                       │  │
|    │    │  - Recognized `/my-app` from request                  │  │
|    │    │  - Proxy pass to: http://web-app-service:8080/my-app  │  │
|    │    │                                                       │  │
|    │    ▼                                                       │  │
|    │ ┌───────────────────────────────────────┐                  │  |
|    │ │ Container Port 8080                   │                  │  |
|    │ └───────────────────────────────────────┘                  │  |
|    │                                                            │  │
|    │       web-app-service (Python Process)                     │  │
|    │       - Listens on: 0.0.0.0:8080                           │  │
|    │       - Processes request for /my-app                      │  │
|    │       - Sends response back to nginx                       │  │
|    │                                                            │  │
|    └────────────────────────────────────────────────────────────┘  |
|                                                                    |
└────────────────────────────────────────────────────────────────────┘

(Response returns through same path)

```

The Nginx container sits at the very front of our server, acting as a traffic controller that accepts requests and routes them to the backend application.

When a user navigates to `http://<HOST_IP>/my-app`, the host machine intercepts traffic on host port `80` (the default HTTP port, therefore omitted from URLs) and maps it directly to the container's port `80` in the `nginx-proxy` container. Nginx reads the URL, matches it against `location /my-app`, and forwards the request to `http://web-app-service:8080/my-app` inside the virtual network. The Python container receives the request because it is listening on all local interfaces (`0.0.0.0:8080`).

The Python server processes the request and automatically generates a standard "Directory Listing" HTML page showing the container's internal folder structure. The response is sent back to Nginx through the same virtual network connection, which then serves it to the user's browser. Note that the Nginx configuration only explicitly declares the incoming route because `proxy_pass` establishes a two-way communication channel; the underlying protocol automatically handles the return path.

---

## Summary

This architecture establishes a secure network perimeter by isolation. Pairing Nginx with Docker Compose forces all public ingress traffic through port 80 of the Nginx gateway container, while the application container remains hidden inside the internal virtual network. 

For future maintenance and expansion, this architecture allows seamless tool integration. To add new components later, you only need to declare the new service inside `docker-compose.yml` and append a corresponding `location` block within `nginx.conf`. 

This multi-container blueprint ensures the foundational infrastructure is structurally ready for horizontal expansion.
