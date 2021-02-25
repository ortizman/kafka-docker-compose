# Construir un nuevo connect (usando Docker)

1. Extender la imagen base de docker connector con el nuevo connector
   Ej:
   ```docker
    FROM confluentinc/cp-kafka-connect-base:6.1.0
    RUN confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.0.1
   ```
2. En caso de ser necesario, instalar cualquier software adicional (como un driver JDBC)
   ```docker
    [...]
    ENV MYSQL_DRIVER_VERSION 5.1.39

    USER root
    RUN curl -k -SL "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-${MYSQL_DRIVER_VERSION}.tar.gz" \
     | tar -xzf - -C /usr/share/java/kafka/ --strip-components=1 mysql-connector-java-${MYSQL_DRIVER_VERSION}/mysql-connector-java-${MYSQL_DRIVER_VERSION}-bin.jar
    USER appuser
    ```
3. construir la nueva imagen
    ```shell
    docker build . -t mysql-connect:1.0.2
    ```

·
# Crear el conector usando KSQLDB
Iniciar el cliente de ksqldb
```shell
docker-compose -f docker-compose-cp-community.yml exec ksqldb-cli ksql http://ksqldb-server:8088
```

# Demo

## Crear Base de datos relacional de ejemplo

### DDL BBDD
```sql
CREATE DATABASE `ptsmock` /*!40100 DEFAULT CHARACTER SET utf8 */;
```

### DDL Tabla de usuarios
```sql
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `created_date` datetime(6) DEFAULT NULL,
  `dni` varchar(255) DEFAULT NULL,
  `due_change_password` bit(1) DEFAULT NULL,
  `email` varchar(255) NOT NULL,
  `enable` bit(1) NOT NULL,
  `first_name` varchar(255) DEFAULT NULL,
  `last_name` varchar(255) DEFAULT NULL,
  `password` varchar(255) DEFAULT NULL,
  `phone` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `email` (`email`),
  UNIQUE KEY `dni` (`dni`)
) ENGINE=InnoDB AUTO_INCREMENT=706 DEFAULT CHARSET=utf8;
```

### Insert de datos
```sql
INSERT INTO `user` (created_date,dni,due_change_password,email,enable,first_name,last_name,password,phone) VALUES
	 ('2020-01-21 00:00:00.0','32345100',0,'admin@yopmail.com',1,'Admin','Admin','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-03-21 00:00:00.0','32345200',0,'usuario00@yopmail.com',1,'usuario 00','Prueba 00','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-03-21 00:00:00.0','32345201',0,'usuario01@yopmail.com',1,'usuario 01','Prueba 01','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-03-21 00:00:00.0','32345202',0,'usuario02@yopmail.com',1,'usuario 02','Prueba 02','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-04-21 00:00:00.0','32345300',0,'usuario03@yopmail.com',1,'usuario 00','Prueba 00','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-04-21 00:00:00.0','32345301',0,'usuario04@yopmail.com',1,'usuario 01','Prueba 01','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-04-21 00:00:00.0','32345302',0,'usuario05@yopmail.com',1,'usuario 02','Prueba 02','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-04-21 00:00:00.0','32345303',0,'usuario06@yopmail.com',1,'usuario 03','Prueba 03','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-04-21 00:00:00.0','32345304',0,'usuario07@yopmail.com',1,'usuario 04','Prueba 04','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL),
	 ('2020-04-21 00:00:00.0','32345305',0,'usuario08@yopmail.com',1,'usuario 05','Prueba 05','$2a$10$oAgHnrcXV4zf4pWacGh75O1c8FVv.Kw8p0qSWqML3/5.5g9hxCh8q',NULL);
```

## Crear el connector MySQL >> Connector >> Kafka 
```sql
CREATE SOURCE CONNECTOR `jdbc-connector` WITH("connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector', "connection.url"='jdbc:mysql://192.168.1.121:3306/iplycdb', "mode"='incrementing', "topic.prefix"='jdbc-', "table.whitelist"='user', "key"='email', "connection.user"='iplyc-user-db', "connection.password"='' );
```

Chequear el nuevo topico
```shell
print 'topic-name' from beginning;
```

Dropear un connector
```shell
drop connector 'topic-name';
```

Parar todos los servicios
```shell
docker-compose -f docker-compose-cp-community.yml down
```