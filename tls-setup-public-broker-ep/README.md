# TLS Setup for Kafka with Public Kafka Broker Endpoint

The aim is to extend the first TLS setup with a public Broker Endpoint so that Kafka is accessible outside AKS Cluster

Also what's changed, we are using 0.19.0 version for Strimzi while the original TLS setup used 0.15.0 version - so there can be some unknowns or failures. Let's see

# Start by Deploying Strimzi Cluster Operator

```
kubectl create ns tls-kafka

curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.19.0/strimzi-cluster-operator-0.19.0.yaml \
  | sed 's/namespace: .*/namespace: tls-kafka/' \
  | kubectl apply -f - -n tls-kafka
```

It creates a bunch of 
* ServiceAccount, ClusterRoles & ClusterRoleBindings for Operator(s) to work properly
* CRDs for various Kafka constructs like Kafka Broker, Topics, Users, Kafka Connect, MirrorMaker etc
* Deployments for Strimzi Cluster Operator 


# Deploy Custom Resource for Kafka 

Add a listener for external: type: loadBalancer -- no TLS auth specified for clients - that will be done as next step

```
kubectl apply -f kafka-cluster.yaml
```

# Get Public Endpoint - Azure Load Balancer 

```
export AKS_RESOURCE_GROUP=kafkaRG
export AKS_CLUSTER_NAME=aksCluster
export AKS_LOCATION=australiaeast
az network lb list -g MC_${AKS_RESOURCE_GROUP}_${AKS_CLUSTER_NAME}_${AKS_LOCATION}
```

```
export CLUSTER_NAME=my-kafka-cluster
kubectl get configmap/${CLUSTER_NAME}-kafka-config -o yaml
```
External Bootstrap -- $KAFKA_CLUSTER_NAME-kafka-external-bootstrap 

```
kubectl get svc kafka-cluster-kafka-external-bootstrap -n tls-kafka -o json | jq -r '.status.loadBalancer.ingress[0].ip'
```


# Producer & Consumer Pods ( Connect from Inside )

Since we used both `plain` and `external` listeners, we are able to connect from inside the cluster (using the `plain` listener) - there is no TLS currently enabled on Plain listener so you can connect without using cluster ca certs.

