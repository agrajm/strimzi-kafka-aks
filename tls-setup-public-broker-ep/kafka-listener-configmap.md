
# Listener Configurations for Kafka

## External Listener with NO TLS Auth + Plain Listener with No TLS

 
  ```
  # External listener
  ##########
  listener.name.external-9094.ssl.client.auth=required
  listener.name.external-9094.ssl.truststore.location=/tmp/kafka/clients.truststore.p12
  listener.name.external-9094.ssl.truststore.password=${CERTS_STORE_PASSWORD}
  listener.name.external-9094.ssl.truststore.type=PKCS12

  listener.name.external-9094.ssl.keystore.location=/tmp/kafka/cluster.keystore.p12
  listener.name.external-9094.ssl.keystore.password=${CERTS_STORE_PASSWORD}
  listener.name.external-9094.ssl.keystore.type=PKCS12

  ##########
  # Common listener configuration
  ##########
  listeners=REPLICATION-9091://0.0.0.0:9091,PLAIN-9092://0.0.0.0:9092,EXTERNAL-9094://0.0.0.0:9094
  advertised.listeners=REPLICATION-9091://kafka-cluster-kafka-${STRIMZI_BROKER_ID}.kafka-cluster-kafka-brokers.tls-kafka.svc:9091,PLAIN-9092://kafka-cluster-kafka-${STRIMZI_BROKER_ID}.kafka-cluster-kafka-brokers.tls-kafka.svc:9092,EXTERNAL-9094://${STRIMZI_EXTERNAL_9094_ADVERTISED_HOSTNAME}:${STRIMZI_EXTERNAL_9094_ADVERTISED_PORT}
  listener.security.protocol.map=REPLICATION-9091:SSL,PLAIN-9092:PLAINTEXT,EXTERNAL-9094:SSL
  inter.broker.listener.name=REPLICATION-9091
  sasl.enabled.mechanisms=
  ssl.secure.random.implementation=SHA1PRNG
  ssl.endpoint.identification.algorithm=HTTPS
  ```

## External Listener with TLS Auth + Plain Listener with No TLS

