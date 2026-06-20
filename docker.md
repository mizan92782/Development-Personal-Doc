# Docker Essential Commands for Daily Development

## Container Commands

```bash
# Show running containers
docker ps

# Show all containers (running + stopped)
docker ps -a

# Show only container IDs
docker ps -q

# Show only container names
docker ps --format "{{.Names}}"

# Show container names and status
docker ps --format "table {{.Names}}\t{{.Status}}"

# Stop a container
docker stop <container_name>

# Start a container
docker start <container_name>

# Restart a container
docker restart <container_name>

# Stop all running containers
docker stop $(docker ps -q)

# Remove a container
docker rm <container_name>

# Force remove container
docker rm -f <container_name>

# Remove all containers
docker rm -f $(docker ps -aq)
```

## Logs

```bash
# Show logs
docker logs <container_name>

# Follow logs in real-time
docker logs -f <container_name>

# Show last 100 log lines
docker logs --tail 100 <container_name>

# Show logs with timestamps
docker logs -t <container_name>
```

## Execute Commands Inside Container

```bash
# Open shell (Alpine Linux)
docker exec -it <container_name> sh

# Open bash shell (Ubuntu/Debian)
docker exec -it <container_name> bash

# Execute a command
docker exec -it <container_name> ls

# Check environment variables
docker exec -it <container_name> env
```

## Inspect Container

```bash
# Full details
docker inspect <container_name>

# Show status only
docker inspect -f '{{.State.Status}}' <container_name>

# Show IP Address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_name>
```

## Resource Usage

```bash
# Monitor all containers
docker stats

# Monitor one container
docker stats <container_name>
```

## Images

```bash
# List images
docker images

# Pull image
docker pull nginx

# Remove image
docker rmi <image_name>

# Remove image forcefully
docker rmi -f <image_name>

# Build image
docker build -t myapp .

# Build image without cache
docker build --no-cache -t myapp .
```

## Networks

```bash
# List networks
docker network ls

# Inspect network
docker network inspect <network_name>

# Create network
docker network create mynetwork

# Remove network
docker network rm mynetwork
```

## Volumes

```bash
# List volumes
docker volume ls

# Create volume
docker volume create myvolume

# Inspect volume
docker volume inspect myvolume

# Remove volume
docker volume rm myvolume
```

## Docker Compose

```bash
# Start services
docker compose up

# Start services in background
docker compose up -d

# Build and start
docker compose up --build

# Build and start in background
docker compose up --build -d

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# Restart services
docker compose restart

# Show logs
docker compose logs

# Follow logs
docker compose logs -f

# Show logs for one service
docker compose logs -f api

# List services
docker compose ps

# Execute shell inside service
docker compose exec api sh

# Rebuild one service
docker compose up --build api
```

## Cleanup Commands

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove unused networks
docker network prune

# Remove unused volumes
docker volume prune

# Remove everything unused
docker system prune

# Remove everything unused including images
docker system prune -a

# Remove everything unused including volumes
docker system prune -a --volumes
```

## Debugging

```bash
# Check why container exited
docker inspect <container_name>

# Check container logs
docker logs <container_name>

# View events in Docker
docker events

# View Docker version
docker version

# View Docker system info
docker info
```

## Postgres Examples

```bash
# Enter postgres container
docker exec -it fastapi_postgres sh

# Connect to postgres
psql -U postgres
```

## Redis Examples

```bash
# Enter redis container
docker exec -it fastapi_redis sh

# Open redis cli
redis-cli
```

## RabbitMQ Examples

```bash
# Enter rabbitmq container
docker exec -it fastapi_rabbitmq sh

# Check queues
rabbitmqctl list_queues
```

## Very Common Daily Commands

```bash
docker ps
docker compose up -d
docker compose down
docker compose logs -f
docker restart <container_name>
docker exec -it <container_name> sh
docker stats
docker inspect <container_name>
docker stop $(docker ps -q)
docker system prune
```
