# n8n Automation Stack (Podman + PostgreSQL)

A lightweight, rootless container deployment for hosting [n8n](https://n8n.io) using Podman, Podman Compose, and PostgreSQL on macOS or Linux.

---

## 📋 Features

- **Rootless Containers:** Improved host security using Podman instead of Docker.
- **Production Database:** Built-in PostgreSQL backend for robust execution logging and persistence.
- **Persistent Storage:** Named volumes for workflow retention and database migrations.
- **Reverse Proxy Ready:** Configured for seamless Nginx and SSL integration.

---

## 🛠️ Prerequisites

- **macOS** with [Homebrew](https://brew.sh/) installed.
- **Terminal** access.

---

## 🚀 Quick Start (macOS)

### 1. Install Podman & Tools

```bash
brew install podman podman-compose
```

### 2. Initialize and Start the Podman VM

```bash
podman machine init
podman machine start
```

Verify that the machine is active:

```bash
podman info
```

### 3. Clone / Set Up Project Directory

```bash
mkdir -p ~/n8n-mac && cd ~/n8n-mac
```

### 4. Configure `docker-compose.yml`

Create a `docker-compose.yml` file in the project root:

```yaml
version: '3.8'

services:
  postgres:
    image: docker.io/library/postgres:16-alpine
    container_name: n8n_postgres
    restart: always
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ChangeThisSecretPassword123!
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: docker.io/n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=ChangeThisSecretPassword123!
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - GENERIC_TIMEZONE=UTC
      - TZ=UTC
    depends_on:
      - postgres
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  postgres_data:
  n8n_data:
```

### 5. Launch n8n

```bash
podman-compose up -d
```

Access the n8n UI in your web browser at:
👉 **[http://localhost:5678](http://localhost:5678)**

---

## 🌐 Exposing n8n to the Public (Nginx Reverse Proxy)

When deploying to a public server, update your environment settings in `docker-compose.yml`:

```yaml
    environment:
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/
```

### Nginx Configuration Template

Create `/etc/nginx/sites-available/n8n.conf` on your public server:

```nginx
server {
    server_name n8n.yourdomain.com;

    # 1. Allow public access to incoming webhooks
    location /webhook/ {
        proxy_pass http://127.0.0.1:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 2. Restrict UI access to specific trusted IPs
    location / {
        allow YOUR_HOME_IP_HERE;
        deny all;

        proxy_pass http://127.0.0.1:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support for n8n UI
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Obtain a free SSL certificate with Certbot:

```bash
sudo certbot --nginx -d n8n.yourdomain.com
```

---

## 🛠️ Management Commands

| Task | Command |
| :--- | :--- |
| **View Running Containers** | `podman ps` |
| **View n8n Logs** | `podman logs -f n8n` |
| **Stop Stack** | `podman-compose down` |
| **Update Images** | `podman-compose pull && podman-compose up -d` |
| **Stop Podman VM (Mac)** | `podman machine stop` |
| **Start Podman VM (Mac)** | `podman machine start` |