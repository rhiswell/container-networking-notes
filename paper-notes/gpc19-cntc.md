# CNTC: A Container Aware Network Traffic Control Framework

## Abstract

- As a lightweith virtualization technology, containers are attracting much attention and widely deployed in the cloud data centers.
- Cloud needs resource isolation.
- Docker uses CGroup to provide CPU, memory, and disk resource isolation.
- But ignore the performance of networked hosts.
- **Although several researches discuss the possibility of leveraging Linux Traffic Control (TC) module to guarantee network bandwidth, they fail to capture the diversity and dynamics of container network resource demands and therefore cannot be applied to container-level network traffic control.**
- This paper proposes a Container Network Traffic Control (CNTC) framwork which **can provide strong isolation and container-level mangement for network resource with joint consideration of container characteristics and QoS.**

Keywords: Container Network, Traffic Control, Network Isolation

## Introduction

TODO