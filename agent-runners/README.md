## 1. Components

### 1.1. Lab routing

```mermaid
flowchart TD
  A(User) --> B(Traefik)
  subgraph "Podman host"
    B -->|langflow.vx| C1(Langflow)
    B -->|n8n.vx| C2(n8n)
    C1 --> D(postgres)
    C2 --> D
  end
```

### 1.2. Quadlets

```mermaid
flowchart TD
  subgraph "Network: lab"
    subgraph "Containers"
      A(n8n)
      B(langflow)
    end
  end
  subgraph "Volumes"
    A1(n8n)
    B1(langflow)
  end
  A --> A1
  B --> B1
```

## 2. Prepare PostgreSQL

Create the respective databases for n8n and Langflow:

```sh
podman exec -i postgres psql "postgres://postgres:password@localhost:5432/postgres" << EOF
CREATE USER n8n WITH PASSWORD 'password';
CREATE DATABASE n8n;
ALTER DATABASE n8n OWNER TO n8n;
EOF
```

```sh
podman exec -i postgres psql "postgres://postgres:password@localhost:5432/postgres" << EOF
CREATE USER langflow WITH PASSWORD 'password';
CREATE DATABASE langflow;
ALTER DATABASE langflow OWNER TO langflow;
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
for item in langflow.volume langflow.container n8n.volume n8n.container; do
  curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/setup/raw/refs/heads/main/agent-runners/quadlets/$item
done
systemctl daemon-reload
podman pull docker.io/langflowai/langflow:latest
podman pull docker.n8n.io/n8nio/n8n:latest
```

Start services:

```sh
systemctl start n8n langflow
```

Check statuses:

```sh
systemctl status n8n langflow
```

Check container logs:

```sh
podman logs n8n
podman logs langflow
```
