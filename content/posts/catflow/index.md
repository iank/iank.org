---
template: post
title: Catflow: A Data Pipeline for Object Recognition
slug: catflow-data-pipeline-object-recognition
featured_image: catflow_header.png
draft: false
date: 2023-07-28T18:18:18.000Z
description: TODO
category: Software
tags:
  - Software
  - Machine Learning
  - Cats
---
The motivation for the project is that while the "fun" part of ML is usually the math and the models, most real or novel applications need to do the tedious work of collecting, annotating, and managing data. So I want to be able to do that well and have a well-defined / reproducible flow that doesn't devolve into a mess of spreadsheets or semi-manual organization steps.

I have a little camera (FROGSBORO) that's mounted on a wall and pointing down at one of my cats' automatic feeders / water bowls. It used to detect motion events and send me short videos via Signal. Just something to let me know they're okay when I'm traveling. There's no third-party cloud, and the field of view isn't much beyond that section of floor. There's no live access, so I can't feed into my anxiety by tuning in or monitoring much beyond those occasional updates.

But I usually don't travel to places with incredible cell service, so getting videos isn't ideal, and the videos are pretty boring anyway. Ideally I'd just like one picture of each cat per day.


## Architecture

So I built this pipeline. The FROGSBORO board POSTs motion events to catflow-ingest, a fastapi endpoint. catflow-ingest uploads the videos to an object storage bucket and publishes tasks to a RabbitMQ exchange. Here, a series of workers process the data. The workers all share a common framework, catflow-worker, which handles the RabbitMQ and S3 portions. This cuts down on code duplication and makes testing easier, since now each separate worker project is little more than a single handler function.

{{< figure src="catflow_pipeline.png" caption="Message queue architecture" alt="A diagram showing the mesasge queue architecture, described below." >}}

Videos are divided into frames by catflow-service-videosplit, and these frames enter two pipelines: ingestion and detection.

## Ingestion

In the ingestion pipeline, we're going to route (some) frames to Label Studio, where they can get labeled and become part of the dataset. In order to reduce the number of frames (and therefore the labeling burden), we use resnet embeddings and a vector database and filter based on similarity to frames already in the dataset (catflow-service-filter). Then pre-annotation is run (catflow-service-inference), and the pre-annotated tasks are passed along to catflow-service-taskcreator which submits them to label studio. Annotated frames can be exported from label studio and used to train a YOLOv5 model (catflow-train)

## Detection

In the detection pipeline, we'll just detect objects in each frame (catflow-service-inference) and then do a simple detection algorithm (catflow-service-detector): For a given collection of frames (motion event), we pick the object which has been detected in most of the frames, then select the middle one. This does a pretty good job of smoothing out a series of detections (e.g. an object moving through the scene but being missed or mis-labeled in 1 of 30 frames) and selecting the middle one usually gives us an example frame that is well in the frame (as the cats tend to move through the field of view). We then use the bounding box information to highlight the cat (gaussian filter and some thresholding on HSV data to darken the rest of the image). That "detection" (highlighted picture of a cat + label information) is sent to Signal via signal-cli-rest-api

## Misc

All of the services (my catflow-* stuff, label studio, pgvector, etc) run in docker via docker-compose, there's an example in catflow-docker

I experimented w/ doing the inference on the edge using ONNX, so I wrote a C++ inference implementation (catflow-edge). It's far too slow on the sam9x60, but maybe I'll use a part w/ an AI accelerator and revisit it.

When training I used comet to manage experiments. Since the datasets are just metadata (json exported from label studio, and I get the images from S3 by uuid), I just take note of the name/tag of the dataset in comet. But I could use something like DVC in the future.

## Lessions

Linting and formatting (pre-commit, ruff, black) was really nice, especially as someone who usually bounces between languages.

Being diligent about testing saved me a bunch of time. While it would often take me about as long to write tests as it did to write the code, writing tests would cut my debugging time drastically. This was especially apparent towards the end of the project, when I had just one last worker to implement and decided not to bother with tests. The deploy/test/debug cycle stretched out much longer than it would've taken me to just write the tests.


The pipeline's architecture evolved from monolithic services to its current modular design. Keeping components well-encapsulated and rigorously tested allowed for confident refactoring.

Notes about Python packaging

Also, this message queue architecture didn't just spring into existence. I started with a couple of monolithic services (catflow-inference, catflow-frameextractor) which evolved and grew as I learned what I wanted to do. The boundaries between those services got really messy, which prompted me to rearchitecture. But keeping components w/in the services well-encapsulated and tested let me refactor with confidence.

