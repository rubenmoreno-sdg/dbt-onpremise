# dbt On-Premises Installation Guide

**Target Environment:**
- **OS:** Ubuntu 24.04.4 LTS / Linux 6.8.0-117-generic
- **Docker:** 29.5.2, build 79eb04c
- **SQL Server:** Microsoft SQL Server 2022 (RTM-CU21) - Remote (DataCenter AYESA, Windows Server 2019)
- **SQL Server Host/Instance:** 10.200.207.194/TBXSQLAPPDES
- **dbt Deployment:** Docker container on Ubuntu

## Prerequisites

Before starting, ensure you have:

1. **SSH Access** to the Ubuntu 24.04 server
2. **Docker & Docker Compose** already installed (version 29.5.2+)
3. **Network Access** to the remote SQL Server at `10.200.207.194:1433`
4. **SQL Server Credentials** (username and password provided by the client)
5. **Git** installed on the server (optional, for version control)

### Verify Prerequisites

```bash
# Check Ubuntu version
lsb_release -a
# Expected: Ubuntu 24.04.4 LTS

# Check Docker
docker --version
# Expected: Docker version 29.5.2 or higher

# Check Docker Compose
docker-compose --version
# Expected: Docker Compose version 1.29.x or higher

# Check network connectivity to SQL Server
nc -zv 10.200.207.194 1433
# Expected: Connection to 10.200.207.194 port 1433 [tcp/*] succeeded!
```

---

## Step 1: Clone or Download the Project

### Option A: Using Git (Recommended)

```bash
# Navigate to your desired directory
cd /opt/

# Clone the repository
git clone https://github.com/rubenmoreno-sdg/dbt-onpremise.git
cd dbt-onpremise

# Verify files
ls -la
```

### Option B: Manual Download

```bash
# Create directory
mkdir -p /opt/dbt-onpremise
cd /opt/dbt-onpremise

# Copy all files from the repository here:
# - docker-compose.yml
# - Dockerfile.dbt
# - README.md
# - INSTALL_ONPREM.md (this file)
# - dbt/ (entire directory)
```

---

## Step 2: Configure Environment Variables

### Create .env File

```bash
# Create the .env file in the project root
nano /opt/dbt-onpremise/.env
```

Add the following (or set these as system environment variables):

```env
# SQL Server Connection (for dbt remote connection)
CLIENT_SQL_PASSWORD=YourSecurePasswordHere
```

**Security Notes:**
- Do NOT commit `.env` to Git
- Use a secure password manager or secret management tool in production
- Restrict file permissions: `chmod 600 .env`

### Verify .env Permissions

```bash
chmod 600 .env
ls -la .env
# Expected: -rw------- (only owner can read/write)
```

---

## Step 3: Update dbt Profiles Configuration

Edit the dbt profiles to point to the remote SQL Server:

```bash
nano dbt/profiles.yml
```

Replace with the client's connection details:

```yaml
default:
  outputs:
    dev:
      type: sqlserver
      driver: 'ODBC Driver 18 for SQL Server'
      host: 10.200.207.194
      port: 1433
      user: your_username          # Provided by client
      password: "{{ env_var('CLIENT_SQL_PASSWORD') }}"
      database: master             # or specify target DB
      schema: dbo
      encrypt: true
      trust_cert: true
  target: dev
```

