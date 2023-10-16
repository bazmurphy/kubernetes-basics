# Kubernetes

## Kubernetes Architecture

- `Master Node` (control plane)

- `Worker Nodes` (each Node has a `kubelet` process running on it)

- `kubelet` is a primary "Node agent" is a process that makes it possible for the Cluster to talk to each other and execute tasks

- each `Worker Node` has containers of different applications deployed on it (a different number of docker containers run on these Worker Nodes)

- Worker Nodes is where the actual work is happening, where your applications are running

- Master Node runs several kubernetes processes that are absolutely neccessary to run and manage the Cluster properly, they are:

- `API Server` which is also a container, it is the ENTRY POINT to the Kubernetes Cluster, this is the process that the differenet kubernetes clients will talk to, like the UI (if using the dashboard), an API (if you are using some automated scripts/technology), and the CLI (command line tool)

- `Controller Manager` - which keeps an overview of the Cluster, whether something needs to be repaired or if a container died and it needs to be restarted etc.

- `Scheduler` which is responsible for scheduling containers on different Nodes based on the workload and available server resources on each Node. It is an intelligent process that decides on which Worker Node the next container should be scheduled on, based on the available resources on those Worker Nodes and the load that that container needs.

- Another key component of the Cluster is an `etcd` key:value storage, which holds at any time the state of the Kubernetes Cluster. It has all the configuration data and all the status data of each Node and each container inside of that Node. And the backing store is made up from this etcd snapshots, because you can recover that whole Cluster state using that etcd snapshot.

- the `Virtual Network` spans all of the Nodes that are part of the Cluster. It turns all of the Nodes inside of the Cluster into one powerful machine that has the sum of all the resources of each indivudal Node.

![](images/02.png)

- The Master Nodes on the Control Plane only have a handlful of master processes, but are much more important, because if you lose a Master Node you can no longer access the Cluster anymore, so that means you have to have a backup of your Master Node, so in Production environments you would have at least 2 Masters inside your Kubernetes Cluster.

- the Worker Nodes have a much higher workload and require much bigger and more resources

## Main Kubernetes Components

- Components: `Pod`, `Service`, `Ingress`, `ConfigMap`, `Secret`, `Deployment`, `StatefulSet`, `DaemonSet`

![](images/03.png)

### Node and Pod

- A `Pod` is the smallest unit in Kubernetes (and is an abstraction over a Container)
- A Pod creates a (running environment) layer on top of the Container, Kubernetes wants to abstract away the specific container technologies (eg. Docker) so you can replace them if you want to
- A Pod usually has only one Application (or Container)
- Each Pod gets it's own IP Address (NOT the Container) and each Pod can communicate with each other using that IP Address
- Pods are Ephemeral, they can be lost very easily, and it will get replaced, but the new Pod that replaces it will get a new IP Address (which is inconvenient if you are communicating with the IP Address) and so another component of Kubernetes: Service is used.

![](images/04.png)

### Service and Ingress

- `Service` is a static IP Address that can be attached to each Pod.
- The Lifecycle of the Pod and Service are NOT connected, so even if the Pod dies the Service and it's IP Address will stay.
- We want our Application accessible through a Browser, and so we have to create an External Service (it opens communication from External Sources)
- But you don't want your Database accessible to Public Requests, so you can create an Internal Service (a type of service that you specify when creating one - Internal is the default)
- The Node can have an External Service with a port eg. http://123.456.789.100:8080/ but that is not great for Production, we would prefer to have a domain name etc. and for that we have another component Ingress.
- So instead of Service, the request first goes to Ingress, which does the forwarding to the Service.

![](images/05.png)

### ConfigMap and Secret

- Where do you usually configure the Endpoint or URL? You usually do it in the application or environment variable, but its usually inside the built image of the application. If the endpoint of the service or service name changed you would have to adjust that URL in the application, re-build the image, push it to dockerhub, pull it into your pod etc. This is a headache. Kubernetes has `ConfigMap` to solve this problem.

![](images/06.png)

- It is your External Configuration of your application
- ConfigMap contains URL of a Database or other services that you use, and in Kubernetes you connect it to the Pod so that the Pod gets the data that ConfigMap contains. So you just adjust the ConfigMap, you don't have to build a new image.

![](images/07.png)

- Also we need a database user and password for example, but we should NOT put these in a ConfigMap it is insecure. For this purpose Kubernetes has `Secret`. Secret is just like ConfigMap but it is used to store Secret Data (credentials) but it is stored in base64encoded format, which is not encrypted by default, so it is meant to be encryptyed using third party tools in Kubernetes. And their are tool sfor this from cloud providers or third party tools to encrypt your Secrets.

- You reference Secret in the Pod, so the Pod can see that data and read the Secret, you can use the Data from ConfigMap or Secret inside of your application Pod using environment variables.

