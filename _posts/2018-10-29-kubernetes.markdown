---
title:  "Setting up Kubernetes and Istio on Minikube"
date:   2018-10-29 10:18:00
description: Setting up Kubernetes, Minikube and Istio on a Mac
---

This is another kubernetes/istio tutorial.  I have found a few tutorials out there, but some of them are out of date, due to some breaking changes in kubernetes and istio. So I decided to document this, for myself, if for nobody else. I used the [quickstart][istio-quickstart-url] as the basis of this article.  So if you find that easier to follow, please do so.

First let me lay out the versions of my Mac OS and the versions of kubernetes, minikube and istio that I am using/installing
* MacOS: 10.13,6 (High Sierra)
* kubernetes(client): 1.12.0 
* kubernetes(server): 1.10.0
* minikube: v0.30.0
* istio: 1.0.3

&nbsp;
&nbsp;

**Install a Hypervisor**

Minikube needs this to virtualize a cluster on your local machine.  I use [VirtualBox][vbox-url], download the mac os version.
Now, I know VirtualBox kind of sucks, so you can try and use Docker/Xhyve.  I had issues getting it to work, but here are a couple of articles; if you are interested in giving it a shot
* [article 1][article-1-url]
* [article 2][article-2-url]

&nbsp;

**Install kubectl** (I use [brew][brew-link] to manage this)
{% highlight ruby %}
> brew install kubernetes-cli
{% endhighlight %}

**Then check the version**
{% highlight ruby %}
> kubectl version
Client Version: version.Info{Major:"1", Minor:"12", GitVersion:"v1.12.0", GitCommit:"0ed33881dc4355495f623c6f22e7dd0b7632b7c0", GitTreeState:"clean", BuildDate:"2018-09-28T15:20:58Z", GoVersion:"go1.11", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-03-26T16:44:10Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
{% endhighlight %}

**Install [minikube][minikube-github]**
{% highlight ruby %}
> brew cask install minikube
{% endhighlight %}

**Set minikube context**
{% highlight ruby %}
> kubectl config use-context minikube
Context "minikube" modified.
{% endhighlight %}

**Download and Setup Istio**
{% highlight ruby %}
> curl -L https://git.io/getLatestIstio | sh -
{% endhighlight %}
You will get a message to export the path,just copy the command and run it
{% highlight ruby %}
> export PATH="$PATH:/Users/christopherlam/git/development/istio-1.0.3/bin"
{% endhighlight %}

**Start up Minikube**

Since we are using VirtualBox we don't have to set the driver; I believe VirtualBox is the default
{% highlight ruby %}

> minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.10.0 
{% endhighlight %}

**Install Istio onto Minikube**

So when you downloaded Istio via the cURL command in the previous steps it downloaed some additional code and examples.  Change into the istio directory i.e.
{% highlight ruby %} 
> cd ~/git/istio-1.0.3
{% endhighlight %}

*Install Istio's [Custom Resource Definitions][custom-resource-definitions-url]*

CRDs are one way to define resources, [Aggregated APIs][aggregated-api-url] is the other way
{% highlight ruby %}
> kubectl apply -f install/kubernetes/helm/istion/templates/crds.yaml
{% endhighlight %}

*Install Istio's Core Components*

I chose to install without mutual TLS authentication between sidecars. There are other options, such as installing with TLS authentication, using Helm, using Helm and Tiller, etc.
{% highlight ruby %}
> kubectl apply -f install/kubernetes/istio-demo.yaml
{% endhighlight %}

*Verify the installation*

{% highlight ruby %}
> kubectl get svc -n istio-system
{% endhighlight %}
You should see services like istio-ingressgateway, istio-telemetry, istio-sidecar-injector, etc

Since we are using minikube, it does not support an external load balancer. The EXTERNAL-IP of istio-ingress and istio-ingressgateway will say <pending>. You will need to access it using the service NodePort, or use port-forwarding instead.

*Install the sample project [BookInfo][bookInfo-url]*

{% highlight ruby %}
> kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
{% endhighlight %}

*Confirm that the services are running*

{% highlight ruby %}
> kubectl get services
NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP              PORT(S)          AGE
details                           ClusterIP      10.108.0.216     <none>                   9080/TCP         4d
kubernetes                        ClusterIP      10.96.0.1        <none>                   443/TCP          4d
productpage                       ClusterIP      10.96.126.187    <none>                   9080/TCP         4d
ratings                           ClusterIP      10.106.50.27     <none>                   9080/TCP         4d
reviews                           ClusterIP      10.109.237.86    <none>                   9080/TCP         4d
{% endhighlight %}

*Confirm pods are running*

{% highlight ruby %}
> kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
details-v1-6865b9b99d-vzz82                        2/2     Running   1          4d
productpage-v1-f8c8fb8-6bldf                       2/2     Running   1          4d
ratings-v1-77f657f55d-66m2t                        2/2     Running   1          4d
reviews-v1-6b7f6db5c5-8gmsg                        2/2     Running   1          4d
reviews-v2-7ff5966b99-hfxcd                        2/2     Running   1          4d
reviews-v3-5df889bcff-2pqdd                        2/2     Running   1          4d
{% endhighlight %}

*Now we need to make the app accessible*

{% highlight ruby %}
> kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
{% endhighlight %}

*Confirm the gateway*

{% highlight ruby %}
> kubectl get gateway
NAME               AGE
bookinfo-gateway   64s
{% endhighlight %}

*Determine the IP address*

Since you are running minikube there is no external load balancer, you can find the IP address of minikube by running the following command
{% highlight ruby%}
> minikube ip
192.168.99.100
{% endhighlight %}

*Determine the port*

This command will return the port number
{% highlight ruby %}
> kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
{% endhighlight %}

Now armed with the IP address and port you can hit the browser with the following url:

http://ip-address:port/productpage
i.e. http://192.168.99.100:31380/productpage


[brew-link]: https://brew.sh/
[minikube-github]: https://github.com/kubernetes/minikube
[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
[vbox-url]: https://www.virtualbox.org/wiki/Downloads
[article-1-url]: https://medium.com/@nicdoye/minikube-without-virtualbox-4b521601ce57
[article-2-url]: https://gist.github.com/inadarei/7c4f4340d65b0cc90d42d6382fb63130
[custom-resource-definitions-url]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions
[istio-quickstart-url]: https://istio.io/docs/setup/kubernetes/quick-start/
[aggregate-api-url]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#api-server-aggregation
[bookInfo-url]: https://istio.io/docs/examples/bookinfo/
