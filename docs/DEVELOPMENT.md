# DTMS Development Guide

## Overview

This guide is for developers who want to contribute to or extend the DTMS project. It covers the development setup, coding standards, testing practices, and contribution workflow.

## Prerequisites

### Required Tools
- **Go 1.22.5+**: Programming language
- **Docker & Docker Compose**: Containerization
- **Make**: Build automation
- **Git**: Version control
- **protoc**: Protocol buffer compiler
- **PostgreSQL Client** (optional): For direct database access

### Go Tools Installation

Install the required Go protocol buffer plugins:

```bash
# Install protoc-gen-go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# Install protoc-gen-go-grpc
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Install grpc-gateway plugin
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest

# Ensure $GOPATH/bin is in your PATH
export PATH=$PATH:$(go env GOPATH)/bin
```

## Development Setup

### 1. Clone and Setup

```bash
# Clone the repository
git clone https://github.com/bit-web24/DTMS.git
cd DTMS

# Create development branches
git checkout -b develop
git checkout -b feature/your-feature-name
```

### 2. Generate Proto Files

```bash
# Compile all protocol buffer files
make

# Verify generated files
ls -la services/*/proto/
```

### 3. Configure Environment

```bash
# Copy environment templates
cp .env.example .env
cp services/user/.env.example services/user/.env
cp services/task/.env.example services/task/.env

# Edit the files with your local configuration
```

### 4. Start Development Environment

```bash
# Start all services
docker-compose up --build

# Or start in detached mode
docker-compose up -d --build

# View logs
docker-compose logs -f
```

### 5. Verify Setup

```bash
# Check service health
curl http://localhost:8081/health  # User service
curl http://localhost:8082/health  # Task service

# Test API gateway
curl http://localhost:8080/v1/users
```

## Project Structure Deep Dive

```
DTMS/
├── docs/                    # Documentation files
│   ├── README.md           # Main documentation
│   ├── ARCHITECTURE.md     # System architecture
│   ├── API_REFERENCE.md    # API documentation
│   └── DEVELOPMENT.md      # This file
├── health/                  # Health check utilities
│   └── health.go           # gRPC health check client
├── initdb/                  # Database initialization
│   ├── init-postgres_user.sql
│   └── init-postgres_task.sql
├── proto/                   # Protocol buffer definitions
│   ├── google/             # Google API annotations
│   │   └── api/
│   │       ├── annotations.proto
│   │       └── http.proto
│   ├── task.proto          # Task service definition
│   └── user.proto          # User service definition
├── services/                # Microservices
│   ├── task/               # Task service
│   │   ├── Dockerfile      # Docker build file
│   │   ├── main.go         # Service entry point
│   │   └── proto/          # Generated Go code
│   │       ├── task.pb.go
│   │       ├── task_grpc.pb.go
│   │       └── task.pb.gw.go
│   └── user/               # User service
│       ├── Dockerfile
│       ├── main.go
│       └── proto/
│           ├── user.pb.go
│           ├── user_grpc.pb.go
│           └── user.pb.gw.go
├── .env                    # Environment variables (not tracked)
├── .env.example           # Environment template
├── docker-compose.yml     # Service orchestration
├── docker-compose.dev.yml # Development overrides
├── Dockerfile             # API Gateway Dockerfile
├── go.mod                 # Go module definition
├── go.sum                 # Go dependencies checksum
├── main.go                # API Gateway entry point
├── Makefile               # Build commands
└── README.md              # Quick start guide
```

## Adding a New Service

Let's walk through adding a "Project" service to manage projects.

### 1. Define the Protocol Buffer

Create `proto/project.proto`:

