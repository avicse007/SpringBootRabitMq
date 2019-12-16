# SpringBootRabitMq
Have a Producer and a Consumer of Room message that producer get from resturant app and sends to RabitMq queue from where consumer consumers it.

# Purpose of Messaging
Messaging provides a mechanism for loosely coupled integration of application components or software, system, and even multiple systems together. It provides a way to communicate loosely, asynchronously, and progressively. There are many protocols that exist that provide this feature, and AMQP is one of the most popular and robust.

## AMQP
AMQP (Advanced Message Queuing Protocol) is a protocol that RabbitMQ uses for messaging. Although RabbitMQ supports some other protocols, AMQP is most preferable due to compatibility and the large set of features it offers.

## RabbitMQ
RabbitMQ is a powerful, open-source message broker. It is the most popular and most widely deployed message broker in the world as per it’s official website.

## RabbitMQ Installation
Visit RabbitMQ official website where you will find download+install link. Click on this link to go to download and install section. You will get some options for download and install for different OS types. Choose the one suitable for the OS you are using and proceed with downloading and installing.

Note that RabbitMQ also requires Erlang to be installed to make it work. You can find the compatible and recommended version of Erlang for the RabbitMQ version you have installed or selected on this link. Start Installing RabbitMQ and respective erlang one-by-one and choose default options if it prompts you to select.

To verify if RabbitMQ is running or not, you can go to windows services and find the RabbitMQ in the list of services.

## RabbitMQ Management UI
After completion of installation, you can visit the RabbitMQ management UI anytime to see the details about exchange, queue, bindings, and messages by opening the management URL in your browser. Default URL is http://localhost:15672 if you have not changed the port number while installing. Default username is guest and password is guest. For more details visit this link.

Image title

I recommend you play around with various options like sending messages to queue and getting messages from queue in the management UI to get a sense of how it works. Don’t worry if you don’t understand these key terms. In the below sections, I am going to describe these key terms.

## RabbitMQ Architectural Design
Image title
## RabbitMQ Key terms

#### Exchange:
Takes a message and routes it to one or more queues. Routing algorithms decides where to send the message from the exchange. Routing algorithms depends on the exchange type and rules called “bindings.”


Image title

#### Topics: Topics are the subject part of the messages. These are the optional parameters for message exchange.

#### Bindings: "Bindings" is the glue that holds exchanges and queues together. These are the rules for routing algorithms.

#### Queue: Queue is a container for messages. It is only bound by the host’s memory and disk limit. Queues are the final destination for messages before being popped up by subscribers.

#### Producer: Producer is a program that sends message to a queue.

#### Consumer: A consumer is a program which receives messages from the queue.

## RabbitMQ Configuration
RabbitMQ configurations can be fed using rabbitmq.conf file. The default file location depends on the OS. The default location of the config file on windows is %APPDATA%\RabbitMQ\

To override the main RabbitMQ config file location, you can use the RABBITMQ_CONFIG_FILE environment variable. Configuration details can be found here.

## Conclusion
I hope this covers the basic concepts of asynchronous messaging with RabbitMQ and how to get started with it. In my next article, you can find the implementation details including source code on how to integrate a Java, Spring Boot application to RabbitMQ message broker to provide asynchronous messaging features.

# Create Spring Boot Project and Add RabbitMQ Dependencies
The first step required to make use of RabbitMQ in your application is to create a spring boot application and add the required dependencies of RabbitMQ into it. If you already have a Spring Boot application and wanted to just integrate RabbitMQ into it, then you can simply add the dependencies without creating a new project. However, creating a separate project for RabbitMQ instead of integrating into the existing project has one advantage that your message queue related code will be outside your actual application, hence, it can be easily shared or pluggable to any of your other application if required.

You can either create a Spring Boot project from spring initializer and import in your IDE, or you can create directly from Spring Tool Suite IDE (if you are using it). Add RabbitMQ dependency spring-boot-starter-amqpin pom.xml.

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
Add and Set Some Configs. In application.properties
These properties names are self-explanatory. The properties will be used in the application for exchange name, queue name, biding, etc.

