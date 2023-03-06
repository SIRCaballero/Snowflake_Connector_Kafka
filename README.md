# Snowflake Connector Kafka

Step 1 --- Getting Ready
======================

In order to execute the steps outlined in this blog post, you will need an AWS account and some familiarity on how to create and connect to your EC2 instances on AWS. On Windows desktops, you will need Putty and WinSCP. MacOS and Linux users can use their Terminal for the same. We also assume that you have created a Key-Pair using for AWS account to access your instances.

We will create a Linux machine on EC2 as follows:. Login to your AWS Console

1.  From EC2 Menu, select Launch Instances
2.  From the Quick Start Menu, select Ubuntu Server 18.04 LTS (HVM), SSD Volume Type 64-bit (x86)
3.  Choose Instance Type t2.medium (this will cost) from the next menu
4.  Click Review and Launch
5.  Go to Edit Security Group -> Add Rule
6.  Launch Instance and go to Instances panel in EC2 Menu
7.  Wait for instance to launch completely until State is Running and Status shows 2/2
8.  Select the instance and edit Instance Name as Kafka Server. Note its Public ipv4 DNS from pane below

Step 2 --- Set Up Snowflake To Receive Data
=========================================

Now, we generate a Private Key to connect to our Snowflake Account. Connect to your Kafka-Server and follow the steps below:

$ openssl genrsa -out rsa_key.pem 2048

1.  Generate a public key referencing the above private key:

$ openssl rsa -in rsa_key.pem -pubout -out rsa_key.pub

