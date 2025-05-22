## 1. Install Cribl

### 1.1. Download and extract Cribl

> [!Note]
>
> The `C` switch specifies the directory to extract to

```sh
yum -y install tar
curl -sLo - $(curl https://cdn.cribl.io/dl/latest-x64) | tar zxvC /opt
```

### 1.2. Add firewall rules

|Port|Protocol|Purpose|
|---|---|---|
|9000|TCP|Cribl web UI|
|9514|TCP, UDP|Cribl default syslog input, change as necessary|
|5986|TCP|WinRM for Windows Event Collector input|

```sh
firewall-cmd --permanent --add-port 9000/tcp
firewall-cmd --permanent --add-port 9514/udp
firewall-cmd --permanent --add-port 9514/tcp
firewall-cmd --permanent --add-port 5986/tcp
firewall-cmd --reload
```

### 1.3. Setup `cribl` user

```sh
useradd cribl
chown -R cribl:cribl /opt/cribl
```

### 1.4. Configure Cribl to run on boot with systemd

```sh
/opt/cribl/bin/cribl boot-start enable -m systemd -u cribl
systemctl start cribl
```

### 1.5. Initial login

> [!Note]
>
> Default login credentials: `admin`: `admin`

![image](https://github.com/user-attachments/assets/5c7c816f-096a-47f5-84fe-aa3abf6f28ba)

![image](https://github.com/user-attachments/assets/17cb1729-d63c-4316-8eaa-203e316a4e35)

![image](https://github.com/user-attachments/assets/41a1ffc9-d286-4d7d-bfd9-b7bbd7c9b9ab)

### 1.6. Configure TLS

> [!Note]
>
> This guide installs Cribl on a host `delta.vx` and uses test certificates from [lab-certs](https://github.com/joetanx/lab-certs)

#### 1.6.1. Add certificate

![image](https://github.com/user-attachments/assets/b43306c4-524c-4acf-a55d-57621d2c58b0)

![image](https://github.com/user-attachments/assets/93710ca0-4bd0-4f21-a108-9c6512ce75b3)

#### 1.6.2. Configure TLS with the added certificate

![image](https://github.com/user-attachments/assets/f71abdc9-94ba-4bb9-95a9-5b5a77f50d46)

![image](https://github.com/user-attachments/assets/bdc37100-2d92-4909-bdf5-c31b8559dfba)

![image](https://github.com/user-attachments/assets/bf9915fe-9f62-4366-9ea6-4a0d04d077e3)
