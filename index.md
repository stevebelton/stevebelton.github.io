## Goodbye Windows, Hello Ubuntu
I decided a few weeks ago that it was time to ditch Windows as my operating system of choice. I might work for Microsoft but the good thing about the new Microsoft is that we are all for personal choice. Don't get me wrong, I will still be using Microsoft technology but now it will be more Cloud focussed, specifically Microsoft365 or Azure.

My day to day job doesn't rely on anything that I can't run outside of Windows, with one exception... Skype for Business. Until that changes, my laptop will remain a Windows based operating system. For now :-)

Working with [Microsoft Azure](https://azure.microsoft.com) means I get to spend a lot of time interfacing with it either via a CLI or programatically via REST APIs, ARM Templates or Go/C#/Python. I'm going to go through the steps I did to configure my fresh **Ubuntu 18.04** installation.

### Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)
Obviously, my main tool of choice get's installed first!

First off we need to add the Azure CLI Repo to our list of sources
```
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

curl -L https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
```
Next we need to install the **apt-transport-https** package as a prerequisite - then we can install the Azure CLI
```
sudo apt-get install apt-transport-https
sudo apt-get update && sudo apt-get install azure-cli
```
That's it. Simples.
To make sure it works, just login.
```
az login
```
Whilst we're doing the Azure CLI we might as well install the Kubernetes CLI, **kubectl**. This allows us to manage a local (minikube) or remote (AKS) Kubernetes cluster. As we all know, microservices & containers are the future people.
```
az aks install-cli
```

### Install [Docker](https://www.docker.com/)
Next up, Docker. 
Since I am using Ubuntu 18.04 it is super simple to install now.
```
sudo apt-get install docker.io
```
That's it! Then we just need to configure it to auto start and give it a quick startup now also.
```
sudo systemctl start docker
sudo systemcrl enable docker
```
Last thing is to add our current user to the *docker* group so that we can run docker commands without *sudo*.
```
sudo usermod -aG docker $USER
```

### Install [Git](https://git-scm.com/)
Absolutely need this installed! With Microsoft's commitment to OSS and with most of our tools in GitHub, this is a must have tool.

> our recent purchase of GitHub makes this even more so!

```
sudo apt-get install git
```

### Install [Visual Studio Code](https://code.visualstudio.com/)
Finally, a decent IDE - I always loved Visual Studio but it was too bloated - I never thought I would be using one based off Javascript though. Time to install it!
```
curl -O https://go.microsoft.com/fwlink/?LinkID=760868
```
This will download the latest version of the .deb file from Microsoft
```
sudo dpkg -i <file>.deb
```
Make sure you replace <file> with the file downloaded in the previous step
```
sudo apt-get install -f
```
This will fix any broken dependencies from dpkg
  
Now we get to the fun stuff. *Minikube* for testing Kubernetes deployments locally and *Golang* for creating applications to demo in containers and Kubernetes.

Before we can install Minikube we need somewhere for it to run, I have chosen to use Oracle Virtualbox as it's easy to use and simple to install. Lets go install that first.

I will create a separate blog post on configuring Visual Studio Code as there's a lot do there.

### Install [Virtualbox](https://www.virtualbox.org/)
Virtualbox is super easy to install.
```
sudo apt-get install virtualbox
```
You might also want to install the Extension Pack but it's not required for Minikube
```
sudo apt install virtualbox-ext-pack
```
When I ran Virtualbox for the first time it asked me to upgrade the extension pack, but it will do that through the UI and guide you through it.

### Install [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)
First off we need to download Minikube.

```
curl -O https://github.com/kubernetes/minikube/releases/download/v0.27.0/minikube-linux-amd64
```
Then we need to move it into the path and give it permission to run as an executable
```
sudo mv ./minikube-linux-amd64 /usr/local/bin/minikube
sudo chmod +x /usr/local/bin/minikube
```
Now we can start Minikube. This will create a new VM inside of Virtualbox where our Kubernetes single node "cluster" will run.
```
minikube start
```
This might take a few minutes as it has to download the image off the internet.
Once this is up and running I recommend you stop minikube with:
```
minikube stop
```
Then, open up Virtualbox and find the minikube VM. Change it's video memory from 8GB to 16GB so that you get rid of the alert that Virtualbox will continue to generate.

![virtualbox Memory](/virtualbox.png)
