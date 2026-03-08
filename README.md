# ✅ Task Manager — Docker Advanced Lab 7

A production-ready full-stack Task Manager application containerized with Docker, featuring load balancing, SSL termination, caching, and health monitoring.

---

## 🏗️ Architecture

```
Internet
    │
    ▼
[ Nginx :443 ] ──── SSL Termination + Load Balancer (least_conn)
    │                        │
    ├────────────────────────┤
    ▼                        ▼
[ Flask1 :8000 ]      [ Flask2 :8000 ]     ← Gunicorn WSGI Workers
    │                        │
    └──────────┬─────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
[ PostgreSQL :5432 ]  [ Redis :6379 ]
  (Persistent DB)     (Cache Layer)
```

| Service    | Image               | Role                          |
|------------|---------------------|-------------------------------|
| `nginx`    | `nginx:alpine`      | Reverse proxy + Load balancer |
| `flask1`   | custom (Dockerfile) | App server — Worker 1         |
| `flask2`   | custom (Dockerfile) | App server — Worker 2         |
| `postgres` | `postgres:16-alpine`| Relational database           |
| `redis`    | `redis:7-alpine`    | In-memory cache               |

---

## 📁 Project Structure

```
task-manager-docker/
├── Dockerfile                 # Multi-stage build (builder + runtime)
├── docker-compose.yml         # 5-service stack definition
├── flask_app.py               # Flask REST API (provided)
├── init.sql                   # DB schema + seed data (provided)
├── requirements.txt           # Python dependencies (provided)
├── .env                       # Secrets — DO NOT commit to Git
├── .dockerignore              # Excludes .env and ssl/ from image
├── conf/
│   └── nginx.conf             # Upstream LB + SSL + rate limiting
├── ssl/
│   ├── generate_ssl.sh        # Self-signed certificate generator
│   └── certs/
│       ├── server.crt
│       └── server.key
└── static/
    └── style.css              # Frontend styles (provided)
```

---

## 🚀 Getting Started

### Prerequisites

- Docker Engine 24+
- Docker Compose v2+
- `openssl` (for SSL certificate generation)

### 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/task-manager-docker.git
cd task-manager-docker
```

### 2. Configure Environment Variables

Create a `.env` file in the project root (never commit this file):

```env
POSTGRES_DB=taskmanager
POSTGRES_USER=taskuser
POSTGRES_PASSWORD=your_secure_password
DATABASE_URL=postgresql://taskuser:your_secure_password@postgres:5432/taskmanager
REDIS_URL=redis://redis:6379/0
```

### 3. Generate SSL Certificate

```bash
cd ssl
bash generate_ssl.sh
cd ..
```

### 4. Build and Start All Services

```bash
docker compose up -d --build
```

> First run takes 3–5 minutes while Docker pulls images and builds the Flask image.

### 5. Verify All Services Are Healthy

```bash
docker compose ps
```

All 5 services must show `(healthy)` status.

---

## 🔌 API Endpoints

| Method  | Endpoint                  | Description              |
|---------|---------------------------|--------------------------|
| `GET`   | `/`                       | Main UI page             |
| `GET`   | `/api/health`             | Health check (DB + Redis)|
| `GET`   | `/api/tasks`              | List all tasks           |
| `POST`  | `/api/tasks`              | Create a new task        |
| `PATCH` | `/api/tasks/<id>/done`    | Mark a task as done      |

### Example Requests

```bash
# Health check
curl -k https://localhost:4443/api/health

# List tasks
curl -k https://localhost:4443/api/tasks

# Create a task
curl -k -X POST https://localhost:4443/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"My Task","priority":"high"}'

# Mark task as done
curl -k -X PATCH https://localhost:4443/api/tasks/1/done
```

---

## 📸 Screenshots

### Application Running at https://localhost

![App Screenshot](screenshots/app.png)

> DB: Connected | Redis: Connected | Worker: flask1

### All 5 Services Healthy

![Docker Compose PS](screenshots/compose-ps.png)

### Health Endpoint Response

![curl /api/health](screenshots/health-check.png)

---

## 🐋 Key Docker Concepts Used

### Multi-Stage Dockerfile
The Dockerfile uses a **builder** stage to compile dependencies and a lean **runtime** stage to run the app — reducing the final image size from ~1GB to ~120MB.

### Health Checks with `depends_on`
Services use `condition: service_healthy` instead of just listing dependencies, ensuring Flask only starts after PostgreSQL and Redis are fully ready.

### Named Volume
`pgdata` is a Docker named volume (not a bind mount), so database data persists across container restarts and `docker compose down`.

### Secrets via `.env`
All credentials are injected via `.env` using `env_file:` — no passwords are hardcoded in `docker-compose.yml`.

### Nginx Load Balancing
Nginx uses `least_conn` to distribute requests to whichever Flask worker has fewer active connections. It also handles HTTP → HTTPS redirects and serves static files directly.

---

## 🛠️ Useful Commands

```bash
# Check status of all services
docker compose ps

# Watch live logs
docker compose logs -f

# Check Flask logs only
docker compose logs flask1

# Access PostgreSQL directly
docker exec -it postgres psql -U taskuser -d taskmanager -c '\dt'

# Check Redis cache
docker exec -it redis redis-cli keys '*'

# Full reset (removes volumes)
docker compose down -v && docker compose up -d --build
```

---

## 🔒 Security Notes

- `.env` is listed in both `.gitignore` and `.dockerignore` — never committed or baked into images
- Flask containers run as a **non-root user** (`appuser`) inside the container
- Nginx enforces **TLSv1.2 and TLSv1.3** only — older insecure protocols are disabled
- Rate limiting is configured via `limit_req_zone` in Nginx to protect against abuse

---

## 📚 Course

**Docker Advanced Course — Lab 7**
