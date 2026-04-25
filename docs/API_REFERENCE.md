# DTMS API Reference Guide

## Overview

This document provides a comprehensive reference for the DTMS (Distributed Task Management System) REST API. The API follows RESTful conventions and is served through the gRPC-Gateway at port 8080.

## Base URL
```
http://localhost:8080
```

## Authentication

Currently, the API does not implement authentication. This is a planned feature for future releases.

## Content Type

All requests must use the `Content-Type: application/json` header.

## Response Format

All API responses follow a consistent JSON format:

### Success Response
```json
{
  "field": "value"
}
```

### Error Response
```json
{
  "error": "error message",
  "code": "ERROR_CODE"
}
```

## Status Codes

- `200 OK`: Request successful
- `201 Created`: Resource created successfully
- `400 Bad Request`: Invalid request body
- `404 Not Found`: Resource not found
- `500 Internal Server Error`: Server error

---

## User Service API

### Endpoints
- Base path: `/v1/users`

### 1. Create User

Creates a new user with the provided username and email.

**Endpoint:** `POST /v1/users`

**Request Body:**
```json
{
  "username": "string",
  "email": "string"
}
```

**Parameters:**
- `username` (required): Unique username for the user (max 255 characters)
- `email` (required): Unique email address for the user (max 255 characters)

**Example Request:**
```bash
curl -X POST http://localhost:8080/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "email": "john@example.com"
  }'
```

**Example Response (201):**
```json
{
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "john_doe",
    "email": "john@example.com"
  }
}
```

**Error Responses:**
- `400 Bad Request`: Missing or invalid fields
- `500 Internal Server Error`: Database constraint violation

---

### 2. Get User

Retrieves a user by their unique ID.

**Endpoint:** `GET /v1/users/{id}`

**Path Parameters:**
- `id` (required): UUID of the user

**Example Request:**
```bash
curl http://localhost:8080/v1/users/550e8400-e29b-41d4-a716-446655440000
```

**Example Response (200):**
```json
{
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "john_doe",
    "email": "john@example.com"
  }
}
```

**Error Responses:**
- `404 Not Found`: User with specified ID does not exist

---

### 3. Delete User

Deletes a user by their unique ID.

**Endpoint:** `DELETE /v1/users/{id}`

**Path Parameters:**
- `id` (required): UUID of the user to delete

**Example Request:**
```bash
curl -X DELETE http://localhost:8080/v1/users/550e8400-e29b-41d4-a716-446655440000
```

**Example Response (200):**
```json
{
  "success": true
}
```

**Error Responses:**
- `404 Not Found`: User with specified ID does not exist

---

### 4. List All Users

Retrieves a list of all users in the system.

**Endpoint:** `GET /v1/users`

**Example Request:**
```bash
curl http://localhost:8080/v1/users
```

**Example Response (200):**
```json
{
  "users": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "username": "john_doe",
      "email": "john@example.com"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "username": "jane_smith",
      "email": "jane@example.com"
    }
  ]
}
```

---

## Task Service API

### Endpoints
- Base path: `/v1/tasks`

### 1. Create Task

Creates a new task with title, description, and optional user assignment.

**Endpoint:** `POST /v1/tasks`

**Request Body:**
```json
{
  "title": "string",
  "description": "string",
  "user_id": "string"
}
```

**Parameters:**
- `title` (required): Title of the task
- `description` (required): Detailed description of the task (max 255 characters)
- `user_id` (optional): UUID of the user to assign the task to

