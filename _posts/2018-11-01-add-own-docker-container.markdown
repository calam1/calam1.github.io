---
title:  "Adding your own Docker Container to the Istio Sample"
date:   2018-11-01 20:18:00
description: Running your own Docker Container with the sample (BookInfo) onto Kubernetes, Istio, and Minikube
---

So now that you set up and ran the [BookInfo][bookInfo-url] example from the previous post; you may want to deploy a Docker container of your own onto the Kubernetes setup.  Doing this will give you a little more insight on how Kubernetes and Istio work.

I assume you know how to create a Docker image.  If not, there are tons of tutorials out there.  

Trick here is to get the Docker image on your laptop onto the Kubernetes Docker repository.  Once you have created your Docker image,  We need to copy it over to the Kubernetes environment with the following command posted. Replace "your_docker_image:latest" with your image name and version


{% highlight ruby %}
> docker save your_docker_image:latest | ssh -o UserKnownHostsFile=/dev/null \
    -o StrictHostKeyChecking=no -o LogLevel=quiet \
    -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) docker load
{% endhighlight %}


Now, lets switch to the Docker instance that Kubernetes is running(I believe it is acutally sharing the same Docker Daemon underneath the covers)
{% highlight ruby %}
> eval $(minikube docker-env)
{% endhighlight %}


Run the docker images command and you should see the image you just copied over


When you want to switch back to your local docker instance just run
{% highlight ruby %}
> eval $(minikube docker-env -u)
{% endhighlight %}


Now you need to create a new yaml file with the following contents, replace "my-own-app" and "my-own-app-image" with your app name and docker image name.  Also replace the port number with your port number.
{% highlight ruby %}
kind: Service
apiVersion: v1
metadata:
  name: my-own-app
  labels:
    app: my-own-app
spec:
  type: NodePort
  selector:
    app: my-own-app
  ports:
  - port: 7001
    targetPort: 7001
    name: http
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: my-own-app
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-own-app
        version: v1
    spec:
      containers:
      - name: my-own-app
        image: my-own-app-image
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 7001
---
{% endhighlight %}


Run the following command to deploy your app, obviouly replacing the my-own-app.yaml file with whatever file you created.
{% highlight ruby %}
> kubectl app -f my-own-app.yaml
{% endhighlight %}


Now just get the IP address of the minikube
{% highlight ruby %}
> minikube ip
{% endhighlight %}

and use that as your IP address.

Run the following to find the forwarding port

{% highlight ruby %}
> kubeclt get services 
{% endhighlight %}


Find the line of your service and find the port, i.e. 7001:30189, the 30189 port number is the external port.


Note: that since we are running minikube, we need to use nodePort in the above yaml file to expose the service.

[bookInfo-url]:https://brew.sh://istio.io/docs/examples/bookinfo/ 
