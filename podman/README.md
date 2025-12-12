## 1. Setup podman

```sh
yum -y install podman
systemctl enable --now podman
```

## 2. Podman quadlets

```mermaid
flowchart TD
  subgraph "Network: lab"
    subgraph "Containers"
      A(traefik)
      B(postgres)
      C(n8n)
      D(langflow)
      E(redis)
      F(elasticsearch)
      G(minio)
      H(rabbitmq)
      I(opencti)
      J(worker)
    end
  end
  A ~~~ C
  C ~~~ B
  A ~~~ D
  D ~~~ B
  A ~~~ I
  subgraph "Volumes"
    A1(traefik-tls)
    A2(traefik-dynamic)
    B1(postgres)
    C1(n8n)
    D1(langflow)
    E1(redis)
    F1(elasticsearch)
    G1(minio)
    H1(rabbitmq)
  end
  A --> A1
  A --> A2
  B --> B1
  C --> C1
  D --> D1
  I ~~~ E
  J ~~~ F
  E --> E1
  F --> F1
  G --> G1
  H --> H1
```

Docker Compose is a popular supported deployment method for various services such as [n8n](https://github.com/n8n-io/n8n-hosting/tree/main/docker-compose/withPostgres), [Langflow](https://github.com/langflow-ai/langflow/docker_example) and [OpenCTI](https://github.com/OpenCTI-Platform/docker/)

Podman [quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) is useful because components are managed independently under its respective systemd unit file

Differences on managing changes:
- Docker compose: edit the `docker-compose.yaml`, run `docker compose up -d`
- Podman quadlet: download the affected systemd unit file, `systemctl daemon-reload`, `systemctl restart <service>`

> [!Tip]
>
> Unlike normal systemd services, there is no need to `systemctl enable` any of the podman quadlets; they always start on boot
