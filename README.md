# Kubernetes Learning Guide
![Kubernetes](https://i.imgur.com/Dgmw44W.png)

**⚠️ status of this tutorial: WIP**
> **disclaimer**
 This is, i'm pretty sure, not the best way to do what I want to do, and in neither cases, I'm not pretending to know how all this stuff works \. This journey is my journey through Kubernetes discovery and it's not the answer to everything. 
>
> My only conviction is that If my journey can help me, maybe it can answer to someone's questions. or not.
>
> Feel free to correct me or add more details on some parts if i've (surely) made mistakes.



## Objectives

The aim of this project is to explore and learn Kubernetes. 

Why? I use containers since 3 years now with docker/docker-compose and even docker swarm. Today, I've met some limitations and reading the internet, docker (And docker swarm) seems more and more abandonned for Kubernetes, i'm making also my move into the learning path. 

How will I begin my journey and what is my goal ?

My goal is to run a simple Symfony Application with a PostgreSQL database with kubernetes. The first step will be to make a PHP+Nginx development environment with microk8s. The second step will be to move forward and integrate PostgreSQL and storage management. Then the last step will be to go to production-grade deployment.

I've decided to use a Symfony app because it offers enough complexity due to its architecture.
![Nginx and PHP interraction](https://i.imgur.com/5KPU2tv.png)

The main constraints with this architure are the following:
- Nginx and PHP communicates through unix socket or tcp socket.
- Nginx does not require PHP to serve resources that are not PHP
- Nginx and PHP needs to access to source code and be in the same folder.

## Tools

I will use [microk8s](https://microk8s.io/) for my experiments but I'm pretty sure it works the same on other k8s distributions. 

note that all my `kubectl` commands will be prefixed with `microk8s.` as microk8s as its own wrapper.

## Experiments
All my experiments are in the experiments folder. In this document, you will find every details of the conducted experiments and in the experiments you'll also be able to see comments of what is what.

## What is a pod ? 
![pod like in pea pods](https://i.imgur.com/t0br9k2.png)

> A Pod (as in a pod of whales or pea pod) is a group of one or more containers, with shared storage/network resources, and a specification for how to run the containers. — [The official documentation](https://kubernetes.io/docs/concepts/workloads/pods/)

So why i'm talking about pods ? I've seen tutorials where nginx and php are in separated pods. But as we know php-fpm will not work standalone and nginx will not work without php for this example, so I've decided to merge both containers in the pod.

### EXP-01: the two-containers Experiment

So my first move was to experiment to run one container with another and try to make them communicate.

For my first experiment, I've decided to run two containers (CTA and CTB) in a single pod. I've decided to use [containous/whoami](https://hub.docker.com/r/containous/whoami) image for CTB and [curlimages/curl](https://hub.docker.com/r/curlimages/curl) for CTA. containous/whoami is a lightweight image which prints http request informations, and nothing more. The best of this is that it's a really lightweight image. CTA image comes with a ready to run curl.

My experiment is simple : 
Make a curl request in CTA to CTB:80 and check if I can see the logs. For this I use `watch` a simple loop command which runs every x time a command:
```sh
watch -n 2 curl CTB:80
```

To try my first experiment run the following command: 
```sh
$ microk8s.kubectl apply -f experiments/exp1_2c_deploy.yaml
deployment.apps/exp1 created

```
The output of the command gives you the name of the deployment. Let's dive into what it has created

```shell
$ microk8s.kubectl get all -A
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
default       pod/exp1-57fb774b64-rrnfx                     2/2     Running   0          48s

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
default       deployment.apps/exp1                      1/1     1            1           48s

NAMESPACE     NAME                                                DESIRED   CURRENT   READY   AGE
default       replicaset.apps/exp1-57fb774b64                     1         1         1       48s
```

In your configuration you maybe have way more infos on this page, I only kept the ones we are interested in. 

For this, we will need the pod's name, here, mine is `pod/exp1-57fb774b64-rrnfx`

Let's check now the logs : 
```sh
$ microk8s.kubectl logs pod/exp1-57fb774b64-rrnfx cta
Every 2.0s: curl 127.0.0.1:80                               2020-12-26 12:06:55

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
Hostname: exp1-57fb774b64-rrnfx
IP: 127.0.0.1
IP: ::1
IP: 10.1.193.3
IP: fe80::d40a:e6ff:fefa:940a
RemoteAddr: 127.0.0.1:50172
GET / HTTP/1.1
Host: 127.0.0.1
User-Agent: curl/7.74.0-DEV
100   204  100   204    0     0   199k      0 --:--:-- --:--:-- --:--:--  199k
Accept: */*
```




To undeploy the experiment:
```sh
microk8s.kubectl delete deployment exp1
```


A deployment is not exactly like a docker-compose file. 
The difference is that it doesn't create a network between every containers but runs both in a same host. 