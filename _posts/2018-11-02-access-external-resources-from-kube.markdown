---
title:  "Accessing an External Resource from Kubernetes"
date:   2018-11-01 22:11:00
description: How to access an external resource like a database from Kubernetes
---
Sometimes not all of your services are running on a Kubernetes cluster.  Maybe your database is sitting on a server external to your cluster, since you do not want it to be ephemeral. In that case how does a service within Kubernetes connect to it?  You just need to create a couple of yaml files and apply them.  Now this may not be the optimal or most restrictive approach, since I do not define selectors; but it should be enough to get you started and getting your app to communicate to your external database.


Create a yaml file and add the following(replace the appropriate info, like the DNS, port, etc)
{% highlight ruby %}
kind: Service
apiVersion: v1
metadata:
  name: external-db
  namespace: default
spec:
  type: ExternalName
  externalName: my.database.com
  ports:
 		- port: 1521
  protocol: TCP
  targetPort: 1521
{% endhighlight %}

{% highlight ruby %}
> kubectl apply -f my-service.yaml
{% endhighlight %}

Create another yaml file and add the following
{% highlight ruby %}
 kind: Endpoints
 apiVersion: v1
  metadata:
    name: external-db
    namespace: default
{% endhighlight %}


{% highlight ruby %}
> kubectl apply -f my-endpoint.yaml
{% endhighlight %}

Now your application should be able to access the external database.
