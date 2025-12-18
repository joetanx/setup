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

## 3. n8n

Pull container image (optional) and start service:

```sh
podman pull docker.n8n.io/n8nio/n8n:latest
systemctl start n8n
```

## 4. Langflow

Pull container image (optional) and start service:

```sh
podman pull docker.io/langflowai/langflow:latest
systemctl start langflow
```

## 5. Check status

```sh
systemctl status n8n langflow
podman logs n8n
podman logs langflow
```
