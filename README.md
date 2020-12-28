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

### Tools

I will use [microk8s](https://microk8s.io/) for my experiments but I'm pretty sure it works the same on other k8s distributions. 

note that all my `kubectl` commands will be prefixed with `microk8s.` as microk8s as its own wrapper.

### Experiments
All my experiments are in the experiments folder. In this document, you will find every details of the conducted experiments and in the experiments you'll also be able to see comments of what is what.

## What is a pod ? 
![pod like in pea pods](https://i.imgur.com/t0br9k2.png)

> A Pod (as in a pod of whales or pea pod) is a group of one or more containers, with shared storage/network resources, and a specification for how to run the containers. — [The official documentation](https://kubernetes.io/docs/concepts/workloads/pods/)

So why i'm talking about pods ? I've seen tutorials where nginx and php are in separated pods. But as we know php-fpm will not work standalone and nginx will not work without php for this example, so I've decided to merge both containers in the pod.

## EXP-01: the two-containers Experiment

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

> **tooltip:**
> ```
> microk8s.kubectl logs $(microk8s.kubectl get pods -o name|grep exp1 | tail -n 1) cta
> ```
> Will get the logs of the last pod named exp1.


To undeploy the experiment:
```sh
microk8s.kubectl delete deployment exp1
```

So what is interesting in this experiment? The first thing to notice is that we are not in the same configuration as with a docker-compose. a docker-compose architecture gives you a network and every containers can communicate each other if they are in the same network by their service name. A pod, on the opposite, contains multiple containers and you can call one container from another using local network. 

The second thing to understand is that you can use docker images to work with kubernetes.

The most important thing to get here is that a deployment is a pod and a replicaSet. So in the configuration you declares a replicaSet and a podTemplate and when you apply the deployment, it will create a number of replicas of the pod defined by the podTemplate.



## EXP-02: The configMap

In order to make my way, I need to inject configuration to nginx to make the http request forwarded to php-fpm. 
My second experiment will be to inject a configuration file into a container and access it from the container.
For this experiment, I will use a [busybox](https://hub.docker.com/_/busybox) container which will run a `tail` command to get the configuration.
The command of this box will be 
```sh
tail -f /to/myfile.txt
```

For this experiment I need to create a configMap. 

> A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.  — [The official documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)

In order to group my experiments, I've combined multiple yaml declarations into a single file, but feel free to split them into two separated files. 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: exp2-config
data:
  my-file-1 : |
    YAY It works
    This is way to awesome !
```
**What is important here** is to understand that the two configurations are separated and therefor this means that my configMap can be reused. 

Then I had to declare in the podTemplate a volume to attach to the pod, and how to use the config keys.

```yaml
    spec:
      volumes:
       - name: my-config 
          configMap:
            name: exp2-config
            items:
              - key: my-file-1 
                path: myfile.txt
```

Then the last part was to declare my container, its command and how to mount the volume in the container. 

```yaml
      containers:
        - name: cta
          image: busybox
          command:
            - tail
            - "-f"
            - /to/myfile.txt
          volumeMounts: 
            - name: my-config
              mountPath: /to/ 
```

This is something interesting as we can use this config to inject a file. 

To test the second configuration run:
```sh
microk8s.kubectl apply -f experiments/exp2_1c_cfg_deploy.yaml

configmap/exp2-config created
deployment.apps/exp2 created
```

As you can see it creates a configMap and a deployment.

```sh
$ micok8s.kubectl get configMap

NAME               DATA   AGE
exp2-config        1      1m
```

Don't forget to delete both deployment and configMap previously created. 

```sh
$ micok8s.kubectl delete configmap/exp2-config deployment.apps/exp2

configmap "exp2-config" deleted
deployment.apps "exp2" deleted
```

## EXP-03 - Share local folder
For this experiment, I wanted to access to local files to use kubernetes as a development environment. 
For this, I decided to mount my `/home` directory and list its content in a busybox container. 

> A `hostPath` volume mounts a file or directory from the host node's filesystem into your Pod. This is not something that most Pods will need, but it offers a powerful escape hatch for some applications. — [The official documentation](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)

You can check exp3 wich gives you a good view of how it works. 

### Method Caveats and how to fix

The main problem of this is that the path is hard-coded, so you cannot embed this config in a git project as it depends on the current's computer. I also tried with `$PWD` and relative path but it doesn't work. 




