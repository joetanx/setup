## 1. Components

### 1.1. Lab routing

```mermaid
flowchart TD
  A(User) --> B(Traefik)
  subgraph "Podman host"
    B -->|opencti.vx| C(OpenCTI)
    C --> E1(Redis)
    C --> E2(Elasticsearch)
    C --> E3(MinIO)
    C --> E4(RabbitMQ)
    C ~~~ E5(Worker)
    E5 --> C
  end
```

### 1.2. Quadlets

```mermaid
flowchart TD
  subgraph "Network: lab"
    subgraph "Containers"
      A(redis)
      B(elasticsearch)
      C(minio)
      D(rabbitmq)
      E(opencti)
      F(worker)
    end
  end
  subgraph "Volumes"
    A1(redis)
    B1(elasticsearch)
    C1(minio)
    D1(rabbitmq)
  end
  E ~~~ A
  F ~~~ B
  A --> A1
  B --> B1
  C --> C1
  D --> D1
```

## 2. Setup OpenCTI Platform

Download the quadlet unit files and reload systemd:

```sh
curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/setup/raw/refs/heads/main/elastic/quadlets/elasticsearch.container
curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/setup/raw/refs/heads/main/elastic/quadlets/elasticsearch.volume
for item in minio.container minio.volume opencti.container rabbitmq.container rabbitmq.volume redis.container redis.volume worker.container; do
  curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/setup/raw/refs/heads/main/opencti/quadlets/$item
done
systemctl daemon-reload
```

Pull container image (optional) and start service:

```sh
podman pull docker.io/library/redis:latest
podman pull docker.io/elastic/elasticsearch:9.2.3
podman pull docker.io/minio/minio:latest
podman pull docker.io/library/rabbitmq:management
podman pull docker.io/opencti/platform:latest
podman pull docker.io/opencti/worker:latest
systemctl start worker
```

> [!Tip]
>
> Notice that only worker needs to be started
> 
> In the systemd uni file, worker `requires` opencti, which `requires` redis, elasticsearch, minio and rabbitmq
>
> This chain of `requires` means starting worker will automatically start all the `requires`
>
> ```mermaid
> flowchart LR
>   A(worker) --> B(opencti)
>   B --> C(redis)
>   B --> D(elasticsearch)
>   B --> E(minio)
>   B --> F(rabbitmq)
> ```

Check status:

```sh
systemctl status redis elasticsearch minio rabbitmq opencti worker
podman logs redis
podman logs elasticsearch
podman logs minio
podman logs rabbitmq
podman logs opencti
podman logs worker
```

## 3. Connectors

Download the quadlet unit files:

```sh
for item in connector-abuseipdb.container connector-alienvault.container connector-malwarebazaar.container connector-threatfox.container connector-urlhaus.container connector-virustotal.container; do
  curl -sL --output-dir /etc/containers/systemd/ -O https://github.com/joetanx/setup/raw/refs/heads/main/opencti/quadlets/$item
done
```

Replace placeholders for AbuseIPDB, AlienVault, VirusTotal and MalwareBazaar:

```sh
sed -i 's/YOUR_API_KEY_HERE/<abuseipdb-api-key>/' /etc/containers/systemd/connector-abuseipdb.container
sed -i 's/YOUR_API_KEY_HERE/<alienvault-api-key>/' /etc/containers/systemd/connector-alienvault.container
sed -i "s/CONNECTOR_TIMESTAMP_HERE/$(date -d '1 day ago' '+%Y-%m-%dT%H:%M:%S')/" /etc/containers/systemd/connector-alienvault.container
sed -i 's/YOUR_API_KEY_HERE/<virustotal-api-key>/' /etc/containers/systemd/connector-virustotal.container
sed -i 's/YOUR_API_KEY_HERE/<malwarebazaar-api-key>/' /etc/containers/systemd/connector-malwarebazaar.container
```

Reload systemd:

```sh
systemctl daemon-reload
```

Pull container image (optional) and start services:

```sh
podman pull docker.io/opencti/connector-abuseipdb-ipblacklist:latest
podman pull docker.io/opencti/connector-alienvault:latest
podman pull docker.io/opencti/connector-malwarebazaar:latest
podman pull docker.io/opencti/connector-threatfox:latest
podman pull docker.io/opencti/connector-urlhaus:latest
podman pull docker.io/opencti/connector-virustotal:latest
systemctl start connector-abuseipdb connector-alienvault connector-malwarebazaar connector-threatfox connector-urlhaus connector-virustotal
```

The connector containers has no logs when it's working fine (no news mean good news?), `podman logs` can be used with `--color`, `--names` and `--timestamps` options to quickly see all the logs:

```sh
podman logs --color -nt connector-abuseipdb connector-alienvault connector-malwarebazaar connector-threatfox connector-urlhaus connector-virustotal
```

## 4. Exploring OpenCTI console

#### Ingestion status

Data → Ingestion

![](https://github.com/user-attachments/assets/5b2628d0-69d3-4714-bda1-291f0ea397e3)

#### Home dashboard

![](https://github.com/user-attachments/assets/9b81de96-b681-4c7e-8fab-ab821078ec01)

#### Reports

Analyses → Reports

![](https://github.com/user-attachments/assets/747e1b5b-09df-41ce-bf31-2a4d6e38a80b)

#### External references

Analyses → External references

![](https://github.com/user-attachments/assets/56b27c02-b308-44fc-9183-91a907929644)

#### Oberservables

Oberservations → Oberservables

![](https://github.com/user-attachments/assets/655959d1-30eb-416d-9d2a-d07f7b8ce73e)

#### Indicators

Oberservations → Indicators

![](https://github.com/user-attachments/assets/678fd768-1a5c-438b-811f-c3c146ac8c40)

#### Intrusion sets

Threats → Intrusion sets

![](https://github.com/user-attachments/assets/e7e2d8cf-26e9-430f-9304-7349c7e46115)

#### Malware

Arsenal → Malware

![](https://github.com/user-attachments/assets/de50f6bd-ee89-48ec-9478-52b0df773007)

#### Vulnerabilities

Arsenal → Vulnerabilities

![](https://github.com/user-attachments/assets/2979a108-2a29-4a21-b993-544ce0082b77)

#### Attack patterns

Techniques → Attack patterns

![](https://github.com/user-attachments/assets/1a35efcd-b412-4454-8e6f-4fd8596aeeb2)