```protobuf
syntax = "proto3";

package project;

import "google/api/annotations.proto";

option go_package = "services/project/proto";

service ProjectService {
    rpc CreateProject (CreateProjectRequest) returns (CreateProjectResponse) {
        option (google.api.http) = {
            post: "/v1/projects"
            body: "*"
        };
    }
    rpc GetProject (GetProjectRequest) returns (GetProjectResponse) {
        option (google.api.http) = {
            get: "/v1/projects/{id}"
        };
    }
    rpc DeleteProject (DeleteProjectRequest) returns (DeleteProjectResponse) {
        option (google.api.http) = {
            delete: "/v1/projects/{id}"
        };
    }
    rpc GetAllProjects (GetAllProjectsRequest) returns (GetAllProjectsResponse) {
        option (google.api.http) = {
            get: "/v1/projects"
        };
    }
}

message Project {
    string id = 1;
    string name = 2;
    string description = 3;
    string owner_id = 4;
}

message CreateProjectRequest {
    string name = 1;
    string description = 2;
    string owner_id = 3;
}

message CreateProjectResponse {
    Project project = 1;
}

message GetProjectRequest {
    string id = 1;
}

message GetProjectResponse {
    Project project = 1;
}

message DeleteProjectRequest {
    string id = 1;
}

message DeleteProjectResponse {
    bool success = 1;
}

message GetAllProjectsRequest {}

message GetAllProjectsResponse {
    repeated Project projects = 1;
}
```

### 2. Update the Makefile

```makefile
PROTOC = protoc
PROTO_DIR = ./proto
SERVICE_TASK_DIR = ./services/task/proto
SERVICE_USER_DIR = ./services/user/proto
SERVICE_PROJECT_DIR = ./services/project/proto  # Add this line
GAPI_DIR = ./proto/google

PROTO_FILES = $(PROTO_DIR)/task.proto $(PROTO_DIR)/user.proto $(PROTO_DIR)/project.proto  # Add project.proto

all: generate

generate: $(PROTO_FILES)
	mkdir -p $(SERVICE_TASK_DIR)
	mkdir -p $(SERVICE_USER_DIR)
	mkdir -p $(SERVICE_PROJECT_DIR)  # Add this line
	
	$(PROTOC) -I $(PROTO_DIR) -I $(GAPI_DIR) \
		--go_out=$(SERVICE_TASK_DIR) --go_opt=paths=source_relative \
		--go-grpc_out=$(SERVICE_TASK_DIR) --go-grpc_opt=paths=source_relative \
		--grpc-gateway_out=$(SERVICE_TASK_DIR) --grpc-gateway_opt=paths=source_relative \
		$(PROTO_DIR)/task.proto
	$(PROTOC) -I $(PROTO_DIR) -I $(GAPI_DIR) \
		--go_out=$(SERVICE_USER_DIR) --go_opt=paths=source_relative \
		--go-grpc_out=$(SERVICE_USER_DIR) --go-grpc_opt=paths=source_relative \
		--grpc-gateway_out=$(SERVICE_USER_DIR) --grpc-gateway_opt=paths=source_relative \
		$(PROTO_DIR)/user.proto
	$(PROTOC) -I $(PROTO_DIR) -I $(GAPI_DIR) \
		--go_out=$(SERVICE_PROJECT_DIR) --go_opt=paths=source_relative \
		--go-grpc_out=$(SERVICE_PROJECT_DIR) --go-grpc_opt=paths=source_relative \
		--grpc-gateway_out=$(SERVICE_PROJECT_DIR) --grpc-gateway_opt=paths=source_relative \
		$(PROTO_DIR)/project.proto  # Add this block

.PHONY: all generate
```

### 3. Generate Go Code

```bash
make
```

### 4. Implement the Service

