# docker-backup-postgres

Docker image of PostgreSQL, cron and tini for scheduling database dumps.

## Usage

This repository is used to create a Docker image that can be used to backup a PostgreSQL database. The image is based on the official PostgreSQL image and adds cron and tini to the image. The image is used to create a container that will run a cron job to backup a PostgreSQL database.

The image is available on Docker Hub at [dimitriosdockerhub/backup-postgres](https://hub.docker.com/r/dimitriosdockerhub/backup-postgres).

## Manual Build

To build the image manually, clone the repository and run the following command:

```sh
docker build -t backup-postgres .
```

## Docker Compose

The following is an example of a `docker-compose.yml` file that can be used to create a PostgreSQL container and a backup container. The backup container will run a cron job to backup the PostgreSQL database.

If the dump fails, the error file will contain the error message. Otherwise, it will be empty. You may want to check the error file to see if the dump was successful using monitoring tools like Zabbix or Prometheus.

```yml
version: "3.9"

services:
  postgres:
    container_name: ${POSTGRES_CONTAINER_NAME:-postgres}
    image: postgres:${POSTGRES_IMAGE_VERSION:-latest}
    hostname: ${POSTGRES_CONTAINER_NAME:-postgres}
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-default_database}
      POSTGRES_USER: ${POSTGRES_USER:-user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "${POSTGRES_EXT_PORT:-10000}:5432" # for development, close in production
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - postgres_network

  backup_postgres:
    container_name: backup_${POSTGRES_CONTAINER_NAME:-postgres}
    image: dimitriosdockerhub/backup-postgres:${POSTGRES_IMAGE_VERSION:-latest}
    hostname: backup_${POSTGRES_CONTAINER_NAME:-postgres}
    restart: unless-stopped
    depends_on:
      - postgres
    volumes:
      - ${POSTGRES_PATH_TO_DUMP:-/backups}:/backups:rw
    environment:
      CRONTAB: |
        ${POSTGRES_CRONTAB:-50 5 * * *} bash -c "PGPASSWORD=${POSTGRES_PASSWORD} pg_dumpall --host=${POSTGRES_CONTAINER_NAME:-postgres} --user=${POSTGRES_USER:-user} > ${POSTGRES_PATH_TO_DUMP:-/backups}/${POSTGRES_BACKUP_NAME:-all-databases-backup}.sql 2>${POSTGRES_PATH_TO_DUMP:-/backups}/${POSTGRES_BACKUP_NAME:-backup-name}.err && gzip -9 -f ${POSTGRES_PATH_TO_DUMP:-/backups}/${POSTGRES_BACKUP_NAME:-all-databases-backup}.sql"
    healthcheck:
      test: ["CMD-SHELL", "pidof cron && command -v pg_dumpall"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - postgres_network

volumes:
  postgres_data:

networks:
  postgres_network:
    driver: bridge
```

### Environment Variables

The following environment variables can be used to configure the PostgreSQL container and the backup container:

```txt
POSTGRES_CONTAINER_NAME=postgres-container
POSTGRES_IMAGE_VERSION=latest
POSTGRES_DATABASE=default_database
POSTGRES_USER=user
POSTGRES_PASSWORD=password
POSTGRES_EXT_PORT=10002
POSTGRES_PATH_TO_DUMP=/backups
POSTGRES_BACKUP_NAME=all-databases-backup
POSTGRES_CRONTAB=*/2 * * * *
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
