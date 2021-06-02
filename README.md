# ksql-quick-build
Quickly enable a ksql environment for learning.

## Goals
* Automate a repeatable environment for KSQL learning.
* Allow external data to be fed into KSQL objects.
* At minimum, enable a topic, stream and table.
* Allow queries to be run against the stream and table.
* Bundle it into a single stack.
* Don't lock-out future ideas.

## Basic Scenario
1. We have orders moving through a web store.
2. Each order transitions through various states as it is processed.
3. At any point in time, we need to query an order's current state using the order ID.
4. The life of orders is in days, and so we'll assume kafka topic retention simply rolls-off older data.

## TLDR - Let me just get it running!
Ok for those who can't wait, the instructions are as follows:
1. Clone the repo.
2. Run **docker-compose up -d**.
3. Wait for all services to be ready.
4. Run **docker-compose ps** to check the state of services.
5. Now let's launch a KSQL Client interactive session to check the state of our objects.
6. Run **docker-compose exec ksqldb-cli ksql http://primary-ksqldb-server:8088**.
7. This should result in a ksql prompt: **ksql >**
8. Let's first check that we have all our objects as expected.
9. Run the following, hitting enter after each to see the results.
   9a. **SHOW TOPICS;**
   9b. **SHOW STREAMS;**
   9c. **SHOW TABLES;**
10. You will see some internal kafka and ksql objects too, but you should see the following:
   10a. Topics - *orders*, *companies*, *spare*
   10b. Streams - *S_ORDERS*
   10c. Tables - *T_ORDERS*
11. If that worked, you are all set.
   11a. If it didn't, try looking at the logs using **docker-compose logs**
   11b. You can also look specifically at the logs for a given image by using **docker-compose logs [image-name]**

### Creating some events
1. Drop one of the *json* files from the */data-files* folder into the */data* folder.
2. This feeds the file parser, which will read each line into a new *order-event* and publish that event to the *orders* topic in Kafka.
3. From there the KSQL magic will turn it into a stream and an aggregate table.
4. Now in your KSQL Client (launched just before) try the following query:
   * **SELECT * FROM t_orders WHERE O_ID IN ('001','002','003','004','005');**
   * Alternatively, I could have used a *BETWEEN* clause.

That should get you going enough to start playing around with running KSQL Queries.
*Have Fun!*

# The Geeky Details
Here's where I delve deep into the way I pulled all this together.
I hope you find it useful and can use some of these concepts and approaches in your own projects.

## Data Model
1 Kafka Topic for orders called **orders**
1 KSQL STREAM for each order-event state called **S_ORDERS**
1 KSQL TABLE for the current state of each order called **T_ORDERS**

For this exercise I didn't bother to further normalise the data, but it could be a future goal. (and as you'll see in the scripts, I created a couple of slots in preparation)

An *order-event* contains the following data:
Field | Type | Purpose
----- | ---- | -------
**ts** | timestamp(epoch) | the event time
**_id** | string | the order id
**guid** | guid | a unique guid for each event
**state** | string (enum) | new order state (ordered>paid>submitted>stocked>assembled>packed>shipped>received>complete)
**name** | string | contact name for order
**company** | string | company name for order
**email** | string | email of contact

### The planned Data Flow
l. A *JSON* file is dropped into a parser target folder.
l. The parser will read each line of the file into an event.
l. Each event is submitted in *JSON* format to the **orders** topic, the *_id* field is defined as the kafka message key.
   l. (Using the *_id* field as a message key ensures all events from the same order are managed by the same kafka broker)
l. A KSQL STREAM (**S_ORDERS**) is created from the **orders** topic, extracting only the *_id*, *state* and *email* data.
l. A KSQL TABLE (**T_ORDERS**) is created from the **S_ORDERS** stream containing the *state* and grouping by the *_id*.
   l. This **T_ORDERS** table allows us to query the latest state for each order using the order ID.

### Building-up the Kafka elements
These are the extracted statements used to build the kafka elements.
* Creating the Kafka Topic:
  * **kafka-topics --create --if-not-exists --bootstrap-server kafka:9092 --partitions 1 --replication-factor 1 --topic orders**
* Creating the KSQL Stream:
  * **CREATE STREAM s_orders (id VARCHAR KEY, state VARCHAR, cust_email VARCHAR) WITH (KAFKA_TOPIC = 'orders', VALUE_FORMAT = 'JSON');**
