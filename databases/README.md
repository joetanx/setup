## 1. Components

### 1.1. Lab routing

```mermaid
flowchart TD
  subgraph "Podman host"
    A(n8n) --> E(postgres)
    B(langflow) --> E
    C(gitlab) --> E
    D(keycloak) --> E
  end
```

### 1.2. Quadlets

```mermaid
flowchart TD
  subgraph "Network: lab"
    subgraph "Containers"
      A(postgres)
    end
  end
  subgraph "Volumes"
    A1(postgres)
  end
  A --> A1
```

## 2. Setup

> [!Note]
>
> The lab runs a single PostgreSQL container that is shared by n8n, Langflow, gitlab, keycloak

Download the quadlet unit files and reload systemd:

```sh
curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/soc-lab/raw/refs/heads/main/quadlets/postgres.volume
curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/soc-lab/raw/refs/heads/main/quadlets/traefik.container
systemctl daemon-reload
```

Pull container image (optional) and start service:

```sh
podman pull docker.io/library/postgres:latest
systemctl start postgres
```

Running single-line commands:

```sh
podman exec postgres psql "postgres://postgres:password@localhost:5432/postgres" -c "CREATE DATABASE langflow;"
```

```sh
podman exec postgres psql "postgres://postgres:password@localhost:5432/postgres" -c "CREATE DATABASE n8n;"
```

Running multi-line commands:

```sh
podman exec -i postgres psql "postgres://postgres:password@localhost:5432/postgres" <<EOF
CREATE USER gitlab WITH PASSWORD 'password';
CREATE DATABASE gitlabhq_production;
GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production TO gitlab;
ALTER USER gitlab WITH SUPERUSER;
EOF
```

```sh
podman exec -i postgres psql "postgres://postgres:password@localhost:5432/postgres" <<EOF
CREATE USER keycloak WITH PASSWORD 'password';
CREATE DATABASE keycloak;
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
ALTER USER keycloak WITH SUPERUSER;
EOF
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
