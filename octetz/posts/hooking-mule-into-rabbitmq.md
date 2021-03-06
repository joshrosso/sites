Hooking Mule into RabbitMQ 
==========================

[ [WTFP License](http://www.wtfpl.net) | [Improve this reference](http://github.com/joshrosso/octetz.com) | [View all references](../index.html) ]

---

<iframe width="640" height="360" src="https://www.youtube.com/embed/G1WQA3XxGRY" frameborder="0" allowfullscreen></iframe>

[RabbitMQ](http://www.rabbitmq.com/) is a messaging solution with a
focus on persistence and routing. It's implemented in Erlang and is a
quite popular message broker for large scale applications. RabbitMQ is
implemented to leverage the AMQP protocol. For such a capable broker,
it's remarkably easy to get bootstrapped and hooked into Mule. Getting
bootstrapped and connected will be our focus in this post, so let's get
started.


Setting up RabbitMQ
-------------------


You'll need to download RabbitMQ. After downloading, install RabbitMQ
based on [the installation guide for your
platform](https://www.rabbitmq.com/download.html). Let's begin by
starting RabbitMQ.

```
rabbitmq-server
```

Your shell will then display RabbitMQ. Simply enter `ctrl + c` when you
wish to stop the broker.

```
            RabbitMQ 3.6.0. Copyright (C) 2007-2015 Pivotal Software, Inc.
##  ##      Licensed under the MPL.  See http://www.rabbitmq.com/
##  ##
##########  Logs: /usr/local/var/log/rabbitmq/rabbit@localhost.log
######  ##        /usr/local/var/log/rabbitmq/rabbit@localhost-sasl.log
##########
            Starting broker... completed with 10 plugins.
```

RabbitMQ features a graphical plugin named rabbitmq\_management. This
will allow us to visually inspect our queues. Enable the plugin with the
following command.

```
rabbitmq-plugins enable rabbitmq_management
```

Login to the management console at <http://localhost:15672>. The default
username and password is `guest`.

![rabbitmq management
home](../imgs/rabbitmq-management-home.png){width="600"}

Setting up Mule
---------------

If you already have a Mule 3.7.0+ standalone instance, skip this
section.

First, [download a standalone Mule runtime from
nexus](https://repository.mulesoft.org/nexus/content/repositories/public/org/mule/distributions/mule-standalone/).
I'll be using the 3.7.0.

```
curl -O https://repository.mulesoft.org/nexus/content/repositories/public/org/mule/distributions/mule-standalone/3.7.0/mule-standalone-3.7.0.tar.gz
```

Unpackage the download; delete the package

```
tar zxvf mule-standalone-3.7.0.tar.gz && \
  rm -v mule-standalone-3.7.0.tar.gz
```

From this point on, I assume you've set `$MULE_HOME` in your environment to the location of `mule-standalone-3.7.0`.           

Creating the Mule project
-------------------------

Using [the mule-app maven
archetype](https://github.com/mulesoft/mule-esb-maven-tools) generate a
new project. Modify the package/artifactId names as you desire.

```
mvn archetype:generate \
  -DarchetypeGroupId=org.mule.tools.maven \
  -DarchetypeArtifactId=maven-archetype-mule-app \
  -DarchetypeVersion=1.2-SNAPSHOT \
  -DgroupId=com.joshrosso -DartifactId=user-registration \
  -Dversion=1.0-SNAPSHOT -Dpackage=com.joshrosso \
  -DmuleVersion=3.7.0 -Dtransports=vm -Dmodules=http
```

I'll now refer to this project location as `$PROJECT_HOME`.

Remove the test case from the project. We won't be running any test
cases in this example.

```
rm -rv $PROJECT_HOME/src/test/java/*
```

Clean `$PROJECT_HOME/src/main/app/mule_config.xml` by removing all
elements inside of the `main` flow and removing all unused namespaces.

```
<?xml version="1.0" encoding="UTF-8"?>

<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
        http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">

  <flow name="main">
  </flow>

</mule>
```

Setup a service
---------------

Let's establish a producer flow exposed externally via http. Then we'll
create a consumer flow that logs the payload. For now, we'll call the
consumer flow via a `flow-ref` to verify the request is received.

```
<http:listener-config name="httpListener"
                      host="localhost"
                      port="8081"/>

<flow name="producer">
  <http:listener config-ref="httpListener" path="/*"/>
  <flow-ref name="consumer"/>
</flow>

<flow name="consumer">
  <logger level="INFO" message="#[message.payloadAs(java.lang.String)]"/>
  <set-payload value="User created"/>
</flow>
```

Start your must standalone instance if you haven't already, and deploy
the application using maven's install command.

```
mvn install
```

We'll now send a request in to verify the data is flowing before hooking
into rabbit. After a call, we should see "User created" as our response
and the payload we POST in the logs.

```
curl -i -X POST \
  -H 'content-type: application/json' \
  -d '{"name" : "bob"}' \
  http://localhost:8081/users
```

Connecting to RabbitMQ 
----------------------

We'll now use the [amqp
transport](https://github.com/mulesoft/mule-transport-amqp) to
communicate with our RabbitMQ broker. First, we'll add the amqp
transport to our `pom.xml`.


<div class="listingblock">

<div class="content">

```
<dependency>
  <groupId>org.mule.transports</groupId>
  <artifactId>mule-transport-amqp</artifactId>
  <version>${amqp.version}</version>
</dependency>
```

At the time of this writting, the newest `amqp.version` is `3.6.2`.
We'll add this to our pom properties.

```
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <mule.version>3.7.0</mule.version>
  <amqp.version>3.6.2</amqp.version>
  <mule.tools.version>1.2-SNAPSHOT</mule.tools.version>
  <jdk.version>1.7</jdk.version>
  <junit.version>4.9</junit.version>
</properties>
```
We'll add the amqp module to our namespaces and create an
`amqp:connector` describing the connection to rabbit.

```
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:amqp="http://www.mulesoft.org/schema/mule/amqp"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
        http://www.mulesoft.org/schema/mule/amqp http://www.mulesoft.org/schema/mule/amqp/current/mule-amqp.xsd
        http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">

  <amqp:connector name="amqpConnector" host="localhost"/>
```

Let's setup a system where we'll publish new users to a queue and
consume them from another flow. We'll add an `amqp:outbound-endpoint` to
our producer flow that sends messages to rabbit. We'll also add an
`amqp:inbound-endpoint` to our consumer flow responsible for reading
messages off the rabbit queue.

```
<flow name="producer">
  <http:listener config-ref="httpListener" path="/*"/>
  <amqp:outbound-endpoint connector-ref="amqpConnector" queueName="NewUser"/>
</flow>

<flow name="consumer">
  <amqp:inbound-endpoint connector-ref="amqpConnector" queueName="NewUser"/>
  <logger level="INFO" message="#[message.payloadAs(java.lang.String)]"/>
  <set-payload value="User created"/>
</flow>
```

We can create this queue from the rabbit management console. Queues can
be added from the 'Queues' tab. Be sure to spell the queue identical to
your queue attribute in Mule.

![rabbitmq addNewQueue](../imgs/rabbitmq_addNewQueue.png){width="500"}

Now that we're all setup, let's redeploy.

```
mvn install
```

Now we'll make many more requests.

```
curl -i -X POST \
  -H 'content-type: application/json' \
  -d '{"name" : "bob"}' \
  http://localhost:8081/users
```

If the queued messages are picked up properly, we'll see the payload
output in our logs.

```
INFO  2016-03-12 17:12:34,780 [[user-registration-1.0-SNAPSHOT].consumer.stage1.07] org.mule.api.processor.LoggerMessageProcessor: {"name" : "bob"}
INFO  2016-03-12 17:12:34,780 [[user-registration-1.0-SNAPSHOT].consumer.stage1.08] org.mule.api.processor.LoggerMessageProcessor: {"name" : "bob"}
INFO  2016-03-12 17:12:34,781 [[user-registration-1.0-SNAPSHOT].consumer.stage1.10] org.mule.api.processor.LoggerMessageProcessor: {"name" : "bob"}
```

Let's now return to the management console at <http://localhost:15672>.
Checkout the overview area where you should now see some traffic has
come through rabbit. You may need to send more requests if more than 60
seconds has passed.

![rabbit mq message
rates](../imgs/rabbit-mq-message-rates.png){width="600"}

That's it! In minutes we're able to spin up a rabbit instance and move
data through queues. While this covers initial connection, checkout the
section below for an example on ack / fetch config.

Playing with ACK and fetch parameters 
-------------------------------------

<div class="sectionbody">

<div class="paragraph">

The `amqp:connector` allows for specifying how you'd like to ack
messages that have been dequeued. For acking, we have 3 options when not
leveraging [amqp
transactions](:https://docs.mulesoft.com/mule-user-guide/v/3.7/amqp-connector-examples#transaction-support).

  ------------ ---------------------------------------------
  AMQP\_AUTO   Default; When dequeued in Mule, ack is sent
  MULE\_AUTO   When flow completes, ack is sent
  MANUAL       Must explicitly send ack or rejection
  ------------ ---------------------------------------------

Let's set the `ackMode` to `MANUAL`, forcing us to specify when a
message gets an ack or rejection.

```xml
<amqp:connector name="amqpConnector" host="localhost" ackMode="MANUAL"/>
```

In our receiving flow, let's ensure only \~50% of requests receive an
ack. For all others, we'll reject and requeue the message.

```
<flow name="consumer">
  <amqp:inbound-endpoint connector-ref="amqpConnector" queueName="NewUser"/>
  <expression-component>
    flowVars['number'] = Math.random() * 100 + 1
  </expression-component>
  <logger level="INFO" message="#[message.payloadAs(java.lang.String)]"/>
  <choice>
    <when expression="#[flowVars['number'] &lt; 50]">
      <amqp:acknowledge-message/>
    </when>
    <otherwise>
      <amqp:reject-message requeue="true"/>
    </otherwise>
  </choice>
</flow>
```

Redeploy the application once again and send some decent load in. I'll
use the following script the push 1000 requests through curl.

```
for i in {1..1000}
  do
    curl -i -X POST \
      -H 'content-type: application/json' \
      -d '{"name" : "bob"}' \
      http://localhost:8081/users
  done
```

You'll likely notice that even though we're processing and re-processing
many requests, it still returns quite quickly. When returning to the
rabbit console, you'll now see a different distribution of delivered
messages vs acked messages.

![rabbit mq message rates high
load](../imgs/rabbit-mq-message-rates-high-load.png){width="700"}

Interestingly, you'll likely have 0 queued messages in the graph above
due to the process being so quick. Let's see if we can change this.

![rabbit mq queue
message](../imgs/rabbit-mq-queue-message.png){width="600"}

We can throttle this by size with `prefetchSize` or quantity with
`perfetchCount`. Let's now ensure our mule only fetches 100 records at a
time.

```
<amqp:connector name="amqpConnector"
                host="localhost"
                ackMode="MANUAL"
                prefetchCount="100"/>
```

Before our next test, let's also introduce a thread sleep of 1 second
between each consumed request.

```
<flow name="consumer">
  <amqp:inbound-endpoint connector-ref="amqpConnector" queueName="NewUser"/>
  <expression-component>
    Thread.sleep(1000);
    flowVars['number'] = Math.random() * 100 + 1;
  </expression-component>
  <!-- other message processors -->
</flow>
```

Let's now redeploy the application, send in the same 1000 request load,
and monitor our activemq queued messages and mule logs. We should now
see the fetch quantity totaling around 100 at any given time. Slowly, we
should also see more and more messages being acked, thus we'll see less
in the queue over time.

![rabbit mq message
totals](../imgs/rabbit-mq-message-totals.png){width="800"}

If you'd like to see the finalized project, feel free to [clone it from
my
GitHub](https://github.com/JoshRosso/mule-rabbitmq-example/tree/hooking-mule-into-rabbitmq).
While there's a ton more to learn about RabbitMQ and amqp, I hope this
proved to be a simple example on how to get bootstrapped, connected, and
playing around with acks/fetchs. Best of luck in your messaging-filled
future!

---

*Last updated: 10/12/2016*

[ [WTFP License](http://www.wtfpl.net) | [Improve this reference](http://github.com/joshrosso/octetz.com) | [View all references](../index.html) ]
