---
layout: default
title: Kafka with Spring Cloud Stream
sitemap:
priority: 0.5
lastmod: 2017-08-29T17:37:00-00:00
---

__Tip submitted by [@eosimosu](https://github.com/eosimosu)__


### Prerequisite

Generate a new app and make sure to select `Asynchronous messages using Apache Kafka` when prompted for technologies you would like to use. A docker compose configuration file is generated and you can start Kafka with the command:

`docker-compose -f src/main/docker/kafka.yml up -d`


### Model

Create a simple model to represent the messages we would be sending through a Kafka topic.

```
public class Greeting {
    private String message;

    public Greeting() {
    }

    public String getMessage() {
        return message;
    }

    public Greeting setMessage(String message) {
        this.message = message;
        return this;
    }
}

```

### Message Channels
Spring Cloud Stream introduced an abstraction layer called `message channels`. Producers sends messages to output channels, and consumers subscribes to input channels for messages.  This gives you the flexibility to work with different messaging systems (called binders) without writing a lot of platform-specific code.

Let's create our output and input channels.

##### Output channel
```
public interface ProducerChannel {

    String CHANNEL = "messageChannel";

    @Output
    MessageChannel messageChannel();
}
```

##### Input channel
```
public interface ConsumerChannel {

    String CHANNEL = "subscribableChannel";

    @Input
    SubscribableChannel subscribableChannel();
}
```


### Configuration 

We need to tell Spring Cloud Stream about our channels in the configuration class generated by Jhipster.
```
@EnableBinding(value = {Source.class, ProducerChannel.class, ConsumerChannel.class})
public class MessagingConfiguration {

}
```

We also need to configure our application to talk to Kafka. 

```
spring:
    cloud:
      stream:
        bindings:
            messageChannel:
                destination: greetings
                content-type: application/json
            subscribableChannel:
                destination: greetings

```

This corresponds to:

`spring.cloud.stream.bindings.<channelName>.destination.<topic>`


### Producer and Consumer

##### Producer Resource
Let's create a simple REST endpoint that we can invoke so send messages to the Kafka topic, `greetings`.

```

@RestController
@RequestMapping("/api")
public class ProducerResource{

    private MessageChannel channel;

    public ProducerResource(ProducerChannel channel) {
        this.channel = channel.messageChannel();
    }

    @GetMapping("/greetings/{count}")
    @Timed
    public void produce(@PathVariable int count) {
        while(count > 0) {
            channel.send(MessageBuilder.withPayload(new Greeting().setMessage("Hello world!: " + count)).build());
            count--;
        }
    }

}
```

##### Consumer Service
We can consume the messages using `StreamListener` for message mapping and automatic type conversion.
```
@Service
public class ConsumerService {

    private final Logger log = LoggerFactory.getLogger(ConsumerService.class);


    @StreamListener(ConsumerChannel.CHANNEL)
    public void consume(Greeting greeting) {
        log.info("Received message: {}.", greeting.getMessage());
    }
}

```

### Running the app

Allow access to the endpoint in `SecurityConfiguration.java`.

`.antMatchers("/api/greetings/**").permitAll()`

If you invoke the endpoint `http://localhost:8080/api/greetings/5`, you should see the messages logged to the console.

You can find the full source code on [GitHub][6].


[6]: https://github.com/eosimosu/jhipster-kafka
