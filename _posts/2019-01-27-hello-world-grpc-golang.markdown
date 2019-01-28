---
title:  "gRPC from Hello World to Kubernetes - Part 1"
date:   2019-01-27 19:04:00
description: Tutorial on using gRPC using Go to deploying it on Kubernetes"
---
I had the privilege to attend Code One(previously named Java One) a few months ago.  And I saw a presentation about gRPC.  To be honest I have never heard of it before.  But it sounded interesting and I wanted to spend some time learning it.  I finally carved out some time to play with gRPC and put my findings on my blog.  This will be a multipart post.  I am not sure how many parts, but I am thinking 3 parts or so.  I will start with a Hello World example, and will attempt to explain some details about gRPC usage in Go.  The next step will be a simple application interacting with a database.  And the last piece will be deploying to Kubernetes.  As stated the first post will be a simple Hello World example.

I will not go into any particulars of Golang, so there will be some assumption that you understand Go enough to do this tutorial.  If not, there are plenty of good resources out there.  I will provide some precursory information about what versions of software I am using, etc.

I have moved on from using a Macbook Pro for my personal development.  So unlike my prior tutorial, I will be using Antergos, which is an Arch Linux Installer, but calling it an installer is arguable.
* Golang 1.11
* Protocol Buffers v3.6.1-linux-x86_64

&nbsp;

**Protocol Buffers and Golang Proto Code Generator**

Before we move on, you will want to install the Protocol Buffers binaries and compiler and the Golang code generator plugin .  

To install the go Protocol Buffer binaries and compiler
I just downloaded the precompiled binary from [here][protocol-buffer-binary-url].  Place this anywhere you like and add the bin to your PATH and if necessary source your shell.
i.e.
{% highlight ruby %}
export PATH=/home/calam/git/protobuf/bin:$PATH
{% endhighlight %}

To install the Golang code generator
{% highlight ruby %}
> go get -u github.com/golang/protobuf/protoc-gen-go
{% endhighlight %}

**Project Setup**

I am using Go modules, which are available starting from Golang 1.11.  So create a directory named *helloworld* somewhere outside of your [GOPATH ][gopath-url].  Go to that directory and run the following:
{% highlight ruby %}
// where <username> is your name, etc
> go mod init github.com/<username>/helloworld
{% endhighlight %}

**Protocol Buffer Creation**

Under the *helloworld* directory create a new directory named *proto*.  Change into that directory and create a file named *helloworld.proto* and type or cut and paste the contents below into that file and save it.
{% highlight ruby %}
syntax="proto3";

package proto;

message HelloWorldRequest {
    string name = 1;
}

message HelloWorldResponse {
    string msg = 1;
}

service HelloWorldService  {
    rpc SayHelloWorld (HelloWorldRequest) returns (HelloWorldResponse);
}
{% endhighlight %}

**Golang Code Generation**

Create the generated class from the proto file.   I ran the following command in the *helloworld* directory

I believe the *-I* is the flag for the path then helloworld.proto is the name of the protocol buffer file.  The *--go_out*, is the output path for the golang language.  *plugins=grpc* tells the command what executable to use and the last *proto* is the output directory.  Please feel free to correct me if I am wrong about any of these values.
{% highlight ruby %}
> protoc -I proto/ helloworld.proto --go_out=plugins=grpc:proto
{% endhighlight %}

Take look at your helloworld directory you should see a generated go class named helloworld.pb.go and your directory structure should be pretty similar to what's shown.
{% highlight ruby%}
 grpc-tutorial
     └── helloworld
	├── go.mod
        ├── go.sum
        └── proto
      		├── helloworld.pb.go
        	└── helloworld.proto
{% endhighlight %}

**gRPC Server**

Now we need to create a grpc server.  Create another directory named *helloworld_server* under the *helloworld* directory.  In that *helloworld_server* directory create a file name *main.go* and place the following contents in the file.
{% highlight ruby %}
package main                                                                                                                                                                                                           

import (                                                                                
	"context"                                                                                                                 
	"log"                                                                       
	"net"

	pb "github.com/calam1/helloworld/proto"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"                                            
)                                                                                                                            

type server struct{}                                     

func (s *server) SayHelloWorld(ctx context.Context, req *pb.HelloWorldRequest) (*pb.HelloWorldResponse, error) {
	log.Printf("Received: %s", req.Name)
	return &pb.HelloWorldResponse{Msg: "Hello " + req.Name}, nil                                           
}                                                 