**Example Request:**
```bash
curl -X POST http://localhost:8080/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Complete documentation",
    "description": "Write comprehensive API documentation",
    "user_id": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

**Example Response (201):**
```json
{
  "task": {
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "title": "Complete documentation",
    "description": "Write comprehensive API documentation",
    "user_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Error Responses:**
- `400 Bad Request`: Missing or invalid fields
- `500 Internal Server Error`: Database error

---

### 2. Get Task

Retrieves a task by its unique ID.

**Endpoint:** `GET /v1/tasks/{id}`

**Path Parameters:**
- `id` (required): UUID of the task

**Example Request:**
```bash
curl http://localhost:8080/v1/tasks/550e8400-e29b-41d4-a716-446655440002
```

**Example Response (200):**
```json
{
  "task": {
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "title": "Complete documentation",
    "description": "Write comprehensive API documentation",
    "user_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Error Responses:**
- `404 Not Found`: Task with specified ID does not exist

---

### 3. Delete Task

Deletes a task by its unique ID.

**Endpoint:** `DELETE /v1/tasks/{id}`

**Path Parameters:**
- `id` (required): UUID of the task to delete

**Example Request:**
```bash
curl -X DELETE http://localhost:8080/v1/tasks/550e8400-e29b-41d4-a716-446655440002
```

**Example Response (200):**
```json
{
  "success": true
}
```

**Error Responses:**
- `404 Not Found`: Task with specified ID does not exist

---

### 4. List All Tasks

Retrieves a list of all tasks in the system.

**Endpoint:** `GET /v1/tasks`

**Example Request:**
```bash
curl http://localhost:8080/v1/tasks
```

**Example Response (200):**
```json
{
  "tasks": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "title": "Complete documentation",
      "description": "Write comprehensive API documentation",
      "user_id": "550e8400-e29b-41d4-a716-446655440000"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440003",
      "title": "Fix bug",
      "description": "Fix authentication issue",
      "user_id": "550e8400-e29b-41d4-a716-446655440001"
    }
  ]
}
```

---

## Health Check API

### Service Health Checks

Individual services expose health check endpoints:

#### User Service Health
**Endpoint:** `http://localhost:8081/health`

**Example Response:**
```
OK
```

#### Task Service Health
**Endpoint:** `http://localhost:8082/health`

**Example Response:**
```
OK
```

---

## gRPC Service API

For direct gRPC communication, the services expose the following:

### User Service
- **Address:** `localhost:50051`
- **Proto Package:** `user`

### Task Service
- **Address:** `localhost:50052`
- **Proto Package:** `task`

### Service Definitions

See the proto files for complete gRPC service definitions:
- `proto/user.proto`
- `proto/task.proto`

---

## Error Handling

### Common Error Formats

#### Validation Error
```json
{
  "error": "Invalid request: username is required",
  "code": "INVALID_ARGUMENT"
}
```

#### Not Found Error
```json
{
  "error": "User not found",
  "code": "NOT_FOUND"
}
```

#### Database Error
```json
{
  "error": "Database constraint violation",
  "code": "INTERNAL"
}
```

### Error Codes

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| INVALID_ARGUMENT | 400 | Request validation failed |
| NOT_FOUND | 404 | Resource not found |
| INTERNAL | 500 | Internal server error |

---

## Rate Limiting

Currently, the API does not implement rate limiting. This is planned for future releases.

---

## Pagination

Currently, list endpoints return all records. Pagination is planned for future releases.

---

## Example Workflows

### Workflow 1: Create User and Assign Task

```bash
# 1. Create a user
USER_RESPONSE=$(curl -s -X POST http://localhost:8080/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "email": "alice@example.com"
  }')

# Extract user ID
USER_ID=$(echo $USER_RESPONSE | jq -r '.user.id')

# 2. Create a task for the user
curl -X POST http://localhost:8080/v1/tasks \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"Review code\",
    \"description\": \"Review pull request #123\",
    \"user_id\": \"$USER_ID\"
  }"
```

### Workflow 2: List All Tasks for a User

```bash
# Get all tasks
TASKS_RESPONSE=$(curl -s http://localhost:8080/v1/tasks)

# Filter tasks by user_id (client-side filtering)
USER_ID="550e8400-e29b-41d4-a716-446655440000"
echo $TASKS_RESPONSE | jq --arg user_id "$USER_ID" '.tasks[] | select(.user_id == $user_id)'
```

---

## OpenAPI/Swagger

The gRPC-Gateway automatically generates OpenAPI documentation. Access it at:
```
http://localhost:8080/swagger.json
```

You can use tools like Swagger UI or Postman to explore the API using this specification.

---

## SDK Examples

### JavaScript (Node.js) Example

