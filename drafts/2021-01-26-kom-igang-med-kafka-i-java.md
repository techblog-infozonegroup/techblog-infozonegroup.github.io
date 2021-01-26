---
title: "Kom igång med Kafka i Java"
tagline: "Implementera Kafka consumers och producers i Java med Spring Apache Kafka Streams"
date: 2021-01-26
author: Anton Liljeberg, systemutvecklare
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/10/close-up-of-hands-contemporary-website-developer-man-typing-and-code-picture-id1167467556.jpg
categories:
  - blog
tags:
  - systemutveckling
---

# Inledning
Det finns många olika sätt att implementera Kafka, I det här fallet kommer vi att kika närmare på [Spring for Apache Kafka](https://spring.io/projects/spring-kafka). Om du inte är bekant med [Spring](https://spring.io) sedan tidigare så är det kort beskrivet ett ramverk för Java som gör många saker lättare att implementera med hjälp utav annoteringar. Till exempel en rest endpoint, kafka consumers/producers och dependency injection.

# Kafka i docker
För att kunna komma igång snabbt med en miljö där vi kan testa att skicka och ta emot meddelanden med Kafka så använder vi en docker image från landoop, gjord för just detta endamål. Förutsatt att du har docker installerat så kan du köra detta kommand:
`docker run --rm -p 3030:3030 -p 9092:9092 -p 2182:2181 -p 8081:8081 -p 8082:8082 -p 8083:8083 -e BROKER_PORT=9092 -e ADV_HOST=127.0.0.1 landoop/fast-data-dev`

Det finns ett grafiskt gränssnitt på [localhost:3030](http://localhost:3030) för att se topics samt meddelanden m.m. Vår docker image kommer automatiskt att skapa några topics och skicka meddelanden på dessa. Det kan vara värt att tänka på att vissa topics använder Avro vilket vi inte kommer att gå igenom i detta inlägg.

# Spring Initializr
För att skapa vårat project är det lättast att komma igång med [Spring Initializr](https://start.spring.io). Jag kommer att använda mig utav Gradle i detta exempel. Längst upp till höger kan du lägga till dependencies, vi kommer att använda oss av `Spring for Apache Kafka Streams [Messaging]` samt `Lombok [Developer Tools]`. När du är klar med konfigurationen så kan du trycka på `GENERATE` längst ned för att ladda ner projektet.

# Öppna Projektet
Det finns många olika java-utvecklingsmiljöer att välja på, i detta exempel kommer vi att använda oss av [IntelliJ](https://www.jetbrains.com/idea/download). Packa upp projektet som vi genererade i Spring Initializer på valfri plats. Öppna sedan projektet i IntelliJ genom att klicka på `Open` och välj sedan mappen som du packade upp.

# Dependencies i Gradle
I projektet kommer du att hitta en fil som heter build.gradle. Här kan vi bland annat lägga till paketreferenser. Vi kommer att använda oss utan `jackson-core` samt `jackson-databind`. Vi kan lägga till en referens till dessa genom att lägga till
``` gradle
implementation 'com.fasterxml.jackson.core:jackson-core:2.12.1'
implementation 'com.fasterxml.jackson.core:jackson-databind:2.12.1'
```
i dependencies listan.

# Konfigurera vår Kafka Consumer
En consumer är precis vad det låter som, det är den kod som konsumerar eller tar emot meddelanden från Kafka. För att denna ska kunna fungera behöver vi ange en address till vår kafka server och ett grupp-id. Om två consumers har samma grupp-id så kommer meddelanden att spridas ut mellan dessa två, de kommer med andra ord inte ta del av alla meddelandena individuellt.

Varje meddelande består av ett key-value par med okänd datatyp. För att kunna tyda dessa behöver vi lägga till en key deserializer samt en value deserializer. 
``` java
package com.example.demo;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.config.KafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;

import java.util.HashMap;
import java.util.Map;

@Configuration
class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return props;
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }

    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```
Om du inte tidigare är bekant med Spring så kan denna klass se förvirrande ut. `@Bean` markerar att metoden returnerar implementationen av en klass som sedan instansieras som en singleton-instans och hanteras av en Spring IoC container. `"@Configuration` markerar att det finns `@Bean`s att läsa in i denna klass. `@Value("")` sätter värdet på fältet till det värdet vi har satt i attributen. I detta fall hämtar vi vår data från vår `resources/application.properties` fil.

Om man konverterade denna klass till C# så skulle den kunna se ut som något som liknar detta:
``` csharp
using org.apache.kafka.clients.consumer.ConsumerConfig;
using org.apache.kafka.common.serialization.StringDeserializer;
using org.springframework.beans.factory.annotation.Value;
using org.springframework.context.annotation.Bean;
using org.springframework.context.annotation.Configuration;
using org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
using org.springframework.kafka.config.KafkaListenerContainerFactory;
using org.springframework.kafka.core.ConsumerFactory;
using org.springframework.kafka.core.DefaultKafkaConsumerFactory;
using org.springframework.kafka.listener.ConcurrentMessageListenerContainer;

using java.util.HashMap;
using java.util.Map;

namespace com.example.demo 
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public Dictionary<string, object> ConsumerConfigs() {
            var props = new Dictionary<string, object>();
            props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-consumer-group");
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            return props;
        }

        public void ConfigureServices(IServiceCollection services)
        {
            var bootstrapServers = Configuration["spring.kafka.bootstrap-servers"];

            services
                .AddSingleton<ConsumerFactory<String, String>>(s => new DefaultKafkaConsumerFactory(ConsumerConfigs()))
                .AddSingleton(s => {
                    var factory = new ConcurrentKafkaListenerContainerFactory<>();
                    factory.setConsumerFactory(s.GetRequiredService<ConsumerFactory<String, String>>());
                    return factory;
                });
        }
    }
}
```

# Konfigurera vår Kafka Producer
Vår producer är den kod som producerar meddelanden. På samma sätt som consumern behöver vi konfigurera denna med några basvärden. Producern behöver inte ett group-id till skillnad från consumern då en producer endast skickar meddelanden.
``` java
package com.example.demo;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;

import java.util.HashMap;
import java.util.Map;

@Configuration
class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return props;
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

# application.settings
application.settings fyller samma funktion som appsettings.json i .NET. Skillnaden är att vi använder key-value pairs i stället för JSON för att definera vår data.
Vi lägger till denna rad i vår application.settings så att vår producer samt consumer kan hämta värdet till vår bootstrapServers variabel.
`spring.kafka.bootstrap-servers=127.0.0.1:9092`

# Konsumera och producera meddelanden
``` java
package com.example.demo;

import lombok.AllArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;

import java.sql.Timestamp;

@EnableKafka
@AllArgsConstructor
@Configuration
public class KafkaDemo {

    private KafkaTemplate template;

    @KafkaListener(topics = "logs_broker")
    void listener(String data) {
        System.out.println("Data> " + data);

        var time = new Timestamp(System.currentTimeMillis()).toString();
        template.send("test", "Log event received " + time);
    }
}
```
I vår `@KafkaListener` måste vi minst specificera vilken/vilka topic(s) vi vill lyssna på. Vi kan även överskrida vilket group-id vi vill använda här. I vår listener ovan så lyssnar vi på kafka-topic:en `logs_broker`. När vi tar emot ett meddelande så printar vi ut meddelandet i konsollen. Sedan använder vi oss av en KafkaTemplate för att skicka vilken tid vi tog emot meddelandet på en ny topic vid namn `test`. `@AllArgsConstructor` är en annotering från lombok och bakom kulissen så skapar den en konstruktor som använder dependency injection för att tilldela en instans till alla våra fält (`template`). Det finns fler sätt att skicka och ta emot data från Kafka i Java men detta var det som jag tyckte var lättast.