# dbt On-Premises Deployment

This project deploys **dbt (data build tool)** on Ubuntu 24.04 LTS connected to a **remote SQL Server 2022** instance (Microsoft SQL Server 2022 RTM-CU21) hosted at DataCenter AYESA (Windows Server 2019).

**Target Environment:**
- **Server OS:** Ubuntu 24.04.4 LTS (Linux 6.8.0-117-generic)
- **Container Runtime:** Docker 29.5.2
- **Database:** Microsoft SQL Server 2022 (Remote at 10.200.207.194/TBXSQLAPPDES)
- **dbt Deployment:** Docker container

---

## Overview

This is a **production-ready** dbt deployment for on-premises environments. Key features:

- **dbt-core** container with SQL Server adapter (dbt-sqlserver)
- **ODBC drivers** pre-installed for SQL Server connectivity
- **Remote SQL Server** connection (no local database required)
- **Pre-configured** for the client's infrastructure (10.200.207.194/TBXSQLAPPDES)
- **Simple Docker Compose** setup for easy deployment and scaling

---

## Project Structure

```
dbt-onpremise/
├── docker-compose.yml          # Container orchestration
├── Dockerfile.dbt              # dbt container image
├── .env                        # Environment variables (SQL password)
├── .gitignore                  # Git ignore rules
├── README.md                   # This file
├── INSTALL_ONPREM.md          # Detailed on-premises installation guide
├── dbt/                        # dbt project directory
│   ├── dbt_project.yml         # dbt configuration
│   ├── profiles.yml            # Connection configuration
│   ├── .user.yml               # dbt user settings
│   ├── models/                 # SQL transformation models
│   ├── seeds/                  # CSV seed data
│   ├── tests/                  # Data quality tests
│   └── target/                 # Generated dbt artifacts (auto-created)
└── INSTALL_ONPREM.md          # Step-by-step on-premises guide
```

---

## Prerequisites

Before deploying, ensure you have:

1. **Infrastructure Requirements**
   - Ubuntu 24.04 LTS server (or compatible Linux distro)
   - Docker 29.5.2 or higher installed
   - Docker Compose installed
   - SSH access to the server

2. **Network Requirements**
   - TCP port `1433` open to the remote SQL Server at `10.200.207.194`
   - Firewall rules configured to allow outbound connections to DataCenter AYESA
   - VPN access (if required by the organization)

3. **SQL Server Access**
   - Valid SQL Server credentials (username and password)
   - User must have permissions to:
     - Create tables and schemas
     - Execute SELECT/INSERT/UPDATE/DELETE queries
     - Access the target database (typically `master` or a dedicated dbt database)

**Verify Prerequisites:**

```bash
# Check Docker
docker --version

# Check Docker Compose
docker-compose --version

# Test SQL Server connectivity
nc -zv 10.200.207.194 1433
# Expected: Connection to 10.200.207.194 port 1433 succeeded!
```

---

## Quick Start (5 minutes)

### 1. SSH to Your Server

```bash
ssh user@your-server.local
cd /opt/
```

### 2. Clone the Repository

```bash
git clone https://github.com/rubenmoreno-sdg/dbt-onpremise.git
cd dbt-onpremise
```

### 3. Configure Credentials

```bash
# Edit .env file
nano .env

# Add your SQL Server password:
# CLIENT_SQL_PASSWORD=your_password_here

# Set secure permissions
chmod 600 .env
```

### 4. Update dbt Profiles

```bash
nano dbt/profiles.yml

# Update username in the file
# (password is read from CLIENT_SQL_PASSWORD env var)
```

### 5. Build and Start

```bash
export CLIENT_SQL_PASSWORD="your_password_here"
docker-compose build
docker-compose up -d dbt
```

### 6. Test Connection

```bash
docker-compose exec -T dbt dbt debug
```

If all checks pass, you're ready to use dbt!

---

## Common dbt Commands

Once `dbt debug` shows all checks passed:

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
```

---

## Container Management

```bash
# View container status
docker-compose ps

# View container logs
docker-compose logs -f dbt

# Stop container
docker-compose down

# Restart container
docker-compose up -d dbt

# Interactive shell in dbt container
docker-compose exec dbt bash