# Message Queue specific configs for app1
app1.exchange.name=app1-exchange
app1.queue.name=app1-queue
app1.routing.key=app1-routing-key
# Message Queue specific configs for app2
app2.exchange.name=app2-exchange
app2.queue.name=app2-queue
app2.routing.key=app2-routing-key
#AMQP RabbitMQ configuration 
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
# Additional RabbitMQ properties
spring.rabbitmq.listener.simple.concurrency=4
spring.rabbitmq.listener.simple.max-concurrency=8
spring.rabbitmq.listener.simple.retry.initial-interval=5000
Create a Properties File Reader Class
## After creating the properties file, let's create a property file reader class to read the properties.

package com.dpk.config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
@Configuration
@PropertySource("classpath:application.properties")
public class ApplicationConfigReader {
@Value("${app1.exchange.name}")
private String app1Exchange;
@Value("${app1.queue.name}")
private String app1Queue;
@Value("${app1.routing.key}")
private String app1RoutingKey;
@Value("${app2.exchange.name}")
private String app2Exchange;
@Value("${app2.queue.name}")
private String app2Queue;
@Value("${app2.routing.key}")
private String app2RoutingKey;
// All getters and setters
}
## Create the Beans for Queue, Exchange, Routing Key, and Binding
package com.dpk;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.rabbit.annotation.EnableRabbit;
import org.springframework.amqp.rabbit.annotation.RabbitListenerConfigurer;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.listener.RabbitListenerEndpointRegistrar;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.servlet.support.SpringBootServletInitializer;
import org.springframework.context.annotation.Bean;
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.messaging.handler.annotation.support.DefaultMessageHandlerMethodFactory;
import com.dpk.config.ApplicationConfigReader;
@EnableRabbit
@SpringBootApplication
public class MsgqApplication extends SpringBootServletInitializer implements RabbitListenerConfigurer {
@Autowired
private ApplicationConfigReader applicationConfig;
public ApplicationConfigReader getApplicationConfig() {
return applicationConfig;
}
public void setApplicationConfig(ApplicationConfigReader applicationConfig) {
this.applicationConfig = applicationConfig;
}
public static void main(String[] args) {
SpringApplication.run(MsgqApplication.class, args);
}
protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
return application.sources(MsgqApplication.class);
}
/* This bean is to read the properties file configs */
@Bean
public ApplicationConfigReader applicationConfig() {
return new ApplicationConfigReader();
}
/* Creating a bean for the Message queue Exchange */
@Bean
public TopicExchange getApp1Exchange() {
return new TopicExchange(getApplicationConfig().getApp1Exchange());
}
/* Creating a bean for the Message queue */
@Bean
public Queue getApp1Queue() {
return new Queue(getApplicationConfig().getApp1Queue());
}
/* Binding between Exchange and Queue using routing key */
@Bean
public Binding declareBindingApp1() {
return BindingBuilder.bind(getApp1Queue()).to(getApp1Exchange()).with(getApplicationConfig().getApp1RoutingKey());
}
/* Creating a bean for the Message queue Exchange */
@Bean
public TopicExchange getApp2Exchange() {
return new TopicExchange(getApplicationConfig().getApp2Exchange());
}
/* Creating a bean for the Message queue */
@Bean
public Queue getApp2Queue() {
return new Queue(getApplicationConfig().getApp2Queue());
}
/* Binding between Exchange and Queue using routing key */
@Bean
public Binding declareBindingApp2() {
return BindingBuilder.bind(getApp2Queue()).to(getApp2Exchange()).with(getApplicationConfig().getApp2RoutingKey());
}
/* Bean for rabbitTemplate */
@Bean
public RabbitTemplate rabbitTemplate(final ConnectionFactory connectionFactory) {
final RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
rabbitTemplate.setMessageConverter(producerJackson2MessageConverter());
return rabbitTemplate;
}
@Bean
public Jackson2JsonMessageConverter producerJackson2MessageConverter() {
return new Jackson2JsonMessageConverter();
}
@Bean
public MappingJackson2MessageConverter consumerJackson2MessageConverter() {
return new MappingJackson2MessageConverter();
}
@Bean
public DefaultMessageHandlerMethodFactory messageHandlerMethodFactory() {
DefaultMessageHandlerMethodFactory factory = new DefaultMessageHandlerMethodFactory();
factory.setMessageConverter(consumerJackson2MessageConverter());
return factory;
}
@Override
public void configureRabbitListeners(final RabbitListenerEndpointRegistrar registrar) {
registrar.setMessageHandlerMethodFactory(messageHandlerMethodFactory());
}
}
## Create a Message Sender
MessageSender is pretty simple. It just makes use of convertAndSend() method of RabbitTemplate to send the message to the queue using exchange, routing-key, and data.

