# Creating analytics visualizations with Kuzzle - part 1


Having tons of data is awesome but visualizing them and getting them to tell you something meaningful is even cooler. That's why I've decided to write a series of 3 articles on how to build a monitoring dashboard using Kuzzle. The first article is about configuring Kuzzle and his analytics ecosystem. The second one will deal with visualizations and Kibana. And to finish, on the third one we will use Google Data Studio for visualize our datas.  

![Kuzzle](img/kuzzle.png)

[Kuzzle](http://kuzzle.io/) is an open-source self-hostable backend solution that can power web, mobile and IoT applications. It allows you to drastically reduce your development time and take advantage of builtin features like real-time data management and geofencing, among others.

Our goal is to build a beautiful dashboard based on the IoT data collected from a custom multi-sensor device that detects luminosity, humidity, temperature and motion. Kuzzle built this device to demo Kuzzle's core features, you may have seen it at CES. The datas we will use was acquired from the device over a period of 5 days, where it sat in our office and captured all our kooky antics. The device used MQTT protocol to communicate with an on-premise Kuzzle Backend instance, which stored any sensor data it received. 

A common need when using Kuzzle in production, is to have insights about it and perform analytics on events occurring during an instance's lifecycle. A common supply for this need is to allow attaching probes to the production instance and send the events somewhere. At Kuzzle, we use Kuzzle to monitor Kuzzle.

![kdc-schema](img/kdc-schema2.png) 

KDC (for Kuzzle Data Collector) is plugged to the primary stack by the probes plugin and wait for configured events occurs. 
When dones we just have to exploit these datas.

The probes plugin can manage 3 kinds of probes : 
 - ```monitor``` probes are basic event counters, used to monitor Kuzzle activity.
 - ```counter``` probes aggregate multiple fired events into a single measurement counter.
 - ```watcher``` probes watch documents and messages activity, counting them or retrieving part of their content.

In a nuthsell we want every time a detected motion is added in our primary Kuzzle stack, the probes catch it and send it to the KDC stack.

At the end of these tutorials, we will have created a dashboard that shows the office's activity (detected motions) over time.

## 1- Preparing your files

Let's get started. First of all, you should install docker and docker-compose. If it's not already done you can check these tutorials : [Install Docker](https://docs.docker.com/install/) and [Install Docker Compose](https://docs.docker.com/compose/install/).

Like said previously we need 2 Kuzzle stacks and a bunch of plugins to be configured in harmony to gets one big stack ready to blow up our datas !

Each stack need a probes plugins and a configuration file to communicate together. An overview of our files organisation will look like that :

![Kuzzle](img/files.png)

For the first step we have to write a Docker Compose file that will launch our production stack. Create a file called `docker-compose.yml`. 

Now add the Kuzzle service in this file. Note that we use environment variables to configure the Elasticsearch connection. We also expose port 7512 which will allow clients to communicate with the service through your docker host.

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
```
Likewise we use volumes to mount the plugin and his configuration file into the Kuzzle service.

Next we need to add Redis (used internally by Kuzzle) to the docker-compose.yml file. Add the following config:

```yaml
  redis:
    image: redis:3.2
```
We will also need Elasticsearch to store the data collected from our custom multi-sensor device: 

```yaml
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
```

We have now configured the full production Kuzzle stack !

Based on the same schema, now we can add the KDC stack on our docker-compose file :

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
      - "./plugins/kuzzle-enterprise-probe:/var/app/plugins/enabled/kuzzle-enterprise-probe/"
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
Kuzzle stores data in document collections within an index. Below is an example document that represents a single motion capture:


```json
{
    "_index":"iot",
    "_type":"device-state",
    "_id":"AWFSDI8RAUgq-wTF-Lwg",
    "_score":1,
    "_source": {
        "device_id":"motion_00000000c9591b74",
        "device_type":"motion-sensor",
        "partial_state":false,
        "state": {
            "motion":true,
        },
        "_kuzzle_info": {
            "author":"iot-device",
            "createdAt":1517834432225,
            "updatedAt":null,
            "updater":null,
            "active":true,
            "deletedAt":null
        }
    }
}
```

Notice that this document is stored in a collection called `device-state`, in an index called `iot`. The data we are interested in is stored in the `state` object, while metadata such as timestamps are stored in the `_kuzzle_info` document. 

At this time, we need to create the first configuration file to tell the kuzzle-enterprise-probe-listener plugin what measure it have to collect. Name it ```kuzzlerc``` and add these lines :

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
              "device_type": "motion-sensor"
            }
          },
          "action": "create",
          "collects": [
            "state.motion"
          ]
        }
      }
    }
  }
}
```

Currently we have correctly configurate our production instance and right now we have to do the same for the KDC instance.

Create another configuration file, call it ```kdcrc``` :

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
              "device_type": "motion-sensor"
            }
          },
          "action": "create",
          "collects": [
            "state.motion"
          ]
        }
      }
    }
  }
}
```

Here we are! we have our full system configured and ready to perform analytics with our IoT datas.
Every time our sensor detect a gesture, a document is send by the plugins to the KDC instance.

Next time we will see how to take advantage and visualize all of these datas with Kibana.




