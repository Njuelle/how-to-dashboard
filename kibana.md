# How to a create dashboard with Kuzzle and Kibana


## 1- Docker compose and configurations file

First we have to write a docker-compose file (in yaml) to launch a stack with Kuzzle, Elastic-search and Kibana. So, create a file called `docker-compose.yml`.

At first we have to add some services in this file, we need Kuzzle, Redis and Elastic-search :

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
```

Now we have a full Kuzzle backend stack ready to blaze your datas ! But don't forget, we are here for making an awesome dashboard, now add Kibana service to your `docker-compose.yml`

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

If you are an observer you see there is a volume mounted for configuring Kibana. What we need now is to create a file `kibana.yml` and put some lines in it :

```yaml
server.name: kibana
server.host: "0"
elasticsearch.url: http://elasticsearch:9200
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

Now you can run this docker-compose file in your favorite terminal

```bash
$ docker-compose up
```
If everything is correct you can see all logs for all services running. You can check if kuzzle is running correctly by browsing http://localhost:7512?pretty=true, Kuzzle will respond you with a list of the existing routes.

Your stack is ready and we can go to Kibana console for create amazing visualizations !

## 2- Configuring Kibana and Kuzzle

Before create graphs with kibana, we have to add some datas to Kuzzle. Kuzzle database is based on JSON document collections. 

First we need to create an index in Kuzzle, for doing that, simply run this command in a new tab in your terminal :

```bash
$ curl -X POST http://localhost:7512/myawesomeindex/_create
```
You just created a new index called 'myawesomeindex'. But this index seems to be a bit alone, he need a collection to be really awesome.

Always in your terminal create a collection by running this command :

```bash
$ curl -X PUT http://localhost:7512/myawesomeindex/myGreatfCollection
```

Now we have a collection but we need documents to visualize, for follow this tutorial, you can download this JSON sample data [here] and import it to Kuzzle.

```bash
$ curl http://localhost:7512/myawesomeindex/mygreatcollection/_create -d @data.json \
--header "Content-Type: application/json"
```

The file you just imported contain sample data from IoT sensor, we have temperature, light level and humidity percents.

Now it's time to go to Kibana console for some configurations ! Browse http://localhost:5601 and you will arrive to the management page of Kibana.


We have to give to Kibana the index name we just created in Kuzzle. Type 'myawesomeindex' in the input text. Kibana will automatically find the time based field. Click on "Create" button.

![Kibana-index-pattern](img/kibana1.png)

Kibana will parse every searchable or aggreagatable fields and show you these fields. 

We just finishing configuring.

## 3- Create visualizations and dashboard

It's the fun part of this tutorial and that's why you are here ! We will create one graph with the datas previously added in Kuzzle, ready? let's go!

Click on "Visualize" button on left menu and click on "Create a visualization".

Our first graph will be a bar graph of humidity variation, so choose "Vertical Bar" and choose your index by clicking on "myawesomeindex".

Kibana is a great tool for visualizing data in real time, but in this tutorial, for easy comprehension we use an extract of a big JSON dump with a date field. First thing to do is to select a time period. Click on time range button on the top right corner (by default it's set to "Last 15 minutes"), now click on "Absolute" and choose a range in the dates pickers. The JSON dump we use start to 01-01-2018 and end to 01-10-2018, so pick these dates and validate.

![Kibana-time-range](img/kibana2.png)

We need to configure our graph, first, the Y axis :
Unfold the "Y-axis" menu and we have to choose an aggregation, select "Average" in the dropdown menu. Next we have to choose the field we want to aggregate. Like I said, we build a graph about relative humidity, so select this field. It's possible to enter a custom label to embellish our graph, do not deprive us and type "Humidity" in the the text box meant for that purpose.

![Kibana-y-axis](img/kibana3.png)

It's time to configure the X axis, we want to sort our humidity data by date.
Click on the "X-Axis" button and select "Date Histogram". Kibana automatically select the date field but we have to choose an interval, pick a daily interval in the dropdown menu. And like previously type a custom label, for exemple "Date".

![Kibana-x-axis](img/kibana4.png)

Now click on the "Apply change" button ![Kibana-apply-button](img/kibana-apply-button.png) to see our beautiful graph.

![Kibana-bar-graph](img/kibana5.png)

