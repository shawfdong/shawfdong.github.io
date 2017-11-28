---
layout: post
title: Notes for "Scalable Microservices with Kubernetes"
tags: [Kubernetes, Docker]
---

My notes for the course "Scalable Microservices with Kubernetes" on [Udacity](https://www.udacity.com/).<!-- more -->

* Table of Contents
{:toc}

## Resources
* Books
  * [Kubernetes: Up and Running](http://cruzcat.ucsc.edu/record=b4733196~S5)
  * [Building Microservices: Defining Fine-Grained Systems](http://shop.oreilly.com/product/0636920033158.do)
  * [Kubernetes in Action](https://www.manning.com/books/kubernetes-in-action)  
* Articles
  * Martin Fowler on the [Pros](http://martinfowler.com/articles/microservices.html) and [Cons of Microservices](http://martinfowler.com/articles/microservice-trade-offs.html)
  * [12-factor manifesto](https://12factor.net/)
  * [12-Fractured Apps](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c)
* Tools
  * [Google Cloud Shell](https://cloud.google.com/shell/docs/)
  * [Docker](https://www.docker.com/)
  * [Kubernetes](http://kubernetes.io/)
  * [Google Container Engine (GKE)](https://cloud.google.com/kubernetes-engine/)
  * [JSON Web Tokens (JWT)](https://jwt.io/)
  * [The Go Programming Language](https://golang.org/)

## Introduction to Microservices
Activate [Google Cloud Shell](https://cloud.google.com/shell/docs/starting-cloud-shell).

List available time zones:
{% highlight shell %}
gcloud compute zones list
{% endhighlight %}

Set a time zone:
{% highlight shell %}
gcloud config set compute/zone us-west1-b
{% endhighlight %}

Set GOPATH:
{% highlight shell %}
echo "export GOPATH=~/go" >> ~/.bashrc
source ~/.bashrc
{% endhighlight %}

Get the code:
{% highlight shell %}
mkdir -p $GOPATH/src/github.com/udacity
cd $GOPATH/src/github.com/udacity
git clone https://github.com/udacity/ud615
cd ud615/app
{% endhighlight %}

On shell 1 - build the *monolith* app:
{% highlight shell %}
cd $GOPATH/src/github.com/udacity/ud615/app
mkdir bin
go build -o ./bin/monolith ./monolith
{% endhighlight %}

On shell 1 - run the *monolith* server:
{% highlight shell %}
$ sudo ./bin/monolith -http :10080
2017/11/22 11:40:51 Starting server...
2017/11/22 11:40:51 Health service listening on 0.0.0.0:81
2017/11/22 11:40:51 HTTP service listening on :10080
{% endhighlight %}

On shell 2 - test the app on shell 2:
{% highlight shell %}
$ curl http://127.0.0.1:10080
{"message":"Hello"}

$ curl http://127.0.0.1:10080/secure
authorization failed
{% endhighlight %}

On shell 2 - authenticate (password is *password*):
{% highlight shell %}
curl http://127.0.0.1:10080/login -u user
{% endhighlight %}

It prints out the token. You can copy and paste the long token in to the next command manually, but copying long, wrapped lines in cloud shell is broken. To work around this, you can either copy the JWT token in pieces, or, more easily, by assigning the token to a shell variable as follows.

On shell 2 - login and assign the value of the JWT to a variable
{% highlight shell %}
TOKEN=$(curl http://127.0.0.1:10080/login -u user | jq -r '.token')
echo $TOKEN
{% endhighlight %}

On shell 2 - access the secure endpoint using the JWT:
{% highlight shell %}
$ curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure
{"message":"Hello"}
{% endhighlight %}

On shell 2 - check out dependencies
{% highlight shell %}
ls vendor
cat vendor/vendor.json
{% endhighlight %}

On shell 1 - build and run the *hello* service
{% highlight shell %}
go build -o ./bin/hello ./hello
sudo ./bin/hello -http 0.0.0.0:10082 -health 0.0.0.0:10081
{% endhighlight %}

On shell 2 - build and run the *auth* service
{% highlight shell %}
go build -o ./bin/auth ./auth
sudo ./bin/auth -http :10090 -health :10091
{% endhighlight %}

On shell 3 - interact with the *auth* and *hello* microservices
{% highlight shell %}
TOKEN=$(curl 127.0.0.1:10090/login -u user | jq -r '.token')
curl -H "Authorization:  Bearer $TOKEN" http://127.0.0.1:10082/secure
{% endhighlight %}

## Building the Containers with Docker
Cloud shell - set compute/zone (Note: Google Cloud shell is an ephemeral instance and will reset if you don't use it for more than 30 minutes. That is why you might have to set some configuration values again)
{% highlight shell %}
gcloud compute zones list
gcloud config set compute/zone us-west1-c
{% endhighlight %}

Cloud shell - launch a new VM instance
{% highlight shell %}
$ gcloud compute instances create ubuntu --image-project ubuntu-os-cloud --image-family ubuntu-1604-lts
Created [https://www.googleapis.com/compute/v1/projects/artful-aardvark/zones/us-west1-c/instances/ubuntu].
NAME    ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
ubuntu  us-west1-c  n1-standard-1               10.138.0.2   35.197.47.143  RUNNING
{% endhighlight %}

Cloud shell - log into the VM instance
{% highlight shell %}
gcloud compute ssh ubuntu
{% endhighlight %}

VM instance - update packages and install nginx
{% highlight shell %}
sudo apt-get update
sudo apt-get install nginx
nginx -v
{% endhighlight %}

VM instance - start nginx
{% highlight shell %}
sudo systemctl start nginx
{% endhighlight %}

Check that it's running
{% highlight shell %}
sudo systemctl status nginx
curl http://127.0.0.1
{% endhighlight %}

Install Docker
{% highlight shell %}
sudo apt-get install docker.io
{% endhighlight %}

Check Docker images
{% highlight shell %}
sudo docker images
{% endhighlight %}

Pull nginx image
{% highlight shell %}
sudo docker pull nginx:1.10.0
sudo docker images
{% endhighlight %}

Verify the versions match
{% highlight shell %}
sudo dpkg -l | grep nginx
{% endhighlight %}

Run the first instance
{% highlight shell %}
sudo docker run -d nginx:1.10.0
{% endhighlight %}

Check if it's up
{% highlight shell %}
sudo docker ps
{% endhighlight %}

Run a different version of nginx
{% highlight shell %}
sudo docker run -d nginx:1.9.3
{% endhighlight %}

Run another version of nginx
{% highlight shell %}
sudo docker run -d nginx:1.10.0
{% endhighlight %}

Check how many instances are running
{% highlight shell %}
sudo docker ps
sudo ps aux | grep nginx
{% endhighlight %}

What's with the container names? If you don't specify a name, Docker gives a container a random name, such as "stoic_williams", "sharp_bartik", "awesome_murdock", or "evil_hawking". These are generated from a list of adjectives and names of famous scientists and hackers. The combination of the names and adjectives is random, except for one case. Want to see what the exception is? Check it out in the [Docker source code](https://github.com/moby/moby/blob/master/pkg/namesgenerator/names-generator.go)!

List all running container processes
{% highlight shell %}
sudo docker ps
{% endhighlight %}

For use in shell scripts you might want to just get a list of container IDs (`-a` stands for all instances, not just running, and `-q` is for "quiet" - show just the numeric ID):
{% highlight shell %}
sudo docker ps -aq
{% endhighlight %}

Inspect the container. You can use either **CONTAINER ID** or **NAMES** field, for example for a `sudo docker ps` output like this:
{% highlight shell %}
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f86cf066c304        nginx:1.10.0        "nginx -g 'daemon off"   8 minutes ago       Up 8 minutes        80/tcp, 443/tcp     sharp_bartik
{% endhighlight %}

You can use either of the following commands:
{% highlight shell %}
sudo docker inspect f86cf066c304
{% endhighlight %}

or
{% highlight shell %}
sudo docker inspect sharp_bartik
{% endhighlight %}

Connect to the nginx using the internal IP. Get the internal IP address either copying from the full inspect file or by assigning it to a shell variable:
{% highlight shell %}
{% raw %}
CN="sharp_bartik"
CIP=$(sudo docker inspect --format '{{ .NetworkSettings.IPAddress }}' $CN)
curl http://$CIP
{% endraw %}
{% endhighlight %}

You can also get all instance IDs and their corresponding IP addresses by doing this:
{% highlight shell %}
{% raw %}
sudo docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress }}' $(sudo docker ps -aq)
{% endraw %}
{% endhighlight %}

Stop an instance
{% highlight shell %}
sudo docker stop <cid>
{% endhighlight %}

or
{% highlight shell %}
sudo docker stop $(sudo docker ps -aq)
{% endhighlight %}

Verify no more instances running
{% highlight shell %}
sudo docker ps
{% endhighlight %}

Remove the docker containers from the system
{% highlight shell %}
sudo docker rm <cid>
{% endhighlight %}

or
{% highlight shell %}
sudo docker rm $(sudo docker ps -aq)
{% endhighlight %}

On the VM Instance, build a static binary of the *monolith* app
{% highlight shell %}
$ cd $GOPATH/src/github.com/udacity/ud615/app/monolith
$ go get -u
$ go build --tags netgo --ldflags '-extldflags "-lm -lstdc++ -static"'

$ ldd monolith
       not a dynamic executable
{% endhighlight %}

Why did you have to build the binary with such an ugly command line? You have to explicitly make the binary static. This is really important in the Docker community right now because alpine has a different implementation of libc. So your go binary wouldn't have had the lib it needed if it wasn't static. You created a static binary so that your application could be self-contained.

Create a container for the app. Look at the Dockerfile
{% highlight shell %}
cat Dockerfile
{% endhighlight %}

Build the app container
{% highlight shell %}
sudo docker build -t monolith:1.0.0 .
{% endhighlight %}

List the *monolith* image
{% highlight shell %}
sudo docker images monolith:1.0.0
{% endhighlight %}

Run the *monolith* container and get it's IP
{% highlight shell %}
sudo docker run -d monolith:1.0.0
sudo docker inspect <container name or cid>
{% endhighlight %}

or
{% highlight shell %}
{% raw %}
CID=$(sudo docker run -d monolith:1.0.0)
CIP=$(sudo docker inspect --format '{{ .NetworkSettings.IPAddress }}' ${CID})
{% endraw %}
{% endhighlight %}

Test the container
{% highlight shell %}
curl <the container IP>
{% endhighlight %}

or
{% highlight shell %}
curl $CIP
{% endhighlight %}

Important note on security: If you are tired of typing "sudo" in front of all Docker commands, and confused why a lot of examples don't have that, please read the following article about implications on security - [Why we don't let non-root users run Docker in CentOS, Fedora, or RHEL](http://www.projectatomic.io/blog/2015/08/why-we-dont-let-non-root-users-run-docker-in-centos-fedora-or-rhel/)

Create docker images for the remaining microservices - auth and hello.

Build the *auth* app
{% highlight shell %}
cd $GOPATH/src/github.com/udacity/ud615/app
cd auth
go build --tags netgo --ldflags '-extldflags "-lm -lstdc++ -static"'
sudo docker build -t auth:1.0.0 .
CID2=$(sudo docker run -d auth:1.0.0)
{% endhighlight %}

Build the *hello* app
{% highlight shell %}
cd $GOPATH/src/github.com/udacity/ud615/app
cd hello
go build --tags netgo --ldflags '-extldflags "-lm -lstdc++ -static"'
sudo docker build -t hello:1.0.0 .
CID3=$(sudo docker run -d hello:1.0.0)
{% endhighlight %}

See the running containers
{% highlight shell %}
sudo docker ps -a
{% endhighlight %}

Public Vs Private Registries ([Comparing Four Hosted Docker Registries](http://rancher.com/comparing-four-hosted-docker-registries/)):
* [Docker Hub](https://hub.docker.com/)
* [Quay](https://quay.io/) is another popular registry because of it’s rich automated workflow for building containers from github.
* [Google Cloud Registry (GCR)](https://cloud.google.com/container-registry/docs/) is a strong options for large enterprises.

See all images
{% highlight shell %}
sudo docker images
Docker tag command help
docker tag -h
{% endhighlight %}

Add your own tag
{% highlight shell %}
sudo docker tag monolith:1.0.0 <your username>/monolith:1.0.0
{% endhighlight %}

For example (you can rename too!)
{% highlight shell %}
sudo docker tag monolith:1.0.0 udacity/example-monolith:1.0.0
{% endhighlight %}

Create account on Dockerhub. To be able to push images to Dockerhub you need to [create an account there](https://hub.docker.com/register/)

Login and use the docker push command
{% highlight shell %}
sudo docker login
sudo docker push udacity/example-monolith:1.0.0
{% endhighlight %}

Repeat for all images you created - *monolith*, *auth* and *hello*!

## Kubernetes
* [Kubernetes command cheat sheet](http://kubernetes.io/docs/user-guide/kubectl-cheatsheet/)
* [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
* [Configure Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
* [Configure Containers Using a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/)
* [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Services](http://kubernetes.io/docs/user-guide/services/)

Use project directory
{% highlight shell %}
cd $GOPATH/src/github/com/udacity/ud615/kubernetes
{% endhighlight %}
**Note:** At any time you can clean up by running the *cleanup.sh* script

Provision a Kubernetes Cluster with GKE using gcloud (GKE is a hosted Kubernetes by Google; GKE clusters can be customized and supports different machine types, number of nodes, and network settings)
{% highlight shell %}
$ gcloud container clusters create k0 --zone=us-west1-c

$ gcloud compute instances list
NAME                               ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-k0-default-pool-1f7bf2a3-3gjp  us-west1-c  n1-standard-1               10.138.0.4   35.203.166.36   RUNNING
gke-k0-default-pool-1f7bf2a3-3twg  us-west1-c  n1-standard-1               10.138.0.2   35.199.167.224  RUNNING
gke-k0-default-pool-1f7bf2a3-80gb  us-west1-c  n1-standard-1               10.138.0.3   35.203.166.97   RUNNING
{% endhighlight %}

Launch a single instance:
{% highlight shell %}
kubectl run nginx --image=nginx:1.10.0
{% endhighlight %}

Get pods
{% highlight shell %}
kubectl get pods
{% endhighlight %}

Expose nginx
{% highlight shell %}
kubectl expose deployment nginx --port 80 --type LoadBalancer
{% endhighlight %}

List services
{% highlight shell %}
$ kubectl get services
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.47.240.1     <none>          443/TCP        8m
nginx        LoadBalancer   10.47.253.101   35.197.110.73   80:30244/TCP   1m

$ curl http://35.197.110.73/
{% endhighlight %}

Explore config file
{% highlight shell %}
cat pods/monolith.yaml
{% endhighlight %}

Create the *monolith* pod
{% highlight shell %}
kubectl create -f pods/monolith.yaml
{% endhighlight %}

Examine pods
{% highlight shell %}
kubectl get pods
{% endhighlight %}
It may take a few seconds before the monolith pod is up and running as the monolith container image needs to be pulled from the Docker Hub before we can run it.

Use the **kubectl describe** command to get more information about the monolith pod
{% highlight shell %}
kubectl describe pods monolith
{% endhighlight %}

On cloud shell 1 - set up port-forwarding
{% highlight shell %}
kubectl port-forward monolith 10080:80
{% endhighlight %}

On cloud shell 2
{% highlight shell %}
curl http://127.0.0.1:10080
curl http://127.0.0.1:10080/secure
{% endhighlight %}

On cloud shell 2 - log in
{% highlight shell %}
curl -u user http://127.0.0.1:10080/login
curl -H "Authorization: Bearer <token>" http://127.0.0.1:10080/secure
{% endhighlight %}

On cloud shell 2 - view logs
{% highlight shell %}
kubectl logs monolith
kubectl logs -f monolith
{% endhighlight %}

On cloud shell 3
{% highlight shell %}
curl http://127.0.0.1:10080
{% endhighlight %}

On cloud shell 2 - exit log watching (Ctrl-C)

You can use the **kubectl exec** command to run an interactive shell inside the monolith Pod. This can come in handy when you want to troubleshoot from within a container:
{% highlight shell %}
kubectl exec monolith --stdin --tty -c monolith /bin/sh
{% endhighlight %}

For example, once we have a shell into the monolith container we can test external connectivity using the ping command.
{% highlight shell %}
ping -c 3 google.com
{% endhighlight %}

When you’re done with the interactive shell be sure to logout.
{% highlight shell %}
exit
{% endhighlight %}

Creating Secrets
{% highlight shell %}
ls tls
{% endhighlight %}
The `cert.pem` and `key.pem` files will be used to secure traffic on the monolith server and the `ca.pem` will be used by HTTP clients as the CA to trust. Since the certs being used by the monolith server where signed by the CA represented by `ca.pem`, HTTP clients that trust `ca.pem` will be able to validate the SSL connection to the monolith server.

Use kubectl to create the tls-certs secret from the TLS certificates stored under the tls directory:
{% highlight shell %}
kubectl create secret generic tls-certs --from-file=tls/
{% endhighlight %}

kubectl will create a key for each file in the tls directory under the tls-certs secret bucket. Use the **kubectl describe** command to verify that:
{% highlight shell %}
kubectl describe secrets tls-certs
{% endhighlight %}

Next we need to create a configmap entry for the `proxy.conf` nginx configuration file using the **kubectl create configmap** command:
{% highlight shell %}
kubectl create configmap nginx-proxy-conf --from-file=nginx/proxy.conf
{% endhighlight %}

Use the **kubectl describe configmap** command to get more details about the nginx-proxy-conf configmap entry:
{% highlight shell %}
kubectl describe configmap nginx-proxy-conf
{% endhighlight %}

Accessing A Secure HTTPS Endpoint
{% highlight shell %}
cat pods/secure-monolith.yaml
{% endhighlight %}

Create the secure-monolith Pod using kubectl:
{% highlight shell %}
kubectl create -f pods/secure-monolith.yaml
kubectl get pods secure-monolith

kubectl port-forward secure-monolith 10443:443

curl --cacert tls/ca.pem https://127.0.0.1:10443

kubectl logs -c nginx secure-monolith
{% endhighlight %}

Create a service:
{% highlight shell %}
cat services/monolith.yaml
kubectl create -f services/monolith.yaml
{% endhighlight %}

Set up firewall rules for the service port(s)
{% highlight shell %}
gcloud compute firewall-rules create allow-monolith-nodeport --allow=tcp:31000
{% endhighlight %}

Get IP addresses of the nodes
{% highlight shell %}
gcloud compute instances list

curl -k https://104.197.223.141:31000
{% endhighlight %}
Not working yet!

Add labels to Pods:
{% highlight shell %}
kubectl get pods -l "app=monolith"
kubectl get pods -l "app=monolith,secure=enabled"
kubectl describe pods secure-monolith | grep Labels
kubectl label pods secure-monolith "secure=enabled"
kubectl describe pods secure-monolith | grep Labels
kubectl describe services monolith | grep Endpoints
curl -k https://104.197.223.141:31000
{% endhighlight %}
Now it works!

## Deploying Microservices
Create deployments:
{% highlight shell %}
cat deployments/auth.yaml
kubectl create -f deployments/auth.yaml
kubectl describe deployments auth
kubectl create -f services/auth.yaml

kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml

kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
kubectl get services frontend
curl -k https://104.197.163.161
{% endhighlight %}

Scale deployments:
{% highlight shell %}
kubectl get replicasets
kubectl get pods -l "app=hello,track=stable"
vim deployments/hello.yaml
kubectl apply -f deployments/hello.yaml
kubectl get replicasets
kubectl get pods
kubectl describe deployment hello
kubectl get services
{% endhighlight %}

Rolling updates:
{% highlight shell %}
vim deployments/auth.yaml
kubectl apply -f deployments/auth.yaml
kubectl describe deployments auth
kubectl get pods
{% endhighlight %}

Finally, delete the Kubernetes cluster:
{% highlight shell %}
$ gcloud container clusters list
NAME  ZONE        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
k0    us-west1-c  1.7.8-gke.0     35.199.190.169  n1-standard-1  1.7.8-gke.0   3          RUNNING

$ gcloud container clusters delete k0 --zone=us-west1-c
{% endhighlight %}
