# PlaySwap

**Convert your Spotify playlists to YouTube with a single click.**

PlaySwap is a distributed system that enables seamless playlist migration between Spotify and YouTube. Built with microservices architecture, it demonstrates modern software engineering practices including domain-driven design, event-driven processing, and cloud-native patterns.

## Quick Links

| Document | Description |
|----------|-------------|
| [Architecture](docs/ARCHITECTURE.md) | System design, service communication, and data flow |
| [Getting Started](docs/GETTING_STARTED.md) | How to run the project locally |

---

## Overview

### The Problem

Music lovers often have playlists on Spotify they'd like to access on YouTube (for music videos, live performances, or sharing). Manually recreating playlists is tedious and time-consuming.

### The Solution

PlaySwap automates this process:
1. Authenticate with your Spotify account
2. Select a playlist to convert
3. Link your YouTube account
4. Watch your playlist being created in real-time

---

## System Architecture

![PlaySwap Architecture](docs/architecture.png)

---

## Tech Stack

| Layer | Technology                          | Purpose |
|-------|-------------------------------------|---------|
| **Frontend** | Next.js 14, TypeScript, TailwindCSS | Modern React with server components |
| **Gateway** | KrakenD 2.7                         | API routing, CORS, request aggregation |
| **Backend** | Java 21/25, Spring Boot 3.4         | OAuth flows, business logic, API integration |
| **Worker** | Go 1.24                             | High-throughput background job processing |
| **Cache/Queue** | Redis 7                             | Session storage, job queue, real-time status |
| **Database** | PostgreSQL 16                       | Conversion history persistence |

---

## Key Features

### For Users
- One-click Spotify playlist conversion to YouTube
- Real-time conversion progress tracking
- Track-by-track matching visualization
- Secure OAuth 2.0 authentication (no passwords stored)

### Technical Highlights
- **Microservices Architecture**: Independent, scalable services
- **Asynchronous Processing**: Queue-based job distribution
- **Rich Domain Model**: Business logic in domain entities
- **Type Safety**: Strong typing across all layers
- **Immutable Design**: Predictable state management

---

## Repositories

| Service | Description | Stack                  |
|---------|-------------|------------------------|
| [spotify-service](https://github.com/Marcelovmendes/spotify-service) | Spotify OAuth + playlist management | Java 21, Spring Boot   |
| [youtube-service](https://github.com/Marcelovmendes/youtube-service) | YouTube OAuth + playlist creation | Java 25, Spring Boot   |
| [conversion-worker](https://github.com/Marcelovmendes/conversion-worker) | Background job processor | Go 1.24                |
| [playswap-front](https://github.com/Marcelovmendes/playswap-front) | User interface | Next.js 14, TypeScript |

---

## Running Locally

See the [Getting Started Guide](docs/GETTING_STARTED.md) for detailed instructions.

**Quick start:**
```bash
# 1. Start infrastructure
docker-compose up -d redis postgres

# 2. Start services (in separate terminals)
cd spotify-service && ./mvnw spring-boot:run
cd youtube-service && ./mvnw spring-boot:run
cd conversion-worker && go run cmd/worker/main.go
cd api-gateway && docker-compose up
cd playswap-front && npm run dev
```

---

## API Limitations (Development Mode)

This project runs on development API keys, which impose restrictions unrelated to the code itself:

### YouTube Data API
- **Daily quota**: ~10,000 units/day (each search costs 100 units)
- **Practical limit**: ~90 track searches per day
- **Test users**: Must be manually added in Google Cloud Console to authenticate

### Spotify API
- **User limit**: Maximum 25 users in development mode
- **Production access**: Current policies require ~250k monthly active listeners on the platform, making it effectively impossible for independent developers

These are platform restrictions, not application limitations.

---

## License

This project is for educational and portfolio purposes.

---

## Author

**Marcelo Mendes** - [GitHub](https://github.com/Marcelovmendes)
