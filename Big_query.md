# Kuzzle, BiqQuery and Data-studio dashboard

Today we want to configure Kuzzle with BiqQuery and Data-studio with the intention of doing a dashboard to monitor our datas with these tools. As you know Data-studio needs to manage data source and for that we will use BigQuery as data warehouse.

A common need when using Kuzzle in production, is to have insights about it and perform analytics on events occurring during an instance's lifecycle. A common supply for this need is to allow attaching probes to the production instance and send the events somewhere. At Kuzzle, we use Kuzzle to monitor Kuzzle.

![kdc-schema](img/kdc-schema.png) 

KDC (for Kuzzle Data Collector) is plugged to the primary stack by the probes plugin and wait for configured events occurs. 
When dones, it sends datas to BigQuery through another plugin. And finally we just have to put these datas in a data source and use them in Data-studio.

The probes plugin can manage 3 kinds of probes : 
 - ```monitor``` probes are basic event counters, used to monitor Kuzzle activity.
 - ```counter``` probes aggregate multiple fired events into a single measurement counter.
 - ```watcher``` probes watch documents and messages activity, counting them or retrieving part of their content.

We will use the last one in this article to send data to our dashboard.

In a precedent article, I have explained that we have an IoT board installed in our office that sends data about some weather measures and detected motions. Today for training purpose, imagine we want monitor only ligth level and view average values sorted by hours in an awesome chart in Data-studio.

In a nuthsell we want every time a light level measure is added in our primary Kuzzle stack, the probes catch it and send it to the BiqQuery appropriate table and feed our graph with these datas.

## 1- Docker-compose and configuration files

Like said previously we need 2 Kuzzle stacks and a bunch of plugins to be configured in harmony to gets one big stack ready to blow up our datas !

For orchestrating all of that, we need to write a docker-compose file. First, the primary stack

```yaml
version: '2'

services:
  kuzzle:
    image: kuzzleio/kuzzle
    ports:
      - "7512:7512"
    volumes:
      - "./../kuzzle-enterprise-probe-listener/:/var/app/plugins/enabled/kuzzle-enterprise-probe-listener/"
      - "./docker-compose/kuzzlerc:/etc/kuzzlerc"
    depends_on:
      - redis
      - elasticsearch
    environment:
      - kuzzle_services__db__client__host=http://elasticsearch:9200
      - kuzzle_services__internalCache__node__host=redis
      - kuzzle_services__memoryStorage__node__host=redis
    

  redis:
    image: redis:3.2

  elasticsearch:
    image: kuzzleio/elasticsearch:5.4.1
    environment:
      - cluster.name=kuzzle
      # disable xpack
      - xpack.security.enabled=false
      - xpack.monitoring.enabled=false
      - xpack.graph.enabled=false
      - xpack.watcher.enabled=false
```
Here we have the plugin "Kuzzle-enterprise-probe-listener" mounted in a volume together with a ```kuzzlerc``` file for configuring the plugin.

After that we can add the KDC stack to our docker-compose :

```yaml
  kdc-kuzzle:
    image: kuzzleio/kuzzle
    ports:
      - "7515:7512"
      - "9229:9229"
    cap_add:
      - SYS_PTRACE
    depends_on:
      - kdc-redis
      - kdc-elasticsearch
    volumes:
      - "./:/var/app/plugins/enabled/kuzzle-enterprise-probe/"
      - "./../kdc-bigquery-connector/:/var/app/plugins/enabled/kdc-bigquery-connector/"
      - "./docker-compose/kdcrc:/etc/kuzzlerc"
    environment:
      - kuzzle_services__db__client__host=http://kdc-elasticsearch:9200
      - kuzzle_services__internalCache__node__host=kdc-redis
      - kuzzle_services__memoryStorage__node__host=kdc-redis
      - NODE_ENV=production

  kdc-redis:
    image: redis:3.2

  kdc-elasticsearch:
    image: kuzzleio/elasticsearch:5.4.1
    environment:
      - cluster.name=kuzzle
      # disable xpack
      - xpack.security.enabled=false
      - xpack.monitoring.enabled=false
      - xpack.graph.enabled=false
      - xpack.watcher.enabled=false

```

As in the primary stack we mount volumes in Kuzzle with the "Kuzzle-enterprise-probe" and the "kdc-bigquery-connector" plugins with proper configuration file.


So, it's time to see these configurations files. The first one will be the Kuzzle listener stack :

