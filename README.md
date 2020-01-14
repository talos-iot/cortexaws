# Example of Cortex on AWS

## Background
This document is designed to be a next step after you've done the [getting started guide](https://github.com/cortexproject/cortex/blob/master/docs/getting_started.md).  It will cover getting your Cortex data off of the `/tmp/cortex` directory and into AWS DynamoDB.  It assumes that you can issue basic PromQL queries.

You'll need an AWS account with appropriate IAM roles to manage DynamoDB.  This guide was tested using a role with `AmazonDynamoDBFullAccess`.  

You will also need docker and docker-compose.  This guide was tested on 
```
docker-compose version 1.25.0, build 0a186604
```


We will use the single instance, single process version of Cortex.  Omitting the `-target` command line argument instructs Cortex to use all the microservices.

## Set up

Git clone this, then store your AWS credentials as environment variables.  

```
export AWS_ACCESS_KEY_ID=<your access key>
export AWS_SECRET_ACCESS_KEY=<your secret>
```

Edit the `cortex/cortex.aws.yaml` file.  Find the schema section.  This section tells Cortex how to route the chunks.  You could, for example, use different chunk or index stores for data scraped at different times.  If you were already running Cortex in production, this is useful as a pathway to upgrade from (say) local storage to DynamoDB.

For this demo, you'll want to change the `from` date to something recent like yesterday.  Otherwise, Cortex will pre-create 1 table for every 168h (1 week) and you'll end up with a lot of tables in your DynamoDB.
```
schema:
  configs:
  - from: 2020-01-10 # <--- Change to yesterday's date
    store: aws-dynamo
    object_store: aws-dynamo
    schema: v10
```

Next we can start the demo.
```
source ./startdemo
docker ps
```
This should show that you have 3 docker containers running: Prometheus, Grafana, and Cortex.  Prometheus is configured to do remote writing to cortex.  

## Verifying durable writes

You should now be able to add Cortex as a datasource in Grafana (default user and password is `admin`).  Add a datasource and put `http://cortex:9009/api/prom` as the URL.  This works because Grafana and Cortex are on the same docker network (cortexaws\_mynet) and so are reachable by name.

Run any query.  You should be able to see the same data in Grafana (querying Cortex) as in Prometheus.  

Log into your AWS DynamoDB dashboard.  You should see (at least) 2 tables: one for chunks and one for the index.  They are both pre-pended with `cortdemo_`.  They should also be set to On-Demand scaling (rather than provisioned capacity).  This is done to reduce the cost of this demo.  To save cost, Cortex has some fancy auto-scaling features that let you have provisioned capacity for your active tables and On-Demand for older less active tables.  Configuring this feature is outside the scope of this demo.

Go have a cup of coffee.  In a few minutes, Cortex should start to populate these tables with data.

Once you see the files start to come in, verify again that you can see data in both Prometheus and Grafana.  Next we are going to destroy the data in Prometheus, and verify that it is indeed still available to Grafana.  

```
docker stop cortexaws_prometheus_1
docker volume rm cortexaws_prometheus_data
docker-compose up -d --build prometheus
```

This destroys the Prometheus data volume.  If you query Prometheus, you should only see any data scraped since it was re-created.  The Cortex data, safely in DynamoDB, is available from the beginning of the demo.

## Clean up

Clean up all the docker containers, volumes, and network
```
docker-compose stop
docker rm cortexaws_prometheus_1 cortexaws_cortex_1 cortexaws_grafana_1
docker volume rm cortexaws_prometheus_data cortexaws_grafana_data
docker network rm cortexaws_mynet
```

Delete the DynamoDB tables.  You can do this on the web console or by
```
aws dynamodb delete-table --table-name cortdemo_index_2610
aws dynamodb delete-table --table-name cortdemo_chunks_2610
```
Note that you may have to substitute a different number above.