Create `services/project/main.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"net/http"
	"os"
	"time"

	"github.com/google/uuid"
	"github.com/joho/godotenv"

	pb "github.com/bit-web24/DTMS/services/project/proto"

	"google.golang.org/grpc"
	"google.golang.org/grpc/health"
	"google.golang.org/grpc/health/grpc_health_v1"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

type Project struct {
	gorm.Model
	ID          string    `gorm:"type:uuid;default:uuid_generate_v4();primary_key"`
	Name        string    `gorm:"size:255;not null"`
	Description string    `gorm:"size:255"`
	OwnerID     string    `gorm:"size:255"`
	CreatedAt   time.Time `gorm:"default:current_timestamp"`
	UpdatedAt   time.Time `gorm:"default:current_timestamp"`
}

type server struct {
	pb.UnimplementedProjectServiceServer
	db *gorm.DB
}

// Implement all service methods...
func (s *server) CreateProject(ctx context.Context, req *pb.CreateProjectRequest) (*pb.CreateProjectResponse, error) {
	project := &Project{
		ID:          uuid.New().String(),
		Name:        req.GetName(),
		Description: req.GetDescription(),
		OwnerID:     req.GetOwnerId(),
	}
	result := s.db.Create(project)
	if result.Error != nil {
		return nil, result.Error
	}
	return &pb.CreateProjectResponse{Project: &pb.Project{
		Id:          project.ID,
		Name:        project.Name,
		Description: project.Description,
		OwnerId:     project.OwnerID,
	}}, nil
}

// ... implement other methods (GetProject, DeleteProject, GetAllProjects)

func main() {
	err := godotenv.Load(".env")
	if err != nil {
		log.Fatalf("Error loading .env file")
	}

	dsn := fmt.Sprintf(
		"host=%s user=%s password=%s dbname=%s port=%s sslmode=%s TimeZone=%s",
		os.Getenv("DB_HOST"),
		os.Getenv("DB_USER"),
		os.Getenv("DB_PASSWORD"),
		os.Getenv("DB_NAME"),
		os.Getenv("DB_PORT"),
		os.Getenv("DB_SSLMODE"),
		os.Getenv("DB_TIME_ZONE"),
	)

	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})

	if err != nil {
		log.Fatalf("failed to connect to database: %v", err)
	}

	db.Exec("CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\"")

	if err := db.AutoMigrate(&Project{}); err != nil {
		log.Fatalf("Migration failed: %v", err)
	}

	lis, err := net.Listen("tcp", ":"+os.Getenv("RPC_PORT"))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterProjectServiceServer(s, &server{db: db})
	healthServer := health.NewServer()
	grpc_health_v1.RegisterHealthServer(s, healthServer)
	healthServer.SetServingStatus("project_service", grpc_health_v1.HealthCheckResponse_SERVING)

	// Start the gRPC server in a separate goroutine
	go func() {
		log.Printf("gRPC server listening at %v", lis.Addr())
		if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
		}
	}()

	// Start an HTTP server for the health check
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("OK"))
	})
	httpPort := os.Getenv("HTTP_PORT")
	log.Printf("HTTP server listening on port %s", httpPort)
	if err := http.ListenAndServe(":"+httpPort, nil); err != nil {
		log.Fatalf("failed to start HTTP server: %v", err)
	}
}
```

### 5. Create Dockerfile

Create `services/project/Dockerfile`:

```dockerfile
FROM golang:1.22.5-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
COPY services/project/proto ./services/project/proto
COPY services/project/main.go ./services/project/
COPY proto ./proto

# Build the service
RUN CGO_ENABLED=0 GOOS=linux go build -o project_service ./services/project

FROM alpine:latest

RUN apk --no-cache add ca-certificates curl

WORKDIR /root/

# Copy the binary
COPY --from=builder /app/project_service .

# Copy env file
COPY services/project/.env .

# Expose ports
EXPOSE 50053 8083

# Run the binary
CMD ["./project_service"]
```

### 6. Update Docker Compose

Add to `docker-compose.yml`:

```yaml
  project_service:
    build:
      context: .
      dockerfile: ./services/project/Dockerfile
    ports:
      - "50053:50053"
    env_file:
      - ./services/project/.env
    depends_on:
      postgres_project:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8083/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - project_net
      - service_net

  postgres_project:
    image: postgres:latest
    environment:
      POSTGRES_USER: bittu
      POSTGRES_PASSWORD: bittu
      POSTGRES_DB: projects
    volumes:
      - postgres-project-data:/var/lib/postgresql/data
      - ./initdb/init-postgres_project.sql:/docker-entrypoint-initdb.d/init-postgres_project.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bittu -d projects"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - project_net

# Add to volumes section
  postgres-project-data:

# Add to networks section
  project_net:
```

### 7. Update API Gateway

Update `main.go`:

```go
// Add import
projectpb "github.com/bit-web24/DTMS/services/project/proto"

// In main function
projectAddr := os.Getenv("SERVICE_PROJECT_ADDR")

if projectAddr == "" {
    log.Fatalf("Environment variable SERVICE_PROJECT_ADDR is not set")
}

projectServiceAddr := fmt.Sprintf("%s:50053", projectAddr)

fmt.Println("Project Service Address:", projectServiceAddr)
health.CheckHealth(projectServiceAddr, "project_service")

// Register with gateway
err = projectpb.RegisterProjectServiceHandlerFromEndpoint(ctx, mux, projectServiceAddr, opts)
if err != nil {
    log.Fatalf("Failed to start HTTP gateway for ProjectService: %v", err)
}
```

