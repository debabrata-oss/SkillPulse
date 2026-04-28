# SkillPulse

> Track your skills. Log your learning. Ship via GitHub Actions.

SkillPulse is a small full-stack project for tracking skills and the hours you spend learning them. It was built as a hands-on DevOps demo: a Go API, a vanilla JS frontend, a MySQL database, all wired together with Docker Compose and shipped to a remote server through a GitHub Actions CI/CD pipeline.

---

## Architecture

![SkillPulse architecture](docs/architecture.svg)

The system has two halves: the **CI/CD pipeline** that builds and deploys the backend image, and the **runtime** stack that runs on the remote server.

**CI/CD flow**

1. Developer pushes code to the `main` branch.
2. **CI workflow** (`ci.yml`) checks out the code, sets up Docker Buildx, logs in to Docker Hub, builds the backend image from `./backend`, and pushes it as `trainwithdpuhan/skillplus-backend:live`.
3. On successful CI completion, the **CD workflow** (`cd.yml`) is triggered via `workflow_run`. It SSHes into the remote server and runs `git pull`, `docker compose pull`, and `docker compose up -d`.
4. Compose pulls the freshly built backend image from Docker Hub and restarts the stack.

**Runtime stack** (three containers on a shared compose network)

| Service | Image | Role |
|---------|-------|------|
| `nginx` | `nginx:alpine` | Serves static frontend on port 80 and reverse-proxies `/api/*` and `/health` to the backend |
| `backend` | `trainwithdpuhan/skillplus-backend:live` | Go + Gin REST API on port 8080 |
| `db` | `mysql:8.4` | MySQL 8.4 with named volume `mysql_data` for persistence |

The `db` service has a healthcheck (`mysqladmin ping`), and `backend` waits for `service_healthy` before starting, so the API never tries to connect to a half-booted database.

---

## Project structure

```
SkillPulse/
├── .github/
│   └── workflows/
│       ├── ci.yml              # Build + push backend image to Docker Hub
│       └── cd.yml              # SSH to server + redeploy on CI success
├── backend/                    # Go (Gin) REST API
│   ├── database/db.go          # MySQL connection pool with retry-on-startup
│   ├── handlers/               # HTTP handlers (skills, logs, dashboard)
│   ├── models/skill.go         # Data models
│   ├── main.go                 # Routes and server bootstrap
│   ├── go.mod
│   └── Dockerfile              # Multi-stage build (golang:1.26-alpine → alpine:3.23)
├── frontend/                   # Static site served by nginx
│   ├── index.html
│   ├── css/style.css
│   └── js/app.js
├── nginx/
│   └── nginx.conf              # Static + /api reverse proxy config
├── mysql/
│   └── init.sql                # Schema + seed data (runs on first DB boot)
├── docker-compose.yml
└── .env                        # Local-dev defaults (do NOT commit real secrets)
```

---

## API

Base URL on the server: `http://<host>/api`

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/skills` | List all skills with aggregated `total_hours` |
| `POST` | `/api/skills` | Create a skill |
| `GET` | `/api/skills/:id` | Get a single skill plus all its learning logs |
| `DELETE` | `/api/skills/:id` | Delete a skill (cascades to its logs) |
| `POST` | `/api/skills/:id/log` | Log a learning session for a skill |
| `GET` | `/api/dashboard` | Totals: skills, hours, logs, top skill |
| `GET` | `/health` | DB ping; returns `200` healthy or `503` unhealthy |

**Example — create a skill**

```bash
curl -X POST http://localhost/api/skills \
  -H "Content-Type: application/json" \
  -d '{"name":"Kubernetes","category":"DevOps","target_hours":60}'
```

**Example — log an hour against skill 1**

```bash
curl -X POST http://localhost/api/skills/1/log \
  -H "Content-Type: application/json" \
  -d '{"hours":1.5,"notes":"Read pod networking docs","log_date":"2026-04-28"}'
```

---

## Local development

### Prerequisites

- Docker and Docker Compose v2
- A `.env` file in the project root (see template below)

### `.env` template

```env
# MySQL
MYSQL_ROOT_PASSWORD=rootpassword123
DB_HOST=db
DB_PORT=3306
DB_USER=skillpulse
DB_PASSWORD=skillpulse123
DB_NAME=skillpulse

# Used by docker-compose to pull the prebuilt backend image
DOCKERHUB_USERNAME=trainwithdpuhan
```

### Run the stack

```bash
docker compose pull        # fetch backend image from Docker Hub
docker compose up -d       # start db, backend, nginx
docker compose logs -f     # tail logs
```

Then open **http://localhost** in a browser. The frontend talks to the API via the nginx reverse proxy, so the same origin serves both.

### Building the backend locally instead of pulling

If you're iterating on Go code, swap the `image:` line for `build: ./backend` in `docker-compose.yml`:

```yaml
backend:
  build: ./backend
  # image: ${DOCKERHUB_USERNAME}/skillplus-backend:live
