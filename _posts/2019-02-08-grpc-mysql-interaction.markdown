---
title: "gRPC and MySql Part 2"
date: 2019-02-08 15:15:00
description: "A simple golang gRPC example interacting with a database"
---
Firstly, make sure you completed the MySql setup that was presented in the last entry. Then we have a little context of the simple user table we will interact with. Or if you rather just run the database locally instead of docker that is fine also; up to you.

Ok some of this will sound familiar to the HelloWorld blog entry as far as the process of creating proto file, etc.  So please bear with me.

So let's create our protobuf file for our User table.  The code is all in [GitHub][github-url].  I created a *user* directory and another directory named *proto*


{% highlight ruby %}
> mkdir -p user/proto
{% endhighlight %}

**Define user.proto**

Now create a file named *user.proto* and add the following content.  This is a simple insert, read, and delete example.  You will see we have requests and responses for all 3 actions.  These requests and responses have the makeup of what is required in the request and what is returned in the response.  You will also see in the bottom of *user.proto* the rpc calls we define.

{% highlight json %}
syntax= "proto3";                                                                                                                                                                                                                            
                                
message User {
  int64 id = 1;
  string firstName = 2;
  string lastName = 3;
  string address = 4;                                  
}                                          

message CreateUserRequest {
  User user = 1;
}

message CreateUserResponse {
  int64 id = 1;
}

message ReadUserRequest {
  int64 id = 1;
}                                   

message ReadUserResponse {
  User user = 1;
}                                   

message DeleteUserRequest {
  int64 id = 1;
}                                    
                                    
message DeleteUserResponse {
  int64 rowsDeleted = 1;
}
                                    
service UserService {
  rpc Create(CreateUserRequest) returns (CreateUserResponse);

  rpc Read(ReadUserRequest) returns (ReadUserResponse);
                                    
  rpc Delete(DeleteUserRequest) returns (DeleteUserResponse);
}                                   

{% endhighlight %}


**Generate Code from user.proto**

I assume you are in the *user* directory we created.  If so run the following command
{% highlight ruby %}
> protoc -I proto/ user.proto --go_out=plugins=grpc:proto 
{% endhighlight %}

You should now see a file named user.pb.go in the *proto* directory now.  If you look at that file you see you will have to implement the *UserServiceServer* and that the *UserServiceClient* is already implemented.

**UserServiceServer Implementation**

I created another directory under *user* named *user_developer* and created a file named *main.go*

{% highlight go %}
package main                                                                                                                                                                                                                                 
import (    
  "context"                                        
  "database/sql"    
  "fmt"    
  "log"    
  "net"    
    
  pb "github.com/calam1/user/proto"    
  _ "github.com/go-sql-driver/mysql"    
  "google.golang.org/grpc"    
  "google.golang.org/grpc/codes"    
  "google.golang.org/grpc/reflection"    
  "google.golang.org/grpc/status"    
)    

// implementation of UserServiceServer defined in generated go file
type userServer struct {    
  db *sql.DB    
}    
    
// NewUserServiceServer returns implemented server    
func NewUserServiceServer(db *sql.DB) pb.UserServiceServer {    
  return &userServer{db: db}    
}    

// get database connection
func (u *userServer) connect(ctx context.Context) (*sql.Conn, error) {    
  c, err := u.db.Conn(ctx)    
  if err != nil {    
    return nil, status.Error(codes.Unknown, "failed to connect to db"+err.Error())    
  }    
  return c, nil    
}    
    
