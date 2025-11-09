# Real-time Leaderboard (Backend) Specification

## Overview

This document describes the architecture and technical specifications for implementing a real-time updated leaderboard feature.

## Objectives

1. Display top 10 users by score in real-time
2. Update scores based on completed user actions
3. Prevent malicious score manipulation
4. Ensure low latency for live updates
5. Scale to handle high concurrent users

## Architecture

This architecture uses events that separate the real-time process of updating scores and distributing data to clients.

### Tech Stack

1. **Backend - API**

    Uses the NodeJs framework (Express) with typescript standard

2. **Database**

    Uses MongoDB as primary database. As a permanent data storage and source of truth.

3. **Caching**

    Uses Redis. The scoreboard (top 10) is stored in a Redis Sorted Set for fast retrieval, reducing load on the main database.

4. **Message Broker**

    Serves as a reliable message broker. When a user’s score updates, a message is published to RabbitMQ. This ensures all API service instances receive update notifications.

5. **Real-time Communication**

    Uses WebSocket (socket.io) for persistent two-way communication between clients (browsers) and servers. This enables real-time scoreboard updates without requiring clients to poll.

6. Authentication

    Uses JWT (JSON Web Tokens)


### Diagram

High Level Diagram

![flow.png](https://res.cloudinary.com/dbp0861hz/image/upload/v1762693748/flow_n8atqn.png)

## API Specification

### 1. Update Score

Increments the user's score when a valid action is completed.

**Endpoint:**

**`POST /api/v1/scores/update`**

**Headers:**

```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**Request Body**

```json
{
  "actionId": "string", // unique uuid
  "actionToken": "string", // Autogenere by backend
  "timestamp": "date",
}

```

**Response (200 OK)**

```json
{
	"status": 200,
	"data": {
		"userId": "string", // UUID
		"newScore": "integer",
		"scoreIncrement": "integer",
		"leaderboardPosition": "integer",
		"inTopTen": "boolean"
	},
}
```

**Error Responses**:

1. `400 Bad Request`: Invalid request format
2. `401 Unauthorized`: Invalid or expired JWT token
3. `403 Forbidden`: Invalid action signature or token
4. `409 Conflict`: Action already completed
5. `429 Too Many Requests`: Rate limit exceeded
6. `500 Internal Server Error`: Server error

### 2. Get Current Leaderboard

This is public. The main function to get leaderboard list

**Endpoint:**

**`POST /api/v1/leaderboards`**

**Query Parameters:**

1. `limit`: integer (default: 10, max: 50)
- `offset`: integer (default: 0)

**Response (200 OK)**

```json
{
  "status": 200,
  "data": {
    "leaderboard": [
      {
        "rank": 1,
        "userId": "string",
        "username": "string",
        "score": "integer",
        "lastUpdated": "date"
      }
    ],
    "totalUsers": "integer",
    "lastRefreshed": "date"
  }
}
```

### 3. Get User Rank

**Endpoint**

`GET /api/v1/scores/{userId}`

**Response (200 OK)**

```json
{
	"status": 200,
	"data": {
		"userId": "string",
		"username": "string",
		"score": "integer",
		"rank": "integer",
		"inTopTen": "boolean",
		"lastUpdated": "Date"
	}
}
```

### 4. WebSocket Events

**Endpoint/ Connection**

`wss://api.testing.com/ws/leaderboard`

**Client to Server**

```tsx

socket.emit('subscribe', {
  userId: 'string'
});
```

**Server to Client**

```tsx

socket.on('leaderboard:initial', (data) => {
	//
});

socket.on('leaderboard:update', (data) => {
	//
});

socket.on('user:rank_update', (data) => {
	//
});
```

## Security

### 1. Authentication

- User authenticates → Receives JWT token
- JWT token included in all API requests
- Token validated on every request
- Token contains: userId, role, expiration

### 2. Action Token

- User initiates action on frontend
- Frontend requests action token: `POST /api/v1/actions/token`
- Backend generates one-time token with:
    - actionId
    - userId
    - timestamp
    - expiration (5 minutes)
- Backend stores token in Redis: `key="action_token:{token}", value="userId|actionId"`
- User completes action on frontend
- Frontend sends score update request:
- Backend validates
- Backend updates score and deletes token from Redis

### 3. Rate Limiting

- **Strategy**: Token bucket algorithm using Redis
- **Limits**:
    - Score updates: 10 requests per minute per user
    - Action token generation: 20 requests per minute per user
    - Leaderboard retrieval: 60 requests per minute per IP

### 3. Audit Trail

Log all score changes with timestamp, source IP, and token ID for auditing.
