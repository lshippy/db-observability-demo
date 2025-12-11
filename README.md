# MySQL Database Observability with Grafana and Alloy

This repository provides two main use cases for MySQL database observability:

1. **[Database Load Testing & Performance Monitoring](#use-case-1-database-load-testing--performance-monitoring)** - Generate synthetic database load and monitor performance
2. **[Grafana Database Migration Testing](#use-case-2-grafana-database-migration-testing)** - Test Grafana upgrades using MySQL as the backend database

Both use cases leverage Grafana Alloy for comprehensive database observability with Grafana Cloud.

## Architecture Components

- **MySQL 8.4 LTS**: Database with Performance Schema enabled for detailed query monitoring
- **Grafana Enterprise**: Visualization and alerting platform (configurable version)
- **Grafana Alloy**: Collects MySQL metrics and logs, forwards to Grafana Cloud
- **Load Testing Tools**: Multiple containers to generate database activity and errors

## Prerequisites

- Docker and Docker Compose
- Grafana Cloud account with:
  - Prometheus endpoint for metrics
  - Loki endpoint for logs
  - Cloud Access Policy token with metrics:write and logs:write permissions

---

# Use Case 1: Database Load Testing & Performance Monitoring

Use to visualize testing database performance, monitoring query patterns, and validating observability setups in dbO11y

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
DB_O11Y_PASSWORD=your_secure_password_here
GRAFANA_DB_PASSWORD=your_grafana_db_password_here
```

### 3. Start Core Services

For load testing, you can optionally skip Grafana and just run MySQL + Alloy:

```bash
# Start MySQL and Alloy only (for pure database monitoring)
docker compose up -d mysql alloy

# OR start everything including Grafana
docker compose up -d mysql grafana alloy
```

### 4. Configure MySQL Users

Connect to MySQL and create the observability user:

```bash
# Connect to MySQL
docker exec -it mysql-dbO11y mysql -u root -prootpass

# Create database and user for observability
CREATE DATABASE IF NOT EXISTS loadtest;
CREATE USER 'db-o11y'@'%' IDENTIFIED BY 'your_secure_password_here';

# Grant necessary permissions
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'db-o11y'@'%';
GRANT SELECT ON performance_schema.* TO 'db-o11y'@'%';
GRANT SELECT ON mysql.* TO 'db-o11y'@'%';
GRANT UPDATE ON performance_schema.setup_consumers TO 'db-o11y'@'%';

# Grant permissions for load testing
GRANT INSERT, UPDATE, DELETE, CREATE ON loadtest.* TO 'db-o11y'@'%';

# Create test tables
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
```

### 5. Update Alloy Secret

Update the MySQL connection string for Alloy:

```bash
echo "db-o11y:your_secure_password_here@tcp(mysql:3306)/" > alloy/secrets/mysql_secret_main
```

### 6. Access Services

- **Grafana**: http://localhost:3000 (admin/admin) - if started
- **Alloy**: http://localhost:12345
- **MySQL**: localhost:3306

## Running Load Tests

Start various load testing scenarios:

```bash
# Start all stress testing services
docker compose --profile stress up -d

# Or start individual tests
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

Use for visualizing testing MySQL metrics/logs during Grafana upgrades.

## Migration Setup

### 1. Initial Setup with Older Grafana Version

Start with an older version of Grafana to simulate a migration scenario:

```yaml
# In docker-compose.yml, set Grafana to older version
grafana:
  image: grafana/grafana-enterprise:11.6.0  # Starting version
```

### 2. Configure Grafana with MySQL Backend

```bash
# Start services
docker compose up -d mysql grafana alloy

# Access Grafana and configure dashboards, data sources, users, etc.
# URL: http://localhost:3000 (admin/admin)
```

### 3. Create Backup Before Migration

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

### 4. Perform Migration

Update Grafana version and restart:

```yaml
# In docker-compose.yml, update Grafana version
grafana:
  image: grafana/grafana-enterprise:12.3.0  # Target version
```

```bash
# Stop Grafana
docker compose stop grafana

# Start with new version
docker compose up -d grafana

# Monitor logs for migration progress
docker logs grafana-dbO11y -f
```

### 5. Verify Migration

- Check that all dashboards are preserved
- Verify data sources still work
- Test user accounts and permissions
- Confirm settings are maintained

### 6. Rollback if Needed

If migration fails, restore from backup:

```bash
# Stop services
docker compose down

# Remove current Grafana data
docker volume rm dbo11y-dc_grafana_data

# Restore from backup
docker run --rm \
  -v dbo11y-dc_grafana_data:/target \
  -v $(pwd)/backup:/backup \
  alpine tar xzf /backup/grafana-YYYYMMDD-HHMMSS-backup.tar.gz -C /target

# Revert docker-compose.yml to previous version
# Start services
docker compose up -d
```

---

## Troubleshooting

## Useful Commands

```bash
# View all logs
docker compose logs -f

# View specific service logs
docker logs alloy-dbO11y -f
docker logs grafana-dbO11y -f
docker logs mysql-dbO11y -f

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
dbo11y-dc/
├── docker-compose.yml          # Main orchestration file
├── .env                        # Environment variables (not in git)
├── alloy/
│   ├── config.alloy           # Alloy configuration
│   └── secrets/
│       └── mysql_secret_main  # MySQL DSN for Alloy
├── backup/                    # Backup directory
│   └── grafana-11.6-backup.tar.gz
└── README.md                  # This file
```
