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


# Deployment

## Local

Todos los servicios estan construidos sobre Docker. Usando la herramienta docker-compose se puede levantar un ambiente local (o en cualquier servidor con docker).

### Levantar todos los servicios

```shell
docker-compose --context default --profile datalake up -d
```

### Bajar todos los servicios

```shell
docker-compose --context default down
```
## AWS ECS
Existe la posibilidad de deployar estos servicios usando ECS como plataforma. Los siguientes links pueden servir de guia para cumplir los pre-requisitos:
* https://docs.docker.com/cloud/ecs-integration/
* https://aws.amazon.com/blogs/containers/deploy-applications-on-amazon-ecs-using-docker-compose/

A continuación se explican muy brevemente los pasos a seguir para deployar sobre AWS ECS.

### Crear un nuevo contexto en Docker
```shell
docker context create ecs aws-ecs
```

El comando anterior mostrará un menú similiar al siguiente:

```shell
? Create a Docker context using:  [Use arrows to move, type to filter]
  An existing AWS profile
  AWS secret and token credentials
> AWS environment variables
```

donde se debe seleccionar el modo en que se proveerá las cerdenciales de AWS.


### Desplegar los servicios

De forma similar a como se opera con el comando docker-compose, se puede desplegar usando el comando __ docker compose __. A continuación un ejemplo:

```shell
docker compose --context aws-ecs up connect 
```

El comando anterior creará un nuevo cluster (si ya no existe) y desplegará el container `connect` con todas sus dependencias (Kafka Broker y Zookeeper) usando AWS ECS.

### Chequear los servicios levantados

```shell
docker compose --context aws-ecs ps
```

### Visualizar logs

```shell
docker compose --context aws-ecs logs connect
```

### Eliminar los servicios

```shell
docker compose --context aws-ecs down
```

---
# Crear un conector usando KSQLDB
Iniciar el cliente de ksqldb
```shell
docker-compose -f docker-compose-cp-community.yml exec ksqldb-cli ksql http://ksqldb-server:8088
```

# Demo

Se creará una base de datos con una tabla y datos mocks. Se extraeran los datos de esa tabla usando el conector JDBC de Kafka Connect y se enviara a un tópico. Luego, usando una expresion SQL en KSQLDB se transforaran los datos para posteriormente ser escritos en otro topico. Nuevamente, usando un conector de salida, se consumieran esos datos para escribirlos en influxdb y usar Grafana para crear nuestro dashboard.

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
CREATE SOURCE CONNECTOR `jdbc-connector` WITH("connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector', "connection.url"='jdbc:mysql://172.17.0.1:3306/ptsmock', "mode"='incrementing', "topic.prefix"='jdbc_', "table.whitelist"='user', "key"='email', "connection.user"='ptsmock', "connection.password"='' );
```
> Para connectarse al ksqldb-cli con docker se puede usar el siguiente comando: 
``` docker-compose exec ksqldb-cli ksql http://ksqldb-server:8088 ```

### Chequear el nuevo topico
```shell
print 'jdbc-user' from beginning;
```

## Crear un conector de salida para influxdb
```sql
CREATE SINK CONNECTOR SINK_INFLUX_01 WITH (
        'connector.class'               = 'io.confluent.influxdb.InfluxDBSinkConnector',
        'value.converter'               = 'org.apache.kafka.connect.json.JsonConverter',
        'topics'                        = 'parser_user_schema',
        'influxdb.url'                  = 'http://influxdb:8086',
        'influxdb.db'                   = 'adaptor_pts',
        'measurement.name.format'       = 'metrics',
        'key.converter.schemas.enable'  = 'false',
        'value.converter.schemas.enable'= 'true',
        'schemas.enable'                = 'false'
  );
```

## Comandos útiles

### Dropear un connector
```shell
drop connector 'jdbc-user';
```

### Stopear todos los servicios
```shell
docker-compose -f docker-compose-cp-community.yml down
```