## Testing Strategies

### Unit Testing

Create test files alongside your source files:

```go
// services/user/main_test.go
package main

import (
	"context"
	"testing"
	"time"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

// Mock database for testing
type MockDB struct {
	mock.Mock
}

func (m *MockDB) Create(value interface{}) *gorm.DB {
	args := m.Called(value)
	return args.Get(0).(*gorm.DB)
}

func TestCreateUser(t *testing.T) {
	// Setup
	mockDB := new(MockDB)
	s := &server{db: mockDB}
	
	req := &pb.CreateUserRequest{
		Username: "test_user",
		Email:    "test@example.com",
	}
	
	// Expectations
	mockDB.On("Create", mock.AnythingOfType("*main.User")).Return(&gorm.DB{Error: nil})
	
	// Execute
	resp, err := s.CreateUser(context.Background(), req)
	
	// Assert
	assert.NoError(t, err)
	assert.NotNil(t, resp)
	assert.Equal(t, "test_user", resp.User.Username)
	assert.Equal(t, "test@example.com", resp.User.Email)
	assert.NotEmpty(t, resp.User.Id)
	
	mockDB.AssertExpectations(t)
}
```

### Integration Testing

Create `tests/integration/` directory:

```go
// tests/integration/api_test.go
package integration

import (
	"bytes"
	"encoding/json"
	"net/http"
	"testing"
	"time"
)

const apiBase = "http://localhost:8080"

func TestUserCRUD(t *testing.T) {
	// Wait for services to be ready
	time.Sleep(5 * time.Second)
	
	// Create user
	userPayload := map[string]string{
		"username": "integration_test",
		"email":    "integration@test.com",
	}
	
	body, _ := json.Marshal(userPayload)
	resp, err := http.Post(apiBase+"/v1/users", "application/json", bytes.NewBuffer(body))
	
	if err != nil {
		t.Fatalf("Failed to create user: %v", err)
	}
	defer resp.Body.Close()
	
	if resp.StatusCode != http.StatusCreated {
		t.Errorf("Expected status 201, got %d", resp.StatusCode)
	}
	
	// Parse response
	var createUserResp map[string]interface{}
	json.NewDecoder(resp.Body).Decode(&createUserResp)
	
	userID := createUserResp["user"].(map[string]interface{})["id"].(string)
	
	// Get user
	resp, err = http.Get(apiBase + "/v1/users/" + userID)
	if err != nil {
		t.Fatalf("Failed to get user: %v", err)
	}
	defer resp.Body.Close()
	
	if resp.StatusCode != http.StatusOK {
		t.Errorf("Expected status 200, got %d", resp.StatusCode)
	}
	
	// Delete user
	req, _ := http.NewRequest("DELETE", apiBase+"/v1/users/"+userID, nil)
	client := &http.Client{}
	resp, err = client.Do(req)
	
	if err != nil {
		t.Fatalf("Failed to delete user: %v", err)
	}
	defer resp.Body.Close()
	
	if resp.StatusCode != http.StatusOK {
		t.Errorf("Expected status 200, got %d", resp.StatusCode)
	}
}
```

### End-to-End Testing

Create `tests/e2e/` directory with scenario tests:

```go
// tests/e2e/workflows_test.go
package e2e

import (
	"testing"
	"time"
)

func TestCompleteWorkflow(t *testing.T) {
	// This test simulates a real user workflow
	// 1. Create a user
	// 2. Create tasks for the user
	// 3. Retrieve tasks
	// 4. Update task status (when implemented)
	// 5. Clean up
}
```

### Running Tests

```bash
# Run all tests
go test ./...

# Run tests with coverage
go test -cover ./...

# Run integration tests
go test ./tests/integration/...

# Run e2e tests
go test ./tests/e2e/...

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

## Database Development

### Migrations

While DTMS uses GORM AutoMigrate, for production you should use proper migrations:

Create `migrations/` directory:

```go
// migrations/001_create_users_table.go
package migrations

import (
	"database/sql"
	"fmt"
)

type Migration001 struct {
	Db *sql.DB
}