```

Then `docker compose up --build`.

### Stopping and resetting

```bash
docker compose down              # stop containers, keep DB data
docker compose down -v           # also drop the mysql_data volume (full reset)
```

---

## CI/CD setup

### GitHub repository secrets

Set these under **Settings → Secrets and variables → Actions**:

| Secret | Used by | Notes |
|--------|---------|-------|
| `DOCKERHUB_USERNAME` | `ci.yml` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | `ci.yml` | Docker Hub access token (create at hub.docker.com → Account Settings → Security) |
| `HOST` | `cd.yml` | Server IP or domain |
| `USERNAME` | `cd.yml` | SSH user on the server |
| `PASSWORD` | `cd.yml` | SSH password (or switch to `key:` with an SSH private key — recommended) |

### Server prerequisites (one-time)

On the deploy target server:

1. Install Docker and the Compose plugin.
2. Clone the repo: `git clone https://github.com/debabrata-oss/SkillPulse.git ~/SkillPulse`.
3. Create `~/SkillPulse/.env` with real values (the file is **not** in git on purpose — those are runtime secrets).
4. Make sure the SSH user can run `docker` without `sudo` (typically `usermod -aG docker $USER`).

After that, every push to `main` builds a new image and the server picks it up automatically.

### How the workflows fit together

- `ci.yml` triggers on every `push`. It only builds and pushes the **backend** image — the frontend is plain static files served straight from the repo by nginx.
- `cd.yml` triggers on `workflow_run` of the CI workflow. It runs only if the upstream CI run completed successfully (you can guard this with `if: github.event.workflow_run.conclusion == 'success'`).

---

## Database schema

Defined in `mysql/init.sql` and applied automatically the first time the `db` container starts (because the file is mounted into `/docker-entrypoint-initdb.d/`).

```sql
skills
├── id            INT PK AUTO_INCREMENT
├── name          VARCHAR(100)
├── category      VARCHAR(50)
├── target_hours  INT
└── created_at    TIMESTAMP

learning_logs
├── id          INT PK AUTO_INCREMENT
├── skill_id    INT  → skills.id  (ON DELETE CASCADE)
├── hours       DECIMAL(4,1)
├── notes       TEXT
├── log_date    DATE
└── created_at  TIMESTAMP
```

The seed data inserts five skills (Docker, Kubernetes, Go, Azure DevOps, Terraform) and a handful of learning logs so the dashboard isn't empty on first boot.

---

## Troubleshooting

**`The "DOCKERHUB_USERNAME" variable is not set` during `docker compose pull`**
The `.env` file is missing or in the wrong directory on the server. Compose reads `.env` from the **same folder as `docker-compose.yml`**. Create `~/SkillPulse/.env` with the values shown above.

**`invalid reference format` on the server**
Same root cause as above — when `${DOCKERHUB_USERNAME}` is empty, the image resolves to `/skillplus-backend:live`, which Docker rejects. Fix the `.env` file.

**`fatal: destination path 'SkillPulse' already exists` in CD logs**
The CD script tries to `git clone` every run. Guard it so the clone only happens once:

```bash
if [ ! -d "$HOME/SkillPulse/.git" ]; then
  git clone https://github.com/debabrata-oss/SkillPulse.git "$HOME/SkillPulse"
fi
cd "$HOME/SkillPulse"
git pull origin main
```

**CD workflow never triggers after a successful CI run**
The `workflows:` value in `cd.yml` must match the CI workflow's `name:` field **exactly**, including case. CI is named `TWS CI workflow` (lowercase `w`), so `cd.yml` needs `workflows: ["TWS CI workflow"]`.

**Backend logs `Waiting for database... attempt N/30`**
Normal during first boot. The Go app retries the MySQL connection up to 30 times (60 s total) while MySQL initializes. If it times out, check `docker compose logs db` — usually a wrong password or a corrupted `mysql_data` volume from a previous run with different credentials. `docker compose down -v` resets it.

---

## Tech stack

- **Backend:** Go 1.26, Gin web framework, `database/sql` with the `go-sql-driver/mysql` driver
- **Frontend:** Plain HTML, CSS, JavaScript (no framework)
- **Database:** MySQL 8.4
- **Reverse proxy / static server:** Nginx (alpine)
- **Orchestration:** Docker Compose v2
- **CI/CD:** GitHub Actions, Docker Hub, `appleboy/ssh-action`