![](https://miro.medium.com/v2/resize:fit:1400/0*PznNNYgZILjHU-qp)

2\. You should see your key pair as above. Let's check the contents of our key file:

$ cat rsa_key.pub rsa_key.pem

3\. Copy the contents of rsa_key.pub except for the comment lines

4\. Now login to Snowflake and go to the Worksheets panel and switch to the SECURITYADMIN role and issue these commands in your worksheet:

-   Create user kafka_snowflake_connector rsa_public_key='<paste contents of rsa_key.pub from the previous step>'
-   Grant role sysadmin to user kaka_snowflake_connector
-   Create database PRODUCTION

5\. Create a Snowflake role with the privileges to work with the connector:

-   Use role SECURITYADMIN;
-   create role kafka_connector_role;
-   Grant usage on database PRODUCTION to role kafka_connector_role;

6\. Grant privileges on the schema to Kafka Connector Role:

-   grant usage on schema PRODUCTION.PUBLIC to role kafka_connector_role;
-   grant create table on schema PRODUCTION.PUBLIC to role kafka_connector_role;
-   grant create stage on schema PRODUCTION.PUBLIC to role kafka_connector_role;
-   grant create pipe on schema PRODUCTION.PUBLIC to role kafka_connector_role;

7\. Grant the custom role to an existing user:

-   grant role kafka_connector_role to user kafka_snowflake_connector;

8\. Make the new role the default role:

-   alter user kafka_snowflake_connector set default_role=kafka_connector_role;

Step 3 --- Install Zookeeper
==========================

Connect to your Kafka-Server and follow the steps below:

1.  Install Java-8:

$ sudo apt-get install openjdk-8-jdk

2\. Download Zookeeper from Apache Zookeeper site:

$ wget <https://mirrors.estointernet.in/apache/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz>

3\. Extract the Zookeeper tarball:

$ tar -xvzf apache-zookeeper-3.6.2-bin.tar.gz

4\. Rename Zookeeper directory for ease of use:

$ mv apache-zookeeper-3.6.2-bin.tar.gz zookeeper

5\. We need a configuration file to start Zookeeper service. Let's use the sample default file by copying it as zoo.cfg:

$ mv mv zookeeper/conf/zoo_sample.cfg zookeeper/conf/zoo.cfg

We now move on to install Kafka on the same machine.

Step 4 --- Install Kafka
======================

1.  Download Kafka from Apache Kafka site. We will use version 2.5.0 with built-in Scala 2.11

$ wget <https://mirrors.estointernet.in/apache/kafka/2.5.0/kafka_2.12-2.5.0.tgz>

2\. Extract Kafka Tarball:

$ tar -xvzf kafka_2.12--2.5.0.tgz

3\. Rename Kafka directory for ease of use:

$ tar mv kafka_2.12--2.5.0 kafka

Step 5 --- Set Up Snowflake Kafka Connector
=========================================

1.  Download Snowflake Kafka Connector Jar files from Maven Central. We will use stable version 1.3 for this exercise.

$ wget <https://repo1.maven.org/maven2/com/snowflake/snowflake-kafka-connector/1.5.0/snowflake-kafka-connector-1.5.0.jar>

2\. Move the downloaded Kafka Connector Jar to kafka/libs directory:

$ mv snowflake-kafka-connector-1.5.0.jar kafka/libs

3\. Next, we configure the Kafka Snowflake Connector to consume topics from our Kafka cluster and push the data into Snowflake:

We will name it snowflakesink for want of a better name. We need to specify Snowflake Connection details, private key for authentication, and target schema and tables in Snowflake. For this let's create a new properties file named connect-snowflake-kafka-connector.properties. and paste the contents below into that file.

Change highlighted properties to suit your sample data and Save the file in kafka/config directory. By default connector will create tables in Snowflake and name them as the Topic itself.

If you wish to load data in specific tables, please provide the topic2table.map property below. For multiple tables follow convention --- <topic1:table1,topic2:table2...>

$ vi kafka/config/connect-snowflake-kafka-connector.properties

> name=snowflakesink
>
> connector.class=com.snowflake.kafka.connector.SnowflakeSinkConnector
>
> tasks.max=1
>
> topics=<Tpoics to consume separated by comma>
>
> #snowflake.topic2table.map=employees.employees.dept_manager:production.departments
>
> buffer.count.records=10000
>
> buffer.flush.time=60
>
> buffer.size.bytes=5000000
>
> snowflake.url.name=ia29518.east-us-2.azure.snowflakecomputing.com/
>
> snowflake.user.name=kafka_connector_role
>
> snowflake.private.key=<< Paster your Snowflake Account Private Key here.
>
> At the end of each line (except for the last) add a '\' as shown below>> MIIEogIBAAKCAQEAsRqORWkYfloAoX2NLK5NjN/iS2aI0ngDi3k8xugo5eobDCtV\
>
> 0tO8KhyelVg5+Pf5i+yOADqtjMsb6w53BFNZWTa8Fznmbse02r4=
>
> snowflake.database.name=production
>
> snowflake.schema.name=public
>
> #key.converter=org.apache.kafka.connect.storage.StringConverter
>
> #value.converter=com.snowflake.kafka.connector.records.SnowflakeAvroConverter
>
> #value.converter.schema.registry.url=http://localhost:8081
>
> #value.converter.basic.auth.credentials.source=USER_INFO
>
> #value.converter.basic.auth.user.info=jane.smith:MyStrongPassword

4\. Save the file. We are now done with all the setup!

Step 6 --- Operation
==================

We are now ready to publish some test data into our Kafka Cluster and push it to Snowflake. For the best learning experience, we suggest that you open multiple Terminal Windows (Say 5) and connect to your EC2 instance from each of these.

Window 1 --- Start Zookeeper Service
----------------------------------

$ kafka/bin/zookeeper-server-start.sh kafka/config/zookeeper.properties

![](https://miro.medium.com/v2/resize:fit:1400/0*fpuLYrXJGD29ORfd)

You should see the service running as above.

Window 2 --- Start Kafka Service:
-------------------------------

$ kafka/bin/kafka-server-start.sh kafka/config/server.properties

![](https://miro.medium.com/v2/resize:fit:1400/0*lzSG5OWSIFYwQbWb)

Window 3 --- Create Sample Data and Publish to a Kafka Topic
----------------------------------------------------------

1.  Create a test data file dividends.json with the following command:

$ vi dividends.com

2\. Paste the following sample data into that file:

> {"exch":{"string":"NYSE"},"symbol":{"string":"CJA"},"date":{"string":"2009--10--16"},"dividend":{"double":0.501}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CPO"},"date":{"string":"2009--12--30"},"dividend":{"double":0.14}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CPO"},"date":{"string":"2009--09--28"},"dividend":{"double":0.14}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CPO"},"date":{"string":"2009--06--26"},"dividend":{"double":0.14}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CJA"},"date":{"string":"2009--08--27"},"dividend":{"double":0.688}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CPO"},"date":{"string":"2009--03--27"},"dividend":{"double":0.14}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CJA"},"date":{"string":"2009--05--27"},"dividend":{"double":0.688}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CPO"},"date":{"string":"2009--01--06"},"dividend":{"double":0.14}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CCS"},"date":{"string":"2009--10--28"},"dividend":{"double":0.414}}
>
> {"exch":{"string":"NYSE"},"symbol":{"string":"CJA"},"date":{"string":"2009--04--16"},"dividend":{"double":0.524}}