* Creating the KSQL Table:
  * **CREATE TABLE t_orders AS SELECT ID AS O_ID, LATEST_BY_OFFSET(STATE) AS O_STATE FROM s_orders GROUP BY ID;**

*It is important to note that when we publish an event, we **MUST** set the Kafka Message Key using the value held in the **_id** field.*
*Without this, the KSQL Stream would not be able to declare the **id** field value as inherited from the message key.*
*You can see this in the **CREATE STREAM** statement above, with the use of the **KEY** directive.  This tells KSQL to inject the kafka message key into the field value **id**.*
*Once we have that in place, we can group on the **id** when we create the table.*
*I also want to point-out the use of the very useful aggregate function **LATEST_BY_OFFSET**, which ensures we get the latest received value for **state** against every order.  Without this function, acheiving this particular use case would have been a great challenge.*

### Generating Synthetic/Mock Data
Data was created using a JSON Mock service:
https://www.mockaroo.com

Using the following Data Config:
* **ts** *Datetime* [from] [to] *epoch*
* **_id** *Custom List* 001,002,003,004,005 *random*
* **guid** *GUID*
* **state** *Custom List* ordered,paid,submitted,stocked,assembled,packed,shipped,received,complete *random*
* **name** *Full Name*
* **company**  *Company Name*
* **email** *Email Address*
And the following Generation Profile:
* *Rows 50, Format JSON, [blank] array, [checked] include null values*

## Constructing the Feeder
The data feeder mechanism was created using FluentBit.
*This was because I have prior experience and find the tool incredibly simple and powerful.*
The config for FluentBit is incredibly simple for this exercise and the bundled plugins meant there was no extra build needed.
First we set a parser section in the fluent-bit.conf file:
```
[SERVICE]
    Parsers_File parsers.conf
```
This tells FluentBit where to find the included set of parser patterns.  You don't need to create the file, as it is already bundled.

Next we tell FluentBit what we want to use as data input:
```
[INPUT]
    Name tail
    Path /data/*
    Read_from_Head True
    Tag events
    Parser json
```
The first **[INPUT]** is a fixed reserved section and is used to indicate an input instruction.
In this case we are using the file tail input plugin and so we define this with **Name tail**.
The **Path** parameter tells FluentBit where to look for data files and in this case we set the path as the local */data* folder.
Each input in FluentBit has to be tagged to distinguish one from another, and this tag is used later in assigning flow instructions.
In the above we have assigned a tag of *events* using **Tag events**.
Finally because this is a *JSON* file, we tell FluentBit that we would like it to use the json parser.  
We do this using the instruction **Parser json**, which will allow FluentBit to find the *json* parser defined in the *parsers.conf* file we set earlier.

As a final step in this simple configuration, we define where we want our data to be sent:
```
[OUTPUT]
    Name kafka
    Match events
    Message_key_field _id
    Brokers kafka:9092
    Topics orders
```
We do this using another reserved section called **[OUTPUT]**.
And again we instruct the plugin type using **Name kafka** to define our output as the *kafka* plugin.
The next item is where we define the flow from the *[INPUT]*.  We want this flow to be routed from the *tail* input that we tagged as *events*.
We do this using the line **Match events**, which tells FluentBit to match any messages with a tag of *events* for this *[OUTPUT]*.
The next three lines are specific to the *kafka* plugin.
**Message_key_field _id** informs FluentBit that we want the inbound field *_id* to be set as the kafka message key.  As you read in the above data section, this is very important and ensures kafka handles every order within the same message broker, allowing us to use that order id as the grouping construct for our KSQL Table.
We have to tell the kafka plugin where to connect to kafka, and we do this using **Brokers kafka:9092** which is the address allocated in the *docker-compose.yml* file.  This *Brokers* section is not in fact a list of all brokers, but a list of what we call *bootstrap brokers* whose job it is to provide new clients with metadata about the cluster, the topics, partitions, brokers and partition leaders.  It is then the job of the client to connect with the appropriate broker for the topics in question.
Lastly we tell FluentBit that we want our messages sent to the *orders* topic and we do this with the line **Topics orders*.

And that's all we have to do with the config of FluentBit.  We save the file as *fluent-bit.conf* in the root of our project folder.
Later we'll instruct the FluentBit container where to locate the config, when we build-out the *docker-compose.yml* file.

