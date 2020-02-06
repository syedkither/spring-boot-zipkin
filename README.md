# ZIPKIN and ELK STACK

Zipkin is very efficient tool for distributed tracing in microservices ecosystem. Distributed tracing, in general, is latency measurement of each component in a distributed transaction where multiple microservices are invoked to serve a single business usecase. Let’s say from our application, we have to call 4 different services/components for a transaction. Here with distributed tracing enabled, we can measure which component took how much time.

Spring Cloud Sleuth may also send tracing statistics to Zipkin. That is another kind of data than the data stored in Elastic Stack. These are timing statistics for each request. Zipkin UI is really simple. You can filter the requests by some criteria like time, service name, and endpoint name.

Once Zipkin is started, we can see the Web UI at http://localhost:9411/zipkin/

Internally it has 4 modules
--------------------------
* Collector – Once any component sends the trace data arrives to Zipkin collector daemon, it is validated, stored, and indexed for lookups by the Zipkin collector.
* Storage – This module store and index the lookup data in backend. Cassandra, ElasticSearch and MySQL are supported.
* Search – This module provides a simple JSON API for finding and retrieving traces stored in backend. The primary consumer of this API is the Web UI.
* Web UI – A very nice UI interface for viewing traces.



Sleuth
-------
Sleuth is a tool from Spring cloud family. It is used to generate the trace id, span id and add these information to the service calls in the headers and MDC, so that It can be used by tools like Zipkin and ELK etc. to store, index and process log files.

Spring Cloud Sleuth is used to generate and attach the trace id, span id to the logs so that these can then be used by tools like Zipkin and ELK for storage and analysis.

Zipkin is a distributed tracing system. It helps gather timing data needed to troubleshoot latency problems in service architectures. Features include both the collection and lookup of this data


The Tools
----------
  Spring Cloud Sleuth: 
  
        A Spring Cloud library that lets you track the progress of subsequent microservices by adding trace and span id’s on the appropriate HTTP request headers. The library is based on the MDC (Mapped Diagnostic Context) concept, where you can easily extract values put to context and display them in the logs.
        
  Zipkin: 
  
    A Java-based distributed tracing application that helps gather timing data for every request propagated between independent services. It has a simple management console where we can find a visualization of the time statistics generated by subsequent services.
  
  ELK Stack: 
  
    Three open source tools — Elasticsearch, Logstash and Kibana form the ELK stack. They are used for searching, analyzing, and visualizing log data in real-time. Elasticsearch is a search and analytics engine. Logstash is a server‑side data processing pipeline that ingests data from multiple sources simultaneously, transforms it, and then sends it to a “stash” like Elasticsearch. Kibana lets us visualize this data with charts and graphs.


How do they work together?
--------------------------
Based on the below diagram (Image A), when the Orchestrator Service makes a HTTP call on the service `/order/{orderId}`, the call is intercepted by Sleuth and it adds the necessary tags to the request headers. After the Orchestrator Service receives the HTTP response, the data is sent asynchronously to Zipkin to prevent delays or failures relating to the tracing system from delaying or breaking the flow.
Sleuth adds two types of IDs to the log file, one called a trace ID and the other called a span ID. The span ID represents a basic unit of work, for example sending an HTTP request. The trace ID contains a set of span IDs, forming a tree-like structure. The trace ID will remain the same as one microservice calls the next.

The logs are published directly to Logstash in this example for convenience, but we can also use Beats. Beats is a simple data shipper that either sits on servers or on containers, that listen to log file locations and ship them to either Logstash for transformation or Elasticsearch.

To enable Sleuth, Zipkin and ELK stack, we need to make the below changes on all core microservices.

    First change is the pom.xml, where we add the cloud-starter dependencies for both Sleuth and Zipkin, also the logback dependencies needed for logstash.

    The second change is to add the URL, in the application.properties for spring to publish data to Zipkin.

    The final change is the logback.xml, to publish the logs to LogStash. The appender publishes all the logs to Logstash running on port 5044, using an Async TCP Appender / File Appender. Again as mentioned above Beats can be used to ship logs to Logstash too.

All the services that need to use the Distributed Tracing feature, will need the above three changes / additions.

use Zipkin to analyze latency in the service calls. Also Sleuth can help us creating the metadata and pass it to Zipkin.



ELK stack configuration
-----------------------
All these three tools are based on JVM and before start installing them, please verify that JDK has been properly configured. Check that standard JDK 1.8 installation, JAVA_HOME and PATH set up is already done.

1. Elasticsearch
    Download latest version of Elasticsearch from this download page and unzip it any folder.
    Run bin\elasticsearch.bat from command prompt.
    By default, it would start at http://localhost:9200
2. Kibana
    Download the latest distribution from download page and unzip into any folder.
    Open config/kibana.yml in an editor and set elasticsearch.url to point at your Elasticsearch instance. In our case as we will use the local instance just uncomment elasticsearch.url: "http://localhost:9200"
    Run bin\kibana.bat from command prompt.
    Once started successfully, Kibana will start on default port 5601 and Kibana UI will be available at http://localhost:5601