package com.dpk;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;
/**
 * Message sender to send message to queue using exchange.
 */
@Component
public class MessageSender {
private static final Logger log = LoggerFactory.getLogger(MessageSender.class);
/**
 * 
 * @param rabbitTemplate
 * @param exchange
 * @param routingKey
 * @param data
 */
public void sendMessage(RabbitTemplate rabbitTemplate, String exchange, String routingKey, Object data) {
log.info("Sending message to the queue using routingKey {}. Message= {}", routingKey, data);
rabbitTemplate.convertAndSend(exchange, routingKey, data);
log.info("The message has been sent to the queue.");
}
}
## Create Message Listeners
Implementing message listener is tricky as it requires handling some of the scenarios like:

How to auto deserialize the message to a POJO
What if listener is making a REST call to some API which is unreachable, or what if an error occurred on the API side while processing the request?
How to make multiple listeners to concurrently pop the message from queue and process
When and how to re-queue the message in the message queue in failure scenarios
Deserializing Message to POJO
Spring provides an annotation @RabbitListener , which can be used to receive messages from the queue. It has a great feature of deserializing the message to a POJO while receiving. The below example illustrates that.

## Error Handling and Message Re-Queuing Feature in Listener
Listener is trying to call an API to process the request that is unreachable
Listener has called the API but error occurred in API while processing the request
In this situation, depending on your business requirement, either you should not re-queue the message, or you should re-queue with max number of trial option to re-try to process it up to a limit.

To not requeue the message in queue, you can throw exception AmqpRejectAndDontRequeueException . For max number of trial handling, you can add an additional parameter in the message to set max number of trial and use it while receiving the message by incrementing it’s value and checking whether total number of trial has not exceeded max limit.

There is an alternative of above approach to add this properties in application.properties and specify max number of attempts:-

spring.rabbitmq.listener.simple.retry.max-attempts=3

## Concurrency Capability
Concurrency feature can be implemented in 2 ways:

Create a thread pool with a specified number of max threads and using ThreadExecutorcall the methods/APIs to process the request.
By using Inbuild concurrency feature. I believe this is the simplest approach to implement concurrency. This requires just making use of 2 properties in application.properties file.
Note: You can set the values of these properties as per your application scalability.

spring.rabbitmq.listener.simple.concurrency=4

spring.rabbitmq.listener.simple.max-concurrency=8

package com.dpk;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.AmqpRejectAndDontRequeueException;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.client.HttpClientErrorException;
import com.dpk.config.ApplicationConfigReader;
import com.dpk.dto.UserDetails;
import com.dpk.util.ApplicationConstant;
/**
 * Message Listener for RabbitMQ
 */
