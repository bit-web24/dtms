# DTMS - Distributed Task Management System

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Features](#features)
4. [Quick Start](#quick-start)
5. [API Documentation](#api-documentation)
6. [Development Guide](#development-guide)
7. [Deployment](#deployment)
8. [Configuration](#configuration)
9. [Troubleshooting](#troubleshooting)
10. [Contributing](#contributing)

## Overview

DTMS is a microservices-based distributed task management system built with Go, gRPC, and PostgreSQL. The system consists of three main services:

- **User Service**: Manages user operations (CRUD)
- **Task Service**: Handles task management (CRUD)
- **API Gateway**: Provides HTTP/REST interface to the gRPC services

The system uses Docker and Docker Compose for containerization and orchestration, making it easy to deploy and scale.

## Architecture

### System Architecture Diagram

```
┌─────────────────┐
│   Client/UI     │
└────────┬────────┘
         │ HTTP/REST
┌────────▼────────┐
│   API Gateway   │
│   (Port 8080)   │
│   - gRPC-Gateway│
└────────┬────────┘
         │ gRPC
┌────────▼────────┐    ┌─────────────────┐
│  User Service   │    │   Task Service  │
│  (Port 50051)   │    │   (Port 50052)  │
│  - gRPC Server  │    │  - gRPC Server  │
└────────┬────────┘    └────────┬────────┘
         │                       │
┌────────▼────────┐    ┌─────────────────┐
│ PostgreSQL User │    │ PostgreSQL Task │
│   Database      │    │   Database      │
└─────────────────┘    └─────────────────┘
```

### Technology Stack

- **Backend**: Go 1.22.5
- **Communication**: gRPC + HTTP/REST (via gRPC-Gateway)
- **Database**: PostgreSQL
- **ORM**: GORM
- **Containerization**: Docker & Docker Compose
- **Protocol Buffers**: v3
- **Health Checks**: gRPC Health Checking Protocol

## Features

### User Service
- Create new users with username and email
- Retrieve user information by ID
- Delete users
- List all users
- Unique constraints on username and email

### Task Service
- Create tasks with title and description
- Assign tasks to users
- Retrieve task information
- Delete tasks
- List all tasks

### System Features
- Health checks for all services
- Docker containerization
- Environment-based configuration
- Auto-migration for database schemas
- UUID-based primary keys
- Timestamp tracking (created_at, updated_at)

## Quick Start

### Prerequisites
- Docker and Docker Compose
- Make utility
- Git

### Installation Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/bit-web24/DTMS.git
   cd DTMS
   ```

2. **Compile Protocol Buffer files**
   ```bash
   make
   ```
   This command generates Go code from the .proto files for both services.

3. **Configure Environment Variables**
   
   Create `.env` file in the root directory:
   ```bash
   SERVICE_USER_ADDR=user_service
   SERVICE_TASK_ADDR=task_service
   ```
   
   Create `services/user/.env`:
   ```bash
   DB_HOST=postgres_user
   DB_USER=bittu
   DB_PASSWORD=bittu
   DB_NAME=users
   DB_PORT=5432
   DB_SSLMODE=disable
   DB_TIME_ZONE=UTC
   RPC_PORT=50051
   HTTP_PORT=8081
   ```
   
   Create `services/task/.env`:
   ```bash
   DB_HOST=postgres_task
   DB_USER=bittu
   DB_PASSWORD=bittu
   DB_NAME=tasks
   DB_PORT=5432
   DB_SSLMODE=disable
   DB_TIME_ZONE=UTC
   RPC_PORT=50052
   HTTP_PORT=8082
   ```

4. **Start the System**
   ```bash
   sudo docker-compose up --build
   ```
   This will:
   - Build Docker images for all services
   - Start PostgreSQL databases
   - Initialize databases with required extensions
   - Start User and Task services
   - Start the API Gateway

5. **Verify the System**
   
   Check health of services:
   ```bash
   # User Service Health
   curl http://localhost:8081/health
   
   # Task Service Health
   curl http://localhost:8082/health
   
   # API Gateway (should list all available endpoints)
   curl http://localhost:8080/
   ```

### Stop the System
```bash
sudo docker-compose down
```

## API Documentation

### Base URL
```
http://localhost:8080
```

### User API Endpoints

#### Create User
- **Endpoint**: `POST /v1/users`
- **Request Body**:
  ```json
  {
    "username": "john_doe",
    "email": "john@example.com"
  }
  ```
- **Response**:
  ```json
  {
    "user": {
      "id": "uuid-string",
      "username": "john_doe",
      "email": "john@example.com"
    }
  }
  ```

#### Get User
- **Endpoint**: `GET /v1/users/{id}`
- **Path Parameters**:
  - `id`: User UUID
- **Response**:
  ```json
  {
    "user": {
      "id": "uuid-string",
      "username": "john_doe",
      "email": "john@example.com"
    }
  }
  ```

#### Delete User
- **Endpoint**: `DELETE /v1/users/{id}`
- **Path Parameters**:
  - `id`: User UUID
- **Response**:
  ```json
  {
    "success": true
  }
  ```

#### List All Users
- **Endpoint**: `GET /v1/users`
- **Response**:
  ```json
  {
    "users": [
      {
        "id": "uuid-string",
        "username": "john_doe",
        "email": "john@example.com"
      }
    ]
  }
  ```

### Task API Endpoints

#### Create Task
- **Endpoint**: `POST /v1/tasks`
- **Request Body**:
  ```json
  {
    "title": "Complete project documentation",
    "description": "Write comprehensive documentation for the DTMS project",
    "user_id": "user-uuid"
  }
  ```
- **Response**:
  ```json
  {
    "task": {
      "id": "uuid-string",
      "title": "Complete project documentation",
      "description": "Write comprehensive documentation for the DTMS project",
      "user_id": "user-uuid"
    }
  }
  ```

#### Get Task
- **Endpoint**: `GET /v1/tasks/{id}`
- **Path Parameters**:
  - `id`: Task UUID
- **Response**:
  ```json
  {
    "task": {
      "id": "uuid-string",
      "title": "Complete project documentation",
      "description": "Write comprehensive documentation for the DTMS project",
      "user_id": "user-uuid"
    }
  }
  ```

#### Delete Task
- **Endpoint**: `DELETE /v1/tasks/{id}`
- **Path Parameters**:
  - `id`: Task UUID
- **Response**:
  ```json
  {
    "success": true
  }
  ```

#### List All Tasks
- **Endpoint**: `GET /v1/tasks`
- **Response**:
  ```json
  {
    "tasks": [
      {
        "id": "uuid-string",
        "title": "Complete project documentation",
        "description": "Write comprehensive documentation for the DTMS project",
        "user_id": "user-uuid"
      }
    ]
  }
  ```

## Development Guide

### Project Structure

```
DTMS/
├── docs/                   # Documentation
├── health/                 # Health check utilities
├── initdb/                 # Database initialization scripts
├── proto/                  # Protocol Buffer definitions
│   ├── google/            # Google API annotations
│   ├── task.proto         # Task service definition
│   └── user.proto         # User service definition
├── services/              # Microservices
│   ├── task/              # Task service
│   │   ├── Dockerfile
│   │   ├── main.go
│   │   └── proto/         # Generated Go code
│   └── user/              # User service
│       ├── Dockerfile
│       ├── main.go
│       └── proto/         # Generated Go code
├── docker-compose.yml     # Docker orchestration
├── Dockerfile            # API Gateway Dockerfile
├── go.mod                # Go modules
├── go.sum                # Go dependencies
├── main.go               # API Gateway entry point
└── Makefile              # Build automation
```

### Adding New Services

1. **Define Protocol Buffers**
   - Create a new .proto file in the `proto/` directory
   - Define your service and messages
   - Include Google API annotations for HTTP mapping

2. **Update Makefile**
   - Add your service directory to the Makefile variables
   - Update the protoc command to include your .proto file

3. **Generate Go Code**
   ```bash
   make
   ```

4. **Implement Service**
   - Create a new directory under `services/`
   - Implement the gRPC server
   - Add health checks
   - Create Dockerfile

5. **Update Docker Compose**
   - Add your service to docker-compose.yml
   - Configure networking and dependencies
   - Add health checks

6. **Update API Gateway**
   - Import your generated package in main.go
   - Register your service handler

### Database Migrations

The system uses GORM's AutoMigrate feature. When you modify your model structs:

1. Update the struct definition in your service's main.go
2. The migration will run automatically on service startup

For manual migrations:
```sql
-- Connect to your database
psql -h localhost -U bittu -d users

-- View tables
\dt

-- View schema
\d users
```

### Testing Services

#### Unit Testing
```bash
# Run tests for a specific service
cd services/user
go test ./...

cd services/task
go test ./...
```

#### Integration Testing
```bash
# Test API endpoints
curl -X POST http://localhost:8080/v1/users \
  -H "Content-Type: application/json" \
  -d '{"username":"test_user","email":"test@example.com"}'

# Verify response
curl http://localhost:8080/v1/users/{user-id}
```

### Debugging

#### View Logs
```bash
# View logs for all services
docker-compose logs

# View logs for specific service
docker-compose logs user_service
docker-compose logs task_service
docker-compose logs dtms
```

#### Connect to Database
```bash
# User database
docker exec -it dtms_postgres_user_1 psql -U bittu -d users

# Task database
docker exec -it dtms_postgres_task_1 psql -U bittu -d tasks
```

## Deployment

### Production Deployment Considerations

1. **Security**
   - Change default passwords
   - Use environment-specific secrets
   - Enable SSL/TLS for database connections
   - Implement authentication/authorization

2. **Scalability**
   - Use Docker Swarm or Kubernetes for orchestration
   - Implement load balancing
   - Configure connection pooling
   - Add caching layer (Redis)

3. **Monitoring**
   - Implement metrics collection (Prometheus)
   - Add distributed tracing (Jaeger)
   - Set up alerting
   - Configure log aggregation (ELK stack)

4. **Data Persistence**
   - Configure persistent volumes for databases
   - Set up regular backups
   - Implement disaster recovery

### Environment Configuration

#### Development
```bash
# Use local Docker Compose
docker-compose -f docker-compose.yml up
```

#### Staging
```bash
# Use staging configuration
docker-compose -f docker-compose.staging.yml up
```

#### Production
```bash
# Use production configuration with overrides
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

## Configuration

### Environment Variables

#### API Gateway (.env)
```bash
SERVICE_USER_ADDR=user_service    # Address of user service
SERVICE_TASK_ADDR=task_service    # Address of task service
```

#### User Service (services/user/.env)
```bash
DB_HOST=postgres_user              # Database host
DB_USER=bittu                      # Database user
DB_PASSWORD=bittu                  # Database password
DB_NAME=users                      # Database name
DB_PORT=5432                       # Database port
DB_SSLMODE=disable                 # SSL mode
DB_TIME_ZONE=UTC                   # Timezone
RPC_PORT=50051                     # gRPC port
HTTP_PORT=8081                     # Health check port
```

#### Task Service (services/task/.env)
```bash
DB_HOST=postgres_task              # Database host
DB_USER=bittu                      # Database user
DB_PASSWORD=bittu                  # Database password
DB_NAME=tasks                      # Database name
DB_PORT=5432                       # Database port
DB_SSLMODE=disable                 # SSL mode
DB_TIME_ZONE=UTC                   # Timezone
RPC_PORT=50052                     # gRPC port
HTTP_PORT=8082                     # Health check port
```

### Docker Configuration

#### Port Mappings
- API Gateway: 8080 (HTTP)
- User Service: 50051 (gRPC), 8081 (Health)
- Task Service: 50052 (gRPC), 8082 (Health)
- PostgreSQL User: 5432
- PostgreSQL Task: 5432

#### Networks
- `service_net`: Communication between gateway and services
- `user_net`: User service and its database
- `task_net`: Task service and its database

## Troubleshooting

### Common Issues

#### Service Won't Start
1. Check if ports are in use:
   ```bash
   netstat -tulpn | grep :8080
   ```
2. Check Docker logs:
   ```bash
   docker-compose logs
   ```

#### Database Connection Errors
1. Verify database is healthy:
   ```bash
   docker exec -it dtms_postgres_user_1 pg_isready -U bittu
   ```
2. Check connection string in .env files

#### gRPC Connection Errors
1. Verify service addresses in docker-compose.yml
2. Check network configuration
3. Ensure services are healthy before starting gateway

### Health Check Failures

If health checks fail:
1. Check if services are running:
   ```bash
   docker ps
   ```
2. Verify health endpoint:
   ```bash
   curl http://localhost:8081/health
   ```
3. Check service logs for errors

### Protocol Buffer Compilation Issues

If `make` fails:
1. Install protoc:
   ```bash
   # Ubuntu/Debian
   sudo apt-get install protobuf-compiler
   
   # macOS
   brew install protobuf
   ```
2. Install Go plugins:
   ```bash
   go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
   go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
   go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
   ```
3. Ensure $GOPATH/bin is in PATH

## Contributing

### Development Workflow

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests for new functionality
5. Ensure all tests pass
6. Submit a pull request

### Code Style

- Follow Go formatting conventions
- Use meaningful variable names
- Add comments for complex logic
- Keep functions small and focused

### Submitting Changes

1. Update documentation for any API changes
2. Add/update protocol buffer definitions if needed
3. Test with Docker Compose
4. Ensure backward compatibility

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For support and questions:
- Create an issue on GitHub
- Check the troubleshooting section
- Review the API documentation