# PostgreSQL Docker Debugging Guide

This guide helps you troubleshoot common issues with your PostgreSQL Docker setup.

## Quick Diagnostic Checklist

### 1. Check Container Status

```bash
# Check if container is running
docker ps

# Check container logs
docker compose logs postgres

# Check container health
docker inspect --format='{{.State.Health.Status}}' <container-name>
```

### 2. Verify Port Binding

```bash
# Linux/Mac
sudo netstat -tlnp | grep 5433
sudo ss -tlnp | grep 5433

# Windows PowerShell
netstat -ano | findstr "5433"
Get-NetTCPConnection -LocalPort 5433
```

Expected output:

```
LISTEN 0  4096  0.0.0.0:5433  0.0.0.0:*  users:(("docker-proxy",pid=XXX,fd=7))
```

### 3. Test Local Connection

```bash
# From the Docker host
psql -h localhost -p 5433 -U postgres -d postgres

# Or using Docker exec
docker exec -it <container-name> psql -U postgres -d postgres

# Test with telnet
telnet localhost 5433
```

### 4. Check Environment Variables

```bash
# View your configuration
cat .env

# Check what container sees
docker exec <container-name> env | grep POSTGRES
```

### 5. Firewall Configuration (Linux)

```bash
# UFW Status
sudo ufw status verbose

# Add rule if needed
sudo ufw allow 5433/tcp
sudo ufw reload

# Check iptables
sudo iptables -L -n | grep 5433
sudo iptables -A INPUT -p tcp --dport 5433 -j ACCEPT
```

### 6. Test Remote Connection

```bash
# From another machine (Linux/Mac)
telnet 203.0.113.10 5433
nc -zv 203.0.113.10 5433

# Windows PowerShell
Test-NetConnection -ComputerName 203.0.113.10 -Port 5433
```

## Common Issues and Solutions

### Issue 1: Connection Timeout

**Symptoms:**

- pgAdmin shows "connection timeout expired"
- Cannot connect remotely but local works

**Causes:**

1. Firewall blocking the port
2. Cloud provider security groups
3. Wrong host/port configuration

**Solutions:**

```bash
# Check if port is listening on all interfaces
sudo ss -tlnp | grep 5433
# Should show 0.0.0.0:5433, not 127.0.0.1:5433

# Open firewall port
sudo ufw allow 5433/tcp

# Test connection
telnet <server-ip> 5433
```

**Cloud Provider Checks:**

- **AWS**: EC2 → Security Groups → Add inbound rule for port 5433
- **Azure**: Network Security Groups → Add inbound rule
- **DigitalOcean**: Networking → Firewall → Add rule
- **Linode**: Firewalls → Add inbound rule
- **Vultr**: Firewall → Add rule

### Issue 2: Authentication Failed

**Symptoms:**

- "password authentication failed for user"
- "FATAL: password authentication failed"

**Solutions:**

```bash
# Verify credentials in .env
cat .env | grep POSTGRES_USER
cat .env | grep POSTGRES_PASSWORD

# Check what PostgreSQL sees
docker exec <container-name> psql -U postgres -c "\du"

# Reset password if needed
docker exec -it <container-name> psql -U postgres -c "ALTER USER postgres PASSWORD 'newpassword';"

# Restart container
docker compose restart
```

### Issue 3: Database Does Not Exist

**Symptoms:**

- "database does not exist"

**Solutions:**

```bash
# List all databases
docker exec <container-name> psql -U postgres -c "\l"

# Create database
docker exec <container-name> psql -U postgres -c "CREATE DATABASE mydatabase;"

# Or connect to default 'postgres' database
# In pgAdmin, use "postgres" as the database name
```

### Issue 4: Volume Mount Issues (PostgreSQL 18+)

**Symptoms:**

```
Error: PostgreSQL Database directory appears to contain a database
Counter to that, there appears to be PostgreSQL data in:
  /var/lib/postgresql/data (unused mount/volume)
```

**Solution:**

Update docker-compose.yml to use `/var/lib/postgresql` instead of `/var/lib/postgresql/data`:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql
```

Then recreate:

```bash
docker compose down -v
docker compose up -d
```

### Issue 5: Container Keeps Restarting

**Symptoms:**

- Container exits immediately
- Status shows "Restarting"

**Solutions:**

```bash
# Check logs for errors
docker compose logs postgres

# Common causes:
# 1. Port already in use
sudo netstat -tlnp | grep 5433

# 2. Volume permissions
docker volume inspect postgres_data_volume

# 3. Invalid environment variables
cat .env

# Remove and recreate
docker compose down -v
docker compose up -d
```

### Issue 6: Permission Denied

**Symptoms:**

- "could not open file: Permission denied"
- "initdb: error: could not create directory"

**Solutions:**

```bash
# Check volume ownership
docker volume inspect postgres_data_volume