# Query SQL Server from container
docker-compose exec -T dbt sqlcmd -S 10.200.207.194 -U your_username -P "$CLIENT_SQL_PASSWORD" -Q "SELECT @@VERSION"
```

---

## Configuration Details

### docker-compose.yml

**dbt Service:**
- Build: `Dockerfile.dbt`
- Container: `dbt_core_onpremise`
- Volume: `./dbt:/usr/app/dbt_project` (mounts local dbt directory)
- Environment:
  - `DBT_PROFILES_DIR=/usr/app/dbt_project` (profiles location)
  - `CLIENT_SQL_PASSWORD` (from .env for database auth)
- Command: `tail -f /dev/null` (keeps container alive)

Note: The `sqlserver` service is NOT included (uses remote SQL Server).

### Dockerfile.dbt

Multi-stage build that:
1. Starts with Python 3.11 slim image
2. Installs system dependencies: curl, gnupg, git, build-essential
3. Adds Microsoft ODBC drivers for SQL Server
4. Installs dbt-core and dbt-sqlserver adapter
5. Sets working directory to `/usr/app/dbt_project`

### dbt/profiles.yml

```yaml
default:
  outputs:
    dev:
      type: sqlserver
      driver: 'ODBC Driver 18 for SQL Server'
      host: 10.200.207.194          # Remote SQL Server IP
      port: 1433
      user: your_username           # SQL Server login
      password: "{{ env_var('CLIENT_SQL_PASSWORD') }}"
      database: master              # Target database
      schema: dbo
      encrypt: true
      trust_cert: true
  target: dev
```

---

## Troubleshooting

### "Connection refused" when running dbt debug

```bash
# Test network connectivity to SQL Server
docker-compose exec -T dbt nc -zv 10.200.207.194 1433

# If fails:
# - Check firewall rules
# - Verify VPN is connected
# - Ask client to confirm port 1433 is open
```

### "Login failed for user" error

```bash
# Verify credentials with client
# Ensure user exists in SQL Server and has proper permissions
```

### "ODBC Driver not found"

```bash
# Rebuild the container
docker-compose build --no-cache dbt
docker-compose up -d dbt
```

### "No space left on device"

```bash
# Check disk space
df -h

# Clean up Docker
docker system prune -a
```

### Container fails to start after reboot

```bash
# Check docker daemon
sudo systemctl status docker
sudo systemctl restart docker

# Restart dbt container
docker-compose up -d dbt
```

---

## Production Setup

### Store Credentials Securely

**Do NOT commit `.env` to Git:**

```bash
# Verify .env is in .gitignore
cat .gitignore | grep ".env"

# Set file permissions
chmod 600 .env
```

### Auto-start with systemd (Optional)

Create `/etc/systemd/system/dbt-onpremise.service`:

```ini
[Unit]
Description=dbt On-Premises Service
After=docker.service

[Service]
Type=simple
WorkingDirectory=/opt/dbt-onpremise
Environment="CLIENT_SQL_PASSWORD=your_password"
ExecStart=/usr/bin/docker-compose up
ExecStop=/usr/bin/docker-compose down
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable dbt-onpremise.service
sudo systemctl start dbt-onpremise.service
```

### Monitor Resource Usage

```bash
docker stats dbt_core_onpremise
```

### Scheduled Runs with Cron

```bash
# Create script: /opt/dbt-onpremise/run_dbt.sh
#!/bin/bash
cd /opt/dbt-onpremise
export CLIENT_SQL_PASSWORD=$(cat /path/to/password.txt)
docker-compose exec -T dbt dbt run
docker-compose exec -T dbt dbt test
```

Add to crontab (runs daily at 2 AM):

```bash
crontab -e
# Add: 0 2 * * * /opt/dbt-onpremise/run_dbt.sh
```

---

## Detailed Documentation

For step-by-step installation instructions with troubleshooting and advanced configuration, see [INSTALL_ONPREM.md](INSTALL_ONPREM.md).

---

## Quick Reference

```bash
# Build image
docker-compose build

# Start container
docker-compose up -d dbt

# Run dbt command
docker-compose exec -T dbt dbt run

# View logs
docker-compose logs -f dbt

# Stop container
docker-compose down

# Shell access
docker-compose exec dbt bash
```

---

## References

- [dbt Documentation](https://docs.getdbt.com)
- [dbt SQL Server Adapter](https://docs.getdbt.com/reference/warehouse-setups/sqlserver-setup)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## Support

- **Server:** Ubuntu 24.04 LTS
- **Docker:** 29.5.2+
- **SQL Server:** Microsoft SQL Server 2022 (10.200.207.194/TBXSQLAPPDES)
- **Issues:** Check [INSTALL_ONPREM.md](INSTALL_ONPREM.md) for detailed troubleshooting

---

**Last Updated:** June 2026  
**Target:** On-Premises Deployment (Ubuntu 24.04 + Remote SQL Server 2022)
