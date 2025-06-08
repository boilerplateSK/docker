# Docker for Python Web Developers: A Comprehensive Guide

A complete guide to containerizing Python web applications using Docker, covering core concepts, best practices, and practical examples for Django and FastAPI applications.

## Table of Contents

- [Introduction](#introduction)
- [Core Docker Concepts](#core-docker-concepts)
- [Essential Docker Commands](#essential-docker-commands)
- [Creating Docker Images](#creating-docker-images)
- [Dockerizing Django Applications](#dockerizing-django-applications)
- [Dockerizing FastAPI Applications](#dockerizing-fastapi-applications)
- [Best Practices](#best-practices)
- [Quick Start](#quick-start)

## Introduction

Docker has revolutionized software development by providing a standardized approach to packaging and running applications. This guide focuses specifically on containerizing Python web applications, offering practical insights for modern development workflows.

### Why Use Docker?

- **Portability & Consistency**: "It works on my machine" becomes "It works everywhere"
- **Efficiency**: Lightweight containers vs heavy virtual machines
- **Isolation & Security**: Each container operates in isolation
- **Simplified Workflows**: Streamlined development, testing, and deployment
- **Vibrant Ecosystem**: Extensive community and tooling support

## Core Docker Concepts

### Containers
- **Runtime instances** of Docker images
- Isolated processes sharing the host OS kernel
- Ephemeral by nature - data is lost when container is removed
- Lightweight and fast startup compared to VMs

### Images
- **Immutable blueprints** for creating containers
- Built from layers (each Dockerfile instruction creates a layer)
- Stored in registries like Docker Hub
- Can be shared and reused across environments

### Volumes
- **Persistent storage** that outlives container lifecycle
- Managed by Docker daemon
- Essential for databases and stateful applications
- Mounted to specific paths within containers

### Networks
- Enable **communication** between containers and external services
- Multiple network drivers available (bridge, host, overlay, etc.)
- Support for custom networks and DNS resolution
- Port publishing for external access

## Essential Docker Commands

### Container Management

```bash
# Create and run a container
docker run -d -p 8000:8000 --name my-app my-image

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container gracefully
docker stop my-app

# Remove a container
docker rm my-app

# Remove container automatically after exit
docker run --rm my-image
```

### Image Management

```bash
# Build an image from Dockerfile
docker build -t my-app:latest .

# Pull an image from registry
docker pull python:3.9-slim

# Push an image to registry
docker push my-registry.com/my-app:latest

# List local images
docker images

# Remove an image
docker rmi my-app:latest

# Remove dangling images
docker rmi $(docker images -f "dangling=true" -q)
```

## Creating Docker Images

### Sample Dockerfile Structure

```dockerfile
# Use lightweight base image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements first (for caching)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m appuser && chown -R appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Define startup command
CMD ["python", "app.py"]
```

### Key Dockerfile Instructions

- `FROM`: Specify base image
- `WORKDIR`: Set working directory
- `COPY/ADD`: Add files to image
- `RUN`: Execute commands during build
- `ENV`: Set environment variables
- `EXPOSE`: Document ports
- `USER`: Set user context
- `CMD/ENTRYPOINT`: Define startup command

## Dockerizing Django Applications

### Project Setup

1. **Create requirements.txt**:
```text
Django==5.1.3
gunicorn==23.0.0
psycopg2-binary==2.9.10
```

2. **Sample Django Dockerfile**:
```dockerfile
# Multi-stage build for optimization
FROM python:3.13-slim AS builder

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.13-slim

# Create non-root user
RUN useradd -m -r appuser && mkdir /app && chown -R appuser /app
WORKDIR /app
USER appuser

# Copy dependencies from builder
COPY --from=builder /usr/local/lib/python3.13/site-packages /usr/local/lib/python3.13/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application code
COPY . .

EXPOSE 8000

# Use Gunicorn for production
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "my_project.wsgi:application"]
```

### Build and Run Django App

```bash
# Build the image
docker build -t django-app:latest .

# Run the container
docker run -d -p 8000:8000 --name my-django-app django-app:latest

# Access at http://localhost:8000
```

## Dockerizing FastAPI Applications

### Project Setup

1. **Create requirements.txt**:
```text
fastapi[standard]>=0.113.0,<0.114.0
uvicorn[standard]>=0.29.0,<0.30.0
```

2. **Sample FastAPI Dockerfile**:
```dockerfile
FROM python:3.9-slim

WORKDIR /code

# Copy requirements first for caching
COPY ./requirements.txt /code/requirements.txt

# Install dependencies
RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt

# Copy application code
COPY ./app /code/app

EXPOSE 80

# Use exec form for proper signal handling
CMD ["fastapi", "run", "app/main.py", "--port", "80"]
```

### Build and Run FastAPI App

```bash
# Build the image
docker build -t fastapi-app:latest .

# Run the container
docker run -d -p 8000:80 --name my-fastapi-app fastapi-app:latest

# Access at http://localhost:8000
# API docs at http://localhost:8000/docs
```

## Best Practices

### Optimization Techniques

1. **Use Lightweight Base Images**
```dockerfile
# Good
FROM python:3.9-slim

# Better for compiled apps
FROM gcr.io/distroless/python3
```

2. **Multi-Stage Builds**
```dockerfile
# Build stage
FROM python:3.9 AS builder
RUN pip install dependencies

# Production stage
FROM python:3.9-slim
COPY --from=builder /usr/local/lib/python3.9/site-packages/ /usr/local/lib/python3.9/site-packages/
```

3. **Minimize Layers**
```dockerfile
# Combine commands
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

4. **Optimize Caching**
```dockerfile
# Copy requirements first
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy code last (changes frequently)
COPY . .
```

### Security Best Practices

1. **Run as Non-Root User**
```dockerfile
RUN useradd -r -s /bin/false appuser
USER appuser
```

2. **Pin Dependencies**
```dockerfile
FROM python:3.9.16-slim  # Not python:3.9 or python:latest
```

3. **Use .dockerignore**
```
__pycache__
*.pyc
.git
.venv
node_modules
.pytest_cache
```

4. **Avoid Hardcoded Secrets**
```bash
# Use environment variables
docker run -e DATABASE_URL=postgres://... my-app

# Or Docker secrets (in swarm mode)
docker service create --secret db_password my-app
```

### Development Workflow

1. **Docker Compose for Multi-Service Apps**
```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    environment:
      - DEBUG=1
    depends_on:
      - db
  
  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

2. **Health Checks**
```dockerfile
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

## Quick Start

### Django Quick Start
```bash
# Clone your Django project
git clone <your-django-repo>
cd your-django-project

# Create Dockerfile (use sample above)
# Create requirements.txt
pip freeze > requirements.txt

# Build and run
docker build -t my-django-app .
docker run -d -p 8000:8000 --name django-container my-django-app
```

### FastAPI Quick Start
```bash
# Clone your FastAPI project
git clone <your-fastapi-repo>
cd your-fastapi-project

# Create Dockerfile (use sample above)
# Build and run
docker build -t my-fastapi-app .
docker run -d -p 8000:80 --name fastapi-container my-fastapi-app
```

## Troubleshooting

### Common Issues

1. **Port Already in Use**
```bash
# Use different host port
docker run -p 8001:8000 my-app
```

2. **Permission Denied**
```bash
# Check user permissions in Dockerfile
USER appuser
```

3. **Build Cache Issues**
```bash
# Force rebuild without cache
docker build --no-cache -t my-app .
```

4. **Container Exits Immediately**
```bash
# Check logs
docker logs container-name

# Run interactively for debugging
docker run -it my-app /bin/bash
```

### Useful Debugging Commands

```bash
# Execute command in running container
docker exec -it container-name /bin/bash

# View container logs
docker logs -f container-name

# Inspect container details
docker inspect container-name

# View resource usage
docker stats

# Clean up system
docker system prune -a
```

## Additional Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Best Practices for Writing Dockerfiles](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

## Contributing

Feel free to contribute to this guide by submitting pull requests or opening issues for improvements and corrections.

## License

This guide is provided under the MIT License. See LICENSE file for details.
