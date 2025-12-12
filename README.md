# MySQL Database Observability with Grafana and Alloy

> **TESTING AND DEMONSTRATION PURPOSES ONLY**
> 
> This repository is designed for testing, learning, and demonstration purposes. It is **NOT intended for production use**. The configurations use default passwords, simplified security settings, and are optimized for ease of setup rather than security or performance.

This repository provides two main use cases for MySQL database observability:

1. **[Database Load Testing & Performance Monitoring](#use-case-1-database-load-testing--performance-monitoring)** - Generate synthetic database load and monitor performance
2. **[Grafana Database Migration Testing](#use-case-2-grafana-database-migration-testing)** - Test Grafana upgrades using MySQL as the backend database

Both use cases leverage Grafana Alloy for integrating with database observability app in Grafana Cloud.

## Architecture Components

- MySQL database
- Grafana Enterprise
- Grafana Alloy: Send db metrics and logs to Grafana Cloud
- Load Testing Tools: Multiple containers to generate database activity and errors

## Prerequisites

- Docker and Docker Compose
- Grafana Cloud account with:
  - Prometheus endpoint for metrics
  - Loki endpoint for logs
  - Cloud Access Policy token with metrics:write and logs:write permissions

## Expected Outcome

After performing the steps in either of the use cases below, your MySQL database should appear in the Grafana Cloud Database Observability app:

1. **Access Grafana Cloud**: Log into your Grafana Cloud instance
2. **Navigate to Database Observability**: Go to the Database Observability app
3. **View Queries Overview**: Your MySQL database should appear in the [Queries Overview dashboard](https://grafana.com/docs/grafana-cloud/monitor-applications/database-observability/navigate-queries-overview/)
4. **Monitor Activity**: You should see:
   - Query metrics and performance data
   - Load test activity (if running stress tests)
   - Database migration activity (during Grafana upgrades)
   - Real-time query samples and wait events

---

# Use Case 1: Database Load Testing & Performance Monitoring

Use to visualize testing database performance, monitoring query patterns, and validating observability setups in dbO11y app

## Setup for Load Testing

### 1. Clone and Setup

```bash
git clone https://github.com/lshippy/db-observability-demo
cd db-observability-demo
```

### 2. Configure Environment Variables

Create a `.env` file with your Grafana Cloud credentials:

```bash
# Grafana Cloud Configuration
GCLOUD_HOSTED_METRICS_URL=https://prometheus-prod-XX-prod-XX-XX.grafana.net/api/prom/push
GCLOUD_HOSTED_METRICS_ID=123456
GCLOUD_HOSTED_LOGS_URL=https://logs-prod-XX.grafana.net/loki/api/v1/push
GCLOUD_HOSTED_LOGS_ID=123456
GCLOUD_ACCESS_POLICY_TOKEN=your_grafana_cloud_access_policy_token_here

# MySQL Passwords
DB_O11Y_PASSWORD=your_secure_password_here          # Password for 'db-o11y' MySQL monitoring user
GRAFANA_DB_PASSWORD=your_grafana_db_password_here    # Password for 'grafana' MySQL user (Grafana's backend database)
```

### 3. Start MySQL

Start MySQL first to configure the database observability db user and loadtest database:

```bash
docker compose up -d mysql
```

### 4. Configure MySQL Users

Connect to MySQL and create the database observability user. For detailed MySQL setup information, refer to the [official Grafana documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/database-observability/get-started/mysql/#set-up-the-mysql-database).

```bash
# Connect to MySQL
docker exec -it mysql-dbO11y mysql -u root -prootpass

# Create database observability user (required for both use cases)
# Use the same password as DB_O11Y_PASSWORD in .env
CREATE USER 'db-o11y'@'%' IDENTIFIED BY 'your_secure_password_here';

# Grant necessary permissions for database monitoring
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'db-o11y'@'%';
GRANT SELECT ON performance_schema.* TO 'db-o11y'@'%';
GRANT SELECT ON mysql.* TO 'db-o11y'@'%';
GRANT UPDATE ON performance_schema.setup_consumers TO 'db-o11y'@'%';

# Create test database and tables (only needed for load testing)
CREATE DATABASE IF NOT EXISTS loadtest;
GRANT INSERT, UPDATE, DELETE, CREATE ON loadtest.* TO 'db-o11y'@'%';

USE loadtest;
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

FLUSH PRIVILEGES;
EXIT;
```

### 5. Update Alloy Secret

Update the MySQL connection string for Alloy (replace `your_secure_password_here` with the same password used for `DB_O11Y_PASSWORD`):

```bash
printf "db-o11y:your_secure_password_here@tcp(mysql:3306)/" > alloy/secrets/mysql_secret_main
```

### 6. Start Alloy

Now start Alloy to begin database monitoring:

```bash
docker compose --profile monitoring up -d
```

**Optional**: Add Grafana as well (will monitor Grafana database, too)

```bash
# Start Grafana as well (optional)
docker compose --profile grafana up -d
```

### 7. Access Services

- **Grafana**: http://localhost:3000 (admin/admin) - if started
- **Alloy**: http://localhost:12345
- **MySQL**: localhost:3306


## Running Load Tests against loadtest database

Start various load testing scenarios:

```bash
# Start all stress testing services (this will also start MySQL)
docker compose --profile stress up -d

# Or combine with monitoring/grafana profiles as needed
docker compose --profile stress --profile monitoring up -d              # Stress + Alloy
docker compose --profile stress --profile grafana up -d                 # Stress + Grafana  
docker compose --profile stress --profile monitoring --profile grafana up -d  # Everything

# Or start individual tests (requires MySQL to be running first)
docker compose up -d mysql-loadtest        # Basic CRUD operations
docker compose up -d mysql-slow-queries    # Slow query generation
docker compose up -d mysql-cpu-bomb        # CPU intensive queries
docker compose up -d mysql-connection-bomb # Connection exhaustion
docker compose up -d mysql-error-generator # SQL errors
docker compose up -d mysql-memory-test     # Memory intensive queries
```

Stop load testing:

```bash
docker compose --profile stress down
```

---

# Use Case 2: Grafana Database Migration Testing

Use for visualizing testing MySQL metrics/logs in dbO11y app during Grafana upgrades.

## Migration Setup

### 1. Initial Setup with Older Grafana Version

Start with an older version of Grafana to simulate a migration scenario:

```yaml
# In docker-compose.yml, set Grafana to older version
grafana:
  image: grafana/grafana-enterprise:11.6.0  # Starting version
```

### 2. Configure MySQL Users (if not done already)

If you skipped Use Case 1, you need to create the observability user. For detailed MySQL setup information, refer to the [official Grafana documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/database-observability/get-started/mysql/#set-up-the-mysql-database).

```bash
# Start MySQL first
docker compose up -d mysql

# Connect to MySQL
docker exec -it mysql-dbO11y mysql -u root -prootpass

# Create observability user (use the same password as DB_O11Y_PASSWORD in .env)
CREATE USER 'db-o11y'@'%' IDENTIFIED BY 'your_secure_password_here';

# Grant necessary permissions for database monitoring
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'db-o11y'@'%';
GRANT SELECT ON performance_schema.* TO 'db-o11y'@'%';
GRANT SELECT ON mysql.* TO 'db-o11y'@'%';
GRANT UPDATE ON performance_schema.setup_consumers TO 'db-o11y'@'%';

FLUSH PRIVILEGES;
EXIT;
```

### 3. Update Alloy Secret (if not done already)

```bash
printf "db-o11y:your_secure_password_here@tcp(mysql:3306)/" > alloy/secrets/mysql_secret_main
# Replace 'your_secure_password_here' with the same password used for DB_O11Y_PASSWORD
```

### 4. Start Grafana and Alloy

```bash
# Start services for migration testing
docker compose --profile grafana --profile monitoring up -d
```

### 5. Access Grafana

```bash
# Access Grafana and configure dashboards, data sources, users, etc.
# URL: http://localhost:3000 (admin/admin)
```


### 6. Create Backup Before Migration

Always backup before upgrading:

```bash
# Create backup directory
mkdir -p backup

# Backup Grafana data volume
docker run --rm \
  -v dbo11y-dc_grafana_data:/source \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/grafana-$(date +%Y%m%d-%H%M%S)-backup.tar.gz -C /source .

# Optional: Backup MySQL data as well
docker run --rm \
  -v dbo11y-dc_mysql_data:/source \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/mysql-$(date +%Y%m%d-%H%M%S)-backup.tar.gz -C /source .
```

### 7. Perform Migration

Update Grafana version and restart:

```yaml
# In docker-compose.yml, update Grafana version
grafana:
  image: grafana/grafana-enterprise:12.3.0  # Target version
```

```bash
# Stop Grafana
docker compose --profile grafana down

# Start with new version
docker compose --profile grafana up -d

# Monitor logs for migration progress
docker logs grafana-dbO11y -f
```

### 8. Rollback if Needed

If migration fails, restore from backup:

```bash
# Stop Grafana
docker compose --profile grafana down
```

Revert Grafana image version in `docker-compose.yml` to previous version (example: change from `grafana/grafana-enterprise:12.3.0` back to `grafana/grafana-enterprise:11.6.0`).

```bash
# Remove current Grafana data
docker volume rm dbo11y-dc_grafana_data

# Restore from backup
docker run --rm \
  -v dbo11y-dc_grafana_data:/target \
  -v $(pwd)/backup:/backup \
  alpine tar xzf /backup/grafana-YYYYMMDD-HHMMSS-backup.tar.gz -C /target

# Start Grafana again
docker compose --profile grafana up -d
```

---


## Troubleshooting

## Useful Commands

```bash
# View all logs (core services)
docker compose logs -f mysql grafana alloy

# View all logs including load tests (if running)
docker compose --profile stress logs -f mysql grafana alloy

# View specific service logs
docker compose logs alloy -f
docker compose logs grafana -f
docker compose logs mysql -f

# Check MySQL Performance Schema
docker exec -it mysql-dbO11y mysql -u root -prootpass -e "
SELECT * FROM performance_schema.setup_consumers 
WHERE NAME LIKE '%events_statements%' OR NAME LIKE '%events_waits%';"

# Test MySQL connection as observability user
docker exec -it mysql-dbO11y mysql -h mysql -u db-o11y -p

# Check Alloy configuration
docker exec alloy-dbO11y cat /etc/alloy/config.alloy
```

## File Structure

```
db-observability-demo/
├── docker-compose.yml          # Main orchestration file
├── .env                        # Environment variables (not in git)
├── alloy/
│   ├── config.alloy           # Alloy configuration
│   └── secrets/
│       └── mysql_secret_main  # MySQL DSN for Alloy (created during setup)
├── backup/                    # Backup directory (created during usage, not in git)
└── README.md                  # This file
```
