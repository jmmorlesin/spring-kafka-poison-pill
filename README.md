# Spring Kafka beyond the basics - Poison Pill

Example project used for my:

* [Kafka Summit 2020 lightning talk](https://events.kafka-summit.org/2020-schedule#element-custom-block-2986770903)
* [Confluent Online talks session](https://www.confluent.io/online-talks/spring-kafka-beyond-the-basics/)
* [Confluent blog post - Spring for Apache Kafka – Beyond the Basics: Can Your Kafka Consumers Handle a Poison Pill?](https://www.confluent.io/blog/spring-kafka-can-your-kafka-consumers-handle-a-poison-pill/)

Using:

* Confluent Kafka 5.5
* Confluent Schema Registry 5.5
* Java 8
* Spring Boot 2.3
* Spring Kafka 2.5
* Apache Avro 1.9.2

## Project modules and applications

| Applications                                  | Port | Avro  | Topic(s)          | Description                                                              |
|-----------------------------------------------|------|-------|-------------------|--------------------------------------------------------------------------|
| stock-quote-producer-avro                     | 8080 | YES  | stock-quotes-avro  | Simple producer of random stock quotes using Spring Kafka & Apache Avro. |
| stock-quote-consumer-avro                     | 8082 | YES  | stock-quotes-avro  | Simple consumer of stock quotes using using Spring Kafka & Apache Avro.  |
| stock-quote-kafka-streams-avro                | 8083 | YES  | stock-quotes-avro  | Simple Kafka Streams application using Spring Kafka & Apache Avro.  |

| Module                                   | Description                                                             |
|------------------------------------------|-------------------------------------------------------------------------|
| stock-quote-avro-model                   | Holds the Avro schema for the Stock Quote including `avro-maven-plugin` to generate Java code based on the Avro Schema. This module is used by both the producer and consumer application.  |

Note Confluent Schema Registry is running on port: `8081` using Docker see: [docker-compose.yml](docker-compose.yml). 

![](project-overview.png)

## Goal

The goal of this example project is to show how protect your Kafka application against Deserialization exceptions (a.k.a. poison pills) leveraging Spring Boot and Spring Kafka.

This example project has 3 different branches:

* `master` : no configuration to protect the consumer application (`stock-quote-consumer-avro`) against the poison pill scenario.
* `handle-poison-pill-log-and-continue-consuming` : configuration to protect the consumer application (`stock-quote-consumer-avro`) against the poison pill scenario by simply logging the poison pill(s) and continue consuming.
* `handle-poison-pill-dead-letter-topic-and-continue-consuming` : configuration to protect the consumer application (`stock-quote-consumer-avro`) against the poison pill scenario by publishing the poison pill(s) to a dead letter topic `stock-quotes-avro.DLT` and continue consuming.

## Build to the project

```
./mvnw clean install
```

## Start / Stop Kafka & Zookeeper

```
docker-compose up -d
```

```
docker-compose down -v
```

## Execute the Poison Pill scenario yourself

First make sure all Docker containers started successfully (state: `Up`):

```
docker-compose ps
``` 

### Start both producer and consumer

* Spring Boot application: `stock-quote-producer-avro`

```
java -jar stock-quote-producer-avro/target/stock-quote-producer-avro-0.0.1-SNAPSHOT.jar
```

* Spring Boot application: `stock-quote-consumer-avro`

```
java -jar stock-quote-consumer-avro/target/stock-quote-consumer-avro-0.0.1-SNAPSHOT.jar
```

### Attach to Kafka

```
docker exec -it kafka bash
```

### Unset JMX Port 

To avoid error:

```
unset JMX_PORT
```

```
Error: JMX connector server communication error: service:jmx:rmi://kafka:9999
sun.management.AgentConfigurationError: java.rmi.server.ExportException: Port already in use: 9999; nested exception is: 
        java.net.BindException: Address already in use (Bind failed)
        at sun.management.jmxremote.ConnectorBootstrap.exportMBeanServer(ConnectorBootstrap.java:800)
        at sun.management.jmxremote.ConnectorBootstrap.startRemoteConnectorServer(ConnectorBootstrap.java:468)
        at sun.management.Agent.startAgent(Agent.java:262)
        at sun.management.Agent.startAgent(Agent.java:452)
Caused by: java.rmi.server.ExportException: Port already in use: 9999; nested exception is: 
        java.net.BindException: Address already in use (Bind failed)
        at sun.rmi.transport.tcp.TCPTransport.listen(TCPTransport.java:346)
        at sun.rmi.transport.tcp.TCPTransport.exportObject(TCPTransport.java:254)
        at sun.rmi.transport.tcp.TCPEndpoint.exportObject(TCPEndpoint.java:411)
        at sun.rmi.transport.LiveRef.exportObject(LiveRef.java:147)
        at sun.rmi.server.UnicastServerRef.exportObject(UnicastServerRef.java:237)
        at sun.management.jmxremote.ConnectorBootstrap$PermanentExporter.exportObject(ConnectorBootstrap.java:199)
        at javax.management.remote.rmi.RMIJRMPServerImpl.export(RMIJRMPServerImpl.java:146)
        at javax.management.remote.rmi.RMIJRMPServerImpl.export(RMIJRMPServerImpl.java:122)
        at javax.management.remote.rmi.RMIConnectorServer.start(RMIConnectorServer.java:404)
        at sun.management.jmxremote.ConnectorBootstrap.exportMBeanServer(ConnectorBootstrap.java:796)
        ... 3 more
Caused by: java.net.BindException: Address already in use (Bind failed)
        at java.net.PlainSocketImpl.socketBind(Native Method)
        at java.net.AbstractPlainSocketImpl.bind(AbstractPlainSocketImpl.java:387)
        at java.net.ServerSocket.bind(ServerSocket.java:375)
        at java.net.ServerSocket.<init>(ServerSocket.java:237)
        at java.net.ServerSocket.<init>(ServerSocket.java:128)
        at sun.rmi.transport.proxy.RMIDirectSocketFactory.createServerSocket(RMIDirectSocketFactory.java:45)
        at sun.rmi.transport.proxy.RMIMasterSocketFactory.createServerSocket(RMIMasterSocketFactory.java:345)
        at sun.rmi.transport.tcp.TCPEndpoint.newServerSocket(TCPEndpoint.java:666)
        at sun.rmi.transport.tcp.TCPTransport.listen(TCPTransport.java:335)
        ... 12 more
```

### Start the Kafka console producer from the command line

```
./usr/bin/kafka-console-producer --broker-list localhost:9092 --topic stock-quotes-avro
```

### Publish the poison pill ;)

The console is waiting for input. Now publish the poison pill:

```
💊
```

### Check the consumer application!

The consumer application will try to deserialize the poison pill but fail.
The application by default will try again, again and again to deserialize the record but will never succeed!. 
For every deserialization exception a log line will be written to the log:

```
java.lang.IllegalStateException: This error handler cannot process 'SerializationException's directly; please consider configuring an 'ErrorHandlingDeserializer' in the value and/or key deserializer
        at org.springframework.kafka.listener.SeekUtils.seekOrRecover(SeekUtils.java:145) ~[spring-kafka-2.5.0.RELEASE.jar!/:2.5.0.RELEASE]
        at org.springframework.kafka.listener.SeekToCurrentErrorHandler.handle(SeekToCurrentErrorHandler.java:103) ~[spring-kafka-2.5.0.RELEASE.jar!/:2.5.0.RELEASE]
        at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.handleConsumerException(KafkaMessageListenerContainer.java:1241) ~[spring-kafka-2.5.0.RELEASE.jar!/:2.5.0.RELEASE]
        at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.run(KafkaMessageListenerContainer.java:1002) ~[spring-kafka-2.5.0.RELEASE.jar!/:2.5.0.RELEASE]
        at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) ~[na:na]
        at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) ~[na:na]
        at java.base/java.lang.Thread.run(Thread.java:834) ~[na:na]
Caused by: org.apache.kafka.common.errors.SerializationException: Error deserializing key/value for partition stock-quotes-avro-1 at offset 69. If needed, please seek past the record to continue consumption.
Caused by: org.apache.kafka.common.errors.SerializationException: Unknown magic byte!
```

Make sure to stop the consumer application!

## How to survive a poison pill scenario?

Now it's time protect the consumer application!
Spring Kafka offers excellent support to protect your Kafka application(s) against the poison pill scenario.
Depending on your use case you have different options.  

### Log the poison pill(s) and continue consuming

By configuring Spring Kafka's: `org.springframework.kafka.support.serializer.ErrorHandlingDeserializer` 

Branch with complete example: 

```
git checkout handle-poison-pill-log-and-continue-consuming
```

Configuration of the Consumer Application (`application.yml`) to configure the: ErrorHandlingDeserializer 

```yml
spring:
  kafka:
    consumer:
      # Configures the Spring Kafka ErrorHandlingDeserializer that delegates to the 'real' deserializers
      key-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
    properties:
      # Delegate deserializers
      spring.deserializer.key.delegate.class: org.apache.kafka.common.serialization.StringDeserializer
      spring.deserializer.value.delegate.class: io.confluent.kafka.serializers.KafkaAvroDeserializer
```

### Publish poison pill(s) to a dead letter topic and continue consuming

By configuring Spring Kafka's: `org.springframework.kafka.support.serializer.ErrorHandlingDeserializer` in combination
with both the `org.springframework.kafka.listener.DeadLetterPublishingRecoverer` and `org.springframework.kafka.listener.SeekToCurrentErrorHandler` 

Branch with complete example: 

```
git checkout handle-poison-pill-dead-letter-topic-and-continue-consuming
```

### Kafka Streams and Deserialization exceptions

From the  [Kafka Streams documentation](https://kafka.apache.org/10/documentation/streams/developer-guide/config-streams.html#default-deserialization-exception-handler): 

The default deserialization exception handler allows you to manage record exceptions that fail to deserialize. This can be caused by corrupt data, incorrect serialization logic, or unhandled record types. These exception handlers are available:

* `LogAndContinueExceptionHandler`: This handler logs the deserialization exception and then signals the processing pipeline to continue processing more records. This log-and-skip strategy allows Kafka Streams to make progress instead of failing if there are records that fail to deserialize.

* `LogAndFailExceptionHandler`: This handler logs the deserialization exception and then signals the processing pipeline to stop processing more records.


```
org.apache.kafka.common.errors.SerializationException: Unknown magic byte!

2020-08-15 22:29:43.745 ERROR 37514 --- [-StreamThread-1] o.a.k.s.p.internals.StreamThread         : stream-thread [stock-quote-kafka-streams-stream-StreamThread-1] Encountered the following unexpected Kafka exception during processing, this usually indicate Streams internal errors:

org.apache.kafka.streams.errors.StreamsException: Deserialization exception handler is set to fail upon a deserialization error. If you would rather have the streaming pipeline continue after a deserialization error, please set the default.deserialization.exception.handler appropriately.
	at org.apache.kafka.streams.processor.internals.RecordDeserializer.deserialize(RecordDeserializer.java:80) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.RecordQueue.updateHead(RecordQueue.java:175) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.RecordQueue.addRawRecords(RecordQueue.java:112) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.PartitionGroup.addRawRecords(PartitionGroup.java:162) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.StreamTask.addRecords(StreamTask.java:765) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.StreamThread.addRecordsToTasks(StreamThread.java:943) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.StreamThread.runOnce(StreamThread.java:764) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.StreamThread.runLoop(StreamThread.java:697) ~[kafka-streams-2.5.1.jar:na]
	at org.apache.kafka.streams.processor.internals.StreamThread.run(StreamThread.java:670) ~[kafka-streams-2.5.1.jar:na]
Caused by: org.apache.kafka.common.errors.SerializationException: Unknown magic byte!

2020-08-15 22:29:43.745  INFO 37514 --- [-StreamThread-1] o.a.k.s.p.internals.StreamThread         : stream-thread [stock-quote-kafka-streams-stream-StreamThread-1] State transition from RUNNING to PENDING_SHUTDOWN
2020-08-15 22:29:43.745  INFO 37514 --- [-StreamThread-1] o.a.k.s.p.internals.StreamThread         : stream-thread [stock-quote-kafka-streams-stream-StreamThread-1] Shutting down
```

```yml
spring:
  kafka:
    streams:
      properties:
        default.deserialization.exception.handler: org.apache.kafka.streams.errors.LogAndContinueExceptionHandler
```

```
org.apache.kafka.common.errors.SerializationException: Unknown magic byte!

2020-08-15 22:47:50.775  WARN 39647 --- [-StreamThread-1] o.a.k.s.p.internals.RecordDeserializer   : stream-thread [stock-quote-kafka-streams-stream-StreamThread-1] task [0_0] Skipping record due to deserialization error. topic=[stock-quotes-avro] partition=[0] offset=[771]
```

Spring Kafka RecoveringDeserializationExceptionHandler:

Spring Kafka provides a the org.springframework.kafka.streams.RecoveringDeserializationExceptionHandler (implementation of Kafka Streams DeserializationExceptionHandler) to be able to publish to a Dead letter topic deserialization exception occurs.

The RecoveringDeserializationExceptionHandler is configured with a ConsumerRecordRecoverer implementation. Spring Kafka provides the DeadLetterPublishingRecoverer which sends the failed record to a dead-letter topic.

```yml
spring:
  kafka:
    streams:
      properties:
        default.deserialization.exception.handler: org.springframework.kafka.streams.RecoveringDeserializationExceptionHandler
```