## Assembling the Automated Environment
As you'll have picked-up, I used docker-compose as the mechanism to assemble my learning environment.
*Reason:* I wanted this to be as simple as possible, because the main aim was really to help people quickly launch a learning environment for KSQL.  Because it is a learning environment, it is clear that things can go wrong and sometimes it is nice to go back and refresh an exercise.  For this reason it is important to be able to quickly start afresh and also to know that startup doesn't require any choices, just a repeatable simple process.  So using containers was an obvious choice.  The reason I decided to use docker-compose was really due to the fact that there was already a basic ksql project in place curtesy of Confluent.  So I took that example and added some extra parts to the mix in order to form this complete testing environment.

### Citing resources
The following resources were used during the creation of this project:
* This was my starting point from where I obtained the bulk of the docker-compose details:
  * https://ksqldb.io/quickstart.html#quickstart-content 
* This guide was also useful and introduced me to 'cub':
  * https://docs.confluent.io/platform/current/installation/docker/development.html
* This is the github page for cub *Confluent Utility Belt*:
  * https://github.com/confluentinc/confluent-docker-utils/blob/master/confluent/docker_utils/cub.py 
* This was a useful KSQL Reference:
  * https://docs.confluent.io/5.4.2/ksql/docs/developer-guide/syntax-reference.html#ksql-syntax-reference 
* This script section was how I handled the KSQL stream and table creation from within docker-compose:
  * https://docs.confluent.io/5.4.2/ksql/docs/installation/install-ksql-with-docker.html#ksql-execute-script-in-cli
* This is the kafka plugin for FluentBit:
  * https://docs.fluentbit.io/manual/pipeline/outputs/kafka
* Here's the Docker Compose guide:
  * https://docs.docker.com/compose/gettingstarted/ 

### The Environment Defined
![Environment Layout](/images/environment-layout.png)

***A run down of the environment***
* *Kafka* relies on *Zookeeper* for cluster resource management.
* *Schema Registry* provides useful features for message definition and that helps with controlling data in the message to maintain data integrity with ksql objects.
* The *KSQLDB Server* is providing the query engine and data storage interface on top of Kafka topics.
* *KSQLDB CLI* is needed to issue queries against the KSQLDB.
* *Kafka Setup* and *KSQL Setup* are utility containers that are required only to create and provision the environment.
  * *Kafka Setup* simply creates kafka topics once Kafka is up and ready.
  * *KSQL Setup* creates the Stream and Table object we need for the learning.
* *Fluent-Bit* provides the data ingestion flow from *data files* containing json formatted strings of event data.
  * Starter Data Files are provided in the */data-files* folder.
  * After things are started-up these need to be dropped into the */data* folder for *FluentBit* to ingest them.

### Walking through the docker-compose.yml file
The *docker-compose.yml* file defines all of the instructions needed to create the container environment.
I'll walk through each section and highlight any interesting points.

#### Zookeeper (service: zookeeper)
Very simple here, we pull the confluent image and set a client connection port *(32181)* and sync time *(200 milliseconds)*.

#### Kafka (service: kafka)
A little more details but essentially, we pull the image, set a port and define that this service is dependent upon the *zookeeper* service.
The interesting config lines here are:
* `KAFKA_BROKER_ID: 1` sets the identity of the broker, and is useful when you have a larger multi-broker cluster.
* `KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181` sets the zookeeper address and port for cluster resource control. (notice how this is the same port defined in the zookeeper service)
* `KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092` sets how Kafka will advertise itself to others.  In this case all clients will use *kafka:9092*.
* `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1` defines how kafka replicates topics, in this case we are only using for learning so we don't replicate any data and use a factor of *1*.

#### Schema Registry (service: schema-registry)
Another simple one, we pull the image, set dependencies on *zookeeper* and *kafka*.
A config here also points to the *zookeeper* address.

#### KSQLDB Server (service: primary-ksqldb-server and additional-ksqldb-server)
Note this time we have two services, which are created to help provide extra capacity.
In addition to pulling the image, you can see that here we use *hostname* and *container_name* to ensure that we can clearly identify the primary and additional server.
The primary is set with a dependency on *kafka* and *schema-registry*, whilst the additional is just dependent on the primary.
Ports are set to ensure no clashes.  Note that primary is also set to map to the local machine port *8088* whilst additional simply sets as *8090*.
Finally both set the connection to *kafka* and *schema-registry* and assign a listener using their respective port.

