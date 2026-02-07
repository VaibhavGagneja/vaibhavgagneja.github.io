---
title: "Part 1: How to Get CKAD (Certified Kubernetes Application Developer) Certification in 2 Weeks"
description: A comprehensive guide to preparing for and passing the CKAD exam with the right resources and study approach.
author: Vaibhav Gagneja
date: 2024-06-02 12:09:45 +0000
categories: [DevOps, Kubernetes]
tags: [ckad, kubernetes, certification, devops, cloud-native, cncf]
image:
  path: https://cdn.hashnode.com/res/hashnode/image/upload/v1717329072620/47b54549-9189-42ab-ad54-b33757b8d286.png
---

## Overview

The Certified Kubernetes Application Developer (CKAD) exam is a certification program developed by the **Cloud Native Computing Foundation (CNCF)** in collaboration with The Linux Foundation. Its primary purpose is to validate the skills, knowledge, and competence of professionals in designing, building, configuring, and exposing cloud-native applications for Kubernetes.

The CKAD exam is **hands-on and performance-based**, requiring candidates to solve practical problems in a command-line environment. It tests the ability to:
- Create and manage Kubernetes objects
- Debug existing configurations
- Use `kubectl` commands effectively

This exam assumes familiarity with container runtimes, microservices architecture, and YAML configuration files.

> **Exam Cost:** $395 (conducted online with strict proctoring)

---

## Syllabus

### Application Design and Build (20%)
- Define, build and modify container images
- Choose and use the right workload resource (Deployment, DaemonSet, CronJob, etc.)
- Understand multi-container Pod design patterns (e.g. sidecar, init and others)
- Utilize persistent and ephemeral volumes

### Application Deployment (20%)
- Use Kubernetes primitives to implement common deployment strategies (e.g. blue/green or canary)
- Understand Deployments and how to perform rolling updates
- Use the Helm package manager to deploy existing packages
- Kustomize

### Application Observability and Maintenance (15%)
- Understand API deprecations
- Implement probes and health checks
- Use built-in CLI tools to monitor Kubernetes applications
- Utilize container logs
- Debugging in Kubernetes

### Application Environment, Configuration and Security (25%)
- Discover and use resources that extend Kubernetes (CRD, Operators)
- Understand authentication, authorization and admission control
- Understand requests, limits, quotas
- Understand ConfigMaps
- Define resource requirements
- Create & consume Secrets
- Understand ServiceAccounts
- Understand Application Security (SecurityContexts, Capabilities, etc.)

### Services and Networking (20%)
- Demonstrate basic understanding of NetworkPolicies
- Provide and troubleshoot access to applications via services
- Use Ingress rules to expose applications

For more information, check the [official website](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/).

---

## Preparing for the CKAD Exam

### 1. Understand the Exam Format and Content

- **Exam Structure**: Performance-based tasks in a command-line environment within two hours
- **Skills Tested**: Application design, configuration, multi-container pods, observability, pod design, services, networking, state persistence, and troubleshooting

### 2. Gain Practical Experience

- **Hands-On Practice**: Set up a Kubernetes cluster (using Minikube or Kind) and practice deploying applications, configuring resources, and managing clusters
- **Key Areas**: Creating and managing pods, deployments, services, and other Kubernetes objects via YAML and `kubectl` commands

### 3. Use Official and Supplementary Study Materials

Use **official Kubernetes documentation** as your primary resource - during the exam, that's your only support!

### 4. Utilize Online Resources

- **Documentation**: Regularly refer to the [Kubernetes official documentation](https://kubernetes.io/docs/home/)
- **Cheat Sheet**: Use the `kubectl` cheat sheet and API reference extensively
- **Online Courses**: Platforms like Udemy, Coursera, and O'Reilly offer comprehensive CKAD preparation courses

### 5. Practice with Mock Exams and Simulators

- **Mock Exams**: Simulate the test environment to help with time management **(most important)**
- **Simulators**: [Killer.sh](https://killer.sh) provides a realistic CKAD exam environment

> **Read my [Part 2 article](/posts/ckad-exam-tips-strategies/) for unique strategy tips and tricks!**

---

## Resources

| Resource | Link |
|----------|------|
| **Kubernetes Course** | [CKAD with Tests (Udemy)](https://www.udemy.com/course/certified-kubernetes-application-developer/) |
| **Exercises** | [CKAD-exercises (GitHub)](https://github.com/dgkanatsios/CKAD-exercises) |
| **More Practice** | [Medium - CKAD Practice Questions](https://medium.com/bb-tutorials-and-thoughts/practice-enough-with-these-questions-for-the-ckad-exam-2f42d1228552) |
| **Theory** | [Yumin Lee's Blog](https://yuminlee2.medium.com/) |