func (m Migration001) Up() error {
	query := `
	CREATE TABLE IF NOT EXISTS users (
		id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
		username VARCHAR(255) UNIQUE NOT NULL,
		email VARCHAR(255) UNIQUE NOT NULL,
		created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
		updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
	);`
	
	_, err := m.Db.Exec(query)
	return err
}

func (m Migration001) Down() error {
	query := "DROP TABLE IF EXISTS users;"
	_, err := m.Db.Exec(query)
	return err
}

func (m Migration001) Description() string {
	return "Create users table with UUID primary key"
}
```

### Database Utilities

Create `scripts/db.sh`:

```bash
#!/bin/bash

# Connect to user database
connect_user() {
    docker exec -it dtms_postgres_user_1 psql -U bittu -d users
}

# Connect to task database
connect_task() {
    docker exec -it dtms_postgres_task_1 psql -U bittu -d tasks
}

# Reset database
reset_user() {
    docker exec -it dtms_postgres_user_1 psql -U bittu -d users -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
    docker exec -it dtms_postgres_user_1 psql -U bittu -d users -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"
}

# Show tables
show_tables() {
    echo "User database tables:"
    docker exec -it dtms_postgres_user_1 psql -U bittu -d users -c "\dt"
    
    echo "Task database tables:"
    docker exec -it dtms_postgres_task_1 psql -U bittu -d tasks -c "\dt"
}

# Import data
import_data() {
    local file=$1
    local db=$2
    
    if [ -z "$file" ] || [ -z "$db" ]; then
        echo "Usage: $0 <file> <database>"
        return 1
    fi
    
    docker exec -i "dtms_postgres_${db}_1" psql -U bittu -d "$db" < "$file"
}

case "$1" in
    user)
        connect_user
        ;;
    task)
        connect_task
        ;;
    reset_user)
        reset_user
        ;;
    reset_task)
        reset_task
        ;;
    tables)
        show_tables
        ;;
    import)
        import_data "$2" "$3"
        ;;
    *)
        echo "Usage: $0 {user|task|reset_user|reset_task|tables|import <file> <database>}"
        exit 1
        ;;
esac
```

## Debugging

### Common Debugging Commands

```bash
# View service logs
docker-compose logs -f user_service
docker-compose logs -f task_service
docker-compose logs -f dtms

# Check running containers
docker ps

# Execute shell in container
docker exec -it dtms_user_service_1 sh

# Check network
docker network ls
docker network inspect dtms_service_net

# Port mapping check
docker port dtms_user_service_1

# Resource usage
docker stats

# Database connection test
docker exec -it dtms_postgres_user_1 pg_isready -U bittu
```

### Debugging with Delve

Add to your service Dockerfile for debugging:

```dockerfile
FROM golang:1.22.5-alpine AS builder

# Install delve
RUN go install github.com/go-delve/delve/cmd/dlv@latest

# ... rest of the build

# For debug mode
FROM alpine:latest

RUN apk --no-cache add ca-certificates
COPY --from=builder /go/bin/dlv /usr/local/bin/
# ... copy other files

# Debug entrypoint
CMD ["dlv", "--listen=:40000", "--headless=true", "--api-version=2", "exec", "./user_service"]
```

### Profiling

Add profiling endpoints to your services:

```go
// In main.go
import _ "net/http/pprof"

// Add to main function
go func() {
	log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

Access profiles at:
- http://localhost:6060/debug/pprof/
- http://localhost:6060/debug/pprof/heap
- http://localhost:6060/debug/pprof/goroutine

## Performance Optimization

### Database Optimization

1. **Add Indexes**:
```go
type User struct {
    gorm.Model
    ID       string `gorm:"type:uuid;primary_key"`
    Username string `gorm:"size:255;not null;unique;index"`
    Email    string `gorm:"size:255;not null;unique;index"`
}

// Add composite index
db.Model(&Task{}).AddIndex("idx_tasks_user_created", "user_id", "created_at")
```

2. **Connection Pooling**:
```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})

// Configure connection pool
sqlDB, _ := db.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

3. **Query Optimization**:
```go
// Use Select to limit columns
db.Select("id, username").Find(&users)

// Use Limit and Offset for pagination
db.Limit(10).Offset(20).Find(&tasks)

// Use Preload for relationships (if added)
db.Preload("Tasks").Find(&users)
```

### gRPC Optimization

1. **Interceptors**:
```go
// Add logging interceptor
func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    log.Printf("Method: %s, Request: %v", info.FullMethod, req)
    resp, err := handler(ctx, req)
    log.Printf("Method: %s, Response: %v", info.FullMethod, resp)
    return resp, err
}

