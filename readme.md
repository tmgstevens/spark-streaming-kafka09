# Spark Streaming with Cloudera Kafka and Spark 2 GA

This repo provides an implementation of Spark 2's Direct Kafka-SparkStreaming. It has been created by forking the implementation at https://github.com/apache/spark/tree/master/external/kafka-0-10/src/main/scala/org/apache/spark/streaming/kafka010 and reworking to build against version 0.9 (CDH2.0) of the Kafka client API.

## Compiling and Deploying

Firstly clone this repo, and from there this project can be built using
```
mvn clean install
```

## Useage

Once compiled, the JAR can be used with Cloudera's Spark 2 GA parcel, which includes the Kafka 0.9/CDK2.0 JARs. It uses a different namespace so-as not to conflict with the other Spark Streaming JARs that we ship.

```
spark2-shell --jars ../spark-streaming-kafka-0-9_2.11-2.0.0.cloudera1.jar --files kafka.keytab#kafka.keytab,kafka-jaas-spark.conf#kafka-jaas-spark.conf --conf "spark.executor.extraJavaOptions=-Djava.security.auth.login.config=./kafka-jaas-spark.conf" --driver-java-options "-Djava.security.auth.login.config=./kafka-jaas-spark.conf"
To adjust logging level use sc.setLogLevel(newLevel).
17/01/12 14:50:35 WARN spark.SparkContext: Use an existing SparkContext, some configuration may not take effect.
Spark context Web UI available at http://10.0.0.251:4040
Spark context available as 'sc' (master = yarn, app id = application_1484225879256_0007).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.0.0.cloudera.beta2
      /_/

Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_67)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka09._
import org.apache.spark.streaming.kafka09.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka09.ConsumerStrategies.Subscribe

val ssc = new StreamingContext(sc, Seconds(5))


val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "ip-10-0-0-251:9093",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean),
  "security.protocol" -> "SASL_SSL",
  "sasl.kerberos.service.name" -> "kafka"	)

val topics = Set("testtopic")
val stream = KafkaUtils.createDirectStream[String, String](
  ssc,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

val lines = stream.map(_.value)
val words = lines.flatMap(_.split(" "))
val wordCounts = words.map(x => (x, 1L)).reduceByKey(_ + _)
wordCounts.print()

ssc.start
```

## Linking
For Scala/Java applications using SBT/Maven project definitions, link your streaming application with the following artifact (which will resolve from your local repo, assuming you built the code as per above).
```
groupId = org.apache.spark
artifactId = spark-streaming-kafka-0-9_2.11
version = 2.0.0.cloudera1
```
## Running using spark2-submit
Once you have a compiled scala program, you can run it as follows:
```
spark2-submit --class com.cloudera.tristan.WordCount.WordCount --deploy-mode client --jars ../spark-streaming-kafka-0-9_2.11-2.0.0.cloudera1.jar --files kafka.keytab#kafka.keytab,kafka-jaas-spark.conf#kafka-jaas-spark.conf --conf "spark.executor.extraJavaOptions=-Djava.security.auth.login.config=./kafka-jaas-spark.conf" --driver-java-options "-Djava.security.auth.login.config=./kafka-jaas-spark.conf" WordCount-0.0.1-SNAPSHOT.jar
```

## Creating a Direct Stream
Note that the namespace for the import includes the version, `org.apache.spark.streaming.kafka09`

### Scala
```
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe

val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "localhost:9092,anotherhost:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean)
)

val topics = Array("topicA", "topicB")
val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

stream.map(record => (record.key, record.value))
```
## Security
Kafka 0.9 / CDK 2.0 support Kerberos authentication and TLS/SSL encryption, however there are some limitations.
Because upstream Kafka does not yet support delegation tokens, in order to use Kerberos, you are required to distribute a keytab into your spark application (using `--files kafka.keytab#kafka.keytab`). This does create a security vulnerability (as the keytab will be localized onto each worker node during execution) and therefore careful consideration should be given before using. As streaming executors are started, each one will need to kinit against the KDC - which will generate load on the KDC (and with enough partitions could result in the KDC suspecting a DDOS attack). That said, this only occurs at executor startup time as once a service ticket as been acquired it does not need to be re-acquired for each streaming window.

### Kerberos

In order to use Kerberos, the following settings are required:
1. Create a keytab for use by the consumers.
1. Create a jaas config file (see below).
1. Set the executor and driver jaas settings to use the jaas config file.
1. Set the appropriate parameters in your Kafka parameters.

Example jaas.conf:
```
KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="./kafka.keytab"
  storeKey=true
  useTicketCache=false
  serviceName="kafka"
  principal="kafkauser@IPA.TRISTAN.CLOUDERA.COM";
};
```

Example Kafka Params:
```
val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "ip-10-0-0-251:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean),
  "security.protocol" -> "SASL_PLAINTEXT",
  "sasl.kerberos.service.name" -> "kafka"	)
```

Example spark-submit:
```
spark2-submit --class com.cloudera.tristan.WordCount.WordCount --deploy-mode client --jars ../spark-streaming-kafka-0-9_2.11-2.0.0.cloudera1.jar --files kafka.keytab#kafka.keytab,kafka-jaas-spark.conf#kafka-jaas-spark.conf --conf "spark.executor.extraJavaOptions=-Djava.security.auth.login.config=./kafka-jaas-spark.conf" --driver-java-options "-Djava.security.auth.login.config=./kafka-jaas-spark.conf" WordCount-0.0.1-SNAPSHOT.jar
```

### TLS/SSL
TLS/SSL is slightly simpler, so long as your root CA is installed across the cluster's trust-stores (e.g. using jssecacerts).
If it isn't then it is just a matter of setting `"security.protocol" -> "SASL_SSL"` in your Kafka Params (assuming you are also using Kerberos), or `"security.protocol" -> "SSL"` if you aren't.

If your truststore is not set up then you will need to include the following parameters:
```
"ssl.truststore.location" -> "/some-directory/kafka.client.truststore.jks",
"ssl.truststore.password" -> "test1234",
```
and ensure that the keystore and truststore are passed to the spark executors.

If client-authentication is required, then you will also need:
```
"ssl.keystore.location" -> "/some-directory/kafka.client.keystore.jks",
"ssl.keystore.password" -> "test1234",
"ssl.key.password" -> "test1234"
```

## Further information
For further details on how to use this consumer, see the Spark 2 GA documentation at https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html, however remember to replace the namespace with `org.apache.spark.streaming.kafka09`.
