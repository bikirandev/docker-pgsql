# Docker PostgreSQL Setup

A production-ready Docker Compose configuration for PostgreSQL with security best practices and health monitoring.

## Features

- 🐘 Latest PostgreSQL image
- 🔒 Localhost-only binding for enhanced security
- 🔄 Automatic restart on failure
- 💾 Persistent data storage with named volumes
- 🏥 Built-in health checks
- ⚙️ Fully configurable via environment variables

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed
- [Docker Compose](https://docs.docker.com/compose/install/) installed
- Basic knowledge of PostgreSQL

## Quick Start

### 1. Clone or Download

```bash
git clone https://github.com/bikirandev/docker-pgsql
cd docker-pgsql
```

### 2. Configure Environment

Copy the example environment file and customize it:

```bash
cp .env.example .env
```

Edit `.env` with your preferred values:

```env
CONTAINER_NAME=postgres-container
POSTGRES_DB=mydatabase
POSTGRES_USER=myuser
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_INITDB_ARGS=
POSTGRES_PORT=5432
VOLUME_NAME=postgres_data_volume
```

### 3. Start PostgreSQL

```bash
docker-compose up -d
```

### 4. Verify Container is Running

```bash
docker ps
```

You should see your PostgreSQL container running with a healthy status.

## Environment Variables

| Variable               | Description                                    | Default                | Required |
| ---------------------- | ---------------------------------------------- | ---------------------- | -------- |
| `CONTAINER_NAME`       | Name of the Docker container                   | `postgres-container`   | Yes      |
| `POSTGRES_DB`          | Default database name to create                | `mydatabase`           | Yes      |
| `POSTGRES_USER`        | PostgreSQL superuser name                      | `myuser`               | Yes      |
| `POSTGRES_PASSWORD`    | PostgreSQL superuser password                  | `mypassword`           | Yes      |
| `POSTGRES_INITDB_ARGS` | Additional initialization arguments            | _(empty)_              | No       |
| `POSTGRES_PORT`        | Host port to expose PostgreSQL                 | `5432`                 | Yes      |
| `VOLUME_NAME`          | Name of the Docker volume for data persistence | `postgres_data_volume` | Yes      |

### POSTGRES_INITDB_ARGS Examples

```env
# Set encoding and locale
POSTGRES_INITDB_ARGS=--encoding=UTF8 --locale=en_US.UTF-8

# Enable data checksums
POSTGRES_INITDB_ARGS=--data-checksums
```

## Docker Compose Commands

### Start the Container

```bash
# Start in detached mode
docker-compose up -d

# Start with logs visible
docker-compose up
```

### Stop the Container

```bash
docker-compose stop
```

### Stop and Remove Container

```bash
docker-compose down
```

### Stop and Remove Container + Volume (⚠️ Deletes all data)

```bash
docker-compose down -v
```

### View Logs

```bash
# Follow logs in real-time
docker-compose logs -f

# View last 100 lines
docker-compose logs --tail=100
```

### Restart the Container

```bash
docker-compose restart
```

## Connecting to PostgreSQL

### From Host Machine

#### Using psql (Command Line)

```bash
psql -h localhost -p 5432 -U myuser -d mydatabase
```

#### Connection String

```
postgresql://myuser:mypassword@localhost:5432/mydatabase
```

#### Connection Parameters

- **Host:** `localhost` or `127.0.0.1`
- **Port:** `5432` (or your custom port from `.env`)
- **Database:** Value from `POSTGRES_DB`
- **Username:** Value from `POSTGRES_USER`
- **Password:** Value from `POSTGRES_PASSWORD`

### From Another Docker Container

If you have other containers in the same Docker Compose network:

```yaml
services:
  your-app:
    # ... other configuration
    environment:
      DATABASE_URL: postgresql://myuser:mypassword@postgres:5432/mydatabase
```

**Note:** Use the service name (`postgres`) as the hostname, not `localhost`.

### Using GUI Tools

Popular PostgreSQL GUI tools that work with this setup:

- **pgAdmin**: [https://www.pgadmin.org/](https://www.pgadmin.org/)
- **DBeaver**: [https://dbeaver.io/](https://dbeaver.io/)
- **DataGrip**: [https://www.jetbrains.com/datagrip/](https://www.jetbrains.com/datagrip/)
- **TablePlus**: [https://tableplus.com/](https://tableplus.com/)

## Health Check

The container includes an automatic health check that:

- Runs every 10 seconds
- Uses `pg_isready` to verify PostgreSQL is accepting connections
- Allows 5 retries before marking as unhealthy
- Has a 5-second timeout per check

Check container health status:

```bash
docker inspect --format='{{.State.Health.Status}}' <container-name>
```

## Data Persistence

PostgreSQL data is stored in a named Docker volume (`postgres_data_volume` by default). This ensures your data persists even if the container is removed.

### Backup Data

```bash
# Create a backup
docker exec -t <container-name> pg_dumpall -c -U myuser > backup.sql

# Or backup a specific database
docker exec -t <container-name> pg_dump -U myuser mydatabase > mydatabase_backup.sql
```

### Restore Data

```bash
# Restore from backup
cat backup.sql | docker exec -i <container-name> psql -U myuser

# Or restore specific database
cat mydatabase_backup.sql | docker exec -i <container-name> psql -U myuser -d mydatabase
```

### Volume Location

To find where Docker stores the volume on your system:

```bash
docker volume inspect postgres_data_volume
```

## Security Considerations

### Current Security Features

✅ **Localhost-only binding**: PostgreSQL is only accessible from the host machine, not from external networks.

✅ **Non-root user**: Container runs as the `postgres` user for enhanced security.

✅ **Health monitoring**: Automatic health checks ensure the database is running properly.

### Additional Security Recommendations

1. **Strong Password**: Use a strong, unique password in your `.env` file
2. **Environment File**: Never commit `.env` to version control
3. **SSL/TLS**: For production, configure SSL certificates
4. **Firewall**: Ensure your host firewall is properly configured
5. **Updates**: Regularly update to the latest PostgreSQL image

### .gitignore

Add these lines to your `.gitignore`:

```
.env
*.sql
backup/
```

## Troubleshooting

### Container Won't Start

Check logs for errors:

```bash
docker-compose logs postgres
```

### Permission Denied Errors

Ensure Docker has permission to create volumes:

```bash
docker-compose down -v
docker-compose up -d
```

### Cannot Connect to Database

1. Verify container is running: `docker ps`
2. Check health status: `docker inspect --format='{{.State.Health.Status}}' <container-name>`
3. Verify port is not in use: `netstat -an | grep 5432` (Windows) or `lsof -i :5432` (Linux/Mac)
4. Check environment variables in `.env` file

### Port Already in Use

If port 5432 is already in use, change `POSTGRES_PORT` in your `.env` file:

```env
POSTGRES_PORT=5433
```

Then restart:

```bash
docker-compose down
docker-compose up -d
```

## Advanced Configuration

### Custom PostgreSQL Configuration

Create a custom `postgresql.conf` file and mount it:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
  - ./postgresql.conf:/etc/postgresql/postgresql.conf
```

### Initialization Scripts

To run SQL scripts on first startup, mount them to `/docker-entrypoint-initdb.d/`:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
  - ./init-scripts:/docker-entrypoint-initdb.d
```

### Multiple Databases

Create an initialization script `init-scripts/create-databases.sh`:

```bash
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE DATABASE app_db;
    CREATE DATABASE test_db;
EOSQL
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Support

If you encounter any issues or have questions:

1. Check the [Troubleshooting](#troubleshooting) section
2. Review [PostgreSQL Docker Hub documentation](https://hub.docker.com/_/postgres)
3. Open an issue in this repository

## Resources

- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [PostgreSQL Docker Image](https://hub.docker.com/_/postgres)

---

**Author:** Kumar Bishojit Paul  
**Last Updated:** December 21, 2025
