## 1. Install MySQL server packages

- Install MySQL server
- Enable MySQL server to start with system
- Allow MySQL communiction on firewalld

```sh
yum -y install mysql-server
systemctl enable --now mysqld
firewall-cmd --add-service mysql --permanent && firewall-cmd --reload
```

## 2. Setup sample world database

- Download and extract sample world database
- Import sample world database with `SOURCE` command

```sh
curl -sLO https://downloads.mysql.com/docs/world-db.tar.gz
tar xvf world-db.tar.gz
mysql -u root -e "SOURCE /root/world-db/world.sql"
```

Verify the database loaded:

```sh
mysql -u root -e "SHOW DATABASES;"
mysql -u root -e "SHOW TABLES FROM world;"
mysql -u root -e "DESCRIBE world.city;"
mysql -u root -e "DESCRIBE world.country;"
mysql -u root -e "DESCRIBE world.countrylanguage;"
```

Verify the database loaded from filesystem:

```sh
ls -l /var/lib/mysql/world
```

## 3. Configure MySQL users

|Username|Password|
|---|---|
|cityapp|Cyberark1|
|cicd|Cyberark1|

```sh
mysql -u root -e "CREATE USER 'cityapp'@'%' IDENTIFIED BY 'Cyberark1';"
mysql -u root -e "GRANT ALL PRIVILEGES ON *.* TO 'cityapp'@'%';"
mysql -u root -e "CREATE USER 'cicd'@'%' IDENTIFIED BY 'Cyberark1';"
mysql -u root -e "GRANT ALL PRIVILEGES ON *.* TO 'cicd'@'%';"
```

Optional:
- Allow root login from remote clients
- Set password for root user

```sh
mysql -u root -e "RENAME USER 'root'@'localhost' TO 'root'@'%';"
mysql -u root -e "ALTER USER 'root' IDENTIFIED BY 'Cyberark1';"
```

Verify users:

```sh
mysql -u root -e "SELECT user,host,plugin FROM mysql.user;"
```

## 4. Clean-up

```console
rm -rf world-db.tar.gz world-db .mysql_history
```
