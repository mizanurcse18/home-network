# 🏗️ Docker Microservices Architecture

## .NET 8 Services on Laptop B + Database on Laptop A

**Date:** 2026-05-25  
**Goal:** Run microservices in Docker on Laptop B, database on Laptop A, then migrate to Linux production server.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HOME NETWORK (192.168.68.x)                  │
│                                                                      │
│   ┌──────────────────────────────┐      ┌────────────────────────┐  │
│   │       LAPTOP B (Docker)      │      │    LAPTOP A (DB)       │  │
│   │  192.168.68.107              │      │  192.168.68.113        │  │
│   │                              │      │                        │  │
│   │  ┌──────────────────────┐    │      │  ┌──────────────────┐  │  │
│   │  │  Docker Compose      │    │      │  │  PostgreSQL /    │  │  │
│   │  │                      │    │      │  │  MySQL            │  │  │
│   │  │  ┌─ security-api ──┐ │    │      │  │  Port: 5432/3306 │  │  │
│   │  │  │  .NET 8         │ │    │      │  └──────────────────┘  │  │
│   │  │  │  Port: 5001     │ │    │      └────────────────────────┘  │
│   │  │  └─────────────────┘ │    │                                  │
│   │  │                      │    │                                  │
│   │  │  ┌─ scm-api ───────┐ │    │                                  │
│   │  │  │  .NET 8         │ │    │                                  │
│   │  │  │  Port: 5002     │ │    │                                  │
│   │  │  └─────────────────┘ │    │                                  │
│   │  │                      │    │                                  │
│   │  │  ┌─ file-service ──┐ │    │                                  │
│   │  │  │  .NET 8         │ │    │                                  │
│   │  │  │  Port: 5003     │ │    │                                  │
│   │  │  └─────────────────┘ │    │                                  │
│   │  └──────────────────────┘    │                                  │
│   └──────────────────────────────┘      └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Network Flow

```
Laptop B (Docker)                        Laptop A (Database)
                                          ──────────────
  security-api:5001 ───┐                 PostgreSQL:5432
  scm-api:5002       ───┼─── LAN ───────► MySQL:3306
  file-service:5003   ──┘
```

### Port Mapping

| Service | Internal Port | Host Port (Laptop B) | Database |
|---------|--------------|---------------------|----------|
| Security API | 8080 | 5001 | PostgreSQL on Laptop A |
| SCM API | 8080 | 5002 | PostgreSQL on Laptop A |
| File Service | 8080 | 5003 | MySQL on Laptop A |

---

## Prerequisites

### On Laptop B

1. **Docker Desktop for Windows** (with Linux containers mode)
   - Download: https://docs.docker.com/desktop/install/windows-install/
   - Enable WSL 2 backend during installation

2. **.NET 8 SDK** (for building locally if needed)
   - Download: https://dotnet.microsoft.com/download/dotnet/8.0

### On Laptop A

1. **PostgreSQL** (or MySQL) installed and running
2. **Firewall port open** for database access from Laptop B

---

## Step 1: Set Up Database on Laptop A

### Option A: PostgreSQL (Recommended)

**Install PostgreSQL** from https://www.postgresql.org/download/windows/

**Configure for remote access:**

1. Edit `C:\Program Files\PostgreSQL\<version>\data\postgresql.conf`:
   ```
   listen_addresses = '*'
   port = 5432
   ```

2. Edit `C:\Program Files\PostgreSQL\<version>\data\pg_hba.conf`:
   ```
   host    all             all             192.168.68.0/24        md5
   ```

3. Restart PostgreSQL service:
   ```cmd
   net stop postgresql-<version>
   net start postgresql-<version>
   ```

4. Add firewall rule:
   ```cmd
   netsh advfirewall firewall add rule name="PostgreSQL" dir=in action=allow protocol=TCP localport=5432
   ```

5. Create databases:
   ```sql
   CREATE DATABASE security_db;
   CREATE DATABASE scm_db;
   CREATE DATABASE file_db;
   ```

