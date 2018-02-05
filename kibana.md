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

## 2- Configuring Kibana

Before create graphs with kibana, we have to add some datas to Kuzzle. Kuzzle database is based on JSON document collections. 

First we need to create an index in Kuzzle, for doing that, simply run this command in a new tab in your terminal :

```bash
$ curl -X PUT http://localhost:7512/myAwesomeIndex/_create
```
You just created a new index called 'myAwesomeIndex'. But this index seems to be a bit alone, he need a collection to be really awesome.

Always in your terminal create a collection by running this command :

```bash
$ curl -X PUT http://localhost:7512/myAwesomeIndex/myWonderfulCollection
```



4) add '_kuzzle_info' metafields

5) create scripted fields

6) create visualizations

7) create dashboard