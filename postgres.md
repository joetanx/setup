## 1. Install PostgreSQL packages

```sh
yum -y install postgresql-server postgresql-contrib
systemctl enable --now postgresql
firewall-cmd --add-service postgresql --permanent && firewall-cmd --reload
```

## 2. Initialize PostgreSQL

```console
[root@foxtrot ~]# postgresql-setup --initdb
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
```

## 3. Configure authentication

Configure `/var/lib/pgsql/data/pg_hba.conf` to use password authentication over `scram-sha-256` hashing

```
⋮
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             0.0.0.0/0               scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 ident
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
⋮
```

Configure `/var/lib/pgsql/data/postgresql.conf` to listen on all addresses and enable `scram-sha-256` hashing

```
⋮
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '0.0.0.0'            # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
#port = 5432                            # (change requires restart)
max_connections = 100                   # (change requires restart)
⋮
# - Authentication -
⋮
password_encryption = scram-sha-256
⋮
```

## 4. Configure PostgreSQL users

### 4.1. Example: Keycloak

|Parameter|Value|
|---|---|
|Database|keycloak|
|Username|keycloak|
|Password|Keycloak123|

```console
[root@foxtrot ~]# cd /var/lib/pgsql
[root@foxtrot pgsql]# sudo -u postgres createuser -s -w keycloak
[root@foxtrot pgsql]# sudo -u postgres psql -c "ALTER USER keycloak WITH PASSWORD 'Keycloak123';"
ALTER ROLE
[root@foxtrot pgsql]# sudo -u postgres psql -c "CREATE DATABASE keycloak;"
CREATE DATABASE
[root@foxtrot pgsql]# sudo -u postgres psql -c "GRANT keycloak TO keycloak;"
GRANT
```

### 4.2. Example: Hashicorp Vault database secrets engine

|Parameter|Value|
|---|---|
|Database|pg_monitor|
|Username|vault|
|Password|Vault123|

```console
[root@foxtrot ~]# cd /var/lib/pgsql
[root@foxtrot pgsql]# sudo -u postgres createuser -s -w vault
[root@foxtrot pgsql]# sudo -u postgres psql -c "ALTER USER vault WITH PASSWORD 'Vault123';"
ALTER ROLE
[root@foxtrot pgsql]# sudo -u postgres psql -c "GRANT pg_monitor TO vault;"
GRANT
```

### 4.3. Example: Hashicorp Boundary

|Parameter|Value|
|---|---|
|Database|boundary|
|Username|boundary|
|Password|Boundary123|

```console
[root@foxtrot ~]# cd /var/lib/pgsql
[root@foxtrot pgsql]# sudo -u postgres createuser -s -w boundary
[root@foxtrot pgsql]# sudo -u postgres psql -c "ALTER USER boundary WITH PASSWORD 'Boundary123';"
ALTER ROLE
[root@foxtrot pgsql]# sudo -u postgres psql -c "CREATE DATABASE boundary;"
CREATE DATABASE
[root@foxtrot pgsql]# sudo -u postgres psql -c "GRANT boundary TO boundary;"
GRANT
```