@Service
public class MessageListener {
    private static final Logger log = LoggerFactory.getLogger(MessageListener.class);
    @Autowired
    ApplicationConfigReader applicationConfigReader;
    /**
     * Message listener for app1
     * @param UserDetails a user defined object used for deserialization of message
     */
    @RabbitListener(queues = "${app1.queue.name}")
    public void receiveMessageForApp1(final UserDetails data) {
    log.info("Received message: {} from app1 queue.", data);
    try {
    log.info("Making REST call to the API");
    //TODO: Code to make REST call
        log.info("<< Exiting receiveMessageForApp1() after API call.");
    } catch(HttpClientErrorException  ex) {
    if(ex.getStatusCode() == HttpStatus.NOT_FOUND) {
        log.info("Delay...");
        try {
    Thread.sleep(ApplicationConstant.MESSAGE_RETRY_DELAY);
    } catch (InterruptedException e) { }
    log.info("Throwing exception so that message will be requed in the queue.");
    // Note: Typically Application specific exception should be thrown below
    throw new RuntimeException();
    } else {
    throw new AmqpRejectAndDontRequeueException(ex); 
    }
    } catch(Exception e) {
    log.error("Internal server error occurred in API call. Bypassing message requeue {}", e);
    throw new AmqpRejectAndDontRequeueException(e); 
    }
    }
    /**
     * Message listener for app2
     * 
     */
    @RabbitListener(queues = "${app2.queue.name}")
    public void receiveMessageForApp2(String reqObj) {
    log.info("Received message: {} from app2 queue.", reqObj);
    try {
    log.info("Making REST call to the API");
    //TODO: Code to make REST call
        log.info("<< Exiting receiveMessageCrawlCI() after API call.");
    } catch(HttpClientErrorException  ex) {
    if(ex.getStatusCode() == HttpStatus.NOT_FOUND) {
        log.info("Delay...");
        try {
    Thread.sleep(ApplicationConstant.MESSAGE_RETRY_DELAY);
    } catch (InterruptedException e) { }
    log.info("Throwing exception so that message will be requed in the queue.");
    // Note: Typically Application specific exception can be thrown below
    throw new RuntimeException();
    } else {
    throw new AmqpRejectAndDontRequeueException(ex); 
    }
    } catch(Exception e) {
    log.error("Internal server error occurred in python server. Bypassing message requeue {}", e);
    throw new AmqpRejectAndDontRequeueException(e); 
    }
    }
}
## Create the Service Endpoint
Finally, create the service class that contains the service endpoint, which will be called by the user. The service class calls the MessageSender to put the message into the queue.

