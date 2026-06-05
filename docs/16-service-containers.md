# Chapter 16: Service Containers

## What Are Service Containers?

Service containers are Docker containers that run alongside a job for its entire duration. They are used to provide supporting infrastructure - such as a database, a cache, or a message broker - that the job's steps connect to during testing.

The primary use case is integration testing: rather than mocking a database, the job starts a real PostgreSQL or MySQL instance as a service container, runs the tests against it, and the container is automatically destroyed when the job ends.

Service containers are defined under the `services` key within a job.

## Defining a Service Container

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Run Integration Tests
        env:
          DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
        run: npm run test:integration
```

### Key Fields

**`image`**: The Docker image to use. Standard Docker Hub images and any registry accessible from the runner can be used.

**`env`**: Environment variables passed to the container on startup. Databases typically require at least a user, password, and database name to initialise correctly.

**`ports`**: Maps a port from the container to the runner's localhost. The format is `<host-port>:<container-port>`. After mapping, the job's steps can connect to the service on `localhost:<host-port>`.

**`options`**: Raw Docker `--option` flags passed to `docker create`. This is where health check parameters are defined.

## Health Checks

Service containers start asynchronously. If a step begins before the database has finished initialising, connection attempts will fail. Health checks instruct GitHub to wait until the container is ready before advancing to the steps.

```yaml
options: >-
  --health-cmd pg_isready
  --health-interval 10s
  --health-timeout 5s
  --health-retries 5
```

| Option | Meaning |
|---|---|
| `--health-cmd` | The command to run to check if the service is ready. For PostgreSQL, `pg_isready` is the standard check. For Redis, `redis-cli ping`. |
| `--health-interval` | How often to run the health check command. |
| `--health-timeout` | Maximum time to wait for the health check command to complete. |
| `--health-retries` | Number of consecutive failures before the container is considered unhealthy. |

GitHub waits for the container to report a healthy status before executing the job's steps.

## Connecting to the Service

### On GitHub-Hosted Runners
When the job runs on a GitHub-hosted runner, service containers are accessible at `localhost` using the mapped host port.

```yaml
env:
  DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
  REDIS_URL: redis://localhost:6379
```

### On Container Jobs (Docker in Docker)
When the job itself runs inside a container (using `container:` at the job level), the service is accessible via the service container's label name (not `localhost`).

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:18
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
    steps:
      - run: npm test
        env:
          # Use the service label 'postgres' as the hostname, not 'localhost'
          DATABASE_URL: postgresql://testuser:testpassword@postgres:5432/testdb
```

## Multiple Service Containers

Multiple services can be defined in the same job. Each service runs as a separate container and is accessible on its mapped port.

```yaml
services:
  postgres:
    image: postgres:15
    env:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpassword
      POSTGRES_DB: testdb
    ports:
      - 5432:5432
    options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5

  redis:
    image: redis:7
    ports:
      - 6379:6379
    options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - 5672:5672
    options: --health-cmd "rabbitmq-diagnostics ping" --health-interval 10s --health-timeout 5s --health-retries 5
```

## Common Database Images and Health Commands

| Database | Image | Health Command |
|---|---|---|
| PostgreSQL | `postgres:15` | `pg_isready` |
| MySQL | `mysql:8` | `mysqladmin ping -h localhost` |
| Redis | `redis:7` | `redis-cli ping` |
| MongoDB | `mongo:6` | `mongosh --eval "db.runCommand({ ping: 1 })"` |
| RabbitMQ | `rabbitmq:3-management` | `rabbitmq-diagnostics ping` |

## Credential Handling for Private Images

If the service container image is hosted in a private registry, credentials must be provided using the `credentials` key.

```yaml
services:
  internal-db:
    image: registry.internal.company.com/custom-postgres:15
    credentials:
      username: ${{ secrets.REGISTRY_USERNAME }}
      password: ${{ secrets.REGISTRY_PASSWORD }}
    ports:
      - 5432:5432
```
