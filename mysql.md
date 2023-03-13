# Setup MySQL database

- Install MySQL server
- Enable MySQL server to start with system
- Allow MySQL communiction on firewalld

```console
yum -y install mysql-server
systemctl enable --now mysqld
firewall-cmd --add-service mysql --permanent && firewall-cmd --reload
```

- Download sample world database

```console
curl -L -O https://downloads.mysql.com/docs/world-db.tar.gz
tar xvf world-db.tar.gz
```

- Login to the MySQL database

```console
mysql -u root
```

- Create MySQL accounts to be used for the sample application

```console
CREATE USER 'cityapp'@'%' IDENTIFIED BY 'Cyberark1';
GRANT ALL PRIVILEGES ON *.* TO 'cityapp'@'%';
CREATE USER 'cicd'@'%' IDENTIFIED BY 'Cyberark1';
GRANT ALL PRIVILEGES ON *.* TO 'cicd'@'%';
```

- Optional:
  - Allow root login from remote clients
  - Set password for root user

```console
RENAME USER 'root'@'localhost' TO 'root'@'%';
ALTER USER 'root' IDENTIFIED BY 'Cyberark1';
```

- Verify users

```console
SELECT user,host FROM mysql.user;
```

- Load sample world database

```console
SOURCE /root/world-db/world.sql
```

- Verify the database loaded

```console
SHOW DATABASES;
SHOW TABLES FROM world;
DESCRIBE world.city;
DESCRIBE world.country;
DESCRIBE world.countrylanguage;
```

- Exit MySQL command

```console
QUIT;
```

- Verify the database loaded from filesystem

```console
ls -l /var/lib/mysql/world
```

- Clean-up

```console
rm -rf world-db.tar.gz world-db .mysql_history
```