#### KSQLDB CLI (service: ksqldb-cli)
Here we pull the image and set the *container_name*, defining a dependency on *primary-ksqldb-server*.
The interesting configs here are:
`entrypoint: /bin/sh` which sets the starting context for the docker session once the container loads.
`tty: true` creates a virtual terminal session within the docker environment.  This allows us to connect to the client via a terminal and run interactive queries.

#### Kafka Setup (service: kafka-setup)
This is a utility container that we use to automate the creation of topics.
We automate this so that we can avoid the need for additional post build steps, and keep the environment build a single step.
The image is also a kafka server, because it contains the necessary kafka utilities for creating topics.
This depends on *kafka*, *schema-registry* and *primary-ksqldb-server* just so we maintain integrity of the environment.
The interesting part is the `command:` item.
There are a few steps of note:
* It is a basic bash script using -c to indicate a concatenation of commands (ie. we want to do more than one thing)
* In `cub kafka-ready` we are using a Confluent utility (*Confluent Utility Belt*) to pause until kafka is fully up and ready to accept requests.  This is useful to reinforce the likelihood that our command to create topics will succeed. (not waiting would likely fail as kafka wouldn't be ready when the container came-up)
* We then have 3 `kafka-topics --create` commands which are the core automation of creating *orders*, *companies* and *spare* kafka topics.  I need *orders* for this exercise, but wanted to show how to create more and thought it would allow an additional exercise with the data at a later time.
* Most of the rest are just wait commands to allow everything to come-up in order.
* Noticably the command `cub ksql-server-ready` is waiting to ensure our *primary-ksqldb-server* is available before we move-on.

#### KSQL Setup (service: ksql-setup)
This is very similar to *kafka-setup* but for KSQL.
I separated these out primarily because I struggled to get all the commands working in a single bash command string, but then thought it would be useful for have separation across the two as different ideas formed.
So this time the image is a ksql-cli image (the same image we used for the *ksqldb-cli* service).
Points of interest are:
* `depends_on:` sets dependency on kafka-setup to ensure we have already created our kafka topic before we attempt to create a KSQL Stream against it.
* `volumes:` defines a map of the local */ksql-scripts* folder into the container's */data/scripts* folder.  This will contain the KSQL needed for our object creation.
* `entrypoint:` defines the script to run on startup that will execute our KSQL commands.
* This once again calls a bash script.
* The while loop essentially waits until it has a positive response from the *primary-ksqldb-server* service.
* Then it uses *ksql-prep-script.sql* to push commands into a *ksql* session on the *primary-ksqldb-server*.
* Finally it sleeps.

The *ksql-prep-script.sql* is part of this build and is stored in the */ksql-scripts* folder.  When we map this local folder to */data/scripts* it allows the **cat** command to pull these commands and execute against the KSQL session.
The script is as follows:
```
CREATE STREAM s_orders (id VARCHAR KEY, state VARCHAR, cust_email VARCHAR) WITH (KAFKA_TOPIC = 'orders', VALUE_FORMAT = 'JSON');
CREATE TABLE t_orders AS SELECT ID AS O_ID, LATEST_BY_OFFSET(STATE) AS O_STATE FROM s_orders GROUP BY ID;
SHOW TOPICS;
SHOW STREAMS;
SHOW TABLES;
SHOW PROPERTIES;
```
You'll notice that we not only create the stream and table objects but gather other data too.  This was to help in any debugging or identifying of problems, as the output of these commands can then be retreived using the `docker-compose logs ksql-setup` command.

The details of the commands is discussed above in the *The Planned Data Flow* section.

#### Fluent-Bit (service: fluent-bit)
Another very simple config where we first pull the fluent-bit image.
We set a dependency on *kafka-setup* to ensure we have a topic to push the data to.
Here's the interesting parts:
* `volumes:` defines two maps as follows:
  * `./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf` maps the included fluent-bit.conf file from the local root folder to the */fluent-bit/etc* folder in the container where it is expected.  This allows FluentBit to read our config and behave as expected.
  * `./data:/data` maps our local */data* folder to the container's */data* folder so that when we drop our data files, the FluentBit tail plugin can read them as if they were inside the container.

Details of the FluentBit config are explained in the above section *Constructing the Feeder*.


# Finally
I hope this is a useful project to get you started on KSQL.
I also hope you find the information in this ReadMe useful and can use it to better your understanding of the approach I've used and tools I've taken advantage of.

