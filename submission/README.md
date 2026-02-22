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

### Product Service (Port 3001)
```bash
curl http://<EC2-PUBLIC-IP>:3001/products
```
Or open in browser: `http://<EC2-PUBLIC-IP>:3001/products`

### Order Service (Port 3002)
```bash
curl http://<EC2-PUBLIC-IP>:3002/orders
```
Or open in browser: `http://<EC2-PUBLIC-IP>:3002/orders`

### Gateway Service (Port 3003)
```bash
curl http://<EC2-PUBLIC-IP>:3003/api/users
curl http://<EC2-PUBLIC-IP>:3003/api/products
curl http://<EC2-PUBLIC-IP>:3003/api/orders
```

> **Note:** If running curl from within the EC2 instance itself, you can use `localhost` instead of the public IP.

---

## Project Structure

```
submission/
├── user-service/
│   └── Dockerfile
├── product-service/
│   └── Dockerfile
├── order-service/
│   └── Dockerfile
├── gateway-service/
│   └── Dockerfile
├── docker-compose.yml
└── README.md
```

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

> Add screenshots here showing:
> 1. `docker-compose up --build` terminal output — all 4 services starting successfully
> 2. `docker-compose ps` — all containers with status `Up`
> 3. Curl or browser responses from each service endpoint