```javascript
const axios = require('axios');

const API_BASE = 'http://localhost:8080';

class DTMSClient {
  constructor(baseURL = API_BASE) {
    this.client = axios.create({ baseURL });
  }

  // User operations
  async createUser(username, email) {
    const response = await this.client.post('/v1/users', { username, email });
    return response.data.user;
  }

  async getUser(userId) {
    const response = await this.client.get(`/v1/users/${userId}`);
    return response.data.user;
  }

  async deleteUser(userId) {
    const response = await this.client.delete(`/v1/users/${userId}`);
    return response.data.success;
  }

  async getAllUsers() {
    const response = await this.client.get('/v1/users');
    return response.data.users;
  }

  // Task operations
  async createTask(title, description, userId) {
    const response = await this.client.post('/v1/tasks', { 
      title, 
      description, 
      user_id: userId 
    });
    return response.data.task;
  }

  async getTask(taskId) {
    const response = await this.client.get(`/v1/tasks/${taskId}`);
    return response.data.task;
  }

  async deleteTask(taskId) {
    const response = await this.client.delete(`/v1/tasks/${taskId}`);
    return response.data.success;
  }

  async getAllTasks() {
    const response = await this.client.get('/v1/tasks');
    return response.data.tasks;
  }
}

// Usage example
async function main() {
  const client = new DTMSClient();
  
  // Create a user
  const user = await client.createUser('bob', 'bob@example.com');
  console.log('Created user:', user);
  
  // Create a task
  const task = await client.createTask(
    'Write tests', 
    'Write unit tests for user service',
    user.id
  );
  console.log('Created task:', task);
  
  // List all tasks
  const tasks = await client.getAllTasks();
  console.log('All tasks:', tasks);
}

main().catch(console.error);
```

### Python Example

```python
import requests
import json

class DTMSClient:
    def __init__(self, base_url='http://localhost:8080'):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({'Content-Type': 'application/json'})
    
    def create_user(self, username, email):
        response = self.session.post(
            f'{self.base_url}/v1/users',
            json={'username': username, 'email': email}
        )
        response.raise_for_status()
        return response.json()['user']
    
    def get_user(self, user_id):
        response = self.session.get(f'{self.base_url}/v1/users/{user_id}')
        response.raise_for_status()
        return response.json()['user']
    
    def create_task(self, title, description, user_id=None):
        data = {'title': title, 'description': description}
        if user_id:
            data['user_id'] = user_id
        
        response = self.session.post(f'{self.base_url}/v1/tasks', json=data)
        response.raise_for_status()
        return response.json()['task']
    
    def get_all_tasks(self):
        response = self.session.get(f'{self.base_url}/v1/tasks')
        response.raise_for_status()
        return response.json()['tasks']

# Usage example
def main():
    client = DTMSClient()
    
    # Create user
    user = client.create_user('charlie', 'charlie@example.com')
    print(f"Created user: {user}")
    
    # Create task
    task = client.create_task(
        'Deploy to production',
        'Deploy latest changes to production',
        user['id']
    )
    print(f"Created task: {task}")
    
    # List all tasks
    tasks = client.get_all_tasks()
    print(f"All tasks: {json.dumps(tasks, indent=2)}")

if __name__ == '__main__':
    main()
```

---

## Testing

### Unit Testing Example (JavaScript)

```javascript
const axios = require('axios');
const { expect } = require('chai');

describe('DTMS API', () => {
  const client = axios.create({ baseURL: 'http://localhost:8080' });
  
  describe('User Service', () => {
    it('should create a user', async () => {
      const response = await client.post('/v1/users', {
        username: 'test_user',
        email: 'test@example.com'
      });
      
      expect(response.status).to.equal(201);
      expect(response.data.user).to.have.property('id');
      expect(response.data.user.username).to.equal('test_user');
    });
    
    it('should get a user', async () => {
      // First create a user
      const createRes = await client.post('/v1/users', {
        username: 'test_user2',
        email: 'test2@example.com'
      });
      
      const userId = createRes.data.user.id;
      
      // Then get the user
      const getRes = await client.get(`/v1/users/${userId}`);
      
      expect(getRes.status).to.equal(200);
      expect(getRes.data.user.id).to.equal(userId);
    });
  });
});
```

### Load Testing (Apache Bench)

```bash
# Test create user endpoint
ab -n 1000 -c 10 -p user.json -T application/json http://localhost:8080/v1/users

# user.json content:
# {"username":"load_test","email":"load@test.com"}
```