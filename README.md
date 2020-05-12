# Ksql Debezium Demo

This project setup a pipeline where two MySql Tables are configured for CDC using Debezium: Customers and Orders. The changes go to two Kafka topics and using Ksql, the stream of Orders is enhance with the related Customer information. They resulting stream is sent to ElasticSearch sink to store the enhanced orders.

## Setup environment

    docker-compose up -d

## Connect to ksqldb server

    docker exec -it ksqldb bash -c 'ksql http://ksqldb:8088'

## Create source connectors

```
CREATE SOURCE CONNECTOR SOURCE_CUSTOMER WITH (
    'connector.class' = 'io.debezium.connector.mysql.MySqlConnector',
    'database.hostname' = 'mysql',
    'database.port' = '3306',
    'database.user' = 'debezium',
    'database.password' = 'dbz',
    'database.server.id' = '101',
    'database.server.name' = 'dbserver',
    'table.whitelist' = 'inventory.customers,inventory.orders',
    'database.history.kafka.bootstrap.servers' = 'kafka:29092',
    'database.history.kafka.topic' = 'dbhistory.demo' ,
    'include.schema.changes' = 'false',
    'transforms'= 'unwrap',
    'transforms.unwrap.type'= 'io.debezium.transforms.ExtractNewRecordState',
    'key.converter'= 'org.apache.kafka.connect.storage.StringConverter',
    'value.converter'= 'io.confluent.connect.avro.AvroConverter',
    'value.converter.schema.registry.url'= 'http://schema-registry:8081'
    );
```

## Create Streams and Table from Topics

```
SET 'auto.offset.reset' = 'earliest';
CREATE STREAM customers_debezium WITH (KAFKA_TOPIC='dbserver.inventory.customers',VALUE_FORMAT='AVRO');
CREATE STREAM customers_stream WITH(KAFKA_TOPIC='customers_by_userId') AS 
SELECT CAST(id AS INTEGER) AS id, first_name, last_name FROM customers_debezium PARTITION BY id EMIT CHANGES;
CREATE TABLE customers (ROWKEY INT PRIMARY KEY, first_name VARCHAR, last_name VARCHAR) WITH (KAFKA_TOPIC='customers_by_userId', VALUE_FORMAT='AVRO');

CREATE STREAM orders_debezium WITH (KAFKA_TOPIC='dbserver.inventory.orders',VALUE_FORMAT='AVRO');
CREATE STREAM orders_stream AS SELECT * FROM orders_debezium PARTITION BY purchaser;

CREATE STREAM ORDERS_WITH_CUSTOMER
       WITH (KAFKA_TOPIC='orders_with_customer')
       AS
SELECT o.ORDER_NUMBER as order_number, o.product_id, o.quantity,
       c.ROWKEY AS customer_id, c.first_name + ' ' + c.last_name AS full_name
FROM orders_stream o LEFT JOIN customers c ON o.purchaser = c.ROWKEY
PARTITION BY order_number
EMIT CHANGES;
```

## Create sink connector

```
CREATE SINK CONNECTOR ELASTIC_AGGREGATE WITH (
  'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
  'connection.url' = 'http://elasticsearch:9200',
  'type.name' = '',
  'behavior.on.malformed.documents' = 'warn',
  'errors.tolerance' = 'all',
  'errors.log.enable' = 'true',
  'errors.log.include.messages' = 'true',
  'topics' = 'orders_with_customer',
  'schema.ignore' = 'true',
  'key.converter' = 'org.apache.kafka.connect.storage.StringConverter',
  'transforms'= 'ExtractTimestamp',
  'transforms.ExtractTimestamp.type'= 'org.apache.kafka.connect.transforms.InsertField$Value',
  'transforms.ExtractTimestamp.timestamp.field' = 'EXTRACT_TS'
);
```

## Test integration

Connect to Mysql:

    docker-compose exec mysql bash -c 'mysql -u $MYSQL_USER -p$MYSQL_PASSWORD inventory'

Check notifications in the output topic:

    SELECT * FROM ORDERS_WITH_CUSTOMER emit changes;

Query ElasticSearch:

```
curl -X GET \
  http://localhost:9200/orders_with_customer/_search \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
        "match_all": {}
    }
}'
```

Trigger a change in the database:

    UPDATE orders SET quantity=20  WHERE order_number=10001;

## Clean up

    docker-compose down

