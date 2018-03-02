# Visualizing Data with Kuzzle Analytics - part 3

![full-dashboard](img/bq-full-dashboard.png)

Last time in [Visualizing Data with Kuzzle Analytics - part 2] we have created a dashboard with Kibana and the probes system showing us detected motion in our office over a lapse of time. This time we will configure Kuzzle with BiqQuery and Data-studio. With the intention of doing a dashboard to monitor and visualize our data with Kuzzle and these tools. As you know Data-studio needs to manage data source and for that we will use BigQuery as data warehouse. 

We have to add the KDC-bigquery-connector plugin to our stack and for change, we will visualize data about light level collected by our IoT sensor.

![kdc-schema](img/kdc-schema3.png)

## 1- Docker-compose and configuration files

First we need to add the KDC-bigquery-connector plugin to our KDC stack in the ```docker-compose.yml``` file.

```yaml
version: '2'

services:
  kuzzle:
    image: kuzzleio/kuzzle
    ports:
      - "7512:7512"
    cap_add:
      - SYS_PTRACE
    depends_on:
      - redis
      - elasticsearch
    environment:
      - kuzzle_services__db__client__host=http://elasticsearch:9200
      - kuzzle_services__internalCache__node__host=redis
      - kuzzle_services__memoryStorage__node__host=redis
      - NODE_ENV=production
    volumes:
      - "./plugins/kuzzle-enterprise-probe-listener/:/var/app/plugins/enabled/kuzzle-enterprise-probe-listener/"
      - "./config/kuzzlerc:/etc/kuzzlerc"

  redis:
    image: redis:3.2

  elasticsearch:
    image: kuzzleio/elasticsearch:5.4.1
    environment:
      - cluster.name=kuzzle
      - xpack.security.enabled=false
      - xpack.monitoring.enabled=false
      - xpack.graph.enabled=false
      - xpack.watcher.enabled=false
      - http.host=0.0.0.0
      - transport.host=0.0.0.0
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
  
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
      - "./plugins/kuzzle-enterprise-probe:/var/app/plugins/enabled/kuzzle-enterprise-probe/"
      - "./plugins/kdc-bigquery-connector:/var/app/plugins/enabled/kdc-bigquery-connector/"
      - "./config/kdcrc:/etc/kuzzlerc"
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

If you look at the ```kdc-kuzzle``` service you will see we have added a new volume to mount our second plugin.

Like I said, this time we want visualize light level in Data-studio so we need to update our configuration files to change our probe system.

Here an example of document sends by our light level sensor :

```JSON
{
  "device_id": "light_lvl_00000000c9591b74",
  "state": {
    "level": 41.47135416666667
  },
  "partial_state": false,
  "device_type": "light-sensor"
}
```

First we have to modify the ```kuzzlerc``` configuration file to add probe watcher that can collect the ```state.level``` value on each document created corresponding to the filter we apply here :

```json
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
              "device_type": "light-sensor"
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

Now, the KDC stack configuration into the ```kdcrc``` file :

```json
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
              "device_type": "light-sensor"
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

To the following, we add the BigQuery plugins configuration in the ```plugins``` JSON object on the same file. We also need to add our configured probes to this plugin. But this time with the schemas of our BigQuery tables.
if you add ```"timestamp":"true"```, the plugin will automatically add the timestamp for each document collected.

```JSON
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
```

We can now run our two stack with docker in a terminal :

```
$ docker-compose up
```

After that, everytime a document is created in the primary stack and matched with our probe filter, the ```state.level``` field on this document will be send to BigQuery on the appropriate table. 


## 2- BigQuery and Data-studio

Assuming you have already configured your google account for using BigQuery (if you want to know more about how to setup BigQuery you can read this [tutorial](https://docs.exploratory.io/import/google-bigquery.html)). Now, you have to create a new project in BigQuery [console](https://bigquery.cloud.google.com/). And, after that create a new dataset.

![BigQuery-create-project](img/bigquery1.png)
![BigQuery-create-dataset](img/bigquery2.png)

it's not necessary to create tables manually in the BigQuery admin console, the Kuzzle BigQuery connector plugin will create them automatically depending on the schema you give in the configuration file.

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

You will see a summary of your table (it's possible to change the type of your fields here). We need to tell to Data-studio what kind of aggregation we want for the ```level``` field. By default it's set to "Sum" but we want an "Average" aggregation so, pick this choice.
Also we want the chart is indexing by hours. Choose "Date Hour" for the type of the ```timestamp``` field.

![BigQuery-config-data](img/bigquery5.png)

Click on the "ADD TO REPORT" button and validate your choice in the modal window.

It's time to add our first chart to this report ! Click on the time series button on the top side menu ![BigQuery-timeseries-button](img/bigquery-timeseries-button.png). Draw a frame in the blank zone. Data-studio will automatically set the data source, the dimension and the metric fields.

![BigQuery-config-chart](img/bigquery4.png)

That's all ! We have our chart representing light level over time, sorted hour by hour.
Of course you can customize your chart, Data-studio give a large panel of options to render your graph like another one.

![BigQuery-chart](img/bigquery3.png)

It's just a short example of what we can do with Data-studio and BigQuery. But that demonstrate how we can use the Kuzzle enterprise probes system to easily dump data to another data warehouse in an asynchrone way without interfere with a production stack.

