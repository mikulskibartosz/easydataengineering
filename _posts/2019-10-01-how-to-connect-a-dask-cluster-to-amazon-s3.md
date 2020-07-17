---
layout: post
title:  "How to connect a Dask cluster (in Docker) to Amazon S3"
author: mikulskibartosz
tags: [ 'Dask', 'AWS', 'Amazon S3' ]
description: "How to connect a Dask cluster (in Docker) to Amazon S3"
featured: false
redirect_to: https://mikulskibartosz.name/how-to-connect-a-dask-cluster-to-amazon-s3
hidden: false
excerpt: "How to connect a Dask cluster (in Docker) to Amazon S3"
---

In this article, we are going to set up a Dask cluster using Docker Compose and configure it to download data from Amazon S3.

I am going to run all of my code on the local machine, but when you have Dask workers configured you can deploy them in a Mesos/Kubernetes cluster using the same configuration (just change the connection strings).

# Docker Compose

First, I'm going to define my Docker Compose file (I will call it: `docker-compose.yaml`). The file starts with the version, so in the first line I put:

```yaml
version: "3.1"
```

## Scheduler

The first service I must run is the Dask scheduler, so I add the following code to my `docker-compose.yaml` file:

```yaml
services:
  scheduler:
    image: daskdev/dask
    hostname: dask-scheduler
    ports:
      - "8786:8786"
      - "8787:8787"
    command: ["dask-scheduler"]
```

We see that it will deploy the Dask scheduler as a `dask-scheduler` host and expose the endpoints used by the workers.

{% include info.html %}

## Workers

In the next step, I have to deploy the workers. In this example, I will configure only one worker, but in a real-life scenerio, you will need many of them (if one worker is enough, you don't need Dask at all).

Note that the worker configuration needs the address of the Dask scheduler and we must mount the directory containing the AWS credentials. I'm going to assume that you pass the credentials as a Docker volume, so the sensitive data is not displayed in any configuration console containing environment variables.

```yaml
  worker:
    image: daskdev/dask
    volumes:
    - $HOME/.aws/credentials:/home/app/.aws/credentials:ro
    hostname: dask-worker
    command: ["dask-worker", "tcp://scheduler:8786"]
```

## Workers (with environment variables)

If for some reason, you need to specify the AWS credentials as environment variables, you can configure the worker in the following way:

```yaml
worker:
    image: daskdev/dask
    environment:
      - AWS_ACCESS_KEY_ID=your_access_key
      - AWS_SECRET_ACCESS_KEY=your_secred
      - AWS_DEFAULT_REGION=the_aws_region
    hostname: dask-worker
    command: ["dask-worker", "tcp://scheduler:8786"]
```

Please remember, that it is not a recommented way because the environment variables (so also the keys and passwords) are usually visible in an administration console, so everyone who can manage your deployed services has access to the keys.

## Notebook

Finally, when I have the scheduler and workers configured, I can add the notebook to the configuration.

```yaml
notebook:
    image: daskdev/dask-notebook
    hostname: notebook
    ports:
      - "8888:8888"
    environment:
      - DASK_SCHEDULER_ADDRESS="tcp://scheduler:8786"
```

Now, when I run the `docker-compose --file docker-compose.yaml up` command, all of the services will start, and I can open the localhost:8888 address in my browser to access the notebook.

Obviously, when you deploy the configuration using Kubernetes or Mesos, the command is going to be different. Check the documentation of the service you are using for deployment.

# Reading files from S3

To create a DataFrame which contains data from an S3 bucket, I have to run the following code in the notebook:

```python
import dask.dataframe as dd
df = dd.read_csv('s3://bucket/directory/file_name-*.csv')
```
