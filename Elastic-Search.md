## ELK 7 on Docker
This lab we'll setup a unsecure ELk stack, using Elasticsearch 7.17

The ELK stack has 3 main components:

* Logstash (log processing)
* ElasticSearch (database)
* Kibana (dashboard)

And has several add ons, including

* x-pack, providing security, alerting, monitoring, reporting, machine learning, and many other capabilities
* graph
* beats, that allows import to elasticsearch to many different ways.

Kibana orginally started as a logging platform with uses Elasticsearch, and later added metrics, and Grafana became a fork inorder to do metrics. Now both systems do both, however you can do more Logging with Kibana, and more Metrics wit Grafana

### Nodes, shards and indexs
In this lab, we'll create just the one Node, and an index. An index can cross multiple Nodes, and the intersection of each is called a Shard. However you can have multiple same index shards on the same node (check this)

You can primary and replicate nodes (P0, R0) to protect data, it also improves search speed.

Data is stored as json objects in an index,

Nodes can have different Roles, but in this lab we will not concern ourselfs with this.

### Access control
Kibana dashboards are open to the public, but can be locked down with an extension (X=Pack) Elasticsearch/Kibana v8 comes locked down by default, checkout the version 8 lab for more details.

### Querying
Since Kibana uses Elastricsearch, you'll be using Lucene or Kuery syntax

### Logstash
Is the log processing engine, that is configured through a yaml file, with 3 made sections

* inputs (defines the type of input to recieve)
* next is filters which defines what tp do with each log message
* outputs defines were to send the output.

### More resources
An excellent introduction to the ELK stack can be found here, it is several hours long. https://www.youtube.com/watch?v=gS_nHTWZEJ8 and it's associated Github page: https://github.com/LisaHJung/Beginners-Crash-Course-to-Elastic-Stack-Series-Table-of-Contents

### Initial Setup

- docker-compose.yaml
```yaml
version: '3.6'
services:
  Elasticsearch:
    image: elasticsearch:7.17.4 # use 7.16.2
    container_name: elasticsearch
    restart: always
    volumes:
    - elastic_data:/usr/share/elasticsearch/data/
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      discovery.type: single-node
    ports:
    - '9200:9200'
    - '9300:9300'
    networks:
      - elk

  Logstash:
    image: logstash:7.17.4
    container_name: logstash
    restart: always
    volumes:
    - ./logstash/:/logstash_dir
    - ./:/root
    command: logstash -f /logstash_dir/logstash.conf
    depends_on:
      - Elasticsearch
    ports:
    - '9600:9600'
    - '5000:5000/tcp'
    - '5000:5000/udp'
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk

  Kibana:
    image: kibana:7.17.4
    container_name: kibana
    restart: always
    ports:
    - '5601:5601'
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
    depends_on:
      - Elasticsearch
    networks:
      - elk
volumes:
  elastic_data: {}

networks:
  elk:
```

Boot up the ELK stack:
```bash
docker-compose up -d
```
In another tab, lets setup some tools/config
```bash
apt update
apt install -y net-tools jq tree
```
Config APT to download various beats:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt-get install apt-transport-https

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update
```

Check the services are running in Docker:
run
```bash
docker ps 
```
to review the ports
* note ES is on port 9200
* Kibana is on 5601
* Logstash is on 5000

Once the Docker-compose has completed, wait a few minutes for the elasticsearch(ES) server to come up, you will get a json response from:
```bash
curl http://localhost:9200

curl http://localhost:9200/_cluster/health?pretty
```
You should now have access to the portal at:

https://localhost:5601

Once in the web portal, select 'explore on my own'

open kibana web, > hamburger > Managment > Dev Tools

lets check the health, paste on line 7 
```text
GET _cluster/health
```
```yaml
{
  "cluster_name" : "docker-cluster",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 9,
  "active_shards" : 9,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 1,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 90.0
}
```

and then the green triangle to run that query.

* take note of the status, number of nodes, and shards

Gather information about node/nodes:
```text
GET _nodes/stats
```

### Explore on your own
If you wish to explore Kibana on your own, you can head to the Kibana home page, and 'try sample data'


## Basic Operation
### check up and running

Lets start generating some logs into logstash/ES:

in another tab start the log generator:

chmod +x sysloggen.sh

- sysloggen.sh
```bash
#!/bin/bash
# Path to netcat
NC="/bin/nc"
# Where are we sending messages from / to?
#ORIG_IP="192.168.190.11"
SOURCES=("192.168.190.2" "192.168.190.3" "192.168.190.4" "192.168.190.5" "192.168.190.6" "192.168.190.7")
#Destination network
DEST_IP="localhost"
# List of messages.
MESSAGES=("Error Event" "Warning Event" "Info Event")
# How long to wait in between sending messages.
SLEEP_SECS=1
# How many message to send at a time.
COUNT=1

