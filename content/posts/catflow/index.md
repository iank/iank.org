---
template: post
title: "Catflow: A Data Pipeline for Object Recognition"
slug: catflow-data-pipeline-object-recognition
featured_image: catflow_header.png
draft: false
date: 2023-07-28T18:18:18.000Z
description: "The hard part in machine learning is collecting and managing data. This is changing as unsupervised learning advances, and we may soon reach a point where models can just vacuum up messy, incomplete, unlabeled data and make sense of it. But for now, in order to do anything interesting, we usually need to be able to collect, filter, annotate, and manage data."
category: Software
tags:
  - Software
  - Machine Learning
  - Cats
---
## Background

About 10 years ago I interned on a machine learning project. We got to develop a new model, work out some math, and do some prototype implementations. It was a lot of fun, but my mentor told me something that stuck with me: Everything we'd done was the easy part. The hard part is collecting and managing data.

There's a reason why 99% of ML content begins with downloading a canned pristine dataset. This is changing as unsupervised learning advances, and we may soon reach a point where models can just vacuum up messy, incomplete, unlabeled data and make sense of it. But for now, in order to do anything interesting, we usually need to be able to collect, filter, annotate, and manage data. So recently I decided I needed to get better at that. A secondary goal was to end up with a robust and reproducible pipeline instead of the usual mess of semi-manual spreadsheets, scripts, and disorganized copies of data files.

## Project

I have a little webcam attached to a [FROGSBORO]({{< ref "frogsboro.md" >}}) board that's mounted on a wall and pointing down at one of my cats' automatic feeders / water bowls. It used to detect motion events and send me short videos via Signal. Just something to let me know they're okay when I'm traveling. There's no third-party cloud, and the field of view isn't much beyond that section of floor. There's no live access, so I can't feed into my anxiety by tuning in or monitoring much beyond those occasional updates.

But I usually don't travel to places with incredible cell service so getting videos isn't ideal. The videos are also pretty boring anyway. I love my cats but there's only so many 3 FPS videos of them drinking water that I need to watch. Ideally I'd like to receive one picture of each cat per day.

I accomplished this by creating a custom dataset and training a [YOLOv5](https://github.com/ultralytics/yolov5) model.

## Architecture

In order to do this in an ongoing way and I need a few components:

* Ingestion
    * There should be a way to collect raw images.
    * There should be a way to label images.
    * There images should be [pre-annotated](https://labelstud.io/guide/predictions.html), so I only have to fix mis-detections.
    * The only manual step should be fixing a bounding box or label and clicking next.
* Detection
    * There should be a way to run inference and send detections to my phone.
* Training
    * There should be a way to export a labeled dataset and train a model.
    * Again, few manual steps. At most one command to update local copy of the data, and another command to train.

So I built this pipeline:

{{< figure src="catflow_pipeline.png" caption="Message queue architecture" alt="A diagram showing the mesasge queue architecture, described below." >}}

The [FROGSBORO]({{< ref "frogsboro.md" >}}) board POSTs motion events to [`catflow-ingest`](https://github.com/iank/catflow-ingest/), a fastapi endpoint. This uploads videos to an object storage bucket and publishes tasks to a RabbitMQ exchange.

Next, a series of workers process the data. The workers all share a common framework, [`catflow-worker`](https://github.com/iank/catflow-worker), which handles the RabbitMQ and S3 portions. This cuts down on code duplication and makes testing easier, since now each separate worker project is little more than a single handler function.

Videos are divided into frames by [`catflow-service-videosplit`](https://github.com/iank/catflow-service-videosplit), and these frames enter two pipelines: ingestion and detection.

### Ingestion

In the ingestion pipeline, we're going to route (some) frames to [Label Studio](https://labelstud.io/), where they can get labeled and become part of the dataset. First we'll have [`catflow-service-filter`](https://github.com/iank/catflow-service-filter) do some filtering to reduce the labeling burden. We use [resnet](https://pytorch.org/hub/pytorch_vision_resnet/) embeddings and a vector database to filter based on similarity to frames already in the dataset.

Then pre-annotation is run ([`catflow-service-inference`](https://github.com/iank/catflow-service-inference)), and the pre-annotated tasks are passed along to [`catflow-service-taskcreator`](https://github.com/iank/catflow-service-taskcreator) which submits them to Label Studio.

### Detection

In the detection pipeline, we'll just detect objects in each frame (catflow-service-inference) and then do a simple detection algorithm (catflow-service-detector): For a given collection of frames (motion event), we pick the object which has been detected in most of the frames, then select the middle one. This does a pretty good job of smoothing out a series of detections (e.g. an object moving through the scene but being missed or mis-labeled in 1 of 30 frames) and selecting the middle one usually gives us an example frame that is well in the frame (as the cats tend to move through the field of view). We then use the bounding box information to highlight the cat (gaussian filter and some thresholding on HSV data to darken the rest of the image). That "detection" (highlighted picture of a cat + label information) is sent to Signal via signal-cli-rest-api

### Training

Link to catflow-train

When training I used comet to manage experiments. Since the datasets are just metadata (json exported from label studio, and I get the images from S3 by uuid), I just take note of the name/tag of the dataset in comet. But I could use something like DVC in the future.

## Misc

All of the services (my catflow-* stuff, label studio, pgvector, etc) run in docker via docker-compose, there's an example in catflow-docker

I experimented w/ doing the inference on the edge using ONNX, so I wrote a C++ inference implementation (catflow-edge). It's far too slow on the sam9x60, but maybe I'll use a part w/ an AI accelerator and revisit it.

## Lessions

Linting and formatting (pre-commit, ruff, black) was really nice, especially as someone who usually bounces between languages.

Being diligent about testing saved me a bunch of time. While it would often take me about as long to write tests as it did to write the code, writing tests would cut my debugging time drastically. This was especially apparent towards the end of the project, when I had just one last worker to implement and decided not to bother with tests. The deploy/test/debug cycle stretched out much longer than it would've taken me to just write the tests.


The pipeline's architecture evolved from monolithic services to its current modular design. Keeping components well-encapsulated and rigorously tested allowed for confident refactoring.

Notes about Python packaging

Also, this message queue architecture didn't just spring into existence. I started with a couple of monolithic services (catflow-inference, catflow-frameextractor) which evolved and grew as I learned what I wanted to do. The boundaries between those services got really messy, which prompted me to rearchitecture. But keeping components w/in the services well-encapsulated and tested let me refactor with confidence.