// Create create a user    
func (u *userServer) Create(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {    
  c, err := u.connect(ctx)    
  if err != nil {    
    return nil, err    
  }    
                                     
  defer c.Close()    
    
  result, err := c.ExecContext(ctx, "INSERT INTO User(`FirstName`, `LastName`, `Address`) VALUES (?, ?, ?)", req.User.FirstName, req.User.LastName, req.User.Address)    
                                     
  if err != nil {                                        
    return nil, status.Error(codes.Unknown, "failed to insert into User table"+err.Error())    
  }    
                                     
  id, err := result.LastInsertId()    
  if err != nil {    
    return nil, status.Error(codes.Unknown, "failed to retrieve last inserted record for User"+err.Error())    
  }    

  return &pb.CreateUserResponse{
    Id: id,
  }, nil
}

// Read get a user
func (u *userServer) Read(ctx context.Context, req *pb.ReadUserRequest) (*pb.ReadUserResponse, error) {
  c, err := u.connect(ctx)
  if err != nil {
    return nil, err
  }

  defer c.Close()

  result, err := c.QueryContext(ctx, "SELECT `ID`, `FIRSTNAME`, `LASTNAME`, `ADDRESS` FROM User WHERE `ID` = ?", req.Id)

  if err != nil {
    return nil, status.Error(codes.Unknown, "failed to select id from User table"+err.Error())
  }

  // don't forget to close the result otherwise you will hang and timeout
  defer result.Close()

  var usr pb.User

  if result.Next() {
    if err := result.Scan(&usr.Id, &usr.FirstName, &usr.LastName, &usr.Address); err != nil {
      return nil, status.Error(codes.Unknown, "failed to retrieve values from User"+err.Error())
    }
  } else {
    return nil, status.Error(codes.NotFound, fmt.Sprintf("User with id %d is not found", req.Id))
  }

  return &pb.ReadUserResponse{
    User: &usr,
  }, nil
}

func (u *userServer) Delete(ctx context.Context, req *pb.DeleteUserRequest) (*pb.DeleteUserResponse, error) {
  c, err := u.connect(ctx)
  if err != nil {
    return nil, err
  }
  defer c.Close()

  result, err := c.ExecContext(ctx, "DELETE FROM User WHERE ID = ?", req.Id)
  if err != nil {
    return nil, status.Error(codes.Unknown, "failed to delete User"+err.Error()) 
  }

  rows, err := result.RowsAffected()
  if err != nil {
    return nil, status.Error(codes.Unknown, "failed to retrieve rows affected"+err.Error())
  }


  if rows == 0 {
    return nil, status.Error(codes.NotFound, fmt.Sprintf("User with id %d is not found", req.Id))
  }

  return &pb.DeleteUserResponse{
    RowsDeleted: rows,
  }, nil
}

func main() {

  // this is hard coded for simplicity sake, you can use golang flags to get the attributes
  dsUser := "chris:password@tcp(172.17.0.2:3306)/customers?User"

  // connect to the db
  db, err := sql.Open("mysql", dsUser)
  if err != nil {
    log.Fatalf("error opening database %v", err)
  }

  defer db.Close()

  // grpc server listen port
  listener, err := net.Listen("tcp", ":50051")
  if err != nil {
    log.Fatalf("listener failed:%v", err)
  }

  // create a new server based off of the proto generated code
  userAPI := NewUserServiceServer(db)

  // create a grpc server
  grpcServer := grpc.NewServer()
  // register grpc server and user service server
  pb.RegisterUserServiceServer(grpcServer, userAPI)
  reflection.Register(grpcServer)
  if err := grpcServer.Serve(listener); err != nil {
    log.Fatalf("failed to serve %v", err)
  }
}                       
{% endhighlight %}

**Note**
In the above code, the server has the database string hard coded.  You can use [flags][golang-flags-url] to read the arguments in and parse them to build the database string

{% highlight go %}
  // dbuser := "<user>:<password>@tcp(<mysql-dns-url>:<db-port>)/<database>?<table>"
  dsUser := "chris:password@tcp(172.17.0.2:3306)/customers?User"
{% endhighlight %}

If you are running docker to host the database then run the following command to get thei ip address for the database url
{% highlight ruby %}
> docker inspect mysql | grep -i ipaddress
{% endhighlight %}


**Build the Client**

Under the *user* directory create a directory name *user_client* and create a file name main.go and add the following content

This client will do a insert, read, and delete as a simple example.

{% highlight go %}
package main                                                                                                                                                                                                                                 
  
import (
  "context"
  "log"                                                       
  "time"

  pb "github.com/calam1/user/proto"
  "google.golang.org/grpc"   
)
                                 
const (
  address = "localhost:50051"
)
  
func main() {
  // insecure grpc connection   
  conn, err := grpc.Dial(address, grpc.WithInsecure())
  if err != nil {   
    log.Fatalf("connection error %v", err)
  }

  defer conn.Close()
  
  client := pb.NewUserServiceClient(conn)
 
  // timeout
  ctx, cxl := context.WithTimeout(context.Background(), 10*time.Second)
  defer cxl()
      
  // create user                                         
  reqCreate := pb.CreateUserRequest{   
    User: &pb.User{
      FirstName: "Chris",             
      LastName:  "Lam",            
      Address:   "123 Main Street USA",
    },
  }
   
  respCreate, err := client.Create(ctx, &reqCreate)
  if err != nil {                 
    log.Fatalf("Create User failed %v", err)
  }                               
  log.Printf("Created User: %v", respCreate)
  
  id := respCreate.Id
   
  // read user                     
  reqRead := pb.ReadUserRequest{
    Id: id,
  }

  respRead, err := client.Read(ctx, &reqRead)
  if err != nil {
    log.Fatalf("read failed %v", err)
  }

  log.Printf("read response %v", respRead)

  // delete user
  reqDelete := pb.DeleteUserRequest{
    Id: id,
  }

  respDelete, err := client.Delete(ctx, &reqDelete)
  if err != nil {
    log.Fatalf("delete failed for %v on erro r%v", id, err)
  }

  log.Printf("delete response %v", respDelete)
}                                                            
{% endhighlight %}

**Build and Run**

*Note: yes, you should not use underscores in golang, camelcase is the preferred style, but it is just easier to read for me*

Under the *user_server* directory run
{% highlight ruby %}
> build go .
{% endhighlight %}
you will see a generated file named *user_server*. 

Start the server
{% highlight ruby %}
> ./user_server
{% endhighlight %}


Under the *user_client* directory run
{% highlight ruby %}
> build go .
{% endhighlight %}
you will see a generated file named *user_client*. 

Start the client
{% highlight ruby %}
> ./user_client
{% endhighlight %}

You will see something like this
{% highlight ruby %}
2019/02/08 14:52:43 Created User: id:7 
2019/02/08 14:52:43 read response user:<id:7 firstName:"Chris" lastName:"Lam" address:"123 Main Street USA" > 
2019/02/08 14:52:43 delete response rowsDeleted:1 
{% endhighlight %}

[github-url]: https://github.com/calam1/grpc-tutorial
[golang-flags-url]: https://golang.org/pkg/flag/