```
// Producer 
export KAFKA_CLUSTER_NAME=kafka-cluster
kubectl run kafka-producer -ti --image=strimzi/kafka:latest-kafka-2.4.0 --namespace=tls-kafka --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list $KAFKA_CLUSTER_NAME-kafka-bootstrap:9092 --topic my-topic

// Consumer
kubectl run kafka-consumer -ti --image=strimzi/kafka:latest-kafka-2.4.0 --namespace=tls-kafka --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server $KAFKA_CLUSTER_NAME-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

# Importing CA Cert in TrustSTore 

To use Kafka CLI or local Java Apps to connect to Kafka Cluster on kubernetes - configure the JDK TrustStore to import the Cluster CA Cert & Password 

```
export CLUSTER_NAME=kafka-cluster
kubectl get secret $CLUSTER_NAME-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca.crt
kubectl get secret $CLUSTER_NAME-cluster-ca-cert -o jsonpath='{.data.ca\.password}' | base64 --decode > ca.password
```

```
export CERT_FILE_PATH=ca.crt
export CERT_PASSWORD_FILE_PATH=ca.password
export KEYSTORE_LOCATION=$JAVA_HOME/lib/security/cacerts
export PASSWORD=`cat $CERT_PASSWORD_FILE_PATH`
export CA_CERT_ALIAS=strimzi-kafka-cert
# you will prompted for the truststore password. for JDK truststore, the default 
# password is "changeit"
# Type yes in response to the 'Trust this certificate? [no]:'
sudo keytool -importcert -alias $CA_CERT_ALIAS -file $CERT_FILE_PATH -keystore $KEYSTORE_LOCATION -keypass $PASSWORD
sudo keytool -list -alias $CA_CERT_ALIAS -keystore $KEYSTORE_LOCATION
```

After importing this certificate, create this file `client-ssl.properties`

## Client SSL properties file 

```
export KAFKA_EXTERNAL_BROKER_IP=$(kubectl get svc kafka-cluster-kafka-external-bootstrap -n tls-kafka -o json | jq -r '.status.loadBalancer.ingress[0].ip')
```

Create a new file with following info
```
bootstrap.servers=$KAFKA_EXTERNAL_BROKER_IP:9094  # Replace with actual value
security.protocol=SSL
ssl.truststore.location=$JAVA_HOME/jre/lib/security/cacerts   # Replace with actual value
ssl.truststore.password=changeit # Whatever is your password - this is default
```

# Producers & Consumers (connect from Outside)

```
export KAFKA_HOME=/mnt/c/Users/agmangal/softwares/linux/kafka_2.12-2.5.0
export TOPIC_NAME=test-strimzi-topic
```

## on a terminal, start producer and send a few messages
```
$KAFKA_HOME/bin/kafka-console-producer.sh --broker-list $KAFKA_EXTERNAL_BROKER_IP:9094 --topic $TOPIC_NAME --producer.config client-ssl.properties
```
# on another terminal, start consumer
```
$KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server $KAFKA_EXTERNAL_BROKER_IP:9094 --topic $TOPIC_NAME --consumer.config client-ssl.properties --from-beginning
```

# TLS Authentication

Now that we can connect to Kafka running in Kubernetes from outside clients with TLS encryption -- we just have imported the Cluster CA certs in our local truststore, the next step is to allow TLS authentication

# Import the Kafka User Private Key & Certificate into KeyStore for Kafka Client

```
export KAFKA_USER_NAME=my-user
kubectl get secret $KAFKA_USER_NAME -o jsonpath='{.data.user\.crt}' | base64 --decode > user.crt
kubectl get secret $KAFKA_USER_NAME -o jsonpath='{.data.user\.key}' | base64 --decode > user.key
kubectl get secret $KAFKA_USER_NAME -o jsonpath='{.data.user\.p12}' | base64 --decode > user.p12
kubectl get secret $KAFKA_USER_NAME -o jsonpath='{.data.user\.password}' | base64 --decode > user.password
```

Create another keystore

```
export USER_P12_FILE_PATH=user.p12
export USER_KEY_PASSWORD_FILE_PATH=user.password
export KEYSTORE_NAME=kafka-auth-keystore.jks
export KEYSTORE_PASSWORD=foobar
export PASSWORD=`cat $USER_KEY_PASSWORD_FILE_PATH`
sudo keytool -importkeystore -deststorepass $KEYSTORE_PASSWORD -destkeystore $KEYSTORE_NAME -srckeystore $USER_P12_FILE_PATH -srcstorepass $PASSWORD -srcstoretype PKCS12
sudo keytool -list -alias $KAFKA_USER_NAME -keystore $KEYSTORE_NAME
```

# Import Cluster CA Cert into TrustStore for Kafka Client

```
export CLUSTER_NAME=my-kafka-cluster
kubectl get secret $CLUSTER_NAME-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca.crt
kubectl get secret $CLUSTER_NAME-cluster-ca-cert -o jsonpath='{.data.ca\.password}' | base64 --decode > ca.password
```

Import

```
export CERT_FILE_PATH=ca.crt
export CERT_PASSWORD_FILE_PATH=ca.password
export KEYSTORE_LOCATION=$JAVA_HOME/jre/lib/security/cacerts
export PASSWORD=`cat $CERT_PASSWORD_FILE_PATH`
sudo keytool -importcert -alias strimzi-kafka-cert -file $CERT_FILE_PATH -keystore $KEYSTORE_LOCATION -keypass $PASSWORD
```

# client-ssl.properties

```
bootstrap.servers=[LOADBALANCER_PUBLIC_IP]:9094
security.protocol=SSL
ssl.truststore.location=[TRUSTSTORE_LOCATION]
ssl.truststore.password=changeit
ssl.keystore.location=kafka-auth-keystore.jks
ssl.keystore.password=foobar
ssl.key.password=[contents of user.password file]
```