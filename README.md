# Lab 2 – LLM Gesture Monitor

> Note: This lab specification is subject to change based on course requirements and instructor feedback.

## Overview

In this lab, students will build an end-to-end IoT system where an ESP32 captures raw time-series motion data from an MPU-6050 sensor and sends it to a local, Dockerized AI proxy for zero-shot intent classification. The classified gesture is then displayed in real time on a live web dashboard. This lab introduces students to sensor data pipelines, containerized services, and LLM-based inference in a practical networking setup.

The ESP32 continuously reads gyroscope data to capture movement patterns for complex gestures such as clockwise rotation, anti-clockwise rotation, and other motion-based inputs. The device follows a fixed sampling cycle consisting of two seconds of high-speed continuous data collection, followed by a five second pause. Each sampling window is packaged into a single large JSON array and transmitted to the backend service over HTTP.

## System Architecture

The system is composed of two containerized microservices that students must deploy locally.

The first service is an AI Proxy developed by the student using Flask. This server receives the raw JSON data from the ESP32 via an HTTP POST request. It performs basic preprocessing and constructs a natural language prompt suitable for zero-shot classification. Students will use the OpenAI SDK and function calling features to extract structured intent predictions from the model. This service will be exposed to the public internet using ngrok so the ESP32 can communicate with it.

The second service is a Live Dashboard provided as a pre-built Docker image. Students will run this container locally and expose it via ngrok. This service hosts a simple web interface that receives the classified intent from the AI proxy and displays the current gesture status in real time. While the image is provided, students are responsible for launching, configuring, and networking it correctly.

## Learning Objectives

This lab focuses on foundational Wi-Fi networking concepts by having students configure the ESP32 to connect to a mobile hotspot and perform HTTP API calls. Students will gain practical experience with containerized deployment by running Docker services locally and exposing them securely using ngrok tunnels.

A core learning goal is understanding how to integrate LLMs into real systems. Students will design a Python server that interfaces with OpenAI APIs to perform zero-shot classification, demonstrating structured output extraction through function calling and prompt engineering techniques.

Students will also develop a high-speed data acquisition pipeline using the MPU-6050 gyroscope, learning how to capture, buffer, serialize, and transmit time-series sensor data efficiently.

## Hardware and Software

The hardware setup includes an ESP32 and an MPU-6050 motion sensor. On the software side, students will use C++ on the ESP32, Python with Flask for the AI proxy, Docker for container management, ngrok for tunneling, and OpenAI APIs for gesture classification.

Further setup instructions, wiring diagrams, and evaluation criteria will be provided later.
