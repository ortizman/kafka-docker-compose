## Levantar todo el ambiente
```shell
docker-compose --profile all up -d
```
## Bajar el ambiente
```shell
docker-compose --profile all down
```
## Ver logs del connector (similar para otros)
```shell
docker-compose logs -f connect
```
## Ejecutar KSQLDB-CLI (crear connectores, streams, tablas, etc)
```shell
docker-compose exec ksqldb-cli ksql http://ksqldb-server:8088 
```
## ingresar a influxdb
```shell
docker-compose exec influxdb influx
```

## crear conectores jdbc mysql
```sql
CREATE SOURCE CONNECTOR `jdbc-connector-transactions` WITH("connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector', "connection.url"='jdbc:mysql://172.17.0.1:3306/ptsmock', "mode"='incrementing', "topic.prefix"='jdbc_', "table.whitelist"='transactions', "key"='beneficiaryId', "connection.user"='ptsmock', "connection.password"='ptsmock' );
```
## setear lectura de los topicos desde el principio
```sql
SET 'auto.offset.reset' = 'earliest';
```

## crear stream de transactions en ksqldb
```sql
CREATE STREAM RAW_TRANSACTIONS (schema struct<type string, fields array<struct<type string, field string, optional boolean, name string, version int>>, optional boolean, name string>, payload struct<id int, beneficiaryId int, originId int, amount int, status string, creationDate bigint, serviceId string, channelId string, reference string> ) with (kafka_topic='jdbc_transactions', value_format='JSON');
```

## Mapear Transacciones para cargarlas en influxdb
```sql
create stream tx_schemaless with (kafka_topic='tx_influxdb') as select struct(service:=payload->serviceId, channel:=payload->channelId) as "tags", payload->amount as "amount", payload->creationDate as "creationDate" from RAW_TRANSACTIONS emit changes;
```

## crear los conectores de kafka streams a influxdb
```sql
CREATE SINK CONNECTOR SINK_INFLUX_TX WITH (
    'connector.class'               = 'io.confluent.influxdb.InfluxDBSinkConnector',
    'topics'                        = 'tx_influxdb',
    'influxdb.url'                  = 'http://influxdb:8086',
    'influxdb.db'                   = 'adaptor_pts',
    'measurement.name.format'       = 'tx',
    'event.time.fieldname'          = 'creationDate',
    'value.converter'               = 'org.apache.kafka.connect.json.JsonConverter',
    'key.converter'                 = 'org.apache.kafka.connect.json.JsonConverter',
    'key.converter.schemas.enable'  = false,
    'value.converter.schemas.enable'= false
);
```

### Generar TX dummy con Kafkacat
```shell
docker run -v /home/manuel/desarrollo/2inn/kafka-docker/test/pts-tx-event-dummy.json:/data/tx-dummy.json -i --network kafka-docker_default confluentinc/cp-kafkacat kafkacat -b broker:29092 -t test -D*** -P -l /data/tx-dummy.json
```
## Create Stream PTS_TX_ENTITIES
```sql
create stream PTS_TX_ENTITIES (
  EventData STRUCT<transactionId STRING, originationHost STRING, serviceId STRING, channelId STRING, trxType STRING, currency STRING, totalAmount STRING, status STRING, initialTime STRING, duration STRING, entities ARRAY <STRUCT<entityId STRING, initialTime STRING, duration DOUBLE, resultCode STRING, resultDesc STRING>>> ) 
with (KEY_FORMAT='NONE', WRAP_SINGLE_VALUE=true, VALUE_FORMAT='JSON', KAFKA_TOPIC='test');
```
### Stream con entidades de la TX como filas
```sql
create stream TX_TO_INFLUX(kafka_topic="pepe") as 
select 
  EXPLODE(EventData->entities)->entityId as entityId, 
  EXPLODE(EventData->entities)->initialTime as entityInitialTime,  
  EXPLODE(EventData->entities)->duration as entityDuration , 
  EXPLODE(EventData->entities)->resultCode as resultCode, 
  EXPLODE(EventData->entities)->resultDesc as resultDesc, 
  eventData->transactionId as transactionId,
  eventData->originationHost as originationHost,
  eventData->serviceId as serviceId,
  eventData->channelId as channelId,
  eventData->trxType as trxType,
  eventData->currency as currency,
  eventData->totalAmount as totalAmount,
  eventData->status as status,
  eventData->initialTime as initialTime,
  eventData->duration as duration
from PTS_TX_ENTITIES emit changes;
```

## Grafana Dashboard
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