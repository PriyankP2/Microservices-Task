# Microservices Containerization – Docker Setup

A containerized Node.js microservices application with four services orchestrated via Docker Compose.

## Architecture

| Service         | Port | Endpoint                                          | Description                            |
|-----------------|------|---------------------------------------------------|----------------------------------------|
| User Service    | 3000 | `/users`                                          | Handles user data                      |
| Product Service | 3001 | `/products`                                       | Manages product catalogue              |
| Order Service   | 3002 | `/orders`                                         | Processes orders                       |
| Gateway Service | 3003 | `/api/users`, `/api/products`, `/api/orders`      | Routes requests to other services      |

All services communicate over a shared Docker bridge network (`microservices-network`).

---

## Prerequisites

- An EC2 instance (Ubuntu recommended) with ports **3000, 3001, 3002, 3003** open in the Security Group
- Docker and Docker Compose installed on the instance

### Install Docker on EC2 (Ubuntu)
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu   # avoids needing sudo for every docker command
newgrp docker                    # apply group change without logging out
```

### Install Docker Compose
```bash
sudo apt install -y docker-compose
docker-compose --version         # verify install
```

---

## Setup & Running

### 1. Clone your repository onto the EC2 instance
```bash
git clone <your-forked-repo-url>
cd submission
```

### 2. Build and start all services
```bash
docker-compose up --build
```

To run in detached (background) mode:
```bash
docker-compose up --build -d
```

### 3. Verify all containers are running
```bash
docker-compose ps
```

All four services should show status `Up`.

### 4. Stop all services
```bash
docker-compose down
```

---

## Testing Each Service

Replace `<EC2-PUBLIC-IP>` with your EC2 instance's public IP (found in the AWS Console under EC2 > Instances).

### User Service (Port 3000)
```bash
curl http://<EC2-PUBLIC-IP>:3000/users
```
Or open in browser: `http://<EC2-PUBLIC-IP>:3000/users`
<img width="652" height="302" alt="image" src="https://github.com/user-attachments/assets/a8587c61-e284-46f3-8899-adae6d6e9520" />

### Product Service (Port 3001)
```bash
curl http://<EC2-PUBLIC-IP>:3001/products
```
Or open in browser: `http://<EC2-PUBLIC-IP>:3001/products`
<img width="675" height="157" alt="image" src="https://github.com/user-attachments/assets/96326554-0aed-4ff5-91ec-b1f99104c0c8" />

### Order Service (Port 3002)
```bash
curl http://<EC2-PUBLIC-IP>:3002/orders
```
Or open in browser: `http://<EC2-PUBLIC-IP>:3002/orders`

<img width="516" height="143" alt="image" src="https://github.com/user-attachments/assets/a63b087c-7892-4c15-9a1b-57d13e35e67b" />


### Gateway Service (Port 3003)
```bash
curl http://<EC2-PUBLIC-IP>:3003/api/users
curl http://<EC2-PUBLIC-IP>:3003/api/products
curl http://<EC2-PUBLIC-IP>:3003/api/orders
```
<img width="542" height="188" alt="image" src="https://github.com/user-attachments/assets/0514df66-81a5-4a12-870f-2ef66cdcf9d5" />

<img width="696" height="182" alt="image" src="https://github.com/user-attachments/assets/f1501a11-c110-4a0a-aa9d-bb723cf982fc" />

<img width="577" height="141" alt="image" src="https://github.com/user-attachments/assets/62943cf3-9516-474d-909a-a7beaba53508" />


> **Note:** If running curl from within the EC2 instance itself, you can use `localhost` instead of the public IP.

---

---

## Dockerfile Overview

Each service uses the same structure — only the EXPOSE port differs:

```dockerfile
FROM node:18-alpine       # Lightweight official Node.js image
WORKDIR /app              # Set working directory inside container
COPY package*.json ./     # Copy dependency manifest first (layer caching)
RUN npm install           # Install dependencies
COPY . .                  # Copy application source code
EXPOSE <port>             # Declare the port the service listens on
CMD ["node", "index.js"]  # Start the application
```

---

## Troubleshooting

### Cannot connect from browser or curl times out
Open ports in your EC2 **Security Group**:
- Go to AWS Console → EC2 → Security Groups → Inbound Rules
- Add rule: Custom TCP | Port range `3000-3003` | Source `0.0.0.0/0`

### A service fails to start — check logs
```bash
docker-compose logs user-service
docker-compose logs product-service
docker-compose logs order-service
docker-compose logs gateway-service
```

### Port already in use on EC2
```bash
sudo lsof -i :3000
sudo kill -9 <PID>
```

### Rebuild after making code changes
```bash
docker-compose down
docker-compose up --build
```

### Full clean reset (removes images and volumes)
```bash
docker-compose down --rmi all --volumes
docker-compose up --build
```

---

## Screenshots

> 1. `docker-compose up --build` terminal output — all 4 services starting successfully
  <img width="902" height="210" alt="image" src="https://github.com/user-attachments/assets/cc81276d-4886-4ff2-8c47-8537223a85d8" />


> 2. `docker-compose ps` — all containers with status `Up`
  <img width="1856" height="132" alt="image" src="https://github.com/user-attachments/assets/a6e96c14-0a0d-4762-a74b-8d7dc955211d" />



> 3. Curl or browser responses from each service endpoint
  <img width="1131" height="232" alt="image" src="https://github.com/user-attachments/assets/f7637810-d8df-4451-86f6-d45f90492dfa" />

