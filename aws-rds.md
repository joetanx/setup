# 1. Setup

<details><summary><h2>1.1. Create AWS RDS (MySQL)</h2></summary>

![image](https://user-images.githubusercontent.com/90442032/226158645-6ade85e1-e898-4b1d-a99b-7ddb7f4ba3c2.png)

![image](https://user-images.githubusercontent.com/90442032/226158646-aa286119-6cff-46a3-9ef4-c951e3d6f3db.png)

![image](https://user-images.githubusercontent.com/90442032/226158649-18338594-f46b-411e-85f3-55a90db71820.png)

![image](https://user-images.githubusercontent.com/90442032/226158654-d1ee9959-393a-4687-ae93-89f0c90b2ed5.png)

![image](https://user-images.githubusercontent.com/90442032/226158658-ae069194-9f4f-4baa-8c83-ab0fe00d7397.png)

![image](https://user-images.githubusercontent.com/90442032/226158662-39ee9e59-e98c-44c7-abd8-c49354d93a11.png)

![image](https://user-images.githubusercontent.com/90442032/226158665-3008abb3-e692-4034-9816-75029220d8b9.png)

![image](https://user-images.githubusercontent.com/90442032/226158669-1dbfd6d2-6e01-4bcb-b20b-27d1dd2c9027.png)

![image](https://user-images.githubusercontent.com/90442032/226158670-b1964791-ec5d-47ac-97f1-f1f1d5bd2e85.png)

![image](https://user-images.githubusercontent.com/90442032/226158673-eee75474-0bf1-4ac9-be00-996aace439a0.png)

![image](https://user-images.githubusercontent.com/90442032/226158675-8221bf54-b6f1-4e8c-aa5d-75544380c1d9.png)

![image](https://user-images.githubusercontent.com/90442032/226158715-b2a4ac95-da92-48f0-b4b4-9760a27e2196.png)

</details>

<details><summary><h2>1.2. Create EC2 instance</h2></summary>

![image](https://user-images.githubusercontent.com/90442032/226159252-c55c852d-4623-4fe9-9ffd-cd255872e6ea.png)

![image](https://user-images.githubusercontent.com/90442032/226159265-c1c184e8-4483-4824-ad9b-19ac13206c22.png)

![image](https://user-images.githubusercontent.com/90442032/226159279-556629f1-935e-414c-8ff6-a47515eb2477.png)

![image](https://user-images.githubusercontent.com/90442032/226159289-243f29ae-3382-4fc0-a636-225ef32d2b99.png)

![image](https://user-images.githubusercontent.com/90442032/226159397-e57e6c67-83a0-441d-97b6-70cabcdec363.png)

![image](https://user-images.githubusercontent.com/90442032/226159412-36f5de4e-c20b-4fee-8b59-f290ec36a130.png)

</details>

## 1.3. Allow communication from EC2 instance to RDS (security groups)

## 1.4. Configure RDS for IAM authentication

## 1.5. Create dbuser and setup world database

Login to RDS with master password

```console
mysql -h jtan-rds.a1b2c3d4e5f6.ap-southeast-1.rds.amazonaws.com -u admin -pCyberark1
CREATE USER cityapp IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS'; 
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER ON *.* TO 'cityapp'@'%';
```

## 1.6. Setup EC2 instance profile

### 1.6.1. Create IAM policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds-db:connect"
            ],
            "Resource": [
                "arn:aws:rds-db:ap-southeast-1:123412341234:dbuser:*/*"
            ]
        }
    ]
}
```

### 1.6.2. Create IAM role

### 1.6.3. Attach IAM role to EC2 instance profile

# 2. EC2 instance connection to RDS using IAM authentication

## 2.1. Test connection using AWS CLI + MySQL client

```console
RDSHOST="jtan-rds.a1b2c3d4e5f6.ap-southeast-1.rds.amazonaws.com"
TOKEN="$(aws rds generate-db-auth-token --hostname $RDSHOST --port 3306 --region ap-southeast-1 --username cityapp)"
mysql --host=$RDSHOST --port=3306 --user=cityapp --enable-cleartext-plugin --password=$TOKEN
```

## 2.2. Python + boto3
