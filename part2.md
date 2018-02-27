# Creating Visualizations with Kuzzle Backend and Kibana

![Kibana-full-dashboard](img/full-dashboard.png)

Having tons of data is awesome but visualizing them and getting them to tell you something meaningful is even cooler. That's why I've decided to write this tutorial on how to build a monitoring dashboard using Kuzzle Backend and Kibana.

![Kuzzle Backend](img/kuzzle.png) ![Kibana](img/kibana.png)

[Kuzzle Backend](http://kuzzle.io/) is an open-source self-hostable backend solution that can power web, mobile and IoT applications. It allows you to drastically reduce your development time and take advantage of builtin features like real-time data management and geofencing, among others.

Today, our goal is to build a beautiful dashboard based on the IoT data collected from a custom multi-sensor device that detects luminosity, humidity, temperature and motion. Kuzzle built this device to demo Kuzzle Backend's core features, you may have seen it at CES. To make it easier for you, I've created an Elasticsearch docker image with preloaded data. This data was acquired from the device over a period of 5 days, where it sat in our office and captured all our kooky antics. The device used MQTT protocol to communicate with an on-premise Kuzzle Backend instance, which stored any sensor data it received. The Kuzzle Backend data was then exported and loaded into the docker image. For more about the image click [here](https://hub.docker.com/r/kuzzleio/es-tuto-kuzzle-kibana/).

At the end of this tutorial, we will have created a dashboard that shows the office's activity (detected motions) over time.

Let's get started. First of all, you should install docker and docker-compose. ****LINK*** or ***INSTRUCTIONS?***


## 1- Preparing your Docker Compose file

First we have to write a Docker Compose file that will launch our stack: Kuzzle Backend, Elasticsearch and Kibana. 

Create a file called `docker-compose.yml`.

Now add the Kuzzle Backend service in this file. Note that we use environment variables to configure the Elasticsearch connection. We also expose port 7512 which will allow clients to communicate with the service through your docker host.

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
```
Next we need to add Redis (used internally by Kuzzle Backend) to the docker-compose.yml file. Add the following config:

```yaml
  redis:
    image: redis:3.2
```
We will also need Elasticsearch to store the data. For the purpose of this tutorial we use a preloaded docker image that contains data collected from our custom multi-sensor device:

```yaml
  elasticsearch:
    image: kuzzleio/es-tuto-kuzzle-kibana
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

We have now configured the full Kuzzle Backend stack! But don't forget: we are here to make an awesome dashboard! So we'll need Kibana. Add this service to your `docker-compose.yml`

```yaml
  kibana:
    image: docker.elastic.co/kibana/kibana:5.4.1
    environment:
      - SERVER_HOST=0.0.0.0
    volumes:
      - ./kibana.yml:/usr/share/kibana/config/kibana.yml
    ports:
      - 5601:5601
```

If you are a fine observer you may have noticed that we are mounting a volume which contains our own Kibana configuration. Now we need to create this `kibana.yml` file with the following content:

```yaml
server.name: kibana
server.host: "0"
elasticsearch.url: http://elasticsearch:9200
```

Now you can run the Docker Compose file in your favorite terminal:

```bash
$ docker-compose up
```
If everything is correct the terminal should display logs for all the services we just configured. To be sure, you can check if kuzzle Backend is working correctly by opening http://localhost:7512?pretty=true in your favorite browser, you should see a list of routes exposed by Kuzzle Backend.

Your stack is ready! Now you can open Kibana and create amazing visualizations!

## 2- Configuring Kibana and Kuzzle Backend

Kuzzle Backend stores data in document collections within an index. Below is an example document that represents a single motion capture:

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

Before creating charts with Kibana, we have to configure it. Open http://localhost:5601 in a browser and you should see the Kibana management page.

First, we need to tell Kibana where to find our timestamps. To do this, click on "Advanced Settings", find the "metaFields" input, and then click the edit button and add `_kuzzle_info.createdAt`. Don't forget to save your changes!

![Kibana-metafields](img/kibana0.png)

Now we have to specify the index name we want Kibana to use.
Click on "Index Patterns" and type `iot` in the input field. Kibana will automatically find the time based field we just added. Click on "Create" button to validate.

![Kibana-index-pattern](img/kibana1.png)

Kibana will now parse every searchable or aggregatable field and show you these fields. 

The next step is to add a scripted field in Kibana. We want to visualize activity based on motion detection so we will focus on the "motion" boolean:

```json
"state": {
  "motion":true
}
```

Unfortunately, only number fields can be aggregated so we need to create a scripted field that converts the boolean to a number.

Click on "scripted fields" tab and on the "Add Scripted Field" button. Give your new field a name, `detected motions` seems nice.
Kibana uses Painless script, that sounds good to my ears. Go to the "Script" textarea and type :

```
doc['state.motion'].value ? 1 : 0
```
You can ignore the other fields.

![Kibana-index-pattern](img/kibana3.png)

This will create a new aggregatable field for each document in our index that returns a 1 or 0 depending on the `state.motion` boolean value.

Save the new scripted field. We just finished the Kibana configuration !

## 3- Creating Visualizations and Dashboards

This is the fun part of this tutorial and that's why you're here ! We're going to create our first graph, ready? Let's go!

Click on the "Visualize" button on the left navigation panel and click "Create a visualization".

We want our first graph to be a bar graph that displays motion over time, so choose "Vertical Bar" and your index by clicking on "iot".

Kibana is a great tool for visualizing data in real-time. However, for the purpose of this tutorial and to make things simple, we are working with a static dataset. First thing to do is to select a time period. Click on the "time range" button on the top right corner (by default it's set to "Last 15 minutes").
Now click on "Absolute" and choose a range in the date picker. Our preloaded data contains entries from February 2nd, 2018 to February 7th, 2018. Pick these dates and then click the "Go" button.

![Kibana-time-range](img/kibana2.png)


Now that we have selected the data and timeline, we have to configure the graph. First unfold the "Y-axis" menu and select "Sum" as the aggregation type in the dropdown menu. Next we have to choose the field we want to aggregate. Find the `detected motions` field and select it. You can enter a custom label for this aggregation and it will be displayed in the legend of your chart. 

![Kibana-y-axis](img/kibana4.png)

It's time to set up the X-axis, remember we want to display the motion over time, so our X-axis must configured with some kind of timestamp.
Click on the "X-Axis" button and select "Date Histogram". Kibana automatically selects the date field but we have to choose an interval. Pick an hourly interval in the dropdown menu. Give it a custom label if you want, for exemple "Date".

![Kibana-x-axis](img/kibana5.png)

Now click on the "Apply change" button![Kibana-apply-button](img/kibana-apply-button.png) to see our beautiful graph.

![Kibana-bar-graph](img/kibana6.png)

Save your new chart by clicking on the save button on the top menu, give it a name and validate.

To finish up we will add this super graph to a dashboard.
Click on the "Dashboard" button on the left navigation panel and click "Create a dashboard".
Like Kibana says: "This dashboard is empty. Let's fill it up!".

![Kibana-add-to-dashboard](img/kibana7.png)

Select the visualization we just created and it will be added to our dashboard! 

There you have it, we now have a dashboard with a beautiful graph showing motion activity over time as captured by our IoT device!

Kibana lets you share your dashboard, just copy the URL and send it to your motion detection groupies or add it to your webpage in an iframe.

Feel free to play around with the data to add other wild graphs to the dashboard :-)