```JSON
{
  "plugins": {
   "kuzzle-enterprise-probe-listener": {
      "threads": 1,
      "probes": {
        "probe_watcher_1": {
          "type":"watcher",
          "index": "iot",
          "collection": "device-state",
          "filter": {
            "equals": {
              "device_type": "light_sensor"
            }
          },
          "action": "create",
          "collects": [
            "state.level"
          ]
        }
      }
    }
  }
}
```

It describe all the probes plugged to the listenner, here only one for light level. We use a watcher probes waiting for document creation in the ```iot``` index and in ```device-state``` collection. We also add a filter to catch only documents with field "device_type" equals to "light_sensor".
When this event happen, the probes will only collect datas that interresting us (```state.level```) and send them to the KDC stack.

Now, the KDC stack configuration :

```JSON
{
  "plugins": {
    "kuzzle-enterprise-probe": {
      "storageIndex": "iot",
      "probes": {
        "probe_watcher_1": {
          "type":"watcher",
          "index": "iot",
          "collection": "device-state",
          "filter": {
            "equals": {
              "device_type": "light_sensor"
            }
          },
          "action": "create",
          "collects": [
            "state.level"
          ]
        }
      }
    },
    "kdc-bigquery-connector": {
      "projectId": "iot-monitor-192608",
      "dataSet": "probes_iot",
      "credentials": {
        //[...] put your google's credentials here
      },
      "probes": {
        "probe_watcher_1": {
          "timestamp": "true",
          "type": "watcher",
          "tableName": "light_sensor",
          "schema": {
            "fields": [
              {
                "name": "level",
                "type": "FLOAT",
                "mode": "NULLABLE"
              }
            ]
          }
        }
      }
    }
  }
}
```

As you can see, the probes plugin configuration parts is exactly the same that for configuring the plugin on the listenner stack.
To the following, we add the BigQuery plugins configuration. We also need to add our configured probes to this plugin. But this time with the schemas of our BigQuery tables.
I add ```"timestamp":"true"```, with that option the plugin will automatically add the timestamp for each document collected.

We can now run our stack with docker in a terminal :

```
$ docker-compose up
```

After that, everytime a document is created in the primary stack and matched with our probe filter, the ```state.level``` field on this document will be send to BigQuery on the appropriate table. 


## 2- BigQuery and Data-studio

Assuming you have already configured your google account for using BigQuery (if you want to know more about how to setup BigQuery you can read this [tutorial](https://docs.exploratory.io/import/google-bigquery.html)). Now, you have to create a new project in BigQuery [console](https://bigquery.cloud.google.com/). And, after that create a new dataset.

![BigQuery-create-project](img/bigquery1.png)
![BigQuery-create-dataset](img/bigquery2.png)

You don't have to manually create tables, the Kuzzle BigQuery connector plugin will create them automatically depending on the schema you give in the configuration file.

```JSON
    "schema": {
        "fields": [
            {
                "name": "level",
                "type": "FLOAT",
                "mode": "NULLABLE"
            }
        ]
    }
```

If you add ```"timestamp":"true"``` in the configuration file, the plugin also add a timestamp field to the BigQuery table.

We can go now to the Data-studio console by browsing https://datastudio.google.com/. Choose a blank template to start a new report.

Data-studio will ask you to add a data-source. Choose "CREATE NEW DATA SOURCE" and validate by clicking on the "ADD TO REPORT" button. 
Now choose BigQuery on the list on left side. Find your project we just created and choose the data set. And finnaly choose the table you want add to Data-studio. Validate by clicking on the "CONNECT" button on the top right corner.

You will see a summary of your table (it's possible to change the type of your fields here). We need to tell to Data-studio what kind of aggregation we want for the ```level``` field. By default it's set to "sum" but we want an "Average" aggregation so, pick this choice.
Also we want the chart is indexing by hours. Choose "Date Hour" for the type of the ```timestamp``` field.

![BigQuery-config-data](img/bigquery5.png)

Click on the "ADD TO REPORT" button and validate your choice in the modal window.

It's time to add our first chart to this report ! Click on the time series button on the top side menu ![BigQuery-timeseries-button](img/bigquery-timeseries-button.png). Draw a frame in the blank zone. Data-studio will automatically set the data source, the dimension and the metric fields.

![BigQuery-config-chart](img/bigquery4.png)

That's all ! We have our chart representing light level over time, sorted hour by hour.
Of course you can customize your chart, Data-studio give a large panel of options to render your graph like another.

![BigQuery-chart](img/bigquery3.png)

