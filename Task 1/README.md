# Task 1: Kubernetes Deployment & Shared Process Namespace

## Overview
This task demonstrates setting up a 3-node Minikube cluster using Calico CNI, deploying a multi-container Pod sharing process namespaces (`shareProcessNamespace: true`), simulating a failing init container, and observing Kubernetes RollingUpdate behavior.

---

## 1. Cluster & Namespace Setup
Created a 3-node Minikube cluster with Calico CNI and created the `webapp` namespace:
```bash
minikube start --nodes 3 --cni=calico --cpus 2 --memory 2200
kubectl create namespace webapp

![alt text](/screenshots/image.png)