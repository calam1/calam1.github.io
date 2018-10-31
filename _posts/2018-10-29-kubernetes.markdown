---
title:  "Setting up Kubernetes and Istio "
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
{% highlight ruby %}
// since we are using VirtualBox we don't have to set the driver; I believe VirtualBox is the default
> minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.10.0 
{% endhighlight %}

**Install Istio onto Minikube**

So when you downloaded Istio via the cURL command in the previous steps it downloaed some additional code and examples.  Change into the istio directory i.e.
{% highlight ruby %} 
> cd ~/git/istio-1.0.3
{% endhighlight %}

*Install Istio's [Custom Resource Definitions][custom-resource-definitions-url]*
{% highlight ruby %}
> kubectl apply -f install/kubernetes/helm/istion/templates/crds.yaml
{% endhighlight %}

**T0 BE CONTINUED**

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll's GitHub repo][jekyll-gh].

[brew-link]: https://brew.sh/
[minikube-github]: https://github.com/kubernetes/minikube
[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
[vbox-url]: https://www.virtualbox.org/wiki/Downloads
[article-1-url]: https://medium.com/@nicdoye/minikube-without-virtualbox-4b521601ce57
[article-2-url]: https://gist.github.com/inadarei/7c4f4340d65b0cc90d42d6382fb63130
[custom-resource-definitions-url]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions
[istio-quickstart-url]: https://istio.io/docs/setup/kubernetes/quick-start/
