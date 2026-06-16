# Proyecto DBT - SQL Server Integration

This project sets up a **dbt (data build tool)** environment. Note: SQL Server is optional — many clients provide a centrally hosted SQL Server instance (installation and maintenance are typically handled on the client/server side). The project can run with a local SQL Server container (included) or connect to an external/client server by configuring `dbt/profiles.yml`.

## Overview

The architecture consists of:
- **SQL Server 2022** container with ODBC drivers (optional — can use client's SQL Server instead)
- **dbt-core** container with dbt-sqlserver adapter
- Network communication between containers for seamless dbt operations
- Cross-platform support (Windows, macOS, Linux)

## Operating System Compatibility

| OS | Status | Notes |
|---|---|---|
| **Windows 10/11** | ✅ Fully Supported | Use PowerShell, WSL2, or Git Bash |
| **macOS (Intel)** | ✅ Fully Supported | Use native Terminal or iTerm2 |
| **macOS (Apple Silicon/M1/M2)** | ✅ Fully Supported | Native ARM support with Docker Desktop 4.11+ |
| **Linux (Ubuntu/Debian)** | ✅ Fully Supported | Native Docker support; may require sudo |
| **Linux (CentOS/RHEL)** | ✅ Fully Supported | May require SELinux configuration |
| **Linux (on-premises)** | ✅ Fully Supported | See deployment notes below |
| **Docker Desktop** | ✅ Recommended | Includes Docker and Compose |
| **Standalone Docker** | ✅ Supported | Install Docker and Compose separately |

### Linux On-Premises Deployment

For production Linux environments (on-premises or VM):

```bash
# Install Docker (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install docker.io docker-compose

# Install Docker (CentOS/RHEL)
sudo yum install docker docker-compose

# Add your user to docker group (optional, to avoid sudo)
sudo usermod -aG docker $USER
newgrp docker  # Apply immediately

# Start docker service
sudo systemctl start docker
sudo systemctl enable docker  # Auto-start on boot

# Verify installation
docker --version
docker-compose --version
```

Then follow the standard setup instructions starting from section 2 (Configure Environment).

## Project Structure

```
proyecto-dbt/
├── docker-compose.yml          # Docker container orchestration
├── Dockerfile.dbt              # dbt container image build
├── .env                        # Environment variables (SQL password)
├── dbt/                        # dbt project directory
│   ├── dbt_project.yml         # dbt project configuration
│   ├── profiles.yml            # dbt profiles & adapter configuration
│   ├── .env                    # dbt-specific environment variables
│   ├── .user.yml               # dbt user settings
│   ├── models/                 # SQL transformation models (user-created)
│   ├── seeds/                  # CSV seed data (user-created)
│   ├── tests/                  # Data quality tests (user-created)
│   └── target/                 # Generated dbt artifacts (auto-created)
└── README.md                   # Project documentation
```

## Setup Instructions

### Prerequisites

**All Platforms:**
- Docker installed ([Installation Guide](https://docs.docker.com/get-docker/))
- Docker Compose installed (included with Docker Desktop)
- Terminal/Shell access
- Git (optional, included in dbt container)

**Platform-Specific Notes:**

| OS | Requirements | Notes |
|---|---|---|
| **Windows** | Docker Desktop with WSL2 or Hyper-V | Use PowerShell, WSL2 terminal, or Git Bash |
| **macOS** | Docker Desktop | Use Terminal or any shell (bash/zsh) |
| **Linux** | Docker & Docker Compose | Use native terminal; may need `sudo` for docker commands |

### 1. Clone or Download Project

```bash
# Windows (PowerShell / WSL2 / Git Bash)
cd $HOME\Desktop
# or
cd ~/Desktop

# macOS / Linux
cd ~/Desktop
# or
cd /home/username/workspace

git clone <repo-url> proyecto-dbt
cd proyecto-dbt
```

**Alternative: Manual Download**
```bash
mkdir -p ~/proyecto-dbt
cd ~/proyecto-dbt
# Copy all project files here
```

### 2. Configure Environment

The project includes a `.env` file with the SQL Server password.

**File: `.env`**
```env
SQL_PASSWORD=TuPasswordSuperSeguroYComplejo2026!
```

Notes:
- If you run the local SQL Server container, this password is used during initialization.
- If you connect to the client's SQL Server, supply credentials in `dbt/profiles.yml` or via environment/secret management; the local `.env` is not required for that case.

Example (client-hosted secret env var):
```env
CLIENT_SQL_PASSWORD=SuperSecretClientPassword
```

**Optional: Custom Password**

Edit `.env` to use a custom password (must be complex - uppercase, lowercase, numbers, symbols):
```bash
# Linux / macOS
nano .env
# Windows (PowerShell)
notepad .env
```

### 3. Build and Start Containers

**All Platforms (same command):**
```bash
docker-compose up --build -d
```

**Output (all platforms):**
```
[+] Building 2.0s
[+] up 2/2
 ✔ Network proyecto-dbt_default Created
 ✔ Container sqlserver_local Started
 ✔ Container dbt_core_local Started
```

**Windows Users with Permission Issues:**
```powershell
# Run PowerShell as Administrator, then:
docker-compose up --build -d
```

**Linux Users with Permission Issues:**
```bash
# Add your user to docker group
sudo usermod -aG docker $USER
# Then log out and log back in, or run:
sudo docker-compose up --build -d
```

**Verify Containers Running (all platforms):**
```bash
docker-compose ps
```

This command:
- Builds the dbt Docker image from `Dockerfile.dbt`
- Starts both SQL Server and dbt containers
- Creates a Docker network for inter-container communication
- Mounts the `./dbt` directory to `/usr/app/dbt_project` in the dbt container

### 4. Verify Configuration with dbt debug

**All Platforms (same command):**
```bash
docker-compose exec -T dbt dbt debug
```

**If command fails on Linux with permission error:**
```bash
sudo docker-compose exec -T dbt dbt debug
```

**Successful Output (all platforms):**
```
Running with dbt=1.11.11
dbt version: 1.11.11
python version: 3.11.13
os info: Linux-6.6.114.1-microsoft-standard-WSL2-x86_64-with-glibc2.31

Using profiles dir at /usr/app/dbt_project
Using profiles.yml file at /usr/app/dbt_project/profiles.yml
Using dbt_project.yml file at /usr/app/dbt_project/dbt_project.yml

adapter type: sqlserver
adapter version: 1.9.2

Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Required dependencies:
  - git [OK found]

Connection:
  server: sqlserver
  port: 1433
  database: master
  schema: dbo
  UID: sa
  authentication: sql
  retries: 3
  login_timeout: 0
  query_timeout: 0
  trace_flag: False
  encrypt: True
  trust_cert: True

Registered adapter: sqlserver=1.9.2
  Connection test: [OK connection ok]

All checks passed!
```

## Docker Configuration Details

## Client environment details (remote connection)

The client provided the following information for the remote database connection:

- **Operating system (server):** Ubuntu 24.04.4 LTS / Linux 6.8.0-117-generic
- **Docker (server):** Docker version 29.5.2, build 79eb04c
- **Database deployment:** Remote host at DataCenter AYESA (Windows Server 2019)
- **Host / Instance:** 10.200.207.194/TBXSQLAPPDES (server IP / instance)
- **SQL Server version:** Microsoft SQL Server 2022 (RTM-CU21) (KB5065865) - 16.0.4215.2 (X64) Developer Edition (64-bit)

Use these values in `dbt/profiles.yml` or ask the client to prepare the host and credentials. Recommended: store the password in an environment variable `CLIENT_SQL_PASSWORD` and do not include it in the repository.

### Client checklist (pre-connection verification)

Please confirm the following before the development team attempts to connect:

- **Network access:** TCP port `1433` open from the development network to `10.200.207.194` (or the required IP range).
- **Instance and name:** Confirm the instance `TBXSQLAPPDES` and that `10.200.207.194/TBXSQLAPPDES` accepts SQL Server connections.
- **Credentials:** Provide a user with sufficient privileges to create tables/schemas (or indicate a read-only user if applicable). Provide username and `CLIENT_SQL_PASSWORD` securely.
- **Firewall / VPN:** Confirm whether a VPN or specific firewall rules are required to allow access from the development network.
- **Authentication:** Confirm authentication type (SQL Auth) and whether `encrypt=true` and `trust_cert` are appropriate for the server environment.
- **Access test:** Provide or run a basic connectivity test (e.g., `sqlcmd` or a test client) and share the result.
- **Instance permissions:** Verify the user has permissions to create databases/schemas in the target database (`master` or the specified DB).
- **Maintenance windows:** Indicate maintenance windows or access restrictions if any.

Optional: if the client prefers that the team does not attempt the initial connection, they can provide a connectivity report and credentials via a secure channel.

### docker-compose.yml

**SQL Server Service (Optional):**
- Image: `mcr.microsoft.com/mssql/server:2022-latest`
- Container: `sqlserver_local`
- Port: `1433:1433` (exposed to host)
- Volume: `sqlserver_data` (persistent storage)
- Environment: `ACCEPT_EULA=Y`, `MSSQL_SA_PASSWORD` (from .env)

If your organization provides a central/client SQL Server, you do not need to run this service locally. To use the client's SQL Server instead:
- Remove or comment out the `sqlserver` service in `docker-compose.yml`.
- Update `dbt/profiles.yml` `host` to the client's SQL Server address and provide credentials via environment variables or secrets.

**dbt Service:**
- Build: `Dockerfile.dbt`
- Container: `dbt_core_local`
- Volume: `./dbt:/usr/app/dbt_project` (mount local dbt directory)
- Environment:
  - `DBT_PROFILES_DIR=/usr/app/dbt_project` (profiles location)
  - `DBT_ENV_SECRET_PASSWORD` (from .env for database auth)
- Depends on: `sqlserver` (waits for SQL Server to be ready)
- Command: `tail -f /dev/null` (keeps container alive)

### Dockerfile.dbt

Multi-stage build that:
1. Starts with Python 3.11 slim image
2. Installs system dependencies: curl, gnupg, git, build-essential
3. Adds Microsoft ODBC drivers (required for SQL Server connections)
4. Installs dbt-core and dbt-sqlserver adapter
5. Sets working directory to `/usr/app/dbt_project`

**Key installations:**
- `dbt-core` - Core dbt framework
- `dbt-sqlserver` - SQL Server adapter
- `msodbcsql18` - Microsoft ODBC Driver 18 for SQL Server
- `git` - Version control & dbt dependency management

### dbt Configuration Files

**dbt_project.yml**
```yaml
name: 'proyecto_dbt'
version: '1.0.0'
config-version: 2
profile: 'default'

models:
  proyecto_dbt:
    materialized: table
```

**profiles.yml**
```yaml
default:
  outputs:
    dev:
      type: sqlserver
      driver: 'ODBC Driver 18 for SQL Server'
      host: sqlserver          # Docker container name
      port: 1433
      user: sa
      password: "{{ env_var('DBT_ENV_SECRET_PASSWORD') }}"
      database: master
      schema: dbo
      encrypt: true
      trust_cert: true
  target: dev
```

**Connection Details:**
- Host: `sqlserver` (Docker container name, resolved via DNS)
- User: `sa` (SQL Server system administrator)
- Password: Injected from environment variable
- Database: `master` (default system database, can be changed)
- Schema: `dbo` (default schema)
- Encryption: Enabled with self-signed certificate trust

## Common dbt Commands

Once `dbt debug` shows all checks passed, you can run (all platforms use same syntax):

```bash
# Load seed data (CSV files)
docker-compose exec -T dbt dbt seed

# Run models (create/update tables)
docker-compose exec -T dbt dbt run

# Test data quality
docker-compose exec -T dbt dbt test

# Generate documentation
docker-compose exec -T dbt dbt docs generate

# Compile without running
docker-compose exec -T dbt dbt compile

# Preview SQL without executing
docker-compose exec -T dbt dbt compile --select model_name
```

**For Linux users requiring sudo:**
```bash
# Prefix all docker-compose commands with sudo
sudo docker-compose exec -T dbt dbt seed
sudo docker-compose exec -T dbt dbt run
```

**Pro Tip (Linux):** Add your user to docker group to avoid using `sudo`:
```bash
sudo usermod -aG docker $USER
# Log out and back in for changes to take effect
```

## Container Management

**All commands work on Windows, macOS, and Linux:**

```bash
# View running containers (all platforms)
docker-compose ps

# View container logs (all platforms)
docker-compose logs sqlserver
docker-compose logs dbt
docker-compose logs -f  # Follow logs in real-time

# Stop containers (all platforms)
docker-compose down

# Remove volumes (data deletion - all platforms)
docker-compose down -v

# Interactive shell in dbt container (all platforms)
docker-compose exec dbt bash

# Database query via dbt container (all platforms)
docker-compose exec -T dbt sqlcmd -S sqlserver -U sa -P <password> -Q "SELECT @@VERSION"
```

**Linux-specific (with sudo if needed):**
```bash
sudo docker-compose ps
sudo docker-compose down
sudo docker-compose logs -f
```

**Windows PowerShell specific:**
```powershell
# Run as Administrator for best experience
docker-compose ps
docker-compose down
docker-compose logs -f
```

**macOS specific notes:**
- Native support for Docker Desktop
- No special permissions usually required
- Use standard terminal or iTerm2

## Troubleshooting

### Cross-Platform Issues

#### Docker daemon not running
```bash
# Windows: Start Docker Desktop
# macOS: Start Docker Desktop
# Linux: Start docker service
sudo systemctl start docker
```

#### Permission denied errors
```bash
# Windows: Run terminal as Administrator
Right-click PowerShell → Run as Administrator

# macOS: Use sudo or adjust permissions

# Linux: Add user to docker group
sudo usermod -aG docker $USER
# Then log out and back in
```

### Container Issues

#### Container not starting
```bash
# Check logs across all platforms
docker-compose logs -f

# Verify .env file exists with SQL_PASSWORD
# Windows
type .env
# macOS / Linux
cat .env

# Check available disk space (important on Linux)
df -h
```

#### Containers exit/stop immediately
```bash
# Check what went wrong
docker-compose logs

# Rebuild from scratch
docker-compose down -v
docker-compose up --build -d
docker-compose logs -f
```

#### Port 1433 already in use
```bash
# Find what's using the port
# Windows PowerShell
netstat -ano | findstr :1433

# macOS / Linux
lsof -i :1433

# Solution: Change port in docker-compose.yml
# Change: ports: - "1433:1433"
# To:     ports: - "1434:1433"
```

### dbt-Specific Issues

#### dbt debug shows "connection ok" but later commands fail
```bash
# Wait longer for SQL Server to initialize
sleep 30
docker-compose exec -T dbt dbt run

# Or check if SQL Server is fully ready
docker-compose logs sqlserver | tail -n 20
```

#### dbt debug shows connection errors
```bash
# Ensure both containers running
docker-compose ps

# Verify profiles.yml has correct host (should be 'sqlserver')
cat dbt/profiles.yml

# Check network connectivity
docker-compose exec -T dbt ping sqlserver
```

#### "ODBC Driver not found" error
```bash
# Rebuild the dbt container to reinstall drivers
docker-compose down
docker-compose build --no-cache dbt
docker-compose up -d
```

#### Permission denied in dbt/models or dbt/seeds directories
```bash
# Fix permissions (macOS / Linux)
chmod 755 dbt/
chmod 755 dbt/models dbt/seeds dbt/tests

# Windows: Usually not needed, but try running as Administrator
```

### OS-Specific Troubleshooting

#### Windows + WSL2 Issues
```powershell
# Check WSL2 integration
docker-compose version

# If WSL2 integration not working, use native Windows containers
# Or regenerate the context using:
docker context ls
docker context use default

# Restart Docker Desktop: Settings > Restart
```

#### Linux + Docker Issues
```bash
# Check docker is running
systemctl status docker

# Check permissions
docker ps  # Should work without sudo after group config

# Check SELinux if using RHEL/CentOS
getenforce
# If not Permissive, may need: semanage fcontext rules

# Check firewall if running docker on different machine
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=1433/tcp --permanent
```

#### macOS + Disk Space
```bash
# Docker Desktop uses significant disk space
# Check available space
df -h

# Clean up old images/containers
docker system prune -a

# Increase Docker Desktop memory allocation in preferences
```

### General Debugging

```bash
# Comprehensive system check (all platforms)
docker-compose ps
docker-compose logs
docker version
docker-compose version

# Capture full error logs for debugging
docker-compose logs > debug.log 2>&1
# Windows PowerShell
docker-compose logs | Out-File debug.log

# Test connectivity between containers
docker-compose exec -T dbt ping sqlserver
docker-compose exec -T sqlserver ping dbt

# Inspect network
docker network ls
docker network inspect proyecto-dbt_default
```

## Next Steps

After confirming `dbt debug` passes all checks:

1. **Create models** - Add SQL transformation files in `dbt/models/`
2. **Add seeds** - Place CSV data files in `dbt/seeds/`
3. **Define tests** - Create data quality tests in `dbt/schema.yml`
4. **Run transformations** - Execute `dbt run` to build tables
5. **Validate quality** - Run `dbt test` to ensure data integrity

## Deployment Scenarios

### Development (Single Machine)

```bash
# Windows / macOS / Linux (local development)
docker-compose up --build -d
# Start developing with dbt
```

### Staging/Production (On-Premises Linux)

```bash
# 1. SSH into server
ssh user@linux-server.com
cd /opt/proyecto-dbt/

# 2. Pull latest code
git pull origin main

# 3. Build and start with persistent volumes
docker-compose -f docker-compose.yml up --build -d

# 4. Verify
docker-compose ps
docker-compose exec -T dbt dbt debug

# 5. Run pipelines
docker-compose exec -T dbt dbt run
```

**Production Recommendations:**
- Store `.env` in secure location outside git repository
- Use volume backups for persistent `sqlserver_data`
- Configure log rotation for container logs
- Monitor resource usage (CPU, memory, disk)
- Set up automated backups of SQL Server data

```bash
# View resource usage
docker stats

# Backup SQL Server data volume
docker run --rm \
  -v proyecto-dbt_sqlserver_data:/data \
  -v /backup:/backup \
  alpine tar czf /backup/sqlserver-backup-$(date +%Y%m%d).tar.gz -C /data .
```

### Cloud Deployment (AWS/Azure/GCP)

For cloud deployment, modify `docker-compose.yml`:

```yaml
services:
  sqlserver:
    # ... other config ...
    volumes:
      - /mnt/data:/var/opt/mssql  # Use cloud-managed storage
  dbt:
    # ... other config ...
    volumes:
      - /mnt/dbt:/usr/app/dbt_project
```

Then deploy using:
- **AWS ECS** - Push image to ECR, deploy via ECS
- **Azure Container Instances** - Push to ACR, deploy via ACI
- **Kubernetes** - Use helm charts for orchestration

## References

- [dbt Documentation](https://docs.getdbt.com)
- [dbt SQL Server Adapter](https://docs.getdbt.com/reference/warehouse-setups/sqlserver-setup)
- [SQL Server Docker Image](https://hub.docker.com/_/microsoft-mssql-server)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Installation by OS](https://docs.docker.com/get-docker/)

## Support

For issues specific to your OS:

- **Windows**: Check Windows Firewall settings, WSL2 integration status
- **macOS**: Verify Docker Desktop memory allocation, storage space
- **Linux**: Check docker daemon status, user permissions, SELinux (if RHEL/CentOS)

## Notes

- This setup uses the `dbo` schema (SQL Server default)
- SQL Server is initialized with `master` database
- dbt profiles reference the container name `sqlserver` directly (no port remapping needed within container network)
- All credentials are environment-based for security
- The dbt container includes git for managing dbt packages and dependencies
- Same configuration works across all OSs without modification (except sudo on Linux when needed)
