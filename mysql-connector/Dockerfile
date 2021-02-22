FROM confluentinc/cp-kafka-connect-base:6.1.0

ENV MYSQL_DRIVER_VERSION 5.1.39

RUN confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.0.1

USER root
RUN curl -k -SL "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-${MYSQL_DRIVER_VERSION}.tar.gz" \
     | tar -xzf - -C /usr/share/java/kafka/ --strip-components=1 mysql-connector-java-${MYSQL_DRIVER_VERSION}/mysql-connector-java-${MYSQL_DRIVER_VERSION}-bin.jar

USER appuser