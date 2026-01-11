# Getting Started

This guide walks you through setting up PlaySwap locally for development.

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Java | 21+ | Backend services |
| Go | 1.24+ | Conversion worker |
| Node.js | 18+ | Frontend |
| Docker | 20+ | Infrastructure (Redis, PostgreSQL) |
| Maven | 3.9+ | Java build tool |

## API Credentials

You'll need developer accounts for both platforms:

### Spotify API
1. Go to [Spotify Developer Dashboard](https://developer.spotify.com/dashboard)
2. Create a new application
3. Add `http://localhost:8080/auth/callback` to Redirect URIs
4. Copy your **Client ID** and **Client Secret**

### YouTube Data API
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project
3. Enable **YouTube Data API v3**
4. Create OAuth 2.0 credentials (Web application)
5. Add `http://localhost:8081/v1/auth/callback` to Authorized redirect URIs
6. Copy your **Client ID** and **Client Secret**

---

## Infrastructure Setup

Start Redis and PostgreSQL using Docker:

```bash
# Create a docker-compose.yml in your workspace
cat > docker-compose.yml << 'EOF'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: playswap
      POSTGRES_PASSWORD: playswap
      POSTGRES_DB: playswap
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  redis_data:
  postgres_data:
EOF

# Start services
docker-compose up -d
```

Verify services are running:
```bash
docker-compose ps
```

---

## Service Setup

### 1. Spotify Service

```bash
cd spotify-service

# Create application.yml or set environment variables
export SPOTIFY_CLIENT_ID=your_client_id
export SPOTIFY_CLIENT_SECRET=your_client_secret
export SPOTIFY_REDIRECT_URI=http://localhost:8080/auth/callback
export SPRING_DATA_REDIS_HOST=localhost
export SPRING_DATA_REDIS_PORT=6379

# Run the service
./mvnw spring-boot:run
```

The service will start on **port 8080**.

### 2. YouTube Service

```bash
cd youtube-service

# Set environment variables
export GOOGLE_CLIENT_ID=your_google_client_id
export GOOGLE_CLIENT_SECRET=your_google_client_secret
export GOOGLE_REDIRECT_URI=http://localhost:8081/v1/auth/callback
export SPRING_DATA_REDIS_HOST=localhost
export SPRING_DATA_REDIS_PORT=6379

# Run the service
./mvnw spring-boot:run
```

The service will start on **port 8081**.

### 3. Conversion Worker

```bash
cd conversion-worker

# Set environment variables
export REDIS_URL=localhost:6379
export POSTGRES_URL=postgres://playswap:playswap@localhost:5432/playswap?sslmode=disable
export SPOTIFY_API_URL=https://api.spotify.com
export YOUTUBE_API_URL=https://www.googleapis.com/youtube/v3

# Run the worker
go run cmd/worker/main.go
```

### 4. API Gateway

```bash
cd api-gateway

# Start with Docker
docker-compose up
```

Or run KrakenD directly:
```bash
export SPOTIFY_SERVICE_URL=http://localhost:8080
export YOUTUBE_SERVICE_URL=http://localhost:8081
export FRONTEND_URL=http://localhost:3000

krakend run -c config/krakend.tmpl
```

The gateway will start on **port 8000**.

### 5. Frontend

```bash
cd playswap-front

# Install dependencies
npm install

# Set environment variables
export NEXT_PUBLIC_GATEWAY_URL=http://localhost:8000

# Run development server
npm run dev
```

The frontend will start on **port 3000**.

---

## Verification

Once all services are running, verify the setup:

1. Open http://localhost:3000
2. Click "Login with Spotify"
3. Authorize the application
4. You should see your Spotify playlists

### Health Checks

| Service | Health Endpoint |
|---------|-----------------|
| Spotify Service | http://localhost:8080/actuator/health |
| YouTube Service | http://localhost:8081/actuator/health |
| API Gateway | http://localhost:8000/__health |

---

## Service Ports Summary

| Service | Port | Purpose |
|---------|------|---------|
| Frontend | 3000 | User interface |
| API Gateway | 8000 | Request routing |
| Spotify Service | 8080 | Spotify integration |
| YouTube Service | 8081 | YouTube integration |
| Redis | 6379 | Session/Queue |
| PostgreSQL | 5432 | Data persistence |

---

## Troubleshooting

### Redis Connection Refused
```bash
# Check if Redis is running
docker-compose ps redis

# View Redis logs
docker-compose logs redis
```

### Spotify OAuth Error
- Verify redirect URI matches exactly (including trailing slashes)
- Check client ID and secret are correct
- Ensure the Spotify app is not in development mode restrictions

### YouTube OAuth Error
- Verify OAuth consent screen is configured
- Add your email to test users if app is in testing mode
- Check redirect URI matches exactly

### CORS Issues
- Ensure API Gateway is running
- Verify FRONTEND_URL environment variable is set correctly
- Check browser console for specific CORS error messages

### Worker Not Processing Jobs
```bash
# Check Redis queue
redis-cli LLEN conversion:queue

# View worker logs
# (worker outputs to stdout)
```

---

## Development Tips

### Running Tests

```bash
# Spotify Service
cd spotify-service && ./mvnw test

# YouTube Service
cd youtube-service && ./mvnw test

# Conversion Worker
cd conversion-worker && go test ./...

# Frontend
cd playswap-front && npm test
```

### Hot Reload

- **Java Services**: Use Spring DevTools (included in dependencies)
- **Go Worker**: Use `air` for live reload: `go install github.com/cosmtrek/air@latest`
- **Frontend**: Built-in with Next.js dev server

### Debugging

Enable debug logging:

```bash
# Java services
export LOGGING_LEVEL_ROOT=DEBUG

# Go worker
export LOG_LEVEL=debug
```
