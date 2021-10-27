---
title: "Automating kubernetes local install"
date: 2021-10-25T06:15:36-04:00
draft: true
socialshare: true
categories:
  - "series"
  - "kubernetes"
  - "technology"
tags:
  - "series"
  - "kubernetes"
  - "technology"
---

During prepration of my Certified Kubernetes Administrator (CKAD) and Certified Kubernetes Security Specialist (CKS)exams, I found the value of having a local kubernetes cluster that can be used to experiment. Initially I follwed the instructions from [kubernetes the hard way by Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way) to manually build a local cluster on my Windows 10 laptop.  
<p>It was a great starting point to experiment, but I had to face many challenges with the local installation. Most of my time was getting wasted in either identifying the changes that made cluster unusable or in rebuilding the cluster again when not able to find the root cause of cluster failure.

<!--more-->
