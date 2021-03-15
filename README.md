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

Todos los servicios estan construidos sobre Docker. 
Usando la herramienta __docker-compose__ se puede levantar un ambiente local (o en cualquier servidor con docker).

### Levantar los servicios esensiales del DataLake

```shell
docker-compose --context default --profile datalake up -d
```

### Levantar todos los servicios (excepto PTS)

```shell
docker-compose --context default --profile all up -d
```

### Levantar solo PTS

```shell
docker-compose --context default --profile pts up -d
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
>El comando anterior mostrará un menú similiar al siguiente:

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
> IMPORTANTE: se debe especificar el contexto `aws-ecs` en cada comando

El comando anterior creará un nuevo cluster (si aún no existe) y desplegará el container `connect` con todas sus dependencias (Kafka Broker y Zookeeper) usando AWS ECS.

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

# Demo

Se creará una base de datos con una tabla y datos mocks. Se extraeran los datos de esa tabla usando el conector JDBC de Kafka Connect y se enviara a un tópico. Luego, usando una expresion SQL en KSQLDB se transforaran los datos para posteriormente ser escritos en otro tópico. Nuevamente, usando un conector de salida, se consumieran esos datos para escribirlos en influxdb y usar Grafana para crear nuestro dashboard.

## Crear Base de datos relacional de __ejemplo__

### DDL BBDD
```sql
CREATE DATABASE `ptsmock` /*!40100 DEFAULT CHARACTER SET utf8 */;
```

### DDL Tabla de "Transacciones"
```sql
CREATE TABLE `transactions` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `beneficiaryId` bigint NOT NULL,
  `originId` bigint DEFAULT NULL,
  `amount` bigint NOT NULL,
  `status` varchar(100) DEFAULT NULL,
  `creationDate` datetime DEFAULT NULL,
  `serviceId` bigint DEFAULT NULL,
  `channelId` bigint DEFAULT NULL,
  `reference` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `transactions_ID_IDX` (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=15 DEFAULT CHARSET=utf8;
```

### Insert de datos
```sql
INSERT INTO ptsmock.transactions (beneficiaryId,originId,amount,status,creationDate,serviceId,channelId,reference) VALUES
	 (101,200,100,'PENDING','2021-03-05 12:57:06.0',3,4,'Prueba'),
	 (102,201,400,'PENDING','2021-03-05 12:57:07.0',3,4,'Prueba'),
	 (101,202,1000,'PENDING','2021-03-05 12:57:07.0',3,4,'Prueba'),
	 (102,200,1500,'PENDING','2021-03-05 19:30:44.0',3,4,'Prueba'),
	 (104,201,1200,'PENDING','2021-03-05 19:35:52.0',3,4,'Prueba'),
	 (103,202,1300,'PENDING','2021-03-05 19:36:59.0',2,4,'Prueba'),
	 (102,200,1400,'PENDING','2021-03-05 19:37:05.0',1,4,'Prueba'),
	 (104,200,1100,'PENDING','2021-03-05 19:38:16.0',1,4,'Prueba'),
	 (103,201,1200,'PENDING','2021-03-05 19:39:17.0',1,4,'Prueba'),
	 (102,203,800,'PENDING','2021-03-05 19:42:27.0',1,4,'Prueba'),
	 (101,204,1800,'PENDING','2021-03-05 19:50:48.0',1,4,'Prueba');
```

> Para los siguientes comandos es necesario crear una sesión en KSQLDB

### Iniciar el cliente de KSQLDB
```shell
docker-compose -f docker-compose-cp-community.yml exec ksqldb-cli ksql http://ksqldb-server:8088
```

### Crear el connector MySQL -> Kafka 
```sql
CREATE SOURCE CONNECTOR `jdbc-connector-transactions` WITH(
    "connector.class"     = 'io.confluent.connect.jdbc.JdbcSourceConnector',
    "connection.url"      = 'jdbc:mysql://172.17.0.1:3306/ptsmock', 
    "mode"                = 'incrementing',
    "topic.prefix"        = 'jdbc_',
    "table.whitelist"     = 'transactions',
    "key"                 = 'beneficiaryId',
    "connection.user"     = 'ptsmock',
    "connection.password" = 'ptsmock' 
  );
```

### Chequear el nuevo topico
```shell
print 'jdbc_transactions' from beginning;
```

# setear lectura de los topicos desde el principio
SET 'auto.offset.reset' = 'earliest';

> El comando anterior indicar a Kafka que los nuevos clientes debe comenzar a consumir los mensajes desde el comienzo del tópico

# crear stream de transactions en ksqldb
CREATE STREAM RAW_TRANSACTIONS (
    schema struct<type string, fields array<struct<type string, field string, optional boolean, name string, version int>>, optional boolean, name string>, 
    payload struct<id int, beneficiaryId int, originId int, amount int, status string, creationDate bigint, serviceId string, channelId string, reference string> 
  ) with (kafka_topic='jdbc_transactions', value_format='JSON');

El anterior comando crea un __stream__ de nombre `RAW_TRANSACTIONS` en KSQLDB. El origen del __stream__ es el tópico `jdbc_transactions`. Los datos en el stream serán los especificados.

# Mapear transacciones para cargarlas en influxdb
CREATE STREAM tx_schemaless WITH (kafka_topic='tx_influxdb') as 
  SELECT struct(service:=payload->serviceId, channel:=payload->channelId) as "tags", payload->amount as "amount", payload->creationDate as "creationDate"
from RAW_TRANSACTIONS emit changes;

El stream anterior procesa los datos y los carga en un nuevo tópico de nombre `tx_influxdb`.

## Crear un conector de salida para influxdb
```sql
CREATE SINK CONNECTOR SINK_INFLUX_TX WITH (
    'connector.class'               = 'io.confluent.influxdb.InfluxDBSinkConnector',
    'topics'                        = 'tx_influxdb',
    'influxdb.url'                  = 'http://influxdb:8086',
    'influxdb.db'                   = 'adaptor_pts',
    'measurement.name.format'       = 'tx',
    'event.time.fieldname'          = 'creationDate'
    'value.converter'               = 'org.apache.kafka.connect.json.JsonConverter',
    'key.converter'                 = 'org.apache.kafka.connect.json.JsonConverter',
    'key.converter.schemas.enable'  = false,
    'value.converter.schemas.enable'= false
);
```

# Grafana Dashboard
```json
{
  "aliasColors": {},
  "dashLength": 10,
  "fieldConfig": {
    "defaults": {
      "custom": {}
    },
    "overrides": []
  },
  "fill": 1,
  "gridPos": {
    "h": 9,
    "w": 12,
    "x": 0,
    "y": 0
  },
  "id": 23763571993,
  "legend": {
    "avg": false,
    "current": false,
    "max": false,
    "min": false,
    "show": true,
    "total": false,
    "values": false
  },
  "lines": true,
  "linewidth": 1,
  "nullPointMode": "null",
  "options": {
    "alertThreshold": true
  },
  "pluginVersion": "7.4.3",
  "pointradius": 2,
  "renderer": "flot",
  "seriesOverrides": [],
  "spaceLength": 10,
  "targets": [
    {
      "groupBy": [
        {
          "params": [
            "1d"
          ],
          "type": "time"
        },
        {
          "params": [
            "0"
          ],
          "type": "fill"
        }
      ],
      "measurement": "tx",
      "orderByTime": "ASC",
      "policy": "autogen",
      "refId": "A",
      "resultFormat": "time_series",
      "select": [
        [
          {
            "params": [
              "amount"
            ],
            "type": "field"
          },
          {
            "params": [],
            "type": "sum"
          }
        ]
      ],
      "tags": [
        {
          "key": "SERVICE",
          "operator": "=",
          "value": "1"
        }
      ]
    }
  ],
  "thresholds": [],
  "timeRegions": [],
  "title": "Panel Title",
  "tooltip": {
    "shared": true,
    "sort": 0,
    "value_type": "individual"
  },
  "type": "graph",
  "xaxis": {
    "buckets": null,
    "mode": "time",
    "name": null,
    "show": true,
    "values": []
  },
  "yaxes": [
    {
      "format": "short",
      "label": null,
      "logBase": 1,
      "max": null,
      "min": null,
      "show": true
    },
    {
      "format": "short",
      "label": null,
      "logBase": 1,
      "max": null,
      "min": null,
      "show": true
    }
  ],
  "yaxis": {
    "align": false,
    "alignLevel": null
  },
  "bars": false,
  "dashes": false,
  "fillGradient": 0,
  "hiddenSeries": false,
  "percentage": false,
  "points": false,
  "stack": false,
  "steppedLine": false,
  "timeFrom": null,
  "timeShift": null,
  "datasource": null
}
```

## Comandos útiles

### Dropear un connector
```shell
drop connector 'jdbc_transactions';
```

### Stopear todos los servicios
```shell
docker-compose --profile all down
```

# query para dashboard en Grafana
SELECT sum("amount") FROM "autogen"."tx" WHERE ("SERVICE" = '1') AND time >= now() - 2d GROUP BY time(1d) fill(0)