![](images/08.png)

### Volume

- If we have a Database Pod(or Container) and it gets restarted the data would be gone. But we want our Data to be persistent. We can use `Volume` for this.
- It attaches a physical storage on a drive to your Pod and that storage could be on:
- A local machine (on the same server Node where the Pod is running)
- Or on a remote storage, outside of the Kubernetes Cluster (a cloud storage or your own premise storage)
- So when the Database Pod (or Container) gets restarted all the Data will there persisted.
- Kubernetes Cluster EXPLICLITLY does NOT manage any data persistence. You are responsible for backing up the data, replicating and managing it.

![](images/09.png)

### Deployment and StatefulSet

- What happens if the application Pod dies, crashes or needs restarting to update the container image? You would have a downtime where a user can't reach your application.
- This is the advantage of "Distributed Systems" and containers.. so instead of relying on just one application Pod and one database Pod, we are replicating everything on multiple "Servers".
- So we would have a replica of our Node also running and IT IS CONNECTED TO THE SAME SERVICE
- Remember the Service is a persistent static IP address with a DNS Name so that you don't have to constantly adjust the endpoint when a Pod dies, the Service is also a Load Balancer, which means the Service will catch the Request and forward it to whichever Pod is less busy.
- In order to create the Replica of the original Pod, you wouldn't create a second Pod but instead define a Blueprint for the Pod and specify how many Replicas of that Pod you want to run. And that Blueprint is called `Deployment`.
- So in practice you are not working with Pods or creating Pods, you are working with Deployments.
- You can scale up or scale down the number of Pods that you need.
- Remember Pod is a layer of abstraction on top of containers, and so Deployment is another abstraction on top of Pods which makes it more convenient to interact with the Pods, replicate them, and do other configuration.

![](images/10.png)

- So if one of the Replicas of your "application" Pod will die the Service will forward the Requests to another

![](images/11.png)

- What if the Database Pod died? Yoru application still wouldn't be accesible... So we need a Database Replica... BUT WE CAN'T REPLICATE a DATABASE using a Deployment. The reason for this is the Database has a STATE meaning its Data, if we have Replicas of a Database Pod they would all have to access the same shared Database Storage and there you would need a mechanism of which Pods are currently WRITING to that storage or which Pods are currently READING from that Storage in order to avoid Data inconsistencies. And that mechanism in addition to the replicating feature is offered by another Kubernetes component called `StatefulSet`. It is mean't specifically for Applications like Databases (Stateful Apps). StatefulSet takes care of Replicating the Pods and Scaling them up or down, but making sure the Databse reads or writes are Synchronised so that no Database inconsistencies are offered.

![](images/12.png)
![](images/13.png)
![](images/14.png)
![](images/15.png)

- WARNING: Deploying StatefulSet in a Kubernetes Cluster is not easy and can be tedious. So Databases are often hosted outside of the Kubernetes Cluster. And just have the Deployments that replicate and scale with no problem inside of the Kubernetes Cluster and communicate with the External Database.

- So now even if one Node crashes we still have another Node running until the other gets replicated. So we can avoid downtime.

![](images/16.png)

## Summary

![](images/17.png)
![](images/18.png)
![](images/19.png)
![](images/20.png)

## Kubernetes Configuration

- All the configuration goes in a Kubernetes Cluster goes through a Master Node (Control Planet) with a process called API Server
- The UI, or API, or CLI all send their Configuration Requests to the API Server which is the only entry point into the Cluster, and the requests have to be in YAML or JSON format.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
  spec:
    containers:
      - name: my-app
        image: my-image
        env:
          - name: SOME_ENV
            value: $SOME_ENV
        ports:
          - containerPort: 8080
```

- With this we are sending a request to configure a component called Deployment which is basically a template/blueprint for creating Pods.
  ```yml
  kind: Deployment
  ```
- In this specific example we tell Kubernetes to create 2 replica Pods

  ```yml
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  ```

- With each Pod replica having a container based on `my-image` running inside

  ```yml
  containers:
    - name: my-app
      image: my-image
  ```

- In addition to that we configure what the Environment Variables and Port Number inside the Container inside of the Pod

  ```yml
  env:
    - name: SOME_ENV
      value: $SOME_ENV
  ports:
    - containerPort: 8080
  ```

- The Configuration Requests in Kubernets are DECLARATIVE, we declare what is our desired outcome from Kubernetes and Kubernetes tries to meet those Requirements.

- For example, since we have declared that we wanted 2 Replica Pods of my-app Deployment to be run in the Cluster and one of those Pods dies, the Controller Manager sees that this Desired State and Actual State are now different. The Desired State is 2 and the Actual State is 1. It goes to work to make sure that this Desired State is recovered, automatically restarting the 2nd Replica of that Pod.

![](images/21.png)

### 3 Parts of a Kubernetes Configuration File

`nginx-deployment.yaml`

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: ...
spec:
  replicas: 2
  selector: ...
  template: ...
```

