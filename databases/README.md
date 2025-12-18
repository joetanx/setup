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

### 2.1. Example commands

Running single-line commands:

```sh
podman exec postgres psql "postgres://postgres:password@localhost:5432/postgres" -c "CREATE DATABASE langflow;"
```

```sh
podman exec postgres psql "postgres://postgres:password@localhost:5432/postgres" -c "CREATE DATABASE n8n;"
```

Running multi-line commands:

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

### 2.2. Troubleshooting commands

|Command|Description|
|---|---|
|`SELECT now()`|• Returns **current timestamp** on the database server (e.g. `2025-12-18 07:58:23.123456+08`)|
|`\l+`|• Lists all **databases**<br>• `+` shows `Size`, `Tablespace`, and `Description`|
|`\du+`|• Lists all **roles/users**<br>• `+` shows `Description`|
|`\dt+`|• Lists all **tables** in current schema<br>• `+` shows `Persistence`, `Access method`, `Size` and `Description`|
