# Plan: Dockerized Offline Caching System with API-Served Cache and CI/CD

Transform the project into a containerized application with a Node.js API server that pre-fetches and serves cached place data, using SQLite for metadata and filesystem storage for photos in both WebP and JPEG formats at multiple resolutions, with Docker secrets management, API quota tracking, and multi-architecture CI/CD pipeline.

## Steps

### 1. Create npm project structure

Create `package.json` with dependencies:
- `express@^4.18` - Web framework for API server
- `better-sqlite3@^9.2` - SQLite database driver
- `node-fetch@^3.3` - HTTP client for Google Places API calls
- `sharp@^0.33` - Image processing library
- `commander@^11.1` - CLI argument parsing
- `cors@^2.8` - CORS middleware
- `multer@^1.4` - File upload handling
- `pino@^8.16` - Logging framework
- `p-queue@^7.4` - Promise queue for concurrency control

Create `server/api.js` - Express server with endpoints:
- Implements WebP/JPEG content negotiation based on Accept header
- Quota tracking middleware incrementing `api_usage` table on each Google API call
- Returns `X-Cache-Status: stale` header with `X-Quota-Reset` timestamp when quota exceeded

Create `server/db.js` - SQLite database schema:
- `places` table: placeId, displayName, formattedAddress, location, svgIconMaskURI, iconBackgroundColor, googleMapsURI, types, cachedAt
- `photos` table: placeId, hasWebP, hasJPEG, sizes (JSON array), cachedAt
- `timeline_files` table: id, filename, format, confidence, uploadedAt, isActive
- `api_usage` table: date, count, quota_limit

Create `server/image-processor.js`:
- Uses `sharp` to generate images at [150, 400, 800, 1200]px widths
- Outputs WebP at quality 85 and JPEG at quality 80
- Maintains aspect ratio for all sizes

Create `server/format-detector.js`:
- Analyzes JSON structure to detect format type (iOS Timeline, Google Takeout, Semantic Segments)
- Returns format type and confidence score (0-100)

Create `scripts/prefetch-places.js`:
- 5-worker pool processing place IDs concurrently
- 200ms minimum delay between requests per worker
- Safe deletion: only removes old data after verified successful re-fetch
- Exponential backoff on errors (429 rate limits, network failures)
- Logs to `/data/logs/prefetch-{timestamp}.log`

### 2. Create Docker configuration

Create `Dockerfile` with multi-stage build:
- **Builder stage**: node:20-slim with build-essential, installs native dependencies for both amd64 and arm64
- **Runtime stage**: node:20-slim + libvips (for sharp) + Caddy binary from caddy:2.7-alpine
- Supervisor script to start Node.js on port 3000 and Caddy on port 8080
- HEALTHCHECK running every 30s: `curl -f http://localhost:3000/health || exit 1`
- Exposed port: 8080

Create `docker-compose.yml`:
- Service `app` with platform specification for multi-arch
- Volumes:
  - `timeline-data:/data` - Persistent storage for cache database, logs, photos
  - `timeline-files:/data/timeline` - Timeline JSON files storage
- Secrets:
  - `google_api_key` loaded from `./secrets/google_api_key.txt`
- Environment variables:
  - `GOOGLE_MAPS_API_KEY_FILE=/run/secrets/google_api_key`
  - `CACHE_TTL_DAYS=180`
  - `DAILY_API_QUOTA=5000`
  - `MAX_CONCURRENT_FETCHES=5`
- Deploy limits: 2GB memory, 2 CPUs
- Restart policy: unless-stopped

Create `.dockerignore`:
- Exclude: node_modules, .git, .vscode, sample_data, *.log, .DS_Store

Create `.env.example`:
- Template for environment variables

Create `secrets/.gitkeep`:
- Placeholder file (actual `secrets/google_api_key.txt` should be in .gitignore)

### 3. Refactor timeline.html removing client-side cache

**Remove existing cache logic:**
- Delete `placeDetailsCache` object definition (~line 1449)
- Remove file save button handler (~line 4046)
- Remove cache load logic (~line 4177)
- Remove IndexedDB functions (~line 4715)

**Add new API functions:**
- `async fetchPlaceFromAPI(placeId)`:
  - Calls `GET /api/cache/${placeId}`
  - Implements retry logic (3 attempts with exponential backoff)
  - Checks response headers for `X-Cache-Status`
  - Displays stale data warning banner when quota exceeded
  - Shows `X-Quota-Reset` date in warning message

- `getPhotoUrl(placeId, size)`:
  - Returns responsive `<picture>` element HTML
  - WebP source for modern browsers
  - JPEG fallback for compatibility
  - Example: `<picture><source type="image/webp" srcset="/api/photo/${placeId}/${size}"><img src="/api/photo/${placeId}/${size}" loading="lazy"></picture>`