### Option B: MySQL

**Install MySQL** from https://dev.mysql.com/downloads/installer/

**Configure for remote access:**

1. Edit `C:\ProgramData\MySQL\MySQL Server <version>\my.ini`:
   ```
   bind-address = 0.0.0.0
   port = 3306
   ```

2. Create user for remote access:
   ```sql
   CREATE USER 'appuser'@'192.168.68.%' IDENTIFIED BY 'YourPassword123!';
   GRANT ALL PRIVILEGES ON *.* TO 'appuser'@'192.168.68.%';
   FLUSH PRIVILEGES;
   ```

3. Add firewall rule:
   ```cmd
   netsh advfirewall firewall add rule name="MySQL" dir=in action=allow protocol=TCP localport=3306
   ```

### Create databases (for both PostgreSQL and MySQL):

Execute these SQL scripts on Laptop A:

```sql
-- For PostgreSQL
CREATE DATABASE security_db;
CREATE DATABASE scm_db;
CREATE DATABASE file_db;

-- Create a single app user
CREATE USER appuser WITH PASSWORD 'YourPassword123!';
GRANT ALL PRIVILEGES ON DATABASE security_db TO appuser;
GRANT ALL PRIVILEGES ON DATABASE scm_db TO appuser;
GRANT ALL PRIVILEGES ON DATABASE file_db TO appuser;
```

---

## Step 2: Docker Setup on Laptop B

### Install Docker Desktop

1. Download from https://docs.docker.com/desktop/install/windows-install/
2. Run installer → **Use WSL 2 instead of Hyper-V** (recommended)
3. After installation, open Docker Desktop
4. Go to **Settings → Resources → WSL Integration** → Enable integration
5. Restart Docker Desktop

### Verify installation

```cmd
docker --version
docker compose version
```

---

## Step 3: Prepare Your .NET 8 Applications

Each service should be a .NET 8 Web API project with this structure:

```
security-api/
├── Dockerfile
├── src/
│   └── Security.Api.csproj
├── appsettings.json
└── appsettings.Docker.json
```

### Sample Dockerfile (for each service)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/Security.Api.csproj", "src/"]
RUN dotnet restore "src/Security.Api.csproj"
COPY . .
RUN dotnet build "src/Security.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "src/Security.Api.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "Security.Api.dll"]
```

### Sample appsettings.Docker.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=192.168.68.113;Port=5432;Database=security_db;Username=appuser;Password=YourPassword123!"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

---

## Step 4: Docker Compose (Run on Laptop B)

Create `docker-compose.yml` at `D:\docker-services\`:

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
      - ConnectionStrings__DefaultConnection=Host=192.168.68.113;Port=5432;Database=security_db;Username=appuser;Password=YourPassword123!
    networks:
      - home-network
    restart: unless-stopped

  scm-api:
    build:
      context: ./scm-api
      dockerfile: Dockerfile
    container_name: scm-api
    ports:
      - "5002:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
      - ConnectionStrings__DefaultConnection=Host=192.168.68.113;Port=5432;Database=scm_db;Username=appuser;Password=YourPassword123!
    networks:
      - home-network
    restart: unless-stopped

  file-service:
    build:
      context: ./file-service
      dockerfile: Dockerfile
    container_name: file-service
    ports:
      - "5003:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Docker
      - ConnectionStrings__DefaultConnection=Host=192.168.68.113;Port=3306;Database=file_db;Username=appuser;Password=YourPassword123!
    networks:
      - home-network
    volumes:
      - file-uploads:/app/uploads
    restart: unless-stopped

networks:
  home-network:
    driver: bridge

volumes:
  file-uploads:
```

### Or use SQL Server on Laptop A (alternative):

```yaml
services:
  security-api:
    # ...
    environment:
      - ConnectionStrings__DefaultConnection=Server=192.168.68.113,1433;Database=security_db;User Id=sa;Password=YourPassword123!;TrustServerCertificate=True
```

