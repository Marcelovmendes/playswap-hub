# Architecture

This document describes the system architecture, service responsibilities, and communication patterns.

##  Overview

PlaySwap follows a **microservices architecture** with asynchronous job processing. Each service has a single responsibility and communicates through well-defined interfaces.



## Service Details

### 1. PlaySwap Frontend

**Repository**: [playswap-front](https://github.com/Marcelovmendes/playswap-front)

**Stack**: Next.js 14, TypeScript, TailwindCSS, Radix UI

**Architecture Patterns**:
- **App Router**: Leverages React Server Components for initial data fetching
- **Client State**: Zustand for authentication state
- **Server State**: TanStack Query for API data caching and real-time polling
- **Form Handling**: React Hook Form with Zod validation

**Key Directories**:
```
src/
├── app/                    # Next.js App Router pages
│   ├── auth/callback/      # Spotify OAuth callback handler
│   ├── auth/youtube/       # YouTube OAuth callback handler
│   └── (auth)/             # Protected routes group
│       ├── dashboard/      # Playlist selection grid
│       ├── playlist/[id]/  # Track preview before conversion
│       └── conversion/[id] # Real-time conversion progress
├── components/             # React components
├── lib/
│   ├── api/               # API client (Axios with interceptors)
│   └── hooks/             # Custom hooks (usePlaylists, useConversionStatus)
└── store/                 # Zustand stores
```

---

### 2. API Gateway

**Stack**: KrakenD 2.7

**Responsibilities**:
- Route aggregation across services
- CORS configuration with credentials support
- Cookie/session propagation to backend services
- Request timeout management (30s default)

**Configuration Structure**:
```
config/
├── krakend.tmpl           # Main gateway configuration
├── templates/
│   ├── auth-spotify-endpoints.tmpl
│   ├── auth-youtube-endpoints.tmpl
│   ├── spotify-playlists-endpoints.tmpl
│   ├── youtube-endpoints.tmpl
│   └── conversion-endpoints.tmpl
└── settings/
    └── settings.tmpl      # Environment variables
```

**Route Mapping**:
| Frontend Path | Backend Service | Backend Path |
|---------------|-----------------|--------------|
| `GET /api/auth/spotify/login` | Spotify Service | `/auth/` |
| `GET /api/playlists` | Spotify Service | `/playlists` |
| `POST /api/conversions` | Spotify Service | `/conversions` |
| `GET /api/conversions/{id}/status` | Spotify Service | `/conversions/{id}/status` |
| `GET /api/auth/youtube/login` | YouTube Service | `/v1/auth/login` |

---

### 3. Spotify Service

**Repository**: [spotify-service](https://github.com/Marcelovmendes/spotify-service)

**Stack**: Java 21, Spring Boot 3.4, Spring Session Redis

**Architecture**: Layered with Rich Domain Model

```
src/main/java/
└── com/playswap/spotify/
    ├── domain/                    # Business logic layer
    │   ├── model/                 # Domain entities
    │   │   ├── Token.java         # OAuth token with expiration logic
    │   │   ├── ConversionJob.java # Conversion job aggregate
    │   │   └── ConversionStatus.java
    │   └── port/                  # Repository interfaces
    │       ├── TokenRepository.java
    │       └── ConversionRepository.java
    │
    ├── application/               # Use cases
    │   ├── AuthUseCase.java       # OAuth flow orchestration
    │   ├── PlaylistUseCase.java   # Playlist retrieval
    │   └── ConversionUseCase.java # Job creation and tracking
    │
    ├── infrastructure/            # Technical implementations
    │   ├── redis/                 # Redis adapters
    │   │   ├── RedisTokenRepository.java
    │   │   └── RedisConversionRepository.java
    │   └── spotify/               # Spotify API client
    │       └── SpotifyApiClient.java
    │
    └── api/                       # REST layer
        ├── controller/
        │   ├── AuthenticationController.java
        │   ├── PlaylistController.java
        │   └── ConversionController.java
        └── dto/
            ├── CreateConversionRequest.java
            └── ConversionStatusResponse.java
```

**Key Flows**:
1. **OAuth with PKCE**: Generates code verifier/challenge, handles callback, stores tokens
2. **Playlist Sync**: Fetches user playlists from Spotify API, caches in session
3. **Job Creation**: Validates playlist, creates job entry, enqueues to Redis

---

### 4. YouTube Service

**Repository**: [youtube-service](https://github.com/Marcelovmendes/youtube-service)

**Stack**: Java 25, Spring Boot 3.5, Google API Client

**Architecture**: Similar layered structure with Result type for error handling

```
src/main/java/
└── com/playswap/youtube/
    ├── domain/
    │   ├── model/
    │   │   ├── YouTubePlaylist.java  # Immutable playlist entity
    │   │   ├── YouTubeVideo.java     # Video with metadata
    │   │   └── VideoId.java          # Value object
    │   └── result/
    │       └── Result.java           # Typed error handling
    │
    ├── application/
    │   ├── PlaylistService.java      # Playlist operations
    │   └── SearchService.java        # Video search
    │
    ├── infrastructure/
    │   ├── google/                   # Google API adapters
    │   └── redis/                    # Session storage
    │
    └── api/
        └── controller/
            └── AuthenticationController.java
```

**Key Features**:
- **Virtual Threads**: Enabled for better I/O concurrency
- **Result Type**: Explicit error handling without exceptions
- **Session Linking**: Links YouTube session to Spotify session for worker access

---

### 5. Conversion Worker

**Repository**: [conversion-worker](https://github.com/Marcelovmendes/conversion-worker)

**Stack**: Go 1.24, pgx (PostgreSQL), go-redis

**Architecture**: Clean Architecture / Ports & Adapters

```
cmd/
└── worker/
    └── main.go                    # Entry point, dependency injection

internal/
├── domain/                        # Core business logic
│   ├── conversion.go              # Conversion aggregate
│   ├── track.go                   # Track entity
│   ├── playlist.go                # Playlist value object
│   └── status.go                  # Status enum
│
├── application/                   # Use cases
│   ├── worker.go                  # Queue consumer loop
│   ├── converter.go               # Conversion orchestrator
│   └── matcher.go                 # Track-to-video matching
│
└── infrastructure/
    ├── redis/
    │   ├── queue.go               # Job queue consumer
    │   ├── status.go              # Real-time status updates
    │   └── session.go             # Token retrieval
    ├── postgres/
    │   ├── conversion_repo.go     # Conversion persistence
    │   └── log_repo.go            # Match logs
    └── http/
        ├── spotify_client.go      # Spotify API calls
        └── youtube_client.go      # YouTube API calls
```
---

## Communication Patterns

### Synchronous (HTTP)
- Frontend → Gateway → Services
- Services → External APIs (Spotify, YouTube)

### Asynchronous (Redis Queue - FIFO)
- Spotify Service enqueues jobs via `LPUSH`
- Worker consumes jobs via `BRPOP` (blocking pop)
- First-In-First-Out processing order
- Status updates stored as Redis keys

### Session Management
```
┌──────────┐     Cookie: SPOTIFY_SESSION=abc123     ┌─────────────────┐
│ Frontend │ ────────────────────────────────────── │ Spotify Service │
└──────────┘                                        └────────┬────────┘
                                                             │
                                                   Stores: abc123 → {token, refresh}
                                                             │
                                                             ▼
                                                     ┌──────────────┐
                                                     │    Redis     │
                                                     └──────────────┘
                                                             │
                                                   Reads: abc123 → {token}
                                                             │
                                                             ▼
                                                    ┌─────────────────┐
                                                    │     Worker      │
                                                    └─────────────────┘
```

---

## Data Models

### Conversion Job (Redis)
```json
{
  "jobId": "uuid",
  "playlistId": "spotify:playlist:xxx",
  "playlistName": "My Playlist",
  "spotifySessionId": "abc123",
  "youtubeSessionId": "xyz789",
  "status": "PROCESSING",
  "totalTracks": 50,
  "processedTracks": 25,
  "createdAt": "2024-01-15T10:30:00Z"
}
```

### Conversion History (PostgreSQL)
```sql
CREATE TABLE conversions (
    id              UUID PRIMARY KEY,
    playlist_id     VARCHAR(255),
    playlist_name   VARCHAR(500),
    total_tracks    INTEGER,
    matched_tracks  INTEGER,
    youtube_playlist_id VARCHAR(255),
    status          VARCHAR(50),
    created_at      TIMESTAMP,
    completed_at    TIMESTAMP
);

CREATE TABLE conversion_logs (
    id              UUID PRIMARY KEY,
    conversion_id   UUID REFERENCES conversions(id),
    track_name      VARCHAR(500),
    artist_name     VARCHAR(500),
    matched_video_id VARCHAR(255),
    match_score     FLOAT,
    created_at      TIMESTAMP
);
```

---

## Scalability Considerations

| Component | Scaling Strategy |
|-----------|-----------------|
| Frontend | CDN + Edge caching (Vercel) |
| Gateway | Horizontal scaling behind load balancer |
| Spotify Service | Stateless, horizontal scaling |
| YouTube Service | Stateless, horizontal scaling |
| Worker | Multiple instances, competing consumers |
| Redis | Redis Cluster for high availability |
| PostgreSQL | Read replicas for query scaling |