3\. Save and exit.

4\. Run the following commands to see if any Kafka Topics are created:

$ kafka/bin/kafka-topics.sh --- list --- zookeeper localhost:2181

5\. The command fails to show any topics. Next, we create a test topic for our use called 'dividends':

$ kafka/bin/kafka-topics.sh --- create --- zookeeper localhost:2181 --- replication-factor 1 --- partitions 1 --- topic dividends

6\. Use the command before to ensure that topic dividends is listed. We now have to publish our data into this topic as messages. Normally, we will write code for the same but for our test, we will use Kafka's Console Producer, which we can run as a command below:

$ kafka/bin/kafka-console-producer.sh --- broker-list localhost:9092 --- topic dividends < dividends.json

7\. Console producer takes inputs from the standard in which we have redirected to our test file. We can now see the messages with the following commands:

$ kafka/bin/kafka-console-consumer.sh --- bootstrap-server localhost:9092 --- topic dividends --- from-beginning

8\. You can Press CTRL-C anytime to exit the console consumer. You can run producer from another window and watch messages into this window. Omit --from-beginning option if you want to see only the latest messages:

![](https://miro.medium.com/v2/resize:fit:1400/0*4DQO7IvT_iJJ-3J0)

Window 4 --- Start Kafka Connect Service with Snowflake-Kafka-Connector
---------------------------------------------------------------------

$ kafka/bin/connect-standalone.sh kafka/config/connect-standalone.properties kafka/config/connect-snowflake-kafka-connector.properties

9\. After pushing the initial set of messages the Kafka Connect service will wait as you see below:

![](https://miro.medium.com/v2/resize:fit:1400/0*4CkMQAG-MvanFGHM)

At any time, you can use check jps command to ensure that all 3 services are running on your machine --- 1. QuorumPeerMain (zookeeper), 2- Kafka and 3-ConnectStandalone (Kafka-Connect Task).

Window 5--- Login to Snowflake
----------------------------

You can now connect to Snowflake and see the data loaded in table production.public.dividends. You can make changes to data and push more events in Kafka topic and you will see those events appearing as fresh rows in your Snowflake table.

Summary
=======

This was a quick-start guide to give you an idea of how to get started with the Snowflake Connector for Kafka. There are many ways in which you can configure the connector for your specific needs. Some next steps are:

1.  Use of AVRO format is instead of JSON significantly reducing the amount of metadata wrapped around each message.
2.  Use of Kafka Schema Registry for automatic detection of changes to source schema and applying them in Snowflake tables.
3.  Use of a REST API to post properties to Kafka Connect server instead of using properties file. REST makes it possible to load new connectors or change properties of existing ones at run time without having to shut down the service.