# Recreate with correct permissions
docker compose down -v
docker compose up -d

# Or fix permissions manually
docker exec -u root <container-name> chown -R postgres:postgres /var/lib/postgresql
```

## Detailed Connection Testing

### Test 1: Container Connectivity

```bash
# Check if PostgreSQL is accepting connections inside container
docker exec <container-name> pg_isready -U postgres

# Should output: accepting connections
```

### Test 2: Local Host Connection

```bash
# From Docker host machine
psql -h localhost -p 5433 -U postgres -d postgres

# If this fails, issue is with Docker setup
```

### Test 3: Network Interface Binding

```bash
# Check what interfaces Docker is listening on
sudo netstat -tlnp | grep 5433

# Should show:
# 0.0.0.0:5433 (all IPv4 interfaces)
# :::5433 (all IPv6 interfaces)

# NOT:
# 127.0.0.1:5433 (localhost only)
```

### Test 4: Remote Connection

```bash
# From your local machine
telnet 203.0.113.10 5433

# Success: Connected to 203.0.113.10
# Failure: Connection refused/timeout → firewall issue
```

### Test 5: PostgreSQL Authentication

```bash
# Test connection with psql
PGPASSWORD='your_password' psql -h 203.0.113.10 -p 5433 -U postgres -d postgres

# Or connection string
psql "postgresql://postgres:your_password@203.0.113.10:5433/postgres"
```

## Diagnostic Commands

### Container Information

```bash
# Full container details
docker inspect <container-name>

# Network settings
docker inspect --format='{{.NetworkSettings.Ports}}' <container-name>

# Environment variables
docker inspect --format='{{.Config.Env}}' <container-name>

# Health check
docker inspect --format='{{.State.Health}}' <container-name>
```

### PostgreSQL Internals

```bash
# Connect to container shell
docker exec -it <container-name> bash

# Check PostgreSQL config
cat /var/lib/postgresql/data/postgresql.conf | grep listen_addresses

# Check pg_hba.conf
cat /var/lib/postgresql/data/pg_hba.conf

# PostgreSQL logs
docker exec <container-name> tail -f /var/lib/postgresql/data/log/postgresql-*.log
```

### Network Diagnostics

```bash
# Check all listening ports
sudo netstat -tlnp

# Check Docker network
docker network ls
docker network inspect docker-pgsql_default

# Test DNS resolution
nslookup 203.0.113.10

# Test route
traceroute 203.0.113.10  # Linux/Mac
tracert 203.0.113.10     # Windows
```

## Configuration Verification

### Correct docker-compose.yml for Remote Access

```yaml
services:
  postgres:
    image: postgres:latest
    container_name: ${CONTAINER_NAME}
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_INITDB_ARGS: ${POSTGRES_INITDB_ARGS}
      POSTGRES_HOST_AUTH_METHOD: md5
    ports:
      - "${POSTGRES_PORT}:5432" # All interfaces
    volumes:
      - postgres_data:/var/lib/postgresql
    user: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Required .env Variables

```env
CONTAINER_NAME=postgres-container
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password
POSTGRES_INITDB_ARGS=
POSTGRES_PORT=5433
VOLUME_NAME=postgres_data_volume
```

## pgAdmin Connection Settings

```
Host name/address: 203.0.113.10
Port: 5433
Maintenance database: postgres
Username: postgres
Password: <from .env file>
```

## Reset Everything

If all else fails, completely reset:

```bash
# Stop and remove everything
docker compose down -v

# Remove container
docker rm -f <container-name>

# Remove volume
docker volume rm postgres_data_volume

# Recreate
docker compose up -d

# Wait for initialization
docker compose logs -f
```

## Enable Debug Logging

Add to docker-compose.yml:

```yaml
environment:
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  POSTGRES_INITDB_ARGS: ${POSTGRES_INITDB_ARGS}
  POSTGRES_HOST_AUTH_METHOD: md5
  POSTGRES_EXTRA_OPTS: "-c log_connections=on -c log_disconnections=on -c log_statement=all"
```

Then check logs:

```bash
docker compose logs -f postgres
```

## Get Help

If you're still stuck, gather this information:

```bash
# System info
uname -a
docker --version
docker compose version

# Container state
docker ps -a
docker compose logs postgres

# Network state
sudo ss -tlnp | grep 5433
sudo ufw status

# Environment
cat .env

# Connection test
telnet localhost 5433
```

Share this output when asking for help on forums or GitHub issues.

## Useful Resources

- [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [PostgreSQL Client Authentication](https://www.postgresql.org/docs/current/client-authentication.html)