func main() {                                                                                                    
	listener, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listener failed:%v:", err)
	}

	grpcServer := grpc.NewServer()
	pb.RegisterHelloWorldServiceServer(grpcServer, &server{})                                                                                                                                                            
	// register reflection service on gRPC server                                                                                                                                                                      
	reflection.Register(grpcServer)                                                                                                                                                                                     
	if err := grpcServer.Serve(listener); err != nil {
		log.Fatalf("failed to server: %v", err)
	}
}
{% endhighlight %}

**gRPC Client**

Create a client to test the helloworld service.  Create another directory under "helloworld" named "helloworld_client".  Create a file named main.go and add the following code
{% highlight ruby %}
package main                                                                                                                                                                                                           

import (                                                                                
	"context"
	"log"
	"os"
	"time"                                                                                                                     

	pb "github.com/calam1/helloworld/proto"
	"google.golang.org/grpc"                                                       
)                                                                                                                            

const (                                                  
	address     = "localhost:50051"
	defaultName = "default_name"
)

func main() {                                     
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {                                                                                                
		log.Fatalf("connection failed: %v", err)                                  
	}
	defer conn.Close()
	client := pb.NewHelloWorldServiceClient(conn)
	name := defaultName
	if len(os.Args) > 1 {
			name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	response, err := client.SayHelloWorld(ctx, &pb.HelloWorldRequest{Name: name})
	if err != nil {
		log.Fatalf("cannot contact helloworld server: %v", err)
	}

	log.Printf("Response is: %s", response.Msg)
}
{% endhighlight %}

**Now run it!**

{% highlight ruby %}
> cd helloworld // the parent directory
> go run helloworld_server/main.go 
{% endhighlight %}

Open another terminal
{% highlight ruby %}
> cd helloworld // the parent directory 
> go run helloworld_client/main.go Chris
{% endhighlight %}

{% highlight ruby %}
// server should see something like this
> 2019/01/26 18:37:10 Received: Chris 

// client should see something like this
> 2019/01/26 18:37:10 Response is: Hello Chris
{% endhighlight %}


**Things of note when implementing a gRPC app in Go**

*Implementing the Interfaces*

* When you generate the code from the protoc command, you can search the output file(helloworld.pb.go) for the word "interface" to show you what interfaces you will need to implement.
{% highlight ruby %}
type HelloWorldServiceServer interface {
  SayHelloWorld(context.Context, *HelloWorldRequest) (*HelloWorldResponse, error)
}
{% endhighlight %}

* in this example we implement this method in the helloworld_server/main.go class
* since this is Go, we use duck typing.  So basically implement that SayHelloWorld method 
* we implemented this method in the main.go under *helloworld_server* directory

{% highlight ruby %}
type server struct{}

func (s *server) SayHelloWorld(ctx context.Context, req *pb.HelloWorldRequest) (*pb.HelloWorldResponse, error) {
  log.Printf("Received: %s", req.Name)
  return &pb.HelloWorldResponse{Msg: "Hello " + req.Name}, nil
}
{% endhighlight %}

*Create a grpc server, and this is probably the most basic way to do this*

{% highlight ruby %}
func main() {
	listener, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listener failed:%v:", err)
	}

  grpcServer := grpc.NewServer()
  pb.RegisterHelloWorldServiceServer(grpcServer, &server{})
  reflection.Register(grpcServer)
  if err := grpcServer.Serve(listener); err != nil {
  	log.Fatalf("failed to server: %v", err)
  }
}
{% endhighlight %}

*Set up a basic client*

{% highlight ruby %}
func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("connection failed: %v", err)
	}

  defer conn.Close()
  
  client := pb.NewHelloWorldServiceClient(conn)
  
  name := defaultName
  if len(os.Args) > 1 {
  	name = os.Args[1]  
  }
  
  ctx, cancel := context.WithTimeout(context.Background(), time.Second)
  defer cancel()
  
  response, err := client.SayHelloWorld(ctx, &pb.HelloWorldRequest{Name: name})
  if err != nil {
  	log.Fatalf("cannot contact helloworld server: %v", err)
  }
  
  log.Printf("Response is: %s", response.Msg)
}                                     
{% endhighlight %}

[gopath-url]: https://github.com/golang/go/wiki/GOPATH
[protocol-buffer-binary-url]: https://github.com/protocolbuffers/protobuf/releases
