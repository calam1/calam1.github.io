---
title:  "Accessing an External Resource from Kubernetes"
date:   2018-11-01 22:11:00
description: How to access an external resource like a database from Kubernetes
---

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
