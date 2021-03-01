# kubernetes-k8s-lab11
Deploying a service onto Kubernetes

Show you the basics of deploying a service into Kubernetes from source code to a running Kubernetes cluster.

######

The container

Kubernetes doesn't run your source code directly. Instead, you hand Kubernetes a container.

A container contains 1) a compiled version of your source code and 2) any/all runtime dependences necessary to run your source code.

In this tutorial, we're going to use Docker as our container format. We've created a simple web application in Python, the hello-webapp. To package the webapp as a Docker container, we create a Dockerfile.

We've created a Dockerfile for you, so just type the following commands to see its contents:

cd hello-webapp

cat Dockerfile

This Dockerfile starts with the base image (Alpine:3.5), installs the necessary Python dependencies for the webapp, exposes a port for the webapp on the container, and then specifies the command to run when the container starts.

You can build the Docker container manually with this command:

docker build -t hello-webapp:v1 .

This command reads the Dockerfile, and then builds a binary image that contains everything necessary for the webapp to run.


######
Running your containerized service

Congratulations! You've successfully encapsulated the webapp and everything needed to run the webapp in a container, from the operating system (Alpine Linux) to the runtime environment (Python) to the actual app (hello world).

Let's run the container:

docker run -d -p 80:8080 hello-webapp:v1

(This command runs the container, and maps the container's port 8080 to the hosts port 80.) We can verify that the webapp is running successfully:

curl host01

(We're now sending an HTTP request to port 80 on host01, which then maps to the 8080 in the container.)

######
Kubernetes and manifests

We've packaged up our service, and now it's time to get it running in Kubernetes.

Why do you need Kubernetes? In short, Kubernetes takes care of all the details of running a container: which VM/server to run the container on, making sure the container has the right amount of memory/CPU/etc., and so forth.

In order for Kubernetes to know how to run your container, you need a Kubernetes manifest.

We've created a Kubernetes manifest file for the hello-webapp service.

cat deployment.yaml

You can see that the manifest is written in YAML, and contains some key bits of information about your service, including a pointer to the Docker image, memory/CPU limits, and ports exposed.

So now we have 1) the code to be run, in a container and 2) the runtime configuration of how to run the code, in the Kubernetes manifest.

We now need one more component before we can actually run our container -- a container registry.

######
The Container Registry

In order for your Kubernetes cluster to run an image, it needs to get a copy of the image.

We typically do this through a container registry: a central service that hosts images. There are dozens of options for container registries, for both on-premise and cloud-only use cases.

In this tutorial, the Katacoda environment has provisioned a personal Docker Registry to store the built image. Set the URL for the Registry as an environment variable with the command:

export REGISTRY=2886795296-5000-cykoria04.environments.katacoda.com

In order to push the Docker Image to the registry, we need to create a tag for the Docker image that contains our Docker repository name.

docker tag hello-webapp:v1 $REGISTRY/hello-webapp:v1

Then, we can push the image we just created to the Docker Registry:

docker push $REGISTRY/hello-webapp:v1

######


Running the service in Kubernetes

Now, let's actually get this service running in Kubernetes. We're going to need to update our deployment.yaml file to point to the image for our particular service. For the purposes of this exercise, we've templated our deployment file with the variable IMAGE_URL, which we'll then instantiate with a sed command:

sed -i -e 's@IMAGE_URL@'"$REGISTRY/hello-webapp:v1"'@' deployment.yaml

(f you run cat deployment.yaml you'll see that your specific Docker repository is now in the deployment.yaml file.)

To gain access to the Kubernetes cluster, the Kubernetes configuration file needs to be downloaded from the master. The configuration file contains the IP addresses and certificates required to communicate securely with Kubernetes.

scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@host01:/root/.kube/config ~/.kube/

Now, we can run actually get the service running:

kubectl apply -f deployment.yaml

We're telling Kubernetes to actually process the information in the manifest.

We can see the services running:

kubectl get services
Accessing the cluster

Now, we want to send an HTTP request to the service. Most Kubernetes services are not exposed to the Internet, for obvious reasons. So we need to be able to access services that are running in the cluster directly.

kubectl get pods

When the service was deployed, a dynamic NodePort was assigned to the Pod. This can be accessed via kubectl get svc

Store the port assigned as a variable for later use.

export PORT=$(kubectl get svc hello-webapp -o go-template='{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}')

We can now send an HTTP request to the service:

curl host01:$PORT

You'll see the "Hello, World" message again. Congratulations, you've got your service running in Kubernetes!
The four pieces required for deployment

We've just performed four main things 1) created a container 2) created the Kubernetes manifest 3) pushed the container to a registry and 4) finally told Kubernetes all about these pieces with an updated manifest.

So, this was a bunch of steps, and if you have to do this over-and-over again (as you would for development), this would quickly grow tedious. Let's take a quick look at how we might script these steps so we could create a more useable workflow.
######


Automating the deployment process

Your service is now on Hacker News, and you want to create a custom greeting. So let's change some of the source code:

sed -i -e 's/Hello World!/Hello Hacker News!!!/' app.py

We could rebuild the container, push it to the registry, and edit the deployment, but you've probably already forgotten all the exact commands, right?

Luckily, Forge is an open source tool for deploying services onto Kubernetes, and it already does the automation (and then some!). Let's try using Forge to do this deployment. We need to do a quick setup of Forge:

forge setup

To setup Forge, enter the URL for our Docker Registry: 2886795296-5000-cykoria04.environments.katacoda.com

Enter the username for the Registry, in this case root.

Enter the organization, again root.

Finally, enter root for the password.

With Forge configured, type:

forge deploy

Forge will automatically build your Docker container (based on your Dockerfile), push the container to your Docker registry of choice, build a deployment.yaml file for you that points to your image, and then deploy the container into Kubernetes.

This process will take a few moments as Kubernetes terminates the existing container and swaps in the new code. We'll need to set up a new port forward command. Let's get the pod status again:

kubectl get pods

As previously, obtain the NodePort assigned to our deployment.

export PORT=$(kubectl get svc hello-webapp -o go-template='{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}')

Now, let's check out our new welcome message:

curl host01:$PORT

Congratulations! You've applied the basic concepts necessary for you to develop and deploy source code into Kubernetes.

######