**Key Configuration Points:**
- `host: 10.200.207.194` - Remote SQL Server IP
- `user: your_username` - SQL Server login (ask client)
- `password` - Retrieved from `CLIENT_SQL_PASSWORD` env var
- `database: master` - Default system database (can be changed)
- `encrypt: true` / `trust_cert: true` - Enable SSL (may need adjustment based on client's SQL Server config)

---

## Step 4: Build and Start Containers

### Set Environment Variable

```bash
# Export the password (replace with actual password provided by client)
export CLIENT_SQL_PASSWORD="YourSecurePasswordHere"

# Verify it's set
echo $CLIENT_SQL_PASSWORD
```

### Build Docker Image

```bash
cd /opt/dbt-onpremise

# Build the dbt container
docker-compose build
```

**Expected Output:**
```
[+] Building 45.2s (14/14) FINISHED
 => [dbt-core internal] load build definition from Dockerfile.dbt
 => [dbt-core] FROM python:3.11-slim-bullseye
 => [dbt-core] RUN apt-get update && apt-get install -y curl gnupg git build-essential
 => ... (ODBC driver installation)
 => [dbt-core] RUN pip install dbt-core dbt-sqlserver
Successfully tagged proyecto-dbt_dbt:latest
```

### Start Only the dbt Container (No Local SQL Server)

Since SQL Server is remote, do NOT start the `sqlserver` container. Edit `docker-compose.yml` to remove or comment out the sqlserver service:

```bash
# Option 1: Edit docker-compose.yml and comment out the sqlserver service
nano docker-compose.yml
```

Comment out or remove the `sqlserver` service section:

```yaml
# version: '3.8'
# 
# # DO NOT RUN LOCAL SQL SERVER - Using remote server at 10.200.207.194
# services:
#   sqlserver:
#     # (commented out - not needed)
```

Then update the `dbt` service to remove the `depends_on`:

```yaml
version: '3.8'

services:
  dbt:
    build:
      context: .
      dockerfile: Dockerfile.dbt
    container_name: dbt_core_onpremise
    volumes:
      - ./dbt:/usr/app/dbt_project
    environment:
      - DBT_PROFILES_DIR=/usr/app/dbt_project
      - CLIENT_SQL_PASSWORD=${CLIENT_SQL_PASSWORD}
    # Remove: depends_on:
    #         - sqlserver
    command: tail -f /dev/null
    stdin_open: true
    tty: true
```

### Start the dbt Container

```bash
# Set environment variable (if not already set)
export CLIENT_SQL_PASSWORD="YourSecurePasswordHere"

# Start only the dbt container
docker-compose up -d dbt

# Verify it's running
docker-compose ps
```

**Expected Output:**
```
NAME              COMMAND               SERVICE   STATUS     PORTS
dbt_core_onpremise   "tail -f /dev/null"   dbt       Up 1 min
```

---

## Step 5: Verify dbt Configuration with debug

### Run dbt Debug

```bash
# Test connection to remote SQL Server
docker-compose exec -T dbt dbt debug
```

**Successful Output (all checks passed):**
```
Running with dbt=1.11.11
dbt version: 1.11.11
python version: 3.11.13
os info: Linux-6.8.0-117-generic-x86_64-with-glibc2.31

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
  server: 10.200.207.194
  port: 1433
  database: master
  schema: dbo
  UID: your_username
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

### Troubleshooting dbt Debug Errors

#### Error: "Connection test: [FAILED connection refused]"
- **Cause:** Cannot reach SQL Server on `10.200.207.194:1433`
- **Solution:**
  ```bash
  # Test network connectivity
  docker-compose exec -T dbt nc -zv 10.200.207.194 1433
  
  # If fails: check firewall rules, VPN, or ask client to open port 1433
  ```

#### Error: "Login failed for user 'your_username'"
- **Cause:** Incorrect credentials or user doesn't exist
- **Solution:**
  ```bash
  # Verify username and password with client
  # Check if user is created in SQL Server and has the right permissions
  ```

#### Error: "ODBC Driver not found"
- **Cause:** ODBC drivers not installed in container
- **Solution:**
  ```bash
  # Rebuild container
  docker-compose build --no-cache dbt
  docker-compose up -d dbt
  ```

#### Error: "encrypt=true but server doesn't support encryption"
- **Solution:** Set `encrypt: false` in `dbt/profiles.yml`:
  ```yaml
  encrypt: false
  trust_cert: false
  ```

---

## Step 6: Run dbt Commands

Once `dbt debug` shows all checks passed:

### Load Seed Data

```bash
docker-compose exec -T dbt dbt seed
```

### Run Models

```bash
docker-compose exec -T dbt dbt run
```

### Test Data Quality

```bash
docker-compose exec -T dbt dbt test
```

### Generate Documentation

```bash
docker-compose exec -T dbt dbt docs generate
```

### Check Models

```bash
docker-compose exec -T dbt dbt compile --select model_name
```

---

## Step 7: Container Management

### View Logs

```bash
# Real-time logs
docker-compose logs -f dbt

# Last 50 lines
docker-compose logs --tail=50 dbt

# Search logs for errors
docker-compose logs dbt | grep -i error
```

### Shell Access

```bash
# Interactive shell in dbt container
docker-compose exec dbt bash

# Run a single command
docker-compose exec -T dbt ls -la dbt/models/
```

### Query SQL Server from Container

```bash
# Test SQL connection using sqlcmd
docker-compose exec -T dbt sqlcmd -S 10.200.207.194 -U your_username -P "$CLIENT_SQL_PASSWORD" -Q "SELECT @@VERSION"
```

### Stop / Restart Container

```bash
# Stop
docker-compose down

# Restart
docker-compose up -d dbt

# Restart with fresh build
docker-compose up -d --build dbt
```

---

## Step 8: Production Setup & Best Practices

### Store .env Securely

**Do NOT commit `.env` to Git:**

```bash
# Verify .env is in .gitignore
cat .gitignore | grep ".env"
# Expected: .env

# If not, add it:
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Ensure .env is ignored"
```

### Use Systemd Service (Optional)

Create a systemd service to auto-start the dbt container on reboot:

```bash
# Create service file
sudo nano /etc/systemd/system/dbt-onpremise.service
```

Add:

```ini
[Unit]
Description=dbt On-Premises Service
After=docker.service
Requires=docker.service

[Service]
Type=simple
WorkingDirectory=/opt/dbt-onpremise
Environment="CLIENT_SQL_PASSWORD=YourSecurePassword"
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

# Check status
sudo systemctl status dbt-onpremise.service
```

### Monitor Resource Usage

```bash
# Monitor container resources
docker stats dbt_core_onpremise

# Expected output:
# CONTAINER           CPU %     MEM USAGE / LIMIT   MEM %
# dbt_core_onpremise  0.5%      450MiB / 8GiB       5.6%
```

### Backup dbt Project

```bash
# Backup dbt directory
tar -czf dbt-backup-$(date +%Y%m%d).tar.gz dbt/

# Backup entire project
tar -czf dbt-project-backup-$(date +%Y%m%d).tar.gz \
  docker-compose.yml \
  Dockerfile.dbt \
  .gitignore \
  dbt/
```

---

## Step 9: Scheduled Runs (Optional)

### Using Docker + Cron

Create a run script:

```bash
# Create script
nano /opt/dbt-onpremise/run_dbt.sh
```

Add:

```bash
#!/bin/bash
set -e

cd /opt/dbt-onpremise
export CLIENT_SQL_PASSWORD=$(cat /path/to/secure/password.txt)

# Run dbt pipeline
docker-compose exec -T dbt dbt run
docker-compose exec -T dbt dbt test

# Log output
echo "dbt run completed at $(date)" >> /var/log/dbt-run.log
```

Make executable and add to crontab:

```bash
chmod +x /opt/dbt-onpremise/run_dbt.sh

# Edit crontab
crontab -e

# Add line (runs daily at 2 AM):
0 2 * * * /opt/dbt-onpremise/run_dbt.sh
```

---

## Troubleshooting Common Issues

### Issue: "Permission denied" when running docker-compose

**Solution:** Add your user to the docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
# Logout and login for changes to take effect
```

### Issue: Container fails to start after reboot

**Solution:** Check docker daemon status:

```bash
# Check if docker is running
sudo systemctl status docker

# Restart docker
sudo systemctl restart docker

# Check container logs
docker-compose logs dbt
```

### Issue: "No space left on device"

**Solution:** Check disk usage:

```bash
# Check disk space
df -h

# Clean up old Docker images/containers
docker system prune -a

# Check Docker volume space
docker volume ls
docker volume prune
```

### Issue: SQL Server connection times out

**Solution:** Verify network and firewall:

```bash
# Check if port 1433 is reachable
sudo apt-get install netcat-openbsd
nc -zv 10.200.207.194 1433

# Check firewall rules
sudo ufw status
# If needed: sudo ufw allow 1433/tcp

# Test from container
docker-compose exec -T dbt nc -zv 10.200.207.194 1433
```

---

## References

- [dbt Documentation](https://docs.getdbt.com)
- [dbt SQL Server Adapter](https://docs.getdbt.com/reference/warehouse-setups/sqlserver-setup)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [SQL Server ODBC Drivers](https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server)

---

## Support & Next Steps

### For Development Team

1. **Initialize dbt project:** Define models in `dbt/models/`
2. **Add seed data:** Place CSV files in `dbt/seeds/`
3. **Create tests:** Add quality tests in `dbt/schema.yml`
4. **Version control:** Commit changes to Git (exclude `.env`)
5. **CI/CD integration:** Set up GitHub Actions for automated testing

### For Client/Operations

- Monitor container health and resource usage
- Set up alerts for failed dbt runs
- Schedule regular backups of SQL Server data
- Review dbt documentation and lineage regularly
- Plan for SQL Server maintenance windows

---

## Quick Reference Commands

```bash
# Navigate to project
cd /opt/dbt-onpremise

# Set credentials
export CLIENT_SQL_PASSWORD="your_password"

# Build image
docker-compose build

# Start container
docker-compose up -d dbt

# Run dbt commands
docker-compose exec -T dbt dbt debug
docker-compose exec -T dbt dbt run
docker-compose exec -T dbt dbt test

# View logs
docker-compose logs -f dbt

# Stop container
docker-compose down

# Interactive shell
docker-compose exec dbt bash
```

---

**Last Updated:** June 2026  
**Maintained By:** Development Team  
**Support:** For issues, check the main [README.md](README.md) or contact the development team.
