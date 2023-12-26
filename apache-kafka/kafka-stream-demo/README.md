# What is kafka streams

Kafka streams helps build real time applications on top of data which is present in kafka.

kafka streams reads a continuous stream of data from kafka, does some processing on this stream of data and outputs the result back to kafka.

# Pre-requisites

Please ensure that you have Maven and Java installed in your System

# What kafka streams application are we planning to build?

We will build a system which will work as follows:

- There will be an input kafka topic in which you will get a list of countries (key) with a value associated with them (value).
- There will be a kafka streams application which reads the countires views data and counts the number of views each country is getting.
- Kafka streams will write the total country view data into an output topic
- The output topic at any point will have the total number of views each of our countries have got.

For example if the input is as follows ( country and count are separated by a :)

```
Ukraine:30
Ukraine:80
France:67
South Africa:58
France:94
Austria:19
Turkey:91
Spain:15
```

Then the output will be as follows:

```
Ukraine:110
France:161
```

# Setup

## Download Kafka

Download kafka 3.6.1 from [https://kafka.apache.org/downloads](https://kafka.apache.org/downloads)

```
wget https://downloads.apache.org/kafka/3.6.1/kafka_2.12-3.6.1.tgz
```

Extract kafka

```
tar xzf  kafka_2.12-3.6.1.tgz
```

Open the kafka folder. **All kafka commands should be run in the kafka folder**

```
cd kafka_2.12-3.6.1
```

## Setup zookeeper and Kafka

Start Apache Kafka cluster:

```
docker-compose up
```


## Create the input and output topic for the application

Let us create an input topic from which the kafka streams application will read data.

This is done using the following command:

```
bin/kafka-topics.sh --create --topic input-topic --partitions 1 --replication-factor 3 --bootstrap-server localhost:9092
```

The topic name is `input-topic` and it has 3 replica(s) and 1 partition

Let us also create an output topic, into which the kafka streams application will write the result to.

This is done using the following command:

```
bin/kafka-topics.sh --create --topic output-topic --partitions 1 --replication-factor 3 --bootstrap-server localhost:9092
```

The topic name is `output-topic` and it has 3 replica(s) and 1 partition

We can list out all the topics created using the following command:

```
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

The above command will print the following:

```
input-topic
output-topic
```

# Writing the kafka streams application

Create a new Java Maven project in your IDE. If you want to know more about maven you can read [this beginners guide to maven](https://adityasridhar.com/posts/how-to-get-started-with-maven)

## pom.xml

In the `pom.xml` file, add the following dependencies. This includes kafka streams library and the log4j library

```
<dependencies>
    <!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka-streams -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
        <version>2.8.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-log4j12 -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
        <version>1.7.30</version>
    </dependency>
</dependencies>
```

Below is the complete `pom.xml` file.

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>kafka-streams-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka-streams -->
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-streams</artifactId>
            <version>2.8.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-log4j12 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.30</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                        <configuration>
                            <archive>
                                <manifest>
                                    <mainClass>KafkaStreamsDemo</mainClass>
                                </manifest>
                            </archive>
                            <descriptorRefs>
                                <descriptorRef>jar-with-dependencies</descriptorRef>
                            </descriptorRefs>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

In the above file we have added the maven compiler plugin to support java 1.8

We have also added the maven assembly plugin so that we can build a **standalone executable jar** for the kafka streams application.

## Kafka Streams Java Code

Here is the complete Java code for the kafka streams application

```
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.*;
import org.apache.log4j.BasicConfigurator;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;

public class KafkaStreamsDemo {
    public static void main(String[] args) {
        BasicConfigurator.configure();
        Logger.getRootLogger().setLevel(Level.INFO);

        final Serde<String> stringSerde = Serdes.String();
        final Serde<Long> longSerde = Serdes.Long();

        final StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> views = builder.stream(
                "input-topic",
                Consumed.with(stringSerde, stringSerde)
        );

        KTable<String, Long> totalViews = views
                .mapValues(v -> Long.parseLong(v))
                .groupByKey(Grouped.with(stringSerde, longSerde))
                .reduce(Long::sum);

        totalViews.toStream().to("output-topic", Produced.with(stringSerde, longSerde));

        final Properties props = new Properties();
        props.putIfAbsent(StreamsConfig.APPLICATION_ID_CONFIG, "streams-totalviews");
        props.putIfAbsent(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        final KafkaStreams streams = new KafkaStreams(builder.build(), props);

        final CountDownLatch latch = new CountDownLatch(1);

        try {
            streams.start();
            latch.await();
        } catch (final Throwable e) {
            System.exit(1);
        }

        Runtime.getRuntime().addShutdownHook(new Thread("streams-totalviews") {
            @Override
            public void run() {
                streams.close();
                latch.countDown();
            }
        });

        System.exit(0);
    }
}
```

Let me go over the code step by step.

First we initialise a default logging mechanism using the following lines of code:

```
BasicConfigurator.configure();
Logger.getRootLogger().setLevel(Level.INFO);
```

Next we setup String, Long Serdes to help with serialising and deserialsing the kafka data

```
final Serde<String> stringSerde = Serdes.String();
final Serde<Long> longSerde = Serdes.Long();
```

Next we build the kafka streams topology. This is where we specify the logic:

```
final StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> views = builder.stream(
        "input-topic",
        Consumed.with(stringSerde, stringSerde)
);

KTable<String, Long> totalViews = views
        .mapValues(v -> Long.parseLong(v))
        .groupByKey(Grouped.with(stringSerde, longSerde))
        .reduce(Long::sum);

totalViews.toStream().to("output-topic", Produced.with(stringSerde, longSerde));
```

First we create a **StreamsBuilder** object

Then we are creating a **KStream** object called **views** to read data from the kafka topic `input-topic`. In this topic both the key and value are strings. Hence we use string serde for both the key and the value

Next we are created a **KTable** object with key as string and value as long. This is the object where we will store our computation result.

We are using java lambda expressions to achieve the following

- Convert the value to Long, since the value here is the country view count
- Group all the same countries together.
- For the Grouped countries, calculate the sum.

Finally the total countries count is written to a kafka topic called `output-topic`. In this topic, the key is a string and the value is a Long.

Next we create a KafkaStreams object using the following code:

```
final Properties props = new Properties();
props.putIfAbsent(StreamsConfig.APPLICATION_ID_CONFIG, "streams-totalviews");
props.putIfAbsent(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

final KafkaStreams streams = new KafkaStreams(builder.build(), props);
```

In the above code we are giving the application an id of `streams-totalviews`. This will help us in identifying the application.

We are also giving the kafka bootstrap server details as `localhost:9092` since this is where our kafka is running.

Then we create a **KafkaStreams** object called **streams**

Until now we have just created the kafka streams flow, but we have not yet started it.

The Next few lines of code, starts the kafka streams application

```
final CountDownLatch latch = new CountDownLatch(1);

try {
    streams.start();
    latch.await();
} catch (final Throwable e) {
    System.exit(1);
}

Runtime.getRuntime().addShutdownHook(new Thread("streams-totalviews") {
    @Override
    public void run() {
        streams.close();
        latch.countDown();
    }
});

System.exit(0);
```

We start the flow using `streams.start()`

We also created a **Shutdown Hook** in the above lines of code. This ensures that if the application is abnormally shutdown, then the kafka streams application will close properly using the `streams.close()` function.

# Running the Kafka streams application

Build the kafka streams application using the following command:

```
mvn clean package
```

This will create a file called `kafka-streams-demo-1.0-SNAPSHOT-jar-with-dependencies.jar` in the **target** folder

Run the application in a terminal using the following command

```
java -jar target/kafka-streams-demo-1.0-SNAPSHOT-jar-with-dependencies.jar
```

Ensure you give the full path to your jar file

In a second terminal, start a kafka producer using the following command

```
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 \
    --topic input-topic \
    --property key.separator=":" \
    --property parse.key=true < ../data/country-list-sample.txt
```

We will be producing into the kafka topic `input-topic`. Also we will separate the countries from the view count by a `:`

In a third terminal, start a kafka consumer using the following command

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic output-topic \
    --property print.key=true \
    --property key.separator="-" \
    --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

We will be consuming from the topic `output-topic`. In the output, the countries will be separated from the total count with a `-`. Also in the output, the key is a string and the value is a Long.

In the Kafak-UI application, you should see the values totaled for each country


```
Ukraine:30
Ukraine:80
France:67
South Africa:58
France:94
Austria:19
```


We can see that the kafka streams application is aggregating all the counts based on the country.
