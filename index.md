## Posts
* [Configuring Azure Container Registry](#acr)
* [Deploying Containerized Go app to Minikube](#deployminikube)
* [Containerizing a Go app](#containergo)
* [Getting started with Go](#golang)
* [Running Minikube](#minikube)
* [Goodbye Windows, Hello Ubuntu](#goodbye)

*Start from the bottom up if you're new here :-)*

## <a name="acr"></a>Configuring Azure Container Registry
In my next post we will look to deploy the Containerized Go app into Azure Kubernetes Service (AKS). Before we can do that we need to create an Azure Container Registry to store our Docker image of the Book API application.

We need to start by logging into the Azure CLI:
```
$ az login
```
Once logged into Azure we then need to list our subscriptions and pick the one we want to deploy our ACR resource into:
```
$ az account list
$ az account set --subscription <subscription ID>
```
Replace the \<subscription ID\> with the *id* field from your chosen subscription.
Next we need to create a Resource Group to hold our Azure Container Registry:
```
$ az group create --name test-group --location eastus
```
I have chosen East US as my region as I will be deploying AKS into the same region. At the time of writing only 5 regions support AKS. You van view the list [here](https://docs.microsoft.com/en-us/azure/aks/container-service-quotas)

Now let's create the ACR resource itself. The ACR name needs to be globally unique as it is assigned to an FQDN.
```
$ az acr create --name mytestacr001 --resource-group test-group --sku Basic --admin-enabled true
```
I am using the Basic SKU as this is just for dev/test and have enabled the *--admin-enabled* flag so that we don't have to bother with creating Service Principals - if this was for a Production deployment then we should leave this to *false*. I will cover Service Principals for AKS another time.

You should see output similar to the following:
```JSON
{
    "adminUserEnabled": true,
    "creationDate": "2018-06-06T01:44:27.008413+00:00",
    "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/test-group/providers/Microsoft.ContainerRegistry/registries/mytestacr001",
    "location": "eastus",
    "loginServer": "mytestacr001.azurecr.io",
    "name": "mytestacr001",
    "provisioningState": "Succeeded",
    "resourceGroup": "test-group",
    "sku": {
      "name": "Basic",
      "tier": "Basic"
    },
    "status": null,
    "storageAccount": null,
    "tags": {},
    "type": "Microsoft.ContainerRegistry/registries"
  }
```

Now we need to login to our Azure Container Registry. This is equivalent to logging into Docker Hub and allows us to push our image up to ACR with regular Docker commands:
```
$ az acr login --name mytestacr001 --resource-group test-group
```
At the moment our bookapp:v1 Docker image is tagged as just a local Docker image, there is no prefix. In order to deploy our image into ACR we need to tag it with our new ACR login server name as a prefix:
```
$ docker tag bookapp:v1 mytestacr001.azurecr.io/bookapp:v1
```
With that done we can now push our image up to our new Azure Container Registry:
```
$ docker push mytestacr001.azurecr.io/bookapp:v1
```
After a few minutes you should see that the container image has been successfully stored in our new registry.
```
The push refers to repository [mytestacr001.azurecr.io/bookapp]
566ebac6410e: Pushed
da4eb6a08c72: Pushed
33e3f07c7ac8: Pushed
933b65eebe7d: Pushed
5ea538dd9f75: Pushed
6fa15c0fadea: Pushed
db38deaf0d0c: Pushed
fbbeadfbd3ca: Pushed
0f6f641d80ca: Pushed
76a66da94657: Pushed
0f3a12fef684: Pushed
v1: digest: sha256:dfd4cb81c5e09166904b1bade3a08524cbfab444ce136769bfdb20152923a35a size: 2633
```
Next post we will deploy this new ACR based container image into AKS!


## <a name="deployminikube"></a>Deploying Containerized Go app to Minikube
In my [last post](#containergo) I containerized a Go application using Docker. Now we're going to deploy this container into our local Minikube environment. You will see how easy this is an why so many people use Minikube for their local Kubernetes testing.

First, make sure Minikube is running and is pointing at the local Minikube cluster. The output should point to minikube-vm.
```
$ minikube status
```
Run the following commands.
```
$ kubectl run bookapp --image=bookapp:v1 --port=8000
$ kubectl expose deployment bookapp --type=LoadBalancer
```
These two commands will deploy the bookapp container to Minikube and then expose the deployment to the local machine - see my [previous post](#minikube) for more detail. If the first command fails with an error finding the Docker image, it is likely you did not run the *eval $(minikube docker-env)* command before creating it. You will need to re-create the Docker image after running this command.

To bring the website up in the local browser, simply run:
```
$ minikube service bookapp
```
You will know see something similar to the image below. Notice how the URL is different to the Docker based URL, with a Kubernetes/Minikube exposed IP address and port.

![minikube deployment](/minikubedeploy.png)

## <a name="containergo"></a>Containerizing a Go app
What I thought I would do in this post is to show you how to run a cool little Go application that acts as simple RESTful API service for a Book application. This application was written by Brad Traversy and is demonstrated/explained in his [YouTube video](https://youtu.be/SonwZ6MF5BE) and demonstrates the [Gorilla MUX router](http://www.gorillatoolkit.org/pkg/mux).

First off, we need to clone the sample repository:
```
$ git clone https://github.com/bradtraversy/go_restapi
```
You can run and test this locally with:
```
$ go get github.com/gorilla/mux
$ go run main.go
```
This will install the Gorilla Mux router and start a web server, listening on port 8000. If you browser to http://localhost:8000/books you should see the 2 sample books being served as JSON. Press CTRL-C to stop the application before continuing.

In order to run this application in Docker, we need to create a Docker file in the same directory with the following contents:
```
FROM golang:latest
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go get github.com/gorilla/mux
RUN go build -o main .
CMD ["/app/main"]
```
This will use the *golang:latest* container image from DockerHub and copy our sample Book API application into the app directory. It will also install the Gorilla Mux router and build the application ready for running on container start.

Now we can build out new container with Docker and give it a test locally! **This is where we need to choose our Docker environment first. If you plan on following my [next post](#deployminikube) where we deploy this container to Minikube, you need to run the step below. If not, skip this step and proceed to the Docker Build command.**
```
$ eval $(minikube docker-env)
```
Running this command will expose the container to the Minikube local Docker registry.
```
$ docker build -t bookapp:v1 -f Dockerfile .
```
Once the Docker image has been built we can run it with:
```
$ docker run -p 8000:8000 bookapp:v1
```
This will expose port 8000 to our local machine so you can then test the application by browsing to http://localhost:8000/books as before, only this time the page will be served from our new container!

![container](/localmachine.png)

## <a name="golang"></a>Getting started with Go
So far in a [previous post](#goodbye) I have setup my new PC with Ubuntu, installed all the tools I need to work with Azure, Docker and Kubernetes. Now lets look at getting Go installed so we can start to develop some applications of our own to deploy into Docker, Minikube or AKS!

### Install Golang
First of we need to download and configure Go to work in Ubuntu.

```
$ curl -O https://dl.google.com/go/go1.10.2.linux-amd.tar.gz
```
This will download Go for us and it's pretty simple to install - so long as we configure our path variables everything should work first time around.

```
$ sudo tar -xvf ./go1.10.2.linux-amd64.tar.gz
$ sudo mv ./go /usr/local
```
The above will extract the Golang folder structure and then move the entire top level folder to /usr/local where we will then reference it by updating our PATH environment variable.
```
$ sudo nano ~/.profile
```
Add the following to the end of the file:
```
$ export PATH=$PATH:/usr/local/go/bin
$ export GOPATH=$HOME/work
```
This assumes you would like a working directory for your Go applications to be in the *work* folder in your home directory.
Save this file and run the following to reload:
```
$ source ~/.profile
```
Next week need to create some working folders:
```
$ mkdir $HOME/work
$ mkdir -p $HOME/work/src/github.com/<your github username>
```
Obviously replacing \<your github username\> with your real GitHub username. This makes working with Import statements easier later.
        
That should be it ! We can now test Go works but creating a *Hello World* application and running that. Copy the following code into $HOME/work/src/github.com/<your github username>/hello/hello.go folder/file.

```go
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```
Once you have saved this file you can run it with:
```
$ cd $HOME/work/src/github.com/<your github username>/hello
$ go run ./hello.go
```
You should see the *hello, world* output.

![helloworld](/helloworld.png)

Next post I will look at Containerizing a Go application.


## <a name="minikube"></a>Running Minikube
My [last post](#goodbye) described how I setup my new Ubuntu desktop PC with all the tools I need to do my job. Now I will look at how we test Minikube is working by deloying a sample application

Now we have Minikube working we can deploy a sample application to test it. First we need to make sure Minikube is running with:

```
$ minikube status
```
Assuming we get something like the below, we're good to go:

```
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100
```
First let's start the deployment (this is taken from the [Kubernetes site](https://kubernetes.io/docs/getting-started-guides/minikube/) for reference):
```
$ kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.10 --port=8080
```
This will download the sample echoserver node.js application and start a new Kubernetes deployment.
You can see the deployment running by entering:
```
$ kubectl get deployment
```
Then we need to expose port 8080 to our host system so we can see the website running. With Minikube this is super easy, completed with just one command:
```
$ kubectl expose deployment hello-minikube --type=NodePort
```
This will automatically create a Kubernetes service and expose the port. Running *kubectl get services* will show you something similar to the below:
```
$ hello-minikube   NodePort       10.100.74.0     <none>        8080:32732/TCP   16s
```
Note the hello-minikube service.
Running the following will query the new service and show the output from the application:
```
$ curl $(minikube service hello-minikube --url)
```
The result should look something similar to:
```
Hostname: hello-minikube-7c77b68cff-dn7pf

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=172.17.0.1
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://192.168.99.100:8080/

Request Headers:
        accept=*/*
        host=192.168.99.100:32732
        user-agent=curl/7.58.0

Request Body:
        -no body in request-
```
We can now delete the service and deployment to clean up Minikube.
```
$ kubectl delete services hello-minikube
$ kubectl delete deployment hello-minikube
```


## <a name="goodbye"></a>Goodbye Windows, Hello Ubuntu
I decided a few weeks ago that it was time to ditch Windows as my operating system of choice. I might work for Microsoft but the good thing about the new Microsoft is that we are all for personal choice. Don't get me wrong, I will still be using Microsoft technology but now it will be more Cloud focussed, specifically Microsoft365 or Azure.

My day to day job doesn't rely on anything that I can't run outside of Windows, with one exception... Skype for Business. Until that changes, my laptop will remain a Windows based operating system. For now :-)

Working with [Microsoft Azure](https://azure.microsoft.com) means I get to spend a lot of time interfacing with it either via a CLI or programatically via REST APIs, ARM Templates or Go/C#/Python. I'm going to go through the steps I did to configure my fresh **Ubuntu 18.04** installation.


![Desktop](/desktop.png)


### Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)
Obviously, my main tool of choice get's installed first!

First off we need to add the Azure CLI Repo to our list of sources
```
$ AZ_REPO=$(lsb_release -cs)
$ echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

$ curl -L https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
```
Next we need to install the **apt-transport-https** package as a prerequisite - then we can install the Azure CLI
```
$ sudo apt-get install apt-transport-https
$ sudo apt-get update && sudo apt-get install azure-cli
```
That's it. Simples.
To make sure it works, just login.
```
$ az login
```
Whilst we're doing the Azure CLI we might as well install the Kubernetes CLI, **kubectl**. This allows us to manage a local (minikube) or remote (AKS) Kubernetes cluster. As we all know, microservices & containers are the future people.
```
$ az aks install-cli
```

### Install [Docker](https://www.docker.com/)
Next up, Docker. 
Since I am using Ubuntu 18.04 it is super simple to install now.
```
$ sudo apt-get install docker.io
```
That's it! Then we just need to configure it to auto start and give it a quick startup now also.
```
$ sudo systemctl start docker
$ sudo systemcrl enable docker
```
Last thing is to add our current user to the *docker* group so that we can run docker commands without *sudo*.
```
$ sudo usermod -aG docker $USER
```

### Install [Git](https://git-scm.com/)
Absolutely need this installed! With Microsoft's commitment to OSS and with most of our tools in GitHub, this is a must have tool.

> our recent purchase of GitHub makes this even more so!

```
$ sudo apt-get install git
```

### Install [Visual Studio Code](https://code.visualstudio.com/)
Finally, a decent IDE - I always loved Visual Studio but it was too bloated - I never thought I would be using one based off Javascript though. Time to install it!
```
$ curl -O https://go.microsoft.com/fwlink/?LinkID=760868
```
This will download the latest version of the .deb file from Microsoft
```
$ sudo dpkg -i <file>.deb
```
Make sure you replace \<file\> with the file downloaded in the previous step
```
$ sudo apt-get install -f
```
This will fix any broken dependencies from dpkg
  
Now we get to the fun stuff. *Minikube* for testing Kubernetes deployments locally and *Golang* for creating applications to demo in containers and Kubernetes.

Before we can install Minikube we need somewhere for it to run, I have chosen to use Oracle Virtualbox as it's easy to use and simple to install. Lets go install that first.

I will create a separate blog post on configuring Visual Studio Code as there's a lot do there.

### Install [Virtualbox](https://www.virtualbox.org/)
Virtualbox is super easy to install.
```
$ sudo apt-get install virtualbox
```
You might also want to install the Extension Pack but it's not required for Minikube
```
$ sudo apt install virtualbox-ext-pack
```
When I ran Virtualbox for the first time it asked me to upgrade the extension pack, but it will do that through the UI and guide you through it.

### Install [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)
First off we need to download Minikube.

```
$ curl -O https://github.com/kubernetes/minikube/releases/download/v0.27.0/minikube-linux-amd64
```
Then we need to move it into the path and give it permission to run as an executable
```
$ sudo mv ./minikube-linux-amd64 /usr/local/bin/minikube
$ sudo chmod +x /usr/local/bin/minikube
```
Now we can start Minikube. This will create a new VM inside of Virtualbox where our Kubernetes single node "cluster" will run.
```
$ minikube start
```
This might take a few minutes as it has to download the image off the internet.
Once this is up and running I recommend you stop minikube with:
```
$ minikube stop
```
Then, open up Virtualbox and find the minikube VM. Change it's video memory from 8GB to 16GB so that you get rid of the alert that Virtualbox will continue to generate.

![virtualbox Memory](/virtualbox.png)
