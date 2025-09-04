# Deploy Your First Application with Kind

Kubernetes is the industry-standard platform for automating the deployment, scaling, and management of containerized applications. Getting started can feel daunting because a full Kubernetes setup can be complex.

This is where [kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker) comes in. It's a tool for creating a lightweight, local Kubernetes cluster right on your machine, using Docker as the foundation. It's the perfect environment for learning and experimenting.

In this tutorial, you will:

* Create a local Kubernetes cluster using Kind.

* Deploy a sample web application.

* Access the application from your local machine.

* Clean up all resources.

# Prerequisites
Before you begin, ensure you have the following tools and resources available.

* A computer with **macOS, Linux, or Windows.**

* **Internet access** to pull container images.

* **Docker**: Kind uses Docker to run the Kubernetes cluster nodes. You can install it from the official [Docker website](https://www.docker.com/products/docker-desktop/).

* **Kind**: You can find the installation instructions for your operating system on the [Kind official documentation](https://kind.sigs.k8s.io/).

* **kubectl**: This is the command-line tool you will use to communicate with your Kubernetes cluster. It is often included with Docker Desktop, but if you need to install it separately, follow the [official Kubernetes instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

# Create a Local Kubernetes Cluster

Your first step is to create the **cluster** (a group of connected machines for running applications). This process will download a container image that acts as a Kubernetes **node** (a single worker machine within the cluster), and start it on your machine using Docker.

## Start the Cluster
To create your cluster, issue the `kind create cluster` command in your terminal.


```
$ kind create cluster
```

> **Note:** By default, this command creates a cluster named `kind`. If you run the command again, it will fail because a cluster with that name already exists. To create multiple clusters, you must give each a unique name using the `--name` flag, like `kind create cluster --name my-other-cluster`.

The output will show the address of the **Kubernetes control plane** (the cluster's "brain" that manages all operations), confirming that your cluster is running and `kubectl` can connect to it successfully.

```
$ Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```
## Verify Cluster Connectivity

Once the setup is complete, you should verify that you can communicate with your new cluster. The `kubectl` command-line tool is your primary way of interacting with the Kubernetes Application Programming Interface (API).

Use the following command to get basic information about your cluster:

```
# The --context flag tells kubectl which cluster to talk to.
$ kubectl cluster-info --context kind-kind
```
You will see the address of the Kubernetes control plane, confirming that your cluster is running and `kubectl` can connect to it successfully.

```
$ Kubernetes control plane is running at https://127.0.0.1:55198
CoreDNS is running at https://127.0.0.1:55198/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
# Deploy the Sample Application

With a running cluster, you can now deploy an application. In Kubernetes, you define the desired state of your application using YAML configuration files, often called "manifests."

## Understand the Kubernetes Objects

To get our app running, we need to tell Kubernetes about two key things using a YAML file. Think of it like a recipe.

**Deployment**: This object tells Kubernetes what container image to run and how many copies (replicas) of it to keep running. If an application instance crashes, the Deployment's controller will automatically replace it to maintain the desired state.

**Service**: Pods can be replaced at any time, and when they are, they get a new internal IP address. A Service provides a stable, internal IP address and DNS name that stays constant. For this guide, we are using the default Service type, `ClusterIP`, which is only reachable from within the cluster. This is all we need, because `kubectl port-forward` will create a secure tunnel from our local machine directly to this internal service address.

> Other service types, like `NodePort` and `LoadBalancer`, are used to expose applications directly to external traffic, but that's a topic for a more advanced tutorial.

## Create the Application Manifest

Create a new file named `app.yaml` and add the following content. The comments in the file explain what each section does.

```yml
# This section defines the Deployment object.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web # The name of our Deployment.
spec:
  replicas: 1 # We want to run one copy of our application Pod.
  selector:
    matchLabels:
      app: web # This links the Deployment to the Pods it manages.
  template: # This is the blueprint for the Pods.
    metadata:
      labels:
        app: web # Pods get this label, matching the selector above.
    spec:
      containers:
      - name: hello-app # A name for the container inside the Pod.
        image: gcr.io/google-samples/hello-app:1.0 # The container image to run.
---
# This section defines the Service object.
apiVersion: v1
kind: Service
metadata:
  name: web # The name of our Service.
spec:
  # This selector tells the Service which Pods to send traffic to.
  selector:
    app: web # The selector finds Pods with the label "app: web".
  ports:
  - protocol: TCP
    port: 8080 # The port the Service will be available on inside the cluster.
    targetPort: 8080 # The port on the container that traffic should be sent to.
```

## Apply the Manifest

Now, instruct Kubernetes to create the objects defined in your manifest file. Use the `kubectl apply` command and point it to your file.

```
$ kubectl apply -f app.yaml
```
You should see a confirmation that the deployment and service were created.

```
$ deployment.apps/web created
service/web created
```

You can verify that the application is running by asking Kubernetes to list the Pods. It might take a few moments for the `STATUS` to change to `Running`.

```
$ kubectl get pods
```

```
$ NAME                 READY   STATUS    RESTARTS   AGE
web-84d778c6cb-p4p6c   1/1     Running   0          25s
```

# Access Your Application

The sample application is now running inside your Kind cluster, but the cluster has its own private network. To access it from your local machine's browser, you need to forward a local port into the cluster.

## Use Port Forwarding

The `kubectl port-forward` command creates a secure tunnel from your local machine directly to your application running inside the cluster. We will target the Service we created, as it provides a stable endpoint.

Issue the following command. It will connect your local port `8080` to the Service's port `8080`.

```
$ kubectl port-forward service/web 8080:8080
```

The command will continue running in your terminal, actively forwarding traffic.

```
$ Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```
## View the App
Now, open a web browser and navigate to `http://localhost:8080`.

If everything worked, your browser should display output similar to this:

```
Hello, world!
Version: 1.0.0
Hostname: web-84d778c6cb-p4p6c
```
To stop the port forwarding, return to your terminal and press `Ctrl+C`.

# Cleanup

When you're finished, it's important to remove all the resources to avoid leaving unused components on your machine. We'll first delete the Kubernetes application resources and then destroy the local cluster itself.

## Remove the Application

Use the same manifest file you used to create the application to delete it. This tells Kubernetes to remove all resources defined in app.yaml.
```
$ kubectl delete -f app.yaml
```
You'll see a confirmation that the resources have been deleted.

```
$ deployment.apps "web" deleted
service "web" deleted
```
## Delete the Cluster

Finally, delete your entire local cluster using the kind command.

```
$ kind delete cluster
```
```
$ Deleting cluster "kind" ...
```
# Next Steps

Congratulations! You have successfully used Kind to create a local Kubernetes cluster, deployed a containerized application using a Deployment, and exposed it with a Service. You also learned how to access your application using `kubectl port-forward`.

To continue your Kubernetes journey, we recommend you:

* **Experiment with Scaling**: Modify the `replicas` field in your `app.yaml` file from `1` to `3`. Re-run `kubectl apply -f app.yaml` and then check the result with `kubectl get pods`. You will see Kubernetes automatically scale your application.

* **Explore Kubernetes Concepts**: Dive deeper into the official documentation for [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [Services](https://kubernetes.io/docs/concepts/services-networking/service/) to understand their full capabilities.

* **Try Other Applications**: Find other container images on [Docker Hub](https://hub.docker.com/) and try to deploy them to your Kind cluster by modifying the `app.yaml` file.