**Update existing functions:**
- Modify `fetchPlaceDetails()` (~line 2183) to use `fetchPlaceFromAPI()`
- Update `fetchPhotoForInfoWindow()` (~line 2477) to use `getPhotoUrl()`

**Add new UI components:**
- Timeline selector dropdown:
  - Populated from `GET /api/timeline/list`
  - onChange calls `POST /api/timeline/select` with selected ID
  
- File upload button with drag-drop:
  - Posts to `POST /api/upload` with FormData
  - Shows format detection confidence score
  - Prompts user to confirm/override if confidence < 80%
  - Displays upload progress bar

- "Pre-fetch All" modal:
  - EventSource connection to `GET /api/prefetch/progress` (Server-Sent Events)
  - Real-time display:
    - Places fetched / total places
    - Current place being processed
    - Estimated time remaining
    - API quota used / remaining
    - Storage used (MB)
  - Controls: Pause, Resume, Cancel buttons

### 4. Configure Caddy and Service Worker

Create `Caddyfile`:
```
:8080 {
    root * /app/public
    file_server
    
    reverse_proxy /api/* localhost:3000 {
        flush_interval -1
    }
    
    encode gzip
    
    header /api/photo/* Cache-Control "public, max-age=2592000"
    
    log {
        output file /data/logs/access.log
        format json
    }
}
```

Create `public/sw.js` (Service Worker):
- Cache version: `v1`
- **Precaching**: 
  - Cache `timeline.html`
  - Cache CDN libraries (moment.js, moment-timezone.js URLs)
- **Strategies**:
  - Network-first with 3s timeout for `/api/cache/*`:
    - Fallback to Cache API on timeout/failure
    - Add stale indicator to UI when serving from cache
  - Cache-first with stale-while-revalidate for `/api/photo/*`:
    - Serve from cache immediately
    - Update cache in background
- **Offline support**:
  - Offline fallback page at `/offline.html`
  - Shows count of cached places
  - Lists recently viewed places
- **Updates**:
  - Notification when new service worker available
  - Prompt user to refresh page

### 5. Create GitHub Actions CI/CD

Create `.github/workflows/docker-build.yml`:

**Triggers:**
- Push to main branch
- Push to tags matching `v*`
- Pull requests

**Jobs:**

1. **test**:
   - Runs `npm test`
   - Validates API endpoints
   - Checks SQLite schema
   - Tests image processing

2. **security-scan**:
   - Runs Snyk container scan per `.github/instructions/snyk_rules.instructions.md`
   - Fails on high severity vulnerabilities
   - Uploads results to GitHub Security tab

3. **build-and-push**:
   - Uses QEMU for multi-architecture emulation
   - Uses Docker buildx for multi-platform builds
   - Platforms: linux/amd64, linux/arm64
   - Caches layers to ghcr.io for faster builds
   - Pushes to `ghcr.io/kurupted/google-maps-timeline-viewer`
   - Tags:
     - `latest` (on push to main)
     - Semver from package.json version (e.g., `v1.0.0`)
     - Git SHA (e.g., `sha-abc1234`)
   - Build arguments:
     - `GIT_SHA` - Git commit SHA
     - `VERSION` - Version from package.json
   - Attestation with `provenance: true`

**Update README.md with comprehensive documentation:**

#### Quick Start
- Prerequisites (Docker, Docker Compose)
- Create `secrets/google_api_key.txt` with API key
- Run `docker-compose up -d`
- Access at `http://localhost:8080`

#### Architecture
- Diagram showing: Browser → Caddy → Node.js API → SQLite/Filesystem
- Component descriptions
- Data flow explanation

#### API Endpoints
Table with columns: Method, Path, Description, Example

#### Configuration
Environment variables table with defaults and descriptions

#### Production Deployment
- Docker Compose example with nginx reverse proxy
- HTTPS setup with Let's Encrypt
- Rate limiting configuration
- Security best practices

#### Volume Management
Backup command:
```bash
docker run --rm -v timeline-data:/data -v $(pwd):/backup ubuntu tar czf /backup/timeline-backup.tar.gz /data
```

Restore command:
```bash
docker run --rm -v timeline-data:/data -v $(pwd):/backup ubuntu tar xzf /backup/timeline-backup.tar.gz -C /
```

#### Troubleshooting
- Quota exceeded errors
- Format detection issues
- Log file locations
- Common error messages and solutions

#### Contributing
- Local development setup (without Docker)
- Running tests
- Code style guidelines
- Submitting pull requests

## Implementation Notes

- All code should include Copilot generation comments as per instruction files
- Multi-architecture Docker builds as required by instruction files
- Security scanning with Snyk as per `.github/instructions/snyk_rules.instructions.md`
- No emojis in code per instruction files
- All container build scripts support both x86 and ARM architectures
- Comprehensive error handling with detailed logging
- API rate limiting respects Google Places API free tier limits
- 6-month cache expiration with safe deletion after successful re-fetch
