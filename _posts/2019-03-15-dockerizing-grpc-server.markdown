---
title: "Dockerizing your gRPC Server and Testing it with grpcurl - Part 3"
date: 2019-03-15 15:14:00
description: "Dockerizing your gRPC server and testing it with grpcurl"
---

I created another directory and just copied some of the code from the last tutorial.  I didn't want to have to deal with the client and moving things around.  The code for this [tutorial][grpc-tutorial-part3] is in [github][grpc-tutorial-part3].

**PreSteps**

Make sure your mysql docker container is running (you need to run it in bridged mode), if you are using docker.  If you do not run this in bridged mode your gRPC server container will not be able to communicate with the docker container running mysql.

{% highlight python %}
> docker run --name=mysql -d --net=bridge mysql/mysql-server
{% endhighlight %}

This is kind of crappy and lazy on my part; but if you remember I hardcoded the IP address of the mySQL container in the server code.  You should use go flags to set this up.  As they say, don't do as I do, do as I say ;)


{% highlight python %}
// hardcoded IP of user/pw/db IP - use go flags and pass these values in; but this will suffice as a toy example
// if you start the mysql docker container first, the IP address will always be 172.17.0.2 - at least this has been my experience on Mac OSX and Linux

dsUser := "chris:password@tcp(172.17.0.2:3306)/customers?User"
{% endhighlight %}


**Dockerfile**

Here is the dockerfile I created for this example.  I am not going to explain the details of how this Dockerfile works.  There are some good resources out there.  I used [this][min-golang-dockerfile] as my guide.


{% highlight python %}
FROM golang:1.11.4 as builder                                                                                                                                                                                                     

# Create the user and group files that will be used in the running container to
# run the process as an unprivileged user.
RUN mkdir /user && \
    echo 'nobody:x:65534:65534:nobody:/:' > /user/passwd && \
    echo 'nobody:x:65534:' > /user/group
    

WORKDIR /home/calam/git/gomodules/grpc-tutorial/user

COPY go.mod .

COPY go.sum .

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build \
    -ldflags="-w -s" \
    -o /app .
    
# Final stage: the running container.
FROM scratch AS final

# Import the user and group files from the first stage.
COPY --from=builder /user/group /user/passwd /etc/


# Import the compiled executable from the first stage.
COPY --from=builder /app /app

EXPOSE 50051

                                                                                                                 
# Perform any further action as an unprivileged user.                                                             
USER nobody:nobody                                                                                               
                                                                                                                   
# Run the compiled binary.                                                                                       
ENTRYPOINT ["/app"]
{% endhighlight %}

Now start up the gRPC server container

{% highlight python %}
> docker run --name=grpc-example --net=bridge -d -p 50051:50051 user-server:latest
{% endhighlight %}

Now you should have 2 docker containers running.  

**Testing your container deployment**

The tricky thing about gRPC is that it is not REST and uses the http2 protocol.  So running a basic curl out of the box does not work.  I found this great open source [grpc curl tool][grpc-curl-github]; just follow the install directions.  I had a little problem due to some messed up GOPATH setups.  Golang setup is for some reason still the bane of my existence, and I need to clean up my linux box set up.


So let's see it in action!

{% highlight python %}
// list services on gRPC server
> grpcurl -plaintext localhost:50051 list
UserService
grpc.reflection.v1alpha.ServerReflection

// describe a service
> grpcurl -plaintext localhost:50051 describe UserService
UserService is a service:
service UserService {
  rpc Create ( .CreateUserRequest ) returns ( .CreateUserResponse );
  rpc Delete ( .DeleteUserRequest ) returns ( .DeleteUserResponse );
  rpc Read ( .ReadUserRequest ) returns ( .ReadUserResponse );
}

// create a user, plaintext is similar or the same to --insecure with cURL
> grpcurl -plaintext -d '{"user": {"firstName": "Chris","lastName": "Lam", "address": "123 Main St"}}' localhost:50051 UserService/Create
{
  "id": 1
}

// read from the service
> grpcurl -plaintext -d '{"id": 1}' localhost:50051 UserService/Read
{
  "user": {
    "id": 1,
    "firstName": "Chris",
    "lastName": "Lam",
    "address": "123 Main St"
  }
}

{% endhighlight %}


Well this concludes my tutorial.  Hope you got a little insight and value from it.  Stay tuned for my next tutorial, it may involve this codebase.  I am thinking about securing the endpoints, instead of having the insecure.  It is something I need to learn to do and this is a nice segue to it.

Thanks for reading!

[min-golang-dockerfile]: https://pierreprinetti.com/blog/2018-the-go-1.11-web-service-dockerfile/
[grpc-curl-github]: https://github.com/fullstorydev/grpcurl
[grpc-tutorial-part3]: https://github.com/calam1/grpc-tutorial/tree/master/user-server/server
