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

Ejemplo
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