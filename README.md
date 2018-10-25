# commoncrawl

#install java first (has to be java-8 for hive):
sudo apt-get install openjdk-8-jre vim

#install hadoop:

[Hadoop Install](https://github.com/texasbones11/commoncrawl/blob/master/hadoop_install.txt)

############################################################
#apache hive
##external tables for the s3 links
##Hive sits on top of hadoop, then you deal only with hive
##set most searched fields to be a partition to improve performance

#install hive:
##download
wget http://apache.mirrors.ionfish.org/hive/hive-3.0.0/apache-hive-3.0.0-bin.tar.gz
##install
tar -xzvf apache-hive-3.0.0-bin.tar.gz
sudo mv apache-hive-3.0.0-bin /usr/local/hive


#set hive environment:
##update path
vim ~/.bashrc
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin
#export CLASSPATH=$CLASSPATH:/usr/local/Hadoop/lib/*:.
#export CLASSPATH=$CLASSPATH:/usr/local/hive/lib/*:.
##source
source ~/.bashrc

##fix conflicts after running "hive"
mv /usr/local/hive/lib/log4j-slf4j-impl-2.10.0.jar ~

cd /usr/local/hive/conf
cp hive-default.xml.template  hive-site.xml
cp hive-env.sh.template hive-env.sh

vi /usr/local/hive/conf/hive-env.sh
export HADOOP_HEAPSIZE=4096

sudo mkdir -p /user/hive/warehouse
sudo chmod 777 /user/hive/warehouse
vi /usr/local/hive/conf/hive-site.xml

put in beginning of file:
  <property>
    <name>system:java.io.tmpdir</name>
    <value>/tmp/hive/java</value>
  </property>
  <property>
    <name>system:user.name</name>
    <value>${user.name}</value>
  </property>
##################
add to bottom of file
##################
  <property>
     <name>javax.jdo.option.ConnectionURL</name>
     <value>jdbc:mysql://localhost/metastore?createDatabaseIfNotExist=true</value>
     <description>metadata is stored in a MySQL server</description>
  </property>
  <property>
     <name>javax.jdo.option.ConnectionDriverName</name>
     <value>com.mysql.jdbc.Driver</value>
     <description>MySQL JDBC driver class</description>
  </property>
  <property>
     <name>javax.jdo.option.ConnectionUserName</name>
     <value>hiveuser</value>
     <description>user name for connecting to mysql server</description>
  </property>
  <property>
     <name>javax.jdo.option.ConnectionPassword</name>
     <value>hivehive</value>
     <description>password for connecting to mysql server</description>
  </property>

#remove bad characters on line 3221
sudo chmod 777 /tmp/hive
############################################################
#install mariadb-server
sudo apt-get install mariadb-server libmysql-java
ln -s /usr/share/java/mysql-connector-java.jar /usr/local/hive/lib/mysql-connector-java.jar

sudo mysql
create user 'hiveuser'@'%' identified by 'hivehive';
grant all on *.* to 'hiveuser'@localhost identified by 'hivehive';
create database metastore;
use metastore;
source /usr/local/hive/scripts/metastore/upgrade/mysql/hive-schema-3.0.0.mysql.sql;
############################################################
#Hiveserver2
hiveserver2
http://localhost:10002/
############################################################
##install JSON SerDe binaries into hadoop https://github.com/rcongiu/Hive-JSON-Serde
wget http://www.congiu.net/hive-json-serde/1.3.8/hdp23/json-serde-1.3.8-jar-with-dependencies.jar
wget http://www.congiu.net/hive-json-serde/1.3.8/hdp23/json-udf-1.3.8-jar-with-dependencies.jar
mv json*.jar /usr/local/hive/lib

#add jar
add jar /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
#create database
CREATE DATABASE crawl;
#add external table
#####JSON array
add jar /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
CREATE EXTERNAL TABLE `crawl`(
  `Container` array<string>,
  `Envelope` array<string>
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ( "ignore.malformed.json" = "true")
STORED AS TEXTFILE
LOCATION
  's3a://commoncrawl/crawl-data/CC-MAIN-2018-17/segments/1524125936833.6/wat/';


struct<Format:string,`WARC-Header-Length`:string>
struct<Filename:string,Compressed:string,Offset:string>
,`Actual-Content-Length`:string,`Block-Digest`:string

####JSON string
CREATE EXTERNAL TABLE `crawl2`(
  `Container` string,
  `Envelope` string
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ( "ignore.malformed.json" = "true")
STORED AS TEXTFILE
LOCATION
  's3a://commoncrawl/crawl-data/CC-MAIN-2018-17/segments/1524125936833.6/wat/';


#test
select * from crawl;

#hive logs are in /tmp/wjones
############################################################
#elastic search

#elastic search solution:
https://aws.amazon.com/blogs/big-data/indexing-common-crawl-metadata-on-amazon-emr-using-cascading-and-elasticsearch/
##github:
https://github.com/aws-samples/aws-big-data-blog/tree/master/aws-blog-elasticsearch-cascading-commoncrawl/commoncrawl.cascading.elasticsearch

#create indexes 
https://github.com/aws-samples/aws-big-data-blog/blob/master/aws-blog-elasticsearch-cascading-commoncrawl/commoncrawl.cascading.elasticsearch/src/main/java/com/amazonaws/bigdatablog/indexcommoncrawl/CommonCrawlIndex.java

#install
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.2.4.tar.gz
tar -xzf elasticsearch-6.2.4.tar.gz
sudo mv elasticsearch-6.2.4 /usr/local/elasticsearch

vim ~/.bashrc
export ES_HOME=/usr/local/elasticsearch
export PATH=$PATH:$ES_HOME/bin

sudo vi /etc/security/limits.conf
*    hard	 nofile		 65536
*    soft    	 nofile          65536

sudo vi /etc/systemd/user.conf
DefaultLimitNOFILE=65536

sudo vi /etc/systemd/system.conf
DefaultLimitNOFILE=65536

#start elasticsearch
elasticsearch

#download jar
wget http://central.maven.org/maven2/org/elasticsearch/elasticsearch-hadoop/6.2.4/elasticsearch-hadoop-6.2.4.jar
mv elasticsearch-hadoop-6.2.4.jar /usr/local/hive/lib/
#hive connections
vi /usr/local/hive/conf/hive-site.xml
<property>
  <name>hive.aux.jars.path</name>
  <value>/usr/local/hive/lib/elasticsearch-hadoop-6.2.4.jar</value>
  <description>A comma separated list (with no spaces) of the jar files</description>
</property>

#hive
ADD JAR /usr/local/hive/lib/elasticsearch-hadoop-6.2.4.jar;

#create table
CREATE EXTERNAL TABLE elastic (data STRING)
STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler'
TBLPROPERTIES('es.nodes' = 'localhost', 'es.resource' = 'elastic/Container', 'es.input.json' = 'yes');

#inset indexes into table
INSERT OVERWRITE TABLE elastic
select * from crawl;

#test
select * from elastic limit 10;

TBLPROPERTIES('es.nodes' = 'localhost', 'es.resource' = 'elastic/Container', 'es.input.json' = 'yes');


################SMALL DATASET EXAMPLE######################
wget https://commoncrawl.s3.amazonaws.com/crawl-data/CC-MAIN-2018-17/segments/1524125936833.6/wat/CC-MAIN-20180419091546-20180419111546-00000.warc.wat.gz
gunzip CC-MAIN-20180419091546-20180419111546-00000.warc.wat.gz

CREATE EXTERNAL TABLE `smallcrawl`(
  `line` STRING
)
ROW FORMAT SERDE 
  'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'input.regex' = '(^\\{\\"Container\\".*$)'
)
STORED AS TEXTFILE
LOCATION
  '/hive/smallcrawl';

LOAD DATA INPATH '/home/wjones/CC-MAIN-20180419091546-20180419111546-00000.warc.wat' INTO TABLE smallcrawl;



################HIVE TUNING#################


There's a couple of things you need to address here:

Total JVM memory allocated vs. JVM heap memory

The total JVM memory allocated is set through these parameters:

mapreduce.map.memory.mb
mapreduce.reduce.memory.mb

The JVM heap memory is set through these parameters:

mapreduce.map.java.opts
mapreduce.reduce.java.opts

You must always ensure that Total memory > heap memory. (Notice that this rule is violated in the parameter values you provided)

Total-to-heap ratio

One of our vendors recommended that we should, for the most part, always use roughly 80% of the total memory for heap. Even with this recommendation you will often encounter various memory errors.

Error: heap memory

Probably need to increase both total and heap.

Error: Permgen space not enough

Need to increase the off-heap memory which means you might be able to decrease the heap memory without having to increase the total memory.

Error: GC overhead limit exceeded

This refers to the amount of time that the JVM is allowed to garbage collect. If too little space is received in a very long time, then it will proceed to error out. Try increasing both total and heap memory.
#######################################################
#THIS IS WORKING!!!!!!!!!! YAYYYYYYY!!!!!!!!!!!
add jar /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
CREATE EXTERNAL TABLE `crawl`(
  `Container` array<string>
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ( "ignore.malformed.json" = "true")
STORED AS TEXTFILE
LOCATION
  's3a://commoncrawl/crawl-data/CC-MAIN-2018-17/segments/1524125936833.6/wat/';


#############################View still has to run at runtime##########################
add jar /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
CREATE TABLE `crawl2`(
  `line` STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES ( "ignore.malformed.json" = "true")
STORED AS TEXTFILE;

INSERT OVERWRITE TABLE crawl2
select * from smallcrawl where line is not null;

CREATE VIEW happy AS SELECT * FROM smallcrawl where line is not null;
#######################Temporary file stored in scratch folder################################
add jar /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
CREATE TEMPORARY EXTERNAL TABLE `crawl3`(
  `line` STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES ( "ignore.malformed.json" = "true")
STORED AS TEXTFILE;
INSERT OVERWRITE TABLE crawl3
select * from smallcrawl where line is not null;

###########################################################################
#Elastiocsearch with json serde # hit an issue with how many columns elasticsearch could do
#hive
ADD JAR /usr/local/hive/lib/elasticsearch-hadoop-6.2.4.jar;
ADD JAR /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
#create table
CREATE EXTERNAL TABLE elastic (data STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
WITH SERDEPROPERTIES ( "ignore.malformed.json" = "true")
STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler'
TBLPROPERTIES('es.nodes' = 'localhost', 'es.resource' = 'elastic/Container', 'es.input.json' = 'yes', 'es.read.field.as.array.exclude' = *);

#inset indexes into table
INSERT OVERWRITE TABLE elastic
select * from smallcrawl where line is not null;

#test
select * from elastic limit 10;

TBLPROPERTIES('es.nodes' = 'localhost', 'es.resource' = 'elastic/Container', 'es.input.json' = 'yes');


, 'es.read.field.as.array.exclude' = *

###########################################################################
#alter table attempt - only let me do one, it overwrite the other
add jar /usr/local/hive/lib/json-serde-1.3.8-jar-with-dependencies.jar;
ALTER TABLE crawl SET SERDE 'org.openx.data.jsonserde.JsonSerDe' WITH SERDEPROPERTIES ("ignore.malformed.json" = "true");

#go back
ALTER TABLE crawl SET SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe' WITH SERDEPROPERTIES ('input.regex' = '(^\\{\\"Container\\".*$)');
############################################################################
#regex attempt that works
CREATE EXTERNAL TABLE `crawl`(
  `line` STRING
)
ROW FORMAT SERDE 
  'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'input.regex' = '(^\\{\\"Container\\".*$)'
)
STORED AS TEXTFILE
LOCATION
  's3a://commoncrawl/crawl-data/CC-MAIN-2018-17/segments/1524125936833.6/wat/';
############################################################################
#index on externalcrawl table #it gave an error and orc is preferred anyway

CREATE INDEX power ON TABLE crawl2(Envelope)
AS 'org.apache.hadoop.hive.ql.index.compaxct.CompactIndexHandler';
#########################################
#search example: trying to get all the rows that contain "crawl" anywhere
select Envelope from crawl2 where instr(Envelope, 'crawl') != 0;