FACILITIES=("kernel" "user" "mail" "system" "security" "syslog" "lpd" "nntp" "uucp" "time" "ftpd" "ntpd" "l
ogaudit")
LEVELS=("emergency" "alert" "critical" "error" "warning" "notice" "info" "debug")
PRIORITIES=(0 1 2 3 4 5 6 7)

while [ 1 ]
do
        for i in $(seq 1 $COUNT)
        do
                # Picks a random syslog message from the list.
                RANDOM_MESSAGE=${MESSAGES[$RANDOM % ${#MESSAGES[@]} ]}
                PRIORITY=${PRIORITIES[$RANDOM % ${#PRIORITIES[@]} ]}
                SOURCE=${SOURCES[$RANDOM % ${#SOURCES[@]} ]}
                FACILITY=${FACILITIES[$RANDOM % ${#FACILITIES[@]} ]}
                LEVEL=${LEVELS[$RANDOM % ${#LEVELS[@]} ]}

                        $NC $DEST_IP -u 5000 -w 1 <<< "<$PRIORITY>`env LANG=us_US.UTF-8 date "+%b %d %H:%M:
%S"` $SOURCE [$FACILITY.$LEVEL] service: $RANDOM_MESSAGE"
                        echo $NC $DEST_IP -u 5000 -w 1  "<$PRIORITY>`env LANG=us_US.UTF-8 date "+%b %d %H:%
M:%S"` $SOURCE service: $RANDOM_MESSAGE"

        done
        sleep $SLEEP_SECS
done
```
```bash
bash sysloggen.sh
```

### Indexing

With data flowing into logstash, and then into ES, the records are being given a datafield of an index, and then being flowed into that index.

Open the ES GUI, and goto Management > Stack Management. And then Data -> Index Management. And you'll see the index logstash-*

Note that the health is yellow - this is due to ES considers an index healthy when it includes a replica. Since this ES only has one node, it won't have a replica.

### Index Pattern
Next we need to have Kibana reconize that index. Still under management, goto Kibana -> Index Patterns.

Click on index patterns, and use the term 'logstash*' and Timestamp field of @timestamp.

You will then be able to use the Analytics -> Discover page to search that index pattern.

! note the following is based on the docker config, for native elk stack config is usually under /etc//

### Logstash
In our docker-compose we have mapped the logstash config file to the local logstash folder
```bash
cat logstash/logstash.conf
input {
    file {
      path => "/root/temp/inlog.log"
    }
    tcp {
      port => 5000
      type => syslog
    }
    udp {
      port => 5000
      type => syslog
    }
}
output {
    elasticsearch {
      hosts => ["http://elasticsearch:9200"]
    }
    stdout { }
}
```

Show the logs of the logstash container
```bash
docker-compose logs -f Logstash
```
(note that the service starts with a capital letter: Logstash)

Look at the options for logstash:
```bash
docker exec -it logstash bin/logstash --help
```
View the available binaries:
```bash
docker exec -it logstash ls /usr/share/logstash/bin/
```
query the logstash api
```bash
curl -XGET 'localhost:9600/?pretty'

curl -XGET 'localhost:9600/_node/stats/'

curl -XGET 'localhost:9600/_node/stats/jvm?pretty'
```
Check out more about the API at (https://www.elastic.co/guide/en/logstash/current/monitoring-logstash.html)

for more on configeration see: https://www.elastic.co/guide/en/logstash/current/configuration.html

https://www.elastic.co/guide/en/logstash/current/advanced-pipeline.html

### ElasticSearch
Show the logs of the ES container
```bash
docker-compose logs -f Elasticsearch
```
Show the elastic config:
```bash
docker exec -it elasticsearch cat /usr/share/elasticsearch/config/elasticsearch.yml
```
View the available binaries:
```bash
docker exec -it elasticsearch ls /usr/share/elasticsearch/bin/
```
and the help
```bash
docker exec -it elasticsearch bin/elasticsearch -h
```
### Kibana
Show the logs of the kibana container
```bash
docker-compose logs -f Kibana
```
Show the kibana config:
```bash
docker exec -it kibana cat /usr/share/kibana/config/kibana.yml
```
View the available binaries:
```bash
docker exec -it kibana ls /usr/share/kibana/bin/

docker exec -it kibana bin/kibana -h
```
==========

### CRUD Operations

#### Basic Setup

Using the same 'Dev Tool' in the web GUI

create an index:
```text
PUT favorite_candy
```
the response "acknowledged" : true, shows the operation was successful

and you should see the http response code, and the time taken above that section in green and white.

to list all indices:
```text
GET /_cat/indices
```
in kibana Home>stack mgmnt > Data > Index Mmgmnt

Now add a document to an index:
```text
POST favorite_candy/_doc
{
  "first_name": "Lisa",
  "candy": "Sour Skittles"
}
```
Note the version number and _id number.

if you want to do a document with a specific ID (1)
```text
PUT favorite_candy/_doc/1
{
  "first_name": "John",
  "candy": "Starburst"
}
```
add the following documents:
```text
PUT favorite_candy/_doc/2
{
  "first_name": "Rachel",
  "candy": "Rolos"
}
PUT favorite_candy/_doc/3
{
  "first_name": "Tom",
  "candy": "Sweet Tarts"
}
```
Lets read one of the docs
```text
GET favorite_candy/_doc/1
```
you can see the _source in the output is the document
Now if you just to resend the same id, it will over write and give you an incremented version number
```text
PUT favorite_candy/_doc/1
{
  "first_name": "Sally",
  "candy": "Snickers"
}
```
If you want to lock a document so it can't be updated, use:
```text
PUT favorite_candy/_create/1
{
  "first_name": "Finn",
  "candy": "Jolly Ranchers"
}
```
Now this will fail since a document with that id already exsists

Update:

Now to update a specific document use:
```text
POST favorite_candy/_update/1
{
  "doc": {
    "candy": "M&M's"
  }
}
```
note the id number,
the 'doc'
the fields you wish updated
that the response shows an incremented version number

Delete:
```text
DELETE favorite_candy/_doc/1
```
to delete a complete index
```text
DELETE favorite_candy
```

### CRUD through http
#### send data through http api
type 'curl -XPUT localhost:9200/movies/ -d'  
note the quotes in the commands that encapsulated the json data

then { ctrl-v tab, and complete as below. Notice the single quotes enclosing the text

{
  "mappings": {
    "properties": {
      "year": { "type": "date" }
    }
  }
}'
ABOVE ISN'T WORKING,, need to be in the ~/bin dir and run ./curl

curl -H "Content-Type: application/json" -XPUT localhost:9200/movies -d '
{ "mappings": { "properties": { "year": { "type": "date" } } } }'

curl -XGET localhost:9200/movies/_mapping

curl -XPUT localhost:9200/movies

wget http://media.sundog-soft.com/es8/movies.json

cat movies.json

note the json file already has the 'create', 'index' and 'id' and that a year field is present, and we have told ES to treat that as a date.

curl -H "Content-Type: application/json" -XPUT localhost:9200/_bulk?pretty --data-binary @movies.json

curl -XGET localhost:9200/movies/_search?pretty

### Lets use faker to send logs

https://github.com/thombashi/elasticsearch-faker
```bash
pip install elasticsearch-faker
```
### using logstash to send a csv

input {
    file {
        path => "path"
        start_position => "beginning"
        sincedb_path = > "/dev/null" # windows "NULL"
    }
}
filter {
    csv {
        separator => ","
        columns => ["col1",col2"]
    }
}
output {
    elasticsearch {
        hosts => "http://localhost:9200"
        index => "new_index_name"
    }
}

then add an index pattern into kibana

## BEATS
lets check the Elastic version
```bash
curl http://localhost:9200
```
and install the same Beats version:

WIP move to step 1 the key update
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt-get install apt-transport-https

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update
```
#### main cmds/setting
test config # to test the yml config test output # to test connection to ES setup # to setup dashboard etc in es/kibana stack

#### metricbeat
```bash
apt install metricbeat=7.17.4
```
WIP update docker to the same

WIP metricbeat setup
```bash
sudo apt-get install metricbeat=7.17.4

curl http://localhost:9200

sudo systemctl enable metricbeat

sudo update-rc.d metricbeat defaults 95 10
```
create some load:
```bash
apt install stress

nproc

stress --cpu 1 --timeout 120

stree --vm 5 --timeout 180

```
