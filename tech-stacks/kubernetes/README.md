# Kubernetes - k8s


## Table of Contents
1. [Mind Map](#mind-map)
1. [Overview](#overview)
1. [Components](#components)
1. [Architecture](#architecture)
1. [YAML config](#yaml-config)
1. [NameSpaces](#namespaces)
1. [Helm](#helm)




## Mind Map
![architecture](/tech-stacks/kubernetes/K8S.jpg)

## Overview
- K8s is a container orchestration tool to build apps with high availability, scalability & robustness
- K8s gained it’s popularity due to the trend from Monolithic to Microservices - the need for small & independent applications

## Components
### Node

- a simple server, can be a physical or a virtual machine
- can set replica number to have copies

### Pod

- smallest unit, basic component of k8s
- abstraction layer/running env over a container - container technology independent
- usually one app per pod
- each pod gets a virtual IP to communicate internally
- **IP** will be changed during replacement - **Service** (permanent IP address) can be attached to a pod (lifecycle independent). Service is also shared by replicas and acts as a LB
- **Ingress** acts as an entry point and redirects plain text domain name to the corresponding IP & port, like a reverse proxy and load balancer, based on routing rules

### ConfigMap

- usually config files live in the app property file, any change will require re-build & re-deploy
- **ConfigMap** lives outside the app and contains info like the urls of other services

### Secret

- similar to **ConfigMap** but for sensitive info like username and password, TLS cert
- base64 encoded

### Volumes

- attaches a physical storage (can be local or remote) to the Pod and provide the reference
- so the replacement of DB pods or containers will not result in data loss

### Deployment

- another abstraction on top of pods
- blueprint for pods management - CRUD
- cannot be used only to replicate DB pods because they are stateful, needs mechanism to define read/write replica, master/slave, ect to avoid data inconsistency - **StatefulSet**
- **StatefulSet** is like “deployment” but for stateful apps to make sure data is synced, it’s complicated so it’s common to host DB outside k8s and not use **StatefulSet**

## Architecture

just install required processes to plain server then you will get master or worker nodes

### Worker Node

#### container runtime

- container runtime needs to be installed on every node according to the container of choice
    - docker
    - CRI-O
    - containerd
    - Windows Containers

#### Kubelet

- node agent on each node, used to schedule pods, register nodes with **apiserver**, assign resources, ect, based on yaml config
- interacts with both node and container
- Kubelets communicate with each other via services

#### Kube Proxy

- intelligent forwarding request from services to pods in a performant way
    - prefer replica on the same node
    - based on resources availability

### Master Node

- user interact with cluster via master node
- schedule, monitor, re-start, add new node, ect
- multiple masters
- less work compared to worker nodes so need less resources, CPU, RAM & storage

#### ApiServer

- cluster gateway, gatekeeper, authentication, request validation
- user interact with api server via client, like UI, k8s api or kubectl

#### Scheduler

- take instruction from api server and send request to worker nodes
- intelligent - look at the available resource like CPU, RAM
- it just decided which node to start, but the actual process is handled by **kubelet**

#### Controller Manager

- detects status changes of pods (die)
- ask scheduler to reschedule

#### etcd

- cluster brain/records
- k-v store of all cluster info and changes
- NOT for application data storage

## YAML config

#### metadata

- name
- label
    - used to connect components

#### kind

- service
- deployment
- secret

#### spec

based on the kind

- replicas
- port
    - port - invoke service itself
    - targetPort - to pod
- selector
    - used to connect components
- template
    - config inside config - for pods
    - image
    - port

#### status

- automatically maintained by k8s, constantly used to compare with spec and plan for the changes needed
- stored in **etcd**

## Namespaces

- clusters inside cluster
- not for small projects
- conflict: same name but different configs across teams
- resource sharing: staging and development can share the same cluster (namespaces)
- access and resources limit control: CRUD in their own namespaces
- used to organize & structure related components together by logic or by functionalities (monitoring, logging, db, elastic, nginx) or by teams
- out-of-box namespaces:
    - default: CRUD resources when not creating new namespace
    - kube-system: not for user, contains system processes, like kubectl
    - kube-public: contains publicly accessible data, like cluster info
    - kube-node-lease: contains info about heartbeat of nodes
- you cannot access some components from another namespace
    - Each namespace must have its own **ConfigMap**, **Secrete**
    - namespaces can share **Service**
- Some components (low-level resources) cannot be put in any namespace, they need to be global
    - Volume
    - Node: Node is a **physical** thing and a container(POD) is running top of it.

## Helm

- package manager for k8s
- package YAML files and distribute them in public and private repo
- Helm Charts is a bundle of YAML files
- templating engine