package com.dpk;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import com.dpk.config.ApplicationConfigReader;
import com.dpk.dto.UserDetails;
import com.dpk.util.ApplicationConstant;
@RestController
@RequestMapping(path = "/userservice")
public class UserService {
private static final Logger log = LoggerFactory.getLogger(UserService.class);
private final RabbitTemplate rabbitTemplate;
private ApplicationConfigReader applicationConfig;
private MessageSender messageSender;
public ApplicationConfigReader getApplicationConfig() {
return applicationConfig;
}
@Autowired
public void setApplicationConfig(ApplicationConfigReader applicationConfig) {
this.applicationConfig = applicationConfig;
}
@Autowired
public UserService(final RabbitTemplate rabbitTemplate) {
this.rabbitTemplate = rabbitTemplate;
}
public MessageSender getMessageSender() {
return messageSender;
}
@Autowired
public void setMessageSender(MessageSender messageSender) {
this.messageSender = messageSender;
}
@RequestMapping(path = "/add", method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<?> sendMessage(@RequestBody UserDetails user) {
String exchange = getApplicationConfig().getApp1Exchange();
String routingKey = getApplicationConfig().getApp1RoutingKey();
/* Sending to Message Queue */
try {
messageSender.sendMessage(rabbitTemplate, exchange, routingKey, user);
return new ResponseEntity<String>(ApplicationConstant.IN_QUEUE, HttpStatus.OK);
} catch (Exception ex) {
log.error("Exception occurred while sending message to the queue. Exception= {}", ex);
return new ResponseEntity(ApplicationConstant.MESSAGE_QUEUE_SEND_ERROR,
HttpStatus.INTERNAL_SERVER_ERROR);
}
}
}



## Implementing a message consumer
- [Instructor] We're going to start our journey with RabbitMQ by implementing the consumer side first, that way we don't get a lot of messages sitting out in the queue, that when we start up the consumer they immediately respond, we want to do this in an orderly fashion. As such, we're going to implement our consumer first. The one thing I want you to make sure is to make sure that you actually have Rabbit running on your system as you get started. Let's jump to our IDE, and we're going to create a new module. Again, I'm going to use the built in function of IntelliJ, you can definitely do this through start.spring.io, or through your plugins in Eclipse, it's totally up to you. But we're going to use the Spring Initializr once again. We're going to create a project in the group com.frankmoley.boot.landon, and we're going to call this project room-cleaner-consumer. Now, I'm going to remove the roomcleanerconsumer from my package, as I've done in all the other examples. This time, we're going to go down to the IO section, and we're going to select AMQP. We'll hit Next, we'll accept all the defaults, and we'll allow this to finish. Now, we'll let Maven do its work. Once this project is up and running, we'll go ahead and navigate to src/main/resources to the application.properties file. Now, I like to create properties for my exchange in queue, that way I can make sure that I can change them as needed. We're going to create a queue name called room-cleaner, and an exchange name called landon-rooms-exchange. These names are really arbitrary in our demo example, but in the real world, they would match whatever your system is actually leveraging within Rabbit. Once that's done, we're going to create a new class in src/main/java in our root package. That class is going to be called RoomCleanerProcessor. Now, we're going to annotate our class with @Component, so that it will be component scanned by Spring when it starts up. The first thing that I want to do is I want to create a private_ObjectMapper, and you'll notice that this comes from Jackson, so even though we didn't bring it onto the classpath, Rabbit brought it on for us. We're going to call that objectMapper. We'll go ahead and make this final. Now we will go ahead and Autowire a constructor, and accept an objectMapper into it. Then we will call super, and set this.objectMapper = objectMapper. We're going to use this so that we can create a JSON message and put it onto our classpath. Before we get too deep into this class, I want to copy from the clr-app in src/main/java, the Room class. We'll go ahead and bring that over, and then we have an object to actually work with. Back to our processor, we need to do a couple other things. I want to go ahead and create a private static final Logger, and we're going to pull the Logger from slf4j, and we will call it LOGGER, and we will pull it from the LoggerFactory.getLogger, and we're going to pass in the RoomCleanerProcessor.class. This way, we can actually output some messages to the logger when we send and receive messages as we go through this project. Now, I want to create a public void method called receiveMessage, and we're going to pass it a String of roomJson. This is what we're actually going to tell Spring to call when it receives a message. Let's go ahead and do a LOGGER.info Message, and just say Message Received. Now, we're going to do a try block. In our try block, we're going to create a room object from the objectMapper.readValue, and we are going to send it a value of roomJson, and a Room.class. Now we will catch an IOException, and we will simply do a LOGGER.error on the exception, and give it a message of Exception caught. All right, so now that we've received the room object, let's go ahead and send to our LOGGER a info message that says Room ready for cleaning, and we will add to that room.getNumber. That way we can actually see in our log messages that a room was received and that it's ready for cleaning, which matches what we're going to actually do in our system. It's important to note that in the real world, you're probably going to do some work. You might send this room to a scheduling system that will then tie it to a housekeeper so that the room can actually be scheduled to be cleaned in a real hotel. But for the purposes of this course, this is sufficient. Let's jump over to our actual application class, and we're going to add a few things to this. The first thing I want to do is I want to add in a Value, and the value that we're going to pass in is ampq.queue.name, and we will set that to a private String of queueName. Go ahead and import .value from Spring, and we're going to do the same thing for our exchange name, amqp.exchange.name, and we will set that equal to an exchange name object of type string. Now, this is all really optional, this is just how I prefer to keep things straight. You definitely can do this manually as we get going in these bean definitions, and hard code it, but this just seems to be a little bit cleaner in the real world. I wanted to extend that as we went through this course. Now we're going to create a Bean of type Queue, and it's the rabbit.amqp.core queue, and we will simply call this method queue, and we will return a new_Queue that has the queueName, and we're going to send false, because we don't need it to be durable for the purposes of this demo. A durable queue will always be there, that way it doesn't lose messages. Now we're going to create a TopicExchange, and we will simply call this topicExchange, and we will return a new TopicExchange, passing it the exchange name. Now, in order for this to work properly, we need to bind the two, so we're going to create a Bean of type Binding, and we will call this binding, and we're going to pass it a queue and a TopicExchange. That way Spring will wire these all together. Go ahead and import from amqp.core for that binding, and we will return a BindingBuilder.bind, and we are going to bind the queue to the topicExchange, and then, this is the interesting thing, you have to tell it also the queueName that you're going to bind to. Once we have that done, we can now do a Bean of type MessageListenerAdapter, and we will simply call this listenerAdapter. What we are going to do here is we are going to send it a RoomCleanerProcessor, which is the class that we created, and here's the interesting thing, this is actually going to be a delegate, so we're going to return a new MessageListenerAdapter, passing it the processor, which is the object that we just created, and then a method name, which, in this case, is receiveMessage, which matches what we put in our class. Now that that's done, we can actually make our listener. I know that this is a lot of config, but you'll see how easy this all works when it's all said and done. We're going to make a SimpleMessageListenerContainer, called container. To this bean we need to pass a couple objects. The first that we're going to pass is a ConnectionFactory. The ConnectionFactory is provided by Spring Boot in the Spring Boot starter AMQP. We also need to pass it our MessageListenerAdapter that we just created. Now, once that's done, we can then build it. We're going to build a SimpleMessageListenerContainer, we'll call it container, = new SimpleMessageListenerContainer, and on that container, we're going to set the ConnectionFactory, equal to the one provided by Spring Boot. Then on the container, we're going to set the queue names. You'll notice that it's a variable arg. All we need to actually pass in here is one queueName. Now, in our container, we are going to set the MessageListener. This is going to be the listenerAdapter that we just sent in. Now, we can simply return our container. Now we're ready to actually run our application. We'll simply come down here to the main, and we'll run our consumer. Now, as Spring actually executes, you should see it connecting to Rabbit, and at this point, we're done with the listener, and it's ready to receive messages. If we produced a message, it would actually show up, we would get logging here as appropriate. Now, let's go create the producer.






## Implementing a message producer

- [Instructor] So now that we've implemented our consumer, it's time to implement our producer. Now there's a couple caveats here. We need to still make sure that Rabbit is up and running. We also need to make sure that our consumer is running, and finally we need to make sure that our web app is running because we're going to consume the web service to push messages out onto the Rabbit queue. So let's jump into our IDE. The first thing that I want you to do is to open up the room-clr-app and go to the pom.xml file. Now to the pom.xml we're going to add a dependency on spring-boot-starter-amqp, and that's from org.springframework.boot, and we'll allow the dependency resolution to occur. Now we're going to open up source, main, resources and go to our application.properties file, and once again we're going to put in two properties. The first is amqp.queue.name, and we're going to use the same name that we did before, which was room-cleaner, and we're going to add another one called amqp.exchange.name, and we will once again set this equal to landon-rooms-exchange. So we're going to jump to our configuration class in source, main, java, our RoomClrAppApplication. We're going to go ahead and inject those values. So just like we did previously, we will set this equal to $amqp.queue.name, and this will be a private string called queueName, and we'll do the same on the exchange name. And we will set that to a private string called exchangeName. So so far everything should look about the same, and it's going to get even more the same here real quick. We're going to create a bean, and that bean is of type Queue from Rabbit core, and it will be named queue, and we will simply return a new Queue, and we're going to pass it the queueName, and, once again, false because it's not a durable queue. And we're going to set another bean for the TopicExchange called topicExchange, and we will return a new TopicExchange using the exchangeName, and finally we need to do a binding. So we will once again create a public bean of type Binding called binding, and we will pass it the queue and the TopicExchange, and we'll bring in the binding from amqp core, and, just like we did before, we will return a BindingBuilder.bind, the queue to the topicExchange, and we will use with and send the queueName to it. We'll go ahead and import in that bindingBuilder. Now that's all that we need to do to our configuration class because we're not building a listener. So let's go to our primer class, and we need to do a few things here. The first thing that I want to do is I want to inject into this a value, and that value is going to be the name of our queue. So once again we need to bring in the amqp.queue.name, and this is another reason why I like to use properties for all of this because it allows me to inject them wherever I need them without having to do any extra work of hard coding, and then if you have to make a change, you've got to change it in multiple places. I'm also going to go and make a logger for this. So private static final Logger, and we'll bring that in from slf4j from the LoggerFactory.getLogger, and we will send it in the class name, which in this case is RoomCleaningPrimer.class. And we'll use that logger to once again log out messages as things happen. Now let's add a couple more private attributes to our class. So we're going to create a private final attribute of type RabbitTemplate, and we will just call it rabbitTemplate, and we're also going to bring in an application context because I want to make sure that we actually kill this application when we're done. So we're going to bring in a configurable application context, and we'll just call this context. And now I need to add one more element, and that's going to be a ObjectMapper from Jackson, and we will simply call this objectMapper. Now these three elements are going to need to be added to our constructor. So we're going to autowire our constructor, and we will pass in a RabbitTemplate, a configurable application context, and a ObjectMapper, and then we will come down here and set this.rabbitTemplate equal to rabbitTemplate, this.context equal to context, and this.objectMapper equal to objectMapper. So at this point we're ready to actually do some work. So let's scroll down to our run method, and you'll see that we called the API using restTemplate, and ultimately we get out of it an array of rooms at which point previously we simply did a System.out on those. So let's remove this System.out, and we're going to now do a lambda. So we're going to call rooms.forEach. We will get a room and we will pass that to our lambda. So now in our lambda I want to do a few things. The first thing I want to do is I'm going to set an info message that's going to be called sending message that way I can see it in action, and I should get one of these for each room that I see. Now I'm going to take and I'm going to create a jsonString from that object. So it's going to be calling objectMapper.writeValueAsString, and we're going to pass it the room. So now I'm going to get a JSON element back out, but you'll see that this causes an exception. So we'll go ahead and wrap that with a simple try catch block, and if we get an error we'll go ahead and call into our LOGGER.error parsing exception and pass it the exception itself. That way we can see that message written out. So now we have this jsonString, so let's actually go and put it on the message queue. So rabbitTemplate.convertAndSend. We're going to send it the queueName and then the jsonString, and that's all you have to do to send a message to Rabbit. Now the final thing that I want to do to make sure that everything closes is I'm going to come down here and set context.close. So I'm going to close the application context so that our message system actually shuts down when it's done. So let's go ahead and execute the prerequisites that I had said before. So we're going to start up our web application and allow it to start, and while that's starting we're going to start up our consumer, and we're really going to be paying attention to the messages spit out in the logs of this consumer as we get started. And now we're ready to start our CLR app, and what you'll see as this Spring starts, it's going to execute the rest call, and it's going to do its processing and send messages. So our application is run. Let's scroll up here in the log messages here a little bit, and you'll see we have several messages saying sending message, which is what we would expect. So if we jump over to our consumer application and we scroll up, you'll see that each one of our rooms received a message that said that the room is ready for cleaning. We got 01, 02, 03, and so on throughout the system. Each message that we received from the rest call was sent to Rabbit, and then the consumer picked up those messages and executed some work based on what we had coded it to do.
