# Running the Stack with Docker (Individual Containers)

This guide shows how to run each tier with plain `docker build` and `docker run` commands — **without Docker Compose**. Use it to understand what each container does and how they connect.

> **Instructor shortcut:** To start everything at once, use `docker compose up --build -d` (see [README.md](README.md)).

---

## What you are building

| Tier | Folder | Container name | Port on your laptop |
|------|--------|----------------|---------------------|
| Database | `application-code/mysql/` | `mysql` | *(internal only — 3306 inside the network)* |
| App (API) | `application-code/app-tier/` | `app-tier` | **4000** |
| Web (React + Nginx) | `application-code/web-tier/` | `web-tier` | **8080** |

All three containers share a Docker network so they can reach each other by name (`mysql`, `app-tier`).

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine + CLI) installed and running
- Clone this repository and open a terminal in the **project root** (the folder that contains `docker-compose.yml`)

---

## Step 0 — Create a shared network

Run this once before starting any container:

```bash
docker network create ab3-network
```

---

## Step 1 — MySQL (database tier)

### Build

MySQL uses the official image — no build step.

### Run

From the project root:

```bash
docker run -d \
  --name mysql \
  --network ab3-network \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=ab3 \
  -v ab3_mysql_data:/var/lib/mysql \
  -v "$(pwd)/application-code/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql:ro" \
  mysql:8.0 \
  --default-authentication-plugin=mysql_native_password
```

The `init.sql` script creates the `transactions` table on first startup.

### Verify

Wait ~20 seconds, then:

```bash
docker logs mysql
```

You should see MySQL report that it is ready for connections.

Optional — confirm the database is up:

```bash
docker exec mysql mysqladmin ping -h localhost -ppassword
```

Expected output includes: `mysqld is alive`

### Stop / remove (when cleaning up)

```bash
docker stop mysql
docker rm mysql
```

> **Note:** The named volume `ab3_mysql_data` keeps your data if you remove the container. Delete it with `docker volume rm ab3_mysql_data` if you want a fresh database.

---

## Step 2 — App tier (Node.js API)

Start MySQL first (Step 1). The app tier connects using these environment variables (same as in `docker-compose.yml`):

| Variable | Value |
|----------|-------|
| `DB_HOST` | `mysql` |
| `DB_USER` | `root` |
| `DB_PWD` | `password` |
| `DB_DATABASE` | `ab3` |

### Build

From the project root:

```bash
docker build -t ab3-app-tier ./application-code/app-tier
```

### Run

```bash
docker run -d \
  --name app-tier \
  --network ab3-network \
  -e DB_HOST=mysql \
  -e DB_USER=root \
  -e DB_PWD=password \
  -e DB_DATABASE=ab3 \
  -p 4000:4000 \
  ab3-app-tier
```

### Verify

```bash
curl http://localhost:4000/health
```

Expected response:

```json
"This is the health check"
```

Try the transactions API:

```bash
curl http://localhost:4000/transaction
```

### Stop / remove

```bash
docker stop app-tier
docker rm app-tier
```

---

## Step 3 — Web tier (React frontend + Nginx)

Start the app tier first (Step 2). Nginx inside this container proxies `/api/` requests to `http://app-tier:4000/` — that hostname works because both containers are on `ab3-network` and the API container is named `app-tier`.

### Build

From the project root:

```bash
docker build \
  --build-arg REACT_APP_API_URL=/api \
  -t ab3-web-tier \
  ./application-code/web-tier
```

### Run

```bash
docker run -d \
  --name web-tier \
  --network ab3-network \
  -p 8080:80 \
  ab3-web-tier
```

### Verify

Open in your browser:

**http://localhost:8080/#/**

Health check:

```bash
curl http://localhost:8080/health
```

Expected: `Web Tier Health Check`

### Stop / remove

```bash
docker stop web-tier
docker rm web-tier
```

---

## Full startup script (copy-paste)

After cloning the repo and creating the network, you can run all three tiers in order:

```bash
# From project root
docker network create ab3-network 2>/dev/null || true

docker run -d \
  --name mysql \
  --network ab3-network \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=ab3 \
  -v ab3_mysql_data:/var/lib/mysql \
  -v "$(pwd)/application-code/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql:ro" \
  mysql:8.0 \
  --default-authentication-plugin=mysql_native_password

echo "Waiting for MySQL..."
until docker exec mysql mysqladmin ping -h localhost -ppassword --silent; do
  sleep 2
done

docker build -t ab3-app-tier ./application-code/app-tier
docker run -d \
  --name app-tier \
  --network ab3-network \
  -e DB_HOST=mysql \
  -e DB_USER=root \
  -e DB_PWD=password \
  -e DB_DATABASE=ab3 \
  -p 4000:4000 \
  ab3-app-tier

docker build \
  --build-arg REACT_APP_API_URL=/api \
  -t ab3-web-tier \
  ./application-code/web-tier
docker run -d \
  --name web-tier \
  --network ab3-network \
  -p 8080:80 \
  ab3-web-tier

echo "Open http://localhost:8080/#/"
```

---

## Tear down everything

```bash
docker stop web-tier app-tier mysql 2>/dev/null
docker rm web-tier app-tier mysql 2>/dev/null
docker network rm ab3-network 2>/dev/null
```

To also delete persisted database data:

```bash
docker volume rm ab3_mysql_data
```

---

## Troubleshooting

| Problem | What to check |
|---------|----------------|
| `app-tier` exits immediately | MySQL may not be ready. Run Step 1 verify commands, then restart app-tier. |
| Browser loads UI but API fails | Ensure `web-tier` and `app-tier` are both on `ab3-network` and the API container is named exactly `app-tier`. |
| `port is already allocated` | Another process (or an old container) is using 4000 or 8080. Run `docker ps` and stop conflicting containers. |
| `container name already in use` | Remove the old container: `docker rm -f mysql` (or `app-tier` / `web-tier`). |
| Empty database after restart | If you removed the `ab3_mysql_data` volume, MySQL re-runs `init.sql` on next first start. |

View logs for any tier:

```bash
docker logs mysql
docker logs app-tier
docker logs web-tier
```

---

## How this maps to Docker Compose

The commands above mirror [docker-compose.yml](docker-compose.yml):

- **mysql** → `mysql` service (image, env vars, volume, init script)
- **app-tier** → `app-tier` service (build context, env vars, port 4000)
- **web-tier** → `web-tier` service (build arg `REACT_APP_API_URL=/api`, port 8080→80)

Compose adds health checks and startup ordering automatically; with individual `docker run` commands, start tiers in order (database → API → web) and wait for MySQL before starting the app tier.
