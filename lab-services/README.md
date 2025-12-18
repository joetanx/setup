## 1. Components

### 1.1. Lab routing

```mermaid
flowchart TD
  A(User) --> B(Traefik)
  subgraph "Podman host"
    B -->|gitlab.vx| C1(gitlab)
    B -->|keycloak.vx| D(keycloak)
    B -->|mail.vx| E(mail)
    C1 --> F(postgres)
    D --> F
    C1 ~~~ C2(gitlab-runner)
    C2 --> C1
  end
```

### 1.2. Quadlets

```mermaid
flowchart TD
  subgraph "Network: lab"
    subgraph "Containers"
      A(gitlab)
      B(gitlab-runner)
      C(keycloak)
      D(mail)
    end
  end
  subgraph "Volumes"
    A1(gitlab-config)
    A2(gitlab-data)
    A3(gitlab-logs)
    B1(gitlab-runner-config)
    D1(mail-data)
    D2(mail-systemdata)
  end
  A --> A1
  A --> A2
  A --> A3
  B --> B1
  D --> D1
  D --> D2
```

## 2. Prepare PostgreSQL for gitlab and keycloak

Create the respective databases:

```sh
podman exec -i postgres psql "postgres://postgres:password@localhost:5432/postgres" << EOF
CREATE USER gitlab WITH PASSWORD 'password';
CREATE DATABASE gitlabhq_production;
ALTER DATABASE gitlabhq_production OWNER TO gitlab;
EOF
```

```sh
podman exec -i postgres psql "postgres://postgres:password@localhost:5432/postgres" << EOF
CREATE USER keycloak WITH PASSWORD 'password';
CREATE DATABASE keycloak;
ALTER DATABASE keycloak OWNER TO keycloak;
EOF
```

> [!Note]
>
> Since [PostgreSQL 15](https://www.postgresql.org/about/news/postgresql-15-released-2526/), **`CREATE` permission from all users are revoked** except a database owner from the `public` (or default) schema.
>
> `GRANT ALL PRIVILEGES ON DATABASE <database> TO <user>;` is not sufficient
>
> `ALTER DATABASE <database> OWNER TO <user>;` is required

## 3. Setup and run

Download quadlet files and pull container images:

```sh
for item in gitlab-config.volume gitlab-data.volume gitlab-logs.volume gitlab.container gitlab-runner.container gitlab-runner-config.volume keycloak.container mail-data.volume mail-systemdata.volume mail.container; do
  curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/setup/raw/refs/heads/main/lab-services/quadlets/$item
done
systemctl daemon-reload
podman pull docker.io/smartertools/smartermail:latest
podman pull docker.io/gitlab/gitlab-ce:latest
podman pull docker.io/gitlab/gitlab-runner:alpine
podman pull docker.io/keycloak/keycloak:latest

```

Start services:

```sh
systemctl start mail gitlab gitlab-runner keycloak
```

Check statuses:

```sh
systemctl status mail gitlab gitlab-runner keycloak
```

Check container logs:

```sh
podman logs mail
podman logs gitlab
podman logs gitlab-runner
podman logs keycloak
```
