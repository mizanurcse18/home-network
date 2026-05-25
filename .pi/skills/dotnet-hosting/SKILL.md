---
name: dotnet-hosting
description: Deploys .NET 8 microservices in Docker with PostgreSQL/MySQL on a separate database host. Covers Dockerfiles, Docker Compose, multi-service architecture (security, SCM, file service), database setup for remote access, firewall configuration, and migration from development to Linux production server. Includes date/time session tracking.
---

# .NET 8 Docker Hosting

Deploy .NET 8 microservices in Docker containers with a dedicated database server on a separate machine.

> ⏱ **Session history is tracked below.** Always read the [Session History](#-session-history) section first to understand what has already been configured before making changes.

## Overview

This skill helps you:
1. Set up Docker Desktop on Windows (WSL 2 backend) for Linux container hosting
2. Configure PostgreSQL or MySQL on a separate laptop/server for remote access
3. Build Dockerfiles for .NET 8 Web API projects
4. Orchestrate multiple microservices (security, SCM, file service) with Docker Compose
5. Open Windows Firewall for service and database ports
6. Migrate seamlessly to a Linux production environment
7. **Track all actions with date/time stamps** for future reference

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LOCAL NETWORK                             │
│                                                              │
│   Docker Host (Windows)            Database Server           │
│   ─────────────────────            ───────────────           │
│                                                              │
│   ┌─ security-api:5001 ──┐         ┌─ PostgreSQL:5432 ───┐  │
│   │  .NET 8 + Linux      │         │  or MySQL:3306      │  │
│   │  Container           │         │                     │  │
│   └──────────────────────┘         └─────────────────────┘  │
│                                                              │
│   ┌─ scm-api:5002 ───────┐         Network: LAN/Wi-Fi       │
│   │  .NET 8 + Linux      │         Subnet: 192.168.x.x      │
│   │  Container           │                                   │
│   └──────────────────────┘                                   │
│                                                              │
│   ┌─ file-service:5003 ──┐                                   │
│   │  .NET 8 + Linux      │                                   │
│   │  Container           │                                   │
│   └──────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Windows 10/11 with WSL 2 support (for Docker Desktop)
- A separate machine for the database (or same machine for testing)
- .NET 8 SDK (for local builds/testing)
- Administrator access for firewall and Docker installation

---

## Step 1: Install Docker Desktop on Windows

### 1.1 Download & Install

1. Download from [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)
2. Run the installer
3. When prompted, select **Use WSL 2 instead of Hyper-V** (recommended)
4. Complete installation and restart if required

### 1.2 Configure WSL 2 Integration

1. Open **Docker Desktop**
2. Go to **Settings** → **Resources** → **WSL Integration**
3. Enable integration with your WSL distro (e.g., Ubuntu)
4. Click **Apply & Restart**

### 1.3 Verify Installation

```cmd
docker --version
docker compose version
docker run hello-world
```

---

## Step 2: Set Up Database Server

### 2.1 Option A: PostgreSQL (Recommended)

#### Install
Download from [PostgreSQL Windows](https://www.postgresql.org/download/windows/)

#### Configure for Remote Access

Edit `C:\Program Files\PostgreSQL\<version>\data\postgresql.conf`:

```
listen_addresses = '*'
port = 5432
```

Edit `C:\Program Files\PostgreSQL\<version>\data\pg_hba.conf`:

```
host    all             all             192.168.0.0/16        md5
```

#### Restart Service

```cmd
net stop postgresql-<version>
net start postgresql-<version>
```

#### Firewall Rule

```cmd
netsh advfirewall firewall add rule name="PostgreSQL" dir=in action=allow protocol=TCP localport=5432
```

#### Create Databases and User

```sql
CREATE DATABASE security_db;
CREATE DATABASE scm_db;
CREATE DATABASE file_db;

CREATE USER appuser WITH PASSWORD 'YourStrong!Pass123';
GRANT ALL PRIVILEGES ON DATABASE security_db TO appuser;
GRANT ALL PRIVILEGES ON DATABASE scm_db TO appuser;
GRANT ALL PRIVILEGES ON DATABASE file_db TO appuser;
```

### 2.2 Option B: MySQL

#### Install
Download from [MySQL Installer](https://dev.mysql.com/downloads/installer/)

#### Configure for Remote Access

Edit `C:\ProgramData\MySQL\MySQL Server <version>\my.ini`:

```
bind-address = 0.0.0.0
port = 3306
```

#### Create Remote User

```sql
CREATE USER 'appuser'@'192.168.%' IDENTIFIED BY 'YourStrong!Pass123';
GRANT ALL PRIVILEGES ON *.* TO 'appuser'@'192.168.%';
FLUSH PRIVILEGES;
```

#### Firewall Rule

```cmd
netsh advfirewall firewall add rule name="MySQL" dir=in action=allow protocol=TCP localport=3306
```

---

## Step 3: Dockerfile for .NET 8 Services

Create this `Dockerfile` in each service project directory:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/Service.Api.csproj", "src/"]
RUN dotnet restore "src/Service.Api.csproj"
COPY . .
RUN dotnet build "src/Service.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "src/Service.Api.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "Service.Api.dll"]
```

### Environment-Specific appsettings

Create `appsettings.Docker.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=DATABASE_IP;Port=5432;Database=security_db;Username=appuser;Password=YourStrong!Pass123"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

---

## Step 4: Docker Compose

Create `docker-compose.yml` on the Docker host machine:

```yaml
version: '3.8'

services:
  security-api:
    build:
      context: ./security-api
      dockerfile: Dockerfile
    container_name: security-api
    ports:
      - "5001:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
      - ConnectionStrings__DefaultConnection=Host=DATABASE_IP;Port=5432;Database=security_db;Username=appuser;Password=YourStrong!Pass123
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  scm-api:
    build:
      context: ./scm-api
      dockerfile: Dockerfile
    container_name: scm-api
    ports:
      - "5002:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
      - ConnectionStrings__DefaultConnection=Host=DATABASE_IP;Port=5432;Database=scm_db;Username=appuser;Password=YourStrong!Pass123
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  file-service:
    build:
      context: ./file-service
      dockerfile: Dockerfile
    container_name: file-service
    ports:
      - "5003:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
      - ConnectionStrings__DefaultConnection=Host=DATABASE_IP;Port=3306;Database=file_db;User=appuser;Password=YourStrong!Pass123
    networks:
      - app-network
    volumes:
      - file-uploads:/app/uploads
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge

volumes:
  file-uploads:
```

> **Replace `DATABASE_IP`** with the actual IP of your database server (e.g., `192.168.68.113`).

---

## Step 5: Run the Services

### Open Firewall Ports (on Docker host)

```cmd
netsh advfirewall firewall add rule name="Security API" dir=in action=allow protocol=TCP localport=5001
netsh advfirewall firewall add rule name="SCM API" dir=in action=allow protocol=TCP localport=5002
netsh advfirewall firewall add rule name="File Service" dir=in action=allow protocol=TCP localport=5003
```

### Deploy

```cmd
cd C:\path\to\docker-compose.yml
docker compose up -d --build
```

### Verify

```cmd
docker compose ps
docker compose logs security-api
```

### Test

```cmd
curl http://localhost:5001/health
curl http://DOCKER_HOST_IP:5001/swagger
```

---

## Step 6: Migrate to Linux Production Server

When moving to a production Linux server, the transition is minimal.

### 6.1 Changes Required

| Aspect | Development (Windows) | Production (Linux) |
|--------|----------------------|-------------------|
| Docker Compose | Same file | Same file (Linux containers) |
| Connection Strings | Local DB IP | Production DB hostname |
| Environment Config | Hardcoded or `.env` | `.env.production` or secrets |
| Reverse Proxy | None | nginx or Traefik |
| Image Registry | Local build | Docker Hub / AWS ECR / GHCR |

### 6.2 Production docker-compose.yml

```yaml
version: '3.8'

services:
  security-api:
    image: your-registry/security-api:latest
    container_name: security-api
    ports:
      - "5001:8080"
    env_file:
      - .env.production
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ... scm-api, file-service same pattern

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - security-api
      - scm-api
      - file-service
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### 6.3 Production .env.production

```env
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__DefaultConnection=Host=prod-db.example.com;Port=5432;Database=security_db;Username=appuser;Password=${DB_PASSWORD}
```

> Use Docker secrets or a vault for production passwords.

### 6.4 Push Images to Registry

```cmd
docker tag security-api your-registry/security-api:latest
docker push your-registry/security-api:latest
```

### 6.5 Deploy to Linux Server

```bash
scp docker-compose.yml user@server:/opt/app/
scp .env.production user@server:/opt/app/
ssh user@server
cd /opt/app
docker compose pull
docker compose up -d
```

---

## Useful Commands

```cmd
:: Build and start
docker compose up -d --build

:: View logs (follow)
docker compose logs -f security-api

:: Rebuild a single service
docker compose up -d --build security-api

:: Stop all
docker compose down

:: Stop and remove volumes
docker compose down -v

:: SSH into container
docker exec -it security-api bash

:: List images
docker images

:: Remove unused images
docker image prune
```

---

## Database Connection Strings Reference

### PostgreSQL
```
Host=SERVER_IP;Port=5432;Database=db_name;Username=appuser;Password=YourStrong!Pass123
```

### MySQL
```
Server=SERVER_IP;Port=3306;Database=db_name;User=appuser;Password=YourStrong!Pass123;SslMode=Preferred
```

### SQL Server
```
Server=SERVER_IP,1433;Database=db_name;User Id=sa;Password=YourStrong!Pass123;TrustServerCertificate=True
```

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| Container can't reach database | Verify DB is running and `listen_addresses = '*'`. Check firewall on DB host. Test with `docker exec <container> ping DB_IP` |
| Port already in use | Change host port in docker-compose.yml (e.g., `5001:8080` → `5004:8080`) |
| Docker Desktop won't start | Enable virtualization in BIOS, enable WSL 2, or use Hyper-V backend |
| .NET app fails to start | Check logs: `docker compose logs <service>` |
| Database connection refused | Ensure `pg_hba.conf` allows remote connections, firewall port is open, and PostgreSQL service is running |
| Slow Docker builds | Optimize Dockerfile: copy .csproj first, restore, then copy rest of code |
| Windows/CRLF line issues | Set `.gitattributes`: `*.sh text eol=lf` |

Quick diagnostics:

```cmd
:: Check Docker status
docker info

:: List running containers
docker ps

:: Check network
docker network ls
docker network inspect app-network

:: Test DB connectivity from container
docker exec security-api ping DB_IP

:: Check logs
docker compose logs --tail=50 -f
```

---

## ⏱ Session History

> **Agent instructions:** Before making any changes, always read this section to understand what's already configured. After making changes, append a new entry with the current date/time and description of what was done.

### Entry Format

```
### YYYY-MM-DD HH:MM — Brief title

**Action:** What was done
**Host(s):** Which machines were involved
**Configuration:** Key IPs, ports, settings applied
**Result:** ✅ Success / ❌ Failure / ⚠️ Partial
**Notes:** Any observations or next steps
```

---

*Add new sessions below this line using the entry format above.*
