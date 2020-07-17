---
layout: post
title:  "Loading tensorflow models from Amazon S3 with Tensorflow Serving"
author: mikulskibartosz
tags: [ 'Tensorflow', 'AWS', 'Amazon S3' ]
description: "How to save the model in a file, upload it to S3, and serve it using the Docker image of Tensorflow Serving"
featured: false
redirect_to: https://mikulskibartosz.name/loading-tensorflow-models-from-amazon-s3-with-tensorflow-serving
hidden: false
excerpt: "How to save the model in a file, upload it to S3, and serve it using the Docker image of Tensorflow Serving"
---

In this article, I am going to show you how to **store a Tensorflow model in a file, upload it to Amazon S3, and configure the Docker image of Tensorflow Serving** to serve that model via REST API.

## Saving the model

Before we start, we have to save a Tensorflow model in a file using the **simple_save function**.

I'm going to assume that you have already trained your model. We need to specify the output directory and make sure that such a location exists. When the target directory is ready, we can call the simple_save function.

```python
import tensorflow as tf
from tensorflow import keras

export_path = "/home/user/some_location" #the absolute path to the target directory
model = ... #a trained Tensorflow model

tf.saved_model.simple_save(
    keras.backend.get_session(),
    export_path,
    inputs={'input_image': model.input},
    outputs={t.name:t for t in model.outputs})
```

{% include info.html %}

## Upload to S3

If you have **Amazon command line tool** installed, you can use the following command to upload the model to S3.

Note that you should replace `/home/user/some_location` with the location where you stored the model and `tensorflow_models/example_model` with your bucket name.

```bash
aws s3 cp /home/user/some_location s3://tensorflow_models/example_model --recursive
```

## Configuring Tensorflow serving to use a model from S3

To serve the model, we are going to need five additional information. The most important part is your AWS account configuration: **AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY**, and the name of the **AWS region**, which you used to store the file.

You should decide what **the name of your model** is. Tensorflow serving will use that name as a part of the REST endpoint URL.

We also need **the address of the S3 endpoint**. If you know the AWS region you use, you can find the S3 endpoint address in the documentation: <a href="https://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region" target="_blank">https://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region</a>

Take a look at the following command and modify the aforementioned parameters.

```bash
docker run -p 8501:8501 \
    -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
    -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
    -e MODEL_BASE_PATH=s3://tensorflow_models/example_model \
    -e MODEL_NAME=example_model \
    -e S3_ENDPOINT=s3.eu-central-1.amazonaws.com \
    -e AWS_REGION=eu-central-1 \
    -e TF_CPP_MIN_LOG_LEVEL=3 \
    -t tensorflow/serving
```

Now, you can run that Docker image, and it will load the model from Amazon S3.