---

## Step 5: Run the Services

On **Laptop B**, open **PowerShell/CMD as Administrator**:

```cmd
cd D:\docker-services
docker compose up -d --build
```

### Verify everything is running

```cmd
docker compose ps
docker compose logs security-api
docker compose logs scm-api
docker compose logs file-service
```

### Test from Laptop A (this laptop)

```cmd
curl http://192.168.68.107:5001/health
curl http://192.168.68.107:5002/health
curl http://192.168.68.107:5003/health
```

Or open in browser:
- `http://192.168.68.107:5001/swagger`
- `http://192.168.68.107:5002/swagger`
- `http://192.168.68.107:5003/swagger`

---

## Step 6: Firewall Rules on Laptop B

Allow the service ports:

```cmd
netsh advfirewall firewall add rule name="Security API" dir=in action=allow protocol=TCP localport=5001
netsh advfirewall firewall add rule name="SCM API" dir=in action=allow protocol=TCP localport=5002
netsh advfirewall firewall add rule name="File Service" dir=in action=allow protocol=TCP localport=5003
```

---

## Step 7: Migrate to Linux Production Server

When ready to deploy to a Linux server, the transition is straightforward:

### Changes needed:

1. **Docker Compose** → No changes (already Linux containers)
2. **Database connection** → Update IP/hostname to the production DB server
3. **Environment variables** → Use a `.env` file instead of hardcoded values

### Production docker-compose.yml (Linux)

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
    # Add healthcheck
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  scm-api:
    image: your-registry/scm-api:latest
    # ... same pattern

  file-service:
    image: your-registry/file-service:latest
    volumes:
      - /data/uploads:/app/uploads
    # ... same pattern

  # Add reverse proxy for production
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - security-api
      - scm-api
      - file-service

networks:
  app-network:
    driver: bridge

volumes:
  uploads:
```

### Production .env.production

```env
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__DefaultConnection=Host=prod-db-server.com;Port=5432;Database=security_db;Username=appuser;Password=${DB_PASSWORD}
```

> 🐳 **Tip:** Use Docker's `--env-file` flag or Docker Swarm/Kubernetes secrets for production passwords instead of plaintext `.env` files.

---

## Useful Docker Commands

```cmd
:: Build and start all services
docker compose up -d --build

:: View logs
docker compose logs -f security-api

:: Rebuild a single service
docker compose up -d --build security-api

:: Stop all services
docker compose down

:: Stop and remove volumes
docker compose down -v

:: SSH into a running container
docker exec -it security-api bash

:: Check resource usage
docker stats
```

---

## Database Connection String Reference

### PostgreSQL (on Laptop A)

```
Host=192.168.68.113;Port=5432;Database=security_db;Username=appuser;Password=YourPassword123!
```

### MySQL (on Laptop A)

```
Server=192.168.68.113;Port=3306;Database=file_db;User=appuser;Password=YourPassword123!;SslMode=Preferred
```

### SQL Server (on Laptop A)

```
Server=192.168.68.113,1433;Database=security_db;User Id=sa;Password=YourPassword123!;TrustServerCertificate=True
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Container can't reach database | Verify PostgreSQL/MySQL is running on Laptop A. Check firewall rules. Try `ping 192.168.68.113` from inside container: `docker exec security-api ping 192.168.68.113` |
| Port already in use | Change host port in docker-compose.yml (e.g., `5001:8080` → `5004:8080`) |
| Docker Desktop not starting | Enable virtualization in BIOS, enable WSL 2 |
| .NET app won't start | Check logs: `docker compose logs security-api` |
| Database connection refused | Make sure `listen_addresses = '*'` in postgresql.conf and firewall allows port |
| Slow build | Use Docker layer caching — put `COPY` for .csproj before `dotnet restore` |
| Windows line endings | Set in `.gitattributes`: `*.sh text eol=lf` |

---

*End of document.*