3. Logstash
    Download the latest distribution from download page and unzip into any folder.
    Create one file logstash.conf as per configuration instructions. We will again come to this point during actual demo time for exact configuration.
  
 logstash.conf - FILE

    input {
    file {
        path => "B:/logback/*.log"		
		mode => "read"
		ignore_older => "37 d"
        codec => "json"
        type => "logback"
    }
}
 
output {
    if [type]=="logback" {
         elasticsearch {
             hosts => [ "localhost:9200" ]
             index => "logback-%{+YYYY.MM.dd}"
        }
    }
}
    
Now run logstash -f logstash.conf to start logstash and make sure system path variable is set for logstash bin folder

show all indexes
---------------
http://localhost:9200/_cat/indices?v

other useful commands
----------------

Show cluster health | pretty formated

curl -XGET 'IP-OF-ELASTIC-SERVER:9200/_cluster/health?pretty'

Show cluster settings | pretty formated

curl -XGET 'IP-OF-ELASTIC-SERVER:9200/_cluster/settings?pretty'

Show all indexes

curl -XGET 'IP-OF-ELASTIC-SERVER:9200/_cat/indices?v'

Show all indexes which start with logsta*

curl -XGET 'IP-OF-ELASTIC-SERVER:9200/_cat/indices/logsta*?v&s=index'

Show all indexes/shards in state unassigned

curl -XGET -H 'Content-Type: application/json' http://IP-OF-ELASTIC-SERVER:9200/_cat/shards | grep UNASSIGNED

Unlock all indexes manually

curl -XPUT -H "Content-Type: application/json" http://IP-OF-ELASTIC-SERVER:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'
Get default logstash template from elasticsearch | pretty formated | stored in temp directory

curl -XGET 'IP-OF-ELASTIC-SERVER:9200/_template/logstash?pretty' > /tmp/logstash-template.json

Upload modified logstash template to elasticsearch from directory where logstash-template.json exists | hint dont forget to remove the logstash parameter in the file

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/_template/logstash' -d "@logstash-template.json"

Run logstash from commandline with all config files in debug mode

/usr/share/logstash/bin/logstash --path.settings /etc/logstash/ -r -f "/etc/logstash/conf.d/*" --log.level debug

Run logstash from commandline with specific config file in debug mode

/usr/share/logstash/bin/logstash --path.settings /etc/logstash/ -r -f "/etc/logstash/conf.d/00-test.config" --log.level debug

Change the number of replicas at one existing index

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/.alert/_settings' -d '{ "index" : {"number_of_replicas" : 0}}'

Change the number of replicas at multiple existing indexes | sets number_of_replicas:0 to all cisco indexes from the year 2018

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/logstash-cisco-2018.*.*/_settings' -d '{ "index" : {"number_of_replicas" : 0}}'

 Delete all indexes from a year

curl -XDELETE 'IP-OF-ELASTIC-SERVER:9200/logstash-cisco-2018.*.*'

Create an index manuall from cli via curl | name of index is aa_test_index | number_of_shards:1 and number_of_replicas:0

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/aa_test_index?pretty' -d '{"settings" : {"index" : {"number_of_shards" : 1,"number_of_replicas" : 0 }}}'

Create an index manuall from cli by using logstash | name of index is aa_test_index_from_cli

/usr/share/logstash/bin/logstash --path.settings "/etc/logstash/" -e 'input { stdin { } } output { elasticsearch { hosts => "IP-OF-ELASTIC-SERVER:9200" index => "aa_test_index_from_cli" } }'

Logstash config test

/usr/share/logstash/bin/logstash --path.settings "/etc/logstash/" --config.test_and_exit -f /etc/logstash/conf.d/

Set existing index to read-only | ! needed for shrinking shards ! | this command sets all shards from december 2018 to read-only

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/logstash-cisco-2018.12.*/_settings' -d '{"index.blocks.write": true}'

Shrink existing shards | shrinks the data from existing index "logstash-cisco-2018.12.24" to the new "smaller index" logstash-cisco-2018.12.24.shrinked | copy all index settings to new one | number_of_shards:1 and number_of_replicas:0 | using best_compression | disable read-only mode

curl -POST -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/logstash-cisco-2018.12.24/_shrink/logstash-cisco-2018.12.24-shrinked?copy_settings=true' -d '{ "settings": { "index.number_of_replicas": 0, "index.number_of_shards": 1, "index.codec": "best_compression", "index.routing.allocation.require._name": null, "index.blocks.write": null }, "aliases": { "my_search_indices": {}  }}'

Increase the elasticsearch shard limit from 1000 to 5000 for each node

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/_cluster/settings' -d '{ "persistent" : {"cluster.max_shards_per_node" : 5000}}'

Set the Cluster Metadata Administrator Email address

curl -XPUT -H 'Content-Type: application/json' 'IP-OF-ELASTIC-SERVER:9200/_cluster/settings' -d '{ "persistent" : {"cluster.metadata.administrator" : "
admin@yourdomain.c
om"}}'


ELK stack is not up and running. Now we need to create few microservices and point logstash to the API log path.
