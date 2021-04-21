# Kafka Cheat Sheet

[![License](https://img.shields.io/badge/License-Apache%202.0-orange.svg?&style=for-the-badge)](https://opensource.org/licenses/Apache-2.0) [<img alt="LinkedIn" src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"/>](https://www.linkedin.com/in/denissavenko/) [<img alt="Blog" src="https://img.shields.io/badge/blog-black.svg?&style=for-the-badge&logo=ghost&logoColor=white"/>](https://dsavenko.me/)

This is a list of commonly used CLI examples, when you work with **Kafka**, **Kafka Connect** and **Schema Registry**.
Feel free to use it as well as post extensions to it.

All commands should be executed from **Apache Kafka** or **Confluent Platform** home directory.
It is also assumed, that Zookeeper, Brokers, Connect Workers and Schema Registry operate on standard ports. Adjust when necessary.

## Table of contents

- [Kafka Cluster Management](#Kafka-Cluster-Management)
- [Kafka Topics](#Kafka-Topics)
- [Kafka Consumers](#Kafka-Consumers)
- [Kafka Producers](#Kafka-Producers)
- [kafkacat](#kafkacat)
- [Kafka Connect](#Kafka-Connect)
- [Schema Registry](#Schema-Registry)

## Kafka Cluster Management

> See also: 
> - ZooKeeper configuration options: https://docs.confluent.io/platform/current/zookeeper/deployment.html#configuration-options
> - Brokers configuration options: https://docs.confluent.io/platform/current/installation/configuration/broker-configs.html
> - REST Proxy v3 for cluster management: https://docs.confluent.io/platform/current/kafka-rest/api.html#crest-api-v3

### Environment setup

```bash
export KAFKA_HEAP_OPTS="-Xmx2G -Xms128M"
export CONFLUENT_HOME="/path/to/confluent-X.X.X"
export PATH="${CONFLUENT_HOME}/bin:$PATH"
export LOG_DIR=/tmp
```

### Starting Kafka Cluster

Start Zookeeper:
```
./bin/zookeeper-server-start ./path/to/zookeeper.properties > ./output/zookeeper.out &
```

Start Kafka Broker:
```
./bin/kafka-server-start ./path/to/broker0.properties > ./output/broker0.out &
```

Start Kafka Connect Worker (distributed mode):
```
./bin/connect-distributed ./path/to/worker0.properties > ./output/worker0.out &
```

Start Schema Registry:
```
./bin/schema-registry-start ./path/to/schema-registry.properties > ./output/schema-registry.out &
```

Start kSQL Server:
```
./bin/ksql-server-start ./path/to/ksql-server.properties > ./output/ksql-server.out &
```

> Note: Standard output of Kafka Cluster processes will be saved in the specified files in this case. It also can be configured in `log4j.properties` which logs should be written and where by each process.

### Stopping Kafka Cluster

> Note: This command searches for PIDs of all Kafka processes, and stops them gracefully.

```bash
#!/bin/bash
jps -m | grep 'QuorumPeerMain\|Kafka\|ConnectDistributed\|SchemaRegistryMain\|KsqlServerMain' | awk '{print $1}' | xargs kill
```

## Kafka Topics

> See also: 
> - https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html
> - REST Proxy v3 for topics management: https://docs.confluent.io/platform/current/kafka-rest/api.html#create-a-topic

### List topics

```
./bin/kafka-topics --zookeeper localhost:2181 --list
```
..or..
```
./bin/kafka-topics --bootstrap-server localhost:9092 --list
```

### Describe a topic

```
./bin/kafka-topics --zookeeper localhost:2181 --describe --topic my-topic
```
..or..
```
./bin/kafka-topics --bootstrap-server localhost:9092 --describe --topic my-topic
```

### Create a topic

> Note: To read more about topic configuration options, refer to [official docs](https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html)

```
./bin/kafka-topics --create --bootstrap-server localhost:9092 --topic my-topic --replication-factor 3 --partitions 3 --config cleanup.policy=compact
```

### Alter topic config

```
./bin/kafka-configs --bootstrap-server localhost:9092 --topic my-topic --alter --add-config 'cleanup.policy=compact,retention.ms=86400000,segment.bytes=1073741824'
```
..or..
```
# deprecated
./bin/kafka-topics --zookeeper localhost:2181 --topic my-topic --alter --config cleanup.policy=compact --config retention.ms=86400000 --config segment.bytes=1073741824
```


> Note: Usage of `kafka-topics` command to alter topics configuration is deprecated, usage of `kafka-configs` command is recommended.

### Delete topic config (reset to default)

```
./bin/kafka-configs --bootstrap-server localhost:9092 --topic my-topic --alter --delete-config 'cleanup.policy,retention.ms'
```

### Purge a topic

```
./bin/kafka-configs --bootstrap-server localhost:9092 --topic my-topic --alter --add-config 'cleanup.policy=delete,retention.ms=100'
```
...wait a minute...
```
./bin/kafka-configs --bootstrap-server localhost:9092 --topic my-topic --alter --delete-config 'cleanup.policy,retention.ms'
```

### Delete a topic

```
./bin/kafka-topics --bootstrap-server localhost:9092 --delete --topic my-topic
```

## Kafka Consumers

> See also: 
> - https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/kafka-commands.html#basic-producer-and-consumer
> - https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html

### Simple consumer console

Plain text:
```
./bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic my-topic --from-beginning
```

Avro:
```
./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 --topic my-topic --property schema.registry.url=http://localhost:8081 --from-beginning
```

### Print key together with value

```bash
./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 \
--topic my-topic \
--property schema.registry.url=http://localhost:8081 \
--property print.key=true \
--property print.value=true \
--property key.separator=":" \
--from-beginning
```

### Use different deserializers for key and value

```bash
./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 \
--topic my-topic \
--property schema.registry.url=http://localhost:8081 \
--property print.key=true \
--property print.value=true \
--property key.separator=":" \
--key-deserializer "org.apache.kafka.common.serialization.StringDeserializer" \
--value-deserializer "io.confluent.kafka.serializers.KafkaAvroDeserializer" \
--from-beginning
```

### Use basic auth for Schema Registry

```bash
export API_KEY="USERNAME"
export API_SECRET="PASSWORD"

./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 \
--topic my-topic \
--property schema.registry.url=http://localhost:8081 \
--property schema.registry.basic.auth.user.info="$API_KEY:$API_SECRET" \
--property basic.auth.credentials.source=USER_INFO \
--property print.key=true \
--property print.value=true \
--property key.separator=":" \
--key-deserializer "org.apache.kafka.common.serialization.StringDeserializer" \
--value-deserializer "io.confluent.kafka.serializers.KafkaAvroDeserializer" \
--from-beginning
```

### Use SASL SSL security for Kafka Broker and Schema Registry connection

```bash
export API_KEY="USERNAME"
export API_SECRET="PASSWORD"

export KAFKA_TRUSTSTORE_LOCATION="/path/to/truststore.jks"
export KAFKA_TRUSTSTORE_PASSPHRASE="<TRUSTSTORE_PASSPHRASE>"
export KAFKA_KEYSTORE_LOCATION=/path/to/keystore.jks
export KAFKA_KEYSTORE_PASSPHRASE="<KEYSTORE_PASSPHRASE>"
export KAFKA_KEY_LOCATION="/path/to/key.pem"
export KAFKA_KEY_PASSPHRASE="<KEY_PASSPHRASE>"

export SCHEMA_REGISTRY_OPTS="-Djavax.net.ssl.keyStore=$KAFKA_KEYSTORE_LOCATION -Djavax.net.ssl.trustStore=$KAFKA_TRUSTSTORE_LOCATION -Djavax.net.ssl.keyStorePassword=$KAFKA_KEYSTORE_PASSPHRASE -Djavax.net.ssl.trustStorePassword=$KAFKA_TRUSTSTORE_PASSPHRASE"

./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 \
--topic my-topic \
--property print.key=true \
--property print.value=true \
--property key.separator=":" \
--key-deserializer "org.apache.kafka.common.serialization.StringDeserializer" \
--value-deserializer "io.confluent.kafka.serializers.KafkaAvroDeserializer" \
--property schema.registry.url=http://localhost:8081 \
--property schema.registry.basic.auth.user.info="$API_KEY:$API_SECRET" \
--property basic.auth.credentials.source=USER_INFO \
--consumer-property security.protocol=SASL_SSL \
--consumer-property sasl.mechanism=PLAIN \
--consumer-property sasl.jaas.config="org.apache.kafka.common.security.plain.PlainLoginModule required username=\"$API_KEY\" password=\"$API_SECRET\";" \
--consumer-property ssl.truststore.location=$KAFKA_TRUSTSTORE_LOCATION \
--consumer-property ssl.truststore.password=$KAFKA_TRUSTSTORE_PASSPHRASE \
--consumer-property ssl.keystore.location=$KAFKA_KEYSTORE_LOCATION \
--consumer-property ssl.keystore.password=$KAFKA_KEYSTORE_PASSPHRASE \
--consumer-property ssl.key.password=$KAFKA_KEY_PASSPHRASE \
--consumer-property ssl.truststore.type=JKS \
--consumer-property ssl.keystore.type=JKS \
--from-beginning
```

### Save consumed Kafka messages to a file

```bash
./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 \
--topic my-topic \
--property print.key=true \
--property print.value=true \
--property key.separator=":" \
--property schema.registry.url=http://localhost:8081 \
--from-beginning
--timeout-ms 25000 \
> ./my-topic-data.json
```

> Note: After consuming all topic messages, the console waits for 25 seconds timeout without new messages, and then exits

## Kafka Producers

> See also: 
> - https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/kafka-commands.html#basic-producer-and-consumer
> - https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html

### Simple producer console

Plain text:
```
./bin/kafka-console-producer --bootstrap-server localhost:9092 --topic my-topic
```

Avro:
```
./bin/kafka-avro-console-producer --bootstrap-server localhost:9092 --topic my-topic --property schema.registry.url=http://localhost:8081 --property value.schema='<value-schema-as-json>'
```

### Parse key together with value

> Note: Topic value schema is read from the file `/path/to/my-topic-value-schema.json`, where schema must be formatted as normal minified JSON (no extra spaces and other whitespace characters).

```bash
./bin/kafka-avro-console-producer --bootstrap-server localhost:9092 \
--topic my-topic \
--property schema.registry.url=http://localhost:8081 \
--property value.schema.file=/path/to/my-topic-value-schema.json \
--property parse.key=true \
--property key.separator=":"
```

### Use different serializer for key

```bash
./bin/kafka-avro-console-producer --bootstrap-server localhost:9092 \
--topic my-topic \
--property schema.registry.url=http://localhost:8081 \
--property value.schema.file=/path/to/my-topic-value-schema.json \
--property parse.key=true \
--property key.separator=":" \
--property key.serializer="org.apache.kafka.common.serialization.StringSerializer" 
```

### Read Kafka messages to produce from a file

```bash
./bin/kafka-avro-console-producer --bootstrap-server localhost:9092 \
--topic my-topic \
--property schema.registry.url=http://localhost:8081 \
--property value.schema.file=/path/to/my-topic-value-schema.json \
--property parse.key=true \
--property key.separator=":" \
--property key.serializer="org.apache.kafka.common.serialization.StringSerializer" \
< ./my-topic-data.json
```

## [kafkacat](https://github.com/edenhill/kafkacat)

> Note: `kafkacat` is a powerful tool to work with Kafka Cluster written in C. It can operate in both consumer and producer mode, it is fast, it can read topic headers, but it currently does not support all Kafka features (i.e. there is no producer mode for Avro serialized topics).

### `kafkacat` for Windows

To build `kafkacat` for Windows from sources, you will need to build it from `win32` directory and have these pre-requisites:
1. NuGet package manager (https://www.nuget.org/downloads)
2. MS Visual C++ Build Tools 14 for Visual Studio 2015 (https://visualstudio.microsoft.com/ru/vs/older-downloads/) - this is not Visual Studio IDE itself, but a subset of stuff to build existing projects. Note this, that exactly this version is currently required to build the project, this can change in future.
3. Add `MSBuild` location to PATH environment variable (usually it is C:\Program Files (x86)\MSBuild\14.0\Bin\)
Change version manually to the current one in win32/win32_config.h (1.6.0 at the moment of writing this)

Now build the project using the official instructions:
```bash
cd win32
nuget restore
msbuild # or, with full path: C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe
```

This should do the trick, you should have the binary then in `%kafkacat_source_base_dir%\win32\x64\Debug`:
```
C:\kafkacat\kafkacat-1.6.0\win32\x64\Debug>kafkacat.exe -V
kafkacat - Apache Kafka producer and consumer tool
https://github.com/edenhill/kafkacat
Copyright (c) 2014-2019, Magnus Edenhill
Version 1.6.0 (Transactions, librdkafka 1.5.0 builtin.features=gzip,snappy,ssl,sasl,regex,lz4,sasl_gssapi,sasl_plain,sasl_scram,plugins,zstd,sasl_oauthbearer)
```

> Note: Currently, the Windows version of `kafkacat` does not support JSON, Avro and Schema Registry

### `kafkacat` for Linux

#### Option 1. Install from repository

Ubuntu/Debian example:
```bash
sudo apt install kafkacat
```

> Note: `kafkacat`, installable from Ubuntu (1.5.0-1.1) or Debian (1.6.0-1) repositories does not support working with Avro and Schema Registry, despite the fact that this functionality was added in version 1.5.0.

#### Option 2. Build from sources

To be able to use `kafkacat` with Avro and Schema registry, download its sources from GitHub and build it yourself from sources with support of `libavro-c` and `libserdes` (as it is mentioned here: https://github.com/edenhill/kafkacat#requirements)

On Ubuntu 20.04, first, you will need these additional package to build the app:
```bash
sudo apt install pkg-config build-essential cmake libtool libssl-dev zlib1g-dev libzstd-dev libsasl2-dev libjansson-dev libcurl4-openssl-dev
```

Then, you can use bootstrap script to build kafkacat with all dependencies:
```bash
./bootstrap.sh
```

Now you are ready to use it:
```
$ ./kafkacat -V
kafkacat - Apache Kafka producer and consumer tool
https://github.com/edenhill/kafkacat
Copyright (c) 2014-2019, Magnus Edenhill
Version 1.6.0 (JSON, Avro, Transactions, librdkafka 1.5.0 builtin.features=gzip,snappy,ssl,sasl,regex,lz4,sasl_gssapi,sasl_plain,sasl_scram,plugins,zstd,sasl_oauthbearer)
```

### kafkacat examples

See `kafkcat` usage examples here: https://github.com/edenhill/kafkacat#examples

## Kafka Connect

> Note: For Kafka Connect and Schema Registry you will need `curl` and `jq` utilities to make requests to their APIs.

> See also:
> - Workers configuration options: https://docs.confluent.io/platform/current/connect/references/allconfigs.html
> - Connectors configuration options: https://docs.confluent.io/platform/current/installation/configuration/connect/index.html
> - Kafka Connect REST API Reference: https://docs.confluent.io/platform/current/connect/references/restapi.html

### List installed Kafka Connect plugins

```
curl -Ss -X GET http://localhost:8083/connector-plugins | jq
```

### List the connectors

```
curl -Ss -X GET http://localhost:8083/connectors | jq
```

### Deploy a connector

```
curl -Ss -X POST -H "Content-Type: application/json" --data @/path/to/my-topic-connector-config.json http://localhost:8083/connectors | jq
```

### Get connector overview (configuration and tasks overview)

```
curl -Ss -X GET http://localhost:8083/connectors/<connector-name> | jq
```

### Get connector config

```
curl -Ss -X GET http://localhost:8083/connectors/<connector-name>/config | jq
```

### Get connector status

```
curl -Ss -X GET http://localhost:8083/connectors/<connector-name>/status | jq
```

### Restart connector

```
curl -Ss -X POST http://localhost:8083/connectors/<connector-name>/restart | jq
```

### Get connector tasks

```
curl -Ss -X GET http://localhost:8083/connectors/<connector-name>/tasks | jq
```

### Get connector task status

```
curl -Ss -X GET http://localhost:8083/connectors/<connector-name>/tasks/0/status | jq
```

### Restart connector task

```
curl -Ss -X POST http://localhost:8083/connectors/<connector-name>/tasks/0/restart | jq
```

### Remove connector

```
curl -Ss -X DELETE http://localhost:8083/connectors/<connector-name> | jq
```

### Get current logging levels

```
curl -Ss http://localhost:8083/admin/loggers | jq
```

### Set logging level for a particular logger

```
curl -Ss -X PUT -H "Content-Type:application/json" http://localhost:8083/admin/loggers/<logger-name> -d '{"level": "DEBUG"}' | jq
```

**Examples:**

```
# sets debug log level for JDBC connector (source and sink)
curl -Ss -X PUT -H "Content-Type:application/json" http://localhost:8083/admin/loggers/io.confluent.connect.jdbc -d '{"level": "DEBUG"}' | jq

# sets debug log level for Kafka Connect Worker Sink tasks
curl -Ss -X PUT -H "Content-Type:application/json" http://localhost:8083/admin/loggers/org.apache.kafka.connect.runtime.WorkerSinkTask -d '{"level": "DEBUG"}' | jq

# sets debug log level for Kafka Connect in general
curl -Ss -X PUT -H "Content-Type:application/json" http://localhost:8083/admin/loggers/org.apache.kafka.connect -d '{"level": "DEBUG"}' | jq
```

## Schema Registry

> See also:
> - Schema Registry REST API Reference: https://docs.confluent.io/platform/current/schema-registry/develop/api.html
> - REST API Usage Examples: https://docs.confluent.io/platform/current/schema-registry/develop/using.html#common-sr-api-usage-examples

### Retrieve currently registered schemas (subjects)

```
curl -Ss -X GET http://localhost:8081/subjects | jq
```

### Retrieve schema versions of a subject

```
curl -Ss -X GET http://localhost:8081/subjects/my-topic-value/versions | jq
```

### Retrieve schema of a subject of a particular version

With metadata (schema ID, version, schema value as escaped JSON):
```
curl -Ss -X GET http://localhost:8081/subjects/my-topic-value/versions/1 | jq
```

Without metadata (schema value as normal JSON):
```
curl -Ss -X GET http://localhost:8081/subjects/my-topic-value/versions/1/schema | jq
```

### Retrieve last schema of a subject

With metadata (schema ID, version, schema value as escaped JSON):
```
curl -Ss -X GET http://localhost:8081/subjects/my-topic-value/versions/latest | jq
```

Without metadata (schema value as normal JSON):
```
curl -Ss -X GET http://localhost:8081/subjects/my-topic-value/versions/latest/schema | jq
```

### Retrieve last schema of a subject, use certificate for the connection to Schema Registry

```
export API_KEY="USERNAME"
export API_SECRET="PASSWORD"

export KEY_LOCATION="/path/to/key.pem"
export KEY_PASSPHRASE="<KEY_PASSPHRASE>"

curl -Ss --cert $KEY_LOCATION:$KEY_PASSPHRASE \
-X GET http://$API_KEY:$API_SECRET@localhost:8081/subjects/my-topic-value/versions/latest/schema
```

### Retrieve last schema of a subject and save it into file

> Note: Schema saved using this command, is suitable for `value.schema.file` property of `kafka-avro-console-producer` utility

```
curl -Ss -X GET http://localhost:8081/subjects/my-topic-value/versions/latest/schema > ./my-topic-value-schema.json
```

### Deploy schema for a subject

> Note: Schema is read from `my-topic-value-schema.json` file from the current directory. Mind the fact, that schema value format must be formatted as mimified escaped JSON, i.e.: `{"schema":"{\"type\":\"string\"}"}`.

```
curl -Ss -X POST -H "Content-Type: application/json" --data @/path/to/my-topic-value-schema.json http://localhost:8081/subjects/my-topic-value/versions | jq
```