// In main
s := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
)
```

2. **Streaming** (for future use):
```go
// Add to proto file
rpc StreamTasks(StreamTasksRequest) returns (stream Task) {}

// Implement in service
func (s *server) StreamTasks(req *pb.StreamTasksRequest, stream pb.TaskService_StreamTasksServer) error {
    // Stream tasks implementation
}
```

## Coding Standards

### Go Style Guide

1. **Package Naming**: Use short, lowercase names
2. **Exported Names**: Use PascalCase for public
3. **Error Handling**: Always check and handle errors
4. **Comments**: Exported functions must have comments
5. **Formatting**: Use `gofmt` and `golint`

### Example Code Style

```go
// UserService handles user-related operations
type UserService struct {
    repo UserRepository
    logger Logger
}

// CreateUser creates a new user with the given details
// It returns the created user or an error if creation fails
func (s *UserService) CreateUser(ctx context.Context, username, email string) (*User, error) {
    // Validate input
    if username == "" {
        return nil, ErrEmptyUsername
    }
    if email == "" {
        return nil, ErrEmptyEmail
    }
    
    // Check if user exists
    exists, err := s.repo.ExistsByUsername(ctx, username)
    if err != nil {
        return nil, fmt.Errorf("failed to check user existence: %w", err)
    }
    if exists {
        return nil, ErrUserAlreadyExists
    }
    
    // Create user
    user := &User{
        ID:       uuid.New().String(),
        Username: username,
        Email:    email,
    }
    
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }
    
    return user, nil
}
```

### Pre-commit Hooks

Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/golangci/golangci-lint
    rev: v1.51.2
    hooks:
      - id: golangci-lint
        args: [--timeout=5m]
```

Install pre-commit:
```bash
pip install pre-commit
pre-commit install
```

## CI/CD Pipeline

### GitHub Actions

Create `.github/workflows/ci.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.22.5'
    
    - name: Cache Go modules
      uses: actions/cache@v3
      with:
        path: ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        restore-keys: |
          ${{ runner.os }}-go-
    
    - name: Install dependencies
      run: go mod download
    
    - name: Run tests
      run: go test -v -race -coverprofile=coverage.out ./...
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out
    
    - name: Build
      run: go build -v ./...

  docker:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          bit-web24/dtms:latest
          bit-web24/dtms:${{ github.sha }}
```

## Contribution Workflow

### 1. Create Feature Branch

```bash
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
```

### 2. Make Changes

- Follow coding standards
- Write tests for new functionality
- Update documentation

### 3. Run Quality Checks

```bash
# Format code
go fmt ./...

# Lint
golangci-lint run

# Run tests
go test ./...

# Build
make build
```

### 4. Commit Changes

```bash
git add .
git commit -m "feat: add project service implementation

- Add project proto definition
- Implement project CRUD operations
- Add database model and migrations
- Update docker-compose configuration

Closes #123"
```

### 5. Push and Create PR

```bash
git push origin feature/your-feature-name
```

Create a pull request with:
- Clear description
- Test steps
- Related issues

### 6. Code Review

- Address reviewer comments
- Update based on feedback
- Keep PR up to date

### 7. Merge

- Squash commits
- Merge to develop
- Delete feature branch

## Resources

### Documentation
- [Go Documentation](https://golang.org/doc/)
- [gRPC Documentation](https://grpc.io/docs/)
- [GORM Documentation](https://gorm.io/docs/)
- [Docker Documentation](https://docs.docker.com/)

### Tools
- [golangci-lint](https://golangci-lint.run/)
- [Delve Debugger](https://github.com/go-delve/delve)
- [pprof](https://golang.org/pkg/net/http/pprof/)

### Best Practices
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Effective Go](https://golang.org/doc/effective_go)
- [gRPC Best Practices](https://grpc.io/blog/grpc-best-practices/)