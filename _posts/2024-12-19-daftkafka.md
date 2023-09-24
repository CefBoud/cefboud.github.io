---
title: "[WIP] daftkafka: A Minimalist Implementation of the Kafka Protocol"
author: cef
date: 2024-12-19
categories: [Technical Writing, Open Source]
tags: [Apache Kafka, Open Source]
render_with_liquid: false
description: An attempt to implement basic components of the Kafka protocol.
---

# Intro
[daftkafka](https://github.com/CefBoud/daftkafka) is a minimalistic implementation of part of the Kafka protocol.

In its current state, the code can simulate topic creation. It spins up the server (TCP listener), when we run run `kafka-topics.sh --create`, the server communicates in Kafka's protocol to make the CLI client think the topic was created. The code aims to achieve this in the simplest way. See the `TODO` section for future plans.

# Testing
I am running `go1.23.4` for my local testing.

1. Start the server locally on port 9092:

    ```go
    go run main.go
    ```
2. I am testing with Kafka 3.9:

    ```bash
    bin/kafka-topics.sh --create --topic mytopic  --bootstrap-server localhost:9092    

    Created topic mytopic.
    ```

# TODO
- [X] Simulate Topic Creation
- [ ] Actually parse requests (so far, only topic name is retrieved)
- [ ] Produce
- [ ] Fetch

# Credits
[Jocko](https://github.com/travisjeffery/jocko) has been a major inspiration for this work.