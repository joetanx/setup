## 1. Components

### 1.1. Lab routing

```mermaid
flowchart TD
  A(User) --> B(Traefik)
  subgraph "Podman host"
    B -->|langflow.vx| C1(Langflow)
    B -->|n8n.vx| C2(n8n)
    C1 --> D(PostgresSQL)
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
podman exec postgres psql "postgres://postgres:password@localhost:5432/postgres" -c "CREATE DATABASE langflow;"
podman exec postgres psql "postgres://postgres:password@localhost:5432/postgres" -c "CREATE DATABASE n8n;"
```

> [!Tip]
>
> Troubleshooting commands:
>
> ```sh
> user=postgres
> password=password
> podman exec postgres psql "postgres://$user:$password@localhost:5432/postgres" -c "SELECT now();"
> podman exec postgres psql "postgres://$user:$password@localhost:5432/postgres" -c "\l+"
> podman exec postgres psql "postgres://$user:$password@localhost:5432/postgres" -c "\du+"
> podman exec postgres psql "postgres://$user:$password@localhost:5432/langflow" -c "\dt+"
> podman exec postgres psql "postgres://$user:$password@localhost:5432/n8n" -c "\dt+"
> ```

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