`nginx-service.yaml`

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: ...
  ports: ...
```

- Every Configuration File has 3 Parts :

1. `metadata`:

   - `metadata` the metadata

   - the `name` of the component, and more...

     ![](images/22.png)

2. `spec`ification:

   - `apiVersion` - each component has a different apiVersion (look them up)

   - `kind` is what kind of Component is this? eg. `Deployment` or `Service`

   - `spec` the specification of the component

   - attributes of `spec` are SPECIFIC to the `kind` of component you are creating. Deploymen has its own spec attributes and Service has its own spec attributes etc.

     ![](images/23.png)

3. `status`:

   - it is automatically generated and added by Kubernetes

   - it uses this to compare the Desired State (in the specification) to the Actual State and is the basis of the Self-Healing aspect of Kubernetes

     ![](images/24.png)

   - Where does that `status` data come from? It comes from the `etcd` in the Master Node (Control Plane)

   - The Cluster Brain one of the Master Processes that stores the Cluster Data

   - The `etcd` holds the current status of ANY Kubernetes Component

     ![](images/25.png)

### Format of a Kubernetes Configuration File

- `yaml` is a human friendly data serialization standard for all programming languages

- but it is very strict about `indentation`!

- you should store the configuration files with your application code, or it can have its own git repository for it's configuration files

## `minikube`

- Usually in a Kubernetes world when you are setting up a production cluster it will look like this:

- You will have Multiple Masters (at least two)

- You will have Multiple Worker Nodes

- Master Nodes and Worker Nodes have their own separate responsibility

- You would have separate virtual or physical machines that each represent a Node
  ![](images/26.png)

- If you want to test something on your local environment or want to try something out, deploying a new application or new components.. obviously setting up a Cluster like this will be difficult if you don't have enough resources.

- For this there is an open source tool called `minikube`

- `minikube` is one Node Cluster where the Master Processes and Worker Processes both run on one Node.

- And the Node will have a `Docker container runtime pre-installed`, so you will be able to run the Pods with Containers on this Node.

  ![](images/27.png)

## `kubectl`

- Now you have a virtual Node on your local machine that represents `minikube` you need some way to interact with the Cluster, you need a way to create Kubernetes Components on that node.

- We do this using `kubectl` which is a Command Line Tool for Kubernetes Clusters

- Remember `minikube` runs both Master and Worker processes in a single Node, one of the master processes API Server is the main entry point into the Kubernetes Cluster, if you want to configure anything or create any component you first have to talk to the API Server with different clients, like UI/Dashboard, API, or CLI which is `kubectl`

- `kubectl` is the most powerful of the 3 clients^ you can do anything in Kubernetes that you want

- Once the `kubectl` submits commands to the API Server. The Worker Processes on `minikube` Node will actually make it happen. The Worker Processes enable Pods to run on Node. To create Pods, create Services, destroy Pods etc.

![](images/28.png)

- `kubectl` is not just for `minikube` Cluster, if you have a Cloud Cluster or a Hybrid Cluster, `kubectl` is the tool to interact with ANY type of Kubernetes setup

![](images/29.png)

## Install & Run `minikube`

- `minikube` can run either as a Container or a Virtual Machine, so we need either a Container Runtime or a Virtual Machine installed on our system.

![](images/30.png)

- And this will be the driver for `minikube`. https://minikube.sigs.k8s.io/docs/drivers/ is a list of all the supported drivers and Docker is the preferred driver.

- This maybe confusing because inside the Kubernetes Cluster Pods we run Docker Containers.
  - `minikube` installation comes with Docker pre-installed to run the containers in the cluster.
  - But Docker as the driver for `minikube` means we are hosting `minikube` as a container on our local machine.

![](images/31.png)

- So we need Docker installed on our machine before we can use `minikube`

- To install `minikube`:
  `curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64`
  `sudo install minikube-linux-amd64 /usr/local/bin/minikube`

- To start it: `minikube start`, we can specify a driver eg. `minikube start --driver docker`

- We can now interact with our Cluster using `kubectl`

- Display all the Nodes in the Cluster: `kubectl get node`

  ```
  NAME       STATUS   ROLES           AGE     VERSION
  minikube   Ready    control-plane   2m27s   v1.27.4
  ```

- So we have a Kubernetes Cluster running locally on our machine, and we can start deploying applications in it.

- We interact with the `minikube` cluster using `kubectl` Command Line Tool

- `minikube` is for the startup and deleting the Cluster, but everything else we do through `kubectl`
